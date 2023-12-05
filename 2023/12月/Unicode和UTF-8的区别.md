> Java/编码

> 最近在做一个分行项目的时候，网络交互使用的是Socket通信，和三方系统在联调接口的时候发现我返回给三方的响应报文编码一直有问题，最后发现是我在写入响应报文的时候没有设置编码格式导致的。

# Unicode、UTF-8和GBK之前的区别

Unicode是一种编码，所谓编码就是一个编号(数字)到字符的一种映射关系，就仅仅是一种一对一的映射而已，可以理解成一个很大的对应表格。Unicode 只是一个符号集，它只规定了符号的二进制代码，却没有规定这个二进制代码应该如何存储。

UTF-8和GBK都是一种编码格式，是用来序列化或存储Unicode编码的数据的。

UTF-8 是全世界统一的一个码表，最大的一个特点，就是它是一种变长的编码方式。它可以使用1~4个字节表示一个符号，根据不同的符号而变化字节长度。

GBK是一种应用于中文的编码格式。

# Java中Unicode、UTF-8 的应用

因为Java在内存中总是使用Unicode表示字符，所以，一个英文字符和一个中文字符都用一个char类型表示，它们都占用两个字节。要显示一个字符的Unicode编码，只需将char类型直接赋值给int类型即可：

```java
public static void main(String[] args) {
    char c1 = 'A'; // A的Unicode表示\u0041
    char c2 = '中'; // 中的Unicode表示\u4e2d
    int i1 = 'A'; // 字母“A”的Unicodde编码是65
    int i2 = '中'; // 汉字“中”的Unicode编码是20013

    System.out.println(i1);
    System.out.println(i2);

    System.out.println(Integer.toHexString(i1));
    System.out.println(Integer.toHexString(i2));
}
```

输出：

```
65
20013
41
4e2d
```

Java的string使用的编码是unicode，但是，当string存在于内存中时(也就是当程序运行时、你在代码中用string类型的引用对它进行操作时、也就是String没有被存在文件中且也没有在网络中传输(序列化)时)，是“**只有编码而没有编码格式的**”。

```java
static void encodingWithCoreJava() {
    String rawString = "中文123abc";
    byte[] bytes = rawString.getBytes(StandardCharsets.UTF_8); // 编码

    String utf8EncodedString = new String(bytes, StandardCharsets.UTF_8); // 解码
    System.out.println(StringUtils.equals(rawString, utf8EncodedString));
}
```

编码：`String.getBytes(charsetName)`得到的`byteArray`是带编码格式的，格式就是你传入的`charsetName`，我们称这个过程为“**编码**”。

解码：`new String(byte[], charsetName)`是把一个byte数组(带编码格式)以`charsetName`指定的编码格式翻译为一个不带编码格式的`String`对象，我们称这个过程为“**解码**”。

所以用什么编码格式编码的，就要用对应的编码格式解码。

# 参考文章

* [Encode a String to UTF-8 in Java](https://www.baeldung.com/java-string-encode-utf-8)
* [关于java中的GBK和UTF-8互转](https://www.jianshu.com/p/f4d4d35feaca)
* [字符编码笔记：ASCII，Unicode 和 UTF-8](https://www.ruanyifeng.com/blog/2007/10/ascii_unicode_and_utf-8.html)