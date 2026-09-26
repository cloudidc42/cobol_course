# Part 015: 🎯 โปรเจกต์เฟส 1: เครื่องคิดเลขและระบบคำนวณเกรดนักเรียน (ขั้นตอนที่ 141–150)

## คำนำของ Part นี้

เดินทางมาถึงจุดสำคัญแล้ว! ตั้งแต่ Part 003 ถึง Part 014 คุณได้เรียนรู้องค์ประกอบพื้นฐานเกือบทั้งหมด
ของภาษา COBOL ทีละชิ้น: โครงสร้าง 4 Divisions และกฎคอลัมน์ (Part 003), การประกาศตัวแปรใน
WORKING-STORAGE SECTION (Part 005), PICTURE Clause (Part 006), การรับ-แสดงผลข้อมูล (Part 007),
การย้ายข้อมูลด้วย MOVE (Part 008), เลขคณิตด้วย ADD/SUBTRACT/MULTIPLY/DIVIDE/COMPUTE (Part 009),
เงื่อนไข IF-ELSE และ Condition Name (Part 010), EVALUATE (Part 011), การวนลูปด้วย PERFORM
(Part 012-013), และการจัดโครงสร้างโปรแกรมด้วย Paragraph/Section (Part 014)

Part นี้คือ **โปรเจกต์รวบยอดเฟส 1 (Phase 1 Milestone Project)** ซึ่งจะไม่มีเนื้อหาใหม่ทางไวยากรณ์
แต่จะพาคุณ **นำทุกสิ่งที่เรียนมาทั้งหมดมาประกอบร่างเป็นโปรแกรมที่ใช้งานได้จริง 2 โปรแกรม**:

1. **เครื่องคิดเลข (Calculator)** — โปรแกรมเมนูแบบวนลูปที่รับตัวเลข 2 จำนวนและคำนวณผลตามที่ผู้ใช้เลือก
   พร้อมป้องกันการหารด้วยศูนย์อย่างถูกต้อง
2. **ระบบคำนวณเกรดนักเรียน (Student Grade Calculator)** — โปรแกรมที่รับคะแนนนักเรียนหลายคนเข้าตาราง
   (OCCURS), คำนวณค่าเฉลี่ย/คะแนนสูงสุด/ต่ำสุด, ตัดเกรดด้วยเงื่อนไขช่วงคะแนน, และพิมพ์รายงานสรุปที่จัดรูปแบบ
   สวยงาม

ทั้งสองโปรแกรมนี้เป็น **โปรแกรมสมบูรณ์ที่คอมไพล์และรันได้จริง 100%** ด้วย GnuCOBOL (เวอร์ชัน 4.0-early
ที่ใช้ทดสอบเนื้อหาทั้งหมดในหลักสูตรนี้) คุณสามารถคัดลอกโค้ดในบทนี้ไปคอมไพล์และรันบนเครื่องของคุณเองได้ทันที
และทุกตัวอย่างผลลัพธ์ที่แสดงในบทนี้คือผลลัพธ์จริงที่ได้จากการรันโค้ดจริง ไม่ใช่ผลลัพธ์ที่แต่งขึ้น

### หมายเหตุสำคัญเรื่องการทดสอบโปรแกรมที่ใช้ `ACCEPT`

โปรแกรมทั้งสองในบทนี้เป็นโปรแกรมแบบโต้ตอบ (interactive) ที่ใช้ `ACCEPT` รับข้อมูลจากผู้ใช้ผ่านคีย์บอร์ด
คำถามที่ตามมาตามธรรมชาติคือ "แล้วเราจะทดสอบโปรแกรมแบบนี้อย่างอัตโนมัติได้อย่างไร ในเมื่อไม่มีใครนั่งพิมพ์
ให้จริง ๆ ตอนเตรียมเอกสารนี้?"

คำตอบคือ **GnuCOBOL รันไทม์อ่านค่าจาก `ACCEPT` (แบบไม่ระบุ `FROM` clause) จาก Standard Input (stdin)
ของโปรเซส** ซึ่งหมายความว่าเราสามารถ **redirect ไฟล์ข้อความหรือส่งค่าผ่าน pipe เข้าไปแทนการพิมพ์จริง**
ได้โดยตรง เช่น:

```bash
cobc -x -o calculator calculator.cob
./calculator < my_input.txt
```

หรือ

```bash
printf -- "1\n10\n5\nY\n5\n" | ./calculator
```

เทคนิคนี้จำลองพฤติกรรมผู้ใช้พิมพ์ค่าแล้วกด Enter ทีละบรรทัดได้อย่างสมบูรณ์แบบ และ **นี่คือวิธีที่ใช้ทดสอบ
ทุกตัวอย่างในบทนี้จริง** — โค้ดทุกตัวที่แสดงคือซอร์สโค้ดตัวจริงที่ถูกคอมไพล์และรัน ไม่ใช่เวอร์ชันเขียนใหม่
แยกต่างหากเพื่อการทดสอบ ผลลัพธ์ที่เห็นคือผลลัพธ์จริงที่ได้จากการป้อนข้อมูลผ่าน stdin เข้าไปในโปรแกรม
ตัวเดียวกันกับที่คุณจะรันบนเครื่องของคุณเองแบบโต้ตอบ

**ข้อแตกต่างเล็กน้อยที่ควรทราบ**: เมื่อรันโปรแกรมแบบโต้ตอบจริงบน terminal ตัว terminal เองจะ echo
(แสดงซ้ำ) ตัวอักษรที่คุณพิมพ์กลับมาบนหน้าจอโดยอัตโนมัติ ทำให้เห็นค่าที่พิมพ์ปรากฏต่อจากข้อความ prompt
แต่เมื่อ redirect ข้อมูลจากไฟล์หรือ pipe (ซึ่งไม่ใช่ terminal จริง) จะไม่มีการ echo ค่าที่ "พิมพ์" กลับมา
ผลลัพธ์ที่แสดงในบทนี้จึงแสดงเฉพาะสิ่งที่โปรแกรม `DISPLAY` ออกมาเท่านั้น (ไม่เห็นค่าตัวเลขที่ "ป้อน" แทรกอยู่
ในบรรทัด prompt) แต่ **ตรรกะการคำนวณและผลลัพธ์ทั้งหมดเป็นค่าจริงที่ถูกต้อง 100%** ตามข้อมูลที่ป้อนเข้าไป

---

## ขั้นตอนที่ 141: ภาพรวมโปรเจกต์และการออกแบบระบบทั้งสอง

### เป้าหมายของโปรเจกต์

ก่อนลงมือเขียนโค้ดสักบรรทัด นักพัฒนามืออาชีพจะเริ่มจาก **การกำหนดความต้องการ (Requirements)** และ
**การออกแบบ (Design)** เสมอ — นี่คือขั้นตอนที่มักถูกมองข้ามโดยผู้เริ่มต้น แต่เป็นสิ่งที่แยกความแตกต่าง
ระหว่างโปรแกรมที่ "เขียนแล้วใช้งานได้บังเอิญ" กับโปรแกรมที่ "ออกแบบมาให้ทำงานถูกต้องและดูแลรักษาได้"

### ความต้องการของโปรแกรมที่ 1: เครื่องคิดเลข (Calculator)

| ข้อ | ความต้องการ |
|---|---|
| 1 | แสดงเมนูให้ผู้ใช้เลือกการดำเนินการ: บวก, ลบ, คูณ, หาร, หรือออกจากโปรแกรม |
| 2 | รับตัวเลข 2 จำนวนจากผู้ใช้ (ต้องรองรับเลขติดลบและเลขทศนิยม) |
| 3 | คำนวณผลลัพธ์ตามการดำเนินการที่เลือก และแสดงผล |
| 4 | ป้องกันการหารด้วยศูนย์ ไม่ให้โปรแกรมแสดงผลลัพธ์ที่ผิดพลาดแบบเงียบ ๆ |
| 5 | ถามผู้ใช้ว่าต้องการคำนวณต่อหรือไม่ หลังจากคำนวณเสร็จแต่ละครั้ง |
| 6 | วนกลับไปที่เมนูจนกว่าผู้ใช้จะเลือกออกจากโปรแกรม |

### ความต้องการของโปรแกรมที่ 2: ระบบคำนวณเกรดนักเรียน (Student Grade Calculator)

| ข้อ | ความต้องการ |
|---|---|
| 1 | รับจำนวนนักเรียนจากผู้ใช้ (จำกัดไม่เกิน 20 คน เพื่อความง่ายในระดับนี้ของหลักสูตร) |
| 2 | รับคะแนนของนักเรียนแต่ละคน (0-100) เก็บลงตาราง พร้อมตรวจสอบความถูกต้องของข้อมูล |
| 3 | คำนวณคะแนนรวม, คะแนนเฉลี่ย, คะแนนสูงสุด, และคะแนนต่ำสุด |
| 4 | ตัดเกรดแต่ละคนตามช่วงคะแนน (A: 90-100, B: 80-89, C: 70-79, D: 60-69, F: 0-59) |
| 5 | นับจำนวนนักเรียนที่ผ่าน (เกรด D ขึ้นไป) และไม่ผ่าน (เกรด F) |
| 6 | พิมพ์รายงานสรุปที่จัดรูปแบบให้อ่านง่าย (ตาราง + สรุปสถิติท้ายรายงาน) |

### สถาปัตยกรรมโปรแกรมในระดับหลักสูตรเฟส 1

จุดสำคัญที่ต้องเข้าใจก่อนเริ่มออกแบบ: ในเฟส 1 ของหลักสูตรนี้ เรา **ยังไม่ได้เรียน** คำสั่ง `COPY`
(Copybook, Part 033) หรือ `CALL` (Subprogram, Part 031-032) ที่ใช้แบ่งโปรแกรมออกเป็น**หลายไฟล์จริง**
ดังนั้นสถาปัตยกรรม "การแบ่งโมดูล" ของโปรเจกต์นี้จะใช้เครื่องมือที่เรามีอยู่แล้วจาก Part 014 เท่านั้น
คือการแบ่งเป็น **Paragraph ที่มีชื่อตามธรรมเนียม Numbered Paragraph Convention** (0000/1000/2000/...)
ภายใน**ไฟล์เดียว** — นี่คือวิธีที่ถูกต้องและเหมาะสมที่สุดสำหรับความรู้ ณ จุดนี้ของหลักสูตร เมื่อไปถึง
Part 033-034 คุณจะได้เรียนวิธีแยกโค้ดที่ใช้ร่วมกันออกเป็น Copybook และ Subprogram หลายไฟล์จริง ๆ
ซึ่งจะนำมาใช้เต็มรูปแบบในโปรเจกต์รวบยอดเฟส 2 (Part 035)

### แผนผังโครงสร้าง Paragraph ของเครื่องคิดเลข

```
0000-MAIN-PROCESS       (ควบคุมลำดับการทำงานหลักทั้งหมด)
├── 1000-SHOW-WELCOME       (แสดงข้อความต้อนรับ)
├── 2000-SHOW-MENU          (แสดงเมนูตัวเลือก)
├── 3000-GET-CHOICE         (รับตัวเลือกจากผู้ใช้)
├── 4000-GET-NUMBERS        (รับตัวเลข 2 จำนวน)
├── 5000-CALCULATE          (คำนวณตามตัวเลือก พร้อมป้องกันหารศูนย์)
├── 6000-SHOW-RESULT        (แสดงผลลัพธ์)
└── 7000-ASK-AGAIN          (ถามว่าจะคำนวณต่อหรือไม่)
```

### แผนผังโครงสร้าง Paragraph ของระบบคำนวณเกรด

```
0000-MAIN-PROCESS            (ควบคุมลำดับการทำงานหลักทั้งหมด)
├── 1000-SHOW-WELCOME            (แสดงข้อความต้อนรับ)
├── 2000-GET-STUDENT-COUNT       (รับจำนวนนักเรียน พร้อมตรวจสอบ)
├── 3000-COLLECT-SCORES          (รับคะแนนทีละคนเข้าตาราง OCCURS)
├── 4000-CALCULATE-STATISTICS    (คำนวณรวม/เฉลี่ย/สูงสุด/ต่ำสุด)
├── 5000-PRINT-REPORT            (พิมพ์รายงาน เรียก 5100 ต่อแต่ละคน)
└── 5100-ASSIGN-GRADE            (ตัดเกรดคนเดียวตามช่วงคะแนน)
```

### การออกแบบ WORKING-STORAGE ของเครื่องคิดเลข

| ตัวแปร | PICTURE | หน้าที่ |
|---|---|---|
| `WS-MENU-CHOICE` | `9(1)` | เก็บตัวเลือกเมนู (1-5) |
| `WS-NUM-1`, `WS-NUM-2` | `S9(7)V99` | ตัวเลขนำเข้า 2 จำนวน (มีเครื่องหมาย รองรับทศนิยม 2 ตำแหน่ง) |
| `WS-CALC-RESULT` | `S9(9)V99` | ผลลัพธ์ (ขนาดใหญ่กว่าตัวตั้งเพื่อรองรับผลคูณที่มีจำนวนหลักมากขึ้น) |
| `WS-AGAIN-ANSWER` | `X(1)` + 88-level | คำตอบ Y/N ว่าจะคำนวณต่อหรือไม่ |
| `WS-DIVIDE-BY-ZERO-FLAG` | `X(1)` + 88-level | flag บอกว่ารอบนี้เกิดการหารด้วยศูนย์หรือไม่ |

สังเกตว่า `WS-CALC-RESULT` ใช้ `PIC S9(9)V99` ในขณะที่ตัวตั้งใช้ `PIC S9(7)V99` เท่านั้น — นี่คือการ
นำความรู้จาก Part 009 ขั้นตอนที่ 83 มาใช้ (ข้อควรระวังเรื่อง MULTIPLY ทำให้จำนวนหลักเพิ่มขึ้นอย่างรวดเร็ว)
เราจึงจงใจให้ฟิลด์ผลลัพธ์มีขนาดใหญ่กว่าเผื่อไว้ ป้องกันการตัดหลักสูงทิ้งแบบเงียบ ๆ ตามที่เรียนใน Part 008

### การออกแบบ WORKING-STORAGE ของระบบคำนวณเกรด

| ตัวแปร | PICTURE | หน้าที่ |
|---|---|---|
| `WS-STUDENT-COUNT` | `9(2)` | จำนวนนักเรียนทั้งหมด (1-20) |
| `WS-STUDENT-IDX` | `9(2)` | ตัวนับ/subscript สำหรับวนลูปเข้าถึงตาราง |
| `WS-STUDENT-SCORE` (ตาราง) | `9(3) OCCURS 20 TIMES` | คะแนนของนักเรียนแต่ละคน |
| `WS-SCORE-TOTAL` | `9(5)` | ผลรวมคะแนนทั้งหมด |
| `WS-SCORE-AVERAGE` | `9(3)V99` | คะแนนเฉลี่ย (ปัดเศษ) |
| `WS-MAX-SCORE`, `WS-MIN-SCORE` | `9(3)` | คะแนนสูงสุด/ต่ำสุด |
| `WS-CURRENT-SCORE` + 88-level ช่วงคะแนน | `9(3)` | ใช้ตัดเกรดทีละคนด้วย Condition Name |
| `WS-CURRENT-GRADE` | `X(1)` | ตัวอักษรเกรดที่ตัดได้ (A/B/C/D/F) |
| `WS-PASS-COUNT`, `WS-FAIL-COUNT` | `9(2)` | ตัวนับจำนวนคนผ่าน/ไม่ผ่าน |

