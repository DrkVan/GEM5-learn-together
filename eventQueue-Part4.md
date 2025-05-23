# GEM5源码解析

## event-queue-Part4

这一部分对EventQueue这个类进行解析。

在此之前，EventQueue里包含的多线程机制注定会是一个需要着重理解的部分，在直接看代码之前我先简单介绍一下：

1. 同步事件(Sync-Events)：同步事件是当前线程的当前事件队列中直接调度的事件。它们的插入和触发严格遵循锁的保护机制。
2. 异步事件(Async-Events)：异步事件是在其他线程或其他队列中调度的事件，通过一个中间队列(async-queue)合并到主队列。

下面我进行通俗解释，因为我觉得上面的描述可能仍然令人困惑。

首先是，同步事件：

现在开始你是一个厨师（线程），你在你的厨房（事件队列）里按照点菜的菜单（事件队列里面的事件）按顺序做菜：

你手上有唯一的厨房钥匙（锁），只有拿到钥匙的厨师才可以操作厨房里面的菜单。现在你正在炒菜（处理事件），突然发现要再加一道菜（调度进来一个新事件），你可以直接在菜单上写下这道菜（同步事件），因为钥匙在你的手里。

这里的关键就是，拥有锁的厨师才能在同一事件修改菜单，避免混乱。

然后是，异步事件：

现在餐厅的规模变大了，一个厨房难以处理巨大的任务量，所以引入了多个厨房（多个线程，这个线程是你的PC里真实的线程，不是你要模拟的多线程）。厨房收到的巨大任务量会变成一个大的菜单（主队列），从此开始就是多个厨房（真实线程）来处理这个大菜单而不是单独的你一个厨房（这样太慢）。

那么现在这么多厨房有了一个主菜单，很多个服务员可能修改这个主菜单把新菜品需求（异步事件）写到这个主菜单上，然后厨房根据主菜单来做菜。

各个厨房从主菜单里去取子菜单，并且依次开始烹饪，你要通知主菜单你烹饪完毕，需要对主菜单进行同步。那么假设每十分钟（一个周期）确认一次（同步一次），那你需要先提交你的结果，然后每十分钟确认一次，也就相当于将async_queue合入主队列。

仿真的过程中就是在仿真周期的末尾将async_queue合并到主队列，这也导致了多线程仿真的不确定性（有的厨房快，有的厨房慢）。

此时可能存在死锁问题：

如果厨房A(线程1)想修改另一个厨房B（线程2）的子菜单，直接去抢钥匙可能会导致死锁（厨房B正拿着线程2的钥匙，同时他也想要线程1的钥匙）。

这种问题的解决方法就是引入ScopedMigration类。

厨房A先放下自己的厨房钥匙（释放当前线程的锁），然后申请另一个厨房的钥匙（锁定目标线程）。

当修改完另一个厨房的菜单后，再回到自己的线程重新拿回自己的钥匙。任何时候，一个厨师只能拥有一个厨房的钥匙来避免死锁。而且线程A操作线程B可能导致模拟的不确定性（两个线程的事件顺序不确定）。

这里我们可以思考确定性和不确定性的问题：

同步事件的处理顺序完全由时间和优先级决定，无论运行多少次结果都是一样的。

跨队列操作（ScopedMigration）可能导致事件顺序依赖线程调度、每次运行结果可能不同。（线程1和线程2同时迁移到对方队列插入事件，谁先拿到锁会影响最终顺序）

然后就是从模拟周期，一个周期的视角来看这个过程是怎样的：

如果要合并异步事件的话，要有两个阶段：

1. 处理当前周期事件：主队列处理所有事件在[当前时间，当前时间+周期)范围内的事件。
2. 合并异步队列：调用handleAsyncInsertions()，将异步队列中的事件插入到主队列中。

到现在，我们开始看代码应该更好理解一些：

