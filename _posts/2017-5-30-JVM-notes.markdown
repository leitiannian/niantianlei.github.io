---
layout: post
title:  "《深入理解Java虚拟机》笔记"
subtitle:  " \"the notes of Understanding the JVM\""

date:   2017-05-30 22:40:18 +0800
tags:
    - 虚拟机
    - Java
author: Nian Tianlei
header-img: "img/post-bg-rwd.jpg"
header-mask: 0.4
catalog:    true
---

> This document is not completed and will be updated in the future.  


这本书之前看过两遍，最近又温习了一次，对Java内存及其溢出、GC算法、虚拟机加载执行等知识有了更深入的了解。把笔记在这里记录下来。  


## Java内存区域与内存溢出异常  
### 运行时数据区  
![a]({{ "/img/post/JVM/1.png" | prepend: site.baseurl }} )  
<b>程序计数器，</b>线程私有，是一块较小的内存空间，可以看作是当前线程所执行的字节码的行号指示器。  
<b>Java虚拟机栈</b>，线程私有，每个线程对应一个栈，生命周期同线程。每个方法执行时会创建一个栈帧用于存储局部变量表、操作数栈、动态链接、方法出口等。每个方法的调用到执行的过程就对应一个栈帧在虚拟机栈中入栈到出栈的过程。  
局部变量表存放了编译期可知的各种基本数据类型、对象引用。  
存在两种异常情况：线程请求的栈深度超出虚拟机允许的范围，将会抛出StackOverflowError异常；如果在动态扩展时无法申请到足够的内存，将会抛出OutOfMemoryError异常。  
<b>本地方法栈，</b>与虚拟机栈作用类似。区别是本地方法栈为虚拟机使用到的Native方法服务。  
<b>Java堆，</b>是虚拟机管理内存中最大的一块。是线程共享的，在虚拟机启动时创建。用于存放对象的实例，现在的收集器基本都是采用分代收集算法进行垃圾回收。  
<b>方法区，</b>所有线程共享的，存储已经被虚拟机加载的类信息，常量，静态变量，即时编译器编译后的代码等。  
<b>运行时常量池，</b>方法区的一部分，存放编译期生成的各种字面量和符号引用。  
### 对象的创建  
检查根据new关键字后面的参数能否在常量池中定位到一个类的符号引用，并且检查这个类是否已被加载、解析和初始化过。如果没有，那必须执行类加载过程。  
类加载通过后，虚拟机将为新生对象分配内存。  
### 对象的访问定位  
Java程序通过栈上的reference数据来操作堆上的具体对象，reference定位访问到堆上对象的方法取决于虚拟机的设定，主流的访问方式有使用句柄和直接指针两种。  
使用句柄需要在Java堆中划出来一块内存放句柄池，reference中存储的就是对象的句柄地址，然后通过句柄池访问实例和方法区中的对象类型数据。   
直接指针访问就是直接指到Java堆对象实例所在的位置，然后对象实例所在的位置会有个指针指向方法区的对象类型数据。   
使用句柄的好处是，对象被移动时只会改变句柄中的实例数据指针，而reference本身不需要修改。  
直接指针访问的好处就是快。  
### OutOfMemoryError异常  
除了程序计数器以外，其它的几个虚拟机内存运行时区域都有可能发生OOM。  
**Java堆溢出**  
通过-Xmx和-Xms将堆设为20M，且不可扩展，此时输出结果如下：
![b]({{ "/img/post/JVM/2.png" | prepend: site.baseurl }} )
**虚拟机栈和本地方法栈溢出**  
虚拟机请求的栈深度大于虚拟机所允许的最大深度，抛出StackOverflowError异常。  
扩展栈时没有申请到足够大的内存空间，则抛出OutOfMemoryError异常。  
单线程模式下，无论是栈帧太大还是虚拟机栈容量太小，一般都只会抛出StackOverflowError异常。  
多线程模式下，会产生OutOfMemoryError异常，为每个线程分配的栈越大越容易异常，因为除去堆区和方法区的部分如果不够线程瓜分，就会内存溢出。  
如果是建立过多线程导致的内存溢出，在不能减少线程数或者更换64位虚拟机的情况下，可以通过减少最大堆和减少栈容量来换取更多线程。  
## 垃圾收集器与内存分配策略  
### 判断对象已死  
- 引用计数算法  
给对象添加一个引用计数器，每当有一个地方引用它时，计数器值+1；引用失效时，计数器值-1；当计数器为0时对象不会再被使用。  
缺点：很难解决对象之间相互循环引用的问题。  
- 可达性分析算法  
通过“GC Roots”的对象作为起点，向下搜索。走过的路径成为引用链。当一个对象到GC Roots没有引用链相连证明此对象不可用。  
在Java中可作为GC Roots的对象包括：  
1. 虚拟机栈（栈帧中的本地变量表）中的引用的对象；  
2. 方法区中的类静态属性引用的对象；  
3. 方法区中常量引用的对象；  
1. 本地方法栈中JNI（即一般所说的native方法）引用的对象。  

