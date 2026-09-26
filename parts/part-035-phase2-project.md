# Part 035: 🎯 โปรเจกต์เฟส 2: ระบบจัดการสินค้าคงคลัง (Inventory Management System) (ขั้นตอนที่ 341–350)

## คำนำของ Part นี้

เดินทางมาถึงจุดสำคัญอีกครั้ง! ตั้งแต่ Part 016 ถึง Part 034 คุณได้เรียนรู้เครื่องมือระดับกลางเกือบทั้งหมด
ของ COBOL ที่จำเป็นสำหรับการสร้างระบบธุรกิจจริง: ตาราง (Tables) และ `OCCURS` (Part 016), Subscript กับ
Index และ `SEARCH`/`SEARCH ALL` (Part 017), ตารางหลายมิติ (Part 018), การจัดการข้อความด้วย `STRING`
(Part 019) และ `UNSTRING` (Part 020), `INSPECT` (Part 021), `REDEFINES` (Part 022), ไฟล์ตามลำดับ
(Part 023), `FILE SECTION`/FD Entry (Part 024), `OPEN`/`CLOSE`/`READ`/`WRITE`/`REWRITE`/`DELETE`
(Part 025), การประมวลผลแบบ Master-Detail (Part 026), `SORT`/`MERGE` (Part 027), ไฟล์แบบ Indexed/ISAM
(Part 028), ไฟล์แบบ Relative (Part 029), File Status Codes (Part 030), Subprogram ด้วย `CALL`
(Part 031), การส่งพารามิเตอร์ `BY REFERENCE`/`BY CONTENT`/`BY VALUE` (Part 032), `COPY` Statement และ
Copybook (Part 033), และสุดท้าย Nested Programs กับ `END PROGRAM` (Part 034)

Part นี้คือ **โปรเจกต์รวบยอดเฟส 2 (Phase 2 Milestone Project)** ซึ่งใช้แนวทางเดียวกับ Part 015
(โปรเจกต์รวบยอดเฟส 1): ไม่มีไวยากรณ์ใหม่ แต่จะนำทุกสิ่งที่เรียนมาทั้งหมดในเฟส 2 มาประกอบร่างเป็น
**ระบบจัดการสินค้าคงคลัง (Inventory Management System)** ที่ใช้งานได้จริง โดยครั้งนี้ต่างจาก Part 015
ตรงที่เรา **มีเครื่องมือครบแล้ว** สำหรับการแบ่งระบบออกเป็นหลายไฟล์จริง ๆ (`COPY` สำหรับ Copybook และ
`CALL` สำหรับ Subprogram) จึงสามารถออกแบบสถาปัตยกรรมที่ใกล้เคียงกับระบบธุรกิจจริงในอุตสาหกรรมได้มากขึ้น
อย่างมีนัยสำคัญ: **โปรแกรมเมนูหลักหนึ่งตัวที่ไม่รู้รายละเอียดการคำนวณเลย คอยเรียก Subprogram แยกต่างหาก
ทีละตัวสำหรับแต่ละหน้าที่ทางธุรกิจ** โดยทุกตัวใช้โครงสร้าง record เดียวกันผ่าน Copybook ที่ใช้ร่วมกัน

ระบบที่จะสร้างในบทนี้ประกอบด้วยไฟล์ต้นฉบับ **8 ไฟล์** ที่ทำงานร่วมกัน:

| ไฟล์ | ชนิด | หน้าที่ |
|---|---|---|
| `INVREC.CPY` | Copybook | โครงสร้าง record ของสินค้า 1 รายการ ใช้ร่วมกันทุกโปรแกรม |
| `invcreate.cob` | โปรแกรม setup (รันครั้งเดียว) | สร้างไฟล์หลัก `INVMAST.DAT` แบบ Indexed เปล่า ๆ |
| `invmain.cob` | โปรแกรมเมนูหลัก (interactive) | แสดงเมนู, รับค่าจากผู้ใช้, `CALL` subprogram ตามตัวเลือก |
| `invadd.cob` | Subprogram | เพิ่มสินค้าใหม่ |
| `invrecv.cob` | Subprogram | รับสินค้าเข้าคลัง (เพิ่มจำนวน) |
| `invissue.cob` | Subprogram | เบิกสินค้าออกจากคลัง (ลดจำนวน พร้อมป้องกันติดลบ) |
| `invsrch.cob` | Subprogram | ค้นหาสินค้าด้วยรหัส |
| `invlist.cob` | Subprogram | พิมพ์รายงานสินค้าทั้งหมด |
| `invlow.cob` | Subprogram | พิมพ์รายงานแจ้งเตือนสินค้าใกล้หมด (ต่ำกว่าจุดสั่งซื้อ) |

ทุกไฟล์ในบทนี้ **คอมไพล์และรันได้จริง 100%** ด้วย GnuCOBOL (เวอร์ชัน 4.0-early ที่ใช้ทดสอบเนื้อหาทั้งหมด
ในหลักสูตรนี้ พร้อมตัวจัดการไฟล์แบบ Indexed เปิดใช้งานอยู่ ตามที่เรียนหลักการไว้ใน Part 028) คุณสามารถ
คัดลอกโค้ดทั้งหมดในบทนี้ไปคอมไพล์และรันบนเครื่องของคุณเองได้ทันที และทุกตัวอย่างผลลัพธ์ที่แสดงในบทนี้
คือผลลัพธ์จริงที่ได้จากการรันโค้ดจริง ไม่ใช่ผลลัพธ์ที่แต่งขึ้น — ระหว่างการเตรียมเนื้อหาบทนี้ เราพบบั๊กจริง
สองจุดจากการทดสอบจริง (ไม่ใช่ตัวอย่างสมมติ) ซึ่งจะเล่าให้ฟังพร้อมวิธีแก้ในขั้นตอนที่ 345 และ 347

### หมายเหตุสำคัญเรื่องการทดสอบ `invmain.cob` (โปรแกรมแบบโต้ตอบ)

เช่นเดียวกับ Part 015 โปรแกรม `invmain.cob` เป็นโปรแกรมแบบโต้ตอบ (interactive) ที่ใช้ `ACCEPT` รับข้อมูล
จากผู้ใช้ผ่านคีย์บอร์ด เราทดสอบโปรแกรมนี้แบบอัตโนมัติด้วยเทคนิคเดียวกับที่ใช้ใน Part 015 ทุกประการ คือ
**redirect ไฟล์ข้อความเข้าไปแทน stdin ของโปรเซส**:

```bash
cobc -x -o invmain invmain.cob invadd.cob invrecv.cob invissue.cob \
    invsrch.cob invlist.cob invlow.cob
./invmain < test_input.txt
```

โปรแกรมอ่านค่าจากไฟล์ `test_input.txt` ทีละบรรทัดเหมือนผู้ใช้พิมพ์แล้วกด Enter ทุกประการ ผลลัพธ์ที่แสดง
ในบทนี้จึงเป็นผลลัพธ์จริงจากการป้อนข้อมูลผ่าน stdin เข้าไปในโปรแกรมตัวเดียวกันกับที่คุณจะรันบนเครื่องของ
คุณเองแบบโต้ตอบ (ข้อแตกต่างเรื่อง terminal echo ก็เหมือนที่อธิบายไว้ใน Part 015 ทุกประการ) การทดสอบเต็ม
รูปแบบทั้งระบบจะอยู่ในขั้นตอนที่ 350

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้ (คอมเมนต์, ชื่อตัวแปร/paragraph, ข้อความใน
> `DISPLAY`/string literal ทุกชนิด) เป็นภาษาอังกฤษ/ASCII ล้วน เพราะขีดจำกัดคอลัมน์ 72 ของ Fixed-Format
> COBOL นับเป็นไบต์ไม่ใช่ตัวอักษร ตามที่อธิบายไว้ใน `docs/COURSE-OUTLINE.md` โค้ดทุกตัวอย่างในนี้ผ่านการ
> คอมไพล์และรันจริงด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง

---

## ขั้นตอนที่ 341: ภาพรวมโปรเจกต์, การออกแบบระบบ, ผังโครงสร้างโปรแกรม, และ Copybook ของ Record

### เป้าหมายของโปรเจกต์

เช่นเดียวกับ Part 015 เราเริ่มจากการกำหนดความต้องการ (Requirements) ก่อนลงมือเขียนโค้ด

| ข้อ | ความต้องการ |
|---|---|
| 1 | เก็บข้อมูลสินค้าแบบถาวร (persist) ด้วยไฟล์แบบ Indexed คีย์คือรหัสสินค้า (product code) |
| 2 | แต่ละสินค้าต้องมี: รหัส, ชื่อ, จำนวนคงเหลือ, ราคาต่อหน่วย, จุดสั่งซื้อ (reorder level) |
| 3 | เพิ่มสินค้าใหม่ได้ และต้องปฏิเสธถ้ารหัสซ้ำกับสินค้าที่มีอยู่แล้ว |
| 4 | รับสินค้าเข้าคลัง (เพิ่มจำนวน) และเบิกสินค้าออก (ลดจำนวน) โดยห้ามให้จำนวนคงเหลือติดลบเด็ดขาด |
| 5 | ค้นหาสินค้าด้วยรหัสได้โดยตรง ไม่ต้องอ่านทีละ record จากต้นไฟล์ |
| 6 | พิมพ์รายงานสินค้าทั้งหมด และรายงานแจ้งเตือนสินค้าที่จำนวนต่ำกว่าจุดสั่งซื้อ |
| 7 | ระบบต้องแบ่งเป็นโมดูลจริง (หลายไฟล์ต้นฉบับ) ไม่ใช่ไฟล์เดียวขนาดใหญ่แบบ Part 015 |

### ทำไมเฟส 2 ถึงออกแบบสถาปัตยกรรมต่างจาก Part 015

ใน Part 015 ขั้นตอนที่ 141 เราอธิบายไว้ชัดเจนว่า ณ จุดนั้นของหลักสูตรเรา **ยังไม่มี** `COPY` และ `CALL`
จึงต้องแบ่งโมดูลด้วย Paragraph ภายในไฟล์เดียว ตอนนี้สถานการณ์เปลี่ยนไปแล้ว: Part 031-032 สอน `CALL`
และการส่งพารามิเตอร์ ส่วน Part 033 สอน `COPY`/Copybook ทำให้เราสามารถออกแบบระบบตามหลัก
**Separation of Concerns** ได้อย่างแท้จริงแบบที่ระบบธุรกิจในอุตสาหกรรมทำกันจริง:

- **`invmain.cob`** รู้แค่ "มีเมนูอะไรบ้าง" และ "กดแล้วต้องเรียก subprogram ตัวไหน" — **ไม่รู้เลย**ว่าไฟล์ถูก
  เก็บอย่างไร หรือการตรวจสอบจำนวนติดลบทำงานอย่างไร
- **Subprogram แต่ละตัว** (`invadd`, `invrecv`, `invissue`, `invsrch`, `invlist`, `invlow`) รู้แค่หน้าที่
  ของตัวเองหน้าที่เดียว เปิด-ปิดไฟล์เอง จัดการ logic ของตัวเองเอง แล้วส่งผลลัพธ์กลับผ่านพารามิเตอร์
- **`INVREC.CPY`** คือ "สัญญา" (contract) กลางที่บอกว่า "record ของสินค้าหนึ่งตัวหน้าตาเป็นแบบนี้" — ทุก
  โปรแกรมที่แตะไฟล์ `INVMAST.DAT` ต้อง `COPY` ไฟล์นี้เข้าไป เพื่อรับประกันว่าทุกโปรแกรมมองเห็น record
  ในรูปแบบเดียวกันเป๊ะ ๆ (ถ้าวันหนึ่งต้องเพิ่มฟิลด์ใหม่ แก้ที่ copybook ไฟล์เดียว แล้ว compile ใหม่ทุกโปรแกรม
  ที่ COPY มันเข้าไป แทนที่จะต้องไล่แก้ทีละไฟล์)

### ผังโครงสร้างระบบ (Architecture Diagram)

```
                    +-----------------------+
                    |      invcreate        |   run ONCE by hand
                    |  (one-time setup job)  |   to set up the file
                    +-----------+-----------+
                                |
                                v
                       [ INVMAST.DAT ]  <-- INDEXED file
                                ^              RECORD KEY =
                                |              INV-PRODUCT-CODE
          +---------------------------------------------------+
          |                    invmain                        |
          |            (interactive menu driver)               |
          |     PERFORM WITH TEST AFTER UNTIL choice = 7        |
          +--+-------+--------+--------+--------+--------+-----+
             | CALL   | CALL   | CALL   | CALL   | CALL   | CALL
             v        v        v        v        v        v
          invadd   invrecv  invissue  invsrch  invlist   invlow
             |        |        |        |        |         |
             +--------+--------+--------+--------+---------+
                                |
                                v
                       [ INVMAST.DAT ]
              (each subprogram OPENs/CLOSEs the SAME
               physical file independently for its own job)

     All 8 source files COPY "INVREC.CPY" for one shared
     record layout - the single source of truth for the data shape.
```

จุดที่ควรสังเกตเป็นพิเศษ: **ลูกศรจาก `invmain` ไปยัง subprogram ทั้ง 6 ตัวเป็นแบบ `CALL`** (เรียกแล้วรอ
ผลลัพธ์กลับมา ตามที่เรียนใน Part 031) ในขณะที่ **ทุกกล่องที่แตะ `INVMAST.DAT` เปิด-ปิดไฟล์ของตัวเองอย่าง
อิสระ** ไม่มีการ "ส่งต่อไฟล์ที่เปิดค้างอยู่" ระหว่างโปรแกรม เพราะ COBOL มาตรฐาน**ไม่รองรับ**การส่ง File
Handle ผ่านพารามิเตอร์ `CALL` ข้าม compilation unit ได้โดยตรง (ต่างจากตัวแปรข้อมูลทั่วไปที่ส่งผ่าน
`LINKAGE SECTION` ได้ตามปกติ) นี่คือเหตุผลที่ subprogram แต่ละตัวต้องมี `SELECT`/`FD` ของตัวเอง แม้จะ
ชี้ไปที่ไฟล์ทางกายภาพเดียวกัน (`"INVMAST.DAT"`) ก็ตาม — เป็นแบบแผนที่พบได้ทั่วไปจริงในระบบ COBOL ที่แบ่ง
โมดูลลักษณะนี้

### Copybook: `INVREC.CPY`

```cobol
      *----------------------------------------------------------
      * INVREC.CPY
      * Shared record layout for the Inventory Management System
      * master file. COPY this into the FD of every program that
      * opens INVMAST.DAT, so all programs agree on one layout.
      *----------------------------------------------------------
       01  INVENTORY-RECORD.
           05  INV-PRODUCT-CODE      PIC X(6).
           05  INV-PRODUCT-NAME      PIC X(25).
           05  INV-QTY-ON-HAND       PIC 9(6).
           05  INV-UNIT-PRICE        PIC 9(6)V99.
           05  INV-REORDER-LEVEL     PIC 9(6).
```

