---
title: netty分包粘包问题 
date: 2021-03-30 14:16:33 
categories: 并发、netty #分类
tags: [锁netty,并发,分包粘包] #文章标签，可空，多标签请用格式，注意:后面有个空格
description: netty自定义协议，实现分包粘包问题。
typora-copy-images-to: netty
---

## 基于长度协议的分包

注：本文 length 均为 16进制表示，eg:  1232 ==> 04D0

[TOC]

### 1. 业务场景介绍

基于长度协议分包（length + value）

在和c语言对接过程中，因数据包长度大于一定长度后分包发送，致使handler接收数据不全出现问题。

### 2. 具体现象描述

1. c语言方 每1011字节就会自动分多小包发送
2. 使用netty自带的长度协议 LengthFieldBasedFrameDecoder 只能接收到半包

### 3. 解决思路

自定义长度协议包，根据length 进行判断包是否完整，进而决定是否进逻辑层处理

### 4. 具体实现

#### 4.1 服务端实现

1. 基础服务端server，三个处理器分别为：日志、自定义长度协议、逻辑处理

```java
@Slf4j
public class NettyServer {

    public static void main(String[] args) {

        // 1. 创建两个工作线程组，两个无线循环(默认线程组为   cpu核数 * 2 )
        EventLoopGroup bossGroup = new NioEventLoopGroup(1); // 处理客户端的链接
        EventLoopGroup workerGroup = new NioEventLoopGroup(8);  // 网络的读写
        try {
            // 2. 服务启动配置
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup,workerGroup)
                    .channel(NioServerSocketChannel.class)  // 使用NioServerSocketChannel 作为服务端的通道实现
                    .option(ChannelOption.SO_BACKLOG, 128)  // 设置线程队列连接个数
                    .childOption(ChannelOption.SO_KEEPALIVE, true)  // 设置保持活动连接状态
                    .childHandler(new ChannelInitializer<SocketChannel>() {  //创建一个通道初始化对象
                        @Override
                        protected void initChannel(SocketChannel ch) { // 给pipeline设置处理器
                            ch.pipeline().addLast(new LoggingHandler(LogLevel.INFO));
                            // 自定义长度协议
                            ch.pipeline().addLast(new MyLengthFieldBasedFrameDecoder(1024,0,2));
                            // netty官方长度协议
//                            ch.pipeline().addLast(new LengthFieldBasedFrameDecoder(1024*2,0,2));
                            ch.pipeline().addLast(new NettyServerHandler());
                        }
                    });
            ChannelFuture channelFuture = bootstrap.bind(9999).sync();
            channelFuture.channel().closeFuture().sync();
        }catch (Exception e){
            log.error("启动异常，优雅关闭",e);
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

}
```

#### 4.2 自定义长度协议处理器

1. 基础构造函数，获取基础数据信息：包最大长度、长度位起始位置、长度位结束位置、逻辑数据截取起始位置、逻辑数据截取长度
2. 使用netty自带当前线程变量记录当前线程基础数据
3. 根据 **_length_** 判断包是否完整
4. 若包完整，则使用 **_super.channelRead(XX,XX);_** 进入下一个handler

