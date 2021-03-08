别再问我 new 字符串创建了几个对象了！我来证明给你看！

## 结论

本文我们通过 `javap -v XXX` 的方式查看编译的代码发现 new String 首次会在字符串常量池中创建此字符串，那也就是说，**通过 new 创建字符串的方式可能会创建 1 个或 2 个对象，如果常量池中已经存在此字符串只会在堆上创建一个变量，并指向字符串常量池中的值，如果字符串常量池中没有相关的字符，会先创建字符串在返回此字符串的引用给堆空间的变量**。我们还将了字符串常量池在 JDK 1.7 和 JDK 1.8 的变化以及编译器对确定字符串的优化，希望能帮你正在的理解字符串的比较。

-----

我想所有 Java 程序员都曾被这个 new String 的问题困扰过，这是一道高频的 Java 面试题，但可惜的是网上众说纷纭，竟然找不到标准的答案。有人说创建了 1 个对象，也有人说创建了 2 个对象，还有人说可能创建了 1 个或 2 个对象，但谁都没有拿出干掉对方的证据，这就让我们这帮吃瓜群众们陷入了两难之中，不知道到底该信谁得。

但是今天，老王就斗胆和大家聊聊这个话题，顺便再拿出点**证据**。

以目前的情况来看，关于 `new String("xxx")` 创建对象个数的答案有 3 种：

1. 有人说创建了 1 个对象；
2. 有人说创建了 2 个对象；
3. 有人说创建了 1 个或 2 个对象。

**而出现多个答案的关键争议点在「字符串常量池」上**，有的说 new 字符串的方式会在常量池创建一个字符串对象，有人说 new 字符串的时候并不会去字符串常量池创建对象，而是在调用 `intern()` 方法时，才会去字符串常量池检测并创建字符串。

那我们就先来说说这个「字符串常量池」。

## 字符串常量池

字符串的分配和其他的对象分配一样，需要耗费高昂的时间和空间为代价，如果需要大量频繁的创建字符串，会极大程度地影响程序的性能，因此 JVM 为了提高性能和减少内存开销引入了字符串常量池（Constant Pool Table）的概念。

字符串常量池相当于给字符串开辟一个常量池空间类似于缓存区，对于直接赋值的字符串（String s="xxx"）来说，在每次创建字符串时优先使用已经存在字符串常量池的字符串，如果字符串常量池没有相关的字符串，会先在字符串常量池中创建该字符串，然后将引用地址返回变量，如下图所示：



