
OkHttp 3.14.x版本源码分析

OkHttp的优点
OkHttp作为当前Android端最火热的网络请求框架，必然有很多的优点。
支持SPDY协议（已经升级为HTTP2.0）
支持HTTP/2 协议，允许连接到同一个主机地址的所有请求共享Socket。这必然会提高请求效率。
在HTTP/2协议不可用的情况下，通过连接池减少请求的延迟。
GZip透明压缩减少传输的数据包大小。
响应缓存，避免同一个重复的网络请求。
基于OKIO操作存储，提供性能。

作者：sososeen09
链接：https://www.jianshu.com/p/c70d0ce5400c
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。


参考链接：
https://blog.csdn.net/qq_41979349/article/details/102823696
https://blog.csdn.net/qq_41979349/article/details/103151302

https://www.jianshu.com/p/82f74db14a18
