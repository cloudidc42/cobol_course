# Part 095: Case Study — ระบบ ERP และห่วงโซ่อุปทาน (ขั้นตอนที่ 941–950)

## คำนำของ Part นี้

Part 094 พาเราไปสำรวจธุรกิจประกันภัย Part นี้จะเปลี่ยนอุตสาหกรรมไปสู่ **ERP (Enterprise Resource
Planning) และห่วงโซ่อุปทาน (Supply Chain)** — ระบบประเภทที่ธุรกิจค้าปลีก โรงงานการผลิต และบริษัท
กระจายสินค้าทั่วโลกใช้บริหารจัดการสต๊อกและการสั่งซื้อ ตามที่ Part 001 เคยกล่าวถึงไว้ว่า **"ระบบค้าปลีก
ขนาดใหญ่สำหรับจัดการสต๊อกสินค้าและห่วงโซ่อุปทาน ที่ต้องประมวลผลแบบ Batch ปริมาณมหาศาลทุกคืน"** ก็เป็น
หนึ่งในอุตสาหกรรมที่ COBOL ยังคงมีบทบาทสำคัญ

ระบบในโปรเจกต์นี้คือการ**ขยายแนวคิด**จากโปรเจกต์รวบยอดเฟส 2 ใน Part 035 (ระบบจัดการสินค้าคงคลัง
Inventory Management System) ที่จัดการสต๊อกของ**คลังสินค้าเดียว** ให้กลายเป็นระบบที่จัดการสต๊อกของ
**หลายคลังสินค้าพร้อมกัน (Multi-warehouse)** พร้อมเพิ่มกระบวนการที่ Part 035 ยังไม่มี คือ **การสั่งซื้อ
สินค้า (Purchase Order)** และ **การรับสินค้าเข้าคลัง (Goods Receipt)** ซึ่งเป็นหัวใจของห่วงโซ่อุปทาน
ที่แท้จริง

| เทคนิคที่นำมาใช้ | เรียนมาจาก / ขยายผลจาก |
|---|---|
| Indexed File พร้อม **Composite Key** (รวม 2 ฟิลด์เป็น key เดียว) | Part 028 (ขยายจาก single-field key) |
| ไฟล์ตัวนับลำดับ (Counter File) สำหรับสร้างเลขที่เอกสารอัตโนมัติ | Part 023 (Sequential File พื้นฐาน) |
| `SORT` พร้อม `INPUT PROCEDURE` และ `RELEASE` | Part 027 |
| Control Break พร้อมยอดรวมย่อยหลายชั้น | Part 026, Part 039 |
| การตรวจสอบเงื่อนไขก่อนอัปเดต (validation cascade) | Part 048, Part 094 |
| แนวคิดคลังสินค้าหลายแห่ง | ขยายจาก Part 035 (`INVMAST.DAT` คลังเดียว) |

### ระบบที่จะสร้างประกอบด้วย 4 โปรแกรมหลัก

1. **`ERPSETUP`** — สร้างไฟล์สต๊อกสินค้าหลัก (Stock Master) แบบ Indexed ที่มี **composite key**
   (รหัสคลัง + รหัสสินค้า) พร้อมข้อมูลตัวอย่าง 9 รายการ ครอบคลุม 3 คลังสินค้า 4 สินค้า และสร้างไฟล์
   ใบสั่งซื้อหลัก (PO Master) เปล่า ๆ พร้อมไฟล์ตัวนับเลขที่ใบสั่งซื้อ
2. **`PORDER`** — ประมวลผลคำขอสั่งซื้อสินค้า ตรวจสอบว่าคลัง+สินค้าที่ขอมีอยู่จริงในระบบ แล้วสร้าง
   ใบสั่งซื้อใหม่พร้อมเลขที่อัตโนมัติ
3. **`GRECEIPT`** — บันทึกการรับสินค้าเข้าคลัง ตรวจสอบว่าใบสั่งซื้อยังไม่ปิด และจำนวนที่รับไม่เกินที่
   สั่งไว้ แล้วปรับปรุงทั้งสถานะใบสั่งซื้อและจำนวนสต๊อกคงเหลือ
4. **`REORDPT`** — รายงานจุดสั่งซื้อซ้ำ (Reorder Point Trigger) แสดงสต๊อกของทุกสินค้าข้ามทุกคลัง
   จัดกลุ่มตามสินค้า (ไม่ใช่ตามคลัง) ด้วยเทคนิค `SORT`+Control Break พร้อมธงเตือนตำแหน่งที่ต่ำกว่า
   จุดสั่งซื้อ

ทั้งหมดเชื่อมกันด้วย **Copybook** `STOCKREC.CPY` และ `POREC.CPY` (เทคนิค Part 033)

### หมายเหตุสำคัญเรื่องการทดสอบและสภาพแวดล้อม

เช่นเดียวกับ Part 094 ระบบนี้ใช้ `ORGANIZATION IS INDEXED` จึงต้องคอมไพล์ด้วย **GnuCOBOL build ที่
เปิดใช้ ISAM handler (Berkeley DB)** ที่ `/opt/gnucobol-isam/`:

```bash
/opt/gnucobol-isam/bin/cobc -x -o <program-name> <program-name>.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./<program-name>
```

**ทุกโปรแกรมและทุกผลลัพธ์ในเอกสารนี้ผ่านการคอมไพล์และรันทดสอบจริง** ด้วย GnuCOBOL
(`cobc (GnuCOBOL) 4.0-early-dev.0`, ISAM handler = BDB 5.3.28) เหมือนที่ยืนยันไว้ตลอดหลักสูตร

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้ (คอมเมนต์, ชื่อตัวแปร/paragraph,
> ข้อความใน `DISPLAY`/string literal ทุกชนิด) เป็นภาษาอังกฤษ/ASCII ล้วน

---

## ขั้นตอนที่ 941: การวิเคราะห์ความต้องการและออกแบบสถาปัตยกรรม Multi-Warehouse

### ทำไมต้องขยายจากคลังเดียวไปหลายคลัง

ใน Part 035 `INVMAST.DAT` มี key เป็นแค่รหัสสินค้า (`INV-PRODUCT-CODE`) — สมมติฐานคือ **มีคลังสินค้า
เพียงแห่งเดียว** สินค้าหนึ่งรหัสจึงมีจำนวนคงเหลือเพียงค่าเดียวในระบบทั้งหมด แต่ในธุรกิจจริง (โดยเฉพาะ
ค้าปลีกหรือการผลิตที่มีหลายสาขา/โรงงาน) สินค้าตัวเดียวกันจะถูกเก็บไว้ที่**หลายที่พร้อมกัน** และแต่ละที่
มีจำนวนคงเหลือของตัวเองที่ไม่เกี่ยวข้องกัน ตัวอย่างเช่น สินค้า "Widget-A" อาจมีเหลือ 150 ชิ้นที่คลัง
กรุงเทพฯ แต่เหลือแค่ 30 ชิ้นที่คลังเชียงใหม่ — ทั้งสองค่านี้ต้องถูกติดตามแยกจากกันโดยสิ้นเชิง

### การออกแบบ Composite Key

วิธีแก้ปัญหานี้ในทางเทคนิคคือการเปลี่ยน key ของไฟล์สต๊อกจาก "รหัสสินค้าอย่างเดียว" เป็น
**"รหัสคลัง + รหัสสินค้า" รวมกัน** (Composite Key) ทำให้ทุกคู่ (คลัง, สินค้า) มี record ของตัวเอง
เป็นเอกเทศ:

```cobol
           05  STOCK-KEY.
               10  STOCK-WH-CODE        PIC X(4).
               10  STOCK-PROD-CODE      PIC X(6).
```

`STOCK-KEY` เป็น group item ที่ครอบ `STOCK-WH-CODE` และ `STOCK-PROD-CODE` ไว้ด้วยกัน เมื่อกำหนด
`RECORD KEY IS STOCK-KEY` ใน `SELECT` clause แล้ว GnuCOBOL จะมองทั้ง 10 ไบต์นี้ (4+6) เป็น "key เดียว"
ในการเรียงลำดับและค้นหา ทำให้ record ของ "WH01/P0001" กับ "WH02/P0001" เป็นคนละ record กันโดยสมบูรณ์
แม้จะเป็นสินค้ารหัสเดียวกันก็ตาม — นี่คือเทคนิคที่ต่อยอดจาก Part 028 ซึ่งสอน key แบบฟิลด์เดียว

### ความต้องการของระบบ (Requirements)

| ข้อ | ความต้องการ |
|---|---|
| 1 | เก็บสต๊อกสินค้าแยกตามคลัง: จำนวนคงเหลือ, จุดสั่งซื้อ (reorder point), จำนวนที่ควรสั่งเมื่อถึงจุดนั้น |
| 2 | รับคำขอสั่งซื้อสินค้า ตรวจสอบว่าคลัง+สินค้านั้นมีอยู่จริงในระบบก่อนสร้างใบสั่งซื้อ |
| 3 | ใบสั่งซื้อแต่ละใบต้องมีเลขที่ไม่ซ้ำกันที่ระบบสร้างขึ้นเองอัตโนมัติ |
| 4 | รับสินค้าเข้าคลังตามใบสั่งซื้อ ปรับสถานะใบสั่งซื้อ (เปิด/รับบางส่วน/ปิด) และปรับสต๊อกของคลังที่ถูกต้อง |
| 5 | ห้ามรับสินค้าเกินจำนวนที่สั่งไว้ในใบสั่งซื้อนั้น |
| 6 | สร้างรายงานแสดงจุดสั่งซื้อซ้ำ จัดกลุ่มตามสินค้า แสดงทุกคลังที่มีสินค้านั้น พร้อมยอดรวมข้ามคลัง |
| 7 | ทุกธุรกรรมที่ผิดเงื่อนไขต้องถูกปฏิเสธพร้อมเหตุผล โดยไม่ทำให้โปรแกรมล่ม |

### สถาปัตยกรรมระบบ: แผนผังโมดูลและการไหลของข้อมูล

```
                    +-------------------+
                    |    ERPSETUP        |  (Step 942 - run once)
                    +--------+-----------+
                             |
                 creates     v
        +----------------------------------------+
        | STOCK.DAT (Indexed, key = WH+PROD)      |
        | POMASTER.DAT (Indexed, key = PO-NUMBER) |
        | POCTL.DAT (PO number counter)           |
        +---------------+--------------------------+
                         ^        ^
             reads/      |        |     reads/
             writes      |        |     writes
        +----------------+--+  +--+------------------+
        |    PORDER          |  |   GRECEIPT           |
        | (Step 943-944)     |  |  (Step 945-946)      |
        +--------+-----------+  +----------+-----------+
                 ^                          ^
                 |                          |
            POREQ.DAT                 GRECPT.DAT
         (PO requests)             (receipt transactions)

                         STOCK.DAT (after all updates)
                                 |
                                 v
                         +-------------------+
                         |     REORDPT        |  (Step 947-948)
                         | SORT by product    |
                         | across warehouses  |
                         +-------------------+
```

### เหตุผลของการตัดสินใจออกแบบที่สำคัญ

- **ทำไมใช้ไฟล์ตัวนับ (`POCTL.DAT`) แทนการให้ผู้ใช้ป้อนเลขที่ใบสั่งซื้อเอง**: เลขที่เอกสารที่มนุษย์
  กำหนดเองมีความเสี่ยงสูงที่จะซ้ำกันหรือข้ามเลข การให้ระบบ**อ่านค่าล่าสุด บวก 1 แล้วบันทึกกลับ**
  (Pattern พื้นฐานของ Sequence Generator ในระบบองค์กรจริง) รับประกันว่าทุกใบสั่งซื้อมีเลขที่ไม่ซ้ำกัน
  โดยอัตโนมัติ แม้จะเรียบง่ายกว่า Sequence Generator ระดับฐานข้อมูลจริง แต่หลักการเดียวกันทุกประการ
- **ทำไม `GRECEIPT` ปฏิเสธการรับเกินจำนวนที่สั่งไว้แบบไม่มีการรับบางส่วน (ไม่ cap ให้อัตโนมัติ)**:
  ต่างจาก `CLAIMPROC` ใน Part 094 ที่ "cap" ยอดจ่ายสินไหมให้ไม่เกินวงเงิน การรับสินค้าเกินจำนวนสั่งซื้อ
  มักบ่งชี้ถึง**ความผิดพลาดในการป้อนข้อมูล** (พิมพ์จำนวนผิด, สแกนบาร์โค้ดซ้ำ) มากกว่าจะเป็นสถานการณ์
  ทางธุรกิจที่ถูกต้อง จึงควรปฏิเสธทั้งรายการให้พนักงานตรวจสอบใหม่ แทนที่จะรับเข้าบางส่วนแล้วปล่อยให้
  ส่วนต่างหายไปเงียบ ๆ
