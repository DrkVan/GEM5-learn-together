# GEM5 Minor Core 解析

本人以Minor Core为模板来学习Minor Core，并进而学习GEM5仿真系统，以此为记录。
以流水线Front-End到Back-End为顺序来进行解析，如有错误，敬请指正。

行文以一个初学者第一视角开始，希望给读者带来一个不错的体验。

## Fetch1

此部分主要包含两部分代码，fetch1.cc 和 fetch1.hh，其中涉及到的系统组件我将会出一个GEM5源码解析来讲述，此处只关注基于系统组件来构建仿真核的这一部分。

解析也会按照官方代码里从上往下进行解析，如有需要跳转的部分我会说明。

### Fetch1.hh


```cpp
class Fetch1 : public Named
{
  ···
}
```

Fetch1是硬件模块取指单元，Minor Core的取指单元由Fetch1和Fetch2两部分共同组成。
此模块继承了Named，Named为系统组件，以下为源代码：

```cpp
class Named
{
  private:
    const std::string _name;

  public:
    Named(const std::string &name_) : _name(name_) { }
    virtual ~Named() = default;

    virtual std::string name() const { return _name; }
};
```

我们可以看到这部分包含了一个字符串，记录这个模块的名字，包含一个方法也就是返回模块的name，这一点很好理解，在debug或者打印log的时候，我们需要模块的名称。

接下来我们继续回到Fetch1的代码本身：

```cpp
class Fetch1 : public Named
{
  protected:
    /** Exposable fetch port */
    class IcachePort : public MinorCPU::MinorCPUPort
    ···
}
```

#### 小节一：用户Port是如何实现的，以及系统组件里的Port

这里在Fetch1模块内部定义了一个IcachePort的类，并且他继承于MinorCPU::MinorCPUPort，其中MinorCPU是这个CPU的顶层这个很好理解，那我们来看MinorCPUPort是什么：

```cpp
class MinorCPUPort : public RequestPort
    {
	……
    };
```

我们看到MinorCPUPort又继承了RequestPort，那我们需要看看RequestPort是什么，首先他是一个重要的系统组件，我们来看到他的源代码：

```cpp
class RequestPort: public Port, public AtomicRequestProtocol,
    public TimingRequestProtocol, public FunctionalRequestProtocol
{
	……
}
```

OK，又是继承，我们继续先看他继承的东西，首先是Port，然后是三种协议，我这里先告知一下，Atomic、Timing、Functional是请求协议，其中Timing协议会模拟真实的协议时序，Atomic则是最快的协议仿真，而Port也是一个重要的系统组件。那我们先看看Port是什么。

这里看来已经到底层了，我们可以解析完这一段再回到Fetch1，可能你已经觉得头疼了，但是这些重复的东西我们只会看一次。Port这部分我把解析放在代码注释里：

