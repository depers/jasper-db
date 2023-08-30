> Java

> 最近在开发过程中我想获取泛型参数的真实使用类型，遇到了ParameterizedType这个类，但是我发现在Java中ParameterizedType只是其中的一个泛型类型，Java开发了基于Type类型的泛型类型体系，下面通过这篇文章一起和我来学习一下吧。

# Type接口的继承关系

Java语言从Java5以后就引入**Type**体系，应该是为了加入泛型而引入的。先看下`Type`接口的类继承图：

![Java中的Type类型体系](/assert/Java中的Type类型体系.png)

从上面的图中，我们可以看到`Type`类由**4个子接口**和**1个子类**组成：

4个子接口：

* **参数化类型**：*ParameterizedType*

* **类型变量**：*TypeVariable* 

* **泛型数组类型**：*GenericArrayType*

* **通配符类型**：*WildcardType*

1个子类:

* ***Class*类型**

# ParameterizedType

`ParameterizedType` **参数化类型**，即带参数的类型，也可以说带`<>`的类型。例如`List<String>`, `User<T>` 等。

关键方法：

* `getActualTypeArguments()`：获取`<>`中的类型定义，例如`Map<K,V>` 那么就得到` [K,V]`的一个数组。
* `getRawType()`：获取`<>`前面的类型，例如例如`Map<K,V> `那么就得到 `Map`。
* `getOwnerType()`：获取当前类型的上一层类型，若当前类型为顶层类型，则返回`null`；例如`Map`有一个内部类`Entry,`  那么在`Map.Entry<K,V>` 上调用这个方法就可以获得 `Map`。

具体实践：

1. 首先我们声明一个类`ParameterizedTypeExample`，我们可以看到这个类，声明了2个字段和1个方法：

   ```java
   public class ParameterizedTypeExample<K, V> {
       
       private Map<K, V> map;
   
       private Map.Entry<String, String> entry;
       
       private ParameterizedTypeExample2 example;
   
       public Map<K, V> getMap(Map<K, V> map){
           return map;
       }
   }
   ```

2. 声明类`ParameterizedTypeExample`的子类`ParameterizedTypeExample2`，子类复写了父类的`getMap()`方法，并且在类的泛型参数上声明了具体的类型，分别是`String`和`Student`类型。

   ```java
   public class ParameterizedTypeExample2 extends ParameterizedTypeExample<String, Student> {
   
       @Override
       public Map<String, Student> getMap(Map<String, Student> map) {
           return super.getMap(map);
       }
   }
   ```

3. 首先我们来测试这一对父子类的`ParameterizedType`类型的信息，值得注意的是：

   * 泛型父类的类型其实是`Object`类型。
   * 泛型子类的父类类型是会携带子类声明的泛型参数。

   ```java
   public static void main(String[] args) throws NoSuchMethodException, NoSuchFieldException {
       testMethod(new ParameterizedTypeExample());
       System.out.println("-------------------------------");
       testClass(new ParameterizedTypeExample2());
   }
   
   
   static void testClass(ParameterizedTypeExample parameterizedTypeExample) {
       // 获取该类父类的类型
       Type type = parameterizedTypeExample.getClass().getGenericSuperclass();
       System.out.println("类的类型：" + type);
   
       // 获取父类类型参数的真实类型
       if (type instanceof ParameterizedType) {
           Type[] types = ((ParameterizedType)type).getActualTypeArguments();
           System.out.println("类的参数化类型的真实参数：" + Arrays.toString(types));
           System.out.println("类的外层类型：" + ((ParameterizedType) type).getRawType());
           System.out.println("类的上层类型：" + ((ParameterizedType) type).getOwnerType());
       }
   }
   ```

   输出：

   ```
   类的类型：class java.lang.Object
   -------------------------------
   类的类型：cn.bravedawn.reflection.type.parameterizedtype.ParameterizedTypeExample<java.lang.String, cn.bravedawn.reflection.type.parameterizedtype.ParameterizedTypeExample$Student>
   类的参数化类型的真实参数：[class java.lang.String, class cn.bravedawn.reflection.type.parameterizedtype.ParameterizedTypeExample$Student]
   类的外层类型：class cn.bravedawn.reflection.type.parameterizedtype.ParameterizedTypeExample
   类的上层类型：null
   ```

