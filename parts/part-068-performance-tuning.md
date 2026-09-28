# Part 068: Performance Tuning สำหรับ Mainframe COBOL (ขั้นตอนที่ 671–680)

## คำนำของ Part นี้

Part 066–067 พาเราไปทำความเข้าใจ Batch Processing Pattern และ DFSORT ซึ่งเป็นเนื้องานหลักของ
COBOL บน Mainframe ที่ต้องประมวลผลข้อมูลปริมาณมหาศาลภายในกรอบเวลาจำกัด ("batch window") ทุกคืน
คำถามที่ตามมาตามธรรมชาติคือ: **แล้วเราจะทำให้โปรแกรม COBOL เหล่านั้นทำงานเร็วขึ้นได้อย่างไร?**

Part นี้ตอบคำถามนั้นด้วยสองระดับที่ต้องแยกให้ชัดตั้งแต่ต้น:

1. **หลักการ Performance ระดับภาษา COBOL** — เรื่องที่เป็นจริงในทุก COBOL compiler รวมถึง GnuCOBOL
   ที่เราใช้ฝึกฝนมาตลอดหลักสูตรนี้ เช่น การเลือกใช้ `PERFORM` แทน `GO TO`, การเลือก `SEARCH` หรือ
   `SEARCH ALL`, การเลือกชนิดข้อมูลตัวเลข, การลดจำนวนครั้งที่เรียก I/O — **หัวข้อกลุ่มนี้เราจะพิสูจน์
   ด้วยการรันจริงและจับเวลาจริงทุกตัวอย่าง** ไม่มีการอ้างตัวเลขลอย ๆ
2. **หลักการ Tuning ระดับ z/OS Mainframe โดยเฉพาะ** — เช่น การปรับค่า `BUFNO` ใน JCL, การอ่านรายงาน
   จาก SMF/RMF — **หัวข้อกลุ่มนี้เป็นความรู้เชิงอ้างอิง (reference/conceptual)** เพราะต้องใช้ z/OS จริง
   หรือ Mainframe Emulator เท่านั้นจึงจะสาธิตได้ ตามที่ระบุไว้ในหมายเหตุมาตรฐานของหลักสูตรตั้งแต่ Part 051
   เราจะบอกไว้ชัดเจนทุกครั้งว่าหัวข้อใดเป็นกลุ่มนี้ เพื่อไม่ให้เข้าใจผิดว่าเป็นสิ่งที่ทดสอบได้จริงในสภาพแวดล้อม
   ของเรา

> **สำคัญเรื่องระเบียบวิธี**: การเปรียบเทียบเวลาทั้งหมดใน Part นี้วัดจากการรันจริงด้วยคำสั่ง `time`
> ของ Linux บนเครื่องเดียวกัน (GnuCOBOL 4.0-early-dev, สถาปัตยกรรม x86_64) รันซ้ำหลายครั้งเพื่อยืนยัน
> ความเสถียรของตัวเลข ตัวเลขที่ได้เป็น**หลักฐานเชิงประจักษ์จากสภาพแวดล้อมนี้เท่านั้น** ไม่ใช่ค่าสัมบูรณ์ที่
> รับประกันว่าจะได้เท่ากันทุกเครื่อง แต่ **อัตราส่วนและทิศทางของความแตกต่าง** (เร็วขึ้นกี่เท่า, ช้าลงกี่เท่า)
> คือบทเรียนสำคัญที่นำไปใช้ได้ทั่วไป และในบางกรณีผลลัพธ์ที่วัดได้จริงยัง **ขัดกับความเชื่อดั้งเดิม** บางอย่าง
> เกี่ยวกับ COBOL ด้วยซ้ำ (ดูขั้นตอนที่ 675) ซึ่งคือเหตุผลว่าทำไมกฎทองของบทนี้คือ **"วัดผลก่อนเชื่อ"**
> เสมอ ไม่ใช่ท่องจำสูตรสำเร็จ

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน โค้ดทุกตัวอย่างที่ระบุว่า
> "ทดสอบจริง" ผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL แล้วทุกตัวอย่าง ส่วนหัวข้อที่ระบุชัดว่า "เชิงแนวคิด/
> z/OS เท่านั้น" จะไม่มีการอ้างว่ารันได้จริงในสภาพแวดล้อมนี้

---

## ขั้นตอนที่ 671: ระเบียบวิธี Performance Tuning — วัดก่อนเชื่อ (Measure, Don't Assume)

### ทำไมต้องเริ่มจากการวัดผล ไม่ใช่เริ่มจาก "สูตรสำเร็จ"

วงการ COBOL/Mainframe มีความเชื่อและ "สูตรสำเร็จ" ด้านประสิทธิภาพส่งต่อกันมาหลายสิบปี บางอย่างยังจริงอยู่
บางอย่างจริงเฉพาะบนฮาร์ดแวร์ Mainframe จริงแต่ไม่จริงบน GnuCOBOL (เพราะสถาปัตยกรรมภายในต่างกันโดย
สิ้นเชิง — ทบทวนจาก Part 001 ขั้นตอนที่ 7: GnuCOBOL แปลง COBOL เป็นภาษา C แล้วคอมไพล์ด้วย `gcc`
อีกที ในขณะที่ IBM Enterprise COBOL คอมไพล์เป็น Machine Code ของ System z โดยตรงและมีชุดคำสั่งพิเศษ
สำหรับเลขฐานสิบที่ฮาร์ดแวร์รองรับ) หลักการสำคัญที่สุดของ Part นี้คือ **"วัดผลจริงในสภาพแวดล้อมของคุณเอง
ก่อนเสมอ อย่าเชื่อสูตรสำเร็จที่ไม่ได้ทดสอบ"**

### เครื่องมือวัดผลที่เราจะใช้ตลอด Part นี้: คำสั่ง `time`

Linux (และ Unix ทั่วไป) มีคำสั่ง `time` ในตัวที่วัดเวลาจริงของการรันโปรแกรมได้ทันที ไม่ต้องเขียนโค้ดวัดเวลา
เพิ่มเอง:

```bash
time ./program_name
```

ผลลัพธ์จะแสดง 3 ค่า:

| ค่า | ความหมาย |
|---|---|
| `real` | เวลาจริงที่ผ่านไป (wall-clock time) — ค่าที่เราสนใจที่สุดในบทนี้ |
| `user` | เวลาที่ CPU ใช้ประมวลผลโค้ดของโปรแกรมเราเอง |
| `sys` | เวลาที่ CPU ใช้เรียก system call (เช่น อ่าน/เขียนไฟล์ผ่าน OS) |

### ตัวอย่างพื้นฐาน: วัดเวลา Compile และเวลา Run แยกกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP671BASELINE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNTER      PIC 9(9)  VALUE 0.
       01  WS-LIMIT        PIC 9(9)  VALUE 5000000.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM UNTIL WS-COUNTER >= WS-LIMIT
               ADD 1 TO WS-COUNTER
           END-PERFORM.
           DISPLAY "Loop finished. WS-COUNTER=" WS-COUNTER.
           STOP RUN.
```

```bash
time cobc -x -o step671_baseline step671_baseline.cob
time ./step671_baseline
```

**ผลลัพธ์จริง (ยืนยันด้วย GnuCOBOL):**

```
== compile timing ==
real	0m0.069s

== run timing ==
Loop finished. WS-COUNTER=005000000
real	0m0.215s
```

### อธิบายจุดสำคัญ

- **เวลา compile กับเวลา run เป็นคนละเรื่องกันโดยสิ้นเชิง** — `cobc` แปลง COBOL เป็น C แล้วเรียก
  `gcc` คอมไพล์เป็น native code (ทบทวนจาก Part 001) ขั้นตอนนี้ทำ**ครั้งเดียว**ตอนพัฒนา ส่วนเวลา `run`
  คือเวลาที่**ผู้ใช้จริงหรือ batch job จริง**ต้องรอทุกครั้งที่โปรแกรมทำงาน — Performance Tuning ในบทนี้
  สนใจเวลา **run** เกือบทั้งหมด ไม่ใช่เวลา compile
- 5,000,000 รอบของการ `ADD 1` ธรรมดาใช้เวลาเพียง 0.215 วินาที บนเครื่องนี้ — ตัวเลขนี้จะเป็น **baseline**
  (เส้นฐาน) ที่เราจะนำไปเทียบกับเทคนิคต่าง ๆ ในขั้นตอนถัดไป
- หลักวิธีการทำงานที่ถูกต้องของ Performance Tuning มืออาชีพคือวงจร **Measure → Identify Bottleneck →
  Fix → Measure Again** เสมอ ไม่ใช่ "เดา" ว่าจุดไหนช้าแล้วแก้โดยไม่วัดผลยืนยัน

### ข้อควรระวัง

- **อย่า optimize ก่อนที่จะรู้ว่าจุดไหนคือคอขวด (bottleneck) จริง ๆ** — การไปปรับแต่งส่วนที่ไม่ใช่จุดช้าที่สุด
  เป็นการเสียเวลาพัฒนาโดยเปล่าประโยชน์ (หลักการที่รู้จักกันในชื่อ "premature optimization")
- ผลการวัดเวลาอาจแกว่งได้เล็กน้อยในแต่ละครั้งที่รัน (ขึ้นกับสถานะของเครื่อง ณ ขณะนั้น) จึงควรรันซ้ำหลายครั้ง
  แล้วดูค่าเฉลี่ยหรือแนวโน้ม ไม่ใช่เชื่อผลจากการรันครั้งเดียว — ทุกตัวเลขใน Part นี้รันซ้ำอย่างน้อย 2-3 ครั้ง
  แล้วจึงนำมาแสดงผล

### แบบฝึกหัดที่ 671.1

**โจทย์**: จงอธิบายว่าทำไมการวัดผล `real` time จึงสำคัญกว่า `user` time สำหรับโปรแกรมที่มีการอ่าน/เขียน
ไฟล์เยอะ ๆ

**เฉลยแนวทาง**: `user` time นับเฉพาะเวลาที่ CPU ประมวลผลโค้ดของโปรแกรมเราเอง แต่เมื่อโปรแกรมมีการ
อ่าน/เขียนไฟล์ (I/O) เวลาส่วนใหญ่มักถูกใช้ไปกับการ**รอ**ดิสก์หรือระบบปฏิบัติการตอบกลับ ซึ่งนับใน `sys`
time หรือเป็นช่วงที่ CPU ว่างรอ (ไม่นับใน `user` เลย) `real` time คือเวลาที่ผู้ใช้งานจริงต้องรอทั้งหมด
รวมทุกอย่างเข้าด้วยกัน จึงเป็นตัวชี้วัดที่ตรงกับประสบการณ์ผู้ใช้และ batch window จริงมากที่สุด

---

## ขั้นตอนที่ 672: PERFORM กับ GO TO — พิสูจน์มายาคติเรื่องความเร็ว

### มายาคติที่ได้ยินบ่อย

หลายคนที่เคยทำงานกับโค้ด COBOL รุ่นเก่าอาจเคยได้ยินความเชื่อว่า **"`GO TO` เร็วกว่า `PERFORM` เพราะ
`PERFORM` ต้อง 'บันทึกตำแหน่งกลับ' (return address) ไว้ก่อนกระโดด ในขณะที่ `GO TO` กระโดดตรง ๆ ไม่ต้อง
จำอะไร"** ในเชิงทฤษฎีล้วน ๆ การกระโดดแบบไม่มีเงื่อนไขคืน (unconditional jump) นั้น "ถูก" กว่าจริง แต่ต้น
ทุนของการเก็บ return address มีขนาดเล็กมากจนแทบไม่มีผลกับ compiler สมัยใหม่ที่แปลงเป็น native code — มา
พิสูจน์ด้วยการรันจริงกันเลย

### ทดสอบ: ลูป 20 ล้านรอบด้วย PERFORM แบบโครงสร้าง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP672PERFORM.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNTER        PIC 9(9)  VALUE 0.
       01  WS-LIMIT          PIC 9(9)  VALUE 20000000.
       01  WS-ACCUM          PIC 9(9)  VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM UNTIL WS-COUNTER >= WS-LIMIT
               ADD 1 TO WS-COUNTER
               ADD 1 TO WS-ACCUM
           END-PERFORM.
           DISPLAY "PERFORM style done. ACCUM=" WS-ACCUM.
           STOP RUN.
```

