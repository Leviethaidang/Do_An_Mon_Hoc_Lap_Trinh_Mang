---
title: "Javascript: fetch, axios và http clients"
date: 2025-10-20T14:29:19+07:00
draft: false
tags: ["javascript","http","fetch","axios","node"]
categories: ["Frontend","Backend"]
---

# Mục tiêu bài học
- Hiểu sự khác biệt giữa Fetch API (browser / Node), axios và các HTTP client phổ biến.  
- Biết cách xử lý JSON, timeouts, cancel/abort, upload multipart, streaming và xử lý lỗi HTTP.  
- Cấu hình instance (axios), interceptors, retry cơ bản và testing (Mock Service Worker / axios-mock-adapter).  
- Lưu ý CORS, cookies/credentials và bảo mật khi gọi API.

---

## 1) Tổng quan nhanh
- Fetch: chuẩn web API, Promise-based; trong Node hiện là built-in (Node 18+) hoặc qua undici/node-fetch.  
- Axios: thư viện phổ biến, chạy trên browser & Node, tự reject trên status != 2xx, hỗ trợ interceptors, transform, timeout, cancel token (v0.x) / AbortController.  
- Các client khác: got (Node), undici (hiệu năng cao), superagent — chọn theo nhu cầu (browser vs Node, streaming, retry).

---

## 2) Fetch API — ví dụ & lưu ý

GET (async/await):
```javascript
const resp = await fetch('/api/users', {
  headers: { 'Accept': 'application/json' }
});
if (!resp.ok) throw new Error(`${resp.status} ${resp.statusText}`);
const data = await resp.json();
```

POST JSON:
```javascript
const body = { name: 'alice' };
const r = await fetch('/api/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify(body)
});
```

Abort / timeout (AbortController):
```javascript
const ac = new AbortController();
const timeout = setTimeout(() => ac.abort(), 5000);
try {
  const r = await fetch('/api/slow', { signal: ac.signal });
  // ...
} catch (err) {
  if (err.name === 'AbortError') console.log('timed out/aborted');
} finally {
  clearTimeout(timeout);
}
```

Lưu ý:
- fetch không reject khi server trả 4xx/5xx — cần kiểm tra response.ok.  
- Chú ý CORS: để gửi cookie cần credentials: 'include' và server phải cho phép Access-Control-Allow-Credentials.

---

## 3) Axios — ví dụ & tính năng chính

Tạo instance & GET:
```javascript
import axios from 'axios';

const api = axios.create({
  baseURL: 'https://api.example.com',
  timeout: 5000, // ms
  withCredentials: true
});

try {
  const resp = await api.get('/items');
  console.log(resp.data); // axios unwraps JSON
} catch (err) {
  if (err.response) {
    // server responded with non-2xx
    console.error(err.response.status, err.response.data);
  } else {
    // network / timeout / cancelled
    console.error(err.message);
  }
}
```

Interceptors (auth header, logging):
```javascript
api.interceptors.request.use(cfg => {
  cfg.headers.Authorization = `Bearer ${getToken()}`;
  return cfg;
});

api.interceptors.response.use(
  r => r,
  err => {
    if (err.response && err.response.status === 401) refreshToken();
    return Promise.reject(err);
  }
);
```

Upload multipart/form-data:
```javascript
const fd = new FormData();
fd.append('file', fileInput.files[0]);
await api.post('/upload', fd, {
  headers: { 'Content-Type': 'multipart/form-data' }
});
```

Lưu ý:
- Axios tự parse JSON và reject on non-2xx — thuận tiện nhưng cần xử lý err.response.  
- Timeout trong axios sẽ reject với code 'ECONNABORTED'.

---

## 4) Server-side (Node) — fetch vs axios vs undici
- Node 18+ có global fetch (độ trễ thấp). undici là client chính thức/hiệu năng cao.  
- got cung cấp retry, hooks, stream-friendly API.  
- Chọn axios khi muốn API nhất quán giữa browser/Node; chọn undici/got cho throughput cao và streaming.

Ví dụ Node fetch:
```javascript
const res = await fetch('https://api.example.com/data');
const json = await res.json();
```

---

## 5) Error handling & retries
- Với fetch: check resp.ok, parse error body if any.  
- Với axios: err.response chứa status/data; err.request khi không có response.  
- Retry: dùng libraries (axios-retry, p-retry, retry-axios) hoặc implement exponential backoff; tránh retry non-idempotent POST trừ khi safe.

---

## 6) CORS, cookies và CSRF
- CORS: server cần Access-Control-Allow-Origin + headers. Với credentials (cookies), set Access-Control-Allow-Credentials: true và client dùng credentials: 'include' / withCredentials: true.  
- CSRF: nếu dùng cookie-based auth, bảo vệ endpoint bằng CSRF token hoặc chuyển sang same-site cookies / use Authorization header (Bearer) để tránh CSRF.

---

## 7) Streaming & large payloads
- Fetch/undici/axios support streaming responses (ReadableStream / Node stream). Với file download/streaming, xử lý chunk-by-chunk để giảm memory.  
- Axios supports responseType: 'stream' in Node.

---

## 8) Testing HTTP clients
- Browser: Mock Service Worker (MSW) để mock network in integration tests.  
- Unit tests: jest-fetch-mock / node-fetch mock for fetch; axios-mock-adapter for axios.  
- Integration: spin up a local test server (express) or use nock to intercept Node HTTP calls.

---

## 9) Security & best practices
- Không gửi secrets/tokens trong URL.  
- Validate/escape inputs before including in URL path.  
- Use HTTPS; verify certificate on server.  
- Limit request rate & implement client-side backoff on 429.  
- Sanitize/validate file uploads (size/type) on server.

---

## 10) Bài tập & nâng cao
- So sánh throughput/latency: fetch (undici) vs axios vs got bằng benchmark đơn giản.  
- Viết retry/backoff wrapper cho fetch với exponential jitter.  
- Tạo axios instance có interceptor refresh-token (queuing requests khi refresh diễn ra).  
- Viết integration test cho frontend API calls bằng Mock Service Worker.  
- Triển khai streaming upload/download: upload file lớn chunked và resume.

---