4. 接着我们来测试这一对父子类的`getMap()`方法的情况：

   ```java
   public static void main(String[] args) throws NoSuchMethodException, NoSuchFieldException {
       testMethod(new ParameterizedTypeExample());
       System.out.println("-------------------------------");
       testMethod(new ParameterizedTypeExample2());
   }
   
   /**
   * 测试方法
   * @param parameterizedTypeExample
   * @throws NoSuchMethodException
   */
   static void testMethod(ParameterizedTypeExample parameterizedTypeExample) throws NoSuchMethodException {
       // 以方法的返回值为准
       Type type = parameterizedTypeExample.getClass().getDeclaredMethod("getMap", Map.class).getGenericReturnType();
   
       System.out.println("从方法形参中获取参数类型：" + Arrays.toString(parameterTypes));
   
       System.out.println("是否为参数化类型：" + (type instanceof ParameterizedType));
       ParameterizedType parameterizedType = (ParameterizedType) type;
   
       // 获得参数化类型中<>里面的类型参数的类型
       System.out.print("实际类型是：");
       Arrays.stream(parameterizedType.getActualTypeArguments()).forEach(t -> {
           System.out.print(t + "; ");
       });
       System.out.println();
       // 返回最外层<>前面那个类型，例如Map<K ,V>，针对K或V来说，返回的就是Map类型
       System.out.println("外层类型：" + parameterizedType.getRawType());
   
       // 如果当前类型为顶层类型，则返回null（通常情况是这样），例如：Map.Entry中的Entry返回的就是Map，A.B.C中的C返回的就是A$B
       System.out.println("上层类型：" + parameterizedType.getOwnerType());
   }
   
   ```

   输出：

   ```
   是否为参数化类型：true
   实际类型是：K; V; 
   外层类型：interface java.util.Map
   上层类型：null
   -------------------------------
   是否为参数化类型：true
   实际类型是：class java.lang.String; class cn.bravedawn.reflection.type.parameterizedtype.ParameterizedTypeExample$Student; 
   外层类型：interface java.util.Map
   上层类型：null
   ```

5. 测试属性的参数化类型，这里我们以`ParameterizedTypeExample`的`entry`属性为例，值得注意的是`entry`参数是有上层类型的，因为这个参数在Map类型的声明是`interface Entry<K, V>`：

   ```java
   public static void main(String[] args) throws NoSuchMethodException, NoSuchFieldException {
       testField("entry");
       System.out.println("-------------------------------");
       testField("example");
   }
   
   
   /**
   * 测试属性
   * @param fieldName
   * @throws NoSuchFieldException
   */
   private static void testField(String fieldName) throws NoSuchFieldException {
       Field f = ParameterizedTypeExample.class.getDeclaredField(fieldName);
       // 获取属性的类型
       Type fieldType = f.getGenericType();
       System.out.println("属性的类型：" + fieldType);
   
       // 判断属性的类型是否为参数化类型
       boolean b = fieldType instanceof ParameterizedType;
       System.out.println("是否为参数化类型："+ b);
   
       // 如果是的话
       if(b){
           // 获取该字段的参数化类型
           ParameterizedType pType = (ParameterizedType) fieldType;
   
           // 获取参数化类型的实际类型
           System.out.print("实际类型是：");
           for (Type type : pType.getActualTypeArguments()) {
               System.out.print(type + "; ");
           }
           System.out.println();
           // 返回最外层<>前面那个类型
           System.out.println("外层类型：" + pType.getRawType());
   
           // 如果当前类型为顶层类型，则返回null（通常情况是这样），例如：Map.Entry中的Entry返回的就是Map，A.B.C中的C返回的就是A$B
           System.out.println("上层类型：" + pType.getOwnerType());
       }
   }
   ```

   输出：

   ```
   是否为参数化类型：true
   实际类型是：class java.lang.String; class cn.bravedawn.reflection.type.parameterizedtype.ParameterizedTypeExample$Student; 
   外层类型：interface java.util.Map
   上层类型：null
   -------------------------------
   属性的类型：java.util.Map$Entry<java.lang.String, java.lang.String>
   是否为参数化类型：true
   实际类型是：class java.lang.String; class java.lang.String; 
   外层类型：interface java.util.Map$Entry
   上层类型：interface java.util.Map
   ```

