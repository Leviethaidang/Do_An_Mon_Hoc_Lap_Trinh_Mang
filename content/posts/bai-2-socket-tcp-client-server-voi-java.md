---
title: "Socket TCP Client/Server với Java – Thiết kế giao thức ứng dụng"
date: 2025-10-20T15:00:00+07:00
draft: false
tags: ["java","tcp","socket","networking"]
categories: ["Java"]
---

# Mục tiêu bài học
- Hiểu **vòng đời kết nối TCP**: `listen → accept → read/write → close`.
- Biết **thiết kế giao thức ứng dụng**: theo dòng (`\n`) hay độ dài khung (length-prefixed).
- Viết **server đa client** bằng Thread/ThreadPool và **client** tương ứng.
- Xử lý lỗi phổ biến: half-close, timeouts, stuck-read.

---

## 1) Nhắc nhanh về TCP Socket
- **Server**: tạo `ServerSocket(port)` → `accept()` trả về `Socket` cho từng client.
- **Client**: `new Socket(host, port)` → lấy `InputStream/OutputStream`.
- Dữ liệu là **stream** liên tục → bạn phải tự **đóng gói thông điệp** (framing).

---

## 2) Thiết kế giao thức lớp ứng dụng (Application Protocol)

### 2.1. Kiểu 1 – Theo dòng (line-based)
- Mỗi thông điệp kết thúc bằng `\n`.  
- Dễ debug, hợp lệnh ngắn (text).
- Ví dụ: `PING\n`, `ECHO Xin chao\n`.

### 2.2. Kiểu 2 – Tiền tố độ dài (length-prefixed)
- Mỗi khung = `[length(int32)] [bytes…]`.  
- Hợp dữ liệu nhị phân, message dài/đa dòng.

> Bài này dùng **line-based** cho gọn.

---

## 3) Giao thức demo (text)
- `PING` → trả `PONG`
- `ECHO <text>` → trả lại `<text>`
- `ADD <a> <b>` → trả `a+b`
- `QUIT` → đóng kết nối

---

## 4) Server đa client dùng ThreadPool (blocking I/O)

```java
// CommandServer.java
import java.io.*;
import java.net.*;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.*;

public class CommandServer {
    public static void main(String[] args) throws IOException {
        int port = 6000;
        ExecutorService pool = Executors.newFixedThreadPool(16); // đơn giản
        try (ServerSocket server = new ServerSocket(port)) {
            System.out.println("[Server] Listening " + port);
            while (true) {
                Socket client = server.accept();
                pool.execute(() -> handle(client));
            }
        }
    }

    static void handle(Socket client) {
        String remote = client.getInetAddress() + ":" + client.getPort();
        System.out.println("[Server] Connected: " + remote);
        try (BufferedReader in = new BufferedReader(
                 new InputStreamReader(client.getInputStream(), StandardCharsets.UTF_8));
             BufferedWriter out = new BufferedWriter(
                 new OutputStreamWriter(client.getOutputStream(), StandardCharsets.UTF_8))) {

            String line;
            while ((line = in.readLine()) != null) {
                String resp = process(line.trim());
                out.write(resp);
                out.write("\n");
                out.flush();
                if (line.equalsIgnoreCase("QUIT")) break;
            }
        } catch (IOException e) {
            System.out.println("[Server] Error " + remote + " : " + e.getMessage());
        } finally {
            try { client.close(); } catch (IOException ignore) {}
            System.out.println("[Server] Closed: " + remote);
        }
    }

    static String process(String cmd) {
        if (cmd.equalsIgnoreCase("PING")) return "PONG";
        if (cmd.toUpperCase().startsWith("ECHO ")) return cmd.substring(5);
        if (cmd.toUpperCase().startsWith("ADD ")) {
            String[] p = cmd.split("\\s+");
            if (p.length == 3) {
                try {
                    long a = Long.parseLong(p[1]);
                    long b = Long.parseLong(p[2]);
                    return Long.toString(a + b);
                } catch (NumberFormatException ignore) {}
            }
            return "ERR usage: ADD <a> <b>";
        }
        if (cmd.equalsIgnoreCase("QUIT")) return "BYE";
        return "ERR unknown command";
    }
}
```
---
## 5) Client dòng lệnh (đọc console → gửi server)

