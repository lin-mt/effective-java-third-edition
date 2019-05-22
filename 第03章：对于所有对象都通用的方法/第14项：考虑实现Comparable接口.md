## 考虑实现Comparable接口

&emsp;&emsp;本章中讨论的其他方法不同，compareTo方法并没有在Object中声明。相反，它是Comparable接口中唯一的方法，它的特征与Object的equals方法类似，只是出了简单的equals比较之外，它还允许进行顺序比较，并且它是通用的。 通过实现Comparable，类表明它的实例具有自然顺序。实现Comparable的对象数组就像这样简单：

```java
Arrays.sort(a);
```

&emsp;&emsp;它同样易于搜索，计算极值，并维护自动排序的Comparable对象集合。例如，以下程序依赖于String实现Comparable接口，将其命令行参数按字母的顺序排列的列表打印出来，并删除重复项：

```java
public class WordList {
    public static void main(String[] args) {
        Set<String> s = new TreeSet<>();
        Collections.addAll(s, args);
        System.out.println(s);
    }
}
```

&emsp;&emsp;通过实现Comparable，您可以让您的类与依赖于此接口的所有许多通用算法（generic algorithm）和集合实现进行互操作。 只需少量工作就可以获得巨大的功能。实际上，Java平台库中的所有值类以及所有枚举类型（第34项）都实现了Comparable。 如果您正在编写具有明显自然顺序的值类，例如字母顺序，数字顺序或时间顺序，则应实现Comparable接口：

```java
public interface Comparable<T> {
    int compareTo(T t);
}
```

&emsp;&emsp;compareTo方法的通用约定与equals方法相似：

&emsp;&emsp;将这个对象与指定对象进行比较。当该对象小于、等于或大于指定对象的时候，分别返回一个负整数、零或者正整数。如果由于指定对象的类型而无法与该对象进行比较，则抛出ClassCastException异常。

&emsp;&emsp;在下面的说明中，符号sgn（表达式）表示数学中的signum函数，它根据表达式（expression）的值为负值、零和正值分别返回-1、0或1。

* 实现者必须确保所有的x和y都满足`sgn(x.compareTo(y)) == -sgn(y.compareTo(x))`。（这也暗示着，当且仅当`y.compareTo(x)`抛出异常时，`x.compareTo(y)`才必须抛出异常。）

* 实现者还必须确保这个比较关系是可传递的：`(x.compareTo(y) > 0 && y.compareTo(z) > 0)`暗示着`x.compareTo(z) > 0`。

* 最后，实现者必须确保`x.compareTo(y) == 0`暗示着所有的z都满足`sgn(x.compareTo(z)) == sgn(y.compareTo(z))`。

* 强烈建议`(x.compareTo(y) == 0) == (x.equals(y))`，但这并非绝对必要。一般来说，任何实现了Compareable接口的类，若违反了这个条件，都应该明确予以说明。推荐使用这样的说法：“注意：该类具有自然的排序功能，但是与equals不一致”

&emsp;&emsp;千万不要被上述约定中的数学关系所迷惑。如同equals约定（第10项），这个约定并没有它看起来那么复杂。与equals方法不同，equals方法在所有对象强加全局等价关系，compareTo不必跨越不同类型的对象：当遇到不同类型的对象时，compareTo被允许抛出ClassCastException。通常，这确实就是compareTo的做法。约定确实允许交互式比较，这通常在由被比较的对象实现的接口中定义（The contract does permit intertype comparisons, which are typically defined in an interface implemented by the objects being compared）。

&emsp;&emsp;就好像违反了hashCode约定的类会破坏其他依赖于哈希的类一样，违反compareTo约定的类也会破坏其他依赖于比较关系的类。依赖于比较关系的类包括有序集合TreeSet和TreeMap以及包含搜索和排序算法的使用程序集Collections和Arrays。

&emsp;&emsp;现在我们来回顾一下compareTo约定中的条款。我们来看看compareTo合同的规定。第一条规定说如果你反转两个对象引用之间的比较方向，就会发生预期的事情：如果第一个对象小于第二个对象，那么第二个对象必须大于第一个对象; 如果第一个对象等于第二个对象，那么第二个对象必须等于第一个对象; 如果第一个对象大于第二个对象，那么第二个对象必须小于第一个对象。第二条指出，如果一个对象大于第二个对象，并且第二个对象又大于第三个对象，那么第一个对象一定大于第三个对象。最后一条指出，在比较时被认为相等的所有对象，他们跟别的对象做比较时一定会产生同样的结果。

&emsp;&emsp;这三个规定的一个结果是，compareTo方法所施加的等同性测试（equality test），也一定遵守跟equals约定所施加的相同的限制条件：自反性、对称性和传递性。因此，下面的告诫也同样适用：除非你乐毅放弃面向对象抽象的优势（第10项），否则无法在用新的值组件扩展可实例化的类的同时保持compareTo约定。针对equals的解决方法也同样适用于compareTo方法。如果你相位一个实现了Comparable接口的类增加值组件，请不要扩展这个类；而是要编写一个不相关的类，期中包含第一个类的一个实例，然后提供一个“视图（view）”方法返回这个实例。这样就可以让你自由地在第二个类上实现compareTo方法，同时允许其客户端在必要的时候，把第二个类的实例视同第一个类的实例（This frees you to implement whatever compareTo method you like on the containing class, while allowing its client to view an instance of the containing class as an instance of the contained class when needed）。