```java
/**
 * netty 服务端逻辑处理类（长度为16进制编码）
 * 这种分包，不支持奇数16进制分包，必须是整个字节，eg：000430245678  分包：3024567   、8 错误。
 * 必须两个十六进制组成一个byte分包才对
 * @author zxl
 * @date 2021/3/23 10:29
 */
@Slf4j
public class MyLengthFieldBasedFrameDecoder extends ChannelInboundHandlerAdapter {

    /*** 实际长度 */
    private FastThreadLocal<Integer> actualLength = new FastThreadLocal<>();
    /*** 实际内容 */
    private FastThreadLocal<byte[]> actualContent = new FastThreadLocal<>();
    /*** 最大长度 */
    private FastThreadLocal<Integer> maxFrameLength = new FastThreadLocal<>();
    /*** 长度位数起始下标 */
    private FastThreadLocal<Integer> lengthFieldOffset = new FastThreadLocal<>();
    /*** 长度位数结束下标 */
    private FastThreadLocal<Integer> lengthFieldLength = new FastThreadLocal<>();
    /*** 长度调整起始下标 */
    private FastThreadLocal<Integer> lengthAdjustment = new FastThreadLocal<>();
    /*** 长度调整结束下标 */
    private FastThreadLocal<Integer> initialBytesToStrip = new FastThreadLocal<>();

    /**
     * 基础构造方法
     * @param maxFrameLength 最大长度
     * @param lengthFieldOffset 长度下标起始位置
     * @param lengthFieldLength 长度所占字节位数
     */
    public MyLengthFieldBasedFrameDecoder(int maxFrameLength, int lengthFieldOffset, int lengthFieldLength) {
        this(maxFrameLength, lengthFieldOffset, lengthFieldLength, 0, 0);
    }

    /**
     * 基础构造方法
     * @param maxFrameLength 最大长度
     * @param lengthFieldOffset 长度下标起始位置
     * @param lengthFieldLength 长度所占字节位数
     * @param lengthAdjustment 真实数据下标起始位置
     * @param initialBytesToStrip 初始化略过字节数
     */
    public MyLengthFieldBasedFrameDecoder(int maxFrameLength, int lengthFieldOffset, int lengthFieldLength, int lengthAdjustment, int initialBytesToStrip) {
        if (maxFrameLength <= 0) {
            throw new IllegalArgumentException("maxFrameLength must be a positive integer: " + maxFrameLength);
        } else if (lengthFieldOffset < 0) {
            throw new IllegalArgumentException("lengthFieldOffset must be a non-negative integer: " + lengthFieldOffset);
        } else if (initialBytesToStrip < 0) {
            throw new IllegalArgumentException("initialBytesToStrip must be a non-negative integer: " + initialBytesToStrip);
        } else if (lengthFieldOffset > maxFrameLength - lengthFieldLength) {
            throw new IllegalArgumentException("maxFrameLength (" + maxFrameLength + ") must be equal to or greater than lengthFieldOffset (" + lengthFieldOffset + ") + lengthFieldLength (" + lengthFieldLength + ").");
        }else{
            this.maxFrameLength.set(maxFrameLength);
            this.lengthFieldOffset.set(lengthFieldOffset);
            this.lengthFieldLength.set(lengthFieldLength);
            this.lengthAdjustment.set(lengthAdjustment);
            this.initialBytesToStrip.set(initialBytesToStrip);
        }
    }

    /**
     * 实际数据读取处理
     * @param ctx 上下文
     * @throws Exception 执行异常
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        // 1. 数据读取
        ByteBuf byteBuf = (ByteBuf) msg;
        byte[] requestMsg = new byte[byteBuf.readableBytes()];
        byteBuf.readBytes(requestMsg);

        // 2. 首次消息判断,并对数据处理
        if(actualContent.get() == null){
            String receiveMsg = bytesToHex(requestMsg);
            // 因为是字节数据长度，2个字节，故需要截取4位长度。下移开始下标*2，假设不是 length + value 格式
            String length = receiveMsg.substring(this.lengthFieldOffset.get(), this.lengthFieldLength.get()*2);
            // 默认长度为 16进制表示
            int i = Integer.parseInt(length, 16);
            log.info("第一次数据接收:{},数据长度:{}",receiveMsg,i);
            if(i + this.lengthFieldLength.get() == requestMsg.length){
                super.channelRead(ctx,requestMsg);
            }
            // 数据长度，带长度位
            actualLength.set(i + this.lengthFieldLength.get());
            actualContent.set(requestMsg);
        }else{
            // 3. 二次分包处理
            byte[] content = actualContent.get();
            int newLength = content.length + requestMsg.length;
            if(newLength > maxFrameLength.get()){
                throw new TooLongFrameException("Adjusted frame length exceeds " + this.maxFrameLength.get() + " - discarding");
            }

            byte[] next = appendByteArray(content,requestMsg);
            log.info("next:{}",bytesToHex(next));
            if(actualLength.get() == newLength){
                byte[] actByte = null;
                if(lengthAdjustment.get() == 0){
                    actByte = Arrays.copyOfRange(next, initialBytesToStrip.get(), next.length);
                }else{
                    byte[] startByte = Arrays.copyOfRange(next, 0, lengthAdjustment.get());
                    byte[] endByte = Arrays.copyOfRange(next, initialBytesToStrip.get(), next.length);
                    actByte = appendByteArray(startByte,endByte);
                }
                super.channelRead(ctx,actByte);
            }else{
                actualContent.set(next);
            }
        }
    }

    /**
     * 执行中异常
     * @param ctx 上下文
     * @param cause 执行异常
     * @throws Exception 方法体异常
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        log.error("执行中异常",cause);
        releaseThreadLocalResource();
        ctx.close();
    }

    /**
     * 连接断开时，释放当前线程变量资源
     * @param ctx 上下文
     * @throws Exception 执行中异常
     */
    @Override
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
        releaseThreadLocalResource();
        log.info("客户端连接断开了+++++++{}",ctx.channel().remoteAddress().toString());
        super.channelUnregistered(ctx);
    }

    /**
     * 释放当前线程变量
     */
    private void releaseThreadLocalResource(){
        maxFrameLength.remove();
        lengthFieldOffset.remove();
        lengthFieldLength.remove();
        lengthAdjustment.remove();
        initialBytesToStrip.remove();
        actualLength.remove();
        actualContent.remove();
    }

    /**
     * byte[] 数组转16进制字符串
     * @param bytes 字节数据
     * @return 16进制字符串
     */
    private static String bytesToHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder();
        for (byte aByte : bytes) {
            String hex = Integer.toHexString(aByte & 0xFF);
            if (hex.length() < 2) {
                sb.append(0);
            }
            sb.append(hex);
        }
        return sb.toString();
    }

    /**
     * 字节数组拼接
     * @param array1 拼接第一个数组
     * @param array2 第二个数组
     * @return 拼接结果
     */
    private byte[] appendByteArray(byte[] array1,byte[] array2){
        byte[] newArray = new byte[array1.length + array2.length];
        System.arraycopy(array1,0,newArray,0,array1.length);
        System.arraycopy(array2,0,newArray,array1.length,array2.length);
        return newArray;
    }
}
```

