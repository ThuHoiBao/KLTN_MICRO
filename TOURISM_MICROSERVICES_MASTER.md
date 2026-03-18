# 🌍 Tourism Microservices — Tài Liệu Kiến Trúc & Kế Hoạch Triển Khai Hoàn Chỉnh

> **Ngày**: 18/03/2026 | **Dự án**: KLTN — Tourism Management System  
> **Mục tiêu**: Chuyển đổi hoàn toàn Monolith → Microservices + Frontend mới + Deploy AWS  
> **Codebase tham chiếu**: `D:\KLTN\tourism-microservices`

---

# PHẦN 1: PHÂN TÍCH & THIẾT KẾ BACKEND MICROSERVICES

## 1.1 Vấn Đề Của Monolith Cũ

| Vấn đề | Biểu hiện |
|--------|-----------|
| **Single DB** | 23 bảng dùng chung → 1 service lỗi DB = toàn bộ sập |
| **Tight Coupling** | `PaymentController` gọi `BookingService` trực tiếp |
| **Scale đồng nhất** | Muốn scale Tour (nhiều người xem) phải scale cả Payment |
| **Deploy rủi ro** | Sửa 1 bug nhỏ = redeploy toàn bộ 200K+ dòng code |
| **Auth phân tán** | Mỗi controller tự parse JWT và query DB `users` |

## 1.2 Bản Đồ Tách Service — Nguyên Tắc

```
Nguyên tắc: 1 Service = 1 Bounded Context = 1 Database độc lập
Giao tiếp: REST (sync, Feign) | Kafka (async, event-driven)
```

## 1.3 Chi Tiết Từng Microservice

### 🔐 Service 1: `identity-service` (:8081) — DB: `tourism_identity`

**Trách nhiệm**: Xác thực + quản lý user — NGUỒN SỰ THẬT DUY NHẤT về identity

| HTTP | Endpoint | Nguồn từ Monolith |
|------|----------|-------------------|
| POST | `/api/auth/register` | `AuthController.register()` |
| POST | `/api/auth/login` | `AuthController.login()` |
| POST | `/api/auth/google/login` | `AuthController.googleLogin()` |
| GET | `/api/auth/verify-email?token=` | `AuthController.verifyEmail()` |
| POST | `/api/auth/resend-verification?email=` | `AuthController.resendVerification()` |
| POST | `/api/auth/refresh-token` | `AuthController.refreshToken()` |
| POST | `/api/auth/logout` | `AuthController.logout()` |
| POST | `/api/auth/logout-all` | `AuthController.logoutAll()` |
| GET | `/api/auth/profile` | `AuthController.getMyProfile()` |
| GET | `/api/users/{id}` | `UserController.getUserById()` |
| PUT | `/api/users/{id}/profile` | `UserController.updateUser()` |
| POST | `/api/admin/users/search` | `UserController.searchUsers()` |
| PATCH | `/api/admin/users/{id}/status` | `UserController.updateUserStatus()` |

**DB Tables**: `users`, `refresh_tokens`, `email_verifications`  
**Integrations**: Cloudinary (avatar), JavaMail (verify email), Google API (OAuth2)  
**Kafka Producer**: `user.registered`

---

### 🗺️ Service 2: `tour-catalog-service` (:8082) — DB: `tourism_catalog`

**Trách nhiệm**: Danh mục tour, lịch khởi hành, địa điểm, tour yêu thích

| HTTP | Endpoint | Nguồn từ Monolith |
|------|----------|-------------------|
| GET | `/api/tours` | `TourController.getAllTours()` |
| GET | `/api/tours/search` | `TourController.searchTours()` |
| GET | `/api/tours/featured` | `TourController.getTop10DeepestDiscountTours()` |
| GET | `/api/tours/{code}` | `TourController.getTourDetail()` |
| GET | `/api/tours/{code}/related` | `TourController.getRelatedTours()` |
| GET | `/api/tours/{id}/departures` | `TourController.getTourDepartures()` |
| GET | `/api/tours/{id}/departures/available` | *(mới)* |
| GET | `/api/locations` | `LocationController` |
| POST | `/api/tours/favorites` | `FavoriteTourController.addFavoriteTour()` |
| DELETE | `/api/tours/favorites/{tourId}` | `FavoriteTourController.removeFavoriteTour()` |
| GET | `/api/tours/favorites/my` | `FavoriteTourController.getUserFavoriteTours()` |
| POST | `/api/admin/tours` | `TourManagementController.createTour()` |
| PUT | `/api/admin/tours/{id}` | `TourManagementController.updateTour()` |
| PUT | `/api/admin/tours/{id}/general-info` | `TourManagementController.updateGeneralInfo()` |
| PUT | `/api/admin/tours/{id}/itinerary` | `TourManagementController.updateItinerary()` |
| DELETE | `/api/admin/tours/{id}` | `TourManagementController.deleteTour()` |
| GET | `/api/admin/tours` | `TourManagementController.getAllTours()` |
| POST | `/api/admin/tours/{id}/thumbnail` | `TourMediaController` |
| POST | `/api/admin/tours/{id}/images` | `TourMediaController` |
| POST | `/api/admin/departures` | `TourDepartureManagementController.createDeparture()` |
| PUT | `/api/admin/departures/{id}` | `TourDepartureManagementController.updateDeparture()` |
| PUT | `/api/admin/departures/{id}/pricing` | `TourDepartureManagementController.updatePricing()` |
| PUT | `/api/admin/departures/{id}/transport` | `TourDepartureManagementController.updateTransport()` |
| POST | `/api/admin/departures/{id}/clone` | `TourDepartureManagementController.cloneDeparture()` |
| DELETE | `/api/admin/departures/{id}` | `TourDepartureManagementController.deleteDeparture()` |
| GET | `/internal/tours/{id}` | *(internal-only for booking-service)* |

