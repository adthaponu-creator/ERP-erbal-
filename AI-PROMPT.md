# AI Prompt — Modify Existing IPC Criteria System

> **Usage:** Copy this entire file and paste it as a prompt to Claude / Cursor / Copilot / any coding AI. The AI will then have full context to modify the existing system.

---

## ROLE

You are a senior full-stack developer modifying an existing pharmaceutical ERP system (Herbal ERP) at `herbal-erp-more.bmscloud.in.th`. The current production page is `/master-data/ipc-criteria/new`. You need to:

1. **Modify** the existing "Create IPC Criteria" page to match a new UX/UI specification.
2. **Add** a new "IPC Recording (Operator)" page that doesn't exist yet.

The new design supports multiple pharmaceutical Quality Control workflows including Multi-Point Numeric with Tare subtraction (Net = Gross − Tare), Multi-stage Acceptance per USP standards, and Multi-round recording with audit trail.

---

## BEFORE YOU START — REQUIRED QUESTIONS

Ask the human these questions **before writing any code**:

1. What is the tech stack of the existing system? (Framework: React/Vue/Angular? Backend: Node/PHP/Python? ORM? DB?)
2. Where is the source code? (Local path, repo URL, or branch to work on?)
3. Does the system already have: (a) User authentication, (b) Batch system, (c) Deviation tracking, (d) Audit logging?
4. What user roles exist? (Operator / QA / Admin / Manager / others?)
5. Is multi-language needed (Thai/English)?
6. Should the new Recording page require electronic signature (re-enter password) before submit?

**Do not assume.** If any answer is unclear, ask follow-ups.

---

## CONTEXT: WHAT EXISTS NOW

The current `/master-data/ipc-criteria/new` page (based on production URL) likely has:
- Basic form: Test Name, Code, Unit, Dosage Form
- Limited criteria types
- Simple sampling plan
- Maybe basic stages

**Inspect the existing code first** before changing anything. Map existing fields to the new schema (see DATA MODEL below) to avoid losing data on migration.

---

## DELIVERABLES

### Page A: `/master-data/ipc-criteria/new` and `/master-data/ipc-criteria/[id]/edit`
Same form component, reused.

### Page B: `/ipc-recording` — new list page
### Page C: `/ipc-recording/[criteriaId]` — new record page
### Sidebar: add "IPC Recording" menu under Production group
### API: extend or add endpoints (see API SPEC below)
### DB: add new tables/columns (see DATA MODEL)
### Migration: data migration script for existing criteria records

---

## PAGE A — FORM SPEC

### Layout
- **Desktop (≥1280px):** 2-column main form + sticky right preview panel (3-col grid: 2 for form, 1 for preview)
- **Tablet (640-1280px):** 2-column form, preview below
- **Mobile (<640px):** single column, sidebar = hamburger drawer, preview at bottom

### Header (sticky top)
- Breadcrumb: `Master Data / IPC Criteria / New`
- Title: `สร้าง IPC Criteria ใหม่`
- Status chip: `● Draft`

### Main Form — 7 Sections (numbered)

#### Section 1: ข้อมูลพื้นฐาน (Basic Information)

Fields:

| Field | Type | Required | Behavior |
|---|---|---|---|
| Test Name | Searchable dropdown + Add-new modal | ✓ | 25+ USP standard options, keyboard nav (↑↓ Enter Esc), real-time search with yellow highlight |
| Code | Text | ✓ | Auto-fill from Test Name as `IPC-{prefix}-{seq3}` (e.g. Weight Variation → `IPC-WV-742`). Editable. Show `⚡ Auto` badge when auto-filled. |
| Unit | Searchable dropdown + Add-new | — | 20+ standard units (mg, g, mm, ml, °C, pH, count, percent, …). Auto-filled by Test Name. |
| Dosage Form | Searchable dropdown + Add-new | — | 29 forms including Thai herbal (Tablet, Capsule, Tincture, Decoction, Balm, Tea Bag, etc.) |
| **Criteria Type** | Large radio cards (grid) | ✓ | 9 types (see Section 2). Changes Section 2 form. |
| Use Context | Multi-select chips | — | Values: `routine`, `release`, `validation`, `troubleshoot`, `changeover`, `cleaning_verify`, `equipment_qual` |
| SOP Step Reference | Nested fieldset | — | sopCode, sopVersion, stepNumber, stepDescription, link |

**Smart Auto-fill** — when Test Name selected from standard list:
- Auto-fills: Code, Unit, Sample Size, Tolerance, Stages, Critical flag
- Shows `⚡ Auto` badge on each auto-filled field
- Shows notification: "เติมให้อัตโนมัติ: Code · Unit · Sample Plan" with dismiss button

**Add-new modal** — every dropdown has `＋ เพิ่มตัวเลือกใหม่...` at bottom. Click → opens centered modal with field-specific inputs. Keyboard: Enter=save, Esc=cancel.

#### Section 2: เกณฑ์มาตรฐาน (Specification)

Form swaps based on `criteriaType`. Each type has a colored banner indicating active type.

