# Part 088: Performance Optimization ระดับสูง (ขั้นตอนที่ 871–880)

## คำนำของ Part นี้

Part 068 สอนหลักการ Performance Tuning พื้นฐานที่จำเป็นสำหรับ COBOL บน Mainframe แล้ว — เราพิสูจน์ว่า
`PERFORM` ไม่ได้ช้ากว่า `GO TO`, พิสูจน์ว่า `SEARCH ALL` เร็วกว่า `SEARCH` แบบ Linear ราว 107 เท่าบนตาราง
20,000 รายการ, พิสูจน์ว่า `COMP-3` ไม่ได้เร็วกว่า `DISPLAY` เสมอไปบน GnuCOBOL, และสอนเรื่องการลดจำนวน
ครั้งของการเรียก I/O ทุกอย่างนี้ยังคงเป็นความจริงและเป็นพื้นฐานสำคัญ **แต่ Part นี้จะพาไปไกลกว่านั้นอีกขั้น**

Part 088 มี 3 สิ่งที่แตกต่างจาก Part 068 อย่างชัดเจน:

1. **สเกลของข้อมูลใหญ่ขึ้น** — Part 068 ทดสอบตาราง 20,000 รายการ Part นี้ขยายเป็น 50,000 รายการ และ
   เพิ่ม**คู่แข่งตัวที่สาม**ที่ Part 068 ไม่เคยพูดถึงเลย: **การอ่านไฟล์ Indexed (ISAM) โดยตรงด้วยคีย์** —
   คำถามที่เราจะตอบคือ "ระหว่างตารางในหน่วยความจำกับไฟล์บนดิสก์ อันไหนเหมาะกับงานแบบไหน?"
2. **มุมมองใหม่ที่ Part 068 ไม่ได้แตะเลย** — Working Set/หน่วยความจำแคช, กลยุทธ์การ cache ข้อมูลในหน่วย
   ความจำเพื่อลด I/O ซ้ำซ้อน, การ optimize นิพจน์ COMPUTE ในระดับที่ละเอียดขึ้น, และการสร้าง "Profiler"
   อย่างง่ายด้วยโค้ด COBOL เองโดยไม่ต้องพึ่งเครื่องมือภายนอกอย่าง `time`
3. **มายาคติใหม่ที่ถูกหักล้างด้วยตัวเลขจริงอีกครั้ง** — เหมือนที่ Part 068 หักล้างความเชื่อเรื่อง `GO TO` และ
   `COMP-3` Part นี้จะแสดงให้เห็นว่าแม้แต่การ "optimize" นิพจน์ COMPUTE ด้วยมือ (เก็บค่าซ้ำไว้ในตัวแปรชั่วคราว)
   ก็**อาจทำให้โปรแกรมช้าลง**ได้จริงในบางกรณี — บทเรียนเดิมจาก Part 068 ยังใช้ได้: **วัดผลก่อนเชื่อเสมอ**

> **ย้ำระเบียบวิธีจาก Part 068**: ตัวเลขเวลาทั้งหมดใน Part นี้วัดจากการรันจริงด้วยคำสั่ง `time` ของ Linux
> บนเครื่องเดียวกัน (GnuCOBOL 4.0-early-dev, สถาปัตยกรรม x86_64) รันซ้ำอย่างน้อย 2 ครั้งต่อกรณีเพื่อยืนยัน
> ความเสถียรของตัวเลข ก่อนนำมาแสดงผล ตัวเลขที่ได้เป็น**หลักฐานเชิงประจักษ์จากสภาพแวดล้อมนี้เท่านั้น**
> แต่ **อัตราส่วนและทิศทางของความแตกต่าง** คือบทเรียนที่นำไปใช้ได้ทั่วไป

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน (comment, ชื่อตัวแปร,
> ข้อความใน DISPLAY) และทุกตัวอย่างผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL แล้วทุกตัวอย่าง — รวมถึงตัวอย่าง
> ที่ค้นพบบั๊กจริงระหว่างการเตรียม Part นี้ (ดูขั้นตอนที่ 872) ซึ่งกลายเป็นบทเรียนสำคัญของ Part นี้ไปด้วย

---

## ขั้นตอนที่ 871: กรอบคิดเรื่องความซับซ้อนเชิงอัลกอริทึม (Algorithmic Complexity) ในโค้ด COBOL

### ทบทวนสั้น ๆ จาก Part 068 ขั้นตอนที่ 674

Part 068 แนะนำ Big-O Notation ผ่านตัวอย่าง `SEARCH` (O(n)) กับ `SEARCH ALL` (O(log n)) แล้ว บทเรียนสำคัญ
ที่สุดจากตรงนั้นคือ **"การปรับปรุงระดับอัลกอริทึมมีพลังมากกว่าการปรับปรุงระดับ syntax เล็กน้อยเสมอ"** —
Part นี้จะขยายกรอบคิดนั้นให้กว้างขึ้น: ไม่ใช่แค่ `SEARCH` กับ `SEARCH ALL` เท่านั้นที่มีความซับซ้อนต่างกัน
**โครงสร้างลูปธรรมดาที่นักพัฒนา COBOL เขียนกันทุกวันก็สามารถซ่อนปัญหา O(n²) ไว้ได้โดยไม่รู้ตัว**

### รูปแบบที่พบบ่อยที่สุดของ O(n²) ในโค้ด COBOL: ลูปซ้อนลูปเพื่อเปรียบเทียบข้อมูลกันเอง

งานประเภท "หาข้อมูลซ้ำ" หรือ "จับคู่ข้อมูลระหว่างสอง record" เป็นงานที่พบบ่อยมากในโปรแกรม Batch — และเป็น
จุดที่นักพัฒนามือใหม่มักเขียนลูปซ้อนลูปโดยไม่ทันคิดถึงผลกระทบเมื่อข้อมูลโต ตัวอย่างต่อไปนี้จำลองงาน
"นับจำนวนธุรกรรมที่ใช้เลขบัญชีซ้ำกับธุรกรรมอื่นในชุดข้อมูลเดียวกัน" ด้วยลูปซ้อนลูปแบบตรงไปตรงมาที่สุด:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP871NESTED.
       AUTHOR. COBOL-COURSE.
      *> Classic O(n^2) trap: for every transaction, scan every OTHER
      *> transaction to look for a matching account number. This is
      *> the pattern behind many slow COBOL batch programs written
      *> without thinking about growth rate.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  TXN-TABLE.
           05  TXN-ENTRY OCCURS 3000 TIMES.
               10  TXN-ACCOUNT      PIC 9(6).
       01  WS-I                 PIC 9(5).
       01  WS-J                 PIC 9(5).
       01  WS-MATCH-COUNT       PIC 9(9) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Fill the table so account numbers repeat every 500 entries -
      *> guarantees real matches to count, not just zeros.
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 3000
               COMPUTE TXN-ACCOUNT(WS-I) =
                   100000 + FUNCTION MOD(WS-I, 500)
           END-PERFORM.

      *> O(n^2): for each of the 3,000 entries, compare against all
      *> 3,000 entries again - 9,000,000 comparisons total.
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 3000
               PERFORM VARYING WS-J FROM 1 BY 1 UNTIL WS-J > 3000
                   IF WS-I NOT = WS-J
                       IF TXN-ACCOUNT(WS-I) = TXN-ACCOUNT(WS-J)
                           ADD 1 TO WS-MATCH-COUNT
                       END-IF
                   END-IF
               END-PERFORM
           END-PERFORM.

           DISPLAY "O(n^2) NESTED SCAN: matches=" WS-MATCH-COUNT.
           STOP RUN.
```

```bash
cobc -x -o step871_nestedloop step871_nestedloop.cob
time ./step871_nestedloop
```

**ผลลัพธ์จริง (n=3,000, 2 รอบการรัน):**

```
O(n^2) NESTED SCAN: matches=000015000
real	0m0.652s
real	0m0.649s
```

### พิสูจน์อัตราการเติบโตแบบ n² ด้วยตัวเลขจริง: เพิ่มขนาดข้อมูลเป็น 2 เท่า

หัวใจของ Big-O ไม่ใช่แค่ "จำนวนวินาทีที่ใช้ ณ ขนาดหนึ่ง" แต่คือ **อัตราการเติบโตของเวลาเมื่อข้อมูลโตขึ้น**
ทดสอบเดียวกันทุกประการ เปลี่ยนแค่ `3000` เป็น `6000` ทุกจุด (ตัวแปรและ `OCCURS`):

```bash
cobc -x -o step871_nestedloop6k step871_nestedloop6k.cob
time ./step871_nestedloop6k
```

**ผลลัพธ์จริง (n=6,000, 2 รอบการรัน):**

```
O(n^2) NESTED SCAN: matches=000066000
real	0m2.596s
real	0m2.613s
```

### วิเคราะห์ผล — ข้อมูลโตเป็น 2 เท่า แต่เวลาโตเป็น 4 เท่า

| ขนาดข้อมูล (n) | จำนวนครั้งเปรียบเทียบ (n²) | เวลาที่ใช้ | อัตราส่วนเทียบกับ n=3,000 |
|---|---|---|---|
| 3,000 | 9,000,000 | ~0.65 วินาที | 1× |
| 6,000 | 36,000,000 | ~2.60 วินาที | **~4×** |

ข้อมูลโตเป็น **2 เท่า** แต่เวลาที่ใช้โตเป็น **4 เท่า (2²)** พอดี — นี่คือลายเซ็นของความซับซ้อนแบบ **O(n²)**
ที่ต้องจำให้ขึ้นใจ: **ทุกครั้งที่ข้อมูลโตเป็น 2 เท่า เวลาจะโตเป็น 4 เท่า, โตเป็น 10 เท่า เวลาจะโตเป็น 100 เท่า**
นี่คือเหตุผลที่โปรแกรม Batch ที่เคยรันเสร็จใน 5 นาทีตอนข้อมูลมี 10,000 รายการ อาจใช้เวลาเป็นชั่วโมงเมื่อ
ข้อมูลโตเป็น 100,000 รายการ ถ้าโค้ดภายในมีรูปแบบลูปซ้อนลูปแบบนี้ซ่อนอยู่

### ทางแก้ที่ถูกต้อง: อย่าเปรียบเทียบกันเอง ให้จัดกลุ่มหรือเรียงลำดับก่อน

งาน "หาข้อมูลซ้ำ" ไม่จำเป็นต้องเป็น O(n²) เลย — ขั้นตอนที่ 874-875 จะพิสูจน์ด้วยตัวเลขจริงว่าการเปลี่ยนไปใช้
`SEARCH ALL` (ต้องเรียงลำดับก่อน, Part 027 สอน `SORT`) หรือไฟล์ Indexed ที่มีโครงสร้าง B-Tree ในตัว
สามารถลดความซับซ้อนจาก O(n²) เหลือ O(n log n) ได้ ซึ่งเมื่อข้อมูลมีขนาดใหญ่ระดับแสนหรือล้านรายการ
ความแตกต่างจะไม่ใช่แค่ "เร็วขึ้น" แต่คือ **"ทำงานเสร็จทันเวลาหรือไม่ทันเวลาเลย"**

### ข้อควรระวัง

- **ลูปซ้อนลูปไม่ได้แปลว่า O(n²) เสมอไป** — ถ้าลูปในเป็นการวนตามจำนวนคงที่ (เช่น วนตรวจ 12 เดือนเสมอ
  ไม่ว่าข้อมูลข้างนอกจะมีกี่รายการ) ความซับซ้อนยังคงเป็น O(n) อยู่ สิ่งที่ทำให้เป็น O(n²) คือ **ขนาดของลูปใน
  ที่แปรผันตามขนาดข้อมูลเดียวกันกับลูปนอก**
- อย่าดูแค่ "โค้ดสั้นดูง่าย" แล้วสรุปว่าเร็ว — โค้ด 5 บรรทัดในตัวอย่างนี้คือต้นเหตุของปัญหา Performance ที่
  ร้ายแรงที่สุดในบทนี้ทั้งหมด (แย่กว่าการเลือกชนิดข้อมูลผิดใน Part 068 มาก) เพราะเป็นปัญหาระดับอัลกอริทึม
  ไม่ใช่ระดับ syntax

### แบบฝึกหัดที่ 871.1

**โจทย์**: หากข้อมูลในตัวอย่างข้างต้นโตขึ้นเป็น 30,000 รายการ (10 เท่าของ 3,000) จงประมาณว่าโปรแกรม
`step871_nestedloop` จะใช้เวลาประมาณกี่วินาที โดยใช้หลักการ O(n²)

**เฉลย**: เมื่อข้อมูลโตเป็น 10 เท่า เวลาจะโตเป็น 10² = 100 เท่า จากฐาน ~0.65 วินาทีที่ n=3,000 จะได้
ประมาณ 0.65 × 100 = **~65 วินาที** — นี่คือเหตุผลที่ต้องระวังรูปแบบลูปซ้อนลูปแบบนี้ตั้งแต่ตอนออกแบบ
ไม่ใช่มารอแก้ตอนที่ระบบ production ช้าจนใช้งานไม่ได้แล้ว

---

## ขั้นตอนที่ 872: ระเบียบวิธีสร้าง Benchmark ที่ยุติธรรม — เตรียมชุดข้อมูล 50,000 รายการ

### ทำไมต้องขยายขนาดจาก 20,000 (Part 068) เป็น 50,000 รายการ

Part 068 ขั้นตอนที่ 673-674 ใช้ตาราง 20,000 รายการเปรียบเทียบ `SEARCH` กับ `SEARCH ALL` Part นี้ขยับขึ้นเป็น
**50,000 รายการ** และเพิ่มคู่แข่งตัวที่สาม (ไฟล์ Indexed) เพื่อให้เห็นภาพที่สมจริงกับงาน Mainframe จริงมากขึ้น
(ตารางรหัสลูกค้า, ตารางบัญชี ฯลฯ ที่มักมีหลักหมื่นถึงหลักแสนรายการ) และเพื่อให้ตัวเลขของ Linear Search
ชัดเจนพอที่จะเห็นผลกระทบจริงแบบไม่ต้องเดา

### หลักการออกแบบ Benchmark ที่ยุติธรรม (Apples-to-Apples)

เพื่อให้การเปรียบเทียบทั้ง 3 เทคนิคในขั้นตอนที่ 873-875 มีความหมาย **ต้องควบคุมตัวแปรให้เหมือนกันทุก
ประการ** ยกเว้นเทคนิคที่ทดสอบ:

1. **ข้อมูลชุดเดียวกันทุกประการ**: 50,000 รายการ, รหัส `1000000` ถึง `1049999` เรียงต่อเนื่อง
2. **รูปแบบการค้นหาเดียวกันทุกประการ**: ค้นหา 50,000 ครั้ง โดย**เอนเอียงไปทางปลายตาราง** (คีย์วนอยู่ใน
   50 รายการสุดท้าย) เพื่อจำลอง **กรณีเลวร้ายที่สุด (worst case)** ของ Linear Search — หลักการเดียวกับ
   Part 068 ขั้นตอนที่ 673 ที่พิสูจน์แล้วว่าจำเป็นสำหรับการทดสอบที่สะท้อนความจริง (ทบทวนแบบฝึกหัดที่ 673.1)
3. **เครื่องเดียวกัน, รันในเซสชันเดียวกัน**: ลดผลกระทบจากความแปรปรวนของสภาพแวดล้อม

### สูตรการสร้างข้อมูล (ใช้ร่วมกันทั้ง 3 โปรแกรมในขั้นตอนถัดไป)

```cobol
      *> Same fill pattern used by ALL THREE benchmark programs in
      *> steps 873-875, so the comparison is fair.
           PERFORM VARYING WS-FILL-IDX FROM 1 BY 1
                   UNTIL WS-FILL-IDX > 50000
               COMPUTE PROD-CODE(WS-FILL-IDX) =
                       1000000 + WS-FILL-IDX - 1
               MOVE "ITEM" TO PROD-NAME(WS-FILL-IDX)
           END-PERFORM.
