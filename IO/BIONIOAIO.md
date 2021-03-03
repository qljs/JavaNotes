

## 一 BIO

BIO（同步阻塞）模型图：

![](..\images\IO\BIO.png) 



BIO的例子：

```java
/**
 * 客户端
 */
public class BIOClient {

    public static void main(String[] args) throws Exception {
        Socket socket = new Socket("127.0.0.1",8088);
        socket.getOutputStream().write("客户端测试发送数据".getBytes());
        byte[] bytes = new byte[1024];
        // 等待服务端返回信息时，会被阻塞在这里
        int read = socket.getInputStream().read(bytes);
        if(-1 != read){
            System.out.println(new String(bytes));
        }
        socket.close();
    }
}

/**
 * 服务端
 */
public class BIOServer {

    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(8088);
        System.out.println("等待客户端建立连接");
        // 客户端没有连接进来时，这里会阻塞
        Socket accept = serverSocket.accept();
        System.out.println("连接已建立，开始接收客户端数据");
        byte[] bytes = new byte[1024];
        // 客户端没有数据发送过来时，会阻塞在这里
        int read = accept.getInputStream().read(bytes);
        if(-1 != read) {
            System.out.println(new String(bytes));
        }

        accept.getOutputStream().write("服务端返回数据".getBytes());
        accept.close();
        serverSocket.close();
    }
}
```

对于BIO而言，在接收请求或者等待数据传输时都是阻塞的，在高并发场景下性能较差，虽然可以采用线程池来异步处理，并且避免线程耗尽问题。



## 二 NIO

NIO（同步非阻塞）实现模式为一个线程可以处理多个请求（连接），客户端发送的请求都会注册到多路复用器上，然后多路复用器轮询处理请求。

I/O多路复用底层一般用Linux API来实现（select、poll、epoll）：

|          | select                           | poll                             | epoll                                                        |
| -------- | -------------------------------- | -------------------------------- | ------------------------------------------------------------ |
| 操作方式 | 遍历                             | 遍历                             | 回调                                                         |
| 底层实现 | 数组                             | 数组                             | 哈希表                                                       |
| IO效率   | 每次需要线性遍历，时间复杂度O(n) | 每次需要线性遍历，时间复杂度O(n) | 事件通知方式，每次由IO事件时，通过调用回调函数实现，时间复杂度O(1) |
| 最大连接 | 有上限                           | 无上限                           | 无上限                                                       |

NIO 的模型图：

![](..\images\IO\NIO.png)



**NIO相对于BIO非阻塞的体现就在，BIO的后端线程需要阻塞等待客户端写数据(比如read方法)，如果客户端不写数据线程就要阻塞， NIO把等待客户端操作的事情交给了selector，selector 负责轮询所有已注册的客户端，发现有事件发生了才转交给后端线程处 理，后端线程不需要做任何阻塞等待，直接处理客户端事件的数据即可，处理完马上结束，或返回线程池供其他客户端事件继续使用。还有就是 channel 的读写是非阻塞的。**



### 1. NIO中核心组件

- **Buffer（缓冲区）**：BIO 是面向流的，而 NIO 是面向缓冲区的，在 NIO 中数据的读写都是操作缓冲区。
- **Channel（通道）**：NIO 通过通道来读写，通道是双向的。
- **Selector（选择器）**：通过将通道注册到选择器上，来监听通道上发生的事件。



## 2. NIO的例子

