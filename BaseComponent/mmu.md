# GEM5 Minor Core 解析

这一部分是对GEM5基础组件BaseMMU的解析：

## MMU

BaseMMU的核心职责是查询TLB或页表来将虚拟地址转换为物理地址。
根据访问模式Read/Write/Execute来验证操作的合法性。
MMU的工作就是协调TLB、页表遍历器和Cache等组件。


mmu.hh和mmu.cc位于gem5/src/arch/generic下面，是MMU的一种基础实现，下面进行二合一地解析：

边看代码边解析：

```cpp
/*前向声明，Base TLB是TLB基类，BaseMMU与之协同*/
class BaseTLB;
/* 基本MMU实现，继承SimObject*/
class BaseMMU : public SimObject
{
  public:
    /*内存访问模式：读、写、执行*/
    /*为什么是Read,Write,Execute三种？*/
    /*一般而言，取指令只需要Read+Execute权限，如果赋予Write权限攻击者可能将恶意指令写入*/
    /*数据段通常只需要Read/Write权限，如果允许Execute权限，则恶意代码可能被写入数据区来执行*/
    enum Mode { Read, Write, Execute };
    /*定义Translation类，用于处理地址转换的生命周期管理*/
    class Translation
    {
      public:
        virtual ~Translation()
        {}

        /* 纯虚函数，衍生类需要实现延迟处理逻辑*/
        virtual void markDelayed() = 0;

        /* 地址转换完成后的回调函数，fault是转换过程中产生的异常，req为内存请求对象实例*/
        /* tc为线程上下文，mode为访问模式，衍生类需要提供地址转换完成后的实现*/
        virtual void finish(const Fault &fault, const RequestPtr &req,
                            ThreadContext *tc, BaseMMU::Mode mode) = 0;

        /* 返回当前转换请求是否已经失效，比如流水线冲刷导致指令取消*/
        virtual bool squashed() const { return false; }  /*默认是未失效的*/
    };
    ...
}
```

Translation作为MMU内部的传播介质，衍生类需要提供延迟处理的实现，以及finish（成功的翻译）的实现，squashed的实现（是否失效）。

继续：

```cpp
class BaseMMU : public SimObject
{
    ...
  protected:
    /*接上文，Params是通过python脚本传过来的一堆参数*/
    typedef BaseMMUParams Params;
    /*构造函数，将data-TLB给dtb,inst-TLB给itb*/
    BaseMMU(const Params &p)
      : SimObject(p), dtb(p.dtb), itb(p.itb)
    {}
    /*获得Inst/Data的TLB实例*/
    BaseTLB*
    getTlb(Mode mode) const
    {
        if (mode == Execute)
            return itb;
        else
            return dtb;
    }

  public:
    /* 初始化TLB层次结构，在系统初始化阶段调用，遍历TLB层次结构，填充指令、数据TLB的初始页表*/
    /* 并且与页表遍历器建立关联*/
    void init() override
    {
        /*
        初始化的默认实现，首先定义lambda函数traverse_hierarchy，用于遍历TLB层级并分类存储
        从starter开始逐级遍历TLB
        */
        auto traverse_hierarchy = [this](BaseTLB *starter) {
            /*这个for循环是一个链表的结构，来实现逐级init*/
            for (BaseTLB *tlb = starter; tlb; tlb = tlb->nextLevel()) {
                /*根据TLB的类型将其插入集合，这里三种类型Inst，Data和Unified很好理解*/
                switch (tlb->type()) {
                case TypeTLB::instruction:
                    if (instruction.find(tlb) == instruction.end())
                        instruction.insert(tlb);
                    break;
                case TypeTLB::data:
                    if (data.find(tlb) == data.end())
                        data.insert(tlb);
                    break;
                case TypeTLB::unified:
                    if (unified.find(tlb) == unified.end())
                        unified.insert(tlb);
                    break;
                default:
                    panic("Invalid TLB type\n");
                }
            }
        };
        /*Inst-TLB初始化 和 Data-TLB初始化*/
        traverse_hierarchy(itb);
        traverse_hierarchy(dtb);
    }
    /* 刷新所有TLB，清空所有TLB条目，包括指令和数据TLB*/
    /* 比如进程切换时，需要确保旧进程的地址映射被清除*/
    virtual void flushAll();
    /* 重置，使MMU恢复到初始状态，模拟器重置时调用*/
    virtual void reset();
    /* 指定页表项使失效，根据虚拟地址和ASID来使得对应TLB条目失效*/
    void demapPage(Addr vaddr, uint64_t asn);
    /* 这里的三种转换方法对应三种协议，Atomic的、Timing的、Functional的*/
    /* Atomic无时序模拟地完成地址转换，是一种快速模拟，无需考虑流水线时序*/
    virtual Fault
    translateAtomic(const RequestPtr &req, ThreadContext *tc,
                    Mode mode)
    {
        /*根据mode选择TLB，如果是Execute返回Inst-TLB，否则都是Data-TLB*/
        /*Atomic协议下的翻译就是直接翻译*/
        return getTlb(mode)->translateAtomic(req, tc, mode);
    }            
    /*有时序地处理翻译，通过Translation对象管理延迟和回调，是一种精确的周期模拟*/
    virtual void
    translateTiming(const RequestPtr &req, ThreadContext *tc,
                    Translation *translation, Mode mode)
    {
        /*是否delay的标志位*/
        bool delayed;
        assert(translation);
        /*进行翻译，如果有错误返回错误类型*/
        Fault fault = translate(req, tc, translation, mode, delayed);
        if (!delayed)
            /*如果翻译成功，调用finish方法*/
            translation->finish(fault, req, tc, mode);
        else
            translation->markDelayed();
    }
    /*功能性的翻译，仅验证地址转换是否合法，不修改系统状态，调试性的工具*/
    virtual Fault
    translateFunctional(const RequestPtr &req, ThreadContext *tc,
                        Mode mode);
    /*验证地址有效性，验证虚拟地址是否可以转换为物理地址，并且返回有效的地址*/
    virtual Addr getValidAddr(Addr vaddr, ThreadContext *tc, Mode mode);
    ...
}

```