- **ทำไม `REORDPT` ใช้ `SORT` แทนการวนอ่าน `STOCK-MASTER` ตรง ๆ**: ไฟล์ `STOCK.DAT` มี key เรียงตาม
  "คลัง แล้วค่อยสินค้า" (เพราะ `STOCK-WH-CODE` มาก่อน `STOCK-PROD-CODE` ใน composite key) แต่รายงาน
  ที่ต้องการคือ "จัดกลุ่มตามสินค้า แสดงทุกคลัง" ซึ่งเป็นลำดับที่**ตรงข้ามกับลำดับการจัดเก็บจริงของไฟล์**
  การ `SORT` จึงจำเป็นเพื่อเปลี่ยนมุมมองข้อมูลโดยไม่ต้องเปลี่ยนโครงสร้างไฟล์หลัก (เทคนิคเดียวกับที่
  Part 027 สอนไว้)

### ทางเลือกด้านสถาปัตยกรรมที่พิจารณาแล้วไม่เลือกใช้

| ทางเลือกที่พิจารณา | เหตุผลที่ไม่เลือกใช้ในโปรเจกต์นี้ |
|---|---|
| สร้างไฟล์สต๊อกแยกไฟล์ต่อคลัง (`STOCK_WH01.DAT`, `STOCK_WH02.DAT`, ...) | ทำให้รายงานข้ามคลัง (เช่น `REORDPT`) ต้องเปิดหลายไฟล์พร้อมกันและรวมผลเอง ซับซ้อนกว่า composite key มากโดยไม่ได้ประโยชน์เพิ่ม |
| ใช้ Alternate Index บนรหัสสินค้าเพื่อให้ `REORDPT` ค้นหาแบบสุ่มตามสินค้าได้โดยตรง | Alternate Index เป็นฟีเจอร์ VSAM ขั้นสูงที่สอนในเชิงแนวคิดใน Part 056 (Mainframe) ไม่ใช่ทุก build ของ GnuCOBOL รองรับแบบเดียวกัน การ SORT ธรรมดาจึงพกพาได้กว้างกว่าและเพียงพอสำหรับขนาดข้อมูลของ Case Study นี้ |
| ให้ `GRECEIPT` cap จำนวนรับให้ไม่เกินยอดสั่งซื้ออัตโนมัติ | ดังที่อธิบายด้านบน อาจซ่อนความผิดพลาดในการป้อนข้อมูลจริงไว้โดยไม่รู้ตัว การปฏิเสธทั้งรายการปลอดภัยกว่าในบริบทนี้ |

### แบบฝึกหัดที่ 941.1

**โจทย์**: จงอธิบายว่าทำไม `STOCK-WH-CODE` ถูกจัดให้อยู่**ก่อน** `STOCK-PROD-CODE` ใน composite key
(ไม่ใช่กลับกัน) และการสลับลำดับนี้จะส่งผลอย่างไรต่อลำดับการอ่านไฟล์แบบ sequential

**เฉลยแนวทาง**: ลำดับของฟิลด์ใน composite key จะกำหนด**ลำดับการเรียง**ของไฟล์เมื่ออ่านแบบ sequential
ตาม key ถ้า `STOCK-WH-CODE` มาก่อน ไฟล์จะถูกเรียงโดยกลุ่มคลังก่อน (ทุก record ของ WH01 มาต่อกันหมด
แล้วค่อยขึ้น WH02) ซึ่งเหมาะกับงานที่ดำเนินการทีละคลัง เช่น "พิมพ์รายงานสต๊อกของคลังนี้ทั้งหมด" ถ้า
สลับให้ `STOCK-PROD-CODE` มาก่อน ไฟล์จะถูกเรียงโดยกลุ่มสินค้าก่อน (ทุกคลังของ P0001 มาต่อกัน แล้วค่อย
ขึ้น P0002) ซึ่งจะเหมาะกับรายงานแบบ `REORDPT` มากกว่าโดยไม่ต้อง `SORT` เลย การออกแบบในบทนี้เลือกให้
คลังมาก่อนเพราะสอดคล้องกับการค้นหาแบบสุ่มที่พบบ่อยกว่าในการปฏิบัติงานประจำวัน (พนักงานคลังมักค้นหา/
อัปเดตสินค้าภายในคลังของตัวเอง) แล้วใช้ `SORT` จัดการกรณีพิเศษอย่าง `REORDPT` แทน

---

## ขั้นตอนที่ 942: Copybooks และโปรแกรม `ERPSETUP`

### Copybook `STOCKREC.CPY`

```cobol
      ******************************************************************
      * STOCKREC.CPY
      * Shared multi-warehouse stock record layout. The key is a
      * COMPOSITE key: warehouse code + product code together, so the
      * same product can have an independent stock level in every
      * warehouse (extends the single-warehouse INVMAST design from
      * Part 035 to multiple locations).
      ******************************************************************
           05  STOCK-KEY.
               10  STOCK-WH-CODE        PIC X(4).
               10  STOCK-PROD-CODE      PIC X(6).
           05  STOCK-PROD-NAME          PIC X(20).
           05  STOCK-QTY-ON-HAND        PIC 9(7).
           05  STOCK-REORDER-POINT      PIC 9(7).
           05  STOCK-REORDER-QTY        PIC 9(7).
           05  STOCK-UNIT-COST          PIC 9(7)V99.
```

### Copybook `POREC.CPY`

```cobol
      ******************************************************************
      * POREC.CPY
      * Shared purchase order record layout. Key is PO-NUMBER.
      * PO-STATUS: O = Open, P = Partially received, C = Closed
      * (fully received), X = Rejected/Cancelled.
      ******************************************************************
           05  PO-NUMBER                PIC X(8).
           05  PO-WH-CODE               PIC X(4).
           05  PO-PROD-CODE             PIC X(6).
           05  PO-SUPPLIER-CODE         PIC X(6).
           05  PO-ORDER-DATE            PIC 9(8).
           05  PO-QTY-ORDERED           PIC 9(7).
           05  PO-QTY-RECEIVED          PIC 9(7).
           05  PO-UNIT-COST             PIC 9(7)V99.
           05  PO-STATUS                PIC X(1).
```

### ข้อมูลตัวอย่าง 9 รายการสต๊อก (3 คลัง x 4 สินค้า, ไม่ครบทุกคู่)

| คลัง | สินค้า | ชื่อ | คงเหลือ | จุดสั่งซื้อ | สั่งซื้อเมื่อถึงจุด | ต้นทุน/หน่วย |
|---|---|---|---|---|---|---|
| WH01 | P0001 | Widget-A | 150 | 100 | 200 | 12.50 |
| WH01 | P0002 | Gadget-B | 40 | 50 | 100 | 25.00 |
| WH01 | P0003 | Bolt-C | 500 | 200 | 300 | 0.75 |
| WH02 | P0001 | Widget-A | 30 | 80 | 150 | 12.50 |
| WH02 | P0002 | Gadget-B | 90 | 60 | 100 | 25.00 |
| WH02 | P0004 | Casing-D | 20 | 25 | 50 | 8.00 |
| WH03 | P0001 | Widget-A | 200 | 100 | 200 | 12.50 |
| WH03 | P0003 | Bolt-C | 60 | 150 | 300 | 0.75 |
| WH03 | P0004 | Casing-D | 75 | 25 | 50 | 8.00 |

สังเกตว่าข้อมูลตั้งต้นมี **4 ตำแหน่งที่ต่ำกว่าจุดสั่งซื้ออยู่แล้ว**: WH01/P0002 (40<50), WH02/P0001
(30<80), WH02/P0004 (20<25), WH03/P0003 (60<150) — ออกแบบไว้เพื่อทดสอบ `REORDPT` ให้เห็นผลชัดเจน

### เต็มโค้ด `ERPSETUP.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ERPSETUP.
      ******************************************************************
      * ERP / supply chain case study - one-time setup job.
      * Creates STOCK-MASTER (indexed, composite key = warehouse code
      * + product code) with nine seed records spread across three
      * warehouses and four products, and creates an empty PO-MASTER
      * (indexed, key = PO-NUMBER) plus a PO number counter file.
      ******************************************************************
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT STOCK-MASTER ASSIGN TO "STOCK.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS STOCK-KEY
               FILE STATUS IS WS-STOCK-STATUS.

           SELECT PO-MASTER ASSIGN TO "POMASTER.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS PO-NUMBER
               FILE STATUS IS WS-PO-STATUS.

           SELECT PO-COUNTER-FILE ASSIGN TO "POCTL.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-CTL-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  STOCK-MASTER.
       01  STOCK-RECORD.
           COPY "stockrec.cpy".

       FD  PO-MASTER.
       01  PO-RECORD.
           COPY "porec.cpy".

       FD  PO-COUNTER-FILE.
       01  PO-COUNTER-RECORD        PIC 9(7).

       WORKING-STORAGE SECTION.
       01  WS-STOCK-STATUS          PIC X(2).
       01  WS-PO-STATUS             PIC X(2).
       01  WS-CTL-STATUS            PIC X(2).
       01  WS-COUNT                 PIC 9(3) VALUE 0.

       01  SEED-TABLE.
           05  SEED-ENTRY OCCURS 9 TIMES.
               10  SEED-WH           PIC X(4).
               10  SEED-PROD         PIC X(6).
               10  SEED-NAME         PIC X(20).
               10  SEED-QTY          PIC 9(7).
               10  SEED-ROP          PIC 9(7).
               10  SEED-ROQ          PIC 9(7).
               10  SEED-COST         PIC 9(7)V99.
       01  WS-IDX                   PIC 9(2).

       PROCEDURE DIVISION.
       000-MAIN.
           PERFORM 100-LOAD-SEED-DATA
           PERFORM 200-CREATE-STOCK-MASTER
           PERFORM 300-CREATE-PO-MASTER
           PERFORM 400-INIT-PO-COUNTER
           DISPLAY "ERPSETUP COMPLETE - STOCK RECORDS LOADED: " WS-COUNT
           STOP RUN.

       200-CREATE-STOCK-MASTER.
           OPEN OUTPUT STOCK-MASTER
           IF WS-STOCK-STATUS NOT = "00"
               DISPLAY "ERROR OPENING STOCK-MASTER: " WS-STOCK-STATUS
               STOP RUN
           END-IF

           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > 9
               MOVE SEED-WH(WS-IDX)    TO STOCK-WH-CODE
               MOVE SEED-PROD(WS-IDX)  TO STOCK-PROD-CODE
               MOVE SEED-NAME(WS-IDX)  TO STOCK-PROD-NAME
               MOVE SEED-QTY(WS-IDX)   TO STOCK-QTY-ON-HAND
               MOVE SEED-ROP(WS-IDX)   TO STOCK-REORDER-POINT
               MOVE SEED-ROQ(WS-IDX)   TO STOCK-REORDER-QTY
               MOVE SEED-COST(WS-IDX)  TO STOCK-UNIT-COST
               WRITE STOCK-RECORD
               IF WS-STOCK-STATUS = "00"
                   ADD 1 TO WS-COUNT
               ELSE
                   DISPLAY "WRITE FAILED " STOCK-KEY
                       " STATUS " WS-STOCK-STATUS
               END-IF
           END-PERFORM

           CLOSE STOCK-MASTER.

       300-CREATE-PO-MASTER.
           OPEN OUTPUT PO-MASTER
           IF WS-PO-STATUS NOT = "00"
               DISPLAY "ERROR OPENING PO-MASTER: " WS-PO-STATUS
               STOP RUN
           END-IF
           CLOSE PO-MASTER.

       400-INIT-PO-COUNTER.
           OPEN OUTPUT PO-COUNTER-FILE
           MOVE 0 TO PO-COUNTER-RECORD
           WRITE PO-COUNTER-RECORD
           CLOSE PO-COUNTER-FILE.

       100-LOAD-SEED-DATA.
           MOVE "WH01" TO SEED-WH(1)
           MOVE "P0001" TO SEED-PROD(1)
           MOVE "WIDGET-A            " TO SEED-NAME(1)
           MOVE 150 TO SEED-QTY(1)
           MOVE 100 TO SEED-ROP(1)
           MOVE 200 TO SEED-ROQ(1)
           MOVE 12.50 TO SEED-COST(1)

           MOVE "WH01" TO SEED-WH(2)
           MOVE "P0002" TO SEED-PROD(2)
           MOVE "GADGET-B            " TO SEED-NAME(2)
           MOVE 40 TO SEED-QTY(2)
           MOVE 50 TO SEED-ROP(2)
           MOVE 100 TO SEED-ROQ(2)
           MOVE 25.00 TO SEED-COST(2)

           MOVE "WH01" TO SEED-WH(3)
           MOVE "P0003" TO SEED-PROD(3)
           MOVE "BOLT-C              " TO SEED-NAME(3)
           MOVE 500 TO SEED-QTY(3)
           MOVE 200 TO SEED-ROP(3)
           MOVE 300 TO SEED-ROQ(3)
           MOVE 0.75 TO SEED-COST(3)

           MOVE "WH02" TO SEED-WH(4)
           MOVE "P0001" TO SEED-PROD(4)
           MOVE "WIDGET-A            " TO SEED-NAME(4)
           MOVE 30 TO SEED-QTY(4)
           MOVE 80 TO SEED-ROP(4)
           MOVE 150 TO SEED-ROQ(4)
           MOVE 12.50 TO SEED-COST(4)

           MOVE "WH02" TO SEED-WH(5)
           MOVE "P0002" TO SEED-PROD(5)
           MOVE "GADGET-B            " TO SEED-NAME(5)
           MOVE 90 TO SEED-QTY(5)
           MOVE 60 TO SEED-ROP(5)
           MOVE 100 TO SEED-ROQ(5)
           MOVE 25.00 TO SEED-COST(5)

           MOVE "WH02" TO SEED-WH(6)
           MOVE "P0004" TO SEED-PROD(6)
           MOVE "CASING-D            " TO SEED-NAME(6)
           MOVE 20 TO SEED-QTY(6)
           MOVE 25 TO SEED-ROP(6)
           MOVE 50 TO SEED-ROQ(6)
           MOVE 8.00 TO SEED-COST(6)

           MOVE "WH03" TO SEED-WH(7)
           MOVE "P0001" TO SEED-PROD(7)
           MOVE "WIDGET-A            " TO SEED-NAME(7)
           MOVE 200 TO SEED-QTY(7)
           MOVE 100 TO SEED-ROP(7)
           MOVE 200 TO SEED-ROQ(7)
           MOVE 12.50 TO SEED-COST(7)

           MOVE "WH03" TO SEED-WH(8)
           MOVE "P0003" TO SEED-PROD(8)
           MOVE "BOLT-C              " TO SEED-NAME(8)
           MOVE 60 TO SEED-QTY(8)
           MOVE 150 TO SEED-ROP(8)
           MOVE 300 TO SEED-ROQ(8)
           MOVE 0.75 TO SEED-COST(8)

           MOVE "WH03" TO SEED-WH(9)
           MOVE "P0004" TO SEED-PROD(9)
           MOVE "CASING-D            " TO SEED-NAME(9)
           MOVE 75 TO SEED-QTY(9)
           MOVE 25 TO SEED-ROP(9)
           MOVE 50 TO SEED-ROQ(9)
           MOVE 8.00 TO SEED-COST(9).
