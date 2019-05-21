### 用enum代替int常量

&emsp;&emsp;枚举类型（enum type）是指由一组固定的常量组成合法值的类型，例如一年中的季节、太阳系中的行星或者一副扑克牌中的花色。在编程语言中还没有引入枚举类型之前，表示枚举类型的常用模式是声明一组具名的int常量，没个类型成员一个常量：

```java
    // The int enum pattern - severely deficient!
    public static final int APPLE_FUJI = 0;
    public static final int APPLE_PIPPIN = 1;
    public static final int APPLE_GRANNY_SMITH = 2;
    public static final int ORANGE_NAVEL = 0;
    public static final int ORANGE_TEMPLE = 1;
    public static final int ORANGE_BLOOD = 2;
```

&emsp;&emsp;这种方法称作int枚举模式（int enum type），存在着诸多不足。它在类型安全方面没有任何帮助，表达能力不足。如果你将apple传到想要orange的方法中，编译器也不会出现警告，还可以用==操作符将apple和orange进行对比，甚至更糟糕：

```java
// Tasty citrus flavored applesauce!
int i = (APPLE_FUJI - ORANGE_TEMPLE) / APPLE_PIPPIN;
```

&emsp;&emsp;注意没个apple常量的名称都以APPLE_作为前缀，每个orange常量则以ORANGE_作为前缀。这是因为Java没有为int枚举组提供命名空间。当两个int枚举具有相同的命名常量时，前缀可以防止名称发生冲突，例如在`ELEMENT_MERCURY`和`PLANET_MERCURY`之间。

&emsp;&emsp;采用int枚举模式的程序是十分脆弱的。因为int枚举是常量变量(constant variables)[JLS, 4.12.4]，所以它们的int值被编译到使用它们的客户端[JLS, 13.1]。如果与枚举常量关联的int发生了变化，客户端就必须重新编译。如果没有重新编译，程序还是可以运行，但是它们的行为就是不确定的。

&emsp;&emsp;将int枚举常量翻译成可打印的字符串，并没有很便利的方法。如果将这种常量打印出来或者从调试器中将它显示出来，你所见到的就是一个数字，这没有太大的用处。要遍历一个组中的所有int枚举常量，甚至获得int枚举组的大小，这些都没有很可靠的方法。

&emsp;&emsp;你还可能碰到这种模式的变体，在这种模式中使用的是String常量，而不是int常量。这样的变体被称作String枚举模式，同样也是我们最不期望的。虽然它为这些常量提供了可打印字符串，但是它会导致性能问题，因为它依赖于字符串的比较操作。更糟糕的是，它会导致初级用户把字符串常量硬编码到客户端代码中，而不是使用适当的域（field）名。如果这样的硬编码字符串常量中包含书写错误，那么，这样的错误在编译时不会被检测到，但是在运行的时候却会报错。

&emsp;&emsp;幸运的是，Java提供了一种可以替代的解决方案，可以避免int和String枚举模式的缺点，并提供许多额外的好处。这就是枚举类型[JLS, 8.9]。下面以最简单的形式演示了这种模式：

```java
public enum Apple { FUJI, PIPPIN, GRANNY_SMITH }
public enum Orange { NAVEL, TEMPLE, BLOOD }
```

&emsp;&emsp;表面上看，这些枚举类型与其他语言中的没什么两样，例如C、C++和C#，但是实际上并非如此。Java的枚举类型是功能十分齐全的类，功能比其他语言中的对等物要更强大得多，Java的枚举本质上是int值。

&emsp;&emsp;Java枚举类型背后的基本想法非常简单：它们就是通过公有的静态final域为每个枚举常量导出的实例的类。因为没有可以访问的构造器，枚举类型是真正的final。因为客户端既不能创建枚举类型的实例，也不能对它进行扩展，因此很可能没有实例，而只有声明过的枚举常量。换句话说，枚举类型是实例是受控制的（原书第6页）。它们是单例（Singleton）的泛型化（第3项），本质上是单元素枚举。

&emsp;&emsp;枚举提供了编译时的类型安全。如果声明一个参数的类型为Apple，就可以保证，被传到该参数上的任何非null的对象引用一定属于三个有效的Apple值之一。试图传递类型错误的值时，会导致编译时错误，就像试图将某种枚举类型的表达式赋给另一种枚举类型的变量，或者试图利用==操作符比较不同枚举类型的值一样。

