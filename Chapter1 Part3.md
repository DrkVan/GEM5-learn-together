# GEM5 Minor Core 解析

本人以Minor Core为模板来学习Minor Core，并进而学习GEM5仿真系统，以此为记录。
以流水线Front-End到Back-End为顺序来进行解析，如有错误，敬请指正。

行文以一个初学者第一视角开始，希望给读者带来一个不错的体验。

## Fetch1

此部分主要包含两部分代码，fetch1.cc 和 fetch1.hh，其中涉及到的系统组件我将会出一个GEM5源码解析来讲述，此处只关注基于系统组件来构建仿真核的这一部分。

解析也会按照官方代码里从上往下进行解析，如有需要跳转的部分我会说明。

### Fetch1.hh

上一节我们解析了FetchRequest部分，并且较为简单的解读了其中包含的一些机制。
我们继续代码的剩余部分

```cpp
class Fetch1 : public Named
{
	...
  protected:
    /* 指向所属的MinorCPU的引用，用于访问CPU的全局状态或触发事件 */
    MinorCPU &cpu;

    /* 流水线通信端口，Latch<T>你可以认为是一种DelayFifo，就是可以仿真延时的fifo*/
    /* 具体机制我们后面再解析，现在你先认为是一种Fifo，不过他可以仿真延时*/
    /* inp是接收来自Execute阶段的分支结果（如分支指令实际跳转地址），用于纠正预测错误*/
    Latch<BranchData>::Output inp;
    /* 向Fetch2阶段发送已获取的指令数据块(ForwardLineData)，用于解码和预测*/
    Latch<ForwardLineData>::Input out;
    /* prediction用于接收来自Fetch2阶段的分支预测结果（分支目标地址），指导后续取指方向*/
    Latch<BranchData>::Output prediction;

    /* 管理对Fetch2阶段输入缓冲区的预留空间 */
    /* 通过预留机制确保Fetch1不会向Fetch2发送超过其处理能力的数据，避免缓冲区溢出*/
    std::vector<InputBuffer<ForwardLineData>> &nextStageReserve;

    /* 本解析Part1提到的ICachePort，这里实例化了一个icachePort，负责CPU与ICache交互 */
    IcachePort icachePort;

    /* lineSnap是指令块的对齐边界（缓存行大小），避免跨行访问*/
    Addr lineSnap;

    /* 单词取指的最大字节数，如maxLineWidth=32，则最多每次取32字节（即使缓存行大小为64Byte）*/
    /* 这样做是为了带宽控制，限制取值宽度以适应流水线处理能力（比如每周期处理四条指令）*/
    Addr maxLineWidth;

    /* 限制未完成取值请求的最大数量（包括在队列中等待或者正在内存子系统中处理的请求）*/
    /*  防止资源耗尽：避免因缓存未命中导致过多未完成请求堆积，耗尽MSHR(Miss Status Handling Register)*/
    /* 流量控制：当达到上限时，Fetch1暂停发送新请求，直到部分请求完成*/
    unsigned int fetchLimit;
	...
}
```

到这里我们了解了Fetch1所包含的一些参数，这一部分应该很好理解，我们继续

```cpp
protected:
    /* FetchState枚举了状态机，目标是管理取指操作再不同场景下的启停逻辑（流水线阻塞，分支预测切换，异常处理） */
    enum FetchState
    {
	/* 取指暂停，不发起新请求，也不处理地址更新*/
	/*  收到唤醒信号（如中断返回，调试继续等），切换到FetchWaitingForPC*/
        FetchHalted, 
	/*  等待新PC，不主动发起新的取指请求，而已经发出的取指请求可继续等待响应*/
        /*  场景：流水线刷新：分支预测错误后，等待新PC。或进程切换时，等待新地址空间的PC，或等待TLB提供PC*/
        FetchWaitingForPC,
	/*  取指进行中，根据当前PC持续发起取指请求，接收Cache/Memory返回的指令数据，传给下一阶段。*/
	/*  无异常或阻塞时的默认状态，从FetchWaitingForPC收到有效PC后切换过来激活。*/
        FetchRunning 
    };
```

