# 02 - Java IO 与 NIO

## 目录

1. [IO 模型概述](#一io-模型概述)
2. [BIO 模型](#二bio-模型)
3. [NIO 核心三件套](#三nio-核心三件套)
4. [NIO 工作流程](#四nio-工作流程)
5. [AIO 异步 IO 模型](#五aio-异步-io-模型)
6. [BIO vs NIO vs AIO 对比](#六bio-vs-nio-vs-aio-对比)
7. [零拷贝原理](#七零拷贝原理)
8. [Java 中的零拷贝实现](#八java-中的零拷贝实现)
9. [代码示例](#九代码示例)
10. [大厂常见面试题](#十大厂常见面试题)

---

## 一、IO 模型概述

### 1.1 同步/异步 与 阻塞/非阻塞

这是两个**独立的维度**，常被混淆：

| 维度 | 含义 | 类比 |
|------|------|------|
| **同步** | 调用方主动等待结果 | 你站在餐厅柜台前等出餐 |
| **异步** | 调用方不等，完成后由被调用方通知 | 你点完餐回到座位，出餐后服务员送来 |
| **阻塞** | 调用方等待期间不能做其他事 | 你站在柜台发呆等着 |
| **非阻塞** | 调用方等待期间可以做其他事 | 你站在柜台前玩手机，时不时看一眼 |

### 1.2 五种 IO 模型（Unix）

```
┌─────────────────────────────────────────────────────────┐
│  用户态                     │  内核态                    │
│                             │                            │
│  ① 阻塞IO (BIO)            │  数据准备 → 数据拷贝       │
│     调用 read ──阻塞等待──────────────────── 返回数据    │
│                             │                            │
│  ② 非阻塞IO                │                            │
│     调用 read → 返回EAGAIN  │  数据未就绪                │
│     调用 read → 返回EAGAIN  │  数据未就绪                │
│     调用 read ──阻塞──────────── 数据拷贝 → 返回数据     │
│                             │                            │
│  ③ IO多路复用 (NIO)         │                            │
│     select/poll/epoll ─阻塞── 监听多个fd                 │
│     某个fd就绪 → read ─────────── 数据拷贝 → 返回数据    │
│                             │                            │
│  ④ 信号驱动IO               │                            │
│     注册信号 → 立即返回     │  数据就绪 → 发送信号       │
│     收到信号 → read ────────────── 数据拷贝 → 返回数据   │
│                             │                            │
│  ⑤ 异步IO (AIO)             │                            │
│     aio_read → 立即返回     │  数据准备 + 拷贝           │
│     内核完成后回调通知       │  全部由内核完成            │
└─────────────────────────────────────────────────────────┘
```

**Java 对应关系：**
- BIO → `java.io` + `java.net.Socket`
- NIO → `java.nio`（IO 多路复用）
- AIO → `java.nio.channels.AsynchronousChannel`（JDK 7）

---

## 二、BIO 模型

### 2.1 传统 Socket 编程

BIO（Blocking IO）是最传统的 IO 模型，**一个连接对应一个线程**。

```
BIO 线程模型：

  客户端1 ──────► 线程1（阻塞等待读写）
  客户端2 ──────► 线程2（阻塞等待读写）
  客户端3 ──────► 线程3（阻塞等待读写）
  ...
  客户端N ──────► 线程N（阻塞等待读写）

  问题：每个连接一个线程，连接数多时线程数爆炸
```

### 2.2 BIO 服务端代码

```java
import java.io.*;
import java.net.*;

public class BIOServer {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("BIO 服务器启动，端口: 8080");
        
        while (true) {
            // accept() 阻塞，等待新连接
            Socket socket = serverSocket.accept();
            System.out.println("新连接: " + socket.getRemoteSocketAddress());
            
            // 每个连接分配一个线程
            new Thread(() -> handleClient(socket)).start();
        }
    }
    
    private static void handleClient(Socket socket) {
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(socket.getInputStream()));
             PrintWriter writer = new PrintWriter(socket.getOutputStream(), true)) {
            
            String line;
            while ((line = reader.readLine()) != null) { // readLine() 阻塞
                System.out.println("收到: " + line);
                writer.println("Echo: " + line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

### 2.3 BIO 的缺点

| 问题 | 说明 |
|------|------|
| 线程开销大 | 每个连接一个线程，1000 连接就需要 1000 线程 |
| 上下文切换 | 大量线程导致频繁的线程上下文切换，CPU 利用率低 |
| 内存浪费 | 每个线程占用栈空间（默认 512KB-1MB），大量线程消耗大量内存 |
| 阻塞等待 | 大部分时间线程在 `read()` 上阻塞，浪费线程资源 |

---

## 三、NIO 核心三件套

Java NIO（New IO / Non-blocking IO）引入于 JDK 1.4，核心组件：**Channel、Buffer、Selector**。

### 3.1 Channel（通道）

Channel 是双向的数据传输通道（BIO 的 Stream 是单向的），必须通过 Buffer 来读写数据。

```
BIO Stream（单向）：
  InputStream  ──读──►  程序
  OutputStream ◄──写──  程序

NIO Channel（双向）：
  Channel ◄──读/写──► Buffer ◄──读/写──► 程序
```

常用 Channel 实现：

| Channel | 说明 |
|---------|------|
| `FileChannel` | 文件读写（不支持非阻塞模式） |
| `SocketChannel` | TCP 客户端 |
| `ServerSocketChannel` | TCP 服务端 |
| `DatagramChannel` | UDP |

### 3.2 Buffer（缓冲区）

Buffer 是一块内存区域，NIO 读写数据都要通过 Buffer。核心属性：

```
Buffer 内部结构：

  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
  │ H │ e │ l │ l │ o │   │   │   │   │   │
  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7   8   9
  ↑                   ↑                   ↑
  |               position            capacity
  mark              (读写位置)         (总容量)
                      ↑
                    limit
                  (可操作上限)
```

```java
// 核心属性
// capacity：总容量，创建后不变
// position：当前读写位置
// limit：可操作上限
// 约束：0 <= mark <= position <= limit <= capacity

// 基本使用
ByteBuffer buffer = ByteBuffer.allocate(1024);

// 写模式 → 读模式
buffer.put("Hello".getBytes());  // 写入数据
buffer.flip();                    // position→0, limit→原position

// 读取数据
while (buffer.hasRemaining()) {
    byte b = buffer.get();
}

// 清空/压缩
buffer.clear();    // position→0, limit→capacity（数据未清除）
buffer.compact();  // 未读数据移到头部，position→未读数据后
```

**Buffer 类型：** `ByteBuffer`（最常用）、`CharBuffer`、`IntBuffer`、`LongBuffer`、`DoubleBuffer`、`FloatBuffer`、`ShortBuffer`。

### 3.3 Selector（选择器）

Selector 是 NIO 实现**IO 多路复用**的核心，一个 Selector 可以监听多个 Channel 的事件。

```
Selector 多路复用：

               ┌──────────┐
  Channel1 ───►│          │
  Channel2 ───►│ Selector │──► 单线程轮询处理就绪事件
  Channel3 ───►│          │
  Channel4 ───►│          │
               └──────────┘

  对比 BIO：4 个连接需要 4 个线程
  NIO：     4 个连接只需 1 个线程 + 1 个 Selector
```

**可监听的事件类型：**

| SelectionKey 常量 | 含义 | 对应 Channel |
|-------------------|------|-------------|
| `OP_ACCEPT` | 有新连接就绪 | ServerSocketChannel |
| `OP_CONNECT` | 连接建立完成 | SocketChannel |
| `OP_READ` | 有数据可读 | SocketChannel |
| `OP_WRITE` | 可以写数据 | SocketChannel |

---

## 四、NIO 工作流程

### 4.1 Selector 多路复用原理

```
NIO 服务端完整工作流程：

① 创建 ServerSocketChannel，绑定端口
② 设置为非阻塞模式
③ 创建 Selector，将 ServerSocketChannel 注册到 Selector（监听 ACCEPT）
④ 循环调用 selector.select()（阻塞等待就绪事件）
    │
    ▼
⑤ 获取就绪的 SelectionKey 集合
    │
    ├── isAcceptable → 接受新连接
    │    └── 将新的 SocketChannel 注册到 Selector（监听 READ）
    │
    ├── isReadable → 读取数据
    │    └── 从 Channel 读数据到 Buffer，处理业务
    │
    └── isWritable → 写入数据
         └── 从 Buffer 写数据到 Channel
```

### 4.2 底层实现：epoll（Linux）

```
Java NIO Selector 在不同操作系统上的实现：

  Linux   →  epoll     （事件驱动，O(1) 就绪通知，高性能）
  macOS   →  kqueue    （类似 epoll）
  Windows →  IOCP      （异步IO）

epoll 工作原理：
  epoll_create  → 创建 epoll 实例（红黑树 + 就绪队列）
  epoll_ctl     → 注册/修改/删除监听的 fd
  epoll_wait    → 阻塞等待就绪事件
  
  ┌──────────────────┐      ┌──────────────┐
  │  红黑树           │      │  就绪队列     │
  │  (所有监听的fd)   │ ───► │  (有事件的fd) │
  └──────────────────┘      └──────────────┘
       注册/删除                epoll_wait 返回
```

---

## 五、AIO 异步 IO 模型

AIO（Asynchronous IO）是 JDK 7 引入的真正的异步 IO，由操作系统内核完成数据拷贝后通知应用程序。

```java
import java.nio.channels.*;
import java.nio.ByteBuffer;
import java.net.InetSocketAddress;

public class AIOServer {
    public static void main(String[] args) throws Exception {
        AsynchronousServerSocketChannel server =
            AsynchronousServerSocketChannel.open()
                .bind(new InetSocketAddress(8080));
        
        System.out.println("AIO 服务器启动，端口: 8080");
        
        // 异步接受连接
        server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
            @Override
            public void completed(AsynchronousSocketChannel client, Void attachment) {
                server.accept(null, this); // 继续接受下一个连接
                
                ByteBuffer buffer = ByteBuffer.allocate(1024);
                // 异步读取
                client.read(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>() {
                    @Override
                    public void completed(Integer result, ByteBuffer buf) {
                        buf.flip();
                        byte[] data = new byte[buf.remaining()];
                        buf.get(data);
                        System.out.println("收到: " + new String(data));
                    }
                    
                    @Override
                    public void failed(Throwable exc, ByteBuffer buf) {
                        exc.printStackTrace();
                    }
                });
            }
            
            @Override
            public void failed(Throwable exc, Void attachment) {
                exc.printStackTrace();
            }
        });
        
        Thread.currentThread().join(); // 防止主线程退出
    }
}
```

**注意：** Linux 下 AIO 实际上是用 epoll 模拟的，真正的内核 AIO（io_uring）支持有限，因此 Netty 放弃了 AIO，选择基于 NIO（epoll）的实现。

---

## 六、BIO vs NIO vs AIO 对比

| 维度 | BIO | NIO | AIO |
|------|-----|-----|-----|
| IO 模型 | 同步阻塞 | 同步非阻塞（IO 多路复用） | 异步非阻塞 |
| 线程模型 | 一连接一线程 | 一个线程管理多个连接 | 无需额外线程 |
| 编程复杂度 | 简单 | 较复杂 | 复杂（回调模式） |
| 吞吐量 | 低 | 高 | 高 |
| 适用场景 | 连接数少、每连接数据量大 | 连接数多、每连接数据量少（如聊天服务器） | 连接数多、每连接数据量大（理论上） |
| JDK 版本 | JDK 1.0 | JDK 1.4 | JDK 7 |
| 框架 | 传统 Socket | Netty、Mina | — |

```
并发连接数 vs 线程数：

  BIO：
  连接数: 10000 → 线程数: 10000 → 系统崩溃

  NIO：
  连接数: 10000 → 线程数: 1~几个 → 稳定运行
                    (Selector 多路复用)
```

---

## 七、零拷贝原理

### 7.1 传统 IO 数据拷贝流程

传统的文件传输（如读文件发到 Socket），涉及 **4 次数据拷贝 + 4 次上下文切换**：

```
传统IO：读文件 → 发送到网络

  用户态                        内核态
    │                             │
    │  ① read() 系统调用           │
    │  ─────────────────────────► │
    │                             │ DMA拷贝: 磁盘 → 内核缓冲区
    │  ② 拷贝: 内核缓冲区 → 用户缓冲区
    │  ◄───────────────────────── │
    │                             │
    │  ③ write() 系统调用          │
    │  ─────────────────────────► │
    │  拷贝: 用户缓冲区 → Socket缓冲区
    │                             │ ④ DMA拷贝: Socket缓冲区 → 网卡
    │  ◄───────────────────────── │
    │                             │

  4次拷贝：磁盘→内核→用户→Socket→网卡
  4次上下文切换：用户→内核→用户→内核→用户
```

### 7.2 mmap 优化

`mmap`（Memory Mapped File）将内核缓冲区映射到用户空间，减少一次拷贝：

```
mmap + write：

  用户态                        内核态
    │                             │
    │  ① mmap() 映射              │
    │  ─────────────────────────► │
    │                             │ DMA拷贝: 磁盘 → 内核缓冲区
    │  (用户空间直接映射内核缓冲区，无需拷贝)
    │  ◄───────────────────────── │
    │                             │
    │  ② write() 系统调用          │
    │  ─────────────────────────► │
    │  拷贝: 内核缓冲区 → Socket缓冲区
    │                             │ ③ DMA拷贝: Socket缓冲区 → 网卡
    │  ◄───────────────────────── │

  3次拷贝：磁盘→内核→Socket→网卡
  4次上下文切换（未减少）
```

### 7.3 sendfile 优化

`sendfile` 系统调用直接在内核态完成数据传输：

```
sendfile（Linux 2.4+ 带 scatter/gather DMA）：

  用户态                        内核态
    │                             │
    │  ① sendfile() 系统调用       │
    │  ─────────────────────────► │
    │                             │ DMA拷贝: 磁盘 → 内核缓冲区
    │                             │ (仅传递fd和offset到Socket缓冲区)
    │                             │ DMA拷贝: 内核缓冲区 → 网卡
    │  ◄───────────────────────── │

  2次拷贝（仅DMA，CPU不参与数据拷贝）
  2次上下文切换
```

### 7.4 三种方式对比

| 方式 | 数据拷贝次数 | 上下文切换 | CPU 参与拷贝 |
|------|-------------|-----------|-------------|
| 传统 IO | 4 次 | 4 次 | 2 次 |
| mmap + write | 3 次 | 4 次 | 1 次 |
| sendfile | 2 次 | 2 次 | 0 次 |

---

## 八、Java 中的零拷贝实现

### 8.1 MappedByteBuffer（mmap）

```java
import java.io.*;
import java.nio.*;
import java.nio.channels.*;

// 使用 mmap 读取大文件
public class MmapDemo {
    public static void main(String[] args) throws Exception {
        RandomAccessFile file = new RandomAccessFile("large-file.dat", "r");
        FileChannel channel = file.getChannel();
        
        // 将文件映射到内存（不会全部加载，按需缺页加载）
        MappedByteBuffer buffer = channel.map(
            FileChannel.MapMode.READ_ONLY, 0, channel.size());
        
        // 直接操作映射内存，无需 read() 系统调用
        while (buffer.hasRemaining()) {
            byte b = buffer.get();
            // 处理数据...
        }
        
        channel.close();
        file.close();
    }
}
```

**适用场景：** 大文件随机读写、数据库索引文件（如 RocketMQ 的 CommitLog）。

### 8.2 FileChannel.transferTo（sendfile）

```java
import java.io.*;
import java.net.*;
import java.nio.channels.*;

// 使用 sendfile 零拷贝发送文件
public class ZeroCopyServer {
    public static void main(String[] args) throws Exception {
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(8080));
        
        SocketChannel clientChannel = serverChannel.accept();
        
        FileChannel fileChannel = new FileInputStream("data.bin").getChannel();
        
        // transferTo 底层调用 sendfile 系统调用
        long transferred = fileChannel.transferTo(
            0, fileChannel.size(), clientChannel);
        
        System.out.println("传输字节数: " + transferred);
        
        fileChannel.close();
        clientChannel.close();
        serverChannel.close();
    }
}
```

**适用场景：** 文件服务器、静态资源传输、Kafka 消息发送。

### 8.3 框架中的零拷贝应用

| 框架 | 零拷贝方式 | 说明 |
|------|-----------|------|
| Kafka | `FileChannel.transferTo` | Consumer 拉取消息时，直接从磁盘到网卡 |
| RocketMQ | `MappedByteBuffer` | CommitLog 通过 mmap 映射到内存 |
| Netty | CompositeByteBuf | 逻辑上合并多个 Buffer，避免内存拷贝 |
| Netty | `FileRegion` | 封装 `FileChannel.transferTo` |

---

## 九、代码示例

### 9.1 NIO 服务端完整示例

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.*;
import java.nio.channels.*;
import java.util.*;

public class NIOServer {
    private Selector selector;
    private ServerSocketChannel serverChannel;
    private static final int PORT = 8080;

    public void start() throws IOException {
        selector = Selector.open();
        serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(PORT));
        serverChannel.configureBlocking(false); // 非阻塞
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        
        System.out.println("NIO 服务器启动，端口: " + PORT);

        while (true) {
            selector.select(); // 阻塞等待就绪事件
            Set<SelectionKey> keys = selector.selectedKeys();
            Iterator<SelectionKey> iterator = keys.iterator();

            while (iterator.hasNext()) {
                SelectionKey key = iterator.next();
                iterator.remove(); // 必须手动移除，否则重复处理

                if (key.isAcceptable()) {
                    handleAccept(key);
                } else if (key.isReadable()) {
                    handleRead(key);
                }
            }
        }
    }

    private void handleAccept(SelectionKey key) throws IOException {
        ServerSocketChannel server = (ServerSocketChannel) key.channel();
        SocketChannel client = server.accept();
        client.configureBlocking(false);
        client.register(selector, SelectionKey.OP_READ);
        System.out.println("新连接: " + client.getRemoteAddress());
    }

    private void handleRead(SelectionKey key) throws IOException {
        SocketChannel client = (SocketChannel) key.channel();
        ByteBuffer buffer = ByteBuffer.allocate(1024);

        int bytesRead = client.read(buffer);
        if (bytesRead == -1) {
            key.cancel();
            client.close();
            System.out.println("连接关闭");
            return;
        }

        buffer.flip();
        byte[] data = new byte[buffer.remaining()];
        buffer.get(data);
        String message = new String(data);
        System.out.println("收到: " + message);

        // 回写
        ByteBuffer response = ByteBuffer.wrap(("Echo: " + message).getBytes());
        client.write(response);
    }

    public static void main(String[] args) throws IOException {
        new NIOServer().start();
    }
}
```

### 9.2 NIO 客户端示例

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.*;
import java.nio.channels.*;

public class NIOClient {
    public static void main(String[] args) throws IOException {
        SocketChannel channel = SocketChannel.open();
        channel.configureBlocking(false);
        channel.connect(new InetSocketAddress("localhost", 8080));

        // 等待连接完成
        while (!channel.finishConnect()) {
            // 非阻塞，可以做其他事
        }

        // 发送数据
        String message = "Hello NIO Server";
        ByteBuffer buffer = ByteBuffer.wrap(message.getBytes());
        channel.write(buffer);

        // 读取响应
        ByteBuffer readBuffer = ByteBuffer.allocate(1024);
        channel.read(readBuffer);
        readBuffer.flip();
        byte[] data = new byte[readBuffer.remaining()];
        readBuffer.get(data);
        System.out.println("收到响应: " + new String(data));

        channel.close();
    }
}
```

---

## 十、大厂常见面试题

### Q1：NIO 和 BIO 的核心区别是什么？

**答：**

| 维度 | BIO | NIO |
|------|-----|-----|
| 面向对象 | 面向流（Stream） | 面向缓冲区（Buffer） |
| 阻塞性 | 阻塞（`read`/`write` 阻塞当前线程） | 非阻塞（通过 Selector 多路复用） |
| 数据流向 | 单向（InputStream/OutputStream） | 双向（Channel） |
| 线程模型 | 一连接一线程 | 一个线程管理多个连接 |

核心区别在于 NIO 的 **Selector 多路复用**：一个线程通过 Selector 监听多个 Channel，只有在 Channel 有事件就绪时才处理，避免了大量线程阻塞等待。

---

### Q2：select、poll、epoll 有什么区别？

**答：**

| 维度 | select | poll | epoll |
|------|--------|------|-------|
| 数据结构 | bitmap（fd_set） | 链表（pollfd） | 红黑树 + 就绪队列 |
| fd 上限 | 1024（FD_SETSIZE） | 无限制 | 无限制 |
| 遍历方式 | 线性遍历所有 fd | 线性遍历所有 fd | 只遍历就绪的 fd |
| 时间复杂度 | O(n) | O(n) | O(1)（就绪通知） |
| fd 拷贝 | 每次调用都要从用户态拷贝到内核 | 同 select | 注册时拷贝一次（epoll_ctl） |
| 触发方式 | 水平触发 | 水平触发 | 支持水平触发 + 边缘触发 |

Java NIO 在 Linux 上使用 **epoll**，这也是 Netty 高性能的底层基础。

---

### Q3：什么是零拷贝？Java 中如何实现？

**答：**

零拷贝（Zero-copy）并非真正的零次拷贝，而是减少 CPU 参与的数据拷贝次数和上下文切换次数。

Java 提供两种零拷贝方式：
1. **`MappedByteBuffer`**（mmap）：通过 `FileChannel.map()` 将文件映射到内存，用户空间直接读写内核缓冲区。
2. **`FileChannel.transferTo()`**（sendfile）：底层调用 `sendfile` 系统调用，数据在内核态直接从文件传输到 Socket。

场景选择：
- 大文件随机读写 → `MappedByteBuffer`（如 RocketMQ）
- 文件网络传输 → `FileChannel.transferTo()`（如 Kafka）

---

### Q4：为什么 Netty 使用 NIO 而不是 AIO？

**答：**

1. **Linux AIO 不成熟**：Linux 的 AIO 实际底层用 epoll 模拟的，并非真正的内核异步 IO，性能没有优势。
2. **Netty 的 NIO 已经足够高效**：Netty 自己实现了 EventLoop 线程模型和内存池，性能极高。
3. **编程模型**：AIO 的回调模式代码复杂，容易出现回调地狱。Netty 基于 NIO 的 Reactor 模型更优雅。
4. **跨平台一致性**：NIO 在各平台表现一致，AIO 在不同操作系统表现差异大。

---

### Q5：DirectByteBuffer 和 HeapByteBuffer 有什么区别？

**答：**

| 维度 | HeapByteBuffer | DirectByteBuffer |
|------|---------------|-----------------|
| 分配位置 | JVM 堆内存 | 堆外内存（直接内存） |
| 创建方式 | `ByteBuffer.allocate()` | `ByteBuffer.allocateDirect()` |
| GC 影响 | 受 GC 管理 | 不受 GC 直接管理 |
| 分配速度 | 快 | 慢（系统调用） |
| IO 性能 | 慢（需先拷贝到直接内存） | 快（避免一次拷贝） |
| 适用场景 | 小数据、频繁分配 | 大数据、长期使用、IO 密集 |

Netty 默认使用 DirectByteBuffer + 池化（PooledByteBufAllocator），兼顾分配速度和 IO 性能。

---

*参考资料：《UNIX网络编程》、《Netty权威指南》、Linux man pages（epoll/sendfile/mmap）*
