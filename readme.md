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
* รองรับ offline/cache ผ่าน IndexedDB

---

# 🧱 Tech Stack

## Frontend

* HTML + CSS + Vanilla JavaScript (Single file)
* IndexedDB (สำหรับ cache/offline)

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
[ Firestore DB ]   → store appData
[ App Check ]      → protect API usage
```

---

# 📦 Data Structure

แอปใช้ state หลัก:

```js
appData = {
  entries: [],
  installments: [],
  trips: []
}
```

เก็บใน Firestore ที่ path:

```text
users/{uid}/finance/main
```

ตัวอย่าง document:

```json
{
  "entries": [],
  "installments": [],
  "trips": [],
  "updatedAt": "2026-04-08T12:00:00Z"
}
```

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
    match /users/{userId}/finance/{docId} {
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
     → load Firestore data
     → set appData
     → render UI
5. ถ้าไม่ login:
     → แสดงหน้า login
```

---

## Save Flow

```text
1. user แก้ข้อมูล
2. update appData
3. debounce save
4. save → Firestore
5. update save status UI
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

---

## 📡 Offline Support

* IndexedDB เป็น cache/fallback

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

สามารถ export JSON:

```js
finance-backup-YYYY-MM-DD.json
```

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
* Firestore document เดียว (ยังไม่ scale)

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

---

# ✅ Final Status

ระบบนี้ถือว่า:

👉 **Production-ready สำหรับ personal use แล้ว**

มีครบ:

* auth
* database
* security
* deploy
* backup

---

# 🙌 Credits

* Built with Firebase + Vanilla JS
* Designed for simplicity & zero-backend architecture
