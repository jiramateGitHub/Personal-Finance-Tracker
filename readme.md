# 📊 Personal Finance Tracker (Static Web + Firebase)

เว็บแอปจัดการรายรับ–รายจ่ายส่วนตัวแบบ **Single HTML File** ที่ใช้เพียง `index.html` ไฟล์เดียวสำหรับ UI, CSS และ JavaScript ทั้งหมด รองรับการใช้งานบน GitHub Pages หรือ static hosting อื่น ๆ และเก็บข้อมูลแยกตามผู้ใช้ผ่าน **Firebase Authentication + Cloud Firestore + App Check**

> README ฉบับนี้อัปเดตให้ตรงกับโค้ด `index.html` ล่าสุด

---

## ✅ Current Status

ระบบอยู่ในสถานะพร้อมใช้งานสำหรับ **personal finance tracking / personal use** โดยมีฟีเจอร์หลักครบถ้วน:

- Login ด้วย Firebase Email/Password
- Cloud sync ผ่าน Firestore แยกข้อมูลตาม `uid`
- Autosave หลังมีการเปลี่ยนแปลงข้อมูล
- Export / Import JSON ด้วย `schemaVersion: 2`
- Monthly dashboard, yearly overview, installment tracker, trip tracker
- Monthly Budget, Goal tracking และ Trip Budget แบบแยกรายหมวด
- Responsive UI พร้อม bottom navigation, modal/task sheet, enhanced select และ date/month picker ภาษาไทย

---

## 🧱 Tech Stack

### Frontend

- HTML + CSS + Vanilla JavaScript ในไฟล์เดียว (`index.html`)
- Google Font: Sarabun
- Flatpickr
  - Date picker
  - Month Select plugin
  - Thai locale
- Responsive layout
  - Bottom navigation
  - Sticky monthly sub-navigation
  - Modal / task sheet สำหรับมือถือ
  - Floating back button
  - Enhanced select พร้อม search ใน dropdown

### Backend / Serverless

- Firebase SDK compat `11.7.1`
- Firebase Authentication
  - Email/Password login
  - Password reset
  - Logout
- Cloud Firestore
  - เก็บข้อมูลแบบ subcollection ต่อ user
  - Batch write + delete stale docs
  - Fingerprint check กัน save ซ้ำ
- Firebase App Check
  - reCAPTCHA v3

### Hosting

- GitHub Pages หรือ static hosting อื่น ๆ
- ไม่มี build step
- ไม่มี backend server

---

## 📁 Project Structure

```text
.
├─ index.html   # แอปทั้งหมด: HTML + CSS + JavaScript
└─ readme.md    # เอกสารโปรเจกต์
```

> โปรเจกต์นี้ไม่จำเป็นต้องมี `package.json`, `npm install` หรือ `npm run build` เพราะ dependency ทั้งหมดโหลดผ่าน CDN

---

## 🏗️ Architecture

```text
[ Browser / index.html ]
        │
        │ Firebase client SDK
        ▼
[ Firebase Auth ]
        │
        │ currentUser.uid
        ▼
[ Cloud Firestore ]
        │
        ├─ users/{uid}/meta/app
        ├─ users/{uid}/profile/main
        ├─ users/{uid}/settings/main
        ├─ users/{uid}/masters/main
        ├─ users/{uid}/transactions/{docId}
        ├─ users/{uid}/recurringRules/{docId}
        ├─ users/{uid}/installmentPlans/{docId}
        ├─ users/{uid}/trips/{docId}
        ├─ users/{uid}/budgets/{docId}
        └─ users/{uid}/goals/{docId}
```

---

## 🔐 Authentication Flow

แอปเริ่มจากหน้า login เท่านั้น ถ้า login สำเร็จจึงโหลดข้อมูลและเข้าใช้งานแอป

```text
1. เปิด index.html
2. initialize Firebase
3. subscribe auth state ด้วย onAuthStateChanged
4. ถ้ามี user:
   - create default user document ถ้ายังไม่มี
   - load Firestore v2 data
   - แปลงเป็น runtime appData
   - render UI ทั้งหมด
5. ถ้าไม่มี user:
   - reset appData เป็นข้อมูลว่าง
   - แสดง login screen
```

