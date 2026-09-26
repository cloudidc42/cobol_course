# Part 030: File Status Codes และการจัดการข้อผิดพลาดของไฟล์ (ขั้นตอนที่ 291–300)

## คำนำของ Part นี้

Part 028 และ 029 พาเราเรียนรู้ไฟล์ทั้ง 3 ประเภทของ COBOL ครบถ้วนแล้ว (Sequential, Indexed,
Relative) ตลอดทางเราได้เห็น `FILE STATUS IS` โผล่มาเป็นระยะ ๆ พร้อมตัวเลขลึกลับอย่าง `21`, `22`,
`23`, `35`, `38` โดยยังไม่ได้อธิบายอย่างเป็นระบบว่าตัวเลขเหล่านี้มาจากไหน มีความหมายครบถ้วนอย่างไร
และควรออกแบบโปรแกรมให้ตรวจสอบมันอย่างเป็นระบบอย่างไร

Part นี้จะรวบรวมทุกอย่างเรื่อง **File Status Codes** ไว้ในที่เดียว: ตารางรหัสมาตรฐานทั้งหมดตามหมวด
หมู่ วิธีทดสอบยืนยันแต่ละรหัสด้วยการทดลองจริง (เช่นเดียวกับที่ทำมาตลอดทั้ง Part 023–029) และเทคนิค
การออกแบบโปรแกรมที่**ไม่ล่มกลางคัน**เมื่อเจอปัญหาไฟล์ ด้วยการตรวจสอบ `FILE STATUS` อย่างเป็นระบบแทน
การปล่อยให้ error จัดการโปรแกรมเอง — ทักษะที่จำเป็นอย่างยิ่งสำหรับโปรแกรม COBOL ระดับองค์กรที่ต้อง
ทำงานตลอด 24 ชั่วโมงโดยไม่มีคนเฝ้าหน้าจอ

> **หมายเหตุขอบเขต**: Part นี้เน้นเทคนิคการตรวจสอบ `FILE STATUS` ด้วยมือ (`IF`, `EVALUATE`) ซึ่งเป็น
> พื้นฐานที่สำคัญที่สุด ส่วนกลไกขั้นสูงกว่าอย่าง `USE` Statement และ `DECLARATIVES` (การดักจับ
> ข้อผิดพลาดแบบรวมศูนย์อัตโนมัติ) จะสอนแยกต่างหากอย่างละเอียดใน **Part 048 (Error Handling ขั้นสูง)**

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน โค้ดทุกตัวอย่างและ
> รหัสสถานะทุกตัวที่กล่าวถึงผ่านการคอมไพล์และรันทดสอบจริงด้วย GnuCOBOL แล้วทุกตัวอย่าง (รหัสที่ไม่
> สามารถทดสอบซ้ำได้ง่ายในสภาพแวดล้อมนี้ เช่น รหัสที่เกี่ยวกับเทปแม่เหล็กหรือขีดจำกัดเฉพาะอุปกรณ์
> จะระบุไว้ชัดเจนว่าเป็นคำอธิบายเชิงทฤษฎี)

---

## ขั้นตอนที่ 291: FILE STATUS คืออะไร — โครงสร้างและหมวดหมู่ตามมาตรฐาน

### ทบทวน: FILE STATUS คือฟิลด์ 2 ตัวอักษรเสมอ

`FILE STATUS IS` เป็น clause ที่เพิ่มเข้าไปใน `SELECT` (เห็นครั้งแรกแบบผ่าน ๆ ใน Part 025 ขั้นตอน
242) เพื่อบอก COBOL ว่า **"ทุกครั้งที่ทำปฏิบัติการกับไฟล์นี้ (`OPEN`, `READ`, `WRITE`, `REWRITE`,
`DELETE`, `START`, `CLOSE`) ให้เก็บรหัสผลลัพธ์ไว้ในตัวแปรนี้เสมอ"** ตัวแปรนี้ต้องเป็น
`PIC XX` (ตัวอักษร 2 ตัว) **เสมอ ไม่ใช่ตัวเลข** แม้ค่าส่วนใหญ่จะหน้าตาเหมือนตัวเลขก็ตาม

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP291-FILE-STATUS-INTRO.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST291.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD          PIC X(30).

       WORKING-STORAGE SECTION.
      *> FILE STATUS is ALWAYS a 2-character alphanumeric field -
      *> never numeric, even though most values look like numbers.
       01  WS-FILE-STATUS           PIC XX.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Every single file operation (OPEN, READ, WRITE, REWRITE,
      *> DELETE, CLOSE, START) updates WS-FILE-STATUS automatically,
      *> whether it succeeds or fails - with NO extra code needed.
           OPEN OUTPUT CUSTOMER-FILE.
           DISPLAY "After OPEN OUTPUT, status = " WS-FILE-STATUS.

           MOVE "0001 SOMCHAI JAIDEE" TO CUSTOMER-RECORD.
           WRITE CUSTOMER-RECORD.
           DISPLAY "After WRITE,        status = " WS-FILE-STATUS.

           CLOSE CUSTOMER-FILE.
           DISPLAY "After CLOSE,        status = " WS-FILE-STATUS.

           DISPLAY " ".
           DISPLAY "First digit meaning (COBOL standard):".
           DISPLAY "  0x = success        1x = end of file".
           DISPLAY "  2x = invalid key    3x = permanent error".
           DISPLAY "  4x = logic error    9x = compiler-specific".
           STOP RUN.
```

**ผลลัพธ์:**

```
After OPEN OUTPUT, status = 00
After WRITE,        status = 00
After CLOSE,        status = 00
 
First digit meaning (COBOL standard):
  0x = success        1x = end of file
  2x = invalid key    3x = permanent error
  4x = logic error    9x = compiler-specific
```

### ตารางหมวดหมู่ตามมาตรฐาน COBOL (ตัวเลขหลักแรก)

| หลักแรก | หมวดหมู่ | ความหมาย |
|---|---|---|
| `0x` | สำเร็จ (Success) | ปฏิบัติการสำเร็จ อาจมีเงื่อนไขพิเศษแนบมาด้วย |
| `1x` | สิ้นสุดไฟล์ (At End) | อ่านถึงจุดสิ้นสุดไฟล์แล้ว ไม่มี record เหลือ |
| `2x` | Key ไม่ถูกต้อง (Invalid Key) | ปัญหาเกี่ยวกับ Key/RRN: ซ้ำ, ไม่พบ, ลำดับผิด |
| `3x` | ข้อผิดพลาดถาวร (Permanent Error) | ปัญหาระดับไฟล์/อุปกรณ์ที่แก้ไขเองในโปรแกรมไม่ได้ |
| `4x` | ข้อผิดพลาดเชิงตรรกะ (Logic Error) | โปรแกรมเรียกคำสั่งผิดจังหวะ/ผิดโหมดที่เปิดไว้ |
| `9x` | กำหนดโดยผู้ผลิต Compiler | รหัสเฉพาะของแต่ละ compiler (ไม่ใช่มาตรฐานกลาง) |

### อธิบายจุดสำคัญ

- ค่า `"00"` คือ **"สำเร็จสมบูรณ์แบบไม่มีเงื่อนไขพิเศษ"** เป็นค่าที่ควรพบมากที่สุดในโปรแกรมที่ทำงาน
  ปกติ ค่าอื่นทุกค่าล้วนบอกใบ้ว่ามีบางอย่างที่**ควรตรวจสอบเพิ่มเติม**
- ตัวเลขหลักที่สองให้รายละเอียดย่อยภายในแต่ละหมวด (เช่น `21` กับ `22` ต่างก็เป็นหมวด `2x` แต่มี
  ความหมายย่อยต่างกันคนละเรื่อง) จะเจาะลึกความหมายครบทุกรหัสสำคัญในขั้นตอนที่ 292–296
- การประกาศ `FILE STATUS IS` **ไม่บังคับ**ในไวยากรณ์ COBOL (โปรแกรมทั้งหมดใน Part 023–029 ส่วนใหญ่
  ก็ไม่ได้ประกาศ) แต่เป็น**แนวปฏิบัติที่ดีอย่างยิ่ง**สำหรับโปรแกรมที่ต้องทำงานกับข้อมูลสำคัญ

### ข้อควรระวัง

- ห้ามประกาศ `WS-FILE-STATUS` เป็น `PIC 99` (ตัวเลข) แม้จะดูสมเหตุสมผลก็ตาม เพราะมาตรฐาน COBOL
  กำหนดตายตัวว่าต้องเป็น alphanumeric (`PIC X` หรือ `PIC XX`) และ compiler บางตัวจะปฏิเสธถ้าประกาศ
  ผิดชนิด
- อย่าลืมว่า `FILE STATUS` เป็นแนวคิด**แยกต่างหาก**จาก `AT END`/`INVALID KEY` ที่เรียนมาตลอด — ทั้ง
  สองกลไกทำงานคู่ขนานกัน (COBOL อัปเดต `FILE STATUS` เสมอไม่ว่าจะมี `AT END`/`INVALID KEY` clause
  หรือไม่ก็ตาม)

### แบบฝึกหัดที่ 291.1

**โจทย์**: จงอธิบายว่าทำไม `FILE STATUS` จึงต้องเป็น `PIC XX` (alphanumeric) แทนที่จะเป็น
`PIC 99` (ตัวเลข) ทั้งที่ค่าที่เก็บดูเหมือนตัวเลขเกือบทั้งหมด

**เฉลย**: เพราะมาตรฐาน COBOL กำหนดรหัสสถานะบางส่วนที่**ไม่ใช่ตัวเลขล้วน** โดยเฉพาะรหัสในหมวด `9x`
ซึ่งเป็นรหัสเฉพาะของแต่ละ compiler ที่อาจใช้ตัวอักษรผสมได้ (เช่น GnuCOBOL บางเวอร์ชันใช้ค่าแบบ
`9` ตามด้วยตัวอักษร) การกำหนดเป็น `PIC XX` แต่แรกทำให้ฟิลด์นี้รองรับได้ทุกกรณีตามมาตรฐานอย่างสมบูรณ์
โดยไม่ต้องกังวลว่าจะมีค่าที่ไม่ใช่ตัวเลขหลุดเข้ามาแล้วเกิด error จากการแปลงชนิดข้อมูล

---

## ขั้นตอนที่ 292: หมวด 0x — ตระกูลความสำเร็จ (00, 02, 05)

### ไม่ใช่ทุก "สำเร็จ" จะเหมือนกัน

แม้จะอยู่ในหมวดความสำเร็จเหมือนกัน แต่รหัสย่อยในหมวด `0x` บอกเงื่อนไขพิเศษที่ควรรู้ไว้:

| รหัส | ความหมาย |
|---|---|
| `00` | สำเร็จสมบูรณ์ ไม่มีเงื่อนไขพิเศษใด ๆ |
| `02` | สำเร็จ แต่เกี่ยวข้องกับ Alternate Key ที่ยอมให้ค่าซ้ำกันได้ |
| `04` | สำเร็จ แต่ความยาว record ที่อ่านได้ไม่ตรงกับที่ประกาศไว้ใน FD (มักเกิดกับ record ความยาวแปรผัน) |
| `05` | สำเร็จ (`OPEN` ไม่ error) แต่ไฟล์ `OPTIONAL` นี้ไม่มีอยู่จริง ณ ขณะเปิด |

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP292-SUCCESS-FAMILY.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST292.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               ALTERNATE RECORD KEY IS CUST-CITY
                   WITH DUPLICATES
               FILE STATUS IS WS-FILE-STATUS.

           SELECT OPTIONAL LOG-FILE ASSIGN TO "MAYBE292.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-LOG-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           05  CUST-ID              PIC 9(5).
           05  CUST-NAME            PIC X(15).
           05  CUST-CITY            PIC X(10).

       FD  LOG-FILE.
       01  LOG-RECORD               PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.
       01  WS-LOG-STATUS            PIC XX.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Status 02 (tested with GnuCOBOL/BDB): writing to a file
      *> that has ANY ALTERNATE RECORD KEY WITH DUPLICATES reports
      *> status 02 on EVERY WRITE to that file - not only when an
      *> actual duplicate value occurs. This is a real, confirmed
      *> behavior: 02 means "this file has a key that CAN carry
      *> duplicates", so always check WS-FILE-STATUS = "00" (exact
      *> plain success) separately from "02" if the distinction
      *> matters to your program.
           OPEN OUTPUT CUSTOMER-FILE.
           MOVE 10001 TO CUST-ID.
           MOVE "SOMCHAI" TO CUST-NAME.
           MOVE "CHIANGMAI" TO CUST-CITY.
           WRITE CUSTOMER-RECORD.
           DISPLAY "First write  (CHIANGMAI) -> status "
               WS-FILE-STATUS.

           MOVE 10002 TO CUST-ID.
           MOVE "PRASERT" TO CUST-NAME.
           MOVE "BANGKOK" TO CUST-CITY.
           WRITE CUSTOMER-RECORD.
           DISPLAY "Second write (BANGKOK)   -> status "
               WS-FILE-STATUS.
           CLOSE CUSTOMER-FILE.

      *> Status 05 - the SELECT clause has OPTIONAL, and the file
      *> genuinely does not exist yet. OPEN still succeeds (the
      *> program does NOT crash, per Part 023 step 229) but the
      *> status tells us clearly that nothing was actually found.
           OPEN INPUT LOG-FILE.
           DISPLAY "OPTIONAL missing file -> status "
               WS-LOG-STATUS.
           CLOSE LOG-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
First write  (CHIANGMAI) -> status 02
Second write (BANGKOK)   -> status 02
OPTIONAL missing file -> status 05
```

