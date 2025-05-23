# GEM5源码解析

## Ticked

边看代码边学习，ticked_object.cc和ticked_object.hh

```cpp
/* Ticked继承了Serializable说明类的对象支持序列化（检查点） */
class Ticked : public Serializable
{
  protected:
    /* 负责此Ticked动作和统计的ClockedObject*/
    ClockedObject &object;

    /* Event的抓手，可参阅eventQueue-Part*.md部分 */
    /* 这里将Event包装了起来，从而将要执行的函数绑定到eventQueue*/
    EventFunctionWrapper event;

    /* 这里是Ticked类的核心方法，在每次事件触发时更新统计计数器、执行evaluate()，周期性触发 */
    void processClockEvent()
    {
        /*实际运行的周期数*/
        ++tickCycles;
        /*总周期数（运行+停止，用于统计整体时间）*/
        ++numCycles;
        /*通知派生类当前周期已处理*/
        countCycles(Cycles(1));
        /*核心逻辑，调用派生类的evaluate()，这样就将evaluate和事件队列关联起来了*/
        evaluate();
        /*若处于运行状态，重新调度到下一个周期，通过绑定的ClockedObject将事件加入事件队列*/
        if (running)
            /*计算下一个时钟边沿的时间点。假设当前为周期T，该方法返回周期T+1的触发时间*/
            object.schedule(event, object.clockEdge(Cycles(1)));
    }

    /* Ticked对象的状态，是否正在运行 */
    bool running;

    /* 最后一次停止的时间，用于计算运行时间 */
    Cycles lastStopped;

  private:
    /* 若无外部导入，本地分配的统计量 */
    statistics::Scalar *numCyclesLocal;

  protected:
    /* 总周期数（运行+停止） */
    statistics::Scalar &numCycles;

    /* 实际运行的周期数 */
    statistics::Scalar tickCycles;

    /* 停止的空闲周期数 idleCycles = numCycles - tickCycles */
    statistics::Formula idleCycles;

  public:
    Ticked(ClockedObject &object_,
        statistics::Scalar *imported_num_cycles = NULL,
        /* 事件优先级，参阅EventQueue*.md*/
        Event::Priority priority = Event::CPU_Tick_Pri):
        /*必须传入一个ClockedObject实例，勇于事件调度*/
        object(object_),
        /**/
        event([this]{ processClockEvent(); }, object_.name(), false, priority),
        running(false),
        lastStopped(0),
        /* 若无外部导入，则本地创建统计量 */
        numCyclesLocal((imported_num_cycles ? NULL : new statistics::Scalar)),
        /* 若无外部导入，则本地创建统计量 */
        numCycles((imported_num_cycles ? *imported_num_cycles :
            *numCyclesLocal)){};

    ...
```
这个构造函数里有一个Lambda表达式构成构造函数，这里可能会令人困惑，所以打断一下，来分析一下这里的构造函数。

这里的核心目的是将Ticked类的成员函数processClockEvent()绑定到Event的事件call中。
```cpp
event([this]{ processClockEvent(); }, object_.name(), false, priority)
```
这里event是EventFunctionWrapper类型的成员变量，负责将函数包装成事件。
EventFunctionWrapper需要接收一个std::function<void(void)>类型的回调函数，用于定义事件触发时的行为。

那么这里为什么要用Lambda表达式呢？
processClockEvent()是Ticked类的非静态成员函数，调用它需要一个Ticked实例（this指针）。
事件系统需要一个无参数且返回void的函数，所以这里捕获this指针，间接调用processClockEvent()。
```cpp
[this]{ processClockEvent(); }
```
这里[this]捕获Ticked对象的this指针，使得在Lambda内部可以访问该对象的成员函数。
当事件被触发时，执行this->processClockEvent()

如果不使用Lambda，可以手动绑定成员函数，比如通过std::bind
```cpp
event(std::bind(&Ticked::processClockEvent,this),...)
```

