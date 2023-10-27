> Java

> 最近在做项目的时候发现String类型的split() 方法比较特别，有几个需要注意的点，在这里写一篇博客记录下一下。在这篇文章中，我们只做使用的记录，不做源码的解析。

# 一、简单的分割

我们在正常使用时，基本上都是将包含特殊标记串联的字符串，使用`split(特殊标记)`将这个字符串解析成一个数组。比如下面的这串代码：

```java
String str = new String("Welcome-to-Runoob");

System.out.println("- 分隔符返回值 :" );
for (String retval: str.split("-")){
	System.out.println(retval);
}
```

输出：

```
- 分隔符返回值 :
Welcome
to
Runoob
```

# 二、设置分割次数的分割

如果一串字符串，我们想限制其分割的份数，这里就引出了`spilit(regex, limit)`的第二个参数`limit`，这个参数的意思就是**分割的份数**。

JDK17中，对应这个字段的文档描述如下：

1. 如果**limit参数为正**，则模式将最多应用limit - 1次，数组的长度将不大于limit，并且数组的最后一项将包含超出最后匹配分隔符的所有输入。
2. 如果**limit参数为零**，则模式将被应用尽可能多的次数，数组可以有任何长度，并且后面的空字符串将被丢弃。
3. 如果**limit参数是负**，那么模式将被应用尽可能多的次数，数组可以有任何长度。

下面我们针对上述的三种情况分别来做代码说明：

## 1. limit参数为正

若limit是大于0的，分割次数可以大于或小于该字符串的最大可分割份数。

```java
String str = new String("Welcome-to-Runoob");
System.out.println("- 分隔符设置分割份数为正数，分割为2份，返回值 :" );
for (String retval: str.split("-", 2)){
    System.out.println(retval);
}

// 正常分割三次就行，多于3次不会报错
System.out.println("- 分隔符设置分割份数为正数，分割为4份，超过了可分割份数，返回值 :" );
for (String retval: str.split("-", 4)){
    System.out.println(retval);
}
```

输出：

```
- 分隔符设置分割份数为正数，分割为2份，返回值 :
Welcome
to-Runoob

- 分隔符设置分割份数为正数，分割为4份，超过了可分割份数，返回值 :
Welcome
to
Runoob
```

## 2. limit参数为0

当limit为0的时候，比较特别的是分割后的元素若为空字符串则会被丢弃。

```java
// limit为0的情况下
System.out.println("limit为0时的情况，分割后的空字符串将会被丢弃：");
String str2 = new String("a-b-c-d---");
System.out.println("分割后的数组: " + Arrays.toString(str2.split("-")));
System.out.println("分割后的数组: " + Arrays.toString(str2.split("-", 0)));
```

输出：

```
limit为0时的情况，分割后的空字符串将会被丢弃：
分割后的数组: [a, b, c, d]
分割后的数组: [a, b, c, d]
```

## 3. limit参数为负

当limit为负数的时候，这种情况恰好弥补了limit为0时丢弃空字符串的不足。

```java
String str2 = new String("a-b-c-d---");
System.out.println("弥补连续多个结尾分隔符会被丢弃情况:");
System.out.println("limit等于0，分割后的数组: " + Arrays.toString(str2.split("-")));
System.out.println("limit小于0，分割后的数组: " + Arrays.toString(str2.split("-", -1)));
```

输出：

```
弥补连续多个结尾分隔符会被丢弃情况:
limit等于0，分割后的数组: [a, b, c, d]
limit小于0，分割后的数组: [a, b, c, d, , , ]
```

# 三、转义字符作为分割符

若`.`、 `$`、 `|` 和 `* `等转义字符作为分割符，必须得加` \\`。

```java
String str3 = new String("www.runoob.com");
System.out.println("转义字符返回值 :" );
for (String retval: str3.split("\\.", 3)){
    System.out.println(retval);
}
```

输出：

```
转义字符返回值 :
www
runoob
com
```

# 四、多个分隔符

如果一段字符串中涉及多个分隔符进行分割，可以用` |` 作为连字符。

```java
String str4 = new String("acount=? and uu =? or n=?");
System.out.println("多个分隔符返回值 :" );
for (String retval: str4.split("and|or")){
    System.out.println(retval);
}
```

输出：

```
多个分隔符返回值 :
acount=? 
 uu =? 
 n=?
```

