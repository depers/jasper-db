| title                | tags | background                                                   | auther | isSlow |
| -------------------- | ---- | ------------------------------------------------------------ | ------ | ------ |
| Java泛型的协变和逆变 | Java | 最近在工作中用到了泛型，之前对泛型也做过整体的学习，但是对于泛型边界描述符或者是说协变和逆变掌握的不够清楚，学的很疑惑，索性通过这篇文章清晰的来梳理一下。 | depers | true   |

# java.util.Collections#copy

关于协变和逆变，请大家记住这个方法。

![Java泛型的协变和逆变](/assert/Java泛型的协变和逆变.png)

# 变型(variant)

编程语言类型系统中的变型，是用来描述类型构造器是如何界定父子类型之间关系的。主要分为三类：

前提：我声明了两个类，一个是`Animal`类，一个是`Dog`类，其中`Dog`是`Animal`类的子类，`f(type)`就是指`type`类型经过`f()`类型构造器的处理转换为一个新的类型。

* 协变（covariant）：原本的父类和子类，经过类型构造器的处理后，`f(Dog)`还是`f(Animal)`的子类；也就是说`Dog`类经过类型构造器的处理后是`Animal`类经过类型构造器处理后的子类。
* 逆变（contravariant）：就是指`f(Animal)`是`f(Dog)`的子类。
* 不变（invariant）：就是指`f(Dog)`和`f(Animal)`之间没有关系。既不是协变也不是逆变。

类型构造器f(type)可以是

* 泛型：`List`、`List`
* 数组：`Animal[]`、`Dog[]`
* 函数/方法：`method(Animal)`、`method(Dog)`

## Java数组的协变规则

1. 数组是支持协变的

    我们声明了两个类Animal和Dog，其中Animal是Dog的父类。下面这段代码是可以编译运行的：

    ```java
    // 数组是支持协变的
    Animal[] animals = new Dog[10];
    animals[0] = new Animal();
    animals[1] = new Dog();
    ```

2. 为什么Java的泛型是支持泛型的

    一是Java数组是1.0版本提出来的，那个时候还没有泛型。

    二是Java设计者希望能够对数组进行通用的处理，如果数组不支持协变的话，很多方法就得为每一种类型编写逻辑了，如下所示：

    ```java
    public static void process(Object[] array) {
    
    }
    
    public static void process(Animal[] animals) {
    
    }
    
    public static void process(Dog[] dogs) {
    
    }
    
    public static void process(Cat[] cats) {
    
    }
    ```

3. 数组的协变存在安全问题

    数组的协变存在安全问题，如果将其他类型的对象赋值到数组中会报错，代码如下：

    ```java
    // a 是单元素的 String 数组
    String[] a = new String[1];
    
    // b 是 Object 的数组
    Object[] b = a;
    
    // 向 b 中赋一个整数。如果 b 确实是 Object 的数组，这是可能的；然而它其实是个 String 的数组，因此会发生 java.lang.ArrayStoreException
    b[0] = 1;
    ```

## Java的泛型的不变规则

让我们通过下面的代码试一下，看下Java的泛型是否支持协变：

```java
// 协变
List<Animal> animalList = new ArrayList<Dog>(); // 编译失败

// 逆变
List<Dog> dogList = new ArrayList<Animal>(); // 编译失败

// 不变
List<Animal> list = new ArrayList<Animal>(); // 编译成功
```

由此可见，Java的泛型只支持不变规则。

Java为什么不支持协变，如果支持协变，那我就可以随便将其它类型的对象放到List中，要知道在Java的List中只能放置同一类型的对象。

```java
List<Animal> animalList = new ArrayList<Dog>();  // 编译失败
animalList.add(new Dog()); // 编译失败
animalList.add(new Cat());  // 编译失败
```

# Java的泛型的协变和逆变

通过上面一节的介绍，大家应该清楚了Java的泛型是不支持协变和逆变的，但是出于多态设计思想，相同逻辑的方法不能因为参数类型的不同而重复写多遍。所以Java的泛型需要支持协变和逆变。

## 泛型的协变，或者说上界描述符

1. 知识点

    * 作用：对泛型参数的上界进行限制，必须是泛型T或是他的子类。
    * 作为方法参数时控制泛型参数类型为指定类型或是他的子类。
    * 上界描述符`extends`适合读取（get）的场景（可以获取具体类型和`Object`类型），并不适合写入（set，传入`null`除外）的场景。

2. 为什么要让泛型实现协变，其主要目的还是为了实现多态，即不需要通过重载多个方法，造成大量的冗余代码，如下所示：

    ```java
    public static void main(String[] args) {
        process(new ArrayList<Animal>());
        //process(new ArrayList<Dog>()); // 编译错误
        //process(new ArrayList<Cat>()); // 编译错误
    }
    
    
    public static void process(List<Animal> list) {
        System.out.println("...");
    }
    ```

