# Part 085: 🎯 โปรเจกต์เฟส 5: ปรับปรุงระบบ Legacy ด้วยสถาปัตยกรรมสมัยใหม่ (ขั้นตอนที่ 841–850)

## คำนำของ Part นี้

เดินทางมาถึงจุดสำคัญอีกครั้ง! เฟส 5 ทั้งหมด (Part 071-084) ได้สอนเครื่องมือและเทคนิคสมัยใหม่จำนวนมาก
ที่ทำให้ COBOL อยู่ร่วมกับโลกเทคโนโลยีปี 2026 ได้อย่างสมบูรณ์: GnuCOBOL และ Ecosystem แบบ Open Source
(Part 071), การเชื่อมต่อกับภาษาอื่นผ่าน `CALL` (Part 072), การสร้าง REST API Wrapper (Part 073), การ
เชื่อมกับ Web Application (Part 074), ฐานข้อมูลสมัยใหม่ (Part 075), Docker (Part 076), แนวคิด Cloud
Modernization และ Strangler Fig Pattern (Part 077), CI/CD Pipeline (Part 078), Unit Testing (Part 079),
Git Workflow (Part 080), Clean Code/Refactoring (Part 081), Design Patterns (Part 082), Legacy System
Analysis (Part 083), และ Migration Strategies (Part 084)

Part นี้คือ **โปรเจกต์รวบยอดเฟส 5 (Phase 5 Milestone Project)** ซึ่งจะนำความรู้ทั้งหมดข้างต้นมาประกอบร่าง
เป็นงานจริงชิ้นเดียว: การ**ปรับปรุงระบบ Legacy ด้วยสถาปัตยกรรมสมัยใหม่แบบครบวงจร** เราจะสร้างระบบตัวอย่าง
ชื่อ **AR-MINI** (Accounts Receivable Mini System) ที่จำลองสถานการณ์งานจริงตั้งแต่ต้นจนจบ:

1. เริ่มจากระบบ Legacy ตัวอย่าง (`ARLEGACY.cob`) ที่เขียนด้วยสไตล์เดียวกับ `LEGACYORD` จาก Part 083
   (โปรแกรมเดียวจบ, ตัวแปรชื่อคลุมเครือ, ไม่มี Separation of Concerns)
2. บันทึก **Golden Master** (ผลลัพธ์ต้นแบบ) ก่อนแตะโค้ดใด ๆ ตามเทคนิค Part 084 ขั้นตอนที่ 838
3. **Refactor** ระบบให้สะอาดขึ้นตามหลัก Part 081 (Extract Subprogram, ตั้งชื่อค่าคงที่) และ Part 082
   (แยก Business Logic ออกจาก File I/O) — พร้อมพิสูจน์ว่าพฤติกรรมยังตรงกับ Golden Master ทุกประการ
4. เขียน **Unit Test** ครอบคลุมตามแนวทาง Part 079 รวมถึงกรณีขอบเขตที่จับบั๊กจริงได้
5. ห่อหุ้มด้วย **REST API** ตามแนวทาง Part 073 และทดสอบ End-to-End ด้วย `curl` จริง
6. สร้าง **CI Pipeline** ตามแนวทาง Part 078 ที่รวมทุกขั้นตอนเป็นอัตโนมัติ
7. ปิดท้ายด้วยแนวทาง **Git Workflow** (Part 080) สำหรับทีมที่ทำงานลักษณะนี้จริง

### หมายเหตุสำคัญเรื่องการทดสอบและสภาพแวดล้อม

**ทุกโปรแกรม ทุกสคริปต์ และทุกผลลัพธ์ในเอกสารนี้ผ่านการคอมไพล์และรันทดสอบจริง** ด้วย GnuCOBOL
(`cobc (GnuCOBOL) 4.0-early-dev.0`), Python 3 (stdlib เท่านั้น ไม่มี Framework ภายนอก), และ `curl`
ระบบนี้ใช้ `LINE SEQUENTIAL` ล้วน (ไม่ใช้ Indexed File) จึงคอมไพล์ได้ด้วย `cobc` มาตรฐานที่ติดตั้งจาก
ตัวจัดการแพ็กเกจทั่วไป ไม่ต้องพึ่งพา ISAM-enabled build แบบ Part 050 คำสั่งคอมไพล์และรันมาตรฐานที่ใช้
ตลอดทั้ง Part นี้คือ:

```bash
cobc -I copybooks -c src/ชื่อไฟล์.cob -o build/ชื่อไฟล์.o
cobc -I copybooks -x -o build/ชื่อโปรแกรม src/main.cob build/*.o
```

### โครงสร้างโปรเจกต์ทั้งหมด

```
ar-mini/
├── legacy/
│   └── arlegacy.cob         (ระบบเดิม "ก่อน" ปรับปรุง - Step 842)
├── copybooks/
│   └── AR-CONST.CPY         (ค่าคงที่ทางธุรกิจ - Step 844)
├── src/
│   ├── ardisc.cob           (Subprogram คำนวณ - Step 844)
│   ├── arledger.cob         (Subprogram บันทึกยอดคงเหลือ - Step 845)
│   └── arcalc.cob           (Orchestrator "หลัง" ปรับปรุง - Step 845)
├── tests/
│   └── testdisc.cob         (Unit Test - Step 846)
├── rest/
│   └── ar_server.py         (REST API Wrapper - Step 847)
├── find_dead_paragraphs.sh  (นำมาจาก Part 083 - Step 843/848)
└── ci.sh                    (CI Pipeline - Step 848)
```

---

## ขั้นตอนที่ 841: วางแผนโปรเจกต์และตัดสินใจเชิงสถาปัตยกรรม (ADR)

### ความต้องการของระบบ AR-MINI

ระบบ AR-MINI คำนวณใบแจ้งหนี้ 1 รายการต่อครั้ง (ราคาสินค้า, จำนวน, ระดับลูกค้า, ภูมิภาค) แล้วปรับปรุงยอด
คงเหลือสะสม (Running Balance) ของลูกค้า — เป็นระบบขนาดเล็กแต่มีองค์ประกอบครบถ้วนตามระบบ AR จริง
(การคำนวณส่วนลด/ภาษี + การบันทึกบัญชีลูกหนี้สะสม) เหมือนที่ Part 050 เคยสร้างไว้ในเฟส 3 แต่ในเฟส 5 นี้
เราจะทำให้มันเข้าถึงได้ผ่าน REST API และมีการทดสอบอัตโนมัติแบบครบวงจร

**ความต้องการ**:

| ข้อ | ความต้องการ |
|---|---|
| 1 | คำนวณส่วนลดแบบขั้นบันได (tiered) ตามระดับลูกค้า (Platinum/Gold/Regular) และยอดสั่งซื้อ |
| 2 | เพิ่มส่วนลดพิเศษ 2% สำหรับคำสั่งซื้อส่งออก (Export) โดยจำกัดส่วนลดรวมไม่เกิน 20% |
| 3 | คำนวณภาษี 7% จากยอดสุทธิหลังหักส่วนลด |
| 4 | สะสมยอดรวมลงในบัญชีลูกหนี้ (Running Balance) แบบต่อเนื่องข้ามการรันแต่ละครั้ง |
| 5 | เปิดให้เรียกใช้งานผ่าน REST API (ไม่ใช่แค่ Batch/Console เท่านั้น) |
| 6 | มี Unit Test ที่ครอบคลุมกรณีขอบเขตของกฎส่วนลด |
| 7 | มี CI Pipeline ที่คอมไพล์+ทดสอบอัตโนมัติทุกครั้งที่มีการแก้ไขโค้ด |

### ADR: การตัดสินใจเชิงสถาปัตยกรรมของโปรเจกต์นี้

ตามรูปแบบ Architecture Decision Record ที่ Part 050 แนะนำไว้ และตามกรอบการประเมินความเสี่ยงจาก Part 084
ขั้นตอนที่ 839 เราบันทึกการตัดสินใจหลักของโปรเจกต์นี้ไว้ดังนี้:

