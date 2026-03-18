# INDIVIDUAL_REPORT_675432100401.md
## Final Lab Sec2 Set 2

## 1. ข้อมูลผู้จัดทำ

| | |
|---|---|
| **ชื่อ-นามสกุล** | นายพนาวุฒน์ อภิปสันติ |
| **รหัสนักศึกษา** | 67543210040-1 |
| **วิชา** | ENGSE207 Software Architecture |
| **งาน** | Final Lab Sec2 Set 2 — Microservices + Activity Tracking + Cloud (Railway) |

---

## 2. ส่วนที่รับผิดชอบ

- Task Service (`task-service/`) ทั้งหมด — เพิ่ม logActivity() ทุก CRUD route
- Activity Service (`activity-service/`) ทั้งหมด — สร้างใหม่
- Frontend (`frontend/index.html`, `frontend/activity.html`)

---

## 3. สิ่งที่ลงมือทำจริง

### Task Service — เพิ่ม logActivity()
เขียน helper `logActivity()` แบบ fire-and-forget แล้วเพิ่มการเรียกใน 3 จุด ได้แก่ หลัง POST /tasks สำเร็จส่ง `TASK_CREATED`, หลัง PUT /tasks/:id สำเร็จและ status เปลี่ยนส่ง `TASK_STATUS_CHANGED` พร้อม old_status และ new_status, และหลัง DELETE /tasks/:id สำเร็จส่ง `TASK_DELETED` นอกจากนี้เพิ่ม `username` field ใน tasks table เพื่อ denormalize ไว้ เพราะ task-db ไม่มี users table

### Activity Service — สร้างใหม่ทั้งหมด
สร้าง service ใหม่ที่ไม่ได้มาจาก Set 1 เลย มี 4 endpoints คือ `POST /api/activity/internal` รับ event จาก services อื่นโดยไม่ต้อง JWT, `GET /api/activity/me` ดู activities ของตัวเองโดยกรองจาก user_id ใน token, `GET /api/activity/all` สำหรับ admin เท่านั้น และ `GET /api/activity/health` สำหรับ health check ออกแบบ activities table ให้มี username เก็บไว้ด้วย (denormalization)

### Frontend — ปรับจาก Set 1
แก้ `index.html` จาก Set 1 โดยเพิ่ม Register tab และ `doRegister()` ที่ส่ง `username` แทน `name` พร้อม auto-login หลัง register สำเร็จ ลบ Profile & JWT tab ออก เปลี่ยน URL จาก relative เป็น `${AUTH}` และ `${TASK}` จาก config.js และเพิ่มลิงก์ไปหน้า `activity.html` แทน `logs.html`

สร้าง `activity.html` ใหม่ทั้งหมด มี timeline แสดง activity พร้อม color coding ตาม event type, filter ตาม event type, stats summary ด้านบน และ tab แยกระหว่าง "กิจกรรมของฉัน" กับ "ทั้งระบบ (admin)"

---

## 4. ปัญหาที่พบและวิธีแก้ (อย่างน้อย 2 ปัญหา)

**ปัญหา 1: TASK_STATUS_CHANGED ไม่ขึ้นใน activity timeline**

เข้าใจผิดว่าทุก PUT request จะส่ง TASK_STATUS_CHANGED event แต่จริงๆ code ตรวจว่า status เปลี่ยนจากค่าเดิมหรือไม่ก่อน ถ้าแก้แค่ title หรือ description โดยที่ status ยังเป็น TODO เหมือนเดิม จะไม่มี event ส่งออกไป ต้องเปลี่ยน status จริงๆ เช่น TODO → IN_PROGRESS ถึงจะเห็นใน activity timeline

**ปัญหา 2: activity.html แสดง 0 ทั้งหมดแม้จะ login และสร้าง task แล้ว**

สาเหตุมาจาก `ACTIVITY_SERVICE_URL` บน Railway ของ auth-service และ task-service ขาด `https://` นำหน้า (ปัญหานี้อยู่ฝั่ง deployment ซึ่งปวริศเป็นคนแก้) ทำให้ logActivity() ส่ง request ไปไม่ถึง activity-service เลย activities table จึงว่างเปล่า

---

## 5. อธิบาย: Denormalization ใน activities table คืออะไร และทำไมต้องทำ

ใน Database-per-Service pattern `activity-db` ไม่มี `users` table เพราะ users อยู่ใน `auth-db` คนละ database กัน ถ้าต้องการแสดง username ใน activity timeline ปกติต้อง JOIN กับ users table แต่ทำไม่ได้เพราะอยู่คนละ database

วิธีแก้คือ denormalize โดยเก็บ `username` ไว้ใน `activities` table เลย ณ เวลาที่ event เกิดขึ้น โดยดึงมาจาก JWT payload ที่ส่งมาพร้อมกับ event

```sql
CREATE TABLE activities (
  user_id    INTEGER NOT NULL,
  username   VARCHAR(50),   -- ← denormalized จาก JWT ณ เวลาที่ event เกิด
  event_type VARCHAR(50) NOT NULL,
  ...
);
```

ผลคือ activity-service แสดง username ได้โดยไม่ต้อง call กลับไปหา auth-service เลย

---

## 6. อธิบาย: ทำไม logActivity() ต้องเป็น fire-and-forget

```javascript
function logActivity({ userId, username, eventType, ... }) {
  fetch(`${ACTIVITY_URL}/api/activity/internal`, {
    method: 'POST', ...
  }).catch(() => {
    console.warn('activity-service unreachable — skipping');
  });
  // ไม่มี await → return ทันที ไม่รอผล
}
```

เหตุผลหลักคือ **Activity tracking เป็น non-critical feature** ถ้าใช้ `await` จะเกิดปัญหาสองอย่างคือ ถ้า Activity Service ล่ม Task Service จะ error ตามทุกครั้งที่สร้างหรือลบ task และ response time จะช้าลงเพราะต้องรอ network round-trip ไปหา activity-service ก่อน

เราเห็นผลจริงระหว่างทำงาน ตอนที่ activity-service ต่อ DB ไม่ได้ task-service ยังสร้างและลบ task ได้ปกติ เพราะ fire-and-forget ทำให้ error ถูกกลืนไปเงียบๆ โดยไม่กระทบการทำงานหลัก

---

## 7. ส่วนที่ยังไม่สมบูรณ์หรืออยากปรับปรุง

- TASK_STATUS_CHANGED ส่งเฉพาะตอน status เปลี่ยน ถ้าอยากให้ทุก update ขึ้นด้วยต้องเพิ่ม TASK_UPDATED event แยก
- activity.html ไม่มี auto-refresh ต้องกดโหลดใหม่เอง ควรเพิ่ม polling หรือ WebSocket
- ไม่มี pagination ที่ frontend แสดงได้แค่ 100 รายการล่าสุด
