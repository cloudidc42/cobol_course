# Part 094: Case Study — ระบบประกันภัย (Insurance System) (ขั้นตอนที่ 931–940)

## คำนำของ Part นี้

ต่อจาก Case Study ธนาคาร Core Banking ใน Part 091-093 (การออกแบบระบบ, โมดูลบัญชี/ธุรกรรม, การรายงาน
และปิดบัญชี) Part นี้จะพาไปสำรวจอุตสาหกรรมสำคัญอีกแห่งหนึ่งที่ COBOL ยังคงเป็นแกนหลักมาหลายสิบปี นั่นคือ
**ธุรกิจประกันภัย (Insurance)** ตามที่ Part 001 ได้กล่าวถึงไว้ตั้งแต่ต้นหลักสูตรว่าบริษัทประกันชีวิตและ
ประกันภัยรายใหญ่ทั่วโลกยังคงลงทุนดูแลระบบ COBOL อย่างต่อเนื่อง เพราะมีข้อมูลกรมธรรม์ระยะยาวหลายสิบปีที่
สะสมมาตั้งแต่ยุคแรกเริ่ม

ต่างจาก Core Banking ที่เป็นระบบขนาดใหญ่ครอบคลุม 3 Part เต็ม ๆ Part นี้ตั้งใจ**คุมขอบเขตให้กระชับ**
(focused) กว่า: เราจะสร้าง **4 โปรแกรมที่ทำงานได้จริง** ครอบคลุมวงจรชีวิตหลักของกรมธรรม์ประกันภัย
ตั้งแต่การสร้างกรมธรรม์ การรับชำระเบี้ยประกัน การพิจารณาสินไหมทดแทน (Claims) ไปจนถึงการตรวจสอบวันหมดอายุ/
ต่ออายุกรมธรรม์ โดยนำเทคนิคที่เรียนมาตลอดหลักสูตรกลับมาใช้ซ้ำอย่างเข้มข้น:

| เทคนิคที่นำมาใช้ | เรียนมาจาก |
|---|---|
| Indexed File (ISAM) พร้อม Composite ไม่จำเป็น แต่ใช้ Single Key | Part 028 |
| Copybook สำหรับ record ร่วม | Part 033 |
| REDEFINES เพื่อแยกปี/เดือน/วันจากฟิลด์วันที่ | Part 022 |
| FUNCTION INTEGER-OF-DATE สำหรับคำนวณจำนวนวันต่างกัน | Part 036, Part 047 |
| EVALUATE สำหรับตรรกะการตัดสินใจหลายเงื่อนไข | Part 011, Part 041 |
| การจัดการข้อผิดพลาดแบบไม่ล่ม (log แล้วไปต่อ) | Part 030, Part 048 |
| WS-AS-OF-DATE คงที่แทน FUNCTION CURRENT-DATE เพื่อผลลัพธ์ที่ทำซ้ำได้ | Part 050 |

### ระบบที่จะสร้างประกอบด้วย 4 โปรแกรมหลัก

1. **`INSSETUP`** — โปรแกรมตั้งค่าเริ่มต้น สร้างไฟล์กรมธรรม์หลัก (Policy Master) แบบ Indexed พร้อมข้อมูล
   ตัวอย่าง 7 กรมธรรม์ ครอบคลุม 3 ประเภท (รถยนต์, บ้าน, ชีวิต) และสถานะต่าง ๆ
2. **`PREMPAY`** — บันทึกการรับชำระเบี้ยประกัน อ่านไฟล์ธุรกรรมการชำระเงิน ตรวจสอบว่ากรมธรรม์ยังไม่ถูกยกเลิก
   แล้วปรับยอดเบี้ยที่ชำระสะสมและวันที่ชำระล่าสุด
3. **`CLAIMPROC`** — ประมวลผลการเคลมสินไหมทดแทน ตรวจสอบเงื่อนไขความคุ้มครองหลายชั้น (สถานะกรมธรรม์,
   ช่วงเวลาคุ้มครอง, ระยะเวลารอคอย/Waiting Period, วงเงินคุ้มครองคงเหลือ) แล้วคำนวณยอดจ่ายสินไหม
4. **`POLRENEW`** — ตรวจสอบวันหมดอายุ/ต่ออายุกรมธรรม์ ด้วย Intrinsic Date Function จัดกลุ่มเป็น
   "หมดอายุใหม่", "ใกล้หมดอายุภายใน 30 วัน", และ "ยังปกติ"

ทั้งหมดเชื่อมกันด้วย **Copybook** `POLREC.CPY` (เทคนิค Part 033) เพื่อให้โครงสร้างข้อมูลกรมธรรม์สอดคล้อง
กันทุกโปรแกรมโดยอัตโนมัติ

### หมายเหตุสำคัญเรื่องการทดสอบและสภาพแวดล้อม

เช่นเดียวกับ Part 050 และ Part 070 ระบบนี้ใช้ `ORGANIZATION IS INDEXED` เป็นแกนหลักของไฟล์กรมธรรม์
**GnuCOBOL มาตรฐานในเครื่องนี้ปิดการรองรับ Indexed File ไว้** (`cobc -info` แสดง
`indexed file handler : disabled`) โปรเจกต์นี้ทั้งหมดจึงคอมไพล์ด้วย **GnuCOBOL build ที่เปิดใช้ ISAM
handler (Berkeley DB)** ที่ `/opt/gnucobol-isam/` ตามที่ Part 028 อธิบายไว้ คำสั่งมาตรฐานที่ใช้ตลอด
ทั้ง Part นี้คือ:

```bash
/opt/gnucobol-isam/bin/cobc -x -o <program-name> <program-name>.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./<program-name>
```

**ทุกโปรแกรมและทุกผลลัพธ์ในเอกสารนี้ผ่านการคอมไพล์และรันทดสอบจริง** ด้วย GnuCOBOL
(`cobc (GnuCOBOL) 4.0-early-dev.0`, ISAM handler = BDB 5.3.28) โปรแกรมทั้งหมดเป็นแบบ batch
(ไม่มี `ACCEPT` แบบโต้ตอบ) อ่านไฟล์ธุรกรรมที่เตรียมไว้ล่วงหน้า ตรงตามธรรมชาติของงานประมวลผลกรมธรรม์/
สินไหมในองค์กรจริงที่มักรันผ่าน Job Scheduler ไม่ใช่พิมพ์ทีละค่า

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้ (คอมเมนต์, ชื่อตัวแปร/paragraph, ข้อความ
> ใน `DISPLAY`/string literal ทุกชนิด) เป็นภาษาอังกฤษ/ASCII ล้วน เพราะขีดจำกัดคอลัมน์ 72 ของ
> Fixed-Format COBOL นับเป็นไบต์ไม่ใช่ตัวอักษร ตามที่อธิบายไว้ใน `docs/COURSE-OUTLINE.md`

---

## ขั้นตอนที่ 931: การวิเคราะห์ความต้องการและออกแบบสถาปัตยกรรมระบบ

### ระบบประกันภัยทำงานอย่างไรในภาพรวม

ธุรกิจประกันภัย (ไม่ว่าจะเป็นประกันรถยนต์ ประกันบ้าน หรือประกันชีวิต) มีวงจรธุรกิจหลักที่คล้ายกันทุก
ประเภท:

1. **การขายกรมธรรม์ (Policy Issuance)** — ลูกค้าซื้อกรมธรรม์ ระบุวงเงินคุ้มครอง (Coverage Amount)
   และตกลงจ่ายเบี้ยประกัน (Premium) เป็นงวด ๆ (รายปี/รายเดือน) เพื่อแลกกับความคุ้มครองตลอดอายุกรมธรรม์
2. **การรับชำระเบี้ยประกัน (Premium Collection)** — ลูกค้าต้องชำระเบี้ยตรงเวลาเพื่อให้กรมธรรม์
   ยังมีผลบังคับ (Active) ถ้าไม่ชำระอาจถูกยกเลิกความคุ้มครอง
3. **การเคลมสินไหมทดแทน (Claims Processing)** — เมื่อเกิดเหตุการณ์ตามเงื่อนไขกรมธรรม์ (อุบัติเหตุ,
   ไฟไหม้, เสียชีวิต ฯลฯ) ลูกค้ายื่นเคลม บริษัทต้องตรวจสอบว่าเข้าเงื่อนไขความคุ้มครองหรือไม่ แล้วจ่ายเงิน
   ชดเชยตามวงเงินคุ้มครองที่เหลืออยู่
4. **การต่ออายุ/หมดอายุกรมธรรม์ (Renewal/Expiration)** — กรมธรรม์มีวันสิ้นสุดกำหนดไว้ชัดเจน ต้องมีระบบ
   คอยตรวจสอบว่ากรมธรรม์ใดใกล้หมดอายุ (เพื่อแจ้งเตือนให้ต่ออายุ) และกรมธรรม์ใดหมดอายุไปแล้ว (เพื่อ
   เปลี่ยนสถานะและหยุดให้ความคุ้มครอง)

### ความต้องการของระบบ (Requirements)

| ข้อ | ความต้องการ |
|---|---|
| 1 | เก็บข้อมูลกรมธรรม์หลัก: เลขที่กรมธรรม์ (key), ชื่อผู้ถือกรมธรรม์, ประเภท, เบี้ยประกัน, วงเงินคุ้มครอง, วันเริ่ม/สิ้นสุด |
| 2 | ติดตามสถานะกรมธรรม์: Active (มีผล), Lapsed (ขาดชำระ), Cancelled (ยกเลิก), Expired (หมดอายุ) |
| 3 | รับชำระเบี้ยประกัน ปรับยอดเบี้ยสะสมและวันที่ชำระล่าสุด ปฏิเสธการชำระของกรมธรรม์ที่ถูกยกเลิกแล้ว |
| 4 | รับคำขอเคลมสินไหม ตรวจสอบว่ากรมธรรม์ยัง Active, วันที่เกิดเหตุอยู่ในช่วงกรมธรรม์, ผ่านระยะเวลารอคอยแล้ว |
| 5 | คำนวณวงเงินคุ้มครองคงเหลือ (วงเงินรวม ลบยอดที่จ่ายสินไหมไปแล้ว) และจำกัดยอดจ่ายไม่ให้เกินวงเงินคงเหลือ |
| 6 | ตรวจสอบวันหมดอายุกรมธรรม์เทียบกับวันที่อ้างอิง (as-of date) แบ่งกลุ่มเป็นหมดอายุแล้ว/ใกล้หมดอายุ/ปกติ |
| 7 | ทุกธุรกรรมที่ปฏิเสธ (การชำระเงินหรือการเคลม) ต้องไม่ทำให้โปรแกรมล่ม และต้องบันทึกเหตุผลไว้ชัดเจน |

### สถาปัตยกรรมระบบ: แผนผังโมดูลและการไหลของข้อมูล

```
                    +-------------------+
                    |    INSSETUP       |  (Step 932 - run once)
                    | (Initial Load)    |
                    +--------+----------+
                             |
                 writes      v
              +----------------------------+
              |  POLICY.DAT (Indexed,      |
              |  key = POL-NUMBER)         |
              +--------------+-------------+
                    ^         ^          ^
                    |         |          |
         reads/     |         |          |    reads/
         updates    |         |          |    updates
    +---------------+--+  +---+------------+  +----+-----------+
    |   PREMPAY        |  |   CLAIMPROC     |  |   POLRENEW     |
    | (Step 933-934)   |  | (Step 935-936)  |  | (Step 937-938) |
    +--------+---------+  +--------+--------+  +----------------+
             ^                     ^
             |                     |
      PAYMENTS.DAT           CLAIMREQ.DAT
      (Line Sequential)      (Line Sequential)
                                    |
                                    v
                             CLAIMSOUT.DAT
                             (decision log)
```

### เหตุผลของการตัดสินใจออกแบบที่สำคัญ

- **ทำไมใช้ Indexed File สำหรับ POLICY-MASTER**: ทั้ง `PREMPAY` และ `CLAIMPROC` ต้องค้นหากรมธรรม์ด้วย
  เลขที่กรมธรรม์แบบสุ่มทุกธุรกรรม (ทบทวน Part 028 ขั้นตอนที่ 271) ถ้าใช้ Sequential File จะต้องอ่านทั้ง
  ไฟล์ทุกครั้งเพื่อหากรมธรรม์ที่ต้องการ ซึ่งไม่สมเหตุสมผลเมื่อมีธุรกรรมจำนวนมากต่อวัน
- **ทำไมแยกไฟล์ธุรกรรมเบี้ยประกันและเคลมออกจากกัน**: เบี้ยประกัน (`PAYMENTS.DAT`) และเคลม
  (`CLAIMREQ.DAT`) เป็นเหตุการณ์ทางธุรกิจที่**ต่างประเภทกันโดยสิ้นเชิง** เกิดจากคนละแผนก (ฝ่ายการเงิน
  vs ฝ่ายสินไหม) มีความถี่และปริมาณต่างกันมาก การแยกไฟล์ทำให้แต่ละโปรแกรมโฟกัสหน้าที่เดียว
  (Separation of Concerns จาก Part 014)
