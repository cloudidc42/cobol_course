# Part 029: Relative Files (ขั้นตอนที่ 281–290)

## คำนำของ Part นี้

Part 028 แนะนำไฟล์ Indexed ที่ใช้ **RECORD KEY** (Key ที่มีความหมายทางธุรกิจ เช่น รหัสลูกค้า) ควบคู่
กับโครงสร้างดัชนี B-Tree เพื่อค้นหา record ได้อย่างรวดเร็ว Part นี้จะแนะนำไฟล์ประเภทที่สามและ
ประเภทสุดท้ายจากตารางที่เห็นครั้งแรกใน Part 023 ขั้นตอน 222: **Relative File**

ไฟล์ Relative แตกต่างจากไฟล์ Indexed ตรงที่**ไม่มี Key ทางธุรกิจและไม่มีโครงสร้างดัชนี B-Tree เลย**
แต่ใช้ **หมายเลขลำดับของ record ในไฟล์ (Relative Record Number หรือ RRN)** เป็นตัวระบุตำแหน่ง
โดยตรงแทน คิดง่าย ๆ เหมือนช่องเก็บของที่มีหมายเลขกำกับเรียงกัน (เช่น ล็อกเกอร์หมายเลข 1, 2, 3, ...)
การ "หา record ที่ 5" ก็คือการกระโดดไปที่ตำแหน่งที่ 5 ของไฟล์โดยตรงในทันที **โดยไม่ต้องผ่านการค้นหา
ในโครงสร้างดัชนีใด ๆ เลย** ทำให้การเข้าถึงเร็วกว่าไฟล์ Indexed ด้วยซ้ำในกรณีที่รู้ตำแหน่งที่ต้องการ
ล่วงหน้า (เช่น ระบบจองที่นั่งที่อ้างอิงด้วยหมายเลขที่นั่งโดยตรง หรือปฏิทินที่อ้างอิงด้วยวันที่ในปี)

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน โค้ดทุกตัวอย่าง
> ผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL แล้วทุกตัวอย่าง (รวมถึงกับดักจริงที่พิสูจน์ด้วยการทดลอง
> เช่นเดียวกับ Part 028) ไฟล์ประเภท `RELATIVE` ใช้ตัวจัดการ (handler) ที่ built-in อยู่ใน GnuCOBOL
> เสมอ **ไม่ต้องพึ่งพาไลบรารี ISAM ภายนอกเหมือนไฟล์ `INDEXED`** จึงใช้งานได้แม้ในเครื่องที่
> `cobc -info` แสดง `indexed file handler : disabled` ก็ตาม

---

## ขั้นตอนที่ 281: แนวคิด Relative Record Number (RRN) — ไม่มี Key ทางธุรกิจ

### RRN คืออะไร

**Relative Record Number (RRN)** คือหมายเลขลำดับที่ระบุตำแหน่งของ record หนึ่ง ๆ ภายในไฟล์
Relative นับเริ่มจาก **1** เสมอ (ไม่ใช่ 0) และเป็นหมายเลขที่ **ไม่มีความหมายทางธุรกิจใด ๆ เลย**
ต่างจาก `RECORD KEY` ของไฟล์ Indexed ที่มักเป็นรหัสลูกค้าหรือรหัสสินค้าที่มีความหมาย — RRN เป็นเพียง
"ตำแหน่งทางกายภาพ" ล้วน ๆ เมื่อประกาศ `ORGANIZATION IS RELATIVE` ต้องระบุ `RELATIVE KEY IS` ชี้ไปยัง
ตัวแปรใน `WORKING-STORAGE SECTION` (ไม่ใช่ field ภายใน record เหมือน `RECORD KEY` ของไฟล์ Indexed)
ที่จะใช้เก็บ/รับค่า RRN นั้น

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP281-RELATIVE-INTRO.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SEAT-FILE ASSIGN TO "SEAT281.DAT"
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS SEQUENTIAL
               RELATIVE KEY IS WS-SEAT-RRN.

       DATA DIVISION.
       FILE SECTION.
       FD  SEAT-FILE.
       01  SEAT-RECORD              PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-SEAT-RRN              PIC 9(4).
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> There is NO business key here at all - COBOL numbers each
      *> record 1, 2, 3, ... automatically as it is WRITTEN. This
      *> number is called the Relative Record Number (RRN).
           OPEN OUTPUT SEAT-FILE.
           MOVE "PASSENGER: SOMCHAI" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           MOVE "PASSENGER: SUDA" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           MOVE "PASSENGER: PRASERT" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           CLOSE SEAT-FILE.
           DISPLAY "Wrote 3 records - RRN 1, 2, 3 assigned".
           DISPLAY "automatically in write order.".

           OPEN INPUT SEAT-FILE.
           PERFORM UNTIL END-OF-FILE
               READ SEAT-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  RRN=" WS-SEAT-RRN
                           " -> " SEAT-RECORD
               END-READ
           END-PERFORM.
           CLOSE SEAT-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Wrote 3 records - RRN 1, 2, 3 assigned
automatically in write order.
  RRN=0001 -> PASSENGER: SOMCHAI
  RRN=0002 -> PASSENGER: SUDA
  RRN=0003 -> PASSENGER: PRASERT
```

### อธิบายทีละส่วน

- `RELATIVE KEY IS WS-SEAT-RRN`: `WS-SEAT-RRN` เป็นตัวแปรใน `WORKING-STORAGE SECTION` ธรรมดา
  (ไม่ใช่ field ใน `01 SEAT-RECORD`) ทำหน้าที่เป็น "ช่องสื่อสาร" ระหว่างโปรแกรมกับระบบไฟล์: เมื่อ
  `READ`/`WRITE` สำเร็จ ระบบจะใส่/อ่านค่า RRN ผ่านตัวแปรนี้เสมอ
- ในโหมด `ACCESS MODE IS SEQUENTIAL` (ค่าเริ่มต้น) และ `OPEN OUTPUT` แต่ละครั้งที่ `WRITE`
  COBOL จะกำหนด RRN ให้อัตโนมัติเรียงจาก 1 ขึ้นไปเรื่อย ๆ ตามลำดับที่ `WRITE` ถูกเรียก
- เมื่อ `READ ... NEXT RECORD` ค่า RRN ของ record ที่อ่านได้จะถูกใส่กลับเข้า `WS-SEAT-RRN`
  โดยอัตโนมัติ ทำให้เรารู้ได้ว่ากำลังอ่าน record ที่เท่าไรอยู่

### ข้อควรระวัง

- `RELATIVE KEY` **ต้องไม่ใช่ field ภายใน record** (ต่างจาก `RECORD KEY` ของไฟล์ Indexed
  ในขั้นตอนที่ 271 ที่ต้องเป็น field ภายใน record เท่านั้น) หากประกาศผิดที่จะเกิด compile error
- RRN เริ่มนับจาก **1 เสมอ ไม่ใช่ 0** ผู้ที่คุ้นเคยกับภาษาโปรแกรมอื่นที่ index อาร์เรย์เริ่มจาก 0
  (เช่น C, Python) มักสับสนจุดนี้

### แบบฝึกหัดที่ 281.1

**โจทย์**: จงอธิบายความแตกต่างสำคัญที่สุดระหว่าง `RECORD KEY` (ไฟล์ Indexed จาก Part 028) กับ
`RELATIVE KEY` (ไฟล์ Relative)

**เฉลย**: `RECORD KEY` เป็น field ที่อยู่**ภายใน record จริง** มีความหมายทางธุรกิจ (เช่น รหัสลูกค้า)
และไฟล์ต้องมีโครงสร้างดัชนี B-Tree แยกต่างหากเพื่อจับคู่ค่า Key กับตำแหน่งจริงบนดิสก์ ในขณะที่
`RELATIVE KEY` เป็นตัวแปรใน `WORKING-STORAGE` ที่**อยู่นอก record** ไม่มีความหมายทางธุรกิจใด ๆ
เป็นเพียงหมายเลขลำดับตำแหน่งทางกายภาพล้วน ๆ และไม่ต้องมีโครงสร้างดัชนีใด ๆ เลยเพราะตำแหน่งคำนวณได้
โดยตรงจากหมายเลข RRN เอง (RRN ที่ 5 ก็คือ byte offset ที่ (5-1) คูณด้วยความยาวของ 1 record)

---

## ขั้นตอนที่ 282: การเขียนกำหนดตำแหน่งเอง — ACCESS MODE IS RANDOM

### จุดเด่นที่แท้จริงของไฟล์ Relative

ขั้นตอนที่ 281 ยังให้ COBOL กำหนด RRN อัตโนมัติ (เรียงจาก 1) ซึ่งยังไม่ต่างจากไฟล์ Sequential มากนัก
ขั้นตอนนี้จะแสดงจุดเด่นที่แท้จริง: การใช้ **`ACCESS MODE IS RANDOM`** เพื่อ **เลือกตำแหน่ง (RRN) ที่
จะเขียนด้วยตนเอง** ทำให้สามารถ "เขียนลงช่องที่ 5 โดยตรง โดยไม่ต้องเขียนช่อง 1–4 ก่อนเลย" — เหมาะมาก
กับสถานการณ์อย่างระบบจองที่นั่ง ที่หมายเลขที่นั่งมีความหมายอยู่แล้วในตัวมันเอง (RRN = หมายเลขที่นั่ง)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP282-RANDOM-WRITE-SLOTS.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SEAT-FILE ASSIGN TO "SEAT282.DAT"
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS RANDOM
               RELATIVE KEY IS WS-SEAT-RRN.

       DATA DIVISION.
       FILE SECTION.
       FD  SEAT-FILE.
       01  SEAT-RECORD              PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-SEAT-RRN              PIC 9(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> With ACCESS MODE RANDOM we choose the RRN (slot number)
      *> OURSELVES before each WRITE - we are placing seat 5's
      *> passenger straight into slot 5, skipping slots 1-4
      *> entirely. This is the killer feature of RELATIVE files:
      *> the slot number IS the business meaning (seat number).
           OPEN OUTPUT SEAT-FILE.

           MOVE 5 TO WS-SEAT-RRN.
           MOVE "SEAT 5: SOMCHAI" TO SEAT-RECORD.
           WRITE SEAT-RECORD
               INVALID KEY
                   DISPLAY "Write to slot 5 failed."
           END-WRITE.

           MOVE 2 TO WS-SEAT-RRN.
           MOVE "SEAT 2: SUDA" TO SEAT-RECORD.
           WRITE SEAT-RECORD
               INVALID KEY
                   DISPLAY "Write to slot 2 failed."
           END-WRITE.

           CLOSE SEAT-FILE.
           DISPLAY "Wrote seat 5 then seat 2 - slots 1,3,4 empty.".

      *> Random READ - jump DIRECTLY to slot 5, no need to read
      *> slots 1 through 4 first.
           OPEN INPUT SEAT-FILE.
           MOVE 5 TO WS-SEAT-RRN.
           READ SEAT-FILE
               INVALID KEY
                   DISPLAY "Slot 5 is empty."
               NOT INVALID KEY
                   DISPLAY "Slot 5 holds: " SEAT-RECORD
           END-READ.

           MOVE 3 TO WS-SEAT-RRN.
           READ SEAT-FILE
               INVALID KEY
                   DISPLAY "Slot 3 is empty (as expected)."
               NOT INVALID KEY
                   DISPLAY "Slot 3 holds: " SEAT-RECORD
           END-READ.
           CLOSE SEAT-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Wrote seat 5 then seat 2 - slots 1,3,4 empty.
Slot 5 holds: SEAT 5: SOMCHAI
Slot 3 is empty (as expected).
```

