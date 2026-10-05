# Microservices demo

| Service | Port | Công nghệ | Vai trò |
|---------|------|-----------|---------|
| gateway-service | **8888** | Spring Cloud Gateway (WebFlux/Netty) | Cổng vào duy nhất, định tuyến request |
| identity-service | 8080 | Spring Boot MVC + Security + JPA + PostgreSQL | Đăng ký / đăng nhập / cấp JWT (Bài 1–3) |
| product-service | 8082 | Spring Boot MVC + Security + JJWT | Resource Server: verify JWT, phân quyền `@PreAuthorize` (Bài 6) |

## Định tuyến (gateway-service/src/main/resources/application.yml)

| Client gọi Gateway | Predicate | Filter | Chuyển tiếp tới |
|--------------------|-----------|--------|-----------------|
| `:8888/identity/api/auth/login` | `Path=/identity/**` | `StripPrefix=1` | `:8080/api/auth/login` |
| `:8888/product/api/products` | `Path=/product/**` | `StripPrefix=1` | `:8082/api/products` |
| path khác | — | — | 404 tại Gateway |

`StripPrefix=1` bỏ đoạn path đầu tiên (`/identity`, `/product`) trước khi chuyển tiếp, nên service bên dưới không cần biết về tiền tố của Gateway.

## Chạy

```bash
cd identity-service && docker compose up -d && ./gradlew bootRun   # terminal 1
cd product-service  && ./gradlew bootRun                           # terminal 2
cd gateway-service  && ./gradlew bootRun                           # terminal 3
```

## Kiểm tra

```bash
curl -X POST http://localhost:8888/identity/api/auth/register -H "Content-Type: application/json" -d '{"username":"bao","password":"123456"}'
curl -X POST http://localhost:8888/identity/api/auth/login    -H "Content-Type: application/json" -d '{"username":"bao","password":"123456"}'
TOKEN=<accessToken lấy từ login>
curl http://localhost:8888/product/api/products -H "Authorization: Bearer $TOKEN"
```

Hoặc import `postman/gateway.postman_collection.json` → Run collection (request Product dùng token lưu từ bước Login).
Log DEBUG của Gateway sẽ in route được khớp, ví dụ `Route matched: identity-service`.

## Bài 6 — Product-Service là Resource Server

```
Client ──Bearer JWT──► Gateway :8888 ──(giữ nguyên header Authorization)──► product-service :8082
                                                                              │
                                     JwtAuthenticationFilter (trước UsernamePasswordAuthenticationFilter)
                                       1. đọc "Authorization: Bearer <token>"
                                       2. verify chữ ký bằng app.jwt.secret (giống identity-service) + exp
                                       3. sub -> username, role "ADMIN" -> [ROLE_ADMIN]
                                       4. UsernamePasswordAuthenticationToken -> SecurityContextHolder
                                                                              │
                                     @PreAuthorize trên ProductController
```

product-service **không có DB user**, không gọi sang identity-service: chỉ tin vào chữ ký JWT.

| Endpoint | Quyền | Không token | Token STUDENT/INSTRUCTOR | Token ADMIN |
|----------|-------|-------------|------------|-------------|
| `GET /api/products` | `isAuthenticated()` | 401 | 200 | 200 |
| `DELETE /api/products/{id}` | `hasRole('ADMIN')` | 401 | **403** | **200** |

Token sai chữ ký / hết hạn → 401 ngay tại filter.

**Tài khoản ADMIN:** `/register` chỉ tạo `STUDENT`/`INSTRUCTOR`, nên identity-service tự tạo sẵn `admin` / `admin123` khi khởi động
(`AdminSeeder`, cấu hình `app.admin.*`, override bằng `ADMIN_USERNAME`/`ADMIN_PASSWORD`).
Hoặc nâng quyền user có sẵn: `UPDATE users SET role = 'ADMIN' WHERE username = 'bao';` rồi đăng nhập lại để lấy token mới.

**Kiểm thử:** import `postman/bai6-product-security.postman_collection.json` → Run.
Kịch bản 1 (STUDENT xóa → 403) là request 6, kịch bản 2 (ADMIN xóa → 200) là request 7.
Dữ liệu sản phẩm nằm trong bộ nhớ, khởi động lại product-service để có lại sản phẩm 1.

## Ghi chú

- gateway-service **không** có `spring-boot-starter-web`: Gateway là ứng dụng reactive chạy trên Netty; nếu có Spring MVC trong classpath, ứng dụng sẽ báo lỗi và không khởi động được.
- gateway-service dùng Spring Boot 3.4.10 + Spring Cloud 2024.0.2 để giữ đúng tên `spring-cloud-starter-gateway` và prefix cấu hình `spring.cloud.gateway.routes`.
  (Từ Spring Cloud 2025.0 artifact được đổi thành `spring-cloud-starter-gateway-server-webflux` và prefix thành `spring.cloud.gateway.server.webflux.*`.)
