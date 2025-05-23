# GEM5源码解析

## Clocked

这一部分学习BaseCPU所继承的ClockedObject，ClockedObject又继承Clocked和SimObject。

相关的代码是clocked_object.hh和clocked_object.cc，二合一地来进行学习：

```cpp

/* ClockedObject继承了Clocked，如果一个仿真的对象需要SimObject的功能且希望是时钟驱动的，
    那么那个仿真的对象应该继承ClockedObject。*/
class Clocked
{

  private:
    /* 下一个时钟边沿的绝对tick值(>=curTick)*/
    /* mutable前缀说明他允许在const方法中修改这个状态*/
    mutable Tick tick;

    /* 当前周期数（与tick对应）*/
    mutable Cycles cycle;

    /* 对齐时钟边沿*/
    void
    update() const
    {
        /*已对齐，无需更新*/
        if (tick >= curTick())
            return;

        /*尝试推进一个周期*/
        tick += clockPeriod();
        ++cycle;

        /*已对齐的话返回*/
        if (tick >= curTick())
            return;

        /*如果单周期无法对齐，则一次性对齐*/
        Cycles elapsedCycles(divCeil(curTick() - tick, clockPeriod()));
        cycle += elapsedCycles;
        tick += elapsedCycles * clockPeriod();
    }

    /* 时钟域 */
    ClockDomain &clockDomain;

  protected:

    /* 创建一个clocked-object并且根据参数设置时钟域*/
    Clocked(ClockDomain &clk_domain)
        : tick(0), cycle(0), clockDomain(clk_domain)
    {
        /* 将自身注册到时钟域，当频率变化时收到回调*/
        clockDomain.registerWithClockDomain(this);
    }

    /*禁止拷贝和赋值*/
    Clocked(Clocked &) = delete;
    Clocked &operator=(Clocked &) = delete;

    /* 析构函数 */
    virtual ~Clocked() { }

    /* 在全局时钟重置时，强制将时钟对齐到当前时间 */
    void
    resetClock() const
    {
        Cycles elapsedCycles(divCeil(curTick(), clockPeriod()));
        cycle = elapsedCycles;
        tick = elapsedCycles * clockPeriod();
    }

    /* 派生类可重写此方法，响应时钟频率的变化（调整内部状态或重新调度事件）*/
    virtual void clockPeriodUpdated() {}

  public:

    /* 对齐时钟边沿、通知派生类时钟已更新，当clockDomain时钟频率变化时，用此方法*/
    void
    updateClockPeriod()
    {
        update();
        clockPeriodUpdated();
    }

    /* 计算未来周期的时钟边沿时间，计算从当前对齐后的时钟边沿开始，经过cycles个周期后的绝对时间(tick)*/
    Tick
    clockEdge(Cycles cycles=Cycles(0)) const
    {
        // align tick to the next clock edge
        update();

        // figure out when this future cycle is
        return tick + clockPeriod() * cycles;
    }

    /* 获取当前对齐后的周期数*/
    Cycles
    curCycle() const
    {  
        update();

        return cycle;
    }

    /* 获取下一周期的起始时间，下一个周期的起始时钟边沿时间*/
    Tick nextCycle() const { return clockEdge(Cycles(1)); }

    /* 时钟频率*/
    uint64_t frequency() const { return sim_clock::Frequency / clockPeriod(); }

    
    Tick clockPeriod() const { return clockDomain.clockPeriod(); }

    /* 电压*/
    double voltage() const { return clockDomain.voltage(); }

    /* 将tick转换为周期数，向上取整。*/
    Cycles
    ticksToCycles(Tick t) const
    {
        return Cycles(divCeil(t, clockPeriod()));
    }

    /* 将周期数转换为tick长度*/
    Tick cyclesToTicks(Cycles c) const { return clockPeriod() * c; }
};

```

这个类就是给你要仿真的类提供了时钟属性。

紧接着就是ClockedObject，继承了SimObject和Clocked。

```cpp

/* SimObject为GEM5所有可参数化、可序列化对象的基类，支持Python配置和检查点机制*/
/* Clocked提供时钟边沿计算、周期管理功能(如clockEdge()、curCycle())*/
class ClockedObject : public SimObject, public Clocked
{
  public:
    ClockedObject(const ClockedObjectParams &p);

    /* 参数类型别名*/
    using Params = ClockedObjectParams;
    /* 检查点相关的 序列化和反序列化*/
    void serialize(CheckpointOut &cp) const override;
    void unserialize(CheckpointIn &cp) override;
    /* 指向电源管理模块的指针，用于动态电压频率调节或功耗估计*/
    PowerState *powerState;
};

```