## for-each循环优先于传统的for循环

&emsp;&emsp;如第45项所述，某些任务最好用流（stream）完成，其他任务最好用迭代完成。这是一个传统的for循环迭代集合：

```java
// Not the best way to iterate over a collection!
for (Iterator<Element> i = c.iterator(); i.hasNext(); ) {
    Element e = i.next();
    ... // Do something with e
}
```

&emsp;&emsp;这是使用传统的for循环迭代数组【的做法】：

```java
// Not the best way to iterate over an array!
for (int i = 0; i < a.length; i++) {
    ... // Do something with a[i]
}
```

&emsp;&emsp;这些做法都比while循环（第57项）更好，但是它们也并不完美。迭代器和索引变量都会造成一些混乱————你所需要的都是【其中的】元素。而且，它们也代表着出错的可能。迭代器在每个循环中出现三次，索引变量出现四次，这使得你有很多机会使用错误的变量。一旦出错，就无法保证编译器能够发现错误。最后，这两种循环是完全不一样的，不必去注意容器的类型，也不必添加（轻微（minor））麻烦来改变这种类型（drawing unnecessary attention to the type of the container and adding a (minor) hassle to changing that type）。

&emsp;&emsp;for-each循环（官方称为“增强语句”）解决了所有这些问题。 它通过隐藏迭代器或索引变量来避免混乱和出错的机会。由此产生的习惯【用法】同样适用于集合和数组，简化了将容器的实现类型从一个切换到另一个的过程：

```java
// The preferred idiom for iterating over collections and arrays
for (Element e : elements) {
    ... // Do something with e
}
```

&emsp;&emsp;当你看到冒号(:)时，可以把它读作“在……里面”。因此上面的循环可以读作：“对于元素中的每个元素e”。利用for-each循环不会有性能损失，甚至用于数组也一样：他们生成的代码与你手工编写的代码基本相同。

&emsp;&emsp;在对多个集合进行嵌套式迭代时，for-each循环对于传统的for循环的这种优势还会更加明显。下面就是人们在试图做嵌套迭代时会犯的错误：

```java
// Can you spot the bug?
enum Suit { CLUB, DIAMOND, HEART, SPADE }
enum Rank { ACE, DEUCE, THREE, FOUR, FIVE, SIX, SEVEN, EIGHT, NINE, TEN, JACK, QUEEN, KING }
...
static Collection<Suit> suits = Arrays.asList(Suit.values());
static Collection<Rank> ranks = Arrays.asList(Rank.values());
List<Card> deck = new ArrayList<>();
    for (Iterator<Suit> i = suits.iterator(); i.hasNext(); )
        for (Iterator<Rank> j = ranks.iterator(); j.hasNext(); )
            deck.add(new Card(i.next(), j.next()));
```

&emsp;&emsp;如果之前没有发现这个BUG也不必难过。许多专家级的程序猿偶尔也会犯这样的错误。问题在于，在迭代器对外部的集合（suits）调用了太多次的next方法了。它应该从外部的循环进行调用，以便每种花色调用一次，但它却是从内部循环调用，因此它是每张牌调用一次。在用完所有花色之后，循环就会抛出NoSuchElementException异常。

&emsp;&emsp;如果真的那么不幸，并且外部集合的大小是内部集合大小的几倍————可能因为它们是相同的集合————循环就会正常终止，但是不会完成你想要的工作。例如，下面是个考虑不周的尝试，要打印一对骰子的所有可能地滚法：

```java
// Same bug, different symptom!
enum Face { ONE, TWO, THREE, FOUR, FIVE, SIX }
...
Collection<Face> faces = EnumSet.allOf(Face.class);
for (Iterator<Face> i = faces.iterator(); i.hasNext(); )
    for (Iterator<Face> j = faces.iterator(); j.hasNext(); )
        System.out.println(i.next() + " " + j.next());
```

&emsp;&emsp;这个程序不会抛出异常，而是只打印6个重复的词（从“ONE ONE”到“SIX SIX”），而不是预计的36种组合。

&emsp;&emsp;为了修正这些示例中的BUG，必须在外部循环的作用域中添加一个变量来保存外部元素：

```java
// Fixed, but ugly - you can do better!
for (Iterator<Suit> i = suits.iterator(); i.hasNext(); ) {
    Suit suit = i.next();
    for (Iterator<Rank> j = ranks.iterator(); j.hasNext(); )
        deck.add(new Card(suit, j.next()));
}
```

&emsp;&emsp;如果使用的是嵌套的for-each循环，这个问题就会完全小时。产生的代码就如你所希望的那样简洁：

```java
// Preferred idiom for nested iteration on collections and arrays
for (Suit suit : suits)
    for (Rank rank : ranks)
        deck.add(new Card(suit, rank));
```

&emsp;&emsp;遗憾的是，有三种常见的情况无法使用for-each循环：

- **破坏性过滤（Destructive filtering）** ————如果需要遍历集合，并删除选定的元素，就需要使用显式的迭代器，以便一颗调用它的remove方法。你通常可以使用在Java 8中添加的Collection的removeIf方法来避免显式遍历。

- **转换** ————如果需要遍历列表或数组并替换其元素的部分或全部值，则需要列表迭代器或数组索引才能替换元素的值。

- **并行迭代** ————如果需要并行地遍历多个集合，就需要显式地控制迭代器或者索引变量，以便所有迭代器或者索引变量都可以得到同步前移（就如上述关于有问题的牌和骰子的示例中无意中所示范的那样）。

&emsp;&emsp;如果你发现自己处于上述任何一种情况，请使用普通的for循环并警惕此项中提到的陷阱。

&emsp;&emsp;for-each循环不仅可以迭代集合和数组，还可以迭代实现Iterable接口的任何对象，该接口由单个方法组成。以下是接口的代码：

```java
public interface Iterable<E> {
    // Returns an iterator over the elements in this iterable
    Iterator<E> iterator();
}
```

&emsp;&emsp;如果你必须从头开始编写自己的Iterator实现，那么实现Iterable有点棘手，但是如果你正在编写一个代表一组元素的类型，你应该强烈考虑让它实现Iterable，即使你选择不让它实现Collection也是如此。这将允许你的用户使用for-each循环迭代你的类型，他们将永远感激你。

&emsp;&emsp;总之，for-each循环在清晰度，灵活性和预防出错方面提供了超越传统for循环的引人注目的优势，而且不会有性能损失。在使用中尽可能让for-each循环优先于for循环。