title: Ready to meeting
date: 2016-03-29 15:36:15
tags: [meeting]
categories": [android, meeting]
---

# Ready points
## 画圆
canvas.diawCircle
paint.setShader
## 触摸事件
## 下拉刷新原理
## LRU Cache的实现
## 绘制原理
## APP 架构 MVP MVVM CLEAN FLUX
## Glide 是如何设计的？相比 Picasso 有哪些优点和不足？Fresco 有什么特点？
这个，可以简单看下别人的文章，理解下流程。即可。
1. ![Triena 公众号文章](http://mp.weixin.qq.com/s?__biz=MzAxNjI3MDkzOQ==&mid=400056342&idx=1&sn=894325d70f16a28bfe8d6a4da31ec304&scene=2&srcid=10210byVbMGLHg7vXUJLgHaR&from=timeline&isappinstalled=0#rd)

## 加密算法 RSA AES DES MD5
## Handler  Looper ThreadLocal 这几个是怎么工作的。
特别是 怎么确保每个线程只有一个 Looper .
1. ThreadLocal 是确保每个线程都有一个数据的副本，所以Looper，在每个线程都会有独自的一个。
每个线程 Thread 有一个变量

```java
/**
     * Normal thread local values.
     */
    ThreadLocal.Values localValues;
```

Thread.Values 有一个成员变量

```java
/**
         * Map entries. Contains alternating keys (ThreadLocal) and values.
         * The length is always a power of 2.
         */
        private Object[] table;
```

 保存的键在，table[index], 保存的值在 table[index]. ThreadLocal 存值和取值方法如下。

 ```java
 public T get() {
         // Optimized for the fast path.
         Thread currentThread = Thread.currentThread();
         Values values = values(currentThread);
         if (values != null) {
             Object[] table = values.table;
             int index = hash & values.mask;
             if (this.reference == table[index]) {
                 return (T) table[index + 1];
             }
         } else {
             values = initializeValues(currentThread);
         }

         return (T) values.getAfterMiss(this);
     }
 ```


 ```java
 public void set(T value) {
        Thread currentThread = Thread.currentThread();
        Values values = values(currentThread);
        if (values == null) {
            values = initializeValues(currentThread);
        }
        values.put(this, value);
    }
```

2. 发送消息，怎么可以切换在不同线程中工作。
我们需要调用 looper.loop() ，才可以正常工作。这个调用时机是在不同的线程，因此这个 looper.loop()  
方法，调用发生在不同的线程，消息的最后处理，是在 looper.loop() 方法 中调用的 。因此可以起到了切换线程了。  

```java
msg.target.dispatchMessage(msg);
```

## AsyncTask 工作原理

## 数据库，可以聊聊 GreenDao 和 ORMLite 设计原理.
## 自定义 View 消除抗锯齿
1. canvas.setDrawFilter(new PaintFlagsDrawFilter(1, Paint.ANTI_ALIAS_FLAG));
2. mBitmapPaint.setAntiAlias(true);
3. mBitmapPaint.setFlags(Paint.ANTI_ALIAS_FLAG);

## AIDL
## Messenger
## Binder
## 多进程
## AMS
主线程的消息循环、主线程如何和AMS如何跨进程交互、SystemServer进程中的各种Service的工作方式  
ActivityThread 通过 AppicationThread 和 AMS 进行进程间通讯， AMS 完成 AppicationThread  
的请求后会回调 ApplicationThread 中的 Binder 方法，然后 ApplicationThread 会向  
ActivityThread 的 Hander 发消息。然后将 ApplicationThread 中的逻辑切换到 ActivityThread  
中去执行。

## 常用组件的细节
Activity的启动模式和标记位、Service同时处于start和bind状态时的停止问题
## 常见的设计模式
## 性能调试工具
例如AS 自带的或者MAT 插件
## 自定义 View 所需的 base 技能
搞懂view的滑动原理
- 搞懂如何实现弹性滑动
- 搞懂view的滑动冲突
- 搞懂view的measure、layout和draw
## 了解系统核心机制
1. 了解SystemServer的启动过程
2. 了解主线程的消息循环模型
3. 了解AMS和PMS的工作原理
4. 能够回答问题”一个应用存在多少个Window？
## 基本知识点的细节
1. Activity的启动模式以及异常情况下不同Activity的表现
2. Service的onBind和onReBind的关联
3. onServiceDisconnected(ComponentName className)和binderDied()的区别
4. AsyncTask在不同版本上的表现细节
5. 线程池的细节和参数配置
