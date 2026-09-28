# Part 091: Case Study - Core Banking ตอนที่ 1: การออกแบบระบบ (ขั้นตอนที่ 901–910)

## คำนำของ Part นี้

เดินทางมาถึงหนึ่งใน Case Study ที่สำคัญที่สุดของหลักสูตรทั้งหมด นับตั้งแต่ Part 086 เราเรียนรู้
สถาปัตยกรรมระบบองค์กรขนาดใหญ่ (Part 086), Microservices (Part 087), Performance ระดับสูง (Part 088),
Security ระดับ Enterprise (Part 089) และการจัดการโปรเจกต์ COBOL ขนาดใหญ่ (Part 090) — ทั้งหมดนี้คือ
"ทฤษฎีระดับมืออาชีพ" ที่เตรียมเราให้พร้อมสำหรับงานจริง

Part 091-093 คือจุดที่เราจะนำทุกอย่างที่เรียนมา**ตลอดทั้งหลักสูตร**มาประกอบร่างเป็นระบบเดียว: **ระบบ
Core Banking (ระบบธนาคารแกนกลาง)** ที่ออกแบบและสร้างขึ้นเองตั้งแต่ต้นจนจบ ครอบคลุม 3 Part ต่อเนื่องกัน:

- **Part 091 (Part นี้)**: การออกแบบระบบ — ข้อกำหนด, โครงสร้างข้อมูล, Copybook, สถาปัตยกรรม, กลยุทธ์
  จัดการข้อผิดพลาด และสร้าง/ทดสอบรากฐานของระบบ (Copybook + โปรแกรมตั้งค่าเริ่มต้น)
- **Part 092**: สร้างโมดูลบัญชีและธุรกรรม — ฝาก, ถอน, โอนเงินระหว่างบัญชี (พร้อมกลไกจำลอง Transaction
  Rollback), และการคำนวณดอกเบี้ยค้างรับรายวัน
- **Part 093**: รายงานและการปิดบัญชี — รายงานธุรกรรมรายวัน, ใบแจ้งยอดบัญชี, การกระทบยอดสิ้นวัน, การผ่าน
  รายการดอกเบี้ยสิ้นเดือน, รายงานสรุปปิดบัญชี และการทดสอบทั้งระบบแบบครบวงจร

### ทำไม Case Study นี้ถึงต่างจาก Part 070

Part 070 (โปรเจกต์เฟส 4) ได้สร้าง "ระบบธนาคารบน Mainframe จำลอง" ไปแล้วรอบหนึ่ง โดยเน้นแสดงให้เห็นว่า
"ฝั่งการประมวลผลไฟล์และแบทช์" ของ Mainframe มีหน้าตาอย่างไร (ผ่าน ACCTINIT, ACCTOPEN, ACCTDEP, ACCTWD,
ACCTINQ/ACCTCLOSE, BATCHINT, STMTGEN, RECONCILE) ระบบนั้นมีบัญชีเพียง 2 ประเภท (Savings/Checking) และ
ยังไม่มีการโอนเงินระหว่างบัญชี

Case Study นี้ถูกออกแบบให้ **กว้างและสมจริงกว่าเดิมอย่างชัดเจน**:

| หัวข้อ | Part 070 (ทบทวน) | Part 091-093 (Case Study นี้) |
|---|---|---|
| ประเภทบัญชี | 2 ประเภท (Savings, Checking) | 3 ประเภท (Savings, Checking + Overdraft, Fixed Deposit) |
| การโอนเงิน | ไม่มี | มี — พร้อมกลไก Compensating Transaction เต็มรูปแบบ |
| ดอกเบี้ย | อัตราคงที่ ผ่านรายการทุกคืน | อัตราแบบขั้นบันได (Tiered) ตามยอดคงเหลือ, คำนวณค้างรับรายวันแล้วผ่านรายการ (post) เฉพาะสิ้นเดือน |
| การกระทบยอด | ตรวจยอดควบคุมพื้นฐาน | ตรวจสอบหลักการบัญชีคู่ (Double-Entry) ของขาการโอนเงินโดยเฉพาะ |
| ความเสี่ยงจากการรันซ้ำ | ไม่ได้กล่าวถึง | ออกแบบ Idempotency Guard ป้องกันการผ่านรายการดอกเบี้ยซ้ำ |

กล่าวอีกนัยหนึ่ง: Part 070 ตอบคำถามว่า "แบตช์กับไฟล์ Indexed บน Mainframe ทำงานอย่างไร" ส่วน Case Study
นี้ตอบคำถามที่ลึกกว่า: **"ระบบธนาคารจริงจัดการกับความซับซ้อนทางธุรกิจและความเสี่ยงเชิงปฏิบัติการอย่างไร"**

### ประกาศความซื่อสัตย์ (อ่านก่อนเริ่ม)

เช่นเดียวกับ Part 070 ทุกตัวอย่างโค้ดในเอกสารนี้**คอมไพล์และรันได้จริงด้วย GnuCOBOL** ทุกผลลัพธ์ที่แสดง
คือผลลัพธ์จริงจากการรันจริง ไม่มีการ "แต่ง" ผลลัพธ์ขึ้นมา ระบบนี้ใช้ **Indexed File (ผ่าน Berkeley DB
เป็น ISAM handler)** แทน VSAM KSDS และใช้ **Sequential File** เป็น Audit Trail เช่นเดียวกับที่อธิบายไว้
ใน Part 028 และ Part 070

ตรวจสอบก่อนเริ่มเสมอ:

```bash
cobc -info | grep -i indexed
```

ถ้าไม่ขึ้นว่า `disabled` (เช่น `indexed file handler : BDB version 5.3.28`) แปลว่าพร้อมใช้งาน ตัวอย่าง
ทั้งหมดใน Part 091-093 ทดสอบจริงด้วย GnuCOBOL build ที่เปิดใช้ BDB ตัวเดียวกับที่ Part 028/070 ใช้

### ภาพรวม 10 ขั้นตอนของ Part นี้

| ขั้นตอน | สิ่งที่ทำ |
|---|---|
| 901 | ข้อกำหนดทางธุรกิจ (Requirements) และขอบเขตของ Case Study |
| 902 | ออกแบบ ACCOUNT-MASTER — โครงสร้างบัญชี 3 ประเภท |
| 903 | ออกแบบ TRANSACTION-LOG — Audit Trail แบบรองรับหลักบัญชีคู่ |
| 904 | หลักการออกแบบ Copybook + สร้าง `cbacct.cpy` |
| 905 | สร้าง `cbtxn.cpy` และเหตุผลเรื่อง SIGN คงเครื่องหมาย |
| 906 | ภาพรวมโมดูลทั้งระบบ (12 โปรแกรม) + แผนภาพสถาปัตยกรรม |
| 907 | มาตรฐานการตั้งชื่อและ Coding Standard ของ Case Study |
| 908 | กลยุทธ์จัดการข้อผิดพลาด: FILE STATUS, REJECTED, FATAL, Idempotency |
| 909 | สร้างและทดสอบ Copybook ทั้งสองไฟล์ผ่านโปรแกรมทดสอบ `TESTCPY` |
| 910 | สร้างและทดสอบ `CBINIT` (ตั้งค่าเริ่มต้น) และ `CBOPEN` (เปิดบัญชีใหม่) |

---

## ขั้นตอนที่ 901: ข้อกำหนดทางธุรกิจและขอบเขตของ Case Study

### เป้าหมายทางธุรกิจ

สมมติว่าเราคือทีมพัฒนาของธนาคารขนาดกลางแห่งหนึ่งที่ได้รับมอบหมายให้สร้าง **แกนระบบบัญชีเงินฝาก (Deposit
Core)** ขึ้นใหม่ด้วย COBOL ทีมธุรกิจให้ข้อกำหนดมาดังนี้:

1. **รองรับบัญชี 3 ประเภท**:
   - **Savings (ออมทรัพย์)** — มีดอกเบี้ย อัตราแบบขั้นบันไดตามยอดคงเหลือ ห้ามติดลบ
   - **Checking (กระแสรายวัน)** — ไม่มีดอกเบี้ย แต่มีวงเงินเบิกเกินบัญชี (Overdraft Limit) ที่กำหนดต่อบัญชี
   - **Fixed Deposit (เงินฝากประจำ)** — ดอกเบี้ยสูงกว่า Savings, ล็อกเงินจนครบกำหนด (Maturity),
     ถอนบางส่วนก่อนครบกำหนดไม่ได้เลย
2. **ธุรกรรมพื้นฐาน**: ฝากเงิน, ถอนเงิน (ตรวจสอบยอดคงเหลือ/วงเงินเบิกเกินบัญชีอย่างเข้มงวด)
3. **โอนเงินระหว่างบัญชี**: ต้องมีกลไกป้องกันไม่ให้เงิน "หายไป" หรือ "งอกขึ้นมาเอง" หากขั้นตอนใดขั้นตอนหนึ่ง
   ล้มเหลวกลางคัน (เช่น บัญชีปลายทางถูกปิดไปแล้ว)
4. **ดอกเบี้ย**: คำนวณค้างรับ (accrue) ทุกวัน แต่ผ่านรายการเข้าบัญชีจริง (capitalize) เฉพาะสิ้นเดือน
5. **ตรวจสอบความถูกต้องประจำวัน (End-of-Day)**: ต้องพิสูจน์ได้ว่าทุกธุรกรรมโอนเงินมี "ขาหนี้" (debit)
   และ "ขาเจ้าหนี้" (credit) จำนวนเท่ากันเสมอ (หลักบัญชีคู่ - Double-Entry Bookkeeping)
6. **รายงาน**: รายงานธุรกรรมรายวัน, ใบแจ้งยอดบัญชีรายลูกค้า, รายงานสรุปปิดบัญชี
7. **ทนต่อความผิดพลาดเชิงปฏิบัติการ**: หากมีคนรันงานแบตช์ปิดบัญชีสิ้นเดือนซ้ำโดยไม่ตั้งใจ ต้องไม่ทำให้
   ลูกค้าได้ดอกเบี้ยซ้ำสองครั้ง (Idempotency)

### สิ่งที่อยู่ในขอบเขต (In Scope)

