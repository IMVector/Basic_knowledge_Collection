java之158问

## 基础篇



#### **1. Arrays.sort实现原理和Collections.sort实现原理**

答：

Java数组和List对象的排序方法分别是Arrays.sort(T[] a)和Collections.sort(List<T> list)，分别位于java.util.Arrays和java.util.Collections类中. 由于List底层采用数组实现，因此Collections.sort方法也是通过调用Arrays.sort方法实现的。

Arrays.sort方法源码很简单，基本的数据类型调用了DualPivotQuicksort.sort方法，对于那些可能需要稳定的对象来说使用归并排序。



#### **2. foreach和while的区别(编译之后)**

1) foreach的执行最终转换成了对iterator的调用

#### **3. 线程池的种类，区别和使用场景**

**1、newFixedThreadPool**创建一个指定工作线程数量的线程池。每当提交一个任务就创建一个工作线程，如果工作线程数量达到线程池初始的最大数，则将提交的任务存入到阻塞队列中排队。

**2、newCachedThreadPool**创建一个可缓存的线程池。这种类型的线程池特点是：

*1).工作线程的创建数量几乎没有限制(其实也有限制的,数目为Interger. MAX_VALUE), 这样可灵活的往线程池中添加线程。*

*2).如果长时间没有往线程池中提交任务，即如果工作线程空闲了指定的时间(默认为1分钟)，则该工作线程将自动终止。终止后，如果你又提交了新的任务，则线程池重新创建一个工作线程。*



**3、newSingleThreadExecutor**创建一个单线程化的Executor，即只创建唯一的工作者线程来执行任务，如果这个线程异常结束，会有另一个取代它，保证顺序执行(我觉得这点是它的特色)。单工作线程最大的特点是可保证顺序地执行各个任务，并且在任意给定的时间不会有多个线程是活动的 。

**4、newScheduleThreadPool**创建一个定长的线程池，而且支持定时的以及周期性的任务执行，类似于Timer。(这种线程池原理暂还没完全了解透彻)

**5、 newWorkStealingPool**创建持有足够线程的线程池来达到快速运算的目的，在内部通过使用多个队列来减少各个线程调度产生的竞争

**总结：**

一.FixedThreadPool是一个典型且优秀的线程池，它具有线程池提高程序效率和节省创建线程时所耗的开销的优点。但是，在线程池空闲时，即线程池中没有可运行任务时，它不会释放工作线程，还会占用一定的系统资源。

二．CachedThreadPool的特点就是在线程池空闲时，即线程池中没有可运行任务时，它会释放工作线程，从而释放工作线程所占用的资源。但是，但当出现新任务时，又要创建一新的工作线程，又要一定的系统开销。并且，在使用CachedThreadPool时，一定要注意控制任务的数量，否则，由于大量线程同时运行，很有会造成系统瘫痪。

所有的底层使用的都是：

**ThreadPoolExecutor 类进行的封装实现。**

#### **4. 分析线程池的实现原理和线程的调度过程**

> **线程池的实现底层都调用的是**

```java
public ThreadPoolExecutor(int corePoolSize,    //核心线程的数量
                          int maximumPoolSize,    //最大线程数量
                          long keepAliveTime,    //超出核心线程数量以外的线程空余存活时间
                          TimeUnit unit,    //存活时间的单位
                          BlockingQueue<Runnable> workQueue,    //保存待执行任务的队列
                          ThreadFactory threadFactory,    //创建新线程使用的工厂
                          RejectedExecutionHandler handler // 当任务无法执行时的处理器
                          ) {...}
```

> **ThreadPoolExecutor.execute**

此方法为需要执行任务的具体方法，线程先加入到corePoolSize大小，如果数量大于这个数，则加入到队列中，如果队列满 并且线程数<maxmmunPoolsize 则添加新的线程处理。如果线程数> maxmmunPoolsize则执行RejectHanlder，进行异常处理。

> **线程存放阻塞队列**

*• ArrayBlockingQueue：基于数组、有界，按 FIFO（先进先出）原则对元素进行排序*

*• LinkedBlockingQueue：基于链表，按FIFO （先进先出） 排序元素*

*• 吞吐量通常要高于 ArrayBlockingQueue*

*• Executors.newFixedThreadPool() 使用了这个队列*

*SynchronousQueue：不存储元素的阻塞队列*

*• 每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态*

*• 吞吐量通常要高于 LinkedBlockingQueue*

*• Executors.newCachedThreadPool使用了这个队列*

*PriorityBlockingQueue：具有优先级的、无限阻塞队列*

> **拒绝策略**

 *CallerRunsPolicy：只要线程池没关闭，就直接用调用者所在线程来运行任务*

* AbortPolicy：直接抛出 RejectedExecutionException 异常*

* DiscardPolicy：悄悄把任务放生，不做了*

* DiscardOldestPolicy：把队列里待最久的那个任务扔了，然后再调用 execute() 试试看能行不*

* 我们也可以实现自己的 RejectedExecutionHandler 接口自定义策略，比如如记录日志什么的*

#### **5. 线程池如何调优**

1） **核心线程数设置**

核心线程数跟CPU内核数量有关系，正常为`1`对`1`设置或$2*cpu$设置较为合理。`N`核服务器，通过执行业务的单线程分析出本地计算时间为`x`，等待时间为`y`，则工作线程数（线程池线程数）设置为 $N*(x+y)/x$，能让CPU的利用率最大化。

