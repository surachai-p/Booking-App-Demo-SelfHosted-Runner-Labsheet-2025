# ใบงาน: Self-Hosted Runner กับ Booking App (Windows)

## 🪟 สำหรับระบบปฏิบัติการ Windows

> 📌 **ข้อกำหนดเบื้องต้น:** ติดตั้ง **Docker Desktop** และ **Git for Windows (รวม Git Bash)** ไว้แล้ว

---

## วัตถุประสงค์

1. อธิบายหลักการทำงานของ Self-Hosted Runner แบบ Pull-based Model ได้
2. ติดตั้งและกำหนดค่า Self-Hosted Runner บนเครื่อง Windows ได้
3. สร้าง CI/CD Pipeline สำหรับ Deploy ระบบจองห้องพักลงบนเครื่อง Local ด้วย Docker Compose ได้
4. ตั้งค่า Nginx Reverse Proxy สำหรับ Backend API ได้
5. ตรวจสอบและ Monitor การ Deploy ด้วย PowerShell Script ได้

---

## ⚠️ ส่วนที่ 0: ตั้งค่า Windows ก่อนเริ่ม (สำคัญมาก!)

> **นักศึกษาต้องทำส่วนนี้ให้ครบก่อน** มิฉะนั้น Docker container จะรัน entrypoint script ไม่ได้  
> สาเหตุ: Windows ใช้ line endings แบบ **CRLF (`\r\n`)** แต่ Linux/Docker ต้องการ **LF (`\n`)** เท่านั้น

### 0.1 ตั้งค่า Git ให้ไม่แปลง Line Endings

เปิด **Git Bash** แล้วรัน:

```bash
# ปิดการแปลง CRLF อัตโนมัติของ Git
git config --global core.autocrlf false
git config --global core.eol lf

# ตรวจสอบ
git config --global core.autocrlf
# ต้องได้: false
```

### 0.2 ตรวจสอบ Docker Desktop

1. เปิด **Docker Desktop**
2. ตรวจสอบว่า Docker Engine ทำงานอยู่ (icon ที่ Taskbar เป็นสีเขียว)
3. เปิด PowerShell แล้วทดสอบ:

```powershell
docker --version
docker ps
```

ต้องไม่มี error

### 0.3 ตรวจสอบ Git Bash

```bash
# เปิด Git Bash แล้วทดสอบ
bash --version
curl --version
```

> 💡 **Tip:** ใบงานนี้ใช้ **Git Bash** สำหรับ command-line ทั้งหมด เพราะ Workflow ใช้ `shell: bash`  
> PowerShell ใช้เฉพาะการ Monitor และติดตั้ง Runner เท่านั้น

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
│  ┌────────────┐   ┌───────────────┐   ┌─────────────────┐  │
│  │ Repository │──>│    Workflow   │──>│   Job Queue     │  │
│  │   (Code)   │   │  (Actions)    │   │ (Pending Jobs)  │  │
│  └────────────┘   └───────────────┘   └─────────────────┘  │
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
                  │   Self-Hosted       │ ← รันบน Windows PC ของคุณ
                  │      Runner         │   (ใช้ Git Bash)
                  └─────────────────────┘
                          │
                          │ 3. Clone repo (Git Bash)
                          │ 4. Execute steps (shell: bash)
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

เปิด **Git Bash** แล้วรัน:

```bash
# Clone repository ที่ fork มา
git clone https://github.com/YOUR_USERNAME/booking-app-demo-2025.git

# เข้าไปในโฟลเดอร์
cd booking-app-demo-2025
```

#### 1.3 สร้าง .gitattributes เพื่อป้องกัน CRLF (ทำทันทีหลัง Clone)

> ⚠️ **ต้องทำก่อนสร้างไฟล์อื่น** เพื่อป้องกัน CRLF แทรกเข้าไปใน shell scripts

```bash
# สร้าง .gitattributes
cat > .gitattributes << 'EOF'
*.sh            text eol=lf
Dockerfile      text eol=lf
docker-compose*.yml text eol=lf
.env*           text eol=lf
*.md            text eol=lf
*.json          text eol=lf
*.js            text eol=lf
*.prisma        text eol=lf
*.yml           text eol=lf
*.yaml          text eol=lf
*.conf          text eol=lf
*.txt           text eol=lf
EOF

# Commit ทันที
git add .gitattributes
git commit -m "chore: enforce LF line endings for Docker compatibility"

# Normalize ไฟล์ทั้งหมดให้เป็น LF
git rm --cached -r .
git reset --hard
```