การใช้ `OCCURS 20 TIMES` ในขั้นตอนนี้เป็นการ**หยิบยืม**แนวคิดจาก Part 013 ขั้นตอนที่ 126 มาใช้ล่วงหน้า
เล็กน้อย (รายละเอียดเต็มรูปแบบของ `OCCURS` เช่น `INDEXED BY`, ตารางหลายมิติ จะสอนเต็มใน Part 016-018)
แต่พื้นฐานที่เรียนมาเพียงพอแล้วสำหรับการสร้างตารางเก็บคะแนนแบบมิติเดียวในโปรเจกต์นี้

### พิสูจน์ว่าโครงร่างคอมไพล์ผ่าน (Skeleton Compile Check)

ก่อนเติมเนื้อหาจริง ลองสร้าง **โครงร่าง (skeleton)** ของเครื่องคิดเลขที่มีแค่โครงสร้าง Paragraph
ครบทุกตัวตามแผนผังข้างต้น แต่ยังไม่มี logic จริง เพื่อยืนยันว่าโครงสร้างที่ออกแบบไว้ถูกต้องตามหลักไวยากรณ์
ก่อนจะเริ่มเติมรายละเอียดในขั้นตอนถัดไป:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CALC-SKELETON.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MENU-CHOICE      PIC 9(1)      VALUE 0.

       PROCEDURE DIVISION.
       0000-MAIN-PROCESS.
           PERFORM 1000-SHOW-WELCOME
           PERFORM 2000-SHOW-MENU
           PERFORM 3000-GET-CHOICE
           PERFORM 4000-GET-NUMBERS
           PERFORM 5000-CALCULATE
           PERFORM 6000-SHOW-RESULT
           PERFORM 7000-ASK-AGAIN
           STOP RUN.

       1000-SHOW-WELCOME.
           DISPLAY "1000-SHOW-WELCOME: not yet implemented".

       2000-SHOW-MENU.
           DISPLAY "2000-SHOW-MENU: not yet implemented".

       3000-GET-CHOICE.
           DISPLAY "3000-GET-CHOICE: not yet implemented".

       4000-GET-NUMBERS.
           DISPLAY "4000-GET-NUMBERS: not yet implemented".

       5000-CALCULATE.
           DISPLAY "5000-CALCULATE: not yet implemented".

       6000-SHOW-RESULT.
           DISPLAY "6000-SHOW-RESULT: not yet implemented".

       7000-ASK-AGAIN.
           DISPLAY "7000-ASK-AGAIN: not yet implemented".
```

**ผลลัพธ์จริงจากการคอมไพล์และรัน**:

```
1000-SHOW-WELCOME: not yet implemented
2000-SHOW-MENU: not yet implemented
3000-GET-CHOICE: not yet implemented
4000-GET-NUMBERS: not yet implemented
5000-CALCULATE: not yet implemented
6000-SHOW-RESULT: not yet implemented
7000-ASK-AGAIN: not yet implemented
```

โครงร่างนี้ยืนยันสองสิ่ง: (1) ลำดับการเรียก Paragraph ทั้ง 7 ตัวจาก `0000-MAIN-PROCESS` ถูกต้องตาม
ไวยากรณ์และคอมไพล์ผ่าน (2) ลำดับการทำงานจริงตรงกับที่ออกแบบไว้ในแผนผังทุกประการ — เทคนิคการเขียน
"โครงร่างก่อน แล้วค่อยเติมเนื้อหา" (skeleton-first, fill-in-later) นี้เป็นวิธีที่มีประโยชน์มากในการพัฒนา
โปรแกรมขนาดใหญ่ เพราะช่วยยืนยันภาพรวมของการไหลของโปรแกรม (control flow) ก่อนที่จะลงรายละเอียด

### ข้อควรระวัง

- การออกแบบล่วงหน้าไม่ได้แปลว่าต้องสมบูรณ์แบบตั้งแต่ครั้งแรก ในทางปฏิบัติแผนผังและ WORKING-STORAGE
  มักถูกปรับแก้เล็กน้อยระหว่างการพัฒนาจริง (เช่นในบทนี้เอง ฟิลด์ `WS-CURRENT-SCORE` พร้อม 88-level
  ถูกออกแบบมาเพื่อใช้ตัดเกรดทีละคน แยกจากตาราง `WS-STUDENT-SCORE` ที่เก็บข้อมูลถาวร)
- อย่าข้ามขั้นตอนการออกแบบไปเขียนโค้ดทันทีในโปรเจกต์ที่มีความซับซ้อนระดับนี้ขึ้นไป เพราะมักนำไปสู่การ
  ต้องเขียนใหม่บางส่วนเมื่อพบว่าโครงสร้างข้อมูลที่เลือกไว้ไม่รองรับความต้องการที่ตามมาทีหลัง

### แบบฝึกหัดที่ 141.1

**โจทย์**: จากแผนผัง Paragraph ของระบบคำนวณเกรด จงอธิบายว่าทำไม `5100-ASSIGN-GRADE` จึงถูกแยกออกมา
เป็น Paragraph ต่างหาก แทนที่จะเขียน logic ตัดเกรดไว้ใน `5000-PRINT-REPORT` โดยตรง

**เฉลยแนวทาง**: เพราะ `5100-ASSIGN-GRADE` ต้องถูกเรียก**ซ้ำหนึ่งครั้งต่อนักเรียนหนึ่งคน**ในขณะที่
`5000-PRINT-REPORT` มีหน้าที่ควบคุมภาพรวมของการพิมพ์รายงานทั้งหมด (หัวรายงาน, วนลูปพิมพ์แต่ละแถว,
สรุปท้ายรายงาน) การแยก logic การตัดเกรดออกมาต่างหากทำให้ `5000-PRINT-REPORT` อ่านง่ายขึ้น (เห็นภาพรวม
ได้ทันทีว่า "วนพิมพ์ทีละคน โดยให้ 5100 เป็นคนตัดเกรด") และยังสอดคล้องกับหลักการแบ่งความรับผิดชอบ
(separation of concerns) ที่เรียนมาใน Part 014 ขั้นตอนที่ 137

---

## ขั้นตอนที่ 142: เริ่มสร้างเครื่องคิดเลข — เมนูหลักด้วย PERFORM + EVALUATE

### แนวคิด

ขั้นตอนนี้เราจะเติมเนื้อหาจริงให้กับ Paragraph ที่ควบคุมเมนู: `1000-SHOW-WELCOME`, `2000-SHOW-MENU`,
`3000-GET-CHOICE` และโครงลูปหลักใน `0000-MAIN-PROCESS` ส่วนการคำนวณจริงยังไม่เติม (จะทำในขั้นตอนที่ 143)
เพื่อให้เห็นทีละขั้นว่าโปรแกรมค่อย ๆ ก่อร่างขึ้นมาอย่างไร

เราเลือกใช้ `PERFORM WITH TEST AFTER UNTIL WS-MENU-CHOICE = 5` เป็นโครงลูปหลัก — นำความรู้จาก
Part 013 ขั้นตอนที่ 125 มาใช้โดยตรง เหตุผลที่เลือก `TEST AFTER` (ทำงานก่อนอย่างน้อย 1 รอบเสมอ แล้ว
ค่อยตรวจสอบเงื่อนไข) แทน `TEST BEFORE` (ค่าเริ่มต้น) คือ **เราต้องการให้เมนูแสดงผลอย่างน้อยหนึ่งครั้งเสมอ**
ก่อนที่จะมีการตรวจสอบว่า "ผู้ใช้เลือกออกหรือยัง" ซึ่งตรงกับพฤติกรรมตามธรรมชาติของโปรแกรมเมนูทุกโปรแกรม
(ไม่มีทางที่ผู้ใช้จะเลือกออกได้ก่อนเห็นเมนูเลยสักครั้ง)

### ตัวอย่างโค้ด (เวอร์ชันขั้นตอนที่ 142 — ยังไม่มีการคำนวณจริง)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP142.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MENU-CHOICE      PIC 9(1)      VALUE 0.

       PROCEDURE DIVISION.
       0000-MAIN-PROCESS.
           PERFORM 1000-SHOW-WELCOME
           PERFORM WITH TEST AFTER
                   UNTIL WS-MENU-CHOICE = 5
               PERFORM 2000-SHOW-MENU
               PERFORM 3000-GET-CHOICE
               EVALUATE WS-MENU-CHOICE
                   WHEN 1 THRU 4
                       DISPLAY "  (Calculation feature coming in the "
                           "next step.)"
                   WHEN 5
                       DISPLAY "Exiting calculator. Goodbye!"
                   WHEN OTHER
                       DISPLAY "Invalid choice, please select 1-5."
               END-EVALUATE
           END-PERFORM
           STOP RUN.

       1000-SHOW-WELCOME.
           DISPLAY "=================================".
           DISPLAY "   SIMPLE COBOL CALCULATOR".
           DISPLAY "=================================".

       2000-SHOW-MENU.
           DISPLAY " ".
           DISPLAY "1. Add        2. Subtract".
           DISPLAY "3. Multiply   4. Divide".
           DISPLAY "5. Exit".

       3000-GET-CHOICE.
           DISPLAY "Enter your choice (1-5): " WITH NO ADVANCING.
           ACCEPT WS-MENU-CHOICE.
```

คอมไพล์และรันด้วย:

```bash
cobc -x -o step142 step142.cob
printf -- "1\n9\n5\n" | ./step142
```

### ผลลัพธ์จริงที่ได้ (ป้อนค่า 1, 9, 5 ตามลำดับ)

```
=================================
   SIMPLE COBOL CALCULATOR
=================================
 
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5):   (Calculation feature coming in the next step.)
 
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Invalid choice, please select 1-5.
 
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Exiting calculator. Goodbye!
```

### อธิบายโค้ดทีละส่วน

- ป้อนค่า `1` ครั้งแรก → เข้า `WHEN 1 THRU 4` → แสดงข้อความชั่วคราวว่ายังไม่ได้ทำจริง แล้ว**วนกลับไป
  แสดงเมนูใหม่โดยอัตโนมัติ** เพราะ `WS-MENU-CHOICE` ยังไม่เท่ากับ 5
- ป้อนค่า `9` ครั้งที่สอง → ไม่ตรงกับ `1 THRU 4` และไม่ตรงกับ `5` → ตกไปที่ `WHEN OTHER` → แสดง
  ข้อความแจ้งตัวเลือกไม่ถูกต้อง แล้ววนกลับไปแสดงเมนูอีกครั้ง (สังเกตว่าโปรแกรม**ไม่พังหรือหยุดทำงาน**
  แม้ผู้ใช้จะป้อนค่าที่ไม่ได้อยู่ในเมนูเลย)
- ป้อนค่า `5` ครั้งที่สาม → เข้า `WHEN 5` → แสดงข้อความอำลา → เงื่อนไข `UNTIL WS-MENU-CHOICE = 5`
  เป็นจริงพอดี → ลูปหยุดทำงาน → `STOP RUN.` จบโปรแกรม

### ข้อควรระวัง

- `WS-MENU-CHOICE` เป็น `PIC 9(1)` รับได้แค่หลักเดียว (0-9) หากผู้ใช้พิมพ์ตัวเลขหลายหลัก เช่น `12`
  ค่าที่เก็บจะถูกตัดเหลือแค่หลักสุดท้ายตามกฎ MOVE/ACCEPT ที่เรียนใน Part 008 ขั้นตอนที่ 72 (ตัดหลักสูง
  ทิ้งแบบเงียบ ๆ) ทำให้ `12` กลายเป็น `2` ซึ่งบังเอิญเป็นตัวเลือกที่ถูกต้อง (Subtract) — นี่เป็นความเสี่ยง
  แฝงที่ควรระวังในโปรแกรมจริง (การตรวจสอบ `IS NUMERIC` หรือรับเป็นฟิลด์ตัวอักษรก่อนตรวจสอบอาจปลอดภัยกว่า
  ในระบบระดับ production แต่เกินขอบเขตของโปรเจกต์ระดับเฟส 1 นี้)
- สังเกตว่า `DISPLAY ... WITH NO ADVANCING` ใน `3000-GET-CHOICE` ทำให้ prompt กับค่าที่ผู้ใช้พิมพ์
  ปรากฏบนบรรทัดเดียวกันเมื่อรันแบบโต้ตอบจริงบน terminal (แต่ในการทดสอบผ่าน redirect จะไม่เห็นค่าที่พิมพ์
  ตามที่อธิบายไว้ในคำนำของ Part นี้)

### แบบฝึกหัดที่ 142.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ `PERFORM WITH TEST AFTER` แทน `PERFORM WITH TEST BEFORE`
(หรือค่า default) จึงเหมาะสมกับโปรแกรมเมนูมากกว่า

**เฉลยแนวทาง**: `PERFORM WITH TEST AFTER` รับประกันว่าเนื้อลูป (แสดงเมนู, รับตัวเลือก) จะทำงาน**อย่างน้อย
หนึ่งครั้งเสมอ**ก่อนตรวจสอบเงื่อนไขหยุด ตรงกับพฤติกรรมตามธรรมชาติของโปรแกรมเมนู เพราะไม่มีทางที่จะรู้ว่า
ผู้ใช้ "ต้องการออก" ได้ก่อนที่จะแสดงเมนูให้เขาเห็นและให้เขาเลือกก่อน หากใช้ `TEST BEFORE` กับค่าเริ่มต้น
`WS-MENU-CHOICE VALUE 0` ก็ยังคงทำงานถูกต้องในกรณีนี้เพราะ 0 ไม่เท่ากับ 5 อยู่แล้ว แต่การเลือกใช้
`TEST AFTER` สื่อความหมายเชิงตรรกะของโปรแกรมได้ตรงกว่า และปลอดภัยกว่าในกรณีที่ค่าเริ่มต้นอาจถูกเปลี่ยน
โดยไม่ตั้งใจในอนาคต

---

## ขั้นตอนที่ 143: เพิ่มการคำนวณด้วย ADD/SUBTRACT/MULTIPLY/DIVIDE

### แนวคิด

ขั้นตอนนี้เราเติม Paragraph `4000-GET-NUMBERS`, `5000-CALCULATE`, และ `6000-SHOW-RESULT` ให้ทำงานจริง
โดยใช้คำสั่งเลขคณิตทั้ง 4 คำสั่งที่เรียนมาจาก Part 009: `ADD ... GIVING`, `SUBTRACT ... FROM ... GIVING`,
`MULTIPLY ... BY ... GIVING`, และ `DIVIDE ... BY ... GIVING ROUNDED` เรา**จงใจยังไม่ป้องกันการหารด้วย
ศูนย์**ในขั้นตอนนี้ เพื่อสาธิตให้เห็นปัญหาจริงก่อน แล้วจึงแก้ไขในขั้นตอนที่ 144