| ฟิลด์ | PICTURE | เหตุผลของขนาด |
|---|---|---|
| `INV-PRODUCT-CODE` | `X(6)` | รหัสสินค้าแบบ `P0001` (ตัวอักษร+ตัวเลขผสมได้) และเป็น **RECORD KEY** ของไฟล์ Indexed |
| `INV-PRODUCT-NAME` | `X(25)` | ชื่อสินค้า ยาวพอสำหรับชื่อสินค้าทั่วไปโดยไม่ยาวเกินจำเป็น |
| `INV-QTY-ON-HAND` | `9(6)` | จำนวนคงเหลือ 0-999,999 หน่วย (ไม่มีเครื่องหมาย เพราะห้ามติดลบตามความต้องการข้อ 4) |
| `INV-UNIT-PRICE` | `9(6)V99` | ราคาต่อหน่วย สูงสุด 999,999.99 พร้อมทศนิยม 2 ตำแหน่งสำหรับสตางค์ |
| `INV-REORDER-LEVEL` | `9(6)` | จุดสั่งซื้อ/จุดแจ้งเตือน ใช้เปรียบเทียบกับจำนวนคงเหลือในรายงานแจ้งเตือน |

record หนึ่งตัวมีความยาวรวม 6+25+6+8+6 = **51 ไบต์** — ตัวเลขนี้ไม่ได้สำคัญมากสำหรับไฟล์ Indexed (ต่างจาก
ไฟล์ Sequential ที่ความกว้างคงที่มีผลโดยตรงต่อการอ่านทีละบรรทัดอย่างที่เรียนใน Part 023-025) แต่ยังคงเป็น
นิสัยที่ดีที่จะคำนวณไว้เสมอตอนออกแบบ

### ธรรมเนียมการตั้งชื่อที่ใช้ตลอดทั้งระบบ

เพื่อให้อ่านโค้ดทั้ง 8 ไฟล์ได้ง่ายและสอดคล้องกัน เราใช้ธรรมเนียมนี้ตลอดทั้งโปรเจกต์:

- **`INV-`** นำหน้าฟิลด์ใน `FD`/copybook (ข้อมูลระดับ record ของไฟล์)
- **`WS-`** นำหน้าตัวแปรใน `WORKING-STORAGE SECTION` ของแต่ละโปรแกรม (ข้อมูลภายในของโปรแกรมนั้นเอง)
- **`LS-`** นำหน้าพารามิเตอร์ใน `LINKAGE SECTION` ของ subprogram (ข้อมูลที่รับส่งกับผู้เรียก ทบทวนจาก
  Part 032)
- ทุก subprogram คืนค่าผลลัพธ์ผ่านพารามิเตอร์ **`LS-STATUS`** เป็นข้อความรหัสสถานะสั้น ๆ เช่น `"OK"`,
  `"DUPLICATE"`, `"NOTFOUND"`, `"INSUFFICIENT"`, `"FILE-ERR"` — ผู้เรียก (`invmain`) ใช้ `EVALUATE
  LS-STATUS` เพื่อตัดสินใจว่าจะแสดงข้อความอะไร แทนที่จะให้ subprogram แสดงผลเองโดยตรง (การแยกแบบนี้
  ทำให้ subprogram นำไปใช้ซ้ำกับ UI แบบอื่นได้ในอนาคต เช่น เว็บหรือ API ตามแนวคิดที่จะเรียนเต็มรูปแบบใน
  เฟส 5)

### ตารางสรุป Subprogram ทั้ง 6 ตัวและพารามิเตอร์

| Subprogram | พารามิเตอร์ที่รับเข้า | พารามิเตอร์ที่ส่งกลับ | ค่า LS-STATUS ที่เป็นไปได้ |
|---|---|---|---|
| `INVADD` | code, name, qty, price, reorder | status | `OK`, `DUPLICATE`, `FILE-ERR` |
| `INVRECV` | code, qty-change | new-qty, status | `OK`, `NOTFOUND`, `FILE-ERR` |
| `INVISSUE` | code, qty-change | new-qty, status | `OK`, `NOTFOUND`, `INSUFFICIENT`, `FILE-ERR` |
| `INVSRCH` | code | name, qty, price, reorder, status | `OK`, `NOTFOUND`, `FILE-ERR` |
| `INVLIST` | (ไม่มี) | record-count | (ไม่มี status แยก) |
| `INVLOW` | (ไม่มี) | low-count | (ไม่มี status แยก) |

### ข้อควรระวัง

- การให้แต่ละ subprogram เปิด-ปิดไฟล์ของตัวเองมี**ต้นทุนด้าน performance เล็กน้อย** (เปิด-ปิดไฟล์ซ้ำ
  หลายรอบต่อการทำธุรกรรมหนึ่งครั้งของผู้ใช้) แต่แลกมาด้วยความง่ายในการทำความเข้าใจและพัฒนาแยกส่วนได้จริง
  ซึ่งเหมาะสมกับขนาดของระบบระดับนี้ ระบบที่มีปริมาณธุรกรรมสูงมากในโลกจริงอาจเลือกใช้เทคนิคอื่น เช่น ให้
  CICS (เรียนในเฟส 4) จัดการเรื่องการเปิดไฟล์ค้างไว้ให้ทั้งระบบแทน
- อย่าลืมว่า `COPY "INVREC.CPY"` ต้องมีเครื่องหมายคำพูดล้อมชื่อไฟล์ และไฟล์ copybook ต้องอยู่ในไดเรกทอรี
  ที่ `cobc` มองเห็น (ค่าเริ่มต้นคือไดเรกทอรีปัจจุบัน ทบทวนจาก Part 033)

### แบบฝึกหัดที่ 341.1

**โจทย์**: จงอธิบายว่าทำไม `INVLIST` และ `INVLOW` จึงไม่มีพารามิเตอร์รับเข้า (input) เลย ในขณะที่
`INVADD`, `INVRECV`, `INVISSUE`, `INVSRCH` ทุกตัวต้องรับ `LS-PRODUCT-CODE` เป็นอย่างน้อย

**เฉลยแนวทาง**: `INVLIST` และ `INVLOW` เป็นรายงานที่ประมวลผล**สินค้าทุกรายการในไฟล์**อ่านตามลำดับตั้งแต่
ต้นจนจบ (`ACCESS MODE IS SEQUENTIAL`) จึงไม่จำเป็นต้องรู้ว่าจะเริ่มที่รหัสไหน ในขณะที่ `INVADD`,
`INVRECV`, `INVISSUE`, `INVSRCH` ทุกตัวทำงานกับ**สินค้ารายการเดียว**ที่ระบุด้วยรหัส (`ACCESS MODE IS
RANDOM`) จึงต้องรับรหัสสินค้าเป็นพารามิเตอร์เพื่อบอกว่าจะเข้าถึง record ไหนในไฟล์ Indexed นี่คือ
ความแตกต่างพื้นฐานระหว่างการเข้าถึงไฟล์แบบ Sequential (ทั้งหมดตามลำดับ) กับ Random (เจาะจงรายการเดียว
ด้วยคีย์) ที่เรียนไว้ใน Part 028

---

## ขั้นตอนที่ 342: การสร้างไฟล์หลักแบบ Indexed — โปรแกรม `invcreate.cob`

### แนวคิด

ก่อนจะมี subprogram ใดทำงานได้ ไฟล์ `INVMAST.DAT` ต้องถูก**สร้างขึ้นก่อน** ในรูปแบบ Indexed ที่ถูกต้อง
เราจึงเริ่มจากโปรแกรมที่ง่ายที่สุดในระบบทั้งหมด: `invcreate.cob` เป็นโปรแกรม **setup แบบรันครั้งเดียว**
(one-time job) เหมือนสคริปต์เตรียมฐานข้อมูลก่อนใช้งานระบบจริง หน้าที่เดียวของมันคือ `OPEN OUTPUT` ไฟล์
Indexed เพื่อสร้างไฟล์เปล่า ๆ แล้วปิดทันที

ทบทวนจาก Part 028: การประกาศไฟล์ Indexed ต้องระบุ `ORGANIZATION IS INDEXED` และ **`RECORD KEY IS`**
ชี้ไปที่ฟิลด์ที่จะใช้เป็นคีย์หลักในการค้นหา (ในที่นี้คือ `INV-PRODUCT-CODE` จาก copybook) การมี
`RECORD KEY` คือสิ่งที่ทำให้ COBOL สามารถ `READ`/`WRITE`/`REWRITE`/`DELETE` record ใดก็ได้โดยตรงด้วย
รหัสของมัน โดยไม่ต้องอ่านไล่ทีละ record จากต้นไฟล์เหมือนไฟล์ Sequential

### ซอร์สโค้ดฉบับสมบูรณ์: `invcreate.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INVCREATE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INVENTORY-FILE ASSIGN TO "INVMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS INV-PRODUCT-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  INVENTORY-FILE.
       COPY "INVREC.CPY".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.

       PROCEDURE DIVISION.
       0000-MAIN.
           OPEN OUTPUT INVENTORY-FILE
           DISPLAY "File status after create: " WS-FILE-STATUS
           CLOSE INVENTORY-FILE
           DISPLAY "Inventory master file INVMAST.DAT created."
           STOP RUN.
```

### คอมไพล์และรัน

```bash
cobc -x -o invcreate invcreate.cob
./invcreate
```

### ผลลัพธ์จริงที่ได้

```
File status after create: 00
Inventory master file INVMAST.DAT created.
```

### อธิบายโค้ดทีละส่วน

- `SELECT INVENTORY-FILE ASSIGN TO "INVMAST.DAT" ORGANIZATION IS INDEXED ... RECORD KEY IS
  INV-PRODUCT-CODE` คือการประกาศไฟล์ Indexed ทั้งหมดในที่เดียว — สังเกตว่า `RECORD KEY` อ้างถึงชื่อ
  ฟิลด์ที่มาจาก copybook โดยตรง (`INV-PRODUCT-CODE`) COBOL รู้จักชื่อนี้ได้เพราะ `COPY "INVREC.CPY"`
  ถูกวางไว้ใต้ `FD INVENTORY-FILE` แล้ว
- `FILE STATUS IS WS-FILE-STATUS` ทบทวนจาก Part 030 — ทุกครั้งที่มีการ `OPEN`/`READ`/`WRITE`/ฯลฯ กับ
  ไฟล์นี้ GnuCOBOL จะปรับปรุงค่า `WS-FILE-STATUS` เป็นรหัส 2 หลักเสมอ `"00"` หมายถึงสำเร็จ
- `OPEN OUTPUT INVENTORY-FILE` สำหรับไฟล์ Indexed มีความหมายเดียวกับไฟล์ Sequential ทุกประการ (ทบทวน
  จาก Part 025): **ลบไฟล์เดิมทั้งหมดถ้ามี แล้วสร้างไฟล์เปล่าใหม่** จึงต้องรันโปรแกรมนี้**เพียงครั้งเดียว**
  ตอนเริ่มต้นใช้งานระบบเท่านั้น (หรือเมื่อต้องการล้างข้อมูลทั้งหมดโดยตั้งใจ)
- โปรแกรมนี้ไม่มีการ `WRITE` record ใด ๆ เลย เพราะหน้าที่ของมันคือ**สร้างโครงสร้างไฟล์**ให้พร้อมสำหรับ
  `INVADD` (ขั้นตอนที่ 343) มาเติมข้อมูลจริงในภายหลัง

### ข้อควรระวัง

- **ถ้าคอมไพล์แล้วเจอ error ทำนองนี้**: `compiler is not configured to support ORGANIZATION INDEXED`
  หมายความว่า GnuCOBOL ที่ติดตั้งอยู่**ไม่ได้ถูก build มาพร้อมตัวจัดการไฟล์ Indexed** (ต้องมีไลบรารีอย่าง
  Berkeley DB หรือ VBISAM ตอน build คอมไพเลอร์) ทางแก้คือติดตั้ง GnuCOBOL ที่ build มาพร้อม `--with-db`
  (Berkeley DB) หรือ `--with-vbisam` — นี่ไม่ใช่บั๊กในโค้ดของเรา แต่เป็นเรื่องการตั้งค่าคอมไพเลอร์ที่ควร
  ตรวจสอบให้เรียบร้อยก่อนเริ่มงานกับไฟล์ Indexed จริงจัง (รายละเอียดการติดตั้งเต็มรูปแบบอยู่ใน Part 002
  และ Part 028)
- **ห้ามรัน `invcreate` ซ้ำหลังจากมีข้อมูลอยู่แล้ว** เพราะ `OPEN OUTPUT` จะล้างข้อมูลทั้งหมดทิ้งโดยไม่มี
  การถามยืนยันใด ๆ เลย (ดูแบบฝึกหัดท้ายขั้นตอนนี้สำหรับแนวทางป้องกัน)

### แบบฝึกหัดที่ 342.1

**โจทย์**: จงแก้ไข `invcreate.cob` ให้ตรวจสอบก่อนว่าไฟล์ `INVMAST.DAT` มีอยู่แล้วหรือไม่ (ลองเปิดด้วย
`OPEN INPUT` ก่อน) ถ้ามีอยู่แล้วให้แสดงข้อความเตือนแทนที่จะเขียนทับทันที

**เฉลยแนวทาง**:

```cobol
       PROCEDURE DIVISION.
       0000-MAIN.
           OPEN INPUT INVENTORY-FILE
           IF WS-FILE-STATUS = "00"
               CLOSE INVENTORY-FILE
               DISPLAY "WARNING: INVMAST.DAT already exists."
               DISPLAY "Setup aborted to avoid data loss."
               STOP RUN
           END-IF

           OPEN OUTPUT INVENTORY-FILE
           DISPLAY "File status after create: " WS-FILE-STATUS
           CLOSE INVENTORY-FILE
           DISPLAY "Inventory master file INVMAST.DAT created."
           STOP RUN.
```

เทคนิคนี้ใช้ `OPEN INPUT` เป็นตัว "ทดสอบ" ว่าไฟล์มีอยู่จริงหรือไม่ (ถ้าไม่มีไฟล์ `WS-FILE-STATUS` จะได้
ค่า `"35"` ตามตาราง File Status ที่เรียนใน Part 030) เป็นแบบแผนป้องกันความผิดพลาดที่ควรมีในระบบ setup
จริงทุกระบบ

---

## ขั้นตอนที่ 343: Subprogram แรก — เพิ่มสินค้าใหม่ด้วย `invadd.cob`

### แนวคิด

`invadd.cob` คือ subprogram ตัวแรกของระบบ ทำหน้าที่เดียว: **รับข้อมูลสินค้าใหม่มาเขียนลงไฟล์** พร้อม
ตรวจสอบว่ารหัสสินค้าซ้ำหรือไม่ นี่คือจุดที่เราใช้ `WRITE ... INVALID KEY` ซึ่งเป็นกลไกมาตรฐานของไฟล์
Indexed สำหรับตรวจจับ **primary key ซ้ำ** (ทบทวนจาก Part 028): เมื่อ `WRITE` record ที่มีค่า RECORD KEY
ตรงกับ record ที่มีอยู่แล้วในไฟล์ COBOL จะไม่เขียนทับ แต่จะกระโดดไปทำ branch `INVALID KEY` แทน

สังเกตโครงสร้าง `LINKAGE SECTION` — นี่คือการนำความรู้จาก Part 031-032 มาใช้เต็มรูปแบบ: พารามิเตอร์ทุกตัว
ถูกส่งแบบ **`BY REFERENCE`** (ค่า default ของ `CALL ... USING` เมื่อไม่ระบุอย่างอื่น) หมายความว่า
`invadd` เขียนค่ากลับเข้าไปในตัวแปรของผู้เรียก (`invmain`) ได้โดยตรงผ่าน `LS-STATUS`

### ซอร์สโค้ดฉบับสมบูรณ์: `invadd.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INVADD.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INVENTORY-FILE ASSIGN TO "INVMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS INV-PRODUCT-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  INVENTORY-FILE.
       COPY "INVREC.CPY".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.

       LINKAGE SECTION.
       01  LS-PRODUCT-CODE          PIC X(6).
       01  LS-PRODUCT-NAME          PIC X(25).
       01  LS-QTY-ON-HAND           PIC 9(6).
       01  LS-UNIT-PRICE            PIC 9(6)V99.
       01  LS-REORDER-LEVEL         PIC 9(6).
       01  LS-STATUS                PIC X(12).

       PROCEDURE DIVISION USING LS-PRODUCT-CODE LS-PRODUCT-NAME
               LS-QTY-ON-HAND LS-UNIT-PRICE LS-REORDER-LEVEL
               LS-STATUS.
       0000-MAIN.
           MOVE SPACES TO LS-STATUS
           OPEN I-O INVENTORY-FILE
           IF WS-FILE-STATUS NOT = "00"
               MOVE "FILE-ERR" TO LS-STATUS
               GOBACK
           END-IF

           MOVE LS-PRODUCT-CODE  TO INV-PRODUCT-CODE
           MOVE LS-PRODUCT-NAME  TO INV-PRODUCT-NAME
           MOVE LS-QTY-ON-HAND   TO INV-QTY-ON-HAND
           MOVE LS-UNIT-PRICE    TO INV-UNIT-PRICE
           MOVE LS-REORDER-LEVEL TO INV-REORDER-LEVEL

           WRITE INVENTORY-RECORD
               INVALID KEY
                   MOVE "DUPLICATE" TO LS-STATUS
               NOT INVALID KEY
                   MOVE "OK" TO LS-STATUS
           END-WRITE

           CLOSE INVENTORY-FILE
           GOBACK.
