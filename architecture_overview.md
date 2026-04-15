# 🏗️ Tourism Microservices — Tài Liệu Tổng Quan Kiến Trúc

> **Dự án:** Hệ thống quản lý tour du lịch — chuyển từ Monolith sang Microservices  
> **Phiên bản:** v3 (4 Business Services)  
> **Thời gian:** 1 tháng (28 ngày)  
> **Stack:** Java 17 · Spring Boot 3.3 · Spring Cloud 2023.0 · React 18 · TypeScript · PostgreSQL · Docker

---

## 1. Tổng Quan Kiến Trúc

### Tại Sao Chọn 4 Services?

| Phương án | Số services | Khả thi 1 tháng | Đủ cho KLTN |
|---|---|---|---|
| tourism-microservices (v1) | 9 services | ❌ | ✅ |
| tourism-microservices-v2 | 10 services | ❌ | ✅ |
| **Phương án v3 (đề xuất)** | **4 services** | ✅ | ✅ |

**Nguyên tắc:** *Mỗi service = 1 Bounded Context rõ ràng, tách DB riêng, giao tiếp qua API Gateway.*

---

## 2. Sơ Đồ Kiến Trúc Tổng Thể

```
┌──────────────────────────────────────────────────────────┐
│                    CLIENT SIDE (React)                    │
│         http://localhost:3000  |  D:\KLTN\client-side     │
└──────────────────────┬───────────────────────────────────┘
                       │ HTTP/REST
                       ▼
┌──────────────────────────────────────────────────────────┐
│               API GATEWAY  :8080                         │
│     Spring Cloud Gateway + JWT Filter + CORS             │
└──┬─────────────┬────────────────┬───────────────┬────────┘
   │             │                │               │
   ▼             ▼                ▼               ▼
┌──────┐   ┌──────────┐   ┌────────────┐   ┌──────────┐
│identity│  │  tour    │   │  booking   │   │ payment  │
│service │  │ service  │   │  service   │   │ service  │
│ :8081  │  │  :8082   │   │   :8083    │   │  :8084   │
│        │  │          │   │            │   │          │
│ DB:    │  │ DB:      │   │ DB:        │   │ DB:      │
│tourism │  │tourism   │   │tourism     │   │tourism   │
│_identity│ │_tour     │   │_booking    │   │_payment  │
└──────┘   └──────────┘   └────────────┘   └──────────┘
     │           │               │               │
     └─────────────────┬─────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│           SERVICE REGISTRY (Eureka)  :8761              │
└─────────────────────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────────────────────┐
│              EXTERNAL SERVICES                           │
│  Gemini AI (Chatbot + Embedding)  │  Pinecone (Vector)  │
│  Cloudinary (Media)  │  VNPay  │  PayOS  │  Sepay       │
│  Gmail SMTP  │  Google OAuth2                           │
└─────────────────────────────────────────────────────────┘
```

---

## 3. Phân Chia 4 Services

### 3.1 Identity Service `:8081` — Xác thực & Người dùng

**Bounded Context:** Quản lý danh tính người dùng, phiên đăng nhập, phân quyền.

**Entities sở hữu:**
- `users` — thông tin tài khoản, profile, coin balance
- `refresh_tokens` — JWT refresh token management

**Chức năng chính:**
- Đăng ký / Đăng nhập email + password
- Google OAuth2 (Single Sign-On)
- Xác thực email (verification token)
- JWT access token + refresh token
- Quản lý profile người dùng (avatar Cloudinary)
- Admin: quản lý danh sách user, block/unblock

**External integrations:** Gmail SMTP · Google OAuth2 · Cloudinary

---

### 3.2 Tour Service `:8082` — Tour + Chatbot AI + Dashboard

**Bounded Context:** Quản lý sản phẩm tour, chatbot tư vấn, thống kê.

