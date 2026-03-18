# 🗺️ Tourism Project — Master Plan 4 Tuần (28 Ngày)

> **Vai trò**: Senior Solution Architect + Full-stack Expert  
> **Mục tiêu**: Hoàn thiện Backend Microservices (`tourism-microservices`) + Xây Frontend mới giao tiếp qua API Gateway  
> **Ngày bắt đầu**: 18/03/2026 → **Ngày hoàn thành**: 15/04/2026

---

## 📊 Phân Tích Dự Án Hiện Tại

### Monolith (`Tourism_Backend`) — Đã có gì?

| Domain | Controllers | Entities |
|--------|-------------|----------|
| Auth/User | `AuthController`, `AdminAuthController`, `UserController` | `User`, `RefreshToken` |
| Tour | `TourController`, `TourManagementController`, `TourDepartureManagementController`, `TourMediaController`, `TourUploadController` | `Tour`, `TourDeparture`, `TourImage`, `TourMedia`, `ItineraryDay`, `DeparturePricing`, `DepartureTransport` |
| Booking | `BookingController`, `PaymentController` | `Booking`, `BookingPassenger`, `Payment`, `RefundInformation` |
| Review | `ReviewController` | `Review`, `ImageReview` |
| Promo | `CouponController` | `Coupon` |
| Location | `LocationController`, `LocationAdminController` | `Location` |
| CMS | `PolicyTemplateController`, `BranchContactController` | `PolicyTemplate`, `BranchContact` |
| Notification | `NotificationController` | `Notification`, `UserNotification` |
| Dashboard | `DashboardController`, `ChatbotController` | — |

**Tech stack Monolith**: Spring Boot 3.3, PostgreSQL, JWT (JJWT 0.11.5), Spring Security, Cloudinary, VNPay, PayOS, SePay, WebSocket, Kafka, Pinecone, Google API, Flyway

### Microservices (`tourism-microservices`) — Đã có gì?

| Service | Port | Database | Status |
|---------|------|----------|--------|
| `service-registry` (Eureka) | 8761 | — | ✅ Có Dockerfile |
| `api-gateway` (Spring Cloud GW) | 8080 | — | ✅ Có JWT filter |
| `identity-service` | 8081 | `tourism_identity` | 🔧 Cần hoàn thiện |
| `tour-catalog-service` | 8082 | `tourism_catalog` | 🔧 Cần hoàn thiện |
| `booking-service` | 8083 | `tourism_booking` | 🔧 Cần hoàn thiện |
| `payment-service` | 8084 | `tourism_payment` | 🔧 Cần hoàn thiện |
| `review-service` | 8085 | `tourism_review` | 🔧 Cần hoàn thiện |
| `promotion-service` | 8086 | `tourism_promotion` | 🔧 Cần hoàn thiện |
| `notification-service` | 8087 | `tourism_notification` | 🔧 Cần hoàn thiện |
| `analytics-service` | 8088 | `tourism_analytics` | 🔧 Cần hoàn thiện |
| `cms-service` | 8089 | `tourism_cms` | 🔧 Cần hoàn thiện |

**Shared libs**: `common-security` (JWT filter), `common-events` (Kafka schemas)  
**Infrastructure**: Kafka + Zookeeper, PostgreSQL 15, Redis 7, Zipkin, Docker Compose

### Frontend (`client-side`) — Hiện tại

| Thành phần | Công nghệ |
|------------|-----------|
| Framework | React 18 + TypeScript + CRA (react-scripts) |
| Routing | react-router-dom v7 |
| State | Redux Toolkit |
| HTTP | Axios |
| UI | Ant Design 5, Bootstrap 5, Lucide React |
| Charts | Recharts |
| Realtime | STOMP.js, SockJS, socket.io-client |
| Payments | Tích hợp VNPay, PayOS redirect |

---

## 🏗️ Kiến Trúc Frontend Mới (Đề Xuất)