### ทดสอบ: ลูป 20 ล้านรอบเดียวกันด้วย GO TO (สไตล์ COBOL รุ่นเก่า)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP672GOTO.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNTER        PIC 9(9)  VALUE 0.
       01  WS-LIMIT          PIC 9(9)  VALUE 20000000.
       01  WS-ACCUM          PIC 9(9)  VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           GO TO LOOP-TEST.
       LOOP-BODY.
           ADD 1 TO WS-COUNTER
           ADD 1 TO WS-ACCUM.
       LOOP-TEST.
           IF WS-COUNTER < WS-LIMIT
               GO TO LOOP-BODY
           END-IF.
           DISPLAY "GO TO style done. ACCUM=" WS-ACCUM.
           STOP RUN.
```

```bash
cobc -x -o step672_perform step672_perform.cob
cobc -x -o step672_goto step672_goto.cob
time ./step672_perform
time ./step672_goto
```

**ผลลัพธ์จริง (3 รอบการรันแต่ละแบบ, ยืนยันความเสถียร):**

```
PERFORM: 1.719s, 1.807s, 1.735s   (เฉลี่ย ~1.75s)
GO TO  : 1.735s, 1.715s, 1.695s   (เฉลี่ย ~1.72s)
```

### วิเคราะห์ผล — มายาคติถูกหักล้างด้วยตัวเลขจริง

ผลต่างระหว่างสองแบบอยู่ในระดับ **นอยส์ของการวัดผล (measurement noise)** ไม่ใช่ความแตกต่างที่มีนัยสำคัญ
เลย เพราะเมื่อ GnuCOBOL แปลง `PERFORM UNTIL` เป็นภาษา C มันแปลงออกมาเป็นโครงสร้างวนลูป (`while`/`for`)
ที่คอมไพเลอร์ C (`gcc`) รู้จักและ**ปรับแต่งได้อย่างมีประสิทธิภาพเต็มที่** ในขณะที่ `GO TO` แปลงเป็น C `goto`
ธรรมดา ซึ่ง `gcc` ก็ปรับแต่งได้ดีไม่แพ้กัน **ต้นทุนของการกระโดดแบบมีเงื่อนไขคืนที่เคยมีนัยสำคัญในคอมไพเลอร์
ยุค 1970-1980 แทบไม่มีผลกับคอมไพเลอร์สมัยใหม่แล้ว**

### สรุปบทเรียนที่ถูกต้อง: เลือก PERFORM เพราะ "อ่านง่ายและดูแลง่าย" ไม่ใช่เพราะ "เร็วกว่า"

เหตุผลที่แท้จริงและสำคัญกว่ามากที่ควรใช้ `PERFORM` แทน `GO TO` (ตามที่เรียนมาตั้งแต่ Part 012) คือ:

1. **Structured Programming ลดบั๊ก** — โค้ดที่มีจุดกระโดดพันกันไปมา ("spaghetti code") ยากต่อการตามรอย
   และแก้ไขโดยไม่ทำให้ตรรกะส่วนอื่นพัง
2. **`PERFORM` มีขอบเขตชัดเจน** — รู้ทันทีว่าลูปเริ่มและจบตรงไหนจาก `END-PERFORM` ในขณะที่ `GO TO` ต้อง
   ไล่หา label ปลายทางเองทั้งโปรแกรม
3. **Compiler สมัยใหม่ปรับแต่ง PERFORM ได้ดีเท่ากัน** (ตามที่พิสูจน์ข้างต้น) — จึงไม่มีข้อแลกเปลี่ยนด้าน
   ความเร็วที่ต้องเสียสละเลยในการเลือกความชัดเจน

### ข้อควรระวัง

- **อย่าเชื่อคำแนะนำด้าน performance โดยไม่มีการทดสอบยืนยัน** แม้แต่คำแนะนำที่ฟังดูสมเหตุสมผลในเชิงทฤษฎี
  ก็อาจไม่จริงกับคอมไพเลอร์หรือสภาพแวดล้อมที่ใช้งานจริง — นี่คือหัวใจของ Part นี้ทั้งบท
- ผลลัพธ์นี้เป็นจริงสำหรับ **GnuCOBOL ที่แปลงเป็น C** เท่านั้น บนคอมไพเลอร์ Mainframe จริงบางรุ่นที่แปลง
  เป็น assembly ของ System z โดยตรง อาจมีรายละเอียดต่างกันได้ แต่หลักการ "structured code ไม่ได้ช้ากว่า
  โดยธรรมชาติ" ยังคงเป็นความจริงที่ยอมรับกันในอุตสาหกรรมมานานแล้ว

### แบบฝึกหัดที่ 672.1

**โจทย์**: จงอธิบายว่าทำไมผลการทดลองนี้จึงเป็นตัวอย่างที่ดีของหลักการ "วัดผลก่อนเชื่อ" ที่แนะนำไว้ในขั้นตอน
ที่ 671

**เฉลยแนวทาง**: หากไม่ทดสอบจริง นักพัฒนาบางคนอาจเลือกเขียนโค้ดด้วย `GO TO` เพื่อหวังผลด้านความเร็ว
โดยยอมแลกกับความอ่านง่ายของโค้ด (premature optimization ที่แนะนำให้หลีกเลี่ยงในขั้นตอนที่ 671) แต่การ
วัดผลจริงแสดงให้เห็นว่า**ไม่มีข้อแลกเปลี่ยนใด ๆ เกิดขึ้นจริงเลย** ทำให้การตัดสินใจที่ถูกต้องคือเลือกใช้
`PERFORM` เสมอเพื่อความชัดเจนและดูแลรักษาง่าย โดยไม่ต้องกังวลเรื่องความเร็วที่เสียไปเลย

---

## ขั้นตอนที่ 673: ต้นทุนที่แท้จริงของ SEARCH (Linear Search) เมื่อตารางใหญ่

### ทบทวนจาก Part 017

Part 017 สอน `SEARCH` (Linear Search — ไล่ตรวจทีละ element ตั้งแต่ต้นจนพบ) และ `SEARCH ALL`
(Binary Search — ตัดครึ่งตารางไปเรื่อย ๆ แต่ต้องเรียงลำดับ (`ASCENDING KEY`) ไว้ก่อน) ขั้นตอนนี้และ
ขั้นตอนถัดไปจะพิสูจน์ด้วยตัวเลขจริงว่า **ความแตกต่างด้านประสิทธิภาพระหว่างสองแบบนี้มหาศาลแค่ไหนเมื่อ
ตารางมีขนาดใหญ่ระดับที่พบได้จริงในงาน Batch ของ Mainframe** (ตารางรหัสสินค้า, ตารางอัตราแลกเปลี่ยน,
ตารางรหัสไปรษณีย์ ฯลฯ ที่มีนับหมื่นถึงนับแสนรายการ)

### ทดสอบ: ตาราง 20,000 รายการ ค้นหา 20,000 ครั้งแบบ Linear SEARCH (กรณีเลวร้ายที่สุด)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP673SEARCH.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-ENTRY OCCURS 20000 TIMES
                   INDEXED BY PT-IDX.
               10  PROD-CODE       PIC 9(6).
               10  PROD-NAME       PIC X(10).

       01  WS-FILL-IDX         PIC 9(5).
       01  WS-SEARCH-PASS      PIC 9(5).
       01  WS-SEARCH-KEY       PIC 9(6).
       01  WS-FOUND-COUNT      PIC 9(7) VALUE 0.
       01  WS-NOTFOUND-COUNT   PIC 9(7) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Build a table of 20,000 entries with codes 100000..119999.
           PERFORM VARYING WS-FILL-IDX FROM 1 BY 1
                   UNTIL WS-FILL-IDX > 20000
               COMPUTE PROD-CODE(WS-FILL-IDX) =
                       100000 + WS-FILL-IDX - 1
               MOVE "ITEM" TO PROD-NAME(WS-FILL-IDX)
           END-PERFORM.

      *> Bias every search toward the far end of the table so each
      *> linear SEARCH must walk nearly all 20,000 entries - this is
      *> the worst case that shows SEARCH's O(n) cost clearly.
           PERFORM VARYING WS-SEARCH-PASS FROM 1 BY 1
                   UNTIL WS-SEARCH-PASS > 20000
               COMPUTE WS-SEARCH-KEY = 100000 + 19999 -
                   FUNCTION MOD(WS-SEARCH-PASS, 50)
               SET PT-IDX TO 1
               SEARCH PRODUCT-ENTRY
                   AT END
                       ADD 1 TO WS-NOTFOUND-COUNT
                   WHEN PROD-CODE(PT-IDX) = WS-SEARCH-KEY
                       ADD 1 TO WS-FOUND-COUNT
               END-SEARCH
           END-PERFORM.

           DISPLAY "SEARCH (linear): found=" WS-FOUND-COUNT
               " not-found=" WS-NOTFOUND-COUNT.
           STOP RUN.
```

```bash
cobc -x -o step673_search step673_search.cob
time ./step673_search
```

**ผลลัพธ์จริง (2 รอบการรัน):**

```
SEARCH (linear): found=0020000 not-found=0000000
real	0m1.805s
real	0m1.831s
```

### อธิบายจุดสำคัญ

- โจทย์นี้จงใจ**เอนเอียงค่าที่ค้นหาไปทางปลายตาราง** (`100000 + 19999 - MOD(pass, 50)` วนอยู่ใน 50
  รายการสุดท้าย) เพื่อจำลอง **กรณีเลวร้ายที่สุด (worst case)** ของ Linear Search ที่ต้องไล่ตรวจเกือบทั้ง
  20,000 รายการทุกครั้งก่อนจะเจอ
- ผลลัพธ์คือใช้เวลาเกือบ **1.8 วินาที** สำหรับการค้นหา 20,000 ครั้ง — คิดเป็นการเปรียบเทียบข้อมูลรวมกัน
  ประมาณ 400 ล้านครั้ง (20,000 ครั้งค้นหา × เฉลี่ยเกือบ 20,000 รายการต่อครั้ง)
- **ความซับซ้อนเชิงเวลาของ Linear Search คือ O(n)** ต่อการค้นหาหนึ่งครั้ง — ยิ่งตารางใหญ่ขึ้นเท่าไร เวลา
  ค้นหาต่อครั้งก็ยิ่งแปรผันตรงตามขนาดตารางนั้น

### ข้อควรระวัง

- ในโปรแกรมจริงที่ตารางมีขนาดเล็ก (เช่น ไม่กี่สิบรายการ) ความแตกต่างระหว่าง Linear และ Binary Search
  **แทบไม่มีนัยสำคัญเลย** — ปัญหาจะปรากฏชัดก็ต่อเมื่อตารางมีขนาดใหญ่ (หลักพันขึ้นไป) **และ**ถูกค้นหาบ่อย
  ครั้ง (เช่น ในลูปประมวลผล transaction จำนวนมาก) พร้อมกัน — เกณฑ์นี้พบได้บ่อยมากในงาน Batch ของ
  Mainframe จริง