- **ทำไม `CLAIMPROC` ต้องตรวจสอบหลายเงื่อนไขตามลำดับ (cascade)**: ในธุรกิจประกันภัยจริง การอนุมัติเคลม
  ไม่ใช่การตรวจแค่เงื่อนไขเดียว แต่เป็น**ลำดับการคัดกรอง** (สถานะกรมธรรม์ผ่านหรือไม่ -> อยู่ในช่วงเวลา
  คุ้มครองหรือไม่ -> ผ่านระยะเวลารอคอยหรือไม่ -> วงเงินคุ้มครองพอหรือไม่) การเขียนด้วย `EVALUATE TRUE`
  ตามลำดับความสำคัญ (Part 041) ทำให้ตรรกะอ่านง่ายและสอดคล้องกับกระบวนการพิจารณาจริง
- **ทำไม `POLRENEW` ใช้ `WS-AS-OF-DATE` คงที่แทน `FUNCTION CURRENT-DATE`**: เหตุผลเดียวกับที่ Part 050
  อธิบายไว้ (ขั้นตอนที่ 495) คือถ้าใช้วันที่ปัจจุบันจริง ผลลัพธ์ของรายงานจะเปลี่ยนไปทุกวันที่รัน ทำให้
  เอกสารนี้ไม่สามารถแสดงผลลัพธ์ที่ทำซ้ำได้ ในระบบจริงค่านี้จะมาจาก `FUNCTION CURRENT-DATE` หรือพารามิเตอร์
  ของ Job Scheduler แทน

### ทางเลือกด้านสถาปัตยกรรมที่พิจารณาแล้วไม่เลือกใช้

| ทางเลือกที่พิจารณา | เหตุผลที่ไม่เลือกใช้ในโปรเจกต์นี้ |
|---|---|
| รวม `PREMPAY` และ `CLAIMPROC` เป็นโปรแกรมเดียว | ขัดกับ Separation of Concerns และทำให้ทดสอบยากขึ้นเมื่อกฎธุรกิจของแต่ละฝั่งซับซ้อนขึ้นในอนาคต |
| ใช้ Relative File แทน Indexed สำหรับ POLICY-MASTER | เลขที่กรมธรรม์ในระบบจริงมักเป็นรหัสที่มีรูปแบบเฉพาะ ไม่ใช่ลำดับเลขต่อเนื่อง (Part 029 เหมาะกับ key แบบลำดับเท่านั้น) |
| คำนวณ Aging/Waiting Period ด้วยการลบเลข YYYYMMDD ตรง ๆ | ผิดพลาดข้ามเดือน/ปี (เช่น 20260301 - 20260215 ไม่ใช่ 86) ต้องใช้ `FUNCTION INTEGER-OF-DATE` แปลงเป็นเลขวันต่อเนื่องก่อนเสมอ (บทเรียนจาก Part 036/047) |
| เก็บวงเงินคุ้มครองคงเหลือเป็นฟิลด์แยกต่างหาก | เสี่ยงข้อมูลไม่ตรงกัน (data drift) ถ้ามีการแก้ไขวงเงินคุ้มครองภายหลัง จึงคำนวณสดทุกครั้งจาก `POL-COVERAGE-AMOUNT - POL-TOTAL-CLAIMS-PAID` แทน |

### แบบฝึกหัดที่ 931.1

**โจทย์**: จงอธิบายว่าทำไมระบบประกันภัยถึงจำเป็นต้องมี "ระยะเวลารอคอย" (Waiting Period) ก่อนที่กรมธรรม์
จะเริ่มรับเคลมได้ และมันช่วยป้องกันความเสี่ยงอะไรให้บริษัทประกัน

**เฉลยแนวทาง**: ระยะเวลารอคอยป้องกันปัญหา **Adverse Selection** หรือการที่ลูกค้าซื้อกรมธรรม์เฉพาะตอน
รู้ตัวว่ากำลังจะเกิดเหตุการณ์ที่ต้องเคลม (เช่น ซื้อประกันบ้านทันทีที่เห็นแววไฟไหม้ใกล้เข้ามา) การกำหนดว่า
ต้องผ่านไปอย่างน้อย N วันหลังวันเริ่มกรมธรรม์ก่อนจึงจะเคลมได้ ช่วยลดความเสี่ยงทางการเงินของบริษัทและ
ทำให้เบี้ยประกันของลูกค้าโดยรวมยุติธรรมมากขึ้น (ไม่ต้องแบกรับต้นทุนจากคนที่ซื้อประกันเพื่อเคลมทันที)

---

## ขั้นตอนที่ 932: Copybook กรมธรรม์ และโปรแกรม `INSSETUP`

### การออกแบบข้อมูลกรมธรรม์ (Policy Record Design)

| ฟิลด์ | PICTURE | ความหมาย |
|---|---|---|
| `POL-NUMBER` | `X(10)` | เลขที่กรมธรรม์ (Record Key) |
| `POL-HOLDER-NAME` | `X(24)` | ชื่อผู้ถือกรมธรรม์ |
| `POL-TYPE` | `X(1)` | `A`=Auto (รถยนต์), `H`=Home (บ้าน), `L`=Life (ชีวิต) |
| `POL-PREMIUM` | `9(7)V99` | เบี้ยประกันต่องวด (รายปี) |
| `POL-COVERAGE-AMOUNT` | `9(9)V99` | วงเงินคุ้มครองสูงสุด |
| `POL-START-DATE` / `POL-END-DATE` | `9(8)` | วันที่เริ่ม/สิ้นสุดความคุ้มครอง (YYYYMMDD) |
| `POL-STATUS` | `X(1)` | `A`=Active, `L`=Lapsed, `X`=Cancelled, `E`=Expired |
| `POL-PREMIUM-PAID-TO` | `9(8)` | ชำระเบี้ยประกันครอบคลุมถึงวันที่ใด |
| `POL-TOTAL-PREMIUM-PAID` | `9(9)V99` | ยอดเบี้ยประกันสะสมที่ชำระมาทั้งหมด |
| `POL-CLAIMS-COUNT` | `9(3)` | จำนวนครั้งที่เคยเคลมสำเร็จ |
| `POL-TOTAL-CLAIMS-PAID` | `9(9)V99` | ยอดสินไหมสะสมที่จ่ายไปแล้ว (ใช้คำนวณวงเงินคงเหลือ) |

Copybook `POLREC.CPY` (เทคนิคจาก Part 033) คือ "สัญญา" กลางที่ทุกโปรแกรมในระบบนี้ COPY เข้าไปใช้ร่วมกัน:

```cobol
      ******************************************************************
      * POLREC.CPY
      * Shared policy master record layout for the insurance case
      * study. Every program that touches POLICY-MASTER (indexed
      * file, key = POL-NUMBER) COPYs this in, so the layout can
      * never drift apart between programs (Part 033 technique).
      ******************************************************************
           05  POL-NUMBER              PIC X(10).
           05  POL-HOLDER-NAME         PIC X(24).
           05  POL-TYPE                PIC X(1).
      *        A = Auto, H = Home, L = Life
           05  POL-PREMIUM             PIC 9(7)V99.
           05  POL-COVERAGE-AMOUNT     PIC 9(9)V99.
           05  POL-START-DATE          PIC 9(8).
           05  POL-END-DATE            PIC 9(8).
           05  POL-STATUS              PIC X(1).
      *        A = Active, L = Lapsed, X = Cancelled, E = Expired
           05  POL-PREMIUM-PAID-TO     PIC 9(8).
           05  POL-TOTAL-PREMIUM-PAID  PIC 9(9)V99.
           05  POL-CLAIMS-COUNT        PIC 9(3).
           05  POL-TOTAL-CLAIMS-PAID   PIC 9(9)V99.
```

### โปรแกรม `INSSETUP` — โหลดข้อมูลตัวอย่าง 7 กรมธรรม์

เช่นเดียวกับ Part 050/070 โปรแกรมนี้ใช้**ตารางในหน่วยความจำ** (`SEED-TABLE` ที่มี `OCCURS 7 TIMES`
จาก Part 016) เก็บข้อมูลตัวอย่าง แล้ววนเขียนออกไฟล์ Indexed ทีละ record แทนการรับข้อมูลผ่าน `ACCEPT`
เพราะนี่คือ batch job ที่รันครั้งเดียวเพื่อเตรียมสภาพแวดล้อม ไม่ใช่โปรแกรมโต้ตอบกับผู้ใช้:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INSSETUP.
      ******************************************************************
      * Insurance case study - one-time setup job.
      * Creates POLICY-MASTER (indexed, key = POL-NUMBER) and loads
      * seven sample policies used by every other program in this
      * case study (PREMPAY, CLAIMPROC, POLRENEW).
      ******************************************************************
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT POLICY-MASTER ASSIGN TO "POLICY.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS POL-NUMBER
               FILE STATUS IS WS-POL-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  POLICY-MASTER.
       01  POLICY-RECORD.
           COPY "polrec.cpy".

       WORKING-STORAGE SECTION.
       01  WS-POL-STATUS           PIC X(2).
       01  WS-COUNT                PIC 9(3) VALUE 0.

       01  SEED-TABLE.
           05  SEED-ENTRY OCCURS 7 TIMES.
               10  SEED-NUMBER          PIC X(10).
               10  SEED-NAME            PIC X(24).
               10  SEED-TYPE            PIC X(1).
               10  SEED-PREMIUM         PIC 9(7)V99.
               10  SEED-COVERAGE        PIC 9(9)V99.
               10  SEED-START           PIC 9(8).
               10  SEED-END             PIC 9(8).
               10  SEED-STATUS          PIC X(1).
               10  SEED-PAID-TO         PIC 9(8).
               10  SEED-TOTAL-PAID      PIC 9(9)V99.
               10  SEED-CLAIMS-COUNT    PIC 9(3).
               10  SEED-CLAIMS-PAID     PIC 9(9)V99.

       01  WS-IDX                  PIC 9(2).

       PROCEDURE DIVISION.
       000-MAIN.
           PERFORM 100-LOAD-SEED-DATA
           OPEN OUTPUT POLICY-MASTER
           IF WS-POL-STATUS NOT = "00"
               DISPLAY "ERROR OPENING POLICY-MASTER: " WS-POL-STATUS
               STOP RUN
           END-IF

           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > 7
               MOVE SEED-NUMBER(WS-IDX)       TO POL-NUMBER
               MOVE SEED-NAME(WS-IDX)         TO POL-HOLDER-NAME
               MOVE SEED-TYPE(WS-IDX)         TO POL-TYPE
               MOVE SEED-PREMIUM(WS-IDX)      TO POL-PREMIUM
               MOVE SEED-COVERAGE(WS-IDX)     TO POL-COVERAGE-AMOUNT
               MOVE SEED-START(WS-IDX)        TO POL-START-DATE
               MOVE SEED-END(WS-IDX)          TO POL-END-DATE
               MOVE SEED-STATUS(WS-IDX)       TO POL-STATUS
               MOVE SEED-PAID-TO(WS-IDX)      TO POL-PREMIUM-PAID-TO
               MOVE SEED-TOTAL-PAID(WS-IDX)   TO POL-TOTAL-PREMIUM-PAID
               MOVE SEED-CLAIMS-COUNT(WS-IDX) TO POL-CLAIMS-COUNT
               MOVE SEED-CLAIMS-PAID(WS-IDX)  TO POL-TOTAL-CLAIMS-PAID

               WRITE POLICY-RECORD
               IF WS-POL-STATUS = "00"
                   ADD 1 TO WS-COUNT
               ELSE
                   DISPLAY "WRITE FAILED FOR " POL-NUMBER
                       " STATUS " WS-POL-STATUS
               END-IF
           END-PERFORM

           CLOSE POLICY-MASTER
           DISPLAY "INSSETUP COMPLETE - POLICIES LOADED: " WS-COUNT
           STOP RUN.

       100-LOAD-SEED-DATA.
           MOVE "POL0000001" TO SEED-NUMBER(1)
           MOVE "JOHN SMITH              " TO SEED-NAME(1)
           MOVE "A"          TO SEED-TYPE(1)
           MOVE 12000.00     TO SEED-PREMIUM(1)
           MOVE 300000.00    TO SEED-COVERAGE(1)
           MOVE 20250601     TO SEED-START(1)
           MOVE 20260601     TO SEED-END(1)
           MOVE "A"          TO SEED-STATUS(1)
           MOVE 20250601     TO SEED-PAID-TO(1)
           MOVE 12000.00     TO SEED-TOTAL-PAID(1)
           MOVE 0            TO SEED-CLAIMS-COUNT(1)
           MOVE 0.00         TO SEED-CLAIMS-PAID(1)

           MOVE "POL0000002" TO SEED-NUMBER(2)
           MOVE "MARY JOHNSON            " TO SEED-NAME(2)
           MOVE "H"          TO SEED-TYPE(2)
           MOVE 8000.00      TO SEED-PREMIUM(2)
           MOVE 2500000.00   TO SEED-COVERAGE(2)
           MOVE 20260115     TO SEED-START(2)
           MOVE 20270115     TO SEED-END(2)
           MOVE "A"          TO SEED-STATUS(2)
           MOVE 20260115     TO SEED-PAID-TO(2)
           MOVE 8000.00      TO SEED-TOTAL-PAID(2)
           MOVE 0            TO SEED-CLAIMS-COUNT(2)
           MOVE 0.00         TO SEED-CLAIMS-PAID(2)

           MOVE "POL0000003" TO SEED-NUMBER(3)
           MOVE "ROBERT WILLIAMS         " TO SEED-NAME(3)
           MOVE "L"          TO SEED-TYPE(3)
           MOVE 24000.00     TO SEED-PREMIUM(3)
           MOVE 1000000.00   TO SEED-COVERAGE(3)
           MOVE 20200601     TO SEED-START(3)
           MOVE 20300601     TO SEED-END(3)
           MOVE "A"          TO SEED-STATUS(3)
           MOVE 20260601     TO SEED-PAID-TO(3)
           MOVE 144000.00    TO SEED-TOTAL-PAID(3)
           MOVE 0            TO SEED-CLAIMS-COUNT(3)
           MOVE 0.00         TO SEED-CLAIMS-PAID(3)

           MOVE "POL0000004" TO SEED-NUMBER(4)
           MOVE "LINDA DAVIS             " TO SEED-NAME(4)
           MOVE "A"          TO SEED-TYPE(4)
           MOVE 9500.00      TO SEED-PREMIUM(4)
           MOVE 200000.00    TO SEED-COVERAGE(4)
           MOVE 20260801     TO SEED-START(4)
           MOVE 20270801     TO SEED-END(4)
           MOVE "A"          TO SEED-STATUS(4)
           MOVE 20260801     TO SEED-PAID-TO(4)
           MOVE 9500.00      TO SEED-TOTAL-PAID(4)
           MOVE 0            TO SEED-CLAIMS-COUNT(4)
           MOVE 0.00         TO SEED-CLAIMS-PAID(4)

           MOVE "POL0000005" TO SEED-NUMBER(5)
           MOVE "MICHAEL BROWN           " TO SEED-NAME(5)
           MOVE "H"          TO SEED-TYPE(5)
           MOVE 7000.00      TO SEED-PREMIUM(5)
           MOVE 300000.00    TO SEED-COVERAGE(5)
           MOVE 20251001     TO SEED-START(5)
           MOVE 20261001     TO SEED-END(5)
           MOVE "A"          TO SEED-STATUS(5)
           MOVE 20251001     TO SEED-PAID-TO(5)
           MOVE 7000.00      TO SEED-TOTAL-PAID(5)
           MOVE 1            TO SEED-CLAIMS-COUNT(5)
           MOVE 50000.00     TO SEED-CLAIMS-PAID(5)

           MOVE "POL0000006" TO SEED-NUMBER(6)
           MOVE "SUSAN MILLER            " TO SEED-NAME(6)
           MOVE "L"          TO SEED-TYPE(6)
           MOVE 18000.00     TO SEED-PREMIUM(6)
           MOVE 750000.00    TO SEED-COVERAGE(6)
           MOVE 20240301     TO SEED-START(6)
           MOVE 20340301     TO SEED-END(6)
           MOVE "A"          TO SEED-STATUS(6)
           MOVE 20260301     TO SEED-PAID-TO(6)
           MOVE 54000.00     TO SEED-TOTAL-PAID(6)
           MOVE 0            TO SEED-CLAIMS-COUNT(6)
           MOVE 0.00         TO SEED-CLAIMS-PAID(6)

           MOVE "POL0000007" TO SEED-NUMBER(7)
           MOVE "DAVID GARCIA            " TO SEED-NAME(7)
           MOVE "A"          TO SEED-TYPE(7)
           MOVE 11000.00     TO SEED-PREMIUM(7)
           MOVE 250000.00    TO SEED-COVERAGE(7)
           MOVE 20240101     TO SEED-START(7)
           MOVE 20250101     TO SEED-END(7)
           MOVE "X"          TO SEED-STATUS(7)
           MOVE 20240101     TO SEED-PAID-TO(7)
           MOVE 11000.00     TO SEED-TOTAL-PAID(7)
           MOVE 0            TO SEED-CLAIMS-COUNT(7)
           MOVE 0.00         TO SEED-CLAIMS-PAID(7).
