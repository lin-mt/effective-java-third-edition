## 对于实例控制，枚举类型优先于 readResolve

&emsp;&emsp;第三项讲述了*单例（singleton）*模式，并且给出了以下这个 Singleton 类的示例。这个类限制了对其构造器的访问，以确保永远只创建一个实例：

```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public void leaveTheBuilding() { ... }
}
```

&emsp;&emsp;正如第 3 项中提到的，如果这个类的声明加上了“implements Serializable”的字样，他就不再是一个单例。无论该类是否使用了默认的序列化形式，还是自定义的序列化形式（第 87 项），都没关系；也跟它是否提供了显示的 readObject 方法（第 88 项）无关。任何一个 readObject 方法，不管是显示的还是默认的，它都返回一个新建的实例，这个新建的实例不同于该类初始化时创建的实例。

&emsp;&emsp;readResolve 特性允许你用 readObject 创建的实例代替另一个实例\[Serialization, 3.7\]。对于一个正在被反序列化的对象，如果它的类定义了一个 readResolve 方法，并且具备正确的声明，那么在反序列化之后，新建的对象上的 readResolve 方法就会被调用。然后，该方法返回的对象引用将被返回，取代新建的对象。在这个特性的绝大多数用法中，指向新建的对象的引用不需要再被保留，因此立即成为垃圾回收的对象。

&emsp;&emsp;如果 Elvis 类要实现 Serializable 接口，下面的 readResolve 方法就足以保证它的单例属性：

```java
// readResolve for instance control - you can do better!
private Object readResolve() {
    // Return the one true Elvis and let the garbage collector
    // take care of the Elvis impersonator.
    return INSTANCE;
}
```

&emsp;&emsp;该方法忽略了被反序列化的对象，只返回该类初始化是创建的那个特殊的 Elvis 实例。因此，Elvis 实例的序列化形式并不需要包含任何实际的数据；所有的实例域都应该被声明为 transient 的。事实上，**如果依赖 readResolve 进行实例控制，带有对象引用类型的所有实例域则都必须声明为 transient 的** 。否则，那种有决心的攻击者就有可能在 readResolve 方法被运行前，取得指向反序列化对象的引用，采用的攻击技术类似于第 88 项中提到过的 MutablePeriod 攻击。

&emsp;&emsp;这种攻击有点复杂，但是底层思想却很简单。如果 Singleton 包含一个非 transient 的对象引用域，这个域的内容就可以在 Singleton 的 readResolve 方法运行之前被反序列化。当对象引用域的内容被反序列化时，它就允许一个精心制作的流“盗用”指向最初被反序列化的 Singleton 的引用。

&emsp;&emsp;以下是它更详细的工作原理。首先，编写一个“盗用者(stealer)”类，它既有 readResolve 方法，又有实例域，实例域指向被序列化的 Singleton 的引用，“盗用者”类就“潜伏”在其中。在序列化流中，用“盗用者”类的实例代替 Singleton 的非 transient 域。你现在就有了一个循环：Singleton 包含“盗用者”类，“盗用者”类则引用该 Singleton。

&emsp;&emsp;由于单例包含“盗用者”类，当这个单例被反序列化的时候，“盗用者”类的 readResolve 方法先运行。因此，当“盗用者”的 readResolve 方法运行时，它的实例域仍然引用被部分反序列化（并且也还没有被解析）的单例。

&emsp;&emsp;“盗用者”的 readResolve 方法从它的实例域中将引用复制到静态域中，以便该引用可以在 readResolve 方法运行之后被访问到。然后这个方法为它所藏身的那个域返回一个正确的类型值。如果没有这么做，当序列化系统试着将“盗用者”引用保存到这个域中时，VM 就会抛出 ClassCastException。

&emsp;&emsp;为了更具体地说明这一点，我们来考虑下面这个被破坏了的单例：

```java
// Broken singleton - has nontransient object reference field!
public class Elvis implements Serializable {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { }
    private String[] favoriteSongs = { "Hound Dog", "Heartbreak Hotel" };
    public void printFavorites() {
        System.out.println(Arrays.toString(favoriteSongs));
    }
    private Object readResolve() {
        return INSTANCE;
    }
}
```

&emsp;&emsp;下面是个“盗用者”类，是根据上面的描述构造的：

