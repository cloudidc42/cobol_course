# Part 005: DATA DIVISION เบื้องต้น: WORKING-STORAGE SECTION (ขั้นตอนที่ 41–50)

## คำนำของ Part นี้

ใน Part 004 คุณได้เรียนรู้ IDENTIFICATION DIVISION และ ENVIRONMENT DIVISION แบบละเอียดครบถ้วน
รวมถึงเห็นตัวอย่างที่มีการประกาศตัวแปรใน DATA DIVISION ผ่านมาบ้างแล้วในหลาย ๆ ตัวอย่าง ถึงเวลาแล้วที่เรา
จะเจาะลึก **DATA DIVISION** อย่างเป็นระบบ โดยเฉพาะ **WORKING-STORAGE SECTION** ซึ่งเป็นส่วนที่คุณจะ
ใช้งานบ่อยที่สุดตลอดทั้งหลักสูตร เพราะเป็นที่เก็บตัวแปรและโครงสร้างข้อมูลเกือบทั้งหมดที่โปรแกรมใช้งาน

Part นี้จะพาคุณทำความเข้าใจว่า WORKING-STORAGE SECTION คืออะไร ใช้ทำอะไร มี Level Number แบบไหนบ้าง
กลุ่มข้อมูล (Group) ต่างจากรายการเดี่ยว (Elementary) อย่างไร VALUE clause ทำงานอย่างไร และวิธีจัดระเบียบ
ตัวแปรให้อ่านง่ายในสไตล์ที่ใช้จริงในอุตสาหกรรม เมื่อจบ Part นี้ คุณจะสามารถออกแบบโครงสร้างข้อมูลของ
โปรแกรม COBOL ได้อย่างมั่นใจ ก่อนที่จะไปเรียนรู้ PICTURE Clause แบบละเอียดใน Part 006

---

## ขั้นตอนที่ 41: DATA DIVISION ภาพรวม — Section ต่าง ๆ ที่เป็นไปได้

### DATA DIVISION คืออะไร

`DATA DIVISION` เป็น Division ที่ 3 ของโปรแกรม COBOL มีหน้าที่**ประกาศข้อมูลและโครงสร้างข้อมูลทั้งหมด**
ที่โปรแกรมจะใช้งาน แบ่งออกเป็นหลาย Section ได้แก่:

| Section | หน้าที่ | จะสอนละเอียดเมื่อไหร่ |
|---|---|---|
| **FILE SECTION** | อธิบายโครงสร้าง record ของไฟล์ที่เปิดใช้งาน (คู่กับ FD) | Part 023-024 |
| **WORKING-STORAGE SECTION** | ประกาศตัวแปรที่มีอยู่ตลอดการทำงานของโปรแกรม | **Part นี้ (005)** |
| **LOCAL-STORAGE SECTION** | ประกาศตัวแปรที่ถูกสร้างใหม่ทุกครั้งที่โปรแกรมถูกเรียก (คล้าย INITIAL PROGRAM แต่เฉพาะบางตัวแปร) | กล่าวถึงเมื่อเรียน Subprogram/Recursive |
| **LINKAGE SECTION** | รับพารามิเตอร์จากโปรแกรมที่เรียกใช้งาน (CALL) | Part 031-032 |

### ตัวอย่าง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DATA-DIVISION-OVERVIEW.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COMPANY-NAME        PIC X(20) VALUE "ACME TRADING CO.".
       01  WS-YEAR-FOUNDED        PIC 9(4)  VALUE 1998.

       PROCEDURE DIVISION.
           DISPLAY "DATA DIVISION can contain FILE SECTION,".
           DISPLAY "WORKING-STORAGE SECTION, LOCAL-STORAGE".
           DISPLAY "SECTION, and LINKAGE SECTION.".
           DISPLAY "This program only uses WORKING-STORAGE:".
           DISPLAY WS-COMPANY-NAME.
           DISPLAY WS-YEAR-FOUNDED.
           STOP RUN.
```

**ผลลัพธ์**:

```
DATA DIVISION can contain FILE SECTION,
WORKING-STORAGE SECTION, LOCAL-STORAGE
SECTION, and LINKAGE SECTION.
This program only uses WORKING-STORAGE:
ACME TRADING CO.
1998
```

### ลำดับของ Section ภายใน DATA DIVISION

หากมีหลาย Section ในโปรแกรมเดียวกัน ต้องเรียงลำดับตามนี้เสมอ:
`FILE SECTION` → `WORKING-STORAGE SECTION` → `LOCAL-STORAGE SECTION` → `LINKAGE SECTION`

โปรแกรมส่วนใหญ่ในช่วงต้นของหลักสูตรนี้จะใช้แค่ `WORKING-STORAGE SECTION` เพียงอย่างเดียว
เพราะยังไม่ได้เรียนเรื่องไฟล์ (Part 023+) หรือ Subprogram (Part 031+)

### ข้อควรระวัง

- แม้จะมีถึง 4 Section ที่เป็นไปได้ แต่**ไม่จำเป็นต้องมีครบทุก Section** ในทุกโปรแกรม ใส่เฉพาะ Section
  ที่โปรแกรมนั้นใช้งานจริงเท่านั้น
- อย่าสับสนระหว่าง "Section" (ส่วนย่อยของ Division) กับ "Division" (ส่วนใหญ่ระดับบนสุด) ทั้งสองคำนี้
  มีความหมายต่างระดับกันใน COBOL

### แบบฝึกหัดที่ 41.1

**โจทย์**: หากโปรแกรมหนึ่งต้องใช้ทั้ง FILE SECTION และ WORKING-STORAGE SECTION จะต้องเรียงลำดับ
การเขียนอย่างไร

**เฉลย**: ต้องเขียน `FILE SECTION.` ก่อน แล้วตามด้วย `WORKING-STORAGE SECTION.` เสมอ ตามลำดับที่
กำหนดไว้ในไวยากรณ์ของ DATA DIVISION

---

## ขั้นตอนที่ 42: WORKING-STORAGE SECTION คืออะไร และมีอายุการใช้งานนานแค่ไหน

### นิยาม

**WORKING-STORAGE SECTION** คือพื้นที่หน่วยความจำที่ใช้เก็บ**ตัวแปรที่มีชีวิตอยู่ตลอดการทำงานของโปรแกรม**
ตั้งแต่โปรแกรมเริ่มทำงานจนกระทั่งจบ (`STOP RUN` หรือ `EXIT PROGRAM`) ค่าที่เก็บไว้ในตัวแปรเหล่านี้จะ
**คงอยู่**ตลอดไป ไม่ว่าคุณจะเรียก paragraph ไหนไปมากี่ครั้งก็ตาม ต่างจากตัวแปร local ในภาษาโปรแกรมสมัยใหม่
หลายภาษาที่ตัวแปรจะหายไปเมื่อออกจากฟังก์ชัน

### ตัวอย่างพิสูจน์การคงอยู่ของค่า

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. WS-PERSISTENCE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RUNNING-TOTAL       PIC 9(5) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARAGRAPH.
           PERFORM ADD-TEN.
           DISPLAY "After 1st call, total = " WS-RUNNING-TOTAL.
           PERFORM ADD-TEN.
           DISPLAY "After 2nd call, total = " WS-RUNNING-TOTAL.
           PERFORM ADD-TEN.
           DISPLAY "After 3rd call, total = " WS-RUNNING-TOTAL.
           DISPLAY "WORKING-STORAGE keeps its value for as long".
           DISPLAY "as the program keeps running.".
           STOP RUN.

       ADD-TEN.
           ADD 10 TO WS-RUNNING-TOTAL.
```

**ผลลัพธ์**:

```
After 1st call, total = 00010
After 2nd call, total = 00020
After 3rd call, total = 00030
WORKING-STORAGE keeps its value for as long
as the program keeps running.
```