我们总结一下FetchState这个状态机，
启动时处于FetchHalted状态，当唤醒/中断返回时进入FetchWaitingForPC状态。
在FetchWaitingForPC状态等待有效PC值，如果此时外部强制暂停就会回到FetchHalted状态。
在FetchWaitingForPC状态时如果收到了有效PC值，或分支触发时会进入FetchRunning状态。
在FetchRunning状态时，如果遇到了流水线阻塞/资源受限会回到FetchWaitingForPC状态。
在FetchRunning状态时，如果遇到了严重异常（不可恢复的错误）就会跳转到FetchHalted状态。
我们继续，来到Fetch1ThreadInfo这个结构体：

Fetch1ThreadInfo结构体封装了每个线程在取指阶段(Fetch1)的周期级状态，包括PC，取指地址，序列号，阻塞状态等。核心目标是实现多线程取指的独立管理，确保各线程的状态隔离和正确流转。

```cpp
struct Fetch1ThreadInfo
    {
        /*  默认构造函数，常见于在以此为模板的容器初始化时调用*/
        Fetch1ThreadInfo() {}
        /*  拷贝构造*/
        Fetch1ThreadInfo(const Fetch1ThreadInfo& other) :
            state(other.state),
            pc(other.pc->clone()),
            streamSeqNum(other.streamSeqNum),
            predictionSeqNum(other.predictionSeqNum),
            blocked(other.blocked)
        { }
	/*  记录线程的取指状态机，当前线程默认为FetchWaitingForPC状态*/
        FetchState state = FetchWaitingForPC;
        /*  保存线程当前的PC状态，PCStateBase是一个结构体，记录了虚拟地址、执行模式和权限*/
        std::unique_ptr<PCStateBase> pc;
        /*   当前正在取指的内存地址 */
        Addr fetchAddr = 0;
        /*  指令流序列号，标记当前线程所属的取指流，当分支预测错误或异常时递增，用于标记新指令流*/
	/*   在Part2时有用到这个变量，isDiscardable()那个函数，其作用用于过滤掉属于旧流的指令*/
        InstSeqNum streamSeqNum = InstId::firstStreamSeqNum;
        /*  预测流序列号，表示线程当前的分支预测上下文。由Fetch2发起，当分支预测结果更新时递增*/
	/*  确保取指与最新预测一致，旧预测的请求可以被安全丢弃*/
        InstSeqNum predictionSeqNum = InstId::firstPredictionSeqNum;
        /*  标记线程是否被阻塞（用于状态报告和调试），如果下游满了会阻塞住，或者因为资源限制*/
        bool blocked = false;
        /*  唤醒保护标志，防止线程在唤醒的第一周期错误地进入睡眠，确保线程被唤醒后起码执行一周期再休眠*/
        bool wakeupGuard = false;
    };
```

总结一下，Fetch1ThreadInfo是服务于多线程运行的，它保存了每个线程的状态。

假设线程A和B的state均为FetchWaitingForPC等待初始PC，此时操作系统调度线程A，并且设置PC入口为0x8000_0000，状态切换到FetchRunning.

Fetch2预测线程A的分支跳转到0x9000_0000，递增predictionSeqNum，Fetch1根据新的predictionSeqNum调整fetchAddr，继续取指。

假如此时时钟中断触发，CPU切换到线程B，此时线程A的state设置为FetchHalted，blocked标记为true。线程B恢复PC和fetchAddr，状态切换为FetchRunning。

还有的情景就是：线程B的未完成请求达到fetchLimit，此时blocked设置为true，暂停取指。直到缓存响应到达后再释放槽位，blocked清除，恢复取指。

```cpp
std::vector<Fetch1ThreadInfo> fetchInfo;
ThreadID threadPriority;
```

紧接着是这两行代码，首先是用一个数组fetchInfo来存储线程信息Fetch1ThreadInfo。
然后是表示当前应该优先处理的线程ID，用于再多个可运行线程间仲裁取指带宽。比如threadPriority=2则优先处理线程2的取指请求。

我们继续解析后续代码：