- อย่าลืม `SET PT-IDX TO 1` ก่อน `SEARCH` ทุกครั้ง (ทบทวนจาก Part 017) มิฉะนั้นการค้นหาจะเริ่มจาก
  ตำแหน่งที่ index ค้างอยู่จากการค้นหาครั้งก่อน ทำให้ผลลัพธ์ผิดพลาด

### แบบฝึกหัดที่ 673.1

**โจทย์**: หากเปลี่ยนให้ค่าที่ค้นหาทุกครั้งอยู่ที่**ตำแหน่งแรก**ของตาราง (`PROD-CODE(1)`) แทนที่จะเป็น
ปลายตาราง จงคาดเดาว่าเวลาที่ใช้จะเปลี่ยนไปอย่างไร และเพราะเหตุใด

**เฉลย**: เวลาที่ใช้จะ**ลดลงอย่างมาก** (เกือบจะเป็นค่าคงที่ไม่ว่าตารางจะใหญ่แค่ไหน) เพราะ Linear Search
เปรียบเทียบจากรายการแรกไปเรื่อย ๆ หากค่าที่ต้องการอยู่ที่ตำแหน่งแรกพอดี การค้นหาจะจบใน**การเปรียบเทียบ
เพียงครั้งเดียว**ทุกครั้ง นี่คือเหตุผลที่การทดสอบ performance ต้องใช้ **กรณีเลวร้ายที่สุด (worst case)**
เสมอ ไม่ใช่กรณีที่เอื้อประโยชน์ให้กับอัลกอริทึมที่กำลังทดสอบ มิฉะนั้นผลลัพธ์จะไม่สะท้อนพฤติกรรมจริงเมื่อใช้
งานกับข้อมูลที่ไม่สามารถควบคุมตำแหน่งได้

---

## ขั้นตอนที่ 674: SEARCH ALL (Binary Search) — ความเร็วที่ต่างกันเกือบ 100 เท่า

### เงื่อนไขที่ต้องมีก่อนใช้ SEARCH ALL

ทบทวนจาก Part 017: `SEARCH ALL` ต้องการให้ตารางประกาศ `ASCENDING KEY IS <field>` และ**ข้อมูลใน
ตารางต้องถูกเรียงตามคีย์นั้นแล้วจริง ๆ** ก่อนค้นหา (COBOL ไม่ตรวจสอบให้อัตโนมัติ — ถ้าข้อมูลไม่ได้เรียงจริง
ผลลัพธ์จะผิดพลาดแบบไม่มีการเตือน) ขั้นตอนนี้ใช้ตารางและรูปแบบการค้นหาชุดเดียวกันทุกประการกับขั้นตอนที่
673 เพื่อให้เปรียบเทียบกันได้อย่างยุติธรรมที่สุด

### ทดสอบ: ตาราง 20,000 รายการเดียวกัน ค้นหา 20,000 ครั้งเดียวกัน ด้วย SEARCH ALL

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP674SEARCHALL.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-ENTRY OCCURS 20000 TIMES
                   ASCENDING KEY IS PROD-CODE
                   INDEXED BY PT-IDX.
               10  PROD-CODE       PIC 9(6).
               10  PROD-NAME       PIC X(10).

       01  WS-FILL-IDX         PIC 9(5).
       01  WS-SEARCH-PASS      PIC 9(5).
       01  WS-SEARCH-KEY       PIC 9(6).
       01  WS-FOUND-COUNT      PIC 9(7) VALUE 0.
       01  WS-NOTFOUND-COUNT   PIC 9(7) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Same table contents and ASCENDING KEY (needed for SEARCH
      *> ALL to perform a binary search) as the linear SEARCH demo
      *> in step 673 - same 20,000 entries, same 20,000 lookups.
           PERFORM VARYING WS-FILL-IDX FROM 1 BY 1
                   UNTIL WS-FILL-IDX > 20000
               COMPUTE PROD-CODE(WS-FILL-IDX) =
                       100000 + WS-FILL-IDX - 1
               MOVE "ITEM" TO PROD-NAME(WS-FILL-IDX)
           END-PERFORM.

           PERFORM VARYING WS-SEARCH-PASS FROM 1 BY 1
                   UNTIL WS-SEARCH-PASS > 20000
               COMPUTE WS-SEARCH-KEY = 100000 + 19999 -
                   FUNCTION MOD(WS-SEARCH-PASS, 50)
               SEARCH ALL PRODUCT-ENTRY
                   AT END
                       ADD 1 TO WS-NOTFOUND-COUNT
                   WHEN PROD-CODE(PT-IDX) = WS-SEARCH-KEY
                       ADD 1 TO WS-FOUND-COUNT
               END-SEARCH
           END-PERFORM.

           DISPLAY "SEARCH ALL (binary): found=" WS-FOUND-COUNT
               " not-found=" WS-NOTFOUND-COUNT.
           STOP RUN.
```

```bash
cobc -x -o step674_searchall step674_searchall.cob
time ./step674_searchall
```

**ผลลัพธ์จริง (2 รอบการรัน):**

```
SEARCH ALL (binary): found=0020000 not-found=0000000
real	0m0.016s
real	0m0.017s
```

### ตารางเปรียบเทียบผลลัพธ์ทั้งสองขั้นตอน

| เทคนิค | เวลาที่ใช้ (20,000 ครั้งค้นหา) | อัตราเร็วขึ้น |
|---|---|---|
| `SEARCH` (Linear) | ~1.82 วินาที | 1× (baseline) |
| `SEARCH ALL` (Binary) | ~0.017 วินาที | **~107× เร็วกว่า** |

### อธิบายจุดสำคัญ

- แม้แต่ในกรณีเลวร้ายที่สุดของ Linear Search (ค่าที่ค้นหาอยู่ปลายตารางเสมอ) **Binary Search ก็ไม่ได้รับ
  ผลกระทบเลย** เพราะการตัดครึ่งตารางทำให้ไม่ว่าค่าที่ต้องการจะอยู่ตำแหน่งไหนก็ใช้จำนวนการเปรียบเทียบใกล้
  เคียงกันเสมอ (ประมาณ log₂(20000) ≈ 15 ครั้งต่อการค้นหา เทียบกับ Linear ที่ใช้เกือบ 20,000 ครั้งต่อการ
  ค้นหาในกรณีนี้)
- **ความซับซ้อนเชิงเวลาของ Binary Search คือ O(log n)** — นี่คือเหตุผลที่ความแตกต่างยิ่งมากขึ้นเรื่อย ๆ
  เมื่อตารางใหญ่ขึ้น (ถ้าตารางมี 2,000,000 รายการแทนที่จะเป็น 20,000 ส่วนต่างจะยิ่งมหาศาลกว่านี้อีกมาก
  เพราะ Linear Search ช้าลงตามสัดส่วน แต่ Binary Search ช้าลงแค่ log₂ ของอัตราส่วนนั้น)
- **นี่คือหนึ่งใน Performance Tuning เทคนิคเดียวที่ทรงพลังที่สุดในบทนี้ทั้งหมด** เพราะเป็นการเปลี่ยน
  อัลกอริทึม (algorithmic improvement) ไม่ใช่แค่การปรับแต่งเล็กน้อย (micro-optimization) — ผลต่างระดับ
  100 เท่าไม่มีทางเกิดขึ้นได้จากเทคนิคระดับ syntax เพียงอย่างเดียว

### ข้อควรระวัง

- ต้นทุนแฝงของ `SEARCH ALL` ที่มักถูกมองข้ามคือ **การต้องเรียงลำดับตารางให้เสร็จก่อน** ถ้าข้อมูลเข้ามา
  แบบไม่เรียงลำดับ (เช่น อ่านจากไฟล์ที่ไม่ได้ SORT มาก่อน) ต้องเสียเวลา sort ตารางก่อนใช้ `SEARCH ALL`
  เสมอ (Part 027 สอน `SORT` Statement) ถ้าตารางถูกค้นหาเพียงครั้งเดียวหรือไม่กี่ครั้ง ต้นทุนการ sort อาจ
  สูงกว่าประโยชน์ที่ได้ — เทคนิคนี้คุ้มค่าที่สุดเมื่อ **ตารางถูกสร้างครั้งเดียวแล้วค้นหาซ้ำหลายพัน/หมื่นครั้ง**
  ซึ่งเป็นรูปแบบการใช้งานทั่วไปในโปรแกรม Batch ของ Mainframe จริง
- ถ้าลืม `ASCENDING KEY` หรือข้อมูลไม่ได้เรียงจริงตามที่ประกาศไว้ `SEARCH ALL` **จะให้ผลลัพธ์ผิดพลาด
  แบบไม่มี error หรือ warning ใด ๆ เลย** เพราะอัลกอริทึม Binary Search สันนิษฐานเสมอว่าข้อมูลเรียงแล้ว —
  ความเสี่ยงนี้ต้องระวังมากกว่าความช้าของ Linear Search เสียอีกในบางกรณี

### แบบฝึกหัดที่ 674.1

**โจทย์**: จงอธิบายว่าทำไมความแตกต่างระหว่าง `SEARCH` และ `SEARCH ALL` จึงเป็นตัวอย่างที่ดีที่สุดในบท
นี้ของหลักการ "การปรับปรุงระดับอัลกอริทึม (algorithmic) มีพลังมากกว่าการปรับปรุงระดับ syntax เล็กน้อย
(micro-optimization) เสมอ"

**เฉลยแนวทาง**: การเปลี่ยนแปลงระดับ syntax เช่นที่เห็นในขั้นตอนที่ 672 (PERFORM เทียบกับ GO TO) ให้
ผลต่างระดับนอยส์การวัดผล (แทบไม่มีนัยสำคัญ) เพราะทั้งสองแบบยังคงทำงานด้วยความซับซ้อนเชิงเวลาแบบ
เดียวกัน (ทำงาน N รอบเท่าเดิม) แต่การเปลี่ยนจาก Linear Search (O(n)) เป็น Binary Search (O(log n))
คือการเปลี่ยน**อัตราการเติบโตของเวลาที่ใช้เมื่อข้อมูลใหญ่ขึ้น** ซึ่งส่งผลกระทบที่ทวีคูณตามขนาดข้อมูล ยิ่ง
ข้อมูลใหญ่เท่าไรความแตกต่างยิ่งมหาศาลขึ้นเรื่อย ๆ แบบไม่มีขีดจำกัด ในขณะที่การปรับปรุงระดับ syntax
ให้ผลต่างคงที่ (หรือแทบไม่มีผลต่าง) ไม่ว่าข้อมูลจะใหญ่แค่ไหนก็ตาม

---

## ขั้นตอนที่ 675: การเลือกชนิดข้อมูลให้เหมาะกับงาน — COMP, COMP-3 และ DISPLAY

### ทบทวนความเชื่อดั้งเดิม

ความเชื่อดั้งเดิมของวงการ COBOL/Mainframe คือ **"COMP (Binary) และ COMP-3 (Packed Decimal) เร็ว
กว่า DISPLAY (Zoned Decimal) เสมอสำหรับงานคำนวณ เพราะฮาร์ดแวร์ Mainframe มีชุดคำสั่งพิเศษที่ทำงาน
กับรูปแบบ Packed/Binary โดยตรง"** — ความเชื่อนี้ **เป็นจริงบนฮาร์ดแวร์ IBM System z จริง** ที่มีชุดคำสั่ง
Packed Decimal Arithmetic ในระดับฮาร์ดแวร์ แต่ **GnuCOBOL ไม่ได้รันบนฮาร์ดแวร์นั้น** — มันแปลงเป็น C
แล้วรันบน CPU สถาปัตยกรรมทั่วไป (x86_64 ในกรณีนี้) ที่ไม่มีชุดคำสั่ง Packed Decimal ระดับฮาร์ดแวร์เลย มา
พิสูจน์กันว่าความเชื่อนี้ยังคงจริงหรือไม่ในสภาพแวดล้อมของเรา

### ทดสอบที่ 1: COMPUTE ที่มีการคูณ (10 ล้านรอบ)

สามโปรแกรมต่อไปนี้ทำงานเหมือนกันทุกประการ ต่างกันแค่ `PICTURE` clause:

```cobol
      *> DISPLAY version (Zoned Decimal - default when USAGE is not specified)
       01  WS-A              PIC S9(9)      VALUE 123456.
       01  WS-B              PIC S9(9)      VALUE 654321.
       01  WS-RESULT         PIC S9(11)     VALUE 0.
           ...
           COMPUTE WS-RESULT = (WS-A + WS-COUNTER) * WS-B