```cpp
class Port
{
  private:
    /* 一个字符串，记录portName，就是记录这个port的名字，再后面debug包括打印log的时候会有用，你总
	要知道是哪个port有问题，或者哪个port的仿真表现。 */
    const std::string portName;
  protected:
    /* 这个部分很简单，就是报告Port未绑定，因为出现Port未绑定的情况就是设计者设计出现了错误，就是报
	告一个FATAL */
    class UnboundPortException {};
    /* 你可能会感到困惑，为什么要声明一个空的异常类，它实际上就是用类型表示错误类型，这样以后如果有
	拓展需求就可以直接用这个类继承exception，然后就可以灵活表示什么原因的exception。

	至于[[noreturn]]的用法，这里是C++11引入的特性，告诉编译器这个函数不会返回调用点，因为程序到此
	为止了*/

    [[noreturn]] void reportUnbound() const;
    /* 这个很好理解，Port应该有一个自己的ID，用于debug和打印log */
    const PortID id;

    /* 这一部分比较重要，首先通讯应该构成请求-响应对，一个port如果是请求port，那么他应该有一个相连的
	响应port，反之亦然，所以一个port应该有一个peer，两个port构成请求-响应对 */
    Port *_peer;

    /* 这一部分也可以理解，port应该是被连接好的，所以有这样一个标志位表示port处于被连接好的状态 */
    bool _connected;

    /* 构造函数，并且构造实例的时候应该指定他的name和他的id */
    Port(const std::string& _name, PortID _id);

  public:

    /* 析构函数 */
    virtual ~Port();

    /* 返回这个port的peer的指针，得到这个port对应的Response-Port或Require-port */
    Port &getPeer() { return *_peer; }

    /* 返回这个port的名字，用于log或debug，这一点好理解 */
    const std::string name() const { return portName; }

    /* 返回这个port的id，用于log或debug */
    PortID getId() const { return id; }

    /* bind方法，也就是把你对应的Response-Port或Require-port的指针拿到，赋给peer ，同时设置connected*/
    virtual void
    bind(Port &peer)
    {
        _peer = &peer;
        _connected = true;
    }

    /* 解除绑定 */
    virtual void
    unbind()
    {
        _peer = nullptr;
        _connected = false;
    }

    /* 返回绑定状态，检查这个port是否是被绑定的 */
    bool isConnected() const { return _connected; }

    /* 端口替换，用当前端口实例安全地替换另一个旧端口，得到旧端口的peer，然后本实例与之连接
	类似于挖墙脚，你把别人的peer撬过来与你形成绑定了。
	可以用于仿真环节里面的热替换或重配置*/
    void
    takeOverFrom(Port *old)
    {
        assert(old);
        assert(old->isConnected());
        assert(!isConnected());
        Port &peer = old->getPeer();
        assert(peer.isConnected());

        // Disconnect the original binding.
        old->unbind();
        peer.unbind();

        // Connect the new binding.
        peer.bind(*this);
        bind(peer);
    }
};
```

到这里我们完成了对port的代码解析，我们可以看到到了底层代码总是简单的， 再复杂的代码都是由简单的代码组成。port是简单的，后面我们会遇到较为困难的部分，不过没事你总会克服困难。

那我们回到上一个层级，也就是RequestPort：

```cpp
class RequestPort: public Port, public AtomicRequestProtocol,
    public TimingRequestProtocol, public FunctionalRequestProtocol {...}
```

三种协议我们先不讨论了，面对三种仿真情况，我们把目光放到RequestPort这个类，还是一样的我在代码里面进行解析：

```cpp
class RequestPort: public Port, public AtomicRequestProtocol,
    public TimingRequestProtocol, public FunctionalRequestProtocol
{
    /*将peer指定为友元函数，这一点很好理解，Response-Port有时候需要RequestPort的信息*/
    friend class ResponsePort;

  private:
    /*指向自己的peer的指针*/
    ResponsePort *_responsePort;

  protected:
    /*指向自己的更高层级*/
    SimObject &owner;

    /*[[deprecated(...)]]是告诉编译器这个函数已弃用，当这个代码被使用时编译器会警告使用替代方案*/
  public:
    [[deprecated("RequestPort ownership is deprecated. "
                 "Owner should now be registered in derived classes.")]]
    RequestPort(const std::string& name, SimObject* _owner,
                PortID id=InvalidPortID);

    /*这个就是替代方案，将name和id信息传给自己的基类Port，*/
    RequestPort(const std::string& name, PortID id=InvalidPortID);
    ...
}
```

我们到这里先暂停，我们把目光放到RequestPort的构造函数，看看是怎么实现的：

```cpp
RequestPort::RequestPort(const std::string& name, PortID _id) :
    Port(name, _id), _responsePort(&defaultResponsePort),
    owner{*reinterpret_cast<SimObject*>(1)}
{
}
```

