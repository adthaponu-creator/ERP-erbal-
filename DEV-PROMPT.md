# 📋 Dev Brief — แก้ไขหน้า IPC Criteria ตาม UX/UI ใหม่

> เอกสารนี้สำหรับ developer ที่จะแก้ไขระบบเดิมที่ https://herbal-erp-more.bmscloud.in.th/master-data/ipc-criteria/new ให้ตรงตาม UX/UI ใหม่

---

## 1. สรุปงานในประโยคเดียว

แก้หน้า **"สร้าง IPC Criteria ใหม่"** เดิม และ **เพิ่มหน้าใหม่ "บันทึกผล IPC (Operator)"** ที่ยังไม่มีในระบบ ตาม UX flow ใหม่ — รองรับ 9 criteria types, Multi-stage acceptance, Multi-trigger, Tare linkage, และ Multi-round recording

---

## 2. Scope การแก้ไข

### 2.1 หน้าที่ต้องแก้ (มีอยู่แล้ว)
- **`/master-data/ipc-criteria/new`** — หน้าสร้าง IPC Criteria
- **`/master-data/ipc-criteria/[id]/edit`** — ถ้ามี ก็ใช้ฟอร์มเดียวกัน

### 2.2 หน้าที่ต้องสร้างใหม่
- **`/ipc-recording`** — หน้า list ของ criteria ที่ตั้งไว้ (สำหรับ operator)
- **`/ipc-recording/[criteriaId]`** — หน้าบันทึกผล per criteria + multi-round

### 2.3 หน้าที่อาจเกี่ยวข้อง
- Sidebar navigation — เพิ่มเมนู "IPC Recording" ใต้กลุ่ม Production

---

## 3. UX Flow ภาพรวม

```
┌──────────────────────────────┐         ┌──────────────────────────────┐
│  หน้า สร้าง IPC Criteria      │         │  หน้า บันทึกผล (Operator)    │
│  (QA / Admin)                │         │                              │
│                              │         │                              │
│  1. กรอกฟอร์ม                │  CREATE │  1. เห็น list ของ criteria   │
│  2. เลือก criteria type      │ ──────► │  2. เลือกตัวที่จะบันทึก       │
│  3. ตั้ง spec, sampling,     │         │  3. กรอกค่าตามแบบ            │
│     trigger, stage           │         │  4. ส่งผล (lock รอบนั้น)     │
│  4. กด CREATE                │         │  5. เริ่มรอบใหม่ได้ถ้าต้อง    │
└──────────────────────────────┘         └──────────────────────────────┘
```

---

## 4. Page 1: "สร้าง IPC Criteria ใหม่"