```

### ทดสอบแบบแยกส่วน (Unit Test) ด้วยโปรแกรมทดสอบเล็ก ๆ

ก่อนที่เมนูหลัก (`invmain`) จะพร้อมใน ขั้นตอนที่ 349 เราทดสอบ `invadd` แบบแยกส่วนได้ทันทีด้วยโปรแกรม
ทดสอบสั้น ๆ ที่ `CALL` มันโดยตรงพร้อมค่าคงที่ (literal) — เทคนิคนี้เรียกว่า **Unit Test แบบง่าย** มีประโยชน์
มากในการยืนยันว่า subprogram ทำงานถูกต้องก่อนเชื่อมเข้ากับ UI จริง:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TESTADD.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CODE       PIC X(6).
       01  WS-NAME       PIC X(25).
       01  WS-QTY        PIC 9(6).
       01  WS-PRICE      PIC 9(6)V99.
       01  WS-REORDER    PIC 9(6).
       01  WS-STATUS     PIC X(12).

       PROCEDURE DIVISION.
           MOVE "P0001" TO WS-CODE
           MOVE "USB Flash Drive 32GB" TO WS-NAME
           MOVE 50 TO WS-QTY
           MOVE 250.00 TO WS-PRICE
           MOVE 20 TO WS-REORDER
           CALL "INVADD" USING WS-CODE WS-NAME WS-QTY WS-PRICE
               WS-REORDER WS-STATUS
           DISPLAY "Add " WS-CODE " -> status: " WS-STATUS

      *> Try adding the SAME code again on purpose (duplicate key).
           CALL "INVADD" USING WS-CODE WS-NAME WS-QTY WS-PRICE
               WS-REORDER WS-STATUS
           DISPLAY "Add " WS-CODE " again -> status: " WS-STATUS
           STOP RUN.
```

### คอมไพล์และรัน

```bash
./invcreate
cobc -x -o test_add test_add.cob invadd.cob
./test_add
```

สังเกตว่าคำสั่ง `cobc` ระบุ**ไฟล์ต้นฉบับสองไฟล์**พร้อมกัน (`test_add.cob` และ `invadd.cob`) — GnuCOBOL
คอมไพล์แต่ละไฟล์แยกกันแล้ว **link ให้เป็นโปรแกรมเดียว (static link)** โดยอัตโนมัติ นี่คือวิธีมาตรฐานที่จะ
ใช้ประกอบระบบทั้งหมดเข้าด้วยกันในขั้นตอนที่ 349

### ผลลัพธ์จริงที่ได้

```
Add P0001  -> status: OK
Add P0001  again -> status: DUPLICATE
```

(หมายเหตุ: มีช่องว่างต่อท้ายค่าที่แสดงเล็กน้อยเพราะ `WS-CODE` เป็น `PIC X(6)` และ `WS-STATUS` เป็น
`PIC X(12)` ที่ไม่เต็มความกว้างเสมอ ซึ่งเป็นพฤติกรรมปกติของฟิลด์ตัวอักษรที่เรียนมาตั้งแต่ Part 006)

### อธิบายโค้ดทีละส่วน

- `OPEN I-O INVENTORY-FILE` — ใช้โหมด `I-O` (ทบทวนจาก Part 025) เพราะไฟล์**มีอยู่แล้ว**จาก `invcreate`
  และเราต้องการทั้งอ่าน (เพื่อให้ COBOL ตรวจสอบคีย์ซ้ำได้) และเขียนในการเปิดครั้งเดียว
- `WRITE INVENTORY-RECORD INVALID KEY ... NOT INVALID KEY ...` คือรูปแบบมาตรฐานสำหรับป้องกัน primary
  key ซ้ำในไฟล์ Indexed — ไม่ต้อง `READ` ตรวจสอบก่อนเองด้วยมือเลย ปล่อยให้ COBOL runtime จัดการให้
  ทั้งหมดผ่านกลไก `INVALID KEY`
- `GOBACK` (แทน `STOP RUN`) คือคำสั่งที่ถูกต้องสำหรับ subprogram — ทบทวนจาก Part 031: `STOP RUN` จะ
  จบการทำงานของ**ทั้งโปรแกรม**ทันที (รวมถึงโปรแกรมที่เรียกมันด้วย!) ในขณะที่ `GOBACK` คืนการควบคุมกลับไป
  ยังผู้เรียกเท่านั้น subprogram ทุกตัวในระบบนี้ต้องใช้ `GOBACK` ไม่ใช่ `STOP RUN`
- `LS-STATUS PIC X(12)` — ทำไมเลือก 12 ตัวอักษรพอดี? เพราะค่าที่ยาวที่สุดที่ subprogram ใดในระบบนี้
  จะส่งกลับคือ `"INSUFFICIENT"` (12 ตัวอักษร) จาก `INVISSUE` ที่จะสร้างในขั้นตอนที่ 345 — เราจงใจออกแบบ
  ขนาดให้พอดีกับค่าที่ยาวที่สุดตั้งแต่ต้น (เรื่องราวจริงว่าทำไมเรารู้ตัวเลข 12 นี้แม่นยำ จะเล่าในขั้นตอนที่ 345)

### ข้อควรระวัง

- `MOVE SPACES TO LS-STATUS` ที่ต้นโปรแกรมคือการ "เคลียร์" ค่าเก่าก่อนเสมอ เป็นนิสัยที่ดีสำหรับ
  พารามิเตอร์ที่ใช้ส่งค่ากลับ (output parameter) แม้ในเคสนี้ทุก path จะ MOVE ค่าใหม่ทับอยู่แล้วก็ตาม
  แต่การเคลียร์ไว้ก่อนช่วยป้องกันข้อผิดพลาดหากมีการเพิ่ม path ใหม่ในอนาคตแล้วลืม set ค่า
- ฟิลด์ `LS-QTY-ON-HAND` เป็น `PIC 9(6)` **ไม่มีเครื่องหมาย** (unsigned) ซึ่งเหมาะกับความต้องการที่ว่า
  จำนวนสินค้าห้ามติดลบ แต่ก็หมายความว่า**ตัวแปรนี้ไม่สามารถเก็บค่าติดลบได้แม้จะพยายามด้วยคำสั่งอื่นก็ตาม**
  (ทบทวนกฎการปัดเศษ/ตัดหลักจาก Part 006 และ Part 008)

### แบบฝึกหัดที่ 343.1

**โจทย์**: จงแก้ไข `invadd.cob` ให้ปฏิเสธการเพิ่มสินค้าที่ `LS-PRODUCT-CODE` เป็นค่าว่าง (spaces ล้วน)
โดยส่งค่า `LS-STATUS` เป็น `"INVALID"` และไม่พยายามเขียนไฟล์เลย

**เฉลยแนวทาง**:

```cobol
       0000-MAIN.
           MOVE SPACES TO LS-STATUS
           IF LS-PRODUCT-CODE = SPACES
               MOVE "INVALID" TO LS-STATUS
               GOBACK
           END-IF

           OPEN I-O INVENTORY-FILE
           ...
```

การตรวจสอบนี้ควรทำ**ก่อน**เปิดไฟล์ด้วยซ้ำ เพื่อหลีกเลี่ยงการเปิด-ปิดไฟล์โดยไม่จำเป็นเมื่อข้อมูลนำเข้า
ผิดตั้งแต่ต้น — เป็นหลักการ "fail fast" ที่ดีสำหรับการเขียนโปรแกรมทุกภาษา

---

## ขั้นตอนที่ 344: รับสินค้าเข้าคลังด้วย `invrecv.cob`

### แนวคิด

`invrecv.cob` (Receive) ทำหน้าที่**เพิ่มจำนวนสินค้า**เมื่อมีของเข้าคลังใหม่ (เช่น รับสินค้าจาก supplier)
นี่คือ subprogram แรกที่ต้อง**อ่าน record เดิมก่อน แก้ไข แล้วเขียนกลับ** ซึ่งใช้คำสั่ง **`REWRITE`**
(ทบทวนเต็มรูปแบบจาก Part 025 ขั้นตอนที่ 245-246) ต่างจาก `WRITE` ที่ใช้สร้าง record ใหม่

รูปแบบการทำงานคือ: `READ` ด้วยคีย์ (`ACCESS MODE IS RANDOM`) → ถ้าเจอ ก็ `ADD` จำนวนที่รับเข้าไปยัง
`INV-QTY-ON-HAND` ในหน่วยความจำ → `REWRITE` เพื่อบันทึกกลับลงไฟล์

### ซอร์สโค้ดฉบับสมบูรณ์: `invrecv.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INVRECV.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INVENTORY-FILE ASSIGN TO "INVMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS INV-PRODUCT-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  INVENTORY-FILE.
       COPY "INVREC.CPY".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.

       LINKAGE SECTION.
       01  LS-PRODUCT-CODE          PIC X(6).
       01  LS-QTY-CHANGE            PIC 9(6).
       01  LS-NEW-QTY               PIC 9(6).
       01  LS-STATUS                PIC X(12).

       PROCEDURE DIVISION USING LS-PRODUCT-CODE LS-QTY-CHANGE
               LS-NEW-QTY LS-STATUS.
       0000-MAIN.
           MOVE SPACES TO LS-STATUS
           MOVE 0 TO LS-NEW-QTY
           OPEN I-O INVENTORY-FILE
           IF WS-FILE-STATUS NOT = "00"
               MOVE "FILE-ERR" TO LS-STATUS
               GOBACK
           END-IF

           MOVE LS-PRODUCT-CODE TO INV-PRODUCT-CODE
           READ INVENTORY-FILE
               INVALID KEY
                   MOVE "NOTFOUND" TO LS-STATUS
               NOT INVALID KEY
                   ADD LS-QTY-CHANGE TO INV-QTY-ON-HAND
                   REWRITE INVENTORY-RECORD
                   MOVE INV-QTY-ON-HAND TO LS-NEW-QTY
                   MOVE "OK" TO LS-STATUS
           END-READ

           CLOSE INVENTORY-FILE
           GOBACK.
```

### ทดสอบแบบแยกส่วน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TESTRECVISSUE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CODE       PIC X(6).
       01  WS-CHANGE     PIC 9(6).
       01  WS-NEW-QTY    PIC 9(6).
       01  WS-STATUS     PIC X(12).

       PROCEDURE DIVISION.
           MOVE "P0001" TO WS-CODE
           MOVE 30 TO WS-CHANGE
           CALL "INVRECV" USING WS-CODE WS-CHANGE WS-NEW-QTY
               WS-STATUS
           DISPLAY "Receive 30 on P0001 -> status: " WS-STATUS
               " new qty: " WS-NEW-QTY

           MOVE "P9999" TO WS-CODE
           MOVE 5 TO WS-CHANGE
           CALL "INVRECV" USING WS-CODE WS-CHANGE WS-NEW-QTY
               WS-STATUS
           DISPLAY "Receive on unknown P9999 -> status: " WS-STATUS
           STOP RUN.
```

(โปรแกรมทดสอบนี้จะขยายเพิ่ม `INVISSUE` เข้าไปด้วยในขั้นตอนที่ 345 เป็น `test_recv_issue.cob` ฉบับ
สมบูรณ์)

### คอมไพล์และรัน

```bash
cobc -x -o test_recv test_recv.cob invrecv.cob
./test_recv
```

### ผลลัพธ์จริงที่ได้ (สืบเนื่องจาก P0001 ที่มีจำนวน 50 จากขั้นตอนที่ 343)

```
Receive 30 on P0001 -> status: OK           new qty: 000080
Receive on unknown P9999 -> status: NOTFOUND
```

### อธิบายโค้ดทีละส่วน

- `READ INVENTORY-FILE INVALID KEY ... NOT INVALID KEY ...` ภายใต้ `ACCESS MODE IS RANDOM` คือการอ่าน
  record ด้วยคีย์โดยตรง — ต้อง `MOVE` ค่าคีย์ที่ต้องการเข้า `INV-PRODUCT-CODE` **ก่อน** `READ` เสมอ
  (ทบทวนจาก Part 028) ถ้าไม่พบคีย์นั้นในไฟล์ COBOL จะเข้า branch `INVALID KEY` แทนที่จะ error
- `ADD LS-QTY-CHANGE TO INV-QTY-ON-HAND` แก้ไขค่าใน**บัฟเฟอร์ในหน่วยความจำ**ของ record ที่เพิ่งอ่านมา
  ยังไม่ได้บันทึกลงไฟล์จริงจนกว่าจะสั่ง `REWRITE`
- `REWRITE INVENTORY-RECORD` ต้องเรียก**หลัง**จาก `READ` record นั้นสำเร็จเสมอ และ**ห้ามเปลี่ยนค่าคีย์**
  (`INV-PRODUCT-CODE`) ก่อน `REWRITE` เด็ดขาด (กฎนี้ทบทวนจาก Part 025 ขั้นตอนที่ 245) โค้ดของเราปลอดภัย
  เพราะแก้แค่ `INV-QTY-ON-HAND` เท่านั้น ไม่แตะ `INV-PRODUCT-CODE`

### ข้อควรระวัง

- **จุดที่ต้องระวังมากที่สุดของขั้นตอนนี้**: `ADD LS-QTY-CHANGE TO INV-QTY-ON-HAND` ไม่มี `ON SIZE
  ERROR` กำกับไว้ หาก `INV-QTY-ON-HAND` (สูงสุด 999,999) บวกแล้วเกินขีดจำกัดของ `PIC 9(6)` COBOL จะ
  **ตัดหลักสูงทิ้งแบบเงียบ ๆ** ตามกฎที่เรียนมาใน Part 009 ขั้นตอนที่ 87 — ในระบบสินค้าคงคลังจริงที่มี
  จำนวนสต๊อกมหาศาล นี่คือความเสี่ยงที่ต้องป้องกันด้วย `ON SIZE ERROR` เสมอ (ดูแบบฝึกหัดท้ายขั้นตอนนี้)
