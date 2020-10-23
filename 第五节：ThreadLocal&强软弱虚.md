[TOC]

### ThreadLocal

ThreadLocal线程局部变量

ThreadLocal是使用空间换时间，synchronized是使用时间换空间

==比如在hibernate中session就存在与ThreadLocal中，避免synchronized的使用==

    ThreadLocal<Person> tl = new ThreadLocal<>();
    System.out.println(tl.get());
    tl.set(new Person());


运行下面的程序，理解ThreadLocal
```
public class ThreadLocal1 {
	volatile static Person p = new Person();
	
	public static void main(String[] args) {
				
		new Thread(()->{
			try {
				TimeUnit.SECONDS.sleep(2);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			
			System.out.println(p.name);
		}).start();
		
		new Thread(()->{
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			p.name = "lisi";
		}).start();
	}
}

class Person {
	String name = "zhangsan";
}

运行结果：
lisi
```

```
public class ThreadLocal2 {
	//volatile static Person p = new Person();
	static ThreadLocal<Person> tl = new ThreadLocal<>();
	
	public static void main(String[] args) {
				
		new Thread(()->{
			try {
				TimeUnit.SECONDS.sleep(2);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			
			System.out.println(tl.get());
		}).start();
		
		new Thread(()->{
			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
			tl.set(new Person());
		}).start();
	}
	
	static class Person {
		String name = "zhangsan";
	}
}

运行结果：
null
```

![501](6255F56B44FB4B5AA8FE04D76036A26F)
    
    
    ThreadLocal<Person> tl = new ThreadLocal<>();
    
    当前的main线程中有t1变量，是一个强引用，指向new ThreadLocal<Person>();
    Thread类中有成员变量threadLocals,而threadLocals是ThreadLocal.ThreadLocalMap类
    说明Thread类中包含了一个map，key=ThreadLocal对象t1  value保存了Person的内存地址
    
    set(Person)
    get-->Person
    
    tl指向new ThreadLocal<>(),是一个强引用
    
    Thread中的threadLocals属性中key对于ThreadLocal是弱引用
    
    1.为什么ThreadLocal中用到了弱引用？
        防止内存泄漏，threadLocals中key对于ThreadLocal对象t1的引用
    2.为了防止内存泄漏，ThreadLocal使用后，需要remove
        因为threadLocals中的key设为null的时候，value对于Person对象的引用还存在；
        如果不清理，因为key=null，value引用的地址无法被访问到，导致内存泄漏
    

**面试题：为什么ThreadLocal中用到了弱引用？**  
    
    防止内存泄漏，针对的是threadLocals中key的引用
    
    Thread类中有threadLocals属性（所属类是：ThreadLocal.ThreadLocalMap），key是对ThreadLocal对象的引用，当t1对象设置为null，使用了弱引用，key就会被GC回收
    
**面试题：为什么ThreadLocal使用后要调用remove方法？**  
    
    防止内存泄漏，，针对的是threadLocals中value的引用

### 补充知识：
    
    声明式事物：
        一个方法去数据库拿到数据库连接(connection)
        声明式事务可以把多个方法视为一个完整的事务
        
        多个方法在一个线程中
        怎么保证多个方法拿到的连接是同一个connection
            把connection放到当前线程的ThreadLocal中
            线程是从ThreadLocal中读取配置，不是从线程池中读取配置


##### ThreadLocal源码

    set
        -Thread.currentThread.map(ThreadLocal,person)
            设到当前线程的map中
    用途：
        -spring的声明式事物，保证同一个connection
```
ThreadLocal类：
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        map.set(this, value);
    } else {
        createMap(t, value);
    }
}

ThreadLocalMap类（在ThreadLocal类中）
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}  

Thread类
ThreadLocal.ThreadLocalMap threadLocals = null;//当前线程的map


static class ThreadLocalMap {

    /**
     * The entries in this hash map extend WeakReference, using
     * its main ref field as the key (which is always a
     * ThreadLocal object).  Note that null keys (i.e. entry.get()
     * == null) mean that the key is no longer referenced, so the
     * entry can be expunged from table.  Such entries are referred to
     * as "stale entries" in the code that follows.
     */
    static class Entry extends WeakReference<ThreadLocal<?>> {//继承弱引用
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    ...
}
```

