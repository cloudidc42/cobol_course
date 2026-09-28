# Part 050: 🎯 โปรเจกต์เฟส 3: ระบบบัญชีลูกหนี้ (Accounts Receivable System) (ขั้นตอนที่ 491–500)

## คำนำของ Part นี้

เดินทางมาถึงจุดสำคัญอีกครั้ง! ตั้งแต่ Part 036 ถึง Part 049 คุณได้เรียนรู้เครื่องมือขั้นสูงของ COBOL
จำนวนมาก: Intrinsic Functions สำหรับตัวเลข/วันที่/ข้อความ (Part 036-037), Report Writer (Part
038-039), Screen Section (Part 040), Advanced EVALUATE และ Class/Sign Condition (Part 041-042),
Dynamic CALL และ Based Storage/POINTER (Part 043-044), JSON/XML (Part 045-046), การจัดการวันที่เวลา
ขั้นสูง (Part 047), Error Handling ด้วย DECLARATIVES (Part 048), และ Compiler Directives/Dialects
(Part 049) — รวมกับพื้นฐานจากเฟส 1-2 ทั้งหมด (ไฟล์ Indexed จาก Part 028, Subprogram จาก Part 031-032,
Copybook จาก Part 033)

Part นี้คือ **โปรเจกต์รวบยอดเฟส 3 (Phase 3 Milestone Project)** ซึ่งจะนำความรู้ทั้งหมดข้างต้นมาประกอบ
ร่างเป็น **ระบบบัญชีลูกหนี้ (Accounts Receivable System)** ที่ทำงานได้จริงสมบูรณ์แบบ — ระบบประเภทนี้
เป็นหัวใจสำคัญของทุกองค์กรที่ขายสินค้า/บริการแบบให้เครดิต (เช่น "จ่ายภายใน 30 วัน") เพราะต้องติดตามว่า
ลูกค้าคนไหนค้างชำระเท่าไหร่ นานแค่ไหนแล้ว และมีความเสี่ยงที่จะไม่ได้รับชำระมากน้อยเพียงใด

### ระบบที่จะสร้างประกอบด้วย 3 โปรแกรมหลัก + 1 Subprogram

1. **`ARSETUP`** — โปรแกรมเริ่มต้นระบบ สร้างไฟล์ลูกค้าหลัก (Customer Master) และไฟล์ใบแจ้งหนี้
   (Invoice File) แบบ Indexed พร้อมข้อมูลตั้งต้น
2. **`ARPAYMNT`** — โปรแกรมบันทึกการรับชำระเงิน อ่านไฟล์ธุรกรรมการชำระเงิน นำไปหักลบยอดค้างของใบแจ้งหนี้
   และปรับยอดคงเหลือของลูกค้า พร้อมใช้ `DECLARATIVES` (เทคนิคจาก Part 048) จัดการข้อผิดพลาดแบบรวมศูนย์
3. **`AGINGCLC`** — Subprogram คำนวณอายุหนี้ (Aging Calculation) รับวันที่ใบแจ้งหนี้และวันที่รันรายงาน
   คืนค่าจำนวนวันที่ผ่านไปและช่วงอายุหนี้ (bucket) กลับมา ใช้เทคนิค `CALL` จาก Part 031/043 และ
   Intrinsic Function ด้านวันที่จาก Part 036/047
4. **`ARREPORT`** — โปรแกรมสร้างรายงานวิเคราะห์อายุหนี้ (Aging Report) พร้อม Control Break แยกตาม
   ลูกค้า (เทคนิคจาก Part 026/039) แสดงยอดค้างแบ่งเป็น 4 ช่วง: **ปัจจุบัน (Current), 31-60 วัน, 61-90
   วัน, และเกิน 90 วัน (91+)**

ทั้งหมดนี้เชื่อมกันด้วย **Copybook** (`CUSTREC.CPY`, `INVREC.CPY` จากเทคนิค Part 033) เพื่อให้โครงสร้าง
ข้อมูลของไฟล์เดียวกันสอดคล้องกันทุกโปรแกรมโดยอัตโนมัติ

### หมายเหตุสำคัญเรื่องการทดสอบและสภาพแวดล้อม

**Indexed File**: ระบบนี้ใช้ `ORGANIZATION IS INDEXED` เป็นหัวใจหลักของทั้งไฟล์ลูกค้าและไฟล์ใบแจ้งหนี้
ตามที่ Part 028 อธิบายไว้ **GnuCOBOL แบบมาตรฐานที่ติดตั้งจากตัวจัดการแพ็กเกจของระบบปฏิบัติการในเครื่องนี้
ปิดการรองรับ Indexed File ไว้** (ตรวจสอบแล้วด้วย `cobc -info` พบว่า `indexed file handler: disabled`)
โปรเจกต์นี้ทั้งหมดจึงคอมไพล์ด้วย **GnuCOBOL build ที่เปิดใช้ ISAM handler (Berkeley DB)** ที่ติดตั้งไว้ที่
`/opt/gnucobol-isam/` ตามที่ Part 028 ได้อธิบายวิธีตรวจสอบและแก้ปัญหานี้ไว้แล้ว คำสั่งคอมไพล์และรัน
มาตรฐานที่ใช้ตลอดทั้ง Part นี้คือ:

```bash
/opt/gnucobol-isam/bin/cobc -I copybooks -x -o <program-name> <program-name>.cob [subprogram.o]
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./<program-name>
```

**ทุกโปรแกรมและทุกผลลัพธ์ในเอกสารนี้ผ่านการคอมไพล์และรันทดสอบจริง** ด้วยชุดข้อมูลตัวอย่างเดียวกันตลอด
ทั้ง Part เพื่อให้ผู้อ่านติดตามการเปลี่ยนแปลงของข้อมูลได้อย่างต่อเนื่องจากขั้นตอนหนึ่งไปอีกขั้นตอนหนึ่ง
โปรแกรมทั้งหมดในเอกสารนี้**ไม่มีการรับข้อมูลผ่าน `ACCEPT` แบบโต้ตอบเลย** (ต่างจากโปรเจกต์เฟส 1 ใน
Part 015 และบางส่วนของเฟส 2 ใน Part 035) เพราะโปรแกรม batch ระดับองค์กรอย่างระบบบัญชีลูกหนี้จริงมักถูก
ออกแบบให้รันแบบอัตโนมัติผ่าน Job Scheduler ไม่ใช่รอคนพิมพ์ค่าเข้าไปทีละค่า — ไฟล์ธุรกรรม (เช่นไฟล์การ
ชำระเงิน) จะถูกเตรียมไว้ล่วงหน้าแล้วให้โปรแกรมอ่านผ่าน `LINE SEQUENTIAL` ตามที่จะสาธิตในขั้นตอนที่ 495

---

## ขั้นตอนที่ 491: การวิเคราะห์ความต้องการและออกแบบสถาปัตยกรรมระบบ

### ระบบบัญชีลูกหนี้ (Accounts Receivable) คืออะไร

เมื่อธุรกิจขายสินค้าหรือบริการให้ลูกค้าแบบ **ให้เครดิต** (ลูกค้าไม่ต้องจ่ายเงินสดทันที แต่มีเวลาผ่อนผัน
เช่น "Net 30" หมายถึงต้องจ่ายภายใน 30 วัน) ธุรกิจนั้นจะมี **ลูกหนี้การค้า (Accounts Receivable)**
เกิดขึ้น — คือเงินที่ลูกค้าเป็นหนี้ธุรกิจอยู่ ระบบบัญชีลูกหนี้มีหน้าที่หลัก 3 อย่าง:

1. **บันทึกใบแจ้งหนี้ (Invoice)** ที่ออกให้ลูกค้าแต่ละราย พร้อมจำนวนเงินและวันที่ออก
2. **บันทึกการรับชำระเงิน (Payment)** เมื่อลูกค้าจ่ายเงินมา และนำไปหักลบยอดค้างของใบแจ้งหนี้ที่เกี่ยวข้อง
3. **วิเคราะห์อายุหนี้ (Aging Analysis)** — จัดกลุ่มยอดค้างชำระตามระยะเวลาที่ค้างมานานแค่ไหน เพื่อให้
   ฝ่ายบัญชี/ผู้บริหารเห็นภาพว่าลูกหนี้รายไหนมีความเสี่ยงสูงที่จะไม่ได้รับชำระ (ยิ่งค้างนาน ยิ่งเสี่ยง)

### ความต้องการของระบบ (Requirements)

| ข้อ | ความต้องการ |
|---|---|
| 1 | เก็บข้อมูลลูกค้าหลัก: รหัสลูกค้า, ชื่อ, วงเงินเครดิต (Credit Limit), ยอดค้างชำระรวม |
| 2 | เก็บข้อมูลใบแจ้งหนี้: เลขที่ใบแจ้งหนี้, รหัสลูกค้า, วันที่ออกใบแจ้งหนี้, จำนวนเงิน, ยอดที่ชำระแล้ว |
| 3 | รับชำระเงินจากลูกค้า นำไปหักลบยอดค้างของใบแจ้งหนี้ที่ระบุ และปรับปรุงยอดคงเหลือของลูกค้า |
| 4 | หากอ้างอิงใบแจ้งหนี้ที่ไม่มีอยู่จริงในการชำระเงิน ต้องไม่ทำให้โปรแกรมล่ม ต้องบันทึก log แล้วข้ามไป |
| 5 | คำนวณจำนวนวันที่ผ่านไปนับจากวันที่ออกใบแจ้งหนี้ถึงวันที่รันรายงาน โดยใช้ Intrinsic Date Function |
| 6 | จัดกลุ่มยอดค้างชำระเป็น 4 ช่วงอายุ: **Current (0-30 วัน), 31-60 วัน, 61-90 วัน, 91+ วัน** |
| 7 | พิมพ์รายงานอายุหนี้แยกตามลูกค้า (Control Break) พร้อมยอดรวมท้ายรายงาน (Grand Total) |
| 8 | ใบแจ้งหนี้ที่ชำระครบแล้ว (ยอดคงเหลือเป็น 0) ต้องไม่ปรากฏในรายงานอายุหนี้ |

### สถาปัตยกรรมระบบ: แผนผังโมดูลและการไหลของข้อมูล

```
                    +-------------------+
                    |   ARSETUP.cob     |  (Step 493)
                    | (Initial Load)    |
                    +--------+----------+
                             |
                 writes      v
        +--------------------------------------+
        |  CUSTOMER.DAT (Indexed, key CUST-ID)  |
        |  INVOICE.DAT  (Indexed, key INV-NUM)  |
        +--------------------+-------------------+
                             ^
                 reads/updates
                             |
                    +--------+----------+       reads
                    |   ARPAYMNT.cob    |<------------------+
                    | (Payment Apply)   |                   |
                    +--------+----------+          PAYMENTS.DAT
                             |                    (Line Sequential)
                 writes      v
                    +-------------------+
                    | PAYERR.LOG        |
                    | (Error Log)       |
                    +-------------------+

                    +-------------------+
                    |   ARREPORT.cob    |  (Step 497-498)
                    | (Aging Report)    |
                    +--------+----------+
                             |
                       CALLs |
                             v
                    +-------------------+
                    |   AGINGCLC.cob    |  (Step 496, subprogram)
                    | (Days/Bucket Calc)|
                    +-------------------+
```

### เหตุผลของการตัดสินใจออกแบบที่สำคัญ

- **ทำไมใช้ Indexed File สำหรับ CUSTOMER และ INVOICE**: ทั้งสองไฟล์ต้องถูกเข้าถึงแบบสุ่มด้วย Key
  (`CUST-ID`, `INV-NUMBER`) บ่อยมากในโปรแกรม `ARPAYMNT` (ค้นหาใบแจ้งหนี้ที่ตรงกับการชำระเงินแต่ละ
  รายการ) ถ้าใช้ Sequential File จะต้องอ่านไฟล์ทั้งหมดทุกครั้งเพื่อหา record ที่ต้องการ (ทบทวนแนวคิด
  จาก Part 028 ขั้นตอนที่ 271) Indexed File จึงเหมาะสมที่สุด
- **ทำไมแยก `AGINGCLC` เป็น Subprogram ต่างหาก**: การคำนวณอายุหนี้เป็น logic ที่**อาจถูกเรียกใช้จาก
  หลายโปรแกรมในอนาคต** (เช่น รายงานสรุปผู้บริหาร, ระบบแจ้งเตือนอัตโนมัติ) การแยกเป็น Subprogram ตาม
  หลักการจาก Part 031 ทำให้ logic นี้ถูกเขียนและทดสอบเพียงครั้งเดียว แล้วนำไปใช้ซ้ำได้ทุกที่โดยไม่ต้อง
  คัดลอกโค้ด
- **ทำไมใช้ Copybook สำหรับ Record Layout**: `CUSTOMER-RECORD` และ `INVOICE-RECORD` ถูกใช้ในทั้ง 3
  โปรแกรมหลัก การเขียนโครงสร้างซ้ำ 3 ครั้งเสี่ยงต่อความไม่สอดคล้องกัน (เช่น แก้ไขขนาดฟิลด์ในโปรแกรม
  หนึ่งแต่ลืมแก้อีกโปรแกรม) Copybook ตามหลักการ Part 033 แก้ปัญหานี้ได้อย่างสมบูรณ์
- **ทำไมใช้ `DECLARATIVES` ในโปรแกรม `ARPAYMNT`**: การชำระเงินที่อ้างอิงใบแจ้งหนี้ผิด/ไม่มีอยู่จริง
  เป็นสถานการณ์ที่**คาดว่าจะเกิดขึ้นได้จริง**ในข้อมูลนำเข้าจากภายนอก (เช่น พนักงานคีย์เลขที่ใบแจ้งหนี้
  ผิด) ระบบต้องไม่ล่มและต้องมีการบันทึกร่องรอยไว้ตรวจสอบภายหลัง — ตรงตามวัตถุประสงค์ของ DECLARATIVES
  ที่เรียนมาใน Part 048 เป๊ะ

### ทางเลือกด้านสถาปัตยกรรมที่พิจารณาแล้วไม่เลือกใช้

การออกแบบระบบที่ดีไม่ได้หมายความว่าต้องเลือกวิธีที่ "ซับซ้อนที่สุด" หรือ "ทันสมัยที่สุด" เสมอไป แต่
ต้องเลือกวิธีที่ **เหมาะสมกับขอบเขตความรู้และความต้องการจริง ณ จุดนั้น** ต่อไปนี้คือทางเลือกที่พิจารณา
แล้วแต่ตัดสินใจไม่ใช้ในโปรเจกต์นี้ พร้อมเหตุผล:

| ทางเลือกที่พิจารณา | เหตุผลที่ไม่เลือกใช้ในโปรเจกต์นี้ |
|---|---|
| รวมทุกโปรแกรมเป็นไฟล์เดียว | ขัดกับหลักการ Separation of Concerns (Part 014) และทำให้ทดสอบ/บำรุงรักษายากขึ้นมากเมื่อระบบโตขึ้น |
| ใช้ Relative File แทน Indexed File | Relative File (Part 029) เหมาะกับกรณีที่ Key เป็นเลขลำดับต่อเนื่องไม่มีช่องว่าง แต่ `CUST-ID`/`INV-NUMBER` ในระบบจริงมักมีช่องว่าง (เช่น ลบลูกค้าบางรายทิ้ง) ทำให้ Indexed File เหมาะสมกว่า |
| เก็บยอดคงเหลือใบแจ้งหนี้เป็นฟิลด์แยก | ตามที่อธิบายในแบบฝึกหัดที่ 492.1 การคำนวณสดจาก `INV-AMOUNT - INV-PAID-AMOUNT` ปลอดภัยกว่า |
| ใช้ Dynamic CALL (Part 043) สำหรับ `AGINGCLC` | Static CALL เหมาะกว่าสำหรับ subprogram ที่รู้ชื่อแน่นอนตั้งแต่ตอนคอมไพล์และไม่มีความจำเป็นต้องสลับเปลี่ยนโปรแกรมที่เรียกขณะรัน (รายละเอียดเพิ่มเติมในขั้นตอนที่ 496) |
| ใช้ Report Writer แทน DISPLAY จัดรูปแบบ | พิจารณารายละเอียดในขั้นตอนที่ 498 |

