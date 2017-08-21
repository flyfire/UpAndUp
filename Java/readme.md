#java内部类实现原理

#泛型擦除

#java多线程实现方式

#ArrayList,LinkedList实现原理

#HashMap实现原理

#ConcurrencyHashMap实现原理


#ReentrantLock实现原理


#java锁，高并发


##JVM
JVM内存结构：程序计数器，JVM方法栈，Native方法栈，JVM方法区，堆。
JVM内存回收策略：
标记清除:首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。容易造成内存碎片。
标记整理:
标记整理法首先需要标记出存活的对象，然后所谓的整理就是把这些存活的对象往一端推。然后就清除边界以外的区域即可。

老年代的垃圾回收器（例如 Serial Old，Parallel Old，到那时不包括CMS，CMS使用的是标记清除法）都是采用这个算法，主要由于老年代的对象都比较持久，不是短暂的。

这样一看，每次整理，将不会产生内存碎片问题，因为也没有分配对象需要查空闲链表了。

伪代码实现

只要涉及到对象的移动，都会存在一个问题，就是假如 A -> B（A 引用 B，A 移动后的对象为 A',B 移动后的对象为 B'，），那么怎么保证 A'-> B'。 其实这里还是利用了一个对象的属性 forwarding（参考：第3章：垃圾收集算法 - 复制算法（伪代码实现与深入分析））。

forwarding 的意思就是 它是 A对象的一个属性，然后记录了 A 对象移动到 A'。forwarding 会记录这个 A'，但是实际上还没移动，就是先记录好。

整个过程分 3 步：

首先，需要设置这个 forwarding，用于知道 A 后面移动到 A'，B 后面移动到 B'，这样才有可能把 A' -> B' 关联上。

set_forwarding_ptr(){
    scan = new_address = $heap_start
    while(scan < $heap_end){
        if(scan.mark == TRUE){
            scan.forwarding = new_address
            new_address += scan.size
        }
        scan += scan.size
    }
}
这段代码感觉还算比较好理解的，也就是记录自己移动后是到哪个位置：scan.forwarding = new_address。（注意，这里只是预先记录一下，还没有真正开始移动）

步骤二，更新指针：

adjust_ptr(){
    for(r : $roots){
        *r = (*r).forwarding
    }

    scan = $heap_start
    while(scan < $heap_end){
        if(scan.mark == TRUE){
            for(child : children(scan)){
                *child = (*child).forwarding
            }
        }
        scan += scan.size
    }
}
这段代码主要说的就是重写 A' 对象移动后指向哪个对象。例如 *child = (*child).forwarding ，这里可以看到 A原来是引用B，(*child).forwarding 表示 B 到时候会变成 B'，此时需要把 child 改成以后的 B' ，那么，此时在 A对象中，已经记录了之后A会变到 A'(forwarding指针记录了)，然后上面那个代码又知道了引用的 B 对象以后要变到哪里去，所以说，最后移动的时候，A' -> B'就很容易关联上。

步骤三：真正的移动

move_obj(){
    sacn = $free = $heap_start
    while(scan < $heap_end){
        if(scan.mark == TRUE){
            new_address = scan.forwarding
            copy_data(new_address,scan,scan,size)
            new_address.forwarding = NULL
            new_address.mark = FALSE
            $free += new_address.size
            scan += scan.size
        }
    }
}
这段主要就是先 copy：copy_data(new_address,scan,scan,size) ，然后位置一直往前挪动：$free += new_address.size

优缺点

耗费的时间在哪里

系统怎么知道A对象后来去到 A' ，那么，这个是需要遍历整个堆的。比如说，发现第一个位置需要GC了，然后 A 不需要，那么就把 A.forwarding 设置到第一个位置这里。所以这里要遍历一遍堆。
设置 A' -> B' 的关联。这个从上面代码实现来看也要遍历一遍堆：while(scan < $heap_end)。但是个人表示怀疑，因为根据 forwarding 知道了 A 变到 A' ，也能由A得到 B，再由 B 的 forwarding 可以得到 B'，所以感觉这里应该可以设置 A' -> B'。
移动对象，这个是肯定要遍历一遍堆的。因为涉及到移动，清除。
所以总的来说，这个算法的效率无论比起复制算法还是标记清除法，遍历的次数都是要多一点。

没有内存碎片问题

所以说分配内存的时候也不需要到空闲链表这个东西

复制算法主要用于新生代中，例如作用于新生代的垃圾处理器：Serial，ParNew，Parallel Scavenge 垃圾收集器，主要因为新生代的对象的存活时间比较短，所以采用这个算法折衷起来是比较好的。

复制算法原理

主要把 内存等分成A，B两块，每次只使用其中的一块，那么，每次垃圾回收的时候，就把使用中的半块（例如A）的存活的对象移动到另外的半块中，然后，直接清除另外的半块（B块）。

这样马上就会有一个问题 ==》岂不是太浪费内存了？

对，等分会非常浪费，所以说现在 JVM 的新生代不是等分的，而是 8：1：1 的比例，每次使用 Eden 区域和其中 1 块 Survivor 区域，垃圾回收的时候，那么就把存活的对象全部移动到另外 1 块 Survivor 区域。

(Ps：如果另外一块 Survivor 区域不够怎么办。这里老年代会进行一个分配担保。（简单来说就是不够了对象直接进入老年代）)

那么，另外可以看出，一个优点就是实现简单，而且没有内存碎片的问题（之后提到的标记清除法（老年代主要的垃圾算法）就会有这个问题）

算法的伪代码实现

《深入理解 Java 虚拟机》这部分的内容没有涉及到代码，参考 《垃圾回收的算法与实现》一书，再深入理解一下这个算法：

copying(){
    $free = $to_start
    for(r : $roots){
        *r = copy(*r)
    }
    swap($from_start, $to_start)
}
首先可以得到所有的 GC roots 对象，也就是所有的根对象，然后对每一个根对象进行复制（此时的复制包括了该根对象的所有子对象）。

copy(obj){
    if(obj.tag != COPIED){
        copy_data($free,obj,obj,size)
        obj.tag = COPIED
        obj.forwarding = $free
        $free += obj.size
        for(child : children(obj.forwarding)){
            *child = copy(*child)
        }
    }
    return obj.forwarding
}
copy_data($free,obj,obj,size) 表示把对象复制到另外的一个区域。

这里如果深入思考的话可能会有个疑惑，就是 比如 A -> B （A引用B），假设 A，B移动到另外的区域后是 A' 和 B'，怎么保证是 A' -> B'（A' 引用 B'） ，而不会混乱呢（比如 A' -> B ）。

解决办法就是利用对象的一个属性 forwarding。这个属性就记录了 A移动到另外一个区域，变成 A' 后的位置是什么，可以看到代码 obj.forwarding = $free。

这里再顺一遍上面的代码，假设只有 A -> B（A 引用 B，A是 GC roots）。那么：

调用 copying() 开始复制。
A 对象先复制。假设 A 复制到另外 1 块区域的地址是 A'，此时会记录下 A.forwarding = A'，此时要注意一点：A' 引用的对象还是 B，因为 B' 还没有呢。
然后复制 A 的子对象 B，同理。最后 B 对象返回 return obj.forwarding 也就是 B 的新地址 B'。
此时又回到 A 流程的 *child = copy(*child) ，那么 A' -> B' 就这样关联上了。
所以说，具体的整个复制流程就是这样子的。

优缺点分析

耗费的时间主要在哪里

复制的时候需要遍历 GC roots，找到所有的活动对象。
清除的时候需要遍历整个堆
这个其实还是比较高效的。

不会发生碎片化

不会发生碎片化意味着什么，意味着分配内存的时候不需要遍历空闲链表。对于某一块内存，系统怎么知道这个内存有多大，能不能分配给一个对象，其实是在空闲链表里有维护。那么，分配对象的时候就会遍历这个空闲链表，比如说，某某某某区域有一个内存，够大，可以分配，好，那么就用这个内存了。

缺点 - 浪费了一定的内存

缺点 - 递归调用

递归调用会有栈溢出风险。

