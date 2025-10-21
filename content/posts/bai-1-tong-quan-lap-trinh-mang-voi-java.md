---
title: "Tổng quan lập trình mạng với JAVA"
date: 2025-10-20T14:29:08+07:00
draft: false
tags: []
categories: []
---
# Mục tiêu bài học
- Hiểu **client/server**, **IP/port**, **socket**, **TCP vs UDP**, **HTTP**.
- Biết các API mạng trong Java: `java.net`, `HttpClient`, NIO.
- Chạy được ví dụ **Echo TCP** và **UDP** cơ bản.

---

## 1) Tổng quan nhanh
- Ứng dụng mạng = **Client** ↔ **Server** trao đổi qua IP.  
- **Địa chỉ**: `IP` (điểm đến) + `Port` (cổng tiến trình).  
- **Socket**: đầu mối kết nối giữa hai tiến trình.  
- **TCP**: tin cậy, có thứ tự → HTTP, tải file.  
- **UDP**: nhanh, không đảm bảo → game, streaming, DNS.

---

## 2) Mô hình TCP/IP (rút gọn cho dev)

| Tầng       | Ví dụ                 | Lập trình trong Java                         |
|------------|-----------------------|----------------------------------------------|
| Ứng dụng   | HTTP, WebSocket, MQTT| `HttpClient`, web frameworks                 |
| Giao vận   | **TCP**, **UDP**      | `Socket/ServerSocket`, `DatagramSocket`      |
| Internet   | IP                    | Hệ điều hành định tuyến                      |
| Liên kết   | Ethernet, Wi-Fi       | Thường không đụng tới                        |

---

## 3) Bộ công cụ Java
- `java.net`: `ServerSocket`, `Socket`, `DatagramSocket`, `DatagramPacket`, `InetAddress`, `URL`…  
- `java.net.http.HttpClient` (Java 11+): HTTP/1.1, HTTP/2, sync/async.  
- NIO (`java.nio.channels`): `ServerSocketChannel`, `SocketChannel`, `Selector` (hiệu năng cao).  
- Thư viện/Framework: **Spring**, **Netty**, **gRPC**, **WebSocket**.

---

## 4) Thực hành: TCP Echo (blocking I/O)

**Server**
```java
// TcpEchoServer.java
import java.io.*;
import java.net.*;

public class TcpEchoServer {
    public static void main(String[] args) throws IOException {
        int port = 5555;
        try (ServerSocket server = new ServerSocket(port)) {
            System.out.println("[Server] Listening " + port);
            while (true) {
                Socket client = server.accept();
                new Thread(() -> handle(client)).start();
            }
        }
    }
    static void handle(Socket client) {
        String remote = client.getInetAddress() + ":" + client.getPort();
        System.out.println("[Server] Connected: " + remote);
        try (BufferedReader in = new BufferedReader(new InputStreamReader(client.getInputStream()));
             BufferedWriter out = new BufferedWriter(new OutputStreamWriter(client.getOutputStream()))) {
            String line;
            while ((line = in.readLine()) != null) {
                out.write("echo:" + line + "\n");
                out.flush();
            }
        } catch (IOException e) {
            System.out.println("[Server] Error " + remote + " : " + e.getMessage());
        }
    }
}
---
```
## 5) Thực hành: UDP gửi/nhận

**Receiver**
```java
// UdpReceiver.java
import java.net.*;

public class UdpReceiver {
    public static void main(String[] args) throws Exception {
        int port = 5556;
        byte[] buf = new byte[1024];
        try (DatagramSocket socket = new DatagramSocket(port)) {
            System.out.println("[UDP-Recv] Listening " + port);
            while (true) {
                DatagramPacket p = new DatagramPacket(buf, buf.length);
                socket.receive(p);
                String msg = new String(p.getData(), 0, p.getLength());
                System.out.printf("[UDP-Recv] %s:%d -> %s%n",
                        p.getAddress(), p.getPort(), msg);
            }
        }
    }
}
---
```
## 5) Thực hành: UDP gửi/nhận

