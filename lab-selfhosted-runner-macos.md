# ใบงาน: Self-Hosted Runner กับ Booking App (macOS)

## 🍎 สำหรับระบบปฏิบัติการ macOS

> 📌 **ข้อกำหนดเบื้องต้น:** ติดตั้ง **Docker Desktop** และ **Git** ไว้แล้ว

---

## วัตถุประสงค์

1. อธิบายหลักการทำงานของ Self-Hosted Runner แบบ Pull-based Model ได้
2. ติดตั้งและกำหนดค่า Self-Hosted Runner บนเครื่อง macOS ได้
3. สร้าง CI/CD Pipeline สำหรับ Deploy ระบบจองห้องพักลงบนเครื่อง Local ด้วย Docker Compose ได้
4. ตั้งค่า Nginx Reverse Proxy สำหรับ Backend API ได้
5. ตรวจสอบและ Monitor การ Deploy ด้วย Script ได้

---

## ทฤษฎีที่เกี่ยวข้อง

### Self-Hosted Runner คืออะไร

Self-Hosted Runner คือเครื่องที่เราติดตั้งและดูแลเอง ทำหน้าที่รัน GitHub Actions workflows โดยใช้กลไก **Pull-based (Polling)** คือ Runner เป็นฝ่ายดึงงานจาก GitHub แทนที่ GitHub จะส่งงานมา

### จุดเด่นของ Pull-based Model

- Runner เป็นฝ่าย **ดึง (Pull)** งาน ไม่ใช่ GitHub ส่ง (Push) มา
- ไม่ต้องเปิด Inbound Port ให้ภายนอกเข้าถึง
- ไม่ต้องมี Static IP Address
- ทำงานได้แม้อยู่หลัง Firewall หรือ NAT

### สถาปัตยกรรมการทำงาน

```
┌─────────────────────────────────────────────────────────────┐
│                    GitHub Cloud Platform                    │
│  ┌────────────┐   ┌───────────────┐   ┌─────────────────┐   │
│  │ Repository │──>│    Workflow   │──>│   Job Queue     │   │
│  │   (Code)   │   │  (Actions)    │   │ (Pending Jobs)  │   │
│  └────────────┘   └───────────────┘   └─────────────────┘   │
│                                                ▲            │
└────────────────────────────────────────────────┼────────────┘
                                                 │
                         Firewall (No Inbound)   │
                         ════════════════════════│═══
                                                 │ HTTPS Polling
                              1. "Any jobs?"     │ (Outbound Only)
                          ┌──────────────────────┘
                          │  2. Response: Job / "No jobs"
                          ▼
                  ┌─────────────────────┐
                  │   Self-Hosted       │ ← รันบน macOS ของคุณ
                  │      Runner         │
                  └─────────────────────┘
                          │
                          │ 3. Clone repo
                          │ 4. Execute steps
                          │ 5. Report status
                          ▼
                  ┌─────────────────────────────┐
                  │  Docker Compose (Local)     │
                  │  ├── PostgreSQL (db)        │
                  │  ├── Backend (Express API)  │
                  │  └── Nginx (Reverse Proxy)  │
                  └─────────────────────────────┘
```

---

## การปฏิบัติการทดลอง

---

### ส่วนที่ 1: เตรียม Repository

#### 1.1 Fork Repository

เนื่องจาก code ถูกเตรียมไว้แล้ว ให้ Fork มาไว้ใน Account ของตัวเอง

1. ไปที่ `https://github.com/surachai-p/booking-app-demo-2025`
2. คลิกปุ่ม **Fork** (มุมขวาบน)
3. เลือก Account ของตัวเองเป็น Owner
4. คงชื่อ: `booking-app-demo-2025`
5. คลิก **Create fork**

> ⚠️ **ความปลอดภัย:** Self-Hosted Runner จะรัน code บนเครื่องของเรา ควรใช้กับ Private Repository เท่านั้น  
> หลัง Fork ให้ไปที่ **Settings → General → Danger Zone → Change visibility → Make private**

#### 1.2 Clone Repository มายัง Local

เปิด **Terminal** แล้วรัน:

```bash
# Clone repository ที่ fork มา
git clone https://github.com/YOUR_USERNAME/booking-app-demo-2025.git

# เข้าไปในโฟลเดอร์
cd booking-app-demo-2025
```

