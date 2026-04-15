# 🏗️ Tourism Microservices — Tài Liệu Tổng Quan

> **Dự án:** Hệ thống quản lý tour du lịch — Chuyển đổi Monolith → Microservices  
> **Phiên bản:** v4 (5 Business Services)  
> **Thời gian thực hiện:** 1 tháng (28 ngày)  
> **Stack:** Java 17 · Spring Boot 3.3 · Spring Cloud 2023.0 · React 18 · TypeScript · PostgreSQL · Docker Compose

---

## 1. Bối Cảnh & Tại Sao Chọn 5 Services?

### Monolith hiện tại
- **Source:** `D:\KLTN\Tourism_Backend` — Spring Boot monolith đầy đủ tính năng
- **23 Entities**, 22 Controllers, ~30 Services, tích hợp: VNPay, PayOS, Sepay, Cloudinary, Gemini AI, Pinecone, Google OAuth2

### Vấn đề khi tách
| Tiêu chí | Ít service (3-4) | Nhiều service (8-10) |
|---|---|---|
| Thời gian code | ✅ Nhanh | ❌ Không kịp 1 tháng |
| Đúng microservices | ⚠️ Mờ ranh giới | ✅ Chuẩn |
| Độ phức tạp infra | ✅ Đơn giản | ❌ Kafka, nhiều DB |
| Phù hợp KLTN | ✅ | ✅ |

### Quyết định chọn **5 Services**

> **Nguyên tắc:** Mỗi service = 1 Bounded Context rõ ràng + DB riêng + triển khai độc lập.  
> Không dùng Kafka — giao tiếp qua **REST/Feign** để đơn giản, dễ debug.

| # | Service | Port | DB | Bounded Context |
|---|---|---|---|---|
| 1 | `api-gateway` | 8080 | — | Routing, JWT verify, Load Balance |
| 2 | `service-registry` | 8761 | — | Eureka Discovery |
| 3 | **`iam-service`** | 8081 | `tourism_iam` | Authentication & Authorization |
| 4 | **`user-service`** | 8085 | `tourism_user` | User Profile, Favorite, Notification |
| 5 | **`tour-service`** | 8082 | `tourism_tour` | Tour, Location, Review, Chatbot AI |
| 6 | **`booking-service`** | 8083 | `tourism_booking` | Booking, Dashboard Analytics |
| 7 | **`payment-service`** | 8084 | `tourism_payment` | VNPay, PayOS, Sepay |

---

## 2. Sơ Đồ Kiến Trúc

```
                          ┌──────────────────────────────────────┐
[Browser/Mobile]          │          API GATEWAY :8080           │
     │                    │  - JWT Verify (Spring Security)       │
     └──── HTTP ─────────►│  - Route → service-registry          │
                          │  - Rate Limit, CORS, Logging          │
                          └────────────┬─────────────────────────┘
                                       │ Route by path prefix
               ┌───────────┬───────────┼───────────┬──────────────┐
               ▼           ▼           ▼           ▼              ▼
         ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
         │iam-svc   │ │user-svc  │ │tour-svc  │ │booking   │ │payment   │
         │:8081     │ │:8085     │ │:8082     │ │svc :8083 │ │svc :8084 │
         │          │ │          │ │+Chatbot  │ │+Dashboard│ │          │
         │tourism   │ │tourism   │ │tourism   │ │tourism   │ │tourism   │
         │_iam (PG) │ │_user(PG) │ │_tour(PG) │ │_bk (PG)  │ │_pay (PG) │
         └─────┬────┘ └─────┬────┘ └─────┬────┘ └─────┬────┘ └─────┬────┘
               │             │             │             │             │
               └─────────────┴─────────────┴─────────────┴─────────────┘
                                    Feign Client (HTTP/REST)

         ┌──────────────────────────────────────────────────────────┐
         │              Service Registry (Eureka) :8761              │
         └──────────────────────────────────────────────────────────┘

         ┌──────────────────────────────────────────────────────────┐
         │                   External Services                       │
         │  Gemini AI (Embedding+Generation)  Pinecone (VectorDB)   │
         │  Cloudinary (Image/Video CDN)      VNPay / PayOS / Sepay │
         │  Google OAuth2                     SMTP (Email)           │
         └──────────────────────────────────────────────────────────┘
```