事实上吗，真正判断一个对象死亡，至少经历两次标记过程：如果一个对象经可达性分析算法发现无引用链，会被第一次标记并进行一次筛选，筛选条件是此对象有无必要执行finalize()方法。（当对象没覆盖finalize()方法，或finalize()方法已被虚拟机调用过，视为没必要执行）  
如果该对象有必要执行finalize()方法，那么把它放在F-Queue队列中，GC将对F-Queue中的对象进行第二次小规模标记。如果对象在finalize()中重新与引用链上的任何一个对象建立关联。那它就仍可存活。  
finalize()方法是对象逃脱的最后一次机会，且只会被调用一次。  
### 垃圾收集算法  
- 标记-清除算法  
首先标记出所有需要回收的对象，在标记完成后统一回收被标记的对象。  
缺点：标记和清除过程的效率都不高；标记清除后会产生大量不连续的内存碎片，导致以后需要分配较大对象时，无法找到足够大的连续内存而不得不出发垃圾收集动作。  
- 复制算法  
该算法将内存按容量划分为大小相等的两块区域，每次只使用其中的一块。当一块内存用完了，就将其中还存活的对象复制到另一块区域上，然后再将已经使用过的内存区域一次性清理掉。解决了内存碎片的问题。  
缺点：内存缩小为原来的一半。  
改进：将内存划分为一块较大的Eden空间和两块较小的Survivor空间，每次使用Eden和其中的一块Survivor。当回收时，将Eden和Survivor中还存活的对象一次性复制到另外一块Survivor空间上，最后清理掉Eden和刚刚用过的Survivor的空间。HotSpot虚拟机默认Eden和Survivor的大小比例是8:1:1。
缺点：无法保证每次回收都只有不多于10%的对象存活；在对象存活率较高时需要执行较多的复制操作，效率将会变低，老年代不能使用这种算法。
- 标记-整理算法  
其中标记过程和“标记-清除”算法一样，但是后续步骤不是直接对可回收的对象进行清理，而是让所有存活的对象都向一端移动，然后直接清理掉端边界以外的内存。  
- 分代收集算法  
当前商业虚拟机的垃圾回收都是采用“分代收集”算法，根据对象的存活周期把Java堆分为新生代和老年代，这样就可以根据各个年代的特点采用最适合的收集算法。在新生代中，每次垃圾收集时都会发现有大量对象死去，只有少量对象存活，那就选择复制算法；而老年代中因为对象存活率较高，没有额外空间对它进行分配担保，就必须采用“标记-清除”或者“标记-整理”算法来进行回收。  

### 垃圾收集器  

| 垃圾收集器        | 算法   |   方式   |  堆区域    |  机制  |
| :--------   | :-----   | :---- | :-----   | :---- |
| Serial收集器 | 复制算法 |  串行  | 新生代 |   Stop The World   |
| Serial Old收集器 | 标记-整理算法 |  串行  | 老年代 |   Stop The World   |
| ParNew收集器    | 复制算法   |   并行   | 新生代  |   Stop The World    |
| Parallel Scavenge收集器    | 复制算法   |   并行   | 新生代  |   Stop The World    |
| Parallel Old收集    | 标记-整理算法   |   并行   | 老年代  |   Stop The World    |
| CMS收集器    | 标记-清除算法   |   并行   | 老年代  |    下面分析    |

Parallel Scavenge收集器可以达到一个可控制的吞吐量（吞吐量：CPU用于运行用户代码的时间与CPU总消耗时间的比值）。  
CMS收集器，目标是获取最短回收停顿时间。收集过程：
1. 初始标记  
1. 并发标记  
1. 重新标记  
1. 并发清除  

其中，初始标记、重新标记仍需要"Stop The World"，但速度快，用时很短；耗时长的并发标记和并发清除过程收集器线程可以和用户线程一起工作。总体上是并发运行的。  
优点：并发运行、低停顿  
缺点：
1. CMS收集器对CPU资源非常敏感。在并发阶段，会占用一部分CPU资源，导致程序变慢，总吞吐量降低。尤其在CPU很少时，对用户程序的影响可能变得很大；  
2. CMS收集器无法处理浮动垃圾。并发清理阶段用户线程还在运行，会产生新的垃圾，这部分垃圾在标记过程之后产生，此次回收无法处理它们；  
3. 基于“标记-清除”算法会产生大量空间碎片，无法找到大连续空间时会出发Full GC。  