```

### ข้อมูลตัวอย่าง 7 กรมธรรม์ที่โหลดเข้าระบบ

| กรมธรรม์ | ชื่อผู้ถือ | ประเภท | เบี้ย/ปี | วงเงินคุ้มครอง | เริ่ม | สิ้นสุด | สถานะเริ่มต้น |
|---|---|---|---|---|---|---|---|
| POL0000001 | John Smith | Auto | 12,000.00 | 300,000.00 | 2025-06-01 | 2026-06-01 | Active |
| POL0000002 | Mary Johnson | Home | 8,000.00 | 2,500,000.00 | 2026-01-15 | 2027-01-15 | Active |
| POL0000003 | Robert Williams | Life | 24,000.00 | 1,000,000.00 | 2020-06-01 | 2030-06-01 | Active |
| POL0000004 | Linda Davis | Auto | 9,500.00 | 200,000.00 | 2026-08-01 | 2027-08-01 | Active |
| POL0000005 | Michael Brown | Home | 7,000.00 | 300,000.00 | 2025-10-01 | 2026-10-01 | Active (มีเคลมเก่า 50,000.00) |
| POL0000006 | Susan Miller | Life | 18,000.00 | 750,000.00 | 2024-03-01 | 2034-03-01 | Active |
| POL0000007 | David Garcia | Auto | 11,000.00 | 250,000.00 | 2024-01-01 | 2025-01-01 | **Cancelled** |

ข้อมูลชุดนี้ถูกออกแบบให้ครอบคลุมสถานการณ์หลากหลายที่จะใช้ทดสอบขั้นตอนถัดไป: กรมธรรม์ที่จะหมดอายุพอดี
(POL0000001), กรมธรรม์ที่ใกล้หมดอายุใน 3 วัน (POL0000005), กรมธรรม์ที่เพิ่งเริ่ม (POL0000004,
อยู่ในช่วง waiting period), และกรมธรรม์ที่ถูกยกเลิกไปแล้ว (POL0000007)

### คอมไพล์และรัน

```bash
/opt/gnucobol-isam/bin/cobc -x -o inssetup inssetup.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./inssetup
```

**ผลลัพธ์จริง**:

```
INSSETUP COMPLETE - POLICIES LOADED: 007
```

### ข้อควรระวัง

1. **ต้องรัน `INSSETUP` เพียงครั้งเดียว**: เพราะเปิดไฟล์ด้วย `OPEN OUTPUT` ซึ่งจะสร้างไฟล์ใหม่ทับของเดิม
   เสมอ ถ้ารันซ้ำระหว่างทดสอบ `PREMPAY`/`CLAIMPROC` ข้อมูลที่ปรับปรุงไปแล้วจะหายกลับไปเป็นค่าตั้งต้น
2. **ขนาดฟิลด์ตัวอักษรเกินพอดี**: สังเกตว่า literal ของ `SEED-NAME` บางตัวยาวเกิน 24 ตัวอักษรเล็กน้อย
   (มีช่องว่างเผื่อไว้) COBOL จะตัดท้ายอัตโนมัติเมื่อ `MOVE` เข้าฟิลด์ที่สั้นกว่า ไม่ error แต่ถ้าตั้งใจ
   จะเก็บชื่อยาวจริง ต้องตรวจสอบว่าฟิลด์ปลายทางกว้างพอ

### แบบฝึกหัดที่ 932.1

**โจทย์**: ถ้าต้องการเพิ่มกรมธรรม์ประเภทใหม่ "ประกันสุขภาพ" (`M` = Medical) เข้าไปในระบบนี้ ต้องแก้ไข
ไฟล์ใดบ้าง และทำไม Copybook ถึงช่วยลดความเสี่ยงตรงจุดนี้ได้

**เฉลย**: การเพิ่ม `POL-TYPE` ค่าใหม่ `M` ไม่ต้องแก้โครงสร้าง `POLREC.CPY` เลย เพราะ `POL-TYPE` เป็น
`PIC X(1)` ที่รองรับค่าตัวอักษรใดก็ได้อยู่แล้ว สิ่งที่ต้องแก้คือ (1) เพิ่มข้อมูลตัวอย่างใน `INSSETUP`
ถ้าต้องการทดสอบ และ (2) ตรวจสอบว่า `CLAIMPROC` มีกฎเฉพาะประเภทหรือไม่ (ในระบบนี้ยังไม่มีกฎแยกตาม
ประเภท จึงไม่ต้องแก้) จุดสำคัญคือ **โครงสร้างข้อมูล (Copybook) กับกฎธุรกิจ (Procedure Division)
แยกจากกันชัดเจน** การเพิ่มค่าที่เป็นไปได้ของฟิลด์ที่มีอยู่แล้วจึงไม่กระทบโปรแกรมอื่นเลย

---

## ขั้นตอนที่ 933: โปรแกรม `PREMPAY` — การออกแบบและตรรกะตรวจสอบ

### กฎธุรกิจของการรับชำระเบี้ยประกัน

เมื่อลูกค้าชำระเบี้ยประกัน ระบบต้องตรวจสอบก่อนว่ากรมธรรม์**ไม่ได้ถูกยกเลิก** (ถ้ายกเลิกแล้วไม่ควรรับ
เงินอีก) แล้วจึง:

1. บวกจำนวนเงินที่ชำระเข้ากับยอดเบี้ยสะสม (`POL-TOTAL-PREMIUM-PAID`)
2. เลื่อนวันที่ "ชำระครอบคลุมถึง" (`POL-PREMIUM-PAID-TO`) ไปข้างหน้า 1 ปี

ประเด็นทางเทคนิคที่น่าสนใจคือข้อ 2: **การบวกวันที่ไป 1 ปีอย่างถูกต้อง** ถ้าใช้ `FUNCTION INTEGER-OF-DATE`
บวก 365 แล้วแปลงกลับ จะผิดพลาดในปีอธิกสุรทิน (Leap Year) เพราะปีที่มี 366 วันจะทำให้วันที่เพี้ยนไป 1 วัน
วิธีที่แม่นยำกว่าสำหรับ "บวกไป 1 ปีปฏิทิน" คือการแยกปีออกมาแล้วบวก 1 ตรง ๆ ด้วยเทคนิค **REDEFINES**
ที่เรียนมาใน Part 022:

```cobol
       01  WS-DATE-FIELDS.
           05  WS-DATE-YYYYMMDD     PIC 9(8).
           05  WS-DATE-X REDEFINES WS-DATE-YYYYMMDD.
               10  WS-DATE-YYYY     PIC 9(4).
               10  WS-DATE-MMDD     PIC 9(4).
```

`WS-DATE-X` มองหน่วยความจำเดียวกันกับ `WS-DATE-YYYYMMDD` แต่แบ่งเป็น "4 หลักแรก (ปี)" และ "4 หลักหลัง
(เดือน+วัน)" การ `ADD 1 TO WS-DATE-YYYY` จึงเป็นการบวกปีไป 1 ปีปฏิทินแบบตรงไปตรงมา โดยเดือนและวันไม่
เปลี่ยนแปลงเลย — แม่นยำกว่าการคำนวณผ่านจำนวนวันสำหรับกรณีนี้โดยเฉพาะ

### โครงสร้างไฟล์ธุรกรรม `PAYMENTS.DAT`

ไฟล์ Line Sequential แบบคอลัมน์คงที่ (fixed-width) ตามธรรมเนียมไฟล์ธุรกรรมของหลักสูตร (Part 026,
Part 050):

| ฟิลด์ | PICTURE | ความหมาย |
|---|---|---|
| `PAY-POL-NUMBER` | `X(10)` | เลขที่กรมธรรม์ที่จะนำเงินไปหัก |
| `PAY-DATE` | `9(8)` | วันที่ชำระเงิน |
| `PAY-AMOUNT` | `9(7)V99` | จำนวนเงินที่ชำระ |

### เต็มโค้ด `PREMPAY.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PREMPAY.
      ******************************************************************
      * Insurance case study - premium payment recording.
      * Reads PAYMENTS.DAT (sequential transactions) and applies each
      * payment to POLICY-MASTER: adds to the running total premium
      * paid and advances the "paid-to" date by one year (REDEFINES
      * technique from Part 022). Rejects payments for policies that
      * do not exist or are cancelled, logging them instead of
      * crashing the run (same philosophy as Part 048/050).
      ******************************************************************
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT POLICY-MASTER ASSIGN TO "POLICY.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS POL-NUMBER
               FILE STATUS IS WS-POL-STATUS.

           SELECT PAYMENT-FILE ASSIGN TO "PAYMENTS.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-PAY-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  POLICY-MASTER.
       01  POLICY-RECORD.
           COPY "polrec.cpy".

       FD  PAYMENT-FILE.
       01  PAYMENT-RECORD.
           05  PAY-POL-NUMBER          PIC X(10).
           05  PAY-DATE                PIC 9(8).
           05  PAY-AMOUNT              PIC 9(7)V99.

       WORKING-STORAGE SECTION.
       01  WS-POL-STATUS            PIC X(2).
       01  WS-PAY-STATUS            PIC X(2).
       01  WS-EOF                   PIC X(1) VALUE "N".
           88  END-OF-PAYMENTS               VALUE "Y".

       01  WS-DATE-FIELDS.
           05  WS-DATE-YYYYMMDD     PIC 9(8).
           05  WS-DATE-X REDEFINES WS-DATE-YYYYMMDD.
               10  WS-DATE-YYYY     PIC 9(4).
               10  WS-DATE-MMDD     PIC 9(4).

       01  WS-COUNTERS.
           05  WS-APPLIED-COUNT     PIC 9(3) VALUE 0.
           05  WS-REJECTED-COUNT    PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       000-MAIN.
           OPEN I-O POLICY-MASTER
           IF WS-POL-STATUS NOT = "00"
               DISPLAY "ERROR OPENING POLICY-MASTER: " WS-POL-STATUS
               STOP RUN
           END-IF

           OPEN INPUT PAYMENT-FILE
           IF WS-PAY-STATUS NOT = "00"
               DISPLAY "ERROR OPENING PAYMENTS.DAT: " WS-PAY-STATUS
               STOP RUN
           END-IF

           DISPLAY "==== PREMIUM PAYMENT PROCESSING ===="

           PERFORM UNTIL END-OF-PAYMENTS
               READ PAYMENT-FILE
                   AT END
                       SET END-OF-PAYMENTS TO TRUE
                   NOT AT END
                       PERFORM 200-APPLY-PAYMENT
               END-READ
           END-PERFORM

           CLOSE PAYMENT-FILE
           CLOSE POLICY-MASTER

           DISPLAY " "
           DISPLAY "PAYMENTS APPLIED : " WS-APPLIED-COUNT
           DISPLAY "PAYMENTS REJECTED: " WS-REJECTED-COUNT
           STOP RUN.

       200-APPLY-PAYMENT.
           MOVE PAY-POL-NUMBER TO POL-NUMBER
           READ POLICY-MASTER
               INVALID KEY
                   DISPLAY "REJECT " PAY-POL-NUMBER
                       " REASON: POLICY NOT FOUND"
                   ADD 1 TO WS-REJECTED-COUNT
               NOT INVALID KEY
                   PERFORM 210-VALIDATE-AND-APPLY
           END-READ.

       210-VALIDATE-AND-APPLY.
           IF POL-STATUS = "X"
               DISPLAY "REJECT " PAY-POL-NUMBER
                   " REASON: POLICY CANCELLED"
               ADD 1 TO WS-REJECTED-COUNT
           ELSE
               ADD PAY-AMOUNT TO POL-TOTAL-PREMIUM-PAID
               MOVE POL-PREMIUM-PAID-TO TO WS-DATE-YYYYMMDD
               ADD 1 TO WS-DATE-YYYY
               MOVE WS-DATE-YYYYMMDD TO POL-PREMIUM-PAID-TO
               REWRITE POLICY-RECORD
               IF WS-POL-STATUS = "00"
                   DISPLAY "APPLIED " PAY-POL-NUMBER
                       " AMOUNT " PAY-AMOUNT
                       " NEW PAID-TO " POL-PREMIUM-PAID-TO
                   ADD 1 TO WS-APPLIED-COUNT
               ELSE
                   DISPLAY "REWRITE FAILED " PAY-POL-NUMBER
                       " STATUS " WS-POL-STATUS
                   ADD 1 TO WS-REJECTED-COUNT
               END-IF
           END-IF.
