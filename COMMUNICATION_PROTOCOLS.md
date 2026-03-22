# 📡 Giao Tiếp Giữa Các Microservices — Giải Thích Đầy Đủ

> **Dành cho dự án:** Tourism Microservices Platform  
> **Liên quan:** `architecture_diagram_animated.drawio` · `ARCHITECTURE.md`

---

## Tổng Quan — 4 Cách Services Giao Tiếp

```
┌─────────────────────────────────────────────────────────────────────┐
│  Synchronous (Đồng bộ — chờ kết quả)   │  Async (Bất đồng bộ)     │
├─────────────────────────────────────────┼───────────────────────────┤
│  REST / HTTP     ← API Gateway routing  │  Kafka     ← Event Bus    │
│  Feign Client    ← Internal service     │  WebSocket ← Push to UI   │
│  gRPC            ← High-perf internal   │                           │
└─────────────────────────────────────────┴───────────────────────────┘
```

---

## 1. gRPC — Là Gì? Dùng Khi Nào?

### 1.1 Định Nghĩa

**gRPC** (Google Remote Procedure Call) là một framework giao tiếp **tốc độ cao, nhị phân** giữa các service. Thay vì gửi JSON text (như REST), gRPC **serialize dữ liệu thành binary** qua **Protocol Buffers (protobuf)**.

```
REST/Feign:  {"id": 1, "name": "Ha Long Tour", "price": 2500000}
             ↕ JSON text → ~47 bytes

gRPC:        \x08\x01\x12\x0eHa Long Tour\x18\x80\xC4\x98\x01
             ↕ Binary → ~18 bytes (60% nhỏ hơn, 5-10x faster)
```

### 1.2 Cách gRPC Hoạt Động

**Bước 1:** Định nghĩa contract trong file `.proto`
```protobuf
// tour.proto
syntax = "proto3";

service TourService {
  rpc GetTourById (TourRequest) returns (TourResponse);
  rpc CheckAvailability (AvailabilityRequest) returns (AvailabilityResponse);
}

message TourRequest {
  int64 tour_id = 1;
}

message TourResponse {
  int64 id = 1;
  string name = 2;
  double price = 3;
  int32 available_slots = 4;
}
```

**Bước 2:** Generate code tự động từ file `.proto`
```
proto compiler → Java stub (client + server code)
```

**Bước 3:** Gọi như gọi method thông thường
```java
// Booking Service gọi Tour Catalog Service qua gRPC
TourResponse tour = tourServiceStub.getTourById(
    TourRequest.newBuilder().setTourId(tourId).build()
);
// Nhận kết quả ngay (synchronous)
```

### 1.3 gRPC vs REST/Feign — So Sánh

| Tiêu chí | REST (JSON) | Feign Client | gRPC |
|---------|-------------|--------------|------|
| **Format** | JSON text | JSON text | Binary protobuf |
| **Tốc độ** | Chậm | Chậm | **Nhanh hơn 5-10x** |
| **Kích thước payload** | Lớn | Lớn | **Nhỏ hơn 60-70%** |
| **Contract** | Swagger/OpenAPI | Interface Java | File `.proto` (type-safe) |
| **Streaming** | Không | Không | **Có (bi-directional)** |
| **Browser support** | ✅ | ✅ | ❌ (không trực tiếp) |
| **Độ phức tạp** | Đơn giản | Đơn giản | Phức tạp hơn |
| **Dùng khi nào** | Public API | Internal call ít traffic | Internal call nhiều traffic |

### 1.4 Ví Dụ Trong Reference Image (Hematology System)

Trong ảnh tham chiếu (Hematology Management App), các kết nối **gRPC** được vẽ là:
```
API Gateway → IAM Service        (gRPC)
IAM Service → Test Order Service (gRPC) ← auth token verification
IAM Service → Monitoring Service (gRPC) ← auth token verification
API Gateway → Instrument Service (gRPC)
```
**Lý do dùng gRPC** trong đó: Hệ thống y tế cần verify JWT token TẠI MỖI service (IAM check) với **latency cực thấp** → gRPC phù hợp hơn REST.

### 1.5 Tourism Project Của Bạn Dùng Gì?

