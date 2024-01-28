> Java/IO

> 之前对Java IO这块知识点的理解不够，导致在项目开发的工程中一直对这块存有疑问，所以特定写一篇文章来讨论下这块知识点。

## IO流的结构和分类

从数据**传输方式**或者说是**运输方式**角度看，可以将 IO 类分为：字节流和字符流。

### 字节流和字符流的区别

* 字节流读取**单个字节**，字符流读取**单个字符**(一个字符根据编码的不同，对应的字节也不同，如 UTF-8 编码中文汉字是 3 个字节，GBK编码中文汉字是 2 个字节。)

* 字节流用来处理**二进制文件**(图片、MP3、视频文件)，字符流用来处理**文本文件**(可以看做是特殊的二进制文件，使用了某种编码，人可以阅读)。

### 按照字节流分类

* 输入流

    ![InputStream](/assert/InputStream.png)

* 输出流

    ![OutputStream](/assert/OutputStream.png)

### 按照字符流分类

* 输入字符

    ![Reader](/assert/Reader.png)

* 输出字符

    ![Writer](/assert/Writer.png)

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

Java的NIO其实就是基于Linux的IO复用模型进行构建的，关于这块我这里就不加赘述了，大家自行搜索。值得一说的是IO复用模型其实也是**同步阻塞**的。同步阻塞是两个概念，阻塞说的是**应用进程在询问内核数据准备情况时，是否需要等待**。同步则是说**是否需要应用进程再去调用recvfrom系统调用**。

Java NIO的具体实现主要有三部分组成，分别是Selector选择器、Channel通道、ByteBuffer 缓冲器。

### Selector选择器

* 功能

    * 一个多路复用器 Selector 可以同时轮询多个 Channel，并确定哪个通道已准备好进行通信，即读取或写入。
    * 选择器用于使用单个线程处理多个通道。它只需要较少的线程来处理通道，对于操作系统来讲，可以节省线程之间的切换的高额成本。

* 实践代码

    * 客户端

        ```java
        public class Client {
        
        
            /**
             * 客户端
             */
        
            public static void main(String[] args) throws IOException, InterruptedException {
                ExecutorService executorService = Executors.newFixedThreadPool(20);
                for (int i = 0; i < 100; i++) {
                    executorService.execute(new MultiClient());
                }
                executorService.shutdown();
            }
        
        
            static class MultiClient implements Runnable {
        
                @Override
                public void run() {
                    try {
                        InetSocketAddress hA = new InetSocketAddress("localhost", 8080);
                        SocketChannel client = SocketChannel.open(hA);
                        System.out.println("The Client is sending messages to server...");
                        // Sending messages to the server
                        String[] msg = new String[]{"Time goes fast.", "What next?", "Bye Bye"};
                        for (int j = 0; j < msg.length; j++) {
                            byte[] message = new String(msg[j]).getBytes();
                            ByteBuffer buffer = ByteBuffer.wrap(message);
                            client.write(buffer);
                            System.out.println(msg[j]);
                            buffer.clear();
                            Thread.sleep(3000);
                        }
                        client.close();
                    } catch (Throwable e) {
                        e.printStackTrace();
                    }
        
                }
            }
        
        }
        ```

    * 服务端

        ```java
        public class Server {
        
            /**
             * 服务端
             */
        
        
            public static void main(String[] args) throws IOException {
                // 创建一个选择器
                Selector selector = Selector.open();
                System.out.println("Selector is open for making connection: " + selector.isOpen());
                // 获取服务器套接字通道
                ServerSocketChannel SS = ServerSocketChannel.open();
                InetSocketAddress hostAddress = new InetSocketAddress("localhost", 8080);
                SS.bind(hostAddress);
                SS.configureBlocking(false);
                int ops = SS.validOps();
                // 使用选择器进行注册
                SelectionKey selectKy = SS.register(selector, ops, null);
                for (; ; ) {
                    System.out.println("Waiting for the select operation...");
                    // 获取有多少通道已经准备好进行通信
                    int noOfKeys = selector.select();
                    System.out.println("The Number of selected keys are: " + noOfKeys);
        
                    // 获取需要进行IO处理的channel
                    Set selectedKeys = selector.selectedKeys();
                    Iterator itr = selectedKeys.iterator();
                    while (itr.hasNext()) {
                        SelectionKey ky = (SelectionKey) itr.next();
                        if (ky.isAcceptable()) {
                            // 接受新的客户端连接
                            SocketChannel client = SS.accept();
                            client.configureBlocking(false);
                            // 将新的连接添加到选择器上，并监听OP_READ事件
                            client.register(selector, SelectionKey.OP_READ);
                            System.out.println("The new connection is accepted from the client: " + client);
                        } else if (ky.isReadable()) {
                            // 从客户端读取数据
                            SocketChannel client = (SocketChannel) ky.channel();
                            ByteBuffer buffer = ByteBuffer.allocate(256);
                            int bytesRead = client.read(buffer);
                            buffer.flip();
                            if (bytesRead > 0) {
                                String output = new String(buffer.array()).trim();
                                System.out.println("Message read from client: " + output);
                                if (output.equals("Bye Bye")) {
                                    client.close();
                                    System.out.println("The Client messages are complete; close the session.");
                                }
                            } else if (bytesRead < 0) {
                                // 客户端断开连接
                                client.close();
                            }
        
                        }
                        itr.remove();
                    } // end of while loop
                } // end of for loop
            }
        }
        ```

