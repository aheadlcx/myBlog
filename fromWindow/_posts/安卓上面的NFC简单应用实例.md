title: 安卓上面的NFC简单应用实例
date: 2015-02-02 16:52:30
tags:
---

#背景
	需要在安装系统上，实现对NFC标签的读取数据功能。


#NFC简介
	NFC，是Near Field Communication（近场通信）的简称。在api 9开始支持nfc，不过不完善，api 10支持比较完善了，建议目标app最小API 为10；


#安卓手机支持的NFC标签类型
	支持IsoDep、NfcA、NfcB、NfcF、NfcV、Ndef、NdefFormatable、MifareClassic和MifareUltralight类型。
如果是是仅仅实现读取数据，建议取MifareClassic类型的NFC 标签。


#NFC标签和android设备的通信
他们之间的通信，在底层由谷歌实现了的，JNI层和native层的代码，针对各个芯片，是有不同的类实现的，没有很好的统一，可读性很差（本人没有去读，【深入理解Android：WiFi、NFC和GPS卷】书中原话）。在应用层，android系统都是讲读取到的信息，组装成了一个intent的。我们可能需要在activity的onCreate中对接受到的intent实现读取，或者在activity的onNewIntent方法中实现读取信息。接下来我们分析这个intent的分发系统


#NFC权限
如果你的app需要使用nfc，必须在清单文件中注册权限。

		<uses-permission android:name="android.permission.NFC" />


#NFC的分发系统
	android为这个包含nfc信息的Intent的分发，创造了2个分发系统。
a、标签分发系统(个人意译名字)；
b、前台分发系统（the Foreground Dispatch System）

##a、标签分发系统
	标签分发系统，是为了选择这个带有nfc信息的intent交给谁来处理，他可识别的级别是activity级别的。

	一共包括ACTION_NDEF_DISCOVERED、ACTION_TECH_DISCOVERED和ACTION_TAG_DISCOVERED。intent被划分为这3种类型的intent。
这3种intent的优先级从高到低。如果 一个带有nfc信息的intent生成、并且多个app注册了nfc或者同一个app里面的多个activity注册了，他会优先寻找注册了ACTION_NDEF_DISCOVERED类型的activity，如果没有注册了ACTION_NDEF_DISCOVERED类型的activity，则会寻找注册了ACTION_TECH_DISCOVERED类型的activity。如此类推。

	以下代码示例为，给activity注册响应ACTION_TECH_DISCOVERED类型的intent。
	 <intent-filter >
                <action android:name="android.nfc.action.TECH_DISCOVERED" />
     </intent-filter>

      <meta-data
                android:name="android.nfc.action.TECH_DISCOVERED"
                android:resource="@xml/nfc_tech_filter" />



        nfc_tech_filter.xml文件如下：
        <?xml version="1.0" encoding="utf-8"?>  
	<resources xmlns:xliff="urn:oasis:names:tc:xliff:document:1.2">  
	    <tech-list>  
	        <tech>android.nfc.tech.IsoDep</tech>  
	    </tech-list>  
	    <tech-list>  
	        <tech>android.nfc.tech.NfcA</tech>  
	    </tech-list>  
	    <tech-list>  
	        <tech>android.nfc.tech.NfcB</tech>  
	    </tech-list>  
	    <tech-list>  
	        <tech>android.nfc.tech.NfcF</tech>  
	    </tech-list>  
	    <tech-list>  
	        <tech>android.nfc.tech.NfcV</tech>  
	    </tech-list>  
	    <tech-list>  
	        <tech>android.nfc.tech.Ndef</tech>  
	    </tech-list>  
	    <tech-list>  
	        <tech>android.nfc.tech.NdefFormatable</tech>  
	    </tech-list>  
	    <tech-list>  
	        <tech>android.nfc.tech.MifareClassic</tech>  
	    </tech-list>  
	    <tech-list>  
	        <tech>android.nfc.tech.MifareUltralight</tech>  
	    </tech-list>  
	    <tech-list>  
	        <tech>android.nfc.tech.NfcA</tech>  
	        <tech>android.nfc.tech.MifareClassic</tech>  
	        <tech>android.nfc.tech.NdefFormatable</tech>  
	    </tech-list>  
	</resources>  


	上述xml文件，意思是，这个activity可识别的nfc标签技术类型。一个NFC信息的intent，我们可以得到他的Tag对象

	Tag tagFromIntent = intent.getParcelableExtra(NfcAdapter.EXTRA_TAG);
	通过这个Tag对象，我们可以 通过tagFromIntent.getTechList()得知这个刷进来的NFC标签，支持的技术类型有哪些。
	tagFromIntent.getTechList()，返回值是String[],他是IsoDep、NfcA、NfcB、NfcF、NfcV、Ndef、NdefFormatable、MifareClassic和MifareUltralight类的全路径名中的一个或者几个或者全部。
	上述我们为activity注册用的nfc_tech_filter.xml文件，他包含了一系列的tech-list，这其中必须有一个tech-list，是这个NFC标签支持的技术类型的子集，这个activity才可以识别到这个nfc。

