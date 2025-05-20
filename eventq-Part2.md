# GEM5 Minor Core 解析

这一部分讲解GEM5模拟器里面的event-queue，代码位于gem5/src/sim/eventq.hh & eventq.cc，我们合二为一，边看代码边学习

## event-queue-Part2

书接上回，我们学习了Flags类和EventBase类，现在我们来看Event这个类：
```cpp

/*Event是Event-Queue里面的一个item，我们后面要学习Event-Queue，进而了解整个仿真系统*/
/*EventBase就是Part1提到的， Serializable是支持检查点机制的，后面再说吧*/
class Event : public EventBase, public Serializable
{
    /*Event-Queue应该是可以访问Event的内部成员的，这一点很好理解*/
    friend class EventQueue;

  private:
    /*nextBin连接不同的时间桶（Bin），相同_when和_priority的事件在同一Bin中*/
    Event *nextBin;
    /*指向同一Bin内下一个事件，同一Bin内的事件按后进先出（LIFO）组织起来*/
    Event *nextInBin;
    /*这样设计可以使得插入和删除都是O(1)操作，优化仿真速度。*/
    /*插入：遍历nextBin找到合适的桶，直接插入到桶内链表的头部(nextInBin)*/
    /*删除：直接修改相邻节点的指针，无需遍历整个队列*/

    /*将事件event插入到curr事件之前*/
    static Event *insertBefore(Event *event, Event *curr)
    {
        // Either way, event will be the top element in the 'in bin' list
        // which is the pointer we need in order to look into the list, so
        // we need to insert that into the bin list.
        if (!curr || *event < *curr) {
            // Insert the event before the current list since it is in the future.
            event->nextBin = curr;
            event->nextInBin = NULL;
        } else {
            // Since we're on the correct list, we need to point to the next list
            event->nextBin = curr->nextBin;  // curr->nextBin can now become stale

            // Insert event at the top of the stack
            event->nextInBin = curr;
        }

        return event;
    };
    /*从链表中移除event事件，last是前驱节点*/
    static Event *removeItem(Event *event, Event *last)
    {
        Event *curr = top;
        Event *next = top->nextInBin;

        // if we removed the top item, we need to handle things specially
        // and just remove the top item, fixing up the next bin pointer of
        // the new top item
        if (event == top) {
            if (!next)
                return top->nextBin;
            next->nextBin = top->nextBin;
            return next;
        }

        // Since we already checked the current element, we're going to
        // keep checking event against the next element.
        while (event != next) {
            if (!next)
                panic("event not found!");

            curr = next;
            next = next->nextInBin;
        }

        // remove next from the 'in bin' list since it's what we're looking for
        curr->nextInBin = next->nextInBin;
        return top;
    }；

    /*_when为时间戳，为事件触发时间，单位为Tick*/
    Tick _when;        
    /*_priority为同一时间桶内的事件优先级，数值越小约优先*/
    Priority _priority; 
    /*管理事件状态，也就是当前Event的状态*/
    Flags flags;

#ifndef NDEBUG
    /*全局计数生成器，每一个event应该有自己的ID，整个Counter就是生成event-ID*/
    static Counter instanceCounter;

    /*event自身的ID，独一无二*/
    Counter instance;

    /*这个event属于哪个event-queue，那么这个指针就会指向那个队列*/
    EventQueue *queue;
#endif

#ifdef EVENTQ_DEBUG
    /*事件创建事件与事件调度事件，这一点比较好理解，debug用的*/
    Tick whenCreated;  
    Tick whenScheduled; 
#endif
    /*设置这个事件被调度的时间，以及这个事件属于的队列*/
    void
    setWhen(Tick when, EventQueue *q)
    {
        _when = when;
#ifndef NDEBUG
        queue = q;
#endif
#ifdef EVENTQ_DEBUG
        whenScheduled = curTick();
#endif
    }
    /*在part1有讲过事件初始化的编码机制，这里就是提供一个API来检查是否event被初始化了*/
    bool
    initialized() const
    {
        return (flags & InitMask) == Initialized;
    }

  protected:
    /*返回这个事件可以被读的状态位*/
    Flags
    getFlags() const
    {
        return flags & PublicRead;
    }
    /*检查事件的状态位是否已经被设置*/
    bool
    isFlagSet(Flags _flags) const
    {
        assert(_flags.noneSet(~PublicRead));
        return flags.isSet(_flags);
    }
    /*设置事件的状态位*/
    void
    setFlags(Flags _flags)
    {
        assert(_flags.noneSet(~PublicWrite));
        flags.set(_flags);
    }
    /*清空事件的状态位*/
    void
    clearFlags(Flags _flags)
    {
        assert(_flags.noneSet(~PublicWrite));
        flags.clear(_flags);
    }
    /*清空事件所有可以被写的状态位*/
    void
    clearFlags()
    {
        flags.clear(PublicWrite);
    }

    /*在启用TRACING_ON时，记录事件的调度、执行和取消动作*/
    virtual void trace(const char *action);   

    /*返回事件的唯一实例ID字符串，用于调试日志*/
    const std::string instanceString() const;

  protected: /* 内存管理类 */
    /* acquire()时事件被加入队列时调用，为带Managed标志的事件提供自动内存管理*/
    void acquire()
    {
        if (flags.isSet(Event::Managed))
            acquireImpl();
    };

    /*若事件不再调度，自动调用delete*/
    void release()
    {
        if (flags.isSet(Event::Managed))
            releaseImpl();
    };
    /*无操作，不过他是虚函数，可能会被重写*/
    virtual void acquireImpl(){};

    virtual void releaseImpl()
    {
        /*不被调度就删除*/
        if (!scheduled())
        delete this;
    }

  public:

    /* 构造函数*/
    Event(Priority p = Default_Pri, Flags f = 0)
        : nextBin(nullptr), nextInBin(nullptr), _when(0), _priority(p),
          flags(Initialized | f)
    {
        assert(f.noneSet(~PublicWrite));
#ifndef NDEBUG
        /*前面提到的，为event分配唯一ID*/
        instance = ++instanceCounter;
        /*初始化：不关联队列*/
        queue = NULL;
#endif
#ifdef EVENTQ_DEBUG
        /*记录创建时间*/
        whenCreated = curTick();
        /*初始的event应该是未调度的*/
        whenScheduled = 0;
#endif
    }

    virtual ~Event();
    /*返回事件名称*/
    virtual const std::string name() const;

    /*返回事件类型描述*/
    virtual const char *description() const {return "generic";};

    /*dump event data*/
    void dump() const;

  public:
    /* 这里是定义事件触发时的具体行为，可以想象到m_event->process()这样的触发场景*/
    /* 当事件队列的当前时间到达事件的_when时间戳时，事件队列会调用此方法*/
    /* 一个很好的方法就是存放函数指针，不过具体设计后面再看*/
    virtual void process() = 0;

    /* 返回事件是否被调度，通过检查事件状态位来实现*/
    bool scheduled() const { return flags.isSet(Scheduled); }

    /* 取消事件，如果事件被取消则设置标志位*/
    void squash() { flags.set(Squashed); }

    /* 返回事件是否被取消*/
    bool squashed() const { return flags.isSet(Squashed); }


    /* 返回这个事件是否是一个退出事件*/
    bool isExitEvent() const { return flags.isSet(IsExitEvent); }

    /* 返回这个事件是否是一个自动管理的事件*/
    bool isManaged() const { return flags.isSet(Managed); }

    /* 返回这个事件是否是一个自动管理（释放内存）的事件*/
    bool isAutoDelete() const { return isManaged(); }

    /* 返回事件触发的事件*/
    Tick when() const { return _when; }

    /* 返回这个事件的优先级*/
    Priority priority() const { return _priority; }

    /*默认返回NULL，这应该是旧的设计保留了下来，返回这个事件是否是一个全局事件*/
    virtual BaseGlobalEvent *globalEvent() { return NULL; }
    /*序列化和反序列化，这里是序列化到检查点，和从检查点恢复两个部分*/
    void serialize(CheckpointOut &cp) const override;
    void unserialize(CheckpointIn &cp) override;
}

```

这一部分的代码比较好理解，已经快构建起整个仿真的核心了。到这里休息一下，eventQueue放在下一个Part。