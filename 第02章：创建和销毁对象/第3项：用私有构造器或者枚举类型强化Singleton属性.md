## 用私有构造器或者枚举类型强化Singleton属性

&emsp;&emsp;Singleton指仅仅被实例化一次的类 \[Gamma95\]。Singleton通常代表无状态的对象，例如函数(第24项)或者本质上唯一的系统组件。使类称为Singleton会使它的客户端测试变得十分困难，因为除非它实现了作为其类型的接口，否则不可能将模拟实现替换为单例。

&emsp;&emsp;实现单例的方法有两种。 两者都基于保持构造函数私有并导出公共静态成员以提供对唯一实例的访问。 在一种方法中，该成员是final字段：

```java
// Singleton with public final field
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public void leaveTheBuilding() { ... }
}
```

&emsp;&emsp;私有构造器只调用一次，用来初始化静态变量`Elvis.INSTANCE`。由于缺少`public`或者`protect`属性的构造器，这就保证了`Elvis`的全局一致性：一旦`Evlis`类被实例化，只会存在一个`Elvis`实例，不多也不少。客户端所做的任何事情都无法改变这一点，但有一点需要注意：享有特权的客户端可以借助`AccessibleObject.setAccessible`方法反射性地调用私有构造函数(第65项)。 如果你需要防御此攻击，请修改构造函数以使其在要求创建第二个实例时抛出异常。

&emsp;&emsp;在实现Singleton的第二种方法中，公有的成员是个静态工厂方法：
```java
// Singleton with static factory
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public static Elvis getInstance() { return INSTANCE; }
    public void leaveTheBuilding() { ... }
}
```

&emsp;&emsp;对于静态方法`Elvis.getInstance`的所有调用，都会返回同一个对象引用，所以，永远不会创建其他的`Elvis`实例(上述提醒依然适用)。

&emsp;&emsp;公有域方法的主要好处在于，组成类的成员的声明很清楚地声明了这个类是一个Singleton：公有的静态域是final的，所以该域总是包含同一个对象的引用。第二个好处就是它更加简单。

&emsp;&emsp;工厂方法的优势之一在于，它提供了灵活性：在不改变其API的前提下，我们可以改变类是否应该为Singleton的想法。工厂方法返回唯一实例，但是，它可以很容易被修改，比如改成每个调用该方法的线程返回一个唯一的实例。第二个优点是，如果你的应用需要，你可以编写泛型单例工厂(第30项)。使用静态工厂的最后一个优点是方法参考可以用作供应商，例如`Elvis::instance`是供应商&lt;Elvis&gt;。除非这些优势中的一个是相关的，否则公共领域方法更可取。(A final advantage of using a static factory is that a method reference can be used as a supplier, for example Elvis::instance is a Supplier&lt;Elvis&gt; .Unless one of these advantages is relevant, the public field approach is preferable.)

&emsp;&emsp;为了使利用这其中一种方法实现的Singleton类编程可序列化的(第12章)，仅仅在声明中加上“implements Serializable”是不够的。为了维护并保证Singleton，必须声明所有实例域都是瞬时(transient)的，并提供一个`readResolve`方法(第89项)。否则，每次反序列化时，都会创建一个新的实例，在我们的示例中，会导致“假冒的Elvis”。为了防止这种情况，要在Elvis类中加入下面这个readResolve方法：

```java
// readResolve method to preserve singleton property
private Object readResolve() {
    // Return the one true Elvis and let the garbage collector
    // take care of the Elvis impersonator.
    return INSTANCE;
}
```

&emsp;&emsp;实现Singleton还有第三种方法。只需要编写一个包含单个元素的枚举类型：

```java
// Enum singleton - the preferred approach
public enum Elvis {
    INSTANCE;
    public void leaveTheBuilding() { ... }
}
```

&emsp;&emsp;这种方法类似于公共领域方法，但它更简洁，免费提供序列化机制，并提供了对多个实例化的铁定保证，即使面对复杂的序列化或反射攻击。这种方法可能会有点不自然，但单元素枚举类型通常是实现单例的最佳方法。请注意，如果你的单例必须扩展`Enum`以外的超类，则不能使用此方法(尽管你可以声明枚举来实现接口)。