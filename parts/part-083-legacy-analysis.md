# Part 083: Legacy System Analysis และ Reverse Engineering (ขั้นตอนที่ 821–830)

## คำนำของ Part นี้

Part 081 และ 082 สอนให้เรารู้จัก Clean Code และ Design Patterns สำหรับ COBOL — เทคนิคเหล่านั้นใช้ได้ดี
เมื่อเรา**เขียนโค้ดใหม่**หรือแก้ไขโค้ดของเราเอง แต่ในโลกการทำงานจริง งานที่พบบ่อยกว่ามากคือการต้องเข้าไป
**ทำความเข้าใจโค้ด COBOL ที่คนอื่นเขียนไว้เมื่อหลายสิบปีก่อน** โดยไม่มีเอกสารประกอบ ไม่มีคนเขียนเดิมให้ถาม
และบางครั้งแม้แต่ตัวแปรก็ตั้งชื่อแบบ `TMP`, `X1`, `WS-FLAG-2` ที่ไม่สื่อความหมายอะไรเลย

นี่คือทักษะที่เรียกว่า **Legacy System Analysis** หรือ **Reverse Engineering**: การอ่านโค้ดที่มีอยู่แล้ว
อย่างเป็นระบบ เพื่อสร้างความเข้าใจใหม่เกี่ยวกับ**โครงสร้างข้อมูล** (data model), **เส้นทางการทำงาน**
(control flow), และที่สำคัญที่สุดคือ **กฎทางธุรกิจ** (business rules) ที่ฝังอยู่ในโค้ด — โดยที่เรามักไม่มี
สิทธิ์แก้ไขโค้ดนั้นเลยด้วยซ้ำในขั้นตอนนี้ เรากำลังแค่ "อ่านและทำความเข้าใจ" เท่านั้น

Part นี้จะแนะนำ **โปรแกรม LEGACYORD** ซึ่งเป็นโปรแกรม COBOL ที่จำลองสถานการณ์ระบบ Legacy จริงอย่างตั้งใจ:
ชื่อผู้เขียนเดิมหายไป ตัวแปรบางตัวตั้งชื่อคลุมเครือ มีการใช้ `GO TO`, มีโค้ดที่ตายแล้ว (dead code) ซ่อนอยู่
และมีกฎธุรกิจซับซ้อนฝังอยู่ใน `IF`/`EVALUATE` ที่ซ้อนกันหลายชั้นโดยไม่มีคำอธิบาย — เราจะใช้โปรแกรมนี้เป็น
"ผู้ป่วย" สำหรับฝึกเทคนิคการวิเคราะห์ทั้ง 10 ขั้นตอนของ Part นี้ ตั้งแต่การอ่าน FD/Copybook เพื่อสร้าง
Data Model ใหม่, การไล่ตาม PERFORM graph, การหา Dead Code, การใช้เครื่องมือ Cross-reference, ไปจนถึงการ
สกัดกฎธุรกิจออกมาเป็นเอกสารที่มนุษย์อ่านเข้าใจ — ทุกเทคนิคจะถูกทดสอบจริงกับโปรแกรมนี้ ไม่ใช่แค่ทฤษฎี

ความรู้จาก Part นี้จะถูกนำไปใช้ต่อใน **Part 084** (ตัดสินใจว่าจะ Migrate ระบบนี้อย่างไร) และ **Part 085**
(โปรเจกต์รวบยอดที่จะปรับปรุงระบบ Legacy จริงด้วยสถาปัตยกรรมสมัยใหม่)

---

## ขั้นตอนที่ 821: ทำไมต้องมี Legacy Analysis และทำความรู้จักระบบ "ผู้ป่วย"

### ทำไม Reverse Engineering ถึงเป็นทักษะที่จำเป็น

ในองค์กรที่มีระบบ COBOL อายุ 20-40 ปี (ตามที่กล่าวถึงใน Part 001) ความจริงที่พบบ่อยมากคือ:

1. **โปรแกรมเมอร์ดั้งเดิมเกษียณหรือลาออกไปแล้ว** — ไม่มีใครอธิบาย "ทำไม" โค้ดถึงเขียนแบบนี้ได้อีก
2. **เอกสารการออกแบบ (Design Document) สูญหายหรือไม่เคยมีอยู่จริง** — มีแต่ตัวโค้ดเป็นความจริงหนึ่งเดียว
   (Code is the only source of truth)
3. **กฎทางธุรกิจสำคัญซ่อนอยู่ในโค้ด** — เช่น "ทำไมลูกค้าระดับ Gold ถึงได้ส่วนลด 7% ไม่ใช่ 5%" คำตอบอาจ
   ไม่ได้มาจากนโยบายบริษัทที่เขียนไว้ที่ไหน แต่มาจากบรรทัดโค้ดที่มีคนเขียนไว้เมื่อ 15 ปีก่อน
4. **ต้องแก้บั๊กหรือเพิ่มฟีเจอร์โดยไม่ทำให้ของเดิมพัง** — จำเป็นต้องเข้าใจผลกระทบ (impact) ก่อนแตะโค้ด

ก่อนที่เราจะ Refactor (Part 081), ใช้ Design Pattern (Part 082), หรือ Migrate (Part 084) สิ่งใด ๆ ก็ตาม
เราต้อง**เข้าใจของเดิมให้ถูกต้องเสียก่อน** มิฉะนั้นความเสี่ยงที่จะทำลายพฤติกรรมที่ถูกต้องของระบบจะสูงมาก

### ทำความรู้จักโปรแกรม "ผู้ป่วย": LEGACYORD

ต่อไปนี้คือซอร์สโค้ดฉบับเต็มของโปรแกรมที่เราจะวิเคราะห์ตลอดทั้ง Part นี้ ลองอ่านผ่าน ๆ ครั้งแรกโดยยัง
ไม่ต้องพยายามเข้าใจทุกบรรทัด — สังเกตแค่ความรู้สึกโดยรวมว่าโค้ดแบบนี้ "อ่านยาก" กว่าโค้ดที่เราเขียนเองใน
Part ก่อน ๆ อย่างไร:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. LEGACYORD.
       AUTHOR. UNKNOWN-LEGACY-TEAM.
      *> Original author information was lost over the years - this
      *> is typical of real legacy code: nobody left who wrote it.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ORD-FILE ASSIGN TO "ORDERS.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-FS.
           SELECT RPT-FILE ASSIGN TO "ORDRPT.TXT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  ORD-FILE.
       01  ORD-REC.
           05  OR-CUST-ID          PIC 9(5).
           05  OR-AMT              PIC 9(7)V99.
           05  OR-TIER             PIC X(1).
           05  OR-REGION           PIC X(1).

       FD  RPT-FILE.
       01  RPT-LINE                PIC X(80).

       WORKING-STORAGE SECTION.
       01  WS-FS                   PIC XX.
       01  WS-EOF                  PIC X VALUE "N".
           88  EOF-REACHED               VALUE "Y".
       01  WS-DPCT                 PIC 9V999.
       01  WS-DAMT                 PIC 9(7)V99.
       01  WS-NET                  PIC 9(7)V99.
       01  WS-CUST-BAL             PIC 9(9)V99 VALUE 0.
       01  WS-REC-COUNT            PIC 9(5) VALUE 0.
       01  WS-BAD-COUNT            PIC 9(5) VALUE 0.
       01  WS-VALID-FLAG           PIC X VALUE "Y".
           88  REC-IS-VALID              VALUE "Y".
           88  REC-IS-INVALID            VALUE "N".

      *> WS-OLD-DPCT was used by OLD-CALC-DISCOUNT-V1 below, back
      *> when discount had a single flat rate. Nobody removed the
      *> field or the paragraph when the tiered scheme replaced it.
       01  WS-OLD-DPCT             PIC 9V99 VALUE 0.10.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM INIT-RTN
           PERFORM PROCESS-LOOP THRU PROCESS-LOOP-EXIT
               UNTIL EOF-REACHED
           PERFORM TERM-RTN
           STOP RUN.

       INIT-RTN.
           OPEN INPUT ORD-FILE
           OPEN OUTPUT RPT-FILE
           MOVE "CUST   AMOUNT  TIER REGION DISC%   NET-AMT  BALANCE"
               TO RPT-LINE
           WRITE RPT-LINE.

       PROCESS-LOOP.
           READ ORD-FILE
               AT END
                   SET EOF-REACHED TO TRUE
               NOT AT END
                   PERFORM VALIDATE-ORDER
                   IF REC-IS-VALID
                       PERFORM CALC-DISCOUNT
                       PERFORM UPDATE-BALANCE
                       PERFORM PRINT-LINE
                   ELSE
                       ADD 1 TO WS-BAD-COUNT
                       GO TO PROCESS-LOOP-EXIT
                   END-IF
           END-READ.
       PROCESS-LOOP-EXIT.
           EXIT.

       VALIDATE-ORDER.
           SET REC-IS-VALID TO TRUE
           ADD 1 TO WS-REC-COUNT
           IF OR-AMT = ZERO
               SET REC-IS-INVALID TO TRUE
           END-IF
           IF OR-TIER NOT = "P" AND OR-TIER NOT = "G"
                   AND OR-TIER NOT = "R"
               SET REC-IS-INVALID TO TRUE
           END-IF.

       CALC-DISCOUNT.
           EVALUATE OR-TIER
               WHEN "P"
                   IF OR-AMT >= 10000.00
                       MOVE 0.150 TO WS-DPCT
                   ELSE
                       IF OR-AMT >= 5000.00
                           MOVE 0.100 TO WS-DPCT
                       ELSE
                           MOVE 0.050 TO WS-DPCT
                       END-IF
                   END-IF
               WHEN "G"
                   IF OR-AMT >= 10000.00
                       MOVE 0.100 TO WS-DPCT
                   ELSE
                       IF OR-AMT >= 5000.00
                           MOVE 0.070 TO WS-DPCT
                       ELSE
                           MOVE 0.030 TO WS-DPCT
                       END-IF
                   END-IF
               WHEN "R"
                   IF OR-AMT >= 10000.00
                       MOVE 0.050 TO WS-DPCT
                   ELSE
                       IF OR-AMT >= 5000.00
                           MOVE 0.020 TO WS-DPCT
                       ELSE
                           MOVE 0.000 TO WS-DPCT
                       END-IF
                   END-IF
           END-EVALUATE

           IF OR-REGION = "X"
               ADD 0.020 TO WS-DPCT
               IF WS-DPCT > 0.200
                   MOVE 0.200 TO WS-DPCT
               END-IF
           END-IF

           COMPUTE WS-DAMT = OR-AMT * WS-DPCT
           COMPUTE WS-NET = OR-AMT - WS-DAMT.

       OLD-CALC-DISCOUNT-V1.
           COMPUTE WS-DAMT = OR-AMT * WS-OLD-DPCT
           COMPUTE WS-NET = OR-AMT - WS-DAMT.

       UPDATE-BALANCE.
           ADD WS-NET TO WS-CUST-BAL.

       PRINT-LINE.
           MOVE SPACES TO RPT-LINE
           STRING OR-CUST-ID    DELIMITED BY SIZE
                   " "          DELIMITED BY SIZE
                   OR-AMT       DELIMITED BY SIZE
                   " "          DELIMITED BY SIZE
                   OR-TIER      DELIMITED BY SIZE
                   "    "       DELIMITED BY SIZE
                   OR-REGION    DELIMITED BY SIZE
                   "      "     DELIMITED BY SIZE
                   WS-DPCT      DELIMITED BY SIZE
                   " "          DELIMITED BY SIZE
                   WS-NET       DELIMITED BY SIZE
                   " "          DELIMITED BY SIZE
                   WS-CUST-BAL  DELIMITED BY SIZE
               INTO RPT-LINE
           END-STRING
           WRITE RPT-LINE.

       TERM-RTN.
           CLOSE ORD-FILE
           CLOSE RPT-FILE
           DISPLAY "RECORDS READ: " WS-REC-COUNT
           DISPLAY "RECORDS REJECTED: " WS-BAD-COUNT
           DISPLAY "FINAL BALANCE: " WS-CUST-BAL.
