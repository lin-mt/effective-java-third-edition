## 保护性地编写 readObject 方法

&emsp;&emsp;第 50 项介绍了一个不可变的日期范围(date-range)类，它包含可变的私有的 Date 类型的字段。该类通过在其构造器和访问方法（accessor）中保护性地拷贝 Date 对象，极力地维护其约束条件和不可变性。下面就是这个类：

```java
// Immutable class that uses defensive copying
public final class Period {
    private final Date start;
    private final Date end;
    /**
    * @param start the beginning of the period
    * @param end the end of the period; must not precede start
    * @throws IllegalArgumentException if start is after end
    * @throws NullPointerException if start or end is null
    */
    public Period(Date start, Date end) {
        this.start = new Date(start.getTime());
        this.end = new Date(end.getTime());
        if (this.start.compareTo(this.end) > 0)
            throw new IllegalArgumentException(start + " after " + end);
    }
    public Date start () { return new Date(start.getTime()); }
    public Date end () { return new Date(end.getTime()); }
    public String toString() { return start + " - " + end; }
    ... // Remainder omitted
}
```

&emsp;&emsp;假设你决定要把这个类做成可序列化的。因为 Period 对象的物理表示法正好反映了它的逻辑数据内容，所以，使用默认的序列化形式没有什么不合理的（第 87 项）。因此，为了使这个类成为可序列化的，似乎你所需要做的也就是在类的声明中增加“implements Serializable”字样。然而，如果你真的这样做，那么这个类将不再保证它的关键约束了。

&emsp;&emsp;问题在于，readObject 方法实际上相当于另一个公有的构造器，如同其他的构造器一样，它也要满足所有注意事项。构造器必须检查其参数的有效性（第 49 项），并且在必要的时候对参数进行保护性拷贝（第 50 项），同样地，readObject 方法也需要这么做。如果 readObject 方法无法做到这两者之一，对于攻击者来说，要违反这个类的约束条件相对就比较简单了。

&emsp;&emsp;不严格地说，readObject 是一个构造函数，它将字节流作为唯一参数。在正常使用中，字节流是通过序列化正常构造的实例生成的。当 readObject 面对一个人工生成的违反类的约束条件的字节流的时候，问题就出现了，它会生成一个违反类的约束条件的对象。通过这样的字节流可以用来创建一个无法使用普通构造函数创建的*不可能的对象（impossible object）*。

&emsp;&emsp;假设我们只简单地将“implements Serializable”添加到 Period 的类声明中。然后，这个丑陋的程序将生成一个 Period 实例，它的结束时间比起始时间还要早。对高位设置的字节值的强制转换是 Java 缺少字节文字与不幸的决策结合而导致字节类型签名的结果【下面的强转(byte)部分】：

```java
public class BogusPeriod {
    // Byte stream couldn't have come from a real Period instance!
    private static final byte[] serializedForm = {
        (byte)0xac, (byte)0xed, 0x00, 0x05, 0x73, 0x72, 0x00, 0x06,
        0x50, 0x65, 0x72, 0x69, 0x6f, 0x64, 0x40, 0x7e, (byte)0xf8,
        0x2b, 0x4f, 0x46, (byte)0xc0, (byte)0xf4, 0x02, 0x00, 0x02,
        0x4c, 0x00, 0x03, 0x65, 0x6e, 0x64, 0x74, 0x00, 0x10, 0x4c,
        0x6a, 0x61, 0x76, 0x61, 0x2f, 0x75, 0x74, 0x69, 0x6c, 0x2f,
        0x44, 0x61, 0x74, 0x65, 0x3b, 0x4c, 0x00, 0x05, 0x73, 0x74,
        0x61, 0x72, 0x74, 0x71, 0x00, 0x7e, 0x00, 0x01, 0x78, 0x70,
        0x73, 0x72, 0x00, 0x0e, 0x6a, 0x61, 0x76, 0x61, 0x2e, 0x75,
        0x74, 0x69, 0x6c, 0x2e, 0x44, 0x61, 0x74, 0x65, 0x68, 0x6a,
        (byte)0x81, 0x01, 0x4b, 0x59, 0x74, 0x19, 0x03, 0x00, 0x00,
        0x78, 0x70, 0x77, 0x08, 0x00, 0x00, 0x00, 0x66, (byte)0xdf,
        0x6e, 0x1e, 0x00, 0x78, 0x73, 0x71, 0x00, 0x7e, 0x00, 0x03,
        0x77, 0x08, 0x00, 0x00, 0x00, (byte)0xd5, 0x17, 0x69, 0x22,
        0x00, 0x78
    };
    public static void main(String[] args) {
        Period p = (Period) deserialize(serializedForm);
        System.out.println(p);
    }
    // Returns the object with the specified serialized form
    static Object deserialize(byte[] sf) {
        try {
            return new ObjectInputStream(new ByteArrayInputStream(sf)).readObject();
        } catch (IOException | ClassNotFoundException e) {
            throw new IllegalArgumentException(e);
        }
    }
}
```