### อธิบายจุดสำคัญ

- `MOVE 5 TO WS-SEAT-RRN` ก่อน `WRITE`: บอก COBOL ว่า "เขียน record นี้ลงตำแหน่งที่ 5 โดยตรง"
  ต่างจากขั้นตอนที่ 281 ที่ COBOL เลือกตำแหน่งให้เองตามลำดับการเขียน
- ช่อง (slot) ที่ไม่เคยถูกเขียนเลย (เช่น RRN 1, 3, 4 ในตัวอย่างนี้) ถือว่า **"ว่าง" หรือ "ไม่มีอยู่
  จริง"** การอ่านช่องเหล่านี้จะได้ `INVALID KEY` ทันที เหมือนกับที่เกิดกับ Key ที่ไม่มีอยู่จริงใน
  ไฟล์ Indexed (Part 028 ขั้นตอน 273)
- `INVALID KEY`/`NOT INVALID KEY` ใช้รูปแบบเดียวกันทุกประการกับไฟล์ Indexed — นี่คือเหตุผลที่ COBOL
  ออกแบบให้ไฟล์ Indexed และ Relative ใช้คำสั่งชุดเดียวกัน (`READ`, `WRITE`, `REWRITE`, `DELETE`,
  `START` พร้อม `INVALID KEY`) เพราะทั้งคู่คือการเข้าถึงแบบสุ่มผ่าน "ตัวระบุตำแหน่ง" เหมือนกัน
  ต่างกันแค่ตัวระบุตำแหน่งเป็น Key ทางธุรกิจ หรือหมายเลขลำดับเท่านั้น

### ข้อควรระวัง

- ต้องระวังไม่ให้ RRN ที่กำหนดเองเกินขอบเขตที่สมเหตุสมผล (เช่น ถ้าบัสมี 40 ที่นั่ง แต่ดันเขียน RRN
  เป็น 9999 โดยไม่ตั้งใจ) เพราะไฟล์ Relative จะจองพื้นที่ตั้งแต่ RRN 1 จนถึง RRN สูงสุดที่เคยเขียน
  เสมอ (แม้ช่องตรงกลางจะว่างก็ตาม) ทำให้ไฟล์มีขนาดใหญ่เกินจำเป็นถ้ากำหนด RRN ผิดพลาด
- `ACCESS MODE IS RANDOM` ไม่รองรับ `READ ... NEXT RECORD` เหมือนไฟล์ Indexed (Part 028 ขั้นตอน
  273) หากต้องการอ่านทั้งแบบสุ่มและเรียงลำดับสลับกัน ต้องใช้ `ACCESS MODE IS DYNAMIC` แทน

### แบบฝึกหัดที่ 282.1

**โจทย์**: จงอธิบายว่าทำไมไฟล์ `SEAT282.DAT` จึงมีขนาดใหญ่กว่าที่ควรจะเป็นถ้าคิดแค่ 2 record ที่
เขียนจริง (`"SEAT 5: SOMCHAI"` และ `"SEAT 2: SUDA"`)

**เฉลย**: เพราะไฟล์ Relative จองพื้นที่บนดิสก์ตั้งแต่ RRN 1 จนถึง RRN สูงสุดที่เคยถูกเขียนเสมอ
(ในที่นี้คือ RRN 5) แม้ตำแหน่ง 1, 3, 4 จะไม่เคยมีข้อมูลจริงก็ตาม ระบบยังคงต้องจองพื้นที่ไว้ให้
เพราะ RRN คือตำแหน่งทางกายภาพโดยตรง (คำนวณจาก byte offset) ไม่ใช่แค่รายการที่มีข้อมูลจริงเหมือน
โครงสร้างดัชนีของไฟล์ Indexed ดังนั้นไฟล์จะมีขนาดเทียบเท่ากับ 5 record เต็ม ไม่ใช่ 2 record

---

## ขั้นตอนที่ 283: การอ่านแบบเรียงลำดับข้ามช่องว่าง — READ NEXT RECORD กับไฟล์ Sparse

### พฤติกรรมของ READ NEXT RECORD เมื่อมีช่องว่าง

เมื่อไฟล์ Relative มีบาง RRN ที่ไม่เคยถูกเขียนเลย (เรียกว่าไฟล์แบบ **Sparse**) คำสั่ง
`READ ... NEXT RECORD` จะทำงานอย่างไร? ขั้นตอนนี้พิสูจน์ด้วยการทดลองจริงว่า **`READ NEXT RECORD`
จะข้ามช่องว่างไปโดยอัตโนมัติ** คืนเฉพาะ record ที่มีข้อมูลจริงเท่านั้น

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP283-SEQ-READ-WITH-GAPS.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SEAT-FILE ASSIGN TO "SEAT283.DAT"
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS DYNAMIC
               RELATIVE KEY IS WS-SEAT-RRN.

       DATA DIVISION.
       FILE SECTION.
       FD  SEAT-FILE.
       01  SEAT-RECORD              PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-SEAT-RRN              PIC 9(4).
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Occupy only seats 2, 5 and 6 out of a 6-seat bus -
      *> seats 1, 3, 4 are left completely empty (never written).
           OPEN OUTPUT SEAT-FILE.
           MOVE 2 TO WS-SEAT-RRN.
           MOVE "SEAT 2: SUDA" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           MOVE 5 TO WS-SEAT-RRN.
           MOVE "SEAT 5: SOMCHAI" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           MOVE 6 TO WS-SEAT-RRN.
           MOVE "SEAT 6: PRASERT" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           CLOSE SEAT-FILE.

      *> ACCESS MODE DYNAMIC lets the SAME file use random WRITE
      *> above and sequential READ NEXT RECORD below. READ NEXT
      *> RECORD skips slots that were never written AUTOMATICALLY,
      *> returning only the 3 occupied seats.
           OPEN INPUT SEAT-FILE.
           DISPLAY "Sequential scan of a SPARSE relative file:".
           PERFORM UNTIL END-OF-FILE
               READ SEAT-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  RRN=" WS-SEAT-RRN
                           " -> " SEAT-RECORD
               END-READ
           END-PERFORM.
           CLOSE SEAT-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Sequential scan of a SPARSE relative file:
  RRN=0002 -> SEAT 2: SUDA
  RRN=0005 -> SEAT 5: SOMCHAI
  RRN=0006 -> SEAT 6: PRASERT
