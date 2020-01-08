## 为所有导出的 API 元素编写文档注释

&emsp;&emsp;如果要想使一个 API 真正可用，就必须为其编写文档。传统意义上的 API 文档是手工生成的，所以保持文档与代码同步是一件很繁琐的事情。Java 编程环境提供了一种被成为 Javadoc 的实用工具。Javadoc 使用特殊格式的*文档注释（documentation comments）*（通常称为*doc 注释（doc comments）*）从源代码自动生成 API 文档。

&emsp;&emsp;虽然这些文档注释规范不是 Java 正式语言的一部分，但它们已经构成了每个程序猿都应该知道的 API。这些规范在*关于如何编写文档注释（How to Write Doc Comments）*的网站上进行了说明\[Javadocguide\]。虽然这个网站在 Java 4 发行之后就没有再进行更新了，但它仍然是个很有价值的资源。Java 9 中添加了一个重要的 doc 标签，{@index}; Java 8 中添加了一个，{@implSpec}; Java 5 中添加了两个，{@literal}和{@code}。上述网站中没有这些标签，但在此项中进行了【相关的】讨论。

&emsp;&emsp;**为了正确地编写 API 文档，必须在每个被导出的类、接口、构造器、方法和域声明之前增加一个文档注释。** 如果类是可序列化的，也应该对它的序列化形式编写文档（第 87 项）。在没有文档注释的情况下，Javadoc 所能够做的也就是重新生成该声明，作为受影响的 API 元素的唯一文档。使用没有文档注释的 API 是非常痛苦的，也很容易出错。公共类不应使用默认构造函数，因为无法为它们提供文档注释。为了编写可维护的代码，你还应该为大多数未导出的类，接口，构造函数，方法和字段编写文档注释，尽管这些注释不需要像导出的 API 元素那样详细。

&emsp;&emsp;**方法的文档注释应该简洁地描述它和客户端之间的约定。** 除了专门为了继承而设计的类中的方法（第 19 项）之外，这个约定应该说明这个方法做了什么，而不是说明它是如何完成这项工作的。文档注释应该列举出这个方法的所有*前提条件（preconditions）*，这些条件是客户端调用它的必要条件，以及它的*后置条件（postconditions）*，这些条件是调用成功后会成立的事情【就是调用成功后会发生什么事情】。一般情况下，前提条件是由@throw 标签针对未受检的异常所隐含描述的；每个未受检的异常都对应一个违背前提条件的例子（precondition violation）。同样的，也可以在一些受影响的参数的@Param 标记中指定前提条件。

&emsp;&emsp;除了前提条件和后置条件之外，每个方法还应该在文档中描述它的*副作用（side effects）*。所谓副作用是指系统状态中可以观察到的变化，它不是为了获得后置条件而明确要求的变化。例如，如果方法启动了后台线程，文档中就应该说明这一点。

&emsp;&emsp;为了完整地描述方法的约定，文档注释应该为每个参数都使用一个@Param 标记，方法使用@return 标记，除非方法的返回类型是 void，以及对于该方法抛出的每个异常，无论是受检的还是未受检的，都有一个@throws 标签（第 74 项）。如果@return 标签的文本内容和方法的描述相同，则可以省略它，具体取决于你遵循的编码标准。

&emsp;&emsp;按照惯例，跟在@param 标签或者@return 标签后面的文字应该是一个名词短语，描述了这个参数或者返回值所表示的值。很少使用算术表达式代替名词短语；请参阅 BigInteger 的示例。跟在@throw 标签之后的文字应该包含单词“if”（如果），后面跟着一个子句（clause），它描述了这个异常将在什么样的条件下会被抛出。按照惯例，@param，@return 或@throws 标签之后的短语或子句不会以句号为结尾。下面这个文档注释演示了所有这些习惯的做法：

