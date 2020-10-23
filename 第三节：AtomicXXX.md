[TOC]

##### 回顾
    内存屏障：  
        loadFence StoreFence
    
##### 知识1：AtomicXXX
    
```
public class AtomicIntegerTest {
    /*private volatile int count =0;

    public synchronized void m (){
        for(int i=0;i<10000;i++){
            count++;
        }
    }*/

    AtomicInteger count = new AtomicInteger(0);
    public void m(){
        for(int i =0;i<10000;i++){
            count.incrementAndGet();
        }
    }

    public static void main(String[] args) {
        AtomicIntegerTest t = new AtomicIntegerTest();
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
```

##### 知识2：increment（三种不同方式实现递增）
    JMH 专业性能测试
```
public class AtomicVsSyncVsLongAddr {
    private static long count1 = 0;
    private static AtomicLong count2 = new AtomicLong(0);
    private static LongAdder count3 = new LongAdder();

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads  =  new Thread[1000];

        Object lock = new Object();
        for(int i =0;i<threads.length;i++){
            threads[i]=new Thread(()->{
                for(int j=0;j<100000;j++){
                    synchronized (lock){
                        count1++;
                    }
                }
            });
        }
        Long start = System.currentTimeMillis();
        for(Thread t:threads) t.start();
        for(Thread t:threads) t.join();
        Long end = System.currentTimeMillis();
        System.out.println("Synchronized  "+count1+" time:"+(end-start));


        for(int i=0;i<threads.length;i++){
            threads[i]= new Thread(()->{
                for(int j=0;j<100000;j++){
                    count2.incrementAndGet();
                }
            });
        }
        start = System.currentTimeMillis();
        for(Thread t:threads) t.start();
        for(Thread t:threads) t.join();
        end = System.currentTimeMillis();
        System.out.println("Atomic  "+count2+" time:"+(end-start));


        for(int i=0;i<threads.length;i++){
            threads[i]= new Thread(()->{
                for(int j=0;j<100000;j++){
                    count3.increment();
                }
            });
        }
        start = System.currentTimeMillis();
        for(Thread t:threads) t.start();
        for(Thread t:threads) t.join();
        end = System.currentTimeMillis();
        System.out.println("LongAdder  "+count3+" time:"+(end-start));
    }
}

第一次结果：
Synchronized  100000000 time:1954
Atomic  100000000 time:1954
LongAdder  100000000 time:191

第二次结果：
Synchronized  100000000 time:1689
Atomic  100000000 time:1727
LongAdder  100000000 time:182
```

    
    结果：
    LongAdder > Atomic > Sync
    这个结果只是说明一个大概
    备注：如果并发线程数以及执行次数发生变化，需要再实际模拟执行效率，上面的结果只做参考

    也就是说，如果线程数不是1000，执行次数不是10万，longAdder不一定就比Atomic快，Synchronized也不一定就会慢
    
##### 知识3：LongAdder   
    面试比较深的知识点
    
    LongAdder也是CAS操作，但内部是分段锁
    
    将要执行的线程，分段放入不同的数组中，让他们分别执行，最后求数组之和
    
    线程数特别多的时候，有优势
![301](E39118934A0C4B8DBA8116426A4E40CD)
    
### 总结：

    三种不同方式实现递增
        synchronized
        AtomicXXX
        LongAddr
        
        LongAddr（分段锁）、CAS(自旋锁)、重量级锁
        性能跟线程数、以及执行的操作有关系
        
