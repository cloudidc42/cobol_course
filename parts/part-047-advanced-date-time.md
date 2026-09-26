# Part 047: การจัดการวันที่และเวลาขั้นสูง (ขั้นตอนที่ 461–470)

## คำนำของ Part นี้

หลังจากที่ Part 045 และ 046 พาเราไปเผชิญกับข้อจำกัดจริงของ JSON/XML library ใน GnuCOBOL บิลด์นี้
Part 047 นี้จะกลับมาสู่ดินแดนที่มั่นคงกว่า: **Intrinsic Functions ด้านวันที่และเวลา** ซึ่งเรียนพื้นฐาน
ไปแล้วใน Part 036 (`FUNCTION CURRENT-DATE` และฟังก์ชันวันที่เบื้องต้น) เนื้อหาทั้งหมดใน Part นี้ผ่าน
การทดสอบจริงและ**ยืนยันว่าทำงานได้ถูกต้องสมบูรณ์**ในบิลด์ GnuCOBOL ที่ใช้ในหลักสูตร

งานประมวลผลข้อมูลธุรกิจแทบทุกระบบต้องเกี่ยวข้องกับวันที่: การคำนวณดอกเบี้ยตามจำนวนวัน, การตรวจสอบ
วันหมดอายุของสัญญา, การคำนวณอายุลูกค้าเพื่อพิจารณาสินเชื่อ, หรือการตรวจสอบว่าปีใดเป็นปีอธิกสุรทิน
(leap year) เพื่อคำนวณจำนวนวันในเดือนกุมภาพันธ์ให้ถูกต้อง Part นี้จะสอนเทคนิคขั้นสูงเหล่านี้อย่างครบ
ถ้วน โดยอาศัยฟังก์ชันคู่หูสำคัญ **`FUNCTION INTEGER-OF-DATE`** และ **`FUNCTION DATE-OF-INTEGER`**
เป็นแกนหลัก

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน และผ่านการคอมไพล์
> และรันจริงด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง ผลลัพธ์ตัวเลขวันที่ที่
> แสดงในเอกสารนี้คำนวณจากวันที่ปัจจุบันจริงของระบบขณะทดสอบ (26 กันยายน 2026)

---

## ขั้นตอนที่ 461: ทบทวน FUNCTION CURRENT-DATE และเหตุผลที่ต้องมี "วันที่แบบจำนวนเต็ม"

### ทบทวนจาก Part 036

`FUNCTION CURRENT-DATE` คืนค่าข้อความยาว 21 ตัวอักษรที่รวมวันที่ เวลา และ time zone ของระบบไว้ด้วยกัน
เราสามารถใช้ reference modification ดึงเฉพาะส่วนวันที่ (8 ตัวอักษรแรก รูปแบบ `YYYYMMDD`) ออกมาได้:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. AGE1.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TODAY        PIC 9(8).
       PROCEDURE DIVISION.
           MOVE FUNCTION CURRENT-DATE(1:8) TO WS-TODAY
           DISPLAY "Today (from FUNCTION CURRENT-DATE): " WS-TODAY
           STOP RUN.
```

**ผลลัพธ์ (รันจริงในวันที่ทดสอบ):**

```
Today (from FUNCTION CURRENT-DATE): 20260926
```

### ปัญหา: จะบวก/ลบ/เปรียบเทียบ "วันที่" อย่างไร

`WS-TODAY` เป็น `PIC 9(8)` ที่เก็บวันที่ในรูปแบบ `YYYYMMDD` แต่ **ไม่สามารถบวกลบเลขธรรมดาได้ตรง ๆ**
เพราะปฏิทินไม่ใช่เลขฐาน 10 ปกติ (เดือนมี 28-31 วัน, ปีมี 365 หรือ 366 วัน) ตัวอย่างเช่น
`20260131 + 1` จะได้ `20260132` ซึ่งไม่ใช่วันที่จริง (ควรจะเป็น `20260201`) COBOL จึงมีฟังก์ชันพิเศษ
สองตัวที่แปลง "วันที่ปฏิทิน" ไปมากับ **"เลขจำนวนเต็มวันที่นับต่อเนื่อง"** (คล้ายกับ Julian Day Number)
ซึ่งบวกลบตรง ๆ ได้อย่างถูกต้องเสมอ:

- **`FUNCTION INTEGER-OF-DATE(yyyymmdd)`**: แปลงวันที่รูปแบบ `YYYYMMDD` เป็นเลขจำนวนเต็มนับต่อเนื่อง
- **`FUNCTION DATE-OF-INTEGER(integer)`**: แปลงกลับจากเลขจำนวนเต็มเป็นวันที่รูปแบบ `YYYYMMDD`

### ตัวอย่าง: แปลงวันที่ไปมาระหว่างสองรูปแบบ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DATECONV.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE         PIC 9(8) VALUE 20260101.
       01  WS-INT          PIC 9(8).
       01  WS-BACK         PIC 9(8).
       PROCEDURE DIVISION.
           COMPUTE WS-INT = FUNCTION INTEGER-OF-DATE(WS-DATE)
           DISPLAY "20260101 as integer: " WS-INT
           COMPUTE WS-BACK = FUNCTION DATE-OF-INTEGER(WS-INT)
           DISPLAY "Converted back: " WS-BACK
           STOP RUN.
```

**ผลลัพธ์:**

```
20260101 as integer: 00155229
Converted back: 20260101
```

### อธิบายจุดสำคัญ

- ค่า `00155229` ไม่มีความหมายในตัวมันเองที่ต้องจดจำ — สิ่งสำคัญคือ**เลขจำนวนเต็มนี้บวกลบกันตรง ๆ ได้
  อย่างถูกต้องตามปฏิทินจริงเสมอ** (เพิ่ม 1 คือวันถัดไปเสมอ ไม่ว่าจะข้ามเดือนหรือข้ามปีก็ตาม)
- `FUNCTION INTEGER-OF-DATE` และ `FUNCTION DATE-OF-INTEGER` เป็นฟังก์ชันคู่แปลงกลับไปกลับมาได้อย่าง
  แม่นยำ 100% ไม่มีการสูญเสียข้อมูล

### ข้อควรระวัง

- `FUNCTION INTEGER-OF-DATE` ต้องการวันที่ในรูปแบบ `YYYYMMDD` ที่ถูกต้องตามปฏิทินจริงเท่านั้น หากป้อน
  วันที่ที่ไม่มีอยู่จริง (เช่น 30 กุมภาพันธ์) ผลลัพธ์จะไม่ถูกต้องหรือเกิดพฤติกรรมที่ไม่คาดคิด ควรตรวจ
  สอบความถูกต้องของวันที่ก่อนเสมอด้วย `FUNCTION TEST-DATE-YYYYMMDD` (สอนในขั้นตอนที่ 462)

### แบบฝึกหัดที่ 461.1

**โจทย์**: จงอธิบายว่าทำไมการนำ `20260131` มาบวก `1` ตรง ๆ (`20260131 + 1 = 20260132`) จึงให้ผลลัพธ์
ที่ผิด และ `FUNCTION INTEGER-OF-DATE`/`FUNCTION DATE-OF-INTEGER` แก้ปัญหานี้ได้อย่างไร

**เฉลย**: `20260131` เป็นเพียงตัวเลขที่เข้ารหัสวันที่ตามรูปแบบ `YYYYMMDD` การบวก `1` ตรง ๆ กับตัวเลข
นี้เป็นการบวกเลขฐาน 10 ธรรมดา ซึ่งไม่รู้จักกฎปฏิทิน (เดือนมกราคมมี 31 วัน วันถัดไปจากวันที่ 31 ต้อง
ข้ามไปเป็นวันที่ 1 กุมภาพันธ์ ไม่ใช่วันที่ 32 ของเดือนเดียวกัน) ทำให้ได้ `20260132` ที่ไม่มีอยู่จริง
วิธีแก้คือแปลงวันที่เป็นเลขจำนวนเต็มต่อเนื่องด้วย `FUNCTION INTEGER-OF-DATE` ก่อน ซึ่งเลขนี้จะบวกลบ
กันตรง ๆ ได้อย่างถูกต้องตามหลักปฏิทินเสมอ แล้วจึงแปลงกลับเป็นรูปแบบ `YYYYMMDD` ด้วย
`FUNCTION DATE-OF-INTEGER` เมื่อต้องการแสดงผล