#### 1.3 ทำความเข้าใจโครงสร้างโปรเจกต์

```bash
ls -la
```

โครงสร้างที่ควรเห็น:

```
booking-app-demo-2025/
├── frontend/          ← React + Vite (ยังไม่ใช้ในใบงานนี้)
├── backend/           ← Node.js + Express + Prisma
│   ├── prisma/
│   ├── routes/
│   ├── server.js
│   ├── Dockerfile
│   ├── docker-entrypoint.sh
│   └── docker-compose.yml   ← ใช้ PostgreSQL local เท่านั้น
└── .github/
    └── workflows/           ← เราจะสร้าง workflow ใหม่ที่นี่
```

> **หมายเหตุ:** `backend/docker-compose.yml` เดิมใช้สำหรับ Dev เท่านั้น (database เดียว)  
> เราจะสร้าง `docker-compose.selfhosted.yml` ที่ root เพื่อ deploy ทั้ง backend + database + nginx

---

### ส่วนที่ 2: สร้างไฟล์ Configuration สำหรับ Self-Hosted Deployment

#### 2.1 สร้าง docker-compose.selfhosted.yml ที่ root ของโปรเจกต์

```bash
# ยืนยันว่าอยู่ที่ root ของโปรเจกต์
pwd
# ควรได้ .../booking-app-demo-2025
```

สร้างไฟล์ `docker-compose.selfhosted.yml`:

```yaml
services:
  # ──────────────────────────────────────
  # PostgreSQL Database
  # ──────────────────────────────────────
  db:
    image: postgres:16-alpine
    container_name: booking-selfhosted-db
    restart: unless-stopped
    environment:
      POSTGRES_DB: booking_app
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - booking_selfhosted_data:/var/lib/postgresql/data
    networks:
      - booking-selfhosted-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d booking_app"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s

  # ──────────────────────────────────────
  # Backend API (Node.js + Express)
  # ──────────────────────────────────────
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: booking-selfhosted-backend
    restart: unless-stopped
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/booking_app
      JWT_SECRET: selfhosted-lab-secret-key-2025
      NODE_ENV: production
      PORT: 3001
    depends_on:
      db:
        condition: service_healthy
    networks:
      - booking-selfhosted-net
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  # ──────────────────────────────────────
  # Nginx Reverse Proxy
  # ──────────────────────────────────────
  nginx:
    image: nginx:alpine
    container_name: booking-selfhosted-nginx
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./nginx.selfhosted.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - backend
    networks:
      - booking-selfhosted-net
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:80/api/rooms"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 15s

volumes:
  booking_selfhosted_data:
    name: booking_selfhosted_data

networks:
  booking-selfhosted-net:
    driver: bridge
    name: booking-selfhosted-net
```

#### 2.2 สร้าง nginx.selfhosted.conf ที่ root ของโปรเจกต์

สร้างไฟล์ `nginx.selfhosted.conf`:

```nginx
events {
    worker_connections 1024;
}

http {
    sendfile on;
    keepalive_timeout 65;

    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    gzip on;
    gzip_types application/json text/plain;

    upstream booking_backend {
        server backend:3001 max_fails=3 fail_timeout=30s;
    }

    server {
        listen 80;
        server_name _;

        # Security Headers
        add_header X-Frame-Options "SAMEORIGIN" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;

        # ส่งทุก request ไปยัง backend API
        location / {
            proxy_pass         http://booking_backend;
            proxy_http_version 1.1;
            proxy_set_header   Host              $host;
            proxy_set_header   X-Real-IP         $remote_addr;
            proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Proto $scheme;
            proxy_connect_timeout 60s;
            proxy_send_timeout    60s;
            proxy_read_timeout    60s;
        }
    }
}
```

#### 2.3 สร้าง GitHub Actions Workflow

สร้างโฟลเดอร์และไฟล์:

```bash
mkdir -p .github/workflows
touch .github/workflows/deploy-selfhosted.yml
```

เปิดไฟล์ `.github/workflows/deploy-selfhosted.yml` แล้วใส่เนื้อหานี้:

