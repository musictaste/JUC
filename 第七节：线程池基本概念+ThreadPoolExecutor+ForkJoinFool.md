### 线程池
    
##### 概念

```
public class T00_MyExecutor implements Executor {
    public static void main(String[] args) {
        new T00_MyExecutor().execute(()->{
            System.out.println("executor execute");
        });

        ExecutorService executorService = Executors.newCachedThreadPool();
    }

    @Override
    public void execute(Runnable command) {
        command.run();
    }
}
```

```
public class T04_ExecutorService {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService service = Executors.newFixedThreadPool(5);
        for (int i = 0; i < 6; i++) {
            service.execute(()->{
                try {
                    TimeUnit.MILLISECONDS.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName());
            });
        }

        System.out.println(service);

        service.shutdown();

        //执行1
        /*System.out.println(service.isTerminated());
        System.out.println(service.isShutdown());*/

        //执行2
        TimeUnit.SECONDS.sleep(5);
        System.out.println(service.isTerminated());
        System.out.println(service.isShutdown());

        System.out.println(service);

    }
}

运行结果：执行1
java.util.concurrent.ThreadPoolExecutor@5e9f23b4[Running, pool size = 5, active threads = 5, queued tasks = 1, completed tasks = 0]
false
true
java.util.concurrent.ThreadPoolExecutor@5e9f23b4[Shutting down, pool size = 5, active threads = 5, queued tasks = 1, completed tasks = 0]
pool-1-thread-4
pool-1-thread-3
pool-1-thread-1
pool-1-thread-2
pool-1-thread-5
pool-1-thread-4

运行结果：执行2
java.util.concurrent.ThreadPoolExecutor@5e9f23b4[Running, pool size = 5, active threads = 5, queued tasks = 1, completed tasks = 0]
pool-1-thread-1
pool-1-thread-4
pool-1-thread-3
pool-1-thread-5
pool-1-thread-2
pool-1-thread-1
true
true
java.util.concurrent.ThreadPoolExecutor@5e9f23b4[Terminated, pool size = 0, active threads = 0, queued tasks = 0, completed tasks = 6]

```

    Executor
        线程的定义和运行可以分开了
        
        是异步的吗？
            不是一个概念，因为把线程放到Executor线程池中，让线程池来运行
            
    ExecutorService
        完善了任务执行器的生命周期
        shutdown//结束
        shutdownNow//马上结束
        isShutdown// 是不是整体都执行完了
        awaitTermination//等着结束，等多长时间，时间到了还不结束的话他返回false
        
        submit
            异步 提交任务
            涉及的类：
                -Future
                    -get() 阻塞的
                -Callable
        
        源码：
            submit

    
    线程池分两种类型：
        ThreadPoolExecutor
            
            
        ForkJoinFool
            -分解汇总的任务
            -用很少的线程可以执行很多的任务（子任务）TPE做不到先执行子任务
            -CPU密集型
        
    
    代码-三个概念    
    
###### Callable

    跟Runnable一样，可以让另外一个线程运行它
    区别：call()有返回值，Runnable没有返回值
        Callable=Runnable + result
    
    Callable为线程池而设计的
    用Future来存储Callable运行的结果

```
public class T01_Callable {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        /*Callable<String> call = new Callable<String>() {
            @Override
            public String call() throws Exception {
                return "callable";
            }
        };*/
        Callable<String> call = ()->{
            return "callable";
        };

        ExecutorService service = Executors.newCachedThreadPool();
        Future<String> future = service.submit(call);//异步
        String result = future.get();//阻塞
        System.out.println(result);

        service.shutdown();
    }
}
```

###### Future
    
    存储将来执行的结果
    
###### FutureTask
    
    class FutureTask<V> implements RunnableFuture<V>
    interface RunnableFuture<V> extends Runnable, Future<V>
    
    FutureTask=Runnable + Future
    把Runnable运行的结果存储在自己中
    
    ForkJoinPool和WorkStealingPool会用到
    