```

### คอมไพล์และรัน

```bash
/opt/gnucobol-isam/bin/cobc -x -o erpsetup erpsetup.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./erpsetup
```

**ผลลัพธ์จริง**:

```
ERPSETUP COMPLETE - STOCK RECORDS LOADED: 009
```

### ข้อควรระวัง

**ต้องบันทึก `PO-COUNTER-RECORD` เป็น `PIC 9(7)` ธรรมดา ไม่ใช่ `PIC X(7)`**: เพราะ `PORDER` (ขั้นตอน
ถัดไป) จะต้อง `ADD 1` เข้าไปในค่านี้โดยตรง ถ้าประกาศเป็นตัวอักษร (`X`) การบวกเลขจะทำไม่ได้และต้อง
แปลงชนิดข้อมูลก่อนทุกครั้ง ซึ่งเพิ่มความซับซ้อนโดยไม่จำเป็น การเลือกชนิดข้อมูลให้ตรงกับการใช้งานตั้งแต่
ขั้นออกแบบช่วยลดโค้ดแปลงชนิดข้อมูล (type conversion) ที่ไม่จำเป็นในภายหลัง

### แบบฝึกหัดที่ 942.1

**โจทย์**: จงอธิบายว่าทำไม `PO-MASTER` ใน `ERPSETUP` ถูกเปิดด้วย `OPEN OUTPUT` แล้ว `CLOSE` ทันที
โดยไม่เขียน record ใด ๆ ลงไปเลย ต่างจาก `STOCK-MASTER` ที่เขียนข้อมูลตั้งต้นเข้าไปด้วย

**เฉลย**: `STOCK-MASTER` ต้องมีข้อมูลตั้งต้นเพราะสต๊อกสินค้าเป็นข้อมูลที่**มีอยู่จริงในโลกธุรกิจอยู่แล้ว**
ก่อนระบบจะเริ่มทำงาน (คลังสินค้าไม่ได้เริ่มจากศูนย์) แต่ **ใบสั่งซื้อ (Purchase Order) ยังไม่มีใบใด
เกิดขึ้นเลย** ณ จุดเริ่มต้นของระบบ — มันจะถูกสร้างขึ้นทีละใบในภายหลังโดยโปรแกรม `PORDER` เท่านั้น
การ `OPEN OUTPUT` แล้ว `CLOSE` ทันทีจึงเป็นวิธีมาตรฐานใน COBOL สำหรับ **"สร้างโครงไฟล์ Indexed เปล่า ๆ
ให้พร้อมใช้งาน"** โดยไม่มี record ใด ๆ อยู่ข้างใน เทียบเท่ากับการสร้างตารางฐานข้อมูลเปล่าที่ยังไม่มีแถว
ข้อมูลในโลกของฐานข้อมูลเชิงสัมพันธ์

---

## ขั้นตอนที่ 943: โปรแกรม `PORDER` — การออกแบบและเทคนิคตัวนับลำดับ

### รูปแบบไฟล์ตัวนับ (Counter File Pattern)

การสร้างเลขที่เอกสารอัตโนมัติที่ไม่ซ้ำกันเป็นปัญหาที่พบบ่อยมากในระบบธุรกิจ วิธีแก้แบบง่ายที่สุดใน
COBOL แบบไฟล์ (ไม่มีฐานข้อมูลที่มี auto-increment ให้ใช้) คือ **ไฟล์ตัวนับ**: เก็บตัวเลขล่าสุดที่ใช้
ไปแล้วไว้ในไฟล์เล็ก ๆ หนึ่งไฟล์ ทุกครั้งที่ต้องการเลขใหม่ ให้ (1) อ่านค่าปัจจุบันเข้ามา (2) บวก 1
(3) ใช้ค่าที่บวกแล้ว (4) เมื่อจบโปรแกรม บันทึกค่าล่าสุดกลับลงไฟล์:

```cobol
       050-READ-COUNTER.
           OPEN INPUT PO-COUNTER-FILE
           READ PO-COUNTER-FILE
               AT END
                   MOVE 0 TO PO-COUNTER-RECORD
           END-READ
           MOVE PO-COUNTER-RECORD TO WS-NEXT-PO-NUM
           CLOSE PO-COUNTER-FILE.

       060-WRITE-COUNTER.
           OPEN OUTPUT PO-COUNTER-FILE
           MOVE WS-NEXT-PO-NUM TO PO-COUNTER-RECORD
           WRITE PO-COUNTER-RECORD
           CLOSE PO-COUNTER-FILE.
```

**ข้อจำกัดที่ต้องตระหนัก**: รูปแบบนี้ใช้ได้ดีเมื่อมี**โปรแกรมเดียวรันในเวลาเดียว** (single-threaded
batch job) ถ้ามีสองโปรเซสรัน `PORDER` พร้อมกันจริง ๆ (concurrent) ทั้งคู่อาจอ่านค่าเดิมก่อนที่อีกฝั่ง
จะบันทึกค่าใหม่ ทำให้ได้เลขที่ซ้ำกัน (Race Condition) — ระบบองค์กรจริงจะใช้กลไกล็อกไฟล์หรือ Database
Sequence ที่มีการควบคุมระดับ transaction แทน แต่สำหรับ batch job ที่รันทีละโปรแกรมตามลำดับ (ซึ่งเป็น
ธรรมชาติของงาน COBOL แบบ Mainframe ส่วนใหญ่ ทบทวน Part 066) รูปแบบไฟล์ตัวนับนี้เพียงพอและใช้งานจริง
กันอย่างแพร่หลาย

### การประกอบเลขที่ใบสั่งซื้อจากตัวเลข

```cobol
       01  WS-NEW-PO-NUMBER.
           05  FILLER                PIC X(2) VALUE "PO".
           05  WS-PO-SEQ             PIC 9(6).
```

`WS-NEW-PO-NUMBER` เป็น group item ที่ประกอบด้วยตัวอักษรคงที่ `"PO"` ตามด้วยตัวเลข 6 หลัก เมื่อ
`MOVE` ตัวเลขลำดับเข้า `WS-PO-SEQ` แล้ว การอ้างอิง `WS-NEW-PO-NUMBER` ทั้งกลุ่มจะได้ค่าเช่น
`"PO000001"` โดยอัตโนมัติ — เทคนิคเดียวกับการประกอบเลขบัญชี/เลขที่เอกสารที่พบได้ทั่วไปในโปรแกรม COBOL
ระดับองค์กร

### เต็มโค้ด `PORDER.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PORDER.
      ******************************************************************
      * ERP / supply chain case study - purchase order processing.
      * Reads POREQ.DAT (sequential PO requests). Each request is
      * validated against STOCK-MASTER (the warehouse/product
      * combination must already exist there); valid requests get the
      * next PO number from POCTL.DAT (a simple running counter file)
      * and are written to PO-MASTER with status Open.
      ******************************************************************
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT STOCK-MASTER ASSIGN TO "STOCK.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS STOCK-KEY
               FILE STATUS IS WS-STOCK-STATUS.

           SELECT PO-MASTER ASSIGN TO "POMASTER.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS PO-NUMBER
               FILE STATUS IS WS-PO-STATUS.

           SELECT PO-REQUEST-FILE ASSIGN TO "POREQ.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-REQ-STATUS.

           SELECT PO-COUNTER-FILE ASSIGN TO "POCTL.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-CTL-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  STOCK-MASTER.
       01  STOCK-RECORD.
           COPY "stockrec.cpy".

       FD  PO-MASTER.
       01  PO-RECORD.
           COPY "porec.cpy".

       FD  PO-REQUEST-FILE.
       01  PO-REQUEST-RECORD.
           05  REQ-WH-CODE             PIC X(4).
           05  REQ-PROD-CODE           PIC X(6).
           05  REQ-SUPPLIER-CODE       PIC X(6).
           05  REQ-QTY-ORDERED         PIC 9(7).
           05  REQ-UNIT-COST           PIC 9(7)V99.
           05  REQ-ORDER-DATE          PIC 9(8).

       FD  PO-COUNTER-FILE.
       01  PO-COUNTER-RECORD           PIC 9(7).

       WORKING-STORAGE SECTION.
       01  WS-STOCK-STATUS          PIC X(2).
       01  WS-PO-STATUS             PIC X(2).
       01  WS-REQ-STATUS            PIC X(2).
       01  WS-CTL-STATUS            PIC X(2).
       01  WS-EOF                   PIC X(1) VALUE "N".
           88  END-OF-REQUESTS                VALUE "Y".

       01  WS-NEXT-PO-NUM           PIC 9(7).
       01  WS-NEW-PO-NUMBER.
           05  FILLER                PIC X(2) VALUE "PO".
           05  WS-PO-SEQ             PIC 9(6).

       01  WS-COUNTERS.
           05  WS-CREATED-COUNT     PIC 9(3) VALUE 0.
           05  WS-REJECTED-COUNT    PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       000-MAIN.
           PERFORM 050-READ-COUNTER
           OPEN INPUT STOCK-MASTER
           OPEN I-O PO-MASTER
           OPEN INPUT PO-REQUEST-FILE

           DISPLAY "==== PURCHASE ORDER PROCESSING ===="

           PERFORM UNTIL END-OF-REQUESTS
               READ PO-REQUEST-FILE
                   AT END
                       SET END-OF-REQUESTS TO TRUE
                   NOT AT END
                       PERFORM 200-PROCESS-REQUEST
               END-READ
           END-PERFORM

           CLOSE STOCK-MASTER
           CLOSE PO-MASTER
           CLOSE PO-REQUEST-FILE
           PERFORM 060-WRITE-COUNTER

           DISPLAY " "
           DISPLAY "PURCHASE ORDERS CREATED : " WS-CREATED-COUNT
           DISPLAY "REQUESTS REJECTED       : " WS-REJECTED-COUNT
           STOP RUN.

       050-READ-COUNTER.
           OPEN INPUT PO-COUNTER-FILE
           READ PO-COUNTER-FILE
               AT END
                   MOVE 0 TO PO-COUNTER-RECORD
           END-READ
           MOVE PO-COUNTER-RECORD TO WS-NEXT-PO-NUM
           CLOSE PO-COUNTER-FILE.

       060-WRITE-COUNTER.
           OPEN OUTPUT PO-COUNTER-FILE
           MOVE WS-NEXT-PO-NUM TO PO-COUNTER-RECORD
           WRITE PO-COUNTER-RECORD
           CLOSE PO-COUNTER-FILE.

       200-PROCESS-REQUEST.
           MOVE REQ-WH-CODE   TO STOCK-WH-CODE
           MOVE REQ-PROD-CODE TO STOCK-PROD-CODE
           READ STOCK-MASTER
               INVALID KEY
                   DISPLAY "REJECT " REQ-WH-CODE "/" REQ-PROD-CODE
                       " REASON: NOT FOUND IN STOCK MASTER"
                   ADD 1 TO WS-REJECTED-COUNT
               NOT INVALID KEY
                   PERFORM 210-CREATE-PO
           END-READ.

       210-CREATE-PO.
           ADD 1 TO WS-NEXT-PO-NUM
           MOVE WS-NEXT-PO-NUM TO WS-PO-SEQ
           MOVE WS-NEW-PO-NUMBER  TO PO-NUMBER
           MOVE REQ-WH-CODE       TO PO-WH-CODE
           MOVE REQ-PROD-CODE     TO PO-PROD-CODE
           MOVE REQ-SUPPLIER-CODE TO PO-SUPPLIER-CODE
           MOVE REQ-ORDER-DATE    TO PO-ORDER-DATE
           MOVE REQ-QTY-ORDERED   TO PO-QTY-ORDERED
           MOVE 0                 TO PO-QTY-RECEIVED
           MOVE REQ-UNIT-COST     TO PO-UNIT-COST
           MOVE "O"               TO PO-STATUS

           WRITE PO-RECORD
           IF WS-PO-STATUS = "00"
               DISPLAY "CREATED " PO-NUMBER " FOR " REQ-WH-CODE
                   "/" REQ-PROD-CODE " QTY " REQ-QTY-ORDERED
               ADD 1 TO WS-CREATED-COUNT
           ELSE
               DISPLAY "WRITE FAILED " PO-NUMBER
                   " STATUS " WS-PO-STATUS
               ADD 1 TO WS-REJECTED-COUNT
           END-IF.