```

```cobol
      *> COMP version (Binary)
       01  WS-A              PIC S9(9) COMP  VALUE 123456.
       01  WS-B              PIC S9(9) COMP  VALUE 654321.
       01  WS-RESULT         PIC S9(11) COMP VALUE 0.
```

```cobol
      *> COMP-3 version (Packed Decimal)
       01  WS-A              PIC S9(9) COMP-3  VALUE 123456.
       01  WS-B              PIC S9(9) COMP-3  VALUE 654321.
       01  WS-RESULT         PIC S9(11) COMP-3 VALUE 0.
```

(โครงสร้างส่วนที่เหลือเหมือนกันทุกไฟล์ — ลูป `COMPUTE WS-RESULT = (WS-A + WS-COUNTER) * WS-B`
10,000,000 รอบ)

```bash
cobc -x -o step675_display step675_display.cob
cobc -x -o step675_comp     step675_comp.cob
cobc -x -o step675_comp3    step675_comp3.cob
time ./step675_display
time ./step675_comp
time ./step675_comp3
```

**ผลลัพธ์จริง (2 รอบการรันแต่ละแบบ):**

```
DISPLAY : 1.264s, 1.278s   (เฉลี่ย ~1.27s)
COMP    : 1.254s, 1.231s   (เฉลี่ย ~1.24s)
COMP-3  : 2.901s, 2.737s   (เฉลี่ย ~2.82s)
```

### ผลลัพธ์ที่ 1 พลิกความเชื่อดั้งเดิมโดยสิ้นเชิง

สำหรับงาน `COMPUTE` ที่มีการคูณแบบนี้ **`COMP-3` กลับช้ากว่า `DISPLAY` และ `COMP` ถึงกว่า 2 เท่า!**
เหตุผลคือ libcob (runtime library ของ GnuCOBOL) ต้อง **แปลงค่า COMP-3 (Packed BCD) กลับเป็นรูปแบบ
เลขฐานสิบภายใน (ผ่าน GMP - GNU Multiple Precision library) ก่อนคำนวณทุกครั้ง แล้วแปลงกลับเป็น Packed
อีกครั้งหลังคำนวณเสร็จ** ต้นทุนการแปลงไปมานี้ (pack/unpack overhead) มากกว่าประโยชน์ที่ควรจะได้จาก
รูปแบบที่กะทัดรัดกว่า เพราะ x86_64 ไม่มีฮาร์ดแวร์ช่วยจัดการ Packed Decimal โดยตรงเหมือน System z

### ทดสอบที่ 2: ADD ธรรมดา ไม่มีการคูณ (30 ล้านรอบ) — ผลลัพธ์กลับตาลปัตรอีกครั้ง

```cobol
      *> DISPLAY
       01  WS-COUNTER   PIC 9(9)      VALUE 0.
       01  WS-ACCUM     PIC 9(9)      VALUE 0.
           PERFORM UNTIL WS-COUNTER >= WS-LIMIT
               ADD 1 TO WS-COUNTER
               ADD 3 TO WS-ACCUM
           END-PERFORM.
```

```cobol
      *> COMP-3
       01  WS-COUNTER   PIC 9(9) COMP-3  VALUE 0.
       01  WS-ACCUM     PIC 9(9) COMP-3  VALUE 0.
```

**ผลลัพธ์จริง (2 รอบการรันแต่ละแบบ):**

```
DISPLAY add-only : 2.588s, 2.582s   (เฉลี่ย ~2.59s)
COMP-3  add-only : 1.549s, 1.636s   (เฉลี่ย ~1.59s)
```

คราวนี้ **`COMP-3` เร็วกว่า `DISPLAY` ประมาณ 1.6 เท่าสำหรับ `ADD` ธรรมดา** — สลับกับผลของทดสอบที่ 1
โดยสิ้นเชิง!

### สรุปบทเรียนที่ถูกต้องที่สุดของขั้นตอนนี้

| ชนิดการคำนวณ | ชนิดข้อมูลที่เร็วกว่าใน GnuCOBOL (สภาพแวดล้อมนี้) |
|---|---|
| `COMPUTE` ที่มีการคูณ/ผลลัพธ์หลักเยอะ | `DISPLAY` หรือ `COMP` (ไม่ใช่ `COMP-3`) |
| `ADD`/`SUBTRACT` สะสมค่าธรรมดา | `COMP-3` |

**บทเรียนที่แท้จริงไม่ใช่ "ชนิดข้อมูลไหนเร็วที่สุด" แต่คือ "คำตอบขึ้นอยู่กับรูปแบบการคำนวณ และต้องวัดผล
จริงกับ workload จริงของคุณเองเสมอ"** — นี่คือเหตุผลที่ขั้นตอนที่ 671 เน้นย้ำหลักการ "วัดก่อนเชื่อ" ตั้งแต่
ต้นบท เพราะแม้แต่ความเชื่อที่ดูมีเหตุผลทางเทคนิคและเป็นจริงบนฮาร์ดแวร์ Mainframe จริง ก็อาจกลับตาลปัตร
สิ้นเชิงเมื่อรันบนคอมไพเลอร์/ฮาร์ดแวร์อื่น

### ข้อควรระวัง

- **อย่านำตัวเลขในขั้นตอนนี้ไปอ้างอิงว่าเป็นจริงบน IBM Enterprise COBOL บน z/OS จริง** เพราะฮาร์ดแวร์
  System z มีชุดคำสั่ง Packed Decimal Arithmetic ในระดับ CPU โดยตรง ทำให้ `COMP-3` มีแนวโน้มเร็วกว่า
  `DISPLAY` อย่างสม่ำเสมอมากกว่าที่เห็นใน GnuCOBOL — **นี่คือตัวอย่างสำคัญที่สุดของ Part นี้ที่แสดงว่าทำไม
  ต้องระบุให้ชัดเจนเสมอว่ากำลังวัดผลบนคอมไพเลอร์/ฮาร์ดแวร์ตัวใด**
- เหตุผลที่ยังควรใช้ `COMP-3`/`COMP` ในโปรแกรมจริงไม่ได้มีแค่เรื่องความเร็วอย่างเดียว — ขนาดพื้นที่จัดเก็บ
  ที่เล็กกว่า (`COMP-3` ใช้ประมาณครึ่งหนึ่งของ `DISPLAY` สำหรับตัวเลขความยาวเท่ากัน) ก็สำคัญมากเมื่อต้อง
  เก็บข้อมูลจำนวนมหาศาลในไฟล์หรือฐานข้อมูล (ทบทวนจาก Part 006)

### แบบฝึกหัดที่ 675.1

**โจทย์**: จากผลการทดลองทั้งสองชุดในขั้นตอนนี้ จงอธิบายว่าทำไมนักพัฒนาที่ต้องการ tuning โปรแกรม COBOL
จริงจึงไม่ควรเปลี่ยนชนิดข้อมูลทั้งโปรแกรมเป็น `COMP-3` แบบเหมารวมโดยไม่ทดสอบก่อน

**เฉลยแนวทาง**: เพราะโปรแกรมจริงมักมีการคำนวณหลายรูปแบบปะปนกัน (`ADD` สะสมยอด, `COMPUTE` คำนวณ
ดอกเบี้ยที่มีการคูณ/หาร ฯลฯ) ผลการทดลองแสดงให้เห็นชัดว่าชนิดข้อมูลที่เหมาะกับการคำนวณแบบหนึ่งอาจช้ากว่า
สำหรับการคำนวณอีกแบบหนึ่งโดยสิ้นเชิง การเปลี่ยนเหมารวมโดยไม่วัดผลอาจทำให้ส่วนที่เคยเร็วกลับช้าลง (เช่น
ในกรณีนี้คือ `COMPUTE` ที่มีการคูณ) ในขณะที่ส่วนที่ควรปรับปรุงจริง (เช่น `ADD` สะสมยอดจำนวนมาก) อาจไม่ได้
ถูกจับตาไปด้วย วิธีที่ถูกต้องคือระบุส่วนที่เป็นคอขวดจริงด้วยการวัดผลก่อน (ขั้นตอนที่ 671) แล้วจึงเลือกปรับ
เฉพาะจุดนั้นตามรูปแบบการคำนวณที่ใช้จริง

---

## ขั้นตอนที่ 676: ลดจำนวนครั้งของการเรียก I/O ให้น้อยที่สุด

### หลักการ: I/O แพงกว่า CPU เสมอ

ไม่ว่าจะเป็น Mainframe จริงหรือ GnuCOBOL บนเครื่อง PC ธรรมดา หลักการที่เป็นจริงเสมอคือ **การเข้าถึง
ดิสก์ (I/O) ช้ากว่าการประมวลผลในหน่วยความจำ (CPU/RAM) หลายเท่าตัวเสมอ** ทุกครั้งที่โปรแกรมเรียกคำสั่ง
`WRITE`/`READ` จะมีต้นทุนคงที่ (overhead) ของการเรียก system call ผ่านชั้น runtime library เกิดขึ้น
**ไม่ว่าข้อมูลที่เขียนจะเล็กแค่ไหนก็ตาม** ดังนั้นการ**ลดจำนวนครั้งที่เรียก I/O** (แม้จะเขียนข้อมูลปริมาณ
เท่าเดิม) มักช่วยเพิ่มประสิทธิภาพได้จริง — นี่คือหลักการเดียวกับที่ทำให้ Mainframe ใช้แนวคิด **Blocking**
(รวมหลาย logical record เข้าเป็น physical block เดียวต่อการอ่าน/เขียนดิสก์หนึ่งครั้ง ตามที่จะกล่าวถึงใน
เชิงแนวคิดที่ขั้นตอนที่ 678)

### ทดสอบ: เขียน 5,000,000 ค่า ด้วย WRITE ทีละ record เทียบกับ WRITE แบบรวมกลุ่ม

**เวอร์ชัน A — WRITE ทีละ 1 record (5,000,000 ครั้ง):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP676MANY.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT OUT-FILE ASSIGN TO "MANY676.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  OUT-FILE.
       01  OUT-RECORD          PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-IDX              PIC 9(7)  VALUE 0.
       01  WS-LIMIT            PIC 9(7)  VALUE 5000000.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT OUT-FILE.
      *> One WRITE call per single 10-byte record: 5,000,000
      *> separate WRITE statement executions.
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > WS-LIMIT
               MOVE WS-IDX TO OUT-RECORD
               WRITE OUT-RECORD
           END-PERFORM.
           CLOSE OUT-FILE.
           DISPLAY "Wrote " WS-LIMIT " records with 1 WRITE each.".
           STOP RUN.
```

