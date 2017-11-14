



# TC的安装

TC是Linux自带的模块，一般情况下不需要另行安装，可以用 man tc 查看tc 相关命令细节，tc 要求内核 2.4.18 以上

```
TC的安装
TC是Linux自带的模块，一般情况下不需要另行安装，可以用 man tc 查看tc 相关命令细节，tc 要求内核 2.4.18 以上
```

# TC原理介绍

​      

Linux中的QoS分为入口(Ingress)部分和出口(Egress)部分，入口部分主要用于进行入口流量限速(policing)，出口部分主要用于队列调度(queuingscheduling)。大多数排队规则(qdisc)都是用于输出方向的，输入方向只有一个排队规则，即ingressqdisc。ingressqdisc本身的功能很有限，但可用于重定向incomingpackets。通过Ingressqdisc把输入方向的数据包重定向到虚拟设备ifb，而ifb的输出方向可以配置多种qdisc，就可以达到对输入方向的流量做队列调度的目的。

 Ingress 限速只能对整个网卡入流量限速，无队列之分：

Ingress 流量的限速

```shell
tc qdisc add dev eth0 ingress

tc filter add dev eth0 parent ffff: protocol ip prio 10 u32 match ipsrc 0.0.0.0/0 police rate 2048kbps burst 1m drop flowid :1

```



Egress 限速：

Linux 操作系统中的流量控制器 TC(Traffic Control) 用于Linux内核的流量控制，它利用队列规定建立处理数据包的队列，并定义队列中的数据包被发送的方式，从而实现对流量的控制。TC 模块实现流量控制功能使用的队列规定分为两类，一类是无类队列规定，另一类是分类队列规定。无类队列规定相对简单，而分类队列规定则引出了分类和过滤器等概念，使其流量控制功能增强。

**        无类队列**规定是对进入网络设备（网卡）的数据流不加区分统一对待的队列规定。使用无类队列规定形成的队列能够接收数据包以及重新编排、延迟或丢弃数据包。这类队列规定形成的队列可以对整个网络设备（网卡）的流量进行整形，但不能细分各种情况。常用的无类队列规定主要有 pfifo_fast（先进先出）、TBF（令牌桶过滤器）、SFQ（随机公平队列）、ID（前向随机丢包）等等。这类队列规定使用的流量整形手段主要是排序、限速和丢包。

**        分类队列**规定是对进入网络设备的数据包根据不同的需求以分类的方式区分对待的队列规定。数据包进入一个分类的队列后，它就需要被送到某一个类中，也就是说需要对数据包做分类处理。对数据包进行分类的工具是过滤器，过滤器会返回一个决定，队列规定就根据这个决定把数据包送入相应的类进行排队。每个子类都可以再次使用它们的过滤器进行进一步的分类。直到不需要进一步分类时，数据包才进入该类包含的队列排队。除了能够包含其他队列规定之外，绝大多数分类的队列规定还能够对流量进行整形。这对于需要同时进行调度（如使用SFQ）和流量控制的场合非常有用。

Linux流量控制的基本原理如下图所示。



![](.\Images\20150511093020861.png)

接收包从输入接口（Input Interface）进来后，经过流量限制（Ingress Policing）丢弃不符合规定的数据包，由输入多路分配器（Input 
De-Multiplexing）进行判断选择。如果接收包的目的地是本主机，那么将该包送给上层处理，否则需要进行转发，将接收包交到转发块（ForwardingBlock）处理。转发块同时也接收本主机上层（TCP、UDP等）产生的包。转发块通过查看路由表，决定所处理包的下一跳。然后，对包进行排列以便将它们传送到输出接口（Output Interface）。**一般我们只能限制网卡发送的数据包，不能限制网卡接收的数据包**，所以我们可以通过改变发送次序来控制传输速率。Linux流量控制主要是在**输出接口排列**时进行处理和实现的。



# TC规则

##         1.流量控制方式

​                   流量控制包括一下几种方式：

### SHAPING（限制）

​                         当流量被限制时，它的传输速率就被控制在某个值以下。限制值可以大大小于有效带宽，这样可以平滑突发数据流量，使网络更为稳定。SHAPING（限制）只适用于向外的流量。

### SCHEDULING（调度）

​                        通过调度数据包的传输，可以在带宽范围内，按照优先级分配带宽。SCHEDULING（调度）也只适用于向外的流量。

### POLICING（策略）