```

### แบบฝึกหัดที่ 943.1

**โจทย์**: ทำไมโปรแกรมนี้ต้องอ่านตัวนับเข้ามา (`050-READ-COUNTER`) **ก่อน**เปิดไฟล์ `PO-REQUEST-FILE`
และบันทึกตัวนับกลับ (`060-WRITE-COUNTER`) **หลัง**ปิดไฟล์ทั้งหมดแล้ว แทนที่จะอ่าน/เขียนตัวนับทุกครั้ง
ที่สร้างใบสั่งซื้อหนึ่งใบ

**เฉลยแนวทาง**: การอ่านค่าตัวนับเพียงครั้งเดียวตอนเริ่มโปรแกรม เก็บไว้ในตัวแปร `WS-NEXT-PO-NUM` แล้ว
ค่อย ๆ `ADD 1` ในหน่วยความจำตลอดการประมวลผล และบันทึกกลับเพียงครั้งเดียวตอนจบ **มีประสิทธิภาพสูงกว่า
มาก** เพราะการเปิด-ปิดไฟล์ (`OPEN`/`CLOSE`) เป็นการดำเนินการที่ใช้ทรัพยากรระบบมากกว่าการบวกเลขใน
หน่วยความจำหลายเท่า ถ้าไฟล์คำขอมีหลายพันรายการ การเปิด-ปิดไฟล์ตัวนับซ้ำทุกรายการจะทำให้โปรแกรมช้าลง
อย่างมีนัยสำคัญโดยไม่จำเป็น เพราะภายในโปรแกรมเดียวไม่มีความเสี่ยงเรื่อง Race Condition อยู่แล้ว
(ทบทวนหลักการต้นทุนของ I/O operation จาก Part 068 เรื่อง Performance Tuning)

---

## ขั้นตอนที่ 944: รันและวิเคราะห์ผลลัพธ์ของ `PORDER`

### ข้อมูลคำขอสั่งซื้อทดสอบ `POREQ.DAT`

| ลำดับ | คลัง | สินค้า | ผู้จำหน่าย | จำนวนสั่ง | ต้นทุน/หน่วย | คาดว่าจะเป็น |
|---|---|---|---|---|---|---|
| 1 | WH02 | P0001 | SUP01 | 150 | 12.50 | สร้างสำเร็จ (มีอยู่ในสต๊อก) |
| 2 | WH02 | P0004 | SUP02 | 50 | 8.00 | สร้างสำเร็จ |
| 3 | WH03 | P0003 | SUP03 | 300 | 0.75 | สร้างสำเร็จ |
| 4 | WH01 | P0099 | SUP01 | 100 | 5.00 | ปฏิเสธ — ไม่มีสินค้า P0099 ในคลัง WH01 |

### คอมไพล์และรัน

```bash
/opt/gnucobol-isam/bin/cobc -x -o porder porder.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./porder
```

**ผลลัพธ์จริง**:

```
==== PURCHASE ORDER PROCESSING ====
CREATED PO000001 FOR WH02/P0001  QTY 0000150
CREATED PO000002 FOR WH02/P0004  QTY 0000050
CREATED PO000003 FOR WH03/P0003  QTY 0000300
REJECT WH01/P0099  REASON: NOT FOUND IN STOCK MASTER

PURCHASE ORDERS CREATED : 003
REQUESTS REJECTED       : 001
```

### วิเคราะห์ผลลัพธ์

ตัวนับสร้างเลขที่ใบสั่งซื้อต่อเนื่อง `PO000001`, `PO000002`, `PO000003` ตามลำดับที่คำขอผ่านการตรวจสอบ
(ไม่ใช่ตามลำดับในไฟล์คำขอเปล่า ๆ — สังเกตว่าคำขอที่ 4 ถูกปฏิเสธ**ก่อน**จะไปถึงขั้นตอนขอเลขที่ใหม่ ทำให้
ไม่มี "PO000004" ที่หายไปโดยไม่มีเหตุผล) นี่คือรายละเอียดสำคัญ: **การขอเลขที่เอกสารใหม่เกิดขึ้นหลังจาก
ผ่านการตรวจสอบ (validation) แล้วเท่านั้น** ทบทวนโค้ด `200-PROCESS-REQUEST` และ `210-CREATE-PO`
จะเห็นว่า `ADD 1 TO WS-NEXT-PO-NUM` อยู่ใน `210-CREATE-PO` ซึ่งถูกเรียกจาก `NOT INVALID KEY` เท่านั้น

### ข้อควรระวัง

**ถ้าสลับลำดับ ให้ขอเลขที่ก่อนตรวจสอบ**: จะเกิด "เลขที่กระโดด" (gap) ในลำดับเอกสาร เช่น มี PO000001,
PO000002, PO000003, PO000005 (ขาด PO000004 ไปเพราะคำขอที่ 4 ถูกปฏิเสธหลังจากขอเลขที่ไปแล้ว) ในระบบ
บัญชี/การตรวจสอบภายใน (Audit) การมีเลขที่เอกสารกระโดดโดยไม่มีคำอธิบายอาจถูกตั้งคำถามว่าเอกสารบางใบ
หายไปหรือถูกลบทิ้งหรือไม่ แม้จะไม่ใช่ข้อผิดพลาดร้ายแรงเสมอไป แต่เป็นแนวปฏิบัติที่ดีกว่าที่จะขอเลขที่
เอกสารก็ต่อเมื่อแน่ใจแล้วว่าจะใช้เลขนั้นจริง ๆ

### แบบฝึกหัดที่ 944.1

**โจทย์**: จงเพิ่มการตรวจสอบใหม่ใน `210-CREATE-PO` ให้ปฏิเสธคำขอที่ `REQ-QTY-ORDERED` เท่ากับ 0
พร้อมเหตุผลที่ชัดเจน

**เฉลย**:

```cobol
       200-PROCESS-REQUEST.
           IF REQ-QTY-ORDERED = 0
               DISPLAY "REJECT " REQ-WH-CODE "/" REQ-PROD-CODE
                   " REASON: ZERO ORDER QUANTITY"
               ADD 1 TO WS-REJECTED-COUNT
           ELSE
               MOVE REQ-WH-CODE   TO STOCK-WH-CODE
               MOVE REQ-PROD-CODE TO STOCK-PROD-CODE
               READ STOCK-MASTER
                   INVALID KEY
                       DISPLAY "REJECT " REQ-WH-CODE "/" REQ-PROD-CODE
                           " REASON: NOT FOUND IN STOCK MASTER"
                       ADD 1 TO WS-REJECTED-COUNT
                   NOT INVALID KEY
                       PERFORM 210-CREATE-PO
               END-READ
           END-IF.
```

การตรวจสอบนี้ป้องกันไม่ให้ระบบสร้างใบสั่งซื้อที่ไม่มีความหมายทางธุรกิจ (สั่งซื้อจำนวน 0 ชิ้น) ซึ่งอาจ
เกิดจากข้อผิดพลาดในการส่งออกข้อมูลจากระบบต้นทางมากกว่าจะเป็นความต้องการที่แท้จริง

---

## ขั้นตอนที่ 945: โปรแกรม `GRECEIPT` — การออกแบบตรรกะการรับสินค้า

### กฎธุรกิจของการรับสินค้าเข้าคลัง

เมื่อสินค้าตามใบสั่งซื้อมาถึงคลัง ต้องมีการ**ตรวจรับ** ก่อนปรับปรุงสต๊อก โดยมีเงื่อนไขตามลำดับ:

```
    รายการรับสินค้าเข้ามา
        |
        v
   [1] พบใบสั่งซื้อนี้หรือไม่?  --ไม่พบ--> REJECT: PO NOT FOUND
        | พบ
        v
   [2] ใบสั่งซื้อยังไม่ปิด/ยกเลิกหรือไม่ (สถานะ O หรือ P)?  --ปิดแล้ว--> REJECT: PO ALREADY CLOSED/CANCELLED
        | ยังเปิดอยู่
        v
   [3] จำนวนรับสะสม + จำนวนรับใหม่ เกินจำนวนที่สั่งไว้หรือไม่?  --เกิน--> REJECT: RECEIPT EXCEEDS ORDER QUANTITY
        | ไม่เกิน
        v
   ปรับปรุงใบสั่งซื้อ: บวกจำนวนรับสะสม, เปลี่ยนสถานะเป็น C (ปิด) ถ้ารับครบ มิฉะนั้นเป็น P (รับบางส่วน)
   ปรับปรุงสต๊อก: บวกจำนวนที่รับเข้าไปในคลัง+สินค้าที่ตรงกับใบสั่งซื้อ
