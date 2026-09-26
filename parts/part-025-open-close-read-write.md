# Part 025: OPEN, CLOSE, READ, WRITE, REWRITE, DELETE (ขั้นตอนที่ 241–250)

## คำนำของ Part นี้

Part 024 พาเราเจาะลึกฝั่ง **โครงสร้างข้อมูล** ของไฟล์ผ่าน `FD Entry` ไปแล้ว มาถึง Part นี้เราจะกลับไปที่
ฝั่ง **PROCEDURE DIVISION** เพื่อเจาะลึกคำสั่งควบคุมไฟล์ทั้งหมดอย่างครบถ้วนที่สุด ตั้งแต่ `OPEN` ทั้ง
4 โหมด, `CLOSE` ที่มีรายละเอียดมากกว่าที่คิด, `READ`/`WRITE` แบบขั้นสูง (`INTO`, `FROM`, `ADVANCING`),
ไปจนถึงคำสั่งใหม่ 2 ตัวที่ยังไม่เคยแตะมาก่อนคือ **`REWRITE`** (แก้ไข record ที่มีอยู่แล้ว) และ
**`DELETE`** (ลบ record) ซึ่งทั้งคู่ต้องใช้โหมด `OPEN I-O` ที่กล่าวถึงแค่ผ่าน ๆ ใน Part 023

สิ่งที่ทำให้ Part นี้พิเศษคือเราจะเจาะลึกพฤติกรรมจริงของ `REWRITE` เมื่อใช้กับไฟล์ `LINE SEQUENTIAL`
ซึ่งมี**รายละเอียดที่ไม่ค่อยมีใครพูดถึง**แต่สำคัญมากในทางปฏิบัติ (พิสูจน์ด้วยการทดลองจริง) รวมถึงข้อเท็จจริง
ที่ว่า `DELETE` **ใช้กับไฟล์ตามลำดับไม่ได้เลย** และเทคนิคทางเลือกที่ใช้กันจริงในอุตสาหกรรมแทน

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน โค้ดทุกตัวอย่าง
> รวมถึงตัวอย่างที่ตั้งใจแสดง compile error/runtime error จริง ผ่านการทดสอบด้วย GnuCOBOL
> (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง

---

## ขั้นตอนที่ 241: OPEN ครบทั้ง 4 โหมด — INPUT, OUTPUT, EXTEND, I-O

### ทบทวนและเติมเต็ม: ตาราง OPEN ทั้ง 4 โหมด

จาก Part 023 เรารู้จัก `OUTPUT`, `INPUT`, `EXTEND` มาแล้ว Part นี้เติมโหมดที่ 4 ให้ครบ:

| โหมด | ทำอะไรได้บ้าง | ผลต่อไฟล์เดิม |
|---|---|---|
| `OPEN OUTPUT` | `WRITE` เท่านั้น | ลบไฟล์เดิมทั้งหมด สร้างใหม่เปล่า |
| `OPEN INPUT` | `READ` เท่านั้น | ไม่แตะต้องไฟล์เลย (read-only) |
| `OPEN EXTEND` | `WRITE` เท่านั้น (ต่อท้าย) | คงของเดิมไว้ เพิ่มต่อท้าย |
| `OPEN I-O` | `READ`, `REWRITE`, `DELETE` | คงของเดิมไว้ แก้ไข record ที่มีอยู่ได้ |

`OPEN I-O` (I-O ย่อมาจาก Input-Output) คือโหมดที่ทำให้โปรแกรม**อ่านและแก้ไข record เดิม**ในไฟล์เดียวกัน
ได้ในการเปิดครั้งเดียว โดยไม่ต้องปิดแล้วเปิดใหม่สลับโหมดไปมา นี่คือกุญแจสำคัญที่ทำให้ `REWRITE` และ
`DELETE` ทำงานได้ (สอนละเอียดในขั้นตอนที่ 245–247)

### ตัวอย่างรวม: สาธิตทั้ง 4 โหมดในโปรแกรมเดียว

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP241-OPEN-MODES.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT LOG-FILE ASSIGN TO "LOG241.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  LOG-FILE.
       01  LOG-RECORD.
           05  LOG-TEXT                PIC X(25).
           05  LOG-SEQ                 PIC 9(3).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG                 PIC X VALUE "N".
           88  END-OF-FILE             VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> 1) OUTPUT - always starts a brand new, empty file.
           OPEN OUTPUT LOG-FILE.
           MOVE "LOGIN by SOMCHAI" TO LOG-TEXT.
           MOVE 1 TO LOG-SEQ.
           WRITE LOG-RECORD.
           CLOSE LOG-FILE.
           DISPLAY "1) OPEN OUTPUT  -> file created fresh.".

      *> 2) INPUT - read-only, cannot WRITE/REWRITE/DELETE here.
           OPEN INPUT LOG-FILE.
           READ LOG-FILE
               AT END
                   DISPLAY "   (empty)"
               NOT AT END
                   DISPLAY "2) OPEN INPUT   -> read: " LOG-TEXT
           END-READ.
           CLOSE LOG-FILE.

      *> 3) EXTEND - append after the existing content.
           OPEN EXTEND LOG-FILE.
           MOVE "LOGOUT by SOMCHAI" TO LOG-TEXT.
           MOVE 2 TO LOG-SEQ.
           WRITE LOG-RECORD.
           CLOSE LOG-FILE.
           DISPLAY "3) OPEN EXTEND  -> appended a 2nd line.".

      *> 4) I-O - read AND rewrite the same file in one pass.
      *> LOG-SEQ (a numeric field) always fills its width, so the
      *> line on disk is never shorter than the full record - this
      *> keeps REWRITE fully safe (the full story is in step 246).
           OPEN I-O LOG-FILE.
           READ LOG-FILE
               AT END
                   SET END-OF-FILE TO TRUE
               NOT AT END
                   MOVE "LOGIN by SOMCHAI (EDITED)" TO LOG-TEXT
                   REWRITE LOG-RECORD
           END-READ.
           CLOSE LOG-FILE.
           DISPLAY "4) OPEN I-O     -> rewrote the 1st line.".

           DISPLAY "---- Final file content ----".
           MOVE "N" TO WS-EOF-FLAG.
           OPEN INPUT LOG-FILE.
           PERFORM UNTIL END-OF-FILE
               READ LOG-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "  " LOG-TEXT " seq=" LOG-SEQ
               END-READ
           END-PERFORM.
           CLOSE LOG-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
1) OPEN OUTPUT  -> file created fresh.
2) OPEN INPUT   -> read: LOGIN by SOMCHAI
3) OPEN EXTEND  -> appended a 2nd line.
4) OPEN I-O     -> rewrote the 1st line.
---- Final file content ----
  LOGIN by SOMCHAI (EDITED) seq=001
  LOGOUT by SOMCHAI         seq=002
```

### อธิบายจุดสำคัญ

- โปรแกรมเดียวสามารถ `OPEN`/`CLOSE` ไฟล์เดียวกันสลับโหมดไปมาได้หลายรอบ ตราบใดที่ `CLOSE` ก่อน
  `OPEN` ใหม่ในโหมดอื่นเสมอ (ทบทวนจาก Part 023)
- `OPEN I-O` เป็นโหมดเดียวที่รองรับทั้ง `READ` **และ** `REWRITE`/`DELETE` ในการเปิดครั้งเดียวกัน —
  ต่างจาก `OPEN INPUT` ที่ทำได้แค่ `READ` อย่างเดียว
- สังเกตการออกแบบ record: `LOG-SEQ` (ตัวเลข) ถูกวางไว้เป็น field สุดท้ายโดยตั้งใจ เพราะข้อมูลตัวเลข
  จะเติมเต็มความกว้างเสมอ (ไม่มีช่องว่างท้ายให้ `LINE SEQUENTIAL` ตัดทิ้ง) นี่เป็นการออกแบบเชิงป้องกัน
  ปัญหาที่จะอธิบายเหตุผลแบบเต็มในขั้นตอนที่ 246

### ข้อควรระวัง

- ในโหมด `OPEN INPUT` การพยายามเรียก `WRITE`, `REWRITE`, หรือ `DELETE` จะทำให้เกิด runtime error
  ทันที เพราะไฟล์ถูกเปิดในโหมดอ่านอย่างเดียว
- ในโหมด `OPEN OUTPUT`/`EXTEND` การพยายามเรียก `READ`, `REWRITE`, หรือ `DELETE` ก็จะเกิด error
  เช่นกัน เพราะไฟล์ไม่ได้เปิดให้อ่าน — จำง่าย ๆ ว่า **โหมดที่เปิดกำหนดว่าใช้คำสั่งอะไรได้บ้างอย่างเข้มงวด**

### แบบฝึกหัดที่ 241.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมข้างต้นจึงต้องเรียก `CLOSE LOG-FILE` ทุกครั้งก่อนจะ `OPEN` ในโหมด
ถัดไป แม้จะเป็นไฟล์เดียวกันก็ตาม

**เฉลย**: เพราะ COBOL อนุญาตให้ไฟล์หนึ่ง ๆ ถูกเปิดอยู่ได้เพียง**สถานะเดียว**ในเวลาเดียวกันเสมอ
(เปิดอยู่ หรือปิดอยู่) การจะเปลี่ยนโหมดการเปิด (เช่น จาก `OUTPUT` ไปเป็น `INPUT`) ต้อง `CLOSE`
สถานะเดิมให้เสร็จสมบูรณ์ก่อนเสมอ มิฉะนั้น compiler จะรายงาน runtime error ทันทีว่าไฟล์ถูกเปิดซ้อน
(ทบทวนแนวคิด OPEN-PROCESS-CLOSE Pattern จาก Part 023)

---

## ขั้นตอนที่ 242: CLOSE Statement โดยละเอียด — ทำไมสำคัญกว่าที่คิด

### ทำไมลืม CLOSE ถึงอันตราย

ใน Part 023 เราเตือนไว้สั้น ๆ ว่าลืม `CLOSE` อาจทำให้ข้อมูลไม่ถูกเขียนลงดิสก์ครบถ้วน สาเหตุคือ COBOL
(เหมือนภาษาโปรแกรมส่วนใหญ่) ไม่ได้เขียนข้อมูลลงดิสก์จริงทันทีทุกครั้งที่ `WRITE` แต่จะเก็บพักไว้ใน
**บัฟเฟอร์ (Buffer)** ในหน่วยความจำก่อนเพื่อประสิทธิภาพ แล้วค่อย "flush" (ระบายข้อมูลจากบัฟเฟอร์ลง
ดิสก์จริง) เมื่อ `CLOSE` เท่านั้น

### ทดลอง: ลืม CLOSE จะเกิดอะไรขึ้นจริง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP242A-FORGOT-CLOSE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT LOG-FILE ASSIGN TO "LOG242A.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  LOG-FILE.
       01  LOG-RECORD                  PIC X(20).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT LOG-FILE.
           MOVE "ENTRY WITHOUT CLOSE" TO LOG-RECORD.
           WRITE LOG-RECORD.
      *> Notice: no CLOSE LOG-FILE here before STOP RUN.
           STOP RUN.
```

