# 【死磕Java并发】—–深入分析volatile的实现原理

{% embed url="https://mp.weixin.qq.com/s?\_\_biz=MzUzMTA2NTU2Ng==&mid=2247483784&idx=1&sn=672cd788380b2096a7e60aae8739d264&chksm=fa497e39cd3ef72fcafe7e9bcc21add3dce0d47019ab6e31a775ba7a7e4adcb580d4b51021a9&scene=21\#wechat\_redirect" %}





作者：大明哥 原文地址：[http://cmsblogs.com/?p=2092](http://cmsblogs.com/?p=2092)

**友情提示：欢迎关注公众号【芋道源码】。😈关注后，拉你进【源码圈】微信群和【大明哥】搞基嗨皮。**

**友情提示：欢迎关注公众号【芋道源码】。😈关注后，拉你进【源码圈】微信群和【\**大明哥\*_】搞基嗨皮。\*_

**友情提示：欢迎关注公众号【芋道源码】。😈关注后，拉你进【源码圈】微信群和【\**大明哥\*_】搞基嗨皮。\*_

通过前面一章我们了解了synchronized是一个重量级的锁，虽然JVM对它做了很多优化，而下面介绍的volatile则是轻量级的synchronized。如果一个变量使用volatile，则它比使用synchronized的成本更加低，因为它不会引起线程上下文的切换和调度。Java语言规范对volatile的定义如下：

> Java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致地更新，线程应该确保通过排他锁单独获得这个变量。

上面比较绕口，通俗点讲就是说一个变量如果用volatile修饰了，则Java可以确保所有线程看到这个变量的值是一致的，如果某个线程对volatile修饰的共享变量进行更新，那么其他线程可以立马看到这个更新，这就是所谓的线程可见性。

volatile虽然看起来比较简单，使用起来无非就是在一个变量前面加上volatile即可，但是要用好并不容易（LZ承认我至今仍然使用不好，在使用时仍然是模棱两可）。

## 内存模型相关概念

理解volatile其实还是有点儿难度的，它与Java的内存模型有关，所以在理解volatile之前我们需要先了解有关Java内存模型的概念，这里只做初步的介绍，后续LZ会详细介绍Java内存模型。

### 操作系统语义

计算机在运行程序时，每条指令都是在CPU中执行的，在执行过程中势必会涉及到数据的读写。我们知道程序运行的数据是存储在主存中，这时就会有一个问题，读写主存中的数据没有CPU中执行指令的速度快，如果任何的交互都需要与主存打交道则会大大影响效率，所以就有了CPU高速缓存。CPU高速缓存为某个CPU独有，只与在该CPU运行的线程有关。

有了CPU高速缓存虽然解决了效率问题，但是它会带来一个新的问题：数据一致性。在程序运行中，会将运行所需要的数据复制一份到CPU高速缓存中，在进行运算时CPU不再也主存打交道，而是直接从高速缓存中读写数据，只有当运行结束后才会将数据刷新到主存中。举一个简单的例子：

```text
i++i++
```

当线程运行这段代码时，首先会从主存中读取i\( i = 1\)，然后复制一份到CPU高速缓存中，然后CPU执行 + 1 （2）的操作，然后将数据（2）写入到告诉缓存中，最后刷新到主存中。其实这样做在单线程中是没有问题的，有问题的是在多线程中。如下：

假如有两个线程A、B都执行这个操作（i++），按照我们正常的逻辑思维主存中的i值应该=3，但事实是这样么？分析如下：

两个线程从主存中读取i的值（1）到各自的高速缓存中，然后线程A执行+1操作并将结果写入高速缓存中，最后写入主存中，此时主存i==2,线程B做同样的操作，主存中的i仍然=2。所以最终结果为2并不是3。这种现象就是缓存一致性问题。

解决缓存一致性方案有两种：

1. 通过在总线加LOCK\#锁的方式
2. 通过缓存一致性协议

但是方案1存在一个问题，它是采用一种独占的方式来实现的，即总线加LOCK\#锁的话，只能有一个CPU能够运行，其他CPU都得阻塞，效率较为低下。

第二种方案，缓存一致性协议（MESI协议）它确保每个缓存中使用的共享变量的副本是一致的。其核心思想如下：当某个CPU在写数据时，如果发现操作的变量是共享变量，则会通知其他CPU告知该变量的缓存行是无效的，因此其他CPU在读取该变量时，发现其无效会重新从主存中加载数据。

![212219343783699](https://gitee.com/baicaihenxiao/imageDB/raw/master/uPic/jpg/2020/07/09/640-20200709115617984-115618.jpg)

### Java内存模型

上面从操作系统层次阐述了如何保证数据一致性，下面我们来看一下Java内存模型，稍微研究一下Java内存模型为我们提供了哪些保证以及在Java中提供了哪些方法和机制来让我们在进行多线程编程时能够保证程序执行的正确性。

在并发编程中我们一般都会遇到这三个基本概念：原子性、可见性、有序性。我们稍微看下volatile

#### 原子性

> 原子性：即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

原子性就像数据库里面的事务一样，他们是一个团队，同生共死。其实理解原子性非常简单，我们看下面一个简单的例子即可：

```text
i = 0;            ---1j = i ;            ---2i++;            ---3i = j + 1;    ---4
```

上面四个操作，有哪个几个是原子操作，那几个不是？如果不是很理解，可能会认为都是原子性操作，其实只有1才是原子操作，其余均不是。

1—在Java中，对基本数据类型的变量和赋值操作都是原子性操作； 2—包含了两个操作：读取i，将i值赋值给j 3—包含了三个操作：读取i值、i + 1 、将+1结果赋值给i； 4—同三一样

在单线程环境下我们可以认为整个步骤都是原子性操作，但是在多线程环境下则不同，Java只保证了基本数据类型的变量和赋值操作才是原子性的（**注：在32位的JDK环境下，对64位数据的读取不是原子性操作\*，如long、double**）。要想在多线程环境下保证原子性，则可以通过锁、synchronized来确保。

> volatile是无法保证复合操作的原子性

#### 可见性

> 可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

在上面已经分析了，在多线程环境下，一个线程对共享变量的操作对其他线程是不可见的。

**Java提供了volatile来保证可见性。**

当一个变量被volatile修饰后，表示着线程本地内存无效，当一个线程修改共享变量后他会立即被更新到主内存中，当其他线程读取共享变量时，它会直接从主内存中读取。 当然，synchronize和锁都可以保证可见性。

#### 有序性

> 有序性：即程序执行的顺序按照代码的先后顺序执行。

在Java内存模型中，为了效率是允许编译器和处理器对指令进行重排序，当然重排序它不会影响单线程的运行结果，但是对多线程会有影响。

Java提供volatile来保证一定的有序性。最著名的例子就是单例模式里面的DCL（双重检查锁）。这里LZ就不再阐述了。

## 剖析volatile原理

JMM比较庞大，不是上面一点点就能够阐述的。上面简单地介绍都是为了volatile做铺垫的。

> volatile可以保证线程可见性且提供了一定的有序性，但是无法保证原子性。在JVM底层volatile是采用“内存屏障”来实现的。

上面那段话，有两层语义

1. 保证可见性、不保证原子性
2. 禁止指令重排序

第一层语义就不做介绍了，下面重点介绍指令重排序。

在执行程序时为了提高性能，编译器和处理器通常会对指令做重排序：

1. 编译器重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序；
2. 处理器重排序。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序；

指令重排序对单线程没有什么影响，他不会影响程序的运行结果，但是会影响多线程的正确性。既然指令重排序会影响到多线程执行的正确性，那么我们就需要禁止重排序。那么JVM是如何禁止重排序的呢？这个问题稍后回答，我们先看另一个原则happens-before，happen-before原则保证了程序的“有序性”，它规定如果两个操作的执行顺序无法从happens-before原则中推到出来，那么他们就不能保证有序性，可以随意进行重排序。其定义如下：

1. 同一个线程中的，前面的操作 happen-before 后续的操作。（即单线程内按代码顺序执行。但是，在不影响在单线程环境执行结果的前提下，编译器和处理器可以进行重排序，这是合法的。换句话说，这一是规则无法保证编译重排和指令重排）。
2. 监视器上的解锁操作 happen-before 其后续的加锁操作。（Synchronized 规则）
3. 对volatile变量的写操作 happen-before 后续的读操作。（volatile 规则）
4. 线程的start\(\) 方法 happen-before 该线程所有的后续操作。（线程启动规则）
5. 线程所有的操作 happen-before 其他线程在该线程上调用 join 返回成功后的操作。
6. 如果 a happen-before b，b happen-before c，则a happen-before c（传递性）。

我们着重看第三点volatile规则：对volatile变量的写操作 happen-before 后续的读操作。为了实现volatile内存语义，JMM会重排序，其规则如下：

对happen-before原则有了稍微的了解，我们再来回答这个问题JVM是如何禁止重排序的？

![20170104-volatile](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

**观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令**。lock前缀指令其实就相当于一个内存屏障。内存屏障是一组处理指令，用来实现对内存操作的顺序限制。volatile的底层就是通过内存屏障来实现的。下图是完成上述规则所需要的内存屏障：

volatile暂且下分析到这里，JMM体系较为庞大，不是三言两语能够说清楚的，后面会结合JMM再一次对volatile深入分析。

![20170104-volatile2](https://gitee.com/baicaihenxiao/imageDB/raw/master/uPic/jpg/2020/07/09/640-20200709115618122-115618.jpg)

## 总结

volatile看起来简单，但是要想理解它还是比较难的，这里只是对其进行基本的了解。volatile相对于synchronized稍微轻量些，在某些场合它可以替代synchronized，但是又不能完全取代synchronized，只有在某些场合才能够使用volatile。使用它必须满足如下两个条件：

1. 对变量的写操作不依赖当前值；
2. 该变量没有包含在具有其他变量的不变式中。

> volatile经常用于两个两个场景：状态标记两、double check

## 参考资料

1. 周志明：《深入理解Java虚拟机》
2. 方腾飞：《Java并发编程的艺术》
3. Java并发编程：volatile关键字解析
4. Java 并发编程：volatile的使用及其原理
