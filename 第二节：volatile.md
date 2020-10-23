    刷算法题：LeetCode  每天都刷  
        刷够200到题，校招难不倒；社招兴趣就行，不怎么会考算法
        
        字节跳动算法考的比较多
        
    阿里七面面试    
        
    Eden区什么时候转到Old区  
        文章：[-XX：PertenureSizeThreshold的默认值和作用](https://www.jianshu.com/p/f7cde625d849)  
        -XX：PertenureSizeThreshold的默认值和作用  
        默认值为0，
        
    Lock：CAS，占用CPU时间的
    Synchronized：重量级锁是不占用cpu时间的，因为是进入OS的wait队列
    
    
##### 设置守护线程

怎么设置守护线程？  
setDaemon(true)就是将当前线程设置为守护线程  

==守护线程的特点：当主线程结束时，守护线程自动终止==  
==必须要在start()方法之前设置==
```
public class SetDaemon extends Thread{
    private int count =0;
    @Override
    public void run() {
        while(true){
            count++;
            System.out.println(Thread.currentThread().getName()+" count:"+count);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }
    }

    public static void main(String[] args) {
        SetDaemon o = new SetDaemon();
        Thread t = new Thread(o,"t1");
        t.setDaemon(true);
        t.start();
        System.out.println("主线程 start");
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("主线程 end");

    }
}

运行结果：
主线程 start
t1 count:1
t1 count:2
主线程 end
```
### volatile
    保证线程间可见性  
        MESI（缓存一致性协议）
        
    禁止指令重排序
        DCL单例（Double check lock）
        跟CPU有关系
        
    堆内存，所有线程共享的内存；
    每个线程都有自己的内存  
    
    线程之间不可见,不加volatile:线程内存中的数据马上写回堆内存，什么时候读不好控制
    
##### MESI

    MESI（Modified Exclusive Shared Or Invalid）
    (也称为伊利诺斯协议，是因为该协议由伊利诺斯州立大学提出）是一种广泛使用的支持写回策略的缓存一致性协议。
    
    CPU高速缓存
    
    CPU中每个缓存行（caceh line)使用4种状态进行标记（使用额外的两位(bit)表示)
    
    M: 被修改（Modified)
    E: 独享的（Exclusive)
    S: 共享的（Shared)
    I: 无效的（Invalid）
    
    == 指令重排序的代码==

##### 知识点1：线程可见性
    volatile 关键字，使一个变量在多个线程间可见
    A B线程都用到一个变量，java默认是A线程中保留一份copy，这样如果B线程修改了该变量，则A线程未必知道
    使用volatile关键字，会让所有线程都会读到变量的修改值
 
    在下面的代码中，running是存在于堆内存的t对象中  当线程t1开始运行的时候，会把running值从内存中读到t1线程的工作区，在运行过程中直接使用这个copy，并不会每次都去
    读取堆内存，这样，当主线程修改running的值之后，t1线程感知不到，所以不会停止运行
    
    使用volatile，将会强制所有线程都去堆内存中读取running的值
    
    == 可以阅读这篇文章进行更深入的理解
    http://www.cnblogs.com/nexiyi/p/java_memory_model_and_thread.html==
    

```
public class C_012 {
    private volatile boolean running =true;

    public void m(){
        System.out.println(Thread.currentThread().getName()+" start");
        while (running){

        }
        System.out.println(Thread.currentThread().getName()+" end");
    }

    public static void main(String[] args) {
        C_012 t = new C_012();
        new Thread(t::m).start();
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        t.running=false;

    }
}
运行结果：
Thread-0 start
Thread-0 end
```
    
##### 知识点2：单例模式-升级为DCL

应用：权限管理者  
饿汉式---懒汉式--锁细化--DCL