### 4.1 Layout (Desktop)
```
┌────────────────────────────────────────────────────────────────┐
│ [Sidebar]  Master Data / IPC Criteria / New          [Draft]   │
├────────────────────────────────────────────────────────────────┤
│ [Sidebar]  ┌──────────────────────────┐  ┌──────────────────┐ │
│            │  Criteria Information    │  │  Live Preview ●  │ │
│            │  (กรอบฟอร์มหลัก)         │  │  (sticky panel)  │ │
│            │                          │  │                  │ │
│            │  1️⃣ ข้อมูลพื้นฐาน        │  │  IPC-CODE-001    │ │
│            │  2️⃣ เกณฑ์มาตรฐาน        │  │  ─────────────   │ │
│            │  3️⃣ Sampling Plan       │  │  Test Name       │ │
│            │  4️⃣ Multi-Stage         │  │  Type · Unit     │ │
│            │  5️⃣ ตรวจสอบเมื่อ        │  │                  │ │
│            │  6️⃣ Derived Calc        │  │  ACCEPTANCE FLOW │ │
│            │  7️⃣ Settings            │  │  S1 → S2 → ...   │ │
│            │                          │  │                  │ │
│            │                          │  │  GMP COMPLIANCE  │ │
│            │                          │  │  ✓ 7/7 checks    │ │
│            └──────────────────────────┘  └──────────────────┘ │
│            ┌──────────────────────────────────────────────┐   │
│            │ Sticky footer: [CANCEL]  [CREATE]            │   │
│            └──────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

### 4.2 Responsive
- **Mobile (<640px)**: single column, sidebar = hamburger drawer, preview ลงล่างสุด
- **Tablet (640-1280px)**: 2 columns, preview ลงล่าง
- **Desktop (1280px+)**: 2 columns + preview ขวา sticky

### 4.3 Section 1: ข้อมูลพื้นฐาน (Basic Information)

| Field | Type | Required | Note |
|---|---|---|---|
| **หัวข้อการทดสอบ (Test Name)** | Searchable dropdown + add-new modal | ✓ | มี 25+ ตัวเลือกมาตรฐาน USP, รองรับค้นหา keyboard navigation (↑↓ Enter Esc) |
| Code | Text | ✓ | Auto-fill จาก Test Name (format: `IPC-{prefix}-{seq}`) — แก้ได้ |
| Unit | Searchable dropdown + add-new | — | มี 20+ หน่วยมาตรฐาน (mg, g, mm, ml, °C, pH, count, ฯลฯ) |
| รูปแบบยา (Dosage Form) | Searchable dropdown + add-new | — | มี 29 รูปแบบ (Tablet, Capsule, Tincture, Decoction, Balm, Tea Bag, ฯลฯ) |
| **ประเภทเกณฑ์ (Criteria Type)** | Radio cards (large) | ✓ | 9 ตัวเลือก — เลือกแล้วฟอร์ม section 2 จะเปลี่ยนตามประเภท |
| Use Context | Multi-select chip | — | `routine` / `release` / `validation` / `troubleshoot` / `changeover` / `cleaning_verify` / `equipment_qual` |
| อ้างอิง SOP/BPR | Nested form | — | sopCode, sopVersion, stepNumber, stepDescription, link |

#### Smart Auto-fill behavior
เมื่อเลือก Test Name จาก list มาตรฐาน → ระบบ auto-fill ดังนี้ พร้อมแสดง badge `⚡ Auto` บนช่องที่ถูกเติม + แสดง notification "เติมให้อัตโนมัติ: Code · Unit · Sample Plan":
- `Code` = `IPC-{prefix}-{seq}` (เช่น Weight Variation → `IPC-WV-742`)
- `Unit` = หน่วยมาตรฐานตาม test
- `Sample Size`, `Tolerance`, `Stages` = USP recommended
- `Critical` = true สำหรับ Sterility, Heavy Metals, Assay

#### Add-new option modal
ทุก dropdown รองรับเพิ่มตัวเลือกใหม่ — เมื่อกด **"＋ เพิ่มตัวเลือกใหม่..."** ที่ปลาย dropdown → popup modal กลางจอ:
- มี field name, description, reference, ฯลฯ ตาม dropdown
- Keyboard: Enter = บันทึก, Esc = ยกเลิก

### 4.4 Section 2: เกณฑ์มาตรฐาน (Specification) — เปลี่ยนตาม Criteria Type

#### 9 Criteria Types ที่ต้องรองรับ

แต่ละ type มีฟอร์มเฉพาะ พร้อม banner สีต่างกัน:

##### (1) Numeric — ค่าตัวเลข ± Tolerance
- Target, ±% Tolerance
- Min/Max readonly คำนวณจาก `Target ± Tolerance%` แสดงเป็นแถบสรุป `285 ≤ value ≤ 315 (300 ± 5%)`

##### (2) Pass / Fail
- `passDefinition` (เกณฑ์ผ่าน)
- `failDefinition` (เกณฑ์ไม่ผ่าน)
- `defaultExpected` = `pass` | `fail`

##### (3) Visual
- `visualCriteria` (คำอธิบายเกณฑ์)
- `visualChecklist` (array ของรายการที่ต้องตรวจ) — add/remove ได้
- `visualReference` (รูปอ้างอิง — URL field)

##### (4) Text
- `textFormat` (รูปแบบ)
- `textExample` (ตัวอย่าง)
- `textRequired` (boolean)

##### (5) Multi-Point Numeric ⭐ สำคัญ
- `pointCount` — จำนวนจุดที่วัด (≥ 2)
- `pointLabel` — คำนำหน้า เช่น "หัวตอก" → ตอนบันทึกเห็น "หัวตอก 1", "หัวตอก 2"...
- `perPointTarget` — Target ต่อจุด
- `perPointTolerance` — ±% Tolerance ต่อจุด
- `aggregateRule` — 4 ทางเลือก:
  - `all_pass` — ทุกจุดต้องผ่าน
  - `mean` — ค่าเฉลี่ยอยู่ในช่วง
  - `rsd` — ความเบี่ยงเบนสัมพัทธ์ ≤ ค่าหนึ่ง
  - `min_max` — ทุกจุดอยู่ในช่วง min-max
- `aggregateLimit` — **คำนวณอัตโนมัติจาก Target ± Tolerance** สำหรับ mean/min_max (read-only แสดงเป็นแถบเขียวอ่อนพร้อม badge "⚡ คำนวณจาก Target ± Tolerance"); ส่วน rsd ให้กรอกเอง
- **⚖️ Tare subtraction** — กล่องสีฟ้า มี dropdown เลือก Tare criteria (filter เฉพาะ Multi-Point + aggregateRule=mean) → ค่า code ของ Tare ที่เลือก เก็บใน `tareSource`

##### (6) Custom Multi-Field
- Array ของ `customFields` — แต่ละ field มี: label, type (number/text/select), unit, target, tolerance, required, note
- มีปุ่ม "⚡ สร้างชุดฟิลด์อัตโนมัติ" — กรอก prefix + count → generate field 1 ถึง N (เช่น "Punch 1" ถึง "Punch 30")

##### (7) Calibration Check
- instrumentName, instrumentId
- standardValue, standardUnit
- toleranceType (`absolute` | `percent`), toleranceValue
- lastCalibrationDate, nextDueDate
- requiresPriorPass (boolean)

##### (8) Tare / Reference
- referenceLabel (เช่น "น้ำหนักภาชนะเปล่า")
- referenceUnit
- storeAs — ชื่อ symbolic ใช้อ้างอิงในสูตรอื่น
- acceptanceMin, acceptanceMax
- expireAfter (`batch` | `shift` | `permanent`)

##### (9) Calculated (formula-based)
- formula — text input รับสูตรเช่น `(gross - tare) / batch_size * 100`
- inputs[] — array ของ input ที่ใช้ในสูตร แต่ละตัวมี: name, source (`this_step` | `derived` | `constant`), fieldRef/criteriaCode
- resultUnit, resultMin, resultMax
- displayDecimals (default 2)
- มีตัวอย่าง: `Net = gross - tare` · `%LOD = (wet - dry) / wet * 100` · `Yield = output / input * 100`

### 4.5 Section 3: Sampling Plan
- Sampling Method — dropdown + add-new (random, systematic, stratified, census, composite)
- Sampling Note — text

### 4.6 Section 4: Multi-Stage Acceptance Criteria ⭐
- Panel แยก กรอบเส้นประสีเขียว มี tags `USP <711>`, `USP <905>`, `USP <701>` มุมขวา
- Stage list (1-3 stages, สามารถ add/remove ได้):
  - Sample Size
  - Failure Tolerance (%)
  - On Fail action: `next` (next stage) | `reject` (reject batch) | `deviation` (บันทึก deviation)
  - คำนวณอัตโนมัติแสดง: "ทดสอบ N ชิ้น · ยอมเสียได้ M ชิ้น · ต้องผ่าน (N-M) ชิ้น" — สูตร `floor(SampleSize × Tolerance%)` แยกต่อ stage
- ถ้า onFail = `deviation` → แสดง **Deviation preview card** (DEV-2026-XXXX, Severity, Status, Assignee)

### 4.7 Section 5: ตรวจสอบเมื่อ (Multi-Trigger Inspection) ⭐
Panel กรอบเขียวอ่อน หัวข้อ "ตรวจสอบเมื่อ — เลือกได้หลายแบบ" + chip "เปิด N/5"

5 trigger types ใช้พร้อมกันได้ แต่ละ trigger มี toggle เปิด/ปิด:

1. **⏱ ตามช่วงเวลา (Time-based)** — `every` (นาที)
2. **📦 ตามจำนวนผลิต (Quantity-based)** — `every`, `unit`, `mode` (`fixed` | `percent`), `percent`
3. **🎯 ที่ milestone ของกระบวนการ** — checkboxes:
   - เริ่ม batch / กลาง batch / ก่อนปิด batch / ก่อน compression / หลัง coating
   - มีปุ่ม "+ เพิ่มตัวเลือก"
4. **⚡ เมื่อมีเหตุการณ์ (Event-triggered)** — checkboxes:
   - หลัง changeover / หลังเปลี่ยน lot วัตถุดิบ / หลัง equipment cleaning / หลัง parameter change / หลัง maintenance
5. **🔄 ครั้งเดียวต่อ batch (Once per batch)** — toggle เปิด/ปิด

### 4.8 Section 6: Derived Calculations (Cross-step)
- Array ของ derived calcs — แต่ละตัวมี: label, formula, sources[], resultUnit, triggerWhen, acceptanceMin/Max, onFail, note

### 4.9 Section 7: Settings
- `critical` — toggle card "Critical" (ถ้าเปิด → operator ใช้เครื่องไม่ได้จนกว่าจะ pass)
- `active` — toggle card "Active"

### 4.10 Live Preview Panel (sticky ขวามือ)
- Header: "Live Preview ●" + indicator จุดเขียวกระพริบ
- แสดง real-time:
  - Code + Critical badge
  - Test Name
  - Dosage Form
  - Criteria Type spec summary
  - **ACCEPTANCE FLOW** — diagram S1 → S2 → S3 พร้อม sample size, tolerance, action
  - **VALIDATION** — ✓ Clean หรือแสดง error/warning issues
  - **GMP COMPLIANCE** — checklist:
    - Test name (USP / Pharmacopoeia)
    - Unique Code (traceability)
    - Multi-stage testing (USP <711>/<905>)
    - Fail-action defined (ICH Q9)
    - Sampling trigger (PIC/S Annex 8)
    - Acceptance criteria specified
    - Active for batch production

### 4.11 Sticky Footer
- ซ้าย: status indicator + พร้อมบันทึก/⚠ ยังไม่ได้เลือก Test Name
- ขวา: [CANCEL] [CREATE] (disabled ถ้าไม่มี testName/code หรือมี validation error)

### 4.12 หลังกด CREATE → Modal กลางจอ มี 3 ทางเลือก:
1. **[+ สร้าง Criteria ตัวใหม่ (ฟอร์มล้าง)]** ← สีเขียวเด่น — ล้างฟอร์ม + scroll top + refresh dropdown
2. **[👷 ไปหน้าบันทึกผล (Operator) →]** — navigate ไป recording page
3. **[อยู่หน้านี้ (แก้ไขต่อ)]** — text button, ปิด modal

### 4.13 Validation Rules
- Show error/warning count บนปุ่ม CREATE
- Field-level: required marker `*`
- Cross-field: เช่น Multi-Point + RSD ต้องมี pointCount ≥ 3
- Disable CREATE ถ้ามี error

---

## 5. Page 2: "บันทึกผล (Operator)" — หน้าใหม่

### 5.1 Empty State (ยังไม่มี criteria)
- Icon ตรงกลาง + ข้อความ "ยังไม่มี IPC Criteria"
- ปุ่ม "+ สร้าง IPC Criteria" — link ไป `/master-data/ipc-criteria/new`

### 5.2 List View
```
┌──────────────────────────────────────────────────────┐
│ IPC RECORDING                                         │
│ เลือกเกณฑ์ที่ต้องการบันทึกผล                          │
│ มี N เกณฑ์ที่ตั้งค่าไว้    [+ สร้าง Criteria ใหม่]   │
├──────────────────────────────────────────────────────┤
│ ┌──────────────────┐  ┌──────────────────┐          │
│ │ IPC-CAP-TARE-001 │  │ IPC-CAP-WV-001   │          │
│ │ MULTI-POINT      │  │ MULTI-POINT      │          │
│ │                  │  │ ! CRITICAL       │          │
│ │ ✓ 1 รอบ          │  │ ✓ 2 รอบ          │          │
│ │ น้ำหนักแคปซูลเปล่า│ │ Weight Variation │          │
│ │ หลายจุด · g       │  │ หลายจุด · g      │          │
│ │ BPR · 8.1        │  │ BPR · 8.3        │          │
│ │ [ดูรอบ→] [+รอบใหม่]│ │[ดูรอบ→] [+รอบใหม่]│          │
│ └──────────────────┘  └──────────────────┘          │
└──────────────────────────────────────────────────────┘
```

แต่ละการ์ดมี:
- Code (mono badge)
- Criteria type chip (สีตาม type)
- Critical badge ถ้า critical
- Status badges:
  - `✓ N รอบ` (เขียว) — จำนวนรอบที่ submitted
  - `• กำลังบันทึก` (เหลือง) — มีรอบ in-progress
- Title (testLabel)
- Subtitle (type description + unit)
- SOP step badge ถ้ามี
- Action bar ด้านล่าง 2 ปุ่ม:
  - **ดำเนินการต่อ / เริ่มบันทึก / ดูรอบทั้งหมด →** (label เปลี่ยนตามสถานะ)
  - **+ รอบใหม่** (สีเขียว) → เปิด modal กรอกเหตุผล → สร้างรอบใหม่ + เข้าหน้า record

### 5.3 Record View — Recorder per Type

#### Header
- ปุ่ม `← รายการ` (กลับ list)
- Code badge + Type chip + Critical badge
- Title (testLabel)

#### Round Tabs (ด้านบนของฟอร์ม)
- "รอบการบันทึก (M/N ส่งแล้ว)" + ปุ่ม **[+ บันทึกรอบใหม่]**
- Horizontal scrollable tabs ของรอบทั้งหมด:
  - **รอบที่ active** = สีเขียวเข้ม (highlighted)
  - **✓ ส่งแล้ว** = สีเขียวอ่อน
  - **• กำลังบันทึก** = สีเหลือง
  - แต่ละ tab แสดง: "รอบ N · HH:MM · status · เหตุผล: ..."

#### Locked Banner (ถ้ารอบที่ active = submitted)
```
🔒 รอบนี้ส่งผลไปแล้วเมื่อ DD/MM/YYYY HH:MM:SS — ดูได้แต่แก้ไขไม่ได้
   ถ้าต้องการบันทึกใหม่ กดปุ่ม + บันทึกรอบใหม่ ด้านบน