```java
/**
* Returns the element at the specified position in this list.
*
* <p>This method is <i>not</i> guaranteed to run in constant
* time. In some implementations it may run in time proportional
* to the element position.
*
* @param index index of element to return; must be
*        non-negative and less than the size of this list
* @return the element at the specified position in this list
* @throws IndexOutOfBoundsException if the index is out of range
*         ({@code index < 0 || index >= this.size()})
*/
E get(int index);
```

&emsp;&emsp;注意，这份文档注释中使用了 HTML 标签（\<P>和\<i>）。Javadoc 工具会把文档注释翻译成 HTML，文档注释中包含的任意 HTML 元素都会出现在生成的 HTML 文档中。有时候，程序猿会把 HTML 表格嵌入到它们的文档注释中，但是这种做法并不多见。

&emsp;&emsp;还要注意，@throws 子句的代码片段中到处使用了 Javadoc 的{@code}标签。它有两个作用：造成该代码片段以代码字体(code font)进行呈现，并限制 HTML 标记和嵌套的 Javadoc 标签在代码片段中进行处理。后一种属性证正是允许我们在代码片段中使用小于号(<)的东西，虽然它是一个 HTML 元字符。要在文档注释中包含多行代码示例，请使用包含在 HTML\<pre>标记内的 Javadoc {@code}标记。换句话说，在代码示例之前加上字符\<pre>{@code 代码跟在这后面}\<pre>。这样可以保留代码中的换行符，并且不需要转义 HTML 元字符，而不需要转义符号（@），如果代码示例使用注释，则必须对其进行转义。

&emsp;&emsp;最后，要注意在文档中使用“this”，按照惯例，当“this”被用在实例方法的文档注释中时，它应该始终是指调用方法的对象。

&emsp;&emsp;如第 15 项所述，当你设计一个继承类时，你必须记录它的*自用模式（self-use patterns）*，因此程序员知道覆盖其方法的含义。应使用 Java 8 中添加的@implSpec 标记记录这些自用模式。回想一下，普通的文档注释描述了方法与其客户端之间的契约; 相反，@implSpec 注释描述了方法及其子类之间的约束，如果子类继承方法或通过 super 调用它，则允许子类依赖于实现的行为。这是它在实践中的表现：

```java
/**
 * Returns true if this collection is empty.
 *
 * @implSpec
 * This implementation returns {@code this.size() == 0}.
 *
 * @return true if this collection is empty
 */
public boolean isEmpty() { ... }
```

&emsp;&emsp;从 Java 9 开始，除非你在命令行中打开【配置】-tag“implSpec：a：Implementation Requirements：”，否则 Javadoc 工具仍会忽略@implSpec 标签。希望这将在随后的版本中得到补救。

&emsp;&emsp;不要忘记，为了产生包含 HTML 元字符的文档，比如小于号(<)、大于号(>)、以及“与”号(&)。让这些字符出现在文档中的最佳办法是用{@literal}标签将它们包围起来，这样就限制了 HTML 标记和嵌套的 Javadoc 标签的处理。除了它不以代码字体渲染文本之外，其余方面就像{@code}标签一样。例如，这个 Javadoc 片段：

```
* A geometric series converges if {@literal |r| < 1}.
```

&emsp;&emsp;产生了这样的文档：““A geometric series converges if |r| < 1.”{@literal}标签也可以只是包住小于号，而不是整个不等式，所产生的文档是一样的，但是在源代码中见到的文档注释的可读性就会更差。这说明了一条通则：**文档注释在源代码和产生的文档中都应该是易于阅读的。** 如果无法做到这两点，产生的文档的可读性要优先于源代码的可读性。

&emsp;&emsp;每个文档注释的第一句话（如下所示）成了该注释所属元素的*概要描述（summary description）*。例如，第 255 页【原书】中文档注释中的概要描述为“返回这个列表中指定位置上的元素”。概要描述必须独立地描述目标元素的功能。为了避免混淆，**同一个类或者接口中的两个成员或者构造器，不应该具有同样的概要描述** 。特别要注意重载的情形，在这种情况下，往往很自然地在描述中使用同样的一句话（但在文档注释中这是不可接受的）。

