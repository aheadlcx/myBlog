title: ' Binder 学习笔记'
date: 2016-04-12 23:34:20
tags:
---

# 备忘要点
1. 客户端远程调用，服务端方法，会挂起当前进程。
那岂不是，客户端就成了卡B了？开子线程去处理，
2. 跨进程之间，不能传递对象。
跨进程之间传递的，都是反序列化的过程，肯定不是同一个对象了。  
配合 RemoteCallbackList<E extends IInterface>
3. 如果 Listern 是在 Binder 线程池中执行的，需要配合 Handler 来切换回 UI 线程。
4. Binder 线程池的概念。客户端调用 服务端的，就会运行在 服务端的 Binder 线程池中。相反，  
服务端，调用客户端的 Listener 就会发生在 客户端的 Binder 线程池中。
5. 服务端进程死了，可以有2种回调。
* 第一种是 onServiceDisconnected 发生在 UI 线程。
* 第二种是 在 BinderDied 方法中，这是发生在 客户端的 Binder 线程池的回调。
6. 安全校验
* 权限校验，自定义 permission 。在 onBind 方法中，检查权限。
* 校验包名。
