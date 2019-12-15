## 利用有限制通配符来提升 API 的灵活性

&emsp;&emsp;如 28 项所述，参数化类型是不可变的（invariant）。换句话说，对于任何两个截然不同的类型 Type1 和 Type2 而言，List<Type1>既不是 List<Type2>的子类型，也不是它的超类型。虽然 List<String>不是 List<Object>的子类型，这与直觉相悖，但是实际上很有意义。你可以将任何对象放进一个 List<Object>中，却只能将字符串放进 List<String>中。因为 List<String>不能完成 List<Object>所能做的所有事情，因此它不是子类型（由 Liskov 替换主体，第 10 项）。

&emsp;&emsp;有时候，我们需要的灵活性要比不可变类型所能提供的更多，考虑第 29 项中的 Stack 类，下面是它的公共 API：

```java
public class Stack<E> {
    public Stack();
    public void push(E e);
    public E pop();
    public boolean isEmpty();
}
```

&emsp;&emsp;假设我们想要增加一个方法，让它按顺序将一系列的元素全部放到堆栈中。这是第一次尝试，如下：

```java
// pushAll method without wildcard type - deficient!
public void pushAll(Iterable<E> src) {
    for (E e : src)
        push(e);
}
```

&emsp;&emsp;这个方法编译时正确无误，但是并非尽如人意。如果 Iterable src 的元素类型与堆栈的完全匹配，就没有问题。但是假如有一个 Stack<Number>，并且调用了 push(intVal)，这里的 intVal 就是 Integer 类型。这是可以的，因为 Integer 是 Number 的一个子类型。因此从逻辑上来说，下面这个方法应该也可以：

```java
Stack<Number> numberStack = new Stack<>();
Iterable<Integer> integers = ... ;
numberStack.pushAll(integers);
```

&emsp;&emsp;但是，如果尝试这么做，就会得到下面的错误信息，因为如前所述，参数化类型是不可变的：

```java
StackTest.java:7: error: incompatible types: Iterable<Integer>
cannot be converted to Iterable<Number>
        numberStack.pushAll(integers);
                            ^
```

&emsp;&emsp;幸运的是，有一种解决办法。Java 提供了一种特殊的参数化类型，称作有限制的通配符类型（bounded wildcard type ），来处理类似的情况。pushAll 的输入参数类型不应该为“E 的 Iterable 接口”，而应该为“E 的某个子类型的 Iterable 接口”，有一个通配符类型证符合此意：Iterable<? Extends E>。（使用关键字 extends 有些误导：回忆以下第 29 项中的说法，确定子类型（subtype）后，每个类型便都是自身的子类型，即便它没有将自身扩展。）我们修改以下 pushAll 来使用这个类型：

```java
// Wildcard type for a parameter that serves as an E producer
public void pushAll(Iterable<? extends E> src) {
    for (E e : src)
        push(e);
}
```

&emsp;&emsp;这么修改了之后，不仅 Stack 可以正确无误地编译，没有通过初始化的 pushAll 声明进行编译的客户端代码也一样可以。因为 Stack 及其客户端正确无误地进行了编译，你就知道一切都是类型安全的了。

&emsp;&emsp;现在假设想要编写一个 popAll 方法，使之与 pushAll 方法相呼应。popAll 方法从堆栈中弹出每个元素，并将这些元素添加到指定的集合中。初次尝试编写的 popAll 方法可能像下面这样：

```java
// popAll method without wildcard type - deficient!
public void popAll(Collection<E> dst) {
    while (!isEmpty())
        dst.add(pop());
}
```

&emsp;&emsp;如果目标集合的元素类型与堆栈的完全匹配，这段代码编译时还是会正确无误，运行得很好。但是，也并不意味着尽如人意。假设你有一个 Stack<Number>和类型 Object 的变量。如果从堆栈中弹出一个元素，并将它保存在该变量中，它的编译和运行都不会出错，那你为何不能也这么做呢？

```java
Stack<Number> numberStack = new Stack<Number>();
Collection<Object> objects = ... ;
numberStack.popAll(objects);
```

&emsp;&emsp;如果试着用上述的 popAll 版本编译这段客户端代码，就会得到一个非常类似于第一次用 pushAll 时所得到的错误：Collection<Object>不是 Collection<Number>的子类型。这次，通配符类型同样提供了一种解决办法。popAll 的输入参数类型不应该为“E 的集合”，而应该为“E 的某种超类的集合”（这里的超类是确定的，因此 E 是它自身的一个超类型[JLS, 4.10]）。仍然有一个通配符类型证实符合此意：Collection<? super E>。让我们修改 popAll 来使用它：

```java
// Wildcard type for parameter that serves as an E consumer
public void popAll(Collection<? super E> dst) {
    while (!isEmpty())
        dst.add(pop());
}
```

&emsp;&emsp;做了这个变动之后，Stack 和客户端代码就都可以正确无误地编译了。