&emsp;&emsp;注意所期待的描述中是否包含句点，因为句点会过早地终止这个描述。例如，一个以“A college degree, such as B.S., M.S. or Ph.D.”开头的文档注释，会产生这样的概要描述：“A college degree, such as B.S., M.S.”问题在于，概要描述在后面接着空格、跳格或者行终结符的第一个句点处（或者在第一块标签处）结束\[Javadoc-ref\]。在这种情况下，缩写“M.S”中的第二个句点就要接着用一个空格。最好的解决办法是，将惹麻烦的句点以及任何与{@literal}关联的文本都包起来，因此在源代码中，句点后面就不再是空格了：

```java
/**
 * A college degree, such as B.S., {@literal M.S.} or Ph.D.
 */
public class Degree { ... }
```

&emsp;&emsp;说概要描述是文档注释中的第一个句子（sentence），这似乎有点误导人。规范指出，概要描述很少是个完整的句子。对于方法和构造器而言，概要描述应该是个完整的动词短语（包含任何对象），它描述了该方法所执行的动作。例如：

- ArrayList(int initialCapacity)—Constructs an empty list with the specified initial capacity. (用指定的初始化容量构造一个空的列表)
- Collection.size()—Returns the number of elements in this collection.(返回该集合中元素的数目)

&emsp;&emsp;如这些示例所示，使用第三人称声明（“returns the number”）而不是第二人称（“return the number”）。

&emsp;&emsp;对于类、接口和域，概要描述应该是一个名词短语，它描述了该类或者接口的实例，或者域本身所代表的事物。例如：

- Instant—An instantaneous point on the time-line.(时间线上的瞬时点。)
- Math.PI—The double value that is closer than any other to pi, the ratio of the circumference of a circle to its diameter.(比 pi 更接近 pi 的 double 值，即圆的周长与直径的比值。)

&emsp;&emsp;在 Java 9 中，客户端索引（client-side index）被添加到 Javadoc 生成的 HTML 中。该索引简化了导航大型 API 文档集的任务，采用页面右上角的搜索框形式。当你在框中输入内容时，你会看到一个匹配页面的下拉菜单。API 元素（例如类，方法和字段）会自动编入索引。有时，你可能希望对 API 很重要的其他术语进行索引。为此目的添加了{@index}标签。索引文档注释中出现的术语就像将其包装在此标记中一样简单，如此片段所示：

```
* This method complies with the {@index IEEE 754} standard.
```

&emsp;&emsp;泛型，枚举和注释需要特别注意文档注释。**记录泛型类型或方法时，请务必记录所有类型参数**：

```java
/**
 * An object that maps keys to values. A map cannot contain
 * duplicate keys; each key can map to at most one value.
 *
 * (Remainder omitted)
 *
 * @param <K> the type of keys maintained by this map
 * @param <V> the type of mapped values
 */
public interface Map<K, V> { ... }
```

&emsp;&emsp;**为枚举类型写文档时，请务必为常量** 以及类型和任何公共方法编写文档。请注意，如果【文档】简短【的话】，你可以将整个文档注释放在一行上：

```java
/**
 * An instrument section of a symphony orchestra.
 */
public enum OrchestraSection {
    /** Woodwinds, such as flute, clarinet, and oboe. */
    WOODWIND,
    /** Brass instruments, such as french horn and trumpet. */
    BRASS,
    /** Percussion instruments, such as timpani and cymbals. */
    PERCUSSION,
    /** Stringed instruments, such as violin and cello. */
    STRING;
}
```

&emsp;&emsp;**为注解类型编写文档时，要确保在文档中说明所有成员** ，以及类型本身。带有名词短语的文档成员，就像是域一样。对于该类型的概要描述，要使用一个动词短语，说明当程序元素具有这种类型的注解时它表示什么意思：

```java
/**
 * Indicates that the annotated method is a test method that
 * must throw the designated exception to pass.
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ExceptionTest {
    /**
    * The exception that the annotated test method must throw
    * in order to pass. (The test is permitted to throw any
    * subtype of the type described by this class object.)
    */
    Class<? extends Throwable> value();
}
```

