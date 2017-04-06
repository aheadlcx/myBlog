title: Android 多进程使用注意点
date: 2017-03-20 11:38:56
tags:
---
记录一下 Android 上面多进程使用中，注意点以及对应方法
<!--more  -->

## 多进程常见需求点
* SharedPreference ，多进程不靠谱。
* 业务中往往需要，有一个保存在内存中的全局变量，各进程中获取到的变量值，需要保持一致。
* 业务代码，有需求需要知道，当前运行在哪个进程。
## 对应的方法
### 解决 SharedPreference 在多进程中的问题
每个进程在内存中，都保留一份 key value 的 map 容器。

put 的时候


+ 往多个进程内存中，把 map 容器，都设置一下值。
+ 通过 contentProvider ，设置一下 SharedPreference ，保存值。

get 的时候


+ 先把自己进程的内存 map 容器，查找一下。
+ 如果进程内存中没有找到，就通过 contentProvider 查找一下。


### 跨进程的全局变量，保持一致。
利用 binder ，进程 A 每次设置变量值的时候，都把另外一个进程的值也设置了。

### 判断在什么进程。
通过对比进程名字。每个进程名字，我们是可以预设的，然后在运行时获取一下进程名字。  
运行时获取进程名字代码如下：  

```java
private String getCurrentProcessName() {
        int pid = android.os.Process.myPid();
        ActivityManager am = (ActivityManager) getSystemService(Context.ACTIVITY_SERVICE);
        List<ActivityManager.RunningAppProcessInfo> infos = am.getRunningAppProcesses();
        if (infos != null && infos.size() > 0) {
            for (ActivityManager.RunningAppProcessInfo appProcess : infos) {
                if (appProcess.pid == pid) {
                    return appProcess.processName;
                }
            }
        }
        return "";
    }
```