```

```cobol
      *> Same worst-case search-key pattern used by ALL THREE
      *> programs: cycles through the LAST 50 entries of the table,
      *> forcing a linear scan to walk almost the entire table.
           COMPUTE WS-SEARCH-KEY = 1000000 + 49999 -
               FUNCTION MOD(WS-SEARCH-PASS, 50)
```

### เรื่องจริงที่พบระหว่างเตรียม Benchmark นี้: บั๊กจาก Column 72 ที่ไม่ใช่ภาษาไทย

ระหว่างเตรียมโค้ดสำหรับ Part นี้ ผู้เขียนพบบั๊กจริงที่คุ้มค่ามากที่จะเล่าไว้ เพราะมันขยายกฎเหล็กเรื่อง column 72
ที่หลักสูตรนี้เน้นย้ำมาตลอด (ปกติเราพูดถึงมันในบริบทตัวอักษรภาษาไทยที่กิน 3 ไบต์ต่อตัวจนล้น column 72 —
ทบทวนจาก `docs/COURSE-OUTLINE.md`) แต่ครั้งนี้ **เป็นภาษาอังกฤษล้วน ๆ ไม่มีตัวอักษรไทยแม้แต่ตัวเดียว**:

```cobol
      *> THIS LINE IS 73 CHARACTERS LONG - ONE CHARACTER PAST THE
      *> COLUMN 72 LIMIT. THE COMPILER SILENTLY DROPS THE LAST
      *> CHARACTER: "5000" BECOMES "500" WITHOUT ANY WARNING AT ALL.
           PERFORM VARYING WS-KEY-IDX FROM 1 BY 1 UNTIL WS-KEY-IDX > 5000
               ADD 1 TO WS-COUNT
           END-PERFORM.
```

เมื่อคอมไพล์และรันโค้ดข้างต้นจริง (ไฟล์ `looptest.cob` ทดสอบแยกต่างหาก) ผลลัพธ์ที่ได้คือ:

```
COUNT=000000500
```

**ลูปทำงานแค่ 500 ครั้ง ไม่ใช่ 5,000 ครั้งตามที่ตั้งใจ!** สาเหตุคือบรรทัดนี้ยาว 73 ตัวอักษร เกิน column 72
ไปแค่ 1 ตัวอักษรพอดี ตัวอักษร `0` ตัวสุดท้ายของเลข `5000` จึงถูกตัดทิ้งอย่างเงียบ ๆ กลายเป็น `500` — โค้ด
**คอมไพล์ผ่านโดยไม่มี error หรือ warning ใด ๆ เลย** เพราะ `500` ก็เป็นตัวเลขที่ถูกต้องตามหลักไวยากรณ์ทุก
ประการ เพียงแต่ไม่ใช่ค่าที่ตั้งใจ

### วิธีแก้: ตัดบรรทัดให้สั้นลงก่อนถึง column 72 เสมอ

```cobol
      *> Same statement, split before column 72 - now safe.
           PERFORM VARYING WS-KEY-IDX FROM 1 BY 1
                   UNTIL WS-KEY-IDX > 5000
               ADD 1 TO WS-COUNT
           END-PERFORM.
```

**ผลลัพธ์จริงหลังแก้ไข:**

```
COUNT=000005000
```

### ข้อควรระวัง

- **กฎเรื่อง column 72 ไม่ได้เป็นปัญหาเฉพาะภาษาไทยเท่านั้น** — โค้ดภาษาอังกฤษล้วนที่มีชื่อตัวแปรยาว
  (`WS-SEARCH-KEY`, `WS-NOTFOUND-COUNT` ฯลฯ ตามธรรมเนียมการตั้งชื่อที่สื่อความหมายชัดเจนของหลักสูตรนี้)
  ก็เสี่ยงเกิน column 72 ได้ง่ายเช่นกัน โดยเฉพาะเมื่อบรรทัดมีทั้งชื่อตัวแปรยาวและค่าคงที่ตัวเลขหลายหลักรวมกัน
- **นี่คืออันตรายที่ร้ายแรงกว่า compile error เสียอีก** — เพราะโปรแกรมคอมไพล์ผ่านและทำงาน "ได้" แต่ให้
  ผลลัพธ์ผิดแบบเงียบ ๆ (silent wrong result) ตรงกับรูปแบบอันตรายเดียวกับที่ Part 069 ขั้นตอนที่ 686
  เตือนไว้เรื่อง input validation — บทเรียนคือ **ต้องตรวจสอบความยาวบรรทัดของโค้ด COBOL แบบ Fixed-Format
  เป็นนิสัยทุกครั้งก่อนเชื่อผลลัพธ์**, สามารถตรวจสอบด้วยคำสั่ง shell ง่าย ๆ เช่น
  `awk '{ if (length($0) > 72) print NR }' program.cob`
- แบบฝึกหัด Benchmark ทั้งหมดในขั้นตอนที่ 873-880 ของ Part นี้ถูกตรวจสอบความยาวบรรทัดด้วยวิธีนี้แล้วทุกไฟล์
  ก่อนนำผลลัพธ์มาเผยแพร่

### แบบฝึกหัดที่ 872.1

**โจทย์**: จงอธิบายว่าทำไมบั๊กแบบนี้ถึง "อันตรายกว่า" การเขียนโค้ดผิดจน compile error ทั้งที่ดูเหมือนเป็น
ปัญหาเล็กน้อยกว่า (แค่ตัวอักษรเดียวเกินมา)

**เฉลยแนวทาง**: เพราะ compile error หยุดกระบวนการทันทีและบังคับให้นักพัฒนาต้องแก้ไขก่อนไปต่อ แต่บั๊ก
แบบ column-72-truncation ทำให้โปรแกรม**คอมไพล์ผ่านและรันได้ตามปกติ** เพียงแต่ให้ผลลัพธ์ที่ผิดเงียบ ๆ
โดยไม่มีสัญญาณเตือนใด ๆ เลย ถ้าไม่มีการตรวจสอบผลลัพธ์อย่างละเอียด (เช่นในกรณีนี้คือสังเกตว่า COUNT ควร
เป็น 5000 แต่กลับได้ 500) บั๊กนี้อาจหลุดรอดไปถึงระบบ production ได้โดยไม่มีใครรู้เป็นเวลานาน ซึ่งตรงกับ
รูปแบบอันตรายที่ร้ายแรงที่สุดที่ Part 069 (Security) เตือนไว้ซ้ำแล้วซ้ำเล่า: **"ผลลัพธ์ที่ดูสมเหตุสมผลแต่
ผิดสนิท อันตรายกว่าโปรแกรม crash เสมอ"**

---

## ขั้นตอนที่ 873: SEARCH (Linear) บนตาราง 50,000 รายการ — วัดคอขวดที่แท้จริง

### โปรแกรมทดสอบฉบับเต็ม

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP873SEARCH.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-ENTRY OCCURS 50000 TIMES
                   INDEXED BY PT-IDX.
               10  PROD-CODE       PIC 9(7).
               10  PROD-NAME       PIC X(10).

       01  WS-FILL-IDX         PIC 9(7).
       01  WS-SEARCH-PASS      PIC 9(7).
       01  WS-SEARCH-KEY       PIC 9(7).
       01  WS-FOUND-COUNT      PIC 9(7) VALUE 0.
       01  WS-NOTFOUND-COUNT   PIC 9(7) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM VARYING WS-FILL-IDX FROM 1 BY 1
                   UNTIL WS-FILL-IDX > 50000
               COMPUTE PROD-CODE(WS-FILL-IDX) =
                       1000000 + WS-FILL-IDX - 1
               MOVE "ITEM" TO PROD-NAME(WS-FILL-IDX)
           END-PERFORM.

           PERFORM VARYING WS-SEARCH-PASS FROM 1 BY 1
                   UNTIL WS-SEARCH-PASS > 50000
               COMPUTE WS-SEARCH-KEY = 1000000 + 49999 -
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
cobc -x -o step873_search step873_search.cob
time ./step873_search
```

**ผลลัพธ์จริง (2 รอบการรัน):**

```
SEARCH (linear): found=0050000 not-found=0000000
real	0m11.628s
real	0m11.567s
```

### อธิบายจุดสำคัญ

- 50,000 รายการ × ค้นหาเกือบทั้งตารางทุกครั้ง (worst case) = ประมาณ **2,500,000,000 (2.5 พันล้าน)**
  ครั้งของการเปรียบเทียบตัวเลข — ใช้เวลารวม **~11.6 วินาที**
- เทียบกับ Part 068 ที่ใช้ตาราง 20,000 รายการแล้วใช้เวลา ~1.8 วินาที: ขนาดตารางโตขึ้น 2.5 เท่า (20,000 →
  50,000) แต่เวลาโตขึ้นประมาณ **6.4 เท่า** (1.8 → 11.6) ใกล้เคียงกับ 2.5² ≈ 6.25 ตามที่คาดจากความซับซ้อน
  O(n²) ของ "n รายการ ค้นหา n ครั้งแบบ worst-case" (คนละมิติกับ O(n) ต่อการค้นหาหนึ่งครั้งที่ Part 068 กล่าวถึง
  — ที่นี่เรานับรวมทั้งหมด n ครั้งของการค้นหาด้วย)
- ตัวเลขนี้คือ **baseline (เส้นฐาน)** ที่จะนำไปเทียบกับขั้นตอนที่ 874 (SEARCH ALL) และขั้นตอนที่ 875
  (ไฟล์ Indexed) ในหัวข้อถัดไป

### ข้อควรระวัง

- ต้นทุนของ Linear Search ที่ระดับ 50,000 รายการนี้ **ใหญ่พอที่จะกระทบ Batch Window จริงได้แล้ว** — ถ้า
  โปรแกรม Batch ต้องทำแบบนี้ซ้ำหลายรอบต่อคืน (เช่น ประมวลผลหลายไฟล์ที่ต้องค้นหาตารางเดียวกัน) เวลาสะสม
  อาจกลายเป็นปัญหาจริงจังได้อย่างรวดเร็ว
- อย่าลืม `SET PT-IDX TO 1` ก่อน `SEARCH` ทุกครั้ง (ทบทวนจาก Part 017 และ Part 068 ขั้นตอนที่ 673)

### แบบฝึกหัดที่ 873.1

