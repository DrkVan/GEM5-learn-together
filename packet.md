# GEM5源码解析

## packet解析

发往内存子系统的数据包用packet这样的数据格式来表征。
这一部分讲解packet.hh和packet.cc，并且从设计者的视角来思考packet为什么是这样的。

首先是第一个类，MemCmd
```cpp
class MemCmd
{
    /*允许Packet类访问MemCmd的私有成员*/
    friend class Packet; 

  public:
    /* 定义了内存命令的类型和属性 */
    /**
     * 基本操作：ReadReq/ReadResp, WriteReq/WriteResp 
     * 缓存维护：WritebackDirty,CleanEvict,InvalidateReq
     * 原子操作：SwapReq, LockedRMWReadReq, StoreCondReq
     * 预取机制：SoftPFReq, HardPFReq
     * 错误处理：ReadError,BadAddressError
     * 模拟专用：PrintReq，FlushReq
     * 同步指令：MemFenceReq，MemSyncReq
     * 事务内存：HTMReq，HTMAbort
     */
    enum Command/*内存命令枚举*/
    {
        InvalidCmd,/*无效请求*/
        ReadReq,/*读请求*/
        ReadResp,/*读响应*/
        ReadRespWithInvalidate,
        WriteReq,/*写请求*/
        WriteResp,/*写响应*/
        WriteCompleteResp,
        WritebackDirty,/*缓存行写回（脏）*/
        WritebackClean,/*缓存行写回（干净）*/
        WriteClean,
        CleanEvict,
        SoftPFReq,
        SoftPFExReq,
        HardPFReq,
        SoftPFResp,/*软件预取请求*/
        HardPFResp,/*硬件预取请求*/
        WriteLineReq,
        UpgradeReq,/*缓存权限升级（如只读升为可写）*/
        SCUpgradeReq,  
        UpgradeResp,/*缓存权限升级（如只读可升为可写）*/
        SCUpgradeFailReq,   
        UpgradeFailResp,       
        ReadExReq,
        ReadExResp,
        ReadCleanReq,
        ReadSharedReq,
        LoadLockedReq,
        StoreCondReq,
        StoreCondFailReq,   
        StoreCondResp,
        LockedRMWReadReq,/*原子锁定的读-修改-写请求*/
        LockedRMWReadResp,/*原子锁定的读-修改-写响应*/
        LockedRMWWriteReq,
        LockedRMWWriteResp,
        SwapReq,
        SwapResp,
        MemFenceReq = SwapResp + 3,
        MemSyncReq,  
        MemSyncResp,
        MemFenceResp,
        CleanSharedReq,
        CleanSharedResp,
        CleanInvalidReq,
        CleanInvalidResp,

        InvalidDestError, 
        BadAddressError, 
        ReadError, 
        WriteError, 
        FunctionalReadError, 
        FunctionalWriteError,

        PrintReq,       
        FlushReq,  
        InvalidateReq,  
        InvalidateResp,
    
        HTMReq,
        HTMReqResp,
        HTMAbort,
        TlbiExtSync,
        NUM_MEM_CMDS/*命令总数*/
    };

  private:
    /**
     * 命令属性枚举
     */
    enum Attribute
    {
        IsRead,         /*数据从响应方流向请求方*/
        IsWrite,        /*数据从请求方流向响应方*/
        IsUpgrade,      /*权限升级*/
        IsInvalidate,   /*使得其他副本失效*/
        IsClean,        //!< Cleans any existing dirty blocks
        NeedsWritable,  /*需可写副本完成操作*/
        IsRequest,      //!< Issued by requester
        IsResponse,     //!< Issue by responder
        NeedsResponse,  //!< Requester needs response from target
        IsEviction,
        IsSWPrefetch,
        IsHWPrefetch,
        IsLlsc,         //!< Alpha/MIPS LL or SC access
        IsLockedRMW,    //!< x86 locked RMW access
        HasData,        //!< There is an associated payload
        IsError,        //!< Error response
        IsPrint,        //!< Print state matching address (for debugging)
        IsFlush,        //!< Flush the address from caches
        FromCache,      //!< Request originated from a caching agent
        NUM_COMMAND_ATTRIBUTES
    };

    static constexpr unsigned long long
    buildAttributes(std::initializer_list<Attribute> attrs)
    {
        unsigned long long ret = 0;
        for (const auto &attr: attrs)
            ret |= (1ULL << attr);
        return ret;
    }

    /**
     * 
     */
    struct CommandInfo
    {
        /// Set of attribute flags.
        const std::bitset<NUM_COMMAND_ATTRIBUTES> attributes; /*属性位掩码*/
        /*对应的响应命令*/
        const Command response;
        /*字符串描述符*/
        const std::string str;

        CommandInfo(std::initializer_list<Attribute> attrs,
                Command _response, const std::string &_str) :
            attributes(buildAttributes(attrs)), response(_response), str(_str)
        {}
    };

    /// Array to map Command enum to associated info.
    static const CommandInfo commandInfo[];

  private:

    Command cmd;

    bool
    testCmdAttrib(MemCmd::Attribute attrib) const
    {
        return commandInfo[cmd].attributes[attrib] != 0;
    }

  public:

    bool isRead() const            { return testCmdAttrib(IsRead); }
    bool isWrite() const           { return testCmdAttrib(IsWrite); }
    bool isUpgrade() const         { return testCmdAttrib(IsUpgrade); }
    bool isRequest() const         { return testCmdAttrib(IsRequest); }
    bool isResponse() const        { return testCmdAttrib(IsResponse); }
    bool needsWritable() const     { return testCmdAttrib(NeedsWritable); }
    bool needsResponse() const     { return testCmdAttrib(NeedsResponse); }
    bool isInvalidate() const      { return testCmdAttrib(IsInvalidate); }
    bool isEviction() const        { return testCmdAttrib(IsEviction); }
    bool isClean() const           { return testCmdAttrib(IsClean); }
    bool fromCache() const         { return testCmdAttrib(FromCache); }

    /**
     * A writeback is an eviction that carries data.
     */
    bool isWriteback() const       { return testCmdAttrib(IsEviction) &&
                                            testCmdAttrib(HasData); }

    /**
     * Check if this particular packet type carries payload data. Note
     * that this does not reflect if the data pointer of the packet is
     * valid or not.
     */
    bool hasData() const        { return testCmdAttrib(HasData); }
    bool isLLSC() const         { return testCmdAttrib(IsLlsc); }
    bool isLockedRMW() const    { return testCmdAttrib(IsLockedRMW); }
    bool isSWPrefetch() const   { return testCmdAttrib(IsSWPrefetch); }
    bool isHWPrefetch() const   { return testCmdAttrib(IsHWPrefetch); }
    bool isPrefetch() const     { return testCmdAttrib(IsSWPrefetch) ||
                                         testCmdAttrib(IsHWPrefetch); }
    bool isError() const        { return testCmdAttrib(IsError); }
    bool isPrint() const        { return testCmdAttrib(IsPrint); }
    bool isFlush() const        { return testCmdAttrib(IsFlush); }

    bool
    isDemand() const
    {
        return (cmd == ReadReq || cmd == WriteReq ||
                cmd == WriteLineReq || cmd == ReadExReq ||
                cmd == ReadCleanReq || cmd == ReadSharedReq);
    }

    Command
    responseCommand() const
    {
        return commandInfo[cmd].response;
    }

    /// Return the string to a cmd given by idx.
    const std::string &toString() const { return commandInfo[cmd].str; }
    int toInt() const { return (int)cmd; }

    MemCmd(Command _cmd) : cmd(_cmd) { }
    MemCmd(int _cmd) : cmd((Command)_cmd) { }
    MemCmd() : cmd(InvalidCmd) { }

    bool operator==(MemCmd c2) const { return (cmd == c2.cmd); }
    bool operator!=(MemCmd c2) const { return (cmd != c2.cmd); }
};

```