**เวอร์ชัน B — รวม 50 ค่าต่อ record แล้ว WRITE (เหลือ 100,000 ครั้ง, ข้อมูลรวมเท่าเดิม):**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP676FEW.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT OUT-FILE ASSIGN TO "FEW676.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  OUT-FILE.
       01  OUT-RECORD          PIC X(500).

       WORKING-STORAGE SECTION.
       01  WS-IDX              PIC 9(7)  VALUE 0.
       01  WS-GROUP-IDX        PIC 9(3)  VALUE 0.
       01  WS-LIMIT            PIC 9(7)  VALUE 5000000.
       01  WS-PIECE            PIC X(10).
       01  WS-OFFSET           PIC 9(4)  VALUE 1.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT OUT-FILE.
      *> Same 5,000,000 ten-byte logical values, but batched 50 at
      *> a time into one 500-byte record - only 100,000 WRITE calls
      *> instead of 5,000,000. The DATA VOLUME on disk is identical;
      *> only the number of WRITE statement EXECUTIONS drops.
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > WS-LIMIT
               MOVE WS-IDX TO WS-PIECE
               MOVE WS-PIECE TO OUT-RECORD(WS-OFFSET:10)
               ADD 1 TO WS-GROUP-IDX
               ADD 10 TO WS-OFFSET
               IF WS-GROUP-IDX = 50
                   WRITE OUT-RECORD
                   MOVE SPACES TO OUT-RECORD
                   MOVE 1 TO WS-OFFSET
                   MOVE 0 TO WS-GROUP-IDX
               END-IF
           END-PERFORM.
           IF WS-GROUP-IDX > 0
               WRITE OUT-RECORD
           END-IF.
           CLOSE OUT-FILE.
           DISPLAY "Wrote " WS-LIMIT
               " logical values batched 50-per-WRITE.".
           STOP RUN.
```

```bash
cobc -x -o step676_manywrites step676_manywrites.cob
cobc -x -o step676_fewwrites  step676_fewwrites.cob
time ./step676_manywrites
time ./step676_fewwrites
```

**ผลลัพธ์จริง (2 รอบการรันแต่ละแบบ):**

```
Many WRITEs (5,000,000 calls)  : 1.319s, 1.340s   (เฉลี่ย ~1.33s)
Few WRITEs   (100,000 calls)   : 0.925s, 0.775s   (เฉลี่ย ~0.85s)
```

### อธิบายจุดสำคัญ

- ข้อมูล**ปริมาณเท่ากันทุกไบต์**ถูกเขียนลงดิสก์ทั้งสองเวอร์ชัน แต่เวอร์ชันที่ลดจำนวน**ครั้งที่เรียก**
  `WRITE` ลง 50 เท่า (จาก 5,000,000 เหลือ 100,000) กลับเร็วขึ้นประมาณ **35-40%**
- ส่วนต่างนี้มาจากต้นทุนคงที่ (fixed overhead) ของการเรียก `WRITE` แต่ละครั้งผ่านชั้น libcob และ
  system call ของ OS — ยิ่งเรียกน้อยครั้ง ต้นทุนคงที่นี้ยิ่งถูกจ่ายน้อยครั้งตามไปด้วย แม้ปริมาณข้อมูลรวมจะ
  เท่ากันก็ตาม
- หลักการนี้ตรงกับแนวคิด **Blocking Factor** บน Mainframe จริงทุกประการ (VSAM/QSAM รวมหลาย logical
  record เป็น physical block เดียวก่อนเขียนดิสก์จริง) ซึ่งจะอธิบายเชิงแนวคิดในขั้นตอนที่ 678

### ข้อควรระวัง

- การรวม record หลายตัวเป็นหนึ่งเดียวแบบในตัวอย่างนี้ (เก็บใน field ความกว้างคงที่) เพิ่มความซับซ้อนของ
  โค้ดพอสมควร (ต้องคำนวณ offset เอง) — ควรใช้เทคนิคนี้เฉพาะเมื่อ**ปริมาณข้อมูลใหญ่พอที่จะเห็นผลต่างจริง**
  (ในระดับหลักแสนถึงล้าน record) สำหรับไฟล์ขนาดเล็กความซับซ้อนที่เพิ่มขึ้นอาจไม่คุ้มค่า
- ในสถานการณ์จริงบน Mainframe การปรับ Blocking Factor ทำผ่าน **JCL DCB parameter** (`BLKSIZE`) หรือ
  ค่า `RECFM`/`LRECL` ไม่ใช่การเขียนโค้ด COBOL จัดการเองแบบในตัวอย่างนี้ — ตัวอย่างนี้สาธิตหลักการ**พื้น
  ฐานเดียวกัน**ในระดับที่ GnuCOBOL ทดสอบได้จริงเท่านั้น

### แบบฝึกหัดที่ 676.1

**โจทย์**: จงอธิบายว่าทำไมการลดจำนวนครั้งที่เรียก I/O จึงเป็นหลักการที่ใช้ได้ทั้งบน GnuCOBOL และบน
Mainframe จริง ทั้งที่สถาปัตยกรรมภายในต่างกันโดยสิ้นเชิง

**เฉลยแนวทาง**: เพราะหลักการนี้ไม่ได้ขึ้นกับรายละเอียดของคอมไพเลอร์หรือฮาร์ดแวร์ แต่ขึ้นกับความจริงพื้น
ฐานของระบบคอมพิวเตอร์ทุกยุคทุกสมัยที่ว่า **การเข้าถึงอุปกรณ์เก็บข้อมูลภายนอก (ดิสก์) มีต้นทุนคงที่ต่อครั้ง
ที่สูงกว่าการประมวลผลในหน่วยความจำมาก** ไม่ว่าจะเป็นการเรียก system call บน Linux หรือการเรียก
Access Method บน z/OS ทั้งคู่ต้องผ่านขั้นตอนการสลับบริบท (context switch) และการจัดการ buffer ที่มี
ต้นทุนคงที่ในระดับของมันเอง การลดจำนวนครั้งจึงลดต้นทุนสะสมนี้ได้เสมอ ไม่ว่าจะรันบนสถาปัตยกรรมใด

---

## ขั้นตอนที่ 677: การ Tuning ระดับ Compiler ด้วย Optimization Flags ของ GnuCOBOL

### แฟล็กที่ cobc มีให้ใช้

`cobc` (ตัวคอมไพเลอร์ของ GnuCOBOL) มีแฟล็กควบคุมระดับการปรับแต่งโค้ด C ที่สร้างขึ้นก่อนส่งต่อให้ `gcc`
คอมไพล์เป็น native code:

| แฟล็ก | ความหมาย |
|---|---|
| `-O0` | ปิดการปรับแต่งทั้งหมด (ค่าที่ใช้ตอนพัฒนา/debug เพื่อให้ stack trace ตรงกับโค้ดต้นฉบับที่สุด) |
| `-O`, `-O2`, `-O3` | เปิดการปรับแต่งโค้ดระดับต่าง ๆ เพิ่มขึ้นตามลำดับ |
| `-Os` | ปรับแต่งเน้นขนาดไฟล์ execute เล็กที่สุด แทนที่จะเน้นความเร็วสูงสุด |

### ทดสอบ: โปรแกรมเดียวกัน (รวมเทคนิคจากขั้นตอนก่อนหน้า) คอมไพล์ด้วย -O0 เทียบกับ -O2

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP677OPT.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-ENTRY OCCURS 20000 TIMES
                   ASCENDING KEY IS PROD-CODE
                   INDEXED BY PT-IDX.
               10  PROD-CODE       PIC 9(6).
               10  PROD-NAME       PIC X(10).

       01  WS-FILL-IDX         PIC 9(5).
       01  WS-SEARCH-PASS      PIC 9(6).
       01  WS-SEARCH-KEY       PIC 9(6).
       01  WS-FOUND-COUNT      PIC 9(8) VALUE 0.
       01  WS-COUNTER          PIC 9(9) VALUE 0.
       01  WS-ACCUM            PIC 9(9) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM VARYING WS-FILL-IDX FROM 1 BY 1
                   UNTIL WS-FILL-IDX > 20000
               COMPUTE PROD-CODE(WS-FILL-IDX) =
                       100000 + WS-FILL-IDX - 1
               MOVE "ITEM" TO PROD-NAME(WS-FILL-IDX)
           END-PERFORM.

           PERFORM VARYING WS-SEARCH-PASS FROM 1 BY 1
                   UNTIL WS-SEARCH-PASS > 200000
               COMPUTE WS-SEARCH-KEY = 100000 +
                   FUNCTION MOD(WS-SEARCH-PASS * 37, 20000)
               SEARCH ALL PRODUCT-ENTRY
                   AT END
                       CONTINUE
                   WHEN PROD-CODE(PT-IDX) = WS-SEARCH-KEY
                       ADD 1 TO WS-FOUND-COUNT
               END-SEARCH
           END-PERFORM.

           PERFORM UNTIL WS-COUNTER >= 20000000
               ADD 1 TO WS-COUNTER
               ADD 1 TO WS-ACCUM
           END-PERFORM.

           DISPLAY "Done. found=" WS-FOUND-COUNT " accum=" WS-ACCUM.
           STOP RUN.
```

```bash
cobc -x -O0 -o step677_o0 step677_opt.cob
cobc -x -O2 -o step677_o2 step677_opt.cob
time ./step677_o0
time ./step677_o2
```

**ผลลัพธ์จริง (2 รอบการรันแต่ละแบบ):**

```
-O0 : 2.101s, 2.146s   (เฉลี่ย ~2.12s)
-O2 : 1.740s, 1.754s   (เฉลี่ย ~1.75s)
```

### อธิบายจุดสำคัญ

- เพียงแค่เพิ่มแฟล็ก `-O2` ตอนคอมไพล์ (**ไม่ต้องแก้โค้ด COBOL แม้แต่บรรทัดเดียว**) โปรแกรมทำงานเร็วขึ้น
  ประมาณ **17-18%** — นี่คือ "quick win" ที่ต้นทุนต่ำที่สุดในบทนี้ทั้งหมดเพราะไม่ต้องเปลี่ยนตรรกะโปรแกรม
  เลย
- `-O2` ทำงานโดยส่งต่อระดับการปรับแต่งไปยัง `gcc` ที่คอมไพล์โค้ด C ที่ `cobc` สร้างขึ้น ทำให้ได้ประโยชน์
  จากเทคนิคการปรับแต่งของ `gcc` เอง เช่น การขยายฟังก์ชันแบบ inline, การจัดสรร register ให้มีประสิทธิภาพ
  ขึ้น, การลบโค้ดที่ไม่จำเป็นออก
- **ข้อแลกเปลี่ยน**: `-O0` ทำให้ debug ง่ายกว่า (stack trace และตัวแปรตรงกับโค้ดต้นฉบับมากที่สุด) จึง
  เหมาะกับช่วงพัฒนา/ทดสอบ ส่วน `-O2` เหมาะกับโปรแกรม production ที่ผ่านการทดสอบแล้วและต้องการความเร็ว
  สูงสุด

### ข้อควรระวัง

- **ไม่ควรใช้ `-O2` ขณะกำลัง debug ปัญหา** เพราะ compiler อาจจัดเรียงหรือรวมโค้ดใหม่จนบรรทัดที่รายงาน
  ตอน error ไม่ตรงกับโค้ดต้นฉบับ 100% เสมอไป
