---
title: Striped64与LongAdder源码阅读
date: 2021-03-04 09:45:52
tags:
- 源码阅读
---

<!--将该代码放入博客模板的head中即可-->
<script type="text/x-mathjax-config">
MathJax.Hub.Config({
tex2jax: {
  inlineMath: [['$','$'], ['\\(','\\)']],
  processEscapes: true
  }
});
</script>
<!--latex数学显示公式-->
<script type="text/javascript" src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML"></script>

[toc]

AtomicLong内部是一个volatile long型变量，由多个线程对这个变量进行CAS操作。多个线程同时对一个变量进行CAS操作，在高并发的场景下仍不够快，如果再要提高性能，该怎么做呢？把一个变量拆成多份，变为多个变量，有些类似于ConcurrentHashMap 的分段锁的例子。

把一个Long型拆成一个base变量外加多个Cell，每个Cell包装了一个Long型变量。当多个线程并发累加的时候，如果并发度低，就直接加到base变量上；如果并发度高，冲突大，平摊到这些Cell上。在最后取值的时候，再把base和这些Cell求sum运算。

## Striped64类

```java
package java.util.concurrent.atomic;
import java.util.function.LongBinaryOperator;
import java.util.function.DoubleBinaryOperator;
import java.util.concurrent.ThreadLocalRandom;

@SuppressWarnings("serial")
abstract class Striped64 extends Number {// 抽象类

    // 把一个Long型拆成一个base变量外加多个Cell，每个Cell包装了一个Long型变量。
    // 当多个线程并发累加的时候，如果并发度低，就直接加到base变量上；
    // 如果并发度高，冲突大，平摊到这些Cell上。
    // 在最后取值的时候，再把base和这些Cell求sum运算。
    @sun.misc.Contended static final class Cell {
        // @sun.misc.Contended伪共享与缓存行填充
        // 在64位x86架构中，缓存行是64字节，也就是8个Long型的大小。
        // 这也意味着当缓存失效，要刷新到主内存的时候，最少要刷新64字节。
        // 当X,Y,Z在同一个cache line 时,X更新后,由于Cache Line是数据交换的基本单位，无法只失效X，
        // 要失效就会失效整行的Cache Line，这会导致Y、Z变量的缓存也失效。
        // 虽然只修改了X变量，本应该只失效X变量的缓存，但Y、Z变量也随之失效。
        // Y、Z变量的数据没有修改，本应该很好地被CPU1和CPU2共享，却没做到，这就是所谓的“伪共享问题”
        //要解决这个问题，需要用到所谓的“缓存行填充”，分别在X、Y、Z后面加上7个无用的Long型，
        // 填充整个缓存行，让X、Y、Z处在三行不同的缓存行中
        // 而在JDK 8中，就不需要写这种晦涩的代码了，只需声明一个@sun.misc.Contended即可

        volatile long value;
        Cell(long x) { value = x; }
        final boolean cas(long cmp, long val) {
            return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
        }

        // Unsafe mechanics
        private static final sun.misc.Unsafe UNSAFE;
        private static final long valueOffset;
        static {
            try {
                UNSAFE = sun.misc.Unsafe.getUnsafe();
                Class<?> ak = Cell.class;
                valueOffset = UNSAFE.objectFieldOffset
                    (ak.getDeclaredField("value"));
            } catch (Exception e) {
                throw new Error(e);
            }
        }
    }

    // 获得CPU核心的数量
    static final int NCPU = Runtime.getRuntime().availableProcessors();

    // 内部的cell数组
    transient volatile Cell[] cells;

    // 并发度低的时候,将变量的值加到base变量上
    transient volatile long base;

    // 如果Cell数组还没有被创建，那么就去获取cellBusy这个共享变量（相当于锁，但是更为轻量级），
    // 如果获取成功，则初始化Cell数组，初始容量为2，初始化完成之后将x保证成一个Cell，
    // 哈希计算之后分散到相应的index上。
    // 如果获取cellBusy失败，那么会试图将x累计到base上，更新失败会重新尝试直到成功。
    transient volatile int cellsBusy;

    Striped64() {
    }

    // BASE的CAS实现
    final boolean casBase(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
    }

    // 获取当前CellBusy的锁,不可重入锁
    final boolean casCellsBusy() {
        return UNSAFE.compareAndSwapInt(this, CELLSBUSY, 0, 1);
    }
    // 计算哈希值
    // getProbe（）为该线程生成一个随机数，用该随机数对数组的长度取模。
    static final int getProbe() {
        return UNSAFE.getInt(Thread.currentThread(), PROBE);
    }

    // 计算哈希值,与ThreadLocalRandom类中的方法相同
    static final int advanceProbe(int probe) {
        probe ^= probe << 13;   // xorshift
        probe ^= probe >>> 17;
        probe ^= probe << 5;
        UNSAFE.putInt(Thread.currentThread(), PROBE, probe);
        return probe;
    }
```