```

ข้อมูลนำเข้าตัวอย่าง (`ORDERS.DAT`) แต่ละบรรทัดยาว 16 ตัวอักษรพอดี ตรงกับ `ORD-REC`:

```bash
cat > ORDERS.DAT << 'EOF'
10001001500000PR
10002000800000GX
10003000000000GR
10004002000000PX
10005000300000RR
EOF
```

คอมไพล์และรัน:

```bash
cobc -x -o legacyord legacyord.cob
./legacyord
```

**ผลลัพธ์จริงที่ได้:**

```
RECORDS READ: 00005
RECORDS REJECTED: 00001
FINAL BALANCE: 000039630.00
```

และไฟล์ `ORDRPT.TXT` ที่ได้:

```
CUST   AMOUNT  TIER REGION DISC%   NET-AMT  BALANCE
10001 001500000 P    R      0150 001275000 00001275000
10002 000800000 G    X      0090 000728000 00002003000
10004 002000000 P    X      0170 001660000 00003663000
10005 000300000 R    R      0000 000300000 00003963000
```

### สังเกตอาการของ "โค้ด Legacy" ในโปรแกรมนี้

- ไม่มีเอกสารอธิบายกฎธุรกิจของส่วนลดเลยแม้แต่บรรทัดเดียว
- มี `GO TO PROCESS-LOOP-EXIT` ซึ่งเป็นรูปแบบควบคุมการไหลที่ภาษาสมัยใหม่หลีกเลี่ยง (ทบทวน Part 014)
- มีย่อหน้า `OLD-CALC-DISCOUNT-V1` ที่ดูเหมือนทำหน้าที่เดียวกับ `CALC-DISCOUNT` — แต่ใช้งานจริงหรือไม่?
- ตัวเลข `10000.00`, `5000.00`, `0.150`, `0.020`, `0.200` ปรากฏตรง ๆ ในโค้ดโดยไม่มีชื่อที่สื่อความหมาย
  (Magic Numbers — ปัญหาที่ Part 081 สอนให้แก้ด้วยการตั้งชื่อค่าคงที่)

ตลอดขั้นตอนที่ 822-830 เราจะ "ผ่าตัด" โปรแกรมนี้ด้วยเทคนิคต่าง ๆ เพื่อตอบคำถามเหล่านี้ทั้งหมดอย่างเป็น
ระบบ โดยที่**ยังไม่แก้โค้ดสักบรรทัดเดียว** — Legacy Analysis คือขั้นตอนก่อนหน้า Refactoring หรือ Migration
เสมอ

### ข้อควรระวัง

- อย่าเพิ่งรีบแก้โค้ด Legacy ก่อนที่จะเข้าใจมันอย่างถ่องแท้ — การแก้ไขโดยไม่เข้าใจผลกระทบทั้งหมดคือสาเหตุ
  อันดับต้น ๆ ของเหตุการณ์ระบบล่มในองค์กรจริง
- โปรแกรม LEGACYORD ในเอกสารนี้ถูกออกแบบมาให้มีปัญหาลักษณะเฉพาะ (GO TO, Dead Code, Magic Numbers) เพื่อ
  จุดประสงค์การสอน — ระบบ Legacy จริงมักซับซ้อนกว่านี้มาก บางระบบมีโค้ดหลายแสนถึงหลายล้านบรรทัด เทคนิค
  เดียวกันนี้ยังใช้ได้ แต่ต้องอาศัยเครื่องมืออัตโนมัติ (Static Analysis Tools) ช่วยมากขึ้นตามขนาดของระบบ

### แบบฝึกหัดที่ 821.1

**โจทย์**: จากการอ่านโค้ด `LEGACYORD` ผ่าน ๆ ครั้งแรก จงลิสต์รายการ "คำถามที่ยังตอบไม่ได้" อย่างน้อย 3 ข้อ
ที่คุณอยากรู้คำตอบก่อนจะแก้ไขโปรแกรมนี้

**เฉลยแนวทาง**: คำตอบขึ้นกับผู้อ่านแต่ละคน แต่ตัวอย่างคำถามที่ควรมี เช่น (1) `OLD-CALC-DISCOUNT-V1` ยังมี
ใครเรียกใช้อยู่หรือไม่ (2) ตัวเลข `10000.00`/`5000.00` มีที่มาจากนโยบายธุรกิจอะไร เปลี่ยนบ่อยแค่ไหน
(3) ทำไมส่วนลดสูงสุดถึงถูกจำกัดไว้ที่ 20% พอดี (4) `GO TO PROCESS-LOOP-EXIT` ส่งผลต่อการทำงานอย่างไรเมื่อ
เทียบกับการไม่มี `GO TO` เลย — คำถามเหล่านี้คือสิ่งที่ขั้นตอนถัดไปในเอกสารนี้จะช่วยตอบทีละข้อ

---

## ขั้นตอนที่ 822: อ่านโครงสร้าง FD/Copybook เพื่อสร้าง Data Model ใหม่

### แนวคิด: Data Division คือแหล่งความจริงที่เชื่อถือได้ที่สุด

เมื่อไม่มีเอกสารออกแบบฐานข้อมูล/ไฟล์ สิ่งแรกที่นักวิเคราะห์ระบบ Legacy ควรทำคือไปอ่าน **`FILE SECTION`**
และ **`WORKING-STORAGE SECTION`** ของ `DATA DIVISION` โดยตรง เพราะ COBOL บังคับให้ทุกฟิลด์มี `PICTURE`
ชัดเจน ทำให้เราสามารถ "ย้อนวิศวกรรม" (reverse-engineer) โครงสร้างข้อมูลจริงของระบบได้แม่นยำ 100% แม้จะ
ไม่มีเอกสารใด ๆ เหลืออยู่เลย — นี่คือข้อดีอย่างหนึ่งของ COBOL ที่ภาษาที่ type ยืดหยุ่นกว่าไม่มี

### สกัด Record Layout จาก FD

จาก `FD ORD-FILE` ของ `LEGACYORD` เราอ่านได้ตรง ๆ ว่าไฟล์ `ORDERS.DAT` มีโครงสร้างต่อไปนี้ (ไม่ต้องเดา
เพราะ COBOL ระบุไว้ชัดเจนทุกไบต์):

| ฟิลด์ | PICTURE | ขนาด (ไบต์) | ตำแหน่งเริ่มต้น | ความหมายที่อนุมานได้ |
|---|---|---|---|---|
| `OR-CUST-ID` | `9(5)` | 5 | 1 | รหัสลูกค้า ตัวเลข 5 หลัก ไม่มีเครื่องหมาย |
| `OR-AMT` | `9(7)V99` | 9 | 6 | จำนวนเงินคำสั่งซื้อ ทศนิยม 2 ตำแหน่งแบบ implied |
| `OR-TIER` | `X(1)` | 1 | 15 | รหัสระดับลูกค้า 1 ตัวอักษร (ค่าที่ใช้จริงพบใน ขั้นตอนที่ 827) |
| `OR-REGION` | `X(1)` | 1 | 16 | รหัสภูมิภาค 1 ตัวอักษร |

รวมความยาว Record ทั้งหมด **16 ไบต์พอดี** — ตัวเลขนี้สำคัญมากเวลาต้องเขียนโปรแกรมใหม่ที่ต้องอ่านไฟล์
เดียวกัน (Part 084/085) เพราะถ้าความยาวไม่ตรง ข้อมูลจะเยื้องผิดตำแหน่งทันที

### สกัดสถานะของระบบจาก WORKING-STORAGE

การไล่อ่าน `WORKING-STORAGE SECTION` บอกเราว่าโปรแกรมนี้เก็บ "สถานะ" (state) อะไรไว้บ้างระหว่างทำงาน:

- `WS-CUST-BAL PIC 9(9)V99` — ยอดสะสมสุทธิของคำสั่งซื้อทั้งหมดที่ประมวลผลสำเร็จ (accumulator แบบไม่มี
  เครื่องหมายลบ แปลว่าธุรกิจนี้ไม่รองรับยอดติดลบ)
- `WS-REC-COUNT` / `WS-BAD-COUNT` — ตัวนับสำหรับสรุปผลการรัน (audit trail แบบพื้นฐาน)
- `88 REC-IS-VALID` / `88 REC-IS-INVALID` — บอกว่าระบบมีแนวคิดเรื่อง "ระเบียนที่ใช้ไม่ได้" อย่างชัดเจน
  (ดูรายละเอียดกฎการตรวจสอบใน ขั้นตอนที่ 828)

### สร้างแผนภาพ Data Model แบบข้อความ (ASCII ER-like Diagram)

เมื่อรวบรวมข้อมูลทั้งหมดแล้ว เราสามารถวาดแผนภาพสรุปสิ่งที่ระบบนี้ "รู้จัก" ได้ดังนี้ — นี่คือเอกสารที่
ควรถูกสร้างขึ้นใหม่ (เพราะของเดิมไม่มี) และเก็บไว้เป็นส่วนหนึ่งของผลงานการวิเคราะห์:

```
+-----------------------------------+
| ORD-REC (from ORDERS.DAT, 16 bytes)|
+-----------------------------------+
| OR-CUST-ID   PIC 9(5)              |
| OR-AMT       PIC 9(7)V99           |
| OR-TIER      PIC X(1)   <- see 827 |
| OR-REGION    PIC X(1)   <- see 827 |
+-----------------------------------+
              |
              | read one at a time,
              | no key relationship
              | enforced by the file
              v