```
public class T03_WeakReference {
    public static void main(String[] args) {
        WeakReference<M> m = new WeakReference<>(new M());

        System.out.println(m.get());
        System.gc();
        System.out.println(m.get());


        ThreadLocal<M> tl = new ThreadLocal<>();
        tl.set(new M());
        tl.remove();

    }
}
运行结果：
com.mashibing.juc.c_022_RefTypeAndThreadLocal.M@506e1b77
null
finalize
```    
            
### 引用：强软弱虚
    
###### 强引用：NormalReference
    
    varhandle是强引用
```
public class M {
    @Override
    protected void finalize() throws Throwable {
        System.out.println("finalize");
    }
}

public class T01_NormalReference {
    public static void main(String[] args) throws IOException {
        M m = new M();
        m = null;
        System.gc(); //DisableExplicitGC

        System.in.read();//阻塞当前线程
    }
}

运行结果：
finalize
```

###### 软引用 SoftReference
    
    软引用是用来描述一些还有用但并非必须的对象。
    对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围进行第二次回收。
    如果这次回收还没有足够的内存，才会抛出内存溢出异常。
    -Xms20M -Xmx20M
    
    应用：适合缓存使用
        读一个大图片放到内存中
        从数据库读一大堆的数据放到内存中
        
        tomcat和memberCache中用到了软引用

==内存不够用的时候，才会被回收==
       
```
public class T02_SoftReference {
    public static void main(String[] args) {
        SoftReference<byte[]> m = new SoftReference<>(new byte[1024*1024*10]);
        //m = null;
        System.out.println(m.get());
        System.gc();
        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println(m.get());

        //再分配一个数组，heap将装不下，这时候系统会垃圾回收，先回收一次，如果不够，会把软引用干掉
        byte[] b = new byte[1024*1024*15];
        System.out.println(m.get());
    }
}

运行结果：
[B@3941a79c
[B@3941a79c
null
```
==书：提问的智慧==
 
###### 弱引用 WeakReference（面试重点）
    
    弱引用遭到gc就会回收
    如果有一个强引用同时存在的话，当强引用消失的话，弱引用自动消失
    
    应用：一般用在容器里
        ThreadLocal
        
==弱引用遇到GC就会被回收   == 
        
![501](9DEA23C6101C426DAA0491537C81DF55)

==作业：WeakHashMap的使用==
```
public class T03_WeakReference {
    public static void main(String[] args) {
        WeakReference<M> m = new WeakReference<>(new M());

        System.out.println(m.get());
        System.gc();
        System.out.println(m.get());


        ThreadLocal<M> tl = new ThreadLocal<>();
        tl.set(new M());
        tl.remove();

    }
}
运行结果：
com.mashibing.juc.c_022_RefTypeAndThreadLocal.M@506e1b77
null
finalize
```


###### 虚引用：PhantomReference  应用程序用不到，写JVM或Netty要用到

    应用：管理堆外内存的
    
    为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知
    
    虚引用，遇到GC时，通知Queue去回收
    
    弱引用：可以get到值
    虚引用：永远get不到值
        get不到值有什么用呢？虚引用对象被干掉后，通知Queue
        
    DirectByteBuffer通过虚引用来实现堆外内存的释放的
    
    java里Unsafe类：直接内存的分配和回收
        freeMemory()
        
        Unsafe类，JDK8可以通过反射使用；JDK9以后不能被使用
        
    
    
    一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，
    也无法通过虚引用来获取一个对象的实例。
    为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。
    虚引用和弱引用对关联对象的回收都不会产生影响，如果只有虚引用活着弱引用关联着对象，
    那么这个对象就会被回收。它们的不同之处在于弱引用的get方法，虚引用的get方法始终返回null,
    弱引用可以使用ReferenceQueue,虚引用必须配合ReferenceQueue使用。
 
    jdk中直接内存的回收就用到虚引用，由于jvm自动内存管理的范围是堆内存，
    而直接内存是在堆内存之外（其实是内存映射文件，自行去理解虚拟内存空间的相关概念），
    所以直接内存的分配和回收都是有Unsafe类去操作，java在申请一块直接内存之后，
    会在堆内存分配一个对象保存这个堆外内存的引用，
    这个对象被垃圾收集器管理，一旦这个对象被回收，
    相应的用户线程会收到通知并对直接内存进行清理工作。
 
    事实上，虚引用有一个很重要的用途就是用来做堆外内存的释放，
    DirectByteBuffer就是通过虚引用来实现堆外内存的释放的。
 
