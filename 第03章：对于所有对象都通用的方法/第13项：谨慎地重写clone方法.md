## 谨慎地重写clone方法

&emsp;&emsp;Cloneable接口的目的是作为对象的一个mixin接口（mixin interface）（第20项），表明这样的类【的对象】是允许克隆的。遗憾的是，它没有达到这个目的。它的主要缺陷是缺少克隆方法，而Object的clone方法【的访问权限】是受保护【protect】的， 如果不采用反射（第65项），就不能仅仅因为它实现了Cloneable而在对象上调用clone方法。即使是反射调用也可能失败，因为无法保证对象具有可访问的clone方法。尽管存在这样那样的缺陷，这项设施仍然被广泛地使用着，因此值得我们去理解它。该项告诉您如何实现良好的clone方法，并讨论何时适合这样做，并提供替代方法。

&emsp;&emsp;既然Cloneable并没有包含任何方法，那么它到底有什么作用呢？它决定了Object中受保护的clone方法实现的行为：如果一个类实现了Cloneable，Object的clone方法将逐个返回该对象的拷贝字段，否则就会抛出CloneNotSupportedException。这是接口的一种极端非典型的用法，也不值得效仿。通常情况下，实现接口是为了表明类可以为它的客户做些什么。在这种情况下，它会修改超类上受保护方法的行为。

&emsp;&emsp;尽管规范中没有表明，**实际上，实现Cloneable的类应该提供一个功能正常的公共克隆方法。** 为了达到这个目的，类和它的所有超类都必须遵守一个相当复杂的、不可实施的，并且基本上没有文档说明的协议。由此得到一种脆弱的、危险的、语言之外的（extralinguistic）机制：无需调用构造器就可以创建对象。

&emsp;&emsp;clone方法的通用约定是非常脆弱的，下面是从Object规范中复制的：

&emsp;&emsp;创建和返回该对象的一个拷贝。这个“拷贝”的精确含义取决于该对象的类。一般的含义是，对于任何对象x，表达式`x.clone() != x`将会是true，并且，表达式`x.clone().getClass() == x.getClass()`将会返回true，但这些都不是绝对的要求。虽然通常情况下，表达式`x.clone().equals(x)`将会是true，但是，这也不是一个绝对的要求。

&emsp;&emsp;按照惯例，此方法返回的对象应该通过调用super.clone来获取。 如果一个类及其所有超类（Object除外）都遵循这个约定，那就是这种情况：

```java
x.clone().getClass() == x.getClass()
```

&emsp;&emsp;按照惯例，返回的对象应该独立于被克隆的对象。要实现这种独立性，可能需要在返回之前修改super.clone返回的对象的一个或多个字段。

&emsp;&emsp;这个机制与构造函数调用链类似（This mechanism is vaguely similar to constructor chaining）【就是子类的构造函数一定需要调用父类的构造函数】，它只是没有强制执行：如果一个类的clone方法返回一个不是通过super.clone而是通过调用构造函数获得的实例，编译器不会报错（complain），但是如果一个该类的子类调用super.clone，生成的对象将具有错误的类，从而阻止子类clone方法的正常运行。如果重写clone的类是final，则可以安全地忽略此约定，因为没有子类，所以不需要担心。但是如果一个final类有clone方法，但是该方法没有调用super.clone方法，那么该类没有理由实现Cloneable，因为它不依赖与Object的clone实现的行为。

&emsp;&emsp;假设你希望在一个类中实现Cloneable，并且它的超类都提供良好的clone方法。先调用super.clone。您获得的对象将是原始的完整功能的副本。在您的类中声明的任何字段将具有与原始值相同的值。如果每个字段都包含原始值或对不可变对象的引用，则返回的对象可能正是您所需要的，在这种情况下，不需要进一步处理。 例如，对于第11项中的PhoneNumber类就是这种情况，但请注意，不可变类应该永远不会提供clone方法，因为它只会鼓励浪费的复制（because it would merely encourage wasteful copying）【就是说复制了也没啥用，复制的每一个结果都是一样的，因为类是不可变的】。有了这个预告，下面是PhoneNumber的clone方法的样子：