เมื่อรันโปรแกรมนี้จริง GnuCOBOL แสดงคำเตือนออกทาง stderr:

```
libcob: warning: implicit CLOSE of LOG-FILE ('LOG242A.DAT')
```

และไฟล์ `LOG242A.DAT` ก็ยังถูกสร้างขึ้นถูกต้องเพราะ **GnuCOBOL แอบ `CLOSE` ไฟล์ที่ยังเปิดค้างอยู่ให้
อัตโนมัติเมื่อโปรแกรมถึง `STOP RUN`** (implicit close) นี่คือความปลอดภัยเชิง runtime ของ GnuCOBOL
เอง **แต่ไม่ควรพึ่งพาพฤติกรรมนี้เด็ดขาด** เพราะ:

1. Compiler/runtime แต่ละตัวมีนโยบายเรื่องนี้ต่างกัน โค้ดที่พึ่ง implicit close อาจไม่พกพาข้ามระบบได้
2. ถ้าโปรแกรม**ล่มกะทันหัน** (เช่น error ร้ายแรงระหว่างทาง ไม่ถึง `STOP RUN` ปกติ) buffer อาจไม่ถูก
   flush เลย ทำให้ข้อมูลบางส่วนหายไปจริง
3. เป็นนิสัยการเขียนโค้ดที่ไม่ดี — คนอ่านโค้ดจะสงสัยว่าลืมจริงหรือไงตั้งใจ

### CLOSE WITH LOCK — ปิดแบบล็อกถาวรในรันนั้น

COBOL มีตัวเลือก `CLOSE ... WITH LOCK` ที่บอกว่า **"ปิดไฟล์นี้แล้วห้ามเปิดมันอีกเด็ดขาดตลอดการรัน
โปรแกรมครั้งนี้"** มีประโยชน์เมื่อต้องการป้องกันไม่ให้ paragraph อื่นเผลอเปิดไฟล์ซ้ำโดยไม่ตั้งใจ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP242B-CLOSE-WITH-LOCK.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT LOG-FILE ASSIGN TO "LOG242B.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  LOG-FILE.
       01  LOG-RECORD                  PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS              PIC XX.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT LOG-FILE.
           MOVE "FINAL ENTRY" TO LOG-RECORD.
           WRITE LOG-RECORD.
      *> WITH LOCK means this file can NEVER be opened again by
      *> this same program run, even in INPUT mode.
           CLOSE LOG-FILE WITH LOCK.
           DISPLAY "Closed with lock.".

           OPEN INPUT LOG-FILE.
           DISPLAY "File status after re-OPEN attempt: "
               WS-FILE-STATUS.
           STOP RUN.
```

**ผลลัพธ์:**

```
Closed with lock.
File status after re-OPEN attempt: 38
```

### อธิบาย FILE STATUS 38

`FILE STATUS IS WS-FILE-STATUS` (เพิ่มเข้าไปใน `SELECT`) เป็นเทคนิคสำคัญที่จะสอนละเอียดใน Part 030
มันบอกให้ COBOL เก็บ**รหัสผลลัพธ์**ของทุกปฏิบัติการไฟล์ (`OPEN`, `READ`, `WRITE`, ...) ไว้ในตัวแปร
2 หลักนี้แทนที่จะปล่อยให้โปรแกรมล่มเมื่อเกิดปัญหา ค่า **`38`** ตามมาตรฐาน COBOL แปลว่า **"พยายามเปิด
ไฟล์ที่ถูกปิดด้วย `WITH LOCK` ไปแล้วก่อนหน้านี้ในรันเดียวกัน"** ซึ่งตรงกับสถานการณ์ในตัวอย่างนี้เป๊ะ ๆ
สังเกตว่าเพราะเรามี `FILE STATUS` คอยตรวจสอบ โปรแกรมจึงไม่ล่ม แต่แสดงรหัส error ให้เราจัดการต่อได้เอง

### ข้อควรระวัง

- อย่าใช้ `WITH LOCK` พร่ำเพรื่อ — ใช้เฉพาะเมื่อแน่ใจจริง ๆ ว่าไฟล์นั้นจะไม่ถูกเปิดอีกในรันเดียวกัน
  เพราะการเปิดซ้ำโดยไม่ได้ตั้งใจ (เช่น `PERFORM` ผิด paragraph) จะกลายเป็น error ที่หาสาเหตุยากขึ้น
  ถ้าไม่ได้ตรวจ `FILE STATUS` ไว้
- การมี `FILE STATUS` clause ใน `SELECT` ไม่ได้บังคับให้ต้องตรวจสอบทุกครั้ง แต่เป็น**แนวปฏิบัติที่ดี
  มาก**สำหรับโปรแกรมระดับองค์กรจริง เพราะช่วยให้จัดการข้อผิดพลาดได้อย่างนุ่มนวลแทนที่จะปล่อยให้ล่ม
  (เนื้อหาเต็มเรื่องนี้อยู่ใน Part 030)

### แบบฝึกหัดที่ 242.1

**โจทย์**: จงอธิบายว่าทำไมการพึ่งพา "implicit close" ของ GnuCOBOL ไม่ใช่แนวทางที่ปลอดภัยสำหรับ
โปรแกรมที่ทำงานกับข้อมูลสำคัญ เช่น ระบบธนาคาร

**เฉลย**: เพราะ implicit close จะเกิดขึ้น**เฉพาะเมื่อโปรแกรมจบการทำงานตามปกติผ่าน `STOP RUN`
เท่านั้น** หากโปรแกรมเกิด crash กะทันหันจากสาเหตุอื่น (เช่น หน่วยความจำเต็ม, ระบบปฏิบัติการ kill
process, ไฟดับ) buffer ที่ยังไม่ถูก flush จะหายไปพร้อมกับข้อมูลที่ควรถูกบันทึก สำหรับระบบธนาคารที่
ทุกธุรกรรมมีมูลค่าทางการเงินจริง การสูญเสียข้อมูลแม้เพียง record เดียวอาจสร้างความเสียหายร้ายแรง
จึงต้อง `CLOSE` อย่างชัดเจนในโค้ดเสมอ ไม่ปล่อยให้ compiler ช่วยเก็บกวาดให้

---

## ขั้นตอนที่ 243: READ Statement ขั้นสูง — READ...INTO

### ปัญหา: READ แล้วต้อง MOVE ทันทีเสมอ

จาก Part 024 ขั้นตอนที่ 239 เราเรียนรู้แนวปฏิบัติที่ดีคือคัดลอก FD record ไปยัง WORKING-STORAGE
record ทันทีหลัง `READ` ด้วยการเขียน `READ` แล้วตามด้วย `MOVE` อีกบรรทัดหนึ่งเสมอ COBOL มีทางลัดที่
รวมสองขั้นตอนนี้เข้าด้วยกันในคำสั่งเดียว นั่นคือ **`READ ... INTO`**

### ตัวอย่าง: READ...INTO ในทางปฏิบัติ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP243-READ-INTO.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT STUDENT-FILE ASSIGN TO "STUDENT243.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  STUDENT-FILE.
       01  FD-STUDENT-RECORD           PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-STUDENT-RECORD           PIC X(10).
       01  WS-EOF-FLAG                 PIC X VALUE "N".
           88  END-OF-FILE             VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT STUDENT-FILE.
           MOVE "STUDENT-1" TO FD-STUDENT-RECORD.
           WRITE FD-STUDENT-RECORD.
           MOVE "STUDENT-2" TO FD-STUDENT-RECORD.
           WRITE FD-STUDENT-RECORD.
           CLOSE STUDENT-FILE.

           OPEN INPUT STUDENT-FILE.
           PERFORM UNTIL END-OF-FILE
      *> READ...INTO does the READ and the MOVE to a
      *> WORKING-STORAGE record in a single statement.
               READ STUDENT-FILE INTO WS-STUDENT-RECORD
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY "Got: [" WS-STUDENT-RECORD "]"
               END-READ
           END-PERFORM.
           CLOSE STUDENT-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Got: [STUDENT-1 ]
Got: [STUDENT-2 ]
```

### อธิบายจุดสำคัญ

- `READ STUDENT-FILE INTO WS-STUDENT-RECORD` เทียบเท่ากับการเขียน `READ STUDENT-FILE` แล้วตามด้วย
  `MOVE FD-STUDENT-RECORD TO WS-STUDENT-RECORD` ในบรรทัดถัดไป **แต่รวมเป็นคำสั่งเดียว**
- ข้อดีสำคัญ: การ `MOVE` จะเกิดขึ้น**เฉพาะกรณีที่อ่านสำเร็จ (NOT AT END) เท่านั้น** — ถ้าอ่านถึงจุด
  สิ้นสุดไฟล์ (`AT END`) COBOL จะ**ไม่** `MOVE` อะไรเข้า `WS-STUDENT-RECORD` เลย (ค่าเดิมที่มีอยู่
  ก่อนหน้าจะยังคงอยู่) พฤติกรรมนี้ตรงกับสามัญสำนึก: ไม่มี record ให้อ่าน ก็ไม่ควรมีอะไรถูกคัดลอก
- ยังคงต้องมี `AT END`/`NOT AT END` เหมือน `READ` ปกติทุกประการ `INTO` เป็นแค่ตัวเสริม ไม่ได้แทนที่
  ส่วนนี้

### ข้อควรระวัง