**Project của bạn dùng Feign Client** (REST-based), không phải gRPC. Đây là lựa chọn **hoàn toàn hợp lý** cho KLTN vì:
- Đơn giản hơn để implement và debug
- Feign tích hợp native với Spring Cloud / Eureka
- Traffic internal không đủ lớn để cần gRPC optimization

**Nếu bạn muốn dùng gRPC** thay Feign, đây là ví dụ:

```
Thay vì:
  Feign: GET http://tour-catalog-service/internal/tours/{id}  (46ms avg)

Dùng gRPC:
  gRPC: tourServiceStub.getTourById(TourRequest) → TourResponse (8ms avg)
```

---

## 2. Feign Client — Internal REST Call

**Project của bạn dùng Feign** cho tất cả internal service-to-service calls:

```java
// Booking Service gọi Tour Catalog
@FeignClient(name = "tour-catalog-service")
public interface TourCatalogClient {
    @GetMapping("/internal/tours/{id}")
    TourResponse getTourById(@PathVariable Long id);
    
    @PutMapping("/internal/tours/{id}/reserve")
    void reserveSlot(@PathVariable Long id, @RequestBody ReserveRequest req);
}

// Trong BookingService:
TourResponse tour = tourCatalogClient.getTourById(tourId);
// ← Feign tự động: serialize → HTTP GET → deserialize → return
```

**Luồng Feign trong Tourism Project:**

```
BookingService ──Feign GET──► TourCatalogService /internal/tours/{id}
                                ← TourResponse{name, price, slots}
BookingService ──Feign POST──► PromotionService /internal/coupons/{code}/apply
                                ← DiscountResponse{discount, finalPrice}
PaymentService ──Feign PUT──► BookingService /internal/bookings/{code}/confirm
                                ← BookingResponse{status: CONFIRMED}
ReviewService ──Feign GET──► BookingService /internal/bookings/check-confirmed
                                ← boolean (đã booking xong chưa)
```

---

## 3. Message Broker (Apache Kafka) — Bất Đồng Bộ

### 3.1 Kafka Là Gì?

**Kafka** là một **Message Broker** — thành phần trung gian giúp các service giao tiếp **bất đồng bộ** mà không cần biết nhau.

```
Không có Kafka (coupling):                 Có Kafka (decoupled):
                                           
BookingService                             BookingService
  → gọi NotificationService (blocking)      → publish("booking.created")
  → gọi AnalyticsService (blocking)                    ↓
  → gọi InventoryService (blocking)         Kafka Topic "booking.created"
  ← chờ tất cả xong mới return                 ↓           ↓           ↓
                                         Notification  Analytics  Inventory
                                           (độc lập)   (độc lập)  (độc lập)
```

### 3.2 Kafka Hoạt Động Như Thế Nào?

**3 khái niệm cốt lõi:**

| Khái niệm | Ý nghĩa | Ví dụ Tourism |
|-----------|---------|---------------|
| **Topic** | "Kênh" nhận message | `booking.created`, `payment.completed` |
| **Producer** | Service gửi message lên topic | Booking Service, Payment Service |
| **Consumer** | Service đọc message từ topic | Notification Service, Analytics Service |

**Ví dụ cụ thể trong Tourism Project:**

```java
// 1. PRODUCER: Booking Service publish event
@Service
public class BookingService {
    private final KafkaTemplate<String, BookingCreatedEvent> kafkaTemplate;
    
    public BookingResponse createBooking(CreateBookingRequest req) {
        // ... xử lý booking ...
        Booking booking = bookingRepository.save(newBooking);
        
        // Publish event → Kafka không chờ consumer xử lý
        kafkaTemplate.send("booking.created", new BookingCreatedEvent(
            booking.getId(),
            booking.getUserId(),
            booking.getTourName(),
            booking.getDepartureDate()
        ));
        
        return BookingResponse.from(booking); // Return NGAY, không chờ email
    }
}

// 2. CONSUMER: Notification Service lắng nghe
@Service
public class NotificationConsumer {
    
    @KafkaListener(topics = "booking.created")
    public void handleBookingCreated(BookingCreatedEvent event) {
        // Xử lý HOÀN TOÀN ĐỘC LẬP với Booking Service
        emailService.sendBookingConfirmation(
            event.getUserId(),
            event.getTourName(),
            event.getDepartureDate()
        );
    }
}

// 3. CONSUMER: Analytics Service cũng lắng nghe CÙNG event
@Service
public class AnalyticsConsumer {
    
    @KafkaListener(topics = "booking.created")
    public void handleBookingCreated(BookingCreatedEvent event) {
        // 2 consumers nhận cùng 1 event
        dailyStatsRepository.incrementBookingCount(LocalDate.now());
        tourStatsRepository.incrementPopularity(event.getTourId());
    }
}
```

