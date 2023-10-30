> Java/网络

> 最近在开发乌鲁木齐分行财政项目的时候发现分行与财政系统的联机交互是通过Socket来通信的，之前做的项目都是通过http协议来通信的，再加上自己对Socket编程这块知识点也不是很清楚，所以写一篇博客来梳理一下。

# 什么是Socket

Socket是一个抽象概念，一个应用程序通过一个Socket来建立一个远程连接，而Socket内部通过TCP/IP协议把数据传输到网络。

Socket、TCP和部分IP的功能都是由操作系统提供的，不同的编程语言只是提供了对操作系统调用的简单的封装。例如，Java提供的几个Socket相关的类就封装了操作系统提供的接口。

* ServerSocket
* Socket

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
        writer.write("hello\n");
        writer.flush();

        while (true) {
            String s = reader.readLine();
            if (s.equals("bye")) {
                writer.write("byte\n");
                writer.flush();
                break;
            }
            writer.write("ok: " + s + "\n");
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

值得注意的是，无论是客户端还是服务器端每每一条消息的结束都是用一个换行符来标识的。我们既可以通过`\n`显示标识，也可以使用`writer.newLine()`来实现。

# 线程池武装的Socket客户端和服务端

下面我来针对服务器端和客户端分别给出使用线程池接收和发送数据的逻辑实现。