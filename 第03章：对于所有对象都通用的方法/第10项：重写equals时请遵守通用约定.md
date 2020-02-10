## 重写 equals 时请遵守通用约定

&emsp;&emsp;重写 equals 方法看起来似乎很简单，但是有许多重写方式会导致错误，而且后果非常严重。最容易避免这类问题的办法就是不覆盖 equals 方法，在这种情况下，类的每个实例都只能与它自身相等。如果满足了以下任何一个条件，那就是正确的做法：

- **类的每个实例都是唯一的。**　对于代表活动实体而不是值（value）的类来说确实如此，例如 Thread。Object 提供的 equals 实现对这些类具有完全正确的行为（The equals implementation provided by Object has exactly the right behavior for these classes）。

- **不关心类是否提供了“逻辑相等（logical equality）”的测试功能。**　例如，java.util.regex.Pattern 可以重写 equals 检查两个 Pattern 实例是否表示完全相同的正则表达式，但设计者并不认为客户端需要或想要此功能。 在这种情况下，从 Object 继承的 equals 实现是理想的方式。

- **超类已经重写了 equals，从超类继承过来的行为对于子类也是合适的。**　例如，大多数的 Set 实现都从 AbstractSet 继承 equals 实现，List 实现从 AbstractList 继承 equals 实现，Map 实现从 AbstractMap 继承 equals 实现。

- **类是私有的或者是包级私有的，可以确定它的 equals 方法永远不会被调用。**　如果你非常讨厌风险，你可以重写 equals 方法，从而确保它不会被意外调用：

```java
@Override public boolean equals(Object o) {
    throw new AssertionError(); // Method is never called
}
```

&emsp;&emsp;那么什么时候重写 equals 方法才是合适的呢？当一个类具有逻辑相等的概念时（不同于对象本身相同的概念），而超类还没有重写 equals。 这通常是“值类（value class）”的情况。 值类指的是只表示值的类，例如 Integer 或 String。程序猿在利用 equals 方法来比较对象的引用时，希望知道它们在逻辑上是否相等，而不是像了解它们是否引用了相同的对象。为了满足程序猿的需求，不仅必须重写 equals 方法，而且这样做也使得这个类的实例可以被用作映射表（map）的键（key），或者集合（set）的元素，使映射或者集合表现出预期的行为。

&emsp;&emsp;有一种“值类”不需要重写 equals 方法，即用实例受控（第 1 项）确保“每个值之多只存在一个对象”的类【单例模式】。枚举类型（第 34 项）就属于这种类。对于这样的类而言，逻辑相同与对象等同是一回事，因此 Object 的 equals 方法等同于逻辑意义上的 equals 方法。

&emsp;&emsp;当你重写 equals 方法的时候，你一定要遵守它的通用约定。下面是约定的内容，来自 Object 的规范：

- **自反性（Reflexive）**：对于任何非 null 的引用值 x，x.equals(x)必须返回 true。
- **对称性（Symmetric）**：对于任何非 null 的引用值 x 和 y，当且仅当 y.equals(x)返回 true 时，x.equals(y)必须返回 true。
- **传递性（Transitive）**：对于任何非 null 的引用值 x、y 和 z，如果 x.equals(y)返回 true，并且 y.equals(z)也返回 true，那么 x.equals(z)也必须返回 true。
- **一致性（Consistent）**：对于任何非 null 的引用值 x 和 y，只要 equals 的比较操作在对象中所用的信息没有被修改，多次调用 x.equals(y)就会一致返回 true，或者一致返地返回 false。
- 对于任何非 null 的引用值 x，x.equals(null)必须返回 false。

&emsp;&emsp;除非你对数学特别感兴趣，否则这些规定看起来可能有点让人感到恐惧，但是绝对不要忽视这些规定！如果你违反了它们，就会发现你的程序表现不正常，甚至崩溃，而且很难阻止失败的根源（and it can be very difficult to pin down the source of the failure）。引用 John Donne 的话说，没有哪个类是孤立的。一个类的实例通常会被频繁地传递给另一个类的实例。有许多类，包括所有的集合类（collections classes）在内，都依赖于传递给它们的对象是否遵守了 equals 约定。