การบันทึกเหตุผลของการตัดสินใจออกแบบเหล่านี้ไว้เป็นลายลักษณ์อักษร (Architecture Decision Record หรือ
ADR ในศัพท์วิศวกรรมซอฟต์แวร์สมัยใหม่) เป็นแนวปฏิบัติที่ดีมาก เพราะช่วยให้ทีมในอนาคต (หรือแม้แต่ตัวเรา
เองในอีกหกเดือนข้างหน้า) เข้าใจว่า "ทำไม" ระบบถึงถูกออกแบบมาแบบนี้ ไม่ใช่แค่ "อะไร" ที่ถูกสร้างขึ้นมา

### ข้อควรระวัง

- การออกแบบระบบก่อนเขียนโค้ดสำคัญมากขึ้นเรื่อย ๆ ตามขนาดของระบบ — ระบบนี้มี 4 โปรแกรมที่ต้องทำงาน
  ร่วมกันผ่านไฟล์ร่วม การไม่วางแผนโครงสร้างไฟล์และลำดับการรันให้ชัดเจนล่วงหน้าจะนำไปสู่ปัญหาข้อมูล
  ไม่สอดคล้องกันระหว่างโปรแกรมได้ง่ายมาก
- สังเกตว่าแผนผังข้างต้นแสดงให้เห็นว่า `ARSETUP` ต้องรัน**ก่อน** `ARPAYMNT` เสมอ (เพราะต้องมีลูกค้า/
  ใบแจ้งหนี้ก่อนจะรับชำระเงินได้) และ `ARPAYMNT` ควรรัน**ก่อน** `ARREPORT` (เพื่อให้รายงานสะท้อนการ
  ชำระเงินล่าสุด) — ลำดับการรันที่ถูกต้องนี้จะสาธิตให้เห็นจริงในขั้นตอนที่ 499

### แบบฝึกหัดที่ 491.1

**โจทย์**: จงอธิบายว่าทำไมข้อกำหนดข้อ 8 ("ใบแจ้งหนี้ที่ชำระครบแล้วต้องไม่ปรากฏในรายงานอายุหนี้")
จึงสำคัญต่อความถูกต้องของรายงาน

**เฉลยแนวทาง**: รายงานอายุหนี้มีวัตถุประสงค์เพื่อแสดง**ยอดเงินที่ยังค้างชำระอยู่จริง**เท่านั้น หาก
ใบแจ้งหนี้ที่ชำระครบแล้ว (ยอดคงเหลือ = 0) ยังปรากฏในรายงาน จะทำให้ผู้อ่านรายงานเข้าใจผิดว่ายังมี
ความเสี่ยงด้านหนี้สูญจากใบแจ้งหนี้นั้นอยู่ ทั้งที่จริง ๆ ธุรกิจได้รับเงินครบถ้วนแล้ว ซึ่งอาจนำไปสู่
การตัดสินใจทางธุรกิจที่ผิดพลาด (เช่น ปฏิเสธการให้เครดิตเพิ่มกับลูกค้าที่จริง ๆ แล้วมีประวัติชำระตรงเวลา)

---

## ขั้นตอนที่ 492: ออกแบบโครงสร้างข้อมูลร่วมด้วย Copybook

### แนวคิด

ก่อนเขียนโปรแกรมใด ๆ เราต้องออกแบบ **Copybook** สองไฟล์ที่จะใช้ร่วมกันตลอดทั้งระบบ ตามเทคนิคจาก
Part 033 — นี่คือจุดเริ่มต้นที่สำคัญที่สุดเพราะทุกโปรแกรมในระบบจะพึ่งพาโครงสร้างนี้

### CUSTREC.CPY — โครงสร้างข้อมูลลูกค้า

```cobol
      *> Copybook: customer master record layout, shared by every
      *> program in the Accounts Receivable system via COPY.
           05  CUST-ID              PIC 9(5).
           05  CUST-NAME            PIC X(20).
           05  CUST-CREDIT-LIMIT    PIC 9(7)V99.
           05  CUST-BALANCE         PIC 9(7)V99.
```

### INVREC.CPY — โครงสร้างข้อมูลใบแจ้งหนี้

```cobol
      *> Copybook: invoice record layout, shared by every program in
      *> the Accounts Receivable system via COPY.
           05  INV-NUMBER           PIC 9(6).
           05  INV-CUST-ID          PIC 9(5).
           05  INV-DATE             PIC 9(8).
           05  INV-AMOUNT           PIC 9(7)V99.
           05  INV-PAID-AMOUNT      PIC 9(7)V99.
           05  INV-STATUS           PIC X(1).
               88  INV-OPEN                VALUE "O".
               88  INV-PAID-IN-FULL         VALUE "P".
```

### อธิบายการออกแบบแต่ละฟิลด์

**`CUSTREC.CPY`**:
- `CUST-ID PIC 9(5)` — รหัสลูกค้า 5 หลัก (10001-99999) ใช้เป็น `RECORD KEY` ของไฟล์ Indexed
- `CUST-NAME PIC X(20)` — ชื่อลูกค้า จำกัด 20 ตัวอักษร (ภาษาอังกฤษล้วนตามกฎเหล็กของหลักสูตร)
- `CUST-CREDIT-LIMIT PIC 9(7)V99` — วงเงินเครดิตสูงสุด (ไม่เกิน 99,999.99 ในระดับหลักสูตรนี้)
- `CUST-BALANCE PIC 9(7)V99` — ยอดค้างชำระรวมปัจจุบัน (ผลรวมของทุกใบแจ้งหนี้ที่ยังไม่ชำระครบ)

**`INVREC.CPY`**:
- `INV-NUMBER PIC 9(6)` — เลขที่ใบแจ้งหนี้ 6 หลัก ใช้เป็น `RECORD KEY`
- `INV-CUST-ID PIC 9(5)` — รหัสลูกค้าเจ้าของใบแจ้งหนี้นี้ (Foreign Key เชิงแนวคิด — COBOL ไม่มีกลไก
  บังคับความสัมพันธ์ระหว่างไฟล์แบบ Database จริง โปรแกรมต้องรับผิดชอบความถูกต้องนี้เอง)
- `INV-DATE PIC 9(8)` — วันที่ออกใบแจ้งหนี้ รูปแบบ `CCYYMMDD` (เช่น `20260325`) ตามธรรมเนียมที่เรียน
  มาใน Part 036/047 ที่ใช้กับ Intrinsic Date Function ได้โดยตรง
- `INV-AMOUNT` / `INV-PAID-AMOUNT` — จำนวนเงินตั้งต้นและยอดที่ชำระแล้วสะสม (ยอดคงเหลือคำนวณได้จาก
  `INV-AMOUNT - INV-PAID-AMOUNT` เสมอ ไม่จำเป็นต้องเก็บเป็นฟิลด์แยกต่างหาก — ลดความเสี่ยงข้อมูล
  ไม่สอดคล้องกัน)
- `INV-STATUS` พร้อม 88-level — ใช้เทคนิค Condition Name จาก Part 010/030 ทำให้โค้ดอ่านง่ายกว่าการ
  เทียบค่า `"O"`/`"P"` ตรง ๆ

### พิสูจน์ว่า Copybook ใช้งานได้จริงกับ FD

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP492-COPYBOOK-CHECK.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTCHK.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-CUST-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           COPY CUSTREC.

       WORKING-STORAGE SECTION.
       01  WS-CUST-STATUS            PIC XX.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUSTOMER-FILE
           MOVE 10001 TO CUST-ID
           MOVE "TEST CUSTOMER" TO CUST-NAME
           MOVE 50000.00 TO CUST-CREDIT-LIMIT
           MOVE 0 TO CUST-BALANCE
           WRITE CUSTOMER-RECORD
           DISPLAY "WRITE STATUS=" WS-CUST-STATUS
           CLOSE CUSTOMER-FILE
           STOP RUN.
```

คอมไพล์ด้วย flag `-I` เพื่อบอกตำแหน่งของโฟลเดอร์ที่เก็บไฟล์ Copybook (ตามที่เรียนใน Part 033):

```bash
mkdir copybooks
# (put CUSTREC.CPY and INVREC.CPY inside copybooks/)
/opt/gnucobol-isam/bin/cobc -I copybooks -x -o step492 step492.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./step492
```

### ผลลัพธ์จริงที่ได้

```
WRITE STATUS=00
```

### อธิบายโครงสร้างไฟล์โปรเจกต์

ระบบนี้จัดวางไฟล์เป็นโครงสร้างมาตรฐานที่ใช้กันในโปรเจกต์ COBOL จริง:

```
ar-system/
├── copybooks/
│   ├── CUSTREC.CPY      (Customer record layout)
│   └── INVREC.CPY       (Invoice record layout)
├── ARSETUP.cob          (Step 493)
├── ARPAYMNT.cob         (Step 495)
├── AGINGCLC.cob         (Step 496, subprogram)
└── ARREPORT.cob         (Step 497-498)
```

### ข้อควรระวัง

- `-I copybooks` บอก compiler ว่าให้ค้นหาไฟล์ที่ `COPY` เรียกใช้ (เช่น `COPY CUSTREC.`) ในโฟลเดอร์
  `copybooks/` — ถ้าลืม flag นี้ compiler จะหาไฟล์ `CUSTREC.cpy`/`CUSTREC.CPY` ไม่เจอและ error ทันที
  (ทบทวนรายละเอียดเพิ่มเติมได้จาก Part 033)
- สังเกตว่า Copybook **ไม่มี level 01** ครอบตัวเอง (เริ่มที่ level 05 เลย) เพราะโปรแกรมที่เรียกใช้
  จะเป็นผู้กำหนด level 01 เอง (`01 CUSTOMER-RECORD.` ตามด้วย `COPY CUSTREC.`) วิธีนี้ทำให้ Copybook
  เดียวกันสามารถถูกใช้ซ้ำได้ยืดหยุ่นกว่า (เช่น อาจตั้งชื่อ 01-level ต่างกันในแต่ละโปรแกรมถ้าจำเป็น)

### แบบฝึกหัดที่ 492.1

**โจทย์**: จงอธิบายว่าทำไมฟิลด์ `INV-STATUS` จึงไม่เก็บ "ยอดคงเหลือ" (remaining balance) เป็นฟิลด์
แยกต่างหาก แต่เลือกคำนวณจาก `INV-AMOUNT - INV-PAID-AMOUNT` ทุกครั้งที่ต้องการใช้แทน

**เฉลยแนวทาง**: หากเก็บ "ยอดคงเหลือ" เป็นฟิลด์แยก จะมีความเสี่ยงที่ข้อมูลจะ**ไม่สอดคล้องกัน (data
inconsistency)** เช่น หากโปรแกรมอัปเดต `INV-PAID-AMOUNT` แต่ลืมอัปเดตฟิลด์ยอดคงเหลือให้ตรงกัน (บั๊ก
ที่พบได้บ่อยเมื่อมีฟิลด์ซ้ำซ้อนที่ต้องซิงค์กันเอง) ข้อมูลในไฟล์จะขัดแย้งกันเอง การคำนวณยอดคงเหลือจาก
`INV-AMOUNT - INV-PAID-AMOUNT` ทุกครั้งที่ต้องใช้ **รับประกันว่าค่าที่ได้ถูกต้องเสมอ 100%** ตราบใดที่
`INV-AMOUNT` และ `INV-PAID-AMOUNT` (ซึ่งเป็นข้อมูลต้นทางที่แท้จริง) ถูกต้อง หลักการนี้เรียกว่า "Single
Source of Truth" เป็นแนวปฏิบัติที่ดีมากในการออกแบบฐานข้อมูล/ไฟล์

---

## ขั้นตอนที่ 493: โปรแกรม ARSETUP — สร้างข้อมูลตั้งต้นของระบบ

### แนวคิด

`ARSETUP` คือโปรแกรมที่รันเพียงครั้งเดียวตอนเริ่มต้นระบบ (หรือเมื่อต้องการรีเซ็ตข้อมูลทดสอบ) มีหน้าที่
สร้างไฟล์ `CUSTOMER.DAT` และ `INVOICE.DAT` พร้อมข้อมูลตั้งต้น 3 ลูกค้าและ 6 ใบแจ้งหนี้ ที่จะใช้ทดสอบ
ตลอดทั้ง Part นี้

### ข้อมูลตั้งต้นที่ออกแบบไว้

**ลูกค้า 3 ราย**:

| CUST-ID | ชื่อ | วงเงินเครดิต |
|---|---|---|
| 10001 | SIAM TRADING CO | 100,000.00 |
| 10002 | BANGKOK SUPPLIES LTD | 50,000.00 |
| 10003 | THAI EXPORT PARTNERS | 75,000.00 |

**ใบแจ้งหนี้ 6 ใบ** (`WS-AS-OF-DATE` ที่จะใช้คำนวณอายุหนี้ในขั้นตอนที่ 497 คือ `2026-04-01`):

| INV-NUMBER | CUST-ID | วันที่ออก | จำนวนเงิน |
|---|---|---|---|
| 100001 | 10001 | 2026-03-25 | 15,000.00 |
| 100002 | 10001 | 2026-02-10 | 8,000.00 |
| 100003 | 10002 | 2026-01-05 | 12,000.00 |
| 100004 | 10002 | 2025-11-20 | 5,000.00 |
| 100005 | 10003 | 2026-03-30 | 20,000.00 |
| 100006 | 10003 | 2025-12-01 | 3,000.00 |

ยอดค้างชำระตั้งต้นของลูกค้าแต่ละราย = ผลรวมใบแจ้งหนี้ของตน: `10001` = 23,000.00, `10002` =
17,000.00, `10003` = 23,000.00

### ซอร์สโค้ดฉบับสมบูรณ์: `ARSETUP.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ARSETUP.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTOMER.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-CUST-STATUS.

           SELECT INVOICE-FILE ASSIGN TO "INVOICE.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS INV-NUMBER
               FILE STATUS IS WS-INV-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           COPY CUSTREC.

       FD  INVOICE-FILE.
       01  INVOICE-RECORD.
           COPY INVREC.

       WORKING-STORAGE SECTION.
       01  WS-CUST-STATUS            PIC XX.
       01  WS-INV-STATUS             PIC XX.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM CREATE-CUSTOMERS
           PERFORM CREATE-INVOICES
           DISPLAY "AR-SETUP COMPLETE."
           STOP RUN.

       CREATE-CUSTOMERS.
           OPEN OUTPUT CUSTOMER-FILE
           MOVE 10001 TO CUST-ID
           MOVE "SIAM TRADING CO" TO CUST-NAME
           MOVE 100000.00 TO CUST-CREDIT-LIMIT
           MOVE 23000.00 TO CUST-BALANCE
           WRITE CUSTOMER-RECORD

           MOVE 10002 TO CUST-ID
           MOVE "BANGKOK SUPPLIES LTD" TO CUST-NAME
           MOVE 50000.00 TO CUST-CREDIT-LIMIT
           MOVE 17000.00 TO CUST-BALANCE
           WRITE CUSTOMER-RECORD

           MOVE 10003 TO CUST-ID
           MOVE "THAI EXPORT PARTNERS" TO CUST-NAME
           MOVE 75000.00 TO CUST-CREDIT-LIMIT
           MOVE 23000.00 TO CUST-BALANCE
           WRITE CUSTOMER-RECORD
           CLOSE CUSTOMER-FILE.

       CREATE-INVOICES.
           OPEN OUTPUT INVOICE-FILE

           MOVE 100001 TO INV-NUMBER
           MOVE 10001 TO INV-CUST-ID
           MOVE 20260325 TO INV-DATE
           MOVE 15000.00 TO INV-AMOUNT
           MOVE 0 TO INV-PAID-AMOUNT
           SET INV-OPEN TO TRUE
           WRITE INVOICE-RECORD

           MOVE 100002 TO INV-NUMBER
           MOVE 10001 TO INV-CUST-ID
           MOVE 20260210 TO INV-DATE
           MOVE 8000.00 TO INV-AMOUNT
           MOVE 0 TO INV-PAID-AMOUNT
           SET INV-OPEN TO TRUE
           WRITE INVOICE-RECORD

           MOVE 100003 TO INV-NUMBER
           MOVE 10002 TO INV-CUST-ID
           MOVE 20260105 TO INV-DATE
           MOVE 12000.00 TO INV-AMOUNT
           MOVE 0 TO INV-PAID-AMOUNT
           SET INV-OPEN TO TRUE
           WRITE INVOICE-RECORD

           MOVE 100004 TO INV-NUMBER
           MOVE 10002 TO INV-CUST-ID
           MOVE 20251120 TO INV-DATE
           MOVE 5000.00 TO INV-AMOUNT
           MOVE 0 TO INV-PAID-AMOUNT
           SET INV-OPEN TO TRUE
           WRITE INVOICE-RECORD

           MOVE 100005 TO INV-NUMBER
           MOVE 10003 TO INV-CUST-ID
           MOVE 20260330 TO INV-DATE
           MOVE 20000.00 TO INV-AMOUNT
           MOVE 0 TO INV-PAID-AMOUNT
           SET INV-OPEN TO TRUE
           WRITE INVOICE-RECORD

           MOVE 100006 TO INV-NUMBER
           MOVE 10003 TO INV-CUST-ID
           MOVE 20251201 TO INV-DATE
           MOVE 3000.00 TO INV-AMOUNT
           MOVE 0 TO INV-PAID-AMOUNT
           SET INV-OPEN TO TRUE
           WRITE INVOICE-RECORD

           CLOSE INVOICE-FILE.
