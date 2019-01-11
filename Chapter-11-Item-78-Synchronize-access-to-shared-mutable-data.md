## Chapter 11. Concurrency（并发）

### Item 78: Synchronize access to shared mutable data（同步访问公共的可变数据）

The synchronized keyword ensures that only a single thread can execute a method or block at one time. Many programmers think of synchronization solely as a means of mutual exclusion, to prevent an object from being seen in an inconsistent state by one thread while it’s being modified by another. In this view, an object is created in a consistent state (Item 17) and locked by the methods that access it. These methods observe the state and optionally cause a state transition, transforming the object from one consistent state to another. Proper use of synchronization guarantees that no method will ever observe the object in an inconsistent state.

synchronized关键字确保一次只有一个线程可以执行一个方法或块。许多程序员认为同步仅仅是一种互斥的方法，以防止一个对象在被另一个线程修改时被一个线程看到处于不一致状态。在这个观点中，对象以一致的状态(Item 17)创建，并由访问它的方法锁定。这些方法观察状态并选择性地引起状态转换，将对象从一个一致的状态转换为另一个一致的状态。正确地使用同步可以确保任何方法都不会观察到处于不一致状态的对象。

This view is correct, but it’s only half the story. Without synchronization, one thread’s changes might not be visible to other threads. Not only does synchronization prevent threads from observing an object in an inconsistent state, but it ensures that each thread entering a synchronized method or block sees the effects of all previous modifications that were guarded by the same lock.

这种观点是正确的，但这只是故事的一半。如果没有同步，一个线程的更改可能对其他线程不可见。同步不仅阻止线程观察处于不一致状态的对象，而且确保每个进入同步方法或块的线程都能看到由相同锁保护的所有以前修改的效果。

The language specification guarantees that reading or writing a variable is atomic unless the variable is of type long or double [JLS, 17.4, 17.7]. In other words, reading a variable other than a long or double is guaranteed to return a value that was stored into that variable by some thread, even if multiple threads modify the variable concurrently and without synchronization.

Java语言规范保证读写变量是原子的，除非该变量类型是long或double [JLS, 17.4, 17.7]。换句话说，读取长或双精度变量以外的变量可以保证返回某个线程存储在该变量中的值，即使多个线程并发地修改该变量且没有同步。

You may hear it said that to improve performance, you should dispense with synchronization when reading or writing atomic data. This advice is dangerously wrong. While the language specification guarantees that a thread will not see an arbitrary value when reading a field, it does not guarantee that a value written by one thread will be visible to another. **Synchronization is required for reliable communication between threads as well as for mutual exclusion.** This is due to a part of the language specification known as the memory model, which specifies when and how changes made by one thread become visible to others [JLS, 17.4; Goetz06, 16].

您可能听说过，为了提高性能，在读写原子数据时应该避免同步。这种建议是危险的错误。虽然语言规范保证线程在读取字段时不会看到任意值，但它不能保证一个线程编写的值对另一个线程是可见的。**同步是线程间可靠通信以及互斥所必需的。** 这是由于被称为内存模型的语言规范的一部分，它指定了一个线程所做的更改何时以及如何对其他线程可见[JLS, 17.4;Goetz06,16)。

The consequences of failing to synchronize access to shared mutable data can be dire even if the data is atomically readable and writable. Consider the task of stopping one thread from another. The libraries provide the Thread.stop method, but this method was deprecated long ago because it is inherently unsafe —its use can result in data corruption. **Do not use Thread.stop.** A recommended way to stop one thread from another is to have the first thread poll a boolean field that is initially false but can be set to true by the second thread to indicate that the first thread is to stop itself. Because reading and writing a boolean field is atomic, some programmers dispense with synchronization when accessing the field:

即使数据是原子可读和可写的，对共享可变数据的同步访问失败的后果也是可怕的。考虑从一个线程停止另一个线程的任务。Java库提供了Thread.stop方法，但这种方法在很久以前就被弃用了，因为它本质上是不安全的——它的使用会导致数据损坏。**不要使用Thread.stop。** 一种建议的从一个线程停止另一个线程的方法是让第一个线程轮询一个布尔字段，该字段最初为false，但可以由第二个线程将其设置为true，以指示第一个线程停止自己。因为读写布尔字段是原子的，一些程序员在访问该字段时不需要同步:

```
// Broken! - How long would you expect this program to run?
public class StopThread {
    private static boolean stopRequested;
    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
        int i = 0;
        while (!stopRequested)
            i++;
        });
    backgroundThread.start();
    TimeUnit.SECONDS.sleep(1);
    stopRequested = true;
    }
}
```

You might expect this program to run for about a second, after which the main thread sets stopRequested to true, causing the background thread’s loop to terminate. On my machine, however, the program never terminates: the background thread loops forever!