PC 寄存器：可以看做是当前线程所执行的字节码的行号指示器。它属于每个线程所私有的。
JVM 方法栈：由一条条的栈帧组成，每个栈帧又包括了局部变量区，操作数区，等。每个方法在执行的时候就会创建一个栈帧（所以说递归会产生大量的栈帧，假如此时又死循环了，那么这里栈帧过多，就会造成 stackoverflow 栈溢出异常。）它也是线程所私有的。
本地方法栈：主要用于支持 java 语言和其他语言（如C语言）的一个交互。
JVM 方法区：存储 JVM 加载了的 类的信息，常量，静态变量等。简单来说就相当于平常所说的永久代。（实际上肯定有区别，但是对于新手来说可以暂且认为差不多，相等）。这部分区域不是线程所私有，而是各个线程所共享的。
JVM 堆：这里主要用于存储对象和数组等实例。是 Java 最重要的一块存储区域，平常所说的堆内存，新生代（eden，survivor区域），老年代就是在这个区域中。因此，这里也是垃圾收集的主要发生的场所。

对于JVM 的内存区域，还有一个经常谈论的地方就是 JVM堆，主要分为了新生代和老年代。其中：

新生代又分为了 1 个 Eden 区域和 2 个 Survivor 区域，比例是 8：1：1。这样的一个划分主要是因为新生代采用的垃圾回收算法为复制算法而导致的。

对于老年代，和新生代的比例默认为 2：1。

对于一些调整的参数这里罗列总结一下：

-XX: Persize：设置非堆内存初始值，默认为1/64。(JDK7的JVM方法区内存)
-XX:MaxPersize：设置非堆内存最大值，默认为1/4。(JDK7的JVM方法区内存)
-Xss：设置每个线程占用堆内存大小，现在默认为1M，以前为256K。设置线程越小，堆内存大小不变，可以创造的线程数越多（当然有一个限度）。但是这个值也需要经过严格的测试再设置。
-Xms：JVM堆初始分配内存。默认为物理内存的1/64，当默认堆内存的空余空间小于40%的时候，这个堆内存就会自动增长到-Xmx指定的最大堆分配内存。
-Xmx ：JVM的最大堆分配内存。默认为物理内存的1/4，当空余内存大于70%的时候，该堆内存又会自动减少到-Xms指定的内存。
-Xmn：指定新生代的内存大小。当堆大小不变的情况下，新生代越大，老生代越小，默认新生代和老年代的比例为1：2，而且这个比例会严重影响系统性能，sun推荐为新生代占整个堆内存3/8。
-XX:SurvivorRatio：新生代中，又分为eden区域和两个Survivor区域。默认比例为8:1:1，也就是说，可以用的内存为90%。该参数用来设置eden和Survivor的比值，默认为8:1。
-XX:NewRatio：年轻代(包括Eden和两个Survivor区)与年老代的比值(除去持久代)

Java对象是如何创建

当我们使用 new 的时候。首先虚拟机会检查该类有没有被加载，如果已经被加载了，那么就能够确定下所需要分配的内存空间的大小。

虚拟机在堆内存中进行分配的时候，有两种方式：

指针碰撞 ：用了的内存放在一边，那么用了的内存和没用的内存中间会有一个指针，然后分配内存的时候直接移动该指针即可。
空闲列表 ：用的内存和没用的内存交错着，这个时候需要虚拟机维护一个空闲列表，然后分配内存的时候就从空闲列表中选取一个内存区域进行分配，最后更新列表。
那么具体用哪种方式是由 垃圾回收器进行决定， 比如说 Serial，ParNew这种回收器使用的就是指针碰撞的方式（也就是带了压缩功能的）。像 CMS 这种垃圾回收器就使用 空闲列表的方式。

TLAB

那么在分配内存的时候，可能会出现多线程问题，解决方式有 CAS 乐观锁，以及 TLAB方式。TLAB的全称是 Thread Local Allocation Buffer。

每个线程 JVM在内存新生代Eden Space中开辟了一小块线程私有的区域，称作TLAB（Thread-local allocation buffer）。默认设定为占用Eden Space的1%。在Java程序中很多对象都是小对象且用过即丢，它们不存在线程共享也适合被快速GC，所以对于小对象通常JVM会优先分配在TLAB上，并且TLAB上的分配由于是线程私有所以没有锁开销。因此在实践中分配多个小对象的效率通常比分配一个大对象的效率要高。

也就是说，Java中每个线程都会有自己的缓冲区称作TLAB（Thread-local allocation buffer），每个TLAB都只有一个线程可以操作，TLAB结合bump-the-pointer技术可以实现快速的对象分配，而不需要任何的锁进行同步，也就是说，在对象分配的时候不用锁住整个堆，而只需要在自己的缓冲区分配即可。

初始化

以上说到 具体虚拟机是如何分配出一块内存的，那么分配完之后，虚拟机层面上首先要进行初始化，这样即使我们代码中不初始化，也可以使用默认值。虚拟机全部赋0初始化之后，就轮到了我们代码层面上的初始化，比如说构造函数中我们自定义的初始化。

这两部做完之后，整个对象的创建就算完成了。

Java对象是如何定位

定位的方式也分为两种：

句柄：也就是说，在堆内存中还分了一小块区域，作为句柄的内存，我们 reference 指向的不是具体的对象，而是这个句柄的地址，再由这个句柄指向我们的对象。 那么这样的好处就是说 GC的时候，对象的移动不会影响到 reference。（有点像代理的意思）
直接指针：也就是说， reference 会直接指向该对象，这样的一个好处自然是快。

编译机制

编译主要是把 .java 文件转换为 .class 文件。 其中转换后的 .class 文件就包含了元数据，方法信息等一些信息。比如说元数据就包含了 Java 文件中声明的常量，也就是我们所说的常量池。

Ps：在编译的时候，应该可以实现一个规范，这个规范可以使得在编译的时候执行一段你自己写的程序。例如 Lombok 的 @DaTa 注解帮我们产生 get set 方法的原理就是这个。

类加载机制

JVM 是通过 一个称为 ClassLoader 的东西 来加载 class 文件，每当 JVM 启动的时候，就会产生 三个 ClassLoader，它们分别是Bootstrap Loader， ExtClassLoader 和 AppClassLoader。 这三个 ClassLoader 的作用是不同的，它们所加载的class 文件也不同。

BootStrap Loader：它所加载的都是 JVM 中最底层的类。
ExtClassLoader：用于加载 Java 的一些库。
AppClassLoader：它主要由 java.class.path 来指定。
JVM 加载 class 文件的时候， 采用了双亲委托模型 ，三个加载器的委托模式如下图所示：
委托模式图

工作过程是：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成， 每一个层次的类加载都是如此 ，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法加载这个加载请求的时候，子加载器才会尝试自己去加载。

那么，对于加载 class 文件， ClassLoader 如何将 class 文件变成一个 java 类的。在 ClassLoader中有一个非常重要的方法叫 findClass() 方法，它接受要加载的类作为它的参数，在该方法中会找到class文件并且读取文件中的内容到一个 byte 数组，然后再调用 另外一个重要的方法：defineClass()，该方法能够把 byte 数组中的内容转化为一个相应的 Class Object。

委托机制，指先委托父转载器寻找目标类， 只有在找不到的情况下才从自己的类路径中查找并装载类，这一点是从安全角度考虑的 ， 试想如果有人编写了一个恶意的基础类，比如String类，并装载到JVM中将会引起多么可怕的后果呢。但是，由于有了全盘负责委托机制，String类 永远是有根装载器装载，这样就避免了事件的发生。另外，我们经常遇到的NoSuchMethodError的错误往往就是因为加载了不同版本的包造成的。

自定义类加载器

默认的类加载器只知道如何从本地系统加载类。如果我们的程序只是在本机跑的话，一般来说默认加载器可以应付。 但是如果我们需要加载远程服务器的类，或者说加载网络上的，不是本机上的类的时候，就需要到自定义类加载器。

实现方式：

自定义类加载器可以选择 继承ClassLoader类，重写里面的方法来实现。里面有三个重要的方法，一个是loadClass()方法，一个是findClass()方法，一个是defineClass()方法。

