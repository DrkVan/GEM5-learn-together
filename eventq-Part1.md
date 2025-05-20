# GEM5 Minor Core 解析

这一部分讲解GEM5模拟器里面的event-queue，代码位于gem5/src/sim/eventq.hh & eventq.cc，我们合二为一，边看代码边学习

## event-queue-Part1

这一部分比较长，我会拆成几段来学习，按照eventq.hh文件的先后顺序来学习：

```cpp
namespace gem5
{
/*前向声明，无需提供完整定义，声明EventQueue和BaseGlobalEvent类的存在*/
class EventQueue;       
class BaseGlobalEvent;

/*全局的Tick变量，其中Tick是GEM5里面表示仿真时间的整数类型*/
extern Tick simQuantum;

/*记录当前已分配的主事件队列数量*/
extern uint32_t numMainEventQueues;

/*一个指针数组，存储所有主事件队列的指针，GEM5支持多事件队列并行执行，每个队列可能对应一个线程或处理单元*/
extern std::vector<EventQueue *> mainEventQueue;

/*__thread关键字表示这是一个线程本地存储(Thread-Local Storage)变量*/
/*每个线程都有自己的 当前事件队列 _curEventQueue，无需加锁即可访问*/
extern __thread EventQueue *_curEventQueue;

/*表示当前是否处于并行执行的模式*/
/*用于控制事件队列是并行处理（多线程）还是串行处理（单线程）*/
extern bool inParallelMode;

/*获取事件队列函数，根据给定的index返回对应的事件队列*/
/*如果索引在现有队列范围内，直接返回对应的EventQueue，如果超出当前队列数量但有效，会动态分配新的队列*/
EventQueue *getEventQueue(uint32_t index);

/*获取当前事件队列*/
/*直接返回线程本地的当前事件队列指针*/
inline EventQueue *curEventQueue() { return _curEventQueue; }
/*设置线程本地的当前事件队列指针为q*/
/*用于切换当前线程的事件队列（处理不同事件队列时切换上下文）*/
inline void curEventQueue(EventQueue *q);

...

```

看到这里我们可以揣摩一下这些全局变量或函数的设计意图
1. 多队列并行：通过mainEventQueue数组支持多个事件队列，允许并行处理事件
2. 线程本地存储：_curEventQueue使用TLS避免锁竞争，提高多线程性能
3. simQuantum确保跨队列事件的事件一致性，防止事件发生顺序的混乱
4. getEventQueue按需分配队列，支持灵活的资源管理

这里再补充一些信息：
Tick是uint64_t的别名，时间轴上的时间单位就是Tick。
事件队列同步： 当队列之间需要交互时（例如队列A的事件触发队列B的事件），必须通过simQuantum来保证事件间隔，避免时间倒退或竞争。
当inParallelMode为true时，应该是为了使用多线程来处理事件队列，为false时退化为单线程模式。

接下来是一个类：EventBase，我们来看看这个类：

```cpp
/* */
class EventBase
{
  protected:
    typedef unsigned short FlagsType;
    typedef ::gem5::Flags<FlagsType> Flags;
    ...
}
```

类的声明开局就是Flags，我们有必要去看看Flags这个类，看看大概他起什么作用，我们来看flags.hh:
```cpp
/*Flags是一个通用的位标志管理模板类，用于封装对整数类型的位操作*/
template <typename T>
class Flags
{
  private:
    /*这里是一个类型检查，要求模板T必须是一个无符号整数，uint8_t、uint16_t等，确保位操作安全性（有符号数要处理符号位）*/
    static_assert(std::is_unsigned_v<T>, "Flag type must be unsigned");

    /*底层存储位标志的无符号整数*/
    T _flags;

  public:
    /*获取模板的类型*/
    typedef T Type;

    /*构造函数，默认初始化所有位为0，也可以让你通过 Flags<uint8_t>(0x01)这样来创建对象*/
    Flags(Type flags=0) : _flags(flags) {}

    /*隐式转换类型符，允许将Flags对象隐式转换为底层整数类型，比如从uint8_t转换为uint16_t*/
    operator const Type() const { return _flags; }
    /*赋值运算符，直接设置底层值，覆盖当前所有位状态。*/
    const Flags<T> &
    operator=(T flags)
    {
        _flags = flags;
        return *this;
    }
    
    /* 掩码后至少有一个位被设置了 */
    bool isSet(Type mask) const { return (_flags & mask); }

    /* 掩码后所有位都被设置*/
    bool allSet(Type mask) const { return (_flags & mask) == mask; }

    /* 掩码后所有位都没被设置*/
    bool noneSet(Type mask) const { return (_flags & mask) == 0; }

    /* 清楚所有的位*/
    void clear() { _flags = 0; }

    /* 根据掩码，清除位*/
    void clear(Type mask) { _flags &= ~mask; }

    /* 设置mask下的位*/
    void set(Type mask) { _flags |= mask; }

    /* 根据条件设置或清除位，如果condition为true就是set，否则就是clear*/
    void
    set(Type mask, bool condition)
    {
        condition ? set(mask) : clear(mask);
    }

    /* 将掩码下传进来的flag值，覆盖掉_flags*/
    void
    replace(Type flags, Type mask)
    {
        _flags = (_flags & ~mask) | (flags & mask);
    }
};

```

