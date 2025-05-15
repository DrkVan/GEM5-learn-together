# GEM5 Minor Core 解析

本人以Minor Core为模板来学习Minor Core，并进而学习GEM5仿真系统，以此为记录。
以流水线Front-End到Back-End为顺序来进行解析，如有错误，敬请指正。

行文以一个初学者第一视角开始，希望给读者带来一个不错的体验。

## Fetch1

此部分主要包含两部分代码，fetch1.cc 和 fetch1.hh，其中涉及到的系统组件我将会出一个GEM5源码解析来讲述，此处只关注基于系统组件来构建仿真核的这一部分。

解析也会按照官方代码里从上往下进行解析，如有需要跳转的部分我会说明。

### Fetch1.hh

上一个Part我们解析了ICachePort这部分，我们追根溯源地解析了Port类的继承与实现，我们继续推进我们在Fetch1.hh的代码。下一部分是成员类FetchRequest，长代码我会把内容放在代码注释中，我们又要开始追根溯源了，不过都是一次性开销。

```cpp
class FetchRequest :
        public BaseMMU::Translation, /* For TLB lookups */
        public Packet::SenderState /* For packing into a Packet */
    {
      protected:
        /*很好理解，记录自己的更高层级也就是Fetch1，得到他的指针*/
        Fetch1 &fetch;

      public:
        /* 枚举：跟踪一个内存请求（获取指令）在地址翻译和内存访问过程中的进度状态，用于Fetch和MMU处理请求*/
        enum FetchRequestState
        {
	    /* 请求刚刚创立，并没有开始处理。场景：对象被初始化，但还没有提交到TLB（地址翻译）或内存 */
            NotIssued, 
	    /* 提交到TLB，等待翻译完成。场景：TLB未命中，此时会触发页表遍历或等待翻译结果 */
            InTranslation, 
	    /* 翻译完成，物理地址已就绪。场景：TLB命中或页表遍历完成，获取物理地址，准备访问内存*/
            Translated, 
	    /* 已经发给MSS(内存子系统)，如缓存和内存控制器，等待响应。场景：物理地址有效，但是缓存未命中
		  	或需要访问主存，等待数据返回 */
            RequestIssuing, 
	    /* 处理完成（可以是成功获取数据，也可以是因为异常或错误而终止。场景：数据返回或发生错误） */
            Complete 
        };
	/*根据刚才的枚举类型，构造一个state，用于记录这个Request在Fetch系统里所处的状态*/
        FetchRequestState state;
	/*到这里我们可以窥见对于请求的设计思考，我们对于一个请求的生命线要全局记录，无论是出于扩展性的
		考虑还是出于项目开发的考虑这样做都很有利于debug和分析dataflow*/
        /*Request的ID，用于标识，至于他的类InstId，先不解析，你可以理解为他就是一个id*/
        InstId id;
	/* 维护自身packet的引用，因为packet可能在MSS（内存子系统）里被修改，比如请求合并（多个请求合并为
		单个数据包），请求拆分（大块请求拆分为多个子数据包，比如缓存行非对齐访问），错误处理（数据包被替换为
		错误响应），缓存一致性操作（数据包因snooping或writeback被修改），这些改动都可以通过packet这个引用被
		及时的同步给packet，确保后续处理是正确的。Packet这个类的解析也是个大活，后面解析*/
	PacketPtr packet;
	/* 这一个指向底层内存请求指针，Request是内存里的一个类，后面解析，这个指针的核心作用是维护取值
		请求（Fetch Request）于内存子系统之间的原始请求信息，确保整个生命周期中能够访问关键的地址、权限、
		上下文等元数据 */
	/*到这里你可能会感到困惑，为什么有了packet还要有一个request，这是因为一个request会产生多个packet
		（如缓存行拆分），但request始终维护着最初的请求实例*/
	RequestPtr request;
	/*Addr就是uint64_t，之所以typedef重定义一下就是为了方便后续更改架构，比如你要将所有的Addr设置为uint32_t
		的话你只需要更改typedef uint32_t Addr即可*/
	Addr pc;
	/*这里的作用是记录取值过程中发生的错误，并通过MSS返回的Packet来说明具体的错误信息，比如Page Fault*/
	Fault fault;
	/*这部分是makePacket，讲述如何构造一个数据包*/
void makePacket()
{
    /* packet就是刚才提到的PacketPtr，这个指针构造一个Packet，将request指针传入，并且默认指定为Read请求，因为
	取指令只能是Read请求。
	至于allocate操作，他就是为数据包Packet动态分配内存空间用于存储数据，就是确保在传输数据时（内存读/写请求或
	响应），确保数据包有足够的空间来保存数据 */
    packet = new Packet(request, MemCmd::ReadReq);
    packet->allocate();

    /* 将当前FetchRequest作为发送者状态附加到Packet中，以便在后续处理（响应返回）能够识别请求的上下文并且恢复
	相关状态，这个状态机的机制后续解析，这个也能单开一节 。这里给一个简述：
	当Packet被发送给MSS时，FetchRequest将其自身附加到Packet的状态栈。当响应返回时，通过栈顶FetchRequest指针
	恢复请求上下文，进而继续处理（数据传给CPU）*/
    /*FetchRequest的生命周期覆盖Packet的整个传输过程（从发送到响应返回），这里涉及到的共享状态是一个非常巧妙的
	设计，后面可以单开一节讲述（这里不知道多少个'单开一节'了）*/
    packet->pushSenderState(this);
}
/* 打印这个ID下的数据，这个是在InstId里有记录，他是如何打印的 */
void reportData(std::ostream &os) const
{
    os << id;
}
/* 这个函数用于判断一个取值请求(FetchRequest)是否可被安全丢弃，核心逻辑就是判断是否处于关键操作（地址翻译或内存
	访问），判断是否序列号匹配，也就是请求的流序列号(streamSeqNum)或预测序列号(PredictionSeqNum)是否已过期*/
bool isDiscardable() const
{
    /*根据所属的线程ID，获得线程状态，这里是支持多线程模拟的设计*/
    /*fetchInfo是Fetch1维护的线程状态数组，每个线程对应一个Fetch1ThreadInfo*/
    /*获取当前请求所在线程的最新状态（当前流序列号和预测序列号）*/
    Fetch1ThreadInfo &thread = fetch.fetchInfo[id.threadId];
    /* 如果这个FetchRequest正处于翻译，或者正处于内存访问的状态时不可丢弃 */
    /* 如果流水线已刷新生成新序列号，或者分支预测错误后生成了新序列号，则旧请求无效，需清理*/
    return state != InTranslation && state != RequestIssuing &&
        (id.streamSeqNum != thread.streamSeqNum ||
        id.predictionSeqNum != thread.predictionSeqNum);
}
/* 获取这个Request的状态，是否已完成*/
bool isComplete() const {return state == Complete;}
/* 通知当前地址翻译操作需要延迟处理（如TLB缺失后需要等待页表遍历）*/
/* 为什么是一个空实现？是因为这个情况已经可以被处理，Fetch已经内置延迟处理机制，故无需额外处理*/
/* ITLB未命中时，需要启动页表遍历，需要告诉系统流水线挺短，直到翻译完成*/
/* 这里给了一个空实现，是因为要满足BaseMMU::Translation接口的形式要求，是一种重载*/
void markDelayed() {}
/* 这个方法用于处理地址翻译的完成，更新取值请求的状态为finish，触发后续操作（内存访问/异常处理）*/
/* 唤醒流水线继续工作，核心作用是将虚拟地址翻译后的结果传递给下一阶段*/
void finish(const Fault &fault_, const RequestPtr &request_,ThreadContext *tc, BaseMMU::Mode mode)
{
    /* 传入的异常对象，可能为NoFault也可以是具体的异常*/
    fault = fault_;
    /* 取指请求的状态机标识（他是Request状态的一种，是之前提到的枚举类型的一部分）*/
    state = Translated;
    /* 调用fetch模块对request done的处理*/
    fetch.handleTLBResponse(this);
    /* CPU的方法，用于事件驱动模拟器中通知流水线在下一个周期恢复活动，如果流水线因等待地址翻译而暂停，恢复*/
    fetch.cpu.wakeupOnEvent(Pipeline::Fetch1StageId);
}
/*构造函数，初始化状态，并且实例化一个request的共享指针*/
        FetchRequest(Fetch1 &fetch_, InstId id_, Addr pc_) :
            SenderState(),
            fetch(fetch_),
            state(NotIssued),
            id(id_),
            packet(NULL),
            request(),
            pc(pc_),
            fault(NoFault)
        {
            request = std::make_shared<Request>();
        }
```