您可能觉得这个程序运行大约一秒钟，之后主线程将stoprequest设置为true，导致后台线程的循环终止。然而，在我的机器上，程序永远不会结束:后台线程永远循环!

The problem is that in the absence of synchronization, there is no guarantee as to when, if ever, the background thread will see the change in the value of stopRequested made by the main thread. In the absence of synchronization, it’s quite acceptable for the virtual machine to transform this code:

问题是，在没有同步的情况下，不能保证后台线程什么时候(如果有的话)会看到主线程stoprequest值的变化。在缺乏同步的情况下，虚拟机可以很好地转换以下代码:

```
while (!stopRequested)
i++;
into this code:
if (!stopRequested)
while (true)
i++;
```

This optimization is known as hoisting, and it is precisely what the OpenJDK Server VM does. The result is a liveness failure: the program fails to make progress. One way to fix the problem is to synchronize access to the stopRequested field. This program terminates in about one second, as expected:

这种优化称为提升，这正是OpenJDK服务器VM所做的。其结果是活性失败：程序无法取得进展。解决此问题的一种方法是同步对stoprequest字段的访问。本程序结束约一秒，如预期:

```
// Properly synchronized cooperative thread termination
public class StopThread {
    private static boolean stopRequested;
    
    private static synchronized void requestStop() {
        stopRequested = true;
    }
    
    private static synchronized boolean stopRequested() {
        return stopRequested;
    }
    
    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested())
            i++;
        });
        backgroundThread.start();
        TimeUnit.SECONDS.sleep(1);
        requestStop();
    }
}
```

Note that both the write method (requestStop) and the read method (stop-Requested) are synchronized. It is not sufficient to synchronize only the write method! **Synchronization is not guaranteed to work unless both read and write operations are synchronized.** Occasionally a program that synchronizes only writes (or reads) may appear to work on some machines, but in this case, appearances are deceiving.

注意，写方法(requestStop)和读方法(stop-Requested)都是同步的。仅同步写方法是不够的！**除非读写操作同步，否则不能保证同步工作。** 有时，一个只同步写(或读)的程序可能在某些机器上工作，但是在这种情况下，表象是骗人的。

The actions of the synchronized methods in StopThread would be atomic even without synchronization. In other words, the synchronization on these methods is used solely for its communication effects, not for mutual exclusion. While the cost of synchronizing on each iteration of the loop is small, there is a correct alternative that is less verbose and whose performance is likely to be better. The locking in the second version of StopThread can be omitted if stopRequested is declared volatile. While the volatile modifier performs no mutual exclusion, it guarantees that any thread that reads the field will see the most recently written value:

即使没有同步，StopThread中同步方法的操作也是原子的。换句话说，这些方法上的同步仅用于其通信效果，而不是用于互斥。虽然在循环的每个迭代上同步的成本很小，但是有一种正确的替代方案，它更简洁，性能可能更好。如果stoprequest声明为volatile，则可以省略StopThread的第二个版本中的锁。虽然volatile修饰符不执行互斥，但它保证任何读取该字段的线程都会看到最近写入的值:

```
// Cooperative thread termination with a volatile field
public class StopThread {
    private static volatile boolean stopRequested;
    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
        int i = 0;
        while (!stopRequested)
            i++;
    });
    backgroundThread.start();
    TimeUnit.SECONDS.sleep(1);
    stopRequested = true;
    }
}
```

You do have to be careful when using volatile. Consider the following method, which is supposed to generate serial numbers:

使用volatile时一定要小心。下面的方法，被设计希望生成序列号:

```
// Broken - requires synchronization!
private static volatile int nextSerialNumber = 0;
public static int generateSerialNumber() {
    return nextSerialNumber++;
}
```

The intent of the method is to guarantee that every invocation returns a unique value (so long as there are no more than 232 invocations). The method’s state consists of a single atomically accessible field, nextSerialNumber, and all possible values of this field are legal. Therefore, no synchronization is necessary to protect its invariants. Still, the method won’t work properly without synchronization.

该方法的目的是确保每次调用都返回唯一的值(只要不超过232个调用)。方法的状态由一个原子可访问的字段nextSerialNumber组成，该字段的所有可能值都是合法的。因此，不需要同步来保护它的不变量。但是，如果没有同步，这个方法就不能正常工作。

The problem is that the increment operator (++) is not atomic. It performs two operations on the nextSerialNumber field: first it reads the value, and then it writes back a new value, equal to the old value plus one. If a second thread reads the field between the time a thread reads the old value and writes back a new one, the second thread will see the same value as the first and return the same serial number. This is a safety failure: the program computes the wrong results.