对于loadClass(String name,Boolean resolve)方法：不推荐重写，因为loadClass()方法做的工作主要为实现双亲委托模型，我们重写的话可能会破坏该模型，并且也增加自己的开发难度。
对于defineClass(String name,byte[] b,int off,int len)方法，主要用于将原始字节转换为Class对象，也不需要我们重写。
对于findClass(String name)方法，根据名称来查找类。把.class文件的内容放入一个byte[]数组中，供defineClass()方法使用。一般我们就重写这个方法。在这个方法中，我们重新定义从哪里，怎么样读取文件内容。这样我们就可以把从本机上改为从网络上得到类，从而实现自定义类加载器。
总而言之：

继承ClassLoader类。
重写findClass(String name)方法。 在这个方法里面我们重新定义从哪里，怎么读取文件内容到byte[]数组中，供defineClass()使用。
例子：

```
public class Sample{      
    public int v1 = 1;      
    public Sample(){      
        System.out.println(“Sample is load by :” + this.getClass().getClassLoader());      
    }      
}      
import java.io.ByteArrayOutputStream;      
import java.io.File;      
import java.io.FileInputStream;      
import java.io.InputStream;      

public class MyClassLoader extends ClassLoader{      
    private String name;      
    private String path = ”d:\\”;      
    private final String fileType = ”.class”;      

    public MyClassLoader(String name){      
        super();      
        this.name = name;      
    }      
    public MyClassLoader(ClassLoader parent, String name){      
        super(parent);      
        this.name = name;      
    }      
    @Override      
    public String toString(){return this.name;}        
    public String getName(){return name;}      
    public void setName(String name){this.name = name;}        
    public String getPath(){return path;}          
    public void setPath(String path){this.path = path;}       
    public String getFileType(){return fileType;}          

    protected Class<?> findClass(String name) throws ClassNotFoundException{      
        byte[] data = this.loadClassData(name);       
        return this.defineClass(name, data, 0, data.length);      
    }      
    private byte[] loadClassData(String name){      
        InputStream is = null;      
        byte[] data = null;      
        ByteArrayOutputStream baos = null;      
        try{      
            this.name = this.name.replace(“.”, ”\\”);      
            is = new FileInputStream(new File(path + name + fileType));      
            baos = new ByteArrayOutputStream();      
            int ch = 0;      
            while (-1 != (ch = is.read())){      
                baos.write(ch);      
            }      
            data = baos.toByteArray();      
        }catch (Exception e){      
            e.printStackTrace();      
        }finally{      
            try{      
                is.close();      
                baos.close();      
            }catch (Exception e){      
                e.printStackTrace();      
            }      
        }      
        return data;      
    }      
    public static void showClassLoader(ClassLoader loader) throws Exception{      
        Class clazz = loader.loadClass(“Sample”);      
        Object object = clazz.newInstance();      
    }      
    public static void main(String[] args) throws Exception{      
        MyClassLoader loader1 = new MyClassLoader(“loader1″);      
        loader1.setPath(“d:\\loader1\\”);      
        MyClassLoader loader2 = new MyClassLoader(loader1, ”loader2″);      
        loader2.setPath(“d:\\loader2\\”);      
        MyClassLoader loader3 = new MyClassLoader(null, ”loader3″);      
        loader3.setPath(“d:\\loader3\\”);      
        showClassLoader(loader2);      
        showClassLoader(loader3);      
    }      
}      
```

线程之间的共享变量存储在主内存（main memory）中，每个线程都有一个私有的本地内存（local memory），本地内存中存储了该线程以读/写共享变量的副本。

本地内存是JMM（Java内存模型）的一个抽象概念，并不真实存在。它涵盖了缓存，写 缓冲区，寄存器以及其 他的硬件和编译器优化。

lock可以说是 synchronized 的一个替代品，synchronized 能做的事，lock 基本都可以做，而且能做得更好。他们的一些区别是：

lock在获取锁的过程可以被中断。
lock可以尝试获取锁，如果锁被其他线程持有，则返回 false，不会使当前线程休眠。
lock在尝试获取锁的时候，传入一个时间参数，如果在这个时间范围内，没有获得锁，那么就是终止请求。
synchronized 会自动释放锁，lock 则不会自动释放锁。
这样可以看到，lock 比起 synchronized 具有更细粒度的控制。但是也不是说 lock 就完全可以取代 synchronized，因为 lock 的学习成本，复杂度等方面要比 synchronized 高，对于初级 java 程序员，使用 synchronized 的风险要比 lock 低。

目录

包括了：

Lock 接口方法分析
RentrantLock
ReadWriteLock
Java Lock接口源码分析

Lock 接口方法如下：

public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
lock，unlock 方法

lock() 可以用于对一段代码进行加锁，这样别的代码在锁释放之前需要进行等待，需要注意， lock不会像 synchronized 那样自动释放锁 ，所以： 一定要放在 try-finally块中，保证锁的释放。 例如：

try {
    lock.lock();
    ......
} finally {
    lock.unlock();  
}
tryLock 方法

tryLock()：尝试获得锁，如果成功，返回 true，否则，返回 false。
tryLock(long time,TimeUnit unit)：在一定的时间内尝试获得锁，并且在这段时间直接可以被打断。如果成功获得，那么将返回 true，否则，返回 false。
lockInterruptibly 方法

这里首先需要了解两个概念才能更好的理解这个方法：

线程的打断机制
Thread类的interrupt,interrupted,isInterrupted方法的区别
对于线程的打断机制， 每个线程都有一个打断标志。

如果线程在 sleep 或 wait 或 join 的时候，此时如果别的进程调用此线程的 interrupt() 方法，此线程会被唤醒并被要求处理InterruptedException；
如果线程在运行，则不会收到提醒。但是 此线程的 “打断标志” 会被设置。
所以说，对于 interrupt() 方法：

不会中断一个正在运行的线程 。
不会中断一个正在运行的线程 。
不会中断一个正在运行的线程 。
对于 interrupt,interrupted,isInterrupted方法的区别：

interrupt 方法上面有说到了。对于 interrupted 和 isInterrupted 方法,stackoverflow 说得很好了：

interrupted() is static and checks the current thread. isInterrupted() is an instance method 
which checks the Thread object that it is called on.

A common error is to call a static method on an instance.

Thread myThread = ...;
if (myThread.interrupted()) {} // WRONG! This might not be checking myThread.
if (myThread.isInterrupted()) {} // Right!

Another difference is that interrupted() also clears the status of the current thread. 
In other words, if you call it twice in a row and the thread is not interrupted 
between the two calls, the second call will return false even if the first call returned true.

The Javadocs tell you important things like this; use them often!
下面再介绍下 lockInterruptibly 方法：

当通过这个方法去获取锁时，如果线程正在等待获取锁，则这个线程能够响应中断，即中断线程的等待状 态。例如当两个线程同时通过lock.lockInterruptibly()想获取某个锁时，假若此时线程A获取到了锁，而线程B只有在等待，那 么对线程B调用threadB.interrupt()方法能够中断线程B的等待过程。

newCondition()

用于获取一个 Conodition 对象。Condition 对象是比 Lock 更细粒度的控制。要很好的理解 condition，个人觉得必须要知道，生产者消费者问题。

简单来说就是，我们都了解 生成者在缓冲区满了的时候需要休眠，此时会再唤起一个线程，那么你此时唤醒的是生成者还是消费者呢，如果是消费者，很好；但是如果是唤醒生产者，那还要再休眠，此时就浪费资源了。condition就可以用来解决这个问题，能保证每次唤醒的都是消费者。具体参考：Java 多线程：Condition 接口

lock 方法大体就介绍到这里。

ReentrantLock

可重入锁：指同一个线程，外层函数获得锁之后，内层递归函数仍有获得该锁的代码，但是不受影响。