```

### ทำไมต้องอัปเดต 2 ไฟล์ในธุรกรรมเดียว (PO-MASTER และ STOCK-MASTER)

การรับสินค้าหนึ่งครั้งส่งผลกระทบต่อ**ข้อมูล 2 ชุดพร้อมกัน**: (1) สถานะความคืบหน้าของใบสั่งซื้อ และ
(2) จำนวนสต๊อกจริงที่มีในคลัง ทั้งสองต้องอัปเดตให้สอดคล้องกันเสมอ (consistency) ถ้าอัปเดตแค่ไฟล์ใด
ไฟล์หนึ่ง ระบบจะมีข้อมูลที่ขัดแย้งกัน (เช่น ใบสั่งซื้อบอกว่ารับครบแล้ว แต่สต๊อกไม่เพิ่มขึ้นจริง)
โค้ดในขั้นตอนที่ 946 จะแสดงให้เห็นว่าทั้งสองการอัปเดตนี้เกิดขึ้นในย่อหน้าเดียวกัน (`220-APPLY-RECEIPT`)
เพื่อให้มั่นใจว่าจะไม่มีกรณีที่อัปเดตสำเร็จแค่ฝั่งเดียว (ในระบบฐานข้อมูลสมัยใหม่ ปัญหานี้จะแก้ด้วย
Transaction/COMMIT แต่สำหรับไฟล์ COBOL แบบดั้งเดิม การเขียนโค้ดให้ทั้งสองการอัปเดตอยู่ใกล้กันและไม่มี
เงื่อนไขให้ออกจากย่อหน้ากลางทางคือแนวทางปฏิบัติที่ดีที่สุดเท่าที่ทำได้)

### แบบฝึกหัดที่ 945.1

**โจทย์**: จงพิจารณา `210-VALIDATE-AND-APPLY` ในขั้นตอนที่ 946 แล้วตอบว่าทำไมการตรวจสอบ
`PO-STATUS = "C" OR PO-STATUS = "X"` ต้องมาก่อนการตรวจสอบจำนวนรับเกิน

**เฉลยแนวทาง**: ถ้าใบสั่งซื้อถูกปิด (`C`) ไปแล้วเพราะรับสินค้าครบตามจำนวนที่สั่งไว้แล้วก่อนหน้านี้
ค่า `PO-QTY-RECEIVED` ที่บันทึกไว้จะเท่ากับ `PO-QTY-ORDERED` พอดี การรับสินค้าเพิ่มเข้ามาอีกแม้เพียง
1 หน่วยก็จะทำให้ `WS-NEW-TOTAL-RECEIVED > PO-QTY-ORDERED` เสมอ ซึ่งก็จะถูกปฏิเสธด้วยเหตุผล
"RECEIPT EXCEEDS ORDER QUANTITY" อยู่ดี **แต่เหตุผลนั้นไม่ตรงกับความจริง** — เหตุผลที่แท้จริงคือ
ใบสั่งซื้อนี้**ปิดไปแล้วตั้งแต่ต้น** ไม่ใช่แค่ "จำนวนที่รับครั้งนี้มากเกินไป" การแยกตรวจสอบสถานะก่อน
ทำให้ระบบให้เหตุผลที่ตรงประเด็นกับพนักงานคลังมากกว่า ซึ่งเป็นหลักการเดียวกับที่ Part 094 ขั้นตอนที่
935 อธิบายไว้เรื่องลำดับการตรวจสอบเงื่อนไขที่ต้องสะท้อนเหตุผลที่ถูกต้องด้วย ไม่ใช่แค่ผลลัพธ์ที่ถูกต้อง

---

## ขั้นตอนที่ 946: เต็มโค้ด `GRECEIPT`, การรัน, และวิเคราะห์ผลลัพธ์

### โครงสร้างไฟล์ธุรกรรมการรับสินค้า `GRECPT.DAT`

| ฟิลด์ | PICTURE | ความหมาย |
|---|---|---|
| `RCT-PO-NUMBER` | `X(8)` | เลขที่ใบสั่งซื้อที่กำลังรับสินค้า |
| `RCT-QTY-RECEIVED` | `9(7)` | จำนวนที่รับเข้าในครั้งนี้ |
| `RCT-RECEIPT-DATE` | `9(8)` | วันที่รับสินค้า |

### เต็มโค้ด `GRECEIPT.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. GRECEIPT.
      ******************************************************************
      * ERP / supply chain case study - goods receipt.
      * Reads GRECPT.DAT (sequential receipt transactions). Each
      * receipt is matched to an open/partial PO in PO-MASTER; a
      * receipt that would exceed the ordered quantity is rejected
      * outright (no partial application) so a PO's received quantity
      * never overshoots what was ordered. Accepted receipts add the
      * quantity to the matching STOCK-MASTER record (by warehouse +
      * product) and update the PO's received quantity and status.
      ******************************************************************
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT STOCK-MASTER ASSIGN TO "STOCK.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS STOCK-KEY
               FILE STATUS IS WS-STOCK-STATUS.

           SELECT PO-MASTER ASSIGN TO "POMASTER.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS PO-NUMBER
               FILE STATUS IS WS-PO-STATUS.

           SELECT RECEIPT-FILE ASSIGN TO "GRECPT.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-RCT-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  STOCK-MASTER.
       01  STOCK-RECORD.
           COPY "stockrec.cpy".

       FD  PO-MASTER.
       01  PO-RECORD.
           COPY "porec.cpy".

       FD  RECEIPT-FILE.
       01  RECEIPT-RECORD.
           05  RCT-PO-NUMBER           PIC X(8).
           05  RCT-QTY-RECEIVED        PIC 9(7).
           05  RCT-RECEIPT-DATE        PIC 9(8).

       WORKING-STORAGE SECTION.
       01  WS-STOCK-STATUS          PIC X(2).
       01  WS-PO-STATUS             PIC X(2).
       01  WS-RCT-STATUS            PIC X(2).
       01  WS-EOF                   PIC X(1) VALUE "N".
           88  END-OF-RECEIPTS                VALUE "Y".

       01  WS-NEW-TOTAL-RECEIVED    PIC 9(7).

       01  WS-COUNTERS.
           05  WS-APPLIED-COUNT     PIC 9(3) VALUE 0.
           05  WS-REJECTED-COUNT    PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       000-MAIN.
           OPEN I-O STOCK-MASTER
           OPEN I-O PO-MASTER
           OPEN INPUT RECEIPT-FILE

           DISPLAY "==== GOODS RECEIPT PROCESSING ===="

           PERFORM UNTIL END-OF-RECEIPTS
               READ RECEIPT-FILE
                   AT END
                       SET END-OF-RECEIPTS TO TRUE
                   NOT AT END
                       PERFORM 200-PROCESS-RECEIPT
               END-READ
           END-PERFORM

           CLOSE STOCK-MASTER
           CLOSE PO-MASTER
           CLOSE RECEIPT-FILE

           DISPLAY " "
           DISPLAY "RECEIPTS APPLIED  : " WS-APPLIED-COUNT
           DISPLAY "RECEIPTS REJECTED : " WS-REJECTED-COUNT
           STOP RUN.

       200-PROCESS-RECEIPT.
           MOVE RCT-PO-NUMBER TO PO-NUMBER
           READ PO-MASTER
               INVALID KEY
                   DISPLAY "REJECT " RCT-PO-NUMBER
                       " REASON: PO NOT FOUND"
                   ADD 1 TO WS-REJECTED-COUNT
               NOT INVALID KEY
                   PERFORM 210-VALIDATE-AND-APPLY
           END-READ.

       210-VALIDATE-AND-APPLY.
           IF PO-STATUS = "C" OR PO-STATUS = "X"
               DISPLAY "REJECT " RCT-PO-NUMBER
                   " REASON: PO ALREADY CLOSED/CANCELLED"
               ADD 1 TO WS-REJECTED-COUNT
           ELSE
               COMPUTE WS-NEW-TOTAL-RECEIVED =
                   PO-QTY-RECEIVED + RCT-QTY-RECEIVED
               IF WS-NEW-TOTAL-RECEIVED > PO-QTY-ORDERED
                   DISPLAY "REJECT " RCT-PO-NUMBER
                       " REASON: RECEIPT EXCEEDS ORDER QUANTITY"
                   ADD 1 TO WS-REJECTED-COUNT
               ELSE
                   PERFORM 220-APPLY-RECEIPT
               END-IF
           END-IF.

       220-APPLY-RECEIPT.
           MOVE WS-NEW-TOTAL-RECEIVED TO PO-QTY-RECEIVED
           IF PO-QTY-RECEIVED = PO-QTY-ORDERED
               MOVE "C" TO PO-STATUS
           ELSE
               MOVE "P" TO PO-STATUS
           END-IF
           REWRITE PO-RECORD

           MOVE PO-WH-CODE   TO STOCK-WH-CODE
           MOVE PO-PROD-CODE TO STOCK-PROD-CODE
           READ STOCK-MASTER
               INVALID KEY
                   DISPLAY "WARNING: STOCK RECORD MISSING FOR "
                       PO-WH-CODE "/" PO-PROD-CODE
               NOT INVALID KEY
                   ADD RCT-QTY-RECEIVED TO STOCK-QTY-ON-HAND
                   REWRITE STOCK-RECORD
           END-READ

           DISPLAY "APPLIED " RCT-PO-NUMBER " QTY "
               RCT-QTY-RECEIVED " NEW PO STATUS " PO-STATUS
               " STOCK NOW " STOCK-QTY-ON-HAND
           ADD 1 TO WS-APPLIED-COUNT.
```

### ข้อมูลธุรกรรมทดสอบ `GRECPT.DAT`

| ลำดับ | ใบสั่งซื้อ | จำนวนรับ | คาดว่าจะเป็น |
|---|---|---|---|
| 1 | PO000001 | 150 | รับครบเต็มจำนวน (สั่งไว้ 150) -> ปิดใบสั่งซื้อ |
| 2 | PO000002 | 30 | รับบางส่วน (สั่งไว้ 50) -> สถานะ Partial |
| 3 | PO000003 | 350 | ปฏิเสธ — เกินจำนวนที่สั่งไว้ (300) |
| 4 | PO999999 | 10 | ปฏิเสธ — ไม่พบใบสั่งซื้อ |

### คอมไพล์และรัน

```bash
/opt/gnucobol-isam/bin/cobc -x -o greceipt greceipt.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./greceipt
```

**ผลลัพธ์จริง**:

```
==== GOODS RECEIPT PROCESSING ====
APPLIED PO000001 QTY 0000150 NEW PO STATUS C STOCK NOW 0000180
APPLIED PO000002 QTY 0000030 NEW PO STATUS P STOCK NOW 0000050
REJECT PO000003 REASON: RECEIPT EXCEEDS ORDER QUANTITY
REJECT PO999999 REASON: PO NOT FOUND

RECEIPTS APPLIED  : 002
RECEIPTS REJECTED : 002
```

### วิเคราะห์ผลลัพธ์

- **PO000001**: WH02/P0001 มีสต๊อกเดิม 30 ชิ้น รับเพิ่ม 150 ชิ้น กลายเป็น **180 ชิ้น** ตรงกันทั้งสอง
  ไฟล์ (PO ปิดสถานะ C, สต๊อกเพิ่มขึ้นจริง)
- **PO000002**: WH02/P0004 มีสต๊อกเดิม 20 ชิ้น รับเพิ่ม 30 ชิ้น กลายเป็น **50 ชิ้น** — สังเกตว่าตัวเลข
  นี้เท่ากับจุดสั่งซื้อ (`STOCK-REORDER-POINT = 25`... รอ) พอดิบพอดีที่ 50 ซึ่งบังเอิญเท่ากับ
  `STOCK-REORDER-QTY` ไม่ใช่ reorder point — จะเห็นผลกระทบต่อรายงาน `REORDPT` ในขั้นตอนที่ 948
- **PO000003 และ PO999999**: ถูกปฏิเสธตามที่ออกแบบไว้ ไม่มีการแก้ไขสต๊อกหรือใบสั่งซื้อใด ๆ เลย

### ข้อควรระวัง

**ความสำคัญของลำดับการอัปเดตในย่อหน้าเดียว**: สังเกตว่าโค้ดอัปเดต `PO-RECORD` (ด้วย `REWRITE PO-RECORD`)
**ก่อน**อัปเดต `STOCK-RECORD` เสมอ ถ้าโปรแกรมเกิด error ระหว่างกลาง (เช่น เครื่องดับกะทันหัน) จะมี
สถานการณ์ที่ใบสั่งซื้อถูกปรับสถานะแล้วแต่สต๊อกยังไม่ได้อัปเดต ซึ่งเป็นความเสี่ยงที่มีอยู่จริงในระบบไฟล์
COBOL แบบดั้งเดิมที่ไม่มี Transaction/Rollback เหมือนฐานข้อมูลสมัยใหม่ ระบบระดับองค์กรจริงมักแก้ปัญหา
นี้ด้วย Audit Trail (ทบทวน Part 070 เรื่อง `TXNLOG.DAT`) ที่บันทึกทุกขั้นตอนแยกไว้ เพื่อให้สามารถ
ตรวจสอบและแก้ไขข้อมูลที่ไม่สอดคล้องกันได้ในภายหลัง หากมีเวลาและงบประมาณเพิ่มเติม ควรพิจารณาเพิ่ม
Audit Trail แบบเดียวกับ Part 070 เข้าไปในระบบ ERP นี้ด้วย

### แบบฝึกหัดที่ 946.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมยังคงแสดงข้อความ `APPLIED` และนับเป็นความสำเร็จ แม้ว่ากรณี
"WARNING: STOCK RECORD MISSING" จะเกิดขึ้น (คือกรณีที่ใบสั่งซื้ออ้างอิงคลัง+สินค้าที่ไม่มีอยู่ใน
`STOCK-MASTER` แล้ว ซึ่งไม่ควรเกิดขึ้นได้ในทางทฤษฎีเพราะ `PORDER` ตรวจสอบไปแล้วตอนสร้างใบสั่งซื้อ)

**เฉลยแนวทาง**: นี่คือจุดที่ออกแบบเป็น**การป้องกันเชิงรับ (defensive programming)** สำหรับสถานการณ์ที่
"ไม่ควรเกิดขึ้น" แต่ก็ยังเผื่อไว้ — เช่น หากมีคนลบ record ใน `STOCK-MASTER` ออกไปหลังจากสร้างใบสั่งซื้อ
ไปแล้ว (ผ่านการบำรุงรักษาข้อมูลโดยตรง ไม่ผ่านโปรแกรมที่ถูกต้อง) ในกรณีนี้การปรับปรุงสถานะใบสั่งซื้อ
(`PO-RECORD`) ยังคงเดินหน้าต่อไปได้อย่างถูกต้อง (สินค้าได้รับจริงตามใบสั่งซื้อ) เพียงแต่ไม่สามารถ
สะท้อนกลับไปที่สต๊อกได้ ซึ่งเป็นการตัดสินใจที่สมเหตุสมผล: **การรับสินค้าเป็นข้อเท็จจริงที่เกิดขึ้นแล้ว
ในโลกจริง (สินค้ามาถึงคลังจริง ๆ) ไม่ควรถูกปฏิเสธเพียงเพราะข้อมูลสต๊อกในระบบไม่สอดคล้องกัน** ควรบันทึก
คำเตือนไว้ให้ทีมงานตรวจสอบและแก้ไขข้อมูลสต๊อกในภายหลังแทน อย่างไรก็ตาม ในระบบจริงควรมีกลไกแจ้งเตือน
(alert) ที่เข้มงวดกว่าการ `DISPLAY` ข้อความบนหน้าจอเฉย ๆ เช่น เขียนลง log ไฟล์แยกต่างหากที่ทีมปฏิบัติการ
ตรวจสอบเป็นประจำ

---

## ขั้นตอนที่ 947: โปรแกรม `REORDPT` — SORT และ Control Break ข้ามคลังสินค้า

### ทำไมต้องใช้ SORT พร้อม INPUT PROCEDURE

ตามที่อธิบายไว้ในขั้นตอนที่ 941 ไฟล์ `STOCK.DAT` เรียงตาม (คลัง, สินค้า) แต่รายงานต้องการมุมมองแบบ
(สินค้า, คลัง) เทคนิคที่เหมาะสมที่สุดคือ `SORT` แบบมี **`INPUT PROCEDURE`** (ทบทวน Part 027) ซึ่งให้
เราเขียนย่อหน้าที่อ่าน `STOCK-MASTER` เองแล้ว `RELEASE` แต่ละ record เข้าสู่กระบวนการ sort โดยตรง
แทนที่จะต้องสร้างไฟล์ extract กลางไว้ก่อน:

```cobol
       000-MAIN.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SD-PROD-CODE
               ON ASCENDING KEY SD-WH-CODE
               INPUT PROCEDURE IS 100-RELEASE-STOCK
               GIVING SORTED-OUTPUT
