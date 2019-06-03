### stringBuffer&StringBuilder

stringBuilder是stringBuffer的线程安全版本

stringbuffer和string操作几乎一样

```java
public void testString() {
    StringBuffer stringBuffer = new StringBuffer();
    stringBuffer.append("this is ");
    stringBuffer.append("a string");
    System.out.println(stringBuffer.toString());
}
```

stringBuffer的定义:

```
 A thread-safe, mutable sequence of characters.
 A string buffer is like a {@link String}, but can be modified. At any
 point in time it contains some particular sequence of characters, but
 the length and content of the sequence can be changed through certain
 method calls.
```

String是不可变的对象，因此每次在对String类进行改变的时候都会生成一个新的string对象，然后将指针指向新的string对象，所以经常要改变字符串长度的话不要使用string，因为每次生成对象都会对系统性能产生影响.

```java
@Test
public void testString() {
    String a = "im a";
    System.out.println(a.hashCode());
    a = a + " with a tail";
    System.out.println(a.hashCode());
}
 /* output:
      3233893
      1745173006*/
```