**โจทย์**: จากตัวเลข Part 068 (20,000 รายการ ~1.8 วินาที) และ Part นี้ (50,000 รายการ ~11.6 วินาที)
จงประมาณเวลาที่ Linear Search แบบเดียวกันจะใช้ถ้าตารางมี 200,000 รายการ (4 เท่าของ 50,000)

**เฉลย**: ด้วยหลักการ O(n²) ของรูปแบบ "n รายการ ค้นหา n ครั้งแบบ worst-case" เมื่อ n โตเป็น 4 เท่า เวลา
จะโตเป็น 4² = 16 เท่า จากฐาน ~11.6 วินาทีที่ 50,000 รายการ จะได้ประมาณ 11.6 × 16 = **~186 วินาที
(ประมาณ 3 นาที)** — ตัวเลขนี้แสดงให้เห็นชัดเจนว่าทำไมองค์กรที่มีตารางขนาดใหญ่ระดับแสนถึงล้านรายการจึง
ไม่มีทางใช้ Linear Search ได้จริงในทางปฏิบัติ

---

## ขั้นตอนที่ 874: SEARCH ALL (Binary) บนตาราง 50,000 รายการเดียวกัน

### โปรแกรมทดสอบฉบับเต็ม (ข้อมูลและรูปแบบการค้นหาเหมือนขั้นตอนที่ 873 ทุกประการ)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP874SEARCHALL.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  PRODUCT-TABLE.
           05  PRODUCT-ENTRY OCCURS 50000 TIMES
                   ASCENDING KEY IS PROD-CODE
                   INDEXED BY PT-IDX.
               10  PROD-CODE       PIC 9(7).
               10  PROD-NAME       PIC X(10).

       01  WS-FILL-IDX         PIC 9(7).
       01  WS-SEARCH-PASS      PIC 9(7).
       01  WS-SEARCH-KEY       PIC 9(7).
       01  WS-FOUND-COUNT      PIC 9(7) VALUE 0.
       01  WS-NOTFOUND-COUNT   PIC 9(7) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM VARYING WS-FILL-IDX FROM 1 BY 1
                   UNTIL WS-FILL-IDX > 50000
               COMPUTE PROD-CODE(WS-FILL-IDX) =
                       1000000 + WS-FILL-IDX - 1
               MOVE "ITEM" TO PROD-NAME(WS-FILL-IDX)
           END-PERFORM.

           PERFORM VARYING WS-SEARCH-PASS FROM 1 BY 1
                   UNTIL WS-SEARCH-PASS > 50000
               COMPUTE WS-SEARCH-KEY = 1000000 + 49999 -
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
cobc -x -o step874_searchall step874_searchall.cob
time ./step874_searchall
```

**ผลลัพธ์จริง (2 รอบการรัน):**

```
SEARCH ALL (binary): found=0050000 not-found=0000000
real	0m0.041s
real	0m0.043s
```

### วิเคราะห์ผล — เร็วกว่า Linear Search ประมาณ 277 เท่า

| เทคนิค | ขนาดตาราง | เวลาที่ใช้ (50,000 ครั้งค้นหา) | เทียบกับ Linear |
|---|---|---|---|
| `SEARCH` (Linear) | 50,000 | ~11.60 วินาที | 1× (baseline) |
| `SEARCH ALL` (Binary) | 50,000 | ~0.042 วินาที | **~277× เร็วกว่า** |

เทียบกับผลของ Part 068 ที่ตาราง 20,000 รายการได้อัตราเร่งประมาณ 107 เท่า — เมื่อขนาดตารางโตขึ้น
(20,000 → 50,000) **อัตราเร่งของ Binary Search เมื่อเทียบกับ Linear Search กลับยิ่งมากขึ้นไปอีก** (107× →
277×) นี่คือธรรมชาติของ O(n) เทียบกับ O(log n): ยิ่งข้อมูลใหญ่ขึ้น ช่องว่างยิ่งถ่างออกแบบไม่มีขีดจำกัด

### ข้อควรระวัง

- ทบทวนจาก Part 068 ขั้นตอนที่ 674: ต้นทุนแฝงของ `SEARCH ALL` คือการต้อง **เรียงลำดับตารางให้เสร็จก่อน**
  เสมอ (`ASCENDING KEY`) และถ้าตารางไม่ได้เรียงจริงตามที่ประกาศไว้ ผลลัพธ์จะผิดพลาดแบบไม่มีการเตือนใด ๆ เลย
- ตาราง 50,000 รายการแบบนี้ต้องอยู่ใน **หน่วยความจำ (RAM) ทั้งหมด** ตลอดเวลาที่โปรแกรมทำงาน — ขั้นตอนที่
  875 จะแนะนำทางเลือกที่ไม่ต้องพึ่งหน่วยความจำขนาดใหญ่ขนาดนี้

### แบบฝึกหัดที่ 874.1

**โจทย์**: จงอธิบายว่าทำไมอัตราเร่งของ `SEARCH ALL` เทียบกับ `SEARCH` (107× ที่ 20,000 รายการ, 277× ที่
50,000 รายการ) ถึงเพิ่มขึ้นเรื่อย ๆ ตามขนาดตาราง แทนที่จะคงที่

**เฉลยแนวทาง**: Linear Search มีความซับซ้อนต่อการค้นหาหนึ่งครั้งแบบ O(n) ในขณะที่ Binary Search มี
ความซับซ้อนแบบ O(log n) — อัตราส่วนระหว่างทั้งสองคือ n / log₂(n) ซึ่ง**เพิ่มขึ้นเรื่อย ๆ**เมื่อ n โตขึ้น
(ที่ n=20,000: log₂(20000)≈14.3 → อัตราส่วน≈1400; ที่ n=50,000: log₂(50000)≈15.6 → อัตราส่วน≈3200)
แม้ตัวเลขทางทฤษฎีจะไม่ตรงกับตัวเลขที่วัดได้จริงเป๊ะ (เพราะมีปัจจัยอื่น เช่น overhead ของ runtime library
ปะปนอยู่) แต่ทิศทางของการเพิ่มขึ้นตรงกันชัดเจน — นี่คือเหตุผลที่ความแตกต่างระหว่างอัลกอริทึมสองแบบยิ่งสำคัญ
มากขึ้นเรื่อย ๆ เมื่อระบบขยายสเกล ไม่ใช่คงที่เหมือนการปรับปรุงระดับ syntax

---

## ขั้นตอนที่ 875: คู่แข่งตัวที่สาม — อ่านไฟล์ Indexed (ISAM) ด้วยคีย์โดยตรง

### ทำไมต้องมีคู่แข่งตัวที่สาม

Part 068 เปรียบเทียบแค่ 2 เทคนิคที่ทำงานในหน่วยความจำล้วน ๆ แต่ในโลกจริง ตารางขนาดใหญ่ระดับแสนถึงล้าน
รายการมักถูกเก็บเป็น **ไฟล์ Indexed (ISAM/VSAM KSDS บน Mainframe จริง)** แทนที่จะโหลดทั้งหมดเข้าหน่วย
ความจำ (ทบทวนจาก Part 028 และ Part 055) คำถามสำคัญคือ: **การอ่านไฟล์ Indexed ด้วยคีย์โดยตรง เร็วช้า
แค่ไหนเมื่อเทียบกับตารางในหน่วยความจำ?**

> **หมายเหตุสภาพแวดล้อม**: ตัวอย่างในขั้นตอนนี้คอมไพล์ด้วย GnuCOBOL รุ่นที่เปิดใช้งาน ISAM
> (`/opt/gnucobol-isam/bin/cobc` พร้อม `LD_LIBRARY_PATH=/opt/gnucobol-isam/lib`) เพื่อให้ `ORGANIZATION
> IS INDEXED` ทำงานได้ — ทดสอบและยืนยันผลจริงแล้วในสภาพแวดล้อมนี้

### ขั้นที่ 1: สร้างไฟล์ Indexed 50,000 รายการ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP875BUILD.
       AUTHOR. COBOL-COURSE.
      *> Builds an indexed (ISAM) file with 50,000 records so the
      *> next program can benchmark direct keyed READ against it.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PRODUCT-FILE ASSIGN TO "PRODIDX875.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS SEQUENTIAL
               RECORD KEY IS FD-PROD-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-FILE.
       01  PRODUCT-RECORD.
           05  FD-PROD-CODE       PIC 9(7).
           05  FD-PROD-NAME       PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS       PIC XX.
       01  WS-FILL-IDX          PIC 9(7).

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT PRODUCT-FILE.
           PERFORM VARYING WS-FILL-IDX FROM 1 BY 1
                   UNTIL WS-FILL-IDX > 50000
               COMPUTE FD-PROD-CODE = 1000000 + WS-FILL-IDX - 1
               MOVE "ITEM" TO FD-PROD-NAME
               WRITE PRODUCT-RECORD
               IF WS-FILE-STATUS NOT = "00"
                   DISPLAY "WRITE ERROR STATUS=" WS-FILE-STATUS
               END-IF
           END-PERFORM.
           CLOSE PRODUCT-FILE.
           DISPLAY "BUILD DONE. 50000 RECORDS WRITTEN.".
           STOP RUN.
```

### ขั้นที่ 2: เบนช์มาร์กการอ่านแบบ Random ด้วยคีย์ (รูปแบบคีย์เดียวกันกับขั้นตอนที่ 873-874 ทุกประการ)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP875READ.
       AUTHOR. COBOL-COURSE.
      *> Benchmarks 50,000 direct keyed READs against the indexed
      *> (ISAM) file built by STEP875BUILD, using the SAME biased
      *> key pattern as the SEARCH/SEARCH ALL benchmarks so all
      *> three techniques are compared fairly.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PRODUCT-FILE ASSIGN TO "PRODIDX875.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS FD-PROD-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-FILE.
       01  PRODUCT-RECORD.
           05  FD-PROD-CODE       PIC 9(7).
           05  FD-PROD-NAME       PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS       PIC XX.
       01  WS-SEARCH-PASS       PIC 9(7).
       01  WS-FOUND-COUNT       PIC 9(7) VALUE 0.
       01  WS-NOTFOUND-COUNT    PIC 9(7) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN I-O PRODUCT-FILE.
           IF WS-FILE-STATUS NOT = "00"
               DISPLAY "OPEN ERROR STATUS=" WS-FILE-STATUS
               STOP RUN
           END-IF.

           PERFORM VARYING WS-SEARCH-PASS FROM 1 BY 1
                   UNTIL WS-SEARCH-PASS > 50000
               COMPUTE FD-PROD-CODE = 1000000 + 49999 -
                   FUNCTION MOD(WS-SEARCH-PASS, 50)
               READ PRODUCT-FILE
                   INVALID KEY
                       ADD 1 TO WS-NOTFOUND-COUNT
                   NOT INVALID KEY
                       ADD 1 TO WS-FOUND-COUNT
               END-READ
           END-PERFORM.

           CLOSE PRODUCT-FILE.
           DISPLAY "INDEXED FILE (keyed READ): found=" WS-FOUND-COUNT
               " not-found=" WS-NOTFOUND-COUNT.
           STOP RUN.
```

```bash
export LD_LIBRARY_PATH=/opt/gnucobol-isam/lib
/opt/gnucobol-isam/bin/cobc -x -o step875_build step875_build_indexed.cob
/opt/gnucobol-isam/bin/cobc -x -o step875_read step875_read_indexed.cob
rm -f PRODIDX875.DAT*
time ./step875_build
time ./step875_read
time ./step875_read
```

**ผลลัพธ์จริง:**

```
BUILD DONE. 50000 RECORDS WRITTEN.
real	0m0.104s

