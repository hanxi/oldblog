---
layout: post
title: "一年工作总结"
tagline: "one yes work summary"
description: "总结第一年工作情况"
tags: ["总结", "杂文"]
categories: ["杂文"]
---
{% include JB/setup %}

工作已经满一年了。。。记得还是去年十月多的时候来广州找工作的，找了大概一个星期左右，然后找到这家公司后就一直待到现在。今年毕业的时候回了趟学校，到现在工作时间大概有一年了。第一个手游项目已经上线有一段时间了，还不知道盈利情况怎么样，又开始了一个新的卡牌游戏。工作一直挺充实的，第一个项目的从零开始我就加入了项目组，从头至尾参与了项目的研发，感觉收获还是有点的，但一时又想不到怎么说。现在新的卡牌游戏用的是上一个项目的框架。既然不知道总结什么，我就把游戏服务端框架说一说吧。


<img src="/assets/media/gameserver.png" alt="Sanjose" class="img-center">

对着上面的框架图来说，首先是网关FGGateway，作为一个连接服务器，处理服务端和客户端的连接，对应的数据分配。我公司是用libevent实现的。代码量很少，方便移植。

逻辑服务器FGServer和FGClient所使用的网络使用select实现的，同一套代码，从网络的角度这逻辑服务器也是一个客户端。

FGClient连接FGServer的过程图上没说到FGGateway的中转过程，这里简单说下流程：

* FGServer
    1. FGServer 调用 connect 连接FGGateway，
    2. 连接成功，FGServer发送第一条协议给FGGateway，携带设备信息告诉FGGateway说自己是服务端。
    3. FGGateway 验证设备信息，成功则把FGServer加到在线服务器列表中。

* FGClient
    1. FGClient 调用 connect 连接FGGateway，
    2. 连接成功，FGClient发送第一条协议给FGGateway，携带设备信息告诉FGGateway是说自己是客户端。
    3. FGGateway 把FGClient保存到客户端队列，
    4. FGClient 发送第二条协议获取在线服务器列表，FGGateway返回在线服务器列表。
    5. FGClient 发送第三条协议：选服协议，FGGateway把FGClient和FGServer绑定。
    6. FGClient 接下来发送协议都会直接转发到FGServer。FGServer发送协议也会根据客户端的fd直接转发到FGClient。

FGServer和FGShmDB的交互，FGShmDB是用共享内存实现的内存数据库，只是简单的实现数据库的数据加载到内存，不支持数据从内存中移除，只是将用到的数据库中的数据按需加载到内存中，这样一来读取数据的速度会加快（第一次是从数据库中加载出来放到内存中，之后都是直接操作内存）。然后FGServer就是操作FGShmDB分享出来的内存。

FGLogin 是比较简单的一个http服务。主要是接收客户端的url请求，返回服务器列表给客户端。用于集合服务器。现在是直接用libevent的http实现的，用lua包装了一下。主要实现了用户注册，修改密码，用户登录，获取服务器列表等操作。其实可以用php实现它的，不知为啥会选用c。