### 3.3 Tất Cả Kafka Topics Trong Tourism Project

```
                     PRODUCERS              TOPICS                    CONSUMERS
                     ─────────           ──────────────           ─────────────────
Identity Service  ──────────────►  user.registered        ──►  Notification (welcome email)
                                                           ──►  Analytics (new user stats)

Booking Service   ──────────────►  booking.created        ──►  Notification (booking email)
                  ──────────────►  booking.confirmed       ──►  Notification (confirmed email)
                  ──────────────►  booking.cancelled       ──►  Notification (cancel email)
                                                           ──►  Analytics (booking stats)

Payment Service   ──────────────►  payment.completed      ──►  Notification (receipt email)
                                                           ──►  Booking (auto-confirm status)
                                                           ──►  Analytics (revenue stats)

Review Service    ──────────────►  review.created         ──►  Analytics (avg rating update)
```

### 3.4 Kafka vs gRPC — Khi Nào Dùng Gì?

```
Dùng gRPC / Feign (synchronous) khi:
  ✅ Cần kết quả NGAY để xử lý tiếp
  ✅ Ví dụ: Kiểm tra tour còn chỗ TRƯỚC khi tạo booking
  ✅ Ví dụ: Verify JWT token trước khi cho phép request

Dùng Kafka (asynchronous) khi:
  ✅ Không cần kết quả ngay
  ✅ 1 event cần nhiều service xử lý đồng thời
  ✅ Service có thể xử lý sau (retry, delay OK)
  ✅ Ví dụ: Gửi email sau khi booking (không cần chờ email xong mới return)
  ✅ Ví dụ: Cập nhật analytics (không urgent)
```

---

## 4. Real-time Notify — WebSocket

### 4.1 Real-time Notify Là Gì?

**WebSocket** là giao thức kết nối **2 chiều, liên tục** giữa server và browser. Khác với HTTP (request→response), WebSocket giữ connection mở để server **chủ động push data** xuống client bất kỳ lúc nào.

```
HTTP (REST):          Client ──request──► Server ──response──► Client
                      ← Mỗi lần cần phải request lại

WebSocket:            Client ◄──── persistent connection ────► Server
                      Server có thể push data bất kỳ lúc nào ──► Client
```

### 4.2 Luồng Real-time Trong Tourism Project

```
1. User mở trang "Booking Status" trên browser
   Browser → WebSocket handshake → Real-time Notify Server (ws://...)
   Connection được giữ mở

2. User thực hiện booking (via REST → Booking Service)
   Booking Service → Kafka.publish("booking.created")

3. Kafka → Real-time Notify Service nhận event
   Real-time Notify Server → push notification xuống browser của user
   Browser hiển thị ngay: "✅ Booking #2024001 đã được tạo thành công!"

4. Sau đó Payment Service xử lý xong
   Kafka → Real-time Notify → push: "💳 Thanh toán thành công! Booking đã confirmed."

5. Tất cả xảy ra NGAY LẬP TỨC, không cần user refresh trang
```

```java
// Real-time Notify Service
@Service
public class RealtimeNotifyService {
    
    private final SimpMessagingTemplate websocketTemplate;
    
    @KafkaListener(topics = {"booking.created", "booking.confirmed", "payment.completed"})
    public void handleEvent(BaseEvent event) {
        // Push notification đến user cụ thể qua WebSocket
        websocketTemplate.convertAndSendToUser(
            event.getUserId().toString(),
            "/queue/notifications",
            NotificationMessage.from(event)
        );
    }
}

// React Frontend subscribe WebSocket
const client = new StompClient({ brokerURL: 'ws://localhost:8080/ws' });
client.subscribe(`/user/queue/notifications`, (message) => {
    const notification = JSON.parse(message.body);
    showToast(notification.title, notification.body); // Hiện toast notification
    dispatch(updateBookingStatus(notification));      // Update Redux store
});
```

