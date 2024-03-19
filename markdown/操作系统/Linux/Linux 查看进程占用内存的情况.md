# Linux 查看进程占用内存的情况

## 背景
原来跑着的MySQL挂了, 最终定位问题是机器内存不够了, 为什么内存不够了呢? 想到前两天部署的服务, 所以想看下该服务的当前的内存占用情况。
> [参考文章-田培融-链接](https://blog.csdn.net/u011296165/article/details/95066920)

## Linux 查看某个进程占用内存的情况
**注意 : 这里显示的内存信息是系统层面的进程的内存使用情况**
Linux中查看某个进程占用内存的情况，执行如下命令即可，将其中的[pid]替换成相应进程的PID号：
```shell
cat /proc/pid/status
```
说明
/proc/[pid]/status中所保存的信息除了内存信息，还包括进程IDs、信号等信息，此处暂时只介绍内存相关的信息。
字段 说明

- VmPeak 进程所使用的虚拟内存的峰值
- VmSize 进程当前使用的虚拟内存的大小
- VmLck 已经锁住的物理内存的大小（锁住的物理内存不能交换到硬盘）
- VmHWM 进程所使用的物理内存的峰值
- VmRSS 进程当前使用的物理内存的大小
- VmData 进程占用的数据段大小
- VmStk 进程占用的栈大小
- VmExe 进程占用的代码段大小（不包括库）
- VmLib 进程所加载的动态库所占用的内存大小（可能与其它进程共享）
- VmPTE 进程占用的页表大小（交换表项数量）
- VmSwap 进程所使用的交换区的大小

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240319232433.png#id=l92k6&originHeight=816&originWidth=545&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
