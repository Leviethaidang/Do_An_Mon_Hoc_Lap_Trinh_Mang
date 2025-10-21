---
title: "UDP & tối ưu hiệu năng trong Java"
date: 2025-10-20T16:30:00+07:00
draft: false
tags: ["java","udp","networking","performance"]
categories: ["Java"]
---

# Mục tiêu bài học
- Hiểu đặc tính cơ bản của UDP: connectionless, datagram, không đảm bảo thứ tự/mất gói.  
- Biết khi nào chọn UDP (low-latency, realtime, custom reliability) và khi nào chọn TCP.  
- Triển khai cơ chế "reliable" tối thiểu trên UDP (stop-and-wait, seq/ACK) bằng DatagramSocket / DatagramChannel.  
- Tối ưu hiệu năng: kích thước gói, buffer reuse, socket buffer, direct buffer, tránh GC.  
- Kiểm thử bằng công cụ (wireshark, iperf, netem) và mô phỏng mất gói/độ trễ.

---

## 1) UDP — tóm tắt nhanh
- UDP gửi/nhận theo datagram (gói nguyên vẹn). Mỗi datagram ≤ MTU (tốt nhất ≤ 1200–1400 bytes để tránh fragmentation).  
- Không có kết nối, không retransmit tự động, không đảm bảo thứ tự hoặc duplicate suppression — mọi logic reliability phải nằm ở ứng dụng nếu cần.  
- Lợi thế: overhead thấp, latency thấp, phù hợp realtime (VoIP, game, streaming custom), multicast.

---

## 2) Khi nào dùng UDP
- Yêu cầu latency thấp hơn throughput/độ tin cậy.  
- Ứng dụng thực hiện chính sách riêng: own FEC/retransmit/ARQ.  
- Multicast hoặc broadcast (LAN) cần gửi tới nhiều receiver.

---

## 3) Minimal Reliable: Stop-and-Wait ARQ (DatagramSocket)

Ý tưởng: gửi một gói có SEQ; chờ ACK từ receiver; nếu timeout thì retransmit.

Receiver (trả ACK):
```java
// SimpleUdpReceiver.java
import java.net.*;

public class SimpleUdpReceiver {
    public static void main(String[] args) throws Exception {
        int port = 7001;
        byte[] buf = new byte[1500];
        try (DatagramSocket socket = new DatagramSocket(port)) {
            System.out.println("UDP receiver listening " + port);
            while (true) {
                DatagramPacket p = new DatagramPacket(buf, buf.length);
                socket.receive(p);
                String msg = new String(p.getData(), 0, p.getLength());
                System.out.println("RECV: " + msg + " from " + p.getAddress() + ":" + p.getPort());

                // parse seq (format "SEQ:<n>|DATA:<...>")
                int seq = -1;
                try {
                    if (msg.startsWith("SEQ:")) {
                        int bar = msg.indexOf('|');
                        seq = Integer.parseInt(msg.substring(4, bar));
                    }
                } catch (Exception ignored) {}

                if (seq >= 0) {
                    String ack = "ACK:" + seq;
                    byte[] a = ack.getBytes();
                    DatagramPacket ap = new DatagramPacket(a, a.length, p.getAddress(), p.getPort());
                    socket.send(ap);
                }
            }
        }
    }
}
```

Sender (retry với timeout):
```java
// SimpleUdpSender.java
import java.net.*;
import java.util.concurrent.TimeUnit;

public class SimpleUdpSender {
    public static void main(String[] args) throws Exception {
        String host = "127.0.0.1";
        int port = 7001;
        int seq = 1;
        String payload = "SEQ:" + seq + "|DATA:hello-udp";
        byte[] data = payload.getBytes();

        try (DatagramSocket socket = new DatagramSocket()) {
            socket.setSoTimeout(500); // ms
            DatagramPacket p = new DatagramPacket(data, data.length, InetAddress.getByName(host), port);

            int maxRetry = 5;
            for (int attempt = 1; attempt <= maxRetry; attempt++) {
                socket.send(p);
                System.out.println("SENT attempt#" + attempt + " : " + payload);

                byte[] buf = new byte[64];
                DatagramPacket ack = new DatagramPacket(buf, buf.length);
                try {
                    socket.receive(ack);
                    String s = new String(ack.getData(), 0, ack.getLength());
                    if (s.equals("ACK:" + seq)) {
                        System.out.println("GOT " + s + " ✅");
                        break;
                    }
                } catch (SocketTimeoutException e) {
                    System.out.println("timeout, retrying...");
                    if (attempt == maxRetry) System.out.println("failed after retries ❌");
                }
                TimeUnit.MILLISECONDS.sleep(100L * attempt);
            }
        }
    }
}
```

