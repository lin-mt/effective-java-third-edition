# effective-java-third-edition

## 介绍
Effective Java 第三版全文翻译，纯属个人业余翻译，不合理的地方，望指正，感激不尽！

### 目录
### 推荐序
### 前言
### 致谢

---
### 第一章 引言

---
### [第二章 创建和销毁对象]()

&emsp;&emsp;本章涉及创建和销毁对象，包括何时以及如何创建它们，何时以及如何避免创建它们，如何确保它们被及时销毁，以及如何管理在销毁之前必须进行的清理操作。

>- [第1项：用静态工厂方法代替构造器]()
>- [第2项：遇到多个构造器参数时要考虑使用构建器]()
>- [第3项：用私有构造器或者枚举类型强化Singleton属性]()
>- [第4项：通过私有构造器强化不可实例化的能力]()
>- [第5项：优先考虑依赖注入来引用资源]()
>- [第6项：避免创建不必要的对象]()
>- [第7项：消除过期的对象引用]()
>- [第8项：避免使用终结方法和清除方法]()
>- [第9项：try-with-resources优先于try-finally]()

---
### [第三章 对于所有对象都通用的方法

&emsp;&emsp;尽管Object是一个具体类，但是设计它主要是为了扩展。它所有的非final方法（equals、hashCode、toString、clone和finalize）都有明确的通用约定（general contracts），因为它们被设计成是要被重写（override）的。任何一个类，它在覆盖这些方法的时候，都有责任遵守这些通用约定；如果做不到这一点，其他依赖于这些约定的类（例如HashMap和HashSet）就无法结合该类一起正常运作。

&emsp;&emsp;本章将讲述何时以及如何覆盖这些非final的Object方法。本章不再讨论finalize方法，因为第7项已经讨论过这个方法了。而Comparable.compareTo虽然不是Object方法，但是本章也对它进行讨论，因为它具有类似的特征。

>- [第10项：覆盖equals时请遵守通用约定]()
>- [第11项：覆盖equals时总要覆盖hashCode]()
>- [第12项：始终要覆盖toString]()
>- [第13项：谨慎地覆盖clone]()
>- [第14项：考虑实现Comparable接口]()

---
### [第四章 类和接口]()

&emsp;&emsp;类和接口是Java程序设计语言的何鑫，他们也是Java语言的基本抽象单元。Java语言提供了许多强大的基本元素，供程序猿来设计类和接口。本章包含的一些指南可以帮助你充分利用这些元素，以便让你编写的类和接口可用、健壮且灵活。

>- [第15项：使类和成员的可访问性最小化]()
>- [第16项：要在公有类而非公有域中使用访问方法]()
>- [第17项：使可变性最小化]()
>- [第18项：复合优先于继承]()
>- [第19项：要么设计继承并提供文档说明，要么禁止继承]()
>- [第20项：接口优于抽象类]()
>- [第21项：为后代设计接口]()
>- [第22项：接口只用于定义类型]()
>- [第23项：类层次优于标签类]()
>- [第24项：静态成员类优于非静态成员类]()
>- [第25项：限制源文件为单个顶级类]()

---
### [第五章 泛型]()

&emsp;&emsp;Java 1.5发行版本中增加了泛型（Generic）。在没有泛型之前，从集合中读取到的每一个对象都必须进行转换。如果有人不小心插入了类型错误的对象，在运行时的转换处理就会出错。有个泛型，可以告诉编译器每个集合中接收哪些对象类型。编译器自动地为你的插入进行转化，并在编译时告知是否插入了类型错误的对象。这样可以使程序既更加安全，也更加清楚，但是这些好处（不仅仅针对集合）是需要付出代价的。本章将告诉您如何最大化利益并最大限度地减少并发症【使用泛型带来的坏处】（complications）。

>- [第26项：请不要使用原生态类型]()
>- [第27项：消除非受检的警告]()
>- [第28项：列表优于数组]()
>- [第29项：优先考虑泛型]()
>- [第30项：优先考虑泛型方法]()
>- [第31项：利用有限制通配符来提升API的灵活性]()
>- [第32项：谨慎并用泛型和可变参数]()
>- [第33项：优先考虑类型安全的异构容器]()

---
### [第六章 枚举和注解]()

&emsp;&emsp;Java支持两种特殊用途的引用类型：一种成为枚举（enum type）类型，以及一种称为注解类型（annotation type）的接口。本章讨论使用这些类型系列的最佳实践。

>- [第34项：用enum代替int常量]()
>- [第35项：用实例域代替序数]()
>- [第36项：用EnumSet代替位域]()
>- [第37项：用EnumMap代替序数索引]()
>- [第38项：用接口模拟可扩展的枚举]()
>- [第39项：注解优先于命名模式]()
>- [第40项：坚持使用Override注解]()
>- [第41项：用标记接口定义类型]()