```

#### Instructions Box
- ถ้ามี `sopStepRef.stepDescription` → แสดงเป็น sky-blue box
- ถ้า critical → แสดง critical warning ใต้

#### Recorder Forms (เปลี่ยนตาม criteriaType)

##### (1) Numeric / Calibration / Tare
- Spec banner (gradient green) แสดงช่วงที่ยอมรับ
- ปุ่มใหญ่ใจกลาง — กดแล้วเปิด **Numpad popup** (touch-friendly)
- แสดงค่า + status (✓ ผ่าน / ✕ นอกเกณฑ์)

##### (2) Pass/Fail
- การ์ดเกณฑ์ผ่าน (เขียว) + การ์ดเกณฑ์ไม่ผ่าน (แดง)
- 2 ปุ่มใหญ่ขนาดเท่ากัน: **PASS** (เขียว) / **FAIL** (แดง)

##### (3) Visual
- การ์ดเกณฑ์การตรวจ
- Checklist tap-toggle — แต่ละ item เป็นปุ่มกดติ๊กได้
- แสดงสถานะ N/M items + "✓ ครบทุกข้อ" ถ้าครบ

##### (4) Text
- การ์ดรูปแบบ + ตัวอย่าง (ถ้ามี)
- Textarea
- Character count

##### (5) Multi-Point Numeric ⭐
- **Tare Banner** (ถ้า `tareSource` มีค่า):
  - 🟦 สีฟ้า: `⚖️ Tare = X.XXXX g (จาก IPC-CAP-TARE-001 รอบล่าสุด) · Net = Gross − X.XXXX g`
  - 🟡 สีเหลือง (error state): `⚠️ ต้องมีค่า Tare ก่อนบันทึก — {ไม่พบ criteria / ยังไม่มีรอบที่ส่งผล}`
- Spec banner (teal gradient) — ถ้าใช้ tare ระบุว่า "ค่า NET ที่ยอมรับต่อจุด"
- Progress bar (filled/total)
- **Grid ของ cells** — 2-3 columns:
  - แต่ละ cell tap-large, กดเปิด numpad
  - ถ้าไม่ใช้ tare: แสดงค่าเดียว + ✓/✕
  - **ถ้าใช้ tare:**
    ```
    ┌──────────────────┐
    │ แคปซูล 1     ✓   │
    │ Gross  0.5969    │
    │ Net    0.5009    │
    └──────────────────┘
    ```
  - สี: เทา (ว่าง) / เขียว (pass) / แดง (fail)
- **Aggregate Result** (ขึ้นเมื่อกรอกครบ pointCount):
  - แสดง Mean (Gross) + Mean (Net) ถ้าใช้ tare
  - Status: ✓ ผ่าน / ✕ ไม่ผ่าน
  - คำนวณตาม `aggregateRule`

##### (6) Custom Multi-Field
- การ์ดแต่ละ field — ตาม type:
  - `number` → ปุ่มเปิด numpad
  - `text` → input
- Required marker `*`
- Target/Tolerance hint แสดง

##### (7) Calculated
- การ์ดแต่ละ input → numpad
- Aggregate Result auto: คำนวณตาม formula ด้วย `new Function()` — แสดง result + status

#### Submit Flow
- **Sticky footer** ล่างจอ:
  - ซ้าย: "บันทึกอัตโนมัติ HH:MM:SS" indicator
  - กลาง: ถ้ายังไม่ครบ → "⚠ เหตุผล" (เช่น "กรอกแล้ว 5/10 จุด — ยังไม่ครบ")
  - ขวา: ปุ่ม **[ส่งผล]** disabled ถ้ายังไม่ครบ; แสดง "ส่งแล้ว" + disabled ถ้า locked

- **Validation rules** ก่อนกดส่ง:
  - Numeric/Calibration/Tare: ค่าไม่ว่าง + เป็นตัวเลข
  - Pass/Fail: เลือกแล้ว
  - Visual: ติ๊กครบทุกข้อ
  - Text: ไม่ว่าง
  - Multi-Point: กรอกครบ pointCount
  - Custom Multi-Field: required fields ครบ
  - Calculated: input ครบทุกตัว

- **Submit confirm modal** หลังกดส่งสำเร็จ:
  - หัว: "ส่งผล รอบ N เรียบร้อย"
  - ตัวเลือก:
    - **[+ เริ่มรอบใหม่ทันที]** (เด่น) → เปิด new-round modal
    - **[กลับรายการ]**
    - **[ปิด]**

#### Numpad Popup
- Fullscreen overlay สีดำโปร่ง
- Card สีขาว max-w-md กลางจอ (mobile: bottom sheet)
- Header สีเขียว: "กำลังกรอก: {label}" + ปุ่มปิด
- Display ค่าปัจจุบัน + unit
- Grid 4×4: `7 8 9 . | 4 5 6 0 | 1 2 3 ⌫ | C ✓บันทึก ✓บันทึก`
- Decimal toggle ตาม `allowDecimal` (default true; unit = "count" → false → แทนด้วยปุ่ม ± สำหรับ negative)
- Tap-friendly (≥ 64px buttons)

#### New Round Modal
- "เริ่มรอบบันทึกใหม่"
- Subtitle: code + testLabel
- Textarea: "เหตุผล (ไม่บังคับ)" — placeholder: "เช่น น้ำหนักแคปซูลเบี่ยงเบนเกิน → ปรับหัวตอกใหม่"
- Hint: "รอบเก่าจะถูกล็อกไว้ (ดูได้แต่แก้ไขไม่ได้) เพื่อรักษา audit trail"
- ปุ่ม: [ยกเลิก] [เริ่มรอบใหม่]

---

## 6. Data Model ที่ต้องเก็บ

### 6.1 Criteria (master data)

```typescript
type Criteria = {
  id: string;
  code: string;                  // unique
  createdAt: Date;
  createdBy: User;
  status: 'DRAFT' | 'ACTIVE' | 'ARCHIVED' | 'RETIRED';
  active: boolean;
  critical: boolean;

  // versioning — ต้องเก็บประวัติทุกครั้งที่แก้
  versions: CriteriaVersion[];
  currentVersionId: string;
};