- `WS-STUDENT-RECORD` และ `FD-STUDENT-RECORD` ยังต้องมีโครงสร้าง/ความยาวที่เข้ากันได้เหมือนการ
  `MOVE` ธรรมดา มิฉะนั้นข้อมูลจะถูกตัดหรือเติมช่องว่างผิดตำแหน่ง (กฎ MOVE เดิมจาก Part 008)
- `READ ... INTO` ไม่ได้ทำให้เขียนโค้ด**สั้นลงมาก** แต่ทำให้โค้ด**อ่านง่ายขึ้นและลดโอกาสลืม `MOVE`**
  ไปหนึ่งจุด ถือเป็นทางเลือกที่ดีเมื่อรูปแบบโปรแกรมเป็นแบบ "อ่านแล้วคัดลอกทันที" ตามที่แนะนำใน
  Part 024 ขั้นตอนที่ 239

### แบบฝึกหัดที่ 243.1

**โจทย์**: จงอธิบายว่าทำไมเมื่อ `READ...INTO` เจอ `AT END` แล้ว `WS-STUDENT-RECORD` จะยังคงมีค่า
จาก record **ก่อนหน้า**ที่อ่านสำเร็จครั้งล่าสุดค้างอยู่ ไม่ใช่ค่าว่างเปล่า

**เฉลย**: เพราะ COBOL ออกแบบ `READ...INTO` ให้การ `MOVE` เกิดขึ้นเฉพาะกรณี `NOT AT END` เท่านั้น
เมื่อถึง `AT END` (ไม่มี record เหลือให้อ่านแล้ว) จะไม่มีการ `MOVE` ใด ๆ เกิดขึ้นเลย ตัวแปร
`WS-STUDENT-RECORD` จึงยังคง "ค้าง" ค่าจากการ `MOVE` ครั้งล่าสุดที่เคยเกิดขึ้นจริง (จาก record ก่อน
หน้า) นี่คือเหตุผลเดียวกับที่ Part 023 เตือนไว้ว่าอย่าใช้ค่าใน record area ในสาขา `AT END`

---

## ขั้นตอนที่ 244: WRITE Statement ขั้นสูง — WRITE...FROM และ ADVANCING

### WRITE...FROM — ทางลัดสำหรับเขียนจาก WORKING-STORAGE โดยตรง

เช่นเดียวกับ `READ...INTO`, คำสั่ง **`WRITE...FROM`** รวมการ `MOVE` และ `WRITE` เข้าด้วยกันสำหรับ
ทิศทางตรงข้าม (จาก WORKING-STORAGE ไปยัง FD record แล้วเขียนทันที)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP244-WRITE-FROM.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MEMO-FILE ASSIGN TO "MEMO244.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  MEMO-FILE.
       01  FD-MEMO-LINE                PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-MEMO-LINE                PIC X(20) VALUE "HELLO FROM WS".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT MEMO-FILE.
      *> WRITE ... FROM moves WS-MEMO-LINE into FD-MEMO-LINE and
      *> writes it, in a single statement.
           WRITE FD-MEMO-LINE FROM WS-MEMO-LINE.

      *> AFTER ADVANCING inserts blank lines before the record is
      *> written - useful for print-style reports (full coverage
      *> is in Part 038, Report Writer). Keep it as the LAST write
      *> in a sequence; mixing it with a later plain WRITE can
      *> give inconsistent spacing on LINE SEQUENTIAL files.
           MOVE "SECTION TITLE" TO WS-MEMO-LINE.
           WRITE FD-MEMO-LINE FROM WS-MEMO-LINE
               AFTER ADVANCING 2 LINES.
           CLOSE MEMO-FILE.
           DISPLAY "MEMO244.DAT written.".
           STOP RUN.
```

**ผลลัพธ์:**

```
MEMO244.DAT written.
```

**เนื้อหาไฟล์ `MEMO244.DAT`** (ตรวจสอบด้วย `cat -A` เพื่อเห็นบรรทัดว่าง):

```
HELLO FROM WS

SECTION TITLE
```

### อธิบายจุดสำคัญ

- `WRITE FD-MEMO-LINE FROM WS-MEMO-LINE` เทียบเท่ากับ `MOVE WS-MEMO-LINE TO FD-MEMO-LINE` ตามด้วย
  `WRITE FD-MEMO-LINE` แต่รวมเป็นบรรทัดเดียว มีประโยชน์มากเมื่อข้อมูลที่จะเขียนถูกเตรียมไว้ใน
  WORKING-STORAGE อยู่แล้ว (เช่น หลังคำนวณเสร็จ)
- `AFTER ADVANCING n LINES` สั่งให้เว้นบรรทัดว่าง n-1 บรรทัดก่อนเขียน record นั้น (ในตัวอย่างนี้
  `ADVANCING 2 LINES` ทำให้เกิดบรรทัดว่าง 1 บรรทัดคั่นระหว่าง `HELLO FROM WS` กับ `SECTION TITLE`)
  clause นี้เดิมออกแบบมาสำหรับควบคุมกระดาษเครื่องพิมพ์ในยุคก่อน จึงเห็นชื่อ "LINES" ที่สื่อถึงการ
  เลื่อนกระดาษ
- clause คู่กัน `BEFORE ADVANCING` ก็มีอยู่เช่นกัน แต่พฤติกรรมที่แน่นอนของทั้งสอง clause เมื่อใช้กับ
  ไฟล์ `LINE SEQUENTIAL` (ซึ่งไม่ใช่ไฟล์พิมพ์จริง) อาจให้ผลไม่ตรงตามสัญชาตญาณเสมอไป การควบคุมการ
  จัดหน้ารายงานอย่างแม่นยำและน่าเชื่อถือเต็มรูปแบบจะสอนด้วยเทคนิคที่เหมาะสมกว่าใน **Part 038
  (Report Writer)**

### ข้อควรระวัง

- ควรใช้ `ADVANCING` เป็นคำสั่ง `WRITE` **สุดท้าย**ของบล็อกโค้ดนั้น หรือใช้ `ADVANCING` กับทุก
  `WRITE` ในบล็อกอย่างสม่ำเสมอ การผสม `WRITE` ธรรมดากับ `WRITE ... ADVANCING` ปะปนกันในไฟล์
  `LINE SEQUENTIAL` เดียวกันอาจให้ผลลัพธ์ที่ไม่คงเส้นคงวา
- `WRITE...FROM` ไม่ได้ยกเว้นกฎเรื่อง `FILLER`/ข้อมูลยังไม่ได้กำหนดค่าจาก Part 024 — ถ้า
  `FD-MEMO-LINE` เป็น group ที่มี `FILLER` ซ้อนอยู่ ก็ยังต้องระวังปัญหาเดิมเช่นกัน

### แบบฝึกหัดที่ 244.1

**โจทย์**: จงอธิบายว่า `WRITE FD-MEMO-LINE FROM WS-MEMO-LINE` กับการเขียนแยก 2 บรรทัด
(`MOVE WS-MEMO-LINE TO FD-MEMO-LINE.` แล้วตามด้วย `WRITE FD-MEMO-LINE.`) ให้ผลลัพธ์ต่างกันหรือไม่

**เฉลย**: ไม่ต่างกันเลยในแง่ผลลัพธ์สุดท้าย ทั้งสองแบบทำสิ่งเดียวกันทุกประการ (คัดลอกค่าจาก
WORKING-STORAGE เข้า FD record แล้วเขียนลงไฟล์) ต่างกันแค่**จำนวนบรรทัดโค้ด**และความสะดวกในการอ่าน
`WRITE...FROM` เป็นเพียงวากยสัมพันธ์ทางลัด (syntactic sugar) ที่ COBOL มอบให้เพื่อความกระชับเท่านั้น

---

## ขั้นตอนที่ 245: REWRITE Statement — แก้ไข Record ที่มีอยู่แล้ว

### แนวคิดพื้นฐานของ REWRITE

`REWRITE` คือคำสั่งที่ใช้ **แทนที่เนื้อหาของ record ที่เพิ่ง `READ` มาล่าสุด ด้วยเนื้อหาใหม่ ณ ตำแหน่ง
เดิมในไฟล์** แตกต่างจาก `WRITE` ที่เพิ่ม record ใหม่เข้าไปในไฟล์ `REWRITE` ไม่เพิ่มจำนวน record เลย
เพียงแค่ "เขียนทับ" record ที่มีอยู่แล้วเท่านั้น ข้อกำหนดสำคัญที่สุดคือ **ไฟล์ต้องถูกเปิดด้วย
`OPEN I-O` เท่านั้น** (ไม่ใช่ `OUTPUT`, `INPUT`, หรือ `EXTEND`)

### ตัวอย่าง: REWRITE พื้นฐานที่ทำงานถูกต้อง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP245-REWRITE-BASICS.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ITEM-FILE ASSIGN TO "ITEM245.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  ITEM-FILE.
       01  ITEM-RECORD                 PIC X(10).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT ITEM-FILE.
           MOVE "AAAAAAAAAA" TO ITEM-RECORD.
           WRITE ITEM-RECORD.
           MOVE "BBBBBBBBBB" TO ITEM-RECORD.
           WRITE ITEM-RECORD.
           CLOSE ITEM-FILE.

      *> REWRITE requires OPEN I-O: the file must be opened for
      *> BOTH reading and writing at the same time.
           OPEN I-O ITEM-FILE.
           READ ITEM-FILE.
           DISPLAY "Read before REWRITE : " ITEM-RECORD.
           MOVE "ZZZZZZZZZZ" TO ITEM-RECORD.
           REWRITE ITEM-RECORD.
           DISPLAY "Rewrote 1st record.".
           CLOSE ITEM-FILE.

           OPEN INPUT ITEM-FILE.
           READ ITEM-FILE.
           DISPLAY "Record 1 is now    : " ITEM-RECORD.
           READ ITEM-FILE.
           DISPLAY "Record 2 unchanged : " ITEM-RECORD.
           CLOSE ITEM-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Read before REWRITE : AAAAAAAAAA
Rewrote 1st record.
Record 1 is now    : ZZZZZZZZZZ
Record 2 unchanged : BBBBBBBBBB
```

### อธิบายลำดับการทำงานของ REWRITE