```
/**
 * DCL
 * double check lock
 * 最小的同步代码块，执行效率高，并且线程是安全的
 **/
public class Singleton_003 {
    public volatile static Singleton_003 INSTANCE;

    public Singleton_003(){}

    public static Singleton_003 getINSTANCE(){
        //其他业务代码
        if(INSTANCE ==null){
            synchronized (Singleton_003.class){
                if(INSTANCE ==null){
                    INSTANCE = new Singleton_003();
                }
            }
        }
        return INSTANCE;
    }

    public static void main(String[] args) {
        for(int i=0;i<10;i++){
            new Thread(()->{
                System.out.println(Singleton_003.getINSTANCE().hashCode());
            }).start();
        }
    }
}
```
##### 面试题：听说过单例模式吗？单例模式中有DCL模式听说过吗？那么DCL要不要加volatile?（美团）
    要加volatile
    第一线程间可见，会出现指令重排序的问题
    
##### 知识点3：DCL为什么要加volatile？
    INSTANCE = new Singleton_003(); 这段代码经过JVM的编译器编译后的指令分为三步：  
    
    第一步：给对象申请内存，成员变量的值为默认值
    第二步：给对象的成员变量初始化
    第三步：把内存的内容赋值给INSTANCE（栈内存）
    
    超高并发的时候会发生指令重排序
        例如：秒杀的时候
        第三步和第二步换了位置
        
        如下图Object字节码指令截图中：
        invokespecial会和astore_1进行交换
        
==synchronized保证了原子性，但是不能阻止重排序==  
==volatile不能保证原子性，不能替换synchronized==  
    
##### 补充知识：  
    IDEA下查看Java字节码插件：jclasslib Bytecode viewer
        
    查看方式：先执行一次代码，然后IDEA--Vies---Show Bytecode with jclaslib  
        