```cpp
/*  ICache状态的管理 */
    enum IcacheState
    {
	/*  正常处理请求，可推进缓存队列*/
        IcacheRunning, 
	/*   请求被拒绝（缓存拥堵）或者缓存未命中，需要重试*/
        IcacheNeedsRetry
    };
	/*  请求队列类型定义，FetchRequestPtr是指向取指请求的智能指针。*/
    typedef Queue<FetchRequestPtr,
        ReportTraitsPtrAdaptor<FetchRequestPtr>,
        NoBubbleTraits<FetchRequestPtr> >
        FetchQueue;

    /*  */
    FetchQueue requests;

    /*  */
    FetchQueue transfers;

    /* ICache的状态，是ICacheRunning还是ICacheNeedsRetry*/
    IcacheState icacheState;

    /*  指令行序列号，唯一标识并排序取指的指令行，当发生分支预测错误或流水线刷新时过滤旧请求 */
    InstSeqNum lineSeqNum;

    /*  请求计数器，在内存系统中未完成的请求数，包括缓存、总线、内存控制器中的请求 */
    unsigned int numFetchesInMemorySystem;
    /*  ITLB中正在处理的请求数，正在进行地址翻译的请求数量 */
    unsigned int numFetchesInITLB
```

到现在我们基本完成了对成员结构体，成员变量等的解析。现在我们把目光放到成员函数上去，并进行解析：

```cpp
std::ostream &
operator <<(std::ostream &os, Fetch1::FetchState state)
{
    switch (state) {
      case Fetch1::FetchHalted:
        os << "FetchHalted";
        break;
      case Fetch1::FetchWaitingForPC:
        os << "FetchWaitingForPC";
        break;
      case Fetch1::FetchRunning:
        os << "FetchRunning";
        break;
      default:
        os << "FetchState-" << static_cast<int>(state);
        break;
    }
    return os;
}
```

这个是对操作符<<进行了定义，主要是让他可以 os << state，如果这么做了，会打印出此时Fetch的状态。

接下来是changeStream()，这个函数是根据分支指令BranchData来更新取指阶段的线程状态、PC和取指地址，处理流水线刷新、线程挂起/恢复、指令流切换等场景。核心逻辑是根据分支原因(branch.reason)调整取指状态机，确保指令流正确切换。

```cpp
void
Fetch1::changeStream(const BranchData &branch)
{
    /* 根据分支数据中的threadId索引到对应的线程状态信息(Fetch1ThreadInfo)*/
    Fetch1ThreadInfo &thread = fetchInfo[branch.threadId];
    /*  根据线程的streamSeqNum和predictionSeqNum,标记新的指令流或预测上下文*/
    /*  前面有提到，SeqNum越小表示越旧，SeqNum越大表示越新，那么旧的应该被删掉*/
    updateExpectedSeqNums(branch);
    
    /* 根据分支原因来管理状态机 */
    switch (branch.reason) {
      case BranchData::SuspendThread:
        {
            if (thread.wakeupGuard) {
		/*  SuspendThread挂起线程，前面提到wakeupGuard保证唤醒后起码执行一次*/
                DPRINTF(Fetch, "Not suspending fetch due to guard: %s\n",
                                branch);
            } else {
		/*  将线程状态切换为FetchWaitingForPC，等待正确的PC值*/
                DPRINTF(Fetch, "Suspending fetch: %s\n", branch);
                thread.state = FetchWaitingForPC;
            }
        }
        break;
      case BranchData::HaltFetch:
        /* 假如发生了严重的异常，直接将Fetch1挂起，而不做操作*/
        DPRINTF(Fetch, "Halting fetch\n");
        thread.state = FetchHalted;
        break;
      default:
	/*  改变指令流，将线程状态切换为FetchRunning，比如普通分支跳转和流水线刷新后恢复取指*/
        DPRINTF(Fetch, "Changing stream on branch: %s\n", branch);
        thread.state = FetchRunning;
        break;
    }
    /*  更新PC值以及取指地址，set函数是将分支目标地址的设置到线程的pc中*/
    set(thread.pc, branch.target);
    /*  fetchAddr更新，将取指地址设为PC的指令地址，确保下次取指从正确位置开始*/
    thread.fetchAddr = thread.pc->instAddr();
}
```

changeStream函数的目标在于流切换，我们继续解析updateExpectedSeqNums():
此函数用于更新取指线程的流序列号(streamSeqNum)和预测序列号(predictionSeqNum)以响应分支事件（如分支预测错误或流水线刷新）。核心目的在于标记指令流版本和预测上下文，确保后续取指操作正确处理有效指令，过滤过时或无效请求。

