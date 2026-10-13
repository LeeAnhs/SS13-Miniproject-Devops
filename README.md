# Bài kiểm tra đầu giờ — Đóng gói và chạy Campus Hub với Docker

## 1. Thông tin sinh viên
- **Họ và tên:** Lê Xuân Ánh
- **Mã sinh viên:** B24DTCN191
- **Email:** anhlx.b24dtcn191@stu.ptit.edu.vn
- **Lớp / Khóa:** D24TXCN03-B / PTIT

---

## 2. Các lệnh dùng để chạy hệ thống

### 2.1. Kiểm thử backend với Gradle
```bash
cd backend
./gradlew test
```
*(Trên Windows có thể chạy `./gradlew.bat test`)*

### 2.2. Đóng gói Backend thành Docker Image
```bash
# Build Docker image cho backend độc lập
docker build -t campus-hub-backend:latest ./backend

# Kiểm tra danh sách Docker image
docker images
```

### 2.3. Khởi chạy toàn bộ hệ thống bằng Docker Compose
```bash
# Kiểm tra tính hợp lệ của file cấu hình Docker Compose
docker compose config

# Khởi chạy hệ thống ở chế độ nền (Detached mode)
docker compose up -d

# Xem trạng thái hoạt động của các container
docker compose ps

# Xem log khởi động của backend
docker compose logs -f backend
```

### 2.4. Dừng hệ thống
```bash
docker compose down
# Hoặc dừng và xoá volume dữ liệu nếu muốn làm mới:
# docker compose down -v
```

---

## 3. Cổng và URL kiểm tra hệ thống

| Dịch vụ | URL / Cổng | Mục đích |
|---|---|---|
| **Backend Health Check** | [http://localhost:8080/actuator/health](http://localhost:8080/actuator/health) | Kiểm tra trạng thái ứng dụng (`UP`) |
| **Backend API (Categories)** | [http://localhost:8080/api/categories](http://localhost:8080/api/categories) | Lấy danh mục, kiểm tra kết nối DB và migration Flyway |
| **MySQL Database** | `localhost:3307` (nội bộ Compose: `mysqldb:3306`) | Cơ sở dữ liệu MySQL |

Lệnh kiểm tra nhanh kết quả Health Check qua terminal:
```bash
curl http://localhost:8080/actuator/health
```

Kết quả mong đợi:
```json
{"status":"UP","groups":["liveness","readiness"]}
```

---

## 4. Minh chứng kết quả thực hiện

### 4.1. Gradle test chạy thành công
```text
> Task :compileJava
> Task :processResources
> Task :classes
> Task :compileTestJava
> Task :processTestResources
> Task :testClasses
> Task :test

BUILD SUCCESSFUL in 19s
5 actionable tasks: 5 executed
```

### 4.2. Docker Image đóng gói thành công
```text
REPOSITORY           TAG       IMAGE ID       CREATED          SIZE
campus-hub-backend   latest    faad252d3fe6   Minutes ago      186MB
```

### 4.3. Docker Compose khởi chạy các service
```text
NAME                 IMAGE                       COMMAND                  SERVICE   STATUS                    PORTS
campus-hub-backend   campus-hub-backend:latest   "java -jar app.jar"      backend   Up (healthy/running)      0.0.0.0:8080->8080/tcp
campus-hub-mysql     mysql:8.0                   "docker-entrypoint.s…"   mysqldb   Up (healthy)              0.0.0.0:3307->3306/tcp
```

### 4.4. Gọi Actuator Health Endpoint thành công
```http
HTTP/1.1 200 
Content-Type: application/vnd.spring-boot.actuator.v3+json

{"status":"UP","groups":["liveness","readiness"]}
```

---

## 5. Cấu trúc Dockerfile & Docker Compose

- **[backend/Dockerfile](backend/Dockerfile)**: Áp dụng kỹ thuật **Multi-stage build**:
  - `Stage 1 (Builder)`: Sử dụng `gradle:8.14-jdk21-alpine` để tải dependencies và biên dịch ứng dụng (`bootJar`) mà không cần phụ thuộc môi trường máy host.
  - `Stage 2 (Runtime)`: Sử dụng `eclipse-temurin:21-jre-alpine` gọn nhẹ, tạo user `appuser` (non-root) để tăng cường bảo mật container.
- **[backend/.dockerignore](backend/.dockerignore)**: Loại bỏ các file tạm, cache Gradle (`.gradle`, `build`), file IDE (`.idea`), log và git để giảm kích thước build context.
- **[docker-compose.yml](docker-compose.yml)**: Khởi tạo 2 services:
  - `mysqldb`: MySQL 8.0 với volume `mysql_data`, thiết lập cấu hình `healthcheck` ping `mysqladmin`.
  - `backend`: Kết nối tới MySQL qua chuỗi kết nối sử dụng hostname `mysqldb`, ràng buộc `depends_on` với điều kiện `service_healthy`.
