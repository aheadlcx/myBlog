title: data-binding笔记
date: 2016-03-01 15:51:40
tags: [data-binding, 笔记]
categories: android
---

本文，记述使用 data-binding 的一些注意事项（主要是，期待的特性，而没有的部分）
<!--more  -->
>[参考链接](https://github.com/LyndonChin/MasteringAndroidDataBinding)
>[杨辉的译文](http://yanghui.name/blog/2016/02/17/data-binding-guide/)

1， 带 id 的 View
在 ViewDataBinding 类中，只要一个 View 有id（例如id是 txtContent），那么就会有一个对应的
成员变量，并且是 public final 的，txtContent .  
这个虽然可以在外部使用，但是如果你 Refactor rename 这个id，会期望这个 ViewDataBinding 类的txtContent  
变量，也一起 rename ,可是事实并不会。因此，重构 rename 的时候，这点会很坑爹。


2, xml布局写错了，导致相应的databinding类生成不了
xml布局写错了，导致所有的databinding类生成不了，因此，会说找不到 databinding 这个包。  
最坑爹的是，**xml里面并没有提示那里错了**。。。


3，注意 import 的导入
例如：
```java
android:visibility="@{user.isAdult ? View.GONE : View.VISIBLE}"
```

如果你想把一些逻辑，直接写到xml里面去，这是可以，但是你必须导入相应的包。  
如果你不到加上这个
```java
<import type="android.view.View"/>
```

在你写 View.Gone 的时候，是不会有智能提示的，而且run的时候会报错。