# TypeVariable

`TypeVariable` **类型变量**。例如`List<T>`中的`T`, `Map<K,V>`中的`K`和`V`，测试类 `TypeTest<T, V extends @Custom Number & Serializable>`中的`T`和`V`。

关键方法：

* `getBounds()`：返回此类型参数的上界列表，如果没有上界则放回`Object`. 例如`K extends @MyAnnotation(1) CharSequence & Serializable`这个类型参数，有两个上界，`CharSequence`和`Serializable`。
* `getGenericDeclaration()`：返回类型参数声明时的载体，例如类 `TypeVariableExample<K extends @MyAnnotation(1) CharSequence & Serializable, V>`，其中`K`的载体就是 `TypeVariableExample`。
* `getName()`：返回源码中类型变量的名称。
* `getAnnotatedBounds()`：Java 1.8引入`AnnotatedType`，如果这个这个泛型参数类型的上界用注解标记了，我们可以通过它拿到相应的注解。

下面我们直接看代码：

1. 首先我们声明一个注解，方便稍后在代码中使用，这段代码中值得注意的是`ElementType.TYPE_USE`这个属性，我们用它来修饰类型：

   ```java
   @Target({ElementType.FIELD, ElementType.TYPE_USE, ElementType.TYPE, ElementType.PARAMETER})
   @Retention(RetentionPolicy.RUNTIME)
   public @interface MyAnnotation {
   
       int value() default 1;
   }
   ```

2. 变量类型的演示代码，下面的代码我们主要通过字段的类型声明来做实验：

   ```java
   public class TypeVariableExample<K extends @MyAnnotation(1) CharSequence & Serializable, V> {
   
       @MyAnnotation(2)
       K key;
   
       Map<K, V> map;
   
       public static void main(String[] args) throws NoSuchFieldException {
           TypeVariableExample<String, Integer> example = new TypeVariableExample<>();
   
           testField(example);
       }
   
       static void testField(TypeVariableExample typeVariableExample) throws NoSuchFieldException {
           Field f = typeVariableExample.getClass().getDeclaredField("key");
           Type type = f.getGenericType();
           boolean b = type instanceof TypeVariable;
           System.out.println("是否为类型变量：" + b);
           
           if (b) {
               TypeVariable typeVariable = (TypeVariable) type;
               System.out.println("字段类型：" + type.getClass());
               System.out.println("返回此类型参数的上边界类型数组：" + Arrays.toString(typeVariable.getBounds()));
               System.out.println("类型参数的声明名称：" + typeVariable.getName());
               System.out.println("类型参数声明时的载体：" + typeVariable.getGenericDeclaration());
               System.out.println("类型参数上边界类型的注解：" + Arrays.asList(typeVariable.getAnnotatedBounds()[0].getAnnotations()));
           }
       }
   
   }
   ```

   输出：

   ```
   是否为类型变量：true
   字段类型：class sun.reflect.generics.reflectiveObjects.TypeVariableImpl
   返回此类型参数的上边界类型数组：[interface java.lang.CharSequence, interface java.io.Serializable]
   类型参数的声明名称：K
   类型参数声明时的载体：class cn.bravedawn.reflection.type.typevariable.TypeVariableExample
   类型参数上边界类型的注解：[@cn.bravedawn.reflection.type.typevariable.MyAnnotation(value=1)]
   ```

   这里最值得注意的就是`getAnnotatedBounds()`方法，他返回一个`AnnotatedType`类型的数组，从这个数组中，我们可以获取声明在类型前面的注解。