`REWRITE` **ต้องตามหลัง `READ` ของ record เดียวกันเสมอ** ในทางปฏิบัติมันคือการบอก COBOL ว่า
"เอา record ที่เพิ่งอ่านมานี้ ไปแทนที่ด้วยเนื้อหาปัจจุบันใน record area" ลำดับที่ถูกต้องคือ:

1. `OPEN I-O` เปิดไฟล์ในโหมดที่รองรับทั้งอ่านและเขียนทับ
2. `READ` อ่าน record ที่ต้องการแก้ไขเข้ามาในหน่วยความจำก่อน
3. `MOVE` ค่าใหม่ทับลงใน field ที่ต้องการเปลี่ยนแปลง (เหมือนที่ทำกับ WORKING-STORAGE ทั่วไป)
4. `REWRITE` เขียนเนื้อหาปัจจุบันของ record area นั้นกลับไปแทนที่ตำแหน่งเดิมในไฟล์

### ข้อควรระวัง

- **ห้าม `REWRITE` โดยไม่มี `READ` มาก่อน** ในรอบนั้น เพราะ COBOL ไม่รู้ว่าจะ "แทนที่ตำแหน่งไหน"
  ถ้าไม่เพิ่งอ่าน record นั้นเข้ามาก่อน
- โปรแกรมตัวอย่างนี้ปลอดภัยเพราะ `ITEM-RECORD` เป็น `PIC X(10)` เดี่ยว ๆ ที่มีค่าเต็มความกว้างเสมอ
  (`AAAAAAAAAA` และ `ZZZZZZZZZZ` ไม่มีช่องว่างท้ายให้ตัดทิ้ง) — ขั้นตอนถัดไปจะเผยให้เห็นว่าเกิด
  อะไรขึ้นเมื่อเงื่อนไขนี้ไม่เป็นจริง

### แบบฝึกหัดที่ 245.1

**โจทย์**: จงอธิบายว่าทำไม `REWRITE` จึงไม่สามารถใช้เพิ่มจำนวน record ในไฟล์ได้ ต่างจาก `WRITE`

**เฉลย**: เพราะ `REWRITE` ถูกออกแบบมาให้ทำงานกับ**ตำแหน่งของ record ที่มีอยู่แล้วเท่านั้น** (ตำแหน่ง
ที่เพิ่งถูก `READ` เข้ามา) มันไม่ได้เพิ่มพื้นที่ใหม่ในไฟล์ เพียงแค่เขียนทับพื้นที่เดิมที่มีอยู่แล้ว
ในขณะที่ `WRITE` จะเพิ่ม record ใหม่เข้าไปในไฟล์เสมอ (ต่อท้ายในกรณี `LINE SEQUENTIAL`) จึงทำให้
จำนวน record ในไฟล์เพิ่มขึ้นได้ ความแตกต่างนี้สอดคล้องกับความหมายของชื่อคำสั่ง: RE-WRITE คือ
"เขียนใหม่" (ทับของเดิม) ไม่ใช่ "เขียนเพิ่ม"

---

## ขั้นตอนที่ 246: ข้อผิดพลาดสำคัญของ REWRITE กับ LINE SEQUENTIAL — พิสูจน์ด้วยการทดลองจริง

### การทดลอง: REWRITE ด้วยข้อมูลที่ยาวกว่าเดิม

นี่คือรายละเอียดที่สำคัญที่สุดของ Part นี้ ซึ่งค้นพบจากการทดสอบจริงและ**ไม่ค่อยมีเอกสารใดพูดถึง**
ลองพิจารณาโปรแกรมต่อไปนี้ ที่ดูเผิน ๆ เหมือนจะทำงานถูกต้องตามที่เรียนในขั้นตอนที่ 245:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP246-REWRITE-PITFALL.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT NOTE-FILE ASSIGN TO "NOTE246.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  NOTE-FILE.
       01  NOTE-RECORD                 PIC X(30).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> The original line uses only 8 of the 30 available bytes.
      *> LINE SEQUENTIAL trims the other 22 trailing spaces when
      *> it writes the line to disk, so only 8 bytes actually
      *> exist on disk for this record.
           OPEN OUTPUT NOTE-FILE.
           MOVE "SHORT-1" TO NOTE-RECORD.
           WRITE NOTE-RECORD.
           CLOSE NOTE-FILE.

           OPEN I-O NOTE-FILE.
           READ NOTE-FILE.
      *> We now try to REWRITE with a much LONGER value than the
      *> original 8-byte line that is actually on disk.
           MOVE "THIS NEW VALUE IS MUCH LONGER" TO NOTE-RECORD.
           REWRITE NOTE-RECORD.
           CLOSE NOTE-FILE.

           OPEN INPUT NOTE-FILE.
           READ NOTE-FILE.
           DISPLAY "Read back after REWRITE: [" NOTE-RECORD "]".
           CLOSE NOTE-FILE.
           STOP RUN.
```

**ผลลัพธ์จริงที่ได้ (ทดสอบยืนยันแล้วด้วย GnuCOBOL):**

```
Read back after REWRITE: [THIS NE                       ]
```

### วิเคราะห์: เกิดอะไรขึ้นกันแน่

สังเกตให้ดี: ข้อความที่เรา `MOVE` เข้าไปคือ `"THIS NEW VALUE IS MUCH LONGER"` (29 ตัวอักษร) แต่สิ่งที่
ถูกเขียนกลับลงไฟล์จริงกลับเป็นแค่ **`"THIS NE"`** (7 ตัวอักษรแรกเท่านั้น) ถูกตัดทิ้งไปดื้อ ๆ โดยไม่มี
error ใด ๆ เตือนเลย! เมื่อตรวจสอบไบต์ดิบของไฟล์ด้วย Python (`open(file, 'rb').read()`) จะพบว่า:

```
b'THIS NE\n'
```

**คำอธิบาย**: `"SHORT-1"` (record เดิมที่เขียนไว้ตอนแรก) มีความยาวแค่ **7 ไบต์** (S-H-O-R-T-hyphen-1)
เพราะ `LINE SEQUENTIAL` ตัดช่องว่างท้ายทิ้งตอนเขียนลงดิสก์ (ทบทวนจาก Part 023 ขั้นตอน 222) ดังนั้น
บนดิสก์จริงมีแค่ 7 ไบต์สำหรับบรรทัดนี้ ไม่ใช่ 30 ไบต์ตามที่ `PIC X(30)` ประกาศไว้

เมื่อเรียก `REWRITE` GnuCOBOL จะ **เขียนทับ ณ ตำแหน่งเดิมด้วยจำนวนไบต์เท่ากับความยาวของบรรทัดเดิมที่
มีอยู่จริงบนดิสก์เท่านั้น (7 ไบต์) ไม่ใช่ความยาวเต็มของ `PIC` ที่ประกาศไว้ (30 ไบต์)** ค่าใหม่ที่ยาว
กว่าจึงถูกตัดท้ายทิ้งไปโดยอัตโนมัติ และที่สำคัญคือ **`FILE STATUS` ยังคงรายงานว่าสำเร็จ (`00`)**
ไม่มีการแจ้งเตือนใด ๆ ว่าข้อมูลถูกตัดทิ้ง — นี่คือกับดักที่อันตรายที่สุดของ `REWRITE` บนไฟล์
`LINE SEQUENTIAL`

### กฎที่ต้องจำ

> **REWRITE บนไฟล์ `LINE SEQUENTIAL` จะปลอดภัย 100% ก็ต่อเมื่อบรรทัดเดิมบนดิสก์มีความยาวเท่ากับ
> ความยาวเต็มของ `PIC` ที่ประกาศไว้ใน `FD` เท่านั้น** (กล่าวคือ ไม่มีช่องว่างท้ายที่ถูกตัดทิ้งตอน
> เขียนครั้งแรก) ซึ่งจะเป็นจริงเสมอเมื่อ field สุดท้ายของ record เป็น**ข้อมูลตัวเลข** (numeric)
> หรือเป็นอักขระคงที่ที่ไม่ใช่ช่องว่าง (เช่น flag/marker ตัวเดียว)

### ทางแก้: บังคับให้ Record เต็มความกว้างเสมอด้วย Field ปลายทางที่ไม่ใช่ช่องว่าง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP246B-REWRITE-FIX.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT NOTE-FILE ASSIGN TO "NOTE246B.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  NOTE-FILE.
       01  NOTE-RECORD.
           05  NOTE-TEXT               PIC X(29).
           05  NOTE-MARKER             PIC X(1).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> NOTE-MARKER is a fixed, non-space trailing byte, so the
      *> full 30-byte width is ALWAYS written to disk - the line
      *> is never shortened by trailing-space trimming. Remember
      *> from step 236: a FILE SECTION VALUE clause is NOT applied
      *> automatically, so NOTE-MARKER must be MOVEd explicitly.
           OPEN OUTPUT NOTE-FILE.
           MOVE "SHORT-1" TO NOTE-TEXT.
           MOVE "#" TO NOTE-MARKER.
           WRITE NOTE-RECORD.
           CLOSE NOTE-FILE.

           OPEN I-O NOTE-FILE.
           READ NOTE-FILE.
           MOVE "THIS NEW VALUE IS MUCH LONGER" TO NOTE-TEXT.
           MOVE "#" TO NOTE-MARKER.
           REWRITE NOTE-RECORD.
           CLOSE NOTE-FILE.

           OPEN INPUT NOTE-FILE.
           READ NOTE-FILE.
           DISPLAY "Read back after REWRITE: [" NOTE-TEXT "]".
           CLOSE NOTE-FILE.
           STOP RUN.
```

**ผลลัพธ์:**

```
Read back after REWRITE: [THIS NEW VALUE IS MUCH LONGER]
```

คราวนี้ทำงานถูกต้องสมบูรณ์! เพราะ `NOTE-MARKER` (ค่า `"#"`) เป็นไบต์สุดท้ายที่**ไม่ใช่ช่องว่าง**เสมอ
ทำให้บรรทัดเดิมบนดิสก์มีความยาวเต็ม 30 ไบต์จริงตั้งแต่การ `WRITE` ครั้งแรก (ไม่มีอะไรถูกตัดทิ้ง)
`REWRITE` ครั้งต่อมาจึงมีพื้นที่เต็ม 30 ไบต์ให้เขียนทับได้อย่างปลอดภัย

