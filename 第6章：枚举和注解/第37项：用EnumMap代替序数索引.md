### 用EnumMap代替序数索引（建议结合第二版和原书一起看）

&emsp;&emsp;有时候，你可能会见到利用ordinal方法（第35项）索引到数组或列表的代码。例如用下面这个简单的类来表示植物：

```java
class Plant {
    enum LifeCycle { ANNUAL, PERENNIAL, BIENNIAL }
    final String name;
    final LifeCycle lifeCycle;

    Plant(String name, LifeCycle lifeCycle) {
        this.name = name;
        this.lifeCycle = lifeCycle;
    }
    @Override public String toString() {
        return name;
    }
}
```

&emsp;&emsp;现在假设有一个Plant数组来表示一座花园的植物，你想要根据它们的生命周期（一年生、多年生或者两年生）进行组织之后将这些植物列出来。如果要这么做的话，需要构建三个集合，每个生命周期一个集合，并遍历整座花园，将每种植物放到相应的集合中。一些程序猿会通过将这些集合放到一个按照生命周期的序数进行索引的数组中来实现这一点。

```java
// Using ordinal() to index into an array - DON'T DO THIS!
Set<Plant>[] plantsByLifeCycle = (Set<Plant>[]) new Set[Plant.LifeCycle.values().length];
for (int i = 0; i < plantsByLifeCycle.length; i++)
    plantsByLifeCycle[i] = new HashSet<>();
for (Plant p : garden)
    plantsByLifeCycle[p.lifeCycle.ordinal()].add(p);
// Print the results
for (int i = 0; i < plantsByLifeCycle.length; i++) {
    System.out.printf("%s: %s%n", Plant.LifeCycle.values()[i], plantsByLifeCycle[i]);
}
```

&emsp;&emsp;这种方法的确可行，但是隐藏着许多问题。因为数组不能与泛型兼容（第28项），程序需要进行未受检的转换，并且不能正确无误地进行编译。因为数组不知道它的索引代表着什么，你必须手工标注（label）这些索引的输出。但是这种方法最引种的问题在于，当你访问一个按照枚举的序数进行索引的数组时，使用正确的int值就是你的职责了，int不能提供枚举的类型安全。你如果使用了错误的值，程序就会悄悄地完成错误的工作，如果幸运的话，会抛出ArrayIndexOutOfBoundException异常。

&emsp;&emsp;这有一种更好的办法可以达到同样的效果。数组实际上充当着从枚举到值的映射，因此可能还要用到Map。更具体地说，有一种非常快速的Map实现专门用于枚举键，称作java.util.EnumMap。以下就是用EnumMap改写后的程序：

```java
// Using an EnumMap to associate data with an enum
Map<Plant.LifeCycle, Set<Plant>> plantsByLifeCycle = new EnumMap<>(Plant.LifeCycle.class);
for (Plant.LifeCycle lc : Plant.LifeCycle.values())
    plantsByLifeCycle.put(lc, new HashSet<>());
for (Plant p : garden)
    plantsByLifeCycle.get(p.lifeCycle).add(p);
System.out.println(plantsByLifeCycle);
```

&emsp;&emsp;这段程序更简短、更清楚、也更加安全，运行速度方面可以与使用序数的程序相媲美。它没有不安全的转换；不必手动标注这些索引的输出，因为映射键知道如何将自身翻译成可打印字符串的枚举；计算数组索引时也不可能出错。EnumMap在运行速度方面之所以能与通过序数索引的数组相媲美，是因为EnumMap在内部使用了这种数组。但是它对程序猿隐藏了这种实现细节集Map的丰富功能和累I型那个安全与数组的快速于一身。注意EnumMap构造器采用键类型的Class对象：这是一个有限制的类型令牌（bounded type token），它提供了运行时的泛型信息（第33项）。

&emsp;&emsp;通过使用流（stream）（第45项）来管理map可以使上面的程序更加简短。这是最简单的基于流的代码，它很大程度上复制了前一个示例的行为：