### อธิบาย

`WS-RUNNING-TOTAL` ถูกประกาศเพียงครั้งเดียวใน WORKING-STORAGE SECTION แต่ paragraph `ADD-TEN`
ถูกเรียกผ่าน `PERFORM` ถึง 3 ครั้ง และทุกครั้งค่าจะสะสมต่อเนื่องจากค่าก่อนหน้า (10 → 20 → 30)
เพราะ `WS-RUNNING-TOTAL` เป็นตัวแปรเดียวกันตลอดทั้งโปรแกรม ไม่ได้ถูกสร้างใหม่ทุกครั้งที่ `PERFORM`
ไปยัง paragraph นั้น

### เปรียบเทียบกับแนวคิด Scope ในภาษาอื่น

โปรแกรมเมอร์ที่เคยเขียนภาษาอื่นมาก่อน (เช่น Python, Java) อาจคุ้นเคยกับแนวคิด "ตัวแปร local ในฟังก์ชัน
จะหายไปเมื่อฟังก์ชันจบการทำงาน" แต่ COBOL แบบดั้งเดิม**ไม่มีแนวคิดนี้ในระดับ paragraph** ตัวแปรทั้งหมด
ใน WORKING-STORAGE SECTION เป็น "Global" ในขอบเขตของโปรแกรมนั้นเสมอ ทุก paragraph มองเห็นและ
แก้ไขตัวแปรเดียวกันได้ทั้งหมด (ส่วน LOCAL-STORAGE SECTION ที่กล่าวถึงในขั้นตอนที่ 41 เป็นข้อยกเว้นพิเศษ
ที่จะเรียนภายหลัง)

### ข้อควรระวัง

- เพราะตัวแปรใน WORKING-STORAGE เป็นแบบ Global ทั้งหมด การตั้งชื่อที่ดีและการจัดกลุ่มที่เป็นระบบ
  (ตามที่จะสอนในขั้นตอนที่ 49) จึงสำคัญมากในโปรแกรมขนาดใหญ่ เพื่อป้องกันความสับสนว่าตัวแปรไหนถูกใช้
  ที่ไหนบ้าง
- ความสามารถในการ "จำ" ค่าข้ามการเรียก paragraph นี้เอง คือสาเหตุที่ทำให้แนวคิด `IS INITIAL PROGRAM`
  ใน Part 004 ขั้นตอนที่ 33 มีความสำคัญ เพราะมันควบคุมว่าพฤติกรรมการ "จำ" นี้จะเกิดขึ้นข้าม
  การเรียก `CALL` จากโปรแกรมอื่นด้วยหรือไม่

### แบบฝึกหัดที่ 42.1

**โจทย์**: จากตัวอย่างข้างต้น หากเพิ่ม `PERFORM ADD-TEN` อีก 2 ครั้งก่อน `STOP RUN.` (รวมเป็น 5 ครั้ง)
ค่าสุดท้ายของ `WS-RUNNING-TOTAL` ที่ยังไม่ได้แสดงผลจะเป็นเท่าไหร่

**เฉลย**: 50 (5 × 10) เพราะแต่ละครั้งที่ `PERFORM ADD-TEN` ค่าจะสะสมเพิ่มขึ้นทีละ 10 อย่างต่อเนื่อง
ไม่มีการรีเซ็ตกลับเป็น 0 ระหว่างการเรียกแต่ละครั้งเลย

---

## ขั้นตอนที่ 43: Level Numbers ภาพรวม (01, 05, 10, ..., 49, 66, 77, 88)

### Level Number คืออะไร

**Level Number** คือตัวเลข 2 หลักที่นำหน้าการประกาศตัวแปรทุกตัวใน DATA DIVISION ใช้บอก**ลำดับชั้น
ของโครงสร้างข้อมูล** ว่าตัวแปรใดเป็น "กลุ่มใหญ่" และตัวแปรใดเป็น "สมาชิกย่อย" ภายในกลุ่มนั้น

| Level Number | ความหมาย |
|---|---|
| `01` | ระดับบนสุดของ record หรือตัวแปรกลุ่ม (Group-level) — บังคับอยู่ใน Area A |
| `02`–`49` | ระดับย่อยภายใน record นั้น ยิ่งตัวเลขมากยิ่งอยู่ลึกเข้าไป (ไม่จำเป็นต้องเรียงติดกันทีละ 1 นิยมใช้ 05, 10, 15... เพื่อเผื่อแทรกภายหลัง) |
| `66` | ใช้กับ `RENAMES` clause เพื่อจัดกลุ่มตัวแปรใหม่จากตัวแปรที่มีอยู่แล้ว (จะกล่าวถึงเมื่อเนื้อหาเกี่ยวข้องมาถึง) |
| `77` | ตัวแปรเดี่ยว (standalone) ที่ไม่มีโครงสร้างกลุ่ม ไม่มีสมาชิกย่อย — บังคับอยู่ใน Area A |
| `88` | Condition Name — ตั้งชื่อเงื่อนไขให้กับค่าที่เป็นไปได้ของตัวแปรอื่น (จะสอนละเอียดใน Part 010) |

### ตัวอย่างที่ใช้ level number หลายแบบพร้อมกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. LEVEL-NUMBERS-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  EMPLOYEE-RECORD.
           05  EMP-ID             PIC 9(5)  VALUE 10023.
           05  EMP-NAME.
               10  EMP-FIRST-NAME PIC X(10) VALUE "MARIA".
               10  EMP-LAST-NAME  PIC X(10) VALUE "GARCIA".
           05  EMP-STATUS         PIC X     VALUE "A".
               88  EMP-IS-ACTIVE  VALUE "A".
               88  EMP-IS-RETIRED VALUE "R".

       77  WS-STANDALONE-COUNTER  PIC 9(3)  VALUE 0.

       PROCEDURE DIVISION.
           DISPLAY "01 = record level, 05/10 = subordinate,".
           DISPLAY "77 = standalone item, 88 = condition name.".
           DISPLAY "Employee ID   : " EMP-ID.
           DISPLAY "Employee name : " EMP-FIRST-NAME " " EMP-LAST-NAME.
           IF EMP-IS-ACTIVE
               DISPLAY "Status        : ACTIVE (via 88-level)"
           END-IF.
           DISPLAY "Standalone 77 : " WS-STANDALONE-COUNTER.
           STOP RUN.