```

`ON ASCENDING KEY SD-PROD-CODE` มาก่อน `SD-WH-CODE` หมายความว่าผลลัพธ์ที่ sort แล้วจะเรียงตาม
**รหัสสินค้าเป็นหลักก่อน แล้วค่อยเรียงตามคลังภายในสินค้าเดียวกัน** ซึ่งตรงกับมุมมองที่รายงานต้องการ
พอดี — นี่คือประโยชน์ของ `SORT` ที่ทำให้เราไม่ต้องเปลี่ยนโครงสร้าง key ของไฟล์หลักเลยแม้แต่นิดเดียว
เพียงแค่ "ขอมุมมองใหม่ชั่วคราว" สำหรับรายงานนี้โดยเฉพาะ

### ย่อหน้า `100-RELEASE-STOCK` — สะพานเชื่อมระหว่าง Indexed File กับ SORT

```cobol
       100-RELEASE-STOCK.
           OPEN INPUT STOCK-MASTER
           PERFORM UNTIL END-OF-STOCK
               READ STOCK-MASTER NEXT RECORD
                   AT END
                       SET END-OF-STOCK TO TRUE
                   NOT AT END
                       MOVE STOCK-PROD-CODE     TO SD-PROD-CODE
                       MOVE STOCK-WH-CODE       TO SD-WH-CODE
                       ...
                       RELEASE SD-RECORD
               END-READ
           END-PERFORM
           CLOSE STOCK-MASTER.
```

สังเกตว่า `INPUT PROCEDURE` ทำหน้าที่เหมือนโปรแกรมย่อยที่ถูกเรียกโดยอัตโนมัติจากคำสั่ง `SORT` เอง —
เราไม่ต้องเขียน `PERFORM 100-RELEASE-STOCK` ด้วยตนเองในที่อื่นเลย เพียงระบุชื่อย่อหน้าไว้ใน
`INPUT PROCEDURE IS` เท่านั้น การ `RELEASE` แต่ละ record เข้าสู่ SD (Sort Description) แทนการ `WRITE`
ปกติ เป็นรูปแบบมาตรฐานของ `SORT` ที่เรียนมาใน Part 027

### การออกแบบ Control Break สำหรับรายงานนี้

หลังจาก sort เสร็จแล้ว เราอ่านผลลัพธ์ (`SORTED-OUTPUT`) ทีละบรรทัดแบบ sequential และตรวจจับ "จุดเปลี่ยน"
(control break) ทุกครั้งที่รหัสสินค้าเปลี่ยนไปจาก record ก่อนหน้า — เทคนิคเดียวกับที่ Part 026 และ
Part 039 สอนไว้ แต่คราวนี้ break level มีเพียงชั้นเดียว (ตามสินค้า) ไม่ใช่หลายชั้นซ้อนกัน:

```cobol
       510-CHECK-BREAK.
           IF WS-FIRST-RECORD = "Y"
               MOVE "N" TO WS-FIRST-RECORD
               MOVE WS-SO-PROD-CODE TO WS-BREAK-PROD-CODE
               PERFORM 540-PRINT-PRODUCT-HEADER
           ELSE
               IF WS-SO-PROD-CODE NOT = WS-BREAK-PROD-CODE
                   PERFORM 530-PRINT-PRODUCT-TOTAL
                   MOVE WS-SO-PROD-CODE TO WS-BREAK-PROD-CODE
                   MOVE 0 TO WS-PROD-TOTAL-QTY
                   MOVE 0 TO WS-PROD-LOW-COUNT
                   PERFORM 540-PRINT-PRODUCT-HEADER
               END-IF
           END-IF.
```

### แบบฝึกหัดที่ 947.1

**โจทย์**: จงอธิบายว่าทำไมต้องมีตัวแปร `WS-FIRST-RECORD` แยกต่างหาก แทนที่จะเริ่มต้น `WS-BREAK-PROD-CODE`
ด้วยค่าที่ "เป็นไปไม่ได้" (เช่น `LOW-VALUES` หรือค่าว่าง) แล้วให้ `IF WS-SO-PROD-CODE NOT = WS-BREAK-PROD-CODE`
ทำงานตั้งแต่ record แรกไปเลย

**เฉลยแนวทาง**: ทั้งสองวิธีใช้ได้จริงและเป็นที่นิยมทั้งคู่ในโค้ด COBOL จริง การใช้ `WS-FIRST-RECORD`
(explicit flag) ทำให้**อ่านโค้ดเข้าใจง่ายกว่า**สำหรับคนที่ไม่คุ้นเคยกับรูปแบบนี้ เพราะสื่อความหมาย
ตรงตัวว่า "นี่คือ record แรก ให้พิมพ์หัวกระดาษโดยไม่ต้องพิมพ์ยอดรวมของกลุ่มก่อนหน้า (เพราะยังไม่มีกลุ่ม
ก่อนหน้า)" ในขณะที่การใช้ค่าเริ่มต้นแบบ "เป็นไปไม่ได้" ต้องอาศัยความเข้าใจว่าค่านั้นจะไม่มีทางตรงกับ
ข้อมูลจริงเสมอ ซึ่งเป็นสมมติฐานที่**อาจผิดพลาดได้**หากมีการเปลี่ยนแปลงรูปแบบของ key ในอนาคต (เช่น ถ้า
`SD-PROD-CODE` เปลี่ยนมารองรับค่า `LOW-VALUES` เป็นรหัสสินค้าจริงในบางกรณีพิเศษ) การใช้ flag ชัดเจน
จึงปลอดภัยกว่าในระยะยาว แม้จะต้องประกาศตัวแปรเพิ่มอีกหนึ่งตัวก็ตาม

---

## ขั้นตอนที่ 948: เต็มโค้ด `REORDPT`, การรัน, และวิเคราะห์ผลลัพธ์

### เต็มโค้ด `REORDPT.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. REORDPT.
      ******************************************************************
      * ERP / supply chain case study - reorder point trigger report.
      * Every STOCK-MASTER record is RELEASEd into a SORT (Part 027
      * technique) keyed by product then warehouse, so the report can
      * show, for each product, every warehouse location together with
      * a running total across locations - a view that the physical
      * file (keyed by warehouse + product) cannot give directly.
      * Any location below its reorder point is flagged for reorder.
      ******************************************************************
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT STOCK-MASTER ASSIGN TO "STOCK.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS STOCK-KEY
               FILE STATUS IS WS-STOCK-STATUS.

           SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK.TMP".

           SELECT SORTED-OUTPUT ASSIGN TO "SORTED.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-OUT-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  STOCK-MASTER.
       01  STOCK-RECORD.
           COPY "stockrec.cpy".

       SD  SORT-WORK-FILE.
       01  SD-RECORD.
           05  SD-PROD-CODE            PIC X(6).
           05  SD-WH-CODE              PIC X(4).
           05  SD-PROD-NAME            PIC X(20).
           05  SD-QTY                  PIC 9(7).
           05  SD-REORDER-POINT        PIC 9(7).
           05  SD-REORDER-QTY          PIC 9(7).
           05  SD-UNIT-COST            PIC 9(7)V99.

       FD  SORTED-OUTPUT.
       01  SORTED-RECORD               PIC X(60).

       WORKING-STORAGE SECTION.
       01  WS-STOCK-STATUS          PIC X(2).
       01  WS-OUT-STATUS            PIC X(2).
       01  WS-EOF                   PIC X(1) VALUE "N".
           88  END-OF-STOCK                   VALUE "Y".
       01  WS-EOF-SORTED            PIC X(1) VALUE "N".
           88  END-OF-SORTED                  VALUE "Y".

       01  WS-SORT-OUT-LINE.
           05  WS-SO-PROD-CODE          PIC X(6).
           05  WS-SO-WH-CODE            PIC X(4).
           05  WS-SO-PROD-NAME          PIC X(20).
           05  WS-SO-QTY                PIC 9(7).
           05  WS-SO-REORDER-POINT      PIC 9(7).
           05  WS-SO-REORDER-QTY        PIC 9(7).
           05  WS-SO-UNIT-COST          PIC 9(7)V99.

       01  WS-FIRST-RECORD          PIC X(1) VALUE "Y".
       01  WS-BREAK-PROD-CODE       PIC X(6).
       01  WS-PROD-TOTAL-QTY        PIC 9(8) VALUE 0.
       01  WS-PROD-LOW-COUNT        PIC 9(3) VALUE 0.

       01  WS-GRAND-LOCATIONS       PIC 9(4) VALUE 0.
       01  WS-GRAND-LOW-COUNT       PIC 9(4) VALUE 0.
       01  WS-GRAND-PRODUCT-COUNT   PIC 9(3) VALUE 0.

       01  WS-DETAIL-LINE.
           05  FILLER                PIC X(4) VALUE SPACES.
           05  WS-D-WH               PIC X(4).
           05  FILLER                PIC X(2) VALUE SPACES.
           05  WS-D-QTY              PIC ZZZ,ZZ9.
           05  FILLER                PIC X(2) VALUE SPACES.
           05  WS-D-ROP              PIC ZZZ,ZZ9.
           05  FILLER                PIC X(2) VALUE SPACES.
           05  WS-D-FLAG             PIC X(24).

       PROCEDURE DIVISION.
       000-MAIN.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SD-PROD-CODE
               ON ASCENDING KEY SD-WH-CODE
               INPUT PROCEDURE IS 100-RELEASE-STOCK
               GIVING SORTED-OUTPUT

           PERFORM 500-PRINT-REPORT
           STOP RUN.

       100-RELEASE-STOCK.
           OPEN INPUT STOCK-MASTER
           IF WS-STOCK-STATUS NOT = "00"
               DISPLAY "ERROR OPENING STOCK-MASTER: " WS-STOCK-STATUS
               STOP RUN
           END-IF

           PERFORM UNTIL END-OF-STOCK
               READ STOCK-MASTER NEXT RECORD
                   AT END
                       SET END-OF-STOCK TO TRUE
                   NOT AT END
                       MOVE STOCK-PROD-CODE     TO SD-PROD-CODE
                       MOVE STOCK-WH-CODE       TO SD-WH-CODE
                       MOVE STOCK-PROD-NAME     TO SD-PROD-NAME
                       MOVE STOCK-QTY-ON-HAND   TO SD-QTY
                       MOVE STOCK-REORDER-POINT TO SD-REORDER-POINT
                       MOVE STOCK-REORDER-QTY   TO SD-REORDER-QTY
                       MOVE STOCK-UNIT-COST     TO SD-UNIT-COST
                       RELEASE SD-RECORD
               END-READ
           END-PERFORM

           CLOSE STOCK-MASTER.

       500-PRINT-REPORT.
           OPEN INPUT SORTED-OUTPUT
           DISPLAY "==== REORDER POINT TRIGGER REPORT ===="
           DISPLAY "(all warehouses, grouped by product)"
           DISPLAY " "

           PERFORM UNTIL END-OF-SORTED
               READ SORTED-OUTPUT
                   AT END
                       SET END-OF-SORTED TO TRUE
                   NOT AT END
                       MOVE SORTED-RECORD TO WS-SORT-OUT-LINE
                       PERFORM 510-CHECK-BREAK
                       PERFORM 520-PRINT-DETAIL
               END-READ
           END-PERFORM

           IF WS-FIRST-RECORD = "N"
               PERFORM 530-PRINT-PRODUCT-TOTAL
           END-IF

           CLOSE SORTED-OUTPUT
           PERFORM 600-PRINT-GRAND-SUMMARY.

       510-CHECK-BREAK.
           IF WS-FIRST-RECORD = "Y"
               MOVE "N" TO WS-FIRST-RECORD
               MOVE WS-SO-PROD-CODE TO WS-BREAK-PROD-CODE
               PERFORM 540-PRINT-PRODUCT-HEADER
           ELSE
               IF WS-SO-PROD-CODE NOT = WS-BREAK-PROD-CODE
                   PERFORM 530-PRINT-PRODUCT-TOTAL
                   MOVE WS-SO-PROD-CODE TO WS-BREAK-PROD-CODE
                   MOVE 0 TO WS-PROD-TOTAL-QTY
                   MOVE 0 TO WS-PROD-LOW-COUNT
                   PERFORM 540-PRINT-PRODUCT-HEADER
               END-IF
           END-IF.

       540-PRINT-PRODUCT-HEADER.
           DISPLAY " "
           DISPLAY "PRODUCT " WS-SO-PROD-CODE " - " WS-SO-PROD-NAME
           ADD 1 TO WS-GRAND-PRODUCT-COUNT.

       520-PRINT-DETAIL.
           MOVE WS-SO-WH-CODE  TO WS-D-WH
           MOVE WS-SO-QTY      TO WS-D-QTY
           MOVE WS-SO-REORDER-POINT TO WS-D-ROP
           IF WS-SO-QTY < WS-SO-REORDER-POINT
               MOVE "*** REORDER NEEDED ***" TO WS-D-FLAG
               ADD 1 TO WS-PROD-LOW-COUNT
               ADD 1 TO WS-GRAND-LOW-COUNT
           ELSE
               MOVE "OK" TO WS-D-FLAG
           END-IF
           DISPLAY WS-DETAIL-LINE
           ADD WS-SO-QTY TO WS-PROD-TOTAL-QTY
           ADD 1 TO WS-GRAND-LOCATIONS.

       530-PRINT-PRODUCT-TOTAL.
           DISPLAY "  ---- TOTAL ACROSS WAREHOUSES: "
               WS-PROD-TOTAL-QTY
               "   LOCATIONS NEEDING REORDER: "
               WS-PROD-LOW-COUNT.

       600-PRINT-GRAND-SUMMARY.
           DISPLAY " "
           DISPLAY "==== SUMMARY ===="
           DISPLAY "PRODUCTS REVIEWED         : " WS-GRAND-PRODUCT-COUNT
           DISPLAY "TOTAL WAREHOUSE LOCATIONS : " WS-GRAND-LOCATIONS
           DISPLAY "LOCATIONS NEEDING REORDER : " WS-GRAND-LOW-COUNT.
