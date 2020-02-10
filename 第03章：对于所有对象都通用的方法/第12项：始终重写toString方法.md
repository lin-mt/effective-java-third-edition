## 始终重写 toString 方法

&emsp;&emsp;虽然 java.lang.Object 提供了 toString 方法的一个实现，但它返回的字符串通常并不是使用类的用户所期望看到的。它由类名后跟“at”符号（@）和散列码的无符号十六进制表示组成，例如 PhoneNumber@163b91。 toString 的通用约定中约定了返回的字符串应该是“一个简洁且信息丰富的表示，人们可以很容易阅读。”虽然可以说 PhoneNumber@163b91 简洁易读，但是与 707-867-5309 相比信息量不大。toString 约定里面还有，“建议所有子类都覆盖这个方法。”确实是个好建议！

&emsp;&emsp;虽然它不像遵守 equals 和 hashCode 约定那样重要（第 10 和第 11 项），**但是提供一个好的 toString 实现会使你的类使用起来更加愉快，并使使用该类的系统更容易调试。** 当对象传递给 println，printf，字符串连接运算符或断言，或由调试器打印时，会自动调用 toString 方法。即使您从未在对象上调用 toString，其他人也可能从未调用。例如，具有对象引用的组件可能在记录的错误消息中包含该对象的字符串表示形式。如果您未能覆盖 toString，那么这个消息几乎有可能是没用的。

&emsp;&emsp;如果你已经为 PhoneNumber 提供一个好的 toString 方法，那么，要产生有用的诊断信息会非常容易：

```java
System.out.println("Failed to connect to " + phoneNumber);
```

&emsp;&emsp;不管是否重写了 toString 方法，程序猿都将以这种方式来产生诊断信息，但是如果没有重写 toString 方法，产生的信息就没什么用。提供好的 toString 方法，不仅有益于这个类的实例，同样也有益于那些包含这些实例的引用对象，特别是集合对象。打印 Map 时有下面这两条消息：`{Jenny=PhoneNumber@163b91}`和`{Jenny=707-867-5309}`你更愿意看到哪个？

&emsp;&emsp;**在实际应用中，toString 方法应该返回对象中包含的所有值得关注的信息，** 比如上述电话号码例子那样。如果对象太大，或者对象中包含的状态信息难以用字符串来表达，这样做就有点不切实际。在这种情况下，toString 应该返回一个摘要信息，例如`Manhattan residential phone directory (1487536 listings)`或者`Thread[main,5,main]`。理想情况下，字符串应该是不言自明的。（Thread 例子不满足这样的要求。）如果未能在其字符串表示中包含所有对象感兴趣的信息，那么特别恼人的惩罚是测试失败报告如下所示：

```java
Assertion failure: expected {abc, 123}, but was {abc, 123}.
```

&emsp;&emsp;在实现 toString 的时候，必须做出一个很重要的决定：是否在文档中指定返回值的格式。对于值类（value class）【比较关注数值的类】，比如电话号码、矩阵之类的，就推荐这么做。指定格式的好处是，它可以被用作一种标准的、明确的适合人们阅读的对象表示法。这种表示法可以用于输入和输出，以及用在将对象持久化成人们可阅读的表现方式（This representation can be used for input and output and in persistent human-readable data objects），例如 CVS 文件。如果指定格式，通常最好提供匹配的静态工厂或构造函数，以便程序员可以轻松地在对象及其字符串表示之间来回转换。 Java 平台库中的许多值类都采用了这种方法，包括 BigInteger，BigDecimal 和大多数基本类型的包装类（boxed primitive classes）。

&emsp;&emsp;指定 toString 返回值的格式也有不足之处：如果这个类已经被广泛使用，一旦指定格式，就必须始终如一地坚持这种格式。程序猿将会编写出相应的代码来解析这种字符串的表示方法、生成这种字符串的表示方法【程序猿会编写出怎么解析我们生成的指定格式的字符串、怎么生成我们指定格式的字符串的方法】，并将其嵌入到持久化的数据中。如果将来的发行版本中改变了表示形式，那么您减将破坏其代码和数据，他们当然会抱怨。如果不指定格式，就可以保留灵活性，便于在将来的发行版本中添加信息，或者改进格式。