&emsp;&emsp;包含同名常量的多个枚举类型可以在一个系统中和平共处，因为每个类型都有自己的命名空间，你可以增加或者重新排列枚举类型中的常量，而无需重新编译它的客户端代码，因为导出常量的域在枚举类型和它的客户端之间提供了一个隔离层：常量值并没有被编译到客户端代码中，而是在int枚举模式中，最终，可以通过调用toString方法，将枚举转换成可打印的字符串。

&emsp;&emsp;除了完善了int枚举模式的不足之外，枚举类型还允许添加任意的方法和域，并实现任意的接口。它们提供了所有Object方法（第3章），实现了Comparable（第14项）和Serializable（第12章），并针对枚举类型的可任意改变性设计了序列化方式。

&emsp;&emsp;那么我们为什么要将方法或者域添加到枚举类型中呢？首先，你可能是想将数据与它的常量关联起来。例如，一个能够返回水果颜色或者返回水果图片的方法，对于我们的Apple和Orange类型来说可能很有好处。你可以利用任何适当的方法来增强枚举类型。枚举类型可以先作为枚举常量的一个简单集合，随着时间的推移再演变成为全功能的抽象。

&emsp;&emsp;举个有关枚举类型的好例子，比如太阳系中的8颗星星。每颗行星都有质量和半径，通过这两个属性可以计算出它的表面重力。从而给定物体的质量，就可以计算出一个物体在行星表面上的重量。下面就是这个枚举。没个枚举常量后面括号中的数值就是传递给构造器的参数。在这个例子中，它们就是行星的质量和半径：

```java
// Enum type with data and behavior
public enum Planet {
    MERCURY(3.302e+23, 2.439e6),
    VENUS (4.869e+24, 6.052e6),
    EARTH (5.975e+24, 6.378e6),
    MARS (6.419e+23, 3.393e6),
    JUPITER(1.899e+27, 7.149e7),
    SATURN (5.685e+26, 6.027e7),
    URANUS (8.683e+25, 2.556e7),
    NEPTUNE(1.024e+26, 2.477e7);
    private final double mass; // In kilograms
    private final double radius; // In meters
    private final double surfaceGravity; // In m / s^2
    // Universal gravitational constant in m^3 / kg s^2
    private static final double G = 6.67300E-11;
    // Constructor
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
        surfaceGravity = G * mass / (radius * radius);
    }
    public double mass() { return mass; }
    public double radius() { return radius; }
    public double surfaceGravity() { return surfaceGravity; }
    public double surfaceWeight(double mass) {
        return mass * surfaceGravity; // F = ma
    }
}
```

&emsp;&emsp;编写一个像Planet这样的枚举类型并不难。**为了将数据与枚举常量关联起来，得声明实例域，并编写一个带有数据并将数据保存在域中的构造器。** 枚举天生就是不可变的，因此所有的域都应该为final的（第17项）。它们可以是公有的，但最好将它们做成是私有的，并提供公有的访问方法（第16项）。在Planet这个示例中，构造器还计算和保存表面重力，但这正是一种优化。每当surfaceWeight方法用到重力时，都会根据质量和半径重新计算，并返回它在该常量所表示的行星上的重量。

&emsp;&emsp;虽然Planet枚举很简单，它的功能却强大得出奇。下面是一个简短的程序，根据某个物体在地球上的重量（以任何单位），打印出一张很棒的表格，显示出该物体在所有8颗行星上的重量（用相同的单位）：

```java
public class WeightTable {
    public static void main(String[] args) {
        double earthWeight = Double.parseDouble(args[0]);
        double mass = earthWeight / Planet.EARTH.surfaceGravity();
        for (Planet p : Planet.values())
            System.out.printf("Weight on %s is %f%n", p, p.surfaceWeight(mass));
        }
}
```

&emsp;&emsp;注意Planet就像所有的枚举一样，它有一个静态的values方法，按照声明顺序返回它的值数组。还要注意toString方法返回没个枚举值的声明名称，使得println和printf的打印变得更加容易。如果你不满意这种字符串表示法，可以通过覆盖toString方法对它进行修改。下面就是用命令行参数185运行这个WeightTable程序（没有重写toString方法）时的结果：

```
Weight on MERCURY is 69.912739
Weight on VENUS is 167.434436
Weight on EARTH is 185.000000
Weight on MARS is 70.226739
Weight on JUPITER is 467.990696
Weight on SATURN is 197.120111
Weight on URANUS is 167.398264
Weight on NEPTUNE is 210.208751
```

