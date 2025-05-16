# GEM5 Minor Core 解析

本人以Minor Core为模板来学习Minor Core，并进而学习GEM5仿真系统，以此为记录。
以流水线Front-End到Back-End为顺序来进行解析，如有错误，敬请指正。

行文以一个初学者第一视角开始，希望给读者带来一个不错的体验。

## pipeline

此部分主要包含两部分代码，pipeline.cc和pipeline.hh.本文会将两个文件糅合在一起讲解，按照阅读的先后顺序来讲解。

你可以认为pipeline是Minor Core的最顶层级，他负责把Fetch1，Fetch2，Decode和Execute等模块组装起来构成核心流水线。我们还是一样一边阅读代码一边解析代码含义。

```cpp
/* Pipeline这个类继承了Ticked，我们有必要先学习Ticked这个基类*/
/* 简述一下，继承Ticked表示流水线是受时钟周期驱动的*/
class Pipeline : public Ticked
{
  protected:
    MinorCPU &cpu;

    /* 一个配置，允许当流水线挂起时跳过时钟 */
    bool allow_idling;

    /* Latch是一种DelayFifo*/
    /* f1ToF2 是 Fetch1到Fetch2的Delay-FIFO*/
    Latch<ForwardLineData> f1ToF2;
    /* f2ToF1 是 Fetch2给Fetch1的Delay-FIFO，主要是分支预测的结果*/
    Latch<BranchData> f2ToF1;
    /* f2ToD 是 Fetch2给Decode阶段的Delay-FIFO*/
    Latch<ForwardInstData> f2ToD;
    /* dToE 是 Decode给Execute阶段的Delay-FIFO*/
    Latch<ForwardInstData> dToE;
    /* eToF1 是 Execute阶段给Fetch1的Delay-FIFO，主要传递实际分支结果*/
    Latch<BranchData> eToF1;
    /* 流水线的四个阶段，从前到后依次是Fetch1 Fetch2 Decode Execute*/
    Execute execute;
    Decode decode;
    Fetch2 fetch2;
    Fetch1 fetch1;

    /* 行动记录器，用于跟踪各阶段活动状态，用于性能分析、流水线停顿诊断和资源利用率统计 */
    /* 如果你想了解MinorActivityRecorder的代码解析，请访问${FILE_ROOT}/Appendix/ActivityRecorder.md，看完再回来*/
    MinorActivityRecorder activityRecorder;

  public:
    /* 流水线阶段标识枚举 */
    enum StageId
    {
        /* CPU整体唤醒标识 */
        CPUStageId = 0,
        /* 各个流水线阶段的标识 */
        Fetch1StageId, Fetch2StageId, DecodeStageId, ExecuteStageId,
        Num_StageId /* 总Stage统计 */
        /* 为什么要整个这个enum： 为每个流水线阶段分配唯一ID，用于ActivityRecorder来进行状态跟踪*/
        /* CPUStageID作为特殊标识，用于CPU整体唤醒事件记录*/
        /* Num_StageId用于自动化遍历阶段（如Num_StageId=4表示共有4个阶段）*/
    };

    /* 先介绍Drained机制：在模拟器需要保存状态或热迁移时，需要排空流水线的未完成指令 */
    /* 如果这个变量是true，表示drain()已调用但是尚未完成排空流程*/
    /* 相关函数：isDrained()检查是否排空，darin()启动排空*/
    bool needToSignalDrained;

  public:
    /* 构造函数，params是由python脚本来动态配置的参数，*/
    /* 关于如何通过python脚本来配置参数：可以看${FILE_ROOT}/Appendix/Python脚本传配置.md*/
    Pipeline(MinorCPU &cpu_, const BaseMinorCPUParams &params):
        /*Ticked是基类，对基类进行初始化，建立周期驱动机制*/
        /*将CPU实例通过cpu_传输，baseStats.numCycles是连接CPU基础统计的周期计数器*/
        /*每个周期自动调用evaluate()*/
        Ticked(cpu_, &(cpu_.BaseCPU::baseStats.numCycles)),
        /*保存对CPU实体的引用*/
        cpu(cpu_),
        allow_idling(params.enableIdling),
        /*如果你忘了Latch这个结构体的解析，可以看${FILE_ROOT}/Appendix/Latch.md*/
        /*首先传递名称，以及传递介质的名称，比如prediction就是分支预测的结果，然后是通过params配置的延迟*/
        f1ToF2(cpu.name() + ".f1ToF2", "lines",
            params.fetch1ToFetch2ForwardDelay),
        f2ToF1(cpu.name() + ".f2ToF1", "prediction",
            params.fetch1ToFetch2BackwardDelay, true),
        f2ToD(cpu.name() + ".f2ToD", "insts",
            params.fetch2ToDecodeForwardDelay),
        dToE(cpu.name() + ".dToE", "insts",
            params.decodeToExecuteForwardDelay),
        eToF1(cpu.name() + ".eToF1", "branch",
            params.executeBranchDelay),
        /*以上都是对Latch的构造，Latch就是一种Delay-FIFO*/
        /*模块的构造函数，首先是名称，然后是将CPU实例传进去，然后是CPU的配置*/
        /*然后dtoE.output()表示Latch的输出端作为Execute输入端连进去，eToF1.input()表示Latch的输入端作为输出端连到Fetch1*/
        execute(cpu.name() + ".execute", cpu, params,
            dToE.output(), eToF1.input()),
        decode(cpu.name() + ".decode", cpu, params,
            f2ToD.output(), dToE.input(), execute.inputBuffer),
        fetch2(cpu.name() + ".fetch2", cpu, params,
            f1ToF2.output(), eToF1.output(), f2ToF1.input(), f2ToD.input(),
            decode.inputBuffer),
        fetch1(cpu.name() + ".fetch1", cpu, params,
            eToF1.output(), f1ToF2.input(), f2ToF1.output(), fetch2.inputBuffer),
        /*现在我们把几个流水线模块连接好了*/
        /*activityRecorder的构造函数，见${FILE_ROOT}/Appendix/ActivityRecorder.md*/
        activityRecorder(cpu.name() + ".activity", Num_StageId,
            /* The max depth of inter-stage FIFOs */
            std::max(params.fetch1ToFetch2ForwardDelay,
            std::max(params.fetch2ToDecodeForwardDelay,
            std::max(params.decodeToExecuteForwardDelay,
            params.executeBranchDelay)))),
        needToSignalDrained(false)
    {
        /*一些错误检查，比如delay必须起码为1，不符合条件就报FATAL*/
        if (params.fetch1ToFetch2ForwardDelay < 1) {
            fatal("%s: fetch1ToFetch2ForwardDelay must be >= 1 (%d)\n",
                cpu.name(), params.fetch1ToFetch2ForwardDelay);
        }

        if (params.fetch2ToDecodeForwardDelay < 1) {
            fatal("%s: fetch2ToDecodeForwardDelay must be >= 1 (%d)\n",
                cpu.name(), params.fetch2ToDecodeForwardDelay);
        }

        if (params.decodeToExecuteForwardDelay < 1) {
            fatal("%s: decodeToExecuteForwardDelay must be >= 1 (%d)\n",
                cpu.name(), params.decodeToExecuteForwardDelay);
        }

        if (params.executeBranchDelay < 1) {
            fatal("%s: executeBranchDelay must be >= 1\n",
                cpu.name(), params.executeBranchDelay);
        }
    }

  public:
    /* 线程激活（尤其是静默唤醒后）需要通过这个操作 */
    void wakeupFetch(ThreadID tid)
    {
        /* 唤醒fetch1，让他开始fetch*/
        fetch1.wakeupFetch(tid);
    }

    /* 排空控制三件套drain(), drainResume(), isDrained() */
    bool drain()
    {
        DPRINTF(MinorCPU, "Draining pipeline by halting inst fetches. "
            " Execution should drain naturally\n");
        /*调用execute模块drain*/
        execute.drain();

        /* isDrained() */
        bool drained = isDrained();
        needToSignalDrained = !drained;

        return drained;
    }

    void drainResume();

    /* */
    bool isDrained();

    /* evaluate是Ticked里的纯虚函数，这里给了重载实现*/
    void evaluate() override
    {
        /* 更新程序计数器 */
        cpu.tick();

        /* 倒序写法，对需要时序的仿真系统这是必要的*/
        execute.evaluate();
        decode.evaluate();
        fetch2.evaluate();
        fetch1.evaluate();
        /* 流水线状态追踪*/
        if (debug::MinorTrace)
            minorTrace();

        /* 这个地方的设计非常重要，把Latch或Delay-FIFO的时序推进放在stage时序推进的后面，这样Latch不会因为输入端的Push或输出端Pop改变自身应有的状态 */
        /* 因为这些evaluate()都是同一个cycle发生的事情，他们之间不应该有cycle内的互相影响。*/
        f1ToF2.evaluate();
        f2ToF1.evaluate();
        f2ToD.evaluate();
        dToE.evaluate();
        eToF1.evaluate();

        /* 记录各阶段活跃状态 */
        activityRecorder.evaluate();

        if (allow_idling) {
            /* TODO：待解释 */
            if (!activityRecorder.active() && !needToSignalDrained) {
                DPRINTF(Quiesce, "Suspending as the processor is idle\n");
                stop();
            }

            /* TODO：待解释，ActivitiyRecorder */
            activityRecorder.deactivateStage(Pipeline::CPUStageId);
            activityRecorder.deactivateStage(Pipeline::Fetch1StageId);
            activityRecorder.deactivateStage(Pipeline::Fetch2StageId);
            activityRecorder.deactivateStage(Pipeline::DecodeStageId);
            activityRecorder.deactivateStage(Pipeline::ExecuteStageId);
        }

        if (needToSignalDrained) /* TODO： 待解释，Drain机制 */
        {
            DPRINTF(Drain, "Still draining\n");
            if (isDrained()) {
                DPRINTF(Drain, "Signalling end of draining\n");
                cpu.signalDrainDone();
                needToSignalDrained = false;
                stop();
            }
        }
    }

    void minorTrace() const;

    /** Functions below here are BaseCPU operations passed on to pipeline
     *  stages */

    /** 返回Fetch1里面的ICachePort */
    MinorCPU::MinorCPUPort &getInstPort();
    /** 返回Execute里面的DCachePort */
    MinorCPU::MinorCPUPort &getDataPort();

    /** 返回activityRecorder接口 */
    MinorActivityRecorder *getActivityRecorder() { return &activityRecorder; }
}

```

这就是MinorCore里pipeline的代码解析，我们可以注意到核心的仿真是evaluate，pipeline的evaluate会调用各个stage的evaluate。