### longAccumulate()方法

- longAccumulate会根据当前线程来计算一个哈希值，然后根据算法(hashCode & (length - 1))来达到取模的效果以定位到该线程被分散到的Cell数组中的位置

- 如果Cell数组还没有被创建，那么就去获取cellBusy这个共享变量（相当于锁，但是更为轻量级），如果获取成功，则初始化Cell数组，初始容量为2，初始化完成之后将x保证成一个Cell，哈希计算之后分散到相应的index上。如果获取cellBusy失败，那么会试图将x累计到base上，更新失败会重新尝试直到成功。

- 如果Cell数组以及被初始化过了，那么就根据线程的哈希值分散到一个Cell数组元素上，获取这个位置上的Cell并且赋值给变量a，这个a很重要，如果a为null，说明该位置还没有被初始化，那么就初始化，当然在初始化之前需要竞争cellBusy变量。

- 如果Cell数组的大小已经最大了（CPU的数量），那么就需要重新计算哈希，来重新分散当前线程到另外一个Cell位置上再走一遍该方法的逻辑，否则就需要对Cell数组进行扩容，然后将原来的计数内容迁移过去。这里面需要注意的是，因为Cell里面保存的是计数值，所以在扩容之后没有必要做其他的处理，直接根据index将旧的Cell数组内容直接复制到新的Cell数组中就可以了。


```java
    final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h;
        // 计算哈希值
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        // 自旋操作
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            // 判断as是否完成初始化
            if ((as = cells) != null && (n = as.length) > 0) {
            // Cell[]数组的大小始终是2的整数次方，在运行中会不断扩容，每次扩容都是增长2倍。下面代码中的as[getProbe（）&m]其实就是对数组的大小取模。因为m=as.length–1，getProbe（）为该线程生成一个随机数，用该随机数对数组的长度取模。因为数组长度是2的整数次方，所以可以用&操作来优化取模运算。
                // 如果目标位置处的元素为空
                if ((a = as[(n - 1) & h]) == null) {
                    // 判断锁是否处于空闲状态
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        // 创建一个新的Cell
                        Cell r = new Cell(x);   // Optimistically create
                        // 尝试获取锁,因为有多多个线程竞争当前的锁,因此要使用CAS操作
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    // 将新建的cell放到index为(m - 1) & h的位置
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                // 释放锁,因为只有当前的线程写这个元素,因此无需CAS操作
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    // 如果锁不是在空闲状态,重新hash看其他位置
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
                    break;
                else if (n >= NCPU || cells != as)
                    // 如果Cell数组的大小已经最大了（CPU的数量），那么就需要重新计算哈希
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 && casCellsBusy()) {// 获取锁
                    try {
                        if (cells == as) {      // Expand table unless stale
                            // 对表进行扩容,扩容的大小是原先的两倍
                            Cell[] rs = new Cell[n << 1];
                            // 复制旧元素到新的表中
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];

                            // 将指针指向新的表
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                // 计算哈希值
                h = advanceProbe(h);
            }
            // 如果cells表为空并且如果Cell数组还没有被创建，那么就去获取cellBusy这个共享变量（相当于锁，但是更为轻量级）
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                // 获取成功
                // 初始化完成之后将x保证成一个Cell，哈希计算之后分散到相应的index上。
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        // 初始化Cell数组，初始容量为2
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(x);
                        cells = rs;
                        init = true;
                    }
                } finally {
                    // 释放锁
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }// 如果获取cellBusy失败，那么会试图将x累计到base上，更新失败会重新尝试直到成功。
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                break;                          // Fall back on using base
        }
    }
```
>注意: Cell[]数组的大小始终是2的整数次方，在运行中会不断扩容，每次扩容都是增长2倍。

