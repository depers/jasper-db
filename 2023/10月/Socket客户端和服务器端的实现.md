> Java/网络

> 最近在开发乌鲁木齐分行财政项目的时候发现分行与财政系统的联机交互是通过Socket来通信的，之前做的项目都是通过http协议来通信的，再加上自己对Socket编程这块知识点也不是很清楚，所以写一篇博客来梳理一下。

# 什么是Socket

Socket是一个抽象概念，一个应用程序通过一个Socket来建立一个远程连接，而Socket内部通过TCP/IP协议把数据传输到网络。

Socket、TCP和部分IP的功能都是由操作系统提供的，不同的编程语言只是提供了对操作系统调用的简单的封装。例如，Java提供的几个Socket相关的类就封装了操作系统提供的接口。

* ServerSocket：该类实现服务器套接字。服务器套接字等待通过网络进入的请求。它根据该请求执行一些操作，然后将结果返回给请求者。
* Socket：该类实现客户端套接字(也称为“套接字”)。套接字是两台机器之间通信的端点。

使用Socket进行网络编程时，本质上就是两个进程之间的网络通信。其中一个进程必须充当服务器端，它会主动监听某个指定的端口，另一个进程必须充当客户端，它必须主动连接服务器的IP地址和指定端口，如果连接成功，服务器端和客户端就成功地建立了一个TCP连接，双方后续就可以随时发送和接收数据。

# 一个简单的Socket客户端服务端程序

## 服务端

先看代码：

Server.java

```java
public static void main(String[] args) throws IOException {
    // 监听6666端口
    ServerSocket server = new ServerSocket(6666);
    System.out.println("Server is running.");
    while (true) {
        // 接收新的连接
        Socket socket = server.accept();
        System.out.println("connected from " + socket.getRemoteSocketAddress());
        // 处理来自客户端的请求
        Thread t = new Handler(socket);
        t.start();
    }
}
```

Handler.java

```java
public class Handler extends Thread{


    private Socket socket;


    public Handler(Socket socket) {
        this.socket = socket;
    }

    @Override
    public void run() {
        try (InputStream input = socket.getInputStream()) {
            try (OutputStream output = socket.getOutputStream()) {
                handle(input, output);
            }
        } catch (Exception e) {
            try {
                socket.close();
            } catch (IOException ex) {
            }
            System.out.println("client disconnected");
        }
    }


    private void handle(InputStream input, OutputStream output) throws IOException {
        BufferedWriter writer = new BufferedWriter(new OutputStreamWriter(output, StandardCharsets.UTF_8));
        BufferedReader reader = new BufferedReader(new InputStreamReader(input, StandardCharsets.UTF_8));
        writer.write("hello");
        writer.newLine(); // 调用此方法来终止输出行
        writer.flush();

        while (true) {
            String s = reader.readLine();
            if (s.equals("bye")) {
                writer.write("byte");
                writer.newLine(); // 调用此方法来终止输出行
                writer.flush();
                break;
            }
            writer.write("ok: " + s);
            writer.newLine(); // 调用此方法来终止输出行
            writer.flush();
        }

    }
}
```

首先我们来看服务端，在Socket服务端需要做的工作主要有三步：

1. 监听请求的端口。
2. 接收客户端的请求。
3. 处理来自客户端的请求，新建线程来处理客户端的请求，从而实现请求的并发处理。

## 客户端

先看代码：

Client.java

```java
public class Client {

    public static void main(String[] args) throws IOException {
        Socket socket = new Socket("localhost", 6666);
        try (InputStream input = socket.getInputStream()) {
            try (OutputStream output = socket.getOutputStream()) {
                handle(input, output);
            }
        }
        socket.close();
        System.out.println("disconnected.");
    }



    private static void handle(InputStream input, OutputStream output) throws IOException {
        var writer = new BufferedWriter(new OutputStreamWriter(output, StandardCharsets.UTF_8));
        var reader = new BufferedReader(new InputStreamReader(input, StandardCharsets.UTF_8));
        Scanner scanner = new Scanner(System.in);
        System.out.println("[server] " + reader.readLine());
        while (true){
            System.out.print(">>> "); // 打印提示
            String s = scanner.nextLine(); // 读取一行输入
            writer.write(s);
            writer.newLine();
            writer.flush();
            String resp = reader.readLine();
            System.out.println("<<< " + resp);
            if (resp.equals("byte")) {
                break;
            }
        }
    }
}

```

