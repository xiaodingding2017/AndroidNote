
webview常见的一些坑

1. webview 在android api16以及之前版本的安全漏洞，该漏洞是因为程序没有正确的限制webview.addjavascriptinterface方法，让远程攻击者可以使用java的反射机制利用该漏洞执行任意的java对象方法。

2. webview动态添加到其他布局的时候，在activity销毁的生命周期时，需要主动调用webview.removeallviews和webview的ondestory方法释放内存，否则会导致内存泄漏。

3. jsbridge ，js桥可以允许远程网页端与android的native端进行通信，通俗的说就是使用js桥可以在android代码中调用网页的js方法，也可以让js调用原生的代码

4. webview.onpagefinished方法，该方法并不靠谱，按照api上面的说法，在web页面完全加载完成的时候会回调该方法，但在实际应用过程中，该方法在跳转url的时候会被多次调用，更加靠谱的方法是使用onprogresschange方法代替该方法的功能，当newProgress为100的时候，即是页面加载完成。

5. 后台耗电问题，webview加载网页的时候，会自动创建线程，如果如果使用不当，这些线程会永远在后台运行，导致你的应用耗电量居高不下，这个问题的解决方式是在activity的ondetory方法中销毁webview。

6. webview硬件加速导致渲染问题，比如加载的时候会有闪屏现象，解决方式就是暂时关闭硬件加速。

7. webview导致内存溢出的原因，主要是因为内部类持有外部类的引用导致外部类无法释放的问题。


参考链接：
WebView全面解析
https://blog.csdn.net/vicwudi/article/details/81990467?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task