**DB Tables**: `tours`, `tour_images`, `tour_departures`, `departure_pricings`, `departure_transports`, `itinerary_days`, `locations`, `favorite_tours`  
**Integrations**: Cloudinary (ảnh tour)

---

### 📦 Service 3: `booking-service` (:8083) — DB: `tourism_booking`

**Trách nhiệm**: Tạo và quản lý booking, xử lý hoàn tiền

| HTTP | Endpoint | Nguồn từ Monolith |
|------|----------|-------------------|
| GET | `/api/bookings/order?tourCode=&departureId=` | `BookingController.getBookingInitInfo()` |
| POST | `/api/bookings` | `BookingController.createBooking()` |
| GET | `/api/bookings/my?status=` | `BookingController.getAllBookingsByUser()` |
| GET | `/api/bookings/code/{code}` | `BookingController.getBookingDetail()` |
| POST | `/api/bookings/{id}/cancel` | `BookingController.cancelBooking()` |
| POST | `/api/bookings/{id}/refund-request` | `BookingController.requestRefund()` |
| POST | `/api/admin/bookings/search` | `BookingController.searchBookings()` |
| PATCH | `/api/admin/bookings/{id}/status` | `BookingController.updateBookingStatus()` |
| POST | `/internal/bookings/{code}/confirm` | *(payment-service gọi)* |
| GET | `/internal/bookings/check-confirmed` | *(review-service gọi)* |

**DB Tables**: `bookings`, `booking_passengers`, `refund_information`  
**Feign Clients**: → `tour-catalog-service` (lấy departure info), → `promotion-service` (validate coupon)  
**Redis**: cache booking by code  
**Kafka Producer**: `booking.created`, `booking.confirmed`, `booking.cancelled`

---

### 💳 Service 4: `payment-service` (:8084) — DB: `tourism_payment`

| HTTP | Endpoint | Nguồn từ Monolith |
|------|----------|-------------------|
| POST | `/api/payments/vnpay/create` | `PaymentController.createVNPayPayment()` |
| GET | `/api/payments/vnpay-callback` | `PaymentController.vnpayCallback()` |
| POST | `/api/payments/payos/create` | `PaymentController.createPayOSPayment()` |
| POST | `/api/payments/payos/webhook` | `PaymentController.handlePayOSWebhook()` |
| GET | `/api/payments/payos/return` | `PaymentController.payosReturn()` |
| GET | `/api/payments/payos/cancel` | `PaymentController.payosCancel()` |
| GET | `/api/payments/status/{orderCode}` | `PaymentController.getPaymentStatus()` |

**DB Tables**: `payments`  
**Feign**: → `booking-service /internal/bookings/{code}/confirm`  
**Kafka Producer**: `payment.completed`

---

### ⭐ Service 5: `review-service` (:8085) — DB: `tourism_review`

| HTTP | Endpoint | Nguồn từ Monolith |
|------|----------|-------------------|
| POST | `/api/reviews` (multipart) | `ReviewController.submitReview()` |
| GET | `/api/reviews/booking/{bookingCode}` | `ReviewController.getReview()` |
| GET | `/api/reviews/tour/{code}` | `ReviewController.getReviewsByTour()` |
| GET | `/api/reviews/tour/{code}/summary` | `ReviewController.getReviewStatistics()` |
| GET | `/api/reviews/my` | *(mới)* |
| DELETE | `/api/reviews/{id}` | *(mới)* |
| DELETE | `/api/admin/reviews/{id}` | *(mới)* |

**DB Tables**: `reviews`, `review_images`  
**Feign**: → `booking-service /internal/bookings/check-confirmed`  
**Cloudinary**: upload ảnh review  
**Kafka Producer**: `review.created`

---

### 🎟️ Service 6: `promotion-service` (:8086) — DB: `tourism_promotion`

| HTTP | Endpoint | Nguồn từ Monolith |
|------|----------|-------------------|
| GET | `/api/promotions/coupons/{code}` | *(user validate)* |
| POST | `/api/admin/coupons` | `CouponController.createCoupon()` |
| PUT | `/api/admin/coupons/{id}` | `CouponController.updateCoupon()` |
| DELETE | `/api/admin/coupons/{id}` | `CouponController.deleteCoupon()` |
| GET | `/api/admin/coupons` | `CouponController.getAllCoupons()` |
| GET | `/api/admin/coupons/search?keyword=` | `CouponController.searchCoupons()` |
| POST | `/internal/coupons/{code}/apply` | *(booking-service gọi)* |

