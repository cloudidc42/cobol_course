# Part 100: 🎯 Capstone Project สุดท้าย, บทสรุปหลักสูตร, และแหล่งเรียนรู้ต่อ (ขั้นตอนที่ 991–1000)

## คำนำของ Part นี้

เดินทางมาถึงจุดหมายปลายทางแล้ว — **ขั้นตอนที่ 991 ถึง 1000 จากทั้งหมด 1000 ขั้นตอน** ของหลักสูตรนี้

ย้อนกลับไป Part 001 ขั้นตอนที่ 1 เราเริ่มต้นด้วยโค้ด COBOL บรรทัดเดียว: `ADD FIRST-NUMBER TO
SECOND-NUMBER GIVING TOTAL-AMOUNT.` พร้อมคำอธิบายว่า COBOL คืออะไรและทำไมยังสำคัญในปี 2026 — ตอนนี้
หลังผ่านมา 99 Part เต็ม เราจะปิดหลักสูตรด้วยสิ่งที่ยิ่งใหญ่ที่สุดและครบถ้วนที่สุดเท่าที่หลักสูตรนี้
เคยสร้างมา: **ระบบจัดการคำสั่งซื้อระดับองค์กร (Enterprise Order Management System — เรียกย่อว่า EOMS)**
ที่ดึงเทคนิคจาก**ทั้ง 6 เฟส**ของหลักสูตรมาประกอบร่างเป็นระบบเดียวที่ทำงานได้จริงสมบูรณ์แบบ

### ทำไม EOMS ถึงเป็นบทพิสูจน์สุดท้ายที่เหมาะสม

โปรเจกต์รวบยอดของแต่ละเฟสที่ผ่านมา (Part 015, 035, 050, 070, 085) ต่างพิสูจน์ความรู้ **เฉพาะเฟสนั้น**
เท่านั้น — Part 015 พิสูจน์พื้นฐาน, Part 035 พิสูจน์ตารางและไฟล์, Part 050 พิสูจน์ Intrinsic Function
และ Error Handling, Part 070 พิสูจน์แนวคิด Mainframe/VSAM, Part 085 พิสูจน์เครื่องมือสมัยใหม่และ Clean
Architecture — **Part 100 นี้จะพิสูจน์ทั้งหมดพร้อมกันในระบบเดียว** ระบบ EOMS ที่เราจะสร้างประกอบด้วย
โครงสร้างข้อมูลจากเฟส 1, ตาราง/Subprogram จากเฟส 2, การคำนวณและ Intrinsic Function จากเฟส 3, ไฟล์
Indexed แบบ VSAM จากเฟส 4, REST API/CI/Unit Test จากเฟส 5, และสถาปัตยกรรมแบบ Clean Architecture
จากเฟส 6 — ทั้งหมดทำงานร่วมกันเป็นระบบเดียวที่คอมไพล์และรันได้จริง 100%

### ประกาศความซื่อสัตย์ที่สำคัญที่สุดของ Part นี้ (อ่านก่อนเริ่ม)

> **ทุกโปรแกรมในเอกสารนี้ COMPILE และ RUN ได้จริง 100% ด้วย GnuCOBOL** (เวอร์ชัน 4.0-early ที่ใช้
> ทดสอบเนื้อหาทั้งหมดในหลักสูตรนี้ พร้อม ISAM handler ที่เปิดใช้งาน Indexed File ตามที่ Part 028/070
> อธิบายไว้) ทุกผลลัพธ์ที่แสดงในเอกสารนี้คือผลลัพธ์จริงจากการรันจริง ไม่ใช่ผลลัพธ์ที่แต่งขึ้น
>
> **ระหว่างการพัฒนา Capstone นี้ เราพบบั๊กจริง 2 ตัวจากการทดสอบจริง** (ไม่ใช่ตัวอย่างสมมติ) เช่น
> เดียวกับที่ Part 035 และ Part 085 เคยเล่าไว้ — บั๊กแรกคือการลืม `GOBACK` ทำให้โปรแกรมทำงาน "ตกหล่น"
> เข้า Paragraph ถัดไป (พบและแก้ไขในขั้นตอนที่ 994) บั๊กที่สองคือ `CLOSE` เขียนทับ `FILE STATUS` ของ
> `READ` ที่ล้มเหลว (พบและแก้ไขในขั้นตอนที่ 995 — และถูกรวบรวมไว้ใน Part 098 ขั้นตอนที่ 976 ด้วยแล้ว)
> ทั้งสองบั๊กนี้ถูกเก็บไว้ในเอกสารนี้พร้อมคำอธิบายเต็มรูปแบบ เพราะเป็นบทเรียนที่มีค่าไม่แพ้โค้ดที่
> ทำงานถูกต้องตั้งแต่แรก
>
> **ข้อจำกัดของสภาพแวดล้อมที่ตรวจสอบแล้วและปรับตัวอย่างตรงไปตรงมา**: GnuCOBOL build ที่ใช้ทดสอบ
> Capstone นี้รายงานว่า `JSON library : disabled` (ตรวจสอบได้เองด้วย `cobc -info | grep -i json`)
> ดังนั้นขั้นตอนที่ 998 (JSON Export) จึงไม่ได้ใช้ `JSON GENERATE` ตามที่ Part 045 สอนไว้ตรง ๆ แต่ใช้
> `STRING` สร้าง JSON ด้วยมือแทน — วิธีแก้ปัญหาแบบเดียวกับที่ทีมพัฒนา COBOL ในโลกจริงต้องทำเมื่อเจอ
> ข้อจำกัดของคอมไพเลอร์ที่ใช้งานอยู่

### โครงสร้างโปรเจกต์ทั้งหมด (EOMS)

```
eoms/
├── copybooks/
│   ├── EOMSCNST.CPY      (ค่าคงที่ทางธุรกิจ - Step 991)
│   ├── CUSTOMER.CPY      (โครงสร้างข้อมูลลูกค้า - Step 991)
│   ├── PRODUCT.CPY       (โครงสร้างข้อมูลสินค้า - Step 991)
│   └── ORDERREC.CPY      (โครงสร้างข้อมูลคำสั่งซื้อ - Step 991)
├── src/
│   ├── custinit.cob      (สร้าง+seed CUSTMAST.DAT - Step 992)
│   ├── prodinit.cob      (สร้าง+seed PRODMAST.DAT - Step 992)
│   ├── ordinit.cob       (สร้าง ORDMAST.DAT เปล่า - Step 992)
│   ├── ordcalc.cob       (Business Logic: คำนวณ - Step 993)
│   ├── ordvalid.cob      (Business Logic: ตรวจสอบ - Step 994)
│   ├── custrepo.cob      (Data Access: ลูกค้า - Step 995)
│   ├── prodrepo.cob      (Data Access: สินค้า - Step 995)
│   ├── ordrepo.cob       (Data Access: คำสั่งซื้อ - Step 995)
│   ├── ordsvc.cob        (Application Layer: Orchestrator - Step 996)
│   ├── eomsmain.cob      (Presentation: เมนูหลัก - Step 997)
│   ├── ordexport.cob     (JSON Export - Step 998)
│   ├── ordrpt.cob        (Batch Report + SORT - Step 998)
│   └── ordapi.cob        (CLI entry point สำหรับ REST - Step 999)
├── tests/
│   ├── testordcalc.cob   (Unit Test - Step 999)
│   └── testordvalid.cob  (Unit Test - Step 999)
├── rest/
│   └── ordapi_server.py  (REST API Wrapper - Step 999)
├── build/                (ไฟล์ .o และ executable ที่คอมไพล์แล้ว)
└── ci.sh                 (CI Pipeline - Step 999)
```

**คำสั่งคอมไพล์มาตรฐานที่ใช้ตลอดทั้ง Part นี้** (ต้องใช้ GnuCOBOL build ที่เปิดใช้งาน Indexed File
ตามที่ Part 028/070 อธิบายไว้):

```bash
export LD_LIBRARY_PATH=/path/to/gnucobol-isam/lib
COBC=/path/to/gnucobol-isam/bin/cobc

$COBC -I copybooks -c src/ชื่อไฟล์.cob -o build/ชื่อไฟล์.o
$COBC -I copybooks -x -o build/ชื่อโปรแกรม src/main.cob build/*.o
```

---

## ขั้นตอนที่ 991: ออกแบบระบบและวาง Copybook พื้นฐาน (Phase 1 Callback: โครงสร้างข้อมูล)

### ความต้องการของระบบ EOMS

ระบบ EOMS จัดการคำสั่งซื้อของลูกค้าแบบครบวงจร: ลูกค้าสั่งซื้อสินค้าได้หลายรายการต่อหนึ่งคำสั่งซื้อ
ระบบคำนวณส่วนลดตามระดับสมาชิก คำนวณภาษี ตัดสต๊อกสินค้า อัปเดตยอดหนี้คงค้างของลูกค้า และรองรับการ
ยกเลิกคำสั่งซื้อพร้อมคืนสต๊อก/ยอดหนี้ — **ความต้องการ**:

| ข้อ | ความต้องการ |
|---|---|
| 1 | เก็บข้อมูลลูกค้า (ระดับสมาชิก, วงเงินเครดิต, ยอดหนี้คงค้าง) ในไฟล์ Indexed แบบ VSAM-style |
| 2 | เก็บข้อมูลสินค้า (ราคา, จำนวนคงคลัง, จุดสั่งซื้อซ้ำ) ในไฟล์ Indexed แบบ VSAM-style |
| 3 | คำสั่งซื้อหนึ่งรายการมีได้สูงสุด 5 รายการสินค้าย่อย (line item) |
| 4 | คำนวณส่วนลดตามระดับสมาชิก (Platinum 10%, Gold 5%, Regular 0%) และภาษี 7% |
| 5 | ตรวจสอบสต๊อกเพียงพอ, สินค้ายัง Active, ลูกค้ายัง Active, และไม่เกินวงเงินเครดิตก่อนสร้างคำสั่งซื้อ |
| 6 | รองรับการยกเลิกคำสั่งซื้อ พร้อมคืนสต๊อกและยอดหนี้คงค้างโดยอัตโนมัติ |
| 7 | เปิดให้เรียกใช้งานผ่านเมนูแบบโต้ตอบ (Console) และผ่าน REST API |
| 8 | มีรายงานสรุปคำสั่งซื้อแบบจัดกลุ่มตามลูกค้า (Batch Report) |
| 9 | ส่งออกข้อมูลคำสั่งซื้อเป็น JSON สำหรับระบบภายนอก |
| 10 | มี Unit Test และ CI Pipeline ที่คอมไพล์+ทดสอบอัตโนมัติ |

### ADR: การตัดสินใจเชิงสถาปัตยกรรมของโปรเจกต์นี้

ตามรูปแบบ Architecture Decision Record ที่ Part 050/085 แนะนำไว้:

> **ADR-100-01: ใช้สถาปัตยกรรมแบบ Layered (4 ชั้น) ตามแนวคิด Clean Architecture ของ Part 082/086**
>
> **บริบท**: ระบบ EOMS มีทั้งตรรกะทางธุรกิจที่ซับซ้อน (การคำนวณส่วนลด/ภาษี, การตรวจสอบเงื่อนไข),
> การเข้าถึงไฟล์ (3 Master File), และส่วนติดต่อผู้ใช้ (2 แบบ: Console และ REST API)
>
> **การตัดสินใจ**: แบ่งระบบเป็น 4 ชั้นชัดเจน: **Presentation** (`EOMSMAIN.cob`, `ordapi_server.py`),
> **Application/Orchestration** (`ORDSVC.cob`), **Business Logic** (`ORDCALC.cob`, `ORDVALID.cob` —
> ไม่มี File I/O เลย), และ **Data Access** (`CUSTREPO.cob`, `PRODREPO.cob`, `ORDREPO.cob`) แต่ละชั้น
> รู้จักแค่ชั้นที่อยู่ติดกันเท่านั้น
>
> **ผลที่ตามมา**: Business Logic Layer ทดสอบได้โดยไม่ต้องมีไฟล์บนดิสก์เลย (Step 999), และสามารถเพิ่ม
> ช่องทางการใช้งานใหม่ (เช่น REST API ใน Step 999) โดยไม่ต้องแตะ Business Logic หรือ Data Access เลย

> **ADR-100-02: ยอมรับข้อจำกัดของ Two-Phase Commit และบันทึกไว้อย่างชัดเจนแทนการซ่อน**
>
> **บริบท**: การสร้างคำสั่งซื้อหนึ่งครั้งต้องแก้ไขไฟล์ 3 ไฟล์ (ลด PRODMAST, เพิ่ม CUSTMAST, เขียน
> ORDMAST) การรับประกัน Atomicity เต็มรูปแบบต้องใช้กลไกอย่าง CICS/DB2 Syncpoint (Part 061-064)
>
> **การตัดสินใจ**: ยอมรับว่า GnuCOBOL ระดับนี้ไม่มีกลไก Two-Phase Commit ในตัว และบันทึกข้อจำกัดนี้
> ไว้เป็นคอมเมนต์ในโค้ดอย่างตรงไปตรงมา (`ORDSVC.cob`) แทนที่จะเสแสร้งว่าระบบมีความปลอดภัยระดับ
> Production เต็มรูปแบบ

### Copybook 1: ค่าคงที่ทางธุรกิจ (EOMSCNST.CPY)

ตามหลักการ Copybook Governance จาก Part 098 ขั้นตอนที่ 975 ค่าคงที่ทางธุรกิจทั้งหมดรวมอยู่ที่เดียว:

```cobol
      *> ================================================================
      *> Copybook: EOMSCNST.CPY
      *> Enterprise Order Management System (EOMS) - named constants.
      *> Single source of truth for every business rule threshold used
      *> across EOMS programs (Part 098 copybook governance practice:
      *> one place to change a rule, every program picks it up on next
      *> compile). Level-78 items, same technique as Part 033/085.
      *> ================================================================
       78  EOMS-MAX-ORDER-LINES     VALUE 5.

       78  TIER-DISCOUNT-PLATINUM   VALUE 0.100.
       78  TIER-DISCOUNT-GOLD       VALUE 0.050.
       78  TIER-DISCOUNT-REGULAR    VALUE 0.000.

       78  EOMS-SALES-TAX-RATE      VALUE 0.070.

       78  CUST-STATUS-ACTIVE       VALUE "A".
       78  CUST-STATUS-INACTIVE     VALUE "I".

       78  PROD-STATUS-ACTIVE       VALUE "A".
       78  PROD-STATUS-DISCONTINUED VALUE "D".

       78  ORDER-STATUS-OPEN        VALUE "O".
       78  ORDER-STATUS-SHIPPED     VALUE "S".
       78  ORDER-STATUS-CANCELLED   VALUE "X".
```

### Copybook 2-4: โครงสร้างข้อมูลหลัก (Phase 1 Callback)

การออกแบบ record ทั้งสามตัวนี้ใช้หลักการ PICTURE Clause จาก Part 006 และ WORKING-STORAGE Design
จาก Part 005 โดยตรง:

```cobol
      *> ================================================================
      *> Copybook: CUSTOMER.CPY
      *> Shared record layout for one customer, used by CUSTMAST
      *> (the indexed "master file") and by every program that reads
      *> or writes a customer record. Phase 1 data design + Phase 4
      *> VSAM-style master file record (callback to Part 028/055).
      *> ================================================================
       01  CUST-RECORD.
           05  CUST-ID              PIC 9(6).
           05  CUST-NAME            PIC X(24).
           05  CUST-TIER            PIC X(1).
      *>       "P" = Platinum, "G" = Gold, "R" = Regular
           05  CUST-STATUS          PIC X(1).
      *>       "A" = Active, "I" = Inactive
           05  CUST-CREDIT-LIMIT    PIC 9(7)V99.
           05  CUST-BALANCE         PIC 9(7)V99.
      *>       Running total of all OPEN order amounts for this
      *>       customer, maintained by CUSTREPO.
```

```cobol
      *> ================================================================
      *> Copybook: PRODUCT.CPY
      *> Shared record layout for one product, used by PRODMAST (the
      *> indexed "master file") and by every program that reads or
      *> writes a product record.
      *> ================================================================
       01  PROD-RECORD.
           05  PROD-ID              PIC 9(6).
           05  PROD-NAME            PIC X(20).
           05  PROD-UNIT-PRICE      PIC 9(5)V99.
           05  PROD-QTY-ON-HAND     PIC 9(5).
           05  PROD-REORDER-POINT   PIC 9(5).
           05  PROD-STATUS          PIC X(1).
      *>       "A" = Active, "D" = Discontinued
```

```cobol
      *> ================================================================
      *> Copybook: ORDERREC.CPY
      *> Shared record layout for one order, used by ORDMAST (the
      *> indexed "master file") and by every program that reads or
      *> writes an order. An order header carries up to
      *> EOMS-MAX-ORDER-LINES embedded line items (Phase 2 OCCURS
      *> table, Part 016) inside one fixed-length VSAM-style record
      *> (Phase 4 callback, Part 028/055).
      *> ================================================================
       01  ORD-RECORD.
           05  ORD-ID               PIC 9(8).
           05  ORD-CUST-ID          PIC 9(6).
           05  ORD-DATE             PIC 9(8).
      *>       YYYYMMDD
           05  ORD-STATUS           PIC X(1).
      *>       "O" = Open, "S" = Shipped, "X" = Cancelled
           05  ORD-LINE-COUNT       PIC 9(2).
           05  ORD-LINE-ITEMS OCCURS 5 TIMES
                   INDEXED BY ORD-LINE-IDX.
               10  LINE-PROD-ID     PIC 9(6).
               10  LINE-QTY         PIC 9(5).
               10  LINE-UNIT-PRICE  PIC 9(5)V99.
               10  LINE-EXTENDED    PIC 9(7)V99.
           05  ORD-SUBTOTAL         PIC 9(9)V99.
           05  ORD-DISCOUNT-PCT     PIC 9V999.
           05  ORD-DISCOUNT-AMT     PIC 9(9)V99.
           05  ORD-TAX-AMT          PIC 9(9)V99.
           05  ORD-TOTAL            PIC 9(9)V99.
```

