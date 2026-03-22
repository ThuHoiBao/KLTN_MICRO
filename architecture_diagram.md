# 🗺️ Tourism Microservices — Architecture Diagram

## 1. Tổng Quan Kiến Trúc (Overview)

```mermaid
graph TB
    subgraph CLIENT["🖥️ Client Layer"]
        FE["React Frontend<br/>Vite + TypeScript<br/>:5173"]
    end

    subgraph INFRA_IN["🔧 Infrastructure — Entry"]
        GW["API Gateway<br/>Spring Cloud Gateway<br/>:8080<br/>[JWT Validate → Inject X-User-Id]"]
        EUR["Service Registry<br/>Eureka Server<br/>:8761"]
    end

    subgraph BUSINESS["💼 Business Services"]
        ID["identity-service<br/>:8081<br/>DB: tourism_identity"]
        TC["tour-catalog-service<br/>:8082<br/>DB: tourism_catalog"]
        BK["booking-service<br/>:8083<br/>DB: tourism_booking"]
        PY["payment-service<br/>:8084<br/>DB: tourism_payment"]
        RV["review-service<br/>:8085<br/>DB: tourism_review"]
        PR["promotion-service<br/>:8086<br/>DB: tourism_promotion"]
        NT["notification-service<br/>:8087<br/>DB: tourism_notification"]
        AN["analytics-service<br/>:8088<br/>DB: tourism_analytics"]
        CM["cms-service<br/>:8089<br/>DB: tourism_cms"]
    end

    subgraph INFRA_MSG["📨 Infrastructure — Messaging & Cache"]
        KF["Apache Kafka<br/>+ Zookeeper"]
        RD["Redis Cache<br/>:6379"]
    end

    subgraph INFRA_OBS["🔍 Infrastructure — Observability"]
        ZP["Zipkin<br/>Distributed Tracing<br/>:9411"]
    end

    subgraph EXTERNAL["🌐 External Services"]
        CLO["Cloudinary<br/>Image Storage"]
        MAIL["SMTP Gmail<br/>Email Service"]
        VNP["VNPay"]
        POS["PayOS"]
        SEP["SePay"]
        GGL["Google OAuth2<br/>API"]
        GEM["Google Gemini<br/>AI API"]
    end

    %% Client → Gateway
    FE -->|"HTTPS REST"| GW

    %% Gateway → Services (HTTP)
    GW -->|"route"| ID
    GW -->|"route"| TC
    GW -->|"route"| BK
    GW -->|"route"| PY
    GW -->|"route"| RV
    GW -->|"route"| PR
    GW -->|"route"| NT
    GW -->|"route"| AN
    GW -->|"route"| CM

    %% Eureka Discovery
    GW -.->|"discover"| EUR
    ID -.->|"register"| EUR
    TC -.->|"register"| EUR
    BK -.->|"register"| EUR
    PY -.->|"register"| EUR
    RV -.->|"register"| EUR
    PR -.->|"register"| EUR
    NT -.->|"register"| EUR
    AN -.->|"register"| EUR
    CM -.->|"register"| EUR

    %% Feign Internal Calls
    BK -->|"Feign: /internal/tours/{id}"| TC
    BK -->|"Feign: /internal/coupons/{code}/apply"| PR
    PY -->|"Feign: /internal/bookings/{code}/confirm"| BK
    RV -->|"Feign: /internal/bookings/check-confirmed"| BK

    %% Redis
    BK -->|"cache"| RD

    %% Kafka Events
    ID -->|"user.registered"| KF
    BK -->|"booking.created<br/>booking.confirmed<br/>booking.cancelled"| KF
    PY -->|"payment.completed"| KF
    RV -->|"review.created"| KF
    KF -->|"consume"| NT
    KF -->|"consume"| AN

    %% Zipkin Tracing
    GW -.->|"trace"| ZP
    BK -.->|"trace"| ZP

    %% External
    ID -->|"OAuth2"| GGL
    ID -->|"upload avatar"| CLO
    ID -->|"send verify email"| MAIL
    TC -->|"upload images"| CLO
    PY -->|"redirect"| VNP
    PY -->|"webhook"| POS
    PY -->|"webhook"| SEP
    NT -->|"send email"| MAIL
    RV -->|"upload photos"| CLO
    AN -->|"AI chat"| GEM

    %% Styles
    classDef gateway fill:#f4a261,stroke:#e76f51,color:#fff,font-weight:bold
    classDef service fill:#457b9d,stroke:#1d3557,color:#fff
    classDef infra fill:#2a9d8f,stroke:#264653,color:#fff
    classDef external fill:#6c757d,stroke:#495057,color:#fff
    classDef client fill:#e9c46a,stroke:#f4a261,color:#333,font-weight:bold
    classDef kafka fill:#e63946,stroke:#c1121f,color:#fff,font-weight:bold

    class GW gateway
    class ID,TC,BK,PY,RV,PR,NT,AN,CM service
    class EUR,RD,ZP infra
    class CLO,MAIL,VNP,POS,SEP,GGL,GEM external
    class FE client
    class KF kafka
```

---

## 2. Luồng Kafka Events (Event-Driven Flow)