```
public class T02_FutureTask {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        FutureTask<Integer> task  = new FutureTask<>(()->{
           return 100;
        });

        new Thread(task).start();
        System.out.println(task.get());//阻塞
    }
}
```
###### CompletableFuture(面试会问，但是实际开发可以研究)
    
    底层用ForkJoinPool
    
    是各种任务的管理类
    
    管理多个Future的结果
    
    对任务的组合
    对任务的lambda的使用
    
    
    supplyAsync
    
    allOf()
    anyOf()
        论文查重，只要一个查到重复就可以
        
    ==读API的用法

面试题：如何对多个Future进行管理
    
    使用CompletableFuture
    
面试题：假设你能够提供一个服务，这个服务查询各大电商网站同一类商品 价格并汇总展示


```
public class T03_CompletableFuture {
    public static void delay(){
        int time  = new Random().nextInt(500);
        try {
            TimeUnit.MILLISECONDS.sleep(time);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.printf("after %s sleep\n",time);
    }

    public static double priceOfTB(){
        delay();
        return 1.00;
    }

    public static double priceOfJD(){
        delay();
        return 2.00;
    }

    public static double priceOfPDD(){
        delay();
        return 2.50;
    }

    public static void main(String[] args) {
        long start,end;

        start = System.currentTimeMillis();

        CompletableFuture<Double> future_JD  = CompletableFuture.supplyAsync(()->priceOfJD());
        CompletableFuture<Double> future_TB  = CompletableFuture.supplyAsync(()->priceOfTB());
        CompletableFuture<Double> future_PDD  = CompletableFuture.supplyAsync(()->priceOfPDD());

        CompletableFuture.allOf(future_JD,future_TB,future_PDD).join();

        CompletableFuture.supplyAsync(()->priceOfJD())
                .thenApply(String::valueOf)
                .thenApply(str-> "price:"+str)
                .thenAccept(System.out::println);

        end = System.currentTimeMillis();

        System.out.println("complete future need time "+(end-start));

        try {
            System.in.read();//
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

运行结果：
after 167 sleep
after 254 sleep
after 258 sleep
complete future need time 263
after 18 sleep
price:2.0
```

###### ThreadPoolExecutor

    extends AbstractExecutorService
    
    阿里手册要求线程池自定义
    
    维护两个集合
        一个线程的集合
        一个任务的集合
        
        底层数据结构：
            HashSet<Worker> workers = new HashSet<>() 
            BlockingQueue<Runnable> workQueue
    
==**线程池定义的7个参数(背会)**==

    int corePoolSize  核心线程，
        核心线程即使最长空闲时间到了，并不归还给操作系统
    
    int maximumPoolSize 最大线程
    long keepAliveTime 最长空闲时间
        灭活
        
    TimeUnit unit 空闲时间的的单位
    
    BlockingQueue<Runnable> workQueue  任务队列
        ArrayBlockingQueue
        LinkedBlockingQueue
        SynchronusQueue   进来一个任务，处理一个任务
        TransferQueue 
        DelayQueue
    
    ThreadFactory threadFactory  线程工厂
        自己定义
            阿里手册规定：
            【强制】创建线程或线程池时请指定有意义的线程名称，方便出错时回溯
            
            【强制】 线程池不允许使用 Executors 去创建，而是通过 ThreadPoolExecutor 的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。
            说明： Executors 返回的线程池对象的弊端如下：
            1） FixedThreadPool 和 SingleThreadPool：允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM。
            2） CachedThreadPool：允许的创建线程数量为 Integer.MAX_VALUE， 可能会创建大量的线程，从而导致 OOM。
            
        默认：Executors.defaultThreadFactory

```
class DefaultThreadFactory

public Thread newThread(Runnable r) {
    Thread t = new Thread(group, r,
                          namePrefix + threadNumber.getAndIncrement(),
                          0);//指定线程名称
    if (t.isDaemon())
        t.setDaemon(false);//设置为非守护线程
    if (t.getPriority() != Thread.NORM_PRIORITY)
        t.setPriority(Thread.NORM_PRIORITY);//设置优先级
    return t;
}
```

        RejectedExecutionHandler handler  拒绝策略
            new ThreadPoolExecutor.DiscardOldestPolicy()
            
            线程池忙，并且线程池队列满了，执行拒绝策略
            
            拒绝策略可自定义
                实际项目都是自定义
                保存到kafka、redis、MQ、数据库，并做好日志
                如果有大量的任务没有处理，说明该加机器了
                
            JDK默认提供了四种，实际项目中很少用到这4个
                -Abort：抛异常
                -Discard:扔掉，不抛异常
                -DiscardOldest:扔掉排队时间最久的
                    应用场景：游戏的位置信息，扔掉最老的位置信息
                -CallerRuns:调用者处理任务
            