#### 4.3 逻辑处理器

1. 获取上一个 **_handler_** 处理之后的数据
2. **_channelRead_** 方法中 **_Object msg_** 对象类型是依据上一个 **_handler_**  中 **_channelRead()_** 传入的类型而定。即 **_super.channelRead(XX,XX);_**  第二个XX 所代表的类型
3. 因上一个 **_handler_** 传入为byte[] ，故下方以byte[] 接收。
4. 原生netty 长度协议 则为ByteBuf 接收。

```java
/**
 * netty 服务端逻辑处理类
 * @author zxl
 * @date 2021/3/23 10:29
 */
@Slf4j
public class NettyServerHandler extends ChannelInboundHandlerAdapter {

    /**
     * 连接注册时执行
     * @param ctx 上下文
     * @throws Exception 执行异常
     */
    @Override
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        log.info("客户端连接{}",ctx.channel().remoteAddress().toString());
        super.channelRegistered(ctx);
    }

    /**
     * 连接关闭时执行
     * @param ctx 上下文
     * @throws Exception 执行异常
     */
    @Override
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
        log.info("客户端连接断开了{}",ctx.channel().remoteAddress().toString());
        super.channelUnregistered(ctx);
    }

    /**
     * 连接活跃
     * @param ctx 上下文
     * @throws Exception 执行异常
     */
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        log.info("连接活跃，有信息进入");
    }

    /**
     * 连接不活跃
     * @param ctx 上下文
     * @throws Exception 执行异常
     */
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        log.info("连接不活跃，信息写完发送完毕");
    }
    /**
     * 实际数据读取处理
     * @param ctx 上下文
     * @throws Exception 执行异常
     */
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {

        if(msg instanceof byte[]){
            byte[] requestMsg = (byte[]) msg;
            log.info("最终接收数据----:{}",bytesToHex(requestMsg));
        }else{
            ByteBuf byteBuf = (ByteBuf) msg;
            byte[] requestMsg = new byte[byteBuf.readableBytes()];
            byteBuf.readBytes(requestMsg);
            log.info("ByteBuf 最终接收数据----:{}",bytesToHex(requestMsg));
        }
    }

    /**
     * 实际数据读取完成处理
     * @param ctx 上下文
     * @throws Exception 执行异常
     */
    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        log.info("数据读取完毕之后 channelReadComplete ");
    }


    /**
     * 连接事件判断
     * @param ctx 上下文
     * @param evt 事件对象
     * @throws Exception 执行异常
     */
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        log.info("channel 用户事件触发");
    }

    /**
     * 实际数据读取完成处理 ,关闭通道
     * @param ctx 上下文
     * @param cause 执行异常
     * @throws Exception 执行异常
     */
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        log.error("执行中异常",cause);
        ctx.close();
    }


    /**
     * byte[] 数组转16进制字符串
     * @param bytes 字节数据
     * @return 16进制字符串
     */
    private static String bytesToHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder();
        for (byte aByte : bytes) {
            String hex = Integer.toHexString(aByte & 0xFF);
            if (hex.length() < 2) {
                sb.append(0);
            }
            sb.append(hex);
        }
        return sb.toString();
    }
}
```