```cpp
void
Fetch1::updateExpectedSeqNums(const BranchData &branch)
{
    /* 根据线程Id，获取对应的线程状态*/
    Fetch1ThreadInfo &thread = fetchInfo[branch.threadId];

    /* 更新流序列号 */
    thread.streamSeqNum = branch.newStreamSeqNum;
    /* 更新预测序列号，如果执行阶段发现预测错误，可能提供旧的newPredictionSeqNum，此时要撤销无效预测 */
    thread.predictionSeqNum = branch.newPredictionSeqNum;
}
```

这个函数就是更新两个指令流。
接下来我们来看processResponse()：
这个方法用于处理取指请求的响应（如缓存命中或内存返回的数据），将结果封装到ForwardLineData结构中传递给下一阶段(Fetch2)，核心功能包括错误处理、数据传递、地址管理和状态更新，确保取指流水线在异常和正常场景下均能工作。

```cpp
void
Fetch1::processResponse(Fetch1::FetchRequestPtr response,
    ForwardLineData &line)
{
    /*  获取线程状态和数据包，packet为从内存子系统中返回的数据包*/
    Fetch1ThreadInfo &thread = fetchInfo[response->id.threadId];
    PacketPtr packet = response->packet;
    /*   根据response里面的fault信息设置line的错误信息*/
    line.setFault(response->fault);
    /*   根据response的序列号和线程ID设置line的ID */
    line.id = response->id;
    /*   深拷贝线程的PC状态到line的pc值 */
    set(line.pc, thread.pc);
    /*   记录取指地址（虚拟地址） */
    line.fetchAddr = response->pc;
    /*   对齐后的基地址*/
    line.lineBaseAddr = response->request->getVaddr();
    /*    这里的设计逻辑：若响应包含错误，则将其传递到Fetch2，触发后续异常处理*/
    /*    通过line.id和line.pc确保Fetch2能正确关联指令流和预测序列号*/
    /*    地址记录：fetchAddr为取指操作的虚拟地址，lineBaseAddr用于指令解码和预测*/
    if (response->fault != NoFault) {
        /*  发生错误时，将线程状态切换到FetchWaitingForPC，暂停新请求等待正确的PC值 */
	/*  无法立即清除队列中的旧请求，需依赖后续流水线刷新或异常处理*/
        DPRINTF(Fetch, "Stopping line fetch because of fault: %s\n",
            response->fault->name());
        thread.state = Fetch1::FetchWaitingForPC;
    } else {
	/*   adoptPacketData将packet的数据指针移动到line，避免内存拷贝。*/
        line.adoptPacketData(packet);
	/*    response->packet设置为空，防止析构函数重复释放数据包内存。*/
        response->packet = NULL;
    }
}
```

这里面包含了一个adoptPacketData()，我们也来解析他：
他位于pipe_data.cc，将ForwardLineData实例的packet指针指向传来的packet。
然后设置lineWidth，bubbleFlag=false表示这是一个有效的实例。

```cpp
void
ForwardLineData::adoptPacketData(Packet *packet)
{
    this->packet = packet;
    /*  packet->req指向Packet关联的原始FetchRequest（请见Part2），返回请求的数据大小*/
    lineWidth = packet->req->getSize();
    /*  表示数据是有效的，而非是气泡*/
    bubbleFlag = false;

    assert(!isFault());
    assert(!line);

    line = packet->getPtr<uint8_t>();
}
```

我们继续，下一个是getScheduledThread():
此函数用于根据配置的线程调度策略选择当前应取指的线程，核心逻辑是生成候选线程优先级列表，并选择第一个满足条件的线程。如果没有可选的线程，则返回InvalidThreadID。

```cpp
inline ThreadID
Fetch1::getScheduledThread()
{
    std::vector<ThreadID> priority_list;

    switch (cpu.threadPolicy) {
    /*  单线程模式，仅线程0*/
      case enums::SingleThreaded:
        priority_list.push_back(0);
        break;
    /*   轮询调度，前面提到了threadPriority记录了当前时刻应该优先处理的线程*/
    /*    会返回一个priority_list，里面的元素以threadPriority记录的元素为第一优先级，其余的依次R-R*/
      case enums::RoundRobin:
        priority_list = cpu.roundRobinPriority(threadPriority);
        break;
      case enums::Random:
    /*    生成一个随机顺序的线程列表，用于测试或负载均衡*/
        priority_list = cpu.randomPriority();
        break;
      default:
        panic("Unknown fetch policy");
    }
    /*    遍历候选线程，选择一个可用的线程，条件是：线程Active，fetch1没有block且FetchRunning*/
    for (auto tid : priority_list) {
        if (cpu.getContext(tid)->status() == ThreadContext::Active &&
            !fetchInfo[tid].blocked &&
            fetchInfo[tid].state == FetchRunning) {
            threadPriority = tid;
	    /* 返回选中的线程  */
            return tid;
        }
    }
   return InvalidThreadID;
}
```

