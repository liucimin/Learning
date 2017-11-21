Tc 网卡多队列时每个队列配置公平队列sfq

qdisc是队列的概念，在没有自己手动给dev增加队列的时候，默认有一个mq 0:的队列，

class是类的概念，在没有使用tc命令给dev增加队列的时候，默认对所有网卡硬件队列对应了一条

class，而此时默认的队列是fifo，可以参看《Linux的高级路由和流量控制HOWTO》进行理解。

如果我们想单独给各个硬件队列配置qos，可以在对应队列的class里增加sfq qdisc并配置

```shell
[root@node2 ~]# tc -s q ls dev ens11f0

qdisc mq 0: root 

 Sent 0 bytes 0 pkt (dropped 

[root@node2 ~]# tc  c ls dev ens11f0
class mq :1 root 
class mq :2 root  
class mq :3 root 
class mq :4 root 
class mq :5 root 
class mq :6 root 
class mq :7 root 
class mq :8 root 

[root@node2 ~]# tc qd add dev ens11f0 parent :1 sfq

#可以看到class上增加的对应的qdisc
[root@node2 ~]# tc cl sh dev ens11f0
class mq :1 root leaf 8001: 
class mq :2 root 
class mq :3 root 
class mq :4 root 
class mq :5 root 
class mq :6 root 
class mq :7 root 
class mq :8 root 

#可以对应的qdisc是sfq， 8001是主序号
[root@node2 ~]# tc qd sh dev ens11f0
qdisc mq 0: root 
qdisc sfq 8001: parent :1 limit 127p quantum 1514b depth 127 divisor 1024 
```

```shell
#如果给dev的root配置新的队列，原默认队列会不见了
[root@node2 ~]# tc qdisc add dev ens11f0 root handle 1: cbq bandwidth 10Mbit avpkt 1000 cell 8 mpu 64
[root@node2 ~]# 
[root@node2 ~]# tc cl sh dev ens11f0
class cbq 1: root rate 10000Kbit (bounded,isolated) prio no-transmit
[root@node2 ~]# 
[root@node2 ~]# tc qd sh dev ens11f0
qdisc cbq 1: root refcnt 9 rate 10000Kbit (bounded,isolated) prio no-transmit
[root@node2 ~]# tc qdisc del dev ens11f0 root handle 1: cbq bandwidth 10Mbit avpkt 1000 cell 8 mpu 64
[root@node2 ~]# tc cl sh dev ens11f0
class mq :1 root 
class mq :2 root 
class mq :3 root 
class mq :4 root 
class mq :5 root 
class mq :6 root 
class mq :7 root 
class mq :8 root 
[root@node2 ~]# 
```



也可以对对应队列配置更为复杂的qdisc：

```shell
[root@node2 ~]# tc qdisc add dev ens11f0 parent :1 cbq bandwidth 10Mbit avpkt 1000 cell 8 mpu 64
[root@node2 ~]# tc cl sh dev ens11f0
class mq :1 root leaf 8003: 
class mq :2 root 
class mq :3 root 
class mq :4 root 
class mq :5 root 
class mq :6 root 
class mq :7 root 
class mq :8 root 
class cbq 8003: root rate 10000Kbit (bounded,isolated) prio no-transmit
[root@node2 ~]# 
[root@node2 ~]# 
[root@node2 ~]# tc cl sh dev ens11f0
class mq :1 root leaf 8003: 
class mq :2 root 
class mq :3 root 
class mq :4 root 
class mq :5 root 
class mq :6 root 
class mq :7 root 
class mq :8 root 
class cbq 8003: root rate 10000Kbit (bounded,isolated) prio no-transmit
```