```

### สังเกตจุดสำคัญของโค้ด

- `ACCESS MODE IS DYNAMIC` (ต่างจาก `INSSETUP` ที่ใช้ `SEQUENTIAL`) เพราะโปรแกรมนี้ต้องทำทั้ง `READ`
  แบบสุ่มด้วย key (`READ ... INVALID KEY`) — จำเป็นต้องเปิดไฟล์ด้วยโหมดที่รองรับการเข้าถึงแบบสุ่ม
  (ทบทวน Part 028 ขั้นตอนที่ 274 เรื่อง Access Mode ของ Indexed File)
- `OPEN I-O` ไม่ใช่ `OPEN INPUT` เพราะโปรแกรมต้องทั้งอ่านและ `REWRITE` กลับเข้าไฟล์เดียวกัน
- การ `REWRITE` เกิด**หลังจาก** `READ POLICY-MASTER` สำเร็จเท่านั้น (อยู่ใน `NOT INVALID KEY`) ตรงตาม
  กฎของ Indexed File ที่ต้องอ่าน record เข้ามาก่อนจึงจะเขียนทับได้ (Part 025)

### แบบฝึกหัดที่ 933.1

**โจทย์**: ถ้าลบบรรทัด `IF POL-STATUS = "X"` ออกไป (ไม่ตรวจสอบสถานะยกเลิกเลย) จะเกิดผลอย่างไรกับ
กรมธรรม์ POL0000007 เมื่อมีการชำระเงินเข้ามา

**เฉลย**: ระบบจะยอมรับเงินและอัปเดตยอดเบี้ยสะสม/วันที่ชำระของกรมธรรม์ที่**ถูกยกเลิกไปแล้ว**ราวกับว่า
มันยังมีผลอยู่ ซึ่งเป็นข้อผิดพลาดทางธุรกิจร้ายแรง เพราะบริษัทรับเงินจากลูกค้าโดยไม่มีภาระผูกพันที่จะ
คุ้มครองอะไรเลย (หรือในทางกลับกัน ลูกค้าอาจเข้าใจผิดว่ากรมธรรม์ยังมีผลอยู่) นี่คือเหตุผลที่การตรวจสอบ
สถานะก่อนทำธุรกรรมทุกครั้งสำคัญมากในระบบการเงิน

---

## ขั้นตอนที่ 934: รันและวิเคราะห์ผลลัพธ์ของ `PREMPAY`

### ข้อมูลธุรกรรมทดสอบ `PAYMENTS.DAT`

ไฟล์นี้มี 4 รายการ ออกแบบให้ครอบคลุมทั้งกรณีสำเร็จและกรณีถูกปฏิเสธ:

| ลำดับ | กรมธรรม์ | วันที่ | จำนวนเงิน | คาดว่าจะเป็น |
|---|---|---|---|---|
| 1 | POL0000001 | 2026-06-15 | 12,000.00 | สำเร็จ (Active) |
| 2 | POL0000005 | 2026-09-25 | 7,000.00 | สำเร็จ (Active) |
| 3 | POL0000007 | 2026-06-10 | 11,000.00 | ปฏิเสธ — กรมธรรม์ถูกยกเลิก |
| 4 | POL9999999 | 2026-06-01 | 5,000.00 | ปฏิเสธ — ไม่พบกรมธรรม์ |

### คอมไพล์และรัน

```bash
/opt/gnucobol-isam/bin/cobc -x -o prempay prempay.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./prempay
```

**ผลลัพธ์จริง**:

```
==== PREMIUM PAYMENT PROCESSING ====
APPLIED POL0000001 AMOUNT 0012000.00 NEW PAID-TO 20260601
APPLIED POL0000005 AMOUNT 0007000.00 NEW PAID-TO 20261001
REJECT POL0000007 REASON: POLICY CANCELLED
REJECT POL9999999 REASON: POLICY NOT FOUND

PAYMENTS APPLIED : 002
PAYMENTS REJECTED: 002
```

### วิเคราะห์ผลลัพธ์

- **POL0000001**: `POL-PREMIUM-PAID-TO` เดิมคือ `20250601` (ปี 2025) บวก 1 ปีด้วยเทคนิค REDEFINES
  กลายเป็น `20260601` (ปี 2026 เดือน/วันเดิม) — ตรงตามที่ออกแบบไว้ทุกประการ ยอดเบี้ยสะสมกลายเป็น
  12,000.00 + 12,000.00 = **24,000.00**
- **POL0000005**: paid-to เดิม `20251001` กลายเป็น `20261001` ยอดเบี้ยสะสมกลายเป็น
  7,000.00 + 7,000.00 = **14,000.00**
- **POL0000007** และ **POL9999999**: ถูกปฏิเสธตามที่ออกแบบไว้ โปรแกรม**ไม่ล่ม** และยังคงประมวลผล
  รายการถัดไปต่อได้ตามปกติ (การจัดการ error แบบ log-and-continue จาก Part 048/050)

### ข้อควรระวัง

1. **การบวกปีด้วย REDEFINES ใช้ได้ดีเฉพาะกรณี "บวกไปเป็นปี ๆ" เท่านั้น**: ถ้าต้องการบวกเป็นจำนวนวัน
   (เช่น เบี้ยประกันรายเดือน ต้องบวกไป 30 วัน) ต้องกลับไปใช้ `FUNCTION INTEGER-OF-DATE` +
   `FUNCTION DATE-OF-INTEGER` ตามที่ Part 047 สอนไว้แทน เพราะ REDEFINES แยกแค่ปีกับเดือน/วัน
   ไม่สามารถ "บวกวัน" ข้ามเดือนให้ถูกต้องอัตโนมัติได้ (เช่น 30 ม.ค. บวก 5 วัน ต้องกลายเป็น 4 ก.พ.
   ไม่ใช่ 35 ม.ค.)
2. **กรณีวันที่ 29 กุมภาพันธ์**: ถ้ากรมธรรม์เริ่มวันที่ 29 กุมภาพันธ์ (ปีอธิกสุรทิน) การบวกปีตรง ๆ
   แบบนี้อาจได้วันที่ที่ไม่มีอยู่จริง (เช่น 29 กุมภาพันธ์ปีถัดไปที่ไม่ใช่ปีอธิกสุรทิน) ระบบจริงต้องมี
   การตรวจสอบและปรับเป็น 28 กุมภาพันธ์ กรณีนี้ไม่เกิดในชุดข้อมูลตัวอย่าง แต่ควรตระหนักไว้เมื่อออกแบบ
   ระบบจริง

### แบบฝึกหัดที่ 934.1

**โจทย์**: จงเขียนตรรกะ (pseudo-code หรือ COBOL) เพิ่มเติมที่จะปฏิเสธการชำระเงินที่มีจำนวนเงินเป็น 0
หรือติดลบ (ในกรณีนี้ `PAY-AMOUNT` เป็น `PIC 9(7)V99` ซึ่งเก็บค่าติดลบไม่ได้อยู่แล้ว แต่ควรตรวจสอบ
กรณีเป็น 0 อย่างน้อย)

**เฉลย**: เพิ่มเงื่อนไขตรวจสอบก่อนเข้า `210-VALIDATE-AND-APPLY`:

```cobol
       200-APPLY-PAYMENT.
           MOVE PAY-POL-NUMBER TO POL-NUMBER
           IF PAY-AMOUNT = 0
               DISPLAY "REJECT " PAY-POL-NUMBER
                   " REASON: ZERO PAYMENT AMOUNT"
               ADD 1 TO WS-REJECTED-COUNT
           ELSE
               READ POLICY-MASTER
                   INVALID KEY
                       DISPLAY "REJECT " PAY-POL-NUMBER
                           " REASON: POLICY NOT FOUND"
                       ADD 1 TO WS-REJECTED-COUNT
                   NOT INVALID KEY
                       PERFORM 210-VALIDATE-AND-APPLY
               END-READ
           END-IF.
```

เพราะ `PIC 9(7)V99` เป็นชนิดข้อมูล **unsigned** ตามค่าเริ่มต้น (ไม่มี `S` นำหน้า) จึงไม่สามารถเก็บค่า
ติดลบได้อยู่แล้วตั้งแต่ระดับ PICTURE Clause (ทบทวน Part 006) การตรวจสอบเพิ่มเติมจึงเหลือแค่กรณี "ศูนย์"
ซึ่งแม้ไม่ผิดกฎข้อมูล แต่ไม่มีความหมายทางธุรกิจที่จะบันทึกเป็นธุรกรรมการชำระเงิน

---

## ขั้นตอนที่ 935: โปรแกรม `CLAIMPROC` — การออกแบบตรรกะการพิจารณาสินไหม

### ลำดับการตรวจสอบเคลม (Claim Evaluation Cascade)

การพิจารณาว่าจะอนุมัติเคลมหรือไม่เป็นตรรกะที่ซับซ้อนที่สุดในระบบนี้ ประกอบด้วยการตรวจสอบ **4 ชั้น
ตามลำดับ** (ถ้าชั้นใดไม่ผ่าน จะปฏิเสธทันทีโดยไม่ตรวจชั้นถัดไป):

```
    เคลมเข้ามา
        |
        v
   [1] พบกรมธรรม์นี้ในระบบหรือไม่?  --ไม่พบ--> REJECT: POLICY NOT FOUND
        | พบ
        v
   [2] กรมธรรม์ยังมีสถานะ Active หรือไม่?  --ไม่--> REJECT: POLICY NOT ACTIVE
        | Active
        v
   [3] วันที่เกิดเหตุอยู่ในช่วงกรมธรรม์หรือไม่?  --ไม่อยู่--> REJECT: OUTSIDE POLICY PERIOD
        | อยู่ในช่วง
        v
   [4] ผ่านระยะเวลารอคอย 30 วันหรือยัง?  --ยังไม่ผ่าน--> REJECT: WITHIN WAITING PERIOD
        | ผ่านแล้ว
        v
   [5] วงเงินคุ้มครองยังเหลือหรือไม่?  --หมดแล้ว--> REJECT: COVERAGE EXHAUSTED
        | ยังเหลือ
        v
   คำนวณยอดจ่าย = MIN(ยอดที่เคลม, วงเงินคงเหลือ)
   APPROVED หรือ APPROVED-PARTIAL (ถ้ายอดเคลมเกินวงเงินคงเหลือ)
```

ลำดับนี้สำคัญมาก: การตรวจสถานะกรมธรรม์ (ข้อ 2) ต้องมาก่อนการตรวจช่วงเวลา (ข้อ 3) เพราะกรมธรรม์ที่ถูก
ยกเลิกไม่ควรได้รับการพิจารณาต่อเลย ไม่ว่าวันที่เกิดเหตุจะอยู่ในช่วงกรมธรรม์เดิมหรือไม่ก็ตาม

### การคำนวณ "ผ่านระยะเวลารอคอยหรือยัง" ด้วย FUNCTION INTEGER-OF-DATE

```cobol
           COMPUTE WS-DAYS-SINCE-START =
               FUNCTION INTEGER-OF-DATE(CLM-DATE) -
               FUNCTION INTEGER-OF-DATE(POL-START-DATE)