一个还算清晰的选择线程的逻辑对吧？
现在我们来看最终的fetchLine，看他是如何工作的：

```cpp
void
Fetch1::fetchLine(ThreadID tid)
{
   /*  获取当前线程的FetchInfo*/
    Fetch1ThreadInfo &thread = fetchInfo[tid];
    /* 计算当前线程的取指地址对齐后的基地址(aligned_pc)、相对于对齐基地址的偏移量以及需要请求的内存数据大小 */
    Addr aligned_pc = thread.fetchAddr & ~((Addr) lineSnap - 1);
    unsigned int line_offset = aligned_pc % lineSnap;
    unsigned int request_size = maxLineWidth - line_offset;
    /*  创建并初始化一个新的取指请求(FetchRequest)，将其插入取指队列以从MSS中获取指令数据。*/
    /*  tid为当前线程ID，streamSeqNum为线程的流序列号，predictionSeqNum为预测序列号，lineSeqNum为行序列号*/
    InstId request_id(tid,
        thread.streamSeqNum, thread.predictionSeqNum,
        lineSeqNum);
    /*  创建取指请求对象，this为Fetch1对象的引用，request_id为生成的指令标识符，fetchAddr为取指地址*/
    FetchRequestPtr request = new FetchRequest(*this, request_id,
            thread.fetchAddr);

    DPRINTF(Fetch, "Inserting fetch into the fetch queue "
        "%s addr: 0x%x pc: %s line_offset: %d request_size: %d\n",
        request_id, aligned_pc, thread.fetchAddr, line_offset, request_size);
    /*  获取线程的上下文ID，并以此设置内存请求的上下文ID*/
    request->request->setContext(cpu.threads[tid]->getTC()->contextId());
    /*   设置虚拟内存请求，对齐后的PC，请求大小，请求类型，请求地址*/
    /*   cpu.instRequestorId是一个全局的参数，每个cpu有一个ID，必须添加在request里，是GEM5的要求*/
    request->request->setVirt(
        aligned_pc, request_size, Request::INST_FETCH, cpu.instRequestorId(),
        /* "I've no idea why we need the PC, but give it"，这是Minor原作者的注释，他也不明白 */
        thread.fetchAddr);
    /*   在ITLB中的取指令数量加一*/
    DPRINTF(Fetch, "Submitting ITLB request\n");
    numFetchesInITLB++;
    /*    设置request的状态为InTranslation*/
    request->state = FetchRequest::InTranslation;

    /* 在传输队列transfers中预留空间，并且将请求加入待处理队列requests */
    transfers.reserve();
    requests.push(request);

    /*  translateTiming就是提交一个时序的地址翻译请求 */
    cpu.threads[request->id.threadId]->mmu->translateTiming(
        request->request,
        cpu.getContext(request->id.threadId),
        request, BaseMMU::Execute);
    /*   每个新取指行分配一个唯一序列号，确保流水线按顺序处理指令*/
    /*   前面提到的，流水线刷新时，通过序列号过滤过时请求*/
    lineSeqNum++;

    /* 更新取指地址，假设对齐地址从aligned_pc开始，读取request_size字节后，下一Inst地址 */
    thread.fetchAddr = aligned_pc + request_size;
}
```

FetchLine就是设置Fetch请求的一个功能函数，按照fetch1.hh里面的顺序，接下来看tryToSendToTransfers():
此函数负责将已翻译完成的取指请求从请求队列requests移动到传输队列transfers，并尝试将请求发送到内存子系统(ICache)。

