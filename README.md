# ttcs_genaral

Đây là khung local dùng chung cho các service `be_service_ttcs`, `qr_service_be_ttcs`, `fe_ttcs` và `api_gateway`.

## Mục tiêu

- Dùng một stack local thống nhất để test giao tiếp giữa các service.
- Dùng chung `RabbitMQ`, `Redis`, `MongoDB` theo `service name` của Docker Compose, không nối thủ công bằng `external network` hay `container_name`.
- Tách `qr_service` API khỏi worker để mô hình local gần với cách deploy trên K8S hơn.

## Thành phần

- `gateway`: Caddy làm cổng vào chung cho local, đọc cấu hình từ `../api_gateway/Caddyfile`.
- `frontend`: build từ `../fe_ttcs`.
- `be-service-api`: API chính của backend.
- `be-service-scheduler`: tiến trình chạy tác vụ định kỳ riêng.
- `qr-service-api`: API của QR service.
- `qr-sync-worker`: worker đồng bộ dữ liệu đăng ký qua RabbitMQ.
- `qr-attendance-worker`: worker xử lý luồng check-in qua RabbitMQ.
- `mongodb`, `redis`, `rabbitmq`: các dịch vụ nền tảng dùng chung.

## Tên miền local

- `http://localhost`: frontend, đồng thời route `/api/*` vào `be-service-api` và `/qr/*` vào `qr-service-api`.
- `http://api.localhost`: gọi trực tiếp `be-service-api` để debug service.
- `http://qr.localhost`: gọi trực tiếp `qr-service-api` để debug service.
- `http://rabbitmq.localhost`: giao diện quản trị RabbitMQ.

## Quy ước route qua gateway

- Frontend public: `http://localhost`
- API chính của backend: `http://localhost/api/*`
- API của QR service: `http://localhost/qr/*`
- Bên trong `qr_service`, đường dẫn vẫn là `/api/*`; gateway sẽ rewrite từ `/qr/*` thành `/api/*` trước khi forward.

Ví dụ:

```text
GET http://localhost/api/health  -> be-service-api /api/health
GET http://localhost/qr/health   -> qr-service-api /api/health
GET http://localhost/qr/docs     -> qr-service-api /api/docs
```

## Vai trò của gateway

- Caddy chỉ đóng vai trò `edge gateway` và `reverse proxy`.
- Gateway xử lý route request, forward header, nén `gzip` và `zstd`, access log và timeout cơ bản.
- JWT và RBAC không đặt ở gateway; mỗi service phải tự verify token và tự kiểm tra quyền.
- Các service nội bộ giao tiếp với nhau qua internal DNS, Redis và RabbitMQ; không đưa business logic vào gateway.
- Gateway được quản lý riêng trong [api_gateway/Caddyfile](/d:/TTCS/api_gateway/Caddyfile), còn `ttcs_genaral` chỉ chịu trách nhiệm orchestration cho stack local.

## Cách dùng

1. Tạo file `.env.local` trong thư mục này dựa trên `.env.local.example`.
2. Chạy lệnh:

```powershell
docker compose -f ttcs_genaral/docker-compose.local.yml --env-file ttcs_genaral/.env.local up --build
```

3. Kiểm tra nhanh:

```powershell
curl http://api.localhost/api/health
curl http://qr.localhost/api/health
curl http://localhost/qr/health
```

## Nguyên tắc của khung

- Mọi service nội bộ kết nối với nhau bằng `service name` của Compose: `mongodb`, `redis`, `rabbitmq`.
- Không dùng `container_name`.
- Không để worker chạy ẩn trong cùng process với API.
- Mỗi microservice sở hữu database logic riêng:
  `BE_MONGO_DATABASE` cho backend và `QR_MONGO_DATABASE` cho QR service.
