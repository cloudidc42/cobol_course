# Part 027: SORT และ MERGE Statement (ขั้นตอนที่ 261–270)

## คำนำของ Part นี้

Part 026 ปิดท้ายด้วยข้อสังเกตสำคัญ: อัลกอริทึม Master-Detail Matching ที่เราสร้างขึ้นทั้งหมด**ต้องการ
ให้ทั้งสองไฟล์เรียงลำดับตาม Key เดียวกันมาก่อนแล้วเสมอ** แต่ในโลกจริง ไฟล์ที่รับเข้ามาจากระบบภายนอก
(เช่น ไฟล์ธุรกรรมจาก ATM, ไฟล์ที่ส่งมาจากสาขาต่าง ๆ) มักไม่เรียงลำดับมาให้ Part นี้จะเติมเต็มช่องว่าง
นั้นด้วยคำสั่งสำคัญ 2 ตัวของ COBOL ที่ออกแบบมาเพื่องานนี้โดยเฉพาะ: **`SORT`** (เรียงลำดับไฟล์ที่ยังไม่
เรียง) และ **`MERGE`** (รวมไฟล์ที่เรียงลำดับแล้วหลายไฟล์เข้าด้วยกัน)

สิ่งที่ทำให้ทั้งสองคำสั่งนี้พิเศษกว่าคำสั่งไฟล์อื่น ๆ ที่เคยเรียนมาคือมันมี **ไฟล์ทำงานพิเศษ (Sort
Work File)** ที่ประกาศด้วย `SD` แทนที่จะเป็น `FD` และมีคำสั่งคู่หูของตัวเอง (`RELEASE`/`RETURN`) แทนที่
จะใช้ `WRITE`/`READ` ตรง ๆ Part นี้จะพาคุณเรียนรู้ตั้งแต่การใช้งานพื้นฐานที่สุด ไปจนถึงการนำผลลัพธ์
จาก `SORT`/`MERGE` ไปป้อนเข้าสู่อัลกอริทึม Master-Detail จาก Part 026 โดยตรง ปิดท้ายด้วยโปรแกรม
รวมที่จำลองไปป์ไลน์งาน Batch แบบสมบูรณ์เหมือนที่ใช้จริงในองค์กรขนาดใหญ่

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน โค้ดทุกตัวอย่าง
> ผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง

---

## ขั้นตอนที่ 261: SORT Statement พื้นฐาน และ SD Entry (Sort Description)

### ไฟล์พิเศษสำหรับ SORT: SD แทน FD

`SORT` ต้องการไฟล์ชั่วคราวสำหรับใช้งานระหว่างการเรียงลำดับ ซึ่งประกาศด้วยคำสงวน **`SD`** (Sort
Description) แทนที่จะเป็น `FD` ที่คุ้นเคย ไฟล์ `SD` นี้**ไม่ต้อง `OPEN`/`CLOSE` เอง** — คำสั่ง `SORT`
จะจัดการเปิด-ปิดให้อัตโนมัติทั้งหมด

### ตัวอย่าง: SORT แบบง่ายที่สุด (USING...GIVING)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP261-SORT-BASICS.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT UNSORTED-FILE ASSIGN TO "UNSORTED261.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORTED-FILE ASSIGN TO "SORTED261.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK261.DAT".

       DATA DIVISION.
       FILE SECTION.
       FD  UNSORTED-FILE.
       01  UNS-RECORD.
           05  UNS-ID                  PIC X(4).
           05  UNS-NAME                PIC X(15).

       FD  SORTED-FILE.
       01  SRT-RECORD.
           05  SRT-ID                  PIC X(4).
           05  SRT-NAME                PIC X(15).

      *> SD (Sort Description) - NOT FD - describes the temporary
      *> work file the SORT verb uses internally while sorting.
       SD  SORT-WORK-FILE.
       01  SORT-RECORD.
           05  SORT-ID                 PIC X(4).
           05  SORT-NAME               PIC X(15).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT UNSORTED-FILE.
           MOVE "A003" TO UNS-ID.
           MOVE "PRASERT KAEWTA " TO UNS-NAME.
           WRITE UNS-RECORD.
           MOVE "A001" TO UNS-ID.
           MOVE "SOMCHAI JAIDEE " TO UNS-NAME.
           WRITE UNS-RECORD.
           MOVE "A002" TO UNS-ID.
           MOVE "SUDA MEECHAI   " TO UNS-NAME.
           WRITE UNS-RECORD.
           CLOSE UNSORTED-FILE.

           DISPLAY "Before SORT: A003, A001, A002 (out of order)".

      *> SORT reads every record from UNSORTED-FILE, sorts them
      *> by SORT-ID, and writes the result straight to
      *> SORTED-FILE. GnuCOBOL opens/closes both files for us.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SORT-ID
               USING UNSORTED-FILE
               GIVING SORTED-FILE.

           OPEN INPUT SORTED-FILE.
           DISPLAY "After SORT:".
           PERFORM 3 TIMES
               READ SORTED-FILE
               DISPLAY "  " SRT-ID " " SRT-NAME
           END-PERFORM.
           CLOSE SORTED-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Before SORT: A003, A001, A002 (out of order)
After SORT:
  A001 SOMCHAI JAIDEE
  A002 SUDA MEECHAI
  A003 PRASERT KAEWTA
```

### อธิบายจุดสำคัญ

- `SD SORT-WORK-FILE.` ประกาศไฟล์ทำงานชั่วคราว มี `01 SORT-RECORD` กำหนดโครงสร้าง record เหมือน
  `FD` ทุกประการ แต่ **ห้าม `OPEN`/`CLOSE` ไฟล์นี้เอง** — `SORT` จัดการให้ทั้งหมด
- `SORT sort-file ON ASCENDING KEY key-field USING input-file GIVING output-file`: รูปแบบที่ง่าย
  ที่สุด อ่านทุก record จาก `USING` (ในที่นี้คือ `UNSORTED-FILE`) เรียงตาม `key-field` แล้วเขียนผล
  ลัพธ์ลง `GIVING` (ในที่นี้คือ `SORTED-FILE`) ทันที — **เราไม่ต้องเขียน `OPEN`/`CLOSE` ให้ทั้งสอง
  ไฟล์นี้เองด้วย** เพราะ `SORT` จัดการให้อัตโนมัติเช่นกัน (สังเกตว่าโค้ดไม่มี
  `OPEN INPUT UNSORTED-FILE` หรือ `OPEN OUTPUT SORTED-FILE` เลยก่อนเรียก `SORT`)
- `SORT-ID` (field ใน `SD`) ต้องมีโครงสร้าง/ตำแหน่งตรงกับ `UNS-ID`/`SRT-ID` (field ใน `FD` ทั้งสอง
  ไฟล์) เพราะ `SORT` จะคัดลอกข้อมูลผ่าน record structure ที่ตรงกันโดยอัตโนมัติ

### ข้อควรระวัง

- ชื่อไฟล์จริงที่ระบุใน `SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK261.DAT"` จะถูกสร้างขึ้นชั่วคราว
  ระหว่างการเรียงลำดับแล้วอาจถูกลบทิ้งหรือเขียนทับหลังจากนั้น อย่าพึ่งพาเนื้อหาของไฟล์นี้เพื่อจุด
  ประสงค์อื่นใด ถือว่าเป็น "พื้นที่ทำงานภายใน" ของคำสั่ง `SORT` เท่านั้น
- ห้ามเขียน `OPEN`/`CLOSE` กับไฟล์ที่ประกาศด้วย `SD` เด็ดขาด จะเกิด compile error ทันที เพราะ `SD`
  ไม่ใช่ไฟล์ที่โปรแกรมควบคุมวงจรชีวิตเอง

### แบบฝึกหัดที่ 261.1

**โจทย์**: จงอธิบายว่าทำไม `SORT` จึงไม่ต้องการให้เราเขียน `OPEN`/`CLOSE` ให้กับ `UNSORTED-FILE`
และ `SORTED-FILE` เองในรูปแบบ `USING...GIVING` ทั้งที่ทั้งสองไฟล์นี้ประกาศด้วย `FD` ตามปกติ

**เฉลย**: เพราะรูปแบบ `USING...GIVING` เป็นรูปแบบที่ง่ายที่สุดของ `SORT` ซึ่งออกแบบมาให้จัดการทุก
ขั้นตอน (เปิดไฟล์ต้นทางเพื่ออ่าน, เรียงลำดับ, เปิดไฟล์ปลายทางเพื่อเขียนผลลัพธ์, และปิดไฟล์ทั้งสอง)
ให้อัตโนมัติภายในคำสั่งเดียว ผู้เขียนโปรแกรมจึงไม่จำเป็นต้องเขียน `OPEN`/`CLOSE` ให้ไฟล์เหล่านี้เอง
(ต่างจากการทำงานกับไฟล์ปกติที่ต้องเขียน `OPEN`/`CLOSE` ทุกครั้ง) ความสะดวกนี้เป็นข้อดีหลักของรูปแบบ
`USING...GIVING` เมื่อเทียบกับรูปแบบที่ซับซ้อนกว่าอย่าง `INPUT PROCEDURE`/`OUTPUT PROCEDURE` ที่จะ
เรียนในขั้นตอนถัดไป

---

## ขั้นตอนที่ 262: เรียงลำดับหลาย Key พร้อมกัน — ASCENDING และ DESCENDING

### เรียงลำดับซ้อนกันหลายชั้น

`SORT` รองรับการระบุ Key **หลายตัวพร้อมกัน** โดยแต่ละ Key จะเรียงลำดับตามลำดับความสำคัญที่เขียนไว้
(Key แรกสำคัญที่สุด, ถ้าเท่ากันค่อยดู Key ถัดไป) และแต่ละ Key สามารถกำหนดทิศทาง **`ASCENDING`**
(น้อยไปมาก) หรือ **`DESCENDING`** (มากไปน้อย) แยกกันได้อย่างอิสระ

### ตัวอย่าง: เรียงตามแผนก (น้อยไปมาก) แล้วเรียงตามเงินเดือน (มากไปน้อย) ภายในแผนกเดียวกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP262-SORT-MULTI-KEY.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT UNSORTED-FILE ASSIGN TO "UNSORTED262.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORTED-FILE ASSIGN TO "SORTED262.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK262.DAT".

       DATA DIVISION.
       FILE SECTION.
       FD  UNSORTED-FILE.
       01  UNS-RECORD.
           05  UNS-DEPT                PIC X(5).
           05  UNS-SALARY              PIC 9(6).
           05  UNS-NAME                PIC X(15).

       FD  SORTED-FILE.
       01  SRT-RECORD.
           05  SRT-DEPT                PIC X(5).
           05  SRT-SALARY              PIC 9(6).
           05  SRT-NAME                PIC X(15).

       SD  SORT-WORK-FILE.
       01  SORT-RECORD.
           05  SORT-DEPT               PIC X(5).
           05  SORT-SALARY             PIC 9(6).
           05  SORT-NAME               PIC X(15).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT UNSORTED-FILE.
           MOVE "SALES" TO UNS-DEPT.
           MOVE 30000 TO UNS-SALARY.
           MOVE "SOMCHAI JAIDEE " TO UNS-NAME.
           WRITE UNS-RECORD.

           MOVE "IT   " TO UNS-DEPT.
           MOVE 45000 TO UNS-SALARY.
           MOVE "SUDA MEECHAI   " TO UNS-NAME.
           WRITE UNS-RECORD.

           MOVE "SALES" TO UNS-DEPT.
           MOVE 50000 TO UNS-SALARY.
           MOVE "PRASERT KAEWTA " TO UNS-NAME.
           WRITE UNS-RECORD.

           MOVE "IT   " TO UNS-DEPT.
           MOVE 60000 TO UNS-SALARY.
           MOVE "WANNA SRISUK   " TO UNS-NAME.
           WRITE UNS-RECORD.
           CLOSE UNSORTED-FILE.

      *> Two-key sort: group by DEPT (ascending), then within
      *> each department, highest salary first (descending).
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SORT-DEPT
               ON DESCENDING KEY SORT-SALARY
               USING UNSORTED-FILE
               GIVING SORTED-FILE.

           OPEN INPUT SORTED-FILE.
           DISPLAY "Sorted by DEPT ascending, SALARY descending:".
           PERFORM 4 TIMES
               READ SORTED-FILE
               DISPLAY "  " SRT-DEPT " " SRT-SALARY " " SRT-NAME
           END-PERFORM.
           CLOSE SORTED-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Sorted by DEPT ascending, SALARY descending:
  IT    060000 WANNA SRISUK
  IT    045000 SUDA MEECHAI
  SALES 050000 PRASERT KAEWTA
  SALES 030000 SOMCHAI JAIDEE
