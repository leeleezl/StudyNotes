# synchronized 原理

## Monitor 原理

我们先来看一下什么是 Monitor ？

Monitor 被翻译成**监视器**或**管程**，每个 Java 对象都可以关联一个 Monitor 对象，如果使用 synchronized 给对象上锁，那么该对象头的Mark Word 就被设置指向 Monitor 对象的指针。 

Monitor 的结构如下：

![Monitor 结构](img\Monitor 结构.png)

- 刚开始 Monitor 的 Owner 为null
- 当 Thread-2 执行 sychronized(Obj) 时，Monitor 的 Owner 会设置为 Thread-2，Monitor 中只能有一个 Owner
- 在 Thread-2 上锁过程中，如果其他线程也来获取这个对象锁，就会进入 EntryList 阻塞
- 当 Thread-2 执行完同步代码块后，会唤醒 EntryList 中的线程来竞争锁，非公平竞争
- WaitSet 中的 Thread-0 和 Thread-1 是之前获得过锁。

## synchronized 实现原理

synchronized 加在**同步代码**块和加在**方法**上的原理是有一些不同的。

### 同步代码块上的 synchronized

```Java
public void syn() {
    Object object = new Object();
    synchronized (object) {
        System.out.println()
    }
}
```

通过`javap -v`反编译之后的字节码如下：

```Java
  public void synDemo();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=4, args_size=1
         0: new           #2                  
         3: dup
         4: invokespecial #1                 
         7: astore_1
         8: aload_1               
         9: dup
        10: astore_2
        11: monitorenter           // 将 lock对象 MarkWord 置为 Monitor 指针
        12: getstatic     #3      
        15: invokestatic  #4      
        18: invokevirtual #5      
        21: invokevirtual #6      
        24: aload_2
        25: monitorexit            // 将 lock对象 MarkWord 重置, 唤醒 EntryList
        26: goto          34
        29: astore_3
        30: aload_2
        31: monitorexit            // 将 lock对象 MarkWord 重置, 唤醒 EntryList
        32: aload_3
        33: athrow
        34: return
      Exception table:
         from    to  target type
            12    26    29   any
            29    32    29   any
      LineNumberTable:
        line 8: 0
        line 9: 8
        line 10: 12
        line 11: 24
        line 12: 34
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      35     0  this   LsychronizedDemo/Syn;
            8      27     1     o   Ljava/lang/Object;
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 29
          locals = [ class sychronizedDemo/Syn, class java/lang/Object, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
    RuntimeVisibleAnnotations:
      0: #23()
}

```

通过字节码文件我们可以看到，sychronized 加在同步代码块上是通过两个指令，`monitorenter` 和 `monitorexit` 实现的。

每个对象都有一个 Monitor ，当 Monitor 被占用时就表示对象处于锁定状态，`monitorenter`指令就是来获取 Monitor 的所有权，`monitorexit`指令就是释放 Monitor 的所有权。这两者的工作流程如下：
**monitorenter**：

1. 如果`monitor`的进入数为0，则线程进入到`monitor`，然后将进入数设置为`1`，该线程称为`monitor`的所有者。
2. 如果是线程已经拥有此`monitor`(即`monitor`进入数不为0)，然后该线程又重新进入`monitor`，则将`monitor`的进入数`+1`，这个即为**锁的重入**。
3. 如果其他线程已经占用了`monitor`，则该线程进入到**阻塞状态，直到`monitor`的进入数为0，该线程再去重新尝试获取`monitor`的所有权**。

**monitorexit**：执行该指令的线程必须是`monitor`的所有者，指令执行时，`monitor`进入数`-1`，如果`-1`后进入数为`0`，那么线程退出`monitor`，不再是这个`monitor`的所有者。这个时候其它阻塞的线程可以尝试获取`monitor`的所有权。

### 同步方法上的synchronized

