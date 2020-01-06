## 如果需要精确的答案，请避免使用 float 和 double

&emsp;&emsp;float 和 double 类型主要是为了科学计算和工程计算而设计的。它们执行*二进制浮点运算（binary floating-point arithmetic）*，这是为了在广泛的数值范围上提供较为精确为快速的近似计算而精心设计的。然而，它们并没有提供完全精确的结果，所以不应该被用于需要精确结果的场合。**float 和 double 类型尤其不适合用于货币计算** ，因为要让一个 float 或者 double 精确地表示 0.1（或者 10 的任何其他负数次方值）是不可能的。

&emsp;&emsp;例如，假设你口袋中有$1.03，花掉了 42¢ 之后还剩下多少钱呢？下面是一个简单的程序片段，来回答这个问题：

```java
System.out.println(1.03 - 0.42);
```

&emsp;&emsp;遗憾的是，它打印出来的是 0.6100000000000001。这并不是个例。假设你的口袋里面有$1，你买了 9 个垫圈，每个为 10¢。那么你应该找回多少零钱呢？

```java
System.out.println(1.00 - 9 * 0.10);
```

&emsp;&emsp;根据这个程序片段，你得到的是\$0.09999999999999998。

&emsp;&emsp;你可能会认为，只要在打印之前将结果做一下舍入就可以解决这个问题，但遗憾的是，这种做法并不总是可行的。例如，假设你的口袋有$1，你看到货架上有一排美味的糖果，标价分别为10¢, 20¢, 30¢, 等等，一直到$1。你打算从标价为 10¢ 的糖果开始，每种买一颗，一直到不能支付货架上下一种价格的糖果位置，那么你可以买多少颗糖果？还会找回多少零钱？下面是一个简单的程序，用来解决这个问题：

```java
// Broken - uses floating point for monetary calculation!
public static void main(String[] args) {
    double funds = 1.00;
    int itemsBought = 0;
    for (double price = 0.10; funds >= price; price += 0.10) {
        funds -= price;
        itemsBought++;
    }
    System.out.println(itemsBought + " items bought.");
    System.out.println("Change: $" + funds);
}
```

&emsp;&emsp;如果你运行这个程序，你会发现你可以支付 3 颗糖果，而且还剩下$0.3999999999999999。这个答案是不正确的！解决这个问题的正确办法是**是使用 BigDecimal、int 或者 long 进行货币计算** 。

&emsp;&emsp;下面的程序是上一个程序的简单翻版，它使用 BigDecimal 类型代替 double。请注意，使用的是 BigDecimal 的 String 构造函数而不是它的 double 构造函数。这是必须的，从而避免在计算中引入不准确的值\[Bloch05, Puzzle 2\]：

```java
public static void main(String[] args) {
    final BigDecimal TEN_CENTS = new BigDecimal(".10");
    int itemsBought = 0;
    BigDecimal funds = new BigDecimal("1.00");
    for (BigDecimal price = TEN_CENTS;
    funds.compareTo(price) >= 0;
    price = price.add(TEN_CENTS)) {
        funds = funds.subtract(price);
        itemsBought++;
    }
    System.out.println(itemsBought + " items bought.");
    System.out.println("Money left over: $" + funds);
}
```

&emsp;&emsp;如果运行这个修改过的程序，就会发现你可以支付 4 颗糖果，还剩下$0.00。这才是正确的答案。

&emsp;&emsp;然而，使用 BigDecimal 有两个缺点：与使用基本运算类型相比，这样做很不方便，而且很慢。如果你解决的是一个简单的问题，后一种缺点并不要紧，但是前一种缺点可能会让你很不舒服。

&emsp;&emsp;除了使用 BigDecimal 之外，还有一种办法是使用 int 或者 long，到底选用 int 或者 long 要取决于涉及数值的大小，同时要自己处理十进制小数点。在这个示例中，最明显的做法是以美分为单位进行计算，而不是以美元为单位。下面是这个例子的简单翻版，展示了这种做法：

```java
public static void main(String[] args) {
    int itemsBought = 0;
    int funds = 100;
    for (int price = 10; funds >= price; price += 10) {
        funds -= price;
        itemsBought++;
    }
    System.out.println(itemsBought + " items bought.");
    System.out.println("Cash left over: " + funds + " cents");
}
```

&emsp;&emsp;总而言之，对于任何需要精确答案的计算任务，请不要使用 float 或者 double。如果你想让系统来记录十进制小数点，并且不介意因为不使用基本类型而带来的不便，就请使用 BigDecimal。使用 BigDecimal 还有一些额外的好处，它允许你完全控制舍入，只要执行需要舍入的操作，就可以从八种舍入模式中进行选择。如果你使用合法的舍入行为执行业务计算，这就能派上用场。如果性能非常关键，并且你又不介意自己记录十进制小数点，而且涉及的数值又不会太大，使用 int 或者 long。如果数值范围没有超过 9 为十进制数字，就可以使用 int；如果不超过 18 位数字，就可以使用 long。如果数值可能超过 18 位数字，就必须使用 BigDecimal。

> - [第 59 项：了解和使用类库](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第09章：通用编程/第59项：了解和使用类库.md)
> - [第 61 项：基本类型优先于装箱基本类型](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第09章：通用编程/第61项：基本类型优先于装箱基本类型.md)