+-----------------------------------+
| Program state (WORKING-STORAGE)    |
+-----------------------------------+
| WS-CUST-BAL   running total        |
| WS-REC-COUNT  audit counter        |
| WS-BAD-COUNT  audit counter        |
+-----------------------------------+
              |
              v
+-----------------------------------+
| RPT-LINE (to ORDRPT.TXT, 80 bytes) |
+-----------------------------------+
```

### ข้อควรระวัง

- อย่าไว้ใจชื่อฟิลด์เพียงอย่างเดียวโดยไม่ดู `PICTURE` — บางระบบ Legacy ตั้งชื่อผิดเพี้ยนจากความหมายจริง
  ไปแล้วเพราะมีการแก้ไขการใช้งานภายหลังโดยไม่เปลี่ยนชื่อตัวแปร (เช่น ฟิลด์ชื่อ `OLD-FLAG` แต่จริง ๆ ถูก
  นำมาใช้เก็บค่าใหม่ที่ไม่เกี่ยวกับความหมายเดิมแล้ว) — ต้องตรวจสอบทุกจุดที่ฟิลด์นั้นถูกใช้งานจริงด้วย
- `PICTURE` บอกแค่ "รูปแบบ" ของข้อมูล ไม่ได้บอก "ความหมายทางธุรกิจ" ทั้งหมด — ต้องอ่าน `PROCEDURE
  DIVISION` ควบคู่ไปด้วยเสมอเพื่อเข้าใจว่าค่าที่ถูกต้องของฟิลด์นั้นคืออะไรบ้าง (ดู ขั้นตอนที่ 827)

### แบบฝึกหัดที่ 822.1

**โจทย์**: จากการอ่าน `FD RPT-FILE` และย่อหน้า `PRINT-LINE` จงอธิบายว่าทำไม `RPT-LINE` ถึงถูกประกาศเป็น
`PIC X(80)` เพียงฟิลด์เดียว แทนที่จะแยกเป็นฟิลด์ย่อยเหมือน `ORD-REC`

**เฉลยแนวทาง**: เพราะ `RPT-LINE` เป็นไฟล์รายงานที่ออกแบบมาให้ **มนุษย์อ่าน** (human-readable report)
ไม่ใช่ไฟล์ที่โปรแกรมอื่นจะต้องมาอ่านค่ากลับเป็นฟิลด์ย่อยอีกที ต่างจาก `ORD-REC` ที่เป็นไฟล์ข้อมูลนำเข้าที่
ต้องมีโครงสร้างฟิลด์ชัดเจนเพื่อให้โปรแกรมประมวลผลค่าต่อได้ การเก็บเป็น `PIC X(80)` ก้อนเดียวแล้วใช้
`STRING` ประกอบข้อความ (ตามที่เห็นใน `PRINT-LINE`) จึงเหมาะสมกับวัตถุประสงค์ของไฟล์รายงานมากกว่า

---

## ขั้นตอนที่ 823: ไล่ตาม PERFORM Graph เพื่อสร้าง Call Graph ของโปรแกรม

### แนวคิด: Call Graph บอกเราว่าโค้ดส่วนไหน "ถูกเรียกจากที่ไหน"

หลังจากเข้าใจ**ข้อมูล**แล้ว ขั้นตอนถัดไปคือทำความเข้าใจ**การไหลของการทำงาน** (control flow) ในโปรแกรม
ที่มีหลายสิบย่อหน้า การไล่อ่านทีละบรรทัดจากบนลงล่างไม่ใช่วิธีที่มีประสิทธิภาพ — เราต้องการ **Call Graph**
คือแผนภาพที่แสดงว่าย่อหน้าไหน `PERFORM` ย่อหน้าไหนต่อบ้าง

### เขียนสคริปต์อัตโนมัติไล่หา PERFORM

แทนที่จะไล่อ่านด้วยตาทีละบรรทัด เราเขียนสคริปต์ shell/awk ง่าย ๆ สกัดความสัมพันธ์นี้ออกมาอัตโนมัติ:

```bash
#!/bin/bash
# perform_graph.sh - list which paragraphs each paragraph performs.
# Usage: ./perform_graph.sh file.cob
SRC="$1"
grep -En '^ {7}[A-Z0-9][A-Z0-9-]*\.$|PERFORM[[:space:]]+[A-Z0-9-]+' "$SRC" \
  | grep -v 'FILE-CONTROL' \
  | awk -F: '
    /^[0-9]+: {7}[A-Z0-9-]+\.$/ {
        line=$0
        sub(/^[0-9]+: */,"",line)
        sub(/\.$/,"",line)
        current=line
        next
    }
    /PERFORM/ {
        line=$0
        n=split(line, parts, "PERFORM")
        target=parts[2]
        gsub(/^[ \t]+/,"",target)
        split(target, w, /[ \t.]/)
        print current " --> " w[1]
    }
'
```

รันจริงกับ `legacyord.cob`:

```bash
chmod +x perform_graph.sh
./perform_graph.sh legacyord.cob
```

**ผลลัพธ์จริงที่ได้:**

```
MAIN-PARA --> INIT-RTN
MAIN-PARA --> PROCESS-LOOP
MAIN-PARA --> TERM-RTN
PROCESS-LOOP --> VALIDATE-ORDER
PROCESS-LOOP --> CALC-DISCOUNT
PROCESS-LOOP --> UPDATE-BALANCE
PROCESS-LOOP --> PRINT-LINE
```

### แปลผลลัพธ์เป็นแผนภาพต้นไม้

```
MAIN-PARA
  |-- INIT-RTN
  |-- PROCESS-LOOP (looped UNTIL EOF-REACHED)
  |     |-- VALIDATE-ORDER
  |     |-- CALC-DISCOUNT
  |     |-- UPDATE-BALANCE
  |     `-- PRINT-LINE
  `-- TERM-RTN
```

### สิ่งสำคัญที่สังเกตได้จากผลลัพธ์นี้ทันที

สังเกตว่า **`OLD-CALC-DISCOUNT-V1` ไม่ปรากฏในรายการนี้เลยแม้แต่บรรทัดเดียว** — ไม่มีย่อหน้าใดใน
โปรแกรม `PERFORM` เรียกมันเลย นี่คือหลักฐานชิ้นแรกที่บอกเราว่าย่อหน้านี้อาจเป็น **Dead Code**
(เราจะพิสูจน์เรื่องนี้อย่างเป็นทางการมากขึ้นใน ขั้นตอนที่ 825 เพราะ `PERFORM` ไม่ใช่วิธีเดียวที่พาการทำงาน
ไปยังย่อหน้าหนึ่งได้ — `GO TO` ก็ทำได้เช่นกัน ตามที่จะอธิบายใน ขั้นตอนที่ 824)

### ข้อควรระวัง

- สคริปต์ตัวอย่างนี้เป็นเครื่องมือแบบ **Text-based/Regex-based** ง่าย ๆ เหมาะกับการเรียนรู้และโปรแกรม
  ขนาดเล็ก-กลาง สำหรับระบบ Legacy ขนาดใหญ่ระดับหลายแสนบรรทัด องค์กรมักใช้เครื่องมือ Static Analysis
  เชิงพาณิชย์ (เช่นจาก Micro Focus, IBM) ที่แยกวิเคราะห์ไวยากรณ์ COBOL อย่างสมบูรณ์แทน regex ธรรมดา
  เพื่อความแม่นยำสูงกว่า แต่**หลักการพื้นฐาน**เหมือนกันทุกประการ
- สคริปต์นี้จับเฉพาะ `PERFORM name` (ไม่รวม `PERFORM name THRU name2` หรือ `PERFORM ... VARYING`
  แบบ inline) — เมื่อนำไปใช้กับโปรแกรมจริงควรปรับ regex ให้ครอบคลุมรูปแบบ `PERFORM` ทุกแบบที่ใช้ในองค์กร

### แบบฝึกหัดที่ 823.1

**โจทย์**: จงอธิบายว่าทำไม Call Graph เพียงอย่างเดียว (จาก `PERFORM`) ยังไม่เพียงพอที่จะสรุปว่าย่อหน้าใด
เป็น Dead Code ได้ 100%

**เฉลยแนวทาง**: เพราะ COBOL มีกลไกอื่นที่พาการทำงานไปยังย่อหน้าหนึ่งได้นอกเหนือจาก `PERFORM` เช่น
`GO TO` ตามที่เห็นใน `PROCESS-LOOP` ของ `LEGACYORD` (ซึ่งมี `GO TO PROCESS-LOOP-EXIT`) เครื่องมือที่
มองหาเฉพาะคำว่า `PERFORM` อาจรายงานย่อหน้าที่ถูกเรียกด้วย `GO TO` ว่าเป็น Dead Code อย่างผิดพลาด
(False Positive) การวิเคราะห์ที่ถูกต้องต้องนับทั้ง `PERFORM` และ `GO TO` ร่วมกันเสมอ

---

## ขั้นตอนที่ 824: ติดตาม GO TO และรูปแบบควบคุมการไหลที่ซับซ้อน

### ทำไม GO TO ถึงเป็นเรื่องที่ต้องระวังเป็นพิเศษ

`GO TO` เป็นคำสั่งที่ COBOL ยุคก่อน Structured Programming (ก่อน COBOL-85 ตามที่กล่าวถึงใน Part 001
ขั้นตอนที่ 6) ใช้กันอย่างแพร่หลาย มันพาการทำงานกระโดดไปยังย่อหน้าอื่นโดยตรง **โดยไม่สนใจขอบเขตของ
`PERFORM` ที่กำลังทำงานอยู่** ทำให้การไล่ตามเส้นทางการทำงานด้วยตาเปล่ายากขึ้นมาก และเป็นสาเหตุสำคัญที่ทำ
ให้โค้ด Legacy จำนวนมาก "อ่านยาก" กว่าที่ควรจะเป็น

### วิเคราะห์ GO TO ตัวเดียวใน LEGACYORD ทีละขั้นตอน

กลับไปดูย่อหน้า `PROCESS-LOOP`:

```cobol
       PROCESS-LOOP.
           READ ORD-FILE
               AT END
                   SET EOF-REACHED TO TRUE
               NOT AT END
                   PERFORM VALIDATE-ORDER
                   IF REC-IS-VALID
                       PERFORM CALC-DISCOUNT
                       PERFORM UPDATE-BALANCE
                       PERFORM PRINT-LINE
                   ELSE
                       ADD 1 TO WS-BAD-COUNT
                       GO TO PROCESS-LOOP-EXIT
                   END-IF
           END-READ.
       PROCESS-LOOP-EXIT.
           EXIT.