接下来我们来看客户端的相关操作

1. 连接指定的服务器和端口。
2. 发送数据给服务端。
3. 接收服务端的信息。

值得注意的是，无论是客户端还是服务器端，每一条消息的结束都是用一个**换行符**来标识的。我们既可以通过`\n`显示标识，也可以使用`writer.newLine()`来实现。

# 线程池武装的Socket客户端和服务端

下面我来针对服务器端和客户端分别给出使用线程池接收和发送数据的逻辑实现。## 

## 服务端

与上面服务端不同的地方在于新增了线程池的逻辑，具体见SocketServer.java：

下面代码中在服务端等待接受客户端连接、等待客户端数据，客户端等待服务端响应这几个地方都可以设置超时时间，相关的代码注释见下面。

```java
@Slf4j
public class SocketServer extends Thread{

    /**
     *  socket服务端
     */
    private ServerSocket serverSocket;

    /**
     * 线程池
     */
    private ExecutorService executorService;

    /**
     * 监听端口
     */
    private int port;


    public SocketServer(int port) {
        try {
            serverSocket = new ServerSocket(port);
            // 设置等待客户端与其建立连接的时间，如果超过这个时间没有客户端与其建立连接，则会报：Accept timed out。这个属性不建议设置，一直让其等待建立连接就好
            // serverSocket.setSoTimeout(5000);
            executorService = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors() + 1);
            log.info("Socket服务已启动，监听端口={}.", port);
        } catch (Throwable e) {
            log.error("初始化Socket服务器失败", e);
            // 程序正常退出
            System.exit(0);
        }

    }


    @Override
    public void run() {
        log.info("启动Socket接口访问服务");

        while (true) {
            try {
                Socket socket = serverSocket.accept();
                // 设置与客户端建立连接后，等待客户端发送数据的超时时间。若超过改时间，服务器就会报：Read timed out
                socket.setSoTimeout(60000);
                log.info("[{}]：接收到来自[{}]的请求", Thread.currentThread().getName(), socket.getRemoteSocketAddress());
                executorService.execute(new SocketServerHandler(socket));
            } catch (Throwable e) {
                log.error("接收请求异常", e);
            }
        }
    }
}
```

这里我们仿照上面的模式业务服务端新增了hander的处理逻辑，具体见SocketServerHandler.java：

```java
@Slf4j
public class SocketServerHandler implements Runnable{

    private Socket socket;

    public SocketServerHandler(Socket socket) {
        this.socket = socket;
    }


    @Override
    public void run() {
        BufferedWriter writer = null;
        BufferedReader reader = null;

        try {
            writer = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
            reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));

            String reqStr = null;
            while ((reqStr = reader.readLine()) != null){
                log.info("请求的报文是：{}", reqStr);
                // 这个地方写自己的路由逻辑
                String respStr = "200|success|hello";
                writer.write(respStr);
                writer.newLine();
                writer.flush();
                log.info("响应的报文是：{}", respStr);
                // 禁用该socket的输出流
                socket.shutdownOutput();
                break;
            }
        } catch (Throwable e) {
            log.error("服务端接收报文信息异常", e);
        } finally {
            try {
                if (writer != null) {
                    writer.close();
                }

                if (reader != null) {
                    reader.close();
                }
            } catch (Throwable e) {
                log.error("资源关闭异常", e);
            }

        }

    }
}
```

## 客户端

客户端的代码没有大的变化，具体的代码SocketClient.java如下：