```
public class T05_ThreadPoolExecutor {

    static class Task implements Runnable{
        private int i;

        public Task(int i) {
            this.i = i;
        }

        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName()+" task "+i);
            try {
                System.in.read();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }

        @Override
        public String toString() {
            return "Task{i=}"+i;
        }
    }

    public static void main(String[] args) {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(2,4,
                60, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(4),
                Executors.defaultThreadFactory(),
                new ThreadPoolExecutor.CallerRunsPolicy()
                );

        for (int i = 0; i < 8; i++) {
            executor.execute(new Task(i));
        }

        System.out.println(executor.getQueue());

        executor.execute(new Task(100));
        System.out.println(executor.getQueue());

        executor.shutdown();
    }

}

运行结果：AbortPolicy
[Task{i=}2, Task{i=}3, Task{i=}4, Task{i=}5]
pool-1-thread-3 task 6
pool-1-thread-2 task 1
pool-1-thread-1 task 0
pool-1-thread-4 task 7
Exception in thread "main" java.util.concurrent.RejectedExecutionException: Task Task{i=}100 rejected from java.util.concurrent.ThreadPoolExecutor@7c3df479[Running, pool size = 4, active threads = 4, queued tasks = 4, completed tasks = 0]
	at java.base/java.util.concurrent.ThreadPoolExecutor$AbortPolicy.rejectedExecution(ThreadPoolExecutor.java:2055)
	at java.base/java.util.concurrent.ThreadPoolExecutor.reject(ThreadPoolExecutor.java:825)
	at java.base/java.util.concurrent.ThreadPoolExecutor.execute(ThreadPoolExecutor.java:1355)
	at com.mashibing.mycode.threadPoolExecutor.T05_ThreadPoolExecutor.main(T05_ThreadPoolExecutor.java:55)


运行结果：DiscardPolicy
[Task{i=}2, Task{i=}3, Task{i=}4, Task{i=}5]
[Task{i=}2, Task{i=}3, Task{i=}4, Task{i=}5]
pool-1-thread-3 task 6
pool-1-thread-2 task 1
pool-1-thread-4 task 7
pool-1-thread-1 task 0

运行结果：DiscardOldestPolicy
[Task{i=}2, Task{i=}3, Task{i=}4, Task{i=}5]
[Task{i=}3, Task{i=}4, Task{i=}5, Task{i=}100]
pool-1-thread-2 task 1
pool-1-thread-4 task 7
pool-1-thread-3 task 6
pool-1-thread-1 task 0

运行结果：CallerRunsPolicy
[Task{i=}2, Task{i=}3, Task{i=}4, Task{i=}5]
pool-1-thread-2 task 1
pool-1-thread-4 task 7
pool-1-thread-1 task 0
main task 100
pool-1-thread-3 task 6

```

### Executors：线程池的工厂
        
###### SingleThreadPool

    面试题：为什么要有单线程的线程池？
        线程池有任务队列，线程池有完整的生命周期管理
        保证任务的顺序
    
```
源码：
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}

```

```
public class T06_SimpleThreadPool {
    public static void main(String[] args) {
        ExecutorService service = Executors.newSingleThreadExecutor();
        for (int i = 0; i < 5; i++) {
            final int j = i;
            service.execute(()->{
                System.out.println(Thread.currentThread().getName()+" "+ j);
            });
        }
        service.shutdown();
    }
}
```

###### CachedThreadPool

```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}

SynchronousQueue为手递手，容量为0的队列

来一个线程，如果线程池中有空闲的线程，则使用；如果没有，则新起一个任务

```


