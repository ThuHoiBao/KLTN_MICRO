
## 1. Cấu Trúc Project

```
tourism-microservices-v2/
├── pom.xml                          ← Parent POM (dependency management)
├── docker-compose.yml
├── .env                             ← tất cả secrets
├── api-gateway/
│   ├── pom.xml
│   └── src/main/
│       ├── resources/application.yml
│       └── java/.../gateway/
│           ├── GatewayApplication.java
│           ├── config/
│           │   ├── GatewayConfig.java        ← route definitions
│           │   └── SecurityConfig.java       ← JWT filter
│           └── filter/
│               └── JwtAuthFilter.java
├── service-registry/
│   ├── pom.xml
│   └── src/main/
│       └── resources/application.yml
├── iam-service/
│   ├── pom.xml
│   └── src/main/java/.../iam/
│       ├── entity/RefreshToken.java
│       ├── repository/RefreshTokenRepository.java
│       ├── service/AuthService(Impl).java
│       ├── service/GoogleAuthService(Impl).java
│       ├── service/EmailService(Impl).java
│       ├── controller/AuthController.java
│       ├── controller/AdminAuthController.java
│       ├── feign/UserClient.java              ← gọi user-service
│       ├── security/JwtProvider.java
│       └── dto/...
├── user-service/
│   ├── pom.xml
│   └── src/main/java/.../user/
│       ├── entity/{User, FavoriteTour, Notification, UserNotification}.java
│       ├── repository/...
│       ├── service/{UserService, FavoriteTourService, NotificationService}(Impl).java
│       ├── controller/{UserController, AdminProfileController}.java
│       ├── controller/{FavoriteTourController, NotificationController}.java
│       ├── controller/internal/UserInternalController.java   ← cho Feign
│       ├── feign/TourClient.java              ← gọi tour-service
│       └── dto/...
├── tour-service/
│   ├── pom.xml
│   └── src/main/java/.../tour/
│       ├── entity/{Tour, TourImage, TourMedia, ItineraryDay, Location,
│       │          TourDeparture, DeparturePricing, DepartureTransport,
│       │          PolicyTemplate, BranchContact, Coupon, Review, ImageReview}.java
│       ├── repository/...
│       ├── service/...
│       ├── service/chatbot/{ChatbotService, VectorService, VectorSyncService}.java
│       ├── controller/{TourController, TourManagementController, ...}.java
│       ├── controller/ChatbotController.java
│       ├── controller/internal/TourInternalController.java   ← cho Feign
│       └── dto/...
├── booking-service/
│   ├── pom.xml
│   └── src/main/java/.../booking/
│       ├── entity/{Booking, BookingPassenger, RefundInformation}.java
│       ├── repository/...
│       ├── service/{BookingService, BookingCleanupService, DashboardService}(Impl).java
│       ├── controller/{BookingController, DashboardController}.java
│       ├── controller/internal/BookingInternalController.java ← cho Feign
│       ├── feign/{TourClient, UserClient, PaymentClient}.java
│       └── dto/...
└── payment-service/
    ├── pom.xml
    └── src/main/java/.../payment/
        ├── entity/Payment.java
        ├── repository/PaymentRepository.java
        ├── service/{PaymentService, PayOSService, SepayService}(Impl).java
        ├── controller/PaymentController.java
        ├── controller/internal/PaymentInternalController.java ← cho Feign
        ├── feign/BookingClient.java
        └── dto/...
```

---

## 2. Parent POM

```xml
<!-- pom.xml (root) -->
<groupId>com.tourism</groupId>
<artifactId>tourism-microservices</artifactId>
<version>1.0.0</version>
<packaging>pom</packaging>

<modules>
  <module>service-registry</module>
  <module>api-gateway</module>
  <module>iam-service</module>
  <module>user-service</module>
  <module>tour-service</module>
  <module>booking-service</module>
  <module>payment-service</module>
</modules>

<properties>
  <java.version>17</java.version>
  <spring-boot.version>3.3.0</spring-boot.version>
  <spring-cloud.version>2023.0.2</spring-cloud.version>
</properties>

<dependencyManagement>
  <dependencies>
    <!-- Spring Boot BOM -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-dependencies</artifactId>
      <version>${spring-boot.version}</version>
      <type>pom</type><scope>import</scope>
    </dependency>
    <!-- Spring Cloud BOM -->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>${spring-cloud.version}</version>
      <type>pom</type><scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

---

## 3. Entity Chi Tiết Từng Service

### 3.1 iam-service — DB: `tourism_iam`

```java
// RefreshToken.java
@Entity @Table(name = "refresh_tokens")
public class RefreshToken extends BaseEntity {
    @Id @GeneratedValue
    private Long id;

