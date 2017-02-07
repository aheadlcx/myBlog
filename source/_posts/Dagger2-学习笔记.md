title: Dagger2 学习笔记
date: 2016-03-23 14:25:56
tags: [android]
categories: [android]
---
Dagger2 学习笔记。
<!--more  -->
# Dagger2 中关键词
1. @Inject
我们一般会在类的成员变量或者构造函数中加上这个注解。
* 在成员变量上，带有这个注解，代表这个成员变量，需要依赖注入。

```java
@Inject
UserPresent mUserPresent;
```

* 在构造函数上，带有这个注解，代表这个类在依赖注入时，调用这个方法来生成实例的。

```java
public class UserPresent {
    AppModel mAppModel;
    ActivityModel mActivityModel;

    @Inject
    public UserPresent(AppModel appModel, ActivityModel activityModel) {
        this.mAppModel = appModel;
        this.mActivityModel = activityModel;
    }

    public AppModel getAppModel() {
        return mAppModel;
    }

    public ActivityModel getActivityModel() {
        return mActivityModel;
    }
}

```

2. @Component
这个是 注入器，Dagger2 编译器库通过他，找到带有 @Inject 注解的成员变量，以及这个成员变量的  
带有 @Inject 的构造函数，这样，就可以联系在一起了。

3. @Module 以及 @Provode
这个主要用于三方库，这个是告诉 component 可以来我这里找生成实例的方法。带有 @Module 注解的类，  
其实都是我们封装的，用于解决获取三方类库的类。在这个类中，带有 @Provide 的方法，就是获取这个对象  
的方法，注入器就会从这个方法获取实例。

4. @Scope
***未明白***

5. @Qualifier
***未明白***

6. @name
***未明白***


# 注入（赋值）触发的时间点
1. 调用 component inject （UserFragment），才会为 UserFragment 中相关的成员变量  
（标注了 InJect 注解）赋值。
2. 也只有调用了 component inject ，才会生成 UserFragment 中相关成员变量的实例。这个实例的创建  
是在，component 的实现类中的，Provider 类中提供的，一般会有一个 get 方法，get 的时候才 new 出来。
3. Dagger2 是编译器注解，写好相关的 component 和 Module 等之后，需要 Make Project 或者  
rebuild 之后才会生成相关的 DaggerXXComponent 代码。
4.

# 自动生成的代码，实现原理
1. @Singleton
用了这个注解的话，那个类，就会有一个 **相关** 的类变成一个枚举。
2. @PerActivity
这个注解，据说是限定跟随 Activity 生命周期的，可是到底怎么实现的，带考究。

# 一些观察到的细节
1. component 类（一般是接口，或者抽象函数）中， inject 的函数，是注入的意思。

```java
@Component(modules = {UserModule.class, ActivityModule.class}, dependencies = ApplicationComponent
        .class)
public interface UserComponent extends ActivityComponent {
    void inject(UserActivity userActivity);
}
```

2. component 类中，还可以有其他函数，例如，返回一个对象（类名暂时称呼为， ActivityModel ）。  
在这个 component 实现类中，会有一个 ActivityModel_Factory 类，这个类会调用 ActivityModel  
中带有注解 @Inject 的构造函数，来构造一个对象返回。

```java
@Component
public interface ActivityComponent {
    ActivityModel getActivityModel();
}
```

```java
private void initialize(final Builder builder) {  
  this.provideUserProvider = UserModule_ProvideUserFactory.create(builder.userModule);
  this.getAppModelProvider = new Factory<AppModel>() {
    @Override public AppModel get() {
      AppModel provided = builder.applicationComponent.getAppModel();
      if (provided == null) {
        throw new NullPointerException("Cannot return null from a non-@Nullable component method");
      }
      return provided;
    }
  };
  this.userPresentProvider = UserPresent_Factory.create(getAppModelProvider, ActivityModel_Factory.create());
  this.userActivityMembersInjector = UserActivity_MembersInjector.create((MembersInjector) MembersInjectors.noOp(), provideUserProvider, userPresentProvider);
}
```

3. 如果 component 类没有提供这个对象的获取方法，而是由 Module 层来提供的话。如下代码  
这个 ActivityModel 获取的实现，有点变化

```java
@Module
public class ActivityModule {

    @Provides
    ActivityModel provideActivityModole(){
        return new ActivityModel().setActivityName("name from ActivityModule");
    }
}
```


```java
private void initialize(final Builder builder) {  
  this.provideUserProvider = UserModule_ProvideUserFactory.create(builder.userModule);
  this.getAppModelProvider = new Factory<AppModel>() {
    @Override public AppModel get() {
      AppModel provided = builder.applicationComponent.getAppModel();
      if (provided == null) {
        throw new NullPointerException("Cannot return null from a non-@Nullable component method");
      }
      return provided;
    }
  };
  this.provideActivityModoleProvider = ActivityModule_ProvideActivityModoleFactory.create(builder.activityModule);
  this.userPresentProvider = UserPresent_Factory.create(getAppModelProvider, provideActivityModoleProvider);
  this.userActivityMembersInjector = UserActivity_MembersInjector.create((MembersInjector) MembersInjectors.noOp(), provideUserProvider, userPresentProvider);
}
```

4. 一个依赖注入的类，如果构造函数带有参数
先见例子代码。我们都知道带有注解 @Inject 的构造函数，那就是我们在注入器注入的时候会调用的函数了。  
可是，现在我们这个注入构造函数带有参数的，这个参数，注入器怎么给到的呢？传 null ？?  
当然不会，其实这个应该才是 Dagger 的魅力。这个也是通过 注入器 component 来寻找提供的。  
要么是自己的带返回值的函数提供，要么是 Module 来提供。  
这里有个 **注意点** ，可能这个 component  
本身自己没有提供这个函数，甚至 他的 Module 也没有提供，但是他可以继承，另外一个 component 来提供，  
或者 dependencies 来提供。这里2种方式，都是可以提供的，在源码中实现也有差别，但是本人目前还看不出这2种  
方式之间的利弊差别。只是如果由 dependencies 来提供的话，需要在初始化 component 的时候，提供  
dependencies 中指向的 component 的实现类，传递进去。

```java
public class UserPresent {
    AppModel mAppModel;
    ActivityModel mActivityModel;

    @Inject
    public UserPresent(AppModel appModel, ActivityModel activityModel) {
        this.mAppModel = appModel;
        this.mActivityModel = activityModel;
    }

    public AppModel getAppModel() {
        return mAppModel;
    }

    public ActivityModel getActivityModel() {
        return mActivityModel;
    }
}
```


```java
@Component(modules = {UserModule.class}, dependencies = ApplicationComponent
        .class)
public interface UserComponent extends ActivityComponent {
    void inject(UserActivity userActivity);
}
```

# Component
## Component 依赖
> Component dependencies

> While subcomponents are the simplest way to compose subgraphs of bindings, subcomponents are tightly coupled with the parents; they may use any binding defined by their ancestor component and subcomponents. As an alternative, components can use bindings only from another component interface by declaring a component dependency. When a type is used as a component dependency, each provision method on the dependency is bound as a provider. Note that only the bindings exposed as provision methods are available through component dependencies.

Component 依赖，如果 A 依赖 B ，那么 B 的 provision 方法就可以提供给 A 使用，类似  
于 Module 中的 provider 方法

# 未明事项
在 Component 中有一个方法，返回了一个抽象类，甚至接口，但是这个 Component 上有 Module ，  
这个 Module 同时，还用注解 Provide 标记了一个方法，返回了一个实现类。这个时候采用了是实现类。