```Java
synchronized public void synDemo1() {
    System.out.println(Thread.currentThread().getName());
}
```

通过`javap -v`反编译之后的字节码如下：

```Java
public synchronized void synDemo1();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #3               
         3: invokestatic  #4                 
         6: invokevirtual #5                 
         9: invokevirtual #6                  
        12: return
      LineNumberTable:
        line 16: 0
        line 17: 12
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      13     0  this   LsychronizedDemo/Syn;
    RuntimeVisibleAnnotations:
      0: #23()
}

```

从字节码文件中可以看到，同步方法上的 synchronized 并没有使用`monitorenter`和`monitorexit`指令，而是在常量池中比普通方法多了`ACC_SYNCHRONIZED`标识，JVM 就根据这个标识来实现方法的同步。

当调用方法的时候，调用指令会检查方法是否有`ACC_SYNCHRONIZED`标识，有的话**线程需要先获取`monitor`，获取成功才能继续执行方法，方法执行完毕之后，线程再释放`monitor`，同一个`monitor`同一时刻只能被一个线程拥有。**

### 两种同步方式的区别

`synchronized`同步代码块的时候通过加入字节码`monitorenter`和`monitorexit`指令来实现`monitor`的获取和释放，也就是需要**JVM通过字节码显式的去获取和释放monitor实现同步**

synchronized同步方法的时候，没有使用这两个指令，而是检查方法的`ACC_SYNCHRONIZED`标志是否被设置，如果设置了则线程需要先去获取monitor，执行完毕了线程再释放monitor，也就是不需要JVM去显式的实现。

**这两个同步方式实际都是通过获取monitor和释放monitor来实现同步的，而monitor的实现依赖于底层操作系统的`mutex`互斥原语，而操作系统实现线程之间的切换的时候需要从用户态转到内核态，这个转成过程开销比较大。**

# sychronized 锁优化

## 对象头

在`HotSpot`虚拟机中，`Java`对象在内存中储存的布局可以分为`3`块区域：**对象头**、**实例数据**、**对齐填充**。**synchronized使用的锁对象储存在对象头中**

对象头由以下三个部分组成：

- Mark Word：记录了对象和锁的有关信息，储存对象自身的运行时数据，如哈希码(HashCode)、`GC`分代年龄、锁标志位、线程持有的锁、偏向线程ID、偏向时间戳、对象分代年龄等。**注意这个Mark Word结构并不是固定的，它会随着锁状态标志的变化而变化，而且里面的数据也会随着锁状态标志的变化而变化，这样做的目的是为了节省空间**。
- 类型指针：指向对象的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。
- 数组长度：这个属性只有数组对象才有，储存着数组对象的长度。

## synchronized 优化

上文说的 synchronized 锁原理其实说的都是**重量级锁**的原理那么上文频繁提到`monitor`对象和对象又存在什么关系呢，或者说`monitor`对象储存在对象的哪个地方呢？其实是**在对象的对象头中，当锁的状态为重量级锁的时候，它的指针即指向`monitor`对象**。当锁的状态为其他状态的时候是没有用上Monitor对象的。

锁膨胀方向是：**无锁——>偏向锁——>轻量级锁——>重量级锁**，并且膨胀方向不可逆。

#### 偏向锁

Java 6 中引入了偏向锁来做进一步优化：只有第一次使用 CAS 将线程 ID 设置到对象的 Mark Word 头，**之后发现这个线程 ID 是自己的就表示没有竞争，不用重新 CAS**。以后只要不发生竞争，这个对象就归该线程所有

### 自旋与适应性自旋

**自旋**：当一个线程取请求某个锁的时候，这个锁正在被其他线程占用，这时此线程不会立刻进入阻塞，而是循环请求锁（自旋），**自旋的目的是因为很多情况下持有锁的线程很快会释放锁，线程可以尝试一直请求锁，所以线程没有必要放弃cpu时间片，因为线程被挂起再唤醒的开销是比较大的**。当然如果线程自旋一定时间还没有获得锁，仍然会被挂起。