INDEXED FILE (keyed READ): found=0050000 not-found=0000000
real	0m0.089s
real	0m0.113s
```

### ตารางสรุปทั้ง 3 เทคนิค — คำตอบของคำถามที่ตั้งไว้ในขั้นตอนที่ 872

| เทคนิค | ที่เก็บข้อมูล | เวลา (50,000 ครั้งค้นหา) | เทียบกับ Linear |
|---|---|---|---|
| `SEARCH` (Linear) | หน่วยความจำ | ~11.60 วินาที | 1× |
| `SEARCH ALL` (Binary) | หน่วยความจำ | ~0.042 วินาที | **~277×** |
| ไฟล์ Indexed (keyed READ) | ดิสก์ (ISAM) | ~0.10 วินาที | **~116×** |

### วิเคราะห์ผล — ทำไมไฟล์ Indexed ช้ากว่า SEARCH ALL ทั้งที่ทั้งคู่เป็น O(log n) โดยประมาณ

ผลลัพธ์ที่น่าสนใจที่สุดของขั้นตอนนี้คือ: **ไฟล์ Indexed (ที่ต้องอ่านจากดิสก์) ช้ากว่า SEARCH ALL (ที่ทำงาน
ในหน่วยความจำล้วน ๆ) ประมาณ 2.4 เท่า** แม้ทั้งคู่จะมีความซับซ้อนเชิงทฤษฎีใกล้เคียงกัน (ไฟล์ Indexed ใช้
โครงสร้าง B-Tree ภายในที่ก็เป็น O(log n) เหมือนกัน) เหตุผลคือ **การเข้าถึงดิสก์ (แม้จะผ่าน OS cache ที่
"อุ่น" อยู่แล้วในกรณีนี้) ยังคงมีต้นทุนของการเรียก system call และการจัดการ buffer ภายใน runtime library
ที่การเข้าถึงหน่วยความจำล้วน ๆ ไม่มี** — สอดคล้องกับหลักการจาก Part 068 ขั้นตอนที่ 676 ที่ว่า **"I/O แพง
กว่า CPU เสมอ"**

### กรอบการตัดสินใจ: เมื่อไรควรใช้เทคนิคไหน

| สถานการณ์ | เทคนิคที่แนะนำ | เหตุผล |
|---|---|---|
| ตารางเล็ก (ไม่กี่พันรายการ) ใช้ตลอดการทำงานของโปรแกรม | `SEARCH ALL` ในหน่วยความจำ | เร็วที่สุด และหน่วยความจำที่ใช้ไม่มาก |
| ตารางใหญ่ระดับแสนถึงล้านรายการ ที่หน่วยความจำพอโหลดได้ทั้งหมด | `SEARCH ALL` ในหน่วยความจำ (ถ้าเวลาที่ใช้โหลด/เรียงยอมรับได้) | ยังคงเร็วที่สุดถ้าโหลดลง RAM ได้ |
| ตารางใหญ่เกินกว่าจะโหลดทั้งหมดลง RAM ได้ในคราวเดียว | ไฟล์ Indexed | ไม่ต้องพึ่งหน่วยความจำขนาดใหญ่ ยังคงเร็วกว่า Linear Search มาก |
| ข้อมูลต้องคงอยู่ถาวรข้ามการรันโปรแกรมหลายครั้ง (persist) | ไฟล์ Indexed | ตารางในหน่วยความจำหายไปทุกครั้งที่โปรแกรมจบ (`STOP RUN`) |

### ข้อควรระวัง

- **อย่าตีความว่า "ไฟล์ Indexed ช้ากว่า SEARCH ALL เสมอ" แล้วสรุปว่าควรโหลดทุกอย่างเข้าหน่วยความจำเสมอ**
  — ตารางในหน่วยความจำมีข้อจำกัดเรื่องขนาด (ขั้นตอนที่ 877 จะพูดถึงผลกระทบของขนาดตารางต่อ Cache) และ
  ข้อมูลจะหายไปเมื่อโปรแกรมจบการทำงาน ไฟล์ Indexed ยังคงจำเป็นสำหรับข้อมูลที่ต้อง**คงอยู่ถาวร**
- ผลการทดสอบนี้วัดบนไฟล์ที่เพิ่งสร้างใหม่และ OS cache ยัง "อุ่น" อยู่ (เพิ่งเขียนไฟล์นี้ไปหมาด ๆ) ในสภาพแวดล้อม
  จริงที่ไฟล์ถูกอ่านครั้งแรกหลังจากไม่ได้แตะมานาน (cold cache) เวลาที่ใช้จริงอาจสูงกว่านี้มากเนื่องจากต้อง
  อ่านจากดิสก์จริง ไม่ใช่จาก OS page cache ในหน่วยความจำ

### แบบฝึกหัดที่ 875.1

**โจทย์**: บริษัทหนึ่งมีตารางอัตราแลกเปลี่ยนเงินตรา 200 สกุล ที่ถูกอ่านซ้ำหลายล้านครั้งในโปรแกรม Batch
คืนหนึ่ง ๆ จากตัวเลือกทั้ง 3 เทคนิคในขั้นตอนนี้ ควรเลือกใช้เทคนิคใด เพราะเหตุใด

**เฉลยแนวทาง**: ควรใช้ `SEARCH ALL` ในหน่วยความจำ เพราะ (1) ขนาดข้อมูลเล็กมาก (200 รายการ) ใช้
หน่วยความจำน้อยมากจนไม่มีนัยสำคัญ (2) ข้อมูลถูกอ่านซ้ำนับล้านครั้ง ทำให้ความแตกต่างด้านความเร็วต่อครั้ง
(แม้เพียงเสี้ยววินาที) ถูกทวีคูณเป็นผลกระทบมหาศาลเมื่อรวมทั้งคืน (3) อัตราแลกเปลี่ยนมักไม่เปลี่ยนแปลงระหว่าง
รอบการประมวลผลหนึ่ง ๆ จึงโหลดเข้าหน่วยความจำครั้งเดียวตอนเริ่มโปรแกรมได้อย่างปลอดภัย — ไฟล์ Indexed
เหมาะกับกรณีที่ข้อมูลใหญ่เกินหน่วยความจำหรือต้องมีการเขียนปรับปรุงข้อมูลถาวรระหว่างการประมวลผลมากกว่า

---

## ขั้นตอนที่ 876: กลยุทธ์ I/O ขั้นสูง — Cache ข้อมูลในหน่วยความจำเพื่อลดการอ่านไฟล์ซ้ำ

### ทบทวนจาก Part 068 ขั้นตอนที่ 676

Part 068 สอนเรื่อง **Blocking**: รวมหลาย record เข้าเป็น 1 การเขียนไฟล์ เพื่อลดจำนวนครั้งของการเรียก
`WRITE` — หลักการนั้นใช้ได้ดีกับการ**เขียน**ข้อมูลตามลำดับ Part นี้ขยายหลักการเดียวกัน ("ลดจำนวนครั้งที่
เรียก I/O") ไปสู่รูปแบบที่พบบ่อยกว่าในงาน Batch จริง: **การอ่านข้อมูลอ้างอิงชุดเดียวกันซ้ำหลายครั้งโดยไม่จำเป็น**

### ปัญหาที่พบบ่อยในโค้ด Batch จริง: อ่านไฟล์เดิมซ้ำในลูป

ลองจินตนาการโปรแกรม Batch ที่ประมวลผล "ธุรกรรม" หลายรอบ (เช่น รอบตรวจสอบ, รอบคำนวณ, รอบสร้างรายงาน)
และในแต่ละรอบต้องใช้ข้อมูลอ้างอิงชุดเดียวกัน (เช่น ยอดคงเหลือบัญชี 5,000 บัญชี) — ถ้าเขียนโค้ดแบบตรงไปตรงมา
โปรแกรมจะ**เปิดไฟล์และอ่านคีย์เดิมซ้ำทุกรอบ** ทั้งที่ข้อมูลไม่ได้เปลี่ยนแปลงระหว่างรอบเลย

### ทดสอบ A: อ่านไฟล์ซ้ำทุกครั้งที่ต้องใช้ข้อมูล (5 รอบ × 5,000 คีย์ = 25,000 ครั้งการอ่านไฟล์)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP878REREADS.
       AUTHOR. COBOL-COURSE.
      *> Simulates a common batch mistake: re-reading the SAME 5,000
      *> reference records from an indexed file over and over again
      *> (5 passes = 25,000 keyed READs total) instead of loading
      *> them into memory once.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PRODUCT-FILE ASSIGN TO "PRODIDX875.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS FD-PROD-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-FILE.
       01  PRODUCT-RECORD.
           05  FD-PROD-CODE       PIC 9(7).
           05  FD-PROD-NAME       PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS       PIC XX.
       01  WS-PASS              PIC 9(3).
       01  WS-KEY-IDX           PIC 9(7).
       01  WS-READ-COUNT        PIC 9(9) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN I-O PRODUCT-FILE.
           PERFORM VARYING WS-PASS FROM 1 BY 1 UNTIL WS-PASS > 5
               PERFORM VARYING WS-KEY-IDX FROM 1 BY 1
                       UNTIL WS-KEY-IDX > 5000
                   COMPUTE FD-PROD-CODE = 1000000 + WS-KEY-IDX - 1
                   READ PRODUCT-FILE
                       NOT INVALID KEY
                           ADD 1 TO WS-READ-COUNT
                   END-READ
               END-PERFORM
           END-PERFORM.
           CLOSE PRODUCT-FILE.
           DISPLAY "RE-READ FROM FILE EACH TIME: total reads="
               WS-READ-COUNT.
           STOP RUN.
```

**ผลลัพธ์จริง (2 รอบการรัน, ใช้ไฟล์ `PRODIDX875.DAT` เดียวกับขั้นตอนที่ 875):**

```
RE-READ FROM FILE EACH TIME: total reads=000025000
real	0m0.040s
real	0m0.038s
```

### ทดสอบ B: โหลดข้อมูลเข้าหน่วยความจำครั้งเดียว แล้วนำมาใช้ซ้ำทุกรอบ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP878CACHED.
       AUTHOR. COBOL-COURSE.
      *> Same workload as STEP878REREADS (5,000 reference records
      *> needed across 5 passes = 25,000 logical lookups) but loads
      *> the 5,000 records into an in-memory table ONCE, then serves
      *> all 25,000 lookups from memory - zero extra file I/O after
      *> the initial load.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PRODUCT-FILE ASSIGN TO "PRODIDX875.DAT"
               ORGANIZATION IS INDEXED
               ACCESS MODE IS RANDOM
               RECORD KEY IS FD-PROD-CODE
               FILE STATUS IS WS-FILE-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-FILE.
       01  PRODUCT-RECORD.
           05  FD-PROD-CODE       PIC 9(7).
           05  FD-PROD-NAME       PIC X(10).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS       PIC XX.
       01  WS-CACHE-TABLE.
           05  WS-CACHE-ENTRY OCCURS 5000 TIMES.
               10  WS-CACHE-CODE   PIC 9(7).
               10  WS-CACHE-NAME   PIC X(10).
       01  WS-PASS              PIC 9(3).
       01  WS-KEY-IDX           PIC 9(7).
       01  WS-READ-COUNT        PIC 9(9) VALUE 0.
       01  WS-LOOKUP-COUNT      PIC 9(9) VALUE 0.
       01  WS-DUMMY-NAME        PIC X(10).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Load the 5,000 records into memory ONCE - this is the only
      *> file I/O this program performs.
           OPEN I-O PRODUCT-FILE.
           PERFORM VARYING WS-KEY-IDX FROM 1 BY 1
                   UNTIL WS-KEY-IDX > 5000
               COMPUTE FD-PROD-CODE = 1000000 + WS-KEY-IDX - 1
               READ PRODUCT-FILE
                   NOT INVALID KEY
                       ADD 1 TO WS-READ-COUNT
                       MOVE FD-PROD-CODE TO WS-CACHE-CODE(WS-KEY-IDX)
                       MOVE FD-PROD-NAME TO WS-CACHE-NAME(WS-KEY-IDX)
               END-READ
           END-PERFORM.
           CLOSE PRODUCT-FILE.

      *> Serve all 5 passes from the in-memory table - no more I/O.
           PERFORM VARYING WS-PASS FROM 1 BY 1 UNTIL WS-PASS > 5
               PERFORM VARYING WS-KEY-IDX FROM 1 BY 1
                       UNTIL WS-KEY-IDX > 5000
      *> Direct offset access into the in-memory table - this is
      *> the "lookup" that replaces each file READ above.
                   MOVE WS-CACHE-NAME(WS-KEY-IDX) TO WS-DUMMY-NAME
                   ADD 1 TO WS-LOOKUP-COUNT
               END-PERFORM
           END-PERFORM.

           DISPLAY "CACHED IN MEMORY: file reads=" WS-READ-COUNT
               " in-memory lookups=" WS-LOOKUP-COUNT.
           STOP RUN.