```

### คอมไพล์และรัน (หลังจากรัน `ERPSETUP` -> `PORDER` -> `GRECEIPT` ตามลำดับแล้ว)

```bash
/opt/gnucobol-isam/bin/cobc -x -o reordpt reordpt.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./reordpt
```

**ผลลัพธ์จริง**:

```
==== REORDER POINT TRIGGER REPORT ====
(all warehouses, grouped by product)


PRODUCT P0001  - WIDGET-A
    WH01      150      100  OK
    WH02      180       80  OK
    WH03      200      100  OK
  ---- TOTAL ACROSS WAREHOUSES: 00000530   LOCATIONS NEEDING REORDER: 000

PRODUCT P0002  - GADGET-B
    WH01       40       50  *** REORDER NEEDED ***
    WH02       90       60  OK
  ---- TOTAL ACROSS WAREHOUSES: 00000130   LOCATIONS NEEDING REORDER: 001

PRODUCT P0003  - BOLT-C
    WH01      500      200  OK
    WH03       60      150  *** REORDER NEEDED ***
  ---- TOTAL ACROSS WAREHOUSES: 00000560   LOCATIONS NEEDING REORDER: 001

PRODUCT P0004  - CASING-D
    WH02       50       25  OK
    WH03       75       25  OK
  ---- TOTAL ACROSS WAREHOUSES: 00000125   LOCATIONS NEEDING REORDER: 000

==== SUMMARY ====
PRODUCTS REVIEWED         : 004
TOTAL WAREHOUSE LOCATIONS : 0009
LOCATIONS NEEDING REORDER : 0002
```

### วิเคราะห์ผลลัพธ์

- **WH02/P0001** เพิ่มจาก 30 เป็น **180** เพราะการรับสินค้าตาม PO000001 ใน Step 946 — รายงานนี้
  สะท้อนสต๊อกล่าสุดหลังการรับสินค้าเรียบร้อยแล้ว ยืนยันว่าข้อมูลไหลจาก `GRECEIPT` มาถึง `REORDPT`
  ถูกต้อง
- **WH02/P0004** เพิ่มจาก 20 เป็น **50** ซึ่งเท่ากับจุดสั่งซื้อ (`STOCK-REORDER-POINT = 25`)... ที่จริง
  50 มากกว่า 25 (ไม่ใช่เท่ากัน) จึงแสดงเป็น "OK" — นี่คือตัวอย่างที่ดีของ**เงื่อนไขขอบเขต (boundary
  condition)**: โค้ดใช้ `IF WS-SO-QTY < WS-SO-REORDER-POINT` (น้อยกว่าอย่างเคร่งครัด) ดังนั้นถ้าสต๊อก
  เท่ากับจุดสั่งซื้อ**พอดี**จะยังถือว่า "OK" ไม่ต้องสั่งซื้อ ต้องน้อยกว่าจริง ๆ เท่านั้นจึงจะติดธง
- **WH01/P0002** (40<50) และ **WH03/P0003** (60<150) ยังคงติดธง "REORDER NEEDED" เหมือนเดิมเพราะไม่มี
  การรับสินค้าเข้าคลังทั้งสองนี้ในรอบนี้

### ข้อควรระวัง

**เงื่อนไข `<` กับ `<=` ส่งผลต่อผลลัพธ์ทางธุรกิจโดยตรง**: หากธุรกิจต้องการให้ "สต๊อกเท่ากับจุดสั่งซื้อ
พอดีก็ควรสั่งซื้อแล้ว" (แนวทางระมัดระวังกว่า) ต้องเปลี่ยนเงื่อนไขเป็น `IF WS-SO-QTY <= WS-SO-REORDER-POINT`
แทน การเลือกใช้ `<` หรือ `<=` ดูเหมือนเป็นรายละเอียดเล็กน้อยมาก แต่ในทางปฏิบัติจริงอาจหมายถึงความ
แตกต่างระหว่าง "มีสินค้าพอขายในสัปดาห์หน้า" กับ "สินค้าหมดกะทันหัน" ทีมพัฒนาต้องยืนยันกับฝ่ายธุรกิจ
เสมอว่าจุดสั่งซื้อ (reorder point) หมายถึง "สั่งซื้อเมื่อสต๊อกลดลง**ต่ำกว่า**นี้" หรือ "สั่งซื้อเมื่อ
สต๊อกลดลง**มาถึง**นี้" ก่อนเขียนโค้ดจริง

### แบบฝึกหัดที่ 948.1

**โจทย์**: สมมติต้องการเพิ่มคอลัมน์ "มูลค่าสต๊อกรวม" (Qty x Unit Cost) ต่อแต่ละแถวในรายงานนี้ ต้องแก้ไข
ส่วนใดของโปรแกรมบ้าง

**เฉลยแนวทาง**: ต้องแก้ 3 จุด: (1) เพิ่มฟิลด์ตัวเลขใหม่ เช่น `WS-D-VALUE PIC ZZZ,ZZZ,ZZ9.99` ใน
`WS-DETAIL-LINE`, (2) ในย่อหน้า `520-PRINT-DETAIL` เพิ่ม `COMPUTE WS-D-VALUE = WS-SO-QTY * WS-SO-UNIT-COST`
ก่อนคำสั่ง `DISPLAY WS-DETAIL-LINE`, และ (3) ถ้าต้องการยอดรวมมูลค่าต่อสินค้าและยอดรวมทั้งหมดด้วย ต้อง
เพิ่มตัวแปรสะสม (accumulator) อีกสองตัวคล้ายกับ `WS-PROD-TOTAL-QTY` และ `WS-GRAND-LOCATIONS` แล้วบวก
เข้าไปในย่อหน้าเดียวกันกับที่บวก `WS-SO-QTY` อยู่แล้ว — สังเกตว่าไม่ต้องแก้ไข `SD-RECORD` หรือ
`WS-SORT-OUT-LINE` เลย เพราะ `SD-UNIT-COST`/`WS-SO-UNIT-COST` ถูกส่งผ่านกระบวนการ SORT ไว้อยู่แล้ว
ตั้งแต่ต้น แม้จะยังไม่ได้ใช้งานในรายงานเดิมก็ตาม — นี่คือประโยชน์ของการออกแบบ SD record ให้มีข้อมูล
ครบถ้วนตั้งแต่แรก แม้จะยังไม่ได้ใช้ทุกฟิลด์ในเวอร์ชันแรกของรายงาน

---

## ขั้นตอนที่ 949: ทดสอบทั้งระบบแบบครบวงจร (End-to-End Integration Test)

### ลำดับการรันที่ถูกต้อง

```bash
cd /tmp/erp-test
/opt/gnucobol-isam/bin/cobc -x -o erpsetup erpsetup.cob
/opt/gnucobol-isam/bin/cobc -x -o porder   porder.cob
/opt/gnucobol-isam/bin/cobc -x -o greceipt greceipt.cob
/opt/gnucobol-isam/bin/cobc -x -o reordpt  reordpt.cob

