### 坚持使用Overide注解

&emsp;&emsp;Java库包含多种注释类型。对于传统的程序猿，其中最重要的就是@Override。该注解只能用于方法声明，它表示被注解的方法声明会覆盖超类型中的声明。如果坚持使用这个注解，可以防止一大类的非法错误。考虑下面的程序，这里的类Bigram表示一个*双子母组*或者有序的字母对：

```java
// Can you spot the bug?
public class Bigram {
    private final char first;
    private final char second;
    public Bigram(char first, char second) {
        this.first = first;
        this.second = second;
    }
    public boolean equals(Bigram b) {
        return b.first == first && b.second == second;
    }
    public int hashCode() {
        return 31 * first + second;
    }
    public static void main(String[] args) {
        Set<Bigram> s = new HashSet<>();
        for (int i = 0; i < 10; i++)
            for (char ch = 'a'; ch <= 'z'; ch++)
                s.add(new Bigram(ch, ch));
        System.out.println(s.size());
    }
}
```

&emsp;&emsp;main程序反复地将26个双字母组添加到集合中，每个双字母组都由两个相同的小写字母组成。然后它打印出集合的大小。你可能以为程序打印出的大小为26，因为集合不能包含重复【元素】。如果你试着运行程序，会发现它打印的不是26而是260.哪里出错了呢？

&emsp;&emsp;很显然，Bigram类的创建者原本想要覆盖equals方法（第10项），同时还记得覆盖了hashCode。遗憾的是，不幸的程序猿没能覆盖equals，而是将它重载了（第52项）。为了覆盖Object.equals，必须定义一个参数为Object类型的equals方法，但是Bigram的equals方法的参数并不是Object类型，因此Bigram从Object继承了equals方法。这个equals方法测试对象的同一性，就像==操作符一样。每个bigram的10个备份中，每一个都与其余的9个不同，因此Object.equals认为它们不相等，这正解释了程序为什么会打印出260的原因。

&emsp;&emsp;幸运的是，编译器可以帮助你发现这个错误，但是只有当你告知编译器你想要覆盖Object.equals时才行。为了做到这一点，要用@Override标注Bigram.equals，如下所示：

```java
@Override
public boolean equals(Bigram b) {
    return b.first == first && b.second == second;
}
```

&emsp;&emsp;如果插入这个注解，并试着重新编译程序，编译器就会产生一条像这样的错误信息：

```java
Bigram.java:10: method does not override or implement a method
from a supertype
    @Override public boolean equals(Bigram b) {
    ^
```

&emsp;&emsp;你就会立即意识到哪里错了，拍拍自己的额头，马上用正确的来取代出错的equals实现（第10项）：

```java
@Override public boolean equals(Object o) {
    if (!(o instanceof Bigram))
        return false;
    Bigram b = (Bigram) o;
    return b.first == first && b.second == second;
}
```

&emsp;&emsp;因此，**你应该在你想要覆盖超类声明的每个方法声明中使用Override注解。** 这一规则有个小小的例外。如果你在编写一个没有标注为抽象的类，并且确信它覆盖了抽象的方法，在这种情况下，就不必将Override注解放在该方法上了。在没有声明为抽象的类中，如果没有覆盖抽象的超类方法，编译器就会发出一条错误消息。但是，你可能希望关注类中所有覆盖超类方法的方法，在这种情况下，也可以放心地标注这些方法。当你选择覆盖方法时，大多数IDE可以设置为自动插入覆盖注释。

&emsp;&emsp;大多数IDE提供了坚持使用Override注解的另一种理由。如果启用相应的代码检验功能，当你的方法没有Override注解，却覆盖了超类方法时，IDE就会产生一条警告。如果坚持使用Override注解，这些警告就会提醒你警惕无意识的覆盖。这些警告补充了编译器的错误信息，提醒你警惕无意识的覆盖失败。在IDE和编译器之间，可以确保你覆盖任何你想要覆盖的方法，无一遗漏。

&emsp;&emsp;Override注解可以用在覆盖接口和类的声明的方法的声明上【Override注解可以用在某些方法的声明上，这些方法是覆盖接口中声明的方法或者类中声明的方法】。随着默认方法的出现【接口中可以使用default修饰方法，提供接口中方法的默认实现】，更好的做法是在接口方法的具体实现上使用Override来确保签名是正确的。如果你知道接口没有默认方法，则可以选择在接口方法的具体声明上省略Override注解来减少混乱【保持代码的整洁性】。

&emsp;&emsp;但是在抽象类或者接口中，还是值得标注所有你想要的方法，来覆盖超类或者超接口方法，无论是具体的还是抽象的。例如，Set接口没有给Collection添加新方法，因此它应该在它的所有方法声明中包括Override注解，以确保它不会意外地给Colection接口添加任何新方法。

&emsp;&emsp;总而言之，如果在你想要的没个方法声明中使用Override注解来覆盖超类声明，编译器就可以替你防止大量的错误，但有一个例外。在具体的类中，不必标注你确信覆盖了抽象方法声明的方法（虽然这么做也没什么坏处）。