- โปรแกรมออนไลน์แบบสั้น (Online-style, จำลองพฤติกรรมคล้าย CICS Transaction ตามแนวทาง Part 061-064
  และ Part 070): เปิดบัญชี, ฝาก, ถอน, โอน
- โปรแกรมแบตช์ (Batch, ตามแนวทาง Part 066): คำนวณดอกเบี้ยค้างรับรายวัน, กระทบยอดสิ้นวัน, ผ่านรายการ
  ดอกเบี้ยสิ้นเดือน, รายงานสรุป, ใบแจ้งยอด
- Audit Trail ที่บันทึกทุกธุรกรรม ทั้งที่สำเร็จและถูกปฏิเสธ

### สิ่งที่อยู่นอกขอบเขต (Out of Scope) — ประกาศไว้อย่างตรงไปตรงมา

- นี่**ไม่ใช่**ระบบที่รันบน z/OS จริง ไม่มี JCL, CICS, DB2 จริง (แนวคิดเหล่านี้สอนแยกไว้แล้วใน Part
  052-065) — เราใช้ GnuCOBOL Indexed File จำลอง VSAM KSDS เช่นเดียวกับที่ Part 028 และ Part 070 ทำ
- ไม่มีการเชื่อมต่อผู้ใช้พร้อมกันหลายคน (Concurrency/Locking) — ทุกโปรแกรมรันทีละครั้ง ทีละกระบวนการ
  (Single-user, single-process) ซึ่งเป็นข้อจำกัดที่ต้องระบุไว้ชัดเจนเมื่อพูดถึงกลไก "atomic-ish" ใน
  Part 092
- ไม่มีการเข้ารหัสหรือ Authentication ของผู้ใช้งาน (หัวข้อ Security แยกสอนไปแล้วใน Part 089)

### ตารางเชื่อมโยงข้อกำหนดกับขั้นตอนที่จะนำไปสร้างจริง (Requirements Traceability)

การเชื่อมโยงข้อกำหนดทางธุรกิจกับขั้นตอนที่จะนำไปสร้างจริงคือแนวปฏิบัติมาตรฐานของโปรเจกต์ COBOL ขนาดใหญ่
(ทบทวนจาก Part 090 เรื่องการจัดการโปรเจกต์) เพื่อให้มั่นใจว่าไม่มีข้อกำหนดใดถูกลืมไประหว่างการพัฒนา:

| ข้อกำหนด | สร้างจริงใน |
|---|---|
| 1. บัญชี 3 ประเภท | Part 091 ขั้นตอนที่ 902, 910 (`CBOPEN`) |
| 2. ฝาก/ถอน | Part 092 ขั้นตอนที่ 913-914 (`CBDEP`, `CBWD`) |
| 3. โอนเงินระหว่างบัญชีพร้อมกลไกป้องกันเงินหาย | Part 092 ขั้นตอนที่ 915-917 (`CBXFER`) |
| 4. ดอกเบี้ยค้างรับรายวัน + ผ่านรายการสิ้นเดือน | Part 092 ขั้นตอนที่ 918 (`CBACCR`) + Part 093 ขั้นตอนที่ 925 (`CBMEND`) |
| 5. ตรวจสอบหลักบัญชีคู่ประจำวัน | Part 093 ขั้นตอนที่ 924 (`CBEOD`) |
| 6. รายงานธุรกรรม/ใบแจ้งยอด/สรุปปิดบัญชี | Part 093 ขั้นตอนที่ 922, 923, 926 (`CBDAILY`, `CBSTMT`, `CBCLOSE`) |
| 7. ทนต่อการรันแบตช์ซ้ำ (Idempotency) | Part 093 ขั้นตอนที่ 925, 928 (`CBMEND` + แนวปฏิบัติเชิงปฏิบัติการ) |

ตารางนี้จะถูกอ้างอิงซ้ำในบทสรุปของ Part 093 เพื่อยืนยันว่าทุกข้อกำหนดได้รับการพิสูจน์ด้วยการทดสอบจริง
ครบถ้วนแล้วเมื่อจบ Case Study

### แบบฝึกหัดที่ 901.1

**โจทย์**: จงอธิบายว่าทำไมข้อกำหนด "ห้ามผ่านรายการดอกเบี้ยซ้ำหากรันแบตช์ผิดพลาดซ้ำ" (ข้อ 7) ถึงสำคัญ
พอๆ กับข้อกำหนดเรื่องความถูกต้องของการคำนวณดอกเบี้ยเอง

**เฉลยแนวทาง**: การคำนวณดอกเบี้ยผิดสร้างความเสียหายทางการเงินครั้งเดียวและตรวจพบได้จากยอดที่ผิดปกติ
แต่ "การรันซ้ำโดยไม่มีตัวป้องกัน" เป็นความเสี่ยงเชิงปฏิบัติการ (Operational Risk) ที่มักเกิดขึ้นจริงในงาน
แบตช์ (เช่น Job scheduler รันซ้ำ, operator กด submit ผิดสองครั้ง) และเมื่อเกิดขึ้น ผลกระทบจะกระจายไปยัง
ลูกค้าทุกบัญชีที่มีดอกเบี้ยพร้อมกันในคราวเดียว ธนาคารจริงจึงถือว่าการออกแบบให้แบตช์ "รันซ้ำได้อย่าง
ปลอดภัย" (Idempotent) เป็นข้อกำหนดพื้นฐานพอๆ กับความถูกต้องของสูตรคำนวณ

---

## ขั้นตอนที่ 902: ออกแบบ ACCOUNT-MASTER — โครงสร้างบัญชี 3 ประเภท

### ทำไมต้องใช้ Indexed File

เช่นเดียวกับ Part 028 และ Part 070 เราเลือก **Indexed File** (`ORGANIZATION IS INDEXED`) สำหรับเก็บ
สถานะปัจจุบันของบัญชี เพราะโปรแกรมออนไลน์ (ฝาก/ถอน/โอน) ต้องเข้าถึงบัญชีใดบัญชีหนึ่งแบบสุ่ม (Random
Access) ด้วยเลขที่บัญชีได้ทันที โดยไม่ต้องอ่านทั้งไฟล์ — พฤติกรรมแบบเดียวกับ VSAM KSDS บน Mainframe จริง

### ผังฟิลด์ ACCOUNT-MASTER

| ฟิลด์ | PICTURE | ความหมาย |
|---|---|---|
| `ACCT-ID` | `9(6)` | รหัสบัญชี (Record Key) |
| `ACCT-NAME` | `X(24)` | ชื่อเจ้าของบัญชี |
| `ACCT-TYPE` | `X(1)` | `S`=Savings, `C`=Checking, `F`=Fixed Deposit |
| `ACCT-STATUS` | `X(1)` | `A`=Active, `C`=Closed, `M`=Matured (เฉพาะ Fixed Deposit) |
| `ACCT-BALANCE` | `S9(9)V99` | ยอดคงเหลือ (Signed — Checking ติดลบได้ในวงเงินเบิกเกินบัญชี) |
| `ACCT-OVERDRAFT-LIMIT` | `9(7)V99` | วงเงินเบิกเกินบัญชี (เฉพาะ Checking, บัญชีอื่นเป็น 0) |
| `ACCT-INTEREST-RATE` | `9(1)V9(4)` | อัตราดอกเบี้ยพื้นฐานต่อปี |
| `ACCT-ACCRUED-INT` | `9(7)V99` | ดอกเบี้ยค้างรับที่ยังไม่ผ่านรายการเข้าบัญชีจริง |
| `ACCT-OPEN-DATE` | `9(8)` | วันที่เปิดบัญชี (YYYYMMDD) |
| `ACCT-MATURITY-DATE` | `9(8)` | วันครบกำหนด (เฉพาะ Fixed Deposit, บัญชีอื่นเป็น 0) |
| `ACCT-LAST-ACCR-DATE` | `9(8)` | วันที่คำนวณดอกเบี้ยค้างรับล่าสุด |
| `ACCT-LAST-STMT-DATE` | `9(8)` | วันที่ผ่านรายการ (post) ดอกเบี้ยครั้งล่าสุด — `0` แปลว่า "ยังไม่เคยผ่านรายการ" |

จุดที่ควรสังเกตเป็นพิเศษ (แตกต่างจาก Part 070 อย่างชัดเจน):

- **`ACCT-BALANCE` เป็น `S9(9)V99` (Signed)** ไม่ใช่ `9(9)V99` เหมือน Part 070 เพราะบัญชี Checking
  ต้องติดลบได้เมื่อใช้วงเงินเบิกเกินบัญชี
- **`ACCT-LAST-STMT-DATE` เริ่มต้นที่ 0 ไม่ใช่วันเปิดบัญชี** — นี่คือรายละเอียดเล็กๆ ที่สำคัญมาก
  (อธิบายเหตุผลเต็มๆ พร้อมบั๊กจริงที่พบระหว่างทดสอบใน Part 093 ขั้นตอนที่ 925) หากตั้งค่าเริ่มต้นเป็น
  วันเปิดบัญชี ตัวป้องกันการผ่านรายการซ้ำ (Idempotency Guard) ใน `CBMEND` จะเข้าใจผิดว่าบัญชีที่เพิ่งเปิด
  ใหม่ "ผ่านรายการดอกเบี้ยไปแล้ว" ในเดือนแรกทันที ทั้งที่ยังไม่เคยผ่านรายการเลยสักครั้ง

### กฎทางธุรกิจของบัญชีแต่ละประเภท (สรุปเพื่อใช้ตลอด 3 Part)

| กฎ | Savings | Checking | Fixed Deposit |
|---|---|---|---|
| ยอดเปิดบัญชีขั้นต่ำ | 500.00 | ไม่จำกัด (รวม 0.00) | 50,000.00 |
| มีดอกเบี้ยหรือไม่ | มี (ขั้นบันได) | ไม่มี (rate = 0) | มี (อัตราคงที่ สูงกว่า Savings) |
| ถอนเงินติดลบได้หรือไม่ | ไม่ได้ (ขั้นต่ำ 0) | ได้ ไม่เกินวงเงินเบิกเกินบัญชี | ถอนบางส่วนไม่ได้เลยจนกว่าจะครบกำหนด |
| วันครบกำหนด | ไม่มี | ไม่มี | มี (เปิดบัญชี + 12 เดือน) |

