###### 首先分析正常收包流程

正常收包流程如下：

进入函数netif_receive_skb()后，skb正式开始协议栈之旅。

![img](http://blog.chinaunix.net/attachment/201108/3/24148050_1312338439sZa0.jpg)



跟OSI七层模型不同，linux根据包结构对网络进行分层。
比如，arp头和ip头都是紧跟在以太网头后面的，所以在linux协议栈中arp和ip地位相同(如上图)
但是在OSI七层模型中，arp属于链路层，ip属于网络层..... 
我们就说arp，ip都属于第二层。下面是网络第二层的处理流程：

**一、相关数据结构**
内核处理网络第二层，有下面2个重要list_head变量 （文件linux_2_6_24/net/core/dev.c)
list_head 链表上挂了很多packet_type数据结构

```


static struct list_head ptype_base[16] __read_mostly;   /* 16 way hashed list */

static struct list_head ptype_all __read_mostly;        /* Taps */

struct packet_type {

    __be16 type;                /* This is really htons(ether_type).*/

    struct net_device   dev;   / NULL is wildcarded here       */

    int     (*func) (struct sk_buff *,

                     struct net_device *,

                     struct packet_type *,

                     struct net_device *);

    struct sk_buff    (gso_segment)(struct sk_buff *skb, int features);

    int    (*gso_send_check)(struct sk_buff *skb);

    void   *af_packet_priv;

    struct list_head    list;

};

```



**type** 成员保存了二层协议类型，ETH_P_IP、ETH_P_ARP等等
**func **成员就是钩子函数了，如 ip_rcv()、arp_rcv()等等



**二、操作packet_type的API**
//把packet_type结构挂在与type对应的list_head上面

```


void **dev_add_pack**(struct packet_type *pt){
 int hash;

    spin_lock_bh(&ptype_lock);

    if (pt->type == htons(ETH_PALL))        //type为ETH_PALL时，挂在ptype_all上面

        list_add_rcu(&pt->list, &ptype_all);

    else {

        hash = ntohs(pt->type) & 15;         //否则，挂在ptype_base[type&15]上面

        list_add_rcu(&pt->list, &ptype_base[hash]);

    }

    spin_unlock_bh(&ptype_lock);

}

//把packet_type从list_head上删除

void dev_remove_pack(struct packet_type *pt){

    __dev_remove_pack(pt);

    synchronize_net();

}

void __dev_remove_pack(struct packet_type *pt){

    struct list_head *head;

    struct packet_type *pt1;

    spin_lock_bh(&ptype_lock);

    if (pt->type == htons(ETH_P_ALL))

        head = &ptype_all;                        //找到链表头

    else

        head = &ptype_base[ntohs(pt->type) & 15]; //

    list_for_each_entry(pt1, head, list) {

        if (pt == pt1) {

            list_del_rcu(&pt->list);

            goto out;

        }

    }

    printk(KERN_WARNING "dev_remove_pack: %p not found.\n", pt);

out:

    spin_unlock_bh(&ptype_lock);

}

```

   

**三、进入二层协议处理函数**



