### 第 9 项：第 9 项：try-with-resources 优先于 try-finally

&emsp;&emsp;Java 库包含许多必须通过调用 close 方法手动关闭的资源。示例包括 InputStream，OutputStream 和 java.sql.Connection。关闭资源经常被客户忽视，可预见的可怕性能后果。虽然其中许多资源使用终结方法作为安全网，但终结方法不能很好地工作(第 8 项)。

&emsp;&emsp;在之前的做法中(Historically)，try-finally 语句是保证资源正确关闭的最佳方式，即使出现异常或在 return 的时候(even in the face of an exception or return)：

```java
// try-finally - No longer the best way to close resources!
static String firstLineOfFile(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try {
        return br.readLine();
    } finally {
        br.close();
    }
}
```

&emsp;&emsp;这看起来并不差，但是当你添加第二个资源的时候，它会变得很糟糕：

```java
// try-finally is ugly when used with more than one resource!
static void copy(String src, String dst) throws IOException {
    InputStream in = new FileInputStream(src);
    try {
        OutputStream out = new FileOutputStream(dst);
        try {
            byte[] buf = new byte[BUFFER_SIZE];
            int n;
            while ((n = in.read(buf)) >= 0)
            out.write(buf, 0, n);
        } finally {
            out.close();
        }
    } finally {
        in.close();
    }
}
```

&emsp;&emsp;可能很难相信，但即使是优秀的程序猿也会在大多数时候弄错。对于初学者来说，我在 Java Puzzlers \[Bloch05\]的第 88 页上弄错了，多年来没有人注意到。事实上，2007 年 Java 库中 close 方法的三分之二使用是错误的。

&emsp;&emsp;即使是使用 try-finally 语句关闭资源的正确代码，如前两个代码示例所示，也有一个微妙的缺陷。try 块和 finally 块中的代码都能够抛出异常。例如，在 firstLineOfFile 方法中，对 readLine 的调用可能由于底层物理设备中的故障而引发异常，并且由于相同的原因，对 close 的调用可能会失败。在这种情况下，第二个异常完全覆盖了第一个异常。异常堆栈跟踪中没有第一个异常的记录，这可能会使实际系统中的调试变得非常复杂 - 通常第一个异常才是你要查看并诊断问题的关键所在(usually it’s the first exception that you want to see in order to diagnose the problem)。虽然有可能编写代码来抑制第二个异常而支持第一个异常，但几乎没有人这样做，因为它太冗余了。

&emsp;&emsp;当 Java 7 引入了 try-with-resources 语句\[JLS，14.20.3\]时，所有这些问题都得到了一并解决。要使用此构造，资源必须实现 AutoCloseable 接口，该接口由单个返回类型为 void 的 close 方法组成。Java 库和第三方库中的许多类和接口现在实现或继承 AutoCloseable。如果编写一个代表必须关闭的资源的类( If you write a class that represents a resource that must be closed)，那么你的类也应该实现 AutoCloseable。

&emsp;&emsp;以下是我们的第一个示例使用 try-with-resources 的方式：

```java
// try-with-resources - the the best way to close resources!
static String firstLineOfFile(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    }
}
```

&emsp;&emsp;以下是我们的第二个示例如何使用 try-with-resources:

```java
// try-with-resources on multiple resources - short and sweet
static void copy(String src, String dst) throws IOException {
    try (InputStream in = new FileInputStream(src);
        OutputStream out = new FileOutputStream(dst)) {
        byte[] buf = new byte[BUFFER_SIZE];
        int n;
        while ((n = in.read(buf)) >= 0)
        out.write(buf, 0, n);
    }
}
```

&emsp;&emsp;try-with-resources 版本不仅比原始版本更短，更易读，而且它们提供了更好的诊断功能。考虑 firstLineOfFile 方法。如果 readLine 调用和关闭(不可见)资源时抛出异常，则后一个异常被抑制而有利于前者(the latter exception is suppressed in favor of the former)。实际上，可以抑制多个异常以保留你实际想要查看的异常。这些被抑制的异常不是被丢弃; 它们被打印在跟踪的堆栈中，并带有一个表示它们被压制了的符号。你还可以使用 getSuppressed 方法以编程的方式访问它们，该方法已添加到 Java 7 中的 Throwable 中。

&emsp;&emsp;你可以将 try 子句放在 try-with-resources 语句中，就像在常规的 try-finally 语句中一样。这允许你处理异常，而不会使用另一层嵌套来破坏你的代码。作为一个有点设计想法的例子(As a slightly contrived example)，这里是我们的 firstLineOfFile 方法的一个版本，它不会抛出异常，但如果无法打开文件或读取文件时，则返回默认值：

```java
// try-with-resources with a catch clause
static String firstLineOfFile(String path, String defaultVal) {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    } catch (IOException e) {
        return defaultVal;
    }
}
```

&emsp;&emsp;这里经验教训是很明确的(The lesson is clear)：在处理必须关闭的资源时，相比于 try-finally，始终优先使用 try-with-resources。生成的代码更短更清晰，它生成的异常更有用。try-with-resources 语句可以在使用必须关闭的资源的同同时轻松编写正确的代码，使用 try-finally 几乎是不可能的。

> - [第 8 项：避免使用终结方法和清除方法](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第02章：创建和销毁对象/第8项：避免使用终结方法和清空方法.md)
> - [第 10 项：覆盖 equals 时请遵守通用约定](https://gitee.com/lin-mt/effective-java-third-edition/blob/master/第03章：对于所有对象都通用的方法/第10项：重写equals时请遵守通用约定.md)
