## 当重写 equals 方法时总要重写 hashCode 方法

&emsp;&emsp;**在每个重写了 equals 方法的类中，也必须重写 hashCode 方法。** 如果你没有这么做的话，就会违反 Object.hashCode 的通用约定，这将让它在 HashMap 和 HashSet 等集合中无法正常运作。这里是约定的内容，摘自 Onject 规范：

- 在应用程序的执行期间，只要对象的 equals 方法的比较操作所用到的信息没有被修改，那么对这同一个对象调用多次，hashCode 方法都必须始终如一地返回相同的值。在不同的程序执行时，该值不需要保持不变（This value need not remain consistent from one execution of an application to another）。

- 如果两个对象调用 equals（Object）方法比较是相等的，那么调用这两个对象中任意一个对象的 hashCode 方法都必须产生相同的整数结果。

- 如果两个对象根据 equals（Object）方法比较是不相等的，那么调用这两个对象中任意一个对象的 hashCode 方法，则不一定要产生不同的结果。无论如何，程序猿应该知道，不相等的对象产生截然不同的结果，有可能提高散列表（hash tables）的性能。

&emsp;&emsp;因没有覆盖 hashCode 而违反第二条的关键约定：相等的对象必须具有相等的散列码（hash code）。根据类的 equals 方法，两个截然不同的实例在逻辑上有可能是相等的，但是，根据 Object 的 hashCode 方法，它们仅仅是两个没有任何共同之处的对象。因此，Object 的 hashCode 方法返回两个看起来像是随机的整数，而不是根据通用约定所要求的那样，返回两个相等的整数。

&emsp;&emsp;例如，假设你视图使用 PhoneNumber 类（第 10 项）的实例作为一个 HashMap 的 key：

```java
Map<PhoneNumber, String> m = new HashMap<>();
m.put(new PhoneNumber(707, 867, 5309), "Jenny");
```

&emsp;&emsp;这时候，你可能期望`m.get(new PhoneNumber(707, 867, 5309))`会返回"Jenny"，但是相反的，它返回的是 null。注意，这里涉及到了两个 PhoneNumber 实例：第一个被用于插入到 HashMap 中，第二个实例与第一个实例相等【逻辑相等】，被用于（试图用于）获取【hashMap 的 value】。由于 PhoneNumber 类没有重写 hashCode 方法，从而导致两个相等的实例具有不相等的散列码，违反了 hashCode 约定。因此，put 方法把电话号码对象存放在一个散列桶（hash bucket）中，get 方法可能从另一个散列桶中查找这个号码。即使这两实例正好被放到同一个散列桶中，get 方法也必定会返回 null，因为 HashMap 有一项优化，可以将与每个项（entry）相关联的散列码缓存起来，如果散列码不匹配，也不必检验对象的等同性。

&emsp;&emsp;修正这个问题非常简单，只需要为 PhoneNumber 类提供一个适当的 hashCode 方法即可。那么，hashCode 方法应该是什么样的呢？编写一个差的方法是没有任何价值的，比如下面这个方法，它总是合法的，但是永远都不应该被正式使用：

```java
// The worst possible legal hashCode implementation - never use!
@Override public int hashCode() { return 42; }
```

&emsp;&emsp;上面这个 hashCode 方法是合法的，因为它确保了相等的对象总是具有同样的散列码。但它也极为恶劣，因为它使得每个对象都具有相同的散列码。因此，每个对象都被映射到同一个散列桶中，使散列表退化为链表（linked list）。它使得本该线性时间运行的程序变成了以平方级时间在运行【时间复杂度从 log(n)变成了 log(n^2) 】。对于规模很大的散列表而言，这会关系到散列表能否正常工作。

&emsp;&emsp;一个好的散列函数通常倾向于“为不相等的对象产生不相等的散列码”。这正是 hashCode 约定中第三条的含义。理想情况下，散列函数应该把集合中不相等的实例均匀地分布到所有可能的散列值上。要想完全达到这种理想的情形是非常困难的。幸运的是，相对接近这种理想情形并不太困难。下面给出一种简单的解决办法：