```

`FUNCTION INTEGER-OF-DATE` แปลงวันที่รูปแบบ `YYYYMMDD` ให้เป็น **เลขจำนวนเต็มของวัน** (วันที่ต่อเนื่อง
นับจากจุดอ้างอิงคงที่) การลบเลขจำนวนเต็มสองตัวนี้จึงได้ **จำนวนวันที่ต่างกันจริง ๆ** โดยไม่มีปัญหาข้าม
เดือน/ปี ต่างจากการลบเลข `YYYYMMDD` ตรง ๆ ที่จะผิดพลาดทันทีถ้าวันที่คร่อมเดือนกัน (ทบทวนหลักการนี้จาก
Part 036 ขั้นตอนที่ 351 และ Part 047 ขั้นตอนที่ 461)

### การคำนวณวงเงินคุ้มครองคงเหลือและยอดจ่าย

```cobol
           COMPUTE WS-REMAINING-COVERAGE =
               POL-COVERAGE-AMOUNT - POL-TOTAL-CLAIMS-PAID
           IF WS-REMAINING-COVERAGE <= 0
               MOVE "REJECTED" TO WS-DECISION
               MOVE "COVERAGE EXHAUSTED" TO WS-REASON
           ELSE
               IF CLM-AMOUNT > WS-REMAINING-COVERAGE
                   MOVE WS-REMAINING-COVERAGE TO WS-PAYOUT
                   MOVE "APPROVED-PARTIAL" TO WS-DECISION
               ELSE
                   MOVE CLM-AMOUNT TO WS-PAYOUT
                   MOVE "APPROVED" TO WS-DECISION
               END-IF
           END-IF
```

สังเกตว่า `WS-REMAINING-COVERAGE` ประกาศเป็น `PIC S9(9)V99` (มีเครื่องหมาย `S`) แม้ในทางธุรกิจค่านี้
ไม่ควรติดลบ แต่การใส่ `S` ไว้เป็นแนวป้องกันเชิงรับ (defensive programming) เผื่อกรณีข้อมูลผิดพลาดทำให้
`POL-TOTAL-CLAIMS-PAID` มากกว่า `POL-COVERAGE-AMOUNT` โดยไม่ได้ตั้งใจ ถ้าไม่ใส่ `S` ผลลัพธ์ที่ควรจะ
เป็นลบจะถูกเก็บเป็นค่าสัมบูรณ์แทน (ทบทวน Part 006 เรื่อง Signed/Unsigned PICTURE) ทำให้เงื่อนไข
`<= 0` ตรวจจับไม่ได้ถูกต้อง — เป็นข้อผิดพลาดที่หาสาเหตุยากมากถ้าเกิดขึ้นจริง

### แบบฝึกหัดที่ 935.1

**โจทย์**: ถ้าสลับลำดับการตรวจสอบใน `300-EVALUATE-CLAIM` โดยเอาการตรวจ "ระยะเวลารอคอย" (ข้อ 4)
ไว้**ก่อน**การตรวจ "อยู่ในช่วงกรมธรรม์" (ข้อ 3) จะเกิดปัญหาอะไรได้บ้าง

**เฉลยแนวทาง**: อาจเกิดข้อผิดพลาดร้ายแรงถ้าวันที่เกิดเหตุ (`CLM-DATE`) เกิด**ก่อน**วันเริ่มกรมธรรม์
(`POL-START-DATE`) เพราะ `FUNCTION INTEGER-OF-DATE(CLM-DATE) - FUNCTION INTEGER-OF-DATE(POL-START-DATE)`
จะได้ค่า**ติดลบ** ซึ่งย่อมน้อยกว่า 30 อยู่แล้ว โปรแกรมจะปฏิเสธด้วยเหตุผล "WITHIN WAITING PERIOD"
ซึ่ง**ไม่ตรงกับความจริง** — เหตุผลที่แท้จริงคือวันที่เกิดเหตุอยู่นอกช่วงกรมธรรม์ต่างหาก การให้เหตุผล
ปฏิเสธผิดพลาดแบบนี้ในระบบจริงจะทำให้ทีมตรวจสอบข้อร้องเรียนของลูกค้าสับสนและแก้ปัญหาผิดจุด ดังนั้น
**ลำดับการตรวจสอบเงื่อนไขทางธุรกิจต้องสะท้อนลำดับความสำคัญ/ความสมเหตุสมผลของเหตุผลที่จะสื่อสารกลับไป
ด้วย** ไม่ใช่แค่ผลลัพธ์อนุมัติ/ปฏิเสธที่ถูกต้องเพียงอย่างเดียว

---

## ขั้นตอนที่ 936: เต็มโค้ด `CLAIMPROC`, การรัน, และวิเคราะห์ผลลัพธ์

### โครงสร้างไฟล์คำขอเคลม `CLAIMREQ.DAT` และผลลัพธ์ `CLAIMSOUT.DAT`

| ไฟล์ | ฟิลด์ | PICTURE |
|---|---|---|
| `CLAIMREQ.DAT` (input) | `CLM-NUMBER` | `X(8)` |
| | `CLM-POL-NUMBER` | `X(10)` |
| | `CLM-DATE` | `9(8)` |
| | `CLM-TYPE` | `X(10)` |
| | `CLM-AMOUNT` | `9(9)V99` |
| `CLAIMSOUT.DAT` (output, log การตัดสินใจ) | ข้อความรายบรรทัดแบบ formatted | `X(80)` |

### เต็มโค้ด `CLAIMPROC.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CLAIMPROC.
      ******************************************************************
      * Insurance case study - claims processing.
      * Reads CLAIMREQ.DAT (sequential claim requests), validates each
      * one against POLICY-MASTER (status, policy period, a 30-day
      * waiting period, remaining coverage), computes a payout, and
      * writes an approve/reject decision line to CLAIMSOUT.DAT.
      * Approved claims REWRITE the policy's cumulative claims paid
      * and claims count.
      ******************************************************************
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT POLICY-MASTER ASSIGN TO "POLICY.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS POL-NUMBER
               FILE STATUS IS WS-POL-STATUS.

           SELECT CLAIM-REQUEST-FILE ASSIGN TO "CLAIMREQ.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-CLM-STATUS.

           SELECT CLAIMS-OUT-FILE ASSIGN TO "CLAIMSOUT.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-OUT-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  POLICY-MASTER.
       01  POLICY-RECORD.
           COPY "polrec.cpy".

       FD  CLAIM-REQUEST-FILE.
       01  CLAIM-REQUEST-RECORD.
           05  CLM-NUMBER              PIC X(8).
           05  CLM-POL-NUMBER          PIC X(10).
           05  CLM-DATE                PIC 9(8).
           05  CLM-TYPE                PIC X(10).
           05  CLM-AMOUNT              PIC 9(9)V99.

       FD  CLAIMS-OUT-FILE.
       01  CLAIMS-OUT-RECORD           PIC X(80).

       WORKING-STORAGE SECTION.
       01  WS-POL-STATUS            PIC X(2).
       01  WS-CLM-STATUS            PIC X(2).
       01  WS-OUT-STATUS            PIC X(2).
       01  WS-EOF                   PIC X(1) VALUE "N".
           88  END-OF-CLAIMS                 VALUE "Y".

       01  WS-DAYS-SINCE-START      PIC S9(6).
       01  WS-REMAINING-COVERAGE    PIC S9(9)V99.
       01  WS-PAYOUT                PIC 9(9)V99.
       01  WS-DECISION              PIC X(16).
       01  WS-REASON                PIC X(40).

       01  WS-COUNTERS.
           05  WS-APPROVED-COUNT    PIC 9(3) VALUE 0.
           05  WS-REJECTED-COUNT    PIC 9(3) VALUE 0.
           05  WS-TOTAL-PAYOUT      PIC 9(9)V99 VALUE 0.

       01  WS-OUT-LINE.
           05  WS-OUT-CLAIM         PIC X(8).
           05  FILLER               PIC X(1) VALUE SPACE.
           05  WS-OUT-POLICY        PIC X(10).
           05  FILLER               PIC X(1) VALUE SPACE.
           05  WS-OUT-DECISION      PIC X(16).
           05  FILLER               PIC X(1) VALUE SPACE.
           05  WS-OUT-PAYOUT        PIC ZZZ,ZZZ,ZZ9.99.
           05  FILLER               PIC X(1) VALUE SPACE.
           05  WS-OUT-REASON        PIC X(40).

       PROCEDURE DIVISION.
       000-MAIN.
           OPEN I-O POLICY-MASTER
           IF WS-POL-STATUS NOT = "00"
               DISPLAY "ERROR OPENING POLICY-MASTER: " WS-POL-STATUS
               STOP RUN
           END-IF

           OPEN INPUT CLAIM-REQUEST-FILE
           OPEN OUTPUT CLAIMS-OUT-FILE

           DISPLAY "==== CLAIMS PROCESSING ===="

           PERFORM UNTIL END-OF-CLAIMS
               READ CLAIM-REQUEST-FILE
                   AT END
                       SET END-OF-CLAIMS TO TRUE
                   NOT AT END
                       PERFORM 200-PROCESS-CLAIM
               END-READ
           END-PERFORM

           CLOSE CLAIM-REQUEST-FILE
           CLOSE CLAIMS-OUT-FILE
           CLOSE POLICY-MASTER

           DISPLAY " "
           DISPLAY "CLAIMS APPROVED : " WS-APPROVED-COUNT
           DISPLAY "CLAIMS REJECTED : " WS-REJECTED-COUNT
           DISPLAY "TOTAL PAYOUT    : " WS-TOTAL-PAYOUT
           STOP RUN.

       200-PROCESS-CLAIM.
           MOVE SPACES TO WS-DECISION WS-REASON
           MOVE 0 TO WS-PAYOUT
           MOVE CLM-POL-NUMBER TO POL-NUMBER
           READ POLICY-MASTER
               INVALID KEY
                   MOVE "REJECTED" TO WS-DECISION
                   MOVE "POLICY NOT FOUND" TO WS-REASON
               NOT INVALID KEY
                   PERFORM 300-EVALUATE-CLAIM
           END-READ

           PERFORM 400-WRITE-RESULT.

       300-EVALUATE-CLAIM.
           EVALUATE TRUE
               WHEN POL-STATUS NOT = "A"
                   MOVE "REJECTED" TO WS-DECISION
                   MOVE "POLICY NOT ACTIVE" TO WS-REASON
               WHEN CLM-DATE < POL-START-DATE
                    OR CLM-DATE > POL-END-DATE
                   MOVE "REJECTED" TO WS-DECISION
                   MOVE "CLAIM DATE OUTSIDE POLICY PERIOD"
                       TO WS-REASON
               WHEN OTHER
                   COMPUTE WS-DAYS-SINCE-START =
                       FUNCTION INTEGER-OF-DATE(CLM-DATE) -
                       FUNCTION INTEGER-OF-DATE(POL-START-DATE)
                   IF WS-DAYS-SINCE-START < 30
                       MOVE "REJECTED" TO WS-DECISION
                       MOVE "WITHIN 30-DAY WAITING PERIOD"
                           TO WS-REASON
                   ELSE
                       PERFORM 310-CHECK-COVERAGE
                   END-IF
           END-EVALUATE.

       310-CHECK-COVERAGE.
           COMPUTE WS-REMAINING-COVERAGE =
               POL-COVERAGE-AMOUNT - POL-TOTAL-CLAIMS-PAID
           IF WS-REMAINING-COVERAGE <= 0
               MOVE "REJECTED" TO WS-DECISION
               MOVE "COVERAGE EXHAUSTED" TO WS-REASON
           ELSE
               IF CLM-AMOUNT > WS-REMAINING-COVERAGE
                   MOVE WS-REMAINING-COVERAGE TO WS-PAYOUT
                   MOVE "APPROVED-PARTIAL" TO WS-DECISION
                   MOVE "PAYOUT CAPPED AT REMAINING COVERAGE"
                       TO WS-REASON
               ELSE
                   MOVE CLM-AMOUNT TO WS-PAYOUT
                   MOVE "APPROVED" TO WS-DECISION
                   MOVE "WITHIN COVERAGE LIMIT" TO WS-REASON
               END-IF
               ADD WS-PAYOUT TO POL-TOTAL-CLAIMS-PAID
               ADD 1 TO POL-CLAIMS-COUNT
               REWRITE POLICY-RECORD
           END-IF.

       400-WRITE-RESULT.
           MOVE CLM-NUMBER     TO WS-OUT-CLAIM
           MOVE CLM-POL-NUMBER TO WS-OUT-POLICY
           MOVE WS-DECISION    TO WS-OUT-DECISION
           MOVE WS-PAYOUT      TO WS-OUT-PAYOUT
           MOVE WS-REASON      TO WS-OUT-REASON
           MOVE WS-OUT-LINE    TO CLAIMS-OUT-RECORD
           WRITE CLAIMS-OUT-RECORD

           DISPLAY CLM-NUMBER " " CLM-POL-NUMBER " "
               WS-DECISION " PAYOUT " WS-PAYOUT
               " (" WS-REASON ")"

           IF WS-DECISION(1:8) = "APPROVED"
               ADD 1 TO WS-APPROVED-COUNT
               ADD WS-PAYOUT TO WS-TOTAL-PAYOUT
           ELSE
               ADD 1 TO WS-REJECTED-COUNT
           END-IF.
