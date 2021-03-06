# volatile关键字

## 1. Java并发这块掌握的怎么样？来谈谈你对volatile关键字的理解吧。

我的理解是，被volatile修饰的共享变量，就会具有以下两个特性：

1. **保证了不同线程对该变量操作的内存可见性。**
2. **禁止指令重排序。**

## 2. 那你可不可以详细的说一下究竟什么是内存可见性，什么又是重排序？

这个要是说起来可就多了，我就从Java内存模型开始说起吧。Java虚拟机规范试图定义一个Java内存模型(JMM)，以屏蔽所有类型的硬件和操作系统内存访问差异，让Java程序在不同的平台上能够达到一致的内存访问效果。简单地说，由于CPU执行指令的速度很快，但是内存访问速度很慢，差异不是一个量级，所以搞处理器的那群大佬们又在CPU里加了好几层高速缓存。

在Java内存模型中，对上述优化进行了一波抽象。**JMM规定所有的变量都在主内存中，类似于上面提到的普通内存，每个线程又包含自己的工作内存，为了便于理解可以看成CPU上的寄存器或者高速缓存。因此，线程的操作都是以工作内存为主，它们只能访问自己的工作内存，并且在工作之前和之后，该值被同步回主内存。**

**线程执行的时候，将首先从主内存读值，再load到工作内存中的副本中，然后传给处理器执行，执行完毕后再给工作内存中的副本赋值，随后工作内存再把值传回给主存，主存中的值才更新。**

使用工作内存和主存，虽然加快了速度，但也带来了一些问题。例如：

```
i = i + 1;
```

假设 i 的初始值为 0 ，当只有一个线程执行它的时候，结果肯定是 1 ，那么当两个线程执行时，得到的结果会是 2 吗？不一定。可能会存在这种情况：

```
线程1： load i from 主存    // i = 0
        i + 1  // i = 1
线程2： load i from主存  // 因为线程1还没将i的值写回主存，所以i还是0
        i +  1 //i = 1
线程1:  save i to 主存
线程2： save i to 主存
```

如果两个线程遵循上面的执行过程，那么 i 的最终值竟然是 1 。如果最后的写回生效的慢，你再读取 i 的值，都可能会是 0 ，这就是缓存不一致的问题。

接下来就要提到您刚才所问的问题了，**JMM主要围绕在并发过程中如何处理并发原子性、可见性和有序性这三个特征来建立的**，通过解决这三个问题，就可以解决缓存不一致的问题。而volatile跟可见性和有序性都有关。

## 3. 那你具体说说这三个特性呢？

**1 . 原子性(Atomicity)：**

在Java中，**对基本数据类型的读取和赋值操作是原子性操作，所谓原子性操作就是指这些操作是不可中断的，要做一定做完，要么就没有执行。** 例如：

```
i = 2;
j = i;
i++;
i = i + 1;
```

以上四个操作， i = 2 是一个读取操作，肯定是原子性操作， j = i 你觉得是原子性操作，但事实上，可以分为两个步骤，一个是读取 i 的值，然后再把值赋给 j ，这已经是两步操作了，不能称为原子操作，`i++ `和 `i = i + 1 `是等效的，读的值，+ 1，然后写回主存，这是三个步骤的操作了。在上面的例子中，最后一个值可能在各种情况下，因为它不会满足原子性。

在本例中，只有一个简单的读取，赋值是一个原子操作，并且只能被分配给一个数字，使用变量来读取变量的值的操作。一个例外是，在虚拟机规范中允许64位数据类型(long和double)，它被划分为两个32位操作，但是JDK的最新实现实现了原子操作。

JMM只实现基本的原子性，比如上面的i++操作，它必须依赖于同步和锁定，以确保整个代码块的原子性。在释放锁之前，线程必须将I的值返回到主内存。

**2 . 可见性(Visibility)：**

**说到可见性，Java使用volatile来提供可见性。当一个变量被volatile修改时，它的变化会立即被刷新到主存，当其他线程需要读取变量时，它会读取内存中的新值。**普通变量不能保证。

事实上，同步和锁定也可以保证可见性。在释放锁之前，线程将把共享变量值刷回主内存，但是同步和锁更昂贵。

**3 . 有序性(Ordering):**

**JMM允许编译器和处理器重新排序指令**，**但是指定了as-if-串行语义，也就是说，无论重新排序，程序的执行结果都不能更改。**例如:

```
double pi = 3.14;    //A
double r = 1;        //B
double s= pi * r * r;//C
```