### วิเคราะห์ผลลัพธ์ที่น่าสนใจ

สังเกตว่า**ทั้งสองการเขียนได้สถานะ `02` เหมือนกัน** แม้ค่า `CUST-CITY` แรก (`"CHIANGMAI"`) จะไม่ซ้ำ
กับใครเลยก็ตาม! นี่คือพฤติกรรมจริงที่ยืนยันด้วยการทดสอบ: **GnuCOBOL รุ่นนี้รายงานสถานะ `02` ทุกครั้ง
ที่เขียนลงไฟล์ที่มี `ALTERNATE RECORD KEY ... WITH DUPLICATES` อยู่ ไม่ว่าค่านั้นจะซ้ำจริงหรือไม่**
(ต่างจากการตีความตามตัวอักษรของมาตรฐานที่มักเข้าใจว่า `02` หมายถึง "ตรวจพบค่าซ้ำจริง" เท่านั้น) ถือ
เป็นความแตกต่างเชิง implementation ที่สำคัญมากที่ต้องรู้ไว้ — ถ้าโปรแกรมต้องแยกแยะ "สำเร็จธรรมดา"
ออกจาก "สำเร็จแบบมี Alternate Key ซ้ำได้" ต้องทดสอบพฤติกรรมจริงของ compiler ที่ใช้งานเสมอ ไม่ควร
เชื่อคำอธิบายจากเอกสารทฤษฎีเพียงอย่างเดียว

### อธิบายจุดสำคัญ

- สถานะ `05` พิสูจน์สิ่งที่ Part 023 ขั้นตอน 229 สอนไว้: `SELECT OPTIONAL` ทำให้ `OPEN` ไฟล์ที่ไม่
  มีอยู่จริง**ไม่ทำให้โปรแกรมล่ม** และตอนนี้เรารู้แล้วว่าเบื้องหลังคือสถานะ `05` ที่ถูกตั้งค่าไว้
  ให้ตรวจสอบได้
- ควรใช้ `IF WS-FILE-STATUS = "00"` (ตรวจสอบค่าตรง ๆ) แทน `IF WS-FILE-STATUS NOT = "00"`
  เมื่อต้องการแยกความสำเร็จแบบ "ไม่มีเงื่อนไขพิเศษ" ออกจากความสำเร็จแบบอื่น (`02`, `04`, `05`)
  อย่างเคร่งครัด

### ข้อควรระวัง

- อย่าเขียนโค้ดที่ถือว่า "ขึ้นต้นด้วย 0 คือโอเค ไม่ต้องสนใจอะไรเพิ่ม" เพราะรหัส `04` และ `05` แม้
  จะอยู่ในหมวดสำเร็จ แต่ก็มีเงื่อนไขพิเศษที่บางโปรแกรมจำเป็นต้องจัดการแตกต่างออกไป (เช่น `05` ควร
  แสดงข้อความแจ้งเตือนที่ต่างจาก `00`)
- ผลลัพธ์ของ `02` ที่พิสูจน์ในขั้นตอนนี้เป็น**พฤติกรรมเฉพาะของ GnuCOBOL เวอร์ชันที่ทดสอบ** อาจ
  แตกต่างใน compiler อื่น ควรทดสอบยืนยันด้วยตนเองเสมอก่อนพึ่งพาพฤติกรรมนี้ในโปรแกรมจริง

### แบบฝึกหัดที่ 292.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมที่ต้องการแยกแยะ "การเขียนที่ Alternate Key ซ้ำจริง" ออกจาก
"การเขียนปกติที่บังเอิญเก็บอยู่ในไฟล์ที่มี Alternate Key WITH DUPLICATES" จึงทำได้ยากถ้าอาศัย
`FILE STATUS` เพียงอย่างเดียวในกรณีของ GnuCOBOL ตามที่ทดสอบในขั้นตอนนี้

**เฉลย**: เพราะจากผลการทดสอบ ทั้งสองกรณีให้สถานะ `02` เหมือนกันทุกประการ ไม่มีความแตกต่างในค่า
`FILE STATUS` ให้แยกแยะได้เลย หากโปรแกรมต้องการทราบจริง ๆ ว่าค่านั้นซ้ำกับ record อื่นหรือไม่ ต้อง
ใช้เทคนิคเพิ่มเติม เช่น การ `READ` ค้นหาด้วย Alternate Key นั้นก่อน `WRITE` เพื่อตรวจสอบว่ามี record
อื่นที่ค่าเดียวกันอยู่แล้วหรือไม่ (คล้ายเทคนิคที่ใช้ใน Part 028 ขั้นตอนที่ 279) แทนที่จะพึ่งพา
`FILE STATUS` เพียงอย่างเดียว

---

## ขั้นตอนที่ 293: หมวด 1x — ตระกูลสิ้นสุดไฟล์ (10, 46)

### ไม่ใช่แค่ AT END — มีรหัสย่อยที่คนส่วนใหญ่ไม่เคยเจอ

Part 023 ขั้นตอน 225 สอน `AT END` ไปแล้วโดยไม่บอกว่าเบื้องหลังคือสถานะ `10` ขั้นตอนนี้จะเปิดเผย
สถานะที่เกี่ยวข้องอีกตัวที่หลายคนไม่เคยเจอเพราะโปรแกรมมักหยุดลูปทันทีที่เจอ `10`: สถานะ **`46`**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP293-END-OF-FILE-FAMILY.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST293.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD          PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT CUSTOMER-FILE.
           MOVE "ONLY RECORD" TO CUSTOMER-RECORD.
           WRITE CUSTOMER-RECORD.
           CLOSE CUSTOMER-FILE.

           OPEN INPUT CUSTOMER-FILE.

      *> Status 00 - the ONE record in the file reads fine.
           READ CUSTOMER-FILE.
           DISPLAY "Read #1 (the only record) -> status "
               WS-FILE-STATUS.

      *> Status 10 - AT END: we asked for the next record but
      *> there isn't one. This is the status behind the AT END
      *> phrase you have used since Part 023.
           READ CUSTOMER-FILE.
           DISPLAY "Read #2 (past the end)    -> status "
               WS-FILE-STATUS.

      *> Status 46 - a DIFFERENT status from 10! It means "you
      *> already read past the end once, and tried to READ AGAIN
      *> without a START to reposition first." Many programs never
      *> see 46 because their loop stops as soon as it sees 10 -
      *> but it is a distinct, real logic-error status.
           READ CUSTOMER-FILE.
           DISPLAY "Read #3 (read again after AT END) -> status "
               WS-FILE-STATUS.

           CLOSE CUSTOMER-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Read #1 (the only record) -> status 00
