## 谨慎使用 Stream 并行

&emsp;&emsp;在主流语言中，在提供便于并发编程任务功能方面，Java 始终处于最前沿【的位置】（Among mainstream languages, Java has always been at the forefront of providing facilities to ease the task of concurrent programming）。当 Java 于 1996 年发布时，它内置了对线程的支持，具有同步和等待/通知【的功能】（When Java was released in 1996, it had built-in support for threads, with synchronization and wait/notify）。Java 5 引入了 java.util.concurrent 库，包含并发集合和执行器框架。 Java 7 引入了 fork-join 包，这是一个用于并行分解（parallel decomposition）的高性能框架。Java 8 引入了流，可以通过对并行方法的单个调用来并行化。用 Java 编写并发程序变得越来越容易，但编写正确快速的并发程序就跟以前一样困难。安全性和活性违规（liveness violations ）是并发编程中的事实，并行流管道也不例外。

&emsp;&emsp;考虑第 45 项中的这个程序：

```java
// Stream-based program to generate the first 20 Mersenne primes
public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
        .filter(mersenne -> mersenne.isProbablePrime(50))
        .limit(20)
        .forEach(System.out::println);
}
static Stream<BigInteger> primes() {
    return Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```

&emsp;&emsp;在我的机器上，该程序立即开始打印质数，并需要 12.5 秒才能完成运行。假设我试图通过向流管道添加对 parallel()的调用来加速它。你认为它的表现会怎样？它【的运行速度】会加快几个百分点吗？还是慢几个百分点？可悲的是，发生的事情是它没有打印任何东西，但是 CPU 使用率飙升至 90％并且无限期地停留在那里（_活性失败(liveness failure)_）。该程序最终可能会终止，但我不愿意去发现【等待这个结果】; 半小时后我强行停止【了程序】。

&emsp;&emsp;这里发生了什么？简而言之，流的库不知道如何并行化此管道并且试探启动（heuristics）失败。即使在最好的情况下，**如果源来自 Stream.iterate，或者使用中间操作限制，并行化管道也不太可能提高其性能(parallelizing a pipeline is unlikely to increase its performance if the source is from Stream.iterate, or the intermediate operation limit is used.)**。这条管道必须应对这两个问题。更糟糕的是，默认的并行化策略是通过假设处理一些额外元素并丢弃任何不需要的结果不会带来任何损失的前提下来处理限制的不可预测性。在这种情况下，找到每个梅森质数需要大约两倍的时间才能找到前一个。因此，计算单个额外元素的成本大致等于计算所有先前元素组合的成本，并且这种看起来没什么损失的管道会使自动并行化算法瘫痪。这个故事的寓意很简单：**不要不加选择的地使用并行化流**。导致的性能后果可能是灾难性的。

&emsp;&emsp;**并行性的性能增益最好是在 ArrayList，HashMap，HashSet 和 ConcurrentHashMap 实例上；int 数组；和 long 数组(performance gains from parallelism are best on streams over ArrayList, HashMap, HashSet, and ConcurrentHashMap instances; arrays; int ranges; and long ranges)**，将这作为一项规则。这些数据结构的共同之处在于它们都可以准确且分成任何所需大小的子范围的代价是很小的，这使得在并行线程之间划分工作变得容易。流库用于执行此任务的抽象是 spliterator，它由 Stream 和 Iterable 上的 spliterator 方法返回。

&emsp;&emsp;所有这些数据结构的另一个重要因素是它们在顺序处理时提供了非常好的*位置引用(locality of reference)*：元素的顺序和【元素的】引用一起存储在存储器中。这些引用所引用的对象在存储器中可能彼此不接近，这减少了位置引用(The objects referred to by those references may not be close to one another in memory, which reduces locality-of-reference.)。对于并行化操作而言，位置引用非常重要：如果没有位置引用，线程大部分时间会处在空闲状态，等待数据从内存传输到处理器的缓存。具有最佳位置引用的数据结构是原始数组，因为数据本身连续存储在存储器中。

&emsp;&emsp;流管道终端操作的本质也会影响并行执行的有效性。如果与管道的整体工作相比在终端操作中完成了大量工作并且该操作本质上是按顺序的，那么并行化管道的有效性是受限的。并行性最佳的终端操作是*减少（reductions）*，其中从管道中出现的所有元素使用 Stream 的 reduce 方法或减少预打包(prepackaged reductions)（例如 min，max，count 和 sum）进行组合。*短路操作(shortcircuiting)*anyMatch，allMatch 和 noneMatch 也适用于并行操作。Stream 的 collect 方法执行的操作（称为*可变约简( mutable reductions)*）不是并行性的良好选择，因为组合集合的开销是很昂贵的。

&emsp;&emsp;如果你编写自己的 Stream，Iterable 或 Collection 实现并且希望获得良好的并行性能，则必须覆盖 spliterator 方法并广泛测试生成的流的并行性能。编写高质量的 spliterators 是很困难的，超出了本书的范围。