可重入锁的最大作用就是 可以避免死锁。 例如：A线程 有两个方法 a 和 b，其中 a 方法会调用 b 方法，假如 a，b 两个方法都需要获得锁，那么首先 a 方法先执行，会获得锁，此时 b方法将永远获得不了锁，b 方法将一直阻塞住， a 方法由于 b 方法没有执行完，它本身也 不释放锁，此时就会造成一个 死锁。
ReentrantLock 就是一个可重入锁。真正使用 锁的时候，一般是 Lock lock ＝ new ReentrantLock()； 然后 使用 Lock 接口方法。

ReadWriteLock

接口代码如下：

public interface ReadWriteLock {  
    Lock readLock();  
    Lock writeLock();  
}  
ReadWriteLock 可以算是 Lock 的一个细分，合理使用有利于提高效率。比如说， 对于一个变量 i， A，B 线程同时读，那么不会造成错误的结果，所以此时是允许并发，但是如果是同时写操作，那么则是有可能造成错误。所以真正使用的时候，可以使用 细分需要的是读锁还是写锁，再相应地进行加锁。

Ps： 从代码也可以看出，ReadWriteLock 和 Lock 没有关系，既不继承，也不是实现。

Condition 是一种更细粒度的并发解决方案。

就拿生产者消费者模式来说，当仓库满了的时候，又再执行到 生产者 线程的时候，会把 该 生产者 线程进行阻塞，再唤起一个线程.

但是此时唤醒的是消费者线程还是生产者线程，是未知的。 如果再次唤醒的还是生产者线程，那么还需要把它进行阻塞，再唤起一个线程，再此循环，直到唤起的是消费者线程。这样就可能存在 时间或者资源上的浪费，

所以说 有了Condition 这个东西。Condition 用 await() 代替了 Object 的 wait() 方法，用 signal() 方法代替 notify() 方法。 注意：Condition 是被绑定到 Lock 中，要创建一个 Lock 的 Condition 必须用 newCondition() 方法。

Condition 在生产者消费者模型中的代码Demo

class BoundedBuffer {      
    final Lock lock = new ReentrantLock();//锁对象      
    final Condition notFull  = lock.newCondition();//写线程条件       
    final Condition notEmpty = lock.newCondition();//读线程条件       

    final Object[] items = new Object[100];//缓存队列      
    int putptr/*写索引*/, takeptr/*读索引*/, count/*队列中存在的数据个数*/;      

    public void put(Object x) throws InterruptedException {      
        lock.lock();      
        try {      
            while (count == items.length)//如果队列满了       
                notFull.await();//阻塞写线程      
            items[putptr] = x;//赋值       
            if (++putptr == items.length) putptr = 0;//如果写索引写到队列的最后一个位置了，那么置为0      
            ++count;//个数++      
            notEmpty.signal();//唤醒读线程      
        } finally {      
            lock.unlock();      
        }      
    }      

    public Object take() throws InterruptedException {      
        lock.lock();      
        try {      
            while (count == 0)//如果队列为空      
                notEmpty.await();//阻塞读线程      
            Object x = items[takeptr];//取值       
            if (++takeptr == items.length) takeptr = 0;//如果读索引读到队列的最后一个位置了，那么置为0      
            --count;//个数--      
            notFull.signal();//唤醒写线程      
            return x;      
        } finally {      
            lock.unlock();      
        }      
    }       
} 
这是一个处于多线程工作环境下的缓存区，缓存区提供了两个方法，put和take，put是存数据，take是取数据，内部有个缓存队列，具体变量和方法说明见代码，这个缓存区类实现的功能：有多个线程往里面存数据和从里面取数据，其缓存队列(先进先出后进后出)能缓存的最大数值是100，多个线程间是互斥的，当缓存队列中存储的值达到100时，将写线程阻塞，并唤醒读线程，当缓存队列中存储的值为0时，将读线程阻塞，并唤醒写线程，下面分析一下代码的执行过程：

一个写线程执行，调用put方法；
判断count是否为100，显然没有100；
继续执行，存入值；
判断当前写入的索引位置++后，是否和100相等，相等将写入索引值变为0，并将count+1；
仅唤醒读线程阻塞队列中的一个；
一个读线程执行，调用take方法；
……
仅唤醒写线程阻塞队列中的一个。
这样由于阻塞该阻塞的，唤醒该唤醒的，阻塞该阻塞的，所以能提高效率。

synchronized 用法

它的修饰对象有几种：

修饰一个类，其作用的范围是synchronized后面括号括起来的部分， 作用的对象是这个类的所有对象。
修饰一个方法，被修饰的方法称为同步方法，其作用的范围是整个方法， 作用的对象是调用这个方法的对象；
修改一个静态的方法，其作用的范围是整个静态方法， 作用的对象是这个类的所有对象；
修饰一个代码块，被修饰的代码块称为同步语句块，其作用的范围是大括号{}括起来的代码， 作用的对象是调用这个代码块的对象；
修饰一个类

其作用的范围是synchronized后面括号括起来的部分，作用的对象是这个类的所有对象，如下代码：

class ClassName {
   public void method() {
      synchronized(ClassName.class) {
         // todo
      }
   }
}
修饰一个方法

synchronized 修饰一个方法很简单，就是在方法的前面加synchronized，例如：

public synchronized void method()
{
   // todo
}
另外，有几点需要注意：

在定义接口方法时不能使用synchronized关键字。
构造方法不能使用synchronized关键字，但可以使用synchronized代码块来进行同步。
synchronized 关键字不能被继承 。如果子类覆盖了父类的 被 synchronized 关键字修饰的方法，那么子类的该方法只要没有 synchronized 关键字，那么就默认没有同步，也就是说，不能继承父类的 synchronized。
修饰静态方法

我们知道 静态方法是属于类的而不属于对象的 。同样的， synchronized修饰的静态方法锁定的是这个类的所有对象 。如下：

public synchronized static void method() {
   // todo
}
修饰代码块

当两个并发线程访问同一个对象object中的这个synchronized(this)同步代码块时，一个时间内只能有一个线程得到执行。另一个线程必须等待当前线程执行完这个代码块以后才能执行该代码块。
当一个线程访问object的一个synchronized(this)同步代码块时，另一个线程仍然可以访问该object中的非synchronized(this)同步代码块。
尤其关键的是，当一个线程访问object的一个synchronized(this)同步代码块时，其他线程对object中所有其它synchronized(this)同步代码块的访问将被阻塞。
第三个例子同样适用其它同步代码块。也就是说，当一个线程访问object的一个synchronized(this)同步代码块时，它就获得了这个object的对象锁。结果，其它线程对该object对象所有同步代码部分的访问都被暂时阻塞。
以上规则对其它对象锁同样适用.
JDK中对 synchronized 的使用举例

对于 synchronized ，个人觉得是一种比较粗糙的加锁，尤其是对整个对象，或者整个类进行加锁的时候。例如，HashTable，它相当于 HashMap的线程安全版本。实现如下：

public synchronized V get(Object key) {
        Entry<?,?> tab[] = table;
        int hash = key.hashCode();
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<?,?> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                return (V)e.value;
            }
        }
        return null;
    }
可以看到，很多方法都是使用了 synchronized 进行了修饰，那么就意味如果还有别的同步方法x，这个x方法和get方法即使在没有冲突的情况下也需要等待执行。这样效率上 必然会有一定的影响，所以会有 ConcurrentHashMap 进行分段加锁。

另外，在JDK中，Collections有一个方法可以把不是线程安全的集合转化为线性安全的集合，它是这样实现的：

public static <T> Collection<T> synchronizedCollection(Collection<T> c) {
        return new SynchronizedCollection<>(c);
    }
 static class SynchronizedCollection<E> implements Collection<E>, Serializable {
        private static final long serialVersionUID = 3053995032091335093L;

        final Collection<E> c;  // Backing Collection
        final Object mutex;     // Object on which to synchronize

        SynchronizedCollection(Collection<E> c) {
            this.c = Objects.requireNonNull(c);
            mutex = this;
        }

        SynchronizedCollection(Collection<E> c, Object mutex) {
            this.c = Objects.requireNonNull(c);
            this.mutex = Objects.requireNonNull(mutex);
        }

        public int size() {
            synchronized (mutex) {return c.size();}
        }

        ......
}
可以看到 在构造函数中 有 mutex = this, 然后在具体的方法使用了 synchronized(mutex)，这样就会对调用该方法的对象进行加锁。还是会出现HashTable 出现的那种问题，也就是效率不高。