##### Type 1: `numeric`
- `target` (number), `tolerance` (% number)
- Auto-computed readonly: Min = target × (1−tol/100), Max = target × (1+tol/100)
- Summary banner: `285 ≤ value ≤ 315 (300 ± 5%)`

##### Type 2: `pass_fail`
- `passDefinition`, `failDefinition` (textareas)
- `defaultExpected`: radio `pass` | `fail`

##### Type 3: `visual`
- `visualCriteria` (text)
- `visualChecklist` (string array — add/remove)
- `visualReference` (URL)

##### Type 4: `text`
- `textFormat` (pattern)
- `textExample`
- `textRequired` (boolean)

##### Type 5: `multi_point` ⭐ CRITICAL FEATURE
Sub-panel with teal dashed border. Fields:
- `pointCount` (number, min 2)
- `pointLabel` (text, e.g. "หัวตอก" produces "หัวตอก 1", "หัวตอก 2"...)
- `perPointTarget`, `perPointTolerance` (%)
- `aggregateRule`: 4 button cards in grid:
  - `all_pass` — ทุกจุดต้องผ่าน
  - `mean` — เฉลี่ยอยู่ในช่วง
  - `rsd` — ความเบี่ยงเบนต่ำ (RSD limit)
  - `min_max` — ทุกจุดอยู่ในช่วง
- `aggregateLimit`:
  - For `mean` and `min_max`: **READ-ONLY display showing computed range** `{min}-{max}` derived from `perPointTarget ± perPointTolerance%`. Show emerald badge "⚡ คำนวณจาก Target ± Tolerance". Use a `useEffect` to sync `aggregateLimit` field whenever Target/Tolerance/Rule changes.
  - For `rsd`: editable text input
- `tareSource` — sub-panel with cyan border. Dropdown filtered to show only Multi-Point criteria with `aggregateRule='mean'`. Value = the selected criteria's `code`. Include refresh button.

##### Type 6: `custom_multi_field`
- `customFields[]` — array of `{ id, label, type: 'number'|'text'|'select', unit, target, tolerance, required, note }`
- "⚡ สร้างชุดฟิลด์อัตโนมัติ" generator: prefix + count → auto-creates Field 1..N (e.g. "Punch 1" .. "Punch 30")
- `customNote` (textarea)

##### Type 7: `calibration`
- `instrumentName`, `instrumentId`
- `standardValue`, `standardUnit`
- `toleranceType`: `absolute` | `percent`
- `toleranceValue`
- `lastCalibrationDate`, `nextDueDate` (dates)
- `requiresPriorPass` (boolean)

##### Type 8: `tare`
- `referenceLabel` (e.g. "น้ำหนักภาชนะเปล่า")
- `referenceUnit`
- `storeAs` (symbolic name for cross-reference)
- `acceptanceMin`, `acceptanceMax`
- `expireAfter`: `batch` | `shift` | `permanent`

##### Type 9: `calculated`
- `formula` (text input, e.g. `(gross - tare) / batch_size * 100`)
- `inputs[]` — array of `{ id, name, source: 'this_step'|'derived'|'constant', fieldRef, criteriaCode }`
- `resultUnit`, `resultMin`, `resultMax`
- `displayDecimals` (default 2)
- Show examples: `Net = gross - tare`, `%LOD = (wet - dry) / wet * 100`, `Yield = output / input * 100`

#### Section 3: Sampling Plan
- `samplingMethod` — dropdown + add-new (`random`, `systematic`, `stratified`, `census`, `composite`)
- `samplingNote` (text)

#### Section 4: Multi-Stage Acceptance Criteria
Sub-panel with green dashed border, USP standard tags top-right (`USP <711>`, `USP <905>`, `USP <701>`).

- Array `stages[]` of 1-3 stages (add/remove buttons)
- Each stage:
  - Stage number badge (S1, S2, S3)
  - `sampleSize`
  - `failureTolerance` (%)
  - `onFail` action: `next` | `reject` | `deviation`
  - Computed display: "ทดสอบ N ชิ้น · ยอมเสียได้ M ชิ้น · ต้องผ่าน (N-M) ชิ้น" where M = `floor(sampleSize × failureTolerance/100)`
- If any stage has `onFail='deviation'` → show **Deviation Preview Card** (DEV-2026-XXXX, Severity, Status, Assignee placeholder)
- Add Stage button at bottom (max 3 stages)
- When removing intermediate stage, auto-update prior stage's `onFail` to `reject`

#### Section 5: ตรวจสอบเมื่อ (Multi-Trigger Inspection) ⭐
Header: "ตรวจสอบเมื่อ — เลือกได้หลายแบบ" + counter chip "เปิด N/5"

5 trigger types — each has main toggle + nested config when on:

1. **`time`** (⏱ ตามช่วงเวลา) — `{ on: bool, every: string }` (minutes)
2. **`quantity`** (📦 ตามจำนวนผลิต) — `{ on, every, unit, mode: 'fixed'|'percent', percent }`
3. **`milestone`** (🎯 ที่ milestone) — `{ on, options: [{ id, label, on }, ...] }` with checkboxes:
   - `batch_start` (เริ่ม batch)
   - `batch_mid` (กลาง batch)
   - `batch_end` (ก่อนปิด batch)
   - `pre_compress` (ก่อน compression)
   - `post_coat` (หลัง coating)
   - + add custom option
