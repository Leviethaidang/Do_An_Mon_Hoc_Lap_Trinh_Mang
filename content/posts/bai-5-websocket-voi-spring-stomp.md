---
title: "Websocket Với Spring Stomp"
date: 2025-10-20T14:29:18+07:00
draft: false
tags: ["websocket","spring","stomp","realtime"]
categories: ["Java","Spring"]
---
# Mục tiêu bài học
- Hiểu khác biệt giữa HTTP và WebSocket (full-duplex, persistent).  
- Nắm STOMP là gì và vì sao dùng STOMP trên WebSocket với Spring.  
- Cấu hình WebSocket trong Spring Boot (WebSocketConfig, MessageBroker).  
- Viết controller xử lý @MessageMapping và client JS dùng SockJS + Stomp.js.  
- Triển khai, bảo mật, scale bằng broker relay (RabbitMQ) và kiểm thử cơ bản.

---

## 1) Tại sao dùng WebSocket?
- HTTP: request → response, mỗi lần cần tạo kết nối (stateless).  
- WebSocket: kết nối persistent, full-duplex, phù hợp realtime (chat, notifications, live metrics).  
- STOMP: đơn giản hoá messaging trên WebSocket (publish/subscribe, destinations).

---

## 2) Kiến trúc chung với Spring
- Client (browser / native) mở WebSocket → Spring nhận và route message qua destinations.  
- Spring cung cấp:
  - SimpleMessageBroker (in-memory) cho dev/small scale.
  - Broker relay (RabbitMQ) để scale nhiều instance và chịu tải lớn.  
- Controller để xử lý inbound messages (@MessageMapping) và gửi outbound (@SendTo, SimpMessagingTemplate).

---

## 3) Cấu hình cơ bản (Spring Boot)

```java
// ...existing code...
// filepath: c:\Users\dangl\OneDrive\Desktop\Blog\content\posts\bai-5-websocket-voi-spring-stomp.md
// WebSocketConfig.java
package com.example.websocket;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.*;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // SockJS fallback, endpoint cho client connect
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*") // production: set cụ thể
                .withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        // Prefix cho controller inbound
        registry.setApplicationDestinationPrefixes("/app");
        // Simple broker cho /topic và /queue
        registry.enableSimpleBroker("/topic", "/queue");
        // Nếu dùng RabbitMQ: registry.enableStompBrokerRelay(...) 
    }
}
```

Ghi chú: production không nên dùng setAllowedOriginPatterns("*"); cấu hình CORS kỹ hơn.

---

## 4) Controller xử lý message

```java
// ChatController.java
package com.example.websocket;

import org.springframework.messaging.handler.annotation.*;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Controller;

@Controller
public class ChatController {

    private final SimpMessagingTemplate template;

    public ChatController(SimpMessagingTemplate template) {
        this.template = template;
    }

    // client gửi đến /app/chat.send
    @MessageMapping("/chat.send")
    @SendTo("/topic/public") // đơn giản: broadcast qua topic
    public ChatMessage send(ChatMessage msg) {
        // xử lý (thêm timestamp, validate, auth...)
        return msg;
    }

    // hoặc gửi riêng bằng SimpMessagingTemplate
    @MessageMapping("/chat.private")
    public void privateMessage(@Payload ChatMessage msg, @Header("simpSessionId") String sessionId) {
        template.convertAndSendToUser(msg.getTargetUser(), "/queue/messages", msg);
    }
}
```

Mẫu ChatMessage: gồm sender, content, type, timestamp.

---

## 5) Client (browser) — SockJS + Stomp.js

```html
<!-- client.html -->
<!doctype html>
<html>
<head><meta charset="utf-8"><title>WS Chat</title></head>
<body>
<script src="https://cdn.jsdelivr.net/npm/sockjs-client@1/dist/sockjs.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/stompjs@2.3.3/lib/stomp.min.js"></script>
<script>
  const socket = new SockJS('/ws');
  const stompClient = Stomp.over(socket);
  stompClient.connect({}, function(frame) {
    console.log('Connected: ' + frame);
    stompClient.subscribe('/topic/public', function(message) {
      console.log('Got:', JSON.parse(message.body));
    });
    // gửi message
    const msg = { sender: 'alice', content: 'hello' };
    stompClient.send('/app/chat.send', {}, JSON.stringify(msg));
  });
</script>
</body>
</html>
```

Ghi chú: Stomp.over có option để disable debug logs; khi cần TLS dùng wss:// và cấu hình proxy/nginx.

---

## 6) Bảo mật & xác thực
- WebSocket kết nối ban đầu qua HTTP(s) handshake → có thể dùng cookie/session hoặc header Authorization (Bearer).  
- Trong Spring, bạn có thể:
  - Trao token trong connect headers (client gửi header khi connect).
  - Sử dụng ChannelInterceptor để kiểm tra token và map principal:
    - implement HandshakeInterceptor hoặc ChannelInterceptorAdapter để validate và set user (SimpMessageHeaderAccessor.setUser()).
- Cẩn thận: không log token, giới hạn origins.

---

## 7) Scale — broker relay (RabbitMQ)
- Khi cần scale nhiều instance và share subscriptions: dùng broker relay.
- Ví dụ configure để relay đến RabbitMQ:

```java
// enable STOMP broker relay
registry.enableStompBrokerRelay("/topic", "/queue")
        .setRelayHost("rabbit.example")
        .setRelayPort(61613)
        .setClientLogin("guest")
        .setClientPasscode("guest");
```

- Khi relay: RabbitMQ hoặc ActiveMQ xử lý routing, durable queue, persistence nếu cần.

---

## 8) Testing & Debugging
- Local dev: dùng enableSimpleBroker; dễ kiểm thử chức năng messaging.  
- Unit test controller: dùng Spring Test + SimpMessagingTemplate mock.  
- Integration test: start server, kết nối STOMP client (ví dụ stompj or WebSocket client) để gửi/subscribe.  
- Debugging tips:
  - Kiểm tra handshake (network tab hoặc logs).  
  - Đăng ký interceptors (ClientInboundChannel) để log messages.  
  - Kiểm tra broker relay kết nối (RabbitMQ logs).

---

## 9) Lỗi thường gặp & mẹo
- Client không nhận message: kiểm tra destination (prefix /app vs /topic), subscription path đúng không.  
- CORS / Origin blocked: cấu hình setAllowedOriginPatterns hoặc proxy CORS.  
- SockJS fallback không hoạt động: kiểm tra static resources và endpoint path.  
- Memory leak: unsubscribe khi component unmount (frontend) và cleanup resources.  
- Throttle spam: rate-limit message per-session, validate payload size.

---

## 10) Bài tập & nâng cao
- Xây chat group: private rooms với destinations theo room id (/topic/room.{id}).  
- Implement presence: track connected users, broadcast join/leave events.  
- Persist message history vào DB và replay khi user join.  
- Thay SimpleBroker bằng RabbitMQ (broker relay) và đo throughput/latency.  
- Thêm per-user rate limit và max payload size.  
- Triển khai TLS (wss) qua reverse-proxy (nginx) và cấu hình secure cookies.

---