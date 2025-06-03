# GEM5源码解析

## timebuf解析

这一部分会轻松一些，是Cycle-Accurate仿真系统里的核心组件timebuf，时序逻辑可以由timbuf内的延迟来表示。
一样的，我们结合源码来进行解析。

```cpp
/*头文件保护，避免重复包含*/
#ifndef __BASE_TIMEBUF_HH__
#define __BASE_TIMEBUF_HH__
/*引入断言、内存操作、STL向量*/
#include <cassert>
#include <cstring>
#include <vector>

/*命名空间*/
namespace gem5
{
/*定义模板类，T为timebuf中存储的数据类型*/
template <class T>
class TimeBuffer
{
  protected:
    /*past为可访问的过去周期数*/
    /*future为可访问的未来周期数*/
    /*size为总缓冲区大小(past+future+1)*/
    /*timebuf编号，用于调试*/
    int past;
    int future;
    unsigned size;
    int _id;
    /*实际分配的内存块（char数组，存放所有T对象）*/
    char *data;
    /*指向每个T对象的指针数组，便于循环队列操作*/
    std::vector<char *> index;
    /*'现在'在循环队列中的位置*/
    unsigned base;
    /*检查索引idx是否合法，在允许的过去和未来范围内*/
    void valid(int idx) const
    {
        assert (idx >= -past && idx <= future);
    }

  public:
    friend class wire;
    /*成员类wire*/
    class wire
    {
        friend class TimeBuffer;
      protected:
        /*指向所属的timebuf*/
        TimeBuffer<T> *buffer;
        /*当前wire指向的时间偏移（相对于'现在'）*/
        int index;
        /*设置wire指向的时间点，检查合法性*/
        void set(int idx)
        {
            buffer->valid(idx);
            index = idx;
        }
        /*可以用TimeBuffer和索引初始化，也可以拷贝构造*/
        wire(TimeBuffer<T> *buf, int i)
            : buffer(buf), index(i)
        { }

      public:
        wire()
        { }

        wire(const wire &i)
            : buffer(i.buffer), index(i.index)
        { }
        /*wire之间赋值*/
        const wire &operator=(const wire &i)
        {
            buffer = i.buffer;
            set(i.index);
            return *this;
        }
        /**/
        const wire &operator=(int idx)
        {
            set(idx);
            return *this;
        }
        /*加偏移*/
        const wire &operator+=(int offset)
        {
            set(index + offset);
            return *this;
        }
        /*减偏移*/
        const wire &operator-=(int offset)
        {
            set(index - offset);
            return *this;
        }
        /*自增自减*/
        wire &operator++()
        {
            set(index + 1);
            return *this;
        }

        wire &operator++(int)
        {
            int i = index;
            set(index + 1);
            return wire(this, i);
        }

        wire &operator--()
        {
            set(index - 1);
            return *this;
        }

        wire &operator--(int)
        {
            int i = index;
            set(index - 1);
            return wire(this, i);
        }
        /*wire的解引用*/
        T &operator*() const { return *buffer->access(index); }
        T *operator->() const { return buffer->access(index); }
    };


  public:
    /*构造函数，分配内存，初始化每个T对象，设置索引*/
    /*data为原始内存块，index保存每个T对象的指针*/
    TimeBuffer(int p, int f)
        : past(p), future(f), size(past + future + 1),
          data(new char[size * sizeof(T)]), index(size), base(0)
    {
        assert(past >= 0 && future >= 0);
        char *ptr = data;
        for (unsigned i = 0; i < size; i++) {
            index[i] = ptr;
            std::memset(ptr, 0, sizeof(T));
            new (ptr) T;
            ptr += sizeof(T);
        }

        _id = -1;
    }
    /*默认构造，用于容器，比如你试着把TimeBuffer塞入另一个容器，需要默认构造*/
    TimeBuffer()
        : data(NULL)
    {
    }
    /*手动析构函数*/
    ~TimeBuffer()
    {
        for (unsigned i = 0; i < size; ++i)
            (reinterpret_cast<T *>(index[i]))->~T();
        delete [] data;
    }
    /*设置和获取TimeBuffer的id*/
    void id(int id)
    {
        _id = id;
    }

    int id()
    {
        return _id;
    }
    /*推进时间，base前进一格，并重置最远的future那一格子（析构旧对象，重建新对象），以此保证TimeBuffer始终只保存past到future范围内的数据*/
    void
    advance()
    {
        if (++base >= size)
            base = 0;

        int ptr = base + future;
        if (ptr >= (int)size)
            ptr -= size;
        (reinterpret_cast<T *>(index[ptr]))->~T();
        std::memset(index[ptr], 0, sizeof(T));
        new (index[ptr]) T;
    }

  protected:
    /*根据当前base和相对时间idx，计算实际在index数组中的位置*/
    inline int calculateVectorIndex(int idx) const
    {
        valid(idx);

        int vector_index = idx + base;
        if (vector_index >= (int)size) {
            vector_index -= size;
        } else if (vector_index < 0) {
            vector_index += size;
        }

        return vector_index;
    }
    
  public:
    /*数据访问*/
    T *access(int idx)
    {
        int vector_index = calculateVectorIndex(idx);

        return reinterpret_cast<T *>(index[vector_index]);
    }
    /*数据访问方式*/
    T &operator[](int idx)
    {
        int vector_index = calculateVectorIndex(idx);

        return reinterpret_cast<T &>(*index[vector_index]);
    }
    /*数据访问方式*/
    const T &operator[] (int idx) const
    {
        int vector_index = calculateVectorIndex(idx);

        return reinterpret_cast<const T &>(*index[vector_index]);
    }
    /*获取指向某个时间点的wire对象*/
    wire getWire(int idx)
    {
        valid(idx);

        return wire(this, idx);
    }

    wire zero()
    {
        return wire(this, 0);
    }
    /*获取缓冲区大小*/
    unsigned getSize()
    {
        return size;
    }
};

}

#endif_
```

