title: Clean 架构学习笔记
date: 2016-03-25 11:20:39
tags: [android, 架构设计]
categories: [架构设计]
---

记录学习 Clean 架构设计的思路和实践。
<!--more  -->
>[原文文章1](http://fernandocejas.com/2014/09/03/architecting-android-the-clean-way/)
>[原文文章2](http://fernandocejas.com/2015/07/18/architecting-android-the-evolution/)

# Clean 架构设计的目的
1. 代码干净清爽。
2. 易于测试和维护。
3. 层次区分
![Clean](https://raw.githubusercontent.com/aheadlcx/myPhotos/master/clean_architecture1.png)


# Clean 架构的分层
1. Domain 层
主要是一些接口，描述了一些基本获取数据逻辑，但是都没有具体实现。这层设计为，**Java Lib**，不依赖任何 Android库。  
**适合使用 JUnit 和 Mockito 进行单元测试；**
2. Data 层
可以理解为 Domain 层的实现，这层是具体的获取数据逻辑的实现。异步操作都在这里实现。 这层为 **Android Library**
**使用Robolectric （ 因为依赖于Android SDK中的类 ）进行集成测试和单元测试。**
3. Presentation 层
这层是实际的展示层，是 **Android Application** ，这层包括整个 MVP 栈（当然也可以是MVC MVVM等）。

![CLEAN Android 架构示意图](https://raw.githubusercontent.com/aheadlcx/myPhotos/master/clean_architecture_android.png)

# Clean 数据流
总结一下，触发顺序如下
View -> Present -> Case(一些项目喜欢称之为 Interactor) -> Reposity -> Case -> Present -> View
![Clean 数据流示意图](https://raw.githubusercontent.com/aheadlcx/myPhotos/master/clean_architecture_evolution.png)

# Clean 架构的代价。
分了几层之后，同一个业务场景，有一个 ViewModel ,专门用户展示用的。另外一个是 根据服务器返回的数据而定的  
ServerModel 。这个 Model 之间的转换，略显蛋疼。

# Clean 架构常用相关类
1. Mapper
负责把 ServerModel  转换为 ViewModel.
2. Case
和 Present 交互，负责拿数据给 Present。
3. Reposity
和 Case 交互，Case 从 Reposity 拿数据。
4. DataStoreFactory
可以创建各种 DataStore ，例如 CloudDataStore 和 DiskDataStore 。可以根据 Case 不同需求拿不同的  DataStore。
5. DataStore
一个数据中心，可以分为 CloudDataStore ，DiskDataStore 。分别可以理解为，从服务器拿的数据。  
从本地拿数据。
# 一个案例实践
## View 初始化 Present

```java
@Override
    public void initPresent() {
        super.initPresent();
        mPresent = new CatePresent(getActivity(), this);
        mPresent.setCateCase(new GetCateList(new PostExecutionThread() {
            @Override
            public Scheduler getScheduler() {
                return AndroidSchedulers.mainThread();
            }
        }, new CateDataRepository()));
    }
```

## Present 的实现
Present 里面包含一个 View 的引用，并且包含一个 CateCase , View 中初始化  Present 的时候，传递了  
一个 CateCase 的实现类 GetCateList 进去.  
这个 Present 拿 Case 的回调，这里用了 RxJava 相关的订阅者回调 Subscriber 。

```java
public class CatePresent {
    private static final String TAG = "CatePresent";

    private Context mContext;
    private CateIUi mUi;
    private CateCase mCateCase;

    public void setCateCase(CateCase cateCase) {
        mCateCase = cateCase;
    }

    public CatePresent(Context context, CateIUi ui) {
        mContext = context;
        mUi = ui;
    }

    public void getData() {
        mCateCase.execute(new UserListSubscriber());

    }

    public void onDestory(){
        if (mCateCase != null) {
            mCateCase.unsubscribe();
        }
    }

    private final class UserListSubscriber extends DefaultSubscriber<List<Cate>> {

        @Override
        public void onCompleted() {
            Log.i(TAG, "onCompleted: ");
        }

        @Override
        public void onError(Throwable e) {
            Log.e(TAG, "onError: ",e );
        }

        @Override
        public void onNext(List<Cate> cates) {
            Log.i(TAG, "onNext: ");
            mUi.fillData(cates);
        }
    }
}

```

## Case 以及 实现类
case代码

```java
public abstract class CateCase {
    private Subscription subscription = Subscriptions.empty();
    private PostExecutionThread mPostExecutionThread;

    public CateCase(PostExecutionThread postExecutionThread) {
        mPostExecutionThread = postExecutionThread;
    }

    protected abstract Observable buildUseCaseObservable();

    public void execute(Subscriber UseCaseSubscriber) {
        this.subscription = this.buildUseCaseObservable()
                .subscribeOn(Schedulers.io())
                .observeOn(mPostExecutionThread.getScheduler())
                .subscribe(UseCaseSubscriber);
    }

    /**
     * Unsubscribes from current {@link rx.Subscription}.
     */
    public void unsubscribe() {
        if (!subscription.isUnsubscribed()) {
            subscription.unsubscribe();
        }
    }
}
```

Case 实现类

 ```java
 public class GetCateList extends CateCase {

     private CateRepository mCateRepository;

     public GetCateList(PostExecutionThread postExecutionThread, CateRepository cateRepository) {
         super(postExecutionThread);
         mCateRepository = cateRepository;
     }

     @Override
     protected Observable buildUseCaseObservable() {
         return mCateRepository.cates();
     }
 }
 ```

## Reposity 的实现

 ```java
 public class CateDataRepository implements CateRepository {
     @Override
     public Observable<List<Cate>> cates() {
         CateDataStoreFactory factory = new CateDataStoreFactory();
         CateDataStore cloundDataStore = factory.createCloundDataStore();
         Observable<List<Cate>> map = CateMapper.transform(cloundDataStore.cateEntityList());
         return map;
     }
 }
 ```

# Clean 架构，理解总结
## 优点
1. 可以实现单元测试。单元测试对象项目的稳定性提供保障。
2. 分层了，代码清晰。

## 缺点
1. 层次多了，跳转页多了，一些简单的页面，如果也强制使用 Clean 的话，而且在可见的未来，也不会变的很复杂的情况下，  
这有点过度设计了。
2. 由于被切分成了 ServerModel 和 ViewModel , Model 之间的转换，在 Model比较复杂的情况下，有点蛋疼。

## 个人理解适用场景
比较复杂的界面，而且需要不断迭代。单元测试，可以为复杂提高稳定性。层次清晰，为不断迭代，提高阅读性。


# Show me the code
[Fernando Cejas 的 demo](https://github.com/android10/Android-CleanArchitecture)
[个人 demo 例子](https://github.com/aheadlcx/LearnClean)
