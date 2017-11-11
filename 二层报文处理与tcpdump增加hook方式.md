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

static struct list_head **ptype_base**[16] __read_mostly;   /* 16 way hashed list */
static struct list_head **ptype_all** __read_mostly;        /* Taps */

struct packet_type {
​    __be16 **type**;                /* This is really htons(ether_type).*/
​    struct net_device   *dev;   /* NULL is wildcarded here       */
​    int     (***func**) (struct sk_buff *,
​                     struct net_device *,
​                     struct packet_type *,
​                     struct net_device *);
​    struct sk_buff    *(*gso_segment)(struct sk_buff *skb, int features);
​    int    (*gso_send_check)(struct sk_buff *skb);
​    void   *af_packet_priv;
​    struct list_head    list;
};

**type** 成员保存了二层协议类型，ETH_P_IP、ETH_P_ARP等等
**func **成员就是钩子函数了，如 ip_rcv()、arp_rcv()等等



**二、操作packet_type的API**
//把packet_type结构挂在与type对应的list_head上面
void **dev_add_pack**(struct packet_type *pt){
​    int hash;
​    spin_lock_bh(&ptype_lock);
​    if (pt->type == htons(ETH_P_ALL))        //type为ETH_P_ALL时，挂在ptype_all上面
​        list_add_rcu(&pt->list, &ptype_all);
​    else {
​        hash = ntohs(pt->type) & 15;         //否则，挂在ptype_base[type&15]上面
​        list_add_rcu(&pt->list, &ptype_base[hash]);
​    }
​    spin_unlock_bh(&ptype_lock);
}

//把packet_type从list_head上删除
void **dev_remove_pack**(struct packet_type *pt){
​    __dev_remove_pack(pt);
​    synchronize_net();
}
void __dev_remove_pack(struct packet_type *pt){
​    struct list_head *head;
​    struct packet_type *pt1;
​    spin_lock_bh(&ptype_lock);
​    if (pt->type == htons(ETH_P_ALL))
​        head = &ptype_all;                        //找到链表头
​    else
​        head = &ptype_base[ntohs(pt->type) & 15]; //

​    list_for_each_entry(pt1, head, list) {
​        if (pt == pt1) {
​            list_del_rcu(&pt->list);
​            goto out;
​        }
​    }
​    printk(KERN_WARNING "dev_remove_pack: %p not found.\n", pt);
out:
​    spin_unlock_bh(&ptype_lock);
}

**三、进入二层协议处理函数**
int netif_receive_skb(struct sk_buff *skb)
{
   //略去一些代码
​    rcu_read_lock();
​    //第一步：先处理 ptype_all 上所有的 packet_type->func()            
​    //所有包都会调func，对性能影响严重！内核默认没挂任何钩子函数
​    list_for_each_entry_rcu(ptype, **&ptype_all**, list) {  //遍历ptye_all链表
​        if (!ptype->dev || ptype->dev == skb->dev) {    //上面的paket_type.type 为 ETH_P_ALL
​            if (pt_prev)                                //对所有包调用paket_type.func()
​                ret = deliver_skb(skb, pt_prev, orig_dev); //此函数最终调用paket_type.func()
​            pt_prev = ptype;
​        }
​    }
​    //第二步：若编译内核时选上BRIDGE，下面会执行网桥模块
​    //调用函数指针 br_handle_frame_hook(skb), 在动态模块 linux_2_6_24/net/bridge/br.c中
​    //br_handle_frame_hook = br_handle_frame;
​    //所以实际函数 br_handle_frame。
​    //注意：在此网桥模块里初始化 skb->pkt_type 为 PACKET_HOST、PACKET_OTHERHOST
​    skb = **handle_bridge**(skb, &pt_prev, &ret, orig_dev);
​    if (!skb) goto out;

​    //第三步：编译内核时选上MAC_VLAN模块，下面才会执行
​    //调用 macvlan_handle_frame_hook(skb), 在动态模块linux_2_6_24/drivers/net/macvlan.c中
​    //macvlan_handle_frame_hook = macvlan_handle_frame; 
​    //所以实际函数为 macvlan_handle_frame。 
​    //注意：此函数里会初始化 skb->pkt_type 为 PACKET_BROADCAST、PACKET_MULTICAST、PACKET_HOST
​    skb = **handle_macvlan**(skb, &pt_prev, &ret, orig_dev);
​    if (!skb)  goto out;

​    //第四步：最后 type = skb->protocol; &ptype_base[ntohs(type)&15]
​    //处理ptype_base[ntohs(type)&15]上的所有的 packet_type->func()
​    //根据第二层不同协议来进入不同的钩子函数，重要的有：**ip_rcv() arp_rcv()**
​    type = skb->protocol;
​    list_for_each_entry_rcu(ptype, **&ptype_base[ntohs(type)&15]**, list) {
​        if (ptype->type == type &&                      //遍历包type所对应的链表
​            (!ptype->dev || ptype->dev == skb->dev)) {  //调用链表上所有pakcet_type.func()
​            if (pt_prev)
​                ret = deliver_skb(skb, pt_prev, orig_dev); //就这里！arp包会调arp_rcv()
​            pt_prev = ptype;                               //        ip包会调ip_rcv()
​        }
​    }
​    if (pt_prev) {
​        ret = pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
​    } else {               //下面就是数据包从协议栈返回来了
​        kfree_skb(skb);    //注意这句，若skb没进入socket的接收队列，则在这里被释放
​        ret = NET_RX_DROP; //若skb进入接收队列，则系统调用取包时skb释放，这里skb引用数减一而已
​    }
out:
​    rcu_read_unlock();
​    return ret;
}

int deliver_skb(struct sk_buff *skb,struct packet_type *pt_prev, struct net_device *orig_dev){
​    **atomic_inc(&skb->users);** **//这句不容忽视，与后面流程的kfree_skb()相呼应**
​    return **pt_prev->func(skb, skb->dev, pt_prev, orig_dev)**;//调函数ip_rcv() arp_rcv()等
}

这里只是将大致流程，arp_rcv(), ip_rcv() 什么的具体流程，以后再写。