```yaml
name: 🚀 Deploy Booking App (Self-Hosted)

on:
  push:
    branches:
      - main
  workflow_dispatch:

env:
  COMPOSE_FILE: docker-compose.selfhosted.yml

jobs:
  deploy:
    name: 🚀 Deploy to Local Machine
    runs-on: self-hosted

    steps:
      # ──────────────────────────────────────────────
      # Step 1: Checkout Code
      # ──────────────────────────────────────────────
      - name: 📥 Checkout Code
        uses: actions/checkout@v4

      # ──────────────────────────────────────────────
      # Step 2: แสดงข้อมูล Deployment
      # ──────────────────────────────────────────────
      - name: 📊 Deployment Info
        run: |
          echo "════════════════════════════════════════"
          echo "🚀 Booking App — Self-Hosted Deployment"
          echo "════════════════════════════════════════"
          echo "📦 Run Number : ${{ github.run_number }}"
          echo "🌿 Branch     : ${{ github.ref_name }}"
          echo "👤 Author     : ${{ github.actor }}"
          echo "💬 Commit     : ${{ github.event.head_commit.message }}"
          echo "🔗 SHA        : ${{ github.sha }}"
          echo "────────────────────────────────────────"
          echo "🖥️  OS         : $(uname -s) $(uname -r)"
          echo "🐳 Docker     : $(docker --version)"
          echo "════════════════════════════════════════"

      # ──────────────────────────────────────────────
      # Step 3: ตรวจสอบไฟล์ที่จำเป็น
      # ──────────────────────────────────────────────
      - name: 🔍 Verify Required Files
        run: |
          echo "🔍 Checking required files..."

          FILES=(
            "docker-compose.selfhosted.yml"
            "nginx.selfhosted.conf"
            "backend/Dockerfile"
            "backend/package.json"
            "backend/package-lock.json"
          )

          ALL_OK=true
          for f in "${FILES[@]}"; do
            if [ -f "$f" ]; then
              echo "  ✅ $f"
            else
              echo "  ❌ $f — NOT FOUND"
              ALL_OK=false
            fi
          done

          if [ "$ALL_OK" = false ]; then
            echo "❌ Required files missing. Aborting."
            exit 1
          fi
          echo "✅ All required files present."

      # ──────────────────────────────────────────────
      # Step 4: หยุด services เดิม
      # ──────────────────────────────────────────────
      - name: 🛑 Stop Existing Services
        run: |
          echo "🛑 Stopping existing services..."
          docker compose -f $COMPOSE_FILE down --remove-orphans || echo "  No services running."
          
          echo "🧹 Removing old images (keep latest)..."
          docker image prune -f || true

      # ──────────────────────────────────────────────
      # Step 5: Build Docker Image
      # ──────────────────────────────────────────────
      - name: 🔨 Build Docker Images
        run: |
          echo "🔨 Building backend image..."
          docker compose -f $COMPOSE_FILE build --no-cache backend
          
          echo "✅ Build complete."
          docker images | grep booking

      # ──────────────────────────────────────────────
      # Step 6: Start Services
      # ──────────────────────────────────────────────
      - name: 🚀 Start Services
        run: |
          echo "🚀 Starting all services..."
          docker compose -f $COMPOSE_FILE up -d
          
          echo "⏳ Waiting for services to initialize (30s)..."
          sleep 30

      # ──────────────────────────────────────────────
      # Step 7: Health Check
      # ──────────────────────────────────────────────
      - name: 🏥 Health Check
        run: |
          echo "🏥 Checking container health..."

          # ตรวจ containers
          for NAME in booking-selfhosted-db booking-selfhosted-backend booking-selfhosted-nginx; do
            STATUS=$(docker inspect --format='{{.State.Status}}' $NAME 2>/dev/null || echo "not found")
            if [ "$STATUS" = "running" ]; then
              echo "  ✅ $NAME: running"
            else
              echo "  ❌ $NAME: $STATUS"
              docker logs $NAME --tail 30 || true
              exit 1
            fi
          done

          # ตรวจ API Endpoint
          echo ""
          echo "🧪 Testing API endpoint..."
          MAX=20
          for i in $(seq 1 $MAX); do
            HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/api/rooms)
            if [ "$HTTP_CODE" = "200" ]; then
              echo "  ✅ GET /api/rooms → HTTP $HTTP_CODE (attempt $i/$MAX)"
              break
            fi
            if [ "$i" -eq "$MAX" ]; then
              echo "  ❌ Health check failed after $MAX attempts (last status: $HTTP_CODE)"
              docker compose -f $COMPOSE_FILE logs --tail 50
              exit 1
            fi
            echo "  ⏳ Attempt $i/$MAX — HTTP $HTTP_CODE — waiting..."
            sleep 3
          done

      # ──────────────────────────────────────────────
      # Step 8: Test API Endpoints
      # ──────────────────────────────────────────────
      - name: 🧪 Test API Endpoints
        run: |
          echo "🧪 Testing Booking App API..."

          # ทดสอบ GET /api/rooms
          echo "→ GET /api/rooms"
          ROOMS=$(curl -sf http://localhost:8080/api/rooms)
          echo "$ROOMS"
          echo "  ✅ Rooms endpoint working"

          # ทดสอบ Login
          echo ""
          echo "→ POST /api/login"
          LOGIN=$(curl -sf -X POST http://localhost:8080/api/login \
            -H "Content-Type: application/json" \
            -d '{"username":"admin","password":"admin123"}')
          echo "$LOGIN"

          if echo "$LOGIN" | grep -q "token"; then
            echo "  ✅ Login endpoint working"
          else
            echo "  ⚠️  Login may not have token (check admin credentials)"
          fi

      # ──────────────────────────────────────────────
      # Step 9: แสดงสถานะ Containers
      # ──────────────────────────────────────────────
      - name: 📊 Display Status
        run: |
          echo "════════════════════════════════════════"
          echo "📊 Container Status"
          echo "════════════════════════════════════════"
          docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" \
            | grep -E "(NAMES|booking-selfhosted)"
          echo ""
          echo "💾 Resource Usage:"
          docker stats --no-stream --format "  {{.Container}}: CPU {{.CPUPerc}}, Mem {{.MemUsage}}" \
            | grep booking-selfhosted || true
          echo "════════════════════════════════════════"

      # ──────────────────────────────────────────────
      # Step 10: Logs (always runs)
      # ──────────────────────────────────────────────
      - name: 📝 Show Logs
        if: always()
        run: |
          echo "════════════════════════════════════════"
          echo "📝 Backend Logs (last 30 lines)"
          echo "════════════════════════════════════════"
          docker logs booking-selfhosted-backend --tail 30 || echo "No logs."
          echo ""
          echo "════════════════════════════════════════"
          echo "📝 Nginx Logs (last 10 lines)"
          echo "════════════════════════════════════════"
          docker logs booking-selfhosted-nginx --tail 10 || echo "No logs."

      # ──────────────────────────────────────────────
      # Step 11: Summary
      # ──────────────────────────────────────────────
      - name: 🎉 Deployment Summary
        if: success()
        run: |
          echo "════════════════════════════════════════"
          echo "✅ Deployment Successful!"
          echo "════════════════════════════════════════"
          echo "🔗 API Base URL : http://localhost:8080"
          echo "📋 Endpoints:"
          echo "   GET  http://localhost:8080/api/rooms"
          echo "   POST http://localhost:8080/api/login"
          echo "   GET  http://localhost:8080/api/bookings"
          echo "📅 Deployed     : $(date)"
          echo "👤 By           : ${{ github.actor }}"
          echo "════════════════════════════════════════"
```