3. 如下面的代码所示，为了让方法process接受Animal子类的集合，我们使用了`extends`关键字 其中`?`被称为通配符，用来表示不确定的类型，通配符和`extends`关键字组成了Java泛型的上界描述符。

    ```java
    public static void main(String[] args) {
        process(new ArrayList<Animal>());
        process(new ArrayList<Dog>());
        process(new ArrayList<Cat>());
    }
    
    
    public static void process(List<? extends Animal> list) {
        Animal animal = list.get(0);
        System.out.println("...");
    }
    ```

4. 泛型的上界描述符，只运行读取，不允许插入（null除外）。

    ```java
    public static void main(String[] args) {
        List<? extends Animal> animals = new ArrayList<>();
        animals.add(null);
    
        //animals.add(new Dog()); // 编译错误
        //animals.add(new Cat()); // 编译错误
    }
    ```

5. Collection类的泛型设计

    下面的三个方法中，可以看到只有`add`方法的参数使用了泛型，这是因为`add`方法是**写入操作**，泛型的上界描述符**不允许进行插入**，可以通过泛型对插入的数据进行校验。

     而`contains()`和`remove()`方法是没有使用泛型的，是因为这两个方法都**不是写入操作**，所以不需要做泛型类型的控制。 

    * `boolean add(E e); `
    * `boolean contains(Object o); `
    * `boolean remove(Object o);`

    实验代码：

    ```java
    public static void main(String[] args) {
        Collection<? extends Animal> animals = new ArrayList<>();
        Dog dog = new Dog();
    
        //animals.add(dog);  // 编译错误
        animals.contains(dog);
        animals.remove(dog);
    }
    ```

6. 上界描述符的作用

    * 如果`extends`关键字用于类上，则他可以对成员变量的泛型类型进行校验。
    * 成员变量可以访问泛型父类的方法。

    具体的实验代码如下：

    ```java
    public class Node <T extends Animal> {  // 编译后没有类型擦除：public class Node<T extends Animal>
        /**
         * extends关键字的用途
         * 1.如果extends关键字用于类上，则他可以对成员变量的泛型类型进行校验
         * 2.成员变量可以访问父类的方法
         */
    
    
        private final T animal; // 编译后没有类型擦除：private final Animal animal;
    
        public static void main(String[] args) {
            Node<Animal> animalNode;
            Node<Dog> dogNode;
            Node<Cat> catNode;
            
            // 可以对成员变量的泛型类型进行校验
            // Node<String> stringNode; // 编译错误
        }
    
    
        public Node(T animal) { // 编译后没有类型擦除：public void Node(Animal animal)
            this.animal = animal;
        }
    
        // 成员变量可以访问泛型父类的方法。
        public void process() {
            animal.eat();
            animal.sleep();
        }
    }
    ```

## 泛型的逆变，或者说下界描述符

1. 知识点

    - 作用：对泛型参数的下界进行限制，必须是泛型T或者他的父类。
    - 作为方法参数时控制泛型参数类型为指定类型或是他的父类。
    - 下界描述符`super`适合写入（set）的场景，并不适合读取（get只能拿到`Object`类型）的场景。

