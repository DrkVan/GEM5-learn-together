# GEM5 Minor Core 解析

这一部分讲解RISCV文件夹下Process.cc和Process.hh代码，合二为一，还是一样边看代码边解释

## Process

边看代码边学习

```cpp

/* 命名空间是loader,loader是一个读数据的工具，会把python脚本指定的elf文件进行读取*/
namespace loader
{
/* 你可以认为ObjectFile就是你读进来的数据，会对ELF文件进行解析*/
class ObjectFile;
} // namespace loader

class System;

class RiscvProcess : public Process
{
  /*RIiscvProcess 继承 Process， 其中Process是基类，Process类可以参阅${FILE_ROOT}/Appendix/Process基类.md*/
  protected:
    /*构造函数，传递由python脚本传递的ProcessParam，并且将objectFile的指针传进去*/
    RiscvProcess(const ProcessParams &params, loader::ObjectFile *objFile);
    template<class IntType>
    /*参数初始化模板方法*/
    void argsInit(int pageSize);

  public:
    /*内存映射方向声明*/
    virtual bool mmapGrowsDown() const override { return false; }

  protected:
    /*随机数生成器*/
    Random::RandomPtr rng = Random::genRandom();
};

class RiscvProcess64 : public RiscvProcess
{
  public:
    RiscvProcess64(const ProcessParams &params, loader::ObjectFile *objFile);

  protected:
    /*初始化CPU寄存器状态*/
    void initState() override; 
};

class RiscvProcess32 : public RiscvProcess
{
  public:
    RiscvProcess32(const ProcessParams &params, loader::ObjectFile *objFile);

  protected:
    /*初始化CPU寄存器状态*/
    void initState() override;
};

#endif // __RISCV_PROCESS_HH__




```