type CriteriaVersion = {
  id: string;
  versionNumber: number;
  type: CriteriaType;            // 9 enum values
  testName: string;
  testLabel: string;
  unit?: string;
  dosageForm?: string;

  // type-specific configs (optional ตาม type)
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
  stages: Stage[];               // multi-stage acceptance
  samplingMethod: string;
  useContext: string[];
  sopStepRef: SopStepRef;
  triggers: Triggers;
  derivedCalcs: DerivedCalc[];

  approvedAt?: Date;
  approvedBy?: User;
  createdAt: Date;
};

type MultiPointSpec = {
  pointCount: string;
  pointLabel: string;
  perPointTarget: string;
  perPointTolerance: string;
  aggregateRule: 'all_pass' | 'mean' | 'rsd' | 'min_max';
  aggregateLimit: string;        // auto-computed for mean/min_max
  tareSource: string;            // code ของ Tare criteria (FK ผ่าน code)
};

type Stage = {
  id: number;
  sampleSize: string;
  failureTolerance: string;
  onFail: 'next' | 'reject' | 'deviation';
  note?: string;
};

type Triggers = {
  time?: { on: boolean; every: string };
  quantity?: { on: boolean; every: string; unit: string; mode: 'fixed' | 'percent'; percent: string };
  milestone?: { on: boolean; options: { id: string; label?: string; on: boolean }[] };
  event?: { on: boolean; options: { id: string; label?: string; on: boolean }[] };
  oncePerBatch?: { on: boolean };
};