&emsp;&emsp;compareTo约定的最后一段是一个强烈的建议，而不是真正的规则，只是说明了compareTo方法施加的等同性测试通常应该返回与equals方法相同的结果。如果遵守此约定，那么由compareTo方法所施加的顺序关系就被认为“与equals一致（consistent with equals）”。如果违反了这条规则，顺序关系就被认为“与equals不一致（inconsistent with equals）。”如果一个类的compareTo方法试驾了一个与equals方法不一致的顺序关系，它仍然能够正常工作，但是，如果一个有序集合（sorted collection）包含了该类的元素，这个集合就可能无法遵守相应集合接口（Colection、Set或Map）的通用约定。这是因为，对于这些接口的通用约定是按照equals方法来定义的，但是有序集合使用了由compareTo方法而不是equals方法所施加的等同性测试。尽管出现这种情况不会造成灾难性的后果，但是应该有所了解。

&emsp;&emsp;例如，考虑BigDecimal类，它的compareTo方法与equals不一致。如果你创建了一个HashSet实例，并且添加`new BigDecimal("1.0")`和`new BigDecimal("1.00")`，这个集合就将包含两个元素，因为新增到集合中的两个BigDecimal实例，通过equals方法来比较时是不相等的。然而，如果你使用TreeSet而不是HashSet来执行同样的过程，集合中将只包含一个元素，因为这两BigDecimal实例在通过使用compareTo方法进行比较时是相等的。（详情请看BigDecimal的文档。）

&emsp;&emsp;编写compareTo方法与编写equals方法相似，但是也存在几处重大差别。因为Comparable接口是参数化的，而且comparable方法是静态的类型，因此不必进行类型检查，也不必对它的参数进行类型转换。如果参数的类型是错的，这个调用甚至不会编译。如果参数是null，调用应该抛出NullPointerException，并且该方法试图访问它的成员时就应该抛出。

&emsp;&emsp;在compareTo方法中，字段比较主要是顺序比较，而不是相等性比较。比较对象引用字段可以是通过递归地调用compareTo方法来实现。如果一个字段没有实现Comparable接口，或者你需要使用一个非标准的排序关系，就可以使用Comparator来代替，你可以编写你自己的comparator或者使用已有的comparator，比如针对第10项中CaseInsensitiveString类的这个compareTo方法使用一个已有的comparator：

```java
// Single-field Comparable with object reference field
public final class CaseInsensitiveString implements Comparable<CaseInsensitiveString> {
    public int compareTo(CaseInsensitiveString cis) {
        return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
    }
    ... // Remainder omitted
}
```

&emsp;&emsp;注意CaseInsensitiveString类实现了Comparable<CaseInsensitiveString>接口。这意味着，CaseInsensitiveString引用只能与另一个CaseInsensitiveString引用进行比较。这是声明一个类来实现Comparable时要遵循的正常模式。

&emsp;&emsp;本书的先前版本建议compareTo方法使用关系运算符<和>比较整数基元【基本数据类型】类型字段，使用静态方法Double.compare和Float.compare比较浮点基元【基本数据类型】字段。在Java 7中，静态比较方法被添加到所有Java的基本数据类型的包装类中。**在compareTo方法中使用关系运算符< 和 >是冗长且容易出错的，不再推荐使用。**

&emsp;&emsp;如果一个类有多个重要字段，那么比较它们的顺序至关重要。从最重要的字段开始，然后往下工作【比较】。 如果比较产生的不是零（零代表相等），那么你就完成了; 并返回结果。 如果最重要的字段相等，则比较次最重要的字段，以此类推，直到找到不相等的字段或比较最不重要的字段。（If the most significant field is equal, compare the nextmost-significant field, and so on, until you find an unequal field or compare the least significant field.） 这是第11项中PhoneNumber类的compareTo方法，演示了这种方法：

```java
// Multiple-field Comparable with primitive fields
public int compareTo(PhoneNumber pn) {
    int result = Short.compare(areaCode, pn.areaCode);
    if (result == 0) {
        result = Short.compare(prefix, pn.prefix);
        if (result == 0)
            result = Short.compare(lineNum, pn.lineNum);
    }
    return result;
}
```

&emsp;&emsp;在Java 8中，Comparator接口配备了一组比较器构造方法，可以精确构建比较器。然后，可以使用这些比较器来实现compareTo方法，这是Comparable接口所要求的。许多程序猿更喜欢这种方法的简洁性，尽管它的性能成本很低：在我的机器上排序PhoneNumber实例的数组大约慢10%。使用这种方法时，考虑使用Java的静态导入功能，这样您就可以通过简单的名称来引用静态比较器构造方法，以获得清晰和简洁【代码】。这就是PhoneNumber的compareTo方法看起来是如何使用这种方法：