# GenericArrayType

`GenericArrayType` **泛型数组类型**，用来作为数组的泛型声明类型。例如`List<T>[] ltArray`，` T[] tArray`两个数组，其中`List<T>[]`和`T[]`就是`GenericArrayType`类型。

关键方法：

* `getGenericComponentType()`：获取泛型类型数组的声明类型，即获取数组方括号`[]`前面的部分。

`GenericArrayType` 泛型数组类型比较简单，下面我们来看代码：

```java
public class GenericArrayTypeExample<T> {

    private T[] tArray;

    private List<T>[] tListArray;


    public static void main(String[] args) throws NoSuchFieldException {
        Field tArrayField = GenericArrayTypeExample.class.getDeclaredField("tArray");
        testField(tArrayField);
        System.out.println("----------------------------");

        Field tListArrayField = GenericArrayTypeExample.class.getDeclaredField("tListArray");
        testField(tListArrayField);
    }

    static void testField(Field field) {
        Type type = field.getGenericType();

        System.out.println("数组参数的类型：" + type);
        System.out.println("是否为GenericArrayType类型：" + (type instanceof GenericArrayType));
        System.out.println("泛型类型数组声明的类型：" + ((GenericArrayType) type).getGenericComponentType());
    }
}
```

输出：

```
数组参数的类型：T[]
是否为GenericArrayType类型：true
泛型类型数组声明的类型：T
----------------------------
数组参数的类型：java.util.List<T>[]
是否为GenericArrayType类型：true
泛型类型数组声明的类型：java.util.List<T>
```

# WildcardType

WildcardType，通配符类型，即带有`?`的泛型参数, 例如 `List<?>`中的`?`，`List<? extends Number>`里的`? extends Number`和`List<? super Integer>`的`? super Integer` 。

核心方法：

* `getUpperBounds()`：获取上界。
* `getLowerBounds()`：获取下界。

下面我们来看一段测试代码：

```java
public class WildcardTypeExample {


    private Map<? super String, ? extends Number> map;


    public static void main(String[] args) throws NoSuchFieldException {

        Field mapField = WildcardTypeExample.class.getDeclaredField("map");
        Type type = mapField.getGenericType();

        System.out.println("是否为WildcardType类型：" + (type instanceof WildcardType));
        System.out.println("是否为ParameterizedType类型：" + (type instanceof ParameterizedType));
        Type[] actualTypes = ((ParameterizedType) type).getActualTypeArguments();
        System.out.println("获取<>中的类型定义：" + Arrays.asList(actualTypes));

        System.out.println("实际类型定义是否为WildcardType类型：" + (actualTypes[0] instanceof WildcardType));
        System.out.println("实际类型定义是否为WildcardType类型：" + (actualTypes[1] instanceof WildcardType));
        WildcardType firstWildcardType = (WildcardType) actualTypes[0];
        WildcardType secondWildcardType = (WildcardType) actualTypes[1];
        System.out.println("获取下界：" + Arrays.toString(firstWildcardType.getLowerBounds()));
        System.out.println("获取上界：" + Arrays.toString(secondWildcardType.getUpperBounds()));
    }
}
```

输出：

```
是否为WildcardType类型：false
是否为ParameterizedType类型：true
获取<>中的类型定义：[? super java.lang.String, ? extends java.lang.Number]
实际类型定义是否为WildcardType类型：true
实际类型定义是否为WildcardType类型：true
获取下界：[class java.lang.String]
获取上界：[class java.lang.Number]
```

# Class类型

我们先来看看Java中对这个类的说明：

Class 类的实例表示正在运行的 Java 应用程序中的类和接口。 枚举类和记录类都是类的一种； 注解也是是接口的一种； 每个数组也属于一个类，该类反映为由具有相同元素类型和维数的所有数组共享的 Class 对象。 原始 Java 类型（boolean、byte、char、short、int、long、float 和 double）和关键字 void 也表示为 Class 对象。

在这个例子中，我发现了一个很重要的点，先看代码：