กฎเหล่านี้จะถูกนำไปใช้จริงในโปรแกรม `CBOPEN` (ขั้นตอนที่ 910) และ `CBWD`/`CBXFER` (Part 092)

### แบบฝึกหัดที่ 902.1

**โจทย์**: ทำไม `ACCT-INTEREST-RATE` และ `ACCT-OVERDRAFT-LIMIT` ถึงถูกออกแบบให้อยู่ใน record เดียวกัน
ทั้งที่บัญชีแต่ละประเภทใช้แค่ฟิลด์ใดฟิลด์หนึ่งเท่านั้น (Savings ไม่ใช้ Overdraft, Checking ไม่ใช้ Interest
Rate) ทำไมไม่แยกเป็นคนละ Record Layout ไปเลย?

**เฉลยแนวทาง**: การออกแบบ "หนึ่ง Record Layout ที่รองรับทุกประเภทบัญชี" (บางครั้งเรียกว่า wide-table /
superset design) ทำให้ทุกโปรแกรมใช้ Copybook เดียวกันได้โดยไม่ต้องมี CASE/REDEFINES ที่ซับซ้อนตามประเภท
บัญชี, การเขียนโปรแกรมที่ค้นหา วนอ่าน หรือ REWRITE บัญชีจึงทำได้ง่ายและเหมือนกันทุกประเภท ข้อเสียคือ
เปลืองพื้นที่เล็กน้อย (ฟิลด์ที่ไม่ใช้เป็น 0 เสมอ) ซึ่งในยุคที่พื้นที่เก็บข้อมูลราคาถูกกว่าอดีตมาก ถือเป็น
Trade-off ที่คุ้มค่ามากเมื่อแลกกับความง่ายในการดูแลรักษาโค้ด (Maintainability)

---

## ขั้นตอนที่ 903: ออกแบบ TRANSACTION-LOG — Audit Trail แบบรองรับหลักบัญชีคู่

### บทบาทของ TRANSACTION-LOG

เช่นเดียวกับ Part 070 เราใช้ **Sequential File แบบ append-only** (`TXNLOG.DAT`) เป็น Audit Trail คู่กับ
`ACCOUNT-MASTER` ตามรูปแบบ Master-Detail ที่เรียนมาตั้งแต่ Part 026: **Master เก็บ "สถานะปัจจุบัน"
ส่วน Sequential Log เก็บ "ประวัติเหตุการณ์ทั้งหมด"** ไม่ว่าจะสำเร็จหรือถูกปฏิเสธ

สิ่งที่ Case Study นี้เพิ่มเข้ามาจาก Part 070 คือฟิลด์ `TXN-GROUP-ID` ซึ่งเป็นหัวใจสำคัญของการทำธุรกรรม
โอนเงินและการกระทบยอดแบบหลักบัญชีคู่ใน Part 093

### ผังฟิลด์ TRANSACTION-LOG (แต่ละแถวคือบรรทัดข้อความความยาว 70 ตัวอักษร)

| ฟิลด์ | PICTURE | ความหมาย |
|---|---|---|
| `TXN-DATE` | `9(8)` | วันที่ทำรายการ (YYYYMMDD) |
| `TXN-TIME` | `9(6)` | เวลาทำรายการ (HHMMSS) |
| `TXN-GROUP-ID` | `9(8)` | เลขอ้างอิงที่เชื่อมสองขาของธุรกรรมโอนเงินเข้าด้วยกัน (`0` = ไม่เกี่ยวข้อง) |
| `TXN-ACCT-ID` | `9(6)` | รหัสบัญชีที่เกี่ยวข้องกับบรรทัดนี้ |
| `TXN-TYPE` | `X(4)` | ประเภทรายการ: `OPEN`, `DEP`, `WD`, `XFRD`, `XFRC`, `RVSL`, `INT`, `CLOS` |
| `TXN-AMOUNT` | `9(9)V99` | จำนวนเงินของรายการนี้ (เป็นบวกเสมอ ไม่ว่าจะเป็นขาฝากหรือขาถอน) |
| `TXN-BALANCE-AFTER` | `S9(9)V99` (SIGN LEADING SEPARATE) | ยอดคงเหลือหลังทำรายการ (ติดลบได้สำหรับ Checking) |
| `TXN-STATUS` | `X(8)` | `OK` หรือ `REJECTED` |

### ทำไมต้องมี `TXN-GROUP-ID`

นี่คือส่วนต่อยอดที่สำคัญที่สุดจาก Part 070 ธุรกรรมโอนเงินหนึ่งครั้งจะสร้าง**สองบรรทัดขึ้นไป**ใน Log:

- `XFRD` (transfer debit) — ขาหักเงินออกจากบัญชีต้นทาง
- `XFRC` (transfer credit) — ขาเติมเงินเข้าบัญชีปลายทาง
- `RVSL` (reversal) — ถ้าขา credit ทำไม่สำเร็จ จะมีขานี้เพิ่มมาเพื่อคืนเงินให้บัญชีต้นทาง

ทั้งสาม (หรือสอง) บรรทัดของธุรกรรมโอนเงิน**ครั้งเดียวกัน**จะมี `TXN-GROUP-ID` เดียวกันเสมอ ทำให้โปรแกรม
กระทบยอด (`CBEOD` ใน Part 093) สามารถพิสูจน์หลักบัญชีคู่ (Double-Entry) ได้ว่า **ทุกขาหักเงิน (debit) ต้อง
มีขาเติมเงิน (credit) จำนวนเท่ากันมารองรับเสมอ** — ถ้าไม่มี (เพราะขา credit ล้มเหลว) ต้องมีขา reversal
มาหักล้างแทน กลไกนี้จะอธิบายละเอียดพร้อมโค้ดจริงใน Part 092 ขั้นตอนที่ 915-917

### ทำไม `TXN-BALANCE-AFTER` ใช้ `SIGN IS LEADING SEPARATE`

`ACCT-BALANCE` ใน Master เป็น `S9(9)V99` ธรรมดา (เครื่องหมายฝังอยู่ในหลักสุดท้ายแบบ overpunch ซึ่งเป็น
ไบต์ที่ไม่ใช่ตัวเลขล้วนเมื่อค่าติดลบ) แต่ไฟล์ `TXNLOG.DAT` เป็น **LINE SEQUENTIAL แบบข้อความล้วน** ที่
มนุษย์ต้องเปิดอ่านได้ตรงๆ (เช่นด้วย `cat`) การใช้ `SIGN IS LEADING SEPARATE` บังคับให้เครื่องหมาย `+`
หรือ `-` เป็นตัวอักษรแยกต่างหากที่ตำแหน่งหน้าสุดของฟิลด์ ทำให้ค่าทุกไบต์ในไฟล์ log เป็นตัวอักษรที่มนุษย์
อ่านได้ทั้งหมด (ตรงตามหลักการอ่านง่ายของ COBOL ที่ Part 001 กล่าวถึง) แทนที่จะมีไบต์แปลกปนอยู่

### แบบฝึกหัดที่ 903.1

**โจทย์**: สมมติธุรกรรมโอนเงินหนึ่งครั้งล้มเหลวตั้งแต่ขั้นตอนตรวจสอบบัญชีต้นทาง (เช่น ยอดไม่พอ) ก่อนที่
จะมีการหักเงินออกจากบัญชีใดเลย จะมีบรรทัดใน `TXNLOG.DAT` กี่บรรทัด และมี `TXN-GROUP-ID` หรือไม่

**เฉลยแนวทาง**: จะมีเพียง**หนึ่งบรรทัด**เท่านั้น คือ `XFRD` ที่มี `TXN-STATUS = REJECTED` (บันทึกไว้เพื่อ
เป็นหลักฐานว่ามีการพยายามทำธุรกรรมนี้) โดยไม่มีการหักเงินจริงเกิดขึ้น (ยอดคงเหลือหลังทำรายการในบรรทัดนี้
จะเท่ากับยอดคงเหลือก่อนทำรายการ) และยังคงมี `TXN-GROUP-ID` ติดมาด้วย เพราะเลขอ้างอิงถูกกำหนดจากอินพุต
ของผู้ใช้ตั้งแต่ต้น ไม่ว่าธุรกรรมจะสำเร็จหรือล้มเหลว

---

## ขั้นตอนที่ 904: หลักการออกแบบ Copybook และสร้าง `cbacct.cpy`

### ทำไม Copybook ยิ่งสำคัญขึ้นเมื่อระบบมีหลายโปรแกรม

ระบบนี้มีโปรแกรมที่แตะ `ACCOUNT-MASTER` ถึง **8 โปรแกรม** (`CBOPEN`, `CBDEP`, `CBWD`, `CBXFER`, `CBACCR`,
`CBSTMT`, `CBEOD` ไม่แตะ Master โดยตรงแต่ `CBMEND`, `CBCLOSE` แตะ) และมีโปรแกรมที่แตะ `TXNLOG.DAT` ถึง
**6 โปรแกรม** ยิ่งจำนวนโปรแกรมมาก ความเสี่ยงที่ใครสักคนจะพิมพ์ `PIC 9(7)V99` แทนที่จะเป็น `PIC 9(9)V99`
ในโปรแกรมใดโปรแกรมหนึ่งก็ยิ่งสูงขึ้น (ปัญหานี้ Part 031 เคยพิสูจน์ให้เห็นแล้วว่าข้อมูลจะตีความผิดเพี้ยน
แบบไม่มี error เตือนใดๆ) — Copybook (Part 033) คือทางแก้เดียวที่รับประกันว่าโครงสร้างข้อมูลจะตรงกัน
100% ในทุกโปรแกรมเสมอ เพราะทุกโปรแกรมดึงเนื้อหาจาก**ไฟล์เดียวกัน**

### `cbacct.cpy`

