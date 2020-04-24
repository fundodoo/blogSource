---
title: TCP 拆包和粘包
categories:
  - Netty
top: false
top_img:
  - /images/top4.jpeg
date: 2020-04-23 13:26:17
tags:
- Netty
keywords: TCP 拆包粘包,netty实战
description: TCP 拆包粘包,netty实战
comments:
cover: /images/netty_tcp_packing.jpg
---

## 简介
> Netty在基于TCP协议的网络通信中，存在**拆包**和**粘包**情况。拆包和粘包同时发生在数据的**发送方**与**接收方**。

> 发送方通过网络每发送一批二进制数据包，那么这次所发送的数据包就称为一帧(`Frame`)。在进行基于TCP的网络传输时，TCP协议会将用户真正要发送的数据根据当前缓存的实际情况对其进行拆分或重组，变为可用于网络传输的`Frame`。

> 在Netty中就是将`ByteBuf`中的数据拆分或重组为二进制的`Frame`。而接收方则需要将接收到的Frame中的数据进行重组或拆分，重新恢复为发送方发送时的`ByteBuf`数据。

### 具体场景描述
- 发送方发送的`ByteBuf`较大，在传输之前会被TCP底层拆分为多个`Frame`进行发送，这个过程称为发送拆包；接收方在接收到需要将这个`Frame`进行合并，这个合并的过程称为接收方粘包。
- 发送方发送的`ByteBuf`较小，无法形成一个`Frame`，此时TCP底层会将很多的这样的小的`ByteBuf`合并为一个`Frame`进行传输，这个合并的过程称为发送方的粘包；接收方在接收到这个Frame后需要进行拆包，拆分出多个原来的小的`ByteBuf`，这个拆分的过程称为接收方拆包。
- 当一个`Frame`无法放入整数倍个`ByteBuf`时，最后一个`ByteBuf`会发生拆包。这个`ByteBuf`中的一部分放入到了一个`Frame`中，另一部分被放入到了另一个`Frame`中。这个过程就是发送方的拆包。但对于将这些ByteBuf放入到一个`Frame`的过程，就是发送方的粘包；当接收方在接收到两个`Frame`后，对于第一个`Frame`的最后部分，与第二个`Frame`的最前部分会进行合并，这个合并的过程就是接收方粘包。但是在将`Frame`中的各个`ByteBuf`拆分出来的过程，就是接收方的拆包。

## 发送方拆包、粘包

### 需求--案例1
> 客户端作为发送方，向服务器发送<span style="color:red;">两个大的`ByteBuf`数据包</span>，这两个数据包会被拆分为若干个`Frame`进行发送。这个过程会发送拆包和粘包
> 服务端作为接收方，直接将接收到的`Frame`解码为String后进行显示，不对这个`Frame`进行粘包和拆包

#### 定义客户端

> 由于客户端仅发送数据，不接收服务端数据，所以这里仅需添加*String的编码器*，用于将字符串编码为`ByteBuf`，用于在TCP上进行传输。

```Java
package senderUnpacking.nettyDemo.fundodoo;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.CharsetUtil;

/**
 * @author 醉探索戈壁
 * @date 2020/4/23 14:02
 */
public class FundodooClient {

    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(eventLoopGroup)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            final ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8))
                                    .addLast(new FundodooClientUnpackingHanler());
                        }
                    });

            ChannelFuture future = bootstrap.connect("127.0.0.1", 8888).sync();
            future.channel().closeFuture().sync();
        }finally {
            if (eventLoopGroup != null) {
                eventLoopGroup.shutdownGracefully();
            }
        }
    }
}

```

> 在`channelActive()`中将数据发送两次，由于客户端不用读取服务器的数据，所以不用重写`channelRead()`方法

