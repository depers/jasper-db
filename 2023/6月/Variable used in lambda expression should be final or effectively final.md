> Java

> 这句话的意思是说Lambda 表达式只能引用标记了 final 的外层局部变量，也就是说不能在 Lambda 内部修改定义在域外的局部变量，否则会编译错误。

# 捕获 Lambda

![final引发的错误](/assert/final引发的错误.png)

Lambda 表达式可以使用在外部作用域中定义的变量。 我们将这些 Lambda表达式称为**捕获 Lambda**。 它们可以捕获**静态变量**、**实例变量**、**局部变量**和**方法参数**，但只有局部变量必须声明为final或effectively final。

在早期的 Java 版本中，当匿名内部类捕获一个局部变量到包围它的方法时，我们会遇到这种情况——我们需要在局部变量之前添加 final 关键字，以使编译器通过。

## 什么是effectively final

effectively final作为Java8引入的新特性，它允许我们不为变量、字段和参数编写 final 修饰符，这些变量、字段和参数可以像 final 一样有效地处理和使用。

**一个非final 的局部变量或方法参数，其值在初始化后就从未更改，那么该变量就是effectively final**。 在Lambda 表达式中，使用局部变量的时候，要求该变量必须是`final` 的，所以effectively final 在 Lambda 表达式上下文中非常有用。

```java
public static void main(String[] args) {
    String name = "hello";
    new Runnable() {
        @Override
        public void run() {
            System.out.println(name);
        }
    };
}
```

上面代码中的name就是一个effectively final变量。

effectively final作为一个语法糖，虽然 `final` 关键字不用显示声明出来，但现在编译器可以识别出这个值是`final`的，这意味着它实际上是`final`的。

# 捕获 Lambda 中的局部变量

首先看一段代码：
```java
Supplier<Integer> incrementer() {
  int start = 0;
  return () -> start++; // 不能修改Lambda表达式中的start参数
}
```

这段代码是无法编译的，因为按照之前捕获逻辑，局部变量会被编译器隐式添加了`final`关键字的，局部变量start在下面的Lambda表达式中进行了修改。

如果我们把start变量声明为一个实例变量，是不是就可以编译通过了呢？

```java
public class LambdaVariables {
  private int start = 0;
  
  Supplier<Integer> incrementer() {
        return () -> start++; 
    		/** 编译之后这里的代码其实是：
    		* return () -> {
        *    return this.start++;
        * };
        */
  }
}
```

答案是这段代码是可以编译通过的。

按照JVM的设计，局部变量是存储在虚拟机栈栈帧中的局部变量表中的，一个方法就是一个栈帧，每个方法从调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。当这个方法执行完毕，栈帧从栈中出栈，随后这块内存就会被回收。而实例变量是存储在堆上的，只要这个对象没有被垃圾回收那么这个值就还是存在的。一般来说，捕获实例变量时，我们可以认为是捕获变量`this`。

# 捕获Lamdba中局部变量的并发问题

我们来看下这段代码，代码中我们模拟了在多线程环境下捕获lamdba的情况，这里的`run`变量不是`final`的，这段代码也无法编译通过。

这里明显存在**可见性**问题，每个线程都有自己的堆栈，那么我们如何确保我们的 `while` 循环看到另一个堆栈中运行变量的变化？

```java
public void localVariableMultithreading() {
    boolean run = true;
    executor.execute(() -> {
        while (run) {
            // do operation
        }
    });
    
    run = false;
}
```

参照上一小节的解决方法，我们将`run`变量变为实例变量，为了保证可见性又为其添加了`volatile`关键字。

```java
private volatile boolean run = true;

public void instanceVariableMultithreading() {
    executor.execute(() -> {
        while (run) {
            // do operation
        }
    });

    run = false;
}
```

# 避免使用变量持有者的变通方案

为了绕过局部变量必须声明为final的限制，我们使用一个数组作为变量的持有者，在Lambda表达式中使用通过数组来使用变量的值。

在单线程应用程序中使用数组存储变量的具体代码如下：

```java
public int workaroundSingleThread() {
    int[] holder = new int[] { 2 };
    IntStream sums = IntStream
      .of(1, 2, 3)
      .map(val -> val + holder[0]);

    holder[0] = 0;

    return sums.sum();
}
```

大家猜一猜这段代码的输出是多少？答案是：6。原因是在执行Lambda时使用的是最新值0进行的运算。

在另一个线程中使用数组存储变量的具体代码如下：

```java
public void workaroundMultithreading() {
    int[] holder = new int[] { 2 };
    Runnable runnable = () -> System.out.println(IntStream
      .of(1, 2, 3)
      .map(val -> val + holder[0])
      .sum());

    new Thread(runnable).start();

    // 模拟耗时操作
    try {
        Thread.sleep(new Random().nextInt(3) * 1000L);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }

    holder[0] = 0;
}
```

这段代码的最后输出是12或是6。这个取决于上面这段耗时操作的时间长短。上面这段程序中会有两个线程，一个是main主线程，一个是执行Lambda表达式的线程。如果main线程的执行时间长于执行Lambda表达式线程的执行时间，则最后输出的值是12，否则就是6。

综合上面两个实验，我们的结论是采用数组存储变量参与Lamdba运算的写法是不可行的。原因有两个：

1. 对数组存储变量的修改会直接影响Lambda表达式的运算结果。
2. 在多线程环境下，数组存储变量参与Lambda运算还与每个线程的执行情况相关，导致编码引入了多过的复杂性。

# 在Lambda表达式中使用原子类

通常在Lambda 表达式和匿名类中不建议使用局部变量，因为你不知道这些变量会在代码块中如何被使用，在多线程环境下可能会造成意想不到的情况。除了上面提到的方案外，有一种替代方法允许我们在这种情况下修改变量，通过原子性实现线程安全。

java.util.concurrent.atomic 包提供了诸如`AtomicReference` 和 `AtomicInteger`之类的类。 我们可以使用它们以原子方式修改 Lambda 表达式中的变量：

我最近在开发过程中就遇到了一个问题，我需要在lamdba表达式中做一个计数的统计，其中我就使用了`AtomicInteger`这个类做的实现，代码如下图：

```java
public static void main(String[] args) {
    List<Integer> list = new ArrayList<>();
    AtomicInteger count = new AtomicInteger(0);
    list.forEach(item -> {
        // do something
        count.incrementAndGet();
    });
}
```

# 参考文章

* [Why Do Local Variables Used in Lambdas Have to Be Final or Effectively Final?](https://www.baeldung.com/java-Lambda-effectively-final-local-variables)
* [Final vs Effectively Final in Java](https://www.baeldung.com/java-effectively-final)
