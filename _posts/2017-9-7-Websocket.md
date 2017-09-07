---
layout: post
title: Websocket介绍与C语言实现与网页通信
tags:
- Protocol
- C
categories: Network
---
暑假学校布置了C语言课程设计的作业，其中需要一个用户图形界面。我就想能不能用点新奇的，于是想到了使用HTML界面，可以用JS做出很多生动的效果。那么，我们要怎么实现网页与C后端的通信呢，就要用到本文所讲解的websocket。<br/>Websocket是HTML5的一种新的协议，它实现了浏览器与服务器的全双工通信。那么websocket和以前的HTTP相比有什么好处呢，我们如何从底层去实现它呢？本文旨在讲解websocket协议的C语言实现和应用。

## 什么是websocket

