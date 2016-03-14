title: Flux 学习笔记
date: 2016-03-04 11:35:35
tags: [android, 架构设计]
categories: android
---
至今还没有见过一种自己满意的 app 架构模式，因此学习 Flux ，多尝试，拖体验各架构的优缺点。
<!--more  -->
>[原文](http://lgvalle.xyz/2015/08/04/flux-architecture/)
>[泡网的翻译](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0816/3311.html)

# Flux 架构关键特点。
## app分层如下
* View : 界面，和 MVP 一样，一般就是 Activity 或者 fragment 。
* Dispatcher ： 处理中心，负责分发 Action 给 Store 的。
* Store ： 记录 app 状态的，接收到 Action ，根据 Action 带过来的 data 和 type 作不同的  
业务逻辑处理，并且发出不同的 change 事件，通知 View 来修改 UI 。 Store 层的操作都是同步的。
* ActionsCreator ： 接收到 View 层的事件，创造 Action，异步数据获取操作，放在这。

## 数据的单向流动，这个是 Flux 的核心，单向的数据流动，容易理解和上手。  
数据流方向是： View -> ActionsCreator -> Dispatcher -> Store -> View
![数据单向流动示意图](https://raw.githubusercontent.com/aheadlcx/myPhotos/master/flux_data_forward.jpg)
## Action ： 这个是通信的基本对象，事件流动必须通过他（或者子类）。


## 一个具体实现
### Action
Action 主要携带信息，包含2个主要属性：
* Type ： 区分标志位，定义了这个 Action 的类型。
* Data ： Action携带的数据，具体实现，看需要，简单点可以定义为 Object，如果考虑到携带多个数据  
也可以设计为 Map等。

```java
public class Action {
    private final String type;
    private final HashMap<String, Object> data;

    Action(String type, HashMap<String, Object> data) {
        this.type = type;
        this.data = data;
    }
}
```

### View
View 其实就是 Activity 或者 fragment ，一般都是通过 事件总线（ EvenBus, otto 之类）来实现  
监听 Store 层的 UI 更新通知。

### ActionsCreator
这里是（异步）获取数据，并且创造 Action 再传递给 Dispatcher 的。如果数据获取是异步的，可以  
再封装一个获取异步数据的类，类似于 MVP 中的 Model。这里的 Model 个人觉得需要思考下这个颗粒度的  
问题，到底是一个界面的数据获取操作，全部封装在一个 Model 里面合适点，还是每个异步获取数据放一个  
Model 。 个人建议按照界面来定义  Model 的颗粒度。

```java
public class ActionsCreator {

    final Dispatcher dispatcher;

    ActionsCreator(Dispatcher dispatcher) {
        this.dispatcher = dispatcher;
    }
    //简单创造的一个 todo
    public void create(String text) {
        dispatcher.dispatch(
                TodoActions.TODO_CREATE,
                TodoActions.KEY_TEXT, text
        );
    }
}
```

### Dispatcher
这就是一个接受到 Action 然后分发到 Store 的过程。  
dispatch 方法，负责分发 Action 给各 Store 。

```java
public class Dispatcher {
    private final Bus bus;
    private static Dispatcher instance;

    public static Dispatcher get(Bus bus) {
        if (instance == null) {
            instance = new Dispatcher(bus);
        }
        return instance;
    }

    Dispatcher(Bus bus) {
        this.bus = bus;
    }

    public void register(final Object cls) {
        bus.register(cls);

    }

    public void unregister(final Object cls) {
        bus.unregister(cls);
    }

    public void emitChange(Store.StoreChangeEvent o) {
        post(o);
    }

    public void dispatch(String type, Object... data) {
        if (isEmpty(type)) {
            throw new IllegalArgumentException("Type must not be empty");
        }

        if (data.length % 2 != 0) {
            throw new IllegalArgumentException("Data must be a valid list of key,value pairs");
        }

        Action.Builder actionBuilder = Action.type(type);
        int i = 0;
        while (i < data.length) {
            String key = (String) data[i++];
            Object value = data[i++];
            actionBuilder.bundle(key, value);
        }
        post(actionBuilder.build());
    }

    private boolean isEmpty(String type) {
        return type == null || type.isEmpty();
    }

    private void post(final Object event) {
        bus.post(event);
    }
}
```

### Store
Store 层负责接受 Dispatcher 发送过来的 Action ，接收 Action 中的 Data 根据 Type 作出相应  
UI 刷新事件。（这事件由 View 订阅）。 View 再从 Store 拿相应的数据刷新 UI 。  
Store 提供 Get 接口，给数据，但是不应该提供 Set 接口，更新数据，要更新数据必须通过 ActionsCreator 。  

![Store](https://raw.githubusercontent.com/aheadlcx/myPhotos/master/flux_store.jpg)


## Flux 优点
### 数据异步
所有的action都是从一个Action Creator触发的：在一处单一的点创建与发起所有用户操作可以大大简化寻找错误的过程。  
忘掉在多个类中寻找某个操作的源头吧 ，所有的事情都是在这里发生的。同时，因为异步调用发生在这之前，  
所有来自于ActionCreator的东西都是同步的。
![data](https://raw.githubusercontent.com/aheadlcx/myPhotos/master/flux_data_synchronous.jpg)

### 不同线程代码，分隔开
* View  和 Store 的代码，都是运行在主线程。
* ActionCreator  ，View 操作 ActionCreator 的都是主线程，异步调用的代码，都封装在 获取数据的 Model 中。  
这方便，项目的分工，加快开发进度。


## Flux 缺点（个人认为）
* 代码量，不少。需要多写很多 Action 和 Store 。
* 每一个 View 操作，都需要经过 ActionCreator ， Dispatcher 和 Store ，调整多，如果需要一眼看出每一个  
View 操作，最后 View监听到的 回调事件，需要较好的命名规范。

## [show me the code](https://github.com/lgvalle/android-flux-todo-app)
