## 返回长度为零的数组或者集合，而不是 null

&emsp;&emsp;像下面这样的方法并不少见：

```java
// Returns null to indicate an empty collection. Don’t do this!
private final List<Cheese> cheesesInStock = ...;
/**
 * @return a list containing all of the cheeses in the shop,
 * or null if no cheeses are available for purchase.
 */
public List<Cheese> getCheeses() {
    return cheesesInStock.isEmpty() ? null : new ArrayList<>(cheesesInStock);
}
```

&emsp;&emsp;把没有奶酪（cheese）可买的情况当作是一种特例，这是不合常理的。这样做会要求客户端中必须有额外的代码来处理 null 返回值，例如：

```java
List<Cheese> cheeses = shop.getCheeses();
if (cheeses != null && cheeses.contains(Cheese.STILTON))
    System.out.println("Jolly good, just the thing.");
```

&emsp;&emsp;对于一个返回 null 而不是零长度数组或者集合的方法，几乎每次用到该方法时都需要这种曲折的方式来处理。这样做很容易出错，因为编写客户端程序的程序猿可能会忘记写这种专门的代码来处理 null 返回值。这样的错误也许好几年都不会注意到，因为这样的方法通常返回一个或者多个对象。返回 null 而不是空的容器（empty container）也会使返回容器的方法的实现变得复杂。

&emsp;&emsp;有时候会有人认为：null 返回值比空的容器或者数组更好，因为它避免了分配空容器所需要的开销。这种观点是站不住脚的，原因有亮点。第一，在这个级别上担心性能问题是不明智的，除非分析表明这个方法正是造成性能问题的真正源头（第 67 项）。第二，可以返回空集合和数组而不分配【空间给】它们。以下是返回可能为空的集合的典型代码。通常，这就是你所需要的：

```java
//The right way to return a possibly empty collection
public List<Cheese> getCheeses() {
    return new ArrayList<>(cheesesInStock);
}
```

&emsp;&emsp;万一你有证据表明分配空的集合会对性能有所损耗，你可以通过重复返回相同的不可变空集合来避免分配【空间】，因为不可变对象可以自由共享（第 17 项）。下面是使用 Collections.emptyList 方法执行此操作的代码。如果你要返回一个集合，你将使用 Collections.emptySet; 如果你要返回 map，则使用 Collections.emptyMap。但请记住，这是一种优化，而且很少需要它。如果你认为需要它，请测试【使用这种优化】之前和之后的性能，以确保它实际上是有所帮助的：

```java
// Optimization - avoids allocating empty collections
public List<Cheese> getCheeses() {
    return cheesesInStock.isEmpty() ? Collections.emptyList() : new ArrayList<>(cheesesInStock);
}
```

&emsp;&emsp;数组的情况与集合的情况相同。永远不要返回 null 来代替零长度的数组。通常，你应该只返回一个正确长度的数组，该数组【长度】可能为零。请注意，我们将一个零长度的数组传递给 toArray 方法来表示【我们】所需的返回的类型，即 Cheese[]：

```java
//The right way to return a possibly empty array
public Cheese[] getCheeses() {
    return cheesesInStock.toArray(new Cheese[0]);
}
```

&emsp;&emsp;如果你认为分配零长度的数组会损耗性能，则可以重复返回相同的零长度的数组，因为所有零长度数组都是不可变的：

```java
// Optimization - avoids allocating empty arrays
private static final Cheese[] EMPTY_CHEESE_ARRAY = new Cheese[0];
public Cheese[] getCheeses() {
    return cheesesInStock.toArray(EMPTY_CHEESE_ARRAY);
}
```

&emsp;&emsp;在优化的版本中，我们将相同的空数组传递到每个 toArray 调用中，并且每当 cheesesInStock 为空时，此数组将从 getCheeses 返回。不要预先分配传递给 toArray 的数组，以期望提高性能。 研究表明它适得其反\[Shipilëv16\]：

```java
// Don’t do this - preallocating the array harms performance!
return cheesesInStock.toArray(new Cheese[cheesesInStock.size()]);
```

&emsp;&emsp;总之，**永远不要返回 null 来代替【返回】空数组或集合**。它会让你的 API 更难以使用并且更容易出错，并且它没有性能优势。

> - [第 53 项：慎用可变参数](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第08章：方法/第53项：慎用可变参数.md)
> - [第 55 项：谨慎返回 optinal](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第08章：方法/第55项：谨慎返回optional.md)