- ต้นทุนของ `-O2` คือ**เวลา compile ที่นานขึ้นเล็กน้อย** (เพราะ compiler ต้องวิเคราะห์โค้ดเพิ่มเติม) แต่
  เนื่องจาก compile ทำเพียงครั้งเดียวในขณะที่ run อาจเกิดขึ้นนับพันนับหมื่นครั้งใน batch job จริง จึงคุ้มค่า
  เสมอสำหรับโปรแกรม production

### แบบฝึกหัดที่ 677.1

**โจทย์**: จงอธิบายว่าทำไมการเพิ่ม compiler flag อย่างเดียวจึงถือเป็นเทคนิค Performance Tuning ที่มี
"ต้นทุนความเสี่ยงต่ำที่สุด" เมื่อเทียบกับเทคนิคอื่น ๆ ในบทนี้ (เช่น การเปลี่ยนจาก SEARCH เป็น SEARCH ALL)

**เฉลยแนวทาง**: เพราะการเปลี่ยน compiler flag **ไม่ได้แก้ไขตรรกะหรือโครงสร้างข้อมูลของโปรแกรมเลย** จึง
ไม่มีความเสี่ยงที่จะทำให้พฤติกรรมของโปรแกรมเปลี่ยนไปหรือเกิดบั๊กใหม่ ในขณะที่การเปลี่ยนจาก `SEARCH` เป็น
`SEARCH ALL` ต้องเพิ่ม `ASCENDING KEY` และรับประกันว่าข้อมูลเรียงลำดับจริงก่อนค้นหาเสมอ (มีความเสี่ยงที่
จะลืมเงื่อนไขนี้และได้ผลลัพธ์ผิดพลาดแบบเงียบ ๆ ตามที่เตือนไว้ในขั้นตอนที่ 674) การเพิ่ม flag ตอนคอมไพล์
จึงควรเป็น**ขั้นตอนแรกที่ลองทำเสมอ**ก่อนจะไปแก้ไขโค้ดที่มีความเสี่ยงสูงกว่า

---

## ขั้นตอนที่ 678: [เชิงแนวคิด/z/OS เท่านั้น] การปรับแต่ง Buffer — BUFNO และ VSAM Buffer Tuning

> **หัวข้อนี้เป็นความรู้เชิงอ้างอิง (Reference/Conceptual) เท่านั้น** เนื้อหาต่อไปนี้อธิบายแนวคิดและไวยากรณ์
> JCL ที่ใช้จริงบน z/OS แต่**ไม่มีตัวอย่างใดในหัวข้อนี้ที่รันได้บน GnuCOBOL** เพราะ `BUFNO` และ VSAM
> Buffer Pool เป็นแนวคิดที่ผูกติดกับ Access Method ของ z/OS โดยตรง (QSAM/VSAM) ซึ่ง GnuCOBOL ไม่มี
> กลไกเทียบเท่าให้ทดสอบ ทบทวนหลักการเดียวกันในระดับที่ทดสอบได้จริงจากขั้นตอนที่ 676 (การลดจำนวนครั้งที่
> เรียก I/O)

### แนวคิด Buffer และ Blocking บน z/OS

เมื่อโปรแกรม COBOL บน Mainframe อ่าน/เขียนไฟล์ QSAM (Sequential) หรือ VSAM ระบบจะไม่อ่าน/เขียนดิสก์
ทีละ logical record จริง ๆ แต่จะอ่าน/เขียนเป็น **Physical Block** ที่รวมหลาย logical record เข้าด้วยกัน
(ตามค่า `BLKSIZE` ใน DCB) แล้วเก็บไว้ใน **Buffer** ในหน่วยความจำก่อน โปรแกรมจึงอ่านข้อมูลจาก Buffer
ในหน่วยความจำ (เร็วมาก) แทนที่จะต้องเข้าถึงดิสก์จริงทุกครั้ง

### พารามิเตอร์ BUFNO ใน JCL DD Statement

`BUFNO` (ทบทวนโครงสร้าง DD Statement จาก Part 052) กำหนด **จำนวน Buffer** ที่จะจองไว้ในหน่วยความจำ
สำหรับไฟล์นั้น:

```jcl
//SYSUT1   DD DSN=PROD.CUSTOMER.MASTER,DISP=SHR,
//            BUFNO=20
```

หลักการ (เชิงแนวคิด): ยิ่ง `BUFNO` มาก ยิ่งมี Buffer สำรองไว้ล่วงหน้ามาก ทำให้โปรแกรมที่อ่านข้อมูลแบบ
เรียงลำดับต่อเนื่อง (Sequential Access Pattern) ได้ประโยชน์จากการ **read-ahead** (ระบบอ่านข้อมูลล่วง
หน้าเข้า buffer ก่อนโปรแกรมจะร้องขอจริง) มากขึ้น แต่ก็แลกมาด้วยการใช้หน่วยความจำมากขึ้นตามไปด้วย

### VSAM Buffer Pool — BUFFERSPACE และ AMP

สำหรับไฟล์ VSAM (ทบทวนจาก Part 055-056) การปรับ Buffer ทำผ่านพารามิเตอร์ `AMP` (Access Method
Parameters) ใน JCL หรือกำหนดไว้ที่ตัว Cluster เองตอน `DEFINE CLUSTER` ด้วย IDCAMS:

```jcl
//SYSUT1   DD DSN=PROD.CUSTOMER.VSAM.KSDS,DISP=SHR,
//            AMP=('BUFND=10,BUFNI=5')
```

| พารามิเตอร์ | ความหมาย |
|---|---|
| `BUFND` | จำนวน Buffer สำหรับข้อมูล (Data Component) |
| `BUFNI` | จำนวน Buffer สำหรับดัชนี (Index Component) — สำคัญมากสำหรับ KSDS ที่ใช้ B-Tree index |

### ทำไมหัวข้อนี้จึงสำคัญแม้จะทดสอบไม่ได้ในหลักสูตรนี้

แม้เราจะพิสูจน์หลักการ "ลดจำนวนครั้งที่เรียก I/O" ได้จริงด้วย GnuCOBOL ในขั้นตอนที่ 676 แต่การ **ปรับแต่ง
Buffer ระดับ Access Method โดยตรง** (ไม่ใช่การเขียนโค้ดจัดการเอง) เป็นเครื่องมือ tuning ที่ **นักพัฒนา
Mainframe มืออาชีพใช้บ่อยมาก** เพราะทำได้จาก JCL โดยไม่ต้องแก้ไข แก้ไข หรือ compile โปรแกรม COBOL
ใหม่เลยแม้แต่บรรทัดเดียว — เป็นความรู้ที่จำเป็นสำหรับการทำงานกับระบบ Mainframe จริงในสายอาชีพ Systems
Programmer หรือ Performance Analyst

### ข้อควรระวัง

- การเพิ่ม `BUFNO`/`BUFND`/`BUFNI` มากเกินไปอาจทำให้ job ใช้หน่วยความจำเกินโควตาที่ได้รับจากระบบ
  (Region Size) จนทำให้ job ล้มเหลว — การปรับค่าเหล่านี้ต้องพิจารณาควบคู่กับทรัพยากรที่มีจริงเสมอ
- ค่าที่เหมาะสมขึ้นกับรูปแบบการเข้าถึงไฟล์ (Sequential ล้วน vs Random ปนกัน) ซึ่งต้องวิเคราะห์จาก SMF/RMF
  (ขั้นตอนถัดไป) ก่อนตัดสินใจปรับ ไม่ใช่การเดาค่าลอย ๆ

### แบบฝึกหัดที่ 678.1

**โจทย์**: จงอธิบายว่าแนวคิดของ `BUFNO` ใน JCL เกี่ยวข้องอย่างไรกับหลักการที่พิสูจน์ด้วยการรันจริงใน
ขั้นตอนที่ 676

**เฉลยแนวทาง**: ทั้งสองเรื่องมีรากฐานเดียวกันคือ **การลดจำนวนครั้งที่ต้องเข้าถึงดิสก์จริงต่อปริมาณข้อมูลที่
ประมวลผล** ขั้นตอนที่ 676 ทำสิ่งนี้ที่**ระดับโค้ด COBOL**โดยตรง (รวมหลาย logical record เข้าเป็น record
เดียวก่อนเขียน) ในขณะที่ `BUFNO`/VSAM Buffer ทำสิ่งนี้ที่**ระดับ Access Method ของระบบปฏิบัติการ**โดย
เก็บ block ข้อมูลไว้ใน buffer ล่วงหน้าโดยที่โปรแกรมเมอร์ไม่ต้องเขียนโค้ดจัดการเอง หลักการพื้นฐาน
("ลด I/O calls") เหมือนกันทุกประการ เพียงแค่ implement ในคนละชั้นของระบบ (Application Layer เทียบกับ
Access Method Layer)

---

## ขั้นตอนที่ 679: [เชิงแนวคิด/z/OS เท่านั้น] การวิเคราะห์ประสิทธิภาพด้วย SMF และ RMF

> **หัวข้อนี้เป็นความรู้เชิงอ้างอิง (Reference/Conceptual) เท่านั้น** SMF และ RMF เป็นเครื่องมือระดับ
> ระบบปฏิบัติการ z/OS ที่ไม่มีอยู่และไม่สามารถจำลองได้บน GnuCOBOL หรือเครื่อง Linux ทั่วไป เนื้อหาต่อไปนี้
> มีไว้เพื่อให้เข้าใจภาพรวมว่าในองค์กรที่มี Mainframe จริง ทีม Performance Analyst วิเคราะห์และตรวจสอบ
> ประสิทธิภาพระดับ System-wide กันอย่างไร

### SMF (System Management Facility) คืออะไร

**SMF** คือกลไกของ z/OS ที่**บันทึก (log) ข้อมูลทุกกิจกรรมสำคัญที่เกิดขึ้นในระบบ** ลงเป็น "SMF Records"
อย่างต่อเนื่อง เช่น:

- เวลาที่แต่ละ Job เริ่มและจบ, CPU time ที่ใช้, I/O count ที่เกิดขึ้น
- การเข้าถึงไฟล์แต่ละไฟล์ (Dataset Access)
- กิจกรรมของ CICS Transaction แต่ละรายการ (เชื่อมโยงกับ Part 061-064)
- ข้อมูลการใช้ทรัพยากรเพื่อการ **Chargeback** (คิดค่าใช้จ่ายทรัพยากรคอมพิวเตอร์ให้แต่ละหน่วยงานในองค์กร
  ตามการใช้งานจริง — แนวคิดนี้สำคัญมากในองค์กรใหญ่ที่แชร์ Mainframe เครื่องเดียวกันหลายแผนก)

ข้อมูล SMF ถูกเก็บเป็น **SMF Record Type** ต่าง ๆ (แยกตามหมวดหมู่ด้วยตัวเลข เช่น Type 30 = Job/Step
information, Type 42 = Storage/dataset activity) แล้วนำไปวิเคราะห์ต่อด้วยเครื่องมือเฉพาะทาง

### RMF (Resource Measurement Facility) คืออะไร

**RMF** คือเครื่องมือที่ **อ่านข้อมูลจาก SMF (และแหล่งข้อมูลระบบอื่น) มาสรุปเป็นรายงานประสิทธิภาพระดับ
System แบบเรียลไทม์หรือย้อนหลัง** เช่น:

- **CPU Utilization** — ใช้ CPU ไปกี่เปอร์เซ็นต์ในแต่ละช่วงเวลา
- **I/O Response Time** — เวลาเฉลี่ยที่แต่ละคำขอ I/O ใช้ในการตอบกลับ, แยกตาม Device/Volume
- **Paging Rate** — อัตราการสลับหน่วยความจำเข้า-ออกดิสก์ (Virtual Memory Paging) — ถ้าสูงเกินไปแปลว่า
  ระบบขาดหน่วยความจำจริง
- **Workflow Manager (WLM) Reports** — วิเคราะห์ว่า Job/Transaction แต่ละกลุ่มได้รับทรัพยากรตรงตาม
  เป้าหมาย (Service Level) ที่กำหนดไว้หรือไม่

### ความสัมพันธ์ระหว่าง SMF/RMF กับการ Tuning โปรแกรม COBOL

เมื่อ Performance Analyst พบจาก RMF ว่า Batch Job หนึ่งใช้เวลานานผิดปกติ ขั้นตอนต่อไปมักเป็น:

1. ดูรายงาน SMF Type 30 ของ Job นั้นเพื่อดู CPU time เทียบกับ Elapsed time — ถ้า Elapsed สูงกว่า CPU
   มากแสดงว่า Job ส่วนใหญ่ใช้เวลา**รอ I/O** ไม่ใช่รอ CPU ประมวลผล (ตรงกับหลักการที่สอนใน Part 671:
   ความแตกต่างระหว่าง `real` time กับ `user` time)
2. ถ้าปัญหาคือ I/O-bound มักไปตรวจสอบ `BUFNO`/VSAM Buffer parameters ต่อ (ขั้นตอนที่ 678)
3. ถ้าปัญหาคือ CPU-bound มักไปตรวจสอบโค้ด COBOL เองว่ามีอัลกอริทึมที่ไม่มีประสิทธิภาพหรือไม่ (เช่น
   Linear Search ในตารางใหญ่ — ตรงกับขั้นตอนที่ 673-674)

**ข้อสังเกตสำคัญ**: หลักการวินิจฉัยปัญหาแบบนี้ (แยก CPU-bound กับ I/O-bound ก่อนตัดสินใจว่าจะแก้ตรงไหน)
คือ**หลักการเดียวกันกับที่เราใช้ตลอด Part นี้** เพียงแต่บน Mainframe จริงมีเครื่องมือ (SMF/RMF) ที่ให้
ข้อมูลละเอียดระดับ System-wide ในขณะที่การฝึกฝนของเราใช้คำสั่ง `time` ในระดับโปรแกรมเดี่ยว

### ข้อควรระวัง

- การอ่านและตีความรายงาน RMF อย่างถูกต้องต้องอาศัยประสบการณ์และความเข้าใจสถาปัตยกรรม z/OS อย่างลึกซึ้ง
  เป็นทักษะเฉพาะทางที่มักแยกเป็นตำแหน่งงาน **Performance Analyst** หรือ **Capacity Planner** ต่างหาก
  จากตำแหน่ง Application Developer
- องค์กรขนาดใหญ่มักมีทีม Systems Programming ดูแลการตั้งค่าและวิเคราะห์ SMF/RMF โดยเฉพาะ นักพัฒนา
  COBOL ทั่วไปมักไม่ต้องตั้งค่าเครื่องมือเหล่านี้เอง แต่ควร**เข้าใจว่ารายงานเหล่านี้บอกอะไร** เพื่อสื่อสารกับ
  ทีม Performance ได้อย่างมีประสิทธิภาพเมื่อโปรแกรมของตนถูกระบุว่าเป็นสาเหตุของปัญหา

### แบบฝึกหัดที่ 679.1

**โจทย์**: จงอธิบายว่าทำไมการแยกแยะว่าปัญหาประสิทธิภาพของ Job หนึ่งเป็น "CPU-bound" หรือ "I/O-bound"
จึงเป็นขั้นตอนแรกที่สำคัญที่สุดก่อนจะเริ่ม tuning โปรแกรม COBOL ของ Job นั้น

**เฉลยแนวทาง**: เพราะเทคนิคการแก้ปัญหาทั้งสองแบบแตกต่างกันโดยสิ้นเชิง ถ้าปัญหาเป็น CPU-bound (โปรแกรม
ใช้เวลาคำนวณนาน) การแก้ที่ถูกต้องคือปรับปรุงอัลกอริทึมหรือชนิดข้อมูล (เช่นขั้นตอนที่ 673-675) แต่ถ้าปัญหา
เป็น I/O-bound (โปรแกรมใช้เวลารอดิสก์นาน) การไปปรับปรุงอัลกอริทึมคำนวณจะไม่ช่วยอะไรเลยเพราะ CPU ไม่ใช่
คอขวด ต้องไปปรับ Buffer/Blocking แทน (ขั้นตอนที่ 676, 678) การ tuning ผิดจุดไม่เพียงเสียเวลาพัฒนาโดย
เปล่าประโยชน์ แต่ยังอาจทำให้โค้ดซับซ้อนขึ้นโดยไม่ได้ประโยชน์ด้านความเร็วเพิ่มขึ้นเลย ตรงกับหลักการ "วัดผล
ก่อนเชื่อ" ที่เป็นแก่นของ Part นี้ทั้งบท

---

## ขั้นตอนที่ 680: สรุปรวม — Checklist และกรณีศึกษาก่อน-หลังการ Tuning แบบครบวงจร

### Performance Tuning Checklist สำหรับโปรแกรม COBOL (เรียงตามลำดับที่ควรทำก่อน-หลัง)

| ลำดับ | เทคนิค | ต้นทุน/ความเสี่ยง | ผลที่คาดหวัง (จากการทดลองใน Part นี้) |
|---|---|---|---|
| 1 | วัดผล baseline ก่อนแก้อะไรเลย (ขั้นตอน 671) | ต่ำมาก | รู้ว่าจุดไหนคือคอขวดจริง |
| 2 | ลองเพิ่ม compiler optimization flag (ขั้นตอน 677) | ต่ำมาก ไม่แก้โค้ด | ~17-20% |
| 3 | ตรวจหาตารางที่ค้นหาด้วย Linear SEARCH ขนาดใหญ่ เปลี่ยนเป็น SEARCH ALL (ขั้นตอน 673-674) | ปานกลาง ต้องเรียงข้อมูลก่อน | สูงถึงหลายสิบ-ร้อยเท่า (ขึ้นกับขนาดตาราง) |
| 4 | ลดจำนวนครั้งที่เรียก I/O ด้วยการ batch ข้อมูล (ขั้นตอน 676) | ปานกลาง เพิ่มความซับซ้อนโค้ด | ~30-40% |
| 5 | เลือกชนิดข้อมูลให้ตรงกับรูปแบบการคำนวณจริง (ขั้นตอน 675) | ต่ำ แต่ต้องทดสอบเฉพาะจุด | ต่างกันได้ถึง 2 เท่า (ทั้งสองทิศทาง) |
| 6 | ตรวจสอบ `PERFORM`/`GO TO` เพื่อความอ่านง่าย (ไม่ใช่เพื่อความเร็ว — ขั้นตอน 672) | - | ไม่มีผลด้านความเร็ว แต่ช่วยดูแลรักษา |
| 7 | (เฉพาะ Mainframe จริง) ปรับ BUFNO/VSAM Buffer ตามรายงาน SMF/RMF (ขั้นตอน 678-679) | ต้องผู้เชี่ยวชาญเฉพาะทาง | ขึ้นกับสภาพแวดล้อมจริง |

สังเกตว่าลำดับนี้เรียงจาก **ต้นทุน/ความเสี่ยงต่ำไปสูง** เสมอ — หลักการมืออาชีพคือทำสิ่งที่ปลอดภัยและได้
ผลเร็วก่อน แล้วค่อยลงลึกไปยังสิ่งที่ซับซ้อนขึ้นเมื่อจำเป็นจริง ๆ เท่านั้น

### กรณีศึกษาครบวงจร: รวมหลายเทคนิคเข้าด้วยกันในโปรแกรมเดียว

**เวอร์ชัน "Naive"** (ไม่ใช้เทคนิคใดเลย): สร้างตาราง 20,000 รายการ, ค้นหา 30,000 ครั้งด้วย Linear
`SEARCH` (กรณีเลวร้ายที่สุด), เขียนไฟล์ผลลัพธ์ 300,000 ค่าด้วย `WRITE` ทีละ record

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP680NAIVE.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT OUT-FILE ASSIGN TO "NAIVE680.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  OUT-FILE.
       01  OUT-RECORD          PIC X(10).

       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-ENTRY OCCURS 20000 TIMES
                   INDEXED BY PT-IDX.
               10  PROD-CODE       PIC 9(6).
               10  PROD-NAME       PIC X(10).

       01  WS-FILL-IDX         PIC 9(7).
       01  WS-LOOKUP-PASS      PIC 9(6).
       01  WS-SEARCH-KEY       PIC 9(6).
       01  WS-FOUND-COUNT      PIC 9(8) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM VARYING WS-FILL-IDX FROM 1 BY 1
                   UNTIL WS-FILL-IDX > 20000
               COMPUTE PROD-CODE(WS-FILL-IDX) =
                       100000 + WS-FILL-IDX - 1
               MOVE "ITEM" TO PROD-NAME(WS-FILL-IDX)
           END-PERFORM.

      *> Technique NOT applied: unsorted linear SEARCH, biased
      *> toward the far end of the table (worst case).
           PERFORM VARYING WS-LOOKUP-PASS FROM 1 BY 1
                   UNTIL WS-LOOKUP-PASS > 30000
               COMPUTE WS-SEARCH-KEY = 100000 + 19999 -
                   FUNCTION MOD(WS-LOOKUP-PASS, 50)
               SET PT-IDX TO 1
               SEARCH PRODUCT-ENTRY
                   AT END
                       CONTINUE
                   WHEN PROD-CODE(PT-IDX) = WS-SEARCH-KEY
                       ADD 1 TO WS-FOUND-COUNT
               END-SEARCH
           END-PERFORM.

      *> Technique NOT applied: one WRITE per tiny record.
           OPEN OUTPUT OUT-FILE.
           PERFORM VARYING WS-FILL-IDX FROM 1 BY 1
                   UNTIL WS-FILL-IDX > 300000
               MOVE WS-FILL-IDX TO OUT-RECORD
               WRITE OUT-RECORD
           END-PERFORM.
           CLOSE OUT-FILE.

           DISPLAY "NAIVE version done. found=" WS-FOUND-COUNT.
           STOP RUN.