**DB Tables**: `coupons`

---

### 🔔 Service 7: `notification-service` (:8087) — DB: `tourism_notification`

| HTTP | Endpoint |
|------|----------|
| GET | `/api/notifications/my` |
| GET | `/api/notifications/unread-count` |
| PUT | `/api/notifications/{id}/read` |
| PUT | `/api/notifications/read-all` |

**Nguồn từ Monolith**: `NotificationController` (4 endpoints)  
**Kafka Consumer**: `user.registered`, `booking.created`, `booking.confirmed`, `booking.cancelled`, `payment.completed`  
**DB Tables**: `notifications`, `user_notifications`  
**JavaMail**: Gửi email HTML

---

### 📊 Service 8: `analytics-service` (:8088) — DB: `tourism_analytics`

| HTTP | Endpoint | Nguồn từ Monolith |
|------|----------|-------------------|
| GET | `/api/analytics/dashboard/summary` | `DashboardController.getDashboardStatistics()` |
| GET | `/api/analytics/dashboard/daily-stats` | `DashboardController.getDashboardAIAnalysis()` |
| GET | `/api/analytics/dashboard/top-tours` | *(mới)* |
| POST | `/api/analytics/chatbot/chat` | `ChatbotController` |

**Kafka Consumer**: tất cả events để cập nhật stats  
**Google Gemini API**: AI chatbot

---

### 📄 Service 9: `cms-service` (:8089) — DB: `tourism_cms`

| HTTP | Endpoint | Nguồn từ Monolith |
|------|----------|-------------------|
| GET | `/api/cms/policies` | `PolicyTemplateController` |
| POST | `/api/cms/policies` | `PolicyTemplateController` |
| GET | `/api/cms/branches` | `BranchContactController` |
| POST | `/api/cms/branches` | `BranchContactController` |

---

## 1.4 Kiến Trúc Authentication Tập Trung

```
┌─────────────────────────────────────────────────────────┐
│                      API GATEWAY :8080                   │
│                                                          │
│  1. PUBLIC routes → forward KHÔNG cần validate JWT:     │
│     GET /api/tours/**, GET /api/reviews/**               │
│     POST /api/auth/login, /api/auth/register             │
│     GET /api/payment/*-callback, POST /api/*/webhook     │
│                                                          │
│  2. PROTECTED routes → JWT Filter:                       │
│     - Extract Bearer token từ Authorization header       │
│     - Verify chữ ký với JWT_SECRET                       │
│     - Inject vào request headers:                        │
│         X-User-Id: 42                                    │
│         X-User-Email: user@example.com                   │
│         X-User-Role: USER / ADMIN                        │
│     - Forward đến downstream service                      │
│                                                          │
│  3. Downstream services: ĐỌC HEADER, không parse JWT     │
│     @RequestHeader("X-User-Id") Long userId              │
│     @PreAuthorize("hasRole('ADMIN')") — từ X-User-Role   │
└─────────────────────────────────────────────────────────┘

Token flow:
  FE → POST /api/auth/login → { accessToken(15min), refreshToken(7days) }
  FE lưu: accessToken → Redux state | refreshToken → localStorage
  401 xảy ra → axios interceptor → POST /api/auth/refresh-token → retry
```

---

## 1.5 Kafka Event Bus — Luồng Đầy Đủ

```
PRODUCER              TOPIC                CONSUMERS
────────              ──────               ─────────
identity-svc  →  user.registered      → notification-svc (welcome email)
booking-svc   →  booking.created      → notification-svc (email + in-app)
                                       → analytics-svc (new_bookings++)
booking-svc   →  booking.confirmed    → notification-svc (in-app notify)
                                       → analytics-svc (confirmed_bookings++)
booking-svc   →  booking.cancelled    → notification-svc (email + in-app)
                                       → analytics-svc (cancelled_bookings++)
payment-svc   →  payment.completed    → notification-svc (receipt email)
                                       → analytics-svc (total_revenue += amount)
review-svc    →  review.created       → analytics-svc (update avg_rating)
```

---

# PHẦN 2: TÁI CẤU TRÚC FRONTEND (REACT)

## 2.1 Stack Frontend Mới

```
Vite + React 18 + TypeScript   ← thay Create React App (chậm, deprecated)
Redux Toolkit                   ← state management (giữ từ FE cũ)
React Router v6                 ← routing (giữ từ FE cũ)
Axios + Interceptors            ← HTTP client → API Gateway :8080
Ant Design 5                    ← UI (giữ từ FE cũ)
Recharts                        ← Admin dashboard charts
SASS                            ← Styling
```

## 2.2 Cấu Trúc Thư Mục Module Hóa

