# VisScan — Local Demo Setup Guide

คู่มือนี้สำหรับผู้ที่ต้องการ run VisScan Platform บนเครื่อง local  
**ข้อกำหนดเบื้องต้น:** ติดตั้ง [Docker Desktop](https://www.docker.com/products/docker-desktop/) แล้ว

---

## ขั้นตอน (5 นาที)

### 1. Clone หรือ Download โปรเจค

```bash
git clone <repository-url>
cd VisScan
```

หรือ unzip folder ที่ได้รับมา แล้ว `cd` เข้าไปใน folder

---

### 2. สร้างไฟล์ .env.local

**macOS / Linux:**
```bash
cp .env.local.example .env.local
```

**Windows (PowerShell):**
```powershell
copy .env.local.example .env.local
```

> ไม่ต้องแก้ค่าอะไรเพิ่ม — ค่า default ใช้งานได้เลย

---

### 3. Build และ Start ทุก Services

```bash
docker compose -f docker/docker-compose.local.yml up --build
```

รอประมาณ **2-5 นาที** (ครั้งแรก — ต้อง download และ build images)  
ครั้งถัดไปจะเร็วกว่ามากเพราะ cache แล้ว

เมื่อเห็น log ประมาณนี้แสดงว่าพร้อมแล้ว:
```
visscan-worker  | [SUCCESS] Superadmin account created/secured!
visscan-web     | ✓ Ready in ...ms
```

---

### 4. เปิดเบราว์เซอร์

ไปที่ **http://localhost:3000**

---

### 5. Login

คลิก **"Admin Login"** (ไม่ต้องใช้ CMU Account)

| Field    | Value                  |
|----------|------------------------|
| Email    | `SuperAdmin@VisScan`   |
| Password | `LocalDemo1234!`       |

> **Admin Password** สามารถเปลี่ยนได้โดยแก้ `ADMIN_PASSWORD` ใน `.env.local`

---

## หยุดการทำงาน

```bash
# หยุดชั่วคราว (เก็บ data ไว้)
docker compose -f docker/docker-compose.local.yml stop

# หยุดและลบ containers (เก็บ database volume ไว้)
docker compose -f docker/docker-compose.local.yml down

# หยุดและลบทุกอย่างรวมถึง database (เริ่มใหม่หมด)
docker compose -f docker/docker-compose.local.yml down -v
```

---

## Services ที่ run อยู่

| Service       | URL / Port                         | หน้าที่                          |
|---------------|------------------------------------|----------------------------------|
| Web App       | http://localhost:3000              | หน้าเว็บหลัก                     |
| RabbitMQ UI   | http://localhost:15672             | ดู message queue (guest/guest)   |
| PostgreSQL    | localhost:5432 (internal)          | Database (ไม่ expose ออกมา)      |

---

## ความแตกต่างจาก Production (มช.)

| Feature           | Local Demo              | Production (มช.)           |
|-------------------|-------------------------|----------------------------|
| Authentication    | Email + Password        | CMU EntraID (Microsoft)    |
| Port              | 3000                    | 80 (HTTPS)                 |
| GitLab CI/CD      | ไม่ทำงาน                | ทำงานเต็มรูปแบบ            |
| Scanner Worker    | ทำงาน (ถ้า config ครบ) | ทำงานเต็มรูปแบบ            |

> Local demo ใช้สำหรับ **ดูหน้าตา UI และ flow** ของระบบเท่านั้น  
> ฟีเจอร์ที่ต้องใช้ GitLab จะไม่ทำงานในโหมดนี้

---

## แก้ปัญหาเบื้องต้น

**Port 3000 ถูกใช้อยู่แล้ว:**  
แก้ `docker/docker-compose.local.yml` บรรทัด `ports` จาก `"3000:3000"` เป็น `"3001:3000"` แล้ว run ใหม่

**Build ล้มเหลว:**  
ลอง `docker compose -f docker/docker-compose.local.yml down -v` แล้ว build ใหม่

**Login ไม่ได้:**  
รอให้ log ของ `visscan-worker` แสดง `[SUCCESS] Superadmin account created` ก่อน แล้วค่อย login

**ต้องการความช่วยเหลือ:**  
ติดต่อทีม DevOps หรือดูเพิ่มเติมที่ `app/docs/getting-started`