- ต้องรัน `invcreate` และมีสินค้าอยู่ในไฟล์แล้ว (จาก `invadd`) ก่อนทดสอบ `invrecv` เสมอ มิฉะนั้นจะได้
  `NOTFOUND` ทุกครั้ง

### แบบฝึกหัดที่ 344.1

**โจทย์**: จงเพิ่ม `ON SIZE ERROR` ให้กับคำสั่ง `ADD` ใน `invrecv.cob` เพื่อป้องกันจำนวนล้นขอบเขต โดยถ้า
เกิด size error ให้ส่ง `LS-STATUS` เป็น `"OVERFLOW"` และไม่บันทึกการเปลี่ยนแปลงใด ๆ

**เฉลยแนวทาง**:

```cobol
               NOT INVALID KEY
                   ADD LS-QTY-CHANGE TO INV-QTY-ON-HAND
                       ON SIZE ERROR
                           MOVE "OVERFLOW" TO LS-STATUS
                       NOT ON SIZE ERROR
                           REWRITE INVENTORY-RECORD
                           MOVE INV-QTY-ON-HAND TO LS-NEW-QTY
                           MOVE "OK" TO LS-STATUS
                   END-ADD
```

เมื่อเกิด `ON SIZE ERROR` ค่า `INV-QTY-ON-HAND` จะ**ไม่ถูกแก้ไข**ตามกฎที่เรียนใน Part 009 (SIZE ERROR
ทำให้ปลายทางไม่เปลี่ยนแปลง) จึงไม่มีการ `REWRITE` เกิดขึ้นเลยในกรณีนี้ ปลอดภัยกว่าเดิมมาก

---

## ขั้นตอนที่ 345: เบิกสินค้าออกจากคลังด้วย `invissue.cob` — และบั๊กจริงเรื่องความยาวฟิลด์

### แนวคิด

`invissue.cob` (Issue) ทำหน้าที่ตรงข้ามกับ `invrecv.cob`: **ลดจำนวนสินค้า**เมื่อมีการเบิกออกไปใช้/ขาย
นี่คือ subprogram ที่สำคัญที่สุดในแง่ความถูกต้องของข้อมูล เพราะต้อง**ป้องกันไม่ให้จำนวนคงเหลือติดลบ**
ตามความต้องการข้อ 4 ของโปรเจกต์ — โครงสร้างเหมือน `invrecv.cob` เกือบทุกประการ ต่างกันแค่ตรวจสอบเงื่อนไข
ก่อน `SUBTRACT` เสมอ

### ซอร์สโค้ดฉบับสมบูรณ์: `invissue.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INVISSUE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INVENTORY-FILE ASSIGN TO "INVMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS INV-PRODUCT-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  INVENTORY-FILE.
       COPY "INVREC.CPY".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.

       LINKAGE SECTION.
       01  LS-PRODUCT-CODE          PIC X(6).
       01  LS-QTY-CHANGE            PIC 9(6).
       01  LS-NEW-QTY               PIC 9(6).
       01  LS-STATUS                PIC X(12).

       PROCEDURE DIVISION USING LS-PRODUCT-CODE LS-QTY-CHANGE
               LS-NEW-QTY LS-STATUS.
       0000-MAIN.
           MOVE SPACES TO LS-STATUS
           MOVE 0 TO LS-NEW-QTY
           OPEN I-O INVENTORY-FILE
           IF WS-FILE-STATUS NOT = "00"
               MOVE "FILE-ERR" TO LS-STATUS
               GOBACK
           END-IF

           MOVE LS-PRODUCT-CODE TO INV-PRODUCT-CODE
           READ INVENTORY-FILE
               INVALID KEY
                   MOVE "NOTFOUND" TO LS-STATUS
               NOT INVALID KEY
                   IF LS-QTY-CHANGE > INV-QTY-ON-HAND
                       MOVE "INSUFFICIENT" TO LS-STATUS
                       MOVE INV-QTY-ON-HAND TO LS-NEW-QTY
                   ELSE
                       SUBTRACT LS-QTY-CHANGE FROM INV-QTY-ON-HAND
                       REWRITE INVENTORY-RECORD
                       MOVE INV-QTY-ON-HAND TO LS-NEW-QTY
                       MOVE "OK" TO LS-STATUS
                   END-IF
           END-READ

           CLOSE INVENTORY-FILE
           GOBACK.
```

### ทดสอบแบบแยกส่วน: `test_recv_issue.cob` (ฉบับสมบูรณ์รวม `INVRECV` และ `INVISSUE`)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TESTRECVISSUE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CODE       PIC X(6).
       01  WS-CHANGE     PIC 9(6).
       01  WS-NEW-QTY    PIC 9(6).
       01  WS-STATUS     PIC X(12).

       PROCEDURE DIVISION.
           MOVE "P0001" TO WS-CODE
           MOVE 30 TO WS-CHANGE
           CALL "INVRECV" USING WS-CODE WS-CHANGE WS-NEW-QTY
               WS-STATUS
           DISPLAY "Receive 30 on P0001 -> status: " WS-STATUS
               " new qty: " WS-NEW-QTY

           MOVE "P9999" TO WS-CODE
           MOVE 5 TO WS-CHANGE
           CALL "INVRECV" USING WS-CODE WS-CHANGE WS-NEW-QTY
               WS-STATUS
           DISPLAY "Receive on unknown P9999 -> status: " WS-STATUS

           MOVE "P0001" TO WS-CODE
           MOVE 200 TO WS-CHANGE
           CALL "INVISSUE" USING WS-CODE WS-CHANGE WS-NEW-QTY
               WS-STATUS
           DISPLAY "Issue 200 from P0001 -> status: " WS-STATUS
               " on hand: " WS-NEW-QTY

           MOVE 30 TO WS-CHANGE
           CALL "INVISSUE" USING WS-CODE WS-CHANGE WS-NEW-QTY
               WS-STATUS
           DISPLAY "Issue 30 from P0001 -> status: " WS-STATUS
               " new qty: " WS-NEW-QTY
           STOP RUN.
```

### คอมไพล์และรัน

```bash
cobc -x -o test_recv_issue test_recv_issue.cob invrecv.cob invissue.cob
./test_recv_issue
```

### ผลลัพธ์จริงที่ได้

```
Receive 30 on P0001 -> status: OK           new qty: 000080
Receive on unknown P9999 -> status: NOTFOUND
Issue 200 from P0001 -> status: INSUFFICIENT on hand: 000080
Issue 30 from P0001 -> status: OK           new qty: 000050
```

สังเกตว่าการเบิก 200 หน่วยจากสินค้าที่มีอยู่แค่ 80 หน่วยถูก**ปฏิเสธ** (`INSUFFICIENT`) และจำนวนคงเหลือ
ยังคง 80 หน่วยเหมือนเดิม (`on hand: 000080`) ไม่มีการเขียนไฟล์ผิดพลาดเกิดขึ้นเลย — ตรงตามความต้องการ
ข้อ 4 ของโปรเจกต์ทุกประการ ส่วนการเบิก 30 หน่วยครั้งถัดมาสำเร็จตามปกติ เหลือ 50 หน่วย

### เรื่องจริงจากการพัฒนา: บั๊กเรื่องความยาวฟิลด์ `LS-STATUS` ที่พบระหว่างทดสอบ

ระหว่างเตรียมเนื้อหาบทนี้ เราเขียน `LS-STATUS` เป็น `PIC X(10)` ในตอนแรก (คิดว่า 10 ตัวอักษรน่าจะพอสำหรับ
คำว่า `"DUPLICATE"` ที่มี 9 ตัว) แต่พอทดสอบ `invissue` จริงกลับได้ผลลัพธ์ที่ผิดเพี้ยนดังนี้:

```
ERROR: not enough stock. On hand: ...
(WS-STATUS แสดงเป็น "INSUFFICIE" แทนที่จะเป็น "INSUFFICIENT")
```

สาเหตุคือคำว่า **`"INSUFFICIENT"` มีความยาว 12 ตัวอักษร** ยาวกว่า `PIC X(10)` ที่เตรียมไว้ 2 ตัว เมื่อ
`MOVE "INSUFFICIENT" TO LS-STATUS` กับฟิลด์ปลายทางที่สั้นกว่า ตามกฎ `MOVE` ที่เรียนมาใน Part 008
ขั้นตอนที่ 72 (ตัดหลักส่วนเกินทิ้งแบบเงียบ ๆ โดยไม่มี error หรือ warning ใด ๆ) ค่าที่ได้จึงกลายเป็น
`"INSUFFICIE"` (ถูกตัด 2 ตัวท้ายทิ้ง) ทำให้ `EVALUATE WS-STATUS WHEN "INSUFFICIENT"` ใน `invmain` ไม่
ตรงกับเงื่อนไขที่ตั้งใจไว้เลย! วิธีแก้คือ**ขยาย `LS-STATUS`/`WS-STATUS` เป็น `PIC X(12)` ในทุกไฟล์ที่
เกี่ยวข้อง** ให้พอดีกับคำที่ยาวที่สุดที่ระบบทั้งหมดจะใช้งาน ซึ่งเป็นค่าที่ใช้อยู่ในซอร์สโค้ดที่แสดงใน
บทนี้ทั้งหมดแล้ว (ทั้ง `invadd.cob`, `invrecv.cob`, `invissue.cob`, `invsrch.cob`, และ `invmain.cob`)

นี่คือตัวอย่างจริงว่าทำไม**การคอมไพล์และรันทดสอบจริงทุกครั้งจึงสำคัญกว่าการอ่านโค้ดด้วยตาเปล่า** — โค้ดที่
ดูถูกต้อง 100% ตอนเขียน (ไม่มี syntax error ไม่มี compile warning) ยังสามารถมีบั๊กเชิงตรรกะที่ร้ายแรงซ่อน
อยู่ได้ และบั๊กประเภทนี้ (การตัดข้อความแบบเงียบ ๆ จาก `MOVE`) เป็นหนึ่งในบั๊กที่พบบ่อยและอันตรายที่สุดใน
โค้ด COBOL จริง เพราะมันไม่ทำให้โปรแกรม crash แต่ทำให้ตรรกะทำงานผิดอย่างเงียบ ๆ

### อธิบายโค้ดทีละส่วน

- `IF LS-QTY-CHANGE > INV-QTY-ON-HAND` คือหัวใจของการป้องกันติดลบ — ตรวจสอบ**ก่อน** `SUBTRACT` เสมอ
  แทนที่จะปล่อยให้ลบแล้วค่อยแก้ (เพราะ `INV-QTY-ON-HAND` เป็น `PIC 9(6)` แบบไม่มีเครื่องหมาย การลบจนติดลบ
  จะทำให้เกิดพฤติกรรมที่ไม่คาดคิด ไม่ใช่แค่ error ที่ตรวจจับได้ง่าย)
- เมื่อเงื่อนไข `INSUFFICIENT` เป็นจริง เรายังคง `MOVE INV-QTY-ON-HAND TO LS-NEW-QTY` (ส่งจำนวนคงเหลือ
  ปัจจุบันกลับไปด้วย) แม้จะไม่ได้แก้ไขอะไรเลยก็ตาม เพื่อให้ผู้เรียกสามารถแสดงข้อความที่เป็นประโยชน์ เช่น
  "มีอยู่แค่ 80 หน่วย" แทนที่จะบอกแค่ว่า "ไม่พอ" เฉย ๆ

### ข้อควรระวัง

- ตรวจสอบให้แน่ใจว่า `LS-STATUS` และ `WS-STATUS` ใน**ทุกไฟล์**ของระบบมีขนาด `PIC X(12)` ตรงกัน เพราะ
  พารามิเตอร์แบบ `BY REFERENCE` (ทบทวนจาก Part 032) ไม่มีการตรวจสอบขนาดข้ามกันระหว่างผู้เรียกกับ
  subprogram โดยคอมไพเลอร์ — ถ้าผู้เรียกประกาศฟิลด์เล็กกว่าที่ subprogram คาดหวัง อาจเกิดการเขียนทับ
  หน่วยความจำที่ไม่ได้ตั้งใจ (out-of-bounds) ซึ่งอันตรายกว่าการตัดข้อความธรรมดามาก
- อย่าลืมว่าฟิลด์ตัวเลขในระบบนี้ (`PIC 9(6)`) ไม่มีเครื่องหมาย — การตรวจสอบ `IF LS-QTY-CHANGE >
  INV-QTY-ON-HAND` ทำงานถูกต้องเพราะเปรียบเทียบตัวเลขสองค่าที่ไม่ติดลบทั้งคู่เสมอ

### แบบฝึกหัดที่ 345.1

**โจทย์**: จงขยาย `invissue.cob` ให้ส่งค่าพารามิเตอร์เพิ่มเติม `LS-BELOW-REORDER` (`PIC X` เป็น `"Y"`
หรือ `"N"`) เพื่อบอกผู้เรียกว่าหลังจากเบิกสำเร็จแล้ว จำนวนคงเหลือตกลงมาต่ำกว่าจุดสั่งซื้อหรือไม่ (เป็นการ
เตือนล่วงหน้า ก่อนที่จะต้องรอรัน `invlow` แยกต่างหาก)

**เฉลยแนวทาง**:

```cobol
       LINKAGE SECTION.
       01  LS-PRODUCT-CODE          PIC X(6).
       01  LS-QTY-CHANGE            PIC 9(6).
       01  LS-NEW-QTY               PIC 9(6).
       01  LS-STATUS                PIC X(12).
       01  LS-BELOW-REORDER         PIC X.

       PROCEDURE DIVISION USING LS-PRODUCT-CODE LS-QTY-CHANGE
               LS-NEW-QTY LS-STATUS LS-BELOW-REORDER.
       0000-MAIN.
           MOVE SPACES TO LS-STATUS
           MOVE "N" TO LS-BELOW-REORDER
           ...
                   ELSE
                       SUBTRACT LS-QTY-CHANGE FROM INV-QTY-ON-HAND
                       REWRITE INVENTORY-RECORD
                       MOVE INV-QTY-ON-HAND TO LS-NEW-QTY
                       MOVE "OK" TO LS-STATUS
                       IF INV-QTY-ON-HAND < INV-REORDER-LEVEL
                           MOVE "Y" TO LS-BELOW-REORDER
                       END-IF
                   END-IF
```

ผู้เรียก (`invmain`) สามารถตรวจสอบ `LS-BELOW-REORDER` หลังเรียก `CALL "INVISSUE"` แล้วแสดงข้อความเตือน
เพิ่มเติมทันที เป็นการนำแนวคิดจากรายงานแจ้งเตือน (ขั้นตอนที่ 348) มาทำงานแบบ real-time ในจุดที่เกิด
เหตุการณ์จริง

---

## ขั้นตอนที่ 346: ค้นหาสินค้าด้วยรหัสด้วย `invsrch.cob`

### แนวคิด

`invsrch.cob` (Search) เป็น subprogram ที่ง่ายที่สุดในกลุ่มที่เข้าถึงข้อมูลรายตัว: เปิดไฟล์แบบ
**`OPEN INPUT`** เท่านั้น (อ่านอย่างเดียว ไม่มีการแก้ไข) แล้ว `READ` ด้วยคีย์ที่ได้รับมา จากนั้นส่งข้อมูล
ทั้งหมดของสินค้ากลับผ่านพารามิเตอร์ output

