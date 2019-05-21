### 用EnumSet代替位域

&emsp;&emsp;如果一个枚举的元素主要用在集合中，一般就使用int枚举模式（第34项），将2的不同倍数赋予每个常量：

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

&emsp;&emsp;这种表示法让你用OR位运算将几个常量合并到一个集合中，称作位域（bit field）：

```java
    text.applyStyles(STYLE_BOLD | STYLE_ITALIC);
```

&emsp;&emsp;位域表示法也允许利用位操作，有效地执行像nuion（并集）和intersection（交集）这样的集合操作。但位域有着int枚举常量的所有缺点，甚至更多。当位域以数字形式打印时，翻译位域比翻译简单的int枚举常量要困难得多。甚至，要遍历位域表示的所有元素也没有很容易的办法。最后，在你编写API的时候就需要预测需要的最大位数，并选择相应的位域（通常为int或long）的类型。一旦选择了类型，在不修改API的情况下就不能超过其宽度（32或64位)。

&emsp;&emsp;有些程序猿优先使用枚举而非int常量，他们在需要传递多组常量集时，仍然倾向于使用位域，其实没理由这么做，因为还有更好的替代方法。java.util包提供了EnumSet类来有效地表示从单个枚举类型中提取的多个值的多个集合。这个类实现Set接口，提供了丰富的功能、类型安全性，以及可以从任何其他Set实现中得到的互用性。但是在内部具体实现上，没个EnumSet内容都表示为位矢量。如果底层枚举类型有64个或者更少的元素————大多如此————整个EnumSet就是用单个long来表示，因此它的性能比得上位域的性能。批处理，如removeAll和retainAll，都是利用位算法来实现的，就像手动替位域实现的那样。但是可以避免手动操作位域时容易出现的错误以及不太雅观的代码，因为EnumSet替你完成了这项艰巨的工作。

&emsp;&emsp;下面是前一个范例改成用枚举代替位域后的代码，它更加简短、更加清楚，也更加安全：

```java
// EnumSet - a modern replacement for bit fields
public class Text {
    public enum Style { BOLD, ITALIC, UNDERLINE, STRIKETHROUGH }
    // Any Set could be passed in, but EnumSet is clearly best
    public void applyStyles(Set<Style> styles) { ... }
}
```

&emsp;&emsp;下面是将EnumSet实例传递给applyStyles方法的客户端代码。EnumSet提供了丰富的静态工厂来轻松创建集合，其中一个如下代码所示：

```java
text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
```

&emsp;&emsp;注意applyStyles方法采用的是Set<Style>而非EnumSet<Style>。虽然看起来好像所有的客户端都可以将EnumSet传到这个方法，但是最好还是接受接口类型而非接受实现类型。这是考虑到可能会有特殊的客户端要传递一些其他的Set实现（第64项）。并且没有什么明显的缺点。

&emsp;&emsp;总而言之，**正是因为枚举类型要用在几个（Set）中，所以没有理由用位域来表示它。** EnumSet类集位域的简洁和性能优势及第34项中所述的枚举类型的所有优点于一身。EnumSet的一个真正的缺点是，从Java 9开始，它不可能创建一个不可变的EnumSet，但这可能会在即将发布的版本中得到补救。在此期间，您可以使用Collections.unmodifiableSet包装EnumSet，但简洁性和性能将受到影响。