到这里Flags就解析完了，看上去像是对uint8_t,uint32_t等数据类型进行了拓展，我们可以思考一下哪些情景会用到这些提供的API：
1. 假设一个事件需要被调度，那么可以调用flags.set(Scheduled | Managed)。
2. 如果一个事件要被取消调度，那么可以使用flags.clear(Scheduled)。
3. 查询一个事件是否被调度，可以调用flags.isSet(Scheduled)。

具体的使用我们可以在后面的设计实例中获得更深刻的理解，这里可以对我们思考“如何设计模拟器事件仿真”这个问题有一些启发。

这里还有一些设计风格上的启发：就是用实际的名称set(),isSet()来代替位操作运算符会更易懂，因为位操作运算符确实容易令人感到困惑。

我们再回到EventBase类的开头:FlagsType是unsigned short类型，是无符号16bit数，然后作为模板参数去构造Flags这个标志位，作为EventBase里面的一个成员。
你可以理解为EventBase里有一个unsigned short类型的成员，不过这个成员支持一些API如set()，会帮助人更形象的理解event调度的过程。

```cpp
class EventBase
{
    ...
    /*我们对Flags这个类进行分析的时候已经能感知到设计者要把他的巧思融合到位操作里，我们来看这些编码*/
    /*0x3f也就是0b0000_0000_0011_1111,也就是低6bit是公开可读的标志位*/
    static const FlagsType PublicRead    = 0x003f; 
    /*0x1d也就是0b0000_0000_0001_1101，也就是这些bit可以公开写的标志位*/
    static const FlagsType PublicWrite   = 0x001d; 
    /*0x1也就是0b1，说明低1bit为1的话，说明事件已经被取消*/
    static const FlagsType Squashed      = 0x0001;
    /*0x2也就是0b10，说明低第二bit为1的话，说明事件被调度*/
    static const FlagsType Scheduled     = 0x0002;
    /*0x4也就是0b100，说明低第三bit为1的话，说明事件被管理，这里的被管理指的是“被生命周期管理器管理”，可以自动释放内存*/
    static const FlagsType Managed       = 0x0004;
    /*AutoDelete 被赋值为Manged，AutoDelete意为调度完成后自动删除事件对象，这也好理解，都被自动释放内存了那就是AutoDelete*/ 
    static const FlagsType AutoDelete    = Managed; 
    /*Reserved0为0x8，也就是0b1000，原代码注释里说过去被用于自动序列化，但是被废弃了，处于版本兼容性考虑保留了下来*/
    static const FlagsType Reserved0     = 0x0008;
    /*这里是0b0001_0000，这里是标志模拟退出的特殊事件*/
    static const FlagsType IsExitEvent   = 0x0010;
    /*这里是0b0010_0000,说明事件处于主事件队列，以此区分主队列与线程本地队列，确保关键事件在主队列中执行*/
    static const FlagsType IsMainQueue   = 0x0020; // on main event queue
    /*初始化验证，也就是验证完成时设置的值，通过这个值来确定事件对象是否被正确的初始化，避免使用未初始化的对象（相信大家都被未初始化的bug恶心过）*/
    static const FlagsType Initialized   = 0x7a40; // somewhat random bits
    /*提取高10位用于初始化验证，0xFFC0=0b1111_1111_1100_0000,前面提到了0x7a40=0b0111_1010_0100_0000*/
    /*这里你发现刚好设计者的“检查初始化”的意图在这里得到了体现*/
    static const FlagsType InitMask      = 0xffc0; // mask for init bits
    ...
}
```