![makedown](https://raw.githubusercontent.com/musictaste/JUC/master/image/201.png)  

**Effctive java中单例是支持枚举的（枚举单例）**
    
    代码写在存储过程中效率非常高
        存储过程是跑在数据库内部的，由数据库引擎直接执行
        读一行数据，处理，再读一行数据
        
        程序：读一行数据，处理，数据写回去
        再读一行数据
        如果读写不在一台机器上，还要经历数据传输，效率差成百上千倍，上万倍也有可能
        
    
    == 内存屏障==
    
##### 知识点4：volatile不能保障原子性，不能替换synchronized
==volatile并不能保证多个线程共同修改runing变量时锁带来的不一致问题，也就是说volatile不能替代synchronized==
    
```
public class C_013 {
    private volatile int count =0;

    public /*synchronized*/ void m (){
        for(int i=0;i<10000;i++){
            count++;
        }
    }

    public static void main(String[] args) {
        C_013 t = new C_013();

        List<Thread> threadList = new ArrayList<>();
        for(int i=0;i<10;i++){
            threadList.add(new Thread(t::m));
        }
        threadList.forEach((o)->o.start());

        threadList.forEach((o)-> {
            try {
                o.join();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
        System.out.println(t.count);
    }
}

运行结果：
28625
32745
每一次运行结果都不一样
```

    
    原因分析：  
    volatile保证了可见性，即count的值可以被其他线程看到  
    但是count++这个操作不是原子性操作  
        
    举例：第一个线程，count++以后，count=1
        第二个线程，读到了count=1，count++,count变为2
        
        第三个线程，也读到了count=1,count++以后，count值还为2
        
如何解决：m方法增加synchronized
    
    
```
public class T05_VolatileVsSync {
	/*volatile*/ int count = 0;

	synchronized void m() { 
		for (int i = 0; i < 10000; i++)
			count++;
	}

	public static void main(String[] args) {
		T05_VolatileVsSync t = new T05_VolatileVsSync();

		List<Thread> threads = new ArrayList<Thread>();

		for (int i = 0; i < 10; i++) {
			threads.add(new Thread(t::m, "thread-" + i));
		}

		threads.forEach((o) -> o.start());

		threads.forEach((o) -> {
			try {
				o.join();
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		});

		System.out.println(t.count);

	}

}
```
**对比上一个程序，可以用synchronized解决，synchronized可以保证可见性和原子性，volatile只能保证可见性**
当然是因为join，线程顺序执行，如果没有join，就需要加volatile

#### 知识点5：锁优化
    第一种方式：锁细化
        
    第二种方式：锁粗化
        应用于：竞争非常激烈的时候
    
    
```
/**
 * synchronized优化
 * 同步代码块中的语句越少越好
 * 比较m1和m2
 * @author mashibing
 */
public class FineCoarseLock {
	
	int count = 0;

	synchronized void m1() {
		//do sth need not sync
		try {
			TimeUnit.SECONDS.sleep(2);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		//业务逻辑中只有下面这句需要sync，这时不应该给整个方法上锁
		count ++;
		
		//do sth need not sync
		try {
			TimeUnit.SECONDS.sleep(2);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
	
	void m2() {
		//do sth need not sync
		try {
			TimeUnit.SECONDS.sleep(2);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		//业务逻辑中只有下面这句需要sync，这时不应该给整个方法上锁
		//采用细粒度的锁，可以使线程争用时间变短，从而提高效率
		synchronized(this) {
			count ++;
		}
		//do sth need not sync
		try {
			TimeUnit.SECONDS.sleep(2);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}

}
```
  
##### 知识点6：==避免将锁定对象的引用变成另外的对象== 

    锁定某对象o，如果o的属性发生改变，不影响锁的使用
    但是如果o变成另外一个对象，则锁定的对象发生改变
    应该避免将锁定对象的引用变成另外的对象

```
public class SyncSameObject {
	
	/*final*/ Object o = new Object();

	void m() {
		synchronized(o) {
			while(true) {
				try {
					TimeUnit.SECONDS.sleep(1);
				} catch (InterruptedException e) {
					e.printStackTrace();
				}
				System.out.println(Thread.currentThread().getName());
				
				
			}
		}
	}
	
	public static void main(String[] args) {
		SyncSameObject t = new SyncSameObject();
		//启动第一个线程
		new Thread(t::m, "t1").start();
		
		try {
			TimeUnit.SECONDS.sleep(3);
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		//创建第二个线程
		Thread t2 = new Thread(t::m, "t2");
		
		t.o = new Object(); //锁对象发生改变，所以t2线程得以执行，如果注释掉这句话，线程2将永远得不到执行机会
		
		t2.start();
	}
}

```
  
  
##### 知识点7：不要以字符串常量作为锁定对象

    不要以字符串常量作为锁定对象
    在下面的例子中，m1和m2其实锁定的是同一个对象
    这种情况还会发生比较诡异的现象，比如你用到了一个类库，在该类库中代码锁定了字符串“Hello”，
    但是你读不到源码，所以你在自己的代码中也锁定了"Hello",这时候就有可能发生非常诡异的死锁阻塞，
    因为你的程序和你用到的类库不经意间使用了同一把锁
    
***
### 总结

1. volatile
- 保证线程可见性
    
    MESI缓存一致性协议
    
- 禁止指令重排序  
    DCL单例（Double check lock）
        
    ==指令重排序是发生在cpu层面的==  
    ==volatile是JVM层面的操作==
        
    ==内存屏障==  
    ==LoadFence和StoreFence==两条原语指令保证不会发生指令重排序
          
    ==synchronized保证了原子性，但是不能阻止重排序==  
    ==volatie不能保证原子性，所以不能替换synchronized==
    
    也就是说volatile并不能保证多个线程共同修改volatile变量时所带来的不一致问题
    volatile修饰的变量，线程间是可见的，但是对于变量的修改不能保证原子性操作，所以不能替代synchronized
    
    避免将锁定对象的引用变成另外的对象
2. 锁优化  
    锁细化   
    锁粗化
    


***
==内存屏障==  
 
==指令重排序的代码==

==ASM ==
        
***
文章：[-XX：PertenureSizeThreshold的默认值和作用](https://www.jianshu.com/p/f7cde625d849)  

阅读这篇文章进行更深入的理解
http://www.cnblogs.com/nexiyi/p/java_memory_model_and_thread.html==