```java
// Clone method for class with no references to mutable state
@Override public PhoneNumber clone() {
    try {
        return (PhoneNumber) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new AssertionError(); // Can't happen
    }
}
```

&emsp;&emsp;为了使此方法起作用，必须修改PhoneNumber的类声明以表明它实现了Cloneable。 虽然Object的clone方法返回Object，但此clone方法返回PhoneNumber。这样做是合法的，也是可取的，因为Java支持协变返回类型（covariant return types）。换句话说，重写方法的返回类型可以是被重写的子类方法的返回类型。这样客户端就不需要进行转换了（This eliminates the need for casting in the client.）。我们必须在返回之前将super.clone的结果从Object转换为PhoneNumber，但是这个转换保证是可以成功的（but the cast is guaranteed to succeed）。

&emsp;&emsp;对super.clone的调用包含在try-catch块中。 这是因为Object声明其clone方法抛出CloneNotSupportedException，这是一个经过检查的异常。 因为PhoneNumber实现了Cloneable，所以我们知道对super.clone的调用会成功。对这个样板文件的需求表明，CloneNotSupportedException应该是不受约束的（第71项）（The need for this boilerplate indicates that  CloneNotSupportedException should have been unchecked (Item 71)）。

&emsp;&emsp;如果对象包含引用可变对象的字段，使用上述这种简单的clone实现可能会导致灾难性的后果。例如，考虑到第7项中的Stack类：

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    public Stack() {
        this.elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }
    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // Eliminate obsolete reference
        return result;
    }
    // Ensure space for at least one more element.
    private void ensureCapacity() {
    if (elements.length == size)
        elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

&emsp;&emsp;假设你希望把这个类做成可克隆的（cloneable）。如果它的clone方法仅仅返回super.clone()，这样得到的Stack实例，在其size具有正确的值，但是它的elements字段将引用与原始Stack实例相同的数组。修改原始的实例会破坏被克隆对象中的约束条件【也就是说修改原始对象中的数组，克隆出来的对象中的数组也会跟着变】，反之亦然。 您将很快发现您的程序产生无意义的结果或抛出NullPointerException。

&emsp;&emsp;如果调用Stack类中唯一的构造器，这种情况就永远不会发生。实际上，clone方法就是另一个构造器；你必须确保它不会伤害到原始的对象，并确保正确地创建被克隆对象中的约束条件（invariants）。为了使Stack上的clone方法正常工作，它必须复制堆栈的内部信息。最简单的方法是在elements数组上递归调用clone：

```java
// Clone method for class with references to mutable state
@Override public Stack clone() {
    try {
        Stack result = (Stack) super.clone();
        result.elements = elements.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```

&emsp;&emsp;请注意，我们不必将elements.clone的结果强制转换为Object []。在数组上调用clone会返回一个数组，其运行时和编译时类型与要克隆的数组的类型相同。 这是复制数组首选的习惯用法。 实际上，数组是克隆工具的唯一引人注目的用途（In fact, arrays are the sole compelling use of the clone facility）。

&emsp;&emsp;还要注意，如果elements字段是final的，上述方法就不能正常工作，因为clone方法是被禁止给elements字段赋新值的。这是一个根本的问题：与序列化一样，Cloneable体系结构与引用可变对象的最终字段的正常使用不兼容，除非可以在对象与其克隆之间安全地共享可变对象。 为了使类可克隆，可能需要从某些字段中删除final修饰符。

&emsp;&emsp;仅递归调用克隆有时是不够的。 例如，假设您正在为哈希表编写clone方法，该哈希表的内部由桶数组组成，每个桶都引用键值对链表中的第一个条目。为了提高性能，该类实现了自己的轻量级单链表，而不是在内部使用java.util.LinkedList：

```java
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;
    private static class Entry {
        final Object key;
        Object value;
        Entry next;
        Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
            ... // Remainder omitted
        }
    }
}
```

&emsp;&emsp;假设您只是递归地克隆bucket数组，就像我们对堆栈所做的那样：

```java
// Broken clone method - results in shared mutable state!
@Override public HashTable clone() {
    try {
        HashTable result = (HashTable) super.clone();
        result.buckets = buckets.clone();
        return result;
    } catch (CloneNotSupportedException e) {
        throw new AssertionError();
    }
}
```


&emsp;&emsp;虽然克隆出来的对象拥有自己的桶数组，但此数组与原始对象引用的是相同的链接列表，这很容易在克隆和原始对象之间出现不确定的行为。 要解决此问题，您必须复制包含每个存储桶的链接列表。 这是一种常见的方法：

```java
// Recursive clone method for class with complex mutable state
public class HashTable implements Cloneable {
    private Entry[] buckets = ...;
    private static class Entry {
        final Object key;
        Object value;
        Entry next;
        Entry(Object key, Object value, Entry next) {
            this.key = key;
            this.value = value;
            this.next = next;
        }
        // Recursively copy the linked list headed by this Entry
        Entry deepCopy() {
            return new Entry(key, value, next == null ? null : next.deepCopy());
        }
    }
    @Override public HashTable clone() {
        try {
            HashTable result = (HashTable) super.clone();
            result.buckets = new Entry[buckets.length];
            for (int i = 0; i < buckets.length; i++)
                if (buckets[i] != null)
                    result.buckets[i] = buckets[i].deepCopy();
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
    ... // Remainder omitted
}
```

&emsp;&emsp;私有类HashTable.Entry已经被加强了，它支持一个“深度拷贝（deep copy）”方法。HashTable上的clone方法分配一个适当大小的新桶数组，并遍历原始桶数组，深度拷贝每个非空桶。Entry上的deepCopy方法以递归方式调用自身，以复制该Entry为首的整个链表。虽然说这种方法很灵活，并且如果散列桶不是很长的话，也可以正常工作，但它不是克隆链表的好方法，因为它会为链表列表中的每个元素消耗一个堆栈帧。如果列表比较长，这很容易导致堆栈溢出。为了防止这种情况，你可以在deepCopy中用遍历代替递归：

```java
// Iteratively copy the linked list headed by this Entry
Entry deepCopy() {
    Entry result = new Entry(key, value, next);
    for (Entry p = result; p.next != null; p = p.next)
        p.next = new Entry(p.next.key, p.next.value, p.next.next);
    return result;
}
```

&emsp;&emsp;克隆复杂可变对象的最后一种方法是调用super.clone，将结果对象中的所有字段设置为其初始状态，然后调用更高级别的方法来重新生成原始对象的状态。 在我们的HashTable示例的情况下，buckets字段将初始化为新的bucket数组，并且将为要克隆的哈希表中的每个键值映射调用put（key，value）方法（未显示）。这种方法通常会产生一种简单，相当优雅的克隆方法，其运行速度不如直接操作克隆【对象】内部的克隆方法。 虽然这种方法很简洁，但它与整个Cloneable体系结构是对立的，因为它盲目地覆盖了一个又一个构成基础体系结构的字段对象的副本。

&emsp;&emsp;与构造函数一样，克隆方法中一定不能在构造的过程中调用可以被重写的方法【只能调用final方法、private方法等】（第19项）。如果clone方法调用了一个在子类中被重写的方法，那么在该方法所在的子类就有机会在修正克隆对象的状态之前执行，这样很有可能会导致克隆对象和原始对象之间状态的不一致。因此，前一段中讨论的`put(key, value)`方法应该是final或者private。（如果它是私有的，它可能是非final公共方法的“辅助方法（helper method）”）

&emsp;&emsp;Object的clone方法被声明为可抛出CloneNotSupportedException异常，但是，重写的【clone】方法就不需要【这么做】，公有的clone方法应该省略这个声明，因为不会抛出受检异常（checked exceptions）的方法【与会爬出异常的方法】更容易使用（第71项）。

&emsp;&emsp;在设计继承类时，您有两种选择（第19项），但无论您选择哪一种，该类都不应该实现Cloneable【接口】。您可以选择通过实现一个适当的受保护的克隆方法来模拟Object的行为，该方法被声明为抛出CloneNotSupportedException异常。这样子类就可以自由地实现Cloneable【接口】，就像它们直接扩展Object一样。或者，您可以选择不去实现一个加工过的克隆方法，并通过提供以下简化了的克隆实现来防止子类去实现它：

```java
// clone method for extendable class not supporting Cloneable
@Override
protected final Object clone() throws CloneNotSupportedException {
    throw new CloneNotSupportedException();
}
```

&emsp;&emsp;这还有一个细节值得注意，如果编写一个实现Cloneable接口的线程安全类，请记住它的clone方法必须得到很好的同步，就像任何其他【同步的】方法一样（第78条）。Object的clone方法没有同步，因此即使其实现方式令人满意，您可能也必须编写返回super.clone()的同步clone方法。

&emsp;&emsp;简而言之，所有实现Cloneable的类都应该使用返回类型为类本身的公共方法来覆盖clone。此方法首先调用super.clone()，然后修复需要修复的任何字段【将某些字段修改为克隆对象的值】。通常，这意味着复制包含对象的内部“深层结构”的任何可变对象，并用指向新对象的引用代替原来指向这些对象的引用。虽然这些内部副本通常可以通过递归调用clone来完成，但这通常不是最佳方法。如果类值包含基本类型的字段或是对不可变对象的引用，则很可能不需要修复任何字段，这条规则也有例外。例如，代表序列号或其他唯一ID的字段需要修复，即使它是基本类型或者不可变的也要被修正。

&emsp;&emsp;真的有必要这么复杂吗？很少有这种必要。如果你扩展一个已经实现Cloneable的类，那么除了实现一个良好行为的clone方法之外别无选择。否则，通常最好提供另一种对象拷贝的方法。更好的对象复制方法是提供拷贝构造器（copy constructor）或拷贝工厂（copy factory）。拷贝构造器知识一个构造器，他接收一个参数，其类型是包含该构造器的类，例如：

```java
// Copy constructor
public Yum(Yum yum) { ... };
```

&emsp;&emsp;一个拷贝工厂是一个类似于拷贝构造器的静态工厂（第1项）：

```java
// Copy factory
public static Yum newInstance(Yum yum) { ... };
```

&emsp;&emsp;拷贝构造器的做法，及其静态工厂方法的变形，都比Cloneable/clone方法具有更多的优点：它们不依赖于某一种很有风险的、语言之外的对象创建机制；他们不需要遵守尚未定制好的文档的规范；它们不会与final字段的正常使用发生冲突；它们不会抛出不必要的受检异常（checked exception）；而且它们不需要进行类型转换。

&emsp;&emsp;此外，拷贝构造函数或工厂可以采用其类型是类实现的接口的参数。例如，按照惯例，所有通用集合实现都提供一个构造函数，其参数的类型为Collection或Map。 基于接口的复制构造函数和工厂（更恰当地称为转换构造函数和转换工厂），允许客户端选择拷贝的实现类型，而不是强迫客户端接受原始的实现类型。例如，假设您有一个HashSet，并且您希望将其拷贝成一个TreeSet。clone方法无法提供此功能，但是用转换构造函数很容易实现：`new TreeSet<>(s)`。

&emsp;&emsp;既然Cloneable具有上述那么多问题，新的接口就不应该对它进行扩展，而且新的可扩展的类也不应该实现它。虽然实现Cloneable接口对final类的危害较小，但应将其视为性能优化，保留用于极少数情况下的合理性（第67项）。通常，复制功能最好由构造函数或工厂提供。 此规则的一个值得注意的例外是数组，最好使用clone方法进行复制。