### 4.3 Vì Sao Cần Cả Kafka VÀ WebSocket?

```
Kafka (Backend-to-backend):
  Booking Service ──kafka──► Notification Service ──smtp──► Email user

WebSocket (Backend-to-browser):
  Kafka ──kafka──► Real-time Service ──websocket──► Browser (toast notification)

Chúng bổ sung cho nhau:
  - Kafka: giao tiếp GIỮA các services (backend world)
  - WebSocket: push data xuống BROWSER (frontend world)
  - Không thể dùng Kafka để push trực tiếp lên browser
  - Không thể dùng WebSocket để scale backend event-driven
```

---

## 5. Sơ Đồ Tổng Hợp — Luồng Đặt Tour Hoàn Chỉnh

```
User (browser)
  │
  │  [1] POST /api/bookings ─────── REST/HTTPS ──────────────────────────╮
  │                                                                       ↓
  │                                                              API Gateway
  │                                                                       │
  │                                                           [2] REST routing
  │                                                                       ↓
  │                                                           Booking Service
  │                                                                       │
  │                                              [3] Feign: GET /internal/tours/{id}
  │                                                                       ↓
  │                                                         Tour Catalog Service
  │                                                            ← TourResponse
  │                                                                       │
  │                                          [4] Feign: POST /internal/coupons/apply
  │                                                                       ↓
  │                                                         Promotion Service
  │                                                            ← DiscountResponse
  │                                                                       │
  │                                                           [5] Save to DB + Redis
  │                                                                       │
  │                                     [6] Kafka.publish("booking.created") ─────╮
  │                                                                       │        │
  │  [7] BookingResponse ←────── REST response ─────────────────────────╯        │
  │  (return ngay, không chờ email)                                               │
  │                                                               Kafka Topic ────┤
  │                                                                       │        │
  │  [8] WebSocket push ◄─── Real-time Service ◄── Kafka consumer        │        │
  │  "Booking #001 created!"                                              │        │
  │                                                          Notification Service ←╯
  │                                                          ← SMTP email sent
  │                                                                       │
  │                                                          Analytics Service ←──╯
  │                                                          ← DailyStats updated
```

---

## 6. Bảng So Sánh Nhanh

| | REST (Public) | Feign (Internal) | gRPC (Internal) | Kafka (Async) | WebSocket |
|--|--|--|--|--|--|
| **Direction** | Request→Response | Request→Response | Request→Response | Publisher→Consumer | Server→Client Push |
| **Speed** | Medium | Medium | **Fast** | N/A (async) | Real-time |
| **Data format** | JSON | JSON | Binary | JSON/Avro | JSON |
| **Browser** | ✅ | ❌ | ❌ | ❌ | ✅ |
| **Scale** | Good | Good | **Excellent** | **Excellent** | Medium |
| **Use in project** | ✅ GW→Service | ✅ Service→Service | ❌ (dùng Feign) | ✅ Events | ✅ Notifications |
| **Retry/Recovery** | Manual | Manual | Manual | **Built-in** | Manual |

---

## 7. Khuyến Nghị Cho Dự Án Của Bạn

**Hiện tại (Feign):** Hoàn toàn ổn cho KLTN. Đơn giản, dễ debug, native với Spring Cloud.

**Nếu muốn upgrade lên gRPC** sau khi bảo vệ:
```
Priority 1: Booking → TourCatalog (hot path, mỗi booking đều gọi)
Priority 2: Payment → Booking (confirm sau thanh toán, critical)
Priority 3: Review → Booking (check-confirmed, nhiều lần gọi)

Giữ Feign cho: Booking → Promotion (ít dùng hơn, coupon queries)
```

**Kafka:** Đã implement đúng. Thêm **Dead Letter Queue** để handle failed events.

**WebSocket:** Đã có Real-time Notify. Thêm heartbeat ping/pong để detect disconnection.