```
public class T07_CachedThreadPool {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService service = Executors.newCachedThreadPool();
        System.out.println(service);
        for (int i = 0; i < 2; i++) {
            service.execute(()->{
                System.out.println(Thread.currentThread().getName());
            });
        }
        System.out.println(service);

        TimeUnit.MILLISECONDS.sleep(100);
        System.out.println(service);

    }
}

运行结果：
  java.util.concurrent.ThreadPoolExecutor@4769b07b[Running, pool size = 0, active threads = 0, queued tasks = 0, completed tasks = 0]
java.util.concurrent.ThreadPoolExecutor@4769b07b[Running, pool size = 2, active threads = 2, queued tasks = 0, completed tasks = 0]
pool-1-thread-1
pool-1-thread-2
java.util.concurrent.ThreadPoolExecutor@4769b07b[Running, pool size = 2, active threads = 0, queued tasks = 0, completed tasks = 2]      

```

###### FixedThreadPool

    应用：并行处理
    
    面试题：并发和并行的区别
        concurrent：指任务提交
            一个cpu可以并发多个线程
            
        parallel：指任务执行
            多个cpu同时处理
        
        并行是并发的子集
        
        参考书《深入理解计算机系统》
        
        小视频：B站能找到
        
        我的机器是6核12线程
        
> hotspot的线程跟操作系统的线程是一一对应的，将来有纤程就不知道了  
> linux操作系统的线程调度策略，有三种：优先级   时间片(默认)  实时
 
 
```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```


```
public class T08_FixedThreadPool {
    static boolean isPrime(int num){
        for (int i = 2; i < num/2; i++) {
            if(num%i == 0) return false;
        }
        return true;
    }

    static List<Integer> getPrimes(int start, int end){
        List<Integer> list = new ArrayList<>();
        for (int i = start; i <=end; i++) {
            if(isPrime(i)) list.add(i);
        }
        return list;
    }

    static class Mytask implements Callable<List<Integer>>{
        int start,end;

        public Mytask(int start, int end) {
            this.start = start;
            this.end = end;
        }

        @Override
        public List<Integer> call() throws Exception {
            return getPrimes(start,end);
        }
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        long start,end;
        start = System.currentTimeMillis();
        getPrimes(0,200000);
        end = System.currentTimeMillis();
        System.out.println(end-start);

        final int cpuCoreNum = 4;//CPU的核心线程数为4，我本机是6核12线程
        ExecutorService service = Executors.newFixedThreadPool(cpuCoreNum);
        Mytask task1 = new Mytask(0,50000);
        Mytask task2 = new Mytask(50001,100000);
        Mytask task3 = new Mytask(100001,150000);
        Mytask task4 = new Mytask(150001,200000);

        Future<List<Integer>> f1= service.submit(task1);
        Future<List<Integer>> f2=service.submit(task2);
        Future<List<Integer>> f3=service.submit(task3);
        Future<List<Integer>> f4=service.submit(task4);

        start = System.currentTimeMillis();
        f1.get();
        f2.get();
        f3.get();
        f4.get();
        end = System.currentTimeMillis();
        System.out.println(end-start);
        service.shutdown();
    }
}

运行结果：
1804
796

```


###### 调整线程池的大小

![801](52FE4BB8D7F147E8B7B029D82136163C)

    多数情况下，根据经验，设置好以后进行压测
    
    这个公式很难计算的，因为W和C的比率是很难确定的
        C中有读取IO的时间，一般count的时间是很短的，几乎为0
        Ucpu也是很难确定的，因为机器上还运行着其他的程序
        
    CPU密集型？
    IO密集型？
    

###### Cached VS Fixed
    
    如果任务忽高忽低，建议Cached
    如果来的任务比较平稳，采用Fixed
    
    阿里的建议是：两个都不用，自己进行估算以后，进行精确定义
    
###### ScheduledThreadPool定时任务线程池(应用不多,了解就可以)

    使用的是：DelayedWorkQueue（*）
    
    定时器框架：
        quartz、cron

    面试题：假如提供了一个闹钟服务，订阅这个服务的人特别多，10亿，怎么优化？
        开放题，暂没有标准答案
        
        1.主服务器把定时任务同步到边缘的机器上
        2.一台机器的优化：使用队列
    
