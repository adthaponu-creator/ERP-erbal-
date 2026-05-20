# IPC Criteria Prototype — Handoff to Development

เอกสารนี้สำหรับทีม dev ที่จะนำ prototype นี้ไปพัฒนาเป็น production application

---

## 📌 Scope

Prototype นี้คือ **single-page HTML** สำหรับระบบ In-Process Control (IPC) ในการผลิตยาสมุนไพร ประกอบด้วย 2 หน้า:

| ไฟล์ | บทบาท | ผู้ใช้ |
|---|---|---|
| `index.html` | สร้าง/แก้ไข IPC Criteria | QA / Admin |
| `record.html` | บันทึกผลการตรวจสอบ | Operator (line) |

**Prototype นี้ใช้เป็น UX/UI reference เท่านั้น** — ไม่ใช่ production-ready code

---

## ⚠️ Critical Limitations

Dev **ต้อง** แก้ทั้งหมดนี้ก่อน deploy production:

### 1. ไม่มี Backend / Database
- ข้อมูลทั้งหมดเก็บใน **browser localStorage**
- หายเมื่อ: ล้าง browser cache, เปลี่ยนเครื่อง, ใช้ incognito
- ไม่ share ระหว่าง user/เครื่อง
- **Required:** REST API + persistent DB (PostgreSQL recommended)

### 2. ไม่มี Authentication / Authorization
- ไม่มี login, ไม่มี role-based access
- ทุกคนทำได้ทุกอย่าง รวมถึงลบข้อมูลทั้งหมด
- **Required:** User auth (JWT/Session), RBAC (operator/QA/admin/manager)

### 3. ไม่ Compliant กับ GMP / 21 CFR Part 11
Prototype ไม่ครอบคลุมข้อกำหนด electronic records ของ FDA/GMP:
- ❌ Electronic signature
- ❌ User identity binding
- ❌ Immutable audit trail (record ที่ submitted แก้ได้ผ่าน localStorage)
- ❌ Reason for change tracking
- ❌ Version history
- ❌ Time stamp ที่ trusted (server-side)

**Required:** ปรึกษา QA/Regulatory ก่อนออกแบบ audit/sign-off layer

### 4. ไม่มี Validation ทาง business logic
- ไม่ enforce SOP version control
- ไม่ check duplicate criteria code
- ไม่ผูก batch number / lot number
- ไม่มี approval workflow
- ไม่มี deviation linkage จริง

### 5. Code Architecture ไม่ production-grade
- ทั้งหมดอยู่ใน 1 HTML file (3,000+ บรรทัด/ไฟล์)
- ใช้ React/Tailwind/Babel via CDN — production ต้องใช้ build pipeline
- ไม่มี TypeScript, no tests, no error boundaries
- ไม่ separate components/hooks/utils

---

## 🎯 Core Features ที่ Prototype ทำให้ดู

### หน้า New IPC (`index.html`)

1. **Criteria Information form** — รองรับ 9 criteria types:
   - `numeric` — ค่าตัวเลข ± tolerance
   - `pass_fail` — ผ่าน/ไม่ผ่าน
   - `visual` — ตรวจสายตา checklist
   - `text` — บันทึกข้อความ
   - `multi_point` — วัดหลายจุด + aggregate (all_pass/mean/rsd/min_max)
   - `custom_multi_field` — หลายฟิลด์
   - `calibration` — ตรวจสอบเครื่องวัด
   - `tare` — น้ำหนักอ้างอิง
   - `calculated` — คำนวณจากสูตร

2. **Smart auto-fill** — เลือก Test Name → เติม Code, Unit, Sample Plan, Stages

3. **Multi-Stage Acceptance (USP <711>, <905>, <701>)** — Stage 1 → Stage 2 → Stage 3 พร้อม `onFail` action: next/reject/deviation

4. **Multi-Trigger inspection** — 5 trigger types ใช้พร้อมกันได้:
   - Time-based (ทุก N นาที)
   - Quantity-based (ทุก N units)
   - Milestone (เริ่ม batch, กลาง, ก่อนปิด, ฯลฯ)
   - Event-driven (changeover, lot change, cleaning, parameter change, maintenance)
   - Once-per-batch

5. **Tare linkage (Multi-Point)** — เลือก Tare criteria จาก dropdown → ระบบเก็บ reference สำหรับใช้ในหน้า record