&emsp;&emsp;**无论你是否决定指定格式，都应该在文档中明确表明你的意图。** 如果你要指定格式，你应该严格地这样去做。例如，这里有一个第 11 项中 PhoneNumber 类的 toString 方法：

```java
/**
* Returns the string representation of this phone number.
* The string consists of twelve characters whose format is
* "XXX-YYY-ZZZZ", where XXX is the area code, YYY is the
* prefix, and ZZZZ is the line number. Each of the capital
* letters represents a single decimal digit.
*
* If any of the three parts of this phone number is too small
* to fill up its field, the field is padded with leading zeros.
* For example, if the value of the line number is 123, the last
* four characters of the string representation will be "0123".
*/
@Override public String toString() {
    return String.format("%03d-%03d-%04d", areaCode, prefix, lineNum);
}
```

&emsp;&emsp;如果你决定不指定格式，那么文档注释部分也应该有如下所示的指示信息：

```java
/**
* Returns a brief description of this potion. The exact details
* of the representation are unspecified and subject to change,
* but the following may be regarded as typical:
*
* "[Potion #9: type=love, smell=turpentine, look=india ink]"
*/
@Override public String toString() { ... }
```

&emsp;&emsp;在阅读这段注释之后，对于那些依赖于格式的细节进行编程或者产生持久化数据的程序猿，则只能自己承担后果。

&emsp;&emsp;无论是否指定格式，**都为 toString 返回值中包含的所有信息，提供一种编程式的访问路径。** 例如，PhoneNumber 类应该包含针对 area code、prefix 和 line number 的访问方法。如果不这么做，就会迫使那些需要这些信息的程序猿不得不自己去解析这些字符串。除了降低了程序的性能，使得程序猿们去做这些不必要的工作之外，如果你改变格式的话，这个解析过程也是很容易出错的，会导致系统崩溃。如果没有提供这些访问方法，即使你已经指明了字符串的格式是可以变化的，这个字符串格式也成了事实上的 API（By failing to provide accessors, you turn the string format into a de facto API, even if you’ve specified that it’s subject to change.）。

&emsp;&emsp;在静态实用程序类中编写 toString 方法是没有意义的（第 4 项）。你也不应该在大多数枚举类型（第 34 项）中编写 toString 方法，因为 Java 为你提供了一个非常好的方法。但是，您应该在任何抽象类中编写 toString 方法，其子类共享一个公共字符串的表示形式。例如，大多数集合实现的 toString 方法都是从抽象集合类继承的。

&emsp;&emsp;第 10 项中讨论的 Google 的开源 AutoValue 工具将为您生成 toString 方法，大多数 IDE 也是如此。 这些方法会以非常适合的方式告诉您每个字段的内容，但不是专门针对类的含义。因此，例如，对我们的 PhoneNumber 类使用自动生成的 toString 方法是不合适的（因为电话号码具有标准字符串表示），但它对于我们的 Potion 类是完全可以接受的。也就是说，自动生成的 toString 方法远比从 Object 继承的方法更好，Object 的 toString 方法不会告诉您对象的值。

&emsp;&emsp;回顾一下，在你编写的每个可实例化的类中重写 Object 的 toString 实现，除非超类已经这样做了。 它使类的使用更加愉快，并有助于调试。toString 方法应该以美学上令人愉悦的格式返回对象的简明有用的描述（The toString method should return a concise, useful description of the object, in an aesthetically pleasing format）。

> - [第 11 项：当重写 equals 方法时也要重写 hashCode 方法](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第03章：对于所有对象都通用的方法/第11项：当重写equals方法时也要重写hashCode方法.md)
> - [第 13 项：谨慎地重写 clone 方法](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第03章：对于所有对象都通用的方法/第13项：谨慎地重写clone方法.md)
