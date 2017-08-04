---
layout: post
title: Internet校验和与C语言实现
tags:
- Net Attack
- C
categories: Network
---
Internet校验和，是一个存在于IP，TCP，UDP等数据包的头部的一个16位的字段，用以检测数据的完整性，本文将讲解网络数据包中的校验和计算算法，并附上C语言实现代码。

## 校验和介绍与实现

校验和，是指对一串数据进行每16位相加，最后再对所得的结果取反码得到，为一个存在于各种网络数据包头部的一个16位的字段。

然而，数据的长度也有可能不是奇数个字节，对于这种情况，我们在最后一个字节的高位补上0。

此外，如果最后计算的和大于十六位，我们就必须对计算出的结果再进行求校验和，直至最终的结果是十六位。

根据以上说明，我们就可以写出基本的校验和计算代码。

```c
/* calculate checksum */
// buf：强制转换为u_short格式的数据
// len：要计算校验和的长度
u_short checksum(u_short *buf, int len) {

    u_long cksum = 0;

    while(len > 1) {
        cksum += *buf++;
        len -= sizeof(u_short);
    }

    if (len)
        cksum += *(u_char *)buf;

    while (cksum >> 16)
        cksum = (cksum >> 16) + (cksum & 0xffff);
    return (u_short)(~cksum);
}
```

代码是非常简单的，接下来我们讨论网络数据包中的校验和字段。

```c
/* TCP packet head */
typedef struct {
    u_short sour_port;
    u_short dest_port;
    u_int sequ_num;
    u_int ackn_num;
    u_short header_len_flag;
    u_short window;
    u_short check_sum;
    u_short surg_point;
} tcp_header;

/* UDP packet head */
typedef struct {
    u_short sour_port;
    u_short dest_port;
    u_short length;
    u_short check_sum;
} udp_header;
```

以上三段代码中的check_sum字段就是校验和了，接下来我们利用以上三个结构解析数据包并计算校验和。

## IP数据包

IP数据包头部的结构如下：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/MITM-Attack-02.png)

代码如下：

```c
/* ip packet head */
typedef struct {
    u_char verson_head;
    u_char type_of_service;
    u_short packet_len;
    u_short packet_id;
    u_short slice_info;
    u_char TTL;
    u_char protocol_type;
    u_short check_sum;
    u_char src_ip[4];
    u_char dest_ip[4];
} ip_header;
```

其中check_sum字段便是我们要计算的校验和，不过要注意的是，IP数据包的校验和只校验头部，也就是说，只计算二十个字节。

## TCP和UDP

TCP数据包头部的结构如下：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/MITM-Attack-03.png)

代码如下：

```c
/* TCP packet head */
typedef struct {
    u_short sour_port;
    u_short dest_port;
    u_int sequ_num;
    u_int ackn_num;
    u_short header_len_flag;
    u_short window;
    u_short check_sum;
    u_short surg_point;
} tcp_header;
```

UDP数据包头部的结构如下：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Internet-Checksum-01.jpg)

代码如下：

```c
/* UDP packet head */
typedef struct {
    u_short sour_port;
    u_short dest_port;
    u_short length;
    u_short check_sum;
} udp_header;
```

同样check_sum字段是我们要计算的校验和。

不过TCP和UDP要复杂一点，在计算校验和的时候，涉及到一个“伪首部”的概念。

伪首部：一个十二位的字段，仅用于校验和的计算，并不在数据包中实际存在，其结构如下：【TCP和UDP的完全相同】

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Internet-Checksum-02.png)

代码如下：

```c
/* TCP/UDP pseudo head */
typedef struct {
    u_long src_ip;
    u_long dst_ip;
    u_char place_holder;
    u_char protocal_type;
    u_short len;
} pseudo_header;
```

* src_ip：源IP地址
* dst_ip：目的IP地址
* place_holder：全为0的一个填充字节
* protocol_type：协议类型
* len：数据包长度

在实际的校验和计算时，在数据包前面加上伪头部进行计算。

## 整合

整合好后的代码如下，该函数的功能是，自动计算网络数据包的校验和，并赋值在相关字段处。

两个参数分别为：指向数据包的u_char *型指针， 数据包的长度。


```c
int PacketCheckSum(u_char *packet, int len) {

    /* reject packet except IPv4 */
    ethernet_header *pEther = (ethernet_header *)packet;
    if (ntohs(pEther->eth_type) != EPT_IPv4)
        return 1;

    ip_header *pIpv4 = (ip_header *)(packet + 14);

    /* TCP checksum */
    if (pIpv4->protocol_type == PROTOCOL_TCP) {
        
        tcp_header *pTcp = (tcp_header *)(packet + 34);
        pseudo_header Pse;
        Pse.src_ip = (u_long)pIpv4->src_ip;
        Pse.dst_ip = (u_long)pIpv4->dest_ip;
        Pse.place_holder = 0;
        Pse.protocal_type = PROTOCOL_TCP;
        Pse.len = htons(len - 34);

        u_char *data = (u_char *)malloc(len - 34 + 12);
        if (data == NULL)
            return 1;
        memset(data, 0, len - 34 + 12);
        memcpy(data, &Pse, 12);
        memcpy(data + 12, packet + 34, len - 34);
        pTcp->check_sum = checksum((u_short *)data, len - 34 + 12);
    }

    /* UDP checksum */
    else if (pIpv4->protocol_type == PROTOCOL_UDP) {

        udp_header *pUdp = (udp_header *)(packet + 34);
        pseudo_header Pse;
        Pse.src_ip = (u_long)pIpv4->src_ip;
        Pse.dst_ip = (u_long)pIpv4->dest_ip;
        Pse.place_holder = 0;
        Pse.protocal_type = PROTOCOL_UDP;
        Pse.len = htons(len - 34);

        u_char *data = (u_char *)malloc(len - 34 + 12);
        if (data == NULL)
            return 1;
        memset(data, 0, len - 34 + 12);
        memcpy(data, &Pse, 12);
        memcpy(data + 12, packet + 34, len - 34);
        pUdp->check_sum = checksum((u_short *)data, len -34 + 12);
    }

    /* IP checksum */
    pIpv4->check_sum = 0;
    pIpv4->check_sum = checksum((u_short *)pIpv4, 20);

    return 0;
}
```