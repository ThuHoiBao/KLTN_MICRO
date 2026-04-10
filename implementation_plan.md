# Tourism Microservices v2 — Master Implementation Plan
### From Monolith to Production-Grade Microservices

> **Author**: Senior Solution Architect
> **Project**: KLTN — Tourism Management System
> **Monolith source**: `D:\KLTN\Tourism_Backend` (22 controllers, 23 entities)
> **Reference doc**: `D:\KLTN\Tourism_Backend\TLCN_FinalReport.docx`
> **Timeline**: 30 days · 10/04/2026 → 09/05/2026
> **Development style**: ⚡ **Backend + Frontend Parallel — each week ships both layers**

## 🗂️ 2-Repo Git Strategy

| Repo | Path | Git Remote |
|------|------|------------|
| **Backend** | `D:\KLTN\tourism-microservices-v2` | `github.com/YOU/tourism-microservices-v2` |
| **Frontend** | `D:\KLTN\tourism-frontend-v2` | `github.com/YOU/tourism-frontend-v2` |

> **Rationale**: Separate repos allow independent deploy pipelines — BE to EC2 via docker-compose, FE to Vercel/S3/Nginx independently.
> Both repos are already initialized with `git init` and first commit on Day 1.

---

## Table of Contents

1. [Monolith Analysis](#1-monolith-analysis)
2. [Architecture Decision: IAM Service](#2-architecture-decision-iam-service)
3. [System Architecture](#3-system-architecture)
4. [API Specification — All Services](#4-api-specification--all-services)
5. [Inter-Service Communication](#5-inter-service-communication)
6. [Kafka Event Bus](#6-kafka-event-bus)
7. [Database Decomposition](#7-database-decomposition)
8. [Infrastructure & External Integrations](#8-infrastructure--external-integrations)
9. [Project Structure](#9-project-structure)
10. [30-Day Parallel Master Plan](#10-30-day-parallel-master-plan)
11. [Frontend Migration Guide](#11-frontend-migration-guide)
12. [Deployment Plan](#12-deployment-plan)
13. [Risk Register](#13-risk-register)
14. [Completion Checklist](#14-completion-checklist)

---

## 1. Monolith Analysis

### 1.1 Why We Migrate

| Pain Point | Evidence in `Tourism_Backend` |
|-----------|-------------------------------|
| **God Database** | 23 tables in 1 schema — 1 slow query locks entire system |
| **Tight Coupling** | `PaymentController` → `BookingService` → `BookingRepository` in same JVM |
| **Uniform Scaling** | Can't scale Tour Catalog (high read) independently from Payment |
| **Risky Deploy** | Fix 1 bug in Review → redeploy 200K+ lines including Payment |
| **Scattered Auth** | Every controller re-parses JWT independently, no central revocation |
| **No Domain Isolation** | `Booking` entity has direct FK to `User`, `Tour`, `Coupon`, `Payment` simultaneously |

### 1.2 Controller → Microservice Mapping

```
Tourism_Backend/controller/ (22 controllers)
│
├── Identity Domain
│   ├── AuthController.java              → iam-service  (login/logout/refresh)
│   ├── AdminAuthController.java         → identity-service
│   ├── AdminProfileController.java      → identity-service
│   └── UserController.java             → identity-service
│
├── Tour Domain
│   ├── TourController.java             → tour-catalog-service
│   ├── TourManagementController.java   → tour-catalog-service
│   ├── TourDepartureManagementController.java → tour-catalog-service
│   ├── TourMediaController.java        → tour-catalog-service
│   ├── TourUploadController.java       → tour-catalog-service
│   ├── FavoriteTourController.java     → tour-catalog-service
│   ├── LocationController.java         → tour-catalog-service
│   └── LocationAdminController.java    → tour-catalog-service
│
├── Booking Domain
│   └── BookingController.java          → booking-service
│
├── Payment Domain
│   └── PaymentController.java          → payment-service
│
├── Review Domain
│   └── ReviewController.java           → review-service
│
├── Promotion Domain
│   └── CouponController.java           → promotion-service
│
├── Notification Domain
│   └── NotificationController.java     → notification-service
│
├── Analytics & AI Domain
│   ├── DashboardController.java        → analytics-service
│   └── ChatbotController.java          → analytics-service
│
└── CMS Domain
    ├── PolicyTemplateController.java   → cms-service
    └── BranchContactController.java    → cms-service
```

### 1.3 Entity Distribution

| Monolith Entity | Microservice DB | Migration Notes |
|----------------|-----------------|-----------------|
| `User` | `tourism_identity` | Source of truth for user identity |
| `RefreshToken` | `tourism_iam` | **Moved to IAM** (centralized token mgmt) |
| `Tour`, `TourImage`, `TourMedia` | `tourism_catalog` | All tour metadata + Cloudinary URLs |
| `TourDeparture`, `DeparturePricing`, `DepartureTransport` | `tourism_catalog` | Departure schedules + pricing tiers |
| `ItineraryDay`, `Location`, `FavoriteTour` | `tourism_catalog` | Itinerary + destinations |
| `Booking`, `BookingPassenger`, `RefundInformation` | `tourism_booking` | Booking lifecycle |
| `Payment` | `tourism_payment` | Payment records per gateway |
| `Review`, `ImageReview` | `tourism_review` | User reviews + Cloudinary images |
| `Coupon` | `tourism_promotion` | Discount codes |
| `Notification`, `UserNotification` | `tourism_notification` | In-app notifications |
| `PolicyTemplate`, `BranchContact` | `tourism_cms` | Content management |
| *(new)* `DailyStats`, `TourStats` | `tourism_analytics` | Aggregated KPI (no raw data) |
| *(new)* `TokenBlacklist` | `tourism_iam` | Centralized logout/revocation |

---

## 2. Architecture Decision: IAM Service

### 2.1 Problem with `common-security` Shared Library (v1 approach)

```
❌ OLD (tourism-microservices v1):
   Every service imports common-security.jar → parses JWT itself

   booking-service  → reads JWT_SECRET → validates token
   payment-service  → reads JWT_SECRET → validates token
   review-service   → reads JWT_SECRET → validates token

   Problems:
   ① JWT_SECRET distributed to ALL services (severe security risk)
   ② Token blacklist (logout) impossible without shared state
   ③ Update JWT library → rebuild ALL services simultaneously
   ④ No central audit log of token lifecycle
```

### 2.2 Solution: IAM Service as Sole Token Authority

```
✅ NEW (tourism-microservices v2):
   JWT_SECRET exists ONLY in: iam-service + api-gateway
   All downstream services: read X-User-* headers, zero JWT parsing

   iam-service     → issues tokens, verifies tokens, manages blacklist
   api-gateway     → calls IAM for every protected request, injects headers
   booking-service → reads X-User-Id header only (no JWT dependency)
   payment-service → reads X-User-Id header only
   review-service  → reads X-User-Id header only
```

### 2.3 Complete Authentication Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│  STEP 1: LOGIN                                                           │
│  Client → POST /api/iam/auth/login { email, password }                  │
│         → API Gateway (public route, bypass JWT check)                  │
│         → IAM Service:                                                   │
│              Feign → identity-service POST /internal/users/authenticate  │
│              ← { userId, email, role, status }                           │
│              IF status=LOCKED → 403 Forbidden                           │
│              Generate accessToken: JWT(sub=userId, jti=UUID, exp=15min) │
│              Generate refreshToken: SecureRandom → Base64url → SHA256   │
│              Save SHA256(refreshToken) → refresh_tokens table           │
│         ← 200 { accessToken, refreshToken, user }                       │
│                                                                          │
│  STEP 2: AUTHENTICATED REQUEST                                           │
│  Client → GET /api/bookings/my [Authorization: Bearer <accessToken>]    │
│         → API Gateway JwtAuthGatewayFilter:                             │
│              Is public path?  YES → forward as-is                       │
│                               NO  → POST /internal/iam/verify {token}  │
│                                     ← { valid, userId, email, role }    │
│              Caffeine cache (TTL=60s): skip IAM call on cache hit       │
│              Inject headers:                                             │
│                X-User-Id:    42                                          │
│                X-User-Email: user@example.com                           │
│                X-User-Role:  USER                                        │
│         → booking-service:                                              │
│              HeaderAuthFilter reads X-User-* → sets SecurityContext     │
│              (NO JWT library, NO JWT_SECRET here)                       │
│                                                                          │
│  STEP 3: LOGOUT                                                          │
│  Client → POST /api/iam/auth/logout [Bearer token]                      │
│         → IAM Service:                                                   │
│              Extract jti claim from token                               │
│              Redis SET jti "" EX <remaining_seconds>   (fast check)     │
│              INSERT INTO token_blacklist (jti, expires_at)              │
│         ← 200 OK                                                        │
│  Next request with same token → IAM: jti in Redis → { valid: false }   │
│         → Gateway returns 401 Unauthorized                              │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.4 HeaderAuthFilter — Downstream Services (~30 lines, NO JWT library)

```java
// Copy to every downstream service — replaces common-security entirely
public class HeaderAuthFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain)
            throws ServletException, IOException {

        String userId = req.getHeader("X-User-Id");
        String email  = req.getHeader("X-User-Email");
        String role   = req.getHeader("X-User-Role");

        if (userId != null && role != null) {
            var principal = new UserPrincipal(Long.parseLong(userId), email, role);
            var auth = new UsernamePasswordAuthenticationToken(
                principal, null,
                List.of(new SimpleGrantedAuthority("ROLE_" + role))
            );
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(req, res);
    }
}

// UserPrincipal — simple record, no JWT imports
public record UserPrincipal(Long userId, String email, String role) {}
```

### 2.5 Gateway Caffeine Cache (Performance)

```yaml
# Prevents IAM from becoming a performance bottleneck
# api-gateway/src/main/resources/application.yml
gateway:
  token-cache:
    max-size: 10000      # max 10K concurrent tokens
    ttl-seconds: 60      # cache for 60 seconds
    # Result: ~90% cache hit rate at steady-state
    # Logout propagation delay: max 60s (acceptable for KLTN)
```

---

## 3. System Architecture

### 3.1 Full System Topology

```
┌──────────────────────────────────────────────────────────────────────────┐
│   React Frontend  :5173 (dev) / :80 (prod)                              │
│   D:\KLTN\client-side  →  migrate to Vite, keep all UI components       │
│   All API calls → http://localhost:8080 (single Gateway endpoint)       │
└──────────────────────────────────┬───────────────────────────────────────┘
                                   │ HTTP
                    ┌──────────────▼──────────────┐
                    │     API GATEWAY  :8080        │
                    │  Spring Cloud Gateway        │
                    │                              │
                    │  JwtAuthGatewayFilter:       │
                    │  ① Check public-paths list   │
                    │  ② Check Caffeine cache      │
                    │  ③ POST /internal/iam/verify │──────► IAM :8090
                    │  ④ Cache result (TTL=60s)    │
                    │  ⑤ Inject X-User-* headers   │
                    └──────────────┬───────────────┘
                                   │ Route by path
         ┌─────────┬───────────────┼──────────────┬──────────┬──────────┐
         │         │               │              │          │          │
      :8090     :8081           :8082          :8083      :8084      :8085
    iam-svc  identity         tour-cat        booking    payment    review
         │         │               │              │          │          │
    [iam_db] [identity_db]   [catalog_db]   [booking_db] [pay_db] [review_db]
         │
         ├──────────────────────────────────────────────────────────┐
         │              │              │              │              │
      :8086          :8087          :8088          :8089   (Eureka :8761)
    promotion      notif          analytics         cms
         │              │              │              │
    [promo_db]  [notif_db]      [analyt_db]      [cms_db]
                    │              │
                    └──────┬───────┘
                    Apache Kafka :9092
                    6 Topics / 5 Consumer Groups
```

### 3.2 Service Catalogue

| # | Service | Port | Database | Key Capabilities |
|---|---------|------|----------|-----------------|
| – | `service-registry` (Eureka) | **8761** | — | Service discovery |
| – | `api-gateway` | **8080** | — | JWT filter, routing, CORS, rate-limit |
| 0 | `iam-service` ⭐ | **8090** | `tourism_iam` | Token issue/verify/blacklist, refresh rotation |
| 1 | `identity-service` | **8081** | `tourism_identity` | User CRUD, email verify, Google OAuth |
| 2 | `tour-catalog-service` | **8082** | `tourism_catalog` | Tours, departures, locations, favorites |
| 3 | `booking-service` | **8083** | `tourism_booking` | Booking lifecycle, Redis cache |
| 4 | `payment-service` | **8084** | `tourism_payment` | VNPay, PayOS, SePay |
| 5 | `review-service` | **8085** | `tourism_review` | Reviews, ratings, Cloudinary images |
| 6 | `promotion-service` | **8086** | `tourism_promotion` | Coupon CRUD + atomic apply |
| 7 | `notification-service` | **8087** | `tourism_notification` | Email (JavaMail) + in-app |
| 8 | `analytics-service` | **8088** | `tourism_analytics` | KPI dashboard, Gemini AI chatbot |
| 9 | `cms-service` | **8089** | `tourism_cms` | Policies, branch contacts |

### 3.3 API Gateway Route Table

```yaml
# infrastructure/api-gateway/src/main/resources/application.yml

spring:
  cloud:
    gateway:
      routes:
        - id: iam-service
          uri: lb://iam-service
          predicates: [Path=/api/iam/**]

        - id: identity-auth
          uri: lb://identity-service
          predicates: [Path=/api/auth/register,/api/auth/verify-email,/api/auth/resend-verification,/api/auth/google/login,/api/auth/profile]

        - id: identity-users
          uri: lb://identity-service
          predicates: [Path=/api/users/**,/api/admin/users/**]

        - id: tour-catalog
          uri: lb://tour-catalog-service
          predicates: [Path=/api/tours/**,/api/locations/**,/api/admin/tours/**,/api/admin/departures/**,/api/admin/locations/**]

        - id: booking
          uri: lb://booking-service
          predicates: [Path=/api/bookings/**,/api/admin/bookings/**]

        - id: payment
          uri: lb://payment-service
          predicates: [Path=/api/payments/**]

        - id: review
          uri: lb://review-service
          predicates: [Path=/api/reviews/**,/api/admin/reviews/**]

        - id: promotion
          uri: lb://promotion-service
          predicates: [Path=/api/promotions/**,/api/admin/coupons/**]

        - id: notification
          uri: lb://notification-service
          predicates: [Path=/api/notifications/**]

        - id: analytics
          uri: lb://analytics-service
          predicates: [Path=/api/analytics/**]

        - id: cms
          uri: lb://cms-service
          predicates: [Path=/api/cms/**]

      globalcors:
        cors-configurations:
          '[/**]':
            allowedOrigins:
              - http://localhost:5173
              - http://localhost:3000
              - https://tourism-kltn.yourdomain.com
            allowedMethods: [GET,POST,PUT,PATCH,DELETE,OPTIONS]
            allowedHeaders: ["*"]
            allowCredentials: true

# Public paths — bypass JWT verification
gateway:
  public-paths:
    - /api/iam/auth/login
    - /api/iam/auth/refresh-token
    - /api/auth/register
    - /api/auth/verify-email
    - /api/auth/resend-verification
    - /api/auth/google/login
    - GET:/api/tours/**
    - GET:/api/reviews/**
    - GET:/api/cms/**
    - GET:/api/locations/**
    - /api/payments/vnpay-return
    - /api/payments/payos-webhook
    - /api/payments/payos/return
    - /api/payments/payos/cancel
    - /api/payments/sepay-webhook
```

---

## 4. API Specification — All Services

---

### 4.0 IAM Service `:8090` ⭐ NEW

**DB**: `tourism_iam` | **Tables**: `refresh_tokens`, `token_blacklist`
**Feign out**: → `identity-service /internal/users/authenticate`
**Redis**: Blacklist cache key=jti, TTL=remaining token lifetime

#### Public Endpoints (via API Gateway)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/iam/auth/login` | Public | Email+password → `{ accessToken, refreshToken, user }` |
| `POST` | `/api/iam/auth/refresh-token` | Public | Rotate token pair |
| `POST` | `/api/iam/auth/logout` | Bearer | Blacklist current jti |
| `POST` | `/api/iam/auth/logout-all` | Bearer | Revoke all refresh tokens for userId |

#### Internal Endpoints (Gateway/Identity → IAM only)

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `POST` | `/internal/iam/verify` | api-gateway | `{ valid, userId, email, role }` |
| `POST` | `/internal/iam/issue` | identity-service | Issue tokens post-register/Google |

**DB Schema**:
```sql
-- tourism_iam
CREATE TABLE refresh_tokens (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    token_hash  VARCHAR(64) UNIQUE NOT NULL,  -- SHA-256(rawToken)
    expires_at  TIMESTAMP NOT NULL,
    revoked     BOOLEAN DEFAULT FALSE,
    device_info VARCHAR(255),
    created_at  TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_rt_user_id ON refresh_tokens(user_id);

CREATE TABLE token_blacklist (
    jti            VARCHAR(36) PRIMARY KEY,   -- JWT "jti" UUID claim
    expires_at     TIMESTAMP NOT NULL,
    blacklisted_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_bl_expires ON token_blacklist(expires_at);
```

---

### 4.1 Identity Service `:8081`

**DB**: `tourism_identity` | **Tables**: `users`, `email_verifications`
**External**: Cloudinary (avatar), JavaMail (verify email), google-api-client (OAuth2)
**Kafka Producer**: `user.registered`
**Changed vs v1**: Removed JWT generation, removed RefreshToken table → all auth in IAM

#### Public Auth

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/auth/register` | Public | `AuthController.register()` | BCrypt → save → email verify → Feign IAM issue → return tokens |
| `GET` | `/api/auth/verify-email?token=` | Public | `AuthController.verifyEmail()` | Activate account via UUID link |
| `POST` | `/api/auth/resend-verification` | Public | `AuthController.resendVerification()` | Rate-limited email resend |
| `POST` | `/api/auth/google/login` | Public | `AuthController.googleLogin()` | Google ID Token → findOrCreate → Feign IAM issue |
| `GET` | `/api/auth/profile` | USER | `AuthController.getMyProfile()` | Reads X-User-Id header |

#### User Management

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/users/{id}` | USER | `UserController.getUserById()` | Get user profile |
| `PATCH` | `/api/users/{id}/profile` | USER | `UserController.updateProfile()` | multipart: fullName, phone, dob, address, avatar → Cloudinary |
| `PATCH` | `/api/users/{id}/change-password` | USER | `UserController.changePassword()` | Verify BCrypt → set new |

#### Admin

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/admin/users` | ADMIN | `AdminAuthController` | Paginated user list |
| `POST` | `/api/admin/users/search` | ADMIN | `UserController` | Filter by keyword/role/status |
| `PATCH` | `/api/admin/users/{id}/status` | ADMIN | `AdminAuthController` | Lock / unlock account |

#### Internal (called by IAM)

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `POST` | `/internal/users/authenticate` | iam-service | Verify email+BCrypt → return user info |
| `GET` | `/internal/users/{id}/info` | iam-service | Token enrichment payload |

---

### 4.2 Tour Catalog Service `:8082`

**DB**: `tourism_catalog` | 8 tables
**External**: Cloudinary (tour images)

#### Public

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/tours` | Public | `TourController.getAllTours()` | Paginated + filter: region, location, keyword, priceFrom, priceTo |
| `GET` | `/api/tours/search` | Public | `TourController.searchTours()` | Full-text search |
| `GET` | `/api/tours/featured` | Public | `TourController.getTop10DeepestDiscountTours()` | Best discount tours |
| `GET` | `/api/tours/code/{code}` | Public | `TourController.getTourDetail()` | Detail by slug/code |
| `GET` | `/api/tours/{id}` | Public | `TourController.getTourDetail()` | Detail by ID |
| `GET` | `/api/tours/{id}/related` | Public | `TourController.getRelatedTours()` | Same region tours |
| `GET` | `/api/tours/{id}/departures` | Public | `TourController.getTourDepartures()` | All departure schedules |
| `GET` | `/api/tours/{id}/departures/available` | Public | *(NEW)* | Departures with slots > 0 |
| `GET` | `/api/locations` | Public | `LocationController.getLocations()` | All destinations |
| `GET` | `/api/locations/{id}` | Public | `LocationController.getById()` | Location detail |

#### Authenticated User

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/tours/favorites` | USER | `FavoriteTourController.addFavoriteTour()` | Add to favorites |
| `DELETE` | `/api/tours/favorites/{tourId}` | USER | `FavoriteTourController.removeFavoriteTour()` | Remove from favorites |
| `GET` | `/api/tours/favorites/my` | USER | `FavoriteTourController.getUserFavoriteTours()` | My favorites list |

#### Admin Tour Management

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/admin/tours` | ADMIN | `TourManagementController.getAllTours()` | All tours for admin table |
| `POST` | `/api/admin/tours` | ADMIN | `TourManagementController.createTour()` | Create + itinerary |
| `PUT` | `/api/admin/tours/{id}` | ADMIN | `TourManagementController.updateTour()` | Full update |
| `PUT` | `/api/admin/tours/{id}/general-info` | ADMIN | `TourManagementController.updateGeneralInfo()` | Basic info |
| `PUT` | `/api/admin/tours/{id}/itinerary` | ADMIN | `TourManagementController.updateItinerary()` | Day-by-day plan |
| `PATCH` | `/api/admin/tours/{id}/status` | ADMIN | `TourManagementController.updateStatus()` | Active/Inactive |
| `DELETE` | `/api/admin/tours/{id}` | ADMIN | `TourManagementController.deleteTour()` | Soft delete |
| `POST` | `/api/admin/tours/{id}/thumbnail` | ADMIN | `TourMediaController` | Upload thumbnail → Cloudinary |
| `POST` | `/api/admin/tours/{id}/images` | ADMIN | `TourUploadController` | Upload gallery images |

#### Admin Departure Management

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/admin/departures` | ADMIN | `TourDepartureManagementController.createDeparture()` | New departure |
| `PUT` | `/api/admin/departures/{id}` | ADMIN | `TourDepartureManagementController.updateDeparture()` | Update departure |
| `PUT` | `/api/admin/departures/{id}/pricing` | ADMIN | `TourDepartureManagementController.updatePricing()` | Update prices |
| `PUT` | `/api/admin/departures/{id}/transport` | ADMIN | `TourDepartureManagementController.updateTransport()` | Transport info |
| `POST` | `/api/admin/departures/{id}/clone` | ADMIN | `TourDepartureManagementController.cloneDeparture()` | Clone to new date |
| `DELETE` | `/api/admin/departures/{id}` | ADMIN | `TourDepartureManagementController.deleteDeparture()` | Delete |

#### Admin Location Management

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/admin/locations` | ADMIN | `LocationAdminController.create()` | Add location |
| `PUT` | `/api/admin/locations/{id}` | ADMIN | `LocationAdminController.update()` | Update location |
| `DELETE` | `/api/admin/locations/{id}` | ADMIN | `LocationAdminController.delete()` | Delete location |

#### Internal Endpoints

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `GET` | `/internal/tours/departures/{id}` | booking-service | Departure info + pricing |
| `PUT` | `/internal/tours/departures/{id}/slots` | booking-service | Decrement available_slots (pessimistic lock) |
| `PUT` | `/internal/tours/departures/{id}/slots/restore` | booking-service | Restore on cancel |

---

### 4.3 Booking Service `:8083`

**DB**: `tourism_booking` | **Tables**: `bookings`, `booking_passengers`, `refund_information`
**Redis**: Cache booking by code (TTL=30min PENDING, permanent CONFIRMED)
**Feign out**: → tour-catalog-service, → promotion-service
**Kafka Producer**: `booking.created`, `booking.confirmed`, `booking.cancelled`

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/bookings/order` | USER | `BookingController.getBookingInitInfo()` | Pre-fill booking form data |
| `POST` | `/api/bookings` | USER | `BookingController.createBooking()` | Full create flow |
| `GET` | `/api/bookings/my` | USER | `BookingController.getAllBookingsByUser()` | My bookings, filterable by status |
| `GET` | `/api/bookings/code/{code}` | USER | `BookingController.getBookingDetail()` | Detail with ownership check |
| `POST` | `/api/bookings/{id}/cancel` | USER | `BookingController.cancelBooking()` | Cancel PENDING + restore slots |
| `POST` | `/api/bookings/{id}/refund-request` | USER | `BookingController.requestRefund()` | Create refund request |
| `POST` | `/api/admin/bookings/search` | ADMIN | `BookingController.searchBookings()` | Admin search |
| `PATCH` | `/api/admin/bookings/{id}/status` | ADMIN | `BookingController.updateBookingStatus()` | Force status change |

**Booking Status FSM**:
```
PENDING_PAYMENT → (payment success)   → CONFIRMED
PENDING_PAYMENT → (user cancel)       → CANCELLED
PENDING_PAYMENT → (30min @Scheduled)  → EXPIRED
CONFIRMED       → (admin cancel)      → CANCELLED
CONFIRMED       → (auto/admin)        → COMPLETED
```

**Create Booking Flow**:
```
POST /api/bookings { tourCode, departureId, couponCode?, passengers[] }

① Feign → tour-catalog  GET /internal/tours/departures/{id}
② Feign → promotion     POST /internal/coupons/{code}/validate
③ total = Σ(passengers × price_by_type) − discount
④ Feign → promotion     POST /internal/coupons/{code}/apply   (atomic)
⑤ Feign → tour-catalog  PUT /internal/tours/departures/{id}/slots
⑥ Save Booking { status=PENDING_PAYMENT, expires_at=now+30min }
⑦ Save BookingPassengers
⑧ Kafka → booking.created
⑨ Redis cache "booking:{code}" TTL=30min
⑩ Return { bookingCode, totalAmount, expiresAt }
```

#### Internal Endpoints

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `POST` | `/internal/bookings/{code}/confirm` | payment-service | CONFIRMED + Kafka booking.confirmed |
| `GET` | `/internal/bookings/check-confirmed` | review-service | `?userId=&tourCode=` → boolean |

---

### 4.4 Payment Service `:8084`

**DB**: `tourism_payment` | **Tables**: `payments`
**Feign out**: → booking-service
**Kafka Producer**: `payment.completed`
**External**: VNPay, PayOS, SePay

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/payments/vnpay/create` | USER | `PaymentController.createVNPayPayment()` | Generate VNPay redirect URL |
| `GET` | `/api/payments/vnpay-return` | Public | `PaymentController.vnpayReturn()` | VNPay callback (verify HMAC → confirm) |
| `POST` | `/api/payments/payos/create` | USER | `PaymentController.createPayOSPayment()` | PayOS checkout + QR |
| `POST` | `/api/payments/payos-webhook` | Public | `PaymentController.handlePayOSWebhook()` | PayOS server webhook |
| `GET` | `/api/payments/payos/return` | Public | `PaymentController.payosReturn()` | PayOS success redirect |
| `GET` | `/api/payments/payos/cancel` | Public | `PaymentController.payosCancel()` | PayOS cancel redirect |
| `POST` | `/api/payments/sepay-webhook` | Public | `PaymentController.handleSepayWebhook()` | Bank transfer webhook |
| `GET` | `/api/payments/status/{orderCode}` | USER | `PaymentController.getPaymentStatus()` | Query status |
| `GET` | `/api/payments/booking/{bookingCode}` | USER | *(NEW)* | Payment detail for booking |

---

### 4.5 Review Service `:8085`

**DB**: `tourism_review` | **Tables**: `reviews`, `review_images`
**Feign out**: → booking-service
**External**: Cloudinary (review images)
**Kafka Producer**: `review.created`

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/reviews` | USER | `ReviewController.submitReview()` | Review + multipart images |
| `GET` | `/api/reviews/booking/{bookingCode}` | USER | `ReviewController.getReview()` | Review for booking |
| `GET` | `/api/reviews/eligibility/{bookingCode}` | USER | *(NEW)* | Can user review? |
| `GET` | `/api/reviews/tour/{code}` | Public | `ReviewController.getReviewsByTour()` | Tour reviews, paginated |
| `GET` | `/api/reviews/tour/{code}/summary` | Public | `ReviewController.getReviewStatistics()` | Avg + star distribution |
| `GET` | `/api/reviews/my` | USER | *(NEW)* | My reviews |
| `DELETE` | `/api/reviews/{id}` | USER | *(NEW)* | Delete own review |
| `DELETE` | `/api/admin/reviews/{id}` | ADMIN | *(NEW)* | Admin delete |

---

### 4.6 Promotion Service `:8086`

**DB**: `tourism_promotion` | **Tables**: `coupons`

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/promotions/coupons/{code}` | USER | `CouponController.validateCoupon()` | Preview before applying |
| `GET` | `/api/admin/coupons` | ADMIN | `CouponController.getAllCoupons()` | Paginated list |
| `GET` | `/api/admin/coupons/search` | ADMIN | `CouponController.searchCoupons()` | Keyword search |
| `POST` | `/api/admin/coupons` | ADMIN | `CouponController.createCoupon()` | Create coupon |
| `PUT` | `/api/admin/coupons/{id}` | ADMIN | `CouponController.updateCoupon()` | Update |
| `DELETE` | `/api/admin/coupons/{id}` | ADMIN | `CouponController.deleteCoupon()` | Delete |

#### Internal Endpoints

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `POST` | `/internal/coupons/{code}/validate` | booking-service | Read-only check (no side effects) |
| `POST` | `/internal/coupons/{code}/apply` | booking-service | Atomic decrement usage_count |
| `POST` | `/internal/coupons/{code}/restore` | booking-service | Restore on cancel |

---

### 4.7 Notification Service `:8087`

**DB**: `tourism_notification` | **Tables**: `notifications`, `user_notifications`
**Kafka Consumer**: 5 topics | **External**: JavaMail + Gmail SMTP

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/notifications/my` | USER | `NotificationController.getMyNotifications()` | In-app list |
| `GET` | `/api/notifications/unread-count` | USER | `NotificationController.getUnreadCount()` | Badge count |
| `PUT` | `/api/notifications/{id}/read` | USER | `NotificationController.markAsRead()` | Mark one read |
| `PUT` | `/api/notifications/read-all` | USER | `NotificationController.markAllAsRead()` | Mark all read |

**Kafka → Action Mapping**:
| Topic | Email | In-App |
|-------|-------|--------|
| `user.registered` | Welcome email | "Tài khoản tạo thành công" |
| `booking.created` | "Đặt tour #{code}" | "Đặt tour {name} thành công" |
| `booking.confirmed` | "Tour đã xác nhận" | "Booking #{code} xác nhận" |
| `booking.cancelled` | "Tour bị huỷ" | "Booking #{code} đã huỷ" |
| `payment.completed` | "Biên lai {amount}đ" | "Thanh toán thành công" |

---

### 4.8 Analytics Service `:8088`

**DB**: `tourism_analytics` | **Tables**: `daily_stats`, `tour_stats`
**Kafka Consumer**: 5 topics | **External**: Google Gemini API

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/analytics/dashboard/summary` | ADMIN | `DashboardController.getDashboardStatistics()` | KPI totals |
| `GET` | `/api/analytics/dashboard/daily-stats` | ADMIN | `DashboardController.getDashboardAIAnalysis()` | 7/30 day chart data |
| `GET` | `/api/analytics/dashboard/top-tours` | ADMIN | *(NEW)* | Top by revenue/bookings |
| `POST` | `/api/analytics/chatbot/chat` | ADMIN | `ChatbotController.chat()` | Gemini AI chatbot |

---

### 4.9 CMS Service `:8089`

**DB**: `tourism_cms` | **Tables**: `policy_templates`, `branch_contacts`

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/cms/policies` | Public | `PolicyTemplateController.getPolicies()` | All policies |
| `GET` | `/api/cms/policies/type/{type}` | Public | `PolicyTemplateController.getPolicyByType()` | Latest by type |
| `POST` | `/api/cms/policies` | ADMIN | `PolicyTemplateController.createPolicy()` | Create |
| `PUT` | `/api/cms/policies/{id}` | ADMIN | `PolicyTemplateController.updatePolicy()` | Update |
| `DELETE` | `/api/cms/policies/{id}` | ADMIN | `PolicyTemplateController.deletePolicy()` | Delete |
| `GET` | `/api/cms/branches` | Public | `BranchContactController.getBranches()` | Branches list |
| `POST` | `/api/cms/branches` | ADMIN | `BranchContactController.createBranch()` | Add branch |
| `PUT` | `/api/cms/branches/{id}` | ADMIN | `BranchContactController.updateBranch()` | Update branch |
| `DELETE` | `/api/cms/branches/{id}` | ADMIN | `BranchContactController.deleteBranch()` | Delete branch |

---

## 5. Inter-Service Communication

### 5.1 Feign Dependency Graph

```
iam-service
  └──► identity-service  POST /internal/users/authenticate
  └──► identity-service  GET  /internal/users/{id}/info

identity-service
  └──► iam-service       POST /internal/iam/issue  (post register/Google)

booking-service
  └──► tour-catalog      GET  /internal/tours/departures/{id}
  └──► tour-catalog      PUT  /internal/tours/departures/{id}/slots
  └──► tour-catalog      PUT  /internal/tours/departures/{id}/slots/restore
  └──► promotion         POST /internal/coupons/{code}/validate
  └──► promotion         POST /internal/coupons/{code}/apply
  └──► promotion         POST /internal/coupons/{code}/restore

payment-service
  └──► booking-service   POST /internal/bookings/{code}/confirm

review-service
  └──► booking-service   GET  /internal/bookings/check-confirmed
```

### 5.2 Internal Endpoint Security

```yaml
# /internal/** endpoints:
# ① NOT in API Gateway route table → unreachable from outside
# ② Only reachable within Docker bridge network (tourism-net)
# ③ No JWT check — service-to-service only
# ④ Optional: X-Internal-Secret header validation
internal.secret: ${INTERNAL_SECRET:dev-secret}
```

---

## 6. Kafka Event Bus

### 6.1 Event Records (`common-events` library)

```java
// shared-libs/common-events/src/main/java/com/tourism/events/

public record UserRegisteredEvent(
    Long userId, String email, String fullName, Instant registeredAt) {}

public record BookingCreatedEvent(
    String bookingCode, Long userId, String userEmail,
    String tourCode, String tourName, String departureDate,
    BigDecimal totalAmount, Integer passengerCount, Instant createdAt) {}

public record BookingConfirmedEvent(
    String bookingCode, Long userId, String userEmail,
    String tourName, String tourCode, String departureDate) {}

public record BookingCancelledEvent(
    String bookingCode, Long userId, String userEmail,
    String tourName, String reason, Instant cancelledAt) {}

public record PaymentCompletedEvent(
    String bookingCode, Long userId, String userEmail,
    BigDecimal amount, String paymentMethod, String orderId, Instant paidAt) {}

public record ReviewCreatedEvent(
    Long reviewId, Long tourId, String tourCode,
    Long userId, Integer rating, Instant createdAt) {}
```

### 6.2 Producer → Consumer Matrix

```
PRODUCER            TOPIC                 CONSUMERS
identity-service  → user.registered     → notification-service

booking-service   → booking.created     → notification-service
                                         → analytics-service

booking-service   → booking.confirmed   → notification-service
                                         → analytics-service

booking-service   → booking.cancelled   → notification-service
                                         → analytics-service

payment-service   → payment.completed   → notification-service
                                         → analytics-service

review-service    → review.created      → analytics-service
```

---

## 7. Database Decomposition

### 7.1 `init-db.sql` (runs once on first postgres start)

```sql
CREATE DATABASE tourism_iam;
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

### 7.2 Cross-Service References (No FK Constraints)

```
-- Booking stores snapshots to avoid Feign calls at read time:
booking.user_id     BIGINT  -- ref identity.users.id (no FK)
booking.tour_id     BIGINT  -- ref catalog.tours.id (no FK)
booking.tour_code   VARCHAR -- snapshot for display (no Feign needed)
booking.tour_name   VARCHAR -- snapshot for display

-- Favorite stores just user_id (no FK across services):
favorite_tours.user_id BIGINT -- ref identity.users.id (no FK)

-- Review stores just user_id:
reviews.user_id     BIGINT  -- ref identity.users.id (no FK)
```

### 7.3 Key DB Schemas

```sql
-- tourism_catalog
CREATE TABLE tour_departures (
    id              BIGSERIAL PRIMARY KEY,
    tour_id         BIGINT NOT NULL,
    departure_date  DATE NOT NULL,
    return_date     DATE,
    available_slots INTEGER NOT NULL,
    total_slots     INTEGER NOT NULL,
    status          VARCHAR(20) DEFAULT 'OPEN',
    note            TEXT
);
-- Use SELECT ... FOR UPDATE when decrementing slots

-- tourism_booking
CREATE TABLE bookings (
    id              BIGSERIAL PRIMARY KEY,
    code            VARCHAR(20) UNIQUE NOT NULL,
    user_id         BIGINT NOT NULL,
    tour_id         BIGINT NOT NULL,
    tour_code       VARCHAR(50) NOT NULL,
    tour_name       VARCHAR(500),
    departure_id    BIGINT NOT NULL,
    departure_date  DATE,
    coupon_code     VARCHAR(50),
    discount_amount DECIMAL(15,2) DEFAULT 0,
    total_amount    DECIMAL(15,2) NOT NULL,
    status          VARCHAR(30) DEFAULT 'PENDING_PAYMENT',
    expires_at      TIMESTAMP,
    created_at      TIMESTAMP DEFAULT NOW()
);
```

---

## 8. Infrastructure & External Integrations

### 8.1 Docker Infrastructure

| Component | Port | Image | Purpose |
|-----------|------|-------|---------|
| Zookeeper | `2181` | `confluentinc/cp-zookeeper:7.5.0` | Kafka coordinator |
| Apache Kafka | `9092` | `confluentinc/cp-kafka:7.5.0` | Event streaming |
| PostgreSQL 15 | `5432` | `postgres:15-alpine` | Single instance, 10 logical DBs |
| Redis 7 | `6379` | `redis:7-alpine` | Token blacklist + booking cache |
| Zipkin | `9411` | `openzipkin/zipkin` | Distributed tracing |
| Eureka | `8761` | *(custom)* | Service registry |

### 8.2 Complete `.env` Reference

```bash
# ── JWT (ONLY iam-service + api-gateway) ──────────────────────────────
JWT_SECRET=your-64-char-random-string-here
JWT_ACCESS_EXPIRY_MS=900000       # 15 minutes
JWT_REFRESH_EXPIRY_MS=604800000   # 7 days
INTERNAL_SECRET=change-in-production

# ── Database ───────────────────────────────────────────────────────────
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres

# ── Redis ──────────────────────────────────────────────────────────────
REDIS_HOST=redis
REDIS_PORT=6379

# ── Kafka ──────────────────────────────────────────────────────────────
KAFKA_BOOTSTRAP_SERVERS=kafka:29092

# ── Cloudinary ─────────────────────────────────────────────────────────
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_API_KEY=your-api-key
CLOUDINARY_API_SECRET=your-api-secret

# ── JavaMail (Gmail App Password) ──────────────────────────────────────
MAIL_USERNAME=your@gmail.com
MAIL_PASSWORD=xxxx-xxxx-xxxx-xxxx

# ── Google ─────────────────────────────────────────────────────────────
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxx

# ── VNPay ──────────────────────────────────────────────────────────────
VNPAY_TMN_CODE=your-tmn-code
VNPAY_HASH_SECRET=your-secret
VNPAY_URL=https://sandbox.vnpayment.vn/paymentv2/vpcpay.html

# ── PayOS ──────────────────────────────────────────────────────────────
PAYOS_CLIENT_ID=your-client-id
PAYOS_API_KEY=your-api-key
PAYOS_CHECKSUM_KEY=your-key

# ── Gemini AI ──────────────────────────────────────────────────────────
GEMINI_API_KEY=your-key

# ── App URLs ───────────────────────────────────────────────────────────
BASE_URL=http://localhost:8080
FRONTEND_URL=http://localhost:5173
```

---

## 9. Project Structure

> ⚡ **2 Separate Git Repos** — backend and frontend deploy independently

### 9.1 Backend Repo — `D:\KLTN\tourism-microservices-v2`

```
tourism-microservices-v2/           ← Git Repo 1: github.com/YOU/tourism-microservices-v2
├── pom.xml                         # Root Maven POM (parent of all 12 modules)
├── docker-compose.yml              # Backend infra + all 10 services (NO frontend)
├── init-db.sql                     # Creates 10 PostgreSQL databases
├── .env / .env.example
├── README.md
│
├── shared-libs/
│   └── common-events/              # Kafka event DTOs ONLY (no security lib)
│       └── src/main/java/com/tourism/events/
│           ├── UserRegisteredEvent.java
│           ├── BookingCreatedEvent.java
│           ├── BookingConfirmedEvent.java
│           ├── BookingCancelledEvent.java
│           ├── PaymentCompletedEvent.java
│           └── ReviewCreatedEvent.java
│
├── infrastructure/
│   ├── service-registry/            # Eureka :8761
│   └── api-gateway/                 # Spring Cloud Gateway :8080
│       └── src/main/java/com/tourism/gateway/
│           ├── filter/JwtAuthGatewayFilter.java
│           ├── config/RouteConfig.java
│           └── client/IamWebClient.java
│
└── services/
    ├── iam-service/                 # :8090 ⭐ sole JWT_SECRET holder
    ├── identity-service/            # :8081
    ├── tour-catalog-service/        # :8082
    ├── booking-service/             # :8083
    ├── payment-service/             # :8084
    ├── review-service/              # :8085
    ├── promotion-service/           # :8086
    ├── notification-service/        # :8087
    ├── analytics-service/           # :8088
    └── cms-service/                 # :8089
```

### 9.2 Frontend Repo — `D:\KLTN\tourism-frontend-v2`

```
tourism-frontend-v2/                ← Git Repo 2: github.com/YOU/tourism-frontend-v2
├── package.json
├── vite.config.ts                  # Proxy: /api → localhost:8080
├── index.html
├── .env / .env.example             # VITE_API_URL, VITE_GOOGLE_CLIENT_ID
├── README.md
│
└── src/
    ├── main.tsx                    # Entry — GoogleOAuthProvider + AuthProvider
    ├── App.tsx                     # All routes (kept from client-side)
    ├── context/AuthContext.tsx  ⚠️  # LOGIN → /iam/auth/login (UPDATED)
    ├── utils/
    │   ├── axiosInstance.ts     ⚠️  # Refresh → /iam/auth/refresh-token (UPDATED)
    │   └── axiosCustomize.js       # Re-exports axiosInstance (backward compat)
    ├── components/                 # All 17 dirs from client-side — KEEP AS-IS
    ├── services/
    │   ├── dashboard/           ⚠️  # Endpoints → /analytics/dashboard/* (UPDATED)
    │   └── notification/           # NEW: notificationService.ts
    ├── dto/ hook/ data/ assets/    # Keep as-is
    └── index.css / App.css         # Keep as-is
```

### 9.3 How Frontend Calls Backend During Development

```
┌──────────────────────────┐         ┌──────────────────────────────┐
│  tourism-frontend-v2     │         │  tourism-microservices-v2    │
│  npm run dev (:5173)     │         │  docker-compose up           │
│                          │         │                              │
│  vite.config.ts proxy:   │         │  API Gateway        :8080    │
│  /api → localhost:8080 ──┼────────►│  IAM Service        :8090    │
│                          │         │  Identity Service   :8081    │
│  CORS: handled by Vite   │         │  Tour Catalog       :8082    │
│  proxy (no CORS issues)  │         │  ... all services            │
└──────────────────────────┘         └──────────────────────────────┘
```

### 9.4 Git Push Commands (after creating remote repos)

**Backend**:
```bash
cd D:\KLTN\tourism-microservices-v2
git remote add origin https://github.com/YOUR_USERNAME/tourism-microservices-v2.git
git branch -M main
git push -u origin main
```

**Frontend**:
```bash
cd D:\KLTN\tourism-frontend-v2
git remote add origin https://github.com/YOUR_USERNAME/tourism-frontend-v2.git
git branch -M main
git push -u origin main
```

### 9.2 Standard Service Package Layout

```
services/{name}/src/main/java/com/tourism/{domain}/
├── {Domain}Application.java
├── controller/
│   ├── PublicController.java       # GET/public endpoints
│   ├── UserController.java         # Authenticated user endpoints
│   ├── AdminController.java        # /api/admin/** (ROLE_ADMIN)
│   └── InternalController.java     # /internal/** (Feign only, no JWT)
├── service/
├── repository/
├── entity/
├── dto/
│   ├── request/
│   └── response/
├── client/                         # Feign clients (if consuming)
├── event/
│   ├── producer/                   # KafkaTemplate wrappers
│   └── consumer/                   # @KafkaListener handlers
├── config/
│   ├── SecurityConfig.java
│   ├── HeaderAuthFilter.java       # ~30 lines, reads X-User-* headers
│   └── UserPrincipal.java          # record { userId, email, role }
└── exception/
    └── GlobalExceptionHandler.java
```

---

## 10. 30-Day Parallel Master Plan

> ⚡ **PARALLEL PRINCIPLE**: Each day has both **`[BE]` Backend** and **`[FE]` Frontend** tracks.  
> Backend and Frontend developers work simultaneously.  
> Each week delivers a **vertical slice** — working features end-to-end.

### Delivery Overview

```
Week 1 (D01–D07) │ Infrastructure + IAM + Gateway  │  Vite Setup + Auth UI
                  │  → Foundation ready             │  → Login/Register/Verify working
─────────────────────────────────────────────────────────────────────────
Week 2 (D08–D14) │ Identity + Tour Catalog          │  HomePage + Tours + Tour Detail
                  │  → Browse tours working         │  → Browse end-to-end with real data
─────────────────────────────────────────────────────────────────────────
Week 3 (D15–D21) │ Booking + Payment + Review       │  Booking Flow + Payment + Reviews
                  │  → Full booking flow working    │  → Complete user journey shipped
─────────────────────────────────────────────────────────────────────────
Week 4 (D22–D28) │ Promotion + Notif + Analytics    │  Admin Dashboard + Integration
                  │  + CMS + Hardening              │  E2E Tests + Docker + Deploy
─────────────────────────────────────────────────────────────────────────
Day 29-30         │ AWS Deployment + Smoke Tests + Final Documentation
```

---

### 🗓️ WEEK 1 — Foundation · Auth · Infrastructure

---

#### 📅 Day 1 — Project Scaffold

**`[BE]`** Backend:
```bash
mkdir D:\KLTN\tourism-microservices-v2 && cd !$
git init

# Create root directory structure
mkdir -p shared-libs/common-events/src/main/java/com/tourism/events
mkdir -p infrastructure/{service-registry,api-gateway}/src/main/java
mkdir -p services/{iam,identity,tour-catalog,booking,payment,review,promotion,notification,analytics,cms}-service/src/main/java
```
- [ ] Root `pom.xml` with all 12 module declarations
- [ ] `shared-libs/common-events/` with 6 event records
- [ ] `docker-compose.yml`: postgres, redis, kafka, zookeeper, zipkin, service-registry
- [ ] `init-db.sql`: 10 CREATE DATABASE statements
- [ ] `.env.example` documented
- [ ] `docker-compose up -d` → verify Eureka at `http://localhost:8761`

**`[FE]`** Frontend:
```bash
cd D:\KLTN\tourism-microservices-v2
npm create vite@latest frontend -- --template react-ts
cd frontend

# Install exact versions from client-side/package.json
npm install axios@1.13.2 @reduxjs/toolkit react-redux react-router-dom@7
npm install antd @ant-design/icons lucide-react react-icons
npm install @react-oauth/google react-toastify recharts swiper date-fns
npm install @stomp/stompjs sockjs-client react-quill react-bootstrap bootstrap
npm install react-datepicker react-select react-confetti react-imask react-markdown dvhcvn
npm install -D sass @types/node
```
- [ ] Copy `src/` from `D:\KLTN\client-side\src\` into `frontend/src/`
- [ ] Setup `vite.config.ts` with proxy `/api → localhost:8080`
- [ ] Create `frontend/.env`: `VITE_API_URL=http://localhost:8080`

**`[BE]` Deliverable**: Docker infra running, Eureka green ✅  
**`[FE]` Deliverable**: `npm run dev` starts at `:5173` with existing UI visible

---

#### 📅 Day 2 — IAM Service: Token Engine

**`[BE]`** IAM Service core:
- [ ] `iam-service/pom.xml`: jjwt-api, spring-data-redis, spring-data-jpa
- [ ] Flyway `V1__init_iam.sql` (refresh_tokens + token_blacklist tables)
- [ ] `TokenService.generateAccessToken(userId, email, role)` → JWT with jti=UUID
- [ ] `TokenService.generateRefreshToken(userId)` → SecureRandom → SHA256 stored
- [ ] `TokenService.verify(rawToken)` → check Redis blacklist → return TokenVerifyResponse
- [ ] `POST /internal/iam/verify` endpoint
- [ ] Redis Caffeine config
- [ ] Unit tests: valid token, expired token, blacklisted token

**`[FE]`** Rewrite axiosInstance + AuthContext:
- [ ] **Rewrite** `src/utils/axiosInstance.ts` (from `axiosCustomize.js`):
  - Change refresh URL: `/auth/refresh-token` → `/iam/auth/refresh-token`
  - Use `import.meta.env.VITE_API_URL` instead of hardcoded URL
  - Remove console.log debug statements
  - Add TypeScript types
- [ ] **Rewrite** `src/context/AuthContext.tsx`:
  - Login: `POST /auth/login` → `POST /iam/auth/login`
  - Logout: `POST /auth/logout` → `POST /iam/auth/logout`
  - Refresh: `POST /auth/refresh-token` → `POST /iam/auth/refresh-token`
  - Google login stays: `POST /auth/google/login` (identity-service route)

**`[BE]` Deliverable**: TokenService unit tests all green  
**`[FE]` Deliverable**: axiosInstance.ts + AuthContext.tsx updated, compiles

---

#### 📅 Day 3 — IAM Service: Login/Logout/Refresh + API Gateway

**`[BE]` IAM** (morning):
- [ ] `IamAuthController`: 4 public endpoints (login/logout/logout-all/refresh-token)
- [ ] `POST /api/iam/auth/login` full flow: Feign identity → generate pair → return
- [ ] `POST /api/iam/auth/logout` → blacklist jti in Redis + DB
- [ ] `POST /api/iam/auth/refresh-token` → validate SHA256 → issue new pair (rotate)
- [ ] `@Scheduled` cleanup expired blacklist rows

**`[BE]` API Gateway** (afternoon):
- [ ] `api-gateway/pom.xml`: spring-cloud-starter-gateway, caffeine, webflux
- [ ] `JwtAuthGatewayFilter.java` GlobalFilter:
  - Check `public-paths` list
  - Caffeine cache (key=token, TTL=60s)
  - `WebClient` POST `/internal/iam/verify`
  - Inject X-User-Id, X-User-Email, X-User-Role headers
  - Return 401 on invalid, 503 on IAM timeout
- [ ] `RouteConfig.java` with all 11 routes + CORS config

**`[FE]`** Login + Register page fixes:
- [ ] Test `Login/Login.tsx` → calls `/iam/auth/login` → confirm token stored
- [ ] Test `RegisterComponent/Register.tsx` → calls `/auth/register` (identity-service)
- [ ] Test `VerifyEmail/VerifyEmail.tsx` → calls `/auth/verify-email`
- [ ] Fix any imports breaking from `.js → .ts` migration
- [ ] Verify Google Sign-In button renders correctly

**`[BE]` Deliverable**: IAM login works via Gateway with token injection verified  
**`[FE]` Deliverable**: Login + Register flows functional in browser

---

#### 📅 Day 4 — Identity Service: Register + Email Verify + Profile

**`[BE]`** Identity Service:
- [ ] `identity-service/pom.xml`: NO jjwt — only spring-boot-starter-web, spring-data-jpa, spring-mail, google-api-client, cloudinary
- [ ] Flyway `V1__init_identity.sql`
- [ ] `POST /api/auth/register` → BCrypt → save → email verification UUID → JavaMail → Feign IAM issue
- [ ] `GET /api/auth/verify-email` → activate account
- [ ] `POST /api/auth/resend-verification` → rate-limit 1/min
- [ ] `POST /api/auth/google/login` → verify Google ID token → findOrCreate user → Feign IAM issue
- [ ] `GET /api/auth/profile` → reads X-User-Id header (no JWT)
- [ ] `HeaderAuthFilter.java` + `SecurityConfig.java`
- [ ] `POST /internal/users/authenticate` (for IAM)
- [ ] HTML email template (reuse from monolith Thymeleaf)

**`[FE]`** Auth flow end-to-end test:
- [ ] Full flow in browser: Register → check email inbox → click verify → Login → get profile
- [ ] Google login button → OAuth popup → token stored
- [ ] Verify `ProtectedRoute.jsx` still works
- [ ] Fix any `process.env.REACT_APP_*` → `import.meta.env.VITE_*` references

**`[BE]` Deliverable**: Email verification received + account activatable  
**`[FE]` Deliverable**: Full auth cycle works in browser end-to-end

---

#### 📅 Day 5 — Identity Service: User Profile + Admin

**`[BE]`**:
- [ ] `GET /api/users/{id}` — user detail
- [ ] `PATCH /api/users/{id}/profile` — multipart + Cloudinary avatar upload
- [ ] `PATCH /api/users/{id}/change-password`
- [ ] `GET /api/admin/users` — paginated
- [ ] `POST /api/admin/users/search`
- [ ] `PATCH /api/admin/users/{id}/status`
- [ ] Kafka producer: `user.registered` event

**`[FE]`** InformationComponent (profile page):
- [ ] Test `InformationComponent/` — profile tab loads from `GET /api/auth/profile`
- [ ] Test avatar upload → `PATCH /api/users/{id}/profile`
- [ ] Test change password → `PATCH /api/users/{id}/change-password`
- [ ] Verify `src/services/user/` endpoints match new service ports

**`[BE]` Deliverable**: Identity service COMPLETE ✅  
**`[FE]` Deliverable**: Profile page functional

---

#### 📅 Day 6 — Integration + Week 1 E2E

**Full Auth E2E Test**:
```
① POST /api/iam/auth/login          → 200 + tokens
② GET  /api/auth/profile            → 200 + user data (X-User-Id from Gateway)
③ PATCH /api/users/{id}/profile     → 200 + Cloudinary URL
④ PATCH /api/users/{id}/change-pwd  → 200
⑤ POST /api/iam/auth/logout         → 200
⑥ GET  /api/auth/profile (old token)→ 401 (blacklisted in IAM)
⑦ POST /api/iam/auth/logout-all     → 200
⑧ POST /api/iam/auth/refresh-token  → 401 (revoked)
```

**`[BE]`**:
- [ ] Run E2E via Postman (week 1 collection)
- [ ] Fix any Feign circuit-breaker issues
- [ ] Verify Zipkin traces flow: `api-gateway → iam-service → identity-service`

**`[FE]`**:
- [ ] Browser test all 8 scenarios above
- [ ] Verify 401 → auto-refresh → retry works in browser
- [ ] Check browser DevTools: no CORS errors

**Git tag**: `git tag v0.1-auth-foundation`

---

#### 📅 Day 7 — Buffer: Fix + Docs + Week 2 Prep

- [ ] Fix any blocking bugs from D1–D6
- [ ] Write Postman collection for Week 1 endpoints
- [ ] Verify Docker compose healthchecks pass
- [ ] Create `gap-analysis.md` — mark all endpoints: ✅ Done / 🔧 WIP / ❌ Pending

---

### 🗓️ WEEK 2 — Tour Catalog · Browsing End-to-End

---

#### 📅 Day 8 — Tour Catalog Service: Schema + Public APIs

**`[BE]`**:
- [ ] Flyway `V1__init_catalog.sql` — 8 tables (tours, tour_images, tour_departures, departure_pricings, departure_transports, itinerary_days, locations, favorite_tours)
- [ ] Port entities from `Tourism_Backend` (Tour, TourDeparture, etc.)
- [ ] `HeaderAuthFilter` + `SecurityConfig`
- [ ] `GET /api/tours` — pagination + filter
- [ ] `GET /api/tours/featured`
- [ ] `GET /api/tours/code/{code}` + `GET /api/tours/{id}`
- [ ] `GET /api/locations`
- [ ] Internal: `GET /internal/tours/departures/{id}`

**`[FE]`** Homepage + Tour listing:
- [ ] Verify `homPageComponent/HomePage.tsx` renders featured tours from `GET /api/tours/featured`
- [ ] Verify `toursPageComponent/ToursPage.tsx` filter (region, priceFrom, priceTo) + pagination
- [ ] Fix response field mapping if JSON shape differs from monolith

**`[BE]` Deliverable**: Public tour APIs working  
**`[FE]` Deliverable**: HomePage shows real tour data from tour-catalog-service

---

#### 📅 Day 9 — Tour Catalog: Admin + Departures + Cloudinary

**`[BE]`**:
- [ ] Admin tour CRUD (9 endpoints) + `@PreAuthorize("hasRole('ADMIN')")`
- [ ] Admin departure management (6 endpoints)
- [ ] Admin location management (3 endpoints)
- [ ] Cloudinary config: thumbnail + gallery upload
- [ ] `GET /api/tours/{id}/departures/available` (new endpoint)
- [ ] `GET /api/tours/{id}/related`
- [ ] Internal slots: `PUT /internal/tours/departures/{id}/slots` (`SELECT … FOR UPDATE`)
- [ ] Favorite tour endpoints (3)

**`[FE]`** Tour Detail page:
- [ ] Verify `TourDetailComponent/TourDetail.tsx` — gallery Swiper, itinerary tabs, departure table
- [ ] Test favorite button → `POST /api/tours/favorites`
- [ ] Fix departure table to use `/api/tours/{id}/departures/available`
- [ ] Test `DestinationSearchComponent` filter → navigate to `/tours?location=...`

**`[BE]` Deliverable**: Tour Catalog COMPLETE ✅  
**`[FE]` Deliverable**: Tour detail fully interactive with real data

---

#### 📅 Day 10 — Admin Tour UI

**`[BE]`**: Buffer + fix tour-catalog issues

**`[FE]`** AdminComponent — Tour Management:
- [ ] Verify `AdminComponent/Pages/` — tour management table loads from `GET /api/admin/tours`
- [ ] Test create tour modal → `POST /api/admin/tours`
- [ ] Test departure management CRUD
- [ ] Test image upload (thumbnail + gallery) → Cloudinary
- [ ] Test location admin CRUD

**`[FE]` Deliverable**: Admin can fully manage tours from browser

---

#### 📅 Day 11 — CMS Service + CMS Frontend

**`[BE]`** CMS Service (simple — done in 1 day):
- [ ] Flyway `V1__init_cms.sql`
- [ ] PolicyTemplate CRUD (5 endpoints)
- [ ] BranchContact CRUD (5 endpoints)

**`[FE]`** CMS pages:
- [ ] Verify policy display in frontend (if any public pages use it)
- [ ] Verify `services/dashboard/` → update all endpoint URLs:
  - `GET /dashboard/statistics` → `GET /api/analytics/dashboard/summary`
  - `GET /dashboard/ai-analysis` → `GET /api/analytics/dashboard/daily-stats`
- [ ] Update `ChatbotWidget/` endpoint: `POST /chatbot/chat` → `POST /api/analytics/chatbot/chat`

**Git tag**: `v0.2-catalog-cms`

---

#### 📅 Day 12 — Week 2 Integration Test

**`[BE]`** + **`[FE]`** Full browse flow:

```
Register               → account created
Login                  → token received
GET /api/tours         → tour list displayed in browser
GET /api/tours/featured → homepage featured section
GET /api/tours/code/{c} → tour detail with gallery, departures
POST /api/tours/favorites → star icon responds
GET /api/cms/policies  → policy content loads
GET /api/locations     → filter dropdown populated
Logout                 → redirect to login
```

Fix all display issues. Update Postman collection.

---

#### 📅 Day 13 — Promotion Service

**`[BE]`**:
- [ ] Flyway `V1__init_promotion.sql`
- [ ] Coupon entity + CRUD
- [ ] `GET /api/promotions/coupons/{code}` — validate before applying
- [ ] Admin CRUD (5 endpoints)
- [ ] Internal: `validate`, `apply` (atomic), `restore`

**`[FE]`** Coupon input in Booking form:
- [ ] Test `TourBookingComponent/` coupon input field → `GET /api/promotions/coupons/{code}`
- [ ] Admin `CouponManagement` CRUD page test

---

#### 📅 Day 14 — Buffer + Week 2 Polish

- [ ] Fix any issues from D8–D13
- [ ] Responsive check for tour pages (375px, 768px, 1440px)
- [ ] Verify Zipkin traces for tour-related requests

---

### 🗓️ WEEK 3 — Booking · Payment · Review · Notifications

---

#### 📅 Day 15 — Booking Service: Create Booking

**`[BE]`**:
- [ ] Flyway `V1__init_booking.sql`
- [ ] Feign clients: `TourCatalogClient`, `PromotionClient`
- [ ] Redis config for booking cache
- [ ] `POST /api/bookings` — full create flow (see §4.3)
- [ ] `GET /api/bookings/order?tourCode=&departureId=`
- [ ] `GET /api/bookings/my`
- [ ] `GET /api/bookings/code/{code}`
- [ ] Internal: `POST /internal/bookings/{code}/confirm`
- [ ] Internal: `GET /internal/bookings/check-confirmed`

**`[FE]`** Booking Form:
- [ ] Test `TourBookingComponent/TourBooking.tsx` → passenger form fills → `POST /api/bookings`
- [ ] Verify booking code returned and displayed
- [ ] Test `GET /api/bookings/my` in InformationComponent bookings tab

**`[BE]` Deliverable**: Booking creation end-to-end  
**`[FE]` Deliverable**: User can complete booking form

---

#### 📅 Day 16 — Booking Service: Cancel + Admin + Kafka

**`[BE]`**:
- [ ] `POST /api/bookings/{id}/cancel` → restore slots Feign + Kafka booking.cancelled
- [ ] `POST /api/bookings/{id}/refund-request`
- [ ] Admin search + status update
- [ ] Kafka producers for all 3 booking events
- [ ] `@Scheduled` cancel EXPIRED bookings (30min timeout)

**`[FE]`** My Bookings page:
- [ ] Test booking list tabs (All / Pending / Confirmed / Cancelled)
- [ ] Test cancel button → confirm modal → `POST /api/bookings/{id}/cancel`
- [ ] Test refund request form
- [ ] Admin: `BookingsManagePage` → search + status update

**`[BE]` Deliverable**: Booking lifecycle complete  
**`[FE]` Deliverable**: My Bookings page fully functional

---

#### 📅 Day 17 — Payment Service

**`[BE]`**:
- [ ] Port `PaymentController` from monolith (near zero changes needed)
- [ ] VNPay: URL generation + HMAC-SHA512 verify + return handler
- [ ] PayOS: checkout create + server webhook + redirect handlers
- [ ] SePay: bank transfer webhook
- [ ] Feign → booking-service confirm on payment success
- [ ] Kafka: `payment.completed`
- [ ] Setup ngrok: `ngrok http 8080` → update sandbox webhook URLs

**`[FE]`** Payment pages:
- [ ] `BookingPaymentComponent/BookingPayment.tsx` → method selection → redirect to VNPay/PayOS
- [ ] `PaymentSuccess.tsx` → confetti + "View My Bookings" button
- [ ] `PaymentFailed.tsx`, `PaymentError.tsx`, `PaymentWaitingPage.tsx`

**`[BE]` Deliverable**: VNPay sandbox payment completes end-to-end → booking CONFIRMED  
**`[FE]` Deliverable**: Payment flow: select → redirect → result page

---

#### 📅 Day 18 — Review Service

**`[BE]`**:
- [ ] Flyway `V1__init_review.sql`
- [ ] Feign: `BookingServiceClient`
- [ ] Cloudinary for images
- [ ] `POST /api/reviews` — full eligibility check flow
- [ ] `GET /api/reviews/tour/{code}` + `/summary`
- [ ] `GET /api/reviews/eligibility/{bookingCode}`
- [ ] My reviews + delete endpoints
- [ ] Kafka: `review.created`

**`[FE]`** Review section in Tour Detail:
- [ ] `TourDetailComponent/` — review tab: rating summary + review list
- [ ] Write review modal: star selection + textarea + image upload
- [ ] Check eligibility before showing write button
- [ ] Test `POST /api/reviews` multipart form

**`[BE]` Deliverable**: Review service COMPLETE ✅  
**`[FE]` Deliverable**: Review read + write working in Tour Detail

---

#### 📅 Day 19 — Notification Service

**`[BE]`**:
- [ ] Flyway `V1__init_notification.sql`
- [ ] 5 `@KafkaListener` handlers
- [ ] HTML email templates (adapt from monolith Thymeleaf):
  - Welcome email, booking confirmation, booking cancelled, tour confirmed, payment receipt
- [ ] REST 4 endpoints: my-notifications, unread-count, mark-read, mark-all-read

**`[FE]`** Notifications:
- [ ] Create `src/services/notification/notificationService.ts`
- [ ] Add notification bell icon to `HeaderComponent/` with unread count badge
- [ ] `GET /api/notifications/unread-count` — polling every 30s
- [ ] Notification dropdown or `/information/notifications` tab
- [ ] `PUT /api/notifications/{id}/read` on click

**`[BE]` Deliverable**: Emails sent on booking/payment events  
**`[FE]` Deliverable**: Notification badge + list visible in header

---

#### 📅 Day 20 — Week 3 Integration Test: Full User Journey

**Complete Booking Journey**:
```
Login → Browse Tours → View Detail → Check Departures
→ Apply Coupon → Create Booking → Choose VNPay
→ VNPay sandbox payment → Booking CONFIRMED
→ Receive email notification
→ View My Bookings (status: CONFIRMED)
→ Write Review (post-completion)
→ View Notifications (booking + payment events)
```

**`[BE]`**:
- [ ] Run full Postman collection
- [ ] Check Kafka consumer lag (should be 0)
- [ ] Verify email received in test inbox

**`[FE]`**:
- [ ] Full browser walkthrough per journey above
- [ ] Fix any response shape mismatches
- [ ] Fix any navigation issues after payment redirect

**Git tag**: `v0.3-full-user-journey`

---

#### 📅 Day 21 — Analytics Service + Admin Dashboard

**`[BE]`**:
- [ ] Flyway `V1__init_analytics.sql` (daily_stats, tour_stats)
- [ ] 5 Kafka consumers → incremental stat updates
- [ ] `GET /api/analytics/dashboard/summary` — total revenue, bookings, users, active tours
- [ ] `GET /api/analytics/dashboard/daily-stats` — last 7/30 days for charts
- [ ] `GET /api/analytics/dashboard/top-tours`
- [ ] `POST /api/analytics/chatbot/chat` — Gemini API with injected stats context

**`[FE]`** Admin Dashboard:
- [ ] AdminComponent Dashboard page — KPI cards + Recharts LineChart + BarChart
- [ ] Wire `services/dashboard/` to new `/api/analytics/dashboard/*` endpoints
- [ ] ChatbotWidget.tsx → `POST /api/analytics/chatbot/chat`
- [ ] Top tours table in dashboard

**`[BE]` Deliverable**: Analytics service COMPLETE ✅  
**`[FE]` Deliverable**: Admin dashboard shows real KPIs + chatbot works

---

### 🗓️ WEEK 4 — Admin · Polish · E2E · Docker · Deploy

---

#### 📅 Day 22 — Admin Management Pages + User Management

**`[BE]`**: Backend feature freeze — only bug fixes

**`[FE]`** Admin pages:
- [ ] `AdminComponent` users page → `GET /api/admin/users`, `PATCH /api/admin/users/{id}/status`
- [ ] Admin bookings management page → search + force status update
- [ ] Admin coupon management (already verified D13)
- [ ] Verify `AddBannerComponent/` if banner management exists
- [ ] Admin notifications view (if applicable)

**`[FE]` Deliverable**: All admin management pages functional

---

#### 📅 Day 23 — Responsive + Polish + Performance

**`[FE]`**:
- [ ] Responsive audit: 375px / 768px / 1024px / 1440px
- [ ] Skeleton loaders for tour list, tour detail, booking list
- [ ] Error boundaries for all major routes
- [ ] Custom 404 page
- [ ] Toast notifications for all user actions (react-toastify)
- [ ] Remove all `console.log` debug statements
- [ ] `<title>` and `<meta name="description">` per page (for thesis defense)
- [ ] Verify `ScrollToTop.jsx` works between routes

**`[BE]`**:
- [ ] Standardize error response format across ALL services:
  ```json
  { "success": false, "message": "...", "data": null, "errors": ["..."] }
  ```
- [ ] Add `spring-boot-starter-actuator` to all services (healthcheck endpoints)

---

#### 📅 Day 24 — E2E Testing: 25-Scenario Checklist

| # | Scenario | FE | BE |
|---|----------|----|----|
| 1 | Register new user | ☐ | ☐ |
| 2 | Verify email via link | ☐ | ☐ |
| 3 | Login email + password | ☐ | ☐ |
| 4 | Login Google OAuth | ☐ | ☐ |
| 5 | Auto token refresh on 401 | ☐ | ☐ |
| 6 | Browse tours with filter | ☐ | ☐ |
| 7 | Search tours by keyword | ☐ | ☐ |
| 8 | View tour detail + itinerary | ☐ | ☐ |
| 9 | Favorite a tour | ☐ | ☐ |
| 10 | Apply coupon code (valid) | ☐ | ☐ |
| 11 | Create booking with coupon | ☐ | ☐ |
| 12 | Pay with VNPay sandbox | ☐ | ☐ |
| 13 | Pay with PayOS QR code | ☐ | ☐ |
| 14 | Receive booking email | — | ☐ |
| 15 | View My Bookings (CONFIRMED) | ☐ | ☐ |
| 16 | Cancel a PENDING booking | ☐ | ☐ |
| 17 | Submit review + images | ☐ | ☐ |
| 18 | View in-app notifications | ☐ | ☐ |
| 19 | Update profile + avatar | ☐ | ☐ |
| 20 | Change password | ☐ | ☐ |
| 21 | Admin view dashboard stats | ☐ | ☐ |
| 22 | Admin create/edit/delete tour | ☐ | ☐ |
| 23 | Admin manage bookings | ☐ | ☐ |
| 24 | AI chatbot returns insight | ☐ | ☐ |
| 25 | Logout-all → tokens dead | ☐ | ☐ |

Fix all blocking issues before D25.

---

#### 📅 Day 25 — Dockerize All Services

**Standard Backend Dockerfile** (multi-stage):
```dockerfile
FROM maven:3.9-eclipse-temurin-17-alpine AS builder
WORKDIR /build
# Build common-events first (if this service depends on it)
COPY shared-libs/common-events ./shared-libs/common-events
RUN mvn install -f ./shared-libs/common-events/pom.xml -DskipTests -q

COPY services/{name}/pom.xml ./services/{name}/pom.xml
RUN mvn dependency:go-offline -f ./services/{name}/pom.xml -q
COPY services/{name}/src ./services/{name}/src
RUN mvn package -f ./services/{name}/pom.xml -DskipTests -q

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /build/services/{name}/target/*.jar app.jar
EXPOSE 8080
HEALTHCHECK --interval=30s --timeout=5s --retries=5 \
    CMD wget -qO- http://localhost:8080/actuator/health || exit 1
ENTRYPOINT ["java",
    "-XX:+UseContainerSupport",
    "-XX:MaxRAMPercentage=70.0",
    "-Djava.security.egd=file:/dev/./urandom",
    "-jar", "app.jar"]
```

**Frontend Dockerfile**:
```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --silent
COPY . .
ARG VITE_API_URL=http://localhost:8080
ENV VITE_API_URL=$VITE_API_URL
RUN npm run build

FROM nginx:1.25-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
HEALTHCHECK CMD wget -qO- http://localhost || exit 1
```

**nginx.conf** (SPA support):
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    location / { try_files $uri $uri/ /index.html; }
}
```

Tasks:
- [ ] Dockerfile for all 10 backend services
- [ ] Dockerfile for frontend
- [ ] Update `docker-compose.yml` with all 15 containers + health checks + `depends_on`
- [ ] `docker-compose up --build -d` → all containers healthy
- [ ] All 10+ services register in Eureka

---

#### 📅 Day 26 — Seed Data + Final Local Verification

**`[BE]`**:
```sql
-- seed-data.sql
-- Admin user (password: Admin123@)
INSERT INTO users (email, password_hash, full_name, role, status)
VALUES ('admin@tourism.vn', '$2a$10$...', 'Administrator', 'ADMIN', 'ACTIVE');

-- 5 sample tours with departures
-- 2 sample coupons
```
- [ ] Run seed data against all DBs
- [ ] Full smoke test: curl all 11 services healthcheck endpoints
- [ ] Verify Zipkin dashboard at `:9411`
- [ ] Export final Postman Collection v2.1

**`[FE]`**:
- [ ] Test complete flows with Docker (not hot reload)
- [ ] Verify CORS from `:5173` → `:8080` works correctly
- [ ] Test with production build: `npm run build && npx vite preview`

---

#### 📅 Day 27 — AWS EC2 Setup + Deploy

**AWS Architecture**:
```
EC2 t3.medium (2 vCPU, 4 GB RAM) — Ubuntu 22.04 LTS
Elastic IP → Cloudflare DNS → tourism-kltn.yourdomain.com
Security Group: inbound 22 (SSH your IP), 80, 443 / outbound all
```

**EC2 Setup**:
```bash
ssh -i kltn.pem ubuntu@<EC2_IP>
sudo apt update && sudo apt install -y docker.io docker-compose-v2 git nginx certbot python3-certbot-nginx
sudo usermod -aG docker ubuntu && newgrp docker

git clone https://github.com/you/tourism-microservices-v2.git
cd tourism-microservices-v2
cp .env.example .env && nano .env     # Fill all production values
```

**Update production env**:
```bash
# In .env — production critical changes:
BASE_URL=https://tourism-kltn.yourdomain.com
FRONTEND_URL=https://tourism-kltn.yourdomain.com
VNPAY_URL=https://pay.vnpay.vn/vpcpay.html   # production VNPay
# Update PayOS webhook URL to production domain
# Update Google OAuth authorized origins
```

---

#### 📅 Day 28 — Deploy + SSL + Nginx

**Deploy sequence**:
```bash
# Stage 1: Infrastructure
docker compose up -d zookeeper kafka postgres redis zipkin service-registry
sleep 60

# Stage 2: Auth layer (critical path)
docker compose up -d iam-service identity-service api-gateway
sleep 45

# Stage 3: Business services
docker compose up -d \
  tour-catalog-service booking-service payment-service \
  review-service promotion-service notification-service \
  analytics-service cms-service

# Stage 4: Frontend
docker compose up -d frontend
sleep 15

# Verify all registered in Eureka
curl http://localhost:8761/eureka/apps | python3 -c "import sys,json; data=json.load(sys.stdin); print(f'Apps: {len(data[\"applications\"][\"application\"])}')"
```

**Nginx + SSL**:
```bash
sudo nano /etc/nginx/sites-available/tourism-kltn
```
```nginx
server {
    listen 80;
    server_name tourism-kltn.yourdomain.com;
    location /api/  { proxy_pass http://localhost:8080/api/;  proxy_set_header Host $host; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; }
    location /      { proxy_pass http://localhost:80; }
}
```
```bash
sudo ln -s /etc/nginx/sites-available/tourism-kltn /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
sudo certbot --nginx -d tourism-kltn.yourdomain.com   # Auto HTTPS
```

---

#### 📅 Day 29 — Production Smoke Tests + Monitoring

```bash
BASE=https://tourism-kltn.yourdomain.com

# Public APIs
curl "$BASE/api/tours" | jq '.content | length'
curl "$BASE/api/cms/policies" | jq 'length'

# Auth flow
TOKEN=$(curl -s -X POST "$BASE/api/iam/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@tourism.vn","password":"Admin123@"}' | jq -r '.accessToken')
echo "Token: ${TOKEN:0:30}..."

# Protected API
curl -H "Authorization: Bearer $TOKEN" "$BASE/api/auth/profile" | jq '.email'

# Admin API
curl -H "Authorization: Bearer $TOKEN" "$BASE/api/analytics/dashboard/summary" | jq .

# Zipkin (internal only)
curl "http://<EC2_IP>:9411/api/v2/services"
```

- [ ] All 5 smoke test curl commands return expected responses
- [ ] VNPay sandbox with production domain URL
- [ ] PayOS sandbox webhook with production domain
- [ ] Check `docker compose logs -f api-gateway` for any 5xx errors
- [ ] Verify email delivery from production SMTP

---

#### 📅 Day 30 — Final Documentation + Handoff

**`[BE]` + `[FE]`**:
- [ ] `README.md` — architecture overview, local setup, env variables, API docs, screenshots
- [ ] Update `D:\KLTN\tourism-microservices\implementation_plan.md` — mark all checkboxes done
- [ ] Export Postman Collection v2.1 JSON
- [ ] Take screenshots: homepage, tour detail, booking flow, admin dashboard, mobile view
- [ ] Record 2-minute demo video (optional but recommended for thesis defense)
- [ ] Final `git commit -m "feat: v1.0 production ready"` + `git tag v1.0-production`
- [ ] Push to GitHub

---

## 11. Frontend Migration Guide

### 11.1 Migration Strategy: CRA → Vite

> **Existing**: `D:\KLTN\client-side` (CRA, `react-scripts 5.0.1`, TypeScript)
> **Target**: `tourism-microservices-v2/frontend/` (Vite 5, same UI)
> **Approach**: **Migrate, not rewrite** — copy all components, change only:
> 1. Build tool: CRA → Vite
> 2. Auth endpoints: 3 URLs change (login, logout, refresh-token → IAM)
> 3. Dashboard endpoints: `/dashboard/*` → `/analytics/dashboard/*`

### 11.2 Existing Component Inventory (Keep All)

```
D:\KLTN\client-side\src\components\  →  copy to  frontend/src/components/

HeaderComponent/          ✅ Keep — add notification badge
FooterComponent/          ✅ Keep
LayoutComponent/          ✅ Keep (MainLayout with header+footer)
homPageComponent/         ✅ Keep (HomePage with featured tours)
toursPageComponent/       ✅ Keep (Tours list with filter sidebar)
TourDetailComponent/      ✅ Keep (Gallery + tabs + departure table)
TourBookingComponent/     ✅ Keep (Passenger form + coupon input)
BookingPaymentComponent/  ✅ Keep (PaymentSuccess/Failed/Waiting/Error)
InformationComponent/     ✅ Keep (Profile + Bookings + Change Password tabs)
Login/                    ✅ Update: POST /iam/auth/login
RegisterComponent/        ✅ Keep (POST /auth/register — identity-service, unchanged)
VerifyEmail/              ✅ Keep
AdminComponent/           ✅ Keep → UPDATE dashboard + chatbot endpoints
ChatbotWidget/            ✅ Update: POST /analytics/chatbot/chat
DestinationSearchComponent/ ✅ Keep
Commons/                  ✅ Keep (shared utility components)
AddBannerComponent/       ✅ Keep
ProtectedRoute.jsx        ✅ Keep
```

### 11.3 App.tsx Routes — Keep Exactly As-Is

```tsx
// All routes from client-side/src/App.tsx stay identical:
/ → HomePage
/tours → ToursPage
/information → InformationComponent
/information/:tab → InformationComponent(tab)
/tour-detail → TourDetail
/tour/:tourCode → TourDetail
/order-booking → TourBooking
/payment-booking → BookingPayment
/verify-email → VerifyEmail
/payment-success → PaymentSuccess
/payment-failed → PaymentFailed
/payment-waiting → PaymentWaitingPage
/payment-error → PaymentError
/register → Register
/login → Login
/admin/* → AdminComponent
```

### 11.4 Key Files to Modify

**`src/utils/axiosInstance.ts`** (rewrite from `axiosCustomize.js`):

```typescript
import axios, { InternalAxiosRequestConfig } from 'axios';

const BASE_URL = `${import.meta.env.VITE_API_URL || 'http://localhost:8080'}/api`;

const instance = axios.create({ baseURL: BASE_URL, timeout: 30000 });

instance.interceptors.request.use(
    (config: InternalAxiosRequestConfig) => {
        const token = localStorage.getItem('accessToken');
        if (token) config.headers.Authorization = `Bearer ${token}`;
        return config;
    }
);

instance.interceptors.response.use(
    (res) => res,
    async (error) => {
        const orig = error.config;
        if (error.response?.status === 401 && !orig._retry) {
            orig._retry = true;
            try {
                const refreshToken = localStorage.getItem('refreshToken');
                if (!refreshToken) throw new Error('no refresh token');
                const res = await axios.post(`${BASE_URL}/iam/auth/refresh-token`, { refreshToken });
                // ☝️ KEY CHANGE: was /auth/refresh-token
                localStorage.setItem('accessToken', res.data.accessToken);
                if (res.data.refreshToken) localStorage.setItem('refreshToken', res.data.refreshToken);
                orig.headers.Authorization = `Bearer ${res.data.accessToken}`;
                return instance(orig);
            } catch {
                localStorage.clear();
                window.location.href = '/login';
            }
        }
        return Promise.reject(error);
    }
);

export default instance;
```

**`src/context/AuthContext.tsx`** — Only 3 lines change:

```typescript
// ① Login — was: POST /auth/login
const response = await axios.post('/iam/auth/login', { email, password });

// ② Logout — was: POST /auth/logout
await axios.post('/iam/auth/logout', {}, {
    headers: { Authorization: `Bearer ${localStorage.getItem('accessToken')}` }
});

// ③ Refresh — was: POST /auth/refresh-token
const response = await axios.post('/iam/auth/refresh-token', { refreshToken });

// ✅ Google login stays unchanged:
const response = await axios.post('/auth/google/login', { idToken }); // identity-service
```

**New `src/services/notification/notificationService.ts`**:
```typescript
import api from '../../utils/axiosInstance';

export const notificationService = {
    getMyNotifications: (page = 0, size = 20) =>
        api.get(`/notifications/my?page=${page}&size=${size}`),
    getUnreadCount: () =>
        api.get('/notifications/unread-count'),
    markAsRead: (id: number) =>
        api.put(`/notifications/${id}/read`),
    markAllAsRead: () =>
        api.put('/notifications/read-all'),
};
```

**Update `src/services/dashboard/`** (change all paths):
```typescript
// was: GET /dashboard/statistics    →  now: GET /analytics/dashboard/summary
// was: GET /dashboard/ai-analysis   →  now: GET /analytics/dashboard/daily-stats
// was: GET /dashboard/top-tours     →  now: GET /analytics/dashboard/top-tours
```

### 11.5 vite.config.ts

```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
    plugins: [react()],
    resolve: {
        alias: { '@': path.resolve(__dirname, 'src') }
    },
    server: {
        port: 5173,
        proxy: {
            '/api': {
                target: 'http://localhost:8080',
                changeOrigin: true,
                // Dev proxy: eliminates CORS entirely during development
            }
        }
    },
    define: {
        'process.env': {}  // Fix any remaining process.env references
    }
});
```

### 11.6 Tech Stack (Exact Versions from client-side/package.json)

| Library | Version | Status |
|---------|---------|--------|
| React | 18.3.1 | ✅ Keep |
| TypeScript | 4.9.5 | ✅ Keep |
| react-router-dom | 7.8.2 | ✅ Keep |
| axios | 1.13.2 | ✅ Keep |
| antd | 5.29.1 | ✅ Keep |
| react-redux + RTK | 9.2.0 + 2.8.2 | ✅ Keep |
| recharts | 3.6.0 | ✅ Keep |
| swiper | 11.2.10 | ✅ Keep |
| @react-oauth/google | 0.12.2 | ✅ Keep |
| react-toastify | 11.0.5 | ✅ Keep |
| react-quill | 2.0.0 | ✅ Keep (admin editor) |
| react-bootstrap | 2.10.10 | ✅ Keep |
| react-confetti | 6.4.0 | ✅ Keep (payment success) |
| dvhcvn | 1.2.x | ✅ Keep (VN address) |
| react-scripts | 5.0.1 | ❌ Remove |
| vite + plugin-react | 5.x + 4.x | ✅ Add |

---

## 12. Deployment Plan

### 12.1 AWS Architecture

```
Internet
  │
  ▼
Cloudflare (DNS + Free CDN)
  │  tourism-kltn.yourdomain.com → EC2 Elastic IP
  ▼
Nginx (SSL termination via Let's Encrypt)
  ├── /api/*  → http://localhost:8080  (API Gateway container)
  └── /*      → http://localhost:80   (Frontend Nginx container)
       │
  EC2 t3.medium (Ubuntu 22.04)
  Docker Compose — 15 containers:
    ├── Infrastructure: postgres, redis, kafka, zookeeper, zipkin, service-registry
    ├── Auth: iam-service, identity-service, api-gateway
    ├── Business: tour-catalog, booking, payment, review, promotion, notification, analytics, cms
    └── Frontend: nginx serving React SPA
```

### 12.2 Cost Estimate (1 month)

| Resource | Config | Cost |
|----------|--------|------|
| EC2 t3.medium | On-Demand | ~$30/mo |
| Elastic IP | 1 static IP | ~$4/mo |
| EBS 30GB gp3 | Storage | ~$2/mo |
| Cloudflare | Free tier | $0 |
| Let's Encrypt | Free | $0 |
| **Total** | | **~$36/month** |

### 12.3 Memory Management (t3.medium = 4 GB)

```yaml
# docker-compose.yml — resource limits
services:
  iam-service:      { deploy: { resources: { limits: { memory: 256m } } } }
  identity-service: { deploy: { resources: { limits: { memory: 256m } } } }
  api-gateway:      { deploy: { resources: { limits: { memory: 384m } } } }
  tour-catalog-service: { deploy: { resources: { limits: { memory: 384m } } } }
  booking-service:  { deploy: { resources: { limits: { memory: 256m } } } }
  payment-service:  { deploy: { resources: { limits: { memory: 256m } } } }
  review-service:   { deploy: { resources: { limits: { memory: 256m } } } }
  promotion-service: { deploy: { resources: { limits: { memory: 192m } } } }
  notification-service: { deploy: { resources: { limits: { memory: 256m } } } }
  analytics-service: { deploy: { resources: { limits: { memory: 384m } } } }
  cms-service:      { deploy: { resources: { limits: { memory: 192m } } } }
  postgres:         { deploy: { resources: { limits: { memory: 512m } } } }
  redis:            { deploy: { resources: { limits: { memory: 128m } } } }
  kafka:            { deploy: { resources: { limits: { memory: 512m } } } }
  # Total: ~3.5 GB (comfortable on t3.medium 4 GB)
```

---

## 13. Risk Register

| Risk | P | I | Mitigation |
|------|---|---|------------|
| IAM Service bottleneck | M | H | Caffeine cache at Gateway (60s TTL) → 90% calls cached |
| Feign circular dependency | M | H | Use `@Lazy` injection; convert one direction to Kafka event |
| Kafka consumer lag / deserialization error | M | M | `spring.json.trusted.packages=com.tourism.events`; `auto-offset-reset=earliest` |
| VNPay/PayOS unreachable webhook locally | H | H | `ngrok http 8080`; update sandbox webhook URLs before testing |
| Slot race condition | M | H | `@Lock(PESSIMISTIC_WRITE)` on TourDeparture; `UPDATE slots WHERE id=? AND slots >= ?` |
| Redis not reachable from service | L | M | Use Docker service name `redis:6379`, never `localhost` inside compose |
| JWT_SECRET leak | L | C | `.env` git-ignored; `JWT_SECRET` only in `iam-service` + `api-gateway` |
| EC2 OOM kill | M | H | Set `deploy.resources.limits.memory` per container; monitor with `docker stats` |
| CORS rejection | M | H | Gateway CORS: `allowed-origins` includes `:5173` and production domain |
| CRA → Vite `process.env` errors | M | M | Add `define: { 'process.env': {} }` in vite.config.ts |
| 401 loop (refresh fails) | L | M | Check `_retry` flag in interceptor; `localStorage.clear()` on refresh failure |

---

## 14. Completion Checklist

### Infrastructure
- [ ] `docker-compose.yml` — 15 containers healthy
- [ ] `init-db.sql` — 10 databases created
- [ ] API Gateway — JWT filter + IAM + Caffeine cache + CORS
- [ ] Eureka — all 10 services registered
- [ ] Zipkin — traces flowing across services
- [ ] Kafka — 6 topics, consumer lag = 0
- [ ] Redis — IAM blacklist + booking cache operational

### Backend Services

| Service | Schema | APIs | Tests | Docker |
|---------|--------|------|-------|--------|
| iam-service | ☐ | ☐ | ☐ | ☐ |
| identity-service | ☐ | ☐ | ☐ | ☐ |
| tour-catalog-service | ☐ | ☐ | ☐ | ☐ |
| booking-service | ☐ | ☐ | ☐ | ☐ |
| payment-service | ☐ | ☐ | ☐ | ☐ |
| review-service | ☐ | ☐ | ☐ | ☐ |
| promotion-service | ☐ | ☐ | ☐ | ☐ |
| notification-service | ☐ | ☐ | ☐ | ☐ |
| analytics-service | ☐ | ☐ | ☐ | ☐ |
| cms-service | ☐ | ☐ | ☐ | ☐ |

### Frontend Migration (from `D:\KLTN\client-side`)

- [ ] Vite project created + all deps installed
- [ ] All 17 component directories copied from client-side
- [ ] `axiosInstance.ts` — refresh URL updated to `/iam/auth/refresh-token`
- [ ] `AuthContext.tsx` — login/logout URLs updated to IAM
- [ ] Dashboard service — endpoints updated to `/analytics/dashboard/*`
- [ ] ChatbotWidget — endpoint updated to `/analytics/chatbot/chat`
- [ ] Notification service added (`notificationService.ts`)
- [ ] Notification badge added to HeaderComponent
- [ ] All 25 E2E scenarios PASS in browser
- [ ] Responsive (375px / 768px / 1440px) verified
- [ ] Production build successful (`npm run build`)

### Deployment
- [ ] EC2 running (t3.medium Ubuntu 22.04)
- [ ] Domain configured (Cloudflare → EC2 Elastic IP)
- [ ] SSL certificate active (Let's Encrypt)
- [ ] All 5 production smoke tests PASS
- [ ] VNPay + PayOS webhook URLs updated to production domain
- [ ] `git tag v1.0-production` pushed

---

> **Stack**: Java 17 · Spring Boot 3.3 · Spring Cloud 2023.0.2 · PostgreSQL 15 · Redis 7 · Kafka 3.5 · React 18 · Vite 5 · TypeScript · Ant Design 5 · Docker · AWS EC2
>
> **Monolith**: `D:\KLTN\Tourism_Backend` — 22 controllers, 23 entities
> **Frontend base**: `D:\KLTN\client-side` — CRA → Vite migration, all UI kept
> **Target project**: `D:\KLTN\tourism-microservices-v2`
