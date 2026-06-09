# LG Subscribe – POND · สารบัญโปรเจกต์

**Path:** `C:\Users\MASTER POND\OneDrive\Documents\Codex\LG Subscribe_By_POND\`  
**เจ้าของ:** Epromoter POND · LG Subscribe Sales Tooling  
**สถานะ:** มิ.ย. 2569 — ราคา QC ครบแล้ว (2026-06-03)

---

## ไฟล์หลัก

| ไฟล์ | หน้าที่ |
|------|--------|
| `lg_subscribe_calculator_.html` | Calculator หลัก (Alpine.js) — แก้ราคาที่นี่ |
| `LG_Subscribe_Calculator_Jun2026.html` | Standalone สำหรับแชร์ (ไม่ต้องการ folder) |
| `customer_list.html` | CRM — อ่านข้อมูลลูกค้าจาก Google Sheets |
| `dashboard.html` | Dashboard ส่วนตัว Epromoter |
| `Code.gs` | Google Apps Script API (deploy ทุกครั้งที่แก้) |
| `parse_promotion_excel.py` | Parser Excel → catalog JSON (ใช้อัปเดตรายเดือน) |

---

## อัปเดตราคารายเดือน (Excel-based)

**Source of truth:** ไฟล์ `Promotion [เดือน]_..._Final.xlsx`

```
1. python parse_promotion_excel.py "Promotion [เดือน]_..._Final.xlsx"
2. ดู output ว่าราคาตรง (OLED77, Wash Tower, ตู้เย็น)
3. อัปเดต promoConfig dateStart/dateEnd ใน calculator
4. อัปเดต catalog เฉพาะรุ่นที่เปลี่ยน
5. rebuild standalone: python -c "..." → LG_Subscribe_Calculator_[เดือน].html
```

---

## Calculator — โครงสร้างสำคัญ

### promoConfig (บรรทัด ~874)
```js
normal:  { dateStart:'2569-06-01', dateEnd:'2569-06-30', newTiers:[{min:2,rate:10}], existTiers:[{min:1,rate:10}] }
special: { ... }  // K-HOUSE Combo
```

### Catalog structure ต่อรุ่น
```js
{ code, name, plans:[{ term, serviceType, serviceCycle, regular, effectiveMonthly,
  promoMonths, postPromoPrice, advancePayment?, billSchedule?[{range,price,note}],
  promo, totalContractMonths, totalSaving }] }
```

### Rounding rule (combo & display) — `bht(n)` / `snapBaht(x)`
- `> 0.5` → ปัดขึ้น | `≤ 0.5` → ปัดลง (ไม่ใช่ Math.round)

### TV/Monitor advance structure
- `advance = 6 × regular`
- b1-12: `regular/2` · b13-18: `floor(regular/2)` (DC6M) · b19-60: `regular`
- 2-period (no DC): b1-12, b13-60=regular, saving=0

---

## CRM — Code.gs Column Map

| Sheet | picCol | statusCol | notesCol | name | phone |
|-------|--------|-----------|----------|------|-------|
| Meta Densu/Credit | M(12) | L(11) | N(13) | F(5) | H(7) |
| Lead Subscribe Lg.com | J(9) | K(10) | I(8) | B(1) | G(6) |
| Lead LG Success | J(9) | I(8) | K(10) | B(1) | C(2) |
| Lead Consult | W(22) | V(21) | X(23) | E(4) | G(6) |
| Lead POP UP Braner | L(11) | K(10) | M(12) | C+D(2+3) | F(5) |

**`cleanDisplay(v, displayVal)`** — ใช้กับ phone: ถ้า GAS คืน Date object ให้ใช้ display value แทน  
**Deploy** Apps Script ทุกครั้งที่แก้ Code.gs

---

## June 2026 — รุ่นที่เปลี่ยนจาก May

| รุ่น | การเปลี่ยน |
|------|-----------|
| 85QNED80BSA.ATM | **เพิ่มใหม่** — regular 1,349, advance 8,094 |
| 75QNED86BSA.ATM | b13-60: 849 (ไม่ใช่ 699) |
| 27GX704A-B.ATM | เพิ่ม DC 50% 3M หลัง advance, saving=525 |
| 34GX90SA-W.ATM | **ลบออก** |
| SAQ/SIQ แอร์ | promoMonths: 12→6 |
| FV1413H4M ซักผ้า | promoMonths: 6→3 |
| IXY แอร์ Inverter | ราคาลด ~100 บาท |

---

## Dependencies & Patterns

- **Alpine.js 3.x** — inline ที่ท้าย `</body>` (standalone file)
- **JSONP** (`script tag`) — เพราะเปิดผ่าน `file://` protocol ไม่มี fetch CORS
- **localStorage** — `lgPromoConfig` reset อัตโนมัติเมื่อ dateStart เปลี่ยน
- **Safari** — ต้องมี `-webkit-backdrop-filter`, Alpine.js ก่อน `</body>`

---

## ไฟล์ reference (ไม่ต้องแก้บ่อย)

- `LG_Subscribe_Pricing_Formula.md` — สูตรคำนวณราคา
- `LG_Subscribe_Service_Policy.md` — นโยบายบริการ
- `Care Service LG Subscribe PDF.pdf` — รายละเอียด care service
- `linkSheet.txt` — URL Google Sheet + Apps Script