#### 2.4 Commit และ Push ไฟล์ทั้งหมด

```bash
# ตรวจสอบไฟล์ที่สร้างใหม่
git status
```

ควรเห็น:
```
Untracked files:
  .github/workflows/deploy-selfhosted.yml
  docker-compose.selfhosted.yml
  nginx.selfhosted.conf
```

```bash
# เพิ่มไฟล์
git add docker-compose.selfhosted.yml nginx.selfhosted.conf .github/workflows/deploy-selfhosted.yml

# Commit
git commit -m "feat: add self-hosted runner deployment config"

# Push (ยังไม่ trigger workflow เพราะ runner ยังไม่ได้ติดตั้ง)
git push origin main
```

---

### ส่วนที่ 3: ติดตั้ง Self-Hosted Runner บน macOS

#### 3.1 ไปที่ Repository Settings

1. ไปที่ GitHub repository ที่ Fork มา
2. คลิก **Settings**
3. เมนูซ้าย → **Actions** → **Runners**
4. คลิก **New self-hosted runner**

#### 3.2 เลือก Operating System

เลือก **macOS** → เลือก Architecture ตามเครื่อง:
- **Intel Mac** → เลือก `x64`
- **Apple Silicon (M1/M2/M3)** → เลือก `arm64`

#### 3.3 Download และ Configure Runner