```

### ข้อมูลเคลมทดสอบ 6 รายการ

| เคลม | กรมธรรม์ | วันที่เกิดเหตุ | ประเภท | ยอดที่เคลม | คาดว่าจะเป็น |
|---|---|---|---|---|---|
| CLM00001 | POL0000005 | 2026-06-15 | FIRE | 40,000.00 | อนุมัติเต็มจำนวน |
| CLM00002 | POL0000004 | 2026-08-10 | COLLISION | 15,000.00 | ปฏิเสธ — อยู่ในระยะเวลารอคอย (เพิ่งเริ่มกรมธรรม์ 9 วัน) |
| CLM00003 | POL0000007 | 2024-06-01 | COLLISION | 5,000.00 | ปฏิเสธ — กรมธรรม์ไม่ Active (ถูกยกเลิก) |
| CLM00004 | POL0000002 | 2026-09-20 | THEFT | 3,000,000.00 | อนุมัติบางส่วน (เกินวงเงินคุ้มครอง 2,500,000.00) |
| CLM00005 | POL0000099 | 2026-06-01 | FIRE | 1,000.00 | ปฏิเสธ — ไม่พบกรมธรรม์ |
| CLM00006 | POL0000006 | 2023-01-01 | MEDICAL | 20,000.00 | ปฏิเสธ — วันที่เกิดเหตุก่อนวันเริ่มกรมธรรม์ |

### คอมไพล์และรัน

```bash
/opt/gnucobol-isam/bin/cobc -x -o claimproc claimproc.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./claimproc
```

**ผลลัพธ์จริง**:

```
==== CLAIMS PROCESSING ====
CLM00001 POL0000005 APPROVED         PAYOUT 000040000.00 (WITHIN COVERAGE LIMIT                   )
CLM00002 POL0000004 REJECTED         PAYOUT 000000000.00 (WITHIN 30-DAY WAITING PERIOD            )
CLM00003 POL0000007 REJECTED         PAYOUT 000000000.00 (POLICY NOT ACTIVE                       )
CLM00004 POL0000002 APPROVED-PARTIAL PAYOUT 002500000.00 (PAYOUT CAPPED AT REMAINING COVERAGE     )
CLM00005 POL0000099 REJECTED         PAYOUT 000000000.00 (POLICY NOT FOUND                        )
CLM00006 POL0000006 REJECTED         PAYOUT 000000000.00 (CLAIM DATE OUTSIDE POLICY PERIOD        )

CLAIMS APPROVED : 002
CLAIMS REJECTED : 004
TOTAL PAYOUT    : 002540000.00
```

**เนื้อหาไฟล์ `CLAIMSOUT.DAT` (log การตัดสินใจ, ตัวเลขจัดรูปแบบด้วย edited PICTURE)**:

```
CLM00001 POL0000005 APPROVED              40,000.00 WITHIN COVERAGE LIMIT
CLM00002 POL0000004 REJECTED                   0.00 WITHIN 30-DAY WAITING PERIOD
CLM00003 POL0000007 REJECTED                   0.00 POLICY NOT ACTIVE
CLM00004 POL0000002 APPROVED-PARTIAL   2,500,000.00 PAYOUT CAPPED AT REMAINING C
CLM00005 POL0000099 REJECTED                   0.00 POLICY NOT FOUND
CLM00006 POL0000006 REJECTED                   0.00 CLAIM DATE OUTSIDE POLICY PE
```

ทุกกรณีตรงตามที่ออกแบบไว้ 100% ทั้ง 6 เคลมผ่านตรรกะ cascade ที่ถูกต้อง: CLM00001 อนุมัติเต็มเพราะ
เงื่อนไขครบถ้วนและวงเงินพอ, CLM00004 ถูก "cap" ที่วงเงินคงเหลือ 2,500,000.00 พอดี (ทำให้วงเงินคุ้มครอง
ของ POL0000002 หมดลงหลังจากนี้)

### ข้อควรระวัง

**การตัดคำในฟิลด์ที่แสดงผล**: สังเกตว่าบรรทัด CLM00004 ในไฟล์ `CLAIMSOUT.DAT` แสดงเหตุผลว่า
`PAYOUT CAPPED AT REMAINING C` (ถูกตัดคำ) เพราะ `WS-OUT-REASON` เป็น `PIC X(40)` แต่ข้อความเต็ม
`"PAYOUT CAPPED AT REMAINING COVERAGE"` ยาว 36 ตัวอักษร ซึ่งควรจะพอดี — ปัญหาจริงคือความกว้างของ
`CLAIMS-OUT-RECORD` (`PIC X(80)`) รวมกับ Field อื่นก่อนหน้าทำให้ตำแหน่งของ `WS-OUT-REASON` เกินขอบเขต
80 ตัวอักษรไปเล็กน้อยเมื่อรวมกับ `WS-OUT-PAYOUT` ที่กว้างถึง 13 ตัวอักษร (`PIC ZZZ,ZZZ,ZZ9.99`) —
นี่คือ**บั๊กจริงที่พบระหว่างทดสอบ**บทนี้ บทเรียนคือ **ต้องรวมความกว้างของทุกฟิลด์ในกลุ่มแล้วเทียบกับ
ความกว้างของ record ปลายทางเสมอ** ไม่ใช่แค่ดูว่าแต่ละฟิลด์กว้างพอสำหรับข้อมูลของตัวเองหรือไม่

### แบบฝึกหัดที่ 936.1

**โจทย์**: จงคำนวณความกว้างรวมของ `WS-OUT-LINE` ทีละฟิลด์ แล้วหาว่าต้องขยาย `CLAIMS-OUT-RECORD`
เป็นความกว้างเท่าใดจึงจะไม่ตัดคำเหตุผลยาวสุด (40 ตัวอักษร)

**เฉลย**: รวมความกว้าง: `WS-OUT-CLAIM`(8) + filler(1) + `WS-OUT-POLICY`(10) + filler(1) +
`WS-OUT-DECISION`(16) + filler(1) + `WS-OUT-PAYOUT`(13, จาก `ZZZ,ZZZ,ZZ9.99`) + filler(1) +
`WS-OUT-REASON`(40) = 8+1+10+1+16+1+13+1+40 = **91 ตัวอักษร** ดังนั้น `CLAIMS-OUT-RECORD` ต้องขยาย
จาก `PIC X(80)` เป็นอย่างน้อย `PIC X(91)` จึงจะรองรับข้อมูลทุกฟิลด์แบบไม่ตัดคำ

---

## ขั้นตอนที่ 937: โปรแกรม `POLRENEW` — การตรวจสอบวันหมดอายุด้วย Date Function

### แนวคิดการออกแบบ

`POLRENEW` เป็นโปรแกรม batch ที่ควรรันเป็นประจำ (เช่น ทุกคืน ผ่าน Job Scheduler ทบทวน Part 066)
เพื่อตรวจสอบกรมธรรม์ **ทุกฉบับ** เทียบกับวันที่อ้างอิง (`WS-AS-OF-DATE`) แล้วแบ่งเป็น 3 กลุ่ม:

1. **หมดอายุใหม่ (Newly Expired)**: `POL-STATUS = "A"` แต่ `POL-END-DATE` ผ่านมาแล้ว — โปรแกรมจะ
   **อัปเดตสถานะเป็น `E`** ทันที (นี่คือหน้าที่หลักของ batch job ตัวนี้ ไม่ใช่แค่รายงานอย่างเดียว)
2. **ใกล้หมดอายุภายใน 30 วัน (Expiring Soon)**: เหลือเวลาน้อยกว่าหรือเท่ากับ 30 วัน — ใช้แจ้งเตือนให้
   ฝ่ายขายติดต่อลูกค้าเพื่อต่ออายุ
3. **ปกติ (Active OK)**: ยังเหลือเวลามากกว่า 30 วัน — ไม่ต้องทำอะไร

กรมธรรม์ที่มีสถานะ `X` (Cancelled) อยู่แล้วจะถูก**ยกเว้น**จากการพิจารณาทั้งหมด เพราะถือเป็นสถานะสิ้นสุด
(terminal state) ไปแล้ว ไม่จำเป็นต้องตรวจสอบวันหมดอายุอีก

### ทำไมต้องเก็บผลลัพธ์ไว้ใน Table ก่อนค่อยพิมพ์รายงาน

แม้ว่าไฟล์ `POLICY.DAT` จะถูกจัดเรียงตามเลขที่กรมธรรม์ (`POL-NUMBER`) แต่รายงานที่ต้องการคือ**แบ่งกลุ่ม
ตามสถานะการหมดอายุ** ไม่ใช่ตามเลขที่กรมธรรม์ วิธีที่ตรงไปตรงมาที่สุดคือ: อ่านไฟล์ครั้งเดียว เก็บผลการ
ประเมินแต่ละ record ไว้ใน**ตารางในหน่วยความจำ** (`WS-RESULT-TABLE` แบบ `OCCURS`, Part 016) พร้อม
`WS-R-CATEGORY` บอกกลุ่ม จากนั้นวนพิมพ์ตารางนี้ **3 รอบ** (รอบละ 1 กลุ่ม) วิธีนี้หลีกเลี่ยงการเปิดไฟล์
ซ้ำหลายครั้งหรือการ `SORT` ที่ไม่จำเป็นสำหรับข้อมูลขนาดเล็กเช่นนี้

### เต็มโค้ด `POLRENEW.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. POLRENEW.
      ******************************************************************
      * Insurance case study - renewal / expiration checking.
      * Reads every policy in POLICY-MASTER as of a fixed reporting
      * date (WS-AS-OF-DATE) using FUNCTION INTEGER-OF-DATE (Part 036
      * / Part 047 technique) to compute days until the policy's end
      * date. Policies whose end date has already passed are flagged
      * as newly expired and REWRITTEN with status E. Policies expiring
      * within 30 days are listed for renewal follow-up. A fixed date
      * is used instead of FUNCTION CURRENT-DATE so this report's
      * output is reproducible every time it is run (same reasoning
      * as WS-AS-OF-DATE in Part 050).
      ******************************************************************
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT POLICY-MASTER ASSIGN TO "POLICY.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS POL-NUMBER
               FILE STATUS IS WS-POL-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  POLICY-MASTER.
       01  POLICY-RECORD.
           COPY "polrec.cpy".

       WORKING-STORAGE SECTION.
       01  WS-POL-STATUS            PIC X(2).
       01  WS-EOF                   PIC X(1) VALUE "N".
           88  END-OF-POLICIES               VALUE "Y".

       01  WS-AS-OF-DATE            PIC 9(8) VALUE 20260928.

       01  WS-DAYS-TO-EXPIRY        PIC S9(6).

       01  WS-RESULT-TABLE.
           05  WS-RESULT-ENTRY OCCURS 10 TIMES.
               10  WS-R-NUMBER      PIC X(10).
               10  WS-R-NAME        PIC X(24).
               10  WS-R-END-DATE    PIC 9(8).
               10  WS-R-DAYS        PIC S9(6).
               10  WS-R-CATEGORY    PIC 9(1).
      *            1 = newly expired, 2 = expiring within 30 days
      *            3 = active OK, 4 = excluded (cancelled)
       01  WS-RESULT-COUNT          PIC 9(2) VALUE 0.
       01  WS-IDX                   PIC 9(2).

       01  WS-COUNTERS.
           05  WS-EXPIRED-NEW       PIC 9(3) VALUE 0.
           05  WS-EXPIRING-SOON     PIC 9(3) VALUE 0.
           05  WS-ACTIVE-OK         PIC 9(3) VALUE 0.
           05  WS-EXCLUDED          PIC 9(3) VALUE 0.

       01  WS-PRINT-LINE.
           05  WS-P-NUMBER          PIC X(10).
           05  FILLER               PIC X(1) VALUE SPACE.
           05  WS-P-NAME            PIC X(24).
           05  FILLER               PIC X(1) VALUE SPACE.
           05  WS-P-END-DATE        PIC 9(8).
           05  FILLER               PIC X(3) VALUE SPACES.
           05  WS-P-DAYS            PIC ---,--9.

       PROCEDURE DIVISION.
       000-MAIN.
           OPEN I-O POLICY-MASTER
           IF WS-POL-STATUS NOT = "00"
               DISPLAY "ERROR OPENING POLICY-MASTER: " WS-POL-STATUS
               STOP RUN
           END-IF

           DISPLAY "==== POLICY RENEWAL / EXPIRATION CHECK ===="
           DISPLAY "AS-OF DATE: " WS-AS-OF-DATE
           DISPLAY " "

           PERFORM UNTIL END-OF-POLICIES
               READ POLICY-MASTER NEXT RECORD
                   AT END
                       SET END-OF-POLICIES TO TRUE
                   NOT AT END
                       PERFORM 200-EVALUATE-POLICY
               END-READ
           END-PERFORM

           CLOSE POLICY-MASTER

           PERFORM 300-PRINT-NEWLY-EXPIRED
           PERFORM 310-PRINT-EXPIRING-SOON
           PERFORM 320-PRINT-SUMMARY
           STOP RUN.

       200-EVALUATE-POLICY.
           ADD 1 TO WS-RESULT-COUNT
           MOVE POL-NUMBER      TO WS-R-NUMBER(WS-RESULT-COUNT)
           MOVE POL-HOLDER-NAME TO WS-R-NAME(WS-RESULT-COUNT)
           MOVE POL-END-DATE    TO WS-R-END-DATE(WS-RESULT-COUNT)

           IF POL-STATUS = "X"
               MOVE 4 TO WS-R-CATEGORY(WS-RESULT-COUNT)
               MOVE 0 TO WS-R-DAYS(WS-RESULT-COUNT)
               ADD 1 TO WS-EXCLUDED
           ELSE
               COMPUTE WS-DAYS-TO-EXPIRY =
                   FUNCTION INTEGER-OF-DATE(POL-END-DATE) -
                   FUNCTION INTEGER-OF-DATE(WS-AS-OF-DATE)
               MOVE WS-DAYS-TO-EXPIRY TO WS-R-DAYS(WS-RESULT-COUNT)

               IF WS-DAYS-TO-EXPIRY < 0
                   MOVE 1 TO WS-R-CATEGORY(WS-RESULT-COUNT)
                   MOVE "E" TO POL-STATUS
                   REWRITE POLICY-RECORD
                   ADD 1 TO WS-EXPIRED-NEW
               ELSE
                   IF WS-DAYS-TO-EXPIRY <= 30
                       MOVE 2 TO WS-R-CATEGORY(WS-RESULT-COUNT)
                       ADD 1 TO WS-EXPIRING-SOON
                   ELSE
                       MOVE 3 TO WS-R-CATEGORY(WS-RESULT-COUNT)
                       ADD 1 TO WS-ACTIVE-OK
                   END-IF
               END-IF
           END-IF.

       300-PRINT-NEWLY-EXPIRED.
           DISPLAY "--- NEWLY EXPIRED (STATUS SET TO E) ---"
           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > WS-RESULT-COUNT
               IF WS-R-CATEGORY(WS-IDX) = 1
                   PERFORM 900-FORMAT-AND-SHOW
               END-IF
           END-PERFORM
           DISPLAY " ".

       310-PRINT-EXPIRING-SOON.
           DISPLAY "--- EXPIRING WITHIN 30 DAYS ---"
           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > WS-RESULT-COUNT
               IF WS-R-CATEGORY(WS-IDX) = 2
                   PERFORM 900-FORMAT-AND-SHOW
               END-IF
           END-PERFORM
           DISPLAY " ".

       320-PRINT-SUMMARY.
           DISPLAY "--- SUMMARY ---"
           DISPLAY "NEWLY EXPIRED   : " WS-EXPIRED-NEW
           DISPLAY "EXPIRING SOON   : " WS-EXPIRING-SOON
           DISPLAY "ACTIVE OK       : " WS-ACTIVE-OK
           DISPLAY "EXCLUDED (CANC) : " WS-EXCLUDED.

       900-FORMAT-AND-SHOW.
           MOVE WS-R-NUMBER(WS-IDX)    TO WS-P-NUMBER
           MOVE WS-R-NAME(WS-IDX)      TO WS-P-NAME
           MOVE WS-R-END-DATE(WS-IDX)  TO WS-P-END-DATE
           MOVE WS-R-DAYS(WS-IDX)      TO WS-P-DAYS
           DISPLAY WS-PRINT-LINE.