---

## 3. Phân Chia Entity Theo Service

### 🔐 iam-service — `tourism_iam`
| Entity | Mô tả |
|---|---|
| `RefreshToken` | Lưu refresh token, `userId` là `INT` (không FK cross-service) |

**Không có `User` trong IAM** — chỉ biết `userId` (Integer). User data nằm ở `user-service`.

---

### 👤 user-service — `tourism_user`
| Entity | Thay đổi từ monolith |
|---|---|
| `User` | Xóa relationship `bookings`, `reviews` (cross-service) |
| `FavoriteTour` | `tour` → `Integer tourId` (no FK) |
| `Notification` | Giữ nguyên |
| `UserNotification` | Giữ nguyên |

---

### 🗺️ tour-service — `tourism_tour`
| Entity | Thay đổi từ monolith |
|---|---|
| `Tour` | Giữ nguyên |
| `TourImage` | Giữ nguyên |
| `TourMedia` | Giữ nguyên |
| `ItineraryDay` | Giữ nguyên |
| `Location` | Giữ nguyên |
| `TourDeparture` | Xóa relationship `bookings` (cross-service) |
| `DeparturePricing` | Giữ nguyên |
| `DepartureTransport` | Giữ nguyên |
| `PolicyTemplate` | Giữ nguyên |
| `BranchContact` | Giữ nguyên |
| `Coupon` | Giữ nguyên |
| `Review` | `userId` → `INT`, `bookingId` → `INT` (no FK) |
| `ImageReview` | Giữ nguyên |

---

### 📋 booking-service — `tourism_booking`
| Entity | Thay đổi từ monolith |
|---|---|
| `Booking` | `user` → `Integer userId`; `tourDeparture` → `Integer departureId`; xóa `payment` |
| `BookingPassenger` | Giữ nguyên |
| `RefundInformation` | Giữ nguyên |

---

### 💳 payment-service — `tourism_payment`
| Entity | Thay đổi từ monolith |
|---|---|
| `Payment` | `booking` → `Integer bookingId` (no FK) |

---

## 4. Tại Sao Chatbot AI Nằm Trong `tour-service`?

```
Chatbot cần truy cập trực tiếp:
  ✅ TourRepository        — tìm tour theo keyword
  ✅ TourDepartureRepository — lấy slot còn trống, giá
  ✅ LocationRepository    — thông tin địa điểm
  ✅ CouponRepository      — kiểm tra coupon hợp lệ
  ✅ ReviewRepository      — đánh giá trung bình tour

Nếu tách AI thành service riêng:
  ❌ Cần 5 Feign calls cho mỗi yêu cầu chat
  ❌ Thêm 1 service, 1 DB, thêm infra
  ❌ Latency tăng đáng kể
```

**→ Đặt trong `tour-service` là lựa chọn tối ưu.**

---

## 5. Tại Sao Dashboard Nằm Trong `booking-service`?

Dashboard phân tích cần:
- **Doanh thu theo ngày/tháng** → query `Payment` (cross-service call từ booking)
- **Số booking theo trạng thái** → query `Booking` (có ngay trong booking-service)
- **Tour bán chạy** → query `Booking` groupBy `departureId` → gọi Feign lấy tên tour

> **Kết luận:** `DashboardController` + `DashboardService` đặt trong `booking-service`, gọi Feign sang `payment-service` lấy revenue khi cần. Frontend tổng hợp từ 2 API nếu cần.

---

## 6. Feign Client Map (Giao Tiếp Giữa Services)

