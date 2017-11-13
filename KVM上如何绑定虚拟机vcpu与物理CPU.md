# KVM上如何绑定虚拟机vcpu与物理CPU



两种方式：

1、taskset

**Taskset命令设置某虚拟机在某个固定cpu上运行**

```
taskset -p000000000000000000000000000000000000100 95090
解释：设置95090这个进程，在cpu8上运行
```



可以使用命令来查看对应进程在哪个cpu上运行

```
ps -e -o pid,args,psr
```



2、使用virsh的命令vcpupin来绑定（Pin guest domain virtual CPUs to physical host CPUs）

​	

```
绑定命令：virsh vcpupin 4 0 8：绑定domain 4  的vcpu0 到物理CPU8
```





Taskset和vcpupin区别：

Taskset是以task（也就是虚拟机）为单位，也就是以虚拟机上的所有cpu为一个单位，与物理机上的cpu进行绑定，它不能指定虚拟机上的某个vcpu与物理机上某个物理cpu进行绑定，其粒度较大。

Vcpupin命令就可以单独把虚拟机上的vcpu与物理机上的物理cpu进行绑定。

比如vm1有4个vcpu（core），物理机有8个cpu（8个core，假如每个core一个线程），taskset能做到把4个vcpu同时绑定到一个或者多个cpu上，但vcpupin能把每个vcpu与每个cpu进行绑定。