ฟังก์ชันที่เกี่ยวข้องในโค้ด:

- `initializeFirebaseServices()`
- `handleAuthStateChanged()`
- `handleAuthenticatedUser()`
- `handleLogin()`
- `handleLogout()`
- `handleResetPassword()`

---

## 💾 Save / Sync Flow

เมื่อ user เพิ่ม แก้ไข หรือลบข้อมูล แอปจะ render ใหม่และเรียก save flow ผ่าน `persistCurrentData()`

```text
1. user แก้ข้อมูลใน runtime appData
2. renderAll()
3. persistCurrentData()
4. แปลง appData เป็น v2 schema ด้วย ensureV2Data()
5. สร้าง fingerprint โดยไม่เอา updatedAt/exportedAt มาคิด
6. ถ้า fingerprint ไม่เปลี่ยน → ไม่ save
7. ถ้าเปลี่ยน → debounce 800ms
8. save ไป Firestore
9. update save status: idle / saving / saved / error
```

รายละเอียดสำคัญ:

- ใช้ `cloudSaveTimer` debounce ประมาณ 800ms
- ใช้ `lastPersistedDataFingerprint` และ `lastCloudSavedFingerprint` เพื่อลดการเขียนซ้ำ
- เขียน Firestore แบบแยก collection และ batch ละไม่เกิน 400 operations
- ถ้า collection เดิมมี doc ที่ไม่มีใน payload ใหม่ จะลบ doc เก่าทิ้งเพื่อให้ cloud state ตรงกับ local state

---

## 📦 Data Model

### Runtime appData

UI ทำงานกับ state หลักตัวนี้:

```js
appData = {
  entries: [],       // รายรับ/รายจ่ายที่ user เพิ่มเอง
  installments: [],  // แผนผ่อนชำระ
  trips: [],         // ทริป + รายการค่าใช้จ่ายในทริป
  budgets: [],       // monthly budget + trip budget
  goals: []          // saving goals
}
```

### V2 Schema สำหรับ Firestore / Export

```js
{
  schemaVersion: 2,
  profile: {
    displayName: '',
    baseCurrency: 'THB',
    locale: 'th-TH',
    timezone: 'Asia/Bangkok'
  },
  settings: {
    defaultView: 'monthly',
    monthStartsOn: 1,
    includePendingInMonthlyTotals: true
  },
  masters: {
    categories: [],
    tags: []
  },
  transactions: [],
  recurringRules: [],
  installmentPlans: [],
  trips: [],
  budgets: [],
  goals: [],
  meta: {
    createdAt: '',
    updatedAt: '',
    exportedAt: null
  }
}
```

### Data Conversion

โค้ดแยก runtime state กับ export/cloud schema ออกจากกัน:

- `buildV2FromCurrentAppData()`
- `buildCurrentAppDataFromV2()`
- `normalizeV2Data()`
- `ensureV2Data()`
- `parseImportedV2Data()`
- `buildExportDataV2()`
- `buildFirestoreV2Payload()`

---

## 🗂️ Firestore Collections

```text
users/{uid}/meta/app
users/{uid}/profile/main
users/{uid}/settings/main
users/{uid}/masters/main
users/{uid}/transactions/{transactionId}
users/{uid}/recurringRules/{ruleId}
users/{uid}/installmentPlans/{planId}
users/{uid}/trips/{tripId}
users/{uid}/budgets/{budgetId}
users/{uid}/goals/{goalId}
```

### Firestore Rules

ใช้ rule นี้เพื่อให้ user อ่าน/เขียนได้เฉพาะข้อมูลตัวเอง:

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

> Rule ด้านบนตรงกับโครงสร้างปัจจุบัน เพราะข้อมูลทั้งหมดอยู่ใต้ `users/{uid}/{collection}/{docId}` หนึ่งระดับ

---

## 🔧 Firebase Setup

### 1. Create Firebase Project

1. เปิด Firebase Console
2. Create Project
3. จะเปิดหรือปิด Google Analytics ก็ได้

### 2. Add Web App

