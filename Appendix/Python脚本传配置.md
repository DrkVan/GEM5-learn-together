# 如何使用python脚本来将Param传进C++

以MinorCore为例子，

比如你找到了BaseMinorCPUParams，那么你就找BaseMinorCPU.py，这个python脚本里能找到一些参数

不会找BaseMinorCPUParams？你可以索引Params，然后某一个*Params被继承，那么这个大概就是你想要的

你可以在这里加入你需要的自定义参数，记住自定义参数不仅要在python里添加，也要在.hh文件里添加

然后在你的GEM5仿真脚本里面加上BaseMinorCPU.${your_config} = ${your_value}

如果是已经存在的参数，那么你可以在python脚本里直接BaseMinorCPU.${your_config} = ${your_value}