## 谨慎地实现 Serializable 接口

&emsp;&emsp;要想使一个类的实例可被序列化，非常简单，只要在它的声明中加入“implemants Serializable”字样即可。正因为太容易了，所以普遍存在这样一种误解，认为程序猿可以毫不费力就可以实现序列化。实际情形要复杂得多。虽然使一个类可序列化的直接成本可以忽略不计，但长期的成本通常是很高的。

&emsp;&emsp;**实现 Serializable 接口而付出的最大代价是，一旦一个类被发布，就大大降低了“改变这个类的实现”的灵活性** 。当一个类实现了 Serializable 接口，它的字节流编码（或者说*序列化形式（serialized form）*）就变成了它的导出的 API 的一部分。一旦这个类被广泛使用，往往必须永远支持这种序列化形式，就好像你必须要支持导出的 API 的所有其他部分一样。如果你不努力设计一种*自定义的序列化形式（custom serialized form）*，而仅仅接受了默认的序列化形式，这种序列化形式将永远地束缚在该类最初的内部表示法上。换句话说，如果你接受了默认的序列化形式，这个类中私有的和包级私有的实例域都将变成导出的 API 的一部分，这不符合“将域的访问权限限制到最低”的实践准则（第 15 项），因此，将它【除了 public 以外的权限修饰符】作为信息隐藏工具，它就失去了有效性。

&emsp;&emsp;如果你接受了默认的序列化形式，并且以后又要改变这个类的内部表示，结果可能导致序列化形式的不兼容。客户端程序企图用这个类的旧版本来序列化一个类，然后用新版本进行反序列化（反之亦然），结果将导致程序失败。在改变内部表示法的同时仍然维持原来的序列化形式（使用 ObjectOutputStream.putFields 和 ObjectInputStream.readFields），这也是可能的，但是做起来比较困难，并且会在源代码中留下一些明显的隐患。如果你选择序列化一个类，你应该仔细地设计一种高质量的序列化形式，并且在很长时间内都愿意使用这种形式（第 87、90 项）。这样做会增加开发最初的成本，但这是值得的。即使是精心设计的序列化形式也会限制一个类的演变；一个设计不良的序列化形式可能会这个类无法演变（an ill-designed serialized form can be crippling）。

&emsp;&emsp;序列化会使类的演变受到限制，这种限制的一个例子与*流的唯一标识符（stream unique identifier）*有关，通常它也被称为*序列版本 UID（serial version UID）*。每个可序列化的类都有一个唯一标识号与它相关联。如果通过声明一个名为 serialVersionUID 的静态 final 的 long 域来指定此数字，系统则会在运行时通过将加密哈希函数（SHA-1）应用于类的结构来自动生成它。这个自动生成的值受类的名称、它实现的接口以及大多数成员（包括编译器生成的合成成员）的影响。如果你更改了以上的任何内容，比如，增加一个便捷方法，生成的序列版本 UID 就会发生变化。如果没有声明序列版本 UID，兼容性将会遭到破坏，从而导致运行时出现 InvalidClassException 异常。

&emsp;&emsp;**实现 Serializable 的第二个代价是，它增加了出现 BUG 和安全漏洞的可能性** 。通常情况下，对象是利用构造器来创建的；序列化机制是一种语言之外的对象创建机制（extralinguistic mechanism）。无论你是接受了默认的行为，还是覆盖了默认的行为，反序列化机制（deserialization）都是一个“隐藏的构造器”，具备与其他构造器相同的特点。因为反序列化机制中没有显式的构造器，所以你会很容易忘记确保以下这一点：反序列化过程必须也要保证所有“由真正的构造器建立起来的约束关系”，并且不允许攻击者访问正在构造过程中的对象的内部信息。依靠默认的反序列化机制，很容易使对象的约束关系遭到破坏，以及遭受到非法访问（第 88 项）。

&emsp;&emsp;**实现 Serializable 的第三个代价是，随着类发行新的版本，测试相关的负担也增加了** 。当一个可序列化的类被修订的时候，很重要的一点是，要检查是否可以“在新版本中序列化一个实例，然后在旧版本中反序列化”，反之亦然。因此，测试所需的工作量与“可序列化的类的数量和可能很大的发行版号”的乘积成正比。你必须确保“序列化-反序列化”过程成功，并且它产生的对象真的是原始对象的复制品。如果在最初编写一个类的时候，就精心设计了自定义的序列化形式，测试的需求就可以有所降低。