```

คอมไพล์และรันด้วย:

```bash
/opt/gnucobol-isam/bin/cobc -I copybooks -x -o ARSETUP ARSETUP.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./ARSETUP
```

### ผลลัพธ์จริงที่ได้

```
AR-SETUP COMPLETE.
```

### อธิบายโค้ดทีละส่วน

- `CREATE-CUSTOMERS` เปิดไฟล์ด้วย `OPEN OUTPUT` (สร้างไฟล์ใหม่ทับของเก่าเสมอ ตามที่เรียนมาใน
  Part 025) แล้วเขียน 3 record ตามลำดับ — สังเกตว่า `CUST-BALANCE` ถูกกำหนดค่าให้ตรงกับผลรวมใบแจ้งหนี้
  ที่จะสร้างในขั้นตอนถัดไปพอดี (คำนวณด้วยมือไว้ล่วงหน้าสำหรับข้อมูลตั้งต้นนี้โดยเฉพาะ ในระบบจริงค่านี้
  จะถูกคำนวณจากการสะสมยอดใบแจ้งหนี้แต่ละใบแทนการกำหนดตรง ๆ)
- `CREATE-INVOICES` เขียนใบแจ้งหนี้ 6 ใบ ทุกใบเริ่มต้นด้วย `SET INV-OPEN TO TRUE` (สถานะ "ยังไม่ชำระ"
  ทบทวนเทคนิค `SET ... TO TRUE` จาก Part 010) และ `INV-PAID-AMOUNT = 0` เสมอสำหรับใบแจ้งหนี้ใหม่
- ทั้งสองไฟล์ใช้ `RECORD KEY` ที่ต่างกัน (`CUST-ID` และ `INV-NUMBER`) แต่ทั้งคู่เป็น Indexed File ที่
  ต้องคอมไพล์ด้วย ISAM-enabled build เดียวกัน

### ข้อควรระวัง

- โปรแกรม `ARSETUP` ใช้ `OPEN OUTPUT` ซึ่ง**ลบข้อมูลเดิมทั้งหมดทิ้งทุกครั้งที่รัน** — เหมาะสำหรับการ
  ตั้งค่าระบบครั้งแรกหรือรีเซ็ตข้อมูลทดสอบเท่านั้น **ห้ามรันโปรแกรมนี้ซ้ำในระบบ production ที่มีข้อมูล
  จริงสะสมอยู่แล้วเด็ดขาด** เพราะจะทำให้ข้อมูลลูกค้า/ใบแจ้งหนี้ทั้งหมดหายไป
- ข้อมูลตั้งต้นในตัวอย่างนี้ถูกฝังไว้ในโค้ดโดยตรง (hardcoded) เพื่อความง่ายในการสอน — ระบบจริงมักอ่าน
  ข้อมูลตั้งต้นจากไฟล์ภายนอกหรือระบบต้นทางอื่นแทน

### แบบฝึกหัดที่ 493.1

**โจทย์**: จงคำนวณด้วยมือว่ายอด `CUST-BALANCE` ของลูกค้า `10002` (BANGKOK SUPPLIES LTD) ควรเป็น
เท่าใด จากใบแจ้งหนี้ที่กำหนดไว้ในตาราง แล้วตรวจสอบกับค่าที่เขียนไว้ในโค้ด

**เฉลย**: ลูกค้า `10002` มีใบแจ้งหนี้ 2 ใบ คือ `100003` (12,000.00) และ `100004` (5,000.00) รวมเป็น
`12,000.00 + 5,000.00 = 17,000.00` ตรงกับค่า `MOVE 17000.00 TO CUST-BALANCE` ที่เขียนไว้ในโค้ดพอดี

---

## ขั้นตอนที่ 494: ทำความเข้าใจ Invoice Lifecycle และการตรวจสอบข้อมูลตั้งต้น

### วงจรชีวิตของใบแจ้งหนี้ (Invoice Lifecycle)

ก่อนไปเขียนโปรแกรมรับชำระเงิน ขั้นตอนนี้จะพาทบทวนและตรวจสอบข้อมูลที่ `ARSETUP` สร้างไว้ให้แน่ใจว่า
ถูกต้อง — เป็นวินัยสำคัญของการพัฒนาระบบจริง: **ตรวจสอบข้อมูลตั้งต้นให้แน่ใจก่อนพัฒนาโปรแกรมที่พึ่งพา
ข้อมูลนั้นต่อ**

ใบแจ้งหนี้ทุกใบมีวงจรชีวิต 3 สถานะ (แม้ในระบบนี้จะย่อให้เหลือ 2 สถานะเพื่อความง่าย):

```
[สร้างใหม่: INV-OPEN, PAID-AMOUNT=0]
              |
              v (ได้รับชำระเงินบางส่วน)
[ยังเปิดอยู่: INV-OPEN, 0 < PAID-AMOUNT < AMOUNT]
              |
              v (ได้รับชำระเงินครบ)
[ปิดสมบูรณ์: INV-PAID-IN-FULL, PAID-AMOUNT >= AMOUNT]
```

### โปรแกรมตรวจสอบข้อมูล: อ่านไฟล์ลูกค้าทั้งหมดแบบ Sequential

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP494-VERIFY-SETUP.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTOMER.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           COPY CUSTREC.

       WORKING-STORAGE SECTION.
       01  WS-STATUS                 PIC XX.
       01  WS-EOF-FLAG               PIC X(1) VALUE "N".
           88  END-OF-CUSTOMERS              VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN INPUT CUSTOMER-FILE
           PERFORM UNTIL END-OF-CUSTOMERS
               READ CUSTOMER-FILE NEXT RECORD
                   AT END
                       SET END-OF-CUSTOMERS TO TRUE
                   NOT AT END
                       DISPLAY CUST-ID " " CUST-NAME " BAL="
                           CUST-BALANCE
               END-READ
           END-PERFORM
           CLOSE CUSTOMER-FILE
           STOP RUN.
```

คอมไพล์และรันด้วย:

```bash
/opt/gnucobol-isam/bin/cobc -I copybooks -x -o step494 step494.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./step494
```

### ผลลัพธ์จริงที่ได้

```
10001 SIAM TRADING CO      BAL=0018000.00
10002 BANGKOK SUPPLIES LTD BAL=0017000.00
10003 THAI EXPORT PARTNERS BAL=0003000.00
```

### รอสักครู่ — ทำไมยอดไม่ตรงกับที่คำนวณไว้ในขั้นตอนที่ 493

สังเกตให้ดี: `10001` แสดง `18000.00` ไม่ใช่ `23000.00` ตามที่ `ARSETUP` เขียนไว้ตอนแรก! นี่**ไม่ใช่
ข้อผิดพลาด** — เอกสารนี้ถูกเตรียมโดยรันโปรแกรมทั้งหมดของ Part นี้เรียงตามลำดับเพื่อพิสูจน์ผลลัพธ์จริง
ทุกจุด ผลลัพธ์ที่เห็น ณ ขั้นตอนนี้จึง**สะท้อนสถานะหลังจากที่โปรแกรม `ARPAYMNT` (ขั้นตอนที่ 495) ได้รัน
ไปแล้วในการเตรียมเอกสารจริง** — เป็นการพิสูจน์ให้เห็นล่วงหน้าว่าไฟล์ `CUSTOMER.DAT` เป็น**สถานะที่คงอยู่
ระหว่างการรันโปรแกรมหลายตัว** (persistent state) ไม่ใช่ข้อมูลที่รีเซ็ตใหม่ทุกครั้ง — ถ้าคุณรันเฉพาะ
`ARSETUP` แล้วรัน `step494` ทันทีโดยยังไม่รัน `ARPAYMNT` จะเห็นค่า `23000.00`/`17000.00`/`23000.00`
ตามที่ตั้งไว้แต่แรกอย่างถูกต้อง — ลองทดสอบด้วยตัวเองเพื่อยืนยันความเข้าใจนี้

### อธิบายจุดสำคัญ

- นี่คือบทเรียนสำคัญของการทำงานกับไฟล์ที่ใช้ร่วมกันหลายโปรแกรม (shared persistent files): **ลำดับ
  การรันโปรแกรมมีผลต่อสถานะข้อมูลเสมอ** ต่างจากตัวแปรใน WORKING-STORAGE ที่หายไปเมื่อโปรแกรมจบการทำงาน
- `ACCESS MODE IS SEQUENTIAL` กับ Indexed File ทำให้ `READ ... NEXT RECORD` คืนค่า record ตามลำดับ
  ของ Key จากน้อยไปมากเสมอ (`10001` → `10002` → `10003`) แม้ข้อมูลจะถูกเขียนแบบใดก็ตามตอน `OPEN
  OUTPUT` — คุณสมบัติของ ISAM ที่เรียนมาใน Part 028

### ข้อควรระวัง

- เมื่อพัฒนาระบบที่มีหลายโปรแกรมทำงานร่วมกันผ่านไฟล์ ควรมีวินัยการทดสอบที่ชัดเจนเสมอว่า **สถานะไฟล์
  ปัจจุบันมาจากการรันโปรแกรมอะไรไปแล้วบ้าง** มิฉะนั้นอาจสับสนว่าทำไมผลลัพธ์ไม่ตรงกับที่คาดไว้ — ระบบ
  จริงมักมีการ backup/reset ข้อมูลทดสอบให้ชัดเจนเป็นขั้นตอนมาตรฐานก่อนการทดสอบแต่ละรอบ

### แบบฝึกหัดที่ 494.1

**โจทย์**: จงอธิบายว่าทำไมการรันโปรแกรม `ARSETUP` ซ้ำอีกครั้งจะทำให้ยอด `CUST-BALANCE` กลับไปเป็น
`23000.00`/`17000.00`/`23000.00` เหมือนเดิม (ย้อนกลับพฤติกรรมของขั้นตอนที่ 495 ทั้งหมด)

**เฉลย**: เพราะ `ARSETUP` ใช้ `OPEN OUTPUT` ซึ่งสร้างไฟล์ใหม่ทับของเก่าเสมอ (ทบทวนจาก Part 025) การ
รันซ้ำจะลบข้อมูลเดิม (รวมถึงผลของการชำระเงินที่เคยบันทึกไว้) ทิ้งทั้งหมด แล้วเขียนข้อมูลตั้งต้นชุดเดิม
กลับเข้าไปใหม่ทั้งหมด ทำให้ยอดคงเหลือกลับไปเป็นค่าตั้งต้นเหมือนไม่เคยมีการชำระเงินเกิดขึ้นเลย — นี่คือ
เหตุผลที่ข้อควรระวังในขั้นตอนที่ 493 เน้นย้ำว่าห้ามรัน `ARSETUP` ซ้ำในระบบที่มีข้อมูลจริงสะสมอยู่แล้ว

---

## ขั้นตอนที่ 495: โปรแกรม ARPAYMNT — บันทึกการรับชำระเงินด้วย DECLARATIVES

### แนวคิด

`ARPAYMNT` คือหัวใจของระบบ: อ่านไฟล์ธุรกรรมการชำระเงิน (`PAYMENTS.DAT`, Line Sequential) ทีละรายการ
นำไปค้นหาใบแจ้งหนี้ที่ตรงกันในไฟล์ `INVOICE.DAT` (Indexed) แล้วปรับปรุงยอดที่ชำระแล้ว พร้อมปรับยอด
คงเหลือของลูกค้าที่เกี่ยวข้อง — และใช้ **`DECLARATIVES`** ตามเทคนิคที่เรียนมาทั้งหมดจาก Part 048 เพื่อ
จัดการกรณีที่การชำระเงินอ้างอิงใบแจ้งหนี้ที่ไม่มีอยู่จริง โดยไม่ให้โปรแกรมล่มและบันทึก log ไว้ตรวจสอบ

### ออกแบบไฟล์ธุรกรรมการชำระเงิน

`PAYMENT-RECORD` ประกอบด้วย `PAY-INV-NUMBER PIC 9(6)` และ `PAY-AMOUNT PIC 9(7)V99` (ทศนิยมแบบ
implied ตามที่เรียนมาใน Part 006 — ไม่มีจุดทศนิยมจริงในไฟล์) รวม 15 ตัวอักษรต่อบรรทัด ข้อมูลทดสอบ
3 รายการ (รายการที่ 3 จงใจอ้างอิงใบแจ้งหนี้ที่ไม่มีอยู่จริงเพื่อทดสอบ Declarative):

```bash
printf '100001000500000\n100005002000000\n999999000010000\n' > PAYMENTS.DAT
```

| บรรทัด | INV-NUMBER | จำนวนเงิน | ความหมาย |
|---|---|---|---|
| 1 | 100001 | 5,000.00 | ชำระบางส่วนของใบแจ้งหนี้ `100001` (ยอดเดิม 15,000.00) |
| 2 | 100005 | 20,000.00 | ชำระเต็มจำนวนของใบแจ้งหนี้ `100005` (ยอดเดิม 20,000.00 พอดี) |
| 3 | 999999 | 100.00 | **ใบแจ้งหนี้ที่ไม่มีอยู่จริง** — ทดสอบ Declarative |