​                        SHAPING（限制）用于处理向外的流量，而POLICING（策略）用于处理接收到的数据。

### DROPPING（丢弃）

​                        如果流量超过某个设定的带宽，就丢弃数据包，不管是向内还是向外。

##         2.流量控制处理对象

​                流量的处理由三种对象控制，它们是**：qdisc（排队规则）、class（类别）和filter（过滤器**）。

​                qdisc（排队规则）是 queueing discipline的简写，它是理解流量控制（traffic control）的基础。**无论何时，内核如果需要通过某个网络接口发送数据包，它都需要按照为这个接口配置的qdisc（排队规则）把数据包加入队列。**然后，内核会尽可能多的从qdisc里面取出数据包，把它们交给网络适配器驱动模块。最简单的qdisc是pfifo他不对进入的数据包做任何的处理，数据包采用先进先出的方式通过队列。不过，它会保存网络接口一时无法处 理的数据包。

​               qddis（排队规则）分为 CLASSLESS QDISC和 CLASSFUL QDISC  类别如下：

### CLASSLESS QDISC （无类别QDISC）

####               1.无类别QDISC包括：

**              [ p | b ]fifo**,使用最简单的qdisc（排队规则），纯粹的先进先出。只有一个参数：limit ，用来设置队列的长度，pfifo是以数据包的个数为单位；bfifo是以字节数为单位。

**             **

**               pfifo_fast**，在编译内核时，如果打开了高级路由器（Advanced Router）编译选项，pfifo_fast 就是系统的标准qdisc(排队规则)。它的队列包括三个波段（band）。在每个波段里面，使用先进先出规则。而三个波段（band）的优先级也不相同，band 0 的优先级最高，band 2的最低。如果band 0里面有数据包，系统就不会处理band 1 里面的数据包，band 1 和 band 2 之间也是一样的。数据包是按照服务类型（Type Of Service，TOS ）被分配到三个波段（band）里面的。

**               red**，red是Random Early Detection（随机早期探测）的简写。如果使用这种qdsic，当带宽的占用接近与规定的带宽时，系统会随机的丢弃一些数据包。他非常适合高带宽的应用。

**              sfq**，sfq是Stochastic Fairness Queueing 的简写。它会按照会话（session --对应与每个TCP 连接或者UDP流）为流量进行排序，然后循环发送每个会话的数据包。

**             tbf**，tbf是 Token Bucket Filter 的简写，适用于把流速降低到某个值。

####             2.无类别qdisc的配置

​           如果没有可分类qdisc，不可分类qdisc 只能附属于设备的根。它们的用法如下:



```
tc qdisc add dev DEV root QDISC QDISC_PARAMETERS  
```

要删除一个不可分类qdisc，需要使用如下命令

```
tc qdisc del dev DEV root  
```

一个网络接口上如果没有设置qdisc，pfifo_fast就作为缺省的qdisc。

### CLASSFUL QDISC(分类 QDISC)

####            可分类QDISC包括：

​        **   CBQ**，CBQ是 Class Based Queueing（基于类别排队）的缩写。它实现了一个丰富的连接共享类别结构，既有限制（shaping）带宽的能力，也具有带宽优先级别管理的能力。带宽限制是通过计算连接的空闲时间完成的。空闲时间的计算标准是数据包离队事件的频率和下层连接（数据链路层）的带宽。

​        

​          ** HTB**，HTB是Hierarchy Token Bucket 的缩写。通过在实践基础上的改进，它实现一个丰富的连接共享类别体系。使用HTB可以很容易地保证每个类别的带宽，虽然它也允许特定的类可以突破带宽上限，占用别的类的带宽。HTB可以通过TBF（Token Bucket Filter）实现带宽限制，也能够划分类别的优先级。

​           **PRIO**，PRIO qdisc 不能限制带宽，因为属于不同类别的数据包是顺序离队的。使用PRIO qdisc 可以很容易对流量进行优先级管理，只有属于高优先级类别的数据包全部发送完毕，参会发送属于低优先级类别的数据包。为了方便管理，需要使用iptables 或者 ipchains 处理数据包的服务类型（Type Of Service，TOS）。