1. เพิ่ม Web App ด้วยปุ่ม `</>`
2. Copy `firebaseConfig`
3. นำค่าไปใส่ใน `index.html`

```js
const firebaseConfig = {
  apiKey: '...',
  authDomain: '...',
  projectId: '...',
  storageBucket: '...',
  messagingSenderId: '...',
  appId: '...'
};
```

### 3. Enable Authentication

ไปที่ Firebase Console:

```text
Build → Authentication → Sign-in method → Email/Password → Enable
```

จากนั้นสร้าง user สำหรับทดสอบ:

```text
Authentication → Users → Add user
```

> โค้ดปัจจุบันมีเฉพาะ login / logout / reset password ยังไม่มีหน้าสมัครสมาชิกเอง

### 4. Enable Firestore

```text
Build → Firestore Database → Create database
```

แนะนำให้เริ่มจาก test mode เฉพาะช่วง setup แล้วเปลี่ยนเป็น rules ด้านบนก่อนใช้งานจริง

### 5. Authorized Domains

เพิ่ม domain ที่ใช้ deploy:

```text
Authentication → Settings → Authorized domains
```

ตัวอย่าง:

```text
localhost
yourname.github.io
custom-domain.com
```

### 6. App Check

1. ไปที่ Firebase Console → App Check
2. เลือก Web App
3. ใช้ reCAPTCHA v3
4. เพิ่ม domain ที่ใช้งานจริง
5. นำ site key ไปใส่ในโค้ด

```js
firebase.appCheck(app).activate(
  'RECAPTCHA_SITE_KEY',
  true
);
```

หลังทดสอบเรียบร้อยแล้วค่อยเปิด enforcement สำหรับ Firestore

```text
App Check → Firestore → Enforce
```

---

## 🚀 Deployment

### Deploy ด้วย GitHub Pages

1. เตรียมไฟล์ใน repo:

```text
index.html
readme.md
```

2. Commit และ push ขึ้น GitHub

```bash
git add index.html readme.md
git commit -m "update finance tracker"
git push
```

3. เปิด GitHub Pages

```text
Repository → Settings → Pages
```

4. เลือก branch เช่น `main` และ folder `/root`
5. เปิด URL ที่ GitHub Pages ให้มา

```text
https://yourname.github.io/repo-name/
```

6. กลับไปเพิ่ม domain นี้ใน Firebase Authorized Domains และ App Check

### Deploy ด้วย Static Hosting อื่น ๆ

ใช้ได้เช่นกัน เพราะเป็น static file:

- Firebase Hosting
- Azure Static Web Apps
- Netlify
- Vercel static output
- Any web server / object storage static website

---

## 🧭 Main Views

แอปมี destination หลักใน bottom navigation 5 จุด:

| Destination | View | รายละเอียด |
|---|---|---|
| `home` | Monthly Summary | หน้าแรก สรุปเดือนปัจจุบัน/ช่วงเดือน, dashboard, action needed, budget/goal insight |
| `entries` | Monthly Entries | รายการรายรับ/รายจ่ายตามตัวกรอง |
| `installments` | Installments | จัดการยอดผ่อนและดู calendar/list view |
| `trips` | Trips | จัดการทริป, รายการค่าใช้จ่าย, trip budget |
| `more` | More / Utility Hub | Export/Import, account, logout, shortcut ไป yearly/budget/goal |

นอกจากนี้มี `yearly` เป็น full-page view ที่เปิดจากหน้า More หรือกด drill-down จากส่วนอื่น

---

## ✨ Feature Overview

### 📅 Monthly View

- เลือกช่วงเดือนเริ่มต้น/สิ้นสุดได้
- ค้นหาแบบ smart keyword เช่น:
  - `ยังไม่จ่าย`
  - `จ่ายแล้ว`
  - `เดือนนี้`
  - `เดือนก่อน`
  - `รายรับ`
  - `รายจ่าย`
  - `ของกิน เกิน 500`
  - `ต่ำกว่า 100`