&emsp;&emsp;现在你已经知道了违反 equals 约定有多么可怕，现在我们就来更细致地讨论这些约定。值得欣慰的是，这些约定虽然看起来很吓人，实际上并不复杂。一旦你理解了这些约定，要遵守它们并不困难。

&emsp;&emsp;那么什么是等价关系呢？大致来说，它是一个运算符，它将一组元素分成子集，这些子集的元素被认为是彼此相等的。这些子集称为等价类。 要使 equals 方法有用，每个等价类中的所有元素必须可以从用户的角度进行互换。现在我们依次检查一下 5 个要求：

&emsp;&emsp;**自反性（Reflexivity）**————第一个要求仅仅说明对象必须等于其自身。很难想象会无意识地违反这一条。假如违背了这一条，然后把该类的实例添加到集合（collection）中，该集合的 contains 方法将会告诉你，该集合不包含你刚才添加的实例。

&emsp;&emsp;**对称性（Symmetry）**————第二个要求是说，任何两个对象对于“他们是否相等”的问题都必须保持一致。与第一个要求不同，若无意中违反第一条，这种情形倒是不难想象。例如，考虑下面的类，它实现了一个区分大小写的字符串。字符串由 toString 保存，但在比较操作中被忽略：

```java
// Broken - violates symmetry!
public final class CaseInsensitiveString {
    private final String s;
    public CaseInsensitiveString(String s) {
        this.s = Objects.requireNonNull(s);
    }
    // Broken - violates symmetry!
    @Override public boolean equals(Object o) {
        if (o instanceof CaseInsensitiveString)
            return s.equalsIgnoreCase(((CaseInsensitiveString) o).s);
        if (o instanceof String) // One-way interoperability!
            return s.equalsIgnoreCase((String) o);
        return false;
    }
    ... // Remainder omitted
}
```

&emsp;&emsp;在这个类中，equals 方法的意图非常好，它企图与普通的字符串（String）对象进行互操作。假设我们有一个不区分大小写的字符串和一个普通的字符串：

```java
CaseInsensitiveString cis = new CaseInsensitiveString("Polish");
String s = "polish";
```

正如所料，`cis.equals(s)`返回 true。问题在于，虽然 CaseInsensitiveString 类中的 equals 方法知道普通的字符串（String）对象，但是 String 类中的 equals 方法却并不知道不区分大小写的字符串。因此，`s.equals(cis)`返回 false，显然违反了对称性。假设你把不区分大小写的字符串对象放到一个集合中：

```java
List<CaseInsensitiveString> list = new ArrayList<>();
list.add(cis);
```

此时 list.contains(s)会返回什么结果呢？谁能知道？在当前 OpenJDK 的实现中，它碰巧返回的是 false，但这只是这个特定实现得出的结果而已。在其他的视线中，它有可能返回 true，或者抛出一个运行时异常。**一旦你违反了 equals 约定，当其他对象面对你的对象的时候，你完全不知道这些对象的行为会怎么样。**

&emsp;&emsp;为了解决这个问题，只需要把企图与 String 互操作的这段代码从 equals 中去掉就可以了。这样做以后，就可以重构该方法，使它变成一条单独的返回语句：

```java
@Override public boolean equals(Object o) {
    return o instanceof CaseInsensitiveString && ((CaseInsensitiveString) o).s.equalsIgnoreCase(s);
}
```

&emsp;&emsp;**传递性**————equals 约定的第三个要求是，如果第一个对象等于第二个对象，并且第二个对象等于第三个对象，则第一个对象一定等于第三个对象。同样地，无意识地违反这条规则的情形也不难想象。考虑子类的情形，它将一个新的值组件（value component）添加到了超类中。换句话说，子类增加的信息会影响到 equals 的比较结果。我们首先以一个简单的不可变的二维整数型 Point 类作为开始：