上面的语句中,可以按照C - > B - >,结果是3.14,但它也可以按照的顺序B - > - > C,因为A和B是两个单独的语句,并依赖,B,C和A和B可以重新排序,但C不能行前面的A和B。JMM确保重新排序不会影响单线程的执行，但容易出现多线程问题。例如，这样的代码:

```java
int a = 0;
bool flag = false;
 
public void write() {
    a = 2;              //1
    flag = true;        //2
}
 
public void multiply() {
    if (flag) {         //3
        int ret = a * a;//4
    }
     
}
```

如果有两个线程执行上面的代码段，线程1首先执行写，然后再乘以线程2。最后，ret的值必须是4?不一定:

![img](https://images2018.cnblogs.com/blog/1246845/201805/1246845-20180513112443139-1944588828.png)

如图1和2所示，在写方法中进行重新排序，线程1对第一个赋值为true，然后执行到线程2,ret直接计算结果，然后再执行线程1，这一次的CaiFu值为2，显然是较晚的步骤。

此时要标记加上volatile关键字，重新排序，可以确保程序的“顺序”，也可以基于重量级的同步和锁定来确保，他们可以确保在代码执行的区域内一次性完成。

此外，JMM有一些内在的规律性，也就是说，没有任何方法可以保证有序，这通常称为发生在原则之前。<< jsr-133: Java内存模型和线程规范>>定义了以下事件:

- **程序顺序规则**： 一个线程中的每个操作，happens-before于该线程中的任意后续操作
- **监视器锁规则**：对一个线程的解锁，happens-before于随后对这个线程的加锁
- **volatile变量规则**： 对一个volatile域的写，happens-before于后续对这个volatile域的读
- **传递性**：如果A happens-before B ,且 B happens-before C, 那么 A happens-before C
- **start()规则**： 如果线程A执行操作`ThreadB_start()`(启动线程B) , 那么A线程的`ThreadB_start()`happens-before 于B中的任意操作
- **join()原则**： 如果A执行`ThreadB.join()`并且成功返回，那么线程B中的任意操作happens-before于线程A从`ThreadB.join()`操作成功返回。
- **interrupt()原则**： 对线程`interrupt()`方法的调用先行发生于被中断线程代码检测到中断事件的发生，可以通过`Thread.interrupted()`方法检测是否有中断发生
- **finalize()原则**：一个对象的初始化完成先行发生于它的`finalize()`方法的开始

第1条程序顺序规则在一个线程中，所有的操作都是有序的，但实际上只要JMM的执行结果允许重新排序，这也是发生的重点——单线程执行结果是正确的，但是也不能保证多线程。

规则2，规则监视器的规则，非常好理解。在锁被添加之前，锁已经被释放，然后它才能继续被锁定。

第三条规则适用于讨论的不稳定性。如果一个行程序编写一个变量，另一个线程读取它，那么在操作之前必须读取写入操作。

第四个规则正在发生。

接下来的几行不会重复。

## 4. volatile关键字如何满足并发编程的三大特性的？

继续上面的代码示例:

```java
int a = 0;
bool flag = false;
 
public void write() {
   a = 2;              //1
   flag = true;        //2
}
 
public void multiply() {
   if (flag) {         //3
       int ret = a * a;//4
   }
    
}
```

**当写一个volatile变量时，JMM将本地内存更改的变量写回到主内存中。**

**当取一个volatile变量时，JMM将使线程对应的本地内存失效，然后线程将从主内存读取共享变量。**

volatile 可以保证线程可见性且提供了一定的有序性，但是无法保证原子性。在 JVM 底层是基于内存屏障实现的。

- 当对非 volatile 变量进行读写的时候，每个线程先从内存拷贝变量到 CPU 缓存中。如果计算机有多个CPU，每个线程可能在不同的 CPU 上被处理，这意味着每个线程可以拷贝到不同的本地内存中。
- 而声明变量是 volatile 的，JVM 保证了每次读变量都从内存中读，跳过本地内存这一步，所以就不会有可见性问题
  - **对 volatile 变量进行写操作时，会在写操作后加一条 store 屏障指令，将工作内存中的共享变量刷新回主内存；**
  - **对 volatile 变量进行读操作时，会在读操作前加一条 load 屏障指令，从主内存中读取共享变量；**

**volatile可以通过内存屏障防止指令重排序问题。硬件层面的内存屏障分为读屏障和写屏障。**

**对于读屏障来说，在指令前插入读屏障，可以让高速缓存中的数据失效，强制重新从主内存加载数据。**
**对于写屏障来说，在指令后插入写屏障，能让写入缓存中的最新数据更新写入主内存，让其他线程可见。**