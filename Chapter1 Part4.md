# GEM5 Minor Core 解析

本人以Minor Core为模板来学习Minor Core，并进而学习GEM5仿真系统，以此为记录。
以流水线Front-End到Back-End为顺序来进行解析，如有错误，敬请指正。

行文以一个初学者第一视角开始，希望给读者带来一个不错的体验。

## Fetch1

此部分主要包含两部分代码，fetch1.cc 和 fetch1.hh，其中涉及到的系统组件我将会出一个GEM5源码解析来讲述，此处只关注基于系统组件来构建仿真核的这一部分。

解析也会按照官方代码里从上往下进行解析，如有需要跳转的部分我会说明。

### Fetch1.hh

上一节我们对各个函数个体进行了解析，这一部分解析evaluate函数，以及我们来讨论一下整个Fetch1是如何工作起来的，evaluate函数较长，我会把解析写在函数主体里。

```cpp
void
Fetch1::evaluate()
{
    /*  获取来自执行阶段的分支信息*/
    const BranchData &execute_branch = *inp.outputWire;
    /*  获取来自fetch2分支预测阶段的分支信息*/
    const BranchData &fetch2_branch = *prediction.outputWire;
    /*  引用Fetch1的数据输出端口，ForwardLineData是用于将取到的指令行打包起来传递给Fetch2*/
    ForwardLineData &line_out = *out.inputWire;
    /*  确保输出线当前为空，避免数据覆盖和竞争*/
    assert(line_out.isBubble());
    /*   多线程阻塞状态更新，nextStageReserve[tid].canReserve()是检查下一阶段(Fetch2)的缓冲区是否空间足够来接收新数据*/
    for (ThreadID tid = 0; tid < cpu.numThreads; tid++)
        fetchInfo[tid].blocked = !nextStageReserve[tid].canReserve();
     
    /* 分支有效性检查，首先threadId并非无效thread，其次exe和fetch2传来的分支信息属于同线程 */
    if (execute_branch.threadId != InvalidThreadID &&
        execute_branch.threadId == fetch2_branch.threadId) {
        /*  获取Fetch1里线程上下文，包含PC值，序列号，阻塞状态*/
        Fetch1ThreadInfo &thread = fetchInfo[execute_branch.threadId];

        /* 处理Execute的分支（实际分支结果），只要不是FetchHalted的状态就切换流 */
        if (execute_branch.isStreamChange()) {
            if (thread.state == FetchHalted) {
                DPRINTF(Fetch, "Halted, ignoring branch: %s\n", execute_branch);
            } else {
                changeStream(execute_branch);
            }
            /* 同一拍内Execute的分支信息优先级更高，可以忽略fetch2的预测结果*/
            if (!fetch2_branch.isBubble()) {
                DPRINTF(Fetch, "Ignoring simultaneous prediction: %s\n",
                    fetch2_branch);
            }
        } else if (thread.state != FetchHalted && fetch2_branch.isStreamChange()) {
            /* 处理Fetch2阶段的预测分支，次优先级，只要Fetch1非停滞就切换流*/
            /*  下面这个判断条件是因为当分支预测的流和thread最新流不一致的话，丢弃分支预测结果*/
            if (fetch2_branch.newStreamSeqNum != thread.streamSeqNum) {
                DPRINTF(Fetch, "Not changing stream on prediction: %s,"
                    " streamSeqNum mismatch\n",
                    fetch2_branch);
            } else {
                changeStream(fetch2_branch);
            }6. Single Precision Mathematical Functions — CUDA Math API Reference Manual 12.8 documentation
        }
    }else{
        /* 分支预测和Execute的结果是不同线程的 */
        if (execute_branch.threadId != InvalidThreadID &&
            execute_branch.isStreamChange()) {

            if (fetchInfo[execute_branch.threadId].state == FetchHalted) {
                DPRINTF(Fetch, "Halted, ignoring branch: %s\n", execute_branch);
            } else {
                /* 并非处于FetchHalted的状态就进行流切换*/
                changeStream(execute_branch);
            }
        }

        if (fetch2_branch.threadId != InvalidThreadID &&
            fetch2_branch.isStreamChange()) {

            if (fetchInfo[fetch2_branch.threadId].state == FetchHalted) {
                DPRINTF(Fetch, "Halted, ignoring branch: %s\n", fetch2_branch);
            } else if (fetch2_branch.newStreamSeqNum != fetchInfo[fetch2_branch.threadId].streamSeqNum) {
                DPRINTF(Fetch, "Not changing stream on prediction: %s,"
                    " streamSeqNum mismatch\n", fetch2_branch);
            } else {
                /*  应用分支预测，根据分支预测的结果切换流*/
                changeStream(fetch2_branch);
            }
        }
    }
    / *分开解析*/
    ...
}
```

上一部分是对Execute、Fetch2执行结果的处理，Execute的分支结果是更高优先级的，而Fetch2的结果是较低优先级的。接下来继续

```cpp
void
Fetch1::evaluate()
{
    ...
    /*  正在Fetch的指令应该小于上限*/
    if (numInFlightFetches() < fetchLimit) {
        /*getScheduledThread()是根据优先级策略得到一个优先级列表，然后选出一个fetch的线程*/
        ThreadID fetch_tid = getScheduledThread();
        
        if (fetch_tid != InvalidThreadID) {
            DPRINTF(Fetch, "Fetching from thread %d\n", fetch_tid);

            /*  根据选出的线程来触发fetchLine*/
            fetchLine(fetch_tid);
            /*  下一个Stage留出空间 */
            nextStageReserve[fetch_tid].reserve();
        } else {
            DPRINTF(Fetch, "No active threads available to fetch from\n");
        }
    }
     
    /* 根据Cache状态，将请求推入内存子系统*/
    stepQueues();

    /* 负责处理传输队列已完成的取指请求 */
    if (!transfers.empty() &&
        transfers.front()->isComplete())
    {
        Fetch1::FetchRequestPtr response = transfers.front();

        if (response->isDiscardable()) {
            /*  下一阶段预留资源*/
            nextStageReserve[response->id.threadId].freeReservation();

            DPRINTF(Fetch, "Discarding translated fetch as it's for"
                " an old stream\n");
            /*  CPU唤醒*/
            cpu.wakeupOnEvent(Pipeline::Fetch1StageId);
        } else {
            DPRINTF(Fetch, "Processing fetched line: %s\n",
                response->id);
            /* 将请求中的指令数据封装到line_out内，供Fetch2阶段读取*/
            processResponse(response, line_out);
        }
        /* 从队列中移除头部请求*/
        popAndDiscard(transfers);
    }

    /* 如果输出线为非空，调用cpu.activityRecorder->activity()标记当前流水线阶段为活跃 */
    if (!line_out.isBubble())
        cpu.activityRecorder->activity();

    /* 重置线程唤醒保护标志，每个周期结束将wakeupGuard置False，为下一周期冲突检查做准备*/
    for (auto& thread : fetchInfo) {
        thread.wakeupGuard = false;
    }
}
```









