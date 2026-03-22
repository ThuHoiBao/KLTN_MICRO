# 📐 Tourism Microservices — Architecture Diagram Guide

> **File**: `architecture_diagram_animated.drawio`  
> **Tool**: [draw.io Desktop](https://github.com/jgraph/drawio-desktop/releases) hoặc [app.diagrams.net](https://app.diagrams.net)

---

## 🚀 Cách mở file

| Phương thức | Cách thực hiện |
|-------------|----------------|
| **draw.io Desktop** (khuyến nghị) | Mở app → File → Open → chọn `.drawio` |
| **app.diagrams.net** (online) | Truy cập web → File → Open from → Device |
| **VS Code** | Cài extension [Draw.io Integration](https://marketplace.visualstudio.com/items?itemName=hediet.vscode-drawio) |

> ⚠️ **Animation nét đứt chuyển động** chỉ hiển thị trong draw.io Desktop và app.diagrams.net — **không** hiển thị trong VS Code preview.

---

## 🗺️ Bố Cục Diagram

```
 ┌─────────────────────────────────────────────────────────────────┐
 │              System Management Module (dashed amber border)      │
 │  ┌──────┐  ┌──────────┐  ┌─────────┐  ┌─────────┐             │
 │  │Identity│  │TourCatalog│  │ Booking │  │ Payment │  ┌──────┐ │
 │  │Service │  │ Service  │  │ Service │  │ Service │  │Kafka │ │
 │  │  [DB]  │  │  [DB]   │  │  [DB]  │  │  [DB]  │  │(MQ)  │ │
 │  └──────┘  └──────────┘  └─────────┘  └─────────┘  │      │ │
 │  ┌──────┐  ┌──────────┐  ┌─────────┐               │      │ │
 │  │Review │  │Promotion │  │  CMS   │                │ Real │ │
 │  │Service│  │ Service  │  │Service │                │-time │ │
 │  │  [DB] │  │  [DB]   │  │  [DB]  │                │Notify│ │
 │  └──────┘  └──────────┘  └─────────┘               └──────┘ │
 │  ┌──────┐  ┌──────────┐  [Redis]     [Zipkin]               │
 │  │Notif. │  │Analytics │                                      │
 │  │Service│  │+AI Chat  │                                      │
 │  │  [DB] │  │  [DB]   │                                      │
 │  └──────┘  └──────────┘                                      │
 │  ╔══════════════════════════════╗                             │
 │  ║       Shared Libraries       ║                             │
 │  ║ common-security | common-evt ║                             │
 │  ╚══════════════════════════════╝                             │
 └─────────────────────────────────────────────────────────────────┘
```

---

## 🎨 Ký Hiệu và Màu Sắc

### Khối hình (Shapes)

| Hình | Ý nghĩa | Màu |
|------|---------|-----|
| 🌐 Globe icon (`www_server`) | Microservice instance | Xanh dương `#1e6db5` |
| 🗄️ Cylinder (`cylinder3`) | PostgreSQL Database | Xanh dương `#1e6db5` |
| 🔶 Router icon | API Gateway (Spring Cloud GW) | Vàng/Amber `#e8a217` |
| 📦 MQ icon | Apache Kafka Message Broker | Vàng/Amber `#e8a217` |
| 🖥️ Workstation icon | Real-time Notify (WebSocket) | Xanh biển `#0288d1` |
| ⬟ Swimlane box | Service container (tên + icon + DB) | Xanh nhạt `#f0f4fa` |

### Module và Container

| Container | Border | Ý nghĩa |
|-----------|--------|---------|
| **System Management Module** | Dashed vàng `#e8a217` | Toàn bộ hệ thống backend |
| **Service box** | Xanh `#5c85c8` solid | 1 microservice độc lập |
| **Shared Libraries** | Tím `#7b1fa2` | Thư viện dùng chung |

---

## ➡️ Loại Đường Kẻ (Edge Types)

> **Kỹ thuật tránh đường kẻ đè nhau:** Mỗi edge từ API Gateway dùng một `exitY` khác nhau (0.07 → 0.93), tạo hiệu ứng "fan-out" — 9 đường kẻ tách ra hoàn toàn, không bao giờ đè lên nhau.

### 1️⃣ HTTP Route — Màu theo service (animated)
```
╌ ╌ ╌ ╌ ╌ ╌ ╌ ╌ ╌ ►  (mỗi service màu riêng, chuyển động)
```
| Service | Màu edge | Hex |
|---------|----------|-----|
| Identity | Tím | `#9C27B0` |
| Tour Catalog | Xanh dương | `#1565C0` |
| Booking | Xanh lá | `#2E7D32` |
| Payment | Cam | `#E65100` |
| Review | Hồng | `#880E4F` |
| Promotion | Tím đậm | `#6A1B9A` |
| CMS | Xanh ngọc | `#004D40` |
| Notification | Navy | `#0D47A1` |
| Analytics | Vàng cam | `#F57F17` |

### 2️⃣ Feign Internal Call — Cam đậm animated
```
╌  ╌  ╌  ╌  ╌  ╌  ►  (dashPattern: 10 5, animationSpeed=5)
```
- **Màu**: `#FF6B35` (cam đậm)
- **Ý nghĩa**: Giao tiếp **đồng bộ** giữa các service qua OpenFeign  
- **Flows**: Booking→Catalog, Booking→Promotion, Payment→Booking, Review→Booking

### 3️⃣ Kafka Publish — Đỏ animated ✨
```
╌   ╌   ╌   ╌   ╌   ►  (dashPattern: 12 5, animationSpeed=4)
```
- **Màu**: `#E63946` (đỏ)
- **Ý nghĩa**: Service publish event lên Kafka broker
- **Flows**: Identity/Booking/Payment/Review → Kafka

### 4️⃣ Kafka Consume — Đỏ đậm animated ✨
```
╌   ╌   ╌   ╌   ╌   ►  (dashPattern: 12 5, strokeColor=#B71C1C)
```
- **Màu**: `#B71C1C` (đỏ đậm)
- **Ý nghĩa**: Kafka broker push event xuống consumer service
- **Flows**: Kafka → Notification, Kafka → Analytics

### 5️⃣ External Service — Xám nhạt animated
```
- - - - - - - - - ►  (dashPattern: 8 4, animationSpeed=3)
```
- **Màu**: `#6c757d` (xám)
- **Ý nghĩa**: Gọi external service bên ngoài hệ thống

---

## 📋 Danh Sách Services

| Service | Port | Database | Chức năng chính |
|---------|------|----------|-----------------|
| **Identity Service** | 8081 | tourism_identity | JWT auth, Google OAuth2, email verify |
| **Tour Catalog Service** | 8082 | tourism_catalog | CRUD tour, tìm kiếm, upload ảnh |
| **Booking Service** | 8083 | tourism_booking | Đặt tour, quản lý booking lifecycle |
| **Payment Service** | 8084 | tourism_payment | VNPay / PayOS / SePay integration |
| **Review Service** | 8085 | tourism_review | Rating + review sau khi booking hoàn thành |
| **Promotion Service** | 8086 | tourism_promotion | Coupon, discount code management |
| **CMS Service** | 8087 | tourism_cms | Quản lý nội dung, blog, banner |
| **Notification Service** | 8088 | tourism_notification | Email/push notification qua SMTP |
| **Analytics + AI Chatbot** | 8089 | tourism_analytics | Google Gemini AI, DailyStats, reporting |

---

## 🔄 Luồng Hoạt Động Chính

### 1. User Đặt Tour

```
User → React Frontend → API Gateway
  → Booking Service
    → Feign: TourCatalog (kiểm tra tour còn chỗ)
    → Feign: Promotion (áp dụng coupon nếu có)
    → Redis (cache pricing)
    → Kafka.publish("booking.created")
      → Notification Service (gửi email xác nhận)
      → Analytics Service (cập nhật stats)
```

### 2. Thanh Toán

```
User → API Gateway → Payment Service
  → VNPay/PayOS/SePay (external)
  ← Webhook callback
  → Feign: Booking Service (confirm booking)
  → Kafka.publish("payment.completed")
    → Notification Service (email receipt)
    → Analytics Service (revenue stats)
```

### 3. Viết Review

```
User → API Gateway → Review Service
  → Feign: Booking Service (kiểm tra booking đã hoàn thành)
  → Lưu review + rating
  → Kafka.publish("review.created")
    → Analytics Service (cập nhật avg rating)
    → CMS Service (highlight review)
```

---

## ⚙️ Cách Bật/Tắt Animation

Animation được cung cấp bởi draw.io qua thuộc tính `flowAnimation`:

```xml
<!-- Bật animation -->
style="dashed=1;dashPattern=10 4;flowAnimation=1;animationSpeed=4;strokeColor=#e8a217;"

<!-- Tắt animation (xóa flowAnimation=1) -->
style="dashed=1;dashPattern=10 4;strokeColor=#e8a217;"
```

Để chỉnh tốc độ:
- `animationSpeed=2` → Chậm (external connections)
- `animationSpeed=4` → Trung bình (Kafka events)
- `animationSpeed=6` → Nhanh (real-time flows)

---

## 🛠️ Chỉnh Sửa Diagram

### Thêm service mới
1. Copy một service box hiện có (Ctrl+C → Ctrl+V)
2. Đổi tên service trong title
3. Kéo thả vào vị trí mong muốn
4. Nối với API Gateway (solid blue) và Kafka nếu cần

### Thay đổi màu edge
1. Click vào edge cần sửa
2. Right-click → Edit Style
3. Đổi `strokeColor=#...`

### Export diagram
- **PNG**: File → Export as → PNG (300 DPI cho chất lượng cao)
- **SVG**: File → Export as → SVG (vector, phù hợp in ấn)
- **PDF**: File → Print → Save as PDF

---

## 🔗 Tài Nguyên

| Tài liệu | Link |
|----------|------|
| Architecture Overview | [tourism_architecture.md.resolved](./tourism_architecture.md.resolved) |
| Master Plan 4 Weeks | [master_plan_4_weeks.md.resolved](./master_plan_4_weeks.md.resolved) |
| draw.io Documentation | [drawio-diagrams.github.io](https://drawio-diagrams.github.io) |
| Mermaid Diagrams | [architecture_diagram.md](./architecture_diagram.md) |