&emsp;&emsp;**实现 Serializable 接口并不是一个很轻松就可以做出的决定** 。如果一个类将要加入到某个框架中，并且该框架依赖于序列化来实现对象传输或者持久化，那么这一点至关重要。此外，它极大地简化了将类用作另一个必须实现 Serializable 的类的组件。但是，实现 Serializable 会产生许多开销。每次设计类的时候，都要权衡一下成本和收益。根据以往的经验，比如 BigInteger 和 Instant 这样的值类型的类实现了 Serializable，而且集合也这么做了。代表活动实体的类，比如线程池（thread pool），应该很少实现 Serializable。

&emsp;&emsp;**为了继承而设计的类（第 19 项）应该很少实现 Serializable 接口，接口也很少继承 Serializable 接口** 。违反此规则会给扩展类或实现接口的任何人带来沉重的负担。有时候违反这条规则是合适的。例如，如果一个类或者接口存在的主要是为了参与到某个框架中，该框架要求所有的参与者都必须实现 Serializable 接口，那么，对于这个类或者接口来说，实现或者扩展 Serializable 接口就是非常有意义的。

&emsp;&emsp;为了继承而设计的类中，实现了 Serializable 接口的包括 Throwable 和 Component。Throwable 实现了 Serializable 接口，所以 RMI 可以将异常从服务器发送到客户端。Component 实现了 Serializable 接口，因此 GUI 可以被发送、保存和恢复，但是即使在今天的 Swing 和 AWT 中，这个工具很少在实践中使用。

&emsp;&emsp;如果实现具有可序列化和可扩展的实例字段的类，则需要注意这几个风险。如果实例字段值上存在任何约束条件，则防止子类覆盖 finalize 方法至关重要，该类可以通过重写 finalize 并将其声明为 final 来完成。否则，该类将容易受到*终结者攻击（finalizer attacks）*（第 8 项）。最后，如果类的实例字段初始化为其默认值（整数类型为零，布尔值为 false，对象引用类型为 null），则会违反约束条件，必须为此添加 readObjectNoData 方法：

```java
// readObjectNoData for stateful extendable serializable classes
private void readObjectNoData() throws InvalidObjectException {
    throw new InvalidObjectException("Stream data required");
}
```

&emsp;&emsp;在 Java 4 中就添加了此方法，以涵盖涉及向现有可序列化类\[Serialization，3.5\]添加可序列化超类的极端情况。

&emsp;&emsp;关于不实现 Serializable 的决定有一点需要注意。如果为继承而设计的类不可序列化，则可能需要额外的努力才能编写可序列化的子类。这种类的正常反序列化要求超类具有可访问的无参数构造函数\[Serialization，1.10\]。如果您不提供这样的构造函数，则强制子类使用序列化代理模式（第 90 项）。

&emsp;&emsp;**内部类（第 24 项）不应该实现 Serializable 接口** 。它们使用编译器产生的*合成域（synthetic field）*来保存指向*外围实例（enclosing instabce）*的引用，以及保存来自外围作用域的局部变量的值。“这些域如何对应到类定义中”并没有明确的规定，就好像没有指定匿名类和局部类的名称一样。因此，内部类的默认序列化形式是定义不清楚的。然而，\*静态成员类（static member class）却可以实现 Serializable 接口。

&emsp;&emsp;总而言之，实现 Serializable 接口只是看起来很容易。除非只在受保护的环境中使用类，其中各个版本之间永远不必进行互操作，并且服务器永远不会暴露给不受信任的数据，否则实现 Serializable 接口是一个很严谨的承诺，应该认真对待。如果一个类允许继承，则需要格外小心。

> - [第 85 项：其他方法优先于 Java 序列化](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第12章：序列化/第85项：其他序列化优先于Java序列化.md)
> - [第 87 项：考虑使用自定义的序列化形式](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第12章：序列化/第87项：考虑使用自定义的序列化形式.md)