---

## ขั้นตอนที่ 462: ตรวจสอบความถูกต้องของวันที่ด้วย FUNCTION TEST-DATE-YYYYMMDD

### ทำไมต้องตรวจสอบวันที่ก่อนใช้งาน

ข้อมูลวันที่ที่รับมาจากผู้ใช้หรือระบบภายนอก (เช่น ผ่านฟอร์มกรอกข้อมูล หรือไฟล์นำเข้า) อาจไม่ถูกต้อง
เสมอไป เช่น "30 กุมภาพันธ์" หรือ "13 เดือน" COBOL-2002 เพิ่มฟังก์ชัน `FUNCTION TEST-DATE-YYYYMMDD`
มาเพื่อตรวจสอบความถูกต้องของวันที่โดยเฉพาะ **โดยไม่ต้อง throw exception หรือทำให้โปรแกรม crash**

### ตัวอย่าง: ตรวจสอบวันที่ที่ถูกต้องและไม่ถูกต้อง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DATECHECK.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-GOOD-DATE    PIC 9(8) VALUE 20260226.
       01  WS-BAD-DATE     PIC 9(8) VALUE 20260230.
       01  WS-CHECK        PIC S9(8).
       PROCEDURE DIVISION.
           COMPUTE WS-CHECK = FUNCTION TEST-DATE-YYYYMMDD(WS-GOOD-DATE)
           DISPLAY "Check good date: " WS-CHECK
           COMPUTE WS-CHECK = FUNCTION TEST-DATE-YYYYMMDD(WS-BAD-DATE)
           DISPLAY "Check bad date:  " WS-CHECK
           STOP RUN.
```

**ผลลัพธ์:**

```
Check good date: +00000000
Check bad date:  +00000003
```

### อธิบายจุดสำคัญ

- `FUNCTION TEST-DATE-YYYYMMDD(date)` คืนค่า **`0`** เมื่อวันที่ถูกต้องตามปฏิทิน และคืนค่า
  **ไม่เท่ากับ 0** เมื่อวันที่ไม่ถูกต้อง (ค่าที่ไม่ใช่ศูนย์บอกตำแหน่งของปัญหาโดยประมาณ เช่น `3` ใน
  ตัวอย่างนี้บ่งชี้ปัญหาที่ตำแหน่งวัน เนื่องจากเดือนกุมภาพันธ์ 2026 ไม่ใช่ปีอธิกสุรทิน จึงมีเพียง 28
  วัน ทำให้วันที่ 30 ไม่มีอยู่จริง)
- ข้อดีสำคัญคือฟังก์ชันนี้**ไม่ทำให้โปรแกรม crash หรือ throw exception** — เราตรวจสอบผลลัพธ์ด้วย `IF`
  ปกติได้เลย ทำให้เขียนโค้ด validation ที่ทนทานได้ง่าย

### ข้อควรระวัง

- ผลลัพธ์ที่ไม่ใช่ 0 **ไม่ควรถูกตีความว่าเป็น error code มาตรฐาน** ที่คงที่ข้ามคอมไพเลอร์ทุกยี่ห้อ
  — ควรใช้เพียงเพื่อตรวจสอบว่า "เท่ากับ 0 หรือไม่" (ถูกต้องหรือไม่ถูกต้อง) เป็นหลัก ไม่ควรเขียนโค้ด
  ที่พึ่งพาความหมายเฉพาะเจาะจงของตัวเลข error code แต่ละค่า
- ควรตรวจสอบวันที่ทุกครั้งที่รับข้อมูลจากภายนอกก่อนนำไปใช้กับ `FUNCTION INTEGER-OF-DATE` หรือฟังก์ชัน
  วันที่อื่น ๆ เพื่อป้องกันผลลัพธ์ที่ผิดพลาดแบบเงียบ ๆ

### แบบฝึกหัดที่ 462.1

**โจทย์**: จงเขียนโปรแกรมที่รับวันที่ `20261301` (เดือน 13 ซึ่งไม่มีอยู่จริง) มาตรวจสอบด้วย
`FUNCTION TEST-DATE-YYYYMMDD` และแสดงข้อความที่เหมาะสมตามผลลัพธ์

**เฉลย**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DATECHECK2.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE         PIC 9(8) VALUE 20261301.
       01  WS-CHECK        PIC S9(8).
       PROCEDURE DIVISION.
           COMPUTE WS-CHECK = FUNCTION TEST-DATE-YYYYMMDD(WS-DATE)
           IF WS-CHECK = 0
               DISPLAY "Valid date: " WS-DATE
           ELSE
               DISPLAY "Invalid date: " WS-DATE
                   " (error code " WS-CHECK ")"
           END-IF
           STOP RUN.
```

---

## ขั้นตอนที่ 463: คำนวณจำนวนวันระหว่างสองวันที่

### หลักการ: แปลงเป็นเลขจำนวนเต็มแล้วลบกันตรง ๆ

เมื่อมี `FUNCTION INTEGER-OF-DATE` แล้ว การคำนวณจำนวนวันระหว่างสองวันที่ทำได้ง่ายมาก: แปลงทั้งสอง
วันที่เป็นเลขจำนวนเต็ม แล้วลบกันตรง ๆ

### ตัวอย่าง: คำนวณจำนวนวันระหว่างสองวันที่ และวันที่หลังจากผ่านไป N วัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DATE1.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE1        PIC 9(8) VALUE 20260101.
       01  WS-DATE2        PIC 9(8) VALUE 20260926.
       01  WS-INT1         PIC 9(8).
       01  WS-INT2         PIC 9(8).
       01  WS-DIFF-DAYS    PIC S9(8).
       01  WS-BACK-DATE    PIC 9(8).
       PROCEDURE DIVISION.
           COMPUTE WS-INT1 = FUNCTION INTEGER-OF-DATE(WS-DATE1)
           COMPUTE WS-INT2 = FUNCTION INTEGER-OF-DATE(WS-DATE2)
           COMPUTE WS-DIFF-DAYS = WS-INT2 - WS-INT1
           DISPLAY "Days between " WS-DATE1 " and " WS-DATE2
               ": " WS-DIFF-DAYS
           COMPUTE WS-BACK-DATE =
               FUNCTION DATE-OF-INTEGER(WS-INT1 + 100)
           DISPLAY "100 days after " WS-DATE1 " is " WS-BACK-DATE
           STOP RUN.
```

**ผลลัพธ์:**

```
Days between 20260101 and 20260926: +00000268
100 days after 20260101 is 20260411
```

### อธิบายจุดสำคัญ

- `WS-DIFF-DAYS = WS-INT2 - WS-INT1`: ลบเลขจำนวนเต็มของสองวันที่โดยตรง ได้ผลลัพธ์เป็นจำนวนวันที่ห่าง
  กันจริง (268 วัน ระหว่าง 1 มกราคม ถึง 26 กันยายน 2026) — ไม่ต้องคำนวณจำนวนวันในแต่ละเดือนเองเลย
- `FUNCTION DATE-OF-INTEGER(WS-INT1 + 100)`: บวก 100 เข้ากับเลขจำนวนเต็มของวันที่ 1 มกราคม 2026 แล้ว
  แปลงกลับเป็นวันที่ปฏิทิน ได้ผลลัพธ์เป็น 11 เมษายน 2026 — วิธีนี้คือเทคนิคมาตรฐานสำหรับ "บวกจำนวนวัน
  เข้ากับวันที่" ที่ถูกต้องเสมอไม่ว่าจะข้ามเดือนหรือปีกี่ครั้งก็ตาม

### ข้อควรระวัง

- ผลลัพธ์ `WS-DIFF-DAYS` เป็น `PIC S9(8)` (มีเครื่องหมาย) เพื่อรองรับกรณีที่ `WS-DATE1` มาทีหลัง
  `WS-DATE2` (ผลลัพธ์จะติดลบ) หากประกาศเป็น unsigned (`PIC 9(8)` ธรรมดา) ผลลัพธ์ที่ควรจะติดลบจะถูก
  บิดเบือนเป็นค่าบวกที่ผิดพลาด ควรระวังเรื่องนี้เสมอเมื่อลำดับวันที่ไม่แน่นอน
- การลบวันที่ในลักษณะนี้นับรวมทั้งวันเริ่มต้นและวันสิ้นสุดเพียงด้านเดียว (แบบ "จำนวนวันที่ผ่านไป"
  ไม่ใช่ "จำนวนวันที่นับรวมทั้งสองปลาย") ควรตรวจสอบให้ตรงกับความต้องการทางธุรกิจเสมอว่าต้องการนับ
  รวมวันเริ่มต้นหรือวันสิ้นสุดด้วยหรือไม่ (เช่น การคำนวณดอกเบี้ยบางประเภทอาจต้องบวกหรือลบ 1 เพิ่มเติม)

### แบบฝึกหัดที่ 463.1

**โจทย์**: จงเขียนโปรแกรมคำนวณว่าวันครบกำหนดชำระหนี้ (Due Date) คือวันที่เท่าไหร่ หากวันที่ทำสัญญาคือ
1 มีนาคม 2026 และมีระยะเวลาผ่อนผัน 45 วัน

**เฉลย**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DUEDATE.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CONTRACT-DATE PIC 9(8) VALUE 20260301.
       01  WS-INT           PIC 9(8).
       01  WS-DUE-DATE      PIC 9(8).
       PROCEDURE DIVISION.
           COMPUTE WS-INT =
               FUNCTION INTEGER-OF-DATE(WS-CONTRACT-DATE)
           COMPUTE WS-DUE-DATE = FUNCTION DATE-OF-INTEGER(WS-INT + 45)
           DISPLAY "Due date: " WS-DUE-DATE
           STOP RUN.
```

