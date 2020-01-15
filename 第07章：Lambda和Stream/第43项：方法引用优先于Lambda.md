## 方法引用优先于 Lambda

&emsp;&emsp;lambda 优于匿名类的主要优点是它们更简洁。Java 提供了一种生成函数对象的方法，它比 lambda 更简洁：方法引用。这是一个程序的代码片段，它维护从任意 key 到 Integer 值的映射。如果该值被解释为 key 实例数的计数，则该程序是多集实现。代码段的功能是将数字 1 与 key 相关联（如果它不在映射中），并在 key 已存在时增加相关值：

```java
map.merge(key, 1, (count, incr) -> count + incr);
```

&emsp;&emsp;请注意，此代码使用 merge 方法，该方法已添加到 Java 8 中的 Map 接口。如果给定键 key 没有映射，则该方法只是插入给定的值; 如果已存在映射，则 merge 将给定的函数应用于当前值和给定值，并使用结果覆盖当前值。这段代码表示 merge 方法的典型用例。

&emsp;&emsp;代码读起来很 nice，但仍然有一些样板【代码】。参数 count 和 incr 不会增加太多值，并且占用相当大的空间。实际上，所有 lambda 告诉你的是该函数返回其两个参数的总和。从 Java 8 开始，Integer（以及所有其他包装的数字基本类型）提供了一个完全相同的静态方法 sum。我们可以简单地传递对此方法的引用，获得相同的结果，并且【代码】看起来不会那么乱：

```java
map.merge(key, 1, Integer::sum);
```

&emsp;&emsp;方法具有的参数越多，使用方法引用可以消除的样板【代码】就越多。但是，在某些 lambda 中，你选择的参数名称提供了有用的文档，使得 lambda 比方法引用更易读和可维护，即使 lambda 更长。

&emsp;&emsp;对于一个你不能用 lambda 做的方法引用，你无能为力（有一个模糊的例外 - 如果你很好奇，请参阅 JLS，9.9-2）。也就是说，方法引用通常会导致更短，更清晰的代码。如果 lambda 变得太长或太复杂，它们也会给你一个方向（out）：你可以将 lambda 中的代码提取到一个新方法中，并用对该方法的引用替换 lambda。你可以为该方法提供一个好名称，并将其记录在核心的内容中。

&emsp;&emsp;如果你使用 IDE 进行编程，如果可以的话，它就会提供方法引用替换 lambda。你要经常（并不总是）接受 IDE 提供的建议。有时候，lambda 将比方法引用更简洁。当方法与 lambda 属于同一类时，这种情况最常发生。例如，考虑这个片段，假定它出现在名为 GoshThisClassNameIsHumongous 的类中：

```java
service.execute(GoshThisClassNameIsHumongous::action);
```

&emsp;&emsp;使用 lambda 看起来像这样：

```java
service.execute(() -> action());
```

&emsp;&emsp;使用方法引用的代码段既不比使用 lambda 的代码段更短也更清晰，所以更喜欢后者。类似地，Function 接口提供了一个通用的静态工厂方法来返回 Identity 函数 Function.identity()。它通常更短更清洁，不使用此方法，而是编写等效的 lambda 内联：x -> x。

&emsp;&emsp;许多方法引用会引用静态方法，但有四种方法引用不会引用静态方法。其中两个是绑定和未绑定的实例方法引用。在绑定引用中，接收对象在方法引用中指定。绑定引用在本质上类似于静态引用：函数对象采用与引用方法相同的参数。在未绑定的引用中，在应用函数对象时，通过方法声明的参数之前的附加参数指定接收对象。未绑定引用通常用作流管道（stream pipelines）（第 45 项）中的映射和过滤功能。最后，对于类和数组，有两种构造函数引用。构造函数引用充当工厂对象。所有五种方法参考总结在下表中：

| Method Ref Type   | Example             | Lambda Equivalent                                      |
| ----------------- | ------------------- | ------------------------------------------------------ |
| Static            | Integer::parseInt   | str -> Integer.parseInt(str)                           |
| Bound             | Integer::parseIntr  | Instant then = Instant.now();<br> t -> then.isAfter(t) |
| Unbound           | String::toLowerCase | str -> str.toLowerCase()                               |
| Class Constructor | TreeMap<K, V>::new  | () -> new TreeMap<K, V>                                |
| Array Constructor | int[]::new          | len -> new int\[len\]                                  |

&emsp;&emsp;总之，方法引用通常提供一种更简洁的 lambda 替代方案。**在使用方法引用可以更简短更清晰的地方，就使用方法引用，如果无法使代码更简短更清晰的地方就坚持使用 lambda。（Where method references are shorter and clearer, use them; where they aren’t, stick with lambdas.）**

> - [第 42 项：Lambda 优先于匿名类](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第07章：Lambda和Stream/第42项：Lambda优先于匿名类.md)
> - [第 44 项：坚持使用标准的函数接口](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第07章：Lambda和Stream/第44项：坚持使用标准的函数接口.md)