```
client-side-new/  (Vite + React 18 + TypeScript)
├── src/
│   ├── api/                    # API layer — gọi qua API Gateway :8080
│   │   ├── axiosInstance.ts    # Base axios + JWT interceptor
│   │   ├── auth.api.ts         # → identity-service
│   │   ├── tours.api.ts        # → tour-catalog-service
│   │   ├── bookings.api.ts     # → booking-service
│   │   ├── payments.api.ts     # → payment-service
│   │   ├── reviews.api.ts      # → review-service
│   │   ├── promotions.api.ts   # → promotion-service
│   │   ├── notifications.api.ts# → notification-service
│   │   ├── analytics.api.ts    # → analytics-service
│   │   └── cms.api.ts          # → cms-service
│   ├── store/                  # Redux Toolkit
│   │   ├── index.ts
│   │   ├── auth/authSlice.ts
│   │   ├── tours/toursSlice.ts
│   │   ├── bookings/bookingsSlice.ts
│   │   └── notifications/notificationsSlice.ts
│   ├── hooks/                  # Custom hooks
│   │   ├── useAuth.ts
│   │   ├── useTours.ts
│   │   └── useBookings.ts
│   ├── pages/                  # Route-level pages
│   │   ├── public/
│   │   │   ├── HomePage.tsx
│   │   │   ├── ToursPage.tsx
│   │   │   └── TourDetailPage.tsx
│   │   ├── auth/
│   │   │   ├── LoginPage.tsx
│   │   │   └── RegisterPage.tsx
│   │   ├── user/
│   │   │   ├── BookingsPage.tsx
│   │   │   ├── ProfilePage.tsx
│   │   │   └── NotificationsPage.tsx
│   │   └── admin/
│   │       ├── DashboardPage.tsx
│   │       ├── ToursManagePage.tsx
│   │       └── BookingsManagePage.tsx
│   ├── components/             # Tái sử dụng
│   │   ├── layout/
│   │   ├── common/
│   │   ├── tour/
│   │   ├── booking/
│   │   └── admin/
│   ├── types/                  # TypeScript interfaces
│   └── utils/                  # Helpers, formatters
```

**Chiến lược Authentication:**
```
FE → POST /api/auth/login → API Gateway → identity-service
     ← { accessToken, refreshToken }
     
FE lưu accessToken trong memory (Redux), refreshToken trong httpOnly cookie
Mỗi request: Authorization: Bearer <accessToken>
API Gateway validate JWT → inject X-User-Id, X-User-Email, X-User-Name headers
Auto-refresh: axios interceptor catch 401 → call refresh-token → retry
```

---

## 📅 MASTER PLAN — 4 TUẦN CHI TIẾT

---

## 🚀 TUẦN 1 (18/03 - 24/03): NỀN TẢNG BACKEND + IDENTITY SERVICE

> **Mục tiêu**: Hệ thống chạy được trên Docker Compose, Identity Service hoàn chỉnh 100%.

---

### 📅 Ngày 1 (Thứ 2, 18/03) — Audit & Setup Môi Trường

**Buổi sáng (3-4 tiếng)**:
- [ ] Chạy `docker-compose up -d` — kiểm tra tất cả containers đang hoạt động
- [ ] Mở Eureka dashboard `http://localhost:8761` — xem service nào đã register
- [ ] Mở Zipkin `http://localhost:9411` — xem tracing hoạt động không
- [ ] Test API Gateway: `curl http://localhost:8080/api/tours` — kiểm tra routing