```Java
package senderUnpacking.nettyDemo.fundodoo;

import io.netty.channel.*;
import io.netty.util.concurrent.EventExecutorGroup;

/**
 * @author 醉探索戈壁
 * @date 2020/4/23 14:08
 */
public class FundodooClientUnpackingHanler extends ChannelInboundHandlerAdapter {

    private String message = "Netty is a NIO client server framework which enables " +
            "quick and easy development of network applications such as protocol servers " +
            "and clients. It greatly simplifies and streamlines network programming such " +
            "as TCP and UDP socket server.\n" +
            "\n" +
            "'Quick and easy' doesn't mean that a resulting application will suffer from " +
            "a maintainability or a performance issue. Netty has been designed carefully with " +
            "the experiences earned from the implementation of a lot of protocols such as FTP, " +
            "SMTP, HTTP, and various binary and text-based legacy protocols. As a result, Netty " +
            "has succeeded to find a way to achieve ease of development, performance, stability, " +
            "and flexibility without a compromise.";

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(message);
        ctx.writeAndFlush(message);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```
 
 #### 定义服务端

 > 由于服务端仅接收客户端的数据，不发送数据，所以这里仅添加String解码器，用于将ByteBuf解码为String

 ```Java
 package senderUnpacking.nettyDemo.fundodoo;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.util.CharsetUtil;

/**
 * @author 醉探索戈壁
 * @date 2020/4/23 14:19
 */
public class FundodooServer {

    public static void main(String[] args) throws InterruptedException {
        NioEventLoopGroup parentGroup = new NioEventLoopGroup();
        NioEventLoopGroup childGroup = new NioEventLoopGroup();

        ServerBootstrap serverBootstrap = new ServerBootstrap();
        try {
            serverBootstrap.group(parentGroup,childGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            final ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new StringDecoder(CharsetUtil.UTF_8))
                                    .addLast(new FundodooServerPrintHandler());
                        }
                    });

            ChannelFuture future = serverBootstrap.bind(8888).sync();
            System.out.println("服务已启动...");
            future.channel().closeFuture().sync();
        }finally {
            if (parentGroup != null) {
                parentGroup.shutdownGracefully();
            }
            if (childGroup != null) {
                childGroup.shutdownGracefully();
            }
        }
    }
}

 ```

 ```Java
 package senderUnpacking.nettyDemo.fundodoo;

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.concurrent.EventExecutorGroup;

/**
 * @author 醉探索戈壁
 * @date 2020/4/23 14:26
 */
public class FundodooServerPrintHandler extends ChannelInboundHandlerAdapter {

    private int counter;

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("Server 接收到的第【" + ++counter + "】个数据包："+ msg);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

 ```

 #### 运行结果
 > **Server 接收到的第【1】个数据包**：<span style="color:red">Netty is a NIO client server</span> framework which enables quick and easy development of network applications such as protocol servers and clients. It greatly simplifies and streamlines network programming such as TCP and UDP socket server.<br /><br />'Quick and easy' doesn't mean that a resulting application will suffer from a maintainability or a performance issue. Netty has been designed carefully with the experiences earned from the implementation of a lot of protocols such as FTP, SMTP, HTTP, and various binary and text-based legacy protocols. As a result, Netty has succeeded to find a way to achieve ease of development, performance, stability, and flexibility without a compromise.<span style="color:red">Netty is a NIO client server</span> framework which enables quick and easy development of network applications such as protocol servers and clients. It greatly simplifies and streamlines network programming such as TCP and UDP socket server.<br /><br />'Quick and easy' doesn't mean that a resulting application will suffer from a maintainability or a performanc
**Server 接收到的第【2】个数据包**：e issue. Netty has been designed carefully with the experiences earned from the implementation of a lot of protocols such as FTP, SMTP, HTTP, and various binary and text-based legacy protocols. As a result, Netty has succeeded to find a way to achieve ease of development, performance, stability, and flexibility without a compromise.


### 需求--案例2
> 客户端作为发送方，向服务端发送<span style="color:red;">100个小的ByteBuf数据包</span>，这100个数据包会被合并为若干个Frame进行发送。**这个过程会发生粘包与拆包**。
> 服务端作为接收方，直接将接收到的`Frame`解码为String后进行显示，**不**对这些`Frame`进行**粘包与拆包**。

#### 客户端定义

> 启动类和案例1一样，不需要更改，只需要修改客户端处理器类`FundodooClientUnpackingHanler`的`channelActive(...)`方法，修改如下

```Java
@Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
//        ctx.writeAndFlush(message);
//        ctx.writeAndFlush(message);
        for (int i=0; i<100; i++){
            ctx.writeAndFlush("Hello Netty");
        }
    }
```

