## Stream 要优先用 Collection 作为返回类型

&emsp;&emsp;许多方法返回元素序列。 在 Java 8 之前，这些方法的明显返回类型是集合接口 Collection，Set 和 List;Iterable; 和数组类型。通常，很容易决定返回哪些类型。准确来说是一个集合接口。如果该方法仅用于启用 for-each 循环或返回的序列无法实现某些 Collection 方法（通常为 contains(Object)），则使用 Iterable 接口。如果返回的元素是基本类型值或者存在严格的性能要求，则使用数组。在 Java 8 中，流被添加到平台中，这使得为返回序列的方法选择恰当的返回类型的任务变得非常复杂。

&emsp;&emsp;你可能听说过，流现在是返回一系列元素的公认选择，正如第 45 项所描述的，流不会使迭代过时：编写好的代码需要适当地组合流和迭代。如果 API 只返回一个流，而某些用户想要使用 for-each 循环迭代返回的序列，那么这些用户理所当然会感到不安。特别令人沮丧的是，Stream 接口包含 Iterable 接口中唯一的抽象方法，Stream 的此方法规范与 Iterable 兼容。

&emsp;&emsp;可悲的是，这个问题没有好的解决方法。乍一看，似乎可以将方法引用传递给 Stream 的迭代器方法。 结果代码可能有点嘈杂和模糊，但并非不合理：

```java
// Won't compile, due to limitations on Java's type inference
for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
    // Process the process
}
```

&emsp;&emsp;不幸的是，如果你尝试编译此代码，你将收到一条错误消息：

```java
Test.java:6: error: method reference not expected here
for (ProcessHandle ph : ProcessHandle.allProcesses()::iterator) {
                        ^
```

&emsp;&emsp;为了使代码编译，你必须将方法引用强制转换为适合参数化的 Iterable：

```java
// Hideous workaround to iterate over a stream
for (ProcessHandle ph : (Iterable<ProcessHandle>) ProcessHandle.allProcesses()::iterator)
```

&emsp;&emsp;此客户端代码有效，但在实践中使用它太嘈杂和模糊。更好的解决方法是使用适配器方法。JDK 没有提供这样的方法，但是使用上面的代码片段中使用的相同技术，可以很容易地编写一个方法。请注意，在适配器方法中不需要强制转换，因为 Java 类型推断在此上下文中正常工作：

```java
// Adapter from Stream<E> to Iterable<E>
public static <E> Iterable<E> iterableOf(Stream<E> stream) {
    return stream::iterator;
}
```

&emsp;&emsp;使用此适配器，你可以使用 for-each 语句迭代任何流：

```java
for (ProcessHandle p : iterableOf(ProcessHandle.allProcesses())) {
    // Process the process
}
```

&emsp;&emsp;请注意，第 34 项中的 Anagrams 程序的流版本使用 Files.lines 方法读取字典，而迭代版本使用 scanner。Files.lines 方法优于 scanner，它可以在读取文件时悄悄地处理（silently swallows）任何异常。理想情况下，我们也会在迭代版本中使用 Files.lines。如果 API 仅提供对序列的流访问并且他们希望使用 for-each 语句迭代序列，那么程序员将会做出这种折中的方法【在迭代版本中使用 Files.lines】。

&emsp;&emsp;相反，想要使用流管道处理序列的程序员理所当然会因为 API 仅提供 Iterable 而感到难过【傲娇】。再提一次 JDK 没有提供适配器，但编写一个是很容易的：

```java
// Adapter from Iterable<E> to Stream<E>
public static <E> Stream<E> streamOf(Iterable<E> iterable) {
    return StreamSupport.stream(iterable.spliterator(), false);
}
```

&emsp;&emsp;如果你正在编写一个返回一系列对象的方法，并且你知道它只会在流管道中使用，那么你当然可以随意返回一个流。类似地，返回仅用于迭代的序列的方法应返回 Iterable。但是，如果你正在编写一个返回序列的公共 API，那么你应该为想要编写流管道的用户以及想要编写 for-each 语句的用户提供服务。除非你有充分的理由相信【使用该 API 的】大多数用户希望使用相同的机制。

