### 源码阅读原则

![image](https://raw.githubusercontent.com/musictaste/JUC/master/image/408.png)
![408](535DB2AF570E41449F15416819C2FC1D)

==作业：tank项目==

### AQS(AbstractQueuedSynchronizer)

    数据结构基础
    设计模式：spring、mybatis
        tank项目
        
    [Template Method设计模式](https://github.com/musictaste/DesignPatterns/blob/master/TemplateMethod%E6%A8%A1%E6%9D%BF%E6%96%B9%E6%B3%95.md)
    
**IDEA UML插件：PlantUML integration**  

    方法调用图
    
![410](24B1674A13AF4E47BE96C0A9F779962B)
   
    
    类图

![411](052AA34967F54A6A8FA5EE4AF5E3C4A9)    
    
    AQS(CLH)
        CLH:ASQ队列又叫CLH队列
        
        state(volatile int)
            共享的数据
    
        Node(双向链表 static final class)
            互相抢夺的线程
            -Thread
            -Node pre
            -Node Next
            head
            tail

==AQS的核心：使用CAS操作tail节点==
           
![409](A6B704A8141F410AA3246E39A5A64936)
```
AQS: 
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

ReentrantLock类：
/**
 * Performs non-fair tryLock.  tryAcquire is implemented in
 * subclasses, but both need nonfair try for trylock method.
 */
@ReservedStackAccess
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

AQS:
/**
 * Creates and enqueues node for current thread and given mode.
 *
 * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
 * @return the new node
 */
private Node addWaiter(Node mode) {
    Node node = new Node(mode);

    for (;;) {
        Node oldTail = tail;
        if (oldTail != null) {
            node.setPrevRelaxed(oldTail);
            if (compareAndSetTail(oldTail, node)) {
                oldTail.next = node;
                return node;
            }
        } else {
            initializeSyncQueue();
        }
    }
}

final void setPrevRelaxed(Node p) {
    PREV.set(this, p);
}

// VarHandle mechanics
private static final VarHandle NEXT;
private static final VarHandle PREV;
private static final VarHandle THREAD;
private static final VarHandle WAITSTATUS;


AQS类：
/**
 * Acquires in exclusive uninterruptible mode for thread already in
 * queue. Used by condition wait methods as well as acquire.
 *
 * @param node the node
 * @param arg the acquire argument
 * @return {@code true} if interrupted while waiting
 */
final boolean acquireQueued(final Node node, int arg) {
    boolean interrupted = false;
    try {
        for (;;) {
            final Node p = node.predecessor();//拿到前置节点
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node))
                interrupted |= parkAndCheckInterrupt();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        if (interrupted)
            selfInterrupt();
        throw t;
    }
}
```
    addWaiter方法剖析：
        为什么AQS效率高？
        为什么要加到链表的尾端要使用CAS？
            正常的话：锁定整个链表，往尾端加节点
            
            现在使用CAS，不锁定整个链表，判断oldTail和链表当前的tail是否一致，如果一致，说明没有别的线程操作
            
            这样的情况下，AQS的效率更高
            
    为什么是双向链表？
        因为要查看前一个节点的状态
        
#### VarHandle变量句柄（JDK1.9以后才有，面试用）
        Object o = new Object();
        在内存中，o指向了  Object的内存地址
        
        VarHandle就是这个o指向 Object对象内存的地址引用
        
        既然有o指向了Object对象，为什么还要用VarHandle？
            JDK9之前使用反射得到Varhandle的效果，但是效率要低
            1.普通属性也可以完成原子性操作   
            2.比反射快，直接操作二进制码
==**1.普通属性也可以完成原子性操作**==   
==**2.比反射快，直接操作二进制码**==  
    
        
            
```
graph LR
o-->Object
varhandle-->Object
```
     
```
public class T01_HelloVarHandle {

    int x = 8;

    private static VarHandle handle;

    static {
        try {
            handle = MethodHandles.lookup().findVarHandle(T01_HelloVarHandle.class, "x", int.class);
        } catch (NoSuchFieldException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        T01_HelloVarHandle t = new T01_HelloVarHandle();

        //plain read / write
        System.out.println((int)handle.get(t));
        handle.set(t,9);
        System.out.println(t.x);

        handle.compareAndSet(t, 9, 10);//原子性操作
        System.out.println(t.x);

        handle.getAndAdd(t, 10);//原子性操作
        System.out.println(t.x);

    }
}

运行结果：
8
9
10
20
```
## 面试题：说说AQS
    
    AQS效率很高
    底层：双向链表+volatile+CAS
        
**面试题：为什么说AQS的底层是volatile+CAS？**

    1.AQS中有state属性，是用volatile修饰的
        
    2.Node(双向链表)，head、next、pred、tail
    调用addWaiter()往队列里加节点的时候，使用了compareAndSetState方法，该方法就是CAS技术，也就是说：使用CAS来操作tail节点

## 面试题：为什么AQS效率高？

    1.AQS中使用了双向链表
    2.往链表的尾端添加节点时候，使用CAS，不需要锁定整个链表
    3.Node中使用VarHandle；
        VarHandle的特点：第一，普通属性也有原子性操作；
        第二：比反射快，直接操作二进制码

## 面试题：为什么要加到链表的尾端要使用CAS？

    1.正常情况下，我们会考虑锁定这个链表，往尾端加节点
        
    现在呢，我使用CAS，不需要多锁定整个链表，我只需要判断oldTail和链表当前的tail是否一致，如果一致，说明没有别的线程来操作
    这样的情况下，AQS的效率更高
    
==作业：countdownlatch的源码==      
    

### 总结：
    AQS源码
        -VarHandle
            -1.普通属性的原子操作
            -2.比反射快，直接操作二进制码

    
    