```java
public class Point {
    private final int x;
    private final int y;
    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }
    @Override public boolean equals(Object o) {
        if (!(o instanceof Point))
            return false;
        Point p = (Point)o;
        return p.x == x && p.y == y;
    }
    ... // Remainder omitted
}
```

假设你想要继承这个类，为一个点添加颜色信息：

```java
public class ColorPoint extends Point {
    private final Color color;
    public ColorPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }
    ... // Remainder omitted
}
```

&emsp;&emsp;equals 方法会怎么样呢？如果完全不提供 equals 方法，而是直接从 Point 继承过来，在 equals 做比较的时候颜色信息就被忽略掉了。虽然这样做不会违反 equals 约定，但是很明显是无法接受的。假设你编写了一个 equals 方法，只有当它的参数是另一个有色点，并且具有相同的位置和颜色时，它才会返回 true：

```java
// Broken - violates symmetry!
@Override public boolean equals(Object o) {
    if (!(o instanceof ColorPoint))
        return false;
    return super.equals(o) && ((ColorPoint) o).color == color;
}
```

&emsp;&emsp;这种方法的问题在于，当将一个点与一个颜色点进行比较时，可能会得到不同的结果，反之亦然。前一种比较忽略了颜色信息，而后一种比较则总是返回 false，因为参数的类型不正确。为了直观地说明问题所在，我们创建一个点和有色点：

```java
Point p = new Point(1, 2);
ColorPoint cp = new ColorPoint(1, 2, Color.RED);
```

然后，`p.equals(cp)`返回 true，cp.equals(p)则返回 false。你可以做这样的尝试来修正这个问题，让 ColorPoint.equals 在进行“混合比较”时忽略颜色信息：

```java
// Broken - violates transitivity!
@Override public boolean equals(Object o) {
    if (!(o instanceof Point))
        return false;
    // If o is a normal Point, do a color-blind comparison
    if (!(o instanceof ColorPoint))
        return o.equals(this);
    // o is a ColorPoint; do a full comparison
    return super.equals(o) && ((ColorPoint) o).color == color;
}
```

这种方法确实提供了对称性，但是却牺牲了传递性：

```java
ColorPoint p1 = new ColorPoint(1, 2, Color.RED);
Point p2 = new Point(1, 2);
ColorPoint p3 = new ColorPoint(1, 2, Color.BLUE);
```

此时，`p1.equals(p2)`和`pe.equals(p3)`都是返回 true，但是`p1.equals(p3)`则返回 false，很显然违反了传递性。前两种比较不考虑颜色信息（“色盲”），而第三种比较则考虑了颜色信息。

&emsp;&emsp;同样，这种方法可以导致无限递归：假设有两个 Point 的子类，叫做 ColorPoint 和 SmellPoint，每一个子类都使用这种 equals 方法，那么调用 myColorPoint。equals(mySmellPonit)将会抛出一个堆栈溢出的异常（StackOverflowError）。

&emsp;&emsp;那么怎么解决呢？这是面向对象语言中关于等价关系的一个基本问题。**我们无法在扩展可实例化的类的同事，既增加新的值组件，同时又保留 equals 约定，**除非愿意放弃面向对象的抽象所带来的优势。

&emsp;&emsp;你可能听说，在 equals 方法中用 getClass 测试代替 instanceof 测试，可以扩展可实例化的类和增加新的值组件，同时保留 equals 约定：

```java
// Broken - violates Liskov substitution principle (page 43)
@Override public boolean equals(Object o) {
    if (o == null || o.getClass() != getClass())
        return false;
    Point p = (Point) o;
        return p.x == x && p.y == y;
}
```

&emsp;&emsp;这段程序只有当对象具有相同的实现时，才能使对象等同。虽然这样也不算太糟糕，但是结果却是无法接受的：Point 子类的一个实例仍然是一个 Point，它仍然需要作为一个函数运行，但是如果采用这种方法它就不能这样做！ 让我们假设我们想要写一个方法来判断一个点是否在单位圆上。 这是我们可以做到的一种方式：