可以从代码设计里看出来TimeBuffer是一种环形时间缓冲区，其实我觉得这不太直观。
不过我在有一次面试经历里面构想过这种TimeBuffer实现形式，不过被面试官否了。

更直观更容易被人理解的方式还是一个deque，保存全局时间戳和什么时候可以pop出数据那种。

通过advance推进时间，自动回收和重建最远的future数据。
我觉得结合某种场景来理解会更让人理解。

下面我们来结合场景说明，假设Stage1向Stage2传递信息，Stage2收到信息需要3个Cycle，那么该怎么做呢：
```cpp
struct Data {
    int value;
    bool valid;
    Data() : value(0), valid(false) {}
};

gem5::TimeBuffer<Data> buffer(1, 4);

int delay = 4;
buffer[delay].value = val;
buffer[delay].valid = true;
```
上面片段为Stage1向Stage2发送数据的操作示意。
那么Stage2如何接收来自Stage1的数据呢：
```cpp
if(buffer[0].valid){
    ...
    buffer[0].valid = false;
}
```
然后推进时间
```cpp
buffer.advance();
```
总结一下：
1. 写入就是buffer[delay],表示数据将在delay周期后到达。
2. 读取就是buffer[0]，表示当前周期到达的数据。
3. advance()是每周期推进一次，所有数据自动“前进一格”。

**但是这个设计会存在一个问题**，这种情况又肯定会存在于仿真系统中：

**如果Cycle0时Stage1向Stage2发送一个Delay为4的数据，Cycle1时Stage1又向Stage2发送一个Delay为3的数据,此时:**

我们可以向TimeBuffer的槽位里塞入一个队列，比如
```cpp
TimeBuffer<Data>
```
改为
```cpp
TimeBuffer<std::queue<Data>>
```
这样就可以保存同一时间槽内的多笔数据。

或者，将数据流切分为互不干扰的TimeBuffer。

TimeBuffer的解析就到这里结束了，是一种环形队列，你可以在Minor等设计实例中发现他的用法，并且可以尝试去发现开源代码设计实例里存在的一些缺陷。