Read #2 (past the end)    -> status 10
Read #3 (read again after AT END) -> status 46
```

### อธิบายจุดสำคัญ

- โปรแกรมนี้**จงใจไม่ใส่ `AT END` clause เลย** เพื่อโชว์ค่า `FILE STATUS` ดิบ ๆ โดยตรง — สังเกตว่า
  โปรแกรม**ไม่ล่ม**แม้ `READ` จะไม่มี `AT END` เพราะเราตรวจสอบผ่าน `WS-FILE-STATUS` แทน (นี่คือ
  วิธีเขียนโปรแกรมแบบ "ตรวจสอบด้วย FILE STATUS ล้วน ๆ" ซึ่งเป็นสไตล์ทางเลือกที่ใช้กันในองค์กรใหญ่
  บางแห่งแทนการพึ่ง `AT END`)
- สถานะ `10` คือสิ่งที่เกิดขึ้นเมื่อ `READ` ไปแล้วไม่มี record เหลือ (ตำแหน่งไฟล์อยู่ที่ปลายสุด) —
  ตรงกับสาขา `AT END` ที่เราคุ้นเคย
- สถานะ `46` คือสิ่งที่เกิดขึ้นเมื่อพยายาม `READ` **ซ้ำอีกครั้ง**หลังจากได้ `10` ไปแล้ว โดยไม่ได้
  `START` เพื่อจัดตำแหน่งใหม่ก่อน — เป็นสัญญาณของ**ข้อผิดพลาดเชิงตรรกะในโปรแกรม** (ลืมหยุดลูปแล้ว
  ยัง `READ` ต่อ) มากกว่าจะเป็นสถานการณ์ปกติ

### ข้อควรระวัง

- โปรแกรมที่ใช้ Pattern มาตรฐาน `PERFORM UNTIL END-OF-FILE` (จาก Part 023 ขั้นตอน 226) จะไม่มีทาง
  เจอสถานะ `46` เลยในทางปฏิบัติ เพราะลูปจะหยุดทันทีที่เจอ `10` — แต่ถ้าเขียนลูปผิดพลาด (เช่น ลืม
  `SET END-OF-FILE TO TRUE` ในสาขา `AT END`) อาจทำให้โปรแกรมเรียก `READ` ซ้ำจนเจอ `46` ได้
- อย่าใช้ `46` เป็นสัญญาณ "จบไฟล์" แทน `10` เพราะความหมายต่างกัน (`46` คือ "จบไฟล์ไปแล้วและพยายาม
  อ่านอีก" ไม่ใช่ "เพิ่งจบไฟล์")

### แบบฝึกหัดที่ 293.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมที่เขียนด้วย Pattern มาตรฐาน `PERFORM UNTIL END-OF-FILE` ร่วมกับ
`AT END` / `SET END-OF-FILE TO TRUE` (จาก Part 023) จึงแทบไม่มีโอกาสเจอสถานะ `46` เลย

**เฉลย**: เพราะ Pattern มาตรฐานนี้ตรวจสอบเงื่อนไข `END-OF-FILE` **ก่อน**ที่จะเรียก `READ` ทุกครั้ง
(`PERFORM UNTIL END-OF-FILE`) เมื่อ `READ` ครั้งใดครั้งหนึ่งได้สถานะ `10` (`AT END`) โปรแกรมจะ
`SET END-OF-FILE TO TRUE` ทันที ทำให้ในรอบถัดไปของลูป เงื่อนไข `PERFORM UNTIL` เป็นจริงและลูปจะ
**หยุดทันทีโดยไม่เรียก `READ` อีกเลย** สถานะ `46` จะเกิดขึ้นก็ต่อเมื่อมีการเรียก `READ` **หลังจาก**
ได้ `10` ไปแล้วเท่านั้น ซึ่ง Pattern มาตรฐานป้องกันสถานการณ์นี้ไว้อยู่แล้วโดยธรรมชาติของโครงสร้าง
ลูป

---

## ขั้นตอนที่ 294: หมวด 2x — ตระกูล Invalid Key (21, 22, 23)

### รวบรวมรหัสที่เจอกระจัดกระจายมาตลอด Part 028–029

รหัสในหมวดนี้ล้วนเกี่ยวกับปัญหา Key/RRN ที่เราเคยเจอมาแล้วทีละตัวใน Part ก่อนหน้า มาถึงขั้นตอนนี้
จะรวบรวมทั้งหมดไว้ในโปรแกรมเดียวเพื่อเปรียบเทียบชัด ๆ:

| รหัส | ความหมาย | เคยเห็นครั้งแรกใน |
|---|---|---|
| `21` | ลำดับ Key ไม่เรียงจากน้อยไปมาก (เฉพาะ `ACCESS MODE SEQUENTIAL`) | Part 028 ขั้นตอน 271 |
| `22` | เขียน/แก้ไข Key ที่ซ้ำกับ record ที่มีอยู่แล้ว | Part 028 ขั้นตอน 278 |
| `23` | ค้นหา Key/RRN ที่ไม่มีอยู่จริงในไฟล์ | Part 028 ขั้นตอน 273 |
| `24` | เขียนเกินขอบเขตที่กำหนดไว้ของไฟล์ (พบยากในทางปฏิบัติ อธิบายเชิงทฤษฎี) | — |

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP294-INVALID-KEY-FAMILY.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
      *> ACCESS MODE SEQUENTIAL - the only mode that enforces
      *> ascending key order, needed to demonstrate status 21.
           SELECT SEQ-FILE ASSIGN TO "PRODSEQ294.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS SQ-CODE
               FILE STATUS IS WS-FILE-STATUS.

      *> ACCESS MODE DYNAMIC - needed for keyed random WRITE/READ,
      *> to demonstrate statuses 22 and 23.
           SELECT DYN-FILE ASSIGN TO "PRODDYN294.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS DY-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  SEQ-FILE.
       01  SEQ-RECORD.
           05  SQ-CODE              PIC 9(3).
           05  SQ-NAME              PIC X(10).

       FD  DYN-FILE.
       01  DYN-RECORD.
           05  DY-CODE              PIC 9(3).
           05  DY-NAME              PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT SEQ-FILE.
           MOVE 500 TO SQ-CODE.
           MOVE "RICE" TO SQ-NAME.
           WRITE SEQ-RECORD.
           DISPLAY "Write code 500          -> status "
               WS-FILE-STATUS.

      *> Status 21 - SEQUENCE ERROR. ACCESS MODE IS SEQUENTIAL with
      *> OPEN OUTPUT requires ascending key order. 300 < 500, so
      *> this WRITE is rejected outright.
           MOVE 300 TO SQ-CODE.
           MOVE "OIL" TO SQ-NAME.
           WRITE SEQ-RECORD.
           DISPLAY "Write code 300 (< 500)  -> status "
               WS-FILE-STATUS.
           CLOSE SEQ-FILE.

           OPEN OUTPUT DYN-FILE.
           MOVE 500 TO DY-CODE.
           MOVE "RICE" TO DY-NAME.
           WRITE DYN-RECORD.
           CLOSE DYN-FILE.

           OPEN I-O DYN-FILE.

      *> Status 22 - DUPLICATE KEY. Code 500 already exists.
           MOVE 500 TO DY-CODE.
           MOVE "IMPOSTOR" TO DY-NAME.
           WRITE DYN-RECORD.
           DISPLAY "Write duplicate code 500 -> status "
               WS-FILE-STATUS.

      *> Status 23 - RECORD NOT FOUND. Code 999 was never written.
           MOVE 999 TO DY-CODE.
           READ DYN-FILE.
           DISPLAY "Read missing code 999    -> status "
               WS-FILE-STATUS.

           CLOSE DYN-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Write code 500          -> status 00
Write code 300 (< 500)  -> status 21
Write duplicate code 500 -> status 22
Read missing code 999    -> status 23
```

### อธิบายจุดสำคัญ

- โปรแกรมนี้ใช้ไฟล์ **สองไฟล์แยกกัน** โดยตั้งใจ (`SEQ-FILE` กับ `DYN-FILE`) เพราะสถานะ `21`
  ต้องการ `ACCESS MODE IS SEQUENTIAL` เท่านั้น (มีแนวคิดเรื่อง "ลำดับ" ชัดเจน) ในขณะที่สถานะ `22`
  และ `23` ต้องการ `ACCESS MODE IS DYNAMIC` (หรือ `RANDOM`) เพื่อเข้าถึงด้วย Key โดยตรง ทั้งสองโหมด
  ให้ผลต่างกันเมื่อเรียก `WRITE`/`READ` แบบเดียวกัน — เป็นข้อพิสูจน์สำคัญว่า **สถานะที่ได้ขึ้นอยู่กับ
  `ACCESS MODE` ที่ประกาศไว้ ไม่ใช่แค่เนื้อหาของ record**
- ทั้ง `21`, `22`, `23` ล้วนเป็นข้อผิดพลาดที่โปรแกรมควรตรวจจับได้ตั้งแต่ก่อนเขียนโปรแกรมจริง เพราะ
  มักมีสาเหตุจากการออกแบบข้อมูลนำเข้าที่ไม่ผ่านการตรวจสอบ (Validation) มาก่อน

### ข้อควรระวัง

- อย่าสับสนระหว่าง `21` (ปัญหาตอน**เขียน**ลำดับผิด) กับ `23` (ปัญหาตอน**อ่าน/ค้นหา**ไม่พบ) แม้ทั้ง
  คู่จะเกี่ยวกับ Key ก็ตาม แต่เกิดจากปฏิบัติการคนละแบบ
- สถานะ `24` (boundary violation) พบได้ยากในทางปฏิบัติกับ GnuCOBOL บนดิสก์สมัยใหม่ เพราะข้อจำกัด
  เรื่องขนาดไฟล์ของระบบปฏิบัติการปัจจุบันกว้างขวางมาก มักพบใน Mainframe รุ่นเก่าที่มีการกำหนด
  ขอบเขตไฟล์ตายตัวไว้ล่วงหน้า (จะกล่าวถึงอีกครั้งเมื่อเรียน VSAM ใน Part 055)

### แบบฝึกหัดที่ 294.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมข้างต้นจึงต้องใช้ไฟล์ 2 ไฟล์แยกกัน (`SEQ-FILE` และ `DYN-FILE`)
แทนที่จะใช้ไฟล์เดียวสาธิตทั้ง 3 สถานะ

**เฉลย**: เพราะ `ACCESS MODE` เป็นคุณสมบัติที่ประกาศตายตัวใน `SELECT` และมีผลต่อพฤติกรรมของคำสั่ง
`WRITE`/`READ` ทั้งไฟล์ตลอดการรันโปรแกรม สถานะ `21` (ลำดับผิด) เกิดขึ้นเฉพาะเมื่อ `ACCESS MODE IS
SEQUENTIAL` เท่านั้น เพราะเป็นโหมดเดียวที่มีกฎ "ต้องเขียนเรียงจากน้อยไปมาก" ในขณะที่สถานะ `22`
และ `23` ต้องการความสามารถในการเข้าถึงด้วย Key โดยตรง (`ACCESS MODE IS DYNAMIC`) ถ้าใช้ไฟล์เดียว
กับ `ACCESS MODE` เดียว จะไม่สามารถสาธิตทั้ง 3 สถานการณ์ให้ครบถ้วนถูกต้องได้ในโปรแกรมเดียวกัน

---

## ขั้นตอนที่ 295: หมวด 3x — ตระกูลข้อผิดพลาดถาวร (35, 38)

### ข้อผิดพลาดที่โปรแกรมแก้ไขเองไม่ได้

รหัสในหมวด `3x` บอกปัญหาระดับไฟล์/สภาพแวดล้อมที่**โปรแกรมไม่สามารถแก้ไขได้ด้วยตัวเอง** ต้องอาศัย
คนหรือกระบวนการภายนอก (เช่น ตรวจสอบว่าไฟล์ input มาถึงหรือยัง)