```cobol
      ******************************************************************
      * CBACCT.CPY
      * Shared ACCOUNT-MASTER record layout for the Core Banking case
      * study (Part 091-093). One record = one bank account. Every
      * program that touches the indexed ACCOUNT-MASTER file COPYs
      * this layout into its FD, so the structure can never drift
      * between programs (Part 033 technique).
      ******************************************************************
           05  ACCT-ID                 PIC 9(6).
           05  ACCT-NAME               PIC X(24).
           05  ACCT-TYPE               PIC X(1).
               88  ACCT-TYPE-SAVINGS       VALUE "S".
               88  ACCT-TYPE-CHECKING      VALUE "C".
               88  ACCT-TYPE-FIXED-DEP     VALUE "F".
           05  ACCT-STATUS             PIC X(1).
               88  ACCT-STATUS-ACTIVE      VALUE "A".
               88  ACCT-STATUS-CLOSED      VALUE "C".
               88  ACCT-STATUS-MATURED     VALUE "M".
           05  ACCT-BALANCE            PIC S9(9)V99.
           05  ACCT-OVERDRAFT-LIMIT    PIC 9(7)V99.
           05  ACCT-INTEREST-RATE      PIC 9(1)V9(4).
           05  ACCT-ACCRUED-INT        PIC 9(7)V99.
           05  ACCT-OPEN-DATE          PIC 9(8).
           05  ACCT-MATURITY-DATE      PIC 9(8).
           05  ACCT-LAST-ACCR-DATE     PIC 9(8).
           05  ACCT-LAST-STMT-DATE     PIC 9(8).
```

บันทึกไว้เป็นไฟล์ `cbacct.cpy` ในโฟลเดอร์เดียวกับไฟล์ `.cob` ทั้งหมด สังเกตว่าเราใส่ **88-level condition
names** (`ACCT-TYPE-SAVINGS`, `ACCT-STATUS-ACTIVE` ฯลฯ) ไว้ในตัว Copybook เองด้วย (เทคนิคจาก Part 010)
เพื่อให้ทุกโปรแกรมที่ COPY ไฟล์นี้ได้เงื่อนไขที่อ่านง่ายไปพร้อมกันโดยอัตโนมัติ ไม่ต้องมาประกาศซ้ำเอง
ทีละโปรแกรม

### แบบฝึกหัดที่ 904.1

**โจทย์**: หากในอนาคตธนาคารต้องการเพิ่มบัญชีประเภทที่ 4 (เช่น "Youth Savings" สำหรับเด็ก) จะต้องแก้ไข
ที่ใดบ้าง และ Copybook ช่วยลดงานตรงจุดนี้อย่างไร

**เฉลยแนวทาง**: ต้องเพิ่ม 88-level ใหม่ (เช่น `88 ACCT-TYPE-YOUTH VALUE "Y".`) ใน `cbacct.cpy` เพียง
**ที่เดียว** จากนั้นทุกโปรแกรมที่ COPY ไฟล์นี้จะมองเห็นเงื่อนไขใหม่ได้ทันทีโดยไม่ต้องแก้ไฟล์อื่นเลย สิ่งที่
ยังต้องแก้แยกต่างหากคือ**ตรรกะทางธุรกิจ**ที่เกี่ยวกับประเภทใหม่ (เช่น กฎยอดเปิดบัญชีขั้นต่ำใน `CBOPEN`
หรืออัตราดอกเบี้ยใน `CBACCR`) ซึ่งเป็นเรื่องคาดหวังอยู่แล้วเพราะเป็น Business Logic ที่ผูกกับแต่ละโปรแกรม
ไม่ใช่ปัญหาเรื่องโครงสร้างข้อมูลไม่ตรงกัน

---

## ขั้นตอนที่ 905: สร้าง `cbtxn.cpy` และเหตุผลเรื่องเครื่องหมาย

### `cbtxn.cpy`

```cobol
      ******************************************************************
      * CBTXN.CPY
      * Shared TRANSACTION-LOG record layout (sequential, append-only
      * audit trail). Every program that CALLs CBAUDIT to write a log
      * line, and every batch report program that reads TXNLOG.DAT
      * back, COPYs this same layout so field positions can never
      * drift between writers and readers.
      ******************************************************************
           05  TXN-DATE                PIC 9(8).
           05  FILLER                  PIC X(1)  VALUE SPACE.
           05  TXN-TIME                PIC 9(6).
           05  FILLER                  PIC X(1)  VALUE SPACE.
           05  TXN-GROUP-ID            PIC 9(8).
           05  FILLER                  PIC X(1)  VALUE SPACE.
           05  TXN-ACCT-ID             PIC 9(6).
           05  FILLER                  PIC X(1)  VALUE SPACE.
           05  TXN-TYPE                PIC X(4).
           05  FILLER                  PIC X(1)  VALUE SPACE.
           05  TXN-AMOUNT              PIC 9(9)V99.
           05  FILLER                  PIC X(1)  VALUE SPACE.
           05  TXN-BALANCE-AFTER       PIC S9(9)V99
                                       SIGN IS LEADING SEPARATE.
           05  FILLER                  PIC X(1)  VALUE SPACE.
           05  TXN-STATUS              PIC X(8).
```

รวมความยาวทั้งบรรทัด: `8+1+6+1+8+1+6+1+4+1+11+1+12+1+8 = 70` ไบต์ ไม่รวมตัวขึ้นบรรทัดใหม่ที่ LINE
SEQUENTIAL จัดการให้อัตโนมัติ

### บทเรียนสำคัญที่พบระหว่างทดสอบจริง: FILE SECTION ไม่ได้ initialize VALUE clause ให้อัตโนมัติ

ระหว่างพัฒนา Case Study นี้ เราพบบั๊กจริงที่คุ้มค่ามากที่จะเล่าไว้ล่วงหน้า (รายละเอียดเต็มอยู่ใน Part 092
ขั้นตอนที่ 912 ตอนสร้าง `CBAUDIT`): แม้ `cbtxn.cpy` จะระบุ `FILLER PIC X(1) VALUE SPACE` ไว้ชัดเจน แต่
**`VALUE` clause ของฟิลด์ใน FILE SECTION จะไม่ถูกนำมาใช้ตั้งค่าเริ่มต้นให้อัตโนมัติเหมือนใน WORKING-
STORAGE SECTION** พื้นที่หน่วยความจำของ record ใน FD จะเป็นค่าขยะ (มักเป็นไบต์ 0x00 จากหน่วยความจำที่
ระบบปฏิบัติการเพิ่งจองให้) จนกว่าโปรแกรมจะ MOVE ค่าลงไปเอง ผลคือถ้าลืม `MOVE SPACES` ก่อนกรอกฟิลด์ทีละ
ตัว ไบต์ FILLER ที่ควรเป็นช่องว่างจะกลายเป็นอักขระที่พิมพ์ไม่ได้ และเมื่อ WRITE ลง LINE SEQUENTIAL file
GnuCOBOL จะปฏิเสธด้วย **FILE STATUS "71" (Bad Character)** ทันที — เราจะเห็นโค้ดที่แก้ปัญหานี้จริงในขั้น
ตอนที่ 912

### แบบฝึกหัดที่ 905.1

**โจทย์**: ทำไมปัญหา "VALUE clause ไม่ initialize ให้อัตโนมัติ" ถึงเกิดกับ FILE SECTION แต่ไม่เกิดกับ
WORKING-STORAGE SECTION?

**เฉลยแนวทาง**: ตัวแปรใน WORKING-STORAGE SECTION มีอยู่เพียงชุดเดียวตลอดอายุโปรแกรม คอมไพเลอร์จึงตั้งค่า
เริ่มต้นให้ครั้งเดียวตอนโปรแกรมเริ่มทำงาน (load time) ได้อย่างมีประสิทธิภาพ ส่วน Record ใน FILE SECTION
ถูกออกแบบมาให้เป็น "หน้าต่าง" ที่ข้อมูลจากไฟล์ไหลผ่านเข้าออกซ้ำๆ หลายพันหรือหลายล้านครั้งต่อการรันหนึ่ง
ครั้ง (ทุกครั้งที่ READ หรือก่อน WRITE) การ initialize ค่าตาม VALUE clause ให้ทุกครั้งก่อนแต่ละ READ/WRITE
จะเสียประสิทธิภาพโดยใช่เหตุ (ย้อนกลับไปดูหลักการ Performance จาก Part 088) มาตรฐาน COBOL จึงกำหนดให้
โปรแกรมเมอร์เป็นผู้รับผิดชอบ initialize record ของตัวเองก่อนใช้งานแทน

---

## ขั้นตอนที่ 906: ภาพรวมโมดูลทั้งระบบและแผนภาพสถาปัตยกรรม

### รายชื่อโปรแกรมทั้งหมดของ Case Study (12 โปรแกรม + 2 Copybook)

| โปรแกรม | ประเภท | สร้างใน | หน้าที่ |
|---|---|---|---|
| `CBINIT` | Setup (รันครั้งเดียว) | Part 091 | สร้าง `ACCTMAST.DAT` และ `TXNLOG.DAT` เปล่า |
| `CBOPEN` | Online-style | Part 091 | เปิดบัญชีใหม่ (ตรวจกฎตามประเภทบัญชี) |
| `CBAUDIT` | Subprogram (CALL) | Part 092 | เขียนบรรทัด Audit Trail ให้ทุกโปรแกรม |
| `CBDEP` | Online-style | Part 092 | ฝากเงิน |
| `CBWD` | Online-style | Part 092 | ถอนเงิน (ตรวจยอด/วงเงินเบิกเกินบัญชี/ล็อก Fixed Deposit) |
| `CBXFER` | Online-style | Part 092 | โอนเงินระหว่างบัญชี พร้อม Compensating Transaction |
| `CBACCR` | Batch (รายวัน) | Part 092 | คำนวณดอกเบี้ยค้างรับ (ขั้นบันไดสำหรับ Savings) |
| `CBDAILY` | Batch (รายวัน) | Part 093 | รายงานธุรกรรมประจำวัน |
| `CBSTMT` | Batch | Part 093 | ใบแจ้งยอดบัญชี (SORT + Control Break) |
| `CBEOD` | Batch (รายวัน) | Part 093 | กระทบยอดสิ้นวัน (ตรวจหลักบัญชีคู่) |
| `CBMEND` | Batch (รายเดือน) | Part 093 | ผ่านรายการดอกเบี้ยสิ้นเดือน + ครบกำหนด Fixed Deposit |
| `CBCLOSE` | Batch | Part 093 | รายงานสรุปปิดบัญชี |

### แผนภาพสถาปัตยกรรมทั้งระบบ