GitHub จะแสดงคำสั่ง Copy มาทำตามทีละขั้น โดย **เปิด Terminal** แล้วรัน:

**ขั้นที่ 1 — สร้างโฟลเดอร์และ Download**

```bash
# สร้างโฟลเดอร์ runner (ที่ home directory)
mkdir ~/actions-runner && cd ~/actions-runner

# ──────────────────────────────────────────────────────────────
# ⚠️ คัดลอกคำสั่ง curl ที่แสดงบน GitHub มาใช้ตรงๆ
#    เพราะ URL จะมี version และ checksum ล่าสุดอยู่แล้ว
# ──────────────────────────────────────────────────────────────
# ตัวอย่าง (อย่าคัดลอกตัวอย่างนี้ ให้ใช้จาก GitHub):
# curl -o actions-runner-osx-arm64-2.xxx.x.tar.gz -L \
#   https://github.com/actions/runner/releases/download/v2.xxx.x/actions-runner-osx-arm64-2.xxx.x.tar.gz
```

**ขั้นที่ 2 — Extract**

```bash
# คัดลอกคำสั่ง tar ที่แสดงบน GitHub มาใช้
# ตัวอย่าง:
# tar xzf ./actions-runner-osx-arm64-2.xxx.x.tar.gz
```

**ขั้นที่ 3 — Configure** (คัดลอกทั้งบรรทัดจาก GitHub เพราะมี Token ชั่วคราวอยู่)

```bash
# คำสั่งจาก GitHub มีรูปแบบนี้:
# ./config.sh --url https://github.com/YOUR_USERNAME/booking-app-demo-2025 --token AXXXXXXXXXXXXX
```

เมื่อระบบถามให้ตอบดังนี้:

```
Enter the name of the runner group [press Enter for Default]:
  → [Enter]

Enter the name of runner (e.g., Macbook):
  → my-mac-runner   (หรือชื่ออื่นตามต้องการ)

This runner will have the following labels: 'self-hosted', 'macOS', 'ARM64'
Enter any additional labels (ex. label-1,label-2) [press Enter to skip]:
  → [Enter]

Enter name of work folder [press Enter for _work]:
  → [Enter]
```

ผลลัพธ์ที่ควรเห็น:

```
√ Runner successfully added
√ Runner connection is good
```

#### 3.4 เริ่มต้น Runner

```bash
# รัน Runner แบบ Interactive (เหมาะสำหรับทดสอบ)
./run.sh
```

ผลลัพธ์ที่ควรเห็น:

```
√ Connected to GitHub

Current runner version: '2.xxx.x'
2025-xx-xx xx:xx:xxZ: Listening for Jobs
```

> 💡 **สำคัญ:** ปล่อยให้ Terminal นี้เปิดอยู่ตลอดการทดลอง Runner ต้องรันอยู่จึงจะรับงานได้

#### 3.5 ตรวจสอบ Runner บน GitHub

1. กลับไปที่ **Settings** → **Actions** → **Runners**
2. ควรเห็น Runner แสดงสถานะ **Idle** สีเขียว

### 📸 บันทึกรูปผลการทดลอง — Runner Status
```
บันทึกภาพหน้า Runners ให้เห็น:
- ชื่อ GitHub Account และ Repository
- Runner แสดงสถานะ "Idle" สีเขียว
- Labels: self-hosted, macOS, ARM64 (หรือ X64)
```

---

### ส่วนที่ 4: Trigger การ Deploy ครั้งแรก

#### 4.1 แก้ไขไฟล์ README เพื่อ Trigger Workflow

เปิด Terminal ใหม่ (ไม่ต้องปิด Terminal ที่รัน Runner) แล้วไปที่โฟลเดอร์โปรเจกต์:

```bash
cd ~/path/to/booking-app-demo-2025
```