```

**ผลลัพธ์จริง (2 รอบการรัน):**

```
CACHED IN MEMORY: file reads=000005000 in-memory lookups=000025000
real	0m0.013s
real	0m0.013s
```

### วิเคราะห์ผล

| แนวทาง | จำนวนครั้งเรียกไฟล์ I/O | เวลาที่ใช้ | เทียบอัตราเร่ง |
|---|---|---|---|
| อ่านไฟล์ซ้ำทุกครั้ง | 25,000 ครั้ง | ~0.039 วินาที | 1× |
| Cache ในหน่วยความจำ | **5,000 ครั้ง** (ลดลง 80%) | ~0.013 วินาที | **~3× เร็วกว่า** |

จำนวนครั้งที่เรียก I/O ลดลง 80% (จาก 25,000 เหลือ 5,000) และเวลาที่ใช้ลดลงเหลือประมาณ 1 ใน 3 — ถึงแม้ตัว
เลขในการทดสอบนี้จะดูเล็ก (หลักมิลลิวินาที) เพราะไฟล์ยังอยู่ใน OS cache ที่ "อุ่น" (ทดสอบต่อจากขั้นตอนที่ 875
ทันที) แต่หลักการสำคัญคือ **"loop-invariant I/O hoisting"** — ถ้าข้อมูลไม่เปลี่ยนแปลงระหว่างรอบการวนซ้ำ
ให้อ่านครั้งเดียวแล้วเก็บไว้ใช้ซ้ำ แทนที่จะอ่านใหม่ทุกครั้งที่ต้องการ หลักการนี้ยิ่งสำคัญมากขึ้นตามสัดส่วนเมื่อ:

1. **จำนวนรอบการวนซ้ำเพิ่มขึ้น** (5 รอบในตัวอย่างนี้ อาจเป็นหลักร้อยหรือหลักพันรอบในงานจริง)
2. **ไฟล์อยู่บนดิสก์ที่ไม่ได้ cache ไว้แล้ว** (cold cache) ซึ่งต้นทุนของแต่ละ READ จะสูงกว่านี้มาก

### ข้อควรระวัง

- เทคนิคนี้ใช้ได้เฉพาะเมื่อ **ข้อมูลอ้างอิงไม่เปลี่ยนแปลงระหว่างรอบการประมวลผล** เท่านั้น — ถ้าข้อมูลอาจถูก
  โปรแกรมอื่นแก้ไขระหว่างที่เรากำลังประมวลผลอยู่ (concurrent update) การ cache ไว้ในหน่วยความจำอาจทำให้
  ได้ข้อมูลเก่าที่ไม่ตรงกับความเป็นจริงอีกต่อไป (stale data) — ต้องพิจารณาความเสี่ยงนี้ให้ดีก่อนนำไปใช้กับ
  ข้อมูลที่เปลี่ยนแปลงบ่อย
- ตัวอย่างนี้ใช้ประโยชน์จากการที่คีย์ต่อเนื่องกัน (`1000000` ถึง `1004999`) ทำให้แปลงคีย์เป็นตำแหน่งในตาราง
  ได้โดยตรง — ถ้าคีย์ไม่ต่อเนื่อง (เช่น เลขบัญชีจริงที่ไม่เรียงกัน) จะต้องใช้ `SEARCH ALL` (ขั้นตอนที่ 874)
  แทนการเข้าถึงโดยตรง ซึ่งยังคงเร็วกว่าการอ่านไฟล์ซ้ำมาก แต่ไม่เร็วเท่าการเข้าถึงโดยตรงแบบนี้

### แบบฝึกหัดที่ 876.1

**โจทย์**: จงอธิบายว่าทำไมเทคนิค "Cache ในหน่วยความจำ" ในขั้นตอนนี้ จึงเป็นการประยุกต์หลักการเดียวกันกับ
"ลดจำนวนครั้งที่เรียก I/O" ของ Part 068 ขั้นตอนที่ 676 แม้จะเป็นคนละสถานการณ์กัน (เขียน vs อ่าน)

**เฉลยแนวทาง**: หลักการร่วมของทั้งสองกรณีคือ **"ทุกครั้งที่เรียก I/O มีต้นทุนคงที่ (overhead) เกิดขึ้นเสมอ
ไม่ว่าจะอ่าน/เขียนข้อมูลปริมาณเท่าใด"** (Part 068 ขั้นตอนที่ 676) — Part 068 ใช้ Blocking ลดจำนวนครั้งของ
การ**เขียน**โดยรวมหลาย record เป็น 1 การเขียน ในขณะที่ Part นี้ลดจำนวนครั้งของการ**อ่าน**โดยอ่านครั้งเดียว
แล้วเก็บผลไว้ใช้ซ้ำ (แทนที่จะอ่านใหม่ทุกครั้งที่ต้องการข้อมูลเดิม) ทั้งสองกรณีมีเป้าหมายเดียวกันคือ **ลดจำนวน
ครั้งของการเรียก I/O ให้น้อยที่สุดเท่าที่จำเป็นจริง** เพียงแค่ประยุกต์ใช้กับการดำเนินการคนละแบบ (WRITE
เทียบกับ READ) เท่านั้น

---

## ขั้นตอนที่ 877: ขนาด Working Set และผลกระทบต่อ CPU Cache

### แนวคิด: หน่วยความจำไม่ได้เร็วเท่ากันหมดทุกจุด

นักพัฒนามักคิดว่า "ตัวแปรอยู่ใน RAM ทั้งหมด เข้าถึงเร็วเท่ากันหมด" แต่ในความเป็นจริง CPU สมัยใหม่มีลำดับชั้น
ของหน่วยความจำ (Memory Hierarchy): CPU Cache (L1/L2/L3) ที่เร็วมากแต่ขนาดเล็ก (มักไม่กี่ MB) และ
RAM หลักที่ใหญ่กว่ามากแต่ช้ากว่า **Working Set** คือปริมาณข้อมูลที่โปรแกรมเข้าถึงซ้ำ ๆ ในช่วงเวลาสั้น ๆ —
ถ้า Working Set มีขนาดเล็กพอที่จะ "พอดี" กับ Cache การเข้าถึงซ้ำจะเร็วขึ้นมาก เพราะ CPU ไม่ต้องวิ่งไปหา
ข้อมูลที่ RAM หลักทุกครั้ง

### ทดสอบ A: ตารางเล็ก (100,000 รายการ) เข้าถึงซ้ำ 4,000 รอบ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP877SMALL.
       AUTHOR. COBOL-COURSE.
      *> Working-set demo: touch 400,000,000 elements total, but all
      *> from a SMALL table (100,000 entries) that is small enough to
      *> stay resident in CPU cache across repeated passes.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  SMALL-TABLE.
           05  SMALL-ENTRY OCCURS 100000 TIMES PIC 9(9) COMP.
       01  WS-DUMMY           PIC 9(9) COMP VALUE 0.
       01  WS-I               PIC 9(9) VALUE 0.
       01  WS-PASS            PIC 9(9) VALUE 0.
       01  WS-TABLE-SIZE      PIC 9(9) VALUE 100000.
       01  WS-PASSES          PIC 9(9) VALUE 4000.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > WS-TABLE-SIZE
               MOVE WS-I TO SMALL-ENTRY(WS-I)
           END-PERFORM.

           PERFORM VARYING WS-PASS FROM 1 BY 1 UNTIL WS-PASS > WS-PASSES
               PERFORM VARYING WS-I FROM 1 BY 1
                       UNTIL WS-I > WS-TABLE-SIZE
                   MOVE SMALL-ENTRY(WS-I) TO WS-DUMMY
               END-PERFORM
           END-PERFORM.

           DISPLAY "SMALL TABLE: touched " WS-TABLE-SIZE
               " x " WS-PASSES " = 400,000,000 elements. LAST="
               WS-DUMMY.
           STOP RUN.
```

**ผลลัพธ์จริง (2 รอบการรัน):**

```
SMALL TABLE: touched 000100000 x 000004000 = 400,000,000 elements.
real	0m21.680s
real	0m21.298s
```

### ทดสอบ B: ตารางใหญ่ (10,000,000 รายการ, ~40MB) เข้าถึงซ้ำ 40 รอบ — จำนวนครั้งที่ "แตะ" ข้อมูลรวมเท่ากัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP877LARGE.
       AUTHOR. COBOL-COURSE.
      *> Working-set demo: touch the SAME total of 400,000,000
      *> elements, but from a LARGE table (10,000,000 entries, about
      *> 40 MB) that does not fit in typical CPU cache, so repeated
      *> passes must keep going back to main memory.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  LARGE-TABLE.
           05  LARGE-ENTRY OCCURS 10000000 TIMES PIC 9(9) COMP.
       01  WS-DUMMY           PIC 9(9) COMP VALUE 0.
       01  WS-I               PIC 9(9) VALUE 0.
       01  WS-PASS            PIC 9(9) VALUE 0.
       01  WS-TABLE-SIZE      PIC 9(9) VALUE 10000000.
       01  WS-PASSES          PIC 9(9) VALUE 40.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > WS-TABLE-SIZE
               MOVE WS-I TO LARGE-ENTRY(WS-I)
           END-PERFORM.

           PERFORM VARYING WS-PASS FROM 1 BY 1 UNTIL WS-PASS > WS-PASSES
               PERFORM VARYING WS-I FROM 1 BY 1
                       UNTIL WS-I > WS-TABLE-SIZE
                   MOVE LARGE-ENTRY(WS-I) TO WS-DUMMY
               END-PERFORM
           END-PERFORM.

           DISPLAY "LARGE TABLE: touched " WS-TABLE-SIZE
               " x " WS-PASSES " = 400,000,000 elements. LAST="
               WS-DUMMY.
           STOP RUN.
```

**ผลลัพธ์จริง (2 รอบการรัน):**

```
LARGE TABLE: touched 010000000 x 000000040 = 400,000,000 elements.
real	0m24.315s
real	0m24.180s
```

### วิเคราะห์ผล — ผลกระทบมีจริง แต่เล็กกว่าที่หลายคนคาดไว้มาก

| การทดสอบ | ขนาดตาราง | จำนวนรอบ | รวมจำนวนครั้งที่แตะข้อมูล | เวลาที่ใช้ |
|---|---|---|---|---|
| A (เล็ก, cache-friendly) | 100,000 (~400KB) | 4,000 | 400,000,000 | ~21.5 วินาที |
| B (ใหญ่, ไม่พอดี cache) | 10,000,000 (~40MB) | 40 | 400,000,000 | ~24.2 วินาที |

แม้จำนวนครั้งที่ "แตะ" ข้อมูลจะเท่ากันเป๊ะทั้งสองกรณี (400 ล้านครั้ง) **ตารางใหญ่ใช้เวลามากกว่าประมาณ 12%**
— ผลกระทบนี้**มีจริงและวัดได้จริง** แต่**เล็กกว่าที่คาดไว้มาก**เมื่อเทียบกับความแตกต่างระดับ 100+ เท่าที่เห็น
ในขั้นตอนที่ 873-875 เหตุผลคือ **libcob (runtime library ของ GnuCOBOL) มี overhead ต่อการเรียก `MOVE`
แต่ละครั้งที่มากกว่าต้นทุนของ cache miss ล้วน ๆ ในภาษา C ดิบ** ทำให้ผลกระทบของ cache ถูก "กลบ" บางส่วน
ด้วย overhead ของ runtime — นี่คือตัวอย่างที่ดีอีกครั้งของหลักการ **"วัดผลก่อนเชื่อ"** จาก Part 068:
ผลกระทบเรื่อง cache ที่เป็นความจริงในระดับ C/Assembly ดิบอาจไม่ปรากฏชัดเจนขนาดเดียวกันเมื่อผ่านชั้น
runtime library ของ COBOL

### ข้อควรระวัง

- **อย่าคาดหวังว่าผลกระทบเรื่อง cache จะมากเท่ากับความแตกต่างระดับอัลกอริทึม** (เช่น Linear เทียบกับ Binary
  Search) — Working Set เป็นปัจจัยรอง ควรพิจารณา**หลังจาก**แก้ไขปัญหาระดับอัลกอริทึมแล้วเท่านั้น
  (ทบทวนหลักการ "premature optimization" จาก Part 068 ขั้นตอนที่ 671)
- ตาราง 10,000,000 รายการ (`PIC 9(9) COMP`) ใช้หน่วยความจำประมาณ 40MB — ต้องแน่ใจว่าเครื่องที่ใช้พัฒนา
  หรือ Mainframe ที่ใช้ production มีหน่วยความจำเพียงพอก่อนประกาศตารางขนาดใหญ่แบบนี้ (ทบทวน Region
  Size จาก Part 052-053)

### แบบฝึกหัดที่ 877.1

**โจทย์**: จากผลการทดลองในขั้นตอนนี้ จงอธิบายว่าทำไมนักพัฒนา COBOL จึงไม่ควรใช้ "การปรับขนาดตารางให้
พอดีกับ CPU Cache" เป็นเทคนิคแรกที่นึกถึงเมื่อโปรแกรมทำงานช้า

**เฉลยแนวทาง**: เพราะผลการทดลองแสดงให้เห็นชัดเจนว่าผลกระทบจากขนาด Working Set ในสภาพแวดล้อม
GnuCOBOL นี้อยู่ในระดับ **~12%** เท่านั้น ในขณะที่การเลือกอัลกอริทึมผิด (เช่น Linear Search แทน Binary
Search จากขั้นตอนที่ 873-874) ทำให้ช้าลงถึง **หลักร้อยเท่า** การไปโฟกัสที่การปรับขนาดตารางให้พอดี Cache
ก่อนที่จะตรวจสอบว่ามีปัญหาระดับอัลกอริทึมหรือไม่ เป็นตัวอย่างของ "premature optimization" ที่เสียเวลาไปกับ
การปรับปรุงที่ให้ผลตอบแทนน้อย ทั้งที่ควรวัดผลหาคอขวดที่แท้จริงก่อนเสมอตามหลักการของ Part 068 ขั้นตอนที่ 671

---

## ขั้นตอนที่ 878: การ Optimize นิพจน์ COMPUTE — เมื่อการ "ช่วย" คอมไพเลอร์กลับทำให้ช้าลง

### มายาคติที่ต้องพิสูจน์: "เก็บค่าที่คำนวณซ้ำไว้ในตัวแปรชั่วคราวย่อมเร็วกว่าเสมอ"

หลักการเขียนโปรแกรมทั่วไปสอนว่า **"อย่าคำนวณค่าเดิมซ้ำหลายครั้ง ให้คำนวณครั้งเดียวแล้วเก็บไว้ใช้ซ้ำ"**
(หลักการที่เรียกว่า Common Subexpression Elimination) หลักการนี้ถูกต้องในหลายภาษา แต่ Part 068 สอนเรา
มาแล้วว่า**ไม่ควรเชื่ออะไรโดยไม่วัดผล** — มาทดสอบกันว่าหลักการนี้จริงแค่ไหนกับ GnuCOBOL

### ทดสอบ A: คำนวณนิพจน์เดิมซ้ำ 3 ครั้งในบรรทัดเดียว (5,000,000 รอบ)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP879REPEATED.
       AUTHOR. COBOL-COURSE.
      *> Recomputes the same sub-expression (WS-A * WS-B) three times
      *> inside one COMPUTE, 5,000,000 times in a loop.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-A            PIC 9(4)  VALUE 23.
       01  WS-B            PIC 9(4)  VALUE 17.
       01  WS-RESULT       PIC 9(9)  VALUE 0.
       01  WS-COUNTER      PIC 9(9)  VALUE 0.
       01  WS-LIMIT        PIC 9(9)  VALUE 5000000.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM UNTIL WS-COUNTER >= WS-LIMIT
               COMPUTE WS-RESULT = (WS-A * WS-B)
                   + (WS-A * WS-B) * 2
                   + (WS-A * WS-B) * 3
               ADD 1 TO WS-COUNTER
           END-PERFORM.
           DISPLAY "REPEATED SUBEXPR done. RESULT=" WS-RESULT.
           STOP RUN.
```

