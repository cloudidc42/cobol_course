# Part 026: การประมวลผลไฟล์แบบ Master-Detail (Matching Records) (ขั้นตอนที่ 251–260)

## คำนำของ Part นี้

Part 023–025 ทำให้เรามีเครื่องมือครบมือแล้วสำหรับการอ่าน-เขียน-แก้ไขไฟล์ตามลำดับหนึ่งไฟล์ แต่ในงาน
ธุรกิจจริง แทบไม่มีระบบไหนทำงานกับไฟล์เพียงไฟล์เดียวโดด ๆ สถานการณ์ที่พบบ่อยที่สุดในโลก COBOL คือ
การมี **ไฟล์หลัก (Master File)** ที่เก็บข้อมูลค่อนข้างคงที่ (เช่น ข้อมูลลูกค้า, ข้อมูลสินค้า) กับ
**ไฟล์รายการเปลี่ยนแปลง (Detail/Transaction File)** ที่เกิดขึ้นใหม่ทุกวัน (เช่น รายการฝาก-ถอนเงิน
ประจำวัน) แล้วต้อง **"จับคู่" (Match)** ข้อมูลทั้งสองไฟล์เข้าด้วยกันเพื่ออัปเดต Master ให้ทันสมัย —
นี่คือหัวใจของงาน **Batch Processing** ที่ธนาคาร ประกันภัย และองค์กรขนาดใหญ่ทั่วโลกรันกันทุกคืน

Part นี้จะพาคุณสร้างอัลกอริทึมการจับคู่ไฟล์ (Sequential Matching / Balanced Line Algorithm) ตั้งแต่
เวอร์ชันไร้เดียงสาที่สุดไปจนถึงเวอร์ชันที่รองรับทุกสถานการณ์จริง: Master ไม่มี Detail ตรงกัน, Detail
ไม่มี Master รองรับ (ข้อมูลกำพร้า), และ Master หนึ่งตัวมี Detail หลายรายการ ปิดท้ายด้วยโปรแกรมรวมที่
อัปเดต Master จริงพร้อมรายงานข้อผิดพลาดแยกต่างหาก

> **หมายเหตุสำคัญ**: เทคนิคในบทนี้ทั้งหมด**ต้องการให้ทั้งสองไฟล์เรียงลำดับตาม Key เดียวกันอยู่แล้ว**
> ก่อนเริ่มประมวลผล ในตัวอย่างของ Part นี้เราจะสร้างไฟล์ทดสอบให้เรียงลำดับถูกต้องด้วยมือไปก่อน
> ส่วนคำสั่งที่ใช้**เรียงลำดับไฟล์จริงที่ยังไม่เรียง** (`SORT` Statement) จะสอนอย่างละเอียดใน
> **Part 027** ถัดไปทันที

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน โค้ดทุกตัวอย่าง
> ผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง

---

## ขั้นตอนที่ 251: แนวคิด Master-Detail Processing และตัวอย่างแบบไร้เดียงสา (Naive)

### สถานการณ์ทางธุรกิจจริง

ลองนึกภาพระบบธนาคารง่าย ๆ ที่มี 2 ไฟล์:

- **`MASTER-FILE`** (ไฟล์หลัก): เก็บข้อมูลลูกค้าแต่ละคน 1 record ต่อ 1 คน ไม่เปลี่ยนแปลงบ่อย
  (สร้างครั้งเดียว แล้วอัปเดตเป็นครั้งคราว)
- **`DETAIL-FILE`** (ไฟล์รายการเปลี่ยนแปลง หรือเรียกอีกชื่อว่า **Transaction File**): เก็บรายการ
  ธุรกรรมที่เกิดขึ้น**ใหม่ทุกวัน** เช่น ลูกค้าคนไหนฝากเงินเท่าไรวันนี้

งานประจำคืนของธนาคาร (Nightly Batch Job) คือการอ่านทั้งสองไฟล์นี้ไปพร้อมกันแล้ว "จับคู่" รายการ
ธุรกรรมของวันนี้เข้ากับบัญชีลูกค้าที่ถูกต้อง เพื่ออัปเดตยอดเงินคงเหลือให้ทันสมัยก่อนเปิดทำการวันถัดไป

### ตัวอย่างแบบไร้เดียงสา (Naive Approach)

ลองดูวิธีที่ตรงไปตรงมาที่สุด: อ่านทั้งสองไฟล์ไปพร้อมกันทีละ record

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP251-CONCEPT-INTRO.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER251.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT DETAIL-FILE ASSIGN TO "DETAIL251.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MST-RECORD.
           05  MST-ID                  PIC X(4).
           05  MST-NAME                PIC X(15).

       FD  DETAIL-FILE.
       01  DTL-RECORD.
           05  DTL-ID                  PIC X(4).
           05  DTL-AMOUNT              PIC 9(5).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG                 PIC X VALUE "N".
           88  END-OF-FILE             VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> MASTER-FILE: the "who" - one row per customer, rarely
      *> changes (created once, updated occasionally).
           OPEN OUTPUT MASTER-FILE.
           MOVE "A001" TO MST-ID.
           MOVE "SOMCHAI JAIDEE " TO MST-NAME.
           WRITE MST-RECORD.
           MOVE "A002" TO MST-ID.
           MOVE "SUDA MEECHAI   " TO MST-NAME.
           WRITE MST-RECORD.
           CLOSE MASTER-FILE.

      *> DETAIL-FILE: the "what happened today" - one row per
      *> transaction, a brand new file every single day.
           OPEN OUTPUT DETAIL-FILE.
           MOVE "A001" TO DTL-ID.
           MOVE 300 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A002" TO DTL-ID.
           MOVE 500 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           CLOSE DETAIL-FILE.

      *> Both files happen to be sorted the same way today and
      *> have exactly one detail per master - so a plain parallel
      *> READ of both files, side by side, seems to "just work".
           OPEN INPUT MASTER-FILE.
           OPEN INPUT DETAIL-FILE.
           PERFORM UNTIL END-OF-FILE
               READ MASTER-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       READ DETAIL-FILE
                       DISPLAY MST-ID " " MST-NAME
                           " -- transaction amount " DTL-AMOUNT
               END-READ
           END-PERFORM.
           CLOSE MASTER-FILE.
           CLOSE DETAIL-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
A001 SOMCHAI JAIDEE  -- transaction amount 00300
A002 SUDA MEECHAI    -- transaction amount 00500
```

### ทำไมวิธีนี้ถึง "ดูเหมือน" ใช้ได้

โปรแกรมนี้ทำงานถูกต้องเพราะสถานการณ์ทดสอบ**เอื้ออำนวยเป็นพิเศษ**: ทั้งสองไฟล์มีจำนวน record เท่ากัน
พอดี (2 record) เรียงลำดับ ID ตรงกันเป๊ะ (A001, A002 ทั้งคู่) และแต่ละ Master มี Detail ตรงกันพอดี
1 รายการเสมอ — แต่นี่คือ**สถานการณ์ที่แทบไม่เคยเกิดขึ้นจริงในโลกธุรกิจ** ขั้นตอนถัดไปจะเผยให้เห็นว่า
สมมติฐานที่ซ่อนอยู่นี้เปราะบางแค่ไหน

### ข้อควรระวัง

- อย่าใช้วิธีนี้ในโปรแกรมจริงเด็ดขาด แม้จะดู "ทำงานได้" ในการทดสอบครั้งแรกก็ตาม เพราะมันมีสมมติฐาน
  ที่ซ่อนอยู่ 3 ข้อพร้อมกัน: (1) ทั้งสองไฟล์เรียงลำดับตรงกัน (2) จำนวน record เท่ากันพอดี
  (3) ทุก Master มี Detail ตรงกันพอดี 1 รายการเสมอ ซึ่งในโลกจริงแทบไม่มีข้อไหนรับประกันได้เลย
- สังเกตว่าโค้ดนี้ `READ DETAIL-FILE` โดยไม่มี `AT END` เตรียมรับมือเลย — ถ้า `DETAIL-FILE` มี
  record น้อยกว่า `MASTER-FILE` โปรแกรมจะ error ทันทีตอน `DETAIL-FILE` หมดก่อน

### แบบฝึกหัดที่ 251.1

**โจทย์**: จงระบุสมมติฐานที่ซ่อนอยู่ทั้ง 3 ข้อในโปรแกรม `STEP251-CONCEPT-INTRO` ที่ทำให้มันดู
เหมือนทำงานถูกต้อง ทั้งที่จริง ๆ แล้วเปราะบางมาก

**เฉลย**: (1) ทั้งสองไฟล์ต้องเรียงลำดับตาม ID เดียวกันทุกประการ (2) จำนวน record ในทั้งสองไฟล์ต้อง
เท่ากันพอดี (3) ทุก record ใน `MASTER-FILE` ต้องมี record คู่กันใน `DETAIL-FILE` แบบ 1 ต่อ 1 พอดี
เสมอ ไม่มี Master ที่ไม่มีธุรกรรม ไม่มีธุรกรรมที่ไม่มี Master รองรับ และไม่มี Master ใดมีธุรกรรม
มากกว่า 1 รายการ — ในความเป็นจริงสถานการณ์เหล่านี้ล้วนเกิดขึ้นเป็นประจำ (ลูกค้าบางคนไม่ทำธุรกรรม
วันนี้, มีข้อมูลผิดพลาดหลุดเข้ามา, ลูกค้าบางคนทำธุรกรรมหลายครั้งต่อวัน)

---

## ขั้นตอนที่ 252: ข้อกำหนดพื้นฐาน — ไฟล์ต้องเรียงลำดับตาม Key เดียวกัน

### ทดลอง: เมื่อไฟล์ไม่เรียงลำดับตรงกัน

ลองดูสิ่งที่เกิดขึ้นเมื่อสมมติฐานข้อแรก (การเรียงลำดับตรงกัน) ถูกละเมิด:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP252-UNSORTED-PROBLEM.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER252.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT DETAIL-FILE ASSIGN TO "DETAIL252.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MST-RECORD.
           05  MST-ID                  PIC X(4).
           05  MST-NAME                PIC X(15).

       FD  DETAIL-FILE.
       01  DTL-RECORD.
           05  DTL-ID                  PIC X(4).
           05  DTL-AMOUNT              PIC 9(5).

       WORKING-STORAGE SECTION.
       01  WS-M-EOF                    PIC X VALUE "N".
       01  WS-D-EOF                    PIC X VALUE "N".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> MASTER-FILE is sorted A001, A002, A003 (correct order)
      *> but DETAIL-FILE was accidentally created OUT of order:
      *> A002 comes before A001. Watch what happens when we try
      *> to match them by simply reading both files in step.
           OPEN OUTPUT MASTER-FILE.
           MOVE "A001" TO MST-ID.
           MOVE "SOMCHAI JAIDEE " TO MST-NAME.
           WRITE MST-RECORD.
           MOVE "A002" TO MST-ID.
           MOVE "SUDA MEECHAI   " TO MST-NAME.
           WRITE MST-RECORD.
           CLOSE MASTER-FILE.

           OPEN OUTPUT DETAIL-FILE.
           MOVE "A002" TO DTL-ID.
           MOVE 500 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A001" TO DTL-ID.
           MOVE 300 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           CLOSE DETAIL-FILE.

           OPEN INPUT MASTER-FILE.
           OPEN INPUT DETAIL-FILE.
           READ MASTER-FILE
               AT END MOVE "Y" TO WS-M-EOF
           END-READ.
           READ DETAIL-FILE
               AT END MOVE "Y" TO WS-D-EOF
           END-READ.

           DISPLAY "Naive side-by-side comparison (WRONG):".
           DISPLAY "  Master: " MST-ID " " MST-NAME.
           DISPLAY "  Detail: " DTL-ID " amount " DTL-AMOUNT.
           IF MST-ID = DTL-ID
               DISPLAY "  => Looks matched, but it is NOT really!"
           ELSE
               DISPLAY "  => IDs do not line up: "
                   MST-ID " vs " DTL-ID
           END-IF.

           CLOSE MASTER-FILE.
           CLOSE DETAIL-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Naive side-by-side comparison (WRONG):
  Master: A001 SOMCHAI JAIDEE
  Detail: A002 amount 00500
  => IDs do not line up: A001 vs A002
```