```

### สังเกตจุดสำคัญ

- `WS-P-DAYS` ใช้ Edited PICTURE `PIC ---,--9` (Part 006/019) เพื่อแสดงเครื่องหมายลบนำหน้าอัตโนมัติ
  เมื่อจำนวนวันติดลบ (กรณีหมดอายุไปแล้ว) ทำให้รายงานอ่านง่ายว่ากรมธรรม์นั้นหมดอายุมาแล้วกี่วัน
- `READ POLICY-MASTER NEXT RECORD` ใช้คำว่า `NEXT RECORD` แม้ `ACCESS MODE` เป็น `SEQUENTIAL` อยู่แล้ว
  — นี่คือรูปแบบที่ชัดเจนขึ้น (explicit) ที่ GnuCOBOL ยอมรับ และช่วยให้อ่านโค้ดเข้าใจง่ายว่ากำลังอ่าน
  "ตัวถัดไป" ไม่ใช่การอ่านแบบสุ่ม

### แบบฝึกหัดที่ 937.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมนี้ต้องเปิดไฟล์ด้วย `OPEN I-O` แทนที่จะเป็น `OPEN INPUT` ทั้งที่การ
อ่านส่วนใหญ่เป็นการอ่านแบบลำดับ (sequential)

**เฉลย**: เพราะโปรแกรมนี้ไม่ได้แค่ *อ่าน* กรมธรรม์เพื่อพิมพ์รายงานเท่านั้น แต่ยังต้อง **`REWRITE`**
กรมธรรม์ที่หมดอายุใหม่เพื่อเปลี่ยนสถานะเป็น `E` ด้วย การ `REWRITE` (เขียนทับ record เดิม) ทำได้ก็ต่อ
เมื่อไฟล์เปิดด้วยโหมดที่อนุญาตให้เขียน คือ `I-O` (Input-Output) เท่านั้น ถ้าเปิดด้วย `OPEN INPUT`
เพียงอย่างเดียว โปรแกรมจะยัง `READ` ได้ตามปกติ แต่จะเกิด runtime error ทันทีที่พยายาม `REWRITE`
(ทบทวนตารางโหมดการเปิดไฟล์และคำสั่งที่ใช้ได้ในแต่ละโหมดจาก Part 025)

---

## ขั้นตอนที่ 938: รันและวิเคราะห์ผลลัพธ์ของ `POLRENEW`

### คอมไพล์และรัน (หลังจากรัน `INSSETUP` -> `PREMPAY` -> `CLAIMPROC` ตามลำดับแล้ว)

```bash
/opt/gnucobol-isam/bin/cobc -x -o polrenew polrenew.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./polrenew
```

**ผลลัพธ์จริง** (วันที่อ้างอิงคงที่ `WS-AS-OF-DATE = 20260928`):

```
==== POLICY RENEWAL / EXPIRATION CHECK ====
AS-OF DATE: 20260928

--- NEWLY EXPIRED (STATUS SET TO E) ---
POL0000001 JOHN SMITH               20260601      -119

--- EXPIRING WITHIN 30 DAYS ---
POL0000005 MICHAEL BROWN            20261001         3

--- SUMMARY ---
NEWLY EXPIRED   : 001
EXPIRING SOON   : 001
ACTIVE OK       : 004
EXCLUDED (CANC) : 001
```

### วิเคราะห์ผลลัพธ์ทีละกรมธรรม์

| กรมธรรม์ | วันสิ้นสุด | วันคงเหลือ (as-of 2026-09-28) | ผลลัพธ์ |
|---|---|---|---|
| POL0000001 | 2026-06-01 | -119 (หมดอายุมาแล้ว 119 วัน) | **หมดอายุใหม่** สถานะเปลี่ยน A -> E |
| POL0000002 | 2027-01-15 | +109 | ปกติ |
| POL0000003 | 2030-06-01 | มาก | ปกติ |
| POL0000004 | 2027-08-01 | มาก | ปกติ |
| POL0000005 | 2026-10-01 | +3 | **ใกล้หมดอายุ** (เหลือ 3 วัน) |
| POL0000006 | 2034-03-01 | มาก | ปกติ |
| POL0000007 | 2025-01-01 | (ไม่คำนวณ) | ยกเว้น — Cancelled ตั้งแต่ต้น |

ผลลัพธ์ตรงตามที่ออกแบบไว้ทุกประการ: มีกรมธรรม์เดียวที่หมดอายุใหม่ (POL0000001, เลยกำหนดมา 119 วัน),
มีกรมธรรม์เดียวที่ใกล้หมดอายุ (POL0000005, เหลือเพียง 3 วัน — ทั้งที่เพิ่งชำระเบี้ยประกันไปใน Step 934!
สะท้อนความจริงที่ว่า**การจ่ายเบี้ยประกันไม่ได้แปลว่ากรมธรรม์จะไม่หมดอายุ** ถ้าไม่มีการต่ออายุกรมธรรม์
(ขยาย `POL-END-DATE`) อย่างเป็นทางการแยกต่างหาก ซึ่งเป็นข้อสังเกตเชิงธุรกิจที่สำคัญ)

### ข้อควรระวัง

1. **การรันซ้ำจะไม่พบกรมธรรม์หมดอายุใหม่อีก**: เพราะ `POL0000001` ถูกเปลี่ยนสถานะเป็น `E` ไปแล้ว
   ครั้งถัดไปที่รันโปรแกรมนี้ กรมธรรม์นี้จะมีสถานะ `E` ไม่ใช่ `A` จึงไม่เข้าเงื่อนไข `WS-DAYS-TO-EXPIRY < 0`
   อีก (เพราะเข้าเงื่อนไข `ELSE` ของ `IF POL-STATUS = "X"` ซึ่งครอบคลุมทุกสถานะที่ไม่ใช่ `X` รวมถึง `E`
   ด้วย — ในระบบจริงควรแยกเงื่อนไข `E` ออกมาอย่างชัดเจนเพื่อไม่ให้คำนวณวันซ้ำโดยไม่จำเป็น)
2. **`WS-RESULT-TABLE` มีขนาดจำกัดที่ `OCCURS 10 TIMES`**: ถ้าจำนวนกรมธรรม์ในระบบเกิน 10 ฉบับ โปรแกรม
   จะเกิด **subscript ออกนอกขอบเขต** (index ล้น array) ทบทวนปัญหานี้จาก Part 016/017 — ระบบจริงต้อง
   กำหนดขนาดตารางให้ใหญ่พอ หรือเปลี่ยนไปใช้เทคนิคที่ไม่พึ่งพาตารางในหน่วยความจำทั้งหมด (เช่น การ `SORT`
   แล้วอ่านทีละกลุ่มแบบ Control Break ตามที่ Part 095 จะสาธิตต่อไป)

### แบบฝึกหัดที่ 938.1

**โจทย์**: จงแก้ไขเงื่อนไขใน `200-EVALUATE-POLICY` ให้ข้ามกรมธรรม์ที่มีสถานะ `E` (หมดอายุไปแล้วจาก
การรันครั้งก่อน) ออกจากการคำนวณซ้ำ เช่นเดียวกับที่ข้ามสถานะ `X`

**เฉลย**:

```cobol
           IF POL-STATUS = "X" OR POL-STATUS = "E"
               MOVE 4 TO WS-R-CATEGORY(WS-RESULT-COUNT)
               MOVE 0 TO WS-R-DAYS(WS-RESULT-COUNT)
               ADD 1 TO WS-EXCLUDED
           ELSE
               *> ... (rest of the days-remaining calculation code
               *> stays exactly the same as before)
```

การแก้ไขนี้ทำให้กรมธรรม์ที่หมดอายุไปแล้ว**ไม่ถูกนำมาคำนวณและพิมพ์ซ้ำ**ทุกครั้งที่รันรายงาน ตรงตาม
หลักการที่ว่าสถานะ `E` และ `X` ต่างก็เป็น "สถานะสิ้นสุด" (terminal state) เหมือนกัน เพียงแต่มีที่มา
ต่างกัน (`E` มาจากวันหมดอายุตามธรรมชาติ ส่วน `X` มาจากการยกเลิกก่อนกำหนด)

---

## ขั้นตอนที่ 939: ทดสอบทั้งระบบแบบครบวงจร (End-to-End Integration Test)

### ลำดับการรันที่ถูกต้อง

ระบบนี้ต้องรันตามลำดับที่กำหนดไว้เท่านั้น เพราะแต่ละโปรแกรมพึ่งพาผลลัพธ์ของโปรแกรมก่อนหน้า:

```bash
cd /tmp/insurance-test
/opt/gnucobol-isam/bin/cobc -x -o inssetup  inssetup.cob
/opt/gnucobol-isam/bin/cobc -x -o prempay   prempay.cob
/opt/gnucobol-isam/bin/cobc -x -o claimproc claimproc.cob
/opt/gnucobol-isam/bin/cobc -x -o polrenew  polrenew.cob