到这里我们解析完了FetchRequest这个成员类的代码，但是其中所包含的函数和机制我们还没解析，因为用中断的方式去解析这些机制太过于分散注意力，所以现在我开始讲述这些机制或函数。

首先是继承，我们还没有讲解继承：

```cpp
class FetchRequest :
        public BaseMMU::Translation, 
        public Packet::SenderState
```

我们先把目光放在BaseMMU::Translation，我们进入这一部分：

```cpp
class Translation
    {
      public:
        virtual ~Translation()
        {}

        /* 之前我们提到了一个空函数就是markDelayed，这里因为标注为了纯虚函数，所以需要一个空实现*/
	/* 由于遍历页表等需要事件的操作，显式地告知需要延时*/
        virtual void markDelayed() = 0;
        /*  也是一个纯虚函数，要求派生类在实例化的时候提供一个finish的实现，这是为了流水线在收到finish状态的时候
	可以继续进行处理，这是Request的生命周期终点*/
        virtual void finish(const Fault &fault, const RequestPtr &req,
                            ThreadContext *tc, BaseMMU::Mode mode) = 0;

        /* 判断一个内存访问请求是否已经被废弃，供页表遍历器在地址翻译过程中进行检查，如果i请求被废弃，页表遍历
	器可以终止当前翻译操作，避免处理无效请求，这个函数有一个默认实现就是return false，可以被派生类重载*/
        virtual bool squashed() const { return false; }
    };
```