### วิเคราะห์ปัญหา

ตัวอย่างนี้พิสูจน์ให้เห็นชัดเจนว่าเมื่อไฟล์ `DETAIL-FILE` ไม่ได้เรียงลำดับตรงกับ `MASTER-FILE`
(A002 มาก่อน A001) การอ่านแบบขนานตรง ๆ จะจับคู่ผิดทันที (ในตัวอย่างนี้โปรแกรมยังตรวจพบว่า ID
ไม่ตรงกัน แต่ถ้าโค้ดไม่มีการตรวจสอบเช่นนี้ มันจะเอาเงิน 500 ของ A002 ไปบวกเข้าบัญชีของ A001 โดย
ไม่รู้ตัว ซึ่งเป็นความผิดพลาดร้ายแรงมากในระบบการเงิน)

### ทางแก้: การเรียงลำดับ (Sorting) ก่อนเสมอ

นี่คือเหตุผลที่ทุกอัลกอริทึม Master-Detail Matching (รวมถึงทุกตัวอย่างที่เหลือใน Part นี้) **ต้อง
มีข้อกำหนดเบื้องต้นเสมอว่าทั้งสองไฟล์ต้องเรียงลำดับตาม Key เดียวกันมาก่อนแล้ว** ในโลกจริง ไฟล์
Transaction ที่รับเข้ามาจากระบบภายนอกมักไม่เรียงลำดับมาให้ จึงจำเป็นต้องมีขั้นตอน**เรียงลำดับไฟล์
ก่อน** โดยใช้ **`SORT` Statement** ซึ่งเป็นหัวข้อทั้งหมดของ **Part 027** ถัดไป

### ข้อควรระวัง

- ในทุกตัวอย่างที่เหลือของ Part นี้ เราจะสร้างไฟล์ทดสอบให้เรียงลำดับถูกต้องด้วยมือ (เพราะยังไม่ได้
  เรียน `SORT`) แต่ในโปรแกรมจริงต้องมีขั้นตอน `SORT` แทรกอยู่ก่อนขั้นตอนการจับคู่เสมอ ไม่มีข้อยกเว้น
- ปัญหานี้เป็นปัญหาเงียบ (Silent Bug) ที่อันตรายที่สุดประเภทหนึ่ง เพราะโปรแกรมไม่ล่มและไม่มี error
  ใด ๆ เตือนเลย เพียงแค่คำนวณผิดโดยไม่มีใครรู้จนกว่าจะมีคนสังเกตเห็นยอดเงินผิดปกติ

### แบบฝึกหัดที่ 252.1

**โจทย์**: จงอธิบายว่าทำไมปัญหาไฟล์ไม่เรียงลำดับจึงอันตรายกว่าปัญหาไฟล์ที่มี record ไม่ครบ
(เช่น Detail ขาดหายไปบางรายการ)

**เฉลย**: เพราะปัญหาไฟล์ไม่เรียงลำดับจะทำให้เกิด**การจับคู่ผิดคน** (เช่น เอาเงินของ A002 ไปบวกให้
A001) ซึ่งเป็นความผิดพลาดที่**เงียบและร้ายแรง**เพราะตัวเลขยังคงดูสมเหตุสมผล (ยอดเงินเปลี่ยนแปลงจริง
แค่เปลี่ยนแปลงผิดบัญชี) ตรวจจับได้ยากมากถ้าไม่มีการกระทบยอด (Reconciliation) ในขณะที่ปัญหา Detail
ขาดหายไปบางรายการ อย่างน้อยก็จะทำให้ Master บางตัว "ไม่มีธุรกรรม" ซึ่งเป็นสถานการณ์ที่อัลกอริทึมที่
ถูกต้อง (ขั้นตอนที่ 255 เป็นต้นไป) จะตรวจจับและจัดการได้อย่างชัดเจน ไม่ทำให้ข้อมูลผิดเพี้ยนไปยัง
บัญชีอื่น

---

## ขั้นตอนที่ 253: อัลกอริทึมพื้นฐาน — Sequential Matching แบบ 1 ต่อ 1

### สร้างโครงลูปที่ปลอดภัยกว่าเดิม

ก่อนจะรับมือกับทุกกรณีที่ซับซ้อน เริ่มจากการสร้างโครงลูปพื้นฐานที่**อ่านทั้งสองไฟล์อย่างเป็นระบบ**
ด้วย paragraph แยกสำหรับแต่ละไฟล์ (`READ-MASTER`, `READ-DETAIL`) ซึ่งเป็นรูปแบบที่จะใช้ตลอด Part นี้

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP253-BASIC-MATCH.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER253.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT DETAIL-FILE ASSIGN TO "DETAIL253.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MST-RECORD.
           05  MST-ID                  PIC X(4).
           05  MST-NAME                PIC X(15).

       FD  DETAIL-FILE.
       01  DTL-RECORD.
           05  DTL-ID                  PIC X(4).
           05  DTL-AMOUNT              PIC 9(5).

       WORKING-STORAGE SECTION.
       01  WS-M-EOF                    PIC X VALUE "N".
           88  MASTER-EOF               VALUE "Y".
       01  WS-D-EOF                    PIC X VALUE "N".
           88  DETAIL-EOF               VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Both files are sorted by ID (A001, A002, A003) - this is
      *> a REQUIREMENT for the matching logic below to work.
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

           OPEN OUTPUT DETAIL-FILE.
           MOVE "A001" TO DTL-ID.
           MOVE 300 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A002" TO DTL-ID.
           MOVE 500 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A003" TO DTL-ID.
           MOVE 150 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           CLOSE DETAIL-FILE.

           OPEN INPUT MASTER-FILE.
           OPEN INPUT DETAIL-FILE.
           PERFORM READ-MASTER.
           PERFORM READ-DETAIL.

           PERFORM UNTIL MASTER-EOF
               DISPLAY MST-ID " " MST-NAME
                   " matches transaction of " DTL-AMOUNT
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
           END-READ.

       READ-DETAIL.
           READ DETAIL-FILE
               AT END
                   SET DETAIL-EOF TO TRUE
           END-READ.