```
                         +------------------+
                         |     CBINIT       |  (run once - creates
                         +--------+---------+   empty master + log)
                                  |
      +---------------------------+---------------------------+
      |            ONLINE-STYLE TRANSACTIONS                   |
      |     (short-lived, one customer request per run,        |
      |      simulates a CICS pseudo-conversational task)      |
      |                                                         |
      |   CBOPEN     CBDEP     CBWD     CBXFER                  |
      |      |          |        |         |                   |
      |      +----+-----+---+----+----+----+                   |
      |           |         |         |                        |
      |           v         v         v                        |
      |      ACCOUNT-MASTER (indexed)   CBAUDIT (CALL) --+      |
      |                                                   |      |
      |                                                   v      |
      |                                            TXNLOG.DAT    |
      |                                          (sequential,    |
      |                                             append)      |
      +---------------------------+----------------------------+
                                  | (end of business day)
      +---------------------------+----------------------------+
      |             NIGHTLY / BATCH WINDOW (Part 066)            |
      |                                                          |
      |   CBACCR   --reads/rewrites-->  ACCOUNT-MASTER            |
      |   CBDAILY  --reads-->           TXNLOG.DAT                |
      |   CBSTMT   --SORT+reads-->      TXNLOG.DAT + ACCOUNT-MASTER|
      |   CBEOD    --reads-->           TXNLOG.DAT                |
      |                                                          |
      |             (repeats every business day)                |
      +----------------------------+----------------------------+
                                   | (end of month)
      +----------------------------+----------------------------+
      |               MONTH-END BATCH WINDOW                     |
      |                                                          |
      |   CBMEND   --reads/rewrites-->  ACCOUNT-MASTER            |
      |   CBCLOSE  --reads-->           ACCOUNT-MASTER            |
      +----------------------------------------------------------+
```

สังเกตว่าโครงสร้างนี้แบ่งเวลาการทำงานเป็น 3 ชั้นตามความถี่: **ต่อธุรกรรม** (online) -> **ต่อวัน** (daily
batch) -> **ต่อเดือน** (month-end batch) ซึ่งเป็นรูปแบบเดียวกับที่ธนาคารจริงใช้งานตลอดมา

### ความสัมพันธ์กับสถาปัตยกรรมองค์กรที่เรียนมาใน Part 086

Part 086 (สถาปัตยกรรมระบบองค์กรขนาดใหญ่) แนะนำแนวคิดการแบ่งชั้นระบบ (Layered Architecture) ออกเป็น
ชั้นนำเสนอ (Presentation), ชั้นตรรกะทางธุรกิจ (Business Logic), และชั้นข้อมูล (Data) — Case Study นี้คือ
ตัวอย่างที่เป็นรูปธรรมของแนวคิดนั้น แม้จะไม่มีชั้นนำเสนอแบบกราฟิกจริง (เพราะเราใช้ `ACCEPT`/`DISPLAY`
ผ่าน terminal แทน) แต่การแบ่งแยกที่ชัดเจนระหว่าง **ตรรกะทางธุรกิจ** (โค้ดใน PROCEDURE DIVISION ของแต่ละ
โปรแกรม เช่น กฎวงเงินเบิกเกินบัญชีใน `CBWD`) กับ **ชั้นข้อมูล** (Copybook + ไฟล์ Indexed/Sequential ที่
ทุกโปรแกรมแชร์ร่วมกัน) คือหัวใจเดียวกันของการออกแบบแบบแบ่งชั้น เพียงแค่ปรับให้เข้ากับข้อจำกัดและจุดแข็ง
ของ COBOL/ISAM โดยเฉพาะ

### แบบฝึกหัดที่ 906.1

**โจทย์**: จากแผนภาพข้างต้น จงอธิบายว่าทำไม `CBACCR` (คำนวณดอกเบี้ยค้างรับ) ถึงถูกจัดอยู่ใน "batch
window รายวัน" ในขณะที่ `CBMEND` (ผ่านรายการดอกเบี้ยจริง) ถูกจัดอยู่ใน "batch window รายเดือน"

**เฉลยแนวทาง**: ดอกเบี้ยธนาคารเกิดขึ้นทุกวันตามยอดคงเหลือของวันนั้นๆ ดังนั้นการคำนวณสะสม (accrue) ต้อง
ทำทุกวันจึงจะแม่นยำ (`CBACCR` จึงอยู่ใน batch รายวัน) แต่การ "จ่ายจริง" เข้าบัญชี (capitalize) เป็น
ธรรมเนียมทางธุรกิจที่ธนาคารส่วนใหญ่เลือกทำเดือนละครั้งเพื่อลดจำนวนรายการในบัญชีลูกค้าและให้ตรงกับรอบ
ใบแจ้งยอด (statement cycle) การแยกสองขั้นตอนนี้ออกจากกันคือสิ่งที่ทำให้ระบบนี้สมจริงกว่า Part 070
ซึ่งคำนวณและผ่านรายการดอกเบี้ยพร้อมกันทุกคืน

---

## ขั้นตอนที่ 907: มาตรฐานการตั้งชื่อและ Coding Standard ของ Case Study

การมีมาตรฐานที่ชัดเจนตั้งแต่ต้น (ตามแนวทาง Clean Code ที่สอนใน Part 081) ช่วยให้โปรแกรมทั้ง 12 ตัว
อ่านแล้วรู้สึกเหมือนเขียนโดยทีมเดียวกัน:

1. **ชื่อโปรแกรม**: ขึ้นต้นด้วย `CB` (Core Banking) ตามด้วยคำย่อของหน้าที่ ยาวไม่เกิน 8 ตัวอักษร (ตาม
   ธรรมเนียม PROGRAM-ID บน Mainframe ที่ Part 051 กล่าวถึง)
2. **ชื่อฟิลด์ใน Copybook**: ขึ้นต้นด้วยคำนำหน้าของ record นั้น (`ACCT-` สำหรับบัญชี, `TXN-` สำหรับ
   ธุรกรรม) เพื่อให้ระบุที่มาได้ทันทีเมื่อเห็นชื่อฟิลด์ลอยๆ ในโค้ด
3. **ตัวแปร WORKING-STORAGE**: ขึ้นต้นด้วย `WS-` เสมอ (ทบทวนจาก Part 005)
4. **ค่าคงที่ธุรกิจ** (เช่น อัตราดอกเบี้ย, ยอดขั้นต่ำ): ประกาศเป็น `01` level ใน WORKING-STORAGE พร้อม
   คอมเมนต์อธิบายที่มา แทนที่จะฝังตัวเลข "ลอยๆ" (Magic Number) ไว้กลาง PROCEDURE DIVISION
5. **สถานะของธุรกรรมใน Log**: มีแค่สองค่าเสมอคือ `"OK"` และ `"REJECTED"` (ตัวพิมพ์ใหญ่ ไม่มีคำอื่นปน)
   เพื่อให้โปรแกรมรายงาน (Part 093) ใช้ `EVALUATE`/`IF` เทียบค่าได้ตรงไปตรงมา
6. **การส่ง Literal เข้า CALL**: ห้ามส่ง String literal สั้นกว่าฟิลด์ปลายทางใน LINKAGE SECTION โดยตรง
   (เหตุผลเต็มๆ อยู่ในขั้นตอนที่ 908 และ Part 092 ขั้นตอนที่ 912) ให้ประกาศเป็นค่าคงที่ WORKING-STORAGE
   ที่มีความยาวตรงกับพารามิเตอร์ปลายทางเสมอ เช่น `01 WS-STATUS-OK PIC X(8) VALUE "OK".`

### แบบฝึกหัดที่ 907.1

**โจทย์**: มาตรฐานข้อ 6 (ห้ามส่ง Literal สั้นเข้า CALL โดยตรง) ดูเหมือนเป็นรายละเอียดปลีกย่อย แต่ทำไม
เอกสารนี้ถึงยกระดับให้เป็น "มาตรฐานบังคับ" ของทั้งระบบ

**เฉลยแนวทาง**: เพราะนี่ไม่ใช่แค่เรื่องสไตล์การเขียนโค้ด แต่เป็นการป้องกัน**บั๊กที่ทำให้เกิดข้อมูลเพี้ยน
แบบไม่มี error เตือน** — เมื่อ CALL แบบ `BY REFERENCE` (ค่าเริ่มต้นของ COBOL) ส่ง Literal สั้นกว่าที่
พารามิเตอร์ปลายทางต้องการ คอมไพเลอร์จะจองพื้นที่ให้ Literal นั้นแค่เท่าความยาวจริงเท่านั้น เมื่อโปรแกรม
ปลายทางอ่านค่าตามความยาวที่ตัวเองประกาศไว้ (ซึ่งยาวกว่า) จะเกิดการอ่านเลยขอบเขตหน่วยความจำที่จองไว้จริง
(อ่านค่าขยะ) เราจะเห็นบั๊กนี้เกิดขึ้นจริงและแก้ไขจริงใน Part 092 ขั้นตอนที่ 912 — การตั้งเป็นมาตรฐานบังคับ
ไว้ล่วงหน้าคือการป้องกันไม่ให้บั๊กแบบเดียวกันเกิดซ้ำในโปรแกรมอื่นๆ ของระบบ

---

## ขั้นตอนที่ 908: กลยุทธ์จัดการข้อผิดพลาด

### สามระดับของการจัดการข้อผิดพลาดในระบบนี้

1. **FILE STATUS ระดับไฟล์** (ทบทวนจาก Part 030): ทุก `SELECT` มี `FILE STATUS IS WS-xxx-STATUS` และ
   ทุก READ/WRITE/REWRITE ที่สำคัญตรวจสอบค่านี้เสมอ นี่คือชั้นการป้องกันที่ใกล้กับฮาร์ดแวร์/ระบบไฟล์มาก
   ที่สุด (เช่น ไฟล์เปิดไม่สำเร็จ, key ซ้ำ, boundary violation)