**适应性自旋**：自适应自旋是对自旋的优化，自选的时间不再固定，而是**由前一次在同一个锁上的自旋时间和当前拥有锁的线程的状态决定的**。例如**线程如果自旋成功了，那么下次自旋的次数会增多**，因为`JVM`认为既然上次成功了，那么这次自旋也很有可能成功，那么它会允许自旋的次数更多。反之，如果**对于某个锁，自旋很少成功**，那么在以后获取这个锁的时候，自旋的次数会变少甚至忽略，避免浪费处理器资源。有了自适应性自旋，随着程序运行和性能监控信息的不断完善，`JVM`对程序锁的状况预测就会变得越来越准确，`JVM`也就变得越来越聪明。

### 锁粗化

锁粗化是虚拟机对另一种极端情况的优化处理，通过扩大锁的范围，避免反复加锁和释放锁。比如下面method1经过锁粗化优化之后就和method2执行效率一样了。

```Java
public void methord1() {
    for(int i=0; i<10000; i++) {
        synchronized(Test.class) {
            System.out.println("hello world");
        }
    }
}

public void methord2() {
    synchronized(test.class) {
        for(int i=0; i<10000; i++) {
            Systom.out.println("hello world");
        }
    }
}
```

### 锁消除

消除锁是虚拟机另外一种锁的优化，这种优化更彻底，在JIT编译时，对运行上下文进行扫描，去除不可能存在竞争的锁。比如下面代码的method3和method4的执行效率是一样的，因为object锁是私有变量，不存在所得竞争关系。

```Java
public void methord3() {
    Object obj = new Object();
    sychronized(obj) {
        System.out.println("hello world");
    }
}

public void methord4() {
    Object obj = new Object();
    System.out.println("hello world");
}
```

### 偏向锁

一句话总结它的作用：**减少同一线程获取锁的代价**。在大多数情况下，锁不存在多线程竞争，总是由同一线程多次获得，那么此时就是偏向锁。

**核心思想：**

如果一个线程获得了锁，那么锁就进入偏向模式，此时`Mark Word`的结构也就变为偏向锁结构，**当该线程再次请求锁时，无需再做任何同步操作，即获取锁的过程只需要检查**`Mark Word`**的锁标记位为偏向锁以及当前线程ID等于**`Mark Word`**的ThreadID即可**，这样就省去了大量有关锁申请的操作

![](img\偏向锁.png)

### 轻量级锁

应用场景：如果一个对象有多线程要加锁，但是加锁时间是错开始（也就是没有竞争），那么可以使用轻量级锁来优化。

假设有两个方法同步块，利用同一个对象加锁

- 创建锁记录（Lock Record）对象，每个线程的栈帧都会包含锁记录的结构，内部可以存储锁定对象的Mark Word
- 让锁记录中的 Object Referrence 指向锁对象，并尝试 CAS 替换 Object 的 Mark Word，将 Mark Word 的值存入锁记录
- 如果 CAS 成功，那么对象头中存储了`锁记录地址和状态00`，表示由该线程给对象加锁。  
- 如果 CAS 失败，有两种情况
  - 如果其他线程已经持有了 Object 的轻量级锁，表明有竞争，进入锁膨胀
  - 如果是自己执行了 synchronized 锁重入，那么再添加一条 Lock Record 作为重入的计数
- 退出 synchronized 代码块的时候，如果锁记录为 null 则表示有锁重入，这时重置锁记录，表示重入计数减一
- 当退出 synchronized 代码块的时候锁记录不为 null 这时使用 CAS 将 Mark Word 的值恢复给对象头
  - 成功，则解锁成功
  - 失败，轻量级锁进行了锁膨胀升级成了重量级锁，进入重量级锁的解锁流程

![](img\轻量级锁.png)