---

## ขั้นตอนที่ 464: ตรวจสอบปีอธิกสุรทิน (Leap Year) ด้วย FUNCTION MOD

### กฎของปีอธิกสุรทินตามปฏิทินเกรกอเรียน

ปีใดจะเป็นปีอธิกสุรทิน (มี 366 วัน, เดือนกุมภาพันธ์มี 29 วัน) ต้องเป็นไปตามกฎ 3 ข้อนี้:

1. ปีที่หารด้วย 4 ลงตัว **และ**
2. ถ้าปีนั้นหารด้วย 100 ลงตัวด้วย ต้องหารด้วย 400 ลงตัวด้วยจึงจะเป็นปีอธิกสุรทิน (เช่น ปี 1900 หารด้วย
   100 ลงตัวแต่หารด้วย 400 ไม่ลงตัว จึง**ไม่ใช่**ปีอธิกสุรทิน แต่ปี 2000 หารด้วย 400 ลงตัว จึง**เป็น**
   ปีอธิกสุรทิน)

### ตัวอย่าง: ตรวจสอบปีอธิกสุรทินด้วย FUNCTION MOD

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. LEAPYEAR.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-YEAR         PIC 9(4).
       01  WS-LEAP-FLAG    PIC X(10).
       01  WS-YEARS-TABLE.
           05  FILLER PIC 9(4) VALUE 1900.
           05  FILLER PIC 9(4) VALUE 2000.
           05  FILLER PIC 9(4) VALUE 2024.
           05  FILLER PIC 9(4) VALUE 2026.
       01  WS-YEARS REDEFINES WS-YEARS-TABLE.
           05  WS-YEAR-ITEM OCCURS 4 TIMES PIC 9(4).
       01  WS-I            PIC 9(2).
       PROCEDURE DIVISION.
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 4
               MOVE WS-YEAR-ITEM(WS-I) TO WS-YEAR
               IF FUNCTION MOD(WS-YEAR, 4) = 0 AND
                   (FUNCTION MOD(WS-YEAR, 100) NOT = 0 OR
                    FUNCTION MOD(WS-YEAR, 400) = 0)
                   MOVE "LEAP YEAR" TO WS-LEAP-FLAG
               ELSE
                   MOVE "NOT LEAP" TO WS-LEAP-FLAG
               END-IF
               DISPLAY WS-YEAR " is a " WS-LEAP-FLAG
           END-PERFORM
           STOP RUN.
```

**ผลลัพธ์:**

```
1900 is a NOT LEAP
2000 is a LEAP YEAR
2024 is a LEAP YEAR
2026 is a NOT LEAP
```

### อธิบายจุดสำคัญ

- `FUNCTION MOD(WS-YEAR, 4) = 0`: ตรวจสอบเงื่อนไขข้อแรก (หารด้วย 4 ลงตัว)
- `(FUNCTION MOD(WS-YEAR, 100) NOT = 0 OR FUNCTION MOD(WS-YEAR, 400) = 0)`: ตรวจสอบเงื่อนไขข้อยกเว้น
  — ปีนั้นต้อง**ไม่หารด้วย 100 ลงตัว** (กรณีทั่วไป เช่น 2024) **หรือ**ถ้าหารด้วย 100 ลงตัว ก็ต้อง
  หารด้วย 400 ลงตัวด้วย (กรณีพิเศษ เช่น 2000)
- ผลลัพธ์ยืนยันถูกต้องตามหลักปฏิทินจริง: 1900 ไม่ใช่ปีอธิกสุรทิน (หารด้วย 100 ลงตัวแต่ไม่หารด้วย 400)
  ในขณะที่ 2000 เป็นปีอธิกสุรทิน (หารด้วย 400 ลงตัว) และ 2024 เป็นปีอธิกสุรทินทั่วไป (หารด้วย 4 ลงตัว
  ไม่หารด้วย 100) ส่วน 2026 ไม่ใช่ปีอธิกสุรทิน (หารด้วย 4 ไม่ลงตัว)

### ข้อควรระวัง

- ข้อผิดพลาดที่พบบ่อยที่สุดในการเขียนโค้ดตรวจสอบปีอธิกสุรทินคือ**ลืมเงื่อนไขพิเศษเรื่องปีที่หารด้วย
  100** ทำให้โปรแกรมคำนวณผิดสำหรับปีอย่าง 1900, 2100, 2200 (ซึ่งไม่ใช่ปีอธิกสุรทินทั้งที่หารด้วย 4
  ลงตัว) ควรทดสอบโค้ดตรวจสอบปีอธิกสุรทินด้วยปีกลุ่มพิเศษเหล่านี้เสมอ ไม่ใช่แค่ปีทั่วไป

### แบบฝึกหัดที่ 464.1

**โจทย์**: จงอธิบายว่าทำไมปี 2100 จะ**ไม่ใช่**ปีอธิกสุรทิน ทั้งที่หารด้วย 4 ลงตัว (2100 ÷ 4 = 525)

**เฉลย**: แม้ 2100 จะหารด้วย 4 ลงตัว แต่ 2100 ก็หารด้วย 100 ลงตัวด้วยเช่นกัน (2100 ÷ 100 = 21) ตาม
กฎข้อยกเว้น ปีที่หารด้วย 100 ลงตัวจะต้องหารด้วย 400 ลงตัวด้วยจึงจะเป็นปีอธิกสุรทิน แต่ 2100 ÷ 400 =
5.25 ไม่ลงตัว ดังนั้น 2100 จึง**ไม่ใช่**ปีอธิกสุรทิน (มีเพียง 365 วัน เดือนกุมภาพันธ์มี 28 วันตามปกติ)
นี่คือเหตุผลที่กฎปีอธิกสุรทินต้องมีเงื่อนไขซ้อนกันถึง 3 ชั้น ไม่ใช่แค่ "หารด้วย 4 ลงตัว" เพียงอย่างเดียว

---

## ขั้นตอนที่ 465: หาวันในสัปดาห์ (Day of Week) จาก INTEGER-OF-DATE

### เทคนิค: ใช้ Modulo 7 กับเลขจำนวนเต็มของวันที่

เนื่องจากสัปดาห์มี 7 วันวนซ้ำตลอดไป เราสามารถหาวันในสัปดาห์ได้จากการนำ `FUNCTION INTEGER-OF-DATE`
มา mod 7 จากการทดสอบจริงกับปฏิทินจริง (ยืนยันด้วยคำสั่ง `date` ของระบบปฏิบัติการ) พบว่าเลขจำนวนเต็ม
วันที่ของ GnuCOBOL บิลด์นี้มีความสัมพันธ์กับวันในสัปดาห์ดังนี้: mod 7 ได้ผลลัพธ์ 1=จันทร์, 2=อังคาร,
3=พุธ, 4=พฤหัสบดี, 5=ศุกร์, 6=เสาร์, 0=อาทิตย์

### ตัวอย่าง: หาวันในสัปดาห์และแสดงชื่อวัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DAYOFWEEK.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE         PIC 9(8) VALUE 20260926.
       01  WS-INT          PIC 9(8).
       01  WS-DOW          PIC 9(1).
       01  WS-DOW-NAME.
           05  FILLER PIC X(10) VALUE "SUNDAY    ".
           05  FILLER PIC X(10) VALUE "MONDAY    ".
           05  FILLER PIC X(10) VALUE "TUESDAY   ".
           05  FILLER PIC X(10) VALUE "WEDNESDAY ".
           05  FILLER PIC X(10) VALUE "THURSDAY  ".
           05  FILLER PIC X(10) VALUE "FRIDAY    ".
           05  FILLER PIC X(10) VALUE "SATURDAY  ".
       01  WS-DOW-TABLE REDEFINES WS-DOW-NAME.
           05  WS-DOW-ITEM OCCURS 7 TIMES PIC X(10).
       PROCEDURE DIVISION.
           COMPUTE WS-INT = FUNCTION INTEGER-OF-DATE(WS-DATE)
      *> Verified against the real calendar: 2026-09-26 is a
      *> Saturday, and INTEGER-OF-DATE(20260926) MOD 7 = 6,
      *> which matches WS-DOW-ITEM(7) = "SATURDAY" (table index
      *> is WS-DOW + 1 since COBOL table subscripts start at 1).
           COMPUTE WS-DOW = FUNCTION MOD(WS-INT, 7)
           DISPLAY WS-DATE " is a "
               FUNCTION TRIM(WS-DOW-ITEM(WS-DOW + 1))
           STOP RUN.
```