```cpp
/**
 * A Packet is used to encapsulate a transfer between two objects in
 * the memory system (e.g., the L1 and L2 cache).  (In contrast, a
 * single Request travels all the way from the requestor to the
 * ultimate destination and back, possibly being conveyed by several
 * different Packets along the way.)
 */
class Packet : public Printable, public Extensible<Packet>
{
  public:
    typedef uint32_t FlagsType;
    typedef gem5::Flags<FlagsType> Flags;

  private:
    enum : FlagsType
    {
        /* 复制数据包时需要保留的标志位集合（低8位）*/
        COPY_FLAGS             = 0x000000FF,

        /* 创建响应包时使用的标志位集合0b1001*/
        RESPONDER_FLAGS        = 0x00000009,

        /* 表示数据块有共享者（不可写） 最低1bit*/
        HAS_SHARERS            = 0x00000001,

        /* 用于多级一致性的原子snoop  最低第2bit*/
        EXPRESS_SNOOP          = 0x00000002,

        /* 响应缓存通知：响应前已有可写副本*/
        RESPONDER_HAD_WRITABLE = 0x00000004,

        /* 表示缓存正在响应snoop请求*/
        CACHE_RESPONDING       = 0x00000008,

        /* 写回/写清操作需要向下游传播*/
        WRITE_THROUGH          = 0x00000010,

        /* 缓存维护操作响应协调标志*/
        SATISFIED              = 0x00000020,

        /* 事务失败标记（已从缓存层次返回）*/
        FAILS_TRANSACTION      = 0x00000040,

        /* 请求来自事务执行模式的CPU*/
        FROM_TRANSACTION       = 0x00000080,

        /* 地址字段有效标志， 数据大小字段有效标志*/
        VALID_ADDR             = 0x00000100,
        VALID_SIZE             = 0x00000200,

        /* 数据指针指向不应该释放的静态数据*/
        STATIC_DATA            = 0x00001000,
        /* 数据指针指向需要释放的动态数组（使用delete[]）*/
        DYNAMIC_DATA           = 0x00002000,

        /* 抑制功能访问失败的错误 */
        SUPPRESS_FUNC_ERROR    = 0x00008000,

        /* 数据块已缓存的标志（抑制预取/驱逐包）*/
        BLOCK_CACHED          = 0x00010000
    };
    ...
}

```