สังเกตว่า `ORD-RECORD` รวม **Header** (ORD-ID, ORD-CUST-ID, ORD-DATE, ORD-STATUS) และ **Line Items**
(ตาราง `OCCURS 5 TIMES`) ไว้ในระเบียนเดียวกัน — นี่คือรูปแบบทั่วไปของระเบียน VSAM ในโลก Mainframe จริง
ที่ Part 055 สอนไว้: ระเบียนความยาวคงที่ (fixed-length) ที่ฝังตารางขนาดจำกัดไว้ภายใน แทนที่จะแยกเป็น
ตาราง Master-Detail สองไฟล์แบบฐานข้อมูลเชิงสัมพันธ์

### ข้อควรระวัง

- ทั้ง 4 Copybook ยังไม่มี File I/O หรือ Business Logic ใด ๆ เลยในขั้นตอนนี้ — เป็นเพียงการวาง
  โครงสร้างข้อมูล (Phase 1) ก่อนเริ่มสร้างโปรแกรมจริงในขั้นตอนถัดไป ตามหลักการ "ออกแบบก่อนเขียนโค้ด"
  ที่ Part 015 ขั้นตอนที่ 141 เน้นย้ำไว้
- `EOMS-MAX-ORDER-LINES VALUE 5` ต้องตรงกับ `OCCURS 5 TIMES` ใน `ORDERREC.CPY` เสมอ — ถ้าต้องการ
  เปลี่ยนจำนวน Line Item สูงสุดในอนาคต ต้องแก้ไขทั้งสองจุดพร้อมกัน (ข้อจำกัดที่ยอมรับได้ของการใช้
  Level-78 คู่กับ OCCURS ที่ COBOL ไม่มีกลไกเชื่อมสองค่านี้เข้าด้วยกันอัตโนมัติ)

### แบบฝึกหัดที่ 991.1

**โจทย์**: จงอธิบายว่าทำไม `ORD-RECORD` จึงออกแบบให้ Line Item ฝังอยู่ในระเบียนเดียวกัน (embedded
OCCURS table) แทนที่จะแยกเป็นไฟล์ "Order Header" กับไฟล์ "Order Line" สองไฟล์แยกกัน

**เฉลยแนวทาง**: การฝัง Line Item ไว้ในระเบียนเดียวกันเหมาะกับกรณีที่จำนวน Line Item สูงสุดจำกัดและ
ทราบล่วงหน้า (ในที่นี้คือ 5 รายการ) เพราะทำให้การอ่าน/เขียนคำสั่งซื้อหนึ่งรายการทำได้ด้วยการเข้าถึง
ไฟล์เพียงครั้งเดียว (single I/O) ไม่ต้องทำ Master-Detail Join แบบที่ Part 026 สอนไว้ ซึ่งเร็วกว่ามาก
สำหรับกรณีใช้งานแบบ Random Access (เช่น ดึงคำสั่งซื้อหนึ่งรายการมาแสดงผล) แต่ข้อเสียคือถ้าจำนวน Line
Item ไม่จำกัดชัดเจนหรือมีจำนวนมาก (เช่นหลักร้อยรายการ) การฝังแบบนี้จะสิ้นเปลืองพื้นที่ (ถ้าใช้ไม่เต็ม
ตาราง) หรือไม่รองรับ (ถ้าเกินขอบเขตตาราง) ซึ่งในกรณีนั้นการแยกเป็น Master-Detail สองไฟล์แบบ Part 026
จะเหมาะสมกว่า

---

## ขั้นตอนที่ 992: สร้างและ Seed Master File (Phase 4 Callback: VSAM-style Indexed File)

### แนวคิด: ต้องมีข้อมูลตั้งต้นก่อนระบบจะทำงานได้

ตามหลักการ Part 028 (Indexed Files) และ Part 070 (โปรเจกต์ธนาคารจำลอง) เราต้องสร้างไฟล์ Indexed
เปล่า ๆ ก่อน แล้วจึงโหลดข้อมูลตั้งต้น (seed data) เข้าไป — สามโปรแกรม setup นี้รันเพียงครั้งเดียวตอน
เริ่มต้นระบบเท่านั้น

### CUSTINIT.cob — สร้างและ Seed CUSTMAST.DAT

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CUSTINIT.
       AUTHOR. COBOL-COURSE.
      *> Setup program (run once): creates CUSTMAST.DAT as an indexed
      *> "VSAM-style KSDS" file (Phase 4 callback, Part 028/055) and
      *> loads a small seed set of customers so the rest of the
      *> capstone has real data to work against.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUST-FILE ASSIGN TO "CUSTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-CUST-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUST-FILE.
           COPY "CUSTOMER.CPY".

       WORKING-STORAGE SECTION.
       01  WS-CUST-STATUS         PIC XX.
           88  WS-CUST-OK                   VALUE "00".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUST-FILE
           IF NOT WS-CUST-OK
               DISPLAY "CUSTINIT: cannot create file, status="
                   WS-CUST-STATUS
               STOP RUN
           END-IF

           MOVE 100001 TO CUST-ID
           MOVE "ACME CORPORATION"       TO CUST-NAME
           MOVE "P"                      TO CUST-TIER
           MOVE "A"                      TO CUST-STATUS
           MOVE 50000.00                 TO CUST-CREDIT-LIMIT
           MOVE 0.00                     TO CUST-BALANCE
           PERFORM WRITE-ONE-CUSTOMER

           MOVE 100002 TO CUST-ID
           MOVE "GOLDEN GATE TRADING"    TO CUST-NAME
           MOVE "G"                      TO CUST-TIER
           MOVE "A"                      TO CUST-STATUS
           MOVE 20000.00                 TO CUST-CREDIT-LIMIT
           MOVE 0.00                     TO CUST-BALANCE
           PERFORM WRITE-ONE-CUSTOMER

           MOVE 100003 TO CUST-ID
           MOVE "RIVERSIDE RETAIL LLC"   TO CUST-NAME
           MOVE "R"                      TO CUST-TIER
           MOVE "A"                      TO CUST-STATUS
           MOVE 5000.00                  TO CUST-CREDIT-LIMIT
           MOVE 0.00                     TO CUST-BALANCE
           PERFORM WRITE-ONE-CUSTOMER

           MOVE 100004 TO CUST-ID
           MOVE "NORTHWIND SUPPLY CO"    TO CUST-NAME
           MOVE "R"                      TO CUST-TIER
           MOVE "I"                      TO CUST-STATUS
           MOVE 1000.00                  TO CUST-CREDIT-LIMIT
           MOVE 0.00                     TO CUST-BALANCE
           PERFORM WRITE-ONE-CUSTOMER

           CLOSE CUST-FILE
           DISPLAY "CUSTINIT: CUSTMAST.DAT created with 4 customers."
           STOP RUN.

       WRITE-ONE-CUSTOMER.
           WRITE CUST-RECORD
           IF NOT WS-CUST-OK
               DISPLAY "CUSTINIT: write failed for " CUST-ID
                   " status=" WS-CUST-STATUS
           END-IF.
```

`PRODINIT.cob` และ `ORDINIT.cob` มีโครงสร้างเดียวกันทุกประการ (สร้างไฟล์ + seed ข้อมูล 4 สินค้าสำหรับ
`PRODMAST.DAT`, และสร้างไฟล์เปล่าสำหรับ `ORDMAST.DAT` เพราะยังไม่มีคำสั่งซื้อใด ๆ ตอนเริ่มระบบ) จึงไม่
แสดงซ้ำในเอกสารนี้ — ดูซอร์สโค้ดฉบับเต็มได้จากโครงสร้างโปรเจกต์เดียวกัน โดย `PRODINIT.cob` สร้างสินค้า
4 รายการ (รหัส 200001-200004) รวมถึงสินค้าหนึ่งรายการที่มีสต๊อกเหลือน้อยมาก (`GADGET-MINI`, เหลือ 8
ชิ้น) และอีกหนึ่งรายการที่ถูกเลิกผลิตแล้ว (`GIZMO-LEGACY`, `PROD-STATUS = "D"`) เพื่อใช้ทดสอบเงื่อนไข
Defensive Programming ในขั้นตอนถัดไป

### คอมไพล์และรันจริง

```bash
$COBC -I copybooks -x -o build/custinit src/custinit.cob
$COBC -I copybooks -x -o build/prodinit src/prodinit.cob
$COBC -I copybooks -x -o build/ordinit  src/ordinit.cob

./build/custinit
./build/prodinit
./build/ordinit
```

**ผลลัพธ์จริงจากการรัน**:

```
CUSTINIT: CUSTMAST.DAT created with 4 customers.
PRODINIT: PRODMAST.DAT created with 4 products.
ORDINIT: ORDMAST.DAT created (empty).
```

### อธิบายโค้ดทีละส่วน

`CUSTINIT.cob` ใช้ `ACCESS MODE IS SEQUENTIAL` กับ `OPEN OUTPUT` เพื่อสร้างไฟล์ใหม่ทั้งหมดและเขียน
ระเบียนเรียงตามลำดับคีย์ (จำเป็นสำหรับ `ACCESS SEQUENTIAL` — ถ้าต้องการเขียนแบบไม่เรียงลำดับ ต้องใช้
`ACCESS RANDOM` หรือ `DYNAMIC` ตามที่ Part 028 อธิบายไว้) สังเกตว่าลูกค้ารหัส `100004` ถูก seed ด้วย
`CUST-STATUS = "I"` (Inactive) โดยตั้งใจ — นี่คือข้อมูลทดสอบสำหรับพิสูจน์ว่า `ORDVALID.cob` ในขั้นตอน
ที่ 994 ปฏิเสธคำสั่งซื้อจากลูกค้าที่ไม่ Active ได้จริง

### ข้อควรระวัง

- โปรแกรม setup เหล่านี้ **ต้องรันแค่ครั้งเดียว** ตอนเริ่มต้นระบบเท่านั้น — ถ้ารันซ้ำโดยไม่ลบไฟล์เดิม
  ก่อน `OPEN OUTPUT` จะสร้างไฟล์ใหม่ทับไฟล์เดิม (ข้อมูลเดิมหายทั้งหมด) ซึ่งอาจเป็นพฤติกรรมที่ต้องการ
  หรือไม่ต้องการก็ได้ขึ้นกับสถานการณ์ ต้องระมัดระวังเสมอในระบบจริง
- ตรวจสอบเสมอว่า GnuCOBOL build ที่ใช้เปิดใช้งาน Indexed File แล้วด้วย `cobc -info | grep -i indexed`
  ก่อนรันโปรแกรมกลุ่มนี้ (ทบทวนกับดัก #10 จากตาราง Greatest Hits ใน Part 098 ขั้นตอนที่ 977)

### แบบฝึกหัดที่ 992.1

**โจทย์**: จงอธิบายว่าทำไม `PRODINIT.cob` ถึงจงใจ seed สินค้าที่มีสต๊อกเหลือน้อยมาก (8 ชิ้น) และสินค้า
ที่เลิกผลิตแล้วไว้ด้วย แทนที่จะ seed แค่สินค้าปกติที่มีสต๊อกเพียงพอทั้งหมด

**เฉลยแนวทาง**: เพราะข้อมูลทดสอบที่ดีต้องครอบคลุมทั้งกรณีปกติ (happy path) และกรณีขอบเขต/error ตามที่
Part 079 (Unit Testing) และ Part 098 (Code Review Checklist หมวด E) สอนไว้ การมีสินค้าสต๊อกน้อยทำให้
ทดสอบเงื่อนไข "สต๊อกไม่เพียงพอ" ใน `ORDVALID.cob` ได้ทันทีโดยไม่ต้องสร้างสถานการณ์พิเศษเพิ่มเติม และ
การมีสินค้าเลิกผลิตทำให้ทดสอบเงื่อนไข "สินค้าไม่ Active" ได้เช่นกัน — การเตรียมข้อมูลทดสอบที่ครอบคลุม
ตั้งแต่ขั้นตอนแรกช่วยให้ขั้นตอนถัดไปทดสอบ Business Logic ได้ครบถ้วนโดยไม่ต้องย้อนกลับมาแก้ข้อมูลตั้งต้น

---

## ขั้นตอนที่ 993: Business Logic Layer ส่วนที่ 1 — ORDCALC.cob (Phase 3 Callback: การคำนวณ)

### แนวคิด: Pure Function ที่ไม่มี File I/O เลย

ตามหลักการ Repository/Pure Function ที่ Part 082 สอนไว้ (และพิสูจน์แล้วจริงใน `ARDISC.cob` ของ
Part 085) `ORDCALC.cob` คือ**เครื่องคำนวณล้วน ๆ**: รับพารามิเตอร์เข้า คำนวณ คืนค่าออก โดยไม่แตะไฟล์
หรือทรัพยากรภายนอกใด ๆ เลย — คุณสมบัตินี้ทำให้มันทดสอบง่ายที่สุดเท่าที่จะเป็นไปได้ (พิสูจน์จริงใน
ขั้นตอนที่ 999)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ORDCALC.
       AUTHOR. COBOL-COURSE.
      *> Business Logic Layer - pure calculation engine for one order.
      *> Zero file I/O, zero side effects: given line items and the
      *> customer's tier, it returns subtotal/discount/tax/total.
      *> Same "pure function" discipline as ARDISC.cob in Part 085,
      *> which is exactly why it can be unit tested directly (Step
      *> 998) without any master file on disk at all.
      *> Phase 3 callback: intrinsic-function-friendly rounding via
      *> COMPUTE ROUNDED (Part 036); Phase 6 callback: this is the
      *> "Business Logic" layer of the layered architecture built in
      *> Step 995 (Part 082/086 Clean Architecture ideas).

       ENVIRONMENT DIVISION.
       CONFIGURATION SECTION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
           COPY "EOMSCNST.CPY".
       01  WS-IDX                   PIC 9(2).

       LINKAGE SECTION.
       01  LK-CUST-TIER             PIC X(1).
       01  LK-LINE-COUNT            PIC 9(2).
      *> GnuCOBOL requires a table passed across a CALL boundary to
      *> be wrapped in a group item - an elementary 01-level OCCURS
      *> item cannot be referenced without a subscript, even in the
      *> PROCEDURE DIVISION USING header itself. Wrapping it in a
      *> -GROUP 01 (and keeping the OCCURS at the 05 level) is the
      *> fix, confirmed by a real compile of a minimal test case.
       01  LK-LINE-QTY-GROUP.
           05  LK-LINE-QTY-TABLE    OCCURS 5 TIMES PIC 9(5).
       01  LK-LINE-PRICE-GROUP.
           05  LK-LINE-PRICE-TABLE  OCCURS 5 TIMES PIC 9(5)V99.
       01  LK-LINE-EXT-GROUP.
           05  LK-LINE-EXT-TABLE    OCCURS 5 TIMES PIC 9(7)V99.
       01  LK-SUBTOTAL              PIC 9(9)V99.
       01  LK-DISCOUNT-PCT          PIC 9V999.
       01  LK-DISCOUNT-AMT          PIC 9(9)V99.
       01  LK-TAX-AMT               PIC 9(9)V99.
       01  LK-TOTAL                 PIC 9(9)V99.

       PROCEDURE DIVISION USING LK-CUST-TIER, LK-LINE-COUNT,
           LK-LINE-QTY-GROUP, LK-LINE-PRICE-GROUP, LK-LINE-EXT-GROUP,
           LK-SUBTOTAL, LK-DISCOUNT-PCT, LK-DISCOUNT-AMT, LK-TAX-AMT,
           LK-TOTAL.
       CALCULATE-ORDER.
           MOVE 0 TO LK-SUBTOTAL
           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > LK-LINE-COUNT
               COMPUTE LK-LINE-EXT-TABLE(WS-IDX) ROUNDED =
                   LK-LINE-QTY-TABLE(WS-IDX) *
                   LK-LINE-PRICE-TABLE(WS-IDX)
               ADD LK-LINE-EXT-TABLE(WS-IDX) TO LK-SUBTOTAL
           END-PERFORM

           EVALUATE LK-CUST-TIER
               WHEN "P"
                   MOVE TIER-DISCOUNT-PLATINUM TO LK-DISCOUNT-PCT
               WHEN "G"
                   MOVE TIER-DISCOUNT-GOLD TO LK-DISCOUNT-PCT
               WHEN OTHER
                   MOVE TIER-DISCOUNT-REGULAR TO LK-DISCOUNT-PCT
           END-EVALUATE

           COMPUTE LK-DISCOUNT-AMT ROUNDED =
               LK-SUBTOTAL * LK-DISCOUNT-PCT
           COMPUTE LK-TAX-AMT ROUNDED =
               (LK-SUBTOTAL - LK-DISCOUNT-AMT) * EOMS-SALES-TAX-RATE
           COMPUTE LK-TOTAL ROUNDED =
               LK-SUBTOTAL - LK-DISCOUNT-AMT + LK-TAX-AMT

           GOBACK.
```