```

### อธิบายจุดสำคัญ

- แผนก `IT` มาก่อน `SALES` เพราะเรียงตามตัวอักษร (I มาก่อน S) ตาม `ON ASCENDING KEY SORT-DEPT`
- ภายในแผนกเดียวกัน คนเงินเดือนสูงกว่าปรากฏก่อนเสมอ (60000 ก่อน 45000 ในแผนก IT, 50000 ก่อน 30000
  ในแผนก SALES) ตาม `ON DESCENDING KEY SORT-SALARY`
- สามารถเขียน `ON ASCENDING KEY key1 key2 ...` (Key หลายตัวในบรรทัดเดียวที่มีทิศทางเดียวกัน) หรือ
  แยกเป็นหลายบรรทัด `ON ASCENDING KEY ... ON DESCENDING KEY ...` (แบบในตัวอย่างนี้) เมื่อแต่ละ Key
  ต้องการทิศทางต่างกัน ก็ได้ทั้งสองแบบ

### ข้อควรระวัง

- ลำดับของ Key **สำคัญมาก** — Key ที่เขียนก่อนจะถูกใช้เปรียบเทียบก่อนเสมอ การสลับลำดับ (เช่น
  เอา `SORT-SALARY` ขึ้นก่อน `SORT-DEPT`) จะให้ผลลัพธ์ที่ต่างไปโดยสิ้นเชิง (เรียงตามเงินเดือนก่อน
  แล้วค่อยแยกแผนก แทนที่จะเป็นแยกแผนกก่อนแล้วค่อยเรียงเงินเดือนภายในแผนก)
- field ที่ใช้เป็น Key ควรเป็นชนิดข้อมูลที่เปรียบเทียบได้อย่างมีความหมาย (ตัวเลขหรือตัวอักษรตาม
  ธรรมชาติของข้อมูล) การใช้ field ที่มีรูปแบบไม่สม่ำเสมอ (เช่น field ตัวอักษรแต่บางที record มี
  ตัวพิมพ์เล็ก บางทีมีตัวพิมพ์ใหญ่) จะให้ผลการเรียงที่ดูแปลกเพราะอิงลำดับไบต์ล้วน ๆ

### แบบฝึกหัดที่ 262.1

**โจทย์**: จงอธิบายผลลัพธ์ที่จะได้ถ้าสลับลำดับ Key เป็น `ON DESCENDING KEY SORT-SALARY` ก่อน
แล้วตามด้วย `ON ASCENDING KEY SORT-DEPT`

**เฉลย**: ผลลัพธ์จะเรียงตามเงินเดือนจากมากไปน้อยเป็นหลักก่อน (60000, 50000, 45000, 30000) โดยไม่
สนใจแผนกเลยจนกว่าจะมีเงินเดือนเท่ากันพอดี (ซึ่งในข้อมูลชุดนี้ไม่มี) ผลลัพธ์ที่ได้คือ: WANNA SRISUK
(IT, 60000), PRASERT KAEWTA (SALES, 50000), SUDA MEECHAI (IT, 45000), SOMCHAI JAIDEE (SALES,
30000) — จะเห็นว่าแผนก IT และ SALES ปะปนกันไปมา ไม่ได้จัดกลุ่มตามแผนกอีกต่อไป เพราะ Key ตัวแรก
(เงินเดือน) มีความสำคัญเหนือกว่า Key ตัวที่สอง (แผนก) เสมอ

---

## ขั้นตอนที่ 263: SORT พร้อม INPUT PROCEDURE — กรองข้อมูลก่อนเรียงลำดับ

### เมื่อ USING ไม่พอ: ต้องการกรอง/แปลงข้อมูลก่อนเรียง

รูปแบบ `USING` ใน 2 ขั้นตอนก่อนหน้าจะดึงข้อมูล**ทุก record**จากไฟล์ต้นทางเข้าสู่กระบวนการเรียงลำดับ
โดยอัตโนมัติ แต่บางครั้งเราต้องการ**กรองข้อมูลที่ไม่ต้องการทิ้งก่อน** (เช่น ธุรกรรมที่มีจำนวนเงินเป็น
ศูนย์ ซึ่งถือเป็นข้อมูลขยะ) COBOL มี **`INPUT PROCEDURE`** สำหรับสถานการณ์นี้: แทนที่จะให้ `SORT`
ดึงข้อมูลจากไฟล์ตรง ๆ เราเขียน paragraph ของเราเองที่ควบคุมว่า record ไหนจะถูกส่งเข้าสู่กระบวนการ
เรียงลำดับบ้าง ผ่านคำสั่ง **`RELEASE`** (ทำหน้าที่เหมือน `WRITE` แต่สำหรับ Sort Work File)

### ตัวอย่าง: กรองธุรกรรมจำนวนเงินเป็นศูนย์ทิ้งก่อนเรียงลำดับ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP263-INPUT-PROCEDURE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT UNSORTED-FILE ASSIGN TO "UNSORTED263.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORTED-FILE ASSIGN TO "SORTED263.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK263.DAT".

       DATA DIVISION.
       FILE SECTION.
       FD  UNSORTED-FILE.
       01  UNS-RECORD.
           05  UNS-ID                  PIC X(4).
           05  UNS-AMOUNT              PIC 9(6).

       FD  SORTED-FILE.
       01  SRT-RECORD.
           05  SRT-ID                  PIC X(4).
           05  SRT-AMOUNT              PIC 9(6).

       SD  SORT-WORK-FILE.
       01  SORT-RECORD.
           05  SORT-ID                 PIC X(4).
           05  SORT-AMOUNT             PIC 9(6).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG                 PIC X VALUE "N".
           88  END-OF-FILE             VALUE "Y".
       01  WS-SKIPPED-COUNT            PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT UNSORTED-FILE.
           MOVE "A003" TO UNS-ID.
           MOVE 50 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "A001" TO UNS-ID.
           MOVE 300 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "A002" TO UNS-ID.
           MOVE 0 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "A004" TO UNS-ID.
           MOVE 175 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           CLOSE UNSORTED-FILE.

      *> INPUT PROCEDURE lets us filter/transform records BEFORE
      *> they even enter the sort - here we drop zero-amount
      *> transactions instead of sorting (and keeping) junk data.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SORT-ID
               INPUT PROCEDURE IS FILTER-AND-FEED
               GIVING SORTED-FILE.

           DISPLAY "Skipped (zero amount): " WS-SKIPPED-COUNT.
           MOVE "N" TO WS-EOF-FLAG.
           OPEN INPUT SORTED-FILE.
           DISPLAY "Sorted (zero-amount records removed):".
           PERFORM UNTIL END-OF-FILE
               READ SORTED-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  " SRT-ID " " SRT-AMOUNT
               END-READ
           END-PERFORM.
           CLOSE SORTED-FILE.
           STOP RUN.

       FILTER-AND-FEED.
           OPEN INPUT UNSORTED-FILE.
           PERFORM UNTIL END-OF-FILE
               READ UNSORTED-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       IF UNS-AMOUNT = 0
                           ADD 1 TO WS-SKIPPED-COUNT
                       ELSE
      *> RELEASE feeds one record into the sort's work file -
      *> it plays the same role WRITE plays for a normal file.
                           MOVE UNS-ID TO SORT-ID
                           MOVE UNS-AMOUNT TO SORT-AMOUNT
                           RELEASE SORT-RECORD
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE UNSORTED-FILE.
```

**ผลลัพธ์:**

```
Skipped (zero amount): 001
Sorted (zero-amount records removed):
  A001 000300
  A003 000050
  A004 000175
```

### อธิบายจุดสำคัญ

- `INPUT PROCEDURE IS FILTER-AND-FEED` บอก `SORT` ว่า **"อย่าดึงข้อมูลจากไฟล์เอง ให้เรียก
  paragraph `FILTER-AND-FEED` แล้วรอรับ record ที่ paragraph นั้นส่งเข้ามาแทน"**
- `RELEASE SORT-RECORD` คือหัวใจของ `INPUT PROCEDURE` — มันทำหน้าที่เหมือน `WRITE` ทุกประการ แต่
  เขียนเข้า **Sort Work File** (`SD`) แทนที่จะเป็นไฟล์ปกติ ทุกครั้งที่เรียก `RELEASE` จะมี record
  หนึ่งตัวถูกส่งเข้าสู่กระบวนการเรียงลำดับ
- ใน `FILTER-AND-FEED` เราเปิด-ปิด `UNSORTED-FILE` เอง (ต่างจากรูปแบบ `USING` ที่ `SORT` เปิดให้
  อัตโนมัติ) เพราะตอนนี้**เรา**เป็นผู้ควบคุมการอ่านไฟล์ต้นทางเองทั้งหมด

### ข้อควรระวัง

- **สังเกตบรรทัด `MOVE "N" TO WS-EOF-FLAG` ก่อนเปิด `SORTED-FILE` รอบที่สอง** — นี่คือบั๊กจริงที่
  พบระหว่างการพัฒนาตัวอย่างนี้! `WS-EOF-FLAG` ถูกตั้งเป็น `"Y"` ค้างมาจากลูปใน `FILTER-AND-FEED`
  (ที่อ่าน `UNSORTED-FILE` จนจบ) ถ้าลืมรีเซ็ตกลับเป็น `"N"` ก่อนลูปที่สองที่อ่าน `SORTED-FILE`
  ลูปนั้นจะไม่ทำงานเลยแม้แต่ครั้งเดียว (ผลลัพธ์จะว่างเปล่า) — นี่คือข้อผิดพลาดแบบเดียวกับที่เตือน
  ไว้ตั้งแต่ Part 023 ขั้นตอนที่ 226 และยังคงพบได้บ่อยเมื่อใช้ตัวแปร EOF-Flag ตัวเดียวกันข้าม
  หลายไฟล์/หลายลูป
- `FILTER-AND-FEED` ต้องเป็น **paragraph** (ไม่ใช่ `SECTION`) ในรูปแบบพื้นฐาน และต้อง `OPEN`/
  `CLOSE` ไฟล์ต้นทาง (`UNSORTED-FILE`) ให้ครบเองภายใน paragraph นั้น มิฉะนั้นจะอ่านไฟล์ไม่ได้