```java
// Naive stream-based approach - unlikely to produce an EnumMap!
System.out.println(Arrays.stream(garden).collect(groupingBy(p -> p.lifeCycle)));
```

&emsp;&emsp;这段代码的问题在于它选择了自己的map实现，并且实际上它不是EnumMap，因此在时间和空间的性能上跟之前使用EnumMap的那个版本不匹配。要解决此问题，请使用Collectors.groupingBy的三参数形式，它允许调用者使用mapFactory参数指定地图实现：

```java
// Using a stream and an EnumMap to associate data with an enum
System.out.println(Arrays.stream(garden).collect(groupingBy(p -> p.lifeCycle, () -> new EnumMap<>(LifeCycle.class), toSet())));
```

&emsp;&emsp;在这种像小玩具的程序中不值得做这种优化，但在一个大量使用map的程序中可能是至关重要的。

&emsp;&emsp;基于流的版本的行为与EmumMap版本的行为略有不同。EnumMap版本始终为植物的每种生命周期创建一个嵌套映射，而基于流的版本仅在花园中具有该生命周期的一个或多种植物时才生成嵌套映射。因此，例如，如果花园中包含年度和多年生，但没有两年生，则plantsByLifeCycle的大小在EnumMap版本中将是三个【plantsByLifeCycle.size() = 3】，而在两个基于流的版本中为两个【plantsByLifeCycle.size() = 2】。

&emsp;&emsp;你还可能见到按照序数进行索引（两次【索引】）的数组的数组，该序数表示两个枚举值的映射。例如，下面这个程序就是使用这样一个数组将两个阶段映射到一个phase转换中（从液体到固体称作凝固，从液体到气体称作沸腾，诸如此类）。

```java
// Using ordinal() to index array of arrays - DON'T DO THIS!
public enum Phase {
    SOLID, LIQUID, GAS;
    public enum Transition {
        MELT, FREEZE, BOIL, CONDENSE, SUBLIME, DEPOSIT;
        // Rows indexed by from-ordinal, cols by to-ordinal
        private static final Transition[][] TRANSITIONS = {
            { null, MELT, SUBLIME },
            { FREEZE, null, BOIL },
            { DEPOSIT, CONDENSE, null }
        };
        // Returns the phase transition from one phase to another
        public static Transition from(Phase from, Phase to) {
            return TRANSITIONS[from.ordinal()][to.ordinal()];
        }
    }
}
```

&emsp;&emsp;这段程序可行，看起来也比较优雅，但是事实并非如此。就像之前那个比较简单的花园示例一样，编译器无法知道序数和数组索引之间的关系。如果在转换表中出了错，或者在修改Phase或者Phase.Transition枚举类型的时候忘记将它更新，程序就会在运行时失败。这种失败的形式可能为ArrayIndexOutOfBoundsException、NullPointerException或者（更糟糕的是）没有任何提示的错误行为。这张表的大小是phase个数的平方，即使非null项的数量比较少。

&emsp;&emsp;同样，利用EnumMap依然可以做得更好一些。因为每个phase转换都是通过一对phase枚举进行索引的，最好将这种关系表示为一个map，这个map的键是一个枚举（起始阶段），值为另一个map，这第二个map的键作为第二个枚举（目标阶段），它的值为结果（阶段转换）。一个phase转换所关联的两个phase，最好通过“数据与phase转换枚举之间的关联”来获取，之后用这个phase转换枚举来初始化嵌套的EnumMap。

```java
// Using a nested EnumMap to associate data with enum pairs
public enum Phase {
    SOLID, LIQUID, GAS;
    public enum Transition {
        MELT(SOLID, LIQUID), FREEZE(LIQUID, SOLID),
        BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID),
        SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID);
        private final Phase from;
        private final Phase to;
        Transition(Phase from, Phase to) {
            this.from = from;
            this.to = to;
        }
        // Initialize the phase transition map
        private static final Map<Phase, Map<Phase, Transition>> m = Stream.of(values()).collect(groupingBy(t -> t.from, () -> new EnumMap<>(Phase.class), toMap(t -> t.to, t -> t, (x, y) -> y, () -> new EnumMap<>(Phase.class))));
        public static Transition from(Phase from, Phase to) {
            return m.get(from).get(to);
        }
    }
}
```

