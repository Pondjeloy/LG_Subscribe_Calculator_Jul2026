# สูตรคำนวณราคา LG Subscribe — Pricing Formula

เอกสารนี้อธิบาย **โครงสร้างข้อมูลราคา** และ **สูตรคำนวณ** ที่ใช้ในไฟล์ `lg_subscribe_calculator_.html` เพื่อให้แก้ไข/เพิ่มข้อมูลได้อย่างถูกต้อง โดยเฉพาะกรณี "ลดซ้อน" เช่น `DC100_DC50%(6M)` ที่หลายคนเข้าใจผิด

---

## 1. โครงสร้างข้อมูลของแต่ละแพ็คเกจ (plan)

แต่ละ plan ใน `catalog[หมวด].models[i].plans[j]` มีฟิลด์ดังนี้:

```js
{
  term: '5Y',                       // อายุสัญญา (5Y/6Y/7Y)
  serviceType: 'Self',              // Visit / Self / No Service
  serviceCycle: 'ทุก 12 เดือน',     // รอบเข้าบริการ
  regular: 499,                     // ราคาปกติ/เดือน (ไม่มีโปร) — ใช้เป็น "ราคาก่อนลด" สำหรับเทียบส่วนลด
  effectiveMonthly: 199,            // ราคาที่จ่ายในช่วงโปรพิเศษ (รอบบิลที่ 1..promoMonths)
  promoMonths: 6,                   // จำนวนเดือนที่ใช้ effectiveMonthly
  postPromoPrice: 399,              // ราคาหลังหมดช่วงโปรพิเศษ (รอบบิลที่ promoMonths+1 ถึงจบสัญญา)
  totalContractMonths: 60,          // อายุสัญญาเป็นเดือน (5Y=60, 6Y=72, 7Y=84)
  totalSaving: 7200,                // ส่วนลดรวมตลอดสัญญา (เทียบกับ regular × totalContractMonths)
  promo: 'ลด 50% 6 บิลแรก (199) จากนั้น 399/เดือน'  // ข้อความอธิบายโปร
}
```

### ⚠️ ข้อผิดพลาดที่พบบ่อย

| ฟิลด์ | สิ่งที่ ผิด/ถูก |
|---|---|
| `effectiveMonthly` | ต้องเป็น **ราคาที่จ่ายจริงในรอบบิลที่ 1..N** (หลังคิดส่วนลดซ้อนทุกชั้นแล้ว) ไม่ใช่ "ราคาโปรชั้นแรก" |
| `postPromoPrice` | ต้องเป็น **ราคาที่จ่ายจริงในรอบบิลที่ N+1 ถึงจบสัญญา** — ส่วนใหญ่ = `regular − DC100/DC150` (โปรลดรายเดือน) ไม่ใช่ `regular` |

---

## 2. สูตรคำนวณยอดบิลและส่วนลด

ในไฟล์ HTML (ฟังก์ชัน `itemComboSchedule` บรรทัด ~1424):

```js
const pm = item.promoMonths || 0;
for (let k = 1; k <= months; k++) {
  bills[k] = (k <= pm) ? item.effectiveMonthly : item.postPromoPrice;
}
```

แปลเป็นสูตร:

```
ยอดจ่ายตลอดสัญญา = (effectiveMonthly × promoMonths) + (postPromoPrice × (N − promoMonths))
ส่วนลดรวม (totalSaving) = (regular × N) − ยอดจ่ายตลอดสัญญา
```

โดยที่ `N = totalContractMonths` (60 / 72 / 84)

---

## 3. การถอดสูตรจาก Policy Code (ลดซ้อน)

อ้างอิงจาก `LG_Subscribe_Policy_Code.md` — ส่วนลดจะคำนวณ **ตามลำดับซ้าย → ขวา**

### กรณี A — โปรชั้นเดียว `DC50%(NM)`
```
VISIT_5Y_12M_DC50%(6M)   regular=599
→ effectiveMonthly = 599 × 0.5 = 299.5 ≈ 299
→ postPromoPrice   = 599 (กลับสู่ราคาปกติ)
→ promoMonths      = 6
```

### กรณี B — ลดซ้อน `DC100_DC50%(6M)` ← **จุดที่มักผิด**
```
SELF_5Y_12M_DC100_DC50%(6M)   regular=499
ขั้นที่ 1: DC100  → 499 − 100 = 399  (ราคาโปรตลอดสัญญา)
ขั้นที่ 2: DC50%(6M) → 399 × 0.5 = 199.5 ≈ 199  (เฉพาะ 6 บิลแรก)

→ effectiveMonthly = 199   (บิล 1-6, หลังลดทั้ง 2 ชั้น)
→ postPromoPrice   = 399   (บิล 7-60, ใช้ราคาโปรชั้นแรก ไม่กลับไป regular)
→ promoMonths      = 6
→ totalSaving      = (499 × 60) − (199 × 6 + 399 × 54)
                   = 29,940 − 22,740 = 7,200
```