&emsp;&emsp;Collection 接口是 Iterable 的子类型，并且具有 stream 方法，因此它提供迭代和流访问。因此，**Collection 或适当的子类型通常是公共序列返回方法的最佳返回类型。** Arrays 还提供了 Arrays.asList 和 Stream.of 方法的简单迭代和流访问。如果你返回的序列小到足以容易地放入内存中，那么最好返回一个标准集合实现，例如 ArrayList 或 HashSet。但是**不要在内存中存储大的序列而只是为了将它作为集合返回。**

&emsp;&emsp;如果你返回的序列很大但可以简洁地表示，请考虑实现一个特殊用途的集合。例如，假设你要返回给定集的*幂集（ power set）*，该集包含其所有子集。{a，b，c}的幂集为{{}，{a}，{b}，{c}，{a，b}，{a，c}，{b，c}，{a，b ， C}}。如果一个集合具有 n 个元素，则其幂集具有 2^n 个。因此，你甚至不应该考虑将幂集存储在标准集合的实现中。但是，在 AbstractList 的帮助下，很容易为此实现自定义集合。

&emsp;&emsp;技巧是使用幂集中每个元素的索引作为位向量，其中索引中的第 n 位表示源集合中是否存在第 n 个元素。本质上，从 0 到 2^n - 1 的二进制数和 n 个元素集的幂集之间存在自然映射。以下是代码：

```java
// Returns the power set of an input set as custom collection
public class PowerSet {
    public static final <E> Collection<Set<E>> of(Set<E> s) {
        List<E> src = new ArrayList<>(s);
        if (src.size() > 30)
            throw new IllegalArgumentException("Set too big " + s);
        return new AbstractList<Set<E>>() {
            @Override public int size() {
                return 1 << src.size(); // 2 to the power srcSize
            }
            @Override public boolean contains(Object o) {
                return o instanceof Set && src.containsAll((Set)o);
            }
            @Override public Set<E> get(int index) {
            Set<E> result = new HashSet<>();
                for (int i = 0; index != 0; i++, index >>= 1)
                    if ((index & 1) == 1)
                        result.add(src.get(i));
                return result;
            }
        };
    }
}
```