6. **Auto-calculated Mean range** — Mean/Min-Max acceptance range คำนวณจาก Target ± Tolerance อัตโนมัติ

7. **Derived Calculations** — สูตรคำนวณ cross-step (เช่น Yield = output / input * 100)

8. **Live Preview Panel** — แสดงตัวอย่างที่ operator จะเห็น + GMP Compliance checklist

### หน้า Recording (`record.html`)

1. **List View** — แสดง criteria ทั้งหมดที่ตั้งไว้
   - Badge สถานะ: ✓ N รอบ / กำลังบันทึก / Critical
   - ปุ่ม "ดูรอบทั้งหมด" + "+ รอบใหม่"

2. **Record View** — ฟอร์มบันทึกตาม criteria type:
   - Numpad popup สำหรับ touch device
   - Real-time validation per cell (pass/fail color)
   - Aggregate calculation (mean, RSD, all_pass, min_max)

3. **Multi-Round Support** — บันทึกหลายรอบต่อ criteria
   - แต่ละรอบมี timestamp + reason
   - รอบที่ submitted ถูก lock (ดูได้แต่แก้ไม่ได้)
   - Audit trail สำหรับการปรับเครื่อง/recheck

4. **Tare Subtraction (Multi-Point)** — ถ้า criteria ผูก tareSource:
   - ดึง mean จากรอบ submitted ล่าสุดของ Tare criteria
   - แต่ละ cell แสดงทั้ง Gross และ Net (= Gross - Tare)
   - เช็ค spec กับ **Net weight** ไม่ใช่ Gross
   - ถ้ายังไม่มี Tare → แสดง warning banner

5. **Submit Validation** — ปุ่มส่งผล disabled จนกว่าจะกรอกครบตาม type

---

## 💾 Data Structures (อ้างอิงจาก localStorage)

### Key: `ipc_criteria_list`

Array ของ criteria objects:

```typescript
type Criteria = {
  id: string;                    // "crit_<timestamp>"
  createdAt: string;             // ISO 8601
  code: string;                  // เช่น "IPC-CAP-TARE-001"
  testName: string;              // dropdown value
  testLabel: string;             // display label
  unit: string;                  // "g" | "mg" | "mm" | ...
  dosageForm: string;            // "capsule" | "tablet" | ...
  criteriaType: CriteriaType;    // หนึ่งใน 9 types
  critical: boolean;
  active: boolean;

  // type-specific fields (อันที่ไม่ใช่ type นั้น = ค่าว่าง/default)
  target?: string;
  tolerance?: string;
  passDefinition?: string;
  failDefinition?: string;
  defaultExpected?: 'pass' | 'fail';
  visualChecklist?: string[];
  visualCriteria?: string;
  visualReference?: string;
  textFormat?: string;
  textExample?: string;
  textRequired?: boolean;
  customFields?: CustomField[];
  customNote?: string;
  multiPoint?: MultiPointSpec;
  calibration?: CalibrationSpec;
  tare?: TareSpec;
  calculated?: CalculatedSpec;

  // shared
  stages: Stage[];                    // Multi-stage acceptance
  samplingMethod: string;             // "random" | "systematic" | "stratified" | "census" | ...
  useContext: string[];               // ["routine", "release", "validation", ...]
  sopStepRef: SopStepRef;
  derivedCalcs: DerivedCalc[];
  // triggers (omitted — see code)
};

type MultiPointSpec = {
  pointCount: string;           // "10"
  pointLabel: string;           // "หัวตอก"
  perPointTarget: string;       // "20"
  perPointTolerance: string;    // "25" (percent)
  aggregateRule: 'all_pass' | 'mean' | 'rsd' | 'min_max';
  aggregateLimit: string;       // "15-25" (auto-computed)
  tareSource: string;           // code ของ tare criteria (ว่าง = ไม่ผูก)
};

type Stage = {
  id: number;
  sampleSize: string;
  failureTolerance: string;     // percent
  onFail: 'next' | 'reject' | 'deviation';
  note?: string;
};

// ... ดู source code สำหรับ types อื่นๆ
```

### Key: `ipc_results`

Object — key = criteria.id, value = result entry:

```typescript
type ResultEntry = {
  criteriaId: string;
  criteriaCode: string;
  updatedAt: string;
  rounds: Round[];
};

type Round = {
  id: string;                   // "round_<timestamp>"
  startedAt: string;            // ISO 8601
  submittedAt: string | null;   // null = in-progress, ISO = locked
  reason: string;               // เหตุผลที่เริ่มรอบใหม่ (เช่น "หลังปรับ tamping pin")
  data: RoundData;
};

type RoundData = {
  type: CriteriaType;
  value?: string;               // numeric/pass_fail/text/calibration/tare
  points?: string[];            // multi_point
  checks?: Record<number, boolean>; // visual checklist
  values?: Record<number, string>;  // custom_multi_field
  inputs?: Record<number, string>;  // calculated
};
```

---

## 🏗 Required for Production

### Backend (must-have)

1. **Database tables (suggested):**
   - `users` (id, role, name, email, ...)
   - `criteria` (id, code, version, status, payload JSON, created_by, ...)
   - `criteria_versions` (immutable history)
   - `recording_rounds` (id, criteria_id, batch_id, started_at, submitted_at, signed_by, signature_hash, data JSON)
   - `batches` (id, batch_number, product, started_at, status, ...)
   - `deviations` (linked to rounds)
   - `audit_log` (immutable, append-only)

2. **API endpoints (REST):**
   ```
   POST   /api/auth/login
   GET    /api/criteria
   POST   /api/criteria
   PUT    /api/criteria/:id            → creates new version, never overwrites
   GET    /api/criteria/:id/versions
   POST   /api/recording/rounds
   POST   /api/recording/rounds/:id/submit  → requires e-signature
   GET    /api/recording/rounds?criteriaId=&batchId=
   GET    /api/audit-log?...
   ```

3. **Electronic signature flow:**
   - บังคับใส่ password อีกครั้งก่อน submit
   - Hash + sign timestamp + user_id + payload
   - Store signature ใน DB แยกจาก data

### Frontend (must-rewrite)

- เปลี่ยนจาก single HTML → React project (Vite/Next.js)
- Component-ize: แยก `CriteriaForm`, `RecorderByType`, `Numpad`, `RoundTabs`, etc.
- State management: React Query สำหรับ server state, Zustand/Redux สำหรับ UI state
- TypeScript strict mode
- Add tests: Vitest unit + Playwright E2E
- i18n: รองรับไทย/อังกฤษ

### Compliance / Quality

- 21 CFR Part 11 walkthrough
- GAMP 5 validation plan
- IQ/OQ/PQ documentation
- Data backup & retention policy

---

## 🧪 Testing the Prototype

```bash
# Clone
git clone https://github.com/adthaponu-creator/ERP-erbal-.git
cd ERP-erbal-

# เปิด index.html ในเบราว์เซอร์ (Windows)
start index.html

# หรือใช้ npm
npm run dev  # serves on http://localhost:5173
```

### Test scenario แนะนำ

1. **สร้าง Tare:** index.html → กรอก IPC-CAP-TARE-001 (Multi-Point, 10 จุด, Aggregate=Mean) → CREATE
2. **สร้าง WV:** กด "สร้าง Criteria ตัวใหม่" → กรอก IPC-CAP-WV-001 → เลือก Tare = TARE-001 → CREATE
3. **บันทึก Tare:** record.html → คลิก TARE-001 → กรอก 10 ค่า → ส่งผล
4. **บันทึก WV:** คลิก WV-001 → ดู Tare banner → กรอก 20 ค่า → ดู Gross/Net auto-calculate
5. **รอบใหม่:** กลับ list → "+ รอบใหม่" บนการ์ด WV → ใส่เหตุผล → บันทึกอีกรอบ

---

## 📞 Questions

- **Business logic / UX:** ติดต่อ product owner ที่ส่ง prototype นี้ให้
- **GMP/Regulatory:** ปรึกษา QA department ก่อนออกแบบ audit trail
- **Reference standards:**
  - USP `<711>` Dissolution
  - USP `<905>` Uniformity of Dosage Units
  - USP `<701>` Disintegration
  - USP `<1216>` Tablet Friability
  - ICH Q9 — Quality Risk Management
  - ICH Q10 — Pharmaceutical Quality System
  - PIC/S Annex 8 — Sampling

---

## License

Prototype นี้ใช้เป็น reference ภายในเท่านั้น — ไม่เผยแพร่สาธารณะ