```cpp
Fetch1::tryToSendToTransfers(FetchRequestPtr request)
{
    /*   确保只有队列头部请求能够被处理，防止乱序执行*/
    if (!requests.empty() && requests.front() != request) {
        DPRINTF(Fetch, "Fetch not at front of requests queue, can't"
            " issue to memory\n");
        return;
    }
    /*   检查request的状态，是否是在翻译中的状态，如果正在地址翻译（ITLB未命中），则无法继续处理*/
    /*   地址翻译由MMU异步完成，完成后会触发回调更新状态至Translated*/
    if (request->state == FetchRequest::InTranslation) {
        DPRINTF(Fetch, "Fetch still in translation, not issuing to"
            " memory\n");
        return;
    }
    /*    处理可丢弃或错误的请求，首先切换request状态为Complete，然后把他传输给Transfers*/
    if (request->isDiscardable() || request->fault != NoFault) {
        request->state = FetchRequest::Complete;
        moveFromRequestsToTransfers(request);

        /* 唤醒cpu */
        cpu.wakeupOnEvent(Pipeline::Fetch1StageId);
    } else if (request->state == FetchRequest::Translated) {
	/*  处理已翻译的请求，创建内存数据包packet，request和packet的关系Part2有讨论*/
        if (!request->packet)
            request->makePacket();
        /*   这个断言是确认数据包需要响应，以确保内存子系统处理完成后返回结果*/
        assert(request->packet->needsResponse());
	/*   把request发送到内存子系统，如果成功则把request转移到transfers*/
        if (tryToSend(request))
            moveFromRequestsToTransfers(request);
    } else {
        DPRINTF(Fetch, "Not advancing line fetch\n");
    }
}
```

总结一下这个函数，首先检查request是否是队列最前端，这一点是为了保证执行顺序，然后检查错误与判断是否可丢弃（主要是检查流序列号正确性，如果是过时的请求就会被丢弃），这两种状态会不会发送到内存子系统。

如果条件都符合，并且翻译完成的话，会把请求尝试发送到内存子系统，然后移动到Transfers。

上述函数里有一个将请求发送给内存子系统的函数tryToSend(request)，下面我们来解析tryToSend()：

此函数负责将取指请求的数据包request->packet发送到ICache端口(ICachePort，我们在Part1详细解析了Port类)，根据发送结果来更新请求状态和缓存端口状态。

```cpp
bool
Fetch1::tryToSend(FetchRequestPtr request)
{
    /*  定义一个bool变量，表示return状态*/
    bool ret = false;
    /*   尝试发送请求，这个请求是Timing协议的*/
    if (icachePort.sendTimingReq(request->packet)) {
        /* 解除请求对Packet的所有权，request的状态变为RequestIssuing，内存子系统未完成请求数+1 */
        request->packet = NULL;
        request->state = FetchRequest::RequestIssuing;
        numFetchesInMemorySystem++;
	/*表示返回成功*/
        ret = true;
    } else {
        /* 缓存未命中，需要重试 */
        icacheState = IcacheNeedsRetry;
    }
    return ret;
}
```

总结一下，这个函数通过icachePort发送Timing请求，如果下游成功接收并返回true，那么表示传输成功。

反之，如果下游并未成功接收，那么此时要标记icacheState的状态为IcacheNeedsRetry。

tryToSendToTransfers()函数中还提到一个moveFromRequestsToTransfers()，这个函数比较简单，我们来看看：

```cpp
void
Fetch1::moveFromRequestsToTransfers(FetchRequestPtr request)
{
    assert(!requests.empty() && requests.front() == request);
    requests.pop();
    transfers.push(request);
}
```

总结一下，这个函数判断request队列非空并且request是前端元素，这里用assert是为了防范设计错误。

然后requests pop一个元素，并且push给transfer，由于这里传递的都是指针，避免了很多的数据拷贝。

```cpp
void
Fetch1::stepQueues()
{
    /*  类似Verilog里对状态机的处理，curState <= nextState，状态机的更新*/
    IcacheState old_icache_state = icacheState;
    /*   ICacheRunning状态的话，移动ITLB中的请求到内存子系统中*/
    switch (icacheState) {
      case IcacheRunning:
        if (!requests.empty()) {
            tryToSendToTransfers(requests.front());
        }
        break;
      case IcacheNeedsRetry:
        break;
    }
    /*  记录状态机的跳转过程，用于调试*/
    if (icacheState != old_icache_state) {
        DPRINTF(Fetch, "Step in state %s moving to state %s\n",
            old_icache_state, icacheState);
    }
}
```

stepQueues函数就是推进状态，并且记录状态机跳转。

按照Fetch1.hh里面的顺序，接下来是一个比较简单的函数popAndDiscard()

```cpp
void
Fetch1::popAndDiscard(FetchQueue &queue)
{
    if (!queue.empty()) {
        delete queue.front();
        queue.pop();
    }
}
```