&emsp;&emsp;直到2006年，在枚举类型被添加到Java之后两年，冥王星才是一颗行星。这提出了一个问题“当你从枚举类型中删除一个元素会发生什么？”答案是任何不引用被删除的元素的客户端程序都将继续正常工作。。因此，例如，我们的WeightTable程序只会打印一个少一行的表。什么是客户端程序引用被删除的元素（在本例中为Planet.Pluto)？如果重新编译客户端程序，编译将失败，并在引用以前行星的【代码】行中显示有用的错误信息；如果你无法重新编译客户端，它将在运行时从此引发一个有用的异常。这可能是你希望的最佳行为，远远优先于你使用int枚举模式获得的行为。

&emsp;&emsp;与枚举常量关联的有些行为，可能只需要在定义了枚举的类或者包中。这种行为最好被实现成私有的或者包级私有的方法。然后，每个枚举常量都带有一组隐蔽的行为，这使得包含该枚举的类或者包在遇到这种常量时都可以做出适当的反应。就像其他的类一样，除非迫不得已要将枚举方法导出至它的客户端，否则都应该将它声明为私有的，如有必要，则声明为包级私有的（第15项）。

&emsp;&emsp;如果一个枚举具有普遍适用性，它就应该成为一个顶层类（top-level class）；如果它只是被用在一个特定的顶层类中，他就应该成为该顶层类的一个成员类（第24项）。例如，java.math.RoundingMode枚举表示十进制小数的舍入模式（rounding mode）。这些舍入模式用于BigDecimal类，但是它们提供了一个非常有用的抽象，这种抽象本质上又不属于BigDecimal类。通过使用RoundingMode变成一个顶层类，库的设计者鼓励任何需要舍入模式的程序猿重用这个枚举，从而增强API之间的一致性。

&emsp;&emsp;Planet示例中所示的方法对于大多数枚举类型来说就足够了，但你有时候会需要更多的方法。每个Planet常量都关联了不同的数据，但你有时候需要将本质上不同的行为（behavior）与每个常量关联起来。例如，假设你在编写一个枚举类型，来表示计算器的四大基本操作（即加减乘除），你想要提供一个方法来执行每个常量所表示的算术运算。有一种方法是通过启用枚举的值来实现：

```java
// Enum type that switches on its own value - questionable
public enum Operation {
    PLUS, MINUS, TIMES, DIVIDE;
    // Do the arithmetic operation represented by this constant
    public double apply(double x, double y) {
        switch(this) {
            case PLUS: return x + y;
            case MINUS: return x - y;
            case TIMES: return x * y;
            case DIVIDE: return x / y;
        }
        throw new AssertionError("Unknown op: " + this);
    }
}
```

&emsp;&emsp;这段代码是可行的，但是不太好看。如果没有throw语句，他就不能进行编译，虽然从技术角度来看代码的结束部分是可以执行到的，但是实际上是不可能执行到这行代码的[JLS, 14.21]。更糟糕的是，这段代码很脆弱。如果你添加了新的枚举常量，却忘记给switch添加相应的条件，枚举仍然是可以编译的，但是当你试图运用新的运算时，就会运行失败。

&emsp;&emsp;幸运的是，有一种更好的方法可以将不同的行为与每个枚举常量关联起来：在枚举类型中声明一个抽象的apply方法，并在特定于常量的类主体（constant-specific class body）中，用具体方法覆盖每个常量的抽象apply方法。这种方法被称作特定于常量的方法实现（constant-specific method implementations）：

```java
// Enum type with constant-specific method implementations
public enum Operation {
    PLUS {public double apply(double x, double y){return x + y;}},
    MINUS {public double apply(double x, double y){return x - y;}},
    TIMES {public double apply(double x, double y){return x * y;}},
    DIVIDE {public double apply(double x, double y){return x / y;}};
    public abstract double apply(double x, double y);
}
```

&emsp;&emsp;如果你在Operation的第二个版本中添加新的常量，你就不可能忘记提供apply方法，因为该方法就紧跟在每个常量声明之后。即使你真的忘记了，编译器也会提醒你，因为枚举类型中的抽象方法必须被它所有常量中的具体方法所覆盖。