ตัวเลขนำเข้าใช้ `PIC S9(7)V99` (มีเครื่องหมาย `S`, ทศนิยม 2 ตำแหน่งด้วย `V99`) ตามที่ออกแบบไว้ใน
ขั้นตอนที่ 141 ทำให้รองรับทั้งเลขลบและเลขทศนิยมได้ตามความต้องการข้อ 2

### ตัวอย่างโค้ด (เวอร์ชันขั้นตอนที่ 143 — ยังไม่ป้องกันหารศูนย์)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP143.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MENU-CHOICE      PIC 9(1)      VALUE 0.
       01  WS-NUM-1            PIC S9(7)V99  VALUE 0.
       01  WS-NUM-2            PIC S9(7)V99  VALUE 0.
       01  WS-CALC-RESULT      PIC S9(9)V99  VALUE 0.

       PROCEDURE DIVISION.
       0000-MAIN-PROCESS.
           PERFORM 1000-SHOW-WELCOME
           PERFORM WITH TEST AFTER
                   UNTIL WS-MENU-CHOICE = 5
               PERFORM 2000-SHOW-MENU
               PERFORM 3000-GET-CHOICE
               EVALUATE WS-MENU-CHOICE
                   WHEN 1 THRU 4
                       PERFORM 4000-GET-NUMBERS
                       PERFORM 5000-CALCULATE
                       PERFORM 6000-SHOW-RESULT
                   WHEN 5
                       DISPLAY "Exiting calculator. Goodbye!"
                   WHEN OTHER
                       DISPLAY "Invalid choice, please select 1-5."
               END-EVALUATE
           END-PERFORM
           STOP RUN.

       1000-SHOW-WELCOME.
           DISPLAY "=================================".
           DISPLAY "   SIMPLE COBOL CALCULATOR".
           DISPLAY "=================================".

       2000-SHOW-MENU.
           DISPLAY " ".
           DISPLAY "1. Add        2. Subtract".
           DISPLAY "3. Multiply   4. Divide".
           DISPLAY "5. Exit".

       3000-GET-CHOICE.
           DISPLAY "Enter your choice (1-5): " WITH NO ADVANCING.
           ACCEPT WS-MENU-CHOICE.

       4000-GET-NUMBERS.
           DISPLAY "Enter first number : " WITH NO ADVANCING.
           ACCEPT WS-NUM-1.
           DISPLAY "Enter second number: " WITH NO ADVANCING.
           ACCEPT WS-NUM-2.

       5000-CALCULATE.
           EVALUATE WS-MENU-CHOICE
               WHEN 1
                   ADD WS-NUM-1 TO WS-NUM-2 GIVING WS-CALC-RESULT
               WHEN 2
                   SUBTRACT WS-NUM-2 FROM WS-NUM-1
                       GIVING WS-CALC-RESULT
               WHEN 3
                   MULTIPLY WS-NUM-1 BY WS-NUM-2
                       GIVING WS-CALC-RESULT
               WHEN 4
      *> WARNING: no zero check yet -- see the pitfall this causes
      *> below, and the fix that comes in the next step.
                   DIVIDE WS-NUM-1 BY WS-NUM-2
                       GIVING WS-CALC-RESULT ROUNDED
           END-EVALUATE.

       6000-SHOW-RESULT.
           DISPLAY "Result = " WS-CALC-RESULT.
```

### ผลลัพธ์จริงที่ได้ (ทดสอบบวก/ลบ/คูณ/หารปกติ แล้วหารด้วยศูนย์)

ป้อนค่าตามลำดับ: เลือก 1 (บวก) 15, 7 → เลือก 2 (ลบ) 15, 7 → เลือก 3 (คูณ) 15, 7 →
เลือก 4 (หาร) 15, 7 → เลือก 4 (หาร) 15, 0 → เลือก 5 (ออก)

```
Result = +000000022.00
Result = +000000008.00
Result = +000000105.00
Result = +000000002.14
Result = +000000002.14
Exiting calculator. Goodbye!
```

### อธิบายโค้ดทีละส่วน

- 15 + 7 = 22.00, 15 - 7 = 8.00, 15 × 7 = 105.00, 15 ÷ 7 = 2.142857... ปัดเศษด้วย `ROUNDED`
  เป็น 2.14 — ผลลัพธ์ทั้งสี่ค่าถูกต้องตามที่คาดหวังทุกประการ
- **จุดที่ต้องสังเกตอย่างละเอียด**: การหารครั้งที่ 2 คือ `15 ÷ 0` ซึ่งควรจะ error หรือแสดงผลที่ชัดเจนว่า
  ผิดพลาด แต่ **ผลลัพธ์ที่แสดงกลับเป็น `+000000002.14` เหมือนกับผลลัพธ์ของการหารครั้งก่อนหน้าทุกประการ!**
  นี่ไม่ใช่ความบังเอิญ — GnuCOBOL ปฏิบัติตามกฎที่เรียนมาใน Part 009 ขั้นตอนที่ 87 อย่างเคร่งครัด: **เมื่อ
  เกิด SIZE ERROR (รวมถึงการหารด้วยศูนย์) ฟิลด์ปลายทางจะไม่ถูกแก้ไขเลย** ดังนั้น `WS-CALC-RESULT`
  จึงยังคงค่าเดิมจากการคำนวณครั้งก่อนหน้าไว้ (2.14 จากการหาร 15÷7 ที่ทำไปก่อนหน้านั้น) โดยไม่มีข้อความ
  แจ้งเตือนใด ๆ เลย

### กับดักสำคัญที่สุดของขั้นตอนนี้: ผลลัพธ์ "ค้าง" ที่ดูสมเหตุสมผลแต่ผิดสนิท

นี่คือกับดักที่ **อันตรายยิ่งกว่าโปรแกรม crash เสียอีก** เพราะถ้าโปรแกรม crash ผู้ใช้จะรู้ทันทีว่ามีบางอย่าง
ผิดพลาด แต่การแสดงผลลัพธ์เก่าที่ดู "สมเหตุสมผล" (เป็นตัวเลขปกติ ไม่ใช่ 0 หรือค่าประหลาด) ทำให้ผู้ใช้
**เข้าใจผิดว่าผลลัพธ์นี้คือคำตอบของการหารด้วยศูนย์ที่เพิ่งทำไป** ซึ่งเป็นข้อมูลเท็จที่อาจถูกนำไปใช้ต่อ
ในระบบธุรกิจจริงโดยไม่มีใครสังเกตเห็นความผิดปกติเลย นี่คือเหตุผลที่ขั้นตอนที่ 144 จะแก้ปัญหานี้อย่างจริงจัง

### ข้อควรระวัง

- **ห้ามปล่อยให้โค้ดคำนวณที่อาจหารด้วยศูนย์ไม่มีการป้องกันเด็ดขาดในโปรแกรมจริง** ตัวอย่างนี้จงใจแสดง
  ปัญหาให้เห็นเพื่อการเรียนรู้เท่านั้น ไม่ควรเป็นแบบแผนที่นำไปใช้งานจริง
- ทดสอบโปรแกรมด้วยกรณี "ค่าที่ผิดปกติ" (เช่น หารด้วยศูนย์, ค่าติดลบ, ค่าที่ใหญ่มาก) เสมอ ไม่ใช่แค่กรณี
  ปกติทั่วไป เพราะบั๊กที่อันตรายที่สุดมักซ่อนอยู่ในเส้นทางที่ไม่ค่อยถูกทดสอบ

### แบบฝึกหัดที่ 143.1

**โจทย์**: จงอธิบายว่าทำไมผลลัพธ์ของการหารด้วยศูนย์ในตัวอย่างนี้จึงแสดงเป็น `2.14` (ค่าจากการหารครั้งก่อน)
แทนที่จะเป็น `0.00`

**เฉลย**: เพราะ COBOL ปฏิบัติตามกฎ SIZE ERROR ที่ว่า "เมื่อการคำนวณล้มเหลว (รวมถึงหารด้วยศูนย์) ฟิลด์
ปลายทางจะไม่ถูกเขียนทับด้วยค่าใหม่เลย" ค่าที่แสดงจึงเป็นค่า**เดิม**ที่ `WS-CALC-RESULT` มีอยู่ก่อนหน้า
คำสั่ง `DIVIDE` ที่ล้มเหลวนั้น ซึ่งในกรณีนี้คือผลลัพธ์ 2.14 จากการหาร 15÷7 ที่ทำสำเร็จไปก่อนหน้า ไม่ใช่
ค่า 0.00 อย่างที่หลายคนอาจคาดเดา (พฤติกรรมนี้อาจต่างกันได้ในบางคอมไพเลอร์ที่ไม่รับประกัน "unchanged"
เมื่อไม่มี `ON SIZE ERROR` กำกับไว้ชัดเจน จึงยิ่งเป็นเหตุผลให้ต้องป้องกันด้วยตนเองเสมอ ไม่พึ่งพาพฤติกรรม
เฉพาะของคอมไพเลอร์ตัวใดตัวหนึ่ง)

---

## ขั้นตอนที่ 144: ป้องกันการหารด้วยศูนย์ และเพิ่มลูป "คำนวณอีกครั้งหรือไม่"

### แนวคิด

ขั้นตอนนี้แก้ไขปัญหาจากขั้นตอนที่ 143 ด้วยสองวิธีที่เทียบเคียงกันได้ (เลือกใช้วิธีใดวิธีหนึ่งก็เพียงพอ):

1. **ตรวจสอบด้วย `IF` ก่อนหาร** — เชิงรุก (proactive) อ่านง่ายตรงไปตรงมา (ใช้ในเวอร์ชันหลักของโปรเจกต์นี้)
2. **ใช้ `ON SIZE ERROR`** — กลไกในตัวคำสั่งเลขคณิตเอง (ทบทวนจาก Part 009 ขั้นตอนที่ 87)

นอกจากนี้ เราจะเพิ่ม Paragraph `7000-ASK-AGAIN` ที่ถามผู้ใช้ว่าต้องการคำนวณต่อหรือไม่หลังจากคำนวณเสร็จ
แต่ละครั้ง — นี่คือการเติมเต็มความต้องการข้อ 5 จากขั้นตอนที่ 141 ทำให้เครื่องคิดเลขสมบูรณ์เต็มรูปแบบ

### วิธีที่ 1: ป้องกันด้วย IF (ใช้ในโปรเจกต์นี้)

```cobol
       5000-CALCULATE.
           MOVE "N" TO WS-DIVIDE-BY-ZERO-FLAG
           EVALUATE WS-MENU-CHOICE
               WHEN 1
                   ADD WS-NUM-1 TO WS-NUM-2 GIVING WS-CALC-RESULT
               WHEN 2
                   SUBTRACT WS-NUM-2 FROM WS-NUM-1
                       GIVING WS-CALC-RESULT
               WHEN 3
                   MULTIPLY WS-NUM-1 BY WS-NUM-2
                       GIVING WS-CALC-RESULT
               WHEN 4
                   IF WS-NUM-2 = 0
                       SET WS-DIVIDE-BY-ZERO TO TRUE
                       DISPLAY "ERROR: Cannot divide by zero!"
                   ELSE
                       DIVIDE WS-NUM-1 BY WS-NUM-2
                           GIVING WS-CALC-RESULT ROUNDED
                   END-IF
           END-EVALUATE.

       6000-SHOW-RESULT.
           IF NOT WS-DIVIDE-BY-ZERO
               DISPLAY "Result = " WS-CALC-RESULT
           END-IF.
```

โดยต้องประกาศตัวแปรและ 88-level เพิ่มเติมใน WORKING-STORAGE:

```cobol
       01  WS-DIVIDE-BY-ZERO-FLAG PIC X(1)   VALUE "N".
           88  WS-DIVIDE-BY-ZERO         VALUE "Y".
```

### วิธีที่ 2: ป้องกันด้วย ON SIZE ERROR (ทางเลือกเทียบเคียง)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SIZEERRALT.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NUM-1        PIC S9(7)V99 VALUE 0.
       01  WS-NUM-2        PIC S9(7)V99 VALUE 0.
       01  WS-CALC-RESULT  PIC S9(9)V99 VALUE 0.

       PROCEDURE DIVISION.
           DISPLAY "Enter first number : " WITH NO ADVANCING.
           ACCEPT WS-NUM-1.
           DISPLAY "Enter second number: " WITH NO ADVANCING.
           ACCEPT WS-NUM-2.
           DIVIDE WS-NUM-1 BY WS-NUM-2
               GIVING WS-CALC-RESULT ROUNDED
               ON SIZE ERROR
                   DISPLAY "ERROR: Cannot divide by zero!"
           END-DIVIDE.
           DISPLAY "Result = " WS-CALC-RESULT.
           STOP RUN.
```

**ผลลัพธ์จริงที่ได้ (ทดสอบทั้งกรณีปกติและหารศูนย์)**:

```
Enter second number: ERROR: Cannot divide by zero!
Result = +000000000.00
---
Enter second number: Result = +000000002.50
```

สังเกตว่าวิธีที่ 2 ให้ผลลัพธ์ `+000000000.00` เมื่อหารศูนย์ เพราะ `WS-CALC-RESULT` ยังไม่เคยถูกกำหนด
ค่ามาก่อนหน้าเลย (เป็นการรันแยกโปรแกรมใหม่ ค่าเริ่มต้นจาก `VALUE 0`) ต่างจากตัวอย่างในขั้นตอนที่ 143
ที่ค่าค้างมาจากการคำนวณครั้งก่อน — ทั้งสองกรณีคือพฤติกรรมเดียวกัน (ฟิลด์ปลายทางไม่ถูกแก้ไขเมื่อเกิด
SIZE ERROR) เพียงแค่ค่า "เดิม" ที่ค้างอยู่ต่างกันเท่านั้น

โปรเจกต์นี้เลือกใช้ **วิธีที่ 1 (IF)** เป็นหลัก เพราะทำให้เห็นตรรกะชัดเจนกว่าสำหรับผู้เริ่มต้น และควบคุม
การแสดงผล (ไม่แสดงบรรทัด "Result = ..." เลยเมื่อเกิดข้อผิดพลาด) ได้ง่ายกว่า

### เพิ่มลูป "คำนวณอีกครั้งหรือไม่"

```cobol
       7000-ASK-AGAIN.
           DISPLAY "Perform another calculation? (Y/N): "
               WITH NO ADVANCING.
           ACCEPT WS-AGAIN-ANSWER.
           IF NOT WS-USER-WANTS-MORE
               MOVE 5 TO WS-MENU-CHOICE
           END-IF.
```

โดยประกาศ:

```cobol
       01  WS-AGAIN-ANSWER     PIC X(1)      VALUE "Y".
           88  WS-USER-WANTS-MORE        VALUE "Y" "y".
```

เทคนิคสำคัญตรงนี้คือ **การใช้ `WS-MENU-CHOICE` ตัวเดียวกันเป็นทั้งตัวเลือกเมนูและกลไกควบคุมการออกจากลูป**
เมื่อผู้ใช้ตอบ "ไม่ต้องการคำนวณต่อ" เราเพียงแค่ `MOVE 5 TO WS-MENU-CHOICE` (ค่าเดียวกับตัวเลือก "Exit"
ในเมนู) ทำให้เงื่อนไข `UNTIL WS-MENU-CHOICE = 5` ที่ `0000-MAIN-PROCESS` เป็นจริงในรอบถัดไปโดยอัตโนมัติ
โดยไม่ต้องสร้าง flag ตัวใหม่แยกต่างหาก — นี่คือการใช้ตัวแปรที่มีอยู่แล้วอย่างมีประสิทธิภาพ