### ข้อควรระวัง (สรุปกฎทองของ REWRITE)

- ก่อนใช้ `REWRITE` กับไฟล์ `LINE SEQUENTIAL` ให้ตรวจสอบเสมอว่า record ของคุณมี field สุดท้ายที่
  รับประกันว่าไม่ใช่ช่องว่าง (ตัวเลข หรือ flag คงที่) มิฉะนั้นอาจเจอการตัดข้อมูลแบบเงียบ ๆ โดยไม่มี
  error ใด ๆ เตือนเลย (`FILE STATUS` ยังรายงาน `00` ปกติ)
- หากไม่สามารถควบคุมโครงสร้าง record ได้ตามนี้ (เช่น record ลงท้ายด้วยชื่อคนที่ความยาวไม่แน่นอน)
  ทางเลือกที่ปลอดภัยกว่าคือใช้เทคนิค **"Copy-Skip"** ที่จะสอนในขั้นตอนที่ 248 (สร้างไฟล์ใหม่ทั้งไฟล์
  แทนการ `REWRITE` เฉพาะจุด) หรือรอเรียนรู้ **Indexed File** ใน Part 028 ที่ใช้ record ความยาวคงที่
  แบบไบนารีจริง ๆ (ไม่มีปัญหาการตัดช่องว่างแบบนี้เลย)
- ปัญหานี้เกิดเฉพาะกับ `LINE SEQUENTIAL` เท่านั้น เพราะเป็นไฟล์แบบข้อความที่ตัดช่องว่างท้ายอัตโนมัติ
  หากใช้ `ORGANIZATION IS SEQUENTIAL` (แบบไบนารี ไม่ตัดช่องว่าง) หรือ `INDEXED`/`RELATIVE`
  ปัญหานี้จะไม่เกิดขึ้นเลยเพราะทุก record มีความยาวเต็มคงที่เสมอ

### แบบฝึกหัดที่ 246.1

**โจทย์**: จงอธิบายว่าทำไมตัวอย่างใน Part 024 ขั้นตอนที่ 240 (Product Master File ที่ใช้
`PM-STATUS PIC X(1)` เป็น field สุดท้าย) จึงไม่มีความเสี่ยงจากปัญหานี้ แม้จะยังไม่เคยใช้ `REWRITE`
ในตัวอย่างนั้นก็ตาม

**เฉลย**: เพราะ `PM-STATUS` (ซึ่งเก็บค่า `"A"` หรือ `"D"` เสมอ) เป็น field สุดท้ายของ `PM-RECORD`
และมันไม่เคยเป็นช่องว่างเลย (ต้องมีค่า A หรือ D เสมอตามที่ออกแบบ) ดังนั้นทุกบรรทัดที่เขียนลงไฟล์
`PRODMAST240.DAT` จะมีความยาวเต็ม 46 ไบต์ (ตามที่คำนวณได้ในแบบฝึกหัดของ Part 024) เสมอ ไม่มีการ
ตัดช่องว่างท้ายเกิดขึ้นเลย หากในอนาคตมีการเพิ่ม `REWRITE` เข้าไปในโปรแกรมนั้น (เช่น แก้ไขราคาสินค้า)
ก็จะทำงานได้อย่างปลอดภัยตามกฎทองที่เรียนในขั้นตอนนี้

---

## ขั้นตอนที่ 247: DELETE Statement — ทำไมใช้กับไฟล์ตามลำดับไม่ได้

### ทดลอง: พยายาม DELETE บนไฟล์ LINE SEQUENTIAL

`DELETE` คือคำสั่งที่ใช้ลบ record ออกจากไฟล์ ฟังดูเป็นธรรมชาติที่จะลองใช้กับไฟล์ที่เรารู้จักมาตลอด
แต่ลองดูสิ่งที่เกิดขึ้นจริงเมื่อคอมไพล์โปรแกรมต่อไปนี้:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP247-DELETE-ATTEMPT.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ITEM-FILE ASSIGN TO "ITEM247.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  ITEM-FILE.
       01  ITEM-RECORD                 PIC X(10).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT ITEM-FILE.
           MOVE "AAAAAAAAAA" TO ITEM-RECORD.
           WRITE ITEM-RECORD.
           MOVE "BBBBBBBBBB" TO ITEM-RECORD.
           WRITE ITEM-RECORD.
           CLOSE ITEM-FILE.

           OPEN I-O ITEM-FILE.
           READ ITEM-FILE.
      *> This DELETE will NOT even compile - see explanation below.
           DELETE ITEM-FILE.
           CLOSE ITEM-FILE.
           STOP RUN.
```

เมื่อพยายามคอมไพล์ด้วย `cobc -x -o step247 step247.cob` จะได้ **compile error ทันที** (ไม่ใช่แค่
runtime error) ดังนี้:

```
step247.cob: in paragraph 'MAIN-PARA':
step247.cob:28: error: DELETE not allowed on LINE SEQUENTIAL files
```

### เหตุผลเชิงกายภาพที่แท้จริง

นี่ไม่ใช่ข้อจำกัดเฉพาะของ GnuCOBOL แต่เป็น**ข้อจำกัดทางกายภาพที่มีมาตั้งแต่ต้นของไฟล์ตามลำดับ**:
ไฟล์ `LINE SEQUENTIAL` (และ `SEQUENTIAL` แบบไบนารีธรรมดา) เก็บข้อมูลเรียงต่อกันเป็นสาย
(byte stream) บนดิสก์อย่างต่อเนื่อง **ไม่มีแนวคิดเรื่อง "ตำแหน่งว่างระหว่างกลาง" ที่จะทำให้ลบ record
กลางไฟล์ได้โดยไม่กระทบ record อื่น** ลองนึกภาพว่าถ้าลบ record ที่ 2 จาก 3 record ออกไปตรง ๆ
record ที่ 3 (และทุก record หลังจากนั้น) จะต้อง**ขยับตำแหน่งทางกายภาพทั้งหมด**บนดิสก์ ซึ่งเป็นงาน
ที่มีต้นทุนสูงมากสำหรับไฟล์ขนาดใหญ่ (ต้องอ่าน-เขียนใหม่ทั้งไฟล์ทุกครั้งที่ลบ) จึง**ไม่มี compiler
COBOL ตัวไหนอนุญาตให้ `DELETE` บนไฟล์ประเภทนี้เลย**

`DELETE` มีไว้ใช้กับไฟล์ประเภท **`INDEXED`** และ **`RELATIVE`** เท่านั้น (สอนใน Part 028–029)
เพราะไฟล์สองประเภทนี้มีโครงสร้างข้อมูลพิเศษ (B-Tree Index หรือ Relative Record Number) ที่ทำให้
"ทำเครื่องหมายว่า record นี้ว่างแล้ว" ได้โดยไม่ต้องขยับ record อื่นเลย

### แล้วถ้าอยากลบ record จากไฟล์ตามลำดับต้องทำอย่างไร

เนื้อหาเต็มของ**เทคนิคทดแทน** 2 แบบที่ใช้กันจริงในอุตสาหกรรมจะสอนในขั้นตอนที่ 248–249 ถัดไป:

1. **Copy-Skip Technique**: สร้างไฟล์ใหม่ทั้งไฟล์ โดยคัดลอกทุก record ยกเว้นตัวที่ต้องการลบ
   (ขั้นตอนที่ 248)
2. **Logical Delete (Soft Delete)**: ไม่ลบ record จริง แค่เปลี่ยนค่า field สถานะให้เป็น "ถูกลบแล้ว"
   แล้วให้โปรแกรมอื่นกรองข้ามมันไปตอนอ่าน (ขั้นตอนที่ 249) — ใช้ `REWRITE` ที่เรียนมาแล้วนั่นเอง

### ข้อควรระวัง

- อย่าพยายาม "หลอก" compiler ด้วยการเปลี่ยน `ORGANIZATION` เป็นอย่างอื่นเพียงเพื่อให้ `DELETE`
  ผ่านโดยไม่เข้าใจผลกระทบ — `INDEXED`/`RELATIVE` มีข้อกำหนดเรื่อง `RECORD KEY`/`RELATIVE KEY`
  เพิ่มเติมที่ต้องเรียนรู้อย่างถูกต้องใน Part 028–029
- Error นี้เป็น **compile-time error** ไม่ใช่ runtime error ซึ่งจริง ๆ แล้วเป็นเรื่องดี เพราะเราจะ
  รู้ปัญหาทันทีตอนคอมไพล์ ไม่ต้องรอให้โปรแกรมรันจริงแล้วค่อยพบปัญหา

### แบบฝึกหัดที่ 247.1

**โจทย์**: จงอธิบายด้วยคำพูดของตัวเองว่าทำไมการลบ record กลางไฟล์ตามลำดับจึงมีต้นทุนสูงกว่าการลบ
record กลางไฟล์แบบ Indexed มาก

**เฉลย**: ไฟล์ตามลำดับเก็บข้อมูลเรียงต่อกันเป็นสายไบต์อย่างต่อเนื่องบนดิสก์โดยไม่มีโครงสร้างช่วย
ค้นหาเพิ่มเติม การลบ record กลางไฟล์จึงจำเป็นต้อง**อ่านและเขียนใหม่ทุก record ที่อยู่หลังจากตำแหน่ง
ที่ลบ**เพื่อปิดช่องว่างที่เกิดขึ้น ซึ่งมีต้นทุนแปรผันตามขนาดไฟล์ทั้งหมด (ยิ่งไฟล์ใหญ่ยิ่งช้า) ในขณะที่
ไฟล์ Indexed มีโครงสร้าง Index (เช่น B-Tree) แยกต่างหากที่ชี้ไปยังตำแหน่งของแต่ละ record การ "ลบ"
จึงทำได้เพียงแค่ทำเครื่องหมายในโครงสร้าง Index ว่า record นั้นใช้ไม่ได้แล้ว โดยไม่ต้องขยับข้อมูล
ที่เหลือเลย ทำให้มีต้นทุนคงที่ (constant time) ไม่ขึ้นกับขนาดไฟล์

---

## ขั้นตอนที่ 248: เทคนิคทดแทน DELETE แบบที่ 1 — Copy-Skip Technique

### แนวคิด: สร้างไฟล์ใหม่โดยข้าม Record ที่ต้องการลบ

เทคนิคที่ใช้กันมากที่สุดในงาน Batch Processing จริงสำหรับ "ลบ" record จากไฟล์ตามลำดับคือ
**อ่านไฟล์เดิมทั้งหมด แล้วเขียนทุก record ยกเว้นตัวที่ต้องการลบลงไฟล์ใหม่** จากนั้นไฟล์ใหม่จะกลายเป็น
ไฟล์หลักตัวใหม่แทนที่ไฟล์เดิม (ในงานจริงมักทำโดยเปลี่ยนชื่อไฟล์ผ่าน script หรือ JCL หลังโปรแกรม
COBOL รันเสร็จ)

### ตัวอย่าง: ลบลูกค้ารหัส C0002 ด้วย Copy-Skip

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP248-COPY-SKIP-DELETE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT OLD-FILE ASSIGN TO "OLD248.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT NEW-FILE ASSIGN TO "NEW248.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  OLD-FILE.
       01  OLD-RECORD.
           05  OLD-ID                  PIC X(5).
           05  OLD-NAME                PIC X(15).

       FD  NEW-FILE.
       01  NEW-RECORD                  PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG                 PIC X VALUE "N".
           88  END-OF-FILE             VALUE "Y".
       01  WS-TARGET-ID                PIC X(5) VALUE "C0002".
       01  WS-KEPT-COUNT               PIC 9(3) VALUE 0.
       01  WS-SKIPPED-COUNT            PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM CREATE-OLD-FILE.
           PERFORM COPY-SKIPPING-TARGET.
           DISPLAY "Kept    : " WS-KEPT-COUNT " record(s)".
           DISPLAY "Skipped : " WS-SKIPPED-COUNT " record(s)".
           STOP RUN.

       CREATE-OLD-FILE.
           OPEN OUTPUT OLD-FILE.
           MOVE "C0001" TO OLD-ID.
           MOVE "SOMCHAI JAIDEE " TO OLD-NAME.
           WRITE OLD-RECORD.
           MOVE "C0002" TO OLD-ID.
           MOVE "SUDA MEECHAI   " TO OLD-NAME.
           WRITE OLD-RECORD.
           MOVE "C0003" TO OLD-ID.
           MOVE "PRASERT KAEWTA " TO OLD-NAME.
           WRITE OLD-RECORD.
           CLOSE OLD-FILE.

       COPY-SKIPPING-TARGET.
      *> Sequential organization cannot DELETE a record from the
      *> middle of a file. The standard workaround is to copy
      *> every record EXCEPT the one we want removed into a brand
      *> new file, then treat the new file as the updated master.
           OPEN INPUT OLD-FILE.
           OPEN OUTPUT NEW-FILE.
           PERFORM UNTIL END-OF-FILE
               READ OLD-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       IF OLD-ID = WS-TARGET-ID
                           ADD 1 TO WS-SKIPPED-COUNT
                       ELSE
                           MOVE OLD-RECORD TO NEW-RECORD
                           WRITE NEW-RECORD
                           ADD 1 TO WS-KEPT-COUNT
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE OLD-FILE.
           CLOSE NEW-FILE.
```

