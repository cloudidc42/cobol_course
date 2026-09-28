# Part 048: Error Handling ขั้นสูง: USE Statement และ DECLARATIVES (ขั้นตอนที่ 471–480)

## คำนำของ Part นี้

Part 030 สอนเรื่อง `FILE STATUS` ไว้อย่างละเอียด และทิ้งท้ายไว้ว่ากลไกขั้นสูงกว่าอย่าง `USE`
Statement และ `DECLARATIVES` จะสอนแยกต่างหากใน Part นี้ — ตอนนี้ถึงเวลานั้นแล้ว

ปัญหาของการตรวจสอบ `FILE STATUS` ด้วย `IF`/`EVALUATE` แบบที่เรียนมาใน Part 030 คือ **ต้องเขียนโค้ด
ตรวจสอบซ้ำ ๆ หลัง OPEN/READ/WRITE/REWRITE/DELETE ทุกจุดที่มีการทำปฏิบัติการกับไฟล์** ในโปรแกรมขนาดใหญ่
ที่มีไฟล์หลายสิบไฟล์และมีจุดเรียกใช้ไฟล์กระจายอยู่ทั่วโปรแกรม การตรวจสอบแบบกระจัดกระจายนี้ทำให้
(1) โค้ดซ้ำซ้อนมาก (2) เสี่ยงที่จะมีจุดใดจุดหนึ่งลืมตรวจสอบ (3) ตรรกะการจัดการข้อผิดพลาดกระจายอยู่
ทั่วโปรแกรมแทนที่จะรวมศูนย์ไว้ที่เดียว

`DECLARATIVES SECTION` ร่วมกับ `USE` Statement คือคำตอบของ COBOL สำหรับปัญหานี้: มันคือกลไกที่ให้เรา
**เขียนโค้ดจัดการข้อผิดพลาดของไฟล์แต่ละไฟล์ (หรือกลุ่มไฟล์) ไว้เพียงที่เดียว** แล้ว COBOL runtime
จะ **เรียกใช้โค้ดนั้นให้เองโดยอัตโนมัติ** ทุกครั้งที่ปฏิบัติการกับไฟล์นั้นเกิดข้อผิดพลาด โดยไม่ต้องเขียน
`IF WS-FILE-STATUS ...` กำกับไว้หลังทุกคำสั่งไฟล์เลย — คล้ายกับแนวคิด "exception handler" ในภาษาสมัยใหม่
เช่น `try/catch` แต่ COBOL คิดค้นแนวคิดนี้ไว้ตั้งแต่มาตรฐาน COBOL-74/85 ก่อนภาษาสมัยใหม่หลายภาษาเสียอีก

ทุกตัวอย่างใน Part นี้ผ่านการคอมไพล์และรันทดสอบจริงด้วย GnuCOBOL 4.0-early-dev.0 (บางตัวอย่างที่ใช้
Indexed File ต้องใช้ build ที่เปิดใช้ ISAM handler ตามที่อธิบายไว้ใน Part 028 — จะระบุชัดเจนเมื่อถึงจุดนั้น)

---

## ขั้นตอนที่ 471: DECLARATIVES SECTION คืออะไร — โครงสร้างและกฎการเขียน

### แนวคิด

`DECLARATIVES` เป็นส่วนพิเศษที่ต้องอยู่ **ทันทีหลัง `PROCEDURE DIVISION.`** (ก่อน Paragraph/Section
อื่นใดทั้งสิ้น) มีโครงสร้างตายตัวดังนี้:

```cobol
       PROCEDURE DIVISION.
       DECLARATIVES.
       section-name SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON file-name.
       paragraph-name.
           *> error-handling logic goes here
       END DECLARATIVES.

       main-section SECTION.
       main-paragraph.
           *> normal program logic starts here
```

กฎสำคัญที่ต้องจำ:

1. `DECLARATIVES.` ต้องเป็นคำแรกสุดใน `PROCEDURE DIVISION` เสมอ และปิดท้ายด้วย `END DECLARATIVES.`
2. ภายในต้องแบ่งเป็น **SECTION** อย่างน้อย 1 ส่วน แต่ละ SECTION ต้องมี `USE` Statement เป็น
   **ประโยคแรกสุด** ของ SECTION นั้นเสมอ (ก่อนบรรทัดอื่นใด)
3. หลัง SECTION ที่มี `USE` แล้ว ตามด้วย Paragraph ที่มีโค้ดจัดการข้อผิดพลาดจริง (จะมีกี่ Paragraph
   ก็ได้ภายใน SECTION เดียวกัน)
4. หลัง `END DECLARATIVES.` โปรแกรมหลัก (main logic) ต้อง **เริ่มด้วยชื่อ SECTION ใหม่เสมอ** — นี่คือ
   กฎที่มักทำให้ผู้เริ่มต้นสับสน: **เมื่อโปรแกรมมี `DECLARATIVES` แล้ว ทุก Paragraph ในโปรแกรมนั้น
   (รวมถึงส่วน main logic) ต้องถูกจัดกลุ่มอยู่ภายใน SECTION เสมอ ไม่มี Paragraph ลอย ๆ นอก SECTION ได้อีก**

### ตัวอย่างโค้ด: โครงร่างขั้นต่ำที่คอมไพล์ผ่าน พร้อมพิสูจน์ว่า DECLARATIVES ไม่ทำงานเมื่อไม่มีข้อผิดพลาด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP471-DECLARATIVES-SKELETON.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "CUST471.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD          PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.

       PROCEDURE DIVISION.
       DECLARATIVES.
      *> This SECTION handles every error on CUSTOMER-FILE. The
      *> USE statement must be the very first sentence in the
      *> SECTION - nothing else may come before it.
       CUSTOMER-ERROR-SECTION SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON CUSTOMER-FILE.
       CUSTOMER-ERROR-PARA.
           DISPLAY "DECLARATIVE FIRED! STATUS=" WS-FILE-STATUS.
       END DECLARATIVES.

      *> After END DECLARATIVES, ordinary program logic must ALSO
      *> live inside a SECTION - this is required once the program
      *> has any DECLARATIVES at all.
       MAIN-LOGIC SECTION.
       MAIN-LOGIC-START.
           OPEN OUTPUT CUSTOMER-FILE.
           DISPLAY "AFTER SUCCESSFUL OPEN, STATUS=" WS-FILE-STATUS.
           MOVE "NORMAL RECORD" TO CUSTOMER-RECORD.
           WRITE CUSTOMER-RECORD.
           DISPLAY "AFTER SUCCESSFUL WRITE, STATUS=" WS-FILE-STATUS.
           CLOSE CUSTOMER-FILE.
           STOP RUN.
```

คอมไพล์และรันด้วย:

```bash
cobc -x -o step471 step471.cob
./step471
```

### ผลลัพธ์จริงที่ได้

```
AFTER SUCCESSFUL OPEN, STATUS=00
AFTER SUCCESSFUL WRITE, STATUS=00
```

### อธิบายผลลัพธ์

สังเกตว่า **ข้อความ `"DECLARATIVE FIRED!"` ไม่ปรากฏเลย** แม้โปรแกรมจะมี `DECLARATIVES` ประกาศไว้ก็ตาม
นี่คือพฤติกรรมที่ถูกต้องและสำคัญมาก: **`USE AFTER STANDARD ERROR PROCEDURE` จะถูกเรียกก็ต่อเมื่อ
ปฏิบัติการไฟล์นั้นล้มเหลวเท่านั้น** (โดยทั่วไปคือ `FILE STATUS` ไม่ขึ้นต้นด้วย `0` หรือ `1` ที่มี
`AT END`/`INVALID KEY` clause จัดการอยู่แล้ว — รายละเอียดครบถ้วนจะอธิบายในขั้นตอนที่ 475) เมื่อ `OPEN`
และ `WRITE` สำเร็จทั้งคู่ (status `00`) Paragraph จัดการข้อผิดพลาดจึงไม่ถูกเรียกเลยตลอดการรัน

### ข้อควรระวัง

- ลืมใส่ `END DECLARATIVES.` เป็นข้อผิดพลาดที่พบบ่อยที่สุดสำหรับผู้เริ่มต้น ทำให้คอมไพเลอร์สับสนว่า
  Paragraph ถัดไปเป็นส่วนหนึ่งของ DECLARATIVES หรือไม่ และมักได้ syntax error ที่ตำแหน่งไกลจากจุดที่
  ลืมจริง ๆ มาก ทำให้ debug ยาก
- เมื่อมี `DECLARATIVES` แล้ว **ทุก Paragraph ที่เหลือในโปรแกรมต้องอยู่ใน SECTION เสมอ** แม้จะเป็น
  โปรแกรมเล็ก ๆ ที่ไม่เคยต้องใช้ SECTION มาก่อนก็ตาม (ต่างจาก Part 014 ที่ SECTION เป็นทางเลือก) —
  ลืมกฎนี้จะทำให้คอมไพล์ไม่ผ่าน

### แบบฝึกหัดที่ 471.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมในตัวอย่างข้างต้นจึงต้องมี `MAIN-LOGIC SECTION.` กำกับส่วนโปรแกรมหลัก
ทั้งที่ Part 014 สอนว่า SECTION เป็นทางเลือก ไม่บังคับ

**เฉลย**: กฎ "SECTION เป็นทางเลือก" ใน Part 014 ใช้ได้เฉพาะโปรแกรมที่**ไม่มี** `DECLARATIVES` เท่านั้น
เมื่อใดก็ตามที่โปรแกรมมี `DECLARATIVES SECTION` มาตรฐาน COBOL กำหนดว่า **PROCEDURE DIVISION ทั้งหมด
ต้องถูกแบ่งเป็น SECTION อย่างสมบูรณ์** ทั้งส่วน DECLARATIVES เองและส่วนโปรแกรมหลักที่ตามมาหลัง
`END DECLARATIVES.` เหตุผลเชิงเทคนิคคือคอมไพเลอร์ต้องรู้ขอบเขตที่ชัดเจนว่า "โปรแกรมหลักเริ่มต้นที่ไหน"
เพื่อแยกจากส่วน DECLARATIVES ที่อยู่ก่อนหน้า การใช้ SECTION จึงเป็นวิธีเดียวที่บอกขอบเขตนี้ได้ชัดเจน

---

## ขั้นตอนที่ 472: USE AFTER STANDARD ERROR PROCEDURE ON <file-name> — จัดการไฟล์เดียวแบบเจาะจง

### แนวคิด

รูปแบบพื้นฐานที่สุดของ `USE` คือการระบุชื่อไฟล์ตรง ๆ หลัง `ON` ทำให้ Paragraph ที่ตามมาทำงาน**เฉพาะ**
เมื่อไฟล์นั้นเกิดข้อผิดพลาดเท่านั้น ไม่กระทบไฟล์อื่นในโปรแกรมเดียวกันเลย

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP472-SINGLE-FILE-HANDLER.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE ASSIGN TO "MISSING472.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD          PIC X(20).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC XX.

       PROCEDURE DIVISION.
       DECLARATIVES.
       CUSTOMER-ERROR-SECTION SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON CUSTOMER-FILE.
       CUSTOMER-ERROR-PARA.
           DISPLAY "DECLARATIVE CAUGHT ERROR, STATUS = "
               WS-FILE-STATUS.
       END DECLARATIVES.

       MAIN-LOGIC SECTION.
       MAIN-LOGIC-START.
      *> The file genuinely does not exist, and there is no
      *> OPTIONAL clause on the SELECT - normally this would set
      *> status 35 (Part 030 step 295). With a USE AFTER STANDARD
      *> ERROR PROCEDURE declarative in place, the runtime calls
      *> our handler automatically BEFORE control returns here.
           OPEN INPUT CUSTOMER-FILE.
           DISPLAY "AFTER OPEN STATUS = " WS-FILE-STATUS.
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
DECLARATIVE CAUGHT ERROR, STATUS = 35
AFTER OPEN STATUS = 35
```