```java
// Initialize unitCircle to contain all Points on the unit circle
private static final Set<Point> unitCircle = Set.of(
    new Point( 1, 0), new Point( 0, 1),
    new Point(-1, 0), new Point( 0, -1));

public static boolean onUnitCircle(Point p) {
    return unitCircle.contains(p);
}
```

&emsp;&emsp;虽然这可能不是实现这种功能的最快方式，不过它的效果很好。但是假设你通过某种不添加值组件的方式扩展了 Point，例如让它的构造器记录创建了多少个实例：

```java
public class CounterPoint extends Point {
    private static final AtomicInteger counter = new AtomicInteger();
    public CounterPoint(int x, int y) {
        super(x, y);
        counter.incrementAndGet();
    }
    public static int numberCreated() { return counter.get(); }
}
```

&emsp;&emsp;里氏替换原则（Liskov substitution principle）认为，一个类型的任何重要属性也将适用于它的子类型，因此为该类型编写的任何方法，在它的子类型上也应该同样运行得很好[Liskov87]。这是我们早期的正式声明，即 Ponit（如 CounterPoint）的子类仍然是 Point，并且必须作为一个 Point 来工作。但是假设我们将 CounterPoint 传递给 onUnitCircle 方法。 如果 Point 类使用基于 getClass 的 equals 方法，则无论 CounterPoint 实例的 x 和 y 坐标如何，onUnitCircle 方法都将返回 false。之所以如此，是因为像 onUnitCircle 方法所用的是 HashSet 这样的集合，利用 equals 方法检验包含条件【即在 Set 集合中插入对象的时候会先检查该对象是否已存在】，没有任何 CounterPonit 实例与任何 Point 实例对应。但是，如果在 Point 上使用适当的基于 instanceof 的 equals 方法，当遇到 CounterPoint 时，相同的 onUnitCircle 方法就可以正常工作。

&emsp;&emsp;虽然没有一种令人满意的办法可以即扩展不可实例化的类，又增加值组件，但还是有一种不错的权宜之计（workaround）：根据第 18 项的建议：组合优先于继承。我们不再让 ColorPoint 扩展 Point，而是在 ColorPoint 中加入一个私有的 Point 字段，以及一个公有的视图（view）方法（第 6 项），此方法返回一个与该有色点处在相同位置的普通 Point 对象：

```java
// Adds a value component without violating the equals contract
public class ColorPoint {
    private final Point point;
    private final Color color;
    public ColorPoint(int x, int y, Color color) {
        point = new Point(x, y);
        this.color = Objects.requireNonNull(color);
    }

    /**
    * Returns the point-view of this color point.
    */
    public Point asPoint() {
        return point;
    }
    @Override public boolean equals(Object o) {
        if (!(o instanceof ColorPoint))
            return false;
        ColorPoint cp = (ColorPoint) o;
        return cp.point.equals(point) && cp.color.equals(color);
    }
    ... // Remainder omitted
}
```

&emsp;&emsp;在 Java 平台类库中，有一些类扩展了可实例化的类，并添加了新的值组件。例如，java.sql.Timestamp 对 java.util.Date 进行了扩展，并增加了 nanoseconds 字段。Timestamp 的 equals 实现却是违反了对称性，如果 Timestamp 和 Date 对象被用于同一个集合中，或者以其他方式被混合在一起，则会引起不正确的行为。Timestamp 类有一个免责声明，告诫程序猿不要混合使用 Date 和 Timestamp 对象。只要你不把它们混合在一起，就不会有麻烦，除此之外没有其他的措施可以防止你这么做，而且结果导致的错误将很难调试。Timestamp 类的这种行为是个错误，不值得效仿。

