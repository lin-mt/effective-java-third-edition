## Lambda 优先于匿名类

&emsp;&emsp;在之前的做法中（Historically），使用单个抽象方法的接口（或很少的抽象类【只有一个抽象方法的抽象类数量比较少】）被用作函数类型。它们的实例称为函数对象，代表函数或行为。自 JDK 1.1 于 1997 年发布以来，创建函数对象的主要方法是匿名类（第 24 项）。下面一个代码片段，用于按长度顺序对字符串列表进行排序，使用匿名类创建排序的比较函数（强制排序顺序）：

```java
// Anonymous class instance as a function object - obsolete!
Collections.sort(words, new Comparator<String>() {
    public int compare(String s1, String s2) {
        return Integer.compare(s1.length(), s2.length());
    }
});
```

&emsp;&emsp;匿名类适用于需要功能对象的经典的面向对象的设计模式，特别是策略模式\[Gamma95\]。Comparator 接口表示用于排序的抽象策略; 上面的匿名类是排序字符串的具体策略。然而，匿名类的冗长使得 Java 中的函数式编程成为一个没有吸引力的前景。

&emsp;&emsp;在 Java 8 中，该语言正式成为这样一种概念，即使用单一抽象方法的接口是特殊的，值得特别对待。这些接口现在称为功能接口，该语言允许你使用 lambda 表达式或简称 lambdas 创建这些接口的实例。Lambdas 在功能上与匿名类相似，但更加简洁。 以下是上面的代码片段如何将匿名类替换为 lambda。样板消失了，行为很明显：

```java
// Lambda expression as function object (replaces anonymous class)
Collections.sort(words, (s1, s2) -> Integer.compare(s1.length(), s2.length()));
```

&emsp;&emsp;请注意，lambda（Comparator <String>）的类型，其参数（s1 和 s2，两个 String）及其返回值（int）的类型不在代码中。 编译器使用称为类型推断的过程从上下文中推导出这些类型。在某些情况下，编译器将无法确定类型，你必须指定它们。类型推断的规则很复杂：它们占据了 JLS 的整个章节\[JLS，18\]。很少有程序员详细了解这些规则，但这没关系。**省略所有 lambda 参数的类型，除非它们的存在使您的程序更清晰。** 如果编译器生成错误，告诉你无法推断 lambda 参数的类型，请指定它。有时你可能必须转换返回值或整个 lambda 表达式，但这种情况很少见。

&emsp;&emsp;关于类型推断，应该添加一个警告。第 26 项告诉你不要使用原始类型，第 29 项告诉你支持泛型类型，第 30 项告诉你支持泛型方法。当你使用 lambdas 时，这个建议是非常重要的，因为编译器获得了从泛型的执行类型推断出的大多数类型信息。如果你不提供此信息，编译器将无法进行类型推断，你必须在 lambdas 中手动指定类型，这将大大增加它们的详细程度【也就是代码量】。举例来说，如果变量词被声明为原始类型 List 而不是参数化类型 List <String>，那么上面的代码片段将无法编译。

&emsp;&emsp;顺便提一下，如果使用比较器构造方法代替 lambda，则片段中的比较器可以更简洁（第 14. 43 项）：

```java
Collections.sort(words, comparingInt(String::length));
```

&emsp;&emsp;实际上，通过利用 Java 8 中添加到 List 接口的 sort 方法，可以使代码段更短：

```java
words.sort(comparingInt(String::length));
```

&emsp;&emsp;将 lambda 添加到语言中使得使用函数对象变得切实可行。例如，请考虑第 34 项中的 Operation 枚举类型。因为每个枚举对其 apply 方法需要不同的行为，所以我们使用特定于常量的类主体并覆盖每个枚举常量中的 apply 方法。为了让你有清晰的记忆，这里是代码：

```java
// Enum type with constant-specific class bodies & data (Item 34)
public enum Operation {
    PLUS("+") {
        public double apply(double x, double y) { return x + y; }
    },
    MINUS("-") {
        public double apply(double x, double y) { return x - y; }
    },
    TIMES("*") {
        public double apply(double x, double y) { return x * y; }
    },
    DIVIDE("/") {
        public double apply(double x, double y) { return x / y; }
    };
    private final String symbol;
    Operation(String symbol) { this.symbol = symbol; }
    @Override
    public String toString() { return symbol; }
    public abstract double apply(double x, double y);
}
```

