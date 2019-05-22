## 优先选择Stream中无副作用的函数

&emsp;&emsp;如果你是一个【使用】流的新手，可能很难掌握它们。仅仅将你的计算【过程】表示为流管道可能很难。当你成功的时候【成功地将计算过程用流管道表示出来】，你的程序会运行，但你可能几乎没有任何好处。Streams不仅仅是一个API，它还是一个基于函数式编程的范例。为了获得流必须提供的表现力，速度和某些情况下的并行性，你必须采用范例和API。

&emsp;&emsp;流范例中最重要的部分是将计算结构化为一系列转换，其中每个阶段的结果尽可能接近前一阶段结果的*纯函数（ pure function ）*。纯函数的【执行】结果取决于其输入：它不依赖于任何可变状态，也不更新任何状态。为了实现这一点，你传递给流操作的任何函数对象（中间或终端）都应该没有副作用。

&emsp;&emsp;有时，你可能会看到类似于此代码段的流代码，它会在文本文件中构建单词的频率表：

```java
// Uses the streams API but not the paradigm--Don't do this!
Map<String, Long> freq = new HashMap<>();
try (Stream<String> words = new Scanner(file).tokens()) {
    words.forEach(word -> {
        freq.merge(word.toLowerCase(), 1L, Long::sum);
    });
}
```

&emsp;&emsp;这段代码出了什么问题？ 毕竟，它使用流，lambdas和方法引用，并得到正确的答案。简单地说，它根本不是流代码; 它的迭代代码伪装成流代码。它没有从流API中获益，并且它比相应的迭代代码更长，更难以阅读，并且可维护性更小。问题源于这样一个事实：这个代码在一个终端forEach操作中完成所有工作，使用一个变异外部状态的lambda（频率表）。执行除了呈现流执行的计算结果之外的任何操作的forEach操作都是“代码中的坏味道”，就比如一个变异状态的lambda。那么这段代码应该怎么样？

```java
// Proper use of streams to initialize a frequency table
Map<String, Long> freq;
try (Stream<String> words = new Scanner(file).tokens()) {
    freq = words
        .collect(groupingBy(String::toLowerCase, counting()));
}
```

&emsp;&emsp;此代码段与前一代码相同，但正确使用了流API。它更短更清晰。那么为什么有人会用另一种方式写呢？ 因为它使用了他们已经熟悉的工具。Java程序员知道如何使用for-each循环，而forEach终端操作是类似的。但forEach操作是终端操作中最不强大的操作之一，也是最不友好的流操作。它很显然是使用了迭代，因此不适合并行化。**forEach操作应仅用于报告流计算的结果，而不是用于执行计算。** 有时，将forEach用于其他目的是有意义的，例如将流计算的结果添加到预先存在的集合中。