```java
    /**
     * Same as longAccumulate, but injecting long/double conversions
     * in too many places to sensibly merge with long version, given
     * the low-overhead requirements of this class. So must instead be
     * maintained by copy/paste/adapt.
     */
    final void doubleAccumulate(double x, DoubleBinaryOperator fn,
                                boolean wasUncontended) {
        int h;
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization
            h = getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            if ((as = cells) != null && (n = as.length) > 0) {
                if ((a = as[(n - 1) & h]) == null) {
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(Double.doubleToRawLongBits(x));
                        if (cellsBusy == 0 && casCellsBusy()) {
                            boolean created = false;
                            try {               // Recheck under lock
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                else if (a.cas(v = a.value,
                               ((fn == null) ?
                                Double.doubleToRawLongBits
                                (Double.longBitsToDouble(v) + x) :
                                Double.doubleToRawLongBits
                                (fn.applyAsDouble
                                 (Double.longBitsToDouble(v), x)))))
                    break;
                else if (n >= NCPU || cells != as)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 && casCellsBusy()) {
                    try {
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
                h = advanceProbe(h);
            }
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table
                    if (cells == as) {
                        Cell[] rs = new Cell[2];
                        rs[h & 1] = new Cell(Double.doubleToRawLongBits(x));
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            else if (casBase(v = base,
                             ((fn == null) ?
                              Double.doubleToRawLongBits
                              (Double.longBitsToDouble(v) + x) :
                              Double.doubleToRawLongBits
                              (fn.applyAsDouble
                               (Double.longBitsToDouble(v), x)))))
                break;                          // Fall back on using base
        }
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long BASE;
    private static final long CELLSBUSY;
    private static final long PROBE;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> sk = Striped64.class;
            BASE = UNSAFE.objectFieldOffset
                (sk.getDeclaredField("base"));
            CELLSBUSY = UNSAFE.objectFieldOffset
                (sk.getDeclaredField("cellsBusy"));
            Class<?> tk = Thread.class;
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomProbe"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }

}

```
## LongAdder类

在LongAdder开篇的注释中，把它和AtomicLong 的使用场景做了比较。它适合高并发的统计场景，而不适合要对某个Long 型变量进行严格同步的场景。
```java
package java.util.concurrent.atomic;
import java.io.Serializable;

public class LongAdder extends Striped64 implements Serializable {
    // 继承Striped64类,这个类要和Striped64结合起来看
    private static final long serialVersionUID = 7249069246863182397L;

    public LongAdder() {
    }
```

当一个线程调用add（x）的时候，首先会尝试使用casBase把x加到base变量上。如果不成功，则再用a.cas（..）函数尝试把x 加到Cell数组的某个元素上。如果还不成功，最后再调用longAccumulate（..）函数。