&emsp;&emsp;第 34 项说 enum 实例字段比特定于常量的类体更可取。使用前者而不是后者，Lambdas 可以轻松实现特定于常量的行为。只需将实现每个枚举常量行为的 lambda 传递给它的构造函数。 构造函数将 lambda 存储在实例字段中，apply 方法将调用转发给 lambda。 生成的代码比原始版本更简单，更清晰：

```java
// Enum with function object fields & constant-specific behavior
public enum Operation {
    PLUS ("+", (x, y) -> x + y),
    MINUS ("-", (x, y) -> x - y),
    TIMES ("*", (x, y) -> x * y),
    DIVIDE("/", (x, y) -> x / y);
    private final String symbol;
    private final DoubleBinaryOperator op;

    Operation(String symbol, DoubleBinaryOperator op) {
        this.symbol = symbol;
        this.op = op;
    }

    @Override
    public String toString() { return symbol; }

    public double apply(double x, double y) {
        return op.applyAsDouble(x, y);
    }
}
```

&emsp;&emsp;请注意，我们使用 DoubleBinaryOperator 接口来表示枚举常量行为的 lambdas。这是 java.util.function（第 44 项）中许多预定义的功能接口之一。它表示一个函数，它接受两个 double 参数并返回一个 double 结果。

&emsp;&emsp;查看基于 lambda 的 Operation 枚举，您可能会认为特定于常量的方法体已经过时了，但事实并非如此。跟类和方法不一样，**lambdas 缺乏名称和文档; 如果一个运算过程不能自我解释【代码就是最好的文档】，或超过几行，请不要将它放在 lambda 中。** 一行【代码】对于 lambda 是理想的，三行【代码】是合理的最大值。如果违反此规则，可能会严重损害程序的可读性。如果 lambda 很长或难以阅读，要么找到简化它的方法，要么重构你的程序来取代 lambda。此外，传递给枚举构造函数的参数在静态上下文中进行运算。因此，枚举构造函数中的 lambdas 无法访问枚举的实例成员。如果枚举类型具有难以理解的特定于常量的行为，无法在几行【代码】中实现，或者需要访问实例字段或方法，则仍然可以使用特定于常量的类主体。

&emsp;&emsp;同样，你可能会认为匿名类在 lambdas 时代已经过时了。这很接近事实，但是你可以用匿名类做一些你无法用 lambdas 做的事情。Lambdas 仅限于函数接口。如果要创建抽象类的实例，可以使用匿名类，但不能使用 lambda。同样，你可以使用匿名类来创建具有多个抽象方法的接口实例。最后，lambda 无法获得对自身的引用。在 lambda 中，this 关键字引用封闭的实例，这通常是你想要的。在匿名类中，this 关键字引用匿名类实例。如果需要从其体内【类内部】访问函数对象，则必须使用匿名类。【在 lambda 表达式中使用 this 关键字，获得的引用是 lambda 所在的实例的引用，在匿名类中使用 this 关键字，获得的是当前匿名类的实例的引用】

&emsp;&emsp;Lambdas 与匿名类都具有无法在实现中可靠地序列化和反序列化它们的属性【lambda 和匿名类都无法被序列化和反序列化】。因此，你应该很少（如果有的话）序列化 lambda（或匿名类实例）。如果您有一个要进行序列化的函数对象，例如 Comparator，请使用私有静态嵌套类的实例（第 24 项）。

&emsp;&emsp;总之，从 Java 8 开始，lambda 是迄今为止表示小函数对象的最佳方式。**除非必须创建非功能接口类型的实例，否则不要对函数对象使用匿名类。** 另外，请记住 lambda 使代表小函数对象变得如此容易，以至于它打开了以前在 Java 中不实用的函数式编程技术的大门。

> - [第 41 项：用标记接口定义类型](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第06章：枚举和注解/第41项：用标记接口定义类型.md)
> - [第 43 项：方法引用优先于 Lambda](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第07章：Lambda和Stream/第43项：方法引用优先于Lambda.md)