```
Executors类
 public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
  
ScheduledThreadPoolExecutor类
public ScheduledThreadPoolExecutor(int corePoolSize) {
        super(corePoolSize, Integer.MAX_VALUE,
              DEFAULT_KEEPALIVE_MILLIS, MILLISECONDS,
              new DelayedWorkQueue());
    }  
    
super-->ThreadPoolExecutor类    
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue) {
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
         Executors.defaultThreadFactory(), defaultHandler);
}    
```

```
代码没看懂
public class T09_ScheduledThreadPool {
    public static void main(String[] args) {
        ScheduledExecutorService service = Executors.newScheduledThreadPool(4);
        service.scheduleAtFixedRate(()->{
            try {
                TimeUnit.MILLISECONDS.sleep(new Random().nextInt(500));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName());
        },0,500, TimeUnit.MILLISECONDS);
    }
}

```

### ==作业：读阿里手册，并发处理相关的规范==

###### 自定义拒绝策略

    代码T14
    
###### WorkStealingPool

    ThreadPoolExecutor是线程池有一个队列，执行任务的时候去队列中拿任务
    
    现在呢：线程池下面有多个线程，每一个线程都有自己单独的队列
    当自己的线程队列为空，会到别的workSteallingPool队列的末端去拿任务
    
    好处：当某个线程占用很长时间，并且还是特别大的任务，使用WorkStealingPool更加灵活，可以减轻该线程的压力
    
    问题：这个线程的队列还有锁吗？
        push、pop线程的时候不加锁
        被别的线程poll的时候，要加锁
        
    ForkJoinFool
    问题：既然有ForkjoinFool，为什么还要有WorkStealingPool？
        没那么复杂，只是提供了一个接口，参数已经定义，方便使用
    
![802](39A5D010A25546A59A9E2608607B3724)

```
源码
public static ExecutorService newWorkStealingPool() {
    return new ForkJoinPool
        (Runtime.getRuntime().availableProcessors(),
         ForkJoinPool.defaultForkJoinWorkerThreadFactory,
         null, true);
}
```


###### ForkJoinFool

    -分解汇总的任务
    -用很少的线程可以执行很多的任务（子任务）TPE做不到先执行子任务
    -CPU密集型
    
    Fork 任务分叉
    Join 任务汇总
    
    类似：mapreduce  大规模数据集的并行运算 分布可靠  一种编程模型
    
    ForkJoinTask比较原始
    一般使用：
        RecursiveAction  递归   不带返回值
        RecusiveTask 递归  有返回值
    
    精灵线程？？

![803](643B281504534FC9B50946802D5F89F9) 


```
笔试题：对100万的随机数进行求和，使用线程池

代码
```

###### ParallelStreamAPI
    
    底层使用ForkJoinPool
    并行流式API
    
    流式处理---lambda表达式

```
代码
```


### 总结：
    
    Executor
    
    ExecutorService
        abstractExecutor
    
    Callable
    Future
    FutureTask
    CompletableFuture(很牛逼的类，加分项)
    
    线程池：
    1.ThreadPoolExecutor
        7个参数：
            1.corePoolSize
            2.maximumPoolSize
            3.keepAliveTime
            4.TimeUnit
            5.BlockingQueue
            6.ThreadFactory
            7.RejectedExecutionHandler
                abort
                Discard
                discardOldest
                CallersRuns
        
        基于TPE的线程池    
            SingleThreadPool
            CachedThreadPool
            FixedThreadPool
            ScheduledThreadPool
    
        ThreadPoolExecutor源码
    
    2.ForkJoinPool
        WorkStealingPool
        
        ForkJoinPool源码比较复杂，也不考，连老师有课程讲
        
    ThreadPoolExecutor和ForkJoinPool的区别
        ThreadPoolExecutor共用一个任务队列
        ForkJoinPool，每个线程有自己的任务队列
        
        
###### 问题：
> 我用的自定义的线程池，运行的某个线程在执行一个三方包的native方法的时候会阻塞住，但是线程状态却还是Runnable，为啥？

    应该是native阻塞了
    
    因为如果调用了wait，那么状态不应该是Runnable
    
    怎么释放：
        native执行完自然释放
        加一个时间，执行一段时间后，native中的线程释放