```

**ผลลัพธ์:**

```
A001 SOMCHAI JAIDEE  matches transaction of 00300
A002 SUDA MEECHAI    matches transaction of 00500
A003 PRASERT KAEWTA  matches transaction of 00150
```

### อธิบายจุดสำคัญ

- แยก `READ-MASTER` และ `READ-DETAIL` ออกเป็น paragraph ของตัวเอง ทำให้ `MAIN-PARA` อ่านง่ายและ
  เรียกใช้ซ้ำได้จากหลายจุด — นี่คือรูปแบบมาตรฐานที่จะใช้ตลอด Part นี้
- โปรแกรมนี้**ยังคงมีข้อจำกัดเดิม**: มันสมมติว่าทุก Master มี Detail ตรงกันพอดี 1 ต่อ 1 เสมอ
  (ยังไม่มีการเปรียบเทียบ Key จริง ๆ) เป็นเพียงก้าวแรกในการวางโครงสร้างลูปที่ถูกต้องเท่านั้น
  ขั้นตอนถัดไปจะเริ่มเปรียบเทียบ Key จริงจัง

### ข้อควรระวัง

- สังเกตว่าเงื่อนไข `PERFORM UNTIL MASTER-EOF` ใช้แค่สถานะของ Master เท่านั้น ยังไม่ได้พิจารณา
  ว่า Detail หมดก่อนหรือยัง — นี่คือจุดอ่อนที่จะแก้ไขในขั้นตอนที่ 255–256
- การจัดวาง `PERFORM READ-MASTER.` และ `PERFORM READ-DETAIL.` ไว้ **ก่อน** เข้าลูปหลัก (แทนที่จะ
  อยู่ในลูปเหมือน Part 023) เป็นรูปแบบที่เรียกว่า **Priming Read** — อ่านครั้งแรกไว้ล่วงหน้าเพื่อให้
  มีข้อมูลพร้อมเปรียบเทียบตั้งแต่ก่อนเข้าลูป ซึ่งจำเป็นมากสำหรับอัลกอริทึมการจับคู่ไฟล์

### แบบฝึกหัดที่ 253.1

**โจทย์**: จงอธิบายว่า "Priming Read" (การอ่านล่วงหน้าก่อนเข้าลูป) คืออะไร และทำไมอัลกอริทึม
Master-Detail Matching จึงจำเป็นต้องใช้เทคนิคนี้

**เฉลย**: Priming Read คือการเรียก `READ` อย่างน้อยหนึ่งครั้งสำหรับแต่ละไฟล์**ก่อน**ที่จะเข้าสู่ลูป
หลักของโปรแกรม เพื่อให้มีข้อมูล record แรกพร้อมอยู่ในหน่วยความจำตั้งแต่ต้น อัลกอริทึม Master-Detail
Matching จำเป็นต้องใช้เทคนิคนี้เพราะทุกรอบของลูปต้อง**เปรียบเทียบ Key ของทั้งสองไฟล์ว่าเท่ากันหรือ
ไม่ก่อนตัดสินใจ**ว่าจะทำอะไรต่อ (จับคู่ ข้าม Master หรือข้าม Detail) หากไม่มีข้อมูลอยู่ในมือตั้งแต่
ก่อนเข้าลูป ก็จะไม่มีอะไรให้เปรียบเทียบได้เลยในรอบแรก

---

## ขั้นตอนที่ 254: อัปเดต Master ด้วย REWRITE ตามข้อมูลจาก Detail

### ประยุกต์ REWRITE (จาก Part 025) เข้ากับการจับคู่ไฟล์

ตอนนี้เรามาทำให้โปรแกรมมีประโยชน์จริง: แทนที่จะแค่ `DISPLAY` ผลลัพธ์ ให้**อัปเดตยอดเงินใน
`MASTER-FILE` จริง** ด้วย `REWRITE` ตามจำนวนเงินที่มาจาก `DETAIL-FILE`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP254-APPLY-TO-MASTER.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER254.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT DETAIL-FILE ASSIGN TO "DETAIL254.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MST-RECORD.
           05  MST-ID                  PIC X(4).
           05  MST-NAME                PIC X(15).
           05  MST-BALANCE             PIC 9(7)V99.

       FD  DETAIL-FILE.
       01  DTL-RECORD.
           05  DTL-ID                  PIC X(4).
           05  DTL-AMOUNT              PIC 9(5)V99.

       WORKING-STORAGE SECTION.
       01  WS-M-EOF                    PIC X VALUE "N".
           88  MASTER-EOF               VALUE "Y".
       01  WS-D-EOF                    PIC X VALUE "N".
           88  DETAIL-EOF               VALUE "Y".
       01  WS-DISPLAY-BAL              PIC ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
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
           MOVE 500.00 TO MST-BALANCE.
           WRITE MST-RECORD.
           CLOSE MASTER-FILE.

           OPEN OUTPUT DETAIL-FILE.
           MOVE "A001" TO DTL-ID.
           MOVE 300.00 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A002" TO DTL-ID.
           MOVE 500.00 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A003" TO DTL-ID.
           MOVE 150.00 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           CLOSE DETAIL-FILE.

      *> MASTER-FILE opens I-O so we can REWRITE each record after
      *> applying its matching deposit amount from DETAIL-FILE.
           OPEN I-O MASTER-FILE.
           OPEN INPUT DETAIL-FILE.
           PERFORM READ-MASTER.
           PERFORM READ-DETAIL.

           PERFORM UNTIL MASTER-EOF
               ADD DTL-AMOUNT TO MST-BALANCE
               REWRITE MST-RECORD
               MOVE MST-BALANCE TO WS-DISPLAY-BAL
               DISPLAY MST-ID " " MST-NAME
                   " new balance " WS-DISPLAY-BAL
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
           END-READ.

       READ-DETAIL.
           READ DETAIL-FILE
               AT END
                   SET DETAIL-EOF TO TRUE
           END-READ.
```

**ผลลัพธ์:**

```
A001 SOMCHAI JAIDEE  new balance   1,300.00
A002 SUDA MEECHAI    new balance   2,500.00
A003 PRASERT KAEWTA  new balance     650.00
```

### อธิบายจุดสำคัญ

- `MASTER-FILE` เปิดด้วย `OPEN I-O` (ทบทวนจาก Part 025) เพื่อให้ `READ` แล้วตามด้วย `REWRITE`
  ได้ในไฟล์เดียวกัน ในขณะที่ `DETAIL-FILE` เปิดแค่ `OPEN INPUT` เพราะเราแค่อ่านมันอย่างเดียว
  ไม่เคยเขียนกลับ
- `MST-BALANCE` ถูกออกแบบให้เป็น field**สุดท้าย**และเป็นชนิด**ตัวเลข** — นี่คือการนำกฎทองของ
  `REWRITE` จาก Part 025 ขั้นตอนที่ 246 มาใช้จริงโดยตรง เพราะข้อมูลตัวเลขไม่มีทางถูกตัดช่องว่างท้าย
  ทำให้ `REWRITE` ปลอดภัย 100% เสมอ
- ตรวจสอบผลลัพธ์ด้วยมือ: 1000.00 + 300.00 = 1300.00, 2000.00 + 500.00 = 2500.00,
  500.00 + 150.00 = 650.00 ตรงกันทั้งหมด

### ข้อควรระวัง

- โปรแกรมนี้ยังคงมีสมมติฐานเดิม (1 ต่อ 1 เท่านั้น) — ถ้า `DETAIL-FILE` มี record มากกว่าหรือน้อยกว่า
  `MASTER-FILE` ผลลัพธ์จะผิดพลาดทันที ขั้นตอนถัดไปจะเริ่มแก้ไขจุดอ่อนนี้อย่างจริงจัง
- ต้อง `ADD DTL-AMOUNT TO MST-BALANCE` **ก่อน** `REWRITE MST-RECORD` เสมอ เพราะ `REWRITE` เขียน
  ค่าปัจจุบันในหน่วยความจำ ณ ขณะนั้นลงไฟล์ (เหมือนกฎเดิมของ `WRITE` จาก Part 023)

### แบบฝึกหัดที่ 254.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมนี้จึงต้องเปิด `DETAIL-FILE` ด้วย `OPEN INPUT` แทนที่จะเป็น
`OPEN I-O` เหมือน `MASTER-FILE`

**เฉลย**: เพราะโปรแกรมนี้**ไม่เคยแก้ไขข้อมูลใน `DETAIL-FILE` เลย** มันแค่อ่านจำนวนเงินจากไฟล์นี้
มาบวกเข้า `MASTER-FILE` เท่านั้น การเปิดด้วย `OPEN INPUT` (อ่านอย่างเดียว) จึงเพียงพอและถูกต้องตาม
หลักการแล้ว การเปิดด้วย `OPEN I-O` ทั้งที่ไม่จำเป็นต้องเขียนจะเป็นการเปิดสิทธิ์เกินความจำเป็น
(Over-permission) ซึ่งขัดกับหลักการเขียนโค้ดที่ดีที่ควรเปิดสิทธิ์เท่าที่จำเป็นเท่านั้น

---

## ขั้นตอนที่ 255: จัดการ Master ที่ไม่มี Detail ตรงกัน — เทคนิค HIGH-VALUES Sentinel

### ปัญหา: Master บางตัวไม่มีธุรกรรมวันนี้

ในโลกจริง ลูกค้าบางคนอาจไม่ทำธุรกรรมเลยในวันนั้น ทำให้ `DETAIL-FILE` มี record น้อยกว่า
`MASTER-FILE` เราจำเป็นต้อง**เปรียบเทียบ Key จริง ๆ** แทนที่จะสมมติว่าตรงกันเสมอ

### เทคนิคสำคัญ: HIGH-VALUES เป็น "ค่าสูงสุดเสมือน"

`HIGH-VALUES` เป็นคำสงวนพิเศษของ COBOL ที่แทนค่าไบต์สูงสุดที่เป็นไปได้ (เช่น `X'FF'` ในหลาย
character set) เมื่อเปรียบเทียบกับข้อมูลตัวอักษรใด ๆ `HIGH-VALUES` จะ**ใหญ่กว่าเสมอ** เทคนิคที่
นิยมใช้ในอัลกอริทึม Master-Detail Matching คือ: **เมื่อไฟล์ไหนอ่านจนจบแล้ว (`AT END`) ให้กำหนดค่า
Key ของไฟล์นั้นเป็น `HIGH-VALUES`** ทำให้ไฟล์ที่หมดแล้วไม่มีทาง "ชนะ" การเปรียบเทียบ Key อีกต่อไป
(มันจะใหญ่กว่า Key ปกติเสมอ) ลดความซับซ้อนของโค้ดลงไปมาก เพราะไม่ต้องเขียน `IF MASTER-EOF ...`
แยกเป็นกรณีพิเศษอีกต่อไป

