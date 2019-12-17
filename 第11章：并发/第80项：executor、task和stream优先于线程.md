## executor、task 和 stream 优先于线程

&emsp;&emsp;本书的第一版包含简单的*工作队列（work queue）*的代码\[Bloch01, Item 49\]。这个类允许客户端将后台线程的异步处理工作排入队列。当不再需要这个工作队列时，客户端可以调用一个方法，让后台线程完成了已经在队列中的所有工作之后，优雅地终止自己。这个实现几乎就像个玩具，但即使如此，它还是需要一整页微妙、精致的代码，一不小心，就容易出现安全问题或者导致活性失败（liveness failure）。幸运的是，你不再需要编写这样的代码了。

&emsp;&emsp;到本书第二版出版时，java.util.concurrent 已添加到 Java 中。该软件包包含一个 Executor Framework，它是一个灵活的基于接口的任务执行工具。创建一个比本书第一版更好的工作队列只需要一行代码：

```java
ExecutorService exec = Executors.newSingleThreadExecutor();
```

&emsp;&emsp;以下是如何提交并执行 runnable:

```java
exec.execute(runnable);
```

&emsp;&emsp;下面是告诉 executor 如何优雅地终止（如果你没有这样做，很可能你的 VM 不会退出）：

```java
exec.shutdown();
```

&emsp;&emsp;你可以利用 executor service 完成更多的事情。例如，可以等待一项指定的任务【执行】完成（就项第 79 项，原书 319 页中的使用 get 方法一样），你可以等待一个任务集合中的任何任务或者所有任务完成（利用 invokeAny 或者 invokeAll 方法），你可以等待 executor service 终止（使用 awaitTermination 方法），你可以在完成任务时逐个检索任务结果（使用一个 ExecutorCompletionService），您可以安排任务在特定时间运行或定期运行（使用一个 ScheduledThreadPoolExecutor），等等。

&emsp;&emsp;如果想让不止一个线程来处理来自这个队列的请求，只要调用一个不同的静态工厂，这个工厂创建了一种不同的 executor service，称作*线程池（thread pool）*。你可以用固定或者可变数目的线程创建一个线程池。java.util.Executors 类包含了静态工厂，能为你提供所需的大多数 executor。然而，如果你想来点特别的，可以直接使用 ThreadPoolExecutor 类。这个类几乎允许你控制线程池每个功能的操作【阿里巴巴规范也是推荐使用这个类】。

&emsp;&emsp;为特定应用程序选择执行的程序服务可能很棘手。对于小的程序，或者轻载的服务器，使用 Executors.newCachedThreadPool 通常是个不错的选择，因为它不需要配置，并且一般情况下能够正确地完成工作。但是对于大负载的服务器来说，缓存的线程池就不是很好的选择了！在缓存的线程池中，被提交的任务没有排成队列，而是直接交给线程执行。如果没有线程可用，就创建一个新的线程。如果服务器负载得太重，导致了它所有的 CPU 都完全被占用了，当有更多的任务时，就会创建更多的线程，这样只会使情况变得更糟。因此，在大负载的产品服务器中，最好使用 Executors.newFixedThreadPool，它为你提供了一个包含固定线程数目的线程池，或者为了最大限度地控制它，就直接使用 ThreadPoolExecutor 类。

&emsp;&emsp;您不仅应该避免编写自己的工作队列，而且通常情况下也应该避免直接使用线程。当您直接使用线程时，线程既可以作为工作单元，也可以作为执行它的机制。在 executor Framework 中，工作单元和执行机制是分开的。抽象的关键是工作单元，也就是*任务（task）*。有两种任务：Runnable 及【功能】相近的 Callable（除了它会返回一个值并且可以抛出任意异常之外，它类似于 Runnable）。执行任务的通用机制是*executor service*。如果你从任务的角度来看问题，并让一个 Executor service 替你执行任务，您可以灵活地选择适当的执行策略以满足您的需求，并在需求发生变化时更改策略。从本质上讲，Executor Framework 所做的工作是执行，犹如 Collections Framework 所做的工作是聚集（aggregation）一样。

&emsp;&emsp;在 Java 7 中，Executor Framework 被扩展为支持[fork-join](https://www.ibm.com/developerworks/cn/java/j-lo-forkjoin/index.html)任务，这些任务由称为 fork-join 池的特殊 executor service 运行。由 ForkJoinTask 实例表示的 fork-join 任务可以拆分为较小的子任务，而包含 ForkJoinPool 的线程不仅处理这些任务，而且还彼此“窃取（steal）”任务以确保所有线程都保持忙碌，从而导致更高的 CPU 利用率，更高的吞吐量和更低的延迟。编写和调优 fork-join 任务很棘手。Parallel 流（第 48 项）写在 fork join 连接池之上，允许你轻松利用其性能优势，假设它们适合你手头上的任务。

&emsp;&emsp;对 Executor Framework 的完整处理超出了本书的范围，但感兴趣的读者可以参考《Java Concurrency in Practice》\[Goetz06\]。

> - [第 79 项：避免过度同步](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第11章：并发/第79项：避免过度同步.md)
> - [第 81 项：并发工具优先于 wait 和 notify](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第11章：并发/第81项：并发工具优先于wait和notify.md)
