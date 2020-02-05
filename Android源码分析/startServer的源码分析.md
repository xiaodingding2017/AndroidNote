# StartService的源码分析

startService是Activity类的成员函数，activity类继承关系：

Activity -> ContentThemeWrapper -> ContentWrapper -> Content

在ContentWrapper类中实现了startservice函数。该函数通过调用ContentImpl类的startService函数实现。

1、ContentImpl中的startService函数里又调用了ActivityManagerSrvice的startService函数；

2、startService函数中调用startServiceLocked函数，该函数中，通过retrieveServiceLocked来解析service这个Intent，就是解析在AndroidManifest.xml定义的Service标签的intent-filter相关内容，然后将解析结果放在res.record中，然后继续调用bringUpServiceLocked进一步处理

3、bringUpServiceLocked函数，调用startProcessLocked函数创建新的进程；

4、startProcessLocked函数，调用Process.start函数创建了一个新的进程，将表示这个新进程的ProcessRecord保存在mPidSelfLocked列表中；

5、 Process.start函数，新建一个进程，然后导入android.app.ActivityThread这个类，然后执行它的main函数；

6、ActivityThread.main函数，创建一个ActivityThread实例，然后调用ActivityThread.attach函数进一步处理。

7、ActivityThread.attach函数，调用attachApplication函数；

```java
final IActivityManager mgr = ActivityManager.getService();
try {
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
```

8、attachApplication函数，通过调用attachApplicationLocked函数进一步处理；

9、attachApplicationLocked函数，通过进程uid和进程名称将之前保存的ServiceRecord找出来，然后通过realStartServiceLocked函数来进一步处理

10、realStartServiceLocked函数，调用scheduleCreateService函数执行启动服务的操作；

11、ActivityThread类的scheduleCreateService函数，将一个CreateServiceData数据放到消息队列中去，sendMessage方法；

12、ActivityThread类的handleMessage函数，这里要处理的消息是CREATE_SERVICE，它调用ActivityThread类的handleCreateService成员函数进一步处理。

13、 ActivityThread.handleCreateService函数

```java
LoadedApk packageInfo = getPackageInfoNoCheck(
        data.info.applicationInfo, data.compatInfo);
Service service = null;
try {
    java.lang.ClassLoader cl = packageInfo.getClassLoader();
    service = packageInfo.getAppFactory()
            .instantiateService(cl, data.info.name, data.intent);
} catch (Exception e) {
    if (!mInstrumentation.onException(service, e)) {
        throw new RuntimeException(
            "Unable to instantiate service " + data.info.name
            + ": " + e.toString(), e);
    }
}
```

其中的data.info.name就是自定义的服务类shy.luo.ashmem.Server了；



14、还是ActivityThread.handleCreateService函数，

```java
try {
    if (localLOGV) Slog.v(TAG, "Creating service " + data.info.name);

    ContextImpl context = ContextImpl.createAppContext(this, packageInfo);
    context.setOuterContext(service);

    Application app = packageInfo.makeApplication(false, mInstrumentation);
    service.attach(context, this, data.info.name, data.token, app,
            ActivityManager.getService());
    service.onCreate();
    mServices.put(data.token, service);
    try {
        ActivityManager.getService().serviceDoneExecuting(
                data.token, SERVICE_DONE_EXECUTING_ANON, 0, 0);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
} catch (Exception e) {
    if (!mInstrumentation.onException(service, e)) {
        throw new RuntimeException(
            "Unable to create service " + data.info.name
            + ": " + e.toString(), e);
    }
}
```

获取到service实例后，接下来就调用了attach函数和onCreate函数。



Android系统在新进程中启动服务的过程中，有三次进程间通信：

1、从主进程调用到ActivityManagerService进程中，完成新进程的创建；

2、从新进程调用到ActivityManagerService进程中，获取要在新进程启动的服务的相关信息；

3、从ActivityManagerService进程又回到新进程中，最终将服务启动起来。





参考链接：<https://blog.csdn.net/Luoshengyang/article/details/6677029>