### ซอร์สโค้ดฉบับสมบูรณ์: `invsrch.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INVSRCH.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INVENTORY-FILE ASSIGN TO "INVMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS INV-PRODUCT-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  INVENTORY-FILE.
       COPY "INVREC.CPY".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.

       LINKAGE SECTION.
       01  LS-PRODUCT-CODE          PIC X(6).
       01  LS-PRODUCT-NAME          PIC X(25).
       01  LS-QTY-ON-HAND           PIC 9(6).
       01  LS-UNIT-PRICE            PIC 9(6)V99.
       01  LS-REORDER-LEVEL         PIC 9(6).
       01  LS-STATUS                PIC X(12).

       PROCEDURE DIVISION USING LS-PRODUCT-CODE LS-PRODUCT-NAME
               LS-QTY-ON-HAND LS-UNIT-PRICE LS-REORDER-LEVEL
               LS-STATUS.
       0000-MAIN.
           MOVE SPACES TO LS-STATUS
           OPEN INPUT INVENTORY-FILE
           IF WS-FILE-STATUS NOT = "00"
               MOVE "FILE-ERR" TO LS-STATUS
               GOBACK
           END-IF

           MOVE LS-PRODUCT-CODE TO INV-PRODUCT-CODE
           READ INVENTORY-FILE
               INVALID KEY
                   MOVE "NOTFOUND" TO LS-STATUS
               NOT INVALID KEY
                   MOVE INV-PRODUCT-NAME  TO LS-PRODUCT-NAME
                   MOVE INV-QTY-ON-HAND   TO LS-QTY-ON-HAND
                   MOVE INV-UNIT-PRICE    TO LS-UNIT-PRICE
                   MOVE INV-REORDER-LEVEL TO LS-REORDER-LEVEL
                   MOVE "OK" TO LS-STATUS
           END-READ

           CLOSE INVENTORY-FILE
           GOBACK.
```

### ทดสอบแบบแยกส่วน: `test_srch.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TESTSRCH.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CODE       PIC X(6).
       01  WS-NAME       PIC X(25).
       01  WS-QTY        PIC 9(6).
       01  WS-PRICE      PIC 9(6)V99.
       01  WS-REORDER    PIC 9(6).
       01  WS-STATUS     PIC X(12).

       PROCEDURE DIVISION.
           MOVE "P0001" TO WS-CODE
           CALL "INVSRCH" USING WS-CODE WS-NAME WS-QTY WS-PRICE
               WS-REORDER WS-STATUS
           DISPLAY "Search P0001 -> status: " WS-STATUS
           IF WS-STATUS = "OK"
               DISPLAY "  Name : " WS-NAME
               DISPLAY "  Qty  : " WS-QTY
               DISPLAY "  Price: " WS-PRICE
           END-IF

           MOVE "ZZZZZZ" TO WS-CODE
           CALL "INVSRCH" USING WS-CODE WS-NAME WS-QTY WS-PRICE
               WS-REORDER WS-STATUS
           DISPLAY "Search ZZZZZZ -> status: " WS-STATUS
           STOP RUN.
```

### คอมไพล์และรัน

```bash
cobc -x -o test_srch test_srch.cob invsrch.cob
./test_srch
```

### ผลลัพธ์จริงที่ได้

```
Search P0001 -> status: OK
  Name : USB Flash Drive 32GB
  Qty  : 000050
  Price: 000250.00
Search ZZZZZZ -> status: NOTFOUND
```

### อธิบายโค้ดทีละส่วน

- `OPEN INPUT` (แทน `OPEN I-O`) เพราะการค้นหาไม่มีการแก้ไขข้อมูลใด ๆ เลย — เลือกโหมดเปิดไฟล์ให้แคบ
  ที่สุดเท่าที่จำเป็นเสมอเป็นหลักปฏิบัติที่ดี (Principle of Least Privilege) ทบทวนแนวคิดจาก Part 025
  ขั้นตอนที่ 241 ที่ว่า "โหมดที่เปิดกำหนดว่าใช้คำสั่งอะไรได้บ้างอย่างเข้มงวด"
- สังเกตว่าโครงสร้าง `READ ... INVALID KEY ... NOT INVALID KEY` เหมือนกับ `invrecv.cob` และ
  `invissue.cob` ทุกประการ — ต่างกันแค่ `invsrch` ไม่มีการแก้ไขค่าใด ๆ ก่อนจบ (ไม่มี `REWRITE`) เพราะ
  เป็นการอ่านอย่างเดียว

### ข้อควรระวัง

- ระบบนี้รองรับการค้นหาด้วย**รหัสที่ตรงกันเป๊ะ**เท่านั้น (Exact Match) เพราะ `RECORD KEY` ของไฟล์
  Indexed ทำงานแบบนี้โดยธรรมชาติ ถ้าต้องการค้นหาแบบ "มีคำนี้อยู่ในชื่อสินค้า" (partial match) จะต้องอ่าน
  ไฟล์ทั้งหมดแบบ Sequential แล้วเทียบด้วย `INSPECT`/`UNSTRING` (ทบทวนจาก Part 019-021) ซึ่งช้ากว่ามาก
  และไม่ใช่จุดแข็งของไฟล์ Indexed

### แบบฝึกหัดที่ 346.1

**โจทย์**: จงเพิ่มพารามิเตอร์ output `LS-IS-LOW-STOCK` (`PIC X`) ให้กับ `invsrch.cob` เพื่อบอกทันทีว่า
สินค้าที่ค้นหาเจอนั้นมีจำนวนต่ำกว่าจุดสั่งซื้อหรือไม่ โดยไม่ต้องเรียก `invlow` แยกต่างหาก

**เฉลยแนวทาง**:

```cobol
                   MOVE "OK" TO LS-STATUS
                   IF INV-QTY-ON-HAND < INV-REORDER-LEVEL
                       MOVE "Y" TO LS-IS-LOW-STOCK
                   ELSE
                       MOVE "N" TO LS-IS-LOW-STOCK
                   END-IF
```

พร้อมเพิ่ม `01 LS-IS-LOW-STOCK PIC X.` ใน `LINKAGE SECTION` และเพิ่มเข้าไปในรายการพารามิเตอร์ทั้งที่
`PROCEDURE DIVISION USING` และทุกจุดที่ `CALL "INVSRCH"`

---

## ขั้นตอนที่ 347: รายงานสินค้าทั้งหมดด้วย `invlist.cob` — และบั๊กจริงเรื่อง WORKING-STORAGE ที่ค้างข้าม CALL

### แนวคิด

`invlist.cob` พิมพ์รายงานสินค้าทั้งหมดในไฟล์ เรียงตามรหัสสินค้าจากน้อยไปมาก (เพราะไฟล์ Indexed เมื่ออ่าน
ด้วย `ACCESS MODE IS SEQUENTIAL` จะอ่านตามลำดับ**ค่าคีย์**เสมอ ไม่ใช่ลำดับที่เขียนเข้าไป — ทบทวนจาก
Part 028) นี่คือ subprogram แรกที่ใช้ **flag ควบคุมการอ่านจนจบไฟล์** (`WS-EOF-FLAG`) ซึ่งทบทวนรูปแบบ
มาตรฐานจาก Part 025-026

### เรื่องจริงจากการพัฒนา: บั๊กที่พบตอนเรียก `INVLIST` ซ้ำสองครั้งในโปรแกรมเดียว

ก่อนจะแสดงซอร์สโค้ดฉบับที่ทำงานถูกต้อง ขอเล่าบั๊กจริงอีกตัวที่พบระหว่างทดสอบ (สำคัญพอที่จะเล่าก่อนเห็น
คำตอบ เพื่อให้เข้าใจว่าทำไมโค้ดจึงมีบรรทัดที่ดูเหมือนไม่จำเป็นบรรทัดหนึ่งอยู่): ตอนแรกเราเขียน `0000-MAIN`
โดยไม่มีการ `MOVE "N" TO WS-EOF-FLAG` ที่จุดเริ่มต้น (คิดว่า `VALUE "N"` ตอนประกาศตัวแปรก็เพียงพอแล้ว)
พอทดสอบเรียก `CALL "INVLIST"` **ครั้งแรก** ในโปรแกรมหนึ่ง ได้ผลลัพธ์ถูกต้องครบ 2 รายการตามที่คาดหวัง
แต่พอเรียก `CALL "INVLIST"` **ครั้งที่สอง** ในโปรแกรม**เดียวกัน** (จำลองสถานการณ์ที่ผู้ใช้เลือกเมนู "List
all products" สองครั้งติดกันในโปรแกรม `invmain`) กลับได้รายงานที่**ว่างเปล่าทันที** (`Total products
listed: 0000`) ทั้งที่ไฟล์ยังมีข้อมูลอยู่ครบ!

สาเหตุที่แท้จริงคือกฎสำคัญของ COBOL ที่มักถูกมองข้าม: **`WORKING-STORAGE SECTION` ของ subprogram จะคง
ค่าล่าสุดไว้ข้ามการ `CALL` แต่ละครั้งเสมอ (ไม่ถูก reset กลับไปเป็นค่า `VALUE` เดิมโดยอัตโนมัติ)** ยกเว้น
จะประกาศโปรแกรมนั้นเป็น `PROGRAM-ID. ... IS INITIAL PROGRAM` เท่านั้น ในเคสของเรา หลังจาก `CALL
"INVLIST"` ครั้งแรกอ่านจนจบไฟล์ `WS-EOF-FLAG` ถูกตั้งเป็น `"Y"` ไว้ (จาก `SET END-OF-FILE TO TRUE`) แล้ว
**ค้างอยู่แบบนั้น** เมื่อ `CALL "INVLIST"` ครั้งที่สองเริ่มทำงาน เงื่อนไข `PERFORM UNTIL END-OF-FILE` เป็น
จริงตั้งแต่ก่อนวนรอบแรกด้วยซ้ำ (เพราะ `END-OF-FILE` เป็น `TRUE` ค้างมาจากรอบก่อน) ทำให้ลูปอ่านไฟล์ทั้งหมด
**ถูกข้ามไปเลยทั้งลูป** โดยไม่มี error หรือสัญญาณเตือนใด ๆ

วิธีแก้คือเพิ่ม `MOVE "N" TO WS-EOF-FLAG` ที่จุดเริ่มต้นของ `0000-MAIN` **ทุกครั้ง** เพื่อ "รีเซ็ต" flag
ก่อนเริ่มลูปใหม่เสมอ ไม่พึ่งพา `VALUE` clause ที่ใช้ได้แค่ตอนโปรแกรมถูกโหลดครั้งแรกเท่านั้น — โค้ดฉบับ
สมบูรณ์ด้านล่างมีการแก้ไขนี้เรียบร้อยแล้ว

### ซอร์สโค้ดฉบับสมบูรณ์: `invlist.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INVLIST.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INVENTORY-FILE ASSIGN TO "INVMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS INV-PRODUCT-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  INVENTORY-FILE.
       COPY "INVREC.CPY".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-PRICE-EDIT            PIC ZZZ,ZZ9.99.
       01  WS-NAME-EDIT             PIC X(25).
       01  WS-QTY-EDIT              PIC ZZZ,ZZ9.

       LINKAGE SECTION.
       01  LS-RECORD-COUNT          PIC 9(4).

       PROCEDURE DIVISION USING LS-RECORD-COUNT.
       0000-MAIN.
           MOVE 0 TO LS-RECORD-COUNT
      *> Reset the flag every call: WORKING-STORAGE in a called
      *> subprogram keeps its last value between CALLs (it is NOT
      *> reinitialized automatically), so without this line the
      *> second CALL "INVLIST" in the same run would start with
      *> END-OF-FILE already TRUE from the previous call and skip
      *> the whole read loop below. See step 347 for the full story.
           MOVE "N" TO WS-EOF-FLAG
           OPEN INPUT INVENTORY-FILE
           IF WS-FILE-STATUS NOT = "00"
               DISPLAY "ERROR: cannot open inventory file ("
                   WS-FILE-STATUS ")"
               GOBACK
           END-IF

           DISPLAY "=================================================="
           DISPLAY "CODE   PRODUCT NAME               QTY   UNIT PRICE"
           DISPLAY "=================================================="

           PERFORM UNTIL END-OF-FILE
               READ INVENTORY-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       ADD 1 TO LS-RECORD-COUNT
                       MOVE INV-PRODUCT-NAME TO WS-NAME-EDIT
                       MOVE INV-QTY-ON-HAND  TO WS-QTY-EDIT
                       MOVE INV-UNIT-PRICE   TO WS-PRICE-EDIT
                       DISPLAY INV-PRODUCT-CODE "  " WS-NAME-EDIT
                           " " WS-QTY-EDIT "  " WS-PRICE-EDIT
               END-READ
           END-PERFORM

           DISPLAY "=================================================="
           CLOSE INVENTORY-FILE
           GOBACK.
```

### คอมไพล์และรัน

```bash
cobc -x -o test_list test_list.cob invlist.cob
./test_list
```

### ผลลัพธ์จริงที่ได้ (ไฟล์มีสินค้า P0001 เพียงรายการเดียวสืบเนื่องจากขั้นตอนก่อนหน้า)

```
==================================================
CODE   PRODUCT NAME               QTY   UNIT PRICE
==================================================
P0001   USB Flash Drive 32GB           50      250.00
==================================================
Record count from INVLIST: 0001
```

### อธิบายโค้ดทีละส่วน

- `LS-RECORD-COUNT` เป็นพารามิเตอร์ output ที่บอกจำนวน record ที่พิมพ์ไปทั้งหมด ผู้เรียกสามารถนำไปแสดง
  สรุปท้ายรายงานได้โดยไม่ต้องนับเอง
- `PIC ZZZ,ZZ9.99` และ `PIC ZZZ,ZZ9` (ตัวแปร edit) คือ Numeric Edited PICTURE ที่เรียนมาตั้งแต่ Part 006
  ใช้จัดรูปแบบตัวเลขให้มีเครื่องหมายจุลภาคคั่นหลักพันและตัดเลข 0 นำหน้าที่ไม่จำเป็นออก ทำให้รายงานอ่านง่าย
  กว่าตัวเลขดิบแบบ `9(6)`
- การอ่านด้วย `ACCESS MODE IS SEQUENTIAL` กับไฟล์ Indexed หมายความว่าทุกครั้งที่ `OPEN INPUT` ใหม่ การ
  อ่านจะ**เริ่มจาก record แรกตามลำดับคีย์เสมอ** (ต่างจากตัวชี้ตำแหน่งไฟล์ทั่วไปที่อาจค้างตำแหน่งเดิม) นี่
  คือเหตุผลที่การ `OPEN`/`CLOSE` ใหม่ทุกครั้งที่ `CALL` เพียงพอสำหรับการอ่านครบทุก record — ปัญหาที่เจอ
  ไม่ได้มาจากตำแหน่งไฟล์ แต่มาจาก `WS-EOF-FLAG` ใน WORKING-STORAGE ที่ค้างข้าม `CALL` ตามที่อธิบายไปแล้ว

### ข้อควรระวัง

- **กฎสำคัญที่สุดของขั้นตอนนี้**: ตัวแปร flag ใด ๆ ที่ใช้ควบคุมลูปใน subprogram ที่อาจถูก `CALL` มากกว่า
  หนึ่งครั้งในโปรแกรมเดียวกัน **ต้อง reset ค่าด้วยตนเองที่จุดเริ่มต้นของ `PROCEDURE DIVISION` เสมอ**
  ห้ามพึ่งพา `VALUE` clause เพียงอย่างเดียว เพราะ `VALUE` มีผลแค่ตอนโปรแกรมถูกโหลดเข้าหน่วยความจำครั้งแรก
  เท่านั้น ไม่ใช่ทุกครั้งที่ `CALL`
- กฎนี้ใช้กับ**ทุกตัวแปรใน WORKING-STORAGE ของ subprogram** ไม่ใช่แค่ flag เท่านั้น — ตัวนับ (counter),
  ตัวสะสมผลรวม (accumulator), หรือค่าเริ่มต้นอื่นใดก็มีความเสี่ยงแบบเดียวกันถ้า subprogram ถูกออกแบบให้
  เรียกซ้ำได้หลายครั้งในหนึ่ง run

### แบบฝึกหัดที่ 347.1: รายงานมูลค่าสินค้าคงคลังรวม (Total Inventory Value Report)

**โจทย์**: จงต่อยอด `invlist.cob` ให้คำนวณและแสดง **มูลค่าสินค้าคงคลังรวมทั้งหมด** (ผลรวมของ จำนวนคงเหลือ
× ราคาต่อหน่วย ของทุกสินค้า) ต่อท้ายรายงาน

**เฉลยแนวทาง**: เขียนเป็นโปรแกรมแยกต่างหาก (ไม่ใช่แก้ `invlist.cob` ที่เป็น subprogram โดยตรง เพราะ
รายงานนี้ไม่ต้องใช้ในเมนูหลัก) ที่เพิ่มตัวแปรสะสมผลรวมและคำนวณทีละ record ระหว่างวนลูป:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EX347TOTAL.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INVENTORY-FILE ASSIGN TO "INVMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS INV-PRODUCT-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  INVENTORY-FILE.
       COPY "INVREC.CPY".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-LINE-VALUE            PIC 9(8)V99.
       01  WS-TOTAL-VALUE           PIC 9(10)V99 VALUE 0.
       01  WS-TOTAL-EDIT            PIC ZZZ,ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       0000-MAIN.
           OPEN INPUT INVENTORY-FILE
           PERFORM UNTIL END-OF-FILE
               READ INVENTORY-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       COMPUTE WS-LINE-VALUE =
                           INV-QTY-ON-HAND * INV-UNIT-PRICE
                       ADD WS-LINE-VALUE TO WS-TOTAL-VALUE
                       DISPLAY INV-PRODUCT-CODE " qty=" INV-QTY-ON-HAND
                           " value=" WS-LINE-VALUE
               END-READ
           END-PERFORM
           CLOSE INVENTORY-FILE

           MOVE WS-TOTAL-VALUE TO WS-TOTAL-EDIT
           DISPLAY "TOTAL INVENTORY VALUE: " WS-TOTAL-EDIT
           STOP RUN.
```