```

**เวอร์ชัน "Tuned"** (ใช้เทคนิคทุกข้อจากขั้นตอน 673-677 พร้อมกัน): ตารางเดียวกันแต่เรียงลำดับด้วย
`ASCENDING KEY` แล้วใช้ `SEARCH ALL`, เขียนไฟล์แบบ batch 50 ค่าต่อ record, คอมไพล์ด้วย `-O2`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP680TUNED.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT OUT-FILE ASSIGN TO "TUNED680.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  OUT-FILE.
       01  OUT-RECORD          PIC X(500).

       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-ENTRY OCCURS 20000 TIMES
                   ASCENDING KEY IS PROD-CODE
                   INDEXED BY PT-IDX.
               10  PROD-CODE       PIC 9(6).
               10  PROD-NAME       PIC X(10).

       01  WS-FILL-IDX         PIC 9(7).
       01  WS-LOOKUP-PASS      PIC 9(6).
       01  WS-SEARCH-KEY       PIC 9(6).
       01  WS-FOUND-COUNT      PIC 9(8) VALUE 0.
       01  WS-PIECE            PIC X(10).
       01  WS-GROUP-IDX        PIC 9(3)  VALUE 0.
       01  WS-OFFSET           PIC 9(4)  VALUE 1.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM VARYING WS-FILL-IDX FROM 1 BY 1
                   UNTIL WS-FILL-IDX > 20000
               COMPUTE PROD-CODE(WS-FILL-IDX) =
                       100000 + WS-FILL-IDX - 1
               MOVE "ITEM" TO PROD-NAME(WS-FILL-IDX)
           END-PERFORM.

      *> Technique applied: table is ASCENDING KEY, SEARCH ALL does
      *> a binary search instead of a linear one.
           PERFORM VARYING WS-LOOKUP-PASS FROM 1 BY 1
                   UNTIL WS-LOOKUP-PASS > 30000
               COMPUTE WS-SEARCH-KEY = 100000 + 19999 -
                   FUNCTION MOD(WS-LOOKUP-PASS, 50)
               SEARCH ALL PRODUCT-ENTRY
                   AT END
                       CONTINUE
                   WHEN PROD-CODE(PT-IDX) = WS-SEARCH-KEY
                       ADD 1 TO WS-FOUND-COUNT
               END-SEARCH
           END-PERFORM.

      *> Technique applied: batch 50 logical values per WRITE call.
           OPEN OUTPUT OUT-FILE.
           PERFORM VARYING WS-FILL-IDX FROM 1 BY 1
                   UNTIL WS-FILL-IDX > 300000
               MOVE WS-FILL-IDX TO WS-PIECE
               MOVE WS-PIECE TO OUT-RECORD(WS-OFFSET:10)
               ADD 1 TO WS-GROUP-IDX
               ADD 10 TO WS-OFFSET
               IF WS-GROUP-IDX = 50
                   WRITE OUT-RECORD
                   MOVE SPACES TO OUT-RECORD
                   MOVE 1 TO WS-OFFSET
                   MOVE 0 TO WS-GROUP-IDX
               END-IF
           END-PERFORM.
           IF WS-GROUP-IDX > 0
               WRITE OUT-RECORD
           END-IF.
           CLOSE OUT-FILE.

           DISPLAY "TUNED version done. found=" WS-FOUND-COUNT.
           STOP RUN.
```

```bash
cobc -x -O2 -o step680_naive step680_naive.cob
cobc -x -O2 -o step680_tuned step680_tuned.cob
time ./step680_naive
time ./step680_tuned
```

**ผลลัพธ์จริง (2 รอบการรันแต่ละแบบ):**

```
NAIVE version done. found=00030000   real 0m0.368s, 0m0.387s   (เฉลี่ย ~0.38s)
TUNED version done. found=00030000   real 0m0.069s, 0m0.061s   (เฉลี่ย ~0.065s)
```

**ผลรวม: เร็วขึ้นประมาณ 5.8 เท่า** เมื่อนำหลายเทคนิคมาใช้ร่วมกัน (ทั้งสองเวอร์ชันให้ผลลัพธ์ทางธุรกิจ
เหมือนกันทุกประการ — `found=00030000` เท่ากัน — พิสูจน์ว่าการ tuning ไม่ได้เปลี่ยนความถูกต้องของผลลัพธ์
เลย ซึ่งเป็นเงื่อนไขที่**สำคัญที่สุด**ของการ tuning ที่ดี)

### บันทึกเรื่องบั๊กที่พบระหว่างเตรียมตัวอย่างนี้ (บทเรียนเสริมที่มีค่า)

ระหว่างเตรียมตัวอย่างในขั้นตอนนี้ พบบั๊กจริงที่คุ้มค่าแก่การเล่าไว้: เวอร์ชันแรกของโค้ด "Naive" ประกาศ
`WS-FILL-IDX` เป็น `PIC 9(5)` (เก็บได้สูงสุด 99999) แต่นำตัวแปรเดียวกันไปใช้เป็นตัวนับในลูปเขียนไฟล์ที่
ต้องวนถึง 300,000 — เมื่อค่าถึง 100,000 การ**ตัดหลักสูงทิ้งแบบเงียบ ๆ** (ทบทวนจาก Part 008 ขั้นตอนที่
72 และ Part 015 ขั้นตอนที่ 142) ทำให้ค่าเวียนกลับไปที่ 0 อีกครั้งแทนที่จะเพิ่มขึ้นต่อ ผลคือเงื่อนไข
`UNTIL WS-FILL-IDX > 300000` **ไม่มีวันเป็นจริง** กลายเป็นลูปไม่รู้จบ (โปรแกรมค้างจริง วัดได้ด้วย `time`
ว่าเกิน 60 วินาทีโดยไม่จบ) การแก้ไขคือขยายเป็น `PIC 9(7)` (รองรับถึง 9,999,999) เท่านั้น

**นี่คือตัวอย่างจริงที่แสดงให้เห็นว่ากับดักพื้นฐานที่สุดของหลักสูตร (silent truncation) ยังคงเป็นสาเหตุของ
บั๊กร้ายแรงได้แม้ในหัวข้อขั้นสูงอย่าง Performance Tuning** — ยืนยันว่าการเข้าใจกฎพื้นฐานให้แน่นตั้งแต่ต้น
หลักสูตรมีความสำคัญไม่ลดลงเลยแม้จะเรียนมาถึงเนื้อหาขั้นสูงแล้วก็ตาม

### ข้อควรระวัง (สรุปรวมทั้งบท)

- **อย่า tune ก่อนวัดผล** และ **อย่าเชื่อผลจากการรันเพียงครั้งเดียว** — หลักการทั้งสองข้อนี้คือแก่นของ
  Performance Tuning ที่น่าเชื่อถือ
- **ตรวจสอบผลลัพธ์ทางธุรกิจให้ตรงกันเสมอ**ก่อนและหลัง tuning (ในกรณีศึกษานี้คือ `found=00030000`
  เท่ากันทั้งสองเวอร์ชัน) — โปรแกรมที่ "เร็วขึ้นแต่ผลลัพธ์ผิด" ไม่มีค่าใดๆ เลยในโลกธุรกิจจริง
- ตัวเลขทั้งหมดในบทนี้เฉพาะเจาะจงกับ GnuCOBOL บนเครื่องที่ใช้ทดสอบ — เมื่อนำไปใช้กับ IBM Enterprise
  COBOL บน z/OS จริง ให้ยึดหลักการ (algorithmic thinking, measure-then-tune, reduce I/O calls) เป็น
  หลัก แล้ว**วัดผลใหม่ในสภาพแวดล้อมนั้นเสมอ** ไม่ใช่นำตัวเลขจาก Part นี้ไปอ้างอิงตรง ๆ

### แบบฝึกหัดที่ 680.1

**โจทย์**: จากกรณีศึกษาในขั้นตอนนี้ จงเรียงลำดับว่าเทคนิคใดใน 3 เทคนิค (SEARCH ALL, batch I/O, -O2)
น่าจะมีส่วนสนับสนุนผลต่าง 5.8 เท่ามากที่สุด และอธิบายเหตุผลโดยอ้างอิงตัวเลขจากขั้นตอนก่อนหน้า

**เฉลยแนวทาง**: จากขั้นตอนที่ 674 การเปลี่ยนจาก Linear เป็น Binary Search ให้ผลต่างสูงถึง ~107 เท่า
สำหรับตารางขนาด 20,000 รายการ ซึ่งมากกว่าผลต่างของเทคนิคอื่นทั้งหมดในบทนี้รวมกันอย่างชัดเจน (batch I/O
ให้ผลต่าง ~1.5 เท่าจากขั้นตอนที่ 676, `-O2` ให้ผลต่าง ~1.2 เท่าจากขั้นตอนที่ 677) ดังนั้น **การเปลี่ยน
SEARCH เป็น SEARCH ALL น่าจะเป็นปัจจัยหลักที่สุด**ของผลต่าง 5.8 เท่าที่วัดได้ในกรณีศึกษานี้ แม้ผลรวมจริง
จะไม่ใช่การคูณตัวเลขทั้งสามเข้าด้วยกันตรง ๆ (เพราะแต่ละเทคนิคส่งผลต่อคนละส่วนของโปรแกรมที่ใช้เวลาไม่
เท่ากัน) แต่หลักฐานเชิงตัวเลขจากขั้นตอนก่อนหน้าชี้ชัดว่าการปรับปรุงระดับอัลกอริทึม (algorithmic) ยังคงเป็น
เทคนิคที่ทรงพลังที่สุดเสมอเมื่อนำมาเปรียบเทียบกับเทคนิคระดับ micro-optimization อื่น ๆ

---

## สรุปท้ายบท

Part นี้พาเราเจาะลึกเรื่อง Performance Tuning สำหรับ COBOL ด้วยแนวทาง **"วัดผลก่อนเชื่อ"** อย่าง
เคร่งครัดตลอดทั้งบท สิ่งที่ได้เรียนรู้มีทั้งเทคนิคที่ทดสอบได้จริงด้วย GnuCOBOL และความรู้เชิงอ้างอิงสำหรับ
สภาพแวดล้อม z/OS จริง:

**เทคนิคที่พิสูจน์ด้วยการรันจริง (ทดสอบได้กับ GnuCOBOL):**

- ระเบียบวิธีวัดผลด้วยคำสั่ง `time` และหลักการ Measure → Identify → Fix → Measure Again
- `PERFORM` ไม่ได้ช้ากว่า `GO TO` เลยบนคอมไพเลอร์สมัยใหม่ — เลือกใช้เพื่อความอ่านง่าย ไม่ใช่เพื่อความเร็ว
- `SEARCH` (Linear) เทียบกับ `SEARCH ALL` (Binary) — ความแตกต่างระดับ **~100 เท่า** สำหรับตารางขนาด
  ใหญ่ เป็นเทคนิคที่ทรงพลังที่สุดในบทนี้เพราะเป็นการปรับปรุงระดับอัลกอริทึม
- ชนิดข้อมูล `DISPLAY`/`COMP`/`COMP-3` — ผลลัพธ์**ขึ้นกับรูปแบบการคำนวณ**อย่างมาก และอาจสวนทาง
  ความเชื่อดั้งเดิมของวงการ Mainframe เมื่อรันบน GnuCOBOL
- การลดจำนวนครั้งที่เรียก I/O ด้วยการ batch ข้อมูล — ปรับปรุงได้ **~30-40%**
- Compiler optimization flag (`-O2`) — "quick win" ต้นทุนต่ำที่ควรลองก่อนเสมอ

**ความรู้เชิงอ้างอิงสำหรับ z/OS จริง (ไม่สามารถทดสอบในสภาพแวดล้อมนี้):**

- `BUFNO` และ VSAM Buffer Pool (`BUFND`/`BUFNI`) ใน JCL — ปรับ I/O buffering ระดับ Access Method
- SMF และ RMF — เครื่องมือวิเคราะห์ประสิทธิภาพระดับ System ที่ใช้แยกปัญหา CPU-bound กับ I/O-bound

Part ถัดไป (Part 069) จะเปลี่ยนโฟกัสไปที่อีกมิติสำคัญของงาน Mainframe: **ความปลอดภัย (Security)**
ผ่านแนวคิด RACF ที่ใช้ควบคุมสิทธิ์การเข้าถึงทรัพยากรบน z/OS พร้อมกับทบทวน secure coding practice
ระดับโค้ด COBOL ที่ทดสอบได้จริงควบคู่กันไป

**[← กลับไป Part 067: DFSORT และ Sort Utilities ขั้นสูง](part-067-dfsort-advanced.md)** |
**[ไปยัง Part 069: Security บน Mainframe (RACF Concepts) →](part-069-mainframe-security.md)**