G1(Garbage first)收集器是当前收集器技术发展的最前沿成果。
特点：
1. 并行与并发。缩短停顿时间，并可以与用户线程并发执行；  
2. 分代收集；  
3. 空间整合。整体上使用标记-整理算法，不会产生空间碎片；  
4. 可预测的停顿。  

G1收集器可以实现在基本不牺牲吞吐量的前提下完成低停顿的内存回收，这是由于它能够极力的避免全区域的垃圾收集，G1收集器将整个Java堆（包括新生代、老年代）划分为多个大小固定的独立区域（Region），并且跟踪各个区域里面的垃圾堆积的价值大小（回收所能获得的空间大小以及回收所需时间的经验值），在后台维护一个优先列表，每次根据允许的收集时间，优先回收价值最大的Region，这种优先级的小区域回收方式，保证了尽可能高的收集效率。  

### 内存分配与回收策略  

1. 对象优先分配在eden分区  
Eden区没有足够空间时，虚拟机将发起一次MinorGC。  
2. 大对象直接进入老年代  
大对象指的是需要大量连续空间的Java对象，例如很长的字符串和数组。
3. 长期存活的对象将进入老年代  
虚拟机给每个对象设置了一个年龄计数器，对象在Eden区出生，经历一次Minor GC之后存活成功进入Survivor区，并且年龄变为1，在Survivor区中每“熬”过一次Minor GC就成长一岁，年龄到15岁（默认）就会进入老年代。  
4. 动态对象年龄判定  
如果Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象直接进入老年代。  
5. 空间分配担保  
在Minor GC之前，虚拟机会先检查老年代的最大可用连续空间是否大于当前新生代所有对象总空间，如果成立，这个Minor GC是安全的。  
如果不成立，虚拟机会查看是否允许担保失败。假如允许担保失败，就会检查老年代最大可用的连续空间是否大于历次晋级到老年代的对象的平均大小，如果大于，则尝试进行一次Minor GC；如果小于，或者不允许冒险，则要进行一次Full GC。  

## 类文件结构  
Class文件是一组以8位字节为基础单位的二进制流。  
文件格式如图：  
![c]({{ "/img/post/JVM/3.png" | prepend: site.baseurl }} )
前4个字节成为“魔数”，是为了确保这个文件是否为一个能被虚拟机接受的Class文件。值为0xCAFEBABE。  
紧跟着的4个字节是版本号，第5,6字节是此版本号，第7,8字节是主版本号。  
比如我随便写了一个打印"Hello World!的代码，然后找到目录用javac命令进行编译，生成了class
文件。如图：
![d]({{ "/img/post/JVM/4.png" | prepend: site.baseurl }} )
1字节是8位，换成16进制表示为2位。即CA是一个字节，FE是一个字节。CAFEBABE共占4字节。  
如图前4个字节是魔数没有错。接着的0x0000表示次版本号0，0x0034表示主板本号52。（52-45+1=8）表示jdk版本为1.8.0。  
接着是常量池，前2个字节表示常量数量(从1开始),我写的程序是0x001d，转为十进制29，即有28个常量，索引值范围1-28。  
常量池中主要存放两大类常量：字面量和符号引用。  
字面量包括文本字符串和声明为final的常量值等。  
符号引用包括三类常量：类和接口的全限定名、字段的名称和描述符、方法的名称和描述符。  
上图中第一个常量标志位为0x0a(10)，查表可知是类中方法的符号引用。  
在常量池结束后，紧跟着的两个字节是访问标志，识别一些类或接口的访问信息，包括：这个Class是类还是接口；是否定义为public类型；是否定义为abstract类型；如果是类的话是否被声明为final等。  
接着是类索引、父类索引、接口索引集合，类索引确定这个类的全限定名，父类索引用于确定这个类的父类的全限定名。均为u2类型数据，接口索引是一组u2类型数据的集合。  
然后字段表集合、方法表集合、属性表集合。字段表用于描述接口或者类中声明的变量。  