**Buổi chiều (3-4 tiếng)**:
- [ ] Đọc source code `identity-service`: kiểm tra đã có gì, thiếu gì so với monolith
- [ ] So sánh [AuthController.java](file:///D:/KLTN/Tourism_Backend/src/main/java/com/tourism/backend/controller/AuthController.java) (monolith) vs `identity-service` — liệt kê gap
- [ ] Tạo file **`gap-analysis.md`** ghi rõ từng endpoint nào đã có/chưa có
- [ ] Cài extension/tool: IntelliJ multimodule run, DBeaver kết nối PostgreSQL

**Deliverable cuối ngày**: File gap-analysis.md rõ ràng, toàn bộ services đang chạy trên Docker

---

### 📅 Ngày 2 (Thứ 3, 19/03) — Identity Service: Auth Core

**Buổi sáng**:
- [ ] Hoàn thiện `POST /api/auth/register` trong identity-service:
  - Validate email chưa tồn tại
  - Mã hoá password (BCrypt)
  - Gửi email xác thực (JavaMail)
  - Publish Kafka event `user.registered`
- [ ] Hoàn thiện `POST /api/auth/login`:
  - Verify credentials
  - Generate JWT (accessToken 15 phút) + refreshToken (7 ngày)
  - Lưu refreshToken vào DB

**Buổi chiều**:
- [ ] Hoàn thiện `POST /api/auth/refresh-token` — cơ chế silent refresh
- [ ] Hoàn thiện `POST /api/auth/logout` + `POST /api/auth/logout-all`
- [ ] Test thủ công với Postman: register → verify email → login → logout

**Mã cần viết**: `AuthController`, `AuthService`, `TokenService`, `RefreshTokenRepository`

---

### 📅 Ngày 3 (Thứ 4, 20/03) — Identity Service: Email Verify + Google OAuth

**Buổi sáng**:
- [ ] Hoàn thiện `GET /api/auth/verify-email?token=...`:
  - Token random UUID 24h expire
  - Redirect sau verify thành công
- [ ] Configure JavaMail với Gmail SMTP trong `application.yml`
- [ ] Tạo email template HTML đẹp (lấy từ monolith Thymeleaf)

**Buổi chiều**:
- [ ] Hoàn thiện `POST /api/auth/google/login`:
  - Nhận Google ID Token từ FE
  - Verify với Google API (`google-api-client`)
  - Nếu user chưa tồn tại → tạo mới
  - Trả về JWT như login thường
- [ ] Test Google login flow end-to-end

---

### 📅 Ngày 4 (Thứ 5, 21/03) — Identity Service: User Profile + Admin

**Buổi sáng**:
- [ ] `GET /api/auth/profile` — lấy thông tin user từ `X-User-Id` header
- [ ] `PATCH /api/users/{id}/profile` — cập nhật tên, SĐT, upload avatar Cloudinary
- [ ] `PATCH /api/users/{id}/change-password` — verify old password, set new

**Buổi chiều**:
- [ ] `GET /api/admin/users` — phân trang, filter, search
- [ ] `PATCH /api/admin/users/{id}/status` — lock/unlock account
- [ ] Bảo mật: `@PreAuthorize("hasRole('ADMIN')")` cho admin endpoints
- [ ] Test toàn bộ identity-service — tạo Postman Collection

**Deliverable**: Identity Service 100% complete, Postman collection đầy đủ

---

### 📅 Ngày 5 (Thứ 6, 22/03) — API Gateway: Routing + JWT Filter

**Buổi sáng**:
- [ ] Review `api-gateway` config: kiểm tra tất cả routes đã đúng chưa
- [ ] Đảm bảo JWT filter hoạt động đúng:
  - Public routes không cần token: `GET /api/tours/**`, `POST /api/auth/**`, `GET /api/reviews/**`, `GET /api/cms/**`
  - Protected routes: tất cả còn lại
- [ ] Filter inject `X-User-Id`, `X-User-Email`, `X-User-Name` vào headers

**Buổi chiều**:
- [ ] Config CORS trong API Gateway: chấp nhận `http://localhost:3000` (FE dev)
- [ ] Test: request không có token → 401, có token → forward đến service
- [ ] Test: token sai → 401, token đúng nhưng role không đủ → 403
- [ ] Thêm route cho `internal/**` — chỉ cho phép từ internal network (không expose ra ngoài)

---

### 📅 Ngày 6 (Thứ 7, 23/03) — Shared Libs + Common Security

**Buổi sáng**:
- [ ] Review `common-security` library:
  - `JwtAuthenticationFilter` — parse JWT từ header, set SecurityContext
  - `UserPrincipal` — có đủ userId, email, role
- [ ] Đảm bảo tất cả services đều import `common-security` và config đúng
- [ ] Review `common-events` — tất cả Kafka event classes đủ fields chưa

**Buổi chiều**:
- [ ] Test inter-service auth: gọi endpoint protected từ một service khác (internal Feign call)
- [ ] Viết README cập nhật cho từng shared lib
- [ ] Kiểm tra [init-db.sql](file:///D:/KLTN/tourism-microservices/init-db.sql) — đảm bảo tất cả 9 databases được tạo khi khởi động

---

### 📅 Ngày 7 (Chủ Nhật, 24/03) — Buffer + Review Tuần 1

- [ ] Fix tất cả bugs từ ngày 1-6
- [ ] Test end-to-end: Register → Verify Email → Login → Get Profile → Change Password
- [ ] Cập nhật `gap-analysis.md` — đánh dấu những gì đã xong
- [ ] Commit & push toàn bộ code Identity + API Gateway lên Git
- [ ] **Chuẩn bị**: Đọc trước entities Tour, Departure của monolith

**✅ Cuối Tuần 1**: Identity Service hoàn chỉnh, API Gateway chuẩn, Docker Compose chạy mượt

---

## 🚀 TUẦN 2 (25/03 - 31/03): TOUR + BOOKING + PAYMENT SERVICES

> **Mục tiêu**: Luồng cốt lõi hoàn chỉnh — Browse Tour → Đặt → Thanh Toán → Xác Nhận

---

### 📅 Ngày 8 (Thứ 2, 25/03) — Tour Catalog Service: CRUD + Schema

**Buổi sáng**:
- [ ] Thiết kế DB schema cho `tourism_catalog`:
  - `tours` table: id, code, title, description, thumbnail_url, region, location_id, status, price_from
  - `tour_images` table: id, tour_id, url, type
  - `tour_departures` table: id, tour_id, departure_date, available_slots, price, status
  - `itinerary_days` table: id, tour_id, day_number, title, description
  - `locations` table: id, name, region, latitude, longitude
- [ ] Viết Flyway migrations (hoặc dùng `spring.jpa.hibernate.ddl-auto=create`)

**Buổi chiều**:
- [ ] Port code từ monolith [Tour.java](file:///D:/KLTN/Tourism_Backend/src/main/java/com/tourism/backend/entity/Tour.java), [TourDeparture.java](file:///D:/KLTN/Tourism_Backend/src/main/java/com/tourism/backend/entity/TourDeparture.java), [Location.java](file:///D:/KLTN/Tourism_Backend/src/main/java/com/tourism/backend/entity/Location.java) → entities mới
- [ ] `GET /api/tours` — filter region, location, keyword, pagination
- [ ] `GET /api/tours/featured` — tours nổi bật (sort by booking count)
- [ ] `GET /api/tours/{id}` — chi tiết tour kèm itinerary

---

### 📅 Ngày 9 (Thứ 3, 26/03) — Tour Catalog Service: Departures + Images

**Buổi sáng**:
- [ ] `GET /api/tours/{id}/departures` — tất cả lịch khởi hành
- [ ] `GET /api/tours/{id}/departures/available` — lịch còn chỗ (available_slots > 0)
- [ ] `GET /api/tours/code/{code}` — tìm tour bằng code
- [ ] `GET /api/locations` — danh sách địa điểm

**Buổi chiều**:
- [ ] Admin endpoints:
  - `POST /api/admin/tours` — tạo tour mới
  - `PUT /api/admin/tours/{id}` — cập nhật
  - `PATCH /api/admin/tours/{id}/status` — active/inactive
  - `POST /api/admin/tours/{id}/thumbnail` — upload ảnh bìa Cloudinary
  - `POST /api/admin/tours/{id}/images` — upload nhiều ảnh gallery
  - `POST /api/admin/tours/{id}/departures` — thêm lịch khởi hành
- [ ] **Internal API**: `GET /internal/tours/{id}` — cho booking-service gọi

---

### 📅 Ngày 10 (Thứ 4, 27/03) — Booking Service: Tạo Booking + Redis Cache

**Buổi sáng**:
- [ ] Thiết kế DB schema `tourism_booking`:
  - `bookings`: id, code, user_id, tour_id, departure_id, status, total_amount, coupon_code
  - `booking_passengers`: id, booking_id, name, dob, passport_no, type
- [ ] `POST /api/bookings` — tạo booking:
  1. Gọi Feign → `tour-catalog-service` lấy departure info
  2. Gọi Feign → `promotion-service` validate coupon (nếu có)
  3. Tính `total_amount = passengers * price - discount`
  4. Tạo booking với status `PENDING_PAYMENT`
  5. Publish Kafka `booking.created`

**Buổi chiều**:
- [ ] Configure Redis Cache trong booking-service
- [ ] Cache thông tin booking: `@Cacheable("bookings")` khi get booking by code
- [ ] `GET /api/bookings/my` — lịch sử booking của user (lấy userId từ header `X-User-Id`)
- [ ] `GET /api/bookings/{id}` + `GET /api/bookings/code/{code}`

---

### 📅 Ngày 11 (Thứ 5, 28/03) — Booking Service: Cancel + Internal APIs

**Buổi sáng**:
- [ ] `POST /api/bookings/{id}/cancel` — user huỷ booking:
  - Chỉ huỷ được khi status là `PENDING_PAYMENT`
  - Publish Kafka `booking.cancelled`
- [ ] `GET /api/admin/bookings` — admin xem tất cả + filter
- [ ] `POST /api/admin/bookings/{id}/cancel` — admin force cancel

**Buổi chiều**:
- [ ] **Internal API** `POST /internal/bookings/{code}/confirm`:
  - Payment service gọi sau khi thanh toán thành công
  - Đổi status sang `CONFIRMED`
  - Publish Kafka `booking.confirmed`
- [ ] **Internal API** `GET /internal/bookings/check-confirmed`:
  - Review service gọi để kiểm tra user đã có booking confirmed cho tour chưa
- [ ] Test Feign clients: booking → tour-catalog, booking → promotion
- [ ] Test Kafka: booking.created consumer ở notification-service có nhận không

---

### 📅 Ngày 12 (Thứ 6, 29/03) — Payment Service: VNPay + PayOS

**Buổi sáng**:
- [ ] Port code VNPay từ monolith [PaymentController.java](file:///D:/KLTN/Tourism_Backend/src/main/java/com/tourism/backend/controller/PaymentController.java):
  - `POST /api/payments/initiate`:
    - Nhận `bookingCode`, `paymentMethod`
    - Generate VNPay redirect URL hoặc PayOS QR
    - Lưu `payment` record với status `PENDING`
  - `GET /api/payments/vnpay-return` — VNPay redirect callback:
    - Verify chữ ký VNPay
    - Gọi Feign → `booking-service` confirm booking
    - Publish Kafka `payment.completed`
    - Redirect FE về trang kết quả

**Buổi chiều**:
- [ ] `POST /api/payments/payos-webhook` — PayOS webhook
- [ ] `POST /api/payments/sepay-webhook` — SePay webhook
- [ ] `GET /api/payments/booking/{code}` — xem trạng thái thanh toán
- [ ] Test luồng hoàn chỉnh: tạo booking → initiate VNPay → giả lập return → booking confirmed

---

### 📅 Ngày 13 (Thứ 7, 30/03) — Promotion + Notification Services

**Buổi sáng — Promotion Service**:
- [ ] `GET /api/promotions/coupons/{code}` — validate coupon
- [ ] Admin CRUD coupon
- [ ] **Internal**: `POST /internal/coupons/{code}/apply` — trừ usage_count

**Buổi chiều — Notification Service**:
- [ ] Kafka consumer `booking.created` → gửi email HTML "Đặt tour thành công" + tạo in-app notification
- [ ] Kafka consumer `booking.cancelled` → email huỷ tour
- [ ] Kafka consumer `booking.confirmed` → email xác nhận
- [ ] Kafka consumer `payment.completed` → email thanh toán
- [ ] Kafka consumer `user.registered` → email welcome
- [ ] `GET /api/notifications/my` — danh sách thông báo của user

---

### 📅 Ngày 14 (Chủ Nhật, 31/03) — Buffer + Integration Test Tuần 2

- [ ] Test full luồng: Register → Login → Browse Tours → Create Booking → Thanh Toán → Email
- [ ] Fix bugs Kafka consumer (thường gặp vấn đề offset, deserialization)
- [ ] Kiểm tra Redis cache: booking cached đúng không, invalidate khi cancel
- [ ] Commit code, update gap-analysis.md

**✅ Cuối Tuần 2**: Core business flow hoàn chỉnh (Tour/Booking/Payment/Notification)

---

## 🚀 TUẦN 3 (01/04 - 07/04): REVIEW + ANALYTICS + CMS + FRONTEND SETUP

> **Mục tiêu**: Một nửa tuần hoàn thiện backend còn lại, một nửa bắt đầu Frontend mới

---

### 📅 Ngày 15 (Thứ 2, 01/04) — Review Service

**Buổi sáng**:
- [ ] DB schema `tourism_review`: `reviews`, `review_images`
- [ ] `POST /api/reviews` — tạo đánh giá:
  1. Lấy `userId` từ header
  2. Gọi Feign → `booking-service` internal `/check-confirmed`
  3. Nếu chưa có booking confirmed → 403
  4. Upload ảnh lên Cloudinary
  5. Publish Kafka `review.created`

**Buổi chiều**:
- [ ] `GET /api/reviews/tour/{code}` — danh sách reviews (public, pagination)
- [ ] `GET /api/reviews/tour/{code}/summary` — avg rating, phân phối sao
- [ ] `GET /api/reviews/my` — reviews của user hiện tại
- [ ] `GET /api/reviews/eligibility/{bookingCode}` — kiểm tra đủ điều kiện đánh giá
- [ ] `DELETE /api/reviews/{id}` + `DELETE /api/reviews/admin/{id}`

---

### 📅 Ngày 16 (Thứ 3, 02/04) — Analytics Service + AI Chatbot

**Buổi sáng**:
- [ ] DB schema `tourism_analytics`: `daily_stats`, `tour_stats`
- [ ] Kafka consumers:
  - `booking.created` → tăng `new_bookings`
  - `booking.confirmed` → tăng `confirmed_bookings`
  - `booking.cancelled` → tăng `cancelled_bookings`
  - `payment.completed` → cộng `total_revenue`
  - `review.created` → incremental average rating
- [ ] `GET /api/analytics/dashboard/summary`
- [ ] `GET /api/analytics/dashboard/daily-stats` (7 ngày gần nhất)
- [ ] `GET /api/analytics/dashboard/top-tours`

**Buổi chiều**:
- [ ] Port `ChatbotController` từ monolith → analytics-service
- [ ] Integrate Google Gemini API (biến môi trường `GEMINI_API_KEY`)
- [ ] `POST /api/analytics/chatbot/chat` — AI trả lời về dữ liệu kinh doanh

---

### 📅 Ngày 17 (Thứ 4, 03/04) — CMS Service + Final Backend Cleanup

**Buổi sáng**:
- [ ] `cms-service`: policies (CRUD), branches (CRUD)
- [ ] `GET /api/cms/policies/type/{type}` — lấy chính sách mới nhất theo loại
- [ ] Test toàn bộ 9 services bằng Postman — báo cáo coverage

**Buổi chiều**:
- [ ] Viết `docker-compose.override.yml` cho dev (mount source, hotreload)
- [ ] Đảm bảo [init-db.sql](file:///D:/KLTN/tourism-microservices/init-db.sql) tạo đúng 9 databases
- [ ] Chuẩn hoá error responses: tất cả services dùng cùng format `{ success, message, data, errors }`
- [ ] **Backend hoàn thiện!** — commit full, tag `v1.0-backend`

---

### 📅 Ngày 18 (Thứ 5, 04/04) — Setup Frontend Mới (Vite + React + TypeScript)

**Buổi sáng**:
- [ ] Khởi tạo project mới: `npm create vite@latest tourism-frontend -- --template react-ts`
- [ ] Cài dependencies:
  ```bash
  npm install axios @reduxjs/toolkit react-redux react-router-dom
  npm install antd @ant-design/icons lucide-react
  npm install @react-oauth/google react-toastify
  npm install recharts swiper react-datepicker
  npm install @stomp/stompjs sockjs-client
  ```
- [ ] Cài dev dependencies: `npm install -D sass`
- [ ] Tạo cấu trúc thư mục như đề xuất ở trên

**Buổi chiều**:
- [ ] Tạo `axiosInstance.ts`:
  - `baseURL: http://localhost:8080` (API Gateway)
  - Request interceptor: attach `Authorization: Bearer <token>`
  - Response interceptor: catch 401 → auto call `/api/auth/refresh-token` → retry
- [ ] Tạo cấu trúc Redux store: `authSlice`, `notificationsSlice`
- [ ] Setup React Router v6 với layouts: PublicLayout, AuthLayout, AdminLayout

---

### 📅 Ngày 19 (Thứ 6, 05/04) — Frontend: Auth Pages

**Buổi sáng**:
- [ ] **LoginPage**: Form email/password + Google Sign-In button
  - Submit → `POST /api/auth/login`
  - Lưu accessToken trong Redux, refreshToken trong localStorage/cookie
  - Redirect về trang trước hoặc Home
- [ ] **RegisterPage**: Form đăng ký + validation
  - Submit → `POST /api/auth/register`
  - Redirect đến trang "Kiểm tra email xác thực"

**Buổi chiều**:
- [ ] **VerifyEmailPage**: Show message "Đã gửi email xác thực" + resend button
- [ ] **Protected Route**: HOC kiểm tra token trong Redux store
- [ ] **Google OAuth**: Integrate `@react-oauth/google` với `GoogleOAuthProvider`
- [ ] Test toàn bộ auth flow trong browser

---

### 📅 Ngày 20 (Thứ 7, 06/04) — Frontend: HomePage + Tour List

**Buổi sáng**:
- [ ] **HomePage** (public):
  - Hero banner với search bar (điểm đến, ngày đi, số người)
  - Section "Tour nổi bật" → `GET /api/tours/featured`
  - Section destinations/regions (lấy từ `/api/locations`)
  - Swiper carousel cho banner
- [ ] Design: dark hero gradient, card tour đẹp với hover animation

**Buổi chiều**:
- [ ] **ToursPage** (public):
  - Filter sidebar: region, price range, departure date
  - Grid layout tour cards
  - Pagination
  - Search: `GET /api/tours?keyword=&region=&location=`
- [ ] **TourCard component**: thumbnail, title, price, rating stars, departure count

---

### 📅 Ngày 21 (Chủ Nhật, 07/04) — Buffer + Review Tuần 3

- [ ] Fix bugs tuần 3
- [ ] Polish design HomePage + ToursPage (animations, responsive)
- [ ] Commit code FE, test với backend local

**✅ Cuối Tuần 3**: Backend 100% hoàn chỉnh, Frontend cơ bản có Auth + Home + Tour List

---

## 🚀 TUẦN 4 (08/04 - 15/04): FRONTEND HOÀN THIỆN + TÍCH HỢP + POLISH

> **Mục tiêu**: Frontend hoàn chỉnh end-to-end, tích hợp toàn bộ API, sẵn sàng demo

---

### 📅 Ngày 22 (Thứ 2, 08/04) — Tour Detail + Booking Flow

**Buổi sáng**:
- [ ] **TourDetailPage** (public):
  - Gallery ảnh (Swiper lightbox)
  - Tabs: Mô tả, Lịch trình (itinerary days), Chính sách
  - Bảng `Lịch khởi hành còn chỗ` `GET /api/tours/{id}/departures/available`
  - Panel "Đặt ngay" → chọn departure, số khách, nhập coupon

**Buổi chiều**:
- [ ] **BookingPage** (protected):
  - Form thông tin hành khách (tên, CCCD, ngày sinh, loại khách)
  - Tóm tắt booking + tổng tiền (có áp dụng coupon)
  - `POST /api/bookings` → redirect đến trang thanh toán

---

### 📅 Ngày 23 (Thứ 3, 09/04) — Payment + Booking History

**Buổi sáng**:
- [ ] **PaymentPage** (protected):
  - Hiển thị tóm tắt đơn hàng
  - Chọn phương thức: VNPay / PayOS / SePay
  - `POST /api/payments/initiate` → nhận paymentUrl → redirect

- [ ] **PaymentResultPage**:
  - VNPay return redirect về `/payment/result?vnp_ResponseCode=00`
  - Parse params, hiển thị "Thanh toán thành công" với animation confetti
  - Link về "My Bookings"

**Buổi chiều**:
- [ ] **MyBookingsPage** (protected):
  - `GET /api/bookings/my` — danh sách bookings theo status
  - Tabs: Tất cả / Chờ thanh toán / Đã xác nhận / Đã huỷ
  - **BookingDetailCard**: mã booking, tour, ngày đi, hành khách, trạng thái
  - Nút "Huỷ" với confirm dialog → `POST /api/bookings/{id}/cancel`

---

### 📅 Ngày 24 (Thứ 4, 10/04) — Reviews + Notifications

**Buổi sáng**:
- [ ] **ReviewSection** trong TourDetailPage:
  - `GET /api/reviews/tour/{code}` — danh sách reviews với avatar, rating, ảnh
  - `GET /api/reviews/tour/{code}/summary` — tổng quan rating (bar chart)
- [ ] **WriteReviewModal** (protected):
  - Chỉ hiện nếu user có booking `CONFIRMED` cho tour này
  - Star rating, text, upload ảnh
  - `POST /api/reviews`

**Buổi chiều**:
- [ ] **NotificationsPage** (protected):
  - `GET /api/notifications/my` — danh sách thông báo
  - Badge đỏ số thông báo chưa đọc trên header
  - Realtime: polling mỗi 30 giây hoặc SSE (nếu có thì dùng)
- [ ] **ProfilePage** (protected):
  - `GET /api/auth/profile` — hiển thị thông tin
  - `PATCH /api/users/{id}/profile` — edit name, phone, avatar upload
  - `PATCH /api/users/{id}/change-password`

---

### 📅 Ngày 25 (Thứ 5, 11/04) — Admin Dashboard

**Buổi sáng**:
- [ ] **Admin Layout**: sidebar navigation, header với avatar
- [ ] **DashboardPage** (admin):
  - `GET /api/analytics/dashboard/summary` — tổng quan
  - `GET /api/analytics/dashboard/daily-stats` → Recharts Line/Bar chart
  - `GET /api/analytics/dashboard/top-tours` → Top 5 tours bảng
  - Stats cards: tổng doanh thu, bookings hôm nay, reviews mới

**Buổi chiều**:
- [ ] **AI Chatbot Widget** (admin):
  - Floating chat button
  - `POST /api/analytics/chatbot/chat` — gửi tin nhắn, nhận câu trả lời AI
  - Hiển thị markdown response (react-markdown)

---

### 📅 Ngày 26 (Thứ 6, 12/04) — Admin: Tour + Booking Management

**Buổi sáng**:
- [ ] **ToursManagePage** (admin):
  - `GET /api/admin/tours` → Ant Design Table với filter, sort
  - Modal tạo tour mới: form với react-quill editor cho description
  - Upload ảnh bìa + gallery (drag-drop)
  - Quản lý lịch khởi hành: thêm/sửa departure

**Buổi chiều**:
- [ ] **BookingsManagePage** (admin):
  - `GET /api/admin/bookings` → bảng tất cả bookings
  - Filter theo status, tour, ngày đặt
  - Admin cancel booking với lý do
- [ ] **CouponsManagePage** (admin):
  - CRUD coupon
- [ ] **UsersManagePage** (admin):
  - `GET /api/admin/users` — danh sách users
  - Lock/unlock tài khoản

---

### 📅 Ngày 27 (Thứ 7, 13/04) — Polish UI + Responsive + Error Handling

**Buổi sáng**:
- [ ] **Responsive design**: kiểm tra tất cả pages trên mobile (375px, 768px, 1024px)
- [ ] **Loading states**: skeleton loading cho tour cards, booking list
- [ ] **Error boundaries**: trang 404, 403, 500 đẹp
- [ ] **Toast notifications**: feedback rõ ràng cho tất cả actions (react-toastify)

**Buổi chiều**:
- [ ] **SEO**: `<title>`, `<meta description>` cho từng page
- [ ] **Performance**: lazy load routes (`React.lazy + Suspense`)
- [ ] **Animations**: page transitions, button hover, card hover scale
- [ ] Kiểm tra toàn bộ flow với backend thực tế — ghi bug list

---

### 📅 Ngày 28 (Chủ Nhật, 14/04) — Final Integration Test + Documentation

**Buổi sáng**:
- [ ] **End-to-end test** toàn bộ user journey:
  1. User đăng ký → xác thực email ✅
  2. Login (email + Google) ✅
  3. Browse tours, xem chi tiết ✅
  4. Đặt tour + điền hành khách ✅
  5. Thanh toán VNPay/PayOS ✅
  6. Nhận email xác nhận ✅
  7. Xem booking history ✅
  8. Viết review sau khi tour ✅
  9. Admin: xem dashboard, quản lý tours/bookings ✅

**Buổi chiều**:
- [ ] Fix critical bugs phát hiện trong morning test
- [ ] `docker-compose up -d` lần cuối — đảm bảo 100% services green
- [ ] Viết [README.md](file:///D:/KLTN/client-side/README.md) cho project tổng thể:
  - Giới thiệu kiến trúc
  - Hướng dẫn cài đặt
  - API endpoints reference
  - Screenshots UI
- [ ] **Commit final**: tag `v1.0-release`

---

## 📋 Tổng Kết Deliverables

| Tuần | Backend | Frontend |
|------|---------|----------|
| Tuần 1 | ✅ Identity Service + API Gateway hoàn chỉnh | — |
| Tuần 2 | ✅ Tour + Booking + Payment + Promotion + Notification | — |
| Tuần 3 | ✅ Review + Analytics + CMS (backend done!) | ✅ Setup + Auth + Home + Tours |
| Tuần 4 | ✅ Bug fixes, integration | ✅ Booking + Payment + Review + Admin + Polish |

---

## ⚠️ Rủi Ro & Cách Xử Lý

| Rủi Ro | Xác Suất | Giải Pháp |
|--------|----------|-----------|
| Feign Client circular dependency | Trung bình | Dùng `@Lazy` injection hoặc RestTemplate thay thế |
| Kafka consumer lag / message loss | Thấp | Config `auto-offset-reset=earliest`, kiểm tra consumer group |
| VNPay/PayOS webhook không nhận được (local) | Cao | Dùng `ngrok http 8080` để expose local webhook endpoint |
| Redis connection trong Docker | Thấp | Đảm bảo `REDIS_HOST=redis` (container name) không phải `localhost` |
| CORS FE → API Gateway | Trung bình | Config `allowed-origins` trong Gateway cho cả `http://localhost:5173` (Vite) |
| JWT secret sync giữa services | Thấp | Dùng chung 1 biến [.env](file:///D:/KLTN/client-side/src/.env) `JWT_SECRET` cho toàn bộ |

---

## 🛠️ Lệnh Chạy Hàng Ngày

```bash
# Khởi động toàn bộ backend
cd D:\KLTN\tourism-microservices
docker-compose up -d

# Xem logs một service cụ thể
docker-compose logs -f identity-service

# Khởi động frontend mới (dev mode)
cd D:\KLTN\tourism-frontend
npm run dev
# → http://localhost:5173

# Kiểm tra tất cả services đã register Eureka
# → http://localhost:8761

# Tracing requests
# → http://localhost:9411
```

---

## 📈 Tiêu Chí Hoàn Thành

- [ ] Tất cả 9 microservices chạy ổn định trong Docker
- [ ] API Gateway routing + JWT auth hoạt động đúng
- [ ] Frontend kết nối 100% qua `http://localhost:8080` (API Gateway)
- [ ] Luồng đặt tour end-to-end được kiểm thử thực tế
- [ ] Admin dashboard hiển thị dữ liệu realtime
- [ ] Email notifications hoạt động (booking created/confirmed/cancelled)
- [ ] Code được commit lên Git với tag version