### กรณี C — ราคาคงที่ `149(NM)` / `149_PRO_DC100`
```
VISIT_5Y_12M_PRO_DC100_149(3M)   regular=849
ขั้นที่ 1: PRO_DC100 → 849 − 100 = 749  (ราคาโปรตลอดสัญญา)
ขั้นที่ 2: 149(3M)   → จ่าย 149 บาท เฉพาะ 3 บิลแรก

→ effectiveMonthly = 149   (บิล 1-3 ราคาคงที่)
→ postPromoPrice   = 749   (บิล 4-60)
→ promoMonths      = 3
→ totalSaving      = (849 × 60) − (149 × 3 + 749 × 57)
                   = 50,940 − 43,140 = 7,800
```

### กรณี D — โปรราคาเดียวตลอดสัญญา `PRO_DC50`
```
VISIT_5Y_6M_PRO_DC50   regular=349
→ ลด 50 บาทตลอดสัญญา

→ effectiveMonthly = 299   (= postPromoPrice ก็ได้)
→ postPromoPrice   = 299
→ promoMonths      = 60   (= totalContractMonths)
→ totalSaving      = (349 − 299) × 60 = 3,000
```

---

## 4. Checklist ก่อน save ทุก plan

ก่อนเพิ่ม/แก้ข้อมูล plan ใดๆ ให้ตรวจ:

1. ✅ `regular > effectiveMonthly` (ถ้ามีโปร) และ `regular ≥ postPromoPrice` เสมอ
2. ✅ ถ้า Policy Code มี `DC100/DC150` → `postPromoPrice` ต้องเป็น `regular − 100/150` ไม่ใช่ `regular`
3. ✅ ถ้า Policy Code มี `DC50%(NM)` ตามหลัง `DC100` → `effectiveMonthly` ต้องเป็น **ครึ่งหนึ่งของราคาหลัง DC100** ไม่ใช่ครึ่งหนึ่งของ regular
4. ✅ ตรวจสมการ:
   ```
   totalSaving == (regular × N) − (effectiveMonthly × promoMonths + postPromoPrice × (N − promoMonths))
   ```

---

## 5. ตัวอย่าง Bug Pattern ที่เคยพบ (และแก้แล้ว)

### Bug — AS60GHWG0.ABAE (Hit), 5Y_Self, 12M

**ก่อนแก้ (ผิด):**
```js
{ regular:499, effectiveMonthly:399, promoMonths:6, postPromoPrice:499, totalSaving:7200 }
```
- ใส่ราคาโปรชั้นแรก (399 = regular − 100) ลงใน `effectiveMonthly` แทนที่จะเป็น 199
- ใส่ `regular` (499) ลงใน `postPromoPrice` ทั้งที่ราคาโปรคือ 399 ตลอดสัญญา
- ยอดจ่ายตามสูตรจะเป็น `399×6 + 499×54 = 29,340` ซึ่งไม่ตรงกับ totalSaving ที่ระบุไว้ (7,200)

**หลังแก้ (ถูก):**
```js
{ regular:499, effectiveMonthly:199, promoMonths:6, postPromoPrice:399, totalSaving:7200 }
```
- `199 × 6 + 399 × 54 = 22,740`
- `(499 × 60) − 22,740 = 7,200` ✓

---

## 6. การเชื่อมโยงกับ Policy Code

| Policy Code | `regular` | `effectiveMonthly` | `promoMonths` | `postPromoPrice` |
|---|---|---|---|---|
| `VISIT_5Y_12M_DC100_DC50%(6M)` | 549 | **224** (=(549−100)/2) | 6 | **449** (=549−100) |
| `SELF_5Y_12M_DC100_DC50%(6M)` | 499 | **199** | 6 | **399** |
| `VISIT_5Y_6M_DC100_DC50%(6M)` | 599 | **249** | 6 | **499** |
| `SELF_5Y_6M_DC100_DC50%(6M)` | 549 | **224** | 6 | **449** |
| `VISIT_5Y_12M_PRO_DC100_149(3M)` | 849 | **149** | 3 | **749** (=849−100) |
| `SELF_5Y_12M_PRO_DC100_149(3M)` | 799 | 149 | 3 | 699 |
| `VISIT_5Y_6M_PRO_DC150_149(3M)` | 1,349 | 149 | 3 | 1,199 (=1349−150) |
| `SELF_5Y_6M_PRO_DC150_149(3M)` | 1,299 | 149 | 3 | 1,149 |
| `VISIT_5Y_6M_PRO_DC50` | 349 | 299 | **60** (=N) | 299 |

---

*อัปเดต 2026-05-24 — สร้างหลังพบ bug ในใบเสนอราคาที่คำนวณยอดรวมเป็น 29,340 บาท (ที่ถูกต้องคือ 22,740 บาท)*