1. 声明一个名为 result 的 int 变量，并将其初始化为对象中第一个重要字段的哈希码 c，如步骤 2.a 中计算的那样。 （回顾第 10 项，重要字段是影响 equals 比较的字段。）

2. 对于对象中每个剩余的重要字段 f，请执行以下操作：

&emsp;&emsp;a. 为该字段计算 int 类型的散列码 c：

&emsp;&emsp;&emsp;&emsp;i.如果该字段是基本类型，则计算`Type.hashCode(f)`，其中 Type 是与 f 的类型对应的基本类型的封装。

&emsp;&emsp;&emsp;&emsp;ii.如果该字段是一个对象引用，并且该类的 equals 方法通过递归地调用 equals 的方式来比较这个字段，则同样为这个字段递归地调用 hashCode。如果需要更复杂的比较，则为这个字段计算一个“范式（canonical representation）”，然后针对这个范式调用 hashCode。如果这个字段的值是 null，则使用 0（或者其他某个数，但是通常是 0）。

&emsp;&emsp;&emsp;&emsp;vii.如果该字段是一个数组，则要把每一个元素当做单独的字段来处理。也就是说，递归地应用上述规则，对每个重要的元素计算一个散列码，然后根据步骤 2.b 中的做法把这些散列值组合起来。如果数组字段中的每个元素都很重要，可以利用 Arrays.hashCode 方法。

&emsp;&emsp;b. 按照下面的公式，把步骤 2.a 中计算得到的散列码 c 合并到 result 中：

```java
result = 31 * result + c;
```

3. 返回 result。

&emsp;&emsp;当你写完 hashCode 方法之后。问问自己“相等的实例是否都具有相等的散列码”。编写单元测试来验证你的推断（除非你使用 AutoValue 来自动生成你的 equals 和 hashCode 方法，在这种情况下您可以安全地省略这些测试）。如果相等的实例有着不相等的散列码，则要找出原因，并修正错误。

&emsp;&emsp;在散列码的计算中你可能会把派生字段（derived fields）排除在外，换句话说，如果一个字段的值可以根据参与计算的其他字段值计算出来，则可以把这样的字段排除在外。您必须排除未在 equals 比较计算中没有使用到的任何字段，否则您可能会违反 hashCode 约定的第二条。

&emsp;&emsp;步骤 2.b 中的乘法部分使得散列值依赖于字段的顺序，如果一个类包含多个相似的字段，这样的乘法运算就会产生一个更好地散列函数。例如，如果 String 散列函数省略了这个乘法部分，那么只是字母顺序不同的所有字符串都会有相同的散列码。之所以选择 31，是因为它是一个奇素数。如果乘数是偶数，并且乘法溢出的话，信息就会丢失，因为与 2 想成等价于移位运算。使用素数的好处并不是很明显，但这是传统（The advantage of using a prime is less clear, but it is traditional）【？？？】。31 的一个很好的特性是乘法可以用移位和减法代替，以便在某些架构上获得更好的性能：`31 * i == (i << 5) - i`。现代的 VM 可以自动完成这种优化。

&emsp;&emsp;现在我们要把上诉的步骤应用在 PhoneNumber 类中：

```java
// Typical hashCode method
@Override public int hashCode() {
    int result = Short.hashCode(areaCode);
    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);
    return result;
}
```

&emsp;&emsp;因为这个方法返回的结果是一个简单的、确定的计算结果，它的输入只是 PhoneNumber 实例中的三个关键字段，因此相等的 PhoneNumber 显然都会有相等的散列码。事实上，这种方法对于 PhoneNumber 来说是一个非常好的 hashCode 实现，与 Java 平台库中的方法相同。它很简单，速度相当快，并且可以合理地将不相等的电话号码分散到不同的散列桶中。

&emsp;&emsp;虽然该项中的方法产生了相当好的散列函数，但它们并不是最先进的。 它们的质量与 Java 平台库的值类型中的散列函数相当，并且适用于大多数用途。 如果您想了解怎么样才能做到哈希函数产生的结果很少出现冲突的情况，请参阅 Guava 的 com.google.common.hash.Hashing [Guava]（If you have a bona fide need for hash functions less likely to produce collisions, see Guava’s com.google.common.hash.Hashing [Guava]）。