```
iam-service ──────────────► user-service
  register/login               /internal/users  (tìm/tạo user)

booking-service ─────────► tour-service
  tạo booking                  /internal/departures/{id}  (kiểm tra slot, giá)
  apply coupon                 /internal/coupons/validate

booking-service ─────────► user-service
  trừ coin                     /internal/users/{id}/coin
  gửi notification             /internal/notifications

booking-service ─────────► payment-service
  tạo payment entry            /internal/payments

payment-service ─────────► booking-service
  confirm thanh toán           /internal/bookings/{id}/confirm-paid

user-service ────────────► tour-service
  hiển thị tên tour yêu thích  /internal/tours/{id}
```

> **Lưu ý:** Các endpoint `/internal/**` KHÔNG qua API Gateway, chỉ nội bộ Eureka.

---

## 7. Database Strategy

| Service | DB Name | Schema chính |
|---|---|---|
| iam-service | `tourism_iam` | `refresh_tokens` |
| user-service | `tourism_user` | `users`, `favorite_tours`, `notifications`, `user_notifications` |
| tour-service | `tourism_tour` | `tours`, `tour_departures`, `locations`, `coupons`, `reviews`, ... |
| booking-service | `tourism_booking` | `bookings`, `booking_passengers`, `refund_informations` |
| payment-service | `tourism_payment` | `payments` |

**Không có foreign key cross-database** — tính nhất quán đảm bảo qua application logic (Feign validate trước khi ghi).

---

## 8. Frontend Strategy

> **Giữ nguyên `D:\KLTN\client-side` (React 18 + Vite + TypeScript)**  
> Chỉ thay đổi duy nhất: `VITE_API_URL` trỏ về `API Gateway :8080`

```
TRƯỚC (monolith):   http://localhost:8088/api/**
SAU (microservices): http://localhost:8080/api/**
```

**Không cần rebuild frontend từ đầu.** API Gateway định tuyến trong suốt:

| Prefix URL | → Service |
|---|---|
| `/api/auth/**` | iam-service |
| `/api/admin/auth/**` | iam-service |
| `/api/users/**` | user-service |
| `/api/favorites/**` | user-service |
| `/api/notifications/**` | user-service |
| `/api/tours/**` | tour-service |
| `/api/locations/**` | tour-service |
| `/api/reviews/**` | tour-service |
| `/api/chatbot/**` | tour-service |
| `/api/bookings/**` | booking-service |
| `/api/admin/dashboard/**` | booking-service |
| `/api/payments/**` | payment-service |

---

## 9. Infrastructure Stack

```yaml
# docker-compose.yml (tóm tắt)
services:
  postgres:      # 1 PostgreSQL instance, 5 databases riêng
  service-registry:  # Eureka :8761
  api-gateway:       # Spring Cloud Gateway :8080
  iam-service:       # :8081
  user-service:      # :8085
  tour-service:      # :8082
  booking-service:   # :8083
  payment-service:   # :8084
```

> Không dùng Redis, không dùng Kafka → giảm complexity tối đa trong 1 tháng.

---

## 10. Kế Hoạch 1 Tháng (28 Ngày)

### 📅 Tuần 1 (Ngày 1–7): Foundation + iam-service + user-service

| Ngày | Việc làm |
|---|---|
| 1 | Tạo project structure: parent `pom.xml`, tất cả module skeleton, `docker-compose.yml` |
| 2 | `service-registry` (Eureka) + `api-gateway` (Spring Cloud Gateway + JWT filter) |
| 3 | `iam-service`: Entity `RefreshToken`, Auth endpoints (register/login/logout/refresh) |
| 4 | `iam-service`: Google OAuth2, email verify, JWT generation, Feign call → `user-service` |
| 5 | `user-service`: Entity `User`, CRUD profile, avatar Cloudinary, coin balance |
| 6 | `user-service`: `FavoriteTour`, `Notification`, `/internal/**` endpoints |
| 7 | Integration test: đăng ký → login → xem profile qua Gateway. Fix bugs. |

### 📅 Tuần 2 (Ngày 8–14): tour-service

