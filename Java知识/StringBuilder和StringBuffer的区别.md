StringBuilder和StringBuffer的区别


StringBuilder的append()方法调用的父类AbstractStringBuilder的append()方法

public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull();
    int len = str.length();
    ensureCapacityInternal(count + len);
    str.getChars(0, len, value, count);
    count += len;
    return this;
}
count += len不是一个原子操作。假设这个时候count值为10，len值为1，两个线程同时执行到了第七行，
拿到的count值都是10，执行完加法运算后将结果赋值给count，所以两个线程执行完后count值为11，而不是12。
这就是为什么测试代码输出的值要比10000小的原因。

那么StringBuffer用什么手段保证线程安全的？
@Override
public synchronized StringBuffer append(String str) {
    toStringCache = null;
    super.append(str);
    return this;
}


参考链接：
https://blog.csdn.net/Farrell_zeng/article/details/100153345
