---
layout: post
title: Websocket介绍与C语言实现服务端
tags:
- Protocol
- C
categories: Network
---
暑假学校布置了C语言课程设计的作业，其中需要一个用户图形界面。我就想能不能用点新奇的，于是想到了使用HTML界面，可以用JS做出很多生动的效果。那么，我们要怎么实现网页与C后端的通信呢，就要用到本文所讲解的websocket。<br/>Websocket是HTML5的一种新的协议，它实现了浏览器与服务器的全双工通信。那么websocket和以前的HTTP相比有什么好处呢，我们如何从底层去实现它呢？本文旨在讲解websocket协议的C语言实现和应用。

## 什么是websocket

以下内容摘自wiki：

>WebSocket一种在单个 TCP 连接上进行全双工通讯的协议。WebSocket通信协议于2011年被IETF定为标准RFC 6455，并被RFC7936所补充规范。WebSocket API也被W3C定为标准。<br/>WebSocket 使得客户端和服务器之间的数据交换变得更加简单，允许服务端主动向客户端推送数据。在 WebSocket API 中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。<br/>现在，很多网站为了实现推送技术，所用的技术都是轮询。轮询是在特定的的时间间隔（如每1秒），由浏览器对服务器发出HTTP请求，然后由服务器返回最新的数据给客户端的浏览器。这种传统的模式带来很明显的缺点，即浏览器需要不断的向服务器发出请求，然而HTTP请求可能包含较长的头部，其中真正有效的数据可能只是很小的一部分，显然这样会浪费很多的带宽等资源。<br/>而比较新的技术去做轮询的效果是Comet。这种技术虽然可以双向通信，但依然需要反复发出请求。而且在Comet中，普遍采用的长链接，也会消耗服务器资源。<br/>在这种情况下，HTML5定义了WebSocket协议，能更好的节省服务器资源和带宽，并且能够更实时地进行通讯。

从这些内容中，我们可以总结，websocket是一种新的，可以节省资源开销，具有更强的实时性的用于浏览器与服务器之间的全双工通信协议。

## Websocket通信建立

Websocket是建立在TCP之上，兼容HTTP协议的，同样不加密和加密也使用默认端口80和443。

Websocket和HTTP协议建立通信的对比流程图如下：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Websocket-01.png)

我们可以看到，HTTP协议每次均需要客户端(浏览器)对服务器发送一个request，然后接收服务器发送过来的消息，服务器无法做到主动向客户端推送信息。而websocket则是先通过一次握手建立连接，之后客户端和服务器之间便可以进行互相通信。因此，这样的协议就非常适合网页与后端需要频繁进行通信交流的情况。

接下来我们来一步步逐个实现整个通信流程。

## Websocket客户端

首先上一段客户端的代码：

```JS

// language：JavaScript

// creat websocket
var ws = new WebSocket('ws://localhost:8080');

// event when the websocket is opened
ws.onopen = function (evt) {
  console.log('Connected to server.');
}

// event when client receive a message
ws.onmessage = function (evt) {
  mainReceive(evt.data);
}

// event when error happens
ws.onerror = function (evt) {
  alert('There is something wrong with your network, please check your server.')
  console.log('Error occured.')
}

// event when the websocket is closed
ws.onclose = function (evt) {
  alert('Connection closed.');
  console.log('Connection closed.');
}

```

### Websocket创建

首先第一步当然是创建一个websocket对象。

```
var ws = new WebSocket('ws://localhost:8080');
```

要使用加密的话就是 `wss://localhost:8080`

### Websocket的四个基本事件

* ws.onopen：websocket打开
* ws.onmessage：接收到消息
* ws.onerror：发生错误
* ws.onclose：websocket关闭

以上四个属性均需要指定一个函数，该函数在发生对应的事件后便会执行。如onopen在打开websocket时，可以指定一些初始化的东西，onmessage可以在接收到消息后对信息加以处理，onerror可以在发生错误时给出一些提示信息或者指定一些错误处理机制，onclose可以在websocket关闭之后给出一些相应处理等等。

### Websocket的发送

这里主要介绍两个常用的方法和属性：

ws.send(buffer)：用以将buffer中的内容发送给服务器。（当然在发送时会加上一些websocket的头部）

ws.bufferedAmount：表示还有多少二进制字节的数据还没有发送出去。这个属性一般用于判断数据是否发送完，以 ws.bufferedAmount == 0 来表示数据发送完毕。

客户端这边是非常简单的，不用考虑很多细节的内容。

## Websocket服务端