```java
public class ElvisImpersonator {
    // Byte stream couldn't have come from a real Elvis instance!
    private static final byte[] serializedForm = {
        (byte)0xac, (byte)0xed, 0x00, 0x05, 0x73, 0x72, 0x00, 0x05,
        0x45, 0x6c, 0x76, 0x69, 0x73, (byte)0x84, (byte)0xe6,
        (byte)0x93, 0x33, (byte)0xc3, (byte)0xf4, (byte)0x8b,
        0x32, 0x02, 0x00, 0x01, 0x4c, 0x00, 0x0d, 0x66, 0x61, 0x76,
        0x6f, 0x72, 0x69, 0x74, 0x65, 0x53, 0x6f, 0x6e, 0x67, 0x73,
        0x74, 0x00, 0x12, 0x4c, 0x6a, 0x61, 0x76, 0x61, 0x2f, 0x6c,
        0x61, 0x6e, 0x67, 0x2f, 0x4f, 0x62, 0x6a, 0x65, 0x63, 0x74,
        0x3b, 0x78, 0x70, 0x73, 0x72, 0x00, 0x0c, 0x45, 0x6c, 0x76,
        0x69, 0x73, 0x53, 0x74, 0x65, 0x61, 0x6c, 0x65, 0x72, 0x00,
        0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x02, 0x00, 0x01,
        0x4c, 0x00, 0x07, 0x70, 0x61, 0x79, 0x6c, 0x6f, 0x61, 0x64,
        0x74, 0x00, 0x07, 0x4c, 0x45, 0x6c, 0x76, 0x69, 0x73, 0x3b,
        0x78, 0x70, 0x71, 0x00, 0x7e, 0x00, 0x02
    };
    public static void main(String[] args) {
        // Initializes ElvisStealer.impersonator and returns
        // the real Elvis (which is Elvis.INSTANCE)
        Elvis elvis = (Elvis) deserialize(serializedForm);
        Elvis impersonator = ElvisStealer.impersonator;
        elvis.printFavorites();
        impersonator.printFavorites();
    }
}
```

&emsp;&emsp;运行这个程序会产生下列输出，最终证明可以创建两个截然不同的 Elvis 实例（包含两种不同的音乐品味）：

[Hound Dog, Heartbreak Hotel][a fool such as i]

&emsp;&emsp;通过将 favorites 域声明为 transient，可以修复这个问题，但是最好把 Elvis 做成是一个单元素(single-element)的枚举类型（第 3 项）。正如 ElvisStealer 攻击所证明的那样，使用 readResolve 方法来防止攻击者访问“临时”反序列化实例是非常脆弱的，需要非常小心。

&emsp;&emsp;如有你将你的可序列化的实例受控（instance-controlled）类编写成枚举，Java 就可以保证除了所声明的常量之外，不会有别的实例，除非攻击者滥用 AccessibleObject.setAccessible 等特权方法。任何能够做到这一点的攻击者已经拥有足够的权限来执行任意的本机代码（native code），并且为之所做的所有努力都是没用的。以下是把我们的 Elvis 写成枚举的例子：

```java
// Enum singleton - the preferred approach
public enum Elvis {
    INSTANCE;
    private String[] favoriteSongs = { "Hound Dog", "Heartbreak Hotel" };
    public void printFavorites() {
        System.out.println(Arrays.toString(favoriteSongs));
    }
}
```

&emsp;&emsp;用 readResolve 进行实例控制并不过时。如果必须编写可序列化的实例受控的类，在编译时还无法知道它的实例，你就无法将类表示成一个枚举类型。

&emsp;&emsp;**readResolve 的可访问性（accessibility）很重要** 。如果把 readResolve 方法放在一个 final 类上，它就应该是私有的。如果把 readResolve 方法放在一个非 final 的类上，就必须认真考虑它的可访问性。如果它是私有的，就不适用于任何子类。如果它是包级私有的，就只适用于同一个包中的子类。如果它是受保护的或者公有的，就适用于所有没有覆盖它的子类。如果 readResolve 方法是受保护的或者公有的，并且子类没有覆盖它，对序列化过的子类实例进行反序列化，就会产生一个超类实例，这样有可能导致 ClassCastException 异常。

&emsp;&emsp;总而言之，你应该尽可能地使用枚举类型来实施实例受控的约束条件。如果做不到，同时又需要一个即可实例化又是实例受控（instance-controlled）的类，就必须提供一个 readResolve 方法，并确保该类的所有实例域都为基本类型，或者是 transient 的。

> - [第 88 项：保护性地编写 readObject 方法](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第12章：序列化/第88项：保护性地编写readObject方法.md)
> - [第 90 项：考虑用序列化代理代替序列化实例](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第12章：序列化/第90项：考虑用序列化代理代替序列化实例.md)