```
tourism-frontend/src/
├── api/                        # API layer — TẤT CẢ qua http://localhost:8080
│   ├── axiosInstance.ts        # Base config + JWT interceptor + auto-refresh
│   ├── auth.api.ts             # → identity-service endpoints
│   ├── tours.api.ts            # → tour-catalog-service endpoints
│   ├── bookings.api.ts         # → booking-service endpoints
│   ├── payments.api.ts         # → payment-service endpoints
│   ├── reviews.api.ts          # → review-service endpoints
│   ├── promotions.api.ts       # → promotion-service endpoints
│   ├── notifications.api.ts    # → notification-service endpoints
│   ├── analytics.api.ts        # → analytics-service endpoints
│   └── cms.api.ts              # → cms-service endpoints
├── store/
│   ├── index.ts
│   ├── auth/authSlice.ts       # { user, accessToken, isAuthenticated }
│   ├── tours/toursSlice.ts
│   └── ui/uiSlice.ts           # { loading, notifications }
├── hooks/
│   ├── useAuth.ts
│   ├── useTours.ts
│   └── useBookings.ts
├── pages/
│   ├── public/                 # Không cần đăng nhập
│   │   ├── HomePage.tsx
│   │   ├── ToursPage.tsx
│   │   └── TourDetailPage.tsx
│   ├── auth/
│   │   ├── LoginPage.tsx
│   │   └── RegisterPage.tsx
│   ├── user/                   # Protected (USER role)
│   │   ├── BookingsPage.tsx
│   │   ├── ProfilePage.tsx
│   │   └── NotificationsPage.tsx
│   └── admin/                  # Protected (ADMIN role)
│       ├── DashboardPage.tsx
│       ├── ToursManagePage.tsx
│       ├── BookingsManagePage.tsx
│       ├── CouponsManagePage.tsx
│       └── UsersManagePage.tsx
├── components/
│   ├── layout/
│   │   ├── PublicLayout.tsx    # Header + Footer
│   │   └── AdminLayout.tsx     # Sidebar + Header
│   ├── common/
│   │   ├── ProtectedRoute.tsx  # Kiểm tra isAuthenticated
│   │   └── AdminRoute.tsx      # Kiểm tra role ADMIN
│   ├── tour/TourCard.tsx
│   ├── booking/BookingForm.tsx
│   └── chatbot/ChatbotWidget.tsx
├── types/                      # TypeScript interfaces
└── utils/formatters.ts
```

## 2.3 axiosInstance.ts — Cấu Hình Gateway

```typescript
// src/api/axiosInstance.ts
import axios from 'axios';
import { store } from '../store';
import { logout, setAccessToken } from '../store/auth/authSlice';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8080',
  timeout: 15000,
});

// Attach token
api.interceptors.request.use((config) => {
  const token = store.getState().auth.accessToken;
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Auto refresh khi 401
let isRefreshing = false;
let failedQueue: any[] = [];

api.interceptors.response.use(
  res => res,
  async (error) => {
    if (error.response?.status === 401 && !error.config._retry) {
      if (isRefreshing) {
        return new Promise((resolve, reject) =>
          failedQueue.push({ resolve, reject })
        ).then(token => {
          error.config.headers.Authorization = `Bearer ${token}`;
          return api(error.config);
        });
      }
      error.config._retry = true;
      isRefreshing = true;
      try {
        const refreshToken = localStorage.getItem('refreshToken');
        const { data } = await axios.post(
          `${api.defaults.baseURL}/api/auth/refresh-token`,
          { refreshToken }
        );
        store.dispatch(setAccessToken(data.accessToken));
        failedQueue.forEach(({ resolve }) => resolve(data.accessToken));
        return api(error.config);
      } catch {
        store.dispatch(logout());
        return Promise.reject(error);
      } finally {
        isRefreshing = false;
        failedQueue = [];
      }
    }
    return Promise.reject(error);
  }
);

export default api;
```

## 2.4 Chiến Lược Tái Sử Dụng Component Cũ (`client-side`)

| Component Cũ | Hành Động | Lý Do |
|--------------|-----------|-------|
| `HeaderComponent`, `FooterComponent` | **Giữ + refactor** | Logic đơn giản, chỉ cần đổi API URL |
| `TourDetailComponent` | **Giữ + refactor** | Port sang Vite, đổi service calls |
| `BookingPaymentComponent` | **Giữ + refactor** | Cập nhật URL payment endpoints |
| `AdminComponent` | **Viết lại hoàn toàn** | Cần tích hợp analytics microservice mới |
| `ChatbotWidget` | **Giữ + refactor** | Đổi API endpoint sang analytics-service |
| `Login`, `RegisterComponent` | **Giữ + refactor** | Chuẩn hoá lại với auth flow mới |
| `homPageComponent` | **Giữ + refactor** | Cập nhật tours.api.ts calls |
| API calls trong `services/` | **Viết lại hoàn toàn** | Tập trung vào 1 axiosInstance duy nhất |

---

# PHẦN 3: MASTER PLAN 28 NGÀY — TỪ CON SỐ 0 ĐẾN DEPLOY AWS