**面试题：虚引用与弱引用的区别**
    
    GC
        虚引用：遇到GC，发系统通知，通知ReferenceQueue进行内存回收
        弱引用：遇到GC就会被回收
    
    get
        虚引用：get不到值
        弱应用：get到值
    
    应用：
        虚引用：对外内存的管理，Unsafe类中直接内存的分配以及回收( freeMemory() )；
            DirectByteBuffer就是通过虚引用来实现对外内存的释放
        
        弱应用：ThreadLocal，防止内存泄漏
    
![image](https://raw.githubusercontent.com/musictaste/JUC/master/image/502.png) 
    
```
VM设置参数：-Xms20M -Xmx20M
不停的往list中放数据，是为了触发gc操作使M被清除，从而发送系统通知给Queue

public class T04_PhantomReference {
    private static final List<Object> LIST = new LinkedList<>();
    private static final ReferenceQueue<M> QUEUE = new ReferenceQueue<>();

    public static void main(String[] args) {

        PhantomReference<M> phantomReference = new PhantomReference<>(new M(), QUEUE);

        new Thread(() -> {
            while (true) {
                LIST.add(new byte[1024 * 1024]);
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                    Thread.currentThread().interrupt();
                }
                System.out.println(phantomReference.get());
            }
        }).start();

        new Thread(() -> {
            while (true) {
                Reference<? extends M> poll = QUEUE.poll();
                if (poll != null) {
                    System.out.println("--- 虚引用对象被jvm回收了 ---- " + poll);
                }
            }
        }).start();

        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
}

运行结果：
null
null
null
null
finalize
null
null
--- 虚引用对象被jvm回收了 ---- java.lang.ref.PhantomReference@541390e7
null
null
Exception in thread "Thread-0" java.lang.OutOfMemoryError: Java heap space
	at java.base/java.util.LinkedList.linkLast(LinkedList.java:146)
	at java.base/java.util.LinkedList.add(LinkedList.java:342)
	at com.mashibing.juc.c_022_RefTypeAndThreadLocal.T04_PhantomReference.lambda$main$0(T04_PhantomReference.java:46)
	at com.mashibing.juc.c_022_RefTypeAndThreadLocal.T04_PhantomReference$$Lambda$14/0x0000000800ba4840.run(Unknown Source)
	at java.base/java.lang.Thread.run(Thread.java:830)
```


### 总结：
![image](https://raw.githubusercontent.com/musictaste/JUC/master/image/501.png) 
    
    请仔细阅读，并理解弱引用
    理解为什么ThreadLocal要用弱引用，以及为什么使用后要remove
    理解内存泄漏
    
    AQS源码
        -VarHandle
            -1.普通属性的原子操作
            -2.比反射快，直接操作二进制码
            
    ThreadLocal
        弱引用
    
    强软弱虚
        软引用：SoftReference
            对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围进行第二次回收。
            如果这次回收还没有足够的内存，才会抛出内存溢出异常。
            应用：适合缓存使用
            
        弱引用：WeakReference
            弱引用遭到gc就会回收
            ThredLocal
            
        虚引用：PhantomReference
            应用：管理堆外内存的
        
### 面试题:
    0.谈一谈强软弱虚
        强：常见的引用，引用消失，GC时被回收
        软：应用于缓存，Tomcat和memberCache就使用了软引用，将大图片或大量数据读到内存中；内存不够的时候才会被回收
        弱：ThreadLocal，防止内存泄漏；遇到GC就会被回收
        虚：堆外内存的管理，get不到值，GC时通知ReferenceQueue进行内存回收；DirectByteBuffer就是通过虚引用对堆外内存进行释放
        
        

    1.synchronized能锁的最小对象是什么？
        synchronized锁的是对象头部markwork中的两个标识位
        
       0 0 1：无锁态
       1 0 1：偏向锁
         0 0：轻量级锁
         1 0：重量级锁
         1 1：GC标记
        
        最小对象这个问题，问的就有问题
        
    2.ThreadLocal弱引用，当强引用不存在了，GC的时候弱引用就会被回收
    
    3.为什么ThreadLocal中用到了弱引用？ 
        防止内存泄漏
    
    4.面试题：为什么ThreadLocal使用后要调用remove方法  
        防止内存泄漏