```java
@Slf4j
public class SocketClient {

    /**
     * Socket客户端
     */

    private final String host;
    private final int port;
    private final Charset charset;
    private Socket socketClient;

    public SocketClient(String host, int port, Charset charset) {
        log.info("Socket客户端地址：host={}, port={}, charset={}.", host, port, charset);
        this.host = host;
        this.port = port;
        this.charset = charset;
    }


    public String send(String reqStr) {
        log.info("客户端Socket启动");

        BufferedWriter writer = null;
        BufferedReader reader = null;
        StringBuilder respStr = new StringBuilder();

        try {
            socketClient = new Socket(InetAddress.getByName(host), port);
            // 设置发送数据给服务端之后，等待服务端响应报文的时间，如果超过这个时间就会报：Read time out
            socketClient.setSoTimeout(5000);

            // 发送请求报文
            writer = new BufferedWriter(new OutputStreamWriter(socketClient.getOutputStream(), StandardCharsets.UTF_8));
            log.info("客户端发送报文内容：{}", reqStr);
            writer.write(reqStr);
            writer.newLine();
            writer.flush();

            // 接收响应报文
            reader = new BufferedReader(new InputStreamReader(socketClient.getInputStream(), StandardCharsets.UTF_8));
            String str = null;
            while ((str = reader.readLine()) != null) {
                respStr.append(str);
            }
            log.info("客户端接收到响应报文");

        } catch (Throwable e) {
            log.error("发送请求失败", e);
        } finally {
            try {
                socketClient.close();
                if (writer != null) {
                    writer.close();
                }

                if (reqStr != null) {
                    reader.close();
                }
            } catch (Throwable e) {
                log.error("IO资源关闭异常", e);
            }
        }

        return respStr.toString();
    }
}
```

最后我们编写一个测试类，测试下我们上面的客户端和服务端的代码：

```java
@Slf4j
public class Test {

    /**
     * 测试服务端和客户端
     */

    public static void main(String[] args) {
        // 启动服务器
        SocketServer socketServer = new SocketServer(6666);
        socketServer.start();

        // 启动客户端发送数据
        SocketClient socketClient = new SocketClient("localhost", 6666, StandardCharsets.UTF_8);
        String respStr = socketClient.send("hello Beijing");
    }
}
```

输出的日志如下：

```
[2023-11-16 14:12:00.085][main][INFO ] [cn.bravedawn.network.socket.v2.SocketServer:41] - Socket服务已启动，监听端口=6666.
[2023-11-16 14:12:00.103][main][INFO ] [cn.bravedawn.network.socket.v2.SocketClient:33] - Socket客户端地址：host=localhost, port=6666, charset=UTF-8.
[2023-11-16 14:12:00.105][main][INFO ] [cn.bravedawn.network.socket.v2.SocketClient:41] - 客户端Socket启动
[2023-11-16 14:12:00.112][Thread-0][INFO ] [cn.bravedawn.network.socket.v2.SocketServer:54] - 启动Socket接口访问服务
[2023-11-16 14:12:00.123][Thread-0][INFO ] [cn.bravedawn.network.socket.v2.SocketServer:61] - [Thread-0]：接收到来自[/127.0.0.1:58005]的请求
[2023-11-16 14:12:00.123][main][INFO ] [cn.bravedawn.network.socket.v2.SocketClient:54] - 客户端发送报文内容：hello Beijing
[2023-11-16 14:12:00.130][pool-1-thread-1][INFO ] [cn.bravedawn.network.socket.v2.SocketServerHandler:36] - 请求的报文是：hello Beijing
[2023-11-16 14:12:00.132][pool-1-thread-1][INFO ] [cn.bravedawn.network.socket.v2.SocketServerHandler:42] - 响应的报文是：200|success|hello
[2023-11-16 14:12:00.132][main][INFO ] [cn.bravedawn.network.socket.v2.SocketClient:65] - 客户端接收到响应报文，报文内容是：200|success|hello
```

# 参考文章

* [TCP编程](https://www.liaoxuefeng.com/wiki/1252599548343744/1305207629676577)
* (A Guide to Java Sockets)[https://www.baeldung.com/a-guide-to-java-sockets]