## Timeline Overview

```
Tuần 1 (D1–D7):  Setup + Infrastructure + Identity Service
Tuần 2 (D8–D14): Tour Catalog + Booking + Payment + Notification
Tuần 3 (D15–D21): Review + Analytics + CMS + Frontend khởi động
Tuần 4 (D22–D28): Frontend hoàn thiện + Dockerize + Deploy AWS
```

---

## 🏗️ TUẦN 1: NỀN TẢNG & IDENTITY SERVICE

### Day 1 (18/03) — Môi Trường & Audit

**Buổi sáng**:
- `docker-compose up -d` → verify 11 containers tại Eureka `:8761`
- Mở DBeaver → kết nối PostgreSQL `:5432` → verify 9 databases
- Test `GET http://localhost:8080/api/tours` qua API Gateway

**Buổi chiều**:
- Đọc toàn bộ source code monolith `Tourism_Backend`
- Tạo `gap-analysis.md` trong project listing từng endpoint đã có/chưa có
- Tạo Postman Collection với folder cho từng service

**File cần tạo**: `D:\KLTN\gap-analysis.md`

---

### Day 2 (19/03) — Identity Service: Core Auth

**Code**:
- `AuthController.java` — 4 endpoints: register, login, refresh-token, logout
- `AuthService.java` — BCrypt, JWT generate, refreshToken lưu DB
- `JwtService.java` — generate/validate JWT (HS256, secret từ `.env`)
- `RefreshTokenEntity` + `RefreshTokenRepository`

**Config** `application.yml`:
```yaml
spring.datasource.url: jdbc:postgresql://postgres:5432/tourism_identity
jwt.secret: ${JWT_SECRET}
jwt.access-token-expiry: 900000     # 15 phút
jwt.refresh-token-expiry: 604800000 # 7 ngày
```

**Test**: `POST /api/auth/register` → `POST /api/auth/login` → nhận token

---

### Day 3 (20/03) — Identity Service: Email + Google OAuth

**Code**:
- `EmailService.java` — JavaMail SMTP, HTML email template
- `GET /api/auth/verify-email?token=` — UUID token 24h
- `POST /api/auth/resend-verification?email=`
- `GoogleAuthService.java` — verify Google ID Token → tạo/update user
- `POST /api/auth/google/login`

**Config** `application.yml`:
```yaml
spring.mail.host: smtp.gmail.com
spring.mail.port: 587
mail.username: ${MAIL_USERNAME}
google.client-id: ${GOOGLE_CLIENT_ID}
```

---

### Day 4 (21/03) — Identity Service: User Profile + Admin APIs

**Code**:
- `GET /api/auth/profile` — đọc `X-User-Id` header
- `PUT /api/users/{id}/profile` — multipart, Cloudinary avatar upload
- `POST /api/admin/users/search` — pagination + filter
- `PATCH /api/admin/users/{id}/status` — ACTIVE/LOCKED
- `@PreAuthorize("hasRole('ADMIN')")` cho admin endpoints

**Test**: Full identity flow với Postman Collection

---

### Day 5 (22/03) — API Gateway: JWT Filter + Routing

**Config** `application.yml` (api-gateway):
```yaml
spring.cloud.gateway.routes:
  - id: identity
    uri: lb://identity-service
    predicates: [Path=/api/auth/**, /api/users/**]
  - id: catalog
    uri: lb://tour-catalog-service
    predicates: [Path=/api/tours/**, /api/locations/**]
  - id: booking
    uri: lb://booking-service
    predicates: [Path=/api/bookings/**]
  # ... tất cả routes

# Public paths (skip JWT):
gateway.public-paths:
  - /api/auth/**
  - GET:/api/tours/**
  - GET:/api/reviews/**
  - GET:/api/cms/**
  - /api/payments/*-callback
  - /api/payments/*/webhook
```

**Code** `JwtAuthFilter.java`:
- Kiểm tra public path → skip
- Parse JWT → set headers: `X-User-Id`, `X-User-Email`, `X-User-Role`
- Invalid → 401

**Test CORS**: `allowed-origins: http://localhost:5173`

---

### Day 6 (23/03) — Shared Libs + Kafka Setup

**`common-security`**:
- `JwtAuthenticationFilter.java` — đọc `X-User-Id`, `X-User-Role` headers → set `SecurityContext`
- `UserPrincipal.java` — `{ id, email, role }`
- `SecurityAutoConfiguration.java` — auto-config cho tất cả services

**`common-events`**:
```java
public record BookingCreatedEvent(String bookingCode, Long userId, String tourCode, BigDecimal amount) {}
public record PaymentCompletedEvent(String bookingCode, Long userId, BigDecimal amount) {}
public record ReviewCreatedEvent(String tourCode, Long tourId, Integer rating) {}
// v.v.
```

**Test Kafka**: produce test message → consumer nhận được

---

### Day 7 (24/03) — Buffer + E2E Test Tuần 1

