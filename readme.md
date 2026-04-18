# 📊 Finance Tracker (Static Web + Firebase)

เว็บแอปสำหรับจัดการรายรับ–รายจ่าย (Income/Expense Tracker) แบบ **HTML + JS + CSS ไฟล์เดียว**
รองรับการใช้งานผ่าน **GitHub Pages** และเก็บข้อมูลบน **Firebase (Auth + Firestore)** โดยไม่ต้องมี backend

---

# 🚀 Project Overview

แอปนี้ถูกออกแบบให้:

* ใช้งานง่าย (single HTML file)
* deploy ได้ทันทีผ่าน GitHub Pages
* ไม่ต้องมี backend server
* ข้อมูล sync ข้ามอุปกรณ์ได้ (ผ่าน Firebase)
* มีระบบ login แยก user

---

# 🧱 Tech Stack

## Frontend

* HTML + CSS + Vanilla JavaScript (Single file)
* Flatpickr (date/month picker พร้อม locale TH)

## Backend (Serverless)

* Firebase

  * Authentication (Email/Password)
  * Cloud Firestore (Database)
  * App Check (Security)

## Hosting

* GitHub Pages

---

# 🏗️ Architecture

```text
[ Browser (HTML App) ]
        │
        │ Firebase SDK (client-side)
        ▼
[ Firebase Auth ]  → login user
[ Firestore DB ]   → store appData (v2 schema)
[ App Check ]      → protect API usage
```

---

# 📦 Data Structure

## Runtime appData (Legacy / UI Layer)

แอปใช้ state หลักที่ UI อ่าน/เขียนโดยตรง:

```js
appData = {
  entries: [],       // รายรับ–รายจ่ายรายวัน
  installments: [],  // แผนผ่อนชำระ
  trips: [],         // ทริปพร้อมรายการค่าใช้จ่าย
  budgets: [],       // งบประมาณรายเดือน + งบทริป (NEW)
  goals: []          // เป้าหมายการเก็บเงิน (NEW)
}
```

## V2 Schema (Firestore / Export)

ข้อมูลที่บันทึกลง Firestore และ export ใช้ schema v2:

```js
{
  schemaVersion: 2,
  profile: { displayName, baseCurrency, locale, timezone },
  settings: { defaultView, monthStartsOn, includePendingInMonthlyTotals },
  masters: { categories: [], tags: [] },
  transactions: [],       // รายการทั้งหมด (manual + trip + installment)
  recurringRules: [],
  installmentPlans: [],
  trips: [],
  budgets: [],            // monthly budget + trip budget
  goals: [],
  meta: { createdAt, updatedAt, exportedAt }
}
```

เก็บใน Firestore แบบแยก subcollection ต่อ data type:

```text
users/{uid}/meta/app
users/{uid}/profile/main
users/{uid}/settings/main
users/{uid}/masters/main
users/{uid}/transactions/{docId}
users/{uid}/recurringRules/{docId}
users/{uid}/installmentPlans/{docId}
users/{uid}/trips/{docId}
users/{uid}/budgets/{docId}
users/{uid}/goals/{docId}
```

อ่านพร้อมกันด้วย `Promise.all()` และเขียนด้วย Firestore batch write


---

# ⚙️ Setup Guide (ตั้งแต่เริ่ม)

## 1. สร้าง Firebase Project

1. ไปที่ Firebase Console
2. Create Project
3. ปิด Google Analytics (optional)

---

## 2. เพิ่ม Web App

* กด `</>` (Web)
* ตั้งชื่อ app
* copy `firebaseConfig`

```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "...",
  projectId: "...",
  ...
};
```

---

## 3. เปิด Firestore

* Build → Firestore Database
* Create database
* เลือก location
* เริ่มด้วย **Test mode**

---

## 4. เปิด Authentication

* Build → Authentication
* Enable:

  * Email/Password

---

## 5. เพิ่ม Authorized Domain

* Authentication → Settings
* Add:

```text
yourname.github.io
```

---

## 6. ตั้ง Firestore Rules