```

### อธิบายจุดสำคัญ

- แม้ไฟล์จะมีขอบเขตตั้งแต่ RRN 1 ถึง 6 (ตามที่อธิบายในขั้นตอนที่ 282) แต่ `READ NEXT RECORD` จะ
  แสดงผลแค่ 3 record ที่มีข้อมูลจริงเท่านั้น (RRN 2, 5, 6) ข้ามช่องว่าง (RRN 1, 3, 4) ไปโดยอัตโนมัติ
  โดยไม่ต้องเขียนโค้ดตรวจสอบเอง
- `ACCESS MODE IS DYNAMIC` ทำให้โปรแกรมเดียวใช้ทั้ง `WRITE` แบบสุ่ม (กำหนด RRN เอง) และ
  `READ NEXT RECORD` แบบเรียงลำดับได้ในไฟล์เดียวกัน (แนวคิดเดียวกับ Part 028 ขั้นตอน 275)
- พฤติกรรมนี้เหมือนกับไฟล์ Indexed ที่อ่านแบบเรียงลำดับ (Part 028 ขั้นตอน 271) ทุกประการ: ทั้งคู่
  แสดงเฉพาะ record ที่มีอยู่จริง ไม่แสดง "ช่องว่าง" ที่ไม่เคยถูกเขียน

### ข้อควรระวัง

- อย่าเข้าใจผิดว่า `READ NEXT RECORD` จะนับ RRN ต่อเนื่องกันเสมอ (`1, 2, 3, ...`) — ค่าที่ได้ใน
  `WS-SEAT-RRN` หลัง `READ` แต่ละครั้งจะเป็น RRN จริงของ record ที่พบ (ซึ่งอาจกระโดดข้ามเลขไปได้)
  ห้ามอนุมานว่า record ถัดไปคือ RRN ปัจจุบัน + 1 เสมอไป
- หากต้องการทราบว่า "ที่นั่งใดว่าง" (ไม่ใช่แค่ "ที่นั่งใดมีคนนั่ง") การอ่านแบบเรียงลำดับนี้**ไม่บอก
  ข้อมูลนั้นให้เลย** ต้องใช้เทคนิคการอ่านแบบสุ่มไล่ทีละ RRN แทน (จะสอนในขั้นตอนที่ 289)

### แบบฝึกหัดที่ 283.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมข้างต้นจึงแสดงผลแค่ 3 บรรทัด ทั้งที่ไฟล์มีขอบเขตถึง RRN 6
(กล่าวคือ ถ้ามองในแง่ "จำนวนช่องทั้งหมด" ควรมี 6 ช่อง)

**เฉลย**: เพราะ `READ NEXT RECORD` ถูกออกแบบมาให้คืนเฉพาะ record ที่**มีข้อมูลอยู่จริง**เท่านั้น
ไม่ใช่ไล่ตามหมายเลข RRN ทุกตัวตั้งแต่ 1 ถึงค่าสูงสุด ช่องที่ไม่เคยถูก `WRITE` เลย (RRN 1, 3, 4) จะ
ถูกข้ามไปโดยสมบูรณ์ในมุมมองของการอ่านแบบเรียงลำดับ เสมือนว่าช่องเหล่านั้น "ไม่มีอยู่" ในไฟล์เลย
แม้พื้นที่ทางกายภาพจะถูกจองไว้แล้วก็ตาม (ตามที่อธิบายในแบบฝึกหัด 282.1)

---

## ขั้นตอนที่ 284: START Statement บนไฟล์ Relative

### แนวคิดเหมือน Indexed แต่ใช้ตัวเลขแทน Key

`START` บนไฟล์ Relative ทำงานแบบเดียวกับไฟล์ Indexed (Part 028 ขั้นตอน 274) ทุกประการ เพียงแต่
เปรียบเทียบด้วยค่า **RRN** แทนค่า Key ทางธุรกิจ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP284-START-RELATIVE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SEAT-FILE ASSIGN TO "SEAT284.DAT"
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS DYNAMIC
               RELATIVE KEY IS WS-SEAT-RRN.

       DATA DIVISION.
       FILE SECTION.
       FD  SEAT-FILE.
       01  SEAT-RECORD              PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-SEAT-RRN              PIC 9(4).
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-FILE          VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT SEAT-FILE.
           MOVE 1 TO WS-SEAT-RRN. MOVE "SEAT 1" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           MOVE 3 TO WS-SEAT-RRN. MOVE "SEAT 3" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           MOVE 4 TO WS-SEAT-RRN. MOVE "SEAT 4" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           MOVE 7 TO WS-SEAT-RRN. MOVE "SEAT 7" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           CLOSE SEAT-FILE.

      *> START jumps the cursor to a given RRN without reading it.
      *> Useful to skip straight to "seat 4 onward" in a report,
      *> for example, without scanning seats 1-3 first.
           OPEN INPUT SEAT-FILE.
           MOVE 4 TO WS-SEAT-RRN.
           START SEAT-FILE KEY IS >= WS-SEAT-RRN
               INVALID KEY
                   DISPLAY "No seat at or after 4."
           END-START.

           DISPLAY "From seat 4 onward:".
           PERFORM UNTIL END-OF-FILE
               READ SEAT-FILE NEXT RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  RRN=" WS-SEAT-RRN
                           " -> " SEAT-RECORD
               END-READ
           END-PERFORM.
           CLOSE SEAT-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
From seat 4 onward:
  RRN=0004 -> SEAT 4
  RRN=0007 -> SEAT 7
```

### อธิบายจุดสำคัญ

- `START SEAT-FILE KEY IS >= WS-SEAT-RRN`: เลื่อนตำแหน่งไปยัง record แรกที่มี RRN **มากกว่าหรือ
  เท่ากับ** 4 (ในที่นี้คือ RRN 4 พอดี เพราะมีอยู่จริง) แล้ว `READ NEXT RECORD` จะไล่อ่านต่อจากตรงนั้น
- สังเกตว่า RRN 3 (ซึ่งน้อยกว่า 4) และ RRN 1, 2 ไม่ปรากฏในผลลัพธ์เลย เพราะ `START` เลื่อนตำแหน่ง
  ผ่านมันไปแล้ว
- เงื่อนไขที่ใช้กับ `START` ของไฟล์ Relative มีครบเหมือนไฟล์ Indexed: `=`, `>`, `>=`, `<`, `<=`

### ข้อควรระวัง

- ค่าที่ `MOVE` เข้า `WS-SEAT-RRN` ก่อน `START` ต้องเป็นตัวเลข RRN ล้วน ๆ ไม่ใช่ค่าทางธุรกิจใด ๆ
  (ต่างจากไฟล์ Indexed ที่ `START` มักใช้ค่าที่มีความหมาย เช่น รหัสลูกค้าหรือชื่อเมือง)
- `START` บนไฟล์ Relative มีประโยชน์มากเมื่อรู้ตำแหน่งเป้าหมายที่ต้องการอยู่แล้ว (เช่น รายงานที่
  ต้องการเริ่มพิมพ์จากที่นั่งกลุ่มหลัง) แต่ถ้าไม่ทราบตำแหน่งที่แน่นอน การไล่อ่านจากต้นไฟล์ปกติอาจ
  เขียนโค้ดง่ายกว่า

### แบบฝึกหัดที่ 284.1

**โจทย์**: จงแก้ไข `START` ในโปรแกรมข้างต้นให้เริ่มอ่านจากที่นั่งที่ **มากกว่า** 3 (ไม่รวมที่นั่ง 3)
แทนที่จะเป็น "ตั้งแต่ 4 เป็นต้นไป" แบบเดิม

**เฉลย**: เปลี่ยนค่าและเงื่อนไขเป็น:

```cobol
           MOVE 3 TO WS-SEAT-RRN.
           START SEAT-FILE KEY IS > WS-SEAT-RRN
               INVALID KEY
                   DISPLAY "No seat after 3."
           END-START.
```

ผลลัพธ์จะเหมือนเดิมทุกประการ (`RRN=4` แล้วตามด้วย `RRN=7`) เพราะ "มากกว่า 3" กับ "ตั้งแต่ 4 เป็น
ต้นไป" ให้ผลเดียวกันในกรณีนี้ที่ RRN เป็นจำนวนเต็ม แต่แนวคิดเรื่องเงื่อนไข `>` กับ `>=` ยังคงต่างกัน
ในเชิงความหมายของโค้ดและสำคัญมากเมื่อใช้กับ Key ที่ไม่ใช่จำนวนเต็ม (เช่น Key ตัวอักษรของไฟล์ Indexed)

---

## ขั้นตอนที่ 285: REWRITE บนไฟล์ Relative

### พฤติกรรมเหมือนไฟล์ Indexed ทุกประการ