我们看到除了正常的将name,id赋给自己的基类Port外，还有对_responsePort和owner的处理。
其实这里defaultResponsePort是一个全局的默认Response-Port实例，在这一个层级构造时不需要指定实际的Response-Port，就先连一个默认的端口，这么做是为什么呢？是为了防止未绑定Response-Port时访问空指针，确保代码在未完全配置时也可以运行（模拟器初始化阶段）。
关于owner，这里就是采用了一个非空引用，避免空引用带来的未定义行为，实际所有者可以在派生类里面注册，这里我们可以学习到一些好的coding style。
那么到这里构造函数就解析完毕，我们回到RequestPort这个类的解读：

```cpp
RequestPort(const std::string& name, PortID id=InvalidPortID); 
    /*析构函数*/
    virtual ~RequestPort();
    /*重载绑定函数*/
    void bind(Port &peer) override 
{
    /*获得response_port的指针，如果response_port指针为空的话就报错*/
    auto *response_port = dynamic_cast<ResponsePort *>(&peer);
    fatal_if(!response_port, "Can't bind port %s to non-response port %s.",
             name(), peer.name());
    /*将获得的response_port指针赋给自己的peer，也就是上文提到的_responsePort*/
    _responsePort = response_port;
    /*调用基类方法，实现绑定*/
 Port::bind(peer);
    /*也在这个地方对Reponse-Port进行绑定，把自己的指针传过去，这个函数的内容就是
        Response-Port根据传过来的指针对自身进行绑定*/
    _responsePort->responderBind(*this);
}
    /*到这里我们结束了对bind的重载的解读，下一个函数是unbind()，望文生义，很好理解*/
 void unbind() override
{
    /*一个检查，如果这个port并非是绑定的就报错，因为一个未绑定的port不可能解绑定*/
    panic_if(!isConnected(), "Can't unbind request port %s which is "
    "not bound.", name());
    /*对自己的peer也进行解绑定*/
    _responsePort->responderUnbind();
    /*将自己的_responsePort指向一个responsePort的默认实现，来避免空引用带来的未定义行为*/
    _responsePort = &defaultResponsePort;
    /*调用自己的基类方法，来实现解绑定*/
    Port::unbind();
}
    /*我们可以看到有一个virtual，默认实现是return false，也就是默认实现里这个port并非是一个
        snooping的port，如果你需要这个port是snooping的话，那么要在派生类里面实现这个函数

        什么时候会涉及到snooping呢，比如Cache一致性里面，cache需要嗅探总线，这种时候需要snooping*/
virtual bool isSnooping() const {return false;}
    /*这里要对AddrRangeList特别说明，我们放在这一小部分后面，先别打断我们的解析*/
AddrRangeList getAddrRanges() const;

/*这里是一个调试工具函数，用于通过RequestPort向ResponsePort发送一个特殊功能性请求(Function的)
    触发对指定内存地址a的信息打印(如地址值，缓存行状态等)。核心目的是在不干扰模拟器正常时序逻辑的
    前提下快速获得目标地址的调试信息*/
void printAddr(Addr a)
{
    /*构造一个Request指针*/
    auto req = std::make_shared<Request>(
        a, 1, 0, Request::funcRequestorId);
    /*构造一个packet*/
    Packet pkt(req, MemCmd::PrintReq);
    /*构造一个打印请求的state，然后传给pkt，关于packet的描述先放后面*/
    Packet::PrintReqState prs(std::cerr);
    pkt.senderState = &prs;
    /*发送功能性请求，不影响时序*/
    sendFunctional(&pkt);
}

/*接下来是对Atomic Protocol / Timing Protocol / Functional的函数重载实现，你可以理解为三种协议的接收
    函数进行重载，我会放在后面而不是这里。三种协议可以单开一节，放在这里会影响你的注意力*/
    ...
/*我们跳过了三种协议的重载实现，直接来到类的末尾*/
/*这个函数用于在GEM5模拟器中为数据包(Packet)添加调试跟踪信息，记录经过的Request-Port和关联的Response-Port，以及其访问的主存地址。主要用于调试端口通信路径并且分析数据包传输行为*/
void addTrace(PacketPtr pkt) const
{
    /*检查全局调试开关debug::PortTrace是否打开，并且检查这个Packet指针是否是有效的*/
    if (!gem5::debug::PortTrace || !pkt)
        return;
   /*TracingExtension是一个拓展类，用于存储跟踪信息，端口名称以及地址
       getExtension是从数据包中获取已附加的拓展，如果不存在的话就构造一个*/
    auto ext = pkt->getExtension<TracingExtension>();
    if (!ext) {
        ext = std::make_shared<TracingExtension>();
        pkt->setExtension(ext);
    }
    /*ext构造出来了，添加端口名称和地址信息于扩展中用于分析*/
    ext->add(name(), _responsePort->name(), pkt->getAddr());
}
/* 从数据包中移除调试跟踪扩展，与addTrace配合使用，完成跟踪信息的生命周期管理，在调试完成后清除
    数据包中的跟踪信息，避免无效数据残留和资源泄露*/
void removeTrace(PacketPtr pkt) const
{
    /*检查全局调试开关并且检查pkt指针是否有效，然后移除ext*/
    if (!gem5::debug::PortTrace || !pkt)
        return;
    auto ext = pkt->getExtension<TracingExtension>();
    panic_if(!ext, "There is no TracingExtension in the packet.");
    ext->remove();
}
/*关于Trace机制我打算单开一节，到目前埋下的坑有三种协议，Trace机制，后面补上*/
}
```

