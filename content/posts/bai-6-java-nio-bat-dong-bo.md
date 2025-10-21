---
title: "Java NIO (Bất đồng bộ)"
date: 2025-10-20T14:29:18+07:00
draft: false
tags: ["java","nio","networking","performance"]
categories: ["Java"]
---
# Mục tiêu bài học
- Hiểu khác biệt giữa IO blocking truyền thống và NIO non-blocking (Selector/Channel/Buffer).  
- Biết cách dùng `Selector`, `ServerSocketChannel`/`SocketChannel`, `DatagramChannel` và `ByteBuffer`.  
- Viết server TCP non-blocking (selector-based) đơn giản và client tương ứng.  
- Nhận biết các pitfalls: flip/clear/compact, busy-loop, interestOps, thread-safety của Selector.  
- Công cụ tối ưu: direct buffers, pooling, multiple selectors / Reactor threads.

---

## 1) Tóm tắt nhanh
- Blocking IO: mỗi kết nối thường được 1 thread — đơn giản nhưng tốn tài nguyên khi có nhiều kết nối.
- NIO (New I/O): dùng Channel (non-blocking) + Selector để multiplex nhiều socket trên vài thread.
- Ba thành phần chính:
  - Channel: `SocketChannel`, `ServerSocketChannel`, `DatagramChannel`.
  - Buffer: `ByteBuffer` (heap hoặc direct).
  - Selector: đăng ký interest (OP_READ, OP_WRITE, OP_ACCEPT, OP_CONNECT).

---

## 2) Nguyên lý hoạt động (very short)
- Mỗi Channel được đặt non-blocking và đăng ký với một Selector.
- Selector.select() trả về các SelectionKey sẵn sàng (read/write/accept).
- Xử lý key: đọc/ghi từ/to ByteBuffer, sau đó điều chỉnh interestOps nếu cần.
- Để ghi không block: giữ dữ liệu chưa gửi trong buffer gắn vào SelectionKey, bật OP_WRITE; khi write xong tắt OP_WRITE.

---

## 3) Ví dụ: Echo server non-blocking (selector-based)

Ví dụ sau minh hoạ server echo đơn giản (single-thread selector). Lưu ý quản lý ByteBuffer gắn vào SelectionKey để giữ dữ liệu còn chưa gửi.

```java
// filepath: c:\Users\dangl\OneDrive\Desktop\Blog\content\posts\bai-6-java-nio-bat-dong-bo.md
// ...existing code...
// NioEchoServer.java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.*;
import java.util.Iterator;
import java.util.Set;

public class NioEchoServer {
    public static void main(String[] args) throws IOException {
        int port = 8000;
        try (Selector selector = Selector.open();
             ServerSocketChannel server = ServerSocketChannel.open()) {

            server.bind(new InetSocketAddress(port));
            server.configureBlocking(false);
            server.register(selector, SelectionKey.OP_ACCEPT);
            System.out.println("[NIO] Listening " + port);

            ByteBuffer readBuffer = ByteBuffer.allocate(4096);

            while (true) {
                selector.select(); // block until events
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> it = keys.iterator();
                while (it.hasNext()) {
                    SelectionKey key = it.next();
                    it.remove();

                    if (!key.isValid()) continue;

                    if (key.isAcceptable()) {
                        ServerSocketChannel srv = (ServerSocketChannel) key.channel();
                        SocketChannel client = srv.accept();
                        client.configureBlocking(false);
                        // attach an output buffer for this client
                        client.register(selector, SelectionKey.OP_READ, ByteBuffer.allocate(8192));
                        System.out.println("[NIO] Accepted " + client.getRemoteAddress());
                    } else if (key.isReadable()) {
                        SocketChannel client = (SocketChannel) key.channel();
                        ByteBuffer outBuf = (ByteBuffer) key.attachment();
                        readBuffer.clear();
                        int read;
                        try {
                            read = client.read(readBuffer);
                        } catch (IOException e) {
                            key.cancel();
                            client.close();
                            System.out.println("[NIO] Read error - closed");
                            continue;
                        }
                        if (read == -1) {
                            key.cancel();
                            client.close();
                            System.out.println("[NIO] Client closed");
                            continue;
                        }
                        readBuffer.flip();
                        // echo: append to outBuf
                        outBuf.put(readBuffer);
                        // prepare to write: enable OP_WRITE
                        key.interestOps(key.interestOps() | SelectionKey.OP_WRITE);
                    } else if (key.isWritable()) {
                        SocketChannel client = (SocketChannel) key.channel();
                        ByteBuffer outBuf = (ByteBuffer) key.attachment();
                        outBuf.flip(); // switch to read mode to write to socket
                        try {
                            int written = client.write(outBuf);
                            // written bytes removed from outBuf by position advance
                            // compact to keep remaining bytes
                            outBuf.compact();
                            if (outBuf.position() == 0) {
                                // nothing left to write: disable OP_WRITE
                                key.interestOps(key.interestOps() & ~SelectionKey.OP_WRITE);
                            }
                        } catch (IOException e) {
                            key.cancel();
                            client.close();
                            System.out.println("[NIO] Write error - closed");
                        }
                    }
                }
            }
        }
    }
}
// ...existing code...
```