```
int netif_receive_skb(struct sk_buff *skb)

{

   //略去一些代码

    rcu_read_lock();

    //第一步：先处理 ptype_all 上所有的 packet_type->func()            

    //所有包都会调func，对性能影响严重！内核默认没挂任何钩子函数

    list_for_each_entry_rcu(ptype, &ptype_all, list) {  //遍历ptye_all链表

        if (!ptype->dev || ptype->dev == skb->dev) {    //上面的paket_type.type 为 ETH_P_ALL

            if (pt_prev)                                //对所有包调用paket_type.func()

                ret = deliver_skb(skb, pt_prev, orig_dev); //此函数最终调用paket_type.func()

            pt_prev = ptype;

        }

    }

    //第二步：若编译内核时选上BRIDGE，下面会执行网桥模块

    //调用函数指针 br_handle_frame_hook(skb), 在动态模块 linux26_24/net/bridge/br.c中

    //br_handle_frame_hook = br_handle_frame;

    //所以实际函数 br_handle_frame。

    //注意：在此网桥模块里初始化 skb->pkt_type 为 PACKET_HOST、PACKET_OTHERHOST

    skb = handle_bridge(skb, &pt_prev, &ret, orig_dev);

    if (!skb) goto out;

    //第三步：编译内核时选上MAC_VLAN模块，下面才会执行

    //调用 macvlan_handle_frame_hook(skb), 在动态模块linux26_24/drivers/net/macvlan.c中

    //macvlan_handle_frame_hook = macvlan_handle_frame; 

    //所以实际函数为 macvlan_handle_frame。 

    //注意：此函数里会初始化 skb->pkt_type 为 PACKET_BROADCAST、PACKET_MULTICAST、PACKET_HOST

    skb = handle_macvlan(skb, &pt_prev, &ret, orig_dev);

    if (!skb)  goto out;

    //第四步：最后 type = skb->protocol; &ptype_base[ntohs(type)&15]

    //处理ptype_base[ntohs(type)&15]上的所有的 packet_type->func()

    //根据第二层不同协议来进入不同的钩子函数，重要的有：ip_rcv() arp_rcv()

    type = skb->protocol;

    list_for_each_entry_rcu(ptype, &ptype_base[ntohs(type)&15], list) {

        if (ptype->type == type &&                      //遍历包type所对应的链表

            (!ptype->dev || ptype->dev == skb->dev)) {  //调用链表上所有pakcet_type.func()

            if (pt_prev)

                ret = deliver_skb(skb, pt_prev, orig_dev); //就这里！arp包会调arp_rcv()

            pt_prev = ptype;                               //        ip包会调ip_rcv()

        }

    }

    if (pt_prev) {

        ret = pt_prev->func(skb, skb->dev, pt_prev, orig_dev);

    } else {               //下面就是数据包从协议栈返回来了

        kfree_skb(skb);    //注意这句，若skb没进入socket的接收队列，则在这里被释放

        ret = NET_RX_DROP; //若skb进入接收队列，则系统调用取包时skb释放，这里skb引用数减一而已

    }

out:

    rcu_read_unlock();

    return ret;

}

int deliver_skb(struct sk_buff *skb,struct packet_type *pt_prev, struct net_device *orig_dev){

    atomic_inc(&skb->users); //这句不容忽视，与后面流程的kfree_skb()相呼应

    return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);//调函数ip_rcv() arp_rcv()等

}


```





这里只是将大致流程，arp_rcv(), ip_rcv() 什么的具体流程，以后再写。



##### 其次我们分析tcpdump加入hook的流程：

1、首先用一个别的例子来帮助理解；

下面是一个使用RAW socket获取802.1x协议(0x888e)的例子，可以帮助理解！

```
int init_sockets()

{

    struct ifreq ifr;

    struct sockaddr_ll addr;

    struct sockaddr_in addr2;

 

    drv->sock = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_PAE));//802.1x协议(0x888e)

    if (drv->sock < 0) {

        perror("socket[PF_PACKET,SOCK_RAW]");

        return -1;

    }

 

    os_memset(&ifr, 0, sizeof(ifr));

    os_strlcpy(ifr.ifr_name, “eth0”, sizeof(ifr.ifr_name));//指定接口eth0

    if (ioctl(drv->sock, SIOCGIFINDEX, &ifr) != 0) {//获取eth0接口的index

        perror("ioctl(SIOCGIFINDEX)");

        return -1;

    }

 

    os_memset(&addr, 0, sizeof(addr));

    addr.sll_family = AF_PACKET;

    addr.sll_ifindex = ifr.ifr_ifindex;

    

    if (bind(drv->sock, (struct sockaddr *) &addr, sizeof(addr)) < 0) {//绑定sock

        perror("bind");

        return -1;

}

...

    return 0;

}

 

void read(int sock...)

{

    int len;

    unsigned char buf[3000];

len = recv(sock, buf, sizeof(buf), 0);

...

}

int send(int sock, char *buf, int bufsize)

{

return send(sock, buf, bufsize, 0);

...

}
```



2、tcpdump使用RAW socket从内核中抓取指定协议的数据包流程分析

