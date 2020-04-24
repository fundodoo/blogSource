---
title: Netty 帧(Frame)解码器
categories:
  - Netty
top: false
top_img:
  - /images/top5.jpeg
date: 2020-04-24 09:18:58
tags:
- Netty
keywords: Netty帧解码器案例,Netty
description:
comments:
cover: /images/netty_tcp_packing.jpg
---

## Netty帧解码器
> 为了解决接收方接收到的数据的混乱性，接收方也可以对接收到的Frame包进行粘包与拆包。
> Netty中以及定义好了很多的接收方粘包拆包解决方案，我们可以直接使用。
> 接收方的粘包和拆包实际在做的工作就是解码工作。这个解码基本思想是：发送方在发送数据中添加一个分隔标记，并告诉接收方标记是什么。这样在接收方接收到Frame后，其会根据事先约定的**分隔标记**将**数据进行拆分或合并**，产生相应的`ByteBuf`数据。这个拆分或合并的过程，称为接**收方的拆包与粘包**

### LineBasedFrameDecoder
> 基于**行分隔符**的帧解码器，即会按照<span style="color:red;">行分隔符</span>对数据进行*拆包粘包*，解码出ByteBuf

#### 客户端启动类
```Java
package decoder.nettyDemo.fundodoo;

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
 * @date 2020/4/24 10:00
 */
public class FundodooClient {

    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.group(eventLoopGroup)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            final ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8))
                                    .addLast(new FundodooClientHandler());
                        }
                    });

            ChannelFuture future = bootstrap.connect("localhost",8888).sync();
            future.channel().closeFuture().sync();
        }finally {
            if (eventLoopGroup != null) {
                eventLoopGroup.shutdownGracefully();
            }
        }
    }
}

```
#### 客户端处理器
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.concurrent.EventExecutorGroup;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:06
 */
public class FundodooClientHandler extends ChannelInboundHandlerAdapter {