&emsp;&emsp;改进的代码使用了一个*收集器（collector）*，这是一个新概念，你必须学习了才能使用流。Collectors API是令人生畏的：它有三十九种方法，其中一些方法有多达五种类型参数。好消息是，你可以从这个API中获得大部分益处，而无需深入研究其完整的复杂性。对于初学者，你可以忽略Collector接口，并将收集器视为封装缩减策略的不透明对象（an opaque object that encapsulates a reduction strategy）。在这种情况下，缩减意味着将流的元素组合成单个对象。收集器生成的对象通常是一个集合（它代表名称收集器（(which accounts for the name collector））。

&emsp;&emsp;用于将流的元素收集到真正的集合中的收集器是很简单的。有三个这样的收集器：toList（），toSet（）和toCollection（collectionFactory）。它们分别返回一个集合，一个列表和一个程序员指定的集合类型。有了这些知识，我们可以编写一个流管道来从频率表中提取前十个列表。

```java
// Pipeline to get a top-ten list of words from a frequency table
List<String> topTen = freq.keySet().stream()
    .sorted(comparing(freq::get).reversed())
    .limit(10)
    .collect(toList());
```

&emsp;&emsp;请注意，我们没有使用其类Collectors限定toList方法。**习惯性地将收集器的所有成员都静态导入是明智的，因为它使流管道更具可读性。**

&emsp;&emsp;这段代码中唯一棘手的是我们传递给sorted【方法】的部分，compare(freq::get).reversed()的比较器。comparing方法是采用密钥提取功能的比较器构造方法（第14项）。该函数接收一个单词，“提取（extraction）”实际上是一个表查找：绑定方法引用freq::get在频率表中查找单词并返回单词在文件中出现的次数。最后，我们在比较器上调用reverse，因此我们将单词【出现的频率】从最频繁到最不频繁进行排序。然后将流限制为十个单词并将它们收集到一个列表中是一件简单的事情。

&emsp;&emsp;之前的代码片段使用Scanner的流方法通过扫描程序获取流。该方法时在Java 9中添加的。如果你使用的是早期版本，则可以使用类似于第47项（streamOf（Iterable <E>））的适配器来将实现了Iterator的scanner转换为流。

&emsp;&emsp;那么Collectors的其他36种方法呢？它们中的大多数存在是为了让你将流收集到map中，这比将它们收集到真实集合中要复杂得多。每个流元素与键和值相关联，并且多个流元素可以与相同的键相关联。

&emsp;&emsp;最简单的map收集器是toMap（keyMapper，valueMapper），它接受两个函数，其中一个函数将一个流元素映射到一个键，另一个函数映射到一个值。我们在第34项的fromString实现中使用了这个收集器来创建从枚举的字符串形式到枚举本身的映射：

```java
// Using a toMap collector to make a map from string to enum
private static final Map<String, Operation> stringToEnum = Stream.of(values()).collect(toMap(Object::toString, e -> e));
```

&emsp;&emsp;如果流中的每个元素都映射到唯一键，则这种简单的toMap形式是完美的。 如果多个流元素映射到同一个键，则管道将以IllegalStateException异常来终止【计算】。

&emsp;&emsp;更复杂的toMap形式（比如groupingBy方法）为你提供了各种方法来提供处理此类冲突的策略。一种方法是除了键和值映射器之外，还为toMap方法提供合并函数。合并函数是BinaryOperator<V>，其中V是映射的值类型。使用合并函数将与键关联的任何其他值与现有值组合，因此，例如，如果合并函数是乘法，则通过值映射最终得到的值是与键关联的所有值的乘积。

&emsp;&emsp;toMap的三参数形式对于创建从键到与该键关联的所选元素的映射也很有用。例如，假设我们有各种艺术家的唱片专辑流，我们想要一个从录音艺术家到最畅销专辑的map映射。这个collector就能完成这项工作。

```java
// Collector to generate a map from key to chosen element for key
Map<Artist, Album> topHits = albums.collect(toMap(Album::artist, a->a, maxBy(comparing(Album::sales))));
```

&emsp;&emsp;请注意，比较器使用静态工厂方法maxBy，它是从BinaryOperator静态导入的。此方法将Comparator<T>转换为BinaryOperator<T>，用于计算指定比较器隐含的最大值。在这种情况下，比较器由比较器构造方法comparing返回，它采用密钥提取器功能(key extractor function)Album::sales。这可能看起来有点复杂，但代码可读性很好。简而言之，它说，“将专辑流转换为map，将每位艺术家映射到销售量最佳专辑的专辑。”这接近问题的陈述【程度】凌然感到惊讶【意思就是说这代码的意思很接近问题的描述（OS：臭不要脸）】。

&emsp;&emsp;toMap的三参数形式的另一个用途是产生一个收集器，当发生冲突时强制执行last-write-wins策略【保留最后一个冲突值】。对于许多流，结果将是不确定的，但如果映射函数可能与键关联的所有值都相同，或者它们都是可接受的，则此收集器的行为可能正是你想要的：

```java
// Collector to impose last-write-wins policy 
toMap(keyMapper, valueMapper, (v1, v2) -> v2)
```

&emsp;&emsp;toMap的第三个也是最后一个版本采用第四个参数，即一个map工厂，用于指定特定的map实现，例如EnumMap或TreeMap。

&emsp;&emsp;toMap的前三个版本也有变体形式，名为toConcurrentMap，它们并行高效运行并生成ConcurrentHashMap实例。

&emsp;&emsp;除了toMap方法之外，Collectors API还提供了groupingBy方法，该方法返回【一个】收集器用来生成基于*分类器函数（classifier function）*将元素分组到类别中的映射。分类器函数接收一个元素并返回它的所属类别。此类别用作元素的map的键。groupingBy方法的最简单版本是仅采用分类器并返回一个映射，其值是每个类别中所有元素的列表。这是我们在第45项中的Anagram程序中使用的收集器，用于生成从按字母顺序排列的单词到共享字母顺序的单词列表的映射：

```java
words.collect(groupingBy(word -> alphabetize(word)))
```

&emsp;&emsp;如果希望groupingBy返回一个生成带有除列表之外的值的映射的收集器，则除了分类器之外，还可以指定*下游收集器（downstream collector）*。下游收集器从一个包含类别中所有元素的流中生成一个值。此参数的最简单用法是传递toSet()，这将生成一个映射，其值是元素集而不是列表。这会生成一个映射，该映射将每个类别与类别中的元素数相关联，而不是包含元素的集合。这就是你在本项目开头的频率表示例中看到的内容：

```java
Map<String, Long> freq = words.collect(groupingBy(String::toLowerCase, counting()));
```

&emsp;&emsp;groupingBy的第三个版本允许你指定除下游收集器之外的map工厂。请注意，此方法违反了标准的telescoping参数列表模式：mapFactory参数位于downStream参数之前，而不是之后。此版本的groupingBy使你可以控制包含的映射以及包含的集合（This version of groupingBy gives you control over the containing map as well as the contained collections），因此，例如，你可以指定一个收集器，该收集器返回一个value为TreeSet的TreeMap。

&emsp;&emsp;groupingByConcurrent方法提供了groupingBy的所有三个重载的变体。 这些变体并行高效运行并生成ConcurrentHashMap实例。还有一个很少使用的grouping的相近【的方法】叫做partitioningBy。代替分类器方法，它接收一个谓词（predicate）并返回键为布尔值的map。此方法有两个重载【版本】，其中一个除谓词之外还包含下游收集器。通过counting方法返回的收集器仅用作下游收集器。通过count方法直接在Stream上提供相同的功能，因此**没有理由说collect(counting())（ there is never a reason to say collect(counting())）** 。此属性还有十五种收集器方法。它们包括九个方法，其名称以summing，averaging和summarizing开头（其功能在相应的基本类型流上可用）。它们还包括reducing方法的所有重载，以及filter，mapping，flatMapping和collectingAndThen方法。大多数程序员可以安心地忽略大多数这种方法。从设计角度来看，这些收集器代表了尝试在收集器中部分复制流的功能，以便下游收集器可以充当“迷你流（ministreams）”。

&emsp;&emsp;我们还有三种Collectors方法尚未提及。虽然他们在Collectors里面，但他们不涉及集合。前两个是minBy和maxBy，它们取比较器并返回由比较器确定的流中的最小或最大元素。它们是Stream接口中min和max方法的小扩展【简单的实现】，是BinaryOperator中类似命名方法返回的二元运算符的收集器类似物。回想一下，我们在最畅销专辑的例子中使用了BinaryOperator.maxBy。

&emsp;&emsp;最后的Collectors方法是join，它只对CharSequence实例的流进行操作，例如字符串。 在其无参数形式中，它返回一个简单地连接元素的收集器。它的一个参数形式采用名为delimiter的单个CharSequence参数，并返回一个连接流元素的收集器，在相邻元素之间插入分隔符。如果传入逗号作为分隔符，则收集器将返回逗号分隔值字符串（但请注意，如果流中的任何元素包含逗号，则字符串将不明确）。除了分隔符之外，三个参数形式还带有前缀和后缀。 生成的收集器会生成类似于打印集合时获得的字符串，例如\[came, saw, conquered\]。

&emsp;&emsp;总之，流管道变成的本质是无副作用的功能对象。这适用于传递给流和相关对象的几乎所有的函数对象（This applies to all of the many function objects passed to streams and related objects）。终端操作forEach仅应用于报告流执行的计算结果，而不是用于执行计算。为了正确使用流，你必须了解收集器。最重要的收集器工厂是toList，toSet，toMap，groupingBy和join。