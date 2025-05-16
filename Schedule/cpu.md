# GEM5 Minor Core 解析

## cpu

这一部分对应两个文件cpu.cc和cpu.hh。在讲解的过程中会合二为一的进行讲解，还是一样，边看代码边学习。

```cpp

/* 提前声明，解决Pipeline和MinorCPU类的循环依赖，也就是解决头文件互相引用的问题*/
/* 这样做是为了允许MinorCPU类通过指针引用Pipeline而无需包含完整定义*/
class Pipeline;

/* 线程类型定义，将SimpleThread重命名为MinorThread，作为MinorCPU的线程实现 */
typedef SimpleThread MinorThread;

} // namespace minor

class MinorCPU : public BaseCPU
{
  protected:
    /* 前面提到的，指向流水线管理的指针，pipeline负责协调Fetch1/Fetch2/Decode/Execute阶段 */
    minor::Pipeline *pipeline;
    /* 随机数生成器，用于线程调度里面的随机策略（线程调度策略里面有一个随机优先级）*/
    Random::RandomPtr rng = Random::genRandom();

  public:
    /* ActivityRecorder是活动状态记录器，用于调试和性能分析 */
    minor::MinorActivityRecorder *activityRecorder;

    /* 存储所有线程的指针，每个线程对应一个MinorThread(也就是SimpleThread) */
    /* 这样做是方便获得线程上下文，通过threads[threadId]->getTC()来获得线程上下文*/
    std::vector<minor::MinorThread *> threads;

  public:
    /* 定义CPU与内存系统交互的端口基类，RequestPort等Port类的解析参阅${FILE_ROOT}/Appendix/Port基类*/
    class MinorCPUPort : public RequestPort
    {
      public:
        MinorCPU &cpu;
      public:
        MinorCPUPort(const std::string& name_, MinorCPU &cpu_)
            : RequestPort(name_), cpu(cpu_)
        { }
    };

    /* 通过BaseMinorCPUParams参数设置（也就是Python脚本里指定的策略），影响roundRobinPriority()等策略的调用 */
    enums::ThreadPolicy threadPolicy;
  protected:
     /* 返回Execute阶段与DCache连接的Port */
    Port &getDataPort() override;

    /* 返回Fetch1阶段与ICache连接的Port */
    Port &getInstPort() override;

  public:
    /* 构造函数*/
    MinorCPU(const BaseMinorCPUParams &params);
    /* 析构函数*/
    ~MinorCPU();

  public:
    /* 初始化线程和内存映射 */
    void init() override
    {
        /* 调用基类方法，BaseCPU基类的解析参阅${FILE_ROOT}/Appendix/BaseCPU基类.md */
        BaseCPU::init();
        /* 错误检查，要求MinorCPU的内存是Timing的内存*/
        if (!params().switched_out && system->getMemoryMode() != enums::timing) {
            fatal("The Minor CPU requires the memory system to be in "
                "'timing' mode.\n");
        }
    }

    /* 启动流水线时钟事件*/
    void startup() override
    {
        DPRINTF(MinorCPU, "MinorCPU startup\n"); 

        /* 调用基类方法startup*/
        BaseCPU::startup();

        for (ThreadID tid = 0; tid < numThreads; tid++)
            /* 对于每一个线程，调用wakeupFetch()*/
            pipeline->wakeupFetch(tid);
    }

    /* 唤醒挂起的线程*/
    void wakeup(ThreadID tid) override
    {
        DPRINTF(Drain, "[tid:%d] MinorCPU wakeup\n", tid);
        assert(tid < numThreads);
        /* 如果线程的状态是挂起的话，就唤醒线程*/
        if (threads[tid]->status() == ThreadContext::Suspended) {
            threads[tid]->activate();
        }
    }




    /** Processor-specific statistics */
    minor::MinorStats stats;

    /** Stats interface from SimObject (by way of BaseCPU) */
    void regStats() override;

    /** Simple inst count interface from BaseCPU */
    Counter totalInsts() const override;
    Counter totalOps() const override;

    void serializeThread(CheckpointOut &cp, ThreadID tid) const override;
    void unserializeThread(CheckpointIn &cp, ThreadID tid) override;

    /** Serialize pipeline data */
    void serialize(CheckpointOut &cp) const override;
    void unserialize(CheckpointIn &cp) override;

    /** Drain interface */
    DrainState drain() override;
    void drainResume() override;
    /** Signal from Pipeline that MinorCPU should signal that a drain
     *  is complete and set its drainState */
    void signalDrainDone();
    void memWriteback() override;

    /** Switching interface from BaseCPU */
    void switchOut() override;
    void takeOverFrom(BaseCPU *old_cpu) override;

    /** Thread activation interface from BaseCPU. */
    void activateContext(ThreadID thread_id) override;
    void suspendContext(ThreadID thread_id) override;

    /** Thread scheduling utility functions */
    std::vector<ThreadID> roundRobinPriority(ThreadID priority)
    {
        std::vector<ThreadID> prio_list;
        for (ThreadID i = 1; i <= numThreads; i++) {
            prio_list.push_back((priority + i) % numThreads);
        }
        return prio_list;
    }

    std::vector<ThreadID> randomPriority()
    {
        std::vector<ThreadID> prio_list;
        for (ThreadID i = 0; i < numThreads; i++) {
            prio_list.push_back(i);
        }

        std::shuffle(prio_list.begin(), prio_list.end(),
                     rng->gen);

        return prio_list;
    }

    /** The tick method in the MinorCPU is simply updating the cycle
     * counters as the ticking of the pipeline stages is already
     * handled by the Pipeline object.
     */
    void tick() { updateCycleCounters(BaseCPU::CPU_STATE_ON); }

    /** Interface for stages to signal that they have become active after
     *  a callback or eventq event where the pipeline itself may have
     *  already been idled.  The stage argument should be from the
     *  enumeration Pipeline::StageId */
    void wakeupOnEvent(unsigned int stage_id);
    EventFunctionWrapper *fetchEventWrapper;
};


```

我们来一个总结：