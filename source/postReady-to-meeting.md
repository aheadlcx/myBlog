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
1. [Triena 公众号文章](http://mp.weixin.qq.com/s?__biz=MzAxNjI3MDkzOQ==&mid=400056342&idx=1&sn=894325d70f16a28bfe8d6a4da31ec304&scene=2&srcid=10210byVbMGLHg7vXUJLgHaR&from=timeline&isappinstalled=0#rd)

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

ThreadLocal.Values 有一个成员变量

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

# codeKK
## 要
1. 需要有明确的内容模块划分且每个模块内容精炼
比如常见的划分：必要的个人信息-主要经历(公司及学校)-专业技能-项目经验。

2. 重点突出工作经历及项目经验
项目经验重点突出项目中你的职责、贡献、突出点。

这是简历的重点，也是面试最主要聊的点。

简历的写法和面试都是有技巧的：要突出你擅长的并且面试官可能感兴趣的，引面试官入瓮。

框架的搭建，主要是服务层框架。
界面退出了，需要 cancel 还没有开始的 request，以及已经开始的request，需要 stop callback
气泡房源相关的触控事件 new library
弹框的封装，结合微博某文章 done
加密
js 和 本地通信


3. 用数字说话
比如团队整体业绩提高多少，项目带来什么收益或是节省多少成本；
App 性能提高多少、Crash 率下降多少；
专业排名(2/200)，员工考评(10/1000)占比之类。

大多数工程师可能不擅长这点，自己挖挖肯定有的。
now crash 0.8972%
total crash 1.7433%
users 22W
可以看 APP研发实录，其中一些 bug 来看看。
**找找友盟数据，以及加薪数据。**
4. PDF 版
Word 版在 Mac 上可能会乱。还收到过有情怀的 Pages 版简历😄
顺便说下
- 很多招聘网站导出来的简历 Mac 上都打不开；
- 猎头会改你简历内容，你要确保他不会删一些东西，比如个人博客、GitHub 等。

5. 格式整齐，段落有序
这个基本的要求，很多人都做不好。

我有代码洁癖，再者默认觉得 Word 排版不好的，要么不用心要么代码规范烂。

6. 明确写好在校及各公司的起始年限

7. 简历保持在两页左右
不至于太短也不至于信息太多，项目经验过多的可压缩，一般面试也就聊一两个主要项目就 ok 了。

当然如果你足够牛逼，给个博客地址或 GitHub 链接也 ok(态度貌似有点傲慢，哈哈)，当然尽量重点详细介绍你的项目经验。

8. 邮件标题及简单问候
邮件及简历附件标题尽量用姓名-原公司-职位。

codeKK 上一些内推职位是我发的，所以有时会收到一个仅含标题和附件的简历。

没有谢谢，没有问候，我的心是冰凉的/(ㄒoㄒ)/~~ 当然大多数情况这些简历的质量都很一般。

我对内推简历的回复率是 100%，而且只要不是太忙，我都会剪短说下简历需要修改的地方。

9. 简历常更新，常删除
收到过近十页的简历，七八年工作经验，项目一个没落，包括在校的实习经历。

简历内容要精简，最重要的是最近一家公司的经历，很多没必要的简介或是略过就行，两三页不能再多了。

## 不要
1. 不要用任何招聘网站的模板、不要 Word 版
尤其是智联招聘、51job 这类该被时代淘汰的站点。

维护一份 Word 版(发送时请用 PDF)，每年更新一次，不跳槽，也能梳理下自己。

2. 不要用“精通”二字
个人简历一直是熟悉 Android xx 部分、熟悉 Java。“精通”真的只能逗逗那些不靠谱公司。

3. 不要写任何国内培训经历、软件证书
即便你是半路出家也不要写“计算机四级之类的证书”，北大青鸟之流的培训就不说了。

有看到写着“获得教育部颁发的 Android 应用工程师证书”，无证程序员的我吓尿了

4. 不要写常年混迹于 xxx 社区、xxxx 论坛
什么版主之流那都是小白干的事，别在里面浪费时间了，没事逛逛 GitHub 这类高质清净的网站。

5. 不要自我评价
因为大多数人都写不好，也容易被某些病态 HR 抓把柄。

6. 项目经验中不要写软件环境、硬件环境、开发工具之类的

7. 个人博客、GitHub 如果没有什么内容就别放太显眼位置了
这种情况写自己每天逛 GitHub，对哪些项目有关注之类的反倒更有用。

当然如果博客、GitHub 很有料，请前置并且剪短介绍里面内容。

8. 头像不用，仅美女除外
9. 民族、政治面貌、联系地址一般情况都不用

以上适用于技术通用情况，特殊情况请勿参考。

所有举例不针对个人，只是把自己的感受经验分享出来，希望大家都有个靓丽的简历，为一份好的工作开个头。

如果大家觉得有用，并且有必要分享一些不错的模板，可点赞，必要的话我找些不错的模板分享出来。