**ผลลัพธ์จริงที่ได้จากการทดสอบ** (รันกับไฟล์ที่มีสินค้า 4 รายการจากการทดสอบเต็มระบบในขั้นตอนที่ 350):

```
P0001  qty=000080 value=00020000.00
P0002  qty=000015 value=00005257.50
P0003  qty=000015 value=00001800.00
P0004  qty=000040 value=00007230.00
TOTAL INVENTORY VALUE:      34,287.50
```

`WS-LINE-VALUE PIC 9(8)V99` ต้องมีจำนวนหลักมากพอสำหรับผลคูณของ `INV-QTY-ON-HAND` (6 หลัก) กับ
`INV-UNIT-PRICE` (6 หลัก + ทศนิยม 2 ตำแหน่ง) ตามกฎการคูณที่เรียนมาใน Part 009 ขั้นตอนที่ 83 (ผลคูณอาจมี
จำนวนหลักมากกว่าตัวตั้งทั้งสองรวมกัน) และ `WS-TOTAL-VALUE PIC 9(10)V99` ต้องใหญ่กว่านั้นอีกเพื่อรองรับ
การสะสมผลรวมจากหลาย record โดยไม่ล้น

---

## ขั้นตอนที่ 348: รายงานแจ้งเตือนสินค้าใกล้หมดด้วย `invlow.cob`

### แนวคิด

`invlow.cob` คือ subprogram สุดท้ายในกลุ่มรายงาน มีโครงสร้างเหมือน `invlist.cob` เกือบทั้งหมด (และได้
แก้ปัญหา `WS-EOF-FLAG` ที่ค้างข้าม `CALL` ไว้ตั้งแต่ต้นแล้ว เพราะเรียนรู้จากบั๊กในขั้นตอนที่ 347 มาก่อน)
ต่างกันตรงเงื่อนไขการกรอง: พิมพ์**เฉพาะสินค้าที่จำนวนคงเหลือต่ำกว่าจุดสั่งซื้อ** (`INV-QTY-ON-HAND <
INV-REORDER-LEVEL`) เท่านั้น ตอบโจทย์ความต้องการข้อ 6 ของโปรเจกต์: การแจ้งเตือนล่วงหน้าก่อนสินค้าจะหมด
สต๊อกจริง

### ซอร์สโค้ดฉบับสมบูรณ์: `invlow.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INVLOW.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INVENTORY-FILE ASSIGN TO "INVMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS INV-PRODUCT-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  INVENTORY-FILE.
       COPY "INVREC.CPY".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".
       01  WS-QTY-EDIT              PIC ZZZ,ZZ9.
       01  WS-REORDER-EDIT          PIC ZZZ,ZZ9.
       01  WS-NAME-EDIT             PIC X(25).

       LINKAGE SECTION.
       01  LS-LOW-COUNT             PIC 9(4).

       PROCEDURE DIVISION USING LS-LOW-COUNT.
       0000-MAIN.
           MOVE 0 TO LS-LOW-COUNT
      *> Same reset as INVLIST: WORKING-STORAGE survives between
      *> CALLs to the same subprogram, so the flag must be put
      *> back to "N" explicitly at the start of every call.
           MOVE "N" TO WS-EOF-FLAG
           OPEN INPUT INVENTORY-FILE
           IF WS-FILE-STATUS NOT = "00"
               DISPLAY "ERROR: cannot open inventory file ("
                   WS-FILE-STATUS ")"
               GOBACK
           END-IF

           DISPLAY "============================================"
           DISPLAY "  LOW STOCK ALERT (Qty below reorder level)"
           DISPLAY "============================================"
           DISPLAY "CODE   PRODUCT NAME               QTY  REORDER"

           PERFORM UNTIL END-OF-FILE
               READ INVENTORY-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       IF INV-QTY-ON-HAND < INV-REORDER-LEVEL
                           ADD 1 TO LS-LOW-COUNT
                           MOVE INV-PRODUCT-NAME  TO WS-NAME-EDIT
                           MOVE INV-QTY-ON-HAND   TO WS-QTY-EDIT
                           MOVE INV-REORDER-LEVEL TO WS-REORDER-EDIT
                           DISPLAY INV-PRODUCT-CODE "  " WS-NAME-EDIT
                               " " WS-QTY-EDIT " " WS-REORDER-EDIT
                       END-IF
               END-READ
           END-PERFORM

           IF LS-LOW-COUNT = 0
               DISPLAY "(No products below reorder level.)"
           END-IF
           DISPLAY "============================================"
           CLOSE INVENTORY-FILE
           GOBACK.
```

### คอมไพล์และรัน (ทดสอบร่วมกับ `invlist.cob` ในโปรแกรมเดียว)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TESTLISTLOW.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNT      PIC 9(4).
       01  WS-LOW-COUNT  PIC 9(4).

       PROCEDURE DIVISION.
           CALL "INVLIST" USING WS-COUNT
           DISPLAY "Record count from INVLIST: " WS-COUNT

           CALL "INVLOW" USING WS-LOW-COUNT
           DISPLAY "Low-stock count from INVLOW: " WS-LOW-COUNT
           STOP RUN.
```

```bash
cobc -x -o test_list_low test_list_low.cob invlist.cob invlow.cob
./test_list_low
```

### ผลลัพธ์จริงที่ได้ (ไฟล์มี P0001 จำนวน 50 หน่วย จุดสั่งซื้อ 20 — ยังไม่ต่ำกว่าจุดสั่งซื้อ)

```
==================================================
CODE   PRODUCT NAME               QTY   UNIT PRICE
==================================================
P0001   USB Flash Drive 32GB           50      250.00
==================================================
Record count from INVLIST: 0001
============================================
  LOW STOCK ALERT (Qty below reorder level)
============================================
CODE   PRODUCT NAME               QTY  REORDER
(No products below reorder level.)
============================================
Low-stock count from INVLOW: 0000
```

ผลลัพธ์นี้ยังยืนยันด้วยว่าบั๊กจากขั้นตอนที่ 347 ถูกแก้ไขเรียบร้อยแล้ว: `CALL "INVLIST"` ทำงานถูกต้องตาม
ปกติแม้จะมี `CALL "INVLOW"` ตามมาในโปรแกรมเดียวกัน (คนละ subprogram กันจึงไม่ชนกัน แต่ทั้งคู่ก็มีการ
reset flag ของตัวเองอย่างถูกต้องแล้วด้วย)

### อธิบายโค้ดทีละส่วน

- `IF INV-QTY-ON-HAND < INV-REORDER-LEVEL` คือเงื่อนไขกรองหลักของรายงานนี้ — เปรียบเทียบฟิลด์ตัวเลขสอง
  ตัวจาก record เดียวกันโดยตรง ไม่ต้องแปลงชนิดข้อมูลใด ๆ เพราะทั้งคู่เป็น `PIC 9(6)` เหมือนกัน
- `IF LS-LOW-COUNT = 0 DISPLAY "(No products below reorder level.)"` เป็นการจัดการ **เคสพิเศษที่ไม่มี
  ข้อมูลเลย** (empty case) ให้แสดงข้อความที่สื่อความหมายชัดเจน แทนที่จะปล่อยให้พื้นที่รายงานว่างเปล่า
  ซึ่งอาจทำให้ผู้ใช้สับสนว่ารายงานทำงานผิดพลาดหรือไม่

### ข้อควรระวัง

- ค่า `INV-REORDER-LEVEL` ที่ตั้งไว้ตอนเพิ่มสินค้า (`invadd`) มีผลโดยตรงต่อความไวของระบบแจ้งเตือนนี้ —
  ตั้งค่าสูงเกินไปจะแจ้งเตือนถี่เกินจำเป็น ตั้งต่ำเกินไปอาจแจ้งเตือนช้าเกินไปจนสินค้าหมดสต๊อกจริงก่อน
  ได้รับการเติมสินค้าทัน (เป็นการตัดสินใจเชิงธุรกิจ ไม่ใช่เชิงเทคนิค)

### แบบฝึกหัดที่ 348.1: เพิ่มฟังก์ชันลบสินค้า (Delete Product)

**โจทย์**: จงเพิ่ม subprogram ใหม่ชื่อ `invdel.cob` สำหรับลบสินค้าออกจากไฟล์ด้วยรหัสสินค้า โดยใช้คำสั่ง
`DELETE` ที่เรียนมาใน Part 025 (ทบทวน: `DELETE` ใช้กับไฟล์ Sequential ไม่ได้เลย แต่ใช้กับไฟล์ Indexed
ได้โดยตรงผ่านคีย์)

**เฉลยแนวทาง**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INVDEL.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INVENTORY-FILE ASSIGN TO "INVMAST.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS INV-PRODUCT-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  INVENTORY-FILE.
       COPY "INVREC.CPY".

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.

       LINKAGE SECTION.
       01  LS-PRODUCT-CODE          PIC X(6).
       01  LS-STATUS                PIC X(12).

       PROCEDURE DIVISION USING LS-PRODUCT-CODE LS-STATUS.
       0000-MAIN.
           MOVE SPACES TO LS-STATUS
           OPEN I-O INVENTORY-FILE
           IF WS-FILE-STATUS NOT = "00"
               MOVE "FILE-ERR" TO LS-STATUS
               GOBACK
           END-IF

           MOVE LS-PRODUCT-CODE TO INV-PRODUCT-CODE
           READ INVENTORY-FILE
               INVALID KEY
                   MOVE "NOTFOUND" TO LS-STATUS
               NOT INVALID KEY
                   DELETE INVENTORY-FILE
                       INVALID KEY
                           MOVE "DEL-ERR" TO LS-STATUS
                       NOT INVALID KEY
                           MOVE "OK" TO LS-STATUS
                   END-DELETE
           END-READ

           CLOSE INVENTORY-FILE
           GOBACK.
