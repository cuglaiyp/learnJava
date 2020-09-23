# package java.util.concurrent.atomic包内的类

基本类型：AtomicInteger、AtomicLong等

引用类型：AtomicReference

- 只能保证引用地址的并发安全
- 不能保证内部引用类型的属性的并发安全

内部维护了一个Unsafe类对象，原子操作都是由该类的对象调用本地的CAS方法实现的。

-------

0000 0111