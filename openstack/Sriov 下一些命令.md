Sriov 下一些命令


1.使用如下命令将网卡的VF数量设置为63个
 ```shell
 echo 63 > /sys/class/net/eth48/devices/SR-IOV_numfs
 ```
2.使用如下命令确认VF数量为63个

 ```shell
ip l sh eth48 |grep vf |wc -l
 ```



3、设置sriov的vf口的mac地址，qos，vlan等等信息

```shell
ip link set dev eth1 vf 1 
可使用ip link help 查看
```



4、查看vf口mac地址，等等

```shell
ip link sh   #查看网卡信息

lspci |grep Eth  #查看pci总线信息

ethtool -S eth1  #查看sriov 网卡的收发统计，收发队列等等信息
```

5、查看网口在的numa节点，等等信息

```shell
cat /sys/class/net/enp130s0f0/device/numa_node

cat /sys/class/net/enp130s0f0/device/virtfn1/numa_node
```



5、网络性能测试等工具

```shell
 iperf -c $IP -u -t 3000 -i 1 -l 64 -b 1000m  #客户端
 iperf -s -u  #服务端
 nmon  #查看各种监控
```