### ผลลัพธ์จริงที่ได้ (โปรแกรมฉบับสมบูรณ์ของขั้นตอนนี้)

ป้อนค่า: เลือก 4 (หาร) 10, 2 → ตอบ Y → เลือก 4 (หาร) 10, 0 → ตอบ Y → เลือก 9 (ไม่ถูกต้อง) →
เลือก 5 (ออก)

```
=================================
   SIMPLE COBOL CALCULATOR
=================================
 
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Enter first number : Enter second number: Result = +000000005.00
Perform another calculation? (Y/N):  
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Enter first number : Enter second number: ERROR: Cannot divide by zero!
Perform another calculation? (Y/N):  
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Invalid choice, please select 1-5.
 
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Exiting calculator. Goodbye!
```

ตอนนี้เมื่อหารด้วยศูนย์ โปรแกรมแสดง **เฉพาะข้อความแจ้งข้อผิดพลาด ไม่มีบรรทัด "Result = ..." ที่ทำให้
เข้าใจผิดปรากฏเลย** ปัญหาจากขั้นตอนที่ 143 ได้รับการแก้ไขอย่างสมบูรณ์

### ข้อควรระวัง

- อย่าลืม `MOVE "N" TO WS-DIVIDE-BY-ZERO-FLAG` ที่ต้นของ `5000-CALCULATE` ทุกครั้ง — นี่คือการ
  "รีเซ็ต" flag ก่อนการคำนวณรอบใหม่ หากลืมขั้นตอนนี้ เมื่อเกิดการหารด้วยศูนย์ครั้งหนึ่งแล้ว flag จะค้าง
  เป็น "Y" ตลอดไป ทำให้การคำนวณที่ถูกต้องในรอบถัดไปถูกซ่อนผลลัพธ์ไปด้วยอย่างผิดพลาด (ทดลองลบบรรทัดนี้
  แล้วสังเกตผลลัพธ์ที่ผิดเพี้ยนได้ด้วยตนเอง)
- `IF NOT WS-USER-WANTS-MORE` จะเป็นจริงเมื่อผู้ใช้ตอบอะไรก็ตามที่ไม่ใช่ "Y" หรือ "y" (รวมถึง "N", "n",
  หรือแม้แต่ค่าว่างเปล่า/พิมพ์ผิด) นี่เป็นการออกแบบที่ปลอดภัย (fail-safe) คือ **เมื่อไม่แน่ใจ ให้ออกจาก
  โปรแกรม** แทนที่จะวนลูปต่อไปเรื่อย ๆ โดยไม่มีทางออกหากผู้ใช้พิมพ์ค่าที่ไม่คาดคิด

### แบบฝึกหัดที่ 144.1

**โจทย์**: จงอธิบายว่าทำไมการใช้ `MOVE 5 TO WS-MENU-CHOICE` ใน `7000-ASK-AGAIN` แทนการสร้างตัวแปร
ควบคุมลูปแยกต่างหาก (เช่น `WS-KEEP-RUNNING`) จึงเป็นทางเลือกที่สมเหตุสมผลในโปรแกรมขนาดเล็กนี้
และข้อเสียที่อาจเกิดขึ้นในโปรแกรมที่ใหญ่ขึ้นคืออะไร

**เฉลยแนวทาง**: ในโปรแกรมขนาดเล็กแบบนี้ การใช้ตัวแปรเดิม (`WS-MENU-CHOICE`) ที่มีความหมายตรงกับค่า
"Exit" อยู่แล้วช่วยลดจำนวนตัวแปรที่ต้องดูแล และตรรกะยังคงอ่านเข้าใจง่าย แต่ข้อเสียคือ **ตัวแปรเดียวถูกใช้
สื่อความหมายสองอย่างพร้อมกัน** ("ตัวเลือกเมนูที่ผู้ใช้เลือก" และ "สัญญาณบอกให้ออกจากลูป") ซึ่งในโปรแกรม
ที่ใหญ่ขึ้นหรือมีคนหลายคนดูแลโค้ดร่วมกัน อาจทำให้เกิดความสับสนได้ง่ายกว่าการมีตัวแปร `WS-KEEP-RUNNING`
แยกต่างหากที่สื่อความหมายเดียวชัดเจน — เป็นตัวอย่างของการแลกเปลี่ยน (trade-off) ระหว่างความกระชับกับ
ความชัดเจนที่โปรแกรมเมอร์ต้องตัดสินใจตามขนาดและบริบทของโปรแกรมจริง

---

## ขั้นตอนที่ 145: เครื่องคิดเลขฉบับสมบูรณ์ และการทดสอบครบทุกเส้นทาง

### แนวคิด

ขั้นตอนนี้รวบรวมทุกส่วนจากขั้นตอนที่ 142-144 เป็น **โปรแกรมเครื่องคิดเลขฉบับสมบูรณ์** พร้อมทดสอบครบทุก
เส้นทางการทำงาน (test coverage): บวก, ลบ, คูณ, หารปกติ, หารศูนย์, ตัวเลือกไม่ถูกต้อง, เลขติดลบ, เลขทศนิยม,
และการออกจากโปรแกรม

### ซอร์สโค้ดฉบับสมบูรณ์: `calculator.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CALCULATOR.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MENU-CHOICE      PIC 9(1)      VALUE 0.
       01  WS-NUM-1            PIC S9(7)V99  VALUE 0.
       01  WS-NUM-2            PIC S9(7)V99  VALUE 0.
       01  WS-CALC-RESULT      PIC S9(9)V99  VALUE 0.
       01  WS-AGAIN-ANSWER     PIC X(1)      VALUE "Y".
           88  WS-USER-WANTS-MORE        VALUE "Y" "y".
       01  WS-DIVIDE-BY-ZERO-FLAG PIC X(1)   VALUE "N".
           88  WS-DIVIDE-BY-ZERO         VALUE "Y".

       PROCEDURE DIVISION.
       0000-MAIN-PROCESS.
           PERFORM 1000-SHOW-WELCOME
           PERFORM WITH TEST AFTER
                   UNTIL WS-MENU-CHOICE = 5
               PERFORM 2000-SHOW-MENU
               PERFORM 3000-GET-CHOICE
               EVALUATE WS-MENU-CHOICE
                   WHEN 1 THRU 4
                       PERFORM 4000-GET-NUMBERS
                       PERFORM 5000-CALCULATE
                       PERFORM 6000-SHOW-RESULT
                       PERFORM 7000-ASK-AGAIN
                   WHEN 5
                       DISPLAY "Exiting calculator. Goodbye!"
                   WHEN OTHER
                       DISPLAY "Invalid choice, please select 1-5."
               END-EVALUATE
           END-PERFORM
           STOP RUN.

       1000-SHOW-WELCOME.
           DISPLAY "=================================".
           DISPLAY "   SIMPLE COBOL CALCULATOR".
           DISPLAY "=================================".

       2000-SHOW-MENU.
           DISPLAY " ".
           DISPLAY "1. Add        2. Subtract".
           DISPLAY "3. Multiply   4. Divide".
           DISPLAY "5. Exit".

       3000-GET-CHOICE.
           DISPLAY "Enter your choice (1-5): " WITH NO ADVANCING.
           ACCEPT WS-MENU-CHOICE.

       4000-GET-NUMBERS.
           DISPLAY "Enter first number : " WITH NO ADVANCING.
           ACCEPT WS-NUM-1.
           DISPLAY "Enter second number: " WITH NO ADVANCING.
           ACCEPT WS-NUM-2.

       5000-CALCULATE.
           MOVE "N" TO WS-DIVIDE-BY-ZERO-FLAG
           EVALUATE WS-MENU-CHOICE
               WHEN 1
                   ADD WS-NUM-1 TO WS-NUM-2 GIVING WS-CALC-RESULT
               WHEN 2
                   SUBTRACT WS-NUM-2 FROM WS-NUM-1
                       GIVING WS-CALC-RESULT
               WHEN 3
                   MULTIPLY WS-NUM-1 BY WS-NUM-2
                       GIVING WS-CALC-RESULT
               WHEN 4
                   IF WS-NUM-2 = 0
                       SET WS-DIVIDE-BY-ZERO TO TRUE
                       DISPLAY "ERROR: Cannot divide by zero!"
                   ELSE
                       DIVIDE WS-NUM-1 BY WS-NUM-2
                           GIVING WS-CALC-RESULT ROUNDED
                   END-IF
           END-EVALUATE.

       6000-SHOW-RESULT.
           IF NOT WS-DIVIDE-BY-ZERO
               DISPLAY "Result = " WS-CALC-RESULT
           END-IF.

       7000-ASK-AGAIN.
           DISPLAY "Perform another calculation? (Y/N): "
               WITH NO ADVANCING.
           ACCEPT WS-AGAIN-ANSWER.
           IF NOT WS-USER-WANTS-MORE
               MOVE 5 TO WS-MENU-CHOICE
           END-IF.
```

### วิธีทดสอบครบทุกเส้นทาง

คอมไพล์โปรแกรมด้วย:

```bash
cobc -x -o calculator calculator.cob
```

สร้างไฟล์ `calc_input.txt` ที่จำลองการพิมพ์ของผู้ใช้ทีละบรรทัด ครอบคลุมทุกเมนู รวมถึงกรณีหารศูนย์และ
ตัวเลือกไม่ถูกต้อง:

```
1
10
5
Y
2
10
3
Y
3
4
5
Y
4
10
2
Y
4
10
0
Y
9
5
```

รันด้วย:

```bash
./calculator < calc_input.txt
```

### ผลลัพธ์จริงที่ได้ (การทดสอบครบทุกเส้นทาง)

```
=================================
   SIMPLE COBOL CALCULATOR
=================================
 
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Enter first number : Enter second number: Result = +000000015.00
Perform another calculation? (Y/N):  
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Enter first number : Enter second number: Result = +000000007.00
Perform another calculation? (Y/N):  
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Enter first number : Enter second number: Result = +000000020.00
Perform another calculation? (Y/N):  
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Enter first number : Enter second number: Result = +000000005.00
Perform another calculation? (Y/N):  
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Enter first number : Enter second number: ERROR: Cannot divide by zero!
Perform another calculation? (Y/N):  
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Invalid choice, please select 1-5.
 
1. Add        2. Subtract
3. Multiply   4. Divide
5. Exit
Enter your choice (1-5): Exiting calculator. Goodbye!
```

**ตรวจสอบผลลัพธ์ทีละรายการ**: 10+5=15.00 ✓, 10-3=7.00 ✓, 4×5=20.00 ✓, 10÷2=5.00 ✓,
10÷0 → แสดง error อย่างถูกต้อง ไม่มีบรรทัด Result ที่ผิดพลาด ✓, ตัวเลือก 9 → แจ้งเตือนถูกต้อง ✓,
ตัวเลือก 5 → ออกจากโปรแกรมถูกต้อง ✓ **ครบทุกเส้นทางตามที่ออกแบบไว้ในขั้นตอนที่ 141**

### ทดสอบเพิ่มเติม: เลขติดลบและเลขทศนิยม

ป้อนค่า: เลือก 1 (บวก) -15.50, 20.25 → ตอบ Y → เลือก 2 (ลบ) 5.5, 10 → ตอบ N

```
Result = +000000004.75
Result = -000000004.50
```

ยืนยันว่า -15.50 + 20.25 = 4.75 และ 5.5 - 10 = -4.50 ถูกต้องทั้งคู่ รวมถึงเครื่องหมาย `+`/`-`
แสดงผลถูกต้องตามค่าจริง (ทบทวนจาก Part 009 เรื่อง `PIC S9...`) — ยืนยันว่าความต้องการข้อ 2 (รองรับ
เลขลบและทศนิยม) สำเร็จสมบูรณ์

### อธิบายภาพรวมสถาปัตยกรรม

โปรแกรมนี้ใช้รูปแบบ **Numbered Paragraph Convention** เต็มรูปแบบตามที่เรียนใน Part 014 ขั้นตอนที่ 136:
`0000` ทำหน้าที่เป็นตัวควบคุมหลัก (main control) ที่อ่านแล้วเห็นภาพรวมได้ทันที ส่วน `1000`-`7000` แต่ละตัว
รับผิดชอบงานเฉพาะทางเพียงอย่างเดียว (single responsibility) — `2000` แสดงเมนูเท่านั้น ไม่ยุ่งกับการรับค่า,
`3000` รับค่าเท่านั้น ไม่ยุ่งกับการคำนวณ เป็นต้น การแบ่งความรับผิดชอบแบบนี้ทำให้เมื่อต้องแก้ไขส่วนใดส่วนหนึ่ง
(เช่น เปลี่ยนข้อความในเมนู) เราจะรู้ทันทีว่าต้องแก้ที่ Paragraph ไหนโดยไม่กระทบส่วนอื่น

### ข้อควรระวัง

- การทดสอบด้วยไฟล์ input ที่ครอบคลุมทุกเส้นทาง (test coverage) เป็นทักษะสำคัญที่ควรฝึกฝนตั้งแต่ต้น
  แม้ในโปรแกรมง่าย ๆ — เมื่อโปรแกรมมีความซับซ้อนมากขึ้นในเฟสถัดไปของหลักสูตร การมีชุดข้อมูลทดสอบที่
  ครอบคลุมจะช่วยจับบั๊กได้เร็วกว่าการทดสอบแบบสุ่มมาก
- อย่าลืมว่าไฟล์ `calc_input.txt` ต้องมีจำนวนบรรทัดข้อมูล**ตรงกับจำนวนครั้งที่โปรแกรมเรียก `ACCEPT`
  พอดี** หากมีค่าน้อยเกินไป (เช่น ลืมใส่ค่า "Y/N" หลังบางเมนู) โปรแกรมจะพยายามอ่านค่าที่ไม่มีอยู่จริง
  จาก stdin ซึ่งอาจทำให้ได้ผลลัพธ์ที่ไม่คาดคิด (ค่าว่างเปล่าหรือค่าตกค้าง) ควรนับจำนวน `ACCEPT` ที่จะ
  ถูกเรียกให้ตรงกับจำนวนบรรทัดในไฟล์ทดสอบเสมอ

### แบบฝึกหัดที่ 145.1 — ต่อยอดโปรเจกต์: เพิ่มตัวดำเนินการมอดูโล (Modulus)

**โจทย์**: จงต่อยอดเครื่องคิดเลขให้มีตัวเลือกที่ 6 คือ "Modulus" (หาเศษจากการหาร) โดยใช้
`FUNCTION MOD` ที่เรียนมาจาก Part 009 ขั้นตอนที่ 89 พร้อมป้องกันการหารด้วยศูนย์เช่นเดียวกับตัวเลือก
Divide

**เฉลย**: แก้ไข 3 จุดในโปรแกรม `calculator.cob` ดังนี้ — (1) เมนู: `DISPLAY "5. Exit        6.
Modulus".` (2) `EVALUATE` ใน `0000-MAIN-PROCESS`: เปลี่ยน `WHEN 1 THRU 4` ให้มี `WHEN 6` ต่อท้าย
เพื่อให้ทำงานชุดเดียวกัน (3) เพิ่ม `WHEN 6` ใหม่ใน `5000-CALCULATE`:

```cobol
               WHEN 1 THRU 4
               WHEN 6
                   PERFORM 4000-GET-NUMBERS
                   PERFORM 5000-CALCULATE
                   PERFORM 6000-SHOW-RESULT
                   PERFORM 7000-ASK-AGAIN
```

```cobol
               WHEN 6
                   IF WS-NUM-2 = 0
                       SET WS-DIVIDE-BY-ZERO TO TRUE
                       DISPLAY "ERROR: Cannot divide by zero!"
                   ELSE
                       COMPUTE WS-CALC-RESULT =
                           FUNCTION MOD(WS-NUM-1, WS-NUM-2)
                   END-IF
```

**ทดสอบจริง (คอมไพล์และรันแล้ว)**: ป้อน `6`, `17`, `5` (17 mod 5) แล้ว `6`, `10`, `0` (มอดูโลศูนย์):

```
Result = +000000002.00
ERROR: Cannot divide by zero!
```

17 mod 5 = 2 ถูกต้อง (17 = 3×5 + 2) และการป้องกันหารศูนย์ยังทำงานถูกต้องกับตัวดำเนินการใหม่นี้ด้วย
เพราะใช้ `WS-NUM-2 = 0` เงื่อนไขเดียวกับ Divide ในการตรวจสอบ

---

## ขั้นตอนที่ 146: เริ่มสร้างระบบคำนวณเกรด — ออกแบบตารางคะแนนด้วย OCCURS

### แนวคิด

เปลี่ยนมาสร้างโปรแกรมที่สองของโปรเจกต์เฟส 1 ขั้นตอนนี้เราจะเติมเนื้อหาให้ `1000-SHOW-WELCOME` และ
`2000-GET-STUDENT-COUNT` พร้อมประกาศตาราง `WS-SCORE-TABLE` ด้วย `OCCURS 20 TIMES` ตามที่ออกแบบไว้
ในขั้นตอนที่ 141

การกำหนดขนาดตารางไว้ที่ 20 (แทนที่จะปล่อยให้ไม่จำกัด) เป็นข้อจำกัดโดยธรรมชาติของ `OCCURS` แบบพื้นฐาน
ที่เรียนในระดับนี้ — ต้องกำหนดขนาดสูงสุดตายตัวตั้งแต่ตอนคอมไพล์เสมอ (`OCCURS` แบบที่ขนาดปรับเปลี่ยนได้
เช่น `OCCURS ... DEPENDING ON` จะสอนใน Part 016 ขั้นตอนถัดไป)

### ตัวอย่างโค้ด (เวอร์ชันขั้นตอนที่ 146)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP146.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STUDENT-COUNT    PIC 9(2)      VALUE 0.
       01  WS-SCORE-TABLE.
           05  WS-STUDENT-SCORE PIC 9(3) OCCURS 20 TIMES.

       PROCEDURE DIVISION.
       0000-MAIN-PROCESS.
           PERFORM 1000-SHOW-WELCOME
           PERFORM 2000-GET-STUDENT-COUNT
           DISPLAY "Table ready for " WS-STUDENT-COUNT " students."
           DISPLAY "(Score collection will be added in the next step.)"
           STOP RUN.

       1000-SHOW-WELCOME.
           DISPLAY "=================================".
           DISPLAY "   STUDENT GRADE CALCULATOR".
           DISPLAY "=================================".

       2000-GET-STUDENT-COUNT.
           DISPLAY "Enter number of students (1-20): "
               WITH NO ADVANCING.
           ACCEPT WS-STUDENT-COUNT.
```

คอมไพล์และรันด้วย:

```bash
cobc -x -o step146 step146.cob
printf -- "5\n" | ./step146
```

### ผลลัพธ์จริงที่ได้

```
=================================
   STUDENT GRADE CALCULATOR
=================================
Enter number of students (1-20): Table ready for 05 students.
(Score collection will be added in the next step.)
```

### อธิบายโค้ดทีละส่วน

- `05 WS-STUDENT-SCORE PIC 9(3) OCCURS 20 TIMES.` ประกาศตารางที่มี 20 ช่อง แต่ละช่องเป็นตัวเลข
  3 หลัก (0-999 แต่จะใช้จริงแค่ 0-100 ตามกฎธุรกิจ) — ขนาดหน่วยความจำรวมของตารางนี้คือ 3×20 = 60 ไบต์
  แม้ผู้ใช้จะกรอกแค่ 5 คนก็ตาม เพราะ COBOL จองพื้นที่ตาม `OCCURS` ไว้เต็มจำนวนเสมอตั้งแต่คอมไพล์
- `WS-STUDENT-COUNT` เก็บว่าผู้ใช้ต้องการใช้ "กี่ช่อง" จาก 20 ช่องที่จองไว้ ซึ่งจะเป็นตัวกำหนดขอบเขต
  ของลูปในขั้นตอนถัดไปทั้งหมด (การรับคะแนน, การคำนวณสถิติ, การพิมพ์รายงาน)

### ข้อควรระวัง

- ขั้นตอนนี้**ยังไม่มีการตรวจสอบ**ว่าค่าที่ผู้ใช้ป้อนอยู่ในช่วง 1-20 หรือไม่ หากผู้ใช้ป้อน `25`
  ตัวแปร `WS-STUDENT-COUNT` (`PIC 9(2)`) จะรับค่าได้สูงสุดแค่ 99 อยู่แล้ว แต่ปัญหาจริงคือถ้าโค้ดในขั้นตอน
  ถัดไปใช้ `WS-STUDENT-COUNT` เป็นขอบเขตลูปเข้าถึงตาราง `OCCURS 20 TIMES` โดยตรง การป้อนค่ามากกว่า 20
  จะทำให้พยายามเข้าถึงตำแหน่งที่ 21-25 ซึ่ง**เกินขอบเขตที่ประกาศไว้** (ปัญหาเดียวกับที่เตือนไว้ใน
  Part 013 ขั้นตอนที่ 126) — จะแก้ไขปัญหานี้อย่างจริงจังในขั้นตอนที่ 147
- อย่าลืมว่า index ของตาราง COBOL เริ่มที่ 1 เสมอ ไม่ใช่ 0 (ทบทวนจาก Part 013 ขั้นตอนที่ 126)

### แบบฝึกหัดที่ 146.1

**โจทย์**: จงคำนวณว่าถ้าต้องการรองรับนักเรียนสูงสุด 50 คนแทนที่จะเป็น 20 คน ต้องแก้ไขโค้ดตรงจุดใดบ้าง
และตาราง `WS-SCORE-TABLE` จะใช้หน่วยความจำรวมกี่ไบต์

**เฉลย**: แก้ `OCCURS 20 TIMES` เป็น `OCCURS 50 TIMES` เพียงจุดเดียวในการประกาศตาราง (ไม่ต้องแก้ไข
Paragraph อื่นเลย เพราะทุกลูปอ้างอิงจาก `WS-STUDENT-COUNT` ซึ่งเป็นค่าที่ผู้ใช้ป้อนเอง ไม่ได้ hard-code
เลข 20 ไว้ในตรรกะการวนลูป) หน่วยความจำที่ใช้จะเป็น 3 ไบต์ × 50 ช่อง = 150 ไบต์ (เพิ่มขึ้นจากเดิม 60 ไบต์)
ควรปรับข้อความ prompt "Enter number of students (1-20)" เป็น "(1-50)" ด้วยเพื่อให้ตรงกับขอบเขตใหม่

---

## ขั้นตอนที่ 147: รับคะแนนด้วย PERFORM VARYING พร้อมตรวจสอบความถูกต้อง

### แนวคิด

ขั้นตอนนี้เพิ่ม Paragraph `3000-COLLECT-SCORES` ที่ถูกเรียกผ่าน `PERFORM ... VARYING` (เทคนิคจาก
Part 014 ขั้นตอนที่ 137: ผสาน `PERFORM VARYING` เข้ากับการเรียก Paragraph) เพื่อรับคะแนนทีละคนเข้า
ตาราง พร้อมทั้งเพิ่มการตรวจสอบขอบเขตให้ `2000-GET-STUDENT-COUNT` (1-20 คน) และตรวจสอบคะแนนแต่ละคน
(0-100 คะแนน) ด้วย `PERFORM WITH TEST AFTER` ผสมกับ `IF` (เทคนิคเดียวกับที่ใช้ในเครื่องคิดเลข
ขั้นตอนที่ 144 ที่ใช้ลูปตรวจสอบซ้ำจนกว่าจะได้ค่าที่ถูกต้อง)

### ตัวอย่างโค้ด (เวอร์ชันขั้นตอนที่ 147)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP147.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STUDENT-COUNT    PIC 9(2)      VALUE 0.
       01  WS-STUDENT-IDX      PIC 9(2)      VALUE 0.
       01  WS-SCORE-TABLE.
           05  WS-STUDENT-SCORE PIC 9(3) OCCURS 20 TIMES.

       PROCEDURE DIVISION.
       0000-MAIN-PROCESS.
           PERFORM 1000-SHOW-WELCOME
           PERFORM 2000-GET-STUDENT-COUNT
           PERFORM 3000-COLLECT-SCORES
                   VARYING WS-STUDENT-IDX FROM 1 BY 1
                   UNTIL WS-STUDENT-IDX > WS-STUDENT-COUNT
           PERFORM 3900-ECHO-TABLE
                   VARYING WS-STUDENT-IDX FROM 1 BY 1
                   UNTIL WS-STUDENT-IDX > WS-STUDENT-COUNT
           STOP RUN.

       1000-SHOW-WELCOME.
           DISPLAY "=================================".
           DISPLAY "   STUDENT GRADE CALCULATOR".
           DISPLAY "=================================".

       2000-GET-STUDENT-COUNT.
           PERFORM WITH TEST AFTER
                   UNTIL WS-STUDENT-COUNT >= 1
                       AND WS-STUDENT-COUNT <= 20
               DISPLAY "Enter number of students (1-20): "
                   WITH NO ADVANCING
               ACCEPT WS-STUDENT-COUNT
               IF WS-STUDENT-COUNT < 1 OR WS-STUDENT-COUNT > 20
                   DISPLAY "Invalid: please enter a value 1-20."
               END-IF
           END-PERFORM.

       3000-COLLECT-SCORES.
           PERFORM WITH TEST AFTER
                   UNTIL WS-STUDENT-SCORE(WS-STUDENT-IDX) <= 100
               DISPLAY "Enter score for student "
                   WS-STUDENT-IDX " (0-100): " WITH NO ADVANCING
               ACCEPT WS-STUDENT-SCORE(WS-STUDENT-IDX)
               IF WS-STUDENT-SCORE(WS-STUDENT-IDX) > 100
                   DISPLAY "Invalid score. Must be 0-100."
               END-IF
           END-PERFORM.

       3900-ECHO-TABLE.
           DISPLAY "Stored score(" WS-STUDENT-IDX ") = "
               WS-STUDENT-SCORE(WS-STUDENT-IDX).
```

(หมายเหตุ: `3900-ECHO-TABLE` เป็น Paragraph ชั่วคราวสำหรับขั้นตอนนี้เท่านั้น เพื่อพิสูจน์ว่าข้อมูลถูกเก็บ
ลงตารางถูกต้อง — Paragraph นี้จะถูกแทนที่ด้วยการพิมพ์รายงานจริงในขั้นตอนที่ 149)

### ผลลัพธ์จริงที่ได้ (ทดสอบทั้งกรณีปกติและกรณีคะแนนไม่ถูกต้อง)

ป้อนค่า: จำนวนนักเรียน `3`, คะแนนคนที่ 1 ป้อน `150` (ผิด) แล้วแก้เป็น `80`, คนที่ 2 ป้อน `70`,
คนที่ 3 ป้อน `60`:

```
=================================
   STUDENT GRADE CALCULATOR
=================================
Enter number of students (1-20): Enter score for student 01 (0-100): Invalid score. Must be 0-100.
Enter score for student 01 (0-100): Enter score for student 02 (0-100): Enter score for student 03 (0-100): Stored score(01) = 080
Stored score(02) = 070
Stored score(03) = 060
```

การป้อนค่า `150` ครั้งแรกถูกปฏิเสธอย่างถูกต้อง และเมื่อแก้ไขเป็น `80` แล้ว ค่าที่ถูกเก็บลงตารางคือ
`080` (ค่าที่ถูกต้อง) ไม่ใช่ `150` ที่ถูกปฏิเสธไปก่อนหน้า

### กับดักที่ค้นพบจริงระหว่างพัฒนา: ฟิลด์ไม่มีเครื่องหมายกลืนเครื่องหมายลบ

ระหว่างทดสอบโปรแกรมนี้ ได้ลองป้อนคะแนนติดลบ (`-25`) เพื่อดูว่าการตรวจสอบ `> 100` จะจับกรณีนี้ได้ด้วย
หรือไม่ ผลลัพธ์ที่ได้จริงคือ:

```
Enter score for student 01 (0-100): Stored score(01) = 025
```

**สังเกตว่าไม่มีข้อความ "Invalid score" ปรากฏเลย และค่าที่เก็บได้คือ `025` ไม่ใช่ `-25`!** นี่คือ
กับดักคลาสสิกที่ทบทวนตรงกับ Part 008 ขั้นตอนที่ 79 (Pitfall 2): เนื่องจาก `WS-STUDENT-SCORE` ประกาศ
เป็น `PIC 9(3)` (**ไม่มี** `S` นำหน้า คือไม่รองรับเครื่องหมาย) เมื่อป้อนค่า `-25` เข้าไป **เครื่องหมายลบ
จะถูกทิ้งไปเงียบ ๆ เหลือแต่ค่าสัมบูรณ์ (absolute value)** กลายเป็น 25 ซึ่งเป็นค่าที่ผ่านการตรวจสอบ
`<= 100` ได้พอดี ทำให้ข้อมูลที่ผิดพลาดจริง (คะแนนติดลบไม่สมเหตุสมผลทางธุรกิจ) หลุดรอดผ่านการตรวจสอบไปได้
โดยไม่มีการแจ้งเตือนใด ๆ

### ข้อควรระวัง

- **ความจริงที่ว่า `PIC 9(3)` (ไม่มีเครื่องหมาย) ไม่สามารถเก็บเลขลบได้จริง ไม่ได้แปลว่าปลอดภัยจาก
  อินพุตที่ติดลบเสมอไป** — ตามที่พิสูจน์ข้างต้น การป้อนเลขลบไม่ทำให้เกิด error แต่กลับถูกแปลงเป็นค่า
  สัมบูรณ์อย่างเงียบ ๆ ในทางปฏิบัติ ควรตรวจสอบด้วยฟิลด์ `PIC S9(3)` (มีเครื่องหมาย) แล้วตรวจสอบเงื่อนไข
  `< 0 OR > 100` แยกต่างหาก หากต้องการดักจับกรณีนี้อย่างเข้มงวดในระบบจริง แต่สำหรับโปรเจกต์ระดับ
  เฟส 1 นี้ เราถือว่านี่เป็นข้อจำกัดที่ทราบและยอมรับได้ (known limitation) และเป็นแบบฝึกหัดที่ดีในการ
  ทบทวนบทเรียนเก่า