感觉这部分没啥特别的，都非常直观，for循环来进行链表递归的写法倒是学到了，很优雅。

```cpp
class BaseMMU : public SimObject
{
    ...
    /*继续上一部分*/
    /*
    MMUTranslation继承自TranslationGen，用于批量生成虚拟地址到物理地址的转换。
    提供一种高效的方式遍历虚拟地址范围，逐页转换为物理地址，而无需按地址调用
    */
    class MMUTranslationGen : public TranslationGen
    {
      private:
        /*线程上下文（包含页表基址等状态）*/
        ThreadContext *tc;
        /*上下文ID，区分不同线程*/
        ContextID cid;
        /*MMU实例*/
        BaseMMU *mmu;
        /*访问权限Read/Write/Execute*/
        BaseMMU::Mode mode;
        /*请求标志，如缓存类型、特权级*/
        Request::Flags flags;
        /*页大小*/
        const Addr pageBytes;
        /*将虚拟地址范围range转换为物理地址范围*/
        void translate(Range &range) const override;
        /*实现逻辑：按页对其拆分range，然后对每一个页调用mmu->translateFunctional获得物理页框*/
        

      public:
        /*构造函数*/
        MMUTranslationGen(Addr page_bytes, Addr new_start, Addr new_size,
                ThreadContext *new_tc, BaseMMU *new_mmu,
                BaseMMU::Mode new_mode, Request::Flags new_flags):
                    TranslationGen(new_start, new_size), tc(new_tc), cid(tc->contextId()),
                    mmu(new_mmu), mode(new_mode), flags(new_flags),
                    pageBytes(page_bytes){};

    };

    /* 生成一个MMUTranslation实例，用于遍历虚拟地址范围[start,start+size)的物理地址转换*/
    virtual TranslationGenPtr translateFunctional(Addr start, Addr size,
            ThreadContext *tc, BaseMMU::Mode mode, Request::Flags flags) = 0;

    /* 在地址转换完成后，验证和修正物理地址的合法性*/
    virtual Fault
    finalizePhysical(const RequestPtr &req, ThreadContext *tc,
                     Mode mode) const;
    /*接管另一个MMU实例的状态（进程切换时尽可能避免软件仿真开销）*/
    virtual void takeOverFrom(BaseMMU *old_mmu);

  public:
    /*Inst-TLB Data-TLB实例*/
    BaseTLB* dtb;
    BaseTLB* itb;

  protected:
    std::set<BaseTLB*> instruction;  /*所有指令TLB*/
    std::set<BaseTLB*> data;         /*所有数据TLB*/
    std::set<BaseTLB*> unified;      /*统一TLB，如L2TLB*/
}

```

MMU的逻辑到这里结束，还是比较直观的。