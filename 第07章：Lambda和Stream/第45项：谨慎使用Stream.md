## 谨慎使用 Stream

&emsp;&emsp;在 Java 8 中添加了 Stream API，以简化顺序或并行执行批量操作的任务。这个 API 提供了两个关键的抽象概念：*流（stream）*表示有限或无限的数据元素序列，*流管道（stream pipeline）*表示对这些元素的多级计算。流中的元素可以来自任何地方。常见的来源包括集合，数组，文件，正则表达式模式匹配器，伪随机数生成器和其他流。流中的数据元素可以是对象的引用或基本类型。支持三种基本类型：int，long 和 double。

&emsp;&emsp;流管道由源流和零个或多个*中间操作（intermediate operations ）*以及一个*终端操作（ terminal operation）*组成。每个中间操作以某种方式转换流，例如将每个元素映射到该元素的函数或过滤掉不满足某些条件的所有元素。 中间操作都将一个流转换为另一个流，其元素类型可以与输入流相同或与之不同。终端操作对从最后的中间操作产生的流执行最终计算，例如将其元素存储到集合中，返回某个元素或打印其所有元素。

&emsp;&emsp;流管道是懒求值（evaluated lazily）：在调用终端操作之前是不会开始求值的，并且不会去计算那些在完成终端操作的过程中不需要的数据元素。这种懒求值使得可以使用无限流。请注意，没有终端操作的流管道是静默无操作的，因此不要忘记包含一个【终端操作】（Stream pipelines are evaluated lazily: evaluation doesn’t start until the terminal operation is invoked, and data elements that aren’t required in order to complete the terminal operation are never computed. This lazy evaluation is what makes it possible to work with infinite streams. Note that a stream pipeline without a terminal operation is a silent no-op, so don’t forget to include one. ）。

&emsp;&emsp;流 API 非常流畅：它旨在允许将构成管道的所有调用链接（chain）到单个表达式中。 实际上，多个管道可以链接（chain）在一起形成一个表达式。

&emsp;&emsp;默认情况下，流管道按顺序运行。使管道并行执行就像在管道中的任何流上调用并行方法一样简单，但很少这样做（第 48 项）。

&emsp;&emsp;流 API 具有足够的通用性（The streams API is sufficiently versatile），几乎任何计算都可以使用流来执行，但仅仅因为你可以这么做并不意味着你应该这样做。如果使用得当，流可以使程序更短更清晰; 如果使用不当，可能会使程序难以阅读和维护。

