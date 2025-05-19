# GEM5 Minor Core 解析

这一部分是对GEM5基础组件BaseMMU里面的TLB的解析,TLB是RISCV的：

## MMU-TLB

文件位于gem5/src/arch/riscv/，包含tlb.hh和tlb.cc两个文件，下面开始边看代码边解析：

```cpp
/*BaseTLB是所有TLB地基类，定义了通用的TLB接口（如地址翻译、刷新操作等）*/
/*这里是RISCV架构下的TLB具体实现*/
class TLB : public BaseTLB
{
    /*类型别名，EntryList是一个链表，用于管理TLBEntry指针*/
    typedef std::list<TlbEntry *> EntryList;

  protected:
    /*TLB总容量，Entry的数量*/
    size_t size;
    /*存储TLBEntry的容器*/
    std::vector<TlbEntry> tlb; 
    /*TlbEntry树形数据结构*/
    TlbEntryTrie trie;
    /*空闲条目的链表*/
    EntryList freeList;
    /*LRU算法的序列号，用于替换策略*/
    uint64_t lruSeq;
    /*页表遍历器（处理缺页时需要遍历页表）*/
    Walker *walker;
    /*继承自GEM5的统计模块,记录TLB的性能指标*/
    struct TlbStats : public statistics::Group
    {
        TlbStats(statistics::Group *parent);

        statistics::Scalar readHits;
        statistics::Scalar readMisses;
        statistics::Scalar readAccesses;
        statistics::Scalar writeHits;
        statistics::Scalar writeMisses;
        statistics::Scalar writeAccesses;

        statistics::Formula hits;
        statistics::Formula misses;
        statistics::Formula accesses;
    } stats;

  public:
    BasePMAChecker *pma;
    PMP *pmp;

  public:
    typedef RiscvTLBParams Params;


```

未完成，施工中。