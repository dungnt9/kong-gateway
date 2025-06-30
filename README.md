# Phone Store Gateway

Kong API Gateway cho Phone Store Microservices Demo với **DB-less mode** (declarative config).

## 🏗️ Kiến trúc

```
Client Request
      ↓
Kong Gateway :8000 (DB-less mode)
      ↓
┌─────────────────┬─────────────────┐
│  ProductService │   OrderService  │
│   localhost:6001│   localhost:6002│
└─────────────────┴─────────────────┘
```

**Kong Gateway** chạy **DB-less mode** với configuration từ file `kong.yml` - không cần PostgreSQL cho routing.

## 🚀 Chạy Kong

```cmd
docker-compose up -d
```

**Kong sẽ tự động load config từ `kong.yml`** - không cần setup routes manual!

## 📡 Endpoints

- **Gateway**: http://localhost:8000 (Client access point)
- **Kong Admin**: http://localhost:8001 (Management API - read-only trong DB-less mode)

## 🛤️ Routes (auto load từ kong.yml)

Tất cả routes đã được định nghĩa trong `kong.yml`:

### ProductService:

- **URL**: `http://localhost:8000/product-service/*`
- **Routes to**: ProductService :6001
- **Strip Path**: true (bỏ `/product-service` khi forward)

### OrderService:

- **URL**: `http://localhost:8000/order-service/*`
- **Routes to**: OrderService :6002
- **Strip Path**: true (bỏ `/order-service` khi forward)

## 🔄 Request Flow

### ProductService qua Gateway:

```
Client → http://localhost:8000/product-service/api/products
Kong → http://host.docker.internal:6001/api/products (strip_path=true)
```

### OrderService qua Gateway:

```
Client → http://localhost:8000/order-service/api/orders
Kong → http://host.docker.internal:6002/api/orders (strip_path=true)
```

### Service Communication:

```
1. Client → Kong:8000/order-service/api/orders (POST create order)
2. Kong → OrderService:6002/api/orders
3. OrderService → ProductService:6001/api/products/1 (get product info)
4. ProductService → response product data
5. OrderService → tạo order với cached product info
6. OrderService → response về Kong
7. Kong → response về Client
```

## 🧪 Test

### Direct services:

```cmd
curl http://localhost:6001/api/products
curl http://localhost:6002/api/orders
```

### Qua Gateway:

```cmd
curl http://localhost:8000/product-service/api/products
curl http://localhost:8000/order-service/api/orders
```

### Demo Service Communication:

**Tạo Product:**

```cmd
curl -X POST http://localhost:8000/product-service/api/products \
  -H "Content-Type: application/json" \
  -d '{
    "name": "iPhone 15 Pro",
    "brand": "Apple",
    "price": 1199.99,
    "description": "Latest iPhone",
    "stock": 50
  }'
```

**Tạo Order (OrderService sẽ gọi ProductService):**

```cmd
curl -X POST http://localhost:8000/order-service/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerName": "John Doe",
    "customerEmail": "john@example.com",
    "productId": 1,
    "quantity": 2
  }'
```

## 🔍 Monitoring

```cmd
# Container status
docker-compose ps

# Kong health
curl http://localhost:8001/status

# View loaded config
curl http://localhost:8001/config
```

## ⚙️ DB-less Mode

### ✅ Lợi ích:

- **No Database**: Không cần PostgreSQL cho routing
- **Config as Code**: Tất cả trong `kong.yml`, có thể version control
- **Fast Startup**: Start nhanh, ít component
- **Simple**: Dễ maintain và deploy
- **Auto Load**: Routes tự động load từ file, không cần manual setup

### Update Config:

1. Edit `kong.yml`
2. `docker-compose restart kong`
3. Config tự động reload

## 🛑 Stop

```cmd
docker-compose down
```