&emsp;&emsp;考虑以下程序，该程序从字典文件中读取单词并打印其大小符合用户指定的最小值的所有相同字母异序词组（anagram groups）。回想一下，如果两个单词由不同顺序的相同字母组成，则它们是相同字母异序词。程序从用户指定的字典文件中读取每个单词并将单词放入 map 中。map 的键是用字母按字母顺序排列的单词，因此“staple”的键是“aelpst”，“petals”的键也是“aelpst”：两个单词是相同字母异序词，所有的相同字母异序词共享相同的字母形式（或 alphagram，因为它有时是已知的（(or alphagram, as it is sometimes known））。map 的值是包含共享按字母顺序排列的形式的所有单词的列表。字典处理完毕后，每个列表都是一个完整的相同字母异序词组。然后程序遍历 map 的 values（）并打印每个大小符合阈值的列表：

```java
// Prints all large anagram groups in a dictionary iteratively
public class Anagrams {
    public static void main(String[] args) throws IOException {
        File dictionary = new File(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        Map<String, Set<String>> groups = new HashMap<>();
        try (Scanner s = new Scanner(dictionary)) {
            while (s.hasNext()) {
                String word = s.next();
                groups.computeIfAbsent(alphabetize(word), (unused) -> new TreeSet<>()).add(word);
            }
        }
        for (Set<String> group : groups.values())
            if (group.size() >= minGroupSize)
                System.out.println(group.size() + ": " + group);
    }

    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}
```

&emsp;&emsp;该计划的一个步骤值得注意。 将每个单词插入到地图中（以粗体显示：groups.computeIfAbsent(alphabetize(word), (unused) -> new TreeSet<>()).add(word);）使用了在 Java 8 中添加的 computeIfAbsent 方法。此方法在 map 中查找键：如果键存在，则该方法仅返回与其关联的值。如果不是，则该方法通过将给定的函数对象应用于键来计算值，将该值与键相关联，并返回计算的值。computeIfAbsent 方法简化了将多个值与每个键相关联的映射的实现。

&emsp;&emsp;现在考虑以下程序，它解决了同样的问题，但大量使用了流。请注意，除了打开字典文件的代码之外，整个程序都包含在一个表达式中。在单独的表达式中打开字典的唯一原因是允许使用 try-with-resources 语句，以确保字典文件已关闭：

```java
// Overuse of streams - don't do this!
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(
                groupingBy(word -> word.chars().sorted()
                    .collect(StringBuilder::new,
                    (sb, c) -> sb.append((char) c),
                    StringBuilder::append).toString()))
            .values().stream()
            .filter(group -> group.size() >= minGroupSize)
            .map(group -> group.size() + ": " + group)
            .forEach(System.out::println);
        }
    }
}
```

&emsp;&emsp;如果你发现此代码难以阅读，请不要担心; 你不是一个人。它更短，但可读性更小，特别是对于不是使用流的专家级程序员。**过度使用流会使程序难以阅读和维护。**

&emsp;&emsp;幸运的是，有一个让人开心的工具。以下程序使用流而不过度使用流来解决相同的问题。结果是一个比原始程序更短更清晰的程序：

```java
// Tasteful use of streams enhances clarity and conciseness
public class Anagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);
        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(groupingBy(word -> alphabetize(word)))
                .values().stream()
                .filter(group -> group.size() >= minGroupSize)
                .forEach(g -> System.out.println(g.size() + ": " + g));
        }
    }
    // alphabetize method is the same as in original version
}
```

&emsp;&emsp;即使你以前很少接触过流，这个程序也不难理解。它在 try-with-resources 块中打开字典文件，获取包含文件中所有行的流。stream 变量被命名为 words，表示流中的每个元素都是一个 word。此流上的管道没有中间操作; 它的终端操作将所有 word 收集到一个 map 中，该 map 按字母顺序排列单词（第 46 项）。这与在以前版本的程序中构建的 map 完全相同。然后在 map 的 values（）中打开一个新的 Stream<List<String>>。当然，这个流中的元素是相同字母异序词组。过滤流以便忽略大小小于 minGroupSize 的所有组，最后，通过终端操作 forEach 打印剩余的组。

&emsp;&emsp;请注意，小心选择了 lambda 参数名称。 参数 g 应该真正命名为 group，但是生成的代码行对于本书来说太宽了。**在没有显式类型的情况下，仔细命名 lambda 参数对于流管道的可读性至关重要。**

&emsp;&emsp;另请注意，单词字母化是在单独的 alphabetize 方法中完成的。这通过提供操作的名称并将实现细节保留在主程序之外来增强可读性。**使用辅助方法对于流管道中的可读性比在迭代代码中更为重要，** 因为管道缺少显式类型信息和命名临时变量。

&emsp;&emsp;可以使用流重新实现 alphabetize 方法，但是基于流的 alphabetize 方法不太清晰，更难以正确编写，并且可能更慢。这些缺陷是由于 Java 缺乏对原始 char 流的支持（这并不意味着 Java 应该支持 char 流；这样做是不可行的）。要演示使用流处理 char 值的危险，请考虑以下代码：

```java
"Hello world!".chars().forEach(System.out::print);
```

&emsp;&emsp;你可能希望它打印 Hello world！，但如果你运行它，你会发现它打印 721011081081113211911111410810033。这是因为“Hello world！”.chars（）返回的流的元素不是 char 值而是 int 值，因此调用的是 print 的 int 重载【方法】。令人遗憾的是，名为 chars 的方法返回一个 int 值流。你可以通过使用强制转换来强制调用正确的重载来修复程序：

```java
"Hello world!".chars().forEach(x -> System.out.print((char) x));
```

&emsp;&emsp;但理想情况下，你应该**避免使用流来处理 char 值。**

&emsp;&emsp;当你开始使用流时，你可能会有将所有循环转换为流的冲动的感觉，但抵制这种高冲动。尽管这只是有可能发生，但它会损害代码库的可读性和可维护性。通常，使用流和迭代的某种组合可以最好地完成复杂程度中等的任务，如上面的 Anagrams 程序所示。因此，**重构现有代码以使用流，并仅在有意义的情况下在新代码中使用它们。**

&emsp;&emsp;如该项目中的程序所示，流管道使用函数对象（通常是 lambdas 或方法引用）表示重复计算，而迭代代码使用代码块表示重复计算。以下操作你可以在代码块中执行，但无法在函数对象中执行：

- 在代码块中，你可以读取或修改范围内的任何局部变量; 在 lambda 中，你只能读取最终或有效的最终变量[JLS 4.12.4]，并且你无法修改任何局部变量。

- 在代码块中，不可以从封闭方法返回，中断或继续封闭循环，或抛出声明此方法被抛出的任何已受检异常; 在一个 lambda 你无法做到这些事情。

&emsp;&emsp;如果使用这些技巧可以更好地表达计算【过程】，那么流就可能不是最好的方式（If a computation is best expressed using these techniques, then it’s probably not a good match for streams）。相反，流可以很容易做一些事情：

- 均匀地转换元素序列
- 过滤元素序列
- 使用单个操作组合元素序列（例如，添加它们，串联（concatenate ）它们或计算它们的最小值）
- 将元素序列累积（accumulate）到集合中，或者通过一些常见属性对它们进行分组
- 在元素序列中搜索满足某个条件的元素

&emsp;&emsp;如果使用这些技巧可以更好地表达计算【过程】，那么流是它的良好候选者。

&emsp;&emsp;使用流很难做的一件事是同时从管道的多个阶段访问相应的元素：一旦将值映射到某个其他值，原始值就会丢失。一种解决方法是将每个值映射到包含原始值和新值的*对对象（pair object）*，但这不是一个令人满意的解决方案，尤其是如果管道的多个阶段需要对对象。 由此产生的代码是混乱和冗长的，这破坏了流的主要目的。如果适当使用的话，更好的解决方法是在需要访问早期阶段值的时候反转映射。（When it is applicable, a better workaround is to invert the mapping when you need access to the earlier-stage value）。

&emsp;&emsp;例如，让我们编写一个程序来打印前 20 个*梅森素数(Mersenne primes)*。为了更新你的记忆，梅森数是一个 2^p-1 的数字。如果 p 是素数，相应的梅森数可能是素数; 如果是这样的话，那就是梅森素数。作为我们管道中的初始流，我们需要所有素数。这是一种返回该（无限）流的方法。我们假设使用静态导入来轻松访问 BigInteger 的静态成员：

```java
static Stream<BigInteger> primes() {
    return Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```

&emsp;&emsp;方法（primes）的名称是描述流的元素的复数名词。强烈建议所有返回流的方法使用此命名约定，因为它增强了流管道的可读性。该方法使用静态工厂 Stream.iterate，它接受两个参数：流中的第一个元素，以及从前一个元素生成流中的下一个元素的函数。这是打印前 20 个梅森素数的程序:

```java
public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
        .filter(mersenne -> mersenne.isProbablePrime(50))
        .limit(20)
        .forEach(System.out::println);
}
```

&emsp;&emsp;这个程序是上文描述中的直接编码：它从素数开始，计算相应的梅森数，过滤掉除素数之外的所有数字（幻数 50 控制概率素性测试（the magic number 50 controls the probabilistic primality tes）），将得到的流限制为 20 个元素，并打印出来。

&emsp;&emsp;现在假设我们想要在每个梅森素数之前加上它的指数（p）。该值仅出现在初始流中，因此在终端操作中无法访问，从而打印结果。幸运的是，通过反转第一个中间操作中发生的映射，可以很容易地计算出梅森数的指数。指数只是二进制表示中的位数，因此该终端操作生成所需的结果：

```java
.forEach(mp -> System.out.println(mp.bitLength() + ": " + mp));
```

&emsp;&emsp;有很多任务，无论是使用流还是迭代都不明显。例如，考虑初始化一副新牌的任务。假设 Card 是一个值不可变的类，它封装了 Rank 和 Suit，两者都是枚举类型。此任务代表任何需要的计算可以从两组中选择所有元素对的任务。数学家称之为两组的*笛卡尔积（Cartesian product ）*。这是一个带有嵌套 for-each 循环的迭代实现，对你来说应该很熟悉：

```java
// Iterative Cartesian product computation
private static List<Card> newDeck() {
    List<Card> result = new ArrayList<>();
    for (Suit suit : Suit.values())
        for (Rank rank : Rank.values())
            result.add(new Card(suit, rank));
    return result;
}
```

&emsp;&emsp;这是一个基于流的实现，它使用了中间操作 flatMap。此操作将流中的每个元素映射到流，然后将所有这些新流连接成单个流（或展平它们(or flattens them)）。请注意，此实现包含嵌套的 lambda，以粗体显示；

```java
// Stream-based Cartesian product computation
private static List<Card> newDeck() {
    return Stream.of(Suit.values())
        .flatMap(suit ->
            Stream.of(Rank.values())
                .map(rank -> new Card(suit, rank)))
        .collect(toList());
}
```

&emsp;&emsp;newDeck 的两个版本中哪一个更好？它归结为个人偏好和你的编程环境。第一个版本更简单，也许感觉更自然。大部分 Java 程序员将能够理解和维护它，但是一些程序员会对第二个（基于流的）版本感觉更舒服。如果你对流和函数式编程很精通，那么它会更简洁，也不会太难理解。如果你不确定自己喜欢哪个版本，则迭代版本可能是更安全的选择。如果你更喜欢流版本，并且你相信其他使用该代码的程序员跟你有共同的偏好，那么你应该使用它。

&emsp;&emsp;总之，一些任务最好用流完成，其他任务最好用迭代完成。 通过组合这两种方法可以最好地完成许多任务。选择哪种方法用于任务没有硬性规定，但有一些有用的启发式方法。在许多情况下，将清楚使用哪种方法; 在某些情况下，它不会。**如果你不确定某个任务是否更适合流或迭代，那么就两个都尝试一下，并看一下哪个更好。**

> - [第 44 项：坚持使用标准的函数接口](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第07章：Lambda和Stream/第44项：坚持使用标准的函数接口.md)
> - [第 46 项：优先选择 Stream 中无副作用的函数](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第07章：Lambda和Stream/第46项：优先选择Stream中无副作用的函数.md)