- E2E: Register → Verify Email → Login (Google) → Get Profile → Change Password
- Fix bugs
- **Git commit**: `git tag v0.1-identity`

---

## 🚀 TUẦN 2: TOUR + BOOKING + PAYMENT + NOTIFICATION

### Day 8 (25/03) — Tour Catalog: Public APIs + DB Schema

**Code**: Flyway migration V1__init.sql cho `tourism_catalog`  
**Entities**: `Tour`, `TourDeparture`, `Location`, `ItineraryDay`, `DeparturePricing`  
**APIs**:
- `GET /api/tours` — paginated, filter `?region=&location=&keyword=`
- `GET /api/tours/search` — full text search
- `GET /api/tours/featured` — sort by is_featured / booking_count
- `GET /api/tours/{code}` — detail + itinerary
- `GET /api/tours/{id}/departures/available`
- `GET /api/locations`

---

### Day 9 (26/03) — Tour Catalog: Admin + Image APIs

**Code**:
- `POST /api/admin/tours` — tạo tour + itinerary
- `PUT /api/admin/tours/{id}/itinerary`
- `POST /api/admin/tours/{id}/thumbnail` — Cloudinary multipart
- `POST /api/admin/tours/{id}/images`
- `POST /api/admin/departures` — tạo departure
- `PUT /api/admin/departures/{id}/pricing`
- `PUT /api/admin/departures/{id}/transport`
- `POST /api/admin/departures/{id}/clone`
- Favorite tour APIs (3 endpoints)
- `GET /internal/tours/{id}` — cho booking-service Feign call

---

### Day 10 (27/03) — Booking Service: Tạo Booking

**Code**: Flyway migrations cho `tourism_booking`  
**Entities**: `Booking`, `BookingPassenger`, `RefundInformation`

**`POST /api/bookings` — luồng**:
```
1. Validate departure còn chỗ (Feign → /internal/tours/{id})
2. Validate coupon nếu có (Feign → /internal/coupons/{code}/apply)
3. Tính total = Σ(passengers * price) - discount
4. Lưu Booking status=PENDING_PAYMENT
5. Giảm available_slots tại tour-catalog (Feign PUT)
6. Kafka publish: booking.created
7. Redis cache booking by code
```

---

### Day 11 (28/03) — Booking Service: Cancel + Internal APIs

**Code**:
- `GET /api/bookings/my?status=` — đọc `X-User-Id` header
- `POST /api/bookings/{id}/cancel` → Kafka `booking.cancelled`
- `POST /api/bookings/{id}/refund-request`
- Admin search + update status
- `POST /internal/bookings/{code}/confirm` → CONFIRMED → Kafka `booking.confirmed`
- `GET /internal/bookings/check-confirmed?userId=&tourCode=`

**Test**: Feign call booking → catalog, booking → promotion

---

### Day 12 (29/03) — Payment Service

**Code** (port từ `PaymentController.java` monolith):
- VNPay: `POST /create` → generate URL → callback → confirm booking (Feign) → Kafka
- PayOS: `POST /create` → QR link → webhook → redirect
- Redirect về FE: `http://localhost:5173/payment-success?bookingCode=...`

⚠️ **Cài ngrok ngay**: `ngrok http 8080` → dùng URL ngrok cho VNPay/PayOS sandbox callback

---

### Day 13 (30/03) — Promotion + Notification Services

**Promotion Service**: CRUD coupon (port từ `CouponController.java`)

**Notification Service — Kafka Consumers**:
```java
@KafkaListener(topics = "booking.created")
void onBookingCreated(BookingCreatedEvent event) {
    emailService.sendBookingConfirmationEmail(event);
    notificationRepo.save(new Notification("Đặt tour thành công", event.userId()));
}
// Tương tự cho 4 events còn lại
```

**REST APIs**: 4 notification endpoints

---

### Day 14 (31/03) — Buffer + Integration Test Tuần 2

- Full flow: Login → Tìm tour → Đặt → Thanh toán VNPay → Nhận email → Xem booking
- Fix Kafka consumer bugs (thường lỗi offset/deserialization)
- **Git commit**: `git tag v0.2-core`

---

## 🔧 TUẦN 3: REVIEW + ANALYTICS + CMS + FRONTEND KHỞI ĐỘNG

### Day 15 (01/04) — Review Service

**Code**:
- `POST /api/reviews` — check booking confirmed (Feign) → upload Cloudinary → Kafka `review.created`
- `GET /api/reviews/tour/{code}` — paginated (public)
- `GET /api/reviews/tour/{code}/summary` — avg rating + star distribution
- Eligibility check + My reviews + Delete

---

### Day 16 (02/04) — Analytics Service + AI Chatbot

**Kafka Consumers**: update `daily_stats` và `tour_stats` từ mọi events  
**Dashboard APIs**: summary, daily-stats (7 ngày), top-tours  
**Chatbot**: Google Gemini API

---

### Day 17 (03/04) — CMS Service + Backend Complete