到这里，我们完成了对RequestPort这一个层级的解析，埋下的坑有Trace机制，Timing/Atomic/Functional三种协议的解析，我后面单开一节。现在我们可以回到MinorCPUPort这一个层级的解析，到这里已经不是系统组件了，而是用户自己的重载实现，好在这里比较简单：

```cpp
/*继承了我们刚才解析的RequestPort*/
class MinorCPUPort : public RequestPort
    {
      public:
        /*Minor CPU顶层*/
        MinorCPU &cpu;

      public:
        /*构造函数顶层，将name传给RequestPort层级的构造函数，并且获得指向顶层的指针*/
        MinorCPUPort(const std::string& name_, MinorCPU &cpu_)
            : RequestPort(name_), cpu(cpu_)
        { }

    };
```

那么MinorCPUPort这一个层级解析完了，我们回到我们的主线上去，ICachePort，他是Fetch1的一员：

```cpp
class IcachePort : public MinorCPU::MinorCPUPort
    {
      protected:
        /*指向ICachePort的顶层，也就是Fetch1模块*/
        Fetch1 &fetch;

      public:
       /*一个空的构造函数，将名称以及顶层CPU指针传给MinorCPUPort这一个层级，并且传递了fetch指针*/
        IcachePort(std::string name, Fetch1 &fetch_, MinorCPU &cpu) :
            MinorCPU::MinorCPUPort(name, cpu), fetch(fetch_)
        { }

      protected:
        /*对recvTimingResp函数的重载，他是属于Timing协议的一部分，在Timing协议里这是一个纯虚函数
    	    可以看到他直接返回了fetch.recvTimingResp(pkt)的结果，可以看到这里还有一层重载*/
        bool recvTimingResp(PacketPtr pkt)
        { return fetch.recvTimingResp(pkt); }
	/*这里也是一样，可以看到在fetch1模块里对这个函数进行了重载，我们后面在Fetch1里面看他的实现*/
        void recvReqRetry() { fetch.recvReqRetry(); }
    };
```

很好，我们目前完成了对ICachePort这个层级的追根溯源，可能你会困惑为什么我要追踪到系统组件的层级，是因为我在过去开发项目的经历中经常遇到对底层细节不理解而导致非常费劲的debug工作，所以现在我为了安全起见都会在学习新事物的开始就追根溯源，这会带来不小的学习成本，不过好在这都是一次性开销。

到这里我们先休息，先收一章。