&emsp;&emsp;结论很明显。**为了获得最大限度的灵活性，要在表示生产者或者消费者的输入参数上使用通配符类型。** 如果某个输入参数既是生产者，又是消费者，那么通配符类型对你就没有什么好处了：因为你需要的是严格的类型匹配，这是不用任何通配符得到的。

&emsp;&emsp;下面的助记符便于让你记住要使用哪种通配符类型：

&emsp;&emsp;**PECS 表示 producer-extends, consumer-super**

&emsp;&emsp;换句话说，如果参数化类型表示一个 T 生产者，就使用<? extends T>；如果它表示一个 T 消费者，就使用<? super T>。在我们的 Stack 示例中，pushAll 的 src 参数产生 E 实例供 Stack 使用，因此 src 相应的类型为 Iterable<? extends E>；popAll 的 dst 参数通过 Stack 消费 E 实例，因此 dst 相应的类型为 Collection<? super E>。PECS 这个助记符突出了使用通配符类型的基本原则。Naftalin 和 Wadler 称之为*Get and Put Principle*[Naftalin 07,2.4]。

&emsp;&emsp;记住这个助记符，让我们看看本章前面的一些方法和构造函数声明。 第 28 项中的 Chooser 构造函数具有以下声明：

```java
public Chooser(Collection<T> choices)
```

&emsp;&emsp;此构造函数仅使用集合选项生成类型为 T 的值（并将其存储以供之后使用），因此其声明应使用 extends T 的通配符类型。这是改造后的构造函数声明：

```java
// Wildcard type for parameter that serves as an T producer
public Chooser(Collection<? extends T> choices)
```

&emsp;&emsp;这一变化实际上有什么区别吗？事实上，的确有区别。假设你有一个 List<Integer>，并且你想将其传递给 Chooser<Number>的构造函数。 这不会使用原始声明进行编译，但是一旦将有限制的通配符类型添加到声明中，它就会执行。

&emsp;&emsp;现在让我们看看第 30 项中的 union 方法。下面是声明：

```java
public static <E> Set<E> union(Set<E> s1, Set<E> s2)
```

&emsp;&emsp;s1 和 s2 这两个参数都是 E 的生产者，所以根据 PECS，这个声明应该是：

```java
public static <E> Set<E> union(Set<? extends E> s1, Set<? extends E> s2)
```

&emsp;&emsp;注意返回类型仍然是 Set<E>。**不要用通配符类型作为返回类型。** 除了为用户提供额外的灵活性之外，它还会强制用户在客户端代码中使用通配符类型。通过修改之后，此代码将干净地编译（With the revised declaration, this code will compile cleanly）：

```java
Set<Integer> integers = Set.of(1, 3, 5);
Set<Double> doubles = Set.of(2.0, 4.0, 6.0);
Set<Number> numbers = union(integers, doubles);
```

&emsp;&emsp;如果使用得当，通配符类型对于类的用户来说几乎是无形的。它们使方法能够接受它们应该接受的参数，并拒绝那些应该拒绝的参数。**如果类的使用者必须考虑通配符类型，类的 API 或许就会出错。**