&emsp;&emsp;请注意，如果输入集具有超过 30 个元素，则 PowerSet.of 会抛出异常。这突出了使用 Collection 作为返回类型的缺点(而 Stream 或 Iterable 没有该缺点）：Collection 具有 int 返回大小方法，该方法将返回序列的长度限制为 Integer.MAX_VALUE 或 2^31-1。如果集合更大，甚至无限，Collection 规范允许 size 方法返回 2^31-1，但这不是一个完全令人满意的解决方案。

&emsp;&emsp;为了在 AbstractCollection 上编写 Collection 实现，你只需要实现 Iterable 所需的两个方法：contains 和 size。通常，编写这些方法的有效实现是很容易的。如果不可行，可能是因为在迭代发生之前无法预先确定序列的内容，返回流或可迭代的，哪种感觉起来更自然就返回哪种。如果你要选择的话，你可以使用两种不同的方法将两种类型都返回。

&emsp;&emsp;有时你会根据实施的方式容易程度选择返回类型。例如，假设你要编写一个返回输入列表的所有（连续）子列表的方法。生成这些子列表只需要三行代码并将它们放在标准集合中，但保存此集合所需的内存是源列表大小的二次方。虽然这并不像指数级的幂集那么糟糕，但显然是不可接受的。正如我们为幂集所做的那样，实现自定义集合将是冗长的，因为 JDK 缺乏 Iterator 框架实现来帮助我们。

&emsp;&emsp;但是【我们可以】直接实现输入列表的所有子列表的流，尽管它确实需要一些洞察力。让我们调用一个子列表，该子列表包含列表的第一个元素和列表的*前缀（prefix）*。例如，（a，b，c）的前缀是（a），（a，b）和（a，b，c）。 类似地，让我们调用包含后缀的最后一个元素的子列表，因此（a，b，c）的后缀是（a，b，c），（b，c）和（c）。洞察的点就是列表的子列表只是前缀的后缀（或相同的后缀的前缀）和空列表。通过这个观点直接就可以有了清晰、合理简洁的实施方案：

```java
// Returns a stream of all the sublists of its input list
public class SubLists {
    public static <E> Stream<List<E>> of(List<E> list) {
        return Stream.concat(Stream.of(Collections.emptyList()), prefixes(list).flatMap(SubLists::suffixes));
    }
    private static <E> Stream<List<E>> prefixes(List<E> list) {
        return IntStream.rangeClosed(1, list.size()).mapToObj(end -> list.subList(0, end));
    }
    private static <E> Stream<List<E>> suffixes(List<E> list) {
        return IntStream.range(0, list.size()).mapToObj(start -> list.subList(start, list.size()));
    }
}
```

&emsp;&emsp;请注意，Stream.concat 方法用于将空列表添加到返回的流中。另请注意，flatMap 方法（第 45 项）用于生成由所有前缀的所有后缀组成的单个流。最后，请注意我们通过映射 IntStream.range 和 IntStream.rangeClosed 返回的连续 int 值流来生成前缀和后缀。粗略地说，这个习惯用法是整数索引上标准 for 循环的流等价物（ This idiom is, roughly speaking, the stream equivalent of the standard for-loop on integer indices）。因此，我们的子列表实现的思想类似明显的嵌套 for 循环：

```java
for (int start = 0; start < src.size(); start++)
    for (int end = start + 1; end <= src.size(); end++)
        System.out.println(src.subList(start, end));
```

&emsp;&emsp;可以将此 for 循环直接转换为流。结果比我们之前的实现更简洁，但可读性稍差。它的思想类似第 45 项中笛卡尔积的流代码：

```java
// Returns a stream of all the sublists of its input list
public static <E> Stream<List<E>> of(List<E> list) {
    return IntStream.range(0, list.size())
        .mapToObj(start -> IntStream.rangeClosed(start + 1, list.size())
        .mapToObj(end -> list.subList(start, end)))
        .flatMap(x -> x);
}
```

&emsp;&emsp;与之前的 for 循环一样，此代码不会产生（emit）空列表。为了解决这个问题，你可以使用 concat，就像我们在之前版本中所做的那样，或者在 rangeClosed 调用中用（int）Math.signum（start）替换 1。

&emsp;&emsp;这些子列表的流实现中的任何一个都很好，但两者都需要用户使用一些 Stream-to-Iterable 适配器或在迭代更自然的地方使用流。Stream-to-Iterable 适配器不仅使客户端代码混乱，而且还会使我的机器上的循环速度降低 2.3 倍。专用的 Collection 实现（此处未显示）相当冗长，但运行速度是我机器上基于流的实现的 1.4 倍。

&emsp;&emsp;总之，在编写返回元素序列的方法时，请记住，你的某些用户可能希望将它们作为流处理，而其他用户可能希望使用它们进行迭代。尽量适应这两个群体。如果返回集合是可行的，那么就这么做【返回集合】。如果你已经拥有集合中的元素，或者序列中的元素数量很小足以证明创建新元素是正确的，那么就返回标准集合，例如 ArrayList。否则，请考虑实现自定义的集合，就像我们为幂集所做的那样。如果返回集合是不可行的，则返回一个流或可迭代的，无论哪个看起来更自然。如果在将来的 Java 版本中，Stream 接口声明被修改为扩展（extend）Iterable，那么你应该随意返回流，因为它们将允许流处理和迭代。

> - [第 46 项：优先选择 Stream 中无副作用的函数](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第07章：Lambda和Stream/第46项：优先选择Stream中无副作用的函数.md)
> - [第 48 项：谨慎使用 Stream 并行](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第07章：Lambda和Stream/第48项：谨慎使用Stream并行.md)
