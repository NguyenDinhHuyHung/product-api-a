# PRODUCT API — HƯỚNG DẪN THỰC HÀNH 1

RESTful API quản lý sản phẩm sử dụng Node.js, Express, MongoDB, Docker Compose, Jest và GitHub Actions.

## Yêu cầu trước khi thực hành

- Git
- Docker Desktop đang chạy
- Postman
- Tài khoản GitHub; tài khoản Docker Hub nếu thực hiện bước CD

Mở PowerShell tại thư mục gốc dự án rồi kiểm tra:

```powershell
git --version
docker --version
docker compose version
```

| Thành phần | Địa chỉ |
|---|---|
| Product API | `http://localhost:3000` |
| Health check | `http://localhost:3000/health` |
| MongoDB từ máy host | `localhost:27018` |
| MongoDB trong Docker network | `nammongodb:27017` |
| Database chính | `productdb` |

---

## 1. Kiểm tra `.gitignore` và `.env` trên GitHub — 1 điểm

### Bước 1.1: Kiểm tra `.gitignore`

File `.gitignore` phải có tối thiểu:

```gitignore
node_modules/
.env
.env.local
coverage/
```

### Bước 1.2: Tạo `.env` trên máy local

Tạo file `.env` ở thư mục gốc với nội dung:

```env
PORT=3000
NODE_ENV=development
MONGO_PORT=27018
MONGO_INITDB_ROOT_USERNAME=admin
MONGO_INITDB_ROOT_PASSWORD=admin123
MONGO_URI=mongodb://admin:admin123@localhost:27018/productdb?authSource=admin
DOCKERHUB_USERNAME=your-dockerhub-username
```

Thay `your-dockerhub-username` bằng tài khoản Docker Hub. Không đưa mật khẩu thật hoặc file `.env` lên GitHub.

### Bước 1.3: Chứng minh `.env` đang được bỏ qua

```powershell
git check-ignore -v .env
git ls-files .env
```

Kết quả mong đợi:

- Lệnh đầu hiển thị quy tắc `.env` trong `.gitignore`.
- Lệnh thứ hai không hiển thị gì, nghĩa là `.env` không được Git theo dõi.

Trên GitHub, mở repository và chứng minh có file `.gitignore` nhưng không có file `.env`.

> Nếu `.env` từng được commit, chạy `git rm --cached .env` rồi commit lại. Lệnh này chỉ ngừng theo dõi, không xóa file local.

---

## 2. Cập nhật mã nguồn và chạy Docker Compose — 1 điểm

### Bước 2.1: Lấy mã nguồn mới nhất

```powershell
git switch main
git pull origin main
```

Nếu đang có thay đổi local chưa commit, hãy commit hoặc lưu lại trước khi chạy `git pull`.

### Bước 2.2: Kiểm tra cấu hình Compose

```powershell
docker compose config
```

Lệnh phải kết thúc mà không báo lỗi YAML hoặc thiếu cấu hình.

### Bước 2.3: Build và khởi động hệ thống

```powershell
docker compose up -d --build
```

Compose sẽ:

1. Tạo network `product-network`.
2. Tạo volume lưu dữ liệu MongoDB.
3. Chạy MongoDB 7 trong container `nammongodb`.
4. Chờ MongoDB healthy rồi chạy container `product-api`.

Nếu muốn xem trực tiếp quá trình khởi động:

```powershell
docker compose up --build
```

Nhấn `Ctrl+C` để dừng chế độ theo dõi; sau đó có thể chạy lại bằng `docker compose up -d`.

---

## 3. System health check — 1 điểm

### Bước 3.1: Gọi endpoint health check

## nên dùng postman GET http://localhost:3000/health
```powershell
curl.exe -i http://localhost:3000/health
```

Kết quả đúng là HTTP `200 OK`, ví dụ:

```json
{
  "status": "OK",
  "service": "product-api",
  "database": "connected",
  "dbReadyState": 1,
  "uptime": 30.25,
  "timestamp": "2026-10-08T03:00:00.000Z"
}
```

`dbReadyState: 1` nghĩa là Mongoose đã kết nối MongoDB.

### Bước 3.2: Kiểm tra health status của Docker

```powershell
docker inspect --format "{{.State.Health.Status}}" nammongodb
docker inspect --format "{{.State.Health.Status}}" product-api
```

Cả hai lệnh phải trả về `healthy`. Nếu API chưa healthy ngay, đợi khoảng 10–30 giây rồi kiểm tra lại.

---

## 4. CLI: Kiểm tra danh sách container trong Docker Engine — 1 điểm

```powershell
docker ps
```



Cần chứng minh có hai container:

| Container | Image | Port mong đợi | Trạng thái |
|---|---|---|---|
| `nammongodb` | `mongo:7` | `27018 -> 27017` | Up, healthy |
| `product-api` | `product-api-*` | `3000 -> 3000` | Up, healthy |

Có thể trình bày gọn bằng:

```powershell
docker ps --format "table {{.Names}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}"
```

---

## 5. Test Postman API thêm sản phẩm — 1 điểm

Trong Postman, tạo request:

- Method: `POST`
- URL: `http://localhost:3000/api/products`
- **Headers**: `Content-Type` = `application/json`
- **Body** → **raw** → chọn **JSON**

Body:

```json
{
  "pid": "P001",
  "pname": "Laptop Dell XPS 15",
  "price": 1500,
  "quantity": 10
}
```

Bấm **Send**. Kết quả mong đợi:

- HTTP `201 Created`.
- Response có `_id`, `pid`, `pname`, `price`, `quantity`, `createdAt`, `updatedAt`.