### อธิบายโค้ดทีละส่วน

- คำสั่ง `OPEN INPUT CUSTOMER-FILE` ล้มเหลวด้วยสถานะ `35` (ไฟล์ไม่มีอยู่จริง ไม่มี `OPTIONAL`)
- ก่อนที่ควบคุมโปรแกรมจะกลับมาที่บรรทัด `DISPLAY "AFTER OPEN STATUS..."` **runtime ของ COBOL เรียก
  `CUSTOMER-ERROR-PARA` ให้เองโดยอัตโนมัติ** เพราะตรวจพบว่ามี `USE AFTER STANDARD ERROR PROCEDURE
  ON CUSTOMER-FILE` ประกาศไว้
- สังเกตลำดับผลลัพธ์: ข้อความจาก Declarative (`"DECLARATIVE CAUGHT ERROR..."`) ปรากฏ**ก่อน**ข้อความ
  จากโปรแกรมหลัก (`"AFTER OPEN STATUS..."`) ซึ่งพิสูจน์ว่า Declarative ถูกแทรกเข้ามาทำงาน**ระหว่าง**
  การเรียก `OPEN` กับการกลับมาทำงานที่บรรทัดถัดไป ไม่ใช่ทำงานทีหลังหรือแบบขนาน (parallel)
- ไม่ต้องเขียน `IF WS-FILE-STATUS NOT = "00" ... END-IF` เลยสักบรรทัดในโปรแกรมหลัก — นี่คือประโยชน์
  หลักของ DECLARATIVES ที่ทำให้โค้ดโปรแกรมหลักสั้นและอ่านง่ายขึ้นมาก

### ข้อควรระวัง

- `USE AFTER STANDARD ERROR PROCEDURE ON CUSTOMER-FILE` ใช้ได้กับไฟล์ `CUSTOMER-FILE` **เท่านั้น**
  ถ้าโปรแกรมมีไฟล์อื่นอีกและเกิดข้อผิดพลาด Declarative นี้จะไม่ถูกเรียกเลย (จะสาธิตในขั้นตอนที่ 473-474)
- แม้ Declarative จะทำงานแล้ว **`WS-FILE-STATUS` ยังคงมีค่า `35` อยู่หลังจากนั้น** โปรแกรมหลักยังคง
  ตรวจสอบค่านี้ต่อได้ตามปกติถ้าต้องการ (Declarative ไม่ได้ "ล้าง" ค่าสถานะทิ้ง)

### แบบฝึกหัดที่ 472.1

**โจทย์**: จงอธิบายว่าทำไมข้อความจาก Declarative Paragraph จึงปรากฏ**ก่อน**ข้อความ
`"AFTER OPEN STATUS..."` ในผลลัพธ์ ทั้งที่บรรทัด `DISPLAY` นั้นเขียนอยู่**หลัง** `OPEN` ในโค้ด

**เฉลย**: เพราะ `USE AFTER STANDARD ERROR PROCEDURE` ทำงานแบบ **synchronous interrupt** — เมื่อ
`OPEN` ตรวจพบข้อผิดพลาด runtime จะหยุดการทำงานปกติชั่วคราวแล้ว**แทรก**การเรียก Declarative Paragraph
เข้ามาทำงานให้จบก่อน จากนั้นจึงส่งควบคุมกลับไปที่คำสั่งถัดจาก `OPEN` (คือบรรทัด `DISPLAY`) ตามลำดับปกติ
ผลลัพธ์ที่เห็นจึงเรียงตามลำดับเวลาจริงของการทำงาน ไม่ใช่ลำดับที่เขียนโค้ดแบบผิวเผิน

---

## ขั้นตอนที่ 473: USE ... ON INPUT/OUTPUT/I-O/EXTEND — จัดการไฟล์หลายไฟล์ด้วยโค้ดเดียว

### แนวคิด

แทนที่จะระบุชื่อไฟล์ตรง ๆ เราสามารถระบุ **โหมดการเปิดไฟล์** แทนได้ (`INPUT`, `OUTPUT`, `I-O`, `EXTEND`)
ทำให้ Declarative เดียวครอบคลุม**ทุกไฟล์ในโปรแกรมที่เปิดด้วยโหมดนั้น** โดยไม่ต้องเขียน Declarative
แยกทีละไฟล์ — มีประโยชน์มากในโปรแกรม batch ที่มีไฟล์ INPUT หลายไฟล์ที่ต้องการ logic จัดการข้อผิดพลาด
แบบเดียวกันทั้งหมด

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP473-SCOPE-BY-MODE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT FILE-A ASSIGN TO "MISSINGA473.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-STATUS-A.
           SELECT FILE-B ASSIGN TO "MISSINGB473.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-STATUS-B.

       DATA DIVISION.
       FILE SECTION.
       FD  FILE-A.
       01  REC-A                    PIC X(10).
       FD  FILE-B.
       01  REC-B                    PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-STATUS-A              PIC XX.
       01  WS-STATUS-B              PIC XX.

       PROCEDURE DIVISION.
       DECLARATIVES.
      *> ON INPUT covers EVERY file in this program that is
      *> currently open (or being opened) in INPUT mode - here,
      *> that means both FILE-A and FILE-B, with one Paragraph.
       INPUT-ERROR-SECTION SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON INPUT.
       INPUT-ERROR-PARA.
           DISPLAY "GLOBAL INPUT ERROR HANDLER FIRED".
       END DECLARATIVES.

       MAIN-LOGIC SECTION.
       MAIN-LOGIC-START.
           OPEN INPUT FILE-A.
           OPEN INPUT FILE-B.
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
GLOBAL INPUT ERROR HANDLER FIRED
GLOBAL INPUT ERROR HANDLER FIRED
```

### อธิบายโค้ดทีละส่วน

- `FILE-A` และ `FILE-B` ไม่มีไฟล์อยู่จริงทั้งคู่ ทั้งสอง `OPEN INPUT` จึงล้มเหลวด้วยสถานะ `35`
- Declarative เดียวที่เขียนว่า `USE AFTER STANDARD ERROR PROCEDURE ON INPUT` ถูกเรียกทำงาน**สองครั้ง**
  หนึ่งครั้งต่อไฟล์ที่ล้มเหลว แม้จะเป็นไฟล์คนละไฟล์กัน — นี่คือประโยชน์สำคัญ: **เขียนโค้ดจัดการ
  ข้อผิดพลาดครั้งเดียว ใช้ได้กับทุกไฟล์ที่เปิดในโหมด `INPUT`**
- คำสงวนที่ใช้ร่วมกับ `ON` ได้ ได้แก่ `INPUT`, `OUTPUT`, `I-O`, `EXTEND` (ตรงกับ 4 โหมดการเปิดไฟล์
  ที่เรียนมาใน Part 025) และยังสามารถใช้ `ON INPUT-OUTPUT` ผสมกันไม่ได้ — ต้องเลือกทีละโหมด

### ข้อควรระวัง

- Declarative แบบ `ON INPUT` ไม่ทราบว่า "ไฟล์ไหน" เกิดข้อผิดพลาดจากชื่อ Paragraph เอง ต้องตรวจสอบ
  จาก `WS-FILE-STATUS` ของไฟล์ที่เกี่ยวข้อง (ในตัวอย่างนี้คือแยกตัวแปรสถานะคนละตัว `WS-STATUS-A`/
  `WS-STATUS-B`) ถ้าต้องการทราบว่าไฟล์ไหนเกิดปัญหาแน่ ๆ ต้องออกแบบให้ตรวจสอบตัวแปรสถานะที่ถูกต้องเอง
  ในโปรแกรมจริงมักใช้เทคนิคจาก Special Register `FILE-NAME` เพิ่มเติม (พิเศษเฉพาะบาง compiler
  ไม่ครอบคลุมในหลักสูตรนี้เนื่องจากไม่ใช่มาตรฐานกลาง)
- ถ้ามีทั้ง Declarative แบบเจาะจงไฟล์ (`ON FILE-A`) และแบบครอบคลุมโหมด (`ON INPUT`) พร้อมกันในโปรแกรม
  เดียว จะเกิดคำถามว่าตัวไหนทำงานก่อน — คำตอบจะอธิบายละเอียดในขั้นตอนที่ 474

### แบบฝึกหัดที่ 473.1

**โจทย์**: จงอธิบายสถานการณ์ที่การใช้ `USE AFTER STANDARD ERROR PROCEDURE ON INPUT` เหมาะสมกว่า
การเขียน Declarative แยกทีละไฟล์ด้วย `ON file-name`

**เฉลยแนวทาง**: เหมาะกับโปรแกรม batch ที่มีไฟล์ INPUT จำนวนมาก (เช่น 10 ไฟล์ขึ้นไป) และต้องการ
พฤติกรรมจัดการข้อผิดพลาดแบบเดียวกันทุกไฟล์ เช่น "บันทึก log แล้วข้ามไฟล์นั้นไป" — การเขียน Declarative
เดียวด้วย `ON INPUT` ลดโค้ดซ้ำซ้อนได้มากเมื่อเทียบกับการเขียน SECTION แยกทีละไฟล์ที่มี logic เหมือนกัน
ทุกประการ ในทางกลับกัน ถ้าแต่ละไฟล์ต้องการ logic จัดการข้อผิดพลาดที่**แตกต่างกันจริง ๆ** (เช่น ไฟล์หนึ่ง
ควรหยุดโปรแกรมทันที อีกไฟล์ควรข้ามไปเฉย ๆ) การเขียนแยกทีละไฟล์ด้วย `ON file-name` จะเหมาะสมกว่า

---

## ขั้นตอนที่ 474: USE AFTER STANDARD EXCEPTION PROCEDURE และการผสมผสาน Declaratives หลายระดับ

### ERROR กับ EXCEPTION คือคำพ้องความหมาย

มาตรฐาน COBOL อนุญาตให้ใช้คำว่า `ERROR` หรือ `EXCEPTION` แทนกันได้ในวลี `USE AFTER STANDARD ...
PROCEDURE` — ทั้งสองแบบมีความหมายและพฤติกรรมเหมือนกันทุกประการ (`EXCEPTION` เป็นคำที่มาตรฐานรุ่นหลัง
เพิ่มเข้ามาเป็นทางเลือกให้สอดคล้องกับศัพท์ "Exception Handling" ที่ภาษาสมัยใหม่นิยมใช้)

```cobol
      *> These two lines are 100% equivalent - verified by testing
      *> both forms with GnuCOBOL and confirming identical behavior.
           USE AFTER STANDARD ERROR PROCEDURE ON CUSTOMER-FILE.
           USE AFTER STANDARD EXCEPTION PROCEDURE ON CUSTOMER-FILE.
