##  优先try-with-resource而不是try-finally

> The Java libraries include many resources that must be closed manually by invok-
> ing a close method. Examples include InputStream, OutputStream, and
> java.sql.Connection. Closing resources is often overlooked by clients, with
> predictably dire performance consequences. While many of these resources use
> finalizers as a safety net, finalizers don’t work very well (Item 8).Historically, a try-finally statement was the best way to guarantee that aresource would be closed properly, even in the face of an exception or return:

在Java类库中包含了很多必须通过手动调用close()方法来关闭资源。例如：`InputStream`,`OutputStream`还有`java.sql.Connection`。关闭资源这个动作很容易被使用者忽略，其性能表现可想而知。虽然大部分这些资源都是用终结方法作为最后的安全线。但是这种终结方法的效果并不是非常的好。在以前我们编写代码的时候，这个`try-finally`语句是我们保证一个资源被关闭的最好的方式，即使是在这个程序抛出异常或者返回的情况下。

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

> This may not look bad, but it gets worse when you add a second resource:

这看起来并不是很糟糕，但是如果你需要添加第二个资源的时候：

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

> It may be hard to believe, but even good programmers got this wrong most of the time. For starters, I got it wrong on page 88 of *Java Puzzlers*[Bloch05], and no one noticed for years. In fact, two-thirds of the uses of the close method in the Java libraries were wrong in 2007.

这个可能会难以置信，即使好的程序猿在很多时候也会犯这个错误。首先，我在Java Puzzlers*[Bloch05]这本书的第88页上搞错了，并且很多年过去了也没有人发现。事实上，在2007年的Java类库中有2/3的`close`方法都用错了。

> Even the correct code for closing resources with try-finally statements, as
> illustrated in the previous two code examples, has a subtle deficiency. The code in
> both the try block and the finally block is capable of throwing exceptions. For
> example, in the firstLineOfFile method, the call to readLine could throw an
> exception due to a failure in the underlying physical device, and the call to close
> could then fail for the same reason. Under these circumstances, the second
> exception completely obliterates the first one. There is no record of the first
> exception in the exception stack trace, which can greatly complicate debugging in
> real systems—usually it’s the first exception that you want to see in order to
> diagnose the problem. While it is possible to write code to suppress the second
> exception in favor of the first, virtually no one did because it’s just too verbose.

即使我们正确的使用了try-resource方法。就像上面两个例子一样，还是会有小缺陷的。try块和finally块的代码都可能抛出异常。例如在`firstLineOfFile`这个方法中，对`readLine`方法的调用，可能会由于底层的物理设备的故障而引发异常，同理对close()方法的调用也可能会失败。但是在这种情况下，close()方法的异常可能会覆盖掉的第一个异常。异常堆栈中没有第一个异常的记录，这个可能会导致实际系统中的调试变得非常复杂。但是通常来说我们需要进行问题调试的时候应该是第一个异常。虽然可能编写代码来抑制第二个异常来保证第一个异常能够正常显示，但是基本上没有人会这么做，因为这会导致代码变得非常冗余。

> All of these problems were solved in one fell swoop when Java 7 introduced
> the try-with-resources statement [JLS, 14.20.3]. To be usable with this construct,
> a resource must implement the AutoCloseable interface, which consists of a
> single void-returning close method. Many classes and interfaces in the Java
> libraries and in third-party libraries now implement or extend AutoCloseable. If
> you write a class that represents a resource that must be closed, your class should
> implement AutoCloseable too.Here’s how our first example looks using try-with-resources:

当Java7引入了try-with-resource语句 [JLS, 14.20.3]的时候，所有的问题都被解决了。要使用这个语法糖，该类必须要实现`AutoCloseable`接口。该接口有一个返回值为void的`close()`方法组成。Java库和第三方库和接口很多都实现或者扩展了AutoCloseable。如果编写一个必须关闭资源的类，MAME您的类也应该实现`AutoCloseable`这个接口。

下面是第一个例子使用try-with-reources语法糖

```java
   // try-with-resources - the the best way to close resources!
   static String firstLineOfFile(String path) throws IOException {
       try (BufferedReader br = new BufferedReader(
               new FileReader(path))) {
           return br.readLine();
} }
```

> And here’s how our second example looks using try-with-resources:

这是第二个例子使用`try-with-reources`

```java
// try-with-resources on multiple resources - short and sweet
static void copy(String src, String dst) throws IOException {
  try (InputStream   in = new FileInputStream(src);
     OutputStream out = new FileOutputStream(dst)) {
     byte[] buf = new byte[BUFFER_SIZE];
     int n;
     while ((n = in.read(buf)) >= 0)
     out.write(buf, 0, n);
  } 
}
```

> Not only are the try-with-resources versions shorter and more readable than the originals, but they provide far better diagnostics. Consider the firstLineOfFile method. If exceptions are thrown by both the readLine call and the (invisible)
> close, the latter exception is *suppressed* in favor of the former. In fact, multiple
> exceptions may be suppressed in order to preserve the exception that you actually
> want to see. These suppressed exceptions are not merely discarded; they are
> printed in the stack trace with a notation saying that they were suppressed. You
> can also access them programmatically with the getSuppressed method, which
> was added to Throwable in Java 7.

`try-with-resource`语法糖不仅比原始语法更短，更易读，而且它们提供了更好的调试功能。 考虑到在`firstLineOfFile`方法中，如果在调用的时候`readLine`和`close()`方法都出现了异常，后面的异常将会覆盖前面的异常。实际上我们可以通过控制后者那个异常来保证我们自己想要看到的异常能够正常的显示出来。这些被限制的异常不仅被丢弃，他们在打印堆栈信息的时候也会被其他的 标记。在Java7中我们可以通过`getSuppressed`访问close()方法中抛出的异常。

> You can put catch clauses on try-with-resources statements, just as you can
> on regular try-finally statements. This allows you to handle exceptions without
> sullying your code with another layer of nesting. As a slightly contrived example,
> here’s a version our firstLineOfFile method that does not throw exceptions, but
> takes a default value to return if it can’t open the file or read from it:

当然你可以在使用`try-with-resource`时候使用catch去捕获异常。我们可以在方法体里面进行处理异常，不让改异常不向上抛出。下面是`firstLineOfFile`的另一个例子，它不会抛出异常，当无法打开文件的时候，他将返回默认值。

```java
// try-with-resources with a catch clause
   static String firstLineOfFile(String path, String defaultVal) {
       try (BufferedReader br = new BufferedReader(
               new FileReader(path))) {
           return br.readLine();
       } catch (IOException e) {
           return defaultVal;
       } 
   }

```

> The lesson is clear: Always use try-with-resources in preference to try-
> finally when working with resources that must be closed. The resulting code is
> shorter and clearer, and the exceptions that it generates are more useful. The try-
> with-resources statement makes it easy to write correct code using resources that
> must be closed, which was practically impossible using try-finally.

总结：当该资源必须需要手动关闭的时候使用`try-with-resources`会比`try-
finally`更好，编写出来的代码更加精简，更加清晰，并且其生成的异常也会更有用。这个`try-with-resources`语法糖可以让我们比使用`try-finally`更容易编写正确关闭资源的代码。