问题是，增量运算符(++)不是原子的。它对nextSerialNumber字段执行两个操作：首先读取值，然后写回一个新值，等于旧值加1。如果第二个线程在读取旧值和写回新值之间读取字段，那么第二个线程将看到与第一个线程相同的值，并返回相同的序列号。这是一个安全故障：程序计算错误的结果。

One way to fix generateSerialNumber is to add the synchronized modifier to its declaration. This ensures that multiple invocations won’t be interleaved and that each invocation of the method will see the effects of all previous invocations. Once you’ve done that, you can and should remove the volatile modifier from nextSerialNumber. To bulletproof the method, use long instead of int, or throw an exception if nextSerialNumber is about to wrap.

修复generateSerialNumber的一种方法是向其声明中添加同步修饰符。这确保了多个调用不会交错，并且该方法的每次调用都将看到以前所有调用的效果。这样做之后，您可以并且应该从nextSerialNumber中删除volatile修饰符。要使该方法不受影响，可以使用long而不是int，或者在nextSerialNumber将要换行时抛出异常。

Better still, follow the advice in Item 59 and use the class AtomicLong, which is part of java.util.concurrent.atomic. This package provides primitives for lock-free, thread-safe programming on single variables. While volatile provides only the communication effects of synchronization, this package also provides atomicity. This is exactly what we want for generateSerialNumber, and it is likely to outperform the synchronized version:

更好的是，遵循第59项中的建议并使用AtomicLong类，它是java.util.concurrent.atomic的一部分。这个包为在单个变量上进行无锁、线程安全的编程提供了原语。volatile只提供同步的通信效果，而这个包还提供原子性。这正是我们想要的generateSerialNumber，它可能会比同步版本更好:

```
// Lock-free synchronization with java.util.concurrent.atomic
private static final AtomicLong nextSerialNum = new AtomicLong();
public static long generateSerialNumber() {
    return nextSerialNum.getAndIncrement();
}
```

The best way to avoid the problems discussed in this item is not to share mutable data. Either share immutable data (Item 17) or don’t share at all. In other words, **confine mutable data to a single thread.** If you adopt this policy, it is important to document it so that the policy is maintained as your program evolves. It is also important to have a deep understanding of the frameworks and libraries you’re using because they may introduce threads that you are unaware of.

避免本小节中讨论的问题的最佳方法是不要共享可变数据。要么共享不可变数据(第17项)，要么根本不共享。换句话说，**将可变数据限制在单个线程中。** 如果您采用此策略，那么记录它是很重要的，以便在您的程序发展过程中维护该策略。深入了解您正在使用的框架和库也很重要，因为它们可能会引入您不知道的线程。

It is acceptable for one thread to modify a data object for a while and then to share it with other threads, synchronizing only the act of sharing the object reference. Other threads can then read the object without further synchronization, so long as it isn’t modified again. Such objects are said to be effectively immutable [Goetz06, 3.5.4]. Transferring such an object reference from one thread to others is called safe publication [Goetz06, 3.5.3]. There are many ways to safely publish an object reference: you can store it in a static field as part of class initialization; you can store it in a volatile field, a final field, or a field that is accessed with normal locking; or you can put it into a concurrent collection (Item 81).

一个线程可以修改一个数据对象一段时间，然后与其他线程共享它，只同步共享对象引用的行为。其他线程可以在不进行进一步同步的情况下读取该对象，只要它不再被修改。这些对象被认为是有效不可变的[Goetz06, 3.5.4]。将这样的对象引用从一个线程传输到另一个线程称为安全发布[Goetz06, 3.5.3]。有许多方法可以安全地发布对象引用:可以将其存储在静态字段中，作为类初始化的一部分;您可以将其存储在易失性字段、最终字段或使用常规锁定访问的字段中;或者您可以将其放入一个并发集合中(第81项)。

In summary, **when multiple threads share mutable data, each thread that reads or writes the data must perform synchronization.** In the absence of synchronization, there is no guarantee that one thread’s changes will be visible to another thread. The penalties for failing to synchronize shared mutable data are liveness and safety failures. These failures are among the most difficult to debug. They can be intermittent and timing-dependent, and program behavior can vary radically from one VM to another. If you need only inter-thread communication, and not mutual exclusion, the volatile modifier is an acceptable form of synchronization, but it can be tricky to use correctly.

总之，当多个线程共享可变数据时，每个读写数据的线程必须执行同步。在没有同步的情况下，不能保证一个线程的更改对另一个线程是可见的。同步共享可变数据失败的代价是活性和安全性失败。这些故障是最难调试的故障之一。它们可能是间歇性的，并且依赖于时间，而且程序行为可能在不同VM之间发生根本的变化。如果您只需要线程间通信，而不需要互斥，那么volatile修饰符是一种可接受的同步形式，但是正确使用它可能比较困难。