```cpp
class EventQueue
{
  private:
    friend void curEventQueue(EventQueue *);
    /*objName标识事件队列的名称，用于*/
    std::string objName;
    /*指向事件队列的链表头部，维护一个按事件和优先级排序的双向链表，所有事件通过head指针串联，形成事件队列*/
    Event *head;
    /*Tick为时钟类型，记录当前事件队列的模拟时间戳，事件触发事件与此值比较来决定是否执行*/
    Tick _curTick;

    /*保护异步事件队列的互斥锁，在跨线程插入异步事件时使用，确保多线程并发操作的安全性*/
    /*UncontendedMutex是低竞争场景优化的轻量级锁，假设大部分时间只有一个线程访问，当其他线程通过global schedule插入异步事件时，需要先获取此锁*/
    UncontendedMutex async_queue_mutex;

    /*异步队列，存储其他线程插入的异步事件。这个事件不会直接进入主队列(head)，而是在模拟周期结束时合并到主队列*/
    std::list<Event*> async_queue;

    /* 核心锁，保护事件队列的操作(插入、删除、处理事件)，所有对主队列的访问必须持有此锁*/
    /*跨队列操作时，通过ScopedMigration或ScopedRelease暂时释放该锁，来安全操作其他队列*/
    UncontendedMutex service_mutex;

    /*向主队列(head)插入或移除事件，私有，只能通过EventQueue的成员数调用*/
    /*锁要求：必须由操作此队列的线程调用（持有service_mutex的线程）*/
    void insert(Event *event);
    void remove(Event *event);
    /*将事件插入异步队列(async-queue)，后续合并到主队列*/
    void asyncInsert(Event *event);
    /*复制构造函数*/
    EventQueue(const EventQueue &);
     ...
}
```

总结一下，主队列(head)由service_mutex保护，处理当前线程的同步事件。

异步队列(async_queue)由async_queue_mutex保护，用来接收其他线程的事件。

合并机制：通过handleAsyncInsertions()在仿真周期的末尾合并到主队列，确保异步事件的触发事件满足_curTick+一个周期

锁的层级和职责：

1. service_mutex:保护主队列的核心锁，事件处理函数中可以安全调用schedule()（因为已持有锁），通过ScopedMigration实现跨队列操作（暂时释放当前锁，获取目标队列的锁）
2. async_queue_mutex:仅保护异步事件的插入操作，确保多线程并发调用schedule(global=true)的安全性。

线程安全与锁迁移：

1. 同步事件的调度必须由持有service_mutex的线程调用insert()，直接操作主队列。
2. 异步事件的调度用schedule(global=true)时，需获取async_queue_mutex，将事件插入async_queue。
3. 跨队列操作：使用ScopedMigration辅助类，释放当前service_mutex，获取目标队列的service_mutex，确保原子性和无死锁。

接下来是介绍ScopedMigration类，这是锁迁移的核心类。

```cpp
class EventQueue
{
     ...
    public:
    class ScopedMigration
    {
      public:
         /* 构造函数*/
        ScopedMigration(EventQueue *_new_eq, bool _doMigrate = true)
            /*new_eq为目标队列，old_eq为原队列，doMigrate为是否需要迁移*/
            :new_eq(*_new_eq), old_eq(*curEventQueue()),
             doMigrate((&new_eq != &old_eq)&&_doMigrate)
        {
            /*可以理解，doMigrate就是迁移，old_eq.unlock()为放弃旧锁*/
            /*new_eq.lock()为获取新锁*/
            if (doMigrate){
                old_eq.unlock();
                new_eq.lock();
                /*线程切换队列到new_eq*/
                curEventQueue(&new_eq);
            }
        }

        ~ScopedMigration()
        {
            /*析构的时候，如果处于迁移状态，将返回原队列*/
            if (doMigrate){
                new_eq.unlock();
                old_eq.lock();
                curEventQueue(&old_eq);
            }
        }

      private:
        EventQueue &new_eq;
        EventQueue &old_eq;
        bool doMigrate;
    };
    ...
}
```

如果看过前序，对这部分代码应该比较好理解。下面是ScopedRelease的类：

```cpp


class EventQueue
{
     ...
    /*临时释放当前事件队列的锁，以避免死锁或允许其他线程临时访问队列*/
    class ScopedRelease
    {
      public:
        /* 构造函数阶段，临时释放当前事件队列的锁，然后在析构函数阶段重新获取锁来确保线程状态安全*/
        ScopedRelease(EventQueue *_eq)
            :  eq(*_eq)
        {
            /*构造函数阶段释放锁*/
            eq.unlock();
        }

        ~ScopedRelease()
        {
            /*析构函数阶段拿回锁*/
            eq.lock();
        }

      private:
        EventQueue &eq;
    };
    ...
}

```