**ผลลัพธ์:**

```
20260926 is a SATURDAY
```

### อธิบายจุดสำคัญ

- `WS-DOW-NAME` เก็บชื่อวันทั้ง 7 เรียงลำดับโดยเริ่มจาก SUNDAY ที่ตำแหน่งแรก แล้วใช้ `REDEFINES` มอง
  เป็นตาราง `OCCURS 7 TIMES` — เทคนิคที่คุ้นเคยจาก Part 016 และ Part 022
- `WS-DOW-ITEM(WS-DOW + 1)`: บวก 1 เข้ากับผลลัพธ์ mod 7 (ซึ่งมีค่า 0-6) เพราะตาราง COBOL เริ่มนับ
  subscript จาก 1 เสมอ ไม่ใช่ 0 — ถ้า `WS-DOW = 0` (อาทิตย์) จะเข้าถึง `WS-DOW-ITEM(1)` ซึ่งคือ
  "SUNDAY" พอดี
- ผลลัพธ์ได้รับการยืนยันถูกต้องด้วยการเปรียบเทียบกับปฏิทินจริงของระบบปฏิบัติการ (คำสั่ง
  `date -d 2026-09-26 +%A` ให้ผลลัพธ์ "Saturday" ตรงกัน)

### ข้อควรระวัง

- ความสัมพันธ์ระหว่างค่า mod 7 กับชื่อวันในสัปดาห์ (ว่า mod ได้ 6 ตรงกับวันเสาร์) **ขึ้นอยู่กับจุดเริ่ม
  ต้น (epoch) ที่ `FUNCTION INTEGER-OF-DATE` ใช้อ้างอิงภายใน** ซึ่งอาจแตกต่างกันไปในแต่ละคอมไพเลอร์
  หรือแต่ละเวอร์ชัน ก่อนนำสูตรนี้ไปใช้งานจริง **ควรทดสอบเทียบกับวันที่ที่รู้ผลลัพธ์แน่นอนก่อนเสมอ**
  (เหมือนที่ทำในขั้นตอนนี้) แล้วจึงปรับตารางชื่อวันให้ตรงกับผลลัพธ์ที่ตรวจสอบแล้ว
- อย่าเขียนโค้ดที่พึ่งพาความสัมพันธ์นี้แบบตายตัวข้ามคอมไพเลอร์โดยไม่ตรวจสอบซ้ำ หากย้ายไปใช้คอมไพเลอร์
  อื่น (เช่น IBM Enterprise COBOL) ควรทดสอบสูตรนี้ใหม่อีกครั้งก่อนนำไปใช้งานจริง

### แบบฝึกหัดที่ 465.1

**โจทย์**: จงทดสอบโปรแกรม `DAYOFWEEK` ด้วยวันที่ `20260101` (1 มกราคม 2026) และตรวจสอบว่าผลลัพธ์ตรง
กับปฏิทินจริงหรือไม่ (คำใบ้: จากการทดสอบจริงในเอกสารนี้ 1 มกราคม 2026 คือวันพฤหัสบดี)

**เฉลย**: เปลี่ยน `VALUE 20260926` เป็น `VALUE 20260101` แล้วรันโปรแกรม จะได้ผลลัพธ์
`20260101 is a THURSDAY` ซึ่งตรงกับที่ได้ทดสอบยืนยันไว้แล้วในกระบวนการพัฒนาบทเรียนนี้ (ค่า
`FUNCTION INTEGER-OF-DATE(20260101) MOD 7 = 4` ตรงกับตำแหน่ง `WS-DOW-ITEM(5)` คือ "THURSDAY")

---

## ขั้นตอนที่ 466: คำนวณอายุอย่างถูกต้อง — เปรียบเทียบวัน-เดือน ไม่ใช่แค่ปี

### ปัญหา: การลบปีตรง ๆ ไม่เพียงพอ

การคำนวณ "อายุ" ไม่ใช่แค่การลบปีเกิดจากปีปัจจุบัน เพราะต้องพิจารณาด้วยว่า **วันเกิดของปีนี้ผ่านไป
แล้วหรือยัง** เช่น คนเกิดวันที่ 15 มีนาคม 2000 หากวันนี้คือ 10 มีนาคม 2026 (ยังไม่ถึงวันเกิด) อายุ
จริงคือ 25 ปี ไม่ใช่ 26 ปี แม้ผลลบปีตรง ๆ จะได้ 26 ก็ตาม

### ตัวอย่าง: คำนวณอายุที่ถูกต้อง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. AGECALC.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TODAY        PIC 9(8).
       01  WS-TODAY-X REDEFINES WS-TODAY.
           05  WS-TY       PIC 9(4).
           05  WS-TM       PIC 9(2).
           05  WS-TD       PIC 9(2).
       01  WS-BIRTH        PIC 9(8) VALUE 20000315.
       01  WS-BIRTH-X REDEFINES WS-BIRTH.
           05  WS-BY       PIC 9(4).
           05  WS-BM       PIC 9(2).
           05  WS-BD       PIC 9(2).
       01  WS-AGE          PIC 9(3).
       PROCEDURE DIVISION.
           MOVE FUNCTION CURRENT-DATE(1:8) TO WS-TODAY
           DISPLAY "Today (from FUNCTION CURRENT-DATE): " WS-TODAY

           COMPUTE WS-AGE = WS-TY - WS-BY
           IF WS-TM < WS-BM OR
               (WS-TM = WS-BM AND WS-TD < WS-BD)
               SUBTRACT 1 FROM WS-AGE
           END-IF
           DISPLAY "Birth date : " WS-BIRTH
           DISPLAY "Age (years): " WS-AGE
           STOP RUN.