2. **การตรวจสอบกฎธุรกิจระดับ Field/Record**: เช่น ยอดเงินต้องเป็นตัวเลข, บัญชีต้องยังเปิดอยู่, ยอด
   คงเหลือต้องพอสำหรับการถอน — ทุกจุดตรวจใช้ `WS-VALID-FLAG` กับ 88-level `WS-ALL-VALID` แบบเดียวกับ
   Part 069 (Secure Coding: ตรวจสอบอินพุตก่อนเชื่อถือเสมอ) และเมื่อพบข้อผิดพลาด โปรแกรมจะ**เขียน log
   entry สถานะ `REJECTED`** ก่อนหยุดทำงานเสมอ (ยกเว้นกรณี Fail-fast ก่อนรู้จักบัญชีเลย)
3. **ข้อผิดพลาดระดับ Transaction (Multi-record)**: เกิดเฉพาะใน `CBXFER` ซึ่งมีการเขียนสองบัญชีในธุรกรรม
   เดียว ถ้าขาที่สองล้มเหลวหลังขาแรกสำเร็จไปแล้ว ระบบต้องทำ **Compensating Transaction** (คืนเงินขาแรก
   กลับ) แทนที่จะปล่อยให้ข้อมูลค้างอยู่ในสถานะครึ่งๆ กลางๆ — รายละเอียดเต็มอยู่ใน Part 092 ขั้นตอนที่
   915-917

### ตาราง FILE STATUS ที่พบบ่อยที่สุดใน Case Study นี้ (ทบทวนจาก Part 030)

| FILE STATUS | ความหมาย | เกิดขึ้นได้ที่ไหนในระบบนี้ |
|---|---|---|
| `00` | สำเร็จ | ทุก I/O ที่ทำงานปกติ |
| `10` | อ่านถึงจุดสิ้นสุดไฟล์ (AT END) | `READ NEXT RECORD` ใน `CBACCR`/`CBMEND`/`CBCLOSE` เมื่อวนอ่านครบทุกบัญชีแล้ว |
| `21` | ลำดับคีย์ผิด (Indexed) | ไม่ควรเกิดในระบบนี้เพราะใช้ `ACCESS MODE IS DYNAMIC` เสมอ |
| `23` | ไม่พบเรคคอร์ดตามคีย์ที่ระบุ | `READ` บัญชีที่ไม่มีอยู่จริงใน `CBDEP`/`CBWD`/`CBXFER` |
| `35` | เปิดไฟล์ไม่สำเร็จเพราะไม่พบไฟล์ | `OPEN EXTEND` ใน `CBAUDIT` ตอนที่ `TXNLOG.DAT` ยังไม่เคยถูกสร้างมาก่อน |
| `71` | มีอักขระที่พิมพ์ไม่ได้ปนอยู่ใน LINE SEQUENTIAL record | บั๊กจริงที่พบใน `CBAUDIT` ระหว่างพัฒนา (รายละเอียดเต็มใน Part 092 ขั้นตอนที่ 912) |

การรู้จักความหมายของ FILE STATUS แต่ละค่าล่วงหน้าทำให้เมื่อพบค่าที่ไม่คาดคิดระหว่างพัฒนา เราสามารถ
วิเคราะห์สาเหตุได้รวดเร็วกว่าการเดาไปเรื่อยๆ — ตารางนี้จะถูกอ้างอิงซ้ำเมื่อเราพบบั๊กจริงใน Part 092

### REJECTED กับ FATAL: สองระดับที่ไม่เหมือนกัน

ตลอดทั้งระบบเราแยกคำอย่างมีความหมาย:

- **REJECTED** = ธุรกิจปฏิเสธธุรกรรมนี้ตามกฎที่ตั้งไว้ (เช่น ยอดไม่พอ) ถือเป็น**เหตุการณ์ปกติที่คาดไว้
  ล่วงหน้า** ระบบยังทำงานถูกต้องสมบูรณ์ — เพียงแค่ปฏิเสธคำขอนั้น
- **FATAL** = เกิดสิ่งที่ไม่ควรเกิดขึ้นเลยในทางทฤษฎี (เช่น REWRITE ล้มเหลวหลังผ่านการตรวจสอบทุกอย่างแล้ว,
  หรือบัญชีที่เพิ่งอ่านสำเร็จหายไปกลางคัน) กรณีนี้โปรแกรมจะ `DISPLAY` ข้อความเตือนชัดเจนว่าต้อง **"manual
  correction required"** เพราะเป็นสถานการณ์ที่ไม่มีกฎธุรกิจใดรองรับไว้ล่วงหน้า ต้องให้คนเข้ามาตรวจสอบ

การแยกสองคำนี้ทำให้ทีมปฏิบัติการ (Operations) อ่าน log แล้วรู้ทันทีว่าเหตุการณ์ไหน "ปกติ ไม่ต้องทำอะไร"
กับเหตุการณ์ไหน "ต้องมีคนเข้ามาดูด่วน"

### แบบฝึกหัดที่ 908.1

**โจทย์**: ในระบบนี้ การที่ลูกค้าพยายามถอนเงินเกินยอดคงเหลือ ถูกจัดเป็น REJECTED หรือ FATAL? เพราะเหตุใด

**เฉลยแนวทาง**: เป็น **REJECTED** เพราะเป็นเหตุการณ์ที่กฎธุรกิจ (business rule) ครอบคลุมไว้อยู่แล้วตั้งแต่
ขั้นตอนออกแบบ (ขั้นตอนที่ 901-902) ระบบตรวจพบและปฏิเสธได้อย่างถูกต้องตามที่ออกแบบไว้ทุกประการ ไม่มีความ
ผิดปกติทางเทคนิคใดๆ เกิดขึ้น จึงไม่ต้องมีคนเข้ามาแก้ไขอะไร เพียงแค่แจ้งลูกค้าว่ายอดไม่พอเท่านั้น

---

## ขั้นตอนที่ 909: สร้างและทดสอบ Copybook ทั้งสองไฟล์

### โปรแกรมทดสอบ `TESTCPY`

ก่อนใช้ Copybook ในระบบจริง เราควรพิสูจน์ก่อนว่ามันถูกต้องทางไวยากรณ์และใช้งานได้จริงด้วยโปรแกรมทดสอบสั้นๆ
ที่ไม่ใช่ส่วนหนึ่งของระบบจริง (throwaway harness):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TESTCPY.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * TESTCPY - compile-time proof that CBACCT.CPY and CBTXN.CPY are
      * syntactically valid and usable inside an ordinary 01 record in
      * WORKING-STORAGE. Not part of the production system - this is a
      * throwaway harness used only to verify the copybooks.
      ******************************************************************

       ENVIRONMENT DIVISION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ACCOUNT-RECORD.
           COPY "cbacct.cpy".

       01  WS-TXN-RECORD.
           COPY "cbtxn.cpy".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 100001         TO ACCT-ID.
           MOVE "JANE DOE"      TO ACCT-NAME.
           MOVE "S"             TO ACCT-TYPE.
           MOVE "A"             TO ACCT-STATUS.
           MOVE 1500.00         TO ACCT-BALANCE.
           MOVE 0.0125          TO ACCT-INTEREST-RATE.

           DISPLAY "ACCT-ID        = " ACCT-ID.
           DISPLAY "ACCT-NAME      = " ACCT-NAME.
           DISPLAY "ACCT-TYPE      = " ACCT-TYPE.
           IF ACCT-TYPE-SAVINGS
               DISPLAY "IS SAVINGS 88  = TRUE"
           END-IF.
           DISPLAY "ACCT-BALANCE   = " ACCT-BALANCE.

           MOVE 20260401        TO TXN-DATE.
           MOVE 093000           TO TXN-TIME.
           MOVE 1                TO TXN-GROUP-ID.
           MOVE 100001            TO TXN-ACCT-ID.
           MOVE "DEP"             TO TXN-TYPE.
           MOVE 1500.00           TO TXN-AMOUNT.
           MOVE 1500.00           TO TXN-BALANCE-AFTER.
           MOVE "OK"               TO TXN-STATUS.

           DISPLAY "TXN-TYPE       = " TXN-TYPE.
           DISPLAY "TXN-BALANCE-AFTER = " TXN-BALANCE-AFTER.

           STOP RUN.