[线程数究竟设多少合理_w3cschool](https://link.zhihu.com/?target=https%3A//www.w3cschool.cn/architectroad/architectroad-set-the-thread.html)

**2） 最大线程数设置**

最大线程数的设置跟任务处理有关系，过多的线程数会对机器cpu和内存有影响。

**3） 阻塞队列的选择**

可以根据需要选择合适的阻塞队列。

#### **6. 线程池的最大线程数目根据什么确定**

**1） 根据任务来确定**

CPU密集型的要适当的调小线程数，IO密集型的可适当调大线程池。

**2） 任务的依赖性**

有依赖的任务，如：先获取某项资源可适当调大线程池数量。

**3） 不宜过多**

过多的线程数会进行线程上下文切换造成影响。

#### **7. 动态代理的几种方式**

**1） 静态代理**

可以在重构，或简单的地方去使用。

**2） 动态代理**

动态代理分为jdk动态代理和cglib动态代理，动态代理会在运行时自动生成class代理对象。其中jdk动态代理只能代理有接口实现的类，cglib是extends方式可以代理具体的类。在使用上，jdk代理通用，而cglib需要额外的jar包支持。

**3） 动态代理和静态代理的区别**

静态代理：对业务方法的代理，需要重复编写。

动态代理：由代理框架直接生成对应的代理方法，不需要自己去写。只需要做对应的实现即可。

**4） 动态代理不能代理的方法**

被static final 和private 修饰的方法无法提供代理。

## 集合篇

**8、集合**

Java 的集合类分为三大类，List、Map和Set，List为可重复的集合，

Map为K、V方式存储的，

Set 为不能重复的集合。

在使用非线程安全的类可以使用Collections.synchronizedMap() 函数返回的线程安全的HashMap。

#### **1) List**

```java
   		List<String> list = new ArrayList<String>(5);
		List<String> alist = new LinkedList<String>();
		List<String>  vec = new Vector<String>();
		List<String>  cpy = new CopyOnWriteArrayList<String>();
```

***ArrayList :\***

![img](https://pic1.zhimg.com/80/v2-dbac9706f12e8c88ff871bbbca9738f0_720w.jpg)

指定大小的数组列表，写入数据需要移动数据，查询数据速度快。

扩容方法调用的为 Arrays.copyOf(elementData, newCapacity); 方法，copy 元素并增加长度。

最大长度为：MAX_ARRAY_SIZE = Integer.MAX_VALUE – 8，超过此长度throw new OutOfMemoryError() 。

线程不安全。随机读取速度快。

***LikedList\***:

![img](https://pic2.zhimg.com/80/v2-98fbb5c030b43a4ef8fff5906a31eb89_720w.jpg)

链表结构，不指定大小，写入数据块，查询慢。

写入不需要copy数据，只需要将上个元素的指针指向当前新加入的元素即可。

Likedlist由于每个元素都有一个指向下个元素的指针，所以适合顺序读取。

数组结构：

```java
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
Add方法：
    void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null);
        last = newNode;
        if (l == null)
            first = newNode;
        else
            l.next = newNode;
        size++;
        modCount++;
    }
Get核心：使用for循环扫描每个元素。
    Node<E> node(int index) {
        // assert isElementIndex(index);
        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```

**Vector:**

线程安全的数组结构(ArrayList)，核心在于此类的每个操作方法都加入了synchronized方法，保证访问的线程安全。

```java
public synchronized E get(int index) {
        if (index >= elementCount)
            throw new ArrayIndexOutOfBoundsException(index);

        return elementData(index);
    }
```

**CopyOnWriteArrayList:**

线程安全的Arraylist，不适合大量写，写的时候会先生成一个copy，写完后修改之前的对象应用到新的对象。

在写的时候加入ReentrantLock 锁，在读取的时候没有用锁机制。

#### ***2)\* Map**

***HashMap\***:

![img](https://pic2.zhimg.com/80/v2-1857d6b91efd781ada326a673e633d29_720w.jpg)

Hash算法实现的KV存储，add方法：

```java
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
```

给key计算一个hash值，然后进行存储。获取元素：

```java
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```

获取元素对比hash值。

***HashTable\*:**

线程安全的KV存储，此种方式的线程安全实现是在方法上加入同步关键字实现的，效率比较低。不推荐使用

***ConcurrentHashMap\***:

Jdk6 和 JDK7 中 ConcurrentHashMap采用了分段锁的设计，只有在同一个分段内才存在竞态关系，不同的分段锁之间没有锁竞争。相比于对整个Map加锁的设计，分段锁大大的提高了高并发环境下的处理能力。但同时，由于不是对整个Map加锁，导致一些需要扫描整个Map的方法（如size(), containsValue()）需要使用特殊的实现，另外一些方法（如clear()）甚至放弃了对一致性的要求（ConcurrentHashMap是弱一致性的，具体请查看），hashtable 是强一致性的，对put的数据马上可get到，而ConcurrentHashMap对put的数据直接get可能获取不到。

***LinkedHashMap\*:**

LinkedHashMap是Hash表和链表的实现，并且依靠着双向链表保证了迭代顺序是插入的顺序。非线程安全，LinkedHashMap除了支持默认的插入顺序，还支持访问顺序。所谓访问顺序(access-order)是指在迭代遍历列表中的元素时最近访问的元素会排在LinkedHashMap的尾部 。从其构造函数中可以看出当accessOrder设为true时即为访问顺序。基于LinkedHashMap的访问顺序的特点，可构造一个LRU（Least Recently Used）最近最少使用简单缓存。也有一些开源的缓存产品如ehcache的淘汰策略（LRU）就是在LinkedHashMap上扩展的。

两个特性：

- Linkedhashmap 是是有序的，插入的顺序和遍历的顺序可以保持一致。
- 支持访问顺序，在迭代遍历列表中的元素时最近访问过的元素排列到末尾。

***TreeMap\***

Treemap在实现的过程中，采用的是红黑树进行的实现。

TreeMap

#### **3） Set**

**HashSet:** 无排序，非线程安全。

**TreeSet**：有序，可以自定义排序规则，非线程安全。

**CopyOnWriteHashSet:**

```java
  Set<String> hset = new CopyOnWriteArraySet<String>();
```

线程安全。Add操作如下：

```java
private boolean addIfAbsent(E e, Object[] snapshot) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] current = getArray();
            int len = current.length;
            if (snapshot != current) {
                // Optimize for lost race to another addXXX operation
                int common = Math.min(snapshot.length, len);
                for (int i = 0; i < common; i++)
                    if (current[i] != snapshot[i] && eq(e, current[i]))
                        return false;
                if (indexOf(e, current, common, len) >= 0)
                        return false;
            }
            Object[] newElements = Arrays.copyOf(current, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```

**4) Queue**

**ArrayBlockingQueue: 固定大小数组队列**

**LinkedBlockingQueue: 链表队列**

**PriorityBlockingQueue: 优先级队列**

**TransferBlockingQueue:**

**DelayBlockingQueue: 延迟队列**

**SynchronousQueue:**

**ConcurrentLinkedQueue：**

**ConcurrentLinkedQueue**

使用CAS非阻塞算法实现使用CAS解决了当前节点与next节点之间的安全链接和对当前节点值的赋值。由于使用CAS没有使用锁，所以获取size的时候有可能进行offer，poll或者remove操作，导致获取的元素个数不精确，所以在并发情况下size函数不是很有用。另外第一次peek或者first时候会把head指向第一个真正的队列元素。

***注意1\***：

ConcurrentLinkedQueue的.size() 是要遍历一遍集合的，很慢的，所以尽量要避免用size，

如果判断队列是否为空最好用isEmpty()而不是用size来判断.

注意问题1：

使用了这个ConcurrentLinkedQueue 类之后是否意味着我们不需要自己进行任何同步或加锁操作了呢？

如果直接使用它提供的函数，比如：queue.add(obj); 或者 queue.poll(obj);，这样我们自己不需要做任何同步。

但如果是非原子操作，比如：

if(!queue.isEmpty()) {

queue.poll(obj);

}

我们很难保证，在调用了isEmpty()之后，poll()之前，这个queue没有被其他线程修改。

#### **9. 了解LinkedHashMap的应用吗**

可以实现基于LRU策略的简单缓存。LinkedHashmap会将访问过的元素排列到队列的最后。此数据结构维护除了维护要给hashmap还有一个访问顺序的链表。

[linkedHashMap的应用 - CSDN博客](https://link.zhihu.com/?target=http%3A//blog.csdn.net/kiss_the_sun/article/details/7848920)

**10. 反射的原理，反射创建类实例的三种方式是什么？**

调用示例：

```java
public class ReflectCase {
 
    public static void main(String[] args) throws Exception {
        Proxy target = new Proxy();
        Method method = Proxy.class.getDeclaredMethod("run");
        method.invoke(target);
    }
 
    static class Proxy {
        public void run() {
            System.out.println("run");
        }
    }
}
```

每次getDeclaredMethod都会生成一个新的实例。

**调用的三种方式：**

```java
package cn.hncu.reflect.test;
import org.junit.Test;
/**
 * 1、演示获取Class c对象的三种方法
 *@author <a"283505495@qq.com">lxd</a>
 *@version 1.0 2017-4-15 下午2:57:05
 *@fileName ReflectGetClass.java
 */
public class ReflectGetClass {

    /**
     * 法1：通过对象---对象.getClass()来获取c(一个Class对象)
     */
    @Test
    public void get1(){
        Person p=new Person("Jack", 23);
        Class c=p.getClass();//来自Object方法
    }

    /**
     * 法2：通过类(类型)---任何数据类型包括(基本数据类型)都有一个静态的属性class ，他就是c 一个Class对象
     */
    @Test
    public void get2(){
        Class c=Person.class;
        Class c2=int.class;
    }

    /**
     * 法3：通过字符串(类全名 )---能够实现解耦：Class.forName(str)
     */
    @Test
    public void get3(){
        try {
            Class c=Class.forName("cn.hncu.reflect.test.Person");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }

}
```



#### **11. cloneable接口实现原理，浅拷贝or深拷贝**

需要复制一份当前对象。

#### **12. Java NIO使用**

#### **13. hashtable和hashmap的区别及实现原理，hashmap会问到数组索引，hash碰撞怎么解决**

***拉链法解决冲突的做法是***

将所有关键字为同义词的结点链接在同一个单链表中。若选定的散列表长度为m，则可将散列表定义为一个由m个头指针组成的指针数组t[0..m-1]。凡是散列地址为i的结点，均插入到以t为头指针的单链表中。t中各分量的初值均应为空指针。在拉链法中，装填因子α可以大于1，但一般均取α≤1。

换句话说：HashCode是使用Key通过Hash函数计算出来的，由于不同的Key，通过此Hash函数可能会算的同样的HashCode，所以此时用了拉链法解决冲突，把HashCode相同的Value连成链表. 但是get的时候根据Key又去桶里找，如果是链表说明是冲突的，此时还需要检测Key是否相同

在解释下，Java中HashMap是利用“拉链法”处理HashCode的碰撞问题。在调用HashMap的put方法或get方法时，都会首先调用hashcode方法，去查找相关的key，当有冲突时，再调用equals方法。hashMap基于hasing原理，我们通过put和get方法存取对象。当我们将键值对传递给put方法时，他调用键对象的hashCode()方法来计算hashCode，然后找到bucket（哈希桶）位置来存储对象。当获取对象时，通过键对象的equals()方法找到正确的键值对，然后返回值对象。HashMap使用链表来解决碰撞问题，当碰撞发生了，对象将会存储在链表的下一个节点中。hashMap在每个链表节点存储键值对对象

#### **14. 反射中，Class.forName和ClassLoader区别**

java中class.forName和classLoader都可用来对类进行加载。前者除了将类的.class文件加载到jvm中之外，还会对类进行解释，执行类中的static块。而classLoader只干一件事情，就是将.class文件加载到jvm中，不会执行static中的内容,只有在newInstance才会去执行static块。Class.forName(name, initialize, loader)带参函数也可控制是否加载static块。并且只有调用了newInstance()方法采用调用构造函数，创建类的对象。

#### **15. String，Stringbuffer，StringBuilder的区别？**

String：不可变对象，对字符的操作会产生一个copy然后进行操作。

StringBuffer：为线程安全的对象，安全操作方式为在方法加入synchronized同步关键字，所有性能不如StringBuilder类。

StringBuilder：线程不安全的字符操作对象。

String a=”xx”

String a=”xx”+”bbb”+”cc”

A的优化种a作为一个常量字符串进行执行。不重新定义很快执行。

#### **16. 有没有可能2个不相等的对象有相同的hashcode**

#### 有可能，使用拉链法解决

#### **17. 简述NIO的最佳实践，比如netty，mina**

#### **18. TreeMap的实现原理**



## JVM篇



**19. 类的实例化顺序，比如父类静态数据，构造函数，字段，子类静态数据，构造函数，字段，他们的执行顺序**

***20. JVM内存分代\***

***21. Java 8的内存分代改进\***

***22. JVM垃圾回收机制，何时触发MinorGC等操作\***

**1） 引用技术算法**

当有增加对对象的应用时，引用计数器+1，当引用失效时，计数器-1，当为0是可以标记为回收。引用技术不好解决的问题是：对象之间的相互循环引用。

对象引用分类：

 **强引用**

Object obj = new Object(); 类似这样的引用。垃圾回收永远不会回收强应用类型。

** 软引用**

系统将要发生OOM之前将这些对象列为回收的范围内，进行第二次回收。Jdk1.2之后它提供了softReference

** 弱引用**

弱引用对象只能生存到下次垃圾收集之前，提供了jdk workReference类来实现。在垃圾回收器执行前，无论当前内存是否够用都会进行垃圾回收。

** 虚引用（幽灵引用）**

这种引用是最弱的一种引用方式，设置虚引用的唯一目的：

能在这个对象在被回收时收到一个系统通知。Jdk使用PhantomReference实现虚引用。

**2） 根搜索算法**

从GC Roots节点开始搜索，当对象没有任何引用链相连时，判定为此的对象不可用。

![img](https://pic1.zhimg.com/80/v2-cf9be2a217e69e0463ab7d48801c5b8c_720w.jpg)

根搜索算法执行两次标记动作以回收。Finalize 会被执行。

**3） 方法区垃圾回收**

条件比较苛刻，回收资源没有堆heap这种快。

#### **23. 垃圾回收算法**

**1) 标记-清除算法**

首先标记出所有需要回收的对象，在标记完成后统一回收掉被标记的对象，此方式的缺点：效率不高，内存回收后不连续。内存碎片较多。

![img](https://pic1.zhimg.com/80/v2-0409d78ccf578701d536745baae87780_720w.jpg)

**2) 复制算法**

将内存按照容量划分为大小相等的两块，每次使用其中的一块，当这块内存用完了就将存货的对象复制到另外一块上面，然后再把已经使用过的内存空间一次清理掉。此种方式不存在内存碎片的问题，但是占用的内存较大。将内存缩小为原来的一半。

![img](https://pic2.zhimg.com/80/v2-b55e2c354c352f3ecec5b10d8b229045_720w.jpg)

![img](https://pic2.zhimg.com/80/v2-6ae47d736bade505d819033a86014ec1_720w.jpg)

**3) 标记-整理算法**

![img](https://pic1.zhimg.com/80/v2-5edff05c5b71dabc51fc77c7e5ef4860_720w.jpg)

![img](https://pic1.zhimg.com/80/v2-c53be86b9c21dcf86dff162ada4ac488_720w.jpg)

**4) 分代收集算法**

![img](https://pic3.zhimg.com/80/v2-48401c786168194204eda63d216dac72_720w.jpg)

#### **24. 垃圾收集器**

**1） Serial收集器**

在JDK1.3之前是虚拟机新生代的唯一选择，它是单线程的收集器在进行垃圾回收时会暂停其他工作。

![img](https://pic1.zhimg.com/80/v2-8b94c4fe134b6df95e6e13391343ba18_720w.jpg)

**2） ParNew收集器**

![img](https://pic2.zhimg.com/80/v2-c502f6a80350b58a8f5ab8cecd93ebb1_720w.jpg)

![img](https://pic4.zhimg.com/80/v2-f5247feea17ebdb497627094f1b07db7_720w.jpg)

**3） Parallel Scavenge 收集器**

![img](https://pic3.zhimg.com/80/v2-05ccdf842bd05879f10c7e966808cad6_720w.jpg)

**4） Serial Old收集器**

![img](https://pic4.zhimg.com/80/v2-9bb50696152f3d9f968cc915b75f1e47_720w.jpg)

**5） Parallel Old收集器**

![img](https://pic2.zhimg.com/80/v2-9084427b30e17f2134bab345e37e0abd_720w.jpg)



**6） CMS收集器**

Concurrent Mark Sweep，并发标记收集器。是以获取最短收集停顿时间为目标的收集算法，收集的整个过程分为4个阶段：

*** 初始标记\***

*** 并发标记\***

*** 重新标记\***

*** 并发清除\***

初始标记和并发标记任然需要stop then world，初始标记标记一下能直接GC Roots关联起来的对象。

![img](https://pic1.zhimg.com/80/v2-91e4cfc4b9db1e14469df6e6e05c1d58_720w.jpg)

CMS对cpu敏感、无法处理浮动垃圾。

CMS对cpu敏感、无法处理浮动垃圾。

**7） G1收集器**

![img](https://pic4.zhimg.com/80/v2-287b764e14c652278819437ea5dcc053_720w.jpg)

**8） 收集器参数**

![img](https://pic2.zhimg.com/80/v2-2cfa322d2d388d9622bf519515931d75_720w.jpg)

#### **25 内存分配与回收策略**

***1） 对象优先在eden分配***

***2） 大对象直接进入老年代***

![img](https://pic3.zhimg.com/80/v2-72ae0b83545714e3539a14a3bb547152_720w.jpg)

**3） 长期存活的对象将进入老年代**

![img](https://pic3.zhimg.com/80/v2-9a36ef9ec57174c64ae500bfaafc38ce_720w.jpg)

**4） 动态对象年龄判断**

![img](https://pic4.zhimg.com/80/v2-22218db65bb0656c7449d3e57b669c3f_720w.jpg)

**5） 空间分配担保**

![img](https://pic1.zhimg.com/80/v2-520492dc387ce7d0fc743f39cb71a704_720w.jpg)

#### **26. jvm中一次完整的GC流程（从ygc到fgc）是怎样的，重点讲讲对象如何晋升到老年代，几种主要的jvm参数**

GC（或Minor GC）：收集 生命周期短的区域(Young area)。

Full GC （或Major GC）：收集生命周期短的区域(Young area)和生命周期比较长的区域(Old area)对整个堆进行垃圾收集。

他们的收集算法不同，所以使用的时间也不同。 GC 效率也会比较高，我们要尽量减少 Full GC 的次数。 当显示调用System.gc() 时，gc does a full collection(both young generation and tenured generation).

***FullGC的触发条件：\***

[触发JVM进行Full GC的情况及应对策略 - CSDN博客](https://link.zhihu.com/?target=http%3A//blog.csdn.net/chenleixing/article/details/46706039/)

**1） System.gc()方法的调用**

此方法的调用是建议JVM进行Full GC,虽然只是建议而非一定,但很多情况下它会触发 Full GC,从而增加Full GC的频率,也即增加了间歇性停顿的次数。强烈影响系建议能不使用此方法就别使用，让虚拟机自己去管理它的内存，可通过通过-XX:+ DisableExplicitGC来禁止RMI调用System.gc。

***2） 老年代代空间不足\***

老年代空间只有在新生代对象转入及创建为大对象、大数组时才会出现不足的现象，当执行Full GC后空间仍然不足，则抛出如下错误：

java.lang.OutOfMemoryError: Java heap space

为避免以上两种状况引起的Full GC，调优时应尽量做到让对象在Minor GC阶段被回收、让对象在新生代多存活一段时间及不要创建过大的对象及数组。

***3） 永久代空间不足\***

JVM规范中运行时数据区域中的方法区，在HotSpot虚拟机中又被习惯称为永生代或者永生区，Permanet Generation中存放的为一些class的信息、常量、静态变量等数据，当系统中要加载的类、反射的类和调用的方法较多时，Permanet Generation可能会被占满，在未配置为采用CMS GC的情况下也会执行Full GC。如果经过Full GC仍然回收不了，那么JVM会抛出如下错误信息：

java.lang.OutOfMemoryError: PermGen space

为避免Perm Gen占满造成Full GC现象，可采用的方法为增大Perm Gen空间或转为使用CMS GC。

***4） 大对象\***

所谓大对象，是指需要大量连续内存空间的java对象，例如很长的数组，此种对象会直接进入老年代，而老年代虽然有很大的剩余空间，但是无法找到足够大的连续空间来分配给当前对象，此种情况就会触发JVM进行Full GC。

为了解决这个问题，CMS垃圾收集器提供了一个可配置的参数，即-XX:+UseCMSCompactAtFullCollection开关参数，用于在“享受”完Full GC服务之后额外免费赠送一个碎片整理的过程，内存整理的过程无法并发的，空间碎片问题没有了，但提顿时间不得不变长了，JVM设计者们还提供了另外一个参数 -XX:CMSFullGCsBeforeCompaction,这个参数用于设置在执行多少次不压缩的Full GC后,跟着来一次带压缩的。

#### **27. java堆内存划分**

[java堆内存的划分 - CSDN博客](https://link.zhihu.com/?target=http%3A//blog.csdn.net/liudezhicsdn/article/details/51057592)

**1） 新生代**

而是将内存分为一块比较大的Eden空间和两块较小的Survivor空间

[jvm中的新生代Eden和survivor区 - CSDN博客](https://link.zhihu.com/?target=http%3A//blog.csdn.net/wy5612087/article/details/52369677)

**参数设置：**

**1） -XX:NewSize和-XX:MaxNewSize**

**用于设置年轻代的大小，建议设为整个堆大小的1/3或者1/4,两个值设为一样大。**

**2)-XX:SurvivorRatio**

**用于设置Eden和其中一个Survivor的比值，这个值也比较重要。**

**3)-XX:+PrintTenuringDistribution**

**这个参数用于显示每次Minor GC时Survivor区中各个年龄段的对象的大小。**

**4).-XX:InitialTenuringThreshol和-XX:MaxTenuringThreshold**

用于设置晋升到老年代的对象年龄的最小值和最大值，每个对象在坚持过一次Minor GC之后，年龄就加1。

2） 老年代

3） 永久代

存放了要加载的类信息、静态变量、final类型的常量、属性和方法信息。JVM用永久代（PermanetGeneration）来存放方法区，（在JDK的HotSpot虚拟机中，可以认为方法区就是永久代，但是在其他类型的虚拟机中，没有永久代的概念，有关信息可以看周志明的书）可通过-XX:PermSize和-XX:MaxPermSize来指定最小值和最大值。

#### **28. 你知道哪几种垃圾收集器，各自的优缺点，重点讲下cms，g1**



#### **29. 新生代和老生代的内存回收策略**

#### **30. Eden和Survivor的比例分配等**

Hotspot中为8:1

#### **31. 深入分析了Classloader，双亲委派机制**

#### **32. JVM的编译优化**

#### **33. 对Java内存模型的理解，以及其在并发中的应用**

#### **34. 指令重排序，内存栅栏等**

#### **35. OOM错误，stackoverflow错误，permgen space错误**

#### **OOM：在jvm栈、方法区和堆heap发生对象创建请求和内存请求时候如果内存不足则会引发OOM报错**

#### **Stackoverflow：如：在递归调用中如果栈帧大小超过最大长度则发生stackoverflow。**

#### **Permgenspace：错误一般发生在对象创建上。伴随着OOM一同出现。**

#### **36. JVM常用参数**



## 并发控制篇



**38. volatile的语义，它修饰的变量一定线程安全吗**

此关键字修饰的变量保证了可见性，多个cpu缓存会造成变量的并发问题。此关键字修饰后会强制写或读主缓存，从而保证了变量的可见性，但是不一定是线程安全的。不能保证线程的安全。比如count++，这种获取值再相加的两步操作就不能保证原子操作。

#### **39. g1和cms区别,吞吐量优先和响应优先的垃圾收集器选择**

#### **40. 说一说你对环境变量classpath的理解？如果一个类不在classpath下，为什么会抛出ClassNotFoundException异常，如果在不改变这个类路径的前期下，怎样才能正确加载这个类？**

#### **41. 说一下强引用、软引用、弱引用、虚引用以及他们之间和gc的关系**

#### **42. JUC/并发相关**

JUC：即java.util.concurrent 包。

[http://raychase.iteye.com/blog/1998965](https://link.zhihu.com/?target=http%3A//raychase.iteye.com/blog/1998965)

***1） 并发容器\***

这些容器的关键方法大部分都实现了线程安全的功能，却不使用同步关键字(synchronized)。值得注意的是Queue接口本身定义的几个常用方法的区别，add方法和offer方法的区别在于超出容量限制时前者抛出异常，后者返回false；

remove方法和poll方法都从队列中拿掉元素并返回，但是他们的区别在于空队列下操作前者抛出异常，而后者返回null；

element方法和peek方法都返回队列顶端的元素，但是不把元素从队列中删掉，区别在于前者在空队列的时候抛出异常，后者返回null。

***2） 阻塞队列\***

BlockingQueue.class，阻塞队列接口

BlockingDeque.class，双端阻塞队列接口

ArrayBlockingQueue.class，阻塞队列，数组实现

LinkedBlockingDeque.class，阻塞双端队列，链表实现

LinkedBlockingQueue.class，阻塞队列，链表实现

DelayQueue.class，阻塞队列，并且元素是Delay的子类，保证元素在达到一定时间后才可以取得到

PriorityBlockingQueue.class，优先级阻塞队列

SynchronousQueue.class，同步队列，但是队列长度为0，生产者放入队列的操作会被阻塞，直到消费者过来取，所以这个队列根本不需要空间存放元素；有点像一个独木桥，一次只能一人通过

***3） 转移队列\***

TransferQueue.class，转移队列接口，生产者要等消费者消费的队列，生产者尝试把元素直接转移给消费者

LinkedTransferQueue.class，转移队列的链表实现，它比SynchronousQueue更快

***4） 其他容器\***

ConcurrentMap.class，并发Map的接口，定义了putIfAbsent(k,v)、remove(k,v)、replace(k,oldV,newV)、replace(k,v)这四个并发场景下特定的方法

ConcurrentHashMap.class，并发HashMap

ConcurrentNavigableMap.class，NavigableMap的实现类，返回最接近的一个元素

ConcurrentSkipListMap.class，它也是NavigableMap的实现类（要求元素之间可以比较），同时它比ConcurrentHashMap更加scalable——ConcurrentHashMap并不保证它的操作时间，并且你可以自己来调整它的load factor；但是ConcurrentSkipListMap可以保证O(log n)的性能，同时不能自己来调整它的并发参数，只有你确实需要快速的遍历操作，并且可以承受额外的插入开销的时候，才去使用它

ConcurrentSkipListSet.class，和上面类似，只不过map变成了set

CopyOnWriteArrayList.class，copy-on-write模式的array list，每当需要插入元素，不在原list上操作，而是会新建立一个list，适合读远远大于写并且写时间并苛刻的场景

CopyOnWriteArraySet.class，和上面类似，list变成set而已

***5） 同步设备\***

CountDownLatch.class，一个线程调用await方法以后，会阻塞地等待计数器被调用countDown直到变成0，功能上和下面的CyclicBarrier有点像

CyclicBarrier.class，也是计数等待，只不过它是利用await方法本身来实现计数器“+1”的操作，一旦计数器上显示的数字达到Barrier可以打破的界限，就会抛出BrokenBarrierException，线程就可以继续往下执行；请参见我写过的这篇文章《同步、异步转化和任务执行》中的Barrier模式

Semaphore.class，功能上很简单，acquire()和release()两个方法，一个尝试获取许可，一个释放许可，Semaphore构造方法提供了传入一个表示该信号量所具备的许可数量。

Exchanger.class，这个类的实例就像是两列飞驰的火车（线程）之间开了一个神奇的小窗口，通过小窗口（exchange方法）可以让两列火车安全地交换数据。

Phaser.class，功能上和第1、2个差不多，但是可以重用，且更加灵活，稍微有点复杂（CountDownLatch是不断-1，CyclicB

***6） 原子对象\***

weakCompareAndSet方法：compareAndSet方法很明确，但是这个是啥？根据JSR规范，调用weakCompareAndSet时并不能保证happen-before的一致性，因此允许存在重排序指令等等虚拟机优化导致这个操作失败（较弱的原子更新操作），但是从Java源代码看，它的实现其实和compareAndSet是一模一样的；

lazySet方法：延时设置变量值，这个等价于set方法，但是由于字段是volatile类型的，因此次字段的修改会比普通字段（非volatile字段）有稍微的性能损耗，所以如果不需要立即读取设置的新值，那么此方法就很有用。

AtomicBoolean.class

AtomicInteger.class

AtomicIntegerArray.class

AtomicIntegerFieldUpdater.class

AtomicLong.class

AtomicLongArray.class

AtomicLongFieldUpdater.class

AtomicMarkableReference.class，它是用来高效表述Object-boolean这样的对象标志位数据结构的，一个对象引用+一个bit标志位

AtomicReference.class

AtomicReferenceArray.class

AtomicReferenceFieldUpdater.class

AtomicStampedReference.class，它和前面的AtomicMarkableReference类似，但是它是用来高效表述Object-int这样的“对象+版本号”数据结构，特别用于解决ABA问题

***7） 锁\***

AbstractOwnableSynchronizer.class，这三个AbstractXXXSynchronizer都是为了创建锁和相关的同步器而提供的基础，锁，还有前面提到的同步设备都借用了它们的实现逻辑

AbstractQueuedLongSynchronizer.class，AbstractOwnableSynchronizer的子类，所有的同步状态都是用long变量来维护的，而不是int，在需要64位的属性来表示状态的时候会很有用

AbstractQueuedSynchronizer.class，为实现依赖于先进先出队列的阻塞锁和相关同步器（信号量、事件等等）提供的一个框架，它依靠int值来表示状态

Lock.class，Lock比synchronized关键字更灵活，而且在吞吐量大的时候效率更高，根据JSR-133的定义，它happens-before的语义和synchronized关键字效果是一模一样的，它唯一的缺点似乎是缺乏了从lock到finally块中unlock这样容易遗漏的固定使用搭配的约束，除了lock和unlock方法以外，还有这样两个值得注意的方法：

lockInterruptibly：如果当前线程没有被中断，就获取锁；否则抛出InterruptedException，并且清除中断

tryLock，只在锁空闲的时候才获取这个锁，否则返回false，所以它不会block代码的执行

ReadWriteLock.class，读写锁，读写分开，读锁是共享锁，写锁是独占锁；对于读-写都要保证严格的实时性和同步性的情况，并且读频率远远大过写，使用读写锁会比普通互斥锁有更好的性能。

ReentrantLock.class，可重入锁（lock行为可以嵌套，但是需要和unlock行为一一对应），有几点需要注意：

构造器支持传入一个表示是否是公平锁的boolean参数，公平锁保证一个阻塞的线程最终能够获得锁，因为是有序的，所以总是可以按照请求的顺序获得锁；不公平锁意味着后请求锁的线程可能在其前面排列的休眠线程恢复前拿到锁，这样就有可能提高并发的性能

还提供了一些监视锁状态的方法，比如isFair、isLocked、hasWaiters、getQueueLength等等

ReentrantReadWriteLock.class，可重入读写锁

Condition.class，使用锁的newCondition方法可以返回一个该锁的Condition对象，如果说锁对象是取代和增强了synchronized关键字的功能的话，那么Condition则是对象wait/notify/notifyAll方法的替代。在下面这个例子中，lock生成了两个condition，一个表示不满，一个表示不空；在put方法调用的时候，需要检查数组是不是已经满了，满了的话就得等待，直到“不满”这个condition被唤醒（notFull.await()）；在take方法调用的时候，需要检查数组是不是已经空了，如果空了就得等待，直到“不空”这个condition被唤醒（notEmpty.await()）：

***8） Fork-join框架\***

这是一个JDK7引入的并行框架，它把流程划分成fork（分解）+join（合并）两个步骤（怎么那么像MapReduce？），传统线程池来实现一个并行任务的时候，经常需要花费大量的时间去等待其他线程执行任务的完成，但是fork-join框架使用work stealing技术缓解了这个问题：



每个工作线程都有一个双端队列，当分给每个任务一个线程去执行的时候，这个任务会放到这个队列的头部；

当这个任务执行完毕，需要和另外一个任务的结果执行合并操作，可是那个任务却没有执行的时候，不会干等，而是把另一个任务放到队列的头部去，让它尽快执行；

当工作线程的队列为空，它会尝试从其他线程的队列尾部偷一个任务过来；

取得的任务可以被进一步分解。

ForkJoinPool.class，ForkJoin框架的任务池，ExecutorService的实现类

ForkJoinTask.class，Future的子类，框架任务的抽象

ForkJoinWorkerThread.class，工作线程

RecursiveTask.class，ForkJoinTask的实现类，compute方法有返回值，下文中有例子

RecursiveAction.class，ForkJoinTask的实现类，compute方法无返回值，只需要覆写compute方法，对于可继续分解的子任务，调用coInvoke方法完成（参数是RecursiveAction子类对象的可变数组）：

***9） 线程池执行器\***

#### **43. ThreadLocal用过么，原理是什么，用的时候要注意什么**

#### **44. 乐观锁 悲观锁**

悲观锁(Pessimistic Lock), 顾名思义，就是很悲观，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会block直到它拿到锁。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。Oracle 里面使用for update关键字

乐观锁(Optimistic Lock), 顾名思义，就是很乐观，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号等机制。乐观锁适用于多读的应用类型，这样可以提高吞吐量，像数据库如果提供类似于write_condition机制的其实都是提供的乐观锁。

两种锁各有优缺点，不可认为一种好于另一种，像乐观锁适用于写比较少的情况下，即冲突真的很少发生的时候，这样可以省去了锁的开销，加大了系统的整个吞吐量。但如果经常产生冲突，上层应用会不断的进行retry，这样反倒是降低了性能，所以这种情况下用悲观锁就比较合适。

#### **45. 可重入锁ReentrantLock**

如果线程A继续再次获得这个锁呢?比如一个方法是synchronized,递归调用自己,那么第一次已经获得了锁,第二次调用的时候还能进入吗? 直观上当然需要能进入.这就要求必须是可重入的.可重入锁又叫做递归锁。

#### **重入锁又叫递归锁：**

#### **递归调用是可以重复获取到锁的。但是其他方法无法获取到锁。如下代码：只打印reentry ok 使用true参数设置的是公平锁。**

```java
public static final ReentrantLock  rlock = new ReentrantLock(true);
	public static void reentry(){
		
		rlock.lock();
		try{
			Thread.sleep(1000);
			System.out.println("reentry ok");
			reentry();
		}catch(Exception e){
			e.printStackTrace();
		}finally {
			rlock.unlock();
		}
	}
	
	public static void getLock(){
		rlock.lock();
		System.out.println("getLock ok");
		rlock.unlock();
	}
	
	public static void main(String[] args) {
		ThreadPoolExecutor td = new ThreadPoolExecutor(2, 5, 500, 
				TimeUnit.MICROSECONDS, new ArrayBlockingQueue<Runnable>(5));
		td.submit(new Runnable() {
			
			@Override
			public void run() { 
				TheadPoolExecute.reentry();
				 System.out.println("m-->::"); 
			}
		});
		
	  td.submit(new Runnable() {
			
			@Override
			public void run() {
				TheadPoolExecute.getLock();
				 System.out.println("m-->::"); 
			}
		});
		 td.shutdown();
```



#### **46) Synchronized和Lock的区别**

![img](https://pic4.zhimg.com/80/v2-03fade817fea05b245d324c3b0b969ef_720w.jpg)

#### **47.synchronized 的原理，什么是自旋锁，偏向锁，轻量级锁，什么叫可重入锁，什么叫公平锁和非公平锁**

锁：[偏向锁，轻量级锁，自旋锁，重量级锁的详细介绍 - wade&luffy - 博客园](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/wade-luffy/p/5969418.html)

**1） 关键字synchroinized**

在编译的时候会生成

Monitorenter 和 monitorexit 指令，此指令用于代码块同步。

monitorenter ：

Each object is associated with a monitor. A monitor is locked if and only if it has an owner. The thread that executes monitorenter attempts to gain ownership of the monitor associated with objectref, as follows:

• If the entry count of the monitor associated with objectref is zero, the thread enters the monitor and sets its entry count to one. The thread is then the owner of the monitor.

• If the thread already owns the monitor associated with objectref, it reenters the monitor, incrementing its entry count.

• If another thread already owns the monitor associated with objectref, the thread blocks until the monitor's entry count is zero, then tries again to gain ownership.

这段话的大概意思为：

每个对象有一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：

1、如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。

2、如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1.

3.如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权。

monitorexit：　

The thread that executes monitorexit must be the owner of the monitor associated with the instance referenced by objectref.

The thread decrements the entry count of the monitor associated with objectref. If as a result the value of the entry count is zero, the thread exits the monitor and is no longer its owner. Other threads that are blocking to enter the monitor are allowed to attempt to do so.

这段话的大概意思为：

执行monitorexit的线程必须是objectref所对应的monitor的所有者。

指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。

　　通过这两段描述，我们应该能很清楚的看出Synchronized的实现原理，Synchronized的语义底层是通过一个monitor的对象来完成，其实wait/notify等方法也依赖于monitor对象，这就是为什么只有在同步的块或者方法中才能调用wait/notify等方法，否则会抛出java.lang.IllegalMonitorStateException的异常的原因。

***2） 锁的概念\***

线程的阻塞和唤醒需要CPU从用户态转为核心态，频繁的阻塞和唤醒对CPU来说是一件负担很重的工作。

Java SE1.6为了减少获得锁和释放锁所带来的性能消耗，引入了“偏向锁”和“轻量级锁”，所以在Java SE1.6里锁一共有四种状态，无锁状态，偏向锁状态，轻量级锁状态和重量级锁状态，它会随着竞争情况逐渐升级。锁可以升级但不能降级，意味着偏向锁升级成轻量级锁后不能降级成偏向锁。这种锁升级却不能降级的策略，目的是为了提高获得锁和释放锁的效率。

> ***偏向锁***

大多数情况下锁不仅不存在多线程竞争，而且总是由同一线程多次获得。偏向锁的目的是在某个线程获得锁之后，消除这个线程锁重入（CAS）的开销，看起来让这个线程得到了偏护。另外，JVM对那种会有多线程加锁，但不存在锁竞争的情况也做了优化，听起来比较拗口，但在现实应用中确实是可能出现这种情况，因为线程之前除了互斥之外也可能发生同步关系，被同步的两个线程（一前一后）对共享对象锁的竞争很可能是没有冲突的。对这种情况，JVM用一个epoch表示一个偏向锁的时间戳（真实地生成一个时间戳代价还是蛮大的，因此这里应当理解为一种类似时间戳的identifier）

> 偏向锁的获取

当一个线程访问同步块并获取锁时，会在对象头和栈帧中的锁记录里存储锁偏向的线程ID，以后该线程在进入和退出同步块时不需要花费CAS操作来加锁和解锁，而只需简单的测试一下对象头的Mark Word里是否存储着指向当前线程的偏向锁，如果测试成功，表示线程已经获得了锁，如果测试失败，则需要再测试下Mark Word中偏向锁的标识是否设置成1（表示当前是偏向锁），如果没有设置，则使用CAS竞争锁，如果设置了，则尝试使用CAS将对象头的偏向锁指向当前线程。

> 偏向锁的撤销

偏向锁使用了一种等到竞争出现才释放锁的机制，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。偏向锁的撤销，需要等待全局安全点（在这个时间点上没有字节码正在执行），它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否活着，如果线程不处于活动状态，则将对象头设置成无锁状态，如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的锁记录，栈中的锁记录和对象头的Mark Word，要么重新偏向于其他线程，要么恢复到无锁或者标记对象不适合作为偏向锁，最后唤醒暂停的线程。

> 偏向锁的设置

关闭偏向锁：偏向锁在Java 6和Java 7里是默认启用的，但是它在应用程序启动几秒钟之后才激活，如有必要可以使用JVM参数来关闭延迟-XX：BiasedLockingStartupDelay = 0。如果你确定自己应用程序里所有的锁通常情况下处于竞争状态，可以通过JVM参数关闭偏向锁-XX:-UseBiasedLocking=false，那么默认会进入轻量级锁状态。

> 自旋锁

线程的阻塞和唤醒需要CPU从用户态转为核心态，频繁的阻塞和唤醒对CPU来说是一件负担很重的工作。同时我们可以发现，很多对象锁的锁定状态只会持续很短的一段时间，例如整数的自加操作，在很短的时间内阻塞并唤醒线程显然不值得，为此引入了自旋锁。

所谓“自旋”，就是让线程去执行一个无意义的循环，循环结束后再去重新竞争锁，如果竞争不到继续循环，循环过程中线程会一直处于running状态，但是基于JVM的线程调度，会出让时间片，所以其他线程依旧有申请锁和释放锁的机会。

自旋锁省去了阻塞锁的时间空间（队列的维护等）开销，但是长时间自旋就变成了“忙式等待”，忙式等待显然还不如阻塞锁。所以自旋的次数一般控制在一个范围内，例如10,100等，在超出这个范围后，自旋锁会升级为阻塞锁。

> 轻量级锁
> 加锁

线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，则自旋获取锁，当自旋获取锁仍然失败时，表示存在其他线程竞争锁(两条或两条以上的线程竞争同一个锁)，则轻量级锁会膨胀成重量级锁。

> 解锁

轻量级解锁时，会使用原子的CAS操作来将Displaced Mark Word替换回到对象头，如果成功，则表示同步过程已完成。如果失败，表示有其他线程尝试过获取该锁，则要在释放锁的同时唤醒被挂起的线程。

> **重量级锁**

重量锁在JVM中又叫对象监视器（Monitor），它很像C中的Mutex，除了具备Mutex(0|1)互斥的功能，它还负责实现了Semaphore(信号量)的功能，也就是说它至少包含一个竞争锁的队列，和一个信号阻塞队列（wait队列），前者负责做互斥，后一个用于做线程同步。

锁的优缺点对比

![img](https://pic4.zhimg.com/80/v2-03fade817fea05b245d324c3b0b969ef_720w.jpg)

#### **48. concurrenthashmap具体实现及其原理，jdk8下的改版**

#### **49. 用过哪些原子类，他们的参数以及原理是什么**

#### **50. cas是什么，他会产生什么问题（ABA问题的解决，如加入修改次数、版本号）**

#### **51. 如果让你实现一个并发安全的链表，你会怎么做**

#### **52. 简述ConcurrentLinkedQueue和LinkedBlockingQueue的用处和不同之处**

#### **前者使用了CAS实现，后者使用读写分离的ReenterLock 实现。**

#### **53. 简述AQS的实现原理**

#### **54. countdowlatch和cyclicbarrier的用法，以及相互之间的差别?**

#### **55. concurrent包中使用过哪些类？分别说说使用在什么场景？为什么要使用？**

#### **56. LockSupport工具**

#### **57. Condition接口及其实现原理**

#### **58. Fork/Join框架的理解**

#### **59. jdk8的parallelStream的理解**

#### **60. 分段锁的原理,锁力度减小的思考**



## 框架容器篇



**61. Spring**

#### **62. Spring AOP与IOC的实现原理**

#### **63. IOC**

控制反转。正常创建对象会在使用的时候主动创建伊依赖的对象。IOC的思想是由容器去控制对象的创建和依赖的注入。称之为反转的思想。

***1） 谁控制谁？由IOC容器控制对象，控制什么？***：主要控制外部资源。

***2） 哪些方面反转了？\***容器帮我们查找注入的对象，对象被动接受依赖对象，所以是反转，依赖对象的获取被反转了。

其实IoC对编程带来的最大改变不是从代码上，而是从思想上，发生了“主从换位”的变化。应用程序原本是老大，要获取什么资源都是主动出击，但是在IoC/DI思想中，应用程序就变成被动的了，被动的等待IoC容器来创建并注入它所需要的资源了。

　　IoC很好的体现了面向对象设计法则之一—— 好莱坞法则：“别找我们，我们找你”；即由IoC容器帮对象找相应的依赖对象并注入，而不是由对象主动去找

***IOC：控制反转\***

由Ioc容器帮我们找到相应的依赖对象并注入。

***DI：依赖注入\***

对象的引用关系不在由自己控制，而是由spring容器帮我们完成。

***64. Spring的beanFactory和factoryBean的区别\***

BeanFactory，以Factory结尾，表示它是一个工厂类(接口)，用于管理Bean的一个工厂。在Spring中，BeanFactory是IOC容器的核心接口，它的职责包括：实例化、定位、配置应用程序中的对象及建立这些对象间的依赖

以Bean结尾，表示它是一个Bean，不同于普通Bean的是：它是实现了FactoryBean<T>接口的Bean，根据该Bean的ID从BeanFactory中获取的实际上是FactoryBean的getObject()返回的对象，而不是FactoryBean本身，如果要获取FactoryBean对象，请在id前面加一个&符号来获取。

***65. 为什么CGlib方式可以对接口实现代理？\***

***66. RMI与代理模式\***

***67. Spring的事务隔离级别，实现原理\***

***1） 事务的特性\***

ACID（原子性、一致性、隔离性、持久性）

***2） 核心接口\***

![img](https://pic1.zhimg.com/80/v2-f057818e7a2269ed592b3ddb57c5f73c_720w.jpg)

各个平台实现：

#### *transactionManager 接口*

jdbc平台：

```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource" />
    </bean>
```

Hibernate：

```xml
    <bean id="transactionManager" class="org.springframework.orm.hibernate3.HibernateTransactionManager">
        <property name="sessionFactory" ref="sessionFactory" />
    </bean>
```

Jpa：

```xml
   <bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
        <property name="sessionFactory" ref="sessionFactory" />
    </bean>
```

***3） 事务的特性\***

![img](https://pic1.zhimg.com/80/v2-86c043d94503dc4f9412feb731d4be64_720w.jpg)

TransactionDefinition：

```java
public interface TransactionDefinition {
    int getPropagationBehavior(); // 返回事务的传播行为
    int getIsolationLevel(); // 返回事务的隔离级别，事务管理器根据它来控制另外一个事务可以看到本事务内的哪些数据
    int getTimeout();  // 返回事务必须在多少秒内完成
    boolean isReadOnly(); // 事务是否只读，事务管理器能够根据这个返回值进行优化，确保事务是只读的
}
```

详细见：

[Spring事务管理（详解+实例） - CSDN博客](https://link.zhihu.com/?target=http%3A//blog.csdn.net/trigl/article/details/50968079)

***4） Spring如何在开启事务后获取到同一个数据库链接***

DataSourceUtils

[Spring 是如何保证事务获取同一个Connection的](https://link.zhihu.com/?target=https%3A//www.cnblogs.com/just4me/p/6428077.html)

基本思路为：将当前获取到的数据库链接生产一个对象并放入ThreadLocal 中，静态变量：

```java
private static final ThreadLocal<Map<Object, Object>> resources =
      new NamedThreadLocal<Map<Object, Object>>("Transactional resources");
```

TransactionSynchronizationManager内部用ThreadLocal对象存储资源，ThreadLocal存储的为DataSource生成的actualKey为key值和ConnectionHolder作为value值封装成的Map。



在某个线程第一次调用时候，封装Map资源为：key值为DataSource生成actualKey【Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);】value值为DataSource获得的Connection对象封装后的ConnectionHolder。

文档中key的生产：

```java
/**
	 * Bind the given resource for the given key to the current thread.
	 * @param key the key to bind the value to (usually the resource factory)
	 * @param value the value to bind (usually the active resource object)
	 * @throws IllegalStateException if there is already a value bound to the thread
	 * @see ResourceTransactionManager#getResourceFactory()
	 */
public static void bindResource(Object key, Object value) throws IllegalStateException {
		Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
		Assert.notNull(value, "Value must not be null");
		Map<Object, Object> map = resources.get();
		// set ThreadLocal Map if none found
		if (map == null) {
			map = new HashMap<>();
			resources.set(map);
		}
		Object oldValue = map.put(actualKey, value);
		// Transparently suppress a ResourceHolder that was marked as void...
		if (oldValue instanceof ResourceHolder && ((ResourceHolder) oldValue).isVoid()) {
			oldValue = null;
		}
		if (oldValue != null) {
			throw new IllegalStateException("Already value [" + oldValue + "] for key [" +
					actualKey + "] bound to thread [" + Thread.currentThread().getName() + "]");
		}
		if (logger.isTraceEnabled()) {
			logger.trace("Bound value [" + value + "] for key [" + actualKey + "] to thread [" +
					Thread.currentThread().getName() + "]");
		}
	}
```



***5） 隔离级别***

***脏读（Dirty reads）***——脏读发生在一个事务读取了另一个事务改写但尚未提交的数据时。如果改写在稍后被回滚了，那么第一个事务获取的数据就是无效的。

***不可重复读（Nonrepeatable read）\***——不可重复读发生在一个事务执行相同的查询两次或两次以上，但是每次都得到不同的数据时。这通常是因为另一个并发事务在两次查询期间进行了更新。

***幻读（Phantom read）\***——幻读与不可重复读类似。它发生在一个事务（T1）读取了几行 数据，接着另一个并发事务（T2）插入了一些数据时。在随后的查询中，第一个事务（T1）就会发现多了一些原本不存在的记录。

不可重复读的重点是修改，同样的条件, 你读取过的数据, 再次读取出来发现值不一样了

幻读的重点在于新增或者删除，同样的条件, 第1次和第2次读出来的记录数不一样

***6） 编程式事务和声明式事务***

Spring提供两种方式的编程式事务管理，分别是：使用TransactionTemplate和直接使用PlatformTransactionManager

#### **68. 对Spring的理解，非单例注入的原理？它的生命周期？循环注入的原理，aop的实现原理，说说aop中的几个术语，它们是怎么相互工作的？**

#### **69. Mybatis的底层实现原理**

#### **70. MVC框架原理，他们都是怎么做url路由的**

#### **71. spring boot特性，优势，适用场景等**

#### **72. quartz和timer对比**

#### **73. spring的controller是单例还是多例，怎么保证并发的安全**



## 分布式相关

此部分内容繁多，只作为一个索引，相关内容请网络自行查询

#### **74. Dubbo的底层实现原理和机制**

#### **75. 描述一个服务从发布到被消费的详细过程**

**76. 分布式系统怎么做服务治理**

**77. 接口的幂等性的概念**

**78. 消息中间件如何解决消息丢失问题**

**79. Dubbo的服务请求失败怎么处理**

**80. 重连机制会不会造成错误**

**81. 对分布式事务的理解**

**82. 如何实现负载均衡，有哪些算法可以实现？**

**83. Zookeeper的用途，选举的原理是什么？**

**84. 数据的垂直拆分水平拆分。**

**85. zookeeper原理和适用场景**

**86. zookeeper watch机制**

**87. redis/zk节点宕机如何处理**

**88. 分布式集群下如何做到唯一序列号**

**89. 如何做一个分布式锁**

**90. 用过哪些MQ，怎么用的，和其他mq比较有什么优缺点，MQ的连接是线程安全的吗**

**91. MQ系统的数据如何保证不丢失**

**92. 列举出你能想到的数据库分库分表策略；分库分表后，如何解决全表查询的问题。**

**93. 算法&数据结构&设计模式**

**94. 海量url去重类问题（布隆过滤器）**

**95. 数组和链表数据结构描述，各自的时间复杂度**

**96. 二叉树遍历**

**97. 快速排序**

**98. BTree相关的操作**

**99. 在工作中遇到过哪些设计模式，是如何应用的**

**100. hash算法的有哪几种，优缺点，使用场景**

**101. 什么是一致性hash**

**102. paxos算法**

**103. 在装饰器模式和代理模式之间，你如何抉择，请结合自身实际情况聊聊**

**104. 代码重构的步骤和原因，如果理解重构到模式？**

**105. 数据库**

**106. MySQL InnoDB存储的文件结构**

**107. 索引树是如何维护的？**

**108. 数据库自增主键可能的问题**

**109. MySQL的几种优化**

**110. mysql索引为什么使用B+树**

**111. 数据库锁表的相关处理**

**112. 索引失效场景**

**113. 高并发下如何做到安全的修改同一行数据，乐观锁和悲观锁是什么，INNODB的行级锁有哪2种，解释其含义**

**114. 数据库会死锁吗，举一个死锁的例子，mysql怎么解决死锁**

**115. Redis&缓存相关**

**116. Redis的并发竞争问题如何解决了解Redis事务的CAS操作吗**

**117. 缓存机器增删如何对系统影响最小，一致性哈希的实现**

**118. Redis持久化的几种方式，优缺点是什么，怎么实现的**

**119. Redis的缓存失效策略**

**120. 缓存穿透的解决办法**

**121. redis集群，高可用，原理**

**122. mySQL里有2000w数据，redis中只存20w的数据，如何保证redis中的数据都是热点数据**

**123. 用Redis和任意语言实现一段恶意登录保护的代码，限制1小时内每用户Id最多只能登录5次**

**124. redis的数据淘汰策略**

**125. 网络相关**

**126. http1.0和http1.1有什么区别**

**127. TCP/IP协议**

**128. TCP三次握手和四次挥手的流程，为什么断开连接要4次,如果握手只有两次，会出现什么**

**129. TIME_WAIT和CLOSE_WAIT的区别**

**130. 说说你知道的几种HTTP响应码**

**131. 当你用浏览器打开一个链接的时候，计算机做了哪些工作步骤**

**132. TCP/IP如何保证可靠性，数据包有哪些数据组成**

**133. 长连接与短连接**

**134. Http请求get和post的区别以及数据包格式**

**135. 简述tcp建立连接3次握手，和断开连接4次握手的过程；关闭连接时，出现TIMEWAIT过多是由什么原因引起，是出现在主动断开方还是被动断开方。**

**136. 其他**

**137. maven解决依赖冲突,快照版和发行版的区别**

**138. Linux下IO模型有几种，各自的含义是什么**

**139. 实际场景问题，海量登录日志如何排序和处理SQL操作，主要是索引和聚合函数的应用**

**140. 实际场景问题解决，典型的TOP K问题**

**141. 线上bug处理流程**

**142. 如何从线上日志发现问题**

**143. linux利用哪些命令，查找哪里出了问题（例如io密集任务，cpu过度）**

**144. 场景问题，有一个第三方接口，有很多个线程去调用获取数据，现在规定每秒钟最多有10个线程同时调用它，如何做到。**

**145. 用三个线程按顺序循环打印abc三个字母，比如abcabcabc。**

**146. 常见的缓存策略有哪些，你们项目中用到了什么缓存系统，如何设计的**

**147. 设计一个秒杀系统，30分钟没付款就自动关闭交易（并发会很高）**

**148. 请列出你所了解的性能测试工具**

**149. 后台系统怎么防止请求重复提交？**

**150. 有多个相同的接口，我想客户端同时请求，然后只需要在第一个请求返回结果的时候返回给客户端**

**151. 全局唯一ID生成方案**

**1） 数据库auto_increment （批量生成）**

**2） Zk自增长临时节点**

**3） Redis inc方法**

**4） Snowflake算法**

**152. 负载均衡算法**

- ***轮询算法\***
- ***IP hash算法\***
- ***权重算法\***
- ***最少连接路由\***

**153. 秒杀业务**

- **接入层提供限流、业务降级，防止应用雪崩效应**
- **应用层使用缓存缓存热点数据、临界资源**
- **限制同UID的访问次数**

**154. 高并发**

**指标：**

- **response time（响应时长）**
- **Throughput吞吐量**
- **QPS （query per second）**
- **并发用户数**

## 架构相关



**155. 提升性能措施**

***1） 垂直扩展（Scale Up）***

提升硬件配置方法，速度快应用层无需改进。单实成本高。单机性能有限。

***2） 水平扩展（Scale Out）\***

水平扩展对应用架构有要求如：session同步、数据数据存储等，弹性大。

（1）反向代理层可以通过“DNS轮询”的方式来进行水平扩展；

（2）站点层可以通过nginx来进行水平扩展；

（3）服务层可以通过服务连接池来进行水平扩展；

（4）数据库可以按照数据范围，或者数据哈希的方式来进行水平扩展；

**156. 高可用（High Availability**）

***1） 冗余设计（集群化）***

***2） 故障自动转移***

整个互联网分层系统架构的高可用，又是通过每一层的冗余+自动故障转移来综合实现的，具体的：

*（1）【客户端层】到【反向代理层】的高可用，是通过反向代理层的冗余实现的，常见实践是keepalived + virtual IP自动故障转移*

*（2）【反向代理层】到【站点层】的高可用，是通过站点层的冗余实现的，常见实践是nginx与web-server之间的存活性探测与自动故障转移*

*（3）【站点层】到【服务层】的高可用，是通过服务层的冗余实现的，常见实践是通过service-connection-pool来保证自动故障转移*

*（4）【服务层】到【缓存层】的高可用，是通过缓存数据的冗余实现的，常见实践是缓存客户端双读双写，或者利用缓存集群的主从数据同步与sentinel保活与自动故障转移；更多的业务场景，对缓存没有高可用要求，可以使用缓存服务化来对调用方屏蔽底层复杂性*

*（5）【服务层】到【数据库“读”】的高可用，是通过读库的冗余实现的，常见实践是通过db-connection-pool来保证自动故障转移*

*（6）【服务层】到【数据库“写”】的高可用，是通过写库的冗余实现的，常见实践是keepalived + virtual IP自动故障转移*

#### **157. 配置中心**

***解决什么问题？***

配置导致系统耦合，架构反向依赖。

***什么痛点？\***

上游痛：扩容的是下游，改配置重启的是上游

下游痛：不知道谁依赖于自己

**配置架构如何演进？**

***一、配置私藏\***

***二、全局配置文件\***

***三、配置中心\***

#### **158. Session同步**

保证session一致性的架构设计常见方法：

*•session同步法：多台web-server相互同步数据*

*•客户端存储法：一个用户只存储自己的数据*

*•反向代理hash一致性：四层hash和七层hash都可以做，保证一个用户的请求落在一台web-server上*

*•后端统一存储：web-server重启和扩容，session也不会丢失*

*对于方案3和方案4，个人建议推荐后者：*

*•web层、service层无状态是大规模分布式系统设计原则之一，session属于状态，不宜放在web层*

*•让专业的软件做专业的事情，web-server存session？还是让cache去做这样的事情吧*