```java
/**
 * 客户端
 */
public class NIOClient {
    public static void main(String[] args) throws Exception {
        SocketChannel socketChannel = SocketChannel.open();
        socketChannel.configureBlocking(false);
        Selector selector = Selector.open();
        socketChannel.connect(new InetSocketAddress("127.0.0.1", 8088));
        socketChannel.register(selector, SelectionKey.OP_CONNECT);

        while (true) {
            selector.select();
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove();
                if(key.isConnectable()) {
                    SocketChannel channel = (SocketChannel) key.channel();
                    if(channel.isConnectionPending()){
                        channel.finishConnect();
                    }
                    channel.configureBlocking(false);
                    ByteBuffer wrap = ByteBuffer.wrap("客户端消息".getBytes());

                    channel.write(wrap);

                    channel.register(selector, SelectionKey.OP_READ);
                } else if(key.isReadable()){
                    SocketChannel channel = (SocketChannel) key.channel();
                    ByteBuffer wrap = ByteBuffer.allocate(1024);
                    int read = channel.read(wrap);
                    if(-1 != read){
                        System.out.println("服务端返回消息：" + new String(wrap.array(),0 ,read));
                    }
                }
            }
        }
    }
}

/**
 * 服务端
 */
public class NIOServer {
    public static void main(String[] args) throws Exception {
        Selector selector = Selector.open();
        ServerSocketChannel serverSocket = ServerSocketChannel.open();
        serverSocket.configureBlocking(false);
        serverSocket.socket().bind(new InetSocketAddress(8088));
        serverSocket.register(selector, SelectionKey.OP_ACCEPT);

        while (true) {
            System.out.println("等待事件发生");
            selector.select();
            Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
            // 获取selector上的事件，进行循环
            while (iterator.hasNext()){
                SelectionKey key = iterator.next();
                iterator.remove();
                // 连接事件
                if(key.isAcceptable()) {
                    System.out.println("客户端连接进来了");
                    SocketChannel socketChannel = serverSocket.accept();
                    socketChannel.configureBlocking(false);
                    // 添加读取事件，等到客户端传送数据过来时触发
                    socketChannel.register(key.selector(), SelectionKey.OP_READ);
                    // 读取事件
                } else if(key.isReadable()) {
                    System.out.println("读事件发生");
                    SocketChannel socketChannel = (SocketChannel) key.channel();
                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                    int read = socketChannel.read(buffer);
                    if(read != -1){
                        System.out.println("从客户端接收的数据：" + new String(buffer.array(),0, read));
                    }
                    ByteBuffer wrap = ByteBuffer.wrap("服务端返回数据".getBytes());
                    socketChannel.write(wrap);
                    key.interestOps(SelectionKey.OP_READ|SelectionKey.OP_WRITE);
                } else if(key.isWritable()){
                    System.out.println("写事件发生");
                    SocketChannel socketChannel = (SocketChannel) key.channel();
                    key.interestOps(SelectionKey.OP_READ);
                }
            }
        }
    }
}
```





## 三 AIO

AIO 即异步非阻塞的 IO 模型，是基于事件和回调机制实现的。

```java
/**
 * 客户端
 */
public class AIOClient {
    public static void main(String[] args) throws Exception {
        AsynchronousSocketChannel socketChannel = AsynchronousSocketChannel.open();
        socketChannel.connect(new InetSocketAddress("127.0.0.1", 8088));

        socketChannel.write(ByteBuffer.wrap("发送数据".getBytes()));
        ByteBuffer allocate = ByteBuffer.allocate(1024);
        Integer integer = socketChannel.read(allocate).get();
        if(-1 != integer){
            System.out.println("客户端收到消息:" + new String(allocate.array(), 0, integer));
        }
    }
}

/**
 * 服务端
 */
public class AIOServer {
    public static void main(String[] args) throws Exception {
        AsynchronousServerSocketChannel serverSocketChannel = AsynchronousServerSocketChannel.open().bind(new InetSocketAddress(8088));
        serverSocketChannel.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>() {
            @Override
            public void completed(AsynchronousSocketChannel socketChannel, Object attachment) {
                // 接收客户端连接
                serverSocketChannel.accept(attachment,this);
                System.out.println("客户端已建立连接,开始读取数据");
                ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
                socketChannel.read(byteBuffer, byteBuffer, new CompletionHandler<Integer, ByteBuffer>() {
                    @Override
                    public void completed(Integer result, ByteBuffer buffer) {
                        buffer.flip();
                        System.out.println(new String(buffer.array(),0, result));
                        socketChannel.write(ByteBuffer.wrap("返回信息".getBytes()));
                    }

                    @Override
                    public void failed(Throwable exc, ByteBuffer attachment) {
                        exc.printStackTrace();
                    }
                });
            }

            @Override
            public void failed(Throwable exc, Object attachment) {
                exc.printStackTrace();
            }
        });

        Thread.sleep(Integer.MAX_VALUE);
    }
}
```