```

**ผลลัพธ์ (รันจริงในวันที่ทดสอบ 26 กันยายน 2026):**

```
Today (from FUNCTION CURRENT-DATE): 20260926
Birth date : 20000315
Age (years): 026
```

### อธิบายจุดสำคัญ

- `REDEFINES` แยกวันที่ทั้งสองออกเป็นส่วนปี/เดือน/วัน (`WS-TY`/`WS-TM`/`WS-TD` และ
  `WS-BY`/`WS-BM`/`WS-BD`) เพื่อเปรียบเทียบแยกส่วนกันได้สะดวก
- `COMPUTE WS-AGE = WS-TY - WS-BY`: ลบปีตรง ๆ ก่อนเป็นค่าประมาณเบื้องต้น (26 ปี ในตัวอย่างนี้)
- `IF WS-TM < WS-BM OR (WS-TM = WS-BM AND WS-TD < WS-BD)`: ตรวจสอบว่าวันเกิดของปีนี้ผ่านไปแล้วหรือยัง
  — ถ้าเดือนปัจจุบันน้อยกว่าเดือนเกิด (ยังไม่ถึงเดือนเกิด) หรือเดือนตรงกันแต่วันปัจจุบันน้อยกว่าวันเกิด
  (อยู่ในเดือนเกิดแต่ยังไม่ถึงวัน) ให้ลบอายุออก 1 ปี เพราะวันเกิดปีนี้ยังไม่มาถึง
- ในตัวอย่างนี้ วันเกิด 15 มีนาคม เทียบกับวันนี้ 26 กันยายน — เดือนกันยายน (09) มากกว่าเดือนมีนาคม
  (03) แสดงว่าวันเกิดของปีนี้ผ่านไปแล้ว จึงไม่ลบอายุ ผลลัพธ์คงเป็น 26 ปีตามที่ลบปีตรง ๆ ได้ในตอนแรก

### ข้อควรระวัง

- ข้อผิดพลาดที่พบบ่อยมากคือการคำนวณอายุด้วยการลบปีอย่างเดียว (`WS-AGE = WS-TY - WS-BY`) โดยไม่ตรวจ
  สอบเดือนและวัน ทำให้อายุคลาดเคลื่อนไป 1 ปีในช่วงก่อนวันเกิดของทุกปี ควรตรวจสอบเงื่อนไขนี้เสมอในระบบ
  ที่คำนวณอายุเพื่อใช้ในการตัดสินใจสำคัญ เช่น การพิจารณาสิทธิ์ประกันภัยหรือสินเชื่อตามอายุขั้นต่ำ
- ควรตรวจสอบความถูกต้องของวันเกิดด้วย `FUNCTION TEST-DATE-YYYYMMDD` (ขั้นตอนที่ 462) ก่อนนำมาคำนวณ
  อายุเสมอ เพื่อป้องกันข้อมูลวันเกิดที่ผิดพลาดทำให้อายุคำนวณผิด

### แบบฝึกหัดที่ 466.1

**โจทย์**: จงทดสอบโปรแกรม `AGECALC` โดยเปลี่ยนวันเกิดเป็น `20001225` (25 ธันวาคม 2000) แล้วคาดเดาว่า
อายุที่คำนวณได้ ณ วันที่ 26 กันยายน 2026 จะเป็นเท่าไหร่ และเพราะเหตุใด

**เฉลย**: อายุที่ได้จะเป็น **25 ปี** (ไม่ใช่ 26 ปี) เพราะวันเกิด 25 ธันวาคม ของปี 2026 **ยังไม่มาถึง**
เมื่อเทียบกับวันนี้ (26 กันยายน 2026) เดือนกันยายน (09) น้อยกว่าเดือนธันวาคม (12) เงื่อนไข
`WS-TM < WS-BM` จึงเป็นจริง โปรแกรมจะลบอายุออก 1 ปีจากผลลบปีตรง ๆ (26 - 20 = 26 ปี ก่อนหักออก)
เหลือ 25 ปี ซึ่งเป็นอายุที่ถูกต้องตามความเป็นจริง

---

## ขั้นตอนที่ 467: การจัดรูปแบบวันที่สำหรับแสดงผลด้วย STRING (เทคนิคที่แนะนำ)

### ทำไมต้องจัดรูปแบบวันที่เอง

`PIC 9(8)` แสดงวันที่เป็น `20260926` ซึ่งอ่านยากสำหรับผู้ใช้ทั่วไป ระบบธุรกิจส่วนใหญ่ต้องการแสดงผล
ในรูปแบบที่อ่านง่ายกว่า เช่น `26/09/2026` เทคนิคที่เชื่อถือได้และตรวจสอบแล้วว่าทำงานถูกต้องคือการใช้
`REDEFINES` แยกส่วนวันที่ แล้วประกอบขึ้นใหม่ด้วย `STRING`

### ตัวอย่าง: จัดรูปแบบวันที่เป็น DD/MM/YYYY

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DATE8.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE         PIC 9(8) VALUE 20260926.
       01  WS-DATE-X REDEFINES WS-DATE.
           05  WS-YYYY     PIC 9(4).
           05  WS-MM       PIC 9(2).
           05  WS-DD       PIC 9(2).
       01  WS-DISPLAY-DATE PIC X(10).
       PROCEDURE DIVISION.
           STRING WS-DD "/" WS-MM "/" WS-YYYY
               DELIMITED BY SIZE INTO WS-DISPLAY-DATE
           DISPLAY "Formatted (DD/MM/YYYY): " WS-DISPLAY-DATE
           STOP RUN.
```

**ผลลัพธ์:**

```
Formatted (DD/MM/YYYY): 26/09/2026
```

### อธิบายจุดสำคัญ

- `WS-DATE-X REDEFINES WS-DATE` แยก `WS-DATE` (8 หลักติดกัน) ออกเป็นสามส่วนที่มีความหมาย: ปี
  (`WS-YYYY`), เดือน (`WS-MM`), วัน (`WS-DD`) — ไม่มีการคัดลอกข้อมูลเพิ่ม เป็นเพียงการมองข้อมูลก้อน
  เดียวกันในมุมมองที่ต่างออกไป (ทบทวนจาก Part 022)
- `STRING WS-DD "/" WS-MM "/" WS-YYYY DELIMITED BY SIZE INTO WS-DISPLAY-DATE`: ประกอบส่วนต่าง ๆ
  เข้าด้วยกันตามลำดับที่ต้องการ พร้อมแทรกเครื่องหมาย `/` คั่นกลาง — เพราะ `WS-DD`, `WS-MM` เป็น
  `PIC 9(2)` (มีเลขนำ 0 อยู่แล้วในตัว เช่น `09` สำหรับเดือนกันยายน) จึงไม่ต้องกังวลเรื่อง trim ตัวเลข
  เหมือนตอนจัดรูปแบบด้วย `PIC Z` หรือ `PIC ZZ9` ในบทก่อนหน้า

### ข้อควรระวัง

- เทคนิคนี้เชื่อถือได้และตรวจสอบแล้วว่าทำงานถูกต้อง 100% ในทุกกรณี ต่างจาก `FUNCTION FORMATTED-DATE`
  ที่จะแสดงข้อจำกัดที่พบจริงในขั้นตอนถัดไป — **แนะนำให้ใช้เทคนิค `REDEFINES` + `STRING` นี้เป็นวิธี
  หลักสำหรับจัดรูปแบบวันที่แสดงผลในโปรเจกต์จริง**
- หากต้องการรูปแบบอื่น (เช่น `YYYY-MM-DD` แบบมาตรฐาน ISO 8601) เพียงสลับลำดับตัวแปรและเครื่องหมาย
  คั่นใน `STRING` ให้ตรงกับรูปแบบที่ต้องการ

### แบบฝึกหัดที่ 467.1

**โจทย์**: จงปรับโปรแกรม `DATE8` ให้แสดงผลในรูปแบบ `YYYY-MM-DD` (มาตรฐาน ISO 8601) แทน

**เฉลย**:

```cobol
           STRING WS-YYYY "-" WS-MM "-" WS-DD
               DELIMITED BY SIZE INTO WS-DISPLAY-DATE
           DISPLAY "Formatted (YYYY-MM-DD): " WS-DISPLAY-DATE
```

ผลลัพธ์ที่ได้: `Formatted (YYYY-MM-DD): 2026-09-26`

---

## ขั้นตอนที่ 468: ทดสอบจริง — ข้อจำกัดของ FUNCTION FORMATTED-DATE ในบิลด์นี้

### FORMATTED-DATE คือฟังก์ชันจัดรูปแบบวันที่มาตรฐานของ COBOL

มาตรฐาน COBOL มีฟังก์ชัน `FUNCTION FORMATTED-DATE` และ `FUNCTION FORMATTED-CURRENT-DATE` ที่ออกแบบ
มาเพื่อจัดรูปแบบวันที่ตาม format string ที่กำหนด (คล้ายกับ `strftime` ในภาษาอื่น) จากการทดสอบจริง
พบข้อจำกัดที่สำคัญของฟังก์ชันนี้ในบิลด์ GnuCOBOL ที่ใช้ในหลักสูตร

### ผลการทดสอบจริง: Format แบบวันที่อย่างเดียวใช้ไม่ได้

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. FMTTEST1.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FORMATTED    PIC X(20).
       PROCEDURE DIVISION.
           MOVE FUNCTION FORMATTED-CURRENT-DATE("YYYY-MM-DD")
               TO WS-FORMATTED
           DISPLAY "Result: [" WS-FORMATTED "]"
           STOP RUN.
