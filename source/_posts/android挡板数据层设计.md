title: android挡板数据层设计
date: 2016-01-27 14:36:10
tags: [android, 挡板数据, app框架设计]
categories: android
---
挡板数据，这一个构思来源于 【APP研发实录】。读后，觉得这玩意，可以解决好多开发过程中的痛点，因此根据自己的想法，实现一下，构思记录如下。
<!--more  -->
# 挡板数据的重要性
调试不依赖服务端等。

# 挡板数据设计几个关注点
1. 挡板数据仅仅适用于开发阶段，给开发人员使用。
2. 提交测试之后，挡板数据不应该呈现出来。
3. 挡板数据相关代码，甚至在测试环境、预生成环境和生成，都不应该出现。

## 挡板层设计雏形
1. 利用 productFlavors 做文章  
其实不同的 productFlavors 会有不同的java代码路径，例如有 productFlavors product 和 dev.  
你的代码文件夹应该有如下java代码路径
* src.main.java.packagename
* src.product.java.packagename
* src.dev.java.packagename

在IDE环境下（指as）,左下角点击 Build Variants 选择不同的build变量，这个build变量是由  
product 和 buidTypes 组成。我们选择不同的build 变量，就会根据 productFlavors 编译对应的java文件路径。  
这样，我们就解决了，**挡板数据仅仅在开发环境下有用**。

2. 利用 productFlavorsCompile 做文章
如上述有2个 productFlavors ，productCompile 会仅仅在 product相关的 Build Variants 下起作用。  
同样的，dev 下也如此。因此，**我们可以仅仅在dev下，编译挡板数据代码**。

3. 一种可以选择的综合方案
为挡板数据新建module mockServer ，根据上述方案，仅仅选择dev下编译此module mockServer ，此 mockServer  
依赖我们 baseLib （目前我设想的是， 所有基类和 modle， 都放在baseLib 中，业务module，再依赖 baseLib）。  
发送服务，可以封装一层send层，每个发服务都封装一个 serviceModle过来。 这个 serviceModle 包含一个字符串字段 targetMockModleApi ，标识  
哪一个 mockModleApi.在业务module的 dev 分支代码中，根据 targetMockModleApi ，通过映射来拿到 mockModleApi ,   
然后 mockModleApi 直接返回 mockModle 。