## 虚拟机类加载机制  
虚拟机把描述类的数据从Class文件加载到内存，并对数据进行校验、转换解析和初始化，最终形成可以呗虚拟机直接使用的Java类型，是虚拟机的类加载机制。  
在Java语言里，类型的加载、连接和初始化过程都是在程序运行期间完成的。  
类从加载到虚拟机到卸载出内存的整个生命周期包括：加载，验证，准备，解析，初始化，使用，卸载。  
类加载的时机：  
1. 遇到new、getstatic、putstatic、或invokestatic这4条字节码指令时，如果类没有进行初始化，则需要先触发其初始化，生成这4条指令最常见的Java代码场景是：使用new关键字实例化对象的时候、读取或者设置一个类的静态字段（被final修饰、已经在编译期把结果放入常量池的静态字段除外）的时候，以及调用一个类的静态方法的时候；  
2. 使用java.lang.reflect包的方法对类进行反射调用的时候，如果类没有进行过初始化，则需要先触发其初始化；  
3. 当初始化一个子类的时候，如果发现其父类还没有进行过初始化，则需要先将父类的初始化；  
4. 当虚拟机启动时，用户需要指定一个要执行的主类（包含main方法的那个类），虚拟机会先初始化这个主类；

对于静态字段，只有直接定义这个字段的类才会被初始化，因此通过其子类来引用父类中定义的静态字段，只会触发其父类的初始化而不会触发子类的初始化。  
通过数组定义来引用类不会触发此类的初始化。  
常量在编译阶段会存入调用类的常量池中，不会触发定义常量类的初始化。  
### 加载  
通过一个类的全限定名来获取定义此类的二进制字节流；  
将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构；  
在Java堆中生成一个代表这个类的java.lang.Class对象，作为方法区这些数据的访问入口。  
### 验证  
验证是连接阶段的第一步，这一阶段的目的是为了确保Class文件的字节流中包含的信息符合当前虚拟机的要求，并且不会危害虚拟机自身的安全。  
虚拟机验证过程的四个阶段：  
文件格式验证；  
元数据验证；  
字节码验证；  
符号引用验证。  
### 准备  
准备阶段是正式为类变量分配内存并设置类变量初始值的阶段，这些内存都将在方法区中进行分配。这个阶段中有两个容易产生混淆的概念，首先是这时候进行内存分配的仅包含类变量（被static修饰的变量），而不包含实例变量，实例变量将会在对象实例化时随着对象一起分配在Java堆中。其次是这里所说的初始值“通常情况”下是数据类型的零值，假设一个类变量的定义为：`public static int value=123;`那么变量value在准备阶段过后的初始值为0，而不是123。  
### 解析  
解析阶段是虚拟机将常量池内的符号引用替换成直接引用的过程。  
符号引用：符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能无歧义的定位到目标即可。符号引用与虚拟机实现的内存布局无关。引用的目标并不一定已经加载到内存中。  
直接引用：直接引用可以是直接指向目标的指针、相对偏移量或者是一个能间接定位到目标的句柄。直接引用是与虚拟机实现的内存布局相关的，同一符号引用在不同虚拟机实例上翻译出来的直接引用一般不会相同。  
解析动作主要针对类或接口、字段、类方法、接口方法、方法类型、方法句柄和调用点限定符。  
### 初始化  
执行类构造器的过程。  
### 双亲委派模型  
从Java虚拟机的角度来讲，只存在两种不同的类加载器：一种是启动类加载器，是虚拟机自身的一部分；另一种是所有其他的类加载器，独立于虚拟机外部。  
双亲委派模型要求除了顶层的启动类加载器外，其余的类加载器都应当有自己的父类加载器。  
双亲委派模型的工作过程：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委托给父类加载器去完成。每一层的类加载器都是如此，因此所有的加载请求最终都应该传递到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需加载的类）时，子加载器才会自己尝试去加载。  
**优点：**  
Java类随着它的类加载器一起具备了一中带有优先级的层次关系。例如位于rt.jar包中的类java.lang.Object，无论哪个加载器加载这个类，最终都是委托给顶层的启动类加载器进行加载，确保了Object类在各种加载器环境中都是同一个类。  

如果自己尝试编写一个与rt.jar类库中已有类重名的Java类，将会发现可以正常编译，但永远无法被加载运行。  
## volatile变量
[关于JMM的博客](https://niantianlei.github.io/2017/12/21/JMM-analysis/)，已经做了详细的介绍，两个语义：保证可见性和禁止指令重排序。这里仅补充个应用场景：  
- 运算结果并不依赖变量的当前值，或者能够确保只有单一的线程修改变量的值。  
- 变量不需要与其他的状态变量共同参与不变约束。  
实际上，就是独立于程序。  

## Java线程调度
线程调度是指系统为线程分配处理器使用权的过程，主要调度方式有两种：协同式调度和抢占式调度。  

协同式调度：线程的执行时间由线程本身来控制，线程把自己的工作执行完了之后，要主动通知系统切换到另外一个线程上。  
优点：实现简单，线程进行完毕才进行线程切换，不会产生线程同步的问题。  

抢占式调度：每个线程由系统分配执行时间，线程的切换不由线程本身决定。Java使用的线程调度方式。  