HTB型的一些注意：
    1. HTB型class具有优先级，prio。可以指定优先级，数字低的优先级高，优先级范围从 0~7，0最高。
      它的效果是：存在空闲带宽时，优先满足高优先级class的需求，使得其可以占用全部空闲带宽，上限为ceil所指定的值。若此时还有剩余空闲带宽，则优先级稍低的class可以借用之。依优先级高低，逐级分配。
    
    2. 相同优先级的class分配空闲带宽时，按照自身class所指定的rate（即保证带宽）之间的比例瓜分空闲带宽。
      例如：clsss A和B优先级相同，他们的rate比为3：5，则class A占用空闲的3/8，B占5/8。
    
    3. tc filter的prio使用提示，prio表示该filter的测试顺序，小的会优先进行测试，如果匹配到，则不会继续测试。故此filter 的prio跟class的prio并不一样，class的prio表明了此class的优先级：当有空闲带宽时prio 数值低的（优先级高）class会优先占用空闲。
     若不同优先级的filter都可以匹配到包，则优先级高的filter起作用。相同优先级的filter（必须为同种类型classifier）严格按照命令的输入顺序进行匹配，先执行的filter起作用。


#### **           操作原理**

**             **类（class）组成一个树，每个类都只有一个父类，而一个类可以有多个子类。某些qdisc （例如：CBQ和 HTB）允许在运行时动态添加类，而其它的qdisc（例如：PRIO）不允许动态建立类。允许动态添加类的qdisc可以有零个或者多个子类，由它们为数据包排队。此外，每个类都有一个叶子qdisc，默认情况下，这个也在qdisc有可分类，不过每个子类只能有一个叶子qdisc。 当一个数据包进入一个分类qdisc，它会被归入某个子类。我们可以使用一下三种方式为数据包归类，不过不是所有的qdisc都能够使用这三种方式。

​            如果过滤器附属于一个类，相关的指令就会对它们进行查询。过滤器能够匹配数据包头所有的域，也可以匹配由ipchains或者iptables做的标记。

​    

​           树的每个节点都可以有自己的过滤器，但是高层的过滤器也可以一直接用于其子类。如果数据包没有被成功归类，就会被排到这个类的叶子qdisc的队中。相关细节在各个qdisc的手册页中。

####           **命名规则**

​          所有的qdisc、类、和过滤器都有ID。ID可以手工设置，也可以由内核自动分配。ID由一个主序列号和一个从序列号组成，两个数字用一个冒号分开。

​          qdisc，一个qdisc会被分配一个主序列号，叫做句柄（handle），然后把从序列号作为类的命名空间。句柄才有像1:0 一样的表达方式。习惯上，需要为有子类的qdisc显式的分配一个句柄。

​          类（Class），在同一个qdisc里面的类共享这个qdisc的主序列号，但是每个类都有自己的从序列号，叫做类识别符（classid）。类识别符只与父qdisc有关，与父类无关。类的命名习惯和qdisc相同。

​          过滤器（Filter），过滤器的ID有三部分，只有在对过滤器进行散列组织才会用到。详情请参考tc-filtes手册页。

#### **         单位**

**            **tc命令所有的参数都可以使用浮点数，可能会涉及到以下计数单位。

​         

##### 带宽或者流速单位：

![](.\Images\20150511134819738.png)

##### 数据的数量单位

![](.\Images\20150511135207564.png)

##### 时间的计量单位：

![](.\Images\20150511135126939.png)



# TC命令

​        tc可以使用以下命令对qdisc、类和过滤器进行操作：

​        add， 在一个节点里加入一个qdisc、类、或者过滤器。添加时，需要传递一个祖先作为参数，传递参数时既可以使用ID也跨越式直接传递设备的根。如果要建立一个qdisc或者过滤器，可以使用句柄（handle）来命名。如果要建立一个类，可以使用类识别符（classid）来命名。

​        remove， 删除由某个句柄（handle）指定的qdisc，根qdisc（root）也可以删除。被删除qdisc上所有的子类以及附属于各个类的过滤器都会被自动删除。

 

​        change， 以替代的方式修改某些条目。除了句柄（handle）和祖先不能修改以外，change命令的语法和add命令相同。换句话说，change命令不能指定节点的位置。

​         replace， 对一个现有节点进行近于原子操作的删除/添加。如果节点不存在，这个命令就会建立节点。

​        link， 只适用于qdisc，替代一个现有的节点。

​        tc命令的格式：       