```

สังเกตว่าเราตรวจสอบ 88-level condition name ด้วย `IF ACCT-TYPE-SAVINGS` แทนที่จะ `DISPLAY` มันตรงๆ —
GnuCOBOL ไม่อนุญาตให้ `DISPLAY` condition-name โดยตรง (มันไม่ใช่ข้อมูล แต่เป็นเงื่อนไข) ต้องใช้ `IF` เพื่อ
ประเมินค่าความจริงก่อนเสมอ

คอมไพล์และรัน (ต้องใช้ GnuCOBOL build ที่เปิด Indexed File เพราะโปรแกรมอื่นในระบบต้องใช้ แม้ `TESTCPY`
เองจะไม่แตะไฟล์ก็ตาม เพื่อความสม่ำเสมอของสภาพแวดล้อม):

```bash
cobc -x -o testcpy testcpy.cob
./testcpy
```

**ผลลัพธ์จริง**:

```
ACCT-ID        = 100001
ACCT-NAME      = JANE DOE
ACCT-TYPE      = S
IS SAVINGS 88  = TRUE
ACCT-BALANCE   = +000001500.00
TXN-TYPE       = DEP
TXN-BALANCE-AFTER = +000001500.00
```

Copybook ทั้งสองไฟล์ผ่านการทดสอบ: ประกาศได้ถูกต้อง, 88-level ทำงานถูกต้อง, และค่าตัวเลขแสดงผลตามรูปแบบ
ที่ออกแบบไว้ (`ACCT-BALANCE` แสดงเครื่องหมาย `+` นำหน้าเพราะเป็นค่า Signed DISPLAY ตามปกติ)

### ข้อควรระวัง

- ต้องรันคำสั่ง compile จากโฟลเดอร์เดียวกับที่เก็บไฟล์ `.cpy` เสมอ เพราะ GnuCOBOL ค้นหาไฟล์ที่ระบุใน
  `COPY "..."` (มีเครื่องหมายคำพูด) จากโฟลเดอร์ปัจจุบันโดยอัตโนมัติ (ทบทวนจาก Part 033)
- `TESTCPY` เป็นเพียงเครื่องมือช่วยตรวจสอบ ไม่ใช่ส่วนหนึ่งของระบบจริง ไม่ต้องนำไปรวมไว้กับโปรแกรมอื่นๆ

### แบบฝึกหัดที่ 909.1

**โจทย์**: จงลองแก้ `TESTCPY` ให้ตรวจสอบ 88-level `ACCT-STATUS-CLOSED` แทน `ACCT-TYPE-SAVINGS` และ
คาดเดาว่าผลลัพธ์จะเป็นอย่างไรเมื่อ `ACCT-STATUS` ถูกตั้งเป็น `"A"`

**เฉลย**: เพิ่มโค้ด `IF ACCT-STATUS-CLOSED DISPLAY "CLOSED" ELSE DISPLAY "NOT CLOSED" END-IF.` เนื่องจาก
`ACCT-STATUS` ถูก MOVE เป็น `"A"` (Active) ไว้ก่อนหน้า เงื่อนไข `ACCT-STATUS-CLOSED` (ซึ่งเทียบกับค่า
`"C"`) จะเป็นเท็จ ผลลัพธ์ที่ได้คือ `NOT CLOSED`

---

## ขั้นตอนที่ 910: สร้างและทดสอบ `CBINIT` และ `CBOPEN`

### `CBINIT` — สร้างไฟล์ Master และ Log เปล่าครั้งแรก

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CBINIT.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * CBINIT - one-time setup program for the Core Banking case
      * study. Creates a brand-new, empty ACCOUNT-MASTER indexed file
      * and a brand-new, empty TXNLOG.DAT audit trail. Run exactly
      * once before the system is used for the first time.
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-MASTER-STATUS.

           SELECT TRANSACTION-LOG ASSIGN TO "TXNLOG.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-LOG-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "cbacct.cpy".

       FD  TRANSACTION-LOG.
       01  TRANSACTION-LINE        PIC X(70).

       WORKING-STORAGE SECTION.
       01  WS-MASTER-STATUS        PIC X(2).
       01  WS-LOG-STATUS           PIC X(2).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT ACCOUNT-MASTER.
           IF WS-MASTER-STATUS = "00"
               DISPLAY "CBINIT: ACCTMAST.DAT created (status "
                   WS-MASTER-STATUS ")"
           ELSE
               DISPLAY "CBINIT: ERROR creating ACCTMAST.DAT, status="
                   WS-MASTER-STATUS
           END-IF.
           CLOSE ACCOUNT-MASTER.

           OPEN OUTPUT TRANSACTION-LOG.
           IF WS-LOG-STATUS = "00"
               DISPLAY "CBINIT: TXNLOG.DAT created (status "
                   WS-LOG-STATUS ")"
           ELSE
               DISPLAY "CBINIT: ERROR creating TXNLOG.DAT, status="
                   WS-LOG-STATUS
           END-IF.
           CLOSE TRANSACTION-LOG.

           DISPLAY "CBINIT: Core Banking system initialized.".
           STOP RUN.
```

```bash
cobc -x -o cbinit cbinit.cob
./cbinit
```

**ผลลัพธ์จริง**:

```
CBINIT: ACCTMAST.DAT created (status 00)
CBINIT: TXNLOG.DAT created (status 00)
CBINIT: Core Banking system initialized.
```

### `CBOPEN` — เปิดบัญชีใหม่ตามกฎแต่ละประเภท

โครงสร้างหลักของ `CBOPEN` (ตัดส่วนตรวจสอบอินพุตซ้ำๆ ออกเพื่อความกระชับ โค้ดฉบับเต็มมีการตรวจสอบครบทุก
เงื่อนไขตามที่ออกแบบไว้ในขั้นตอนที่ 902):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CBOPEN.
       AUTHOR. COBOL-COURSE.

      ******************************************************************
      * CBOPEN - online-style transaction: open a new account. Applies
      * type-specific business rules for the three account types this
      * case study supports: SAVINGS (interest-bearing, tiered rate
      * computed later by CBACCR), CHECKING (transaction account with
      * an overdraft limit, no interest), and FIXED-DEP (locked time
      * deposit, higher flat rate, 12-month maturity term).
      ******************************************************************

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-MASTER ASSIGN TO "ACCTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACCT-ID
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-MASTER.
       01  ACCOUNT-RECORD.
           COPY "cbacct.cpy".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC X(2).
       01  WS-TODAY                 PIC 9(8)   VALUE 0.
       01  WS-NOW                   PIC 9(6)   VALUE 0.
       01  WS-RAW-ID                PIC X(6)   VALUE SPACES.
       01  WS-RAW-AMOUNT            PIC X(9)   VALUE SPACES.
       01  WS-AMOUNT-CENTS          PIC 9(9)   VALUE 0.
       01  WS-RAW-OD                PIC X(7)   VALUE SPACES.
       01  WS-OD-CENTS              PIC 9(7)   VALUE 0.
       01  WS-VALID-FLAG            PIC X(1)   VALUE "Y".
           88  WS-ALL-VALID                 VALUE "Y".

       01  WS-RATE-SAVINGS          PIC 9(1)V9(4) VALUE 0.0125.
       01  WS-RATE-FIXED-DEP        PIC 9(1)V9(4) VALUE 0.0325.
       01  WS-MIN-SAVINGS           PIC 9(9)V99   VALUE 500.00.
       01  WS-MIN-FIXED-DEP         PIC 9(9)V99   VALUE 50000.00.

       01  WS-ZERO-GROUP            PIC 9(8)   VALUE 0.
       01  WS-TXN-TYPE-OPEN         PIC X(4)   VALUE "OPEN".
       01  WS-STATUS-OK             PIC X(8)   VALUE "OK".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "Y" TO WS-VALID-FLAG.
           OPEN I-O ACCOUNT-MASTER.
           DISPLAY "=== CBOPEN: Open New Account ===".

           DISPLAY "Enter processing date (YYYYMMDD): "
               WITH NO ADVANCING.
           ACCEPT WS-TODAY.
           DISPLAY "Enter processing time (HHMMSS): "
               WITH NO ADVANCING.
           ACCEPT WS-NOW.

           DISPLAY "Enter 6-digit account ID: " WITH NO ADVANCING.
           ACCEPT WS-RAW-ID.
           IF WS-RAW-ID IS NOT NUMERIC
               DISPLAY "ERROR: account ID must be numeric."
               MOVE "N" TO WS-VALID-FLAG
           ELSE
               MOVE WS-RAW-ID TO ACCT-ID
               READ ACCOUNT-MASTER
                   INVALID KEY
                       CONTINUE
                   NOT INVALID KEY
                       DISPLAY "ERROR: account " ACCT-ID
                           " already exists."
                       MOVE "N" TO WS-VALID-FLAG
               END-READ
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Enter account holder name (up to 24 chars): "
                   WITH NO ADVANCING
               ACCEPT ACCT-NAME
               DISPLAY "Enter type (S=Savings C=Checking F=FixedDep): "
                   WITH NO ADVANCING
               ACCEPT ACCT-TYPE
               IF NOT (ACCT-TYPE-SAVINGS OR ACCT-TYPE-CHECKING
                       OR ACCT-TYPE-FIXED-DEP)
                   DISPLAY "ERROR: type must be S, C, or F."
                   MOVE "N" TO WS-VALID-FLAG
               END-IF
           END-IF.

           IF WS-ALL-VALID
               DISPLAY "Enter opening deposit in cents (9 digits): "
                   WITH NO ADVANCING
               ACCEPT WS-RAW-AMOUNT
               IF WS-RAW-AMOUNT IS NOT NUMERIC
                   DISPLAY "ERROR: opening deposit must be numeric."
                   MOVE "N" TO WS-VALID-FLAG
               ELSE
                   MOVE WS-RAW-AMOUNT TO WS-AMOUNT-CENTS
                   COMPUTE ACCT-BALANCE = WS-AMOUNT-CENTS / 100
               END-IF
           END-IF.

           IF WS-ALL-VALID AND ACCT-TYPE-SAVINGS
                   AND ACCT-BALANCE < WS-MIN-SAVINGS
               DISPLAY "ERROR: Savings minimum opening balance is "
                   WS-MIN-SAVINGS "."
               MOVE "N" TO WS-VALID-FLAG
           END-IF.
           IF WS-ALL-VALID AND ACCT-TYPE-FIXED-DEP
                   AND ACCT-BALANCE < WS-MIN-FIXED-DEP
               DISPLAY "ERROR: Fixed Deposit minimum opening balance "
                   "is " WS-MIN-FIXED-DEP "."
               MOVE "N" TO WS-VALID-FLAG
           END-IF.

           IF WS-ALL-VALID AND ACCT-TYPE-CHECKING
               DISPLAY "Enter overdraft limit in cents (7 digits): "
                   WITH NO ADVANCING
               ACCEPT WS-RAW-OD
               IF WS-RAW-OD IS NOT NUMERIC
                   DISPLAY "ERROR: overdraft limit must be numeric."
                   MOVE "N" TO WS-VALID-FLAG
               ELSE
                   MOVE WS-RAW-OD TO WS-OD-CENTS
                   COMPUTE ACCT-OVERDRAFT-LIMIT = WS-OD-CENTS / 100
               END-IF
           END-IF.

           IF WS-ALL-VALID
               MOVE "A"           TO ACCT-STATUS
               MOVE WS-TODAY      TO ACCT-OPEN-DATE
               MOVE 0             TO ACCT-ACCRUED-INT
               MOVE 0             TO ACCT-MATURITY-DATE
               IF NOT ACCT-TYPE-CHECKING
                   MOVE 0         TO ACCT-OVERDRAFT-LIMIT
               END-IF
               MOVE 0             TO ACCT-INTEREST-RATE
      *> 0 means "never posted" - it must NOT equal the year+month of
      *> any real processing date, or CBMEND's idempotency guard
      *> (Part 093) would wrongly think this brand-new account was
      *> already posted in its very first month.
               MOVE 0             TO ACCT-LAST-STMT-DATE
               MOVE 0             TO ACCT-LAST-ACCR-DATE
               EVALUATE TRUE
                   WHEN ACCT-TYPE-SAVINGS
                       MOVE WS-RATE-SAVINGS TO ACCT-INTEREST-RATE
                   WHEN ACCT-TYPE-FIXED-DEP
                       MOVE WS-RATE-FIXED-DEP TO ACCT-INTEREST-RATE
      *> Simplified 12-month term: adding 10000 to a YYYYMMDD date
      *> increments the year digits only. Good enough for this
      *> teaching example; production code would use a proper
      *> calendar-aware date routine (see Part 047).
                       COMPUTE ACCT-MATURITY-DATE = WS-TODAY + 10000
               END-EVALUATE
               WRITE ACCOUNT-RECORD
                   INVALID KEY
                       DISPLAY "ERROR writing new account, status="
                           WS-FILE-STATUS
                       MOVE "N" TO WS-VALID-FLAG
               END-WRITE
               IF WS-ALL-VALID
                   CALL "CBAUDIT" USING WS-TODAY WS-NOW WS-ZERO-GROUP
                       ACCT-ID WS-TXN-TYPE-OPEN ACCT-BALANCE
                       ACCT-BALANCE WS-STATUS-OK
                   DISPLAY "CBOPEN: account " ACCT-ID
                       " opened successfully, type=" ACCT-TYPE
                       " balance=" ACCT-BALANCE
               END-IF
           ELSE
               DISPLAY "CBOPEN: account not opened."
           END-IF.
           CLOSE ACCOUNT-MASTER.
           STOP RUN.