- `cms-service`: PolicyTemplate + BranchContact CRUD
- Chuẩn hoá error format: `{ success, message, data, errors }` cho tất cả services
- `docker-compose up -d` → verify 100% healthy
- **BACKEND HOÀN THIỆN — `git tag v1.0-backend`**

---

### Day 18 (04/04) — Setup Vite Frontend

```bash
cd D:\KLTN
npm create vite@latest tourism-frontend -- --template react-ts
cd tourism-frontend
npm install axios @reduxjs/toolkit react-redux react-router-dom antd @ant-design/icons
npm install lucide-react react-toastify recharts swiper @react-oauth/google
npm install @stomp/stompjs sockjs-client date-fns react-datepicker
npm install -D sass @types/node
```

- Tạo cấu trúc thư mục (Phần 2.2)
- Tạo `axiosInstance.ts` (Phần 2.3)
- Setup Redux store + React Router layouts

---

### Day 19 (05/04) — Frontend: Auth Pages

- `LoginPage` + `RegisterPage` → kết nối identity-service
- Google OAuth: `GoogleOAuthProvider` + `GoogleLogin` button
- `ProtectedRoute` + `AdminRoute` components
- Test login flow trong browser

---

### Day 20 (06/04) — Frontend: Homepage + Tours List

- `HomePage`: Hero banner, featured tours, destinations
- `ToursPage`: Filter sidebar + grid + pagination
- `TourCard` component với hover animation

---

### Day 21 (07/04) — Buffer + Review Tuần 3

- Fix bugs, responsive mobile check
- **`git commit` FE progress**

---

## 🎨 TUẦN 4: FRONTEND HOÀN THIỆN + DOCKERIZE + AWS DEPLOY

### Day 22 (08/04) — Tour Detail + Booking Flow

- `TourDetailPage`: gallery (Swiper), itinerary tabs, departure table, booking panel
- `BookingPage`: passenger form → `POST /api/bookings`

---

### Day 23 (09/04) — Payment + My Bookings

- `PaymentPage`: chọn phương thức → `POST /api/payments/vnpay/create | payos/create`
- `PaymentResultPage`: parse URL params → success/fail UI + confetti
- `MyBookingsPage`: tabs by status + cancel + refund request

---

### Day 24 (10/04) — Reviews + Notifications + Profile

- Reviews section trong TourDetail + WriteReviewModal
- `NotificationsPage` + unread badge trong header
- `ProfilePage`: edit info + change password + avatar upload

---

### Day 25 (11/04) — Admin Dashboard + Management Pages

- `DashboardPage`: stats cards + Recharts + AI chatbot widget
- `ToursManagePage`: Ant Design Table + CRUD modal + upload ảnh
- `BookingsManagePage` + `CouponsManagePage` + `UsersManagePage`

---

### Day 26 (12/04) — Polish + E2E Testing

- Responsive: 375px, 768px, 1024px, 1440px
- Skeleton loading, error boundaries (404/403/500)
- Full E2E test: checklist 20 scenarios
- Fix critical bugs

---

### Day 27 (13/04) — Dockerize Toàn Bộ Hệ Thống

**Dockerfile cho Frontend** (`tourism-frontend/Dockerfile`):
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

**nginx.conf** (SPA routing):
```nginx
server {
    listen 80;
    location / {
        root /usr/share/nginx/html;
        try_files $uri /index.html;  # React Router SPA support
    }
    location /api {
        proxy_pass http://api-gateway:8080;  # Forward API calls
    }
}
```

**Thêm vào `docker-compose.yml`**:
```yaml
  frontend:
    build:
      context: ../tourism-frontend
      args:
        VITE_API_URL: http://localhost:8080
    ports:
      - "80:80"
    depends_on: [api-gateway]
    networks: [tourism-net]
```

**Verify**: `docker-compose up --build -d` → toàn bộ 12 services chạy

---

### Day 28 (14/04) — Deploy AWS 🚀

#### Kiến Trúc AWS Target

```
Internet
   │
   ▼
AWS Application Load Balancer (ALB)
   ├── /api/**  → Backend (API Gateway container)
   └── /**      → Frontend (S3 + CloudFront hoặc Nginx container)
         │
    EC2 Instance (t3.medium) hoặc ECS Fargate
    ├── docker-compose hoặc ECS Task Definitions
    ├── Tất cả 11 containers chạy
    └── Mount EBS volume cho Kafka/Postgres data
         │
    AWS RDS PostgreSQL (db.t3.micro)
    └── 9 databases: tourism_identity, tourism_catalog, ...
         │
    AWS ElastiCache Redis (cache.t3.micro)
    └── Booking cache
```

#### Quy Trình Deploy (Solo Dev — 1 ngày)

**Bước 1: Provision AWS Resources** (1-2 tiếng)
```bash
# 1. Tạo EC2 Instance: Ubuntu 22.04, t3.medium (2CPU, 4GB RAM)
# 2. Security Group: mở port 80, 443, 8080, 8761 (chỉ internal)
# 3. RDS PostgreSQL: db.t3.micro, Multi-AZ = false (KLTN không cần HA)
# 4. ElastiCache Redis: cache.t3.micro, single node
# 5. Elastic IP cho EC2
```

