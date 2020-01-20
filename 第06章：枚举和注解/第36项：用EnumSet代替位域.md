## 用 EnumSet 代替位域

&emsp;&emsp;如果一个枚举的元素主要用在集合中，一般就使用 int 枚举模式（第 34 项），将 2 的不同倍数赋予每个常量：

```java
// Bit field enumeration constants - OBSOLETE!
public class Text {
    public static final int STYLE_BOLD = 1 << 0; // 1
    public static final int STYLE_ITALIC = 1 << 1; // 2
    public static final int STYLE_UNDERLINE = 1 << 2; // 4
    public static final int STYLE_STRIKETHROUGH = 1 << 3; // 8
    // Parameter is bitwise OR of zero or more STYLE_ constants
    public void applyStyles(int styles) { ... }
}
```

&emsp;&emsp;这种表示法让你用'或'运算将几个常量合并到一个集合中，称作位域（bit field）：

```java
    text.applyStyles(STYLE_BOLD | STYLE_ITALIC);
```

&emsp;&emsp;位域表示法也允许利用位操作，有效地执行像 nuion（并集）和 intersection（交集）这样的集合操作。但位域有着 int 枚举常量的所有缺点，甚至更多。当位域以数字形式打印时，翻译位域比翻译简单的 int 枚举常量要困难得多。甚至，要遍历位域表示的所有元素也没有很容易的办法。最后，在你编写 API 的时候就需要预测需要的最大位数，并选择相应的位域（通常为 int 或 long）类型。一旦选择了类型，在不修改 API 的情况下就不能超过其宽度（32 或 64 位)。

&emsp;&emsp;有些程序猿优先使用枚举而非 int 常量，他们在需要传递多组常量集时，仍然倾向于使用位域，其实没理由这么做，因为还有更好的替代方法。java.util 包提供了 EnumSet 类来有效地表示从单个枚举类型中提取的多个值的多个集合。这个类实现 Set 接口，提供了丰富的功能、类型安全性，以及可以从任何其他 Set 实现中得到的互用性。但是在内部的具体实现上，每个 EnumSet 内容都表示为位矢量。如果底层枚举类型有 64 个或者更少的元素————大多如此————整个 EnumSet 就是用单个 long 来表示，因此它的性能比得上位域的性能。批处理，如 removeAll 和 retainAll，都是利用位算法来实现的，就像手动替位域实现的那样。但是可以避免手动操作位域时容易出现的错误以及不太雅观的代码，因为 EnumSet 替你完成了这项艰巨的工作。

&emsp;&emsp;下面是前一个范例改成用枚举代替位域后的代码，它更加简短、更加清楚，也更加安全：

```java
// EnumSet - a modern replacement for bit fields
public class Text {
    public enum Style { BOLD, ITALIC, UNDERLINE, STRIKETHROUGH }
    // Any Set could be passed in, but EnumSet is clearly best
    public void applyStyles(Set<Style> styles) { ... }
}
```

&emsp;&emsp;下面是将 EnumSet 实例传递给 applyStyles 方法的客户端代码。EnumSet 提供了丰富的静态工厂来轻松创建集合，其中一个如下代码所示：

```java
text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
```

&emsp;&emsp;注意 applyStyles 方法采用的是 Set\<Style\>而非 EnumSet\<Style\>。虽然看起来好像所有的客户端都可以将 EnumSet 传到这个方法，但是最好还是接受接口类型而非接受实现类型。这是考虑到可能会有特殊的客户端要传递一些其他的 Set 实现（第 64 项）。并且没有什么明显的缺点。

&emsp;&emsp;总而言之，**正是因为枚举类型要用在几个（Set）中，所以没有理由用位域来表示它**。EnumSet 类集位域的简洁和性能优势以及第 34 项中所述的枚举类型的所有优点于一身。EnumSet 的一个真正的缺点是，从 Java 9 开始，它不可能创建一个不可变的 EnumSet，但这可能会在即将发布的版本中得到补救。在此期间，您可以使用 Collections.unmodifiableSet 包装 EnumSet，但简洁性和性能将受到影响。

> - [第 35 项：用实例域代替序数](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第06章：枚举和注解/第35项：用实例域代替序数.md)
> - [第 37 项：用 EnumMap 代替序数索引](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第06章：枚举和注解/第37项：用EnumMap代替序数索引.md)