4. **`event`** (⚡ เมื่อมีเหตุการณ์) — `{ on, options: [{ id, label, on }, ...] }`:
   - `changeover` (หลัง changeover)
   - `lot_change` (หลังเปลี่ยน lot วัตถุดิบ)
   - `cleaning` (หลัง equipment cleaning)
   - `param` (หลัง parameter change)
   - `maintenance` (หลัง maintenance)
5. **`oncePerBatch`** (🔄 ครั้งเดียวต่อ batch) — `{ on: bool }`

#### Section 6: Derived Calculations (Cross-step)
- `derivedCalcs[]` — array of `{ id, label, formula, sources[], resultUnit, triggerWhen, acceptanceMin, acceptanceMax, onFail, note }`
- Can add/remove
- Example: "อัตราการบรรจุเฉลี่ย (capsules/min)" using inputs from other criteria codes

#### Section 7: Settings
- `critical` (boolean) — toggle card "Critical" with explanation "ถ้าเปิด → operator ใช้เครื่องไม่ได้จนกว่าจะ pass"
- `active` (boolean) — toggle card "Active"

### Right Sticky Preview Panel

Real-time updates as user types:

```
Live Preview ●  (green pulsing dot)
ตัวอย่างที่ operator จะเห็น

┌────────────────────────────────────┐
│ IPC-CAP-WV-001        [! CRITICAL] │
│ Weight Variation                   │
│ Capsule — แคปซูล                   │
│                                    │
│ 🎯 Multi-Point Numeric             │
│    20 จุด · Target 0.500 ± 7.5%    │
│    ⚖️ Tare จาก IPC-CAP-TARE-001    │
│    Rule: ALL PASS · 0.4625-0.5375  │
└────────────────────────────────────┘

ACCEPTANCE FLOW
┌─────┐
│ S1  │ สุ่ม 20 ชิ้น · เสียได้ 1
└─────┘ tolerance 5%
   ↓ ถ้าไม่ผ่าน
┌─────┐
│ S2  │ สุ่ม 40 ชิ้น · เสียได้ 2
└─────┘ tolerance 5%
   ↓ ถ้าไม่ผ่าน
   ✕ Reject Batch

VALIDATION    ✓ Clean
ไม่พบปัญหา cross-field

GMP COMPLIANCE         7/7 · 100%
✓ Test name (USP / Pharmacopoeia)
✓ Unique Code (traceability)
✓ Multi-stage testing (USP <711>/<905>)
✓ Fail-action defined (ICH Q9)
✓ Sampling trigger (PIC/S Annex 8)
✓ Acceptance criteria specified
✓ Active for batch production
```

### Sticky Footer
- Left: status indicator
  - If `testName` set: green pulsing dot + "พร้อมบันทึก: {code}"
  - Else: "⚠ กรุณาเลือก Test Name ก่อน"
- Right: `[CANCEL]` `[CREATE]`
- CREATE disabled when: `!testName || !code || errorCount > 0`
- Show error count badge: `CREATE (3 err)`

### Post-CREATE Modal (centered)

```
┌─────────────────────────────────────────┐
│ ✓  สร้าง IPC Criteria เรียบร้อย          │
│    บันทึก IPC-XXX แล้ว                   │
│    รวมทั้งหมด N รายการ              [×] │
│                                          │
│ ต้องการทำอะไรต่อ?                       │
│ ┌─────────────────────────────────────┐ │
│ │ +  สร้าง Criteria ตัวใหม่ (ฟอร์มล้าง)│ │  ← primary (green)
│ └─────────────────────────────────────┘ │
│ ┌─────────────────────────────────────┐ │
│ │ 👷 ไปหน้าบันทึกผล (Operator) →      │ │  ← secondary
│ └─────────────────────────────────────┘ │
│        อยู่หน้านี้ (แก้ไขต่อ)            │  ← text button
└─────────────────────────────────────────┘
```

Reset button must:
- Clear all form fields back to defaults
- Reset `autoFilled` tracking set
- Scroll to top smoothly
- Refresh the `tareSource` dropdown to show newly-created criteria

---

## PAGE B & C — RECORDING SPEC

### Page B: List View `/ipc-recording`

#### Empty state
- Centered icon + text "ยังไม่มี IPC Criteria"
- Button: "+ สร้าง IPC Criteria" → links to `/master-data/ipc-criteria/new`

#### List
- Header: title + count + button "+ สร้าง Criteria ใหม่"
- Grid of cards (2-col desktop, 1-col mobile)

Each card:
```
┌────────────────────────────────────┐
│ IPC-CAP-WV-001   MULTI-POINT       │
│                  ! CRITICAL        │
│                  ✓ 2 รอบ           │
│                  • กำลังบันทึก     │
│                                    │
│ Weight Variation                   │
│ หลายจุด · g                        │
│ BPR · 8.3                          │
├────────────────────────────────────┤
│ [ดำเนินการต่อ →] | [+ รอบใหม่]    │
└────────────────────────────────────┘
```

