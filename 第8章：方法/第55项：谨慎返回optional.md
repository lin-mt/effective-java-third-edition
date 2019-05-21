### 谨慎返回optional

&emsp;&emsp;在Java 8之前，在编写在某些情况下无法返回值的方法时，可以采用两种方法。 你可以抛出异常，也可以返回null（假设返回类型是对象引用类型）。这些方法都不完美。【出现】特殊的情况才应该保留异常（第69项），抛出异常【的成本】是很昂贵的，因为在创建异常时会捕获整个堆栈【的】跟踪【信息】。返回null没有这些缺点，但它有自己的缺点。如果方法返回null，则客户端必须包含（include【使用】）特殊的代码以处理返回null的可能性，除非程序员可以证明无法返回null。如果客户端忽略检查空返回并在某些数据结构中存储空返回值，则NullPointerException可能在将来的某个任意时间的某个位置出现问题，而该位置与出现这个问题的根源不在同一个地方【这么翻译应该是没毛病的】（a NullPointerException may result at some arbitrary time in the future, at some place in the code that has nothing to do with the problem）。

&emsp;&emsp;在Java 8中，有第三种方法来编写可能无法返回值的方法。Optional <T>类表示一个不可变容器，它可以包含单个非空T引用，也可以不包含任何内容。一个不包含任何内容的optional被认为是空的。一个值存在optional里面就可以认为是非空的（A value is said to be present in an optional that is not empty.）【大概、可能、也许、应该是这个意思吧】。一个optional本质上是一个可容纳最多一个元素的不可变的集合。Optional<T>不是实现Collection<T>【接口】，但原则上是可以的。

&emsp;&emsp;在某些情况下，概念上【应该】返回T，但是可能无法执行此操作【返回T】的方法可以声明为返回Optional<T>。这允许该方法返回空结果来表示它无法返回有效的结果。返回Optional的方法比抛出异常的方法更灵活，更易于使用，并且比返回null的方法更不容易出错。

&emsp;&emsp;在第30项中，我们展示了这种方法，根据元素的自然顺序计算集合中的最大值。

```java
// Returns maximum value in collection - throws exception if empty
public static <E extends Comparable<E>> E max(Collection<E> c) {
    if (c.isEmpty())
        throw new IllegalArgumentException("Empty collection");
    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
            result = Objects.requireNonNull(e);
    return result;
}
```

&emsp;&emsp;如果给定集合为空，则此方法抛出IllegalArgumentException。【这是】我们在第30项中提到，更好的选择是返回Optional <E>。以下是修改它之后的外观：

```java
// Returns maximum value in collection as an Optional<E>
public static <E extends Comparable<E>> Optional<E> max(Collection<E> c) {
    if (c.isEmpty())
        return Optional.empty();

    E result = null;
    for (E e : c)
        if (result == null || e.compareTo(result) > 0)
    result = Objects.requireNonNull(e);
    return Optional.of(result);
}
```

&emsp;&emsp;如你所见，返回的optional很简单。你所要做的就是使用适当的静态工厂【方法】创建optional。在这个程序中，我们使用两个【静态工厂方法】：Optional.empty（）返回一个空的optional，Optional.of（value）返回一个包含给定的非null值的optional。将null传递给Optional.of（value）是一个编程错误。如果这样做，该方法会通过抛出NullPointerException来响应（responds）。Optional.ofNullable（value）方法接受一个可能为null的值，如果传入null则返回一个空的optional。永远不要从返回Optional的方法返回一个空值：它会破坏工具（facility）的整个意义。

&emsp;&emsp;流上的很多终端操作返回optional。如果我们重写max方法以使用流，Stream的max操作会为我们生成一个optional（尽管我们必须传入一个显式比较器）：

```java
// Returns max val in collection as Optional<E> - uses stream
public static <E extends Comparable<E>> Optional<E> max(Collection<E> c) {
    return c.stream().max(Comparator.naturalOrder());
}
```

&emsp;&emsp;那么如何选择返回optional而不是返回null或抛出异常？**optional在精神(spirit)上与检查异常类似(Optionals are similar in spirit to checked exceptions)**（第71项）。因为它们迫使【使用】API的用户面对可能没有返回值的事实。抛出未经检查的异常或返回null允许用户忽略此可能性，并可能产生可怕的后果。但是，抛出已检查的异常需要在客户端中添加额外的样板代码。

&emsp;&emsp;如果方法返回一个optional，则客户端可以选择在方法无法返回值时要采取的操作。 你可以指定默认值：

```java
// Using an optional to provide a chosen default value
String lastWordInLexicon = max(words).orElse("No words...");
```

&emsp;&emsp;或者你可以抛出任何适当的异常。请注意，我们传入异常工厂而不是实际异常。除非实际抛出异常，要不然这么做可以避免创建异常的开销：

```java
// Using an optional to throw a chosen exception
Toy myToy = max(toys).orElseThrow(TemperTantrumException::new);
```

&emsp;&emsp;如果你可以证明一个optional是非空的，那么你可以从Optional中获取值，而不【需要】指定当optional是空的时要采取的操作，但是如果你错了，你的代码将抛出NoSuchElementException：

```java
// Using optional when you know there’s a return value
Element lastNobleGas = max(Elements.NOBLE_GASES).get();
```

&emsp;&emsp;有时你可能会遇到这样的情况，即获取默认值【的成本】很昂贵，并且除非有必要【获取默认值】，否则你希望避免这种成本【的开销】。对于这些情况，Optional提供了一种方法，该方法接受Supplier<T>【参数】并仅在必要时【才会】调用它。这个方法叫做orElseGet，但也许它应该被称为orElseCompute，因为它与名称以compute开头的三个Map方法密切相关。有几种可选方法可用于处理更专业的情况：filter，map，flatMap和ifPresent。在Java 9中，添加了另外两个方法：or和ifPresentOrElse。如果上述的基本方法与你的情况不匹配，请查看这些更高级方法的文档，看看它们是否能完成这项工作。

