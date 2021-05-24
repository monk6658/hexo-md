---
title: JNA的使用 
date: 2021-04-09 14:16:33 
categories: JAVA #分类
tags: [java调用c++方法,JNA] 
description: java中使用dll类库中c、c++方法。
---

[TOC]

# 一、RMI TCP CONNECTION 内存问题

## 1. 问题概述

1. 内存有规律的涨到峰值，然后一次GC猛降？

![image-20210429093747283](/RMI TCP CONNECTION 内存问题/image-20210429093747283.png)

2. 在抽样器下有一个线程 **RMI TCP Connection** 一直以高速的字节分配运行，86w 字节/s ？

![image-20210429094021784](/RMI TCP CONNECTION 内存问题/image-20210429094021784.png)

## 2. 初步猜测

>理想状态：
>
>1. 这个线程是内存分析工具在计算分析应用所占用的内存
>
>非理想状态：
>
>1. 程序中一直存在垃圾对象，致使GC无法回收

## 3. 测试

1. 写一个demo，让主线程阻塞30分钟，然后用内存分析工具再去查看线程内存使用情况

```java
public class RMITcpDemo {
    public static void main(String[] args) {
        System.out.println("demo start...");
        try {
            Thread.sleep(30 * 60 * 1000);
            //阻塞主线程
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

2. 按道理这个demo只会创建一些基本的线程对象（垃圾回收线程、命令监听处理等线程），使用了连接工具之后，就会多出一个RMI TCP Connection线程去分析应用内存占用情况而且还是高速的分配字节内存使用

![image-20210429094832791](/RMI TCP CONNECTION 内存问题/image-20210429094832791.png)

# 二、参考链接

[1. RMI TCP CONNECTION 内存问题](https://blog.csdn.net/q82682810X/article/details/111400669)