到这里我们可以深入思考一下这些状态码，首先设计者先为位操作提供了一些API，然后紧接着设计了一些特别的编码。
总结一下设计意图，首先是16bit的无符号数：
1. 低第1bit表示是否事件被取消，可读可写
2. 低第2bit表示是否事件被调度，仅可读
3. 低第3bit表示事件是否生命周期自动管理，可读可写
4. 低第4bit是一个被弃用的一个设计，可读可写
5. 低第5bit是标志是否是一个退出的事件，可读可写
6. 低第6bit是标志事件是否是主队列事件，仅可读
7. 0x7a40和0xffc0编码的设计是为了确保事件被初始化

是一个很好的编码风格实例，避免容易令人感到困惑的代码。

```cpp
class EventBase
{
    ...
  public:
    /* 方便版本管理，并且对数据类型赋予了含义*/
    /* 提供了127个优先级别，数值越小优先级越高，负数大于整数*/
    typedef int8_t Priority;

    /* SCHAR_MIN是Signed-Char Min，是-128，而SCHAR_MAX是127*/
    static const Priority Minimum_Pri =          SCHAR_MIN;

    /* 启用调试跟踪（最高优先级），在周期开始时立即启用追踪*/
    static const Priority Debug_Enable_Pri =          -101;

    /* 调试时断点触发，紧随其后处理断点，确保调试器捕获所有操作，debug应该优先于系统操作*/
    static const Priority Debug_Break_Pri =           -100;

    /* 切换CPU时，需要先取消旧CPU的Tick事件，再调度新CPU的Tick，确保切换操作在时钟周期事件前完成*/
    static const Priority CPU_Switch_Pri =             -31;

    /* 延迟写回，延迟的跨集群写回需优先于常规写回（默认优先级为0） */
    static const Priority Delayed_Writeback_Pri =       -1;

    /* 默认优先级，如内存访问、计算指令的默认优先级*/
    static const Priority Default_Pri =                  0;

    /* 调整电压频率后更新统计信息，需要等待所有状态更新完成*/
    static const Priority DVFS_Update_Pri =             31;

    /* 序列化优先级，序列化模拟状态（生成检查点），需要在CPU Tick前完成*/
    static const Priority Serialize_Pri =               32;

    /* CPU Tick事件，CPU时钟周期推进，在写回操作后执行，保证数据正确性*/
    static const Priority CPU_Tick_Pri =                50;

    /* CPU线程退出，在CPU_Tick之后处理，避免提前终止未完成操作*/
    static const Priority CPU_Exit_Pri =                64;

    /* 统计事件，周期性统计信息输出或重置计数器*/
    static const Priority Stat_Event_Pri =              90;

    /* 进程通知，通知外部模拟进度，比如用于UI更新*/
    static const Priority Progress_Event_Pri =          95;

    /* 模拟退出*/
    static const Priority Sim_Exit_Pri =               100;

    /* 最高正数优先级*/
    static const Priority Maximum_Pri =          SCHAR_MAX;
};

```

这里可以总结一下，并且学习一下设计者的巧思：

1. 分层管理信息：调试级 > 系统级 > 默认级 > 状态级 > 核心级 > 统计级 > 退出级， 其实在systemC和UVM框架的设计里也有类似的做法。
2. 扩展性：数值分布这么稀疏，我觉得应该是为了扩展性考虑，以后如果扩展优先级，这里要留出空间
3. 退出顺序： 把退出模拟放在最后，是为了确保无资源泄露。

假设同一周期内发生了以下需要模拟的事件：断点调试、CPU切换、常规指令、CPU Tick、统计输出，虽然真实的物理系统就是Tick推进波形。
但是我们从模拟器的角度看去，要模拟的优先级应该为断点调试->CPU切换->常规指令->CPU Tick->统计输出。

剩下的就是我的自我思考，可以供你参考：
Q1:为什么Delayed_WriteBack要优先于常规写回？
A: 延迟写回需要优先处理来避免死锁或数据冲突。

Q2:我们要自己加优先级，该怎么加？
A: 调试相关：<-100，系统操作：-31 ~ -1，普通事件：0~30，状态更新：31~49，核心事件：50~89，统计/退出：>90。

Q3:为什么Serialize的优先级要比CPU Tick优先？
A: Serialize需要捕获完整的CPU状态，如果在CPU Tick之后进行，可能遗漏周期内未完成的操作。

Flags类和EventBase类的设计可以体现出很多巧思，学习优秀的设计实例总是能获得很大的帮助，这里先休息一下。eventQ的剩余内容放在后面。