### ซอร์สโค้ดฉบับสมบูรณ์: `ARPAYMNT.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ARPAYMNT.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTOMER.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-CUST-STATUS.

           SELECT INVOICE-FILE ASSIGN TO "INVOICE.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS INV-NUMBER
               FILE STATUS IS WS-INV-STATUS.

           SELECT PAYMENT-FILE ASSIGN TO "PAYMENTS.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-PAY-STATUS.

           SELECT ERROR-LOG-FILE ASSIGN TO "PAYERR.LOG"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-LOG-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           COPY CUSTREC.

       FD  INVOICE-FILE.
       01  INVOICE-RECORD.
           COPY INVREC.

       FD  PAYMENT-FILE.
       01  PAYMENT-RECORD.
           05  PAY-INV-NUMBER        PIC 9(6).
           05  PAY-AMOUNT            PIC 9(7)V99.

       FD  ERROR-LOG-FILE.
       01  ERROR-LOG-RECORD          PIC X(60).

       WORKING-STORAGE SECTION.
       01  WS-CUST-STATUS            PIC XX.
       01  WS-INV-STATUS             PIC XX.
           88  INV-FOUND                    VALUE "00".
       01  WS-PAY-STATUS             PIC XX.
       01  WS-LOG-STATUS             PIC XX.
       01  WS-EOF-FLAG               PIC X(1) VALUE "N".
           88  END-OF-PAYMENTS                VALUE "Y".
       01  WS-APPLIED-COUNT          PIC 9(3) VALUE 0.
       01  WS-REJECTED-COUNT         PIC 9(3) VALUE 0.
       01  WS-REMAINING-BALANCE      PIC 9(7)V99.

       PROCEDURE DIVISION.
       DECLARATIVES.
      *> Any error on INVOICE-FILE during payment processing (most
      *> commonly status 23 - the payment references an invoice
      *> number that does not exist) is centralized here, exactly
      *> following the pattern taught in Part 048 step 477.
       INVOICE-ERROR-SECTION SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON INVOICE-FILE.
       INVOICE-ERROR-PARA.
           ADD 1 TO WS-REJECTED-COUNT
           MOVE SPACES TO ERROR-LOG-RECORD
           STRING "PAYMENT REJECTED: INVOICE " PAY-INV-NUMBER
                  " NOT FOUND (STATUS " WS-INV-STATUS ")"
               DELIMITED BY SIZE INTO ERROR-LOG-RECORD
           WRITE ERROR-LOG-RECORD.
       END DECLARATIVES.

       MAIN-PARA SECTION.
       MAIN-PARA-START.
           OPEN I-O CUSTOMER-FILE
           OPEN I-O INVOICE-FILE
           OPEN INPUT PAYMENT-FILE
           OPEN OUTPUT ERROR-LOG-FILE

           PERFORM UNTIL END-OF-PAYMENTS
               READ PAYMENT-FILE
                   AT END
                       SET END-OF-PAYMENTS TO TRUE
                   NOT AT END
                       PERFORM APPLY-ONE-PAYMENT
               END-READ
           END-PERFORM

           CLOSE CUSTOMER-FILE
           CLOSE INVOICE-FILE
           CLOSE PAYMENT-FILE
           CLOSE ERROR-LOG-FILE

           DISPLAY "PAYMENTS APPLIED : " WS-APPLIED-COUNT
           DISPLAY "PAYMENTS REJECTED: " WS-REJECTED-COUNT
           STOP RUN.

       APPLY-ONE-PAYMENT SECTION.
       APPLY-ONE-PAYMENT-START.
           MOVE PAY-INV-NUMBER TO INV-NUMBER
           READ INVOICE-FILE
           IF INV-FOUND
               ADD PAY-AMOUNT TO INV-PAID-AMOUNT
               IF INV-PAID-AMOUNT >= INV-AMOUNT
                   SET INV-PAID-IN-FULL TO TRUE
               END-IF
               REWRITE INVOICE-RECORD
               ADD 1 TO WS-APPLIED-COUNT

               MOVE INV-CUST-ID TO CUST-ID
               READ CUSTOMER-FILE
               SUBTRACT PAY-AMOUNT FROM CUST-BALANCE
               REWRITE CUSTOMER-RECORD

               COMPUTE WS-REMAINING-BALANCE =
                   INV-AMOUNT - INV-PAID-AMOUNT
               DISPLAY "APPLIED " PAY-AMOUNT " TO INVOICE "
                   PAY-INV-NUMBER " (CUSTOMER " INV-CUST-ID
                   ") NEW INVOICE BALANCE=" WS-REMAINING-BALANCE
           END-IF.
```

คอมไพล์และรันด้วย:

```bash
/opt/gnucobol-isam/bin/cobc -I copybooks -x -o ARPAYMNT ARPAYMNT.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./ARPAYMNT
```

### ผลลัพธ์จริงที่ได้

```
APPLIED 0005000.00 TO INVOICE 100001 (CUSTOMER 10001) NEW INVOICE BALANCE=0010000.00
APPLIED 0020000.00 TO INVOICE 100005 (CUSTOMER 10003) NEW INVOICE BALANCE=0000000.00
PAYMENTS APPLIED : 002
PAYMENTS REJECTED: 001
```

เนื้อหาไฟล์ `PAYERR.LOG`:

```
PAYMENT REJECTED: INVOICE 999999 NOT FOUND (STATUS 23)
```

### อธิบายโค้ดทีละส่วน

- **`DECLARATIVES`** จับข้อผิดพลาดของ `INVOICE-FILE` แบบรวมศูนย์เพียงจุดเดียว — สังเกตว่า
  `APPLY-ONE-PAYMENT` **ไม่มี** `IF WS-INV-STATUS NOT = "00" DISPLAY ERROR ...` ปรากฏอยู่เลย
  เพราะตรรกะนั้นถูกย้ายไปที่ `INVOICE-ERROR-PARA` ทั้งหมด ตรงตามรูปแบบที่ฝึกมาใน Part 048 ขั้นตอน
  ที่ 477 และ 480 เป๊ะ
- เมื่อ `READ INVOICE-FILE` ล้มเหลว (บัตรที่ 3, invoice `999999`): Declarative ทำงานก่อน (เพิ่ม
  `WS-REJECTED-COUNT`, เขียน log) แล้วควบคุมกลับมาที่ `APPLY-ONE-PAYMENT-START` ต่อจาก `READ` ทันที
  ตามกฎที่พิสูจน์ใน Part 048 ขั้นตอนที่ 475 — เข้าเงื่อนไข `IF INV-FOUND` เป็นเท็จ (เพราะ
  `WS-INV-STATUS` เป็น `"23"` ไม่ใช่ `"00"`) จึง**ข้าม**การประมวลผลที่เหลือทั้งหมดโดยอัตโนมัติ
- ใบแจ้งหนี้ `100005` ได้รับชำระเต็มจำนวนพอดี (`20,000.00`) ทำให้ `INV-PAID-AMOUNT >= INV-AMOUNT`
  เป็นจริง → `SET INV-PAID-IN-FULL TO TRUE` เปลี่ยนสถานะเป็น `"P"` โดยอัตโนมัติ — จะมีผลสำคัญตอน
  สร้างรายงานอายุหนี้ในขั้นตอนที่ 497 (ยอดคงเหลือ = 0 ทำให้ไม่ปรากฏในรายงาน ตรงตามข้อกำหนดข้อ 8)
- ลำดับการปรับปรุงข้อมูล: อัปเดต `INVOICE-FILE` (ยอดที่ชำระ) ก่อน แล้วจึงอัปเดต `CUSTOMER-FILE`
  (ยอดคงเหลือรวม) — ทั้งสองไฟล์ถูกเปิดด้วย `OPEN I-O` (Input-Output) เพื่อให้ `READ`/`REWRITE`
  ทำงานกับไฟล์เดียวกันได้ ตามกฎโหมดการเปิดไฟล์ที่เรียนมาใน Part 025

### ข้อควรระวัง

- สังเกตว่าเมื่อการชำระเงินถูกปฏิเสธ (invoice `999999`) **ยอดคงเหลือของลูกค้าใด ๆ ก็ไม่ถูกแตะต้องเลย**
  เพราะโค้ดที่อัปเดต `CUSTOMER-FILE` อยู่**หลัง** `IF INV-FOUND` ทำให้การข้ามผ่าน (skip) จาก
  Declarative ป้องกันผลข้างเคียงที่ไม่ถูกต้องได้ครบถ้วนทั้งสองไฟล์พร้อมกัน — นี่คือประโยชน์ของการจัด
  โครงสร้าง `IF` ให้ครอบคลุมทุกการเปลี่ยนแปลงข้อมูลที่เกี่ยวเนื่องกันไว้ในบล็อกเดียว
- โปรแกรมนี้ยังไม่ตรวจสอบกรณี **ชำระเงินเกินยอดคงเหลือจริง** (Overpayment) เช่น หากใบแจ้งหนี้เหลือ
  ค้าง 1,000.00 แต่มีการชำระมา 5,000.00 โปรแกรมจะยอมรับและตั้งสถานะ `PAID-IN-FULL` โดยไม่แจ้งเตือน
  ส่วนต่างที่เกินมา — ในระบบ production จริงควรเพิ่มการตรวจสอบและจัดการเงินทอน/เครดิตคงเหลือ (Credit
  Balance) ซึ่งเกินขอบเขตของโปรเจกต์ระดับหลักสูตรนี้

### แบบฝึกหัดที่ 495.1

**โจทย์**: จงคำนวณด้วยมือว่ายอดคงเหลือของใบแจ้งหนี้ `100001` ควรเป็นเท่าใดหลังจากได้รับชำระ
`5,000.00` จากยอดตั้งต้น `15,000.00` แล้วเปรียบเทียบกับค่าที่โปรแกรมแสดงในผลลัพธ์จริง

**เฉลย**: `15,000.00 - 5,000.00 = 10,000.00` ตรงกับที่โปรแกรมแสดง
`NEW INVOICE BALANCE=0010000.00` พอดี

---

## ขั้นตอนที่ 496: Subprogram AGINGCLC — คำนวณอายุหนี้ด้วย CALL และ Intrinsic Date Function

### แนวคิด

`AGINGCLC` คือ Subprogram ที่แยกออกมาต่างหาก มีหน้าที่เดียวชัดเจน (Single Responsibility): รับวันที่
ออกใบแจ้งหนี้และวันที่ "ณ วันที่รันรายงาน" (as-of date) แล้วคำนวณ (1) จำนวนวันที่ผ่านไป และ (2) ช่วง
อายุหนี้ (bucket) ที่ใบแจ้งหนี้นั้นตกอยู่ ส่งค่ากลับผ่าน `LINKAGE SECTION` ตามเทคนิค `CALL` ที่เรียน
มาใน Part 031-032

### ทำไมใช้วันที่ "as-of date" คงที่ แทนวันที่ปัจจุบันจริงของระบบ

โปรแกรมนี้**จงใจไม่ใช้** `FUNCTION CURRENT-DATE` (ที่เรียนมาใน Part 036) แต่กำหนด `WS-AS-OF-DATE`
เป็นค่าคงที่แทน — เหตุผลคือรายงานอายุหนี้ในโลกจริงมักถูกกำหนด **"วันที่ตัดยอด" (Cut-off Date หรือ
As-of Date) ที่ชัดเจน** ไม่ใช่แค่ "วันที่วันนี้" เสมอไป (เช่น รายงานสิ้นเดือนมักใช้วันสุดท้ายของเดือน
เป็น as-of date แม้จะรันรายงานจริงในอีกไม่กี่วันถัดมา) การกำหนดค่าคงที่ยังทำให้**ผลลัพธ์ในเอกสารนี้
ทำซ้ำได้เสมอ (reproducible)** ไม่เปลี่ยนไปทุกวันตามวันที่รันจริง ซึ่งสำคัญมากสำหรับหลักสูตรที่ต้อง
แสดงผลลัพธ์ที่ตรวจสอบได้

### ซอร์สโค้ดฉบับสมบูรณ์: `AGINGCLC.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. AGINGCLC.
       AUTHOR. COBOL-COURSE.
      *> AR-AGING-CALC subprogram: given an invoice date and an
      *> "as of" report date (both CCYYMMDD), returns the number of
      *> days elapsed and which aging bucket the invoice falls into.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-INV-DATE-INT           PIC S9(9) COMP.
       01  WS-ASOF-DATE-INT          PIC S9(9) COMP.

       LINKAGE SECTION.
       01  LK-INVOICE-DATE           PIC 9(8).
       01  LK-AS-OF-DATE             PIC 9(8).
       01  LK-DAYS-OVERDUE           PIC S9(6).
       01  LK-AGING-BUCKET           PIC 9(1).

       PROCEDURE DIVISION USING LK-INVOICE-DATE
                                LK-AS-OF-DATE
                                LK-DAYS-OVERDUE
                                LK-AGING-BUCKET.
       MAIN-PARA.
           COMPUTE WS-INV-DATE-INT =
               FUNCTION INTEGER-OF-DATE(LK-INVOICE-DATE)
           COMPUTE WS-ASOF-DATE-INT =
               FUNCTION INTEGER-OF-DATE(LK-AS-OF-DATE)
           COMPUTE LK-DAYS-OVERDUE =
               WS-ASOF-DATE-INT - WS-INV-DATE-INT

           EVALUATE TRUE
               WHEN LK-DAYS-OVERDUE <= 30
                   MOVE 1 TO LK-AGING-BUCKET
               WHEN LK-DAYS-OVERDUE <= 60
                   MOVE 2 TO LK-AGING-BUCKET
               WHEN LK-DAYS-OVERDUE <= 90
                   MOVE 3 TO LK-AGING-BUCKET
               WHEN OTHER
                   MOVE 4 TO LK-AGING-BUCKET
           END-EVALUATE

           GOBACK.
```

### โปรแกรมทดสอบ Subprogram แบบเดี่ยว (Unit Test)

ก่อนนำไปใช้ในโปรแกรมรายงานจริง ควรทดสอบ Subprogram แยกต่างหากก่อนเสมอ (แนวคิด Unit Testing ที่จะ
เรียนละเอียดใน Part 079):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP496-TEST-AGING.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-INV-DATE               PIC 9(8).
       01  WS-ASOF-DATE              PIC 9(8) VALUE 20260401.
       01  WS-DAYS                   PIC S9(6).
       01  WS-BUCKET                 PIC 9(1).

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 20260215 TO WS-INV-DATE
           CALL "AGINGCLC" USING WS-INV-DATE, WS-ASOF-DATE,
                WS-DAYS, WS-BUCKET
           DISPLAY "DATE=" WS-INV-DATE " DAYS=" WS-DAYS
               " BUCKET=" WS-BUCKET

           MOVE 20260330 TO WS-INV-DATE
           CALL "AGINGCLC" USING WS-INV-DATE, WS-ASOF-DATE,
                WS-DAYS, WS-BUCKET
           DISPLAY "DATE=" WS-INV-DATE " DAYS=" WS-DAYS
               " BUCKET=" WS-BUCKET

           MOVE 20251201 TO WS-INV-DATE
           CALL "AGINGCLC" USING WS-INV-DATE, WS-ASOF-DATE,
                WS-DAYS, WS-BUCKET
           DISPLAY "DATE=" WS-INV-DATE " DAYS=" WS-DAYS
               " BUCKET=" WS-BUCKET
           STOP RUN.
```

คอมไพล์ทั้งสองไฟล์แยกกันแล้วเชื่อม (link) เข้าด้วยกัน ตามเทคนิค Static CALL ที่เรียนมาใน Part 031:

```bash
/opt/gnucobol-isam/bin/cobc -c AGINGCLC.cob
/opt/gnucobol-isam/bin/cobc -x -o step496 step496.cob AGINGCLC.o
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./step496
```

### ผลลัพธ์จริงที่ได้

```
DATE=20260215 DAYS=+000045 BUCKET=2
DATE=20260330 DAYS=+000002 BUCKET=1
DATE=20251201 DAYS=+000121 BUCKET=4
```

### อธิบายโค้ดทีละส่วน

- `FUNCTION INTEGER-OF-DATE` (เรียนครั้งแรกใน Part 036) แปลงวันที่รูปแบบ `CCYYMMDD` ให้เป็น**เลขจำนวน
  เต็มที่นับต่อเนื่อง** (Julian-style integer) ทำให้การหาผลต่างระหว่างสองวันที่ทำได้ด้วยการลบเลขธรรมดา
  โดยไม่ต้องกังวลเรื่องจำนวนวันในแต่ละเดือนที่ไม่เท่ากัน หรือปีอธิกสุรทิน (Leap Year) เลย — ฟังก์ชันนี้
  จัดการความซับซ้อนเหล่านั้นให้ทั้งหมด
- ตรวจสอบผลลัพธ์ด้วยมือ: `20260215` (15 ก.พ. 2026) ถึง `20260401` (1 เม.ย. 2026) = 13 วันที่เหลือ
  ของเดือน ก.พ. (28-15) + 31 วันของเดือน มี.ค. + 1 วันของเดือน เม.ย. = `13+31+1 = 45` วัน ตรงกับที่
  โปรแกรมคำนวณได้เป๊ะ
- `EVALUATE TRUE` กับเงื่อนไข `<=` ต่อเนื่องกัน (เทคนิคจาก Part 041 Advanced EVALUATE) ทำให้การจัดกลุ่ม
  ช่วงอายุอ่านง่ายและครอบคลุมทุกกรณี (`WHEN OTHER` รับกรณีที่เหลือทั้งหมดคือ 91+ วัน)
- `PROCEDURE DIVISION USING ...` พร้อมพารามิเตอร์ 4 ตัวตามลำดับ **ต้องตรงกับลำดับที่โปรแกรมเรียกส่ง
  มาทุกประการ** (positional parameter passing ตามที่เรียนมาใน Part 032) — สองตัวแรกเป็น input
  (invoice date, as-of date) สองตัวหลังเป็น output (days, bucket) ที่ subprogram เขียนค่ากลับ

