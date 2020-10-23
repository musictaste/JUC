#### CAS(无锁优化，自旋)

    compare and swap
    cas(V,Expected,NewValue)
        -if V=E
         V=new
         otherwise try again or fail
        
        -保证了原子性，CPU原语支持,指令中间不能被打断
        
    ABA问题
        加版本号
        AtomicStampedReference类（带版本号）
        
![image](https://raw.githubusercontent.com/musictaste/JUC/master/image/220.png)

##### 知识点1：AtomicXXX

    由于一些常见的操作总是需要加锁，java提供了一些常见操作的类
    这些类不是synchronized的重量级锁，而是采用CAS技术来实现的
    
    解决同样的问题的更高效的方法，使用AtomXXX类
==AtomXXX类本身方法都是原子性的，但不能保证多个方法连续调用是原子性的==
    

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

    
##### 知识点：CAS原理
    以 java.util.concurrent 中的 AtomicInteger 为例
    incrementAndGet方法，该方法的作用相当于 ++i 操作
    
    incrementAndGet采用了CAS操作，每次从内存中读取数据，然后将此数据和 +1 后的结果进行CAS操作，
    如果成功就返回结果，否则重试直到成功为止
    
```
/**
 * Atomically increments by one the current value.
 *
 * @return the updated value
 */
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```
    
**sun.misc.Unsafe**的**compareAndSwapInt**==利用JNI（Java Native Interface）来完成CPU指令的操作==
    
```
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```


下面从分析比较常用的CPU（intel x86）来解释CAS的实现原理。

下面是sun.misc.Unsafe类的compareAndSwapInt()方法的源代码：

```
public final native boolean compareAndSwapInt(
    Object var1, long var2, int var4, int var5);

```
compareAndSwapInt这个本地方法在JDK中依次调用的C++代码为

```
#define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                        __asm je L0      \
                        __asm _emit 0xF0 \
                        __asm L0:
 
 inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
   // alternative for InterlockedCompareExchange
  int mp = os::is_MP();
   __asm {
     mov edx, dest
     mov ecx, exchange_value
     mov eax, compare_value
     LOCK_IF_MP(mp)
     cmpxchg dword ptr [edx], ecx
   }
 }
```

如上面源代码所示，**程序会根据当前处理器的类型来决定是否为cmpxchg指令添加lock前缀**。

==如果程序是在多处理器上运行，就为cmpxchg指令加上lock前缀（lock cmpxchg）==。

反之，如果程序是在单处理器上运行，就省略lock前缀（==单处理器自身会维护单处理器内的顺序一致性，不需要lock前缀提供的内存屏障效果==）。

    _asm 汇编语言
        LOCK_IF_MP  
            一个cpu，直接执行cmpxchg指令
            多个cpu，执行lock cmpxchg指令
            
    cmpxchg汇编指令
    lock cmpxchg指令：
        
==cmpxchg指令，没有原子性，不能保证读和写之间不能被其他cpu改写==

==lock指令，保证当前cpu对当前值修改的时候，其他cpu不能进行修改，保证了原子性==

##### 知识点：ABA问题
    解决：当前的值加版本号
    或者JDK中加boolean类型:从Java1.5开始JDK的atomic包里提供了一个类AtomicStampedReference来解决ABA问题，该类带版本号信息
    
ABA问题的影响

    如果是基本数据类型，无所谓
    如果是引用类型，会有问题
        从引用地址来看，引用对象O没有改变，但是引用对象O中的属性A，已经发生了改变
    
        举例：我和女朋友分手了，复合之前呢，女朋友其实已经又交往了另外几个男朋友


##### 知识点：Unsafe （简单了解）
    JDK8 不能调用，只能JDK底层类才能调用
    JDK9以后已经关闭了
    查看JDK11源码，发现方法变成了弱引用

    allocateMemory   分配内存
    
    Unsafe类==c、C++的指针
    
```
public class HelloUnsafe {
    static class M {
        private M() {}
        int i =0;
    }

   public static void main(String[] args) throws InstantiationException {
        Unsafe unsafe = Unsafe.getUnsafe();
        M m = (M)unsafe.allocateInstance(M.class);
        m.i = 9;
        System.out.println(m.i);
    }
}
运行结果：
Exception in thread "main" java.lang.SecurityException: Unsafe
	at sun.misc.Unsafe.getUnsafe(Unsafe.java:90)
	at com.mashibing.juc.c_018_01_Unsafe.HelloUnsafe.main(HelloUnsafe.java:15)
```

    
## 总结
    CAS
        无锁优化，乐观锁，自旋
        
        compare and swap
        
        cas(V,Expected,NewValue)
        -if V=E
         V=new
         otherwise try again or fail
        
        -保证了原子性，CPU原语支持,指令中间不能被打断
        
        AtomicXXX类
        
        ABA问题
            version
            AtomicStampedReference类
            
            基本类型，影响不大
            引用类型，女朋友分手，复合之前，又有其他好几个男朋友
            
        java支持：Unsafe类
        
        原理：
            java.util.concurrent 中的 AtomicInteger
            sun.misc.Unsafe类的compareAndSwapInt()
            _asm汇编指令：
                LOCK_IF_MP  
                一个cpu，直接执行cmpxchg指令
                多个cpu，执行lock cmpxchg指令
        
![image](https://raw.githubusercontent.com/musictaste/JUC/master/image/220.png)            