```

**ผลลัพธ์**:

```
01 = record level, 05/10 = subordinate,
77 = standalone item, 88 = condition name.
Employee ID   : 10023
Employee name : MARIA      GARCIA
Status        : ACTIVE (via 88-level)
Standalone 77 : 000
```

### อธิบายลำดับชั้น

- `01 EMPLOYEE-RECORD.` — record ระดับบนสุด ไม่มี PIC ของตัวเอง เพราะเป็นแค่กลุ่มที่รวมสมาชิกย่อย
- `05 EMP-ID`, `05 EMP-NAME`, `05 EMP-STATUS` — สมาชิกระดับ 05 ภายใน `EMPLOYEE-RECORD`
- `10 EMP-FIRST-NAME`, `10 EMP-LAST-NAME` — สมาชิกระดับ 10 ที่อยู่ลึกเข้าไปอีกชั้นภายใน `EMP-NAME`
  (ซึ่งตัว `EMP-NAME` เองก็เป็นกลุ่มย่อยที่ไม่มี PIC เหมือนกับ `EMPLOYEE-RECORD`)
- `88 EMP-IS-ACTIVE`, `88 EMP-IS-RETIRED` — Condition Name ที่ผูกกับค่าเฉพาะของ `EMP-STATUS`
  ทำให้เขียน `IF EMP-IS-ACTIVE` แทนการเขียน `IF EMP-STATUS = "A"` ได้ อ่านง่ายกว่ามาก
- `77 WS-STANDALONE-COUNTER` — ตัวแปรเดี่ยวแยกออกมาต่างหาก ไม่เกี่ยวข้องกับ `EMPLOYEE-RECORD` เลย

### ทำไมนิยมกระโดดเลข 05, 10, 15 แทนที่จะเรียง 01, 02, 03

ธรรมเนียมที่ใช้กันแพร่หลายคือการกระโดดทีละ 5 (01, 05, 10, 15, ...) เพื่อ**เผื่อพื้นที่**ไว้สำหรับแทรก
level number ใหม่ในอนาคตโดยไม่ต้องแก้เลขเดิมทั้งหมด เช่น หากภายหลังต้องการเพิ่มฟิลด์ใหม่ระหว่าง
`EMP-ID` (05) กับ `EMP-NAME` (05) ก็สามารถแทรกด้วยเลข `07` ได้โดยไม่กระทบโครงสร้างเดิมเลย

### ข้อควรระวัง

- Level number ไม่จำเป็นต้องเรียงต่อเนื่องกันทีละ 1 แต่ต้อง**สอดคล้องกับลำดับชั้นที่ตั้งใจ** เสมอ
  (ตัวเลขที่มากกว่าต้องอยู่ "ลึกกว่า" ตัวเลขที่น้อยกว่าในกลุ่มเดียวกัน)
- `01` และ `77` เท่านั้นที่บังคับต้องอยู่ใน Area A (ตามที่เรียนใน Part 003 ขั้นตอนที่ 24)
- ตัวแปรกลุ่ม (เช่น `EMPLOYEE-RECORD`, `EMP-NAME`) **ไม่มี** PIC clause ของตัวเอง เพราะขนาดของมัน
  คำนวณจากผลรวมของสมาชิกย่อยทั้งหมดโดยอัตโนมัติ

### แบบฝึกหัดที่ 43.1

**โจทย์**: จากตัวอย่างข้างต้น `EMPLOYEE-RECORD` มีขนาดรวมกี่ไบต์ (ให้นับจากขนาด PIC ของสมาชิกย่อยระดับ
ล่างสุดทั้งหมด)

**เฉลย**: EMP-ID (5) + EMP-FIRST-NAME (10) + EMP-LAST-NAME (10) + EMP-STATUS (1) = 26 ไบต์
(EMP-NAME ไม่นับซ้ำเพราะเป็นกลุ่มที่รวม EMP-FIRST-NAME และ EMP-LAST-NAME ไว้แล้ว ไม่ได้เพิ่มไบต์เอง)

---

## ขั้นตอนที่ 44: การประกาศตัวแปรอย่างง่ายด้วย 01-level และ PIC

### รูปแบบพื้นฐานที่ใช้บ่อยที่สุด

ในโปรแกรมจำนวนมาก เราไม่จำเป็นต้องสร้างโครงสร้างกลุ่มที่ซับซ้อนเสมอไป การประกาศตัวแปรเดี่ยว ๆ ด้วย
`01` ตามด้วย `PIC` โดยตรงเป็นรูปแบบที่ใช้บ่อยที่สุดในโปรแกรมขนาดเล็กถึงกลาง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SIMPLE-VARIABLES-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-NAME       PIC X(15) VALUE "ROBERT LEE".
       01  WS-AGE                 PIC 9(3)  VALUE 34.
       01  WS-BALANCE             PIC 9(6)V99 VALUE 1500.75.

       PROCEDURE DIVISION.
           DISPLAY "Name    : " WS-CUSTOMER-NAME.
           DISPLAY "Age     : " WS-AGE.
           DISPLAY "Balance : " WS-BALANCE.
           STOP RUN.
```

**ผลลัพธ์**:

```
Name    : ROBERT LEE
Age     : 034
Balance : 001500.75
```

### อธิบาย

แต่ละบรรทัดในตัวอย่างนี้เป็นตัวแปร `01` ที่**เป็นอิสระจากกันโดยสมบูรณ์** ไม่มีความสัมพันธ์เชิงกลุ่มใด ๆ
ระหว่างกัน (ต่างจากตัวอย่างในขั้นตอนที่ 43 ที่ `01` ตัวเดียวมีสมาชิกย่อยหลายตัว) รูปแบบนี้เหมาะสำหรับ
ตัวแปรที่ไม่เกี่ยวข้องกันโดยตรง เช่น ชื่อลูกค้า, อายุ, ยอดเงินคงเหลือ ที่แม้จะอยู่ใน "บริบท" เดียวกัน
(ข้อมูลลูกค้าคนหนึ่ง) แต่ผู้เขียนเลือกไม่จัดกลุ่มไว้ในตัวอย่างนี้เพื่อความเรียบง่าย

รายละเอียดของ PICTURE clause (`PIC X(15)`, `PIC 9(3)`, `PIC 9(6)V99`) จะสอนอย่างละเอียดครบถ้วนใน
**Part 006** ขอให้เข้าใจในตอนนี้เพียงว่า `X` หมายถึงข้อความทั่วไป, `9` หมายถึงตัวเลข, และตัวเลขในวงเล็บ
บอกจำนวนหลัก

### ข้อควรระวัง