悲观锁与乐观锁

我们都知道，cpu是时分复用的，也就是把cpu的时间片，分配给不同的thread/process轮流执行，时间片与时间片之间，需要进行cpu切换，也就是会发生进程的切换。切换涉及到清空寄存器，缓存数据。然后重新加载新的thread所需数据。当一个线程被挂起时，加入到阻塞队列，在一定的时间或条件下，在通过notify()，notifyAll()唤醒回来。

在某个资源不可用的时候，就将cpu让出，把当前等待线程切换为阻塞状态。等到资源(比如一个共享数据）可用了，那么就将线程唤醒，让他进入runnable状态等待cpu调度。这就是典型的悲观锁的实现。 独占锁是一种悲观锁，synchronized就是一种独占锁，它假设最坏的情况，并且只有在确保其它线程不会造成干扰的情况下执行，会导致其它所有需要锁的线程挂起，等待持有锁的线程释放锁。

但是，由于在进程挂起和恢复执行过程中存在着很大的开销。 当一个线程正在等待锁时，它不能做任何事，所以悲观锁有很大的缺点。举个例子，如果一个线程需要某个资源，但是这个资源的占用时间很短，当线程第一次抢占这个资源时，可能这个资源被占用，如果此时挂起这个线程，可能立刻就发现资源可用，然后又需要花费很长的时间重新抢占锁，时间代价就会非常的高。

所以就有了乐观锁的概念，他的核心思路就是，每次不加锁而是假设没有冲突而去完成某项操作，如果因为冲突失败就重试，直到成功为止。 在上面的例子中，某个线程可以不让出cpu,而是一直while循环，如果失败就重试，直到成功为止。所以，当数据争用不严重时，乐观锁效果更好。 比如CAS就是一种乐观锁思想的应用。

Java中CAS的实现

CAS就是Compare and Swap的意思，比较并操作。很多的cpu直接支持CAS指令。CAS是项乐观锁技术，当多个线程尝试使用CAS同时更新同一个变量时，只有其中一个线程能更新变量的值，而其它线程都失败， 失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次尝试。

CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

JDK1.5中引入了底层的支持，在int、long和对象的引用等类型上都公开了CAS的操作，并且JVM把它们编译为底层硬件提供的最有效的方法，在运行CAS的平台上，运行时把它们编译为相应的机器指令。在java.util.concurrent.atomic包下面的所有的原子变量类型中，比如AtomicInteger，都使用了这些底层的JVM支持为数字类型的引用类型提供一种高效的CAS操作。

ABA 问题

在CAS操作中，会出现ABA问题。 就是如果V的值先由A变成B，再由B变成A，那么仍然认为是发生了变化，并需要重新执行算法中的步骤。

有简单的解决方案：不是更新某个引用的值，而是更新两个值，包括一个引用和一个版本号，即使这个值由A变为B，然后为变为A，版本号也是不同的。

AtomicStampedReference和AtomicMarkableReference支持在两个变量上执行原子的条件更新。AtomicStampedReference更新一个“对象-引用”二元组，通过在引用上加上“版本号”，从而避免ABA问题，AtomicMarkableReference将更新一个“对象引用-布尔值”的二元组。

AtomicInteger的实现

AtomicInteger 是一个支持原子操作的 Integer 类 ，就是保证对AtomicInteger类型变量的增加和减少操作是原子性的，不会出现多个线程下的数据不一致问题。如果不使用 AtomicInteger，要实现一个按顺序获取的 ID，就必须在每次获取时进行加锁操作，以避免出现并发时获取到同样的 ID 的现象。

接下来通过源代码来看AtomicInteger具体是如何实现的原子操作。首先看incrementAndGet() 方法，下面是具体的代码：

public final int incrementAndGet() {  
        for (;;) {  
            int current = get();  
            int next = current + 1;  
            if (compareAndSet(current, next))  
                return next;  
        }  
}  
通过源码，可以知道，这个方法的做法为先获取到当前的 value 属性值，然后将 value 加 1，赋值给一个局部的 next 变量，然而，这两步都是非线程安全的，但是内部有一个死循环，不断去做compareAndSet操作，直到成功为止，也就是修改的根本在compareAndSet方法里面，compareAndSet()方法的代码如下：

public final boolean compareAndSet(int expect, int update) {  
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);  
}  
compareAndSet()方法调用的compareAndSwapInt()方法的声明如下，是一个native方法:
publicfinal native boolean compareAndSwapInt(Object var1, long var2, int var4, intvar5);

compareAndSet 传入的为执行方法时获取到的 value 属性值，next 为加 1 后的值， compareAndSet所做的为调用 Sun 的 UnSafe 的 compareAndSwapInt 方法来完成，此方法为 native 方法，compareAndSwapInt 基于的是CPU 的 CAS指令来实现的。 所以基于 CAS 的操作可认为是无阻塞的，一个线程的失败或挂起不会引起其它线程也失败或挂起。并且由于 CAS 操作是 CPU 原语，所以性能比较好。

类似的，还有decrementAndGet()方法。它和incrementAndGet()的区别是将 value 减 1，赋值给next 变量。

AtomicInteger中还有getAndIncrement() 和getAndDecrement() 方法，他们的实现原理和上面的两个方法完全相同，区别是返回值不同，前两个方法返回的是改变之后的值，即next。而这两个方法返回的是改变之前的值，即current。还有很多的其他方法，就不列举了。

我们都知道，所谓线程池，那么就是相当于有一个池子，线程就放在这个池子中进行重复利用， 能够减去了线程的创建和销毁所带来的代价。 但是这样并不能很好的解释线程池的原理，下面从代码的角度分析一下线程池的实现。

线程池的相关类

对于原理，在 Java 中，有几个接口，类 值得我们关注：

Executor
ExecutorService
AbstractExecutorService
ThreadPoolExecutor
Executor

public interface Executor {
    void execute(Runnable command);
}
Executor 接口只有一个 方法，execute，并且需要 传入一个 Runnable 类型的参数。 那么它的作用自然是 具体的执行参数传入的任务。

ExecutorService

public interface ExecutorService extends Executor {

    void shutdown();

    List<Runnable> shutdownNow();

    boolean isShutdown();

    <T> Future<T> submit(Callable<T> task);

    <T> Future<T> submit(Runnable task, T result);

    Future<?> submit(Runnable task);

    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;
    ......
}
ExecutorService 接口继承了 Executor，并且提供了一些其他的方法，比如说：

shutdownNow ： 关闭线程池，返回放入了线程池，但是还没开始执行的线程。
submit ： 执行的任务 允许拥有返回值。
invokeAll ： 运行把任务放进集合中，进行批量的执行，并且能有返回值
这三个方法也可以说是这个接口重点扩展的方法。

Ps：execute 和 submit 区别：

submit 有返回值，execute 没有返回值。 所以说可以根据任务有无返回值选择对应的方法。
submit 方便异常的处理。 如果任务可能会抛出异常，而且希望外面的调用者能够感知这些异常，那么就需要调用 submit 方法，通过捕获 Future.get 抛出的异常。
AbstractExecutorService

AbstractExecutorService 是一个抽象类，主要完成了 对 submit 方法，invokeAll 方法 的实现。 但是其实它的内部还是调用了 execute 方法，例如：

public Future<?> submit(Runnable task) {
    if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null);
        execute(ftask);
        return ftask;
    }
ThreadPoolExecutor

ThreadPoolExecutor 继承了 AbstractExecutorService，并且实现了最重要的 execute 方法 ，是我们主要需要研究的类。另外，整个线程池是如何实现的呢？

在该类中，有两个成员变量 非常的重要：