type SopStepRef = {
  sopCode: string;
  sopVersion: string;
  stepNumber: string;
  stepDescription: string;
  link?: string;
};
```

### 6.2 Recording Round

```typescript
type RecordingRound = {
  id: string;
  criteriaId: string;            // FK
  criteriaVersionId: string;     // pin ไปที่ version ตอนบันทึก
  batchId?: string;              // FK (ถ้ามี batch system)
  roundIndex: number;            // 1, 2, 3, ...
  reason?: string;               // เหตุผลที่เริ่มรอบใหม่
  startedAt: Date;
  startedBy: User;
  data: RoundData;               // เนื้อหาผล ตาม type

  // Submit info (immutable หลัง set)
  submittedAt?: Date;
  submittedBy?: User;
  signatureHash?: string;        // HMAC ของ data+user+timestamp
  signatureMethod?: 'PASSWORD' | 'PIN' | 'OAUTH_REAUTH';

  // Denormalized for query
  passed?: boolean;
  computedMean?: number;
  outcomeNote?: string;
};

type RoundData = {
  type: CriteriaType;
  value?: string;                // numeric/pass_fail/text/calibration/tare
  points?: string[];             // multi_point
  checks?: Record<number, boolean>;  // visual
  values?: Record<number, string>;   // custom_multi_field
  inputs?: Record<number, string>;   // calculated
};
```

### 6.3 Tare Linkage Logic (สำคัญ)

เมื่อ operator บันทึก criteria ที่มี `multiPoint.tareSource = "IPC-CAP-TARE-001"`:

1. **Query latest submitted round** ของ Tare criteria (filter: `criteriaId where code=tareSource`, `submittedAt != null`, order `submittedAt DESC`, limit 1)
2. ดึง `data.points` → คำนวณ mean
3. ถ้าไม่พบ → แสดง warning banner (operator ยังบันทึกได้แต่ไม่มี net)
4. ถ้าพบ → แสดง Tare banner + ในแต่ละ cell แสดง Gross + Net (Net = Gross − tareMean) + เช็ค spec กับ Net

### 6.4 Audit Log (21 CFR Part 11)

ทุก action บน Criteria + RecordingRound ต้องเขียน audit log:

```typescript
type AuditLog = {
  id: string;
  timestamp: Date;
  userId: string;
  action: 'CREATE' | 'UPDATE' | 'SUBMIT' | 'SIGN' | 'APPROVE' | 'ARCHIVE' | ...;
  entityType: 'Criteria' | 'RecordingRound' | 'Deviation' | ...;
  entityId: string;
  changes?: { before: unknown; after: unknown };
  reason?: string;               // GMP requirement
  ipAddress?: string;
  userAgent?: string;
};
```

**กฎ:** ห้าม UPDATE/DELETE row — append only

---

## 7. API Endpoints ที่ต้องสร้าง

```
# Auth
POST   /api/auth/login
POST   /api/auth/logout
POST   /api/auth/verify          # re-auth สำหรับ e-signature