```cpp
class Packet : public Printable, public Extensible<Packet>
{
    ...
    /* 包状态标志的集合*/
    Flags flags;

  public:
    typedef MemCmd::Command Command;

    /*内存操作命令枚举（来自MemCmd枚举）*/
    MemCmd cmd;
    /*包的全局唯一标识符*/
    const PacketId id;

    /*原始请求的智能指针*/
    RequestPtr req;

  private:
   /**
    * 数据传输指针
    */
    PacketDataPtr data;

    /* 请求地址（虚拟/物理）*/
    Addr addr;

    /* 是否访问安全内存空间（arm TrustZone）*/
    bool _isSecure;

    /* 请求数据大小*/
    unsigned size;

    /**
     * 用于功能读操作的字节有效性标记
     */
    std::vector<bool> bytesValid;

    /**
     * 服务质量优先级
     */
    uint8_t _qosValue;

    // hardware transactional memory

    /* 硬件事务内存失败原因*/
    HtmCacheFailure htmReturnReason;

    /* HTM事务全局唯一标识符（用于调试）*/
    uint64_t htmTransactionUid;

  public:

    /* 时序控制成员，从接受包到传输头部的额外延迟（用于建模Crossbar延迟）*/
    uint32_t headerDelay;

    /* snooping造成的额外延迟*/
    uint32_t snoopDelay;

    /**
     * 从接受包到完成有效载荷传输的总延迟（含headerDelay）
     */
    uint32_t payloadDelay;

    /**
     * SenderState：
     * 是一个单向链表，保存前一个状态的指针（形成状态栈）
     * 可以根据SenderState获得上游信息
     */
    struct SenderState
    {
        SenderState* predecessor;
        SenderState() : predecessor(NULL) {}
        virtual ~SenderState() {}
    };

    /**
     * PrintReqState扩展类
     */
    class PrintReqState : public SenderState
    {
      private:
        /**
         * An entry in the label stack.
         */
        struct LabelStackEntry
        {
            const std::string label;    /*当前层级标签*/
            std::string *prefix;        /*当前前缀指针*/
            bool labelPrinted;          /*标签是否已打印*/
            LabelStackEntry(const std::string &_label, std::string *_prefix);
        };

        typedef std::list<LabelStackEntry> LabelStack;  /*标签栈*/
        LabelStack labelStack;                          

        std::string *curPrefixPtr;                      /*当前前缀指针*/

      public:
        std::ostream &os;                               /*输出流引用*/
        const int verbosity;                            /*详细级别*/

        PrintReqState(std::ostream &os, int verbosity = 0);
        ~PrintReqState();

        /**
         * Returns the current line prefix.
         */
        const std::string &curPrefix() { return *curPrefixPtr; }

        /**
         * Push a label onto the label stack, and prepend the given
         * prefix string onto the current prefix.  Labels will only be
         * printed if an object within the label's scope is printed.
         */
        void pushLabel(const std::string &lbl,
                       const std::string &prefix = "  ");

        /**
         * Pop a label off the label stack.
         */
        void popLabel();

        /**
         * Print all of the pending unprinted labels on the
         * stack. Called by printObj(), so normally not called by
         * users unless bypassing printObj().
         */
        void printLabels();

        /**
         * Print a Printable object to os, because it matched the
         * address on a PrintReq.
         */
        void printObj(Printable *obj);
    };

    /**
     * This packet's sender state.  Devices should use dynamic_cast<>
     * to cast to the state appropriate to the sender.  The intent of
     * this variable is to allow a device to attach extra information
     * to a request. A response packet must return the sender state
     * that was attached to the original request (even if a new packet
     * is created).
     */
    SenderState *senderState;

    /**
     * Push a new sender state to the packet and make the current
     * sender state the predecessor of the new one. This should be
     * prefered over direct manipulation of the senderState member
     * variable.
     *
     * @param sender_state SenderState to push at the top of the stack
     */
    void pushSenderState(SenderState *sender_state);

    /**
     * Pop the top of the state stack and return a pointer to it. This
     * assumes the current sender state is not NULL. This should be
     * preferred over direct manipulation of the senderState member
     * variable.
     *
     * @return The current top of the stack
     */
    SenderState *popSenderState();

    /**
     * Go through the sender state stack and return the first instance
     * that is of type T (as determined by a dynamic_cast). If there
     * is no sender state of type T, NULL is returned.
     *
     * @return The topmost state of type T
     */
    template <typename T>
    T * findNextSenderState() const
    {
        T *t = NULL;
        SenderState* sender_state = senderState;
        while (t == NULL && sender_state != NULL) {
            t = dynamic_cast<T*>(sender_state);
            sender_state = sender_state->predecessor;
        }
        return t;
    }

};

```