```

**ผลลัพธ์จริงที่ได้จากการทดสอบ** (ลบ `P0002` ออกจากไฟล์ที่มีสินค้า 4 รายการ แล้วเรียก `INVLIST` ยืนยัน):

```
Delete P0002 -> status: OK
Delete P0002 again -> status: NOTFOUND
==================================================
CODE   PRODUCT NAME               QTY   UNIT PRICE
==================================================
P0001   USB Flash Drive 32GB           80      250.00
P0003   USB-C Cable 1m                 15      120.00
P0004   HDMI Cable 2m                  40      180.75
==================================================
Products remaining: 0003
```

สังเกตว่าต้อง `READ` record นั้นให้สำเร็จก่อนเสมอ (COBOL กำหนดให้ตำแหน่งที่จะ `DELETE` มาจาก record ที่
เพิ่ง `READ` ล่าสุด) และการลบครั้งที่สองกับรหัสเดิมได้ `NOTFOUND` อย่างถูกต้องเพราะสินค้าถูกลบไปแล้วจริง
— พฤติกรรมนี้ตรงกับที่ Part 025 ขั้นตอนที่ 247 อธิบายไว้ว่า `DELETE` ใช้ได้เฉพาะไฟล์ที่รองรับการเข้าถึง
ด้วยคีย์หรือหมายเลข record (Indexed/Relative) เท่านั้น

---

## ขั้นตอนที่ 349: โปรแกรมเมนูหลัก `invmain.cob` — เชื่อมทุก Subprogram เข้าด้วยกัน

### แนวคิด

ถึงเวลาประกอบร่างระบบทั้งหมดเข้าด้วยกัน! `invmain.cob` คือโปรแกรมเดียวในระบบที่ผู้ใช้จะรันโดยตรง มันไม่มี
`FILE-CONTROL`, ไม่มี `FD`, และไม่ `COPY "INVREC.CPY"` เลยด้วยซ้ำ — เพราะมันไม่เคยแตะไฟล์ `INVMAST.DAT`
โดยตรง หน้าที่ทั้งหมดของมันคือ **แสดงเมนู → รับค่าจากผู้ใช้ → `CALL` subprogram ที่ถูกต้อง → แสดงผลลัพธ์
ที่ subprogram ส่งกลับมา** เท่านั้น นี่คือรูปธรรมของหลักการ **Separation of Concerns** ที่วางแผนไว้ตั้งแต่
ขั้นตอนที่ 341

โครงลูปหลักใช้ `PERFORM WITH TEST AFTER UNTIL WS-MENU-CHOICE = 7` แบบเดียวกับเครื่องคิดเลขใน Part 015
ขั้นตอนที่ 142 (แสดงเมนูอย่างน้อยหนึ่งครั้งเสมอก่อนตรวจสอบเงื่อนไขออก) ร่วมกับ `EVALUATE` เพื่อกระจายไปยัง
Paragraph ที่ `CALL` subprogram แต่ละตัว

### ซอร์สโค้ดฉบับสมบูรณ์: `invmain.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INVMAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MENU-CHOICE           PIC 9(1)  VALUE 0.

       01  WS-PRODUCT-CODE          PIC X(6).
       01  WS-PRODUCT-NAME          PIC X(25).
       01  WS-QTY-ON-HAND           PIC 9(6).
       01  WS-UNIT-PRICE            PIC 9(6)V99.
       01  WS-REORDER-LEVEL         PIC 9(6).
       01  WS-QTY-CHANGE            PIC 9(6).
       01  WS-NEW-QTY               PIC 9(6).
       01  WS-STATUS                PIC X(12).
       01  WS-RECORD-COUNT          PIC 9(4).
       01  WS-LOW-COUNT             PIC 9(4).
       01  WS-PRICE-EDIT            PIC ZZZ,ZZ9.99.
       01  WS-QTY-EDIT              PIC ZZZ,ZZ9.

       PROCEDURE DIVISION.
       0000-MAIN-PROCESS.
           DISPLAY "===================================="
           DISPLAY "   INVENTORY MANAGEMENT SYSTEM"
           DISPLAY "===================================="
           PERFORM WITH TEST AFTER
                   UNTIL WS-MENU-CHOICE = 7
               PERFORM 1000-SHOW-MENU
               PERFORM 2000-GET-CHOICE
               EVALUATE WS-MENU-CHOICE
                   WHEN 1
                       PERFORM 3000-ADD-PRODUCT
                   WHEN 2
                       PERFORM 4000-RECEIVE-STOCK
                   WHEN 3
                       PERFORM 5000-ISSUE-STOCK
                   WHEN 4
                       PERFORM 6000-SEARCH-PRODUCT
                   WHEN 5
                       PERFORM 7000-LIST-ALL
                   WHEN 6
                       PERFORM 8000-LOW-STOCK-REPORT
                   WHEN 7
                       DISPLAY "Exiting Inventory System. Goodbye!"
                   WHEN OTHER
                       DISPLAY "Invalid choice, please select 1-7."
               END-EVALUATE
           END-PERFORM
           STOP RUN.

       1000-SHOW-MENU.
           DISPLAY " ".
           DISPLAY "1. Add new product".
           DISPLAY "2. Receive stock (increase quantity)".
           DISPLAY "3. Issue stock (decrease quantity)".
           DISPLAY "4. Search product by code".
           DISPLAY "5. List all products".
           DISPLAY "6. Low stock alert report".
           DISPLAY "7. Exit".

       2000-GET-CHOICE.
           DISPLAY "Enter your choice (1-7): " WITH NO ADVANCING.
           ACCEPT WS-MENU-CHOICE.

       3000-ADD-PRODUCT.
           DISPLAY "Product code (6 chars) : " WITH NO ADVANCING.
           ACCEPT WS-PRODUCT-CODE.
           DISPLAY "Product name           : " WITH NO ADVANCING.
           ACCEPT WS-PRODUCT-NAME.
           DISPLAY "Quantity on hand       : " WITH NO ADVANCING.
           ACCEPT WS-QTY-ON-HAND.
           DISPLAY "Unit price (e.g 199.50): " WITH NO ADVANCING.
           ACCEPT WS-UNIT-PRICE.
           DISPLAY "Reorder level          : " WITH NO ADVANCING.
           ACCEPT WS-REORDER-LEVEL.

           CALL "INVADD" USING WS-PRODUCT-CODE WS-PRODUCT-NAME
               WS-QTY-ON-HAND WS-UNIT-PRICE WS-REORDER-LEVEL
               WS-STATUS.

           EVALUATE WS-STATUS
               WHEN "OK"
                   DISPLAY "Product " WS-PRODUCT-CODE " added."
               WHEN "DUPLICATE"
                   DISPLAY "ERROR: product code already exists."
               WHEN OTHER
                   DISPLAY "ERROR: could not add product ("
                       WS-STATUS ")"
           END-EVALUATE.

       4000-RECEIVE-STOCK.
           DISPLAY "Product code           : " WITH NO ADVANCING.
           ACCEPT WS-PRODUCT-CODE.
           DISPLAY "Quantity received      : " WITH NO ADVANCING.
           ACCEPT WS-QTY-CHANGE.

           CALL "INVRECV" USING WS-PRODUCT-CODE WS-QTY-CHANGE
               WS-NEW-QTY WS-STATUS.

           EVALUATE WS-STATUS
               WHEN "OK"
                   MOVE WS-NEW-QTY TO WS-QTY-EDIT
                   DISPLAY "New quantity on hand: " WS-QTY-EDIT
               WHEN "NOTFOUND"
                   DISPLAY "ERROR: product code not found."
               WHEN OTHER
                   DISPLAY "ERROR: " WS-STATUS
           END-EVALUATE.

       5000-ISSUE-STOCK.
           DISPLAY "Product code           : " WITH NO ADVANCING.
           ACCEPT WS-PRODUCT-CODE.
           DISPLAY "Quantity issued        : " WITH NO ADVANCING.
           ACCEPT WS-QTY-CHANGE.

           CALL "INVISSUE" USING WS-PRODUCT-CODE WS-QTY-CHANGE
               WS-NEW-QTY WS-STATUS.

           EVALUATE WS-STATUS
               WHEN "OK"
                   MOVE WS-NEW-QTY TO WS-QTY-EDIT
                   DISPLAY "New quantity on hand: " WS-QTY-EDIT
               WHEN "NOTFOUND"
                   DISPLAY "ERROR: product code not found."
               WHEN "INSUFFICIENT"
                   MOVE WS-NEW-QTY TO WS-QTY-EDIT
                   DISPLAY "ERROR: not enough stock. On hand: "
                       WS-QTY-EDIT
               WHEN OTHER
                   DISPLAY "ERROR: " WS-STATUS
           END-EVALUATE.

       6000-SEARCH-PRODUCT.
           DISPLAY "Product code to search : " WITH NO ADVANCING.
           ACCEPT WS-PRODUCT-CODE.

           CALL "INVSRCH" USING WS-PRODUCT-CODE WS-PRODUCT-NAME
               WS-QTY-ON-HAND WS-UNIT-PRICE WS-REORDER-LEVEL
               WS-STATUS.

           IF WS-STATUS = "OK"
               MOVE WS-UNIT-PRICE TO WS-PRICE-EDIT
               MOVE WS-QTY-ON-HAND TO WS-QTY-EDIT
               DISPLAY "Code    : " WS-PRODUCT-CODE
               DISPLAY "Name    : " WS-PRODUCT-NAME
               DISPLAY "Qty     : " WS-QTY-EDIT
               DISPLAY "Price   : " WS-PRICE-EDIT
               DISPLAY "Reorder : " WS-REORDER-LEVEL
           ELSE
               DISPLAY "ERROR: product not found."
           END-IF.

       7000-LIST-ALL.
           CALL "INVLIST" USING WS-RECORD-COUNT.
           DISPLAY "Total products listed: " WS-RECORD-COUNT.

       8000-LOW-STOCK-REPORT.
           CALL "INVLOW" USING WS-LOW-COUNT.
           DISPLAY "Total low-stock items: " WS-LOW-COUNT.
```

### คอมไพล์: การ Link หลายไฟล์ต้นฉบับเป็นโปรแกรมเดียว

```bash
cobc -x -o invmain invmain.cob invadd.cob invrecv.cob invissue.cob \
    invsrch.cob invlist.cob invlow.cob
```

คำสั่งนี้คือหัวใจสำคัญของการ "ประกอบระบบ" ทั้งหมด: `cobc` คอมไพล์ไฟล์ต้นฉบับ**ทั้งเจ็ดไฟล์แยกกัน**
(เหมือนที่เคยทำในขั้นตอนที่ 343-348 กับโปรแกรมทดสอบ) แล้ว **static link** ให้กลายเป็นไฟล์ execute
(`invmain`) เพียงไฟล์เดียว ผลลัพธ์คือโปรแกรมที่ CALL ระหว่างกันได้ทันทีโดยไม่ต้องพึ่ง dynamic loading หรือ
environment variable ใด ๆ ตอนรัน (`-x` บอกให้สร้างโปรแกรมที่ execute ได้ ต่างจาก `-m` ที่ใช้สร้าง module
แยกสำหรับทดสอบทีละตัวในขั้นตอนก่อนหน้า)

### ผลลัพธ์การคอมไพล์

```
(ไม่มีข้อความ error หรือ warning ใด ๆ ปรากฏ - คอมไพล์และ link ผ่านเรียบร้อย)
```

### อธิบายโค้ดทีละส่วน

- `CALL "INVADD" USING WS-PRODUCT-CODE WS-PRODUCT-NAME ...` — ชื่อโปรแกรมใน `CALL` เป็น string literal
  (`"INVADD"`) ตรงกับ `PROGRAM-ID. INVADD.` ใน `invadd.cob` เป๊ะ (ทบทวนจาก Part 031) และ**ลำดับ**
  พารามิเตอร์ต้องตรงกับลำดับใน `PROCEDURE DIVISION USING` ของ subprogram นั้นทุกประการ COBOL จับคู่
  พารามิเตอร์ด้วย**ตำแหน่ง** ไม่ใช่ชื่อ (ต่างจากภาษาสมัยใหม่บางภาษาที่รองรับ named arguments)
- สังเกตว่าตัวแปรใน `invmain.cob` ใช้คำนำหน้า `WS-` (เพราะเป็น WORKING-STORAGE ของโปรแกรมนี้เอง) ในขณะที่
  ตัวแปรฝั่งรับใน subprogram ใช้คำนำหน้า `LS-` (LINKAGE SECTION) — **ชื่อไม่จำเป็นต้องตรงกัน** ระหว่าง
  ผู้เรียกกับผู้ถูกเรียก สิ่งที่ต้องตรงกันคือ**ลำดับ**และ**ชนิด/ขนาดข้อมูล** (`PICTURE`) เท่านั้น
- `EVALUATE WS-STATUS WHEN "OK" ... WHEN "DUPLICATE" ...` คือรูปแบบมาตรฐานที่ `invmain` ใช้ตัดสินใจแสดง
  ข้อความอะไรตามรหัสสถานะที่ subprogram ส่งกลับมา — ตรรกะทางธุรกิจ (ตรวจสอบซ้ำ/ติดลบ/ไม่พบ) ทั้งหมดอยู่
  ใน subprogram ส่วน `invmain` มีหน้าที่แค่ "แปล" รหัสสถานะเป็นข้อความที่มนุษย์อ่านเข้าใจเท่านั้น

### ข้อควรระวัง

- **ต้องรัน `invcreate` ก่อนรัน `invmain` เป็นครั้งแรกเสมอ** มิฉะนั้นทุกการเรียก subprogram ที่แตะไฟล์
  จะได้ `WS-FILE-STATUS` ที่ไม่ใช่ `"00"` และคืนค่า `"FILE-ERR"` กลับมาทันที (ได้ทดสอบพฤติกรรมนี้จริง
  แล้ว: การพยายาม Add ก่อนรัน `invcreate` ให้ผลลัพธ์ `ERROR: could not add product (FILE-ERR    )`
  ตรงตามที่ออกแบบไว้ ไม่ใช่โปรแกรม crash)
- ลำดับไฟล์ต้นฉบับในคำสั่ง `cobc -x -o invmain ...` **ไม่มีผลต่อการทำงาน** (ไม่เหมือนบางภาษาที่ต้อง
  ประกาศก่อนใช้) เพราะ `CALL` ใน COBOL อ้างอิงชื่อโปรแกรมด้วย string ไม่ใช่การอ้างอิงแบบ compile-time
  แต่ควรจัดลำดับให้อ่านง่าย (เช่น ไฟล์หลักก่อน ตามด้วย subprogram ตามลำดับเมนู) เพื่อความสะดวกของทีมพัฒนา

### แบบฝึกหัดที่ 349.1

**โจทย์**: จงอธิบายว่าทำไม `invmain.cob` จึงไม่มี `ENVIRONMENT DIVISION`, `FILE SECTION`, หรือ
`COPY "INVREC.CPY"` เลย ทั้งที่ subprogram อีก 6 ตัวมีครบทุกอย่างนี้เหมือนกันหมด

**เฉลยแนวทาง**: เพราะ `invmain.cob` **ไม่เคยเปิดหรือแตะไฟล์ `INVMAST.DAT` โดยตรง**เลยแม้แต่ครั้งเดียว —
มันมอบหมายงานทั้งหมดที่เกี่ยวกับไฟล์ให้ subprogram แต่ละตัวรับผิดชอบแทน (ตามหลักการ Separation of
Concerns ที่วางแผนไว้ในขั้นตอนที่ 341) `ENVIRONMENT DIVISION`/`FILE SECTION`/`COPY "INVREC.CPY"` จำเป็น
ก็ต่อเมื่อโปรแกรมนั้นมี `SELECT`/`FD` เป็นของตัวเอง ซึ่ง `invmain.cob` ไม่มีเลย มันรู้จักแค่ "ชื่อ
subprogram ที่จะ CALL" และ "พารามิเตอร์ที่จะส่งเข้า-รับออก" เท่านั้น ไม่จำเป็นต้องรู้จักโครงสร้าง record
ภายในไฟล์แต่อย่างใด — เป็นตัวอย่างที่ชัดเจนมากของการซ่อนรายละเอียดการทำงาน (encapsulation) ผ่านการแบ่ง
โมดูลด้วย `CALL`

---

## ขั้นตอนที่ 350: ทดสอบระบบทั้งหมดแบบ End-to-End และสรุปเฟส 2

### แนวคิด

ขั้นตอนสุดท้ายนี้คือบทพิสูจน์ว่าทั้ง 8 ไฟล์ที่สร้างมาตลอด Part นี้**ทำงานร่วมกันเป็นระบบเดียวได้จริง**
ไม่ใช่แค่ทำงานแยกส่วนได้เท่านั้น เราจะจำลองสถานการณ์ใช้งานจริงที่สมจริง: เพิ่มสินค้าหลายรายการ ทำธุรกรรม
รับ/เบิกสินค้า รวมถึงกรณีที่ต้องถูกปฏิเสธ (เบิกเกินจำนวนที่มี) ค้นหาสินค้า พิมพ์รายงาน และให้ระบบแจ้งเตือน
สินค้าใกล้หมดสต๊อกโดยอัตโนมัติ ทั้งหมดผ่านโปรแกรมเมนูหลัก (`invmain`) เพียงตัวเดียวที่ผู้ใช้จริงจะสัมผัส

### ขั้นตอนการ Build ระบบทั้งหมด

```bash
# 1) Compile the one-time setup program.
cobc -x -o invcreate invcreate.cob

# 2) Compile the interactive menu, linking all six subprograms
#    into one executable.
cobc -x -o invmain invmain.cob invadd.cob invrecv.cob invissue.cob \
    invsrch.cob invlist.cob invlow.cob