> 或者去除启动类的String编码器,然后在channelActive()方法里手动编码

```Java
bootstrap.group(eventLoopGroup)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            final ChannelPipeline pipeline = ch.pipeline();
//                            pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8))// 去除String编码器，在自定义处理器里进行手工编码
                            pipeline.addLast(new FundodooClientUnpackingHanler());
                        }
                    });
```

```Java
@Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
//        ctx.writeAndFlush(message);
//        ctx.writeAndFlush(message);
        byte[] msgBytes = "Hello Netty".getBytes();
        ByteBuf buf = null;
        for (int i=0; i<100; i++){
            buf = Unpooled.buffer(msgBytes.length);
            buf.writeBytes(msgBytes);
            ctx.writeAndFlush(buf);
        }
    }
```

#### 定义服务端
> 和案例1一样

#### 运行结果
> Server 接收到的第【1】个数据包：Hello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello Netty
Server 接收到的第【2】个数据包：Hello NettyHello Netty
Server 接收到的第【3】个数据包：Hello NettyHello Netty
Server 接收到的第【4】个数据包：Hello NettyHello Netty
Server 接收到的第【5】个数据包：Hello Netty
Server 接收到的第【6】个数据包：Hello Netty
Server 接收到的第【7】个数据包：Hello Netty
Server 接收到的第【8】个数据包：Hello Netty
Server 接收到的第【9】个数据包：Hello Netty
Server 接收到的第【10】个数据包：Hello Netty
Server 接收到的第【11】个数据包：Hello Netty
Server 接收到的第【12】个数据包：Hello Netty
Server 接收到的第【13】个数据包：Hello Netty
Server 接收到的第【14】个数据包：Hello Netty
Server 接收到的第【15】个数据包：Hello Netty
Server 接收到的第【16】个数据包：Hello Netty
Server 接收到的第【17】个数据包：Hello Netty
Server 接收到的第【18】个数据包：Hello Netty
Server 接收到的第【19】个数据包：Hello Netty
Server 接收到的第【20】个数据包：Hello Netty
Server 接收到的第【21】个数据包：Hello Netty
Server 接收到的第【22】个数据包：Hello Netty
Server 接收到的第【23】个数据包：Hello Netty
Server 接收到的第【24】个数据包：Hello Netty
Server 接收到的第【25】个数据包：Hello Netty
Server 接收到的第【26】个数据包：Hello Netty
Server 接收到的第【27】个数据包：Hello Netty
Server 接收到的第【28】个数据包：Hello NettyHello Netty
Server 接收到的第【29】个数据包：Hello NettyHello Netty
Server 接收到的第【30】个数据包：Hello Netty
Server 接收到的第【31】个数据包：Hello Netty
Server 接收到的第【32】个数据包：Hello NettyHello Netty
Server 接收到的第【33】个数据包：Hello Netty
Server 接收到的第【34】个数据包：Hello Netty
Server 接收到的第【35】个数据包：Hello Netty
Server 接收到的第【36】个数据包：Hello NettyHello Netty
Server 接收到的第【37】个数据包：Hello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello Netty
Server 接收到的第【38】个数据包：Hello Netty
Server 接收到的第【39】个数据包：Hello Netty
Server 接收到的第【40】个数据包：Hello Netty
Server 接收到的第【41】个数据包：Hello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello NettyHello Netty
Server 接收到的第【42】个数据包：Hello NettyHello NettyHello Netty
Server 接收到的第【43】个数据包：Hello Netty
Server 接收到的第【44】个数据包：Hello Netty
Server 接收到的第【45】个数据包：Hello Netty
Server 接收到的第【46】个数据包：Hello Netty
Server 接收到的第【47】个数据包：Hello NettyHello NettyHello NettyHello Netty
Server 接收到的第【48】个数据包：Hello NettyHello NettyHello Netty
Server 接收到的第【49】个数据包：Hello Netty
Server 接收到的第【50】个数据包：Hello Netty
Server 接收到的第【51】个数据包：Hello Netty
Server 接收到的第【52】个数据包：Hello Netty
Server 接收到的第【53】个数据包：Hello Netty

### 服务端数据处理
> 见后续文章(**Netty_Netty 帧(Frame)解码器**)