export LD_LIBRARY_PATH=/opt/gnucobol-isam/lib
rm -f STOCK.DAT* POMASTER.DAT* POCTL.DAT SORTED.DAT SORTWORK.TMP
./erpsetup    # 1. สร้างสต๊อกตั้งต้น 9 รายการ, ใบสั่งซื้อเปล่า, ตัวนับเริ่มที่ 0
./porder      # 2. สร้างใบสั่งซื้อ 3 ใบสำเร็จ, ปฏิเสธ 1 คำขอ
./greceipt    # 3. รับสินค้า 2 รายการสำเร็จ, ปฏิเสธ 2 รายการ
./reordpt     # 4. รายงานจุดสั่งซื้อซ้ำข้ามทุกคลัง
```

### ตรวจสอบความสอดคล้องของข้อมูลแบบ end-to-end (Data Integrity Trace)

| จุดตรวจสอบ | ค่าที่คาดหวัง | ผลจริง | สรุป |
|---|---|---|---|
| จำนวนสต๊อกเริ่มต้นทั้งหมด | 9 รายการ | `ERPSETUP COMPLETE - STOCK RECORDS LOADED: 009` | ตรงกัน |
| ใบสั่งซื้อที่สร้างสำเร็จ | 3 ใบ (`PO000001`-`PO000003`) | `PURCHASE ORDERS CREATED : 003` | ตรงกัน |
| สต๊อก WH02/P0001 หลังรับ PO000001 | 30 + 150 = 180 | รายงาน `REORDPT` แสดง 180 | ตรงกัน |
| สต๊อก WH02/P0004 หลังรับ PO000002 บางส่วน | 20 + 30 = 50 | รายงาน `REORDPT` แสดง 50 | ตรงกัน |
| สต๊อกที่ไม่ถูกแตะต้องเลย (เช่น WH01/P0001) | 150 (ค่าตั้งต้น ไม่เปลี่ยน) | รายงาน `REORDPT` แสดง 150 | ตรงกัน |
| จำนวนตำแหน่งที่ต้องสั่งซื้อซ้ำ | 2 (WH01/P0002, WH03/P0003) — ทั้งสองไม่ได้รับสินค้าในรอบนี้ | `LOCATIONS NEEDING REORDER : 0002` | ตรงกัน |

การตรวจสอบทีละจุดนี้ยืนยันว่าข้อมูลไหลผ่านทั้ง 4 โปรแกรมถูกต้องแบบ end-to-end: การสร้างใบสั่งซื้อใน
`PORDER` ตรวจสอบกับสต๊อกที่ `ERPSETUP` สร้างไว้ได้ถูกต้อง, การรับสินค้าใน `GRECEIPT` อัปเดตทั้งใบสั่งซื้อ
และสต๊อกให้สอดคล้องกัน, และ `REORDPT` อ่านสต๊อกล่าสุดหลังการอัปเดตทั้งหมดมาสร้างรายงานที่ถูกต้อง

### แบบฝึกหัดที่ 949.1

**โจทย์**: หากรัน `porder` อีกครั้งพร้อมไฟล์ `POREQ.DAT` เดิม (ไม่เปลี่ยนแปลงคำขอ) จะเกิดอะไรขึ้นกับ
เลขที่ใบสั่งซื้อที่ได้ และทำไม

**เฉลย**: จะได้ใบสั่งซื้อเลขที่ใหม่ต่อเนื่องจากเดิม คือ `PO000004`, `PO000005`, `PO000006` (คำขอที่ 4
ยังคงถูกปฏิเสธเหมือนเดิมเพราะ P0099 ยังไม่มีในสต๊อก) **ไม่ใช่** `PO000001` ซ้ำเดิม เพราะ `POCTL.DAT`
ถูกบันทึกค่าล่าสุด (3) ไว้จากการรันครั้งแรกแล้ว เมื่อรันครั้งที่สอง โปรแกรมจะอ่านค่า 3 ขึ้นมาแล้วนับ
ต่อจากตรงนั้น นี่คือพฤติกรรมที่ถูกต้องสำหรับระบบสร้างเลขที่เอกสารอัตโนมัติ — แต่ก็หมายความว่าถ้าต้อง
การทดสอบซ้ำโดยได้ผลลัพธ์เหมือนเดิมทุกประการ (reproducible) ต้องรีเซ็ต `POCTL.DAT` กลับเป็น 0 ก่อน
(หรือรัน `ERPSETUP` ใหม่ทั้งหมด) เช่นเดียวกับที่ต้องลบ `STOCK.DAT`/`POMASTER.DAT` ก่อนทดสอบซ้ำ

---

## ขั้นตอนที่ 950: สรุปโปรเจกต์, ข้อจำกัดที่ตั้งใจไว้, และแบบฝึกหัดขยายผล

### สิ่งที่ระบบนี้ทำได้จริง 100%

- จัดเก็บและติดตามสต๊อกสินค้าแยกตามคลังด้วย composite key บน Indexed File จริง
- สร้างใบสั่งซื้อพร้อมเลขที่อัตโนมัติผ่านรูปแบบไฟล์ตัวนับ พร้อมตรวจสอบว่าสินค้า/คลังมีอยู่จริง
- รับสินค้าเข้าคลังพร้อมอัปเดตทั้งสถานะใบสั่งซื้อและจำนวนสต๊อกให้สอดคล้องกันเสมอ ป้องกันการรับเกิน
- สร้างรายงานจุดสั่งซื้อซ้ำข้ามคลังด้วยเทคนิค `SORT`+`INPUT PROCEDURE`+Control Break อย่างสมบูรณ์

### สิ่งที่ระบบนี้**ตั้งใจไม่ทำ** (เพื่อคุมขอบเขตให้กระชับตามที่ตั้งใจไว้)

| ไม่ได้ทำ | เหตุผล / ทางเลือกในระบบจริง |
|---|---|
| การโอนย้ายสต๊อกระหว่างคลัง (Stock Transfer) | เป็นกระบวนการที่ซับซ้อนกว่านี้ (ต้องหักออกจากคลังต้นทางและบวกเข้าคลังปลายทางแบบ atomic) เหมาะเป็นหัวข้อขยายผลแยกต่างหาก |
| การสร้างใบสั่งซื้ออัตโนมัติเมื่อถึงจุดสั่งซื้อ (Auto-PO from Reorder Report) | `REORDPT` ในบทนี้เป็นรายงานให้มนุษย์ตัดสินใจเท่านั้น ยังไม่เชื่อมต่อกลับไปสร้างใบสั่งซื้อให้อัตโนมัติ ซึ่งเป็นส่วนขยายที่สมเหตุสมผลมาก (ดูแบบฝึกหัดที่ 950.1) |
| การจัดการผู้จำหน่าย (Supplier Master) แยกต่างหาก | ในระบบนี้ `PO-SUPPLIER-CODE` เป็นแค่รหัสอ้างอิง ไม่มีไฟล์ผู้จำหน่ายแยกให้ตรวจสอบว่ารหัสนั้นมีอยู่จริงหรือไม่ |
| ราคาต้นทุนถัวเฉลี่ยเคลื่อนที่ (Moving Average Cost) เมื่อรับสินค้าที่ต้นทุนต่างจากเดิม | เป็นหัวข้อการบัญชีต้นทุนที่ลึกกว่าขอบเขตของ Case Study นี้ ระบบปัจจุบันเก็บแค่ต้นทุนต่อหน่วยคงที่ |

### เชื่อมโยงกับหลักสูตรทั้งหมด

- **Part 035** เป็นจุดตั้งต้นของแนวคิดคลังสินค้า ซึ่ง Part นี้ขยายให้รองรับหลายคลัง
- **Part 027** (SORT/MERGE) และ **Part 026/039** (Master-Detail, Control Break) เป็นหัวใจของ `REORDPT`
- **Part 028** (Indexed Files) ขยายไปสู่แนวคิด composite key ในบทนี้
- **Part 094** (Insurance) และ Part นี้ (ERP) ร่วมกันแสดงให้เห็นว่าเทคนิคเดียวกัน (Indexed File,
  Copybook, validation cascade, error handling แบบไม่ล่ม) นำไปประยุกต์ใช้ได้กับอุตสาหกรรมที่แตกต่าง
  กันโดยสิ้นเชิง — นี่คือพลังของการเรียนรู้ "หลักการ" มากกว่าแค่ "ตัวอย่างเฉพาะเจาะจง"

### แบบฝึกหัดที่ 950.1 (แบบฝึกหัดขยายผลปิดท้าย Part)

**โจทย์**: จงออกแบบ (เชิงโครงสร้าง ไม่ต้องเขียนโค้ดเต็ม) ว่าจะขยาย `REORDPT` ให้กลายเป็น
`AUTOPO` — โปรแกรมที่**สร้างใบสั่งซื้ออัตโนมัติ**ทันทีที่พบตำแหน่งต่ำกว่าจุดสั่งซื้อ (โดยใช้จำนวน
`STOCK-REORDER-QTY` เป็นจำนวนที่จะสั่ง) แทนที่จะแค่พิมพ์รายงานเฉย ๆ — ต้องเปลี่ยนแปลงอะไรบ้าง

**เฉลยแนวทาง**: แนวทางที่ดีที่สุดคือ**ไม่เขียน `REORDPT` ใหม่ทั้งหมด** แต่ปรับ `520-PRINT-DETAIL` ให้
นอกจากจะพิมพ์ธง "REORDER NEEDED" แล้ว ยังเรียก logic การสร้างใบสั่งซื้อที่มีอยู่แล้วใน `PORDER`
(ย่อหน้า `210-CREATE-PO`) ไปด้วย — ในทางปฏิบัติหมายความว่าต้อง (1) เพิ่ม `SELECT PO-MASTER` และ
`SELECT PO-COUNTER-FILE` เข้าไปใน `FILE-CONTROL` ของ `REORDPT` (หรือแยกเป็นโปรแกรมใหม่ชื่อ `AUTOPO`
ที่ทำหน้าที่นี้แทน ตามหลัก Separation of Concerns ที่เน้นย้ำมาตลอดหลักสูตร), (2) เมื่อพบ
`WS-SO-QTY < WS-SO-REORDER-POINT` ให้เรียกย่อหน้าที่เทียบเท่า `210-CREATE-PO` โดยใช้
`WS-SO-REORDER-QTY` เป็นจำนวนสั่งซื้อ และกำหนดผู้จำหน่ายเริ่มต้น (default supplier) เนื่องจากรายงาน
เดิมไม่มีข้อมูลผู้จำหน่ายที่ควรสั่งจาก และ (3) ต้องพิจารณาว่าจะเกิดอะไรขึ้นถ้าตำแหน่งเดียวกันมี
ใบสั่งซื้อที่ยังเปิดอยู่แล้ว (สถานะ `O`/`P`) — ไม่ควรสร้างใบสั่งซื้อซ้ำซ้อนสำหรับสินค้าที่กำลังรออยู่
แล้ว ซึ่งต้องเพิ่มการตรวจสอบใบสั่งซื้อที่เปิดอยู่ของคลัง+สินค้านั้นก่อนสร้างใบใหม่เสมอ — นี่คือตัวอย่าง
ที่ดีของการที่ "รายงาน" (read-only) กับ "โปรแกรมที่แก้ไขข้อมูล" (read-write) มีความรับผิดชอบและความ
เสี่ยงต่างกันมาก แม้จะใช้ logic การตรวจสอบเงื่อนไขตัวเดียวกันก็ตาม

---

## สรุปท้ายบท

ใน Part นี้ เราได้สร้าง Case Study ระบบ ERP และห่วงโซ่อุปทานที่ทำงานได้จริงสมบูรณ์แบบด้วย 4 โปรแกรม:

- **`ERPSETUP`**: สร้างสต๊อกสินค้าหลักแบบ Indexed พร้อม **composite key** (คลัง+สินค้า) ที่ขยาย
  แนวคิดคลังเดียวจาก Part 035 ให้รองรับหลายคลังพร้อมกัน
- **`PORDER`**: ประมวลผลคำขอสั่งซื้อพร้อมเทคนิคไฟล์ตัวนับสำหรับสร้างเลขที่เอกสารอัตโนมัติที่ไม่ซ้ำกัน
- **`GRECEIPT`**: บันทึกการรับสินค้า พร้อมอัปเดตทั้งสถานะใบสั่งซื้อและสต๊อกให้สอดคล้องกันเสมอ
  ป้องกันการรับสินค้าเกินจำนวนที่สั่งไว้
- **`REORDPT`**: รายงานจุดสั่งซื้อซ้ำข้ามคลังทั้งหมดด้วยเทคนิค `SORT`+`INPUT PROCEDURE`+Control Break
  ที่เปลี่ยนมุมมองข้อมูลจาก (คลัง,สินค้า) เป็น (สินค้า,คลัง) โดยไม่ต้องแก้โครงสร้างไฟล์หลัก

ทุกโปรแกรมผ่านการคอมไพล์และรันทดสอบจริงด้วย GnuCOBOL (ISAM build) ครบทุกกรณี พร้อมการตรวจสอบความ
ถูกต้องของข้อมูลแบบ end-to-end ที่ยืนยันว่าทั้งระบบทำงานสอดคล้องกันตลอดทั้ง pipeline

Case Study ทั้งสอง (ประกันภัยใน Part 094 และ ERP/Supply Chain ใน Part นี้) ร่วมกับ Core Banking ใน
Part 091-093 แสดงให้เห็นว่าเทคนิค COBOL ชุดเดียวกัน (Indexed File, Copybook, SORT, Control Break,
Date Functions, Error Handling) สามารถนำไปแก้ปัญหาทางธุรกิจที่หลากหลายได้จริง ต่อไปหลักสูตรจะเปลี่ยน
ทิศทางจาก "การสร้างระบบ" ไปสู่ "การเตรียมตัวเข้าสู่วงการวิชาชีพ COBOL" ใน Part 096-097

**[ไปยัง Part 096: การเตรียมตัวสอบใบรับรอง COBOL และแนวทางอาชีพ →](part-096-certification-career.md)**

---

*Part นี้อยู่ในเฟส 6: ระดับมืออาชีพ/โลก (Parts 086-100)*
*Part ก่อนหน้า: [Part 094: Case Study — ระบบประกันภัย (Insurance System)](part-094-insurance-case-study.md)*
*Part ถัดไป: [Part 096: การเตรียมตัวสอบใบรับรอง COBOL และแนวทางอาชีพ](part-096-certification-career.md)*
*ลำดับ Case Study เฟส 6: Part 091-093 (Core Banking) -> Part 094 (Insurance) -> Part 095 (ERP/Supply Chain, Part นี้) -> Part 096 (Certification/Career) -> Part 097 (Interview Questions) -> Part 098 (Best Practices)*