    @Column(unique = true, nullable = false, length = 512)
    private String token;

    // KHÔNG dùng @ManyToOne User — cross-service
    @Column(nullable = false)
    private Integer userId;     // ref → user-service

    @Column(nullable = false)
    private Instant expiryDate;

    private boolean revoked = false;
}
```

**application.yml (iam-service):**
```yaml
server:
  port: 8081
spring:
  application:
    name: iam-service
  datasource:
    url: jdbc:postgresql://localhost:5432/tourism_iam
    username: ${DB_USER}
    password: ${DB_PASS}
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/

app:
  jwt:
    secret: ${JWT_SECRET}
    expiration: 3600000        # 1 hour
    refresh-expiration: 604800000  # 7 days

google:
  client-id: ${GOOGLE_CLIENT_ID}
  client-secret: ${GOOGLE_CLIENT_SECRET}
  redirect-uri: ${GOOGLE_REDIRECT_URI}

feign:
  user-service-url: http://user-service
```

---

### 3.2 user-service — DB: `tourism_user`

```java
// User.java
@Entity @Table(name = "users")
public class User extends BaseEntity {
    @Id @GeneratedValue
    private Integer userID;

    @Column(unique = true, nullable = false)
    private String email;

    private String password;     // BCrypt (null nếu OAuth2)
    private String fullName;
    private String phone;
    private String avatar;       // Cloudinary URL
    private String provinceName;
    private String districtName;

    @Enumerated(EnumType.STRING)
    private UserRole role;       // ADMIN | CUSTOMER

    private Boolean status = true;
    private Boolean isEmailVerified = false;
    private String verificationToken;

    @Column(precision = 19, scale = 4)
    private BigDecimal coinBalance = BigDecimal.ZERO;

    private LocalDate dateOfBirth;
    private Instant lastActiveAt;

    // KHÔNG có: List<Booking>, List<Review> — cross-service
    // KHÔNG có: List<FavoriteTour> (query ngược lại từ FavoriteTour entity)
}

// FavoriteTour.java
@Entity @Table(name = "favorite_tours",
    uniqueConstraints = @UniqueConstraint(columnNames = {"user_id", "tour_id"}))
public class FavoriteTour extends BaseEntity {
    @Id @GeneratedValue
    private Integer favoriteID;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    // KHÔNG dùng @ManyToOne Tour — cross-service
    @Column(nullable = false)
    private Integer tourId;     // ref → tour-service

    private Instant createdAt;
}

// Notification.java (giữ nguyên từ monolith, chỉ đổi package)
@Entity @Table(name = "notifications")
public class Notification extends BaseEntity {
    @Id @GeneratedValue
    private Integer notificationID;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    private String type;    // BOOKING_CREATED, PAYMENT_SUCCESS, BOOKING_CANCELLED, ...
    private String title;

    @Column(columnDefinition = "TEXT")
    private String message;

    @JdbcTypeCode(SqlTypes.JSON)
    private JsonNode metadata;  // {bookingId, tourName, ...}

    private Boolean isRead = false;
    private Instant readAt;
}
```

---

### 3.3 tour-service — DB: `tourism_tour`

```java
// Tour.java (giữ nguyên, chỉ xóa bookings list)
@Entity @Table(name = "tours")
public class Tour extends BaseEntity {
    @Id @GeneratedValue
    private Integer tourID;
    private String tourCode;    // UNIQUE
    private String tourName;
    private String duration;    // "3N2Đ"
    private String attractions;
    private String meals;

    @ManyToOne @JoinColumn(name = "start_location_id")
    private Location startLocation;

    @ManyToOne @JoinColumn(name = "end_location_id")
    private Location endLocation;

    private String idealTime;
    private String tripTransportation;
    private String suitableCustomer;
    private String hotel;
    private Boolean status = true;
    private BigDecimal minPrice;  // denormalized để search nhanh

    @OneToMany(mappedBy = "tour", cascade = CascadeType.ALL)
    private List<TourImage> images;