### แบบฝึกหัดที่ 263.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ `INPUT PROCEDURE` จึงยืดหยุ่นกว่ารูปแบบ `USING` ธรรมดา แม้จะต้อง
เขียนโค้ดเพิ่มขึ้น

**เฉลย**: เพราะรูปแบบ `USING` จะดึง**ทุก record**จากไฟล์ต้นทางเข้าสู่กระบวนการเรียงลำดับโดยไม่มี
การควบคุมใด ๆ ในขณะที่ `INPUT PROCEDURE` ให้เราเขียนตรรกะเองได้อย่างอิสระก่อนตัดสินใจว่าจะ
`RELEASE` record นั้นเข้าสู่การเรียงลำดับหรือไม่ ทำให้สามารถกรองข้อมูลขยะ (เหมือนตัวอย่างนี้),
แปลงรูปแบบข้อมูลก่อนเรียง, หรือแม้แต่รวมข้อมูลจากหลายไฟล์เข้าด้วยกันก่อนส่งเข้า `SORT` ได้ในขั้นตอน
เดียวกัน ซึ่งเป็นสิ่งที่ `USING` ทำไม่ได้เลย

---

## ขั้นตอนที่ 264: SORT พร้อม OUTPUT PROCEDURE — ประมวลผลหลังเรียงลำดับ

### เมื่อ GIVING ไม่พอ: ต้องการประมวลผลผลลัพธ์ที่เรียงแล้วก่อนเขียนไฟล์จริง

เช่นเดียวกับ `INPUT PROCEDURE` ที่ควบคุมข้อมูล**ก่อน**เข้าสู่การเรียงลำดับ **`OUTPUT PROCEDURE`**
ให้เราควบคุมข้อมูล**หลัง**เรียงลำดับเสร็จแล้ว ก่อนที่จะเขียนผลลัพธ์สุดท้ายลงไฟล์จริง ผ่านคำสั่ง
**`RETURN`** (ทำหน้าที่เหมือน `READ` แต่ดึงจาก Sort Work File ที่เรียงลำดับเสร็จแล้ว)

### ตัวอย่าง: คำนวณยอดสะสม (Running Total) ระหว่างเขียนรายงาน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP264-OUTPUT-PROCEDURE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT UNSORTED-FILE ASSIGN TO "UNSORTED264.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT REPORT-FILE ASSIGN TO "REPORT264.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK264.DAT".

       DATA DIVISION.
       FILE SECTION.
       FD  UNSORTED-FILE.
       01  UNS-RECORD.
           05  UNS-ID                  PIC X(4).
           05  UNS-AMOUNT              PIC 9(6).

       FD  REPORT-FILE.
       01  RPT-RECORD                  PIC X(50).

       SD  SORT-WORK-FILE.
       01  SORT-RECORD.
           05  SORT-ID                 PIC X(4).
           05  SORT-AMOUNT             PIC 9(6).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG                 PIC X VALUE "N".
           88  END-OF-SORT              VALUE "Y".
       01  WS-RUNNING-TOTAL            PIC 9(7) VALUE 0.
       01  WS-DISPLAY-TOTAL            PIC ZZZ,ZZ9.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT UNSORTED-FILE.
           MOVE "A003" TO UNS-ID.
           MOVE 150 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "A001" TO UNS-ID.
           MOVE 300 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "A002" TO UNS-ID.
           MOVE 500 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           CLOSE UNSORTED-FILE.

      *> OUTPUT PROCEDURE receives records ALREADY sorted, one at
      *> a time via RETURN, and can compute a running total before
      *> writing the final human-readable report.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SORT-ID
               USING UNSORTED-FILE
               OUTPUT PROCEDURE IS BUILD-REPORT.

           OPEN INPUT REPORT-FILE.
           MOVE "N" TO WS-EOF-FLAG.
           DISPLAY "Report file content:".
           PERFORM UNTIL END-OF-SORT
               READ REPORT-FILE
                   AT END
                       SET END-OF-SORT TO TRUE
                   NOT AT END
                       DISPLAY "  " RPT-RECORD
               END-READ
           END-PERFORM.
           CLOSE REPORT-FILE.
           STOP RUN.

       BUILD-REPORT.
           OPEN OUTPUT REPORT-FILE.
           MOVE "N" TO WS-EOF-FLAG.
           PERFORM UNTIL END-OF-SORT
      *> RETURN pulls the next SORTED record out of the sort -
      *> it plays the same role READ plays for a normal file.
               RETURN SORT-WORK-FILE
                   AT END
                       SET END-OF-SORT TO TRUE
                   NOT AT END
                       ADD SORT-AMOUNT TO WS-RUNNING-TOTAL
                       MOVE WS-RUNNING-TOTAL TO WS-DISPLAY-TOTAL
                       MOVE SPACES TO RPT-RECORD
                       STRING SORT-ID " amount=" SORT-AMOUNT
                           " running total=" WS-DISPLAY-TOTAL
                           DELIMITED BY SIZE
                           INTO RPT-RECORD
                       WRITE RPT-RECORD
               END-RETURN
           END-PERFORM.
           CLOSE REPORT-FILE.
```

**ผลลัพธ์:**

```
Report file content:
  A001 amount=000300 running total=    300
  A002 amount=000500 running total=    800
  A003 amount=000150 running total=    950
```

### อธิบายจุดสำคัญ

- `RETURN SORT-WORK-FILE AT END ... NOT AT END ...` มีโครงสร้างเหมือน `READ...AT END` ทุกประการ
  เพียงแค่เปลี่ยนคำสั่งและแหล่งข้อมูลเป็น Sort Work File ที่**เรียงลำดับเสร็จแล้ว** (สังเกตว่า
  ผลลัพธ์ออกมาตามลำดับ A001, A002, A003 แม้ข้อมูลต้นทางจะเขียนเป็น A003, A001, A002)
- ยอดสะสม (running total) คำนวณถูกต้องตามลำดับที่เรียงแล้ว: 300 → 300+500=800 → 800+150=950
  ซึ่งพิสูจน์ว่า `OUTPUT PROCEDURE` ได้รับข้อมูลตามลำดับที่ถูกเรียงจริง ไม่ใช่ลำดับดั้งเดิม
- `BUILD-REPORT` ต้อง `OPEN OUTPUT REPORT-FILE` เองก่อนเริ่มลูป `RETURN` และ `CLOSE` เองหลังจบ
  ลูป เหมือนกับที่ `FILTER-AND-FEED` ในขั้นตอนก่อนหน้าต้องจัดการไฟล์ต้นทางเอง

### ข้อควรระวัง

- ต้องกำหนด `RPT-RECORD` ให้กว้างพอสำหรับข้อความที่จะ `STRING` เข้าไปเสมอ (ในตัวอย่างนี้ใช้
  `PIC X(50)` ซึ่งเพียงพอ) มิฉะนั้นข้อความจะถูกตัดทิ้งแบบเงียบ ๆ ตามกฎของ `STRING` (ทบทวนจาก
  Part 019)
- `WS-EOF-FLAG` ในตัวอย่างนี้ถูกใช้ร่วมกันระหว่างลูปใน `BUILD-REPORT` (สำหรับ `RETURN`) และลูปใน
  `MAIN-PARA` (สำหรับ `READ REPORT-FILE`) จึงต้อง `MOVE "N" TO WS-EOF-FLAG` ก่อนเข้าลูปทั้งสองครั้ง
  เสมอ เช่นเดียวกับข้อควรระวังในขั้นตอนที่ 263

### แบบฝึกหัดที่ 264.1

**โจทย์**: จงอธิบายว่าทำไม `OUTPUT PROCEDURE` จึงเหมาะกับการคำนวณยอดสะสม (running total) มากกว่า
การคำนวณยอดสะสมจากไฟล์ต้นทางที่ยังไม่เรียงลำดับโดยตรง

**เฉลย**: เพราะยอดสะสมที่มีความหมาย (เช่น ยอดสะสมเรียงตาม ID จากน้อยไปมาก) ต้องการให้ข้อมูลผ่าน
เข้ามาตาม**ลำดับที่ถูกต้อง**เสียก่อน หากคำนวณยอดสะสมจากไฟล์ต้นทางที่ยังไม่เรียงลำดับ (เช่น A003,
A001, A002 ตามลำดับเดิม) ยอดสะสมที่ได้จะไม่สื่อความหมายอะไรเลยสำหรับรายงานที่ต้องการนำเสนอตามลำดับ
ID `OUTPUT PROCEDURE` รับประกันว่าข้อมูลที่ผ่านเข้ามาทาง `RETURN` จะเรียงลำดับสมบูรณ์แล้วเสมอ ทำให้
การคำนวณยอดสะสมระหว่างทางมีความหมายตรงตามที่ต้องการนำเสนอในรายงานจริง

---

## ขั้นตอนที่ 265: ใช้ INPUT PROCEDURE และ OUTPUT PROCEDURE พร้อมกัน

### รวมพลังทั้งสองฝั่งเข้าด้วยกัน

`SORT` หนึ่งคำสั่งสามารถมีทั้ง `INPUT PROCEDURE` และ `OUTPUT PROCEDURE` พร้อมกันได้ ทำให้ควบคุมได้
ทั้งข้อมูล**ก่อน**และ**หลัง**การเรียงลำดับในคำสั่งเดียว

### ตัวอย่าง: กรองข้อมูลก่อนเรียง แล้วสร้างรายงานมีเลขลำดับหลังเรียง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP265-BOTH-PROCEDURES.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT UNSORTED-FILE ASSIGN TO "UNSORTED265.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT REPORT-FILE ASSIGN TO "REPORT265.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK265.DAT".

       DATA DIVISION.
       FILE SECTION.
       FD  UNSORTED-FILE.
       01  UNS-RECORD.
           05  UNS-ID                  PIC X(4).
           05  UNS-AMOUNT              PIC 9(6).

       FD  REPORT-FILE.
       01  RPT-RECORD                  PIC X(50).

       SD  SORT-WORK-FILE.
       01  SORT-RECORD.
           05  SORT-ID                 PIC X(4).
           05  SORT-AMOUNT             PIC 9(6).

       WORKING-STORAGE SECTION.
       01  WS-U-EOF                    PIC X VALUE "N".
           88  UNSORTED-EOF             VALUE "Y".
       01  WS-S-EOF                    PIC X VALUE "N".
           88  END-OF-SORT              VALUE "Y".
       01  WS-VALID-COUNT              PIC 9(3) VALUE 0.
       01  WS-SKIPPED-COUNT            PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT UNSORTED-FILE.
           MOVE "A003" TO UNS-ID.
           MOVE 150 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "A001" TO UNS-ID.
           MOVE 0 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           MOVE "A002" TO UNS-ID.
           MOVE 500 TO UNS-AMOUNT.
           WRITE UNS-RECORD.
           CLOSE UNSORTED-FILE.

      *> Filter BEFORE sorting (INPUT PROCEDURE) and format a
      *> report AFTER sorting (OUTPUT PROCEDURE) - both in one
      *> single SORT statement.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SORT-ID
               INPUT PROCEDURE IS FILTER-ZERO-AMOUNTS
               OUTPUT PROCEDURE IS BUILD-NUMBERED-REPORT.

           OPEN INPUT REPORT-FILE.
           MOVE "N" TO WS-S-EOF.
           DISPLAY "Final report:".
           PERFORM UNTIL END-OF-SORT
               READ REPORT-FILE
                   AT END
                       SET END-OF-SORT TO TRUE
                   NOT AT END
                       DISPLAY "  " RPT-RECORD
               END-READ
           END-PERFORM.
           CLOSE REPORT-FILE.
           DISPLAY "Valid  : " WS-VALID-COUNT.
           DISPLAY "Skipped: " WS-SKIPPED-COUNT.
           STOP RUN.

       FILTER-ZERO-AMOUNTS.
           OPEN INPUT UNSORTED-FILE.
           PERFORM UNTIL UNSORTED-EOF
               READ UNSORTED-FILE
                   AT END
                       SET UNSORTED-EOF TO TRUE
                   NOT AT END
                       IF UNS-AMOUNT = 0
                           ADD 1 TO WS-SKIPPED-COUNT
                       ELSE
                           MOVE UNS-ID TO SORT-ID
                           MOVE UNS-AMOUNT TO SORT-AMOUNT
                           RELEASE SORT-RECORD
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE UNSORTED-FILE.

       BUILD-NUMBERED-REPORT.
           OPEN OUTPUT REPORT-FILE.
           MOVE "N" TO WS-S-EOF.
           PERFORM UNTIL END-OF-SORT
               RETURN SORT-WORK-FILE
                   AT END
                       SET END-OF-SORT TO TRUE
                   NOT AT END
                       ADD 1 TO WS-VALID-COUNT
                       MOVE SPACES TO RPT-RECORD
                       STRING WS-VALID-COUNT ". " SORT-ID
                           " -- " SORT-AMOUNT
                           DELIMITED BY SIZE
                           INTO RPT-RECORD
                       WRITE RPT-RECORD
               END-RETURN
           END-PERFORM.
           CLOSE REPORT-FILE.
```

