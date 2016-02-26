title: android studio 小技巧
date: 2016-01-28 13:36:59
tags: [as小技巧, 笔记]
categories: android
---
收集一些常用的as小技巧
<!--more  -->

# Productivity Guide

1. Complete Statement
Use ⇧⌘⏎ to complete a current statement such as if, do-while, try-catch, return (or a method call) into a syntactically correct construct (e.g. add curly braces).  
如果你输入if(str!=null) 这个时候按下 shift cmd + enter 就会给你补全合适的结构体

2. Generate code
cmd + N 可以提示展示自动化插入的code。配合 prettify 插件，解决findViewById，一级棒！

3. 重写和实现方法
ctrl + o 是重写方法。  
ctrl + | 是实现接口。（试验，没通过，可能快捷键被屏蔽了。）这个功能，更习惯，alt + enter。

4. Surround with
选中代码块，  alt + cmd + T 可以选择插入   if  try-catch 等语句。

5. Select Popup
alt + F1 就是选择用什么方式打开当前选中的文件

 6. find usage
 alt + F7.
 show usage
 alt + cmd + F7

7. extend selection
扩大选择范围  
alt + 向上箭头

8. 重构代码块
选中代码块 alt + cmd + m 就会把当前代码块抽离出来，成为一个独立的方法体

9.  增加返回值
alt + cmd + v
这个快捷键，可以让我们先写函数，然后自动增加返回值。

10. 利用构造函数形参，快速增加成员变量并赋值
alt + enter 选择 bind constructor parameters to fields

11. Log
光标放在类中（不要放在方法体内）
logt  会弹出一个popup，自动给出一个以当前类名的TAG  
logi loge logr  logm 等等会有一系列友好的代码模板

12. 把编辑框全屏。
Hide All Panels

ctrl+shift+f12

关闭或者恢复其他窗口。在编写代码的时候非常方便的全屏编辑框。
这个，其实也可以把其他弹出的框隐藏掉来实现。很多其他弹出框都可以用对应的快捷键来隐藏，find框之类的。

13. 调试时，Evaluate调试
alt + f8，可以自己编写临时代码查看值。

14. 调试时，查看某个值来源
Find Actions(ctrl+shift+a)输入”Analyze Data Flow to Here”，可以查看某个变量某个参数其值是如何一路赋值过来的。
对于分析代码非常有用。

15. 如果想2个方法块代码，上下移动
可以先选中其中一个方法的全部或部分代码，shift + cmd + up / down  就可以整体移动了
# 收集的细节

1. Language Injection
用过JSON字符串吗？说不定你已经把JSON字符串当做某个GSON解串器的测试固件来用了，也很清楚要管理所有的反斜杠有多麻烦。

所幸IntelliJ有Language Injection的功能，可以实现在JSON编辑器里编辑JSON碎片，再由IntelliJ将碎片以转义字符串的形式放在代码中。

Inject Language/Reference是一个intention action，用⌥+Return或⌘+⇧+A，搜索它，就可以启动了。
