# TEAM_SPLIT.md — Final Lab Sec2 Set 2

## Team Members

| รหัสนักศึกษา | ชื่อ-นามสกุล |
|---|---|
| 67543210037-7 | นายปวริศ คูณศรี |
| 67543210040-1 | นายพนาวุฒน์ อภิปสันติ |

---

## Work Allocation

### นายปวริศ คูณศรี (67543210037-7)

- **Auth Service** — `auth-service/` ทั้งหมด
  - เพิ่ม `POST /api/auth/register` route
  - เพิ่ม `logToDB()` บันทึก log ลง auth-db
  - เพิ่ม `logActivity()` fire-and-forget ส่ง event ไป Activity Service
  - ปรับ login route ให้ส่ง `USER_LOGIN` event
  - `auth-service/init.sql` — users table + logs table + seed users
  - Deploy Auth Service + auth-db บน Railway
  - ตั้งค่า Environment Variables บน Railway
- **Frontend** -   - `config.js` — Railway Service URLs
  
- **Screenshot** - ทดสอบแต่ละ Test case

- **docker-compose.yml** + `.env.example`

### นายพนาวุฒน์ อภิปสันติ (67543210040-1)

- **Task Service** — `task-service/` ทั้งหมด
  - เพิ่ม `logActivity()` ใน CRUD routes ทุก endpoint
  - เพิ่ม `logToDB()` บันทึก log ลง task-db
  - `task-service/init.sql` — tasks table + logs table
  - Deploy Task Service + task-db บน Railway

- **Activity Service** — `activity-service/` ทั้งหมด (สร้างใหม่)
  - `POST /api/activity/internal` — รับ event จาก services อื่น
  - `GET /api/activity/me` — ดู activities ของตัวเอง
  - `GET /api/activity/all` — admin เท่านั้น
  - `activity-service/init.sql`
  - Deploy Activity Service + activity-db บน Railway
  - อัปเดต `ACTIVITY_SERVICE_URL` ใน Auth และ Task Service

- **Frontend** — `frontend/` ทั้งหมด
  - `index.html` — ปรับจาก Set 1: เพิ่ม Register tab, ลบ Profile tab, ลบ Log Dashboard, เพิ่ม Activity link
  - `activity.html` — Activity Timeline (สร้างใหม่)


---

## Shared Responsibilities

- Architecture diagram
- README.md
- INDIVIDUAL_REPORT ของแต่ละคน

---

## Integration Notes

1. **JWT_SECRET** ต้องตั้งค่าเหมือนกันทุก service บน Railway มิฉะนั้น token จาก Auth Service จะ verify ไม่ผ่านใน Task/Activity Service

2. **ACTIVITY_SERVICE_URL** — Auth Service และ Task Service ต้องรู้ URL จริงของ Activity Service บน Railway จึงต้อง deploy Activity Service ก่อน แล้วค่อยอัปเดต env ของ Auth/Task Service

3. **fire-and-forget** — `logActivity()` ใช้ `.catch(() => {})` ไม่มี `await` ทำให้ถ้า Activity Service ล่ม Auth/Task Service ยังทำงานได้ปกติ

4. **Database-per-Service** — แต่ละ service มี DB แยก ทำให้ Task Service ไม่มี users table ต้องเก็บ `username` ไว้ใน tasks table เอง (denormalization)