&emsp;&emsp;被用来初始化 SerializableForm 的 byte 数组常量是这样产生的：首先对一个正常的 Period 实例进行序列化，然后对得到的字节流进行手工编辑。对于这个例子而言，字节流的细节并不重要，但是如果你很好奇的话，可以在*Java Object Serialization Specification*\[Serialization, 6\]中查到有关序列化字节流格式的描述信息。如果你运行这个程序，它就会打印出“Fri Jan 01 12:00:00 PST 1999 - Sun Jan 01 12:00:00 PST 1984”。只要把 Period 声明成可序列化的，就会使我们创建出违反其约束条件的对象。

&emsp;&emsp;为了修正这个问题，你可以为 Period 提供一个 readObject 方法，该方法首先调用 defaultReadObject，然后检查被反序列化之后的对象的有效性。如果有效性检查失败，readObject 方法就抛出一个 InvalidObjectExceptio 异常，防止完成序列化：

```java
// readObject method with validity checking - insufficient!
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
    s.defaultReadObject();
    // Check that our invariants are satisfied
    if (start.compareTo(end) > 0)
        throw new InvalidObjectException(start +" after "+ end);
}
```

&emsp;&emsp;尽管这种修正方式避免了攻击者创建无效的 Period 实例，但是，这里仍然隐藏着一个更为微妙的问题。通过伪造字节流，要想创建可变的 Period 实例仍然是有可能的，做法是：字节流以一个有效地 Period 实例开头，然后附加上两个额外的引用，指向 Period 实例中的两个私有的 Date 域。攻击者从 ObjectInputStream 中读取 Period 实例，然后读取附加在其后面的“恶意编写制造的对象引用”。这些对象引用使得攻击者能够访问到 Period 对象内部的私有 Date 域所引用的对象。通过改变这些 Date 实例，攻击者可以改变 Period 实例。下面的类演示了这种攻击：

```java
public class MutablePeriod {
    // A period instance
    public final Period period;
    // period's start field, to which we shouldn't have access
    public final Date start;
    // period's end field, to which we shouldn't have access
    public final Date end;
    public MutablePeriod() {
        try {
            ByteArrayOutputStream bos = new ByteArrayOutputStream();
            ObjectOutputStream out = new ObjectOutputStream(bos);
            // Serialize a valid Period instance
            out.writeObject(new Period(new Date(), new Date()));
            /*
            * Append rogue "previous object refs" for internal
            * Date fields in Period. For details, see "Java
            * Object Serialization Specification," Section 6.4.
            */
            byte[] ref = { 0x71, 0, 0x7e, 0, 5 }; // Ref #5
            bos.write(ref); // The start field
            ref[4] = 4; // Ref # 4
            bos.write(ref); // The end field
            // Deserialize Period and "stolen" Date references
            ObjectInputStream in = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()));
            period = (Period) in.readObject();
            start = (Date) in.readObject();
            end = (Date) in.readObject();
        } catch (IOException | ClassNotFoundException e) {
            throw new AssertionError(e);
        }
    }
}
```

&emsp;&emsp;要查看攻击的结果，请运行以下程序：

```java
public static void main(String[] args) {
    MutablePeriod mp = new MutablePeriod();
    Period p = mp.period;
    Date pEnd = mp.end;
    // Let's turn back the clock
    pEnd.setYear(78);
    System.out.println(p);
    // Bring back the 60s!
    pEnd.setYear(69);
    System.out.println(p);
}
```

&emsp;&emsp;在我的时区（locale）中，运行此程序会产生以下输出：

Wed Nov 22 00:21:29 PST 2017 - Wed Nov 22 00:21:29 PST 1978
Wed Nov 22 00:21:29 PST 2017 - Sat Nov 22 00:21:29 PST 1969

