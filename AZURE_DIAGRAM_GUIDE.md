# 📐 Hướng Dẫn Đọc Diagram — Tourism Azure Style

> **File diagram:** [`tourism_azure_complete.drawio`](./tourism_azure_complete.drawio)  
> **Mở bằng:** draw.io Desktop hoặc [app.diagrams.net](https://app.diagrams.net)

---

## 1. Bố Cục Tổng Thể

```
[User] → [Future Travel App] → [API Gateway :8080]
                                        │
                              ┌─────────▼─────────────────────────────┐
                              │     System Management Module           │
                              │                                        │
                              │  ROW 1: IAM | Catalog | Booking | Pay  │
                              │             ──────bus1──────           │
                              │  ROW 2: Review | Promotion | CMS       │
                              │             ──────bus2──────    [Kafka]│
                              │  ROW 3: Notification | Analytics [RT]  │
                              │                                        │
                              │  Redis · Zipkin · Shared Libraries     │
                              └────────────────────────────────────────┘
```

---

## 2. Ý Nghĩa Các Icon

| Icon | Nguồn | Ý nghĩa |
|------|-------|---------|
| ![Azure App Services](img/lib/azure2/app_services/App_Services.svg) App Services | Azure | Microservice instance (Spring Boot) |
| ![Azure PostgreSQL](img/lib/azure2/databases/Azure_Database_PostgreSQL_Server.svg) PostgreSQL | Azure | Database riêng của từng service |
| ![SignalR](img/lib/azure2/web/SignalR.svg) SignalR | Azure | Real-time Notify (WebSocket) |
| 🟠 AMQP shape | Alibaba | Apache Kafka (Message Broker) |
| 🌐 Router icon | Cisco | API Gateway (Spring Cloud Gateway) |
| 👤 User shape | Office | End User |
| 📱 Device shape | Azure | Frontend App (React + Vite) |

---

## 3. Loại Đường Kẻ (Edges)

| Màu | Loại | Ý nghĩa |
|-----|------|---------|
| **Xanh dương** `#1ba1e2` ╌╌►  animated | REST routing | API Gateway → Service `:8080` |
| **Xanh lá** `#2E7D32` ╌╌► animated | Feign internal | Service A gọi Service B đồng bộ |
| **Cam** `#FF6A00` ╌╌► animated | Kafka Event | Publish/Consume bất đồng bộ |
| **Xám** `#9e9e9e` - - - | External | Kết nối ra ngoài hệ thống |

### Bus Line Pattern

Các đường kẻ từ API Gateway đến service trong cùng 1 row đều đi qua một **điểm waypoint chung** tạo hiệu ứng "đường trục" (bus line) ngang trong mỗi row. Draw.io vẽ chúng trên cùng tuyến ngang trước khi rẽ vào service.

---

## 4. Danh Sách Service & Kết Nối

### 4.1 IAM Service `:8081`
- **DB:** `tourism_identity` (PostgreSQL)
- **Nhận:** `GET/POST /api/auth/**` từ API Gateway
- **Publish Kafka:** `user.registered`
- **External:** Cloudinary (avatar), SMTP (verify email), Google OAuth2

### 4.2 Tour Catalog Service `:8082`
- **DB:** `tourism_catalog`
- **Nhận:** `GET /api/tours/**` từ API Gateway
- **Nhận Feign:** Booking gọi `GET /internal/tours/{id}` (kiểm tra tour, lấy giá)

### 4.3 Booking Service `:8083`
- **DB:** `tourism_booking` + Redis Cache
- **Nhận:** `POST/GET /api/bookings/**` từ API Gateway
- **Gọi Feign →** TourCatalog: kiểm tra tour còn slot
- **Gọi Feign →** Promotion: apply coupon code
- **Nhận Feign ←** Payment: confirm booking sau thanh toán
- **Nhận Feign ←** Review: verify booking đã COMPLETED
- **Publish Kafka:** `booking.created`, `booking.confirmed`, `booking.cancelled`

### 4.4 Payment Service `:8084`
- **DB:** `tourism_payment`
- **Nhận:** `POST /api/payments/**` từ API Gateway
- **Gọi Feign →** Booking: confirm sau khi thanh toán xong
- **Publish Kafka:** `payment.completed`
- **External:** VNPay, PayOS, SePay

### 4.5 Review Service `:8085`
- **DB:** `tourism_review`
- **Nhận:** `POST /api/reviews/**` từ API Gateway
- **Gọi Feign →** Booking: kiểm tra booking đã COMPLETED mới cho review
- **Publish Kafka:** `review.created`

### 4.6 Promotion Service `:8086`
- **DB:** `tourism_promotion`
- **Nhận:** `GET /api/promotions/**` từ API Gateway
- **Cung cấp Feign:** `POST /internal/coupons/{code}/apply` cho Booking

### 4.7 CMS Service `:8089`
- **DB:** `tourism_cms`
- **Nhận:** `GET/POST /api/cms/**` từ API Gateway
- Blog, banner, policy, chi nhánh

### 4.8 Notification Service `:8087`
- **DB:** `tourism_notification`  
- **Nhận:** `GET /api/notifications/**` từ API Gateway
- **Consume Kafka:** tất cả events → gửi email qua SMTP Gmail

### 4.9 Analytics + AI Service `:8088`
- **DB:** `tourism_analytics`
- **Nhận:** `/api/analytics/**` từ API Gateway
- **Consume Kafka:** tất cả events → DailyStats
- **External:** Google Gemini AI (chatbot trợ lý)

---

## 5. Kafka — Luồng Event Chi Tiết

```
PRODUCER ──────────────► TOPIC ─────────────────────► CONSUMERS
IAM       user.registered    ──► Notification (welcome email)
                               ──► Analytics (user count)

Booking   booking.created    ──► Notification (email xác nhận)
          booking.confirmed  ──► Notification (email confirmed)
          booking.cancelled  ──► Notification (email huỷ)

Payment   payment.completed  ──► Notification (email receipt)
                               ──► Analytics (revenue stats)

Review    review.created     ──► Analytics (update avg rating)

ALL events ─────────────────► Real-time Notify (WebSocket push)
```

---

## 6. Hạ Tầng Chung

| Thành phần | Port | Mục đích |
|-----------|------|---------|
| **Eureka Server** | `:8761` | Service discovery — GW tự tìm địa chỉ service |
| **Redis Cache** | `:6379` | Booking cache pricing từ catalog |
| **Zipkin** | `:9411` | Distributed tracing — theo dõi latency |
| **Real-time Notify** | — | WebSocket push notifications lên browser |
| **Apache Kafka** | `:9092` | Event bus bất đồng bộ |

---

## 7. Shared Libraries (Maven modules)

```
common-security
├── JwtAuthenticationFilter   ← tất cả services import
└── UserPrincipal             ← đối tượng user sau khi auth

common-events
├── UserRegisteredEvent
├── BookingCreatedEvent / BookingConfirmedEvent / BookingCancelledEvent
├── PaymentCompletedEvent
└── ReviewCreatedEvent
```

---

## 8. Cách Bật Animation Trong Draw.io

1. Mở file trong **draw.io Desktop** hoặc **app.diagrams.net**
2. Menu **View → Tooltips** ✅
3. Menu **View → Animate** ✅ — các đường kẻ nét đứt sẽ chuyển động
4. Hoặc: chuột phải vào edge → **Edit Style** → đảm bảo có `flowAnimation=1`

> ⚠️ Animation **không hoạt động** trong VS Code draw.io extension preview.

---

*File này giải thích `tourism_azure_complete.drawio` — diagram kiến trúc Tourism Microservices Platform với Azure icon style.*