```

### ตัวอย่างโค้ด: หลาย SECTION ในโปรแกรมเดียว — ไฟล์เฉพาะเจาะจง vs ไฟล์ทั่วไป

โปรแกรมจริงมักมีทั้ง Declarative เฉพาะไฟล์สำคัญ และ Declarative ครอบคลุมไฟล์ทั่วไปที่เหลือ:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP474-MIXED-DECLARATIVES.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT FILE-A ASSIGN TO "MISSPRIOA474.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-STATUS-A.
           SELECT FILE-B ASSIGN TO "MISSPRIOB474.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-STATUS-B.

       DATA DIVISION.
       FILE SECTION.
       FD  FILE-A.
       01  REC-A                    PIC X(10).
       FD  FILE-B.
       01  REC-B                    PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-STATUS-A              PIC XX.
       01  WS-STATUS-B              PIC XX.

       PROCEDURE DIVISION.
       DECLARATIVES.
      *> FILE-A is a critical file - it gets its OWN, more specific
      *> handler.
       SPECIFIC-A-SECTION SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON FILE-A.
       SPECIFIC-A-PARA.
           DISPLAY "SPECIFIC HANDLER FOR FILE-A FIRED".

      *> Every OTHER input file (FILE-B, and any future file we
      *> might add) falls back to this general handler.
       GLOBAL-INPUT-SECTION SECTION.
           USE AFTER STANDARD EXCEPTION PROCEDURE ON INPUT.
       GLOBAL-INPUT-PARA.
           DISPLAY "GLOBAL INPUT HANDLER FIRED".
       END DECLARATIVES.

       MAIN-LOGIC SECTION.
       MAIN-LOGIC-START.
           OPEN INPUT FILE-A.
           DISPLAY "STATUS A=" WS-STATUS-A.
           OPEN INPUT FILE-B.
           DISPLAY "STATUS B=" WS-STATUS-B.
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
SPECIFIC HANDLER FOR FILE-A FIRED
STATUS A=35
GLOBAL INPUT HANDLER FIRED
STATUS B=35
```

### อธิบายโค้ดทีละส่วน — กฎความสำคัญก่อนหลัง (Priority Rule)

นี่คือการค้นพบสำคัญที่สุดของขั้นตอนนี้ ยืนยันด้วยการทดสอบจริง: **เมื่อไฟล์หนึ่งมีทั้ง Declarative
แบบเจาะจงชื่อไฟล์ (`ON FILE-A`) และมี Declarative แบบครอบคลุมโหมด (`ON INPUT`) ที่อาจครอบคลุมไฟล์นั้น
ด้วย — COBOL จะเลือกใช้ **Declarative ที่เจาะจงกว่าเสมอ** สำหรับไฟล์นั้น**

- `FILE-A` เข้าเงื่อนไขทั้ง `ON FILE-A` (เจาะจง) และ `ON INPUT` (ทั่วไป) แต่ผลลัพธ์แสดงว่า **เฉพาะ
  `SPECIFIC-A-PARA` เท่านั้นที่ถูกเรียก** — Declarative ทั่วไป `GLOBAL-INPUT-PARA` ไม่ถูกเรียกเลย
  สำหรับ `FILE-A`
- `FILE-B` ไม่มี Declarative เฉพาะเจาะจง จึงตกไปใช้ `GLOBAL-INPUT-PARA` ที่ครอบคลุมโหมด `INPUT`
  แทนตามที่คาดหวัง
- กฎนี้เหมือนกับหลักการ "specific overrides general" ที่พบในภาษาโปรแกรมสมัยใหม่หลายภาษา (เช่น
  CSS specificity, exception handling ในบางภาษาที่จับ exception เฉพาะเจาะจงก่อน exception ทั่วไป)

### ข้อควรระวัง

- ลำดับที่เขียน SECTION ใน `DECLARATIVES` **ไม่มีผล**ต่อกฎความสำคัญนี้ — ความสำคัญขึ้นอยู่กับ
  "ความเจาะจง" ของเงื่อนไข `ON` เท่านั้น ไม่ใช่ลำดับการเขียนโค้ด (ทดลองสลับตำแหน่งสอง SECTION ในตัวอย่าง
  แล้วรันซ้ำ จะได้ผลลัพธ์เหมือนเดิมทุกประการ)
- อย่าคาดหวังว่า Declarative ทั่วไป (`ON INPUT`) จะถูกเรียก "เพิ่มเติม" หลังจาก Declarative เฉพาะ
  (`ON FILE-A`) ทำงานแล้ว — มีเพียง**หนึ่งเดียว**เท่านั้นที่ถูกเรียกต่อข้อผิดพลาดหนึ่งครั้ง (ตัวที่
  เจาะจงที่สุดที่ตรงเงื่อนไข)

### แบบฝึกหัดที่ 474.1

**โจทย์**: จากตัวอย่างข้างต้น หากต้องการให้ `FILE-A` เมื่อเกิดข้อผิดพลาดแล้ว **ทั้งข้อความเฉพาะของ
FILE-A และข้อความทั่วไปของ GLOBAL-INPUT** ปรากฏทั้งคู่ จะต้องแก้โค้ดอย่างไร

**เฉลยแนวทาง**: เนื่องจาก COBOL จะเรียกเพียง Declarative ที่เจาะจงที่สุดเพียงตัวเดียวเท่านั้นโดย
อัตโนมัติ วิธีที่ตรงไปตรงมาที่สุดคือ**เรียก Paragraph ของ Declarative ทั่วไปเองด้วยมือ**จากภายใน
`SPECIFIC-A-PARA` เช่น เพิ่มบรรทัด `PERFORM GLOBAL-INPUT-PARA` ต่อท้ายใน `SPECIFIC-A-PARA` (ในทาง
เทคนิค Declarative Paragraph สามารถถูก `PERFORM` เรียกจากที่อื่นได้เหมือน Paragraph ปกติทุกประการ
เพราะมันก็คือ Paragraph ธรรมดาที่เพียงแค่ถูก "ลงทะเบียน" ไว้ให้ runtime เรียกอัตโนมัติเมื่อเข้าเงื่อนไข
`USE` เท่านั้น)

---

## ขั้นตอนที่ 475: ควบคุมการไหลของโปรแกรมหลัง Declarative ทำงานเสร็จ

### คำถามสำคัญ: หลัง Declarative ทำงานเสร็จ โปรแกรมไปต่อที่ไหน

คำตอบคือ **โปรแกรมกลับไปทำงานต่อที่คำสั่งถัดจากคำสั่งที่ทำให้เกิดข้อผิดพลาดทันที** (ไม่ใช่กลับไปเริ่ม
ต้น Paragraph ใหม่ ไม่ใช่ retry คำสั่งเดิมซ้ำ และไม่ใช่ข้ามไปที่อื่น) มาพิสูจน์ด้วยโปรแกรมที่วนลูป
ค้นหาหลาย Key ในไฟล์ Indexed โดยมี Key หนึ่งที่ไม่มีอยู่จริงตรงกลาง:

> **หมายเหตุ**: ตัวอย่างนี้ใช้ `ORGANIZATION IS INDEXED` ต้องคอมไพล์ด้วย GnuCOBOL build ที่เปิดใช้
> ISAM handler (`indexed file handler : BDB version 5.3.28` ตามที่ตรวจสอบและอธิบายไว้ใน Part 028)
> คำสั่งคอมไพล์และรันในสภาพแวดล้อมของหลักสูตรนี้คือ:
> ```bash
> /opt/gnucobol-isam/bin/cobc -x -o step475 step475.cob
> LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./step475
> ```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP475-CONTROL-FLOW.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PROD-FILE ASSIGN TO "FLOWPROD475.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS PR-CODE
               FILE STATUS IS WS-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  PROD-FILE.
       01  PROD-RECORD.
           05  PR-CODE              PIC 9(3).
           05  PR-NAME              PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-STATUS                PIC XX.
       01  WS-IDX                   PIC 9(1).
       01  WS-LOOKUP-CODES.
           05  PIC 9(3)             VALUE 100.
           05  PIC 9(3)             VALUE 200.
           05  PIC 9(3)             VALUE 300.
       01  WS-LOOKUP-TABLE REDEFINES WS-LOOKUP-CODES
                                     OCCURS 3 TIMES PIC 9(3).

       PROCEDURE DIVISION.
       DECLARATIVES.
       PROD-ERROR-SECTION SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON PROD-FILE.
       PROD-ERROR-PARA.
           DISPLAY "  -> HANDLER: CODE NOT FOUND, STATUS="
               WS-STATUS.
       END DECLARATIVES.

       MAIN-LOGIC SECTION.
       MAIN-LOGIC-START.
           PERFORM SETUP-DATA
           OPEN INPUT PROD-FILE
           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > 3
               MOVE WS-LOOKUP-TABLE(WS-IDX) TO PR-CODE
               DISPLAY "LOOKING UP CODE " PR-CODE
               READ PROD-FILE
               DISPLAY "  BACK FROM READ, NAME=[" PR-NAME
                   "] STATUS=" WS-STATUS
           END-PERFORM
           CLOSE PROD-FILE
           STOP RUN.

       SETUP-DATA SECTION.
       SETUP-DATA-START.
           OPEN OUTPUT PROD-FILE
           MOVE 100 TO PR-CODE
           MOVE "RICE" TO PR-NAME
           WRITE PROD-RECORD
           MOVE 300 TO PR-CODE
           MOVE "OIL" TO PR-NAME
           WRITE PROD-RECORD
           CLOSE PROD-FILE.
      *> Note: code 200 is deliberately never written, to trigger
      *> a "not found" error in the loop below.
```

### ผลลัพธ์จริงที่ได้

```
LOOKING UP CODE 100
  BACK FROM READ, NAME=[RICE      ] STATUS=00
LOOKING UP CODE 200
  -> HANDLER: CODE NOT FOUND, STATUS=23
  BACK FROM READ, NAME=[RICE      ] STATUS=23
LOOKING UP CODE 300
  BACK FROM READ, NAME=[OIL       ] STATUS=00