&emsp;&emsp;如果这些方法都不符合你的需求，Optional提供了isPresent（）方法，可以将其视为安全阀。如果Optional包含值，则返回true；如果为空，则返回false。你可以使用此方法在optional的结果上执行你喜欢的任何处理，但请确保明智地使用它。isPresent的许多用途可以很好地被上面提到的方法之一取代。生成的代码通常更短，更清晰，更符合语言习惯（idiomatic）。

&emsp;&emsp;例如，请考虑此代码段，它打印进程父进程的进程ID，如果进程没有父进程，则为N / A. 该代码段使用Java 9中引入的ProcessHandle类：

```java
Optional<ProcessHandle> parentProcess = ph.parent();
System.out.println("Parent PID: " + (parentProcess.isPresent() ? String.valueOf(parentProcess.get().pid()) : "N/A"));
```

&emsp;&emsp;上面的代码片段可以替换为使用Optional的map函数的代码片段：

```java
System.out.println("Parent PID: " + ph.parent().map(h -> String.valueOf(h.pid())).orElse("N/A"));
```

&emsp;&emsp;使用流进行编程时，发现自己使用Stream<Optional<T>>并要求包含非空选项中的所有元素的Stream<T>可以继续执行【代码】【也就是说当你要求返回的所有Stream<Optional<T>>里面，如果Optional里面是有值的，那就继续执行代码，没有值的话就忽略】，这种情况是很常见的。如果你正在使用Java 8，那么下面是【教你】如何弥补【这种】差距【就是教你如何写出高逼格的代码】：

```java
streamOfOptionals.filter(Optional::isPresent).map(Optional::get)
```

&emsp;&emsp;在Java 9中，Optional配备了stream（）方法。此方法是一个适配器，它将Optional变为包含元素的Stream（如果在optional中存在，或者如果它为空则为none）。结合Stream的flatMap方法（第45项），此方法为上面的代码片段提供了简洁的替代【实现】：

```java
streamOfOptionals.flatMap(Optional::stream)
```

&emsp;&emsp;并非所有返回类型都受益于optional的【这种】处理【方式】。**容器类型，包括collections，map，stream，array和optional都不应该包含在optional里面**【就是这些类型的值都不应该作为optional里面的值】。你应该只返回一个空的List <T>（第54项），而不是返回一个空的Optional<List<T>>。返回空的容器，客户端可以去掉需要处理optional的代码。ProcessHandle类确实是一个有争议（arguments）的方法，它返回Optional<String[]>，但是这个方法应该被视为一个不能被效仿的反常现象（anomaly）。

&emsp;&emsp;那么什么时候应该声明一个方法来返回Optional <T>而不是T？通常，**通常，如果一个方法可能无法返回结果，则应该被声明为返回Optional<\T>，如果未返回结果，则客户端必须执行特殊处理** 。 也就是说，返回Optional<T>并非没有成本。Optional是必须分配【空间】和初始化的对象，从optional中读取值需要额外的间接【处理】。这使得optional不适合在某些有性能限制的情况下使用。【那些】特殊的方法是否属于这种范畴只能通过仔细的测试才能确定（第67项）。

&emsp;&emsp;包含自动装箱的基本类型的optional跟返回基本类型的optional相比，【返回自动装箱的基本类型】的成本是非常昂贵的，因为optional有两个级别的装箱操作，而不是零个没有【意思是optional有自己的装箱操作？】。因此，库设计人员认为适合为基本类型int，long和double提供Optional <T>的类似物（analogues）。这些可选类型是OptionalInt，OptionalLong和OptionalDouble。它们包含Optional<T>中大多数的方法（并非全部）。因此，**你永远不应该返回一个自动装箱的基本类型** ，除了“较次的基本类型（minor primitive types）”之外，还有Boolean，Byte，Character，Short，和Float。

&emsp;&emsp;到目前为止，我们已经讨论了返回optional并在返回后处理它们。我们还没有讨论其他可能的用途，这是因为optional其他的大多数使用方式都是可疑的。例如，你不应该使用optional作为map的值。如果这样做，你有两种方法可以从map中表达key的逻辑缺失：要么key可以不在map中，要么它可以存在并映射到空的optional。这表明了不必要的复杂性，具有很大的混淆和错误的可能性。更一般地说，在集合或数组中使用optional作为键，值或元素几乎都是不合适的。

&emsp;&emsp;这留下了一个无法回答的大问题。 是否适合在实例字段中存储optional？通常它是一种“坏味道”：它表明你可能应该有一个包含optional字段的子类。但有时它可能是合理的。考虑第2项中我们的NutritionFacts类的情况。一个NutritionFacts实例包含许多不需要的字段。你不可能拥有一种子类，子类的包含这些字段每种可能的组合。此外，这些字段包括基本类型，这使得直接表达【字段】不存在变得很棘手（You can’t have a subclass for every possible combination of these fields. Also, the fields have primitive types, which make it awkward to express absence directly）。NutritionFacts的最佳API是将为每个可选字段从getter返回一个optional，因此将这些optional作为字段存储在对象中是很有意义的。

&emsp;&emsp;总之，如果你发现自己编写的方法无法始终返回值，并且你认为【使用该】方法的用户每次调用它时都考虑这种可能性，那么你应该返回一个optional。但是，你应该意识到返回选optional会真正产生性能影响；对于对性能有要求的方法（for performance-critical methods），最好返回null或抛出异常。最后，除了作为返回值，你应该最大限度的不在其他地方使用optional（you should rarely use an optional in any other capacity than as a return value.）。