## 并发工具优先于 wait 和 notify

&emsp;&emsp;本书的第一版专门用一项来【说明如何】正确使用 wait 和 notify\[Bloch01, 第 50 项\]。它的建议仍然有效，并在本项末尾进行了总结，但这个建议远不如以前那么重要。这是因为几乎没有理由再使用 wait 和 notify 了。自从 Java 5 以来，该平台提供了更高级别的并发工具，它们可以做一些你之前必须在 wait 和 notify 上手写代码来完成的各项工作。**鉴于正确地使用 wait 和 notify 比较困难，就应该使用更高级的并发工具来代替。**

&emsp;&emsp;java.util.concurrent 中更高级的工具分成三类：Executor Framework、并发集合（concurrent collections）、以及同步器（synchronizer），Executor Framework 在第 80 项已经简单介绍过了。并发集合和同步器将在本项中进行简单的阐述。

&emsp;&emsp;并发集合为标准的集合接口提供了高性能的并发实现，如 List、Queue 和 Map。为了提供高并发性，这些实现在内部自己内部同步（第 79 项）。因此，**并发集合中不可能排除并发活动；将它锁定只会使程序的速度变慢。**

&emsp;&emsp;因为您不能排除并发集合上的并发活动，所以您也不能以原子组合的方式对它们的方法进行调用（Because you can’t exclude concurrent activity on concurrent collections, you can’t atomically compose method invocations on them either）。因此有些集合接口已经通过*依赖状态的修改操作（state-dependent modify operations）*进行了扩展，它将几个基本操作合并到了单个原子操作中。事实证明，这些操作对并发集合非常有用，它们使用默认方法（第 21 项）添加到 Java 8 中相应的集合接口中。

&emsp;&emsp;例如，Map 的 putIfAbsent（key，value）方法插入键的映射（如果不存在）并返回与键关联的先前值【也就是被替换的值】，如果没有则返回 null。这样可以轻松实现线程安全的规范化映射。此方法模拟 String.intern 的行为：

```java
// Concurrent canonicalizing map atop ConcurrentMap - not optimal
private static final ConcurrentMap<String, String> map = new ConcurrentHashMap<>();
public static String intern(String s) {
    String previousValue = map.putIfAbsent(s, s);
    return previousValue == null ? s : previousValue;
}
```

&emsp;&emsp;事实上，你可以做得更好。ConcurrentHashMap 针对检索操作进行了优化，例如 get。因此，如果 get 表明有必要，最初只需调用 get，再调用 putIfAbsent：

```java
// Concurrent canonicalizing map atop ConcurrentMap - faster!
public static String intern(String s) {
    String result = map.get(s);
    if (result == null) {
        result = map.putIfAbsent(s, s);
        if (result == null)
            result = s;
    }
    return result;
}
```

&emsp;&emsp;ConcurrentHashMap 除了提供卓越的并发性之外，速度也非常快。在我的机器上，上面这个优化过的 intern 方法比 String.intern 速度快了超过六倍（但请记住，String.intern 必须采用一些策略来防止在长期存在的应用程序中泄漏内存）。并发集合使同步集合在很大程度上已经过时。例如，**优先使用 ConcurrentHashMap ，而不是 Collections.synchronizedMap** 。简单地用并发映射替换同步映射可以显著提高并发应用程序的性能。

&emsp;&emsp;有些集合接口已经通过*阻塞操作（blocking operations）*进行了扩展，它们会一直等待（或者*阻塞（block）*）直到可以成功执行为止。例如，BlockingQueue 扩展了 Queue 接口，并添加了包括 take 在内的几个方法，它们从队列中删除并返回头元素，如果队列为空，就等待。这样就允许将阻塞队列用于*工作队列（work queue）*，也被称为*生产者-消费者队列（producer-consumer queues）*，一个或者多个*消费者线程（consumer thread）*在工作队列中添加工作项目，一个或者多个*消费者线程（producer thread）*则从工作队列中将可用的工作项目取出队列并对它进行处理。正如你所期望的那样，大多数 ExecutorService 实现（包括 ThreadPoolExecutor）都使用 BlockingQueue（第 80 项）。