แก้ไขไฟล์ `README.md` (หรือสร้าง `VERSION.txt`):

```bash
# สร้างไฟล์ VERSION.txt
echo "Self-Hosted Deploy - Run 1" > VERSION.txt

# Commit และ Push
git add VERSION.txt
git commit -m "test: trigger first self-hosted deployment"
git push origin main
```

#### 4.2 ติดตามการ Deploy

**1. ดูที่ Terminal ที่รัน Runner:**

```
2025-xx-xx xx:xx:xxZ: Running job: 🚀 Deploy to Local Machine
```

**2. ดูบน GitHub:**
- ไปที่ tab **Actions** ของ Repository
- คลิก Workflow run ล่าสุด "🚀 Deploy Booking App (Self-Hosted)"
- ดู log แบบ real-time

**3. หลัง Deploy เสร็จ ทดสอบ API:**

```bash
# ทดสอบ GET /api/rooms
curl -s http://localhost:8080/api/rooms | python3 -m json.tool

# ทดสอบ Login
curl -s -X POST http://localhost:8080/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'

# ดู containers ที่รันอยู่
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

**ผลลัพธ์ที่ควรเห็น (ตัวอย่าง rooms):**

```json
[
  {
    "id": 1,
    "name": "Deluxe Room",
    "type": "Deluxe",
    "capacity": 2,
    "price": 1500
  }
]
```

### 📸 บันทึกรูปผลการทดลอง — docker logs

```bash
docker logs booking-selfhosted-backend --tail 30
```

```
บันทึกภาพผลการรันคำสั่งนี้
```

---

### ส่วนที่ 5: ทดสอบ CI/CD — แก้ไขและ Push อีกครั้ง

#### 5.1 แก้ไขข้อมูลห้องพัก (ทดสอบ Pipeline)

เปิดไฟล์ `backend/prisma/seed.js` หรือสร้างไฟล์ `VERSION.txt` ใหม่:

```bash
# อัปเดต VERSION เพื่อ trigger workflow
echo "Self-Hosted Deploy - Run 2 - $(date)" > VERSION.txt

git add VERSION.txt
git commit -m "feat: second deployment test with self-hosted runner"
git push origin main
```

#### 5.2 ดู Runner รับงานใหม่

ดูที่ Terminal ที่รัน Runner จะเห็น:

```
2025-xx-xx xx:xx:xxZ: Job completed: success
2025-xx-xx xx:xx:xxZ: Listening for Jobs
...
2025-xx-xx xx:xx:xxZ: Running job: 🚀 Deploy to Local Machine
```

---

### ส่วนที่ 6: Monitoring Script

#### 6.1 สร้าง monitor.sh

สร้างไฟล์ `monitor.sh` ที่ root ของโปรเจกต์:

```bash
#!/bin/bash

SEP="════════════════════════════════════════"

echo "$SEP"
echo "🔍 Booking App — Deployment Monitor"
echo "$SEP"
echo ""

# ── Runner Status ──
echo "📊 Runner Status:"
if pgrep -f "Runner.Listener" > /dev/null 2>&1; then
  echo "  ✅ Self-Hosted Runner: Running"
else
  echo "  ❌ Self-Hosted Runner: Not Found"
fi
echo ""

# ── Container Status ──
echo "🐳 Container Status:"
for NAME in booking-selfhosted-db booking-selfhosted-backend booking-selfhosted-nginx; do
  STATUS=$(docker inspect --format='{{.State.Status}} (health: {{.State.Health.Status}})' $NAME 2>/dev/null || echo "not found")
  if [[ "$STATUS" == running* ]]; then
    echo "  ✅ $NAME: $STATUS"
  else
    echo "  ❌ $NAME: $STATUS"
  fi
done
echo ""

# ── API Endpoints ──
echo "🌐 API Endpoint Status:"
ROOMS_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/api/rooms 2>/dev/null)
if [ "$ROOMS_CODE" = "200" ]; then
  echo "  ✅ GET /api/rooms  → HTTP $ROOMS_CODE"
else
  echo "  ❌ GET /api/rooms  → HTTP $ROOMS_CODE"
fi