- การตรวจสอบข้อมูลนำเข้า (input validation) ไม่มีทางสมบูรณ์แบบ 100% ได้ในครั้งเดียว — นักพัฒนาที่ดี
  จะค้นหาและบันทึกข้อจำกัดที่ทราบไว้เสมอ (documented limitation) แทนที่จะสมมติว่าโค้ดปลอดภัยสมบูรณ์
  โดยไม่ได้ทดสอบกรณีขอบ (edge case) จริง

### แบบฝึกหัดที่ 147.1

**โจทย์**: จงแก้ไข `3000-COLLECT-SCORES` ให้ตรวจจับกรณีคะแนนติดลบได้อย่างถูกต้อง โดยเปลี่ยน
`WS-STUDENT-SCORE` เป็น `PIC S9(3)` แล้วปรับเงื่อนไขตรวจสอบ

**เฉลยแนวทาง**:

```cobol
       01  WS-SCORE-TABLE.
           05  WS-STUDENT-SCORE PIC S9(3) OCCURS 20 TIMES.
       ...
       3000-COLLECT-SCORES.
           PERFORM WITH TEST AFTER
                   UNTIL WS-STUDENT-SCORE(WS-STUDENT-IDX) >= 0
                       AND WS-STUDENT-SCORE(WS-STUDENT-IDX) <= 100
               DISPLAY "Enter score for student "
                   WS-STUDENT-IDX " (0-100): " WITH NO ADVANCING
               ACCEPT WS-STUDENT-SCORE(WS-STUDENT-IDX)
               IF WS-STUDENT-SCORE(WS-STUDENT-IDX) < 0
                   OR WS-STUDENT-SCORE(WS-STUDENT-IDX) > 100
                   DISPLAY "Invalid score. Must be 0-100."
               END-IF
           END-PERFORM.
```

การเปลี่ยนเป็น `PIC S9(3)` ทำให้ฟิลด์นี้เก็บเครื่องหมายลบไว้ได้จริง เงื่อนไข `< 0` จึงสามารถตรวจจับ
ค่าติดลบและบังคับให้ผู้ใช้ป้อนใหม่ได้อย่างถูกต้อง แก้ไขกับดักที่พบในขั้นตอนนี้ได้อย่างสมบูรณ์

---

## ขั้นตอนที่ 148: คำนวณค่าเฉลี่ย คะแนนสูงสุด และคะแนนต่ำสุดด้วย COMPUTE

### แนวคิด

ขั้นตอนนี้เพิ่ม Paragraph `4000-CALCULATE-STATISTICS` ที่วนลูปผ่านตารางคะแนนหนึ่งรอบ สะสมผลรวมด้วย
`ADD` และเปรียบเทียบหาค่าสูงสุด/ต่ำสุดด้วย `IF` (รูปแบบ accumulate-while-iterate ที่ทบทวนจาก Part 013
ขั้นตอนที่ 126) แล้วคำนวณค่าเฉลี่ยด้วย `COMPUTE ... ROUNDED` (ทบทวนจาก Part 009 ขั้นตอนที่ 86)

### ตัวอย่างโค้ด (เวอร์ชันขั้นตอนที่ 148)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP148.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STUDENT-COUNT    PIC 9(2)      VALUE 0.
       01  WS-STUDENT-IDX      PIC 9(2)      VALUE 0.
       01  WS-SCORE-TABLE.
           05  WS-STUDENT-SCORE PIC 9(3) OCCURS 20 TIMES.
       01  WS-SCORE-TOTAL      PIC 9(5)      VALUE 0.
       01  WS-SCORE-AVERAGE    PIC 9(3)V99   VALUE 0.
       01  WS-MAX-SCORE        PIC 9(3)      VALUE 0.
       01  WS-MIN-SCORE        PIC 9(3)      VALUE 100.

       PROCEDURE DIVISION.
       0000-MAIN-PROCESS.
           PERFORM 1000-SHOW-WELCOME
           PERFORM 2000-GET-STUDENT-COUNT
           PERFORM 3000-COLLECT-SCORES
                   VARYING WS-STUDENT-IDX FROM 1 BY 1
                   UNTIL WS-STUDENT-IDX > WS-STUDENT-COUNT
           PERFORM 4000-CALCULATE-STATISTICS
           DISPLAY "Total  : " WS-SCORE-TOTAL
           DISPLAY "Average: " WS-SCORE-AVERAGE
           DISPLAY "Highest: " WS-MAX-SCORE
           DISPLAY "Lowest : " WS-MIN-SCORE
           STOP RUN.

       1000-SHOW-WELCOME.
           DISPLAY "=================================".
           DISPLAY "   STUDENT GRADE CALCULATOR".
           DISPLAY "=================================".

       2000-GET-STUDENT-COUNT.
           PERFORM WITH TEST AFTER
                   UNTIL WS-STUDENT-COUNT >= 1
                       AND WS-STUDENT-COUNT <= 20
               DISPLAY "Enter number of students (1-20): "
                   WITH NO ADVANCING
               ACCEPT WS-STUDENT-COUNT
               IF WS-STUDENT-COUNT < 1 OR WS-STUDENT-COUNT > 20
                   DISPLAY "Invalid: please enter a value 1-20."
               END-IF
           END-PERFORM.

       3000-COLLECT-SCORES.
           PERFORM WITH TEST AFTER
                   UNTIL WS-STUDENT-SCORE(WS-STUDENT-IDX) <= 100
               DISPLAY "Enter score for student "
                   WS-STUDENT-IDX " (0-100): " WITH NO ADVANCING
               ACCEPT WS-STUDENT-SCORE(WS-STUDENT-IDX)
               IF WS-STUDENT-SCORE(WS-STUDENT-IDX) > 100
                   DISPLAY "Invalid score. Must be 0-100."
               END-IF
           END-PERFORM.

       4000-CALCULATE-STATISTICS.
           PERFORM VARYING WS-STUDENT-IDX FROM 1 BY 1
                   UNTIL WS-STUDENT-IDX > WS-STUDENT-COUNT
               ADD WS-STUDENT-SCORE(WS-STUDENT-IDX) TO WS-SCORE-TOTAL
               IF WS-STUDENT-SCORE(WS-STUDENT-IDX) > WS-MAX-SCORE
                   MOVE WS-STUDENT-SCORE(WS-STUDENT-IDX) TO WS-MAX-SCORE
               END-IF
               IF WS-STUDENT-SCORE(WS-STUDENT-IDX) < WS-MIN-SCORE
                   MOVE WS-STUDENT-SCORE(WS-STUDENT-IDX) TO WS-MIN-SCORE
               END-IF
           END-PERFORM
           COMPUTE WS-SCORE-AVERAGE ROUNDED =
               WS-SCORE-TOTAL / WS-STUDENT-COUNT.
```

### ผลลัพธ์จริงที่ได้

ป้อนค่า: จำนวนนักเรียน `4`, คะแนน `80`, `90`, `70`, `60`:

```
=================================
   STUDENT GRADE CALCULATOR
=================================
Enter number of students (1-20): Enter score for student 01 (0-100): Enter score for student 02 (0-100): Enter score for student 03 (0-100): Enter score for student 04 (0-100): Total  : 00300
Average: 075.00
Highest: 090
Lowest : 060
```

ตรวจสอบด้วยมือ: 80+90+70+60 = 300 ✓, ค่าเฉลี่ย 300÷4 = 75.00 ✓, สูงสุด 90 ✓, ต่ำสุด 60 ✓
ผลลัพธ์ถูกต้องครบทุกค่า

### อธิบายโค้ดทีละส่วน

- `WS-MAX-SCORE` เริ่มต้นที่ `VALUE 0` เพื่อให้คะแนนแรกที่พบจะมากกว่าค่าเริ่มต้นเสมอและถูกบันทึกไว้
  ส่วน `WS-MIN-SCORE` เริ่มต้นที่ `VALUE 100` (คะแนนสูงสุดที่เป็นไปได้) เพื่อให้คะแนนแรกที่พบจะน้อยกว่า
  ค่าเริ่มต้นเสมอเช่นกัน — นี่คือเทคนิคมาตรฐานในการหาค่าสูงสุด/ต่ำสุดที่ทบทวนจาก Part 013 ขั้นตอนที่ 126
  (แบบฝึกหัด 126.1) ที่ต้อง**เลือกค่าเริ่มต้นให้อยู่ตรงข้ามกับสิ่งที่กำลังค้นหา**
- `COMPUTE WS-SCORE-AVERAGE ROUNDED = WS-SCORE-TOTAL / WS-STUDENT-COUNT.` ใช้ `ROUNDED` เพื่อ
  ปัดเศษค่าเฉลี่ยให้แม่นยำ (ทบทวนจาก Part 009 ขั้นตอนที่ 86) — หากลืมใส่ `ROUNDED` ค่าเฉลี่ยที่มีเศษ
  ทศนิยมจะถูกตัดทิ้งแทนที่จะปัดเศษ ซึ่งอาจทำให้ผลลัพธ์คลาดเคลื่อนเล็กน้อยจากความเป็นจริง

### ข้อควรระวัง

- `WS-SCORE-TOTAL` ประกาศเป็น `PIC 9(5)` (สูงสุด 99999) ในขณะที่คะแนนสูงสุดต่อคนคือ 100 และรองรับ
  ได้สูงสุด 20 คน ผลรวมสูงสุดที่เป็นไปได้คือ 100×20 = 2000 ซึ่งเล็กกว่าขีดจำกัดของฟิลด์มาก — การเผื่อ
  ขนาดฟิลด์ให้ใหญ่กว่าที่จำเป็นเล็กน้อยเป็นแนวปฏิบัติที่ปลอดภัย (ทบทวนจาก Part 008-009 เรื่องอันตราย
  ของฟิลด์ที่เล็กเกินไป) แต่ก็ไม่ควรใหญ่เกินจำเป็นจนสิ้นเปลืองหน่วยความจำโดยไม่มีเหตุผล
- **ห้ามลืม**ว่าถ้า `WS-STUDENT-COUNT` เป็น 0 คำสั่ง `COMPUTE ... = WS-SCORE-TOTAL / WS-STUDENT-COUNT`
  จะกลายเป็นการหารด้วยศูนย์ทันที แต่เนื่องจากขั้นตอนที่ 147 ได้ป้องกันไว้แล้วว่า `WS-STUDENT-COUNT`
  ต้องมีค่าอย่างน้อย 1 เสมอ (ผ่านการตรวจสอบใน `2000-GET-STUDENT-COUNT`) กรณีนี้จึงไม่มีทางเกิดขึ้นได้
  ในโปรแกรมฉบับสมบูรณ์ — นี่คือตัวอย่างที่ดีว่าทำไมการตรวจสอบข้อมูลนำเข้าตั้งแต่ต้น (ขั้นตอนที่ 147)
  จึงสำคัญมากต่อความถูกต้องของขั้นตอนคำนวณที่ตามมาทีหลัง

### แบบฝึกหัดที่ 148.1

**โจทย์**: จงเพิ่มการคำนวณ "ผลต่างระหว่างคะแนนสูงสุดกับคะแนนต่ำสุด" (range) เก็บไว้ในตัวแปรใหม่ชื่อ
`WS-SCORE-RANGE`

**เฉลย**: เพิ่มตัวแปรและคำสั่งคำนวณดังนี้:

```cobol
       01  WS-SCORE-RANGE      PIC 9(3)      VALUE 0.
       ...
       4000-CALCULATE-STATISTICS.
           PERFORM VARYING WS-STUDENT-IDX FROM 1 BY 1
                   UNTIL WS-STUDENT-IDX > WS-STUDENT-COUNT
               ADD WS-STUDENT-SCORE(WS-STUDENT-IDX) TO WS-SCORE-TOTAL
               IF WS-STUDENT-SCORE(WS-STUDENT-IDX) > WS-MAX-SCORE
                   MOVE WS-STUDENT-SCORE(WS-STUDENT-IDX) TO WS-MAX-SCORE
               END-IF
               IF WS-STUDENT-SCORE(WS-STUDENT-IDX) < WS-MIN-SCORE
                   MOVE WS-STUDENT-SCORE(WS-STUDENT-IDX) TO WS-MIN-SCORE
               END-IF
           END-PERFORM
           COMPUTE WS-SCORE-AVERAGE ROUNDED =
               WS-SCORE-TOTAL / WS-STUDENT-COUNT
           SUBTRACT WS-MIN-SCORE FROM WS-MAX-SCORE
               GIVING WS-SCORE-RANGE.