&emsp;&emsp;Objects 类有一个静态方法，它接受任意数量的对象并为它们返回一个哈希码。 此方法名为 hash，允许您编写单行 hashCode 方法，其质量与根据这项中的编写方法相当。不幸的是，它们运行得更慢，因为它们需要创建数组来传递可变数量的参数，以及如果任何参数是基本类型的装箱和拆箱。 建议仅在性能要求不重要的情况下使用此样式的哈希函数。 这是使用该方法编写的 PhoneNumber 的哈希函数：

```java
// One-line hashCode method - mediocre performance
@Override public int hashCode() {
    return Objects.hash(lineNum, prefix, areaCode);
}
```

&emsp;&emsp;如果类是不可变的并且计算哈希码的成本很高，则可以考虑在对象中缓存哈希码，而不是在每次请求时重新计算哈希码。 如果您认为某类型的大多数对象将用作哈希键（If you believe that most objects of this type will be used as hash keys），则应在创建实例时计算哈希码。 否则，您可能会在第一次调用 hashCode 时选择懒加载（lazily initialized）哈希码。 需要注意确保在存在延迟初始化字段的情况下该类保持线程安全（第 83 项）。现在就是这样，我们的 PhoneNumber 类不值得做这种处理，只是为了向您展示它是如何完成的。请注意，hashCode 字段的初始值（在本例中为 0）不应该是常用创建实例的哈希码（should not be the hash code of a commonly created instance）：

```java
// hashCode method with lazily initialized cached hash code
private int hashCode; // Automatically initialized to 0
@Override public int hashCode() {
    int result = hashCode;
    if (result == 0) {
        result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        hashCode = result;
    }
    return result;
}
```

&emsp;&emsp;不要试图从散列码计算中排除掉一个对象的关键部分来提高性能。虽然这样得到的散列函数运行起来可能更快，但是它的效果不见得会好，可能会导致散列表慢到根本无法使用。特别是，哈希函数可能面临大量实例，这些实例主要区别于您选择忽略的区域。 如果发生这种情况，哈希函数会将所有的这些实例映射到某些哈希码，该程序本应该以线性时间运行，但是它会以平方级的运行时间【复杂度从 log(n)变成了 log(n^2) 】。

&emsp;&emsp;这不仅仅是一个理论上的问题。在 Java 2 之前，String 散列函数最多使用 16 个字符，这些字符在整个字符串中均匀分布，从第一个字符开始。 对于大型分层名称集合（例如 URL），此函数完全表现了前面提到的病态行为。

&emsp;&emsp;**不要为 hashCode 返回的值提供详细的规范，这样客户端就不会合理地依赖它; 这使您可以灵活地进行更改。** Java 库中的许多类（如 String 和 Integer）将 hashCode 方法返回的确切值指定为实例值的函数。这不是一个好主意，而是一个我们不得不忍受的错误：它阻碍了在未来版本中改进哈希函数的能力。如果未规定散列函数的细节，那么当你在散列函数中发现缺陷或发现更好的散列函数时，则可以在后续版本中更改它。

&emsp;&emsp;总之，每次覆盖 equals 时都必须覆盖 hashCode，否则程序将无法正常运行。 您的 hashCode 方法必须遵守 Object 中指定的规定，并且必须合理地将不相等的哈希代码分配给不相等的实例。如果觉得有点单调乏味，就使用第 51 页【原书本】的方式，这很容易实现。如第 10 项所述，AutoValue 框架提供了手动编写 equals 和 hashCode 方法的详细替代方案，IDE 也提供了一些此功能。

> - [第 10 项：覆盖 equals 时请遵守通用约定](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第03章：对于所有对象都通用的方法/重写equals时请遵守通用约定.md)
> - [第 12 项：始终重写 toString 方法](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第03章：对于所有对象都通用的方法/第12项：始终重写toString方法.md)