这个很简单，如果queue非空的话，析构queue顶端元素，然后pop

然后是对ITLB响应的处理函数handleTLBResponse()：

这段函数用于处理从ITLB返回的地址翻译响应，更新请求状态并尝试将请求推进到内存访问阶段。

```cpp
void
Fetch1::handleTLBResponse(FetchRequestPtr response)
{
    /*  减少ITLB处理请求计数*/
    numFetchesInITLB--;
    /*   处理翻译错误，生成日志，比如缺页异常和权限错误*/
    if (response->fault != NoFault) {
        DPRINTF(Fetch, "Fault in address ITLB translation: %s, "
            "paddr: 0x%x, vaddr: 0x%x\n",
            response->fault->name(),
            (response->request->hasPaddr() ?
                response->request->getPaddr() : 0),
            response->request->getVaddr());
        /*   更细粒度的跟踪日志，辅助分析错误路径*/
        if (debug::MinorTrace)
            minorTraceResponseLine(name(), response);
    } else {
        DPRINTF(Fetch, "Got ITLB response\n");
    }
    /*  切换response的状态，切换为Translated，无论成功与否，表示翻译阶段的结束*/
    /*   错误信息通过response->fault传递，后续阶段会处理*/
    response->state = FetchRequest::Translated;
    /*   调用这个函数，尝试把请求发送到内存子系统*/
    tryToSendToTransfers(response);
}
```

这里出现过的minorTrace机制暂且不表，这是一种跟踪bug辅助调谐的功能，我觉得这个看设计者的喜好，不过可以进去看看设计者的debug调试思路，看看能不能从中获得启发。

后续还有一个简单的函数numInFlightFetches()：

```cpp
unsigned int
Fetch1::numInFlightFetches()
{
    return requests.occupiedSpace() +
        transfers.occupiedSpace();
}
```

就是返回requests和transfers里面占据的元素数量，就是正在Fetch的请求。

接下来是Timing协议的一部分，recvTimingResp()，Part1里对port有过描述，recvTimingResp()是TimingPortocol里定义过的纯虚函数，ICachePort那个层级也重载了recvTimingResp()，指向这里，我们来看实现:

此函数用于处理从内存子系统返回的时序响应（Cache或内存中返回的指令数据）。

```cpp
bool
Fetch1::recvTimingResp(PacketPtr response)
{
    DPRINTF(Fetch, "recvTimingResp %d\n", numFetchesInMemorySystem);

    /*  从响应里提取发送时保存的请求对象FetchRequest*/
    /*  safe_case是一种安全转换，确保senderState指针指向FetchRequest对象*/
    FetchRequestPtr fetch_request = safe_cast<FetchRequestPtr>
        (response->popSenderState());

    /* 确保fetch_request的packet为空，将响应包重新绑定到请求*/
    assert(!fetch_request->packet);
    fetch_request->packet = response;
    /*  更新计数器，更新fetch_request的状态为Complete*/
    /*  Complete表示已结束对内存访问阶段，准备移交给Fetch2处理*/
    numFetchesInMemorySystem--;
    fetch_request->state = FetchRequest::Complete;
    /*  调试追踪*/
    if (debug::MinorTrace)
        minorTraceResponseLine(name(), fetch_request);
    /*   如果响应是错误的，打印错误包*/
    if (response->isError()) {
        DPRINTF(Fetch, "Received error response packet: %s\n",
            fetch_request->id);
    }

    /* 唤醒流水线 */
    cpu.wakeupOnEvent(Pipeline::Fetch1StageId);
    return true;
}
```

接下来是recvReqRetry()

```cpp
void
Fetch1::recvReqRetry()
{
    /*  断言icacheState的状态为ICacheNeedsRetry，且requests是非空的*/
    DPRINTF(Fetch, "recvRetry\n");
    assert(icacheState == IcacheNeedsRetry);
    assert(!requests.empty());
    
    FetchRequestPtr retryRequest = requests.front();

    icacheState = IcacheRunning;
    /*重试发送request*/
    if (tryToSend(retryRequest))
        moveFromRequestsToTransfers(retryRequest);
}
```

到这里主要描述函数个体，不是很成系统，不过至少我们对每个函数个体有了一个比较深刻的解析，下一个Part我会讲解他们是如何通过evaluate函数串起来的。到这里先休息一下。