&emsp;&emsp;*同步器（Synchronizers）*是一些使线程能够等待另一个线程的对象，允许它们协调动作。最常用的同步器是 CountDownLatch 和 Semaphore。 不太常用的是 CyclicBarrier 和 Exchanger。最强大的同步器是 Phaser。

&emsp;&emsp;倒计时锁存器（Countdown latche）是一次性使用的屏障，允许一个或多个线程等待一个或多个其他线程执行某些操作。CountDownLatch 的唯一构造函数接受一个 int 参数，该参数是指在允许所有等待的线程继续之前必须在锁存器上调用 countDown 方法的次数。

&emsp;&emsp;在这个简单的基本类型上构建有用的东西是非常容易的。例如，假设您要构建一个简单的框架来定时执行一个操作的并发。这个框架包含仅仅只有一个方法，这个方法带有一个执行该动作的 executor，一个并发级别（表示要并发执行该动作的次数），以及表示该动作的 runnable。在计时器线程启动时钟【clock，翻译成计时会不会好一点】之前，所有工作线程都准备好自己要运行的操作。当最后一个工作线程准备好执行该动作时，timer 线程就“打响第一枪”，同时允许工作线程执行该动作。一旦最后一个工作线程执行完该动作，timer 线程就立即停止计时。直接在 wait 和 notify 之上实现这个逻辑至少来说会很混乱，而在 CountDownLatch 之上实现则相单简单：

```java
// Simple framework for timing concurrent execution
public static long time(Executor executor, int concurrency, Runnable action) throws InterruptedException {
    CountDownLatch ready = new CountDownLatch(concurrency);
    CountDownLatch start = new CountDownLatch(1);
    CountDownLatch done = new CountDownLatch(concurrency);
    for (int i = 0; i < concurrency; i++) {
        executor.execute(() -> {
            ready.countDown(); // Tell timer we're ready
            try {
                start.await(); // Wait till peers are ready
                action.run();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                done.countDown(); // Tell timer we're done
            }
        });
    }
    ready.await(); // Wait for all workers to be ready
    long startNanos = System.nanoTime();
    start.countDown(); // And they're off!
    done.await(); // Wait for all workers to finish
    return System.nanoTime() - startNanos;
}
```

&emsp;&emsp;注意这个方法使用了三个倒计数锁存器。第一个是 ready，工作线程用它来告诉 timer 线程它们已经准备好了。然后工作线程在第二个锁存器上等待，也就是 start。当最后一个工作线程调用 read.countDown 时，timer 线程记录下起始时间，并调用 start.countDown，并允许所有的工作线程继续进行。然后 timer 线程在第三个锁存器（即 done）上等待，直到最后一个工作线程运行完该动作，并调用 done.countDown。一旦调用这个，timer 线程就会苏醒过来，并记录下结束的时间。

&emsp;&emsp;还有一些细节值得注意。传递给 time 方法的 executor 必须允许创建至少与指定并发级别一样多的线程，否则这个测试就永远不会结束。这就是*线程饥饿死锁（thread starvation deadlock）*\[Goetz06, 8.1.1\]。如果线程捕捉到 InterruptedException，就会利用习惯用法 Thread.currentThread().interrupt()重新断言中断，并从它的 run 方法中返回。这样就允许 executor 在必要的时候处理终端，事实上也是如此。注意，这利用了 System.nanoTime()来给活动定时。**对于间歇性的定时，始终应该优先使用 System.nanoTime，而不是使用 System.currentTimeMillis** 。System.nanoTime 更加准确也更加精确，它不受系统的实时时钟的调整所影响。最后，请注意，此示例中的代码不会产生准确的计时，除非操作执行了大量工作，例如一秒钟或更长时间。准确的微基准测试是非常困难的，最好借助于 jmh\[JMH\]等专用框架来完成。

&emsp;&emsp;本项仅仅触及了并发工具的一些皮毛。例如，前面的例子中的那三个倒计数锁存器其实可以用一个 CyclicBarrier 或者 Phaser 实例来代替。这样产生的代码更加简洁，但是理解起来比较困难。