### Channel通道

* 功能：从缓冲器获取数据，或者向缓冲器发送数据。

- Channel和IO流类似，但也有区别：

    - Channel通道可以读取和写入通道。流通常是单向的（读取或写入）
    - Channel可以异步读取和写入。
    - Channel始终读取或写入缓冲区。

- Java NIO对Channel的重要实现

    - `FileChannel`：从文件中读取和向文件写入
    - `DatagramChannel`：可以通过UDP网络读取和写入数据
    - `SocketChannel`：可以通过TCP网络读取和写入数据
        - 功能：用于将通道与 TCP（传输控制协议）网络套接字连接起来。它等同于网络编程中使用的 Java 网络套接字。
    - `ServerSocketChannel`：监听TCP连接，对于每个传入的连接都会创建一个ScoketChannel

- 实践

    ```java
    public class GetChannel {
    
        /**
         * 从FileInputStream、FileOutputStream、RandomAccessFile中获取FileChannel
         */
    
    
        private static final int BSIZE = 1024;
    
    
        public static void main(String[] args) throws IOException {
    
            // 写一个文件
            FileChannel fc = new FileOutputStream("data.txt").getChannel();
            fc.write(ByteBuffer.wrap("some text".getBytes()));
            fc.close();
    
            // 向文件末尾追加内容
            fc = new RandomAccessFile("data.txt", "rw").getChannel();
            fc.position(fc.size()); // 移动到文件的末尾
            fc.write(ByteBuffer.wrap("some more".getBytes()));
            fc.close();
    
            // 读取文件，只读情况下需要显示的调用allocate()方法来分配ByteBuffer
            fc = new FileInputStream("data.txt").getChannel();
            ByteBuffer buff = ByteBuffer.allocate(BSIZE);
            // 将文件内容读到指定的缓冲区中
            fc.read(buff);
            // 将buffer设置为读模式
            buff.flip();
            while (buff.hasRemaining()) {
                System.out.println((char) buff.get());
            }
        }
    }
    ```

### ByteBuffer缓冲器

* 功能
    * Java NIO 缓冲区在与 NIO 通道交互时使用。数据从通道读取到缓冲区，并从缓冲区写入通道。
    * 缓冲区本质上是一个内存块，您可以在其中写入数据，然后可以稍后再次读取这些数据。

- 核心参数（四个指针）

    - mark
        - 缓冲区的标记。
    - capacity
        - 缓冲区的容量。
        - 默认指向最后一个元素。
    - position
        - 写入模式下，指向插入数据的下一个单元格，最大值为capacity-1。
        - 读取模式下，指向下一个要读取数据的位置。
        - 默认指向第一个元素。
    - limit
        - 写入模式下，指的是写入缓冲区数据量的限制，等于capacity。
        - 读取模式下，指的是从缓冲区可以读取数据量的限制。
        - 默认指向最后一个元素。

- 缓冲区类型（视图缓冲器）

    - `ByteBuffer`：字节缓冲区
    - `MappedByteBuffer`：MappedByte缓冲区
    - `CharBuffer`：字符缓冲区
    - `DoubleBuffer`：双浮点缓冲区
    - `FloatBuffer`：浮点缓冲区
    - `IntBuffer`：整型缓冲区
    - `LongBuffer` ：长整型缓冲区
    - `ShortBuffer` ：短整型缓冲区

- 核心方法

    - `flip()`：将limit设置为position，position设置为0。切换读写模式，此方法用于准备从缓冲区读取已经写入的数据，建议在`channel.write(buffer)`或是`buffer.get()`之前调用。

- 实践

    ```java
    public class GetData {
      
        /**
        * ByteBuffer支持的基本类型
        */
    
    
        private static final int bsize = 1024;
    
        public static void main(String[] args) {
            // 默认分配的字节都是0
            ByteBuffer bb = ByteBuffer.allocate(1024);
    
            int i = 0;
            while (i++ < bb.limit()) {
                if (bb.get() != 0) {
                    System.out.println("nonzero");
                }
            }
    
            System.out.println("i = " + i);
            bb.rewind();
            // 存储和读取一个字符数组
            bb.asCharBuffer().put("fengxiao!");
            char c;
            while ((c = bb.getChar()) != 0) {
                System.out.print(c + " ");
            }
            System.out.println();
    
            bb.rewind();
            // 存储和读取一个short
            bb.asShortBuffer().put((short) 471142);
            System.out.println(bb.getShort());
    
            // 存储和读取一个int
            bb.rewind();
            bb.asIntBuffer().put(99471142);
            System.out.println(bb.getInt());
    
            // 存储和读取一个long
            bb.rewind();
            bb.asLongBuffer().put(99471142);
            System.out.println(bb.getLong());
    
            // 存储和读取一个float
            bb.rewind();
            bb.asFloatBuffer().put(99471142);
            System.out.println(bb.getFloat());
    
            // 存储和读取一个double
            bb.rewind();
            bb.asDoubleBuffer().put(99471142);
            System.out.println(bb.getDouble());
        }
    }
    ```
    
    