```

จากตัวอย่างข้อมูล (80, 90, 70, 60) ผลต่างจะเป็น 90 - 60 = 30 คะแนน ค่านี้มีประโยชน์ในการดูว่าคะแนน
ของนักเรียนในกลุ่มนี้กระจายตัวมากน้อยเพียงใด

---

## ขั้นตอนที่ 149: ตัดเกรดด้วย EVALUATE + 88-level และพิมพ์รายงานฉบับสมบูรณ์

### แนวคิด

ขั้นตอนสุดท้ายของการสร้างระบบคำนวณเกรด: เพิ่ม Paragraph `5000-PRINT-REPORT` และ `5100-ASSIGN-GRADE`
ที่ผสาน **Condition Name แบบช่วงค่า (88-level กับ `THRU`)** จาก Part 010 ขั้นตอนที่ 97 เข้ากับ
**`EVALUATE TRUE`** จาก Part 011 เพื่อตัดเกรดแต่ละคนอย่างอ่านง่ายและปลอดภัย พร้อมจัดรูปแบบรายงาน
ด้วย Numeric-Edited PICTURE (`Z9`, `ZZ9`, `ZZ9.99`) ที่ทบทวนจาก Part 008 ขั้นตอนที่ 77

### ซอร์สโค้ดฉบับสมบูรณ์: `gradecalc.cob`

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. GRADE-CALCULATOR.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STUDENT-COUNT    PIC 9(2)      VALUE 0.
       01  WS-STUDENT-IDX      PIC 9(2)      VALUE 0.
       01  WS-SCORE-TABLE.
           05  WS-STUDENT-SCORE PIC 9(3) OCCURS 20 TIMES.
       01  WS-SCORE-TOTAL      PIC 9(5)      VALUE 0.
       01  WS-SCORE-AVERAGE    PIC 9(3)V99   VALUE 0.
       01  WS-MAX-SCORE        PIC 9(3)      VALUE 0.
       01  WS-MIN-SCORE        PIC 9(3)      VALUE 100.
       01  WS-PASS-COUNT       PIC 9(2)      VALUE 0.
       01  WS-FAIL-COUNT       PIC 9(2)      VALUE 0.

       01  WS-CURRENT-SCORE    PIC 9(3)      VALUE 0.
           88  SCORE-IS-A                VALUE 90 THRU 100.
           88  SCORE-IS-B                VALUE 80 THRU 89.
           88  SCORE-IS-C                VALUE 70 THRU 79.
           88  SCORE-IS-D                VALUE 60 THRU 69.
           88  SCORE-IS-F                VALUE 0  THRU 59.

       01  WS-CURRENT-GRADE    PIC X(1)      VALUE SPACE.

       01  WS-IDX-DISPLAY      PIC Z9.
       01  WS-SCORE-DISPLAY    PIC ZZ9.
       01  WS-AVG-DISPLAY      PIC ZZ9.99.

       PROCEDURE DIVISION.
       0000-MAIN-PROCESS.
           PERFORM 1000-SHOW-WELCOME
           PERFORM 2000-GET-STUDENT-COUNT
           PERFORM 3000-COLLECT-SCORES
                   VARYING WS-STUDENT-IDX FROM 1 BY 1
                   UNTIL WS-STUDENT-IDX > WS-STUDENT-COUNT
           PERFORM 4000-CALCULATE-STATISTICS
           PERFORM 5000-PRINT-REPORT
           STOP RUN.

       1000-SHOW-WELCOME.
           DISPLAY "=================================".
           DISPLAY "   STUDENT GRADE CALCULATOR".
           DISPLAY "=================================".

       2000-GET-STUDENT-COUNT.
           PERFORM WITH TEST AFTER
                   UNTIL WS-STUDENT-COUNT >= 1
                       AND WS-STUDENT-COUNT <= 20
               DISPLAY "Enter number of students (1-20): "
                   WITH NO ADVANCING
               ACCEPT WS-STUDENT-COUNT
               IF WS-STUDENT-COUNT < 1 OR WS-STUDENT-COUNT > 20
                   DISPLAY "Invalid: please enter a value 1-20."
               END-IF
           END-PERFORM.

       3000-COLLECT-SCORES.
           PERFORM WITH TEST AFTER
                   UNTIL WS-STUDENT-SCORE(WS-STUDENT-IDX) <= 100
               DISPLAY "Enter score for student "
                   WS-STUDENT-IDX " (0-100): " WITH NO ADVANCING
               ACCEPT WS-STUDENT-SCORE(WS-STUDENT-IDX)
               IF WS-STUDENT-SCORE(WS-STUDENT-IDX) > 100
                   DISPLAY "Invalid score. Must be 0-100."
               END-IF
           END-PERFORM.

       4000-CALCULATE-STATISTICS.
           PERFORM VARYING WS-STUDENT-IDX FROM 1 BY 1
                   UNTIL WS-STUDENT-IDX > WS-STUDENT-COUNT
               ADD WS-STUDENT-SCORE(WS-STUDENT-IDX) TO WS-SCORE-TOTAL
               IF WS-STUDENT-SCORE(WS-STUDENT-IDX) > WS-MAX-SCORE
                   MOVE WS-STUDENT-SCORE(WS-STUDENT-IDX) TO WS-MAX-SCORE
               END-IF
               IF WS-STUDENT-SCORE(WS-STUDENT-IDX) < WS-MIN-SCORE
                   MOVE WS-STUDENT-SCORE(WS-STUDENT-IDX) TO WS-MIN-SCORE
               END-IF
           END-PERFORM
           COMPUTE WS-SCORE-AVERAGE ROUNDED =
               WS-SCORE-TOTAL / WS-STUDENT-COUNT.

       5000-PRINT-REPORT.
           DISPLAY " ".
           DISPLAY "======= GRADE REPORT =======".
           DISPLAY "No.  Score  Grade".
           DISPLAY "---  -----  -----".
           PERFORM VARYING WS-STUDENT-IDX FROM 1 BY 1
                   UNTIL WS-STUDENT-IDX > WS-STUDENT-COUNT
               MOVE WS-STUDENT-SCORE(WS-STUDENT-IDX) TO WS-CURRENT-SCORE
               PERFORM 5100-ASSIGN-GRADE
               MOVE WS-STUDENT-IDX TO WS-IDX-DISPLAY
               MOVE WS-CURRENT-SCORE TO WS-SCORE-DISPLAY
               DISPLAY " " WS-IDX-DISPLAY "   " WS-SCORE-DISPLAY
                   "    " WS-CURRENT-GRADE
           END-PERFORM
           DISPLAY "-----------------------------".
           MOVE WS-SCORE-AVERAGE TO WS-AVG-DISPLAY
           DISPLAY "Total students : " WS-STUDENT-COUNT.
           DISPLAY "Highest score  : " WS-MAX-SCORE.
           DISPLAY "Lowest score   : " WS-MIN-SCORE.
           DISPLAY "Average score  : " WS-AVG-DISPLAY.
           DISPLAY "Passed (D-A)   : " WS-PASS-COUNT.
           DISPLAY "Failed (F)     : " WS-FAIL-COUNT.
           DISPLAY "=============================".

       5100-ASSIGN-GRADE.
           EVALUATE TRUE
               WHEN SCORE-IS-A
                   MOVE "A" TO WS-CURRENT-GRADE
                   ADD 1 TO WS-PASS-COUNT
               WHEN SCORE-IS-B
                   MOVE "B" TO WS-CURRENT-GRADE
                   ADD 1 TO WS-PASS-COUNT
               WHEN SCORE-IS-C
                   MOVE "C" TO WS-CURRENT-GRADE
                   ADD 1 TO WS-PASS-COUNT
               WHEN SCORE-IS-D
                   MOVE "D" TO WS-CURRENT-GRADE
                   ADD 1 TO WS-PASS-COUNT
               WHEN SCORE-IS-F
                   MOVE "F" TO WS-CURRENT-GRADE
                   ADD 1 TO WS-FAIL-COUNT
               WHEN OTHER
                   MOVE "?" TO WS-CURRENT-GRADE
           END-EVALUATE.
```

### วิธีทดสอบ

```bash
cobc -x -o gradecalc gradecalc.cob
```

ไฟล์ `grade_input.txt`:

```
5
95
82
71
58
100
```

รันด้วย `./gradecalc < grade_input.txt`

### ผลลัพธ์จริงที่ได้

```
=================================
   STUDENT GRADE CALCULATOR
=================================
Enter number of students (1-20): Enter score for student 01 (0-100): Enter score for student 02 (0-100): Enter score for student 03 (0-100): Enter score for student 04 (0-100): Enter score for student 05 (0-100): 
======= GRADE REPORT =======
No.  Score  Grade
---  -----  -----
  1    95    A
  2    82    B
  3    71    C
  4    58    F
  5   100    A
-----------------------------
Total students : 05
Highest score  : 100
Lowest score   : 058
Average score  :  81.20
Passed (D-A)   : 04
Failed (F)     : 01
=============================
```

**ตรวจสอบผลลัพธ์**: 95→A ✓, 82→B ✓, 71→C ✓, 58→F ✓, 100→A ✓ (ทุกเกรดตรงกับช่วงคะแนนที่กำหนด
ผ่าน 88-level) ค่าเฉลี่ย (95+82+71+58+100)/5 = 406/5 = 81.20 ✓, สูงสุด 100 ✓, ต่ำสุด 58 ✓,
ผ่าน 4 คน (ทุกเกรดยกเว้น F) ✓, ไม่ผ่าน 1 คน ✓ — **ถูกต้องครบทุกค่า**

### อธิบายโค้ดทีละส่วน

- `EVALUATE TRUE` ร่วมกับ `WHEN SCORE-IS-A`, `WHEN SCORE-IS-B` ฯลฯ คือรูปแบบมาตรฐานที่ COBOL
  นิยมใช้เพื่อตรวจสอบ **Condition Name หลายตัวเรียงตามลำดับ** — วิธีนี้อ่านได้เกือบเหมือนภาษาอังกฤษ
  ("ประเมินว่าเงื่อนไขไหนเป็นจริง: ถ้าคะแนนเป็น A ให้ทำสิ่งนี้ ถ้าเป็น B ให้ทำสิ่งนั้น") และปลอดภัยกว่า
  การเขียน Nested IF ซ้อนกันหลายชั้นแบบที่เรียนใน Part 010 ขั้นตอนที่ 92 มาก เพราะไม่ต้องนับ `END-IF`
  ให้ครบและไม่มีปัญหา dangling-else
- `MOVE WS-STUDENT-SCORE(WS-STUDENT-IDX) TO WS-CURRENT-SCORE` คือขั้นตอนสำคัญที่**คัดลอกค่าจาก
  ตาราง**มาไว้ในตัวแปรเดี่ยว `WS-CURRENT-SCORE` ก่อนตัดเกรด เหตุผลคือ 88-level (`SCORE-IS-A` ฯลฯ)
  ต้องผูกกับตัวแปรเดี่ยวเดี่ยวตัวหนึ่งที่ประกาศไว้ตายตัวตอนคอมไพล์ ไม่สามารถผูกกับสมาชิกตารางที่ subscript
  เปลี่ยนไปเรื่อย ๆ ได้โดยตรง (ข้อจำกัดนี้เป็นเหตุผลสำคัญที่ทำให้ต้องมีตัวแปรกลาง `WS-CURRENT-SCORE`
  แยกจากตาราง `WS-STUDENT-SCORE`)
- `WS-IDX-DISPLAY PIC Z9` และ `WS-SCORE-DISPLAY PIC ZZ9` ใช้ตัด 0 นำหน้าที่ไม่จำเป็นออกจากการแสดงผล
  (Zero Suppression ทบทวนจาก Part 008 ขั้นตอนที่ 77) ทำให้รายงานอ่านง่ายกว่าการแสดง `01`, `095` ตรง ๆ
  จากฟิลด์ `PIC 9(2)`/`PIC 9(3)` ธรรมดา

### ข้อควรระวัง

- 88-level แบบ `THRU` (เช่น `SCORE-IS-A VALUE 90 THRU 100`) **ต้องออกแบบให้ช่วงคะแนนไม่ทับซ้อนกัน**
  เหมือนที่เตือนไว้ใน Part 010 ขั้นตอนที่ 97 มิเช่นนั้นคะแนนบางค่าอาจตรงกับหลายเงื่อนไขพร้อมกัน ทำให้
  ผลลัพธ์ขึ้นอยู่กับลำดับการเขียน `WHEN` (COBOL จะหยุดที่เงื่อนไขแรกที่เป็นจริงเสมอ) — ตัวอย่างนี้ออกแบบ
  ช่วงคะแนนให้ครอบคลุมตั้งแต่ 0 ถึง 100 พอดีโดยไม่มีช่องว่างหรือทับซ้อน (F: 0-59, D: 60-69, C: 70-79,
  B: 80-89, A: 90-100)
- `WHEN OTHER` ใน `5100-ASSIGN-GRADE` ทำหน้าที่เป็นตาข่ายนิรภัย (safety net) สำหรับกรณีที่คะแนน
  ไม่ตรงกับช่วงใดเลย (ในทางทฤษฎีไม่ควรเกิดขึ้นเพราะช่วงคะแนนครอบคลุม 0-100 ครบถ้วน และการตรวจสอบใน
  ขั้นตอนที่ 147 บังคับให้คะแนนอยู่ในช่วง 0-100 อยู่แล้ว) แต่การมี `WHEN OTHER` ไว้เสมอเป็นแนวปฏิบัติ
  ที่ดีเพื่อป้องกันพฤติกรรมที่ไม่คาดคิดในกรณีที่โค้ดถูกแก้ไขในอนาคตแล้วมีช่องว่างในช่วงคะแนนโดยไม่ตั้งใจ

### แบบฝึกหัดที่ 149.1 — ต่อยอดโปรเจกต์: แสดงจำนวนนักเรียนแต่ละเกรด

**โจทย์**: จงต่อยอดระบบคำนวณเกรดให้แสดงจำนวนนักเรียนที่ได้แต่ละเกรด (A, B, C, D, F) แยกกันในรายงานสรุป
นอกเหนือจากจำนวนผ่าน/ไม่ผ่านที่มีอยู่แล้ว

**เฉลย**: เพิ่มตัวนับ 5 ตัวและอัปเดตใน `5100-ASSIGN-GRADE`:

```cobol
       01  WS-COUNT-A          PIC 9(2)      VALUE 0.
       01  WS-COUNT-B          PIC 9(2)      VALUE 0.
       01  WS-COUNT-C          PIC 9(2)      VALUE 0.
       01  WS-COUNT-D          PIC 9(2)      VALUE 0.
       01  WS-COUNT-F          PIC 9(2)      VALUE 0.
```

```cobol
       5100-ASSIGN-GRADE.
           EVALUATE TRUE
               WHEN SCORE-IS-A
                   MOVE "A" TO WS-CURRENT-GRADE
                   ADD 1 TO WS-PASS-COUNT
                   ADD 1 TO WS-COUNT-A
               WHEN SCORE-IS-B
                   MOVE "B" TO WS-CURRENT-GRADE
                   ADD 1 TO WS-PASS-COUNT
                   ADD 1 TO WS-COUNT-B
               WHEN SCORE-IS-C
                   MOVE "C" TO WS-CURRENT-GRADE
                   ADD 1 TO WS-PASS-COUNT
                   ADD 1 TO WS-COUNT-C
               WHEN SCORE-IS-D
                   MOVE "D" TO WS-CURRENT-GRADE
                   ADD 1 TO WS-PASS-COUNT
                   ADD 1 TO WS-COUNT-D
               WHEN SCORE-IS-F
                   MOVE "F" TO WS-CURRENT-GRADE
                   ADD 1 TO WS-FAIL-COUNT
                   ADD 1 TO WS-COUNT-F
               WHEN OTHER
                   MOVE "?" TO WS-CURRENT-GRADE
           END-EVALUATE.
```

แล้วเพิ่มบรรทัดแสดงผลใน `5000-PRINT-REPORT` ก่อนบรรทัดปิดท้าย `"============================="`:

```cobol
           DISPLAY "Grade A count  : " WS-COUNT-A.
           DISPLAY "Grade B count  : " WS-COUNT-B.
           DISPLAY "Grade C count  : " WS-COUNT-C.
           DISPLAY "Grade D count  : " WS-COUNT-D.
           DISPLAY "Grade F count  : " WS-COUNT-F.
```

**ทดสอบจริง (คอมไพล์และรันแล้ว)** ด้วยข้อมูลชุดเดียวกัน (95, 82, 71, 58, 100):

```
Passed (D-A)   : 04
Failed (F)     : 01
Grade A count  : 02
Grade B count  : 01
Grade C count  : 01
Grade D count  : 00
Grade F count  : 01
```

ผลลัพธ์ถูกต้อง: มีเกรด A สองคน (95 และ 100), B หนึ่งคน (82), C หนึ่งคน (71), D ไม่มีเลย, F หนึ่งคน (58)
รวมกันได้ 2+1+1+0+1 = 5 คน ตรงกับจำนวนนักเรียนทั้งหมดพอดี

---

## ขั้นตอนที่ 150: การรวมระบบ การทดสอบครบทุกกรณี และสรุปเฟส 1

### แนวคิด

ขั้นตอนสุดท้ายของ Part นี้คือการทดสอบทั้งสองโปรแกรมอย่างละเอียดในกรณีขอบ (edge cases) ที่สำคัญ
เพื่อยืนยันว่าทุกความต้องการที่กำหนดไว้ในขั้นตอนที่ 141 ได้รับการตอบสนองอย่างสมบูรณ์ จากนั้นจะสรุป
ภาพรวมของทั้งเฟส 1 (Part 001-015)

