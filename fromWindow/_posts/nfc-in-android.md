title: NFC On Android
date: 2015-01-28 15:16:14
tags:[原创，NFC]
---
#NFC在android平台支持的范围
在api 9开始支持nfc，不过不完善，api 10支持比较完善了，在2015年，这些估计都不是事儿了吧。还要兼容2.2,2.3么。。恐怕找不到2.2和2.3还支持NFC的机器了吧？

#android平台支持的nfc标签类型
nfc标签，目前没有一个统一的标准，欧洲、日本和美国，都在用比较主流的标注（都在用自家的）。至于在中国，估计就看城市的leader了，起码2015年还没看到官方说主流使用哪一种。如果是自家用用，建议使用M1类型。目前android（api 21）支持IsoDep、NfcA、NfcB、NfcF、NfcV、Ndef、NdefFormatable、MifareClassic和MifareUltralight类型。

#nfc标签和android设备的通信
他们之间的通信，在底层由谷歌实现了的，JNI层和native层的代码，针对各个芯片，没有很好的统一，可读性很差（本人没有去读，【深入理解Android：WiFi、NFC和GPS卷】书中原话）。在应用层，android系统都是讲读取到的信息，组装成了一个intent的。接下来我们分析这个intent的分发系统

#nfc权限
如果你的app需要使用nfc，必须在清单文件中注册权限。
	<uses-permission android:name="android.permission.NFC" />

#nfc标签的分发系统
一共包括前台分发系统和标签分发系统。
##前台分发系统
如果当前activity，注册了nfc某种类型，如果检测到这种类型nfc标签了，则直接交由这个该activity来处理（这个也是有例外的，这个请查看注1）；
注册前台分发系统代码如下：
	PendingIntent pendingIntent = PendingIntent.getActivity(
    	this, 0, new Intent(this, getClass()).addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP), 0);

    IntentFilter ndef = new IntentFilter(NfcAdapter.ACTION_TECH_DISCOVERED);//识别TECH级别的分发，这个一共有NDEF、TECH和TAG，具体区别查看下面的标签分发系统
    try {
        ndef.addDataType("*/*");    /* Handles all MIME based dispatches.
                                       You should specify only the ones that you need. */
    }
    catch (MalformedMimeTypeException e) {
        throw new RuntimeException("fail", e);
    }
   intentFiltersArray = new IntentFilter[] {ndef, };

   techListsArray = new String[][] { new String[] { MifareClassic.class.getName() } };//识别MifareClassic类型。

	mNfcAdapter = NfcAdapter.getDefaultAdapter(mActivity);

   mAdapter.enableForegroundDispatch(this, pendingIntent, intentFiltersArray, techListsArray);//该方法必须在OnResume和OnPause方法之间执行


   注销前台分发系统如下
   mNfcAdapter.disableForegroundDispatch(mActivity);//该方法必须在OnResume和OnPause方法之间执行
















注解：
1、如果一个nfc tag有ndef数据，而且这个ndef数据中包含了aap数据（就是写了app的包名），则会直接进入到这app数据指定的app中。