##前台分发系统（the Foreground Dispatch System）

	前台分发系统是为了解决这么一个情景的。如果一个手机有多个app注册了响应所有的nfc信息，app A已经在前台显示了，activity A1目前在栈顶并且希望优先响应nfc标签。那么这个时候如果有个nfc刷卡了，还是交给标签分发系统，去派发这个intent的话，就显得有点不妥了。
	前台分发系统，就是解决了这个问题，可以让activity A1优先响应。

	注册和取消注册前台分发系统，必须分别在onResume和onPause之间进行，并且不可以调换顺序（需要在主线程进行这个操作）

###注册前台分发系统
	下面实例代码为，注册识别ACTION_TECH_DISCOVERED类型的nfc标签，支持MifareClassic技术类型的的nfc标签。
	PendingIntent mPendingIntent = null;
		IntentFilter[] mFilters = null;
		String[][] mTechLists = null;
		NfcAdapter mNfcAdapter = null;
		mNfcAdapter = NfcAdapter.getDefaultAdapter(mActivity);
		mPendingIntent = PendingIntent.getActivity(mActivity, 0, new Intent(
				mActivity, mActivity.getClass())
				.addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP), 0);

		IntentFilter ndef = new IntentFilter(NfcAdapter.ACTION_TECH_DISCOVERED);

		mFilters = new IntentFilter[] { ndef };

		// Setup a tech list for all NfcF tags
		mTechLists = new String[][] { new String[] { MifareClassic.class
				.getName() } };

		if (mNfcAdapter != null) {
			if (mNfcAdapter.isEnabled()) {
				mNfcAdapter.enableForegroundDispatch(mActivity, mPendingIntent,
						mFilters, mTechLists);
			}
		}

###取消注册前台分发系统

	mNfcAdapter = NfcAdapter.getDefaultAdapter(mActivity);
		if (mNfcAdapter != null) {
			if (mNfcAdapter.isEnabled()) {
				mNfcAdapter.disableForegroundDispatch(mActivity);
			}
		}


#MifareClassic类型的NFC标签的数据结构
	M1卡就是说容量是1K的卡，M4的卡就是容量4K的卡，市场上M1卡，略多点。
M1卡一共有16个扇区（每个扇区有4个数据块，即总共有64个数据块），每个扇区分别有下标为0、1、2、3的数据块。第3个数据块为秘钥块。读取每个扇区里面的数据，
都必须先验证秘钥，验证通过了才可以读取其他数据块。秘钥块的数据结构为 key A 控制子 key B。怎么重新写秘钥，暂时没试过。