# 3) Run the setup program once to create an empty INVMAST.DAT.
./invcreate
```

### สถานการณ์ทดสอบ (Test Scenario)

เราจำลองการใช้งานร้านขายอุปกรณ์คอมพิวเตอร์ขนาดเล็กที่มีสินค้า 4 รายการ พร้อมทำธุรกรรมที่ครอบคลุมทุก
เส้นทางสำคัญของระบบ (test coverage) ผ่านไฟล์ `test_input.txt` ที่ป้อนผ่าน stdin ตามเทคนิคที่อธิบายไว้ใน
คำนำของ Part นี้:

```
1
P0001
USB Flash Drive 32GB
50
250.00
20
1
P0002
Wireless Mouse
15
350.50
20
1
P0003
USB-C Cable 1m
100
120.00
30
2
P0001
30
3
P0003
85
3
P0001
200
4
P0002
5
6
1
P0004
HDMI Cable 2m
40
180.75
10
5
6
7
```

ลำดับเหตุการณ์ที่ไฟล์นี้จำลอง:

1. เพิ่มสินค้า 3 รายการ: `P0001` (50 หน่วย, จุดสั่งซื้อ 20), `P0002` (15 หน่วย, จุดสั่งซื้อ 20 — **ต่ำกว่า
   จุดสั่งซื้อตั้งแต่แรก**), `P0003` (100 หน่วย, จุดสั่งซื้อ 30)
2. รับสินค้าเข้าคลัง: `P0001` +30 หน่วย (50 → 80)
3. เบิกสินค้า: `P0003` -85 หน่วย (100 → 15, **ตกลงมาต่ำกว่าจุดสั่งซื้อ 30**)
4. พยายามเบิกสินค้าเกินจำนวนที่มี: `P0001` -200 หน่วย (มีแค่ 80 หน่วย → **ต้องถูกปฏิเสธ**)
5. ค้นหาสินค้า `P0002`
6. พิมพ์รายงานสินค้าทั้งหมด (คาดว่าได้ 3 รายการ)
7. พิมพ์รายงานแจ้งเตือนสินค้าใกล้หมด (คาดว่าได้ 2 รายการ: `P0002` และ `P0003`)
8. เพิ่มสินค้ารายการที่ 4: `P0004` (40 หน่วย, จุดสั่งซื้อ 10)
9. พิมพ์รายงานสินค้าทั้งหมดอีกครั้ง (คาดว่าได้ครบ 4 รายการ — พิสูจน์ว่าบั๊ก `WS-EOF-FLAG` จากขั้นตอนที่
   347 ถูกแก้ไขแล้วจริง เพราะนี่คือการเรียก `INVLIST` เป็นครั้งที่สองในการรันเดียวกัน)
10. พิมพ์รายงานแจ้งเตือนสินค้าใกล้หมดอีกครั้ง (คาดว่ายังคงได้ 2 รายการเดิม เพราะ `P0004` ยังไม่ต่ำกว่า
    จุดสั่งซื้อ)
11. ออกจากโปรแกรม

### รันระบบเต็มรูปแบบ

```bash
./invmain < test_input.txt
```

### ผลลัพธ์จริงที่ได้ (บันทึกจากการรันจริงทั้งหมด ไม่มีการตัดต่อ)

```
====================================
   INVENTORY MANAGEMENT SYSTEM
====================================

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): Product code (6 chars) : Product name           : Quantity on hand       : Unit price (e.g 199.50): Reorder level          : Product P0001  added.

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): Product code (6 chars) : Product name           : Quantity on hand       : Unit price (e.g 199.50): Reorder level          : Product P0002  added.

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): Product code (6 chars) : Product name           : Quantity on hand       : Unit price (e.g 199.50): Reorder level          : Product P0003  added.

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): Product code           : Quantity received      : New quantity on hand:      80

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): Product code           : Quantity issued        : New quantity on hand:      15

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): Product code           : Quantity issued        : ERROR: not enough stock. On hand:      80

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): Product code to search : Code    : P0002
Name    : Wireless Mouse
Qty     :      15
Price   :     350.50
Reorder : 000020

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): ==================================================
CODE   PRODUCT NAME               QTY   UNIT PRICE
==================================================
P0001   USB Flash Drive 32GB           80      250.00
P0002   Wireless Mouse                 15      350.50
P0003   USB-C Cable 1m                 15      120.00
==================================================
Total products listed: 0003

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): ============================================
  LOW STOCK ALERT (Qty below reorder level)
============================================
CODE   PRODUCT NAME               QTY  REORDER
P0002   Wireless Mouse                 15      20
P0003   USB-C Cable 1m                 15      30
============================================
Total low-stock items: 0002

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): Product code (6 chars) : Product name           : Quantity on hand       : Unit price (e.g 199.50): Reorder level          : Product P0004  added.

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): ==================================================
CODE   PRODUCT NAME               QTY   UNIT PRICE
==================================================
P0001   USB Flash Drive 32GB           80      250.00
P0002   Wireless Mouse                 15      350.50
P0003   USB-C Cable 1m                 15      120.00
P0004   HDMI Cable 2m                  40      180.75
==================================================
Total products listed: 0004

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): ============================================
  LOW STOCK ALERT (Qty below reorder level)
============================================
CODE   PRODUCT NAME               QTY  REORDER
P0002   Wireless Mouse                 15      20
P0003   USB-C Cable 1m                 15      30
============================================
Total low-stock items: 0002

1. Add new product
2. Receive stock (increase quantity)
3. Issue stock (decrease quantity)
4. Search product by code
5. List all products
6. Low stock alert report
7. Exit
Enter your choice (1-7): Exiting Inventory System. Goodbye!
```

### วิเคราะห์ผลลัพธ์: ทุกความต้องการของโปรเจกต์ได้รับการพิสูจน์แล้วจริง

| ความต้องการ (จากขั้นตอนที่ 341) | หลักฐานจากผลลัพธ์จริงข้างต้น |
|---|---|
| เพิ่มสินค้าใหม่ | `Product P0001 added.` / `P0002 added.` / `P0003 added.` / `P0004 added.` |
| ปฏิเสธรหัสซ้ำ | พิสูจน์แยกต่างหากในขั้นตอนที่ 343 (`ERROR: product code already exists.`) |
| รับสินค้าเข้าคลัง | `New quantity on hand: 80` (50 + 30) |
| เบิกสินค้าออก | `New quantity on hand: 15` (100 - 85) |
| ป้องกันเบิกเกิน/ติดลบ | `ERROR: not enough stock. On hand: 80` เมื่อพยายามเบิก 200 จาก 80 |
| ค้นหาด้วยรหัส | แสดงข้อมูลครบของ `P0002` ถูกต้องทุกฟิลด์ |
| รายงานสินค้าทั้งหมด | แสดงครบ 3 รายการ แล้วครบ 4 รายการหลังเพิ่ม `P0004` |
| แจ้งเตือนสินค้าใกล้หมด | แสดง `P0002` และ `P0003` ถูกต้อง (ทั้งสองมีจำนวนต่ำกว่าจุดสั่งซื้อของตัวเอง) |
| ระบบแบ่งโมดูลจริง | คอมไพล์จาก 7 ไฟล์ต้นฉบับ link เป็น `invmain` หนึ่งไฟล์สำเร็จ ไม่มี error |
| `INVLIST`/`INVLOW` เรียกซ้ำได้ | เรียกทั้งคู่สองครั้งในการรันเดียวกัน ได้ผลลัพธ์ถูกต้องทั้งสองครั้ง |

### ข้อควรระวัง

- ระบบนี้เป็น **โปรเจกต์ระดับหลักสูตร (educational)** ยังไม่มีกลไกป้องกันการเข้าถึงไฟล์พร้อมกันจากหลาย
  ผู้ใช้ (concurrent access/file locking) ซึ่งเป็นเรื่องสำคัญมากในระบบธุรกิจจริงที่มีผู้ใช้หลายคนพร้อมกัน
  — หัวข้อนี้จะเรียนเชิงลึกเมื่อไปถึง CICS ในเฟส 4 ที่ออกแบบมาสำหรับ Transaction Processing แบบ concurrent
  โดยเฉพาะ
- ทุกครั้งที่แก้ไข `INVREC.CPY` (เช่น เพิ่มฟิลด์ใหม่) ต้อง **คอมไพล์ใหม่ทุกไฟล์ที่ `COPY` มันเข้าไป**
  (ทั้ง 7 ไฟล์ที่มี `FD`) ไม่ใช่แค่ไฟล์ที่แก้โดยตรง เพราะ `COPY` แทรกโค้ดตอนคอมไพล์ ไม่ใช่ตอนรัน (ทบทวน
  จาก Part 033)

### แบบฝึกหัดที่ 350.1

**โจทย์**: จากผลลัพธ์การรันจริงข้างต้น จงอธิบายว่าทำไมรายการ "Low stock alert report" ครั้งที่สอง (หลัง
เพิ่ม `P0004`) จึงยังคงแสดงแค่ 2 รายการเหมือนเดิม (`P0002`, `P0003`) ทั้งที่ตอนนี้มีสินค้าในระบบทั้งหมด
4 รายการแล้ว

**เฉลย**: เพราะ `P0004` ถูกเพิ่มเข้ามาด้วยจำนวน 40 หน่วย และจุดสั่งซื้อ 10 หน่วย เงื่อนไขของ `invlow.cob`
คือ `INV-QTY-ON-HAND < INV-REORDER-LEVEL` ซึ่งสำหรับ `P0004` คือ `40 < 10` ซึ่ง**เป็นเท็จ** (40 มากกว่า
10 มาก) จึงไม่ถูกกรองเข้ารายงาน ในขณะที่ `P0001` (80 หน่วย, จุดสั่งซื้อ 20) ก็ไม่เข้าเงื่อนไขเช่นกัน
(`80 < 20` เป็นเท็จ) มีเพียง `P0002` (`15 < 20` เป็นจริง) และ `P0003` (`15 < 30` เป็นจริง) เท่านั้นที่
ผ่านเงื่อนไข ผลลัพธ์นี้ยืนยันว่ารายงานทำงานถูกต้องตามตรรกะที่ออกแบบไว้ทุกประการ ไม่ใช่การแสดงผลค้างจาก
การเรียกครั้งก่อน (ซึ่งเป็นบั๊กที่เคยพบและแก้ไปแล้วในขั้นตอนที่ 347)

---

## สรุปท้ายบท

Part นี้คือจุดสิ้นสุดของ **เฟส 2: ระดับกลาง (Intermediate, Parts 016-035)** ของหลักสูตร เราได้นำความรู้
ทั้งหมดตลอด 20 Part ที่ผ่านมา มาประกอบร่างเป็นระบบจัดการสินค้าคงคลังที่ทำงานได้จริงและแบ่งโมดูลอย่างเป็น
ระบบ:

- **Part 016-018 (ตารางและ OCCURS)**: แม้โปรเจกต์นี้จะเน้นไฟล์มากกว่าตาราง แต่แนวคิดเรื่องการจัดกลุ่ม
  ข้อมูลที่ซ้ำกันเป็นชุดที่เรียนจาก `OCCURS` เป็นรากฐานความเข้าใจเรื่อง record และ field ที่ใช้ตลอดทั้ง
  โปรเจกต์
- **Part 019-022 (STRING/UNSTRING/INSPECT/REDEFINES)**: เทคนิคการจัดการข้อความเหล่านี้เป็นเครื่องมือ
  พื้นฐานที่จะกลับมาใช้บ่อยเมื่อระบบต้องประมวลผลข้อความจากผู้ใช้หรือไฟล์ภายนอกในเฟสถัดไป
- **Part 023-027 (ไฟล์ Sequential, FD, OPEN/CLOSE/READ/WRITE, Master-Detail, SORT/MERGE)**: รากฐาน
  สำคัญที่สุดของโปรเจกต์นี้ — แนวคิดเรื่อง `OPEN`/`CLOSE`, การออกแบบ `FD`, และรูปแบบ `READ ... AT END`
  ถูกนำมาใช้ซ้ำในทุก subprogram ที่เข้าถึงไฟล์
- **Part 028-030 (Indexed Files, Relative Files, File Status)**: หัวใจของโปรเจกต์นี้ — `RECORD KEY`,
  `ACCESS MODE IS RANDOM/SEQUENTIAL`, และ `INVALID KEY` คือสิ่งที่ทำให้ `INVMAST.DAT` ค้นหา-แก้ไข
  สินค้าทีละรายการด้วยรหัสได้อย่างรวดเร็ว โดยไม่ต้องอ่านทั้งไฟล์
- **Part 031-032 (CALL, การส่งพารามิเตอร์)**: กระดูกสันหลังของสถาปัตยกรรมทั้งระบบ — ทำให้ `invmain`
  แยกออกจาก subprogram ทั้ง 6 ตัวได้อย่างสมบูรณ์ ผ่าน `LINKAGE SECTION` และการส่งค่า `BY REFERENCE`
- **Part 033 (COPY/Copybook)**: `INVREC.CPY` คือ "สัญญา" กลางที่ทำให้ทั้ง 7 ไฟล์ที่แตะไฟล์มองเห็น
  โครงสร้าง record แบบเดียวกันเป๊ะ โดยไม่ต้องคัดลอกโค้ดซ้ำ
- **Part 034 (Nested Programs, END PROGRAM)**: แม้โปรเจกต์นี้เลือกใช้ subprogram แบบไฟล์แยก (separately
  compiled) แทน nested programs แต่แนวคิดเรื่องขอบเขตของโปรแกรมและการคืนการควบคุมด้วย `GOBACK` ที่เรียน
  มาก็เป็นพื้นฐานเดียวกัน

ที่สำคัญไม่แพ้ไวยากรณ์คือ **บทเรียนจากบั๊กจริงสองจุด**ที่พบระหว่างพัฒนาโปรเจกต์นี้: การตัดข้อความแบบ
เงียบ ๆ จาก `MOVE` เมื่อฟิลด์ปลายทางสั้นเกินไป (ขั้นตอนที่ 345) และ `WORKING-STORAGE` ของ subprogram
ที่ค้างค่าข้าม `CALL` แต่ละครั้ง (ขั้นตอนที่ 347) — ทั้งสองเป็นบั๊กประเภทที่**ไม่ทำให้โปรแกรม crash**
แต่ทำให้ตรรกะทำงานผิดอย่างเงียบ ๆ ซึ่งเป็นบั๊กที่อันตรายที่สุดในโค้ด COBOL ระดับธุรกิจจริง และเป็นเหตุผล
สำคัญที่สุดว่าทำไมหลักสูตรนี้ยืนยันเสมอว่า **ต้องคอมไพล์และรันทดสอบจริงทุกตัวอย่าง ไม่ใช่แค่อ่านโค้ด
ด้วยตา**

เฟส 2 จบลงแล้วด้วยระบบที่มีไฟล์ต้นฉบับ 8 ไฟล์ทำงานร่วมกันเป็นหนึ่งเดียว — นี่คือก้าวสำคัญจากการเขียน
โปรแกรมไฟล์เดียวในเฟส 1 สู่การออกแบบระบบแบบโมดูลที่ใกล้เคียงกับงานจริงในอุตสาหกรรมมากขึ้นอย่างมีนัยสำคัญ

**เฟส 3: ขั้นสูง (Parts 036-050)** ที่กำลังจะมาถึงจะพาคุณไปไกลกว่านี้อีกขั้น: Intrinsic Functions สำหรับ
ตัวเลขและวันที่, Report Writer Feature, Screen Section สำหรับสร้าง UI แบบ text-mode, JSON/XML
GENERATE/PARSE สำหรับเชื่อมต่อกับโลกภายนอก, และ Error Handling ขั้นสูงด้วย `DECLARATIVES` ก่อนจะจบเฟส
ด้วยโปรเจกต์รวบยอดที่ใหญ่กว่านี้อีก: **ระบบบัญชีลูกหนี้ (Accounts Receivable System)** ใน Part 050

**[← กลับไปยัง Part 034: Nested Programs และ END PROGRAM](part-034-nested-programs.md)**

**[ไปยัง Part 036: Intrinsic Functions: ตัวเลขและวันที่ (FUNCTION) →](part-036-intrinsic-functions-numeric-date.md)**