```shell
             tc  qdisc [ add | change | replace | link ] dev DEV [ parent qdisc-id | root ] [ handle qdisc-id ] qdisc [ qdisc specific parameters ]  
      
             tc class [ add | change | replace ] dev DEV parent qdisc-id  [  classid class-id ] qdisc [ qdisc specific parameters ]  
      
             tc filter [ add | change | replace ] dev DEV [ parent qdisc-id | root ] protocol protocol prio priority filtertype [ filtertype specific param‐eters ] flowid flow-id  
      
             tc [ FORMAT ] qdisc show [ dev DEV ]  
      
             tc [ FORMAT ] class show dev DEV  
       
             tc filter show dev DEV  
      
    ###### FORMAT := { -s[tatistics] | -d[etails] | -r[aw] | -p[retty] | i[ec] }  
```

# 具体操作

​         Linux流量控制主要分为建立队列、建立分类和建立过滤器三个方面。

##         **  基本实现步骤**。

​                   1) 针对网络物理设备（如以太网卡eth0）绑定一个队列qdisc；

​                   2) 在该队列上建立分类class；

​                   3) 为每一分类建立一个基于路由的过滤器filter；

​                   4) 最后与过滤器相配合，建立特定的路由表。

​           

##          ** 环境模拟实例**。

​                    流量控制器上的以太网卡（eth0）的IP地址为 192.168.1.66， 在其上建立一个CBQ队列。假设包的平均大小为1000字节，包间隔发送单元的大小            为8字节，可接收冲突的发送最长包的数目为20字节。

   

​                     加入有三种类型的流量需要控制：

​                         1）是发往主机1的，其IP地址为192.168.1.24。其流量带宽控制在8Mbit，优先级为 2；

​                          2）是发往主机2的，其IP地址为192.168.1.30。其流量带宽控制在1Mbit，优先级为1；

​                          3）是发往子网1的，其子网号为192.168.1.0。子网掩码为255.255.255.0。流量带宽控制在1Mbit，优先级为6。

 

##             建立队列

​              一般情况下，针对一个网卡只需建立一个队列。               

```shell
    ###将一个cbq队列绑定到网络物理设备eth0上，其编号为1:0；网络物理设备eth0的实际带宽为10Mbit，包的平均大小为1000字节；包间隔发送单元的大小为8字节，最小传输包大小为64字节。  
      
    tc qdisc add dev eth0 root handle 1: cbq bandwidth 10Mbit avpkt 1000 cell 8 mpu 64  
```



## 建立分类

​           一般情况下，针对一个队列需建立一个根分类，然后再在其上建立子分类。对于分类，按其分类的编号顺序起作用，编号小的优先；一但符合某个分类匹配规则，通过该分类发送数据包，则其后的分类不在起作用。

```shell
    ##创建根分类1:1；分配带宽为10Mbit，优先级别为8.  
      
    tc class add dev eth0 parent 1:0 classid 1:1 cbq bandidth 10Mbit rate 10Mbit maxburst 20 allot 1514 prio 8 avpkt 1000 cell 8 weight 1Mbit  
      
    ###该队列的最大可用带宽为10Mbit，实际分配的带宽为10Mbit，可接收冲突的发送最长包数目为20字节；最大传输单元加MAC头的大小为1514字节，优先级别为8，包的平均大小为1000字节，包间隔发送单元的大小为8字节，相当于实际带宽的加权速率为1Mbit。  
    
        ## 创建分类1:2，其父类为1:1，分配带宽为 8Mbit， 优先级别为2.  
      
    tc class add dev eth0 parent 1:1 cbq bandwidth 10Mbit rate 8Mbit maxburst 20 allot 1514 prio 2 avpkt 1000 cell 8 weight 800Kbit split 1:0 bounded  
      
    ##  该队列的最大可用带宽为10Mbit，实际分配的带宽为8Mbit，可接收冲突的发送最长包数目为20字节；最大传输单元加MAC头的大小为1514字节，优先级别为2，包的平均大小为1000字节，包间隔发送单元的大小为8字节，相当于实际带宽的加权速率为800Kbit，分类的分离点为1:0，且不可借用未使用带宽。  
    
        ## 创建分类 1:4，其父分类为1:1，分配带宽为1Mbit，优先级别为6。  
      
    tc class add dev eth0 parent 1:1 classid 1:4 cbq bandwidth 10 Mbit rate 1Mbit maxburst 20 allot 1514 prio 6 avpkt 1000 cell 8 weight 100Kbit split 1:0  
      
    ## 该队列的最大可用带宽为10Mbit，实际分配的带宽为1Mbit，可接收冲突的发送最长包数目为20字节；最大传输单元加MAC头的大小为1514字节，优先级别为6，包的平均大小为1000字节，包间隔发送单元的大小为8字节，相当于实际带宽的加权速率为100Kbit，分类的分离点为1:0  
    
```