| รหัส | ความหมาย | เคยเห็นครั้งแรกใน |
|---|---|---|
| `35` | เปิดไฟล์ (`INPUT`/`I-O`) ที่ไม่มีอยู่จริง โดยไม่มี `OPTIONAL` | Part 023 ขั้นตอน 229 |
| `37` | โหมด `OPEN` ขัดแย้งกับคุณสมบัติของไฟล์/อุปกรณ์ (พบยาก อธิบายเชิงทฤษฎี) | — |
| `38` | เปิดไฟล์ที่เคยถูก `CLOSE ... WITH LOCK` ไปแล้วในรันเดียวกัน | Part 025 ขั้นตอน 242 |
| `39` | คุณสมบัติไฟล์ขัดแย้งกัน (เช่น ขนาด record ไม่ตรงกับตอนสร้างไฟล์เดิม) | — |

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP295-PERMANENT-ERROR-FAMILY.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MISSING-FILE ASSIGN TO "MISSING295.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-FILE-STATUS.

           SELECT LOG-FILE ASSIGN TO "LOG295.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-LOG-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  MISSING-FILE.
       01  MISSING-RECORD           PIC X(10).

       FD  LOG-FILE.
       01  LOG-RECORD               PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.
       01  WS-LOG-STATUS            PIC XX.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Status 35 - the file genuinely does not exist, AND the
      *> SELECT clause has NO "OPTIONAL" to protect the OPEN. This
      *> is a PERMANENT error - the program should not try again
      *> without a human fixing the underlying problem.
           OPEN INPUT MISSING-FILE.
           DISPLAY "OPEN missing (no OPTIONAL) -> status "
               WS-FILE-STATUS.

      *> Status 38 - the file WAS opened and closed WITH LOCK
      *> earlier in THIS SAME run, and something now tries to
      *> OPEN it again. Recap of Part 025 step 242, revisited here
      *> as part of the 3x permanent-error family.
           OPEN OUTPUT LOG-FILE.
           MOVE "FINAL LINE" TO LOG-RECORD.
           WRITE LOG-RECORD.
           CLOSE LOG-FILE WITH LOCK.
           DISPLAY "CLOSE WITH LOCK            -> status "
               WS-LOG-STATUS.

           OPEN INPUT LOG-FILE.
           DISPLAY "Re-OPEN after WITH LOCK    -> status "
               WS-LOG-STATUS.

           STOP RUN.
```

**ผลลัพธ์:**

```
OPEN missing (no OPTIONAL) -> status 35
CLOSE WITH LOCK            -> status 00
Re-OPEN after WITH LOCK    -> status 38
```

### ค้นพบสำคัญ: FILE STATUS ป้องกันโปรแกรมล่มได้จริง

จุดที่น่าสังเกตที่สุดในขั้นตอนนี้คือบรรทัดแรก: **`OPEN INPUT MISSING-FILE` ที่ไฟล์ไม่มี `OPTIONAL`
และไฟล์ไม่มีอยู่จริง กลับ "ไม่ทำให้โปรแกรมล่ม"** ทั้งที่ Part 023 ขั้นตอน 229 เคยพิสูจน์ว่าสถานการณ์
เดียวกันนี้**ทำให้โปรแกรมล่มทันที**ด้วยข้อความ `libcob: error: file does not exist` — ความแตกต่าง
คือครั้งนี้เรามี `FILE STATUS IS WS-FILE-STATUS` ประกาศไว้ **การมี `FILE STATUS` เพียงอย่างเดียว
(แม้ไม่มี `OPTIONAL`) ก็เพียงพอที่จะป้องกันโปรแกรมจากการล่มในสถานการณ์นี้** — COBOL จะรายงานสถานะ
`35` ให้ตรวจสอบแทนที่จะปล่อยให้ runtime error หยุดโปรแกรมทันที

### อธิบายจุดสำคัญ

- นี่คือเหตุผลสำคัญที่สุดว่าทำไมโปรแกรมระดับองค์กรควรมี `FILE STATUS` แทบทุกไฟล์: มันเปลี่ยน
  "ข้อผิดพลาดร้ายแรงที่ทำให้โปรแกรมหยุดทำงานทันที" ให้กลายเป็น**ข้อมูลที่โปรแกรมนำไปตัดสินใจต่อได้**
  โดยไม่ล่ม
- `38` ยังคงทำงานเหมือนที่ Part 025 อธิบายไว้ทุกประการ: การ `CLOSE ... WITH LOCK` เป็นการล็อกไฟล์
  ถาวรสำหรับรันนั้น การพยายามเปิดซ้ำจะได้สถานะ `38` เสมอ

### ข้อควรระวัง

- แม้ `FILE STATUS` จะป้องกันโปรแกรมจากการล่มในกรณี `OPEN` ล้มเหลว **แต่โปรแกรมยังคงต้องตรวจสอบ
  ค่าสถานะและตัดสินใจเองว่าจะทำอย่างไรต่อ** ถ้าไม่ตรวจสอบเลยและปล่อยให้โปรแกรมทำงานต่อราวกับ `OPEN`
  สำเร็จ (เช่น ไปเรียก `READ` ต่อทันที) จะได้ผลลัพธ์ที่ผิดพลาดแบบเงียบ ๆ แทน (เช่น สถานะ `47`
  ตามที่จะเห็นในขั้นตอนที่ 296)
- รหัส `37` และ `39` พบได้ยากในสภาพแวดล้อมทดสอบทั่วไป (มักเกี่ยวข้องกับอุปกรณ์จัดเก็บข้อมูลเฉพาะทาง
  หรือการเปลี่ยนคุณสมบัติไฟล์ระหว่างรัน) การทดสอบในเอกสารนี้ไม่สามารถ trigger รหัสทั้งสองได้อย่าง
  น่าเชื่อถือด้วย GnuCOBOL บนไฟล์ระบบทั่วไป จึงกล่าวถึงในเชิงทฤษฎีเท่านั้น

### แบบฝึกหัดที่ 295.1

**โจทย์**: จงอธิบายว่าทำไมการมี `FILE STATUS IS WS-FILE-STATUS` เพียงอย่างเดียว (โดยไม่มี
`OPTIONAL`) จึงเปลี่ยนพฤติกรรมของ `OPEN INPUT` บนไฟล์ที่ไม่มีอยู่จริง จากการล่มโปรแกรม (Part 023
ขั้นตอน 229) กลายเป็นแค่รายงานสถานะ `35`

**เฉลย**: เพราะ GnuCOBOL (และ COBOL compiler ส่วนใหญ่) ถือว่า **การที่โปรแกรมประกาศ `FILE STATUS`
ไว้ คือสัญญาณว่าโปรแกรมเมอร์ตั้งใจจะ "รับผิดชอบตรวจสอบข้อผิดพลาดเอง"** จึงไม่จำเป็นต้องหยุดโปรแกรม
ทันทีด้วย runtime error อีกต่อไป เพียงแค่บันทึกรหัสสถานะไว้ในตัวแปรที่ระบุแล้วปล่อยให้โปรแกรม
ทำงานต่อไปตามปกติ (ตัวโปรแกรมเมอร์เป็นผู้ตัดสินใจเองว่าจะเช็คค่านั้นและทำอะไรต่อ) ต่างจากกรณีที่ไม่มี
`FILE STATUS` เลย ซึ่ง COBOL ถือว่าโปรแกรมเมอร์ไม่ได้เตรียมรับมือกับข้อผิดพลาดใด ๆ จึงเลือกที่จะ
หยุดโปรแกรมทันทีเพื่อป้องกันความเสียหายที่อาจร้ายแรงกว่าจากการทำงานต่อด้วยข้อมูลที่ไม่สมบูรณ์

---

## ขั้นตอนที่ 296: หมวด 4x — ตระกูลข้อผิดพลาดเชิงตรรกะ (41, 42, 47, 48, 49)

### เมื่อโปรแกรมเรียกคำสั่งผิดจังหวะ

รหัสในหมวด `4x` ล้วนเกิดจาก**โปรแกรมเมอร์เขียนโค้ดเรียกคำสั่งผิดจังหวะ** ไม่ใช่ปัญหาข้อมูลหรือ
สภาพแวดล้อม — เป็นสัญญาณของบั๊กในโค้ดเองที่ควรแก้ไขที่ต้นเหตุ ไม่ใช่แค่ดักจับสถานะไว้เฉย ๆ

| รหัส | ความหมาย |
|---|---|
| `41` | `OPEN` ไฟล์ที่**เปิดค้างอยู่แล้ว** |
| `42` | `CLOSE` ไฟล์ที่**ไม่ได้เปิดอยู่** |
| `43` | `REWRITE`/`DELETE` โดยไม่มี `READ` ที่ถูกต้องมาก่อน (โหมด `SEQUENTIAL`) |
| `46` | `READ` ซ้ำหลังจากได้ `AT END` แล้ว (อธิบายไปแล้วในขั้นตอนที่ 293) |
| `47` | `READ` ไฟล์ที่**ไม่ได้เปิดในโหมด `INPUT`/`I-O`** |
| `48` | `WRITE` ไฟล์ที่**ไม่ได้เปิดในโหมด `OUTPUT`/`I-O`/`EXTEND`** |
| `49` | `REWRITE`/`DELETE` ไฟล์ที่**ไม่ได้เปิดในโหมด `I-O`** |

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP296-LOGIC-ERROR-FAMILY.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT DATA-FILE ASSIGN TO "DATA296.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  DATA-FILE.
       01  DATA-RECORD              PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Status 41 - OPEN attempted on a file that is ALREADY open.
           OPEN OUTPUT DATA-FILE.
           OPEN OUTPUT DATA-FILE.
           DISPLAY "OPEN an already-open file -> status "
               WS-FILE-STATUS.

           MOVE "HELLO" TO DATA-RECORD.
           WRITE DATA-RECORD.
           CLOSE DATA-FILE.

      *> Status 42 - CLOSE attempted on a file that is NOT open.
           CLOSE DATA-FILE.
           DISPLAY "CLOSE an already-closed file -> status "
               WS-FILE-STATUS.

      *> Status 47 - READ attempted on a file opened for OUTPUT
      *> (not INPUT or I-O), so reading is not allowed.
           OPEN OUTPUT DATA-FILE.
           READ DATA-FILE.
           DISPLAY "READ a file opened OUTPUT -> status "
               WS-FILE-STATUS.
           CLOSE DATA-FILE.

      *> Status 48 - WRITE attempted on a file opened for INPUT
      *> (not OUTPUT, I-O or EXTEND), so writing is not allowed.
           OPEN INPUT DATA-FILE.
           WRITE DATA-RECORD.
           DISPLAY "WRITE a file opened INPUT -> status "
               WS-FILE-STATUS.
           CLOSE DATA-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
OPEN an already-open file -> status 41
CLOSE an already-closed file -> status 42
READ a file opened OUTPUT -> status 47
WRITE a file opened INPUT -> status 48
```