如果你阅读了eventQueue部分的内容，到这里应该能理解一个Ticked对象是如何与eventQueue关联起来的。
一个Ticked对象对应一个EventFunctionWrapper，在调度时刻执行的是Ticked对象的evaluate()方法。
核心的函数就是processClockEvent()。
    
```cpp
    /*允许派生类正确释放资源*/
    virtual ~Ticked() { }

    /* 统计量注册 */
    void regStats()
    {
        if (numCyclesLocal) {
            numCycles
                .name(object.name() + ".totalTickCycles")
                .desc("Number of cycles that the object ticked or was stopped");
        }

        tickCycles
            .name(object.name() + ".tickCycles")
            .desc("Number of cycles that the object actually ticked");

        idleCycles
            .name(object.name() + ".idleCycles")
            .desc("Total number of cycles that the object has spent stopped");
        idleCycles = numCycles - tickCycles;
    };

    /* 启动计时 */
    void
    start()
    {
        if (!running) {
            /* 如果事件未调度，安排到下一周期*/
            if (!event.scheduled())
                object.schedule(event, object.clockEdge(Cycles(1)));
            running = true;
            /* 累计停止期间的周期数 */
            numCycles += cyclesSinceLastStopped();
            /* 通知派生类周期变化*/
            countCycles(cyclesSinceLastStopped());
        }
    }

    /** How long have we been stopped for? */
    Cycles
    cyclesSinceLastStopped() const
    {
        return object.curCycle() - lastStopped;
    }

    /** Reset stopped time to current time */
    void
    resetLastStopped()
    {
        lastStopped = object.curCycle();
    }

    /* 如果事件被调度了，那么将事件移除eventQueue，阻止后续触发 */
    void
    stop()
    {
        if (running) {
            if (event.scheduled())
                object.deschedule(event);
            running = false;
            /*记录停止事件为当前周期*/
            resetLastStopped();
        }
    }

    /* 序列化与反序列化，将Ticked对象的状态保存到检查点(checkpoint)并恢复 */
    /* 支持模拟器的暂停/继续或调试*/
    void serialize(CheckpointOut &cp) const override
    {
        uint64_t lastStoppedUint = lastStopped;

        paramOut(cp, "lastStopped", lastStoppedUint);
    };

    void unserialize(CheckpointIn &cp) override
    {
        uint64_t lastStoppedUint = 0;

        optParamIn(cp, "lastStopped", lastStoppedUint);

        lastStopped = Cycles(lastStoppedUint);
    };

    /* 派生类必须实现evaluate，就是派生类在这个仿真周期内要做什么 */
    virtual void evaluate() = 0;

    /* 当周期变化时（如启动/恢复时），通知派生类进行扩展统计或触发探针*/
    /* delta为自从上次回调以来的周期数（通常为1）*/
    virtual void countCycles(Cycles delta) {}
};

/* 多重继承ClockedObject和Ticked，将Ticked的周期性事件机制与ClockedObject的时钟域管理绑定，形成可复用的基类*/
/* ClockedObject提供时钟域和时钟边沿计算能力 */
/* Ticked管理周期性事件调度和统计 */
class TickedObject : public ClockedObject, public Ticked
{
  public:
    TickedObject(const TickedObjectParams &params,
        Event::Priority priority = Event::CPU_Tick_Pri):
        ClockedObject(params),
        Ticked(*this, NULL, priority)
        {}

    /* 解决多重继承的名称冲突，将regStats、serialize、unserialize方法引入当前作用域，允许派生类直接调用 */
    using ClockedObject::regStats;
    using ClockedObject::serialize;
    using ClockedObject::unserialize;

    /* 注册统计量，序列化与反序列化 */
    void regStats() override
    {
        Ticked::regStats();
        ClockedObject::regStats();
    };

    void serialize(CheckpointOut &cp) const override
    {
        Ticked::serialize(cp);
        ClockedObject::serialize(cp);
    };

    void unserialize(CheckpointIn &cp) override
    {
        Ticked::unserialize(cp);
        ClockedObject::unserialize(cp);
    };

};

```

可以回忆一下从Flags到TickedObject，我们已经建立起比较深的联系了，就是这个仿真内核是如何work的。