**ผลลัพธ์จริง (3 รอบการรัน):**

```
REPEATED SUBEXPR done. RESULT=000002346
real	0m0.594s
real	0m0.637s
real	0m0.633s
```

### ทดสอบ B: คำนวณครั้งเดียวเก็บไว้ในตัวแปรชั่วคราว แล้วนำมาใช้ซ้ำ (5,000,000 รอบเท่ากัน)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP879CACHED.
       AUTHOR. COBOL-COURSE.
      *> Computes the shared sub-expression (WS-A * WS-B) ONCE into
      *> WS-TEMP, then reuses WS-TEMP - same 5,000,000 iterations,
      *> same arithmetic result as STEP879REPEATED.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-A            PIC 9(4)  VALUE 23.
       01  WS-B            PIC 9(4)  VALUE 17.
       01  WS-TEMP         PIC 9(9)  VALUE 0.
       01  WS-RESULT       PIC 9(9)  VALUE 0.
       01  WS-COUNTER      PIC 9(9)  VALUE 0.
       01  WS-LIMIT        PIC 9(9)  VALUE 5000000.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM UNTIL WS-COUNTER >= WS-LIMIT
               COMPUTE WS-TEMP = WS-A * WS-B
               COMPUTE WS-RESULT = WS-TEMP
                   + WS-TEMP * 2
                   + WS-TEMP * 3
               ADD 1 TO WS-COUNTER
           END-PERFORM.
           DISPLAY "CACHED SUBEXPR done. RESULT=" WS-RESULT.
           STOP RUN.
```

**ผลลัพธ์จริง (3 รอบการรัน):**

```
CACHED SUBEXPR done. RESULT=000002346
real	0m0.701s
real	0m0.702s
real	0m0.739s
```

### วิเคราะห์ผล — มายาคติถูกหักล้างอีกครั้ง: เวอร์ชัน "ไม่ optimize" กลับเร็วกว่า

| เวอร์ชัน | เวลาเฉลี่ย | ผลลัพธ์ |
|---|---|---|
| คำนวณซ้ำ 3 ครั้ง (`STEP879REPEATED`) | **~0.62 วินาที** | เร็วกว่า |
| เก็บค่าไว้ในตัวแปรชั่วคราว (`STEP879CACHED`) | ~0.71 วินาที | ช้ากว่าประมาณ 15% |

ผลลัพธ์นี้ **ขัดกับสัญชาตญาณโดยสิ้นเชิง** เหตุผลที่เป็นไปได้มากที่สุดคือ GnuCOBOL แปลงทั้งสองเวอร์ชันเป็น
โค้ดภาษา C แล้วส่งให้ `gcc` คอมไพล์ต่อ (ทบทวนจาก Part 001) — `gcc` มักทำ **Common Subexpression
Elimination ให้อัตโนมัติอยู่แล้ว**ในเวอร์ชัน A (เห็นว่า `WS-A * WS-B` ปรากฏซ้ำ 3 ครั้งในนิพจน์เดียวกัน จึง
คำนวณครั้งเดียวแล้วนำผลไปใช้ซ้ำในระดับ C code) ในขณะที่เวอร์ชัน B **บังคับให้มีการ MOVE ค่าเข้า-ออกจาก
ตัวแปร `WS-TEMP` เพิ่มอีกหนึ่งครั้งอย่างชัดเจน** (การกำหนดค่าตัวแปร COBOL หนึ่งตัวมีต้นทุนของ runtime
library ที่ไม่ใช่ศูนย์เสมอ) ทำให้เวอร์ชันที่ "ตั้งใจ optimize" กลับมีงานให้ทำมากกว่าเดิม

### ทดสอบเพิ่มเติม: ต้นทุนของ ON SIZE ERROR ที่ไม่เคยเกิดขึ้นจริง (10,000,000 รอบ)

`ON SIZE ERROR` เป็นเครื่องมือป้องกันการ overflow ที่สำคัญ (ทบทวนจาก Part 009) แต่มันก็มีต้นทุน — มาวัดกัน
ว่าต้นทุนนี้มากแค่ไหนเมื่อเงื่อนไข error ไม่เคยเกิดขึ้นจริงเลยสักครั้ง:

```cobol
      *> WITHOUT ON SIZE ERROR
           COMPUTE WS-RESULT = WS-A * WS-B + WS-COUNTER.
```

```cobol
      *> WITH ON SIZE ERROR - never actually triggers in this test,
      *> isolating the pure overhead of the check itself.
           COMPUTE WS-RESULT = WS-A * WS-B + WS-COUNTER
               ON SIZE ERROR
                   ADD 1 TO WS-ERROR-COUNT
           END-COMPUTE.
```

```bash
cobc -x -o step879_nosizeerr step879_nosizeerr.cob
cobc -x -o step879_sizeerr step879_sizeerr.cob
time ./step879_nosizeerr
time ./step879_sizeerr
```

**ผลลัพธ์จริง (2 รอบการรันแต่ละแบบ):**

```
NO ON SIZE ERROR   : 0.994s, 1.022s   (เฉลี่ย ~1.01s)
WITH ON SIZE ERROR : 1.658s, 1.624s   (เฉลี่ย ~1.64s)
```

การเพิ่ม `ON SIZE ERROR` ที่ไม่เคยเกิด error จริงเลยแม้แต่ครั้งเดียว ทำให้โปรแกรมช้าลงประมาณ **62%** —
ต้นทุนนี้**มีจริงและมีนัยสำคัญ** มากกว่าความแตกต่างเรื่อง subexpression ในการทดสอบก่อนหน้ามาก

### สรุปบทเรียนของขั้นตอนนี้

1. **การ "optimize" นิพจน์ COMPUTE ด้วยมือ อาจไม่ได้ผลหรือกลับทำให้ช้าลง** เพราะคอมไพเลอร์ระดับ C
   (ที่ GnuCOBOL อาศัยอยู่) มักทำ optimization พื้นฐานเช่น CSE ให้อัตโนมัติอยู่แล้ว
2. **`ON SIZE ERROR` มีต้นทุนจริงที่วัดได้ชัดเจน** — ควรใช้เมื่อมีความเสี่ยง overflow จริงเท่านั้น (เช่น
   การคำนวณดอกเบี้ยทบต้นที่ผลลัพธ์อาจโตเกินคาด) ไม่ใช่ใส่ไว้ "เผื่อ" ในทุกจุดโดยไม่พิจารณา
3. **บทเรียนที่แท้จริงไม่ใช่ "อย่าใช้ ON SIZE ERROR" หรือ "อย่าเก็บค่าไว้ในตัวแปรชั่วคราว"** แต่คือ
   **"วัดผลก่อนตัดสินใจเสมอ โดยเฉพาะเมื่อกำลังจะแลกความชัดเจนของโค้ดกับความเร็วที่ยังไม่ได้พิสูจน์"**

### ข้อควรระวัง

- **อย่าลบ `ON SIZE ERROR` ออกจากโค้ดที่มีความเสี่ยง overflow จริงเพียงเพราะเห็นตัวเลขในขั้นตอนนี้** —
  ต้นทุนของการคำนวณผิดพลาดแบบเงียบ ๆ (silent wrong result ตามที่ขั้นตอนที่ 872 พิสูจน์ไว้) มักสูงกว่า
  ต้นทุนด้านเวลาที่เสียไปมาก โดยเฉพาะในระบบการเงินที่ทบทวนมาตลอดหลักสูตรนี้
- ผลลัพธ์เรื่อง subexpression ที่ "เวอร์ชันไม่ optimize เร็วกว่า" **ไม่ใช่กฎตายตัว** — ขึ้นอยู่กับรุ่นของ
  `gcc` ที่ GnuCOBOL เรียกใช้ และความซับซ้อนของนิพจน์ ในนิพจน์ที่ซับซ้อนกว่านี้มาก (เช่น เรียก subprogram
  หรือ Intrinsic Function ที่มีค่าใช้จ่ายสูงซ้ำหลายครั้ง) การเก็บผลลัพธ์ไว้ใช้ซ้ำอาจยังคงคุ้มค่าอยู่ — **ต้อง
  วัดผลกับนิพจน์จริงของตัวเองเสมอ ไม่ใช่เชื่อผลจากตัวอย่างนี้ตรง ๆ**

### แบบฝึกหัดที่ 878.1

**โจทย์**: จงอธิบายว่าทำไมผลการทดลองเรื่อง subexpression ในขั้นตอนนี้ จึงยิ่งตอกย้ำหลักการ "วัดผลก่อนเชื่อ"
จาก Part 068 ให้หนักแน่นยิ่งขึ้นไปอีก เมื่อเทียบกับตัวอย่างมายาคติอื่น ๆ ที่เคยพิสูจน์มาแล้ว (PERFORM vs
GO TO, COMP-3 vs DISPLAY)

**เฉลยแนวทาง**: ตัวอย่างก่อนหน้า (PERFORM vs GO TO, COMP-3 vs DISPLAY) พิสูจน์ว่า **"สิ่งที่เชื่อกันมานาน
อาจไม่จริงอีกต่อไป"** แต่ยังเป็นการเปรียบเทียบระหว่างสองแนวทางที่ต่างกันชัดเจน ในขณะที่ตัวอย่างเรื่อง
subexpression ในขั้นตอนนี้พิสูจน์ให้เห็นสิ่งที่รุนแรงกว่านั้นคือ **"ความพยายาม optimize ด้วยมือโดยอาศัย
สามัญสำนึกล้วน ๆ (ไม่คำนวณค่าซ้ำ ย่อมดีกว่าเสมอ) อาจกลับทำให้ผลลัพธ์แย่ลงกว่าการไม่ทำอะไรเลย" — นี่คือ
ระดับที่อันตรายกว่าการเชื่อมายาคติเสียอีก เพราะเป็นการลงมือ "แก้ไข" โค้ดที่ทำงานถูกต้องอยู่แล้วด้วยความ
มั่นใจว่าจะดีขึ้น แต่กลับแย่ลงโดยไม่รู้ตัว หากไม่วัดผลยืนยันก่อนและหลังการเปลี่ยนแปลงทุกครั้ง

---

## ขั้นตอนที่ 879: สร้าง "Profiler" อย่างง่ายด้วยโค้ด COBOL เอง

### ทำไมบางครั้งเราใช้คำสั่ง `time` ของ Linux ไม่ได้

ตลอด Part 068 และ Part นี้ เราใช้คำสั่ง `time` ของ Linux วัดเวลาทั้งโปรแกรมจากภายนอก — วิธีนี้ใช้ได้ดี
เมื่อพัฒนาบนเครื่อง PC/Linux ทั่วไป แต่ในสภาพแวดล้อม Mainframe จริงหรือสภาพแวดล้อมที่ควบคุมเข้มงวด
(เช่น รันผ่าน JCL batch job ที่ไม่มีสิทธิ์เข้าถึง shell โดยตรง) เราอาจต้องการวัดเวลา **เฉพาะบางส่วนของ
โปรแกรม** (เช่น "ขั้นตอนอ่านไฟล์ใช้เวลาเท่าไหร่ เทียบกับขั้นตอนคำนวณ") โดยไม่พึ่งเครื่องมือภายนอกเลย —
นี่คือที่มาของแนวคิด **"Poor Man's Profiler"**: ใช้ `FUNCTION CURRENT-DATE` ของ COBOL เองสร้างนาฬิกาจับ
เวลาภายในโปรแกรม

### เทคนิคหลัก: แปลง FUNCTION CURRENT-DATE เป็นตัวเลขเปรียบเทียบได้

`FUNCTION CURRENT-DATE` คืนค่าข้อความ 21 ตัวอักษรตามรูปแบบ `YYYYMMDDHHMMSSss+HHMM` — ส่วนที่เรา
สนใจสำหรับการจับเวลาคือ `HH`, `MM`, `SS`, และ `ss` (เศษ 1/100 วินาที) เราแปลงส่วนนี้เป็น **"จำนวนเศษ
1/100 วินาทีรวมนับจากเที่ยงคืน"** เพื่อให้ลบกันได้ตรง ๆ

```cobol
       01  WS-NOW                   PIC X(21).
       01  WS-NOW-FIELDS REDEFINES WS-NOW.
           05  WS-NOW-DATE          PIC X(8).
           05  WS-NOW-HH            PIC 9(2).
           05  WS-NOW-MM            PIC 9(2).
           05  WS-NOW-SS            PIC 9(2).
           05  WS-NOW-CS            PIC 9(2).
           05  FILLER               PIC X(5).
       01  WS-NOW-CS-VALUE          PIC 9(9).