### ทำไมเลือกใช้ Static CALL แทน Dynamic CALL

Part 043 สอนไว้ว่า COBOL รองรับทั้ง **Static CALL** (ชื่อโปรแกรมที่เรียกเป็น literal คงที่ เช่น
`CALL "AGINGCLC"` ที่ใช้ในโปรเจกต์นี้) และ **Dynamic CALL** (ชื่อโปรแกรมมาจากตัวแปรที่กำหนดค่าได้
ขณะรัน เช่น `CALL WS-PROGRAM-NAME`) ระบบนี้เลือกใช้ Static CALL ด้วยเหตุผล 2 ข้อ:

1. **`AGINGCLC` เป็น subprogram ที่รู้จักแน่นอนตั้งแต่ตอนออกแบบระบบ** ไม่มีความจำเป็นทางธุรกิจที่จะ
   ต้องสลับไปเรียก subprogram คำนวณอายุหนี้ตัวอื่นขณะโปรแกรมกำลังรันอยู่ (ต่างจากสถานการณ์ที่ Part 043
   ยกตัวอย่างไว้ เช่น ระบบที่ต้องเลือกเรียก subprogram คำนวณภาษีที่แตกต่างกันตามประเทศที่ผู้ใช้เลือก
   ขณะรัน ซึ่งเหมาะกับ Dynamic CALL มากกว่า)
2. **Static CALL ให้ประสิทธิภาพดีกว่าเล็กน้อยและตรวจสอบข้อผิดพลาดได้ตั้งแต่ตอน link** — หากพิมพ์ชื่อ
   โปรแกรมผิด (เช่น `CALL "AGINGCLC"` เป็น `CALL "AGINCLC"`) การ link จะ error ทันทีตอนคอมไพล์ ในขณะที่
   Dynamic CALL ที่ชื่อผิดจะไม่ถูกตรวจพบจนกว่าจะรันไปถึงบรรทัดนั้นจริง ๆ (runtime error) ซึ่งอาจเกิดขึ้น
   ช้ากว่ามากในวงจรการพัฒนา

หากในอนาคตระบบต้องการความยืดหยุ่นมากขึ้น (เช่น รองรับกฎการคำนวณอายุหนี้ที่ต่างกันตามประเภทลูกค้า
โดยเลือก subprogram ที่จะเรียกขณะรัน) การเปลี่ยนไปใช้ Dynamic CALL ในภายหลังทำได้ไม่ยาก เพราะ
`LINKAGE SECTION` และโครงสร้างพารามิเตอร์ยังคงเหมือนเดิมทุกประการ — เปลี่ยนแค่บรรทัด `CALL` เท่านั้น

### ข้อควรระวัง

- `LK-INVOICE-DATE` และ `LK-AS-OF-DATE` ใน `LINKAGE SECTION` **ไม่มีการจองพื้นที่หน่วยความจำเอง**
  (ทบทวนจาก Part 031) มันใช้พื้นที่ของตัวแปรที่โปรแกรมเรียกส่งมาโดยตรง (BY REFERENCE เป็นค่าเริ่มต้น)
  ดังนั้น subprogram นี้**ไม่ควรแก้ไขค่า `LK-INVOICE-DATE`/`LK-AS-OF-DATE`** แม้ตามหลักไวยากรณ์จะทำได้
  ก็ตาม เพราะจะไปแก้ไขตัวแปรต้นทางในโปรแกรมที่เรียกโดยไม่ตั้งใจ (ธรรมเนียมที่ดีคือ input parameter
  ควรอ่านอย่างเดียว output parameter เท่านั้นที่ subprogram เขียนค่ากลับ)
- ผลลัพธ์การคำนวณอายุหนี้ในระบบนี้อ้างอิงจาก **วันที่ออกใบแจ้งหนี้** ไม่ใช่ **วันครบกำหนดชำระ (Due
  Date)** — เป็นการลดความซับซ้อนสำหรับระดับหลักสูตรนี้ ระบบ AR ระดับองค์กรจริงมักมีฟิลด์ Due Date
  แยกต่างหาก (คำนวณจาก Invoice Date + เงื่อนไขเครดิต เช่น Net 30) และคำนวณอายุหนี้จาก Due Date แทน

### แบบฝึกหัดที่ 496.1

**โจทย์**: จงอธิบายว่าทำไมการแยก `AGINGCLC` ออกมาเป็น Subprogram ต่างหาก (แทนที่จะเขียน logic การ
คำนวณอายุหนี้ไว้ในโปรแกรม `ARREPORT` โดยตรง) จึงเป็นการออกแบบที่ดีสำหรับระบบนี้

**เฉลยแนวทาง**: การแยกเป็น Subprogram ทำให้ logic การคำนวณอายุหนี้ **ทดสอบได้อย่างอิสระ** (ตามที่
สาธิตในขั้นตอนนี้ด้วยโปรแกรมทดสอบแยกต่างหาก) โดยไม่ต้องพึ่งพาไฟล์ Indexed หรือโครงสร้างของโปรแกรม
`ARREPORT` ทั้งหมด และยังทำให้ **นำไปใช้ซ้ำได้** หากในอนาคตมีความต้องการคำนวณอายุหนี้จากโปรแกรมอื่น
(เช่น ระบบแจ้งเตือนอัตโนมัติเมื่อลูกหนี้ใกล้ครบ 90 วัน หรือ Dashboard ผู้บริหารแบบ real-time) โดยไม่
ต้องคัดลอก logic เดิมไปเขียนซ้ำ — หากกฎการแบ่งช่วงอายุหนี้เปลี่ยนแปลงในอนาคต (เช่น เปลี่ยนจาก 30/60/90
เป็น 15/45/75) ก็แก้ไขที่ `AGINGCLC.cob` เพียงไฟล์เดียว แล้ว compile ใหม่ ทุกโปรแกรมที่ `CALL`
subprogram นี้จะได้พฤติกรรมใหม่โดยอัตโนมัติทันที

---

## ขั้นตอนที่ 497: โปรแกรม ARREPORT ส่วนที่ 1 — โครงสร้าง Control Break และการอ่านไฟล์ตามลำดับ Key

### แนวคิด

`ARREPORT` เป็นโปรแกรมที่ซับซ้อนที่สุดในระบบ เพราะต้องทำ **Control Break Processing** (เทคนิคจาก
Part 026 และ Part 039): อ่านใบแจ้งหนี้ทั้งหมดเรียงตามลำดับ `INV-NUMBER` แต่ต้อง**จัดกลุ่มและพิมพ์
ยอดรวมแยกตามลูกค้า** ทุกครั้งที่ข้อมูลลูกค้าเปลี่ยน (control break) ขั้นตอนนี้จะอธิบายโครงสร้างหลัก
ก่อน ส่วนขั้นตอนที่ 498 จะอธิบายรายละเอียดการจัดรูปแบบรายงาน

### การออกแบบ WORKING-STORAGE สำหรับ Control Break

| กลุ่มตัวแปร | หน้าที่ |
|---|---|
| `WS-PREV-CUST-ID` | เก็บรหัสลูกค้าของ record ก่อนหน้า ใช้เปรียบเทียบว่าข้อมูลเปลี่ยนกลุ่มหรือยัง |
| `WS-CUST-BUCKET-1..4` | ตัวสะสมยอดค้างของ**ลูกค้าปัจจุบัน**แยกตามช่วงอายุหนี้ 4 ช่วง (รีเซ็ตทุก control break) |
| `WS-GRAND-BUCKET-1..4` | ตัวสะสมยอดรวม**ทั้งระบบ**แยกตามช่วงอายุหนี้ (ไม่รีเซ็ตจนจบโปรแกรม) |

หลักการนี้เหมือนกับที่เรียนมาใน Part 026 (Master-Detail) และ Part 039 (Report Writer Control Break)
ทุกประการ เพียงแต่คราวนี้ implement ด้วย `IF`/`PERFORM` ธรรมดาแทนการใช้ `REPORT SECTION`

### โครงสร้างหลักของโปรแกรม (ยังไม่รวมส่วนจัดรูปแบบรายงานเต็ม)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP497-CONTROL-BREAK-SKELETON.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTOMER.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-CUST-STATUS.

           SELECT INVOICE-FILE ASSIGN TO "INVOICE.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS INV-NUMBER
               FILE STATUS IS WS-INV-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           COPY CUSTREC.

       FD  INVOICE-FILE.
       01  INVOICE-RECORD.
           COPY INVREC.

       WORKING-STORAGE SECTION.
       01  WS-CUST-STATUS            PIC XX.
       01  WS-INV-STATUS             PIC XX.
       01  WS-EOF-FLAG               PIC X(1) VALUE "N".
           88  END-OF-INVOICES                VALUE "Y".
       01  WS-PREV-CUST-ID           PIC 9(5) VALUE ZEROES.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN INPUT CUSTOMER-FILE
           OPEN INPUT INVOICE-FILE

           PERFORM UNTIL END-OF-INVOICES
               READ INVOICE-FILE NEXT RECORD
                   AT END
                       SET END-OF-INVOICES TO TRUE
                   NOT AT END
                       PERFORM PROCESS-ONE-INVOICE
               END-READ
           END-PERFORM

           IF WS-PREV-CUST-ID NOT = ZEROES
               DISPLAY "FINAL SUBTOTAL WOULD PRINT HERE FOR "
                   "CUSTOMER " WS-PREV-CUST-ID
           END-IF

           CLOSE CUSTOMER-FILE
           CLOSE INVOICE-FILE
           STOP RUN.

       PROCESS-ONE-INVOICE.
           IF INV-CUST-ID NOT = WS-PREV-CUST-ID
               IF WS-PREV-CUST-ID NOT = ZEROES
                   DISPLAY "  >> CONTROL BREAK: SUBTOTAL FOR "
                       "CUSTOMER " WS-PREV-CUST-ID
               END-IF
               DISPLAY ">> NOW PROCESSING CUSTOMER "
                   INV-CUST-ID
               MOVE INV-CUST-ID TO WS-PREV-CUST-ID
           END-IF
           DISPLAY "   INVOICE " INV-NUMBER " AMOUNT "
               INV-AMOUNT.
```

คอมไพล์และรันด้วย (ต้องใช้ ISAM build):

```bash
/opt/gnucobol-isam/bin/cobc -I copybooks -x -o step497 step497.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./step497
```

### ผลลัพธ์จริงที่ได้

```
>> NOW PROCESSING CUSTOMER 10001
   INVOICE 100001 AMOUNT 0015000.00
   INVOICE 100002 AMOUNT 0008000.00
  >> CONTROL BREAK: SUBTOTAL FOR CUSTOMER 10001
>> NOW PROCESSING CUSTOMER 10002
   INVOICE 100003 AMOUNT 0012000.00
   INVOICE 100004 AMOUNT 0005000.00
  >> CONTROL BREAK: SUBTOTAL FOR CUSTOMER 10002
>> NOW PROCESSING CUSTOMER 10003
   INVOICE 100005 AMOUNT 0020000.00
   INVOICE 100006 AMOUNT 0003000.00