&emsp;&emsp;虽然你始终应该优先使用并发工具，而不是使用 wait 和 notify，但可能必须维护使用了 wait 和 notify 的遗留代码。wait 方法被用来使线程等待某些条件。它必须在同步区域内部被调用，这个同步区域将对象锁定在了调用 wait 方法的对象上。下面是使用 wait 方法的标准模式：

```java
// The standard idiom for using the wait method
synchronized (obj) {
    while (<condition does not hold>)
        obj.wait(); // (Releases lock, and reacquires on wakeup)
    ... // Perform action appropriate to condition
}
```

&emsp;&emsp;**始终应该使用 while 循环模式来调用 wait 方法；永远不要在循环之外调用 wait 方法** 。循环会在等待之前和之后测试条件【是否成立】。

&emsp;&emsp;在等待之前测试条件【是否成立】，当条件已经成立时就跳过等待，这对于确保*活性（liveness）*是必要的。如果条件已经成立，并且在线程等待之前，notify（或者 notifyAll）方法已经被调用，则无法保证该线程将会从等待中苏醒过来。

&emsp;&emsp;在等待之后测试条件【是否成立】，如果条件不成立的话就继续等待，这对于确保安全性（safety）是必要的。当条件不成立的时候，如果线程继续执行，则可能会破坏被锁保护的约束关系。当条件不成立的时候，下面有一些理由可以使一个线程苏醒过来：

- 另一个线程可能已经得到了锁，并且从一个线程调用 notify 那一刻起，到等待线程苏醒过来的这段时间中，得到锁的线程已经改变了受保护的状态。

- 条件并不成立的情况下，另一个线程可能意外地或者恶意地调用了 notify。在公有可访问的对象上等待，这些类实际上把自己暴露在了这种危险的境地中。公有可访问对象的同步方法中包含的 wait 都会出现这样的问题。

- 通知线程（notifying thread)在唤醒等待线程时可能会过度“大方”。例如，即使只有某一些等待线程的条件已经被满足，但是通知线程可能仍然调用 notifyAll。

- 在没有通知的情况下，等待线程也可能(但很少)会苏醒过来。这被称为\*伪唤醒（spurious wakeup）\[POSIX, 11.4.3.6.1; Java9-api\]。

&emsp;&emsp;一个相关的话题是，为了唤醒正在等待的线程，你应该使用 notify 还是 notifyAll（回忆一下，notify 唤醒的是某个正在等待的线程，假设有这样的线程存在，而 notifyAll 唤醒的则是所有正在等待的线程）。一种常见的说法是，你应该总是使用 notyfiAll。这是合理而保守的建议。它总会产生正确的结果，因为它可以保证你将唤醒所有需要被唤醒的线程。你可能也会唤醒其他一些线程，但是这不会影响程序的正确性。这些线程醒来之后，会检查它们正在等待的条件，如果发现条件并不满足，就会继续等待。

&emsp;&emsp;从优化的角度来看，如果可能在处于等待的所有线程都在等待相同的条件并且一次只有一个线程可以从条件变为 true 的时候受益【被唤醒】，则可以选择调用 notify 而不是 notifyAll。

&emsp;&emsp;即使前面的那些条件都满足了，也许还是有理由使用 notifyAll 而不是 notify。就好像把 wait 调用放在一个循环中，以避免在公有可访问对象上的意外或者恶意的通知一样，与此类似，使用 notifyAll 代替 notify 可以避免来自不相关线程的意外或者恶意的等待。否则，这样的等待会“吞掉”一个关键的通知，使真正的接受线程无限地等待下去。

&emsp;&emsp;简而言之，直接使用 wait 和 notify 就像用“并发汇编语言”进行编程一样，而 java.util.concurrent 则提供了更高级的语言。**没有理由在新代码中使用 wait 和 notify，即使有，也是极少的** 。如果你在维护使用了 wait 和 notify 的代码，务必确保始终是利用标准的模式从 while 循环内部调用 wait。一般情况下，你应该优先使用 notifyAll，而不是使用 notify。如果使用 notify，请一定要小心，以确保程序的活性（liveness）。

> - [第 80 项：executor、task 和 stream 优先于线程](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第11章：并发/第80项：executor、task和stream优先于线程.md)
> - [第 82 项：线程安全性的文档化](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第11章：并发/第82项：线程安全性的文档化.md)