private final HashSet<Worker> workers = new HashSet<Worker>();
private final BlockingQueue<Runnable> workQueue;
对于 workers 变量，主要存在了线程对象 Worker，Worker 实现了 Runnable 接口。而对于 workQueue 变量，主要存放了需要执行的任务。 这样其实可以猜到， 整个线程池的实现原理应该是 workQueue 中不断的取出需要执行的任务，放在 workers 中进行处理。

另外，当线程池中的线程用完了之后，多余的任务会等待，那么这个等待的过程是 怎么实现的呢？ 其实如果熟悉 BlockingQueue，那么就会马上知道，_是利用了 BlockingQueue 的take 方法进行处理_。

下面具体代码分析：

   public void execute(Runnable command) {
        ......
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        ......
    }
首先，这里需要先理解两个概念。我们在创建线程池的时候，通常会指定两个变量，一个是maximumPoolSize，另外一个是 corePoolSize。

对于 maximumPoolSize：指的是 线程池中最多允许有多少个线程。
对于 corePoolSize： 指的是线程池中正在运行的线程。
在 线程池中，有这样的设定，我们加入一个任务进行执行，

如果现在线程池中正在运行的线程数量大于 corePoolSize 指定的值而 小于maximumPoolSize 指定的值，那么就会创建一个线程对该任务进行执行，一旦一个线程被创建运行。
如果线程池中的线程数量大于corePoolSize，那么这个任务执行完毕后，该线程会被回收；如果 小于corePoolSize，那么该线程即使空闲，也不会被回收。下个任务过来，那么就使用这个空闲线程。
对于上述代码，首先有：

if (workerCountOf(c) < corePoolSize)

也就是说，判断现在的线程数量是否小于corePoolSize，如果小于，那么就创建一个线程执行该任务，也就是执行

addWorker(command, true)

如果大于，那么就把该任务放进队列当中，即

workQueue.offer(command)

那么，addWorker 是干什么的呢？

private boolean addWorker(Runnable firstTask, boolean core) {    
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
             ......
    }
在这里可以看到一些关键代码，例如 w = new Worker(firstTask)， 以及 workers.add(w); 从这里 我们就可以看到，创建 线程对象 并且加入到 线程 队列中。但是，我们现在还没有看到具体是怎么执行任务的，继续追踪
w = new Worker(firstTask)，如下代码：

private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        ......

        final Thread thread;

        Runnable firstTask;
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }
        public void run() {
            runWorker(this);
        }
        ......
对于 runWorker 方法：

final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } 
                ......
    }
在这段代码中，就有很多关键的信息，比如说，Runnable task = w.firstTask;如果为空，那么就 执行 task = getTask()，如果不为空，那么就 进行 task.run(); 调用其方法， 这里也就是具体的执行的任务 。

现在知道了是怎么样执行具体的任务，那么假如任务的数量 大于 线程池的数量，那么是怎么实现等待的呢，这里就需要看到getTask() 的具体实现了，如下：

private Runnable getTask() {
        for (;;) {
           ......
            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
这里可以看到， 一个 for 死循环，以及

Runnable r = timed ? workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) : workQueue.take();

而 workQueue 是 BlockingQueue 类型，也就是带有阻塞的功能。

这就是 线程如何等待执行的。

总结

现在就可以知道，大致的线程池实现原理：

首先，各自存放线程和任务，其中，任务带有阻塞。

private final HashSet<Worker> workers = new HashSet<Worker>();
private final BlockingQueue<Runnable> workQueue;
然后，在 execute 方法中 进行 addWorker(command，true)，也就是创建一个线程，把任务放进去执行；或者是直接把任务放入到任务队列中。

接着 如果是 addWorker，那么就会 new Worker(task) -》调用其中 run() 方法，在Worker 的run() 方法中，调用 runWorker(this); 方法 -》在该方法中就会具体执行我们的任务 task.run(); 同时这个 runWorker方法相当于是个死循环，正常情况下就会一直取出 任务队列中的任务来执行，这就保证了线程 不会销毁。

所以，这也是为什么常说的线程池可以避免线程的频繁创建和 销毁带来的性能消耗。

Collection 与 List

Collection 是 Java 集合的一个根接口，JDK 没有它的实现类。 内部仅仅做 add()，remove()，contains()，size() 等方法的声明。

List 接口是Collection 接口的一个子类，在Collection 基础上扩充了方法。同时可以对每个元素插入的位置进行精确的控制，它的主要实现类有 ArrayList，Vector，LinkedList。

这里需要注意的是，List 接口的实现类

插入的值允许为空，也允许重复。
插入的值允许为空，也允许重复。
插入的值允许为空，也允许重复。
ArrayList

ArrayList 实现了 List 接口，意味着可以插入空值，也可以插入重复的值，非同步 ，它是 基于数组 的一个实现。

部分成员变量如下：

private static final int DEFAULT_CAPACITY = 10;                        //默认初始值    
transient Object[] elementData;                                        //存放数据的数组    
这里可以看出，elementData 提示了是基于数组的实现， DEFAULT_CAPACITY 指名了默认容量为 10 个。

下面重点理解 add()方法，这将有助于理解 ArrayList 的应用场景和性能消耗：

add():

public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }

    ensureExplicitCapacity(minCapacity);
}
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
    grow(minCapacity);
}
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}
这里可以看到，进行一个 add()方法，首先通过 ensureCapacityInternal(size + 1); 判断需不需要扩容;然后再进行 elementData[size++] = e。 插入的时间复杂度O(1)。是非常高效的。（But，这不是在固定位置插入的情况下），如果在固定位置插入，那么代码：

public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
System.arraycopy(elementData, index, elementData, index + 1, size - index); 这个可以看出，这时候就是比较低效的了。

那么网上很多说 ArrayList 不适合 增删操作非常多的操作，这是怎么回事呢？ 首先可以看到这句话： elementData = Arrays.copyOf(elementData, newCapacity); 需要知道的是， Arrays.copyOf 函数的内部实现是再创建一个数组，然后把旧的数组的值一个个复制到新数组中。当经常增加操作的时候，容量不够的时候，就会进行上述的扩容操作，这样性能自然就下来了。 或者说，当我们在固定位置进行增删的时候，都会进行 System.arraycopy(elementData, index, elementData, index + 1, size - index); 也是非常低效的。

分析了低效出现的原因，那么我们就可以知道：如果我们需要经常进行特定位置的增删操作，那么最好还是不要用这个了，但是，如果我们基本上没有固定位置的增删操作，最好是要预估数据量的大小，然后再初始化最小容量，这样可以有效的避免扩容。如下代码：

ArrayList<Integer> arrayList = new ArrayList<Integer>(20);

总结：

ArrayList 可以插入空值，也可以插入重复值
ArrayList 是基于数组的时候，所以很多数组的特性也直接应用到了 ArrayList。
ArrayList 的性能消耗主要来源于扩容和固定位置的增删。
ArrayList 创建的时候 需要考虑是否要初始化最小容量，以此避免扩容带来的消耗。
Vector

上述的 ArrayList 不是线程安全的。那么 Vector 就可以看作是 ArrayList 的一个线程安全版本。由于也是实现了 List 接口，所以也是 可以插入空值，可以插入重复的值。 它和 HashTable 一样，是属于一种同步容器，而不是一种并发容器。（参考《Java并发编程实战》，类似CopyOnWriteArrayList，ConcurrentHashMap这种就属于并发容器）

内部成员变量：

protected Object[] elementData;
public Vector() {
    this(10);
}
可以看到，也是基于 数组的实现，初始化也是 10 个容量。 那么，再来看看 add()方法是否和 ArrayList 相同。

public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}
可以看到，和 ArrayList 也是一样的，只是加了 synchronized 进行同步， 其实很多其他方法都是通过加 synchronized 来实现同步。

总结：

可以插入空值，也可以插入重复值
也是基于数组的时候，所以很多数组的特性也直接应用到了 Vector。
性能消耗也主要来源于 扩容。
创建的时候 需要考虑是否要初始化最小容量，以此避免扩容带来的消耗。
相当于 ArrayList 的线程安全版本，实现同步的方式 是通过 synchronized。
LinkedList