- กรองตามหมวดหมู่ ประเภท และช่วงยอดเงิน
- เรียงตามวันที่, ยอดเงิน หรือชื่อรายการ
- Daily dashboard สำหรับสรุปวันนี้
- Action Needed สำหรับรายการที่ควรจัดการเร็ว ๆ นี้
- Summary card:
  - รายรับรวม
  - รายจ่ายรวม
  - คงเหลือสุทธิ
  - จำนวนรายการ
  - ยอด readonly รวมจาก installment/trip ที่ถูกแสดงร่วม
- Quick actions:
  - เพิ่มรายรับ
  - เพิ่มรายจ่าย
  - เพิ่มยอดผ่อน
  - สร้างทริป
- Recent/Frequent entries สำหรับสร้างรายการซ้ำจากรายการที่ใช้บ่อย
- Quick filter chips สำหรับช่วยดูรายการเร็วขึ้น

### ⚡ Quick Add

ใน modal รายรับ/รายจ่ายสามารถพิมพ์รายการสั้น ๆ แล้วบันทึกได้ทันที เช่น:

```text
+ ค่าไฟ 1200 ยังไม่จ่าย
โบนัส 5000
grab ไป office 180
ข้าว 80 เมื่อวาน
```

ระบบจะพยายามตรวจให้อัตโนมัติ:

- ประเภท รายรับ/รายจ่าย
- หมวดหมู่ เช่น ของกิน, เดินทาง, ค่าน้ำมัน, เงินเดือน, ท่องเที่ยว, ไฟฟ้า, โบนัส
- วันที่ เช่น วันนี้, เมื่อวาน, พรุ่งนี้
- สถานะจ่ายแล้ว/ยังไม่จ่าย

### 🧾 Entries

- เพิ่ม/แก้ไข/ลบรายการรายรับรายจ่าย
- รายจ่ายมีสถานะ `จ่ายแล้ว` / `ยังไม่จ่าย`
- รายรับถือว่า cleared อัตโนมัติ
- ทำรายการซ้ำรายเดือนในครั้งเดียวได้ สูงสุด 60 เดือน
- หลังเพิ่มรายการจะ highlight รายการที่เพิ่งสร้าง

### 💰 Monthly Budget

- ตั้งงบประมาณรายเดือนแยกตามหมวดหมู่
- ถ้าเดือน+หมวดซ้ำ ระบบจะ merge/อัปเดตรายการเดิมแทนการสร้างซ้ำ
- คำนวณยอดใช้จริงจากรายการรายจ่ายในเดือนนั้น
- แสดง progress และสถานะ:
  - `safe`
  - `near-limit`
  - `over-budget`
- threshold เริ่มต้นคือ 80% และ 100%
- มี Budget/Goal Insight เพื่อแจ้งเตือนหมวดที่ใกล้เกินงบหรือเกินงบแล้ว

### 🎯 Goal

- สร้างเป้าหมายการออม
- กำหนดชื่อ goal, ยอดเป้าหมาย, ยอดปัจจุบัน, วันเป้าหมาย และสถานะ
- สถานะที่รองรับ:
  - `active`
  - `paused`
  - `completed`
- แสดง progress percentage และยอดคงเหลือ
- มี quick update สำหรับอัปเดตยอดปัจจุบัน
- ถ้ายอดปัจจุบันถึงเป้าหมาย ระบบแสดงเป็น completed ใน UI

### 📆 Yearly Overview

- เลือกปีได้
- สรุปรายรับ/รายจ่าย/คงเหลือทั้งปี
- แสดงยอด readonly ทั้งปี
- แสดง grid 12 เดือน
- กดเดือนเพื่อกลับไปดู Monthly View ของเดือนนั้นได้

### 💳 Installments

- จัดการแผนผ่อนชำระ
- รองรับข้อมูล:
  - ชื่อรายการ
  - หมวดหมู่
  - ยอดต่อเดือน
  - จำนวนเดือนทั้งหมด
  - เดือนที่จ่ายแล้ว
  - เดือนเริ่มผ่อน
  - ยอดตั้งต้น
  - ยอดคงเหลือ override
  - วันที่ครบกำหนดจ่าย
  - รูปแบบดอกเบี้ย
  - อัตราดอกเบี้ย
  - หมายเหตุ
