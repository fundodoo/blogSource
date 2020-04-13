---
title: netty 核心概念
categories: 
  - Netty
top: false
top_img:
  - /images/top3.jpeg
date: 2020-04-13 09:15:34
tags: Netty
keywords: Netty
description:
comments:
cover: /images/netty.jpeg
---

## Netty简介
> Netty是一个**异步事件驱动**的网络应用程序框架，用于**快速**开发**可维护**的**高性能**服务器和客户端

> Netty是一个**NIO**客户机-服务器框架，它支持快速、简单地开发网络应用程序；极大的**简化**了网络编程，如TCP和UDP套接字服务器

> **快速和简单** 并不意味着生成的应用程序将收到可维护性或性能问题的影响。Netty经过精心设计，并积累了许多协议(Ftp、Smtp、http、https...)的实施经验，以及各种二进制和基于文本的遗留协议。因此，Netty成功的找到了一种方法，在不妥协的情况下实现了**易于开发**、**高性能**、**稳定性**和**灵活性**。


## Netty中的核心概念

### Channel
> 管道，`Channel`是对`Socket`的封装，其包含了**一组API**，大大**简化**了直接对`Socket`进行**操作**的复杂性

### EventLoopGroup
> `EventLoopGroup`是一个`EventLoop`**池**，包含很多的`EventLoop`

> **Netty**为每个`Channel`分配一个`EventLoop`，用于处理用户的连接请求、对用户请求的处理等所有事件。`EventLoop`本身只是一个线程驱动，在其生命周期内只会绑定一个线程，让该线程处理一个`Channel`的所有IO事件

> 一个`Channel`一旦与一个`EventLoop`相绑定，那么在Channel的整个生命周期内是不能改变的。一个EventLoop可以与多个Channel绑定，即`Channel`与`EventLoop`的关系是 ***n:1***,而`EventLoop`与`线程`的关系是 ***1:1***。


### ServerBootStrap、BootStrap
> 用于配置整个Netty代码，将**各个组建关联**起来
> **服务端**使用的是`ServerBootStrap`
> **客户端**使用的是`BootStrap`

### ChannelHandler与ChannelPipeline
> `ChannelHandler`是对`Channel`中**数据的处理器**，这些处理器可以是系统本身定义好的编解码器，也可以用户自定义编解码器。这些处理器会被统一添加到一个`ChannelPipeline`的对象中，然后按照添加的数据依次对`Channel`中的**数据进行处理**

### ChannelFuture
> **Netty**中所有的I/O操作都是**异步**的，即操作不会立即得到返回结果，所以Netty中定义了一个`ChannelFuture`对象作为这个异步操作的**代言人**，表示异步操作本身。如果想获取到该异步操作的返回值，可以通过该异步操作对象的**addListener()**方法为该异步操作添加监听器，为其注册回调：当结果出来后马上调用执行
- Netty的异步编程模型都是建立在**Future与回调**概念之上的


## Netty执行流程
![Netty_process](/images/netty_process.png)

## Demo(Http)

```Xml
<!-- netty-all依赖       -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.36.Final</version>
</dependency>

<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.6</version>
    <scope>provided</scope>
</dependency>
```

```Java
package http.nettyDemo.fundodoo;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpServerCodec;

/**
 * @author drunk
 */
public class FundodooServer {

    public static void main(String[] args) throws InterruptedException {
        NioEventLoopGroup parentGroup = new NioEventLoopGroup();
        NioEventLoopGroup workGroup = new NioEventLoopGroup();

        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap.group(parentGroup,workGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast(new HttpServerCodec())
                                .addLast(new HttpCustomerHandler());

                    }
                });

        try {
            ChannelFuture future = serverBootstrap.bind(8888).sync();
            future.channel().closeFuture().sync();
        }finally {
            parentGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }

    }
}
```