---
### [第七章 Lambda和Stream]()

&emsp;&emsp;在Java 8中，添加了函数接口，lambda和方法引用，以便更容易地创建函数对象。在这些语法（language）更改的同时添加进了流API，以便为处理数据元素序列提供库支持。在本章中，我们将讨论如何充分利用这些工具。

>- [第42项：Lambda优先于匿名类]()
>- [第43项：方法引用优先于Lambda]()
>- [第44项：坚持使用标准的函数接口]()
>- [第45项：谨慎使用Stream]()
>- [第46项：优先选择Stream中无副作用的函数]()
>- [第47项：Stream要优先用Collection作为返回类型]()
>- [第48项：谨慎使用Stream并行]()

---
### [第八章 方法]()

&emsp;&emsp;本章讨论了方法设计的几个方面：如何处理参数和返回值，如何设计方法签名以及如何为方法编写文档。本章中的大部分内容适用于构造函数和方法。 与第4章一样，本章重点介绍可用性，健壮性和灵活性。

>- [第49项：检查参数的有效性]()
>- [第50项：必要时进行保护性拷贝]()
>- [第51项：谨慎设计方法签名]()
>- [第52项：慎用重载]()
>- [第53项：慎用可变参数]()
>- [第54项：返回零长度的数组或者集合，而不是null]()
>- [第55项：谨慎返回optinal]()
>- [第56项：为所有导出的API元素编写文档注释]()

---
### [第九章 通用编程]()

&emsp;&emsp;本章主要讨论Java语言的具体细节，讨论了局部变量、控制结构、类库、数据类型，以及两种不是由语言本身提供的机制（reflection和native method，反射机制和本地方法）。最后讨论了优化和命名惯例。

>- [第57项：将局部变量的作用域最小化]()
>- [第58项：for-each循环优先于传统的for循环]()
>- [第59项：了解和使用类库]()
>- [第60项：如果需要精确的答案，请避免使用float和double]()
>- [第61项：基本类型优先于装箱基本类型]()
>- [第62项：如果其他类型更适合，则尽量避免使用字符串]()
>- [第63项：了解字符串连接的性能]()
>- [第64项：通过接口引用对象]()
>- [第65项：接口优先于反射机制]()
>- [第66项：谨慎地使用本地方法]()
>- [第67项：谨慎地进行优化]()
>- [第68项：遵守普遍接受的命名惯例]()

---
### [第十章 异常]()

&emsp;&emsp;在最通常情况下，异常可以提高程序的可读性，可靠性和可维护性。如果使用不当，可能会产生相反的效果。本章提供有效使用异常的指南。

>- [第69项：只针对异常的情况才使用异常]()
>- [第70项：对可恢复的情况使用受检异常，对编程错误使用运行时异常]()
>- [第71项：避免不必要地使用受检异常]()
>- [第72项：优先使用标准的异常]()
>- [第73项：抛出与抽象对应的异常]()
>- [第74项：每个方法抛出的所有异常都要建立文档]()
>- [第75项：在细节消息中包含失败-捕获信息]()
>- [第76项：努力使失败保持原子性]()
>- [第77项：不要忽略异常]()

---
### 第十一章 并发

&emsp;&emsp;线程（RHREADS）机制允许同时进行多个活动。并发编程要比单线程编程要困难得多，因为有很多东西可能出错，也很难重现失败。你无法避免并发。它本来就存在Java平台中了，如果你要从多核处理器中获得更好的性能，并发也是一个必要条件，这些现在都是十分普遍的了。本章阐述的建议可以帮助你编写出清晰、正确、文档组织良好的并发程序。

>- [第78项：同步访问共享的可变数据
>- [第79项：避免过度同步
>- [第80项：executor、task和stream优先于线程
>- [第81项：并发工具优先于wait和notify
>- [第82项：线程安全性的文档化
>- [第83项：慎用延迟初始化
>- [第84项：不要依赖于线程调度器

---
### 第十二章 序列化

&emsp;&emsp;本章关注*对象序列化（object serialization）* ，它是一个Java的框架，用来将对象编码为字节流（*序列化（serializing）*），并从其编码中重构对象（*反序列化（deserializing）*）。一旦对象被序列化，其编码可以从一个VM发送到另一个VM或存储在磁盘上以便以后反序列化。本章重点介绍序列化的危险以及如何将序列化最小化。

>- [第85项：其他方法优先于Java序列化
>- [第86项：谨慎地实现Serializable接口
>- [第87项：考虑使用自定义的序列化形式
>- [第88项：保护性地编写readObject方法
>- [第89项：对于实例控制，枚举类型优先于readResolve
>- [第90项：考虑用序列化代理代替序列化实例

---
### 附录 与第2版中项目的对应关系
### 参考文献