```

**คำถามสำคัญ**: `GO TO PROCESS-LOOP-EXIT` ปลอดภัยหรือไม่? คำตอบขึ้นอยู่กับว่าย่อหน้านี้ถูก `PERFORM`
เรียกใช้งานอย่างไรที่ `MAIN-PARA`:

```cobol
           PERFORM PROCESS-LOOP THRU PROCESS-LOOP-EXIT
               UNTIL EOF-REACHED
```

สังเกตคำว่า **`THRU PROCESS-LOOP-EXIT`** — นี่คือกุญแจสำคัญ: มันบอกคอมไพเลอร์ว่า "ขอบเขตของการ
`PERFORM` ครั้งนี้ครอบคลุมตั้งแต่ `PROCESS-LOOP` ไปจนถึง `PROCESS-LOOP-EXIT`" ดังนั้นเมื่อ `GO TO
PROCESS-LOOP-EXIT` ทำงาน มันแค่กระโดดไปยัง**จุดสิ้นสุดของขอบเขตเดียวกัน** แล้วควบคุมจะกลับไปที่
`PERFORM` ตามปกติ (เหมือนกับใช้ `CONTINUE`/`break` ในภาษาสมัยใหม่เพื่อออกจากรอบการวนซ้ำปัจจุบัน)

### พิสูจน์ด้วยการทดลองจริง: ถ้าไม่มี THRU จะเกิดอะไรขึ้น

นี่คือการทดลองที่สำคัญมากสำหรับผู้เรียนที่วิเคราะห์ระบบ Legacy: ลองเปลี่ยนบรรทัดใน `MAIN-PARA` เป็น
`PERFORM PROCESS-LOOP UNTIL EOF-REACHED` (ตัด `THRU PROCESS-LOOP-EXIT` ออก) แล้วคอมไพล์และรันใหม่กับ
ข้อมูลชุดเดิม:

```
RECORDS READ: 00004
RECORDS REJECTED: 00001
FINAL BALANCE: 000020030.00
```

ผลลัพธ์ **ผิดไปจากเดิมทันที** (`FINAL BALANCE` ควรเป็น `39630.00` ไม่ใช่ `20030.00`, `RECORDS READ`
ควรเป็น `5` ไม่ใช่ `4`) เพราะเมื่อไม่มี `THRU` ขอบเขตของ `PERFORM` ครอบคลุมแค่ย่อหน้า `PROCESS-LOOP`
เพียงย่อหน้าเดียว การ `GO TO PROCESS-LOOP-EXIT` จึงกระโดดออก**นอกขอบเขต** ของ `PERFORM` ไปยังย่อหน้าที่
ไม่ได้ถูกครอบคลุมไว้ แล้ว**ตกลงมา (fall through)** ต่อไปยังย่อหน้าถัดไปตามลำดับกายภาพในซอร์สโค้ด
(`VALIDATE-ORDER`) โดยไม่ได้ตั้งใจ ทำให้โปรแกรมประมวลผลย่อหน้าที่ไม่ควรถูกเรียกซ้อนเข้าไปอีก และวนลูป
ผิดเพี้ยนไปหมด — นี่คือตัวอย่างจริงว่าทำไม `GO TO` ในโค้ด Legacy ถึงอันตรายถ้าไม่เข้าใจขอบเขต `PERFORM
... THRU` ที่ครอบมันอยู่

### ข้อควรระวัง

- เมื่อพบ `GO TO` ในโค้ด Legacy ให้**ไล่หาก่อนเสมอ**ว่าย่อหน้าที่มันอยู่ถูก `PERFORM` เรียกแบบมี `THRU`
  หรือไม่ มิฉะนั้นการวิเคราะห์ควบคุมการไหลจะผิดพลาดได้ง่ายมาก
- อย่าลบหรือแก้ไข `GO TO` ในขั้นตอนการวิเคราะห์ (analysis) — งานแก้ไขโครงสร้างควบคุมการไหลควรทำในขั้นตอน
  Refactoring (Part 081) หลังจากเข้าใจพฤติกรรมเดิมอย่างถ่องแท้และมี Golden Master Test (Part 085 ขั้นตอนที่
  842) รองรับไว้แล้วเท่านั้น

### แบบฝึกหัดที่ 824.1

**โจทย์**: จงอธิบายว่าทำไมการทดลอง "ตัด `THRU PROCESS-LOOP-EXIT` ออก" ในขั้นตอนนี้ถึงเป็นวิธีวิเคราะห์
ที่ปลอดภัย ทั้งที่ Part นี้เพิ่งเตือนว่าไม่ควรแก้โค้ด Legacy ก่อนเข้าใจมัน

**เฉลยแนวทาง**: เพราะการทดลองนี้ทำใน**สำเนาแยกต่างหาก** (เช่นสภาพแวดล้อมทดสอบ/scratch directory) ไม่ใช่
ระบบ Production จริง และมีจุดประสงค์เพื่อ**ยืนยันความเข้าใจ** ไม่ใช่เพื่อส่งมอบเป็นโค้ดจริง หลักการ
"ห้ามแก้ก่อนเข้าใจ" หมายถึงห้ามแก้ไข**ระบบที่ใช้งานจริง**โดยไม่มีการทดสอบรองรับ แต่การทดลองแก้ไขชั่วคราว
ในสภาพแวดล้อมที่ปลอดภัยเพื่อพิสูจน์สมมติฐานเกี่ยวกับพฤติกรรมของโค้ด เป็นเทคนิคมาตรฐานของ Reverse
Engineering ที่ช่วยให้เข้าใจโค้ดได้เร็วและแม่นยำขึ้นมาก

---

## ขั้นตอนที่ 825: หา Dead Code ด้วยเทคนิคอัตโนมัติ

### แนวคิด: Dead Code คือย่อหน้าที่ไม่มีทาง "เข้าถึง" ได้เลย

จากขั้นตอนที่ 823-824 เรารู้แล้วว่าย่อหน้าหนึ่งจะถูกเข้าถึงได้ผ่าน `PERFORM` หรือ `GO TO` เท่านั้น
(นอกเหนือจากการ fall-through ตามลำดับกายภาพ ซึ่งไม่นับเพราะ `PERFORM` ของย่อหน้าเดี่ยวจะไม่ fall-through
ออกนอกย่อหน้าของตัวเอง) ดังนั้นย่อหน้าที่ **ไม่มีทั้ง `PERFORM` และ `GO TO` ใด ๆ ในทั้งไฟล์ชี้มาหามันเลย**
คือผู้ต้องสงสัยอันดับหนึ่งว่าเป็น **Dead Code**

### เขียนสคริปต์ค้นหา Dead Paragraph

```bash
#!/bin/bash
# find_dead_paragraphs.sh - naive but effective dead-paragraph finder
# for fixed-format COBOL. Usage: ./find_dead_paragraphs.sh file.cob
SRC="$1"

# Step 1: list paragraph names. A paragraph header in fixed format is a
# line that starts in Area A (columns 8-11) with a name followed by a
# period, and is not a division/section header.
PARAS=$(grep -E '^ {7}[A-Z0-9][A-Z0-9-]*\.$' "$SRC" | sed 's/\.$//' | sed 's/^ *//')

echo "Paragraphs found:"
echo "$PARAS" | sed 's/^/  - /'
echo ""
echo "Paragraphs with NO incoming PERFORM/GO TO (dead-code candidates):"
while IFS= read -r P; do
    [ -z "$P" ] && continue
    HITS=$(grep -Ec "(PERFORM|GO TO)[[:space:]]+$P([[:space:]]|\.|$)" "$SRC")
    if [ "$HITS" -eq 0 ]; then
        echo "  -> $P (0 references)"
    fi
done <<< "$PARAS"
```

รันจริงกับ `legacyord.cob`:

```bash
chmod +x find_dead_paragraphs.sh
./find_dead_paragraphs.sh legacyord.cob
```

**ผลลัพธ์จริงที่ได้:**

```
Paragraphs found:
  - FILE-CONTROL
  - MAIN-PARA
  - INIT-RTN
  - PROCESS-LOOP
  - PROCESS-LOOP-EXIT
  - VALIDATE-ORDER
  - CALC-DISCOUNT
  - OLD-CALC-DISCOUNT-V1
  - UPDATE-BALANCE
  - PRINT-LINE
  - TERM-RTN

Paragraphs with NO incoming PERFORM/GO TO (dead-code candidates):
  -> FILE-CONTROL (0 references)
  -> MAIN-PARA (0 references)
  -> OLD-CALC-DISCOUNT-V1 (0 references)
