---
title: "Bảo mật: SSL/TLS & HTTPS cơ bản"
date: 2025-10-20T14:29:19+07:00
draft: false
tags: ["security","tls","ssl","https","java"]
categories: ["Security","Java"]
---

# Mục tiêu bài học
- Hiểu khác nhau giữa SSL và TLS, vai trò của HTTPS.  
- Biết certificate, private key, CA, keystore/truststore trong Java.  
- Sinh certificate test (keytool / openssl) và cấu hình SSL trong Java (SSLContext, HttpsURLConnection, Tomcat/Spring Boot).  
- Nắm mutual TLS (mTLS), lỗi phổ biến và công cụ kiểm thử.

---

## 1) Khái niệm nhanh
- SSL = tiền thân; TLS = phiên bản hiện đại, an toàn hơn. Thường nói chung là "TLS".  
- HTTPS = HTTP chạy trên TLS (bảo mật kết nối: confidentiality, integrity, authentication).  
- Certificate (X.509) chứa public key và thông tin chủ sở hữu; private key giữ bí mật. CA ký certificate để clients tin tưởng.

---

## 2) Keystore vs Truststore (Java)
- Keystore: chứa private key + cert chain của server/app (dùng cho server để chứng thực bản thân).  
- Truststore: chứa CA certificates mà app tin tưởng (dùng client để kiểm tra server cert, hoặc server để xác thực client trong mTLS).  
- Java mặc định: cacerts là truststore hệ thống. Có thể cấu hình riêng bằng -Djavax.net.ssl.keyStore / -Djavax.net.ssl.trustStore.

---

## 3) Sinh certificate test (keytool trên Windows)

Tạo keystore chứa keypair self-signed:
```cmd
keytool -genkeypair -alias myserver -keyalg RSA -keysize 2048 -validity 365 \
 -keystore server.keystore.jks -dname "CN=localhost,OU=Dev,O=Company,L=City,ST=State,C=VN" -storepass changeit -keypass changeit
```

Xuất certificate (public) để client trust:
```cmd
keytool -export -alias myserver -file server.cer -keystore server.keystore.jks -storepass changeit
```

Tạo truststore (client) và import cert:
```cmd
keytool -import -alias myserver -file server.cer -keystore client.truststore.jks -storepass changeit -noprompt
```

Ghi chú: self-signed chỉ dùng dev/testing. Với production dùng certificate ký bởi CA hợp lệ (Let's Encrypt, commercial CA).

---

## 4) Sinh bằng OpenSSL (nếu cần PEM + PKCS12)

Tạo private key + CSR + self-signed cert:
```bash
openssl req -x509 -newkey rsa:2048 -nodes -keyout key.pem -out cert.pem -days 365 -subj "/CN=localhost"
```

Chuyển sang PKCS12 (Java có thể import .p12):
```bash
openssl pkcs12 -export -in cert.pem -inkey key.pem -name myserver -out server.p12 -passout pass:changeit
```

Import vào JKS nếu cần:
```cmd
keytool -importkeystore -destkeystore server.keystore.jks -srckeystore server.p12 -srcstoretype pkcs12 -alias myserver -deststorepass changeit -srcstorepass changeit
```

---

## 5) Ví dụ Java — SSLContext dùng keystore/truststore

Sử dụng keystore (client truststore) khi tạo HttpsURLConnection:
```java
// SSLClientExample.java
import javax.net.ssl.*;
import java.io.*;
import java.net.*;
import java.security.*;
import java.security.cert.CertificateException;

public class SSLClientExample {
    public static void main(String[] args) throws Exception {
        String truststore = "client.truststore.jks";
        char[] pass = "changeit".toCharArray();

        KeyStore ts = KeyStore.getInstance("JKS");
        try (FileInputStream fis = new FileInputStream(truststore)) {
            ts.load(fis, pass);
        }

        TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        tmf.init(ts);

        SSLContext ctx = SSLContext.getInstance("TLS");
        ctx.init(null, tmf.getTrustManagers(), null);
        HttpsURLConnection.setDefaultSSLSocketFactory(ctx.getSocketFactory());

        URL url = new URL("https://localhost:8443/hello");
        HttpsURLConnection con = (HttpsURLConnection) url.openConnection();
        System.out.println(con.getResponseCode());
    }
}
```

Ví dụ server đơn giản dùng Jetty/Tomcat hoặc Java SSLServerSocket cho thử nghiệm (không khuyến nghị production).

---

## 6) Spring Boot / Tomcat — cấu hình keystore

application.properties:
```properties
server.port=8443
server.ssl.key-store=classpath:server.p12
server.ssl.key-store-password=changeit
server.ssl.key-store-type=PKCS12
server.ssl.key-alias=myserver
```

Để bật mTLS (yêu cầu client certificate):
```properties
server.ssl.client-auth=need
server.ssl.trust-store=classpath:truststore.jks
server.ssl.trust-store-password=changeit
```

---

## 7) Mutual TLS (mTLS) — khi dùng và lưu ý
- mTLS: cả server và client trao chứng thực. Dùng trong hệ thống máy chủ-to-máy chủ, API nội bộ, hoặc khi muốn xác thực mạnh.  
- Yêu cầu: server có truststore chứa cert client CA; client có keystore chứa private key + cert.  
- Quản lý certificate lifecycle là thách thức (rotation, revocation).

---

## 8) Lỗi thường gặp & cách xử lý

- "PKIX path building failed" → certificate chain không được tin cậy: import CA/server cert vào truststore.  
- Hostname verification fail → CN/SAN của cert không khớp hostname (dev: CN=localhost hoặc thêm SAN).  
- Wrong keystore type/password → kiểm tra PKCS12 vs JKS và mật khẩu.  
- TLS protocol mismatch (SSLHandshakeException) → bật TLS phiên bản phù hợp (TLSv1.2/1.3) và cipher suites phù hợp.  
- Keystore không chứa private key (chỉ có cert exported) → import PKCS12 chứa private key.

---

## 9) Kiểm thử & công cụ
- Trên máy dev:
  - openssl s_client -connect localhost:8443 -showcerts  (kiểm tra handshake, chứng chỉ)
  - curl -vk --key client.key --cert client.crt https://localhost:8443 (kiểm thử mTLS)
- Trên Java:
  - bật debug TLS: -Djavax.net.debug=ssl,handshake,keymanager,trustmanager
- Certificate scanning / monitoring: kiểm tra expiry, fingerprint, CRL/OCSP nếu production.

---

## 10) Bài tập & hướng đi tiếp
- Cấu hình Spring Boot HTTPS với certificate Let's Encrypt (acme / certbot) và reverse-proxy (nginx).  
- Bật mTLS cho một endpoint nội bộ và viết client Java để gọi endpoint (keystore + truststore).  
- Thử handshake và debug với openssl s_client và javax.net.debug.  
- Nghiên cứu certificate rotation tự động và ACME protocol (Let's Encrypt).  
- Tìm hiểu HSTS, HPKP (deprecated), và TLS 1.3 cải tiến so với TLS 1.2.

---