*![字符串常量池示意图.png](https://user-gold-cdn.xitu.io/2020/4/17/17185b71e1254c63?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)*

以上说法可以通过如下代码进行证明：



```
public class StringExample {
    public static void main(String[] args) {
        String s1 = "Java";
        String s2 = "Java";
        System.out.println(s1 == s2);
    }
}
复制代码
```

以上程序的执行结果为：`true`，说明变量 s1 和变量 s2 指向的是同一个地址。

在这里我们顺便说一下字符串常量池的再不同 JDK 版本的变化。

## 常量池的内存布局

从**JDK 1.7 之后把永生代换成的元空间，把字符串常量池从方法区移到了 Java 堆上**。

JDK 1.7 内存布局如下图所示：

![JDK 1.7 内存布局.png](https://user-gold-cdn.xitu.io/2020/4/17/17185b71e294b6bb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

JDK 1.8 内存布局如下图所示：

![JDK 1.8 内存布局.png](https://user-gold-cdn.xitu.io/2020/4/17/17185b71e2e144fc?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

**JDK 1.8 与 JDK 1.7 最大的区别是 JDK 1.8 将永久代取消，并设立了元空间**。官方给的说明是**由于永久代内存经常不够用或发生内存泄露**，会爆出 java.lang.OutOfMemoryError: PermGen 的异常，所以把将永久区废弃而改用元空间了，改为了使用本地内存空间，官网解释详情：[openjdk.java.net/jeps/122](http://openjdk.java.net/jeps/122)



## 答案解密

认为 new 方式创建了 1 个对象的人认为，new String 只是在堆上创建了一个对象，只有在使用 `intern()` 时才去常量池中查找并创建字符串。

认为 new 方式创建了 2 个对象的人认为，new String 会在堆上创建一个对象，并且在字符串常量池中也创建一个字符串。

认为 new 方式有可能创建 1 个或 2 个对象的人认为，new String 会先去常量池中判断有没有此字符串，如果有则只在堆上创建一个字符串并且指向常量池中的字符串，如果常量池中没有此字符串，则会创建 2 个对象，先在常量池中新建此字符串，然后把此引用返回给堆上的对象，如下图所示：

![new 字符串常量池.png](https://user-gold-cdn.xitu.io/2020/4/17/17185b71e45e6c16?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



**老王认为正确的答案：创建 1 个或者 2 个对象**。

### 技术论证

解铃还须系铃人，回到问题的那个争议点上，new String 到底会不会在常量池中创建字符呢？我们通过反编译下面这段代码就可以得出正确的结论，代码如下：

```
public class StringExample {
    public static void main(String[] args) {
        String s1 = new String("javaer-wang");
        String s2 = "wang-javaer";
        String s3 = "wang-javaer";
    }
}
复制代码
```

首先我们使用 `javac StringExample.java` 编译代码，然后我们再使用 `javap -v StringExample` 查看编译的结果，相关信息如下：

```
Classfile /Users/admin/github/blog-example/blog-example/src/main/java/com/example/StringExample.class
  Last modified 2020年4月16日; size 401 bytes
  SHA-256 checksum 89833a7365ef2930ac1bc3d7b88dcc5162da4b98996eaac397940d8997c94d8e
  Compiled from "StringExample.java"
public class com.example.StringExample
  minor version: 0
  major version: 58
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #16                         // com/example/StringExample
  super_class: #2                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #2.#3          // java/lang/Object."<init>":()V
   #2 = Class              #4             // java/lang/Object
   #3 = NameAndType        #5:#6          // "<init>":()V
   #4 = Utf8               java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Class              #8             // java/lang/String
   #8 = Utf8               java/lang/String
   #9 = String             #10            // javaer-wang
  #10 = Utf8               javaer-wang
  #11 = Methodref          #7.#12         // java/lang/String."<init>":(Ljava/lang/String;)V
  #12 = NameAndType        #5:#13         // "<init>":(Ljava/lang/String;)V
  #13 = Utf8               (Ljava/lang/String;)V
  #14 = String             #15            // wang-javaer
  #15 = Utf8               wang-javaer
  #16 = Class              #17            // com/example/StringExample
  #17 = Utf8               com/example/StringExample
  #18 = Utf8               Code
  #19 = Utf8               LineNumberTable
  #20 = Utf8               main
  #21 = Utf8               ([Ljava/lang/String;)V
  #22 = Utf8               SourceFile
  #23 = Utf8               StringExample.java
{
  public com.example.StringExample();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=3, locals=4, args_size=1
         0: new           #7                  // class java/lang/String
         3: dup
         4: ldc           #9                  // String javaer-wang
         6: invokespecial #11                 // Method java/lang/String."<init>":(Ljava/lang/String;)V
         9: astore_1
        10: ldc           #14                 // String wang-javaer
        12: astore_2
        13: ldc           #14                 // String wang-javaer
        15: astore_3
        16: return
      LineNumberTable:
        line 5: 0
        line 6: 10
        line 7: 13
        line 8: 16
}
SourceFile: "StringExample.java"
复制代码
```

> 备注：以上代码的运行也编译环境为 jdk1.8.0_101。

其中 `Constant pool` 表示字符串常量池，我们在字符串编译期的字符串常量池中找到了我们 `String s1 = new String("javaer-wang");` 定义的“javaer-wang”字符，在信息 `#10 = Utf8 javaer-wang` 可以看出，也就是**在编译期 new 方式创建的字符串就会被放入到编译期的字符串常量池中**，也就是说 new String 的方式会首先去判断字符串常量池，如果没有就会新建字符串那么就会创建 2 个对象，如果已经存在就只会在堆中创建一个对象指向字符串常量池中的字符串。

那么问题来了，以下这段代码的执行结果为 true 还是 false？

```
String s1 = new String("javaer-wang");
String s2 = new String("javaer-wang");
System.out.println(s1 == s2);
复制代码
```

既然 new String 会在常量池中创建字符串，那么执行的结果就应该是 true 了。其实并不是，这里对比的变量 s1 和 s2 堆上地址，因为堆上的地址是不同的，所以结果一定是 false，如下图所示：



![字符串引用.png](https://user-gold-cdn.xitu.io/2020/4/17/17185b71e2f9e78a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

从图中可以看出 s1 和 s2 的引用一定是相同的，而 s3 和 s4 的引用是不同的，对应的程序代码如下：



```
public static void main(String[] args) {
    String s1 = "Java";
    String s2 = "Java";
    String s3 = new String("Java");
    String s4 = new String("Java");
    System.out.println(s1 == s2);
    System.out.println(s3 == s4);
}
复制代码
```

程序执行的结果也符合预期：

> true false

## 扩展知识

我们知道 String 是 final 修饰的，也就是说一定被赋值就不能被修改了。但编译器除了有字符串常量池的优化之外，还会对编译期可以确认的字符串进行优化，例如以下代码：

```
public static void main(String[] args) {
    String s1 = "abc";
    String s2 = "ab" + "c";
    String s3 = "a" + "b" + "c";
    System.out.println(s1 == s2);
    System.out.println(s1 == s3);
}
复制代码
```

按照 String 不能被修改的思想来看，s2 应该会在字符串常量池创建两个字符串“ab”和“c”，s3 会创建三个字符串，他们的引用对比结果也一定是 false，但其实不是，他们的结果都是 true，这是编译器优化的功劳。

同样我们使用 `javac StringExample.java` 先编译代码，再使用 `javap -c StringExample` 命令查看编译的代码如下：

```
警告: 文件 ./StringExample.class 不包含类 StringExample
Compiled from "StringExample.java"
public class com.example.StringExample {
  public com.example.StringExample();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: ldc           #7                  // String abc
       2: astore_1
       3: ldc           #7                  // String abc
       5: astore_2
       6: ldc           #7                  // String abc
       8: astore_3
       9: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
      12: aload_1
      13: aload_2
      14: if_acmpne     21
      17: iconst_1
      18: goto          22
      21: iconst_0
      22: invokevirtual #15                 // Method java/io/PrintStream.println:(Z)V
      25: getstatic     #9                  // Field java/lang/System.out:Ljava/io/PrintStream;
      28: aload_1
      29: aload_3
      30: if_acmpne     37
      33: iconst_1
      34: goto          38
      37: iconst_0
      38: invokevirtual #15                 // Method java/io/PrintStream.println:(Z)V
      41: return
}
复制代码
```

从 Code 3、6 可以看出字符串都被编译器优化成了字符串“abc”了。

## 总结

本文我们通过 `javap -v XXX` 的方式查看编译的代码发现 new String 首次会在字符串常量池中创建此字符串，那也就是说，通过 new 创建字符串的方式可能会创建 1 个或 2 个对象，如果常量池中已经存在此字符串只会在堆上创建一个变量，并指向字符串常量池中的值，如果字符串常量池中没有相关的字符，会先创建字符串在返回此字符串的引用给堆空间的变量。我们还将了字符串常量池在 JDK 1.7 和 JDK 1.8 的变化以及编译器对确定字符串的优化，希望能帮你正在的理解字符串的比较。

```
作者：Java中文社群
链接：https://juejin.cn/post/6844904129752465416
来源：掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。
```