#nfc信息的读取
	每刷一次卡，我们都是希望在当前界面刷，所以一般都是在onNewIntent方法中处理相关的读取操作。
	以下实例代码为读取MifareClassic类型的NFC标签

	Tag tagFromIntent = intent.getParcelableExtra(NfcAdapter.EXTRA_TAG);
	MifareClassic mfc = MifareClassic.get(tagFromIntent);

	NFCInfo nfcInfo = new NFCInfo();//人为封装的一个实体类，保存nfc相关信息
		nfcInfo.erroCode = -1;
		nfcInfo.error = "读取失败";

	try {
			String metaInfo = "";
			// Enable I/O operations to the tag from this TagTechnology object.
			mfc.connect();
			int type = mfc.getType();// 获取TAG的类型
			int sectorCount = mfc.getSectorCount();// 获取TAG中包含的扇区数
			String typeS = "";
			switch (type) {
			case MifareClassic.TYPE_CLASSIC:
				typeS = "TYPE_CLASSIC";
				break;
			case MifareClassic.TYPE_PLUS:
				typeS = "TYPE_PLUS";
				break;
			case MifareClassic.TYPE_PRO:
				typeS = "TYPE_PRO";
				break;
			case MifareClassic.TYPE_UNKNOWN:
				typeS = "TYPE_UNKNOWN";
				break;
			}

			nfcInfo.cardType = typeS;
			nfcInfo.cardInfo = String.format("%d个扇区,%d个块,%dB存储空间",
					mfc.getSectorCount(), mfc.getBlockCount(), mfc.getSize());

			metaInfo += "卡片类型：" + typeS + "\n共" + sectorCount + "个扇区\n共"
					+ mfc.getBlockCount() + "个块\n存储空间: " + mfc.getSize()
					+ "B\n";

			if (0 < sectorCount && sectorCount >= 1) {//这里意思是，读取第一个扇区
				int sectionIndex = 0;
				auth = mfc.authenticateSectorWithKeyA(sectionIndex,
						MifareClassic.KEY_DEFAULT);//采用默认的key

				int bCount;
				int bIndex;
				if (auth) {
					bCount = mfc.getBlockCountInSector(sectionIndex);
					bIndex = mfc.sectorToBlock(sectionIndex); // 一个块的开始地址

					 for (int i = 0; i < bCount; i++) {
					 byte[] data = mfc.readBlock(bIndex);//@return 16 byte block
					//byte[] data = mfc.readBlock(2); //这是读取第二个数据块

					//if (!isEmptyData(data)) {
						String hexString = toHexString(data);// 把16 byte转换为hexString ，【代码见注1】
						byte[] temp = toByteArray(hexString);// 再把hexString转换为所需的byte，【代码见注2】

						String strData = new String(temp);// 转换为String
						// String strData = new String(data);
						Log.i("data-->", strData);
						nfcInfo.data += strData;
						nfcInfo.erroCode = 0;
					//}
					bIndex++;
					Log.i("data-->", "data-->" + bIndex);
					 }
				} else {
					nfcInfo.error = "认证失败";
					nfcInfo.erroCode = -1;
				}
			}
			result = auth + "" + metaInfo;
		} catch (Exception e) {
			e.printStackTrace();
			nfcInfo.erroCode = -1;
		} finally {
			try {
				mfc.close();
			} catch (Exception e2) {
				e2.printStackTrace();
			}
		}



注释
注1：
	/**
	 * Converts the byte array to HEX string. 字节 数组转字符串
	 * 
	 * @param buffer
	 *            the buffer.
	 * @return the HEX string.
	 */
	public static String toHexString(byte[] buffer) {

		String bufferString = "";
		if (buffer != null) {

			for (int i = 0; i < buffer.length; i++) {

				String hexChar = Integer.toHexString(buffer[i] & 0xFF);
				if (hexChar.length() == 1) {
					hexChar = "0" + hexChar;
				}

				bufferString += hexChar.toUpperCase() + " ";
			}
		}
		return bufferString;
	}

注2：
	/**
	 * Converts the HEX string to byte array. 字符串转字节数组
	 * 
	 * @param hexString
	 *            the HEX string.
	 * @return the byte array.
	 */
	public static byte[] toByteArray(String hexString) {

		int hexStringLength = hexString.length();
		byte[] byteArray = null;
		int count = 0;
		char c;
		int i;

		// Count number of hex characters
		for (i = 0; i < hexStringLength; i++) {

			c = hexString.charAt(i);
			if (c >= '0' && c <= '9' || c >= 'A' && c <= 'F' || c >= 'a'
					&& c <= 'f') {
				count++;
			}
		}

		byteArray = new byte[(count + 1) / 2];
		boolean first = true;
		int len = 0;
		int value;
		for (i = 0; i < hexStringLength; i++) {

			c = hexString.charAt(i);
			if (c >= '0' && c <= '9') {
				value = c - '0';
			} else if (c >= 'A' && c <= 'F') {
				value = c - 'A' + 10;
			} else if (c >= 'a' && c <= 'f') {
				value = c - 'a' + 10;
			} else {
				value = -1;
			}

			if (value >= 0) {

				if (first) {

					byteArray[len] = (byte) (value << 4);

				} else {

					byteArray[len] |= value;
					len++;
				}

				first = !first;
			}
		}

		return byteArray;
	}