Card states:
- Status badges:
  - `✓ N รอบ` (emerald) — submitted rounds count
  - `• กำลังบันทึก` (amber) — has in-progress round
- Action bar (clickable, action below):
  - **Left action** (changes label based on state):
    - No rounds: "เริ่มบันทึก →"
    - Has in-progress: "ดำเนินการต่อ →"
    - Has only submitted: "ดูรอบทั้งหมด →"
  - **Right action**: "+ รอบใหม่" → opens reason modal → creates new round + navigates to record view with new round active

#### New Round Modal (from list card)
- Title: "เริ่มรอบบันทึกใหม่"
- Show code + testLabel
- Textarea: "เหตุผล (ไม่บังคับ)" placeholder: "เช่น น้ำหนักแคปซูลเบี่ยงเบนเกิน → ปรับหัวตอกใหม่"
- Hint: "รอบใหม่จะถูกสร้างทันที พร้อมพาเข้าหน้าบันทึก"
- Buttons: [ยกเลิก] [เริ่มบันทึก →]

### Page C: Record View `/ipc-recording/[criteriaId]`

#### Header
- Back button "← รายการ"
- Code badge + Type chip + Critical badge
- Title (testLabel)

#### Instructions Box
If `sopStepRef.stepDescription` exists → show sky-blue info box with description.
If `critical` → show red warning sub-box.

#### Round Tabs Strip
```
รอบการบันทึก (2/3 ส่งแล้ว)        [+ บันทึกรอบใหม่]
┌──────────┐ ┌──────────┐ ┌──────────────────┐
│ รอบ 1    │ │ รอบ 2    │ │ รอบ 3      ●    │  ← active (dark green)
│ 08:15    │ │ 09:15    │ │ 10:30            │
│ ✓ ส่งแล้ว│ │ ✓ ส่งแล้ว│ │ • กำลังบันทึก    │
│          │ │ เหตุผล:  │ │                  │
│          │ │ ปรับ pin │ │                  │
└──────────┘ └──────────┘ └──────────────────┘
```

- Horizontal scroll on mobile
- Click tab → switch active round
- Tab colors:
  - Active: `bg-emerald-600 text-white`
  - Submitted: `bg-emerald-50 border-emerald-300 text-emerald-700`
  - In-progress: `bg-amber-50 border-amber-300 text-amber-700`

#### Locked Banner (when active round is submitted)
Slate box:
```
🔒 รอบนี้ส่งผลไปแล้วเมื่อ {datetime} — ดูได้แต่แก้ไขไม่ได้
   ถ้าต้องการบันทึกใหม่ กดปุ่ม + บันทึกรอบใหม่ ด้านบน
```
Form below becomes `pointer-events-none opacity-70`.

#### Form (changes by criteriaType)

##### Numeric / Calibration / Tare
1. Spec banner (gradient green/indigo/cyan): show acceptance range
2. Big tap-button to open Numpad
3. After value entered:
   - Pass (green) / Fail (red) cell
   - Status text below

##### Pass/Fail
- 2 banners showing passDefinition / failDefinition
- 2 big buttons side-by-side: PASS (green) / FAIL (red)
- Selected one is highlighted

##### Visual
- Card showing visualCriteria
- Checklist of items as tap-toggle buttons
- Counter: "Checklist (N/M)" with "✓ ครบทุกข้อ" badge when complete

##### Text
- Card showing textFormat + textExample (if any)
- Textarea with character counter

##### Multi-Point ⭐
**Tare Banner** at top (if `multiPoint.tareSource` is set):

Lookup logic:
1. Query: find criteria where `code = multiPoint.tareSource`
2. Query: find that criteria's latest submitted RecordingRound
3. Get `data.points` → compute mean of valid numbers
4. If found → show cyan banner; if not → show amber warning banner

Cyan (success):
```
⚖️ Tare หักลบอัตโนมัติ
    0.0960 g
จาก IPC-CAP-TARE-001 — น้ำหนักแคปซูลเปล่า (รอบล่าสุด)
Net = Gross − 0.0960 g
```

Amber (warning):
```
⚠️ ต้องมีค่า Tare ก่อนบันทึก
    ยังไม่มีรอบที่ส่งผลของ IPC-CAP-TARE-001
Tare criteria: IPC-CAP-TARE-001 — น้ำหนักแคปซูลเปล่า
```

**Spec Banner**: gradient teal showing acceptance range. If using tare, label = "ค่า NET ที่ยอมรับต่อจุด" and sub-text adds "เช็คกับ NET weight".

**Progress bar**: shows `filled/pointCount` and percentage.

**Cells grid** (2-3 columns):

Without tare:
```
┌────────────┐
│ จุดที่ 1  ✓ │
│   285.5    │
└────────────┘
```

**With tare**:
```
┌──────────────────┐
│ แคปซูล 1     ✓   │
│ Gross  0.5969    │
│ Net    0.5009    │
└──────────────────┘
```