| Ngày | Việc làm |
|---|---|
| 8 | `tour-service`: Entity Tour, Location. Port `TourServiceImpl`, `LocationServiceImpl` |
| 9 | `TourDeparture`, `DeparturePricing`, `DepartureTransport`. Port departure services |
| 10 | `TourImage`, `TourMedia`, Cloudinary upload. Port `TourManagementServiceImpl` |
| 11 | `ItineraryDay`, `PolicyTemplate`, `BranchContact`, `Coupon` |
| 12 | `Review`, `ImageReview`. Port `ReviewServiceImpl`. Review flow sau booking |
| 13 | **Chatbot AI**: Port `ChatbotService`, `VectorService`, `VectorSyncService` (Gemini + Pinecone) |
| 14 | `/internal/**` endpoints cho booking-service. Test toàn bộ tour-service |

### 📅 Tuần 3 (Ngày 15–21): booking-service + payment-service

| Ngày | Việc làm |
|---|---|
| 15 | `booking-service`: Entity `Booking`, `BookingPassenger`, Port `BookingServiceImpl` |
| 16 | Booking flow: validate slot (Feign → tour), validate coupon, trừ coin (Feign → user) |
| 17 | `RefundInformation`, hủy booking, `BookingCleanupService` (scheduled) |
| 18 | Notification sau booking (Feign → user-service push notification + email) |
| 19 | **Dashboard**: `DashboardService` — doanh thu, booking stats, top tours |
| 20 | `payment-service`: Port `PaymentServiceImpl` (VNPay), `PayOSService`, `SepayServiceImpl` |
| 21 | Payment callback → Feign → booking-service confirm. End-to-end booking+payment test |

### 📅 Tuần 4 (Ngày 22–28): Frontend + Deploy + Polish

| Ngày | Việc làm |
|---|---|
| 22 | Update `client-side`: đổi `VITE_API_URL` → Gateway. Test login, xem tour |
| 23 | Test frontend: booking flow, payment redirect, hiển thị notification |
| 24 | Test frontend: chatbot, dashboard admin, review |
| 25 | Viết `docker-compose.yml` đầy đủ + `.env` template. Build Docker images |
| 26 | Deploy lên VPS/local: smoke test toàn bộ flow. Fix production bugs |
| 27 | Performance: kiểm tra slow Feign calls. Tối ưu query N+1 nếu có |
| 28 | Viết báo cáo KLTN section Architecture. Chụp demo. Buffer |

---

## 11. Quyết Định Kiến Trúc Quan Trọng

| Quyết định | Lựa chọn | Lý do |
|---|---|---|
| Số services | 5 | Đủ micro, đủ bounded context, khả thi 1 tháng |
| Messaging | REST/Feign (không Kafka) | Đơn giản, dễ debug, phù hợp KLTN |
| Database | 1 PostgreSQL, 5 schema/DB | Dễ quản lý local/VPS, không cần nhiều port |
| Chatbot AI | Trong `tour-service` | Truy cập trực tiếp Tour/Departure/Location repositories |
| Dashboard | Trong `booking-service` | Truy cập trực tiếp Booking data |
| Notification | Trong `user-service` | User-centric data, push từ services khác qua Feign |
| Frontend | Giữ `client-side`, chỉ đổi URL | Không mất thời gian rebuild |
| Auth | JWT stateless | Mỗi service tự verify JWT, không cần gọi iam-service mỗi request |
| Deploy | Docker Compose | Phù hợp KLTN, 1 lệnh `docker-compose up` |

---

## 12. Tài Liệu Liên Quan

| File | Mô tả |
|---|---|
| [`DETAIL.md`](./DETAIL.md) | Entity mapping chi tiết, API endpoints, cấu hình docker-compose |
| [`entity_diagram_microservices.drawio`](./entity_diagram_microservices.drawio) | Class diagram phân màu theo service |
| [`architecture_detail.md`](./architecture_detail.md) | Tài liệu chi tiết cũ (tham khảo) |