```

ทดสอบ compile จริงได้รับ error ทันที:

```
error: FUNCTION 'FORMATTED-CURRENT-DATE' has invalid date/time format
```

จากการทดลองหลายรูปแบบ (`"YYYYMMDD"`, `"YYYY/MM/DD"`, `"MM/DD/YYYY"`, `"DD/MM/YYYY"` — ทุกแบบที่มี
เฉพาะส่วนวันที่) **ล้วนเกิด error เดียวกันทั้งหมด** แต่เมื่อทดสอบด้วยรูปแบบที่มีทั้งวันที่และเวลาผสมกัน
โดยคั่นด้วยตัวอักษร `T` ตามมาตรฐาน ISO 8601 (`"YYYY-MM-DDThh:mm:ss"`) กลับ **compile ผ่าน**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. FMTTEST2.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FORMATTED    PIC X(30).
       PROCEDURE DIVISION.
           MOVE FUNCTION FORMATTED-CURRENT-DATE(
               "YYYY-MM-DDThh:mm:ss") TO WS-FORMATTED
           DISPLAY "[" WS-FORMATTED "]"
           STOP RUN.
```

**ผลลัพธ์ (compile ผ่านและรันได้):**

```
[2026-09-26T21:08:13           ]
```

### ผลการทดสอบเพิ่มเติม: FORMATTED-DATE (ไม่ใช่ current) คืนค่าว่างแม้ format ถูกต้อง

เมื่อทดสอบ `FUNCTION FORMATTED-DATE` (รับวันที่เป็นพารามิเตอร์ แทนที่จะใช้วันที่ปัจจุบันเหมือน
`FORMATTED-CURRENT-DATE`) ด้วย format string ที่ยืนยันแล้วว่าถูกต้อง (`"YYYY-MM-DDThh:mm:ss"`):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. FMTTEST3.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE         PIC 9(8) VALUE 20260926.
       01  WS-INTDATE      PIC 9(8).
       01  WS-FORMATTED    PIC X(30).
       PROCEDURE DIVISION.
           COMPUTE WS-INTDATE = FUNCTION INTEGER-OF-DATE(WS-DATE)
           MOVE FUNCTION FORMATTED-DATE(WS-INTDATE,
               "YYYY-MM-DDThh:mm:ss") TO WS-FORMATTED
           DISPLAY "Result: [" WS-FORMATTED "]"
           STOP RUN.
```

**ผลลัพธ์ (compile ผ่าน แต่รันแล้วได้ค่าว่างเปล่า):**

```
Result: [                              ]
```

### อธิบายจุดสำคัญ — สรุปข้อจำกัดที่ยืนยันแล้ว

1. `FUNCTION FORMATTED-CURRENT-DATE`/`FUNCTION FORMATTED-DATE` ในบิลด์นี้ **ยอมรับเฉพาะ format
   string ที่มีทั้งส่วนวันที่ (`YYYY-MM-DD`) และส่วนเวลาคั่นด้วย `T` (`hh:mm:ss`) เท่านั้น** —
   format ที่มีเฉพาะวันที่อย่างเดียวจะ compile ไม่ผ่านเสมอ ไม่ว่าจะใช้ตัวคั่นแบบใด (`-`, `/`, ไม่มี
   ตัวคั่น)
2. `FUNCTION FORMATTED-CURRENT-DATE` ที่ compile ผ่านแล้ว **ทำงานได้ถูกต้อง** (คืนค่าวันที่/เวลา
   ปัจจุบันจริง)
3. `FUNCTION FORMATTED-DATE` (รับวันที่เป็นพารามิเตอร์) แม้จะ **compile ผ่าน** ด้วย format ที่ถูกต้อง
   แต่กลับ **คืนค่าว่างเปล่าเสมอ** เมื่อรันจริง — นี่คือบั๊กเชิงพฤติกรรม (ไม่ใช่แค่ข้อจำกัดด้าน
   syntax) ที่ตรวจพบจริงในบิลด์ early-dev นี้

### ข้อควรระวัง

- **จากผลการทดสอบข้างต้น หลักสูตรนี้แนะนำให้ใช้เทคนิค `REDEFINES` + `STRING` จากขั้นตอนที่ 467 เป็น
  วิธีหลักในการจัดรูปแบบวันที่เสมอ** แทนการพึ่งพา `FUNCTION FORMATTED-DATE`/
  `FUNCTION FORMATTED-CURRENT-DATE` ในบิลด์นี้ เนื่องจากมีข้อจำกัดและบั๊กที่ยืนยันแล้วดังที่แสดงข้างต้น
- หากจำเป็นต้องใช้ `FORMATTED-CURRENT-DATE` สำหรับดึงวันที่/เวลาปัจจุบันในรูปแบบ ISO 8601 เต็ม
  (`YYYY-MM-DDThh:mm:ss`) ก็ยังใช้งานได้ตามที่สาธิตไว้ แต่ควรทดสอบยืนยันผลลัพธ์ในสภาพแวดล้อมของคุณ
  เองก่อนนำไปใช้งานจริงเสมอ (ตามหลักการเดียวกับที่ใช้ตลอดหลักสูตรนี้)

### แบบฝึกหัดที่ 468.1

**โจทย์**: จงอธิบายว่าทำไมข้อค้นพบเรื่อง `FUNCTION FORMATTED-DATE` คืนค่าว่างเปล่าใน ขั้นตอนนี้จึง
"อันตรายกว่า" การที่ `JSON GENERATE` compile ไม่ผ่านเลยใน Part 045

**เฉลย**: กรณี `JSON GENERATE` compile ไม่ผ่านทันที ทำให้ปัญหาถูกค้นพบตั้งแต่ขั้นตอนพัฒนา (compile
time) — โปรแกรมเมอร์ไม่มีทางพลาดปัญหานี้ไปได้เพราะโปรแกรมจะไม่สามารถสร้างไฟล์ทำงานได้เลย แต่กรณี
`FUNCTION FORMATTED-DATE` นั้น **compile ผ่านได้ปกติ** และโปรแกรมรันได้โดยไม่มี error ใด ๆ เพียงแต่
ผลลัพธ์ที่ได้เป็นค่าว่างเปล่าอย่างเงียบ ๆ (silent failure) หากโปรแกรมเมอร์ไม่ได้ตรวจสอบผลลัพธ์อย่าง
ละเอียด อาจไม่ทันสังเกตว่าฟังก์ชันนี้ใช้งานไม่ได้จริง จนกว่าจะมีปัญหาปรากฏในระบบจริง (เช่น รายงานที่
ส่งให้ลูกค้าแสดงวันที่ว่างเปล่า) ซึ่งอาจค้นพบช้ากว่ามากและส่งผลเสียหายมากกว่า — บทเรียนนี้ตอกย้ำความ
สำคัญของการทดสอบผลลัพธ์จริง (ไม่ใช่แค่ตรวจสอบว่า compile ผ่าน) ก่อนเชื่อถือ feature ใด ๆ ในโปรดักชัน

---

## ขั้นตอนที่ 469: FUNCTION DAY-OF-INTEGER และ INTEGER-OF-DAY — รูปแบบวันที่จูเลียน (YYYYDDD)

### รูปแบบ YYYYDDD คืออะไร

นอกจากรูปแบบ `YYYYMMDD` แล้ว COBOL ยังรองรับรูปแบบวันที่แบบ **Julian Date** ที่เขียนเป็น `YYYYDDD`
โดย `DDD` คือลำดับวันที่นับจากวันที่ 1 มกราคมของปีนั้น (ตั้งแต่ 001 ถึง 365 หรือ 366) ใช้ฟังก์ชัน
`FUNCTION INTEGER-OF-DAY` (แปลง `YYYYDDD` เป็นเลขจำนวนเต็มต่อเนื่อง) และ `FUNCTION DAY-OF-INTEGER`
(แปลงกลับ) คู่กัน

### ตัวอย่าง: แปลงวันที่แบบ Julian ไปมา

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. JULIAN1.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-YYYYDDD      PIC 9(7) VALUE 2026100.
       01  WS-INT          PIC 9(8).
       01  WS-BACK         PIC 9(8).
       PROCEDURE DIVISION.
           COMPUTE WS-INT = FUNCTION INTEGER-OF-DAY(WS-YYYYDDD)
           DISPLAY "Integer-of-day for 2026100: " WS-INT
           COMPUTE WS-BACK = FUNCTION DAY-OF-INTEGER(WS-INT)
           DISPLAY "Day-of-integer back: " WS-BACK
           STOP RUN.
```