**ผลลัพธ์:**

```
Kept    : 002 record(s)
Skipped : 001 record(s)
```

**เนื้อหาไฟล์ `NEW248.DAT`:**

```
C0001SOMCHAI JAIDEE
C0003PRASERT KAEWTA
```

### อธิบายจุดสำคัญ

- โปรแกรมนี้ใช้ **2 ไฟล์พร้อมกัน**: `OLD-FILE` (เปิดแบบ `INPUT` สำหรับอ่าน) และ `NEW-FILE`
  (เปิดแบบ `OUTPUT` สำหรับเขียน) — COBOL อนุญาตให้เปิดหลายไฟล์พร้อมกันได้เสมอ ตราบใดที่แต่ละไฟล์
  มีชื่อ Internal Name ของตัวเอง (ทบทวนจาก `SELECT` หลายตัวใน `FILE-CONTROL`)
  ตัวแปรนับ (`WS-KEPT-COUNT`, `WS-SKIPPED-COUNT`) ช่วยยืนยันว่าตรรกะการกรองทำงานถูกต้อง
- เงื่อนไข `IF OLD-ID = WS-TARGET-ID` คือหัวใจของเทคนิคนี้: record ที่ตรงเงื่อนไข**จะไม่ถูกเขียน**
  ลง `NEW-FILE` เลย ในขณะที่ record อื่นทั้งหมดถูกคัดลอกไปตามปกติ
- ในงานจริงระดับ production ขั้นตอนถัดไปหลังโปรแกรมนี้รันเสร็จคือการแทนที่ไฟล์เดิมด้วยไฟล์ใหม่
  (เช่น คำสั่ง `mv NEW248.DAT OLD248.DAT` ในระบบ Unix/Linux หรือขั้นตอนจัดการไฟล์ใน JCL บน
  Mainframe) ซึ่งเป็นงานระดับ **Operating System/Job Control** ไม่ใช่ส่วนของโปรแกรม COBOL เอง

### ข้อควรระวัง

- เทคนิคนี้มีต้นทุนตามที่อธิบายไว้ในขั้นตอนที่ 247: ต้องอ่าน-เขียนทั้งไฟล์ทุกครั้งที่ต้องการลบ
  แม้จะลบแค่ 1 record จาก 1 ล้าน record ก็ต้องประมวลผลทั้ง 1 ล้าน record เสมอ (เหมาะกับงาน Batch
  ตอนกลางคืนที่ประมวลผลทีละมาก ๆ อยู่แล้ว ไม่เหมาะกับงานที่ต้องการลบทันทีแบบ Real-time)
  Part 028 จะแสดงให้เห็นว่า Indexed File แก้ปัญหานี้ได้อย่างไร
- ระวังอย่าเผลอเปิดไฟล์ปลายทาง (`NEW-FILE`) ด้วยชื่อไฟล์จริงเดียวกันกับไฟล์ต้นทาง (`OLD-FILE`)
  เพราะ `OPEN OUTPUT` จะลบไฟล์ต้นทางทิ้งทันทีก่อนที่จะอ่านมันเสร็จ ทำให้ข้อมูลสูญหายถาวร — ต้องใช้
  ชื่อไฟล์จริงที่ต่างกันเสมอระหว่างขั้นตอนประมวลผล แล้วค่อยเปลี่ยนชื่อไฟล์ทีหลัง

### แบบฝึกหัดที่ 248.1

**โจทย์**: จงปรับโปรแกรมข้างต้นให้ลบลูกค้าที่มี**เงินคงเหลือน้อยกว่า 100 บาท** แทนที่จะลบตาม ID
ที่กำหนดตายตัว (สมมติว่าเพิ่ม field `OLD-BALANCE PIC 9(7)V99` เข้าไปใน `OLD-RECORD`)

**เฉลย**: เปลี่ยนเงื่อนไขใน `COPY-SKIPPING-TARGET` จาก `IF OLD-ID = WS-TARGET-ID` เป็น:

```cobol
                       IF OLD-BALANCE < 100.00
                           ADD 1 TO WS-SKIPPED-COUNT
                       ELSE
                           MOVE OLD-RECORD TO NEW-RECORD
                           WRITE NEW-RECORD
                           ADD 1 TO WS-KEPT-COUNT
                       END-IF
```

หลักการเดิมยังคงเหมือนเดิมทุกประการ เพียงแค่เปลี่ยนเงื่อนไขการกรองจากการเปรียบเทียบ ID เป็นการ
เปรียบเทียบยอดเงิน แสดงให้เห็นว่าเทคนิค Copy-Skip นี้ยืดหยุ่นสามารถใช้กับเงื่อนไขการลบแบบใดก็ได้

---

## ขั้นตอนที่ 249: เทคนิคทดแทน DELETE แบบที่ 2 — Logical Delete (Soft Delete)

### แนวคิด: ไม่ลบจริง แค่ทำเครื่องหมาย

เทคนิคที่สองที่นิยมไม่แพ้กัน (และมักได้รับความนิยมมากกว่าในระบบธุรกิจจริง เช่น ระบบธนาคาร) คือ
**Logical Delete** หรือ **Soft Delete**: แทนที่จะลบ record ออกจากไฟล์จริง ๆ เราเพียงแค่เปลี่ยนค่า
field สถานะของ record นั้นให้เป็น "ถูกลบแล้ว" (`REWRITE` ที่เรียนในขั้นตอนที่ 245) แล้วให้ทุก
โปรแกรมที่อ่านไฟล์นี้**กรอง record ที่มีสถานะนี้ทิ้งไป**เสมอ ข้อดีสำคัญคือ**ข้อมูลไม่สูญหายจริง**
เก็บไว้เป็นประวัติ (Audit Trail) ได้ ซึ่งสำคัญมากในธุรกิจที่ต้องตรวจสอบย้อนหลัง