### การทดสอบเครื่องคิดเลข: กรณีขอบทั้งหมด

**กรณีที่ 1: เลขติดลบผสมกับเลขทศนิยม** (ทดสอบแล้วใน ขั้นตอนที่ 145)

```
Result = +000000004.75    (จาก -15.50 + 20.25)
Result = -000000004.50    (จาก 5.5 - 10)
```

**กรณีที่ 2: ขอบเขตคะแนนเกรด (boundary values)** — ทดสอบทุกจุดต่อของช่วงเกรด (59/60, 69/70, 79/80,
89/90) เพื่อยืนยันว่า 88-level แบบ `THRU` ตัดขอบเขตถูกต้องเป๊ะไม่มีรอยต่อผิดพลาด:

ป้อนคะแนน: 59, 60, 69, 70, 79, 80, 89, 90, 100

```
======= GRADE REPORT =======
No.  Score  Grade
---  -----  -----
  1    59    F
  2    60    D
  3    69    D
  4    70    C
  5    79    C
  6    80    B
  7    89    B
  8    90    A
  9   100    A
-----------------------------
Total students : 09
Highest score  : 100
Lowest score   : 059
Average score  :  77.33
Passed (D-A)   : 08
Failed (F)     : 01
=============================
```

ทุกจุดต่อขอบเขตถูกต้องแม่นยำ: 59→F, 60→D (จุดต่อ F/D ถูกต้อง), 69→D, 70→C (จุดต่อ D/C ถูกต้อง),
79→C, 80→B (จุดต่อ C/B ถูกต้อง), 89→B, 90→A (จุดต่อ B/A ถูกต้อง) — ไม่มีคะแนนใดตกหล่นหรือถูกจัด
เกรดผิดพลาดที่รอยต่อเลย ค่าเฉลี่ย (59+60+69+70+79+80+89+90+100)/9 = 696/9 = 77.333... ปัดเศษเป็น
77.33 ถูกต้องตามกฎ `ROUNDED`

**กรณีที่ 3: ตรวจสอบข้อมูลนำเข้าที่ไม่ถูกต้องซ้อนกันหลายชั้น** — ทดสอบว่าการ validate จำนวนนักเรียน
และ validate คะแนนทำงานร่วมกันถูกต้อง แม้ผู้ใช้จะป้อนค่าผิดซ้ำ ๆ หลายครั้งติดกัน:

ป้อนค่า: จำนวนนักเรียน `0` (ผิด) → `25` (ผิด) → `3` (ถูก), คะแนนคนที่ 1 ป้อน `150` (ผิด) → `80` (ถูก),
คนที่ 2: `70`, คนที่ 3: `60`

```
Enter number of students (1-20): Invalid: please enter a value 1-20.
Enter number of students (1-20): Invalid: please enter a value 1-20.
Enter number of students (1-20): Enter score for student 01 (0-100): Invalid score. Must be 0-100.
Enter score for student 01 (0-100): Enter score for student 02 (0-100): Enter score for student 03 (0-100): 
======= GRADE REPORT =======
No.  Score  Grade
---  -----  -----
  1    80    B
  2    70    C
  3    60    D
-----------------------------
Total students : 03
Highest score  : 080
Lowest score   : 060
Average score  :  70.00
Passed (D-A)   : 03
Failed (F)     : 00
=============================
```

การตรวจสอบทั้งสองชั้น (จำนวนนักเรียนและคะแนนแต่ละคน) ทำงานถูกต้องอย่างเป็นอิสระจากกัน แม้ผู้ใช้จะป้อน
ค่าผิดติดต่อกันหลายครั้งก็ตาม — ยืนยันว่าโครงสร้าง `PERFORM WITH TEST AFTER` ที่ใช้ในทั้งสองจุดทำงาน
ได้อย่างน่าเชื่อถือ

### สรุปผลการทดสอบทั้งโปรเจกต์

| ความต้องการ (จากขั้นตอนที่ 141) | สถานะ | หลักฐาน |
|---|---|---|
| เครื่องคิดเลข: เมนู 5 ตัวเลือกพร้อมวนลูป | ✅ ผ่าน | ขั้นตอนที่ 142, 145 |
| เครื่องคิดเลข: รับเลขลบและทศนิยม | ✅ ผ่าน | ขั้นตอนที่ 145 (กรณีที่ 1 ข้างต้น) |
| เครื่องคิดเลข: ป้องกันหารด้วยศูนย์ | ✅ ผ่าน | ขั้นตอนที่ 144-145 |
| เครื่องคิดเลข: ถามคำนวณต่อหรือไม่ | ✅ ผ่าน | ขั้นตอนที่ 144-145 |
| ระบบเกรด: รับจำนวนนักเรียน 1-20 พร้อมตรวจสอบ | ✅ ผ่าน | ขั้นตอนที่ 147, 150 (กรณีที่ 3) |
| ระบบเกรด: รับคะแนนพร้อมตรวจสอบ 0-100 | ✅ ผ่าน | ขั้นตอนที่ 147, 150 (กรณีที่ 3) |
| ระบบเกรด: คำนวณรวม/เฉลี่ย/สูงสุด/ต่ำสุด | ✅ ผ่าน | ขั้นตอนที่ 148 |
| ระบบเกรด: ตัดเกรดตามช่วงคะแนนถูกต้องที่ขอบเขต | ✅ ผ่าน | ขั้นตอนที่ 150 (กรณีที่ 2) |
| ระบบเกรด: นับจำนวนผ่าน/ไม่ผ่าน | ✅ ผ่าน | ขั้นตอนที่ 149 |
| ระบบเกรด: รายงานจัดรูปแบบอ่านง่าย | ✅ ผ่าน | ขั้นตอนที่ 149 |

ความต้องการทั้งหมด 10 ข้อจากทั้งสองโปรแกรมผ่านการทดสอบครบถ้วน โปรเจกต์เฟส 1 บรรลุเป้าหมายที่ตั้งไว้

### สถาปัตยกรรมโดยรวมของโปรเจกต์ (ทบทวนภาพใหญ่)

```
                    โปรเจกต์เฟส 1: Calculator + Grade System
                    ==========================================
                              (COBOL Source Files)

    calculator.cob                          gradecalc.cob
    ─────────────                           ─────────────
    PROGRAM-ID. CALCULATOR                  PROGRAM-ID. GRADE-CALCULATOR
    │                                       │
    ├─ WORKING-STORAGE                      ├─ WORKING-STORAGE
    │   (ตัวแปรเดี่ยว + 88-level Y/N)        │   (ตาราง OCCURS 20 + 88-level ช่วงคะแนน)
    │                                       │
    └─ PROCEDURE DIVISION                   └─ PROCEDURE DIVISION
        0000-MAIN-PROCESS                       0000-MAIN-PROCESS
        ├─ 1000-SHOW-WELCOME                    ├─ 1000-SHOW-WELCOME
        ├─ 2000-SHOW-MENU                       ├─ 2000-GET-STUDENT-COUNT
        ├─ 3000-GET-CHOICE                      ├─ 3000-COLLECT-SCORES
        ├─ 4000-GET-NUMBERS                     ├─ 4000-CALCULATE-STATISTICS
        ├─ 5000-CALCULATE                       └─ 5000-PRINT-REPORT
        ├─ 6000-SHOW-RESULT                         └─ 5100-ASSIGN-GRADE
        └─ 7000-ASK-AGAIN
```

ทั้งสองโปรแกรมเป็น**อิสระจากกันโดยสมบูรณ์** (แต่ละไฟล์คอมไพล์และรันแยกกันเป็นโปรแกรมของตัวเอง) แต่ใช้
**รูปแบบสถาปัตยกรรมเดียวกัน** (Numbered Paragraph Convention จาก Part 014) ซึ่งเป็นแนวปฏิบัติที่ดี
ในการทำงานจริง — เมื่อทีมพัฒนาใช้แบบแผนเดียวกันในทุกโปรแกรม โปรแกรมเมอร์คนใหม่ที่มาอ่านโค้ดจะปรับตัว
เข้ากับโปรแกรมที่สองได้เร็วกว่ามาก เพราะโครงสร้างพื้นฐาน (`0000` ควบคุมหลัก, ตัวเลขสูงขึ้นตามลำดับงาน)
เหมือนกันทั้งหมด

**หมายเหตุเรื่องการแยกไฟล์**: ตามที่อธิบายไว้ในขั้นตอนที่ 141 โปรเจกต์นี้ยังไม่ได้แบ่งเป็นหลายไฟล์จริง
(เช่น แยก Copybook สำหรับ WORKING-STORAGE ที่ใช้ร่วมกัน หรือแยก Subprogram สำหรับ logic ที่ใช้ซ้ำ)
เพราะเครื่องมือเหล่านั้น (`COPY`, `CALL`) ยังไม่ได้สอนจนกว่าจะถึง Part 031-033 เมื่อถึงจุดนั้น หากย้อนกลับ
มาดูโปรเจกต์นี้ คุณจะสามารถมองเห็นได้ทันทีว่าส่วนไหนควรแยกเป็น Copybook (เช่น 88-level ของเกรดอาจใช้ร่วม
กับโปรแกรมอื่นในอนาคต) และส่วนไหนควรแยกเป็น Subprogram (เช่น `5100-ASSIGN-GRADE` อาจกลายเป็น
Subprogram ที่ใช้ร่วมกันได้หลายระบบ)

### สรุปท้ายบท

Part นี้คือจุดสิ้นสุดของ **เฟส 1: พื้นฐาน (Foundations)** ของหลักสูตร ครอบคลุม Part 001-015 และ
ขั้นตอนที่ 1-150 มาทบทวนภาพรวมทั้งหมดที่คุณได้เรียนรู้มา:

- **Part 001**: ประวัติศาสตร์ COBOL, เหตุผลที่ยังสำคัญในปี 2026, ภาพรวมอุตสาหกรรมและตลาดงาน
- **Part 002**: ติดตั้ง GnuCOBOL, VS Code, คอมไพล์โปรแกรมแรก
- **Part 003**: โครงสร้าง 4 Divisions และกฎคอลัมน์แบบ Fixed-Format (รวมถึงเหตุผลว่าทำไมโค้ดต้องเป็น
  ภาษาอังกฤษ — กฎเหล็กที่ใช้ตลอดทั้งหลักสูตร)
- **Part 004**: IDENTIFICATION DIVISION และ ENVIRONMENT DIVISION แบบละเอียด
- **Part 005**: WORKING-STORAGE SECTION, Level Number, Group/Elementary Item, FILLER
- **Part 006**: PICTURE Clause และชนิดข้อมูล
- **Part 007**: DISPLAY และ ACCEPT
- **Part 008**: MOVE Statement และกฎการย้ายข้อมูลทั้งหมด (ตัวเลขชิดขวา, ตัวอักษรชิดซ้าย, การตัดข้อมูล
  แบบเงียบ ๆ)
- **Part 009**: เลขคณิต ADD/SUBTRACT/MULTIPLY/DIVIDE/COMPUTE, ROUNDED, ON SIZE ERROR
- **Part 010**: IF-ELSE, Condition Names (88-level), เงื่อนไขผสม AND/OR/NOT
- **Part 011**: EVALUATE Statement
- **Part 012-013**: PERFORM พื้นฐาน, PERFORM UNTIL, PERFORM VARYING, ลูปซ้อนหลายชั้น, TEST BEFORE/AFTER
- **Part 014**: Paragraph, Section, PERFORM...THRU, Numbered Paragraph Convention, กับดัก Fall-through
- **Part 015 (Part นี้)**: นำทุกอย่างมารวมกันเป็นโปรแกรมที่ใช้งานได้จริง 2 โปรแกรม

ทักษะสำคัญที่สุดที่ Part นี้เน้นย้ำไม่ใช่ไวยากรณ์ใหม่ แต่คือ **วิธีคิดแบบวิศวกรซอฟต์แวร์**: เริ่มจาก
การกำหนดความต้องการและออกแบบก่อนเขียนโค้ด (ขั้นตอนที่ 141), สร้างทีละส่วนแล้วทดสอบทันทีในแต่ละขั้น
(ขั้นตอนที่ 142-149), และปิดท้ายด้วยการทดสอบครบทุกกรณีขอบก่อนประกาศว่าโปรเจกต์เสร็จสมบูรณ์ (ขั้นตอนนี้)
นี่คือกระบวนการเดียวกันกับที่นักพัฒนา COBOL มืออาชีพใช้ในการสร้างระบบธุรกิจจริงที่มีความซับซ้อนสูงกว่านี้
มากในโลกการทำงาน

คุณยังได้เห็นตัวอย่างจริงหลายครั้งตลอด Part นี้ว่า **บั๊กที่อันตรายที่สุดใน COBOL มักไม่ใช่บั๊กที่ทำให้
โปรแกรม crash** แต่เป็นบั๊กที่ทำให้โปรแกรม**ทำงานต่อไปได้ปกติแต่ให้ผลลัพธ์ที่ผิดพลาดอย่างเงียบ ๆ**
(ผลลัพธ์ค้างจากการหารศูนย์ที่ไม่ได้ป้องกัน, เครื่องหมายลบที่หายไปจากฟิลด์ไม่มีเครื่องหมาย) — การตระหนักรู้
ถึงกับดักเหล่านี้และรู้วิธีป้องกันคือสิ่งที่แยกโปรแกรมเมอร์ COBOL ที่มีประสบการณ์ออกจากผู้เริ่มต้น

### สู่เฟส 2: ระดับกลาง

เฟส 2 ของหลักสูตร (Part 016-035, ขั้นตอนที่ 151-350) จะพาคุณลึกเข้าไปในหัวข้อที่คุณได้ **แตะขอบเพียง
เล็กน้อย** ในโปรเจกต์นี้แล้ว: `OCCURS` ที่ใช้เก็บคะแนนนักเรียนในบทนี้เป็นเพียงพื้นฐานที่สุดของตาราง
(Table) — Part 016 จะสอนเรื่อง OCCURS แบบเต็มรูปแบบ, Part 017 จะสอน SEARCH สำหรับค้นหาข้อมูลใน
ตารางอย่างมีประสิทธิภาพกว่าการวนลูปแบบ manual, และ Part 018 จะสอนตารางหลายมิติ นอกจากนี้เฟส 2
ยังจะแนะนำการจัดการไฟล์ (Sequential Files, Part 023 เป็นต้นไป) และ Subprogram (`CALL`, Part 031)
ที่จะทำให้คุณสามารถแบ่งโปรแกรมออกเป็นหลายไฟล์จริงได้เป็นครั้งแรก — เฟส 2 จะจบลงด้วยโปรเจกต์รวบยอด
ที่ใหญ่และซับซ้อนกว่านี้มาก: **ระบบจัดการสินค้าคงคลัง (Inventory Management System)** ใน Part 035

**[← กลับไป Part 014: โครงสร้างโปรแกรมแบบมีโมดูล](part-014-paragraphs-sections.md)** |
**[ไปยัง Part 016: ตาราง (Tables) และ OCCURS Clause →](part-016-tables-occurs.md)**