    @OneToMany(mappedBy = "tour", cascade = CascadeType.ALL)
    private List<TourMedia> mediaList;

    @OneToMany(mappedBy = "tour", cascade = CascadeType.ALL)
    private List<ItineraryDay> itinerary;

    @OneToMany(mappedBy = "tour", cascade = CascadeType.ALL)
    private List<TourDeparture> departures;

    // KHÔNG có: List<Review>, List<Booking>
}

// TourDeparture.java
@Entity @Table(name = "tour_departures")
public class TourDeparture extends BaseEntity {
    @Id @GeneratedValue
    private Integer departureID;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "tour_id", nullable = false)
    private Tour tour;

    private LocalDate departureDate;
    private LocalDate returnDate;
    private Integer availableSlots;
    private Integer bookedSlots = 0;  // denormalized, cập nhật khi booking confirmed

    @Column(columnDefinition = "TEXT")
    private String tourGuideInfo;

    @Enumerated(EnumType.STRING)
    private DepartureStatus status;   // OPEN | CLOSED | CANCELLED

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "policy_template_id")
    private PolicyTemplate policyTemplate;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "coupon_id")
    private Coupon coupon;  // coupon gắn với departure cụ thể (nullable)

    @OneToMany(mappedBy = "tourDeparture", cascade = CascadeType.ALL)
    private List<DeparturePricing> pricings;

    @OneToMany(mappedBy = "tourDeparture", cascade = CascadeType.ALL)
    private List<DepartureTransport> transports;

    // KHÔNG có: List<Booking> — cross-service
}

// Review.java
@Entity @Table(name = "reviews",
    uniqueConstraints = @UniqueConstraint(columnNames = "booking_id"))
public class Review extends BaseEntity {
    @Id @GeneratedValue
    private Integer reviewID;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "tour_id", nullable = false)
    private Tour tour;

    // KHÔNG dùng @ManyToOne User, Booking — cross-service
    @Column(nullable = false)
    private Integer userId;      // ref → user-service

    @Column(nullable = false, unique = true)
    private Integer bookingId;   // ref → booking-service (1 booking 1 review)

    @Column(nullable = false)
    private Integer rating;      // 1-5

    @Column(columnDefinition = "TEXT")
    private String comment;

    private Boolean isVisible = true;

    @OneToMany(mappedBy = "review", cascade = CascadeType.ALL)
    private List<ImageReview> images;
}
```

---

### 3.4 booking-service — DB: `tourism_booking`

```java
// Booking.java
@Entity @Table(name = "bookings")
public class Booking extends BaseEntity {
    @Id @GeneratedValue
    private Integer bookingID;

    @Column(unique = true, nullable = false)
    private String bookingCode;   // BK-20240415-XXXX

    // KHÔNG dùng @ManyToOne User, TourDeparture — cross-service
    @Column(nullable = false)
    private Integer userId;         // ref → user-service

    @Column(nullable = false)
    private Integer departureId;    // ref → tour-service

    // Snapshot data tại thời điểm đặt (tránh phụ thuộc vào service khác sau này)
    private String tourNameSnapshot;
    private String departureDateSnapshot;

    private Instant bookingDate;
    private String contactEmail;
    private String contactFullName;
    private String contactPhone;
    private String contactAddress;

    @Column(columnDefinition = "TEXT")
    private String customerNote;

    private Integer totalPassengers;

    @Column(precision = 19, scale = 4)
    private BigDecimal subtotalPrice;

    @Column(precision = 19, scale = 4)
    private BigDecimal surcharge = BigDecimal.ZERO;

    @Column(precision = 19, scale = 4)
    private BigDecimal couponDiscount = BigDecimal.ZERO;

    @Column(precision = 19, scale = 4)
    private BigDecimal paidByCoin = BigDecimal.ZERO;

    @Column(precision = 19, scale = 4)
    private BigDecimal totalPrice;

    @Enumerated(EnumType.STRING)
    private BookingStatus bookingStatus;

    private String appliedCouponCode;

    @Column(columnDefinition = "TEXT")
    private String cancelReason;

    @Column(precision = 19, scale = 4)
    private BigDecimal refundAmount = BigDecimal.ZERO;

    // cross-service ref (không FK)
    private Integer paymentId;  // ref → payment-service

