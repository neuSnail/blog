---
title: JOL查看对象内存结构
date: 2019-04-04 22:26:47
tags: 
- java 
- tools
categories: java
---
jol是一款查看java对象内存结构的工具,第一次使用时着实踩了不少坑。 jol的maven依赖如下：
```xml
        <dependency>
            <groupId>org.openjdk.jol</groupId>
            <artifactId>jol-core</artifactId>
            <version>0.9</version>
        </dependency>
```
测试类代码：
```java
public class ObjectHeadTest {
	private int intValue = 0;
    public static void main(String[] args) {
            ObjectHeadTest object = new ObjectHeadTest();
            //打印hashcode
            System.out.println(object.hashCode());
            //查看字节序
            System.out.println(ByteOrder.nativeOrder());

            //打印当前jvm信息
            System.out.println(VM.current().details());
            String classLayout = ClassLayout.parseInstance(object).toPrintable();
            System.out.println(classLayout);
        }
}
```
上面的代码直接执行得到如下结果：
```bash
2034688500
LITTLE_ENDIAN
# Running 64-bit HotSpot VM.
# Using compressed oop with 3-bit shift.
# Using compressed klass with 0x0000000800000000 base address and 0-bit shift.
# Objects are 8 bytes aligned.
# Field sizes by type: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]
# Array element sizes: 4, 1, 1, 2, 2, 4, 4, 8, 8 [bytes]

jdk_test.ObjectHeadTest object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           01 f4 e1 46 (00000001 11110100 11100001 01000110) (1189213185)
      4     4        (object header)                           79 00 00 00 (01111001 00000000 00000000 00000000) (121)
      8     4        (object header)                           40 70 06 00 (01000000 01110000 00000110 00000000) (421952)
     12     4    int ObjectHeadTest.intValue                   0
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```
可以看到ObjectHeadTest实例的header只有12个字节。而64位的jvm对象的header应该占16个字节128位。造成这种情况的原因是jvm默认开启了指针压缩,在64位vm上会把64位的对象指针压缩成32位(原因和细节这里暂不延伸 有兴趣可以自行搜索下) 。这就导致原本应该占8byte的klass对象指针只占了4byte。使用`-XX:-UseCompressedOops `参数可以关闭指针压缩。开启和关闭指针压缩的头部信息如下两张表所示：

开启指针压缩：

```
-------------------------------------------------------------------------------------------------------------|--------------------|
|                                            Object Header (96 bits)                                           |        State       |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                  Mark Word (64 bits)                           |    Klass Word (32 bits)     |                    |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
| unused:25 | identity_hashcode:31 | cms_free:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Normal       |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
| thread:54 |       epoch:2        | cms_free:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Biased       |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                         ptr_to_lock_record                            | lock:2 |    OOP to metadata object   | Lightweight Locked |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                     ptr_to_heavyweight_monitor                        | lock:2 |    OOP to metadata object   | Heavyweight Locked |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                                                       | lock:2 |    OOP to metadata object   |    Marked for GC   |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|

```

关闭指针压缩
```
|------------------------------------------------------------------------------------------------------------|--------------------|
|                                            Object Header (128 bits)                                        |        State       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                  Mark Word (64 bits)                         |    Klass Word (64 bits)     |                    |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
| unused:25 | identity_hashcode:31 | unused:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Normal       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
| thread:54 |       epoch:2        | unused:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Biased       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                       ptr_to_lock_record:62                         | lock:2 |    OOP to metadata object   | Lightweight Locked |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                     ptr_to_heavyweight_monitor:62                   | lock:2 |    OOP to metadata object   | Heavyweight Locked |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                                                     | lock:2 |    OOP to metadata object   |    Marked for GC   |
|------------------------------------------------------------------------------|-----------------------------|--------------------|

```
> Object Header form thanks for https://gist.github.com/arturmkrtchyan/43d6135e8a15798cc46c

还有一个我现在都不太明白的踩坑点 在上面我调用了一遍object.hashCode()来查看hash码，如果不手动调用一遍instance的hashCode方法 classLayout打印出来的hashcode就是空的
```
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           40 70 06 00 (01000000 01110000 00000110 00000000) (421952)
     12     4    int ObjectHeadTest.intValue                   0
     20     4        (loss due to the next object alignment)
```

`System.out.println(ByteOrder.nativeOrder());` 可以查看当前cpu的字节序。输出是LITTLE_ENDIAN意味着是小端序。
* 小端序：数据的高位字节存放在地址的高端 低位字节存放在地址低端

* 大端序： 数据的高位字节存放在地址的低端 低位字节存放在地址高端
比如一个整形0x1234567 ，1是高位数据，7是低位数据。按照小端序01放在内存地址的高位，比如放在0x100  ，23就放在0x101以此类推。大端序反之。
![字节序](https://gitee.com/neusnail/img/raw/master/blogimg/%E5%AD%97%E8%8A%82%E5%BA%8F.gif)
因为堆的地址序是从低到高的所以大端序更符合人类的阅读习惯。而大部分intel和amd的cpu都是使用的小端序。关闭指针压缩的输出如下:
```
jdk_test.ObjectHeadTest object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           09 f4 e1 46 (00001001 11110100 11100001 01000110) (1189213193)
      4     4        (object header)                           79 00 00 00 (01111001 00000000 00000000 00000000) (121)
      8     4        (object header)                           18 55 a0 ab (00011000 01010101 10100000 10101011) (-1415555816)
     12     4        (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
     16     4    int ObjectHeadTest.intValue                   0
     20     4        (loss due to the next object alignment)
Instance size: 24 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

```
前8位是markword,后8位是Klass指针。先不管Klass指针，我们把markword转成大端序便于查看
```shell
#markword with big endian
00000000 00000000 00000000 01111001 01000110 11100001 11110100 00001001
```

前25位是未使用的，之后的31位是hashcode 001111001010001101110000111110100（第一个0为符号位）转成十进制即为2034688500和hashCode()方法的值一致
最后的8位00001001第一个0没使用 后面的001对应age,biased_lock和lock。这三个字段的详细说明如下：
> - age - number of garbage collections the object has survived. It is incremented every time an object is copied within the young generation. When the age field reaches the value of max-tenuring-threshold, the object is promoted to the old generation.
> - biased_lock - contains 1 if the biased locking is enabled for the class. 0 if the biased locking is disabled for the class. To find out more on biased locking check out my previous post.
> - lock - the lock state of the object. 00 - Lightweight Locked, 01 - Unlocked or Biased, 10 - Heavyweight Locked, 11 - Marked for Garbage Collection. To find out more on locking/synchronization check out synchronization post. 

age是对象经历的gc次数，biased_lock代表是否使用偏向锁，lock是锁的状态这里001表示没有使用锁

header占用12个字节，之后的4个字节是对象的int属性，之后又4个字节的对齐填充。

由于HotSpot VM的自动内存管理系统要求对象起始地址必须是8字节的整数倍，也就是说就是对象的大小必须是8字节的整数倍。如上header加上fileld是20字节，所以需要4字节的对齐填充。