&emsp;&emsp;在 Java 8 之前，类型推导（type inference）规则不够聪明，无法处理以前的代码片段，这需要编译器使用上下文指定的返回类型（或目标类型）来推断 E 的类型。union 调用的目标类型前面显示的是 Set<Number>。如果你尝试在早期版本的 java 中编译片段（适当替换 Set.of 工厂((with an appropriate replacement for the Set.of factory)），你将收到一个很长的，错综复杂的错误消息，如下所示：

```java
Union.java:14: error: incompatible types
        Set<Number> numbers = union(integers, doubles);
                                  ^
 required: Set<Number>
 found: Set<INT#1>
 where INT#1,INT#2 are intersection types:
    INT#1 extends Number,Comparable<? extends INT#2>
    INT#2 extends Number,Comparable<?>
```

&emsp;&emsp;幸运的是，有一种办法可以处理这种错误。如果编译器不能推断你希望它拥有的类型，可以通过一个*显示的类型参数（explicit type argument）* [JLS, 15.12]来告诉它要使用哪种类型。甚至在 Java 8 中引入目标类型之前，这不是你经常需要做的事情，这很好，因为显式类型参数并不是很好。通过添加显式类型参数，如此处所示，代码片段在 Java 8 之前的版本中准确无误地编译：

```java
// Explicit type parameter - required prior to Java 8
Set<Number> numbers = Union.<Number>union(integers, doubles);
```

&emsp;&emsp;接下来，我们把注意力转向第 30 项中的 max 方法。以下是初始的声明：

```java
public static <T extends Comparable<T>> T max(List<T> list)
```

&emsp;&emsp;下面是修改过的使用通配符类型的声明：

```java
public static <T extends Comparable<? super T>> T max( List<? extends T> list)
```

&emsp;&emsp;为了从初始声明中得到修改后的版本，要应用 PECS 转换两次。最直接的是运用到参数 list。它产生 T 实例，因此将类型从 List<T>改成 List<? extends T>。更灵活的是运用到类型参数 T。这是我们第一次见到将通配符运用到类型参数。最初 T 被指定用来扩展 Comparable<T>，但是 T 的 comparable 消费 T 实例（并产生表示顺序关系的整值）。因此，参数化；类型 Comparable<T>被有限制通配符类型 Comparable<? super T>取代。comparables 始终是消费者，因此**使用时始终应该是 Comparable<? super T>优先于 Comparable<\T>。** 对于 comparator 也一样，因此**使用时始终应该是 Comparator<? super T>优先于 Comparator<\T>。**

&emsp;&emsp;修改过的 max 声明可能是整本书中最复杂的方法声明了。所增加的复杂代码真的起作用了么？是的，起作用了。下面是一个简单的列表实例，在初始的声明中不允许这样，修改过的版本则可以：

```java
List<ScheduledFuture<?>> scheduledFutures = ... ;
```

&emsp;&emsp;不能将处事方法声明运用给这个列表的原因在于 ScheduledFuture 没有实现 Comparable<ScheduledFuture>接口。相反，它是扩展 Comparable<Delayed>接口的 Delayed 接口的子接口。换句话说，ScheduledFuture 实例并非只能与其他 ScheduledFuture 实例相比较；它可以与任何 Delayed 实例相比较，这就足以导致初始声明时就会被拒绝。更一般的说，通配符需要支持不直接实现 Comparable（或 Comparator）类型的类型（支持扩展类型）（More generally, the wildcard is required to support types that do not implement Comparable (or Comparator) directly but extend a type that does.）

&emsp;&emsp;还有一个与通配符相关的话题需要讨论。类型参数和通配符之间存在二义性，可以使用其中一个声明许多方法。例如：以下是可能的两种静态方法声明，来交换列表中两个索引的项目。第一个使用无限制类型参数（第 30 项），第二个使用无限制通配符：

```java
// Two possible declarations for the swap method
public static <E> void swap(List<E> list, int i, int j);
public static void swap(List<?> list, int i, int j);
```

&emsp;&emsp;你更喜欢这两种方法中的哪一种呢？为什么？在公共 API 中，第二种更好一些，因为它更简单。将它传到一个列表中————任何列表————方法就会交换被索引的元素。不用担心类型参数。不用担心类型参数。一般来说，**如果类型参数只在方法声明中出现一次，就可以用通配符取代它。** 如果是无限制的类型参数，使用无限制的通配符取代它；如果是有限制的类型参数，就用有限制的通配符取代它。

&emsp;&emsp;将第二中声明用于 swap 方法会有一个问题，它优先使用通配符而非类型参数：下面这个简单的实现都不能编译：

```java
public static void swap(List<?> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));
}
```

&emsp;&emsp;试着编译时会产生这条没有什么用处的错误消息：

```java
Swap.java:5: error: incompatible types: Object cannot be
converted to CAP#1
        list.set(i, list.set(j, list.get(i)));
                                        ^
 where CAP#1 is a fresh type-variable:
 CAP#1 extends Object from capture of ?
```

&emsp;&emsp;不能将元素放回到刚刚从中取出来的列表中，这似乎不太对劲。问题在于 list 的类型为 List<?>，你不能把null之外的任何值放到List<?>中。幸运的是，有一种方式可以实现这个方法，无需求助于不安全的转换或者原生态类型（raw type）。这种想法就是编写一个私有的辅助方法来捕捉通配符类型。为了捕捉类型，辅助方法必须是泛型方法，像下面这样：

```java
public static void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);
}

// Private helper method for wildcard capture
private static <E> void swapHelper(List<E> list, int i, int j) {
    list.set(i, list.set(j, list.get(i)));
}
```

&emsp;&emsp;swapHelper 方法知道 list 是一个 List<E>。因此，它知道从这个列表中去除的任何值均为 E 类型，并且知道将 E 类型的任何值放进列表都是安全的。swap 这个有些费解的实现编译起来却是正确无误的。它允许我们导出 swap 这个比较好的基于通配符的声明，同时在内部利用更加复杂的泛型方法。swap 方法的客户端不一定要面对更加复杂得 swapHelper 声明，但是它们的确从中受益。值得注意的是，辅助方法恰好具有我们认为对公共方法过于复杂的签名。

&emsp;&emsp;总而言之，在 API 中使用通配符类型虽然比较需要技巧，但是使 API 变得灵活得多。如果编写的是将被广泛使用的类库，则一定要适当地利用通配符类型。记住基本的原则：producer-extends, consumer-super (PECS)。还要记住所有的 comparable 和 comparator 都是消费者。

> - [第 30 项：优先考虑泛型方法](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第05章：泛型/第30项：优先考虑泛型方法.md)
> - [第 32 项：谨慎并用泛型和可变参数](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第05章：泛型/第32项：明智地结合泛型和可变参数.md)