2. 我们现在有一个场景，将`src`列表的元素复制到`dest`列表中，我们该怎么做呢？

    下面的代码中，我们没有使用边界符。可以看到由于泛型不支持协变，所以我们之间将`Dog`的子类传递给方法参数是会变异失败的。

    ```java
    public static void copyHappyDog(List<Dog> dest, List<Dog> src) {
        for (Dog dog : src) {
            dest.add(dog);
        }
    }
    
    
    /**
     * 如果将Dog子类的集合传入到src中就会编译失败
     */
    public static void main(String[] args) {
    
        copyHappyDog(new ArrayList<Dog>(), new ArrayList<Dog>());
        //copyHappyDog(new ArrayList<Dog>(), new ArrayList<HsqDog>()); // 编译失败
        //copyHappyDog(new ArrayList<Dog>(), new ArrayList<SmyDog>()); // 编译失败
    }
    ```

    按照上面的讲解，我们其实可以使用`extends`上界描述符，使泛型支持协变。除了刚才的需求外，我们还想将源集合中的元素添加到`Dog`父类的集合中，该怎么办呢，直接添加到`Animal`容器和`Object`容器是会报错的？

    ```java
    public static void copyHappyDog(List<Dog> dest, List<? extends Dog> src) {
        for (Dog dog : src) {
            dest.add(dog);
        }
    }
    
    
    /**
     * copyHappyDog()方法的src参数使用上边界符进行限定，Dog子类的列表都可以接受
     * 但是如果参数dest传递Dog父类的集合就会编译失败
     */
    public static void main(String[] args) {
    
        copyHappyDog(new ArrayList<Dog>(), new ArrayList<Dog>());
        copyHappyDog(new ArrayList<Dog>(), new ArrayList<HsqDog>());
        copyHappyDog(new ArrayList<Dog>(), new ArrayList<SmyDog>());
        //copyHappyDog(new ArrayList<Animal>(), new ArrayList<SmyDog>()); // 编译失败
        //copyHappyDog(new ArrayList<Object>(), new ArrayList<SmyDog>()); // 编译失败
    }
    ```

    这里我们就需要让泛型支持逆变，也就是说我们可以将一个经过类型构造器处理的新的父类型传递给新的子类型，从而实现逆变。这样的话，`copyHappyDog()`方法中，我们就可以将源容器中的元素添加到目标容器的父容器中了。

    ```java
    public static void copyHappyDog(List<? super Dog> dest, List<? extends Dog> src) {
        for (Dog dog : src) {
            dest.add(dog);
        }
    }
    
    
    /**
     * copyHappyDog()方法的dest参数使用 下边界符 进行限定，Dog父类的列表都可以接受
     */
    public static void main(String[] args) {
    
        copyHappyDog(new ArrayList<Dog>(), new ArrayList<Dog>());
        copyHappyDog(new ArrayList<Dog>(), new ArrayList<HsqDog>());
        copyHappyDog(new ArrayList<Dog>(), new ArrayList<SmyDog>());
        copyHappyDog(new ArrayList<Animal>(), new ArrayList<SmyDog>());
        copyHappyDog(new ArrayList<Object>(), new ArrayList<SmyDog>());
    }
    
    ```

3. 泛型下界描述符的重要特征

    * 泛型的下界描述符是支持添加元素的，因为他没有类型转换失败的风险。

        ```java
        public static void func(List<? super Dog> list) {
            list.add(new Dog());
            list.add(new HsqDog());
            list.add(new SmyDog());
        }
        ```

    * 泛型的下界描述符修饰的集合**还是只能添加指定类型或是指定类型的子类**，添加其他类型是会编译失败的，**切记不要将接收泛型类和添加泛型类弄混了**。

        ```java
        public static void func(List<? super Dog> list) {
            //list.add(new Animal()); // 添加其他元素就会编译错误
            //list.add(new Cat());  // 编译错误
        }
        ```

## 使用场景

大家就会发现，协变和逆变正好相反，上界发生协变只读不写，下界发生逆变只写不读。所以两者的使用场景也就出来了，正如`copyHappyDog()`方法。

```java
/** 
 * 我们现在有一个场景，将src列表的元素复制到dest列表中
 * @param dest 目标列表
 * @param src 源列表
 */
public static void copyHappyDog(List<? super Dog> dest, List<? extends Dog> src) {
    for (Dog dog : src) {
        dest.add(dog);
    }
}
```

* 我对泛型**只读不写**时，使用协变，上界描述符。 
* 如果我对泛型**只写不读**时，使用逆变，下界描述符。 
* 如果我**又想写又想读**，那你不需要使用逆变和协变，你需要**使用准确的泛型类型**。 
* **泛型的协变和逆变一般只使用在方法上**，声明形参来接收调用者传递的实参，之前的代码中使用协变和逆变修饰变量和类是为了方便演示

# PECS原则

- PECS 是指 Producer Extends、Consumer Super，即：
    - 使用 extends 来创建作为生产者的范型容器，该容器只能作为生产者，只能从该容器中读取元素。
    - 使用 super 来创建作为消费者的范型容器，该容器只能作为消费者，只能向该容器里写入元素。

关于该原则，大家不用死记硬背，记住`java.util.Collections#copy`方法就行，它遵守了该原则。

# 参考文章

* [bilibili-RudeCrab-泛型之不变&数组之协变](https://www.bilibili.com/video/BV18d4y1m75K?t=0.5)
* [bilibili-RudeCrab-泛型之协变&上界](https://www.bilibili.com/video/BV1KY4y1w7dq?t=0.7)
* [bilibili-RudeCrab-泛型之逆变&下界](https://www.bilibili.com/video/BV1eU4y1z75J?t=15.3)
* [《Java编程思想》第15章泛型-15.10](https://book.douban.com/subject/2130190/)
* [维基百科-协变与逆变](https://zh.wikipedia.org/zh-hans/%E5%8D%8F%E5%8F%98%E4%B8%8E%E9%80%86%E5%8F%98#%E8%BF%94%E5%9B%9E%E5%80%BC%E7%9A%84%E5%8D%8F%E5%8F%98)