Java解释型语言，首先编译成中间文件class文件，由各平台的JVM来解析class文件，解释成对应平台的机器码执行。
Go是编译型语言，直接由编译器编译成机器码，但是它的跨平台是在编译期确定的，会根据编译指令生成对应平台的机器码。如何做到呢？Runtime里面会实现各个平台的代码段，编译工具会根据平台选取不同的代码实现。有点类似于宏的概念。

Go里面的执行单位是goroutine，Java里面是线程。
Goroutine有独立的栈空间，每个Goroutine都有初始的栈空间，在运行期会随着函数执行发生栈空间的扩张和缩小。这里golang出现过两个方案，一个是分裂栈一个是复制栈。
Java每个线程有独立的栈空间，叫做虚拟机栈，一个方法依赖的局部变量表、操作数栈、动态链接、方法出口等信息。局部变量表是编译期确定的。栈是可以动态扩展的，这里可能会引发OOM。
还有个线程私有的空间是程序计数器，存的是线程正在执行的字节码地址，这样线程调度到不同的核上后，也可以继续执行，不会断片。
Java所有线程共享的是堆区，一般情况下Java的所有对象实例和数组都在这里分配内存，逃逸分析和标量替换技术可以改善这一现状。GC主要工作的对象就是这里，按照生存时间来划分有新生代和老年代。从设计角度来讲，其实跟Golang的类似，有线程独有的内存缓冲区。
Java方法区也是所有线程共享的，主要存储已被虚拟机加载的类信息、常量、静态变量、即使编译器产生的代码等。
Java本地方法栈，主要是针对调用Native方法产生内存分配。

### new的工作过程
关于Java对象的分配，一般有两种办法，第一种是内存规整的，也就是没使用和使用中的分开，用分界点来划分开；另一种是使用中和未使用的交错分布，然后通过链表来标记。所以Serial和ParNew带有压缩整理功能的GC算法 都是使用第一种来分配，Mark-Sweep这种使用后者。除了内存布局不同之外还有一个是如何保证并发安全性，这里就提到了线程的本地内存缓存池，也就是每个线程在分配内存时，先从自己的缓存池中取，其次本地缓存用完后，要向中央申请就需要加锁了。Go里面也采用的是这种，回头详细介绍Golang内存分配的时候多写点细节。
分配完内存后，有个初始化过程也就是zero value，在这一点上和Golang一样。紧接着Java里面还有个是对对象头进行设置。比如对象名、对象元数据标记、哈希吗、gc分代年代等。
紧接着是init,也就是自定义的逻辑了。

关于内存泄漏和内存溢出。前者指的是内存申请后没有合理的回收释放，表象是随着程序运行占用内存会逐渐增多，直到发生内存溢出.

### Java GC

#### 如何分析对象的引用？
1、引用计数，循环引用问题。
2、可达性分析，主要依赖于GC Roots到节点的连接是否是连续的。在Java里面可用做GC Roots的对象有虚拟机栈中引用的对象、方法区中类静态属性引用的对象、方法区中常量引用的对象、本地方法栈中JNI引用的对象。

Java对引用进行了分级，分为强引用、软引用、弱引用和虚引用。其中强引用是程序中比较明显的引用关系。而软引用是一些有用但非必须的对象，这些对象会在发生内存溢出之前进行二次回收。而弱引用是非必须对象，被弱引用关联的对象只能生存到下次GC发生之前。虚引用是一种不会对对象生存时间产生影响的引用，这种引用的目的是这个对象被垃圾回收之时会收到一个系统通知。
进行完可达性分析之后，还需要进行两次标记。其中第一次是筛选该对象是否实现过finalize函数或者说是否执行过finalize，如果是实现过且没有执行，则将这些对象放在F-Queue中，由单独的线程负责执行。在这个队列中的对象也会进行小范围的扫描。

如何回收方法区？一般的回收主要集中在堆区的新生代，而对于永久代的回收主要集中在无用的常量和无用的类，无用的类需要满足三个条件:  该类的对象全部被回收、加载该类的classLoader已经被回收、该类对应的java.lang.Class对象没有任何地方被引用。

#### Mark-Sweep算法
主要有两个不足：效率不高；空间间隔。

#### 复制算法
为了解决效率不高的问题，采用复制算法，主要针对新生代对象。每次一大块内存，使用完了，然后复制到新的内存上，把原来的回收掉，移动内存指针即可。一个弊端就是内存需要预留一倍。改进的办法就是把内存分成一块较大的Eden和两块较小的Survivor。

#### 整理标记Mark-Compact
因为复制算法在存活率较高时会频繁的复制降低效率，而且需要更多的内存，所以不适合老年代。
这个算法标记阶段与Mark-Sweep的标记阶段一样，只是后面是把所有存活的对象向一端移动，然后清理掉边界另一边的内存。

#### 可达性分析中的STW
如何去快速枚举GC Roots？JIT编译器在编译的时候会标记OopMap，标明有对象指针。那么在哪些点去标记OopMap？选取一些合适的safepoint，比如函数调用，循环跳转，异常跳转等。那如何去触发？主要有抢占和主动两种，大部分是主动中断。需要GC介入时，会设置一个标记，各线程去检查标记，如果是true，则会进行挂起。

### GC实现
#### CMS收集器主要分为四个步骤:
1、初始标记 2、并发标记 3、重新比较 4、并发清除。其中1和3需要STW。
初始标记是标记GC Roots能直接关联到的对象。并发标记就是GC Roots Tracing的过程；重新标记是为了解决在并发标记期间用户线程运行导致产生变动的那部分对象。
因为在并发清除阶段还会有垃圾产生，所以CMS会在内存占用达到68%时触发GC。
主要的三个问题：1、CPU敏感 2、刚才提到的浮动垃圾 3、内存碎片 Full GC来整理

#### G1收集器
也是四个步骤：1、初始标记 2、并发标记 3、最终标记 4、筛选回收
与CMS核心不同点是会预测一个回收效率最高的选项，也就是回收哪些region的数据 可以最优（停顿时间）。
不同Region之间的对象引用关系如何来维护？使用的Writer Barry。一旦有对象引用写操作产生就由writer barry触发检查。