tcpdump也是在二层抓包的，用的是libpcap库，它的基本原理是
1.先创建socket，内核dev_add_packet()挂上自己的钩子函数
2.然后在钩子函数中，把skb放到自己的接收队列中，
3.接着系统调用recv取出skb来，把数据包skb->data拷贝到用户空间
4.最后关闭socket，内核dev_remove_packet()删除自己的钩子函数

下面是一些重要的数据结构，用到的钩子函数都在这里初始化好了

```
static const struct proto_ops packet_ops = {
    .family =    PF_PACKET,
    .owner =    THIS_MODULE,
    .release =    packet_release,    //关闭socket的时候调这个
    .bind =        packet_bind,
    .connect =    sock_no_connect,
    .socketpair =    sock_no_socketpair,
    .accept =    sock_no_accept,
    .getname =    packet_getname,
    .poll =        packet_poll,
    .ioctl =    packet_ioctl,
    .listen =    sock_no_listen,
    .shutdown =    sock_no_shutdown,
    .setsockopt =    packet_setsockopt,
    .getsockopt =    packet_getsockopt,
    .sendmsg =    packet_sendmsg,
    .recvmsg =    packet_recvmsg,   //socket收包的时候调这个
    .mmap =        packet_mmap,
    .sendpage =    sock_no_sendpage,
};

static struct net_proto_family packet_family_ops = {
    .family =    PF_PACKET,
    .create =    packet_create,     //创建socket的时候调这个
    .owner    =    THIS_MODULE,
};

```

**1 系统调用socket**
libpcap系统调用socket，内核最终调用 packet_create

这里有个映射表可以参考：

| 用户空间        | 内核空间                    | 说明           |
| ----------- | ----------------------- | ------------ |
| socket()    | packet_create()         | 创建特定类型socket |
| ioctl()     | packet_ioctl()          |              |
| bind[()]()  | **packet_bind()**       | 绑定socket     |
| recv()      | packet_recvmsg()        |              |
| send()      | packet_sendmsg()        |              |
| setsocket() | **packet_setsockopt**() |              |
| getsocket() | **packet_getsockopt**() |              |



下面是内核态packet_create是如何将钩子函数载入二层的：

```
static int packet_create(struct net *net, struct socket *sock, int protocol){

    po->prot_hook.func = packet_rcv;   //初始化钩子函数指针

    po->prot_hook.af_packet_priv = sk;

    if (protocol) {

        po->prot_hook.type = protocol;  //类型是系统调用socket形参指定的

        dev_add_pack(&po->prot_hook);//这里将hook放入了二层收函数里

        sock_hold(sk);

        po->running = 1;

    }

    return(0);

}

```



**2 钩子函数 packet_rcv 将skb放入到接收队列**
文件 linux_2_6_24/net/packet/af_packet.c
简单来说，packet_rcv中，skb越过了整个协议栈，直接进入队列



**3 系统调用recv**

tcpdump在建立好socket后，可以使用系统调用recv、read、recvmsg，

内核最终会调用**packet_recvmsg**
从接收队列中取出skb，将数据包内容skb->data拷贝到用户空间，后续处理由tcpdump自己处理：



**4 系统调用close**
内核最终会调用packet_release

```
static int packet_release(struct socket *sock){

    struct sock *sk = sock->sk;

    struct packet_sock *po;

    if (!sk)  return 0;

    po = pkt_sk(sk);

    write_lock_bh(&packet_sklist_lock);

    sk_del_node_init(sk);

    write_unlock_bh(&packet_sklist_lock);

    // Unhook packet receive handler.

    if (po->running) {

        dev_remove_pack(&po->prot_hook);   //就是这句！！把packet_type从链表中删除

        po->running = 0;

        po->num = 0;

        __sock_put(sk);

    }

    packet_flush_mclist(sk);

     // Now the socket is dead. No more input will appear.

    sock_orphan(sk);

    sock->sk = NULL;

    /* Purge queues */

    skb_queue_purge(&sk->sk_receive_queue);

    sk_refcnt_debug_release(sk);

    sock_put(sk);

    return 0;

}



```