```

### อ่านผลลัพธ์อย่างมีวิจารณญาณ — เครื่องมือช่วยได้แต่ไม่แทนที่การคิดของมนุษย์

ผลลัพธ์นี้มีทั้ง **True Positive** และ **False Positive** ที่ต้องแยกแยะเอง:

1. **`FILE-CONTROL`** — False Positive ชัดเจน สคริปต์แบบ regex ง่าย ๆ นี้จับ `FILE-CONTROL.` ที่เป็น
   หัวข้อใน `ENVIRONMENT DIVISION` ผิดว่าเป็นชื่อย่อหน้า ทั้งที่จริงมันไม่ใช่ย่อหน้าใน `PROCEDURE
   DIVISION` เลย — นี่คือตัวอย่างว่าทำไมเครื่องมือวิเคราะห์แบบ regex ถึงต้องใช้อย่างระมัดระวัง
2. **`MAIN-PARA`** — False Positive เช่นกัน แต่ด้วยเหตุผลต่างกัน: มันคือย่อหน้า**แรก**ของ `PROCEDURE
   DIVISION` ซึ่งถูกเรียกโดยอัตโนมัติเมื่อโปรแกรมเริ่มทำงาน (runtime เรียกมันโดยตรง ไม่มีใครใน COBOL
   source ต้อง `PERFORM` มันอีก) จึงเป็น "จุดเริ่มต้น" (root) ของ Call Graph เสมอ ไม่ใช่ Dead Code
3. **`OLD-CALC-DISCOUNT-V1`** — **True Positive** นี่คือ Dead Code จริง ยืนยันได้จากทั้งขั้นตอนที่ 823
   (ไม่มีใน Call Graph จาก `PERFORM`) และขั้นตอนนี้ (ไม่มี `GO TO` มาหาเช่นกัน) — ปลอดภัยที่จะเสนอให้ลบ
   ทิ้งในขั้นตอน Refactoring ต่อไป (แต่ต้องมี Golden Master Test คุ้มครองไว้ก่อนเสมอ ตามที่จะสอนใน
   Part 085 ขั้นตอนที่ 842)

### ข้อควรระวัง

- เครื่องมือวิเคราะห์อัตโนมัติแบบใดก็ตาม (ไม่ว่าจะ regex ธรรมดาแบบนี้ หรือเครื่องมือเชิงพาณิชย์ราคาแพง)
  ล้วนมีโอกาสเกิด False Positive/Negative ได้เสมอ **ต้องตรวจสอบผลลัพธ์ด้วยความเข้าใจบริบทของมนุษย์เสมอ**
  ก่อนตัดสินใจลบโค้ดใด ๆ ออกจากระบบจริง
- ห้ามลบย่อหน้าที่สงสัยว่าเป็น Dead Code ทันทีที่พบ — ควรบันทึกไว้เป็นรายการ "ผู้ต้องสงสัย" แล้วยืนยัน
  ด้วยวิธีอื่นเพิ่มเติม (เช่น ค้นหาในระบบ Version Control ทั้งหมดของบริษัทว่ามีโปรแกรมอื่นเรียกใช้ผ่าน
  Dynamic `CALL` จากชื่อที่สร้างขึ้นระหว่างรันหรือไม่ ตามเทคนิค Part 043) ก่อนลบออกจริง

### แบบฝึกหัดที่ 825.1

**โจทย์**: หากเปลี่ยนสคริปต์ `find_dead_paragraphs.sh` ให้เพิ่มเงื่อนไขยกเว้นชื่อ `MAIN-PARA` และ
`FILE-CONTROL` ออกจากผลลัพธ์เสมอ จะทำให้เครื่องมือนี้แม่นยำขึ้นหรือไม่ เพราะเหตุใด

**เฉลยแนวทาง**: แม่นยำขึ้นสำหรับกรณีเฉพาะนี้ (เพราะทั้งสองชื่อนี้เป็น False Positive ที่ยืนยันได้แน่นอน)
แต่ยังไม่ใช่วิธีแก้ปัญหาที่สมบูรณ์ เพราะชื่อย่อหน้าแรกของ `PROCEDURE DIVISION` ในแต่ละโปรแกรมไม่จำเป็น
ต้องชื่อ `MAIN-PARA` เสมอไป (อาจชื่อ `0000-MAIN`, `START-HERE` ฯลฯ) การแก้ปัญหาที่แข็งแรงกว่าคือให้สคริปต์
ตรวจจับ**ย่อหน้าแรกสุด**ในไฟล์โดยอัตโนมัติ (ตำแหน่งแรกหลัง `PROCEDURE DIVISION.`) แล้วยกเว้นมันเสมอ แทนที่
จะ hardcode ชื่อเฉพาะเจาะจงไว้ — และควรกรอง `FILE-CONTROL`/`DATA DIVISION`/ชื่อหัวข้อ Division/Section
อื่น ๆ ออกจากรายการชื่อย่อหน้าตั้งแต่ขั้นตอนแรกด้วยเช่นกัน

---

## ขั้นตอนที่ 826: Cross-Reference Listing — ใช้เครื่องมือคอมไพเลอร์ช่วยวิเคราะห์

### แนวคิด: Cross-Reference บอกว่าตัวแปร/ย่อหน้าแต่ละตัวถูกใช้ที่ไหนบ้าง

นอกจากสคริปต์ที่เราเขียนเอง GnuCOBOL เองก็มีความสามารถสร้างรายงาน Cross-Reference ในตัว มาลองทดสอบว่า
ใช้งานได้จริงหรือไม่ในสภาพแวดล้อมของเรา

### ทดสอบ cobc -Xref

```bash
cobc -x -Xref -o legacyord_xref legacyord.cob
```

**ผลลัพธ์จริงที่ได้ (ข้อความ error จริง ไม่ใช่การจำลอง):**

```
cobc: error: -Xref option requires a listing file
```

เมื่อลองเพิ่มไฟล์ listing (`-t`) ตามที่ error แนะนำโดยนัย และตรวจสอบดูให้แน่ใจ พบว่าตัวเลือก `-Xref` ใน
GnuCOBOL เวอร์ชันนี้ต้องพึ่งพาโปรแกรมภายนอกชื่อ **`cobxref`** (ตามที่ `cobc --help` ระบุไว้ตรง ๆ ว่า
`"generate cross reference through 'cobxref' (V. Coen's 'cobxref' must be in path)"`) และเครื่องที่ใช้
เตรียมเอกสารนี้**ไม่ได้ติดตั้ง `cobxref` ไว้** — นี่คือสถานการณ์จริงที่พบได้บ่อยมากในงาน Legacy Analysis:
เครื่องมือที่เอกสารกล่าวถึงไม่ได้แปลว่าจะพร้อมใช้งานในทุกสภาพแวดล้อมเสมอไป ต้องตรวจสอบเองก่อนเชื่อ

### ทางเลือกที่ใช้งานได้จริงในสภาพแวดล้อมนี้: -t (Program Listing)

ตัวเลือก `-t <file>` ของ `cobc` ใช้งานได้โดยไม่ต้องพึ่งเครื่องมือภายนอกใด ๆ และให้ผลลัพธ์เป็น "Program
Listing" — สำเนาซอร์สโค้ดพร้อมเลขบรรทัดในรูปแบบมาตรฐานที่ใช้ตรวจสอบร่วมกับทีมได้:

```bash
cobc -x -t legacyord.lst -o legacyord legacyord.cob
head -30 legacyord.lst
```

**ผลลัพธ์บางส่วนที่ได้จริง:**

```
GnuCOBOL 4.0-early-dev. legacyord.cob        Mon Sep 28 19:18:04 2026  Page 0001

LINE    PG/LN  A...B............................................................

000001         IDENTIFICATION DIVISION.
000002         PROGRAM-ID. LEGACYORD.
000003         AUTHOR. UNKNOWN-LEGACY-TEAM.
...
```

### บทเรียนสำคัญจากขั้นตอนนี้: อย่าเชื่อว่าเครื่องมือจะพร้อมใช้เสมอ

นี่คือบทเรียนที่มีค่ามากพอ ๆ กับตัวเทคนิคเอง: เมื่อทำงานกับระบบ Legacy ในองค์กรจริง เรามักเจอเอกสารเก่าที่
กล่าวถึงเครื่องมือ/ระบบที่ถูก decommission ไปแล้ว หรือเครื่องมือที่ต้องขอสิทธิ์ติดตั้งเพิ่มเติมซึ่งอาจใช้
เวลานาน — วินัยที่ดีคือ**ทดสอบเครื่องมือจริงก่อนวางแผนงานที่พึ่งพามัน** และเตรียมทางเลือกสำรอง (เช่น
สคริปต์ที่เราเขียนเองใน ขั้นตอนที่ 823/825) ไว้เสมอ แทนที่จะติดขัดเมื่อพบว่าเครื่องมือหลักใช้งานไม่ได้

### ข้อควรระวัง

- ก่อนวางแผนกระบวนการ Legacy Analysis ขนาดใหญ่โดยอ้างอิงเครื่องมือใดเครื่องมือหนึ่ง ให้ทดสอบเครื่องมือ
  นั้นกับไฟล์ตัวอย่างเล็ก ๆ ก่อนเสมอ เพื่อยืนยันว่าใช้งานได้จริงในสภาพแวดล้อมที่มีอยู่
- ตัวเลือก `-t`/`-T` ของ `cobc` ให้ "Program Listing" ไม่ใช่ "Cross-Reference" ที่สมบูรณ์แบบ (ไม่มี
  ตารางสรุปว่าตัวแปรแต่ละตัวถูกใช้ที่บรรทัดใดบ้างเหมือน `cobxref` หรือเครื่องมือเชิงพาณิชย์) — สำหรับ
  Cross-Reference ระดับตัวแปรที่ครบถ้วนกว่านี้ ต้องพึ่งพาสคริปต์เสริม (เช่นใช้ `grep -n` ค้นหาทุกจุดที่
  ตัวแปรหนึ่งถูกอ้างอิง) หรือเครื่องมือภายนอกที่ติดตั้งเพิ่มเติมให้ครบ

### แบบฝึกหัดที่ 826.1

**โจทย์**: จงเขียนคำสั่ง `grep` หนึ่งบรรทัดที่ใช้หาว่าตัวแปร `WS-DPCT` ถูกอ้างอิงที่บรรทัดใดบ้างใน
`legacyord.cob` เพื่อใช้เป็น Cross-Reference แบบง่ายทดแทน `cobxref` ที่ไม่มีในเครื่อง

**เฉลย**:

```bash
grep -n 'WS-DPCT' legacyord.cob
```

คำสั่งนี้แสดงทุกบรรทัดที่มีคำว่า `WS-DPCT` ปรากฏอยู่ พร้อมเลขบรรทัด ทำให้เห็นภาพรวมได้ทันทีว่าตัวแปรนี้
ถูกกำหนดค่าที่ไหนบ้าง (`MOVE ... TO WS-DPCT`) และถูกอ่านค่าไปใช้ที่ไหนบ้าง (`COMPUTE ... WS-DPCT`,
`STRING ... WS-DPCT`) — เป็น Cross-Reference แบบพื้นฐานที่สุดแต่ใช้งานได้ทันทีโดยไม่ต้องติดตั้งอะไรเพิ่ม

---

## ขั้นตอนที่ 827: สกัดกฎทางธุรกิจจาก IF/EVALUATE ที่ซับซ้อน

### แนวคิด: กฎธุรกิจที่แท้จริงอยู่ใน PROCEDURE DIVISION ไม่ใช่ในเอกสาร

ขั้นตอนที่สำคัญและยากที่สุดของ Legacy Analysis คือการอ่าน `IF`/`EVALUATE` ที่ซ้อนกันหลายชั้น แล้วแปลง
มันให้เป็น**ตารางการตัดสินใจ (Decision Table)** ที่มนุษย์อ่านเข้าใจง่าย โดยไม่พลาดเงื่อนไขใด ๆ ไป — นี่
คืองานที่ต้องอาศัยความรอบคอบสูงมาก เพราะการอ่านตกหล่นแม้เพียงเงื่อนไขเดียวอาจทำให้ระบบใหม่ (Part 084/085)
ทำงานผิดจากของเดิม

### ไล่อ่านย่อหน้า CALC-DISCOUNT ทีละชั้น

กลับไปดูย่อหน้าที่ซับซ้อนที่สุดของ `LEGACYORD`:

```cobol
       CALC-DISCOUNT.
           EVALUATE OR-TIER
               WHEN "P"
                   IF OR-AMT >= 10000.00
                       MOVE 0.150 TO WS-DPCT
                   ELSE
                       IF OR-AMT >= 5000.00
                           MOVE 0.100 TO WS-DPCT
                       ELSE
                           MOVE 0.050 TO WS-DPCT
                       END-IF
                   END-IF
      *> (WHEN "G" and WHEN "R" follow the same nested pattern)
           END-EVALUATE

           IF OR-REGION = "X"
               ADD 0.020 TO WS-DPCT
               IF WS-DPCT > 0.200
                   MOVE 0.200 TO WS-DPCT
               END-IF
           END-IF