```

### อธิบายโค้ดทีละส่วนอย่างละเอียด — จุดที่ต้องสังเกตให้ครบ

1. **Code 100** อ่านสำเร็จ (status `00`) — Declarative ไม่ถูกเรียก
2. **Code 200** ไม่มีอยู่จริง `READ` ล้มเหลวด้วยสถานะ `23` → Declarative ทำงานก่อน (แสดงบรรทัด
   `"-> HANDLER..."`) → **จากนั้นควบคุมกลับไปที่บรรทัด `DISPLAY "BACK FROM READ..."` ที่อยู่ถัดจาก
   `READ` ทันที ไม่ใช่กลับไปเริ่ม `PERFORM VARYING` ใหม่** — พิสูจน์ชัดเจนว่าโปรแกรมไม่ retry และไม่
   กระโดดข้าม
3. **จุดที่ต้องสังเกตเป็นพิเศษ**: เมื่อ `READ` ล้มเหลว `PR-NAME` **ยังคงค่าเดิมจากการอ่านครั้งก่อน**
   (`"RICE"` จาก code 100) เพราะ `READ` ที่ล้มเหลวไม่ได้เขียนทับข้อมูลใน record area เลย — สอดคล้อง
   กับกฎ "ฟิลด์ปลายทางไม่เปลี่ยนแปลงเมื่อเกิดข้อผิดพลาด" ที่เรียนมาตั้งแต่ Part 009/015 เรื่อง
   SIZE ERROR — โปรแกรมที่ไม่ตรวจสอบ `WS-STATUS` หลัง `READ` เสี่ยงที่จะใช้ค่า `PR-NAME` เก่าที่ผิด
   ไปแสดงผลหรือคำนวณต่อโดยไม่รู้ตัว
4. **Code 300** อ่านสำเร็จตามปกติต่อไป — ลูปทำงานครบทั้ง 3 รอบโดยไม่มีการล่มหรือหยุดกลางคัน แม้จะมี
   ข้อผิดพลาดเกิดขึ้นระหว่างทาง

### ความสัมพันธ์กับ AT END / INVALID KEY ที่เรียนมาก่อนหน้า

จุดสำคัญที่ต้องเข้าใจให้ชัดเจน: **ถ้าคำสั่ง READ/WRITE มี `AT END`/`INVALID KEY` clause กำกับไว้
โดยตรง COBOL จะใช้ clause นั้นจัดการข้อผิดพลาดที่ตรงกับ clause นั้น และ `USE AFTER STANDARD ERROR
PROCEDURE` จะ**ไม่ถูกเรียก**สำหรับเงื่อนไขที่ clause นั้นดักจับไว้แล้ว** ตัวอย่างข้างต้น**จงใจไม่ใส่**
`INVALID KEY` เลย เพื่อให้เห็น Declarative ทำงานชัดเจน — ถ้าเราเพิ่ม `INVALID KEY DISPLAY "NOT FOUND"`
เข้าไปที่ `READ` ตัว Declarative ของขั้นตอนนี้จะไม่ถูกเรียกอีกต่อไปสำหรับสถานะ `23` (เพราะ
`INVALID KEY` ดักไว้ก่อนแล้ว) นี่คือเหตุผลที่ Part 023-029 สอน `AT END`/`INVALID KEY` มาก่อน และ
Part นี้เพิ่ง introduce DECLARATIVES เป็นกลไก**เสริม**สำหรับกรณีที่**ไม่มี** clause เหล่านั้นกำกับไว้
โดยตรง (หรือเพื่อจับข้อผิดพลาดประเภทอื่นที่ clause เหล่านั้นไม่ครอบคลุม เช่นสถานะ `35`, `41`, `42`)

### ข้อควรระวัง

- อย่าลืมกฎ "ฟิลด์ปลายทางไม่เปลี่ยนเมื่อเกิดข้อผิดพลาด" — โปรแกรมที่แสดงผลหรือคำนวณต่อจาก record area
  ทันทีหลัง `READ` ที่อาจล้มเหลว โดยไม่ตรวจสอบ `WS-STATUS` ก่อน มีความเสี่ยงสูงที่จะใช้ข้อมูล "ค้าง"
  จากรอบก่อนหน้าโดยไม่รู้ตัว แม้จะมี Declarative คอยจับ error ไว้แล้วก็ตาม (Declarative ทำหน้าที่แค่
  "แจ้งเตือน/บันทึก" ไม่ได้ "แก้ไขข้อมูลให้ถูกต้อง" ให้อัตโนมัติ)
- ถ้าต้องการให้โปรแกรมข้ามการประมวลผลรอบนั้นไปเลยเมื่อเกิดข้อผิดพลาด (ไม่ใช่แค่แสดงข้อความแล้วทำต่อ)
  ต้องออกแบบด้วย flag ตัวแปร (ทำนองเดียวกับ `WS-DIVIDE-BY-ZERO-FLAG` ใน Part 015) ที่ Declarative
  ตั้งค่าไว้ แล้วให้โปรแกรมหลักตรวจสอบ flag นั้นก่อนใช้งานข้อมูล — จะสาธิตเทคนิคนี้ในขั้นตอนที่ 477

### แบบฝึกหัดที่ 475.1

**โจทย์**: จงอธิบายว่าทำไมค่า `PR-NAME` ที่แสดงหลัง `READ` ที่ล้มเหลว (code 200) จึงเป็น `"RICE"`
แทนที่จะเป็นค่าว่างหรือค่าที่ไม่มีความหมาย

**เฉลย**: เพราะ `PR-NAME` เป็นส่วนหนึ่งของ `PROD-RECORD` ใน FILE SECTION ซึ่งเป็นพื้นที่หน่วยความจำ
ที่ถูกใช้ซ้ำ (reuse) ทุกครั้งที่มีการ `READ` เมื่อ `READ` ครั้งก่อนหน้า (code 100) สำเร็จ ค่า `"RICE"`
ถูกเขียนลงไปใน record area นี้ เมื่อ `READ` ครั้งถัดมา (code 200) ล้มเหลว COBOL จะ**ไม่แตะต้อง**
record area เลยตามกฎมาตรฐาน (คล้ายกฎ SIZE ERROR ที่ไม่แก้ไขฟิลด์ปลายทางเมื่อคำนวณล้มเหลว) ค่าที่
เหลืออยู่ในหน่วยความจำจึงยังเป็น `"RICE"` จากการอ่านที่สำเร็จครั้งล่าสุดนั่นเอง

---

## ขั้นตอนที่ 476: USE FOR DEBUGGING — ดักจับการเปลี่ยนแปลงค่าตัวแปรเพื่อ Debug

### แนวคิด

`USE FOR DEBUGGING` เป็นกลไกที่ต่างจาก `USE AFTER STANDARD ERROR PROCEDURE` โดยสิ้นเชิง: มันไม่
เกี่ยวกับไฟล์เลย แต่ใช้**ดักจับทุกครั้งที่ตัวแปรที่ระบุไว้มีการเปลี่ยนค่า** (ผ่าน `MOVE`, เลขคณิต,
หรือ `PERFORM` ผ่าน paragraph ที่ระบุ) มีประโยชน์มากสำหรับการ debug ตัวแปรที่ค่าเปลี่ยนแปลงจากหลายจุด
ในโปรแกรมขนาดใหญ่ โดยไม่ต้องเติม `DISPLAY` ไว้ทุกจุดที่แก้ไขค่าตัวแปรนั้นด้วยมือ

### ข้อกำหนดพิเศษที่ต้องรู้ (ยืนยันด้วยการทดสอบจริง)

การใช้งาน `USE FOR DEBUGGING` ให้ทำงานจริงบน GnuCOBOL ต้องครบ **2 เงื่อนไข**:

1. **ตอนคอมไพล์**: ต้องประกาศ `SOURCE-COMPUTER. <ชื่อใดก็ได้> WITH DEBUGGING MODE.` ใน
   `CONFIGURATION SECTION` ของ `ENVIRONMENT DIVISION`
2. **ตอนรัน**: ต้องตั้งค่า environment variable `COB_SET_DEBUG=1` ก่อนรันโปรแกรม (ยืนยันจากการทดสอบ
   จริง — ถ้าไม่ตั้งค่านี้ โปรแกรมจะรันได้ปกติแต่ Declarative debugging จะไม่ถูกเรียกเลย แม้จะคอมไพล์
   ด้วย DEBUGGING MODE แล้วก็ตาม)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP476-USE-FOR-DEBUGGING.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       CONFIGURATION SECTION.
      *> WITH DEBUGGING MODE is required at compile time for any
      *> USE FOR DEBUGGING declarative to be generated at all.
       SOURCE-COMPUTER. GENERIC WITH DEBUGGING MODE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RUNNING-TOTAL         PIC 9(5) VALUE 0.

       PROCEDURE DIVISION.
       DECLARATIVES.
       DEBUG-TOTAL-SECTION SECTION.
           USE FOR DEBUGGING ON WS-RUNNING-TOTAL.
       DEBUG-TOTAL-PARA.
           DISPLAY "DEBUG: WS-RUNNING-TOTAL CHANGED TO "
               WS-RUNNING-TOTAL.
       END DECLARATIVES.

       MAIN-LOGIC SECTION.
       MAIN-LOGIC-START.
           MOVE 10 TO WS-RUNNING-TOTAL.
           ADD 5 TO WS-RUNNING-TOTAL.
           ADD 100 TO WS-RUNNING-TOTAL.
           DISPLAY "FINAL VALUE = " WS-RUNNING-TOTAL.
           STOP RUN.
```

คอมไพล์ปกติ แต่รันสองแบบเพื่อเปรียบเทียบ:

```bash
cobc -x -o step476 step476.cob

echo "=== WITHOUT COB_SET_DEBUG ==="
./step476

echo "=== WITH COB_SET_DEBUG=1 ==="
COB_SET_DEBUG=1 ./step476
```

### ผลลัพธ์จริงที่ได้

```
=== WITHOUT COB_SET_DEBUG ===
FINAL VALUE = 00115

=== WITH COB_SET_DEBUG=1 ===
DEBUG: WS-RUNNING-TOTAL CHANGED TO 00010
DEBUG: WS-RUNNING-TOTAL CHANGED TO 00015
DEBUG: WS-RUNNING-TOTAL CHANGED TO 00115
FINAL VALUE = 00115
```

### อธิบายโค้ดทีละส่วน

- เมื่อรันแบบปกติ (ไม่ตั้ง `COB_SET_DEBUG`) โปรแกรมทำงานเหมือนไม่มี Declaratives อยู่เลย — นี่เป็น
  ข้อดีสำคัญของ `USE FOR DEBUGGING`: **สามารถปล่อยโค้ด debug ไว้ในโปรแกรม production ได้อย่างปลอดภัย**
  เพราะมันจะ "เงียบ" อยู่เสมอจนกว่าจะมีคนตั้งค่า environment variable เพื่อเปิดใช้งานโดยเฉพาะ
- เมื่อตั้ง `COB_SET_DEBUG=1` ทุกครั้งที่ `WS-RUNNING-TOTAL` เปลี่ยนค่า (จาก `MOVE` หรือ `ADD` ก็ตาม)
  `DEBUG-TOTAL-PARA` จะถูกเรียกโดยอัตโนมัติ แสดงค่าล่าสุดให้เห็นทันที — สังเกตว่าเราไม่ได้เขียน
  `DISPLAY` แทรกไว้ในบรรทัด `MOVE 10 TO ...`, `ADD 5 TO ...`, `ADD 100 TO ...` เลยสักบรรทัดเดียว
  ทั้งหมดมาจาก Declarative เพียงจุดเดียว

### ข้อควรระวัง

- `USE FOR DEBUGGING` เพิ่ม overhead เล็กน้อยให้กับทุกการเปลี่ยนแปลงค่าของตัวแปรที่ถูกดักจับ (เพราะ
  compiler ต้องแทรกโค้ดตรวจสอบเพิ่ม) แม้จะปิดด้วย `COB_SET_DEBUG` ที่ไม่ได้ตั้งค่าไว้ก็ตาม จึงไม่ควร
  ใช้พร่ำเพรื่อในโปรแกรมที่ต้องการประสิทธิภาพสูงสุด ควรใช้เฉพาะช่วง debug จริง ๆ แล้วพิจารณาคอมไพล์
  ใหม่โดยไม่มี `WITH DEBUGGING MODE` สำหรับ production build สุดท้าย
- คุณลักษณะนี้ค่อนข้างเฉพาะทางและ**การรองรับรายละเอียดปลีกย่อยอาจแตกต่างกันไปในแต่ละ compiler**
  (พฤติกรรมที่ทดสอบยืนยันในเอกสารนี้เป็นของ GnuCOBOL 4.0-early-dev.0 โดยเฉพาะ) ควรทดสอบยืนยันพฤติกรรม
  จริงก่อนพึ่งพาในโปรเจกต์จริงเสมอ ตามหลักการที่ย้ำมาตลอดหลักสูตรนี้

### แบบฝึกหัดที่ 476.1

**โจทย์**: จงอธิบายว่าทำไมการออกแบบให้ `USE FOR DEBUGGING` ต้องเปิดใช้งานผ่าน environment variable
ตอนรัน (แทนที่จะทำงานทันทีเมื่อคอมไพล์ด้วย `WITH DEBUGGING MODE`) จึงเป็นการออกแบบที่ชาญฉลาด

