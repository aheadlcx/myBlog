title: View绘制注意点以及重要生命周期方法
date: 2015-12-21 19:38:09
tags:
---
<!-- 本文主要讲述对View的非常用用法以及自定义View要点 -->

>本文参考了众多前辈的文章和书籍
>>[任玉刚的相关文章和书籍](http://blog.csdn.net/singwhatiwanna)
>>[郭霖博客](http://blog.csdn.net/guolin_blog)
>>

# 拿到view的测试宽高
* activity的onWindowFocusChanged
onWindowFocusChanged方法，会被调用多次，每次失去焦点或者获得焦点，都会调用。（频繁onResune和Onpause，就会多次调用）  
* View.post(new Runnable(){//- [ ] 为什么投递到消息队列的尾部，就一定可以获取到宽高

  public void run(){
    view.GetMeasureWidth;
  }
  })
* ViewTreeObserver
加布局监听回调，***切记，在activity或者fragment销毁时，必须移除此监听***
* 手动measure,只有是具体数值或者wrap-content才可以手动测量。wrap-content 情况下是用如下代码测量的
这是因为View的measureSpec是使用低30位来算的，最大值就是2#30 -1 ,也就是 （1 << 30） -1.

```java
int widthMeasureSpec = measureSpec.makeMeasureSpec((1 << 30) -1 , measureSpec.AT_MOST)
view.measure(widthMeasureSpec, heightMeasureSpec);
```

# View绘制注意点
* 子View（如果是继承view，没有复写onMeasure 方法）绘制的时候，宽高，都是父View的剩余空间，不管是Match-parent还是wrap-content  
自定义View，如果继承与View，可以参考TextView的onMeasure 方法  
这是因为，在AT_MOST和EXACTLY下，父View调用getChildMeasureSpec方法时，主要子View不是写死的dp，给子View返回的measureSpec中  
的SpecSize都是父View的SpecSize - （父padding 和子view的margin）。view的onMeasure方法，在AT_MOST和EXACTLY下，都是直接SpecSize

