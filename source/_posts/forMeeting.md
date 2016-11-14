title: forMeeting
date: 2016-08-31 17:41:17
tags:
---

## 知识点

###  hasCode 和 equals 方法
equals 方法是表示逻辑相等。
如果覆盖了 equals 方法，那么就必须重写 hasCode 方法。
hasCode 方法的意思是，  
+ 如果在一个执行当中，equals 相等，那么就必须返回2个一样的整数。  
+ 如果在一次执行当中，equals 不相等，虽然没有强制要求返回2个不一样的整数，但是最好  
返回不一样的整数，这是很多容器 需要用到的（ 很多 map 类）

### 线程相关 Thread Runnable  Callable  Future  FutureTask
开子线程就2种办法：  
+ Thread.start  
+ ExecuteService.execute( Runnable )


Callable  功能和 Runnable 类似，但是它是有返回值的。  
Future 是和 Callable 配合使用的，可以查询是否执行完毕和结果。是一个接口类。  
FutureTask 是实现了 Runnable 和 Future 。

### 线程池
#### 线程池的理解
Executor 是一个线程池接口， ThreadPoolExecutor 是线程池的真正实现  

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         threadFactory, defaultHandler);
}
```

+ corePoolSize 是核心线程池数目，核心线程，默认是一直活着的。除非把  
 allowCoreThreadTimeOut 设置为 true ，则在超过了 keepAliveTime 之后就会被终止。  
+ maximumPoolSize ,最大的活跃线程数量，正在运行状态的线程，不能超过这个数量。超过的就  
进入到阻塞状态。
+ keepAliveTime 这个就是非核心线程超过这个时间，就会被终止。
+ unit 时间单位，TimeUnit.Seconds 秒
+ workQueue 任务队列，execute 提交的 runnable 就保存在这里。
+ threadFactory 线程工厂，提供新线程的方法


一般的线程池是无序的，不能确保执行顺序的，如果需要自定义一个有序实行的线程池的话，高版本的  
AsyncTask 就有一个很好的例子了。

```java
private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
                        scheduleNext();
                    }
                }
            });
            if (mActive == null) {
                scheduleNext();
            }
        }

        protected synchronized void scheduleNext() {
            if ((mActive = mTasks.poll()) != null) {
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```

####  提供的几个默认线程池
+ FixedThreadPool  
Executors.newFixedThreadPool( int nThreads)  
固定数量的线程池，永远活着。
+ CachedThreadPool  
Executors.newCachedThreadPool 来生成。  
没有核心线程，线程的数量可以说无限大，如果没有空闲的线程，就新开。如果有空闲线程，就复用。  
是适合大量的耗时较短的情景
+ ScheduledThreadPool  
核心线程数量是固定的，非核心线程是不固定的，可以回收闲置的非核心线程。因此比较适合，  
有固定周期的重复任务和定时任务的结合的情景。
+ SingleThreadExecutor  
只有一个核心线程。因此执行肯定是顺序的了。不需要处理线性安全问题了

### 注解
常用的情景，有2个关键地方：  
+ 注解的生命周期 @Retention  
RetentionPolicy.SOURCE 是仅仅编译器有效果。  
RetentionPolicy.RUNTIME 运行时有效果。
+ 作用域 @TARGET  
ElementType.TYPE 和 ElementType.FEILD 等  

### dp px sp
dp 和 sp 基本一样，sp 可以跟随系统设置 （初衷是好，但是容易造成 bug ）。  
PPI = √（长度像素数² + 宽度像素数²） / 屏幕对角线英寸数
dp和px的换算公式 ：
dp*ppi/160 = px。比如1dp x 320ppi/160 = 2px。


### raw 和 assets 文件夹的作用，二者有何区别
+ 相同点  
都是不会被编译成二进制
+ 不同点  
raw 不可以有子文件夹， assets 则可以。 raw 可以生成 R 文件的 id ，assets 则不可以。  
读取，raw :  
InputStream in = getResource().openRawResource(R.id.filename);  
assets:  
AssetManager am = getAssets();  
InputStream in = .open("filename");

### 内存泄露的情景以及处理办法
+ 静态变量
+ 单例
+ 无限循环的属性动画
+ handler  
handler.removeCallbacksAndMessages(object token);

```java
@Override
protected void onDestroy() {
    super.onDestroy();
    handler.removeCallbacksAndMessages(null);
    // removeCallbacksAndMessages,当参数为null的时候，可以清除掉所有跟当前handler相关的Runnable和Message。
    // 我们在onDestroy中调用次方法也就不会发生内存泄漏了。
}

```

+ 尽量不要让生命周期远远长于 Activity 的对象持有 Activity 的引用
+ 兜底回收内存，把 大内存对象 bitmap 和 drawable 等，在 Activity 的 onDestroy 中，  
主动释放，取消监听器等，让内存泄露的就是一个空壳 Activity
+ 图片按需加载，加载到内存中之前，先取得一个合适的 inSampleSize 值。
+ 使用多进程，对于 WebView 和图库之类，使用一个单独的 tools 进程。
+ 字符串拼接优化
+ 资源重用, 如果有一个对象经常创立而且是短时间使用的话，对象池是一个不错的选择方案。
+ 选用合理的数据格式 使用SparseArray, SparseBooleanArray, and  
LongSparseArray来代替Hashmap