**ผลลัพธ์:**

```
Final report:
  001. A002 -- 000500
  002. A003 -- 000150
Valid  : 002
Skipped: 001
```

### อธิบายจุดสำคัญ

- ใช้ตัวแปร EOF แยกกัน 2 ตัว: `WS-U-EOF` สำหรับ `UNSORTED-FILE` (ใช้ใน `FILTER-ZERO-AMOUNTS`
  เท่านั้น) และ `WS-S-EOF` สำหรับ Sort Work File (ใช้ทั้งใน `BUILD-NUMBERED-REPORT` และใน
  `MAIN-PARA`) การแยกตัวแปรตามไฟล์ที่ควบคุมช่วยลดความสับสนและลดโอกาสเกิดบั๊กแบบขั้นตอนที่ 263
- `A001` (ที่มี `UNS-AMOUNT = 0`) ถูกกรองทิ้งใน `FILTER-ZERO-AMOUNTS` ก่อนเข้าสู่การเรียงลำดับ
  ดังนั้นรายงานสุดท้ายจึงมีแค่ 2 รายการ (A002, A003) ไม่ใช่ 3 รายการ และเรียงตามลำดับ ID ถูกต้อง
- เลขลำดับ (`001.`, `002.`) ถูกสร้างขึ้นใน `OUTPUT PROCEDURE` โดยนับจาก `WS-VALID-COUNT` ที่เพิ่ม
  ขึ้นทุกครั้งที่ `RETURN` สำเร็จ แสดงให้เห็นว่าสามารถสร้างข้อมูลใหม่ (ไม่ได้มีอยู่ในไฟล์ต้นฉบับ)
  ระหว่างขั้นตอน Output ได้อย่างอิสระ

### ข้อควรระวัง

- เมื่อใช้ทั้ง `INPUT PROCEDURE` และ `OUTPUT PROCEDURE` พร้อมกัน จะ**ไม่มี** `USING`/`GIVING` ปรากฏ
  ในคำสั่ง `SORT` เลย เพราะทั้งการนำเข้าและส่งออกถูกควบคุมโดย paragraph ที่เราเขียนเองทั้งหมด
- ต้องระวังอย่าลืม `OPEN`/`CLOSE` ไฟล์จริง (`UNSORTED-FILE` ใน `FILTER-ZERO-AMOUNTS` และ
  `REPORT-FILE` ใน `BUILD-NUMBERED-REPORT`) เพราะตอนนี้ `SORT` ไม่ได้จัดการให้แล้วเลยทั้งสองฝั่ง

### แบบฝึกหัดที่ 265.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมนี้จึงต้องใช้ตัวแปร EOF แยกกันสองตัว (`WS-U-EOF` และ `WS-S-EOF`)
แทนที่จะใช้ตัวแปรเดียวร่วมกันเหมือนตัวอย่างส่วนใหญ่ก่อนหน้านี้ในหลักสูตร

**เฉลย**: เพราะโปรแกรมนี้ทำงานกับ**สองแหล่งข้อมูลที่แยกจากกันโดยสิ้นเชิง**ในเวลาที่อาจจะยังไม่จบ
พร้อมกัน: `UNSORTED-FILE` (ควบคุมด้วย `WS-U-EOF` ใน `FILTER-ZERO-AMOUNTS`) และ Sort Work File
(ควบคุมด้วย `WS-S-EOF` ทั้งใน `BUILD-NUMBERED-REPORT` และใน `MAIN-PARA`) หากใช้ตัวแปรเดียวร่วมกัน
ทั้งหมด การอ่านจบของไฟล์หนึ่งอาจไปรบกวนสถานะการอ่านของอีกไฟล์หนึ่งโดยไม่ตั้งใจ (เหมือนบั๊กที่พบใน
ขั้นตอนที่ 263) การแยกตัวแปรตามไฟล์ที่รับผิดชอบอย่างชัดเจนจึงเป็นแนวปฏิบัติที่ปลอดภัยกว่ามากเมื่อ
โปรแกรมมีความซับซ้อนขึ้น

---

## ขั้นตอนที่ 266: เชื่อมต่อ SORT เข้ากับอัลกอริทึม Master-Detail จาก Part 026

### ปิดช่องว่างที่ Part 026 ทิ้งไว้

ถึงเวลาพิสูจน์ว่าทุกอย่างที่เรียนมาเชื่อมโยงกัน จำได้ไหมว่า Part 026 ขั้นตอนที่ 252 แสดงให้เห็นว่า
อัลกอริทึม Master-Detail Matching จะพังทันทีถ้าไฟล์ไม่เรียงลำดับ ตอนนี้เรามี `SORT` แล้ว จึงสามารถ
**เรียงลำดับไฟล์ Detail ที่ยังไม่เรียงให้เรียบร้อยก่อน** แล้วค่อยส่งต่อเข้าสู่อัลกอริทึมจับคู่ไฟล์
ที่สร้างไว้แล้วได้ทันที