&emsp;&emsp;虽然 Period 实例被创建之后，它的约束条件没有被破坏，但是要随意地修改它的内部组件仍然是有可能的。一旦攻击者获得了一个可变的 Period 实例，他就可以将这个实例传递给一个“安全性依赖于 Period 的不可变性”的类，从而造成更大的危害。这种推断并不牵强：实际上，有许多类的安全性就是依赖于 String 的不可变性。

&emsp;&emsp;问题的根源在于，Period 的 readObject 方法并没有完成足够的保护性拷贝。**当一个对象反序列化的时候，对客户端不得拥有的对象引用的任何字段进行保护性拷贝至关重要** 。因此，对于每个可序列化的不可变类，如果它包含了私有的可变组建，那么在它的 readObject 方法中，必须要对这些组件进行保护性拷贝。下面的 readObject 方法可以确保 Period 的约束条件不会遭到破坏，以保持它的不可变性。

```java
// readObject method with defensive copying and validity checking
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
    s.defaultReadObject();
    // Defensively copy our mutable components
    start = new Date(start.getTime());
    end = new Date(end.getTime());
    // Check that our invariants are satisfied
    if (start.compareTo(end) > 0)
        throw new InvalidObjectException(start +" after "+ end);
}
```

&emsp;&emsp;注意，保护性拷贝是在有效性检查之前进行的，而且，我们没有使用 Date 的 clone 方法来进行保护性拷贝。这两个细节对于保护 Period 免受攻击是必要的（第 50 项）。同时也要注意到，对于 final 域，保护性拷贝是不可能的。为了使用 readObject 方法，我们必须要将 start 和 end 域做成非 final 的。这是很遗憾的，但是这还算是相对比较好的做法。有了这个新的 readObject 方法，并去掉了 start 和 end 域的 final 修饰符之后，MutablePeriod 类将不再有效。此时，上面的攻击程序就会产生这样的输出：

Wed Nov 22 00:23:41 PST 2017 - Wed Nov 22 00:23:41 PST 2017
Wed Nov 22 00:23:41 PST 2017 - Wed Nov 22 00:23:41 PST 2017

&emsp;&emsp;这是一个简单的“石蕊”测试，用于判断默认的 readObject 方法是否适用于某个类：您是否愿意添加一个公共构造函数，该构造函数将对象中每个非瞬时字段的值作为参数，并将值存储在字段中而不进行任何验证？如果没有，则必须提供 readObject 方法，并且必须执行构造函数所需的所有有效性检查和保护性拷贝。或者，您可以使用*序列化代理模式（serialization proxy pattern）*（第 90 项）。强烈建议使用此模式，因为它需要花费大量精力进行安全反序列化。

&emsp;&emsp;readObject 方法和构造函数之间还有一个相似之处，它们适用于非 final 的可序列化类。与构造函数一样，readObject 方法不能直接或间接调用可覆盖的方法（第 19 项）。如果违反此规则并且重写了相关方法，则重写方法将在子类的状态被反序列化之前运行。就可能会导致程序失败\[Bloch05，Puzzle 91\]。

&emsp;&emsp;总而言之，每当你编写 readObject 方法的时候，都要这样想：你在编写一个公有的构造器，无论给它传递什么样的字节流，它都必须产生一个有效的实例。不要假设这个字节流一定代表着一个真正被序列化过的实例。虽然在本项的例子中，类使用了默认的序列化形式，但是，所有讨论到的有可能发生的问题也同样适用于使用自定义序列化形式的类。下面以摘要的形式给出一些指导方针，有助于编写出更加健壮的 readObject 方法：

- 对于对象引用域必须保持为私有的类，要保护性地拷贝这些域中的每个对象。不可变类的可变组件就属于这一类别。

- 对于任何约束条件，如果检查失败，则抛出一个 InvalidObjectException 异常。这些检查动作应该跟在所有的保护性拷贝之后。

- 如果整个对象图在被反序列化之后必须进行验证，就应该使用 ObjectInputValidation 接口（不在本书中讨论）。

- 无论是直接的方式还是间接的方式，都不要调用类中任何可被覆盖的方法。

> - [第 87 项：考虑使用自定义的序列化形式](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第12章：序列化/第87项：考虑使用自定义的序列化形式.md)
> - [第 89 项：对于实例控制，枚举类型优先于 readResolve](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第12章：序列化/第89项：对于实例控制，枚举类型优先于readResolve.md)
