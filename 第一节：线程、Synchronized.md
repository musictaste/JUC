## 了解

    disruptor：MQ，单机环境下效率最高的消息队列

    学习:  
        上天：  
            锻炼解决问题的能力  
            高并发、缓存、 大流量、大数据量  

        入地：  
            面试  
            JVM  OS  算法     线程  IO

## 1.基础概念

### 1.1 线程

![image](https://raw.githubusercontent.com/musictaste/JUC/master/image/02.jpg)  

    进程：应用程序QQ，登录  
    线程:一个程序里不同的执行路径就叫做线程  
    协程/纤程(quasar)
      
### 1.2 线程的定义与启动
    第一种： extends Thread，重写run方法；  
        启动start方法，不是run方法  
**run方法是顺序执行，start方法是异步执行**

    第二种：implements Runnable  重写run方法；  
        启动需要创建Thread，把run对象放入Thread，然后执行Thread的start方法

    第三种，第二种的变形，jdk8，lamda表示

    第四种：通过线程池，Executors.newCachedThredPool，实际线程池也是通过Thread和Runnable中的一种
```java
public class T02_HowToCreateThread {
    static class MyThread extends Thread {
        @Override
        public void run() {
            System.out.println("Hello MyThread!");
        }
    }

    static class MyRun implements Runnable {
        @Override
        public void run() {
            System.out.println("Hello MyRun!");
        }
    }

    public static void main(String[] args) {
        new MyThread().start();
        new Thread(new MyRun()).start();
        new Thread(()->{
            System.out.println("Hello Lambda!");
        }).start();
    }

}
```
 
##### 面试题：//请你告诉我启动线程的五种方式  
    1：Thread  
    2: Runnable  
    3:Executors.newCachedThred  
        第三种通过线程池，实际线程池也是使用的Thread和Runnable中的一种，当然通过start方法启动
      
### 1.3 常用方法：Sleep/Yield/Join
    Sleep,sleep完回到就绪状态  
        Thread.sleep(500);  

    Yield ：让出一下cpu，回到就绪状态，至于其他线程能不能抢到，它不管  
        应用场景：很少，没有用过；性能测试的压测可能会用到

    Join：让另一个线程加入进来，用于：等待另外一个线程的结束  


##### 面试题：三个线程T1,T2,T3,怎么保证三个线程顺序执行  
    答案：在主方法中，分别调用T1.join()/T2.join()/T3.join()  
        更精确的实现：T1中调用T2.join(),T2中调用T3.join()  

### 1.4 线程的状态：知识点面试不多，学术研究

![makedown](https://raw.githubusercontent.com/musictaste/JUC/master/image/03.png)  

    查看状态：o.getState()  

    1.new  
    
    2.Runnable  
        ready  
        running  

    3.TimeWaiting  
        Thread.sleep(time)  
        o.wait(time)  
        o.join(time)  
        LockSupport.parkNanos()  
        LockSupport.parkUntil()  

    4.Waiting  
        o.wait()  
        o.join()  
        LockSupport.park();  
    
        o.notify();  
        o.notifyAll();  
        LockSupport.unpark();  
    
    5.Blocked  
        syschronized  
    
    6.Teminated  

    线程被挂起：cpu控制  

##### ==问题1：这里说的线程跟操作系统中的线程是一一对应的吗==？  
    这个的看JVM，以前的JVM跟操作系统是一一对应的  
    现在呢，不好说；hotspot应该是一一对应的  

但是纤程跟操作系统中的线程不是一一对应的  

##### 问题2：如何关闭线程？  
    自己不要关闭线程，JDK中的stop()这个方法太粗暴，容易导致线程状态的不一致  
        而应该让线程正常结束

##### 问题3：阻塞状态，interrupt()  
    JDK锁的原码，以及Netty源码中中会使用interrupt()的控制逻辑，是为了程序的健壮

    但是在项目代码中很少会用到interrupt()方法来控制  业务逻辑

    应用：当你在读取一个网络文件的时候，如果没有成功，设置了Thread.sleep,   
    并且sleep了很长时间，如果你想结束这个线程，调用interrupt方法，并捕获InterruptedException异常，进行资源释放

    参考文章：https://blog.csdn.net/m0_37605407/article/details/86710196

**Java在设计阻塞时中断方法时，即在调用interrupt()设置线程为中断状态后，如果线程遇到阻塞就会抛出InterruptedException异常，并且将会重置线程的中断状态**

```java
public class text3 {
	public static void main(String[] args) {
		Thread t = new Thread(){
			@Override
			public void run() {
				// TODO Auto-generated method stub
				super.run();
				System.out.println("遇到阻塞");				
				try {
					Thread.sleep(1000);
				} catch (InterruptedException e) {
					System.out.println("阻塞被打断");
					// 在这里清理资源
					e.printStackTrace();
				}
				System.out.println("阻塞结束");
				System.out.println("结束线程");
			}
		};
		t.start();
		try {
			Thread.sleep(500); // 为了确保线程已经运行
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("调用interrupt（）");
		t.interrupt();
	}
}

运行结果
    遇到阻塞
    调用interrupt（）
    阻塞被打断
    java.lang.InterruptedException: sleep interrupted
        at java.lang.Thread.sleep(Native Method)
        at text$1.run(text.java:17)
    阻塞结束
    结束线程
```

可以这么理解，调用interrupt()，便将线程的中断标志位置为true，当遇到阻塞（或者在调用interrupt方法中，正处于阻塞），都会立马中断阻塞，将线程的中断标志位置为false，并抛出异常  

**除了遇到阻塞会抛出异常，并重置线程中断状态，Thread还有提供了一个interrupted()的静态方法，可以将当前的线程的中断状态重置**
```
public class text1_1 {
	public static void main(String[] args) {
		new Thread(){
			@Override
			public void run() {
				// TODO Auto-generated method stub
				super.run();
				interrupt(); // 将线程置为中断状态
				Thread.interrupted(); // 重置线程中断状态
				System.out.println("遇到阻塞");				
				try {
					Thread.sleep(1000);
				} catch (InterruptedException e) {
					System.out.println("阻塞被打断");
					// 在这里清理资源
					e.printStackTrace();
				}
				System.out.println("阻塞结束");
				System.out.println("结束线程");
			}
		}.start();
		
	}
```

##### 问题4：这块面试题多吗？  
    面试题不多，学术研究比较多  
    linux中进程跟线程差不多  

### 1.5 syschronized关键字

#### 知识1：锁的是对象；  
    锁定一段代码块，指的是锁定对象以后，才能执行这段代码  
      
    公开课：hashcode与markword  
      
#### 知识2：syschronized的底层实现
    其实JVM并没有做具体的规范  
    hotspot是在这个对象的头上面，记录锁信息  

    记住下面这张图  
![makedown](https://raw.githubusercontent.com/musictaste/JUC/master/image/04.png)
      
#### 知识3： synchronized 方法等同于synchronized(this)  
    细微区分：如果synchronized(this)中只有这一个方法则没有区别  
```
public class T1 {
   private int count = 10;
   public synchronized void m() { //等同于在方法的代码执行时要synchronized(this)
      count--;
      System.out.println(Thread.currentThread().getName() + " count = " + count);
   }
   public void n() { //访问这个方法的时候不需要上锁
      count++;
   }
}
```


#### 知识4：public static synchronized m(){}等同于synchronized(T.class)  
```
public class T {
   private static int count = 10;
   public synchronized static void m() { //这里等同于synchronized(FineCoarseLock.class)
      count--;
      System.out.println(Thread.currentThread().getName() + " count = " + count);
   }
   public static void mm() {
      synchronized(T.class) { //考虑一下这里写synchronized(this)是否可以？
         count --;
      }
   }
}
```

##### 问题：synchronized(T.class)是单例的吗？  
    一般情况下是单例的。  
    同一个classLoader是单例，不同classLoader呢，不是单例的，但是不同classLoader也不能互相访问，不能访问也就没有关系  
          
#### 知识5：小程序：005 有什么问题  
```
public class C_005 {
    private /*volatile*/ int count =10;

    public synchronized void m(){
        count--;
        System.out.println(Thread.currentThread().getName()+",count:"+count);
    }

    public static void main(String[] args) {
        C_005 t = new C_005();
        for(int i=0;i<10;i++){
            new Thread(t::m,"Thread"+i).start();

//            new Thread(()->t.m(),"Thread"+i).start();

            //1.8之前的写法
            /*new Thread(new Runnable() {
                @Override
                public void run() {
                    t.m();
                }
            },"Thread"+i).start();*/
        }
    }

}
```
        
    m方法没有加synchronized关键字，出现的问题的是：  
    如果线程1，count-- count变为9, 现在线程2也执行了count-- count变为8   
    那么在打印的时候，打印出来的count=8，而不是9  
            
    解决:m方法加synchronized关键字  
      
    另外呢，count 不需要加volatie关键字；volatile和synchronized到达的效果一样，除了线程间可见性  
     
##### 知识6：面试题：同步和非同步方法是否可以同时调用?   
    可以  
    举例：你上厕所，别人可以同时擦马桶
```
public class C_007 {
    public synchronized void m1(){
        System.out.println(Thread.currentThread().getName()+",m1,start");
        try {
            TimeUnit.MILLISECONDS.sleep(10000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName()+",m1,end");
    }

    public void m2(){
        try {
            TimeUnit.MILLISECONDS.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName()+",m2 end");
    }

    public static void main(String[] args) {
        C_007 t = new C_007();
        new Thread(t::m1,"Thread1").start();
        new Thread(t::m2,"Thread2").start();
    }
}

Thread1,m1,start
Thread2,m2 end
Thread1,m1,end
```
  
       
##### 面试题：模拟银行账户，对业务写方法加锁，对业务读方法不加锁，这样行不行？  
    如果业务逻辑允许加锁，则不加锁；加锁以后，性能影响100倍  
    如果业务逻辑不允许，会产生脏读（dirty read ），这时则不允许  
    
```
public class C_008 {
    private String name;
    private int balance;

    public synchronized void write(String name,int balance){
        this.name = name;
        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        this.balance = balance;
    }

    public int read(){
        return this.balance;
    }

    public static void main(String[] args) {
        C_008 t = new C_008();
        new Thread(()->t.write("test",100)).start();

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(t.balance);

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(t.balance);
    }
}
```

      
##### 知识7：可重入：  
```
public class C_009 {
    synchronized void m1(){
        System.out.println("m1 start");

        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        m2();
        System.out.println("m1 end");
    }

    synchronized void m2(){
        System.out.println("m2");
    }

    public static void main(String[] args) {
        C_009 t = new C_009();
        new Thread(t::m1).start();
    }
}
```
  
    可重入：一个同步方法可以调用另一个同步方法，一个线程已经拥有某个对象的锁，再次申请的时候仍然会得到该对象的锁，也就是说synchronized获得的锁是可重入的  
    
    必须可以，如果不可以会发生死锁
          
    父子类的测试（程序010），如果不可重入，则会发生死锁。  
        supper.m 
    
    m和supper.m都加了锁  
    锁的TT（子类），TT中有指针指向T（父类），锁的还是同一个对象，是可以的，也是重入锁  
```
public class C_010 {
    synchronized void m(){
        System.out.println("super start");
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("supper end");

    }

    public static void main(String[] args) {
        new Thread(()->new TT().mm()).start();
    }

}
class TT extends C_010{
     synchronized void mm(){
        System.out.println("child start");
        super.m();
        System.out.println("child end");
    }
}

child start
super start
supper end
child end
```

##### 面试题：为什么Synchronized是可重入的?  
    如果不可以会发生死锁  

##### 知识点8：==异常情况下,会释放锁== 
    程序在执行过程中，如果出现异常，默认情况锁会被释放；

    所以，在并发处理的过程中，有异常要多加小心，不然可能会发生不一致的情况。
    比如，在一个web app处理过程中，多个servlet线程共同访问同一个资源，这时如果异常处理不合适，在第一个线程中抛出异常，其他线程就会进入同步代码区，有可能会访问到异常产生时的数据。
    因此要非常小心的处理同步业务逻辑中的异常
```
public class C_011 {
    private int count =0;
    synchronized void m(){
        while (true){
            count++;
            System.out.println(Thread.currentThread().getName()+",count:"+count);
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            if(count ==5){
                int i =1/0;
                System.out.println(i);
            }
        }

    }

    public static void main(String[] args) {
        C_011 t = new C_011();
        Runnable r= new Runnable() {
            @Override
            public void run() {
                t.m();
            }
        };

        new Thread(r,"t1").start();
        try {
            TimeUnit.SECONDS.sleep(1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        new Thread(r,"t2").start();
    }
}

t1,count:1
t1,count:2
t1,count:3
t1,count:4
t1,count:5
t2,count:6
Exception in thread "t1" java.lang.ArithmeticException: / by zero
	at com.mashibing.mycode.C_011.m(C_011.java:24)
	at com.mashibing.mycode.C_011$1.run(C_011.java:36)
	at java.lang.Thread.run(Thread.java:748)
t2,count:7
t2,count:8
t2,count:9
t2,count:10
````
     

##### 知识点9:  synchronized的底层实现以及锁升级  

    JDK早期的 重量级 - OS--效率低  
    
    后来JDK1.5改进  
        锁升级的概念：  
        文章：我就是厕所所长 （一 二）  

    
- ==锁升级---Hotspot的实现==：  
        
    1.sync (Object)（无锁态） 
    2.markword 记录这个线程ID （偏向锁）  
    3.如果线程争用：升级为   自旋锁(轻量级锁、无锁、自适应自旋锁)  
        使用CAS技术  
    4.自旋锁旋10次以后，升级为重量级锁 - OS  

- 锁降级  

    ==如果一些线程结束以后，锁并不会降级==  
    特殊情况下，GC的时候

- 用户态

    ==自旋锁是占CPU的==  
    很多服务需要内核态来帮助完成，比如启动、关闭、切换一个线程

- 内核态：  
    
    ==线程是放入操作系统的等待队列，不占cpu==

- 应用场景：  
        
    **执行时间短（加锁代码），线程数少，用自旋**  
    **执行时间长，线程数多，用系统锁**  

- ==synchronized(Object)规定：不能用String常量、Integer、Long==

    原因：String常量，导致其他使用String的类库，同一个线程会出现重入现象；如果不是同一个线程会发生死锁情况；  

    Integer，一旦值改变，会生产一个新的对象 

```
/**
 * 不要以字符串常量作为锁定对象
 * 在下面的例子中，m1和m2其实锁定的是同一个对象
 * 这种情况还会发生比较诡异的现象，比如你用到了一个类库，在该类库中代码锁定了字符串“Hello”，
 * 但是你读不到源码，所以你在自己的代码中也锁定了"Hello",这时候就有可能发生非常诡异的死锁阻塞，
 * 因为你的程序和你用到的类库不经意间使用了同一把锁
 * 
 * jetty
 * 
 * @author mashibing
 */
public class DoNotLockString {
	
	String s1 = "Hello";
	String s2 = "Hello";

	void m1() {
		synchronized(s1) {
			try {
				System.out.println("m1 start");
				System.in.read();
			} catch (IOException e) {
				e.printStackTrace();
			}

			System.out.println("m1 end");
		}
	}
	
	void m2() {
		synchronized(s2) {
			try {
				System.out.println("m2 start");
				System.in.read();
			} catch (IOException e) {
				e.printStackTrace();
			}
			System.out.println("m2 end");
		}
	}

	public static void main(String[] args) {
		DoNotLockString object = new DoNotLockString();
		new Thread(()->object.m1(),"T1").start();
		new Thread(()->object.m2(),"T2").start();
	}

}

运行结果：
m1 start

```



***
## 总结：
    1.线程的概念、启动方式、常用方法、状态   
        三种创建方式
        五种启动方式
        6种状态
![makedown](https://raw.githubusercontent.com/musictaste/JUC/master/image/03.png)  

    2.synchronized(Object)  
        规定：不能用String常量、Integer、Long  

        原因：String常量，导致其他使用String的类库，同一个线程会出现重入现象；如果不是同一个线程会发生死锁情况；  

        Integer，一旦值改变，会生产一个新的对象

    3.线程同步  
        3.1 synchronized  
            锁的是对象，不是代码  
            this  
            static --T.class  
            锁定方法与非锁定方法可以同时执行  
            同步过程中，发生异常，会释放锁  
            锁升级  
            无锁态、偏向锁、自旋锁、重量级锁  
            自旋锁以及重量级锁的应用场景  
            执行时间短（加锁代码），线程数少，用自旋  
            执行时间长，线程数多，用系统锁  
            用户态、内核态  
    4.锁升级的详细内容
    
    
![makedown](https://raw.githubusercontent.com/musictaste/JUC/master/image/04.png)
  
  
***
synchronized方法的线程调用
```
private /*volatile*/ int count =10;
public synchronized void m(){
    count--;
    System.out.println(Thread.currentThread().getName()+",count:"+count);
}

new Thread(t::m,"Thread"+i).start();

new Thread(()->t.m(),"Thread"+i).start();

//1.8之前的写法
new Thread(new Runnable() {
    @Override
    public void run() {
        t.m();
    }
},"Thread"+i).start();
```


---
==锁升级的详细内容==