这里有一些关键的设计要点：
1. 避免死锁：当线程需要执行某些可能等待的操作，如果继续持有锁，其他线程可能因为无法获取锁而进入死锁状态。此时ScopedRelease临时释放锁，允许其他线程操作队列。
2. RAII(Resource Acquisition is Initialization)设计模式保证安全性，即使释放锁期间抛出异常，析构函数仍然会重新获取锁，锁的释放事件严格限定在ScopedRelease对象作用域内。
3. 锁的持有者： 调用ScopedRelease的线程必须已持有目标队列的锁。开发者需要确保ScopedRelease仅在合法持有锁的上下文中使用。

如何使用，我想是类似于此：
```cpp
{ /*当前线程是持有队列A的锁*/
    EventQueue *QA = ...;
    EventQueue::ScopedRelease release(QA); /*构造函数释放锁*/
    ...
} /*作用域结束后，通过析构函数自动重新获取队列A的锁*/
```

我们把ScopedMigration和ScopedRelease放在一起总结一下，有助于我们深刻理解：
1. ScopedRelease释放当前队列的锁，ScopedMigration是释放当前锁，获取目标队列的锁。
2. ScopedRelease是面向线程自己的队列的，允许其他线程临时操作自己的队列。ScopedMigration是安全地操作别人地队列。
3. ScopedRelease是等待外部操作或同步。ScopedMigration用于跨队列插入事件或控制权转移。

现在我们介绍完了ScopedMigration和ScopedRelease两个重要类，他们是多线程仿真的重要辅助类。

现在让我们回到EventQueue本身，继续我们的代码学习：