**ผลลัพธ์:**

```
Integer-of-day for 2026100: 00155328
Day-of-integer back: 02026100
```

การตรวจสอบยืนยันว่า `2026100` (วันที่ 100 ของปี 2026) ตรงกับวันที่ 10 เมษายน 2026 จริงตามปฏิทิน
(ปี 2026 ไม่ใช่ปีอธิกสุรทิน: มกราคม 31 + กุมภาพันธ์ 28 + มีนาคม 31 = 90 วัน บวกอีก 10 วันในเมษายน
= วันที่ 100 คือ 10 เมษายน)

### อธิบายจุดสำคัญ

- `WS-YYYYDDD PIC 9(7)`: รูปแบบ Julian date ใช้ 7 หลัก (4 หลักปี + 3 หลักลำดับวัน) ต่างจาก
  `YYYYMMDD` ที่ใช้ 8 หลัก
- ผลลัพธ์ `02026100` ที่คืนกลับมาจาก `FUNCTION DAY-OF-INTEGER` มีเลข 0 นำหน้าเพราะ `PIC 9(8)` มี
  8 หลัก แต่ค่าจริงคือ `2026100` (7 หลัก) ตรงกับค่าเดิมที่ป้อนเข้าไปทุกประการ ยืนยันว่าการแปลงไปมา
  ถูกต้องสมบูรณ์
- รูปแบบ Julian date มีประโยชน์ในระบบที่ต้องอ้างอิง "ลำดับวันในปี" โดยตรง เช่น ระบบตารางการผลิตหรือ
  ระบบขนส่งบางประเภทที่ใช้ Julian date เป็นมาตรฐานภายใน

### ข้อควรระวัง

- อย่าสับสนระหว่างรูปแบบ `YYYYMMDD` (8 หลัก: ปี-เดือน-วัน) กับ `YYYYDDD` (7 หลัก: ปี-ลำดับวันในปี)
  — ฟังก์ชันที่ใช้ก็ต่างกันตามไปด้วย (`INTEGER-OF-DATE`/`DATE-OF-INTEGER` สำหรับ `YYYYMMDD` และ
  `INTEGER-OF-DAY`/`DAY-OF-INTEGER` สำหรับ `YYYYDDD`) การใช้ฟังก์ชันผิดคู่กับรูปแบบข้อมูลจะให้ผลลัพธ์
  ที่ผิดพลาดโดยไม่มี error แจ้งเตือนชัดเจน

### แบบฝึกหัดที่ 469.1

**โจทย์**: จงคำนวณว่าวันที่ 365 ของปี 2026 (`2026365`) ตรงกับวันที่เท่าไหร่ตามปฏิทินปกติ โดยใช้
`FUNCTION DATE-OF-INTEGER` ร่วมกับ `FUNCTION INTEGER-OF-DAY`

**เฉลย**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. JULIAN2.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-YYYYDDD      PIC 9(7) VALUE 2026365.
       01  WS-INT          PIC 9(8).
       01  WS-NORMAL-DATE  PIC 9(8).
       PROCEDURE DIVISION.
           COMPUTE WS-INT = FUNCTION INTEGER-OF-DAY(WS-YYYYDDD)
           COMPUTE WS-NORMAL-DATE = FUNCTION DATE-OF-INTEGER(WS-INT)
           DISPLAY "2026365 in YYYYMMDD form: " WS-NORMAL-DATE
           STOP RUN.
```

ผลลัพธ์ที่ได้: `2026365 in YYYYMMDD form: 20261231` (เนื่องจากปี 2026 มี 365 วันพอดี วันที่ 365 จึง
ตรงกับวันสิ้นปี 31 ธันวาคม)

---

## ขั้นตอนที่ 470: ตัวอย่างรวม — โปรแกรมยูทิลิตี้จัดการวันที่แบบครบวงจร

### ภาพรวมโปรแกรมสุดท้ายของ Part นี้

เราจะรวมทุกเทคนิคที่เรียนมาตลอด Part นี้เข้าด้วยกัน: การตรวจสอบความถูกต้อง, การคำนวณผลต่างวันที่,
การคำนวณอายุที่ถูกต้อง, การตรวจสอบปีอธิกสุรทิน, การหาวันในสัปดาห์, และการจัดรูปแบบวันที่ด้วยเทคนิคที่
เชื่อถือได้ ให้เป็นโปรแกรมยูทิลิตี้เดียวที่สมบูรณ์

### โปรแกรมสมบูรณ์: FINAL470

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. FINAL470.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATE1        PIC 9(8) VALUE 20000315.
       01  WS-DATE1-X REDEFINES WS-DATE1.
           05  WS-D1-YYYY  PIC 9(4).
           05  WS-D1-MM    PIC 9(2).
           05  WS-D1-DD    PIC 9(2).
       01  WS-DATE2        PIC 9(8) VALUE 20260926.
       01  WS-DATE2-X REDEFINES WS-DATE2.
           05  WS-D2-YYYY  PIC 9(4).
           05  WS-D2-MM    PIC 9(2).
           05  WS-D2-DD    PIC 9(2).
       01  WS-CHECK        PIC S9(8).
       01  WS-INT1         PIC 9(8).
       01  WS-INT2         PIC 9(8).
       01  WS-DIFF-DAYS    PIC S9(8).
       01  WS-AGE          PIC 9(3).
       01  WS-DOW          PIC 9(1).
       01  WS-DOW-NAME.
           05  FILLER PIC X(10) VALUE "SUNDAY    ".
           05  FILLER PIC X(10) VALUE "MONDAY    ".
           05  FILLER PIC X(10) VALUE "TUESDAY   ".
           05  FILLER PIC X(10) VALUE "WEDNESDAY ".
           05  FILLER PIC X(10) VALUE "THURSDAY  ".
           05  FILLER PIC X(10) VALUE "FRIDAY    ".
           05  FILLER PIC X(10) VALUE "SATURDAY  ".
       01  WS-DOW-TABLE REDEFINES WS-DOW-NAME.
           05  WS-DOW-ITEM OCCURS 7 TIMES PIC X(10).
       01  WS-LEAP-FLAG    PIC X(10).
       01  WS-DISPLAY-DATE PIC X(10).
       PROCEDURE DIVISION.
      *> 1. Validate both dates
           COMPUTE WS-CHECK = FUNCTION TEST-DATE-YYYYMMDD(WS-DATE1)
           IF WS-CHECK NOT = 0
               DISPLAY "WS-DATE1 is invalid, code=" WS-CHECK
               STOP RUN
           END-IF
           COMPUTE WS-CHECK = FUNCTION TEST-DATE-YYYYMMDD(WS-DATE2)
           IF WS-CHECK NOT = 0
               DISPLAY "WS-DATE2 is invalid, code=" WS-CHECK
               STOP RUN
           END-IF
           DISPLAY "Both dates are valid."

      *> 2. Difference in days between the two dates
           COMPUTE WS-INT1 = FUNCTION INTEGER-OF-DATE(WS-DATE1)
           COMPUTE WS-INT2 = FUNCTION INTEGER-OF-DATE(WS-DATE2)
           COMPUTE WS-DIFF-DAYS = WS-INT2 - WS-INT1
           DISPLAY "Days between dates: " WS-DIFF-DAYS

      *> 3. Age calculation (accounting for whether the birthday
      *> in the current year has already passed)
           COMPUTE WS-AGE = WS-D2-YYYY - WS-D1-YYYY
           IF WS-D2-MM < WS-D1-MM OR
               (WS-D2-MM = WS-D1-MM AND WS-D2-DD < WS-D1-DD)
               SUBTRACT 1 FROM WS-AGE
           END-IF
           DISPLAY "Age as of second date: " WS-AGE " years"

      *> 4. Leap year check on the second date's year
           IF FUNCTION MOD(WS-D2-YYYY, 4) = 0 AND
               (FUNCTION MOD(WS-D2-YYYY, 100) NOT = 0 OR
                FUNCTION MOD(WS-D2-YYYY, 400) = 0)
               MOVE "LEAP YEAR" TO WS-LEAP-FLAG
           ELSE
               MOVE "NOT LEAP" TO WS-LEAP-FLAG
           END-IF
           DISPLAY WS-D2-YYYY " is a " WS-LEAP-FLAG

      *> 5. Day of week for the second date
           COMPUTE WS-DOW = FUNCTION MOD(WS-INT2, 7)
           DISPLAY "Day of week: "
               FUNCTION TRIM(WS-DOW-ITEM(WS-DOW + 1))

      *> 6. Manual formatting for display
           STRING WS-D2-DD "/" WS-D2-MM "/" WS-D2-YYYY
               DELIMITED BY SIZE INTO WS-DISPLAY-DATE
           DISPLAY "Formatted: " WS-DISPLAY-DATE
           STOP RUN.
```

