# GEM5源码解析

## event-queue-Part5

到这里eventq.hh和eventq.cc的代码快学习完了，仿真最核心的地方基本已经掌握了。

上一Part的要点就是仿真链表和多线程并行仿真机制。剩下的部分会比较简单：

```cpp
class EventQueue

{
    ...

    /*全局函数curEventQueue*/
    inline void
    curEventQueue(EventQueue *q)
    {   
        /*设置全局当前正在仿真事件队列*/
        _curEventQueue = q;
        /*更新全局的当前时间戳*/
        Gem5Internal::_curTickPtr = (q == nullptr) ? nullptr : &q->_curTick;
        /*这些都是线程本地变量，每个线程管理自己的事件队列*/
    }

    void dumpMainQueue();

    class EventManager
    {
    protected:
        /* 指向关联的事件队列对象，所有操作通过这个指针委托给底层队列 */
        EventQueue *eventq;

    public:
        /* 通过不同的方式来初始化eventq*/
        EventManager(EventManager &em) : eventq(em.eventq) {}
        EventManager(EventManager *em) : eventq(em->eventq) {}
        EventManager(EventQueue *eq) : eventq(eq) {}

        /* 返回关联的事件队列指针*/
        EventQueue *
        eventQueue() const
        {
            return eventq;
        }

        /* 透传给自己关联的eventq，以及时间*/
        void
        schedule(Event &event, Tick when)
        {
            eventq->schedule(&event, when);
        }

        /* 解除调度*/
        void
        deschedule(Event &event)
        {
            eventq->deschedule(&event);
        }

        /* 透传，重调度*/
        void
        reschedule(Event &event, Tick when, bool always = false)
        {
            eventq->reschedule(&event, when, always);
        }

        /* 通过指针的方式调度*/
        void
        schedule(Event *event, Tick when)
        {
            eventq->schedule(event, when);
        }

        void
        deschedule(Event *event)
        {
            eventq->deschedule(event);
        }

        void
        reschedule(Event *event, Tick when, bool always = false)
        {
            eventq->reschedule(event, when, always);
        }

        /*触发事件队列唤醒，可能用于集成外部调度器时会用到这个方法来通知队列处理待执行事件*/
        void wakeupEventQueue(Tick when = (Tick)-1)
        {
            eventq->wakeup(when);
        }

        /*设置关联队列的当前时间戳*/
        void setCurTick(Tick newVal) { eventq->setCurTick(newVal); }
    };
    ...
}

```

我在学习到这个地方的时候感觉这个EventManager和EventQueue好像大同小异，至于为什么要封装这一层我的想法是这样的：
1. 职责分离：EventQueue的核心职责是管理事件队列的数据结构和调度算法，而EventManager负责提供对外的调度接口，隐藏队列的复杂性，不然用户得直接操作队列指针。
2. 拓展性的考虑：EventManager可以持有不同类型的EventQueue派生类，对于调用方无需修改EventQueue代码，还是如以往地使用EventManager。
3. 设计模式：这种设计模式应该叫“外观模式”，通过简化的接口封装复杂子系统。
4. 设计模式：“代理模式”，EventManager作为代理，调用者可以在后期添加额外逻辑，然后再转发请求给EventQueue。

OK，我们回到代码本身，继续学习：

```cpp

/*模板参数F*/
template <auto F>
class MemberEventWrapper final: public Event, public Named
{
    /*CLASS类型推导，MemberFunctionClass_t推导成员函数所属的类*/
    using CLASS = MemberFunctionClass_t<F>;
    /*返回类型检查，确保成员函数返回void（事件处理函数通常无返回值）*/
    static_assert(std::is_same_v<void, MemberFunctionReturn_t<F>>);
    /*确保成员函数没有参数（事件处理函数无需外部参数）*/
    static_assert(std::is_same_v<MemberFunctionArgsTuple_t<F>, std::tuple<>>);

public:
    /*告诉编译器，这是被弃用的版本，用了会报warning*/
    [[deprecated("Use reference version of this constructor instead")]]
    MemberEventWrapper(CLASS *object,
                       bool del = false,
                       Priority p = Default_Pri):
        MemberEventWrapper{*object, del, p}
    {}

    /* Event(p)设置事件优先级，mObject存储对象引用，del为true时设置AutoDelete标志，事件执行后自动销毁*/
    MemberEventWrapper(CLASS &object,
                       bool del = false,
                       Priority p = Default_Pri):
        Event(p),
        Named(object.name() + ".wrapped_event"),
        mObject(&object)
    {
        if (del) setFlags(AutoDelete);
        gem5_assert(mObject);
    }

    /*使用成员函数指针F调用对象mObject的成员函数，(mObject->*F)() 时C++成员函数指针的调用方式*/
    void process() override {
        (mObject->*F)();
    }

    const char *description() const override { return "EventWrapped"; }
private:
    CLASS *mObject;
};

```

到这里，我们可以思考如何调度Event呢？

可以联想到这样的情况：

```cpp

class Module {
public:
    void process() { /* 处理逻辑 */ }
    std::string name() const { return "ModuleName"; }
};

MyDevice device;
/* 创建包装器，自动删除，优先级默认 */
/* 把Module里面的process函数指针传给eventWrapper，设置Managed*/
auto *eventWrapper = new MemberEventWrapper<&Module::process>(device, true);
eventWrapper->schedule(100); /*在eventQueue里面设置调度顺序和仿真的事件*/
```

我们的仿真内核已经快搞定了。

代码后面还提供了EventFunctionWrapper类的实现，但是已经被弃用了，我们可以看一看，也是一种通过函数指针来实现调度的方式。

还有两个宏

```cpp

/* 在GEM5的序列化(checkpoint)机制中，SERIALIZE_EVENT和UNSERIALIZE_EVENT宏用于对简化事件对象的状态保存和恢复*/
/* 将事件对象的当前状态（触发事件、优先级、是否已调度等）保存到检查点，cp为检查点对象，#event表示将变量名event转换为字符串*/
#define SERIALIZE_EVENT(event) event.serializeSection(cp, #event);
/* 底层原理，在Checkpoint中创建一个以#event命名的子段*/
/* 调用事件的serialize方法，将成员变量(如_when、flags)写入检查点*/

/* 反序列化事件状态,从检查点读取事件的状态并恢复到对象event中，使用checkpointReschedule()来把事件重新插入事件队列*/
#define UNSERIALIZE_EVENT(event)                        \
    do {                                                \
        event.unserializeSection(cp, #event);           \
        eventQueue()->checkpointReschedule(&event);     \
    } while (0)

```

到这里，一个非常关键非常核心的仿真内核代码解析完了，从Flags到MemberEventWrapper，可以学习到不少的代码设计思路与优秀的代码风格。