Cell behavior:
- Tap → opens Numpad with label "แคปซูล 1 (Gross)" if using tare
- Auto-saves on every keystroke
- Pass check uses **Net value** if tare is applied; otherwise raw value
- States: empty (gray) | pass (green) | fail (red)

**Aggregate Result** (shows when all cells filled):
- Calculate based on `aggregateRule`:
  - `all_pass`: every net point in range
  - `mean`: mean (of net values if tare) in `aggregateLimit` range
  - `rsd`: standard deviation / mean × 100 ≤ limit
  - `min_max`: every net point in `aggregateLimit` range
- Display:
  - If tare: show both "Mean (Gross): X.XXXX" and "Mean (Net): X.XXXX" (big)
  - Without tare: just Mean
  - Status badge: ✓ ผ่าน / ✕ ไม่ผ่าน
- Label: e.g. "ผลรวม (auto) — Mean 0.4625–0.5375 · เทียบกับ NET"

##### Custom Multi-Field
- Render each field per its type:
  - `number`: tap to open Numpad
  - `text`: text input
- Pass/fail check based on each field's `target ± tolerance`
- Show required marker `*`

##### Calculated
- Inputs (each tap → Numpad)
- Bottom: Auto-evaluation card showing result and pass/fail vs `resultMin`-`resultMax`
- Formula evaluation: use safe expression evaluator (NOT `eval`/`new Function` in production — use a math expression parser library like `mathjs`)

#### Numpad Popup Component

```
┌─────────────────────────────────────┐
│ [emerald header]                    │
│ กำลังกรอก                     [×]   │
│ แคปซูล 1 (Gross)                    │
├─────────────────────────────────────┤
│                                     │
│            0.5969 g                 │ ← big mono display
│                                     │
│ ┌─────┐ ┌─────┐ ┌─────┐            │
│ │  7  │ │  8  │ │  9  │            │
│ └─────┘ └─────┘ └─────┘            │
│ ┌─────┐ ┌─────┐ ┌─────┐            │
│ │  4  │ │  5  │ │  6  │            │
│ └─────┘ └─────┘ └─────┘            │
│ ┌─────┐ ┌─────┐ ┌─────┐            │
│ │  1  │ │  2  │ │  3  │            │
│ └─────┘ └─────┘ └─────┘            │
│ ┌─────┐ ┌─────┐ ┌─────┐            │
│ │  ·  │ │  0  │ │  ⌫  │            │
│ └─────┘ └─────┘ └─────┘            │
│ ┌─────────┐ ┌──────────────────┐   │
│ │ เคลียร์ │ │  ✓ บันทึก         │   │
│ └─────────┘ └──────────────────┘   │
└─────────────────────────────────────┘
```

- Touch-friendly: buttons ≥ 64px
- Backdrop click closes
- If `allowDecimal=false` (unit='count') → replace `·` with `±` for negative numbers
- Mobile: bottom sheet; Desktop: centered modal

#### Sticky Footer
- Left: "บันทึกอัตโนมัติ HH:MM:SS" with pulsing green dot when recently saved
- Center (if not locked + not valid): amber text "⚠ {reason}"
- Right: `[ส่งผล]` button
  - Disabled if: `isLocked || !validation.ok`
  - When locked: shows "ส่งแล้ว"
  - Tooltip on disabled: shows validation reason

#### Validation Rules per Type
- `numeric` / `calibration` / `tare`: value non-empty and is a number
- `pass_fail`: value === 'pass' || 'fail'
- `visual`: all checklist items checked
- `text`: value non-empty after trim
- `multi_point`: all points filled with valid numbers (count = pointCount)
- `custom_multi_field`: all required fields filled (numeric must parse)
- `calculated`: all inputs filled with valid numbers

#### Submit Flow
1. User clicks "ส่งผล"
2. **(Optional) E-signature step**: prompt for password re-entry. On verify → continue. (Add this if GMP compliance is required.)
3. Update RecordingRound: `submittedAt = now`, `submittedBy = currentUser`, `signatureHash = HMAC(data + user + timestamp + secret)`
4. Lock the round (immutable from now)
5. Write audit log
6. Show success modal:

```
┌─────────────────────────────────────────┐
│ ✓ ส่งผล รอบ N เรียบร้อย                  │
│                                          │
│ ผลถูกล็อกแล้ว — รหัส IPC-XXX             │
│ หากต้องการบันทึกซ้ำ (เช่น หลังปรับเครื่อง)│
│ กด "เริ่มรอบใหม่"                        │
│                                          │
│ ┌─────────────────────────────────────┐ │
│ │  + เริ่มรอบใหม่ทันที                 │ │  ← primary
│ └─────────────────────────────────────┘ │
│ ┌─────────────────────────────────────┐ │
│ │  กลับรายการ                          │ │
│ └─────────────────────────────────────┘ │
│           ปิด                            │
└─────────────────────────────────────────┘
```

---

## DATA MODEL