&emsp;&emsp;**并行化流不仅会导致性能不佳，包括活性失败; 它可能导致不正确的结果和不可预测的行为（安全性失败）**。使用映射器，过滤器和其他程序员提供的不符合其规范的功能对象的管道并行化可能会导致安全性失败。Stream 规范对这些功能对象提出了严格的要求。例如，传递给 Stream 的 reduce 操作的累加器和组合器函数必须是关联的，非侵入的和无状态的。如果你违反了这些要求（其中一些在第 46 项中讨论过），但按顺序运行你的管道，则可能会产生正确的结果; 如果你将它并行化，它可能会失败，也许是灾难性的。

&emsp;&emsp;沿着这些思路，值得注意的是，即使并行化的梅森素数程序已经完成，它也不会以正确的（升序）顺序打印素数。要保留顺序版本显示的顺序，你必须使用 forEachOrdered 替换 forEach 终端操作，该操作保证以*相遇顺序(encounter order)*遍历并行流。

&emsp;&emsp;即使假设你正在使用有效可拆分的源流(带有一个并行化或代价低的终端操作)和非侵入（non-interfering）的函数对象，你无法从并行化中获得很好的加速效果，除非管道做了足够的实际工作来抵消使用并行化相关的成本（unless the pipeline is doing enough real work to offset the costs associated with parallelism）。作个非常粗略的估计，流中元素的数量乘以每个元素执行的代码行数应该至少为十万\[Lea14\]。

&emsp;&emsp;重要的是要记住并行化流是严格的性能优化。与任何优化一样，你必须在更改之前和之后测试性能，以确保它【的优化是】值得做【的】（第 67 项）。理想情况下，你应该在实际的系统设置中执行测试。通常，程序中的所有并行流管道都在公共 fork-join 线程池中运行。单个行为不当的管道可能会影响系统中其他不相关部分的行为。

&emsp;&emsp;听起来使用流并行会一直在违背你的意愿，它们确实是这样的(If it sounds like the odds are stacked against you when parallelizing stream pipelines, it’s because they are.)。那些维护数百万行代码的人大量使用流，只发现了在很少数的地方使用并行流是有效地。这并不意味着你应该避免并行化流。**在适当的情况下，只需通过向流管道添加并行调用，就可以实现处理器内核数量的近线性(near-linear)加速**。某些领域，例如机器学习和数据处理，特别适合这些加速。

&emsp;&emsp;作为并行性有效的流管道的一个简单示例，请考虑此函数来计算 π（n），素数小于或等于 n：

```java
// Prime-counting stream pipeline - benefits from parallelization
static long pi(long n) {
    return LongStream.rangeClosed(2, n)
        .mapToObj(BigInteger::valueOf)
        .filter(i -> i.isProbablePrime(50))
        .count();
}
```

&emsp;&emsp;在我的机器上，使用此功能计算 π（10^8）需要 31 秒。 只需添加 parallel（）调用即可将时间缩短为 9.2 秒：

```java
// Prime-counting stream pipeline - parallel version
static long pi(long n) {
    return LongStream.rangeClosed(2, n)
        .parallel()
        .mapToObj(BigInteger::valueOf)
        .filter(i -> i.isProbablePrime(50))
        .count();
}
```

&emsp;&emsp;换句话说，并行化计算可以在我的四核机器上将其加速 3.7 倍。 值得注意的是，这并不是你在实践中如何计算大 n 值的 π（n）。有更高效的算法，特别是 Lehmer 的公式。

&emsp;&emsp;如果要并行化随机数流，请从 SplittableRandom 实例开始，而不是 ThreadLocalRandom（或基本上过时的 Random）。SplittableRandom 是专门为此而设计的，具有线性加速的潜力。ThreadLocalRandom 设计用于单个线程，并将自适应为并行流的源，但不会像 SplittableRandom 一样快。随机同步每个操作，因此会导致过度（近似杀戮）的争抢（so it will result in excessive, parallelism-killing contention）【意思应该是导致的资源争抢会很激烈】。

&emsp;&emsp;总之，除非你有充分的理由相信它将保持计算的正确性并提高其速度，否则就不应该尝试并行化流管道。不恰当地并行化流的成本可能是程序失败或性能灾难。如果你认为并行性可能是合理的，请确保在并行运行时代码保持【运行结果的】正确，并在实际条件下进行详细的性能测试。如果你的代码仍然正确并且这些实验证明你对性能提升的猜疑，那么才能在生产环境的代码中使用并行化流(If your code remains correct and these experiments bear out your suspicion of increased performance, then and only then parallelize the stream in production code.)。

> - [第 47 项：Stream 要优先用 Collection 作为返回类型](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第07章：Lambda和Stream/第47项：Stream要优先用Collection作为返回类型.md)
> - [第 49 项：检查参数的有效性](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第08章：方法/第49项：检查参数的有效性.md)