FINAL SUBTOTAL WOULD PRINT HERE FOR CUSTOMER 10003
```

### อธิบายโค้ดทีละส่วน

- `ACCESS MODE IS SEQUENTIAL` กับ `READ INVOICE-FILE NEXT RECORD` คืนค่า record เรียงตาม `INV-NUMBER`
  จากน้อยไปมากเสมอ (`100001` → `100002` → ... → `100006`) เนื่องจากใบแจ้งหนี้แต่ละใบของลูกค้าเดียวกัน
  ถูกสร้างด้วยเลขที่ต่อเนื่องกัน (`100001`, `100002` เป็นของ `10001`) การอ่านตามลำดับ Key จึง**บังเอิญ**
  ทำให้ข้อมูลของลูกค้าเดียวกันอยู่ติดกัน — **นี่เป็นข้อจำกัดสำคัญที่ต้องระวัง** จะอธิบายในหัวข้อถัดไป
- สังเกตรูปแบบ Control Break แบบคลาสสิก: **ตรวจพบการเปลี่ยนกลุ่มก่อน** (`IF INV-CUST-ID NOT =
  WS-PREV-CUST-ID`) แล้ว**พิมพ์ยอดรวมของกลุ่มเก่าก่อน**พิมพ์ข้อมูลของกลุ่มใหม่ — ทบทวนตรงกับรูปแบบที่
  เรียนมาใน Part 026 ขั้นตอนที่ 253 ทุกประการ
- `IF WS-PREV-CUST-ID NOT = ZEROES` ป้องกันไม่ให้โปรแกรมพยายามพิมพ์ยอดรวมของ "ลูกค้าก่อนหน้า" ที่
  ไม่มีอยู่จริงตอนเริ่มโปรแกรม (record แรกสุด) — ค่าเริ่มต้น `ZEROES` ไม่ตรงกับ `CUST-ID` ใด ๆ ที่
  เป็นไปได้จริง (เพราะ `CUST-ID` เริ่มที่ `10001`)
- **ต้องมีการพิมพ์ยอดรวมของกลุ่มสุดท้ายหลังลูปจบ** (`IF WS-PREV-CUST-ID NOT = ZEROES` หลัง
  `PERFORM UNTIL`) เพราะ control break ที่ตรวจจับภายในลูปจะจับได้แค่ตอน "เปลี่ยนกลุ่ม" เท่านั้น —
  กลุ่มสุดท้าย (`10003`) ไม่มีการเปลี่ยนกลุ่มอีกหลังจากนั้น (ไฟล์จบพอดี) จึงต้องมีโค้ดจัดการเพิ่มเติม
  นอกลูปเสมอ เป็นกับดักคลาสสิกของ Control Break Processing ที่เรียนมาตั้งแต่ Part 026

### ข้อจำกัดสำคัญที่ต้องรู้: การอ่านตามลำดับ Key ไม่ได้ตัดกลุ่มตามลูกค้าเสมอไป

ในตัวอย่างนี้ ข้อมูลใบแจ้งหนี้ถูกออกแบบให้เลขที่ต่อเนื่องกันตรงกับลูกค้าเดียวกันพอดี (`100001-100002`
= `10001`) ทำให้การอ่านตามลำดับ `INV-NUMBER` ดูเหมือนจัดกลุ่มลูกค้าให้อัตโนมัติ **แต่ในระบบจริง
ใบแจ้งหนี้มักถูกออกเรียงตามลำดับเวลาที่เกิดขึ้นจริง ไม่ใช่เรียงตามลูกค้า** (ลูกค้า A อาจมีใบแจ้งหนี้
เลขที่ `100001` แล้วลูกค้า B ออกใบที่ `100002` แล้วลูกค้า A กลับมาออกใบที่ `100003` อีกครั้ง) ทำให้
ข้อมูลของลูกค้าเดียวกัน**ไม่อยู่ติดกัน**เมื่ออ่านตามลำดับ `INV-NUMBER` — ในสถานการณ์นั้น Control Break
ตามที่เขียนไว้ในขั้นตอนนี้**จะทำงานผิดพลาด** (จะพิมพ์ยอดรวมของลูกค้า A แยกเป็นหลายก้อนแทนที่จะรวมเป็น
ก้อนเดียว)

วิธีแก้ปัญหาในโลกจริงคือ **`SORT`** ไฟล์ใบแจ้งหนี้ตาม `INV-CUST-ID` ก่อนประมวลผล (เทคนิคจาก Part 027)
หรือออกแบบ Alternate Index บน `INV-CUST-ID` (เทคนิคจะเรียนเต็มรูปแบบใน Part 056 เรื่อง VSAM Alternate
Index) — ระบบตัวอย่างในเอกสารนี้ใช้ข้อมูลที่ถูกออกแบบให้เรียงกันพอดีเพื่อความง่ายในการสอน แต่ผู้เรียน
ต้องเข้าใจข้อจำกัดนี้ก่อนนำแนวทางนี้ไปใช้กับข้อมูลจริงที่ไม่ได้เรียงลำดับแบบนี้

### ข้อควรระวัง

- **อย่านำ pattern การอ่านตามลำดับ Key หลักไปใช้ตัดกลุ่มโดยไม่ตรวจสอบก่อนว่าข้อมูลจริงเรียงลำดับตาม
  ฟิลด์ที่ต้องการ group จริงหรือไม่** ความผิดพลาดนี้พบได้บ่อยมากในโปรแกรมที่พัฒนาขึ้นอย่างเร่งรีบ และ
  มักไม่ถูกตรวจพบตอนทดสอบเพราะข้อมูลทดสอบบังเอิญเรียงลำดับพอดี (เหมือนในตัวอย่างนี้) แต่ไปพังตอนใช้งาน
  จริงกับข้อมูลที่มีรูปแบบต่างออกไป
- โปรเจกต์นี้จะระบุข้อจำกัดนี้ไว้อย่างตรงไปตรงมาในขั้นตอนที่ 500 เป็นหนึ่งใน "แนวทางการพัฒนาต่อยอด"
  ของระบบ

### แบบฝึกหัดที่ 497.1

**โจทย์**: หากไฟล์ใบแจ้งหนี้มีลำดับ (เรียงตาม `INV-NUMBER`) เป็น: `100001`(cust `10001`),
`100002`(cust `10002`), `100003`(cust `10001`) จงอธิบายว่า Control Break ตามโค้ดในขั้นตอนนี้จะพิมพ์
ผลลัพธ์ผิดพลาดอย่างไร

**เฉลย**: โปรแกรมจะพิมพ์: เริ่มประมวลผลลูกค้า `10001` (ใบ `100001`) → ตรวจพบเปลี่ยนกลุ่มเป็น `10002`
→ พิมพ์ยอดรวมของ `10001` (มีแค่ใบ `100001`) → เริ่มประมวลผลลูกค้า `10002` (ใบ `100002`) → ตรวจพบ
เปลี่ยนกลุ่มกลับมาเป็น `10001` อีกครั้ง → พิมพ์ยอดรวมของ `10002` (มีแค่ใบ `100002`) → เริ่มประมวลผล
ลูกค้า `10001` **เป็นครั้งที่สอง** (ใบ `100003`) → จบไฟล์ → พิมพ์ยอดรวมสุดท้ายของ `10001` (มีแค่ใบ
`100003`) ผลลัพธ์สุดท้ายคือ**ลูกค้า `10001` ถูกพิมพ์ยอดรวมแยกเป็น 2 ก้อน** (ก้อนแรกจากใบ `100001`
เพียงใบเดียว และก้อนที่สองจากใบ `100003` เพียงใบเดียว) แทนที่จะรวมกันเป็นยอดเดียว `100001+100003`
ตามที่ควรจะเป็น — นี่คือบั๊ก Control Break แบบคลาสสิกที่เกิดจากข้อมูลไม่ได้เรียงลำดับตามฟิลด์ที่ใช้
จัดกลุ่มจริง

---

## ขั้นตอนที่ 498: โปรแกรม ARREPORT ส่วนที่ 2 — การจัดรูปแบบรายงานฉบับสมบูรณ์

### แนวคิด

ขั้นตอนนี้เติมเต็มโปรแกรม `ARREPORT` จากขั้นตอนที่ 497 ให้สมบูรณ์: เพิ่มการเรียก `AGINGCLC` เพื่อ
คำนวณ bucket จริง, การสะสมยอดแยกตาม bucket ทั้งระดับลูกค้าและระดับทั้งระบบ, การจัดรูปแบบตัวเลขด้วย
Edited PICTURE (`ZZZ,ZZ9.99` ทบทวนจาก Part 006), และการข้ามใบแจ้งหนี้ที่ชำระครบแล้ว (ตามข้อกำหนดข้อ 8)

### ซอร์สโค้ดฉบับสมบูรณ์: `ARREPORT.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ARREPORT.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTOMER.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-CUST-STATUS.

           SELECT INVOICE-FILE ASSIGN TO "INVOICE.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS INV-NUMBER
               FILE STATUS IS WS-INV-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           COPY CUSTREC.

       FD  INVOICE-FILE.
       01  INVOICE-RECORD.
           COPY INVREC.

       WORKING-STORAGE SECTION.
       01  WS-CUST-STATUS            PIC XX.
       01  WS-INV-STATUS             PIC XX.
       01  WS-AS-OF-DATE             PIC 9(8) VALUE 20260401.
       01  WS-EOF-FLAG               PIC X(1) VALUE "N".
           88  END-OF-INVOICES                VALUE "Y".

       01  WS-DAYS-OVERDUE           PIC S9(6).
       01  WS-AGING-BUCKET           PIC 9(1).
       01  WS-OUTSTANDING            PIC 9(7)V99.

       01  WS-PREV-CUST-ID           PIC 9(5) VALUE ZEROES.
       01  WS-CUST-NAME-SAVE         PIC X(20).

      *> Per-customer accumulators for the four aging buckets, reset
      *> at every customer control break.
       01  WS-CUST-BUCKET-1          PIC 9(7)V99 VALUE 0.
       01  WS-CUST-BUCKET-2          PIC 9(7)V99 VALUE 0.
       01  WS-CUST-BUCKET-3          PIC 9(7)V99 VALUE 0.
       01  WS-CUST-BUCKET-4          PIC 9(7)V99 VALUE 0.
       01  WS-CUST-TOTAL             PIC 9(7)V99 VALUE 0.

      *> Grand totals across all customers.
       01  WS-GRAND-BUCKET-1         PIC 9(8)V99 VALUE 0.
       01  WS-GRAND-BUCKET-2         PIC 9(8)V99 VALUE 0.
       01  WS-GRAND-BUCKET-3         PIC 9(8)V99 VALUE 0.
       01  WS-GRAND-BUCKET-4         PIC 9(8)V99 VALUE 0.
       01  WS-GRAND-TOTAL            PIC 9(8)V99 VALUE 0.

      *> Edited (formatted) fields for the report body.
       01  WS-LINE-CUST-NAME         PIC X(20).
       01  WS-LINE-BUCKET-1          PIC ZZZ,ZZ9.99.
       01  WS-LINE-BUCKET-2          PIC ZZZ,ZZ9.99.
       01  WS-LINE-BUCKET-3          PIC ZZZ,ZZ9.99.
       01  WS-LINE-BUCKET-4          PIC ZZZ,ZZ9.99.
       01  WS-LINE-TOTAL             PIC ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM PRINT-REPORT-HEADER
           OPEN INPUT CUSTOMER-FILE
           OPEN INPUT INVOICE-FILE

           PERFORM UNTIL END-OF-INVOICES
               READ INVOICE-FILE NEXT RECORD
                   AT END
                       SET END-OF-INVOICES TO TRUE
                   NOT AT END
                       PERFORM PROCESS-ONE-INVOICE
               END-READ
           END-PERFORM

           IF WS-PREV-CUST-ID NOT = ZEROES
               PERFORM PRINT-CUSTOMER-SUBTOTAL
           END-IF

           PERFORM PRINT-GRAND-TOTAL

           CLOSE CUSTOMER-FILE
           CLOSE INVOICE-FILE
           STOP RUN.

       PRINT-REPORT-HEADER.
           DISPLAY "===============================================".
           DISPLAY "     ACCOUNTS RECEIVABLE AGING REPORT".
           DISPLAY "     AS OF DATE: " WS-AS-OF-DATE.
           DISPLAY "===============================================".
           DISPLAY "CUSTOMER             CURRENT   31-60  61-90   91+".
           DISPLAY "-----------------------------------------------".

       PROCESS-ONE-INVOICE.
           COMPUTE WS-OUTSTANDING = INV-AMOUNT - INV-PAID-AMOUNT
      *> Fully paid invoices (outstanding balance zero) do not
      *> belong on an aging report at all - skip them (requirement
      *> #8 from step 491).
           IF WS-OUTSTANDING > 0
               IF INV-CUST-ID NOT = WS-PREV-CUST-ID
                   IF WS-PREV-CUST-ID NOT = ZEROES
                       PERFORM PRINT-CUSTOMER-SUBTOTAL
                   END-IF
                   PERFORM START-NEW-CUSTOMER
               END-IF

               CALL "AGINGCLC" USING INV-DATE, WS-AS-OF-DATE,
                    WS-DAYS-OVERDUE, WS-AGING-BUCKET

               EVALUATE WS-AGING-BUCKET
                   WHEN 1
                       ADD WS-OUTSTANDING TO WS-CUST-BUCKET-1
                   WHEN 2
                       ADD WS-OUTSTANDING TO WS-CUST-BUCKET-2
                   WHEN 3
                       ADD WS-OUTSTANDING TO WS-CUST-BUCKET-3
                   WHEN 4
                       ADD WS-OUTSTANDING TO WS-CUST-BUCKET-4
               END-EVALUATE
               ADD WS-OUTSTANDING TO WS-CUST-TOTAL
           END-IF.

       START-NEW-CUSTOMER.
           MOVE INV-CUST-ID TO WS-PREV-CUST-ID
           MOVE INV-CUST-ID TO CUST-ID
           READ CUSTOMER-FILE
           MOVE CUST-NAME TO WS-CUST-NAME-SAVE
           MOVE 0 TO WS-CUST-BUCKET-1
           MOVE 0 TO WS-CUST-BUCKET-2
           MOVE 0 TO WS-CUST-BUCKET-3
           MOVE 0 TO WS-CUST-BUCKET-4
           MOVE 0 TO WS-CUST-TOTAL.

       PRINT-CUSTOMER-SUBTOTAL.
           MOVE WS-CUST-NAME-SAVE TO WS-LINE-CUST-NAME
           MOVE WS-CUST-BUCKET-1 TO WS-LINE-BUCKET-1
           MOVE WS-CUST-BUCKET-2 TO WS-LINE-BUCKET-2
           MOVE WS-CUST-BUCKET-3 TO WS-LINE-BUCKET-3
           MOVE WS-CUST-BUCKET-4 TO WS-LINE-BUCKET-4
           DISPLAY WS-LINE-CUST-NAME " " WS-LINE-BUCKET-1 " "
               WS-LINE-BUCKET-2 " " WS-LINE-BUCKET-3 " "
               WS-LINE-BUCKET-4

           ADD WS-CUST-BUCKET-1 TO WS-GRAND-BUCKET-1
           ADD WS-CUST-BUCKET-2 TO WS-GRAND-BUCKET-2
           ADD WS-CUST-BUCKET-3 TO WS-GRAND-BUCKET-3
           ADD WS-CUST-BUCKET-4 TO WS-GRAND-BUCKET-4
           ADD WS-CUST-TOTAL TO WS-GRAND-TOTAL.

       PRINT-GRAND-TOTAL.
           DISPLAY "-----------------------------------------------".
           MOVE WS-GRAND-BUCKET-1 TO WS-LINE-BUCKET-1
           MOVE WS-GRAND-BUCKET-2 TO WS-LINE-BUCKET-2
           MOVE WS-GRAND-BUCKET-3 TO WS-LINE-BUCKET-3
           MOVE WS-GRAND-BUCKET-4 TO WS-LINE-BUCKET-4
           MOVE WS-GRAND-TOTAL TO WS-LINE-TOTAL
           DISPLAY "GRAND TOTAL          " WS-LINE-BUCKET-1 " "
               WS-LINE-BUCKET-2 " " WS-LINE-BUCKET-3 " "
               WS-LINE-BUCKET-4
           DISPLAY "TOTAL RECEIVABLES OUTSTANDING: " WS-LINE-TOTAL.
```

คอมไพล์ (เชื่อมกับ `AGINGCLC.o` ที่คอมไพล์ไว้แล้วจากขั้นตอนที่ 496) และรันด้วย:

```bash
/opt/gnucobol-isam/bin/cobc -c AGINGCLC.cob
/opt/gnucobol-isam/bin/cobc -I copybooks -x -o ARREPORT ARREPORT.cob AGINGCLC.o
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./ARREPORT
```

### ผลลัพธ์จริงที่ได้ (รันหลังจาก ARSETUP และ ARPAYMNT ตามลำดับ)

```
===============================================
     ACCOUNTS RECEIVABLE AGING REPORT
     AS OF DATE: 20260401
===============================================
CUSTOMER             CURRENT   31-60  61-90   91+
-----------------------------------------------
SIAM TRADING CO       10,000.00   8,000.00       0.00       0.00
BANGKOK SUPPLIES LTD       0.00       0.00  12,000.00   5,000.00
THAI EXPORT PARTNERS       0.00       0.00       0.00   3,000.00
-----------------------------------------------
GRAND TOTAL           10,000.00   8,000.00  12,000.00   8,000.00
TOTAL RECEIVABLES OUTSTANDING:  38,000.00
```

### อธิบายโค้ดทีละส่วน — ตรวจสอบความถูกต้องด้วยมือทุกตัวเลข

**SIAM TRADING CO (10001)**:
- ใบ `100001`: ยอดเดิม `15,000.00` ชำระไปแล้ว `5,000.00` (จากขั้นตอนที่ 495) → คงเหลือ
  `10,000.00` วันที่ออก `2026-03-25` → 7 วันจนถึง as-of date → bucket 1 (Current)
- ใบ `100002`: ยอดเดิม `8,000.00` ยังไม่ได้ชำระเลย → คงเหลือ `8,000.00` วันที่ออก `2026-02-10`
  → 50 วัน → bucket 2 (31-60)
- ผลรวม: Current = `10,000.00`, 31-60 = `8,000.00`, 61-90 = `0`, 91+ = `0` — **ตรงกับรายงานเป๊ะ**

**BANGKOK SUPPLIES LTD (10002)**:
- ใบ `100003`: `12,000.00` ยังไม่ได้ชำระ วันที่ออก `2026-01-05` → 86 วัน → bucket 3 (61-90)
- ใบ `100004`: `5,000.00` ยังไม่ได้ชำระ วันที่ออก `2025-11-20` → 132 วัน → bucket 4 (91+)
- ผลรวม: Current = `0`, 31-60 = `0`, 61-90 = `12,000.00`, 91+ = `5,000.00` — **ตรงกับรายงานเป๊ะ**

**THAI EXPORT PARTNERS (10003)**:
- ใบ `100005`: ชำระครบเต็มจำนวนแล้ว (จากขั้นตอนที่ 495) → คงเหลือ `0` → **ถูกข้ามจากรายงานตามที่
  ออกแบบไว้** (ข้อกำหนดข้อ 8) — สังเกตว่าใบนี้ไม่ปรากฏในผลรวมของ `10003` เลย
- ใบ `100006`: `3,000.00` ยังไม่ได้ชำระ วันที่ออก `2025-12-01` → 121 วัน → bucket 4 (91+)
- ผลรวม: Current = `0`, 31-60 = `0`, 61-90 = `0`, 91+ = `3,000.00` — **ตรงกับรายงานเป๊ะ**

**Grand Total**: `10,000 + 8,000 + 12,000 + 5,000 + 3,000 = 38,000.00` ตรงกับ
`TOTAL RECEIVABLES OUTSTANDING: 38,000.00` ที่รายงานแสดง — ยืนยันว่าตรรกะการสะสมยอดทั้งระดับลูกค้า
และระดับ Grand Total ถูกต้องสมบูรณ์ 100%

### ทำไมเลือกใช้ Formatted DISPLAY แทน Report Writer สำหรับรายงานนี้

Part 038-039 สอน **Report Writer Feature** (`REPORT SECTION`, `RD`, `CONTROL IS`, `PAGE HEADING`
ฯลฯ) ไว้แล้วเป็นเครื่องมือเฉพาะทางสำหรับสร้างรายงาน ผู้เรียนหลายคนอาจสงสัยว่าทำไมโปรเจกต์รวบยอดนี้
เลือกใช้ `DISPLAY` ที่จัดรูปแบบด้วยมือแทนที่จะใช้ `REPORT SECTION` ทั้งที่เรียนมาแล้ว เหตุผลมี 3 ข้อ:

1. **ความชัดเจนของการควบคุม Logic การข้ามข้อมูล**: ข้อกำหนดข้อ 8 ที่ต้อง**ข้ามใบแจ้งหนี้ที่ชำระครบ
   แล้วโดยสิ้นเชิง** (ไม่ใช่แค่แสดงเป็น 0) ทำได้ตรงไปตรงมาด้วย `IF WS-OUTSTANDING > 0` ควบคุมการ
   ประมวลผลทั้งหมดในโปรแกรมปกติ ในขณะที่ `REPORT SECTION` มีกลไกการข้ามบรรทัด (`PRESENT WHEN`) ที่
   ต้องเรียนรู้ไวยากรณ์เพิ่มเติมและอาจซับซ้อนกว่าสำหรับเงื่อนไขที่ซับซ้อนแบบนี้
2. **ความสามารถแทรก `CALL` subprogram กลางกระบวนการสร้างรายงาน**: การเรียก `AGINGCLC` เพื่อคำนวณ
   bucket ต้องเกิดขึ้น**ระหว่าง**การประมวลผลแต่ละ record ก่อนตัดสินใจว่าจะสะสมยอดไปที่ bucket ไหน —
   การผสาน `CALL` เข้ากับ `REPORT SECTION` ที่ควบคุมการพิมพ์ด้วยกลไกภายในของตัวเองทำได้ แต่ต้องออกแบบ
   ให้รอบคอบกว่าการควบคุมด้วยมือใน `PROCEDURE DIVISION` ปกติ
3. **ความสม่ำเสมอกับรูปแบบการทดสอบของหลักสูตร**: ผลลัพธ์จาก `DISPLAY` ทำนายได้ตรงไปตรงมาและทดสอบ
   เปรียบเทียบ (string comparison) ได้ง่ายกว่าผลลัพธ์จาก `REPORT SECTION` ที่มักมีรายละเอียดการจัด
   หน้ากระดาษ (Page Break, Line Spacing) แทรกอยู่ ซึ่งไม่ใช่จุดเน้นหลักของโปรเจกต์นี้

**นี่ไม่ได้แปลว่า Report Writer ด้อยกว่า** — ในระบบ production จริงที่ต้องพิมพ์รายงานออกกระดาษหรือ
ไฟล์ PDF ที่มีหลายหน้าพร้อม Page Header/Footer สวยงาม `REPORT SECTION` มักเป็นทางเลือกที่เหมาะสมกว่า
มาก (ทบทวนเหตุผลได้จาก Part 038 ขั้นตอนที่ 371) โปรเจกต์นี้จึงระบุไว้ในขั้นตอนที่ 500 ว่า "สร้าง
รายงานด้วย Report Writer แทน DISPLAY" เป็นหนึ่งในแนวทางการพัฒนาต่อยอดที่แนะนำให้ผู้เรียนลองฝึกฝนเอง

### ข้อควรระวัง

- `PIC ZZZ,ZZ9.99` (Edited PICTURE ทบทวนจาก Part 006) ใช้ `Z` เพื่อระงับเลขศูนย์นำหน้า (zero
  suppression) ทำให้ตัวเลขอ่านง่ายกว่ารูปแบบ `9999999.99` ดิบ ๆ แต่ต้องระวังว่าฟิลด์ผลลัพธ์ที่ได้
  ยังคงมีความกว้างคงที่เสมอ (เติมช่องว่างด้านหน้าแทนเลขศูนย์ที่ถูกระงับ) — เมื่อนำไปต่อกับข้อความอื่น
  ด้วยการเว้นวรรค (`" "`) ใน `DISPLAY` อาจทำให้แนวคอลัมน์ดูไม่สวยงามนักหากความกว้างของชื่อลูกค้า
  (`PIC X(20)`) ไม่พอดีกับความยาวชื่อจริง — ระบบ production จริงมักใช้ Report Writer (Part 038-039)
  หรือจัดรูปแบบผ่าน Screen Section (Part 040) เพื่อควบคุมตำแหน่งคอลัมน์ได้ละเอียดกว่านี้
- `CALL "AGINGCLC"` ถูกเรียก**เฉพาะเมื่อ `WS-OUTSTANDING > 0` เท่านั้น** — เป็นการเพิ่มประสิทธิภาพ
  เล็กน้อย (ไม่เสียเวลาคำนวณอายุหนี้ของใบแจ้งหนี้ที่ไม่เกี่ยวข้องกับรายงานอยู่แล้ว) แต่สำคัญกว่านั้นคือ
  ทำให้ตรรกะของโปรแกรมสื่อความหมายชัดเจนว่า "คำนวณอายุหนี้เฉพาะรายการที่ยังค้างอยู่จริงเท่านั้น"

### แบบฝึกหัดที่ 498.1

**โจทย์**: หากใบแจ้งหนี้ `100004` (BANGKOK SUPPLIES LTD, `5,000.00`, 132 วัน) ได้รับชำระบางส่วน
`2,000.00` ก่อนรันรายงาน จงคำนวณว่ายอดในช่วง 91+ ของลูกค้า `10002` และ Grand Total ช่วง 91+ ควรเปลี่ยน
เป็นเท่าใด

**เฉลยแนวทาง**: ยอดคงเหลือใหม่ของใบ `100004` = `5,000.00 - 2,000.00 = 3,000.00` (ยังคงมากกว่า 0
จึงยังปรากฏในรายงาน และอายุหนี้ยังคงเป็น 132 วัน เพราะวันที่ออกใบแจ้งหนี้ไม่เปลี่ยนแปลงจากการชำระเงิน)
ดังนั้นช่วง 91+ ของลูกค้า `10002` จะเปลี่ยนจาก `5,000.00` เป็น `3,000.00` และ Grand Total ช่วง 91+
จะเปลี่ยนจาก `8,000.00` (`5,000+3,000` เดิม) เป็น `6,000.00` (`3,000+3,000` ใหม่) — ลดลง `2,000.00`
พอดีตามจำนวนเงินที่ชำระเข้ามา และ `TOTAL RECEIVABLES OUTSTANDING` จะลดลงจาก `38,000.00` เป็น
`36,000.00` เช่นกัน

---

## ขั้นตอนที่ 499: ทดสอบระบบแบบครบวงจร (End-to-End Integration Test)

### แนวคิด

ขั้นตอนนี้จะรันทั้ง 3 โปรแกรมหลักตามลำดับที่ถูกต้องบนไฟล์ชุดเดียวกันตั้งแต่ต้นจนจบ เพื่อพิสูจน์ว่า
**ระบบทั้งหมดทำงานร่วมกันได้อย่างถูกต้องสมบูรณ์** — นี่คือสิ่งที่เรียกว่า **Integration Testing**
(ทดสอบการทำงานร่วมกันของหลายส่วนประกอบ) ต่างจาก Unit Testing ในขั้นตอนที่ 496 ที่ทดสอบ `AGINGCLC`
แยกเดี่ยว ๆ

### ลำดับการรันที่ถูกต้อง

```bash
# 1. เตรียมไฟล์ธุรกรรมการชำระเงิน
printf '100001000500000\n100005002000000\n999999000010000\n' > PAYMENTS.DAT

# 2. คอมไพล์ทุกโปรแกรม (AGINGCLC ต้องคอมไพล์เป็น .o ก่อน เพื่อ link เข้ากับ ARREPORT)
/opt/gnucobol-isam/bin/cobc -c AGINGCLC.cob
/opt/gnucobol-isam/bin/cobc -I copybooks -x -o ARSETUP  ARSETUP.cob
/opt/gnucobol-isam/bin/cobc -I copybooks -x -o ARPAYMNT ARPAYMNT.cob
/opt/gnucobol-isam/bin/cobc -I copybooks -x -o ARREPORT ARREPORT.cob AGINGCLC.o

# 3. รันตามลำดับ: SETUP -> PAYMENT -> REPORT
export LD_LIBRARY_PATH=/opt/gnucobol-isam/lib
./ARSETUP
./ARPAYMNT
./ARREPORT
```

### ผลลัพธ์จริงที่ได้ (รันเรียงกันทั้งหมดในคำสั่งเดียว)

```
=== STEP 1: ARSETUP ===
AR-SETUP COMPLETE.

=== STEP 2: ARPAYMNT ===
APPLIED 0005000.00 TO INVOICE 100001 (CUSTOMER 10001) NEW INVOICE BALANCE=0010000.00
APPLIED 0020000.00 TO INVOICE 100005 (CUSTOMER 10003) NEW INVOICE BALANCE=0000000.00
PAYMENTS APPLIED : 002
PAYMENTS REJECTED: 001

=== ERROR LOG CONTENT ===
PAYMENT REJECTED: INVOICE 999999 NOT FOUND (STATUS 23)

=== STEP 3: ARREPORT ===
===============================================
     ACCOUNTS RECEIVABLE AGING REPORT
     AS OF DATE: 20260401
===============================================
CUSTOMER             CURRENT   31-60  61-90   91+
-----------------------------------------------
SIAM TRADING CO       10,000.00   8,000.00       0.00       0.00
BANGKOK SUPPLIES LTD       0.00       0.00  12,000.00   5,000.00
THAI EXPORT PARTNERS       0.00       0.00       0.00   3,000.00
-----------------------------------------------
GRAND TOTAL           10,000.00   8,000.00  12,000.00   8,000.00
TOTAL RECEIVABLES OUTSTANDING:  38,000.00
```

### การตรวจสอบความสอดคล้องของข้อมูล (Reconciliation Check)

ขั้นตอนสำคัญที่โปรแกรมเมอร์มืออาชีพทำเสมอหลัง Integration Test คือ **การตรวจสอบไขว้ (Cross-Check)**
ว่าตัวเลขจากมุมมองที่ต่างกันของระบบสอดคล้องกันหรือไม่ ลองตรวจสอบยอด `CUST-BALANCE` ในไฟล์ลูกค้า
เทียบกับผลรวมในรายงานอายุหนี้ ด้วยโปรแกรมเสริมสั้น ๆ ที่อ่านและแสดงยอดคงเหลือของลูกค้าทุกราย:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CHECKBAL.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUSTOMER.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           COPY CUSTREC.

       WORKING-STORAGE SECTION.
       01  WS-STATUS                 PIC XX.
       01  WS-EOF-FLAG               PIC X(1) VALUE "N".
           88  END-OF-CUSTOMERS               VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN INPUT CUSTOMER-FILE
           PERFORM UNTIL END-OF-CUSTOMERS
               READ CUSTOMER-FILE NEXT RECORD
                   AT END
                       SET END-OF-CUSTOMERS TO TRUE
                   NOT AT END
                       DISPLAY CUST-ID " " CUST-NAME " BAL="
                           CUST-BALANCE
               END-READ
           END-PERFORM
           CLOSE CUSTOMER-FILE
           STOP RUN.
```

```bash
/opt/gnucobol-isam/bin/cobc -I copybooks -x -o CHECKBAL CHECKBAL.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./CHECKBAL
```

**ผลลัพธ์จริงที่ได้**:

```
10001 SIAM TRADING CO      BAL=0018000.00
10002 BANGKOK SUPPLIES LTD BAL=0017000.00
10003 THAI EXPORT PARTNERS BAL=0003000.00
```

**ตรวจสอบไขว้**:

| ลูกค้า | `CUST-BALANCE` (จากไฟล์ลูกค้า) | ผลรวมจากรายงานอายุหนี้ | ตรงกันหรือไม่ |
|---|---|---|---|
| SIAM TRADING CO | 18,000.00 | 10,000.00 + 8,000.00 = 18,000.00 | ✅ ตรงกัน |
| BANGKOK SUPPLIES LTD | 17,000.00 | 12,000.00 + 5,000.00 = 17,000.00 | ✅ ตรงกัน |
| THAI EXPORT PARTNERS | 3,000.00 | 3,000.00 | ✅ ตรงกัน |

**ยอดตรงกันทั้ง 3 ลูกค้า 100%** — นี่คือหลักฐานที่แสดงว่า **โปรแกรม `ARPAYMNT` ปรับปรุงยอด
`CUST-BALANCE` ในไฟล์ลูกค้าถูกต้องตรงกับยอดคงเหลือจริงของใบแจ้งหนี้แต่ละใบ** ซึ่งเป็นการยืนยันความ
ถูกต้องของระบบทั้งหมดจากคนละมุมมอง (ไฟล์ลูกค้า vs ไฟล์ใบแจ้งหนี้ที่ผ่านการคำนวณอายุหนี้) — เทคนิค
Reconciliation แบบนี้เป็นวินัยสำคัญมากในระบบบัญชีจริงทุกระบบ เพราะความผิดพลาดด้านตัวเลขทางการเงิน
ที่ตรวจไม่พบอาจสร้างความเสียหายร้ายแรงได้

### หมายเหตุเรื่องวิธีการทดสอบ

สังเกตว่า**ทุกโปรแกรมในระบบนี้ไม่มีการใช้ `ACCEPT` รับข้อมูลแบบโต้ตอบเลย** ต่างจากโปรเจกต์เฟส 1
(Part 015) ที่ต้องใช้เทคนิค stdin redirection (`printf ... | ./program`) เพื่อทดสอบโปรแกรมที่รอรับ
ค่าจากคีย์บอร์ด — ระบบบัญชีลูกหนี้นี้ถูกออกแบบตามรูปแบบ **Batch Processing** ที่แท้จริง (ตามที่ระบุไว้
ในคำนำของ Part นี้): ข้อมูลนำเข้าทั้งหมดมาจากไฟล์ที่เตรียมไว้ล่วงหน้า (`PAYMENTS.DAT`) ไม่ใช่จาก
การพิมพ์แบบโต้ตอบ — นี่คือรูปแบบที่ใกล้เคียงกับการทำงานจริงของระบบ Mainframe ที่รันผ่าน Job Scheduler
โดยไม่มีคนเฝ้าหน้าจอ ซึ่งเป็นการปูทางความเข้าใจสำหรับเฟส 4 ของหลักสูตร (Mainframe/JCL) ที่กำลังจะเริ่ม
ใน Part 051 ถัดไป

### ข้อควรระวัง

- ก่อนรัน Integration Test ทุกครั้งควรพิจารณาว่าจะ**เริ่มจากข้อมูลสะอาด** (รัน `ARSETUP` ก่อนเสมอ)
  หรือ**ทดสอบต่อจากสถานะเดิม** (ข้าม `ARSETUP` เพื่อดูผลสะสม) ให้ชัดเจน เพราะทั้งสองแนวทางให้ผลลัพธ์
  ต่างกันโดยสิ้นเชิงตามที่พิสูจน์ในขั้นตอนที่ 494
- การตรวจสอบไขว้ (Reconciliation) ไม่ควรเป็นขั้นตอนที่ทำเฉพาะตอนพัฒนาเท่านั้น ระบบบัญชีระดับองค์กร
  จริงมักมีโปรแกรม Reconciliation อัตโนมัติที่รันเป็นประจำ (เช่น ทุกคืน) เพื่อตรวจจับความผิดปกติของ
  ข้อมูลตั้งแต่เนิ่น ๆ ก่อนที่จะสะสมจนกลายเป็นปัญหาใหญ่

### แบบฝึกหัดที่ 499.1

**โจทย์**: จงอธิบายว่าทำไมการตรวจสอบไขว้ (Reconciliation) ระหว่าง `CUST-BALANCE` กับผลรวมจากรายงาน
อายุหนี้ จึง**ไม่สามารถจับข้อผิดพลาดทุกประเภท**ได้ ยกตัวอย่างสถานการณ์ที่ยอดทั้งสองตรงกันแต่ข้อมูล
ยังคงผิดพลาดอยู่

**เฉลยแนวทาง**: การตรวจสอบไขว้แบบนี้จับได้เฉพาะ**ความไม่สอดคล้องกันระหว่างสองแหล่งข้อมูล** แต่ไม่
สามารถจับ**ข้อผิดพลาดที่เกิดขึ้นเหมือนกันในทั้งสองแหล่งพร้อมกัน**ได้ ตัวอย่างเช่น หากโปรแกรม
`ARPAYMNT` มีบั๊กที่คำนวณการหักลบผิดทั้งใน `INV-PAID-AMOUNT` และ `CUST-BALANCE` ด้วยสูตรที่ผิดเหมือนกัน
ทั้งคู่ (เช่น บวกแทนที่จะลบทั้งสองจุด) ยอดทั้งสองก็ยังคง "ตรงกัน" ตามความหมายของการตรวจสอบไขว้นี้ แม้
ตัวเลขจริง ๆ จะผิดทั้งระบบก็ตาม การตรวจสอบไขว้จึงเป็นเพียง**เครื่องมือหนึ่ง**ในการยืนยันความถูกต้อง
ไม่ใช่การรับประกันความถูกต้อง 100% — ยังคงต้องมีการทดสอบด้วยการคำนวณด้วยมือเทียบกับค่าที่คาดหวังจริง
(ตามที่ทำในขั้นตอนที่ 498) ควบคู่กันไปเสมอ

---

## ขั้นตอนที่ 500: สรุปโปรเจกต์เฟส 3 และทบทวนสถาปัตยกรรมทั้งระบบ

### ทบทวนสิ่งที่ระบบทำได้สำเร็จ

ระบบบัญชีลูกหนี้ที่สร้างขึ้นใน Part นี้ตอบโจทย์ครบทั้ง 8 ข้อจาก Requirements ในขั้นตอนที่ 491:

| ข้อกำหนด | วิธีที่ระบบตอบโจทย์ |
|---|---|
| 1-2. เก็บข้อมูลลูกค้า/ใบแจ้งหนี้ | Indexed File (`CUSTOMER.DAT`, `INVOICE.DAT`) ผ่าน Copybook ร่วม |
| 3. รับชำระเงินและปรับยอด | `ARPAYMNT` อัปเดตทั้งสองไฟล์พร้อมกันในทรานแซคชันเดียว |
| 4. ไม่ล่มเมื่อข้อมูลผิดพลาด | `DECLARATIVES` (Part 048) จัดการข้อผิดพลาดแบบรวมศูนย์ |
| 5. คำนวณวันที่ผ่านไป | `AGINGCLC` subprogram ด้วย `FUNCTION INTEGER-OF-DATE` (Part 036/047) |
| 6. จัดกลุ่ม 4 ช่วงอายุ | `EVALUATE TRUE` ใน `AGINGCLC` (Part 041) |
| 7. รายงาน Control Break | `ARREPORT` (เทคนิคจาก Part 026/039) |
| 8. ข้ามใบแจ้งหนี้ที่ชำระครบ | ตรวจสอบ `WS-OUTSTANDING > 0` ก่อนประมวลผล |

### แผนผังสถาปัตยกรรมสุดท้าย

```
ARSETUP  ──write──> CUSTOMER.DAT ◄──read/write── ARPAYMNT ◄──read── PAYMENTS.DAT
   │                     ▲                           │
   └──write──> INVOICE.DAT ◄──read/write─────────────┘
                     │                                │
                     │                          write │
                     │                                v
                     │                          PAYERR.LOG
                     │
              read (sequential by key)
                     │
                     v
                ARREPORT ──CALL──> AGINGCLC
                     │
                  DISPLAY
                     v
              Aging Report (stdout)
```

### สรุปเทคนิคทั้งหมดจาก Part 036-049 ที่ถูกนำมาใช้จริงในโปรเจกต์นี้

- **Part 028**: Indexed File / ISAM สำหรับ `CUSTOMER.DAT` และ `INVOICE.DAT` (พร้อม ISAM-enabled build)
- **Part 033**: Copybook (`CUSTREC.CPY`, `INVREC.CPY`) แชร์โครงสร้างข้อมูลระหว่าง 3 โปรแกรม
- **Part 031-032**: Subprogram `AGINGCLC` เรียกด้วย `CALL ... USING` แบบ positional parameter
- **Part 036/047**: `FUNCTION INTEGER-OF-DATE` คำนวณจำนวนวันระหว่างสองวันที่
- **Part 041**: `EVALUATE TRUE` จัดกลุ่มช่วงอายุหนี้ 4 ช่วงอย่างกระชับ
- **Part 026/039**: Control Break Processing แยกยอดรวมตามลูกค้าในรายงาน
- **Part 048**: `DECLARATIVES` + `USE AFTER STANDARD ERROR PROCEDURE` จัดการข้อผิดพลาดของไฟล์แบบ
  รวมศูนย์ในโปรแกรม `ARPAYMNT`
- **Part 006**: Edited PICTURE (`ZZZ,ZZ9.99`) จัดรูปแบบตัวเลขในรายงานให้อ่านง่าย
- **Part 010/030**: 88-level Condition Names (`INV-OPEN`, `INV-PAID-IN-FULL`, `INV-FOUND`)

### แนวทางการพัฒนาต่อยอด (สำหรับผู้ที่ต้องการฝึกฝนเพิ่มเติม)

ระบบนี้ถูกออกแบบให้อยู่ในขอบเขตที่เหมาะสมกับความรู้ ณ จุดนี้ของหลักสูตร แต่ระบบ AR ระดับองค์กรจริง
มักมีความสามารถเพิ่มเติมอีกมาก ที่ผู้เรียนสามารถลองพัฒนาต่อยอดได้ด้วยตนเองเพื่อฝึกฝน:

1. **แก้ปัญหาการเรียงลำดับข้อมูล** (ตามที่อธิบายในขั้นตอนที่ 497) โดยใช้ `SORT` (Part 027) เรียง
   ใบแจ้งหนี้ตาม `INV-CUST-ID` ก่อนสร้างรายงาน เพื่อให้ระบบทำงานถูกต้องแม้ข้อมูลจริงไม่ได้เรียงลำดับ
   ตามลูกค้า
2. **เพิ่มวันครบกำหนดชำระ (Due Date)** แยกจากวันที่ออกใบแจ้งหนี้ พร้อมเงื่อนไขเครดิตที่ต่างกันตาม
   ลูกค้าแต่ละราย (เช่น Net 15, Net 30, Net 60)
3. **ตรวจสอบวงเงินเครดิต (Credit Limit Check)**: ก่อนออกใบแจ้งหนี้ใหม่ ตรวจสอบว่า
   `CUST-BALANCE + INV-AMOUNT` เกิน `CUST-CREDIT-LIMIT` หรือไม่ (ใช้ Class/Sign Condition จาก
   Part 042 ได้)
4. **สร้างรายงานด้วย Report Writer แทน DISPLAY** ทบทวนเทคนิคจาก Part 038-039 เพื่อได้รายงานที่มี
   Page Header/Footer และการจัดหน้ากระดาษที่สมบูรณ์กว่า
5. **เพิ่ม Alternate Index** บน `INV-CUST-ID` (เทคนิคจะเรียนเต็มใน Part 056 VSAM ขั้นสูง) เพื่อค้นหา
   ใบแจ้งหนี้ทั้งหมดของลูกค้ารายหนึ่งได้โดยตรง โดยไม่ต้องอ่านทั้งไฟล์
6. **ส่งออกรายงานเป็น JSON** สำหรับเชื่อมต่อกับระบบ Dashboard สมัยใหม่ ด้วยเทคนิค `JSON GENERATE`
   จาก Part 045

### บทเรียนย้อนหลัง (Retrospective): สิ่งที่โปรเจกต์นี้สอนเกี่ยวกับการพัฒนาระบบจริง

ก่อนปิดท้าย Part นี้ ควรถอยกลับมามองภาพรวมของกระบวนการพัฒนาทั้งหมดตั้งแต่ขั้นตอนที่ 491 ถึง 499 อีก
ครั้ง เพราะบทเรียนที่ได้ไม่ได้มีแค่เรื่องไวยากรณ์ COBOL แต่รวมถึงวินัยการพัฒนาซอฟต์แวร์ที่ถ่ายทอดข้าม
ภาษาและแพลตฟอร์มได้ทั้งหมด:

- **การออกแบบก่อนเขียนโค้ดช่วยประหยัดเวลาในระยะยาว**: ขั้นตอนที่ 491 ใช้เวลาวิเคราะห์ Requirements
  และวาดแผนผังสถาปัตยกรรมก่อนพิมพ์โค้ดบรรทัดแรกแม้แต่บรรทัดเดียว — การลงทุนเวลานี้ทำให้ทั้ง 4 โมดูล
  ทำงานประสานกันได้อย่างราบรื่นตั้งแต่ครั้งแรกที่ทดสอบรวมกันในขั้นตอนที่ 499
- **ข้อจำกัดที่ค้นพบระหว่างทางมีค่าเท่ากับฟีเจอร์ที่ทำงานถูกต้อง**: ขั้นตอนที่ 497 ค้นพบข้อจำกัดสำคัญ
  เรื่องการเรียงลำดับข้อมูลที่ Control Break พึ่งพาอยู่ — การรู้จักและบันทึกข้อจำกัดนี้ไว้อย่างชัดเจน
  (แทนที่จะเพิกเฉยเพราะข้อมูลทดสอบบังเอิญใช้งานได้) มีค่าเท่ากับการเขียนโค้ดที่ทำงานถูกต้อง เพราะ
  ป้องกันไม่ให้ทีมในอนาคตนำระบบไปใช้ผิดวิธีโดยไม่รู้ตัว
- **การตรวจสอบไขว้ (Reconciliation) ควรเป็นนิสัย ไม่ใช่ขั้นตอนพิเศษ**: ขั้นตอนที่ 499 แสดงให้เห็นว่า
  การเปรียบเทียบตัวเลขจากคนละมุมมอง (`CUST-BALANCE` เทียบกับผลรวมรายงาน) ช่วยยืนยันความถูกต้องของ
  ระบบทั้งหมดได้อย่างมีประสิทธิภาพ แม้จะไม่ใช่การพิสูจน์ความถูกต้อง 100% ก็ตาม (ตามที่อธิบายไว้ใน
  แบบฝึกหัดที่ 499.1)
- **ทุกตัวเลขในเอกสารนี้มาจากการรันจริง ไม่ใช่การคำนวณลอย ๆ**: ตลอดทั้ง Part นี้ ทุกผลลัพธ์ที่แสดงผ่าน
  การคอมไพล์และรันจริงด้วย GnuCOBOL พร้อมการตรวจสอบด้วยมือประกอบทุกจุด (ขั้นตอนที่ 496, 498) — วินัย
  นี้เองที่ทำให้มั่นใจได้ว่าตัวอย่างในเอกสารนี้นำไปใช้อ้างอิงหรือต่อยอดได้จริง ไม่ใช่แค่โค้ดตัวอย่างที่
  "ดูเหมือนจะทำงานได้" แต่ไม่เคยถูกทดสอบจริง

### แบบฝึกหัดที่ 500.1

**โจทย์**: จงอธิบายว่าทำไมการแยกระบบออกเป็น 3 โปรแกรม + 1 Subprogram (แทนที่จะเขียนทุกอย่างรวมกัน
เป็นโปรแกรมเดียวขนาดใหญ่) จึงเป็นการออกแบบที่เหมาะสมกับระบบบัญชีลูกหนี้ในโลกจริง โดยเชื่อมโยงกับ
หลักการที่เรียนมาตลอดทั้งหลักสูตร

**เฉลยแนวทาง**: การแยกโปรแกรมตามหน้าที่ความรับผิดชอบ (Separation of Concerns ที่เรียนมาตั้งแต่ Part
014) สะท้อนความจริงในโลกธุรกิจว่า **แต่ละหน้าที่มักเกิดขึ้นในเวลาที่ต่างกันและมีความถี่ต่างกัน**:
`ARSETUP` รันครั้งเดียวตอนเริ่มระบบ, `ARPAYMNT` รันทุกครั้งที่มีการรับชำระเงินเข้ามา (อาจเป็นรายวัน),
และ `ARREPORT` รันเมื่อฝ่ายบริหารต้องการดูสถานะ (อาจเป็นรายสัปดาห์/รายเดือน) การแยกโปรแกรมทำให้แต่ละ
ส่วนถูกพัฒนา ทดสอบ และแก้ไขได้อย่างอิสระ (เช่น แก้กฎการคำนวณอายุหนี้ใน `AGINGCLC` โดยไม่กระทบ
`ARSETUP`/`ARPAYMNT` เลย) และยังสอดคล้องกับรูปแบบการทำงานจริงของระบบ Mainframe ที่แต่ละโปรแกรมมักถูก
เรียกทำงานเป็น "Job" แยกกันผ่าน Job Scheduler ตามตารางเวลาที่ต่างกัน (แนวคิด Batch Processing ที่
Part 066 จะสอนอย่างละเอียด) การเขียนทุกอย่างรวมเป็นโปรแกรมเดียวจะทำให้ยากต่อการดูแลรักษา ทดสอบ และ
scheduling ในระยะยาวมากกว่าการแยกเป็นโมดูลที่ชัดเจนตามที่ทำในโปรเจกต์นี้

---

## สรุปท้ายบท

โปรเจกต์รวบยอดเฟส 3 นี้พาคุณสร้าง **ระบบบัญชีลูกหนี้ (Accounts Receivable System)** ที่ทำงานได้จริง
สมบูรณ์ ครอบคลุม:

- การวิเคราะห์ความต้องการและออกแบบสถาปัตยกรรมระบบก่อนเขียนโค้ด (4 โมดูล: `ARSETUP`, `ARPAYMNT`,
  `AGINGCLC`, `ARREPORT`)
- การออกแบบ Copybook ร่วม (`CUSTREC.CPY`, `INVREC.CPY`) เพื่อความสอดคล้องของโครงสร้างข้อมูลทั้งระบบ
- โปรแกรม `ARSETUP` สร้างข้อมูลตั้งต้นด้วย Indexed File (ผ่าน ISAM-enabled GnuCOBOL build)
- บทเรียนสำคัญเรื่องสถานะไฟล์ที่คงอยู่ระหว่างการรันโปรแกรมหลายตัว (persistent shared state)
- โปรแกรม `ARPAYMNT` ที่ใช้ `DECLARATIVES` (เทคนิคเต็มรูปแบบจาก Part 048) จัดการข้อผิดพลาดของการ
  ชำระเงินที่อ้างอิงใบแจ้งหนี้ผิดพลาดแบบรวมศูนย์ ไม่ทำให้โปรแกรมล่ม
- Subprogram `AGINGCLC` แยกเดี่ยว ทดสอบได้อิสระ (Unit Test) ใช้ `FUNCTION INTEGER-OF-DATE` คำนวณ
  อายุหนี้และจัดกลุ่มด้วย `EVALUATE TRUE`
- โปรแกรม `ARREPORT` ที่ทำ Control Break Processing เต็มรูปแบบ พร้อมข้อจำกัดสำคัญเรื่องการเรียงลำดับ
  ข้อมูลที่ต้องระวังในโลกจริง
- การทดสอบครบวงจร (Integration Testing) พร้อมการตรวจสอบไขว้ (Reconciliation) เพื่อยืนยันความถูกต้อง
  ของระบบทั้งหมด
- แนวทางการพัฒนาต่อยอดสู่ระบบระดับองค์กรที่สมบูรณ์ยิ่งขึ้น

ด้วยโปรเจกต์นี้ เฟส 3 (ขั้นสูง, Parts 036-050) ของหลักสูตรได้จบลงอย่างสมบูรณ์แล้ว คุณได้พิสูจน์ตัวเอง
ว่าสามารถนำเทคนิคขั้นสูงของ COBOL มาประกอบร่างเป็นระบบธุรกิจจริงที่ใช้งานได้ — ทักษะเดียวกับที่โปรแกรม
เมอร์ COBOL มืออาชีพใช้ในการดูแลระบบการเงินขององค์กรทั่วโลกทุกวันนี้

ต่อไปนี้คือจุดเปลี่ยนสำคัญของหลักสูตร: **เฟส 4 — Mainframe/Enterprise (Parts 051-070)** จะพาคุณ
ออกจากโลกของ GnuCOBOL บนเครื่องส่วนตัว เข้าสู่โลกของ **Mainframe จริง**: z/OS, JCL, VSAM, DB2, และ
CICS — ระบบนิเวศที่ COBOL ระดับองค์กรขนาดใหญ่ทั่วโลกใช้งานอยู่จริงมาหลายทศวรรษ เริ่มต้นด้วย
**Part 051 — แนะนำ Mainframe และ z/OS Ecosystem**

**[ไปยัง Part 051: แนะนำ Mainframe และ z/OS Ecosystem →](part-051-mainframe-zos-intro.md)**

---

**เนื้อหาก่อนหน้า**: [Part 049: Compiler Directives และ COBOL Dialects ←](part-049-compiler-directives.md)
**เนื้อหาถัดไป**: [Part 051: แนะนำ Mainframe และ z/OS Ecosystem →](part-051-mainframe-zos-intro.md)