```js
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId}/{collection}/{docId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

---

## 7. สร้าง User Test

* Authentication → Users → Add user

---

# 🔐 Security Setup

## Firebase Config ปลอดภัยไหม?

✅ ปลอดภัย:

* `firebaseConfig` ใส่ใน public repo ได้

❌ ห้าม:

* service account JSON
* admin key
* secret key

---

## App Check (กัน bot / abuse)

### Setup:

1. ไปที่ App Check
2. เลือก Web App
3. ใช้ reCAPTCHA v3
4. ใส่ domain

### ในโค้ด:

```js
firebase.appCheck(app).activate(
  "RECAPTCHA_SITE_KEY",
  true
);
```

### เปิด Enforcement:

* App Check → Firestore → Enforce

---

# 🔄 App Flow

## Startup Flow

```text
1. โหลดหน้าเว็บ
2. initialize Firebase
3. ตรวจ auth state
4. ถ้า login:
     → load Firestore data (v2 schema)
     → แปลงเป็น runtime appData
     → render UI (monthly view เป็น default)
5. ถ้าไม่ login:
     → แสดงหน้า login
```

---

## Save Flow

```text
1. user แก้ข้อมูล
2. update appData (runtime)
3. แปลงเป็น v2 schema ด้วย ensureV2Data()
4. เช็ค fingerprint (ถ้าไม่เปลี่ยน → ไม่ save)
5. debounce → save to Firestore
6. update save status UI
```

---

# 🔑 Features

## ✅ Authentication

* Email/Password login
* Logout
* Reset password

---

## 💾 Cloud Sync

* sync ข้อมูลผ่าน Firestore
* แยกข้อมูลตาม user (uid)
* fingerprint check เพื่อไม่ save ซ้ำโดยไม่จำเป็น

---

## 🔐 Security

* Firestore Rules (user isolation)
* App Check (anti-abuse)

---

## 💬 Save Status UI

สถานะ:

* idle
* saving
* saved
* error

---

## 📦 Backup Export

สามารถ export JSON (v2 schema):

```js
finance-backup-YYYY-MM-DD.json
```

---

# 🗓️ หน้าหลัก (Views)

แอปมี 4 มุมมองหลัก สลับได้จากเมนูบนหน้าจอ:

| View | ชื่อ | คำอธิบาย |
|---|---|---|
| `monthly` | รายเดือน | ดูและจัดการรายรับ–รายจ่ายตามช่วงเดือน |
| `yearly` | ภาพรวมทั้งปี | สรุปรายเดือนแบบ grid ภายในปีที่เลือก |
| `trips` | ทริป | จัดการค่าใช้จ่ายแยกตามทริป |
| `installments` | ยอดผ่อน | ติดตามแผนผ่อนชำระรายเดือน |

---

# ✨ Feature Overview

ภาพรวมฟีเจอร์ทั้งหมดในระบบ:

## 📅 รายเดือน (Monthly View)

* **Daily Dashboard** – สรุปสถานะวันนี้ทันทีเมื่อเปิดหน้า รวมถึงรายการที่ต้องจัดการในเดือนนี้
* **Smart Keyword Search** – ค้นหาด้วยภาษาธรรมชาติ เช่น `ยังไม่จ่าย`, `เดือนนี้`, `ของกิน เกิน 500`
* **ตัวกรองละเอียด** – กรองตามหมวดหมู่, ประเภท, ช่วงยอดเงิน และเรียงลำดับได้หลายแบบ
* **สรุปช่วงเดือน** – แสดงรายรับรวม, รายจ่ายรวม, คงเหลือสุทธิ, จำนวนรายการ และยอด readonly จากการผ่อน
* **Monthly Sub-navigation** – แถบ sub-tab แบบ sticky แยกส่วนของ monthly view ให้ใช้งานง่ายขึ้น
* **รายการใช้บ่อย** – กดครั้งเดียวเพื่อสร้างรายการเดิมซ้ำสำหรับวันนี้
* **Repeat Entry** – สร้างรายการซ้ำหลายงวดพร้อมกันได้

## 💰 Budget (NEW)

* ตั้งงบประมาณรายหมวดสำหรับแต่ละเดือน
* แสดง progress bar พร้อม status: `on-track`, `near-limit`, `over-budget`
* มี **Budget/Goal Insight** ที่คอยแจ้งเตือนแบบ soft เมื่อหมวดใดใกล้เกินงบ

## 🎯 Goal (NEW)

* เพิ่มเป้าหมายการเก็บเงินพร้อมกำหนดยอดเป้าหมายและวันที่
* อัปเดตยอดปัจจุบันได้ด้วยมือ (Quick Update)
* สถานะ: `Active`, `Paused`, `Completed` (auto เมื่อถึงเป้า)
* แสดง % คืบหน้าและยอดคงเหลือที่ต้องเก็บ

## 🌤️ ภาพรวมทั้งปี (Yearly View)

* แสดงสรุปรายรับ–รายจ่ายแบบ grid แยกตามเดือน
* กดที่เดือนใดก็ไปหน้า monthly ของเดือนนั้นได้ทันที
* เลื่อนปีได้อิสระทั้งย้อนหลังและล่วงหน้า

## ✈️ ทริป (Trip View)

* จัดการค่าใช้จ่ายแยกเป็นทริปอิสระ
* **Trip Detail Tabs**: แยก `ภาพรวม`, `รายการจริง`, `แผนงบ` ออกจากกัน
* **Trip Budget (NEW)** – วางแผนงบแยกตามหมวดหมู่ภายในทริป พร้อมเทียบกับค่าจริงอัตโนมัติ
* เชื่อมรายการผ่อนเข้ากับทริปได้ (linked installment)
* สลับ List View / Calendar View ได้

## 💳 ยอดผ่อน (Installments View)

* บันทึกแผนผ่อนพร้อมดอกเบี้ย (flat-rate / effective rate / ไม่มีดอกเบี้ย)
* Smart Hint คำนวณยอดรวมและยอดคงเหลืออัตโนมัติ
* กรองตามสถานะ, เดือน, คีย์เวิร์ด
* สลับ List View / Calendar View ได้

## 🛠️ UI / UX

* **Flatpickr** date picker พร้อม locale ภาษาไทย
* **Enhanced Select** – custom dropdown พร้อมช่องค้นหาในตัว
* **Collapsible Cards** – ปุ่มพับ/ขยาย card ได้ทุกส่วน
* **Mobile Responsive** – hamburger menu และ layout ปรับตาม breakpoint
* **Floating Back Button** – ปุ่มกลับลอยอยู่มุมขวาล่างเมื่อ drill-down ลึก

---

# 🧪 Testing Guide

## ทดสอบ Rules

### ต้องผ่าน:

* user A → อ่านของตัวเองได้
* user B → อ่านของ A ไม่ได้
* not login → เข้าไม่ได้

---

## ทดสอบ App Check

* เปิดเว็บปกติ → ใช้งานได้
* ยิง API จากที่อื่น → ต้องโดน block

---

# 🚀 Deployment

## GitHub Pages

1. push repo
2. เปิด Pages
3. เข้า URL:

```text
https://yourname.github.io/repo-name/
```

---

# ⚠️ Known Limitations

* ไม่มี conflict resolution (multi-device overwrite)
* ไม่มี version history
* save แบบ last-write-wins
* Budget และ Goal ยังไม่มีการ link อัตโนมัติกับรายการ entry (update ยอด Goal ด้วยมือ)

---

# 🔮 Future Improvements

## Level 2

* offline-first sync
* retry queue

## Level 3

* conflict resolution
* versioning

## Level 4

* multi-user sharing
* analytics
* recurring rules (ตั้งรายการซ้ำอัตโนมัติตาม recurringRules ใน v2 schema)

---

# 🧠 Key Design Decisions

## ทำไมใช้ Firebase

* ไม่ต้องมี backend
* ใช้งานจาก frontend ได้ตรง
* มี auth + db + security ครบ

## ทำไมใช้ Firestore

* เหมาะกับ document-based
* query ง่าย
* scale ได้ในอนาคต

## ทำไมใช้ Single HTML

* deploy ง่ายสุด
* เหมาะกับ GitHub Pages

## ทำไมมี Runtime appData และ V2 Schema แยกกัน

* Runtime `appData` เป็น UI-friendly format ที่อ่านเขียนง่าย
* V2 Schema เป็น normalized format สำหรับ Firestore และ export ที่ extensible กว่า
* แปลงไป-มาด้วย `buildV2FromCurrentAppData()` และ `buildCurrentAppDataFromV2()`

---

# ✅ Final Status

ระบบนี้ถือว่า:

👉 **Production-ready สำหรับ personal use แล้ว**

มีครบ:

* auth
* database (v2 schema)
* budget & goal tracking
* trip budget planning
* security
* deploy
* backup (v2 JSON export)

---

# 🙌 Credits

* Built with Firebase + Vanilla JS
* Designed for simplicity & zero-backend architecture