**ผลลัพธ์:**

```
Both dates are valid.
Days between dates: +00009691
Age as of second date: 026 years
2026 is a NOT LEAP
Day of week: SATURDAY
Formatted: 26/09/2026
```

### อธิบายจุดสำคัญ

- โปรแกรมนี้ดำเนินการ 6 ขั้นตอนตามลำดับ โดยแต่ละขั้นตอนใช้เทคนิคที่เรียนมาก่อนหน้าใน Part นี้:
  ตรวจสอบความถูกต้อง (ขั้นตอนที่ 462) → คำนวณผลต่างวันที่ (463) → คำนวณอายุที่ถูกต้อง (466) →
  ตรวจสอบปีอธิกสุรทิน (464) → หาวันในสัปดาห์ (465) → จัดรูปแบบแสดงผล (467)
- การตรวจสอบความถูกต้องของวันที่ทั้งสองก่อนเริ่มคำนวณ (`STOP RUN` ทันทีถ้าไม่ถูกต้อง) เป็นแนวปฏิบัติ
  ที่ดีที่ป้องกันการคำนวณต่อด้วยข้อมูลที่ผิดพลาด ซึ่งอาจนำไปสู่ผลลัพธ์ที่ผิดเพี้ยนแบบเงียบ ๆ ในขั้นตอน
  ถัดไปทั้งหมด
- ผลลัพธ์ทั้งหมดสอดคล้องกับความเป็นจริง: ผู้ที่เกิดวันที่ 15 มีนาคม 2000 มีอายุ 26 ปีเต็ม ณ วันที่
  26 กันยายน 2026 (เพราะวันเกิดของปีนี้ผ่านไปแล้ว), ปี 2026 ไม่ใช่ปีอธิกสุรทิน, และวันที่ 26 กันยายน
  2026 ตรงกับวันเสาร์

### ข้อควรระวัง

- โปรแกรมตัวอย่างนี้ใช้ literal ค่าคงที่สำหรับวันที่ทั้งสอง ในระบบจริงควรรับค่าจากผู้ใช้หรือไฟล์ข้อมูล
  แล้วผ่านการตรวจสอบความถูกต้องแบบเดียวกันนี้ก่อนนำไปประมวลผลเสมอ
- ทุกเทคนิคในโปรแกรมนี้ (ยกเว้นการจัดรูปแบบด้วย `STRING`) พึ่งพา `FUNCTION INTEGER-OF-DATE` เป็นแกน
  หลัก ตอกย้ำว่าฟังก์ชันนี้เป็นเครื่องมือสำคัญที่สุดตัวหนึ่งสำหรับงานคำนวณวันที่ใน COBOL ควรทำความ
  เข้าใจให้แม่นยำก่อนนำไปประยุกต์ใช้กับสถานการณ์ทางธุรกิจอื่น ๆ

### แบบฝึกหัดที่ 470.1

**โจทย์**: จงต่อยอดโปรแกรม `FINAL470` ให้แสดงข้อความเพิ่มเติมว่า "ELIGIBLE FOR SENIOR DISCOUNT" หาก
อายุที่คำนวณได้มากกว่าหรือเท่ากับ 60 ปี มิฉะนั้นแสดง "NOT ELIGIBLE"

**เฉลย**: เพิ่มโค้ดหลังขั้นตอนคำนวณอายุ (ขั้นตอนที่ 3 ในโปรแกรม):

```cobol
           DISPLAY "Age as of second date: " WS-AGE " years"
           IF WS-AGE >= 60
               DISPLAY "ELIGIBLE FOR SENIOR DISCOUNT"
           ELSE
               DISPLAY "NOT ELIGIBLE"
           END-IF
```

เนื่องจากอายุที่คำนวณได้คือ 26 ปี (น้อยกว่า 60) ผลลัพธ์ที่เพิ่มเข้ามาจะแสดง "NOT ELIGIBLE"

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้การจัดการวันที่และเวลาขั้นสูงใน COBOL อย่างครบถ้วน โดยทุกเทคนิคผ่านการ
ทดสอบจริงและยืนยันว่าใช้งานได้ถูกต้องในบิลด์ GnuCOBOL ของหลักสูตร:

- หลักการ "วันที่แบบจำนวนเต็มต่อเนื่อง" ผ่าน `FUNCTION INTEGER-OF-DATE`/`FUNCTION DATE-OF-INTEGER`
  เป็นแกนหลักของการคำนวณวันที่ทุกประเภท
- การตรวจสอบความถูกต้องของวันที่ด้วย `FUNCTION TEST-DATE-YYYYMMDD` โดยไม่ทำให้โปรแกรม crash
- การคำนวณจำนวนวันระหว่างสองวันที่ และการบวก/ลบจำนวนวันเข้ากับวันที่
- การตรวจสอบปีอธิกสุรทินด้วย `FUNCTION MOD` ตามกฎ 3 ชั้นของปฏิทินเกรกอเรียน
- การหาวันในสัปดาห์ด้วยเทคนิค modulo 7 บนเลขจำนวนเต็มวันที่ (พร้อมย้ำความสำคัญของการทดสอบเทียบกับ
  ปฏิทินจริงก่อนใช้งาน)
- การคำนวณอายุที่ถูกต้องโดยพิจารณาว่าวันเกิดของปีปัจจุบันผ่านไปแล้วหรือยัง
- เทคนิคจัดรูปแบบวันที่ที่เชื่อถือได้ด้วย `REDEFINES` + `STRING`
- **ข้อค้นพบสำคัญจากการทดสอบจริง**: `FUNCTION FORMATTED-DATE`/`FORMATTED-CURRENT-DATE` มีข้อจำกัด
  เรื่อง format string (ต้องมีส่วนเวลาเสมอ) และ `FORMATTED-DATE` (ที่รับวันที่เป็นพารามิเตอร์) คืนค่า
  ว่างเปล่าอย่างเงียบ ๆ แม้ compile ผ่าน — จึงแนะนำเทคนิค `REDEFINES` + `STRING` เป็นทางเลือกหลัก
- รูปแบบวันที่แบบ Julian (`YYYYDDD`) ผ่าน `FUNCTION INTEGER-OF-DAY`/`FUNCTION DAY-OF-INTEGER`
- โปรแกรมยูทิลิตี้รวมที่ผสานทุกเทคนิคเข้าด้วยกันเป็นระบบตรวจสอบและคำนวณวันที่ที่ใช้งานได้จริง

Part นี้ปิดท้ายเนื้อหาเรื่องการแลกเปลี่ยนข้อมูลและการคำนวณเชิงเวลาของเฟส 3 ใน Part ถัดไปเราจะเปลี่ยน
ทิศทางไปสู่หัวข้อสำคัญอีกด้านหนึ่งของการเขียนโปรแกรมระดับมืออาชีพ: **Error Handling ขั้นสูง** ผ่าน
คำสั่ง `USE Statement` และ `DECLARATIVES` ซึ่งเป็นกลไกมาตรฐานของ COBOL สำหรับดักจับและจัดการข้อผิด
พลาดของไฟล์และการประมวลผลอย่างเป็นระบบ แทนการตรวจสอบทีละจุดด้วย `IF` เหมือนที่ผ่านมา

**[← กลับไป Part 046: XML GENERATE และ XML PARSE](part-046-xml-generate-parse.md)** | **[ไปยัง Part 048: Error Handling ขั้นสูง — USE Statement และ DECLARATIVES →](part-048-use-declaratives.md)**