### 5. 测试方式

> 上述代码为 长度均为2个字节，故以下也遵循代码走。
>
> 测试数据包：`000C321453215435413415206753`
>
> 000C ==>  12 (十进制)
>
> 注：传输方式是以byte[] 进行传输，转16进制后，每两个16进制数为一个byte，故整体数据包长度一定为偶数。故测分包问题时，切不可把16进制表示的数据包分奇数拆分，奇数拆分后涉及到byte自动补零问题，最终导致分包测试失败，逻辑数据不准确。

1. 使用socket发送工具，组长度协议包( lenth[0x] + value)     

![image-20210331151558717](\netty\image-20210331151558717.png)

2. 先整体发送 `000C321453215435413415206753`，查看是否会进入逻辑handler。查看没问题

   ![image-20210331152602573](\netty\image-20210331152602573.png)

3. 关闭工具重新打开（目前都是短连接测试），先发 `000C32145321543541341520`  ，再发`6753` 查看结果

   ![image-20210331153715016](\netty\image-20210331153715016.png)

4. 上述测试没有问题，使用厂商机器测试

### 6. 注意问题

1. 传输方式是以byte[] 进行传输，转16进制后，每两个16进制数为一个byte，故整体数据包长度一定为偶数。故测分包问题时，切不可把16进制表示的数据包分奇数拆分，奇数拆分后涉及到byte自动补零问题，最终导致分包测试失败，逻辑数据不准确。
2. 使用socket工具发送，使用netty原生的`LengthFieldBasedFrameDecoder`即可满足需求
3. `LengthFieldBasedFrameDecoder`长度协议，长度也必须为16进制表示，不然取前几位转16进制之后，容易抛出异常`Adjusted frame length exceeds`
4. 如若没有长度协议，服务端会每1024字节分包读取（待解决处理）

## 其他分包解决方案

1. 定长协议 

   ```java
   new FixedLengthFrameDecoder(1024)
   ```

2. 分隔符协议

   ```java
   //设置特殊分隔符
   ByteBuf buf = Unpooled.copiedBuffer("_".getBytes());
   new DelimiterBasedFrameDecoder(1024, buf);
   ```

3. 待补充。。