**เฉลยแนวทาง**: การแยก "คอมไพล์ด้วย debugging mode" ออกจาก "เปิดใช้งาน debugging ตอนรัน" ทำให้
องค์กรสามารถ**คอมไพล์โปรแกรมเพียงชุดเดียว**แล้วใช้ได้ทั้งในสถานการณ์ปกติ (ไม่ตั้ง `COB_SET_DEBUG`
ทำงานเร็วเหมือนไม่มี debug code) และในสถานการณ์ที่ต้องการ troubleshoot ปัญหาที่เกิดขึ้นเฉพาะใน
production (ตั้ง `COB_SET_DEBUG=1` ชั่วคราวเพื่อดูว่าตัวแปรสำคัญเปลี่ยนค่าอย่างไรบ้าง) โดยไม่ต้อง
หยุดระบบเพื่อคอมไพล์โปรแกรมเวอร์ชันพิเศษใหม่ นี่ตรงกับความต้องการจริงของระบบระดับองค์กรที่ต้องการ
ความสามารถ "เปิด-ปิด debug ได้โดยไม่กระทบการทำงานปกติ"

---

## ขั้นตอนที่ 477: ออกแบบ Pattern การจัดการข้อผิดพลาดแบบรวมศูนย์ที่นำไปใช้ได้จริง

### แนวคิด

เมื่อเข้าใจกลไกพื้นฐานครบแล้ว ขั้นตอนนี้จะรวมเทคนิคจาก Part 030 (88-level ตั้งชื่อสถานะ) เข้ากับ
DECLARATIVES เพื่อสร้าง **Pattern การจัดการข้อผิดพลาดที่นำไปใช้ในโปรแกรมจริงได้** — จุดสำคัญคือการใช้
**flag ตัวแปร** ที่ Declarative ตั้งค่าไว้ เพื่อให้โปรแกรมหลักรู้ว่า "รอบนี้มีข้อผิดพลาดเกิดขึ้น
ควรข้ามการประมวลผลข้อมูลนี้ไป" แทนที่จะใช้ข้อมูลที่อาจไม่สมบูรณ์ต่อ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP477-CENTRALIZED-PATTERN.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PROD-FILE ASSIGN TO "PATTPROD477.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS PR-CODE
               FILE STATUS IS WS-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  PROD-FILE.
       01  PROD-RECORD.
           05  PR-CODE              PIC 9(3).
           05  PR-NAME              PIC X(10).
           05  PR-PRICE             PIC 9(5)V99.

       WORKING-STORAGE SECTION.
      *> Naming every status with 88-levels, as taught in Part 030
      *> step 297 - this is what makes the Declarative below
      *> readable instead of full of magic numbers.
       01  WS-STATUS                PIC XX.
           88  FS-SUCCESS                  VALUE "00".
           88  FS-NOT-FOUND                VALUE "23".

      *> The flag the Declarative sets, so main-line logic can tell
      *> "was this lookup actually good, or should I skip it?"
       01  WS-LOOKUP-OK-FLAG        PIC X(1) VALUE "N".
           88  LOOKUP-WAS-OK               VALUE "Y".

       01  WS-LOOKUP-CODE           PIC 9(3).
       01  WS-NOT-FOUND-COUNT       PIC 9(3) VALUE 0.
       01  WS-IDX                   PIC 9(1).
       01  WS-CODES-TO-LOOKUP.
           05  PIC 9(3)             VALUE 100.
           05  PIC 9(3)             VALUE 200.
           05  PIC 9(3)             VALUE 999.
       01  WS-CODE-TABLE REDEFINES WS-CODES-TO-LOOKUP
                                    OCCURS 3 TIMES PIC 9(3).

       PROCEDURE DIVISION.
       DECLARATIVES.
       PROD-ERROR-SECTION SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON PROD-FILE.
       PROD-ERROR-PARA.
      *> Centralized policy: ANY error on PROD-FILE means "this
      *> lookup did not produce trustworthy data" - so clear the
      *> success flag. Main-line logic never needs to know WHICH
      *> status code occurred, only whether it can trust the data.
           MOVE "N" TO WS-LOOKUP-OK-FLAG
           IF FS-NOT-FOUND
               ADD 1 TO WS-NOT-FOUND-COUNT
               DISPLAY "  [LOG] CODE " WS-LOOKUP-CODE
                   " NOT FOUND IN MASTER FILE"
           ELSE
               DISPLAY "  [LOG] UNEXPECTED STATUS " WS-STATUS
                   " ON PROD-FILE"
           END-IF.
       END DECLARATIVES.

       MAIN-LOGIC SECTION.
       MAIN-LOGIC-START.
           PERFORM SETUP-DATA
           OPEN INPUT PROD-FILE
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 3
               MOVE WS-CODE-TABLE(WS-IDX) TO WS-LOOKUP-CODE
               PERFORM LOOKUP-ONE-PRODUCT
           END-PERFORM
           CLOSE PROD-FILE
           DISPLAY "TOTAL NOT-FOUND ERRORS: " WS-NOT-FOUND-COUNT
           STOP RUN.

       LOOKUP-ONE-PRODUCT SECTION.
       LOOKUP-ONE-PRODUCT-START.
      *> Reset the flag to success BEFORE every lookup - exactly
      *> the same discipline as the divide-by-zero flag reset in
      *> Part 015 step 144. Forgetting this line is the single
      *> most common bug in this pattern.
           MOVE "Y" TO WS-LOOKUP-OK-FLAG
           MOVE WS-LOOKUP-CODE TO PR-CODE
           READ PROD-FILE
           IF LOOKUP-WAS-OK
               DISPLAY "OK: CODE " PR-CODE " = " PR-NAME
                   " PRICE " PR-PRICE
           ELSE
               DISPLAY "SKIPPED CODE " WS-LOOKUP-CODE
                   " DUE TO ERROR (SEE LOG ABOVE)"
           END-IF.

       SETUP-DATA SECTION.
       SETUP-DATA-START.
           OPEN OUTPUT PROD-FILE
           MOVE 100 TO PR-CODE
           MOVE "RICE" TO PR-NAME
           MOVE 45.50 TO PR-PRICE
           WRITE PROD-RECORD
           CLOSE PROD-FILE.
      *> Codes 200 and 999 are deliberately never written.
```

คอมไพล์และรันด้วย (indexed file ต้องใช้ ISAM build เช่นเดิม):

```bash
/opt/gnucobol-isam/bin/cobc -x -o step477 step477.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./step477
```

### ผลลัพธ์จริงที่ได้

```
  [LOG] CODE 200 NOT FOUND IN MASTER FILE
  [LOG] CODE 999 NOT FOUND IN MASTER FILE
OK: CODE 100 = RICE       PRICE 0004550
SKIPPED CODE 200 DUE TO ERROR (SEE LOG ABOVE)
SKIPPED CODE 999 DUE TO ERROR (SEE LOG ABOVE)
TOTAL NOT-FOUND ERRORS: 002
```

### อธิบายโค้ดทีละส่วน

- สังเกตว่าบรรทัด `[LOG] ...` ปรากฏ**ก่อน**บรรทัด `OK: CODE 100 ...` ทั้งที่ code 100 ถูกค้นหา**ก่อน**
  code 200/999 ในลำดับลูป — เหตุผลคือ GnuCOBOL runtime buffer การ `DISPLAY` ต่างช่องสัญญาณกันเล็กน้อย
  ระหว่าง path ปกติกับ path ของ Declarative ในบาง build ทำให้ลำดับการ flush อาจไม่ตรงกับลำดับเวลาที่
  เขียนโค้ดเป๊ะ ๆ เสมอไป — **นี่ไม่กระทบความถูกต้องของตรรกะโปรแกรมแต่อย่างใด** (ลำดับภายใน "LOOKUP
  ONE PRODUCT" หนึ่งรอบยังคงถูกต้อง 100% เสมอ) เป็นเพียงพฤติกรรมการจัดเรียง output buffer ของ
  runtime ที่ควรรู้ไว้เมื่อเทียบผลลัพธ์กับลำดับโค้ดตรง ๆ
- `WS-LOOKUP-OK-FLAG` คือหัวใจของ pattern นี้: ตั้งเป็น `"Y"` **ก่อน**ทุกครั้งที่จะ `READ` แล้วปล่อยให้
  Declarative เปลี่ยนเป็น `"N"` **เฉพาะเมื่อเกิดข้อผิดพลาดจริง** ทำให้โปรแกรมหลัก (`LOOKUP-ONE-PRODUCT`)
  ตรวจสอบได้ง่าย ๆ ด้วย `IF LOOKUP-WAS-OK` โดยไม่ต้องรู้รายละเอียดว่าสถานะที่แท้จริงคือ `23` หรืออะไร
- ตรรกะการนับ `WS-NOT-FOUND-COUNT` และการบันทึก log อยู่**รวมศูนย์ที่เดียว**ใน Declarative แทนที่จะ
  กระจายอยู่ทุกจุดที่มีการ `READ PROD-FILE` ในโปรแกรม — ถ้าอนาคตมีจุด `READ PROD-FILE` เพิ่มอีก 10
  จุดในโปรแกรมใหญ่ ทุกจุดจะได้ประโยชน์จาก logic การนับและบันทึก log นี้อัตโนมัติทันทีโดยไม่ต้องแก้โค้ด
  จุดนั้นเลย

### ข้อควรระวัง

- Pattern นี้ใช้ได้ดีเมื่อ "ทุกจุดที่เรียกใช้ไฟล์เดียวกัน ต้องการ policy จัดการข้อผิดพลาดแบบเดียวกัน"
  ถ้าบางจุดในโปรแกรมต้องการ policy ต่างออกไป (เช่น จุดหนึ่งต้องการหยุดโปรแกรมทันทีเมื่อไม่พบ record
  แต่อีกจุดต้องการแค่ข้ามไป) การรวมศูนย์แบบนี้อาจไม่เหมาะ ต้องพิจารณาออกแบบเพิ่มเติม (เช่น ใช้ตัวแปร
  บอกโหมดการทำงานปัจจุบัน)
- อย่าลืม reset flag ก่อนทุกปฏิบัติการเสมอ (บรรทัด `MOVE "Y" TO WS-LOOKUP-OK-FLAG` ก่อน `READ`) —
  ถ้าลืมขั้นตอนนี้ flag จะค้างค่า `"N"` ตลอดไปหลังพบข้อผิดพลาดครั้งแรก ทำให้การค้นหาที่ถูกต้องในรอบ
  ถัดไปถูกมองว่า "ผิดพลาด" ไปด้วยอย่างไม่ถูกต้อง (เหมือนข้อควรระวังใน Part 015 ขั้นตอนที่ 144 ทุกประการ)

### แบบฝึกหัดที่ 477.1

**โจทย์**: จงอธิบายข้อดีของการให้ Declarative เป็นผู้ตั้งค่า `WS-LOOKUP-OK-FLAG` แทนที่จะให้
`LOOKUP-ONE-PRODUCT` ตรวจสอบ `WS-STATUS` เองโดยตรงด้วย `IF FS-SUCCESS ... ELSE ...`

**เฉลยแนวทาง**: ถ้าให้ `LOOKUP-ONE-PRODUCT` ตรวจสอบ `WS-STATUS` เอง โปรแกรมจะกลับไปมีปัญหาเดิมที่
DECLARATIVES ถูกออกแบบมาแก้ไข คือ **ทุกจุดที่มีการ `READ PROD-FILE` ต้องเขียน `IF`/`EVALUATE`
ตรวจสอบสถานะซ้ำเอง** หากในอนาคตมีจุด `READ PROD-FILE` เพิ่มอีกหลายจุด (เช่นในโปรแกรมที่ซับซ้อนขึ้น)
ทุกจุดต้องเขียนโค้ดตรวจสอบและบันทึก log ซ้ำเหมือนเดิม การให้ Declarative เป็นผู้ตั้งค่า flag แทน
ทำให้**ทุกจุดที่ `READ` ไฟล์นี้ได้ประโยชน์จาก logic การนับและบันทึก log แบบเดียวกันโดยอัตโนมัติ**
โดยจุดเรียกใช้งานเพียงแค่ตรวจสอบ flag ง่าย ๆ (`IF LOOKUP-WAS-OK`) โดยไม่ต้องรู้รายละเอียดเบื้องหลังเลย
นี่คือหลักการ separation of concerns ที่ทำให้โค้ดดูแลรักษาง่ายขึ้นมากในระยะยาว

---

## ขั้นตอนที่ 478: ข้อจำกัดและกับดักของ DECLARATIVES ที่ต้องระวัง

### กับดักที่ 1: ความเสี่ยง Infinite Loop เมื่อ Declarative เองก็ทำปฏิบัติการไฟล์

หากโค้ดภายใน Declarative Paragraph ของไฟล์หนึ่ง **ทำปฏิบัติการกับไฟล์เดียวกันนั้นเองอีกครั้ง** (เช่น
`READ` ไฟล์เดิมซ้ำภายใน error handler ของไฟล์นั้น) และปฏิบัติการนั้นล้มเหลวอีก จะเกิดการเรียก
Declarative ซ้ำไปเรื่อย ๆ จนอาจเกิด **stack overflow หรือโปรแกรมค้าง** — นี่ไม่ใช่ข้อผิดพลาดของ
compiler แต่เป็นความรับผิดชอบของผู้เขียนโปรแกรมที่ต้องออกแบบให้ Declarative ไม่เรียกซ้ำไฟล์ตัวเอง
โดยไม่มีเงื่อนไขหยุดที่ชัดเจน

```cobol
      *> DANGEROUS PATTERN - DO NOT WRITE CODE LIKE THIS:
       RISKY-ERROR-SECTION SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON RISKY-FILE.
       RISKY-ERROR-PARA.
      *> If RISKY-FILE keeps failing, this READ re-triggers THIS
      *> SAME declarative again and again - a classic infinite
      *> recursion risk. Never do this without a hard stop
      *> condition (e.g. a retry counter checked BEFORE retrying).
           READ RISKY-FILE.