```

การไล่อ่านทีละเงื่อนไข: ก่อนอื่นแยกตามค่า `OR-TIER` (3 กรณี: P, G, R) แล้วในแต่ละกรณีแยกตามช่วงของ
`OR-AMT` (3 ช่วง: >=10000, >=5000 แต่ <10000, และ <5000) รวมเป็น 9 กรณีพื้นฐาน บวกกับกฎเสริมเรื่อง
`OR-REGION = "X"` ที่ใช้ได้กับทุกกรณี (บวกเพิ่ม 2% แต่ไม่เกิน 20% รวม)

### แปลงเป็น Decision Table (เอกสารที่ควรสร้างขึ้นใหม่)

| Tier (`OR-TIER`) | เงื่อนไขยอดสั่งซื้อ (`OR-AMT`) | ส่วนลดพื้นฐาน | ส่วนลดถ้า Region = "X" (Export) |
|---|---|---|---|
| P (Platinum) | >= 10,000.00 | 15.0% | 17.0% |
| P (Platinum) | 5,000.00 – 9,999.99 | 10.0% | 12.0% |
| P (Platinum) | < 5,000.00 | 5.0% | 7.0% |
| G (Gold) | >= 10,000.00 | 10.0% | 12.0% |
| G (Gold) | 5,000.00 – 9,999.99 | 7.0% | 9.0% |
| G (Gold) | < 5,000.00 | 3.0% | 5.0% |
| R (Regular) | >= 10,000.00 | 5.0% | 7.0% |
| R (Regular) | 5,000.00 – 9,999.99 | 2.0% | 4.0% |
| R (Regular) | < 5,000.00 | 0.0% | 2.0% |

**กฎเสริม**: ส่วนลดรวมทุกกรณีถูกจำกัดเพดานสูงสุดไว้ที่ **20%** เสมอ (พบได้จากเงื่อนไข `IF WS-DPCT >
0.200 MOVE 0.200 TO WS-DPCT`) — สังเกตว่าเพดานนี้ไม่มีผลกับกรณีในตารางข้างต้นเลยสักแถว (แถวสูงสุดคือ
17% ยังไม่ถึงเพดาน) ซึ่งอาจแปลว่ากฎนี้ถูกเขียนไว้เผื่ออนาคต หรือเคยมีระดับลูกค้า/ส่วนลดที่สูงกว่านี้ในอดีต
ที่ถูกลบออกไปแล้ว — เป็นคำถามที่ควรบันทึกไว้ถามผู้เกี่ยวข้องฝ่ายธุรกิจต่อไป

### ข้อควรระวัง

- ตารางการตัดสินใจที่สร้างขึ้นต้อง**ทดสอบยืนยันกับโปรแกรมจริง**เสมอ (ไม่ใช่แค่อ่านโค้ดแล้วสรุปเอง) —
  วิธีที่ดีที่สุดคือป้อนข้อมูลทดสอบที่ครอบคลุมทุกแถวของตาราง แล้วเปรียบเทียบผลลัพธ์จริงจากโปรแกรมกับค่าที่
  คำนวณจากตารางที่เราสร้างขึ้น ตามที่สาธิตไว้แล้วในขั้นตอนที่ 821 (ตรวจสอบ `RECORDS READ`, `FINAL
  BALANCE` ตรงกับสูตรที่คำนวณด้วยมือ)
- ระวังเงื่อนไขขอบเขต (boundary condition) เช่น `OR-AMT >= 5000.00` กับ `OR-AMT > 5000.00` มีความหมาย
  ต่างกัน แม้ดูคล้ายกันมากในสายตา การอ่านผิดพลาดตรงเครื่องหมาย `>=` กับ `>` เพียงตัวเดียวสามารถทำให้ระบบ
  ใหม่คำนวณผลลัพธ์ผิดพลาดในกรณีขอบเขตพอดี (exactly at the boundary) ได้ — Part 085 ขั้นตอนที่ 846 จะ
  แสดงตัวอย่างจริงว่าบั๊กแบบนี้เกิดขึ้นได้อย่างไรและ Unit Test ช่วยจับมันได้อย่างไร

### แบบฝึกหัดที่ 827.1

**โจทย์**: ลูกค้าระดับ Gold (`OR-TIER = "G"`) สั่งซื้อสินค้ามูลค่า `OR-AMT = 5000.00` พอดี (ไม่ใช่ Export)
จากตาราง Decision Table ที่สร้างไว้ ส่วนลดที่ควรได้รับคือกี่เปอร์เซ็นต์ และทำไม

**เฉลย**: 7.0% เพราะเงื่อนไข `OR-AMT >= 5000.00` เป็นจริงเมื่อ `OR-AMT` เท่ากับ 5000.00 พอดี (`>=` รวม
กรณีเท่ากันด้วย) ทำให้ตกอยู่ในแถว "5,000.00 – 9,999.99" ไม่ใช่แถว "< 5,000.00" — นี่คือตัวอย่างที่แสดง
ให้เห็นว่าทำไมข้อควรระวังเรื่องขอบเขต `>=` กับ `>` ในหัวข้อนี้ถึงสำคัญมาก

---

## ขั้นตอนที่ 828: วิเคราะห์กฎการตรวจสอบข้อมูล (Data Validation Rules)

### แนวคิด: กฎ Validation บอกเราว่าระบบ "คาดหวัง" อะไรจากข้อมูลนำเข้า

นอกจากกฎการคำนวณแล้ว ย่อหน้า `VALIDATE-ORDER` ยังบอกเราถึงกฎเกณฑ์ที่ระบบใช้ตัดสินว่าข้อมูลใดถือว่า
"ใช้งานได้" — ข้อมูลนี้สำคัญมากสำหรับการออกแบบระบบใหม่ (Part 084/085) เพราะระบบใหม่ต้องปฏิเสธข้อมูลแบบ
เดียวกันกับระบบเดิม มิฉะนั้นจะเกิดพฤติกรรมที่ไม่สอดคล้องกัน

### ไล่อ่าน VALIDATE-ORDER

```cobol
       VALIDATE-ORDER.
           SET REC-IS-VALID TO TRUE
           ADD 1 TO WS-REC-COUNT
           IF OR-AMT = ZERO
               SET REC-IS-INVALID TO TRUE
           END-IF
           IF OR-TIER NOT = "P" AND OR-TIER NOT = "G"
                   AND OR-TIER NOT = "R"
               SET REC-IS-INVALID TO TRUE
           END-IF.