&emsp;&emsp;注意，你可以在一个抽象（abstrace）类的子类中增加新的值组件，而不会违反 equals 约定。这对于类层次结构很重要，您可以通过遵循第 23 项中的建议“用类层次（class hierarchies）代替标签类（tagged classes）”来获得类层次结构。例如，你可能有一个抽象的 Shap 类，它没有任何值组件，Circle 子类添加了一个 radius 字段，Rectangle 子类添加了 length 和 width 字段。只要不可能直接创建超类的实例，前面所述的种种问题就都不会发生。

&emsp;&emsp;**一致性（Consistency）**————equals 约定的第四个要求是，如果两个对象相等，他们就必须始终保持相等，除非它们中有一个（或者两个都）被修改了。换句话说，可变对象在不同的时候可以与不同的对象相等，而不可变对象则不会这样。当你在写一个类的时候，应该仔细考虑清楚它是否应该是不可变的（第 17 项）。如果认为它应该是不可变的，就必须保证 equals 方法满足这样的限制条件：相等的对象永远相等，不相等的对象永远不相等。

&emsp;&emsp;无论类是否是不可变的，都不要使 equals 方法依赖于不可靠的资源。如果你违反了这个禁令，就很难满足一致性要求。例如，java.net.URL 的 equals 方法依赖于对 URL 中主机 IP 地址的比较。将一个主机名转变成 IP 地址可能需要访问网络，随着时间的推移，不确保会产生相同的结果。这样会导致 URL 的 equals 方法违反 equals 约定，在实践中有可能引发一些问题。URL 中 equals 方法的这种行为是一个很大的错误，而且不应该被效仿。遗憾的是，因为兼容性的要求，这一行为无法被改变。为了避免这种问题，equals 方法应该只对驻留在内存的对象执行确定性计算。

&emsp;&emsp;**非空性（Non-nullity）**————最后一个要求没有名称，我姑且称它为“非空性（Non-nullity）”。意思是指所有对象都必须不等于 null。虽然很难想象在调用 o.equals（null）时偶然返回 true，但不难想象会不小心抛出 NullPointerException。通用约定是禁止这样做的。许多类的 equals 方法都通过一个显示的 null 测试来防止这种情况：

```java
@Override public boolean equals(Object o) {
    if (o == null)
        return false;
    ...
}
```

&emsp;&emsp;这项测试是不必要的。为了测试其参数的等同性，equals 方法必须先把参数转换成适当的类型，以便可以调用它的访问方法（accessor），或者访问它的字段。在进行转换之前，equals 方法必须使用 instanceof 操作符，检查其参数是否为正确的类型：

```java
@Override public boolean equals(Object o) {
    if (!(o instanceof MyType))
        return false;
    MyType mt = (MyType) o;
    ...
}
```

&emsp;&emsp;如果漏掉了这一步的类型检查，并且传递给 equals 方法的参数又是错误的类型，那么 equals 方法就会抛出`ClassCastException`异常，这就违反了 equals 的约定。但是，如果 instanceof 的第一个运算对象（operand）是 null，那么，不管第二个操作对象（operand）是哪种类型，instanceof 操作符都会返回 false[JLS, 15.20.2]，因此，如果把 null 传递给 equals 方法，类型检查就会返回 false，所以不需要单独的 null 检查。

&emsp;&emsp;结合所有这些要求，得出了以下实现高质量 equals 方法的诀窍：

1. **使用==操作符检查“参数是否为这个对象的引用”。** 如果是，则返回 true。这只不过是一种性能优化，如果比较操作有可能很昂贵，就值得这么做。
2. **使用 instanceof 操作符检查“参数是否为正确的类型”。** 如果不是，则返回 false，一般来说，所谓的“正确的类型”是指 equals 方法所在的那个类。有些情况下，它是指该类所实现的某个接口。如果类实现的接口改进了 equals 约定，允许在实现了该接口的类之间进行比较，那么就使用接口。集合接口（collection interfaces）如 Set、List、Map 和 Map.Entry 具有这样的特性。
3. **把参数转换成正确的类型。** 因为转换之前进行过 instanceof 测试，所以确保会成功。
4. **对于该类中的每个“关键（significant）”字段，检查参数中的字段是否与该对象中对应的字段相匹配。** 如果这些测试全部成功，则返回 true，否则返回 false。如果第 2 步中的类型是个借口，就必须通过接口方法访问参数中的字段；如果该类型是类，也许就能够直接访问参数中的字段，这就要取决于它们的可访问性。