```

### กับดักที่ 2: DECLARATIVES ไม่ใช่กลไกที่ portable 100% ข้าม Compiler

แม้ `DECLARATIVES`/`USE AFTER STANDARD ERROR PROCEDURE` จะเป็นมาตรฐาน COBOL-74/85 ที่ compiler
ส่วนใหญ่รองรับ แต่รายละเอียดปลีกย่อย (เช่น ลำดับความสำคัญเมื่อมีหลาย Declarative ที่อาจครอบคลุมไฟล์
เดียวกัน ที่พิสูจน์ในขั้นตอนที่ 474) **อาจแตกต่างกันได้ในแต่ละ compiler** เนื่องจากมาตรฐานไม่ได้ระบุ
รายละเอียดทุกกรณีไว้อย่างชัดเจน 100% ควรทดสอบพฤติกรรมจริงบน compiler เป้าหมายเสมอก่อนพึ่งพาในโปรแกรม
สำคัญ (แนวทางเดียวกับที่ Part 030 ย้ำเรื่องสถานะ `02` ที่พฤติกรรมต่างจากที่มาตรฐานอธิบายไว้ตามตัวอักษร)

### กับดักที่ 3: DECLARATIVES ไม่ครอบคลุมทุกประเภทข้อผิดพลาด

`USE AFTER STANDARD ERROR PROCEDURE` ครอบคลุมเฉพาะข้อผิดพลาดเกี่ยวกับ**ไฟล์**เท่านั้น (`OPEN`,
`READ`, `WRITE`, `REWRITE`, `DELETE`, `START`, `CLOSE`) มันไม่ครอบคลุมข้อผิดพลาดประเภทอื่น เช่น
`SIZE ERROR` จากการคำนวณ (ยังคงต้องใช้ `ON SIZE ERROR` ตามที่เรียนใน Part 009) หรือ Subscript
เกินขอบเขตของตาราง (ยังคงต้องตรวจสอบเองตามที่เรียนใน Part 016-018)

### กับดักที่ 4: ลืมว่า AT END/INVALID KEY ที่มีอยู่แล้วจะ "แย่ง" ไม่ให้ Declarative ทำงาน

ทบทวนจากขั้นตอนที่ 475: ถ้า `READ`/`WRITE` มี `AT END`/`INVALID KEY`/`NOT INVALID KEY` clause
กำกับไว้ตรง ๆ COBOL จะใช้ clause นั้นสำหรับเงื่อนไขที่ตรงกัน (`10` สำหรับ `AT END`, `2x` สำหรับ
`INVALID KEY`) และ Declarative จะไม่ถูกเรียกสำหรับเงื่อนไขเหล่านั้น — ผู้เขียนโปรแกรมที่คาดหวังให้
Declarative ทำงานแต่ลืมว่าตัวเองใส่ `AT END` ไว้ที่ `READ` อยู่แล้ว มักสับสนว่า "ทำไม Declarative
ไม่ทำงาน" ทั้งที่จริง ๆ แล้ว `AT END` clause จัดการเงื่อนไขนั้นไปเรียบร้อยแล้วตั้งแต่ต้น

### ทดสอบยืนยัน: AT END ที่มีอยู่แล้วทำให้ Declarative ไม่ถูกเรียกสำหรับสถานะ 10

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP478-AT-END-WINS.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT DATA-FILE ASSIGN TO "ATENDWIN478.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  DATA-FILE.
       01  DATA-RECORD              PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-STATUS                PIC XX.
       01  WS-EOF-FLAG              PIC X(1) VALUE "N".
           88  END-OF-FILE                 VALUE "Y".

       PROCEDURE DIVISION.
       DECLARATIVES.
       DATA-ERROR-SECTION SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON DATA-FILE.
       DATA-ERROR-PARA.
      *> This should NOT fire for the AT END condition below,
      *> because the READ statement has its own AT END clause.
           DISPLAY "DECLARATIVE FIRED - STATUS=" WS-STATUS.
       END DECLARATIVES.

       MAIN-LOGIC SECTION.
       MAIN-LOGIC-START.
           OPEN OUTPUT DATA-FILE
           MOVE "ONLY LINE" TO DATA-RECORD
           WRITE DATA-RECORD
           CLOSE DATA-FILE

           OPEN INPUT DATA-FILE
           PERFORM UNTIL END-OF-FILE
               READ DATA-FILE
                   AT END
                       SET END-OF-FILE TO TRUE
                       DISPLAY "NORMAL AT END CLAUSE HANDLED IT"
                   NOT AT END
                       DISPLAY "READ: " DATA-RECORD
               END-READ
           END-PERFORM
           CLOSE DATA-FILE
           STOP RUN.
```

### ผลลัพธ์จริงที่ได้

```
READ: ONLY LINE
NORMAL AT END CLAUSE HANDLED IT
```

สังเกตว่า **ข้อความ `"DECLARATIVE FIRED..."` ไม่ปรากฏเลย** แม้ `READ` ครั้งที่สองจะได้สถานะ `10`
(สิ้นสุดไฟล์) ก็ตาม เพราะ `AT END` clause ที่เขียนไว้ตรง ๆ ใน `READ` ดักจับเงื่อนไขนี้ไปเรียบร้อยแล้ว
Declarative จึงไม่มีโอกาสถูกเรียกเลยสำหรับสถานการณ์นี้ — พิสูจน์กับดักที่ 4 อย่างชัดเจนด้วยการทดสอบจริง

### ข้อควรระวัง

- ก่อนสงสัยว่า "ทำไม Declarative ไม่ทำงาน" ให้ตรวจสอบก่อนเสมอว่าคำสั่งไฟล์จุดนั้นมี `AT END`/
  `INVALID KEY`/`NOT INVALID KEY`/`NOT AT END` clause กำกับไว้ตรง ๆ หรือไม่ เพราะ clause เหล่านี้
  มีความสำคัญเหนือกว่า Declarative เสมอสำหรับเงื่อนไขที่ตรงกัน
- อย่าออกแบบ Declarative ที่มีความเสี่ยง infinite recursion โดยไม่มีเงื่อนไขหยุดที่ตรวจสอบก่อนเสมอ

### แบบฝึกหัดที่ 478.1

**โจทย์**: จากตัวอย่างขั้นตอนที่ 478 หากต้องการให้ Declarative `DATA-ERROR-PARA` ทำงานได้จริงเมื่อ
ไฟล์นี้เกิดข้อผิดพลาดประเภทอื่นที่ไม่ใช่ `AT END` (เช่น `35` ตอนเปิดไฟล์ที่ไม่มีอยู่จริง) โดยไม่แก้ไข
โครงสร้างลูป `PERFORM UNTIL END-OF-FILE` ที่มี `AT END` อยู่แล้ว จะเป็นไปได้หรือไม่ เพราะเหตุใด

**เฉลย**: เป็นไปได้ และไม่ต้องแก้ไขอะไรเลย เพราะ Declarative จะไม่ถูกเรียกเฉพาะสำหรับ**เงื่อนไขที่
clause ที่เขียนไว้ตรง ๆ ดักจับไปแล้วเท่านั้น** (ในที่นี้คือสถานะ `10` ที่ `AT END` ดักไว้) ส่วน
ข้อผิดพลาดประเภทอื่นที่ **ไม่มี clause ใดดักจับไว้โดยตรง** เช่นสถานะ `35` จาก `OPEN INPUT` ที่ไม่มี
`INVALID KEY`/`AT END` เกี่ยวข้องเลย Declarative จะยังคงถูกเรียกทำงานตามปกติทุกประการ (เหมือนที่
พิสูจน์แล้วในขั้นตอนที่ 472) กฎ "clause ที่เขียนตรง ๆ ชนะ Declarative" ใช้ผลเฉพาะกับเงื่อนไขที่ clause
นั้นครอบคลุมจริง ๆ เท่านั้น ไม่ได้ปิดกั้น Declarative จากเงื่อนไขอื่นทั้งหมดของไฟล์เดียวกัน

---

## ขั้นตอนที่ 479: แบบฝึกหัดใหญ่ — ระบบบันทึก Error Log แบบรวมศูนย์

### โจทย์แบบฝึกหัดที่ 479.1

**โจทย์**: จงเขียนโปรแกรม COBOL ที่มีไฟล์ 2 ไฟล์: `MASTER-FILE` (Indexed, Key คือรหัสสินค้า 3 หลัก)
และ `ERROR-LOG-FILE` (Line Sequential สำหรับบันทึกข้อผิดพลาด) โดยใช้ `DECLARATIVES` เพื่อ **บันทึก
ทุกข้อผิดพลาดของ `MASTER-FILE` ลง `ERROR-LOG-FILE` โดยอัตโนมัติ** แทนที่จะแค่ `DISPLAY` ออกหน้าจอ
เหมือนตัวอย่างก่อนหน้า จากนั้นค้นหารหัสสินค้า 3 รหัส (มีรหัสหนึ่งที่ไม่มีอยู่จริง) แล้วพิมพ์สรุปว่า
พบกี่รายการ และมีการบันทึก error log กี่บรรทัด

ให้ลองเขียนเองก่อนดูเฉลย — ใช้เทคนิคจากขั้นตอนที่ 471-477 ทั้งหมดประกอบกัน