### ตัวอย่าง: ทำเครื่องหมายลูกค้าเป็น "ถูกลบ" ด้วย REWRITE

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP249-LOGICAL-DELETE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST249.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUST-RECORD.
           05  CUST-ID                 PIC X(5).
           05  CUST-NAME               PIC X(15).
           05  CUST-STATUS             PIC X(1).
               88  CUST-ACTIVE         VALUE "A".
               88  CUST-DELETED        VALUE "D".

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG                 PIC X VALUE "N".
           88  END-OF-FILE             VALUE "Y".
       01  WS-TARGET-ID                PIC X(5) VALUE "C0002".

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM CREATE-CUSTOMERS.
           PERFORM MARK-AS-DELETED.
           PERFORM LIST-ACTIVE-ONLY.
           STOP RUN.

       CREATE-CUSTOMERS.
           OPEN OUTPUT CUSTOMER-FILE.
           MOVE "C0001" TO CUST-ID.
           MOVE "SOMCHAI JAIDEE " TO CUST-NAME.
           SET CUST-ACTIVE TO TRUE.
           WRITE CUST-RECORD.

           MOVE "C0002" TO CUST-ID.
           MOVE "SUDA MEECHAI   " TO CUST-NAME.
           SET CUST-ACTIVE TO TRUE.
           WRITE CUST-RECORD.

           MOVE "C0003" TO CUST-ID.
           MOVE "PRASERT KAEWTA " TO CUST-NAME.
           SET CUST-ACTIVE TO TRUE.
           WRITE CUST-RECORD.
           CLOSE CUSTOMER-FILE.
           DISPLAY "CUST249.DAT created with 3 active customers.".

       MARK-AS-DELETED.
      *> OPEN I-O lets us READ a record and REWRITE it back in
      *> place, changing only its status flag - the record is
      *> never physically removed from the file.
           OPEN I-O CUSTOMER-FILE.
           PERFORM UNTIL END-OF-FILE
               READ CUSTOMER-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       IF CUST-ID = WS-TARGET-ID
                           SET CUST-DELETED TO TRUE
                           REWRITE CUST-RECORD
                           DISPLAY "Marked " CUST-ID
                               " as logically deleted."
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE CUSTOMER-FILE.

       LIST-ACTIVE-ONLY.
           MOVE "N" TO WS-EOF-FLAG.
           OPEN INPUT CUSTOMER-FILE.
           DISPLAY "---- Active customers only ----".
           PERFORM UNTIL END-OF-FILE
               READ CUSTOMER-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       IF CUST-ACTIVE
                           DISPLAY CUST-ID " " CUST-NAME
                       END-IF
               END-READ
           END-PERFORM.
           CLOSE CUSTOMER-FILE.
```

**ผลลัพธ์:**

```
CUST249.DAT created with 3 active customers.
Marked C0002 as logically deleted.
---- Active customers only ----
C0001 SOMCHAI JAIDEE
C0003 PRASERT KAEWTA
```

**เนื้อหาไฟล์ `CUST249.DAT` หลังรัน** (สังเกตว่า C0002 ยังอยู่ในไฟล์จริง แค่สถานะเปลี่ยน):

```
C0001SOMCHAI JAIDEE A
C0002SUDA MEECHAI   D
C0003PRASERT KAEWTA A
```

### อธิบายจุดสำคัญ

- `CUST-STATUS` ออกแบบให้เป็น field สุดท้ายที่**ไม่มีทางเป็นช่องว่าง** (ต้องเป็น `"A"` หรือ `"D"`
  เสมอ) — นี่คือการนำกฎทองจากขั้นตอนที่ 246 มาใช้จริงโดยตรง ทำให้ `REWRITE` ในโปรแกรมนี้ปลอดภัย
  100% ไม่มีความเสี่ยงเรื่องความยาวบรรทัดถูกตัดทิ้ง
- เทคนิคนี้ใช้ `88` level (Condition Name จาก Part 010) สองตัวคู่กัน (`CUST-ACTIVE`,
  `CUST-DELETED`) ทำให้โค้ดอ่านง่ายเหมือนภาษาอังกฤษทั้ง `SET CUST-DELETED TO TRUE` และ
  `IF CUST-ACTIVE`
- เปรียบเทียบกับ Copy-Skip (ขั้นตอนที่ 248): Logical Delete **เร็วกว่ามาก** เพราะแก้แค่ record
  เดียวด้วย `REWRITE` ไม่ต้องอ่าน-เขียนทั้งไฟล์ใหม่ แต่ข้อเสียคือไฟล์จะ**ไม่เล็กลง**เลย (record
  ที่ถูกลบยังกินพื้นที่อยู่) และทุกโปรแกรมที่อ่านไฟล์นี้ในอนาคต**ต้องจำเสมอ**ว่าต้องกรอง
  `CUST-DELETED` ทิ้ง มิฉะนั้นจะเห็นข้อมูลที่ควรถูกลบไปแล้วปะปนอยู่

### ข้อควรระวัง

- Logical Delete เปลี่ยน "ภาระ" จากตอนลบ ไปเป็น "ภาระที่ทุกโปรแกรมอ่านไฟล์ต้องกรองเอง" — ถ้ามี
  โปรแกรมใดโปรแกรมหนึ่งในระบบลืมกรอง `CUST-DELETED` จะเกิดบั๊กที่ข้อมูลที่ควรถูกลบไปแล้วกลับมา
  แสดงผลอีกครั้ง ซึ่งอาจร้ายแรงมากในระบบธุรกิจจริง (เช่น แสดงบัญชีลูกค้าที่ปิดไปแล้วในรายงาน)
- ในระบบจริงระดับองค์กร มักผสมทั้งสองเทคนิคเข้าด้วยกัน: ใช้ Logical Delete สำหรับการทำงานประจำวัน
  (เร็ว ตรวจสอบย้อนหลังได้) แล้วรันงาน Batch ตอนกลางคืนหรือรายเดือนด้วยเทคนิค Copy-Skip เพื่อ
  "อัดไฟล์" (compact) ลบ record ที่มีสถานะ deleted ทิ้งจริง ๆ เป็นระยะเพื่อประหยัดพื้นที่

### แบบฝึกหัดที่ 249.1

**โจทย์**: จงอธิบายว่าทำไม Logical Delete จึงเหมาะกับระบบธนาคารมากกว่า Copy-Skip Technique
สำหรับงาน "ปิดบัญชีลูกค้า" ที่เกิดขึ้นระหว่างวันทำการ

**เฉลย**: เพราะระบบธนาคารต้องการ**ความรวดเร็วและ Audit Trail** สำหรับการทำธุรกรรมระหว่างวันทำการ
Logical Delete ทำงานเร็วมาก (แก้ไขแค่ record เดียวด้วย `REWRITE`) เหมาะกับการตอบสนองทันทีที่ลูกค้า
ขอปิดบัญชี ในขณะที่ Copy-Skip ต้องประมวลผลทั้งไฟล์ซึ่งช้าเกินไปสำหรับงาน Real-time นอกจากนี้
Logical Delete ยังเก็บประวัติไว้ในไฟล์ (สถานะ "ปิดแล้ว" แทนที่จะหายไปเลย) ซึ่งจำเป็นสำหรับการตรวจสอบ
ทางบัญชีและกฎหมายที่มักกำหนดให้สถาบันการเงินต้องเก็บประวัติธุรกรรมและบัญชีย้อนหลังได้หลายปี
ส่วนงาน "อัดไฟล์" ด้วย Copy-Skip ค่อยทำเป็นงาน Batch ตอนกลางคืนหรือช่วงเวลาที่ระบบไม่ยุ่งแทน

---

## ขั้นตอนที่ 250: โปรแกรมรวม — วงจรการอัปเดตไฟล์แบบสมบูรณ์

### โจทย์: ระบบฝากเงินเข้าบัญชีพร้อมบันทึกประวัติการเปลี่ยนแปลง

ขั้นตอนสุดท้ายของ Part นี้จะรวมทุกคำสั่งที่เรียนมา: `OPEN` ทุกโหมด, `READ`, `WRITE`, `REWRITE`,
พร้อมการทำงานกับ**สองไฟล์พร้อมกัน**อย่างปลอดภัยตามกฎทองที่เรียนมาทั้งหมด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP250-FULL-UPDATE-CYCLE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-FILE ASSIGN TO "ACCOUNT250.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.
           SELECT LOG-FILE ASSIGN TO "CHANGELOG250.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-FILE.
       01  ACCT-RECORD.
           05  ACCT-ID                 PIC X(5).
           05  ACCT-NAME               PIC X(15).
           05  ACCT-BALANCE            PIC 9(7)V99.

       FD  LOG-FILE.
       01  LOG-RECORD                  PIC X(60).

       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG                 PIC X VALUE "N".
           88  END-OF-FILE             VALUE "Y".
       01  WS-DEPOSIT-AMOUNT           PIC 9(5)V99.
       01  WS-OLD-BALANCE              PIC 9(7)V99.
       01  WS-DISPLAY-OLD              PIC ZZZ,ZZ9.99.
       01  WS-DISPLAY-NEW              PIC ZZZ,ZZ9.99.
       01  WS-DISPLAY-DEPOSIT          PIC ZZ,ZZ9.99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM CREATE-ACCOUNTS.
           PERFORM APPLY-DEPOSITS.
           PERFORM PRINT-FINAL-REPORT.
           STOP RUN.

       CREATE-ACCOUNTS.
           OPEN OUTPUT ACCOUNT-FILE.
           MOVE "A001" TO ACCT-ID.
           MOVE "SOMCHAI JAIDEE " TO ACCT-NAME.
           MOVE 1000.00 TO ACCT-BALANCE.
           WRITE ACCT-RECORD.

           MOVE "A002" TO ACCT-ID.
           MOVE "SUDA MEECHAI   " TO ACCT-NAME.
           MOVE 2500.50 TO ACCT-BALANCE.
           WRITE ACCT-RECORD.

           MOVE "A003" TO ACCT-ID.
           MOVE "PRASERT KAEWTA " TO ACCT-NAME.
           MOVE 300.00 TO ACCT-BALANCE.
           WRITE ACCT-RECORD.
           CLOSE ACCOUNT-FILE.
           DISPLAY "ACCOUNT250.DAT created with 3 accounts.".

       APPLY-DEPOSITS.
      *> OPEN I-O lets this single paragraph both READ every
      *> account and REWRITE the ones that need a new balance,
      *> while a second file records a text log of every change.
           OPEN I-O ACCOUNT-FILE.
           OPEN OUTPUT LOG-FILE.

           PERFORM UNTIL END-OF-FILE
               READ ACCOUNT-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       PERFORM DECIDE-AND-APPLY-DEPOSIT
               END-READ
           END-PERFORM.

           CLOSE ACCOUNT-FILE.
           CLOSE LOG-FILE.

       DECIDE-AND-APPLY-DEPOSIT.
           MOVE 0 TO WS-DEPOSIT-AMOUNT.
           IF ACCT-ID = "A001"
               MOVE 500.00 TO WS-DEPOSIT-AMOUNT
           END-IF.
           IF ACCT-ID = "A003"
               MOVE 150.25 TO WS-DEPOSIT-AMOUNT
           END-IF.

           IF WS-DEPOSIT-AMOUNT > 0
               MOVE ACCT-BALANCE TO WS-OLD-BALANCE
               ADD WS-DEPOSIT-AMOUNT TO ACCT-BALANCE
               REWRITE ACCT-RECORD

               MOVE WS-OLD-BALANCE TO WS-DISPLAY-OLD
               MOVE ACCT-BALANCE TO WS-DISPLAY-NEW
               MOVE WS-DEPOSIT-AMOUNT TO WS-DISPLAY-DEPOSIT
      *> Clear LOG-RECORD first (step 236's rule) - STRING never
      *> touches bytes beyond what it fills, so leftover garbage
      *> could remain otherwise.
               MOVE SPACES TO LOG-RECORD
               STRING ACCT-ID DELIMITED BY SIZE
                   " deposit " WS-DISPLAY-DEPOSIT DELIMITED BY SIZE
                   " old=" WS-DISPLAY-OLD DELIMITED BY SIZE
                   " new=" WS-DISPLAY-NEW DELIMITED BY SIZE
                   INTO LOG-RECORD
               WRITE LOG-RECORD
           END-IF.

       PRINT-FINAL-REPORT.
           MOVE "N" TO WS-EOF-FLAG.
           OPEN INPUT ACCOUNT-FILE.
           DISPLAY "---- Final account balances ----".
           PERFORM UNTIL END-OF-FILE
               READ ACCOUNT-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       MOVE ACCT-BALANCE TO WS-DISPLAY-NEW
                       DISPLAY ACCT-ID " " ACCT-NAME " "
                           WS-DISPLAY-NEW
               END-READ
           END-PERFORM.
           CLOSE ACCOUNT-FILE.

           MOVE "N" TO WS-EOF-FLAG.
           OPEN INPUT LOG-FILE.
           DISPLAY "---- Change log ----".
           PERFORM UNTIL END-OF-FILE
               READ LOG-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                   NOT AT END
                       DISPLAY LOG-RECORD
               END-READ
           END-PERFORM.
           CLOSE LOG-FILE.
```