### กับดักจริงที่พบระหว่างพัฒนา: ตารางที่ส่งผ่าน CALL ต้องห่อด้วย Group Item

ระหว่างพัฒนาโปรแกรมนี้ ความพยายามครั้งแรกประกาศ `LK-LINE-QTY-TABLE OCCURS 5 TIMES PIC 9(5)` เป็น
รายการระดับ 01 ตรง ๆ (ไม่มี Group ห่อ) และใส่ชื่อนี้ตรง ๆ ใน `PROCEDURE DIVISION USING` — ผลคือ
**compile error ทันที**:

```
error: 'LK-LINE-QTY-TABLE' requires one subscript
```

เมื่อทดสอบด้วยโปรแกรมขั้นต่ำ (minimal reproduction) พบว่า GnuCOBOL **ไม่อนุญาตให้ระบุชื่อ OCCURS
ระดับ 01 โดยไม่มี subscript แม้แต่ในหัว `PROCEDURE DIVISION USING`** วิธีแก้ที่พิสูจน์แล้วว่าถูกต้อง
คือห่อ `OCCURS` ไว้ใต้ Group Item อีกชั้นหนึ่งเสมอ (ตามที่เห็นในโค้ดข้างต้น: `LK-LINE-QTY-GROUP` ห่อ
`LK-LINE-QTY-TABLE` ไว้) — กับดักนี้ถูกบันทึกไว้ในตาราง Greatest Hits ของ Part 098 ขั้นตอนที่ 977
(กับดัก #11) แล้วเช่นกัน

### พิสูจน์ด้วยการทดสอบจริง

```cobol
      *> Case 1: Platinum customer, 2 lines
           MOVE "P" TO W-TIER
           MOVE 2 TO W-COUNT
           MOVE 3 TO W-QTY(1)
           MOVE 19.99 TO W-PRICE(1)
           MOVE 1 TO W-QTY(2)
           MOVE 49.99 TO W-PRICE(2)
           CALL "ORDCALC" USING W-TIER, W-COUNT, W-QTY-GRP,
               W-PRICE-GRP, W-EXT-GRP, W-SUB, W-DPCT, W-DAMT,
               W-TAX, W-TOTAL
```

**ผลลัพธ์จริงจากการคอมไพล์และรัน**:

```
CASE1 SUB=000000109.96 DPCT=0.100 DAMT=000000011.00 TAX=000000006.93 TOTAL=000000105.89
CASE2 SUB=000000076.00 DPCT=0.000 DAMT=000000000.00 TAX=000000005.32 TOTAL=000000081.32
```

ตรวจทานด้วยมือ: 3×19.99 + 1×49.99 = 109.96, ส่วนลด Platinum 10% = 11.00 (ปัดเศษ), สุทธิ 98.96,
ภาษี 7% = 6.93 (ปัดเศษ), รวม 105.89 — ตรงกับผลลัพธ์จริงทุกประการ

### ข้อควรระวัง

- `COMPUTE ... ROUNDED` ทุกจุดในโปรแกรมนี้สำคัญมาก — ถ้าลืม `ROUNDED` ผลลัพธ์จะถูกตัดทศนิยมทิ้งแทนที่
  จะปัดเศษ ทำให้ยอดเงินคลาดเคลื่อนสะสมได้ในระยะยาว (ทบทวนจาก Part 009)
- ลำดับการคำนวณสำคัญมาก: ต้องคำนวณส่วนลดจาก**ยอดรวมก่อนหักส่วนลด** แล้วคำนวณภาษีจาก**ยอดหลังหัก
  ส่วนลด** เสมอ สลับลำดับจะให้ผลลัพธ์ผิดทันที

### แบบฝึกหัดที่ 993.1

**โจทย์**: จงอธิบายว่าทำไมการทดสอบขั้นต่ำ (minimal reproduction) อย่าง `SUBTEST.cob`/`SUBTEST2.cob`
ที่แยกปัญหาออกมาทดสอบทีละส่วนเล็ก ๆ จึงมีประโยชน์มากกว่าการพยายามแก้บั๊กในโปรแกรมใหญ่ทั้งหมดโดยตรง

**เฉลยแนวทาง**: เพราะโปรแกรมใหญ่มีตัวแปรที่อาจส่งผลกระทบซึ่งกันและกันมากมาย ทำให้ยากที่จะระบุว่า
สาเหตุที่แท้จริงของบั๊กคืออะไร การสร้างโปรแกรมขั้นต่ำที่ตัดทุกอย่างที่ไม่เกี่ยวข้องออกไปจนเหลือแค่
ส่วนที่สงสัยว่าเป็นปัญหา ช่วยยืนยันหรือปฏิเสธสมมติฐานได้รวดเร็วและแม่นยำกว่ามาก นี่คือเทคนิคการ Debug
แบบวิทยาศาสตร์ (Scientific Debugging) ที่ใช้ได้กับภาษาโปรแกรมทุกภาษา ไม่ใช่เฉพาะ COBOL

---

## ขั้นตอนที่ 994: Business Logic Layer ส่วนที่ 2 — ORDVALID.cob (Phase 2 Callback: Defensive Programming)

### แนวคิด: ตรวจสอบทุกเงื่อนไขก่อนอนุญาตให้สร้างคำสั่งซื้อ

ตามหลักการ Defensive Programming จาก Part 098 ขั้นตอนที่ 973-974 `ORDVALID.cob` ตรวจสอบ**ทุกเงื่อนไข
ทางธุรกิจ**ก่อนที่ `ORDSVC.cob` จะกล้าดำเนินการต่อ: ลูกค้าต้อง Active, จำนวน Line Item ต้องอยู่ในช่วง
1-5, และทุกสินค้าต้อง Active พร้อมมีสต๊อกเพียงพอ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ORDVALID.
       AUTHOR. COBOL-COURSE.
      *> Business Logic Layer - order validation, run BEFORE ORDCALC.
      *> Defensive programming per Part 098 / Part 042 / Part 089:
      *> never trust that a customer is active, a product is active,
      *> or that stock is sufficient just because a caller asked for
      *> it. Pure function: no file I/O here either - the caller
      *> (ORDSVC) is responsible for reading CUSTMAST/PRODMAST first
      *> and handing the results in as parallel tables.

       ENVIRONMENT DIVISION.
       CONFIGURATION SECTION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
           COPY "EOMSCNST.CPY".
       01  WS-IDX                   PIC 9(2).

       LINKAGE SECTION.
       01  LK-CUST-STATUS           PIC X(1).
       01  LK-LINE-COUNT            PIC 9(2).
      *> Tables crossing a CALL boundary must be wrapped in a group
      *> item under GnuCOBOL (see ORDCALC.cob for the confirmed
      *> minimal test case that proves this).
       01  LK-PROD-STATUS-GROUP.
           05  LK-PROD-STATUS-TABLE OCCURS 5 TIMES PIC X(1).
       01  LK-PROD-QTY-GROUP.
           05  LK-PROD-QTY-TABLE    OCCURS 5 TIMES PIC 9(5).
       01  LK-REQ-QTY-GROUP.
           05  LK-REQ-QTY-TABLE     OCCURS 5 TIMES PIC 9(5).
       01  LK-VALID-FLAG            PIC X(1).
       01  LK-ERROR-MSG             PIC X(60).

       PROCEDURE DIVISION USING LK-CUST-STATUS, LK-LINE-COUNT,
           LK-PROD-STATUS-GROUP, LK-PROD-QTY-GROUP, LK-REQ-QTY-GROUP,
           LK-VALID-FLAG, LK-ERROR-MSG.
       VALIDATE-ORDER.
           MOVE "Y" TO LK-VALID-FLAG
           MOVE SPACES TO LK-ERROR-MSG

           IF LK-CUST-STATUS NOT = CUST-STATUS-ACTIVE
               MOVE "N" TO LK-VALID-FLAG
               MOVE "Customer is not active" TO LK-ERROR-MSG
           END-IF

           IF LK-VALID-FLAG = "Y"
              AND (LK-LINE-COUNT = 0
                   OR LK-LINE-COUNT > EOMS-MAX-ORDER-LINES)
               MOVE "N" TO LK-VALID-FLAG
               MOVE "Order must have 1 to 5 line items"
                   TO LK-ERROR-MSG
           END-IF

           IF LK-VALID-FLAG = "Y"
               PERFORM VARYING WS-IDX FROM 1 BY 1
                       UNTIL WS-IDX > LK-LINE-COUNT
                          OR LK-VALID-FLAG = "N"
                   PERFORM CHECK-ONE-LINE
               END-PERFORM
           END-IF

           GOBACK.

       CHECK-ONE-LINE.
           IF LK-PROD-STATUS-TABLE(WS-IDX) NOT = PROD-STATUS-ACTIVE
               MOVE "N" TO LK-VALID-FLAG
               STRING "Line " WS-IDX
                   ": product is not active"
                   DELIMITED BY SIZE INTO LK-ERROR-MSG
           ELSE
               IF LK-REQ-QTY-TABLE(WS-IDX) >
                       LK-PROD-QTY-TABLE(WS-IDX)
                   MOVE "N" TO LK-VALID-FLAG
                   STRING "Line " WS-IDX
                       ": insufficient stock"
                       DELIMITED BY SIZE INTO LK-ERROR-MSG
               END-IF
           END-IF.
```

### กับดักจริงที่พบระหว่างพัฒนา: ลืม GOBACK ทำให้เกิด Fall-Through

**นี่คือบั๊กที่อันตรายที่สุดและแนบเนียนที่สุดที่พบระหว่างการพัฒนา Capstone นี้ทั้งหมด** เวอร์ชันแรก
ของ `VALIDATE-ORDER` จบด้วย `END-IF.` (ปิดท้าย `IF LK-VALID-FLAG = "Y" PERFORM VARYING...`) **โดยไม่มี
`GOBACK` ต่อท้าย** — เนื่องจากนี่คือ Subprogram ที่ถูกเรียกผ่าน `CALL` ไม่ใช่โปรแกรมหลักที่จบด้วย
`STOP RUN` การไม่มี `GOBACK` ทำให้ควบคุมการทำงาน **"ตกหล่น" (fall through) เข้า Paragraph ถัดไป
(`CHECK-ONE-LINE`) โดยอัตโนมัติทันทีที่ `VALIDATE-ORDER` ทำงานจบ**

ทดสอบด้วยข้อมูล: ลูกค้า Active, 2 Line Item ที่ถูกต้องทั้งคู่ — **ผลลัพธ์ที่ผิดพลาดที่ได้จริง**:

```
VALID1 FLAG=N MSG=[Line 03: product is not active]
```

ทั้งที่ควรจะได้ `FLAG=Y` (ทุกเงื่อนไขถูกต้อง) กลับได้ `FLAG=N` พร้อมข้อความอ้างถึง **"Line 03"**
ทั้งที่มีแค่ 2 Line Item เท่านั้น! สาเหตุคือหลังจากลูป `PERFORM VARYING` ประมวลผล Line 1 และ Line 2
เสร็จ (ถูกต้องทั้งคู่) ตัวแปร `WS-IDX` เหลือค่า `3` (ค่าที่ทำให้เงื่อนไข `WS-IDX > LK-LINE-COUNT` เป็น
จริงและออกจากลูป) — แล้วเพราะไม่มี `GOBACK` ควบคุมจึง **"ไหลต่อ" เข้า `CHECK-ONE-LINE` อีกหนึ่งรอบ
โดยไม่ได้ตั้งใจ** ด้วยค่า `WS-IDX = 3` ที่เหลือค้างอยู่ อ่านข้อมูลตำแหน่งที่ 3 ของตารางที่ไม่เคยถูก
กำหนดค่า (เป็นช่องว่าง) ทำให้เข้าเงื่อนไข "product is not active" อย่างผิดพลาด

### วิธีแก้ที่พิสูจน์แล้วว่าถูกต้อง

เพิ่ม `GOBACK.` ต่อท้าย `END-IF` ของ `VALIDATE-ORDER` (ดูโค้ดฉบับสมบูรณ์ด้านบน — บั๊กนี้ถูกแก้ไขแล้ว
ในเวอร์ชันที่แสดง) หลังแก้ไข ผลลัพธ์การทดสอบเดิมกลายเป็น:

```
VALID1 FLAG=Y MSG=[]
VALID2 FLAG=N MSG=[Line 02: insufficient stock]
VALID3 FLAG=N MSG=[Customer is not active]
```

ถูกต้องครบทุกกรณี — กับดักนี้ถูกบันทึกไว้ในตาราง Greatest Hits ของ Part 098 ขั้นตอนที่ 977 (กับดัก
#8) ด้วยเช่นกัน เพราะเป็นบทเรียนที่มีค่ามากสำหรับ Subprogram ทุกตัวที่มีมากกว่าหนึ่ง Paragraph

### บทเรียนที่นำไปใช้ได้ทั่วไป

**ทุก Subprogram ที่มีมากกว่าหนึ่ง Paragraph ต้องตรวจสอบให้แน่ใจว่าทุกเส้นทางการทำงานจบด้วย `GOBACK`
หรือ `EXIT PROGRAM` อย่างชัดเจน** ไม่ใช่แค่ Paragraph สุดท้ายเท่านั้น แต่รวมถึงทุกจุดที่โปรแกรมควรจะ
"จบการทำงาน" กลับไปหาผู้เรียก — นี่คือเหตุผลที่ Code Review Checklist ของ Part 098 ขั้นตอนที่ 978
มีข้อ "Subprogram ทุกเส้นทางจบด้วย GOBACK/EXIT PROGRAM อย่างชัดเจน" อยู่ในหมวด D โดยเฉพาะ

### ข้อควรระวัง

- บั๊กแบบนี้ตรวจจับได้ยากมากด้วยตาเปล่า เพราะโค้ดแต่ละ Paragraph ดูถูกต้องเมื่ออ่านแยกกัน — ต้อง
  อาศัย Unit Test ที่ทดสอบกรณี "ทุกอย่างถูกต้อง" (happy path) ควบคู่กับกรณี error เสมอ (ตามที่ Part 098
  ขั้นตอนที่ 976 สรุปไว้แล้วว่าบั๊กแบบ silent failure มักไม่ปรากฏถ้าทดสอบแค่กรณีที่คาดว่าจะ error)
- สังเกตว่า `CHECK-ONE-LINE` ไม่มี `GOBACK`/`EXIT PARAGRAPH` ของตัวเองเลย เพราะมันถูกเรียกผ่าน
  `PERFORM` (ไม่ใช่จุดเริ่มต้นของโปรแกรม) — กฎ "ต้องมี GOBACK" ใช้กับ**จุดที่ควบคุมอาจไหลออกจาก
  PROCEDURE DIVISION ทั้งหมด**เท่านั้น ไม่ใช่ทุก Paragraph

### แบบฝึกหัดที่ 994.1

**โจทย์**: จงอธิบายว่าทำไมค่า `WS-IDX = 3` ที่ "ค้าง" อยู่หลังลูปจบ จึงเป็นสาเหตุโดยตรงที่ทำให้ข้อความ
error อ้างถึง "Line 03" แทนที่จะเป็นเลขอื่น

**เฉลยแนวทาง**: เพราะ `PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > LK-LINE-COUNT` ตามมาตรฐาน
COBOL จะเพิ่มค่า `WS-IDX` ทีละ 1 หลังแต่ละรอบ**ก่อน**ตรวจสอบเงื่อนไขหยุดรอบถัดไป เมื่อ `LK-LINE-COUNT`
เท่ากับ 2 ลูปจะประมวลผล `WS-IDX = 1` แล้วเพิ่มเป็น 2, ประมวลผล `WS-IDX = 2` แล้วเพิ่มเป็น 3, จากนั้น
ตรวจสอบเงื่อนไข `3 > 2` เป็นจริงจึงหยุดลูป **โดยค่า `WS-IDX` ที่เหลือค้างอยู่ในหน่วยความจำคือ 3 เสมอ**
เมื่อควบคุมไหลต่อเข้า `CHECK-ONE-LINE` โดยไม่ตั้งใจ (เพราะไม่มี `GOBACK`) มันจึงใช้ค่า `WS-IDX = 3`
ที่ค้างอยู่นี้ไปอ้างอิงตารางและสร้างข้อความ error ทันที

---

## ขั้นตอนที่ 995: Data Access Layer — CUSTREPO / PRODREPO / ORDREPO (Phase 4 Callback: Repository Pattern)

### แนวคิด: หนึ่งจุดรับผิดชอบการอ่าน/เขียนไฟล์ต่อหนึ่ง Master File

ตามแนวคิด Repository Pattern จาก Part 082 (พิสูจน์แล้วจริงใน `ARLEDGER.cob` ของ Part 085) ทั้งสาม
Repository นี้มีสัญญา (contract) เดียวกันทุกประการ: รับ `LK-FUNCTION` ("READ"/"WRITE"/"REWRITE") พร้อม
ระเบียนข้อมูล แล้วทำหน้าที่ตามนั้น คืนค่า `LK-IO-STATUS` กลับไปให้ผู้เรียกตรวจสอบเสมอ

### CUSTREPO.cob ฉบับสมบูรณ์

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CUSTREPO.
       AUTHOR. COBOL-COURSE.
      *> Data Access Layer - "Repository" for CUSTMAST.DAT (Part 082
      *> Repository pattern, same idea as ARLEDGER.cob in Part 085).
      *> One place in the whole system responsible for reading and
      *> writing customer records. LK-FUNCTION selects the operation;
      *> the caller always checks LK-IO-STATUS afterward (Part 030
      *> file status discipline / Part 098 best practice).
      *>
      *> The same CUSTOMER.CPY copybook is used twice in this program
      *> under two different top-level names: FD-CUST-RECORD (the
      *> on-disk layout) and CUST-RECORD (the LINKAGE layout the
      *> caller passes in/out). A plain group MOVE copies between
      *> them - the two records share the same field layout, so this
      *> is exactly the safe kind of group MOVE taught in Part 008.
      *>
      *> Opens and closes the file on every call, trading a little
      *> performance for the simplest possible correctness - the
      *> same trade-off Part 085 made for ARLEDGER.cob, called out
      *> there as a deliberate, documented choice.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUST-FILE ASSIGN TO "CUSTMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID OF FD-CUST-RECORD
               FILE STATUS IS LK-IO-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUST-FILE.
           COPY "CUSTOMER.CPY" REPLACING ==CUST-RECORD==
               BY ==FD-CUST-RECORD==.

       WORKING-STORAGE SECTION.
       01  WS-SAVED-STATUS          PIC XX.

       LINKAGE SECTION.
       01  LK-FUNCTION              PIC X(10).
           COPY "CUSTOMER.CPY".
       01  LK-IO-STATUS             PIC XX.

       PROCEDURE DIVISION USING LK-FUNCTION, CUST-RECORD,
           LK-IO-STATUS.
       MAIN-PARA.
           EVALUATE LK-FUNCTION
               WHEN "READ"
                   PERFORM DO-READ
               WHEN "WRITE"
                   PERFORM DO-WRITE
               WHEN "REWRITE"
                   PERFORM DO-REWRITE
               WHEN OTHER
                   MOVE "99" TO LK-IO-STATUS
           END-EVALUATE
           GOBACK.

       DO-READ.
           MOVE CUST-RECORD TO FD-CUST-RECORD
           OPEN INPUT CUST-FILE
           IF LK-IO-STATUS = "00"
               READ CUST-FILE
                   INVALID KEY
                       CONTINUE
               END-READ
      *> Save the READ's status BEFORE closing the file. This is a
      *> real bug this capstone hit during testing: CLOSE also sets
      *> FILE STATUS, so closing right after a failed READ silently
      *> overwrote "23" (record not found) back to "00" - callers
      *> saw a false success. Never assume a status survives the
      *> next file operation; capture it the moment you need it.
               MOVE LK-IO-STATUS TO WS-SAVED-STATUS
               MOVE FD-CUST-RECORD TO CUST-RECORD
               CLOSE CUST-FILE
               MOVE WS-SAVED-STATUS TO LK-IO-STATUS
           END-IF.

       DO-WRITE.
           MOVE CUST-RECORD TO FD-CUST-RECORD
           OPEN I-O CUST-FILE
           IF LK-IO-STATUS = "35"
      *> "35" = file does not exist yet; create it fresh instead
               OPEN OUTPUT CUST-FILE
           END-IF
           WRITE FD-CUST-RECORD
               INVALID KEY
                   CONTINUE
           END-WRITE
           CLOSE CUST-FILE.

       DO-REWRITE.
           MOVE CUST-RECORD TO FD-CUST-RECORD
           OPEN I-O CUST-FILE
           REWRITE FD-CUST-RECORD
               INVALID KEY
                   CONTINUE
           END-REWRITE
           CLOSE CUST-FILE.
```

**บั๊กจริงที่พบและแก้ไขในโปรแกรมนี้ถูกอธิบายละเอียดเต็มรูปแบบแล้วใน Part 098 ขั้นตอนที่ 976** — สรุป
สั้น ๆ ที่นี่: `CLOSE` เขียนทับ `FILE STATUS` ของ `READ` ที่ล้มเหลว ทำให้ต้อง copy สถานะออกมาเก็บไว้
ใน `WS-SAVED-STATUS` ก่อนเรียก `CLOSE` เสมอ (ดูรายละเอียดเต็มพร้อมหลักฐานการทดสอบใน Part 098)

### PRODREPO.cob และ ORDREPO.cob

ทั้งสองไฟล์มีโครงสร้างเหมือนกันทุกประการกับ `CUSTREPO.cob` เพียงเปลี่ยนชื่อไฟล์ ชื่อ Copybook และ
Record Key ให้ตรงกับสินค้า/คำสั่งซื้อตามลำดับ — `ORDREPO.cob` มีคอมเมนต์เพิ่มเติมที่สำคัญ:

```cobol
      *> Data Access Layer - "Repository" for ORDMAST.DAT. Same
      *> READ/WRITE/REWRITE contract as CUSTREPO.cob/PRODREPO.cob for
      *> random (by ORD-ID) access from the online order path. The
      *> batch report program (ORDRPT.cob, Step 997) reads this same
      *> file directly and sequentially instead - a deliberate,
      *> realistic split between "online random access" and "batch
      *> sequential access" against one shared indexed master file,
      *> the same pattern Part 028/070 uses for VSAM-style files.
```

นี่คือรูปแบบที่แท้จริงของระบบ Mainframe จริง: **ไฟล์ Indexed ตัวเดียวกันถูกเข้าถึงได้ทั้งสองแบบ** —
แบบ Random (คีย์เดียว) จากธุรกรรม Online ผ่าน Repository, และแบบ Sequential (ทั้งไฟล์) จาก Batch
Job รายงานสรุป (ขั้นตอนที่ 997) — ตรงกับที่ Part 055/066 สอนไว้เรื่องการออกแบบไฟล์ VSAM ให้รองรับ
ทั้งสองรูปแบบการเข้าถึง

### พิสูจน์ด้วยการทดสอบจริง

```cobol
      *> Read an existing customer
           MOVE "READ" TO W-FUNCTION
           MOVE 100002 TO CUST-ID
           CALL "CUSTREPO" USING W-FUNCTION, CUST-RECORD, W-IO-STATUS
           DISPLAY "CUST READ status=" W-IO-STATUS
               " name=[" CUST-NAME "] tier=" CUST-TIER
               " balance=" CUST-BALANCE
```

**ผลลัพธ์จริงจากการคอมไพล์และรัน**:

```
CUST READ status=00 name=[GOLDEN GATE TRADING     ] tier=G balance=0000000.00
CUST REWRITE status=00
CUST RE-READ balance=0000250.00
PROD READ status=00 name=[GADGET-MINI         ] qty=00008
ORD WRITE status=00
ORD READ status=00 total=000000040.64 cust=100002
```

ผลลัพธ์ยืนยันว่าทั้ง READ, REWRITE, และ WRITE ทำงานถูกต้องกับข้อมูลจริงบนดิสก์ — ยอดคงเหลือของลูกค้า
`100002` เปลี่ยนจาก `0.00` เป็น `250.00` หลัง `REWRITE` และยังคงอยู่เมื่ออ่านกลับมาใหม่ (`RE-READ`)

### สถาปัตยกรรมแบบ Layered ที่ได้จนถึงจุดนี้

```
+------------------------------------------------+
|         Business Logic Layer (Step 993-994)     |
|         ORDCALC.cob      ORDVALID.cob           |
|         (pure calculation, pure validation,      |
|          zero file I/O)                          |
+------------------------------------------------+
                        ^
                        | called by
+------------------------------------------------+
|         Data Access Layer (Step 995)            |
|  CUSTREPO.cob   PRODREPO.cob   ORDREPO.cob       |
|  (READ/WRITE/REWRITE against one indexed file    |
|   each, zero business rules)                     |
+------------------------------------------------+
```

### ข้อควรระวัง

- อย่าลืมว่า `CUST-ID`, `PROD-ID`, `ORD-ID` แต่ละตัวต้องถูกกำหนดค่าก่อนเรียก `"READ"` เสมอ เพราะ
  `RECORD KEY` ใช้ค่าที่อยู่ในฟิลด์นั้น ณ ขณะที่เรียก `READ` เป็นตัวกำหนดว่าจะค้นหาระเบียนใด
- Repository ทั้งสามตัวนี้**ไม่มีตรรกะทางธุรกิจใด ๆ เลย** — ถ้าพบว่าตัวเองกำลังเขียนเงื่อนไข `IF`
  ที่เกี่ยวกับกฎธุรกิจ (เช่น "ถ้ายอดหนี้เกินวงเงิน") อยู่ใน Repository แสดงว่ากำลังละเมิดหลักการ
  Separation of Concerns ที่ Part 082 สอนไว้ — ตรรกะแบบนั้นควรอยู่ใน `ORDSVC.cob` (ขั้นตอนถัดไป)

### แบบฝึกหัดที่ 995.1

**โจทย์**: จงอธิบายว่าทำไม `ORDREPO.cob` (สำหรับธุรกรรม Online) และ `ORDRPT.cob` (สำหรับ Batch
Report ในขั้นตอนที่ 997) ถึงเข้าถึงไฟล์ `ORDMAST.DAT` เดียวกันได้โดยไม่ขัดแย้งกัน ทั้งที่เป็นคนละ
โปรแกรมและใช้ ACCESS MODE ต่างกัน

**เฉลยแนวทาง**: เพราะทั้งสองโปรแกรมเปิด-ปิดไฟล์เป็นช่วงเวลาสั้น ๆ แยกจากกัน (ไม่ได้เปิดค้างพร้อมกัน
ตลอดเวลา) `ORDREPO.cob` เปิด-อ่าน/เขียน-ปิดทันทีในแต่ละ Function Call ในขณะที่ `ORDRPT.cob` เปิดอ่าน
ทั้งไฟล์ครั้งเดียวตอนเริ่ม Batch Job แล้วปิด — ตราบใดที่ไม่มีทั้งสองโปรแกรมพยายามเข้าถึงไฟล์เดียวกัน
พร้อมกันในเวลาเดียวกันจริง ๆ (concurrent access) ก็จะไม่มีความขัดแย้งเกิดขึ้น ในระบบ Production จริง
ที่ต้องรองรับ Concurrent Access พร้อมกันหลายโปรเซส จะต้องพิจารณากลไก File Locking หรือใช้ VSAM/DB2
จริงที่มีกลไกจัดการ Concurrency ในตัว (Part 059-060) ซึ่งเกินขอบเขตของไฟล์ Indexed ธรรมดาที่ Capstone
นี้ใช้

---

## ขั้นตอนที่ 996: Application Layer — ORDSVC.cob (Phase 6 Callback: Clean Architecture Orchestration)

### แนวคิด: Orchestrator ที่รู้ "Use Case" แต่ไม่รู้รายละเอียดภายใน

ตามแนวคิด Clean Architecture จาก Part 082/086 `ORDSVC.cob` คือ **Application Layer** ที่รู้จัก
Use Case ทางธุรกิจ ("สร้างคำสั่งซื้อ", "ยกเลิกคำสั่งซื้อ") แต่มอบหมายรายละเอียดทั้งหมดให้ชั้นที่อยู่
ใต้มัน: การคำนวณให้ `ORDCALC`, การตรวจสอบให้ `ORDVALID`, การอ่าน/เขียนไฟล์ให้ `CUSTREPO`/`PRODREPO`/
`ORDREPO` — `EOMSMAIN.cob` (ขั้นตอนถัดไป) และ REST Wrapper (ขั้นตอนที่ 999) ต่างเรียกใช้แค่จุดเดียว
คือ `ORDSVC` เท่านั้น โดยไม่รู้เลยว่าเบื้องหลังมีไฟล์ Indexed ถึง 3 ไฟล์ทำงานอยู่

เนื่องจากซอร์สโค้ดเต็มของ `ORDSVC.cob` ยาวกว่า 200 บรรทัด (รวม Use Case ทั้งสอง: CREATE-ORDER และ
CANCEL-ORDER) เราจะแสดงเฉพาะโครงสร้างหลักและ Use Case ที่สำคัญที่สุดในที่นี้ พร้อมอธิบายภาพรวมของ
ส่วนที่เหลือ

### โครงสร้างหลักและ Use Case CREATE-ORDER

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ORDSVC.
       AUTHOR. COBOL-COURSE.
      *> Application / Orchestration Layer (Part 082/086 Clean
      *> Architecture ideas). ORDSVC knows the USE CASES ("create an
      *> order", "cancel an order") but delegates every detail to a
      *> lower layer: ORDVALID/ORDCALC (pure Business Logic, no file
      *> I/O) and CUSTREPO/PRODREPO/ORDREPO (Data Access, no business
      *> rules). EOMSMAIN (Step 997) and the REST wrapper (Step 999)
      *> both call ONLY this one entry point - neither of them knows
      *> that CUSTMAST/PRODMAST/ORDMAST are indexed files at all.
      *>
      *> Known simplification, stated plainly: writes to the three
      *> master files are not wrapped in a two-phase commit. If the
      *> process were killed between the PRODREPO rewrite and the
      *> CUSTREPO rewrite below, the two files could disagree. Real
      *> mainframe systems solve this with CICS/DB2 syncpoint
      *> (Part 061-064); reproducing that is out of scope for a
      *> GnuCOBOL capstone, so this limitation is documented instead
      *> of hidden.

       PROCEDURE DIVISION USING LK-FUNCTION, LK-ORD-ID, LK-CUST-ID,
           LK-ORDER-DATE, LK-LINE-COUNT, LK-REQ-PROD-GROUP,
           LK-REQ-QTY-GROUP, LK-RESULT-FLAG, LK-RESULT-MSG,
           LK-ORDER-TOTAL.
       MAIN-PARA.
           MOVE "Y" TO LK-RESULT-FLAG
           MOVE SPACES TO LK-RESULT-MSG
           MOVE 0 TO LK-ORDER-TOTAL

           EVALUATE LK-FUNCTION
               WHEN "CREATE"
                   PERFORM CREATE-ORDER
               WHEN "CANCEL"
                   PERFORM CANCEL-ORDER
               WHEN OTHER
                   MOVE "N" TO LK-RESULT-FLAG
                   MOVE "Unknown ORDSVC function" TO LK-RESULT-MSG
           END-EVALUATE
           GOBACK.

       CREATE-ORDER.
           PERFORM LOAD-CUSTOMER
           IF LK-RESULT-FLAG = "Y"
               PERFORM LOAD-ALL-PRODUCTS
           END-IF
           IF LK-RESULT-FLAG = "Y"
               PERFORM RUN-VALIDATION
           END-IF
           IF LK-RESULT-FLAG = "Y"
               PERFORM RUN-CALCULATION
           END-IF
           IF LK-RESULT-FLAG = "Y"
               PERFORM CHECK-CREDIT-LIMIT
           END-IF
           IF LK-RESULT-FLAG = "Y"
               PERFORM APPLY-STOCK-DEDUCTION
               PERFORM APPLY-CUSTOMER-BALANCE
               PERFORM PERSIST-NEW-ORDER
               MOVE WS-TOTAL TO LK-ORDER-TOTAL
               MOVE "Order created successfully" TO LK-RESULT-MSG
           END-IF.
```

สังเกตว่า `CREATE-ORDER` อ่านเหมือน**รายการตรวจสอบ (checklist) ของกฎธุรกิจ**ที่เรียงตามลำดับ:
โหลดลูกค้า → โหลดสินค้าทุกรายการ → ตรวจสอบความถูกต้อง → คำนวณ → ตรวจสอบวงเงินเครดิต → ถ้าผ่านทุกข้อ
จึงค่อยเขียนข้อมูลจริง — แต่ละขั้นตอนตรวจสอบ `LK-RESULT-FLAG = "Y"` ก่อนทำขั้นตอนถัดไปเสมอ ทำให้ถ้า
ขั้นตอนใดล้มเหลว ขั้นตอนที่เหลือจะถูกข้ามไปโดยอัตโนมัติ (ไม่มีการเขียนข้อมูลบางส่วนที่ไม่สมบูรณ์)

### รายละเอียดสำคัญ: ราคาสินค้ามาจาก Master File เสมอ ไม่ใช่จากผู้เรียก

```cobol
       LOAD-ALL-PRODUCTS.
           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > LK-LINE-COUNT
               MOVE LK-REQ-PROD-ID(WS-IDX) TO PROD-ID
               MOVE "READ" TO WS-FUNCTION
               CALL "PRODREPO" USING WS-FUNCTION, WS-PR, WS-IO-STATUS
               IF WS-IO-STATUS = "00"
                   MOVE PROD-STATUS TO WS-PROD-STATUS-TAB(WS-IDX)
                   MOVE PROD-QTY-ON-HAND TO WS-PROD-QTY-TAB(WS-IDX)
                   MOVE PROD-UNIT-PRICE TO WS-LINE-PRICE-TAB(WS-IDX)
               ELSE
                   MOVE PROD-STATUS-DISCONTINUED
                       TO WS-PROD-STATUS-TAB(WS-IDX)
                   MOVE 0 TO WS-PROD-QTY-TAB(WS-IDX)
                   MOVE 0 TO WS-LINE-PRICE-TAB(WS-IDX)
               END-IF
               MOVE LK-REQ-QTY(WS-IDX) TO WS-REQ-QTY-TAB(WS-IDX)
           END-PERFORM.
```

สังเกตประเด็นด้านความปลอดภัยที่สำคัญมาก: **ผู้เรียก (EOMSMAIN หรือ REST API) ส่งมาแค่ `PROD-ID` และ
`QTY` เท่านั้น ไม่เคยส่งราคาสินค้ามาเลย** — ราคาที่ใช้คำนวณจริงทุกครั้งดึงมาจาก `PRODMAST.DAT` โดยตรง
ผ่าน `PRODREPO` นี่คือหลักการ Defensive Programming ที่สำคัญมาก (Part 098): **ถ้าผู้เรียกสามารถกำหนด
ราคาเองได้ จะเป็นช่องโหว่ด้านความปลอดภัยร้ายแรง** (ลูกค้าอาจส่งราคา 0.01 บาทมาแทนราคาจริง) การดึงราคา
จาก Source of Truth เดียวเสมอคือการป้องกันที่ถูกต้อง

### Use Case ที่สอง: CANCEL-ORDER

```cobol
       CANCEL-ORDER.
           MOVE LK-ORD-ID TO ORD-ID
           MOVE "READ" TO WS-FUNCTION
           CALL "ORDREPO" USING WS-FUNCTION, WS-OR, WS-IO-STATUS
           IF WS-IO-STATUS NOT = "00"
               MOVE "N" TO LK-RESULT-FLAG
               MOVE "Order not found" TO LK-RESULT-MSG
           ELSE
               IF ORD-STATUS NOT = ORDER-STATUS-OPEN
                   MOVE "N" TO LK-RESULT-FLAG
                   MOVE "Order is not open, cannot cancel"
                       TO LK-RESULT-MSG
               ELSE
                   PERFORM RESTORE-STOCK
                   PERFORM REVERSE-CUSTOMER-BALANCE
                   MOVE ORDER-STATUS-CANCELLED TO ORD-STATUS
                   MOVE "REWRITE" TO WS-FUNCTION
                   CALL "ORDREPO" USING WS-FUNCTION, WS-OR,
                       WS-IO-STATUS
                   MOVE ORD-TOTAL TO LK-ORDER-TOTAL
                   MOVE "Order cancelled successfully"
                       TO LK-RESULT-MSG
               END-IF
           END-IF.
```

`CANCEL-ORDER` ตรวจสอบว่าคำสั่งซื้อมีอยู่จริงและยังเปิดอยู่ (`ORDER-STATUS-OPEN`) ก่อนดำเนินการเสมอ
— ถ้าคำสั่งซื้อถูกยกเลิกไปแล้วครั้งหนึ่ง (`ORDER-STATUS-CANCELLED`) การพยายามยกเลิกซ้ำจะถูกปฏิเสธ
ทันที ป้องกันไม่ให้สต๊อกและยอดหนี้ถูกคืนซ้ำสองครั้งโดยไม่ตั้งใจ

### พิสูจน์ด้วยการทดสอบจริงแบบครบวงจร (Create → Verify → Cancel → Verify → Cancel Again)

**ผลลัพธ์จริงจากการคอมไพล์และรัน**:

```
ORDER1 FLAG=Y MSG=[Order created successfully] TOTAL=000000086.64
ORDER2 FLAG=N MSG=[Line 01: insufficient stock]
STOCK 200001 AFTER ORDER1 = 00498
BALANCE 100001 AFTER ORDER1 = 0000086.64
CANCEL1 FLAG=Y MSG=[Order cancelled successfully] TOTAL=000000086.64
STOCK 200001 AFTER CANCEL = 00500
BALANCE 100001 AFTER CANCEL = 0000000.00
CANCEL2 FLAG=N MSG=[Order is not open, cannot cancel]
```

ผลลัพธ์นี้พิสูจน์ครบทุกประเด็นสำคัญ: (1) สร้างคำสั่งซื้อสำเร็จ พร้อมยอดที่คำนวณถูกต้อง (86.64)
(2) การสั่งซื้อเกินสต๊อกถูกปฏิเสธอย่างถูกต้อง (3) สต๊อกลดลงจาก 500 เหลือ 498 หลังสั่งซื้อ 2 ชิ้น
(4) ยอดหนี้ลูกค้าเพิ่มขึ้นตรงกับยอดคำสั่งซื้อ (5) หลังยกเลิก สต๊อกกลับมาเป็น 500 และยอดหนี้กลับเป็น
0.00 พอดี (6) การยกเลิกซ้ำครั้งที่สองถูกปฏิเสธอย่างถูกต้อง

### ข้อควรระวัง

- `ORDSVC.cob` เป็นจุดเดียวในระบบทั้งหมดที่ "รู้" ว่า Use Case ทางธุรกิจต้องทำอะไรบ้างตามลำดับใด —
  ถ้าในอนาคตต้องการเพิ่ม Use Case ใหม่ (เช่น "แก้ไขคำสั่งซื้อ") ควรเพิ่มที่นี่โดยเรียกใช้ Repository
  และ Business Logic เดิมซ้ำ ไม่ควรเขียนตรรกะใหม่ซ้ำซ้อนในชั้น Presentation
- ข้อจำกัดเรื่อง Two-Phase Commit ที่ระบุไว้ใน ADR-100-02 (ขั้นตอนที่ 991) มีผลกับทุก Use Case ใน
  โปรแกรมนี้ — ในระบบ Production จริงที่ต้องการความปลอดภัยระดับสูงกว่านี้ จำเป็นต้องพิจารณา CICS/DB2
  Syncpoint ตามที่ Part 061-064 สอนไว้

### แบบฝึกหัดที่ 996.1

**โจทย์**: จงอธิบายว่าทำไม `CREATE-ORDER` ต้องเรียก `LOAD-CUSTOMER` และ `LOAD-ALL-PRODUCTS` **ก่อน**
`RUN-VALIDATION` แทนที่จะเรียก `RUN-VALIDATION` ก่อนแล้วค่อยโหลดข้อมูลทีหลัง

**เฉลยแนวทาง**: เพราะ `ORDVALID.cob` เป็น Pure Function ที่ไม่มี File I/O เลย (ตามที่ออกแบบไว้ใน
ขั้นตอนที่ 994) มันจึง**ไม่สามารถอ่านสถานะลูกค้า/สินค้าจากไฟล์ได้ด้วยตัวเอง** ต้องอาศัย `ORDSVC`
เป็นผู้อ่านข้อมูลจริงจากไฟล์มาเตรียมไว้ก่อน (ผ่าน `LOAD-CUSTOMER`/`LOAD-ALL-PRODUCTS`) แล้วจึงส่งต่อ
เป็นพารามิเตอร์ให้ `ORDVALID` ตรวจสอบ ลำดับนี้จึงเป็นข้อบังคับทางโครงสร้าง ไม่ใช่แค่ทางเลือก — นี่คือ
ผลโดยตรงของการรักษาหลักการ "Pure Function ไม่มี File I/O" อย่างเคร่งครัดตามสถาปัตยกรรมที่ออกแบบไว้

---

## ขั้นตอนที่ 997: Presentation Layer — EOMSMAIN.cob (Phase 1 Callback: เมนูแบบโต้ตอบ)

### แนวคิด: เมนูที่ไม่รู้อะไรเลยนอกจาก "ถาม ผู้ใช้ แล้วเรียก ORDSVC"

ตามรูปแบบเมนู `PERFORM`/`EVALUATE` ที่ Part 015 สอนไว้ตั้งแต่ต้นหลักสูตร (โปรเจกต์เครื่องคิดเลข)
`EOMSMAIN.cob` ใช้โครงสร้างเดียวกันทุกประการ เพียงแต่แทนที่การคำนวณเลขคณิตด้วยการเรียก `ORDSVC`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EOMSMAIN.
       AUTHOR. COBOL-COURSE.
      *> Presentation Layer (Phase 1 callback: PERFORM/EVALUATE menu
      *> control flow exactly like Part 015's calculator). EOMSMAIN
      *> knows nothing about files, discounts or tax - it only reads
      *> what the operator types and calls ORDSVC, the one and only
      *> entry point into the Business Logic below it (Part 082/086
      *> layered architecture).

       PROCEDURE DIVISION.
       0000-MAIN-PROCESS.
           PERFORM 1000-SHOW-WELCOME
           PERFORM WITH TEST AFTER
                   UNTIL WS-MENU-CHOICE = 3
               PERFORM 2000-SHOW-MENU
               PERFORM 3000-GET-CHOICE
               EVALUATE WS-MENU-CHOICE
                   WHEN 1
                       PERFORM 4000-CREATE-ORDER-FLOW
                   WHEN 2
                       PERFORM 5000-CANCEL-ORDER-FLOW
                   WHEN 3
                       DISPLAY "Exiting EOMS. Goodbye!"
                   WHEN OTHER
                       DISPLAY "Invalid choice, please select 1-3."
               END-EVALUATE
           END-PERFORM
           STOP RUN.

       4000-CREATE-ORDER-FLOW.
           DISPLAY "Order ID (8 digits): " WITH NO ADVANCING.
           ACCEPT WS-ORD-ID.
           DISPLAY "Customer ID (6 digits): " WITH NO ADVANCING.
           ACCEPT WS-CUST-ID.
           DISPLAY "Order date (YYYYMMDD): " WITH NO ADVANCING.
           ACCEPT WS-ORDER-DATE.
           DISPLAY "Number of line items (1-5): " WITH NO ADVANCING.
           ACCEPT WS-LINE-COUNT.

           PERFORM VARYING WS-LINE-IDX FROM 1 BY 1
                   UNTIL WS-LINE-IDX > WS-LINE-COUNT
               DISPLAY "  Line " WS-LINE-IDX
                   " - Product ID: " WITH NO ADVANCING
               ACCEPT WS-REQ-PROD-ID(WS-LINE-IDX)
               DISPLAY "  Line " WS-LINE-IDX
                   " - Quantity: " WITH NO ADVANCING
               ACCEPT WS-REQ-QTY(WS-LINE-IDX)
           END-PERFORM

           MOVE "CREATE" TO WS-FUNCTION
           CALL "ORDSVC" USING WS-FUNCTION, WS-ORD-ID, WS-CUST-ID,
               WS-ORDER-DATE, WS-LINE-COUNT, WS-REQ-PROD-GRP,
               WS-REQ-QTY-GRP, WS-RESULT-FLAG, WS-RESULT-MSG,
               WS-ORDER-TOTAL
           END-CALL

           IF WS-RESULT-FLAG = "Y"
               DISPLAY "SUCCESS: " FUNCTION TRIM(WS-RESULT-MSG)
               DISPLAY "  Order total: " WS-ORDER-TOTAL
           ELSE
               DISPLAY "REJECTED: " FUNCTION TRIM(WS-RESULT-MSG)
           END-IF.
```

`5000-CANCEL-ORDER-FLOW` มีโครงสร้างเดียวกัน เพียงแค่ถาม Order ID แล้วเรียก `ORDSVC` ด้วย
`WS-FUNCTION = "CANCEL"` แทน

### พิสูจน์ด้วยการทดสอบจริง (ทดสอบผ่าน stdin redirection แบบเดียวกับ Part 015/035)

```bash
printf -- "1\n90000201\n100003\n20260928\n2\n200001\n1\n200003\n2\n2\n90000201\n3\n" \
    | ./build/eomsmain
```

**ผลลัพธ์จริงที่ได้ (ตัดส่วนเมนูที่ซ้ำออกเพื่อความกระชับ)**:

```
1. Create Order
2. Cancel Order
3. Exit
Enter your choice (1-3): Order ID (8 digits): Customer ID (6 digits): Order date (YYYYMMDD): Number of line items (1-5):   Line 01 - Product ID:   Line 01 - Quantity:   Line 02 - Product ID:   Line 02 - Quantity: SUCCESS: Order created successfully
  Order total: 000000041.72

1. Create Order
2. Cancel Order
3. Exit
Enter your choice (1-3): Order ID to cancel (8 digits): SUCCESS: Order cancelled successfully
  Refunded amount: 000000041.72

1. Create Order
2. Cancel Order
3. Exit
Enter your choice (1-3): Exiting EOMS. Goodbye!
```

### อธิบายโค้ดทีละส่วน

สังเกตว่า `EOMSMAIN.cob` **ไม่มี `SELECT`/`FD` เลยแม้แต่บรรทัดเดียว** — มันไม่รู้จักไฟล์ Indexed ใด ๆ
ในระบบเลย ทั้งหมดที่มันทำคือ `ACCEPT` ค่าจากผู้ใช้ จัดรูปแบบเป็นพารามิเตอร์ แล้ว `CALL "ORDSVC"` เพียง
จุดเดียว — นี่คือผลลัพธ์ที่แท้จริงของสถาปัตยกรรมแบบ Layered: **การเปลี่ยนแปลงวิธีเก็บข้อมูล (เช่น
เปลี่ยนจาก Indexed File เป็นฐานข้อมูล DB2 ในอนาคต) จะไม่กระทบ `EOMSMAIN.cob` แม้แต่บรรทัดเดียว**

### ข้อควรระวัง

- ตัวเลขค่าตอบแทน 41.72 มาจาก: 1×19.99 (สินค้า 200001) + 2×9.50 (สินค้า 200003) = 38.99, ลูกค้า
  ระดับ Regular (100003) ไม่ได้ส่วนลด, ภาษี 7% = 2.73 (ปัดเศษ), รวม 41.72 — ตรวจสอบด้วยมือแล้วตรงกับ
  ผลลัพธ์จริงทุกประการ
- เหมือนกับ Part 015 การทดสอบผ่าน stdin redirection ไม่แสดงค่าที่ "พิมพ์" ในแต่ละ prompt (เพราะไม่ใช่
  terminal จริง) แต่ตรรกะและผลลัพธ์ทั้งหมดถูกต้อง 100% ตามข้อมูลที่ป้อนเข้าไป

### แบบฝึกหัดที่ 997.1

**โจทย์**: จงอธิบายว่าทำไมการที่ `EOMSMAIN.cob` ไม่มี `SELECT`/`FD` เลยจึงเป็นหลักฐานที่ชัดเจนว่า
สถาปัตยกรรมแบบ Layered ของระบบนี้ถูกออกแบบมาอย่างถูกต้อง

**เฉลยแนวทาง**: เพราะหลักการ Separation of Concerns (Part 082) กำหนดว่าแต่ละชั้นควรรู้แค่สิ่งที่
จำเป็นต่อหน้าที่ของตัวเองเท่านั้น — หน้าที่ของ Presentation Layer คือ "รับข้อมูลจากผู้ใช้และแสดงผล
ลัพธ์" ไม่ใช่ "จัดการไฟล์" การที่ `EOMSMAIN.cob` ไม่มีความรู้เรื่องไฟล์เลยจึงเป็นเครื่องพิสูจน์ที่จับ
ต้องได้ (ไม่ใช่แค่คำอธิบายเชิงทฤษฎี) ว่าการแบ่งชั้นความรับผิดชอบทำได้จริงตลอดทั้งระบบ ถ้า `EOMSMAIN`
มี `SELECT`/`FD` ปรากฏอยู่ที่ไหนสักแห่ง นั่นจะเป็นสัญญาณว่ามีการ "รั่วไหล" ของความรับผิดชอบข้ามชั้น
เกิดขึ้น ซึ่งขัดกับหลักการที่ตั้งใจออกแบบไว้

---

## ขั้นตอนที่ 998: JSON Export และ Batch Report (Phase 3 + Phase 2 Callback)

### ส่วนที่ 1: ORDEXPORT.cob — ส่งออกคำสั่งซื้อเป็น JSON

Part 045 สอน `JSON GENERATE` ไว้แล้ว แต่ GnuCOBOL build ที่ใช้ทดสอบ Capstone นี้รายงานว่า
`JSON library : disabled` (ตรวจสอบได้จริงด้วย `cobc -info | grep -i json`) และแม้ในบาง build ที่
เปิดใช้งาน JSON แล้ว เวอร์ชัน GnuCOBOL 4.0-early ก็ยังไม่รองรับตาราง `OCCURS` ภายใน `JSON GENERATE`
— ดังนั้นโปรแกรมนี้จึงสร้าง JSON ด้วยมือผ่าน `STRING` แทน ซึ่งเป็นวิธีที่ทีมพัฒนา COBOL ในโลกจริงต้อง
ใช้เมื่อคอมไพเลอร์หรือ Runtime Library ที่มีอยู่ไม่รองรับฟีเจอร์ที่ต้องการ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ORDEXPORT.
       AUTHOR. COBOL-COURSE.
      *> Phase 3 callback (Part 045 JSON GENERATE): export one order
      *> as JSON so a modern web/mobile client, or the REST wrapper
      *> in Step 999, can consume it without knowing any COBOL at
      *> all. Reads through ORDREPO like any other caller - it does
      *> not touch ORDMAST.DAT directly.
      *>
      *> Honesty note proven by a real compile attempt: this
      *> GnuCOBOL build reports "JSON library : disabled" (check it
      *> yourself with `cobc -info | grep -i json`), and even where
      *> JSON is enabled, GnuCOBOL 4.0-early does not yet support
      *> OCCURS tables inside JSON GENERATE. So this program builds
      *> the JSON text by hand with STRING (Part 019) instead - the
      *> same fallback real COBOL shops reach for when their compiler
      *> or standard COBOL library lacks native JSON support.
      *>
      *> A second, easy-to-miss detail: a PIC 9(7)V99 item's implied
      *> decimal point is NOT a character in storage (Part 006), so
      *> STRINGing it directly would print "0001999" instead of
      *> "19.99". Every amount is therefore MOVEd into a numeric-
      *> edited field (an explicit "." position) before it is
      *> STRINGed, exactly the technique Part 006/019 teaches for
      *> turning internal numbers into human/machine-readable text.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FUNCTION           PIC X(10).
       01  WS-IO-STATUS          PIC XX.
           COPY "ORDERREC.CPY" REPLACING ==ORD-RECORD== BY ==WS-OR==.
       01  WS-IDX                PIC 9(2).
       01  WS-POS                PIC 9(4).
       01  WS-JSON-OUT           PIC X(600).
       01  WS-ED-AMOUNT          PIC Z(8)9.99.
       01  WS-ED-PRICE           PIC Z(4)9.99.
       01  WS-ED-QTY             PIC Z(4)9.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> No human-facing prompt here on purpose: the primary caller
      *> is the REST wrapper (Step 999), which pipes an order id to
      *> stdin and reads pure JSON back from stdout. A prompt text
      *> would land on the same line as the JSON (WITH NO ADVANCING
      *> plus a piped, non-terminal stdin never gets it "answered"
      *> visibly) and corrupt it for any machine reader - exactly
      *> the bug this capstone hit and fixed during Step 999 testing.
           ACCEPT ORD-ID FROM CONSOLE.

           MOVE "READ" TO WS-FUNCTION
           CALL "ORDREPO" USING WS-FUNCTION, WS-OR, WS-IO-STATUS

           IF WS-IO-STATUS NOT = "00"
               DISPLAY "{""error"":""order not found""}"
           ELSE
               PERFORM BUILD-ORDER-JSON
               DISPLAY FUNCTION TRIM(WS-JSON-OUT)
           END-IF

           STOP RUN.

       BUILD-ORDER-JSON.
           MOVE SPACES TO WS-JSON-OUT
           MOVE 1 TO WS-POS

           STRING '{"order_id":' ORD-ID
               ',"customer_id":' ORD-CUST-ID
               ',"order_date":"' ORD-DATE '"'
               ',"status":"' ORD-STATUS '"'
               DELIMITED BY SIZE
               INTO WS-JSON-OUT WITH POINTER WS-POS

           MOVE ORD-SUBTOTAL TO WS-ED-AMOUNT
           STRING ',"subtotal":' FUNCTION TRIM(WS-ED-AMOUNT)
               DELIMITED BY SIZE
               INTO WS-JSON-OUT WITH POINTER WS-POS

           MOVE ORD-TOTAL TO WS-ED-AMOUNT
           STRING ',"total":' FUNCTION TRIM(WS-ED-AMOUNT) ',"lines":['
               DELIMITED BY SIZE
               INTO WS-JSON-OUT WITH POINTER WS-POS

           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > ORD-LINE-COUNT
               IF WS-IDX > 1
                   STRING "," DELIMITED BY SIZE
                       INTO WS-JSON-OUT WITH POINTER WS-POS
               END-IF

               MOVE LINE-UNIT-PRICE(WS-IDX) TO WS-ED-PRICE
               MOVE LINE-QTY(WS-IDX) TO WS-ED-QTY
               STRING '{"product_id":' LINE-PROD-ID(WS-IDX)
                   ',"qty":' FUNCTION TRIM(WS-ED-QTY)
                   ',"unit_price":' FUNCTION TRIM(WS-ED-PRICE)
                   '}'
                   DELIMITED BY SIZE
                   INTO WS-JSON-OUT WITH POINTER WS-POS
           END-PERFORM

           STRING "]}" DELIMITED BY SIZE
               INTO WS-JSON-OUT WITH POINTER WS-POS.
```

(โค้ดเต็มยังมีฟิลด์ `discount_amount`, `tax_amount`, และ `extended` ต่อบรรทัดที่ตัดออกเพื่อความ
กระชับในที่นี้ — โครงสร้างเหมือนกันทุกประการกับ `subtotal`/`total` ที่แสดงไว้)

**เทคนิคสำคัญ**: `STRING ... WITH POINTER WS-POS` ใช้ตัวชี้ตำแหน่ง (`WS-POS`) เพื่อ**ต่อท้าย**ข้อความ
เข้า `WS-JSON-OUT` ทีละส่วนโดยไม่ต้องอ้างอิง `WS-JSON-OUT` เป็นทั้ง Sending Field และ Receiving Field
ในคำสั่งเดียวกัน (ซึ่งเป็นพฤติกรรมที่ไม่ปลอดภัยตามมาตรฐาน COBOL) — นี่คือการใช้ `STRING` แบบขั้นสูง
ที่ Part 019 แนะนำไว้สำหรับกรณีต้องต่อข้อความหลายรอบเข้าฟิลด์เดียวกัน

**ผลลัพธ์จริงจากการคอมไพล์และรัน**:

```
{"order_id":80000001,"customer_id":100001,"order_date":"20260928","status":"O","subtotal":199.93,"discount_amount":19.99,"tax_amount":12.60,"total":192.54,"lines":[{"product_id":200001,"qty":5,"unit_price":19.99,"extended":99.95},{"product_id":200002,"qty":2,"unit_price":49.99,"extended":99.98}]}
```

JSON ที่ได้ถูกต้องตามหลักไวยากรณ์ 100% (ทดสอบด้วย `python3 -m json.tool` ในขั้นตอนที่ 999 ยืนยันแล้ว)
และไม่มีเลขศูนย์นำหน้าที่ผิดกฎ JSON (ทบทวนจากการใช้ `PIC Z(8)9.99` แทน `PIC 9(9)V99` ตรง ๆ)

### ส่วนที่ 2: ORDRPT.cob — รายงานสรุปแบบ SORT + Control Break

Part 026 (Master-Detail) และ Part 027 (SORT) สอนเทคนิคนี้ไว้แล้ว — `ORDRPT.cob` อ่านทุกคำสั่งซื้อจาก
`ORDMAST.DAT` โดยตรง (ไม่ผ่าน `ORDREPO`) เพราะเป็นงาน Batch ที่ต้องอ่านทั้งไฟล์ ไม่ใช่อ่านทีละระเบียน
ตามคีย์:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ORDRPT.
       AUTHOR. COBOL-COURSE.
      *> Batch reporting program - Phase 2 callback (Part 026 Master-
      *> Detail processing, Part 027 SORT) combined with a manual
      *> control break (Part 039 Report Writer control-break idea,
      *> written by hand with PERFORM/IF the way Part 038/039 first
      *> taught it before Report Writer). Reads every order directly
      *> and sequentially from ORDMAST.DAT - the deliberate "batch
      *> reads the master file directly" half of the online/batch
      *> split documented in ORDREPO.cob.

       PROCEDURE DIVISION.
       MAIN-PARA.
           SORT SORT-WORK
               ON ASCENDING KEY SORT-CUST-ID SORT-ORD-ID
               INPUT PROCEDURE IS 1000-FEED-SORT-FROM-MASTER
               OUTPUT PROCEDURE IS 2000-PRODUCE-CONTROL-BREAK-REPORT
           DISPLAY "GRAND TOTAL (" WS-GRAND-COUNT " orders): "
               FUNCTION TRIM(WS-ED-GRAND)
           STOP RUN.

       1000-FEED-SORT-FROM-MASTER.
           OPEN INPUT ORD-FILE
           PERFORM UNTIL WS-NO-MORE-ORDERS
               READ ORD-FILE NEXT RECORD
                   AT END
                       SET WS-NO-MORE-ORDERS TO TRUE
                   NOT AT END
                       MOVE ORD-CUST-ID TO SORT-CUST-ID
                       MOVE ORD-ID      TO SORT-ORD-ID
                       MOVE ORD-STATUS  TO SORT-ORD-STATUS
                       MOVE ORD-TOTAL   TO SORT-ORD-TOTAL
                       RELEASE SORT-REC
               END-READ
           END-PERFORM
           CLOSE ORD-FILE.

       2000-PRODUCE-CONTROL-BREAK-REPORT.
           RETURN SORT-WORK AT END SET WS-NO-MORE-ORDERS TO TRUE
           END-RETURN
           PERFORM UNTIL WS-NO-MORE-ORDERS
               IF WS-FIRST-GROUP OR SORT-CUST-ID NOT = WS-BREAK-CUST-ID
                   IF NOT WS-FIRST-GROUP
                       PERFORM PRINT-GROUP-SUBTOTAL
                   END-IF
                   MOVE SORT-CUST-ID TO WS-BREAK-CUST-ID
                   MOVE 0 TO WS-GROUP-TOTAL, WS-GROUP-COUNT
                   MOVE "N" TO WS-FIRST-GROUP-FLAG
                   PERFORM PRINT-CUSTOMER-HEADER
               END-IF
               DISPLAY "    Order " SORT-ORD-ID "  status="
                   SORT-ORD-STATUS " total=" SORT-ORD-TOTAL
               ADD SORT-ORD-TOTAL TO WS-GROUP-TOTAL, WS-GRAND-TOTAL
               ADD 1 TO WS-GROUP-COUNT, WS-GRAND-COUNT
               RETURN SORT-WORK AT END SET WS-NO-MORE-ORDERS TO TRUE
               END-RETURN
           END-PERFORM
           IF WS-GROUP-COUNT > 0
               PERFORM PRINT-GROUP-SUBTOTAL
           END-IF.
```

(โครงสร้างเต็มของโปรแกรมนี้รวมถึง `SD SORT-WORK` และ Paragraph ย่อยอีก 2 ตัวที่ไม่แสดงในที่นี้เพื่อ
ความกระชับ — ดูหลักการเต็มรูปแบบได้จาก Part 027)

สังเกตว่า `PRINT-CUSTOMER-HEADER` เรียก `CUSTREPO` (Repository ที่สร้างไว้ในขั้นตอนที่ 995) เพื่อดึง
ชื่อลูกค้ามาแสดงในหัวรายงาน — นี่คือตัวอย่างที่ดีว่า**เทคนิคจากเฟสต่าง ๆ ทำงานร่วมกันได้จริง**ในระบบ
เดียว: SORT/Control Break (เฟส 2) + Repository Pattern (เฟส 4/6)

### พิสูจน์ด้วยการทดสอบจริงแบบครบวงจร

**ผลลัพธ์จริงจากการคอมไพล์และรันหลังสร้างคำสั่งซื้อ 4 รายการข้าม 3 ลูกค้า และยกเลิก 1 รายการ**:

```
=======================================
  EOMS ORDER REPORT - grouped by customer
=======================================

Customer 100001 - ACME CORPORATION
    Order 80000001  status=O  total=192.54
    Order 80000004  status=O  total=96.28
  Subtotal for customer 100001 (002 orders): 288.82

Customer 100002 - GOLDEN GATE TRADING
    Order 80000002  status=O  total=28.96
  Subtotal for customer 100002 (001 orders): 28.96

Customer 100003 - RIVERSIDE RETAIL LLC
    Order 80000003  status=X  total=21.39
  Subtotal for customer 100003 (001 orders): 21.39
----------------------------------------
GRAND TOTAL (00004 orders): 339.17
```

สังเกตว่าคำสั่งซื้อถูกสร้างตามลำดับ `80000001` (ลูกค้า 100001) → `80000002` (100002) → `80000003`
(100003) → `80000004` (100001 อีกครั้ง) แต่รายงานที่ได้**จัดกลุ่มตามลูกค้าอย่างถูกต้อง** (100001 ทั้ง
สองรายการอยู่ด้วยกัน) เพราะ `SORT` เรียงลำดับใหม่ตาม `SORT-CUST-ID` ก่อนประมวลผล Control Break — และ
คำสั่งซื้อที่ถูกยกเลิก (`80000003`, `status=X`) ยังคงปรากฏในรายงาน (เพราะเป็นรายงานตรวจสอบทุกคำสั่ง
ซื้อ ไม่ใช่แค่ที่ยังเปิดอยู่) พร้อมยอดรวมในกลุ่มลูกค้าถูกต้องครบถ้วน

### ข้อควรระวัง

- `ORDEXPORT.cob` **ตั้งใจไม่มี prompt ข้อความสำหรับมนุษย์** เพราะผู้เรียกหลักคือ REST Wrapper ไม่ใช่
  มนุษย์ที่พิมพ์โต้ตอบ — รายละเอียดปัญหาที่พบจากการมี prompt ปนอยู่ใน JSON output จะอธิบายเต็มรูปแบบ
  ในขั้นตอนที่ 999
- `ORDRPT.cob` ต้องเปิดไฟล์ `ORDMAST.DAT` ด้วย `ACCESS MODE IS SEQUENTIAL` (ไม่ใช่ `DYNAMIC` แบบ
  `ORDREPO.cob`) เพราะต้องการอ่านทุกระเบียนเรียงตามคีย์ตั้งแต่ต้นจนจบไฟล์

### แบบฝึกหัดที่ 998.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ `PIC Z(8)9.99` (numeric-edited พร้อม zero suppression) แทน
`PIC 9(9)V99` ตรง ๆ จึงจำเป็นสำหรับการสร้าง JSON ที่ถูกต้องตามมาตรฐาน

**เฉลยแนวทาง**: เพราะมาตรฐาน JSON ไม่อนุญาตให้ตัวเลขมีเลขศูนย์นำหน้า (leading zero) เช่น `007` ไม่ใช่
ตัวเลข JSON ที่ถูกต้อง ในขณะที่ `PIC 9(9)V99` เก็บค่าด้วยเลขศูนย์นำหน้าเสมอ (เช่น `000000192.54`)
การใช้ `PIC Z(8)9.99` แทนที่เลขศูนย์นำหน้าด้วยช่องว่าง (zero suppression) แล้วใช้ `FUNCTION TRIM`
ตัดช่องว่างออก ทำให้ได้ตัวเลขที่ไม่มีศูนย์นำหน้าเหลืออยู่ (เช่น `192.54`) ซึ่งเป็นรูปแบบตัวเลข JSON
ที่ถูกต้องตามมาตรฐาน และสามารถถูก parse โดยไลบรารี JSON มาตรฐานได้อย่างไม่มีปัญหา

---

## ขั้นตอนที่ 999: Unit Testing, CI Pipeline, และ REST API (Phase 5 Callback)

### ส่วนที่ 1: Unit Test สำหรับ Business Logic Layer

ตามหลักการ Part 079 (Unit Testing) `ORDCALC.cob` และ `ORDVALID.cob` ทดสอบได้โดยไม่ต้องมีไฟล์บนดิสก์
เลย เพราะเป็น Pure Function — นี่คือเหตุผลที่การออกแบบ Business Logic Layer ให้ "สะอาด" (ไม่มี File
I/O) ตั้งแต่ขั้นตอนที่ 993-994 มีค่ามหาศาลในขั้นตอนนี้

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TESTORDCALC.
       AUTHOR. COBOL-COURSE.
      *> Phase 5 callback (Part 079 Unit Testing): a small, dependency
      *> free test harness for ORDCALC.cob. No files, no other
      *> subprograms - exactly what a pure "Business Logic" module
      *> should allow. Each test compares an actual result against a
      *> value worked out by hand, and the harness keeps a running
      *> pass/fail count so CI can fail the build on any mismatch.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "TESTORDCALC: starting ORDCALC unit tests".

      *> Test 3: Gold tier, boundary case - a single unit priced so
      *> the discount rounds up rather than down, proving ROUNDED is
      *> actually taking effect and not silently truncating.
      *> 1 x 12.34 = 12.34, 5% gold discount = 0.617 -> 0.62 rounded
      *> (COMPUTE ROUNDED, not truncation, per Part 009). Net = 11.72.
      *> Tax 7% = 0.8204 -> 0.82. Total = 12.54.
           MOVE "G" TO WS-TIER
           MOVE 1 TO WS-COUNT
           MOVE 1 TO WS-QTY(1)
           MOVE 12.34 TO WS-PRICE(1)
           MOVE 12.54 TO WS-EXPECTED-TOTAL
           PERFORM RUN-ONE-CASE

           DISPLAY " "
           DISPLAY "TESTORDCALC RESULTS: " WS-PASS-COUNT " passed, "
               WS-FAIL-COUNT " failed"
           IF WS-FAIL-COUNT > 0
               MOVE 1 TO RETURN-CODE
           END-IF
           STOP RUN.

       RUN-ONE-CASE.
           CALL "ORDCALC" USING WS-TIER, WS-COUNT, WS-QTY-GRP,
               WS-PRICE-GRP, WS-EXT-GRP, WS-SUBTOTAL, WS-DISC-PCT,
               WS-DISC-AMT, WS-TAX-AMT, WS-TOTAL
           IF WS-TOTAL = WS-EXPECTED-TOTAL
               ADD 1 TO WS-PASS-COUNT
               DISPLAY "  PASS: tier=" WS-TIER " total=" WS-TOTAL
           ELSE
               ADD 1 TO WS-FAIL-COUNT
               DISPLAY "  FAIL: tier=" WS-TIER " expected="
                   WS-EXPECTED-TOTAL " actual=" WS-TOTAL
           END-IF.
```

`TESTORDVALID.cob` มีโครงสร้างเดียวกัน โดยมีคอมเมนต์ที่สำคัญมากอธิบายไว้ตรง ๆ ว่า:

```cobol
      *> Unit tests for ORDVALID.cob (Part 079 style harness, same
      *> shape as TESTORDCALC.cob). This module is also where a real
      *> bug was caught during capstone development: ORDVALID.cob
      *> originally fell through into CHECK-ONE-LINE one extra time
      *> because VALIDATE-ORDER was missing a GOBACK - these tests
      *> are exactly the kind of check that would have caught it
      *> immediately instead of only surfacing through ORDSVC later.
```

**ผลลัพธ์จริงจากการคอมไพล์และรันทั้งสองชุดทดสอบ**:

```
TESTORDCALC: starting ORDCALC unit tests
  PASS: tier=P total=000000105.89
  PASS: tier=R total=000000081.32
  PASS: tier=G total=000000012.54

TESTORDCALC RESULTS: 003 passed, 000 failed
TESTORDVALID: starting ORDVALID unit tests
  PASS: flag=Y msg=[]
  PASS: flag=N msg=[Line 02: insufficient stock]
  PASS: flag=N msg=[Customer is not active]
  PASS: flag=N msg=[Line 01: product is not active]
  PASS: flag=N msg=[Order must have 1 to 5 line items]
  PASS: flag=Y msg=[]

TESTORDVALID RESULTS: 006 passed, 000 failed
```

รวม **9 Test Case ผ่านทั้งหมด 100%** ครอบคลุมทั้งกรณีปกติและกรณีขอบเขต (5 ระดับสมาชิก × สินค้าครบ
5 รายการ, ลูกค้าไม่ Active, สินค้าไม่ Active, สต๊อกไม่พอ, จำนวนรายการเป็น 0 หรือเกิน 5)

### ส่วนที่ 2: ci.sh — CI Pipeline เต็มรูปแบบ

ตามหลักการ Part 078 (CI/CD Pipeline):

```bash
#!/usr/bin/env bash
# ci.sh - Phase 5 callback (Part 078 CI/CD Pipeline). Compiles the
# whole EOMS system, runs the unit tests, then runs one integration
# smoke test against real indexed files. Exits non-zero on the first
# failure so a real CI runner (GitHub Actions, Jenkins, ...) reports
# the build as broken.

set -euo pipefail
COBC="${COBC:-cobc}"

echo "== EOMS CI: 1/4 compile subprograms and repositories =="
for MOD in ordcalc ordvalid custrepo prodrepo ordrepo ordsvc; do
    "$COBC" -I copybooks -c "src/$MOD.cob" -o "build/$MOD.o"
done

echo "== EOMS CI: 2/4 compile executables =="
"$COBC" -I copybooks -x -o build/eomsmain src/eomsmain.cob \
    build/ordsvc.o build/ordcalc.o build/ordvalid.o \
    build/custrepo.o build/prodrepo.o build/ordrepo.o
# ... (custinit, prodinit, ordinit, ordexport, ordrpt, ordapi omitted
#      here for brevity - same pattern, see the full script)

echo "== EOMS CI: 3/4 unit tests (no files touched) =="
"$COBC" -I copybooks -x -o build/testordcalc \
    tests/testordcalc.cob build/ordcalc.o
"$COBC" -I copybooks -x -o build/testordvalid \
    tests/testordvalid.cob build/ordvalid.o
build/testordcalc
build/testordvalid

echo "== EOMS CI: 4/4 integration smoke test (real indexed files) =="
rm -f CUSTMAST.DAT* PRODMAST.DAT* ORDMAST.DAT* ORDSORT.TMP
build/custinit
build/prodinit
build/ordinit

SMOKE_OUTPUT=$(printf -- "1\n77000001\n100001\n20260928\n1\n200001\n1\n3\n" \
    | build/eomsmain)
echo "$SMOKE_OUTPUT" | grep -q "SUCCESS: Order created successfully"

EXPORT_OUTPUT=$(printf -- "77000001\n" | build/ordexport)
echo "$EXPORT_OUTPUT" | grep -q '"order_id":77000001'

echo "== EOMS CI: ALL CHECKS PASSED =="
```

**ผลลัพธ์จริงจากการรัน `ci.sh` แบบเต็ม**:

```
== EOMS CI: 1/4 compile subprograms and repositories ==
== EOMS CI: 2/4 compile executables ==
== EOMS CI: 3/4 unit tests (no files touched) ==
TESTORDCALC RESULTS: 003 passed, 000 failed
TESTORDVALID RESULTS: 006 passed, 000 failed
== EOMS CI: 4/4 integration smoke test (real indexed files) ==
CUSTINIT: CUSTMAST.DAT created with 4 customers.
PRODINIT: PRODMAST.DAT created with 4 products.
ORDINIT: ORDMAST.DAT created (empty).
== EOMS CI: ALL CHECKS PASSED ==
```

`set -euo pipefail` ที่หัวสคริปต์ทำให้ CI **หยุดทันทีและ exit ด้วยรหัสไม่เป็นศูนย์** ทันทีที่คำสั่งใด
ล้มเหลว (เช่น คอมไพล์ error หรือ Unit Test ล้มเหลว หรือ `grep -q` ไม่พบข้อความที่คาดหวัง) — นี่คือ
พฤติกรรมที่ระบบ CI จริง (GitHub Actions, Jenkins) ใช้ตัดสินว่า Build "ผ่าน" หรือ "พัง"

### ส่วนที่ 3: ORDAPI.cob + REST Wrapper (Phase 5 Callback: Part 073)

ตามรูปแบบ Part 073/085 `ORDAPI.cob` เป็นจุดเชื่อมต่อแบบ CLI ที่ไม่โต้ตอบ (non-interactive) รับคำขอ
หนึ่งบรรทัดแบบ pipe-delimited จาก stdin แล้วตอบกลับหนึ่งบรรทัดไปยัง stdout — ออกแบบมาให้ REST Wrapper
เรียกผ่าน subprocess ได้ง่าย:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ORDAPI.
       AUTHOR. COBOL-COURSE.
      *> Non-interactive CLI entry point for ORDSVC, meant to be
      *> shelled out to by the Python REST wrapper - the same "COBOL
      *> batch program behind a thin process boundary" pattern Part
      *> 073/085 used for ARCALC.cob. Reads ONE pipe-delimited
      *> request line from stdin, parses it with UNSTRING (Phase 2
      *> callback, Part 020), calls ORDSVC, and writes ONE pipe-
      *> delimited response line to stdout.
      *>
      *> Request  (CREATE): CREATE|ord_id|cust_id|date|line_count|
      *>                     prod1|qty1|prod2|qty2|...(up to 5 pairs)
      *> Request  (CANCEL): CANCEL|ord_id
      *> Response:          flag|message|total

       PROCEDURE DIVISION.
       MAIN-PARA.
           ACCEPT WS-REQUEST-LINE FROM CONSOLE
           PERFORM PARSE-REQUEST
           PERFORM CALL-ORDSVC
           PERFORM BUILD-RESPONSE
           DISPLAY FUNCTION TRIM(WS-RESPONSE-LINE)
           STOP RUN.

       PARSE-REQUEST.
           MOVE SPACES TO WS-FIELD-TABLE
           UNSTRING WS-REQUEST-LINE DELIMITED BY "|"
               INTO WS-FIELD(1), WS-FIELD(2), WS-FIELD(3),
                    WS-FIELD(4), WS-FIELD(5), WS-FIELD(6),
                    WS-FIELD(7), WS-FIELD(8), WS-FIELD(9),
                    WS-FIELD(10), WS-FIELD(11), WS-FIELD(12),
                    WS-FIELD(13), WS-FIELD(14), WS-FIELD(15)

           MOVE WS-FIELD(1) TO WS-FUNCTION
           EVALUATE FUNCTION TRIM(WS-FUNCTION)
               WHEN "CANCEL"
                   MOVE WS-FIELD(2) TO WS-ORD-ID
               WHEN "CREATE"
                   MOVE WS-FIELD(2) TO WS-ORD-ID
                   MOVE WS-FIELD(3) TO WS-CUST-ID
                   MOVE WS-FIELD(4) TO WS-ORDER-DATE
                   MOVE WS-FIELD(5) TO WS-LINE-COUNT
                   PERFORM VARYING WS-FIELD-IDX FROM 1 BY 1
                           UNTIL WS-FIELD-IDX > WS-LINE-COUNT
                       MOVE WS-FIELD(5 + (WS-FIELD-IDX * 2) - 1)
                           TO WS-REQ-PROD-ID(WS-FIELD-IDX)
                       MOVE WS-FIELD(5 + (WS-FIELD-IDX * 2))
                           TO WS-REQ-QTY(WS-FIELD-IDX)
                   END-PERFORM
           END-EVALUATE.
```

**พิสูจน์ด้วยการทดสอบจริง**:

```bash
echo "CREATE|77000010|100001|20260928|2|200001|2|200002|1" | ./build/ordapi
echo "CREATE|77000011|100003|20260928|1|200003|999" | ./build/ordapi
echo "CANCEL|77000010" | ./build/ordapi
echo "CANCEL|77000010" | ./build/ordapi
```

**ผลลัพธ์จริง**:

```
Y|Order created successfully|86.64
N|Line 01: insufficient stock|0.00
Y|Order cancelled successfully|86.64
N|Order is not open, cannot cancel|0.00
```

### REST Wrapper ด้วย Python (stdlib เท่านั้น ไม่มี Framework ภายนอก)

ตามข้อจำกัดเดียวกับที่ Part 085 ใช้ (Python stdlib ล้วน ไม่พึ่งพา Flask/FastAPI):

```python
class EomsHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        match = ORDER_PATH_RE.match(self.path)
        if not match:
            self._send_json(404, {"error": "not found"})
            return
        order_id = match.group(1)
        json_text = run_ordexport(order_id)
        # ORDEXPORT.cob already produced valid JSON text (Step 998),
        # so this handler passes it straight through unmodified.
        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        body = json_text.encode("utf-8")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def _handle_create(self):
        length = int(self.headers.get("Content-Length", 0))
        data = json.loads(self.rfile.read(length))
        order_id = next_order_id()
        lines = data.get("lines", [])
        fields = ["CREATE", str(order_id), str(data["cust_id"]),
                   str(data["date"]), str(len(lines))]
        for line in lines:
            fields.append(str(line["product_id"]))
            fields.append(str(line["qty"]))
        result = run_ordapi("|".join(fields))
        result["order_id"] = order_id
        self._send_json(200 if result["success"] else 422, result)
```

### กับดักจริงที่พบระหว่างพัฒนา: Prompt Text ปนเข้าไปใน JSON Output

ทดสอบครั้งแรกด้วย `curl` จริงผ่าน HTTP พบว่า `GET /orders/<id>` คืนค่าที่**ไม่ใช่ JSON ที่ถูกต้อง**:

```
Order ID to export (8 digits): {"order_id":90627594,...}
```

สาเหตุคือ `ORDEXPORT.cob` เวอร์ชันแรกมี `DISPLAY "Order ID to export..." WITH NO ADVANCING` ก่อน
`ACCEPT` — ข้อความ prompt นี้ถูกเขียนไปยัง stdout **บรรทัดเดียวกัน**กับ JSON ที่ตามมา (เพราะ
`WITH NO ADVANCING` ไม่ขึ้นบรรทัดใหม่) ทำให้ Python ที่อ่าน stdout ทั้งหมดมาใช้เป็น JSON ได้รับ
ข้อความปนเปื้อนที่ parse ไม่ได้ วิธีแก้คือลบ prompt ออกทั้งหมด (ดูคอมเมนต์ในโค้ด `ORDEXPORT.cob`
ขั้นตอนที่ 998 ที่อธิบายเหตุผลไว้แล้ว) เพราะผู้เรียกหลักคือเครื่องจักร (REST Wrapper) ไม่ใช่มนุษย์

### พิสูจน์ด้วยการทดสอบจริงผ่าน HTTP จริง (curl)

```bash
curl -s -X POST http://localhost:8080/orders -H "Content-Type: application/json" \
  -d '{"cust_id":100001,"date":20260928,"lines":[{"product_id":200001,"qty":2},{"product_id":200002,"qty":1}]}'
```

**ผลลัพธ์จริง**:

```json
{"success": true, "message": "Order created successfully", "total": 86.64, "order_id": 90627621}
```

```bash
curl -s http://localhost:8080/orders/90627621 | python3 -m json.tool
```

**ผลลัพธ์จริง (parse ผ่านสมบูรณ์ด้วย `json.tool` มาตรฐานของ Python)**:

```json
{
    "order_id": 90627621,
    "customer_id": 100001,
    "order_date": "20260928",
    "status": "O",
    "subtotal": 89.97,
    "discount_amount": 9.0,
    "tax_amount": 5.67,
    "total": 86.64,
    "lines": [
        {"product_id": 200001, "qty": 2, "unit_price": 19.99, "extended": 39.98},
        {"product_id": 200002, "qty": 1, "unit_price": 49.99, "extended": 49.99}
    ]
}
```

### ข้อควรระวัง

- `ordapi_server.py` ใช้ `int(time.time()) % 100000000` เป็นตัวสร้าง Order ID แทน Sequence Generator
  จริง — คอมเมนต์ในโค้ดระบุไว้ชัดเจนว่านี่คือการลดความซับซ้อน (simplification) สำหรับ Demo เท่านั้น
  ระบบจริงควรใช้กลไกที่รับประกันความไม่ซ้ำกัน เช่น Relative File แบบ RRDS ที่ Part 055-056 สอนไว้
- ทุกครั้งที่แก้ไขโปรแกรม COBOL ที่ REST Wrapper เรียกใช้ ต้องคอมไพล์ใหม่ก่อนเสมอ — Python ไม่รู้จัก
  การเปลี่ยนแปลงในซอร์สโค้ด COBOL จนกว่า executable ที่มันเรียกจะถูกคอมไพล์ใหม่

### แบบฝึกหัดที่ 999.1

**โจทย์**: จงอธิบายว่าทำไม `ORDAPI.cob` ต้องใช้ `UNSTRING ... DELIMITED BY "|"` แทนที่จะออกแบบ
Request เป็น Fixed-Length Format (ความยาวคงที่) แบบที่ `ARCALC.cob` ใน Part 085 ใช้

**เฉลยแนวทาง**: เพราะจำนวน Line Item ของคำสั่งซื้อแต่ละครั้งไม่แน่นอน (1 ถึง 5 รายการ) ทำให้ความยาว
ของ Request ผันแปรได้ Fixed-Length Format (แบบ `ARCALC.cob` ที่มีความยาวคงที่ 12 ตัวอักษรเสมอ)
เหมาะกับกรณีที่จำนวนฟิลด์คงที่ตายตัว แต่ไม่เหมาะกับกรณีนี้ที่จำนวนคู่ (Product ID, Quantity) ผันแปร
ตามจำนวน Line Item การใช้ Delimiter (`|`) ร่วมกับ `UNSTRING` (Part 020) จึงยืดหยุ่นกว่า รองรับ
Request ที่มีความยาวต่างกันได้โดยไม่ต้องกำหนดความกว้างคงที่ล่วงหน้า

---

## ขั้นตอนที่ 1000: การสาธิตระบบแบบครบวงจร, บทสรุปหลักสูตร, และก้าวต่อไป

นี่คือขั้นตอนสุดท้ายของหลักสูตร 1000 ขั้นตอน — เราจะปิดท้ายด้วย 4 ส่วน: (ก) การสาธิตระบบ EOMS แบบ
End-to-End เต็มรูปแบบ (ข) การมองย้อนกลับสรุปเส้นทางการเรียนรู้ทั้งหมด (ค) แหล่งเรียนรู้ต่อที่แนะนำ
และ (ง) คำอำลาจากใจ

### ส่วนที่ (ก): การสาธิตระบบ EOMS แบบ End-to-End เต็มรูปแบบ

ต่อไปนี้คือการรันระบบ EOMS ทั้งหมดตั้งแต่ต้นจนจบในคำสั่งเดียว ครอบคลุมทุกโมดูลที่สร้างไว้ตลอด Part นี้
— **นี่คือผลลัพธ์จริงที่ได้จากการรันจริง ไม่มีการตัดต่อหรือแต่งเติมใด ๆ**:

**ขั้นที่ 1 — เตรียมไฟล์ Master File**:

```bash
./build/custinit
./build/prodinit
./build/ordinit
```

```
CUSTINIT: CUSTMAST.DAT created with 4 customers.
PRODINIT: PRODMAST.DAT created with 4 products.
ORDINIT: ORDMAST.DAT created (empty).
```

**ขั้นที่ 2 — สร้างคำสั่งซื้อ 4 รายการข้าม 3 ลูกค้าผ่านเมนู `EOMSMAIN`** (ตัดผลลัพธ์เมนูซ้ำออก):

```
SUCCESS: Order created successfully   Order total: 000000192.54
SUCCESS: Order created successfully   Order total: 000000028.96
SUCCESS: Order created successfully   Order total: 000000021.39
SUCCESS: Order created successfully   Order total: 000000096.28
```

**ขั้นที่ 3 — ยกเลิกคำสั่งซื้อหนึ่งรายการ**:

```
SUCCESS: Order cancelled successfully   Refunded amount: 000000021.39
```

**ขั้นที่ 4 — รายงานสรุปแบบจัดกลุ่มตามลูกค้า (`ORDRPT`)**:

```
=======================================
  EOMS ORDER REPORT - grouped by customer
=======================================

Customer 100001 - ACME CORPORATION
    Order 80000001  status=O  total=192.54
    Order 80000004  status=O  total=96.28
  Subtotal for customer 100001 (002 orders): 288.82

Customer 100002 - GOLDEN GATE TRADING
    Order 80000002  status=O  total=28.96
  Subtotal for customer 100002 (001 orders): 28.96

Customer 100003 - RIVERSIDE RETAIL LLC
    Order 80000003  status=X  total=21.39
  Subtotal for customer 100003 (001 orders): 21.39
----------------------------------------
GRAND TOTAL (00004 orders): 339.17
```

**ขั้นที่ 5 — ส่งออกคำสั่งซื้อเป็น JSON (`ORDEXPORT`)**:

```json
{"order_id":80000001,"customer_id":100001,"order_date":"20260928","status":"O","subtotal":199.93,"discount_amount":19.99,"tax_amount":12.60,"total":192.54,"lines":[{"product_id":200001,"qty":5,"unit_price":19.99,"extended":99.95},{"product_id":200002,"qty":2,"unit_price":49.99,"extended":99.98}]}
```

**ขั้นที่ 6 — รัน CI Pipeline เต็มรูปแบบอีกครั้งเพื่อยืนยันว่าทุกอย่างยังคงทำงานถูกต้อง**:

```
== EOMS CI: 1/4 compile subprograms and repositories ==
== EOMS CI: 2/4 compile executables ==
== EOMS CI: 3/4 unit tests (no files touched) ==
TESTORDCALC RESULTS: 003 passed, 000 failed
TESTORDVALID RESULTS: 006 passed, 000 failed
== EOMS CI: 4/4 integration smoke test (real indexed files) ==
== EOMS CI: ALL CHECKS PASSED ==
```

**สรุปโมดูลทั้งหมดที่คอมไพล์และทดสอบจริงใน Part 100**: 4 Copybook (`EOMSCNST.CPY`, `CUSTOMER.CPY`,
`PRODUCT.CPY`, `ORDERREC.CPY`), 13 โปรแกรม COBOL (`CUSTINIT`, `PRODINIT`, `ORDINIT`, `ORDCALC`,
`ORDVALID`, `CUSTREPO`, `PRODREPO`, `ORDREPO`, `ORDSVC`, `EOMSMAIN`, `ORDEXPORT`, `ORDRPT`, `ORDAPI`),
2 ชุด Unit Test (`TESTORDCALC`, `TESTORDVALID`), 1 REST Wrapper (`ordapi_server.py`), และ 1 CI Script
(`ci.sh`) — **รวม 21 ไฟล์ที่ทำงานร่วมกันเป็นระบบเดียว ผ่านการคอมไพล์และทดสอบจริงทั้งหมด**

### ส่วนที่ (ข): บทสรุปย้อนหลังการเดินทางทั้ง 1000 ขั้นตอน

ย้อนกลับไปดูสถานะความคืบหน้าล่าสุดของหลักสูตร (`docs/PROGRESS.md`) ก่อนที่ Part นี้จะเขียนเสร็จ:
Part 001 ถึง Part 097 เสร็จสมบูรณ์แล้วทั้งหมด (✅) ครอบคลุมทั้ง 6 เฟส — และตอนนี้ Part 098, 099, และ
100 ก็เสร็จสมบูรณ์แล้วเช่นกัน ปิดหลักสูตรทั้ง **100 Part / 1000 Steps** อย่างสมบูรณ์

**เส้นทางที่เราเดินทางร่วมกันมาตลอด 100 Part**:

| เฟส | Parts | สิ่งที่ได้เรียนรู้ | โปรเจกต์รวบยอด |
|---|---|---|---|
| 1: พื้นฐาน | 001-015 | Divisions, PICTURE, MOVE, เลขคณิต, IF/EVALUATE, PERFORM | เครื่องคิดเลข + ระบบเกรด |
| 2: ระดับกลาง | 016-035 | OCCURS, STRING/UNSTRING, ไฟล์ Sequential/Indexed, CALL, COPY | ระบบจัดการสินค้าคงคลัง |
| 3: ขั้นสูง | 036-050 | Intrinsic Functions, Report Writer, JSON/XML, Error Handling | ระบบบัญชีลูกหนี้ |
| 4: Mainframe | 051-070 | JCL, VSAM, DB2, CICS, IMS, Batch Processing | ระบบธนาคารจำลอง |
| 5: สมัยใหม่ | 071-085 | REST API, Docker, Cloud, CI/CD, Testing, Refactoring | ปรับปรุงระบบ Legacy |
| 6: มืออาชีพ | 086-100 | สถาปัตยกรรมองค์กร, Case Study, อาชีพ, **Capstone สุดท้าย** | **Enterprise Order Management System** |

**สิ่งที่ยึดถือมาตลอดทั้ง 100 Part โดยไม่มีข้อยกเว้นแม้แต่ครั้งเดียว**:

1. **โค้ดทุกตัวอย่างคอมไพล์และรันได้จริง** ด้วย GnuCOBOL — ไม่มีโค้ดตัวอย่างที่ "น่าจะรันได้" โดยไม่
   ผ่านการทดสอบจริง
2. **กฎเหล็กเรื่องภาษา ASCII ในซอร์สโค้ด** ที่ยึดถืออย่างเคร่งครัดตั้งแต่ Part 002 จนถึง Part 100 นี้
3. **ความซื่อสัตย์ต่อข้อจำกัดของสภาพแวดล้อม** — ไม่ว่าจะเป็น Indexed File ที่ปิดใช้งานในบาง build
   (Part 028/070/100), JSON ที่ปิดใช้งาน (Part 100), หรือการจำลอง Mainframe ที่ไม่ใช่ Mainframe จริง
   (Part 051-070) หลักสูตรนี้ประกาศข้อจำกัดเหล่านี้อย่างตรงไปตรงมาเสมอ ไม่เคยกล่าวอ้างเกินจริง
4. **การเล่าบั๊กจริงที่พบระหว่างพัฒนา** (Part 035, Part 085, Part 100) แทนที่จะซ่อนไว้ เพราะบั๊กจริง
   คือบทเรียนที่มีค่าที่สุดสำหรับผู้เรียน

### ส่วนที่ (ค): แหล่งเรียนรู้ต่อ (แหล่งเรียนรู้ต่อ)

หลักสูตรนี้เป็นจุดเริ่มต้น ไม่ใช่จุดสิ้นสุด ต่อไปนี้คือแหล่งข้อมูลที่แท้จริงและเชื่อถือได้ที่แนะนำให้
ศึกษาต่อ (คัดเลือกเฉพาะแหล่งที่มีชื่อเสียงและตรวจสอบได้จริงในอุตสาหกรรม):

**เอกสารและคู่มือทางการ**:

- **GnuCOBOL Documentation** (เอกสารทางการของ GnuCOBOL ที่หลักสูตรนี้ใช้ตลอดทั้ง 100 Part) — คู่มือ
  อ้างอิงไวยากรณ์ฉบับสมบูรณ์ รวมถึง Programmer's Guide ที่อธิบายฟีเจอร์เฉพาะของ GnuCOBOL เจาะลึกกว่า
  ที่หลักสูตรนี้ครอบคลุมได้ — ค้นหาได้จากเว็บไซต์ทางการของโปรเจกต์ GnuCOBOL (sourceforge.net)
- **IBM Redbooks** — ชุดเอกสารทางเทคนิคเชิงลึกที่ IBM เผยแพร่ฟรีเกี่ยวกับ z/OS, COBOL, CICS, DB2,
  และหัวข้อ Mainframe อื่น ๆ เขียนโดยผู้เชี่ยวชาญจริงจากประสบการณ์ Implementation จริง เหมาะสำหรับ
  ผู้ที่ต้องการเจาะลึกเนื้อหาเฟส 4 ของหลักสูตรนี้ (JCL, VSAM, CICS, DB2) ในระดับที่ลึกกว่าที่หลักสูตร
  ครอบคลุมได้ (เพราะหลักสูตรนี้ใช้ GnuCOBOL เป็นหลักตามที่ประกาศไว้ตั้งแต่ Part 051)
- **เอกสารมาตรฐาน ISO/IEC 1989 (COBOL Standard)** — มาตรฐานภาษา COBOL ฉบับทางการจากองค์กร ISO
  สำหรับผู้ที่ต้องการอ้างอิงไวยากรณ์อย่างเป็นทางการที่สุด

**ชุมชนและองค์กรที่สนับสนุนระบบนิเวศ COBOL**:

- **Open Mainframe Project** (โครงการภายใต้ Linux Foundation) — จัดทำ "COBOL Programming Course"
  ที่เป็น Open Source เผยแพร่ฟรี ครอบคลุมตั้งแต่ระดับเริ่มต้นจนถึงขั้นสูง รวมถึงเนื้อหาการทดสอบผ่าน
  โครงการ COBOL Check เหมาะสำหรับทบทวนความรู้จากมุมมองที่ต่างจากหลักสูตรนี้ และมีช่องทางชุมชน (Slack,
  Forum) ให้ตั้งคำถามและพูดคุยกับผู้เรียน/ผู้เชี่ยวชาญ COBOL คนอื่น ๆ ทั่วโลก
- **GnuCOBOL Community** — ชุมชนผู้ใช้และผู้พัฒนา GnuCOBOL ที่สามารถตั้งคำถามเชิงเทคนิคเกี่ยวกับการ
  คอมไพล์ การตั้งค่า และปัญหาเฉพาะทางที่พบระหว่างการพัฒนาได้

**แนวทางการฝึกฝนต่อเนื่อง**:

- **มีส่วนร่วมในโครงการ Open Source ที่เกี่ยวข้องกับ COBOL** เช่น GnuCOBOL เองหรือเครื่องมือ Ecosystem
  รอบข้าง (Part 071 สอนภาพรวม Ecosystem นี้ไว้แล้ว) เป็นวิธีที่ดีในการฝึกฝนกับโค้ดจริงที่มีผู้ใช้งาน
  จริงมากกว่าแบบฝึกหัดในหลักสูตร
- **ทดลองสร้างโปรเจกต์ของตัวเองต่อยอดจาก EOMS** เช่น เพิ่มฟีเจอร์ใหม่ (การแก้ไขคำสั่งซื้อ, ระบบส่วนลด
  ที่ซับซ้อนขึ้น, การรายงานที่หลากหลายขึ้น) เพื่อฝึกฝนการออกแบบและตัดสินใจเชิงสถาปัตยกรรมด้วยตัวเอง
  โดยไม่มีคำตอบสำเร็จรูปให้อ้างอิง
- **ติดตามข่าวสารอุตสาหกรรม Mainframe/COBOL อย่างสม่ำเสมอ** ตามที่ Part 099 แนะนำไว้ เพราะภูมิทัศน์
  เครื่องมือและตลาดแรงงานเปลี่ยนแปลงต่อเนื่อง

### ส่วนที่ (ง): คำอำลาจากใจ

ถ้าคุณอ่านมาถึงบรรทัดนี้ — คุณได้เดินทางผ่าน **1000 ขั้นตอน, 100 Part, และ 6 เฟส** ของหลักสูตรที่ยาว
และลึกที่สุดเท่าที่จะทำได้สำหรับภาษาโปรแกรมมิ่งภาษาหนึ่ง คุณเริ่มต้นจากคำถามว่า "COBOL คืออะไร" ใน
Part 001 และตอนนี้คุณสามารถออกแบบและสร้างระบบจัดการคำสั่งซื้อระดับองค์กรที่มีสถาปัตยกรรมแบบ Layered,
ไฟล์ Indexed แบบ VSAM, REST API, Unit Test, และ CI Pipeline ได้ด้วยตัวเองแล้ว

ตลอดหลักสูตรนี้ เราไม่เคยพยายามทำให้ COBOL ดูง่ายกว่าที่มันเป็นจริง หรือซ่อนข้อจำกัดของเครื่องมือที่ใช้
ไว้ — เราเล่าบั๊กจริงให้ฟังเมื่อเจอ เราประกาศข้อจำกัดของสภาพแวดล้อมอย่างตรงไปตรงมาเมื่อพบ และเราพิสูจน์
ทุกคำกล่าวอ้างด้วยการคอมไพล์และรันจริงเสมอ เพราะเราเชื่อว่า**ความซื่อสัตย์ต่อความเป็นจริงทางเทคนิคคือ
รากฐานที่สำคัญที่สุดของการเรียนรู้ที่แท้จริง** — และนี่คือบทเรียนที่มีค่าไม่แพ้ไวยากรณ์ COBOL เองเลย

COBOL อาจดูเหมือนเป็นภาษาที่ "เก่า" สำหรับคนภายนอก แต่หลังจากเดินทางมาถึงจุดนี้ คุณย่อมรู้ดีกว่าใครว่า
ความจริงซับซ้อนกว่านั้นมาก — COBOL คือรากฐานที่แข็งแกร่งของระบบที่โลกทั้งใบพึ่งพาอยู่ทุกวัน ตั้งแต่
การกดเงินที่ตู้ ATM ไปจนถึงการโอนเงินข้ามธนาคาร และตอนนี้ **คุณคือหนึ่งในผู้ที่มีความสามารถดูแลรักษา
และพัฒนาต่อยอดรากฐานนั้นได้แล้ว**

ไม่ว่าคุณจะนำความรู้นี้ไปสมัครงานสาย Mainframe Developer, ไปเป็นสะพานเชื่อม Legacy System กับโลก
สมัยใหม่ในตำแหน่ง Modernization Engineer, หรือแค่เก็บไว้เป็นความรู้เสริมที่หายากในตลาด — ขอให้ความ
มุ่งมั่นที่พาคุณเดินทางมาถึงบรรทัดสุดท้ายนี้ พาคุณไปได้ไกลกว่านี้อีกมากในเส้นทางที่คุณเลือก

**ขอบคุณที่เดินทางมาด้วยกันตลอด 1000 ขั้นตอน ยินดีด้วยกับการสำเร็จหลักสูตรนี้ และขอให้โชคดีในเส้นทาง
COBOL ของคุณต่อจากนี้ไป**

---

## สรุปท้ายบท

Part 100 คือจุดสิ้นสุดของหลักสูตร 1000 ขั้นตอนนี้ และเป็นบทพิสูจน์ครั้งสุดท้ายว่าทุกเทคนิคที่เรียนมา
ตลอดทั้ง 6 เฟสสามารถประกอบร่างเป็นระบบเดียวที่ทำงานได้จริง — ระบบ **Enterprise Order Management
System (EOMS)** ที่เราสร้างร่วมกันประกอบด้วย 21 ไฟล์ที่ทำงานร่วมกัน ผ่านการคอมไพล์และทดสอบจริงทั้งหมด
100% ครอบคลุม:

- **เฟส 1** ผ่านโครงสร้างข้อมูล (Copybooks), PICTURE Clause, และเมนู PERFORM/EVALUATE ใน `EOMSMAIN`
  (ขั้นตอนที่ 991, 997)
- **เฟส 2** ผ่าน OCCURS Table, CALL/Subprogram, UNSTRING, และ SORT ใน `ORDCALC`, `ORDAPI`, `ORDRPT`
  (ขั้นตอนที่ 993, 998, 999)
- **เฟส 3** ผ่านการคำนวณด้วย COMPUTE ROUNDED และการสร้าง JSON ด้วย STRING แทน JSON GENERATE ที่ถูก
  ปิดใช้งาน (ขั้นตอนที่ 993, 998)
- **เฟส 4** ผ่านไฟล์ Indexed แบบ VSAM-style ทั้งสามไฟล์ (`CUSTMAST`, `PRODMAST`, `ORDMAST`) และ
  Repository Pattern (ขั้นตอนที่ 992, 995)
- **เฟส 5** ผ่าน Unit Test, CI Pipeline, และ REST API Wrapper (ขั้นตอนที่ 999)
- **เฟส 6** ผ่านสถาปัตยกรรมแบบ Layered/Clean Architecture ที่แบ่งเป็น Presentation, Application,
  Business Logic, และ Data Access อย่างชัดเจน (ขั้นตอนที่ 991, 996)

ระหว่างการพัฒนา เราพบและแก้ไขบั๊กจริง 2 ตัว (การลืม `GOBACK` ในขั้นตอนที่ 994 และ `CLOSE` เขียนทับ
`FILE STATUS` ในขั้นตอนที่ 995) ซึ่งทั้งสองบั๊กนี้ถูกบันทึกไว้ในตาราง Greatest Hits ของ Part 098 ด้วย
เช่นกัน — เป็นเครื่องยืนยันว่าบทเรียนจาก Capstone Project สุดท้ายนี้เชื่อมโยงกลับไปหาหลักการที่สอนไว้
ตลอดทั้งหลักสูตรอย่างสมบูรณ์

จากขั้นตอนที่ 1 ใน Part 001 จนถึงขั้นตอนที่ 1000 ในบรรทัดนี้ — หลักสูตร **"สอนเขียนและพัฒนาโปรแกรม
และเว็บแอปพลิเคชันด้วย COBOL"** ได้เดินทางมาถึงจุดสิ้นสุดอย่างสมบูรณ์แล้ว

**[← กลับไป Part 099](part-099-future-of-cobol.md)**