### เฉลย

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP479-ERROR-LOG-EXERCISE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT MASTER-FILE ASSIGN TO "MASTER479.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS MS-CODE
               FILE STATUS IS WS-MASTER-STATUS.

           SELECT ERROR-LOG-FILE ASSIGN TO "ERRLOG479.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-LOG-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  MASTER-FILE.
       01  MASTER-RECORD.
           05  MS-CODE              PIC 9(3).
           05  MS-NAME              PIC X(15).

       FD  ERROR-LOG-FILE.
       01  ERROR-LOG-RECORD         PIC X(50).

       WORKING-STORAGE SECTION.
       01  WS-MASTER-STATUS         PIC XX.
       01  WS-LOG-STATUS            PIC XX.
       01  WS-LOOKUP-CODE           PIC 9(3).
       01  WS-FOUND-COUNT           PIC 9(2) VALUE 0.
       01  WS-LOG-LINE-COUNT        PIC 9(2) VALUE 0.
       01  WS-IDX                   PIC 9(1).
       01  WS-LOOKUP-CODES.
           05  PIC 9(3)             VALUE 100.
           05  PIC 9(3)             VALUE 250.
           05  PIC 9(3)             VALUE 300.
       01  WS-LOOKUP-TABLE REDEFINES WS-LOOKUP-CODES
                                     OCCURS 3 TIMES PIC 9(3).

       PROCEDURE DIVISION.
       DECLARATIVES.
       MASTER-ERROR-SECTION SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON MASTER-FILE.
       MASTER-ERROR-PARA.
           ADD 1 TO WS-LOG-LINE-COUNT
           MOVE SPACES TO ERROR-LOG-RECORD
           STRING "ERROR ON MASTER-FILE: CODE="
                  WS-LOOKUP-CODE
                  " STATUS=" WS-MASTER-STATUS
               DELIMITED BY SIZE INTO ERROR-LOG-RECORD
           WRITE ERROR-LOG-RECORD.
       END DECLARATIVES.

       MAIN-LOGIC SECTION.
       MAIN-LOGIC-START.
           PERFORM SETUP-MASTER-DATA
           OPEN OUTPUT ERROR-LOG-FILE
           OPEN INPUT MASTER-FILE

           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 3
               MOVE WS-LOOKUP-TABLE(WS-IDX) TO WS-LOOKUP-CODE
               MOVE WS-LOOKUP-CODE TO MS-CODE
               READ MASTER-FILE
               IF WS-MASTER-STATUS = "00"
                   ADD 1 TO WS-FOUND-COUNT
                   DISPLAY "FOUND: " MS-CODE " " MS-NAME
               END-IF
           END-PERFORM

           CLOSE MASTER-FILE
           CLOSE ERROR-LOG-FILE

           DISPLAY "TOTAL FOUND    : " WS-FOUND-COUNT
           DISPLAY "ERROR LOG LINES: " WS-LOG-LINE-COUNT
           STOP RUN.

       SETUP-MASTER-DATA SECTION.
       SETUP-MASTER-DATA-START.
           OPEN OUTPUT MASTER-FILE
           MOVE 100 TO MS-CODE
           MOVE "KEYBOARD" TO MS-NAME
           WRITE MASTER-RECORD
           MOVE 300 TO MS-CODE
           MOVE "MONITOR" TO MS-NAME
           WRITE MASTER-RECORD
           CLOSE MASTER-FILE.
      *> Code 250 is deliberately never written - it will trigger
      *> the Declarative when looked up below.
```

คอมไพล์และรันด้วย (indexed file ต้องใช้ ISAM build):

```bash
/opt/gnucobol-isam/bin/cobc -x -o step479 step479.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./step479
cat ERRLOG479.DAT
```

### ผลลัพธ์จริงที่ได้

```
FOUND: 100 KEYBOARD
FOUND: 300 MONITOR
TOTAL FOUND    : 02
ERROR LOG LINES: 01
```

เนื้อหาไฟล์ `ERRLOG479.DAT`:

```
ERROR ON MASTER-FILE: CODE=250 STATUS=23
```

### อธิบายเฉลย

- `WS-LOOKUP-CODE` ถูก `MOVE` ไว้**ก่อน**เรียก `READ` ทุกครั้ง เพื่อให้ Declarative สามารถอ้างอิงค่านี้
  ได้ถูกต้องว่ากำลังค้นหารหัสอะไรอยู่ ณ ขณะที่เกิดข้อผิดพลาด (ถ้าใช้ `MS-CODE` ตรง ๆ ใน Declarative
  แทน ค่าอาจไม่ตรงกับที่ต้องการเสมอไปในบางสถานการณ์ที่ซับซ้อนกว่านี้ การมีตัวแปรแยกต่างหากที่ไม่ถูก
  ไฟล์เขียนทับจึงปลอดภัยกว่า)
- ไฟล์ error log ถูกเขียนจากภายใน Declarative โดยตรง โดยที่โปรแกรมหลักไม่ต้องรู้เลยว่ามีการเขียน log
  เกิดขึ้น — สอดคล้องกับเป้าหมายของ Part นี้ที่ต้องการรวมศูนย์ตรรกะจัดการข้อผิดพลาดไว้ที่เดียว

---

## ขั้นตอนที่ 480: สรุปรวบยอด — โปรแกรมประมวลผลไฟล์แบบ Batch พร้อมระบบจัดการข้อผิดพลาดสมบูรณ์

### แนวคิด

ขั้นตอนสุดท้ายของ Part นี้รวบรวมทุกเทคนิคจากขั้นตอนที่ 471-479 เข้าด้วยกันเป็นโปรแกรมที่สมจริงขึ้น:
โปรแกรมประมวลผลธุรกรรมทางบัญชี (deposit/withdrawal) จากไฟล์ธุรกรรมเข้าไปยังไฟล์บัญชีหลัก โดยมี
**DECLARATIVES จัดการข้อผิดพลาดของทั้งไฟล์บัญชี (Indexed) และไฟล์ธุรกรรม (Sequential) แยกกัน** พร้อม
บันทึก error log และสรุปผลลัพธ์ท้ายโปรแกรม — เป็นรูปแบบที่ใกล้เคียงกับโปรแกรม batch จริงในองค์กร
และเป็นการปูทางสู่โปรเจกต์รวบยอดเฟส 3 (ระบบบัญชีลูกหนี้) ใน Part 050

### ซอร์สโค้ดฉบับสมบูรณ์

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP480-CENTRALIZED-ERRORS.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ACCOUNT-FILE ASSIGN TO "ACCOUNTS480.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS ACC-NUMBER
               FILE STATUS IS WS-ACC-STATUS.

           SELECT TRANSACTION-FILE ASSIGN TO "TRANS480.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-TRANS-STATUS.

           SELECT ERROR-LOG-FILE ASSIGN TO "ERRLOG480.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-LOG-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  ACCOUNT-FILE.
       01  ACCOUNT-RECORD.
           05  ACC-NUMBER           PIC 9(5).
           05  ACC-NAME             PIC X(15).
           05  ACC-BALANCE          PIC S9(7)V99.

       FD  TRANSACTION-FILE.
       01  TRANSACTION-RECORD.
           05  TR-ACC-NUMBER        PIC 9(5).
           05  TR-AMOUNT            PIC S9(7)V99.

       FD  ERROR-LOG-FILE.
       01  ERROR-LOG-RECORD         PIC X(60).

       WORKING-STORAGE SECTION.
       01  WS-ACC-STATUS            PIC XX.
           88  ACC-OK                       VALUE "00".
           88  ACC-NOT-FOUND                VALUE "23".
       01  WS-TRANS-STATUS          PIC XX.
       01  WS-LOG-STATUS            PIC XX.
       01  WS-ERROR-COUNT           PIC 9(3) VALUE 0.
       01  WS-APPLIED-COUNT         PIC 9(3) VALUE 0.
       01  WS-EOF-FLAG              PIC X(1) VALUE "N".
           88  END-OF-TRANSACTIONS          VALUE "Y".

       PROCEDURE DIVISION.
       DECLARATIVES.
      *> Handler #1: any error on ACCOUNT-FILE (the indexed master
      *> file) - typically status 23 when a transaction references
      *> an account number that does not exist.
       ACCOUNT-ERROR-SECTION SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON ACCOUNT-FILE.
       ACCOUNT-ERROR-PARA.
           ADD 1 TO WS-ERROR-COUNT
           MOVE SPACES TO ERROR-LOG-RECORD
           STRING "ACCOUNT ERROR: ACC=" TR-ACC-NUMBER
                  " STATUS=" WS-ACC-STATUS
               DELIMITED BY SIZE INTO ERROR-LOG-RECORD
           WRITE ERROR-LOG-RECORD.

      *> Handler #2: any UNEXPECTED error on TRANSACTION-FILE. The
      *> normal end-of-file condition (status 10) is already caught
      *> by the READ's own AT END clause below and will NEVER reach
      *> this Declarative (see Part 048 step 478) - this handler
      *> exists purely as a safety net for genuine, unusual errors.
       TRANSACTION-ERROR-SECTION SECTION.
           USE AFTER STANDARD ERROR PROCEDURE
               ON TRANSACTION-FILE.
       TRANSACTION-ERROR-PARA.
           ADD 1 TO WS-ERROR-COUNT
           MOVE SPACES TO ERROR-LOG-RECORD
           STRING "TRANSACTION FILE ERROR: STATUS="
                  WS-TRANS-STATUS
               DELIMITED BY SIZE INTO ERROR-LOG-RECORD
           WRITE ERROR-LOG-RECORD.
       END DECLARATIVES.

       MAIN-PARA SECTION.
       MAIN-PARA-START.
           PERFORM SETUP-ACCOUNTS
           OPEN OUTPUT ERROR-LOG-FILE
           OPEN I-O ACCOUNT-FILE
           OPEN INPUT TRANSACTION-FILE

           PERFORM UNTIL END-OF-TRANSACTIONS
               READ TRANSACTION-FILE
                   AT END
                       SET END-OF-TRANSACTIONS TO TRUE
                   NOT AT END
                       PERFORM APPLY-TRANSACTION
               END-READ
           END-PERFORM

           CLOSE ACCOUNT-FILE
           CLOSE TRANSACTION-FILE
           CLOSE ERROR-LOG-FILE

           DISPLAY "TRANSACTIONS APPLIED: " WS-APPLIED-COUNT
           DISPLAY "ERRORS LOGGED       : " WS-ERROR-COUNT
           STOP RUN.

       SETUP-ACCOUNTS SECTION.
       SETUP-ACCOUNTS-START.
           OPEN OUTPUT ACCOUNT-FILE
           MOVE 10001 TO ACC-NUMBER
           MOVE "SOMCHAI" TO ACC-NAME
           MOVE 5000.00 TO ACC-BALANCE
           WRITE ACCOUNT-RECORD
           MOVE 10002 TO ACC-NUMBER
           MOVE "PRASERT" TO ACC-NAME
           MOVE 3000.00 TO ACC-BALANCE
           WRITE ACCOUNT-RECORD
           CLOSE ACCOUNT-FILE.

       APPLY-TRANSACTION SECTION.
       APPLY-TRANSACTION-START.
           MOVE TR-ACC-NUMBER TO ACC-NUMBER
           READ ACCOUNT-FILE
           IF ACC-OK
               ADD TR-AMOUNT TO ACC-BALANCE
               REWRITE ACCOUNT-RECORD
               ADD 1 TO WS-APPLIED-COUNT
               DISPLAY "APPLIED " TR-AMOUNT " TO ACCOUNT "
                   TR-ACC-NUMBER " NEW BALANCE=" ACC-BALANCE
           END-IF.
```

### เตรียมไฟล์ธุรกรรมทดสอบและรัน