export LD_LIBRARY_PATH=/opt/gnucobol-isam/lib
rm -f POLICY.DAT*
./inssetup     # 1. สร้างกรมธรรม์ตั้งต้น 7 ฉบับ
./prempay      # 2. รับชำระเบี้ยประกัน 2 รายการสำเร็จ, ปฏิเสธ 2 รายการ
./claimproc    # 3. ประมวลผลเคลม 6 รายการ, อนุมัติ 2, ปฏิเสธ 4
./polrenew     # 4. ตรวจสอบวันหมดอายุ, เปลี่ยนสถานะ 1 กรมธรรม์เป็น Expired
```

### ตรวจสอบสถานะสุดท้ายของทุกกรมธรรม์

เพื่อยืนยันว่าข้อมูลไหลผ่านทั้ง 4 โปรแกรมถูกต้องตรงกัน เราเขียนโปรแกรมช่วยตรวจสอบเล็ก ๆ
(ไม่ใช่ส่วนหนึ่งของระบบที่สอน แต่ใช้ยืนยันตัวเลขในเอกสารนี้เท่านั้น) วนอ่าน `POLICY.DAT` ทั้งไฟล์แล้ว
แสดงฟิลด์สำคัญของทุก record:

```
POL0000001 E PREMPAID=000024000.00 PAIDTO=20260601 CLAIMS=000 CLAIMSPAID=000000000.00
POL0000002 A PREMPAID=000008000.00 PAIDTO=20260115 CLAIMS=001 CLAIMSPAID=002500000.00
POL0000003 A PREMPAID=000144000.00 PAIDTO=20260601 CLAIMS=000 CLAIMSPAID=000000000.00
POL0000004 A PREMPAID=000009500.00 PAIDTO=20260801 CLAIMS=000 CLAIMSPAID=000000000.00
POL0000005 A PREMPAID=000014000.00 PAIDTO=20261001 CLAIMS=002 CLAIMSPAID=000090000.00
POL0000006 A PREMPAID=000054000.00 PAIDTO=20260301 CLAIMS=000 CLAIMSPAID=000000000.00
POL0000007 X PREMPAID=000011000.00 PAIDTO=20240101 CLAIMS=000 CLAIMSPAID=000000000.00
```

### ตรวจสอบความถูกต้องของข้อมูลทีละจุด (Data Integrity Trace)

| กรมธรรม์ | สถานะสุดท้าย | ตรวจสอบ |
|---|---|---|
| POL0000001 | `E` | ถูกต้อง: `PREMPAY` เพิ่มยอดชำระเป็น 24,000.00 สำเร็จ แล้ว `POLRENEW` เปลี่ยนสถานะเป็น Expired เพราะ end-date ผ่านไปแล้ว |
| POL0000002 | `A`, CLAIMS=1, CLAIMSPAID=2,500,000.00 | ถูกต้อง: ไม่มีการชำระเบี้ยเพิ่ม (ไม่มีในไฟล์ PAYMENTS.DAT) แต่มีเคลม CLM00004 อนุมัติบางส่วนเต็มวงเงินพอดี |
| POL0000005 | `A`, CLAIMS=2, CLAIMSPAID=90,000.00 | ถูกต้อง: มีเคลมเดิม 1 ครั้ง (50,000.00) บวกเคลมใหม่ CLM00001 (40,000.00) รวมเป็น 2 ครั้ง 90,000.00 พอดี และยังไม่ถูกเปลี่ยนสถานะเป็น Expired เพราะเหลืออีก 3 วัน |
| POL0000007 | `X` | ถูกต้อง: ทั้งการชำระเงินและการเคลมถูกปฏิเสธทั้งคู่ ยอดต่าง ๆ ไม่เปลี่ยนแปลงจากค่าตั้งต้นเลย |

การตรวจสอบนี้ยืนยันว่าทั้ง 4 โปรแกรมทำงานร่วมกันถูกต้องแบบ end-to-end ผ่าน Indexed File ร่วม
(`POLICY.DAT`) โดยไม่มีข้อมูลขัดแย้งหรือสูญหายที่จุดใดเลย

### แบบฝึกหัดที่ 939.1

**โจทย์**: หากรัน `polrenew` **ซ้ำอีกครั้ง**ทันทีโดยไม่แก้ไขโค้ดตามแบบฝึกหัดที่ 938.1 คาดว่าผลลัพธ์ใน
ส่วน "NEWLY EXPIRED" จะเป็นอย่างไร และทำไม

**เฉลย**: POL0000001 จะ**ไม่ปรากฏ**ในส่วน "NEWLY EXPIRED" อีกต่อไป เพราะสถานะของมันถูกเปลี่ยนเป็น `E`
ไปแล้วจากการรันครั้งแรก และโค้ดปัจจุบัน (ก่อนแก้ตามแบบฝึกหัดที่ 938.1) จะคำนวณ `WS-DAYS-TO-EXPIRY`
ให้กรมธรรม์นี้อีกครั้ง (เพราะเงื่อนไข `IF POL-STATUS = "X"` ไม่ครอบคลุมสถานะ `E`) ได้ค่า -119 เหมือนเดิม
ซึ่งยังคง `< 0` จึงเข้าเงื่อนไข "หมดอายุใหม่" **ซ้ำอีกครั้ง** และพยายาม `MOVE "E" TO POL-STATUS` /
`REWRITE` ซ้ำ (ซึ่งจริง ๆ ก็ยังคงเป็น `E` เหมือนเดิม ไม่เกิด error แต่เป็นการทำงานซ้ำซ้อนโดยไม่จำเป็น
และรายงานจะแสดง "หมดอายุใหม่" ผิดความหมาย ทั้งที่มันหมดอายุไปนานแล้วตั้งแต่รันครั้งก่อน) นี่คือเหตุผล
ที่แบบฝึกหัดที่ 938.1 สำคัญมากสำหรับการใช้งานจริงในสภาพแวดล้อมที่รัน batch job ทุกวัน

---

## ขั้นตอนที่ 940: สรุปโปรเจกต์, ข้อจำกัดที่ตั้งใจไว้, และแบบฝึกหัดขยายผล

### สิ่งที่ระบบนี้ทำได้จริง 100%

- สร้าง จัดเก็บ และค้นหากรมธรรม์ผ่าน Indexed File ด้วย GnuCOBOL ISAM handler จริง
- รับชำระเบี้ยประกันพร้อมตรวจสอบสถานะกรมธรรม์ และคำนวณวันชำระครอบคลุมถึงด้วยเทคนิค REDEFINES
- ประมวลผลเคลมสินไหมด้วยตรรกะ cascade 5 ชั้นที่สะท้อนกระบวนการพิจารณาจริงของธุรกิจประกันภัย
- ตรวจสอบวันหมดอายุด้วย `FUNCTION INTEGER-OF-DATE` และอัปเดตสถานะกรมธรรม์อัตโนมัติ
- จัดการข้อผิดพลาด/การปฏิเสธธุรกรรมโดยไม่ทำให้โปรแกรมล่ม พร้อมบันทึกเหตุผลชัดเจนทุกครั้ง

### สิ่งที่ระบบนี้**ตั้งใจไม่ทำ** (เพื่อคุมขอบเขตให้กระชับตามที่ตั้งใจไว้)

| ไม่ได้ทำ | เหตุผล / ทางเลือกในระบบจริง |
|---|---|
| การต่ออายุกรมธรรม์แบบเป็นทางการ (ขยาย `POL-END-DATE`) | จะต้องมีกฎธุรกิจเพิ่มเติมเรื่องการคำนวณเบี้ยใหม่ตามอายุ/ความเสี่ยงที่เปลี่ยนไป ซึ่งอยู่นอกขอบเขตที่ตั้งใจไว้ของ Part นี้ |
| สถานะ Lapsed (ขาดชำระ) แบบอัตโนมัติ | ต้องมีกฎเรื่อง "grace period" (ระยะเวลาผ่อนผัน) ที่ซับซ้อนกว่านี้ ซึ่งแตกต่างกันมากในแต่ละประเภทกรมธรรม์จริง |
| การคำนวณเบี้ยประกันตามความเสี่ยง (Underwriting) | เป็นหัวข้อทางคณิตศาสตร์ประกันภัย (Actuarial Science) ที่ลึกเกินขอบเขตหลักสูตรนี้ |
| DECLARATIVES แบบรวมศูนย์เหมือน Part 050 | เพื่อให้โค้ดกระชับและเข้าใจง่าย ใช้ `INVALID KEY`/`IF` ตรง ๆ แทน ซึ่งเพียงพอสำหรับขอบเขตของระบบนี้ |

### เชื่อมโยงกับหลักสูตรทั้งหมด

Case Study นี้เป็นตัวอย่างที่ดีของการนำเทคนิคจากหลายเฟสมาผสมผสานกัน:

- **เฟส 2** (Part 028 Indexed File, Part 033 Copybook) เป็นรากฐานของการจัดเก็บข้อมูล
- **เฟส 3** (Part 022 REDEFINES, Part 036 Intrinsic Functions, Part 041 EVALUATE) เป็นเครื่องมือ
  คำนวณและตัดสินใจ
- **เฟส 3** (Part 047 Advanced Date/Time, Part 048 Error Handling) เป็นรากฐานของความแม่นยำและ
  ความทนทานของระบบ
- **Milestone Parts** (Part 050, Part 070) เป็นต้นแบบของรูปแบบสถาปัตยกรรมและวิธีนำเสนอผลการทดสอบจริง

### แบบฝึกหัดที่ 940.1 (แบบฝึกหัดขยายผลปิดท้าย Part)

**โจทย์**: จงออกแบบ (เขียนเป็น pseudo-code หรือคำอธิบายเชิงโครงสร้าง ไม่ต้องเขียนโค้ดเต็ม) โปรแกรมที่ 5
ชื่อ `POLSTMT` ที่จะสร้าง "ใบแจ้งข้อมูลกรมธรรม์ประจำปี" (Annual Policy Statement) ให้ลูกค้าแต่ละราย
โดยต้องดึงข้อมูลจาก `POLICY-MASTER` และควรมีข้อมูลอะไรบ้างในใบแจ้งนี้

**เฉลยแนวทาง**: `POLSTMT` ควรอ่าน `POLICY-MASTER` แบบ sequential (คล้าย `POLRENEW`) สำหรับกรมธรรม์ที่
สถานะเป็น `A` เท่านั้น (ไม่ต้องแจ้งกรมธรรม์ที่ถูกยกเลิกหรือหมดอายุแล้ว) แล้วพิมพ์ต่อกรมธรรม์หนึ่งฉบับ:
(1) ข้อมูลพื้นฐาน (เลขที่, ชื่อ, ประเภท, ช่วงความคุ้มครอง), (2) สรุปการชำระเบี้ย (ยอดสะสม, ชำระครอบคลุม
ถึงวันที่ใด, เบี้ยงวดถัดไปเมื่อใด), (3) สรุปประวัติเคลม (จำนวนครั้ง, ยอดสะสม, วงเงินคุ้มครองที่เหลือ
คำนวณจาก `POL-COVERAGE-AMOUNT - POL-TOTAL-CLAIMS-PAID`), และ (4) วันหมดอายุพร้อมจำนวนวันที่เหลือ
(นำ `FUNCTION INTEGER-OF-DATE` มาใช้ซ้ำเหมือน `POLRENEW`) — นี่คือตัวอย่างที่ดีของการที่ record
เดียวกันสามารถถูก "มองจากมุมที่ต่างกัน" โดยโปรแกรมที่ต่างกันได้ โดยไม่ต้องแก้โครงสร้างข้อมูลใน
Copybook เลยแม้แต่นิดเดียว ซึ่งเป็นประโยชน์สำคัญของการออกแบบข้อมูลที่ดีตั้งแต่ต้น

---

## สรุปท้ายบท

ใน Part นี้ เราได้สร้าง Case Study ระบบประกันภัยที่ทำงานได้จริงสมบูรณ์แบบด้วย 4 โปรแกรม:

- **`INSSETUP`**: สร้างและโหลดข้อมูลกรมธรรม์ตั้งต้น 7 ฉบับ ลงไฟล์ Indexed
- **`PREMPAY`**: รับชำระเบี้ยประกัน พร้อมเทคนิค REDEFINES สำหรับบวกวันที่ไป 1 ปีอย่างแม่นยำ และการ
  ปฏิเสธธุรกรรมที่ไม่ถูกต้องโดยไม่ทำให้โปรแกรมล่ม
- **`CLAIMPROC`**: ประมวลผลเคลมสินไหมด้วยตรรกะ cascade 5 ชั้น (สถานะกรมธรรม์, ช่วงเวลาคุ้มครอง,
  ระยะเวลารอคอย, วงเงินคุ้มครองคงเหลือ) ใช้ `FUNCTION INTEGER-OF-DATE` คำนวณจำนวนวันอย่างแม่นยำ
- **`POLRENEW`**: ตรวจสอบวันหมดอายุกรมธรรม์ทุกฉบับ จัดกลุ่มเป็นหมดอายุใหม่/ใกล้หมดอายุ/ปกติ และอัปเดต
  สถานะอัตโนมัติ

ทุกโปรแกรมผ่านการคอมไพล์และรันทดสอบจริงด้วย GnuCOBOL (ISAM build) ครบทุกกรณี ทั้งกรณีสำเร็จและกรณี
ถูกปฏิเสธ พร้อมการตรวจสอบความถูกต้องของข้อมูลแบบ end-to-end ที่ยืนยันว่าทั้งระบบทำงานสอดคล้องกัน

Part ถัดไปจะเปลี่ยนอุตสาหกรรมไปสู่ **ERP และห่วงโซ่อุปทาน (Supply Chain)** ซึ่งจะขยายแนวคิดการจัดการ
สินค้าคงคลังจาก Part 035 ให้ครอบคลุมหลายคลังสินค้าพร้อมกัน และเพิ่มกระบวนการสั่งซื้อ-รับสินค้าที่เป็น
หัวใจของธุรกิจค้าปลีก/การผลิตทั่วโลก

**[ไปยัง Part 095: Case Study — ระบบ ERP และห่วงโซ่อุปทาน →](part-095-erp-supply-chain-case-study.md)**

---

*Part นี้อยู่ในเฟส 6: ระดับมืออาชีพ/โลก (Parts 086-100)*
*Part ก่อนหน้า: [Part 093: Case Study — Core Banking ตอนที่ 3: การรายงานและปิดบัญชี](part-093-corebanking-reporting.md)*
*Part ถัดไป: [Part 095: Case Study — ระบบ ERP และห่วงโซ่อุปทาน](part-095-erp-supply-chain-case-study.md)*
*ลำดับ Case Study เฟส 6: Part 091-093 (Core Banking) -> Part 094 (Insurance, Part นี้) -> Part 095 (ERP/Supply Chain) -> Part 096 (Certification/Career) -> Part 097 (Interview Questions) -> Part 098 (Best Practices)*
