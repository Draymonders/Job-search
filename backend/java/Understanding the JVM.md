## 深入理解Java虚拟机
### 参数汇总
`-Xss` 栈容量
`-Xmx` 堆最大 `-Xms` 堆最小

### GC算法
#### 标记清除
分为两个阶段，标记和清除
不足之处： `效率低` `产生不连续的内存碎片`

#### 复制算法
大内存分为两块大小相等的内存
不足之处`实现简单，运行高效`
IBM研究表明，新生代中对象`98%`都是朝生夕死
Eden | survivor from | survivor to 默认比例 8:1:1
只有`10%`内存会被浪费

#### 标记整理
复制算法在`对象存活率较高`时就需要进行较多的复制操作

#### 分代收集
新生代 老年代 永久代(meta元空间)

### 垃圾收集器
#### CMS收集器
`Concurrent Mark Sweep` 收集器是一种以获取**最短回收停顿时间**为目标的收集齐。
分为四个步骤
- 初始标记
- 并发标记
- 重新标记
- 并发清除
初始标记和重新标记仍然需要`stop the world`
初始标记 就是标记`GC ROOTs`能直接连接到的对象
并发标记 就是不断 Tracing 的过程
重新标记，为了修正并发标记期间因用户运行而产生的变动的记录

**缺点**
- CMS不会导致用户线程停顿，但是占用了一部分CPU资源 默认启动的线程数是 `(CPU数量+3)/4 `,在CPU数量少的机器上资源占用较多，CPU数量越多，影响越小
- 无法处理浮动垃圾，**浮动垃圾**是并发清理阶段产生的新的垃圾，这部分垃圾在标记之后，因此只能下次GC再清理掉
- CMS 是 标记清除，会造成许多内存碎片

#### G1收集器
优点:
- 并发与并行， 在多核场景下，可以利用少部分核心GC, 减少全局`stop the world`的情况发生
- G1 使用的是 标记整理, 是基于`复制`实现的, 不会出现内存碎片
- G1 比 CMS 优点， 可以建立**可预测的停顿时间模型**

G1有一个优先队列，里面的垃圾堆有这样的比较 **回收所获得的空间大小以及回收所使用的时间**,这也是`Garbage First`的由来

有一个`Remembered Set`来避免全堆扫描, 不同Region区，之间有对象的引用的话就放到`Remembered Set`里面，当 枚举GC Roots 进行回收时，有了`Remembered set` 不用扫描全堆也能进行回收

大体步骤
- 初始标记
- 并发标记
- 最终标记
- 筛选回收


#### 内存分配与回收
Minor GC | Major GC | FULL GC
- 新生代GC(Minor GC) 指发生在新生代的垃圾收集动作，Minor GC 非常频繁
- 老年代GC(Major GC/Full GC) 在老年代的GC，伴随着至少一次的Minor GC， `Major GC` 比 `Minor GC` 慢10倍以上

默认15次 Minor GC 晋升到老年代中 ，或者 在`survivor` 空间中相同年龄所有对象大小大于`survivor`空间的一半,可以直接进入老年代
### 虚拟机性能监控与故障处理工具
- jps Process status tool 显示制定系统内所有的虚拟机进程
- jstat Statistics Monitoring Tool 收集HotSpot虚拟机各方面的运行数据
- jinfo Configuration Info For Java 显示虚拟机配置信息
- jmap Memory Map for Java 生成Java转储文件(heapdump文件)
- jhat JVM Heap Analysis Tool 分析 heapdump 文件，建立HTTP服务器，查看分析结果
- jstack Stack Trace for Java 显示虚拟机的线程快照
#### 常用命令汇总
```
jps -l 查看所有的线程 并且输出主类的全名
jstat -gc port 250 20 每250秒查询一次 port进程的垃圾收集情况，一共执行20次
jmap -dump:format=b,file=eclipse.bin port 查看port进程的dump 信息
jhat eclipse.bin 查看dump文件信息
jstack -l port 查看堆栈分析 （-l 显示锁信息， -m 显示JNI的堆栈信息）
```

### 多态
1. 静态分派 `Human h = new Man()`
2. 动态分派
- `javap`查看源代码 可以看到 方法加载的指令`invokevirtual`
- 解析过程分为下面几个步骤
1. 找到操作数栈中的第一个元素所指向的对象的**类型**
2. 如果在类型中找到与常量中的描述符和简单名称都相符的方法，进行权限校验，如果通过就直接返回这个方法的引用，否则，抛`IllegalAccessEoor`
3. 否则，按照继承关系从下往下进行第二部的搜索过程
4. 如果始终没有找到合适的方法，则抛出`AbstractMethodError`异常

### 运行期优化
一个方法的参数被传递到其他方法中，称为方法逃逸

一个对象赋值给类变量活其他线程中访问的实例变量所引用，称为线程逃逸

如果能证明一个对象没有逃逸。那么我们就能做一些高效的优化
#### 栈上分配
#### 同步消除
能发现变量不会逃逸出线程， 就不会有竞争，就不需要用锁
#### 标量替换
如果一个对象没有发生逃逸，并且这个对象可以被拆成多个标量的话，那么这个对象可能都不用生成

### 线程安全
> 线程安全：当多个线程访问同一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方进行任何其他的协调操作，调用这个对象的行为都可以获得正确的结果，那这个对象是线程安全的。

### OOM
OOM（内存溢出）情况有以下三种：

java.lang.OutOfMemoryError: Java heap space ------> java堆内存溢出  （主要讲解）
可能原因：一般由于内存泄露或者堆的大小设置不当引起。
对于内存泄露，需要通过内存监控软件查找程序中的泄露代码。
调整：确定不是代码问题后，堆大小可以通过虚拟机参数-Xms,-Xmx等修改。

Java.lang.OutOfMemoryError: PermGen space ----java永久代溢出
可能原因：方法区溢出了，一般出现于大量Class或者jsp页面，或者采用cglib等反射机制的情况，因为上述情况会产生大量的Class信息存储于方法区。
调整：通过更改方法区的大小解决，用类似-XX:PermSize=64m –XX:MaxPermSize=256m的形式修改。

Java.lang.StackOverflowError ---- 不会抛OOM error
可能原因：JVM栈溢出，一般是由于程序中存在死循环或者深度递归调用造成的，栈大小设置太小也会出现此种溢出。
调整：可以通过虚拟机参数-Xss来设置栈的大小。