`REWRITE` บนไฟล์ Relative ปลอดภัย 100% เหมือนไฟล์ Indexed (Part 028 ขั้นตอน 276) เพราะเก็บข้อมูล
เป็น record ความยาวคงที่แบบไบนารีเช่นเดียวกัน ไม่มีปัญหาการตัดช่องว่างท้ายบรรทัดแบบ `LINE
SEQUENTIAL` เลย

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP285-REWRITE-RELATIVE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SEAT-FILE ASSIGN TO "SEAT285.DAT"
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS DYNAMIC
               RELATIVE KEY IS WS-SEAT-RRN.

       DATA DIVISION.
       FILE SECTION.
       FD  SEAT-FILE.
       01  SEAT-RECORD              PIC X(25).

       WORKING-STORAGE SECTION.
       01  WS-SEAT-RRN              PIC 9(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT SEAT-FILE.
           MOVE 1 TO WS-SEAT-RRN.
           MOVE "SEAT 1: EMPTY" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           CLOSE SEAT-FILE.

      *> REWRITE on a RELATIVE file just needs the RRN to already
      *> be occupied - it overwrites that exact slot directly.
      *> Just like INDEXED files, records are fixed-length, so
      *> there is no trailing-space trimming risk at all.
           OPEN I-O SEAT-FILE.
           MOVE 1 TO WS-SEAT-RRN.
           READ SEAT-FILE
               INVALID KEY
                   DISPLAY "Seat 1 not found."
               NOT INVALID KEY
                   MOVE "SEAT 1: SOMCHAI JAIDEE" TO SEAT-RECORD
                   REWRITE SEAT-RECORD
                       INVALID KEY
                           DISPLAY "Rewrite failed."
                   END-REWRITE
                   DISPLAY "Seat 1 updated."
           END-READ.
           CLOSE SEAT-FILE.

           OPEN INPUT SEAT-FILE.
           MOVE 1 TO WS-SEAT-RRN.
           READ SEAT-FILE.
           DISPLAY "Confirmed on disk: " SEAT-RECORD.
           CLOSE SEAT-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Seat 1 updated.
Confirmed on disk: SEAT 1: SOMCHAI JAIDEE
```

### อธิบายจุดสำคัญ

- ลำดับการทำงานเหมือนกับ `REWRITE` บนไฟล์ Indexed ทุกประการ: `READ` มาก่อน แก้ค่าใน record area
  แล้วเรียก `REWRITE`
- ต่างจากไฟล์ Indexed ตรงที่ `REWRITE` บนไฟล์ Relative **ไม่มีข้อจำกัดเรื่อง Key ซ้ำ** เพราะไม่มี
  Key ทางธุรกิจให้ต้องรักษาความ unique เลย เพียงแค่ RRN ต้องเป็นตำแหน่งที่มีข้อมูลอยู่แล้วเท่านั้น
- ถ้าต้องการ "ย้าย" record ไปยัง RRN อื่น ต้องใช้ `DELETE` ตำแหน่งเดิมแล้ว `WRITE` ใหม่ที่ RRN ใหม่
  เหมือนกับหลักการของไฟล์ Indexed ทุกประการ (`REWRITE` แก้ไขได้แค่ "เนื้อหา" ไม่ใช่ "ตำแหน่ง")

### ข้อควรระวัง

- แม้ RRN จะเป็นแค่ตัวเลข แต่ก็ยังต้อง `READ` มาก่อนเสมอเช่นเดียวกับไฟล์ Indexed ก่อนที่จะ
  `REWRITE` ได้ — กฎนี้ใช้เหมือนกันทั้งไฟล์ Indexed และ Relative ไม่มีข้อยกเว้น
- ต้องเปิดไฟล์ด้วย `OPEN I-O` เท่านั้น (กฎเดิมจาก Part 025 ขั้นตอน 241 ที่ใช้กับไฟล์ทุกประเภท)

### แบบฝึกหัดที่ 285.1

**โจทย์**: จงอธิบายว่าทำไม `REWRITE` บนไฟล์ Relative จึงไม่มีความเสี่ยงเรื่อง "Key ซ้ำ" เหมือนที่
เคยพิสูจน์ไว้กับ `WRITE` บนไฟล์ Indexed ใน Part 028 ขั้นตอนที่ 278

**เฉลย**: เพราะไฟล์ Relative ไม่มี `RECORD KEY` ทางธุรกิจที่ต้องคงความไม่ซ้ำกันเลย ตัวระบุตำแหน่ง
เดียวที่มีคือ RRN ซึ่งเป็นแค่หมายเลขลำดับทางกายภาพ การ `REWRITE` เพียงแค่เขียนทับเนื้อหาที่ตำแหน่ง
เดิมที่มีอยู่แล้ว ไม่ได้สร้างตำแหน่งใหม่หรือกระทบ RRN ของ record อื่นเลย จึงไม่มีแนวคิดเรื่อง
"ค่าซ้ำกัน" ให้ต้องกังวลแบบที่เกิดกับ `RECORD KEY` ของไฟล์ Indexed

---

## ขั้นตอนที่ 286: DELETE บนไฟล์ Relative และการนำช่องว่างกลับมาใช้ใหม่

### DELETE ทำงานได้จริงเหมือนไฟล์ Indexed

`DELETE` บนไฟล์ Relative ทำงานได้จริงเช่นเดียวกับไฟล์ Indexed (Part 028 ขั้นตอน 277) ต่างจากไฟล์
`LINE SEQUENTIAL` ที่ทำไม่ได้เลย และที่สำคัญคือ **ช่อง (RRN) ที่ถูกลบแล้วสามารถนำกลับมาเขียนใหม่ได้
ทันที** — เหมาะมากกับสถานการณ์ "ลูกค้ายกเลิกที่นั่ง แล้วมีคนอื่นมาจองที่นั่งเดิมต่อ"

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP286-DELETE-RELATIVE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SEAT-FILE ASSIGN TO "SEAT286.DAT"
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS DYNAMIC
               RELATIVE KEY IS WS-SEAT-RRN.

       DATA DIVISION.
       FILE SECTION.
       FD  SEAT-FILE.
       01  SEAT-RECORD              PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-SEAT-RRN              PIC 9(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT SEAT-FILE.
           MOVE 1 TO WS-SEAT-RRN. MOVE "SEAT 1: SOMCHAI" TO
               SEAT-RECORD.
           WRITE SEAT-RECORD.
           MOVE 2 TO WS-SEAT-RRN. MOVE "SEAT 2: SUDA" TO
               SEAT-RECORD.
           WRITE SEAT-RECORD.
           CLOSE SEAT-FILE.

      *> DELETE on a RELATIVE file, like INDEXED files (Part 028,
      *> step 277), actually works - unlike LINE SEQUENTIAL. The
      *> slot becomes free again and can be WRITTEN to later - a
      *> perfect model for "customer cancels seat 2".
           OPEN I-O SEAT-FILE.
           MOVE 2 TO WS-SEAT-RRN.
           DELETE SEAT-FILE RECORD
               INVALID KEY
                   DISPLAY "Delete failed."
               NOT INVALID KEY
                   DISPLAY "Seat 2 cancelled - slot freed."
           END-DELETE.
           CLOSE SEAT-FILE.

           OPEN INPUT SEAT-FILE.
           MOVE 2 TO WS-SEAT-RRN.
           READ SEAT-FILE
               INVALID KEY
                   DISPLAY "Seat 2 confirmed empty."
               NOT INVALID KEY
                   DISPLAY "Still occupied?! " SEAT-RECORD
           END-READ.
           CLOSE SEAT-FILE.

      *> A new passenger books the now-empty seat 2 again.
           OPEN I-O SEAT-FILE.
           MOVE 2 TO WS-SEAT-RRN.
           MOVE "SEAT 2: WANNA" TO SEAT-RECORD.
           WRITE SEAT-RECORD
               INVALID KEY
                   DISPLAY "Rebooking seat 2 failed."
               NOT INVALID KEY
                   DISPLAY "Seat 2 rebooked: " SEAT-RECORD
           END-WRITE.
           CLOSE SEAT-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Seat 2 cancelled - slot freed.
Seat 2 confirmed empty.
Seat 2 rebooked: SEAT 2: WANNA
```

### อธิบายจุดสำคัญ

- **ทดสอบยืนยันแล้ว**: บนไฟล์ Relative ที่เปิดด้วย `ACCESS MODE IS DYNAMIC` คำสั่ง
  `DELETE SEAT-FILE RECORD` **ไม่ต้อง `READ` มาก่อนเลย** เพียงแค่ `MOVE` ค่า RRN ที่ต้องการลบเข้า
  `WS-SEAT-RRN` แล้วเรียก `DELETE` ได้ทันที (เหมือนกับไฟล์ Indexed ในโหมด `RANDOM`/`DYNAMIC` ตามที่
  แก้ไขความเข้าใจไว้ใน Part 028 ขั้นตอน 277)
- หลัง `DELETE` ตำแหน่ง RRN 2 จะกลับสู่สถานะ "ว่าง" ทันที และสามารถ `WRITE` ข้อมูลใหม่ลงตำแหน่ง
  เดิมนั้นได้อีกครั้งโดยไม่มีปัญหาใด ๆ (ต่างจากไฟล์ Indexed ที่แม้จะ `DELETE` แล้วก็ยัง `WRITE`
  Key เดิมซ้ำได้เหมือนกัน แต่แนวคิดเรื่อง "นำตำแหน่งกลับมาใช้ใหม่" ชัดเจนกว่ามากในไฟล์ Relative
  เพราะ RRN คือตำแหน่งทางกายภาพโดยตรง)

### ข้อควรระวัง

- การลบแล้วเขียนใหม่ที่ RRN เดิมซ้ำ ๆ กันบ่อยครั้ง (เช่น ระบบจองที่นั่งที่มีการยกเลิก-จองใหม่
  ตลอดเวลา) เป็นรูปแบบการใช้งานปกติของไฟล์ Relative และไม่ก่อให้เกิดปัญหาด้าน performance ใด ๆ
  เพราะเป็นการเขียนทับตำแหน่งเดิมโดยตรง ไม่ต้องปรับปรุงโครงสร้างดัชนีใด ๆ เหมือนไฟล์ Indexed
- อย่าลืมว่าการ `DELETE` เป็นการลบถาวรทันทีเช่นเดียวกับไฟล์ Indexed ไม่มี UNDO ให้ใช้

### แบบฝึกหัดที่ 286.1

**โจทย์**: จงอธิบายว่าทำไมการนำ RRN ที่ถูก `DELETE` แล้วกลับมาใช้ใหม่จึงเป็นเรื่องปกติและไม่มีปัญหา
ใด ๆ ในไฟล์ Relative ในขณะที่การ "นำ Key เดิมกลับมาใช้ใหม่" ในไฟล์ Indexed (หลัง `DELETE`) ต้อง
พิจารณาให้รอบคอบกว่าในระบบธุรกิจจริง

**เฉลย**: เพราะ RRN ไม่มีความหมายทางธุรกิจใด ๆ เลย เป็นเพียงหมายเลขตำแหน่งทางกายภาพ การใช้ RRN
ซ้ำจึงไม่กระทบความถูกต้องของข้อมูลทางธุรกิจแต่อย่างใด (ที่นั่งหมายเลข 2 ก็ยังคงหมายถึง "ที่นั่งที่ 2
บนรถบัสคันนี้" เสมอ ไม่ว่าใครจะนั่งอยู่) แต่ `RECORD KEY` ของไฟล์ Indexed มักมีความหมายทางธุรกิจ
ที่ผูกกับข้อมูลในโลกจริง (เช่น รหัสลูกค้า) การนำรหัสเดิมที่เคยเป็นของลูกค้าคนหนึ่งกลับมาใช้กับ
ลูกค้าคนใหม่อาจสร้างความสับสนหรือปัญหาการตรวจสอบย้อนหลัง (Audit Trail) ในระบบธุรกิจจริงได้ จึงต้อง
พิจารณานโยบายการนำ Key กลับมาใช้ใหม่อย่างรอบคอบกว่ามาก

---

## ขั้นตอนที่ 287: ACCESS MODE IS DYNAMIC — ผสมการค้นหาแบบสุ่มกับการวนหาช่องว่าง

### โจทย์จริง: หาที่นั่งว่างช่องแรกโดยอัตโนมัติ

ในสถานการณ์จริง ผู้โดยสารมักไม่ระบุหมายเลขที่นั่งเอง แต่ต้องการ **"ที่นั่งว่างช่องแรกที่มี"**
ขั้นตอนนี้แสดงเทคนิคการไล่ตรวจสอบทีละ RRN ด้วยการอ่านแบบสุ่ม (ไม่ใช่ `READ NEXT RECORD`) จนกว่าจะ
เจอ `INVALID KEY` (แปลว่าว่าง) แล้วจอง (WRITE) ทันทีโดยใช้ `ACCESS MODE IS DYNAMIC` ในเซสชันเดียว

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP287-DYNAMIC-FIND-SLOT.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SEAT-FILE ASSIGN TO "SEAT287.DAT"
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS DYNAMIC
               RELATIVE KEY IS WS-SEAT-RRN.

       DATA DIVISION.
       FILE SECTION.
       FD  SEAT-FILE.
       01  SEAT-RECORD              PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-SEAT-RRN              PIC 9(4).
       01  WS-FOUND-FLAG            PIC X VALUE "N".
           88  SLOT-FOUND           VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT SEAT-FILE.
           MOVE 1 TO WS-SEAT-RRN. MOVE "SEAT 1: SOMCHAI" TO
               SEAT-RECORD.
           WRITE SEAT-RECORD.
           MOVE 3 TO WS-SEAT-RRN. MOVE "SEAT 3: SUDA" TO
               SEAT-RECORD.
           WRITE SEAT-RECORD.
           CLOSE SEAT-FILE.

      *> ACCESS MODE DYNAMIC lets us combine a RANDOM probe loop
      *> (trying RRN 1, 2, 3, ... one at a time with plain READ)
      *> with a RANDOM WRITE the moment we find an empty slot -
      *> both use the SAME OPEN I-O session.
           OPEN I-O SEAT-FILE.
           MOVE 1 TO WS-SEAT-RRN.
           PERFORM UNTIL SLOT-FOUND
               READ SEAT-FILE
                   INVALID KEY
                       SET SLOT-FOUND TO TRUE
                       DISPLAY "Seat " WS-SEAT-RRN
                           " is empty - booking it."
                   NOT INVALID KEY
                       DISPLAY "Seat " WS-SEAT-RRN
                           " occupied: " SEAT-RECORD
                       ADD 1 TO WS-SEAT-RRN
               END-READ
           END-PERFORM.

           MOVE "SEAT: NEW PASSENGER" TO SEAT-RECORD.
           WRITE SEAT-RECORD
               INVALID KEY
                   DISPLAY "Booking failed."
               NOT INVALID KEY
                   DISPLAY "Booked seat " WS-SEAT-RRN
                       " successfully."
           END-WRITE.
           CLOSE SEAT-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Seat 0001 occupied: SEAT 1: SOMCHAI
Seat 0002 is empty - booking it.
Booked seat 0002 successfully.
```

### อธิบายจุดสำคัญ

- ลูปนี้เริ่มจาก `WS-SEAT-RRN = 1` แล้วไล่ `READ` (แบบสุ่ม ไม่ใช่ `NEXT RECORD`) ทีละตำแหน่ง:
  ถ้าเจอ `NOT INVALID KEY` (มีคนนั่งแล้ว) จะ `ADD 1` แล้วลองตำแหน่งถัดไป ถ้าเจอ `INVALID KEY`
  (ว่าง) จะหยุดลูปทันทีด้วย `SET SLOT-FOUND TO TRUE`
- เมื่อพบตำแหน่งว่างแล้ว ค่า `WS-SEAT-RRN` ยังคงค้างอยู่ที่ตำแหน่งนั้น (RRN 2) ทำให้ `WRITE` ถัดมา
  เขียนลงตำแหน่งที่ถูกต้องพอดีโดยไม่ต้องตั้งค่าซ้ำ
- นี่คือรูปแบบการเขียนโค้ดที่**ใช้ได้จริง**ในระบบจองที่นั่ง/ห้อง/คิวจริง ๆ เพราะเป็นการค้นหา
  O(n) ในกรณีเลวร้ายที่สุด (ไล่ทุกที่นั่ง) แต่ในทางปฏิบัติจะเจอที่ว่างเร็วมากถ้าที่นั่งส่วนใหญ่ว่าง

### ข้อควรระวัง

- ถ้าที่นั่งเต็มทั้งหมด ลูปนี้จะไล่ RRN ไปเรื่อย ๆ ไม่มีที่สิ้นสุด (ไม่มีเงื่อนไขหยุดเมื่อเกิน
  จำนวนที่นั่งสูงสุด) ในโปรแกรมใช้งานจริงต้องเพิ่มเงื่อนไข `OR WS-SEAT-RRN > WS-MAX-SEATS` ใน
  `PERFORM UNTIL` เพื่อป้องกันลูปไม่รู้จบ (ดูตัวอย่างที่แก้ไขปัญหานี้แล้วในขั้นตอนที่ 290)
- เทคนิคนี้เหมาะกับไฟล์ที่ไม่ใหญ่มากนัก (ที่นั่งหลักสิบถึงหลักร้อย) ถ้าต้องจัดการ "ที่ว่าง" ของ
  ข้อมูลนับล้านตำแหน่ง ควรพิจารณาโครงสร้างข้อมูลเพิ่มเติม (เช่น bitmap ของที่ว่าง) แทนการไล่ทีละ
  ตำแหน่งแบบนี้

### แบบฝึกหัดที่ 287.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมข้างต้นจึงใช้ `READ` (แบบสุ่ม) แทนที่จะใช้ `READ NEXT RECORD` ใน
การไล่หาที่นั่งว่าง

**เฉลย**: เพราะ `READ NEXT RECORD` คืนค่าเฉพาะ record ที่**มีข้อมูลอยู่จริง**เท่านั้น (ตามที่พิสูจน์
ในขั้นตอนที่ 283) มันจะข้ามที่นั่งว่างไปโดยอัตโนมัติ ทำให้ไม่มีทางรู้เลยว่าที่นั่งไหน "ว่าง" ผ่าน
`READ NEXT RECORD` ในขณะที่ `READ` แบบสุ่มด้วย RRN ที่กำหนดเอง จะตรวจสอบตำแหน่งที่ระบุตรง ๆ ว่า
มีข้อมูลอยู่หรือไม่ (`INVALID KEY` แปลว่าว่าง) ทำให้สามารถไล่ตรวจทีละตำแหน่งเพื่อหา "ช่องว่าง" ได้
อย่างแม่นยำ

---

## ขั้นตอนที่ 288: คำนวณ RRN จาก Business Key โดยตรง — การเข้าถึงแบบ O(1) แท้จริง

### เมื่อ Business Key แปลงเป็น RRN ได้ด้วยสูตรคณิตศาสตร์

จุดแข็งที่สุดของไฟล์ Relative จะเปล่งประกายเต็มที่เมื่อ Business Key ของข้อมูลสามารถ **แปลงเป็น
RRN ได้ด้วยสูตรคณิตศาสตร์ง่าย ๆ** เช่น ถ้ารหัสสินค้าวิ่งต่อเนื่องตั้งแต่ 1000–1999 เราสามารถคำนวณ
`RRN = รหัสสินค้า - 1000 + 1` ได้ทันที ทำให้การค้นหาไม่ต้องผ่านโครงสร้างดัชนี B-Tree เหมือนไฟล์
Indexed เลยแม้แต่ขั้นตอนเดียว — เป็นการเข้าถึงแบบ **O(1)** ที่แท้จริง (คำนวณตำแหน่งแล้วกระโดดไปอ่าน
ตรง ๆ)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP288-COMPUTED-RRN.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PRODUCT-FILE ASSIGN TO "PROD288.DAT"
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS DYNAMIC
               RELATIVE KEY IS WS-PRODUCT-RRN.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-FILE.
       01  PRODUCT-RECORD.
           05  PR-CODE              PIC 9(4).
           05  PR-NAME              PIC X(15).

       WORKING-STORAGE SECTION.
       01  WS-PRODUCT-RRN           PIC 9(4).
       01  WS-LOOKUP-CODE           PIC 9(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Product codes always run 1000-1999. Instead of relying on
      *> an index engine to find them (like INDEXED files do), we
      *> COMPUTE the slot number directly from the business key -
      *> this is a pure O(1) lookup with ZERO index overhead.
           OPEN OUTPUT PRODUCT-FILE.

           MOVE 1000 TO WS-LOOKUP-CODE.
           COMPUTE WS-PRODUCT-RRN = WS-LOOKUP-CODE - 1000 + 1.
           MOVE WS-LOOKUP-CODE TO PR-CODE.
           MOVE "RICE BAG 5KG" TO PR-NAME.
           WRITE PRODUCT-RECORD.

           MOVE 1050 TO WS-LOOKUP-CODE.
           COMPUTE WS-PRODUCT-RRN = WS-LOOKUP-CODE - 1000 + 1.
           MOVE WS-LOOKUP-CODE TO PR-CODE.
           MOVE "COOKING OIL 1L" TO PR-NAME.
           WRITE PRODUCT-RECORD.

           CLOSE PRODUCT-FILE.
           DISPLAY "Product 1000 -> slot 1, product 1050 -> slot 51".

      *> Looking up product 1050 needs ONE arithmetic step and ONE
      *> direct disk read - no B-Tree traversal at all.
           OPEN INPUT PRODUCT-FILE.
           MOVE 1050 TO WS-LOOKUP-CODE.
           COMPUTE WS-PRODUCT-RRN = WS-LOOKUP-CODE - 1000 + 1.
           READ PRODUCT-FILE
               INVALID KEY
                   DISPLAY "Product 1050 not found."
               NOT INVALID KEY
                   DISPLAY "Found directly at slot "
                       WS-PRODUCT-RRN ": " PR-NAME
           END-READ.
           CLOSE PRODUCT-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Product 1000 -> slot 1, product 1050 -> slot 51
Found directly at slot 0051: COOKING OIL 1L
```

### อธิบายจุดสำคัญ

- `COMPUTE WS-PRODUCT-RRN = WS-LOOKUP-CODE - 1000 + 1`: สูตรแปลงรหัสสินค้า (1000–1999) ให้เป็น
  RRN (1–1000) เป็นสูตรเชิงเส้นง่าย ๆ ที่ใช้ทรัพยากรคำนวณแทบไม่มีนัยสำคัญเลย
- เก็บ `PR-CODE` (รหัสสินค้าจริง) ไว้ในตัว record ด้วย แม้จะไม่ได้ใช้เป็น Key ในการค้นหา เพราะเมื่อ
  อ่าน record ออกมาแล้ว เรายังต้องการทราบว่า "รหัสสินค้าจริงคืออะไร" (RRN เพียงอย่างเดียวไม่บอก
  ข้อมูลทางธุรกิจ)
- เทคนิคนี้เร็วกว่าไฟล์ Indexed ในทางทฤษฎี เพราะไฟล์ Indexed ยังต้องเสียเวลาค้นหาในโครงสร้าง
  B-Tree (แม้จะเร็วระดับ `O(log n)` ก็ตาม) ในขณะที่การคำนวณ RRN ตรงเป็นการเข้าถึงแบบ `O(1)`
  แท้จริงที่ไม่ต้องค้นหาอะไรเลย

### ข้อควรระวัง

- เทคนิคนี้ใช้ได้ก็ต่อเมื่อ Business Key มีคุณสมบัติ **ต่อเนื่องและมีขอบเขตที่ทราบล่วงหน้า**
  (เช่น รหัส 1000–1999 พอดี) ถ้ารหัสสินค้ากระโดดไปมาแบบไม่มีรูปแบบ (เช่น `"A7X92"`, `"ZZ001"`)
  จะไม่สามารถแปลงเป็นสูตรคณิตศาสตร์ง่าย ๆ ได้ ต้องกลับไปใช้ไฟล์ Indexed แทน
- ถ้าธุรกิจในอนาคตขยายรหัสสินค้าเกิน 1999 (เช่น กลายเป็น 2500) โปรแกรมต้องแก้สูตรคำนวณ และไฟล์อาจ
  ต้องขยายขนาดเพื่อรองรับ RRN ที่มากขึ้นตามไปด้วย เป็นข้อจำกัดด้านความยืดหยุ่นที่ต้องแลกกับ
  ความเร็วที่ได้มา

### แบบฝึกหัดที่ 288.1

**โจทย์**: ถ้ารหัสพนักงานวิ่งตั้งแต่ 5000–5499 จงเขียนสูตร `COMPUTE` ที่แปลงรหัสพนักงานเป็น RRN
ที่เริ่มจาก 1

**เฉลย**:

```cobol
           COMPUTE WS-EMPLOYEE-RRN = WS-EMPLOYEE-CODE - 5000 + 1.
```

รหัส 5000 จะได้ RRN = 1, รหัส 5499 จะได้ RRN = 500 ครอบคลุมพนักงานทั้งหมด 500 คนพอดีในช่วง RRN
1 ถึง 500

---

## ขั้นตอนที่ 289: รายงานผังที่นั่งแบบเต็ม — ข้อจำกัดที่ต้องแลกมากับความเร็ว

### ปัญหา: ไฟล์ Relative ไม่มี "สารบัญ" ของช่องว่าง

ขั้นตอนนี้จะแสดงให้เห็น**ข้อจำกัดที่สำคัญที่สุด**ของไฟล์ Relative เมื่อเทียบกับไฟล์ Indexed:
**ไม่มีวิธีใดจะ "แสดงรายการที่นั่งว่างทั้งหมด" ได้โดยตรง** ต้องไล่ตรวจสอบทุกตำแหน่งตั้งแต่ 1 จนถึง
ค่าสูงสุดด้วยการอ่านแบบสุ่มทีละตำแหน่งเท่านั้น (ตรงข้ามกับการอ่านแบบเรียงลำดับในขั้นตอนที่ 283 ที่
ข้ามที่ว่างไปหมด)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP289-SEAT-MAP-REPORT.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SEAT-FILE ASSIGN TO "SEAT289.DAT"
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS DYNAMIC
               RELATIVE KEY IS WS-SEAT-RRN.

       DATA DIVISION.
       FILE SECTION.
       FD  SEAT-FILE.
       01  SEAT-RECORD              PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-SEAT-RRN              PIC 9(4).
       01  WS-TOTAL-SEATS           PIC 9(4) VALUE 10.
       01  WS-OCCUPIED-COUNT        PIC 9(4) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT SEAT-FILE.
           MOVE 1 TO WS-SEAT-RRN. MOVE "SOMCHAI" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           MOVE 4 TO WS-SEAT-RRN. MOVE "SUDA" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           MOVE 5 TO WS-SEAT-RRN. MOVE "PRASERT" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           MOVE 9 TO WS-SEAT-RRN. MOVE "WANNA" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           CLOSE SEAT-FILE.

      *> A RELATIVE file has NO built-in way to list "which slots
      *> exist" other than a full sequential scan (which SKIPS
      *> empty slots, step 283) or, as shown here, probing every
      *> possible RRN one by one with a RANDOM READ. This is the
      *> classic trade-off: RELATIVE gives blazing-fast direct
      *> access, but no free "table of contents" of empty slots.
           OPEN INPUT SEAT-FILE.
           DISPLAY "---- Full seat map (1 to 10) ----".
           MOVE 1 TO WS-SEAT-RRN.
           PERFORM WS-TOTAL-SEATS TIMES
               READ SEAT-FILE
                   INVALID KEY
                       DISPLAY "  Seat " WS-SEAT-RRN
                           ": available"
                   NOT INVALID KEY
                       DISPLAY "  Seat " WS-SEAT-RRN
                           ": " SEAT-RECORD
                       ADD 1 TO WS-OCCUPIED-COUNT
               END-READ
               ADD 1 TO WS-SEAT-RRN
           END-PERFORM.
           CLOSE SEAT-FILE.

           DISPLAY "Occupied: " WS-OCCUPIED-COUNT " out of "
               WS-TOTAL-SEATS.
           STOP RUN.
```

**ผลลัพธ์:**

```
---- Full seat map (1 to 10) ----
  Seat 0001: SOMCHAI
  Seat 0002: available
  Seat 0003: available
  Seat 0004: SUDA
  Seat 0005: PRASERT
  Seat 0006: available
  Seat 0007: available
  Seat 0008: available
  Seat 0009: WANNA
  Seat 0010: available
Occupied: 0004 out of 0010
```

### อธิบายจุดสำคัญ

- `PERFORM WS-TOTAL-SEATS TIMES` ร่วมกับ `ADD 1 TO WS-SEAT-RRN` ท้ายลูป: ไล่ตรวจสอบทุกตำแหน่ง
  ตั้งแต่ 1 ถึง 10 ด้วยการอ่านแบบสุ่มทีละตำแหน่ง (ไม่ใช่ `READ NEXT RECORD`) ทำให้เห็นครบทั้งที่
  นั่งที่มีคนนั่งและที่นั่งว่าง
- นี่คือ**การแลกเปลี่ยน (Trade-off) พื้นฐาน**ของไฟล์ Relative: ได้ความเร็วในการเข้าถึงแบบสุ่มที่
  รู้ตำแหน่งอยู่แล้วสูงสุด แต่ต้อง "จ่ายราคา" ด้วยการไม่มีวิธีลัดในการแจกแจงรายการช่องว่างทั้งหมด
  (ไฟล์ Indexed ทำสิ่งนี้ได้ง่ายกว่าด้วย `READ NEXT RECORD` ธรรมดา เพราะมันมีแนวคิดว่า "ไม่มี
  record ที่ไม่สมบูรณ์" อยู่แล้วในตัว)
- จำนวนครั้งที่ต้อง `READ` เท่ากับจำนวนที่นั่งทั้งหมดเสมอ (`WS-TOTAL-SEATS`) ไม่ว่าจะมีคนนั่งกี่คน
  ก็ตาม ต่างจากการอ่านแบบเรียงลำดับที่จำนวนครั้งเท่ากับจำนวน record ที่มีข้อมูลจริงเท่านั้น

### ข้อควรระวัง

- ถ้าจำนวนที่นั่งทั้งหมด (`WS-TOTAL-SEATS`) มีค่ามาก (เช่น หลักหมื่นหลักแสน) การไล่อ่านแบบนี้จะ
  ช้าลงตามสัดส่วน ควรพิจารณาว่าจำเป็นต้อง "แสดงผังเต็ม" บ่อยแค่ไหนในระบบจริง หรือจะเก็บสถิติ
  จำนวนที่ว่างไว้แยกต่างหาก (เช่น ตัวนับใน record พิเศษ) แทนการไล่คำนวณใหม่ทุกครั้ง
- ต้องรู้จำนวนที่นั่งทั้งหมด (`WS-TOTAL-SEATS`) ล่วงหน้าเสมอ เพราะไฟล์ Relative ไม่มีวิธีถามว่า
  "ไฟล์นี้มีกี่ตำแหน่งทั้งหมด" นอกจากดูจากขนาดไฟล์จริงบนดิสก์เอง (ไฟล์ Indexed ก็มีข้อจำกัด
  คล้ายกันเรื่องการนับจำนวน record ทั้งหมด)

### แบบฝึกหัดที่ 289.1

**โจทย์**: จงอธิบายว่าทำไมการอ่านแบบเรียงลำดับ (`READ NEXT RECORD` จากขั้นตอนที่ 283) จึงไม่
สามารถใช้แทนเทคนิคในขั้นตอนนี้ได้ ทั้งที่ดูเหมือนจะ "เร็วกว่า" เพราะอ่านแค่ 4 record ไม่ใช่ 10

**เฉลย**: เพราะโจทย์ของขั้นตอนนี้คือ "ต้องการทราบว่าที่นั่งใด**ว่าง**" ไม่ใช่ "ต้องการทราบว่าที่นั่ง
ใดมีคนนั่ง" การอ่านแบบเรียงลำดับด้วย `READ NEXT RECORD` จะข้ามที่นั่งว่างไปโดยอัตโนมัติเสมอ (ตามที่
พิสูจน์ในขั้นตอนที่ 283) ทำให้ไม่มีทางรู้เลยว่าที่นั่งที่ 2, 3, 6, 7, 8, 10 นั้น "ว่างจริง" หรือ
"ไม่เคยมีอยู่" ต่างจากการไล่อ่านแบบสุ่มทีละตำแหน่งที่ตรวจสอบทุกตำแหน่งอย่างชัดเจนไม่มีการข้าม แม้จะ
ใช้จำนวนครั้ง `READ` มากกว่าก็ตาม แลกกับความสมบูรณ์ของข้อมูลที่ได้

---

## ขั้นตอนที่ 290: โปรแกรมรวบยอด — ระบบจองที่นั่งรถบัสแบบครบวงจร

### เป้าหมาย

ขั้นตอนสุดท้ายของ Part นี้รวมทุกเทคนิคที่เรียนมาทั้ง 9 ขั้นตอนเข้าด้วยกัน จำลองระบบจองที่นั่งรถบัส
ที่มีฟังก์ชันครบวงจร: เริ่มต้นระบบ → จองที่นั่งว่างช่องแรกอัตโนมัติ (`WRITE`) → ยกเลิกที่นั่ง
(`DELETE`) → จองซ้ำในช่องที่ว่างแล้ว → เปลี่ยนชื่อผู้โดยสาร (`REWRITE`) → พิมพ์ผังที่นั่งสุดท้าย

ระหว่างทางโปรแกรมนี้ยังเผยให้เห็น**กับดักสำคัญที่พิสูจน์จริงด้วยการทดสอบ**: ไฟล์ Relative ที่ยังไม่
เคยมีการเขียนอะไรเลยแม้แต่ record เดียว จะตอบสนองต่อการ `READ` แบบสุ่มด้วยสถานะ **`10` (AT END)**
แทนที่จะเป็น **`23` (INVALID KEY)** ที่เราคาดหวัง เนื่องจาก `READ` แบบสุ่มเข้าใจแค่ `INVALID KEY`
เท่านั้น (ไม่มี `AT END` ให้ใช้) ผลคือ**ทั้งสองสาขาจะไม่ทำงานเลยและโปรแกรมจะหยุดทำงานกะทันหัน**
ถ้าไม่ป้องกันไว้ก่อน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP290-BUS-RESERVATION.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SEAT-FILE ASSIGN TO "BUS290.DAT"
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS DYNAMIC
               RELATIVE KEY IS WS-SEAT-RRN.

       DATA DIVISION.
       FILE SECTION.
       FD  SEAT-FILE.
       01  SEAT-RECORD              PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-SEAT-RRN              PIC 9(4).
       01  WS-TOTAL-SEATS           PIC 9(4) VALUE 8.
       01  WS-FOUND-FLAG            PIC X.
           88  SLOT-FOUND           VALUE "Y".
       01  WS-PASSENGER-NAME        PIC X(20).
       01  WS-TARGET-SEAT           PIC 9(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM INIT-EMPTY-BUS.

           MOVE "SOMCHAI JAIDEE" TO WS-PASSENGER-NAME.
           PERFORM BOOK-SEAT.

           MOVE "SUDA MEECHAI" TO WS-PASSENGER-NAME.
           PERFORM BOOK-SEAT.

           MOVE 1 TO WS-TARGET-SEAT.
           PERFORM CANCEL-SEAT.

           MOVE "PRASERT KAEWTA" TO WS-PASSENGER-NAME.
           PERFORM BOOK-SEAT.

           MOVE 2 TO WS-TARGET-SEAT.
           MOVE "SUDA (UPDATED NAME)" TO WS-PASSENGER-NAME.
           PERFORM CHANGE-PASSENGER.

           PERFORM PRINT-SEAT-MAP.
           STOP RUN.

       INIT-EMPTY-BUS.
      *> IMPORTANT (confirmed by testing): a RELATIVE file that has
      *> NEVER had any record written at all answers a random READ
      *> with status 10 (AT END) instead of an INVALID KEY - and
      *> since a random READ only understands INVALID KEY, neither
      *> branch fires and the program aborts. The fix: write ONE
      *> placeholder record at the highest slot, then delete it.
      *> This fixes the file's extent so every later random READ
      *> on an empty slot correctly reports INVALID KEY (status
      *> 23) as expected.
           OPEN OUTPUT SEAT-FILE.
           MOVE WS-TOTAL-SEATS TO WS-SEAT-RRN.
           MOVE "PLACEHOLDER" TO SEAT-RECORD.
           WRITE SEAT-RECORD.
           CLOSE SEAT-FILE.

           OPEN I-O SEAT-FILE.
           MOVE WS-TOTAL-SEATS TO WS-SEAT-RRN.
           DELETE SEAT-FILE RECORD.
           CLOSE SEAT-FILE.

           DISPLAY "1) Bus initialized with " WS-TOTAL-SEATS
               " seats, all empty.".

       BOOK-SEAT.
           OPEN I-O SEAT-FILE.
           MOVE "N" TO WS-FOUND-FLAG.
           MOVE 1 TO WS-SEAT-RRN.
           PERFORM UNTIL SLOT-FOUND OR WS-SEAT-RRN > WS-TOTAL-SEATS
               READ SEAT-FILE
                   INVALID KEY
                       SET SLOT-FOUND TO TRUE
                   NOT INVALID KEY
                       ADD 1 TO WS-SEAT-RRN
               END-READ
           END-PERFORM.
           IF SLOT-FOUND
               MOVE WS-PASSENGER-NAME TO SEAT-RECORD
               WRITE SEAT-RECORD
                   INVALID KEY
                       DISPLAY "   Booking failed unexpectedly."
                   NOT INVALID KEY
                       DISPLAY "2) Booked seat " WS-SEAT-RRN
                           " for " WS-PASSENGER-NAME
               END-WRITE
           ELSE
               DISPLAY "2) Bus is full - cannot book "
                   WS-PASSENGER-NAME
           END-IF.
           CLOSE SEAT-FILE.

       CANCEL-SEAT.
           OPEN I-O SEAT-FILE.
           MOVE WS-TARGET-SEAT TO WS-SEAT-RRN.
           DELETE SEAT-FILE RECORD
               INVALID KEY
                   DISPLAY "3) Cancel failed - seat "
                       WS-TARGET-SEAT " was already empty."
               NOT INVALID KEY
                   DISPLAY "3) Cancelled seat " WS-TARGET-SEAT "."
           END-DELETE.
           CLOSE SEAT-FILE.

       CHANGE-PASSENGER.
           OPEN I-O SEAT-FILE.
           MOVE WS-TARGET-SEAT TO WS-SEAT-RRN.
           READ SEAT-FILE
               INVALID KEY
                   DISPLAY "4) Seat " WS-TARGET-SEAT " is empty -"
                       " nothing to rename."
               NOT INVALID KEY
                   MOVE WS-PASSENGER-NAME TO SEAT-RECORD
                   REWRITE SEAT-RECORD
                   DISPLAY "4) Seat " WS-TARGET-SEAT
                       " renamed to " WS-PASSENGER-NAME
           END-READ.
           CLOSE SEAT-FILE.

       PRINT-SEAT-MAP.
           DISPLAY "5) ---- Final seat map ----".
           OPEN INPUT SEAT-FILE.
           MOVE 1 TO WS-SEAT-RRN.
           PERFORM WS-TOTAL-SEATS TIMES
               READ SEAT-FILE
                   INVALID KEY
                       DISPLAY "   Seat " WS-SEAT-RRN ": empty"
                   NOT INVALID KEY
                       DISPLAY "   Seat " WS-SEAT-RRN ": "
                           SEAT-RECORD
               END-READ
               ADD 1 TO WS-SEAT-RRN
           END-PERFORM.
           CLOSE SEAT-FILE.
```

**ผลลัพธ์:**

```
1) Bus initialized with 0008 seats, all empty.
2) Booked seat 0001 for SOMCHAI JAIDEE
2) Booked seat 0002 for SUDA MEECHAI
3) Cancelled seat 0001.
2) Booked seat 0001 for PRASERT KAEWTA
4) Seat 0002 renamed to SUDA (UPDATED NAME)
5) ---- Final seat map ----
   Seat 0001: PRASERT KAEWTA
   Seat 0002: SUDA (UPDATED NAME)
   Seat 0003: empty
   Seat 0004: empty
   Seat 0005: empty
   Seat 0006: empty
   Seat 0007: empty
   Seat 0008: empty
```

### อธิบายภาพรวมของโปรแกรม

โปรแกรมนี้แบ่งงานเป็น 5 paragraph ตามหลักการ Single Responsibility เช่นเดียวกับโปรแกรมรวบยอดของ
Part 028 ขั้นตอน 280:

1. **INIT-EMPTY-BUS**: เตรียมไฟล์เปล่าอย่างปลอดภัย (แก้ปัญหา AT END ที่อธิบายไว้ข้างต้น)
2. **BOOK-SEAT**: ค้นหาที่นั่งว่างช่องแรกด้วยเทคนิคจากขั้นตอนที่ 287 แล้วจองด้วย `WRITE`
3. **CANCEL-SEAT**: ยกเลิกที่นั่งด้วย `DELETE` โดยไม่ต้อง `READ` ก่อน (ตามที่พิสูจน์ในขั้นตอนที่
   286)
4. **CHANGE-PASSENGER**: เปลี่ยนชื่อผู้โดยสารในที่นั่งที่จองแล้วด้วย `REWRITE`
5. **PRINT-SEAT-MAP**: แสดงผังที่นั่งทั้งหมดด้วยเทคนิคไล่อ่านทีละตำแหน่งจากขั้นตอนที่ 289

สังเกตผลลัพธ์ท้ายสุด: ที่นั่ง 1 ถูกจองครั้งแรกโดย SOMCHAI แล้วถูกยกเลิก จากนั้น PRASERT มาจองซ้ำ
ที่ตำแหน่งเดิม (RRN 1) เพราะเป็นตำแหน่งว่างช่องแรกอีกครั้ง — พิสูจน์ว่าการนำ RRN กลับมาใช้ใหม่
ทำงานได้อย่างถูกต้องสมบูรณ์

### ข้อควรระวัง

- กับดักเรื่อง "ไฟล์ว่างสนิท" ที่พบในขั้นตอนนี้เป็นตัวอย่างที่ดีว่าทำไมการ **ทดสอบโปรแกรมจริงด้วย
  ข้อมูลขอบเขต (Edge Case) เช่น ไฟล์เปล่า** จึงสำคัญมาก แม้โค้ดจะดูถูกต้องสมบูรณ์แบบเมื่อทดสอบด้วย
  ไฟล์ที่มีข้อมูลอยู่แล้วก็ตาม
- เทคนิค "เขียน placeholder แล้วลบทิ้ง" ใน `INIT-EMPTY-BUS` เป็นวิธีแก้ปัญหาเฉพาะหน้าที่ใช้ได้ผล
  จริงกับ GnuCOBOL แต่พฤติกรรมนี้**อาจแตกต่างกันไปในแต่ละ COBOL compiler** ควรทดสอบยืนยันพฤติกรรม
  นี้เสมอเมื่อย้ายโปรแกรมไปรันบน compiler อื่น (Part 049 จะพูดถึงความแตกต่างระหว่าง COBOL Dialects
  โดยละเอียด)

### แบบฝึกหัดที่ 290.1

**โจทย์**: จงเพิ่มการตรวจสอบใน `BOOK-SEAT` ว่าถ้าค้นหาจนถึง `WS-SEAT-RRN > WS-TOTAL-SEATS` แล้ว
ยังไม่พบที่ว่างเลย (รถเต็ม) โปรแกรมควรแสดงข้อความอะไร และเงื่อนไขนี้ถูกป้องกันไว้แล้วหรือยังในโค้ด
ต้นฉบับ

**เฉลย**: เงื่อนไขนี้ถูกป้องกันไว้แล้วบางส่วนผ่าน `PERFORM UNTIL SLOT-FOUND OR WS-SEAT-RRN >
WS-TOTAL-SEATS` ซึ่งทำให้ลูปหยุดแม้จะยังไม่พบที่ว่าง แต่หลังจากลูปจบ โค้ดยังคง `IF SLOT-FOUND`
ตรวจสอบอยู่แล้ว และมีสาขา `ELSE` แสดงข้อความ `"Bus is full - cannot book ..."` ไว้รองรับกรณีนี้
อยู่แล้วในโค้ดต้นฉบับ ทดลองได้ด้วยการเปลี่ยน `WS-TOTAL-SEATS` เป็น `VALUE 1` แล้วรันโปรแกรมใหม่
(หลังจองที่นั่งที่ 1 ไปแล้ว การจองครั้งที่สองจะแสดงข้อความ "Bus is full" ทันที)

---

## สรุปท้ายบท

Part นี้แนะนำไฟล์ประเภทที่สามและประเภทสุดท้ายของ COBOL: **Relative File** สรุปสิ่งที่เรียนรู้:

- แนวคิด Relative Record Number (RRN): ตัวระบุตำแหน่งทางกายภาพล้วน ๆ ที่ไม่มีความหมายทางธุรกิจ
  ต่างจาก `RECORD KEY` ของไฟล์ Indexed
- การเขียนแบบอัตโนมัติ (`ACCESS MODE IS SEQUENTIAL`) เทียบกับการกำหนดตำแหน่งเอง
  (`ACCESS MODE IS RANDOM`)
- พฤติกรรม `READ NEXT RECORD` ที่ข้ามช่องว่างอัตโนมัติในไฟล์แบบ Sparse
- `START` สำหรับกำหนดจุดเริ่มต้นการอ่านด้วยเงื่อนไขเชิง RRN
- `REWRITE` และ `DELETE` ที่ทำงานได้อย่างปลอดภัยเหมือนไฟล์ Indexed พร้อมความสามารถนำ RRN ที่ถูกลบ
  กลับมาใช้ใหม่ได้ทันที
- `ACCESS MODE IS DYNAMIC` สำหรับผสมการค้นหาแบบสุ่มกับการไล่หาช่องว่าง (เทคนิคหาที่นั่งว่างช่องแรก)
- การคำนวณ RRN โดยตรงจาก Business Key ด้วยสูตรคณิตศาสตร์ ทำให้เกิดการเข้าถึงแบบ `O(1)` แท้จริง
- ข้อจำกัดสำคัญ: ไฟล์ Relative ไม่มี "สารบัญ" ของช่องว่าง ต้องไล่ตรวจสอบทีละตำแหน่งเพื่อสร้างรายงาน
  ผังเต็ม
- กับดักจริงที่พิสูจน์ด้วยการทดสอบ: ไฟล์ว่างสนิทตอบสนองต่อการอ่านแบบสุ่มด้วยสถานะ AT END แทน
  INVALID KEY และวิธีแก้ไขด้วยเทคนิค placeholder
- โปรแกรมรวบยอดระบบจองที่นั่งรถบัสแบบครบวงจร

ด้วย Part 028–029 คุณได้เรียนรู้ไฟล์ทั้ง 3 ประเภทของ COBOL ครบถ้วนแล้ว (Sequential, Indexed,
Relative) Part ถัดไป (**Part 030**) จะไม่แนะนำไฟล์ประเภทใหม่ แต่จะเจาะลึก **File Status Codes**
อย่างเป็นระบบ — ตารางรหัสสถานะทั้งหมดที่เคยเห็นผ่าน ๆ มาตลอด Part 025–029 (เช่น `00`, `10`, `21`,
`22`, `23`, `35`, `38`) พร้อมเทคนิคการจัดการข้อผิดพลาดของไฟล์อย่างเป็นระบบสำหรับโปรแกรมระดับองค์กร
จริงที่ต้องรับมือกับความผิดพลาดอย่างสง่างามโดยไม่ล่มกลางคัน

**[← กลับไป Part 028](part-028-indexed-files.md)** | **[ไปยัง Part 030: File Status Codes →](part-030-file-status-codes.md)**