```

สกัดกฎได้ 2 ข้อ:

1. **กฎที่ 1**: `OR-AMT` ต้องไม่เป็นศูนย์ — คำสั่งซื้อมูลค่า 0.00 ถือว่าไม่ถูกต้อง (มีเหตุผลทางธุรกิจที่
   สมเหตุสมผล: คำสั่งซื้อมูลค่าศูนย์อาจเป็นข้อมูลผิดพลาดจากต้นทาง ไม่ใช่ธุรกรรมจริง)
2. **กฎที่ 2**: `OR-TIER` ต้องเป็นหนึ่งใน `"P"`, `"G"`, หรือ `"R"` เท่านั้น — ค่าอื่นใดถือว่าไม่ถูกต้อง
   (ยืนยันขอบเขตของค่าที่เป็นไปได้ของฟิลด์นี้ ซึ่งไม่ได้ถูกบังคับด้วย `PICTURE` เพราะ `PIC X(1)` รับ
   ตัวอักษรอะไรก็ได้ — กฎเรื่องค่าที่ยอมรับได้ถูกบังคับด้วยโค้ดใน `PROCEDURE DIVISION` เท่านั้น)

### สังเกต: ไม่มีการตรวจสอบ OR-REGION เลย

สิ่งที่น่าสนใจไม่แพ้กฎที่มีอยู่คือ **กฎที่ไม่มีอยู่**: `VALIDATE-ORDER` ไม่ได้ตรวจสอบค่า `OR-REGION`
เลยแม้แต่น้อย ทั้งที่ `CALC-DISCOUNT` ใช้เงื่อนไข `IF OR-REGION = "X"` อย่างชัดเจน คำถามที่ควรบันทึกไว้คือ:
ถ้า `OR-REGION` เป็นค่าอื่นที่ไม่ใช่ `"X"` (เช่นค่าว่างหรือค่าขยะ) ระบบจะถือว่าเป็น "ไม่ใช่ Export" โดย
อัตโนมัติเสมอ — นี่อาจเป็นพฤติกรรมที่ตั้งใจ (ค่าอะไรก็ตามที่ไม่ใช่ "X" ถือว่าในประเทศ) หรืออาจเป็นช่องโหว่
ที่ไม่มีใครสังเกตเห็นมาก่อนก็ได้ ต้องบันทึกเป็นคำถามเปิด (open question) สำหรับฝ่ายธุรกิจ

### ตรวจสอบผลกระทบจริงด้วยข้อมูลทดสอบ

จากข้อมูลตัวอย่างในขั้นตอนที่ 821 บรรทัดที่ 3 (`10003000000000GR`) มี `OR-AMT = 0` จึงถูกปฏิเสธ ตรงกับ
`RECORDS REJECTED: 00001` ในผลลัพธ์จริง — ยืนยันว่ากฎที่ 1 ทำงานตามที่วิเคราะห์ไว้จริง

### ข้อควรระวัง

- อย่าสรุปกฎ Validation จากการอ่านโค้ดเพียงอย่างเดียวโดยไม่ทดสอบยืนยัน — ควรเตรียมข้อมูลทดสอบที่ครอบคลุม
  ทุกกรณี "ขอบ" ของกฎ (เช่น `OR-TIER = "X"` ที่ไม่ใช่ค่าที่ยอมรับ, `OR-AMT = 0.01` ที่เกือบเป็นศูนย์)
  แล้วรันจริงเพื่อยืนยันว่าพฤติกรรมตรงกับที่วิเคราะห์ไว้
- การไม่มีกฎตรวจสอบ (absence of validation) ก็เป็นข้อมูลสำคัญพอ ๆ กับการมีกฎ — ต้องบันทึกไว้เป็นส่วนหนึ่ง
  ของเอกสารวิเคราะห์เสมอ เพราะระบบใหม่ที่เพิ่มการตรวจสอบเข้าไปโดยไม่ได้ตั้งใจ (เช่น มือใหม่คิดว่าควร
  ตรวจสอบ `OR-REGION` ด้วย "เพื่อความถูกต้อง") อาจทำให้ระบบใหม่ปฏิเสธข้อมูลที่ระบบเดิมเคยยอมรับ กลายเป็น
  พฤติกรรมที่เปลี่ยนไปโดยไม่ได้ตั้งใจ

### แบบฝึกหัดที่ 828.1

**โจทย์**: จงออกแบบชุดข้อมูลทดสอบ 1 บรรทัดที่จะช่วยพิสูจน์ว่ากฎที่ 2 (`OR-TIER` ต้องเป็น P/G/R เท่านั้น)
ทำงานถูกต้องจริง แล้วอธิบายผลลัพธ์ที่คาดว่าจะได้

**เฉลยแนวทาง**: เตรียมบรรทัดเช่น `10006000500000ZR` (cust=10006, amt=5000.00, tier="Z" ซึ่งไม่ใช่ P/G/R,
region="R") นำไปทดสอบกับโปรแกรม คาดว่า `VALIDATE-ORDER` จะ set `REC-IS-INVALID` เพราะ tier ไม่ตรงกับ
เงื่อนไขทั้งสาม ทำให้ `WS-BAD-COUNT` เพิ่มขึ้น 1 และระเบียนนี้จะไม่ปรากฏในรายงาน `ORDRPT.TXT` เลย — ตรงกับ
พฤติกรรมเดียวกับที่เกิดกับระเบียน `10003` ที่มี `OR-AMT = 0` ในขั้นตอนที่ 821

---

## ขั้นตอนที่ 829: เขียนเอกสารสรุปผลการวิเคราะห์ (Reverse-Engineered Design Document)

### แนวคิด: ผลลัพธ์ของ Legacy Analysis ต้องถูกบันทึกไว้เป็นลายลักษณ์อักษร

การวิเคราะห์ทั้งหมดในขั้นตอนที่ 821-828 จะไร้ค่าถ้าไม่ถูกบันทึกไว้เป็นเอกสารที่คนอื่น (หรือแม้แต่ตัวเราเอง
ในอีก 6 เดือนข้างหน้า) สามารถอ่านและใช้อ้างอิงต่อได้ นี่คือขั้นตอนที่มักถูกมองข้ามในทีมที่เร่งรีบ แต่เป็น
ขั้นตอนที่ให้ผลตอบแทนคุ้มค่าที่สุดในระยะยาว

### โครงสร้างเอกสารมาตรฐานที่แนะนำ

ต่อไปนี้คือโครงเอกสารสรุปผลการวิเคราะห์ระบบ `LEGACYORD` ที่รวบรวมทุกสิ่งที่ค้นพบจากขั้นตอนที่ 821-828
เข้าด้วยกัน — รูปแบบนี้สามารถใช้เป็นแม่แบบสำหรับการวิเคราะห์ระบบ Legacy อื่น ๆ ได้ทันที:

```
=====================================================
REVERSE-ENGINEERED DESIGN DOCUMENT: LEGACYORD.cob
=====================================================

1. PURPOSE (inferred)
   Reads order records, applies a tiered customer discount plus
   an export bonus, accumulates a running customer balance, and
   writes a human-readable report.

2. DATA MODEL (Step 822)
   Input:  ORDERS.DAT, 16-byte fixed records (see ORD-REC layout)
   Output: ORDRPT.TXT, 80-byte human-readable lines

3. CONTROL FLOW (Step 823-824)
   MAIN-PARA -> INIT-RTN, PROCESS-LOOP (loop), TERM-RTN
   PROCESS-LOOP -> VALIDATE-ORDER, CALC-DISCOUNT,
                   UPDATE-BALANCE, PRINT-LINE
   NOTE: PROCESS-LOOP uses GO TO to its own THRU boundary to skip
   invalid records - equivalent to "continue" in modern languages.

4. DEAD CODE (Step 825)
   OLD-CALC-DISCOUNT-V1 - zero incoming PERFORM/GO TO references.
   Candidate for removal, pending confirmation no external program
   calls it dynamically.

5. BUSINESS RULES (Step 827) - see full decision table
   Tiered discount by OR-TIER and OR-AMT threshold, plus a flat
   +2% export bonus for OR-REGION = "X", capped at 20% total.
   OPEN QUESTION: the 20% cap is never reached by current rates -
   origin unknown, needs business confirmation.

6. VALIDATION RULES (Step 828)
   - OR-AMT must not be zero.
   - OR-TIER must be one of "P", "G", "R".
   - OR-REGION is NOT validated at all (open question).

7. RISKS FOR ANY FUTURE CHANGE
   - Nested nine-way discount logic is easy to misread at the
     boundary between ">=" and ">".
   - GO TO relies on the THRU clause in the caller; removing GO TO
     without checking the PERFORM boundary can silently break
     control flow (verified experimentally in Step 824).
=====================================================
```

### ทำไมรูปแบบนี้ถึงมีประโยชน์

เอกสารนี้ทำหน้าที่เป็น **สะพานเชื่อม** ระหว่างสิ่งที่ค้นพบจากการวิเคราะห์ (ซึ่งกระจัดกระจายอยู่ในขั้นตอน
ต่าง ๆ) กับผู้มีส่วนได้ส่วนเสียคนอื่น (นักพัฒนาที่จะมาต่อยอด, Business Analyst ที่ต้องยืนยันกฎธุรกิจ,
หรือแม้แต่ทีม QA ที่ต้องออกแบบชุดทดสอบ) — สังเกตว่าเอกสารนี้แยกส่วน **"สิ่งที่รู้แน่นอน"** (จากการอ่าน
โค้ดและทดสอบยืนยันแล้ว) ออกจาก **"คำถามเปิด"** (OPEN QUESTION) อย่างชัดเจน ซึ่งเป็นวินัยสำคัญมาก — ไม่ควร
เดาคำตอบของคำถามเปิดเอง แต่ควรระบุไว้ตรง ๆ ว่ายังไม่รู้ แล้วส่งต่อให้ผู้ที่มีอำนาจตัดสินใจทางธุรกิจยืนยัน

### ข้อควรระวัง

- เอกสารสรุปนี้ควรถูกเก็บไว้ในที่ที่ทีมเข้าถึงได้ง่าย (เช่น Wiki ภายในองค์กร หรือเก็บไว้คู่กับซอร์สโค้ด
  ใน Git ตามเทคนิค Part 080) ไม่ใช่แค่ไฟล์ส่วนตัวที่เก็บไว้ในเครื่องคนวิเคราะห์คนเดียว มิฉะนั้นความรู้จะ
  หายไปอีกครั้งเมื่อคนนั้นย้ายงาน — ซ้ำรอยปัญหาเดิมที่ทำให้ต้องมาทำ Reverse Engineering ตั้งแต่แรก
- เอกสารนี้ควรถูกปรับปรุงทุกครั้งที่มีการค้นพบใหม่ระหว่างการ Migration/Refactoring จริง (Part 084/085) —
  มันไม่ใช่เอกสารที่เขียนครั้งเดียวจบ แต่เป็น "living document" ที่งอกเงยไปพร้อมกับความเข้าใจของทีม

### แบบฝึกหัดที่ 829.1

**โจทย์**: จงเพิ่มหัวข้อที่ 8 ในเอกสารสรุปข้างต้น ชื่อ "TEST DATA USED" ที่บันทึกไว้ว่าการวิเคราะห์นี้
ยืนยันด้วยข้อมูลทดสอบชุดใด เพื่อให้คนอื่นสามารถรันซ้ำและยืนยันผลได้เอง

**เฉลยแนวทาง**:

```
8. TEST DATA USED
   ORDERS.DAT (5 lines, see Step 821), verified output:
     RECORDS READ: 5, RECORDS REJECTED: 1 (line 3, OR-AMT=0)
     FINAL BALANCE: 39630.00
   Manual calculation cross-checked against Step 827 decision
   table for all 4 accepted records - see Step 821/827 for detail.
