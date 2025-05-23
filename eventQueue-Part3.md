# GEM5源码解析

## event-queue-Part3

书接上回，这一部分继续对eventq.hh和eventq.cc合二为一地进行解析。
这一部分简单一点，讲解Event类对操作符的重载，设计的核心目的是定义事件再队列中的排序规则。

```cpp
/* 小于比较，首先是比较触发时间，再比较优先级*/
inline bool
operator<(const Event &l, const Event &r)
{
    return l.when() < r.when() ||
        (l.when() == r.when() && l.priority() < r.priority());
}

/* 大于比较，首先是比较触发时间，再比较优先级*/
inline bool
operator>(const Event &l, const Event &r)
{
    return l.when() > r.when() ||
        (l.when() == r.when() && l.priority() > r.priority());
}

/* 小于等于的重载要注意，并不接受触发时间的小于等于，仅仅是优先级的小于等于*/
inline bool
operator<=(const Event &l, const Event &r)
{
    return l.when() < r.when() ||
        (l.when() == r.when() && l.priority() <= r.priority());
}

/* 大于等于的重载一样，并不是触发时间的大于等于，而是优先级的大于等于*/
inline bool
operator>=(const Event &l, const Event &r)
{
    return l.when() > r.when() ||
        (l.when() == r.when() && l.priority() >= r.priority());
}

/* 触发时间相等且优先级相等*/
inline bool
operator==(const Event &l, const Event &r)
{
    return l.when() == r.when() && l.priority() == r.priority();
}

/* 触发时间不相等且优先级不相等*/
inline bool
operator!=(const Event &l, const Event &r)
{
    return l.when() != r.when() || l.priority() != r.priority();
}
```

OK，Event的比较逻辑到这里结束，作为一个小憩，下一个Part讲解EventQueue这个类。