### Migration strategy
1. Inspect existing `ipc_criteria` table.
2. Create new tables (versions, recording_rounds, audit_log, etc.) — don't drop existing.
3. Write migration script: for each existing criteria row, create a CriteriaVersion v1 from its current data, link via `currentVersionId`.
4. Add new columns gradually with defaults so existing code doesn't break.

### Schema (use these definitions; adapt to your ORM/DB)

```typescript
// Roles
enum Role { OPERATOR, QA, ADMIN, MANAGER }

// User (likely exists — extend if needed)
type User = {
  id: string
  email: string
  name: string
  passwordHash: string
  role: Role
  active: boolean
}

// Batch (link recording to production batch)
type Batch = {
  id: string
  batchNumber: string  // unique
  product: string
  startedAt?: Date
  completedAt?: Date
  status: 'PLANNED' | 'IN_PROGRESS' | 'COMPLETED' | 'REJECTED' | 'ON_HOLD'
}

// Criteria (master record — minimal, versioned)
type Criteria = {
  id: string
  code: string  // unique, e.g. "IPC-CAP-TARE-001"
  status: 'DRAFT' | 'ACTIVE' | 'ARCHIVED' | 'RETIRED'
  active: boolean
  critical: boolean
  currentVersionId: string  // FK to latest CriteriaVersion
  createdAt: Date
  createdById: string
}

// CriteriaVersion (immutable — new row on every change)
type CriteriaVersion = {
  id: string
  criteriaId: string
  versionNumber: number  // 1, 2, 3...
  type: CriteriaType  // 9 enum values
  testName: string
  testLabel: string
  unit?: string
  dosageForm?: string
  payload: JSON  // see below
  approvedAt?: Date
  approvedById?: string
  createdAt: Date
}

type CriteriaType =
  | 'numeric' | 'pass_fail' | 'visual' | 'text'
  | 'multi_point' | 'custom_multi_field'
  | 'calibration' | 'tare' | 'calculated'

// CriteriaVersion.payload structure
type CriteriaPayload = {
  // Type-specific (only one set, based on type)
  target?: string
  tolerance?: string
  passDefinition?: string
  failDefinition?: string
  defaultExpected?: 'pass' | 'fail'
  visualChecklist?: string[]
  visualCriteria?: string
  visualReference?: string
  textFormat?: string
  textExample?: string
  textRequired?: boolean
  customFields?: CustomField[]
  customNote?: string
  multiPoint?: MultiPointSpec
  calibration?: CalibrationSpec
  tare?: TareSpec
  calculated?: CalculatedSpec

  // Shared
  stages: Stage[]
  samplingMethod: string
  useContext: string[]
  sopStepRef: SopStepRef
  triggers: Triggers
  derivedCalcs: DerivedCalc[]
}

type MultiPointSpec = {
  pointCount: string
  pointLabel: string
  perPointTarget: string
  perPointTolerance: string
  aggregateRule: 'all_pass' | 'mean' | 'rsd' | 'min_max'
  aggregateLimit: string  // auto-computed for mean/min_max
  tareSource: string      // code of linked Tare criteria
}

type Stage = {
  id: number
  sampleSize: string
  failureTolerance: string
  onFail: 'next' | 'reject' | 'deviation'
  note?: string
}

type Triggers = {
  time?: { on: boolean; every: string }
  quantity?: { on: boolean; every: string; unit: string; mode: 'fixed'|'percent'; percent: string }
  milestone?: { on: boolean; options: TriggerOption[] }
  event?: { on: boolean; options: TriggerOption[] }
  oncePerBatch?: { on: boolean }
}

type TriggerOption = { id: string; label?: string; on: boolean }

type SopStepRef = {
  sopCode: string
  sopVersion: string
  stepNumber: string
  stepDescription: string
  link?: string
}

// RecordingRound — per criteria per batch per round
type RecordingRound = {
  id: string
  criteriaId: string
  criteriaVersionId: string  // pin to specific version at recording time
  batchId?: string
  roundIndex: number  // 1, 2, 3... within (criteriaId, batchId)
  reason?: string  // e.g. "หลังปรับ tamping pin"
  startedAt: Date
  startedById: string

  data: JSON  // RoundData

  // Submit info (immutable once set)
  submittedAt?: Date
  submittedById?: string
  signatureHash?: string
  signatureMethod?: 'PASSWORD' | 'PIN' | 'OAUTH_REAUTH'

  // Denormalized
  passed?: boolean
  computedMean?: number
  outcomeNote?: string
}

type RoundData = {
  type: CriteriaType
  value?: string                    // numeric / pass_fail / text / calibration / tare
  points?: string[]                 // multi_point
  checks?: Record<number, boolean>  // visual
  values?: Record<number, string>   // custom_multi_field
  inputs?: Record<number, string>   // calculated
}

// AuditLog (append-only, 21 CFR Part 11)
type AuditLog = {
  id: string
  timestamp: Date
  userId: string
  action: 'CREATE' | 'UPDATE' | 'SUBMIT' | 'SIGN' | 'APPROVE' | 'ARCHIVE' | 'LOGIN' | 'LOGOUT'
  entityType: string  // 'Criteria' | 'RecordingRound' | 'Deviation' | etc.
  entityId: string
  changes?: { before: unknown; after: unknown }
  reason?: string  // GMP requirement
  ipAddress?: string
  userAgent?: string
}
```

