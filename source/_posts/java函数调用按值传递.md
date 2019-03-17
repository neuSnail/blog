---
title: java函数调用按值传递
date: 2018-03-17 13:53:21
tags: java
---

在c++中一个函数调用是值传递还是引用传递在函数声明时即可以很清楚的分辨，但是java中并没有显示的指针和引用使用.这就引起了一些人的误解和争执，其实没必要争什么 java官方也给了回答：`java中所有的调用都是值传递`。
在证明这点前先讲一下java的局部变量存储策略（熟悉的同学可以跳过）。我们都知道jvm会将内存划为栈区和堆区。每个线程会持有一个内存栈，类函数在执行时的局部变量和对象的引用都保存在栈中。而new出来的对象保存在堆上(和C中的指针很像)  `Person personA=new Person("A")`这段代码personA是Person实体的引用保存在栈上。personA实际存储的是Person实例的地址。
画个图就是这样：

![image-20190316230141929](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190316230141929.png)

下面举一个赋值的例子

```java
 class Person {
        private String name;

        Person(String name) {
            this.name = name;
        }

        public void sayName() {
            System.out.println(name);
        }

        public void setName(String name) {
            this.name = name;
        }
    }
```

```java
 @Test
    public void objectTransmit() {
        //personA 指向Person("A")的堆地址
        Person personA = new Person("A");
        //将personA的堆地址赋值给personB 此时personB也指向Person("A")
        Person personB = personA;
        //更改personB的名字
        personB.setName("A's new name");
        personA.sayName();
        //新建一个PersonC并将堆地址赋值给B
        personB = new Person("C");
        //此时A还是指向PersonB指向C
        personA.sayName();
        personB.sayName();
    }
```

输出结果

A's new name
A's new name
C

上面代码第6行实际是将personA栈区的值赋值给personB，此时personA和B都持有Person("A")实例的地址，所以此时无论对personA还是personB操作会修改实例的的属性。而第11行将一个新的实例赋值给personB 使得personB持有Person("C")的地址，但这并没有对personA产生影响,personA还是持有Person("A")实例的地址。



理解了上述问题后我们再来看下面的例子，下面的代码两次sayName输出的都是"new name"

```java
   @Test
    public void methodCall() {
        Person personA = new Person("A");
        func1(personA);
        personA.sayName();

        func2(personA);
        personA.sayName();
    }

    private void func1(Person person) {
        person.setName("new name");
    }

    private void func2(Person person) {
        person = null;
    }
```

有些人可能会奇怪 在func2中不是把person赋值为null了吗 第二次sayName()不应该报空指针吗? 实际上func2中会对实参personA做一次拷贝,将这个拷贝指向null并不会影响主函数中的实际参数.而func1中会修改实际对象中的属性是因为实参的拷贝也是指向Person("A")对象



下图展示了这一过程 其中person参数是对personA的拷贝

![image-20190317121033320](https://gitee.com/neusnail/img/raw/master/blogimg/image-20190317121033320.png)

#### 总结

java中的函数调用无论传递的是值还是引用,都是按值传递的,即在函数中会做一次拷贝.对该拷贝进行的操作(不涉及堆区对象的操作)不会影响原始值. 但由于拷贝和原始引用都指向同一个堆区对象,调用对象方法时都会反应在实际对象上.
