# Tourism Microservices Platform
## Implementation Plan & Architecture Reference

> **Author**: Senior Solution Architect  
> **Project**: KLTN — Tourism Management System  
> **Strategy**: Monolith → Microservices Migration  
> **Monolith source**: `D:\KLTN\Tourism_Backend`  
> **Target project**: `D:\KLTN\tourism-microservices`  
> **Last updated**: 10/04/2026

---

## Table of Contents

1. [Monolith Analysis](#1-monolith-analysis)
2. [Microservices Architecture](#2-microservices-architecture)
3. [Authentication Architecture](#3-authentication-architecture)
4. [API Mapping: Monolith → Microservices](#4-api-mapping-monolith--microservices)
5. [Inter-Service Communication](#5-inter-service-communication)
6. [Kafka Event Bus](#6-kafka-event-bus)
7. [Infrastructure & External Integrations](#7-infrastructure--external-integrations)
8. [Database Decomposition](#8-database-decomposition)
9. [28-Day Master Plan](#9-28-day-master-plan)
10. [Risk Register](#10-risk-register)
11. [Completion Checklist](#11-completion-checklist)

---

## 1. Monolith Analysis

### 1.1 Current State — `Tourism_Backend`

**Tech Stack**: Spring Boot 3.3 · PostgreSQL · Spring Security · JJWT 0.11.5 · Kafka · Redis · Cloudinary · VNPay · PayOS · SePay · Google API · JavaMail · Flyway · WebSocket

**Problem Statement**

| Problem | Symptom in Monolith |
|---------|-------------------|
| **Shared Database** | 23 tables in a single schema — one failing query can lock the entire system |
| **Tight Coupling** | `PaymentController` calls `BookingService` directly in-process |
| **Uniform Scaling** | Cannot scale Tour Catalog independently from Payment |
| **Risky Deployment** | Fixing one bug requires redeploying 200K+ lines of code |
| **Scattered Auth Logic** | Every controller parses JWT independently |
| **No Domain Isolation** | `Booking` entity has direct FK to `User`, `Tour`, `Coupon`, `Payment` |

### 1.2 Monolith Controller Inventory

```
Tourism_Backend/src/main/java/com/tourism/backend/controller/
├── Auth & User
│   ├── AuthController.java             # Login, register, refresh, logout, email verify, Google OAuth
│   ├── AdminAuthController.java        # Admin login, admin profile
│   ├── AdminProfileController.java     # Admin profile management
│   └── UserController.java             # User CRUD, profile update, change password
│
├── Tour
│   ├── TourController.java             # Public tour listing, search, detail, featured
│   ├── TourManagementController.java   # Admin: create/update/delete tour
│   ├── TourDepartureManagementController.java  # Admin: departure CRUD + clone
│   ├── TourMediaController.java        # Upload tour thumbnail
│   ├── TourUploadController.java       # Bulk image upload
│   └── FavoriteTourController.java     # User favorites
│
├── Booking & Payment
│   ├── BookingController.java          # Create, list, cancel, refund request
│   └── PaymentController.java          # VNPay, PayOS, SePay integration
│
├── Review
│   └── ReviewController.java           # Submit review + listing
│
├── Promotion
│   └── CouponController.java           # Coupon CRUD + validation
│
├── Location
│   ├── LocationController.java         # Public location listing
│   └── LocationAdminController.java    # Admin location CRUD
│
├── CMS
│   ├── PolicyTemplateController.java   # Policy CRUD
│   └── BranchContactController.java    # Branch/office CRUD
│
├── Notification
│   └── NotificationController.java     # In-app notifications
│
└── Analytics
    ├── DashboardController.java        # Admin dashboard summary
    └── ChatbotController.java          # AI chatbot via Gemini API
```

### 1.3 Monolith Entity Inventory

```
23 Entities → will be distributed across 9 microservice databases:

Auth/Identity   : User, RefreshToken
Tour            : Tour, TourImage, TourMedia, TourDeparture,
                  DeparturePricing, DepartureTransport, ItineraryDay,
                  Location, FavoriteTour
Booking         : Booking, BookingPassenger, RefundInformation
Payment         : Payment
Review          : Review, ImageReview
Promotion       : Coupon
CMS             : PolicyTemplate, BranchContact
Notification    : Notification, UserNotification
```

---

## 2. Microservices Architecture

### 2.1 Decomposition Principle

```
One Service  =  One Bounded Context  =  One PostgreSQL Database
Communication: REST sync (Feign) | Kafka async (Event-Driven)
```

### 2.2 Service Topology

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          REACT FRONTEND (:5173)                             │
│                     All requests → http://localhost:8080                    │
└──────────────────────────────────┬──────────────────────────────────────────┘
                                   │ HTTPS
                                   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│              API GATEWAY  (:8080)  —  Spring Cloud Gateway                  │
│   • JWT Filter: validate token → inject X-User-Id / X-User-Role headers     │
│   • Route table: path-based routing to downstream services                  │
│   • CORS: allow http://localhost:5173                                        │
└──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬─────────────┘
       │      │      │      │      │      │      │      │      │
       ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
  :8081  :8082  :8083  :8084  :8085  :8086  :8087  :8088  :8089
identity tour  booking payment review promo  notif  analy  cms
                                              │      │      │
                                         ─────────────────────
                                         Apache Kafka (:9092)
                                         ─────────────────────
       │      │      │      │      │      │      │      │      │
       ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
  tourism_ tourism_ tourism_ tourism_ tourism_ tourism_ tourism_ tourism_ tourism_
  identity catalog booking payment review promotion notification analytics cms
     (PG)    (PG)    (PG)    (PG)    (PG)   (PG)      (PG)       (PG)    (PG)
```

### 2.3 Service Catalogue

| # | Service | Port | Database | Spring Boot |
|---|---------|------|----------|-------------|
| — | `service-registry` (Eureka) | **8761** | — | Spring Cloud Netflix Eureka |
| — | `api-gateway` | **8080** | — | Spring Cloud Gateway |
| 1 | `identity-service` | **8081** | `tourism_identity` | Spring Boot 3.3 |
| 2 | `tour-catalog-service` | **8082** | `tourism_catalog` | Spring Boot 3.3 |
| 3 | `booking-service` | **8083** | `tourism_booking` | Spring Boot 3.3 |
| 4 | `payment-service` | **8084** | `tourism_payment` | Spring Boot 3.3 |
| 5 | `review-service` | **8085** | `tourism_review` | Spring Boot 3.3 |
| 6 | `promotion-service` | **8086** | `tourism_promotion` | Spring Boot 3.3 |
| 7 | `notification-service` | **8087** | `tourism_notification` | Spring Boot 3.3 |
| 8 | `analytics-service` | **8088** | `tourism_analytics` | Spring Boot 3.3 |
| 9 | `cms-service` | **8089** | `tourism_cms` | Spring Boot 3.3 |

### 2.4 Project Structure

```
tourism-microservices/
├── pom.xml                        # Root Maven POM (parent for all modules)
├── docker-compose.yml             # Full stack: infra + all services
├── init-db.sql                    # Create 9 PostgreSQL databases on startup
├── .env / .env.example            # Shared environment variables
│
├── shared-libs/
│   ├── common-security/           # JWT filter + UserPrincipal (imported by all services)
│   └── common-events/             # Kafka event DTOs (imported by producers/consumers)
│
├── infrastructure/
│   ├── service-registry/          # Eureka Server
│   └── api-gateway/               # Spring Cloud Gateway + JWT GlobalFilter
│
└── services/
    ├── identity-service/
    ├── tour-catalog-service/
    ├── booking-service/
    ├── payment-service/
    ├── review-service/
    ├── promotion-service/
    ├── notification-service/
    ├── analytics-service/
    └── cms-service/
```

**Standard service layout** (every service follows this):
```
services/{service-name}/
├── pom.xml
├── Dockerfile
└── src/main/java/com/tourism/{domain}/
    ├── {Domain}ServiceApplication.java
    ├── controller/        # REST endpoints
    ├── service/           # Business logic
    ├── repository/        # JPA repositories
    ├── entity/            # JPA entities
    ├── dto/
    │   ├── request/       # Input DTOs
    │   └── response/      # Output DTOs
    ├── client/            # Feign clients (if needed)
    ├── event/             # Kafka producers/consumers
    ├── config/            # Security, Kafka, Redis, Cloudinary configs
    └── exception/         # GlobalExceptionHandler + custom exceptions
```

---

## 3. Authentication Architecture

### 3.1 JWT Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│  1. LOGIN                                                                │
│     Client → POST /api/auth/login → Gateway → identity-service          │
│     identity-service: verify BCrypt → generate JWT (15min access +      │
│                        7-day refresh) → return { accessToken,            │
│                        refreshToken, user }                              │
│                                                                          │
│  2. AUTHENTICATED REQUEST                                                │
│     Client → GET /api/bookings/my                                        │
│     [Headers: Authorization: Bearer <accessToken>]                       │
│           ↓                                                              │
│     API Gateway JwtAuthFilter:                                           │
│       • Is public path? → forward as-is                                 │
│       • Extract token → verify signature with JWT_SECRET                │
│       • Decode claims → inject downstream headers:                       │
│           X-User-Id: 42                                                  │
│           X-User-Email: user@example.com                                 │
│           X-User-Role: USER | ADMIN                                      │
│       → Forward to booking-service                                       │
│                                                                          │
│  3. SERVICE RECEIVES REQUEST                                             │
│     booking-service receives X-User-Id header                            │
│     common-security filter: set SecurityContext from headers             │
│     @GetMapping("/my") reads: auth.getName() = userId                   │
│                                                                          │
│  4. AUTO REFRESH                                                         │
│     Gateway returns 401 → axios interceptor:                             │
│       POST /api/auth/refresh-token { refreshToken }                     │
│       ← new { accessToken }                                              │
│       Retry original request                                             │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Public vs Protected Routes

```yaml
# api-gateway — routes that bypass JWT filter:
gateway.public-paths:
  - /api/auth/**                   # login, register, verify-email, google login
  - GET /api/tours/**              # public tour browsing
  - GET /api/reviews/**            # public review reading
  - GET /api/cms/**                # public CMS content
  - GET /api/locations/**          # public location list
  - GET /api/payments/vnpay-return # VNPay redirect callback
  - POST /api/payments/*-webhook   # Payment gateway webhooks
```

### 3.3 Role-Based Access

| Role | Access Scope |
|------|-------------|
| `ANONYMOUS` | All public routes |
| `USER` | + Booking, review, profile, notifications |
| `ADMIN` | + `/api/admin/**` all management endpoints |

---

## 4. API Mapping: Monolith → Microservices

### 4.1 Identity Service `:8081` ← `AuthController` + `UserController` + `AdminAuthController`

**Database**: `tourism_identity`  
**Entities**: `users`, `refresh_tokens`, `email_verifications`  
**Dependencies**: Cloudinary (avatar), JavaMail (email verify), Google API (OAuth2)  
**Kafka Producer**: `user.registered`

#### Auth Endpoints

| Method | Path | Auth | Source Monolith | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/auth/register` | Public | `AuthController.register()` | Đăng ký + gửi email xác thực |
| `POST` | `/api/auth/login` | Public | `AuthController.login()` | Đăng nhập, trả JWT pair |
| `POST` | `/api/auth/google/login` | Public | `AuthController.googleLogin()` | Google OAuth2 → JWT |
| `GET` | `/api/auth/verify-email?token=` | Public | `AuthController.verifyEmail()` | Kích hoạt tài khoản qua email |
| `POST` | `/api/auth/resend-verification` | Public | `AuthController.resendVerification()` | Gửi lại email xác thực |
| `POST` | `/api/auth/refresh-token` | Public | `AuthController.refreshToken()` | Silent token refresh |
| `POST` | `/api/auth/logout` | USER | `AuthController.logout()` | Huỷ refresh token hiện tại |
| `POST` | `/api/auth/logout-all` | USER | `AuthController.logoutAll()` | Huỷ tất cả sessions |
| `GET` | `/api/auth/profile` | USER | `AuthController.getMyProfile()` | Thông tin user từ X-User-Id |

#### User Endpoints

| Method | Path | Auth | Source Monolith | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/users/{id}` | USER | `UserController.getUserById()` | Xem thông tin user |
| `PATCH` | `/api/users/{id}/profile` | USER | `UserController.updateProfile()` | Cập nhật profile + avatar Cloudinary |
| `PATCH` | `/api/users/{id}/change-password` | USER | `UserController.changePassword()` | Đổi mật khẩu |
| `GET` | `/api/admin/users` | ADMIN | `UserController.getAllUsers()` | Danh sách users có pagination |
| `POST` | `/api/admin/users/search` | ADMIN | `UserController.searchUsers()` | Tìm kiếm user |
| `PATCH` | `/api/admin/users/{id}/status` | ADMIN | `UserController.updateStatus()` | Khoá / mở khoá tài khoản |

---

### 4.2 Tour Catalog Service `:8082` ← `TourController` + `TourManagementController` + `TourDepartureManagementController` + `FavoriteTourController`

**Database**: `tourism_catalog`  
**Entities**: `tours`, `tour_images`, `tour_departures`, `departure_pricings`, `departure_transports`, `itinerary_days`, `locations`, `favorite_tours`  
**Dependencies**: Cloudinary (tour images)

#### Public Tour Endpoints

| Method | Path | Auth | Source Monolith | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/tours` | Public | `TourController.getAllTours()` | Danh sách tour, filter + pagination |
| `GET` | `/api/tours/search` | Public | `TourController.searchTours()` | Full-text search |
| `GET` | `/api/tours/featured` | Public | `TourController.getTop10DeepestDiscountTours()` | Tour nổi bật / giảm giá sâu |
| `GET` | `/api/tours/code/{code}` | Public | `TourController.getTourDetail()` | Chi tiết tour theo code |
| `GET` | `/api/tours/{id}` | Public | `TourController.getTourDetail()` | Chi tiết tour theo ID |
| `GET` | `/api/tours/{id}/related` | Public | `TourController.getRelatedTours()` | Tour cùng khu vực |
| `GET` | `/api/tours/{id}/departures` | Public | `TourController.getTourDepartures()` | Tất cả lịch khởi hành |
| `GET` | `/api/tours/{id}/departures/available` | Public | *(New)* | Lịch khởi hành còn chỗ |
| `GET` | `/api/locations` | Public | `LocationController.getLocations()` | Danh sách địa điểm |

#### Authenticated Tour Endpoints

| Method | Path | Auth | Source Monolith | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/tours/favorites` | USER | `FavoriteTourController.addFavoriteTour()` | Thêm vào yêu thích |
| `DELETE` | `/api/tours/favorites/{tourId}` | USER | `FavoriteTourController.removeFavoriteTour()` | Xoá khỏi yêu thích |
| `GET` | `/api/tours/favorites/my` | USER | `FavoriteTourController.getUserFavoriteTours()` | Danh sách yêu thích |

#### Admin Tour Management

| Method | Path | Auth | Source Monolith | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/admin/tours` | ADMIN | `TourManagementController.getAllTours()` | Tất cả tour cho admin |
| `POST` | `/api/admin/tours` | ADMIN | `TourManagementController.createTour()` | Tạo tour mới |
| `PUT` | `/api/admin/tours/{id}` | ADMIN | `TourManagementController.updateTour()` | Cập nhật tour |
| `PUT` | `/api/admin/tours/{id}/general-info` | ADMIN | `TourManagementController.updateGeneralInfo()` | Cập nhật thông tin chung |
| `PUT` | `/api/admin/tours/{id}/itinerary` | ADMIN | `TourManagementController.updateItinerary()` | Cập nhật lịch trình |
| `PATCH` | `/api/admin/tours/{id}/status` | ADMIN | `TourManagementController.updateStatus()` | Ẩn/hiện tour |
| `DELETE` | `/api/admin/tours/{id}` | ADMIN | `TourManagementController.deleteTour()` | Xoá tour |
| `POST` | `/api/admin/tours/{id}/thumbnail` | ADMIN | `TourMediaController` | Upload ảnh bìa Cloudinary |
| `POST` | `/api/admin/tours/{id}/images` | ADMIN | `TourUploadController` | Upload nhiều ảnh gallery |
| `POST` | `/api/admin/departures` | ADMIN | `TourDepartureManagementController.createDeparture()` | Tạo lịch khởi hành |
| `PUT` | `/api/admin/departures/{id}` | ADMIN | `TourDepartureManagementController.updateDeparture()` | Cập nhật lịch |
| `PUT` | `/api/admin/departures/{id}/pricing` | ADMIN | `TourDepartureManagementController.updatePricing()` | Cập nhật giá |
| `PUT` | `/api/admin/departures/{id}/transport` | ADMIN | `TourDepartureManagementController.updateTransport()` | Cập nhật phương tiện |
| `POST` | `/api/admin/departures/{id}/clone` | ADMIN | `TourDepartureManagementController.cloneDeparture()` | Nhân bản lịch khởi hành |
| `DELETE` | `/api/admin/departures/{id}` | ADMIN | `TourDepartureManagementController.deleteDeparture()` | Xoá lịch khởi hành |
| `GET` | `/api/admin/locations` | ADMIN | `LocationAdminController.getAll()` | Quản lý địa điểm |
| `POST` | `/api/admin/locations` | ADMIN | `LocationAdminController.create()` | Thêm địa điểm |

#### Internal Endpoints (Feign only, not exposed via Gateway)

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `GET` | `/internal/tours/{id}` | booking-service | Lấy thông tin departure + giá |
| `PUT` | `/internal/tours/{departureId}/slots` | booking-service | Giảm available_slots sau booking |

---

### 4.3 Booking Service `:8083` ← `BookingController`

**Database**: `tourism_booking`  
**Entities**: `bookings`, `booking_passengers`, `refund_information`  
**Redis**: Cache booking by code (TTL 30 min)  
**Feign**: → tour-catalog-service, → promotion-service  
**Kafka Producer**: `booking.created`, `booking.confirmed`, `booking.cancelled`

| Method | Path | Auth | Source Monolith | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/bookings/order` | USER | `BookingController.getBookingInitInfo()` | Lấy thông tin khởi tạo booking |
| `POST` | `/api/bookings` | USER | `BookingController.createBooking()` | Tạo booking mới |
| `GET` | `/api/bookings/my` | USER | `BookingController.getAllBookingsByUser()` | Lịch sử booking của user |
| `GET` | `/api/bookings/code/{code}` | USER | `BookingController.getBookingDetail()` | Chi tiết booking theo mã |
| `POST` | `/api/bookings/{id}/cancel` | USER | `BookingController.cancelBooking()` | Huỷ booking |
| `POST` | `/api/bookings/{id}/refund-request` | USER | `BookingController.requestRefund()` | Yêu cầu hoàn tiền |
| `POST` | `/api/admin/bookings/search` | ADMIN | `BookingController.searchBookings()` | Tìm kiếm booking |
| `PATCH` | `/api/admin/bookings/{id}/status` | ADMIN | `BookingController.updateBookingStatus()` | Cập nhật trạng thái |

**Booking creation flow**:
```
POST /api/bookings { tourCode, departureId, couponCode?, passengers[] }
  1. Feign → tour-catalog  GET /internal/tours/{departureId}   # validate slots + price
  2. Feign → promotion     POST /internal/coupons/{code}/apply  # validate + lock coupon
  3. totalAmount = Σ(passengers × price) − discount
  4. Save Booking { status: PENDING_PAYMENT, expiresAt: +30min }
  5. Feign → tour-catalog  PUT /internal/tours/{id}/slots      # decrement available
  6. Kafka publish: booking.created
  7. Redis: cache "booking:{code}" TTL=30min
  8. Return { bookingCode, totalAmount, expiresAt }
```

#### Internal Endpoints

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `POST` | `/internal/bookings/{code}/confirm` | payment-service | Chuyển status → CONFIRMED + Kafka |
| `GET` | `/internal/bookings/check-confirmed` | review-service | Kiểm tra user đã có booking COMPLETED chưa |

---

### 4.4 Payment Service `:8084` ← `PaymentController`

**Database**: `tourism_payment`  
**Entities**: `payments`  
**Feign**: → booking-service  
**Kafka Producer**: `payment.completed`

| Method | Path | Auth | Source Monolith | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/payments/vnpay/create` | USER | `PaymentController.createVNPayPayment()` | Tạo VNPay payment URL |
| `GET` | `/api/payments/vnpay-return` | Public | `PaymentController.vnpayReturn()` | VNPay redirect callback |
| `POST` | `/api/payments/payos/create` | USER | `PaymentController.createPayOSPayment()` | Tạo PayOS checkout link |
| `POST` | `/api/payments/payos-webhook` | Public | `PaymentController.handlePayOSWebhook()` | PayOS webhook |
| `GET` | `/api/payments/payos/return` | Public | `PaymentController.payosReturn()` | PayOS redirect |
| `GET` | `/api/payments/payos/cancel` | Public | `PaymentController.payosCancel()` | PayOS cancel redirect |
| `POST` | `/api/payments/sepay-webhook` | Public | `PaymentController.handleSepayWebhook()` | SePay bank transfer webhook |
| `GET` | `/api/payments/status/{orderCode}` | USER | `PaymentController.getPaymentStatus()` | Trạng thái thanh toán |

**VNPay payment flow**:
```
POST /api/payments/vnpay/create  →  generate VNPay URL  →  redirect client
VNPay → GET /api/payments/vnpay-return?vnp_ResponseCode=00&...
  → verify HMAC signature
  → Feign: POST /internal/bookings/{code}/confirm
  → Kafka: payment.completed
  → redirect to frontend /payment/result?success=true
```

---

### 4.5 Review Service `:8085` ← `ReviewController`

**Database**: `tourism_review`  
**Entities**: `reviews`, `review_images`  
**Feign**: → booking-service  
**Cloudinary**: Upload ảnh review  
**Kafka Producer**: `review.created`

| Method | Path | Auth | Source Monolith | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/reviews` | USER | `ReviewController.submitReview()` | Gửi review (multipart: rating, text, images) |
| `GET` | `/api/reviews/booking/{bookingCode}` | USER | `ReviewController.getReview()` | Xem review theo booking |
| `GET` | `/api/reviews/eligibility/{bookingCode}` | USER | *(New)* | Kiểm tra đủ điều kiện review |
| `GET` | `/api/reviews/tour/{code}` | Public | `ReviewController.getReviewsByTour()` | Danh sách review của tour |
| `GET` | `/api/reviews/tour/{code}/summary` | Public | `ReviewController.getReviewStatistics()` | Thống kê rating (avg + phân phối sao) |
| `GET` | `/api/reviews/my` | USER | *(New)* | Reviews của user hiện tại |
| `DELETE` | `/api/reviews/{id}` | USER | *(New)* | Xoá review của mình |
| `DELETE` | `/api/admin/reviews/{id}` | ADMIN | *(New)* | Admin xoá bất kỳ review |

**Review submission flow**:
```
POST /api/reviews { bookingCode, rating, comment, images[] }
  1. Feign → booking-service GET /internal/bookings/check-confirmed
     # Check user has a CONFIRMED booking for this tour
  2. Upload images[] → Cloudinary  →  get secure URLs
  3. Save Review + ReviewImages
  4. Kafka publish: review.created { tourId, tourCode, rating }
```

---

### 4.6 Promotion Service `:8086` ← `CouponController`

**Database**: `tourism_promotion`  
**Entities**: `coupons`

| Method | Path | Auth | Source Monolith | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/promotions/coupons/{code}` | USER | `CouponController.validateCoupon()` | Validate coupon trước khi đặt |
| `GET` | `/api/admin/coupons` | ADMIN | `CouponController.getAllCoupons()` | Danh sách coupons |
| `GET` | `/api/admin/coupons/search` | ADMIN | `CouponController.searchCoupons()` | Tìm kiếm coupon |
| `POST` | `/api/admin/coupons` | ADMIN | `CouponController.createCoupon()` | Tạo coupon mới |
| `PUT` | `/api/admin/coupons/{id}` | ADMIN | `CouponController.updateCoupon()` | Cập nhật coupon |
| `DELETE` | `/api/admin/coupons/{id}` | ADMIN | `CouponController.deleteCoupon()` | Xoá coupon |

#### Internal Endpoint

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `POST` | `/internal/coupons/{code}/apply` | booking-service | Validate + giảm `usage_count` atomically |

---

### 4.7 Notification Service `:8087` ← `NotificationController`

**Database**: `tourism_notification`  
**Entities**: `notifications`, `user_notifications`  
**Kafka Consumer**: all 5 events  
**JavaMail**: Gmail SMTP

| Method | Path | Auth | Source Monolith | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/notifications/my` | USER | `NotificationController.getMyNotifications()` | In-app notifications |
| `GET` | `/api/notifications/unread-count` | USER | `NotificationController.getUnreadCount()` | Badge count |
| `PUT` | `/api/notifications/{id}/read` | USER | `NotificationController.markAsRead()` | Đánh dấu đã đọc |
| `PUT` | `/api/notifications/read-all` | USER | `NotificationController.markAllAsRead()` | Đọc tất cả |

**Kafka Consumer handlers**:
```
user.registered    → Email: "Chào mừng bạn đến với Tourism!"
                   → In-app: "Tài khoản đã được kích hoạt"

booking.created    → Email: "Đặt tour thành công - Mã booking: {code}"
                   → In-app: "Đặt tour {tourName} thành công"

booking.confirmed  → Email: "Tour của bạn đã được xác nhận"
                   → In-app: "Booking #{code} đã xác nhận"

booking.cancelled  → Email: "Thông báo huỷ tour"
                   → In-app: "Booking #{code} đã bị huỷ"

payment.completed  → Email: "Biên lai thanh toán — {amount}đ"
                   → In-app: "Thanh toán thành công"
```

---

### 4.8 Analytics Service `:8088` ← `DashboardController` + `ChatbotController`

**Database**: `tourism_analytics`  
**Entities**: `daily_stats`, `tour_stats`  
**Kafka Consumer**: all events  
**External**: Google Gemini API

| Method | Path | Auth | Source Monolith | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/analytics/dashboard/summary` | ADMIN | `DashboardController.getDashboardStatistics()` | Tổng quan KPIs |
| `GET` | `/api/analytics/dashboard/daily-stats` | ADMIN | `DashboardController.getDashboardAIAnalysis()` | Stats 7 ngày (chart data) |
| `GET` | `/api/analytics/dashboard/top-tours` | ADMIN | *(New)* | Top tours theo doanh thu |
| `POST` | `/api/analytics/chatbot/chat` | ADMIN | `ChatbotController.chat()` | AI chatbot Gemini |

**Kafka Consumer → DB update**:
```
booking.created    → daily_stats.new_bookings++
booking.confirmed  → daily_stats.confirmed_bookings++
booking.cancelled  → daily_stats.cancelled_bookings++
payment.completed  → daily_stats.total_revenue += amount
review.created     → tour_stats.avg_rating = rolling average
```

---

### 4.9 CMS Service `:8089` ← `PolicyTemplateController` + `BranchContactController`

**Database**: `tourism_cms`  
**Entities**: `policy_templates`, `branch_contacts`

| Method | Path | Auth | Source Monolith | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/cms/policies` | Public | `PolicyTemplateController.getPolicies()` | Danh sách chính sách |
| `GET` | `/api/cms/policies/type/{type}` | Public | `PolicyTemplateController.getPolicyByType()` | Chính sách theo loại |
| `POST` | `/api/cms/policies` | ADMIN | `PolicyTemplateController.createPolicy()` | Tạo chính sách |
| `PUT` | `/api/cms/policies/{id}` | ADMIN | `PolicyTemplateController.updatePolicy()` | Cập nhật chính sách |
| `DELETE` | `/api/cms/policies/{id}` | ADMIN | `PolicyTemplateController.deletePolicy()` | Xoá chính sách |
| `GET` | `/api/cms/branches` | Public | `BranchContactController.getBranches()` | Danh sách chi nhánh |
| `POST` | `/api/cms/branches` | ADMIN | `BranchContactController.createBranch()` | Tạo chi nhánh |
| `PUT` | `/api/cms/branches/{id}` | ADMIN | `BranchContactController.updateBranch()` | Cập nhật chi nhánh |
| `DELETE` | `/api/cms/branches/{id}` | ADMIN | `BranchContactController.deleteBranch()` | Xoá chi nhánh |

---

## 5. Inter-Service Communication

### 5.1 Feign Client Map

```
booking-service
  └──► tour-catalog-service   GET  /internal/tours/{id}           (check availability)
  └──► tour-catalog-service   PUT  /internal/tours/{id}/slots    (decrement slots)
  └──► promotion-service      POST /internal/coupons/{code}/apply (apply coupon)

payment-service
  └──► booking-service        POST /internal/bookings/{code}/confirm (confirm after payment)

review-service
  └──► booking-service        GET  /internal/bookings/check-confirmed (verify eligibility)

identity-service (future)
  └──► (self-contained, no Feign calls)
```

### 5.2 Internal Endpoint Security Strategy

All `/internal/**` endpoints:
- **Not exposed** via API Gateway routing table
- Only reachable within Docker bridge network
- No JWT required — called service-to-service only
- Consider adding `X-Internal-Secret` header for additional hardening

---

## 6. Kafka Event Bus

### 6.1 Topic Definitions (common-events library)

```java
// shared-libs/common-events/src/main/java/com/tourism/events/

public record UserRegisteredEvent(
    Long userId, String email, String fullName, Instant registeredAt
) {}

public record BookingCreatedEvent(
    String bookingCode, Long userId, String userEmail,
    String tourCode, String tourName, String departureDate,
    BigDecimal totalAmount, Integer passengerCount
) {}

public record BookingConfirmedEvent(
    String bookingCode, Long userId, String userEmail,
    String tourName, String departureDate
) {}

public record BookingCancelledEvent(
    String bookingCode, Long userId, String userEmail,
    String tourName, String reason
) {}

public record PaymentCompletedEvent(
    String bookingCode, Long userId, String userEmail,
    BigDecimal amount, String paymentMethod, Instant paidAt
) {}

public record ReviewCreatedEvent(
    Long tourId, String tourCode, Integer rating, Long reviewerId
) {}
```

### 6.2 Producer → Topic → Consumer Matrix

```
┌──────────────────┬──────────────────────┬──────────────────────────────────┐
│    PRODUCER      │       TOPIC          │          CONSUMERS               │
├──────────────────┼──────────────────────┼──────────────────────────────────┤
│ identity-service │ user.registered      │ notification-service (welcome)   │
├──────────────────┼──────────────────────┼──────────────────────────────────┤
│ booking-service  │ booking.created      │ notification-service (email)     │
│                  │                      │ analytics-service (stats++)      │
├──────────────────┼──────────────────────┼──────────────────────────────────┤
│ booking-service  │ booking.confirmed    │ notification-service (email)     │
│                  │                      │ analytics-service (stats++)      │
├──────────────────┼──────────────────────┼──────────────────────────────────┤
│ booking-service  │ booking.cancelled    │ notification-service (email)     │
│                  │                      │ analytics-service (stats++)      │
├──────────────────┼──────────────────────┼──────────────────────────────────┤
│ payment-service  │ payment.completed    │ notification-service (receipt)   │
│                  │                      │ analytics-service (revenue++)    │
├──────────────────┼──────────────────────┼──────────────────────────────────┤
│ review-service   │ review.created       │ analytics-service (rating avg)   │
└──────────────────┴──────────────────────┴──────────────────────────────────┘
```

### 6.3 Kafka Configuration

```yaml
# Each producer service
spring:
  kafka:
    producer:
      bootstrap-servers: kafka:29092
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

# Each consumer service
spring:
  kafka:
    consumer:
      bootstrap-servers: kafka:29092
      group-id: ${spring.application.name}-group
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "com.tourism.events"
```

---

## 7. Infrastructure & External Integrations

### 7.1 Infrastructure Components

| Component | Port | Purpose |
|-----------|------|---------|
| Apache Kafka | `9092` | Async event streaming |
| Zookeeper | `2181` | Kafka coordinator |
| PostgreSQL 15 | `5432` | Relational DB (9 logical databases) |
| Redis 7 | `6379` | Booking cache, session store |
| Zipkin | `9411` | Distributed tracing |
| Eureka Server | `8761` | Service registry & discovery |

### 7.2 External Service Integrations

| Service | Used by | Integration Method | Purpose |
|---------|---------|-------------------|---------|
| **Cloudinary** | identity, tour-catalog, review | REST API (cloudinary-java-sdk) | Image upload & CDN |
| **Gmail SMTP** | identity, notification | JavaMail + Spring Mail | Email verification & notifications |
| **Google OAuth2** | identity-service | google-api-client library | Social login |
| **VNPay** | payment-service | URL signing + redirect | Payment gateway |
| **PayOS** | payment-service | REST API + webhook | Payment gateway |
| **SePay** | payment-service | Webhook (bank transfer) | Payment gateway |
| **Google Gemini** | analytics-service | REST API | AI chatbot |

### 7.3 Environment Variables Reference

```bash
# Shared across services
JWT_SECRET=<64-char random string>

# Database (PostgreSQL)
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres

# Redis
REDIS_HOST=redis
REDIS_PORT=6379

# Kafka
KAFKA_BOOTSTRAP_SERVERS=kafka:29092

# Cloudinary
CLOUDINARY_CLOUD_NAME=<name>
CLOUDINARY_API_KEY=<key>
CLOUDINARY_API_SECRET=<secret>

# Mail
MAIL_USERNAME=<gmail>
MAIL_PASSWORD=<app-password>

# Google
GOOGLE_CLIENT_ID=<client-id>
GOOGLE_CLIENT_SECRET=<client-secret>

# Payment — VNPay
VNPAY_TMN_CODE=<code>
VNPAY_HASH_SECRET=<secret>
VNPAY_URL=https://sandbox.vnpayment.vn/paymentv2/vpcpay.html

# Payment — PayOS
PAYOS_CLIENT_ID=<id>
PAYOS_API_KEY=<key>
PAYOS_CHECKSUM_KEY=<key>

# AI Chatbot
GEMINI_API_KEY=<key>

# ngrok (for local webhook testing)
NGROK_URL=https://<id>.ngrok.io
```

---

## 8. Database Decomposition

### 8.1 From 1 Monolith DB → 9 Isolated DBs

```sql
-- init-db.sql  (runs once when postgres container starts)
CREATE DATABASE tourism_identity;
CREATE DATABASE tourism_catalog;
CREATE DATABASE tourism_booking;
CREATE DATABASE tourism_payment;
CREATE DATABASE tourism_review;
CREATE DATABASE tourism_promotion;
CREATE DATABASE tourism_notification;
CREATE DATABASE tourism_analytics;
CREATE DATABASE tourism_cms;
```

### 8.2 Entity Distribution

| Monolith Entity | Microservice Database | Key Notes |
|----------------|----------------------|-----------|
| `User` | `tourism_identity` | Source of truth for user identity |
| `RefreshToken` | `tourism_identity` | JWT refresh token management |
| `Tour` | `tourism_catalog` | All tour metadata |
| `TourImage`, `TourMedia` | `tourism_catalog` | Cloudinary URLs stored |
| `TourDeparture` | `tourism_catalog` | Availability + pricing |
| `DeparturePricing`, `DepartureTransport` | `tourism_catalog` | Pricing tiers |
| `ItineraryDay` | `tourism_catalog` | Day-by-day travel plan |
| `Location` | `tourism_catalog` | Destinations |
| `FavoriteTour` | `tourism_catalog` | Stores `userId` (no FK cross-service) |
| `Booking` | `tourism_booking` | Lifecycle: PENDING → CONFIRMED → CANCELLED |
| `BookingPassenger` | `tourism_booking` | Passenger details per booking |
| `RefundInformation` | `tourism_booking` | Refund requests |
| `Payment` | `tourism_payment` | Payment records per gateway |
| `Review`, `ImageReview` | `tourism_review` | User reviews + images |
| `Coupon` | `tourism_promotion` | Discount codes |
| `Notification`, `UserNotification` | `tourism_notification` | In-app notifications |
| `PolicyTemplate` | `tourism_cms` | Travel policies |
| `BranchContact` | `tourism_cms` | Office/branch info |
| `DailyStats`, `TourStats` | `tourism_analytics` | Aggregated stats (no raw data) |

### 8.3 Cross-Service References (No Foreign Keys)

Since each service owns its DB, cross-service references use **IDs only** (no FK constraints):

```
booking.user_id        (references identity.users.id — no FK)
booking.tour_id        (references catalog.tours.id  — no FK)
booking.departure_id   (references catalog.tour_departures.id — no FK)
review.user_id         (references identity.users.id — no FK)
favorite_tour.user_id  (references identity.users.id — no FK)
notification.user_id   (references identity.users.id — no FK)
```

---

## 9. 28-Day Master Plan

### Overview

```
Week 1 (D01–D07) │ Foundation: Environment audit + Identity Service + API Gateway
Week 2 (D08–D14) │ Core Services: Tour Catalog + Booking + Payment + Notification
Week 3 (D15–D21) │ Remaining Backend + Frontend Setup: Review + Analytics + CMS + React bootstrap
Week 4 (D22–D28) │ Frontend completion: Booking flow + Admin + Polish + E2E Test
```

---

### 🗓️ Week 1: Foundation & Identity Service

---

#### Day 1 — Environment Audit & Gap Analysis

**Morning**:
```bash
cd D:\KLTN\tourism-microservices
docker-compose up -d
# Verify: http://localhost:8761 (Eureka), http://localhost:9411 (Zipkin)
# Test:   curl http://localhost:8080/api/tours
```

**Afternoon**:
- Read all `identity-service` source → compare against `AuthController.java` in monolith
- Create `gap-analysis.md` — list every endpoint with status: ✅ Done / 🔧 Partial / ❌ Missing
- Setup DBeaver → connect to PostgreSQL `:5432` → verify all 9 databases exist
- Create Postman workspace with folders per service

**Deliverable**: `gap-analysis.md` committed, all containers healthy

---

#### Day 2 — Identity Service: Core Auth

**Tasks**:
- [ ] `POST /api/auth/register` — validate uniqueness, BCrypt hash, send verification email, Kafka `user.registered`
- [ ] `POST /api/auth/login` — verify credentials, generate JWT (HS256, 15min), save refresh token (7 days)
- [ ] `POST /api/auth/refresh-token` — validate refresh token, issue new access token (rotate refresh)
- [ ] `POST /api/auth/logout` - revoke refresh token

**Config**:
```yaml
jwt.secret: ${JWT_SECRET}
jwt.access-token-expiry: 900000     # 15 minutes
jwt.refresh-token-expiry: 604800000 # 7 days
```

**Test**: Postman — register → (simulate verify) → login → get token → refresh → logout

---

#### Day 3 — Identity Service: Email Verification + Google OAuth

**Tasks**:
- [ ] `GET /api/auth/verify-email?token=` — UUID token, 24h TTL, update `email_verified=true`
- [ ] `POST /api/auth/resend-verification` — rate-limited resend
- [ ] Configure JavaMail: `spring.mail.host=smtp.gmail.com`, App Password
- [ ] HTML email template (reuse from monolith Thymeleaf)
- [ ] `POST /api/auth/google/login` — verify Google ID Token via `google-api-client`, findOrCreate user, return JWT

**Test**: Email send from local → check Gmail inbox → click verify link → status changes

---

#### Day 4 — Identity Service: User Profile + Admin

**Tasks**:
- [ ] `GET /api/auth/profile` — read `auth.getName()` (userId from security context)
- [ ] `PATCH /api/users/{id}/profile` — multipart: fullName, phone, dateOfBirth, avatar (Cloudinary)
- [ ] `PATCH /api/users/{id}/change-password` — verify old BCrypt → set new
- [ ] `GET /api/admin/users` — paginated, sorted, filterable
- [ ] `POST /api/admin/users/search` — keyword + status filter
- [ ] `PATCH /api/admin/users/{id}/status` — active/locked toggle

**Test**: Full Postman collection for identity-service — all 15 endpoints covered

**Deliverable**: Identity Service ✅ COMPLETE — commit `feat: identity-service complete`

---

#### Day 5 — API Gateway: JWT Filter + Routing

**Tasks**:
- [ ] Review and complete gateway route table (all 9 services)
- [ ] Verify `JwtAuthFilter` (GlobalFilter):
  - Skip public paths (configurable list)
  - Parse JWT → extract claims
  - Inject `X-User-Id`, `X-User-Email`, `X-User-Role` headers
  - Return 401 on invalid/expired token
- [ ] CORS: `allowed-origins: http://localhost:5173, http://localhost:3000`
- [ ] Test: unauthorized → 401, wrong role → 403, valid → forward

```yaml
# Sample gateway route
spring.cloud.gateway.routes:
  - id: identity-service
    uri: lb://identity-service
    predicates: [Path=/api/auth/**, /api/users/**, /api/admin/users/**]
  - id: tour-catalog-service
    uri: lb://tour-catalog-service
    predicates: [Path=/api/tours/**, /api/locations/**, /api/admin/tours/**, /api/admin/departures/**, /api/admin/locations/**]
```

---

#### Day 6 — common-security + common-events Review

**Tasks**:
- [ ] Verify `common-security` `JwtAuthenticationFilter`:
  - Reads `X-User-Id` and `X-User-Role` headers (NOT re-parsing JWT)
  - Sets `UsernamePasswordAuthenticationToken` in `SecurityContextHolder`
  - `UserPrincipal` has: `userId`, `email`, `role`
- [ ] Verify all critical services import `common-security` correctly
- [ ] Verify `common-events` has all 6 event records with correct fields
- [ ] Test Kafka: produce test message from identity-service → consume in notification-service

---

#### Day 7 — Buffer + Week 1 E2E Test

**E2E Scenario**:
```
1. POST /api/auth/register         → 200 OK
2. GET  /api/auth/verify-email     → 200 OK (email_verified = true)
3. POST /api/auth/login            → 200 + tokens
4. GET  /api/auth/profile          → 200 + user data
5. PATCH /api/users/{id}/profile   → 200 + updated
6. POST /api/auth/logout-all       → 200
7. GET  /api/auth/profile          → 401 (token revoked)
```

**Git commit**: `git tag v0.1-identity`

---

### 🗓️ Week 2: Core Business Services

---

#### Day 8 — Tour Catalog Service: Public APIs & Schema

**Tasks**:
- [ ] Write Flyway migration `V1__init_catalog.sql` — 9 tables
- [ ] Port entities from monolith: `Tour`, `TourDeparture`, `Location`, `ItineraryDay`, `DeparturePricing`, `DepartureTransport`
- [ ] `GET /api/tours` — pagination + filter: `region`, `location`, `keyword`, `priceFrom`, `priceTo`
- [ ] `GET /api/tours/search` — full-text search
- [ ] `GET /api/tours/featured` — sort by booking count / discount
- [ ] `GET /api/tours/{code}` — detail + itinerary days
- [ ] `GET /api/tours/{id}/departures/available` — filter available_slots > 0
- [ ] `GET /api/locations`

---

#### Day 9 — Tour Catalog Service: Admin + Images + Internal

**Tasks**:
- [ ] Admin tour CRUD (7 endpoints) with `@PreAuthorize("hasRole('ADMIN')")`
- [ ] Cloudinary integration: thumbnail + gallery upload
- [ ] Admin departure management: create / update / pricing / transport / clone / delete
- [ ] Favorite tour: add / remove / list my favorites
- [ ] **Internal**: `GET /internal/tours/{id}` — return departure details + price for booking-service
- [ ] **Internal**: `PUT /internal/tours/{id}/slots` — atomic slot decrement

---

#### Day 10 — Booking Service: Create Booking

**Tasks**:
- [ ] Flyway `V1__init_booking.sql` — 3 tables
- [ ] Port entities: `Booking`, `BookingPassenger`, `RefundInformation`
- [ ] Configure Feign clients: `TourCatalogClient`, `PromotionClient`
- [ ] `POST /api/bookings` — implement full flow (see §4.3 above)
- [ ] Redis caching: `@Cacheable("bookings")` on `getBookingByCode`
- [ ] `GET /api/bookings/order?tourCode=&departureId=` — pre-fill booking form

---

#### Day 11 — Booking Service: Cancel + Admin + Internal

**Tasks**:
- [ ] `GET /api/bookings/my?status=` — filter by status with pagination
- [ ] `GET /api/bookings/code/{code}` — with ownership check
- [ ] `POST /api/bookings/{id}/cancel` → Kafka `booking.cancelled`, restore slots
- [ ] `POST /api/bookings/{id}/refund-request` — create RefundInformation record
- [ ] Admin: search bookings, update status
- [ ] **Internal**: `POST /internal/bookings/{code}/confirm` — status → CONFIRMED + Kafka
- [ ] **Internal**: `GET /internal/bookings/check-confirmed` — check by userId + tourCode

**Test**: Full Feign chain — booking → catalog → promotion → confirm

---

#### Day 12 — Payment Service

**Tasks**:
- [ ] Port `PaymentController` from monolith (unchanged logic, new service boundary)
- [ ] VNPay: generate URL, handle return callback, verify HMAC
- [ ] PayOS: create checkout, handle webhook, redirect handlers
- [ ] SePay: bank transfer webhook
- [ ] Feign → booking-service to confirm on payment success
- [ ] `GET /api/payments/status/{orderCode}` — payment status query

> ⚠️ **Local webhook testing**: run `ngrok http 8080`, update VNPay/PayOS sandbox webhook URLs to ngrok URL

---

#### Day 13 — Promotion + Notification Services

**Promotion**:
- [ ] Coupon entity + Flyway migration
- [ ] Admin CRUD (5 endpoints)
- [ ] `GET /api/promotions/coupons/{code}` — validate for user
- [ ] `POST /internal/coupons/{code}/apply` — atomically apply coupon (decrement usage_count)

**Notification**:
- [ ] Configure 5 Kafka `@KafkaListener` handlers
- [ ] Email HTML templates per event type (adapt from monolith Thymeleaf)
- [ ] Save in-app notifications to DB
- [ ] REST: 4 notification endpoints

---

#### Day 14 — Buffer + Week 2 Integration Test

**E2E Flow**:
```
Login → GET /api/tours → GET /api/tours/{code} 
→ POST /api/bookings → POST /api/payments/vnpay/create
→ (ngrok) VNPay callback → booking CONFIRMED
→ GET email notification
→ GET /api/bookings/my → status=CONFIRMED
```

**Git tag**: `v0.2-core-services`

---

### 🗓️ Week 3: Remaining Backend + Frontend Bootstrap

---

#### Day 15 — Review Service

**Tasks**:
- [ ] Flyway migration `tourism_review`
- [ ] `POST /api/reviews` — full flow (see §4.5 above)
- [ ] Public: `GET /api/reviews/tour/{code}` + `GET /api/reviews/tour/{code}/summary`
- [ ] User: my reviews, eligibility check, delete own review
- [ ] Admin: delete any review

---

#### Day 16 — Analytics Service + AI Chatbot

**Tasks**:
- [ ] DB schema: `daily_stats` (date, new_bookings, confirmed_bookings, cancelled_bookings, total_revenue), `tour_stats` (tour_id, total_bookings, avg_rating)
- [ ] 5 Kafka consumers → incremental stat updates
- [ ] 3 dashboard REST endpoints
- [ ] Google Gemini integration: `POST /api/analytics/chatbot/chat`
  - System prompt: inject current stats context
  - Return AI-generated business insight in markdown

---

#### Day 17 — CMS Service + Backend Hardening

**Tasks**:
- [ ] CMS: PolicyTemplate + BranchContact CRUD (9 endpoints total)
- [ ] Standardise error response format across ALL services:
  ```json
  { "success": false, "message": "...", "data": null, "errors": ["..."] }
  ```
- [ ] `docker-compose up -d` → verify all 9 services green ✅
- [ ] Generate Postman collection export

**Git tag**: `v1.0-backend-complete`

---

#### Day 18 — Frontend Bootstrap (Vite + React + TypeScript)

```bash
npm create vite@latest tourism-frontend -- --template react-ts
cd tourism-frontend
npm install axios @reduxjs/toolkit react-redux react-router-dom antd @ant-design/icons
npm install lucide-react react-toastify recharts swiper
npm install @react-oauth/google @stomp/stompjs sockjs-client date-fns
npm install -D sass @types/node
```

**Tasks**:
- [ ] `axiosInstance.ts` — baseURL: API Gateway, request interceptor attaches Bearer token, response interceptor auto-refresh on 401
- [ ] Redux store: `authSlice` (user, accessToken, isAuthenticated), `uiSlice` (loading, toasts)
- [ ] React Router v6: `PublicLayout` (header+footer), `AdminLayout` (sidebar), `ProtectedRoute`, `AdminRoute`

---

#### Day 19 — Frontend: Auth Pages

**Tasks**:
- [ ] `LoginPage`: form + Google Sign-In button
- [ ] `RegisterPage`: form + submit → redirect to "Check your email"
- [ ] `VerifyEmailPage`: confirmation + resend button
- [ ] Google OAuth: `GoogleOAuthProvider` + `useGoogleLogin` hook
- [ ] Redux auth slice: store token in memory (not localStorage) for security
- [ ] Test auth flow end-to-end in browser

---

#### Day 20 — Frontend: HomePage + Tours List

**Tasks**:
- [ ] `HomePage`: Swiper hero banner + featured tours section + destinations grid
- [ ] `ToursPage`: filter sidebar + masonry/grid tour cards + pagination
- [ ] `TourCard` component: image, title, location, price-from, rating, available departures count

---

#### Day 21 — Buffer + Week 3 Review

- Fix bugs from D18–D20
- Responsive check: 375px/768px/1024px
- Git commit frontend progress

---

### 🗓️ Week 4: Frontend Completion + Polish + E2E

---

#### Day 22 — Tour Detail + Booking Form

**Tasks**:
- [ ] `TourDetailPage`: Swiper lightbox gallery + tabs (Description / Itinerary / Reviews / Policy) + departure table with "Book Now" CTA
- [ ] `BookingPage`: passenger roster form with dynamic add/remove + `POST /api/bookings` + redirect to payment

---

#### Day 23 — Payment + My Bookings

**Tasks**:
- [ ] `PaymentPage`: order summary + method selector (VNPay / PayOS / SePay) + `POST /api/payments/*/create`
- [ ] `PaymentResultPage`: parse query params, success animation (confetti), redirect to bookings
- [ ] `MyBookingsPage`: tabbed (All / Pending / Confirmed / Cancelled) + cancel button + refund request modal

---

#### Day 24 — Reviews + Notifications + Profile

**Tasks**:
- [ ] `ReviewSection` in TourDetailPage: rating summary bar chart + review list with avatars + photos
- [ ] `WriteReviewModal`: star picker + textarea + multi-image upload → `POST /api/reviews`
- [ ] `NotificationsPage`: list + mark as read + badge on header icon
- [ ] `ProfilePage`: view + edit form + avatar upload + change password section

---

#### Day 25 — Admin Dashboard + Management Pages

**Tasks**:
- [ ] `AdminLayout`: collapsible sidebar + breadcrumb
- [ ] `DashboardPage`: 4 KPI cards + Recharts LineChart (7-day revenue) + BarChart (bookings) + top-tours table + AI chatbot widget
- [ ] `ToursManagePage`: Ant Design Table + create/edit modal (with Tour form) + image upload drawer
- [ ] `BookingsManagePage`: table + status badge + admin cancel action
- [ ] `CouponsManagePage` + `UsersManagePage`: CRUD tables

---

#### Day 26 — Responsive + Error Handling + Performance

**Tasks**:
- [ ] Responsive audit: test 375px / 768px / 1024px / 1440px
- [ ] Skeleton loaders for TourCard, TourDetailPage, BookingList
- [ ] Custom error pages: 404, 403, 500
- [ ] Toast notifications for all user actions (react-toastify)
- [ ] Route-based code splitting: `React.lazy + Suspense`
- [ ] `<title>` and `<meta name="description">` per page

---

#### Day 27 — Final Integration Test

**20-Scenario E2E Checklist**:

| # | Scenario | Status |
|---|----------|--------|
| 1 | Register new user | ☐ |
| 2 | Verify email via link | ☐ |
| 3 | Login with email+password | ☐ |
| 4 | Login with Google OAuth | ☐ |
| 5 | Browse tours with filter | ☐ |
| 6 | View tour detail + itinerary | ☐ |
| 7 | Book tour with coupon | ☐ |
| 8 | Pay with VNPay sandbox | ☐ |
| 9 | Pay with PayOS QR | ☐ |
| 10 | Receive booking confirmation email | ☐ |
| 11 | View My Bookings — status CONFIRMED | ☐ |
| 12 | Cancel a PENDING booking | ☐ |
| 13 | Submit review with images | ☐ |
| 14 | View in-app notifications | ☐ |
| 15 | Update profile + avatar | ☐ |
| 16 | Admin view dashboard | ☐ |
| 17 | Admin create/edit tour | ☐ |
| 18 | Admin manage departures | ☐ |
| 19 | AI chatbot returns business insight | ☐ |
| 20 | Logout-all → all tokens revoked | ☐ |

---

#### Day 28 — Final Commit + Documentation

**Tasks**:
- [ ] Fix all blocking bugs from Day 27
- [ ] Write `README.md` — architecture overview, local setup guide, env variables reference, screenshots
- [ ] `docker-compose up -d` final verify — 100% healthy
- [ ] Export Postman collection v2.1 JSON
- [ ] **Git tag**: `v1.0-release`

---

## 10. Risk Register

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| Feign circular dependency (A→B→A) | Medium | High | Use `@Lazy` injection; restructure to use Kafka for one direction |
| Kafka consumer deserialization error | Medium | Medium | Add `spring.json.trusted.packages`, test with `auto-offset-reset=earliest` |
| VNPay/PayOS webhook unreachable on localhost | High | High | Use `ngrok http 8080` during development, update sandbox webhook URL |
| Redis host misconfiguration | Low | Medium | Always use service name `redis` (not `localhost`) inside Docker |
| CORS rejection FE → Gateway | Medium | High | Configure `allowed-origins` in gateway to include Vite port 5173 |
| JWT_SECRET mismatch between services | Low | Critical | Single `.env` file mounted to all services; verify at startup |
| Slot race condition on booking | Medium | High | Use `@Transactional` + pessimistic lock on `UPDATE slots WHERE id=? AND slots >= ?` |
| common-security version drift | Low | Medium | Pin version in root `pom.xml`; use `${project.version}` consistently |

---

## 11. Completion Checklist

### Backend Services

| Service | Schema | APIs | Feign | Kafka | Tests | Status |
|---------|--------|------|-------|-------|-------|--------|
| identity-service | ☐ | ☐ | — | Producer | ☐ | ☐ |
| tour-catalog-service | ☐ | ☐ | — | — | ☐ | ☐ |
| booking-service | ☐ | ☐ | ☐ | Producer | ☐ | ☐ |
| payment-service | ☐ | ☐ | ☐ | Producer | ☐ | ☐ |
| review-service | ☐ | ☐ | ☐ | Producer | ☐ | ☐ |
| promotion-service | ☐ | ☐ | — | — | ☐ | ☐ |
| notification-service | ☐ | ☐ | — | Consumer | ☐ | ☐ |
| analytics-service | ☐ | ☐ | — | Consumer | ☐ | ☐ |
| cms-service | ☐ | ☐ | — | — | ☐ | ☐ |

### Infrastructure

- [ ] `docker-compose.yml` — all 13 containers healthy
- [ ] `init-db.sql` — 9 databases created
- [ ] API Gateway — JWT filter + routing + CORS
- [ ] Eureka — all services registered
- [ ] Zipkin — traces flowing
- [ ] Redis — booking cache working
- [ ] Kafka — all 6 topics flowing

### Frontend

- [ ] Auth (Login + Register + Email Verify + Google)
- [ ] Public pages (Home + Tours + Tour Detail)
- [ ] User pages (My Bookings + Profile + Notifications)
- [ ] Booking flow (Form → Payment → Result)
- [ ] Review section (read + write)
- [ ] Admin (Dashboard + Tour CRUD + Booking + Coupon + Users)
- [ ] Responsive (375px → 1440px)
- [ ] Error boundaries + loading states
- [ ] E2E 20 scenarios PASS

---

*Generated from: monolith analysis of `D:\KLTN\Tourism_Backend` + existing microservices at `D:\KLTN\tourism-microservices`*  
*Tech: Java 17 · Spring Boot 3.3 · Spring Cloud 2023.0.2 · PostgreSQL 15 · Redis 7 · Kafka 3.5 · React 18 · Vite 5 · TypeScript · Ant Design 5*