- รูปแบบดอกเบี้ย:
  - ไม่มีดอกเบี้ย (`none`)
  - ลดต้นลดดอก / EMI (`reducing`)
  - Flat Rate (`flat`)
- มี Smart Hint เพื่อช่วยคำนวณยอด
- Filter ตาม keyword, สถานะ และช่วงเดือน
- สลับมุมมอง List / Calendar
- Mark paid/unpaid รายเดือนได้

### 🧳 Trips

- สร้างทริปพร้อมชื่อ, destination, budget, start/end date และ note
- เพิ่มรายการค่าใช้จ่ายเข้าแต่ละทริป
- Filter ทริปตาม keyword, ปี, เดือน, หมวดหมู่ และสถานะทริป
- สถานะทริป:
  - upcoming
  - ongoing
  - completed
- สลับมุมมอง List / Calendar
- Trip detail tabs:
  - Overview
  - Actual items
  - Plan / Budget
- รายการทริปสามารถ link กับ installment ได้
- ถ้ารายการทริป link กับ installment ระบบจะใช้สถานะ/ยอดจาก installment เป็น readonly reference เพื่อลดการนับซ้ำ

### 🧳 Trip Budget

- จัดแผนงบทริปแยกตามหมวดหมู่
- เพิ่ม/แก้ไข/ลบบรรทัดงบรายหมวด
- เทียบ Planned vs Actual อัตโนมัติ
- แสดงสถานะใกล้เกินงบ/เกินงบในระดับทริป
- ถ้าทริปมี budget รวม แต่ยังไม่มี budget รายหมวด ระบบสามารถ seed เป็นแผนงบเริ่มต้นได้

### ➕ More / Utility Hub

- แสดงบัญชีที่ login อยู่
- Logout
- Save status
- Export JSON
- Import JSON
- Shortcut ไป Yearly Overview
- Shortcut เปิด Budget modal
- Shortcut เปิด Goal modal

---

## 📤 Export / Import JSON

### Export

กด `Export JSON` จากหน้า More

- Export เป็น v2 schema
- ถ้า browser รองรับ File System Access API และมี file handle จาก import เดิม ระบบจะพยายามเขียนทับไฟล์เดิม
- ถ้าเขียนทับไม่ได้ จะ download เป็นไฟล์ใหม่
- ชื่อ fallback ปัจจุบัน:

```text
finance-data-YYYYMMDD-HHmmss.json
```

### Import

กด `Import` จากหน้า More

- รองรับเฉพาะ JSON ที่มี `schemaVersion: 2`
- import แล้วจะแปลงเป็น runtime `appData`
- ถ้า login อยู่ ระบบจะพยายาม sync ข้อมูลที่ import ขึ้น Firestore ด้วย

ตัวอย่าง root structure:

```json
{
  "schemaVersion": 2,
  "profile": {},
  "settings": {},
  "masters": {},
  "transactions": [],
  "recurringRules": [],
  "installmentPlans": [],
  "trips": [],
  "budgets": [],
  "goals": [],
  "meta": {}
}
```

---

## 🧪 Testing Checklist

### Authentication

- Login ด้วย email/password สำเร็จ
- ใส่รหัสผิดแล้วแสดง error
- Reset password ส่ง email ได้
- Logout แล้วกลับหน้า login

### Firestore Rules

- User A อ่าน/เขียนข้อมูลตัวเองได้
- User A อ่าน/เขียนข้อมูล User B ไม่ได้
- ไม่ login แล้วอ่าน/เขียนไม่ได้

### Cloud Sync

- เพิ่มรายการแล้ว save status เปลี่ยนเป็น saving → saved
- Refresh หน้าแล้วข้อมูลยังอยู่
- Login user เดิมจาก browser อื่นแล้วโหลดข้อมูลได้
- เพิ่มข้อมูลซ้ำ ๆ แล้วไม่มี duplicate จาก autosave

### Export / Import

- Export แล้วได้ JSON v2
- Import JSON v2 แล้ว render ถูกต้อง
- Import ไฟล์ที่ไม่มี `schemaVersion` แล้วต้องแจ้ง error
- Import ไฟล์ที่ไม่ใช่ v2 แล้วต้องแจ้ง error