---

## API ENDPOINTS

Follow REST conventions. All endpoints require authentication unless noted.

```
# Authentication (may already exist)
POST   /api/auth/login                  { email, password } → JWT/session
POST   /api/auth/logout
POST   /api/auth/verify                 { password } → boolean  (for e-signature)

# Criteria
GET    /api/criteria                    ?status=&type=&active=&search=
GET    /api/criteria/:id                returns full + versions list
GET    /api/criteria/by-code/:code      lookup by code
POST   /api/criteria                    create new criteria (version 1)
PUT    /api/criteria/:id                "update" → actually creates new version, never overwrites
POST   /api/criteria/:id/archive
GET    /api/criteria/:id/versions

# Recording
GET    /api/recording/rounds            ?criteriaId=&batchId=&submitted=
POST   /api/recording/rounds            { criteriaId, batchId?, reason? } → creates new round, returns id
PUT    /api/recording/rounds/:id        { data } → update in-progress data (block if submittedAt set)
POST   /api/recording/rounds/:id/submit { password? } → set submittedAt + signature, lock
GET    /api/recording/rounds/:id

# Tare lookup helper
GET    /api/criteria/by-code/:code/latest-submitted-round
       → returns the latest RecordingRound where submittedAt != null

# Audit
GET    /api/audit-log                   ?entityType=&entityId=&userId=&from=&to=
```

### POST `/api/recording/rounds/:id/submit` example

```typescript
// Server-side
async function submitRound(id, password) {
  const round = await db.recordingRound.findById(id)
  if (round.submittedAt) throw new Error('Already submitted')

  // E-signature: re-verify password
  const verified = await verifyPassword(currentUser.id, password)
  if (!verified) throw new Error('Invalid password')

  const timestamp = new Date().toISOString()
  const signatureHash = crypto.createHmac('sha256', process.env.SIGNATURE_SECRET)
    .update(JSON.stringify({ data: round.data, userId: currentUser.id, timestamp }))
    .digest('hex')

  // Compute pass/fail server-side for trust
  const computed = computeOutcome(round.data, round.criteriaVersion.payload)

  const updated = await db.recordingRound.update(id, {
    submittedAt: timestamp,
    submittedById: currentUser.id,
    signatureHash,
    signatureMethod: 'PASSWORD',
    passed: computed.passed,
    computedMean: computed.mean,
  })

  await db.auditLog.create({
    action: 'SUBMIT',
    entityType: 'RecordingRound',
    entityId: id,
    userId: currentUser.id,
    changes: { passed: computed.passed, mean: computed.mean },
  })

  return updated
}
```

---

## COMPONENT BREAKDOWN (suggested)

```
components/
├── forms/
│   ├── CriteriaForm.tsx           — main form orchestrator
│   ├── BasicInfoSection.tsx
│   ├── SpecificationSection.tsx   — switches by criteriaType
│   ├── specs/
│   │   ├── NumericSpec.tsx
│   │   ├── PassFailSpec.tsx
│   │   ├── VisualSpec.tsx
│   │   ├── TextSpec.tsx
│   │   ├── MultiPointSpec.tsx     — includes auto-computed range + tare dropdown
│   │   ├── CustomFieldsSpec.tsx
│   │   ├── CalibrationSpec.tsx
│   │   ├── TareSpec.tsx
│   │   └── CalculatedSpec.tsx
│   ├── SamplingPlanSection.tsx
│   ├── MultiStageSection.tsx
│   ├── TriggersSection.tsx
│   ├── DerivedCalcsSection.tsx
│   └── SettingsSection.tsx
├── recorders/
│   ├── RecorderByType.tsx         — switch by type
│   ├── NumericRecorder.tsx
│   ├── PassFailRecorder.tsx
│   ├── VisualRecorder.tsx
│   ├── TextRecorder.tsx
│   ├── MultiPointRecorder.tsx     — includes tare lookup + gross/net display
│   ├── CustomFieldsRecorder.tsx
│   ├── CalibrationRecorder.tsx
│   ├── TareRecorder.tsx
│   └── CalculatedRecorder.tsx
├── ui/
│   ├── SearchableSelect.tsx
│   ├── EditableSelect.tsx         — SearchableSelect + add-new modal
│   ├── AddOptionModal.tsx
│   ├── Numpad.tsx
│   ├── Toggle.tsx
│   ├── SpecBanner.tsx
│   ├── InstructionsBox.tsx
│   ├── RoundTabs.tsx
│   ├── NewRoundModal.tsx
│   └── PostCreateModal.tsx
└── preview/
    ├── LivePreviewPanel.tsx
    ├── AcceptanceFlow.tsx
    ├── ValidationCard.tsx
    └── GmpComplianceCard.tsx

pages/
├── master-data/ipc-criteria/
│   ├── index.tsx           — list (may exist)
│   ├── new.tsx             — uses CriteriaForm
│   └── [id]/edit.tsx       — uses CriteriaForm
└── ipc-recording/
    ├── index.tsx           — list view
    └── [criteriaId].tsx    — record view
```

