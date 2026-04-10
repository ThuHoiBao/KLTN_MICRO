# Tourism Microservices v2 — Master Implementation Plan
### From Monolith to Production-Grade Microservices

> **Author**: Senior Solution Architect  
> **Project**: KLTN — Tourism Management System  
> **Backend target**: `D:\KLTN\tourism-microservices-v2` *(project mới hoàn toàn)*  
> **Frontend source**: `D:\KLTN\client-side` *(migrate CRA → Vite, giữ UI/UX)*  
> **Monolith source**: `D:\KLTN\Tourism_Backend`  
> **Reference**: `D:\KLTN\Tourism_Backend\TLCN_FinalReport.docx`  
> **Timeline**: 30 days · Start: 10/04/2026 · Deploy: 09/05/2026

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
10. [30-Day Master Plan](#10-30-day-master-plan)
11. [Frontend Architecture](#11-frontend-architecture)
12. [Deployment Plan](#12-deployment-plan)
13. [Risk Register](#13-risk-register)
14. [Completion Checklist](#14-completion-checklist)

---

## 1. Monolith Analysis

### 1.1 Why We Migrate

| Pain Point | Evidence in `Tourism_Backend` |
|-----------|-------------------------------|
| **God Database** | 23 tables in 1 schema — 1 slow query blocks entire system |
| **Tight Coupling** | `BookingController` → `PaymentService` → `BookingRepository` directly |
| **Uniform Scaling** | Want to scale Tour Catalog (high read) but must scale everything |
| **Risky Deploy** | Fix 1 bug in Review → redeploy 200K+ lines affecting Payment |
| **Scattered Auth** | Every controller re-parses JWT independently, no central control |
| **No Domain Isolation** | `Booking` entity has direct FK to `User`, `Tour`, `Coupon`, `Payment` simultaneously |

### 1.2 Monolith Controller → Service Mapping

```
Tourism_Backend Controllers (22 total)
│
├── Auth & Identity Domain
│   ├── AuthController.java             → identity-service
│   ├── AdminAuthController.java        → identity-service
│   ├── AdminProfileController.java     → identity-service
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
├── Analytics Domain
│   ├── DashboardController.java        → analytics-service
│   └── ChatbotController.java          → analytics-service
│
└── CMS Domain
    ├── PolicyTemplateController.java   → cms-service
    └── BranchContactController.java    → cms-service
```

### 1.3 Entity Distribution Plan

| Monolith Entity | Migrates To | Notes |
|----------------|-------------|-------|
| `User` | `tourism_identity` | Identity source of truth |
| `RefreshToken` | `tourism_iam` | Moved to IAM service |
| `Tour`, `TourImage`, `TourMedia` | `tourism_catalog` | |
| `TourDeparture`, `DeparturePricing`, `DepartureTransport` | `tourism_catalog` | |
| `ItineraryDay`, `Location`, `FavoriteTour` | `tourism_catalog` | |
| `Booking`, `BookingPassenger`, `RefundInformation` | `tourism_booking` | |
| `Payment` | `tourism_payment` | |
| `Review`, `ImageReview` | `tourism_review` | |
| `Coupon` | `tourism_promotion` | |
| `Notification`, `UserNotification` | `tourism_notification` | |
| `PolicyTemplate`, `BranchContact` | `tourism_cms` | |
| *(new)* `DailyStats`, `TourStats` | `tourism_analytics` | Aggregated, no raw data |
| *(new)* `TokenBlacklist` | `tourism_iam` | Centralized token management |

---

## 2. Architecture Decision: IAM Service

### 2.1 The Problem with `common-security` Shared Library

In `tourism-microservices` (v1), every service imports `common-security.jar`:

```
❌ Old approach (Shared Library):
  booking-service → imports common-security.jar → parses JWT itself
  payment-service → imports common-security.jar → parses JWT itself
  review-service  → imports common-security.jar → parses JWT itself
  
  Problems:
  • JWT_SECRET must be distributed to ALL services (security risk)
  • Token blacklist (logout) is impossible without shared state
  • Updating JWT library requires rebuilding ALL services
  • No central control over token lifecycle
```

### 2.2 The Solution: IAM Service as Central Authority

```
✅ New approach (IAM Service):
  ALL services receive: X-User-Id, X-User-Email, X-User-Role headers
  ONLY IAM Service knows: JWT_SECRET, token blacklist

  JWT_SECRET  →  only in iam-service + api-gateway
  All others  →  read X-User-* headers, no JWT parsing at all
```

### 2.3 Token Verification Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    COMPLETE AUTHENTICATION FLOW                           │
│                                                                          │
│  Step 1: Login                                                           │
│  ─────────────────────────────────────────────────────────────────────  │
│  Client → POST /api/iam/auth/login { email, password }                  │
│         → API Gateway (public route, skip JWT check)                    │
│         → IAM Service:                                                   │
│              Feign ──► identity-service POST /internal/users/auth        │
│              ◄── { userId, email, role, status }                         │
│              generate accessToken (JWT, 15min, contains jti=UUID)        │
│              generate refreshToken (opaque random, SHA256 stored)        │
│              save to refresh_tokens table                                │
│         ◄── { accessToken, refreshToken, user }                          │
│                                                                          │
│  Step 2: Authenticated Request                                           │
│  ─────────────────────────────────────────────────────────────────────  │
│  Client → GET /api/bookings/my                                           │
│           [Authorization: Bearer <accessToken>]                          │
│         → API Gateway JwtAuthGatewayFilter:                              │
│              Is public path? → YES: forward as-is                        │
│                               NO: POST /internal/iam/verify { token }   │
│                                   → IAM Service:                         │
│                                       parse JWT                          │
│                                       check token_blacklist              │
│                                       ← { valid, userId, email, role }   │
│              inject headers:                                             │
│                X-User-Id: 42                                             │
│                X-User-Email: user@example.com                            │
│                X-User-Role: USER                                         │
│          → booking-service:                                              │
│              HeaderAuthFilter reads X-User-* headers                     │
│              sets SecurityContext (NO JWT parsing)                       │
│              @PreAuthorize("hasRole('ADMIN')") works normally            │
│                                                                          │
│  Step 3: Logout                                                          │
│  ─────────────────────────────────────────────────────────────────────  │
│  Client → POST /api/iam/auth/logout [Authorization: Bearer token]        │
│         → IAM Service:                                                   │
│              extract jti from token                                      │
│              INSERT INTO token_blacklist (jti, expires_at)               │
│              cache in Redis (key=jti, TTL=remaining token lifetime)      │
│         ← 200 OK                                                         │
│  Next request with same token → IAM returns { valid: false } → 401       │
└──────────────────────────────────────────────────────────────────────────┘
```

### 2.4 Gateway Token Caching (Performance)

```
To prevent IAM becoming a bottleneck:
  API Gateway → Caffeine local cache (key=token, TTL=60s, max=10,000)
  Cache Hit  → skip IAM call → inject headers directly
  Cache Miss → POST /internal/iam/verify → cache result

  Result: ~90% cache hit rate at steady state
  Logout propagation: max 60s delay (acceptable for KLTN)
```

### 2.5 HeaderAuthFilter — Per-Service (~30 lines, no JWT library)

```java
// Each downstream service has this filter — NO jjwt dependency needed
public class HeaderAuthFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req,
                                    HttpServletResponse res,
                                    FilterChain chain) throws ServletException, IOException {
        String userId = req.getHeader("X-User-Id");
        String email  = req.getHeader("X-User-Email");
        String role   = req.getHeader("X-User-Role");

        if (userId != null && role != null) {
            var auth = new UsernamePasswordAuthenticationToken(
                new UserPrincipal(Long.parseLong(userId), email, role),
                null,
                List.of(new SimpleGrantedAuthority("ROLE_" + role))
            );
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(req, res);
    }
}
```

---

## 3. System Architecture

### 3.1 Service Topology

```
┌─────────────────────────────────────────────────────────────────────┐
│                   REACT FRONTEND  (:5173 dev / :80 prod)           │
│         All HTTP calls → http://localhost:8080 (API Gateway)        │
└─────────────────────────────────┬───────────────────────────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │      API GATEWAY :8080      │
                    │   Spring Cloud Gateway      │
                    │                            │
                    │  JwtAuthGatewayFilter:      │
                    │  IF protected route:        │
                    │    POST /internal/iam/verify│──► IAM :8090
                    │    inject X-User-* headers  │
                    │  Route table → services     │
                    └─────────────┬───────────────┘
                                  │
     ┌────────┬────────┬──────────┼──────────┬────────┬────────┐
     │        │        │          │          │        │        │
  :8090    :8081    :8082      :8083      :8084    :8085    :8086
   IAM   identity  tour-cat  booking   payment  review   promo
     │        │        │          │          │        │        │
     │        │        │          │          │        │        │
  tourism_ tourism_ tourism_ tourism_ tourism_ tourism_ tourism_
    iam   identity  catalog  booking  payment  review  promotion

                    │
     ┌──────────────┼──────────────┐
     │              │              │
  :8087          :8088          :8089
 notif          analytics        cms
     │              │              │
 tourism_      tourism_       tourism_
notification  analytics        cms

                    │
          ┌─────────┴─────────┐
          │  Apache Kafka     │ (Async Event Bus)
          │  6 Topics         │
          └───────────────────┘
```

### 3.2 Service Catalogue

| # | Service | Port | Database | Role |
|---|---------|------|----------|------|
| — | `service-registry` (Eureka) | **8761** | — | Service discovery |
| — | `api-gateway` | **8080** | — | Single entry point + JWT filter |
| 0 | `iam-service` ⭐ | **8090** | `tourism_iam` | Token issuer, verifier, blacklist |
| 1 | `identity-service` | **8081** | `tourism_identity` | User CRUD, email verify, Google OAuth |
| 2 | `tour-catalog-service` | **8082** | `tourism_catalog` | Tours, departures, locations, favorites |
| 3 | `booking-service` | **8083** | `tourism_booking` | Booking lifecycle |
| 4 | `payment-service` | **8084** | `tourism_payment` | VNPay, PayOS, SePay |
| 5 | `review-service` | **8085** | `tourism_review` | Reviews + ratings |
| 6 | `promotion-service` | **8086** | `tourism_promotion` | Coupons |
| 7 | `notification-service` | **8087** | `tourism_notification` | Email + in-app |
| 8 | `analytics-service` | **8088** | `tourism_analytics` | Dashboard + AI chatbot |
| 9 | `cms-service` | **8089** | `tourism_cms` | Policy + branch content |

### 3.3 API Gateway Route Table

```yaml
# api-gateway/src/main/resources/application.yml

spring.cloud.gateway.routes:
  # IAM (auth token management)
  - id: iam-auth
    uri: lb://iam-service
    predicates:
      - Path=/api/iam/**

  # Identity (user management)
  - id: identity-auth
    uri: lb://identity-service
    predicates:
      - Path=/api/auth/register, /api/auth/verify-email,
              /api/auth/resend-verification, /api/auth/google/login
  - id: identity-users
    uri: lb://identity-service
    predicates:
      - Path=/api/users/**, /api/admin/users/**

  # Tour Catalog
  - id: tour-catalog
    uri: lb://tour-catalog-service
    predicates:
      - Path=/api/tours/**, /api/locations/**, /api/admin/tours/**,
              /api/admin/departures/**, /api/admin/locations/**

  # Booking
  - id: booking
    uri: lb://booking-service
    predicates:
      - Path=/api/bookings/**, /api/admin/bookings/**

  # Payment
  - id: payment
    uri: lb://payment-service
    predicates:
      - Path=/api/payments/**

  # Review
  - id: review
    uri: lb://review-service
    predicates:
      - Path=/api/reviews/**, /api/admin/reviews/**

  # Promotion
  - id: promotion
    uri: lb://promotion-service
    predicates:
      - Path=/api/promotions/**, /api/admin/coupons/**

  # Notification
  - id: notification
    uri: lb://notification-service
    predicates:
      - Path=/api/notifications/**

  # Analytics
  - id: analytics
    uri: lb://analytics-service
    predicates:
      - Path=/api/analytics/**

  # CMS
  - id: cms
    uri: lb://cms-service
    predicates:
      - Path=/api/cms/**

# Public paths — skip JWT verification
gateway.public-paths:
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

### 4.0 IAM Service `:8090` — ⭐ NEW SERVICE

**Database**: `tourism_iam`  
**Tables**: `refresh_tokens`, `token_blacklist`  
**Feign**: → identity-service `/internal/users/authenticate`  
**Redis**: Blacklist cache (key=jti, TTL=token remaining lifetime)

#### Public Auth Endpoints (via Gateway)

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/iam/auth/login` | Public | Login → return `{ accessToken, refreshToken, user }` |
| `POST` | `/api/iam/auth/refresh-token` | Public | Refresh → rotate tokens |
| `POST` | `/api/iam/auth/logout` | Bearer | Blacklist current access token |
| `POST` | `/api/iam/auth/logout-all` | Bearer | Revoke all refresh tokens for user |

#### Internal Endpoints (Gateway → IAM only, not routed externally)

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `POST` | `/internal/iam/verify` | api-gateway | Verify token, return `{ valid, userId, email, role }` |
| `POST` | `/internal/iam/issue` | identity-service | Issue tokens after register/Google login |

**DB Schema**:
```sql
-- tourism_iam

CREATE TABLE refresh_tokens (
    id           BIGSERIAL PRIMARY KEY,
    user_id      BIGINT NOT NULL,
    token_hash   VARCHAR(64) NOT NULL UNIQUE,   -- SHA-256 of raw token
    expires_at   TIMESTAMP NOT NULL,
    revoked      BOOLEAN DEFAULT FALSE,
    device_info  VARCHAR(255),
    created_at   TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_rt_user_id ON refresh_tokens(user_id);

CREATE TABLE token_blacklist (
    jti          VARCHAR(36) PRIMARY KEY,        -- JWT "jti" claim (UUID)
    expires_at   TIMESTAMP NOT NULL,             -- For scheduled cleanup
    blacklisted_at TIMESTAMP DEFAULT NOW()
);
CREATE INDEX idx_bl_expires ON token_blacklist(expires_at);
```

**Login Flow**:
```
POST /api/iam/auth/login { email, password }
  1. Feign → identity-service POST /internal/users/authenticate
  2. ← { userId, email, role, status }
  3. IF status == LOCKED → throw 403
  4. Generate accessToken: JWT(sub=userId, email, role, jti=UUID, exp=15min)
  5. Generate refreshToken: SecureRandom(32 bytes) → Base64
  6. Store SHA256(refreshToken) in refresh_tokens table
  7. Return { accessToken, refreshToken, expiresIn: 900 }
```

---

### 4.1 Identity Service `:8081`

**Database**: `tourism_identity`  
**Tables**: `users`, `email_verifications`  
**Dependencies**: Cloudinary, JavaMail, Google OAuth2  
**Kafka Producer**: `user.registered`

> ⚠️ **Changed from v1**: Removed `RefreshToken` table (moved to IAM). Removed JWT generation. Added internal endpoints for IAM to call.

#### Auth Endpoints

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/auth/register` | Public | `AuthController.register()` | Validate → save user → Feign IAM issue token → return tokens + user |
| `GET` | `/api/auth/verify-email` | Public | `AuthController.verifyEmail()` | Activate account via UUID token link |
| `POST` | `/api/auth/resend-verification` | Public | `AuthController.resendVerification()` | Rate-limited resend |
| `POST` | `/api/auth/google/login` | Public | `AuthController.googleLogin()` | Google ID Token → findOrCreate → Feign IAM issue token |
| `GET` | `/api/auth/profile` | USER | `AuthController.getMyProfile()` | Return user from `X-User-Id` header |

#### User Endpoints

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/users/{id}` | USER | `UserController.getUserById()` | Get user by ID |
| `PATCH` | `/api/users/{id}/profile` | USER | `UserController.updateProfile()` | Update name/phone/dob/address + avatar (Cloudinary multipart) |
| `PATCH` | `/api/users/{id}/change-password` | USER | `UserController.changePassword()` | Verify old BCrypt → set new |

#### Admin Endpoints

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/admin/users` | ADMIN | `AdminAuthController` | Paginated user list |
| `POST` | `/api/admin/users/search` | ADMIN | `UserController.searchUsers()` | Filter by keyword, role, status |
| `PATCH` | `/api/admin/users/{id}/status` | ADMIN | `AdminAuthController` | Lock / unlock account |

#### Internal Endpoints (called by IAM Service)

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `POST` | `/internal/users/authenticate` | iam-service | Verify email+password → return user info |
| `GET` | `/internal/users/{id}/info` | iam-service | Get user info for token enrichment |

**Register Flow**:
```
POST /api/auth/register { email, password, fullName }
  1. Validate email not taken
  2. BCrypt hash password → save User { status: PENDING_VERIFICATION }
  3. Generate email verification UUID (24h TTL) → send email (JavaMail)
  4. Kafka publish: user.registered
  5. Feign → IAM POST /internal/iam/issue { userId, email, role }
  6. ← { accessToken, refreshToken }
  7. Return 201 { accessToken, refreshToken, user }
```

---

### 4.2 Tour Catalog Service `:8082`

**Database**: `tourism_catalog`  
**Tables**: `tours`, `tour_images`, `tour_departures`, `departure_pricings`, `departure_transports`, `itinerary_days`, `locations`, `favorite_tours`  
**Dependencies**: Cloudinary

#### Public Endpoints

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/tours` | Public | `TourController.getAllTours()` | Paginated list, filter: `region`, `location`, `keyword`, `priceFrom`, `priceTo` |
| `GET` | `/api/tours/search` | Public | `TourController.searchTours()` | Full-text search |
| `GET` | `/api/tours/featured` | Public | `TourController.getTop10DeepestDiscountTours()` | Featured/most discounted tours |
| `GET` | `/api/tours/code/{code}` | Public | `TourController.getTourDetail()` | Tour detail by code (slug) |
| `GET` | `/api/tours/{id}` | Public | `TourController.getTourDetail()` | Tour detail by ID |
| `GET` | `/api/tours/{id}/related` | Public | `TourController.getRelatedTours()` | Tours in same region/location |
| `GET` | `/api/tours/{id}/departures` | Public | `TourController.getTourDepartures()` | All departure schedules |
| `GET` | `/api/tours/{id}/departures/available` | Public | *(NEW)* | Departures with `available_slots > 0` |
| `GET` | `/api/locations` | Public | `LocationController.getLocations()` | All locations (for filter dropdown) |
| `GET` | `/api/locations/{id}` | Public | `LocationController.getById()` | Location detail |

#### Authenticated User Endpoints

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/tours/favorites` | USER | `FavoriteTourController.addFavoriteTour()` | Add tour to favorites |
| `DELETE` | `/api/tours/favorites/{tourId}` | USER | `FavoriteTourController.removeFavoriteTour()` | Remove from favorites |
| `GET` | `/api/tours/favorites/my` | USER | `FavoriteTourController.getUserFavoriteTours()` | My favorites list |

#### Admin Tour Management

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/admin/tours` | ADMIN | `TourManagementController.getAllTours()` | All tours for admin table |
| `POST` | `/api/admin/tours` | ADMIN | `TourManagementController.createTour()` | Create tour + itinerary |
| `PUT` | `/api/admin/tours/{id}` | ADMIN | `TourManagementController.updateTour()` | Full update |
| `PUT` | `/api/admin/tours/{id}/general-info` | ADMIN | `TourManagementController.updateGeneralInfo()` | Update basic info |
| `PUT` | `/api/admin/tours/{id}/itinerary` | ADMIN | `TourManagementController.updateItinerary()` | Update day-by-day itinerary |
| `PATCH` | `/api/admin/tours/{id}/status` | ADMIN | `TourManagementController.updateStatus()` | Active / Inactive |
| `DELETE` | `/api/admin/tours/{id}` | ADMIN | `TourManagementController.deleteTour()` | Soft delete |
| `POST` | `/api/admin/tours/{id}/thumbnail` | ADMIN | `TourMediaController` | Upload thumbnail → Cloudinary |
| `POST` | `/api/admin/tours/{id}/images` | ADMIN | `TourUploadController` | Upload multiple gallery images |

#### Admin Departure Management

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/admin/departures` | ADMIN | `TourDepartureManagementController.createDeparture()` | Add departure schedule |
| `PUT` | `/api/admin/departures/{id}` | ADMIN | `TourDepartureManagementController.updateDeparture()` | Update departure |
| `PUT` | `/api/admin/departures/{id}/pricing` | ADMIN | `TourDepartureManagementController.updatePricing()` | Update price tiers |
| `PUT` | `/api/admin/departures/{id}/transport` | ADMIN | `TourDepartureManagementController.updateTransport()` | Update transport info |
| `POST` | `/api/admin/departures/{id}/clone` | ADMIN | `TourDepartureManagementController.cloneDeparture()` | Clone to new date |
| `DELETE` | `/api/admin/departures/{id}` | ADMIN | `TourDepartureManagementController.deleteDeparture()` | Delete departure |

#### Admin Location Management

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/admin/locations` | ADMIN | `LocationAdminController.create()` | Add new location |
| `PUT` | `/api/admin/locations/{id}` | ADMIN | `LocationAdminController.update()` | Update location |
| `DELETE` | `/api/admin/locations/{id}` | ADMIN | `LocationAdminController.delete()` | Delete location |

#### Internal Endpoints

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `GET` | `/internal/tours/departures/{id}` | booking-service | Get departure details + pricing for booking |
| `PUT` | `/internal/tours/departures/{id}/slots` | booking-service | Decrement `available_slots` atomically |
| `PUT` | `/internal/tours/departures/{id}/slots/restore` | booking-service | Restore slots on booking cancel |

---

### 4.3 Booking Service `:8083`

**Database**: `tourism_booking`  
**Tables**: `bookings`, `booking_passengers`, `refund_information`  
**Redis**: Cache booking by code (TTL=30min while PENDING, no expiry after CONFIRMED)  
**Feign**: → tour-catalog-service, → promotion-service  
**Kafka Producer**: `booking.created`, `booking.confirmed`, `booking.cancelled`

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/bookings/order` | USER | `BookingController.getBookingInitInfo()` | Pre-fetch tour + departure info for booking form |
| `POST` | `/api/bookings` | USER | `BookingController.createBooking()` | Create booking |
| `GET` | `/api/bookings/my` | USER | `BookingController.getAllBookingsByUser()` | User's bookings, filter by status |
| `GET` | `/api/bookings/code/{code}` | USER | `BookingController.getBookingDetail()` | Booking detail (with ownership check) |
| `POST` | `/api/bookings/{id}/cancel` | USER | `BookingController.cancelBooking()` | Cancel PENDING booking |
| `POST` | `/api/bookings/{id}/refund-request` | USER | `BookingController.requestRefund()` | Create refund request |
| `POST` | `/api/admin/bookings/search` | ADMIN | `BookingController.searchBookings()` | Admin search with filters |
| `PATCH` | `/api/admin/bookings/{id}/status` | ADMIN | `BookingController.updateBookingStatus()` | Force status update |

**Booking Status Machine**:
```
PENDING_PAYMENT ──(payment success)──► CONFIRMED
PENDING_PAYMENT ──(user cancel)──────► CANCELLED
PENDING_PAYMENT ──(30min timeout)────► EXPIRED
CONFIRMED ───────(admin cancel)──────► CANCELLED
CONFIRMED ───────(tour completed)────► COMPLETED
```

**Create Booking Flow**:
```
POST /api/bookings
  Body: { tourCode, departureId, couponCode?, passengers: [{ fullName, dob, type }] }

  1. GET Departure info:   Feign → tour-catalog GET /internal/tours/departures/{departureId}
  2. Validate coupon:      Feign → promotion    POST /internal/coupons/{code}/validate
  3. Calculate total:      Σ(passengers × price_by_type) - coupon_discount
  4. Lock coupon:          Feign → promotion    POST /internal/coupons/{code}/apply
  5. Decrement slots:      Feign → tour-catalog PUT /internal/tours/departures/{id}/slots
  6. Save Booking { status: PENDING_PAYMENT, expires_at: now+30min }
  7. Save BookingPassengers
  8. Kafka publish:        booking.created event
  9. Redis cache:          "booking:{code}" → BookingDto (TTL=30min)
  10. Return { bookingCode, totalAmount, expiresAt }
```

#### Internal Endpoints

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `POST` | `/internal/bookings/{code}/confirm` | payment-service | CONFIRMED + Kafka `booking.confirmed` |
| `GET` | `/internal/bookings/check-confirmed` | review-service | Check `?userId=&tourCode=` has CONFIRMED booking |

---

### 4.4 Payment Service `:8084`

**Database**: `tourism_payment`  
**Tables**: `payments`  
**Feign**: → booking-service  
**Kafka Producer**: `payment.completed`

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/payments/vnpay/create` | USER | `PaymentController.createVNPayPayment()` | Generate VNPay redirect URL |
| `GET` | `/api/payments/vnpay-return` | Public | `PaymentController.vnpayReturn()` | VNPay redirect callback |
| `POST` | `/api/payments/payos/create` | USER | `PaymentController.createPayOSPayment()` | Create PayOS checkout + QR |
| `POST` | `/api/payments/payos-webhook` | Public | `PaymentController.handlePayOSWebhook()` | PayOS server-to-server webhook |
| `GET` | `/api/payments/payos/return` | Public | `PaymentController.payosReturn()` | PayOS redirect on success |
| `GET` | `/api/payments/payos/cancel` | Public | `PaymentController.payosCancel()` | PayOS redirect on cancel |
| `POST` | `/api/payments/sepay-webhook` | Public | `PaymentController.handleSepayWebhook()` | SePay bank transfer webhook |
| `GET` | `/api/payments/status/{orderCode}` | USER | `PaymentController.getPaymentStatus()` | Query payment status |
| `GET` | `/api/payments/booking/{bookingCode}` | USER | *(NEW)* | Get payment for a booking |

**VNPay Flow**:
```
POST /api/payments/vnpay/create { bookingCode, returnUrl }
  → Save Payment { status: PENDING, method: VNPAY }
  → Build VNPay URL (HMAC-SHA512 signed)
  ← { paymentUrl }

GET /api/payments/vnpay-return?vnp_ResponseCode=00&vnp_SecureHash=...
  → Verify HMAC-SHA512 signature
  → if 00 (success):
      Feign → booking-service POST /internal/bookings/{code}/confirm
      Update Payment { status: COMPLETED }
      Kafka publish: payment.completed
  → Redirect to frontend /payment/result?success=true&bookingCode=...
```

---

### 4.5 Review Service `:8085`

**Database**: `tourism_review`  
**Tables**: `reviews`, `review_images`  
**Feign**: → booking-service  
**Cloudinary**: Review image upload  
**Kafka Producer**: `review.created`

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `POST` | `/api/reviews` | USER | `ReviewController.submitReview()` | Submit review (multipart) |
| `GET` | `/api/reviews/booking/{bookingCode}` | USER | `ReviewController.getReview()` | Review for specific booking |
| `GET` | `/api/reviews/eligibility/{bookingCode}` | USER | *(NEW)* | Can user review this booking? |
| `GET` | `/api/reviews/tour/{code}` | Public | `ReviewController.getReviewsByTour()` | All reviews for a tour, paginated |
| `GET` | `/api/reviews/tour/{code}/summary` | Public | `ReviewController.getReviewStatistics()` | Avg rating + star distribution |
| `GET` | `/api/reviews/my` | USER | *(NEW)* | My reviews |
| `DELETE` | `/api/reviews/{id}` | USER | *(NEW)* | Delete own review |
| `DELETE` | `/api/admin/reviews/{id}` | ADMIN | *(NEW)* | Admin delete any review |

**Review Submission Flow**:
```
POST /api/reviews (multipart/form-data)
  Fields: bookingCode, rating (1-5), comment, images[] (optional)

  1. Read X-User-Id from header
  2. Feign → booking-service GET /internal/bookings/check-confirmed
     ?userId={userId}&tourCode={tourCode}
  3. if not confirmed → 403 "Bạn chưa hoàn thành tour này"
  4. if already reviewed → 409 "Bạn đã đánh giá tour này"
  5. Upload images[] → Cloudinary → get secure_urls
  6. Save Review + ReviewImages
  7. Kafka publish: review.created { tourId, tourCode, rating }
```

---

### 4.6 Promotion Service `:8086`

**Database**: `tourism_promotion`  
**Tables**: `coupons`

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/promotions/coupons/{code}` | USER | `CouponController.validateCoupon()` | Preview coupon info before applying |
| `GET` | `/api/admin/coupons` | ADMIN | `CouponController.getAllCoupons()` | All coupons with pagination |
| `GET` | `/api/admin/coupons/search` | ADMIN | `CouponController.searchCoupons()` | Search by keyword |
| `POST` | `/api/admin/coupons` | ADMIN | `CouponController.createCoupon()` | Create coupon |
| `PUT` | `/api/admin/coupons/{id}` | ADMIN | `CouponController.updateCoupon()` | Update coupon |
| `DELETE` | `/api/admin/coupons/{id}` | ADMIN | `CouponController.deleteCoupon()` | Delete coupon |

#### Internal Endpoints

| Method | Path | Caller | Description |
|--------|------|--------|-------------|
| `POST` | `/internal/coupons/{code}/validate` | booking-service | Validate (read-only check, no side effects) |
| `POST` | `/internal/coupons/{code}/apply` | booking-service | Atomically decrement `usage_count` |
| `POST` | `/internal/coupons/{code}/restore` | booking-service | Restore on booking cancel |

---

### 4.7 Notification Service `:8087`

**Database**: `tourism_notification`  
**Tables**: `notifications`, `user_notifications`  
**Kafka Consumer**: 5 topics  
**JavaMail**: Gmail SMTP + HTML templates

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/notifications/my` | USER | `NotificationController.getMyNotifications()` | In-app notification list |
| `GET` | `/api/notifications/unread-count` | USER | `NotificationController.getUnreadCount()` | Badge count |
| `PUT` | `/api/notifications/{id}/read` | USER | `NotificationController.markAsRead()` | Mark one as read |
| `PUT` | `/api/notifications/read-all` | USER | `NotificationController.markAllAsRead()` | Mark all as read |

**Kafka Event → Action Mapping**:

| Topic | Email Subject | In-App Message |
|-------|--------------|----------------|
| `user.registered` | "Chào mừng {name} đến với Tourism!" | "Tài khoản đã tạo thành công" |
| `booking.created` | "Đặt tour thành công — Mã #{code}" | "Đặt tour {tourName} thành công" |
| `booking.confirmed` | "Tour của bạn đã được xác nhận" | "Booking #{code} đã xác nhận" |
| `booking.cancelled` | "Thông báo: Tour bị huỷ" | "Booking #{code} đã bị huỷ" |
| `payment.completed` | "Biên lai thanh toán {amount}đ" | "Thanh toán thành công" |

---

### 4.8 Analytics Service `:8088`

**Database**: `tourism_analytics`  
**Tables**: `daily_stats`, `tour_stats`  
**Kafka Consumer**: 5 topics  
**External**: Google Gemini API

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/analytics/dashboard/summary` | ADMIN | `DashboardController.getDashboardStatistics()` | Total revenue, bookings, users, tours |
| `GET` | `/api/analytics/dashboard/daily-stats` | ADMIN | `DashboardController.getDashboardAIAnalysis()` | Last 7/30 days stats for charts |
| `GET` | `/api/analytics/dashboard/top-tours` | ADMIN | *(NEW)* | Top tours by revenue/bookings |
| `POST` | `/api/analytics/chatbot/chat` | ADMIN | `ChatbotController.chat()` | AI chatbot powered by Gemini |

**Kafka → Stats Update**:
```
booking.created    →  daily_stats.new_bookings++
booking.confirmed  →  daily_stats.confirmed_bookings++
booking.cancelled  →  daily_stats.cancelled_bookings++
payment.completed  →  daily_stats.total_revenue += amount
                       tour_stats.total_revenue += amount
review.created     →  tour_stats: recalculate avg_rating
```

**Chatbot System Prompt** (inject context):
```
Bạn là trợ lý AI cho quản lý hệ thống du lịch. Dữ liệu hôm nay:
- Doanh thu: {today_revenue}đ
- Bookings mới: {new_bookings}
- Tours active: {active_tours}
Trả lời câu hỏi về dữ liệu kinh doanh bằng tiếng Việt.
```

---

### 4.9 CMS Service `:8089`

**Database**: `tourism_cms`  
**Tables**: `policy_templates`, `branch_contacts`

| Method | Path | Auth | Monolith Source | Description |
|--------|------|------|-----------------|-------------|
| `GET` | `/api/cms/policies` | Public | `PolicyTemplateController.getPolicies()` | All policies |
| `GET` | `/api/cms/policies/type/{type}` | Public | `PolicyTemplateController.getPolicyByType()` | Latest policy by type |
| `POST` | `/api/cms/policies` | ADMIN | `PolicyTemplateController.createPolicy()` | Create policy |
| `PUT` | `/api/cms/policies/{id}` | ADMIN | `PolicyTemplateController.updatePolicy()` | Update policy |
| `DELETE` | `/api/cms/policies/{id}` | ADMIN | `PolicyTemplateController.deletePolicy()` | Delete policy |
| `GET` | `/api/cms/branches` | Public | `BranchContactController.getBranches()` | All branch offices |
| `POST` | `/api/cms/branches` | ADMIN | `BranchContactController.createBranch()` | Add branch |
| `PUT` | `/api/cms/branches/{id}` | ADMIN | `BranchContactController.updateBranch()` | Update branch |
| `DELETE` | `/api/cms/branches/{id}` | ADMIN | `BranchContactController.deleteBranch()` | Delete branch |

---

## 5. Inter-Service Communication

### 5.1 Feign Client Dependency Graph

```
iam-service
  └──► identity-service     POST /internal/users/authenticate
  └──► identity-service     GET  /internal/users/{id}/info

identity-service
  └──► iam-service          POST /internal/iam/issue (after register/google login)

booking-service
  └──► tour-catalog-service GET  /internal/tours/departures/{id}
  └──► tour-catalog-service PUT  /internal/tours/departures/{id}/slots
  └──► tour-catalog-service PUT  /internal/tours/departures/{id}/slots/restore
  └──► promotion-service    POST /internal/coupons/{code}/validate
  └──► promotion-service    POST /internal/coupons/{code}/apply
  └──► promotion-service    POST /internal/coupons/{code}/restore

payment-service
  └──► booking-service      POST /internal/bookings/{code}/confirm

review-service
  └──► booking-service      GET  /internal/bookings/check-confirmed
```

### 5.2 Internal Endpoint Security

```yaml
# All /internal/** endpoints:
# 1. NOT routed through API Gateway (not in route table)
# 2. Only reachable within Docker bridge network (tourism-net)
# 3. No JWT required — pure service-to-service
# 4. Add X-Internal-Secret header for extra security (optional)

# application.yml for each service:
internal.secret: ${INTERNAL_SECRET:dev-secret-only}

# In InternalController:
@RequestHeader("X-Internal-Secret") String secret  # validate this
```

---

## 6. Kafka Event Bus

### 6.1 Event Definitions (`common-events` library)

```java
// shared-libs/common-events/src/main/java/com/tourism/events/

public record UserRegisteredEvent(
    Long userId, String email, String fullName, Instant registeredAt
) {}

public record BookingCreatedEvent(
    String bookingCode, Long userId, String userEmail,
    String tourCode, String tourName, String departureDate,
    BigDecimal totalAmount, Integer passengerCount, Instant createdAt
) {}

public record BookingConfirmedEvent(
    String bookingCode, Long userId, String userEmail,
    String tourName, String tourCode, String departureDate
) {}

public record BookingCancelledEvent(
    String bookingCode, Long userId, String userEmail,
    String tourName, String reason, Instant cancelledAt
) {}

public record PaymentCompletedEvent(
    String bookingCode, Long userId, String userEmail,
    BigDecimal amount, String paymentMethod, String orderId, Instant paidAt
) {}

public record ReviewCreatedEvent(
    Long reviewId, Long tourId, String tourCode,
    Long userId, Integer rating, Instant createdAt
) {}
```

### 6.2 Event Flow Matrix

```
PRODUCER            TOPIC                  CONSUMERS
──────────────      ─────────────────────  ────────────────────────────
identity-service  → user.registered      → notification-service
                                          → analytics-service

booking-service   → booking.created      → notification-service
                                          → analytics-service

booking-service   → booking.confirmed    → notification-service
                                          → analytics-service

booking-service   → booking.cancelled    → notification-service
                                          → analytics-service

payment-service   → payment.completed    → notification-service
                                          → analytics-service

review-service    → review.created       → analytics-service
```

---

## 7. Database Decomposition

### 7.1 Schema: `tourism_iam`

```sql
CREATE TABLE refresh_tokens (
    id           BIGSERIAL PRIMARY KEY,
    user_id      BIGINT NOT NULL,
    token_hash   VARCHAR(64) NOT NULL UNIQUE,
    expires_at   TIMESTAMP NOT NULL,
    revoked      BOOLEAN DEFAULT FALSE,
    device_info  VARCHAR(255),
    created_at   TIMESTAMP DEFAULT NOW()
);

CREATE TABLE token_blacklist (
    jti            VARCHAR(36) PRIMARY KEY,
    expires_at     TIMESTAMP NOT NULL,
    blacklisted_at TIMESTAMP DEFAULT NOW()
);
```

### 7.2 Schema: `tourism_identity`

```sql
CREATE TABLE users (
    id               BIGSERIAL PRIMARY KEY,
    email            VARCHAR(255) UNIQUE NOT NULL,
    password_hash    VARCHAR(60),           -- null for Google users
    full_name        VARCHAR(255),
    phone            VARCHAR(20),
    date_of_birth    DATE,
    avatar_url       VARCHAR(512),
    role             VARCHAR(20) DEFAULT 'USER',  -- USER | ADMIN
    status           VARCHAR(20) DEFAULT 'PENDING_VERIFICATION',
    google_id        VARCHAR(255),
    province_code    VARCHAR(10),
    province_name    VARCHAR(100),
    district_code    VARCHAR(10),
    district_name    VARCHAR(100),
    created_at       TIMESTAMP DEFAULT NOW(),
    updated_at       TIMESTAMP DEFAULT NOW()
);

CREATE TABLE email_verifications (
    id         BIGSERIAL PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    token      VARCHAR(36) UNIQUE NOT NULL,  -- UUID
    expires_at TIMESTAMP NOT NULL,
    used       BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### 7.3 Schema: `tourism_catalog`

```sql
CREATE TABLE locations (
    id        BIGSERIAL PRIMARY KEY,
    name      VARCHAR(255) NOT NULL,
    region    VARCHAR(100),
    latitude  DECIMAL(10,8),
    longitude DECIMAL(11,8)
);

CREATE TABLE tours (
    id             BIGSERIAL PRIMARY KEY,
    code           VARCHAR(50) UNIQUE NOT NULL,
    title          VARCHAR(500) NOT NULL,
    description    TEXT,
    highlights     TEXT,
    thumbnail_url  VARCHAR(512),
    region         VARCHAR(100),
    location_id    BIGINT REFERENCES locations(id),
    category       VARCHAR(100),
    status         VARCHAR(20) DEFAULT 'ACTIVE',
    price_from     DECIMAL(15,2),
    duration_days  INTEGER,
    booking_count  INTEGER DEFAULT 0,
    created_at     TIMESTAMP DEFAULT NOW()
);

CREATE TABLE tour_images (id, tour_id, url, type, sort_order);
CREATE TABLE itinerary_days (id, tour_id, day_number, title, description, meals, accommodation);
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
CREATE TABLE departure_pricings (id, departure_id, passenger_type, price, description);
CREATE TABLE departure_transports (id, departure_id, type, details, departure_point, return_point);
CREATE TABLE favorite_tours (id, user_id, tour_id, created_at, UNIQUE(user_id, tour_id));
```

### 7.4 Schema: `tourism_booking`

```sql
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
    note            TEXT,
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE booking_passengers (
    id           BIGSERIAL PRIMARY KEY,
    booking_id   BIGINT NOT NULL,
    full_name    VARCHAR(255) NOT NULL,
    date_of_birth DATE,
    id_number    VARCHAR(50),
    type         VARCHAR(20),  -- ADULT | CHILD | INFANT
    price        DECIMAL(15,2)
);

CREATE TABLE refund_information (
    id              BIGSERIAL PRIMARY KEY,
    booking_id      BIGINT UNIQUE NOT NULL,
    bank_name       VARCHAR(255),
    account_number  VARCHAR(50),
    account_holder  VARCHAR(255),
    amount          DECIMAL(15,2),
    status          VARCHAR(20) DEFAULT 'PENDING',
    requested_at    TIMESTAMP DEFAULT NOW()
);
```

### 7.5 Cross-Service Reference Strategy

```
No foreign keys across service boundaries.
Cross-service references are stored as plain IDs (BIGINT) with a snapshot
of key display data to avoid Feign calls at read time.

Example — booking stores:
  user_id    BIGINT  (references identity.users.id — no FK constraint)
  tour_id    BIGINT  (references catalog.tours.id — no FK constraint)
  tour_code  VARCHAR (snapshot — avoids tour-catalog call for display)
  tour_name  VARCHAR (snapshot — for display without cross-service call)
```

---

## 8. Infrastructure & External Integrations

### 8.1 Infrastructure Stack

| Component | Port | Image | Purpose |
|-----------|------|-------|---------|
| Apache Kafka | `9092` | `confluentinc/cp-kafka:7.5.0` | Async event streaming |
| Zookeeper | `2181` | `confluentinc/cp-zookeeper:7.5.0` | Kafka coordinator |
| PostgreSQL 15 | `5432` | `postgres:15-alpine` | Single instance, 10 logical databases |
| Redis 7 | `6379` | `redis:7-alpine` | Token blacklist cache + booking cache |
| Zipkin | `9411` | `openzipkin/zipkin` | Distributed tracing |
| Eureka | `8761` | *(custom Spring Boot)* | Service registry |

### 8.2 External Integrations

| Service | Used by | Method |
|---------|---------|--------|
| **Cloudinary CDN** | identity, tour-catalog, review | Java SDK |
| **Gmail SMTP** | identity, notification | Spring Mail |
| **Google OAuth2** | identity-service | `google-api-client` library |
| **VNPay** | payment-service | REST URL + HMAC-SHA512 |
| **PayOS** | payment-service | REST API + webhook |
| **SePay** | payment-service | Webhook |
| **Google Gemini** | analytics-service | REST API |
| **ngrok** | Dev local testing | Expose webhook endpoints |

### 8.3 Complete `.env` Reference

```bash
# ── JWT (only for iam-service and api-gateway) ──
JWT_SECRET=your-64-char-random-secret-here
JWT_ACCESS_EXPIRY_MS=900000       # 15 minutes
JWT_REFRESH_EXPIRY_MS=604800000   # 7 days

# ── Internal service communication ──
INTERNAL_SECRET=change-this-in-production

# ── Database ──
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres

# ── Redis ──
REDIS_HOST=redis
REDIS_PORT=6379

# ── Kafka ──
KAFKA_BOOTSTRAP_SERVERS=kafka:29092

# ── Cloudinary ──
CLOUDINARY_CLOUD_NAME=your-cloud-name
CLOUDINARY_API_KEY=your-api-key
CLOUDINARY_API_SECRET=your-api-secret

# ── JavaMail (Gmail) ──
MAIL_USERNAME=your-email@gmail.com
MAIL_PASSWORD=your-app-password   # 16-char Google App Password

# ── Google OAuth2 ──
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=GOCSPX-xxx

# ── Payment: VNPay ──
VNPAY_TMN_CODE=your-tmn-code
VNPAY_HASH_SECRET=your-hash-secret
VNPAY_URL=https://sandbox.vnpayment.vn/paymentv2/vpcpay.html
VNPAY_RETURN_URL=${BASE_URL}/api/payments/vnpay-return

# ── Payment: PayOS ──
PAYOS_CLIENT_ID=your-client-id
PAYOS_API_KEY=your-api-key
PAYOS_CHECKSUM_KEY=your-checksum-key

# ── AI ──
GEMINI_API_KEY=your-gemini-key

# ── App base URL (update for production) ──
BASE_URL=http://localhost:8080
FRONTEND_URL=http://localhost:5173
```

---

## 9. Project Structure

### 9.1 Root Layout

```
D:\KLTN\tourism-microservices-v2\
├── pom.xml                       # Root Maven POM — manages all modules
├── docker-compose.yml            # Full stack production compose
├── docker-compose.dev.yml        # Dev override (host mounts if needed)
├── init-db.sql                   # Creates 10 PostgreSQL databases
├── .env.example                  # Template with all required vars
├── .gitignore
├── README.md
│
├── shared-libs/
│   └── common-events/            # Kafka event DTOs only (NO security lib)
│       ├── pom.xml
│       └── src/main/java/com/tourism/events/
│           ├── UserRegisteredEvent.java
│           ├── BookingCreatedEvent.java
│           ├── BookingConfirmedEvent.java
│           ├── BookingCancelledEvent.java
│           ├── PaymentCompletedEvent.java
│           └── ReviewCreatedEvent.java
│
├── infrastructure/
│   ├── service-registry/         # Eureka Server :8761
│   │   ├── pom.xml
│   │   ├── Dockerfile
│   │   └── src/...
│   └── api-gateway/              # Spring Cloud Gateway :8080
│       ├── pom.xml
│       ├── Dockerfile
│       └── src/main/java/com/tourism/gateway/
│           ├── filter/
│           │   └── JwtAuthGatewayFilter.java   # Calls IAM verify
│           ├── config/
│           │   ├── RouteConfig.java
│           │   └── PublicPathConfig.java
│           └── client/
│               └── IamClient.java              # WebClient to IAM
│
└── services/
    ├── iam-service/              # :8090 — NEW
    ├── identity-service/         # :8081
    ├── tour-catalog-service/     # :8082
    ├── booking-service/          # :8083
    ├── payment-service/          # :8084
    ├── review-service/           # :8085
    ├── promotion-service/        # :8086
    ├── notification-service/     # :8087
    ├── analytics-service/        # :8088
    └── cms-service/              # :8089
```

### 9.2 Standard Service Layout

```
services/{service-name}/
├── pom.xml
├── Dockerfile
└── src/
    ├── main/
    │   ├── java/com/tourism/{domain}/
    │   │   ├── {Domain}Application.java
    │   │   ├── controller/
    │   │   │   ├── {Public}Controller.java      # /api/** endpoints
    │   │   │   ├── {Admin}Controller.java        # /api/admin/** endpoints
    │   │   │   └── InternalController.java       # /internal/** (Feign only)
    │   │   ├── service/
    │   │   ├── repository/
    │   │   ├── entity/
    │   │   ├── dto/
    │   │   │   ├── request/
    │   │   │   └── response/
    │   │   ├── client/                           # Feign clients (if any)
    │   │   ├── event/
    │   │   │   ├── producer/                     # KafkaTemplate producers
    │   │   │   └── consumer/                     # @KafkaListener consumers
    │   │   ├── config/
    │   │   │   ├── SecurityConfig.java           # Uses HeaderAuthFilter
    │   │   │   ├── HeaderAuthFilter.java         # ~30 lines, reads X-User-*
    │   │   │   └── UserPrincipal.java            # record { userId, email, role }
    │   │   └── exception/
    │   │       └── GlobalExceptionHandler.java
    │   └── resources/
    │       ├── application.yml
    │       └── db/migration/
    │           └── V1__init.sql
    └── test/
        └── java/com/tourism/{domain}/
            ├── controller/                       # MockMvc tests
            └── service/                          # Unit tests
```

### 9.3 Root `pom.xml` Structure

```xml
<modules>
    <!-- Shared Event DTOs only -->
    <module>shared-libs/common-events</module>

    <!-- Infrastructure -->
    <module>infrastructure/service-registry</module>
    <module>infrastructure/api-gateway</module>

    <!-- Business Services -->
    <module>services/iam-service</module>
    <module>services/identity-service</module>
    <module>services/tour-catalog-service</module>
    <module>services/booking-service</module>
    <module>services/payment-service</module>
    <module>services/review-service</module>
    <module>services/promotion-service</module>
    <module>services/notification-service</module>
    <module>services/analytics-service</module>
    <module>services/cms-service</module>
</modules>

<properties>
    <java.version>17</java.version>
    <spring-boot.version>3.3.0</spring-boot.version>
    <spring-cloud.version>2023.0.2</spring-cloud.version>
    <!-- NO jjwt in most services — only iam-service and api-gateway -->
</properties>
```

---

## 10. 30-Day Master Plan

### Timeline Overview

```
Week 1  (D01–D07)  Infrastructure Scaffold + IAM Service + API Gateway
Week 2  (D08–D14)  Identity + Tour Catalog + Booking Services
Week 3  (D15–D21)  Payment + Review + Promotion + Notification + Analytics + CMS
Week 4  (D22–D27)  React Frontend (Auth → Tours → Booking → Admin)
Days 28–30         End-to-End Testing + Dockerize + Deploy AWS
```

---

### 📅 Day 1 — Project Scaffold & Infrastructure

**Goal**: New project created, infra containers running, Eureka healthy

**Tasks**:
```bash
mkdir D:\KLTN\tourism-microservices-v2
cd D:\KLTN\tourism-microservices-v2
git init && git remote add origin <repo-url>
```

- [ ] Create root `pom.xml` with all module declarations
- [ ] Create `shared-libs/common-events/` with 6 event record classes
- [ ] Create `infrastructure/service-registry/` (port from v1, no changes)
- [ ] Create `docker-compose.yml`:
  - zookeeper, kafka, postgres (single instance), redis, zipkin, service-registry
- [ ] Create `init-db.sql`:
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
- [ ] Create `.env.example` with all variables documented
- [ ] `docker-compose up -d zookeeper kafka postgres redis zipkin service-registry`
- [ ] Verify: `http://localhost:8761` (Eureka dashboard)

**Git commit**: `chore: project scaffold + infrastructure v2`

---

### 📅 Day 2 — IAM Service: Token Engine

**Goal**: IAM can issue and verify JWT tokens independently

**Core classes**:
```java
// TokenService.java
String generateAccessToken(Long userId, String email, String role)
// → JWT: sub=userId, email, role, jti=UUID, iat, exp=now+15min

String generateRefreshToken(Long userId)
// → SecureRandom(32 bytes) → Base64url → hash SHA256 → save to DB

TokenVerifyResponse verify(String rawToken)
// → parse JWT (catch ExpiredJwtException, SignatureException)
// → check Redis blacklist (key=jti)
// → if miss Redis: check DB token_blacklist
// → return { valid, userId, email, role }

void blacklist(String jti, Instant expiresAt)
// → Redis SET jti "" EX <remaining_seconds>
// → INSERT INTO token_blacklist
```

- [ ] IAM `pom.xml`: jjwt-api, jjwt-impl, spring-boot-starter-data-jpa, spring-data-redis
- [ ] Flyway `V1__init_iam.sql`
- [ ] `TokenService` + `RefreshTokenService`
- [ ] `POST /internal/iam/verify` endpoint
- [ ] `POST /internal/iam/issue` endpoint
- [ ] Redis connection config
- [ ] Unit tests: `TokenServiceTest` (valid, expired, blacklisted)

---

### 📅 Day 3 — IAM Service: Login, Logout, Refresh

**Goal**: Login → tokens → refresh → logout → tokens dead

- [ ] `IamAuthController`: `/api/iam/auth/login|logout|logout-all|refresh-token`
- [ ] `IdentityServiceClient` (Feign to identity-service)
- [ ] `POST /api/iam/auth/login` full flow (see §4.0)
- [ ] `POST /api/iam/auth/logout` — blacklist jti + revoke refresh token
- [ ] `POST /api/iam/auth/logout-all` — revoke all refresh tokens for userId
- [ ] `POST /api/iam/auth/refresh-token` — validate hash → issue new pair (rotation)
- [ ] Scheduled cleanup: `@Scheduled(cron="0 0 * * * *")` delete expired blacklist rows
- [ ] Postman tests: login → verify → logout → verify dead

---

### 📅 Day 4 — API Gateway: JWT Filter → IAM

**Goal**: Gateway calls IAM to verify; injects headers; public paths bypass

```java
// JwtAuthGatewayFilter.java
// Key points:
// 1. Caffeine cache: token → TokenVerifyResponse (TTL=60s)
// 2. Path matcher for public paths from config
// 3. WebClient (non-blocking) to iam-service
// 4. On IAM error (timeout etc.) → return 503, not 401
```

- [ ] `api-gateway/pom.xml`: spring-cloud-starter-gateway, spring-cloud-starter-netflix-eureka-client, caffeine, spring-boot-starter-webflux
- [ ] `JwtAuthGatewayFilter.java` with Caffeine cache
- [ ] `RouteConfig.java` with all routes (see §3.3)
- [ ] `PublicPathConfig.java` loaded from application.yml
- [ ] CORS config: allow `http://localhost:5173`
- [ ] Test matrix:
  - Public route without token → 200
  - Protected route without token → 401
  - Protected route with valid token → forward with X-User-* headers
  - Protected route with expired token → 401
  - Protected route after logout → 401 (within 60s cache TTL: may briefly succeed)

---

### 📅 Day 5 — Identity Service: User Registration + Email Verify

**Goal**: Register → email sent → click verify → account active

- [ ] `identity-service/pom.xml`: NO jjwt dependency
- [ ] Flyway `V1__init_identity.sql`
- [ ] `User` entity, `EmailVerification` entity
- [ ] `POST /api/auth/register`:
  - validate uniqueness
  - BCrypt hash password
  - save User (status=PENDING)
  - save EmailVerification (UUID, 24h)
  - send HTML email (JavaMail)
  - Kafka publish: user.registered
  - Feign → IAM POST /internal/iam/issue → get tokens
  - return 201 with tokens
- [ ] `GET /api/auth/verify-email?token=`
- [ ] `POST /api/auth/resend-verification`
- [ ] JavaMail HTML template for verification email
- [ ] `InternalUserController`:
  - `POST /internal/users/authenticate`
  - `GET /internal/users/{id}/info`

---

### 📅 Day 6 — Identity Service: Google OAuth + Profile + Admin

**Goal**: All identity-service endpoints complete

- [ ] `POST /api/auth/google/login`:
  - Verify Google ID Token via google-api-client
  - findOrCreate user
  - Feign → IAM issue token
- [ ] `GET /api/auth/profile` (reads X-User-Id header)
- [ ] `PATCH /api/users/{id}/profile` (multipart + Cloudinary)
- [ ] `PATCH /api/users/{id}/change-password`
- [ ] `GET /api/admin/users`, `POST /api/admin/users/search`
- [ ] `PATCH /api/admin/users/{id}/status`
- [ ] `HeaderAuthFilter.java` + `SecurityConfig.java` (based on §2.5)
- [ ] Unit tests for UserService

---

### 📅 Day 7 — Buffer + Week 1 E2E Test

**Full E2E Scenario**:
```
1. Register       → 201 + tokens
2. Verify email   → 200 (account active)
3. Login          → 200 + tokens
4. Get profile    → 200 (using X-User-Id header from gateway)
5. Update avatar  → 200 (Cloudinary URL in response)
6. Google login   → 200 + tokens
7. Logout         → 200
8. Use old token  → 401 (blacklisted)
9. Logout-all     → 200
10. Use refresh   → 401 (revoked)
```

**Git tag**: `v0.1-iam-identity`

---

### 📅 Day 8 — Tour Catalog: DB Schema + Public APIs

**Goal**: Public tour browsing works end-to-end

- [ ] Flyway `V1__init_catalog.sql` (all 8 tables)
- [ ] Port entities from monolith
- [ ] `HeaderAuthFilter` (copy §2.5 pattern)
- [ ] Public APIs:
  - `GET /api/tours` (filter + pagination)
  - `GET /api/tours/featured`
  - `GET /api/tours/code/{code}` + `GET /api/tours/{id}`
  - `GET /api/tours/{id}/departures/available`
  - `GET /api/locations`
- [ ] Test: `curl http://localhost:8080/api/tours` → JSON array

---

### 📅 Day 9 — Tour Catalog: Admin APIs + Cloudinary + Internal

- [ ] Admin tour CRUD (8 endpoints) + `@PreAuthorize("hasRole('ADMIN')")`
- [ ] Admin departure management (6 endpoints)
- [ ] Admin location management (3 endpoints)
- [ ] Favorite tour endpoints (3)
- [ ] Cloudinary config + thumbnail/gallery upload
- [ ] Internal endpoints:
  - `GET /internal/tours/departures/{id}`
  - `PUT /internal/tours/departures/{id}/slots` (use `@Lock(PESSIMISTIC_WRITE)`)
  - `PUT /internal/tours/departures/{id}/slots/restore`

---

### 📅 Day 10 — Booking Service: Create Booking

- [ ] Flyway `V1__init_booking.sql`
- [ ] Feign clients: `TourCatalogClient`, `PromotionClient`
- [ ] Redis config for booking cache
- [ ] `POST /api/bookings` — full create flow (§4.3)
- [ ] `GET /api/bookings/order?tourCode=&departureId=`
- [ ] `GET /api/bookings/my`
- [ ] `GET /api/bookings/code/{code}`

---

### 📅 Day 11 — Booking Service: Cancel + Internal

- [ ] `POST /api/bookings/{id}/cancel` + slot restore Feign call
- [ ] `POST /api/bookings/{id}/refund-request`
- [ ] Admin search + status update
- [ ] `POST /internal/bookings/{code}/confirm`
- [ ] `GET /internal/bookings/check-confirmed`
- [ ] Kafka producers for all booking events
- [ ] `@Scheduled` cleanup for EXPIRED bookings

---

### 📅 Day 12 — Payment Service

- [ ] Port `PaymentController` from monolith (minimal changes)
- [ ] VNPay: URL generation + HMAC verify + return handler
- [ ] PayOS: checkout create + webhook + redirect handlers
- [ ] SePay: webhook handler
- [ ] Feign → booking-service confirm
- [ ] Kafka: payment.completed
- [ ] ngrok setup for local webhook testing:
  ```bash
  ngrok http 8080
  # Update VNPay/PayOS sandbox returnUrl to ngrok URL
  ```

---

### 📅 Day 13 — Review + Promotion Services

**Promotion**:
- [ ] CRUD coupons + internal endpoints (validate/apply/restore)

**Review**:
- [ ] Flyway migration
- [ ] Feign → booking-service
- [ ] Cloudinary for review images
- [ ] All review endpoints (§4.5)
- [ ] Kafka: review.created

---

### 📅 Day 14 — Notification + Analytics + CMS Services

**Notification**:
- [ ] 5 Kafka `@KafkaListener` handlers
- [ ] 5 HTML email templates
- [ ] 4 REST endpoints

**Analytics**:
- [ ] DB: daily_stats + tour_stats
- [ ] 5 Kafka consumers → stat updates
- [ ] 3 dashboard REST endpoints
- [ ] Gemini chatbot endpoint

**CMS**:
- [ ] PolicyTemplate CRUD
- [ ] BranchContact CRUD

**Git tag**: `v1.0-backend-complete`  
**Verify**: `docker-compose up -d` → 12 containers healthy in Eureka

---

### 📅 Day 15 — Integration Test: Full Backend

**Complete business flow test**:
```
Register → Verify Email → Login
→ Browse Tours → View Detail → Check Departures
→ Create Booking (with coupon) → VNPay Payment
→ Booking CONFIRMED → Email Received → Notification In-App
→ View My Bookings → Write Review
→ Admin: View Dashboard → Stats Updated → AI Chatbot
→ Logout-All → Tokens Dead
```

Fix all bugs. Export Postman Collection.

---

### 📅 Day 16 — Frontend Migration: CRA → Vite Setup

> **Strategy**: Không viết lại từ đầu — **migrate** codebase `D:\KLTN\client-side` sang Vite.  
> Giữ nguyên toàn bộ components, styles, và UI logic. Chỉ thay đổi:
> 1. Build tool: CRA (`react-scripts`) → Vite  
> 2. Auth endpoints: `/api/auth/*` → `/api/iam/auth/*` (cho login/logout/refresh)
> 3. axiosCustomize.js → axiosInstance.ts (TypeScript + point to IAM)

**Existing components to keep** (`D:\KLTN\client-side\src\components\`):
```
HeaderComponent/          → Giữ nguyên, update auth calls
FooterComponent/          → Giữ nguyên
LayoutComponent/          → Giữ nguyên (MainLayout)
homPageComponent/         → Giữ nguyên
toursPageComponent/       → Giữ nguyên
TourDetailComponent/      → Giữ nguyên
TourBookingComponent/     → Giữ nguyên
BookingPaymentComponent/  → Giữ nguyên (PaymentSuccess/Failed/Waiting/Error)
Login/                    → Update: POST /api/iam/auth/login
RegisterComponent/        → Update: POST /api/auth/register (identity)
VerifyEmail/              → Giữ nguyên
InformationComponent/     → Giữ nguyên (profile + bookings)
AdminComponent/           → Giữ nguyên + update API calls
ChatbotWidget/            → Giữ nguyên
DestinationSearchComponent/ → Giữ nguyên
```

**Migration steps**:
```bash
# 1. Tạo project Vite mới
cd D:\KLTN
npm create vite@latest tourism-frontend-v2 -- --template react-ts
cd tourism-frontend-v2

# 2. Cài đúng dependencies từ client-side/package.json
npm install axios @reduxjs/toolkit react-redux react-router-dom@7
npm install antd @ant-design/icons lucide-react react-icons
npm install @react-oauth/google react-toastify recharts
npm install swiper date-fns @stomp/stompjs sockjs-client
npm install react-quill react-bootstrap bootstrap
npm install react-datepicker react-select react-confetti
npm install react-imask react-markdown dvhcvn
npm install -D sass @types/node
```

- [ ] Copy toàn bộ `src/components/` từ `client-side` vào `tourism-frontend-v2/src/components/`
- [ ] Copy `src/context/`, `src/dto/`, `src/data/`, `src/hook/`, `src/assets/`
- [ ] **Rewrite** `src/utils/axiosCustomize.js` → `src/utils/axiosInstance.ts`
- [ ] **Rewrite** `src/context/AuthContext.jsx` → update login/logout/refresh URLs
- [ ] Copy và update `src/App.tsx` với routes giống hệt `client-side/src/App.tsx`
- [ ] Cấu hình `vite.config.ts` với alias `@` → `src/`

---

### 📅 Day 17 — Frontend: axiosInstance + AuthContext Migration

**`src/utils/axiosInstance.ts`** — Key changes from existing `axiosCustomize.js`:

```typescript
import axios from 'axios';

const BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:8080/api';

const instance = axios.create({ baseURL: BASE_URL });

instance.interceptors.request.use((config) => {
    const token = localStorage.getItem('accessToken');
    if (token) config.headers.Authorization = `Bearer ${token}`;
    return config;
});

instance.interceptors.response.use(
    (res) => res,
    async (error) => {
        const originalRequest = error.config;
        if (error.response?.status === 401 && !originalRequest._retry) {
            originalRequest._retry = true;
            try {
                const refreshToken = localStorage.getItem('refreshToken');
                if (!refreshToken) throw new Error('No refresh token');

                // ✅ KEY CHANGE: was /auth/refresh-token → now /iam/auth/refresh-token
                const response = await axios.post(
                    `${BASE_URL}/iam/auth/refresh-token`,
                    { refreshToken }
                );
                const { accessToken, refreshToken: newRT } = response.data;
                localStorage.setItem('accessToken', accessToken);
                if (newRT) localStorage.setItem('refreshToken', newRT);
                originalRequest.headers.Authorization = `Bearer ${accessToken}`;
                return instance(originalRequest);
            } catch {
                localStorage.removeItem('accessToken');
                localStorage.removeItem('refreshToken');
                localStorage.removeItem('user');
                window.location.href = '/login';
            }
        }
        return Promise.reject(error);
    }
);

export default instance;
```

**`src/context/AuthContext.tsx`** — Key changes from existing `AuthContext.jsx`:
```typescript
// ✅ Login: /auth/login  →  /iam/auth/login
const response = await axios.post('/iam/auth/login', { email, password });

// ✅ Google login: /auth/google/login  →  stays at /auth/google/login (identity-service)
const response = await axios.post('/auth/google/login', { idToken });

// ✅ Logout: /auth/logout  →  /iam/auth/logout
await axios.post('/iam/auth/logout', {}, {
    headers: { Authorization: `Bearer ${localStorage.getItem('accessToken')}` }
});

// ✅ Refresh: /auth/refresh-token  →  /iam/auth/refresh-token
const response = await axios.post('/iam/auth/refresh-token', { refreshToken });
```

- [ ] Rewrite `axiosInstance.ts` với changes trên
- [ ] Rewrite `AuthContext.tsx` với correct IAM endpoints
- [ ] Update `src/services/user/` calls (chỉ đổi baseURL, endpoints giữ nguyên)
- [ ] Browser test: login → token stored → refresh → logout

---

### 📅 Day 18 — Frontend: Tour + Booking Services Update

> Các components đã có sẵn — chỉ cần update API base URL và response shape từ microservices

**`src/services/tours/`** — Update endpoints:
```typescript
// GET /api/tours → tour-catalog-service (unchanged path)
// GET /api/tours/{id}/departures/available → NEW endpoint (thêm /available)
// GET /api/locations → unchanged
export const getFeaturedTours = () => api.get('/tours/featured');
export const getAvailableDepartures = (tourId: number) =>
    api.get(`/tours/${tourId}/departures/available`);
```

**`src/services/booking/`** — Update endpoints:
```typescript
// POST /api/bookings → unchanged (booking-service)
// GET /api/bookings/my → unchanged
// POST /api/bookings/{id}/cancel → unchanged
```

**`src/services/payment/`** — Already correct paths

- [ ] Verify `homPageComponent/HomePage` renders với API response shape từ tour-catalog-service
- [ ] Verify `toursPageComponent/ToursPage` filter + pagination hoạt động
- [ ] Verify `TourDetailComponent/TourDetail` hiển thị departure table
- [ ] Verify `TourBookingComponent/TourBooking` tạo booking (check coupon endpoint: `/promotions/coupons/{code}`)
- [ ] Fix response mapping nếu field names thay đổi
- [ ] Test: Browse → Detail → Book flow hoàn chỉnh

---

### 📅 Day 19 — Frontend: Payment + Notification Services

**`src/services/payment/`** — Verify paths:
```typescript
// POST /api/payments/vnpay/create → unchanged
// GET  /api/payments/vnpay-return → unchanged (public, no auth)
// POST /api/payments/payos/create → unchanged
// POST /api/payments/sepay-webhook → unchanged (public)
```

**Add notifications service** (new — was not in monolith frontend):
```typescript
// src/services/notification/notificationService.ts
import api from '../../utils/axiosInstance';

export const getMyNotifications = () => api.get('/notifications/my');
export const getUnreadCount = () => api.get('/notifications/unread-count');
export const markAsRead = (id: number) => api.put(`/notifications/${id}/read`);
export const markAllAsRead = () => api.put('/notifications/read-all');
```

- [ ] Verify `BookingPaymentComponent/BookingPayment` → payment select → VNPay/PayOS redirect
- [ ] Verify `BookingPaymentComponent/PaymentSuccess` → parse URL params, show confetti
- [ ] Verify `BookingPaymentComponent/PaymentFailed`, `PaymentError`, `PaymentWaiting`
- [ ] Add notification bell to `HeaderComponent` → call `getUnreadCount()` on mount
- [ ] Add notification count badge in header

---

### 📅 Day 20 — Frontend: Reviews + Profile + Admin Updates

**`src/services/review/`** — Update + add endpoints:
```typescript
// POST /api/reviews → unchanged (review-service)
// GET  /api/reviews/tour/{code} → unchanged
// GET  /api/reviews/tour/{code}/summary → NEW (add to service)
export const getReviewSummary = (tourCode: string) =>
    api.get(`/reviews/tour/${tourCode}/summary`);
export const checkEligibility = (bookingCode: string) =>
    api.get(`/reviews/eligibility/${bookingCode}`);
```

**Update `InformationComponent`** (profile + bookings tab):
- Profile tab: `PATCH /api/users/{id}/profile` (unchanged)
- Bookings tab: `GET /api/bookings/my` (unchanged)
- Add Notifications tab: `GET /api/notifications/my`
- Change password: `PATCH /api/users/{id}/change-password` (unchanged)

**Update `AdminComponent` pages**:
- Dashboard API: `GET /api/analytics/dashboard/summary` ← was `/dashboard/*`
- ChatbotWidget: `POST /api/analytics/chatbot/chat` ← was `/chatbot/chat`
- Coupons: `GET /api/admin/coupons` (unchanged)
- Users admin: `GET /api/admin/users` (unchanged)

- [ ] Fix `InformationComponent` tabs to include Notifications
- [ ] Fix AdminComponent dashboard → analytics-service endpoints
- [ ] Fix ChatbotWidget endpoint
- [ ] Verify `CouponManagement` CRUD endpoints

---

### 📅 Day 21 — Frontend: Reviews + Notifications + Profile

- [ ] `ReviewSection` (in TourDetailPage): rating summary + review cards with images
- [ ] `WriteReviewModal`: star picker + text + multi-image upload
- [ ] `NotificationsPage` + unread badge on header
- [ ] `ProfilePage`: view + edit + avatar upload + change password

---

### 📅 Day 22 — Frontend: Admin Dashboard + Tour Management

- [ ] `AdminLayout`: collapsible sidebar + breadcrumbs
- [ ] `DashboardPage`: 4 KPI stat cards + Recharts line+bar charts + top-tours table + AI chatbot widget (floating)
- [ ] `ToursManagePage`: Ant Design Table + create/edit modal + image upload drawer + departure management

---

### 📅 Day 23 — Frontend: Admin Booking + User + Coupon Management

- [ ] `BookingsManagePage`: table + filter + status badge + admin cancel
- [ ] `UsersManagePage`: table + lock/unlock action
- [ ] `CouponsManagePage`: CRUD table

---

### 📅 Day 24 — Frontend: Polish + Responsive

- [ ] Responsive audit: 375px / 768px / 1200px / 1440px
- [ ] Skeleton loaders for all major data-fetching views
- [ ] Error boundaries + custom 404/403/500 pages
- [ ] Toast notifications for all user-facing actions
- [ ] `React.lazy` + `Suspense` for all route pages
- [ ] `<title>` and `<meta name="description">` per page

---

### 📅 Day 25 — E2E Testing: 20-Scenario Checklist

| # | Scenario | Pass |
|---|----------|------|
| 1 | Register new user | ☐ |
| 2 | Verify email via link | ☐ |
| 3 | Login email + password | ☐ |
| 4 | Login Google OAuth | ☐ |
| 5 | Browse tours with filter | ☐ |
| 6 | View tour detail + itinerary | ☐ |
| 7 | Favorite a tour | ☐ |
| 8 | Create booking with coupon | ☐ |
| 9 | Pay with VNPay (sandbox) | ☐ |
| 10 | Pay with PayOS QR code | ☐ |
| 11 | Receive booking confirmation email | ☐ |
| 12 | View My Bookings — status CONFIRMED | ☐ |
| 13 | Cancel a PENDING booking | ☐ |
| 14 | Submit review with images | ☐ |
| 15 | View in-app notifications | ☐ |
| 16 | Update profile + avatar | ☐ |
| 17 | Admin: create/edit/delete tour | ☐ |
| 18 | Admin: view dashboard + charts | ☐ |
| 19 | AI chatbot returns business insight | ☐ |
| 20 | Logout-all → both tokens dead | ☐ |

Fix all blocking issues.

---

### 📅 Day 26 — Dockerize All Services

**Standard `Dockerfile`** (multi-stage, used for all backend services):
```dockerfile
FROM maven:3.9-eclipse-temurin-17-alpine AS builder
WORKDIR /build

# For services that depend on common-events:
COPY shared-libs/common-events /build/common-events
RUN mvn install -f /build/common-events/pom.xml -DskipTests -q

COPY services/{service-name}/pom.xml .
RUN mvn dependency:go-offline -q
COPY services/{service-name}/src ./src
RUN mvn package -DskipTests -q

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /build/target/*.jar app.jar
EXPOSE ${SERVER_PORT:-8080}
HEALTHCHECK --interval=30s --timeout=5s --retries=5 \
    CMD wget -qO- http://localhost:${SERVER_PORT}/actuator/health || exit 1
ENTRYPOINT ["java", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "-jar", "app.jar"]
```

**Frontend `Dockerfile`**:
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
HEALTHCHECK CMD wget -qO- http://localhost/index.html || exit 1
```

**`nginx.conf`** (SPA support):
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;  # React Router SPA
    }

    location /api/ {
        proxy_pass http://api-gateway:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

- [ ] Dockerfile for all 10 backend services
- [ ] Dockerfile for frontend
- [ ] Update `docker-compose.yml` with all services + healthchecks + depends_on
- [ ] `docker-compose up --build -d` → verify 15 containers healthy

---

### 📅 Day 27 — Deploy AWS: EC2 Setup

**AWS Resources needed**:
```
EC2: t3.medium (2 vCPU, 4 GB RAM) — Ubuntu 22.04 LTS
  → Run all 15 Docker containers (sufficient for KLTN load)

Elastic IP: static IP for EC2
Network: default VPC, Security Group:
  Inbound:  22 (SSH, your IP only), 80 (HTTP), 443 (HTTPS)
  Outbound: all (for Docker pulls, external APIs)

Domain: tourism-kltn.yourdomain.com → EC2 Elastic IP
        (use Cloudflare free tier for DNS)
```

**EC2 Setup**:
```bash
# Connect
ssh -i kltn.pem ubuntu@<EC2_IP>

# Install Docker + Docker Compose
sudo apt update && sudo apt install -y docker.io docker-compose-v2 git nginx certbot python3-certbot-nginx
sudo usermod -aG docker ubuntu
newgrp docker

# Clone project
git clone https://github.com/you/tourism-microservices-v2.git
cd tourism-microservices-v2
cp .env.example .env
nano .env    # Fill in all production values

# Update VITE_API_URL to production domain in .env
# Update VNPay returnUrl to production URL
# Update PayOS webhook URL to production URL
```

---

### 📅 Day 28 — Deploy AWS: Run + SSL + Domain

**Deploy sequence**:
```bash
# 1. Infrastructure first
docker compose up -d zookeeper kafka postgres redis zipkin service-registry
sleep 45  # Wait for Eureka to be ready

# 2. Auth layer
docker compose up -d iam-service identity-service api-gateway
sleep 30  # Wait for services to register

# 3. Business services all at once
docker compose up -d \
    tour-catalog-service booking-service payment-service \
    review-service promotion-service notification-service \
    analytics-service cms-service

# 4. Frontend
docker compose up -d frontend

# 5. Verify all registered in Eureka
curl http://localhost:8761/eureka/apps | python3 -m json.tool
```

**Nginx reverse proxy + SSL**:
```bash
# /etc/nginx/sites-available/tourism
server {
    listen 80;
    server_name tourism-kltn.yourdomain.com;

    location /api/ {
        proxy_pass http://localhost:8080/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location / {
        proxy_pass http://localhost:80;
    }
}

sudo certbot --nginx -d tourism-kltn.yourdomain.com
# → HTTPS enabled automatically
```

---

### 📅 Day 29 — Production Smoke Test + Monitoring

**Smoke Tests**:
```bash
BASE=https://tourism-kltn.yourdomain.com

# Public APIs
curl $BASE/api/tours | jq '.content | length'
curl $BASE/api/cms/policies | jq length

# Auth flow
TOKEN=$(curl -s -X POST $BASE/api/iam/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@test.com","password":"Admin123@"}' \
  | jq -r '.accessToken')

# Protected API
curl -H "Authorization: Bearer $TOKEN" $BASE/api/auth/profile | jq .

# Admin API
curl -H "Authorization: Bearer $TOKEN" $BASE/api/analytics/dashboard/summary | jq .

# Eureka
curl http://<EC2_IP>:8761

# Zipkin
curl http://<EC2_IP>:9411/zipkin/
```

**Monitoring setup**:
- [ ] `docker compose logs -f api-gateway` — watch gateway logs
- [ ] Set up Zipkin trace analysis
- [ ] Test VNPay payment in sandbox with real ngrok/domain URL

---

### 📅 Day 30 — Final Documentation + Handoff

- [ ] Update `README.md` with:
  - Architecture overview + diagram
  - Local development setup (step-by-step)
  - Environment variables reference
  - API base URLs per service
  - Deploy guide
- [ ] Export Postman Collection v2.1
- [ ] Create seed data script: admin user + 5 sample tours + 2 coupons
- [ ] Final `docker-compose up -d` verify — 100% healthy
- [ ] **Git tag**: `v1.0-production`
- [ ] Push to GitHub with complete README

---

## 11. Frontend Architecture

### 11.1 Migration Strategy: CRA → Vite

> **Existing frontend**: `D:\KLTN\client-side` (Create React App, `react-scripts 5.0.1`, TypeScript)  
> **Target**: `D:\KLTN\tourism-microservices-v2\frontend` — Vite 5 + same stack  
> **Approach**: **Migrate, not rewrite.** Keep all components, pages, styles. Only change auth endpoints and build tooling.

**What stays the same (copy directly)**:
- All React components (17 component directories)
- All SCSS/CSS styles
- Business logic (tour browsing, booking flow, payment flow)
- Redux state shape
- React Router routes structure

**What changes (after copy)**:
| File | Change |
|------|--------|
| `axiosCustomize.js` → `axiosInstance.ts` | Update refresh URL: `/auth/refresh-token` → `/iam/auth/refresh-token` |
| `AuthContext.jsx` → `AuthContext.tsx` | Update login: `/auth/login` → `/iam/auth/login` |
| `AuthContext.jsx` → `AuthContext.tsx` | Update logout: `/auth/logout` → `/iam/auth/logout` |
| `package.json` | Remove `react-scripts`, add `vite`, `@vitejs/plugin-react` |
| `public/index.html` → `index.html` | Move to root level (Vite convention) |
| `src/services/dashboard/` | Update endpoints to `/api/analytics/dashboard/*` |
| `src/components/ChatbotWidget/` | Update endpoint to `/api/analytics/chatbot/chat` |

### 11.2 Existing Frontend Component Inventory

Based on `D:\KLTN\client-side\src\`:

```
client-side/src/
├── App.tsx                  # Routes — keep exactly as-is
├── context/
│   └── AuthContext.jsx         # → UPDATE: login/logout/refresh URLs
├── utils/
│   ├── axiosCustomize.js       # → REWRITE as axiosInstance.ts
│   └── ScrollToTop.jsx         # keep
├── services/               # Map to Gateway endpoints
│   ├── api.ts                  # re-export of axiosInstance (keep)
│   ├── tours/                  # → tour-catalog-service :8082 (mostly unchanged)
│   ├── booking/                # → booking-service :8083 (unchanged)
│   ├── payment/                # → payment-service :8084 (unchanged)
│   ├── review/                 # → review-service :8085 (add summary endpoint)
│   ├── favoriteTour/           # → tour-catalog-service (unchanged)
│   ├── location/               # → tour-catalog-service (unchanged)
│   ├── user/                   # → identity-service :8081 (unchanged)
│   ├── dashboard/              # → UPDATE: /analytics/dashboard/*
│   └── websocket.js            # keep (Socket.IO/STOMP)
├── components/
│   ├── HeaderComponent/        # keep → add notification badge
│   ├── FooterComponent/        # keep
│   ├── LayoutComponent/        # keep (MainLayout)
│   ├── homPageComponent/       # keep (HomePage)
│   ├── toursPageComponent/     # keep (ToursPage filter+grid)
│   ├── TourDetailComponent/    # keep (TourDetail with gallery, departures)
│   ├── TourBookingComponent/   # keep (passenger form + coupon)
│   ├── BookingPaymentComponent/ # keep (all payment result pages)
│   ├── InformationComponent/   # keep (profile + bookings tabs)
│   ├── Login/                  # → UPDATE endpoint
│   ├── RegisterComponent/      # keep (endpoint same: /auth/register)
│   ├── VerifyEmail/            # keep
│   ├── AdminComponent/         # keep → UPDATE dashboard + chatbot endpoints
│   ├── ChatbotWidget/          # → UPDATE endpoint to /analytics/chatbot/chat
│   ├── DestinationSearchComponent/ # keep
│   ├── Commons/                # keep (shared UI components)
│   ├── AddBannerComponent/     # keep (admin banner management)
│   └── ProtectedRoute.jsx      # keep
├── dto/                     # TypeScript interfaces — keep
├── data/                    # Static data — keep
└── hook/                    # Custom hooks — keep
```

**New files to add** (not in monolith frontend):
```
src/services/notification/
└── notificationService.ts   # GET /notifications/my, /unread-count, PUT /read
```

### 11.3 Existing Routes (App.tsx) — Keep Exactly

```
/                    → HomePage
/tours               → ToursPage
/information         → InformationComponent (profile + bookings)
/information/:tab    → InformationComponent (tab-aware)
/tour-detail         → TourDetail
/tour/:tourCode      → TourDetail
/order-booking       → TourBooking
/payment-booking     → BookingPayment
/verify-email        → VerifyEmail
/payment-success     → PaymentSuccess
/payment-failed      → PaymentFailed
/payment-waiting     → PaymentWaitingPage
/payment-error       → PaymentError
/register            → Register
/login               → Login
/admin/*             → AdminComponent (nested admin routes)
```

### 11.4 Existing Tech Stack (Keep Exact Versions)

| Library | Version | Status |
|---------|---------|--------|
| React | 18.3.1 | ✅ Keep |
| TypeScript | 4.9.5 | ✅ Keep |
| react-router-dom | 7.8.2 | ✅ Keep |
| axios | 1.13.2 | ✅ Keep |
| antd | 5.29.1 | ✅ Keep |
| @ant-design/icons | 6.1.0 | ✅ Keep |
| react-redux | 9.2.0 | ✅ Keep |
| @reduxjs/toolkit | 2.8.2 | ✅ Keep |
| recharts | 3.6.0 | ✅ Keep |
| swiper | 11.2.10 | ✅ Keep |
| @react-oauth/google | 0.12.2 | ✅ Keep |
| react-toastify | 11.0.5 | ✅ Keep |
| react-quill | 2.0.0 | ✅ Keep (admin editor) |
| react-bootstrap | 2.10.10 | ✅ Keep |
| react-confetti | 6.4.0 | ✅ Keep (payment success) |
| dvhcvn | 1.2.x | ✅ Keep (VN address picker) |
| lucide-react | 0.542.0 | ✅ Keep |
| react-scripts | 5.0.1 | ❌ **Remove** |
| vite | 5.x | ✅ **Add** (replaces react-scripts) |
| @vitejs/plugin-react | 4.x | ✅ **Add** |

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
        // Proxy /api to API Gateway during development
        // Eliminates CORS issues for local dev
      }
    }
  },
  define: {
    // CRA uses process.env, Vite uses import.meta.env
    // Replace any process.env.REACT_APP_* with import.meta.env.VITE_*
    'process.env': {}
  }
});
```

### 11.6 axiosInstance.ts (Updated from axiosCustomize.js)

Key differences from existing `axiosCustomize.js`:
1. `BASE_URL` now reads `import.meta.env.VITE_API_URL` (Vite) instead of hardcoded
2. Refresh token URL: `/auth/refresh-token` → `/iam/auth/refresh-token`
3. TypeScript types added
4. Remove debug console.log statements

```typescript
import axios, { InternalAxiosRequestConfig } from 'axios';

const BASE_URL = import.meta.env.VITE_API_URL
    ? `${import.meta.env.VITE_API_URL}/api`
    : 'http://localhost:8080/api';

const instance = axios.create({
    baseURL: BASE_URL,
    timeout: 30000,
});

instance.interceptors.request.use(
    (config: InternalAxiosRequestConfig) => {
        const token = localStorage.getItem('accessToken');
        if (token) {
            config.headers.Authorization = `Bearer ${token}`;
        }
        return config;
    },
    (error) => Promise.reject(error)
);

instance.interceptors.response.use(
    (response) => response,
    async (error) => {
        const originalRequest = error.config;

        if (error.response?.status === 401 && !originalRequest._retry) {
            originalRequest._retry = true;

            try {
                const refreshToken = localStorage.getItem('refreshToken');
                if (!refreshToken) throw new Error('No refresh token available');

                // ✅ Updated: calls IAM service (was /auth/refresh-token)
                const response = await axios.post(
                    `${BASE_URL}/iam/auth/refresh-token`,
                    { refreshToken },
                    { headers: { 'Content-Type': 'application/json' } }
                );

                const { accessToken, refreshToken: newRefreshToken } = response.data;
                localStorage.setItem('accessToken', accessToken);
                if (newRefreshToken) {
                    localStorage.setItem('refreshToken', newRefreshToken);
                }

                originalRequest.headers.Authorization = `Bearer ${accessToken}`;
                return instance(originalRequest);

            } catch (refreshError) {
                localStorage.removeItem('accessToken');
                localStorage.removeItem('refreshToken');
                localStorage.removeItem('user');
                window.location.href = '/login';
                return Promise.reject(refreshError);
            }
        }

        return Promise.reject(error);
    }
);

export default instance;
```

### 11.7 Frontend `.env` File

```bash
# .env (for local development)
VITE_API_URL=http://localhost:8080
VITE_GOOGLE_CLIENT_ID=your-google-client-id

# .env.production (for Docker build)
VITE_API_URL=https://tourism-kltn.yourdomain.com
VITE_GOOGLE_CLIENT_ID=your-google-client-id
```

### 11.2 Directory Structure

```
tourism-frontend-v2/src/
├── api/
│   ├── axiosInstance.ts     # Base axios + JWT interceptor + auto-refresh
│   ├── iam.api.ts           # → /api/iam/auth/**
│   ├── auth.api.ts          # → /api/auth/** (identity-service)
│   ├── tours.api.ts         # → /api/tours/**
│   ├── bookings.api.ts      # → /api/bookings/**
│   ├── payments.api.ts      # → /api/payments/**
│   ├── reviews.api.ts       # → /api/reviews/**
│   ├── promotions.api.ts    # → /api/promotions/**
│   ├── notifications.api.ts # → /api/notifications/**
│   ├── analytics.api.ts     # → /api/analytics/**
│   └── cms.api.ts           # → /api/cms/**
│
├── store/
│   ├── index.ts
│   ├── auth/authSlice.ts    # { user, accessToken, isAuthenticated }
│   └── ui/uiSlice.ts        # { loading, modals }
│
├── pages/
│   ├── public/
│   │   ├── HomePage.tsx
│   │   ├── ToursPage.tsx
│   │   └── TourDetailPage.tsx
│   ├── auth/
│   │   ├── LoginPage.tsx
│   │   ├── RegisterPage.tsx
│   │   └── VerifyEmailPage.tsx
│   ├── user/
│   │   ├── BookingsPage.tsx
│   │   ├── ProfilePage.tsx
│   │   └── NotificationsPage.tsx
│   ├── booking/
│   │   ├── BookingFormPage.tsx
│   │   ├── PaymentPage.tsx
│   │   └── PaymentResultPage.tsx
│   └── admin/
│       ├── DashboardPage.tsx
│       ├── ToursManagePage.tsx
│       ├── BookingsManagePage.tsx
│       ├── CouponsManagePage.tsx
│       └── UsersManagePage.tsx
│
├── components/
│   ├── layout/
│   │   ├── PublicLayout.tsx    # Header (nav, auth, notifications) + Footer
│   │   └── AdminLayout.tsx     # Sidebar + Header
│   ├── common/
│   │   ├── ProtectedRoute.tsx  # Check isAuthenticated
│   │   └── AdminRoute.tsx      # Check role === 'ADMIN'
│   ├── tour/
│   │   ├── TourCard.tsx
│   │   ├── TourGallery.tsx     # Swiper lightbox
│   │   └── DepartureTable.tsx
│   ├── booking/
│   │   ├── PassengerForm.tsx
│   │   └── BookingCard.tsx
│   ├── review/
│   │   ├── ReviewList.tsx
│   │   ├── RatingSummary.tsx
│   │   └── WriteReviewModal.tsx
│   └── admin/
│       └── ChatbotWidget.tsx   # Floating chatbot
│
└── types/                      # All TypeScript interfaces
    ├── auth.types.ts
    ├── tour.types.ts
    ├── booking.types.ts
    └── ...
```

### 11.3 axiosInstance.ts

```typescript
import axios from 'axios';
import { store } from '../store';
import { logout, setAccessToken } from '../store/auth/authSlice';

const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:8080',
  timeout: 15000,
});

// Attach Bearer token
api.interceptors.request.use((config) => {
  const token = store.getState().auth.accessToken;
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Auto-refresh on 401
let isRefreshing = false;
let failedQueue: Array<{ resolve: Function; reject: Function }> = [];

api.interceptors.response.use(
  (res) => res,
  async (error) => {
    if (error.response?.status === 401 && !error.config._retry) {
      if (isRefreshing) {
        return new Promise((resolve, reject) =>
          failedQueue.push({ resolve, reject })
        ).then((token) => {
          error.config.headers.Authorization = `Bearer ${token}`;
          return api(error.config);
        });
      }
      error.config._retry = true;
      isRefreshing = true;
      try {
        const refreshToken = localStorage.getItem('refreshToken');
        const { data } = await axios.post(
          `${api.defaults.baseURL}/api/iam/auth/refresh-token`,
          { refreshToken }
        );
        store.dispatch(setAccessToken(data.accessToken));
        localStorage.setItem('refreshToken', data.refreshToken);
        failedQueue.forEach(({ resolve }) => resolve(data.accessToken));
        return api(error.config);
      } catch {
        store.dispatch(logout());
        localStorage.removeItem('refreshToken');
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

---

## 12. Deployment Plan

### 12.1 AWS Architecture

```
Internet
    │
    ▼
Cloudflare (DNS + DDoS protection — Free)
    │  tourism-kltn.yourdomain.com → EC2 Elastic IP
    ▼
Nginx on EC2 (reverse proxy + SSL via Let's Encrypt)
    ├── /api/* → localhost:8080 (API Gateway container)
    └── /*     → localhost:80  (Frontend Nginx container)
         │
    EC2 t3.medium (Ubuntu 22.04)
    Docker Compose: 15 containers
         │
    ┌────┴────────────────────────────────┐
    │  PostgreSQL (single instance)        │
    │  Redis                               │
    │  Kafka + Zookeeper                   │
    │  Zipkin                              │
    │  Eureka (service-registry)           │
    │  api-gateway                         │
    │  iam-service                         │
    │  identity-service                    │
    │  tour-catalog-service                │
    │  booking-service                     │
    │  payment-service                     │
    │  review-service                      │
    │  promotion-service                   │
    │  notification-service                │
    │  analytics-service                   │
    │  cms-service                         │
    │  frontend (Nginx)                    │
    └─────────────────────────────────────┘
```

### 12.2 Cost Estimate (1 month, KLTN demo)

| Resource | Config | Cost/Month |
|----------|--------|-----------|
| EC2 t3.medium | On-Demand | ~$30 |
| Elastic IP | 1 address | ~$4 |
| EBS (30 GB gp3) | Storage | ~$2 |
| CloudFront/Cloudflare | Free tier | $0 |
| Let's Encrypt SSL | Free | $0 |
| **Total** | | **~$36/month** |

> 💡 Use `t3.small` (~$15/month) if memory is not an issue — set JVM flags to limit Spring Boot memory usage per service.

### 12.3 JVM Optimization for Multi-Service EC2

```dockerfile
# In each service Dockerfile ENTRYPOINT:
ENTRYPOINT ["java",
    "-XX:+UseContainerSupport",
    "-XX:MaxRAMPercentage=60.0",    # Each service uses max 60% of its container limit
    "-XX:InitialRAMPercentage=25.0",
    "-Xss512k",                      # Reduce stack size
    "-Djava.security.egd=file:/dev/./urandom",
    "-jar", "app.jar"]
```

```yaml
# docker-compose.yml — memory limits per service
deploy:
  resources:
    limits:
      memory: 512m    # Most services
    # iam-service, identity-service: 256m
    # analytics-service: 768m (Gemini API calls)
```

---

## 13. Risk Register

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|------------|
| IAM Service becomes bottleneck | Medium | High | Caffeine cache in Gateway (60s TTL) reduces IAM calls by ~90% |
| Feign circular dependency | Medium | High | Use `@Lazy` injection; convert one direction to Kafka |
| Kafka consumer deserialization error | Medium | Medium | Add `spring.json.trusted.packages`, use `auto-offset-reset=earliest` |
| VNPay/PayOS webhook unreachable locally | High | High | Use `ngrok http 8080` → update sandbox webhook URLs |
| Slot race condition on booking | Medium | High | `@Transactional` + `UPDATE slots WHERE id=? AND slots >= ?` pessimistic lock |
| Redis connection fails | Low | Medium | Use Docker service name `redis:6379`, not `localhost` |
| JWT_SECRET distributed to wrong service | Low | Critical | Only `iam-service` and `api-gateway` have JWT_SECRET in `.env` |
| EC2 OOM (15 containers) | Medium | High | Set `deploy.resources.limits.memory` per container; use t3.medium min |
| CORS rejection FE → Gateway | Medium | High | `allowed-origins: http://localhost:5173` in gateway CORS config |
| Logout doesn't work within 60s (cache) | Low | Low | Acceptable for KLTN; production would use shorter TTL or event-driven cache invalidation |

---

## 14. Completion Checklist

### Infrastructure

- [ ] `docker-compose.yml` — 15 containers with healthchecks
- [ ] `init-db.sql` — 10 database created
- [ ] API Gateway — JWT filter + IAM call + Caffeine cache + CORS
- [ ] Eureka — all 10 services registered
- [ ] Zipkin — distributed traces visible
- [ ] Kafka — 6 topics flowing with no consumer lag
- [ ] Redis — blacklist + booking cache working

### Backend Services

| Service | DB | APIs | Feign | Kafka | Tests | Docker |
|---------|-----|------|-------|-------|-------|--------|
| iam-service | ☐ | ☐ | ☐ | — | ☐ | ☐ |
| identity-service | ☐ | ☐ | ☐ | Prod | ☐ | ☐ |
| tour-catalog-service | ☐ | ☐ | — | — | ☐ | ☐ |
| booking-service | ☐ | ☐ | ☐ | Prod | ☐ | ☐ |
| payment-service | ☐ | ☐ | ☐ | Prod | ☐ | ☐ |
| review-service | ☐ | ☐ | ☐ | Prod | ☐ | ☐ |
| promotion-service | ☐ | ☐ | — | — | ☐ | ☐ |
| notification-service | ☐ | ☐ | — | Cons | ☐ | ☐ |
| analytics-service | ☐ | ☐ | — | Cons | ☐ | ☐ |
| cms-service | ☐ | ☐ | — | — | ☐ | ☐ |

### Frontend

- [ ] Auth (Login + Register + Email Verify + Google OAuth)
- [ ] Token management (IAM refresh, memory storage, auto-retry)
- [ ] Public pages (Home + Tours + Tour Detail + Locations)
- [ ] User pages (My Bookings + Profile + Notifications)
- [ ] Booking flow (Form → Payment → Result)
- [ ] Review (read/write + images)
- [ ] Admin Dashboard (stats + charts + chatbot)
- [ ] Admin Management (Tour CRUD + Booking + User + Coupon)
- [ ] Responsive (375px → 1440px)
- [ ] Error boundaries + skeleton loading
- [ ] E2E 20-scenario checklist PASS

### Deployment

- [ ] All Dockerfiles built successfully
- [ ] EC2 instance running (t3.medium)
- [ ] Domain configured (Cloudflare)
- [ ] SSL certificate (Let's Encrypt via Certbot)
- [ ] Production environment variables set
- [ ] VNPay/PayOS webhook URLs updated to production domain
- [ ] Production smoke tests PASS (4 curl tests)
- [ ] Git tag `v1.0-production` pushed

---

> **Tech Stack Summary**:
> Java 17 · Spring Boot 3.3 · Spring Cloud 2023.0.2 · PostgreSQL 15 · Redis 7 · Apache Kafka 3.5 · React 18 · Vite 5 · TypeScript · Ant Design 5 · Docker · AWS EC2
>
> **Project**: `D:\KLTN\tourism-microservices-v2`  
> **Source analysis**: `D:\KLTN\Tourism_Backend` (22 controllers, 23 entities)  
> **Reference docs**: `D:\KLTN\Tourism_Backend\TLCN_FinalReport.docx`