同样我们首先先上一段代码，再来逐个讲解。

```c
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>
#include <fcntl.h>
#include <sys/shm.h>
#include <signal.h>
#include <arpa/inet.h>
#include <netinet/in.h>
#include <errno.h>

int main(void) {

  // create socket
  int sock_server;
  sock_server = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
  if (sock_server < 0) {
    printf("Creat socket failed!\n");
    return 1;
  }

  // create server sockaddr struct
  struct sockaddr_in servaddr;
  bzero(&servaddr, sizeof(servaddr));
  servaddr.sin_family = AF_INET;
  servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
  servaddr.sin_port = htons(PORT);

  // bind
  if (bind(sock_server, (struct sockaddr*)&servaddr, sizeo(servaddr)) == -1) {
    printf("Bind error!\n");
    return 1;
  }

  // listen
  if (listen(sock_server, QUEUE) == -1) {
    printf("Listen error!\n");
    return 1;
  }

  // no error
  printf("\n");
  printf("Server working......\n\n");

  // create client socket sockaddr
  int count = 0;
  char buff[BUFF_SIZE];
  struct sockaddr_in clieaddr;
  u_int len = sizeof(clieaddr);

  // build connection
  int connect = accept(sock_server, (struct sockaddr*)&clieaddr, &len);
  shakehands(connect);

  // main
  while (1) {

    // payload
    websocket_head head;
    char payload[BUFF_SIZE];
    bzero(payload, BUFF_SIZE);

    // receive message
    int Ecode = receive_and_parse(connect, &head);
    if (Ecode < 0)
      continue;

    // get message
    int readSize = read(connect, payload, 1024);
    if (readSize <= 0)
      continue;

    // decode message
    DecodeMessage(payload, readSize, head.masking_key);
    printf("Receive message from client:\n\n");
    printf("%s\n", payload);

    // main control
    Backstage_Main(payload, sizeof(payload));

    // send message to client
    SendMessage(connect, payload, sizeof(payload));

    }

    // close
    close(connect);
    close(sock_server);
    return 0;
}
```

### Websocket握手

在主函数中我们可以看到在connect之后有一个shakehands函数，作用就是实现客户端与服务器的握手，以建立之间的通信。

握手操作建立在HTTP协议上，首先需要客户端向服务器发送一个HTTP报文，我们用写好的测试程序得到的报文如下：

```http
GET / HTTP/1.1
Host: localhost:8080
Connection: Upgrade
Pragma: no-cache
Cache-Control: no-cache
Upgrade: websocket
Origin: file://
Sec-WebSocket-Version: 13
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.79 Safari/537.36
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.8,en-US;q=0.6,en;q=0.4,ja;q=0.2
Sec-WebSocket-Key: 80gv+MbuBdkfeMa5iUEXcA==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```

可以看到，Upgrade标明了这是一个websocket握手请求，而Sec-WebSocket-Key是客户端发来的秘钥，用以进行校验。而我们服务端，就需要对这样秘钥进行处理，并向客户端发送一个应答，当客户端检验无误之后，一个websocket通信就建立完成了。

当我们构造一个如下的数据包，发送给客户端，握手就完成了。

开头的两句 `101 Switching Protocols` 和 `Upgrade: websocket`就表明将协议切换到了websocket。

```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: iTYucvheSlIi+ZMY6hfOm1w2F0w=
```

接下来，我们看如何构造这个数据包。

首先给出一个魔幻的秘钥串：

```
#define KEY "258EAFA5-E914-47DA-95CA-C5AB0DC85B11"
```

这个字符串是websocket标准规定好了的，不能更改。

我们需要做的是：

1. 将websocket请求报文中的 Sec-WebSocket-Accept 字段的秘钥后面添加 258EAFA5-E914-47DA-95CA-C5AB0DC85B11。
2. 将整个字段进行SHA1哈希。
3. 将整个字段进行base64编码。

SHA1和base64编码我们可以使用openssl库来实现，其中base64实现的代码如下：

```c
#include <openssl/sha.h>
#include <openssl/pem.h>
#include <openssl/bio.h>
#include <openssl/evp.h>

// Base64 encode function, used to build connection
int base64_encode(char *in_str, int in_len, char *out_str) {

  BIO *b64, *bio;
  BUF_MEM *bptr = NULL;
  size_t size = 0;

  if (in_str == NULL || out_str == NULL)
    return -1;

  b64 = BIO_new(BIO_f_base64());
  bio = BIO_new(BIO_s_mem());
  bio = BIO_push(b64, bio);

  BIO_write(bio, in_str, in_len);
  BIO_flush(bio);

  BIO_get_mem_ptr(bio, &bptr);
  memcpy(out_str, bptr->data, bptr->length);
  out_str[bptr->length-1] = '\0';
  size = bptr->length;

  BIO_free_all(bio);
  return size;
}
```

