# Service

service 是Android四大组件之一，需要在androidmanifest文件中定义。

## Service基本用法

1、使用startservice方法，activity和service没有关联，activity销毁后，service还可以继续运行。

service的回调方法执行：onCreate->onStartCommond;多次调用startService方法，onCreate方法只会执行一次，onStartCommond方法会执行多次。

2、使用bindService方法，这个时候activity和service就会绑定，如果需要解绑，需要调用unbindService方法。

service回调方法执行：onCreate->onBind；

自定义一个Binder类的子类，并在onBind方法返回该类的实例，具体任务在改实例中实现。

在activity中创建一个ServiceConnection的匿名类，在里面重写了onServiceConnected()方法和onServiceDisconnected()方法，这两个方法分别会在Activity与Service建立关联和解除关联的时候调用。在onServiceConnected方法里调用bind实例的方法。

bindService()方法接收三个参数，第一个参数就是刚刚构建出的Intent对象，第二个参数是前面创建出的ServiceConnection的实例，第三个参数是一个标志位，这里传入BIND_AUTO_CREATE表示在Activity和Service建立关联后自动创建Service，这会使得MyService中的onCreate()方法得到执行，但onStartCommand()方法不会执行。

## Service和Thread的关系

service和Thread没有任何关系，一般情况service是运行在主线程中的，如果service里有耗时操作，就在service里创建子线程。不在activity里创建子线程执行耗时操作是因为activity对thread的控制没有service好，activity如果被销毁了，就没办法在获取到thread的实例，而service可以不依赖activity，很好的能控制thread，activity只需要和service建立关联和解除关联就可以了。

## 前台Service

如果需要提高service的优先级，可以将自定义的service设置成为前台service。具体是在onCreate方法中，创建一个Notification，创建一个PendingIntent，表示点击通知后的行为，然后使用startForeground（1，notification）方法将通知展示在状态栏。这样service就变成了一个前台service。

## 远程Service

一般的service想要变成远程service，只需要在androidmanifest文件中，service标签里添加android:process=":remote"属性就可以了。但是activity和远程service不能像之前那样进行通信了，因为activity和远程service在不同进程中，需要使用IPC。这里会使用AIDL进行[进程间通信](/IPC/进程间通信.md)。







参考链接：https://blog.csdn.net/guolin_blog/article/details/11952435