```cpp

class EventQueue
{
    ...
    /* objName为事件队列的名称（调试或日志），head(Null)初始化主队列链表表头为空，初始队列应该是空的*/、
    /* _curTick(0)设置当前模拟事件为0（起始时刻）*/
    EventQueue(const std::string &n)
    : objName(n), head(NULL), _curTick(0)
    {
        /*创建一个新的事件队列准备接收事件，objName用于区分不同队列（如不同的CPU或device）*/
    };

    /* 这两个应该都好理解，一个是返回事件队列的名字，一个是设置事件队列的名字*/
    virtual const std::string name() const { return objName; }
    void name(const std::string &st) { objName = st; }

    /* schedule方法，Event是事件，when就是何时调度，global=false说明默认是单线程仿真，global=true就会涉及到上文提到的多线程仿真机制。*/
    void
    schedule(Event *event, Tick when, bool global=false)
    {   
        /* 如果when比当前时刻还要小，说明设计中存在错误*/
        assert(when >= getCurTick());
        /* 一个event的状态码应该是未被调度，一个被调度了的event再次被调度说明设计中存在错误*/
        assert(!event->scheduled());
        /* 一个event应该是初始化好的，这部分如果遗忘可以参阅Part1的Flags部分的代码解析*/
        assert(event->initialized());

        /*event 设置when，将事件队列的指针传进去，每个event实例有一个_when实例说明自己什么时刻应该被触发*/
        event->setWhen(when, this);

        /*软件多线程仿真的必要操作，inParallelMode表示并行仿真，global为异步插入事件的标志*/
        if (inParallelMode && (this != curEventQueue() || global)) {
            /*异步插入event*/
            asyncInsert(event);
            /*aysncInsert的代码：
                async_queue_mutex.lock();      获得异步队列的锁
                async_queue.push_back(event);  对异步队列插入event
                async_queue_mutex.unlock();    释放异步队列的锁
            */
        } else {
            /*同步插入event*/
            insert(event);
            /*insert的代码：
                if (!head || *event <= *head) {     回忆一下event重载的操作符<=，比较触发时间，然后比较优先级
                    head = Event::insertBefore(event, head);  如果队列为空，或新事件触发时间早于头节点，或相同时间但优先级更高，则新事件插入head之前，并更新head指针
                    return;
                }

                Event *prev = head;         如果是非空的队列，获得head的指针 
                Event *curr = head->nextBin;   curr指向head的下一个时间桶(时间槽)
                while (curr && *curr < *event) {   移动prev和curr直到找到第一个时间>=新事件的Bin或者链表的末尾
                    prev = curr;
                    curr = curr->nextBin;
                }

                prev->nextBin = Event::insertBefore(event, curr);  插入新事件并更新链表，将新事件插入curr所在Bin的前面，返回新事件的节点
            */
        }
        /*
            这里还是过于抽象了，我们来结合实例理解一下这个过程：
            假设事件队列当前结构如下(按事件升序排列)：
                head->[t=100,pri=1]->nextBin->[t=200,pri=2]->nextBin->NULL
            现在我们要插入新事件[t=150,pri=3]:
                head的时间(100) < 新事件时间(150)， 进入遍历逻辑
                curr初始指向t=200,pri=2.
                curr时间(200) > 新事件时间(150)，终止循环。
            插入新事件：
                新事件插入到prev(head)和curr(t=200)之间
            现在事件队列变成了：
                head->[t=100,pri=1]->nextBin->[t=150,pri=3]->nextBin->[t=200,pri=2]->NULL
            这样会好理解多了对吧
        */

        /* 设置event的状态码，此event已被调度*/
        event->flags.set(Event::Scheduled);
        /* 管理Event的声明周期，检查Event的Managed是否被设置，如果Managed，则实现acquireImpl()实现具体的管理逻辑*/
        event->acquire();

        if (debug::Event)
            event->trace("scheduled");
    }

    /* 从事件队列中移除指定事件，并更新事件状态和引用计数*/
    void
    deschedule(Event *event)
    {
        /*event应该是被调度过的*/
        assert(event->scheduled());
        /*event应该是被初始化过的*/
        assert(event->initialized());
        /*并行仿真模式下必须由所属线程调用*/
        assert(!inParallelMode || this == curEventQueue());

        /*从队列中移除事件*/
        remove(event);
        /*
        我们来看看remove的代码：
        {
            if (head == NULL)
                panic("event not found!");

            assert(event->queue == this);

            // deal with an event on the head's 'in bin' list (event has the same
            // time as the head)
            if (*head == *event) {
                head = Event::removeItem(event, head);
                return;
            }

            // Find the 'in bin' list that this event belongs on
            Event *prev = head;
            Event *curr = head->nextBin;
            while (curr && *curr < *event) {
                prev = curr;
                curr = curr->nextBin;
            }

            if (!curr || *curr != *event)
                panic("event not found!");

            // curr points to the top item of the the correct 'in bin' list, when
            // we remove an item, it returns the new top item (which may be
            // unchanged)
            prev->nextBin = Event::removeItem(event, curr);
        }
        */
        /*清除“被取消”标志*/
        event->flags.clear(Event::Squashed);
        /*设置事件为“未调度”*/
        event->flags.clear(Event::Scheduled);

        if (debug::Event)
            event->trace("descheduled");
        /*释放event*/
        event->release();
        /*
        {
            如果被设置了自动生命周期管理，那么调用releaseImpl来释放
            if (flags.isSet(Event::Managed))
                releaseImpl();
        }
        */
    }

    /* 重新调度事件，调整触发事件，若事件未调度且always=true，则强制调度*/
    void
    reschedule(Event *event, Tick when, bool always=false)
    {
        /*时间检查，被调度初始化检查，线程安全检查*/
        assert(when >= getCurTick());
        assert(always || event->scheduled());
        assert(event->initialized());
        assert(!inParallelMode || this == curEventQueue());
        /*若事件已经被调度，从队列中移除*/
        if (event->scheduled()) {
            remove(event);
        } else {
            /*调用生命周期管理方法*/
            event->acquire();
        }
        /*设置event的触发时间，并且给他事件队列*/
        event->setWhen(when, this);
        /*同步插入事件*/
        insert(event);
        /*从被取消状态变为被调度状态*/
        event->flags.clear(Event::Squashed);
        event->flags.set(Event::Scheduled);

        if (debug::Event)
            event->trace("rescheduled");
    }
    /*返回队列中下一个即将处理事件的触发时间(head->when())，使用前应该确保队列非空*/
    Tick nextTick() const { return head->when(); }
    /*直接设置当前事件队列的模拟时间_curTick，用于初始化或时间同步，手动设置的newVal应该大于原时间，不然会出错*/
    void setCurTick(Tick newVal) { _curTick = newVal; }

    /* 返回当前事件队列的模拟时间*/
    Tick getCurTick() const { return _curTick; }
    /* 返回队列头事件指针head*/
    Event *getHead() const { return head; }
    /* 处理队列中的下一个事件(头事件)，推进模拟事件*/
    Event *serviceOne()
    {
        /*线程安全锁定，锁定当前队列的service_mutex，确保在事件处理期间队列不会被其他线程修改*/
        std::lock_guard<EventQueue> lock(*this);
        /*去除队列头事件*/
        Event *event = head;
        Event *next = head->nextInBin;
        event->flags.clear(Event::Scheduled);

        if (next) {
            /*next为时间桶内下一个event，将next指向的下一个时间桶传递给head，nextInBin是同一个时间桶内，nextBin为下一个时间桶*/
            next->nextBin = head->nextBin;
            head = next;
        } else {
            /*时间桶内就head这一个event，跳到下一时间桶*/
            head = head->nextBin;
        }

        if (!event->squashed()) {
            /*设置事件队列当前的时钟*/
            setCurTick(event->when());
            if (debug::Event)
                event->trace("executed");
            /*触发event里面的方法*/
            event->process();
            /*处理退出事件（如模拟终止）*/
            if (event->isExitEvent()) {
                /*断言确保退出事件未被Managed或事件属于非主队列（避免资源泄露）*/
                assert(!event->flags.isSet(Event::Managed) ||
                    !event->flags.isSet(Event::IsMainQueue)); // would be silly
                return event;/*返回退出事件*/
            }
        } else {
            /*事件被取消，那么此时应该清除取消标志*/
            event->flags.clear(Event::Squashed);
        }
        /*触发Managed里面的托管方法*/
        event->release();
        return NULL;
    };

    /* empty意味着在当下时间桶里，时间桶里的事件已经清空，如果没清空就一直serviceOne()*/
    void
    serviceEvents(Tick when)
    {
        while (!empty()) {
            if (nextTick() > when)
                break;
            
            serviceOne();
        }
        /*设置事件队列的时间*/
        setCurTick(when);
    }

    /*如果队列里没事件了，则empty为true*/
    bool empty() const { return head == NULL; }

    /* 打印事件队列里面的全部事件*/
    void dump() const;

    bool debugVerify() const
    {
        std::unordered_map<long, bool> map;

        Tick time = 0;
        short priority = 0;

        Event *nextBin = head;
        while (nextBin) {
            Event *nextInBin = nextBin;
            while (nextInBin) {
                if (nextInBin->when() < time) {
                    cprintf("time goes backwards!");
                    nextInBin->dump();
                    return false;
                } else if (nextInBin->when() == time &&
                        nextInBin->priority() < priority) {
                    cprintf("priority inverted!");
                    nextInBin->dump();
                    return false;
                }

                if (map[reinterpret_cast<long>(nextInBin)]) {
                    cprintf("Node already seen");
                    nextInBin->dump();
                    return false;
                }
                map[reinterpret_cast<long>(nextInBin)] = true;

                time = nextInBin->when();
                priority = nextInBin->priority();

                nextInBin = nextInBin->nextInBin;
            }

            nextBin = nextBin->nextBin;
        }

        return true;
    };

    /* 将异步队列中的事件同步到主队列(head)，确保跨线程调度的事件按正确顺序插入主队列*/
    void handleAsyncInsertions()
    {
        /*确保当前线程持有队列锁*/
        assert(this == curEventQueue());
        async_queue_mutex.lock();

        /*将异步队列事件插入主队列，并且移除异步队列的事件*/
        while (!async_queue.empty()) {
            insert(async_queue.front());
            async_queue.pop_front();
        }
        /*释放锁*/
        async_queue_mutex.unlock();
    };

    /* 用于通知事件队列需要唤醒，比如外部事件插入，派生类里可覆盖*/
    /* (Tick)-1表示立即唤醒*/
    virtual void wakeup(Tick when = (Tick)-1) { }

    /* 替换事件队列的头节点(head)，返回旧节点。用于临时切换事件队列*/
    /* 使用时应该确保新头节点>= _curTick*/
    Event* replaceHead(Event* s)
    {
        Event* t = head;
        head = s;
        return t;
    };

    /* lock和unlock操作*/
    void lock() { service_mutex.lock(); }
    void unlock() { service_mutex.unlock(); }

    /* 从检查点恢复后，重新调度事件到队列中，用于反序列化过程中恢复事件调度的状态。*/
    void checkpointReschedule(Event *event)
    {
        /*如果事件在序列化前已被调度，则重新插入队列*/
        if (event->flags.isSet(Event::Scheduled))
            insert(event);
    };

    virtual ~EventQueue()
    {
        while (!empty())
            deschedule(getHead());
    }
};

```

到现在我们完成了EventQueue的学习，但是还有一些剩余的内容，我会放在Part5，小憩一下。

这一部分可以把Bin和InBin那里的结构参考链表代码多想一下，是个很巧妙的设计。