    @OneToMany(mappedBy = "booking", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<BookingPassenger> passengers;

    @OneToOne(mappedBy = "booking", cascade = CascadeType.ALL)
    private RefundInformation refundInformation;
}

// BookingPassenger.java (giữ nguyên từ monolith)
@Entity @Table(name = "booking_passengers")
public class BookingPassenger extends BaseEntity {
    @Id @GeneratedValue
    private Integer bookingPassengerID;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "booking_id", nullable = false)
    private Booking booking;

    private String fullName;
    private String gender;
    private LocalDate dateOfBirth;

    @Enumerated(EnumType.STRING)
    private PassengerType passengerType;

    @Column(precision = 19, scale = 4)
    private BigDecimal basePrice;   // giá snapshot tại lúc đặt

    private Boolean requiresSingleRoom = false;

    @Column(precision = 19, scale = 4)
    private BigDecimal singleRoomSurcharge = BigDecimal.ZERO;
}
```

---

### 3.5 payment-service — DB: `tourism_payment`

```java
// Payment.java
@Entity @Table(name = "payments")
public class Payment extends BaseEntity {
    @Id @GeneratedValue
    private Integer paymentID;

    // KHÔNG dùng @ManyToOne Booking — cross-service
    @Column(nullable = false, unique = true)
    private Integer bookingId;   // ref → booking-service (1 booking 1 payment)

    @Enumerated(EnumType.STRING)
    private PaymentMethod paymentMethod;   // VNPAY | PAYOS | SEPAY | COIN

    @Column(precision = 19, scale = 4)
    private BigDecimal amount;

    private String transactionId;       // mã giao dịch từ cổng TT
    private String bankTransactionNo;
    private String bankCode;            // VCB, TCB, ...

    @Enumerated(EnumType.STRING)
    private PaymentStatus status;       // PENDING | SUCCESS | FAILED | REFUNDED

    private Instant paymentDate;
    private Instant timeLimit;          // hạn thanh toán (15 phút)

    // VNPay specific
    private String vnpayOrderInfo;
    private String vnpayPayUrl;

    // PayOS specific
    private Long payosOrderCode;
    private String payosCheckoutUrl;