## 创建过滤器

​           过滤器主要服务于分类。

​           一般只需针对根分类提供一个过滤器，然后为每个子分类提供路由映射。

```shell
## 1)应用路由分类器到cbq队列的根，父分类编号为1:0；过滤协议为ip，优先级别为100，过滤器为基于路由表。  
    tc filter add dev eth0 parent 1:0 protocol ip prio 100 route  
  
## 2)建立路由映射分类 1:2 , 1:3 , 1:4  
    tc filter add dev eth0 parent 1:0 protocol ip prio 100 route to 2 flowid 1:2  
    tc filter add dev eth0 parent 1:0 protocol ip prio 100 route to 3 flowid 1:3  
    tc filter add dev eth0 parent 1:0 protocol ip prio 100 route to 4 flowid 1:4 
```

## 建立路由

​         该路由是与前面所建立的路由映射一一对应。

```shell
 # 1)发往主机192.168.1.24的数据包通过分类2转发（分类2的速率8Mbit）  
         ip route add 192.168.1.24 dev eth0 via 192.168.1.66 realm 2   
     # 2)发往主机192.168.1.30的数据包通过分类3转发（分类3的速率1Mbit）  
         ip route add 192.168.1.30 dev eth0 via 192.168.1.66 realm 3  
     # 3)发往子网192.168.1.0/24 的数据包通过分类4转发（分类4的速率1Mbit）  
         ip route add 192.168.1.0/24 dev eth0 via 192.168.1.66 realm 4  
  
注：一般对于流泪控制器所直接连接的网段建议使用IP主机地址流量控制限制，不要使用子网流量控制限制。如一定需要对直连子网使用子网流量控制限制，则在建立该子网的路由映射前，需将原先由系统建立的路由删除，才可以完成相应步骤  
```



##  监视

​         主要包括对现有队列、分类、过滤器和路由状况进行监视。

​          1）显示队列的状况

```shell
## 简单显示指定设备（这里为eth0）的队列状况  
   tc qdisc ls dev eth0  
     qdisc cbq 1: rate 10Mbit (bounded,isolated) prio no-transmit  
  
## 详细显示指定设备（这里为eth0）的队列状况  
   tc -s qdisc ls dev eth0  
     qdisc cbq 1: rate 10Mbit (bounded,isolated) prio no-transmit   
         Sent 7646731 bytes 13232 pkts (dropped 0, overlimits 0)  
  borrowed 0 overactions 0 avgidle 31 undertime 0  
  
## 这里主要显示了通过该队列发送了13232个数据包，数据流量为7646731个字节，丢弃的包数目为0，超过速率限制的包数目为0。
```



2）显示分类的状况

```shell
     ## 简单显示指定设备（这里为eth0）的分类状况  
        tc class ls dev eth0  
        class cbq 1: root rate 10Mbit (bounded,isolated) prio no-transmit  
        class cbq 1:1 parent 1: rate 10Mbit prio no-transmit  
        class cbq 1:2 parent 1:1 rate 8Mbit prio (bounded) prio 2  
        class cbq 1:3 parent 1:1 rate 1Mbit prio 1  
        class cbq 1:4 parent 1:1 rate 1Mbit prio 6  
      
    ## 详细显示指定设备（这里为eth0）的分类状况  
       tc -s class ls dev eth0  
        class cbq 1: root rate 10000Kbit (bounded,isolated) prio no-transmit  
         Sent 17725304 bytes 32088 pkt (dropped 0, overlimits 0 requeues 0)   
         backlog 0b 0p requeues 0   
         borrowed 0 overactions 0 avgidle 31 undertime 0  
       class cbq 1:1 parent 1: rate 10000Kbit prio no-transmit  
        Sent 16627774 bytes 28884 pkts (dropped 0, overlimits 0 requeues 0)   
        backlog 0b 0p requeues 0   
        borrowed 16163 overactions 0 avgidle 587 undertime 0  
      class cbq 1:2 parent 1:1 rate 8000Kbit (bounded) prio 2  
       Sent 628829 bytes 3130 pkts (dropped 0, overlimits 0 requeues 0)   
       backlog 0b 0p requeues 0   
       borrowed 0 overactions 0 avgidle 4137 undertime 0  
      class cbq 1:3 parent 1:1 rate 1000Kbit prio 1  
       Sent 0 bytes 0 pkts (dropped 0, overlimits 0 requeues 0)   
       backlog 0b 0p requeues 0   
       borrowed 0 overactions 0 avgidle 3.19309e+06 undertime 0  
     class cbq 1:4 parent 1:1 rate 1000Kbit prio 6  
      Sent 552879 bytes 8076 pkts (dropped 0, overlimits 0 requeues 0)   
      backlog 0b 0p requeues 0   
      borrowed 3797 overactions 0 avgidle 159557 undertime 0  
      
    ##这里主要显示了通过不同分类发送的数据包，数据流量，丢弃的包数目，超过速率限制的包数目等等。其中根分类(class cbq 1:0)的状况与队列的状况类似  
      
    ##例如，分类class cbq 1:4 发送了8076个包，数据流量为 552879 个字节，丢弃的包数目为0，超过速率限制的包数目为0。  
```