&emsp;&emsp;对于既不是 float 类型也不是 double 类型的基本类型字段，可以使用`==`操作符进行比较；对于对象引用类型的字段，可以递归地调用 equals 方法；对于 float 字段，可以使用静态方法`Float.compare(float, float)`方法进行比较；对于 double 字段，使用`Double.compare(double,double)`。对 float 和 double 字段进行特殊的处理是有必要的，因为存在着 Float.NaN、-0.0f 以及类似的 double 常量；详细信息请参考 JLS 15.21.1 或者 Float.equals 的文档。虽然你可以使用静态方法 Float.equals 和 Double.equals 来比较 float 和 double 字段，但是这会在每次比较时产生自动装箱，这会导致性能不佳。 对于数组字段，请将这些指导原则应用于每个元素。 如果数组字段中的每个元素都很重要，请使用 Arrays.equals 方法中的其中一个方法进行比较。

&emsp;&emsp;有些对象引用类型的字段包含 null 可能是合法的，所有，为了避免可能导致的 NullPointerException 异常，使用静态方法`Objects.equals(Object, Object)`来检查这些字段是否相等。

&emsp;&emsp;对于有些类，例如 CaseInsensitiveString 类，字段的比较要比简单的相等性测试复杂得多。如果是这种情况，可能会希望保存该字段的一个“范式（canonical form）”，这样 equals 方法就可以根据这些范式进行低开销的精确比较，而不是高开销的非精确比较。这种方法对于不可变类（第 17 项）是最为合适的；如果对象可能发生变化，就必须保证其范式保持最新。

&emsp;&emsp;字段的比较顺序可能会影响到 equals 方法的性能。为了获得最佳的性能，你应该最先比较最有可能不一致的字段，或者是开销最低的字段，最理想的情况是两个条件同时满足的字段。你不应该去比较那些不属于对象逻辑状态的字段，例如用于同步操作的 Lock 字段。您不需要去比较那些可以从关键字段计算出来的派生字段（derived fields），但是这样做【比较派生字段】有可能提高 equals 方法的性能。如果派生字段代表了整个对象的综合描述，比较这个字段可以节省当比较失败时去比较实际数据所需要的开销【也就是说当（条件一）派生对象可以代表两个对象是否相等的时候，同时（条件二）比较这个派生字段的开销比比较实际数据的开销小的情况下，我们可以使用派生字段进行对象是否相等的比较】。例如，假设有一个 Polygon 类，并缓存了该区域。如果两个多边形【Polygon 的实例】有着不同的区域，就没有必要去比较他们的边和至高点。

5. **当你编写完 equals 方法之后，应该问你自己三个问题：它是否具有对称性？是否具有传递性？是否具有一致性？** 并且不要只是自问，还要编写单元测试来检验这些特性，除非你使用 AutoValue【谷歌的一个开源框架，下面有提到】（原文 49 页）自动生成 equals 方法，在这种情况下，您可以安全地忽略测试。如果属性无法保持，找出原因，并相应地修改 equals 方法。当然，你的 equals 方法也必须满足另外两个属性（自反性和非空性），但这两个一般是相通的（If the properties fail to hold, figure out why, and modify the equals method accordingly. Of course your equals method must also satisfy the other two properties (reflexivity and non-nullity), but these two usually take care of themselves）。

&emsp;&emsp;在这个简单的 PhoneNumber 类中显示了根据前面的诀窍构造的 equals 方法：