```Java
package http.nettyDemo.fundodoo;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.CompositeByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.handler.codec.http.*;
import io.netty.util.CharsetUtil;

/**
 * @author drunk
 */
public class HttpCustomerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof HttpRequest) {
            HttpRequest httpRequest = (HttpRequest) msg;
            System.out.println("请求方式："+ httpRequest.method().name());

            if ("/favicon.ioc".equals(httpRequest.uri())) {
                System.out.println("不处理/favicon.ioc");
                return;
            }

            ByteBuf contents = Unpooled.copiedBuffer("hello Netty World", CharsetUtil.UTF_8);

            DefaultFullHttpResponse response = 
              new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK,contents);

            HttpHeaders httpHeaders = response.headers();
            httpHeaders.set(HttpHeaderNames.CONTENT_TYPE,"text/plain");
            httpHeaders.set(HttpHeaderNames.CONTENT_LENGTH,contents.readableBytes());

            ctx.writeAndFlush(response).addListener(ChannelFutureListener.CLOSE);
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
       cause.printStackTrace();
       ctx.close();
    }


}

```

## Socket编程
> 前面的工程是一个仅存在服务端的Http请求的服务器，而Netty中最为常见的是C/S构架的Socket代码。

### 需求
1. 客户端连接上服务端后，其马上会向服务端发送一个数据
2. 服务端接收到数据后，马上向客户端回复一个数据
3. 客户端收到数据后，便会向服务端发送一个数据

### 编码实现

```Java
package socket.nettyDemo.fundodoo;


import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.CharsetUtil;

/**
 * @author 醉探索戈壁
 *
 */
public class FundodooServer {

    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup parentGroup = new NioEventLoopGroup();
        EventLoopGroup workGroup = new NioEventLoopGroup();

        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(parentGroup,workGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8))
                                .addLast(new StringDecoder(CharsetUtil.UTF_8))
                                .addLast(new CustomSocketServerHandler());
                    }
                });
        try {
            ChannelFuture future = bootstrap.bind(8888).sync();
            future.channel().closeFuture().sync();
        }finally {
            parentGroup.shutdownGracefully();
            workGroup.shutdownGracefully();
        }
    }
}

```
```Java
package socket.nettyDemo.fundodoo;

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.concurrent.EventExecutorGroup;

import java.util.concurrent.TimeUnit;

/**
 * @author 醉探索戈壁
 * @date 2020/4/13 13:33
 */
public class CustomSocketServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println(ctx.channel().remoteAddress() + "===" + msg);
        ctx.writeAndFlush("from server " + System.currentTimeMillis());
        TimeUnit.MILLISECONDS.sleep(500);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```

```Java
package socket.nettyDemo.fundodoo;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.CharsetUtil;

import java.util.logging.SocketHandler;

/**
 * @author 醉探索戈壁
 * @date 2020/4/13 13:40
 */
public class FundodooClient {

    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup workGroup = new NioEventLoopGroup();

        Bootstrap bootstrap = new Bootstrap();
        bootstrap.group(workGroup)
                .channel(NioSocketChannel.class)
                .handler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8))
                                .addLast(new StringDecoder(CharsetUtil.UTF_8))
                                .addLast(new CustomSocketClientHandler());
                    }
                });
        try {
            ChannelFuture future = bootstrap.connect("localhost", 8888).sync();
            future.channel().closeFuture().sync();
        }finally {
            if (null != workGroup) {
                workGroup.shutdownGracefully();
            }
        }
    }
}

```
```Java
package socket.nettyDemo.fundodoo;

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.concurrent.EventExecutorGroup;

import java.util.concurrent.TimeUnit;

/**
 * @author 醉探索戈壁
 * @date 2020/4/13 13:48
 */
public class CustomSocketClientHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println(msg);
        ctx.writeAndFlush("from client" + System.currentTimeMillis());
        TimeUnit.MILLISECONDS.sleep(500);
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush("connected:" + System.currentTimeMillis());
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```