Ghi chú: gắn ByteBuffer cho mỗi key giúp quản lý dữ liệu chưa gửi; dùng compact() giữ phần dữ liệu chưa đọc/ghi.

---

## 4) Client non-blocking (connect + write + read)

```java
// NioClient.java
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;

public class NioClient {
    public static void main(String[] args) throws Exception {
        try (SocketChannel ch = SocketChannel.open()) {
            ch.configureBlocking(true); // simpler: blocking client
            ch.connect(new InetSocketAddress("127.0.0.1", 8000));
            ByteBuffer buf = ByteBuffer.wrap("hello nio\n".getBytes());
            ch.write(buf);
            buf.clear();
            int read = ch.read(buf);
            if (read > 0) {
                buf.flip();
                byte[] b = new byte[buf.remaining()];
                buf.get(b);
                System.out.println("Echo: " + new String(b));
            }
        }
    }
}
```

Ghi chú: client có thể non-blocking với Selector tương tự server, nhưng cho testing dùng blocking SocketChannel là đủ.

---

## 5) Các lỗi phổ biến & lưu ý
- Buffer lifecycle: luôn flip() trước read-from-buffer hoặc write-to-channel; clear() để reuse; compact() khi giữ dữ liệu còn lại.
- interestOps và OP_WRITE:
  - Không nên luôn bật OP_WRITE — sẽ gây busy-loop nếu socket luôn writable.
  - Bật OP_WRITE chỉ khi còn dữ liệu để ghi, tắt khi buffer trống.
- Selector.select() vs selectNow():
  - select() block; selectNow() không block (dùng thận trọng để tránh busy-loop).
- Thay đổi interestOps từ thread khác:
  - Nếu thay đổi interestOps từ thread khác, gọi selector.wakeup() để đảm bảo select() phản ứng.
- Attachment của SelectionKey:
  - Dùng attachment để lưu state per-connection (buffers, parsing state, timestamps).
- Large messages / fragmentation:
  - NIO trả raw bytes; cần framing (newline, length-prefix) giống TCP blocking.
- Thread model:
  - Single-thread selector ok cho moderate connections. Với nhiều CPU/cores chia theo Reactor threads (acceptor thread + multiple worker selectors).
- Direct vs heap ByteBuffer:
  - Direct (allocateDirect) giúp giảm copy sang native nhưng tốn chi phí allocate; pool direct buffers cho throughput cao.

---

## 6) Performance tips
- Reuse ByteBuffer (pool) để giảm GC pressure.
- Group IO threads: 1 acceptor + N worker selectors (reactor pattern).
- Minimize allocations trong hot path; parse in-place in ByteBuffer when possible.
- Use SO_REUSEADDR, set socket options (SO_RCVBUF/SO_SNDBUF) phù hợp:
  - ((SocketChannel)key.channel()).socket().setReceiveBufferSize(...)
- For Linux: epoll (JVM uses epoll via SelectorProvider) scales better than select() emulation.
- Consider frameworks (Netty) for production: provide pooling, zero-copy optimizations, robust pipeline.

---

## 7) Kiểm thử & debug
- Local testing: telnet/nc connect server để gửi dữ liệu thô.
- Log minimal state (accept/read/write events, errors). Tránh log mỗi byte in hot path.
- Use tools: jstack, jmap, VisualVM to inspect thread/GC. Use perf or bpftrace for syscall-level profiling.
- Simulate load: wrk, tsung, or custom client with many connections.

---

## 8) Bài tập
- Viết echo server non-blocking hỗ trợ nhiều client và cho phép broadcast message tới tất cả kết nối.
- Thêm framing: chuyển server sang line-based (đọc tới '\n') và xử lý từng message.
- Triển khai reactor: 1 acceptor thread + 2 worker selector threads. So sánh throughput/CPU với single-thread selector.
- Thêm SSL/TLS: dùng SSLEngine để tích hợp TLS trên non-blocking Channel (khó hơn, đọc thêm SSLEngine docs).
- Migrate sang Netty: viết cùng tính năng bằng Netty và so sánh code complexity & performance.

---