LOGIN_CODE=$(curl -s -o /dev/null -w "%{http_code}" -X POST http://localhost:8080/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' 2>/dev/null)
if [ "$LOGIN_CODE" = "200" ]; then
  echo "  ✅ POST /api/login → HTTP $LOGIN_CODE"
else
  echo "  ⚠️  POST /api/login → HTTP $LOGIN_CODE"
fi
echo ""

# ── Resource Usage ──
echo "💾 Resource Usage:"
docker stats --no-stream \
  --format "  {{.Container}}: CPU {{.CPUPerc}}, Mem {{.MemUsage}}" \
  2>/dev/null | grep booking-selfhosted || echo "  (No containers running)"
echo ""

echo "$SEP"
echo "🕐 Last updated: $(date '+%Y-%m-%d %H:%M:%S')"
echo "$SEP"
```

#### 6.2 ให้สิทธิ์และรัน

```bash
chmod +x monitor.sh

# รันครั้งเดียว
./monitor.sh

# รันซ้ำทุก 10 วินาที (กด Ctrl+C เพื่อหยุด)
watch -n 10 ./monitor.sh
```

### 📸 บันทึกรูปผลการทดลอง — monitor.sh

```
บันทึกภาพผลการรันคำสั่ง ./monitor.sh
```

---

### ส่วนที่ 7: ทำความสะอาด (Cleanup)

เมื่อเสร็จการทดลองให้รัน:

```bash
# หยุด containers ทั้งหมด
docker compose -f docker-compose.selfhosted.yml down

# ลบ volumes (ข้อมูล database)
docker compose -f docker-compose.selfhosted.yml down -v

# หยุด Runner (กด Ctrl+C ที่ Terminal ที่รัน run.sh)
```

---

## สรุปจุดสำคัญ

### ✅ สิ่งที่เรียนรู้

| หัวข้อ | รายละเอียด |
|---|---|
| Pull-based Model | Runner เป็นฝ่าย Poll งาน — ไม่ต้องเปิด Inbound Port |
| Self-Hosted vs Cloud | รันบนเครื่องตัวเองได้ เข้าถึง Local Resources ได้ |
| Docker Compose | รวม DB + Backend + Nginx ใน Compose เดียว |
| Health Check | ตรวจสอบ Container และ API ก่อน Report Success |
| Secrets Safety | ไม่ควรใช้กับ Public Repo เพราะ PR จากคนอื่นอาจรัน code บนเครื่องเรา |

### ❌ สิ่งที่ต้องหลีกเลี่ยง

1. ❌ ใช้ Self-Hosted Runner กับ Public Repository
2. ❌ Hard-code passwords/secrets ใน docker-compose (ใช้ GitHub Secrets แทนใน production)
3. ❌ รัน Runner ด้วย root user
4. ❌ ลืมปิด Runner เมื่อไม่ใช้งาน

---

## คำถามท้ายบท

### 1. Pull-based Model คืออะไร มีข้อดีด้านความปลอดภัยอย่างไร

<details>
<summary>คำตอบ</summary>

เขียนคำตอบที่นี่

</details>

### 2. ทำไมห้ามใช้ Self-Hosted Runner กับ Public Repository

<details>
<summary>คำตอบ</summary>

เขียนคำตอบที่นี่

</details>

### 3. docker-compose.selfhosted.yml ที่สร้างใหม่ต่างจาก backend/docker-compose.yml เดิมอย่างไร

<details>
<summary>คำตอบ</summary>

เขียนคำตอบที่นี่

</details>

### 4. Nginx ทำหน้าที่อะไรในระบบนี้ ทำไมต้องมี Reverse Proxy

<details>
<summary>คำตอบ</summary>

เขียนคำตอบที่นี่

</details>

### 5. Health Check ใน Workflow มีความสำคัญอย่างไร ถ้าไม่มีจะเกิดอะไรขึ้น

<details>
<summary>คำตอบ</summary>

เขียนคำตอบที่นี่

</details>

---

## เอกสารอ้างอิง

- [GitHub Actions Self-Hosted Runners](https://docs.github.com/en/actions/hosting-your-own-runners)
- [Security Hardening for Self-Hosted Runners](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#hardening-for-self-hosted-runners)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [Nginx Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [booking-app-demo-2025 Repository](https://github.com/surachai-p/booking-app-demo-2025)