```mermaid
flowchart LR
    subgraph PRODUCERS["📤 Event Producers"]
        ID_P["identity-service"]
        BK_P["booking-service"]
        PY_P["payment-service"]
        RV_P["review-service"]
    end

    subgraph KAFKA["🔴 Apache Kafka Topics"]
        T1["user.registered"]
        T2["booking.created"]
        T3["booking.confirmed"]
        T4["booking.cancelled"]
        T5["payment.completed"]
        T6["review.created"]
    end

    subgraph CONSUMERS["📥 Event Consumers"]
        NT_C["notification-service<br/>📧 Email + In-App"]
        AN_C["analytics-service<br/>📊 Stats + Revenue"]
    end

    ID_P -->|publish| T1
    BK_P -->|publish| T2
    BK_P -->|publish| T3
    BK_P -->|publish| T4
    PY_P -->|publish| T5
    RV_P -->|publish| T6

    T1 -->|welcome email| NT_C
    T2 -->|email đặt tour + notification| NT_C
    T2 -->|new_bookings++| AN_C
    T3 -->|email xác nhận| NT_C
    T3 -->|confirmed_bookings++| AN_C
    T4 -->|email huỷ tour| NT_C
    T4 -->|cancelled_bookings++| AN_C
    T5 -->|email thanh toán| NT_C
    T5 -->|total_revenue += amount| AN_C
    T6 -->|update avgRating| AN_C
```

---

## 3. Luồng Đặt Tour (Core Business Flow)

```mermaid
sequenceDiagram
    actor User
    participant FE as React Frontend
    participant GW as API Gateway
    participant ID as identity-service
    participant TC as tour-catalog-service
    participant BK as booking-service
    participant PR as promotion-service
    participant PY as payment-service
    participant KF as Kafka
    participant NT as notification-service

    User->>FE: Đăng nhập
    FE->>GW: POST /api/auth/login
    GW->>ID: forward
    ID-->>FE: {accessToken, refreshToken}

    User->>FE: Xem tour & chọn lịch khởi hành
    FE->>GW: GET /api/tours/{id}/departures/available
    GW->>TC: forward
    TC-->>FE: danh sách departure còn chỗ

    User->>FE: Đặt tour (nhập hành khách + coupon)
    FE->>GW: POST /api/bookings [Bearer token]
    GW->>BK: forward + inject X-User-Id
    BK->>TC: Feign /internal/tours/{id} (lấy giá)
    BK->>PR: Feign /internal/coupons/{code}/apply (validate coupon)
    BK-->>BK: Tính total_amount, tạo booking PENDING_PAYMENT
    BK->>KF: publish booking.created
    BK-->>FE: {bookingCode}

    KF->>NT: consume booking.created
    NT-->>User: Email "Đặt tour thành công"

    User->>FE: Thanh toán (chọn VNPay/PayOS/SePay)
    FE->>GW: POST /api/payments/initiate
    GW->>PY: forward
    PY-->>FE: {paymentUrl}
    FE-->>User: Redirect đến trang thanh toán

    User-->>PY: Thanh toán thành công (callback)
    PY->>BK: Feign /internal/bookings/{code}/confirm
    BK-->>BK: status → CONFIRMED
    BK->>KF: publish booking.confirmed
    PY->>KF: publish payment.completed

    KF->>NT: consume booking.confirmed
    NT-->>User: Email "Xác nhận thành công"
    KF->>NT: consume payment.completed
    NT-->>User: Email "Thanh toán thành công"
```

---

## 4. Sơ Đồ Database Per Service

```mermaid
graph LR
    subgraph PG["🐘 PostgreSQL 15"]
        DB1[("tourism_identity")]
        DB2[("tourism_catalog")]
        DB3[("tourism_booking")]
        DB4[("tourism_payment")]
        DB5[("tourism_review")]
        DB6[("tourism_promotion")]
        DB7[("tourism_notification")]
        DB8[("tourism_analytics")]
        DB9[("tourism_cms")]
    end

    ID["identity-service :8081"] --> DB1
    TC["tour-catalog-service :8082"] --> DB2
    BK["booking-service :8083"] --> DB3
    PY["payment-service :8084"] --> DB4
    RV["review-service :8085"] --> DB5
    PR["promotion-service :8086"] --> DB6
    NT["notification-service :8087"] --> DB7
    AN["analytics-service :8088"] --> DB8
    CM["cms-service :8089"] --> DB9

    classDef svc fill:#457b9d,stroke:#1d3557,color:#fff
    classDef db fill:#2a9d8f,stroke:#264653,color:#fff
    class ID,TC,BK,PY,RV,PR,NT,AN,CM svc
    class DB1,DB2,DB3,DB4,DB5,DB6,DB7,DB8,DB9 db
```

---

## 5. Bảng Tổng Hợp Services

| Service | Port | Database | Feign Calls | Kafka Publish | Kafka Consume | External |
|---------|------|----------|-------------|---------------|---------------|----------|
| **api-gateway** | 8080 | — | — | — | — | — |
| **service-registry** | 8761 | — | — | — | — | — |
| **identity-service** | 8081 | tourism_identity | — | `user.registered` | — | Google OAuth, Cloudinary, SMTP |
| **tour-catalog-service** | 8082 | tourism_catalog | — | — | — | Cloudinary |
| **booking-service** | 8083 | tourism_booking | tour-catalog, promotion | `booking.created/confirmed/cancelled` | — | Redis |
| **payment-service** | 8084 | tourism_payment | booking | `payment.completed` | — | VNPay, PayOS, SePay |
| **review-service** | 8085 | tourism_review | booking | `review.created` | — | Cloudinary |
| **promotion-service** | 8086 | tourism_promotion | — | — | — | — |
| **notification-service** | 8087 | tourism_notification | — | — | `booking.*`, `payment.completed`, `user.registered` | SMTP |
| **analytics-service** | 8088 | tourism_analytics | — | — | `booking.*`, `payment.completed`, `review.created` | Google Gemini AI |
| **cms-service** | 8089 | tourism_cms | — | — | — | — |