```

การบันทึกข้อมูลทดสอบและผลลัพธ์ที่คาดหวังไว้แบบนี้เป็นก้าวแรกสู่การสร้าง **Golden Master Test** ที่ Part
085 (ขั้นตอนที่ 842) จะใช้เป็นตาข่ายนิรภัยก่อนเริ่ม Refactor ระบบจริง

---

## ขั้นตอนที่ 830: สรุป Checklist กระบวนการ Legacy Analysis และก้าวต่อไป

### Checklist สรุปกระบวนการทั้ง 9 ขั้นตอนที่ผ่านมา

ก่อนจะตัดสินใจว่าจะทำอะไรกับระบบ Legacy ต่อไป (Refactor? Rewrite? Wrap ด้วย API?) ควรตรวจสอบว่าได้ทำ
ครบทุกข้อต่อไปนี้แล้วหรือยัง:

| ลำดับ | รายการตรวจสอบ | อ้างอิงขั้นตอน |
|---|---|---|
| 1 | อ่านโครงสร้างไฟล์ทุกไฟล์จาก FD/Copybook ครบถ้วน สร้าง Data Model ใหม่แล้ว | 822 |
| 2 | สร้าง Call Graph ของ `PERFORM` ครบทุกย่อหน้าแล้ว | 823 |
| 3 | ตรวจสอบ `GO TO` ทุกจุด และยืนยันขอบเขต `PERFORM ... THRU` ที่เกี่ยวข้องแล้ว | 824 |
| 4 | ระบุ Dead Code ที่เป็นไปได้ครบถ้วน และตรวจสอบ False Positive แล้ว | 825 |
| 5 | ทดสอบเครื่องมือ Cross-Reference ที่มีอยู่จริงในสภาพแวดล้อม (ไม่ใช่แค่ที่มีในเอกสาร) | 826 |
| 6 | สกัดกฎธุรกิจทั้งหมดจาก `IF`/`EVALUATE` เป็น Decision Table และทดสอบยืนยันแล้ว | 827 |
| 7 | สกัดกฎ Validation ทั้งหมด รวมถึงบันทึกกรณีที่ "ไม่มีการตรวจสอบ" ด้วย | 828 |
| 8 | เขียนเอกสารสรุปผลการวิเคราะห์ครบทุกหัวข้อ พร้อมแยกคำถามเปิดออกจากข้อเท็จจริง | 829 |
| 9 | เตรียมชุดข้อมูลทดสอบพร้อมผลลัพธ์ที่คาดหวัง (ต้นแบบของ Golden Master Test) | 829 |

### สิ่งที่ Part นี้ยังไม่ได้ทำ (ตั้งใจ)

สังเกตว่าตลอด 830 ขั้นตอนที่ผ่านมา **เรายังไม่ได้แก้โค้ด `LEGACYORD.cob` แม้แต่บรรทัดเดียว** — นี่คือ
ธรรมชาติของ Legacy Analysis: มันเป็นกระบวนการ**สืบสวน** (investigation) ไม่ใช่กระบวนการ**แก้ไข**
(modification) ผลลัพธ์ที่ได้จาก Part นี้ทั้งหมด (Data Model, Call Graph, Decision Table, เอกสารสรุป)
คือ**ข้อมูลนำเข้า**ที่จำเป็นสำหรับการตัดสินใจในขั้นตอนถัดไป

### เชื่อมโยงสู่ Part 084

คำถามที่ตามมาตามธรรมชาติหลังจากวิเคราะห์ระบบเสร็จคือ: **"แล้วเราควรทำอะไรกับมันต่อ?"** — เขียนใหม่ทั้งหมด
ด้วยภาษาสมัยใหม่? ห่อหุ้มด้วย API แล้วค่อย ๆ แทนที่ทีละส่วน? หรือ Refactor ให้สะอาดขึ้นโดยคงภาษา COBOL
เดิมไว้? คำถามเหล่านี้คือหัวใจของ **Part 084: Migration Strategies** ซึ่งจะใช้ผลการวิเคราะห์จาก Part นี้
โดยตรงในการตัดสินใจเลือกกลยุทธ์ที่เหมาะสม และ **Part 085** จะนำทั้งสองส่วนมารวมกันเป็นโปรเจกต์จริงที่
ปรับปรุงระบบ Legacy ด้วยสถาปัตยกรรมสมัยใหม่แบบครบวงจร

### ข้อควรระวัง

- อย่าข้ามขั้นตอน Legacy Analysis ไปเริ่ม Migration ทันทีเพียงเพราะ "ดูแล้วน่าจะเข้าใจโค้ดแล้ว" — ความ
  มั่นใจที่ไม่ผ่านการทดสอบยืนยันเป็นสาเหตุอันดับต้น ๆ ของโครงการ Modernization ที่ล้มเหลวในองค์กรจริง
- checklist นี้เป็นจุดเริ่มต้นที่ดี แต่ระบบ Legacy ขนาดใหญ่กว่านี้มากอาจต้องเพิ่มขั้นตอน เช่น การวิเคราะห์
  ความสัมพันธ์ระหว่างหลายโปรแกรม (Program Dependency, ทบทวน Part 031/043), การวิเคราะห์ Batch Job
  Scheduling (ทบทวน Part 066), หรือการวิเคราะห์ Database Schema แยกต่างหาก (ทบทวน Part 057-060)

### แบบฝึกหัดที่ 830.1

**โจทย์**: สมมติทีมของคุณได้รับมอบหมายให้วิเคราะห์ระบบ Legacy จริงที่มีขนาด 5,000 บรรทัด แบ่งเป็น 8
โปรแกรมที่เรียกกันด้วย `CALL` จงประเมินว่า Checklist ในขั้นตอนนี้ต้องปรับเปลี่ยนหรือเพิ่มเติมอย่างไรบ้าง
เพื่อรองรับความซับซ้อนที่เพิ่มขึ้นนี้

**เฉลยแนวทาง**: ต้องเพิ่มอย่างน้อย (1) การสร้าง **Program Call Graph** ระดับโปรแกรม (ไม่ใช่แค่ย่อหน้า)
โดยไล่หา `CALL "ชื่อโปรแกรม"` ทุกจุด รวมถึงตรวจสอบว่าเป็น Static หรือ Dynamic CALL (ทบทวน Part 043) เพราะ
Dynamic CALL ที่สร้างชื่อโปรแกรมขึ้นระหว่างรันจะไม่ปรากฏในการค้นหาด้วย grep ธรรมดา (2) วิเคราะห์ Copybook
ที่ใช้ร่วมกันระหว่างโปรแกรม (ทบทวน Part 033) เพื่อเข้าใจว่าการแก้ไขโครงสร้างข้อมูลหนึ่งจะกระทบโปรแกรมใด
บ้าง (3) จัดลำดับความสำคัญ (Prioritization) ว่าจะวิเคราะห์โปรแกรมไหนก่อน เพราะการวิเคราะห์ทั้ง 8 โปรแกรม
พร้อมกันแบบละเอียดเท่ากันหมดอาจไม่คุ้มเวลา ควรเริ่มจากโปรแกรมที่มีความเสี่ยงสูงสุดหรือมีแผนจะแก้ไขก่อน

---

## สรุปท้ายบท

Part 083 พาเราเข้าสู่ทักษะสำคัญของงาน COBOL ระดับมืออาชีพที่มักถูกมองข้ามในหลักสูตรทั่วไป: **การวิเคราะห์
และทำความเข้าใจระบบ Legacy ที่ไม่มีเอกสารประกอบ** โดยใช้โปรแกรม `LEGACYORD` เป็นกรณีศึกษาตลอดทั้ง 10
ขั้นตอน เราได้เรียนรู้:

- การอ่านโครงสร้าง `FD`/`WORKING-STORAGE` เพื่อสร้าง Data Model ใหม่โดยไม่ต้องมีเอกสารเดิม (ขั้นตอนที่ 822)
- การเขียนสคริปต์อัตโนมัติไล่ตาม `PERFORM` เพื่อสร้าง Call Graph (ขั้นตอนที่ 823)
- การวิเคราะห์ `GO TO` ร่วมกับขอบเขต `PERFORM ... THRU` และพิสูจน์ผลกระทบด้วยการทดลองจริง (ขั้นตอนที่ 824)
- การหา Dead Code ด้วยสคริปต์ พร้อมแยกแยะ False Positive จาก True Positive (ขั้นตอนที่ 825)
- การทดสอบเครื่องมือ Cross-Reference ของคอมไพเลอร์จริง (`cobc -Xref` ล้มเหลวเพราะไม่มี `cobxref`) และหา
  ทางเลือกสำรองที่ใช้งานได้จริง (ขั้นตอนที่ 826)
- การสกัดกฎธุรกิจที่ซับซ้อนจาก `IF`/`EVALUATE` ซ้อนหลายชั้นให้เป็น Decision Table (ขั้นตอนที่ 827)
- การวิเคราะห์กฎ Validation รวมถึงการสังเกตกฎที่ "ไม่มีอยู่" (ขั้นตอนที่ 828)
- การเขียนเอกสารสรุปผลการวิเคราะห์แบบมาตรฐานที่แยกข้อเท็จจริงจากคำถามเปิด (ขั้นตอนที่ 829)
- Checklist สรุปกระบวนการทั้งหมด และการเชื่อมโยงสู่การตัดสินใจ Migration ต่อไป (ขั้นตอนที่ 830)

ทักษะเหล่านี้เป็นรากฐานสำคัญที่จะถูกนำไปใช้ต่อทันทีใน **Part 084** ซึ่งจะสอนกลยุทธ์การ Migrate ระบบ
COBOL ไปสู่ภาษาอื่น และ **Part 085** ซึ่งเป็นโปรเจกต์รวบยอดปิดท้ายเฟส 5 ที่จะนำระบบ Legacy จริงมาปรับปรุง
ด้วยสถาปัตยกรรมสมัยใหม่แบบครบวงจร โดยใช้เทคนิคการวิเคราะห์จาก Part นี้เป็นจุดเริ่มต้น

**[ไปยัง Part 084: Migration Strategies: COBOL to Java/C# →](part-084-migration-strategies.md)**