### อธิบายจุดสำคัญ

- ทุกสถานการณ์ในโปรแกรมนี้คือ**การเรียกคำสั่งผิดโหมดที่เปิดไว้** ตรงกับกฎเหล็กที่ Part 025 ขั้นตอน
  241 สอนไว้ว่า **"โหมดที่เปิดกำหนดว่าใช้คำสั่งอะไรได้บ้างอย่างเข้มงวด"** — ขั้นตอนนี้พิสูจน์กฎนั้น
  ด้วยรหัสสถานะที่ชัดเจนแทนคำอธิบายลอย ๆ
- สังเกตว่าโปรแกรมยัง**ไม่ล่ม**แม้จะเรียกคำสั่งผิดจังหวะถึง 4 ครั้งติดกัน เพราะมี `FILE STATUS`
  คอยรับค่าผลลัพธ์ไว้เสมอ (ย้ำแนวคิดจากขั้นตอนที่ 295) แต่ **การไม่ล่มไม่ได้แปลว่าปฏิบัติการนั้น
  สำเร็จ** เช่น `WRITE DATA-RECORD` ตอนเปิด `INPUT` จะไม่มีอะไรถูกเขียนจริงเลย
- ไม่ได้สาธิตสถานะ `43` (REWRITE/DELETE ไม่มี READ ก่อนในโหมด SEQUENTIAL) ในโปรแกรมนี้เพราะได้
  พิสูจน์แล้วอย่างละเอียดผ่านหลักการ `REWRITE` ต้อง `READ` ก่อนเสมอใน Part 025–029 (ทดสอบยืนยันว่า
  ให้ผลสถานะ `43` จริงเมื่อฝ่าฝืนกฎนี้ในโหมด `ACCESS SEQUENTIAL`)

### ข้อควรระวัง

- ข้อผิดพลาดในหมวด `4x` **ควรถูกกำจัดออกไปตั้งแต่ขั้นตอนพัฒนา/ทดสอบโปรแกรม** ไม่ใช่ปล่อยให้เกิด
  ในระบบจริงแล้วค่อยจัดการ เพราะเป็นสัญญาณของบั๊กในโค้ด ไม่ใช่ปัญหาข้อมูลหรือสภาพแวดล้อมที่หลีก
  เลี่ยงไม่ได้เหมือนหมวด `2x`/`3x`
- การเขียน `IF WS-FILE-STATUS NOT = "00"` แบบกว้าง ๆ ครอบคลุมทุกกรณีอาจซ่อนบั๊กในหมวด `4x` ไว้
  โดยไม่รู้ตัว (โปรแกรมทำงานต่อได้แต่ไม่ได้ผลลัพธ์ตามที่ต้องการ) ควรพิจารณาแยกจัดการแต่ละหมวดตาม
  ความเหมาะสม เช่น หมวด `4x` ควรบันทึก log และแจ้งเตือนทีมพัฒนาทันที ไม่ใช่แค่ข้ามไปเงียบ ๆ

### แบบฝึกหัดที่ 296.1

**โจทย์**: จงอธิบายว่าทำไมข้อผิดพลาดในหมวด `4x` จึงถูกจัดเป็น "Logic Error" ในขณะที่หมวด `3x` ถูก
จัดเป็น "Permanent Error" ทั้งที่ทั้งคู่ดู "ร้ายแรง" พอ ๆ กัน

**เฉลย**: เพราะสาเหตุของทั้งสองหมวดต่างกันโดยพื้นฐาน หมวด `3x` (Permanent Error) เกิดจากปัจจัย
**ภายนอกโปรแกรม** ที่ตัวโค้ดควบคุมไม่ได้โดยตรง เช่น ไฟล์ไม่มีอยู่จริงเพราะยังไม่ถูกส่งมา หรือ
ปัญหาที่อุปกรณ์จัดเก็บข้อมูล ซึ่งแก้ไขได้ด้วยการรอหรือแก้ปัญหาสภาพแวดล้อม ไม่ใช่แก้โค้ด ในขณะที่
หมวด `4x` (Logic Error) เกิดจาก**ตัวโค้ดเองเรียกคำสั่งผิดลำดับหรือผิดเงื่อนไข** เช่น `WRITE` ไฟล์
ที่เปิดโหมด `INPUT` ไว้ ซึ่งเป็นสิ่งที่**แก้ไขได้แน่นอนด้วยการแก้โค้ดให้ถูกต้อง** ไม่ขึ้นกับปัจจัย
ภายนอกใด ๆ เลย นี่คือเหตุผลที่ทั้งสองหมวดถูกแยกออกจากกันอย่างชัดเจนตามมาตรฐาน

---

## ขั้นตอนที่ 297: ทำให้โค้ดอ่านง่ายขึ้นด้วย 88-level Condition Names

### ปัญหา: Magic Number ในโค้ดตรวจสอบสถานะ

การเขียน `IF WS-FILE-STATUS = "22"` ซ้ำ ๆ ทั่วทั้งโปรแกรมทำให้โค้ดอ่านยากและเสี่ยงพิมพ์ผิด
เทคนิคที่ COBOL มีให้ตั้งแต่ Part 010 (Condition Names หรือ 88-level) สามารถนำมาใช้ตั้งชื่อที่
มีความหมายให้กับค่าสถานะแต่ละกลุ่มได้ ทำให้โค้ดอ่านง่ายขึ้นมาก

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP297-STATUS-CONDITION-NAMES.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PRODUCT-FILE ASSIGN TO "PROD297.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS PR-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-FILE.
       01  PRODUCT-RECORD.
           05  PR-CODE              PIC 9(3).
           05  PR-NAME              PIC X(10).

       WORKING-STORAGE SECTION.
      *> Naming every meaningful status value with an 88-level
      *> condition turns "IF WS-FILE-STATUS = '22'" (a magic
      *> number) into "IF FS-DUPLICATE-KEY" (self-documenting).
       01  WS-FILE-STATUS           PIC XX.
           88  FS-SUCCESS           VALUE "00" "02".
           88  FS-END-OF-FILE       VALUE "10".
           88  FS-NOT-FOUND         VALUE "23".
           88  FS-DUPLICATE-KEY     VALUE "22".
           88  FS-FILE-MISSING      VALUE "35".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT PRODUCT-FILE.
           MOVE 100 TO PR-CODE.
           MOVE "RICE" TO PR-NAME.
           WRITE PRODUCT-RECORD.
           PERFORM SHOW-WRITE-RESULT.
           CLOSE PRODUCT-FILE.

           OPEN I-O PRODUCT-FILE.
           MOVE 100 TO PR-CODE.
           MOVE "IMPOSTOR" TO PR-NAME.
           WRITE PRODUCT-RECORD.
           PERFORM SHOW-WRITE-RESULT.

           MOVE 999 TO PR-CODE.
           READ PRODUCT-FILE.
           PERFORM SHOW-READ-RESULT.

           CLOSE PRODUCT-FILE.
           STOP RUN.

       SHOW-WRITE-RESULT.
           EVALUATE TRUE
               WHEN FS-SUCCESS
                   DISPLAY "WRITE ok."
               WHEN FS-DUPLICATE-KEY
                   DISPLAY "WRITE rejected - code already exists."
               WHEN OTHER
                   DISPLAY "WRITE failed - status " WS-FILE-STATUS
           END-EVALUATE.

       SHOW-READ-RESULT.
           EVALUATE TRUE
               WHEN FS-SUCCESS
                   DISPLAY "READ ok: " PR-NAME
               WHEN FS-NOT-FOUND
                   DISPLAY "READ - no such product code."
               WHEN FS-END-OF-FILE
                   DISPLAY "READ - reached end of file."
               WHEN OTHER
                   DISPLAY "READ failed - status " WS-FILE-STATUS
           END-EVALUATE.
```

**ผลลัพธ์:**

```
WRITE ok.
WRITE rejected - code already exists.
READ - no such product code.
```

### อธิบายจุดสำคัญ

- `88 FS-SUCCESS VALUE "00" "02".`: หนึ่ง condition name สามารถครอบคลุม**หลายค่า**ได้พร้อมกัน
  (ทบทวนจาก Part 010) — ในที่นี้รวมทั้ง `00` (สำเร็จธรรมดา) และ `02` (สำเร็จแบบมี duplicate
  alternate key ที่อนุญาต) เข้าเป็นแนวคิดเดียวว่า "สำเร็จ" เพราะโปรแกรมนี้ไม่สนใจแยกความแตกต่าง
  ระหว่างสองกรณีนี้
- `EVALUATE TRUE` ร่วมกับ 88-level (เทคนิคจาก Part 011) ทำให้โค้ดอ่านเหมือนประโยคภาษาอังกฤษ:
  "WHEN FS-DUPLICATE-KEY" อ่านเข้าใจได้ทันทีโดยไม่ต้องจำว่า `22` แปลว่าอะไร
- เทคนิคนี้ทำให้การแก้ไข/ขยายโค้ดในอนาคตง่ายขึ้นมาก เพราะถ้าต้องเพิ่มการตรวจสอบสถานะใหม่ ก็แค่
  เพิ่ม 88-level ใหม่ที่จุดประกาศเดียว ไม่ต้องไล่หา magic number ทั่วทั้งโปรแกรม

### ข้อควรระวัง

- ต้องระวังไม่ให้ค่าใน 88-level ซ้อนทับกัน (overlap) โดยไม่ตั้งใจ เช่น ถ้ามี `88 FS-SUCCESS VALUE
  "00" "02"` และ `88 FS-DUPLICATE-KEY VALUE "02"` พร้อมกัน ค่า `"02"` จะทำให้ทั้งสอง condition
  เป็นจริงพร้อมกัน ซึ่งอาจทำให้ผลลัพธ์ของ `EVALUATE TRUE` ขึ้นกับลำดับ `WHEN` ที่เขียนไว้เท่านั้น
  (ตัวอย่างนี้จงใจไม่ใช้ `02` ซ้ำในสอง 88-level เพื่อหลีกเลี่ยงความสับสนนี้)
- 88-level เหล่านี้ผูกกับ `WS-FILE-STATUS` ของไฟล์**เฉพาะที่ประกาศไว้ในโปรแกรมนี้เท่านั้น** ถ้ามี
  หลายไฟล์ในโปรแกรมเดียวกัน (เหมือนขั้นตอนที่ 294) แต่ละไฟล์ต้องมีชุด 88-level ของตัวเอง หรือใช้
  ตัวแปร `WS-FILE-STATUS` ร่วมกันถ้าต้องการใช้ชุด 88-level เดียว (ต้องระวังเรื่องการเขียนทับค่ากัน)

### แบบฝึกหัดที่ 297.1

**โจทย์**: จงเพิ่ม 88-level ชื่อ `FS-SEQUENCE-ERROR` สำหรับสถานะ `21` และแก้ `SHOW-WRITE-RESULT`
ให้แสดงข้อความ `"WRITE rejected - keys must be written in order."` เมื่อเจอสถานะนี้

**เฉลย**: เพิ่มการประกาศ 88-level:

```cobol
           88  FS-SEQUENCE-ERROR    VALUE "21".