**Entities sở hữu:**
- `tours` — thông tin tour
- `tour_departures` — lịch khởi hành
- `departure_pricing` — bảng giá theo loại khách
- `departure_transports` — thông tin phương tiện
- `itinerary_days` — lịch trình chi tiết
- `tour_images` — ảnh tour
- `tour_media` — video tour
- `locations` — địa điểm du lịch
- `policy_templates` — template chính sách hủy tour
- `branch_contacts` — thông tin chi nhánh
- `coupons` — mã giảm giá (gắn với departure)
- `reviews` — đánh giá sau chuyến đi
- `image_reviews` — ảnh trong đánh giá
- `favorite_tours` — tour yêu thích

**Chức năng chính:**
- CRUD tour & quản lý lịch khởi hành (Admin)
- Tìm kiếm, lọc tour (user)
- Upload ảnh/video Cloudinary
- Chatbot AI: RAG với Gemini + Pinecone
- Dashboard analytics (query từ DB riêng của booking-service qua REST)

> **Lý do Chatbot ở đây:** ChatbotService query trực tiếp TourRepository, TourDepartureRepository, ReviewRepository, CouponRepository, LocationRepository — tất cả đều nằm ở tour-service. Tách ra sẽ cần 5 Feign call/request.

**External integrations:** Cloudinary · Gemini API · Pinecone

---

### 3.3 Booking Service `:8083` — Đặt Tour & Thông Báo

**Bounded Context:** Quản lý vòng đời đặt tour, coupon, thông báo nội bộ, dashboard admin.

**Entities sở hữu:**
- `bookings` — đơn đặt tour
- `booking_passengers` — thông tin từng hành khách
- `refund_informations` — thông tin hoàn tiền
- `notifications` — thông báo hệ thống
- `user_notifications` — trạng thái đọc của từng user

**Chức năng chính:**
- Tạo booking từ TourDeparture (Feign call sang tour-service kiểm tra slot)
- Giải phóng slot khi booking cancelled/expired
- Lấy thông tin booking + passenger
- Gửi email xác nhận booking (Gmail SMTP)
- Thông báo in-app (Notification)
- Dashboard Admin: doanh thu theo ngày/tuần/tháng, booking stats

**External integrations:** Gmail SMTP · Feign → tour-service · Feign → payment-service

---

### 3.4 Payment Service `:8084` — Thanh Toán

**Bounded Context:** Xử lý giao dịch thanh toán qua các cổng.

**Entities sở hữu:**
- `payments` — lịch sử giao dịch thanh toán

**Chức năng chính:**
- Tạo link thanh toán VNPay
- Xử lý PayOS (QR + link)
- Đối chiếu giao dịch Sepay (webhook)
- Callback handler (public endpoints không cần JWT)
- Cập nhật trạng thái booking sau thanh toán (Feign → booking-service)

**External integrations:** VNPay · PayOS · Sepay · Feign → booking-service

---

## 4. Giao Tiếp Giữa Services

```
booking-service ──Feign──► tour-service       (kiểm tra slot, lấy tour info)
booking-service ──Feign──► payment-service    (kiểm tra trạng thái payment)
payment-service ──Feign──► booking-service    (cập nhật booking sau payment)
tour-service    ──Feign──► booking-service    (dashboard: lấy booking stats)
```

**Không dùng Kafka** — REST/Feign đồng bộ đủ cho KLTN, infra đơn giản hơn.

### JWT Propagation

```
Frontend → [Bearer token] → API Gateway → [X-User-Id, X-User-Role header] → Services
```

API Gateway xác thực JWT, extract userId + role rồi forward qua header. Các services không cần xác thực lại JWT.

---

## 5. Database Strategy

| Service | Database | Số bảng chính |
|---|---|---|
| identity-service | `tourism_identity` | 2 |
| tour-service | `tourism_tour` | 14 |
| booking-service | `tourism_booking` | 5 |
| payment-service | `tourism_payment` | 1 |

**Shared database**: Không — mỗi service có DB riêng.  
**Cross-service ID**: Lưu dạng `Long userId` (không có FK cross-service).

---

## 6. Infrastructure Stack