    private String msg = "Hello Fundodoo" + System.getProperty("line.separator");

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        for (int i = 0; i < 50; i++) {
            ctx.writeAndFlush(msg);
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```
#### 服务端启动类
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.LineBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.util.CharsetUtil;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:12
 */
public class FundodooServer {

    public static void main(String[] args) throws InterruptedException {
        final NioEventLoopGroup parentGroup = new NioEventLoopGroup();
        final NioEventLoopGroup childGroup = new NioEventLoopGroup();

        ServerBootstrap bootstrap = new ServerBootstrap();
        try {
            bootstrap.group(parentGroup,childGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            final ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new LineBasedFrameDecoder(512)) //添加行分隔符解码器
                                    .addLast(new StringDecoder(CharsetUtil.UTF_8))
                                    .addLast(new FundodooServerhandler());
                        }
                    });

            ChannelFuture future = bootstrap.bind(8888).sync();
            future.channel().closeFuture().sync();
        }finally {
            parentGroup.shutdownGracefully();
            childGroup.shutdownGracefully();
        }

    }
}

```
#### 服务端处理器
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.concurrent.EventExecutorGroup;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:23
 */
public class FundodooServerhandler extends ChannelInboundHandlerAdapter {

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
> 请自行运行程序 **-_-**

### DelimiterBasedFrameDecoder
> 基于**分隔符**的帧解码器，即会按照<span style="color:red;">指定分隔符</span>对数据进行*拆包粘包*，解码出`ByteBuf`

> Demo分隔符为：**###Fundodoo###**

#### 客户端启动类
```Java
package decoder.nettyDemo.fundodoo;

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
 * @date 2020/4/24 10:00
 */
public class FundodooClient {

    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.group(eventLoopGroup)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            final ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8))
                                    .addLast(new FundodooClientHandler());
                        }
                    });

            ChannelFuture future = bootstrap.connect("localhost",8888).sync();
            future.channel().closeFuture().sync();
        }finally {
            if (eventLoopGroup != null) {
                eventLoopGroup.shutdownGracefully();
            }
        }
    }
}

```
#### 客户端处理器
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.concurrent.EventExecutorGroup;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:06
 */
public class FundodooClientHandler extends ChannelInboundHandlerAdapter {

    private String msg = "Netty is a NIO client server framework which enables ###Fundodoo###" +
            "quick and easy development of network applications such as protocol ###Fundodoo###" +
            "servers and clients. It greatly simplifies and streamlines network ###Fundodoo###" +
            "programming such as TCP and UDP socket server.";

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(msg);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```
#### 服务端启动类
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.DelimiterBasedFrameDecoder;
import io.netty.handler.codec.LineBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.util.CharsetUtil;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:12
 */
public class FundodooServer {

    public static void main(String[] args) throws InterruptedException {
        final NioEventLoopGroup parentGroup = new NioEventLoopGroup();
        final NioEventLoopGroup childGroup = new NioEventLoopGroup();

        ServerBootstrap bootstrap = new ServerBootstrap();
        try {
            bootstrap.group(parentGroup,childGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            final ChannelPipeline pipeline = ch.pipeline();
                            ByteBuf delimiter = Unpooled.copiedBuffer("###Fundodoo###".getBytes());
                            pipeline.addLast(new DelimiterBasedFrameDecoder(512,delimiter))
                                    .addLast(new StringDecoder(CharsetUtil.UTF_8))
                                    .addLast(new FundodooServerhandler());
                        }
                    });

            ChannelFuture future = bootstrap.bind(8888).sync();
            future.channel().closeFuture().sync();
        }finally {
            parentGroup.shutdownGracefully();
            childGroup.shutdownGracefully();
        }

    }
}

```
#### 服务端处理器
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.concurrent.EventExecutorGroup;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:23
 */
public class FundodooServerhandler extends ChannelInboundHandlerAdapter {

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
> Server 接收到的第【1】个数据包：Netty is a NIO client server framework which enables 
Server 接收到的第【2】个数据包：quick and easy development of network applications such as protocol 
Server 接收到的第【3】个数据包：servers and clients. It greatly simplifies and streamlines network


### FixedLengthFrameDecoder
> 基于**固定长度**帧解码器，即会按照<span style="color:red;">指定的长度</span>对`Frame`中的数据进行拆包粘包

> Demo使用的长度为**10**

#### 客户端启动类
```Java
package decoder.nettyDemo.fundodoo;

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
 * @date 2020/4/24 10:00
 */
public class FundodooClient {

    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.group(eventLoopGroup)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            final ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8))
                                    .addLast(new FundodooClientHandler());
                        }
                    });

            ChannelFuture future = bootstrap.connect("localhost",8888).sync();
            future.channel().closeFuture().sync();
        }finally {
            if (eventLoopGroup != null) {
                eventLoopGroup.shutdownGracefully();
            }
        }
    }
}

```
#### 客户端处理器
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.concurrent.EventExecutorGroup;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:06
 */
public class FundodooClientHandler extends ChannelInboundHandlerAdapter {

    private String msg = "Netty is a NIO client server framework which enables" +
            "quick and easy development of network applications such as protocol" +
            "servers and clients.";

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(msg);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```
#### 服务端启动类
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.DelimiterBasedFrameDecoder;
import io.netty.handler.codec.FixedLengthFrameDecoder;
import io.netty.handler.codec.LineBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.util.CharsetUtil;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:12
 */
public class FundodooServer {

    public static void main(String[] args) throws InterruptedException {
        final NioEventLoopGroup parentGroup = new NioEventLoopGroup();
        final NioEventLoopGroup childGroup = new NioEventLoopGroup();

        ServerBootstrap bootstrap = new ServerBootstrap();
        try {
            bootstrap.group(parentGroup,childGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            final ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new FixedLengthFrameDecoder(10))
                                    .addLast(new StringDecoder(CharsetUtil.UTF_8))
                                    .addLast(new FundodooServerhandler());
                        }
                    });

            ChannelFuture future = bootstrap.bind(8888).sync();
            future.channel().closeFuture().sync();
        }finally {
            parentGroup.shutdownGracefully();
            childGroup.shutdownGracefully();
        }

    }
}

```
#### 服务端处理器
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.concurrent.EventExecutorGroup;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:23
 */
public class FundodooServerhandler extends ChannelInboundHandlerAdapter {

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
> Server 接收到的第【1】个数据包：Netty is a
Server 接收到的第【2】个数据包： NIO clien
Server 接收到的第【3】个数据包：t server f
Server 接收到的第【4】个数据包：ramework w
Server 接收到的第【5】个数据包：hich enabl
Server 接收到的第【6】个数据包：esquick an
Server 接收到的第【7】个数据包：d easy dev
Server 接收到的第【8】个数据包：elopment o
Server 接收到的第【9】个数据包：f network 
Server 接收到的第【10】个数据包：applicatio
Server 接收到的第【11】个数据包：ns such as
Server 接收到的第【12】个数据包： protocols
Server 接收到的第【13】个数据包：ervers and


### LengthFieldBasedFrameDecoder
> 基于**长度域**的帧解码器，用于对`LengthFieldPrepender`**编码器**编码后的数据进行解码的。

#### 构造器参数
- maxFrameLength：要解码的Frame的最大长度
- lengthFieldOffset：长度域的偏移量
- lengthFieldLength：长度域的长度
- lengthAdjustment：要添加到长度域值中的补偿值，长度矫正值
- initialBytesToStrip：从解码帧中要剥去的前面字节

#### 客户端启动类
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.LengthFieldBasedFrameDecoder;
import io.netty.handler.codec.LengthFieldPrepender;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.CharsetUtil;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:00
 */
public class FundodooClient {

    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

        Bootstrap bootstrap = new Bootstrap();
        try {
            bootstrap.group(eventLoopGroup)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            final ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new LengthFieldBasedFrameDecoder(50,0,4,0,4))
                                    .addLast(new LengthFieldPrepender(4))
                                    .addLast(new StringDecoder(CharsetUtil.UTF_8))
                                    .addLast(new StringEncoder(CharsetUtil.UTF_8))
                                    .addLast(new FundodooClientHandler());
                        }
                    });

            ChannelFuture future = bootstrap.connect("localhost",8888).sync();
            future.channel().closeFuture().sync();
        }finally {
            if (eventLoopGroup != null) {
                eventLoopGroup.shutdownGracefully();
            }
        }
    }
}

```
#### 客户端处理器
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.concurrent.EventExecutorGroup;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:06
 */
public class FundodooClientHandler extends ChannelInboundHandlerAdapter {

    private String msg = "Netty is a NIO client server framework";

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.writeAndFlush(msg);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}

```
#### 服务端启动类
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.*;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.CharsetUtil;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:12
 */
public class FundodooServer {

    public static void main(String[] args) throws InterruptedException {
        final NioEventLoopGroup parentGroup = new NioEventLoopGroup();
        final NioEventLoopGroup childGroup = new NioEventLoopGroup();

        ServerBootstrap bootstrap = new ServerBootstrap();
        try {
            bootstrap.group(parentGroup,childGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            final ChannelPipeline pipeline = ch.pipeline();
                            pipeline.addLast(new LengthFieldBasedFrameDecoder(50,0,4,0,4))
                                    .addLast(new LengthFieldPrepender(4))
                                    .addLast(new StringDecoder(CharsetUtil.UTF_8))
                                    .addLast(new StringEncoder(CharsetUtil.UTF_8))
                                    .addLast(new FundodooServerhandler());
                        }
                    });

            ChannelFuture future = bootstrap.bind(8888).sync();
            future.channel().closeFuture().sync();
        }finally {
            parentGroup.shutdownGracefully();
            childGroup.shutdownGracefully();
        }

    }
}

```
#### 服务端处理器
```Java
package decoder.nettyDemo.fundodoo;

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.concurrent.EventExecutorGroup;

/**
 * @author 醉探索戈壁
 * @date 2020/4/24 10:23
 */
public class FundodooServerhandler extends ChannelInboundHandlerAdapter {

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
> Server 接收到的第【1】个数据包：Netty is a NIO client server framework