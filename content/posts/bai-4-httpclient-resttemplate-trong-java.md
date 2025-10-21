---
title: "Httpclient Resttemplate Trong Java"
date: 2025-10-20T14:29:18+07:00
draft: false
tags: []
categories: []
---
# Mục tiêu bài học
- Hiểu khác biệt giữa Java 11+ HttpClient, Apache HttpClient, Spring RestTemplate và reactive WebClient.  
- Biết cấu hình timeout, connection pooling, headers và xử lý JSON.  
- Viết ví dụ GET/POST đồng bộ và bất đồng bộ, và cấu hình RestTemplate dùng connection-pooling.  
- Kiểm thử đơn vị và xử lý lỗi phổ biến (timeouts, redirects, 4xx/5xx).

---

## 1) Tổng quan nhanh
- Java 11 HttpClient: chuẩn, tích hợp trong JDK, hỗ trợ sync/async (CompletableFuture), HTTP/2.  
- Apache HttpClient: linh hoạt, nhiều tùy chỉnh (pooling, interceptors).  
- Spring RestTemplate: wrapper tiện dụng cho REST trong ứng dụng Spring (blocking). Đang được thay dần bằng WebClient cho reactive.  
- WebClient (Spring WebFlux): non-blocking, phù hợp tải cao/streaming.

---

## 2) Khi chọn cái nào
- Ứng dụng Spring MVC truyền thống: RestTemplate (hoặc RestTemplate với HttpComponents để pooling).  
- Ứng dụng non-blocking / reactive: WebClient.  
- Không dùng Spring / muốn đơn giản: Java HttpClient.  
- Cần tùy chỉnh thấp-hơn/legacy: Apache HttpClient.

---

## 3) Java 11+ HttpClient — ví dụ cơ bản

GET đồng bộ:
```java
// Java 11 HttpClient - sync GET
import java.net.http.*;
import java.net.*;
import java.time.Duration;

HttpClient client = HttpClient.newBuilder()
    .connectTimeout(Duration.ofSeconds(5))
    .build();

HttpRequest req = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/get"))
    .GET()
    .header("Accept", "application/json")
    .build();

HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
System.out.println(resp.statusCode());
System.out.println(resp.body());
```

POST JSON async:
```java
// Java 11 HttpClient - async POST
String json = "{\"name\":\"alice\"}";
HttpRequest req = HttpRequest.newBuilder()
    .uri(URI.create("https://httpbin.org/post"))
    .timeout(Duration.ofSeconds(10))
    .header("Content-Type", "application/json")
    .POST(HttpRequest.BodyPublishers.ofString(json))
    .build();

client.sendAsync(req, HttpResponse.BodyHandlers.ofString())
      .thenApply(HttpResponse::body)
      .thenAccept(System.out::println)
      .join();
```

Timeouts: setConnectTimeout trên builder; set timeout cho từng request với HttpRequest.timeout().

---

## 4) Spring RestTemplate — ví dụ cơ bản

Sử dụng RestTemplate đơn giản:
```java
// RestTemplate - simple GET/POST
RestTemplate rt = new RestTemplate();
String url = "https://httpbin.org/get";
ResponseEntity<String> r = rt.getForEntity(url, String.class);

String postUrl = "https://httpbin.org/post";
Map<String,String> body = Map.of("name","bob");
ResponseEntity<String> p = rt.postForEntity(postUrl, body, String.class);
```

Cấu hình timeout (SimpleClientHttpRequestFactory):
```java
SimpleClientHttpRequestFactory f = new SimpleClientHttpRequestFactory();
f.setConnectTimeout(3000);
f.setReadTimeout(5000);
RestTemplate rt = new RestTemplate(f);
```

Pooling (khi cần throughput): dùng Apache HttpClient + HttpComponentsClientHttpRequestFactory:
```java
PoolingHttpClientConnectionManager cm = new PoolingHttpClientConnectionManager();
cm.setMaxTotal(200);
cm.setDefaultMaxPerRoute(50);

CloseableHttpClient httpClient = HttpClients.custom()
    .setConnectionManager(cm)
    .build();

HttpComponentsClientHttpRequestFactory rf = new HttpComponentsClientHttpRequestFactory(httpClient);
rf.setConnectTimeout(3000);
rf.setReadTimeout(5000);

RestTemplate rt = new RestTemplate(rf);
```

---

## 5) JSON mapping & headers
- Dùng Jackson (ObjectMapper) hoặc let Spring tự convert nếu dùng RestTemplate + MappingJackson2HttpMessageConverter.
- Luôn set Content-Type/Accept khi gửi JSON.  
- Ví dụ deserialize với HttpClient + ObjectMapper:
```java
ObjectMapper om = new ObjectMapper();
MyResp respObj = om.readValue(resp.body(), MyResp.class);
```

---

## 6) Retry, circuit-breaker & backoff
- Không tự làm retry ở mọi chỗ: dùng library (Resilience4j, Spring Retry) để retry/backoff/circuit-breaker.  
- Với blocking clients (RestTemplate, HttpClient sync), retry sẽ block thread — cân nhắc thread pool.

---

## 7) TLS, 인증 và headers bảo mật
- TLS: cấu hình SSLContext khi cần truststore/keystore tuỳ chỉnh (HttpClient/Apache HttpClient/RestTemplate đều hỗ trợ).  
- Authorization: Bearer token trong header Authorization. Tránh in token vào logs.

---

## 8) Kiểm thử & debugging
- Local test: dùng MockWebServer (OkHttp) hoặc WireMock để mock responses.  
- Unit test RestTemplate: dùng MockRestServiceServer (spring-test) hoặc mock RestTemplate via Mockito.  
- Kiểm tra redirect/302, 4xx/5xx xử lý rõ ràng (nên wrap lỗi vào exception có ý nghĩa).

Ví dụ MockRestServiceServer:
```java
MockRestServiceServer server = MockRestServiceServer.createServer(restTemplate);
server.expect(requestTo("/api")).andRespond(withSuccess("{\"ok\":true}", MediaType.APPLICATION_JSON));
```

---

## 9) Lỗi thường gặp & mẹo
- SocketTimeoutException: kiểm tra read/connect timeout.  
- Connection leak: khi dùng Apache pooling, đảm bảo reuse/close đúng và cấu hình TTL cho connections.  
- Chunked/streaming: dùng BodyHandlers.ofInputStream / Stream để xử lý streaming lớn.  
- Charset: ensure UTF-8 when serializing/deserializing.  
- HTTP/2: HttpClient hỗ trợ HTTP/2; server cũng phải hỗ trợ để tận dụng.

---

## 10) Bài tập & hướng đi tiếp
- Thay RestTemplate bằng WebClient: viết cùng API nhưng non-blocking, benchmark throughput/latency.  
- Cấu hình RestTemplate với PoolingHttpClientConnectionManager và đo TPS khi gọi nhiều endpoint đồng thời.  
- Viết integration test với WireMock cho 5xx, 429 rate-limit, và retry/backoff bằng Resilience4j.  
- Triển khai tracing: thêm OpenTelemetry/Zipkin headers để trace xuyên dịch vụ.

---