&emsp;&emsp;包级私有的文档注释就应该放在一个称作 package-info.java 的文件中。除了这些注释之外，package-info.java 还必须包含一个包声明，并且可以在此声明中包含注释。同样，如果你选择使用模块系统（第 15 项），则应将模块级注释放在 module-info.java 文件中。

&emsp;&emsp;类的导出 API 有两个特征经常被人忽视，即线程安全性和可序列化性。**类是否是线程安全的，应该在文档中对它的线程安全级别进行声明** ，如 82 项所述。如果类是可序列化的，就应该在文档中说明它的序列化形式，如 87 项所述。

&emsp;&emsp;Javadoc 具有“继承”方法注释的能力。如果 API 元素没有文档注释，Javadoc 将会搜索最为适用的文档注释，接口的文档注释优先于超类的文档注释。搜索算法的细节可以在《The Javadoc Reference Guide》\[Javadoc-ref\]中找到。也可以利用{@inheritDoc}标签从超类型中继承文档注释的部分内容。这意味着，不说别的，类还可以重用它所实现的接口的文档注释，而不需要拷贝这些注释。这项机制有可能减轻维护多个几乎相同的文档注释的负担，但它使用起来比较需要一些小技巧（tricky），并具有一些局限性。关于这一点的详情超出了本书的范围，在此不做讨论。

&emsp;&emsp;关于文档注释，应该添加一个附加说明（caveat）。虽然有必要为所有导出的 API 元素提供文档注释，但这么做并非永远就足够了。对于由多个相互关联的类组成的复杂 API，通常需要使用描述 API 总体体系结构的额外文档来补充文档注释。如果存在此类文档，则相关的类或包文档注释应包含指向它的链接。Javadoc 会自动检查是否符合此项中的许多建议。 在 Java 7 中，需要在命令行打开【设置】-Xdoclint 才能获得此行为。在 Java 8 和 9 中，默认情况下启用检查。checkstyle 等 IDE 插件会进一步检查是否符合这些建议\[Burn01\]。你还可以通过 HTML 有效性检查器运行 Javadoc 生成的 HTML 文件，从而降低文档注释中出错的可能性。这将检测 HTML 标记的许多不正确的用法。有几个这样的检查器可供下载，你可以使用 W3C 标记验证服务\[W3C-validator\]在网站上验证 HTML。在验证生成的 HTML 时，请记住，从 Java 9 开始，Javadoc 能够生成 HTML5 以及 HTML 4.01，但默认情况下仍会生成 HTML 4.01。 如果希望 Javadoc 生成 HTML5，请在命令行使用-html5 命令【打开设置】。

&emsp;&emsp;本项中描述的约定涵盖了基本的惯例。虽然在撰写本文时已经是十五年前了，但撰写文档注释的权威指南仍然是《How to Write Doc Comments》\[Javadoc-guide\]。

&emsp;&emsp;如果你遵守此项目中的指南，生成的文档应该可以为你的 API 提供清晰的描述。但是，唯一可以确定的方法是**阅读 Javadoc 工具生成的网页** 。对于其他打算使用该 API 的人，这么做是值得的。正如测试程序几乎不可避免地导致代码发生一些变化一样，阅读文档通常会导致对文档注释进行至少一些小的更改。

&emsp;&emsp;总而言之，要为 API 编写文档，文档注释是最好、最有效的途径。对于所有可导出的 API 元素来说，使用文档注释应该被看作是强制性的。要采用一致的风格来遵循标准的约定。记住，在文档注释内部出现任何 HTML 标签都是允许的，但是 HTML 元字符必须要经过转义。

> - [第 55 项：谨慎返回 optinal](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第08章：方法/第55项：谨慎返回optional.md)
> - [第 57 项：将局部变量的作用域最小化](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第09章：通用编程/第57项：将局部变量的作用域最小化.md)