**Bước 2: Setup EC2** (30 phút)
```bash
ssh -i kltn.pem ubuntu@<EC2_PUBLIC_IP>

# Cài Docker + Docker Compose
sudo apt update && sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker ubuntu

# Clone project
git clone https://github.com/yourname/tourism-microservices.git
cd tourism-microservices
```

**Bước 3: Cấu Hình Environment** (30 phút)
```bash
cp .env.example .env
vim .env
# Điền:
# DB_HOST=<RDS_ENDPOINT>          (thay postgres container)
# REDIS_HOST=<ELASTICACHE_ENDPOINT> (thay redis container)
# JWT_SECRET=<random-64-chars>
# MAIL_USERNAME=...
# CLOUDINARY_CLOUD_NAME=...
# VNPAY_TMN_CODE=...
# PAYOS_CLIENT_ID=...
# GEMINI_API_KEY=...
# VITE_API_URL=http://<EC2_PUBLIC_IP>
```

**Bước 4: Build & Deploy** (1-2 tiếng)
```bash
# Build tất cả images
docker compose build

# Chạy infrastructure trước
docker compose up -d zookeeper kafka zipkin service-registry

# Đợi Eureka healthy (30s) rồi chạy services
sleep 30
docker compose up -d api-gateway identity-service tour-catalog-service

# Chạy tiếp các services
docker compose up -d booking-service payment-service review-service
docker compose up -d promotion-service notification-service analytics-service cms-service

# Chạy frontend
docker compose up -d frontend
```

**Bước 5: Cấu Hình ALB + Domain** (30 phút)
```
ALB Target Groups:
  - frontend-tg  → EC2:80
  - backend-tg   → EC2:8080  (path-based: /api/*)

Route 53 (nếu có domain):
  tourism.yourdomain.com → ALB DNS

VNPay/PayOS webhook URL:
  https://tourism.yourdomain.com/api/payments/vnpay-callback
  https://tourism.yourdomain.com/api/payments/payos/webhook
```

**Bước 6: Smoke Test Production**
```bash
# Test API Gateway
curl https://tourism.yourdomain.com/api/tours

# Test Auth
curl -X POST https://tourism.yourdomain.com/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@test.com","password":"123456"}'

# Monitor
docker compose logs -f api-gateway
# Zipkin: http://<EC2_IP>:9411
# Eureka: http://<EC2_IP>:8761
```

#### Chi Phí AWS Ước Tính (KLTN — 1 tháng)

| Service | Loại | Chi Phí/Tháng |
|---------|------|----------------|
| EC2 t3.medium | On-Demand | ~$30 |
| RDS db.t3.micro | PostgreSQL | ~$15 |
| ElastiCache t3.micro | Redis | ~$12 |
| ALB | Application LB | ~$18 |
| **Tổng** | | **~$75/tháng** |

> 💡 **Mẹo tiết kiệm**: Dùng EC2 t3.small (~$15) + chạy PostgreSQL và Redis trong Docker container trên cùng EC2 để tiết kiệm $27/tháng → **~$33/tháng tổng**

---

## PHẦN 4: CHECKLIST HOÀN THÀNH

### ✅ Backend
- [ ] service-registry (Eureka) — running
- [ ] api-gateway — JWT filter + CORS + routing đúng
- [ ] identity-service — Auth + User + Google OAuth + Email verify
- [ ] tour-catalog-service — Tour + Departure + Location + Images + Favorites
- [ ] booking-service — Booking flow + Redis + Internal APIs
- [ ] payment-service — VNPay + PayOS + SePay webhooks
- [ ] review-service — Review + Rating summary + Cloudinary
- [ ] promotion-service — Coupon CRUD + validate
- [ ] notification-service — 5 Kafka consumers + email HTML
- [ ] analytics-service — Dashboard stats + AI Chatbot (Gemini)
- [ ] cms-service — Policy + Branch

### ✅ Frontend
- [ ] Auth pages (Login + Register + Verify Email)
- [ ] Home + Tours list với filter
- [ ] Tour detail (gallery + itinerary + reviews + booking form)
- [ ] Booking flow (form → payment → result)
- [ ] My Bookings + cancel + refund
- [ ] Profile + notifications
- [ ] Admin Dashboard + CRUD tables
- [ ] Responsive mobile
- [ ] Error handling + loading states

### ✅ DevOps / Deploy
- [ ] Dockerfile cho tất cả services
- [ ] docker-compose.yml hoàn chỉnh với frontend
- [ ] `.env` đầy đủ tất cả secrets
- [ ] EC2 instance running
- [ ] Domain / ALB configured
- [ ] Smoke test production pass

---

> 📌 **Tài liệu liên quan**:
> - API Mapping chi tiết: `KLTN_ARCHITECTURE_PART1.md`
> - DB Schema đầy đủ: `KLTN_ARCHITECTURE_PART2.md`
> - Project microservices: `D:\KLTN\tourism-microservices`