BaseMMU::Translation部分的解析已经结束，现在我们把目光放在SenderState：

```cpp
struct SenderState
    {
	/*指向栈中前一个SenderState对象的指针，构成单向链表结构*/
        /* 当数据包经过多个模块处理时，每个模块可添加自己的状态到栈顶，同时保留之前的状态链*/
	/* 响应返回时，模块”后进先出“地弹出自己地状态，恢复前驱状态，确保上下文一致性*/
        SenderState* predecessor;
        SenderState() : predecessor(NULL) {}
        virtual ~SenderState() {}
    };
```

这里虽然SenderState的代码简单，但是背后的设计思想并不简单，我们来想象一个场景，L1Cache发送请求，L2Cache接收请求并发给内存控制器，内存控制器接收请求并返回响应，L2Cache处理响应，并且L1Cache处理响应。这是最长的Request路径，这个Request路径可能在L1Cache，L2Cache等任意阶段结束，在Request向前回滚的时候需要恢复状态，那么通过这个单向链条就可以实现状态恢复。
你可能要问为什么要这样设计呢？Slave向Master发送响应Master自己处理不就完事了吗？我认为这样设计是为了做上下文隔离，模块A处理完数据包后能正确恢复模块B的状态，而模块B无需感知模块A的存在。其实我写此文的时候理解也不深刻，可能后续会想的更明白。
至少目前我们完成了对FetchRequest的解析，我们明白了Fetch1模块是如何构造一个FetchRequest，并且这个Request会在系统里大概做怎样的事情，以及初步的状态管理机制，上下文恢复机制等。
到这里也休息一下吧，下一节再继续。