### Feature Regression

- เพิ่มรายรับ/รายจ่าย manual
- Quick Add
- Repeat entry หลายเดือน
- เพิ่ม/แก้ไข/ลบ Budget
- เพิ่ม/แก้ไข/ลบ Goal
- เพิ่ม/แก้ไข/ลบ Installment
- Mark paid/unpaid ของ installment รายเดือน
- เพิ่ม/แก้ไข/ลบ Trip
- เพิ่ม/แก้ไข/ลบ Trip item
- เพิ่ม/แก้ไข/ลบ Trip Budget line
- Link trip item กับ installment
- Yearly drill-down กลับไป monthly

---

## ⚠️ Known Limitations

- ยังไม่มีหน้า register account ใน UI
- ยังไม่มี offline-first sync / retry queue แบบเต็มรูปแบบ
- ยังไม่มี conflict resolution สำหรับหลายอุปกรณ์แก้ข้อมูลพร้อมกัน
- Save model เป็น last-write-wins
- ยังไม่มี version history / undo history
- `recurringRules` มีอยู่ใน v2 schema แล้ว แต่ยังไม่มี UI สำหรับ recurring rule อัตโนมัติแบบเต็มรูปแบบ
- Goal ยังอัปเดตยอดปัจจุบันด้วยมือ ไม่ได้ link กับรายการรายรับ/รายจ่ายแบบอัตโนมัติ
- Firebase config อยู่ฝั่ง client ตามรูปแบบ Firebase Web App ปกติ จึงต้องพึ่ง Firestore Rules + App Check เพื่อควบคุมความปลอดภัย

---

## 🔐 Security Notes

### Firebase Config Public ได้ไหม?

โดยปกติ `firebaseConfig` ของ Firebase Web App สามารถอยู่ใน frontend ได้ เพราะไม่ใช่ service account secret

สิ่งที่ห้าม commit:

- Firebase Admin SDK service account JSON
- Private key
- Secret key ฝั่ง server
- Token ส่วนตัว

สิ่งที่ต้องตั้งให้ถูก:

- Firestore Rules
- Firebase Authorized Domains
- App Check domain
- App Check enforcement สำหรับ Firestore เมื่อพร้อมใช้งานจริง

---

## 🧠 Key Design Decisions

### ทำไมใช้ Single HTML File

- Deploy ง่ายมาก
- เหมาะกับ GitHub Pages
- ไม่มี build process
- เหมาะกับ personal tool ที่ต้องการแก้ไขเร็ว

### ทำไมแยก Runtime appData กับ V2 Schema

- Runtime `appData` เหมาะกับ UI และ logic เดิม
- V2 Schema เหมาะกับ Firestore และ JSON export
- ทำให้ระบบรองรับการขยาย data model ได้ง่ายขึ้น

### ทำไมแยก Firestore เป็นหลาย collection

- ลดขนาด document เดียวที่ใหญ่เกินไป
- sync รายการจำนวนมากได้ยืดหยุ่นขึ้น
- จัดประเภทข้อมูลชัดเจน เช่น transactions, trips, budgets, goals

### ทำไมใช้ Firebase

- ไม่ต้องทำ backend server
- มี Auth + Database + Security ในชุดเดียว
- เหมาะกับ static web app
- ใช้งานข้ามอุปกรณ์ได้ง่าย

---

## 🔮 Future Improvements

### Level 1

- เพิ่มหน้า register account
- เพิ่ม manual refresh cloud data
- เพิ่ม confirmation ก่อน import overwrite cloud data

### Level 2

- Offline-first cache
- Retry queue สำหรับ save fail
- Conflict detection จาก `updatedAt` หรือ revision number

### Level 3

- Version history
- Restore snapshot
- Multi-user sharing / family finance

### Level 4

- Recurring rules UI
- Auto-generate recurring transactions
- Goal auto-contribution จากรายการออมเงิน/ลงทุน
- Analytics dashboard เพิ่มเติม

---

## 🙌 Credits

Built with:

- Firebase
- Flatpickr
- Vanilla JavaScript
- Sarabun font

Designed for simple, zero-backend personal finance tracking.