#### 1.4 ทำความเข้าใจโครงสร้างโปรเจกต์

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
│   ├── docker-entrypoint.sh   ← ต้องเป็น LF เท่านั้น!
│   └── docker-compose.yml
└── .github/
    └── workflows/           ← เราจะสร้าง workflow ใหม่ที่นี่
```

> **หมายเหตุ:** เราจะสร้าง `docker-compose.selfhosted.yml` ที่ root เพื่อ deploy ทั้ง backend + database + nginx

---

### ส่วนที่ 2: สร้างไฟล์ Configuration สำหรับ Self-Hosted Deployment

> 💡 ใช้ **Git Bash** สำหรับทุกคำสั่งในส่วนนี้ หรือสร้างไฟล์ด้วย VS Code แล้ว Set line ending เป็น LF

#### 2.1 สร้าง docker-compose.selfhosted.yml

สร้างไฟล์ `docker-compose.selfhosted.yml` ที่ root ของโปรเจกต์:

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

> 🪟 **Windows — ตรวจสอบ Line Ending:**  
> ถ้าสร้างด้วย VS Code ให้มองมุมขวาล่าง ต้องเห็น `LF` ไม่ใช่ `CRLF`  
> ถ้าเห็น `CRLF` ให้คลิกเพื่อเปลี่ยนเป็น `LF` แล้วบันทึกใหม่

#### 2.2 สร้าง nginx.selfhosted.conf

สร้างไฟล์ `nginx.selfhosted.conf` ที่ root ของโปรเจกต์:

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

**Git Bash:**

```bash
mkdir -p .github/workflows
touch .github/workflows/deploy-selfhosted.yml
```

เปิดไฟล์ `.github/workflows/deploy-selfhosted.yml` ด้วย VS Code แล้ววางเนื้อหานี้:

```yaml
name: 🚀 Deploy Booking App (Self-Hosted Windows)

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
        shell: bash
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
          echo "🖥️  Platform  : Windows (Self-Hosted)"
          echo "🐳 Docker    : $(docker --version)"
          echo "════════════════════════════════════════"

      # ──────────────────────────────────────────────
      # Step 3: ตรวจสอบไฟล์ที่จำเป็น
      # ──────────────────────────────────────────────
      - name: 🔍 Verify Required Files
        shell: bash
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
        shell: bash
        run: |
          echo "🛑 Stopping existing services..."
          docker compose -f $COMPOSE_FILE down --remove-orphans || echo "  No services running."
          
          echo "🧹 Removing old images..."
          docker image prune -f || true

      # ──────────────────────────────────────────────
      # Step 5: Build Docker Image
      # ──────────────────────────────────────────────
      - name: 🔨 Build Docker Images
        shell: bash
        run: |
          echo "🔨 Building backend image..."
          docker compose -f $COMPOSE_FILE build --no-cache backend
          
          echo "✅ Build complete."
          docker images | grep booking

      # ──────────────────────────────────────────────
      # Step 6: Start Services
      # ──────────────────────────────────────────────
      - name: 🚀 Start Services
        shell: bash
        run: |
          echo "🚀 Starting all services..."
          docker compose -f $COMPOSE_FILE up -d
          
          echo "⏳ Waiting for services to initialize (30s)..."
          sleep 30

      # ──────────────────────────────────────────────
      # Step 7: Health Check
      # ──────────────────────────────────────────────
      - name: 🏥 Health Check
        shell: bash
        run: |
          echo "🏥 Checking container health..."

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
              echo "  ❌ Health check failed after $MAX attempts (last: $HTTP_CODE)"
              docker compose -f $COMPOSE_FILE logs --tail 50
              exit 1
            fi
            echo "  ⏳ Attempt $i/$MAX — HTTP $HTTP_CODE — waiting..."
            sleep 3
          done

      # ──────────────────────────────────────────────
      # Step 8: Install jq (Windows ต้องการ)
      # ──────────────────────────────────────────────
      - name: 🔧 Install jq
        uses: dcarbone/install-jq-action@v3

      # ──────────────────────────────────────────────
      # Step 9: Test API Endpoints
      # ──────────────────────────────────────────────
      - name: 🧪 Test API Endpoints
        shell: bash
        run: |
          echo "🧪 Testing Booking App API..."

          echo "→ GET /api/rooms"
          ROOMS=$(curl -sf http://localhost:8080/api/rooms)
          echo "$ROOMS" | jq '.'
          echo "  ✅ Rooms endpoint working"

          echo ""
          echo "→ POST /api/login"
          LOGIN=$(curl -sf -X POST http://localhost:8080/api/login \
            -H "Content-Type: application/json" \
            -d '{"username":"admin","password":"admin123"}')
          echo "$LOGIN" | jq '.'

          if echo "$LOGIN" | jq -e '.token' > /dev/null 2>&1; then
            echo "  ✅ Login endpoint working"
          else
            echo "  ⚠️  Login response received (check credentials if unexpected)"
          fi

      # ──────────────────────────────────────────────
      # Step 10: แสดงสถานะ Containers
      # ──────────────────────────────────────────────
      - name: 📊 Display Status
        shell: bash
        run: |
          echo "════════════════════════════════════════"
          echo "📊 Container Status"
          echo "════════════════════════════════════════"
          docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" \
            | grep -E "(NAMES|booking-selfhosted)" || docker ps
          echo "════════════════════════════════════════"

      # ──────────────────────────────────────────────
      # Step 11: Logs (always runs)
      # ──────────────────────────────────────────────
      - name: 📝 Show Logs
        if: always()
        shell: bash
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
      # Step 12: Summary
      # ──────────────────────────────────────────────
      - name: 🎉 Deployment Summary
        if: success()
        shell: bash
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
          echo "💻 Platform     : Windows (Self-Hosted)"
          echo "════════════════════════════════════════"
```

> 🪟 **หมายเหตุสำคัญสำหรับ Windows:**  
> - ทุก step ต้องมี `shell: bash` เพื่อใช้ Git Bash แทน PowerShell
> - มี step `Install jq` พิเศษเพราะ Windows ไม่มี jq ติดมาในตัว

#### 2.4 Commit และ Push ไฟล์ทั้งหมด

เปิด **Git Bash** แล้วรัน:

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
git commit -m "feat: add self-hosted runner deployment config for Windows"

# Push (ยังไม่ trigger workflow เพราะ runner ยังไม่ได้ติดตั้ง)
git push origin main
```

---

### ส่วนที่ 3: ติดตั้ง Self-Hosted Runner บน Windows

#### 3.1 ไปที่ Repository Settings

1. ไปที่ GitHub repository ที่ Fork มา
2. คลิก **Settings**
3. เมนูซ้าย → **Actions** → **Runners**
4. คลิก **New self-hosted runner**

#### 3.2 เลือก Operating System

เลือก **Windows** → เลือก Architecture **x64**

#### 3.3 Download และ Configure Runner

เปิด **PowerShell** (ปกติ ไม่ต้อง Administrator) แล้วรันทีละขั้น:

**ขั้นที่ 1 — สร้างโฟลเดอร์**

```powershell
# สร้างโฟลเดอร์ที่ root ของ C: หรือ D:
cd C:\
mkdir actions-runner
cd actions-runner
```

**ขั้นที่ 2 — Download Runner Package**

```powershell
# ─────────────────────────────────────────────────────────────
# ⚠️ คัดลอกคำสั่ง Invoke-WebRequest จาก GitHub Settings มาใช้
#    เพราะ URL มี version และ checksum ล่าสุดอยู่แล้ว
# ─────────────────────────────────────────────────────────────
# ตัวอย่าง (อย่าคัดลอกตัวอย่างนี้ ให้ใช้จาก GitHub):
# Invoke-WebRequest -Uri https://github.com/actions/runner/releases/download/v2.xxx.x/actions-runner-win-x64-2.xxx.x.zip `
#   -OutFile actions-runner-win-x64-2.xxx.x.zip
```

**ขั้นที่ 3 — Extract**

```powershell
# คัดลอกคำสั่ง Add-Type จาก GitHub มาใช้
# ตัวอย่าง:
# Add-Type -AssemblyName System.IO.Compression.FileSystem
# [System.IO.Compression.ZipFile]::ExtractToDirectory("$PWD/actions-runner-win-x64-2.xxx.x.zip", "$PWD")
```

**ขั้นที่ 4 — Configure** (คัดลอกทั้งบรรทัดจาก GitHub เพราะมี Token ชั่วคราว)

```powershell
# คำสั่งจาก GitHub มีรูปแบบนี้:
# .\config.cmd --url https://github.com/YOUR_USERNAME/booking-app-demo-2025 --token AXXXXXXXXXXXXX
```

เมื่อระบบถามให้ตอบดังนี้:

```
Enter the name of the runner group [press Enter for Default]:
  → [Enter]

Enter the name of runner (e.g., Macbook):
  → my-windows-runner   (หรือชื่ออื่นตามต้องการ)

This runner will have the following labels: 'self-hosted', 'Windows', 'X64'
Enter any additional labels (ex. label-1,label-2) [press Enter to skip]:
  → [Enter]

Enter name of work folder [press Enter for _work]:
  → [Enter]

Would you like to run the runner as service? (Y/N) [press Enter for N]:
  → N
```

ผลลัพธ์ที่ควรเห็น:

```
√ Runner successfully added
√ Runner connection is good
```

#### 3.4 เริ่มต้น Runner

```powershell
# รัน Runner แบบ Interactive (เหมาะสำหรับทดสอบ)
.\run.cmd
```

ผลลัพธ์ที่ควรเห็น:

```
√ Connected to GitHub

Current runner version: '2.xxx.x'
2025-xx-xx xx:xx:xxZ: Listening for Jobs
```

> 💡 **สำคัญ:** ปล่อยให้ PowerShell นี้เปิดอยู่ตลอดการทดลอง Runner ต้องรันอยู่จึงจะรับงานได้

#### 3.5 ตรวจสอบ Runner บน GitHub

1. กลับไปที่ **Settings** → **Actions** → **Runners**
2. ควรเห็น Runner แสดงสถานะ **Idle** สีเขียว
3. Labels ต้องมี: `self-hosted`, `Windows`, `X64`

### 📸 บันทึกรูปผลการทดลอง — Runner Status

```
บันทึกภาพหน้า Runners ให้เห็น:
- ชื่อ GitHub Account และ Repository
- Runner แสดงสถานะ "Idle" สีเขียว
- Labels: self-hosted, Windows, X64
```

---

### ส่วนที่ 4: Trigger การ Deploy ครั้งแรก

#### 4.1 แก้ไขไฟล์เพื่อ Trigger Workflow

เปิด **Git Bash ใหม่** (อย่าปิด PowerShell ที่รัน Runner) แล้วไปที่โฟลเดอร์โปรเจกต์:

```bash
cd /c/path/to/booking-app-demo-2025
# หรือถ้า clone ไว้ที่ Desktop:
cd ~/Desktop/booking-app-demo-2025
```

สร้างไฟล์ `VERSION.txt` เพื่อ trigger workflow:

```bash
echo "Self-Hosted Deploy - Run 1" > VERSION.txt

git add VERSION.txt
git commit -m "test: trigger first self-hosted deployment on Windows"
git push origin main
```

#### 4.2 ติดตามการ Deploy

**1. ดูที่ PowerShell ที่รัน Runner:**

```
2025-xx-xx xx:xx:xxZ: Running job: 🚀 Deploy to Local Machine
```

**2. ดูบน GitHub:**
- ไปที่ tab **Actions** ของ Repository
- คลิก Workflow run ล่าสุด "🚀 Deploy Booking App (Self-Hosted Windows)"
- ดู log แบบ real-time

**3. หลัง Deploy เสร็จ ทดสอบ API:**

**Git Bash:**

```bash
# ทดสอบ GET /api/rooms
curl -s http://localhost:8080/api/rooms

# ทดสอบ Login
curl -s -X POST http://localhost:8080/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'

# ดู containers ที่รันอยู่
docker ps
```

**PowerShell (ถ้า curl ไม่พร้อมใช้):**

```powershell
# ทดสอบ GET /api/rooms
Invoke-WebRequest http://localhost:8080/api/rooms | Select-Object -ExpandProperty Content

# ดู containers
docker ps
```

**ผลลัพธ์ที่ควรเห็น:**

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

#### 5.1 Trigger Workflow ครั้งที่ 2

เปิด **Git Bash** แล้วรัน:

```bash
# อัปเดต VERSION เพื่อ trigger workflow
echo "Self-Hosted Deploy - Run 2 - $(date)" > VERSION.txt

git add VERSION.txt
git commit -m "feat: second deployment test with Windows self-hosted runner"
git push origin main
```

#### 5.2 ดู Runner รับงานใหม่

ดูที่ PowerShell ที่รัน Runner จะเห็น:

```
2025-xx-xx xx:xx:xxZ: Job completed: success
2025-xx-xx xx:xx:xxZ: Listening for Jobs
...
2025-xx-xx xx:xx:xxZ: Running job: 🚀 Deploy to Local Machine
```

---

### ส่วนที่ 6: Monitoring Script (PowerShell)

#### 6.1 สร้าง monitor.ps1

สร้างไฟล์ `monitor.ps1` ที่ root ของโปรเจกต์ด้วย VS Code:

```powershell
# monitor.ps1 — Booking App Self-Hosted Monitor (Windows)

$Sep = "=" * 44

Write-Host $Sep -ForegroundColor Cyan
Write-Host "  Booking App — Deployment Monitor (Windows)" -ForegroundColor White
Write-Host $Sep -ForegroundColor Cyan
Write-Host ""

# ── Runner Status ──
Write-Host "[1] Runner Status:" -ForegroundColor Yellow
$runnerProcess = Get-Process -ErrorAction SilentlyContinue | Where-Object {
  $_.ProcessName -like "*Runner.Listener*" -or $_.ProcessName -like "*Runner.Worker*"
}
if ($runnerProcess) {
  Write-Host "  OK: Self-Hosted Runner is Running" -ForegroundColor Green
} else {
  Write-Host "  WARN: Runner process not detected (may still be active)" -ForegroundColor DarkYellow
}
Write-Host ""

# ── Container Status ──
Write-Host "[2] Container Status:" -ForegroundColor Yellow
$containers = @(
  @{Name="booking-selfhosted-db";      Label="PostgreSQL"},
  @{Name="booking-selfhosted-backend"; Label="Backend API"},
  @{Name="booking-selfhosted-nginx";   Label="Nginx Proxy"}
)
foreach ($c in $containers) {
  $state = docker inspect --format "{{.State.Status}}" $c.Name 2>$null
  if ($state -eq "running") {
    Write-Host ("  OK: {0} ({1}): running" -f $c.Label, $c.Name) -ForegroundColor Green
  } else {
    $msg = if ($state) { $state } else { "not found" }
    Write-Host ("  FAIL: {0} ({1}): {2}" -f $c.Label, $c.Name, $msg) -ForegroundColor Red
  }
}
Write-Host ""

# ── API Endpoints ──
Write-Host "[3] API Endpoint Status:" -ForegroundColor Yellow
try {
  $resp = Invoke-WebRequest -Uri "http://localhost:8080/api/rooms" `
    -UseBasicParsing -TimeoutSec 5 -ErrorAction Stop
  Write-Host ("  OK: GET /api/rooms - HTTP {0}" -f $resp.StatusCode) -ForegroundColor Green
  $rooms = $resp.Content | ConvertFrom-Json
  Write-Host ("     Found {0} room(s) in database" -f $rooms.Count) -ForegroundColor Gray
} catch {
  Write-Host "  FAIL: Cannot reach http://localhost:8080/api/rooms" -ForegroundColor Red
}

try {
  $body = '{"username":"admin","password":"admin123"}'
  $loginResp = Invoke-WebRequest -Uri "http://localhost:8080/api/login" `
    -Method POST -Body $body -ContentType "application/json" `
    -UseBasicParsing -TimeoutSec 5 -ErrorAction Stop
  $loginJson = $loginResp.Content | ConvertFrom-Json
  if ($loginJson.token) {
    Write-Host "  OK: POST /api/login - Token received" -ForegroundColor Green
  } else {
    Write-Host ("  WARN: POST /api/login - HTTP {0} (no token)" -f $loginResp.StatusCode) -ForegroundColor DarkYellow
  }
} catch {
  Write-Host "  FAIL: POST /api/login failed" -ForegroundColor Red
}
Write-Host ""

# ── Resource Usage ──
Write-Host "[4] Resource Usage:" -ForegroundColor Yellow
$stats = docker stats --no-stream --format "{{.Container}}: CPU {{.CPUPerc}}, Mem {{.MemUsage}}" 2>$null
if ($stats) {
  $stats | Where-Object { $_ -like "*booking-selfhosted*" } | ForEach-Object {
    Write-Host ("  {0}" -f $_) -ForegroundColor Gray
  }
} else {
  Write-Host "  (No data)" -ForegroundColor Gray
}
Write-Host ""

Write-Host $Sep -ForegroundColor Cyan
Write-Host ("  Updated: {0}" -f (Get-Date -Format "yyyy-MM-dd HH:mm:ss")) -ForegroundColor Gray
Write-Host $Sep -ForegroundColor Cyan
```

#### 6.2 ตั้งค่า Execution Policy และรัน

**PowerShell:**

```powershell
# อนุญาตให้รัน script (ถ้า PowerShell ปฏิเสธ)
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# เข้าไปที่โฟลเดอร์โปรเจกต์
cd C:\path\to\booking-app-demo-2025

# รันครั้งเดียว
.\monitor.ps1
```

```powershell
# รันซ้ำทุก 10 วินาที (กด Ctrl+C เพื่อหยุด)
while ($true) {
  Clear-Host
  .\monitor.ps1
  Start-Sleep -Seconds 10
}
```

### 📸 บันทึกรูปผลการทดลอง — monitor.ps1

```
บันทึกภาพผลการรัน .\monitor.ps1
```

---

### ส่วนที่ 7: Troubleshooting บน Windows

#### ปัญหา 1: `exec ./docker-entrypoint.sh: no such file or directory`

**สาเหตุ:** ไฟล์ `docker-entrypoint.sh` มี CRLF line endings

**แก้ไข (Git Bash):**

```bash
# ตรวจสอบ line endings
cat -A backend/docker-entrypoint.sh | head -3
# ถ้าเห็น ^M$ แสดงว่าเป็น CRLF → ต้องแก้

# แก้ไข line endings
sed -i 's/\r$//' backend/docker-entrypoint.sh

# ตรวจสอบอีกครั้ง (ต้องเห็น $ ไม่มี ^M)
cat -A backend/docker-entrypoint.sh | head -3

# Commit และ Push
git add backend/docker-entrypoint.sh
git commit -m "fix: convert CRLF to LF in docker-entrypoint.sh"
git push origin main
```

#### ปัญหา 2: Docker Desktop ไม่ทำงาน

**แก้ไข:**

1. เปิด Docker Desktop และรอให้ Engine start (icon สีเขียว)
2. เปิด PowerShell ใหม่ แล้วรัน `docker ps`
3. ถ้าติดปัญหา WSL2: เปิด PowerShell as Administrator แล้วรัน:

```powershell
wsl --update
```

#### ปัญหา 3: Port 8080 ถูกใช้งานแล้ว

**แก้ไข (PowerShell):**

```powershell
# ดูว่า process ไหนใช้ port 8080
netstat -ano | findstr :8080

# ปิด process (ใส่ PID จากคำสั่งด้านบน)
taskkill /PID <PID> /F
```

หรือเปลี่ยน port ใน `docker-compose.selfhosted.yml`:

```yaml
ports:
  - "8081:80"   # เปลี่ยนจาก 8080 เป็น 8081
```

#### ปัญหา 4: Runner ไม่ Connect GitHub

**แก้ไข:**

- Token มีอายุสั้น (ประมาณ 1 ชั่วโมง) ถ้า configure ไม่ทัน ให้ขอ Token ใหม่จาก GitHub Settings
- ตรวจสอบ Internet Connection

#### ปัญหา 5: `git push` ถาม Username/Password

**แก้ไข:** ใช้ Personal Access Token แทน Password

1. GitHub → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)**
2. สร้าง token ใหม่ เลือก scope `repo`
3. ใช้ token นี้แทน password เมื่อ push

---

### ส่วนที่ 8: ทำความสะอาด (Cleanup)

เมื่อเสร็จการทดลองให้รัน:

**Git Bash:**

```bash
# หยุด containers ทั้งหมดและลบ network
docker compose -f docker-compose.selfhosted.yml down

# ลบ volumes (ข้อมูล database) ด้วย
docker compose -f docker-compose.selfhosted.yml down -v
```

```powershell
# หยุด Runner (กด Ctrl+C ที่ PowerShell ที่รัน .\run.cmd)
```

---

## สรุปจุดสำคัญ

### ✅ สิ่งที่เรียนรู้

| หัวข้อ | รายละเอียด |
|---|---|
| Pull-based Model | Runner เป็นฝ่าย Poll งาน — ไม่ต้องเปิด Inbound Port |
| CRLF Problem | Windows ต้องระวัง Line Ending เมื่อใช้กับ Docker/Linux |
| `shell: bash` | ทุก step ต้องระบุเพื่อใช้ Git Bash บน Windows |
| Docker Compose | รวม DB + Backend + Nginx ใน Compose เดียว |
| Health Check | ตรวจสอบ Container และ API ก่อน Report Success |
| jq on Windows | ต้องติดตั้งเพิ่มด้วย `dcarbone/install-jq-action` |

### ❌ สิ่งที่ต้องหลีกเลี่ยง

1. ❌ ใช้ Self-Hosted Runner กับ Public Repository
2. ❌ ลืมตั้งค่า `core.autocrlf false` ก่อนเริ่ม
3. ❌ สร้างไฟล์ `.sh` และ `.yml` โดยไม่ตรวจสอบ LF
4. ❌ รัน Workflow steps โดยไม่ระบุ `shell: bash`
5. ❌ Hard-code passwords ใน docker-compose
6. ❌ ลืมเปิด Docker Desktop ก่อน Push code

### 🪟 เฉพาะ Windows

| สิ่งที่ต้องทำ | เหตุผล |
|---|---|
| `git config --global core.autocrlf false` | ป้องกัน Git แปลง LF เป็น CRLF |
| สร้าง `.gitattributes` ทันทีหลัง Clone | ล็อก LF ให้ทุกไฟล์ |
| ใช้ Git Bash สำหรับ command-line | รองรับ bash syntax ใน workflow |
| ระบุ `shell: bash` ทุก step | บังคับใช้ Git Bash แทน PowerShell |
| ติดตั้ง jq ใน workflow | Windows ไม่มี jq ในตัว |

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

### 3. ปัญหา CRLF บน Windows คืออะไร และส่งผลต่อ Docker อย่างไร

<details>
<summary>คำตอบ</summary>

เขียนคำตอบที่นี่

</details>

### 4. Nginx ทำหน้าที่อะไรในระบบนี้ ทำไมต้องมี Reverse Proxy

<details>
<summary>คำตอบ</summary>

เขียนคำตอบที่นี่

</details>

### 5. ทำไม Workflow steps บน Windows Runner ต้องระบุ `shell: bash` ทุก step

<details>
<summary>คำตอบ</summary>

เขียนคำตอบที่นี่

</details>

### 6. Health Check ใน Workflow มีความสำคัญอย่างไร ถ้าไม่มีจะเกิดอะไรขึ้น

<details>
<summary>คำตอบ</summary>

เขียนคำตอบที่นี่

</details>

---

## เอกสารอ้างอิง

- [GitHub Actions Self-Hosted Runners](https://docs.github.com/en/actions/hosting-your-own-runners)
- [Security Hardening for Self-Hosted Runners](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions#hardening-for-self-hosted-runners)
- [Self-Hosted Runner on Windows](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners#requirements-for-self-hosted-runner-machines)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [Nginx Reverse Proxy Guide](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)
- [Git for Windows](https://git-scm.com/download/win)
- [booking-app-demo-2025 Repository](https://github.com/surachai-p/booking-app-demo-2025)