```java
// CommandClient.java
import java.io.*;
import java.net.*;
import java.nio.charset.StandardCharsets;

public class CommandClient {
    public static void main(String[] args) throws IOException {
        String host = "127.0.0.1";
        int port = 6000;
        try (Socket s = new Socket(host, port);
             BufferedReader console = new BufferedReader(new InputStreamReader(System.in, StandardCharsets.UTF_8));
             BufferedReader in = new BufferedReader(new InputStreamReader(s.getInputStream(), StandardCharsets.UTF_8));
             BufferedWriter out = new BufferedWriter(new OutputStreamWriter(s.getOutputStream(), StandardCharsets.UTF_8))) {

            System.out.println("Connected. Type commands (PING/ECHO/ADD/QUIT).");
            String line;
            while ((line = console.readLine()) != null) {
                out.write(line);
                out.write("\n");
                out.flush();
                String resp = in.readLine();
                if (resp == null) break;
                System.out.println("=> " + resp);
                if ("BYE".equals(resp)) break;
            }
        }
    }
}
```
---

## 6) Timeout, half-close và shutdown
- Timeout đọc: dùng socket.setSoTimeout(ms) để tránh kẹt vô hạn ở readLine(); khi quá thời gian, ném SocketTimeoutException.
- Đóng một nửa kết nối (half-close):
- socket.shutdownOutput() để báo “hết dữ liệu ghi” nhưng vẫn đọc phản hồi từ server; hữu ích khi muốn kết thúc request dài mà không đóng hẳn socket.
- Với giao thức line-based đơn giản, thường không cần half-close.
---

## 7) Kiểm thử nhanh

1. Biên dịch & chạy server (Terminal A):
```bash
javac CommandServer.java && java CommandServer
```

2. Kết nối bằng client (Terminal B):
```bash
javac CommandClient.java && java CommandClient
```

3. Thử các lệnh (ở client):
- PING → PONG
- ECHO xin chao → xin chao
- ADD 10 25 → 35
- QUIT → BYE (client đóng)

Ví dụ tương tác:
```text
Client: PING
Server: PONG

Client: ECHO xin chao
Server: xin chao

Client: ADD 10 25
Server: 35

Client: QUIT
Server: BYE
```

## 8) Lỗi thường gặp & mẹo

- Không xuống dòng: server dùng line-based, nếu client không gửi '\n' thì server sẽ chờ — đảm bảo gọi out.write(... + "\n") và flush().
- Unicode vỡ: luôn dùng StandardCharsets.UTF_8 cho Reader/Writer.
- Port bận: đổi port hoặc kill process đang chiếm port (Windows: netstat -ano | findstr :6000 → taskkill /PID <pid> /F).
- Read block vô hạn: đặt timeout đọc: socket.setSoTimeout(ms) → bắt SocketTimeoutException.
- Too many threads: dùng ThreadPool (Executors) hoặc chuyển sang NIO/selector nếu cần hàng nghìn kết nối.
- Half-close: dùng socket.shutdownOutput() khi muốn báo EOF phía gửi nhưng vẫn muốn đọc phản hồi.
- Logging: log các lỗi/exception đầy đủ (có timestamp) giúp debug nhiều kết nối.

## 9) Bài tập nhỏ

- Thêm lệnh mới:
  - TIME → trả thời gian server hiện tại (ISO-8601).
  - REVERSE <text> → trả chuỗi đảo ngược của <text>.
- Ghi log: lưu mọi lệnh/response và địa chỉ client vào server.log (synchronized/async logger).
- Rate limit: nếu 1 client gửi >50 lệnh/giây → trả "ERR rate limit" và/hoặc ngắt kết nối. Gợi ý: dùng sliding window hoặc token bucket theo client.
- Unit test: viết test JUnit mô phỏng client socket để kiểm thử lệnh và lỗi (timeout, EOF).
- Nâng cấp: thêm command AUTH <token> để kiểm soát truy cập.

## 10) Hướng đi tiếp

- NIO/Selector: xử lý nhiều kết nối trên một hoặc vài thread (non-blocking).
- Netty: framework cho pipeline, codec, dễ triển khai length-prefixed binary protocol.
- Bảo mật: TLS/SSL (SSLSocket/SSLServerSocket) — học certificate, keystore.
- Hiệu năng & vận hành: benchmarking, profiling, pooling, backpressure, circuit breaker.
- Test tự động: viết integration tests, fuzzing, stress test (wrk/tsung).
- Triển khai: container hoá (Docker), giám sát (Prometheus), logging tập trung.