```

```cobol
       GET-NOW-CS.
      *> Pack HH/MM/SS/hundredths into one "total hundredths since
      *> midnight" number for easy subtraction.
           MOVE FUNCTION CURRENT-DATE TO WS-NOW
           COMPUTE WS-NOW-CS-VALUE =
               ((WS-NOW-HH * 60 + WS-NOW-MM) * 60 + WS-NOW-SS)
               * 100 + WS-NOW-CS.
```

### โปรแกรมฉบับเต็ม: วัดเวลา 3 ช่วงงานในโปรแกรมเดียว

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP880PROFILER.
       AUTHOR. COBOL-COURSE.
      *> A "poor man's profiler": times three stages of work using
      *> only FUNCTION CURRENT-DATE (no external tools, no `time`
      *> command) and DISPLAYs the elapsed hundredths of a second
      *> for each stage.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NOW                   PIC X(21).
       01  WS-NOW-FIELDS REDEFINES WS-NOW.
           05  WS-NOW-DATE          PIC X(8).
           05  WS-NOW-HH            PIC 9(2).
           05  WS-NOW-MM            PIC 9(2).
           05  WS-NOW-SS            PIC 9(2).
           05  WS-NOW-CS            PIC 9(2).
           05  FILLER               PIC X(5).

       01  WS-NOW-CS-VALUE          PIC 9(9).
       01  WS-START-CS              PIC 9(9).
       01  WS-END-CS                PIC 9(9).
       01  WS-ELAPSED-CS            PIC 9(9).
       01  WS-STAGE-NAME            PIC X(30).

       01  WS-COUNTER               PIC 9(9) VALUE 0.
       01  WS-LIMIT                 PIC 9(9).
       01  WS-ACCUM                 PIC 9(9) VALUE 0.

       01  WS-TABLE.
           05  WS-ENTRY OCCURS 30000 TIMES
                   ASCENDING KEY IS WS-CODE
                   INDEXED BY WS-IDX.
               10  WS-CODE          PIC 9(7).
       01  WS-FILL-IDX              PIC 9(7).
       01  WS-SEARCH-PASS           PIC 9(7).
       01  WS-SEARCH-KEY            PIC 9(7).
       01  WS-FOUND-COUNT           PIC 9(7) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== PROFILING RUN START ===".

      *> ---- Stage 1: a tight arithmetic loop ----
           MOVE "STAGE 1 - ADD LOOP (8M TIMES)" TO WS-STAGE-NAME
           PERFORM GET-NOW-CS
           MOVE WS-NOW-CS-VALUE TO WS-START-CS
           MOVE 8000000 TO WS-LIMIT
           MOVE 0 TO WS-COUNTER
           PERFORM UNTIL WS-COUNTER >= WS-LIMIT
               ADD 1 TO WS-COUNTER
               ADD 1 TO WS-ACCUM
           END-PERFORM
           PERFORM GET-NOW-CS
           MOVE WS-NOW-CS-VALUE TO WS-END-CS
           PERFORM REPORT-ELAPSED

      *> ---- Stage 2: build a table then SEARCH ALL it repeatedly ----
           MOVE "STAGE 2 - BUILD+SEARCH ALL 30K" TO WS-STAGE-NAME
           PERFORM GET-NOW-CS
           MOVE WS-NOW-CS-VALUE TO WS-START-CS
           PERFORM VARYING WS-FILL-IDX FROM 1 BY 1
                   UNTIL WS-FILL-IDX > 30000
               COMPUTE WS-CODE(WS-FILL-IDX) =
                       2000000 + WS-FILL-IDX - 1
           END-PERFORM
           PERFORM VARYING WS-SEARCH-PASS FROM 1 BY 1
                   UNTIL WS-SEARCH-PASS > 30000
               COMPUTE WS-SEARCH-KEY = 2000000 + 29999 -
                   FUNCTION MOD(WS-SEARCH-PASS, 50)
               SEARCH ALL WS-ENTRY
                   AT END
                       CONTINUE
                   WHEN WS-CODE(WS-IDX) = WS-SEARCH-KEY
                       ADD 1 TO WS-FOUND-COUNT
               END-SEARCH
           END-PERFORM
           PERFORM GET-NOW-CS
           MOVE WS-NOW-CS-VALUE TO WS-END-CS
           PERFORM REPORT-ELAPSED

      *> ---- Stage 3: a COMPUTE-heavy loop ----
           MOVE "STAGE 3 - COMPUTE LOOP (3M)" TO WS-STAGE-NAME
           PERFORM GET-NOW-CS
           MOVE WS-NOW-CS-VALUE TO WS-START-CS
           MOVE 3000000 TO WS-LIMIT
           MOVE 0 TO WS-COUNTER
           PERFORM UNTIL WS-COUNTER >= WS-LIMIT
               COMPUTE WS-ACCUM = (WS-COUNTER * 3 + 7) / 2
               ADD 1 TO WS-COUNTER
           END-PERFORM
           PERFORM GET-NOW-CS
           MOVE WS-NOW-CS-VALUE TO WS-END-CS
           PERFORM REPORT-ELAPSED

           DISPLAY "=== PROFILING RUN END (found="
               WS-FOUND-COUNT ") ===".
           STOP RUN.

       GET-NOW-CS.
      *> FUNCTION CURRENT-DATE returns a 21-character string:
      *> YYYYMMDDHHMMSSss+HHMM. Pack HH/MM/SS/hundredths into one
      *> "total hundredths since midnight" number for easy
      *> subtraction. This is a stopwatch, not a calendar, so we
      *> do not need the date part at all.
           MOVE FUNCTION CURRENT-DATE TO WS-NOW
           COMPUTE WS-NOW-CS-VALUE =
               ((WS-NOW-HH * 60 + WS-NOW-MM) * 60 + WS-NOW-SS)
               * 100 + WS-NOW-CS.

       REPORT-ELAPSED.
      *> Elapsed time within the same run, expressed as total
      *> hundredths-of-a-second since midnight; safe as long as a
      *> single stage does not cross midnight (true for any stage
      *> that finishes in well under 24 hours).
           COMPUTE WS-ELAPSED-CS = WS-END-CS - WS-START-CS
           DISPLAY WS-STAGE-NAME ": " WS-ELAPSED-CS
               " hundredths of a second".
```

```bash
cobc -x -o step880_profiler step880_profiler.cob
./step880_profiler
```

**ผลลัพธ์จริง (2 รอบการรัน):**

```
=== PROFILING RUN START ===
STAGE 1 - ADD LOOP (8M TIMES) : 000000068 hundredths of a second
STAGE 2 - BUILD+SEARCH ALL 30K: 000000002 hundredths of a second
STAGE 3 - COMPUTE LOOP (3M)   : 000000084 hundredths of a second
=== PROFILING RUN END (found=0030000) ===

=== PROFILING RUN START ===
STAGE 1 - ADD LOOP (8M TIMES) : 000000066 hundredths of a second
STAGE 2 - BUILD+SEARCH ALL 30K: 000000002 hundredths of a second
STAGE 3 - COMPUTE LOOP (3M)   : 000000077 hundredths of a second
=== PROFILING RUN END (found=0030000) ===
```

### อธิบายจุดสำคัญ

- ผลลัพธ์แสดงให้เห็นชัดเจนว่า **Stage 2 (สร้างตารางและค้นหาด้วย `SEARCH ALL` 30,000 ครั้ง) เร็วกว่า
  Stage 1 และ Stage 3 มาก** (~2 เซนติวินาที เทียบกับ ~68 และ ~84 เซนติวินาที) แม้ Stage 2 จะ "ดูเหมือน"
  ทำงานซับซ้อนกว่า (สร้างตาราง + ค้นหา) — นี่คือตัวอย่างจริงของการที่ **"โค้ดที่ดูซับซ้อนกว่าอาจเร็วกว่าโค้ด
  ที่ดูเรียบง่าย"** เพราะใช้อัลกอริทึมที่มีความซับซ้อนเชิงเวลาต่ำกว่า (`SEARCH ALL` เป็น O(log n) ในขณะที่
  ลูป ADD/COMPUTE ธรรมดาเป็น O(n) แต่มีค่าคงที่ต่อรอบสูงกว่าในกรณีนี้)
- เทคนิคนี้**ไม่ต้องพึ่งเครื่องมือภายนอกใด ๆ เลย** — ใช้ได้แม้ในสภาพแวดล้อมที่รันผ่าน JCL batch job ที่ไม่มี
  สิทธิ์เข้าถึง shell โดยตรง เพียงแค่ดู Job Log ที่มี `DISPLAY` output ก็เพียงพอ
- ย่อหน้า `GET-NOW-CS` และ `REPORT-ELAPSED` สามารถแยกออกไปเป็น **Subprogram แยกต่างหาก** (ทบทวนจาก
  Part 031) เพื่อนำไปใช้ซ้ำในหลายโปรแกรมได้ทันที โดยไม่ต้องคัดลอกโค้ดตรรกะการคำนวณเวลาไปทุกที่

### ข้อควรระวัง

- **เทคนิคนี้ใช้ไม่ได้ถ้าช่วงเวลาที่วัดข้ามเที่ยงคืนพอดี** (เช่น เริ่มวัดตอน 23:59:58 แล้วจบตอน 00:00:03)
  เพราะค่า "จำนวนเศษ 1/100 วินาทีนับจากเที่ยงคืน" จะรีเซ็ตกลับเป็น 0 ทำให้ผลลัพธ์ติดลบหรือผิดพลาด — สำหรับ
  งาน Batch ที่มักรันข้ามคืนจริง ควรใช้ `FUNCTION CURRENT-DATE` เต็มรูปแบบรวมวันที่ด้วย ไม่ใช่แค่เวลาอย่างเดียว
  แบบในตัวอย่างนี้ที่ออกแบบมาเพื่อความเรียบง่ายในการสอนเท่านั้น
- ความละเอียดของ `FUNCTION CURRENT-DATE` คือ **1/100 วินาที (เซนติวินาที)** เท่านั้น ไม่เหมาะกับการวัดงาน
  ที่เสร็จเร็วกว่านั้นมาก (เช่น ต้องการวัดระดับไมโครวินาที) — สำหรับกรณีนั้นคำสั่ง `time` ของ Linux ยังคงแม่นยำ
  กว่า แต่สำหรับงาน Batch ที่มักใช้เวลาระดับวินาทีถึงนาที ความละเอียดระดับนี้เพียงพอแล้ว

### แบบฝึกหัดที่ 879.1

**โจทย์**: จงอธิบายว่าทำไม "Poor Man's Profiler" แบบในขั้นตอนนี้ถึงมีประโยชน์มากเป็นพิเศษสำหรับโปรแกรม
COBOL บน Mainframe จริง เมื่อเทียบกับการพึ่งพาคำสั่ง `time` ของ Linux เพียงอย่างเดียว