&emsp;&emsp;特定于常量的方法实现可以与特定于常量的数据结合起来。例如，下面的Operation覆盖了toString来返回通常与该操作相关联的符号：

```java
// Enum type with constant-specific class bodies and data
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

&emsp;&emsp;在有些情况下，在枚举中覆盖toString非常有用。例如，上述的toString实现使得打印算术表达式变得非常容易，如这段小程序所示：

```java
public static void main(String[] args) {
    double x = Double.parseDouble(args[0]);
    double y = Double.parseDouble(args[1]);
    for (Operation op : Operation.values())
        System.out.printf("%f %s %f = %f%n", x, op, y, op.apply(x, y));
}
```

&emsp;&emsp;用2和4作为命令行参数运行这段程序，会输出：

```
2.000000 + 4.000000 = 6.000000
2.000000 - 4.000000 = -2.000000
2.000000 * 4.000000 = 8.000000
2.000000 / 4.000000 = 0.500000
```

&emsp;&emsp;枚举类型有一个自动产生的valueOf(String)方法，它将常量的名字转变成常量本身。如果在枚举类型中覆盖toString，要考虑编写一个fromString方法，将定制的字符串表示法变回相应的枚举。下列代码（适当地改变了类型名称）可以为任何枚举完成这一技巧，只要每个常量都有一个独特的字符串表示法：

```java
// Implementing a fromString method on an enum type
private static final Map<String, Operation> stringToEnum = Stream.of(values()).collect(toMap(Object::toString, e -> e));
// Returns Operation for string, if any
public static Optional<Operation> fromString(String symbol) {
    return Optional.ofNullable(stringToEnum.get(symbol));
}
```

&emsp;&emsp;注意，在枚举常量被创建之后，Operation常量就会被放入一个已经初始化了的静态域stringToEnum的map中。前面的代码使用了一个stream（第7章），而不是values()方法返回的数组（The previous code uses a stream (Chapter 7) over the array returned by the values() method）；在Java 8之前，我们将创建一个空的哈希映射（hash map），并遍历values数组，将字符串跟枚举的映射插入一个map中，如果你愿意，仍然可以这样做。但请注意，试图使每个常量都从自己的构造器将自身放入到map中是不会起作用的。这会导致编译时错误，这是好事，因为如果它是合法的，它将在运行时导致NullPointerException。除常量变量外不允许枚举构造函数访问枚举的静态字段（第34项）。这种限制是必要的，因为枚举构造函数运行时尚未初始化静态字段。这种限制的一个特例是枚举常量不能从它们的构造函数中相互访问（A special case of this restriction is that enum constants cannot access one another from their constructors.）。

&emsp;&emsp;另外请注意，fromString方法返回Optional<String>。这表示允许该方法传入的字符串不表示有效操作，并且它强制客户端面对这种可能性（第55项）。

&emsp;&emsp;特定于常量的方法实现有一个美中不足的地方，它们使得在枚举常量中共享代码变得更加困难了。例如，考虑用一个枚举表示薪资包中的工作天数。这个枚举有一个方法，根据给定某工人的基本工资（按小时）和当天工作的分钟数计算当天工人的工资。在五个工作日，任何超过正常班次的工作都会产生加班费; 在两个周末的日子里，所有工作都会产生加班费。使用switch语句，通过将多个案例标签应用于两个代码片段中的每一个，可以轻松地进行此计算：

```java
// Enum that switches on its value to share code - questionable
enum PayrollDay {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY;
    private static final int MINS_PER_SHIFT = 8 * 60;
    int pay(int minutesWorked, int payRate) {
        int basePay = minutesWorked * payRate;
        int overtimePay;
        switch(this) {
            case SATURDAY: case SUNDAY: // Weekend
                overtimePay = basePay / 2;
            break;
            default: // Weekday
                overtimePay = minutesWorked <= MINS_PER_SHIFT ? 0 : (minutesWorked - MINS_PER_SHIFT) * payRate / 2;
        }
        return basePay + overtimePay;
    }
}
```

&emsp;&emsp;毫无疑问，这段代码时简洁的，但从维护的角度来看，它是非常危险的。假设将一个元素添加到该枚举中，或许是一个表示假期天数的特殊值，但是忘记给switch语句添加相应的case。程序依然可以编译，但pay方法会悄悄地将假期的工资计算成与正常工作日的相同。

&emsp;&emsp;为了利用特定于常量的方法实现安全地执行工资计算，你可能必须重复计算没个常量的加班工资，或者将计算移到两个辅助方法中（一个用来计算工作日，一个用来计算双休日），并从没个常量调用相应的辅助方法。这任何一种方法都会长生相当数量的样板代码，结果降低了可读性，并增加了出错的机率。

&emsp;&emsp;通过用计算工作日加班工资的具体方法代替PayrollDay中抽象的overtimePay方法，可以减少样板代码。这样，就只有双休日必须覆盖该方法了。但是这样也有着与switch语句一样的不足：如果又增加了一天而没有覆盖overtimePay方法，就会悄悄地延续工作日的计算。

&emsp;&emsp;你真正想要的就是每当添加一个枚举常量时，就强制选择一种加班报酬。幸运的是，有一种很好的方法可以实现这一点。这种想法就是将加班工资计算移到一个私有的嵌套枚举中，将这个枚举策略（strategy enum）的实例传到PayrollDay枚举的构造器中。之后PayrollDay枚举将加班工资计算委托给策略枚举，PayrollDay中就不需要switch语句或者特定于常量的方法实现了。虽然这种模式没有switch语句那么简洁，但更加安全，也更加灵活：

```java
// The strategy enum pattern
enum PayrollDay {
    MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY,
    SATURDAY(PayType.WEEKEND), SUNDAY(PayType.WEEKEND);
    private final PayType payType;
    PayrollDay(PayType payType) { this.payType = payType; }
    PayrollDay() { this(PayType.WEEKDAY); } // Default
    int pay(int minutesWorked, int payRate) {
        return payType.pay(minutesWorked, payRate);
    }
    // The strategy enum type
    private enum PayType {
        WEEKDAY {
            int overtimePay(int minsWorked, int payRate) {
                return minsWorked <= MINS_PER_SHIFT ? 0 : (minsWorked - MINS_PER_SHIFT) * payRate / 2;
            }
        },
        WEEKEND {
            int overtimePay(int minsWorked, int payRate) {
                return minsWorked * payRate / 2;
            }
        };
        abstract int overtimePay(int mins, int payRate);
        private static final int MINS_PER_SHIFT = 8 * 60;
        int pay(int minsWorked, int payRate) {
        int basePay = minsWorked * payRate;
            return basePay + overtimePay(minsWorked, payRate);
        }
    }
}
```

&emsp;&emsp;如果枚举中的switch语句不是在枚举中实现特定于常量的行为的一种很好的选择，那么它们还有什么用处呢？**枚举中的switch语句适合给外部的枚举类型增加特定于常量的行为。** 例如，假设Operation枚举不受你的控制，你希望它有一个实例方法来返回每个运算的反运算。你可以用下列静态方法模拟这种效果：

```java
// Switch on an enum to simulate a missing method
public static Operation inverse(Operation op) {
    switch(op) {
        case PLUS: return Operation.MINUS;
        case MINUS: return Operation.PLUS;
        case TIMES: return Operation.DIVIDE;
        case DIVIDE: return Operation.TIMES;
        default: throw new AssertionError("Unknown op: " + op);
    }
}
```

&emsp;&emsp;如果方法根本不属于枚举类型，那么你应该在可控制的枚举类型上使用该技巧。该方法可能有某些用途，但不足以证明它应该包含在枚举类型中。

&emsp;&emsp;一般来说，枚举在性能上与int常量差不多。枚举的一个小小的性能缺点就是在加载和初始化枚举类型会有空间和时间成本，但在实践中不必太在意。

&emsp;&emsp;那么什么时候应该使用枚举呢？**只要有一组在编译时就是已知的成员常量，就可以使用枚举。** 当然，这包括“自然枚举类型”，例如，行星，星期几和棋子。但也包括其他集合，您可以在编译时了解所有可能的值，例如，菜单上的选项，操作代码和命令行标志。**枚举类型中的常量集不必一直保持固定。** 枚举功能专门设计用于允许枚举类型的二进制兼容演变。

&emsp;&emsp;总之，枚举类型跟int常量相比，优点是显而易见的。枚举更具有可读性，更安全，更强大。许多枚举不需要显示构造函数或成员，但许多枚举则受益于将每个常量与数据【枚举的成员】相关联并提供这个数据影响行为的方法。在这种相对少见的情况下，特定于常量的方法要优先于启用自有值的枚举。如果多个枚举常量同时共享相同的行为，则考虑策略枚举。