> **ADR-085-01: เลือกกลยุทธ์ Wrap/Encapsulate + Refactor (ไม่ใช่ Full Rewrite เป็น Java/C#)**
>
> **บริบท**: ระบบ AR-MINI มีขนาดเล็ก มีกฎธุรกิจที่ต้องคำนวณทศนิยมแม่นยำระดับสตางค์ (ตามที่ Part 084
> ขั้นตอนที่ 836-837 พิสูจน์ว่า `double` ของภาษาอื่นมีความเสี่ยง) และต้องการเห็นผลลัพธ์เร็ว
>
> **การตัดสินใจ**: คง Business Logic ไว้เป็นภาษา COBOL ทั้งหมด (ได้ประโยชน์จาก Fixed-Point Decimal ที่
> แม่นยำ 100% ตามธรรมชาติของภาษา) แต่ **Refactor โครงสร้างภายใน** ให้แยก Business Logic ออกจาก File I/O
> (Part 082 แนวคิด Repository/Separation of Concerns) แล้วห่อหุ้มด้วย REST API (Part 073) แทนการเขียน
> ใหม่เป็นภาษาอื่น
>
> **ผลที่ตามมา**: ทีม Web/Mobile ที่ไม่รู้จัก COBOL เลยสามารถเรียกใช้ระบบผ่าน HTTP/JSON ได้ทันที ในขณะที่
> แกนคำนวณทางการเงินยังคงความแม่นยำแบบ COBOL ไว้ครบถ้วน — ตรงกับที่ Part 084 ขั้นตอนที่ 834 อธิบายไว้ว่า
> เป็นกลยุทธ์ที่นิยมที่สุดในทางปฏิบัติจริง

> **ADR-085-02: ใช้ Golden Master Testing เป็นตาข่ายนิรภัยหลักตลอดการ Refactor**
>
> **บริบท**: การ Refactor ย่อหน้าขนาดใหญ่ให้เป็น Subprogram หลายตัวมีความเสี่ยงที่จะเปลี่ยนพฤติกรรมโดย
> ไม่ตั้งใจ
>
> **การตัดสินใจ**: บันทึกผลลัพธ์ของ `ARLEGACY.cob` ไว้ก่อนแก้โค้ดใด ๆ (ขั้นตอนที่ 842) แล้วเปรียบเทียบ
> ผลลัพธ์แบบไบต์ต่อไบต์ทุกครั้งหลัง Refactor (ขั้นตอนที่ 844-845) ตามเทคนิคที่ Part 084 ขั้นตอนที่ 838
> แนะนำไว้

### ข้อควรระวัง

- ADR ทั้งสองข้อนี้ถูกเขียนไว้**ก่อน**ลงมือเขียนโค้ดจริง — นี่คือวินัยสำคัญของงานสถาปัตยกรรมที่ดี: การ
  ตัดสินใจเชิงโครงสร้างควรถูกไตร่ตรองและบันทึกไว้ล่วงหน้า ไม่ใช่ค่อยคิดย้อนหลังหลังเขียนโค้ดเสร็จแล้ว
- โปรเจกต์นี้จงใจใช้ขนาดเล็กเพื่อให้เห็นภาพรวมกระบวนการทั้งหมดได้ภายใน 10 ขั้นตอน — ระบบ Legacy จริงใน
  องค์กรมักมีขนาดใหญ่กว่านี้มาก แต่**หลักการและลำดับขั้นตอนเดียวกันนี้ใช้ได้กับระบบทุกขนาด**

### แบบฝึกหัดที่ 841.1

**โจทย์**: จงอธิบายว่าทำไม ADR-085-02 (Golden Master Testing) ถึงต้องถูกตัดสินใจ**ก่อน**เริ่ม Refactor
ใน ADR-085-01 ไม่ใช่ตัดสินใจทีหลังหลังจาก Refactor เสร็จแล้ว

**เฉลยแนวทาง**: เพราะ Golden Master ต้องถูกบันทึกจาก**พฤติกรรมของระบบเดิมก่อนถูกแก้ไข** หากเริ่ม Refactor
ไปแล้วจึงค่อยคิดจะบันทึก Golden Master ทีหลัง เราจะไม่มีทาง "ย้อนกลับไปหาความจริงดั้งเดิม" ได้อีก เพราะ
โค้ดที่ถูกแก้ไขแล้วอาจมีพฤติกรรมเปลี่ยนไปแล้วโดยไม่รู้ตัว การบันทึก Golden Master จึงต้องเป็นขั้นตอน
**แรกสุด** ก่อนแตะโค้ดใด ๆ เสมอ ตรงกับลำดับขั้นตอนที่ 842 (บันทึก Golden Master) มาก่อนขั้นตอนที่ 844
(เริ่ม Refactor) ในโปรเจกต์นี้

---

## ขั้นตอนที่ 842: Step 1 — ระบบ Legacy ตั้งต้นและการบันทึก Golden Master

### แนวคิด: ต้องมี "ของเดิม" ก่อนจะปรับปรุงอะไรได้

ตามหลักการจาก Part 083/084 เราต้องมีระบบ Legacy ตั้งต้นที่ทำงานได้จริงก่อน แล้วจึงบันทึกพฤติกรรมของมันไว้
เป็น Golden Master ก่อนแตะโค้ดใด ๆ

### ซอร์สโค้ดฉบับเต็ม: ARLEGACY.cob

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ARLEGACY.
       AUTHOR. UNKNOWN-LEGACY-TEAM.
      *> The "before" picture: one invoice calculation + ledger
      *> program, written the way a lot of real legacy AR code
      *> actually looks - everything in one paragraph, magic
      *> numbers, cryptic working names, no separation between
      *> "calculate" and "persist". This is the system Part 085
      *> will modernize step by step.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT BAL-FILE ASSIGN TO "BALANCE.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-BFS.

       DATA DIVISION.
       FILE SECTION.
       FD  BAL-FILE.
       01  BAL-LINE                PIC 9(9)V99.

       WORKING-STORAGE SECTION.
       01  WS-REQ-LINE             PIC X(12).
       01  WS-REQ-FIELDS REDEFINES WS-REQ-LINE.
           05  X1                  PIC 9(5)V99.
           05  X2                  PIC 9(3).
           05  X3                  PIC X(1).
           05  X4                  PIC X(1).
       01  WS-BFS                  PIC XX.
       01  TMP                     PIC 9(7)V99.
       01  TMP2                    PIC 9V999.
       01  TMP3                    PIC 9(7)V99.
       01  TMP4                    PIC 9(7)V99.
       01  TMP5                    PIC 9(7)V99.
       01  TMP6                    PIC 9(7)V99.
       01  OLDBAL                  PIC 9(9)V99 VALUE 0.
       01  NEWBAL                  PIC 9(9)V99 VALUE 0.
       01  WS-EOF                  PIC X VALUE "N".

       PROCEDURE DIVISION.
       MAIN-PARA.
           ACCEPT WS-REQ-LINE FROM CONSOLE

      *> subtotal
           COMPUTE TMP ROUNDED = X1 * X2

      *> tiered discount - magic numbers, nobody left a comment
      *> saying where 10000/5000/0.15/0.10/... came from
           IF X3 = "P"
               IF TMP >= 10000.00
                   MOVE 0.150 TO TMP2
               ELSE
                   IF TMP >= 5000.00
                       MOVE 0.100 TO TMP2
                   ELSE
                       MOVE 0.050 TO TMP2
                   END-IF
               END-IF
           ELSE
               IF X3 = "G"
                   IF TMP >= 10000.00
                       MOVE 0.100 TO TMP2
                   ELSE
                       IF TMP >= 5000.00
                           MOVE 0.070 TO TMP2
                       ELSE
                           MOVE 0.030 TO TMP2
                       END-IF
                   END-IF
               ELSE
                   IF TMP >= 10000.00
                       MOVE 0.050 TO TMP2
                   ELSE
                       IF TMP >= 5000.00
                           MOVE 0.020 TO TMP2
                       ELSE
                           MOVE 0.000 TO TMP2
                       END-IF
                   END-IF
               END-IF
           END-IF

           IF X4 = "X"
               ADD 0.020 TO TMP2
               IF TMP2 > 0.200
                   MOVE 0.200 TO TMP2
               END-IF
           END-IF

           COMPUTE TMP3 ROUNDED = TMP * TMP2
           COMPUTE TMP4 ROUNDED = TMP - TMP3
           COMPUTE TMP5 ROUNDED = TMP4 * 0.070
           COMPUTE TMP6 ROUNDED = TMP4 + TMP5

      *> read current ledger balance, if any
           OPEN INPUT BAL-FILE
           IF WS-BFS = "35"
               MOVE 0 TO OLDBAL
           ELSE
               READ BAL-FILE
                   AT END MOVE 0 TO OLDBAL
                   NOT AT END MOVE BAL-LINE TO OLDBAL
               END-READ
           END-IF
           CLOSE BAL-FILE

           ADD OLDBAL TO TMP6 GIVING NEWBAL

           OPEN OUTPUT BAL-FILE
           MOVE NEWBAL TO BAL-LINE
           WRITE BAL-LINE
           CLOSE BAL-FILE

           DISPLAY TMP "|" TMP2 "|" TMP3 "|" TMP4 "|" TMP5 "|"
               TMP6 "|" NEWBAL
           STOP RUN.
```

สังเกตว่านี่คือโปรแกรมสไตล์เดียวกับ `LEGACYORD` จาก Part 083 ทุกประการ: ตัวแปรชื่อ `TMP`, `X1`-`X4` ที่
ไม่สื่อความหมาย, Magic Numbers (`10000.00`, `0.150` ฯลฯ), และทุกอย่างอยู่ใน `MAIN-PARA` เพียงย่อหน้าเดียว
โดยไม่มีการแยก "คำนวณ" ออกจาก "บันทึกไฟล์" เลย

### รูปแบบข้อมูลนำเข้า (Request Line)

โปรแกรมอ่าน 1 บรรทัดจาก stdin ความยาว 12 ตัวอักษรพอดี: ราคาสินค้า (`PIC 9(5)V99`, 7 หลัก), จำนวน
(`PIC 9(3)`, 3 หลัก), ระดับลูกค้า (1 ตัวอักษร: P/G/R), ภูมิภาค (1 ตัวอักษร: X สำหรับ Export)

### คอมไพล์และบันทึก Golden Master

```bash
cobc -x -o arlegacy legacy/arlegacy.cob
rm -f BALANCE.DAT

echo "0001999003PR" | ./arlegacy
echo "0002000004PX" | ./arlegacy
```

**ผลลัพธ์จริงที่ได้ (นี่คือ Golden Master ที่จะใช้เปรียบเทียบตลอดทั้ง Part นี้):**

```
0000059.97|0.050|0000003.00|0000056.97|0000003.99|0000060.96|000000060.96
0000080.00|0.070|0000005.60|0000074.40|0000005.21|0000079.61|000000140.57
```

และไฟล์ `BALANCE.DAT` สุดท้าย:

```bash
cat BALANCE.DAT
```

```
00000014057
```

### บันทึก Golden Master ไว้เป็นไฟล์อ้างอิง

```bash
cat > golden_master.txt << 'EOF'
INPUT: 0001999003PR
OUTPUT: 0000059.97|0.050|0000003.00|0000056.97|0000003.99|0000060.96|000000060.96

INPUT: 0002000004PX (run after the above, same BALANCE.DAT)
OUTPUT: 0000080.00|0.070|0000005.60|0000074.40|0000005.21|0000079.61|000000140.57

FINAL BALANCE.DAT CONTENT: 00000014057
EOF
```

### ข้อควรระวัง

- สังเกตว่า Golden Master ต้องระบุ**ลำดับการรัน**ให้ชัดเจนด้วย (run 2 ต้องรันต่อจาก run 1 โดยไม่ลบ
  `BALANCE.DAT` ระหว่างทาง) เพราะระบบนี้มีสถานะที่คงอยู่ข้ามการรัน (persistent state) เหมือนที่ Part 050
  เคยเน้นย้ำไว้ในขั้นตอนที่ 494
- ห้ามแก้ไข `ARLEGACY.cob` อีกหลังจากขั้นตอนนี้ — มันจะถูกเก็บไว้เป็นข้อมูลอ้างอิง (reference) ในโฟลเดอร์
  `legacy/` ตลอดไป ส่วนงาน Refactor ทั้งหมดจะเกิดขึ้นในโฟลเดอร์ `src/` แยกต่างหาก

### แบบฝึกหัดที่ 842.1

**โจทย์**: จงอธิบายว่าทำไม Golden Master ในขั้นตอนนี้ต้องมี 2 กรณีทดสอบ (ไม่ใช่แค่ 1 กรณี) จึงจะเพียงพอ
สำหรับการตรวจสอบ Refactor ในขั้นตอนที่ 845

**เฉลยแนวทาง**: เพราะกรณีที่ 1 (`OR-TIER = "P"`, ไม่ Export) ทดสอบแค่เส้นทางคำนวณพื้นฐาน แต่ไม่ทดสอบกฎ
โบนัสส่งออก ในขณะที่กรณีที่ 2 (`OR-TIER = "P"`, Export) ทดสอบทั้งกฎพื้นฐานและกฎโบนัสส่งออกร่วมกัน นอกจาก
นี้กรณีที่ 2 ยังทดสอบว่ายอดคงเหลือสะสม (`BALANCE.DAT`) ทำงานถูกต้องข้ามการรันหลายครั้งด้วย — ยิ่งมีกรณี
ทดสอบที่ครอบคลุมเส้นทางต่าง ๆ ของโค้ดมากเท่าไร Golden Master ก็ยิ่งน่าเชื่อถือมากขึ้นเท่านั้น (แม้ในทาง
ปฏิบัติจริงควรมีมากกว่า 2 กรณีสำหรับระบบที่ซับซ้อนกว่านี้ ตามที่ Unit Test ในขั้นตอนที่ 846 จะครอบคลุม
เพิ่มเติม)

---

## ขั้นตอนที่ 843: ใช้เทคนิค Legacy Analysis จาก Part 083 วิเคราะห์ ARLEGACY.cob

### นำเครื่องมือจาก Part 083 มาใช้ซ้ำ

ก่อนเริ่ม Refactor เราควรวิเคราะห์ `ARLEGACY.cob` ด้วยเทคนิคเดียวกับที่ Part 083 สอนไว้ทั้งหมด — นี่คือ
ตัวอย่างที่ดีว่าทักษะจาก Part หนึ่งสามารถนำไปใช้ซ้ำได้จริงในโปรเจกต์ถัดไป ไม่ใช่แค่ทฤษฎีที่เรียนแล้วลืม

### รัน find_dead_paragraphs.sh (จาก Part 083 ขั้นตอนที่ 825) กับ ARLEGACY.cob

```bash
./find_dead_paragraphs.sh legacy/arlegacy.cob
```

**ผลลัพธ์จริงที่ได้:**

```
Paragraphs found:
  - FILE-CONTROL
  - MAIN-PARA

Paragraphs with NO incoming PERFORM/GO TO (dead-code candidates):
  -> FILE-CONTROL (0 references)
  -> MAIN-PARA (0 references)
```

### การตีความผลลัพธ์ — "ไม่มี Dead Code" ไม่ได้แปลว่า "โค้ดดี"

ต่างจาก `LEGACYORD` ใน Part 083 ที่พบ Dead Code จริง (`OLD-CALC-DISCOUNT-V1`) ผลลัพธ์นี้แสดงให้เห็นว่า
`ARLEGACY.cob` **ไม่มี Dead Code เลย** — แต่นี่ไม่ใช่ข่าวดีเสมอไป! เหตุผลที่ไม่พบ Dead Code คือ**ทุกอย่าง
ถูกยัดรวมอยู่ใน `MAIN-PARA` เพียงย่อหน้าเดียว** ไม่มีการแบ่งย่อหน้าเลย จึงไม่มี "ย่อหน้าที่ไม่ถูกเรียก"
ให้พบ — นี่คือ Code Smell ที่ต่างออกไป: **"God Paragraph"** (ย่อหน้าเดียวทำทุกอย่าง) ซึ่งเป็นปัญหาที่
Part 081 (Clean Code) สอนให้แก้ด้วยเทคนิค Extract Method/Extract Subprogram

### สกัดกฎธุรกิจด้วยเทคนิคจาก Part 083 ขั้นตอนที่ 827

ไล่อ่าน `IF X3 = "P" ... ELSE IF X3 = "G" ... ELSE ...` ได้ Decision Table เดียวกันทุกประการกับที่
Part 083 สกัดจาก `LEGACYORD` (เพราะ `ARLEGACY` ใช้กฎธุรกิจเดียวกัน เพียงแค่เปลี่ยนบริบทจาก "รายงานแบบ
Batch" เป็น "คำนวณทีละใบแจ้งหนี้"):

| Tier | เงื่อนไขยอดสั่งซื้อ | ส่วนลดพื้นฐาน | ส่วนลดถ้า Export |
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

พร้อมกฎเสริม: ภาษี 7% คงที่จากยอดสุทธิ, ส่วนลดรวมสูงสุด 20%

Decision Table นี้คือแผนที่ที่เราจะใช้เขียน `ARDISC.cob` ใหม่ในขั้นตอนที่ 844 — และจะใช้เป็นฐานสำหรับ
เขียนชื่อค่าคงที่ที่สื่อความหมายใน Copybook ด้วย

### ข้อควรระวัง

- แม้ระบบนี้จะเล็กพอที่จะ "อ่านแล้วเข้าใจได้เลย" แต่การรันเครื่องมือวิเคราะห์อัตโนมัติยังคงมีประโยชน์เสมอ
  — มันช่วยยืนยันความเข้าใจของมนุษย์ด้วยหลักฐานที่ตรวจสอบได้ซ้ำ (reproducible) แทนที่จะพึ่งพาความมั่นใจ
  ส่วนตัวเพียงอย่างเดียว
- "ไม่มีปัญหาที่เครื่องมือตรวจพบ" ไม่เท่ากับ "ไม่มีปัญหา" เสมอ — ต้องใช้วิจารณญาณของมนุษย์ควบคู่กันเสมอ
  ตามที่ Part 083 ขั้นตอนที่ 825 เน้นย้ำไว้

### แบบฝึกหัดที่ 843.1

**โจทย์**: จงอธิบายว่าทำไม "God Paragraph" (ทุกอย่างอยู่ในย่อหน้าเดียว) ถึงทำให้เขียน Unit Test ยากกว่า
ระบบที่แยกเป็นหลายย่อหน้า/Subprogram

**เฉลยแนวทาง**: เพราะ Unit Test ที่ดีต้องทดสอบ**หน่วยงานเล็ก ๆ แยกจากกัน** (เช่นทดสอบแค่การคำนวณส่วนลด
โดยไม่ต้องยุ่งกับไฟล์) แต่ใน `ARLEGACY.cob` การคำนวณส่วนลดถูกผูกติดกับการอ่าน/เขียนไฟล์ `BALANCE.DAT`
ไว้ในย่อหน้าเดียวกันอย่างแยกไม่ออก ทำให้การทดสอบการคำนวณอย่างเดียว (โดยไม่ต้องมีไฟล์บนดิสก์จริง) เป็นไป
ไม่ได้เลยถ้าไม่ Refactor ก่อน — นี่คือเหตุผลสำคัญที่ขั้นตอนที่ 844 จะแยก `ARDISC.cob` (คำนวณล้วน ไม่มี
ไฟล์) ออกจาก `ARLEDGER.cob` (จัดการไฟล์ล้วน) ก่อนที่จะเขียน Unit Test ในขั้นตอนที่ 846

---

## ขั้นตอนที่ 844: Refactor ขั้นที่ 1 — แยกตรรกะคำนวณเป็น ARDISC.cob

### แนวคิด: Extract Subprogram (Part 081) + Pure Function (Part 082)

ขั้นตอนแรกของการ Refactor คือแยก**ตรรกะคำนวณล้วน ๆ** (ไม่มี File I/O, ไม่มี Side Effect) ออกมาเป็น
Subprogram ต่างหาก ตามเทคนิค Extract Subprogram ที่ Part 081 สอนไว้ นี่คือ Subprogram แบบ "Pure
Function" ตามแนวคิด Part 082: รับค่าเข้า คำนวณ คืนค่าออก โดยไม่แก้ไขสถานะอะไรนอกเหนือจากพารามิเตอร์ที่
รับมา — คุณสมบัตินี้ทำให้มันทดสอบง่ายที่สุดเท่าที่จะเป็นไปได้

### สร้าง Copybook ค่าคงที่ก่อน: AR-CONST.CPY

```cobol
      *> Copybook: AR-CONST.CPY
      *> Named constants for the AR-MINI invoice calculation. These
      *> replace the unexplained magic numbers (10000.00, 0.150,
      *> 0.070, ...) that ARLEGACY.cob scattered through nested IFs.
       78  TIER-HIGH-THRESHOLD     VALUE 10000.00.
       78  TIER-MID-THRESHOLD      VALUE 5000.00.

       78  PLATINUM-RATE-HIGH      VALUE 0.150.
       78  PLATINUM-RATE-MID       VALUE 0.100.
       78  PLATINUM-RATE-LOW       VALUE 0.050.

       78  GOLD-RATE-HIGH          VALUE 0.100.
       78  GOLD-RATE-MID           VALUE 0.070.
       78  GOLD-RATE-LOW           VALUE 0.030.

       78  REGULAR-RATE-HIGH       VALUE 0.050.
       78  REGULAR-RATE-MID        VALUE 0.020.
       78  REGULAR-RATE-LOW        VALUE 0.000.

       78  EXPORT-BONUS-PCT        VALUE 0.020.
       78  MAX-DISCOUNT-PCT        VALUE 0.200.

       78  SALES-TAX-RATE          VALUE 0.070.
```

ตามเทคนิค Level 78 ที่เรียนมาใน Part 033 ขั้นตอนที่ 327 — ทุกค่าที่เคยเป็น "Magic Number" ใน
`ARLEGACY.cob` ตอนนี้มีชื่อที่สื่อความหมายแล้ว

### ซอร์สโค้ดฉบับเต็ม: ARDISC.cob

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ARDISC.
       AUTHOR. COBOL-COURSE.
      *> Refactored discount/tax calculation engine. Pulled out of
      *> ARLEGACY.cob's single monolithic paragraph (Part 081 Extract
      *> Subprogram refactoring) so it can be called, and unit
      *> tested, on its own - with zero file I/O and zero side
      *> effects. Same formula, same rounding points, same output
      *> as the legacy program: this is a pure behavior-preserving
      *> refactor, proven against the ARLEGACY golden master.

       ENVIRONMENT DIVISION.
       CONFIGURATION SECTION.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
           COPY "AR-CONST.CPY".

       LINKAGE SECTION.
       01  LK-PRICE                PIC 9(5)V99.
       01  LK-QTY                  PIC 9(3).
       01  LK-TIER                 PIC X(1).
       01  LK-REGION               PIC X(1).
       01  LK-SUBTOTAL             PIC 9(7)V99.
       01  LK-DISC-PCT             PIC 9V999.
       01  LK-DISC-AMT             PIC 9(7)V99.
       01  LK-NET                  PIC 9(7)V99.
       01  LK-TAX-AMT              PIC 9(7)V99.
       01  LK-TOTAL                PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-PRICE, LK-QTY, LK-TIER, LK-REGION,
                                 LK-SUBTOTAL, LK-DISC-PCT, LK-DISC-AMT,
                                 LK-NET, LK-TAX-AMT, LK-TOTAL.
       CALCULATE-INVOICE.
           COMPUTE LK-SUBTOTAL ROUNDED = LK-PRICE * LK-QTY
           PERFORM DETERMINE-DISCOUNT-RATE
           PERFORM APPLY-EXPORT-BONUS
           PERFORM COMPUTE-AMOUNTS
           GOBACK.

      *> Same three tiers as the legacy code, now with named
      *> constants and one flat EVALUATE instead of three levels of
      *> nested IF/ELSE - directly readable as a decision table.
       DETERMINE-DISCOUNT-RATE.
           EVALUATE TRUE
               WHEN LK-TIER = "P" AND LK-SUBTOTAL >= TIER-HIGH-THRESHOLD
                   MOVE PLATINUM-RATE-HIGH TO LK-DISC-PCT
               WHEN LK-TIER = "P" AND LK-SUBTOTAL >= TIER-MID-THRESHOLD
                   MOVE PLATINUM-RATE-MID TO LK-DISC-PCT
               WHEN LK-TIER = "P"
                   MOVE PLATINUM-RATE-LOW TO LK-DISC-PCT
               WHEN LK-TIER = "G" AND LK-SUBTOTAL >= TIER-HIGH-THRESHOLD
                   MOVE GOLD-RATE-HIGH TO LK-DISC-PCT
               WHEN LK-TIER = "G" AND LK-SUBTOTAL >= TIER-MID-THRESHOLD
                   MOVE GOLD-RATE-MID TO LK-DISC-PCT
               WHEN LK-TIER = "G"
                   MOVE GOLD-RATE-LOW TO LK-DISC-PCT
               WHEN LK-SUBTOTAL >= TIER-HIGH-THRESHOLD
                   MOVE REGULAR-RATE-HIGH TO LK-DISC-PCT
               WHEN LK-SUBTOTAL >= TIER-MID-THRESHOLD
                   MOVE REGULAR-RATE-MID TO LK-DISC-PCT
               WHEN OTHER
                   MOVE REGULAR-RATE-LOW TO LK-DISC-PCT
           END-EVALUATE.

       APPLY-EXPORT-BONUS.
           IF LK-REGION = "X"
               ADD EXPORT-BONUS-PCT TO LK-DISC-PCT
               IF LK-DISC-PCT > MAX-DISCOUNT-PCT
                   MOVE MAX-DISCOUNT-PCT TO LK-DISC-PCT
               END-IF
           END-IF.

       COMPUTE-AMOUNTS.
           COMPUTE LK-DISC-AMT ROUNDED = LK-SUBTOTAL * LK-DISC-PCT
           COMPUTE LK-NET ROUNDED = LK-SUBTOTAL - LK-DISC-AMT
           COMPUTE LK-TAX-AMT ROUNDED = LK-NET * SALES-TAX-RATE
           COMPUTE LK-TOTAL ROUNDED = LK-NET + LK-TAX-AMT.
```

### เปรียบเทียบก่อน/หลังการ Refactor

| ด้าน | ARLEGACY.cob (ก่อน) | ARDISC.cob (หลัง) |
|---|---|---|
| จำนวนย่อหน้า | 1 (`MAIN-PARA` ทำทุกอย่าง) | 4 (แต่ละย่อหน้าทำหน้าที่เดียว) |
| ชื่อตัวแปร | `TMP`, `TMP2`...`TMP6`, `X1`-`X4` | `LK-SUBTOTAL`, `LK-DISC-PCT` ฯลฯ (สื่อความหมาย) |
| Magic Numbers | มี (`10000.00`, `0.150` ฝังตรง ๆ) | ไม่มี (ใช้ชื่อค่าคงที่จาก Copybook) |
| โครงสร้างเงื่อนไข | `IF`/`ELSE` ซ้อน 3 ชั้น | `EVALUATE TRUE` แบบเรียบ (Decision Table ที่อ่านตรง ๆ ได้) |
| File I/O ปนอยู่หรือไม่ | ปนอยู่ (`OPEN`/`READ`/`WRITE` ใน `MAIN-PARA` เดียวกัน) | ไม่มีเลย (Pure Function, พิสูจน์จาก `PROCEDURE DIVISION` ไม่มี `SELECT`/`FD`) |

### พิสูจน์ Regression: เรียก ARDISC.cob โดยตรงด้วยข้อมูลเดียวกับ Golden Master

สร้างโปรแกรมทดสอบชั่วคราวเรียก `ARDISC` ด้วยอินพุตเดียวกับ Golden Master ในขั้นตอนที่ 842 (ยังไม่ต้องมี
`ARLEDGER`/`ARCALC` เลยด้วยซ้ำ — เพราะ `ARDISC` ไม่ยุ่งกับไฟล์):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DRV.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  P PIC 9(5)V99.
       01  Q PIC 9(3).
       01  T PIC X.
       01  R PIC X.
       01  SUB PIC 9(7)V99.
       01  DP  PIC 9V999.
       01  DA  PIC 9(7)V99.
       01  NT  PIC 9(7)V99.
       01  TX  PIC 9(7)V99.
       01  TT  PIC 9(7)V99.
       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 19.99 TO P
           MOVE 3 TO Q
           MOVE "P" TO T
           MOVE "R" TO R
           CALL "ARDISC" USING P, Q, T, R, SUB, DP, DA, NT, TX, TT
           DISPLAY "CASE1: " SUB "|" DP "|" DA "|" NT "|" TX "|" TT

           MOVE 20.00 TO P
           MOVE 4 TO Q
           MOVE "P" TO T
           MOVE "X" TO R
           CALL "ARDISC" USING P, Q, T, R, SUB, DP, DA, NT, TX, TT
           DISPLAY "CASE2: " SUB "|" DP "|" DA "|" NT "|" TX "|" TT
           STOP RUN.
```

```bash
cobc -I copybooks -c src/ardisc.cob -o build/ardisc.o
cobc -I copybooks -x -o drv drv.cob build/ardisc.o
./drv
```

**ผลลัพธ์จริงที่ได้:**

```
CASE1: 0000059.97|0.050|0000003.00|0000056.97|0000003.99|0000060.96
CASE2: 0000080.00|0.070|0000005.60|0000074.40|0000005.21|0000079.61
```

**เปรียบเทียบกับ Golden Master จากขั้นตอนที่ 842**: `0000059.97|0.050|0000003.00|0000056.97|
0000003.99|0000060.96` (ส่วนคำนวณ ไม่รวมยอดคงเหลือสะสม) — **ตรงกันทุกตัวอักษร** ยืนยันว่าการแยก Logic
ออกมาเป็น `ARDISC.cob` เป็น Refactor ที่ **คงพฤติกรรมเดิมไว้ครบถ้วน 100%** (Behavior-Preserving
Refactoring)

### ข้อควรระวัง

- โปรแกรมทดสอบชั่วคราว (`DRV.cob`) ในขั้นตอนนี้เป็นแค่เครื่องมือช่วยยืนยันระหว่างการ Refactor ไม่ใช่
  Unit Test ที่เป็นทางการ — Unit Test ที่แท้จริงพร้อมชุดกรณีทดสอบที่ครอบคลุมกว่านี้จะสร้างในขั้นตอนที่ 846
- สังเกตว่า `LK-DISC-PCT` ยังคงเป็น `PIC 9V999` เหมือนเดิมทุกประการ (ไม่ได้เปลี่ยนเป็น `BigDecimal` หรือ
  ชนิดข้อมูลอื่น) — ตรงกับการตัดสินใจใน ADR-085-01 ที่ Part นี้เลือกคง COBOL ไว้เป็นแกนคำนวณ ไม่ได้
  Migrate ไปภาษาอื่น

### แบบฝึกหัดที่ 844.1

**โจทย์**: จงอธิบายว่าทำไมการเปลี่ยนจาก `IF`/`ELSE` ซ้อน 3 ชั้นเป็น `EVALUATE TRUE` แบบเรียบ (ใน
`DETERMINE-DISCOUNT-RATE`) ถึงไม่เปลี่ยนพฤติกรรมของโปรแกรม ทั้งที่โครงสร้างโค้ดดูต่างกันมาก

**เฉลยแนวทาง**: เพราะ `EVALUATE TRUE` ทำงานโดยตรวจสอบเงื่อนไขแต่ละ `WHEN` **ตามลำดับจากบนลงล่าง** และ
หยุดที่เงื่อนไขแรกที่เป็นจริง เหมือนกับ `IF`/`ELSE IF`/`ELSE` ซ้อนกันทุกประการ เพียงแต่เขียนในรูปแบบที่
แบนราบกว่า (ไม่ซ้อนหลายชั้น) — ตราบใดที่ลำดับเงื่อนไขใน `WHEN` แต่ละบรรทัดตรงกับลำดับการตรวจสอบเดิมใน
`IF`/`ELSE` (เช่น ตรวจ `LK-TIER = "P" AND ... >= TIER-HIGH-THRESHOLD` ก่อนตรวจ `LK-TIER = "P" AND ...
>= TIER-MID-THRESHOLD` เสมอ) ผลลัพธ์จะเหมือนกันทุกกรณี การทดสอบ Regression ในขั้นตอนนี้คือหลักฐานที่
ยืนยันว่าการแปลงโครงสร้างนี้ถูกต้องจริง ไม่ใช่แค่การคาดเดา

---

## ขั้นตอนที่ 845: Refactor ขั้นที่ 2 — แยก ARLEDGER.cob และประกอบ ARCALC.cob

### แนวคิด: Repository Pattern (Part 082) สำหรับ File I/O

ขั้นตอนถัดไปคือแยกส่วนที่เหลือ (การอ่าน/เขียนยอดคงเหลือ) ออกเป็น Subprogram ต่างหาก ตามแนวคิด
**Repository** ที่ Part 082 สอนไว้: มีจุดเดียวในระบบที่รับผิดชอบการอ่าน/เขียนข้อมูลถาวร (persistent
data) ทำให้ Business Logic ส่วนอื่นไม่ต้องรู้รายละเอียดว่าข้อมูลถูกเก็บไว้ที่ไหน/อย่างไร

### ซอร์สโค้ดฉบับเต็ม: ARLEDGER.cob

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ARLEDGER.
       AUTHOR. COBOL-COURSE.
      *> Refactored ledger update. Pulled out of ARLEGACY.cob's
      *> monolithic paragraph so that file I/O (which needs a real
      *> filesystem and cannot easily be unit tested) is isolated
      *> away from the pure calculation in ARDISC.cob (which can).
      *> This is the "Repository" idea from Part 082 applied to a
      *> single flat file: one place responsible for reading and
      *> writing the customer balance.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT BAL-FILE ASSIGN TO "BALANCE.DAT"
               ORGANIZATION IS LINE SEQUENTIAL
               FILE STATUS IS WS-BAL-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  BAL-FILE.
       01  BAL-RECORD               PIC 9(9)V99.

       WORKING-STORAGE SECTION.
       01  WS-BAL-STATUS            PIC XX.
       01  WS-OLD-BALANCE           PIC 9(9)V99 VALUE 0.

       LINKAGE SECTION.
       01  LK-AMOUNT                PIC 9(7)V99.
       01  LK-NEW-BALANCE           PIC 9(9)V99.

       PROCEDURE DIVISION USING LK-AMOUNT, LK-NEW-BALANCE.
       APPLY-TO-LEDGER.
           PERFORM READ-CURRENT-BALANCE
           ADD WS-OLD-BALANCE TO LK-AMOUNT GIVING LK-NEW-BALANCE
           PERFORM WRITE-NEW-BALANCE
           GOBACK.

       READ-CURRENT-BALANCE.
           MOVE 0 TO WS-OLD-BALANCE
           OPEN INPUT BAL-FILE
           IF WS-BAL-STATUS = "00"
               READ BAL-FILE
                   AT END
                       MOVE 0 TO WS-OLD-BALANCE
                   NOT AT END
                       MOVE BAL-RECORD TO WS-OLD-BALANCE
               END-READ
               CLOSE BAL-FILE
           END-IF.

       WRITE-NEW-BALANCE.
           OPEN OUTPUT BAL-FILE
           MOVE LK-NEW-BALANCE TO BAL-RECORD
           WRITE BAL-RECORD
           CLOSE BAL-FILE.
```

### ซอร์สโค้ดฉบับเต็ม: ARCALC.cob (Orchestrator แทนที่ ARLEGACY.cob)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ARCALC.
       AUTHOR. COBOL-COURSE.
      *> Refactored orchestrator, replacing ARLEGACY.cob. Reads one
      *> fixed-width request line from stdin, delegates the actual
      *> work to two focused subprograms (ARDISC for calculation,
      *> ARLEDGER for persistence), and prints the same pipe
      *> delimited response line the legacy program printed - so
      *> everything calling this program (including the REST
      *> wrapper in Step 847) sees no difference in behavior.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-REQUEST-LINE          PIC X(12).
       01  WS-REQUEST-FIELDS REDEFINES WS-REQUEST-LINE.
           05  WS-REQ-PRICE         PIC 9(5)V99.
           05  WS-REQ-QTY           PIC 9(3).
           05  WS-REQ-TIER          PIC X(1).
           05  WS-REQ-REGION        PIC X(1).

       01  WS-SUBTOTAL              PIC 9(7)V99.
       01  WS-DISC-PCT              PIC 9V999.
       01  WS-DISC-AMT              PIC 9(7)V99.
       01  WS-NET                   PIC 9(7)V99.
       01  WS-TAX-AMT               PIC 9(7)V99.
       01  WS-TOTAL                 PIC 9(7)V99.
       01  WS-NEW-BALANCE           PIC 9(9)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           ACCEPT WS-REQUEST-LINE FROM CONSOLE

           CALL "ARDISC" USING WS-REQ-PRICE, WS-REQ-QTY, WS-REQ-TIER,
               WS-REQ-REGION, WS-SUBTOTAL, WS-DISC-PCT, WS-DISC-AMT,
               WS-NET, WS-TAX-AMT, WS-TOTAL
           END-CALL

           CALL "ARLEDGER" USING WS-TOTAL, WS-NEW-BALANCE
           END-CALL

           DISPLAY WS-SUBTOTAL "|" WS-DISC-PCT "|" WS-DISC-AMT "|"
               WS-NET "|" WS-TAX-AMT "|" WS-TOTAL "|" WS-NEW-BALANCE

           STOP RUN.
```

สังเกตว่า `ARCALC.cob` สั้นลงมากเมื่อเทียบกับ `ARLEGACY.cob` — มันแค่ **"สั่งการ" (orchestrate)** ให้
`ARDISC` คำนวณและ `ARLEDGER` บันทึก โดยไม่รู้รายละเอียดภายในของทั้งสองเลย ตรงกับหลักการ **Single
Responsibility Principle** ที่ Part 081/082 สอนไว้

### สถาปัตยกรรมแบบ Layered ที่ได้จากการ Refactor นี้

```
+---------------------------+
|   ARCALC.cob              |  <- Orchestration Layer
|   (อ่าน request, เรียก      |     (จะถูกห่อหุ้มด้วย REST
|    ทั้งสอง subprogram,      |      API ใน Step 847)
|    แสดงผลลัพธ์)             |
+------+--------------+-----+
       |              |
       v              v
+-------------+  +----------------+
| ARDISC.cob  |  | ARLEDGER.cob   |
| (Business   |  | (Data Access / |
|  Logic      |  |  Repository    |
|  Layer -    |  |  Layer -       |
|  pure calc, |  |  file I/O      |
|  no I/O)    |  |  only)         |
+-------------+  +----------------+
```

สถาปัตยกรรมแบบนี้คือ **Layered Architecture** ขั้นพื้นฐาน (Presentation / Business Logic / Data Access)
ที่ Part 086 จะขยายความในบริบทองค์กรขนาดใหญ่ต่อไป

### พิสูจน์ Regression แบบสมบูรณ์: รัน ARCALC.cob เทียบกับ Golden Master

```bash
rm -rf build BALANCE.DAT
mkdir -p build
cobc -I copybooks -c src/ardisc.cob   -o build/ardisc.o
cobc -I copybooks -c src/arledger.cob -o build/arledger.o
cobc -I copybooks -x -o build/arcalc src/arcalc.cob build/ardisc.o build/arledger.o

echo "0001999003PR" | ./build/arcalc
echo "0002000004PX" | ./build/arcalc
cat BALANCE.DAT
```

**ผลลัพธ์จริงที่ได้:**

```
0000059.97|0.050|0000003.00|0000056.97|0000003.99|0000060.96|000000060.96
0000080.00|0.070|0000005.60|0000074.40|0000005.21|0000079.61|000000140.57
00000014057
```

**เปรียบเทียบกับ Golden Master จากขั้นตอนที่ 842 — ตรงกันทุกไบต์ 100%** ทั้งผลลัพธ์แต่ละบรรทัดและเนื้อหา
ไฟล์ `BALANCE.DAT` สุดท้าย นี่คือหลักฐานยืนยันว่า**การ Refactor ทั้งหมดจนถึงจุดนี้ไม่ได้เปลี่ยนพฤติกรรม
ของระบบแม้แต่น้อย** แม้โครงสร้างภายในจะเปลี่ยนจากโปรแกรมเดียว 1 ไฟล์เป็น 3 ไฟล์ที่ทำงานร่วมกันแล้วก็ตาม

### ข้อควรระวัง

- ระวังอย่าลืมลบ `BALANCE.DAT` เก่าก่อนเริ่มทดสอบใหม่ทุกครั้ง (`rm -f BALANCE.DAT`) มิฉะนั้นยอดคงเหลือ
  สะสมจากการทดสอบครั้งก่อนจะปนเข้ามาทำให้ผลลัพธ์ไม่ตรงกับ Golden Master โดยไม่ใช่เพราะโค้ดผิดพลาดแต่
  อย่างใด — นี่เป็นกับดักทั่วไปเมื่อทดสอบระบบที่มีสถานะถาวร
- `ARLEDGER.cob` ยังคงใช้ `OPEN INPUT`/`OPEN OUTPUT` แยกกัน 2 ครั้ง (อ่านแล้วปิด, เขียนแล้วปิด) แทนที่จะ
  ใช้ `OPEN I-O` — นี่เป็นการตัดสินใจคงพฤติกรรมเดิมของ `ARLEGACY.cob` ไว้ทุกประการเพื่อการ Refactor ที่
  ปลอดภัยที่สุด การปรับปรุงเพิ่มเติม (เช่นเปลี่ยนเป็น `OPEN I-O`) ควรทำเป็นขั้นตอนแยกต่างหากพร้อม Golden
  Master Test คุ้มครองอีกครั้ง ไม่ควรทำพร้อมกับการแยก Subprogram ในคราวเดียว (Refactor ทีละก้าวเล็ก ๆ
  ตามหลักการ Part 081)

### แบบฝึกหัดที่ 845.1

**โจทย์**: จงอธิบายว่าทำไมการทดสอบ Regression ในขั้นตอนนี้ (845) ถึงสำคัญกว่าการทดสอบในขั้นตอนที่ 844
ทั้งที่ทั้งสองขั้นตอนต่างก็เป็นการ Refactor เหมือนกัน

**เฉลยแนวทาง**: เพราะขั้นตอนที่ 844 ทดสอบแค่ `ARDISC.cob` แยกเดี่ยว (ยังไม่มี File I/O เข้ามาเกี่ยวข้อง)
ในขณะที่ขั้นตอนที่ 845 เป็นการทดสอบ**ระบบทั้งหมดที่ประกอบร่างสมบูรณ์แล้ว** (`ARCALC` เรียก `ARDISC` และ
`ARLEDGER` ร่วมกัน ผ่านกลไก `CALL` ระหว่างโปรแกรมจริง พร้อมการอ่าน/เขียนไฟล์จริงบนดิสก์) ความเสี่ยงที่จะ
เกิดข้อผิดพลาดจากการเชื่อมต่อระหว่าง Subprogram (เช่น ลำดับพารามิเตอร์ใน `CALL` ไม่ตรงกับ `LINKAGE
SECTION`, หรือไฟล์ `BALANCE.DAT` ถูกอ่าน/เขียนผิดจังหวะ) มีสูงกว่าการทดสอบแต่ละส่วนแยกกันมาก การทดสอบ
Regression แบบเต็มระบบในขั้นตอนนี้จึงเป็นหลักฐานที่หนักแน่นที่สุดว่าการ Refactor ทั้งหมดสำเร็จอย่างสมบูรณ์

---

## ขั้นตอนที่ 846: Unit Testing ARDISC.cob ตามแนวทาง Part 079

### แนวคิด: ทำไมต้องมี Unit Test เพิ่มเติม ทั้งที่มี Golden Master แล้ว

Golden Master Test (ขั้นตอนที่ 842/845) ตรวจสอบว่า**พฤติกรรมโดยรวมของระบบไม่เปลี่ยนแปลง** แต่มันมีข้อ
จำกัด: มันทดสอบแค่ 2 กรณีที่เลือกไว้ตอนต้น ไม่ครอบคลุมทุกเส้นทางของ Decision Table ที่มี 9+ กรณี — **Unit
Test** ตามแนวทาง Part 079 เข้ามาเติมเต็มช่องว่างนี้ โดยทดสอบ `ARDISC.cob` โดยตรงด้วยชุดกรณีทดสอบที่
ออกแบบมาให้ครอบคลุมทุกเส้นทางสำคัญ รวมถึง**กรณีขอบเขต** (boundary case) ที่ Part 083/084 เตือนไว้ว่า
เสี่ยงต่อความเข้าใจผิด

### ซอร์สโค้ดฉบับเต็ม: TESTDISC.cob

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TESTDISC.
       AUTHOR. COBOL-COURSE.
      *> Unit test harness for ARDISC.cob, in the style introduced in
      *> Part 079: each test case sets inputs, CALLs the subprogram
      *> directly (no file I/O, no stdin - that is the whole point of
      *> having extracted ARDISC out of ARLEGACY.cob), and compares
      *> every output field against a hand-checked expected value.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  T-PRICE                 PIC 9(5)V99.
       01  T-QTY                   PIC 9(3).
       01  T-TIER                  PIC X(1).
       01  T-REGION                PIC X(1).
       01  T-SUBTOTAL              PIC 9(7)V99.
       01  T-DISC-PCT              PIC 9V999.
       01  T-DISC-AMT              PIC 9(7)V99.
       01  T-NET                   PIC 9(7)V99.
       01  T-TAX-AMT               PIC 9(7)V99.
       01  T-TOTAL                 PIC 9(7)V99.

       01  E-SUBTOTAL              PIC 9(7)V99.
       01  E-DISC-PCT              PIC 9V999.
       01  E-DISC-AMT              PIC 9(7)V99.
       01  E-NET                   PIC 9(7)V99.
       01  E-TAX-AMT               PIC 9(7)V99.
       01  E-TOTAL                 PIC 9(7)V99.

       01  WS-CASE-NAME            PIC X(40).
       01  WS-PASS-COUNT           PIC 9(3) VALUE 0.
       01  WS-FAIL-COUNT           PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM TEST-PLATINUM-HIGH-DOMESTIC
           PERFORM TEST-GOLD-MID-EXPORT
           PERFORM TEST-REGULAR-LOW-DOMESTIC
           PERFORM TEST-PLATINUM-LOW-EXPORT
           PERFORM TEST-GOLD-BOUNDARY-EXACT-10000

           DISPLAY "-----------------------------------------"
           DISPLAY "PASSED: " WS-PASS-COUNT "  FAILED: "
               WS-FAIL-COUNT
           IF WS-FAIL-COUNT > 0
               MOVE 1 TO RETURN-CODE
           ELSE
               MOVE 0 TO RETURN-CODE
           END-IF
           STOP RUN.

       TEST-PLATINUM-HIGH-DOMESTIC.
           MOVE "PLATINUM-HIGH-DOMESTIC" TO WS-CASE-NAME
           MOVE 200.00 TO T-PRICE
           MOVE 60     TO T-QTY
           MOVE "P"    TO T-TIER
           MOVE "R"    TO T-REGION
           MOVE 12000.00 TO E-SUBTOTAL
           MOVE 0.150     TO E-DISC-PCT
           MOVE 1800.00   TO E-DISC-AMT
           MOVE 10200.00  TO E-NET
           MOVE 714.00    TO E-TAX-AMT
           MOVE 10914.00  TO E-TOTAL
           PERFORM RUN-CASE-AND-CHECK.

       TEST-GOLD-MID-EXPORT.
           MOVE "GOLD-MID-EXPORT" TO WS-CASE-NAME
           MOVE 100.00 TO T-PRICE
           MOVE 60     TO T-QTY
           MOVE "G"    TO T-TIER
           MOVE "X"    TO T-REGION
           MOVE 6000.00  TO E-SUBTOTAL
           MOVE 0.090    TO E-DISC-PCT
           MOVE 540.00   TO E-DISC-AMT
           MOVE 5460.00  TO E-NET
           MOVE 382.20   TO E-TAX-AMT
           MOVE 5842.20  TO E-TOTAL
           PERFORM RUN-CASE-AND-CHECK.

       TEST-REGULAR-LOW-DOMESTIC.
           MOVE "REGULAR-LOW-DOMESTIC" TO WS-CASE-NAME
           MOVE 50.00 TO T-PRICE
           MOVE 10    TO T-QTY
           MOVE "R"   TO T-TIER
           MOVE "R"   TO T-REGION
           MOVE 500.00 TO E-SUBTOTAL
           MOVE 0.000  TO E-DISC-PCT
           MOVE 0.00   TO E-DISC-AMT
           MOVE 500.00 TO E-NET
           MOVE 35.00  TO E-TAX-AMT
           MOVE 535.00 TO E-TOTAL
           PERFORM RUN-CASE-AND-CHECK.

       TEST-PLATINUM-LOW-EXPORT.
           MOVE "PLATINUM-LOW-EXPORT" TO WS-CASE-NAME
           MOVE 10.00 TO T-PRICE
           MOVE 10    TO T-QTY
           MOVE "P"   TO T-TIER
           MOVE "X"   TO T-REGION
           MOVE 100.00 TO E-SUBTOTAL
           MOVE 0.070  TO E-DISC-PCT
           MOVE 7.00   TO E-DISC-AMT
           MOVE 93.00  TO E-NET
           MOVE 6.51   TO E-TAX-AMT
           MOVE 99.51  TO E-TOTAL
           PERFORM RUN-CASE-AND-CHECK.

      *> This case pins down the HIGH-tier threshold boundary:
      *> subtotal exactly equal to 10000.00 must count as ">=", not
      *> just "greater than". Step 846 shows this test catching a
      *> real off-by-one bug introduced while refactoring the
      *> nested IFs into an EVALUATE.
       TEST-GOLD-BOUNDARY-EXACT-10000.
           MOVE "GOLD-BOUNDARY-EXACT-10000" TO WS-CASE-NAME
           MOVE 100.00 TO T-PRICE
           MOVE 100    TO T-QTY
           MOVE "G"    TO T-TIER
           MOVE "R"    TO T-REGION
           MOVE 10000.00 TO E-SUBTOTAL
           MOVE 0.100    TO E-DISC-PCT
           MOVE 1000.00  TO E-DISC-AMT
           MOVE 9000.00  TO E-NET
           MOVE 630.00   TO E-TAX-AMT
           MOVE 9630.00  TO E-TOTAL
           PERFORM RUN-CASE-AND-CHECK.

       RUN-CASE-AND-CHECK.
           CALL "ARDISC" USING T-PRICE, T-QTY, T-TIER, T-REGION,
               T-SUBTOTAL, T-DISC-PCT, T-DISC-AMT, T-NET, T-TAX-AMT,
               T-TOTAL
           END-CALL

           IF T-SUBTOTAL = E-SUBTOTAL AND T-DISC-PCT = E-DISC-PCT
              AND T-DISC-AMT = E-DISC-AMT AND T-NET = E-NET
              AND T-TAX-AMT = E-TAX-AMT AND T-TOTAL = E-TOTAL
               ADD 1 TO WS-PASS-COUNT
               DISPLAY "PASS - " WS-CASE-NAME
           ELSE
               ADD 1 TO WS-FAIL-COUNT
               DISPLAY "FAIL - " WS-CASE-NAME
               DISPLAY "  expected: " E-SUBTOTAL "|" E-DISC-PCT "|"
                   E-DISC-AMT "|" E-NET "|" E-TAX-AMT "|" E-TOTAL
               DISPLAY "  actual:   " T-SUBTOTAL "|" T-DISC-PCT "|"
                   T-DISC-AMT "|" T-NET "|" T-TAX-AMT "|" T-TOTAL
           END-IF.
```

สังเกตกรณีทดสอบสุดท้าย **`TEST-GOLD-BOUNDARY-EXACT-10000`** — มันจงใจทดสอบยอดสั่งซื้อที่เท่ากับ
`10,000.00` **พอดี** ตามที่ Part 083 ขั้นตอนที่ 827 เตือนไว้ว่าเป็นจุดเสี่ยงต่อความเข้าใจผิดระหว่าง `>=`
กับ `>`

### สาธิตของจริง: Unit Test จับบั๊กที่เกิดจากการ Refactor ผิดพลาด

เพื่อพิสูจน์ว่า Unit Test ชุดนี้**ทำงานได้จริง ไม่ใช่แค่โชว์ผ่าน** ลองจำลองสถานการณ์ที่ผู้พัฒนาคนหนึ่ง
Refactor `DETERMINE-DISCOUNT-RATE` โดยพิมพ์ผิดพลาดเล็กน้อย: ใช้ `>` แทนที่จะเป็น `>=` ในทุกเงื่อนไข
เปรียบเทียบเกณฑ์ (`LK-SUBTOTAL > TIER-HIGH-THRESHOLD` แทนที่จะเป็น `LK-SUBTOTAL >= TIER-HIGH-THRESHOLD`)
— บั๊กแบบนี้เกิดขึ้นได้ง่ายมากในความเป็นจริงเมื่อเปลี่ยนจาก `IF`/`ELSE` ซ้อนหลายชั้นมาเป็น `EVALUATE`

```bash
cobc -I copybooks -c src/ardisc_buggy.cob -o build/ardisc_buggy.o
cobc -I copybooks -x -o build/testdisc_buggy tests/testdisc.cob build/ardisc_buggy.o
./build/testdisc_buggy
echo "EXIT CODE: $?"
```

**ผลลัพธ์จริงที่ได้ (คอมไพล์และรันจริง ไม่ใช่การจำลอง):**

```
PASS - PLATINUM-HIGH-DOMESTIC
PASS - GOLD-MID-EXPORT
PASS - REGULAR-LOW-DOMESTIC
PASS - PLATINUM-LOW-EXPORT
FAIL - GOLD-BOUNDARY-EXACT-10000
  expected: 0010000.00|0.100|0001000.00|0009000.00|0000630.00|0009630.00
  actual:   0010000.00|0.070|0000700.00|0009300.00|0000651.00|0009951.00
-----------------------------------------
PASSED: 004  FAILED: 001
EXIT CODE: 1
```

**Unit Test จับบั๊กได้ทันที** — เมื่อยอดสั่งซื้อเท่ากับ 10,000.00 พอดี โปรแกรมที่มีบั๊ก (`>` แทน `>=`)
คำนวณว่ายังไม่ถึงเกณฑ์ HIGH จึงใช้ส่วนลด MID (7%) แทนที่จะเป็น HIGH (10%) ที่ถูกต้อง — 4 กรณีแรกยังคง
ผ่านตามปกติเพราะไม่ได้ทดสอบค่าที่ขอบเขตพอดี มีเพียงกรณีที่ออกแบบมาเฉพาะสำหรับทดสอบขอบเขตเท่านั้นที่จับ
บั๊กนี้ได้ — **นี่คือเหตุผลที่ Unit Test ต้องออกแบบกรณีทดสอบขอบเขตอย่างตั้งใจ ไม่ใช่แค่สุ่มเลือกค่าทั่วไป**

### แก้บั๊กแล้วรันใหม่

```bash
cobc -I copybooks -c src/ardisc.cob -o build/ardisc.o
cobc -I copybooks -x -o build/testdisc tests/testdisc.cob build/ardisc.o
./build/testdisc
echo "EXIT CODE: $?"
```

**ผลลัพธ์จริงที่ได้:**

```
PASS - PLATINUM-HIGH-DOMESTIC
PASS - GOLD-MID-EXPORT
PASS - REGULAR-LOW-DOMESTIC
PASS - PLATINUM-LOW-EXPORT
PASS - GOLD-BOUNDARY-EXACT-10000
-----------------------------------------
PASSED: 005  FAILED: 000
EXIT CODE: 0
```

ทุกกรณีผ่านแล้ว และ **`RETURN-CODE`** ถูกตั้งเป็น 0 (ผ่าน) / 1 (ไม่ผ่าน) โดยเจตนา — นี่คือกลไกสำคัญที่
ขั้นตอนที่ 848 (CI Pipeline) จะใช้ตรวจสอบผลการทดสอบแบบอัตโนมัติ

### ข้อควรระวัง

- ค่า Exit Code ของโปรแกรม COBOL มาจาก `RETURN-CODE` ซึ่งเป็นตัวแปรพิเศษที่ GnuCOBOL รองรับในตัว
  (ทบทวนแนวคิดใกล้เคียงจาก Part 048 เรื่อง Error Handling) — การตั้งค่านี้อย่างถูกต้องสำคัญมากสำหรับ
  การผสาน Unit Test เข้ากับเครื่องมือ CI/CD ภายนอกที่ตรวจสอบ Exit Code เป็นมาตรฐาน
- ชุดทดสอบทั้ง 5 กรณีนี้ยังไม่ครอบคลุม**ทุก**เส้นทางของ Decision Table ทั้ง 18 กรณี (9 tier×amount x 2
  region) — ในทางปฏิบัติจริงควรเพิ่มกรณีทดสอบให้ครอบคลุมมากกว่านี้ แบบฝึกหัดต่อไปนี้จะให้ฝึกเพิ่มกรณี
  ทดสอบเอง

### แบบฝึกหัดที่ 846.1

**โจทย์**: จงเขียนกรณีทดสอบเพิ่มเติมอีก 1 กรณีที่ทดสอบขอบเขตของเกณฑ์ `TIER-MID-THRESHOLD`
(5,000.00 พอดี) สำหรับลูกค้าระดับ Regular (`"R"`) ที่ไม่ใช่ Export

**เฉลย**:

```cobol
       TEST-REGULAR-BOUNDARY-EXACT-5000.
           MOVE "REGULAR-BOUNDARY-EXACT-5000" TO WS-CASE-NAME
           MOVE 100.00 TO T-PRICE
           MOVE 50     TO T-QTY
           MOVE "R"    TO T-TIER
           MOVE "R"    TO T-REGION
           MOVE 5000.00 TO E-SUBTOTAL
           MOVE 0.020   TO E-DISC-PCT
           MOVE 100.00  TO E-DISC-AMT
           MOVE 4900.00 TO E-NET
           MOVE 343.00  TO E-TAX-AMT
           MOVE 5243.00 TO E-TOTAL
           PERFORM RUN-CASE-AND-CHECK.
```

(subtotal = 100.00 × 50 = 5,000.00 พอดี ตกอยู่ในเกณฑ์ MID ของ Regular คือ 2% ไม่ใช่ 0% ของเกณฑ์ LOW)
พร้อมเพิ่ม `PERFORM TEST-REGULAR-BOUNDARY-EXACT-5000` ใน `MAIN-PARA`

---

## ขั้นตอนที่ 847: ห่อหุ้มด้วย REST API ตามแนวทาง Part 073

### แนวคิด: REST Wrapper คือ Presentation Layer ชั้นใหม่

ตามการตัดสินใจ ADR-085-01 (ขั้นตอนที่ 841) เราจะห่อหุ้ม `ARCALC.cob` ด้วย REST API แทนที่จะแก้ไข COBOL
ให้พูดภาษา HTTP โดยตรง — Part 073 เคยสอนหลักการนี้ไว้แล้ว: ให้ภาษาที่ถนัดงาน Network (Python) ทำหน้าที่
Wrapper แล้วเรียก COBOL ผ่าน Subprocess เหมือนเรียกเครื่องมือบรรทัดคำสั่งตัวหนึ่ง

### ซอร์สโค้ดฉบับเต็ม: ar_server.py

```python
#!/usr/bin/env python3
"""AR-MINI REST wrapper (Part 073 style) - stdlib only, no
third-party framework. POSTs a JSON invoice request, spawns the
compiled ARCALC COBOL program as a subprocess for exactly this one
request, and turns its pipe-delimited stdout line into JSON.
"""
import json
import os
import subprocess
from http.server import BaseHTTPRequestHandler, HTTPServer

ARCALC_PATH = os.environ.get("ARCALC_PATH", "./arcalc")

TIER_CODES = {"platinum": "P", "gold": "G", "regular": "R"}


def build_request_line(data):
    price = float(data["price"])
    qty = int(data["qty"])
    tier_raw = str(data["tier"]).strip().upper()
    tier = TIER_CODES.get(tier_raw.lower(), tier_raw[:1])
    region = str(data.get("region", "R")).strip().upper()[:1] or "R"

    # PIC 9(5)V99 -> 7 digits, unscaled (multiply by 100, zero pad)
    price_field = "%07d" % round(price * 100)
    qty_field = "%03d" % qty
    line = price_field + qty_field + tier + region
    if len(line) != 12:
        raise ValueError("request line must be exactly 12 characters, "
                          "got %d (%r)" % (len(line), line))
    return line


def parse_response_line(line):
    parts = line.strip().split("|")
    keys = ["subtotal", "discount_pct", "discount_amt", "net",
             "tax_amt", "total", "new_balance"]
    return {k: float(v) for k, v in zip(keys, parts)}


class Handler(BaseHTTPRequestHandler):
    def _send_json(self, status, payload):
        body = json.dumps(payload).encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body)))
        self.end_headers()
        self.wfile.write(body)

    def do_POST(self):
        if self.path != "/invoice":
            self._send_json(404, {"error": "not found"})
            return
        length = int(self.headers.get("Content-Length", "0"))
        raw = self.rfile.read(length)
        try:
            data = json.loads(raw)
            req_line = build_request_line(data)
        except (ValueError, KeyError, TypeError) as exc:
            self._send_json(400, {"error": "bad request: %s" % exc})
            return

        try:
            result = subprocess.run(
                [ARCALC_PATH],
                input=(req_line + "\n").encode("ascii"),
                capture_output=True,
                timeout=5,
            )
        except OSError as exc:
            self._send_json(502, {"error": "cobol process failed: %s"
                                   % exc})
            return

        if result.returncode != 0:
            self._send_json(502, {
                "error": "ARCALC exited with code %d" % result.returncode,
                "stderr": result.stderr.decode("utf-8", "replace"),
            })
            return

        out_line = result.stdout.decode("ascii").strip()
        self._send_json(200, parse_response_line(out_line))

    def log_message(self, fmt, *args):
        pass  # keep test output quiet - a real service would log


def main():
    server = HTTPServer(("127.0.0.1", 8085), Handler)
    print("AR-MINI REST wrapper listening on http://127.0.0.1:8085")
    server.serve_forever()


if __name__ == "__main__":
    main()
```

### อธิบายจุดสำคัญ

- `build_request_line()` แปลง JSON ที่มนุษย์อ่านง่าย (`{"tier": "platinum"}`) ให้เป็นบรรทัด Fixed-Width
  12 ตัวอักษรที่ `ARCALC.cob` เข้าใจ — นี่คือหน้าที่หลักของ Wrapper: **แปลระหว่างโลกสองใบ** โดยที่ COBOL
  ไม่ต้องรู้จัก JSON เลยแม้แต่น้อย
- `subprocess.run([ARCALC_PATH], input=...)` เรียกโปรแกรม COBOL ที่คอมไพล์แล้วเป็น Process แยกต่างหาก
  หนึ่งครั้งต่อหนึ่ง Request — รูปแบบเดียวกับที่ Part 073 สอนไว้ (ไม่มี Native Binding ที่ซับซ้อน แค่เรียก
  ทำนองเดียวกับเรียกคำสั่งบรรทัดคำสั่งทั่วไป)
- `ARCALC_PATH` อ่านจาก Environment Variable พร้อมค่า default — ทำให้ปรับ Path ได้ยืดหยุ่นเมื่อรันจาก
  CI Pipeline (ขั้นตอนที่ 848) โดยไม่ต้องแก้โค้ด Python

### ทดสอบ End-to-End ด้วย curl จริง

```bash
rm -f BALANCE.DAT
cp build/arcalc rest/arcalc
cd rest
python3 ar_server.py &
sleep 1

curl -s -X POST http://127.0.0.1:8085/invoice \
    -H 'Content-Type: application/json' \
    -d '{"price":19.99,"qty":3,"tier":"platinum","region":"R"}'
echo ""

curl -s -X POST http://127.0.0.1:8085/invoice \
    -H 'Content-Type: application/json' \
    -d '{"price":20.00,"qty":4,"tier":"P","region":"X"}'
echo ""

curl -s -X POST http://127.0.0.1:8085/invoice \
    -H 'Content-Type: application/json' \
    -d '{"price":"oops"}'
echo ""
```

**ผลลัพธ์จริงที่ได้:**

```
{"subtotal": 59.97, "discount_pct": 0.05, "discount_amt": 3.0, "net": 56.97, "tax_amt": 3.99, "total": 60.96, "new_balance": 60.96}
{"subtotal": 80.0, "discount_pct": 0.07, "discount_amt": 5.6, "net": 74.4, "tax_amt": 5.21, "total": 79.61, "new_balance": 140.57}
{"error": "bad request: could not convert string to float: 'oops'"}
```

**ผลลัพธ์ตรงกับ Golden Master ทุกตัวเลข** (`60.96` และ `140.57` ตรงกับขั้นตอนที่ 842/845 พอดี) พิสูจน์
ว่าการห่อหุ้มด้วย REST API **ไม่ได้เปลี่ยนพฤติกรรมการคำนวณเลย** และระบบจัดการ Request ที่ผิดพลาด
(`tier`/`price` ที่แปลงเป็นตัวเลขไม่ได้) ได้อย่างเหมาะสมด้วย HTTP status code 400 แทนที่จะทำให้ Server
ล่ม

### ข้อควรระวัง

- สังเกตว่า Wrapper **ไม่ได้เขียนตรรกะทางธุรกิจซ้ำเลย** — มันแค่แปลงรูปแบบข้อมูลไปมาเท่านั้น กฎ Business
  Logic ทั้งหมดยังคงอยู่ใน `ARDISC.cob` เพียงจุดเดียว (Single Source of Truth) ตรงกับหลักการที่ Part 033
  เคยสอนไว้เรื่อง Copybook — หลักการเดียวกันนี้ใช้ได้กับการออกแบบ Layer ด้วยเช่นกัน
- ระบบนี้เรียก Subprocess ใหม่ทุก Request (ไม่มีการ Cache หรือ Reuse process) ซึ่งเหมาะกับปริมาณ Request
  ต่ำ-ปานกลางสำหรับการเรียนรู้ แต่ระบบ Production ที่มี Traffic สูงมากอาจต้องพิจารณาแนวทางอื่นเพิ่มเติม
  (เช่น Connection Pooling ของ CICS ตามที่ Part 061-064 เคยกล่าวถึง) ซึ่งอยู่นอกขอบเขตของ Part นี้

### แบบฝึกหัดที่ 847.1

**โจทย์**: จงอธิบายว่าทำไมการทดสอบ Request ที่ผิดพลาด (`{"price":"oops"}`) ในขั้นตอนนี้ถึงสำคัญไม่แพ้
การทดสอบ Request ที่ถูกต้อง

**เฉลยแนวทาง**: เพราะระบบที่ใช้งานจริงจะได้รับข้อมูลจากภายนอก (ผู้ใช้, ระบบอื่น) ซึ่งไม่สามารถรับประกันได้
ว่าจะถูกต้องเสมอ — REST API ที่ดีต้องจัดการกับ Input ที่ผิดพลาดอย่างสุภาพ (ส่ง HTTP 400 พร้อมข้อความ
อธิบายที่ชัดเจน) แทนที่จะปล่อยให้ Server ล่ม (crash) หรือส่งพฤติกรรมที่ไม่คาดคิดกลับไป การทดสอบ Error
Case จึงเป็นส่วนสำคัญของการยืนยันคุณภาพ API เทียบเท่ากับการทดสอบ Happy Path (กรณีที่ข้อมูลถูกต้อง) และ
ในเคสนี้ยังพิสูจน์ด้วยว่าข้อผิดพลาดถูกจับที่ชั้น Python **ก่อน**ที่จะส่งต่อไปเรียก COBOL เลยด้วยซ้ำ ทำให้
ไม่มีการสิ้นเปลืองทรัพยากรเรียก Subprocess โดยไม่จำเป็น

---

## ขั้นตอนที่ 848: CI Pipeline ตามแนวทาง Part 078

### แนวคิด: รวมทุกขั้นตอนก่อนหน้าเป็นกระบวนการอัตโนมัติเดียว

ขั้นตอนที่ 842-847 ทั้งหมดถูกทำด้วยมือทีละคำสั่ง — Part 078 สอนไว้ว่าในทีมงานจริง กระบวนการเหล่านี้ต้อง
ถูก**ทำให้เป็นอัตโนมัติ**ผ่าน CI Pipeline เพื่อให้ทุกครั้งที่มีคนแก้โค้ด ระบบตรวจสอบทั้งหมดถูกรันซ้ำโดย
อัตโนมัติโดยไม่ต้องพึ่งพาความจำของมนุษย์ว่า "อย่าลืมรันคำสั่งนั้นคำสั่งนี้ก่อน commit"

### ซอร์สโค้ดฉบับเต็ม: ci.sh

```bash
#!/bin/bash
# ci.sh - AR-MINI continuous integration pipeline (Part 078 style).
# Run from the ar-mini/ project root. Every step must pass for the
# script to exit 0; any failure stops the pipeline immediately.
set -e
cd "$(dirname "$0")"

echo "=================================================="
echo "STAGE 1/4: COMPILE"
echo "=================================================="
rm -rf build
mkdir -p build
cobc -I copybooks -c src/ardisc.cob   -o build/ardisc.o
cobc -I copybooks -c src/arledger.cob -o build/arledger.o
cobc -I copybooks -x -o build/arcalc \
     src/arcalc.cob build/ardisc.o build/arledger.o
echo "compile OK"
echo ""

echo "=================================================="
echo "STAGE 2/4: STATIC CHECK (dead-paragraph scan)"
echo "=================================================="
# Reuses the Part 083 technique as an automated quality gate: any
# paragraph with zero PERFORM/GO TO references is worth a second
# look before it ships. MAIN-PARA (the entry point) is expected to
# show up here and is not a failure by itself.
./find_dead_paragraphs.sh src/arcalc.cob || true
./find_dead_paragraphs.sh src/ardisc.cob || true
./find_dead_paragraphs.sh src/arledger.cob || true
echo ""

echo "=================================================="
echo "STAGE 3/4: UNIT TESTS"
echo "=================================================="
cobc -I copybooks -x -o build/testdisc tests/testdisc.cob build/ardisc.o
./build/testdisc
echo "unit tests OK"
echo ""

echo "=================================================="
echo "STAGE 4/4: INTEGRATION SMOKE TEST (REST wrapper)"
echo "=================================================="
rm -f BALANCE.DAT build/BALANCE.DAT
cp build/arcalc rest/arcalc
( cd rest && ARCALC_PATH=./arcalc python3 ar_server.py > /tmp/ar_server.log 2>&1 & echo $! > /tmp/ar_server.pid )
sleep 1

RESPONSE=$(curl -s -X POST http://127.0.0.1:8085/invoice \
    -H 'Content-Type: application/json' \
    -d '{"price":19.99,"qty":3,"tier":"platinum","region":"R"}')
echo "server response: $RESPONSE"

kill "$(cat /tmp/ar_server.pid)" 2>/dev/null || true
rm -f rest/arcalc rest/BALANCE.DAT /tmp/ar_server.pid

echo "$RESPONSE" | grep -q '"total": 60.96' && echo "smoke test OK" || {
    echo "smoke test FAILED: unexpected response"
    exit 1
}
echo ""

echo "=================================================="
echo "PIPELINE PASSED"
echo "=================================================="
```

### อธิบายโครงสร้าง Pipeline

Pipeline นี้มี 4 ขั้นตอน ตรงกับ 4 เทคนิคที่เรียนมาตลอดเฟส 5: **Compile** (พื้นฐานที่สุด), **Static
Check** (นำเทคนิค Part 083 กลับมาใช้เป็น Quality Gate อัตโนมัติ), **Unit Test** (Part 079, ขั้นตอนที่
846), และ **Integration Smoke Test** (ทดสอบ REST Wrapper จาก Part 073/ขั้นตอนที่ 847 แบบอัตโนมัติ)
`set -e` ที่หัวสคริปต์ทำให้ Pipeline **หยุดทันทีที่ขั้นตอนใดล้มเหลว** — ตรงกับหลักการ "Fail Fast" ของ
CI/CD ที่ Part 078 สอนไว้

### รันจริง: กรณีที่ทุกอย่างผ่าน

```bash
chmod +x ci.sh find_dead_paragraphs.sh
./ci.sh
```

**ผลลัพธ์จริงที่ได้ (ตัดบางส่วนเพื่อความกระชับ):**

```
==================================================
STAGE 1/4: COMPILE
==================================================
compile OK

==================================================
STAGE 2/4: STATIC CHECK (dead-paragraph scan)
==================================================
Paragraphs found:
  - MAIN-PARA

Paragraphs with NO incoming PERFORM/GO TO (dead-code candidates):
  -> MAIN-PARA (0 references)
...
==================================================
STAGE 3/4: UNIT TESTS
==================================================
PASS - PLATINUM-HIGH-DOMESTIC
PASS - GOLD-MID-EXPORT
PASS - REGULAR-LOW-DOMESTIC
PASS - PLATINUM-LOW-EXPORT
PASS - GOLD-BOUNDARY-EXACT-10000
-----------------------------------------
PASSED: 005  FAILED: 000
unit tests OK

==================================================
STAGE 4/4: INTEGRATION SMOKE TEST (REST wrapper)
==================================================
server response: {"subtotal": 59.97, "discount_pct": 0.05, "discount_amt": 3.0, "net": 56.97, "tax_amt": 3.99, "total": 60.96, "new_balance": 60.96}
smoke test OK

==================================================
PIPELINE PASSED
==================================================
```

### รันจริง: กรณีที่มีบั๊กในโค้ด (Pipeline ต้องหยุดและรายงานความล้มเหลว)

จำลองสถานการณ์จริงที่ผู้พัฒนา commit โค้ดที่มีบั๊ก `>` แทน `>=` (เหมือนขั้นตอนที่ 846) เข้าไปแทนที่
`src/ardisc.cob` แล้วรัน Pipeline อีกครั้ง:

```bash
cp src/ardisc.cob /tmp/ardisc_good_backup.cob
cp src/ardisc_buggy.cob src/ardisc.cob   # จำลองว่าคอมมิตโค้ดที่มีบั๊กเข้ามา
./ci.sh; echo "CI EXIT CODE: $?"
```

**ผลลัพธ์จริงที่ได้:**

```
==================================================
STAGE 1/4: COMPILE
==================================================
compile OK

==================================================
STAGE 2/4: STATIC CHECK (dead-paragraph scan)
==================================================
...
==================================================
STAGE 3/4: UNIT TESTS
==================================================
PASS - PLATINUM-HIGH-DOMESTIC
PASS - GOLD-MID-EXPORT
PASS - REGULAR-LOW-DOMESTIC
PASS - PLATINUM-LOW-EXPORT
FAIL - GOLD-BOUNDARY-EXACT-10000
  expected: 0010000.00|0.100|0001000.00|0009000.00|0000630.00|0009630.00
  actual:   0010000.00|0.070|0000700.00|0009300.00|0000651.00|0009951.00
-----------------------------------------
PASSED: 004  FAILED: 001
CI EXIT CODE: 1
```

สังเกตว่า Pipeline **หยุดทันที**หลังจาก Stage 3 ล้มเหลว (เพราะ `set -e` และ `./build/testdisc` คืนค่า
Exit Code ที่ไม่ใช่ 0) — **ไม่มีการรัน Stage 4 (Integration Smoke Test) เลย** ป้องกันไม่ให้โค้ดที่มีบั๊ก
ถูกนำไป Deploy ต่อ นี่คือพฤติกรรมที่ถูกต้องของ CI Pipeline: หยุดที่จุดแรกที่พบปัญหา ไม่เสียเวลาทดสอบ
ขั้นตอนถัดไปที่ต้องพึ่งพาความถูกต้องของขั้นตอนก่อนหน้า

```bash
cp /tmp/ardisc_good_backup.cob src/ardisc.cob   # แก้บั๊กกลับ
./ci.sh
```

รันซ้ำแล้วกลับมาเป็น `PIPELINE PASSED` เหมือนเดิมทุกประการ

### ข้อควรระวัง

- สังเกตว่า Stage 2 (Static Check) ใช้ `|| true` ต่อท้ายทุกคำสั่ง — หมายความว่าขั้นตอนนี้**รายงานผลแต่
  ไม่ทำให้ Pipeline หยุด**แม้จะพบย่อหน้าที่น่าสงสัย เพราะอย่างที่ Part 083 ขั้นตอนที่ 825 อธิบายไว้ ผล
  จากเครื่องมือนี้มี False Positive ได้ (`MAIN-PARA`) การให้มันหยุด Pipeline ทันทีจะสร้างความรำคาญโดย
  ไม่จำเป็น ในทางปฏิบัติจริงควรใช้เป็น "ข้อมูลแจ้งเตือน" ให้มนุษย์ทบทวนเพิ่มเติมมากกว่าใช้เป็นเงื่อนไข
  Pass/Fail แบบเข้มงวด
- Integration Smoke Test (Stage 4) เปิด Server จริงบน Port 8085 — ในสภาพแวดล้อม CI จริงต้องระวังเรื่อง
  Port ชนกันหากมีหลาย Pipeline รันพร้อมกัน (แนวทางแก้ไขทั่วไปคือสุ่ม Port หรือใช้ Container แยกตาม
  Part 076 ที่เคยสอนเรื่อง Docker)

### แบบฝึกหัดที่ 848.1

**โจทย์**: จงอธิบายว่าทำไมการที่ Pipeline "หยุดทันทีที่ Stage 3 ล้มเหลว โดยไม่รัน Stage 4 ต่อ" ถึงเป็น
พฤติกรรมที่ถูกต้อง แทนที่จะรันทุก Stage ให้ครบแล้วค่อยสรุปผลรวมทีเดียว

**เฉลยแนวทาง**: เพราะ Stage 4 (Integration Smoke Test) **พึ่งพา**ความถูกต้องของโค้ดที่ผ่านมาจาก Stage
ก่อนหน้า — หากโค้ดมีบั๊กที่ Unit Test จับได้แล้ว การเดินหน้าไปสร้าง Server และทดสอบ REST API ต่อก็ไม่มี
ประโยชน์อะไรเพิ่มเติม เพราะรู้อยู่แล้วว่าผลลัพธ์จะผิดตามไปด้วย (สิ้นเปลืองเวลาและทรัพยากรของเครื่อง CI
โดยเปล่าประโยชน์) หลักการ "Fail Fast" ยังช่วยให้ผู้พัฒนาได้รับ Feedback เร็วที่สุดเท่าที่จะทำได้ว่าอะไร
ผิดพลาด (จาก log ของ Stage 3 ที่ยังคงแสดงผลชัดเจนอยู่) แทนที่จะต้องรอดู log ยาว ๆ ของทุก Stage ก่อนจะ
รู้ว่าจุดล้มเหลวจริง ๆ อยู่ตรงไหน

---

## ขั้นตอนที่ 849: Git Workflow สำหรับทีมที่ทำงานลักษณะนี้ (ทบทวน Part 080)

### แนวคิด: การ Refactor ควรถูกบันทึกเป็นประวัติที่อ่านได้

Part 080 สอนหลักการ Git Workflow สำหรับทีม COBOL ไว้แล้ว ขั้นตอนนี้จะแสดงให้เห็นว่าจะนำหลักการนั้นมาใช้
กับโปรเจกต์ Modernization แบบ Part นี้อย่างไร — โดยเฉพาะการแบ่ง Commit ให้สอดคล้องกับแต่ละขั้นตอนของ
การ Refactor ที่ทำมาตั้งแต่ขั้นตอนที่ 842

> **หมายเหตุ**: คำสั่ง Git ในขั้นตอนนี้เป็น**ตัวอย่างประกอบการอธิบาย** (illustrative) แสดงรูปแบบที่ทีม
> ควรใช้จริงเมื่อทำงานกับ Repository ของโปรเจกต์ ไม่ใช่คำสั่งที่ถูกรันจริงกับ Repository ของหลักสูตรนี้

### กลยุทธ์ Branch: หนึ่ง Feature Branch ต่อหนึ่งขั้นตอนการ Refactor

```
main
 |
 +-- feature/golden-master-baseline        (ขั้นตอนที่ 842)
 |     commit: "Add ARLEGACY.cob baseline and record golden master output"
 |
 +-- feature/extract-ardisc-subprogram     (ขั้นตอนที่ 844)
 |     commit: "Extract ARDISC.cob calculation engine from ARLEGACY"
 |     commit: "Add AR-CONST.CPY named constants, replacing magic numbers"
 |     commit: "Verify ARDISC output matches golden master (regression proof)"
 |
 +-- feature/extract-arledger-and-arcalc   (ขั้นตอนที่ 845)
 |     commit: "Extract ARLEDGER.cob repository subprogram"
 |     commit: "Add ARCALC.cob orchestrator, replacing ARLEGACY.cob"
 |     commit: "Verify full refactored system matches golden master byte-for-byte"
 |
 +-- feature/unit-tests-ardisc             (ขั้นตอนที่ 846)
 |     commit: "Add TESTDISC.cob unit test harness with 5 cases"
 |     commit: "Fix off-by-one boundary bug caught by TEST-GOLD-BOUNDARY-EXACT-10000"
 |
 +-- feature/rest-api-wrapper              (ขั้นตอนที่ 847)
 |     commit: "Add ar_server.py REST wrapper around ARCALC"
 |     commit: "Add error handling for malformed invoice requests"
 |
 `-- feature/ci-pipeline                   (ขั้นตอนที่ 848)
       commit: "Add ci.sh pipeline: compile, static check, unit test, smoke test"
```

### ตัวอย่าง Commit Message ที่ดีสำหรับงาน Refactor (ทบทวนหลักการจาก Part 080)

```
Extract ARDISC.cob calculation engine from ARLEGACY

ARLEGACY.cob mixed discount/tax calculation with file I/O in a
single MAIN-PARA, making it impossible to unit test the calculation
in isolation (see Part 083 Step 843 analysis).

This commit extracts the pure calculation into ARDISC.cob as a
subprogram with no file access, using named constants from
AR-CONST.CPY instead of the magic numbers ARLEGACY.cob had.

Verified against the golden master recorded in Step 842: both test
inputs produce byte-for-byte identical output for every calculated
field (subtotal, discount_pct, discount_amt, net, tax_amt, total).

No behavior change - this is a pure refactor.
```

สังเกตโครงสร้างของ Commit Message ที่ดีนี้: **บรรทัดแรกสั้นกระชับ** (อธิบาย "อะไร"), ตามด้วย**เนื้อหา
อธิบาย "ทำไม"** (บริบทของปัญหาที่แก้), และ**อ้างอิงหลักฐานการทดสอบ** (Golden Master) — ตรงกับหลักการ
Commit Message ที่ดีที่ Part 080 สอนไว้

### ตัวอย่าง Pull Request Description สำหรับการรวม Feature Branch

```
## Title: Extract calculation engine from ARLEGACY (Step 844)

## Summary
- Extracts ARDISC.cob (pure calculation subprogram) from
  ARLEGACY.cob's monolithic MAIN-PARA
- Adds AR-CONST.CPY replacing all magic numbers with named 78-level
  constants
- Zero behavior change: verified against golden master from Step 842

## Test plan
- [x] cobc -I copybooks -c src/ardisc.cob compiles cleanly
- [x] Driver program calling ARDISC with golden master inputs
      produces byte-for-byte identical output
- [x] find_dead_paragraphs.sh shows no unexpected dead paragraphs
```

### ข้อควรระวัง

- **Commit ทีละก้าวเล็ก ๆ ที่ทดสอบผ่านแล้วเสมอ** — อย่ารวมการแยก `ARDISC` และ `ARLEDGER` ไว้ใน Commit
  เดียวกัน แม้จะดูเหมือนเป็นงานที่เกี่ยวข้องกัน เพราะหาก Golden Master ล้มเหลวหลัง Commit จะได้รู้ทันทีว่า
  ปัญหาอยู่ที่การเปลี่ยนแปลงชิ้นไหน (ทบทวนหลักการ `git bisect` จาก Part 080)
- Pull Request ที่ดีควรมี "Test plan" ที่ชัดเจนเสมอ โดยเฉพาะงาน Refactor ที่อ้างว่า "ไม่เปลี่ยนพฤติกรรม"
  — ผู้ตรวจสอบ (Reviewer) ต้องเห็นหลักฐานที่ยืนยันคำกล่าวอ้างนั้นได้ ไม่ใช่แค่เชื่อคำพูดเปล่า ๆ

### แบบฝึกหัดที่ 849.1

**โจทย์**: จงเขียน Commit Message สำหรับการเปลี่ยนแปลงในขั้นตอนที่ 846 (พบและแก้บั๊ก off-by-one จาก
Unit Test) ตามรูปแบบที่ดีที่แสดงไว้ข้างต้น

**เฉลยแนวทาง**:

```
Fix off-by-one boundary bug in DETERMINE-DISCOUNT-RATE

TEST-GOLD-BOUNDARY-EXACT-10000 in TESTDISC.cob caught a bug where
the EVALUATE refactor in ARDISC.cob used ">" instead of ">=" when
comparing subtotal against TIER-HIGH-THRESHOLD and
TIER-MID-THRESHOLD, so an order of exactly 10,000.00 was incorrectly
classified into the MID tier (7%) instead of HIGH (10%).

Fixed by restoring ">=" in all six threshold comparisons.

Before fix: 4/5 unit tests passed (TEST-GOLD-BOUNDARY-EXACT-10000
failed).
After fix: 5/5 unit tests passed.
```

---

## ขั้นตอนที่ 850: การทดสอบบูรณาการครั้งสุดท้ายและบทสรุปโปรเจกต์

### รัน Pipeline ทั้งหมดอีกครั้งเป็นการยืนยันครั้งสุดท้าย

ก่อนปิดโปรเจกต์ ให้รัน `ci.sh` อีกครั้งจากสถานะโค้ดล่าสุด เพื่อยืนยันว่าทุกส่วนที่สร้างขึ้นตลอด 9 ขั้นตอน
ที่ผ่านมาทำงานร่วมกันได้อย่างสมบูรณ์:

```bash
./ci.sh
```

**ผลลัพธ์จริงที่ได้:**

```
==================================================
STAGE 1/4: COMPILE
==================================================
compile OK

==================================================
STAGE 2/4: STATIC CHECK (dead-paragraph scan)
==================================================
Paragraphs found:
  - MAIN-PARA
Paragraphs with NO incoming PERFORM/GO TO (dead-code candidates):
  -> MAIN-PARA (0 references)
...
==================================================
STAGE 3/4: UNIT TESTS
==================================================
PASS - PLATINUM-HIGH-DOMESTIC
PASS - GOLD-MID-EXPORT
PASS - REGULAR-LOW-DOMESTIC
PASS - PLATINUM-LOW-EXPORT
PASS - GOLD-BOUNDARY-EXACT-10000
-----------------------------------------
PASSED: 005  FAILED: 000
unit tests OK

==================================================
STAGE 4/4: INTEGRATION SMOKE TEST (REST wrapper)
==================================================
server response: {"subtotal": 59.97, "discount_pct": 0.05, "discount_amt": 3.0, "net": 56.97, "tax_amt": 3.99, "total": 60.96, "new_balance": 60.96}
smoke test OK

==================================================
PIPELINE PASSED
==================================================
```

### Checklist สรุปทุกเทคนิคจาก Part 071-084 ที่ถูกนำมาใช้จริงในโปรเจกต์นี้

| Part | เทคนิค | นำไปใช้ที่ไหนในโปรเจกต์นี้ |
|---|---|---|
| 073 | REST API Wrapper | `ar_server.py` ห่อหุ้ม `ARCALC.cob` (ขั้นตอนที่ 847) |
| 077 | Strangler Fig / Wrap-Encapsulate | ADR-085-01 เลือกคง COBOL + ห่อหุ้มแทน Rewrite (ขั้นตอนที่ 841) |
| 078 | CI/CD Pipeline | `ci.sh` 4 ขั้นตอนอัตโนมัติ (ขั้นตอนที่ 848) |
| 079 | Unit Testing | `TESTDISC.cob` พร้อม Table-driven test cases (ขั้นตอนที่ 846) |
| 080 | Git Workflow | Feature Branch + Commit Message + PR ตัวอย่าง (ขั้นตอนที่ 849) |
| 081 | Clean Code / Refactoring | Extract Subprogram, ตั้งชื่อค่าคงที่, ลบ Magic Number (ขั้นตอนที่ 844) |
| 082 | Design Patterns | Repository Pattern (`ARLEDGER`), Pure Function (`ARDISC`) (ขั้นตอนที่ 844-845) |
| 083 | Legacy Analysis | `find_dead_paragraphs.sh`, Decision Table สกัดกฎธุรกิจ (ขั้นตอนที่ 843) |
| 084 | Migration Strategy + Golden Master | ADR เลือกกลยุทธ์, Golden Master Testing ตลอดโปรเจกต์ (ขั้นตอนที่ 841-842) |

### บทสรุปเชิง Retrospective (สิ่งที่เรียนรู้จากโปรเจกต์นี้)

1. **การวิเคราะห์ก่อนแก้ไขเสมอ** (Part 083) ทำให้เรารู้ล่วงหน้าว่ากฎธุรกิจมีกี่กรณี ก่อนที่จะเริ่ม
   ออกแบบ Copybook ค่าคงที่และ Unit Test
2. **Golden Master Testing เป็นตาข่ายนิรภัยที่ทรงพลังที่สุด** สำหรับการ Refactor แบบทีละก้าว — ทุกครั้ง
   ที่โครงสร้างโค้ดเปลี่ยนไปมาก (จาก 1 โปรแกรมเป็น 3 โปรแกรม) เราพิสูจน์ได้ทันทีว่าพฤติกรรมยังเหมือนเดิม
3. **Unit Test จับบั๊กที่ Golden Master พลาดได้** — กรณีขอบเขต (`10000.00` พอดี) ไม่ได้อยู่ใน Golden
   Master เดิมเลย แต่ Unit Test ที่ออกแบบมาเฉพาะจับมันได้ทันที นี่คือเหตุผลที่ต้องมีทั้งสองเทคนิคควบคู่กัน
4. **REST API ทำให้ COBOL เข้าถึงได้จากทุกที่** โดยที่ไม่ต้องแก้ไข Business Logic แม้แต่บรรทัดเดียว —
   พิสูจน์แนวคิด Wrap/Encapsulate จาก Part 077/084 ว่าใช้งานได้จริง
5. **CI Pipeline ทำให้ทุกการตรวจสอบเกิดขึ้นอัตโนมัติ** — ลดความเสี่ยงที่มนุษย์จะลืมรันการทดสอบบางอย่าง
   ก่อน Deploy

### ก้าวต่อไป: จาก "ปรับปรุงระบบเดียว" สู่ "สถาปัตยกรรมระดับองค์กร"

โปรเจกต์ AR-MINI ใน Part นี้เป็นตัวอย่างขนาดเล็กที่แสดงกระบวนการปรับปรุงระบบ Legacy แบบครบวงจร แต่ใน
โลกจริง องค์กรขนาดใหญ่มักมีระบบลักษณะนี้**นับร้อยนับพันระบบ**ทำงานร่วมกัน คำถามที่ตามมาตามธรรมชาติคือ:
เมื่อมีหลายระบบ (หลาย `ARCALC` หลาย `ARDISC`) ที่ต้องดูแล จะออกแบบสถาปัตยกรรมอย่างไรให้จัดการได้อย่างมี
ระเบียบ? Copybook ที่ใช้ร่วมกันหลายทีมจะควบคุมเวอร์ชันอย่างไร? ระบบ Batch ที่ต้องรันตามตารางเวลาจะจัด
โครงสร้างอย่างไร? — คำถามเหล่านี้คือจุดเริ่มต้นของ **เฟส 6: ระดับมืออาชีพ/โลก (Professional/World-class)**
ที่จะเริ่มต้นด้วย **Part 086: สถาปัตยกรรมระบบองค์กรขนาดใหญ่ด้วย COBOL**

### ข้อควรระวัง

- โปรเจกต์นี้จบลงด้วยระบบที่ "พร้อมทำงาน" แต่**ยังไม่ใช่ระบบ Production ที่สมบูรณ์** — ยังขาดองค์ประกอบ
  สำคัญอื่น ๆ ที่ระบบจริงต้องมี เช่น Logging, Monitoring, Authentication/Authorization บน REST API,
  และการจัดการ Concurrent Request ที่เขียนไฟล์ `BALANCE.DAT` พร้อมกันหลาย Request (Race Condition) —
  หัวข้อเหล่านี้จะถูกกล่าวถึงเพิ่มเติมในเฟส 6 ต่อไป
- อย่าลืมว่าทุกเทคนิคในโปรเจกต์นี้**ปรับขนาดได้** (scalable) — หลักการเดียวกัน (Golden Master, Extract
  Subprogram, Unit Test, REST Wrapper, CI Pipeline) ใช้ได้กับระบบขนาดใหญ่กว่านี้มาก เพียงแต่ต้องใช้เวลา
  และความระมัดระวังมากขึ้นตามสัดส่วนของขนาดระบบ

### แบบฝึกหัดที่ 850.1

**โจทย์**: จงเขียนสรุป 3-5 ประโยคของคุณเอง อธิบายว่าถ้าต้องอธิบายให้ผู้บริหารที่ไม่ใช่สายเทคนิคฟังว่า
โปรเจกต์ AR-MINI นี้ทำอะไรไปบ้าง และทำไมองค์กรควรลงทุนทำแบบนี้กับระบบ Legacy จริงของตัวเอง

**เฉลยแนวทาง**: คำตอบขึ้นกับผู้เรียนแต่ละคน แต่ควรมีใจความสำคัญคือ: เรานำระบบเก่าที่ทำงานถูกต้องอยู่แล้ว
มาปรับปรุงโครงสร้างภายในให้ดูแลง่ายขึ้น (แยกส่วนคำนวณออกจากส่วนบันทึกข้อมูล) โดย**ไม่เปลี่ยนพฤติกรรมการ
ทำงานแม้แต่น้อย** (พิสูจน์ได้ด้วยการทดสอบเปรียบเทียบผลลัพธ์ก่อน-หลังทุกขั้นตอน) จากนั้นเปิดให้ระบบใหม่ ๆ
(เว็บไซต์, แอปมือถือ) เรียกใช้งานระบบเก่านี้ได้ผ่านช่องทางมาตรฐาน (REST API) โดยไม่ต้องเขียนระบบคำนวณ
ทางการเงินที่ซับซ้อนขึ้นใหม่ตั้งแต่ต้น (ซึ่งมีความเสี่ยงสูงและใช้เวลานาน) สุดท้ายเราสร้างระบบตรวจสอบ
อัตโนมัติที่คอยยืนยันว่าทุกการแก้ไขในอนาคตจะไม่ทำให้ระบบพัง — องค์กรควรลงทุนทำแบบนี้เพราะมันช่วยลดความ
เสี่ยงในการปรับปรุงระบบเก่าที่สำคัญต่อธุรกิจ ในขณะที่ยังคงความเร็วในการนำเทคโนโลยีใหม่มาใช้งานได้

---

## สรุปท้ายบท

Part 085 คือจุดสูงสุดของเฟส 5: เราได้นำเทคนิคทั้งหมดจาก Part 071-084 มาประกอบร่างเป็นโปรเจกต์จริงที่
ทำงานได้สมบูรณ์แบบ ตั้งแต่ต้นจนจบ:

- วางแผนโปรเจกต์และบันทึก Architecture Decision Record ก่อนลงมือทำ (ขั้นตอนที่ 841)
- สร้างระบบ Legacy ตั้งต้นและบันทึก Golden Master ก่อนแตะโค้ดใด ๆ (ขั้นตอนที่ 842)
- นำเทคนิค Legacy Analysis จาก Part 083 มาใช้วิเคราะห์ระบบตั้งต้นซ้ำ (ขั้นตอนที่ 843)
- Refactor แยกตรรกะคำนวณเป็น `ARDISC.cob` พร้อมพิสูจน์ Regression ด้วย Golden Master (ขั้นตอนที่ 844)
- Refactor แยกส่วนจัดการไฟล์เป็น `ARLEDGER.cob` และประกอบเป็น `ARCALC.cob` พร้อมพิสูจน์ Regression แบบ
  เต็มระบบ (ขั้นตอนที่ 845)
- เขียน Unit Test ครอบคลุมกรณีขอบเขต และสาธิตการจับบั๊กจริงจากการ Refactor (ขั้นตอนที่ 846)
- ห่อหุ้มระบบด้วย REST API และทดสอบ End-to-End ด้วย `curl` จริง (ขั้นตอนที่ 847)
- สร้าง CI Pipeline ที่รวมทุกขั้นตอนเป็นอัตโนมัติ พร้อมสาธิตทั้งกรณีผ่านและกรณีล้มเหลว (ขั้นตอนที่ 848)
- วางแนวทาง Git Workflow สำหรับทีมที่ทำงานลักษณะนี้จริง (ขั้นตอนที่ 849)
- ทดสอบบูรณาการครั้งสุดท้ายและสรุปบทเรียนทั้งหมดของโปรเจกต์ (ขั้นตอนที่ 850)

**ทุกไฟล์ ทุกคำสั่ง และทุกผลลัพธ์ในเอกสารนี้ผ่านการทดสอบจริง** — นี่คือหลักฐานที่แสดงว่ากระบวนการ
Modernization ที่สอนไว้ตลอดเฟส 5 ไม่ใช่แค่ทฤษฎี แต่เป็นกระบวนการที่ลงมือทำได้จริงและให้ผลลัพธ์ที่พิสูจน์ได้

เฟส 5 "COBOL สมัยใหม่" จบลงแล้วอย่างสมบูรณ์ ต่อจากนี้หลักสูตรจะก้าวเข้าสู่ **เฟส 6: ระดับมืออาชีพ/โลก
(Professional/World-class)** ซึ่งจะขยายขอบเขตจาก "ปรับปรุงระบบเดียว" ไปสู่ "สถาปัตยกรรมระดับองค์กรที่มี
หลายสิบหลายร้อยระบบทำงานร่วมกัน" เริ่มต้นด้วย **Part 086: สถาปัตยกรรมระบบองค์กรขนาดใหญ่ด้วย COBOL**

**[ไปยัง Part 086: สถาปัตยกรรมระบบองค์กรขนาดใหญ่ด้วย COBOL →](part-086-enterprise-architecture.md)**