Ghi chú: Stop-and-wait đơn giản nhưng throughput thấp. Để tăng throughput cần sliding-window / selective repeat.

---

## 4) Sliding window & cải tiến
- Thay vì chờ 1 ACK, giữ cửa sổ W gói được gửi cùng lúc; ghi seq trên mỗi gói; peer ACK từng seq (cumulative hoặc selective).  
- Cần buffer retransmit, xử lý duplicate, và timeout per-packet hoặc per-window.  
- Tham khảo thuật toán TCP (SACK, fast retransmit) nếu muốn triển khai tương tự.

---

## 5) NIO — DatagramChannel (non-blocking) example

Sử dụng DatagramChannel giúp tích hợp với Selector hoặc dùng non-blocking loop cho cao cấp hơn.

```java
// UdpNioEcho.java
import java.net.*;
import java.nio.ByteBuffer;
import java.nio.channels.DatagramChannel;

public class UdpNioEcho {
    public static void main(String[] args) throws Exception {
        try (DatagramChannel ch = DatagramChannel.open()) {
            ch.bind(new InetSocketAddress(7002));
            ch.configureBlocking(false);
            ByteBuffer buf = ByteBuffer.allocateDirect(2048);
            System.out.println("NIO UDP listening 7002");
            while (true) {
                buf.clear();
                SocketAddress addr = ch.receive(buf);
                if (addr == null) {
                    Thread.sleep(10);
                    continue;
                }
                buf.flip();
                ch.send(buf, addr); // echo back
            }
        }
    }
}
```

Ghi chú: khi dùng selector, register channel với OP_READ để scale trên ít thread hơn.

---

## 6) Tối ưu hiệu năng — checklist
- Kích thước gói:
  - Giữ payload ≤ MTU minus headers (~1200 bytes an toàn trên Internet) để tránh IP fragmentation.
- Tái sử dụng buffer/DatagramPacket:
  - Tránh tạo byte[]/DatagramPacket cho mỗi gói trong hot path — reuse để giảm GC.
- Socket buffer:
  - Increase SO_RCVBUF / SO_SNDBUF nếu lượng traffic cao:
    - socket.setReceiveBufferSize(1 << 20); // 1MB
- Direct ByteBuffer (DatagramChannel) để giảm copy qua native layer.
- Batching & zero-allocation:
  - Pre-serialize headers, dùng pooled buffers, avoid per-packet logging.
- CPU affinity / busy-loop:
  - Tránh select/poll busy-loop; tune select timeout or use epoll-based selector provider (JVM does on Linux).
- Avoid blocking operations (DB, disk) on UDP receive worker — hand off to worker thread pool if needed.

---

## 7) Testing, debugging & tools
- Wireshark / tshark: quan sát loss, retransmit, fragmentation, RTT.
- tc (Linux netem) để mô phỏng delay/loss/jitter:
  - tc qdisc add dev lo root netem loss 5% delay 50ms
- iperf (udp): đo throughput & loss:
  - iperf3 -u -c server -b 10M
- Log minimal metadata (seq, timestamp) để tính loss/RTT; export metrics (Prometheus) cho monitoring.

---

## 8) Lỗi thường gặp & mẹo
- Fragmentation → mất cả message nếu 1 fragment mất: tránh gói lớn.  
- GC spikes gây jitter → giảm allocations, reuse buffers.  
- Socket buffer too small → kernel drop → tăng SO_RCVBUF.  
- Race / duplicate ACKs → robust seq/ack design; dedup by seq.  
- NAT timeout → gửi keep-alive nếu cần peer-to-peer.

---

## 9) Security considerations
- UDP dễ bị spoofing / amplification (NTP/DNS reflexive attacks).  
- Validate source, use application-layer authentication (HMAC, token).  
- Nếu cần bảo mật, áp dụng DTLS (Datagram TLS) thay vì tự làm encryption.

---

## 10) Bài tập
- Triển khai sliding-window sender/receiver (cumulative ACK) và đo throughput so với stop-and-wait.  
- Viết line-based protocol (newline terminated) trên UDP: làm sao detect message boundary và xử lý fragment? (thực tế: tránh; dùng length-prefix hoặc tách ứng dụng).  
- So sánh DatagramSocket vs DatagramChannel: benchmark latency & throughput.  
- Thêm FEC (forward error correction) đơn giản: gửi 1 parity packet sau N data packets; đo giảm retransmit.  
- Tạo testbench: client sinh traffic theo pattern burst/steady, dùng netem để inject loss/jitter và thu metrics (loss, RTT P50/P95/P99).

---