### ตัวอย่าง: จัดการ Master ที่ไม่มี Detail (A002 ไม่มีธุรกรรม)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP255-UNMATCHED-MASTER.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER255.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT DETAIL-FILE ASSIGN TO "DETAIL255.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MST-RECORD.
           05  MST-ID                  PIC X(4).
           05  MST-NAME                PIC X(15).

       FD  DETAIL-FILE.
       01  DTL-RECORD.
           05  DTL-ID                  PIC X(4).
           05  DTL-AMOUNT              PIC 9(5).

       WORKING-STORAGE SECTION.
       01  WS-M-EOF                    PIC X VALUE "N".
           88  MASTER-EOF               VALUE "Y".
       01  WS-D-EOF                    PIC X VALUE "N".
           88  DETAIL-EOF               VALUE "Y".
      *> A "sentinel" key used once a file runs out of records.
      *> HIGH-VALUES sorts higher than any normal ID, so an
      *> exhausted file's key never wins a comparison by mistake.
       01  WS-MASTER-KEY                PIC X(4).
       01  WS-DETAIL-KEY                PIC X(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> DETAIL-FILE has NO transaction for A002 on purpose.
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

           OPEN OUTPUT DETAIL-FILE.
           MOVE "A001" TO DTL-ID.
           MOVE 300 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A003" TO DTL-ID.
           MOVE 150 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           CLOSE DETAIL-FILE.

           OPEN INPUT MASTER-FILE.
           OPEN INPUT DETAIL-FILE.
           PERFORM READ-MASTER.
           PERFORM READ-DETAIL.

           PERFORM UNTIL MASTER-EOF
               IF WS-MASTER-KEY = WS-DETAIL-KEY
                   DISPLAY MST-ID " " MST-NAME
                       " -- MATCHED, amount " DTL-AMOUNT
                   PERFORM READ-MASTER
                   PERFORM READ-DETAIL
               ELSE
                   IF WS-MASTER-KEY < WS-DETAIL-KEY
                       DISPLAY MST-ID " " MST-NAME
                           " -- NO transaction today"
                       PERFORM READ-MASTER
                   ELSE
                       DISPLAY "  (skipping stray detail "
                           DTL-ID ")"
                       PERFORM READ-DETAIL
                   END-IF
               END-IF
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
A001 SOMCHAI JAIDEE  -- MATCHED, amount 00300
A002 SUDA MEECHAI    -- NO transaction today
A003 PRASERT KAEWTA  -- MATCHED, amount 00150
```

### อธิบายจุดสำคัญ

- `WS-MASTER-KEY` และ `WS-DETAIL-KEY` เป็นตัวแปรแยกต่างหากจาก `MST-ID`/`DTL-ID` โดยตั้งใจ เพื่อให้
  เก็บค่า `HIGH-VALUES` ได้เมื่อไฟล์หมด (ในขณะที่ `MST-ID`/`DTL-ID` เป็น field ใน `FD` ซึ่งจะยังคง
  ค่าจาก record สุดท้ายที่อ่านสำเร็จค้างอยู่ ทบทวนจาก Part 023 ขั้นตอนที่ 225)
- เมื่อ `A002` (Master) เทียบกับ `A003` (Detail ตัวถัดไปที่เหลืออยู่) จะพบว่า `WS-MASTER-KEY <
  WS-DETAIL-KEY` (เพราะ `"A002" < "A003"` ตามลำดับตัวอักษร) โปรแกรมจึงสรุปถูกต้องว่า A002 "ไม่มี
  ธุรกรรมวันนี้" แล้วเลื่อนอ่าน Master ตัวถัดไปโดยไม่แตะ Detail เลย
- สาขา `ELSE` สุดท้าย (กรณี `WS-MASTER-KEY > WS-DETAIL-KEY`) จะยังไม่เกิดขึ้นในตัวอย่างนี้ เพราะ
  ข้อมูลทดสอบยังไม่มี "Detail กำพร้า" (orphan) แต่เตรียมโค้ดไว้รองรับแล้ว — ขั้นตอนถัดไปจะทำให้
  สาขานี้ทำงานจริง

### ข้อควรระวัง

- ต้องกำหนดขนาดของ `WS-MASTER-KEY`/`WS-DETAIL-KEY` ให้ตรงกับ `MST-ID`/`DTL-ID` เสมอ (`PIC X(4)`
  ในตัวอย่างนี้) มิฉะนั้นการเปรียบเทียบอาจผิดเพี้ยนจากการเติมช่องว่างไม่ตรงกัน
- `MOVE HIGH-VALUES TO ws-key` ต้องอยู่ใน**สาขา `AT END` เท่านั้น** ถ้าใส่ผิดตำแหน่ง (เช่น ใส่ใน
  `NOT AT END` โดยไม่ตั้งใจ) ตรรกะทั้งหมดจะพังทันที

### แบบฝึกหัดที่ 255.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ `HIGH-VALUES` เป็น sentinel จึงทำให้โค้ดเรียบง่ายกว่าการเขียน
`IF MASTER-EOF OR WS-MASTER-KEY < WS-DETAIL-KEY` ตรง ๆ

**เฉลย**: เพราะเทคนิค `HIGH-VALUES` ทำให้เงื่อนไข `MASTER-EOF` ถูก "ซ่อน" อยู่ในการเปรียบเทียบ Key
เพียงเงื่อนไขเดียวโดยอัตโนมัติ — เราไม่จำเป็นต้องเขียนแยกกรณี "ไฟล์หมดหรือยัง" ออกจากกรณี "Key
เทียบกันแล้วใครใหญ่กว่า" เพราะไฟล์ที่หมดจะถูกแทนด้วย Key ที่ใหญ่ที่สุดเท่าที่จะเป็นไปได้เสมอ ทำให้
ตรรกะการเปรียบเทียบ 3 ทาง (เท่ากัน/น้อยกว่า/มากกว่า) ครอบคลุมทุกกรณีรวมถึงกรณีไฟล์หมดได้ในตัวโดย
ไม่ต้องเขียนเงื่อนไขซ้อนเพิ่มเติมเลย ทำให้โค้ดสั้นกระชับและมีโอกาสเขียนผิดพลาดน้อยลงมาก

---

## ขั้นตอนที่ 256: จัดการ Detail กำพร้า (Orphan) ด้วย EVALUATE TRUE

### ปัญหา: ธุรกรรมที่ไม่มี Master รองรับ

อีกสถานการณ์ที่พบได้บ่อยในโลกจริงคือ **ข้อมูลผิดพลาด**: มีรายการธุรกรรมเข้ามาสำหรับบัญชีที่ไม่มีอยู่
จริงในระบบ (เช่น พิมพ์เลขบัญชีผิด หรือบัญชีถูกปิดไปแล้วแต่ธุรกรรมยังหลุดเข้ามา) นี่คือกรณี
`WS-MASTER-KEY > WS-DETAIL-KEY` ที่ต้องจัดการอย่างระมัดระวัง**ไม่ใช่แค่ข้าม ควรรายงานเป็นข้อผิดพลาด**

### ตัวอย่าง: ตรวจจับและรายงาน Orphan Transaction

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP256-ORPHAN-DETAIL.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER256.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT DETAIL-FILE ASSIGN TO "DETAIL256.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MST-RECORD.
           05  MST-ID                  PIC X(4).
           05  MST-NAME                PIC X(15).

       FD  DETAIL-FILE.
       01  DTL-RECORD.
           05  DTL-ID                  PIC X(4).
           05  DTL-AMOUNT              PIC 9(5).

       WORKING-STORAGE SECTION.
       01  WS-M-EOF                    PIC X VALUE "N".
           88  MASTER-EOF               VALUE "Y".
       01  WS-D-EOF                    PIC X VALUE "N".
           88  DETAIL-EOF               VALUE "Y".
       01  WS-MASTER-KEY                PIC X(4).
       01  WS-DETAIL-KEY                PIC X(4).
       01  WS-ORPHAN-COUNT              PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> DETAIL-FILE has a transaction for "A099" which does NOT
      *> exist in MASTER-FILE at all - a classic orphan/error
      *> transaction that a real batch job must report, not just
      *> silently ignore.
           OPEN OUTPUT MASTER-FILE.
           MOVE "A001" TO MST-ID.
           MOVE "SOMCHAI JAIDEE " TO MST-NAME.
           WRITE MST-RECORD.
           MOVE "A002" TO MST-ID.
           MOVE "SUDA MEECHAI   " TO MST-NAME.
           WRITE MST-RECORD.
           CLOSE MASTER-FILE.

           OPEN OUTPUT DETAIL-FILE.
           MOVE "A001" TO DTL-ID.
           MOVE 300 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A002" TO DTL-ID.
           MOVE 500 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A099" TO DTL-ID.
           MOVE 999 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           CLOSE DETAIL-FILE.

           OPEN INPUT MASTER-FILE.
           OPEN INPUT DETAIL-FILE.
           PERFORM READ-MASTER.
           PERFORM READ-DETAIL.

      *> Loop until BOTH files are exhausted, so a trailing orphan
      *> detail (after the last master) is still processed.
           PERFORM UNTIL MASTER-EOF AND DETAIL-EOF
               EVALUATE TRUE
                   WHEN WS-MASTER-KEY = WS-DETAIL-KEY
                       DISPLAY MST-ID " " MST-NAME
                           " -- MATCHED, amount " DTL-AMOUNT
                       PERFORM READ-MASTER
                       PERFORM READ-DETAIL
                   WHEN WS-MASTER-KEY < WS-DETAIL-KEY
                       DISPLAY MST-ID " " MST-NAME
                           " -- NO transaction today"
                       PERFORM READ-MASTER
                   WHEN OTHER
                       DISPLAY "  ** ORPHAN TRANSACTION: ID "
                           DTL-ID " amount " DTL-AMOUNT
                           " has NO matching master **"
                       ADD 1 TO WS-ORPHAN-COUNT
                       PERFORM READ-DETAIL
               END-EVALUATE
           END-PERFORM.

           DISPLAY "Total orphan transactions: " WS-ORPHAN-COUNT.
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
A001 SOMCHAI JAIDEE  -- MATCHED, amount 00300
A002 SUDA MEECHAI    -- MATCHED, amount 00500
  ** ORPHAN TRANSACTION: ID A099 amount 00999 has NO matching master **
Total orphan transactions: 001
```

### อธิบายจุดสำคัญ — ทำไมต้องเปลี่ยนเงื่อนไขลูปเป็น AND

สังเกตการเปลี่ยนแปลงสำคัญ: `PERFORM UNTIL MASTER-EOF AND DETAIL-EOF` (ใช้ `AND` แทนที่จะเป็นแค่
`MASTER-EOF` เหมือนขั้นตอนก่อนหน้า) เหตุผลคือ **Orphan Transaction อาจปรากฏหลังจาก Master หมดแล้ว
ก็ได้** (เช่นในตัวอย่างนี้ `A099` มาหลัง `A002` ซึ่งเป็น Master ตัวสุดท้าย) ถ้ายังใช้เงื่อนไขเดิม
(`UNTIL MASTER-EOF`) ลูปจะจบทันทีที่ Master หมด โดยไม่ทันได้ตรวจพบ Orphan Transaction ที่หลงเหลือ
อยู่ท้ายไฟล์ Detail เลย

`EVALUATE TRUE` (ทบทวนจาก Part 011) ช่วยให้เขียนเงื่อนไข 3 ทาง (เท่ากัน/น้อยกว่า/มากกว่า) ได้อ่าน
ง่ายกว่า `IF...ELSE IF...ELSE` ซ้อนกันหลายชั้นมาก โดยเฉพาะเมื่อมีมากกว่า 2 เงื่อนไข

### ข้อควรระวัง

- อย่าลืมเปลี่ยนเงื่อนไขลูปจาก `UNTIL MASTER-EOF` เป็น `UNTIL MASTER-EOF AND DETAIL-EOF` เมื่อ
  เริ่มรองรับ Orphan Detail มิฉะนั้น Orphan ที่อยู่ท้ายไฟล์จะไม่ถูกตรวจพบเลย (บั๊กที่พบบ่อยมาก
  เมื่อพัฒนาอัลกอริทึมนี้แบบค่อยเป็นค่อยไป)
- ในระบบจริง Orphan Transaction ไม่ควรแค่ `DISPLAY` แล้วปล่อยผ่าน แต่ควรถูกบันทึกลงไฟล์รายงาน
  ข้อผิดพลาดแยกต่างหากเพื่อให้ทีมปฏิบัติการตรวจสอบภายหลัง (เทคนิคนี้จะสอนในขั้นตอนที่ 259)

### แบบฝึกหัดที่ 256.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมนี้จึงยังคงทำงานถูกต้อง แม้ Orphan Transaction (`A099`) จะมี
Key ที่**มากกว่า** Master ตัวสุดท้าย (`A002`) ในไฟล์ทุกกรณี ไม่ใช่แค่กรณีที่ Master หมดพอดี

**เฉลย**: เพราะเมื่อ Master หมด (`MASTER-EOF`) โปรแกรมจะกำหนด `WS-MASTER-KEY` เป็น `HIGH-VALUES`
ทันที ทำให้ `WS-MASTER-KEY` (ซึ่งตอนนี้คือค่าสูงสุดเสมือน) ไม่มีทาง**น้อยกว่า**อะไรได้อีกเลย ดังนั้น
ทุก Detail ที่เหลืออยู่หลังจากนี้ (ไม่ว่าจะมี Key เท่าไรก็ตาม ตราบใดที่ยังไม่ใช่ `HIGH-VALUES`)
จะตกไปอยู่ในสาขา `WHEN OTHER` (กรณี `WS-MASTER-KEY > WS-DETAIL-KEY`) โดยอัตโนมัติ ซึ่งตรงกับความ
เป็นจริงพอดี: ธุรกรรมใด ๆ ที่เหลืออยู่หลังจาก Master หมดแล้ว ย่อมเป็น Orphan ทั้งหมดโดยนิยาม

---

## ขั้นตอนที่ 257: จัดการ Master หนึ่งตัวมี Detail หลายรายการ (One-to-Many)

### ปัญหา: ลูกค้าทำธุรกรรมหลายครั้งในวันเดียว

สถานการณ์จริงที่พบบ่อยที่สุดคือลูกค้าคนหนึ่งอาจฝาก-ถอนเงินหลายครั้งในวันเดียวกัน ทำให้มี Detail
record หลายรายการที่มี Key เดียวกันติดกัน เราต้องใช้**ลูปซ้อนด้านใน** เพื่อดูดซับ (consume) ทุก
Detail ที่ตรงกับ Master ตัวปัจจุบันให้หมดก่อนจะเลื่อนไป Master ตัวถัดไป

### ตัวอย่าง: รวมยอดธุรกรรมทั้งหมดของแต่ละ Master

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP257-ONE-TO-MANY.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER257.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT DETAIL-FILE ASSIGN TO "DETAIL257.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MST-RECORD.
           05  MST-ID                  PIC X(4).
           05  MST-NAME                PIC X(15).

       FD  DETAIL-FILE.
       01  DTL-RECORD.
           05  DTL-ID                  PIC X(4).
           05  DTL-AMOUNT              PIC 9(5).

       WORKING-STORAGE SECTION.
       01  WS-M-EOF                    PIC X VALUE "N".
           88  MASTER-EOF               VALUE "Y".
       01  WS-D-EOF                    PIC X VALUE "N".
           88  DETAIL-EOF               VALUE "Y".
       01  WS-MASTER-KEY                PIC X(4).
       01  WS-DETAIL-KEY                PIC X(4).
       01  WS-MASTER-TOTAL              PIC 9(6) VALUE 0.
       01  WS-DETAIL-COUNT              PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> A001 has THREE transactions today - the classic
      *> one-master-to-many-details situation.
           OPEN OUTPUT MASTER-FILE.
           MOVE "A001" TO MST-ID.
           MOVE "SOMCHAI JAIDEE " TO MST-NAME.
           WRITE MST-RECORD.
           MOVE "A002" TO MST-ID.
           MOVE "SUDA MEECHAI   " TO MST-NAME.
           WRITE MST-RECORD.
           CLOSE MASTER-FILE.

           OPEN OUTPUT DETAIL-FILE.
           MOVE "A001" TO DTL-ID.
           MOVE 100 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A001" TO DTL-ID.
           MOVE 200 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A001" TO DTL-ID.
           MOVE 50 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A002" TO DTL-ID.
           MOVE 500 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           CLOSE DETAIL-FILE.

           OPEN INPUT MASTER-FILE.
           OPEN INPUT DETAIL-FILE.
           PERFORM READ-MASTER.
           PERFORM READ-DETAIL.

           PERFORM UNTIL MASTER-EOF
               MOVE 0 TO WS-MASTER-TOTAL
               MOVE 0 TO WS-DETAIL-COUNT
      *> Keep consuming details AS LONG AS they belong to the
      *> CURRENT master - this is what makes 1-to-many work.
               PERFORM UNTIL WS-DETAIL-KEY NOT = WS-MASTER-KEY
                   ADD DTL-AMOUNT TO WS-MASTER-TOTAL
                   ADD 1 TO WS-DETAIL-COUNT
                   PERFORM READ-DETAIL
               END-PERFORM
               DISPLAY MST-ID " " MST-NAME " -- "
                   WS-DETAIL-COUNT " transaction(s), total "
                   WS-MASTER-TOTAL
               PERFORM READ-MASTER
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
A001 SOMCHAI JAIDEE  -- 003 transaction(s), total 000350
A002 SUDA MEECHAI    -- 001 transaction(s), total 000500
```

### อธิบายจุดสำคัญ — ลูปซ้อนคือหัวใจของ One-to-Many

`PERFORM UNTIL WS-DETAIL-KEY NOT = WS-MASTER-KEY` คือลูปด้านในที่ **"ดูดซับ" (consume)** ทุก
Detail record ที่มี Key ตรงกับ Master ตัวปัจจุบันไปเรื่อย ๆ จนกว่า Key ของ Detail จะเปลี่ยนไป (ไม่
ว่าจะเป็นเพราะเจอ Master ตัวถัดไป หรือไฟล์ Detail หมดแล้วกลายเป็น `HIGH-VALUES`) เมื่อลูปในจบ
`WS-DETAIL-KEY` จะชี้ไปที่ Detail ตัวแรกของ Master **ตัวถัดไป**พอดี ทำให้รอบถัดไปของลูปนอกทำงาน
ต่อได้อย่างถูกต้องทันที โดยไม่ต้องมีการจัดการพิเศษใด ๆ เพิ่มเติม

100 + 200 + 50 = 350 สำหรับ A001 (3 รายการ) และ 500 สำหรับ A002 (1 รายการ) ตรงกับผลลัพธ์ที่ได้

### ข้อควรระวัง

- ต้องรีเซ็ต `WS-MASTER-TOTAL` และ `WS-DETAIL-COUNT` กลับเป็น 0 **ทุกครั้ง**ที่เริ่มประมวลผล
  Master ตัวใหม่ (ก่อนเข้าลูปใน) มิฉะนั้นยอดรวมจะสะสมข้าม Master ผิดพลาด
- ลำดับการทำงานสำคัญมาก: ต้อง `ADD`/`ADD 1` **ก่อน** `PERFORM READ-DETAIL` ในลูปใน ไม่เช่นนั้นจะ
  พลาดข้อมูลของ Detail ตัวปัจจุบันไป (ทบทวนหลักการเดียวกับ Part 023 ขั้นตอนที่ 226)

### แบบฝึกหัดที่ 257.1

**โจทย์**: จงอธิบายว่าทำไมลูปใน `PERFORM UNTIL WS-DETAIL-KEY NOT = WS-MASTER-KEY` จึงยังทำงาน
ถูกต้อง แม้ในกรณีที่ Master ตัวหนึ่งไม่มี Detail ตรงกันเลยสักรายการ (Key ไม่ตรงตั้งแต่รอบแรก)

**เฉลย**: เพราะ `PERFORM UNTIL` จะตรวจสอบเงื่อนไข**ก่อน**เข้าลูปเสมอ (ต่างจาก `PERFORM ... WITH
TEST AFTER`) หาก `WS-DETAIL-KEY` ไม่ตรงกับ `WS-MASTER-KEY` ตั้งแต่ต้น เงื่อนไข
`WS-DETAIL-KEY NOT = WS-MASTER-KEY` จะเป็นจริงทันที ทำให้ลูปในไม่ทำงานแม้แต่รอบเดียว
`WS-MASTER-TOTAL` และ `WS-DETAIL-COUNT` จึงยังคงเป็น 0 ตามที่รีเซ็ตไว้ ซึ่งตรงกับความเป็นจริง
(Master ตัวนั้นไม่มีธุรกรรมเลย) พอดี — พฤติกรรมนี้เป็นเหตุผลที่ Part นี้เลือกใช้ `PERFORM UNTIL`
(ทดสอบก่อนเข้า) แทน `PERFORM ... WITH TEST AFTER` (ทดสอบหลังเข้า) มาโดยตลอด

---

## ขั้นตอนที่ 258: อัลกอริทึมสมบูรณ์ — Balanced Line Algorithm รวมทุกกรณี

### รวมทุกเทคนิคเข้าด้วยกัน

ถึงเวลารวมทุกอย่างที่เรียนมา (Unmatched Master, Orphan Detail, One-to-Many) เข้าเป็นอัลกอริทึม
เดียวที่สมบูรณ์ — นี่คือรูปแบบที่เรียกกันในวงการว่า **"Balanced Line Algorithm"** หรือ
**"Sequential Match/Merge Algorithm"** ซึ่งเป็นรูปแบบมาตรฐานที่ใช้ในงาน Batch Processing ทั่วโลก

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP258-FULL-ALGORITHM.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER258.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT DETAIL-FILE ASSIGN TO "DETAIL258.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MST-RECORD.
           05  MST-ID                  PIC X(4).
           05  MST-NAME                PIC X(15).

       FD  DETAIL-FILE.
       01  DTL-RECORD.
           05  DTL-ID                  PIC X(4).
           05  DTL-AMOUNT              PIC 9(5).

       WORKING-STORAGE SECTION.
       01  WS-M-EOF                    PIC X VALUE "N".
           88  MASTER-EOF               VALUE "Y".
       01  WS-D-EOF                    PIC X VALUE "N".
           88  DETAIL-EOF               VALUE "Y".
       01  WS-MASTER-KEY                PIC X(4).
       01  WS-DETAIL-KEY                PIC X(4).
       01  WS-MASTER-TOTAL              PIC 9(6) VALUE 0.
       01  WS-DETAIL-COUNT              PIC 9(3) VALUE 0.
       01  WS-ORPHAN-COUNT              PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> This test data combines all THREE situations at once:
      *>  A001 - two transactions (one-to-many)
      *>  A002 - no transaction at all (unmatched master)
      *>  A003 - one transaction (simple 1-to-1 match)
      *>  A099 - a transaction with NO master at all (orphan)
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

           OPEN OUTPUT DETAIL-FILE.
           MOVE "A001" TO DTL-ID.
           MOVE 100 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A001" TO DTL-ID.
           MOVE 200 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A003" TO DTL-ID.
           MOVE 150 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A099" TO DTL-ID.
           MOVE 999 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           CLOSE DETAIL-FILE.

           OPEN INPUT MASTER-FILE.
           OPEN INPUT DETAIL-FILE.
           PERFORM READ-MASTER.
           PERFORM READ-DETAIL.

      *> The classic "balanced line" / sequential-match algorithm:
      *> compare the two current keys and decide EQUAL / LESS /
      *> GREATER on every pass, looping until BOTH files are done.
           PERFORM UNTIL MASTER-EOF AND DETAIL-EOF
               EVALUATE TRUE
                   WHEN WS-MASTER-KEY = WS-DETAIL-KEY
                       PERFORM PROCESS-MATCHED-MASTER
                   WHEN WS-MASTER-KEY < WS-DETAIL-KEY
                       DISPLAY MST-ID " " MST-NAME
                           " -- no transaction today"
                       PERFORM READ-MASTER
                   WHEN OTHER
                       DISPLAY "  ** ORPHAN: " DTL-ID
                           " amount " DTL-AMOUNT " has no master **"
                       ADD 1 TO WS-ORPHAN-COUNT
                       PERFORM READ-DETAIL
               END-EVALUATE
           END-PERFORM.

           DISPLAY "Total orphan transactions: " WS-ORPHAN-COUNT.
           CLOSE MASTER-FILE.
           CLOSE DETAIL-FILE.
           STOP RUN.

       PROCESS-MATCHED-MASTER.
           MOVE 0 TO WS-MASTER-TOTAL.
           MOVE 0 TO WS-DETAIL-COUNT.
           PERFORM UNTIL WS-DETAIL-KEY NOT = WS-MASTER-KEY
               ADD DTL-AMOUNT TO WS-MASTER-TOTAL
               ADD 1 TO WS-DETAIL-COUNT
               PERFORM READ-DETAIL
           END-PERFORM.
           DISPLAY MST-ID " " MST-NAME " -- "
               WS-DETAIL-COUNT " transaction(s), total "
               WS-MASTER-TOTAL.
           PERFORM READ-MASTER.

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
A001 SOMCHAI JAIDEE  -- 002 transaction(s), total 000300
A002 SUDA MEECHAI    -- no transaction today
A003 PRASERT KAEWTA  -- 001 transaction(s), total 000150
  ** ORPHAN: A099 amount 00999 has no master **
Total orphan transactions: 001
```

### อธิบายภาพรวมของอัลกอริทึม

โปรแกรมนี้คือ**อัลกอริทึมมาตรฐานฉบับสมบูรณ์**ที่ประกอบด้วยส่วนสำคัญ 4 ส่วน:

1. **Priming Read** (ขั้นตอนที่ 253): อ่าน record แรกของทั้งสองไฟล์ก่อนเข้าลูปหลัก
2. **HIGH-VALUES Sentinel** (ขั้นตอนที่ 255): ทำให้ไฟล์ที่หมดแล้วไม่ชนะการเปรียบเทียบ Key อีก
3. **EVALUATE TRUE แบบ 3 ทาง** (ขั้นตอนที่ 256): EQUAL → จับคู่, LESS → Master ไม่มี Detail,
   GREATER → Detail กำพร้า (Orphan)
4. **ลูปในสำหรับ One-to-Many** (ขั้นตอนที่ 257): ดูดซับ Detail ทุกตัวที่ตรงกับ Master ปัจจุบัน
   ก่อนเลื่อนไป Master ถัดไป

การแยก `PROCESS-MATCHED-MASTER` ออกเป็น paragraph ต่างหาก (แทนที่จะเขียนลูปในซ้อนอยู่ตรง ๆ ใน
`MAIN-PARA`) ทำให้โค้ดสะอาดขึ้นมาก ตามหลัก Modularization ที่เรียนมาตั้งแต่ Part 014

### ข้อควรระวัง

- อัลกอริทึมนี้คือ**แม่แบบ (Template)** ที่ใช้ซ้ำได้กับสถานการณ์ Master-Detail แทบทุกแบบในโลกจริง
  เพียงเปลี่ยนชื่อไฟล์ ชนิดข้อมูล และตรรกะข้างในแต่ละสาขาให้ตรงกับโจทย์ธุรกิจ โครงหลักของอัลกอริทึม
  (Priming Read + Sentinel + 3-way EVALUATE + ลูปในสำหรับ One-to-Many) ยังคงเหมือนเดิมเสมอ
- ควรท่องจำโครงสร้างนี้ให้แม่น เพราะจะพบเจอซ้ำแล้วซ้ำเล่าตลอดการทำงานกับ COBOL ในโลกจริง โดยเฉพาะ
  งาน Batch Processing ระดับองค์กร (และจะพบอีกครั้งในรูปแบบที่คล้ายกันเมื่อเรียน `MERGE` Statement
  ใน Part 027 ถัดไป)

### แบบฝึกหัดที่ 258.1

**โจทย์**: จงอธิบายว่าทำไมการแยก `PROCESS-MATCHED-MASTER` ออกเป็น paragraph ต่างหาก จึงทำให้โค้ด
ใน `MAIN-PARA` อ่านง่ายขึ้นอย่างชัดเจน เมื่อเทียบกับการเขียนทุกอย่างไว้ใน `MAIN-PARA` เดียว

**เฉลย**: เพราะเมื่อแยกออกมาแล้ว `MAIN-PARA` จะเหลือแค่โครงหลักของอัลกอริทึม (การเปรียบเทียบ Key
3 ทางและตัดสินใจว่าจะทำอะไร) โดยไม่ต้องปะปนกับรายละเอียดการคำนวณยอดรวมและนับจำนวนรายการที่ซับซ้อน
กว่า คนอ่านโค้ดสามารถเข้าใจ**ภาพรวม**ของอัลกอริทึมได้ทันทีจาก `MAIN-PARA` เพียงอย่างเดียว โดยไม่ต้อง
สนใจรายละเอียดการทำงานภายในของแต่ละกรณีจนกว่าจะต้องการเจาะลึกจริง ๆ นี่คือหลักการ "Separation of
Concerns" (แยกความรับผิดชอบ) ที่สำคัญมากสำหรับการเขียนโปรแกรมขนาดใหญ่ให้ดูแลรักษาได้ในระยะยาว

---

## ขั้นตอนที่ 259: รายงานข้อผิดพลาดแยกต่างหาก — Exception Report File

### ทำไมต้องมีไฟล์รายงานข้อผิดพลาดแยก

ในระบบจริงระดับองค์กร การ `DISPLAY` ข้อผิดพลาดออกหน้าจอเฉย ๆ ไม่เพียงพอ เพราะงาน Batch มักรันตอน
กลางคืนโดยไม่มีคนเฝ้าหน้าจอ จึงต้อง**บันทึกข้อผิดพลาดลงไฟล์แยกต่างหาก** (Exception Report /
Error Report) ให้ทีมปฏิบัติการตรวจสอบในเช้าวันถัดไป

### ตัวอย่าง: เขียน Orphan Transaction ลงไฟล์ Exception พร้อมสรุปผล

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP259-EXCEPTION-REPORT.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER259.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT DETAIL-FILE ASSIGN TO "DETAIL259.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT EXCEPTION-FILE ASSIGN TO "EXCEPTION259.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MST-RECORD.
           05  MST-ID                  PIC X(4).
           05  MST-NAME                PIC X(15).

       FD  DETAIL-FILE.
       01  DTL-RECORD.
           05  DTL-ID                  PIC X(4).
           05  DTL-AMOUNT              PIC 9(5).

       FD  EXCEPTION-FILE.
       01  EXC-RECORD                  PIC X(40).

       WORKING-STORAGE SECTION.
       01  WS-M-EOF                    PIC X VALUE "N".
           88  MASTER-EOF               VALUE "Y".
       01  WS-D-EOF                    PIC X VALUE "N".
           88  DETAIL-EOF               VALUE "Y".
       01  WS-MASTER-KEY                PIC X(4).
       01  WS-DETAIL-KEY                PIC X(4).
       01  WS-MATCHED-COUNT             PIC 9(3) VALUE 0.
       01  WS-UNMATCHED-COUNT           PIC 9(3) VALUE 0.
       01  WS-ORPHAN-COUNT              PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT MASTER-FILE.
           MOVE "A001" TO MST-ID.
           MOVE "SOMCHAI JAIDEE " TO MST-NAME.
           WRITE MST-RECORD.
           MOVE "A002" TO MST-ID.
           MOVE "SUDA MEECHAI   " TO MST-NAME.
           WRITE MST-RECORD.
           CLOSE MASTER-FILE.

           OPEN OUTPUT DETAIL-FILE.
           MOVE "A001" TO DTL-ID.
           MOVE 300 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A099" TO DTL-ID.
           MOVE 999 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           CLOSE DETAIL-FILE.

           OPEN INPUT MASTER-FILE.
           OPEN INPUT DETAIL-FILE.
           OPEN OUTPUT EXCEPTION-FILE.
           PERFORM READ-MASTER.
           PERFORM READ-DETAIL.

           PERFORM UNTIL MASTER-EOF AND DETAIL-EOF
               EVALUATE TRUE
                   WHEN WS-MASTER-KEY = WS-DETAIL-KEY
                       ADD 1 TO WS-MATCHED-COUNT
                       DISPLAY MST-ID " " MST-NAME " -- MATCHED"
                       PERFORM READ-MASTER
                       PERFORM READ-DETAIL
                   WHEN WS-MASTER-KEY < WS-DETAIL-KEY
                       ADD 1 TO WS-UNMATCHED-COUNT
                       DISPLAY MST-ID " " MST-NAME
                           " -- no transaction"
                       PERFORM READ-MASTER
                   WHEN OTHER
                       ADD 1 TO WS-ORPHAN-COUNT
                       MOVE SPACES TO EXC-RECORD
                       STRING "ORPHAN ID=" DTL-ID
                           " AMOUNT=" DTL-AMOUNT
                           DELIMITED BY SIZE
                           INTO EXC-RECORD
                       WRITE EXC-RECORD
                       PERFORM READ-DETAIL
               END-EVALUATE
           END-PERFORM.

           CLOSE MASTER-FILE.
           CLOSE DETAIL-FILE.
           CLOSE EXCEPTION-FILE.

           DISPLAY "---- Summary ----".
           DISPLAY "Matched   : " WS-MATCHED-COUNT.
           DISPLAY "Unmatched : " WS-UNMATCHED-COUNT.
           DISPLAY "Orphans   : " WS-ORPHAN-COUNT.
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
A001 SOMCHAI JAIDEE  -- MATCHED
A002 SUDA MEECHAI    -- no transaction
---- Summary ----
Matched   : 001
Unmatched : 001
Orphans   : 001
```

**เนื้อหาไฟล์ `EXCEPTION259.DAT`:**

```
ORPHAN ID=A099 AMOUNT=00999
```

### อธิบายจุดสำคัญ

- โปรแกรมนี้เปิดพร้อมกันถึง**สามไฟล์**: `MASTER-FILE` (`INPUT`), `DETAIL-FILE` (`INPUT`), และ
  `EXCEPTION-FILE` (`OUTPUT`) — ยืนยันอีกครั้งว่า COBOL รองรับการเปิดหลายไฟล์พร้อมกันในโหมดต่างกัน
  ได้อย่างไม่จำกัด ตราบใดที่แต่ละไฟล์มี Internal Name ของตัวเอง
- สังเกตว่า console (หน้าจอ) แสดงเฉพาะสถานะปกติ (matched/unmatched) ส่วนข้อผิดพลาด (orphan) ถูก
  แยกไปเก็บในไฟล์ `EXCEPTION259.DAT` ต่างหาก ไม่ปะปนกัน — ทำให้ทั้งรายงานสรุปและรายงานข้อผิดพลาด
  อ่านง่าย ตรงจุดประสงค์ของแต่ละฝ่ายที่ต้องใช้งาน
- ตัวนับ 3 ตัว (`WS-MATCHED-COUNT`, `WS-UNMATCHED-COUNT`, `WS-ORPHAN-COUNT`) ให้ภาพสรุปผลการรัน
  แบบตัวเลขที่ตรวจสอบได้อย่างรวดเร็ว โดยไม่ต้องไล่อ่านทุกบรรทัดของรายงาน

### ข้อควรระวัง

- `MOVE SPACES TO EXC-RECORD` ก่อน `STRING` ทุกครั้งยังคงจำเป็นเสมอ (กฎจาก Part 024 ขั้นตอนที่
  236) มิฉะนั้นจะเจอ `status = 71` ทันทีเหมือนที่พิสูจน์มาแล้วหลายครั้งในหลักสูตรนี้
- ในระบบจริง มักมีไฟล์ Exception แยกกันหลายประเภทตามความรุนแรง เช่น `WARNING.DAT` สำหรับกรณีที่
  ยังประมวลผลต่อได้ และ `CRITICAL.DAT` สำหรับกรณีที่ต้องหยุดงาน Batch ทั้งหมดทันทีเพื่อรอคนตรวจสอบ

### แบบฝึกหัดที่ 259.1

**โจทย์**: จงปรับโปรแกรมให้เขียนข้อมูล **Master ที่ไม่มีธุรกรรม** (unmatched master) ลงไฟล์รายงาน
แยกอีกไฟล์หนึ่งชื่อ `NOACTIVITY.DAT` นอกเหนือจากไฟล์ Exception เดิม

**เฉลย**: เพิ่ม `SELECT NOACTIVITY-FILE` และ `FD` ใหม่ (`01 NOACT-RECORD PIC X(30).`) แล้วแก้ไข
สาขา `WHEN WS-MASTER-KEY < WS-DETAIL-KEY` ให้เขียนลงไฟล์ใหม่ด้วย:

```cobol
                   WHEN WS-MASTER-KEY < WS-DETAIL-KEY
                       ADD 1 TO WS-UNMATCHED-COUNT
                       MOVE SPACES TO NOACT-RECORD
                       STRING MST-ID " " MST-NAME
                           DELIMITED BY SIZE
                           INTO NOACT-RECORD
                       WRITE NOACT-RECORD
                       PERFORM READ-MASTER
```

พร้อมเพิ่ม `OPEN OUTPUT NOACTIVITY-FILE` ก่อนลูปหลักและ `CLOSE NOACTIVITY-FILE` หลังลูปจบ ผลลัพธ์
คือระบบจะมีไฟล์รายงานแยกกันชัดเจนสามระดับ: รายงานปกติ (หน้าจอ), รายงาน Master ที่ไม่มีธุรกรรม
(`NOACTIVITY.DAT`), และรายงานข้อผิดพลาดร้ายแรง (`EXCEPTION259.DAT`)

---

## ขั้นตอนที่ 260: โปรแกรมรวมสุดท้าย — ระบบประมวลผล Transaction ประจำวันแบบสมบูรณ์

### โจทย์: งาน Batch อัปเดตบัญชีลูกค้าประจำคืน

ขั้นตอนสุดท้ายของ Part นี้จะรวมทุกเทคนิคที่เรียนมาตลอด Part 023–026 เข้าไว้ในโปรแกรมเดียวที่สมบูรณ์
ที่สุด: อัปเดตยอดเงินจริงด้วย `REWRITE`, จัดการทุกกรณี (matched/unmatched/orphan/one-to-many),
บันทึกรายงานข้อผิดพลาดแยก, และแสดงผลลัพธ์สุดท้ายครบถ้วน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP260-FULL-BATCH-UPDATE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER260.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT DETAIL-FILE ASSIGN TO "DETAIL260.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT EXCEPTION-FILE ASSIGN TO "EXCEPTION260.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MST-RECORD.
           05  MST-ID                  PIC X(4).
           05  MST-NAME                PIC X(15).
           05  MST-BALANCE             PIC 9(7)V99.

       FD  DETAIL-FILE.
       01  DTL-RECORD.
           05  DTL-ID                  PIC X(4).
           05  DTL-AMOUNT              PIC 9(5)V99.

       FD  EXCEPTION-FILE.
       01  EXC-RECORD                  PIC X(40).

       WORKING-STORAGE SECTION.
       01  WS-M-EOF                    PIC X VALUE "N".
           88  MASTER-EOF               VALUE "Y".
       01  WS-D-EOF                    PIC X VALUE "N".
           88  DETAIL-EOF               VALUE "Y".
       01  WS-MASTER-KEY                PIC X(4).
       01  WS-DETAIL-KEY                PIC X(4).
       01  WS-DEPOSIT-TOTAL             PIC 9(6)V99 VALUE 0.
       01  WS-DEPOSIT-COUNT             PIC 9(3) VALUE 0.
       01  WS-MATCHED-COUNT             PIC 9(3) VALUE 0.
       01  WS-UNMATCHED-COUNT           PIC 9(3) VALUE 0.
       01  WS-ORPHAN-COUNT              PIC 9(3) VALUE 0.
       01  WS-DISPLAY-AMOUNT            PIC ZZZ9.99.
       01  WS-DISPLAY-BAL               PIC ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM CREATE-TEST-DATA.
           PERFORM RUN-BATCH-UPDATE.
           PERFORM PRINT-FINAL-MASTER.
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

      *> A001: two deposits today. A002: none. A003: one deposit.
      *> A099: a stray transaction for an account that does not
      *> exist in the master file at all.
           OPEN OUTPUT DETAIL-FILE.
           MOVE "A001" TO DTL-ID.
           MOVE 100.00 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A001" TO DTL-ID.
           MOVE 250.50 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A003" TO DTL-ID.
           MOVE 75.25 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           MOVE "A099" TO DTL-ID.
           MOVE 999.00 TO DTL-AMOUNT.
           WRITE DTL-RECORD.
           CLOSE DETAIL-FILE.

       RUN-BATCH-UPDATE.
      *> MASTER-FILE opens I-O (read + REWRITE); DETAIL-FILE and
      *> EXCEPTION-FILE only need one direction each.
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
                       ADD 1 TO WS-UNMATCHED-COUNT
                       DISPLAY MST-ID " " MST-NAME
                           " -- no deposit today, balance"
                           " unchanged"
                       PERFORM READ-MASTER
                   WHEN OTHER
                       PERFORM LOG-ORPHAN-TRANSACTION
                       PERFORM READ-DETAIL
               END-EVALUATE
           END-PERFORM.

           CLOSE MASTER-FILE.
           CLOSE DETAIL-FILE.
           CLOSE EXCEPTION-FILE.

           DISPLAY "---- Batch summary ----".
           DISPLAY "Matched masters  : " WS-MATCHED-COUNT.
           DISPLAY "Unmatched masters: " WS-UNMATCHED-COUNT.
           DISPLAY "Orphan details   : " WS-ORPHAN-COUNT.

       PROCESS-MATCHED-MASTER.
           ADD 1 TO WS-MATCHED-COUNT.
           MOVE 0 TO WS-DEPOSIT-TOTAL.
           MOVE 0 TO WS-DEPOSIT-COUNT.
           PERFORM UNTIL WS-DETAIL-KEY NOT = WS-MASTER-KEY
               ADD DTL-AMOUNT TO WS-DEPOSIT-TOTAL
               ADD 1 TO WS-DEPOSIT-COUNT
               PERFORM READ-DETAIL
           END-PERFORM.
           ADD WS-DEPOSIT-TOTAL TO MST-BALANCE.
           REWRITE MST-RECORD.
           MOVE MST-BALANCE TO WS-DISPLAY-BAL.
           DISPLAY MST-ID " " MST-NAME " -- " WS-DEPOSIT-COUNT
               " deposit(s), new balance " WS-DISPLAY-BAL.
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

       PRINT-FINAL-MASTER.
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

           DISPLAY "---- Exception file content ----".
           OPEN INPUT EXCEPTION-FILE.
           MOVE "N" TO WS-D-EOF.
           PERFORM UNTIL DETAIL-EOF
               READ EXCEPTION-FILE
                   AT END
                       SET DETAIL-EOF TO TRUE
                   NOT AT END
                       DISPLAY "  " EXC-RECORD
               END-READ
           END-PERFORM.
           CLOSE EXCEPTION-FILE.

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
A001 SOMCHAI JAIDEE  -- 002 deposit(s), new balance   1,350.50
A002 SUDA MEECHAI    -- no deposit today, balance unchanged
A003 PRASERT KAEWTA  -- 001 deposit(s), new balance     375.25
---- Batch summary ----
Matched masters  : 002
Unmatched masters: 001
Orphan details   : 001
---- Final master balances ----
A001 SOMCHAI JAIDEE    1,350.50
A002 SUDA MEECHAI      2,000.00
A003 PRASERT KAEWTA      375.25
---- Exception file content ----
  ORPHAN ID=A099 AMOUNT= 999.00
```

### อธิบายภาพรวมของโปรแกรม

โปรแกรมนี้คือการนำทุกความรู้จาก Part 023–026 มาประกอบกันเป็นระบบที่ใกล้เคียงงานจริงที่สุด:

- **`CREATE-TEST-DATA`**: จำลองข้อมูลตั้งต้น (ในระบบจริง `MASTER-FILE` มาจากการรันครั้งก่อนหน้า
  และ `DETAIL-FILE` มาจากระบบหน้าร้าน/ATM ที่ส่งมาทุกวัน)
- **`RUN-BATCH-UPDATE`**: หัวใจของอัลกอริทึม Balanced Line ที่รวม `REWRITE` (อัปเดตยอดเงินจริง)
  เข้ากับการรายงานข้อผิดพลาดแยกไฟล์ (`EXCEPTION-FILE`)
- ตรวจสอบผลลัพธ์: A001 = 1000 + 100 + 250.50 = **1,350.50** ✓, A002 = 2000 (ไม่เปลี่ยน) ✓,
  A003 = 300 + 75.25 = **375.25** ✓ ทั้งหมดตรงกับที่แสดงในรายงานสรุปและรายงานยอดสุดท้าย
- **`PRINT-FINAL-MASTER`**: เปิดไฟล์ทั้งสองกลับมาอ่านอิสระเพื่อยืนยันผลลัพธ์สุดท้าย แสดงให้เห็นว่า
  ข้อมูลถูกบันทึกลงดิสก์จริงถูกต้อง ไม่ใช่แค่ค่าที่ค้างอยู่ในหน่วยความจำระหว่างการประมวลผล

### ข้อควรระวัง

- โปรแกรมนี้ยังคงยึดกฎทองจาก Part 025 ขั้นตอนที่ 246 อย่างเคร่งครัด: `MST-BALANCE` เป็นตัวเลขและ
  อยู่ท้ายสุดของ record ทำให้ `REWRITE` ปลอดภัยเสมอ
- ในงาน Production จริง ขั้นตอน `CREATE-TEST-DATA` จะไม่มีอยู่ในโปรแกรมเดียวกับ `RUN-BATCH-UPDATE`
  เพราะ `MASTER-FILE` ควรเป็นไฟล์ที่**มีอยู่แล้วจากการรันครั้งก่อน** ไม่ใช่ถูกสร้างใหม่ทุกครั้ง — ใน
  ตัวอย่างนี้รวมไว้ในโปรแกรมเดียวเพื่อความสะดวกในการทดสอบและรันซ้ำได้เองเท่านั้น

### แบบฝึกหัดที่ 260.1 (แบบฝึกหัดสรุปท้าย Part)

**โจทย์**: จงเพิ่มการตรวจสอบว่า**ยอดฝากรวมของ Master แต่ละตัวห้ามเกิน 500 บาทต่อวัน** หากเกิน
ให้เขียนรายการนั้นลงไฟล์ Exception แทนที่จะ `REWRITE` ยอดเงินตามปกติ (ถือว่าธุรกรรมนั้นถูกระงับไว้
ตรวจสอบก่อน)

**เฉลย**: แก้ไข `PROCESS-MATCHED-MASTER` ให้ตรวจสอบเงื่อนไขก่อน `REWRITE`:

```cobol
       PROCESS-MATCHED-MASTER.
           ADD 1 TO WS-MATCHED-COUNT.
           MOVE 0 TO WS-DEPOSIT-TOTAL.
           MOVE 0 TO WS-DEPOSIT-COUNT.
           PERFORM UNTIL WS-DETAIL-KEY NOT = WS-MASTER-KEY
               ADD DTL-AMOUNT TO WS-DEPOSIT-TOTAL
               ADD 1 TO WS-DEPOSIT-COUNT
               PERFORM READ-DETAIL
           END-PERFORM.

           IF WS-DEPOSIT-TOTAL > 500.00
               MOVE SPACES TO EXC-RECORD
               STRING "HOLD FOR REVIEW: " MST-ID
                   " total=" WS-DEPOSIT-TOTAL
                   DELIMITED BY SIZE
                   INTO EXC-RECORD
               WRITE EXC-RECORD
           ELSE
               ADD WS-DEPOSIT-TOTAL TO MST-BALANCE
               REWRITE MST-RECORD
           END-IF.

           MOVE MST-BALANCE TO WS-DISPLAY-BAL.
           DISPLAY MST-ID " " MST-NAME " -- " WS-DEPOSIT-COUNT
               " deposit(s), balance " WS-DISPLAY-BAL.
           PERFORM READ-MASTER.
```

ด้วยข้อมูลทดสอบเดิม A001 มียอดฝากรวม 350.50 (ไม่เกิน 500) จึงยัง `REWRITE` ตามปกติ แต่ถ้าเพิ่ม
ธุรกรรมให้ A001 มียอดฝากรวมเกิน 500 บาท รายการนั้นจะถูกเขียนลง `EXCEPTION-FILE` แทนที่จะอัปเดต
ยอดเงินทันที แสดงให้เห็นว่าโครงสร้างอัลกอริทึมนี้ขยายรองรับกฎทางธุรกิจเพิ่มเติมได้อย่างยืดหยุ่นมาก

---

## สรุปท้ายบท

ใน Part นี้เราได้สร้างอัลกอริทึม Master-Detail Matching (Balanced Line Algorithm) แบบครบวงจร:

- แนวคิดพื้นฐานของ Master File กับ Detail/Transaction File และเหตุผลที่ต้องจับคู่กัน
- ข้อกำหนดเบื้องต้นที่ขาดไม่ได้: ทั้งสองไฟล์ต้องเรียงลำดับตาม Key เดียวกัน (พิสูจน์ด้วยตัวอย่างที่
  แสดงผลลัพธ์ผิดพลาดเมื่อไม่เรียงลำดับ)
- โครงลูปพื้นฐานพร้อม Priming Read และการแยก `READ-MASTER`/`READ-DETAIL` เป็น paragraph
- การประยุกต์ `REWRITE` (จาก Part 025) เพื่ออัปเดต Master ตามข้อมูลจาก Detail จริง
- เทคนิค **HIGH-VALUES Sentinel** ที่ทำให้จัดการไฟล์หมด (EOF) ได้อย่างเรียบง่ายในตัวการเปรียบเทียบ
  Key เดียว
- การจัดการ Master ที่ไม่มี Detail ตรงกัน (Unmatched Master) และ Detail ที่ไม่มี Master รองรับ
  (Orphan Detail) ด้วย `EVALUATE TRUE` แบบ 3 ทาง
- การจัดการ Master หนึ่งตัวมี Detail หลายรายการ (One-to-Many) ด้วยลูปในที่ดูดซับ record
- **Balanced Line Algorithm ฉบับสมบูรณ์** ที่รวมทุกกรณีเข้าด้วยกัน พร้อมการแยก paragraph ตามหลัก
  Modularization
- การบันทึกรายงานข้อผิดพลาดแยกไฟล์ (Exception Report) แทนการแสดงผลหน้าจอเพียงอย่างเดียว
- โปรแกรมรวมสุดท้ายที่จำลองงาน Batch อัปเดตบัญชีลูกค้าประจำคืนแบบใกล้เคียงงานจริงที่สุด

Part ถัดไป (**Part 027**) จะเติมเต็มช่องว่างสำคัญที่ Part นี้ทิ้งไว้: **เราสมมติมาตลอดว่าไฟล์
เรียงลำดับอยู่แล้ว แต่ในโลกจริงไฟล์ที่รับเข้ามามักไม่เรียงลำดับ** เราจะเรียนรู้คำสั่ง **`SORT`**
สำหรับเรียงลำดับไฟล์ให้พร้อมก่อนนำไปใช้กับอัลกอริทึม Master-Detail ที่เพิ่งสร้างเสร็จใน Part นี้
รวมถึงคำสั่ง **`MERGE`** สำหรับรวมไฟล์ที่เรียงลำดับแล้วหลายไฟล์เข้าด้วยกันอย่างมีประสิทธิภาพ

**[← กลับไป Part 025](part-025-open-close-read-write.md)** | **[ไปยัง Part 027: SORT และ MERGE Statement →](part-027-sort-merge.md)**