    // Sepay: polling match
    private String sepayContent;       // nội dung chuyển khoản để match
}
```

---

## 4. API Gateway — Routes

```yaml
# api-gateway/src/main/resources/application.yml
server:
  port: 8080
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        # iam-service
        - id: iam-auth
          uri: lb://iam-service
          predicates:
            - Path=/api/auth/**, /api/admin/auth/**

        # user-service
        - id: user-profile
          uri: lb://user-service
          predicates:
            - Path=/api/users/**, /api/admin/users/**,
                   /api/favorites/**, /api/notifications/**

        # tour-service
        - id: tour-public
          uri: lb://tour-service
          predicates:
            - Path=/api/tours/**, /api/locations/**,
                   /api/reviews/**, /api/chatbot/**,
                   /api/admin/tours/**, /api/admin/departures/**,
                   /api/admin/locations/**, /api/admin/coupons/**,
                   /api/cms/**

        # booking-service
        - id: booking
          uri: lb://booking-service
          predicates:
            - Path=/api/bookings/**, /api/admin/dashboard/**

        # payment-service
        - id: payment
          uri: lb://payment-service
          predicates:
            - Path=/api/payments/**

      # Global CORS
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "http://localhost:5173"
              - "${FRONTEND_URL}"
            allowedMethods: ["GET","POST","PUT","DELETE","OPTIONS","PATCH"]
            allowedHeaders: ["*"]
            allowCredentials: true

  # JWT verification tại Gateway
  security:
    jwt:
      secret: ${JWT_SECRET}

# Public paths (không cần JWT)
app:
  security:
    public-paths:
      - /api/auth/login
      - /api/auth/register
      - /api/auth/refresh
      - /api/auth/google/**
      - /api/auth/verify-email/**
      - /api/tours        # browse tours without login
      - /api/tours/**
      - /api/locations/**
      - /api/payments/vnpay/callback   # VNPay redirect
      - /api/payments/payos/webhook    # PayOS webhook
      - /api/payments/sepay/webhook    # Sepay webhook
```

---

## 5. Internal Feign Endpoints

Các endpoint `/internal/**` chỉ gọi nội bộ (không qua Gateway, không JWT user).  
Gateway config exclude `/internal/**` khỏi routing ra ngoài.

### user-service — `UserInternalController`
```
POST   /internal/users              ← iam-service: tạo user mới khi register
GET    /internal/users/email/{email} ← iam-service: tìm user theo email (login)
GET    /internal/users/{id}         ← booking/tour: lấy thông tin user
PUT    /internal/users/{id}/coin    ← booking-service: cộng/trừ coin
POST   /internal/notifications      ← booking-service: push notification
```

### tour-service — `TourInternalController`
```
GET    /internal/tours/{id}                    ← user-service: tên tour cho favorite
GET    /internal/departures/{id}               ← booking-service: info departure
PUT    /internal/departures/{id}/slots         ← booking-service: giảm availableSlots
GET    /internal/coupons/validate?code=&amount= ← booking-service: validate coupon
GET    /internal/tours/{id}/avg-rating         ← booking-service: dashboard
```

### booking-service — `BookingInternalController`
```
GET    /internal/bookings/{id}                 ← payment-service: lấy booking info
PUT    /internal/bookings/{id}/confirm-paid    ← payment-service: sau thanh toán thành công
GET    /internal/bookings/stats?from=&to=      ← (internal dashboard aggregation)
```

### payment-service — `PaymentInternalController`
```
POST   /internal/payments                      ← booking-service: tạo payment entry
GET    /internal/payments/booking/{bookingId}  ← booking-service: lấy payment status
```

---

## 6. Docker Compose

```yaml
# docker-compose.yml
version: '3.9'
services:

  postgres:
    image: postgres:16-alpine
    container_name: tourism-postgres
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASS}
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init-db.sql
    networks: [tourism-net]

  service-registry:
    build: ./service-registry
    container_name: tourism-registry
    ports:
      - "8761:8761"
    environment:
      SPRING_PROFILES_ACTIVE: docker
    networks: [tourism-net]

  api-gateway:
    build: ./api-gateway
    container_name: tourism-gateway
    ports:
      - "8080:8080"
    environment:
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://service-registry:8761/eureka/
      JWT_SECRET: ${JWT_SECRET}
      FRONTEND_URL: ${FRONTEND_URL}
    depends_on: [service-registry]
    networks: [tourism-net]

  iam-service:
    build: ./iam-service
    container_name: tourism-iam
    ports:
      - "8081:8081"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/tourism_iam
      DB_USER: ${DB_USER}
      DB_PASS: ${DB_PASS}
      JWT_SECRET: ${JWT_SECRET}
      GOOGLE_CLIENT_ID: ${GOOGLE_CLIENT_ID}
      GOOGLE_CLIENT_SECRET: ${GOOGLE_CLIENT_SECRET}
      GOOGLE_REDIRECT_URI: ${GOOGLE_REDIRECT_URI}
      MAIL_HOST: ${MAIL_HOST}
      MAIL_USERNAME: ${MAIL_USERNAME}
      MAIL_PASSWORD: ${MAIL_PASSWORD}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://service-registry:8761/eureka/
    depends_on: [postgres, service-registry]
    networks: [tourism-net]

  user-service:
    build: ./user-service
    container_name: tourism-user
    ports:
      - "8085:8085"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/tourism_user
      DB_USER: ${DB_USER}
      DB_PASS: ${DB_PASS}
      CLOUDINARY_CLOUD_NAME: ${CLOUDINARY_CLOUD_NAME}
      CLOUDINARY_API_KEY: ${CLOUDINARY_API_KEY}
      CLOUDINARY_API_SECRET: ${CLOUDINARY_API_SECRET}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://service-registry:8761/eureka/
    depends_on: [postgres, service-registry]
    networks: [tourism-net]

  tour-service:
    build: ./tour-service
    container_name: tourism-tour
    ports:
      - "8082:8082"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/tourism_tour
      DB_USER: ${DB_USER}
      DB_PASS: ${DB_PASS}
      CLOUDINARY_CLOUD_NAME: ${CLOUDINARY_CLOUD_NAME}
      CLOUDINARY_API_KEY: ${CLOUDINARY_API_KEY}
      CLOUDINARY_API_SECRET: ${CLOUDINARY_API_SECRET}
      GEMINI_API_KEY: ${GEMINI_API_KEY}
      PINECONE_API_KEY: ${PINECONE_API_KEY}
      PINECONE_INDEX_NAME: ${PINECONE_INDEX_NAME}
      PINECONE_BASE_URL: ${PINECONE_BASE_URL}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://service-registry:8761/eureka/
    depends_on: [postgres, service-registry]
    networks: [tourism-net]

  booking-service:
    build: ./booking-service
    container_name: tourism-booking
    ports:
      - "8083:8083"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/tourism_booking
      DB_USER: ${DB_USER}
      DB_PASS: ${DB_PASS}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://service-registry:8761/eureka/
    depends_on: [postgres, service-registry]
    networks: [tourism-net]

  payment-service:
    build: ./payment-service
    container_name: tourism-payment
    ports:
      - "8084:8084"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/tourism_payment
      DB_USER: ${DB_USER}
      DB_PASS: ${DB_PASS}
      VNPAY_TMN_CODE: ${VNPAY_TMN_CODE}
      VNPAY_HASH_SECRET: ${VNPAY_HASH_SECRET}
      VNPAY_URL: ${VNPAY_URL}
      PAYOS_CLIENT_ID: ${PAYOS_CLIENT_ID}
      PAYOS_API_KEY: ${PAYOS_API_KEY}
      PAYOS_CHECKSUM_KEY: ${PAYOS_CHECKSUM_KEY}
      SEPAY_API_TOKEN: ${SEPAY_API_TOKEN}
      SEPAY_BANK_ACCOUNT: ${SEPAY_BANK_ACCOUNT}
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://service-registry:8761/eureka/
    depends_on: [postgres, service-registry]
    networks: [tourism-net]

volumes:
  pgdata:
networks:
  tourism-net:
    driver: bridge
```

---

## 7. Init DB Script

```sql
-- init-db.sql
CREATE DATABASE tourism_iam;
CREATE DATABASE tourism_user;
CREATE DATABASE tourism_tour;
CREATE DATABASE tourism_booking;
CREATE DATABASE tourism_payment;
```

---

## 8. .env Template

```dotenv
# Database
DB_USER=tourism_user
DB_PASS=tourism_pass123

# JWT
JWT_SECRET=your-very-long-jwt-secret-at-least-256-bits

# Google OAuth2
GOOGLE_CLIENT_ID=xxx.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=xxx
GOOGLE_REDIRECT_URI=http://localhost:8080/api/auth/google/callback

# Mail
MAIL_HOST=smtp.gmail.com
MAIL_USERNAME=your@gmail.com
MAIL_PASSWORD=your-app-password

# Cloudinary
CLOUDINARY_CLOUD_NAME=xxx
CLOUDINARY_API_KEY=xxx
CLOUDINARY_API_SECRET=xxx

# Gemini AI
GEMINI_API_KEY=xxx
GEMINI_EMBEDDING_MODEL=models/text-embedding-004
GEMINI_CHAT_MODEL=gemini-2.0-flash

# Pinecone
PINECONE_API_KEY=xxx
PINECONE_INDEX_NAME=tourism-tours
PINECONE_BASE_URL=https://xxx.svc.pinecone.io

# VNPay
VNPAY_TMN_CODE=xxx
VNPAY_HASH_SECRET=xxx
VNPAY_URL=https://sandbox.vnpayment.vn/paymentv2/vpcpay.html
VNPAY_RETURN_URL=http://localhost:8080/api/payments/vnpay/callback

# PayOS
PAYOS_CLIENT_ID=xxx
PAYOS_API_KEY=xxx
PAYOS_CHECKSUM_KEY=xxx

# Sepay
SEPAY_API_TOKEN=xxx
SEPAY_BANK_ACCOUNT=xxx

# Frontend
FRONTEND_URL=http://localhost:5173
```

---

## 9. Mapping File Monolith → Microservice

| File gốc (monolith) | → Service | Ghi chú |
|---|---|---|
| `RefreshToken.java` | iam-service | `userId: Integer` thay vì FK |
| `User.java` | user-service | Xóa `List<Booking>`, `List<Review>` |
| `FavoriteTour.java` | user-service | `tourId: Integer` thay vì FK |
| `Notification.java` | user-service | Giữ nguyên |
| `UserNotification.java` | user-service | Giữ nguyên |
| `Tour.java` | tour-service | Giữ nguyên |
| `TourImage.java` | tour-service | Giữ nguyên |
| `TourMedia.java` | tour-service | Giữ nguyên |
| `ItineraryDay.java` | tour-service | Giữ nguyên |
| `Location.java` | tour-service | Giữ nguyên |
| `TourDeparture.java` | tour-service | Xóa `List<Booking>` |
| `DeparturePricing.java` | tour-service | Giữ nguyên |
| `DepartureTransport.java` | tour-service | Giữ nguyên |
| `PolicyTemplate.java` | tour-service | Giữ nguyên |
| `BranchContact.java` | tour-service | Giữ nguyên |
| `Coupon.java` | tour-service | Giữ nguyên |
| `Review.java` | tour-service | `userId: Integer`, `bookingId: Integer` |
| `ImageReview.java` | tour-service | Giữ nguyên |
| `Booking.java` | booking-service | `userId: Integer`, `departureId: Integer` |
| `BookingPassenger.java` | booking-service | Giữ nguyên |
| `RefundInformation.java` | booking-service | Giữ nguyên |
| `Payment.java` | payment-service | `bookingId: Integer` thay vì FK |

### Controllers
| File gốc | → Service |
|---|---|
| `AuthController.java` | iam-service |
| `AdminAuthController.java` | iam-service |
| `UserController.java` | user-service |
| `AdminProfileController.java` | user-service |
| `FavoriteTourController.java` | user-service |
| `NotificationController.java` | user-service |
| `TourController.java` | tour-service |
| `TourManagementController.java` | tour-service |
| `TourDepartureManagementController.java` | tour-service |
| `TourUploadController.java` | tour-service |
| `TourMediaController.java` | tour-service |
| `LocationController.java` | tour-service |
| `LocationAdminController.java` | tour-service |
| `PolicyTemplateController.java` | tour-service |
| `BranchContactController.java` | tour-service |
| `CouponController.java` | tour-service |
| `ReviewController.java` | tour-service |
| `ChatbotController.java` | tour-service |
| `BookingController.java` | booking-service |
| `DashboardController.java` | booking-service |
| `PaymentController.java` | payment-service |
| `UserFakerController.java` | ❌ Bỏ (dev-only) |

### Services
| File gốc | → Service |
|---|---|
| `AuthServiceImpl.java` | iam-service |
| `GoogleAuthServiceImpl.java` | iam-service |
| `EmailServiceImpl.java` | iam-service + user-service (copy) |
| `MailServiceImpl.java` | iam-service + user-service (copy) |
| `UserServiceImpl.java` | user-service |
| `FavoriteTourServiceImpl.java` | user-service |
| `NotificationServiceImpl.java` | user-service |
| `CloudinaryServiceImpl.java` | user-service + tour-service (copy) |
| `TourServiceImpl.java` | tour-service |
| `TourManagementServiceImpl.java` | tour-service |
| `TourDepartureServiceImpl.java` | tour-service |
| `TourMediaServiceImpl.java` | tour-service |
| `LocationServiceImpl.java` | tour-service |
| `PolicyTemplateServiceImpl.java` | tour-service |
| `BranchContactServiceImpl.java` | tour-service |
| `CouponServiceImpl.java` | tour-service |
| `ReviewServiceImpl.java` | tour-service |
| `chatbot/ChatbotService.java` | tour-service |
| `chatbot/VectorService.java` | tour-service |
| `chatbot/VectorSyncService.java` | tour-service |
| `BookingServiceImpl.java` | booking-service |
| `BookingCleanupServiceImpl.java` | booking-service |
| `DashboardServiceImpl.java` | booking-service |
| `PaymentServiceImpl.java` | payment-service |
| `PayOSService.java` | payment-service |
| `SepayServiceImpl.java` | payment-service |

---

## 10. Booking Flow (End-to-End)

```
[Frontend]
  POST /api/bookings
    ↓
[api-gateway] → JWT verify → route → booking-service
    ↓
[booking-service.BookingService.createBooking()]
  1. Feign → tour-service  GET /internal/departures/{id}
     → Lấy: availableSlots, giá, tourName, departureDate
     → Validate: còn slot không?

  2. Feign → tour-service  GET /internal/coupons/validate?code=XXX&amount=YYY
     → Validate coupon hợp lệ, tính discount

  3. Tính totalPrice = subtotal - couponDiscount - paidByCoin + surcharge

  4. Nếu paidByCoin > 0:
     Feign → user-service  PUT /internal/users/{userId}/coin  {amount: -N}

  5. Lưu Booking (status=PENDING_PAYMENT) + BookingPassengers

  6. Feign → payment-service  POST /internal/payments
     → Tạo Payment entry (status=PENDING)
     → Nhận back: {paymentId, payUrl}

  7. Feign → tour-service  PUT /internal/departures/{id}/slots  {decrement: N}

  8. Feign → user-service  POST /internal/notifications
     → Push notification "Booking đang chờ thanh toán"

  9. Return {bookingId, payUrl}
    ↓
[Frontend] → redirect → VNPay/PayOS
    ↓
[VNPay/PayOS callback]  →  [payment-service.PaymentController]
  1. Verify signature
  2. Cập nhật Payment.status = SUCCESS
  3. Feign → booking-service  PUT /internal/bookings/{id}/confirm-paid
     → Booking.status = PENDING_CONFIRMATION (hoặc CONFIRMED)
  4. booking-service → Feign → user-service: push notification "Thanh toán thành công"
```

---

## 11. Frontend Update (client-side)

### Chỉ cần thay đổi:

**`D:\KLTN\client-side\.env`**
```dotenv
# TRƯỚC
VITE_API_URL=http://localhost:8088

# SAU
VITE_API_URL=http://localhost:8080
```

**Không cần thay đổi bất kỳ component React nào** — API path `/api/...` giữ nguyên, Gateway lo routing.

### Nếu cần fix CORS:
```typescript
// src/lib/axios.ts (nếu có)
const api = axios.create({
  baseURL: import.meta.env.VITE_API_URL,
  withCredentials: true,   // nếu dùng cookie
});
```

---

## 12. Checklist Ngày 1 — Scaffold

```bash
# 1. Tạo thư mục project
cd D:\KLTN\tourism-microservices-v2

# 2. Tạo parent pom.xml
# (copy template từ tài liệu này)

# 3. Tạo từng module với Spring Initializr hoặc copy từ monolith

# 4. Tạo init-db.sql và docker-compose.yml

# 5. Chạy PostgreSQL
docker-compose up -d postgres

# 6. Tạo databases
docker exec -it tourism-postgres psql -U tourism_user -c \
  "CREATE DATABASE tourism_iam; CREATE DATABASE tourism_user; \
   CREATE DATABASE tourism_tour; CREATE DATABASE tourism_booking; \
   CREATE DATABASE tourism_payment;"

# 7. Chạy Eureka
cd service-registry && mvn spring-boot:run

# 8. Chạy Gateway
cd api-gateway && mvn spring-boot:run

# Verify: http://localhost:8761 → thấy Eureka dashboard
```

---

## 13. Tips Port Code Nhanh Từ Monolith

### Bước chuẩn cho mỗi service:

1. **Copy entity** từ `Tourism_Backend/entity/` → đổi package
2. **Xóa cross-service `@ManyToOne`** → thay `Integer xxxId`
3. **Copy repository** → đổi package, giữ nguyên method signatures
4. **Copy service** → đổi package, thay cross-service calls bằng Feign
5. **Copy controller** → đổi package, thêm `@RequestHeader("X-User-Id")` thay vì `SecurityContext`
6. **Viết Feign Client** cho các service cần gọi
7. **Thêm `application.yml`** với port + datasource + eureka

### Lấy userId từ JWT tại service (không gọi iam-service):
```java
// Mỗi service tự parse JWT header được inject bởi Gateway
@GetMapping("/profile")
public ResponseEntity<?> getProfile(
    @RequestHeader("X-User-Id") Integer userId,
    @RequestHeader("X-User-Role") String role) {
    // Gateway đã verify JWT và forward header X-User-Id, X-User-Role
}
```

### Gateway filter inject header:
```java
// GatewayConfig.java
.filter((exchange, chain) -> {
    String token = extractToken(exchange.getRequest());
    Claims claims = jwtProvider.parseToken(token);
    ServerHttpRequest modified = exchange.getRequest().mutate()
        .header("X-User-Id", claims.get("userId").toString())
        .header("X-User-Role", claims.get("role").toString())
        .build();
    return chain.filter(exchange.mutate().request(modified).build());
})
```

---

*Tài liệu tổng quan: [OVERVIEW.md](./OVERVIEW.md)*  
*Diagram: [entity_diagram_microservices.drawio](./entity_diagram_microservices.drawio)*