&emsp;&emsp;初始化phase转换的map的代码看起来可能有点复杂。map的类型为Map<Phase, Map<Phase, Transition>>，which means “map from (source) phase to map from (destination) phase to transition.” 【这就不翻译了，费劲！反正就是对这个Map<Phase, Map<Phase, Transition>>的解释，懂的人自然懂！】这个映射map使用级联序列初始化两个collector。第一个collector根据phase的起点对转换进行分组，第二个collector创建一个EnumMap，其中包含从目标phase到转换的映射。第二个collector中的合并函数((x,y) -> y)并未使用；这是必需的，因为我们需要指定一个Map工厂才能获得EnumMap，而Collectors提供了可扩展的工厂（telescoping factories）。本书的前一版使用显示迭代来初始化phase转换map。代码更冗长，但可以说更容易理解。

附前一版代码（原书没有，我加的）：
```java
public enum Phase {
    SOLID, LIQUID, GAS;
    public enum Transition {
        MELT(SOLID, LIQUID), FREEZE(LIQUID, SOLID),
        BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID),
        SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID);
        private final Phase from;
        private final Phase to;
        Transition(Phase from, Phase to) {
            this.from = from;
            this.to = to;
        }
        // Initialize the phase transition map
        private static final Map<Phase, Map<Phase, Transition>> m = new EnumMap<Phase, Map<Phase, Transition>>(Phase.class);
        static {
            for(Phase p: Phase.values())
                m.put(p, new EnumMap<Phase, Transition>(Phase.class));
            for(Transition trans: Transition.values())
                m.get(trans.from).put(trans.to, trans);
        }
        public static Transition from(Phase from, Phase to) {
            return m.get(from).get(to);
        }
    }
}
```

&emsp;&emsp;现在假设想要给系统添加一个新的phase：plasma（离子）或者电离气体。只有两个转换与这个phase关联：电离化，它将气体变成离子；以及消电离化，将离子变成气体。为了更新基于数组的程序，必须给Phase添加一种新常量，给Phase.Transition添加两种新常量，用一种新的16个元素的版本取代原来9个元素的数组的数组。如果给数组添加的元素过多或过少，或者元素放置不妥当，可就麻烦了：程序可以编译，但是会在运行时失败。为了更新基于EnumMap的版本，所要做得就是必须将PLASMA添加到Phase列表，并将IONIZE(GAS，PLASMA)和DEIONIZE(PLASMA, GAS)添加到Phase中Transition的列表中：

```java
// Adding a new phase using the nested EnumMap implementation
public enum Phase {
    SOLID, LIQUID, GAS, PLASMA;
    public enum Transition {
        MELT(SOLID, LIQUID), FREEZE(LIQUID, SOLID),
        BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID),
        SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID),
        IONIZE(GAS, PLASMA), DEIONIZE(PLASMA, GAS);
        ... // Remainder unchanged
    }
}
```

&emsp;&emsp;该程序负责其他所有事情，让你几乎没有犯错的机会。在【代码】内部，map里面的map是通过一系列阵列实现的，因此你只需花费很少的空间或时间成本就可以增加【代码的】清晰度，安全性和易维护性。

&emsp;&emsp;为了简洁起见，上述示例使用null来表示不存在状态变化（其中往返是相同的）。这不是一个好习惯，很可能在运行时导致NullPointerException异常。为这个问题设计一个干净，优雅的解决方案是非常棘手的，并且生成的程序足够长，以至于它们会减损此项目中的主要成本。

&emsp;&emsp;总而言之，**最好不要用序数来索引数组，而要使用EnumMap。** 如果你所表示的这种关系是多维的，就使用EnumMap<..., EnumMap<...>>。应用程序的程序猿在一般情况下都不适用Enum.ordinal，即使要用也很少，因此这是一种特殊情况（第35项）