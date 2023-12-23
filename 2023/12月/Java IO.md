# Java IO

## IO流的结构和分类

从数据**传输方式**或者说是**运输方式**角度看，可以将 IO 类分为：字节流和字符流。

### 字节流和字符流的区别

* 字节流读取**单个字节**，字符流读取**单个字符**(一个字符根据编码的不同，对应的字节也不同，如 UTF-8 编码中文汉字是 3 个字节，GBK编码中文汉字是 2 个字节。)

* 字节流用来处理**二进制文件**(图片、MP3、视频文件)，字符流用来处理**文本文件**(可以看做是特殊的二进制文件，使用了某种编码，人可以阅读)。

### 按照字节流分类

* 输入流

    ![InputStream](/assert/InputStream.png)

* 输出流

    ![](/assert/OutputStream.png)

### 按照字符流分类

* 输入字符

    ![](/assert/Reader.png)

* 输出字符

    ![](/assert/Writer.png)

### 按照功能分类

1. 文件操作

   FileInputStream、FileOutputStream、FileReader、FileWriter

2. 数组操作（内存操作）

   * 字节数组(byte[]): ByteArrayInputStream、ByteArrayOutputStream
   * 字符数组(char[]): CharArrayReader、CharArrayWriter

3. 管道操作

   PipedInputStream、PipedOutputStream、PipedReader、PipedWriter

4. 基本数据类型操作

   DataInputStream、DataOutputStream

5. 缓冲操作（提高IO性能）

   BufferedInputStream、BufferedOutputStream、BufferedReader、BufferedWriter

6. 打印操作

   PrintStream、PrintWriter

7. 对象序列化和反序列化

   ObjectInputStream、ObjectOutputStream

8. 字节转字符（编码控制）

   InputStreamReader、OutputStreamWriter

## NIO