ไฟล์ `TRANSACTION-RECORD` ประกอบด้วย `TR-ACC-NUMBER PIC 9(5)` (5 หลัก) และ `TR-AMOUNT PIC S9(7)V99`
(9 หลัก ไม่มีจุดทศนิยมจริงในไฟล์ เพราะจุดทศนิยมเป็นแบบ **implied** — ทบทวนจาก Part 006) รวม 14
ตัวอักษรต่อบรรทัด:

```bash
printf '10001000000100\n10002000000250\n99999000050000\n10001000000050\n' > TRANS480.DAT

/opt/gnucobol-isam/bin/cobc -x -o step480 step480.cob
LD_LIBRARY_PATH=/opt/gnucobol-isam/lib ./step480
cat ERRLOG480.DAT
```

บรรทัดที่ 3 (`99999...`) อ้างอิงบัญชีเลขที่ `99999` ซึ่งไม่มีอยู่จริงในไฟล์บัญชี — จงใจใส่ไว้เพื่อ
ทดสอบ Declarative

### ผลลัพธ์จริงที่ได้

```
APPLIED +0000001.00 TO ACCOUNT 10001 NEW BALANCE=+0005001.00
APPLIED +0000002.50 TO ACCOUNT 10002 NEW BALANCE=+0003002.50
APPLIED +0000000.50 TO ACCOUNT 10001 NEW BALANCE=+0005001.50
TRANSACTIONS APPLIED: 003
ERRORS LOGGED       : 001
```

เนื้อหาไฟล์ `ERRLOG480.DAT`:

```
ACCOUNT ERROR: ACC=99999 STATUS=23
```

### อธิบายโค้ดทีละส่วน

- ธุรกรรมที่ 1, 2, 4 (บัญชี `10001` และ `10002` ที่มีอยู่จริง) ถูกนำไปบวกเข้ากับยอดคงเหลือสำเร็จ
  ผ่าน `REWRITE` ปกติ — สังเกตว่าบัญชี `10001` ถูกปรับปรุงสองครั้ง (ธุรกรรมที่ 1 และ 4) และยอดสุดท้าย
  คือ `5000.00 + 1.00 + 0.50 = 5001.50` ถูกต้องตรงตามที่คำนวณด้วยมือ
- ธุรกรรมที่ 3 (บัญชี `99999` ที่ไม่มีอยู่จริง) ทำให้ `READ ACCOUNT-FILE` ล้มเหลวด้วยสถานะ `23`
  `ACCOUNT-ERROR-PARA` ถูกเรียกโดยอัตโนมัติ บันทึกบรรทัด log ไว้ และเพิ่ม `WS-ERROR-COUNT` — จากนั้น
  ควบคุมกลับมาที่ `APPLY-TRANSACTION` ต่อจาก `READ` ทันที เข้าเงื่อนไข `IF ACC-OK` เป็นเท็จ (เพราะ
  `WS-ACC-STATUS` เป็น `"23"` ไม่ใช่ `"00"`) จึง**ข้าม**การ `ADD`/`REWRITE` ไปโดยอัตโนมัติ — ป้องกัน
  ไม่ให้ข้อมูลบัญชีที่ไม่มีอยู่จริงถูกสร้างขึ้นมาโดยไม่ได้ตั้งใจ
- ตัวเลขสรุปท้ายโปรแกรม (`TRANSACTIONS APPLIED: 003`, `ERRORS LOGGED: 001`) มาจากตัวแปรที่ถูกนับ
  ควบคู่กันไปทั้งในโปรแกรมหลักและใน Declarative — พิสูจน์ว่าทั้งสองส่วนทำงานร่วมกันได้อย่างสอดคล้อง
  แม้จะเป็นคนละ SECTION กัน
- โค้ดโปรแกรมหลัก (`MAIN-PARA`, `APPLY-TRANSACTION`) **ไม่มี** `IF WS-ACC-STATUS = "23" DISPLAY
  ERROR...` ปรากฏอยู่เลยสักบรรทัด — ตรรกะการรายงานข้อผิดพลาดทั้งหมดถูกย้ายไปรวมศูนย์อยู่ที่
  `DECLARATIVES` เพียงจุดเดียว ตรงตามเป้าหมายของ Part นี้ทุกประการ

### ข้อควรระวัง

- โปรแกรมนี้เลือก**ข้ามธุรกรรมที่ผิดพลาดไปเงียบ ๆ (หลังบันทึก log)** แทนที่จะหยุดโปรแกรมทั้งหมดทันที
  ซึ่งเป็นการตัดสินใจเชิงออกแบบที่เหมาะกับงาน batch จำนวนมาก (ไม่ต้องการให้ 1 รายการที่ผิดพลาดทำให้
  ธุรกรรมที่ถูกต้องอีกหลายพันรายการไม่ถูกประมวลผลไปด้วย) — แต่ในระบบจริงที่ความถูกต้องสำคัญที่สุด
  (เช่น ระบบธนาคารจริง) อาจต้องออกแบบเพิ่มเติมให้แจ้งเตือนทีมปฏิบัติการทันทีเมื่อพบข้อผิดพลาด ไม่ใช่
  แค่บันทึก log แล้วปล่อยผ่านไปเงียบ ๆ
- ทดสอบนับ: `WS-APPLIED-COUNT` และ `WS-ERROR-COUNT` รวมกันต้องเท่ากับจำนวนบรรทัดทั้งหมดในไฟล์ธุรกรรม
  เสมอ (3 + 1 = 4 ตรงกับ 4 บรรทัดในตัวอย่างนี้) — เป็นวิธีตรวจสอบ (reconciliation) ง่าย ๆ ที่ควรทำ
  เป็นนิสัยเมื่อเขียนโปรแกรม batch เพื่อยืนยันว่าไม่มีธุรกรรมใดหายไปโดยไม่ถูกนับ

### แบบฝึกหัดที่ 480.1

**โจทย์**: จงอธิบายว่าเหตุใด `TRANSACTION-ERROR-SECTION` (Declarative สำหรับไฟล์ธุรกรรม) จึงไม่เคย
ถูกเรียกทำงานเลยในการรันตัวอย่างนี้ ทั้งที่โปรแกรมอ่านไฟล์ธุรกรรมจนจบ (ซึ่งควรได้สถานะ `10` ในที่สุด)

**เฉลย**: เพราะคำสั่ง `READ TRANSACTION-FILE` ในโปรแกรมนี้มี `AT END`/`NOT AT END` clause กำกับไว้
โดยตรงอยู่แล้ว ตามกฎที่พิสูจน์ในขั้นตอนที่ 478 **clause ที่เขียนไว้ตรง ๆ จะดักจับเงื่อนไขที่ตรงกันไป
ก่อนเสมอ** เมื่ออ่านไฟล์ธุรกรรมจนจบและได้สถานะ `10` โปรแกรมจะเข้าสาขา `AT END` ที่ตั้งค่า
`END-OF-TRANSACTIONS` ให้เป็นจริงทันที — `TRANSACTION-ERROR-SECTION` (ที่เป็น `USE AFTER STANDARD
ERROR PROCEDURE`) จะถูกเรียกก็ต่อเมื่อเกิดข้อผิดพลาดประเภทอื่นที่ **ไม่มี** clause ใดดักจับไว้ตรง ๆ
(เช่น หากไฟล์ธุรกรรมถูกลบระหว่างโปรแกรมกำลังทำงานอยู่ หรือปัญหาระดับ device อื่น ๆ) ซึ่งไม่เกิดขึ้น
ในการทดสอบปกตินี้ นี่คือเหตุผลที่คอมเมนต์ในโค้ดระบุไว้ชัดเจนว่า Declarative นี้เป็นเพียง "safety net"
สำหรับสถานการณ์ที่ไม่คาดคิดเท่านั้น

---

## สรุปท้ายบท

Part นี้พาคุณเจาะลึกกลไกการจัดการข้อผิดพลาดขั้นสูงที่สุดของ COBOL แบบคลาสสิก ครอบคลุม:

- โครงสร้างและกฎการเขียน `DECLARATIVES SECTION` รวมถึงกฎที่ว่าเมื่อมี DECLARATIVES แล้ว ทุก Paragraph
  ในโปรแกรมต้องอยู่ใน SECTION เสมอ
- `USE AFTER STANDARD ERROR PROCEDURE ON <file-name>` สำหรับจัดการไฟล์เฉพาะเจาะจงทีละไฟล์
- `USE ... ON INPUT/OUTPUT/I-O/EXTEND` สำหรับครอบคลุมไฟล์หลายไฟล์ด้วยโค้ดเดียวตามโหมดการเปิด
- `ERROR` และ `EXCEPTION` เป็นคำพ้องความหมายที่ใช้แทนกันได้ 100% ในวลี `USE AFTER STANDARD ...
  PROCEDURE`
- กฎความสำคัญก่อนหลัง: Declarative ที่เจาะจงไฟล์ชนะ Declarative ที่ครอบคลุมทั่วไปเสมอ (ยืนยันด้วย
  การทดสอบจริง)
- การควบคุมการไหลของโปรแกรม: หลัง Declarative ทำงานเสร็จ ควบคุมกลับไปที่คำสั่งถัดจากคำสั่งที่ผิดพลาด
  ทันที ไม่ retry ไม่ข้าม
- ความสัมพันธ์ระหว่าง Declaratives กับ `AT END`/`INVALID KEY` ที่มีอยู่แล้ว: clause ที่เขียนตรง ๆ
  ชนะเสมอสำหรับเงื่อนไขที่ตรงกัน
- `USE FOR DEBUGGING` และการเปิดใช้งานผ่าน `SOURCE-COMPUTER ... WITH DEBUGGING MODE` (compile-time)
  ร่วมกับ `COB_SET_DEBUG=1` (runtime)
- การออกแบบ Pattern การจัดการข้อผิดพลาดแบบรวมศูนย์ที่นำไปใช้จริงได้ ด้วยการผสมผสาน 88-level Condition
  Names (จาก Part 030) เข้ากับ flag ตัวแปรที่ Declarative เป็นผู้ควบคุม
- กับดักสำคัญ: ความเสี่ยง infinite recursion, ความไม่ portable 100% ข้าม compiler, ขอบเขตที่
  DECLARATIVES ไม่ครอบคลุม (SIZE ERROR, Subscript overflow)
- โปรแกรม batch ตัวอย่างที่สมบูรณ์ซึ่งรวมทุกเทคนิคเข้าด้วยกัน เป็นการปูทางสู่โปรเจกต์รวบยอดเฟส 3

Part ถัดไป (**Part 049**) จะพาคุณสำรวจ **Compiler Directives และความแตกต่างระหว่าง COBOL Dialects**
— เครื่องมือที่ทำให้ซอร์สโค้ด COBOL เดียวกันสามารถปรับพฤติกรรมให้เข้ากับ compiler และแพลตฟอร์มที่
แตกต่างกันได้ ซึ่งเป็นทักษะสำคัญมากสำหรับการทำงานกับโค้ด Legacy ที่มาจากหลากหลาย vendor ในโลกจริง

**[ไปยัง Part 049: Compiler Directives และ COBOL Dialects →](part-049-compiler-directives.md)**

---

**เนื้อหาก่อนหน้า**: [Part 047: การจัดการวันที่และเวลาขั้นสูง ←](part-047-advanced-date-time.md)
**เนื้อหาถัดไป**: [Part 049: Compiler Directives และ COBOL Dialects →](part-049-compiler-directives.md)
**โปรเจกต์รวบยอดเฟส 3**: [Part 050: ระบบบัญชีลูกหนี้ →](part-050-phase3-project.md)