整个握手函数代码如下：

```c
// read a line string form buffer
int readline(char *buff, int pos, char *line) {

  int len = strlen(buff);
  for (; pos < len; pos++) {
    if (buff[pos] =='\r' && buff[pos + 1] == '\n')
      return pos + 2;
    else
      *(line++) = buff[pos];
  }
	return -1;
}

// shakehands with client and build connection
int shakehands(int sock_client) {

  // definations
  int pos = 0;
  char request[BUFF_SIZE];
  char line[LINE_SIZE];
  //Sec-WebSocket-Accept
  char sec_accept[32];
  //sha1 data
  char sha1_data[SHA_DIGEST_LENGTH + 1];
  //reponse head buffer
  char response[BUFF_SIZE];

  // receive request
  if (read(sock_client, request, sizeof(request)) <= 0)
    return -1;
  printf("Receive HTTP request from cilent :\n\n");
  printf("%s\n", request);

  // prase request
  while ((request[pos] != '\r' || request[pos + 1] != '\n') && pos != -1) {

    // traversal lines
    bzero(&line, sizeof(line));
    pos = readline(request, pos, line);

    if (strstr(line, "Sec-WebSocket-Key") != NULL) {

      strcat(line, KEY);
      SHA1((unsigned char*)&line + 19, strlen(line + 19), (unsigned char*)&sha1_data);
      base64_encode(sha1_data, strlen(sha1_data), sec_accept);

      // build HTTP response
      sprintf(response, \
        "HTTP/1.1 101 Switching Protocols\r\n" \
        "Upgrade: websocket\r\n" \
        "Connection: Upgrade\r\n" \
        "Sec-WebSocket-Accept: %s\r\n" \
        "\r\n", sec_accept);
      printf("Send HTTP response to client :\n\n");
      printf("%s\n", response);

      // send response
      write(sock_client, response, strlen(response));
      break;
    }

  }
  return 0;
}
```

### Websocket数据包解析

握手之后，连接已经建立，接下来需要做的就是从客户端发来的数据包中解析出数据。

首先介绍websocket协议的数据帧：

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-------+-+-------------+-------------------------------+
 |F|R|R|R| opcode|M| Payload len |    Extended payload length    |
 |I|S|S|S|  (4)  |A|     (7)     |             (16/64)           |
 |N|V|V|V|       |S|             |   (if payload len==126/127)   |
 | |1|2|3|       |K|             |                               |
 +-+-+-+-+-------+-+-------------+ - - - - - - - - - - - - - - - +
 |     Extended payload length continued, if payload len == 127  |
 + - - - - - - - - - - - - - - - +-------------------------------+
 |                               |Masking-key, if MASK set to 1  |
 +-------------------------------+-------------------------------+
 | Masking-key (continued)       |          Payload Data         |
 +-------------------------------- - - - - - - - - - - - - - - - +
 :                     Payload Data continued ...                :
 + - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - +
 |                     Payload Data continued ...                |
 +---------------------------------------------------------------+
 ```

* FIN：
 
    * 占1bit，标识是否为此消息的最后一个数据包
    * 0x0 : 还有后续帧
    * 0x1 : 最后一帧

* RSV1, RSV2, RSV3：

    * 用于扩展协议，各占1bit，一般来说是0
    * 如果没有定义非零值，并且收到了非零的RSV，则websocket链接会失败

* opcode：

    * 数据包类型，占4bits
    * 0x0：代表连续的帧
    * 0x1：text帧
    * 0x2：binary帧
    * 0x3-7：为非控制帧而预留的
    * 0x8：关闭握手帧
    * 0x9：ping帧
    * 0xA：pong帧
    * 0xB-F：为非控制帧而预留的

* MASK：

    * 占1bit，用于标识Payload Data是否经过掩码处理.
    * 0x1：Masking-key域的数据即是掩码密钥，用于解码PayloadData。(非常重要)

* Extended payload length：

    * Payload Data的长度，占7bits，7+16bits，7+64bits
    * 如果其值在0-125，则是payload的真实长度。
    * 如果值是126，则后面2个字节形成的16bits无符号整型数的值是Payload Data的真实长度。(注意，网络字节序，需要转换)
    * 如果值是Payload Data，则后面8个字节形成的64bits无符号整型数的值是payload的真实长度。(注意，网络字节序，需要转换)

* Masking-key：

    * 占4bits，是数据进行加密的掩码，我们需要这个掩码来解码获得实际数据。(非常重要)

* Payload Data：

    * 应用层的数据，有可能被加密。

接下来我们就需要一个函数，做这样的事情：

1. 读取客户端发来的数据包
2. 寻找mask字段，看是否存在掩码
3. 寻找payload_length字段，根据数值的区间，来确定真正的长度
4. 寻找masking_key字段，并记下备用。

这里需要注意网络字节序的问题，所以我在这里还给出了一个strreverse函数来将字符串取反。

有了masking_key之后，我们就可以拿去解码后面的Payload Data，从而获得客户端发来的数据。

整个过程用到的结构体和代码的实现如下：

```c