LinkedList 实现了 List 接口，所以LinkedList 也可以放入重复的值，也可以放入空值。LinkedList不支持同步。LinkedList 不同于ArrayList 和Vector，它是使用链表的数据结构，不再是数组。

成员变量：

transient int size = 0;    //数量    
transient Node<E> first;   //首节点    
transient Node<E> last;    //最后节点  

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
那么，在进行 add() 方法的时候：

public boolean add(E e) {    
    linkLast(e);    
    return true;    
}   
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
当进行增删的时候，只需要改变指针，并不会像数组那样出现整体数据的大规模移动，复制等消耗性能的操作。

ArrayList，Vector，LinkedList 的整体对比

实现方式：
ArrayList，Vector 是基于数组的实现。
LinkedList 是基于链表的实现。
同步问题：
ArrayList,LinkedList 不是线程安全的。
Vector 是线程安全的，实现方式是在方法中加 synchronized 进行限定。
适用场景和性能消耗
ArrayList 和 Vector 由于是基于数组的实现，所以在固定位置插入，固定位置删除这方面会直接是 O(n)的时间复杂度，另外可能会出现扩容的问题，这是非常消耗性能的。
LinkedList 不会出现扩容的问题，所以比较适合增删的操作。但是由于不是数组的实现，所以在 定位方面必须使用 遍历的方式，这也会有 O(n)的时间复杂度.

HashMap 概念

对于 Map ，最直观就是理解就是键值对，映射，key-value 形式。一个映射不能包含重复的键，一个键只能有一个值。平常我们使用的时候，最常用的无非就是 HashMap。

HashMap 实现了 Map 接口，允许使用 null 值 和 null 键，并且不保证映射顺序。

HashMap 有两个参数影响性能：

初始容量：表示哈希表在其容量自动增加之前可以达到多满的一种尺度
加载因子：当哈希表中的条目超过了容量和加载因子的乘积的时候，就会进行重哈希操作。
如下成员变量源码：

static final float DEFAULT_LOAD_FACTOR = 0.75f;
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
transient Node<K,V>[] table;
可以看到，默认加载因子为 0.75， 默认容量为 1 << 4，也就是 16。加载因子过高，容易产生哈希冲突，加载因子过小，容易浪费空间，0.75是一种折中。

另外，整个 HashMap 的实现原理可以简单的理解成： 当我们 put 的时候，首先根据 key 算出一个数值 x，然后在 table[x] 中存放我们的值。 这样有一个好处是，以后的 get 等操作的时间复杂度直接就是O(1)，因为 HashMap 内部就是基于数组的一个实现。

另外，是怎么算出这个位置的，非常非常推荐看下 JDK 源码中 HashMap 的 hash 方法原理是什么？

put 方法的实现 与 哈希冲突

下面再结合代码重点分析下 HashMap 的 put 方法的内部实现 和 哈希冲突的解决办法：

public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                        break;
                    }
                if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
首先我们看到 hash(key) 这个就是表示要根据 key 值算出一个数值，以此来决定在 table 数组的哪一个位置存放我们的数值。

（Ps：这个 hash(key) 方法 也是大有讲究的，会严重影响性能，实现得不好会让 HashMap 的 O(1) 时间复杂度降到 O(n)，在JDK8以下的版本中带来灾难性影响。它需要保证得出的数在哈希表中的均匀分布，目的就是要减少哈希冲突）

重要说明一下：

JDK8 中哈希冲突过多，链表会转红黑树，时间复杂度是O(logn)，不会是O(n)
JDK8 中哈希冲突过多，链表会转红黑树，时间复杂度是O(logn)，不会是O(n)
JDK8 中哈希冲突过多，链表会转红黑树，时间复杂度是O(logn)，不会是O(n)
然后，我们再看到：