### ตัวอย่าง: SORT ไฟล์ Transaction ที่ไม่เรียงลำดับ แล้วจับคู่กับ Master

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP266-SORT-THEN-MATCH.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER266.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT RAW-DETAIL-FILE ASSIGN TO "RAWDETAIL266.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT DETAIL-FILE ASSIGN TO "DETAIL266.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT SORT-WORK-FILE ASSIGN TO "SORTWORK266.DAT".

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MST-RECORD.
           05  MST-ID                  PIC X(4).
           05  MST-NAME                PIC X(15).

       FD  RAW-DETAIL-FILE.
       01  RAW-RECORD.
           05  RAW-ID                  PIC X(4).
           05  RAW-AMOUNT              PIC 9(5).

       FD  DETAIL-FILE.
       01  DTL-RECORD.
           05  DTL-ID                  PIC X(4).
           05  DTL-AMOUNT              PIC 9(5).

       SD  SORT-WORK-FILE.
       01  SORT-RECORD.
           05  SORT-ID                 PIC X(4).
           05  SORT-AMOUNT             PIC 9(5).

       WORKING-STORAGE SECTION.
       01  WS-M-EOF                    PIC X VALUE "N".
           88  MASTER-EOF               VALUE "Y".
       01  WS-D-EOF                    PIC X VALUE "N".
           88  DETAIL-EOF               VALUE "Y".
       01  WS-MASTER-KEY                PIC X(4).
       01  WS-DETAIL-KEY                PIC X(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT MASTER-FILE.
           MOVE "A001" TO MST-ID.
           MOVE "SOMCHAI JAIDEE " TO MST-NAME.
           WRITE MST-RECORD.
           MOVE "A002" TO MST-ID.
           MOVE "SUDA MEECHAI   " TO MST-NAME.
           WRITE MST-RECORD.
           MOVE "A003" TO MST-ID.
           MOVE "PRASERT KAEWTA " TO MST-NAME.
           WRITE MST-RECORD.
           CLOSE MASTER-FILE.

      *> RAW-DETAIL-FILE simulates a real-world feed: transactions
      *> arrived out of order, exactly like step 252's problem.
           OPEN OUTPUT RAW-DETAIL-FILE.
           MOVE "A003" TO RAW-ID.
           MOVE 150 TO RAW-AMOUNT.
           WRITE RAW-RECORD.
           MOVE "A001" TO RAW-ID.
           MOVE 300 TO RAW-AMOUNT.
           WRITE RAW-RECORD.
           MOVE "A002" TO RAW-ID.
           MOVE 500 TO RAW-AMOUNT.
           WRITE RAW-RECORD.
           CLOSE RAW-DETAIL-FILE.

      *> Step 1: SORT the raw feed into DETAIL-FILE first - this
      *> satisfies the "both files sorted" requirement from
      *> Part 026, step 252.
           SORT SORT-WORK-FILE
               ON ASCENDING KEY SORT-ID
               USING RAW-DETAIL-FILE
               GIVING DETAIL-FILE.
           DISPLAY "Step 1: DETAIL-FILE sorted by ID.".

      *> Step 2: run the balanced-line matching algorithm from
      *> Part 026 against the now-sorted DETAIL-FILE.
           OPEN INPUT MASTER-FILE.
           OPEN INPUT DETAIL-FILE.
           PERFORM READ-MASTER.
           PERFORM READ-DETAIL.
           DISPLAY "Step 2: matching against MASTER-FILE:".
           PERFORM UNTIL MASTER-EOF
               IF WS-MASTER-KEY = WS-DETAIL-KEY
                   DISPLAY "  " MST-ID " " MST-NAME
                       " -- amount " DTL-AMOUNT
               ELSE
                   DISPLAY "  " MST-ID " " MST-NAME
                       " -- no transaction"
               END-IF
               PERFORM READ-MASTER
               PERFORM READ-DETAIL
           END-PERFORM.
           CLOSE MASTER-FILE.
           CLOSE DETAIL-FILE.
           STOP RUN.

       READ-MASTER.
           READ MASTER-FILE
               AT END
                   SET MASTER-EOF TO TRUE
                   MOVE HIGH-VALUES TO WS-MASTER-KEY
               NOT AT END
                   MOVE MST-ID TO WS-MASTER-KEY
           END-READ.

       READ-DETAIL.
           READ DETAIL-FILE
               AT END
                   SET DETAIL-EOF TO TRUE
                   MOVE HIGH-VALUES TO WS-DETAIL-KEY
               NOT AT END
                   MOVE DTL-ID TO WS-DETAIL-KEY
           END-READ.
```

**ผลลัพธ์:**

```
Step 1: DETAIL-FILE sorted by ID.
Step 2: matching against MASTER-FILE:
  A001 SOMCHAI JAIDEE  -- amount 00300
  A002 SUDA MEECHAI    -- amount 00500
  A003 PRASERT KAEWTA  -- amount 00150
```

### อธิบายจุดสำคัญ

- `RAW-DETAIL-FILE` เขียนด้วยลำดับ A003, A001, A002 (ไม่เรียงลำดับ) เหมือนกับปัญหาที่พบใน
  Part 026 ขั้นตอนที่ 252 เป๊ะ ๆ
- ขั้นตอนที่ 1 ใช้ `SORT ... USING RAW-DETAIL-FILE GIVING DETAIL-FILE` เพื่อสร้าง `DETAIL-FILE`
  ที่เรียงลำดับถูกต้องแล้ว โดยไม่ต้องแก้ไขอัลกอริทึมการจับคู่ในขั้นตอนที่ 2 เลยแม้แต่นิดเดียว
- ขั้นตอนที่ 2 คือโค้ดจับคู่แบบเดียวกับ Part 026 ขั้นตอนที่ 255 ทุกประการ (Priming Read +
  HIGH-VALUES Sentinel) แสดงให้เห็นว่า**เมื่อไฟล์เรียงลำดับถูกต้องแล้ว อัลกอริทึมเดิมทำงานได้ถูก
  ต้องทันที** — นี่คือเหตุผลที่ Part 026 กับ Part 027 ถูกออกแบบให้ต่อเนื่องกัน

### ข้อควรระวัง

- ในระบบจริง ขั้นตอน `SORT` และขั้นตอนการจับคู่มักแยกเป็นคนละโปรแกรม (หรือคนละ Step ใน JCL บน
  Mainframe ที่จะเรียนใน Part 052) เพื่อให้แต่ละโปรแกรมมีหน้าที่ชัดเจนตามหลัก Modularization
  แต่การรวมไว้ในโปรแกรมเดียวเพื่อการเรียนรู้ก็ยังคงถูกต้องตามหลักการเดียวกัน
- อย่าลืมว่า `DETAIL-FILE` (ผลลัพธ์จาก `SORT`) ต้องมีโครงสร้าง record ที่ตรงกับที่อัลกอริทึมการ
  จับคู่คาดหวังไว้ (`DTL-ID`, `DTL-AMOUNT`) — `SORT` แค่เรียงลำดับ ไม่ได้เปลี่ยนโครงสร้างข้อมูลใด ๆ

### แบบฝึกหัดที่ 266.1

**โจทย์**: จงอธิบายว่าทำไมการแยก `SORT` กับอัลกอริทึมการจับคู่ออกเป็นสองขั้นตอนที่ชัดเจน (Step 1
และ Step 2 ในโปรแกรมนี้) จึงเป็นแนวทางที่ดีกว่าการพยายามเรียงลำดับและจับคู่ไปพร้อมกันในลูปเดียว

**เฉลย**: เพราะการแยกสองขั้นตอนออกจากกันทำให้แต่ละส่วนมีความรับผิดชอบเดียวที่ชัดเจน (Single
Responsibility) — `SORT` มีหน้าที่แค่เรียงลำดับ ส่วนอัลกอริทึมการจับคู่มีหน้าที่แค่จับคู่ข้อมูลที่
เรียงลำดับแล้ว ทำให้ง่ายต่อการทดสอบและแก้ไขแต่ละส่วนแยกจากกัน (เช่น ถ้าผลลัพธ์ผิดพลาด สามารถตรวจสอบ
ได้ทันทีว่าปัญหาอยู่ที่ขั้นตอนการเรียงลำดับหรือขั้นตอนการจับคู่) นอกจากนี้อัลกอริทึมการจับคู่ที่เขียน
ไว้ตั้งแต่ Part 026 ยังใช้ซ้ำได้ทันทีโดยไม่ต้องแก้ไขอะไรเลย ตราบใดที่ข้อมูลที่ป้อนเข้ามาเรียงลำดับ
ถูกต้องแล้ว

---

## ขั้นตอนที่ 267: MERGE Statement พื้นฐาน — รวมไฟล์ที่เรียงลำดับแล้ว

### MERGE ต่างจาก SORT อย่างไร

**`MERGE`** มีหน้าตาคล้าย `SORT` มาก (ใช้ `SD`, `ON ASCENDING/DESCENDING KEY`, `USING`, `GIVING`
เหมือนกันทุกประการ) แต่มีข้อแตกต่างสำคัญ 1 ข้อ: **`MERGE` คาดหวังว่าไฟล์ต้นทางทุกไฟล์เรียงลำดับตาม
Key นั้นอยู่แล้ว** หน้าที่ของมันคือ**รวม**ไฟล์ที่เรียงลำดับแล้วหลายไฟล์เข้าเป็นไฟล์เดียวที่ยังคง
เรียงลำดับอยู่ ไม่ใช่การเรียงลำดับใหม่ตั้งแต่ต้นเหมือน `SORT`

### ตัวอย่าง: รวมรายชื่อลูกค้าจาก 2 สาขาที่เรียงลำดับแล้ว

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP267-MERGE-BASICS.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT BRANCH-A-FILE ASSIGN TO "BRANCHA267.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT BRANCH-B-FILE ASSIGN TO "BRANCHB267.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT COMBINED-FILE ASSIGN TO "COMBINED267.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT MERGE-WORK-FILE ASSIGN TO "MERGEWORK267.DAT".

       DATA DIVISION.
       FILE SECTION.
       FD  BRANCH-A-FILE.
       01  A-RECORD.
           05  A-ID                    PIC X(4).
           05  A-NAME                  PIC X(15).

       FD  BRANCH-B-FILE.
       01  B-RECORD.
           05  B-ID                    PIC X(4).
           05  B-NAME                  PIC X(15).

       FD  COMBINED-FILE.
       01  C-RECORD.
           05  C-ID                    PIC X(4).
           05  C-NAME                  PIC X(15).

       SD  MERGE-WORK-FILE.
       01  MERGE-RECORD.
           05  MERGE-ID                PIC X(4).
           05  MERGE-NAME              PIC X(15).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Both input files are ALREADY sorted by ID - this is the
      *> key difference from SORT, which accepts unsorted input.
           OPEN OUTPUT BRANCH-A-FILE.
           MOVE "A001" TO A-ID.
           MOVE "SOMCHAI JAIDEE " TO A-NAME.
           WRITE A-RECORD.
           MOVE "A003" TO A-ID.
           MOVE "PRASERT KAEWTA " TO A-NAME.
           WRITE A-RECORD.
           CLOSE BRANCH-A-FILE.

           OPEN OUTPUT BRANCH-B-FILE.
           MOVE "A002" TO B-ID.
           MOVE "SUDA MEECHAI   " TO B-NAME.
           WRITE B-RECORD.
           MOVE "A004" TO B-ID.
           MOVE "WANNA SRISUK   " TO B-NAME.
           WRITE B-RECORD.
           CLOSE BRANCH-B-FILE.

      *> MERGE combines two (or more) ALREADY-sorted files into
      *> one, keeping the overall order - much cheaper than a full
      *> SORT because it never has to reorder anything, only pick
      *> the smaller of the two current keys at each step.
           MERGE MERGE-WORK-FILE
               ON ASCENDING KEY MERGE-ID
               USING BRANCH-A-FILE BRANCH-B-FILE
               GIVING COMBINED-FILE.

           OPEN INPUT COMBINED-FILE.
           DISPLAY "Combined customer list from both branches:".
           PERFORM 4 TIMES
               READ COMBINED-FILE
               DISPLAY "  " C-ID " " C-NAME
           END-PERFORM.
           CLOSE COMBINED-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Combined customer list from both branches:
  A001 SOMCHAI JAIDEE
  A002 SUDA MEECHAI
  A003 PRASERT KAEWTA
  A004 WANNA SRISUK
```

### อธิบายจุดสำคัญ

- `BRANCH-A-FILE` มี A001, A003 (เรียงแล้ว) และ `BRANCH-B-FILE` มี A002, A004 (เรียงแล้วเช่นกัน)
  — `MERGE` แค่ "สอด" record จากทั้งสองไฟล์เข้าด้วยกันตามลำดับ Key โดยไม่ต้องเรียงใหม่ทั้งหมด
- ไวยากรณ์ `MERGE work-file ON ASCENDING KEY key USING file1 file2 GIVING output-file` เหมือนกับ
  `SORT` ทุกประการ เพียงแค่เปลี่ยนคำสั่งจาก `SORT` เป็น `MERGE`
- แนวคิดเชิงประสิทธิภาพ: อัลกอริทึม Merge แท้จริงในทางทฤษฎีคอมพิวเตอร์มีความซับซ้อนแค่ O(n) (อ่าน
  ทุก record จากทุกไฟล์แค่ครั้งเดียว เทียบกันแล้วเลือกตัวที่เล็กกว่าไปเรื่อย ๆ) ในขณะที่ `SORT`
  ทั่วไปมีความซับซ้อน O(n log n) — เมื่อรู้อยู่แล้วว่าข้อมูลเรียงลำดับมาแล้ว การใช้ `MERGE` จึง
  มีประสิทธิภาพเหนือกว่า `SORT` ในทางทฤษฎีเสมอ

### ข้อควรระวัง

- **`MERGE` ไม่รองรับ `INPUT PROCEDURE`** (เพราะไม่มีประโยชน์ที่จะกรองข้อมูลก่อนรวม ในเมื่อข้อมูล
  ควรจะเรียงลำดับมาสมบูรณ์แล้ว) แต่ยังรองรับ `OUTPUT PROCEDURE` ได้เหมือน `SORT` สำหรับประมวลผล
  ข้อมูลหลังรวมเสร็จ (เช่น คำนวณยอดสะสมข้ามหลายสาขา)
- ต้องแน่ใจว่าทุกไฟล์ใน `USING` เรียงลำดับตาม Key เดียวกันจริง ๆ ก่อนเรียก `MERGE` เสมอ — ขั้นตอน
  ถัดไปจะแสดงให้เห็นความเสี่ยงเมื่อสมมติฐานนี้ไม่เป็นจริง

### แบบฝึกหัดที่ 267.1

**โจทย์**: จงอธิบายว่าทำไมการรวมรายชื่อลูกค้าจากหลายสาขาด้วย `MERGE` (เมื่อแต่ละสาขาเรียงลำดับ
ข้อมูลของตัวเองไว้แล้ว) จึงมีประสิทธิภาพดีกว่าการนำไฟล์ทั้งหมดมารวมกันเป็นไฟล์เดียวก่อน แล้วค่อยใช้
`SORT` เรียงลำดับใหม่ทั้งหมด

**เฉลย**: เพราะ `SORT` ต้องเปรียบเทียบและจัดเรียง record ทั้งหมดใหม่ตั้งแต่ต้น (ความซับซ้อนระดับ
O(n log n)) โดยไม่ได้ใช้ประโยชน์จากข้อเท็จจริงที่ว่าแต่ละไฟล์ย่อยเรียงลำดับอยู่แล้ว ในขณะที่
`MERGE` ใช้ประโยชน์จากข้อเท็จจริงนี้เต็มที่ โดยเปรียบเทียบแค่ record ปัจจุบันของแต่ละไฟล์ (ไม่ใช่
ทุก record กับทุก record) แล้วเลือกตัวที่มี Key น้อยที่สุดออกมาก่อนเสมอ (ความซับซ้อนระดับ O(n))
เมื่อจำนวนไฟล์และจำนวน record มีขนาดใหญ่มากในระบบจริง (เช่น รวมข้อมูลจากสาขาหลายร้อยแห่ง) ความ
แตกต่างด้านประสิทธิภาพนี้มีนัยสำคัญมาก

---

## ขั้นตอนที่ 268: MERGE หลายไฟล์พร้อมกัน (มากกว่า 2 ไฟล์)

### MERGE ไม่จำกัดแค่ 2 ไฟล์

`MERGE` รองรับการรวมไฟล์**มากกว่า 2 ไฟล์พร้อมกัน**ได้ในคำสั่งเดียว เพียงแค่ระบุชื่อไฟล์ทั้งหมดต่อ
กันหลัง `USING`

### ตัวอย่าง: รวมข้อมูลจาก 3 สาขาพร้อมกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP268-MERGE-THREE-FILES.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT BRANCH-A-FILE ASSIGN TO "BRANCHA268.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT BRANCH-B-FILE ASSIGN TO "BRANCHB268.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT BRANCH-C-FILE ASSIGN TO "BRANCHC268.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT COMBINED-FILE ASSIGN TO "COMBINED268.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT MERGE-WORK-FILE ASSIGN TO "MERGEWORK268.DAT".

       DATA DIVISION.
       FILE SECTION.
       FD  BRANCH-A-FILE.
       01  A-RECORD                    PIC X(4).

       FD  BRANCH-B-FILE.
       01  B-RECORD                    PIC X(4).

       FD  BRANCH-C-FILE.
       01  C-RECORD                    PIC X(4).

       FD  COMBINED-FILE.
       01  COMBINED-RECORD             PIC X(4).

       SD  MERGE-WORK-FILE.
       01  MERGE-RECORD                PIC X(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Three branches, each already sorted by customer ID.
           OPEN OUTPUT BRANCH-A-FILE.
           MOVE "A001" TO A-RECORD.
           WRITE A-RECORD.
           MOVE "A004" TO A-RECORD.
           WRITE A-RECORD.
           CLOSE BRANCH-A-FILE.

           OPEN OUTPUT BRANCH-B-FILE.
           MOVE "A002" TO B-RECORD.
           WRITE B-RECORD.
           MOVE "A005" TO B-RECORD.
           WRITE B-RECORD.
           CLOSE BRANCH-B-FILE.

           OPEN OUTPUT BRANCH-C-FILE.
           MOVE "A003" TO C-RECORD.
           WRITE C-RECORD.
           MOVE "A006" TO C-RECORD.
           WRITE C-RECORD.
           CLOSE BRANCH-C-FILE.

      *> MERGE accepts more than 2 files at once - USING can list
      *> as many already-sorted files as needed.
           MERGE MERGE-WORK-FILE
               ON ASCENDING KEY MERGE-RECORD
               USING BRANCH-A-FILE BRANCH-B-FILE BRANCH-C-FILE
               GIVING COMBINED-FILE.

           OPEN INPUT COMBINED-FILE.
           DISPLAY "All 3 branches merged into one ID list:".
           PERFORM 6 TIMES
               READ COMBINED-FILE
               DISPLAY "  " COMBINED-RECORD
           END-PERFORM.
           CLOSE COMBINED-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
All 3 branches merged into one ID list:
  A001
  A002
  A003
  A004
  A005
  A006
```

### อธิบายจุดสำคัญ

- `USING BRANCH-A-FILE BRANCH-B-FILE BRANCH-C-FILE` ระบุไฟล์ต้นทางทั้ง 3 ไฟล์ไว้ในบรรทัดเดียว
  คั่นด้วยช่องว่าง (ไม่ต้องมีเครื่องหมายจุลภาค) — `MERGE` จะรวมทั้ง 3 ไฟล์เป็นไฟล์เดียวที่เรียง
  ลำดับสมบูรณ์ในคำสั่งเดียว
- แต่ละสาขามีข้อมูลสลับกัน (A001,A004 / A002,A005 / A003,A006) แต่ผลลัพธ์สุดท้ายเรียงลำดับ
  ต่อเนื่องกันสมบูรณ์ (A001 ถึง A006) แสดงให้เห็นว่า `MERGE` จัดการไฟล์หลายไฟล์พร้อมกันได้อย่างถูก
  ต้อง ไม่ใช่แค่ 2 ไฟล์

### ข้อควรระวัง

- ยิ่งมีไฟล์มากขึ้น ยิ่งต้องมั่นใจมากขึ้นว่า**ทุกไฟล์**เรียงลำดับตาม Key เดียวกันจริง เพราะไฟล์ใด
  ไฟล์หนึ่งที่ไม่เรียงลำดับจะกระทบผลลัพธ์รวมทั้งหมด (รายละเอียดเต็มในขั้นตอนถัดไป)
- ไม่มีข้อจำกัดตายตัวว่า `MERGE` รวมได้สูงสุดกี่ไฟล์ในทางทฤษฎี แต่ในทางปฏิบัติจำนวนไฟล์ที่มากเกินไป
  อาจทำให้โค้ดอ่านยากและจัดการยาก ควรพิจารณาแบ่งเป็นหลายขั้นตอนหากมีไฟล์จำนวนมากจริง ๆ

### แบบฝึกหัดที่ 268.1

**โจทย์**: จงอธิบายว่าทำไม `MERGE` จึงไม่มีข้อจำกัดเรื่องต้องเป็น 2 ไฟล์เท่านั้น ในขณะที่การจับคู่
Master-Detail จาก Part 026 ที่เราสร้างไว้ (ซึ่งใช้ตัวแปร `WS-MASTER-KEY`/`WS-DETAIL-KEY` เปรียบ
เทียบกัน 2 ค่า) ถูกออกแบบมาสำหรับ 2 ไฟล์เป็นหลัก

**เฉลย**: เพราะ `MERGE` เป็นคำสั่งสำเร็จรูปที่ compiler จัดการตรรกะการเปรียบเทียบ Key ของทุกไฟล์
ให้เองภายในเครื่องยนต์ของมันเอง (ไม่ว่าจะมี 2, 3, หรือ 10 ไฟล์ก็ตาม) โปรแกรมเมอร์เพียงระบุรายชื่อ
ไฟล์เท่านั้น ในขณะที่อัลกอริทึม Master-Detail ที่เราเขียนเองด้วยมือใน Part 026 ใช้ตัวแปรเปรียบเทียบ
Key ที่เขียนขึ้นมาเองโดยเฉพาะสำหรับ 2 แหล่งข้อมูล (Master กับ Detail) การจะขยายให้รองรับ 3 แหล่ง
ข้อมูลขึ้นไปด้วยมือจะต้องเขียนตรรกะเปรียบเทียบที่ซับซ้อนขึ้นมาก (ต้องหาค่าต่ำสุดจากหลายค่าพร้อมกัน)
ซึ่งเป็นเหตุผลหนึ่งที่ควรใช้ `MERGE` ของ COBOL เองแทนการเขียนอัลกอริทึมเทียบเท่าด้วยมือ เมื่อโจทย์
เป็นการรวมไฟล์ที่เรียงลำดับแล้วล้วน ๆ โดยไม่มีตรรกะทางธุรกิจอื่นแทรกอยู่

---

## ขั้นตอนที่ 269: ความเสี่ยงของ MERGE เมื่อข้อมูลไม่เรียงลำดับจริง

### การทดลอง: MERGE เมื่อไฟล์หนึ่งไม่เรียงลำดับ

มาตรฐาน COBOL ระบุไว้ชัดเจนว่า **ถ้าไฟล์ที่ป้อนเข้า `MERGE` ไม่เรียงลำดับตาม Key จริง ผลลัพธ์ที่ได้
ถือเป็น "ไม่นิยาม" (Undefined Behavior)** — ลองดูสิ่งที่เกิดขึ้นจริงเมื่อทดสอบกับ GnuCOBOL:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP269-MERGE-UNSORTED-RISK.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT BRANCH-A-FILE ASSIGN TO "BRANCHA269.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT BRANCH-B-FILE ASSIGN TO "BRANCHB269.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT COMBINED-FILE ASSIGN TO "COMBINED269.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT MERGE-WORK-FILE ASSIGN TO "MERGEWORK269.DAT".

       DATA DIVISION.
       FILE SECTION.
       FD  BRANCH-A-FILE.
       01  A-RECORD                    PIC X(4).

       FD  BRANCH-B-FILE.
       01  B-RECORD                    PIC X(4).

       FD  COMBINED-FILE.
       01  COMBINED-RECORD             PIC X(4).

       SD  MERGE-WORK-FILE.
       01  MERGE-RECORD                PIC X(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> BRANCH-A-FILE is deliberately NOT sorted: A004 appears
      *> before A001. The COBOL standard leaves MERGE's result
      *> UNDEFINED when an input is not already sorted.
           OPEN OUTPUT BRANCH-A-FILE.
           MOVE "A004" TO A-RECORD.
           WRITE A-RECORD.
           MOVE "A001" TO A-RECORD.
           WRITE A-RECORD.
           CLOSE BRANCH-A-FILE.

           OPEN OUTPUT BRANCH-B-FILE.
           MOVE "A002" TO B-RECORD.
           WRITE B-RECORD.
           MOVE "A005" TO B-RECORD.
           WRITE B-RECORD.
           CLOSE BRANCH-B-FILE.

           MERGE MERGE-WORK-FILE
               ON ASCENDING KEY MERGE-RECORD
               USING BRANCH-A-FILE BRANCH-B-FILE
               GIVING COMBINED-FILE.

           OPEN INPUT COMBINED-FILE.
           DISPLAY "Result even though BRANCH-A-FILE was unsorted:".
           PERFORM 4 TIMES
               READ COMBINED-FILE
               DISPLAY "  " COMBINED-RECORD
           END-PERFORM.
           CLOSE COMBINED-FILE.
           STOP RUN.
```

**ผลลัพธ์ที่ทดสอบจริงด้วย GnuCOBOL (น่าประหลาดใจ):**

```
Result even though BRANCH-A-FILE was unsorted:
  A001
  A002
  A004
  A005
```

### วิเคราะห์: ทำไมผลลัพธ์ถึง "ถูกต้อง" ทั้งที่ป้อนข้อมูลไม่เรียงลำดับ

ผลลัพธ์ที่ได้จาก GnuCOBOL **ถูกต้องสมบูรณ์**แม้ `BRANCH-A-FILE` จะไม่เรียงลำดับ (A004 มาก่อน A001)
เพราะเหตุผลเชิงเทคนิคคือ **การ implement `MERGE` ของ GnuCOBOL เวอร์ชันนี้ในทางปฏิบัติทำการเรียง
ลำดับข้อมูลทั้งหมดใหม่จริง ๆ เบื้องหลัง** (คล้ายกับพฤติกรรมของ `SORT`) แทนที่จะทำ true k-way merge
ตามทฤษฎีที่อาศัยสมมติฐานว่าข้อมูลเรียงมาแล้ว

**นี่คือจุดที่ต้องระวังอย่างยิ่ง**: แม้ผลลัพธ์จาก GnuCOBOL จะดู "ถูกต้อง" ในการทดสอบนี้ **มาตรฐาน
COBOL ไม่ได้รับประกันพฤติกรรมนี้เลย** compiler ตัวอื่น (เช่น IBM Enterprise COBOL บน Mainframe จริง
ที่ใช้ true merge algorithm เพื่อประสิทธิภาพสูงสุดตามที่ออกแบบไว้) **อาจให้ผลลัพธ์ที่ผิดพลาดแบบ
เงียบ ๆ** เมื่อป้อนไฟล์ที่ไม่เรียงลำดับจริง เพราะมันจะเชื่อสมมติฐานว่าไฟล์เรียงมาแล้วและไม่เสียเวลา
ตรวจสอบซ้ำ

### กฎที่ต้องจำ

> **ห้ามพึ่งพาพฤติกรรม "ใจดี" ของ GnuCOBOL ที่ยอมรับไฟล์ไม่เรียงลำดับใน `MERGE` เด็ดขาด** ต้องมั่นใจ
> เสมอว่าทุกไฟล์ที่ป้อนเข้า `MERGE` เรียงลำดับตาม Key จริงมาก่อนแล้ว (โดยใช้ `SORT` ล่วงหน้าถ้าจำเป็น)
> เพื่อให้โค้ดพกพาข้าม compiler และสภาพแวดล้อม Mainframe จริงได้อย่างปลอดภัย นี่คือเหตุผลสำคัญที่
> ควรใช้ `SORT` เมื่อไม่แน่ใจว่าข้อมูลเรียงลำดับหรือไม่ และสงวน `MERGE` ไว้เฉพาะกรณีที่**มั่นใจ
> จริง ๆ**ว่าทุกไฟล์ต้นทางเรียงลำดับมาแล้วเท่านั้น (เช่น ไฟล์ที่เพิ่งผ่าน `SORT` มาในขั้นตอนก่อนหน้า
> ของ Batch Job เดียวกัน)

### ข้อควรระวัง

- ความแตกต่างระหว่าง `SORT` กับ `MERGE` ไม่ได้อยู่ที่ "ผลลัพธ์ที่ได้" (ซึ่งอาจดูเหมือนกันในการ
  ทดสอบเล็ก ๆ) แต่อยู่ที่ **"สัญญา" (Contract) ที่แต่ละคำสั่งให้ไว้กับ compiler**: `SORT` สัญญาว่า
  จะเรียงให้ไม่ว่าข้อมูลจะเป็นอย่างไร ส่วน `MERGE` สัญญาว่าจะรวมให้อย่างมีประสิทธิภาพ**โดยเชื่อใจ**
  ว่าข้อมูลเรียงมาแล้ว การผิดสัญญานี้ (ป้อนข้อมูลไม่เรียงให้ `MERGE`) คือบั๊กเชิงตรรกะที่ผู้เขียน
  โปรแกรมต้องรับผิดชอบเอง ไม่ใช่สิ่งที่ compiler ต้องตรวจจับให้
- เมื่อไม่แน่ใจว่าไฟล์เรียงลำดับหรือไม่ ให้เลือกใช้ `SORT` ไว้ก่อนเสมอ (ปลอดภัยกว่าแต่ช้ากว่า)
  แล้วค่อยพิจารณาเปลี่ยนไปใช้ `MERGE` ภายหลังเมื่อมั่นใจแล้วว่าข้อมูลต้นทางเรียงลำดับจริง และ
  ต้องการเพิ่มประสิทธิภาพของระบบ

### แบบฝึกหัดที่ 269.1

**โจทย์**: จงอธิบายว่าทำไมการทดสอบเพียงครั้งเดียวที่ให้ผลลัพธ์ "ถูกต้อง" (เหมือนในขั้นตอนนี้) จึง
**ไม่เพียงพอ**ที่จะสรุปว่าการป้อนไฟล์ไม่เรียงลำดับให้ `MERGE` เป็นวิธีที่ปลอดภัยสำหรับใช้งานจริง

**เฉลย**: เพราะพฤติกรรมที่สังเกตได้จากการทดสอบเพียงครั้งเดียวบน compiler ตัวเดียว (GnuCOBOL เวอร์ชัน
นี้) เป็นเพียง**รายละเอียดการ implement ภายใน**ของ compiler นั้น ๆ ไม่ใช่สิ่งที่มาตรฐาน COBOL
รับประกันไว้ หากวันหนึ่งเปลี่ยนไปใช้ compiler ตัวอื่น (เช่น ย้ายไปรันบน Mainframe จริงด้วย IBM
Enterprise COBOL ตามที่จะเรียนในเฟส 4) หรือแม้แต่ GnuCOBOL เวอร์ชันใหม่กว่าที่ปรับปรุงการ
implement `MERGE` ให้เป็น true k-way merge เพื่อประสิทธิภาพที่ดีขึ้น โปรแกรมเดียวกันนี้อาจให้ผล
ลัพธ์ที่ผิดพลาดทันทีโดยไม่มีการเตือนใด ๆ เลย การเขียนโค้ดที่ถูกต้องตามสัญญาของมาตรฐานภาษาเสมอ (ไม่
ใช่แค่พฤติกรรมที่สังเกตได้ของ compiler ตัวใดตัวหนึ่ง) จึงเป็นหลักการสำคัญของการเขียนโปรแกรมที่มี
คุณภาพและพกพาข้ามระบบได้จริง

---

## ขั้นตอนที่ 270: โปรแกรมรวมสุดท้าย — ไปป์ไลน์ SORT + MERGE + Master-Detail แบบสมบูรณ์

### โจทย์: งาน Batch ที่รวมข้อมูลจากหลายสาขาแล้วอัปเดตบัญชีกลาง

ขั้นตอนสุดท้ายของ Part นี้จะจำลองสถานการณ์จริงที่ใกล้เคียงธนาคารขนาดใหญ่ที่สุด: มีสาขา 2 แห่งส่ง
ไฟล์ธุรกรรมมาให้ (แต่ละสาขาเรียงลำดับข้อมูลของตัวเองเรียบร้อยแล้ว) เราต้อง **`MERGE`** รวมทั้งสอง
ไฟล์เป็นไฟล์เดียว แล้วนำไปป้อนเข้าอัลกอริทึม **Master-Detail Matching** จาก Part 026 เพื่ออัปเดต
ยอดเงินจริงด้วย `REWRITE` พร้อมรายงานข้อผิดพลาดแยกไฟล์

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP270-FULL-PIPELINE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER270.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT BRANCH-A-FILE ASSIGN TO "BRANCHA270.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT BRANCH-B-FILE ASSIGN TO "BRANCHB270.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT DETAIL-FILE ASSIGN TO "DETAIL270.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT EXCEPTION-FILE ASSIGN TO "EXCEPTION270.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT MERGE-WORK-FILE ASSIGN TO "MERGEWORK270.DAT".

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MST-RECORD.
           05  MST-ID                  PIC X(4).
           05  MST-NAME                PIC X(15).
           05  MST-BALANCE             PIC 9(7)V99.

       FD  BRANCH-A-FILE.
       01  BA-RECORD.
           05  BA-ID                   PIC X(4).
           05  BA-AMOUNT               PIC 9(5)V99.

       FD  BRANCH-B-FILE.
       01  BB-RECORD.
           05  BB-ID                   PIC X(4).
           05  BB-AMOUNT               PIC 9(5)V99.

       FD  DETAIL-FILE.
       01  DTL-RECORD.
           05  DTL-ID                  PIC X(4).
           05  DTL-AMOUNT              PIC 9(5)V99.

       FD  EXCEPTION-FILE.
       01  EXC-RECORD                  PIC X(40).

       SD  MERGE-WORK-FILE.
       01  MERGE-RECORD.
           05  MERGE-ID                PIC X(4).
           05  MERGE-AMOUNT            PIC 9(5)V99.

       WORKING-STORAGE SECTION.
       01  WS-M-EOF                    PIC X VALUE "N".
           88  MASTER-EOF               VALUE "Y".
       01  WS-D-EOF                    PIC X VALUE "N".
           88  DETAIL-EOF               VALUE "Y".
       01  WS-MASTER-KEY                PIC X(4).
       01  WS-DETAIL-KEY                PIC X(4).
       01  WS-DEPOSIT-TOTAL             PIC 9(6)V99 VALUE 0.
       01  WS-MATCHED-COUNT             PIC 9(3) VALUE 0.
       01  WS-ORPHAN-COUNT              PIC 9(3) VALUE 0.
       01  WS-DISPLAY-AMOUNT            PIC ZZZ9.99.
       01  WS-DISPLAY-BAL               PIC ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM CREATE-TEST-DATA.
           PERFORM MERGE-BRANCH-FEEDS.
           PERFORM RUN-BATCH-UPDATE.
           PERFORM PRINT-FINAL-RESULTS.
           STOP RUN.

       CREATE-TEST-DATA.
           OPEN OUTPUT MASTER-FILE.
           MOVE "A001" TO MST-ID.
           MOVE "SOMCHAI JAIDEE " TO MST-NAME.
           MOVE 1000.00 TO MST-BALANCE.
           WRITE MST-RECORD.
           MOVE "A002" TO MST-ID.
           MOVE "SUDA MEECHAI   " TO MST-NAME.
           MOVE 2000.00 TO MST-BALANCE.
           WRITE MST-RECORD.
           MOVE "A003" TO MST-ID.
           MOVE "PRASERT KAEWTA " TO MST-NAME.
           MOVE 300.00 TO MST-BALANCE.
           WRITE MST-RECORD.
           CLOSE MASTER-FILE.

      *> Branch A: already sorted internally (its own ATM feed).
           OPEN OUTPUT BRANCH-A-FILE.
           MOVE "A001" TO BA-ID.
           MOVE 100.00 TO BA-AMOUNT.
           WRITE BA-RECORD.
           MOVE "A003" TO BA-ID.
           MOVE 75.25 TO BA-AMOUNT.
           WRITE BA-RECORD.
           CLOSE BRANCH-A-FILE.

      *> Branch B: also sorted internally, but includes a stray
      *> transaction (A099) for an account that does not exist.
           OPEN OUTPUT BRANCH-B-FILE.
           MOVE "A001" TO BB-ID.
           MOVE 250.50 TO BB-AMOUNT.
           WRITE BB-RECORD.
           MOVE "A099" TO BB-ID.
           MOVE 999.00 TO BB-AMOUNT.
           WRITE BB-RECORD.
           CLOSE BRANCH-B-FILE.

       MERGE-BRANCH-FEEDS.
      *> Step 1: MERGE the two already-sorted branch feeds into
      *> one combined, still-sorted DETAIL-FILE.
           MERGE MERGE-WORK-FILE
               ON ASCENDING KEY MERGE-ID
               USING BRANCH-A-FILE BRANCH-B-FILE
               GIVING DETAIL-FILE.
           DISPLAY "Step 1: branch feeds merged into DETAIL-FILE.".

       RUN-BATCH-UPDATE.
      *> Step 2: the Part 026 balanced-line matching algorithm,
      *> now fed by the merged, correctly-ordered DETAIL-FILE.
           OPEN I-O MASTER-FILE.
           OPEN INPUT DETAIL-FILE.
           OPEN OUTPUT EXCEPTION-FILE.
           PERFORM READ-MASTER.
           PERFORM READ-DETAIL.

           PERFORM UNTIL MASTER-EOF AND DETAIL-EOF
               EVALUATE TRUE
                   WHEN WS-MASTER-KEY = WS-DETAIL-KEY
                       PERFORM PROCESS-MATCHED-MASTER
                   WHEN WS-MASTER-KEY < WS-DETAIL-KEY
                       PERFORM READ-MASTER
                   WHEN OTHER
                       PERFORM LOG-ORPHAN-TRANSACTION
                       PERFORM READ-DETAIL
               END-EVALUATE
           END-PERFORM.

           CLOSE MASTER-FILE.
           CLOSE DETAIL-FILE.
           CLOSE EXCEPTION-FILE.
           DISPLAY "Step 2: batch update complete.".

       PROCESS-MATCHED-MASTER.
           ADD 1 TO WS-MATCHED-COUNT.
           MOVE 0 TO WS-DEPOSIT-TOTAL.
           PERFORM UNTIL WS-DETAIL-KEY NOT = WS-MASTER-KEY
               ADD DTL-AMOUNT TO WS-DEPOSIT-TOTAL
               PERFORM READ-DETAIL
           END-PERFORM.
           ADD WS-DEPOSIT-TOTAL TO MST-BALANCE.
           REWRITE MST-RECORD.
           PERFORM READ-MASTER.

       LOG-ORPHAN-TRANSACTION.
           ADD 1 TO WS-ORPHAN-COUNT.
           MOVE DTL-AMOUNT TO WS-DISPLAY-AMOUNT.
           MOVE SPACES TO EXC-RECORD.
           STRING "ORPHAN ID=" DTL-ID
               " AMOUNT=" WS-DISPLAY-AMOUNT
               DELIMITED BY SIZE
               INTO EXC-RECORD.
           WRITE EXC-RECORD.

       PRINT-FINAL-RESULTS.
           MOVE "N" TO WS-M-EOF.
           OPEN INPUT MASTER-FILE.
           DISPLAY "---- Final master balances ----".
           PERFORM UNTIL MASTER-EOF
               READ MASTER-FILE
                   AT END
                       SET MASTER-EOF TO TRUE
                   NOT AT END
                       MOVE MST-BALANCE TO WS-DISPLAY-BAL
                       DISPLAY MST-ID " " MST-NAME " "
                           WS-DISPLAY-BAL
               END-READ
           END-PERFORM.
           CLOSE MASTER-FILE.

           DISPLAY "---- Exception file ----".
           MOVE "N" TO WS-D-EOF.
           OPEN INPUT EXCEPTION-FILE.
           PERFORM UNTIL DETAIL-EOF
               READ EXCEPTION-FILE
                   AT END
                       SET DETAIL-EOF TO TRUE
                   NOT AT END
                       DISPLAY "  " EXC-RECORD
               END-READ
           END-PERFORM.
           CLOSE EXCEPTION-FILE.

           DISPLAY "Matched: " WS-MATCHED-COUNT
               "   Orphans: " WS-ORPHAN-COUNT.

       READ-MASTER.
           READ MASTER-FILE
               AT END
                   SET MASTER-EOF TO TRUE
                   MOVE HIGH-VALUES TO WS-MASTER-KEY
               NOT AT END
                   MOVE MST-ID TO WS-MASTER-KEY
           END-READ.

       READ-DETAIL.
           READ DETAIL-FILE
               AT END
                   SET DETAIL-EOF TO TRUE
                   MOVE HIGH-VALUES TO WS-DETAIL-KEY
               NOT AT END
                   MOVE DTL-ID TO WS-DETAIL-KEY
           END-READ.
```

**ผลลัพธ์:**

```
Step 1: branch feeds merged into DETAIL-FILE.
Step 2: batch update complete.
---- Final master balances ----
A001 SOMCHAI JAIDEE    1,350.50
A002 SUDA MEECHAI      2,000.00
A003 PRASERT KAEWTA      375.25
---- Exception file ----
  ORPHAN ID=A099 AMOUNT= 999.00
Matched: 002   Orphans: 001
```

### อธิบายภาพรวมของโปรแกรม — สรุปทั้ง Part 025–027

โปรแกรมนี้คือจุดสูงสุดที่รวมทุกความรู้จาก 3 Part เข้าด้วยกัน:

1. **`CREATE-TEST-DATA`**: จำลองข้อมูลจาก 2 สาขา ซึ่งแต่ละสาขาส่งไฟล์ที่เรียงลำดับภายในตัวเองแล้ว
   (สถานการณ์จริงที่พบบ่อย: แต่ละสาขามีระบบของตัวเองที่เรียงลำดับให้ก่อนส่งเข้าส่วนกลาง)
2. **`MERGE-BRANCH-FEEDS`**: ใช้ `MERGE` (ไม่ใช่ `SORT`) เพราะรู้แน่ชัดว่าทั้งสองไฟล์เรียงลำดับ
   มาแล้ว — เลือกใช้เครื่องมือที่มีประสิทธิภาพเหมาะสมกับสถานการณ์ตามที่เรียนในขั้นตอนที่ 267
3. **`RUN-BATCH-UPDATE`**: อัลกอริทึม Balanced Line จาก Part 026 ทำงานกับ `DETAIL-FILE` ที่ผ่าน
   การ `MERGE` มาแล้วโดยตรง พร้อม `REWRITE` อัปเดตยอดเงินจริงและบันทึก Exception แยกไฟล์
4. **`PRINT-FINAL-RESULTS`**: ยืนยันผลลัพธ์สุดท้ายทั้งหมด — A001 = 1000+100+250.50 = **1,350.50**,
   A002 ไม่มีธุรกรรมจากทั้งสองสาขาเลยจึงคงที่ **2,000.00**, A003 = 300+75.25 = **375.25**, และ
   ธุรกรรม A099 (ไม่มีบัญชีจริง) ถูกตรวจจับเป็น Orphan อย่างถูกต้อง

### ข้อควรระวัง

- โปรแกรมนี้เลือกใช้ `MERGE` แทน `SORT` เพราะข้อมูลทดสอบถูกออกแบบให้แต่ละสาขาเรียงลำดับมาก่อนแล้ว
  หากไม่มั่นใจว่าข้อมูลจากสาขาใดสาขาหนึ่งเรียงลำดับจริงหรือไม่ **ควรใช้ `SORT` แทนเสมอ** ตามกฎ
  ทองจากขั้นตอนที่ 269 แม้จะช้ากว่าเล็กน้อยแต่ปลอดภัยกว่ามาก
- สังเกตว่าโครงสร้างอัลกอริทึมทั้งหมด (Priming Read, HIGH-VALUES Sentinel, EVALUATE 3 ทาง, ลูปใน
  สำหรับ One-to-Many, REWRITE ปลอดภัยตามกฎทองของ Part 025) ถูกนำมาใช้ซ้ำได้ทั้งหมดโดยไม่ต้องแก้ไข
  แม้แต่บรรทัดเดียว เพียงแค่เปลี่ยนแหล่งที่มาของ `DETAIL-FILE` จาก "ไฟล์เดียวที่เรียงแล้ว" เป็น
  "ผลลัพธ์จากการ MERGE หลายไฟล์" เท่านั้น — นี่คือพลังของการออกแบบโปรแกรมแบบแยกส่วน (Modular
  Design) ที่เรียนมาตลอดหลักสูตร

### แบบฝึกหัดที่ 270.1 (แบบฝึกหัดสรุปท้าย Part)

**โจทย์**: จงปรับโปรแกรมให้รองรับสาขาที่ 3 (`BRANCH-C-FILE`) เพิ่มเข้ามาในขั้นตอน `MERGE` โดยไม่
ต้องแก้ไขส่วนอื่นของโปรแกรมเลย

**เฉลย**: เพิ่ม `SELECT BRANCH-C-FILE` และ `FD BRANCH-C-FILE` (โครงสร้างเดียวกับ Branch A/B)
สร้างข้อมูลทดสอบใน `CREATE-TEST-DATA` แล้วแก้ไขแค่บรรทัดเดียวใน `MERGE-BRANCH-FEEDS`:

```cobol
           MERGE MERGE-WORK-FILE
               ON ASCENDING KEY MERGE-ID
               USING BRANCH-A-FILE BRANCH-B-FILE BRANCH-C-FILE
               GIVING DETAIL-FILE.
```

ส่วน `RUN-BATCH-UPDATE`, `PROCESS-MATCHED-MASTER`, `LOG-ORPHAN-TRANSACTION` และ paragraph อื่น ๆ
ทั้งหมด**ไม่ต้องแก้ไขแม้แต่บรรทัดเดียว** เพราะทุก paragraph เหล่านั้นทำงานกับ `DETAIL-FILE` ที่เป็น
ผลลัพธ์สุดท้ายเท่านั้น ไม่สนใจว่าเบื้องหลังมันมาจากการ `MERGE` กี่ไฟล์ — นี่คือตัวอย่างที่ชัดเจนของ
ประโยชน์จากการออกแบบโปรแกรมแบบแยกส่วนที่เน้นย้ำมาตลอดทั้ง 3 Part นี้

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้ `SORT` และ `MERGE` ซึ่งเป็นชิ้นส่วนสุดท้ายที่ทำให้อัลกอริทึม Master-Detail
จาก Part 026 ใช้งานได้จริงกับข้อมูลจากโลกภายนอกที่ไม่เรียงลำดับมาให้:

- โครงสร้างพิเศษของ `SD` (Sort Description) ที่ต่างจาก `FD` และไม่ต้อง `OPEN`/`CLOSE` เอง
- `SORT` แบบพื้นฐาน (`USING...GIVING`) และการเรียงลำดับหลาย Key พร้อมกันด้วย `ASCENDING`/
  `DESCENDING`
- `INPUT PROCEDURE` กับคำสั่ง `RELEASE` สำหรับกรอง/แปลงข้อมูลก่อนเข้าสู่การเรียงลำดับ
- `OUTPUT PROCEDURE` กับคำสั่ง `RETURN` สำหรับประมวลผลข้อมูลหลังเรียงลำดับเสร็จแล้ว ก่อนเขียนไฟล์
  จริง
- การใช้ทั้ง `INPUT PROCEDURE` และ `OUTPUT PROCEDURE` พร้อมกันในคำสั่งเดียว
- การเชื่อมต่อผลลัพธ์จาก `SORT` เข้ากับอัลกอริทึม Master-Detail จาก Part 026 โดยตรง แก้ปัญหาที่
  ทิ้งค้างไว้ตั้งแต่ขั้นตอนที่ 252
- `MERGE` สำหรับรวมไฟล์ที่เรียงลำดับแล้วหลายไฟล์ (2 ไฟล์ขึ้นไป) อย่างมีประสิทธิภาพ
- **ความเสี่ยงสำคัญของ `MERGE`** เมื่อข้อมูลไม่เรียงลำดับจริง — พิสูจน์ด้วยการทดลองว่า GnuCOBOL
  "ใจดี" เกินไปจนอาจทำให้เข้าใจผิดว่าปลอดภัย ทั้งที่มาตรฐาน COBOL ไม่รับประกันพฤติกรรมนี้เลย
- โปรแกรมรวมสุดท้ายที่จำลองไปป์ไลน์งาน Batch แบบสมบูรณ์: หลายสาขา → `MERGE` → Master-Detail
  Matching → `REWRITE` อัปเดตยอดเงิน → รายงานข้อผิดพลาดแยกไฟล์

ด้วย Part 023–027 คุณได้เรียนรู้การประมวลผลไฟล์ตามลำดับอย่างครบวงจรแล้ว ตั้งแต่พื้นฐานที่สุดไปจนถึง
อัลกอริทึมระดับที่ใช้งานจริงในองค์กร Part ถัดไป (**Part 028**) จะพาคุณก้าวข้ามข้อจำกัดที่สำคัญที่สุด
ของไฟล์ตามลำดับที่เจอมาตลอด (ต้องอ่านเรียงตามลำดับเท่านั้น ค้นหา record กลางไฟล์โดยตรงไม่ได้ และ
`DELETE` ก็ใช้ไม่ได้เลยตามที่เรียนใน Part 025) ด้วยการแนะนำ **Indexed Files** และแนวคิด **ISAM**
ซึ่งช่วยให้ค้นหา แก้ไข และลบ record ใดก็ได้โดยตรงผ่าน Key โดยไม่ต้องอ่านไฟล์ทั้งหมดตั้งแต่ต้น

**[← กลับไป Part 026](part-026-master-detail-processing.md)** | **[ไปยัง Part 028: Indexed Files และ ISAM →](part-028-indexed-files.md)**