```java
public class ClassExample<T> {

	// 声明一个接口
    interface Example<T> {

    }

    // 1.该类的变量
    private ClassExample classExample;

    // 2.泛型参数具体化的变量
    private ClassExample<String> classExample2;

    // 3.泛型参数没有具体化的变量
    private ClassExample<T> classExample3;

    // 4.测试泛型参数具体化的List变量
    private List<String> list;

    // 5.测试泛型参数没有具体化的List变量
    private List<T> list2;

    // 6.自己实现一个接口的变量，并将其泛型参数具体化
    private Example<String> example;

    // 7.自己实现一个接口的变量，其泛型参数不做具体化
    private Example<T> example2;

    public static void main(String[] args) throws NoSuchFieldException {
        System.out.println("参数：classExample--------------------------------");
        Field classField = ClassExample.class.getDeclaredField("classExample");
        testField(classField);

        System.out.println("参数：classExample--------------------------------");
        Field classField2 = ClassExample.class.getDeclaredField("classExample");
        testField(classField2);

        System.out.println("参数：classExample3--------------------------------");
        Field classField3 = ClassExample.class.getDeclaredField("classExample3");
        testField(classField3);

        System.out.println("参数：list--------------------------------");
        Field list = ClassExample.class.getDeclaredField("list");
        testField(list);

        System.out.println("参数：list2--------------------------------");
        Field list2 = ClassExample.class.getDeclaredField("list2");
        testField(list2);

        System.out.println("参数：example--------------------------------");
        Field example = ClassExample.class.getDeclaredField("example");
        testField(example);

        System.out.println("参数：example2--------------------------------");
        Field example2 = ClassExample.class.getDeclaredField("example2");
        testField(example2);

    }


    static void testField(Field field) {
        Type type = field.getGenericType();
        System.out.println("字段类型：" + type);
        System.out.println("是否为Class类型：" + (type instanceof Class));
        System.out.println("是否为ParameterizedType类型：" + (type instanceof ParameterizedType));
        System.out.println();
    }
}
```

输出：

```
参数：classExample--------------------------------
字段类型：class cn.bravedawn.reflection.type.class_.ClassExample
是否为Class类型：true
是否为ParameterizedType类型：false

参数：classExample--------------------------------
字段类型：class cn.bravedawn.reflection.type.class_.ClassExample
是否为Class类型：true
是否为ParameterizedType类型：false

参数：classExample3--------------------------------
字段类型：cn.bravedawn.reflection.type.class_.ClassExample<T>
是否为Class类型：false
是否为ParameterizedType类型：true

参数：list--------------------------------
字段类型：java.util.List<java.lang.String>
是否为Class类型：false
是否为ParameterizedType类型：true

参数：list2--------------------------------
字段类型：java.util.List<T>
是否为Class类型：false
是否为ParameterizedType类型：true

参数：example--------------------------------
字段类型：cn.bravedawn.reflection.type.class_.ClassExample$Example<java.lang.String>
是否为Class类型：false
是否为ParameterizedType类型：true

参数：example2--------------------------------
字段类型：cn.bravedawn.reflection.type.class_.ClassExample$Example<T>
是否为Class类型：false
是否为ParameterizedType类型：true
```

这个类一共有7个字段，其中第1和第2个参数是`Class`类型，其他均为`ParameterizedType`类型，这是为什么呢？

第三个参数`classExample3`，他的类型是`ClassExample<T>`，很明显他就是`ParameterizedType`类型。

剩余的四个参数都是**接口类型**的变量，所以无论他是否将泛型参数具体化，他都是`ParameterizedType`类型。

## Class类型与ParameterizedType类型的区别

看了上面的例子大家就会发现，判断`User<String>`是`Class`类型还是`ParameterizedType`类型，这取决于`User`是接口还是**类**，如果是类的话`User<String>`就是`Class`类型，如果`User`是**接口**，则`User<String>`就是`ParameterizedType`类型。

# 参考文章

* [秒懂Java类型（Type）系统]()