if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
else {
    ......
这就表示，如果没有 哈希冲突，那么就可以放入数据 tab[i] = newNode(hash, key, value, null); 如果有哈希冲突，那么就执行 else 需要解决哈希冲突。

那么放入数据 其实就是 建立一个 Node 节点，该 Node节点有属性 key，value，分别保存我们的 key 值 和 value 值，然后再把这个 Node 节点放入到 table 数组中，并没有什么神秘的地方。

static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
上述可以看到 Node 节点中 有一个 Node<K,V> next; ，其实仔细思考下就应该知道这个是用来解决哈希冲突的。下面再看看是如何解决哈希冲突的：

哈希冲突：通俗的讲就是首先我们进行一次 put 操作，算出了我们要在 table 数组的 x 位置放入这个值。那么下次再进行一个 put 操作的时候，又算出了我们要在 table 数组的 x 位置放入这个值，那之前已经放入过值了，那现在怎么处理呢？

其实就是通过链表法进行解决。

首先，如果有哈希冲突，那么：

if (p.hash == hash &&
    ((k = p.key) == key || (key != null && key.equals(k))))
e = p;
需要判断 两者的 key 是否一样的，因为 HashMap 不能加入重复的键。如果一样，那么就覆盖，如果不一样，那么就先判断是不是 TreeNode 类型的：

 else if (p instanceof TreeNode)
    e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
这里表示 是不是现在已经转红黑树了（在大量哈希冲突的情况下，链表会转红黑树），一般我们小数据的情况下，是不会转的，所以这里暂时不考虑这种情况（Ps：本人也没太深入研究红黑树，所以就不说这个了）。

如果是正常情况下，会执行下面的语句来解决哈希冲突：

for (int binCount = 0; ; ++binCount) {
    if ((e = p.next) == null) {
        p.next = newNode(hash, key, value, null);
        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
            treeifyBin(tab, hash);
        break;
    }
    if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
        break;
    p = e;
}
这里其实就是用链表法来解决。并且：

冲突的节点放在链表的最下面。
冲突的节点放在链表的最下面。
冲突的节点放在链表的最下面。
因为 首先有：p = tab[i = (n - 1) & hash] ，再 for 循环，然后有 if ((e = p.next) == null) { ,并且如果 当前节点的下一个节点有值的话，那么就 p = e;，这就说明了放在最下面。

强烈建议自己拿笔拿纸画画。

总结

一个映射不能包含重复的键，一个键只能有一个值。允许使用 null 值 和 null 键，并且不保证映射顺序。
HashMap 解决冲突的办法先是使用链表法，然后如果哈希冲突过多，那么会把链表转换成红黑树，以此来保证效率。
如果出现了哈希冲突，那么新加入的节点放在链表的最前面。

LinkedHashMap 概述

由于 HashMap 中的 entry 是无序的，也就是说，当 map.entrySet() 得到的 entry的集合既不能按照插入顺序排列，也不能按照访问先后顺序排序。

那么 LinkedHashMap 就可以解决这个问题。LinkedHashMap 可以让元素：

按照插入顺序排列
也可以按照访问的先后顺序排序
这里需要注意，不能按照 key 的大小排序，那是 TreeMap 的功能，不要弄混淆了。

如下 Demo：

public class Test {
    public static void main(String args[]){
        Map<String,String> map = new LinkedHashMap<>(16,0.75f,true);

        map.put("A","a");
        map.put("B","b");
        map.put("C","c");
        map.put("D","d");
        map.put("E","e");

        for(Map.Entry<String,String> entry : map.entrySet()) {
            System.out.println(entry);
        }

        map.get("D");
        map.get("B");
        System.out.println("==========");

        for(Map.Entry<String,String> entry : map.entrySet()) {
            System.out.println(entry);
        }
    }
}
那么输出：

A=a
B=b
C=c
D=d
E=e
==========
A=a
C=c
E=e
D=d
B=b
这里可以看到，在 map.get("D"); 和 map.get("B"); 之后，打印的最后就变了，这个是按照访问来排序。

原理

LinkedHashMap 继承于 HashMap，很多方法都是直接用了 HashMap 的方法，那么本身再在 HashMap 的基础上提供一个排序的功能。

它的成员变量有：

transient LinkedHashMap.Entry<K,V> head;
transient LinkedHashMap.Entry<K,V> tail;
final boolean accessOrder;
这个时候其实可以猜到，其实是用了一个链表来维护顺序。这个 accssOrder 表示是否需要按照访问顺序来排序，如果不需要，那么默认就是按照插入顺序来排序。如上代码的构造方法 LinkedHashMap<>(16,0.75f,true); 的 true 就表示了需要按照访问顺序。

put(K key, V value)

在进行 put(K key, V value) 操作的时候， LinkedHashMap 实际上是调用了 HashMap 的 put(K key, V value) 方法，然后在 HashMap 中的 put(K key, V value) 方法后，有一句：

afterNodeAccess(e);
这个方法就是留给 LinkedHashMap 来实现。除此之外，还有

// Callbacks to allow LinkedHashMap post-actions
void afterNodeAccess(Node<K,V> p) { }
void afterNodeInsertion(boolean evict) { }
void afterNodeRemoval(Node<K,V> p) { }
并且注释已经写明了是给 LinkedHashMap 来用的。

到这里，我们就可以知道原理：LinkedHashMap 进行 put(K key, V value) 操作的时候，调用了 HashMap 的 put(K key, V value)， 然后 HashMap 再调用回 LinkedHashMap 特定的方法，来实现排序，get(),remove() 方法同理。而 LinkedHashMap 实现排序的方式就是通过链表。

参考：Java LinkedHashMap工作原理及实现 ，用的其实是一个双向链表。

再具体一下 afterNodeAccess(e);，以下这个是 HashMap 的 put 方法：

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    ......
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ......
}
可以看到，在 else 的 for 循环中，put 了这个 key-value 之后，会在下面 if (e != null) 调用 afterNodeAccess(e);：

这里要注意一点，这里的 节点类型是：

static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
可以看到，保存了 before, after; 其实这个就是用来引用前后节点的，这样可以用来做双端队列。

void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
这个代码就可以具体看到，首先，判断你是不是要按照访问顺序来使用，如果不是的话，那这里什么也不做。（具体可以用纸笔画画）。

我们知道，HashMap 放入的 Entry 是没有顺序的，而 LinkedHashMap 是根据 put 的顺序或者是访问的顺序来排序，而不是根据 key 的大小来排序。

那么 TreeMap 就有这样一个作用：根据 key 的大小进行排序。默认是升序的。

TreeMap 源码注释的第一句就说了：

A Red-Black tree based {@link NavigableMap} implementation.
所以首先要对红黑树有所了解。参考 wiki：红黑树，是一种自平衡二叉查找树。那么对于二叉查找树，参考 wiki：二叉搜索树 ，是指一棵树：

任意节点的左子树不空，则左子树上所有结点的值均小于它的根结点的值；
任意节点的右子树不空，则右子树上所有结点的值均大于它的根结点的值；
任意节点的左、右子树也分别为二叉查找树；
没有键值相等的节点。
这样子的一棵树就成为二叉搜索树，那么这个树是会倾斜的，一倾斜就会导致效率的下降，所以说就有了一个平衡的概念，使其不会倾斜。

红黑树就采用了一些算法，使得能够保存平衡，具体的红黑树的概念实现个人觉得有点复杂，也先不了解了，这里主要是看看 TreeMap 是如何使用的。具体原理可以参考： Java提高篇（二七）-----TreeMap

TreeMap 的使用

根据源码的注释：

* The map is sorted according to the {@linkplain Comparable natural
* ordering} of its keys, or by a {@link Comparator} provided at map
* creation time, depending on which constructor is used.
那么，要么是我们放入的 Entry 实现了 Comparable 接口，要么是我们在创建 TreeMap 的时候提供一个 Comparator。

方式一：Entry 实现了 Comparable

如下代码：

public class Test {
    public static void main(String args[]){
        Map<Student,String> map = new TreeMap<>();

        Student student1 = new Student(3);
        Student student2 = new Student(1);
        Student student3 = new Student(2);
        Student student4 = new Student(0);
        Student student5 = new Student(5);

        map.put(student1,"a");
        map.put(student2,"b");
        map.put(student3,"c");
        map.put(student4,"d");
        map.put(student5,"e");

        Iterator<HashMap.Entry<Student,String>> iterator = map.entrySet().iterator();

        while (iterator.hasNext()){
            System.out.println(iterator.next().getKey().getStudentNum());
        }

    }

    static class Student implements Comparable<Student>{
        private int studentNum;
        public Student(int studentNum){
            this.studentNum = studentNum;
        }
        public int getStudentNum(){
            return this.studentNum;
        }

        @Override
        public int compareTo(Student o) {
            if(o == null){
                return 1;
            }
            if(o == this){
                return 0;
            }
            if(this.studentNum == o.getStudentNum()){
                return 0;
            } else {
                return studentNum > o.getStudentNum() ? 1 : -1;
            }
        }
    }
}
输出如下：

0
1
2
3
5
方式二：提供 Comparator

public class Test {
    public static void main(String args[]){

        Comparator<Student> comparator = new Comparator<Student>() {
            @Override
            public int compare(Student o1, Student o2) {
                if(o1.studentNum == o2.getStudentNum()){
                    return 0;
                } else {
                    return o1.getStudentNum() > o2.getStudentNum() ? 1 : -1;
                }
            }
        };

        Map<Student,String> map = new TreeMap<>(comparator);

        Student student1 = new Student(3);
        Student student2 = new Student(1);
        Student student3 = new Student(2);
        Student student4 = new Student(0);
        Student student5 = new Student(5);

        map.put(student1,"a");
        map.put(student2,"b");
        map.put(student3,"c");
        map.put(student4,"d");
        map.put(student5,"e");

        Iterator<HashMap.Entry<Student,String>> iterator = map.entrySet().iterator();

        while (iterator.hasNext()){
            System.out.println(iterator.next().getKey().getStudentNum());
        }
    }

    static class Student{
        private int studentNum;
        public Student(int studentNum){
            this.studentNum = studentNum;
        }
        public int getStudentNum(){
            return this.studentNum;
        }
    }
}
可以获得同样输出。

HashSet

HashSet 实现了 Set 接口，而 Set 接口是继承于 Collection 接口，所以可以认为 Set 接口是 List 接口的兄弟。

对于 Set 接口，如注释所说：

 * A collection that contains no duplicate elements.  More formally, sets
 * contain no pair of elements <code>e1</code> and <code>e2</code> such that
 * <code>e1.equals(e2)</code>, and at most one null element.  As implied by
 * its name, this interface models the mathematical <i>set</i> abstraction.
所以说不重复，而且HashSet 底层使用的是 HashMap，所以也无序。 这两点是和 List 接口下的 ArrayList，LinkedList 等的一大区别。

我们知道，HashMap 是键值对的形式，HashSet 没有什么键值对的概念，那它内部具体又是怎么利用 HashMap 实现的呢？

其实就是 HashSet 用了一个空对象，如 private static final Object PRESENT = new Object

用这个空对象来填充 HashMap 的 value 域
用这个空对象来填充 HashMap 的 value 域
用这个空对象来填充 HashMap 的 value 域
如下面的 add 方法：

private transient HashMap<E,Object> map;

public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
所以从这里就可以看出：

利用了 HashMap 实现。HashSet 的方法就是调用 HashMap 的对应的方法。
用空对象来填充 HashMap 的 value 域
TreeSet

这里需要知道，平常我们所使用的 ArrayList，LinkedList 的元素是按照插入的时候的顺序排列的，而不是按照这个元素的大小之类的排序，假如我们需要有这个功能的话，那么 TreeSet 就派上用场了。

当创建一个 TreeSet 的时候：

private transient NavigableMap<E,Object> m;

public TreeSet() {
    this(new TreeMap<E,Object>());
}
TreeSet(NavigableMap<E,Object> m) {
    this.m = m;
}
可以看到，内部是使用了 TreeMap。

当 add(E e) 方法的时候：

private static final Object PRESENT = new Object();

public boolean add(E e) {
    return m.put(e, PRESENT)==null;
}
也是和 HashSet 一样，令对应的 Map 的 value 域为一个随便不要的对象就行。

其他的方法如：

public boolean remove(Object o) {
    return m.remove(o)==PRESENT;
}
public void clear() {
    m.clear();
}
等等都是类似的，这里不再累赘。