| Component | Technology | Port |
|---|---|---|
| API Gateway | Spring Cloud Gateway | 8080 |
| Service Registry | Netflix Eureka | 8761 |
| Database | PostgreSQL 15 | 5432 |
| Cache (optional) | Redis 7 | 6379 |
| Container | Docker + Docker Compose | — |

---

## 7. Frontend Architecture

**Project:** `D:\KLTN\client-side` — React 18 + TypeScript  

**Kết nối:** Toàn bộ API call qua `API_URL = http://localhost:8080` (API Gateway)

**Các trang chính (hiện có):**
- `/` — Trang chủ
- `/tours` — Danh sách tour + tìm kiếm
- `/tour/:tourCode` — Chi tiết tour
- `/order-booking` — Form đặt tour
- `/payment-booking` — Thanh toán
- `/payment-success`, `/payment-failed`, `/payment-waiting` — Kết quả thanh toán
- `/register`, `/login`, `/verify-email` — Xác thực
- `/information/:tab` — Thông tin chung / tự tư vấn
- `/admin/*` — Dashboard quản trị

**Thay đổi cần thiết:** Cập nhật `services/` API URLs từ `:8080` (monolith) → `:8080` (gateway, cùng port, không đổi)

---

## 8. Chatbot AI — Quyết Định Kiến Trúc

### Kiến trúc RAG (Retrieval-Augmented Generation)

```
User nhắn tin
    │
    ▼ (1) Tạo vector embedding câu hỏi
Gemini API (text-embedding-004)
    │
    ▼ (2) Tìm kiếm semantic
Pinecone (vector DB cloud)
    │
    ▼ (3) Build context (Tour + Departure + Coupon + Review + Location)
    │
    ▼ (4) Gọi Gemini Flash sinh trả lời
Gemini API (gemini-2.0-flash)
    │
    ▼ (5) Trả về reply + TourSuggestions + QuickActions
Frontend ChatbotWidget
```

### Tại Sao KHÔNG tách Chatbot thành service riêng?

| Dependency | Nằm ở | Nếu tách |
|---|---|---|
| TourRepository | tour-service | Feign call #1 |
| TourDepartureRepository | tour-service | Feign call #2 |
| ReviewRepository | tour-service | Feign call #3 |
| CouponRepository | tour-service | Feign call #4 |
| LocationRepository | tour-service | Feign call #5 |

→ **Mỗi chat request cần 5 Feign calls** → latency cao, code phức tạp.  
→ **Giữ trong tour-service** = query DB trực tiếp, latency thấp, code đơn giản.

### Vector Sync Strategy

```
tour-service/VectorSyncService:
- @Scheduled(cron = "0 0 2 * * *") — tự động 2AM hàng ngày
- POST /api/chatbot/sync — admin trigger thủ công
- Hook sau khi tạo/sửa tour: gọi syncTourSummary()
```

---

## 9. Dashboard Analytics — Quyết Định Kiến Trúc

### KHÔNG tách thành analytics-service riêng vì:

Dashboard admin cần:
- `COUNT(bookings)`, `SUM(totalPrice)` → DB của **booking-service**
- `COUNT(tours)`, avg rating → DB của **tour-service**
- `COUNT(users)` → DB của **identity-service**

→ Tách ra sẽ phải Feign tất cả, thêm 1 service chỉ để aggregate.

### Giải pháp:

**booking-service** expose:  
`GET /api/admin/dashboard` — aggregate booking stats + revenue

**tour-service** expose:  
`GET /api/admin/tour-stats` — top tours, avg rating

**Frontend** gọi 2 endpoint trên, tự tổng hợp trong Dashboard component.

---

## 10. Common Module

```
common/
└── common-security/   ← Shared JWT logic, Security Config, User context
    ├── JwtUtil.java
    ├── JwtAuthenticationFilter.java
    └── UserPrincipal.java
```

Mỗi service import `common-security` thay vì tự viết lại JWT.

---

## 11. Tóm Tắt Quyết Định Kiến Trúc