Ví dụ:

```json
{
  "pid": "P001",
  "pname": "Laptop Dell XPS 15",
  "price": 1500,
  "quantity": 10,
  "_id": "...",
  "createdAt": "...",
  "updatedAt": "..."
}
```

Nếu nhận lỗi `Product with pid 'P001' already exists`, đổi `pid` thành `P002` hoặc xóa dữ liệu cũ trước khi gửi lại.

Kiểm tra danh sách sản phẩm bằng:

```text
GET http://localhost:3000/api/products
```

---

## 6. CLI: Kiểm tra dữ liệu trong MongoDB container — 1 điểm

### Cách 1: Mở Mongo Shell tương tác

```powershell
docker exec -it nammongodb mongosh -u admin -p admin123 --authenticationDatabase admin productdb
```

Trong `mongosh`, chạy:

```javascript

db.products.find()
```

Gõ `exit` để thoát:

```javascript
exit
```

### Cách 2: Kiểm tra bằng một lệnh CLI

```powershell
docker exec nammongodb mongosh -u admin -p admin123 --authenticationDatabase admin productdb --quiet --eval "db.products.find().forEach(p => printjson(p))"
```

Kết quả phải có sản phẩm vừa thêm từ Postman, chẳng hạn `pid: 'P001'`.

> Nếu đã đổi tài khoản hoặc mật khẩu trong `.env`, thay `admin` và `admin123` bằng giá trị tương ứng.

---

## 7. CLI: Kiểm tra log của `product-api` container — 1 điểm

docker logs product-api-container

## 8. CLI: Test script thực hiện CI/CD — 1 điểm
## git add . / git commit -m"" / git push
Project có các script trong `package.json`:

```json
{
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js",
    "test:unit": "jest tests/simple.test.js --runInBand --forceExit --verbose",
    "test": "jest tests/product.test.js --runInBand --forceExit --detectOpenHandles --verbose",
    "test:all": "jest --runInBand --forceExit --detectOpenHandles --verbose"
  }
}
```

### Bước 8.1: Cài dependency

```powershell
npm ci
```

### Bước 8.2: Chạy integration test với MongoDB trong Docker Compose

Đảm bảo `nammongodb` đang chạy, sau đó:

```powershell
$env:NODE_ENV = "test"
$env:MONGO_URI = "mongodb://admin:admin123@localhost:27018/productdb_test?authSource=admin"
npm test
```

Xóa biến môi trường khỏi phiên PowerShell sau khi test:

```powershell
Remove-Item Env:NODE_ENV
Remove-Item Env:MONGO_URI
```

`tests/product.test.js` thực hiện 14 integration test:

- CREATE: tạo mới, trùng `pid`, thiếu trường, giá âm.
- READ: lấy danh sách, lấy theo `pid`, kiểm tra `404`.
- UPDATE: cập nhật thành công và các trường hợp lỗi.
- DELETE: xóa thành công và kiểm tra dữ liệu không còn tồn tại.
- HEALTH: API và MongoDB phải hoạt động.

Kết quả mong đợi:

```text
Test Suites: 1 passed, 1 total
Tests:       14 passed, 14 total
```

### Bước 8.3: Kiểm tra workflow CI/CD

Workflow: `.github/workflows/test-productci-prod.yml`.

```text
Push/Pull Request vào main
          |
          v
MongoDB service -> npm ci -> npm test
          |
          v
Build Docker image -> chạy container -> kiểm tra /health
          |
          v
CI thành công trên main -> build và push image lên Docker Hub
```

Để CD push được image:

1. Vào GitHub repository → **Settings** → **Secrets and variables** → **Actions**.
2. Tạo secret `DOCKERHUB_USERNAME` chứa username Docker Hub.
3. Tạo secret `DOCKERHUB_TOKEN` chứa access token Docker Hub.

Commit và push để kích hoạt pipeline:

```powershell
git add README.md
git commit -m "docs: add Practical 1 step-by-step guide"
git push origin main
```

Mở tab **Actions** và kiểm tra workflow **Product API - Production CI/CD Pipeline**. Job `cd-dockerhub` chỉ chạy sau khi `ci-test` thành công và workflow đang ở nhánh `main`.

---

## Bảng kiểm trước khi giảng viên chấm

| STT | Nội dung | Minh chứng cần trình bày | Điểm |
|---:|---|---|---:|
| 1 | `.gitignore` và `.env` | GitHub có `.gitignore`, không có `.env`; chạy `git check-ignore` | 1 |
| 2 | Docker Compose | `docker compose up -d --build` thành công | 1 |
| 3 | System health check | `/health` trả HTTP 200 và database `connected` | 1 |
| 4 | Danh sách container | `docker ps` có `nammongodb` và `product-api` healthy | 1 |
| 5 | Postman thêm sản phẩm | POST trả HTTP 201 và response có `_id` | 1 |
| 6 | Dữ liệu MongoDB | `db.products.find()` có sản phẩm vừa thêm | 1 |
| 7 | Log API | `docker logs product-api` cho thấy API và DB hoạt động | 1 |
| 8 | Test CI/CD | `npm test` pass 14 test và GitHub Actions thành công | 1 |

## Dừng hệ thống sau khi thực hành

Dừng và xóa container/network nhưng giữ dữ liệu MongoDB:

```powershell
docker compose down
```

Chỉ khi muốn xóa luôn volume và toàn bộ dữ liệu database:

```powershell
docker compose down -v
```

> `docker compose down -v` xóa dữ liệu MongoDB trong volume và không thể khôi phục bằng Compose.