**Sender**
```java
// UdpSender.java
import java.net.*;

public class UdpSender {
    public static void main(String[] args) throws Exception {
        try (DatagramSocket socket = new DatagramSocket()) {
            byte[] data = "hello-udp".getBytes();
            DatagramPacket p = new DatagramPacket(
                data, data.length, InetAddress.getByName("127.0.0.1"), 5556);
            socket.send(p);
            System.out.println("[UDP-Send] sent");
        }
    }
}
---
```
## 6) Gọi HTTP bằng HttpClient (Java 11+)

**Demo**
```java
// HttpGetDemo.java
import java.net.URI;
import java.net.http.*;

public class HttpGetDemo {
    public static void main(String[] args) throws Exception {
        HttpClient client = HttpClient.newHttpClient();
        HttpRequest req = HttpRequest.newBuilder()
                .uri(URI.create("https://httpbin.org/get"))
                .GET().build();
        HttpResponse<String> resp =
                client.send(req, HttpResponse.BodyHandlers.ofString());
        System.out.println("Status: " + resp.statusCode());
        System.out.println(resp.body());
    }
}
---
```
---

## 7) TCP vs UDP – nên dùng khi nào?

| Tiêu chí         | TCP                             | UDP                               |
|------------------|----------------------------------|------------------------------------|
| Đảm bảo tin cậy  | Có (ACK, retransmit, in-order)  | Không                              |
| Độ trễ           | Cao hơn                          | Thấp                               |
| Kiểu truyền      | Kết nối (stream)                | Không kết nối (datagram)           |
| Kiểm soát tắc nghẽn | Có                            | Tối thiểu/không                    |
| Phù hợp cho      | HTTP, FTP, gRPC, tải file        | VoIP, live video, game, DNS        |

**Mẹo nhanh**: API/web → **TCP/HTTP**. Truyền thời gian thực (telemetry/game/voice) → cân nhắc **UDP** và tự bù lỗi/kiểm tra mất gói.

---

## 8) Lỗi hay gặp & cách xử lý

- **`Address already in use`**: Port đang bận → đổi port (>=1024) hoặc tắt tiến trình cũ.  
- **Firewall/Antivirus chặn**: Không kết nối được giữa 2 máy → mở port trên firewall, kiểm tra địa chỉ đúng LAN/WAN.  
- **Deadlock khi đọc/ghi**: Mỗi bên đợi bên kia trước → thống nhất **giao thức** (kết thúc dòng `\n`, header length, JSON…) và đóng stream đúng lúc.  
- **Không đóng socket/stream**: Rò rỉ mô tả tệp → dùng `try-with-resources` hoặc `finally` để `close()`.  
- **NAT/Router** (UDP): Gói không về được do NAT → giữ “keep-alive” định kỳ hoặc dùng STUN/TURN khi cần xuyên NAT.  
- **Encoding**: Bên gửi UTF-8, bên nhận mặc định khác → luôn chỉ định `new InputStreamReader(..., StandardCharsets.UTF_8)` khi cần.

---

## 9) Bài tập đề nghị (đưa vào repo của bạn)

1. **Echo nâng cao**: thêm lệnh `TIME`, `UPPER <text>`, `QUIT`.  
2. **Chat mini**: server đa client, broadcast tin nhắn cho mọi người.  
3. **UDP “ping”**: gửi số tăng dần, receiver in ra và thống kê tỉ lệ mất gói.  
4. **HTTP downloader**: tải file theo URL, hiển thị % tiến độ; thử resume bằng header `Range`.  
5. **Refactor sang NIO**: viết Echo Server dùng `Selector` để phục vụ nhiều client trên 1 thread.

---

## 10) Lộ trình các bài tiếp theo trên blog

- **Bài 2**: TCP chi tiết & thiết kế giao thức tầng ứng dụng.  
- **Bài 3**: UDP & tối ưu hiệu năng (buffer, MTU, batch send/recv).  
- **Bài 4**: `HttpClient` – RESTful, JSON, timeout/retry.  
- **Bài 5**: WebSocket (Java ↔ JavaScript) – chat realtime.  
- **Bài 6**: NIO/Netty – server hiệu năng cao.  
- **Bài 7**: TLS/HTTPS cơ bản – chứng chỉ, SNI, HSTS.  
- **Bài 8**: Spring Boot REST API + bảo mật cơ bản.  
- **Bài 9**: Realtime với STOMP/Socket.IO và tích hợp frontend.
