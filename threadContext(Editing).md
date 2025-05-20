# GEM5 Minor Core 解析

这一部分是对GEM5 CPU部分threadContext的解析：

## ThreadContext

这一部分代码位于gem5/src/cpu/thread_context.hh和gem5/src/cpu/thread_context.cc。
合二为一，边看代码边解析：

```cpp
/*PCEventScope时GEM5中用于管理PC事件的基类*/
/*PCEventScope包含两个纯虚函数remove和schedule*/
/*ThreadContext表示一个线程的上下文，包含线程状态、寄存器、内存映射等信息，是一个核心抽象*/
class ThreadContext : public PCEventScope
{
  protected:
    /*useForClone是一个标志位，控制该线程上下文是否可以被克隆*/
    bool useForClone = false;

  public:
    /*获得是否可被克隆的信息*/
    bool getUseForClone() { return useForClone; }
    /*设置是否可被克隆的标志*/
    void setUseForClone(bool new_val) { useForClone = new_val; }

    enum Status
    {
        /*线程运行中，线程正在执行指令*/
        Active,

        /*临时挂起，等待同步（如锁、信号量）或其他事件*/
        Suspended,

        /*正在退出，已触发退出，但需要等待事件完成（如资源释放）*/
        Halting,

        /*已终止：线程完全退出，模拟结束时如果所有线程出于此状态，则停止模拟*/
        Halted
    };
    ...
}
```

ThreadContext是模拟线程的核心类，每个线程为一个仿真实例。你可以认为模拟器是以线程为粒度来进行仿真的。
线程在运行过程中可能经历状态迁移，比如：
    Active->Suspended: 等待锁或中断
    Active->Halting: 执行退出指令
    Halting->Halted: 资源释放完成

useForClone用于控制是否允许通过此上下文创建新线程，比如在线程池场景下，会复用上下文配置。

继续：

```cpp

class ThreadContext : public PCEventScope
{
    ...
    /*析构函数，确保派生类的析构函数被正确调用*/
    virtual ~ThreadContext() { };
    /*获得相关的CPU实例*/
    virtual BaseCPU *getCpuPtr() = 0;
    /*返回CPU的逻辑ID*/
    virtual int cpuId() const = 0;
    /*返回CPU的物理插槽ID*/
    virtual uint32_t socketId() const = 0;
    /*在多核和多CPU系统中，标识线程所属的CPU及物理位置*/

    /*返回线程的ID（CPU内部编号）*/
    virtual int threadId() const = 0;
    /*设置线程ID*/
    virtual void setThreadId(int id) = 0;
    /*返回全局唯一的上下文ID*/
    virtual ContextID contextId() const = 0;
    /*设置上下文ID*/
    virtual void setContextId(ContextID id) = 0;
    /*以上部分是为了在CPU核上模拟多线程（如SMT超线程）时，区分不同硬件线程*/

    /*获得MMU实例*/
    virtual BaseMMU *getMMUPtr() = 0;
    /*获取验证器CPU*/
    virtual CheckerCPU *getCheckerCpuPtr() = 0;
    /*获得ISA对象*/
    virtual BaseISA *getIsaPtr() const = 0;
    /*获取指令解码器*/
    virtual InstDecoder *getDecoderPtr() = 0;
    /*获取模拟的系统实例*/
    virtual System *getSystemPtr() = 0;
    /*发送功能性的内存请求*/
    virtual void sendFunctional(PacketPtr pkt);
    /*获取关联的Process实例*/
    virtual Process *getProcessPtr() = 0;
    /*绑定Process实例，Process为进程，Thread为线程，通过Process对象管理内存映射和资源*/
    virtual void setProcessPtr(Process *p) = 0;
    /*返回线程当前的状态，Active，Suspended等*/
    virtual Status status() const = 0;
    /*设置线程当前状态*/
    virtual void setStatus(Status new_status) = 0;

    /*激活线程，状态设置为Active*/
    virtual void activate() = 0;

    /*挂起线程，状态设置为Suspended*/
    virtual void suspend() = 0;

    /*终止线程，状态设置为Halted*/
    virtual void halt() = 0;

    /*工具函数，暂停线程*/
    void quiesce();

    /*暂停并且在指定周期后激活*/
    void quiesceTick(Tick resume);
    /*接管旧的线程上下文状态*/
    virtual void takeOverFrom(ThreadContext *old_context) = 0;
    /*注册统计信息*/
    virtual void regStats(const std::string &name) {};
    /**/
    virtual void scheduleInstCountEvent(Event *event, Tick count) = 0;
    /**/
    virtual void descheduleInstCountEvent(Event *event) = 0;
    /*获取当前已执行指令数*/
    virtual Tick getCurrentInstCount() = 0;

    virtual Tick readLastActivate() = 0;
    virtual Tick readLastSuspend() = 0;
    /*复制架构寄存器到另一个线程上下文*/
    virtual void copyArchRegs(ThreadContext *tc) = 0;
    /*清空架构寄存器*/
    virtual void clearArchRegs() = 0;

    /*读取寄存器值*/
    virtual RegVal getReg(const RegId &reg) const;

    virtual void getReg(const RegId &reg, void *val) const = 0;
    /*获取可写的寄存器指针*/
    virtual void *getWritableReg(const RegId &reg) = 0;
    /*写入寄存器值*/
    virtual void setReg(const RegId &reg, RegVal val);
    virtual void setReg(const RegId &reg, const void *val) = 0;
    /*获取当前PC的状态*/
    virtual const PCStateBase &pcState() const = 0;
    /*设置PC状态*/
    virtual void pcState(const PCStateBase &val) = 0;
    void
    pcState(Addr addr)
    {
        std::unique_ptr<PCStateBase> new_pc(getIsaPtr()->newPCState(addr));
        pcState(*new_pc);
    }
    /*设置PC（不记录，如分支预测）*/
    virtual void pcStateNoRecord(const PCStateBase &val) = 0;

    virtual RegVal readMiscRegNoEffect(RegIndex misc_reg) const = 0;
    
    virtual RegVal readMiscReg(RegIndex misc_reg) = 0;

    virtual void setMiscRegNoEffect(RegIndex misc_reg, RegVal val) = 0;

    virtual void setMiscReg(RegIndex misc_reg, RegVal val) = 0;

    virtual unsigned readStCondFailures() const = 0;

    virtual void setStCondFailures(unsigned sc_failures) = 0;

    /*终止线程*/
    virtual int exit() { return 1; };
    /*比较两个上下文状态，用于调试*/
    static void compare(ThreadContext *one, ThreadContext *two);

    virtual void htmAbortTransaction(uint64_t htm_uid,
                                     HtmFailureFaultCause cause) = 0;
    virtual BaseHTMCheckpointPtr& getHtmCheckpointPtr() = 0;
    virtual void setHtmCheckpointPtr(BaseHTMCheckpointPtr cpt) = 0;
}



```