3）显示过滤器的状况

```shell
    tc -s filter ls dev eth0  
    filter parent 1: protocol ip pref 100 route   
    filter parent 1: protocol ip pref 100 route fh 0xffff0002 flowid 1:2 to 2   
    filter parent 1: protocol ip pref 100 route fh 0xffff0003 flowid 1:3 to 3   
    filter parent 1: protocol ip pref 100 route fh 0xffff0004 flowid 1:4 to 4   
      
    ## 这里flowid 1:2 代表分类 class cbq 1:2, to 2 代表通过路由 2 发送  
```

4）显示现有路由的状况

```shell
    ip route  
    default via 192.168.1.1 dev eth0   
    192.168.1.0/24 dev eth0  proto kernel  scope link  src 192.168.1.66   
    192.168.1.24 via 192.168.1.66 dev eth0 realm 2   
    192.168.1.30 via 192.168.1.66 dev eth0 realm 3   
      
    #如上所示，结尾包含有realm 的显示行是起作用的路由过滤器。  
```

## 维护

​          主要包括对队列、分类、过滤器和路由的增添、修改和删除。

​          增添动作一般依照  队列 -> 分类 -> 过滤器 -> 路由   的顺序进行；修改动作则没有什么要求；删除则依照   路由 -> 过滤器 -> 分类 -> 队列  的顺序进行。   

​           1）队列的维护

​            一般对于一台流量控制器来说，出厂时针对每个以太网卡均已配置好一个队列了，通常情况下对队列无需进行增添、修改和删除动作。

​            2）分类的维护

​               增添，增添动作通过 tc class add 命令实现，如前面所示。

​              修改，修改动作通过 tc class change 命令实现，如下所示：

```shell
tc class change dev eth0 parent 1:1 classid 1:2 cbq bandwidth 10Mbit rate 7Mbit maxburst 20 allot 1514 prio 2 avpkt 1000 cell 8 weigth 700Kbit split 1:0 bounded  
```

 对于bounded 命令应慎用，一旦添加后就进行修改，只可通过删除后再添加来实现。

​              删除，删除动作只在该分类没有工作前才可以进行，一旦通过该分类发送过数据，则无法删除它。因此，需要通过shell文件方式来修改，通过重新启动来完成删除动作。

​           3）过滤器的维护

​                  增添，增添动作通过 tc filter add 命令实现，如前面所示。

​                  修改，修改动作通过 tc filter change 命令实现，如下所示：

```shell
tc filter change dev eth0 parent 1:0 protocol ip prio 100 route to 10 flowid 1:8  
```

​		删除，删除动作通过 tc filter del 命令实现，如下所示：

```shell
tc filter del dev eth0 parent 1:0 protocol ip prio 100 route to 10  
```

4)与过滤器————映射路由的维护

​                 增添，增添动作通过 ip route add 命令实现，如前面所示。

​                 修改，修改动作通过 ip route change 命令实现，如下所示：

```shell
   ip route change 192.168.1.30 dev eth0 via 192.168.1.66 realm 8  
```



​		删除，删除动作通过 ip route del 命令实现，如下所示：

```shell
    ip route del 192.168.1.30 dev eth0 via 192.168.1.66 realm 8  
      
    ip route del 192.168.1.0/24 dev eth0 via 192.168.1.66 realm 4  
```





PS:

[Open vSwitch之QoS的实现]: http://www.sdnlab.com/19208.html