# Criteria
GET    /api/criteria             # list (filter: status, type, active)
GET    /api/criteria/:id         # detail + versions
POST   /api/criteria             # create new (creates version 1)
PUT    /api/criteria/:id         # update → creates new version (ห้าม overwrite)
GET    /api/criteria/:id/versions
POST   /api/criteria/:id/archive

# Recording
GET    /api/recording/rounds?criteriaId=&batchId=&status=
POST   /api/recording/rounds                    # start new round
PUT    /api/recording/rounds/:id                # update in-progress data
POST   /api/recording/rounds/:id/submit         # require password + create signature

# Tare lookup (helper)
GET    /api/criteria/by-code/:code/latest-round  # for tare subtraction

# Audit
GET    /api/audit-log?entityType=&entityId=&userId=
```

---

## 8. Compliance Requirements (GMP / 21 CFR Part 11)

ก่อน deploy ต้องคุยกับ QA/Regulatory ว่า:

- [ ] Electronic signature workflow — ขอ password อีกครั้งก่อน submit?
- [ ] Audit trail retention period (กี่ปี?)
- [ ] User roles & permissions matrix
- [ ] Backup/restore procedure
- [ ] System validation: IQ/OQ/PQ documents
- [ ] Data integrity (ALCOA+)
  - **A**ttributable — ใครทำ
  - **L**egible — อ่านได้
  - **C**ontemporaneous — บันทึก ณ ขณะนั้น
  - **O**riginal — ต้นฉบับ
  - **A**ccurate — ถูกต้อง
  - **+** Complete, Consistent, Enduring, Available

---

## 9. References

มาตรฐานที่ใช้:
- **USP `<711>`** — Dissolution
- **USP `<905>`** — Uniformity of Dosage Units
- **USP `<701>`** — Disintegration
- **USP `<1216>`** — Tablet Friability
- **ICH Q8** — Pharmaceutical Development (CQA)
- **ICH Q9** — Quality Risk Management
- **ICH Q10** — Pharmaceutical Quality System
- **PIC/S Annex 8** — Sampling
- **21 CFR Part 11** — Electronic Records & Signatures
- **GAMP 5** — System validation

---

## 10. Test Scenarios (Critical Path)

Dev ต้อง verify ทุก scenario นี้ผ่านก่อนส่งงาน:

### Scenario 1: สร้าง Tare + WV และผูกกัน
1. สร้าง `IPC-CAP-TARE-001` — Multi-Point, 10 จุด, Aggregate=Mean → CREATE
2. กด "สร้าง Criteria ตัวใหม่" → สร้าง `IPC-CAP-WV-001` — Multi-Point, 20 จุด, Aggregate=All Pass, **เลือก Tare = TARE-001**
3. ตรวจ DB: WV-001.multiPoint.tareSource = "IPC-CAP-TARE-001"

### Scenario 2: บันทึก Tare → WV ใช้ค่าหัก
1. เปิดหน้า recording → คลิก TARE-001 → กรอก 10 ค่า (0.0958, 0.0962, ...) → ส่งผล
2. คลิก WV-001 → ดู Tare banner: "⚖️ Tare = 0.0960 g (จาก TARE-001 รอบล่าสุด)"
3. กรอก Gross 0.5969 → cell แสดง Net = 0.5009 → ✓ pass (เพราะอยู่ใน 0.4625–0.5375)

### Scenario 3: Multi-Round
1. WV-001 รอบ 1 → กรอก 20 ค่า → ส่งผล (lock)
2. ที่ list → กด "+ รอบใหม่" บนการ์ด WV-001 → กรอกเหตุผล "ปรับ tamping pin" → เริ่ม
3. รอบ 1 ต้อง lock (แก้ไม่ได้) — รอบ 2 active
4. ตรวจ DB: 2 RecordingRound rows, roundIndex = 1 และ 2

### Scenario 4: Validation
1. WV-001 รอบใหม่ — กรอกแค่ 5/20 ช่อง
2. ปุ่ม "ส่งผล" ต้อง disabled + แสดง "⚠ กรอกแล้ว 5/20 จุด — ยังไม่ครบ"

### Scenario 5: Tare ไม่มี → Warning
1. ลบ TARE-001 หรือยังไม่ submit รอบไหน
2. เปิด WV-001 → ต้องขึ้น warning banner สีเหลือง

### Scenario 6: Trigger Multi-select
1. สร้าง criteria → ที่ section "ตรวจสอบเมื่อ" → เปิด Time-based + Milestone (start batch) + Event (changeover)
2. ตรวจ DB: triggers.time.on=true, triggers.milestone.on=true (start batch option on=true), triggers.event.on=true (changeover on=true)
3. Live Preview แสดงทั้ง 3 triggers

### Scenario 7: Auto-computed Mean Range
1. Multi-Point spec: Target=0.5, Tolerance=7.5%, Aggregate=Mean
2. ฟิลด์ Mean range ต้องแสดง `0.4625 - 0.5375` readonly พร้อม badge "⚡ คำนวณจาก Target ± Tolerance"
3. เปลี่ยน Tolerance เป็น 10 → field อัปเดต `0.45 - 0.55` อัตโนมัติ

### Scenario 8: Audit Trail
1. สร้าง criteria → ตรวจ audit_log: action=CREATE
2. แก้ไข → audit_log: action=UPDATE + changes (before/after) + version ใหม่ใน criteria_versions
3. Submit recording → audit_log: action=SUBMIT + signature_hash ถูก set

---

## 11. UI/UX Detail Reference (Visual)

หากต้องการดู visual reference ของ UI ที่ออกแบบ — ผมมี HTML prototype ที่รันได้ในเบราว์เซอร์ (single-file React + Tailwind via CDN) ขอจากผู้ส่งงานได้ ใช้เป็น clickable mockup เพื่อเข้าใจ interaction และ layout

**ไฟล์:**
- `index.html` — หน้า "สร้าง IPC Criteria ใหม่" (≈3,400 บรรทัด)
- `record.html` — หน้า "บันทึกผล" (≈1,300 บรรทัด)

วิธีดู: เปิดไฟล์ในเบราว์เซอร์ตรงๆ (ไม่ต้องลง dependencies)

---

## 12. คำถามที่ควรถามก่อนเริ่มงาน

1. **Batch system มีในระบบเดิมหรือยัง?** ถ้ามี → ผูก RecordingRound กับ batchId; ถ้ายัง → ทำงานแบบ standalone ก่อน
2. **User roles ตอนนี้แบ่งยังไง?** Operator/QA/Admin/Manager หรืออื่น?
3. **Electronic signature policy** — บังคับกรอก password ก่อน submit หรือใช้ session-based?
4. **Deviation system** — มีระบบจัดการ deviation อยู่แล้วหรือต้องสร้างใหม่?
5. **Multi-language** — รองรับไทย/อังกฤษ หรือไทยอย่างเดียว?
6. **ระบบเดิมใช้ tech stack อะไร?** (Framework, ORM, DB) — เพื่อให้ port ได้ตรงสไตล์

---

จบเอกสาร — ถ้ามีข้อสงสัยอะไรในการแก้ไข ติดต่อผู้ส่งงานครับ