- แม้จะเขียนแยกเป็น `01` หลายตัวได้ แต่ถ้าข้อมูลมีความสัมพันธ์กันเชิงตรรกะ (เช่น เป็นข้อมูลของ "ลูกค้า
  คนเดียวกัน") การจัดกลุ่มด้วยโครงสร้าง group ตามขั้นตอนที่ 43 มักจะดีกว่าในระยะยาว เพราะช่วยให้
  MOVE ข้อมูลทั้งกลุ่มได้ในคำสั่งเดียว (จะเห็นตัวอย่างในขั้นตอนที่ 45)

### แบบฝึกหัดที่ 44.1

**โจทย์**: จงประกาศตัวแปร `01` สามตัวสำหรับเก็บ รหัสสินค้า (6 ตัวอักษร), จำนวนคงเหลือในสต็อก (ตัวเลข
4 หลัก), และราคาต่อหน่วย (ตัวเลข 5 หลัก มีทศนิยม 2 ตำแหน่ง)

**เฉลย**:

```cobol
       01  WS-PRODUCT-CODE        PIC X(6).
       01  WS-STOCK-QTY           PIC 9(4).
       01  WS-UNIT-PRICE          PIC 9(5)V99.
```

---

## ขั้นตอนที่ 45: Group Items กับ Elementary Items — MOVE ทั้งกลุ่มในคำสั่งเดียว

### นิยาม

- **Elementary Item** คือตัวแปรที่มี `PIC` เป็นของตัวเอง ถือเป็นหน่วยข้อมูลที่เล็กที่สุดที่นำมาใช้งานได้
- **Group Item** คือตัวแปรที่**ไม่มี** `PIC` เป็นของตัวเอง แต่ประกอบด้วย elementary item หลายตัวรวมกัน
  เมื่อ COBOL มองที่ group item ทั้งก้อน จะปฏิบัติกับมันเสมือนเป็น**ข้อความ (alphanumeric) ก้อนเดียว**
  โดยอัตโนมัติ ไม่ว่าสมาชิกภายในจะเป็นตัวเลขหรือตัวอักษรก็ตาม

### ตัวอย่างที่แสดงพลังของ Group Item

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. GROUP-ELEMENTARY-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE-TODAY.
           05  WS-YEAR            PIC 9(4).
           05  WS-MONTH           PIC 9(2).
           05  WS-DAY             PIC 9(2).

       01  WS-DATE-COPY           PIC X(8).

       PROCEDURE DIVISION.
           MOVE 2026 TO WS-YEAR.
           MOVE 9    TO WS-MONTH.
           MOVE 26   TO WS-DAY.

           DISPLAY "Elementary items : " WS-YEAR "-" WS-MONTH
                   "-" WS-DAY.

           MOVE WS-DATE-TODAY TO WS-DATE-COPY.
           DISPLAY "Whole group moved as one text field: "
                   WS-DATE-COPY.
           STOP RUN.
```

**ผลลัพธ์**:

```
Elementary items : 2026-09-26
Whole group moved as one text field: 20260926
```

### อธิบายการทำงาน

- เราสามารถ `MOVE` ค่าเข้าไปที่ elementary item แต่ละตัว (`WS-YEAR`, `WS-MONTH`, `WS-DAY`)
  แยกกันได้ตามปกติ เพราะแต่ละตัวมี `PIC` เป็นของตัวเอง
- แต่เมื่อเราเขียน `MOVE WS-DATE-TODAY TO WS-DATE-COPY` ซึ่ง `WS-DATE-TODAY` เป็น**กลุ่ม (group)**
  ที่รวม `WS-YEAR` + `WS-MONTH` + `WS-DAY` เข้าด้วยกัน COBOL จะมองกลุ่มนี้เป็นข้อความยาว 8 ตัวอักษร
  (4+2+2) แล้วคัดลอกทั้งก้อนไปยัง `WS-DATE-COPY` ในครั้งเดียว ได้ผลลัพธ์ `20260926`
  โดยไม่ต้องเขียน MOVE แยกทีละฟิลด์ถึง 3 คำสั่ง

### ทำไม Group MOVE ถึงมีประโยชน์มาก

ในการเขียนโปรแกรมประมวลผล record จำนวนมาก (เช่น อ่านข้อมูลจากไฟล์ทีละ record) การ MOVE ทั้ง group
ในคำสั่งเดียวช่วยลดจำนวนบรรทัดโค้ดได้มาก และยังทำงาน**เร็วกว่า**การ MOVE ทีละฟิลด์ในหลายกรณี เพราะ
เป็นการคัดลอกหน่วยความจำต่อเนื่องกันเพียงครั้งเดียว

### ข้อควรระวัง

- เมื่อ MOVE ทั้ง group ข้อมูลจะถูกปฏิบัติเป็น**alphanumeric เสมอ** แม้สมาชิกภายในจะเป็นตัวเลขก็ตาม
  ซึ่งหมายความว่ากฎการจัดตำแหน่ง (ชิดซ้าย/ชิดขวา, เติมด้วยศูนย์/เว้นวรรค) จะเป็นไปตามกฎของ
  alphanumeric MOVE ไม่ใช่กฎของ numeric MOVE (รายละเอียดกฎการ MOVE ทั้งสองแบบจะสอนอย่างละเอียด
  ใน Part 008)
- ขนาดของ group ปลายทางและต้นทางต้อง**เท่ากันพอดี** หรือ COBOL จะตัด/เติมช่องว่างตามกฎ alphanumeric
  ตามปกติ ซึ่งอาจทำให้ข้อมูลบางส่วนหายไปถ้าขนาดไม่ตรงกัน

### แบบฝึกหัดที่ 45.1

**โจทย์**: หาก `WS-DATE-COPY` มีขนาดเพียง `PIC X(6)` แทนที่จะเป็น `PIC X(8)` ผลลัพธ์ของการ
`MOVE WS-DATE-TODAY TO WS-DATE-COPY` จะเป็นอย่างไร

**เฉลยแนวทาง**: เนื่องจากการ MOVE ระหว่าง group ถือเป็น alphanumeric MOVE ค่าต้นทาง `20260926`
(8 ตัวอักษร) จะถูกตัดให้เหลือ 6 ตัวอักษรแรกเท่านั้น ได้ผลลัพธ์ `202609` โดยตัวเลข `26` ที่แทนวันที่
จะหายไป ซึ่งเป็นตัวอย่างของ**การตัดข้อมูลอย่างเงียบ ๆ**ตามกฎ PIC X ที่เรียนไปแล้วใน Part 006
(หากเรียนมาถึงจุดนั้นแล้ว) — ควรระวังให้ขนาดปลายทางเท่ากับหรือมากกว่าต้นทางเสมอ

---

## ขั้นตอนที่ 46: VALUE Clause — การกำหนดค่าเริ่มต้น

### VALUE ทำหน้าที่กำหนดค่าตอนเริ่มโปรแกรม

`VALUE` clause กำหนด**ค่าเริ่มต้น**ให้กับตัวแปรตั้งแต่ตอนที่โปรแกรมเริ่มทำงาน (หรือตอนที่ WORKING-STORAGE
ถูกจัดสรรหน่วยความจำ) หากไม่ใส่ `VALUE` ไว้ ค่าภายในตัวแปรนั้น**ไม่มีการรับประกันมาตรฐาน**ว่าจะเป็นอะไร

### ทดสอบเปรียบเทียบ: มี VALUE กับไม่มี VALUE

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. VALUE-CLAUSE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-WITH-VALUE          PIC 9(4) VALUE 500.
       01  WS-NO-VALUE-NUMERIC    PIC 9(4).
       01  WS-NO-VALUE-TEXT       PIC X(10).

       PROCEDURE DIVISION.
           DISPLAY "With VALUE 500       : [" WS-WITH-VALUE "]".
           DISPLAY "No VALUE, numeric 9(4): [" WS-NO-VALUE-NUMERIC
                   "]".
           DISPLAY "No VALUE, text X(10)  : [" WS-NO-VALUE-TEXT "]".
           DISPLAY "Never assume a field without VALUE starts".
           DISPLAY "at zero or spaces -- always initialize it".
           DISPLAY "yourself before relying on its content.".
           STOP RUN.
```

**ผลลัพธ์จริงจากการรันบนเครื่องนี้ (GnuCOBOL 4.0-early)**:

```
With VALUE 500       : [0500]
No VALUE, numeric 9(4): [0000]
No VALUE, text X(10)  : [          ]
Never assume a field without VALUE starts
at zero or spaces -- always initialize it
yourself before relying on its content.
```

### ทำไมถึงเขียนว่า "ไม่มีการรับประกันมาตรฐาน" ทั้งที่ผลลัพธ์ออกมาเป็น 0000 และช่องว่างพอดี

นี่คือจุดสำคัญที่ต้องระวังมาก: ผลลัพธ์ข้างต้นเป็นพฤติกรรมที่ GnuCOBOL รุ่นนี้เลือกทำ (initialize
หน่วยความจำ static ให้เป็นศูนย์/ช่องว่างให้อัตโนมัติ) แต่**มาตรฐาน COBOL ไม่ได้บังคับ**ว่าคอมไพเลอร์ทุกตัว
หรือทุกสภาพแวดล้อมต้องทำแบบนี้เสมอไป ในระบบ Mainframe จริงหรือคอมไพเลอร์อื่น ตัวแปรที่ไม่มี `VALUE`
**อาจมีขยะ (garbage)** หลงเหลือจากการใช้งานหน่วยความจำก่อนหน้าอยู่ก็ได้ โดยเฉพาะถ้าโปรแกรมถูกเรียก
ซ้ำหลายครั้งในสภาพแวดล้อมที่ไม่รีเซ็ตหน่วยความจำให้ (เช่นบางกรณีของ CICS Transaction)

### หลักปฏิบัติที่ปลอดภัยที่สุด

**กำหนด VALUE ให้กับทุกตัวแปรที่มีความสำคัญเสมอ** โดยเฉพาะตัวแปรตัวนับ (counter), ตัวแปรสะสม (accumulator),
และ flag ต่าง ๆ อย่าไว้ใจว่าตัวแปรจะเริ่มต้นเป็น 0 หรือช่องว่างโดยอัตโนมัติเสมอไป แม้ในทางปฏิบัติจริงกับ
GnuCOBOL รุ่นนี้จะเป็นเช่นนั้นก็ตาม เพราะโค้ดที่เขียนควรพกพาได้ (portable) ข้ามคอมไพเลอร์และสภาพแวดล้อม

### ข้อควรระวัง

- อย่าเขียนโปรแกรมที่พึ่งพา "ค่าเริ่มต้นที่ไม่ได้ระบุ" โดยเด็ดขาด แม้จะทดสอบแล้วว่าได้ผลลัพธ์ที่ต้องการ
  บนเครื่องพัฒนาของคุณ เพราะเมื่อนำไปรันบนสภาพแวดล้อมอื่นอาจได้ผลลัพธ์ต่างออกไป
- `VALUE` ของ group item จะกำหนดให้กับสมาชิกย่อยทั้งหมดพร้อมกันได้เฉพาะกรณีพิเศษเท่านั้น
  (เช่น `VALUE SPACES` หรือ `VALUE ZEROS` ที่ใช้ได้กับทั้ง group) การกำหนด literal เฉพาะเจาะจง
  ให้กับ group โดยตรงมักไม่ทำในทางปฏิบัติ นิยมกำหนด VALUE ที่ระดับ elementary item แต่ละตัวแทน

### แบบฝึกหัดที่ 46.1

**โจทย์**: จงอธิบายว่าทำไมนักพัฒนาที่ระมัดระวังจึงไม่ควรเขียนโค้ดที่พึ่งพาพฤติกรรม "ตัวแปรตัวเลขที่ไม่มี
VALUE จะเป็น 0 เสมอ" แม้จะทดสอบแล้วว่าเป็นจริงบนเครื่องของตน

**เฉลยแนวทาง**: เพราะพฤติกรรมนี้ไม่ได้ถูกรับประกันไว้ในมาตรฐาน COBOL อย่างเป็นทางการ เป็นเพียง
รายละเอียดการ implement ของคอมไพเลอร์/runtime แต่ละตัวที่อาจแตกต่างกัน หากย้ายโค้ดไปรันบนคอมไพเลอร์อื่น
เวอร์ชันอื่น หรือสภาพแวดล้อมอื่น (เช่น Mainframe จริง หรือโปรแกรมที่ถูกเรียกซ้ำในหน่วยความจำเดิม)
อาจพบว่าตัวแปรมีค่าขยะที่ไม่คาดคิด ทำให้เกิดบั๊กที่ยากต่อการวินิจฉัยมาก การกำหนด VALUE อย่างชัดเจนเสมอ
คือวิธีที่ปลอดภัยและพกพาได้ (portable) มากกว่า

---

## ขั้นตอนที่ 47: 77-Level — ตัวแปรเดี่ยวที่ไม่มีสมาชิกย่อย

### กฎของ 77-Level

Level number `77` ใช้สำหรับประกาศ**ตัวแปรเดี่ยว (Standalone Item)** ที่รับประกันว่า**จะไม่มีวันมีสมาชิก
ย่อย**ได้เลย ต่างจาก `01` ที่สามารถเป็นได้ทั้งตัวแปรเดี่ยวหรือกลุ่มก็ได้ 77-level จึงสื่อความหมายให้ผู้อ่าน
โค้ดรู้ทันทีว่า "นี่คือค่าเดี่ยว ๆ ไม่ต้องมองหาสมาชิกย่อยข้างใน"

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. LEVEL-77-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       77  WS-ORDER-COUNT         PIC 9(5) VALUE 0.
       77  WS-GRAND-TOTAL         PIC 9(9)V99 VALUE 0.

       PROCEDURE DIVISION.
           ADD 1 TO WS-ORDER-COUNT.
           ADD 199.99 TO WS-GRAND-TOTAL.
           DISPLAY "Level 77 items are independent, standalone".
           DISPLAY "items -- they cannot own subordinate items.".
           DISPLAY "Order count  : " WS-ORDER-COUNT.
           DISPLAY "Grand total  : " WS-GRAND-TOTAL.
           STOP RUN.
```

**ผลลัพธ์**:

```
Level 77 items are independent, standalone
items -- they cannot own subordinate items.
Order count  : 00001
Grand total  : 000000199.99
```

### พิสูจน์กฎด้วยการทำผิดตั้งใจ

ลองประกาศ `77` แล้วพยายามใส่สมาชิกย่อย (ซึ่งเป็นสิ่งต้องห้าม):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. LEVEL-77-BAD-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       77  WS-BAD-GROUP.
           05  WS-BAD-PART        PIC X(5).

       PROCEDURE DIVISION.
           DISPLAY "This should fail to compile.".
           STOP RUN.
```

**ผลลัพธ์การคอมไพล์จริง (error)**:

```
s47b-level-77-bad.cob:7: error: level number must begin with 01 or 77
s47b-level-77-bad.cob:6: error: PICTURE clause required for 'WS-BAD-GROUP'
```

คอมไพเลอร์ปฏิเสธทันที เพราะเมื่อประกาศ `WS-BAD-GROUP` เป็น `77` แต่ไม่ใส่ PIC (เพราะตั้งใจให้เป็นกลุ่ม)
คอมไพเลอร์บังคับว่าตัวถัดไปในลำดับต้องขึ้นต้นด้วย `01` หรือ `77` เท่านั้น (ไม่ใช่ `05`) และยังฟ้องอีกว่า
`WS-BAD-GROUP` ที่เป็น 77-level ต้องมี PICTURE clause เสมอ เพราะ 77-level ห้ามเป็นกลุ่มโดยเด็ดขาด

### เมื่อไหร่ควรเลือกใช้ 77 แทน 01

ในทางปฏิบัติ โปรแกรมเมอร์จำนวนมากเลือกใช้ `01` สำหรับตัวแปรเดี่ยวเช่นกัน (ซึ่งก็ทำได้ถูกต้องตามไวยากรณ์)
แต่การใช้ `77` อย่างตั้งใจสำหรับตัวแปรที่**ไม่มีวันเป็นกลุ่ม**ช่วยสื่อความหมายให้ชัดเจนขึ้น และในระบบ
Legacy จริงจำนวนมาก โปรแกรมเมอร์รุ่นก่อนนิยมใช้ 77-level แยกไว้เป็นสัดส่วนต่างหากจากกลุ่ม 01-level
ที่ซับซ้อนกว่า เพื่อให้อ่านแยกแยะได้ง่ายว่าส่วนไหนคือ "ค่าเดี่ยว ๆ ง่าย ๆ" และส่วนไหนคือ "โครงสร้าง record
ที่ซับซ้อน"

### ข้อควรระวัง

- 77-level เช่นเดียวกับ 01-level ต้องอยู่ใน Area A เสมอ (ตามที่เรียนใน Part 003 ขั้นตอนที่ 24)
- ห้ามใส่ level number ที่มากกว่า 77 ตามหลัง 77-level ทันที (เช่น พยายามใส่ 88-level condition name
  ตามหลัง 77 โดยตรงแบบไม่มี PIC ก่อน) — ต้องมี PIC ของ 77-level นั้นก่อนเสมอ

### แบบฝึกหัดที่ 47.1

**โจทย์**: จงอธิบายว่าทำไม error message `level number must begin with 01 or 77` ถึงปรากฏขึ้น
เมื่อพยายามใส่ level `05` ตามหลัง 77-level ที่ไม่มี PIC

**เฉลย**: เพราะเมื่อ `WS-BAD-GROUP` ถูกประกาศเป็น level 77 โดยไม่มี PIC คอมไพเลอร์จะตีความว่าผู้เขียน
กำลังพยายามสร้าง "กลุ่ม" ขึ้นจาก 77-level ซึ่งเป็นสิ่งต้องห้ามตามกฎของ COBOL (77-level ต้องเป็นตัวแปร
เดี่ยวที่มี PIC เสมอ ห้ามมีสมาชิกย่อย) คอมไพเลอร์จึงปฏิเสธ item ระดับ 05 ที่ตามมา เพราะไม่มี level 01
หรือ 77 ที่ถูกต้องมารองรับให้เป็น "หัวกลุ่ม" ของมัน

---

## ขั้นตอนที่ 48: FILLER — การเว้นพื้นที่ที่ไม่ต้องอ้างอิงชื่อ

### FILLER คืออะไร

`FILLER` เป็นชื่อพิเศษที่ใช้แทนตำแหน่งของ elementary item ที่**เราไม่จำเป็นต้องอ้างอิงถึงในโค้ดเลย**
มักใช้เพื่อจองพื้นที่สำหรับการจัดวางข้อมูลให้ตรงคอลัมน์ (เช่น ในรายงานหรือไฟล์ที่มีรูปแบบตายตัว)
โดยไม่ต้องคิดชื่อตัวแปรที่มีความหมายให้กับทุกช่องว่างที่ไม่ได้ใช้งานจริง

### ตัวอย่าง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. FILLER-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-REPORT-LINE.
           05  FILLER             PIC X(5)  VALUE "NAME:".
           05  WS-NAME-OUT        PIC X(15) VALUE "ALAN TURING".
           05  FILLER             PIC X(5)  VALUE "AGE: ".
           05  WS-AGE-OUT         PIC 9(3)  VALUE 41.

       PROCEDURE DIVISION.
           DISPLAY "FILLER reserves space we never need to".
           DISPLAY "reference by name -- only for spacing".
           DISPLAY "or layout, such as fixed report columns:".
           DISPLAY WS-REPORT-LINE.
           STOP RUN.
```

**ผลลัพธ์**:

```
FILLER reserves space we never need to
reference by name -- only for spacing
or layout, such as fixed report columns:
NAME:ALAN TURING    AGE: 041
```

### อธิบาย

`WS-REPORT-LINE` เป็น group item ที่ประกอบด้วยฉลาก "NAME:" (FILLER คงที่), ชื่อจริง (`WS-NAME-OUT`),
ฉลาก "AGE: " (FILLER คงที่อีกตัว), และอายุ (`WS-AGE-OUT`) เมื่อ `DISPLAY WS-REPORT-LINE` ทั้ง group
COBOL จะแสดงทุกสมาชิกเรียงต่อกันเป็นบรรทัดเดียว ได้ผลลัพธ์ที่มีรูปแบบคงที่สวยงามโดยไม่ต้องเขียน
`DISPLAY` แยกทีละส่วนแล้วมาต่อกันเอง

### จุดสำคัญ: FILLER ใช้ชื่อซ้ำกันได้ไม่จำกัด

สังเกตว่าในตัวอย่างมี `FILLER` ปรากฏถึง 2 ครั้งในกลุ่มเดียวกัน ซึ่งเป็นไปได้ปกติ เพราะ `FILLER`
**ไม่ใช่ชื่อจริงที่ต้องไม่ซ้ำกัน** (Unique) เหมือนตัวแปรทั่วไป มันเป็นคำสงวนพิเศษที่บอกคอมไพเลอร์ว่า
"ช่องนี้ไม่มีใครอ้างอิงถึงโดยตรง จะซ้ำกี่ตัวก็ได้ในโปรแกรมเดียวกัน"

### ทำไมยังต้องมีอยู่ในยุคปัจจุบัน

แม้การเขียนโปรแกรมสมัยใหม่จะนิยมตั้งชื่อให้กับทุกอย่างเพื่อความชัดเจน แต่ `FILLER` ยังมีประโยชน์มากใน
สถานการณ์ที่ต้อง**จองพื้นที่ตามข้อกำหนดของไฟล์ Legacy ที่ตายตัว** เช่น การอ่านไฟล์ที่มี record length
คงที่ 200 ไบต์ แต่โปรแกรมของเราสนใจใช้งานแค่บางฟิลด์เท่านั้น ฟิลด์ที่เหลือที่ไม่เกี่ยวข้องก็ประกาศเป็น
`FILLER` เพื่อให้ขนาดรวมของ record ตรงกับไฟล์จริง โดยไม่ต้องตั้งชื่อความหมายให้กับทุกไบต์ที่ไม่ได้ใช้

### ข้อควรระวัง

- เนื่องจาก `FILLER` ไม่มีชื่อที่อ้างอิงได้ **จึงไม่สามารถ MOVE ข้อมูลเข้า/ออกจาก FILLER โดยตรงในโค้ด
  ได้เลย** ต้องเข้าถึงผ่านการ MOVE ทั้ง group เท่านั้น (หรือใช้ `REDEFINES` ซึ่งจะสอนใน Part 022)
- GnuCOBOL รุ่นใหม่บางรุ่นยังอนุญาตให้**ละคำว่า `FILLER` ไปเลย** (เว้นชื่อว่างไว้) ก็ถือว่าเป็น FILLER
  โดยปริยาย แต่หลักสูตรนี้จะเขียนคำว่า `FILLER` ให้ชัดเจนเสมอเพื่อความสอดคล้องกับโค้ด Legacy ส่วนใหญ่

### แบบฝึกหัดที่ 48.1

**โจทย์**: จงอธิบายว่าทำไมการมี `FILLER` ซ้ำกันหลายตัวในกลุ่มเดียวกันจึงไม่ทำให้เกิด "ชื่อซ้ำกัน" error
เหมือนกับตัวแปรทั่วไป

**เฉลย**: เพราะ `FILLER` เป็นคำสงวนพิเศษของ COBOL ที่คอมไพเลอร์ปฏิบัติต่างจากชื่อ Identifier ทั่วไป
โดยเจตนา มันถูกออกแบบมาให้เป็น "ช่องว่างที่ไม่มีตัวตนในเชิงการอ้างอิง" ดังนั้นกฎ "ชื่อต้องไม่ซ้ำกัน"
ที่ใช้กับตัวแปรทั่วไปจึงไม่ถูกนำมาบังคับใช้กับ `FILLER` เลย

---

## ขั้นตอนที่ 49: การจัดระเบียบ WORKING-STORAGE ที่ดี (Naming Conventions)

### หลักการจัดระเบียบที่ใช้ในอุตสาหกรรมจริง

เมื่อโปรแกรมมีขนาดใหญ่ขึ้น (หลายร้อยหรือหลายพันตัวแปร) การจัดระเบียบ WORKING-STORAGE ให้เป็นระบบ
มีความสำคัญมาก หลักการที่นิยมใช้กันแพร่หลายมีดังนี้:

1. **ใช้ prefix ที่สื่อความหมาย** เช่น `WS-` (Working-Storage ทั่วไป), `WS-FLAG-` (สำหรับ flag/switch)
2. **จัดกลุ่มฟิลด์ที่เกี่ยวข้องกันไว้ภายใต้ 01-level เดียวกัน** แทนที่จะกระจายเป็น 01-level แยกกันหมด
3. **ใช้ 88-level ควบคู่กับฟิลด์สถานะ** เพื่อให้โค้ดใน PROCEDURE DIVISION อ่านเหมือนภาษาอังกฤษ
4. **จัดกลุ่มตามหน้าที่การใช้งาน** เช่น แยกกลุ่มข้อมูลนำเข้า (input), ข้อมูลคำนวณระหว่างทาง
   (working/intermediate), และข้อมูลผลลัพธ์ (output) ออกจากกันให้ชัดเจน

### ตัวอย่างที่สาธิตหลักการทั้งหมด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. NAMING-CONVENTIONS-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> Group related fields under one 01-level record with a
      *> clear prefix so the whole record reads like a sentence.
       01  WS-INVOICE-RECORD.
           05  WS-INVOICE-NUMBER  PIC 9(6)    VALUE 100234.
           05  WS-INVOICE-DATE.
               10  WS-INV-YEAR    PIC 9(4)    VALUE 2026.
               10  WS-INV-MONTH   PIC 9(2)    VALUE 9.
               10  WS-INV-DAY     PIC 9(2)    VALUE 26.
           05  WS-INVOICE-AMOUNT  PIC 9(7)V99 VALUE 4599.00.

      *> Flags and switches are grouped separately and use
      *> a WS-FLAG-* naming pattern with matching 88-levels.
       01  WS-INVOICE-STATUS      PIC X       VALUE "P".
           88  WS-STATUS-PAID     VALUE "P".
           88  WS-STATUS-PENDING  VALUE "N".

       PROCEDURE DIVISION.
           DISPLAY "Invoice #  : " WS-INVOICE-NUMBER.
           DISPLAY "Date       : " WS-INV-YEAR "-" WS-INV-MONTH
                   "-" WS-INV-DAY.
           DISPLAY "Amount     : " WS-INVOICE-AMOUNT.
           IF WS-STATUS-PAID
               DISPLAY "Status     : PAID"
           END-IF.
           STOP RUN.
```

**ผลลัพธ์**:

```
Invoice #  : 100234
Date       : 2026-09-26
Amount     : 0004599.00
Status     : PAID
```

### อธิบายประโยชน์เชิงการดูแลรักษา

ลองเปรียบเทียบการอ่าน `IF WS-STATUS-PAID` (จากตัวอย่างนี้) กับ `IF WS-INVOICE-STATUS = "P"`
(ถ้าไม่ใช้ 88-level) — แบบแรกอ่านแล้วเข้าใจทันทีว่า "ถ้าสถานะคือจ่ายแล้ว" โดยไม่ต้องจำว่า `"P"`
หมายถึงอะไร ในขณะที่แบบหลังต้องเปิดไปดูว่า `"P"` ถูกนิยามไว้ว่าหมายถึง "Paid" ที่ไหนสักแห่ง
นี่คือพลังของการตั้งชื่อที่ดีร่วมกับ 88-level ที่ทำให้โค้ด COBOL อ่านได้ใกล้เคียงภาษาอังกฤษตามที่
Grace Hopper ตั้งใจไว้ตั้งแต่ต้น (ตามที่กล่าวถึงใน Part 001)

### ธรรมเนียมการเยื้องบรรทัด (Indentation) ที่แนะนำ

นอกจากการตั้งชื่อ การเยื้องบรรทัดให้สอดคล้องกับ level number ก็สำคัญไม่แพ้กัน:
- Level 01 เริ่มที่ตำแหน่งซ้ายสุดของ Area A
- Level ที่ลึกกว่าแต่ละชั้น เยื้องเข้าไปทีละ 4 ช่องว่างอย่างสม่ำเสมอ (ตามที่เห็นในทุกตัวอย่างของหลักสูตรนี้)

การเยื้องที่สม่ำเสมอทำให้มองเห็นโครงสร้างลำดับชั้นของข้อมูลได้ทันทีด้วยสายตา แม้จะเปิดไฟล์ด้วย
Text Editor ธรรมดาที่ไม่มี Syntax Highlighting ก็ตาม

### ข้อควรระวัง

- อย่าตั้งชื่อสั้นเกินไปจนสื่อความหมายไม่ได้ (เช่น `WS-X`, `WS-TMP1`) ในโปรแกรมขนาดใหญ่ที่มีตัวแปร
  หลายร้อยตัว ชื่อที่ไม่สื่อความหมายจะทำให้การดูแลรักษาโค้ดยากขึ้นมากในระยะยาว
- อย่าตั้งชื่อยาวเกินไปจนอ่านยากหรือทำให้บรรทัดเกิน 72 คอลัมน์ (เชื่อมโยงกับกฎคอลัมน์ที่เรียนใน
  Part 003) ต้องหาจุดสมดุลระหว่างความชัดเจนกับความกระชับ

### แบบฝึกหัดที่ 49.1

**โจทย์**: จงตั้งชื่อ 88-level เพิ่มเติมสำหรับ `WS-INVOICE-STATUS` ที่แทนค่า `"C"` หมายถึง "Cancelled"
(ยกเลิก)

**เฉลย**:

```cobol
           88  WS-STATUS-CANCELLED VALUE "C".
```

เพิ่มบรรทัดนี้ต่อจาก `88 WS-STATUS-PENDING VALUE "N".` ในกลุ่มเดียวกัน จากนั้นจะสามารถเขียน
`IF WS-STATUS-CANCELLED` ในโค้ดได้ทันที

---

## ขั้นตอนที่ 50: ตัวอย่างโปรแกรมรวม และข้อผิดพลาดที่พบบ่อยใน WORKING-STORAGE

### โปรแกรมรวมที่ใช้ทุกแนวคิดจาก Part นี้

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. WS-COMBINED-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STUDENT-RECORD.
           05  WS-STUDENT-ID      PIC 9(5)    VALUE 55012.
           05  WS-STUDENT-NAME    PIC X(20)   VALUE "SUDA KITTISAK".
           05  WS-STUDENT-SCORES.
               10  WS-SCORE-MATH  PIC 9(3)    VALUE 88.
               10  WS-SCORE-ENG   PIC 9(3)    VALUE 92.
       77  WS-SCORE-AVERAGE       PIC 9(3)V99 VALUE 0.
       01  FILLER                 PIC X(1)    VALUE SPACE.

       PROCEDURE DIVISION.
           COMPUTE WS-SCORE-AVERAGE =
               (WS-SCORE-MATH + WS-SCORE-ENG) / 2.
           DISPLAY "Student ID   : " WS-STUDENT-ID.
           DISPLAY "Student Name : " WS-STUDENT-NAME.
           DISPLAY "Math Score   : " WS-SCORE-MATH.
           DISPLAY "Eng Score    : " WS-SCORE-ENG.
           DISPLAY "Average      : " WS-SCORE-AVERAGE.
           STOP RUN.
```

**ผลลัพธ์**:

```
Student ID   : 55012
Student Name : SUDA KITTISAK
Math Score   : 088
Eng Score    : 092
Average      : 090.00
```

โปรแกรมนี้รวม: group item หลายชั้น (`WS-STUDENT-RECORD` → `WS-STUDENT-SCORES` → คะแนนแต่ละวิชา),
77-level แยกต่างหาก (`WS-SCORE-AVERAGE`), และ `FILLER` ที่ระดับ 01 (ซึ่งก็ทำได้เช่นกัน ไม่จำเป็นต้อง
อยู่ในกลุ่มเสมอไป)

### ข้อผิดพลาดที่พบบ่อย 1: ตั้งชื่อตัวแปรซ้ำกันโดยไม่ตั้งใจ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DUPLICATE-NAME-BUG.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-AMOUNT              PIC 9(5) VALUE 100.
       01  WS-AMOUNT              PIC 9(5) VALUE 200.

       PROCEDURE DIVISION.
           DISPLAY WS-AMOUNT.
           STOP RUN.
```

**ผลลัพธ์การคอมไพล์จริง (error)**:

```
s50b-duplicate-name-bug.cob:10: error: 'WS-AMOUNT' is ambiguous; needs qualification
s50b-duplicate-name-bug.cob:6: error: 'WS-AMOUNT' defined here
s50b-duplicate-name-bug.cob:7: error: 'WS-AMOUNT' defined here
```

คอมไพเลอร์แจ้งว่าชื่อ `WS-AMOUNT` **กำกวม (ambiguous)** เพราะประกาศซ้ำถึง 2 ครั้งในระดับเดียวกัน
เมื่อ PROCEDURE DIVISION พยายามอ้างอิงถึง `WS-AMOUNT` คอมไพเลอร์ไม่รู้ว่าหมายถึงตัวไหน จึงปฏิเสธทันที
**วิธีแก้**: เปลี่ยนชื่อให้ไม่ซ้ำกัน เช่น `WS-AMOUNT-1` และ `WS-AMOUNT-2` หรือลบตัวที่ไม่ได้ใช้ออก

### ข้อผิดพลาดที่พบบ่อย 2: จัดลำดับ Level Number ไม่สอดคล้องกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. LEVEL-SEQUENCE-BUG.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-RECORD.
           10  WS-CUSTOMER-NAME   PIC X(15).
           05  WS-CUSTOMER-CITY   PIC X(15).

       PROCEDURE DIVISION.
           DISPLAY "This level order is confusing to the reader.".
           STOP RUN.
```

**ผลลัพธ์การคอมไพล์จริง (error)**:

```
s50c-level-sequence-bug.cob:8: error: no previous data item of level 05
```

ปัญหาเกิดจากการที่ `WS-CUSTOMER-NAME` ถูกประกาศเป็น level `10` ก่อน แล้วตามด้วย `WS-CUSTOMER-CITY`
ที่เป็น level `05` ซึ่ง**ตื้นกว่า** — COBOL คาดหวังว่า level number ที่ตื้นกว่าจะต้องปรากฏ**ก่อน**
level ที่ลึกกว่าเสมอในการนิยามโครงสร้างลำดับชั้น เมื่อเจอ level 05 ที่ไม่มี "ที่มาที่ไป" ที่ถูกต้อง
(ไม่มี level ที่ตื้นกว่าหรือเท่ากันมาก่อนหน้าเพื่อให้มันสังกัดอยู่ในกลุ่มเดียวกันอย่างสมเหตุสมผล)
จึงเกิด error **วิธีแก้**: สลับลำดับให้ `WS-CUSTOMER-CITY` (05) มาก่อน หรือปรับให้ทั้งสองฟิลด์อยู่ใน
level เดียวกัน (เช่นทั้งคู่เป็น 05)

### ตารางสรุป Error ที่พบบ่อยใน WORKING-STORAGE SECTION

| Error message | สาเหตุ | วิธีแก้ |
|---|---|---|
| `'X' is ambiguous; needs qualification` | ประกาศชื่อตัวแปรซ้ำกันในระดับเดียวกัน | เปลี่ยนชื่อให้ไม่ซ้ำ หรือใช้ qualification (`OF`/`IN`) |
| `no previous data item of level N` | จัดลำดับ level number ไม่สอดคล้องกับโครงสร้างลำดับชั้น | เรียง level number จากตื้นไปลึกให้ถูกต้อง |
| `level number must begin with 01 or 77` | พยายามใส่สมาชิกย่อยให้กับ 77-level | เปลี่ยนเป็น 01-level หากต้องการให้เป็นกลุ่ม |
| `PICTURE clause required for 'X'` | ลืมใส่ PIC ให้กับ elementary item | เพิ่ม PIC ให้ครบทุก elementary item |

### ข้อควรระวัง

- Error เกี่ยวกับ "ambiguous" ไม่ได้แปลว่าโปรแกรมผิดเสมอไปในทางไวยากรณ์ระดับ COBOL ทั่วไป
  (ในบางกรณี COBOL อนุญาตให้ชื่อซ้ำกันได้ถ้าอยู่คนละกลุ่มและใช้ `OF`/`IN` ระบุให้ชัดเจนว่าหมายถึงตัวไหน
  ซึ่งเป็นเทคนิคขั้นสูงที่จะกล่าวถึงเมื่อจำเป็น) แต่สำหรับผู้เริ่มต้น **ทางที่ปลอดภัยที่สุดคือตั้งชื่อ
  ตัวแปรทุกตัวให้ไม่ซ้ำกันเลยในทั้งโปรแกรม**
- เมื่อเจอ error เกี่ยวกับ level number ให้กลับไปนับดูโครงสร้างทั้งหมดตั้งแต่ 01-level ลงมาว่า
  ลำดับตื้น-ลึกสอดคล้องกันหรือไม่

### แบบฝึกหัดที่ 50.1 (แบบฝึกหัดสรุปท้าย Part)

**โจทย์**: จงออกแบบโครงสร้าง WORKING-STORAGE สำหรับเก็บข้อมูลหนังสือ 1 เล่ม ประกอบด้วย: รหัส ISBN
(13 หลัก), ชื่อหนังสือ (สูงสุด 50 ตัวอักษร), ราคา (ตัวเลข 5 หลัก ทศนิยม 2 ตำแหน่ง), และสถานะว่า
"มีสต็อก" หรือ "หมดสต็อก" (ใช้ 88-level)

**เฉลย**:

```cobol
       01  WS-BOOK-RECORD.
           05  WS-BOOK-ISBN       PIC 9(13).
           05  WS-BOOK-TITLE      PIC X(50).
           05  WS-BOOK-PRICE      PIC 9(5)V99.
           05  WS-BOOK-STOCK-FLAG PIC X.
               88  WS-IN-STOCK    VALUE "Y".
               88  WS-OUT-OF-STOCK VALUE "N".
```

โครงสร้างนี้ใช้ group item เพื่อรวมข้อมูลหนังสือทั้งหมดไว้ด้วยกัน และใช้ 88-level เพื่อให้เขียน
`IF WS-IN-STOCK` ได้อย่างอ่านง่ายใน PROCEDURE DIVISION ในอนาคต

---

## สรุปท้ายบท

ใน Part นี้ คุณได้เรียนรู้:

- ภาพรวมของ DATA DIVISION และ Section ทั้ง 4 ที่เป็นไปได้ (FILE, WORKING-STORAGE, LOCAL-STORAGE, LINKAGE)
- WORKING-STORAGE SECTION และการที่ค่าตัวแปรคงอยู่ตลอดการทำงานของโปรแกรม พิสูจน์ด้วยการทดลองจริง
- Level Numbers ทั้งหมด (01, 05-49, 66, 77, 88) และความหมายของแต่ละแบบ
- การประกาศตัวแปรอย่างง่ายด้วย 01-level และ PIC โดยตรง
- Group Items กับ Elementary Items และพลังของการ MOVE ทั้งกลุ่มในคำสั่งเดียว
- VALUE Clause และเหตุผลว่าทำไมไม่ควรพึ่งพาค่าเริ่มต้นที่ไม่ได้ระบุไว้อย่างชัดเจน
- 77-Level สำหรับตัวแปรเดี่ยว พร้อมพิสูจน์กฎด้วยตัวอย่าง error จริง
- FILLER สำหรับการเว้นพื้นที่ที่ไม่ต้องอ้างอิงชื่อ
- หลักการจัดระเบียบ WORKING-STORAGE ที่ดีตามธรรมเนียมอุตสาหกรรมจริง (Naming Convention)
- ตัวอย่างโปรแกรมรวมและ Error ที่พบบ่อยพร้อมวิธีแก้

ตอนนี้คุณมีพื้นฐานที่แข็งแรงมากในการประกาศและจัดระเบียบตัวแปรใน WORKING-STORAGE SECTION แล้ว
ใน **Part 006** เราจะเจาะลึก **PICTURE Clause** อย่างละเอียดที่สุด ทั้งชนิดข้อมูลตัวเลข ตัวอักษร
USAGE clause แบบต่าง ๆ และ Editing Character สำหรับจัดรูปแบบผลลัพธ์ให้สวยงามแบบมืออาชีพ

**[← กลับไป Part 004](part-004-identification-environment-division.md)** | **[ไปยัง Part 006: PICTURE Clause →](part-006-picture-clause.md)**
