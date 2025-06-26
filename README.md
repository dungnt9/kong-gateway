# Phone Store Gateway

Kong API Gateway cho Phone Store Microservices Demo.

## 🏗️ Kiến trúc

```
Client Request
      ↓
Kong Gateway :8000
      ↓
┌─────────────────┬─────────────────┐
│  ProductService │   OrderService  │
│   localhost:6001│   localhost:6002│
└─────────────────┴─────────────────┘
```

**Kong Gateway** đóng vai trò **API Gateway** - điểm vào duy nhất cho tất cả requests, sau đó route đến các microservices tương ứng.

## 🚀 Chạy Kong

```cmd
docker-compose up -d
```

1. Pull Kong và PostgreSQL images (nếu chưa có)
2. Start PostgreSQL database
3. Chạy Kong migrations (setup DB schema)
4. Start Kong Gateway

## 📡 Endpoints

- **Gateway**: http://localhost:8000 (Client access point)
- **Kong Admin**: http://localhost:8001 (Management API)
- **PostgreSQL**: localhost:5432 (Kong metadata only)

## ⚙️ Setup Routes

Sau khi Kong chạy, setup routes để kết nối services:

### ProductService Route:

```cmd
# Tạo service definition
curl -X POST http://localhost:8001/services/ \
  --data "name=product-service" \
  --data "url=http://host.docker.internal:6001"

# Tạo route rule
curl -X POST http://localhost:8001/services/product-service/routes \
  --data "hosts[]=localhost" \
  --data "paths[]=/api/products" \
  --data "strip_path=false"
```

### OrderService Route:

```cmd
# Tạo service definition
curl -X POST http://localhost:8001/services/ \
  --data "name=order-service" \
  --data "url=http://host.docker.internal:6002"

# Tạo route rule
curl -X POST http://localhost:8001/services/order-service/routes \
  --data "hosts[]=localhost" \
  --data "paths[]=/api/orders" \
  --data "strip_path=false"
```

**Giải thích:**

- `host.docker.internal`: Kong container kết nối đến localhost của host machine
- `strip_path=false`: Giữ nguyên path khi forward request
- `hosts[]=localhost`: Chỉ accept requests từ localhost

## 🔄 Request Flow

### Ví dụ: Client gọi ProductService qua Gateway

```
1. Client → http://localhost:8000/api/products
2. Kong nhận request, check routing rules
3. Kong tìm thấy rule: /api/products → product-service
4. Kong forward → http://host.docker.internal:6001/api/products
5. ProductService xử lý và trả response
6. Kong trả response về Client
```

### Service Communication Flow (OrderService gọi ProductService):

```
1. Client → Kong:8000/api/orders (POST create order)
2. Kong → OrderService:6002/api/orders
3. OrderService → ProductService:6001/api/products/1 (get product info)
4. ProductService → response product data
5. OrderService → tạo order với cached product info
6. OrderService → response về Kong
7. Kong → response về Client
```

## 🧪 Test Demo

### 1. Test direct services (đảm bảo services đang chạy):

```cmd
curl http://localhost:6001/api/products
curl http://localhost:6002/api/orders
```

### 2. Test qua Gateway:

```cmd
curl http://localhost:8000/api/products
curl http://localhost:8000/api/orders
```

### 3. Demo Service Communication:

**Tạo Product:**

```cmd
curl -X POST http://localhost:8000/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "iPhone 15 Pro",
    "brand": "Apple",
    "price": 1199.99,
    "description": "Latest iPhone with advanced features",
    "stock": 50
  }'
```

**Tạo Order (OrderService sẽ gọi ProductService):**

```cmd
curl -X POST http://localhost:8000/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerName": "John Doe",
    "customerEmail": "john@example.com",
    "productId": 1,
    "quantity": 2
  }'
```

**Kiểm tra Order đã có thông tin Product cached:**

```cmd
curl http://localhost:8000/api/orders/1
```

## 🔍 Monitoring & Debug

### Check Status:

```cmd
# Container status
docker-compose ps

# Kong health
curl http://localhost:8001/status

# View services & routes configured
curl http://localhost:8001/services
curl http://localhost:8001/routes
```

### View Logs:

```cmd
# All containers
docker-compose logs

# Kong only
docker-compose logs kong

# Database only
docker-compose logs kong-database
```

## 💡 Tại sao dùng API Gateway?

### ✅ Lợi ích:

1. **Single Entry Point**: Client chỉ cần biết 1 endpoint thay vì nhiều services
2. **Service Discovery**: Gateway biết services ở đâu, client không cần biết
3. **Load Balancing**: Có thể scale multiple instances của cùng 1 service
4. **Security**: Authentication, rate limiting tập trung
5. **Monitoring**: Logs, metrics tập trung cho tất cả requests
6. **API Versioning**: Dễ dàng manage versions và backward compatibility

## 🛠️ Components

### Kong Gateway:

- **Port 8000**: Proxy port (client requests)
- **Port 8001**: Admin API (configuration)
- **Database**: PostgreSQL (lưu configuration, không phải business data)

### Services đã có:

- **ProductService**: localhost:6001 (CRUD products)
- **OrderService**: localhost:6002 (CRUD orders + gọi ProductService)

## 🚨 Troubleshooting

### Kong không start:

```cmd
# Check logs
docker-compose logs kong

# Restart
docker-compose restart kong
```

### Routes không work:

```cmd
# Verify routes exist
curl http://localhost:8001/routes

# Re-setup routes
# (copy commands từ section Setup Routes)
```

### Service connection failed:

```cmd
# Test direct service
curl http://localhost:6001/api/products

# Check if services running
netstat -an | findstr :6001
netstat -an | findstr :6002
```

## 🛑 Stop Gateway

```cmd
# Stop containers
docker-compose down

# Stop và xóa volumes (clean reset)
docker-compose down -v
```
