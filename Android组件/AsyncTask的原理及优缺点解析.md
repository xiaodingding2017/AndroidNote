
AsyncTask理解：
AsyncTask是Handler与线程池的封装。
网络请求等耗时操作在线程池中完成，通过handler发送给主线程完成UI更新。
使用线程池的主要原因是避免不必要的创建及销毁线程的开销。
AsyncTask的不足：
内存泄漏问题
AsyncTask对象必须在主线程中创建，这与内部sHandler有关，下面会解释。
AsyncTask对象的execute方法必须在主线程中调用。
一个AsyncTask对象只能调用一次execute方法

参考链接：
https://www.jianshu.com/p/7ad6b5a94e94

https://www.jianshu.com/p/aeabb33acb89