---

## STYLING

Use the existing system's CSS framework. Reference colors from the prototype:

- Brand emerald: `#10b981` (primary), `#047857` (dark)
- Pass: `#10b981`
- Fail: `#ef4444`
- Warn: `#f59e0b`
- Critical/Tare info cyan: `#06b6d4`

Recommended touch targets:
- `.tap` = min-height 48px
- `.tap-lg` = min-height 64px (for operator/numpad)

Font: Inter + Sarabun (Google Fonts) for Thai readability.

---

## TEST SCENARIOS (Acceptance Criteria)

Don't mark the task complete until all of these pass:

1. **Create + link**: Create TARE-001 (Multi-Point, Aggregate=Mean), then create WV-001 (Multi-Point) and select TARE-001 in tareSource dropdown. Verify DB: WV-001's `multiPoint.tareSource = "IPC-CAP-TARE-001"`.

2. **Auto-computed range**: In Multi-Point spec, set Target=0.5, Tolerance=7.5%, Aggregate=Mean. The aggregateLimit field shows "0.4625-0.5375" automatically. Change Tolerance to 10. Field updates to "0.45-0.55".

3. **Tare subtraction**: Submit TARE-001 with 10 values (avg = 0.0960). Open WV-001 record view. Tare banner shows "0.0960 g". Enter Gross 0.5969 in cell 1. Cell displays Gross=0.5969, Net=0.5009, status ✓ (because in 0.4625-0.5375 range).

4. **Multi-round**: Submit WV-001 round 1. From list, click "+ รอบใหม่" on WV-001 card. Enter reason "ปรับ tamping pin". New round created (roundIndex=2), navigates to record view with round 2 active. Round 1 tab visible and clickable but its content is read-only.

5. **Submit validation**: WV-001 new round — fill only 5 of 20 cells. Submit button disabled, shows "⚠ กรอกแล้ว 5/20 จุด — ยังไม่ครบ".

6. **Tare missing warning**: Delete TARE-001 (or never submit a round). Open WV-001 record. Amber warning banner shows. Operator can still type values but Net column shows "—".

7. **Multi-trigger**: Create criteria with 3 triggers enabled (time + milestone + event). DB shows `triggers.time.on=true`, `triggers.milestone.on=true` with one milestone option `on=true`, `triggers.event.on=true` with one event option `on=true`. Live Preview lists all 3.

8. **Audit trail**: Every action (CREATE, UPDATE, SUBMIT, ARCHIVE) writes to `audit_log`. Verify `entityType`, `entityId`, `userId`, `timestamp`, `changes` are populated. `audit_log` rows are never UPDATEd or DELETEd.

9. **E-signature** (if implemented): Submit recording → prompts for password → on correct password, `signatureHash` is set on the round. On wrong password, submission rejected. Verify hash can be re-computed and matches.

10. **Responsive**: Form usable on mobile (320px width). Sidebar collapses to hamburger. Preview panel moves below form on tablet. Numpad bottom-sheet on mobile, modal on desktop.

---

## DELIVERY CHECKLIST

Before submitting PR:

- [ ] All 10 test scenarios pass manually
- [ ] DB migration script tested on a copy of production data
- [ ] No data loss on migration (existing criteria still work)
- [ ] Audit log written for all mutating actions
- [ ] TypeScript strict mode (if using TS)
- [ ] No console errors/warnings on critical path
- [ ] Works in Chrome + Safari + Firefox (latest 2 versions)
- [ ] Thai text renders correctly (no missing glyphs)
- [ ] All forms validate before submit
- [ ] Loading states for all async actions
- [ ] Error toasts for API failures
- [ ] PR description includes: tested scenarios, migration plan, rollback plan, screenshots of new pages

---

## IMPORTANT REMINDERS

- **Versioning**: When user "edits" a criteria, never UPDATE the existing row. Create a new CriteriaVersion and update `currentVersionId` on Criteria.
- **Immutability**: Once `RecordingRound.submittedAt` is set, the row is read-only forever. Validate this in API layer.
- **No deletes**: Use `status='ARCHIVED'` instead of DELETE for criteria. Audit log is append-only.
- **Thai support**: All UI strings should support Thai. Use Sarabun font fallback. Test character widths.
- **Touch-first**: Recording page will be used on tablets by operators. Buttons ≥ 48px tap target. Avoid hover-only interactions.
- **Avoid `eval`**: For Calculated type formula evaluation, use `mathjs` or similar safe parser. Never use `new Function()` or `eval()` on user input.

---

## ASK BEFORE PROCEEDING

After reading this entire document:

1. Confirm tech stack & code location
2. Confirm which roles exist
3. Confirm whether e-signature is required
4. Confirm batch system status
5. Confirm i18n requirement

Then propose a step-by-step implementation plan (with estimated PRs/chunks) before writing code. Don't write the entire system in one go — break it into reviewable PRs.

End of prompt.