**เฉลยแนวทาง**: เพราะโปรแกรม COBOL บน Mainframe จริงมักรันผ่าน JCL batch job (ทบทวน Part 052-053)
ซึ่งไม่มีสภาพแวดล้อม shell แบบ Linux ให้เรียกคำสั่ง `time` ได้โดยตรง — ทุกอย่างต้องทำผ่านกลไกที่มีอยู่ใน
COBOL หรือ JCL เองเท่านั้น นอกจากนี้คำสั่ง `time` วัดได้แค่**เวลารวมทั้งโปรแกรม** แต่ Poor Man's Profiler
สามารถวัด**แต่ละช่วงงานย่อยภายในโปรแกรมเดียวกัน**ได้ (เช่น "ขั้นตอนอ่านไฟล์ใช้เวลาเท่าไหร่ เทียบกับขั้นตอน
คำนวณดอกเบี้ย") ซึ่งช่วยระบุคอขวดที่แท้จริงได้แม่นยำกว่ามาก ตรงกับหลักการ "Measure → Identify Bottleneck"
จาก Part 068 ขั้นตอนที่ 671 ที่ต้องรู้ว่าจุดไหนช้าจริงก่อนจะไปแก้ไข

---

## ขั้นตอนที่ 880: สร้างกรอบการตัดสินใจ (Decision Framework) จากทุกเบนช์มาร์กใน Part นี้

### ทำไมต้องมีกรอบการตัดสินใจ

ตลอดขั้นตอนที่ 871-879 เราวัดผลจริงของหลายเทคนิค (SEARCH เชิงเส้น, SEARCH ALL แบบ binary, ไฟล์
Indexed, การทำ Cache, การจัดขนาด Working Set, การ optimize COMPUTE) แต่ตัวเลขเปล่า ๆ ไม่มีประโยชน์
ถ้าไม่สามารถนำไปใช้ตัดสินใจในสถานการณ์งานจริงได้ทันที ขั้นตอนสุดท้ายนี้จะรวบรวมทุกบทเรียนให้เป็น
**กรอบการตัดสินใจที่เป็นโค้ดจริง** ไม่ใช่แค่ตารางในเอกสาร

### ตัวอย่างโค้ด: โปรแกรมช่วยตัดสินใจเลือกเทคนิค

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PERF-DECISION-GUIDE.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TABLE-SIZE           PIC 9(7) VALUE 0.
       01  WS-LOOKUP-FREQUENCY     PIC X(6) VALUE SPACES.
       01  WS-RECOMMENDATION       PIC X(40) VALUE SPACES.

       01  WS-SCENARIO-COUNT       PIC 9(2) VALUE 4.
       01  WS-IDX                  PIC 9(2).

       01  WS-SCENARIOS.
           05  WS-SCENARIO OCCURS 4 TIMES.
               10  WS-SC-SIZE          PIC 9(7).
               10  WS-SC-FREQ          PIC X(6).
               10  WS-SC-LABEL         PIC X(30).

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 0000100 TO WS-SC-SIZE(1)
           MOVE "LOW   " TO WS-SC-FREQ(1)
           MOVE "Small lookup table, rare access" TO WS-SC-LABEL(1)

           MOVE 0050000 TO WS-SC-SIZE(2)
           MOVE "HIGH  " TO WS-SC-FREQ(2)
           MOVE "Large table, searched constantly" TO WS-SC-LABEL(2)

           MOVE 5000000 TO WS-SC-SIZE(3)
           MOVE "LOW   " TO WS-SC-FREQ(3)
           MOVE "Huge master file, occasional read" TO WS-SC-LABEL(3)

           MOVE 0002000 TO WS-SC-SIZE(4)
           MOVE "HIGH  " TO WS-SC-FREQ(4)
           MOVE "Medium table, hot loop lookup" TO WS-SC-LABEL(4)

           DISPLAY "=== PERFORMANCE TECHNIQUE DECISION GUIDE ===".

           PERFORM VARYING WS-IDX FROM 1 BY 1
                   UNTIL WS-IDX > WS-SCENARIO-COUNT
               MOVE WS-SC-SIZE(WS-IDX) TO WS-TABLE-SIZE
               MOVE WS-SC-FREQ(WS-IDX) TO WS-LOOKUP-FREQUENCY
               PERFORM RECOMMEND-TECHNIQUE
               DISPLAY "SCENARIO " WS-IDX ": " WS-SC-LABEL(WS-IDX)
               DISPLAY "  SIZE=" WS-TABLE-SIZE
                       " FREQ=" WS-LOOKUP-FREQUENCY
               DISPLAY "  RECOMMENDATION: " WS-RECOMMENDATION
           END-PERFORM.

           STOP RUN.

       RECOMMEND-TECHNIQUE.
           IF WS-TABLE-SIZE > 1000000
               MOVE "INDEXED FILE (fits on disk, keyed access)"
                   TO WS-RECOMMENDATION
           ELSE
               IF WS-LOOKUP-FREQUENCY = "HIGH  "
                   MOVE "IN-MEMORY TABLE + SEARCH ALL (binary)"
                       TO WS-RECOMMENDATION
               ELSE
                   MOVE "IN-MEMORY TABLE + SEARCH (linear is fine)"
                       TO WS-RECOMMENDATION
               END-IF
           END-IF.
```

### ผลลัพธ์การรันจริง

```
=== PERFORMANCE TECHNIQUE DECISION GUIDE ===
SCENARIO 01: Small lookup table, rare acces
  SIZE=0000100 FREQ=LOW
  RECOMMENDATION: IN-MEMORY TABLE + SEARCH (linear is fine
SCENARIO 02: Large table, searched constant
  SIZE=0050000 FREQ=HIGH
  RECOMMENDATION: IN-MEMORY TABLE + SEARCH ALL (binary)
SCENARIO 03: Huge master file, occasional r
  SIZE=5000000 FREQ=LOW
  RECOMMENDATION: INDEXED FILE (fits on disk, keyed access
SCENARIO 04: Medium table, hot loop lookup
  SIZE=0002000 FREQ=HIGH
  RECOMMENDATION: IN-MEMORY TABLE + SEARCH ALL (binary)
```

### อธิบายจุดสำคัญ

- เกณฑ์ `WS-TABLE-SIZE > 1000000` มาจากขั้นตอนที่ 875 ที่พิสูจน์ว่าตารางในหน่วยความจำขนาดใหญ่มาก
  เริ่มไม่คุ้มค่าเมื่อเทียบกับไฟล์ Indexed (ทั้งเรื่องพื้นที่หน่วยความจำและเวลาโหลดตอนเริ่มโปรแกรม)
- เกณฑ์ความถี่ในการค้นหา (`HIGH`/`LOW`) มาจากขั้นตอนที่ 873-874: ถ้าค้นหาบ่อยมาก ส่วนต่าง ~277 เท่า
  ระหว่าง `SEARCH` กับ `SEARCH ALL` จะสะสมกลายเป็นความแตกต่างมหาศาล แต่ถ้าค้นหาไม่กี่ครั้ง ความแตกต่าง
  นี้แทบไม่มีผลกระทบต่อเวลาทำงานรวม จึงไม่คุ้มที่จะเพิ่มความซับซ้อนของการเรียงข้อมูลให้ `SEARCH ALL`
- โครงสร้างแบบนี้ (ตาราง scenario + subprogram ตัดสินใจ) สามารถขยายเพิ่มเกณฑ์อื่น ๆ ได้ง่าย เช่น
  จำนวนครั้งที่ต้องอ่านไฟล์ซ้ำ (จากขั้นตอนที่ 876) หรือข้อจำกัดด้านหน่วยความจำของเครื่องเป้าหมาย
  (จากขั้นตอนที่ 877)

### ข้อควรระวัง

- กรอบการตัดสินใจนี้เป็น**จุดเริ่มต้นที่ดี ไม่ใช่กฎตายตัว** เกณฑ์ตัวเลข (1,000,000 รายการ, HIGH/LOW)
  ควรปรับตามผลเบนช์มาร์กจริงของระบบและฮาร์ดแวร์ที่ใช้งานจริงเสมอ ตามหลักการ "วัดผลก่อนเชื่อ" ที่ย้ำ
  มาตลอดทั้ง Part
- อย่าลืมว่าเกณฑ์เหล่านี้วัดจากสภาพแวดล้อมทดสอบเฉพาะของหลักสูตรนี้ (ดูขั้นตอนที่ 872) ตัวเลขจริงบน
  เครื่อง Mainframe หรือเซิร์ฟเวอร์ Production อาจแตกต่างไปมาก แต่**หลักการเปรียบเทียบเชิงสัมพัทธ์**
  (Indexed เหมาะกับข้อมูลใหญ่, SEARCH ALL เหมาะกับการค้นหาบ่อย) ยังคงใช้ได้เสมอ

### แบบฝึกหัดที่ 880.1

**โจทย์**: จงเพิ่ม scenario ที่ 5 เข้าไปในตาราง `WS-SCENARIOS`: ตารางขนาด 300,000 รายการ ที่ถูกค้นหา
บ่อยมาก (`HIGH`) แล้ววิเคราะห์ว่าโปรแกรมควรแนะนำเทคนิคใดตามเกณฑ์ปัจจุบัน และเกณฑ์นี้เหมาะสมหรือไม่

**เฉลยแนวทาง**: ตามเกณฑ์ปัจจุบัน (`WS-TABLE-SIZE > 1000000`) ตาราง 300,000 รายการยังไม่เกินเกณฑ์
จึงจะได้คำแนะนำ "IN-MEMORY TABLE + SEARCH ALL (binary)" ซึ่งสอดคล้องกับผลเบนช์มาร์กในขั้นตอนที่ 874
ที่แสดงว่า `SEARCH ALL` บนตาราง 50,000 รายการยังคงเร็วมาก (~0.042 วินาที) อย่างไรก็ตาม ในทางปฏิบัติ
ควรพิจารณาเพิ่มเติมว่าตาราง 300,000 รายการใช้หน่วยความจำเท่าไหร่ (ขึ้นกับขนาดของแต่ละ record) และ
เครื่องเป้าหมายมีหน่วยความจำเพียงพอหรือไม่ — หากมีข้อจำกัดด้านหน่วยความจำ อาจต้องปรับเกณฑ์ให้ต่ำกว่า
1,000,000 รายการลงมา

---

## สรุปท้ายบท

Part 088 ขยายเรื่อง Performance ของ COBOL จาก Part 068 ให้ลึกและกว้างขึ้นอีกขั้น ด้วยเบนช์มาร์กจริงและ
ตัวเลขที่วัดได้จริงทุกกรณี:

- **ขั้นตอนที่ 871**: กรอบคิดเรื่องความซับซ้อนเชิงอัลกอริทึม — พิสูจน์ด้วยตัวเลขจริงว่าลูปซ้อนลูปแบบ O(n²)
  ที่ n=3,000 ใช้เวลา 0.65 วินาที แต่ที่ n=6,000 (2 เท่า) กลับใช้เวลา 2.6 วินาที (4 เท่า) ตรงตามทฤษฎี O(n²)
- **ขั้นตอนที่ 872**: ระเบียบวิธีสร้าง Benchmark ที่ยุติธรรม พร้อมเล่าบั๊กจริงที่พบระหว่างเตรียมเนื้อหา — การ
  ล้น column 72 ด้วยโค้ดภาษาอังกฤษล้วน ทำให้ตัวเลข `5000` กลายเป็น `500` แบบเงียบ ๆ
- **ขั้นตอนที่ 873-875**: เปรียบเทียบ 3 เทคนิคบนตาราง 50,000 รายการชุดเดียวกัน — `SEARCH` (Linear)
  ~11.6 วินาที, `SEARCH ALL` (Binary) ~0.042 วินาที (~277× เร็วกว่า), ไฟล์ Indexed (ISAM) ~0.10 วินาที
  (~116× เร็วกว่า Linear แต่ช้ากว่า SEARCH ALL ในหน่วยความจำ ~2.4 เท่า) พร้อมกรอบการตัดสินใจว่าเมื่อไร
  ควรใช้เทคนิคไหน
- **ขั้นตอนที่ 876**: กลยุทธ์ Cache ข้อมูลในหน่วยความจำเพื่อลดการอ่านไฟล์ซ้ำ — ลดจำนวนครั้ง I/O จาก 25,000
  เหลือ 5,000 ครั้ง (ลด 80%) ทำให้เร็วขึ้นประมาณ 3 เท่า
- **ขั้นตอนที่ 877**: ผลกระทบของขนาด Working Set ต่อ CPU Cache — พิสูจน์ว่ามีผลจริง (~12% ช้ากว่า) แต่
  เล็กกว่าความแตกต่างระดับอัลกอริทึมมาก
- **ขั้นตอนที่ 878**: การ optimize นิพจน์ COMPUTE — พบผลลัพธ์ที่ขัดสัญชาตญาณอีกครั้ง (การเก็บค่าไว้ในตัวแปร
  ชั่วคราวกลับช้ากว่า 15%) และพิสูจน์ต้นทุนจริงของ `ON SIZE ERROR` (~62% ช้าลงเมื่อไม่เคย error จริง)
- **ขั้นตอนที่ 879**: สร้าง "Poor Man's Profiler" ด้วย `FUNCTION CURRENT-DATE` เอง โดยไม่ต้องพึ่งเครื่องมือ
  ภายนอก เหมาะสำหรับสภาพแวดล้อม Mainframe จริงที่ไม่มี shell ให้เรียก `time`

บทเรียนที่สำคัญที่สุดที่ย้ำซ้ำตลอดทั้ง Part 068 และ Part 088 คือ: **"วัดผลก่อนเชื่อเสมอ"** — แม้แต่หลักการที่
ฟังดูสมเหตุสมผลที่สุด (เก็บค่าที่คำนวณซ้ำไว้ใช้ใหม่, ตารางเล็กย่อมเร็วกว่าเพราะพอดี cache) ก็อาจไม่จริงเสมอไป
ในสภาพแวดล้อมการทำงานจริงของคุณ ทักษะนี้สำคัญมากขึ้นเรื่อย ๆ เมื่อเราเข้าใกล้จุดสิ้นสุดของหลักสูตร ที่ต้อง
นำความรู้ทั้งหมดมาประยุกต์กับระบบระดับองค์กรจริง

Part ถัดไปจะเปลี่ยนมุมมองจาก "ความเร็ว" ไปสู่ "ความปลอดภัยระดับองค์กร" — **Part 089: Security ระดับ
Enterprise สำหรับแอปพลิเคชัน COBOL** จะขยายจาก Part 069 (ที่เน้น RACF/Mainframe) ไปสู่การเขียนโค้ด
ที่ปลอดภัยสำหรับแอปพลิเคชัน COBOL ในบริบทกว้างขึ้น เช่น การป้องกัน SQL Injection เมื่อ COBOL เรียกใช้
ฐานข้อมูล, การปกปิดข้อมูลอ่อนไหวในผลลัพธ์, และ Audit Logging ระดับองค์กร

**[ไปยัง Part 089: Security ระดับ Enterprise สำหรับแอปพลิเคชัน COBOL →](part-089-enterprise-security.md)**