```

แล้วเพิ่ม `WHEN` ใหม่ใน `SHOW-WRITE-RESULT` **ก่อน** `WHEN OTHER`:

```cobol
               WHEN FS-SEQUENCE-ERROR
                   DISPLAY "WRITE rejected - keys must be "
                       "written in order."
```

---

## ขั้นตอนที่ 298: การตรวจสอบแบบทั่วไปด้วยหลักแรกของ FILE STATUS

### เมื่อไม่จำเป็นต้องรู้รหัสละเอียด แค่รู้ "หมวดหมู่" ก็พอ

บางสถานการณ์โปรแกรมไม่จำเป็นต้องแยกแยะรหัสละเอียดทุกตัว แค่ต้องการรู้ว่า **"อยู่ในหมวดไหน"**
(สำเร็จ/จบไฟล์/Key ผิด/ผิดพลาดถาวร/ตรรกะผิด) ก็เพียงพอสำหรับตัดสินใจแล้ว เทคนิคนี้ใช้ **Reference
Modification** (ทบทวนจาก Part 006, 019) ดึงแค่ตัวอักษรตัวแรกของ `WS-FILE-STATUS` ออกมาตรวจสอบ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP298-GENERIC-CATEGORY-CHECK.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PRODUCT-FILE ASSIGN TO "PROD298.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS PR-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-FILE.
       01  PRODUCT-RECORD.
           05  PR-CODE              PIC 9(3).
           05  PR-NAME              PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.
       01  WS-STATUS-CATEGORY       PIC X.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT PRODUCT-FILE.
           MOVE 100 TO PR-CODE.
           MOVE "RICE" TO PR-NAME.
           WRITE PRODUCT-RECORD.
           PERFORM REPORT-STATUS-CATEGORY.
           CLOSE PRODUCT-FILE.
           STOP RUN.

      *> One generic paragraph, reused for EVERY file operation in
      *> the program, no matter which file or which exact status -
      *> this is the pattern real COBOL shops build a whole file
      *> I/O utility copybook around (COPY is taught in Part 033).
       REPORT-STATUS-CATEGORY.
      *> WS-FILE-STATUS (1:1) is reference modification (Part 019
      *> STRING/Part 006 review): it takes just the FIRST character
      *> of the 2-character status field, which is the category
      *> digit per the COBOL standard.
           MOVE WS-FILE-STATUS (1:1) TO WS-STATUS-CATEGORY.
           EVALUATE WS-STATUS-CATEGORY
               WHEN "0"
                   DISPLAY "[" WS-FILE-STATUS "] category 0x: "
                       "success"
               WHEN "1"
                   DISPLAY "[" WS-FILE-STATUS "] category 1x: "
                       "end of file"
               WHEN "2"
                   DISPLAY "[" WS-FILE-STATUS "] category 2x: "
                       "invalid key"
               WHEN "3"
                   DISPLAY "[" WS-FILE-STATUS "] category 3x: "
                       "permanent error"
               WHEN "4"
                   DISPLAY "[" WS-FILE-STATUS "] category 4x: "
                       "logic error"
               WHEN OTHER
                   DISPLAY "[" WS-FILE-STATUS "] category "
                       WS-STATUS-CATEGORY "x: implementor-defined"
           END-EVALUATE.
```

**ผลลัพธ์:**

```
[00] category 0x: success
```

### อธิบายจุดสำคัญ

- `WS-FILE-STATUS (1:1)` คือ Reference Modification: `(ตำแหน่งเริ่มต้น:ความยาว)` — `(1:1)` แปลว่า
  "เริ่มจากตัวที่ 1 ความยาว 1 ตัวอักษร" ก็คือตัวอักษรตัวแรกนั่นเอง เทคนิคนี้ใช้ได้กับฟิลด์
  alphanumeric ทุกตัวไม่จำกัดเฉพาะ `FILE STATUS`
- ข้อดีของแนวทางนี้คือ **paragraph เดียวใช้ได้กับทุกไฟล์และทุกสถานะในโปรแกรม** ไม่ต้องเขียน
  `EVALUATE` แยกสำหรับทุกปฏิบัติการเหมือนขั้นตอนที่ 297 — เหมาะกับการเขียน**รายงาน log** หรือ
  **debug trace** ที่ต้องการเห็นภาพรวมของทุกปฏิบัติการโดยไม่สนใจรายละเอียดปลีกย่อย
- โปรแกรมนี้เตือนความจำสำคัญเรื่อง**การไหลของโปรแกรม (Control Flow)**: ถ้าลืม `STOP RUN` ก่อน
  paragraph อื่น โปรแกรมจะ "ไหลตก" (Fall Through) เข้าไปทำงานใน paragraph ถัดไปโดยไม่ได้ตั้งใจ
  ทันที (ทบทวนโครงสร้าง Section/Paragraph จาก Part 014)

### ข้อควรระวัง

- **ต้องมี `STOP RUN` ปิดท้าย `MAIN-PARA` เสมอ** ก่อนที่จะมี paragraph อื่นตามมา มิฉะนั้นโปรแกรม
  จะไหลตกเข้าไปทำงานใน paragraph ถัดไปโดยอัตโนมัติ (ยืนยันจากการทดลองจริงระหว่างพัฒนาตัวอย่างนี้:
  ถ้าลืม `STOP RUN` ก่อน `REPORT-STATUS-CATEGORY` โปรแกรมจะเรียก paragraph นั้นซ้ำอีกครั้งโดยไม่ได้
  ตั้งใจหลัง `CLOSE`)
- Reference Modification `(1:1)` ใช้ได้เฉพาะดึงตัวอักษรตัวแรกของฟิลด์ 2 ตัวเท่านั้น ถ้าต้องการ
  ตัวอักษรตัวที่สอง (หลักหน่วยของรหัส) ต้องใช้ `WS-FILE-STATUS (2:1)` แทน

### แบบฝึกหัดที่ 298.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมต้นฉบับ (ก่อนแก้ไข) ที่ลืมใส่ `STOP RUN` หลัง `CLOSE
PRODUCT-FILE.` ในบรรทัดแรกของ `MAIN-PARA` จึงแสดงผลลัพธ์ 4 บรรทัดแทนที่จะเป็น 3 บรรทัดตามที่ตั้งใจ
ทั้งที่มีการเรียก `PERFORM REPORT-STATUS-CATEGORY` แค่ 3 ครั้งในโค้ด

**เฉลย**: เพราะเมื่อ `MAIN-PARA` ทำงานมาถึงบรรทัดสุดท้าย (`CLOSE PRODUCT-FILE.`) โดยไม่มี
`STOP RUN` มาปิดท้าย COBOL จะไม่หยุดโปรแกรม แต่จะ**ไหลต่อเข้าไปยัง paragraph ถัดไปในลำดับที่เขียน
ไว้ในซอร์สโค้ดโดยอัตโนมัติ** (พฤติกรรมนี้เรียกว่า "Fall Through" ซึ่งทบทวนจาก Part 014) ซึ่งใน
กรณีนี้คือ `REPORT-STATUS-CATEGORY` ทำให้ paragraph นั้นถูกเรียกทำงานเป็นครั้งที่ 4 โดยไม่ได้ตั้งใจ
โดยใช้ค่า `WS-FILE-STATUS` ที่ค้างมาจากคำสั่ง `CLOSE` ก่อนหน้า (ซึ่งบังเอิญเป็น `"00"` เพราะ `CLOSE`
สำเร็จ) จึงเห็นบรรทัดที่ 4 เป็น `[00] category 0x: success` เพิ่มขึ้นมาโดยไม่มีการ `PERFORM`
เรียกมันตรง ๆ เลย

---

## ขั้นตอนที่ 299: รูปแบบการใช้งานจริง — OPTIONAL + FILE STATUS สำหรับไฟล์ที่อาจมาไม่ทัน

### สถานการณ์จริง: ไฟล์ Batch ที่อาจยังไม่มาถึง

ระบบ Batch Processing ในองค์กรจริงมักต้องรับมือกับสถานการณ์ **"ไฟล์ input วันนี้อาจยังไม่ถูกส่งมา"**
(เช่น รอไฟล์ธุรกรรมจากสาขาอื่นทาง FTP) ขั้นตอนนี้แสดง Pattern มาตรฐานที่ผสาน `OPTIONAL` (Part 023)
เข้ากับ `FILE STATUS` (Part นี้) เพื่อแยกแยะ **"ไฟล์ไม่มาวันนี้ (ปกติ)"** ออกจาก **"เกิดข้อผิดพลาด
ไม่คาดคิด (ผิดปกติ)"** อย่างชัดเจน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP299-OPTIONAL-STATUS-PATTERN.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT OPTIONAL TXN-FILE ASSIGN TO "TXN299.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  TXN-FILE.
       01  TXN-RECORD                PIC X(30).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS            PIC XX.
           88  FS-SUCCESS            VALUE "00".
           88  FS-END-OF-FILE        VALUE "10".
           88  FS-OPTIONAL-MISSING   VALUE "05".
       01  WS-EOF-FLAG               PIC X VALUE "N".
           88  END-OF-FILE           VALUE "Y".
       01  WS-TXN-COUNT              PIC 9(4) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> This is the standard, production-grade pattern for a file
      *> that legitimately may not exist yet (e.g. "today's batch
      *> file has not arrived"): OPTIONAL prevents a crash, and
      *> checking FS-OPTIONAL-MISSING right after OPEN tells the
      *> program exactly which of the two real situations it is
      *> in, instead of guessing from the record count alone.
           OPEN INPUT TXN-FILE.

           EVALUATE TRUE
               WHEN FS-SUCCESS
                   DISPLAY "Transaction file found - processing."
                   PERFORM PROCESS-ALL-TRANSACTIONS
               WHEN FS-OPTIONAL-MISSING
                   DISPLAY "No transaction file today - that is"
                   DISPLAY "fine, nothing to process."
               WHEN OTHER
                   DISPLAY "Unexpected error opening file! "
                       "Status = " WS-FILE-STATUS
                   DISPLAY "Aborting - a human needs to check "
                       "this."
           END-EVALUATE.

           CLOSE TXN-FILE.
           DISPLAY "Total transactions processed: "
               WS-TXN-COUNT.
           STOP RUN.

       PROCESS-ALL-TRANSACTIONS.
           PERFORM UNTIL END-OF-FILE
               READ TXN-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       ADD 1 TO WS-TXN-COUNT
                       DISPLAY "  " TXN-RECORD
               END-READ
           END-PERFORM.
```