```java
// Comparable with comparator construction methods
private static final Comparator<PhoneNumber> COMPARATOR =
    comparingInt((PhoneNumber pn) -> pn.areaCode)
        .thenComparingInt(pn -> pn.prefix)
        .thenComparingInt(pn -> pn.lineNum);
public int compareTo(PhoneNumber pn) {
    return COMPARATOR.compare(this, pn);
}
```

&emsp;&emsp;此实现在类初始化时构建比较器，使用了两种比较器构造方法。第一种是comparingInt。它是一个静态方法，它使用了一个键提取器函数，它将对象引用映射到int类型的键，并返回一个比较器，该比较器根据该键对实例进行排序。在前面的示例中，comparisonInt采用lambda()从PhoneNumber中提取区域代码【areaCode】，并返回Comparator<PhoneNumber>，根据区号对电话号码进行排序。请注意，lambda显式指定其输入参数的类型（PhoneNumber pn）。事实证明，在这种情况下，Java的类型推断并不足以为自己确定类型，因此我们不得不帮助它来编译程序（It turns out that in this situation,Java’s type inference isn’t powerful enough to figure the type out for itself, so we’re forced to help it in order to make the program compile）。

&emsp;&emsp;如果两个电话号码具有相同的区号，我们需要进一步细化比较，这正是第二个比较器构造方法thenComparingInt所做的事情。它是Comparator上的一个实例方法，它接受一个int key提取器函数，并返回一个比较器，该比较器首先应用原始比较器，然后使用提取的键来断开关系（It is an instance method on Comparator that takes an int key extractor function, and returns a comparator that first applies the original comparator and then uses the extracted key to break ties）。 您可以根据需要将尽可能多的调用堆叠到thenComparingInt，从而产生字典顺序（You can stack up as many calls to thenComparingInt as you like, resulting in a lexicographic ordering）。在上面的示例中，我们将两个调用堆叠到thenComparingInt，从而产生一个排序，其二级密钥是前缀，其三级密钥是行号。请注意，我们没有必要指定传递给thenComparingInt的任一调用的键提取器函数的参数类型：Java的类型推断足够聪明，可以自己解决这个问题（In the example above, we stack up two calls to thenComparingInt,resulting in an ordering whose secondary key is the prefix and whose tertiary keyis the line number. Note that we did not have to specify the parameter type of the key extractor function passed to either of the calls to thenComparingInt: Java’s type inference was smart enough to figure this one out for itself）。

&emsp;&emsp;对于基本类型long和double，也有comparingInt和thenComparingInt类似的东西。Int版本也适用于【取值范围】比较窄的整数类型，例如short，就像我们的PhoneNumber实例所示。double版本也可用于float。这提供了所有Java的数字基本类型的覆盖。【也就是说double版本可以用于Java的所有数字的基本类型：double、float、int、short、long。byte】

&emsp;&emsp;对象引用类型也有比较器构造方法。一个有两个名为comparing的重载静态方法（The static method, named comparing, has two overloadings）。一个是需要key的提取器，并使用key的自然顺序。第二个采用key提取器和比较器来提取key（One takes a key extractor and uses the keys’ natural order. The second takes both a key extractor and a comparator to be used on the extracted keys）。这有三个名为thenComparing的重载的实例方法，第一个重载方法只使用比较器（comparator），并使用它来提供二级顺序【二级排序方式】。第二个重载方法只使用一个key提取器，并使用key的自然排序作为二级顺序。最后一个重载方法需要一个key提取器和一个用于比较提取的key的比较器（The final overloading takes both a key extractor and a comparator to be used on the extracted keys）。

&emsp;&emsp;有时，您可能会看到compareTo或compare方法，这些方法依赖于以下事实：如果第一个值小于第二个值，则两个值之间的差值为负，如果两个值相等则为零，如果第一个值更大则为正值。这是一个例子：

```java
// BROKEN difference-based comparator - violates transitivity!
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return o1.hashCode() - o2.hashCode();
    }
};
```

&emsp;&emsp;不要使用这种技巧。它充满了整数溢出和IEEE 754浮点运算伪像的危险[JLS 15.20.1,15.21.1]（It is fraught with danger from integer overflow and IEEE 754 floating point arithmetic artifacts [JLS 15.20.1, 15.21.1]）。此外，所得到的方法不太可能比使用项中描述的技术编写的方法快得多。使用静态比较方法：

```java
// Comparator based on static compare method
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return Integer.compare(o1.hashCode(), o2.hashCode());
    }
};
```

&emsp;&emsp;或者一个比较器的构造方法：

```java
// Comparator based on Comparator construction method
static Comparator<Object> hashCodeOrder = Comparator.comparingInt(o -> o.hashCode());
```

&emsp;&emsp;总之，无论何时实现具有合理排序的值类，都应该让类实现Comparable接口，以便可以在基于比较的集合中轻松地对其实例进行排序，搜索和使用。compareTo方法的实现中在比较字段的值的时侯，请避免使用 < 和 > 运算符。 而是使用基本类型的包装类中的静态比较方法或比较器接口中的比较器构造方法。