**ผลลัพธ์:**

```
ACCOUNT250.DAT created with 3 accounts.
---- Final account balances ----
A001  SOMCHAI JAIDEE    1,500.00
A002  SUDA MEECHAI      2,500.50
A003  PRASERT KAEWTA      450.25
---- Change log ----
A001  deposit    500.00 old=  1,000.00 new=  1,500.00
A003  deposit    150.25 old=    300.00 new=    450.25
```

### อธิบายภาพรวมของโปรแกรม

โปรแกรมนี้ผสานทุกเทคนิคของ Part 025 เข้าด้วยกัน:

- **`CREATE-ACCOUNTS`**: ใช้ `OPEN OUTPUT`/`WRITE`/`CLOSE` ตามปกติเพื่อสร้างไฟล์ตั้งต้น
- **`APPLY-DEPOSITS`**: เปิด `ACCOUNT-FILE` ด้วย `OPEN I-O` (เพื่อ `READ` แล้ว `REWRITE` ได้ในไฟล์
  เดียวกัน) พร้อมกับเปิด `LOG-FILE` ด้วย `OPEN OUTPUT` แยกต่างหาก (สาธิตการเปิดหลายไฟล์พร้อมกัน
  ในโหมดที่ต่างกัน)
- `ACCT-BALANCE` เป็น field ตัวเลขและอยู่**ท้ายสุด**ของ `ACCT-RECORD` — ยึดตามกฎทองของขั้นตอนที่
  246 ทำให้ `REWRITE` ปลอดภัยเสมอไม่ว่ายอดเงินจะเปลี่ยนไปกี่หลักก็ตาม
- **`PRINT-FINAL-REPORT`**: เปิดทั้งสองไฟล์กลับมาอ่าน (`OPEN INPUT`) เพื่อแสดงผลลัพธ์สุดท้าย
  ยืนยันว่าทั้งการอัปเดตยอดเงินและการบันทึก log ทำงานถูกต้องตรงกัน (500.00 + 1,000.00 = 1,500.00
  และ 150.25 + 300.00 = 450.25 ตรงกันทั้งสองไฟล์)

### ข้อควรระวัง

- สังเกตว่าก่อนเปิด `ACCOUNT-FILE`/`LOG-FILE` ใหม่แต่ละครั้งใน `PRINT-FINAL-REPORT` เราต้องรีเซ็ต
  `WS-EOF-FLAG` กลับเป็น `"N"` ด้วยตนเอง (`MOVE "N" TO WS-EOF-FLAG`) เพราะตัวแปรนี้ยังคงค่า `"Y"`
  ค้างมาจากลูปก่อนหน้าใน `APPLY-DEPOSITS` — ถ้าลืมขั้นตอนนี้ ลูป `PERFORM UNTIL END-OF-FILE` รอบ
  ถัดไปจะไม่ทำงานเลยแม้แต่ครั้งเดียว (ทบทวนข้อผิดพลาดแบบเดียวกันจาก Part 023 ขั้นตอนที่ 226)
- `MOVE SPACES TO LOG-RECORD` ก่อน `STRING` ทุกครั้งคือการนำกฎจาก Part 024 ขั้นตอนที่ 236 มาใช้
  จริง — ถ้าตัดบรรทัดนี้ทิ้งจะเจอ error `status = 71` ทันทีเหมือนที่พิสูจน์มาแล้ว เพราะ `STRING`
  เติมข้อมูลแค่ส่วนที่ระบุ ไม่ได้ล้างพื้นที่ทั้งหมดของ `LOG-RECORD` ให้อัตโนมัติ

### แบบฝึกหัดที่ 250.1 (แบบฝึกหัดสรุปท้าย Part)

**โจทย์**: จงเพิ่มความสามารถให้โปรแกรมสามารถ**ถอนเงิน** (withdrawal) จากบัญชี `A002` จำนวน 200.50
บาทด้วย นอกเหนือจากการฝากเงินที่มีอยู่แล้ว โดยต้องบันทึกลง log ด้วยเช่นกัน

**เฉลย**: เพิ่มตัวแปรและเงื่อนไขใน `DECIDE-AND-APPLY-DEPOSIT` (อาจเปลี่ยนชื่อ paragraph เป็น
`DECIDE-AND-APPLY-CHANGE` ให้ตรงความหมายมากขึ้นในโปรแกรมจริง):

```cobol
           IF ACCT-ID = "A002"
               MOVE ACCT-BALANCE TO WS-OLD-BALANCE
               SUBTRACT 200.50 FROM ACCT-BALANCE
               REWRITE ACCT-RECORD

               MOVE WS-OLD-BALANCE TO WS-DISPLAY-OLD
               MOVE ACCT-BALANCE TO WS-DISPLAY-NEW
               MOVE SPACES TO LOG-RECORD
               STRING ACCT-ID DELIMITED BY SIZE
                   " withdraw 200.50" DELIMITED BY SIZE
                   " old=" WS-DISPLAY-OLD DELIMITED BY SIZE
                   " new=" WS-DISPLAY-NEW DELIMITED BY SIZE
                   INTO LOG-RECORD
               WRITE LOG-RECORD
           END-IF
```

หลังเพิ่มโค้ดนี้ บัญชี `A002` จะมียอดคงเหลือใหม่ 2,500.50 - 200.50 = **2,300.00** และมี log
บันทึกการถอนเงินเพิ่มเติมแยกจาก log การฝากเงินของบัญชีอื่น ๆ แสดงให้เห็นว่าโครงสร้างโปรแกรมแบบ
`OPEN I-O` + `REWRITE` + Log File รองรับการขยายตรรกะทางธุรกิจได้อย่างยืดหยุ่น

---

## สรุปท้ายบท

ใน Part นี้เราได้เจาะลึกคำสั่งควบคุมไฟล์ทั้งหมดของ COBOL อย่างครบถ้วน:

- `OPEN` ครบทั้ง 4 โหมด (`INPUT`, `OUTPUT`, `EXTEND`, `I-O`) และข้อจำกัดของแต่ละโหมด
- `CLOSE` โดยละเอียด รวมถึงอันตรายของการลืม `CLOSE` (แม้ GnuCOBOL จะช่วย implicit close ให้)
  และ `CLOSE ... WITH LOCK`
- `READ...INTO` และ `WRITE...FROM` ที่รวมการ `MOVE` เข้ากับ `READ`/`WRITE` ในคำสั่งเดียว
- `WRITE...ADVANCING` สำหรับควบคุมบรรทัดว่างเบื้องต้น (รายละเอียดเต็มใน Part 038)
- **`REWRITE`** และกฎทองที่พิสูจน์จากการทดลองจริง: ปลอดภัยเมื่อบรรทัดเดิมเต็มความกว้าง `PIC`
  เท่านั้น มิฉะนั้นข้อมูลจะถูกตัดทิ้งแบบเงียบ ๆ โดยไม่มี error เตือน
- **`DELETE`** และเหตุผลเชิงกายภาพที่ใช้กับไฟล์ตามลำดับไม่ได้เลย (compile-time error ทันที)
- เทคนิคทดแทน `DELETE` สองแบบที่ใช้จริงในอุตสาหกรรม: **Copy-Skip Technique** และ
  **Logical Delete (Soft Delete)** พร้อมข้อดี-ข้อเสียของแต่ละแบบ
- โปรแกรมรวมที่สาธิตวงจรการอัปเดตไฟล์แบบสมบูรณ์ พร้อมการบันทึกประวัติการเปลี่ยนแปลงคู่ขนาน

Part ถัดไป (**Part 026**) จะพาคุณไปประยุกต์ใช้ทุกเทคนิคที่เรียนมาใน Part 023–025 กับปัญหาที่พบบ่อย
ที่สุดในงาน Batch Processing จริง: **การประมวลผลไฟล์แบบ Master-Detail (Matching Records)** ซึ่งเป็น
เทคนิคการจับคู่ข้อมูลจากไฟล์ 2 ไฟล์ขึ้นไปที่เรียงลำดับตาม Key เดียวกัน (เช่น ไฟล์ Master ลูกค้ากับ
ไฟล์ Transaction ประจำวัน) ให้ทำงานร่วมกันอย่างมีประสิทธิภาพ

**[← กลับไป Part 024](part-024-file-section-fd.md)** | **[ไปยัง Part 026: การประมวลผลไฟล์แบบ Master-Detail →](part-026-master-detail-processing.md)**