typedef struct {
  u_char fin;
  u_char opencode;
  u_char mask;
  u_long payload_length;
  u_char masking_key[4];
} websocket_head;

// receive message from client and parse it
int receive_and_parse(int fd, websocket_head *pWS) {

  char tmp;

  // get fin and opencode
  if (read(fd, &tmp, 1) <= 0)
    return -1;
  pWS->fin = ((tmp & 0x80) == 0x80);
  pWS->opencode = (tmp & 0x0F);

  // get mask and plyload length
  if (read(fd, &tmp, 1) <= 0)
    return -1;
  pWS->mask = ((tmp & 0x80) == 0X80);
  pWS->payload_length = (tmp & 0x7F);

  // the next two bytes is true payload length
  if (pWS->payload_length == 126) {
    char extern_len[2];
    if (read(fd, extern_len, 2) <= 0)
      return -1;
    pWS->payload_length = (extern_len[0] & 0xff) << 8 | (extern_len[1] & 0xff);
  }

  // the next eight bytes is the true payload length
  else if (pWS->payload_length == 127) {
    char extern_len[8];
    if (read(fd, &extern_len, 8) <= 0)
      return -1;
    strreverse(extern_len, 8);
    memcpy(&(pWS->payload_length), extern_len, 8);
  }

  // get masking key
  if (read(fd, pWS->masking_key, 4) <= 0)
    return -1;
    
  return 0;
}

// decode message
void DecodeMessage(char *data, int len, u_char *mask) {
  int i;
  for (i = 0; i < len; i++)
    *(data + i) ^=*(mask + (i % 4));
}

```

### Websocket数据包发送

得到客户端发来的信息后，我们就可以加以处理，分析，等等。接下来需要考虑的就是向客户端发送数据包。

同样的，我们也需要构造一个数据帧，服务端向客户端发送的数据帧和客户端向服务端发送的数据帧还略有不同。

服务端向客户端发送的数据帧只有三个部分

1. 0x81
2. 内容的长度，格式和客户端向服务端发送的数据帧中的 Extended payload length 字段一致
3. 需要发送的内容

当客户端接收到数据包之后，经浏览器处理后自动就是第三部分“需要发送的内容”了。

具体实现的代码如下，关键知道了数据长度的处理方式后写起来就很容易。

```c

// send message to client
int SendMessage(int fd, char *payload, int payload_length) {

  char *buffer;
  int head_length;

  if (payload_length < 126) {
    buffer = (char *)malloc(2 + payload_length);
    buffer[0] = 0x81;
    buffer[1] = payload_length;
    head_length = 2;
  }

  else if (payload_length < 0xffff) {
    buffer = (char *)malloc(4 + payload_length);
    buffer[0] = 0x81;
    buffer[1] = 126;
    buffer[2] = (payload_length >> 8 & 0xff);
    buffer[3] = (payload_length & 0xff);
    head_length = 4;
  }

  else {
    buffer = (char *)malloc(12 + payload_length);
    buffer[0] = 0x81;
    buffer[1] = 127;
    memcpy(buffer + 2, &payload_length, sizeof(unsigned long long));
    strreverse(buffer + 2, sizeof(unsigned long long));
    head_length = 12;
  }

  // send packet
  memcpy(buffer + head_length, payload, payload_length);
  if (write(fd, buffer, head_length + payload_length) <= 0)
    return -1;

  free(buffer);
  return 0;
}

```

## 应用展示

最后我将它应用在了C语言课程设计中，为一个物流配送管理系统HTML前端和C后台之间建立通信。

效果如下：

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Websocket-02.png)

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Websocket-03.png)