```java
    public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        // 如果cell不为空,或者base不可用(先尝试的base,base不行再去写cell)
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            // 是否存在争夺
            boolean uncontended = true;
            // as==null 没有初始化
            // as.length==0没有初始化
            // as[index]处的元素为空,需要新建一个cell
            // 否则加上这个值x
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended = a.cas(v = a.value, v + x)))
                longAccumulate(x, null, uncontended);
        }
    }

    public void increment() {
        add(1L);
    }

    public void decrement() {
        add(-1L);
    }
    // 对表中每个不为空的元素求和
    public long sum() {
        Cell[] as = cells; Cell a;
        long sum = base;
        if (as != null) {
            // 累加表中的所有非空项目
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }
    // 对表中所有不为空的项目置0
    public void reset() {
        Cell[] as = cells; Cell a;
        base = 0L;
        if (as != null) {
            // 所有的元素置0
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    a.value = 0L;
            }
        }
    }

    // 计算sum之后再置零
    public long sumThenReset() {
        Cell[] as = cells; Cell a;
        long sum = base;
        base = 0L;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null) {
                    sum += a.value;
                    a.value = 0L;
                }
            }
        }
        return sum;
    }


    public String toString() {
        return Long.toString(sum());
    }


    public long longValue() {
        return sum();
    }

    public int intValue() {
        return (int)sum();
    }

    public float floatValue() {
        return (float)sum();
    }

    public double doubleValue() {
        return (double)sum();
    }

    // 序列化代理内部类
    private static class SerializationProxy implements Serializable {
        private static final long serialVersionUID = 7249069246863182397L;

        // 当前sum的值
        private final long value;

        SerializationProxy(LongAdder a) {
            value = a.sum();
        }

        // 这里返回base的值
        private Object readResolve() {
            LongAdder a = new LongAdder();
            a.base = value;
            return a;
        }
    }
    private Object writeReplace() {
        return new SerializationProxy(this);
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.InvalidObjectException {
        throw new java.io.InvalidObjectException("Proxy required");
    }

}

```

## LongAccumulator类

LongAdder只能进行累加操作，并且初始值默认为0；LongAccumulator可以自己定义一个二元操作符，并且可以传入一个初始值。

**自定义二元操作符的接口**
```java
@FunctionalInterface
public interface LongBinaryOperator {
    long applyAsLong(long left, long right);
}
```
```java
public class LongAccumulator extends Striped64 implements Serializable {
    private static final long serialVersionUID = 7249069246863182397L;
    // 自己定义一个二元操作符
    private final LongBinaryOperator function;
    // 初始值
    private final long identity;
    // 传入自定义的二元操作符与初始值
    public LongAccumulator(LongBinaryOperator accumulatorFunction,
                           long identity) {
        this.function = accumulatorFunction;
        base = this.identity = identity;
    }

    public void accumulate(long x) {
        Cell[] as; long b, v, r; int m; Cell a;
        if ((as = cells) != null ||
            (r = function.applyAsLong(b = base, x)) != b && !casBase(b, r)) {
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[getProbe() & m]) == null ||
                !(uncontended =
                  (r = function.applyAsLong(v = a.value, x)) == v ||
                  a.cas(v, r)))
                longAccumulate(x, function, uncontended);
        }
    }

    // 获取所有的经自定义运算后的结果
    public long get() {
        Cell[] as = cells; Cell a;
        long result = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    result = function.applyAsLong(result, a.value);
            }
        }
        return result;
    }

    // 重置所有不为空的表项目中的值为初始值
    public void reset() {
        Cell[] as = cells; Cell a;
        base = identity;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    a.value = identity;
            }
        }
    }

    // 先获取所有的经自定义运算后的结果,再重置值
    public long getThenReset() {
        Cell[] as = cells; Cell a;
        long result = base;
        base = identity;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null) {
                    long v = a.value;
                    a.value = identity;
                    result = function.applyAsLong(result, v);
                }
            }
        }
        return result;
    }

    public String toString() {
        return Long.toString(get());
    }

    public long longValue() {
        return get();
    }

    public int intValue() {
        return (int)get();
    }

    public float floatValue() {
        return (float)get();
    }

    public double doubleValue() {
        return (double)get();
    }
    // 序列化与反序列化相关
    private static class SerializationProxy implements Serializable {
        private static final long serialVersionUID = 7249069246863182397L;
        private final long value;
        private final LongBinaryOperator function;
        private final long identity;

        SerializationProxy(LongAccumulator a) {
            function = a.function;
            identity = a.identity;
            value = a.get();
        }
        // 返回base
        private Object readResolve() {
            LongAccumulator a = new LongAccumulator(function, identity);
            a.base = value;
            return a;
        }
    }

    private Object writeReplace() {
        return new SerializationProxy(this);
    }
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.InvalidObjectException {
        throw new java.io.InvalidObjectException("Proxy required");
    }

}

```


> 参考文献
>
> 作者：一字马胡
> 链接：https://www.jianshu.com/p/30d328e9353b
> 来源：简书
> 著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。