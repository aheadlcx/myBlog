title: 关于安卓中的服务service
date: 2014-12-14 20:54:44
tags: android
---

# 服务的2种启动方式以及区别
service 可以用startService和bindService来启动。
startService开始了服务，开始者就不能直接控制service了。
bindservice方式，可以返回一个自定义binder给开始者，java中非静态内部类保持外部类的引用，这样开始者可以可以直接控制service了。
bindService方式，意思是让activity和service关联在一起，一般是要求在同一个进程中的。

# 服务所在线程和进程
## 主线程中的service
service必须在清单文件中注册，如果没有特别指定，service是处于UI线程和，当然了，进程和UI线程一致。这个可以通过Looper.mainLooper()和Looper.myLooper()，android.os.Process.myPid(),来验证。
## 远程service
在清单文件中注册的时候，加上属性android:process=":remote"即可，验证方式同上，进程ID变了，而且应用程序的包名也变了，多了.remote标识。

# service和activity之间的通信
service和activity之间的通信，必须是由bindService方式启动的。
## 【方式一】，利用广播。

广播效率比较低，一般而言，不是首选。
## 【方式二】，利用binder和结合service的接口内部类。
activity（作为开始者举例），bindService方式启动服务之后，服务会返回一个binder子类，activity实现service的内部类接口，然后通过binder拿到service的实例，设置内部接口实现类为activity，最后，在service中，需要更新UI的时候或者需要告诉activity一些信息的时候，就可以调用内部类的对象的方法了。

## 【方式三】使用messenger通信

messenger在构造方法中可以实现和handle关联，使用messenger通信可以发送message，使用messenger通信发送的message交给与之关联的handle处理。我们可以注意到Message里面有个messenger变量replyTo。我们可以在activity和service中各实现一个messenger和关联的handle。首先，service中onBinder返回的binder可以通过service中的messenger Smessenger.getBinder(),那么activity就可以拿到了service中的Smessenger了。那么service怎么拿到activity中的messenger呢，这个就得通过message中的replyTo变量了。在activity中拿到的Service引用，发送一个message，这个message的replyTo为activity的messenger，这样这个message就带有了activity中的messenger了，这个message是service中的Smessenger发送的，也就是交给service的handle处理了。
文字比较累赘，词不达意的地方，敬请谅解。代码如下。


这里是activity
#java
        public class MessengerAct extends Activity {

        private Messenger sMessenger = null;

        private final Messenger aMessenger = new Messenger(new ActHandler());

        private class ActHandler extends Handler {
            @Override
            public void handleMessage(Message msg) {
                // TODO Auto-generated method stub
                super.handleMessage(msg);
            }
        }

        @Override
        protected void onCreate(Bundle savedInstanceState) {
            // TODO Auto-generated method stub
            super.onCreate(savedInstanceState);

            initReceiver();
            initMsg();
        }

        private void initReceiver() {
            Intent ReI = new Intent(this, MessengerReceiver.class);
            bindService(ReI, new ServiceConnection() {

                @Override
                public void onServiceDisconnected(ComponentName name) {
                    // TODO Auto-generated method stub

                }

                @Override
                public void onServiceConnected(ComponentName name, IBinder service) {
                    // 拿到服务器的Messenger实例
                    sMessenger = new Messenger(service);

                }
            }, Context.BIND_AUTO_CREATE);
        }

        private void initMsg() {
            Bundle actBundle = new Bundle();
            actBundle.putString("test", "从act发来的消息");
            Message msg = Message.obtain();
            msg.setData(actBundle);
            msg.replyTo = aMessenger;
            try {
                sMessenger.send(msg);
            } catch (RemoteException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }

        }
        @Override
        protected void onStop() {
            // TODO Auto-generated method stub
            super.onStop();
        }
    }
    此为service

    public class MessengerReceiver extends Service{

        public static String TAG = MessengerReceiver.class.getSimpleName();

        private Messenger aMessenger = null;

        private final Messenger sMessenger = new Messenger(new RecHandler());

        private class RecHandler extends Handler{

            @Override
            public void handleMessage(Message msg) {
                // TODO Auto-generated method stub
                super.handleMessage(msg);
                //拿到activity的Messenger实例
                aMessenger = msg.replyTo;
                Message msg1 = Message.obtain();
                try {
                    aMessenger.send(msg1);
                } catch (RemoteException e) {
                    // TODO Auto-generated catch block
                    e.printStackTrace();
                }
            }
        }
        @Override
        public IBinder onBind(Intent intent) {
            // TODO Auto-generated method stub
            return sMessenger.getBinder();
        }

        @Override
        public void onCreate() {
            // TODO Auto-generated method stub
            super.onCreate();
            Log.i(TAG, "process id is "+android.os.Process.myPid());

            Notification mNotification = new Notification(R.drawable.ic_launcher, "通知come", System.currentTimeMillis());

            Intent n= new Intent(this,SecondAct.class);
            PendingIntent mPendingIntent = PendingIntent.getActivity(this, 0, n, 0);
            mNotification.setLatestEventInfo(this, "title", " content text", mPendingIntent);
            startForeground(1, mNotification);
        }



    }


##【方式四】AIDL，一般用于实现多个应用程序共享同一个Service的功能。

首先需要建立aidl文件，基本和java语言类似。
    package com.example.mycode.aidl;
    interface MyAIDLService{
        int incre(int a, int b);

        int decre(int a, int b);

    }
APP A中的AService aservice，在清单文件中注册个action，因为在APP B中是没有APP中的service这个类的，必须隐式调用。
#java
    <service android:name=".receiver.MessengerReceiver"
     android:process=":remote" >
     <intent-filter>
    <action android:name="com.example.mycode.test.aidlservice"/>
    </intent-filter>
     </service>
在A中的 AService的onbind方法中，返回
    MyAIDLService.Stub aidlBinder = new Stub() {

        @Override
        public int incre(int a, int b) throws RemoteException {
            // TODO Auto-generated method stub
            return a + b;
        }

        @Override
        public int decre(int a, int b) throws RemoteException {
            // TODO Auto-generated method stub
            return a - b;
        }
    };

    其中stub，就是binder的抽象子类。
    在APP B中，把APP中的AIDL文件copy过来，必须把原来的包路径也copy过来。这样在APP B中就可以调用service中的函数了。
    private ServiceConnection connection = new  ServiceConnection() {

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }

        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
//          myBinder = (MyBinder) service;
            mMyAIDLService = MyAIDLService.Stub.asInterface(service);
            try {
                int result = mMyAIDLService.incre(2, 5);
                Log.i(TAG, "result"+result+"");
            } catch (Exception e) {
                // TODO: handle exception
            }
        }
    };

    这样就实现了多APP操作一个service了。就是比较麻烦点。




引用[第一行代码作者guolin](http://blog.csdn.net/guolin_blog/article/details/11952435)