**ผลลัพธ์เมื่อไฟล์ยังไม่มาถึง:**

```
No transaction file today - that is
fine, nothing to process.
Total transactions processed: 0000
```

**ผลลัพธ์เมื่อไฟล์มาถึงแล้ว** (มี 2 บรรทัดข้อมูลอยู่ใน `TXN299.DAT`):

```
Transaction file found - processing.
  0001 DEPOSIT 500.00
  0002 WITHDRAW 200.00
Total transactions processed: 0002
```

### อธิบายจุดสำคัญ

- `EVALUATE TRUE` ตรวจสอบทันทีหลัง `OPEN` **ก่อน**ที่จะเข้าสู่ลูปประมวลผลใด ๆ — เป็นรูปแบบที่
  แนะนำอย่างยิ่งเพราะแยกแยะ "เหตุผลที่ไม่มีข้อมูลให้ประมวลผล" ได้ชัดเจนตั้งแต่ต้น แทนที่จะปล่อยให้
  ลูปทำงานแล้วค่อยมานับ `WS-TXN-COUNT = 0` ทีหลัง (ซึ่งบอกไม่ได้ว่า "ไฟล์ไม่มา" หรือ "ไฟล์มาแต่ว่าง
  เปล่า" ต่างกัน)
- สาขา `WHEN OTHER` สำคัญมาก: มันดักจับสถานะที่**ไม่คาดคิดเลย** เช่น `30` (permanent error อื่น ๆ)
  ทำให้โปรแกรมไม่ทึกทักเอาว่า "ไม่ใช่ 00 ก็คือไฟล์ไม่มา" ซึ่งเป็นข้อผิดพลาดเชิงตรรกะที่พบบ่อยมาก
  ในโปรแกรมที่เขียนแบบไม่รอบคอบ
- Pattern นี้คือรากฐานของสิ่งที่ Part 048 จะสอนต่อยอดด้วย `DECLARATIVES`/`USE` Statement (การ
  ดักจับแบบรวมศูนย์อัตโนมัติ แทนการเขียน `EVALUATE` ทุกจุดด้วยมือ)

### ข้อควรระวัง

- อย่าลืมว่า `FS-OPTIONAL-MISSING VALUE "05"` ใช้ได้เฉพาะเมื่อ `SELECT` มีคำว่า `OPTIONAL` เท่านั้น
  ถ้าลืมใส่ `OPTIONAL` ไฟล์ที่ไม่มีอยู่จริงจะได้สถานะ `35` แทน (ตามที่พิสูจน์ในขั้นตอนที่ 295) ไม่ใช่
  `05`
- ควรทดสอบโปรแกรมแบบนี้ทั้งสองสถานการณ์เสมอ (ไฟล์มี/ไฟล์ไม่มี) ก่อนนำไปใช้งานจริง เพราะเป็นเงื่อนไข
  ที่แยกออกจากกันโดยสิ้นเชิงในโค้ด การทดสอบแค่กรณีเดียวอาจพลาดบั๊กในอีกกรณีได้ง่าย

### แบบฝึกหัดที่ 299.1

**โจทย์**: จงอธิบายว่าทำไมการนับ `WS-TXN-COUNT = 0` เพียงอย่างเดียว (โดยไม่ตรวจสอบ `FILE STATUS`
หลัง `OPEN`) จึงไม่เพียงพอที่จะแยกแยะระหว่าง "ไฟล์ไม่มาวันนี้" กับ "ไฟล์มาถึงแล้วแต่ไม่มีธุรกรรมเลย"

**เฉลย**: เพราะทั้งสองสถานการณ์ให้ผลลัพธ์ `WS-TXN-COUNT = 0` เหมือนกันทุกประการเมื่อดูจากผลลัพธ์
ปลายทางเพียงอย่างเดียว แต่ความหมายทางธุรกิจต่างกันโดยสิ้นเชิง: "ไฟล์ไม่มาวันนี้" อาจหมายถึงปัญหาที่
ต้นทาง (สาขาลืมส่งไฟล์ หรือระบบ FTP ขัดข้อง) ที่ควรแจ้งเตือนทีมปฏิบัติการ ในขณะที่ "ไฟล์มาถึงแล้ว
แต่ว่างเปล่า" อาจเป็นเรื่องปกติ (วันหยุดที่ไม่มีธุรกรรมเกิดขึ้นจริง) การตรวจสอบ `FILE STATUS`
ทันทีหลัง `OPEN` (ได้ `05` หรือ `00`) คือวิธีเดียวที่แยกแยะสองสถานการณ์นี้ออกจากกันได้อย่างแม่นยำ
ตั้งแต่ต้น ก่อนที่จะไปถึงขั้นตอนนับจำนวน record เลยด้วยซ้ำ

---

## ขั้นตอนที่ 300: โปรแกรมรวบยอด — ระบบปรับปรุงไฟล์หลักแบบทนทานต่อข้อผิดพลาด

### เป้าหมาย

ขั้นตอนสุดท้ายของ Part นี้ (และของทั้งกลุ่ม Part 023–030 เรื่องการประมวลผลไฟล์) รวมทุกอย่างเข้า
ด้วยกัน: อ่านไฟล์ธุรกรรม (แนวคิด Master-Detail จาก Part 026) มาปรับปรุงไฟล์หลักแบบ Indexed
(Part 028) ด้วยการ `WRITE`/`REWRITE`/`DELETE` พร้อมตรวจสอบ `FILE STATUS` อย่างเป็นระบบทุกจุด
(Part นี้) และบันทึกรายการที่ผิดพลาดลงไฟล์ error แยกต่างหาก แทนที่จะปล่อยให้โปรแกรมล่มหรือข้าม
ปัญหาไปเงียบ ๆ — นี่คือรูปแบบที่ใช้งานจริงในระบบประมวลผล Batch ระดับองค์กร

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP300-ROBUST-MASTER-UPDATE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT TXN-FILE ASSIGN TO "TXN300.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-TXN-STATUS.

           SELECT MASTER-FILE ASSIGN TO "MASTER300.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS MA-CODE
               FILE STATUS IS WS-MASTER-STATUS.

           SELECT ERROR-FILE ASSIGN TO "ERRORS300.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-ERROR-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  TXN-FILE.
       01  TXN-RECORD.
           05  TX-ACTION            PIC X(1).
           05  TX-CODE              PIC 9(3).
           05  TX-NAME              PIC X(10).
           05  TX-PRICE             PIC 9(5)V99.

       FD  MASTER-FILE.
       01  MASTER-RECORD.
           05  MA-CODE              PIC 9(3).
           05  MA-NAME              PIC X(10).
           05  MA-PRICE             PIC 9(5)V99.

       FD  ERROR-FILE.
       01  ERROR-RECORD             PIC X(60).

       WORKING-STORAGE SECTION.
       01  WS-TXN-STATUS            PIC XX.
       01  WS-MASTER-STATUS         PIC XX.
           88  MS-SUCCESS           VALUE "00" "02".
           88  MS-DUPLICATE-KEY     VALUE "22".
           88  MS-NOT-FOUND         VALUE "23".
       01  WS-ERROR-STATUS          PIC XX.
       01  WS-EOF-FLAG              PIC X VALUE "N".
           88  END-OF-TXN           VALUE "Y".
       01  WS-APPLIED-COUNT         PIC 9(4) VALUE 0.
       01  WS-ERROR-COUNT           PIC 9(4) VALUE 0.
       01  WS-ERROR-LINE.
           05  EL-CODE              PIC 9(3).
           05  FILLER               PIC X(1) VALUE SPACE.
           05  EL-ACTION            PIC X(1).
           05  FILLER               PIC X(1) VALUE SPACE.
           05  EL-REASON            PIC X(40).

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM SETUP-MASTER-FILE.
           PERFORM SETUP-TXN-FILE.

           OPEN I-O MASTER-FILE.
           OPEN INPUT TXN-FILE.
           OPEN OUTPUT ERROR-FILE.

           PERFORM UNTIL END-OF-TXN
               READ TXN-FILE
                   AT END
                       SET END-OF-TXN TO TRUE
                   NOT AT END
                       PERFORM APPLY-ONE-TRANSACTION
               END-READ
           END-PERFORM.

           CLOSE TXN-FILE.
           CLOSE MASTER-FILE.
           CLOSE ERROR-FILE.

           DISPLAY "---- Run summary ----".
           DISPLAY "Applied successfully: " WS-APPLIED-COUNT.
           DISPLAY "Rejected with errors: " WS-ERROR-COUNT.
           PERFORM PRINT-FINAL-MASTER.
           STOP RUN.

       SETUP-MASTER-FILE.
           OPEN OUTPUT MASTER-FILE.
           MOVE 100 TO MA-CODE.
           MOVE "RICE" TO MA-NAME.
           MOVE 350.00 TO MA-PRICE.
           WRITE MASTER-RECORD.
           MOVE 200 TO MA-CODE.
           MOVE "OIL" TO MA-NAME.
           MOVE 95.50 TO MA-PRICE.
           WRITE MASTER-RECORD.
           CLOSE MASTER-FILE.

       SETUP-TXN-FILE.
      *> A=Add, U=Update price, D=Delete. Transaction 3 deliberately
      *> adds a DUPLICATE code (100) and transaction 5 deliberately
      *> updates a code (999) that does not exist, to exercise the
      *> error path. Fields are MOVEd individually (not typed as one
      *> long literal) so column alignment can never be miscounted.
           OPEN OUTPUT TXN-FILE.

           MOVE "A" TO TX-ACTION.
           MOVE 300 TO TX-CODE.
           MOVE "NOODLE" TO TX-NAME.
           MOVE 6.00 TO TX-PRICE.
           WRITE TXN-RECORD.

           MOVE "U" TO TX-ACTION.
           MOVE 100 TO TX-CODE.
           MOVE "RICE" TO TX-NAME.
           MOVE 400.00 TO TX-PRICE.
           WRITE TXN-RECORD.

           MOVE "A" TO TX-ACTION.
           MOVE 100 TO TX-CODE.
           MOVE "RICE" TO TX-NAME.
           MOVE 400.00 TO TX-PRICE.
           WRITE TXN-RECORD.

           MOVE "D" TO TX-ACTION.
           MOVE 200 TO TX-CODE.
           MOVE "OIL" TO TX-NAME.
           MOVE 0 TO TX-PRICE.
           WRITE TXN-RECORD.

           MOVE "U" TO TX-ACTION.
           MOVE 999 TO TX-CODE.
           MOVE "GHOST" TO TX-NAME.
           MOVE 1.00 TO TX-PRICE.
           WRITE TXN-RECORD.

           CLOSE TXN-FILE.

       APPLY-ONE-TRANSACTION.
           MOVE TX-CODE TO MA-CODE.
           EVALUATE TX-ACTION
               WHEN "A"
                   MOVE TX-NAME TO MA-NAME
                   MOVE TX-PRICE TO MA-PRICE
                   WRITE MASTER-RECORD
                   PERFORM CHECK-MASTER-RESULT
               WHEN "U"
                   READ MASTER-FILE
                   IF MS-SUCCESS
                       MOVE TX-PRICE TO MA-PRICE
                       REWRITE MASTER-RECORD
                       PERFORM CHECK-MASTER-RESULT
                   ELSE
                       PERFORM CHECK-MASTER-RESULT
                   END-IF
               WHEN "D"
                   DELETE MASTER-FILE RECORD
                   PERFORM CHECK-MASTER-RESULT
               WHEN OTHER
                   MOVE TX-CODE TO EL-CODE
                   MOVE TX-ACTION TO EL-ACTION
                   MOVE "UNKNOWN ACTION CODE" TO EL-REASON
                   PERFORM WRITE-ERROR-LINE
           END-EVALUATE.

       CHECK-MASTER-RESULT.
           IF MS-SUCCESS
               ADD 1 TO WS-APPLIED-COUNT
           ELSE
               MOVE TX-CODE TO EL-CODE
               MOVE TX-ACTION TO EL-ACTION
               EVALUATE TRUE
                   WHEN MS-DUPLICATE-KEY
                       MOVE "DUPLICATE PRODUCT CODE" TO EL-REASON
                   WHEN MS-NOT-FOUND
                       MOVE "PRODUCT CODE NOT FOUND" TO EL-REASON
                   WHEN OTHER
                       STRING "UNEXPECTED STATUS "
                           WS-MASTER-STATUS
                           DELIMITED BY SIZE INTO EL-REASON
               END-EVALUATE
               PERFORM WRITE-ERROR-LINE
           END-IF.

       WRITE-ERROR-LINE.
           ADD 1 TO WS-ERROR-COUNT.
           MOVE WS-ERROR-LINE TO ERROR-RECORD.
           WRITE ERROR-RECORD.
           DISPLAY "  REJECTED: " ERROR-RECORD.

       PRINT-FINAL-MASTER.
           DISPLAY "---- Final master file ----".
           MOVE "N" TO WS-EOF-FLAG.
           OPEN INPUT MASTER-FILE.
           PERFORM UNTIL END-OF-TXN
               READ MASTER-FILE NEXT RECORD
                   AT END
                       SET END-OF-TXN TO TRUE
                   NOT AT END
                       DISPLAY "  " MA-CODE " " MA-NAME " "
                           MA-PRICE
               END-READ
           END-PERFORM.
           CLOSE MASTER-FILE.
```

**ผลลัพธ์:**

```
  REJECTED: 100 A DUPLICATE PRODUCT CODE
  REJECTED: 999 U PRODUCT CODE NOT FOUND
---- Run summary ----
Applied successfully: 0003
Rejected with errors: 0002
---- Final master file ----
  100 RICE       00400.00
  300 NOODLE     00006.00
```

### อธิบายภาพรวมของโปรแกรม

โปรแกรมนี้จำลองการรัน Batch Job ปรับปรุงไฟล์หลักสินค้าด้วยธุรกรรม 5 รายการ:

1. **เพิ่มสินค้า 300 (NOODLE)** — สำเร็จ
2. **แก้ไขราคาสินค้า 100 (RICE) เป็น 400.00** — สำเร็จ
3. **เพิ่มสินค้า 100 ซ้ำ** — ถูกปฏิเสธด้วยสถานะ `22` (Duplicate Key)
4. **ลบสินค้า 200 (OIL)** — สำเร็จ
5. **แก้ไขราคาสินค้า 999 ที่ไม่มีอยู่จริง** — ถูกปฏิเสธด้วยสถานะ `23` (Not Found)

สังเกตว่า `CHECK-MASTER-RESULT` เป็น**จุดตรวจสอบสถานะแบบรวมศูนย์**ที่ใช้ร่วมกันทั้ง 3 ปฏิบัติการ
(`WRITE`, `REWRITE`, `DELETE`) ทำให้ตรรกะการจัดการข้อผิดพลาดไม่กระจัดกระจายอยู่ทั่วโปรแกรม ตรงตาม
แนวคิดเรื่อง Single Responsibility ที่เน้นย้ำมาตั้งแต่ Part 014 และไฟล์ error (`ERRORS300.DAT`)
เก็บบันทึกทุกรายการที่ล้มเหลวไว้ให้ทีมปฏิบัติการตรวจสอบภายหลังได้ แทนที่จะหายไปเงียบ ๆ

### ข้อควรระวัง

- โปรแกรมนี้ยังคง**ประมวลผลธุรกรรมที่เหลือต่อไป**แม้จะเจอข้อผิดพลาดในบางรายการ (ธุรกรรมที่ 4 และ
  5 ยังคงถูกประมวลผลตามปกติแม้ธุรกรรมที่ 3 จะล้มเหลว) นี่คือการตัดสินใจเชิงออกแบบที่สำคัญ: บาง
  ระบบอาจต้องการ **"หยุดทั้ง batch ทันทีที่เจอข้อผิดพลาดแรก"** แทน ขึ้นอยู่กับความต้องการทางธุรกิจ
  ว่าข้อผิดพลาดบางส่วนยอมรับได้หรือไม่
- ไฟล์ error ในตัวอย่างนี้เป็นเพียงตัวอย่างพื้นฐาน (`PIC X(60)` ธรรมดา) ระบบจริงมักต้องการรายละเอียด
  เพิ่มเติม เช่น วันเวลาที่เกิดข้อผิดพลาด หรือรหัสสถานะดิบเก็บไว้ด้วยเพื่อการตรวจสอบย้อนหลัง

### แบบฝึกหัดที่ 300.1

**โจทย์**: จงอธิบายว่าทำไมการออกแบบให้ `CHECK-MASTER-RESULT` เป็น paragraph กลางที่ใช้ร่วมกันทั้ง
`WRITE`, `REWRITE`, และ `DELETE` จึงดีกว่าการเขียนตรวจสอบ `IF MS-SUCCESS ... ELSE ...` แยกกันสาม
ชุดในแต่ละ `WHEN` ของ `EVALUATE TX-ACTION`

**เฉลย**: เพราะตรรกะการตัดสินใจว่า "ควรทำอะไรเมื่อสำเร็จ" (`ADD 1 TO WS-APPLIED-COUNT`) และ
"ควรทำอะไรเมื่อล้มเหลว" (บันทึกลง error log พร้อมเหตุผล) เป็น**ตรรกะเดียวกันทุกประการ**ไม่ว่าจะเป็น
ปฏิบัติการ `WRITE`, `REWRITE`, หรือ `DELETE` ก็ตาม การรวมไว้ใน paragraph เดียวทำให้ถ้าในอนาคตต้อง
แก้ไขรูปแบบข้อความ error หรือเพิ่มการตรวจสอบสถานะใหม่ (เช่น เพิ่ม `24`) จะแก้ไขแค่**จุดเดียว**แทนที่
จะต้องไล่แก้ทั้ง 3 จุดที่กระจัดกระจายอยู่ ลดความเสี่ยงที่จะแก้ไม่ครบหรือแก้ไม่ตรงกันระหว่างจุดต่าง ๆ
ตรงตามหลักการ "Don't Repeat Yourself" ที่เป็นแนวปฏิบัติที่ดีในการเขียนโปรแกรมทุกภาษา

---

## สรุปท้ายบท

Part นี้ปิดท้ายกลุ่มเนื้อหาเรื่องการประมวลผลไฟล์ (Part 023–030) ด้วยการสอน **File Status Codes**
อย่างเป็นระบบ สรุปสิ่งที่เรียนรู้:

- โครงสร้างของ `FILE STATUS` (ฟิลด์ 2 ตัวอักษรเสมอ) และตารางหมวดหมู่ตามหลักแรก (`0x`–`4x`, `9x`)
- หมวด `0x`: ความสำเร็จที่มีเงื่อนไขพิเศษ (`00`, `02`, `05`) และพฤติกรรมจริงของ `02` ที่ทดสอบยืนยัน
  กับ GnuCOBOL
- หมวด `1x`: `10` (จบไฟล์) และ `46` (อ่านซ้ำหลังจบไฟล์) ซึ่งเป็นคนละสถานะกัน
- หมวด `2x`: `21` (ลำดับผิด), `22` (Key ซ้ำ), `23` (ไม่พบ) รวบรวมจาก Part 028–029 มาเปรียบเทียบ
- หมวด `3x`: `35` (ไฟล์ไม่มีอยู่จริง), `38` (ถูกล็อกไว้) และการค้นพบสำคัญว่า `FILE STATUS` ป้องกัน
  โปรแกรมจากการล่มได้จริง
- หมวด `4x`: ข้อผิดพลาดเชิงตรรกะจากการเรียกคำสั่งผิดโหมด (`41`, `42`, `47`, `48`, `49`)
- เทคนิค 88-level Condition Names เพื่อทำให้โค้ดตรวจสอบสถานะอ่านง่ายขึ้น (`FS-DUPLICATE-KEY`
  แทน `"22"`)
- เทคนิคการตรวจสอบแบบทั่วไปด้วย Reference Modification บนหลักแรกของ `FILE STATUS`
- Pattern การใช้งานจริง: `OPTIONAL` ผสาน `FILE STATUS` สำหรับไฟล์ที่อาจมาไม่ทัน
- โปรแกรมรวบยอดระบบปรับปรุงไฟล์หลักที่ทนทานต่อข้อผิดพลาด พร้อมไฟล์บันทึกข้อผิดพลาดแยกต่างหาก

ด้วย Part 023–030 คุณได้เรียนรู้การประมวลผลไฟล์ใน COBOL อย่างครบวงจร ตั้งแต่พื้นฐานที่สุดไปจนถึง
การจัดการข้อผิดพลาดอย่างเป็นระบบ ถึงเวลาก้าวไปสู่หัวข้อใหม่ที่สำคัญไม่แพ้กัน: **Part 031** จะแนะนำ
**Subprograms** และคำสั่ง **`CALL`** — เทคนิคการแบ่งโปรแกรมขนาดใหญ่ออกเป็นโปรแกรมย่อยที่เรียกใช้
ซึ่งกันและกันได้ วางรากฐานสำหรับการเขียนระบบ COBOL ขนาดใหญ่ที่ดูแลรักษาง่ายในระยะยาว

**[← กลับไป Part 029](part-029-relative-files.md)** | **[ไปยัง Part 031: Subprograms และ CALL Statement →](part-031-call-statement.md)**