| Quyết định | Chọn | Lý do |
|---|---|---|
| Số services | 4 | Cân bằng: đủ micro, đủ thời gian |
| Database | Riêng mỗi service | Đúng nguyên tắc microservices |
| Chatbot | Trong tour-service | Phụ thuộc tour data |
| Dashboard | Trong booking-service | Phụ thuộc booking/revenue data |
| Messaging | REST/Feign (không Kafka) | KLTN, đơn giản, dễ debug |
| Redis | Optional | Chỉ cần nếu cache booking slot |
| Frontend | Giữ client-side, cập nhật API | Không rebuild từ đầu |
| Deploy | Docker Compose | Phù hợp KLTN local/VPS |

---

## 12. Mapping Chi Tiết: Monolith → Từng Microservice

> Port code từ `D:\KLTN\Tourism_Backend\src\main\java\com\tourism\backend\`

---

### 🔐 identity-service

#### Entities (`entity/`)
| File gốc (monolith) | Giữ nguyên? | Ghi chú |
|---|---|---|
| `User.java` | ✅ Giữ | Xóa relationship `bookings`, `favoriteTours`, `reviews` |
| `RefreshToken.java` | ✅ Giữ | Giữ nguyên |

#### Repositories (`repository/`)
| File gốc | Chuyển sang |
|---|---|
| `UserRepository.java` | identity-service |
| `RefreshTokenRepository.java` | identity-service |

#### Services (`service/impl/`)
| File gốc | Chuyển sang | Chức năng |
|---|---|---|
| `AuthServiceImpl.java` | identity-service | register, login, logout, refresh token |
| `GoogleAuthServiceImpl.java` | identity-service | Google OAuth2 callback, tạo/tìm user |
| `UserServiceImpl.java` | identity-service | CRUD profile, coin balance, avatar |
| `EmailServiceImpl.java` | identity-service | Gửi email verify, welcome |
| `MailServiceImpl.java` | identity-service | Template email HTML (Thymeleaf) |
| `CloudinaryServiceImpl.java` | identity-service *(partial)* | Chỉ phần upload avatar |

#### Controllers (`controller/`)
| File gốc | Chuyển sang | Endpoints chính |
|---|---|---|
| `AuthController.java` | identity-service | `/api/auth/**` — register, login, logout, refresh |
| `AdminAuthController.java` | identity-service | `/api/admin/auth/**` — admin login |
| `UserController.java` | identity-service | `/api/users/**` — profile, change password |
| `AdminProfileController.java` | identity-service | `/api/admin/users/**` — list, block user |
| `UserFakerController.java` | identity-service *(dev)* | `/api/dev/fake-users` — seed data |

---

### 🗺️ tour-service

#### Entities (`entity/`)
| File gốc (monolith) | Giữ nguyên? | Ghi chú |
|---|---|---|
| `Tour.java` | ✅ Giữ | |
| `TourImage.java` | ✅ Giữ | |
| `TourMedia.java` | ✅ Giữ | |
| `ItineraryDay.java` | ✅ Giữ | |
| `Location.java` | ✅ Giữ | |
| `TourDeparture.java` | ✅ Giữ | Xóa relationship `bookings` (cross-service) |
| `DeparturePricing.java` | ✅ Giữ | |
| `DepartureTransport.java` | ✅ Giữ | |
| `PolicyTemplate.java` | ✅ Giữ | |
| `BranchContact.java` | ✅ Giữ | |
| `Coupon.java` | ✅ Giữ | Xóa FK `tourDeparture` nếu cross-service |
| `Review.java` | ⚠️ Sửa | `booking_id` → `INT` (no FK, cross-service); `user_id` → `INT` (no FK) |
| `ImageReview.java` | ✅ Giữ | |
| `FavoriteTour.java` | ⚠️ Sửa | `user_id` → `INT` (no FK, cross-service) |

#### Repositories (`repository/`)
| File gốc | Chuyển sang |
|---|---|
| `TourRepository.java` | tour-service |
| `TourImageRepository.java` | tour-service |
| `TourMediaRepository.java` | tour-service |
| `ItineraryDayRepository.java` | tour-service |
| `LocationRepository.java` | tour-service |
| `TourDepartureRepository.java` | tour-service |
| `DeparturePricingRepository.java` | tour-service |
| `DepartureTransportRepository.java` | tour-service |
| `PolicyTemplateRepository.java` | tour-service |
| `BranchContactRepository.java` | tour-service |
| `CouponRepository.java` | tour-service |
| `ReviewRepository.java` | tour-service |
| `ImageReviewRepository.java` | tour-service |
| `FavoriteTourRepository.java` | tour-service |

#### Services (`service/impl/` + `service/chatbot/`)
| File gốc | Chuyển sang | Chức năng |
|---|---|---|
| `TourServiceImpl.java` | tour-service | Tìm kiếm, lọc, lấy chi tiết tour |
| `TourManagementServiceImpl.java` | tour-service | Admin CRUD tour, upload ảnh |
| `TourDepartureServiceImpl.java` | tour-service | Quản lý lịch khởi hành, pricing, transport |
| `TourMediaServiceImpl.java` | tour-service | Upload/xóa video Cloudinary |
| `LocationServiceImpl.java` | tour-service | CRUD địa điểm |
| `PolicyTemplateServiceImpl.java` | tour-service | CRUD template chính sách |
| `BranchContactServiceImpl.java` | tour-service | CRUD chi nhánh |
| `CouponServiceImpl.java` | tour-service | Tạo coupon, validate, apply |
| `ReviewServiceImpl.java` | tour-service | Tạo/xem review sau booking |
| `FavoriteTourServiceImpl.java` | tour-service | Thêm/bỏ/xem tour yêu thích |
| `CloudinaryServiceImpl.java` | tour-service *(partial)* | Upload ảnh tour, video |
| `GeminiAIServiceImpl.java` | tour-service | REST call Gemini API |
| `chatbot/ChatbotService.java` | tour-service | RAG pipeline: embed → search → generate |
| `chatbot/VectorService.java` | tour-service | Pinecone CRUD (upsert, query, delete) |
| `chatbot/VectorSyncService.java` | tour-service | Sync tour/location data lên Pinecone |

#### Controllers (`controller/`)
| File gốc | Chuyển sang | Endpoints chính |
|---|---|---|
| `TourController.java` | tour-service | `GET /api/tours/**` — danh sách, chi tiết, tìm kiếm |
| `TourManagementController.java` | tour-service | `/api/admin/tours/**` — CRUD admin |
| `TourDepartureManagementController.java` | tour-service | `/api/admin/departures/**` |
| `TourUploadController.java` | tour-service | `/api/admin/tours/{id}/images` |
| `TourMediaController.java` | tour-service | `/api/admin/tours/{id}/media` |
| `LocationController.java` | tour-service | `GET /api/locations/**` |
| `LocationAdminController.java` | tour-service | `/api/admin/locations/**` |
| `PolicyTemplateController.java` | tour-service | `/api/cms/policies/**` |
| `BranchContactController.java` | tour-service | `/api/cms/branches/**` |
| `CouponController.java` | tour-service | `/api/admin/coupons/**` |
| `ReviewController.java` | tour-service | `/api/reviews/**` |
| `FavoriteTourController.java` | tour-service | `/api/favorites/**` |
| `ChatbotController.java` | tour-service | `/api/chatbot/**` |

---

### 📋 booking-service

#### Entities (`entity/`)
| File gốc (monolith) | Giữ nguyên? | Ghi chú |
|---|---|---|
| `Booking.java` | ⚠️ Sửa | `user` → `Integer userId` (no FK); `tourDeparture` → `Integer departureId` (no FK); xóa `payment` relationship |
| `BookingPassenger.java` | ✅ Giữ | |
| `RefundInformation.java` | ✅ Giữ | |
| `Notification.java` | ⚠️ Sửa | `user` → `Integer userId` (no FK) |
| `UserNotification.java` | ✅ Giữ | |

#### Repositories (`repository/`)
| File gốc | Chuyển sang |
|---|---|
| `BookingRepository.java` | booking-service |
| `RefundInformationRepository.java` | booking-service |
| `NotificationRepository.java` | booking-service |
| `UserNotificationRepository.java` | booking-service |
| *(BookingPassengerRepository)* | booking-service *(tạo mới, monolith dùng cascade)* |

#### Services (`service/impl/`)
| File gốc | Chuyển sang | Chức năng |
|---|---|---|
| `BookingServiceImpl.java` | booking-service | Tạo booking, tính tiền, apply coupon, hủy booking |
| `BookingCleanupServiceImpl.java` | booking-service | Scheduled: hủy booking PENDING quá 15 phút |
| `NotificationServiceImpl.java` | booking-service | Tạo/gửi notification in-app |
| `NotificationSyncService.java` | booking-service | Sync notification realtime (WebSocket optional) |
| `DashboardServiceImpl.java` | booking-service | Tổng doanh thu, booking theo ngày/tháng |
| `EmailServiceImpl.java` | booking-service *(partial)* | Gửi email confirm booking |
| `MailServiceImpl.java` | booking-service *(partial)* | Template email HTML |

#### Controllers (`controller/`)
| File gốc | Chuyển sang | Endpoints chính |
|---|---|---|
| `BookingController.java` | booking-service | `/api/bookings/**` — tạo, xem, hủy, hoàn tiền |
| `NotificationController.java` | booking-service | `/api/notifications/**` |
| `DashboardController.java` | booking-service | `/api/admin/dashboard/**` |

---

### 💳 payment-service

#### Entities (`entity/`)
| File gốc (monolith) | Giữ nguyên? | Ghi chú |
|---|---|---|
| `Payment.java` | ⚠️ Sửa | `booking` → `Integer bookingId` (no FK, cross-service) |

#### Repositories (`repository/`)
| File gốc | Chuyển sang |
|---|---|
| `PaymentRepository.java` | payment-service |

#### Services (`service/impl/`)
| File gốc | Chuyển sang | Chức năng |
|---|---|---|
| `PaymentServiceImpl.java` | payment-service | Tạo payment, VNPay flow, callback handler |
| `PayOSService.java` | payment-service | Tạo link PayOS, webhook handler |
| `SepayServiceImpl.java` | payment-service | Polling Sepay, match bank transaction |

#### Controllers (`controller/`)
| File gốc | Chuyển sang | Endpoints chính |
|---|---|---|
| `PaymentController.java` | payment-service | `/api/payments/**` — tạo link, callback, webhook |

---

### ♻️ Tổng Hợp: File Dùng Chung

| File gốc | Dùng ở | Cách xử lý |
|---|---|---|
| `EmailServiceImpl.java` | identity + booking | Copy vào cả 2, sửa package |
| `MailServiceImpl.java` | identity + booking | Copy vào cả 2, sửa package |
| `CloudinaryServiceImpl.java` | identity + tour | Copy vào cả 2, sửa package |
| `BaseEntity.java` | tất cả | Copy vào mỗi service |
| All `enums/` | tất cả | Copy vào mỗi service tương ứng |
| All `dto/` | tất cả | Copy và tạo lại DTO phù hợp từng service |
| `WebSocketService.java` | booking (optional) | Copy nếu cần notification realtime |

---

### 🗑️ File Không Port (Không Cần Trong Micro)

| File gốc | Lý do bỏ |
|---|---|
| `config/SecurityConfig.java` | API Gateway xử lý JWT, mỗi service dùng `common-security` |
| `security/` (toàn bộ folder) | Thay bằng `common-security` module |
| `UserFakerController.java` | Dev-only, bỏ khi production |
| `GeminiAIServiceImpl.java` | Logic nằm thẳng trong `ChatbotService`, bỏ abstraction này |

---

### 📊 Thống Kê Mapping

| Service | Entities | Repositories | Services | Controllers |
|---|---|---|---|---|
| identity-service | 2 | 2 | 6 | 5 |
| tour-service | 14 | 14 | 13 | 13 |
| booking-service | 5 | 5 | 7 | 3 |
| payment-service | 1 | 1 | 3 | 1 |
| **Tổng** | **22** | **22** | **29** | **22** |
