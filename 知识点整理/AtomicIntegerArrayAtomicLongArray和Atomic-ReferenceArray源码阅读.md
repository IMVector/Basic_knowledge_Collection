---
title: AtomicIntegerArrayAtomicLongArray和Atomic-ReferenceArray源码阅读
date: 2021-03-04 09:11:01
tags:
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

Concurrent包提供了AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray三个数组元素的原子操作。注意，这里并不是说对整个数组的操作是原子的，**而是针对数组中一个元素的原子操作而言**。





AtomicIntegerArray的大多数操作与AtomicInteger相同,不同之处在于AutomicIntegerArray中需要通过index计算要修改或者查询元素的内存偏移量,此外还需要计算这个内存偏移量.



### 内存偏移量的计算方法

    base表示数组的首地址的位置，scale表示一个数组元素的大小，
    i的偏移量则等于：i*scale+base。
```java
package java.util.concurrent.atomic;
import java.util.function.IntUnaryOperator;
import java.util.function.IntBinaryOperator;
import sun.misc.Unsafe;

public class AtomicIntegerArray implements java.io.Serializable {
    private static final long serialVersionUID = 2862133569453604235L;

    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // ------------------------------------------------------
    // 以下两个元素用了Unsafe类的arrayBaseOffset和arrayIndexScale两个函数来获取
    
    // 用来转换内存地址,表示数组的首地址的位置
    private static final int base = unsafe.arrayBaseOffset(int[].class);
    // 用来转换内存地址
    private static final int shift;
    // ------------------------------------------------------
	// 保存的底层数组
    private final int[] array;

    // base表示数组的首地址的位置，scale表示一个数组元素的大小，
    // i的偏移量则等于：i*scale+base。
    static {
        // scale是2的整数次方
        int scale = unsafe.arrayIndexScale(int[].class);
        if ((scale & (scale - 1)) != 0)
            throw new Error("data type scale not a power of two");
        // 该方法的作用是返回无符号整型i的最高非零位前面的0的个数，包括符号位在内；
        
        // 如果i为负数，这个方法将会返回0，符号位为1.
        // 比如说，10的二进制表示为 0000 0000 0000 0000 0000 0000 0000 1010
        // java的整型长度为32位。那么这个方法返回的就是28
        shift = 31 - Integer.numberOfLeadingZeros(scale);
    }
	// 检查数组下标是否越界
    private long checkedByteOffset(int i) {
        if (i < 0 || i >= array.length)
            throw new IndexOutOfBoundsException("index " + i);

        return byteOffset(i);
    }
	// 将下标i转换成内存的偏移量
    private static long byteOffset(int i) {
        return ((long) i << shift) + base;
    }


    public AtomicIntegerArray(int length) {
        array = new int[length];
    }

    public AtomicIntegerArray(int[] array) {
        // Visibility guaranteed by final field guarantees
        this.array = array.clone();
    }


    public final int length() {
        return array.length;
    }

    public final int get(int i) {
        return getRaw(checkedByteOffset(i));
    }

    private int getRaw(long offset) {
        return unsafe.getIntVolatile(array, offset);
    }

	// 设置数组中第index[i]元素的值
    public final void set(int i, int newValue) {
        // checkedByteOffset(i)首先将数组下标i转换成内存的偏移量
        unsafe.putIntVolatile(array, checkedByteOffset(i), newValue);
    }

    public final void lazySet(int i, int newValue) {
        unsafe.putOrderedInt(array, checkedByteOffset(i), newValue);
    }

    public final int getAndSet(int i, int newValue) {
        return unsafe.getAndSetInt(array, checkedByteOffset(i), newValue);
    }


    public final boolean compareAndSet(int i, int expect, int update) {
        // 首先根据inde计算内存的偏移量
        return compareAndSetRaw(checkedByteOffset(i), expect, update);
    }

    private boolean compareAndSetRaw(long offset, int expect, int update) {
        // 调用unsafe中的CAS原子操作
        // 第1个参数是int[]对象，第2个参数是下标i对应的内存偏移量，第3个和第4个参数分别是旧值和新值。
        return unsafe.compareAndSwapInt(array, offset, expect, update);
    }

    public final boolean weakCompareAndSet(int i, int expect, int update) {
        return compareAndSet(i, expect, update);
    }

    public final int getAndIncrement(int i) {
        return getAndAdd(i, 1);
    }

    public final int getAndDecrement(int i) {
        return getAndAdd(i, -1);
    }

    public final int getAndAdd(int i, int delta) {
        return unsafe.getAndAddInt(array, checkedByteOffset(i), delta);
    }

    public final int incrementAndGet(int i) {
        return getAndAdd(i, 1) + 1;
    }


    public final int decrementAndGet(int i) {
        return getAndAdd(i, -1) - 1;
    }

    public final int addAndGet(int i, int delta) {
        return getAndAdd(i, delta) + delta;
    }


    public final int getAndUpdate(int i, IntUnaryOperator updateFunction) {
        long offset = checkedByteOffset(i);
        int prev, next;
        do {
            prev = getRaw(offset);
            next = updateFunction.applyAsInt(prev);
        } while (!compareAndSetRaw(offset, prev, next));
        return prev;
    }

    public final int updateAndGet(int i, IntUnaryOperator updateFunction) {
        long offset = checkedByteOffset(i);
        int prev, next;
        do {
            prev = getRaw(offset);
            next = updateFunction.applyAsInt(prev);
        } while (!compareAndSetRaw(offset, prev, next));
        return next;
    }

    public final int getAndAccumulate(int i, int x,
                                      IntBinaryOperator accumulatorFunction) {
        long offset = checkedByteOffset(i);
        int prev, next;
        do {
            prev = getRaw(offset);
            next = accumulatorFunction.applyAsInt(prev, x);
        } while (!compareAndSetRaw(offset, prev, next));
        return prev;
    }

    public final int accumulateAndGet(int i, int x,
                                      IntBinaryOperator accumulatorFunction) {
        long offset = checkedByteOffset(i);
        int prev, next;
        do {
            prev = getRaw(offset);
            next = accumulatorFunction.applyAsInt(prev, x);
        } while (!compareAndSetRaw(offset, prev, next));
        return next;
    }


    public String toString() {
        int iMax = array.length - 1;
        if (iMax == -1)
            return "[]";

        StringBuilder b = new StringBuilder();
        b.append('[');
        for (int i = 0; ; i++) {
            b.append(getRaw(byteOffset(i)));
            if (i == iMax)
                return b.append(']').toString();
            b.append(',').append(' ');
        }
    }

}

```