```java
// Class with a typical equals method
public final class PhoneNumber {
    private final short areaCode, prefix, lineNum;
    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = rangeCheck(areaCode, 999, "area code");
        this.prefix = rangeCheck(prefix, 999, "prefix");
        this.lineNum = rangeCheck(lineNum, 9999, "line num");
    }
    private static short rangeCheck(int val, int max, String arg) {
        if (val < 0 || val > max)
            throw new IllegalArgumentException(arg + ": " + val);
        return (short) val;
    }
    @Override public boolean equals(Object o) {
        if (o == this)
            return true;
        if (!(o instanceof PhoneNumber))
            return false;
        PhoneNumber pn = (PhoneNumber)o;
        return pn.lineNum == lineNum && pn.prefix == prefix && pn.areaCode == areaCode;
    }
    ... // Remainder omitted
}
```

&emsp;&emsp;下面是最后的一些说明：

- **重写 equals 方法时总要重写 hashCode 方法（第 11 项）。**
- **不要企图让 equals 方法过于智能。** 如果只是简单地测试域中的值是否相等，则不难做到遵守 equals 约定。如果想过度地去寻求各种等价关系，则很容易陷入麻烦之中。把任何任何一种别名形式考虑到等价的范围内，往往是一个坏主意。例如，File 类不应该试图把指向同一个文件的符号链接（symbolic link）当做相等的对象来看待。所幸 File 类没有这样做。
- **不要将 equals 声明中的 Object 对象替换为其他的类型。** 程序猿编写出下面这样的 equals 方法并不少见，这会使程序猿花上数小时都搞不清为什么它不能正常工作：

```java
// Broken - parameter type must be Object!
public boolean equals(MyClass o) {
    ...
}
```

&emsp;&emsp;问题在于，这个方法并没有重写 Object.equals 方法，因为它的参数应该是 Object 类型，相反，它重载了 Object.equals（第 52 项）。在原有 equals 方法的基础上，再提供一个“强类型（strongly typed）”的 equals 方法，这是无法接受的，因为它可能导致子类中的 Override 注释产生误报并提供错误的安全感（because it can cause Override annotations in subclasses to generate false positives and provide a false sense of security）【可能会导致子类中的 Override 注释在编译的时候报错】。

&emsp;&emsp;Override 注释的用法一致，就如本项中所示，可以防止犯这种错误（第 40 项）。这个 equals 方法不能编译，错误消息将会告诉你到底哪里出了问题：

```java
// Still broken, but won’t compile
@Override public boolean equals(MyClass o) {
    ...
}
```

&emsp;&emsp;编写和测试 equals（和 hashCode）方法很繁琐，结果代码很平常。 手动编写和测试这些方法的一个很好的替代方法是使用 Google 的开源 AutoValue 框架，该框架会自动为您生成这些方法，由类中的单个注释触发。 在大多数情况下，AutoValue 生成的方法与您自己编写的方法基本相同。

&emsp;&emsp;IDE 也有生成 equals 和 hashCode 方法的工具，但结果源代码比使用 AutoValue 的代码更冗长，更不易读，不会自动跟踪类中的更改，因此需要测试。 也就是说，让 IDE 生成 equals（和 hashCode）方法通常比手动实现它们更可取，因为 IDE 不会造成粗心的错误，人类也会这样做。

&emsp;&emsp;总之，不要重写 equals 方法，除非您不得不这么做：在许多情况下，从 Object 继承的实现完全符合您的要求。 如果你确实重写了 equals，请确保比较所有类的关键字段，并使用之前提到的五个诀窍对它进行测试（If you do override equals, make sure to compare all of the class’s significant fields and to compare them in a manner that preserves all five provisions of the equals contract）。

> - [第 9 项：try-with-resources 优先于 try-finally](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第02章：创建和销毁对象/第9项：try-with-resources优先于try-finally.md)
> - [第 11 项：当重写 equals 方法时也要重写 hashCode 方法](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第03章：对于所有对象都通用的方法/第11项：当重写equals方法时也要重写hashCode方法.md)