```

> หมายเหตุ: โค้ดข้างต้นคือไฟล์ต้นฉบับฉบับเต็มที่คอมไพล์และรันได้จริงทุกบรรทัด ไม่มีการย่อส่วนใดๆ — มีการ
> `ACCEPT` และตรวจสอบอินพุตครบทุกฟิลด์ (วันที่, เวลา, รหัสบัญชีซ้ำ, ชื่อ, ประเภท, ยอดขั้นต่ำตามประเภท,
> วงเงินเบิกเกินบัญชีสำหรับ Checking) ตามรูปแบบเดียวกับที่ `ACCTOPEN` ใน Part 070 วางรากฐานไว้ — รับข้อมูล
> ตัวเลขที่มีทศนิยมในหน่วย **สตางค์ (cents)** เสมอ (เช่น กรอก `000150000` แทน `1500.00`) เพื่อเลี่ยงปัญหา
> `ACCEPT` รับจุดทศนิยมเข้าฟิลด์ `PIC X` แล้วทำให้ `IS NOT NUMERIC` ตรวจพบว่าไม่ใช่ตัวเลข (เพราะจุดทศนิยม
> ไม่ใช่หลักตัวเลข) — เทคนิคเดียวกับที่ `ACCTOPEN` ใน Part 070 ใช้ ก่อนบันทึกเรคคอร์ดใหม่ลงดิสก์เสมอ จะ
> `CALL "CBAUDIT"` เพื่อบันทึก log ประเภท `OPEN` ด้วยเช่นกัน (ตามรูปแบบที่ Part 092 ขั้นตอนที่ 912 อธิบาย
> การแก้บั๊กไว้)

### ทดสอบเปิดบัญชีทั้ง 3 ประเภท

```bash
cobc -x cbopen.cob cbaudit.cob -o cbopen
```

> หมายเหตุสำคัญเรื่องการ CALL: `CBOPEN` เรียก subprogram `CBAUDIT` (จะสร้างจริงใน Part 092 ขั้นตอนที่
> 912) ด้วยคำสั่ง `CALL "CBAUDIT" USING ...` — ในขั้นตอนนี้เราคอมไพล์ทั้งสองไฟล์ `.cob` เข้าด้วยกันเป็น
> executable เดียว (Static Call ตามแนวทาง Part 031) ซึ่งเป็นวิธีที่ตรงไปตรงมาและไม่ต้องพึ่งพา environment
> variable ใดๆ ในขณะรัน ต่างจากการทำ Dynamic Call ผ่านไฟล์ `.so` แยกต่างหาก

```bash
printf '20260401\n090000\n100001\nJANE DOE\nS\n000150000\n' | ./cbopen
printf '20260401\n091500\n200001\nACME TRADING CO\nC\n000000000\n0005000\n' | ./cbopen
printf '20260401\n093000\n300001\nSOMCHAI SAVER\nF\n005000000\n' | ./cbopen
```

**ผลลัพธ์จริง**:

```
=== CBOPEN: Open New Account ===
Enter processing date (YYYYMMDD): Enter processing time (HHMMSS): Enter 6-digit account ID: Enter account holder name (up to 24 chars): Enter type (S=Savings C=Checking F=FixedDep): Enter opening deposit in cents (9 digits): CBOPEN: account 100001 opened successfully, type=S balance=+000001500.00
=== CBOPEN: Open New Account ===
Enter processing date (YYYYMMDD): Enter processing time (HHMMSS): Enter 6-digit account ID: Enter account holder name (up to 24 chars): Enter type (S=Savings C=Checking F=FixedDep): Enter opening deposit in cents (9 digits): Enter overdraft limit in cents (7 digits): CBOPEN: account 200001 opened successfully, type=C balance=+000000000.00
=== CBOPEN: Open New Account ===
Enter processing date (YYYYMMDD): Enter processing time (HHMMSS): Enter 6-digit account ID: Enter account holder name (up to 24 chars): Enter type (S=Savings C=Checking F=FixedDep): Enter opening deposit in cents (9 digits): CBOPEN: account 300001 opened successfully, type=F balance=+000050000.00
```

และเมื่อตรวจสอบไฟล์ Audit Trail:

```bash
cat TXNLOG.DAT
```

```
20260401 090000 00000000 100001 OPEN 00000150000 +00000150000 OK
20260401 091500 00000000 200001 OPEN 00000000000 +00000000000 OK
20260401 093000 00000000 300001 OPEN 00005000000 +00005000000 OK
```

ทั้งสามบัญชีถูกสร้างสำเร็จ ครบทั้ง 3 ประเภท และทุกการเปิดบัญชีถูกบันทึกลง Audit Trail ทันที — **นี่คือ
รากฐานที่ Part 092 จะนำไปต่อยอดสร้างโมดูลฝาก/ถอน/โอนเงิน**

### ข้อควรระวัง

- `CBINIT` เขียนทับไฟล์เดิมทุกครั้งที่รัน (`OPEN OUTPUT`) — รันได้เพียงครั้งเดียวก่อนเริ่มใช้งานระบบจริง
  เท่านั้น เช่นเดียวกับที่ Part 070 เตือนไว้
- ต้องคอมไพล์ `cbopen.cob` พร้อมกับ `cbaudit.cob` เสมอ (รวมถึงทุกโปรแกรมอื่นที่ CALL `CBAUDIT` ในภายหลัง)
  ไม่เช่นนั้นจะได้ error runtime `module 'CBAUDIT' not found`
- ทุกโปรแกรมต้องรันในโฟลเดอร์เดียวกับ `ACCTMAST.DAT`, `TXNLOG.DAT` และไฟล์ `.cpy` เสมอ เพราะอ้างอิงชื่อ
  ไฟล์แบบ relative path ทั้งหมด

### แบบฝึกหัดที่ 910.1

**โจทย์**: หากลองเปิดบัญชีเลขที่ `100001` ซ้ำอีกครั้ง (เลขบัญชีเดียวกับที่เปิดไปแล้ว) คาดว่าโปรแกรมจะ
ตอบสนองอย่างไร และเพราะกลไกใดในโค้ด

**เฉลย**: โปรแกรมจะแสดง `ERROR: account 100001 already exists.` และไม่เปิดบัญชีซ้ำ เพราะก่อนเขียนบัญชี
ใหม่ โปรแกรมทำ `READ ACCOUNT-MASTER` ด้วยรหัสบัญชีที่ผู้ใช้กรอกก่อนเสมอ (ทบทวนเทคนิคจาก Part 028): ถ้า
`READ` สำเร็จ (`NOT INVALID KEY`) แปลว่ามีบัญชีนี้อยู่แล้วในระบบ โปรแกรมจึงตั้ง `WS-VALID-FLAG` เป็น `"N"`
และหยุดกระบวนการเปิดบัญชีทันทีโดยไม่มีการ `WRITE` ใดๆ เกิดขึ้น

---

## สรุปท้ายบท

Part นี้วางรากฐานทั้งหมดของ Case Study Core Banking ไว้ครบถ้วน:

- กำหนดขอบเขตและข้อกำหนดทางธุรกิจที่ทำให้ระบบนี้สมจริงและกว้างกว่า Part 070 อย่างชัดเจน (3 ประเภทบัญชี,
  การโอนเงิน, ดอกเบี้ยขั้นบันได, การกระทบยอดแบบหลักบัญชีคู่, Idempotency)
- ออกแบบโครงสร้างข้อมูลหลักสองไฟล์: `ACCOUNT-MASTER` (Indexed) และ `TRANSACTION-LOG` (Sequential พร้อม
  `TXN-GROUP-ID` สำหรับเชื่อมขาธุรกรรมโอนเงิน)
- สร้าง Copybook สองไฟล์ (`cbacct.cpy`, `cbtxn.cpy`) และพิสูจน์ด้วยการคอมไพล์และรันจริงผ่าน `TESTCPY`
- วางแผนสถาปัตยกรรมทั้ง 12 โปรแกรม พร้อมมาตรฐานการตั้งชื่อและกลยุทธ์จัดการข้อผิดพลาดที่จะใช้ตลอดทั้ง
  3 Part
- สร้างและทดสอบ `CBINIT` และ `CBOPEN` จนได้บัญชีทั้ง 3 ประเภทพร้อมใช้งานจริง พร้อม Audit Trail เริ่มต้น

ราก ฐานทุกอย่างพร้อมแล้ว **Part 092** จะนำบัญชีทั้ง 3 บัญชีนี้ไปต่อยอดสร้างโมดูลฝากเงิน ถอนเงิน และที่
สำคัญที่สุดคือ**การโอนเงินระหว่างบัญชีพร้อมกลไก Compensating Transaction** ซึ่งเป็นหัวใจสำคัญที่สุดของ
Case Study นี้

**[กลับไป Part 090: การจัดการโปรเจกต์ COBOL ขนาดใหญ่](part-090-project-management.md)**
**[ไปยัง Part 092: Case Study Core Banking ตอนที่ 2 - โมดูลบัญชีและธุรกรรม →](part-092-corebanking-transactions.md)**
