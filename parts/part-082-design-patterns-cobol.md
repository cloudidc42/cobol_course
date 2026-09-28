# Part 082: Design Patterns ในโลก COBOL (ขั้นตอนที่ 811–820)

## คำนำของ Part นี้

Part 081 สอนให้เราปรับปรุงโครงสร้างภายในของโค้ด COBOL ด้วยเทคนิค Refactoring ต่าง ๆ จนได้โค้ดที่
สะอาด อ่านง่าย และแยกเป็น subprogram ที่ทดสอบได้อิสระ ถึงตอนนี้เรามีเครื่องมือครบมือแล้วสำหรับก้าว
ต่อไป: การนำ **Design Patterns** — รูปแบบการแก้ปัญหาการออกแบบซอฟต์แวร์ที่เกิดซ้ำบ่อย ๆ ซึ่งเป็นที่
รู้จักกันดีในโลกของภาษา Object-Oriented อย่าง Java และ C++ — มาประยุกต์ใช้กับ COBOL ที่เป็นภาษา
เชิงกระบวนการ (Procedural) ล้วน ๆ

หลายคนอาจสงสัยว่า "Design Pattern ที่ออกแบบมาสำหรับ OOP จะใช้กับ COBOL ที่ไม่มี class/object ได้
อย่างไร" คำตอบคือ **Design Pattern ไม่ใช่ไวยากรณ์เฉพาะภาษา แต่เป็นแนวคิดการแก้ปัญหาการออกแบบที่เป็น
นามธรรม** — สิ่งที่ Java ทำด้วย interface และ polymorphism, COBOL สามารถทำได้ด้วยกลไกที่มีอยู่แล้ว
ตั้งแต่ Part 031 (`CALL`), Part 011/041 (`EVALUATE`), และคุณสมบัติของ WORKING-STORAGE ที่คงค่าข้าม
การเรียก (ทบทวนจาก Part 031 ขั้นตอนที่ 309)

Part นี้จะพาคุณสร้าง**ตัวอย่างที่คอมไพล์และรันได้จริง** สำหรับ 4 pattern หลัก: **Strategy**,
**Factory**, **Singleton**, และ **Template Method** — ทุกตัวอย่างผ่านการทดสอบรันจริงแล้วทั้งหมด

---

## ขั้นตอนที่ 811: แนวคิด Design Patterns ในบริบทของภาษาเชิงกระบวนการ

### Design Pattern คืออะไรกันแน่

**Design Pattern** คือ**แนวทางแก้ปัญหาการออกแบบที่ถูกพิสูจน์แล้วว่าได้ผลดี** สำหรับปัญหาที่เกิดซ้ำ ๆ
ในการออกแบบซอฟต์แวร์ — มันไม่ใช่โค้ดสำเร็จรูปที่ copy-paste ได้ตรง ๆ แต่เป็น**แนวคิด**ที่ต้องแปลง
เป็นโค้ดจริงให้เหมาะกับภาษาและบริบทของแต่ละโปรเจกต์ หนังสือคลาสสิกที่รวบรวม pattern เหล่านี้ไว้คือ
"Design Patterns: Elements of Reusable Object-Oriented Software" (1994) โดยกลุ่มผู้เขียนที่รู้จัก
กันในชื่อ **"Gang of Four" (GoF)**

### ทำไม Pattern ที่ออกแบบมาสำหรับ OOP ถึงยังใช้ได้กับ COBOL

หัวใจของ Design Pattern ส่วนใหญ่คือการแก้ปัญหา**"จะทำอย่างไรให้ระบบยืดหยุ่นและขยายได้โดยไม่ต้อง
แก้ไขโค้ดเดิมทุกจุด"** — ปัญหานี้ไม่ได้ผูกติดกับ OOP โดยเฉพาะ COBOL มีกลไกที่ทำหน้าที่คล้ายกันได้:

| กลไกใน OOP | กลไกเทียบเท่าใน COBOL | ตัวอย่างที่จะสอนใน Part นี้ |
|---|---|---|
| Interface / Abstract Class | Subprogram ที่มี "สัญญา" พารามิเตอร์เหมือนกัน (`LINKAGE SECTION` เดียวกัน) | ขั้นตอนที่ 812 |
| Polymorphism (เลือก implementation ตอน runtime) | `CALL` ด้วยชื่อโปรแกรมที่เก็บในตัวแปร (ทบทวนจาก Part 031 ขั้นตอนที่ 305) | ขั้นตอนที่ 813 |
| Object ที่มี state ส่วนตัว | WORKING-STORAGE ของ subprogram ที่คงค่าข้ามการเรียก (Part 031 ขั้นตอนที่ 309) | ขั้นตอนที่ 815 |
| Constructor / Factory Method | Subprogram ที่สร้าง record พร้อมค่าเริ่มต้นที่ถูกต้อง | ขั้นตอนที่ 814 |

### ข้อควรระวังสำคัญที่สุดของ Part นี้

**อย่า force ใช้ Design Pattern เพียงเพราะ "มันเท่" หรือ "ภาษาอื่นเขาทำกัน"** — Pattern มีไว้แก้
ปัญหาที่มีอยู่จริง ถ้าโปรแกรมง่าย ๆ ไม่มีความจำเป็นต้องขยายหรือเปลี่ยนแปลงพฤติกรรมตาม runtime เลย
การเขียน `IF`/`EVALUATE` ตรง ๆ แบบที่ Part 081 สอนไว้ (Clean Code แบบพื้นฐาน) ก็เพียงพอและเรียบง่าย
กว่ามาก ประเด็นนี้จะขยายความในขั้นตอนที่ 819

### แบบฝึกหัดที่ 811.1

**โจทย์**: จงอธิบายว่าทำไมแนวคิด "ยืดหยุ่นและขยายได้โดยไม่ต้องแก้ไขโค้ดเดิม" ถึงเป็นแก่นของ Design
Pattern ส่วนใหญ่ มากกว่าจะเป็นเรื่องของ syntax เฉพาะภาษาใดภาษาหนึ่ง

**เฉลย**: Design Pattern ถูกคิดค้นขึ้นมาจากการสังเกตปัญหาที่เกิดซ้ำในการออกแบบซอฟต์แวร์ ไม่ว่าจะ
เขียนด้วยภาษาอะไรก็ตาม เช่น ปัญหา "เมื่อมีอัลกอริทึมหลายแบบที่สลับใช้กันได้ จะเพิ่มอัลกอริทึมใหม่
โดยไม่ต้องแก้โค้ดที่เรียกใช้อัลกอริทึมเดิมได้อย่างไร" (ซึ่งคือปัญหาที่ Strategy Pattern แก้) ปัญหานี้
เกิดขึ้นทั้งใน Java ที่ใช้ interface, ใน Python ที่ใช้ first-class function, และใน COBOL ที่ใช้
subprogram dispatch ต่างกันแค่**กลไกทางไวยากรณ์**ที่แต่ละภาษาใช้แก้ปัญหาเดียวกันนี้เท่านั้น หัวใจ
ของ pattern คือ**โครงสร้างความสัมพันธ์ระหว่างส่วนต่าง ๆ ของระบบ** (เช่น "ผู้เรียกไม่ควรรู้รายละเอียด
ของแต่ละอัลกอริทึม") ซึ่งเป็นแนวคิดระดับสถาปัตยกรรมที่อยู่เหนือกว่าไวยากรณ์ของภาษาใดภาษาหนึ่ง ทำให้
สามารถนำมาประยุกต์ใช้ข้ามภาษาที่มีธรรมชาติต่างกันโดยสิ้นเชิงได้ ตราบใดที่ภาษานั้นมีกลไกพื้นฐานที่
เพียงพอ (เช่น การเรียกโค้ดแบบ indirect ผ่านชื่อที่กำหนดตอน runtime)

---

## ขั้นตอนที่ 812: Strategy Pattern ผ่าน EVALUATE และ Subprogram Dispatch

### ปัญหาที่ Strategy Pattern แก้

สมมติระบบต้องคำนวณค่าขนส่งสินค้า โดยมีหลายวิธีคำนวณ (มาตรฐาน, ด่วน, ระหว่างประเทศ) ที่มีสูตรต่างกัน
สิ้นเชิง — **Strategy Pattern** ให้เราแยกแต่ละ "วิธีคำนวณ" (algorithm) ออกเป็นหน่วยอิสระที่มี
**"สัญญา" (interface) เดียวกัน** แล้วให้ผู้เรียกเลือกใช้วิธีไหนก็ได้โดยไม่ต้องรู้รายละเอียดภายใน
ของแต่ละวิธี

### สร้าง Strategy แต่ละแบบเป็น Subprogram แยก

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STDSHIPPING.
       AUTHOR. COBOL-COURSE.

      *> One STRATEGY: standard shipping cost calculation.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-WEIGHT-KG             PIC 9(3)V99.
       01  LK-COST                  PIC 9(5)V99.

       PROCEDURE DIVISION USING LK-WEIGHT-KG LK-COST.
       MAIN-PARA.
           COMPUTE LK-COST ROUNDED = 20.00 + (LK-WEIGHT-KG * 5.00).
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EXPSHIPPING.
       AUTHOR. COBOL-COURSE.

      *> Another STRATEGY: express shipping cost calculation - same
      *> interface (weight in, cost out) but a completely different
      *> algorithm, and the caller does not need to know which.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-WEIGHT-KG             PIC 9(3)V99.
       01  LK-COST                  PIC 9(5)V99.

       PROCEDURE DIVISION USING LK-WEIGHT-KG LK-COST.
       MAIN-PARA.
           COMPUTE LK-COST ROUNDED = 100.00 + (LK-WEIGHT-KG * 15.00).
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INTSHIPPING.
       AUTHOR. COBOL-COURSE.

      *> Third STRATEGY: international shipping.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-WEIGHT-KG             PIC 9(3)V99.
       01  LK-COST                  PIC 9(5)V99.

       PROCEDURE DIVISION USING LK-WEIGHT-KG LK-COST.
       MAIN-PARA.
           COMPUTE LK-COST ROUNDED = 500.00 + (LK-WEIGHT-KG * 40.00).
           GOBACK.
```

### โปรแกรมหลัก: Dispatch ด้วย EVALUATE

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP812MAIN.
       AUTHOR. COBOL-COURSE.

      *> STRATEGY PATTERN: the caller picks a shipping METHOD code and
      *> dispatches to a matching subprogram via EVALUATE. Each
      *> subprogram implements the SAME interface (weight in, cost
      *> out) but a different algorithm - swapping strategies is just
      *> a matter of the EVALUATE branch chosen, not rewriting logic.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SHIP-METHOD           PIC X(3).
       01  WS-WEIGHT-KG             PIC 9(3)V99.
       01  WS-COST                  PIC 9(5)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 10.00 TO WS-WEIGHT-KG.

           MOVE "STD" TO WS-SHIP-METHOD.
           PERFORM DISPATCH-SHIPPING-STRATEGY.
           DISPLAY WS-SHIP-METHOD " weight=" WS-WEIGHT-KG
               " cost=" WS-COST.

           MOVE "EXP" TO WS-SHIP-METHOD.
           PERFORM DISPATCH-SHIPPING-STRATEGY.
           DISPLAY WS-SHIP-METHOD " weight=" WS-WEIGHT-KG
               " cost=" WS-COST.

           MOVE "INT" TO WS-SHIP-METHOD.
           PERFORM DISPATCH-SHIPPING-STRATEGY.
           DISPLAY WS-SHIP-METHOD " weight=" WS-WEIGHT-KG
               " cost=" WS-COST.
           STOP RUN.

       DISPATCH-SHIPPING-STRATEGY.
           EVALUATE WS-SHIP-METHOD
               WHEN "STD"
                   CALL "STDSHIPPING" USING WS-WEIGHT-KG WS-COST
               WHEN "EXP"
                   CALL "EXPSHIPPING" USING WS-WEIGHT-KG WS-COST
               WHEN "INT"
                   CALL "INTSHIPPING" USING WS-WEIGHT-KG WS-COST
               WHEN OTHER
                   MOVE 0 TO WS-COST
           END-EVALUATE.
```

**คอมไพล์และรันจริง:**

```bash
cobc -x -o step812main step812main.cob stdshipping.cob expshipping.cob intshipping.cob
./step812main
```

**ผลลัพธ์จริง (ยืนยันด้วยการรัน):**

```
STD weight=010.00 cost=00070.00
EXP weight=010.00 cost=00250.00
INT weight=010.00 cost=00900.00
```

(ตรวจสอบด้วยมือ: STD = 20 + 10×5 = 70.00; EXP = 100 + 10×15 = 250.00; INT = 500 + 10×40 = 900.00 ✔)

### อธิบายจุดสำคัญ

- ทั้ง 3 subprogram (`STDSHIPPING`, `EXPSHIPPING`, `INTSHIPPING`) มี **"สัญญา" (interface) เดียวกัน
  เป๊ะ**: รับ `LK-WEIGHT-KG` เข้า คืน `LK-COST` ออก แม้สูตรคำนวณภายในจะต่างกันโดยสิ้นเชิง — นี่คือ
  หัวใจของ Strategy Pattern: **ผู้เรียกไม่จำเป็นต้องรู้เลยว่าแต่ละ strategy คำนวณอย่างไรภายใน**
  รู้แค่ว่าส่ง weight เข้าไปแล้วจะได้ cost กลับมา
- `DISPATCH-SHIPPING-STRATEGY` (paragraph ที่ทำหน้าที่เลือก strategy) เป็น**จุดเดียว**ในระบบที่รู้จัก
  ชื่อโปรแกรมทั้ง 3 ตัว — ถ้าต้องการเพิ่ม strategy ใหม่ (เช่น "ECO" สำหรับขนส่งแบบประหยัดพลังงาน)
  จะต้องแก้ไขแค่ paragraph นี้เพียงจุดเดียว โดยไม่ต้องแตะ `MAIN-PARA` หรือ strategy ที่มีอยู่เดิม
  เลยแม้แต่บรรทัดเดียว
- เทียบกับภาษา OOP: `STDSHIPPING`, `EXPSHIPPING`, `INTSHIPPING` เทียบเท่ากับ**คลาสที่ implement
  interface เดียวกัน** (เช่น `ShippingStrategy`) และ `DISPATCH-SHIPPING-STRATEGY` เทียบเท่ากับ
  **Context object** ที่เลือกใช้ strategy ตัวใดตัวหนึ่งตาม runtime

### ข้อควรระวัง

- Strategy Pattern แบบ `EVALUATE` ยังคงต้อง**แก้ไข paragraph dispatch ทุกครั้งที่เพิ่ม strategy
  ใหม่** — ยังไม่ใช่การขยายระบบแบบ "เพิ่มได้โดยไม่แตะโค้ดเดิมเลย" อย่างสมบูรณ์ (ขั้นตอนที่ 813 จะ
  แก้ข้อจำกัดนี้ด้วยวิธี data-driven dispatch)
- ต้องรักษาวินัยให้ทุก strategy subprogram มี**พารามิเตอร์ที่ตรงกันทุกประการ** (ทบทวนความเสี่ยงจาก
  Part 031 ขั้นตอนที่ 304) เพราะ COBOL compiler ไม่มีทางตรวจสอบให้ว่า subprogram ทั้ง 3 ตัวที่ควร
  "implement interface เดียวกัน" จะมี `PROCEDURE DIVISION USING` ตรงกันจริงหรือไม่ — เป็นภาระของ
  ทีมพัฒนาที่ต้องดูแลเอง (แนะนำให้เขียนเอกสารระบุ "สัญญา" ของแต่ละ strategy ไว้ชัดเจน)

### แบบฝึกหัดที่ 812.1

**โจทย์**: จงออกแบบและเขียน subprogram strategy ใหม่ชื่อ `ECOSHIPPING` ที่คิดค่าส่งฐาน 10.00 บวก
น้ำหนักคูณ 3.00 บาทต่อกิโลกรัม แล้วเพิ่มเข้าไปใน `DISPATCH-SHIPPING-STRATEGY`

**เฉลย**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ECOSHIPPING.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-WEIGHT-KG             PIC 9(3)V99.
       01  LK-COST                  PIC 9(5)V99.

       PROCEDURE DIVISION USING LK-WEIGHT-KG LK-COST.
       MAIN-PARA.
           COMPUTE LK-COST ROUNDED = 10.00 + (LK-WEIGHT-KG * 3.00).
           GOBACK.
```

และแก้ไข `DISPATCH-SHIPPING-STRATEGY`:

```cobol
       DISPATCH-SHIPPING-STRATEGY.
           EVALUATE WS-SHIP-METHOD
               WHEN "STD"
                   CALL "STDSHIPPING" USING WS-WEIGHT-KG WS-COST
               WHEN "EXP"
                   CALL "EXPSHIPPING" USING WS-WEIGHT-KG WS-COST
               WHEN "INT"
                   CALL "INTSHIPPING" USING WS-WEIGHT-KG WS-COST
               WHEN "ECO"
                   CALL "ECOSHIPPING" USING WS-WEIGHT-KG WS-COST
               WHEN OTHER
                   MOVE 0 TO WS-COST
           END-EVALUATE.
```

สังเกตว่า `ECOSHIPPING` มี interface เดียวกันทุกประการกับ strategy อื่น (`LK-WEIGHT-KG` เข้า,
`LK-COST` ออก) ทำให้เสียบเข้าระบบ dispatch เดิมได้ทันทีโดยไม่ต้องแก้ไขโครงสร้างอื่นใดเลย นี่คือ
ประโยชน์ของการรักษา "สัญญา" ให้เหมือนกันระหว่างทุก strategy ตามที่ pattern นี้ต้องการ

---

## ขั้นตอนที่ 813: Strategy Pattern แบบ Data-Driven — เพิ่ม Strategy โดยไม่แก้ EVALUATE

### ข้อจำกัดของ EVALUATE Dispatch ที่ต้องแก้ไข

จากขั้นตอนที่ 812 ทุกครั้งที่เพิ่ม strategy ใหม่ ต้องแก้ไข paragraph `DISPATCH-SHIPPING-STRATEGY`
เสมอ ขั้นตอนนี้จะแสดงเทคนิคที่ก้าวหน้ากว่า: ใช้**ตาราง (table)** จับคู่ "รหัส method" กับ "ชื่อ
โปรแกรม" แล้วค้นหาและ `CALL` ด้วยชื่อที่เก็บในตัวแปร (ทบทวนเทคนิคนี้จาก Part 031 ขั้นตอนที่ 305)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP813MAIN.
       AUTHOR. COBOL-COURSE.

      *> STRATEGY PATTERN, data-driven version: instead of an
      *> EVALUATE that must be edited every time a new strategy is
      *> added, a TABLE maps each method code to a program name. To
      *> add a new shipping strategy, add one row to the table - the
      *> dispatch logic itself never changes again.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STRATEGY-TABLE.
           05  FILLER.
               10  FILLER PIC X(3) VALUE "STD".
               10  FILLER PIC X(15) VALUE "STDSHIPPING".
           05  FILLER.
               10  FILLER PIC X(3) VALUE "EXP".
               10  FILLER PIC X(15) VALUE "EXPSHIPPING".
           05  FILLER.
               10  FILLER PIC X(3) VALUE "INT".
               10  FILLER PIC X(15) VALUE "INTSHIPPING".
       01  WS-STRATEGY-TABLE-R REDEFINES WS-STRATEGY-TABLE.
           05  WS-STRATEGY-ENTRY OCCURS 3 TIMES
                   INDEXED BY WS-STRAT-IDX.
               10  WS-STRAT-METHOD  PIC X(3).
               10  WS-STRAT-PROGRAM PIC X(15).

       01  WS-SHIP-METHOD           PIC X(3).
       01  WS-PROGRAM-NAME          PIC X(15).
       01  WS-WEIGHT-KG             PIC 9(3)V99 VALUE 10.00.
       01  WS-COST                  PIC 9(5)V99.
       01  WS-FOUND-FLAG            PIC X(1) VALUE "N".
           88  STRATEGY-FOUND               VALUE "Y".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "STD" TO WS-SHIP-METHOD.
           PERFORM RUN-STRATEGY.

           MOVE "EXP" TO WS-SHIP-METHOD.
           PERFORM RUN-STRATEGY.

           MOVE "INT" TO WS-SHIP-METHOD.
           PERFORM RUN-STRATEGY.

           MOVE "ZZZ" TO WS-SHIP-METHOD.
           PERFORM RUN-STRATEGY.
           STOP RUN.

       RUN-STRATEGY.
           MOVE "N" TO WS-FOUND-FLAG.
           SEARCH WS-STRATEGY-ENTRY
               AT END
                   DISPLAY "no strategy registered for ["
                       WS-SHIP-METHOD "]"
               WHEN WS-STRAT-METHOD(WS-STRAT-IDX) = WS-SHIP-METHOD
                   MOVE "Y" TO WS-FOUND-FLAG
                   MOVE WS-STRAT-PROGRAM(WS-STRAT-IDX)
                       TO WS-PROGRAM-NAME
           END-SEARCH.
           IF STRATEGY-FOUND
               CALL WS-PROGRAM-NAME USING WS-WEIGHT-KG WS-COST
               DISPLAY WS-SHIP-METHOD " -> " WS-PROGRAM-NAME
                   " cost=" WS-COST
           END-IF.
```

**คอมไพล์และรันจริง:**

```bash
cobc -x -o step813main step813main.cob stdshipping.cob expshipping.cob intshipping.cob
./step813main
```

**ผลลัพธ์จริง (ยืนยันด้วยการรัน — รวมกรณีที่ไม่พบ strategy ด้วย):**

```
STD -> STDSHIPPING     cost=00070.00
EXP -> EXPSHIPPING     cost=00250.00
INT -> INTSHIPPING     cost=00900.00
no strategy registered for [ZZZ]
```

### อธิบายจุดสำคัญ

- `WS-STRATEGY-TABLE` (ทบทวน `OCCURS`/`REDEFINES`/`INDEXED BY`/`SEARCH` จาก Part 016–017, 022)
  เก็บคู่ "รหัส method → ชื่อโปรแกรม" เป็นข้อมูล ไม่ใช่ตรรกะ — การเพิ่ม strategy ใหม่กลายเป็นแค่
  **เพิ่มแถวข้อมูลในตาราง** ไม่ต้องแก้ไข paragraph `RUN-STRATEGY` เลยแม้แต่บรรทัดเดียว
- `SEARCH WS-STRATEGY-ENTRY` ค้นหาแถวที่ `WS-STRAT-METHOD` ตรงกับ `WS-SHIP-METHOD` ที่ต้องการ แล้ว
  ดึงชื่อโปรแกรมมาเก็บไว้ที่ `WS-PROGRAM-NAME` — จากนั้น `CALL WS-PROGRAM-NAME USING ...` เป็นการ
  `CALL` ด้วยชื่อในตัวแปร (Dynamic-style Dispatch) ที่ทบทวนมาจาก Part 031 ขั้นตอนที่ 305
- กรณี `"ZZZ"` ที่ไม่มีในตาราง แสดงให้เห็นการจัดการ**กรณี strategy ไม่พบ**อย่างปลอดภัยผ่าน `AT END`
  ของ `SEARCH` — ไม่มีการ `CALL` โปรแกรมที่ไม่มีอยู่จริงเลย (ซึ่งจะทำให้เกิด runtime error ทันที
  ถ้าพลาดจุดนี้ไป)

### ข้อควรระวัง

- วิธี data-driven นี้เพิ่มความยืดหยุ่นแต่ก็เพิ่ม**ความซับซ้อนของโค้ด**ด้วย (ต้องเข้าใจ `SEARCH`,
  `REDEFINES`, `INDEXED BY` พร้อมกัน) — สำหรับระบบที่มี strategy แค่ 2–3 แบบและไม่ค่อยเปลี่ยนแปลง
  การใช้ `EVALUATE` ตรง ๆ แบบขั้นตอนที่ 812 อาจเรียบง่ายและเพียงพอกว่า ควรเลือกใช้ตามขนาดและอัตรา
  การเปลี่ยนแปลงของระบบจริง ไม่ใช่เลือกเพราะ "ซับซ้อนกว่าดูเท่กว่า"
  (ทบทวนคำเตือนจากขั้นตอนที่ 811)
  ต้องระวังไม่ใช้ `SEARCH` โดยไม่มี `AT END` (ถ้าไม่พบข้อมูลที่ค้นหา และไม่มี `AT END` จัดการ
  โปรแกรมจะไหลผ่าน `SEARCH` ไปโดยไม่ทำอะไรเลยและค่าตัวแปรที่เกี่ยวข้องอาจค้างจากรอบก่อนหน้า)
- ชื่อโปรแกรมที่เก็บในตาราง (`WS-STRAT-PROGRAM PIC X(15)`) ต้องมีความกว้างเพียงพอสำหรับชื่อโปรแกรม
  ที่ยาวที่สุดที่จะใช้เสมอ (ทบทวนกับดักเรื่องความกว้างจาก Part 031 ขั้นตอนที่ 305)

### แบบฝึกหัดที่ 813.1

**โจทย์**: จงอธิบายว่าทำไมการเพิ่ม strategy ใหม่ในเวอร์ชัน data-driven (ขั้นตอนนี้) ถึง**ไม่ต้อง
แก้ไข** paragraph `RUN-STRATEGY` เลย ในขณะที่เวอร์ชัน `EVALUATE` (ขั้นตอนที่ 812) ต้องแก้ไข
paragraph `DISPATCH-SHIPPING-STRATEGY` ทุกครั้ง

**เฉลย**: ในเวอร์ชัน `EVALUATE`, ความสัมพันธ์ระหว่าง "รหัส method" กับ "ชื่อโปรแกรมที่จะ CALL" ถูก
**เขียนฝังอยู่ในโค้ดโดยตรง** (`WHEN "STD" CALL "STDSHIPPING" ...`) ทำให้การเพิ่มความสัมพันธ์ใหม่
ต้องเพิ่มบรรทัดโค้ดใหม่เข้าไปในตัวโปรแกรมเสมอ ในขณะที่เวอร์ชัน data-driven, ความสัมพันธ์เดียวกันนี้
ถูกเก็บเป็น**ข้อมูล**ในตาราง `WS-STRATEGY-TABLE` แยกต่างหากจากตรรกะการค้นหาและเรียกใช้ (`SEARCH`
+ `CALL WS-PROGRAM-NAME`) ทำให้ paragraph `RUN-STRATEGY` ทำหน้าที่แค่ **"ค้นหาแล้วเรียก"** อย่าง
เดียวโดยไม่สนใจว่าตารางมีกี่แถวหรือมีความสัมพันธ์อะไรอยู่บ้าง การเพิ่มความสัมพันธ์ใหม่จึงเป็นการ
**เพิ่มข้อมูล** (แถวใหม่ในตาราง) แทนที่จะเป็นการ**เพิ่มโค้ด** ซึ่งเป็นหลักการที่เรียกว่า "แยกข้อมูล
ออกจากตรรกะ" (Separation of Data and Logic) ที่เป็นรากฐานสำคัญของการออกแบบระบบที่ขยายง่าย

---

## ขั้นตอนที่ 814: Factory-like Pattern ผ่าน Subprogram สร้าง Record

### ปัญหาที่ Factory Pattern แก้

เมื่อการสร้างข้อมูลชนิดหนึ่ง ๆ (เช่น บัญชีธนาคาร) มี**กฎการกำหนดค่าเริ่มต้นที่ซับซ้อนและแตกต่างกัน
ไปตามประเภท** (บัญชีออมทรัพย์มีอัตราดอกเบี้ย ยอดขั้นต่ำต่างจากบัญชีกระแสรายวัน) การให้ผู้เรียกแต่ละ
จุดในระบบต้องจำกฎเหล่านี้เองและกำหนดค่าด้วยมือทุกครั้งจะเสี่ยงต่อความผิดพลาดและความไม่สอดคล้องกัน
สูงมาก **Factory Pattern** แก้ปัญหานี้ด้วยการรวมกฎการสร้างทั้งหมดไว้ที่**จุดเดียว** (subprogram
"โรงงาน") ที่ผู้เรียกแค่บอกว่า "อยากได้ประเภทไหน" แล้วรับ record ที่สมบูรณ์กลับมาทันที

### AccountFactory: Subprogram ที่สร้าง Record ตามประเภท

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ACCOUNTFACTORY.
       AUTHOR. COBOL-COURSE.

      *> FACTORY-LIKE PATTERN: the caller does not build an account
      *> record field by field - it asks this subprogram to "create"
      *> one of a given TYPE, and receives back a fully-populated
      *> record with the right defaults for that type. Callers never
      *> need to know the construction rules for each account type.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-ACCOUNT-TYPE          PIC X(1).
       01  LK-ACCOUNT-ID            PIC 9(5).
       01  LK-NEW-ACCOUNT.
           05  LK-ACC-NUMBER        PIC X(8).
           05  LK-ACC-TYPE-OUT      PIC X(1).
           05  LK-INTEREST-RATE     PIC 9V9999.
           05  LK-MIN-BALANCE       PIC 9(5)V99.
           05  LK-BALANCE           PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-ACCOUNT-TYPE LK-ACCOUNT-ID
               LK-NEW-ACCOUNT.
       MAIN-PARA.
           MOVE LK-ACCOUNT-TYPE TO LK-ACC-TYPE-OUT.
           MOVE 0 TO LK-BALANCE.
           EVALUATE LK-ACCOUNT-TYPE
               WHEN "S"
                   STRING "SA" LK-ACCOUNT-ID DELIMITED BY SIZE
                       INTO LK-ACC-NUMBER
                   MOVE 0.0150 TO LK-INTEREST-RATE
                   MOVE 500.00 TO LK-MIN-BALANCE
               WHEN "C"
                   STRING "CH" LK-ACCOUNT-ID DELIMITED BY SIZE
                       INTO LK-ACC-NUMBER
                   MOVE 0.0000 TO LK-INTEREST-RATE
                   MOVE 0.00 TO LK-MIN-BALANCE
               WHEN OTHER
                   MOVE SPACES TO LK-ACC-NUMBER
                   MOVE 0.0000 TO LK-INTEREST-RATE
                   MOVE 0.00 TO LK-MIN-BALANCE
                   MOVE 1 TO RETURN-CODE
           END-EVALUATE.
           GOBACK.
```

### โปรแกรมหลักที่เรียกใช้ Factory

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP814MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ACCOUNT-TYPE          PIC X(1).
       01  WS-NEW-ID                PIC 9(5).
       01  WS-ACCOUNT.
           05  WS-ACC-NUMBER        PIC X(8).
           05  WS-ACC-TYPE          PIC X(1).
           05  WS-INTEREST-RATE     PIC 9V9999.
           05  WS-MIN-BALANCE       PIC 9(5)V99.
           05  WS-BALANCE           PIC 9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "S" TO WS-ACCOUNT-TYPE.
           MOVE 10001 TO WS-NEW-ID.
           CALL "ACCOUNTFACTORY" USING WS-ACCOUNT-TYPE WS-NEW-ID
               WS-ACCOUNT.
           DISPLAY "Created: " WS-ACC-NUMBER " type=" WS-ACC-TYPE
               " rate=" WS-INTEREST-RATE " min=" WS-MIN-BALANCE.

           MOVE "C" TO WS-ACCOUNT-TYPE.
           MOVE 20002 TO WS-NEW-ID.
           CALL "ACCOUNTFACTORY" USING WS-ACCOUNT-TYPE WS-NEW-ID
               WS-ACCOUNT.
           DISPLAY "Created: " WS-ACC-NUMBER " type=" WS-ACC-TYPE
               " rate=" WS-INTEREST-RATE " min=" WS-MIN-BALANCE.
           STOP RUN.
```

**คอมไพล์และรันจริง:**

```bash
cobc -x -o step814main step814main.cob accountfactory.cob
./step814main
```

**ผลลัพธ์จริง (ยืนยันด้วยการรัน):**

```
Created: SA10001  type=S rate=0.0150 min=00500.00
Created: CH20002  type=C rate=0.0000 min=00000.00
```

### กับดักจริงที่พบระหว่างพัฒนาตัวอย่างนี้ (บทเรียนสำคัญ)

ระหว่างการพัฒนาตัวอย่างนี้ ได้ลองเรียก `CALL "ACCOUNTFACTORY" USING "S" 10001 WS-ACCOUNT` โดยส่ง
**ตัวเลข literal (`10001`) ตรง ๆ แทนที่จะเป็นตัวแปร** เข้าไปเป็นพารามิเตอร์ที่สองแทน `WS-NEW-ID`
— ผลลัพธ์ที่ได้จริงกลับเป็นข้อมูลขยะที่อ่านไม่ได้เลย (`SA  'H` แทนที่จะเป็น `SA10001`)

**บทเรียนนี้ยืนยันคำเตือนจาก Part 031 ขั้นตอนที่ 305 อีกครั้งอย่างเป็นรูปธรรม**: การส่ง numeric
literal ตรง ๆ เข้า `CALL ... USING` มีความเสี่ยงที่จะได้ผลลัพธ์ที่ไม่ตรงกับที่ subprogram คาดหวัง
เพราะการแทนค่า literal ภายในของ compiler อาจไม่ตรงกับ PICTURE ที่ subprogram ประกาศไว้ใน
`LINKAGE SECTION` เป๊ะเสมอไป **วิธีที่ปลอดภัย 100% คือย้ายค่าเข้าตัวแปร WORKING-STORAGE ที่มี
PICTURE ตรงกันก่อนเสมอ** (ตามที่โค้ดตัวอย่างที่ถูกต้องข้างบนแสดงไว้ — ใช้ `WS-NEW-ID` แทนการพิมพ์
`10001` ตรง ๆ)

### อธิบายจุดสำคัญ

- `STRING "SA" LK-ACCOUNT-ID DELIMITED BY SIZE INTO LK-ACC-NUMBER` (ทบทวนจาก Part 019) ต่อคำนำหน้า
  (`"SA"` หรือ `"CH"`) เข้ากับเลขบัญชีเพื่อสร้างเลขที่บัญชีที่มีรูปแบบสอดคล้องกับประเภทบัญชีโดย
  อัตโนมัติ — นี่คือส่วนหนึ่งของ "กฎการสร้าง" ที่ผู้เรียกไม่จำเป็นต้องรู้เลย
- ผู้เรียก (`STEP814MAIN`) **ไม่จำเป็นต้องรู้เลย**ว่าบัญชีออมทรัพย์ต้องมีอัตราดอกเบี้ย 1.5% และ
  ยอดขั้นต่ำ 500 บาท หรือบัญชีกระแสรายวันไม่มีดอกเบี้ยเลย — กฎทั้งหมดนี้ถูกห่อหุ้ม (encapsulate)
  ไว้ใน `ACCOUNTFACTORY` เพียงจุดเดียว ทำให้ถ้ากฎเปลี่ยนแปลง (เช่น ธนาคารปรับอัตราดอกเบี้ยใหม่)
  ต้องแก้ไขแค่ที่เดียวเท่านั้น (เชื่อมโยงกับหลักการ Extract Subprogram จาก Part 081 ขั้นตอนที่ 805)
- `MOVE 1 TO RETURN-CODE` ใน `WHEN OTHER`: Factory ปฏิเสธการสร้างบัญชีประเภทที่ไม่รู้จักอย่างชัดเจน
  แทนที่จะสร้าง record ที่มีค่าผิดเพี้ยนออกมาเงียบ ๆ (ทบทวนแนวคิด validation จาก Part 079)

### ข้อควรระวัง

- **ระวังการส่ง literal ตรง ๆ เข้า `CALL ... USING`** ตามที่พิสูจน์ในกับดักข้างต้น — ควรใช้ตัวแปร
  ที่มี PICTURE ตรงกับ `LINKAGE SECTION` ของ subprogram เสมอ ไม่ว่าค่าที่จะส่งจะดู "ตายตัว" แค่ไหน
  ก็ตาม
- Factory subprogram ที่มีหลายประเภทมาก (มากกว่า 4–5 ประเภท) ควรพิจารณาผสมผสานกับเทคนิค data-driven
  dispatch จากขั้นตอนที่ 813 เพื่อไม่ให้ `EVALUATE` ภายในยาวเกินไปจนอ่านยาก

### แบบฝึกหัดที่ 814.1

**โจทย์**: จงอธิบายว่าทำไมการที่ `ACCOUNTFACTORY` ตั้งค่า `LK-BALANCE` เป็น `0` เสมอไม่ว่าประเภท
บัญชีจะเป็นอะไร (นอกเหนือจาก `EVALUATE`) ถึงเป็นการออกแบบที่เหมาะสมสำหรับ subprogram ประเภท
"สร้าง record ใหม่"

**เฉลย**: บัญชีธนาคารใหม่ที่เพิ่งเปิด**ควรมียอดเงินเริ่มต้นเป็นศูนย์เสมอ**ไม่ว่าจะเป็นประเภทใดก็ตาม
(ยอดเงินจริงจะถูกฝากเข้าในขั้นตอนถัดไปหลังจากเปิดบัญชีสำเร็จแล้ว ซึ่งเป็นความรับผิดชอบของโปรแกรมอื่น
ไม่ใช่ของ Factory) การตั้งค่านี้ไว้**นอก** `EVALUATE` (ก่อนเข้าเงื่อนไขแยกตามประเภท) แทนที่จะซ้ำ
คำสั่ง `MOVE 0 TO LK-BALANCE` ไว้ในทุก `WHEN` สะท้อนหลักการ **"กฎที่ใช้ร่วมกันทุกกรณีควรเขียนไว้
ที่เดียว ไม่ซ้ำในแต่ละสาขาของเงื่อนไข"** ซึ่งเป็นการประยุกต์ใช้แนวคิดลดความซ้ำซ้อนของโค้ด (Part 081
ขั้นตอนที่ 807) เข้ากับการออกแบบ Factory — ทำให้ถ้าในอนาคตมีกฎเปลี่ยนแปลง (เช่น "บัญชี VIP ควรเปิด
ด้วยโบนัสเริ่มต้น 100 บาท") จะรู้ทันทีว่าต้องเพิ่มเงื่อนไขพิเศษที่ไหนโดยไม่กระทบค่าเริ่มต้นร่วมของ
ประเภทอื่น

---

## ขั้นตอนที่ 815: Singleton-like Pattern ผ่านสถานะที่คงอยู่ใน Subprogram

### ปัญหาที่ Singleton Pattern แก้

บางครั้งระบบต้องการให้มี **"ตัวนับ" หรือ "สถานะควบคุม" เพียงชุดเดียวที่ใช้ร่วมกันทั้งระบบ** เช่น
เลขที่ใบเสร็จที่ต้องไม่ซ้ำกันไม่ว่าจะถูกขอจากส่วนไหนของโปรแกรมก็ตาม **Singleton Pattern** รับประกัน
ว่ามี "instance" เดียวของสถานะนี้เท่านั้นในระบบทั้งหมด

### กลไกของ COBOL ที่ทำให้ Singleton เกิดขึ้นได้ตามธรรมชาติ

ทบทวนจาก Part 031 ขั้นตอนที่ 309: **WORKING-STORAGE ของ subprogram คงค่าอยู่ข้ามการเรียกหลายครั้ง**
(จนกว่าจะถูก `CANCEL`) — คุณสมบัตินี้เองที่ทำให้ subprogram หนึ่งตัวสามารถทำหน้าที่เป็น "เจ้าของ
สถานะเดียวที่ใช้ร่วมกัน" ได้ตามธรรมชาติ โดยไม่ต้องมีกลไกพิเศษอื่นใดเพิ่มเติมเลย

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. BATCHCOUNTER.
       AUTHOR. COBOL-COURSE.

      *> SINGLETON-LIKE PATTERN: this subprogram owns ONE piece of
      *> shared state (WS-COUNTER) in its own WORKING-STORAGE. Because
      *> GnuCOBOL keeps a called subprogram's WORKING-STORAGE alive
      *> between CALLs (until CANCELed), every caller that CALLs
      *> BATCHCOUNTER shares the SAME single counter instance - no
      *> caller can create a second, independent counter.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNTER               PIC 9(5) VALUE 0.

       LINKAGE SECTION.
       01  LK-NEXT-VALUE            PIC 9(5).

       PROCEDURE DIVISION USING LK-NEXT-VALUE.
       MAIN-PARA.
           ADD 1 TO WS-COUNTER.
           MOVE WS-COUNTER TO LK-NEXT-VALUE.
           GOBACK.
```

### โปรแกรมหลักที่เรียกใช้จากหลายจุด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP815MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TICKET-NO             PIC 9(5).

       PROCEDURE DIVISION.
       MAIN-PARA.
           CALL "BATCHCOUNTER" USING WS-TICKET-NO.
           DISPLAY "Module A got ticket " WS-TICKET-NO.

           CALL "BATCHCOUNTER" USING WS-TICKET-NO.
           DISPLAY "Module B got ticket " WS-TICKET-NO.

           CALL "BATCHCOUNTER" USING WS-TICKET-NO.
           DISPLAY "Module C got ticket " WS-TICKET-NO.

           DISPLAY "-- CANCEL BATCHCOUNTER: resets its state --".
           CANCEL "BATCHCOUNTER".

           CALL "BATCHCOUNTER" USING WS-TICKET-NO.
           DISPLAY "Module D got ticket " WS-TICKET-NO
               " (back to 1 after CANCEL)".
           STOP RUN.
```

**คอมไพล์และรันจริง:**

```bash
cobc -x -o step815main step815main.cob batchcounter.cob
./step815main
```

**ผลลัพธ์จริง (ยืนยันด้วยการรัน):**

```
Module A got ticket 00001
Module B got ticket 00002
Module C got ticket 00003
-- CANCEL BATCHCOUNTER: resets its state --
Module D got ticket 00001 (back to 1 after CANCEL)
```

### อธิบายจุดสำคัญ

- แม้ `MAIN-PARA` จะเรียก `CALL "BATCHCOUNTER"` ถึง 3 ครั้งราวกับเป็น "Module A", "B", "C" คนละ
  ส่วนของระบบ แต่**ตัวนับที่ได้กลับมาเพิ่มขึ้นต่อเนื่องกันเสมอ (1, 2, 3)** — พิสูจน์ว่า
  `WS-COUNTER` ภายใน `BATCHCOUNTER` เป็น**สถานะเดียวที่ใช้ร่วมกัน**อย่างแท้จริง ไม่ได้ถูกสร้างขึ้น
  ใหม่ทุกครั้งที่ `CALL`
- `CANCEL "BATCHCOUNTER"` (ทบทวนจาก Part 031 ขั้นตอนที่ 309) **รีเซ็ต WORKING-STORAGE ของ
  subprogram กลับสู่ค่าเริ่มต้น** ทำให้การเรียกครั้งถัดไปหลัง `CANCEL` เริ่มนับใหม่จาก 1 — นี่คือ
  "จุดอ่อน" ที่สำคัญของ Singleton แบบ COBOL: **สถานะ "เดี่ยว" นี้ไม่ได้คงอยู่ตลอดไปเหมือน Singleton
  ในภาษา OOP ที่มักคงอยู่ตลอดอายุของโปรแกรม** — มันคงอยู่แค่ตราบใดที่ยังไม่มีใคร `CANCEL` มันเท่านั้น
- เทียบกับภาษา OOP: `WS-COUNTER` เทียบเท่ากับ **private static field** ของ Singleton class และการ
  ไม่มี constructor ที่เรียกได้จากภายนอก (COBOL ไม่มีแนวคิด "สร้าง instance ใหม่" ของ subprogram
  เองอยู่แล้ว มีแต่ `CALL` เรียกใช้) ทำให้พฤติกรรม "มี instance เดียว" เกิดขึ้นเองตามธรรมชาติของ
  ภาษาโดยไม่ต้องเขียนกลไกป้องกันการสร้างซ้ำเพิ่มเติมเหมือนที่ Java ต้องทำ (เช่น private constructor
  + static getInstance() method)

### ข้อควรระวัง

- **Singleton-like Pattern แบบนี้ใช้ได้เฉพาะภายในโปรแกรม executable เดียวกัน** (โปรแกรมทั้งหมดที่
  ถูก `cobc` link รวมกันเป็นไฟล์เดียว) — ถ้าเป็นระบบที่มีหลาย process แยกกันทำงาน (เช่น หลาย batch
  job รันพร้อมกันคนละ process) แต่ละ process จะมี `WS-COUNTER` ของตัวเองแยกกันโดยสิ้นเชิง **ไม่ใช่
  ตัวนับเดียวกันข้าม process** ถ้าต้องการตัวนับที่ใช้ร่วมกันข้าม process จริง ๆ (เช่น เลขที่ใบเสร็จ
  ที่ต้องไม่ซ้ำกันทั้งองค์กร) ต้องใช้กลไกภายนอก เช่น ไฟล์ควบคุมที่ทุก process อ่าน/เขียนร่วมกัน
  (มีการล็อกป้องกัน concurrent access) หรือฐานข้อมูลกลาง (ทบทวนแนวคิด DB2 จาก Part 057)
- การพึ่งพา WORKING-STORAGE ที่คงค่าข้ามการเรียกแบบนี้ต้อง**ระวังเรื่อง test isolation**อย่างมาก
  (ทบทวนปัญหานี้จาก Part 079 ขั้นตอนที่ 781) — ถ้าเขียน unit test ที่เรียก `BATCHCOUNTER` หลาย
  test case โดยไม่ `CANCEL` ระหว่างแต่ละ test case ผลลัพธ์ของแต่ละ test จะขึ้นกับลำดับการรันและ
  ค่าที่ค้างจาก test ก่อนหน้า ทำให้ test ไม่เป็นอิสระจากกันซึ่งขัดกับหลักการพื้นฐานของ unit testing
  ที่ดี (ควร `CANCEL "BATCHCOUNTER".` ก่อนเริ่ม test แต่ละกรณีเสมอถ้าต้องการผลลัพธ์ที่คาดเดาได้)

### แบบฝึกหัดที่ 815.1

**โจทย์**: จงอธิบายว่าทำไมผลลัพธ์ของ "Module D" ถึงกลับไปเริ่มที่ `00001` อีกครั้งหลัง `CANCEL
"BATCHCOUNTER"` ทั้งที่ก่อนหน้านั้นตัวนับเดินไปถึง `00003` แล้ว

**เฉลย**: `CANCEL` เป็นคำสั่งที่บอก COBOL runtime ให้**ทำลายสถานะ (state) ทั้งหมดของ subprogram
ที่ระบุ** รวมถึงค่าทั้งหมดใน WORKING-STORAGE ของมัน แล้วเตรียมพร้อมให้การ `CALL` ครั้งถัดไปเริ่มต้น
subprogram นั้นใหม่ตั้งแต่ต้น**ราวกับว่าไม่เคยถูกเรียกมาก่อนเลย** เมื่อ `BATCHCOUNTER` ถูกเรียกใหม่
หลัง `CANCEL`, `WS-COUNTER` จะถูกกำหนดค่าเริ่มต้นใหม่ตาม clause `VALUE 0` ที่ประกาศไว้ ทำให้
`ADD 1 TO WS-COUNTER` ในรอบนี้ได้ผลลัพธ์เป็น `1` เหมือนกับการเรียกครั้งแรกสุดของโปรแกรมทั้งหมด
พฤติกรรมนี้แสดงให้เห็นชัดเจนว่า "ความเป็น Singleton" ของ subprogram ใน COBOL **ไม่ได้ผูกกับอายุของ
โปรแกรมทั้งหมดโดยอัตโนมัติ** แต่ผูกกับ**ช่วงเวลาที่ยังไม่มีใคร `CANCEL` มันเท่านั้น** ซึ่งเป็นความ
แตกต่างที่สำคัญจาก Singleton pattern ในภาษา OOP ที่ instance เดียวมักคงอยู่ตลอดอายุการทำงานของ
โปรแกรมโดยไม่มีกลไกใดมา "รีเซ็ต" มันได้ง่ายขนาดนี้

---

## ขั้นตอนที่ 816: Template Method Pattern ผ่าน Driver ที่เรียก Hook Paragraph

### ปัญหาที่ Template Method Pattern แก้

สมมติระบบต้องพิมพ์รายงานหลายแบบ (สรุปย่อ, รายละเอียดเต็ม) ที่มี**โครงร่างเหมือนกันทุกประการ**
(หัวรายงาน → เนื้อหา → ท้ายรายงาน) แต่**ส่วนเนื้อหาตรงกลางต่างกันไปตามชนิดของรายงาน**
**Template Method Pattern** แยก "โครงร่างที่ตายตัว" (skeleton) ออกจาก "ส่วนที่เปลี่ยนแปลงได้"
(hook) อย่างชัดเจน — โครงร่างถูกกำหนดไว้ที่เดียวและไม่เปลี่ยนแปลง ส่วน hook สามารถสลับเปลี่ยนได้
อิสระ

### สร้าง Hook แต่ละแบบเป็น Subprogram แยก

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SUMMARYBODY.
       AUTHOR. COBOL-COURSE.

      *> One "hook" implementation for the varying step of the
      *> template - prints a short summary body.
       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "  [body] Total sales this month: 458,200.00".
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DETAILBODY.
       AUTHOR. COBOL-COURSE.

      *> Another "hook" implementation for the same varying step -
      *> prints a detailed, itemized body instead.
       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "  [body] Item 1: Widget A ..... 1,200.00".
           DISPLAY "  [body] Item 2: Widget B ..... 3,400.50".
           DISPLAY "  [body] Item 3: Widget C ..... 2,150.75".
           GOBACK.
```

### โปรแกรมหลัก: Template (โครงร่างตายตัว) ที่เรียก Hook

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP816MAIN.
       AUTHOR. COBOL-COURSE.

      *> TEMPLATE METHOD PATTERN: the overall algorithm skeleton
      *> (header, then body, then footer) is FIXED and lives in one
      *> place (PRINT-REPORT). The "body" step is the customizable
      *> HOOK - which subprogram gets CALLed for it varies, but the
      *> surrounding skeleton never changes no matter which hook runs.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-BODY-PROGRAM          PIC X(12).

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "=== Report variant: SUMMARY ===".
           MOVE "SUMMARYBODY" TO WS-BODY-PROGRAM.
           PERFORM PRINT-REPORT.

           DISPLAY "=== Report variant: DETAIL ===".
           MOVE "DETAILBODY" TO WS-BODY-PROGRAM.
           PERFORM PRINT-REPORT.
           STOP RUN.

      *> This is the fixed "template" - always header, then the
      *> hook, then footer, in this exact order, no matter which
      *> body program is plugged in via WS-BODY-PROGRAM.
       PRINT-REPORT.
           PERFORM PRINT-HEADER.
           CALL WS-BODY-PROGRAM.
           PERFORM PRINT-FOOTER.

       PRINT-HEADER.
           DISPLAY "  [header] Monthly Sales Report".
           DISPLAY "  [header] ----------------------".

       PRINT-FOOTER.
           DISPLAY "  [footer] -- End of report --".
```

**คอมไพล์และรันจริง:**

```bash
cobc -x -o step816main step816main.cob summarybody.cob detailbody.cob
./step816main
```

**ผลลัพธ์จริง (ยืนยันด้วยการรัน):**

```
=== Report variant: SUMMARY ===
  [header] Monthly Sales Report
  [header] ----------------------
  [body] Total sales this month: 458,200.00
  [footer] -- End of report --
=== Report variant: DETAIL ===
  [header] Monthly Sales Report
  [header] ----------------------
  [body] Item 1: Widget A ..... 1,200.00
  [body] Item 2: Widget B ..... 3,400.50
  [body] Item 3: Widget C ..... 2,150.75
  [footer] -- End of report --
```

### อธิบายจุดสำคัญ

- `PRINT-REPORT` คือ **"Template"** — ลำดับ `PERFORM PRINT-HEADER` → `CALL WS-BODY-PROGRAM` →
  `PERFORM PRINT-FOOTER` **ไม่เปลี่ยนแปลงเลย** ไม่ว่าจะพิมพ์รายงานแบบใด นี่คือส่วน "โครงร่างตายตัว"
  ของ pattern
- `WS-BODY-PROGRAM` คือ **"Hook"** ที่เปลี่ยนแปลงได้ — การกำหนดค่า `WS-BODY-PROGRAM` ก่อนเรียก
  `PERFORM PRINT-REPORT` แต่ละครั้งคือจุดที่ "เลือก variant" โดยที่โครงสร้างของ `PRINT-REPORT` เอง
  ไม่ต้องรู้เลยว่ากำลังเรียก `SUMMARYBODY` หรือ `DETAILBODY` อยู่
- สังเกตว่าทั้ง `SUMMARYBODY` และ `DETAILBODY` เป็น subprogram ที่**ไม่รับพารามิเตอร์เลย**
  (`PROCEDURE DIVISION.` เปล่า ๆ ไม่มี `USING`) เพราะในตัวอย่างนี้ hook แค่พิมพ์ข้อมูลของตัวเองออก
  มาตรง ๆ — ในระบบจริงที่ซับซ้อนกว่านี้ hook อาจต้องรับพารามิเตอร์ร่วมกัน (เช่น ช่วงวันที่ของ
  รายงาน) ซึ่งจะต้องออกแบบ "สัญญา" ของพารามิเตอร์ให้ hook ทุกตัวรับเหมือนกัน (คล้ายหลักการ Strategy
  Pattern จากขั้นตอนที่ 812)

### ข้อควรระวัง

- Template Method Pattern แบบนี้เหมาะกับกรณีที่ **โครงร่างของกระบวนการชัดเจนและตายตัว** — ถ้าลำดับ
  ขั้นตอน (ไม่ใช่แค่เนื้อหาระหว่างขั้นตอน) เปลี่ยนแปลงไปตาม variant ด้วย (เช่น รายงานบางแบบไม่มี
  ส่วนหัวเลย) pattern นี้จะไม่เหมาะสมอีกต่อไป เพราะ template ควรมีแค่**โครงร่างเดียวที่ใช้ร่วมกัน
  ได้กับทุก variant**
- ต้องระวังเรื่อง**สัญญาของ hook ที่ไม่ตรงกัน** เช่นเดียวกับ Strategy Pattern — ถ้า hook ตัวใดตัว
  หนึ่งถูกออกแบบให้ต้องการพารามิเตอร์ที่ hook ตัวอื่นไม่ต้องการ `PRINT-REPORT` (Template) จะต้อง
  รู้เรื่องพารามิเตอร์ที่ต่างกันนี้ ซึ่งขัดกับหลักการที่ Template ควรเรียก hook แบบเดียวกันเสมอ
  ไม่ว่าจะเป็น hook ตัวไหน

### แบบฝึกหัดที่ 816.1

**โจทย์**: จงออกแบบ hook ใหม่ชื่อ `EMPTYBODY` ที่แสดงข้อความ "No sales data available." แทนเนื้อหา
ปกติ แล้วอธิบายว่าทำไม `PRINT-REPORT` (Template) ไม่จำเป็นต้องแก้ไขแม้แต่บรรทัดเดียวเพื่อรองรับ
hook ใหม่นี้

**เฉลย**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EMPTYBODY.
       AUTHOR. COBOL-COURSE.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "  [body] No sales data available.".
           GOBACK.
```

เพียงเพิ่มการตั้งค่า `MOVE "EMPTYBODY" TO WS-BODY-PROGRAM.` ก่อน `PERFORM PRINT-REPORT` ใน
`MAIN-PARA` ก็สามารถใช้ hook ใหม่นี้ได้ทันที เหตุผลที่ `PRINT-REPORT` ไม่ต้องแก้ไขเลยคือ
`PRINT-REPORT` ไม่เคยรู้จักชื่อของ hook ที่เจาะจง (`SUMMARYBODY`, `DETAILBODY`, หรือ `EMPTYBODY`)
โดยตรงเลยแม้แต่ครั้งเดียว — มันแค่ `CALL WS-BODY-PROGRAM` ซึ่งเป็น**ชื่อในตัวแปร**ที่ผู้เรียก
(`MAIN-PARA`) เป็นผู้กำหนดค่าให้ก่อนเสมอ (ทบทวนเทคนิค `CALL` ด้วยชื่อในตัวแปรจาก Part 031 ขั้นตอน
ที่ 305) นี่คือจุดแข็งสำคัญของการออกแบบ Template Method แบบ COBOL: **Template ผูกกับ "แนวคิดของ
hook" ไม่ใช่ "ชื่อของ hook ตัวใดตัวหนึ่งโดยเฉพาะ"** ทำให้เพิ่ม hook ใหม่ได้เรื่อย ๆ โดยไม่กระทบ
โครงร่างเดิมเลย ตราบใดที่ hook ใหม่ยังคงมี "สัญญา" (ในที่นี้คือไม่รับพารามิเตอร์ใด ๆ) ตรงกับที่
Template คาดหวังไว้

---

## ขั้นตอนที่ 817: เปรียบเทียบ Design Patterns ใน COBOL กับภาษา OOP — ข้อจำกัดที่ต้องยอมรับ

### ตารางเปรียบเทียบโดยตรง

| แง่มุม | ภาษา OOP (เช่น Java) | COBOL |
|---|---|---|
| การบังคับ "สัญญา" (interface) | Compiler ตรวจสอบ method signature ให้อัตโนมัติ | ไม่มีการตรวจสอบเลย ต้องอาศัยวินัยของทีม (Part 031 ขั้นตอนที่ 304) |
| Polymorphism | มีในตัวภาษาโดยตรง (virtual method dispatch) | จำลองด้วย `CALL` ชื่อในตัวแปร (Part 031 ขั้นตอนที่ 305) |
| การป้องกัน Singleton ถูกสร้างซ้ำ | private constructor + static instance | เกิดขึ้นเองตามธรรมชาติจาก WORKING-STORAGE ที่คงค่า แต่เสี่ยงต่อ `CANCEL` |
| Encapsulation (ซ่อนรายละเอียดภายใน) | private field/method ระดับภาษา | ไม่มีกลไกบังคับ — ผู้เรียกยังคงเข้าถึง WORKING-STORAGE ผ่านพารามิเตอร์ที่ส่งเข้ามาได้เสมอถ้าออกแบบไม่รัดกุม |
| Inheritance (การสืบทอด) | มีในตัวภาษาโดยตรง | ไม่มีกลไกเทียบเท่าที่ตรงไปตรงมา ต้องจำลองด้วยการ COPY โครงสร้างร่วม (Part 033) |

### ทำไมความแตกต่างเหล่านี้ถึงสำคัญต่อการตัดสินใจ

ข้อจำกัดที่สำคัญที่สุดคือ **การขาดการตรวจสอบ "สัญญา" โดย compiler** — ในภาษาที่มี interface บังคับ
ถ้า class ใดไม่ implement method ที่ interface กำหนดไว้ครบ จะคอมไพล์ไม่ผ่านทันที แต่ใน COBOL ถ้า
subprogram ตัวหนึ่งใน Strategy Pattern (ขั้นตอนที่ 812) ลืมรับพารามิเตอร์ตัวหนึ่งไป **จะไม่มีทาง
รู้เลยจนกว่าจะรันจริงแล้วเจอ Segmentation Fault** (ปัญหาคลาสสิกจาก Part 031 ขั้นตอนที่ 304)

### แนวทางลดความเสี่ยงจากข้อจำกัดนี้

| แนวทาง | รายละเอียด |
|---|---|
| เขียนเอกสาร "สัญญา" ของแต่ละ pattern อย่างชัดเจน | ระบุพารามิเตอร์ที่ทุก strategy/hook ต้องมีเหมือนกัน |
| ใช้ Copybook ร่วมกันสำหรับ record ที่ซับซ้อน | ลดความเสี่ยงโครงสร้างไม่ตรงกัน (ทบทวน Part 033, Part 080 ขั้นตอนที่ 796) |
| เขียน Unit Test ให้ครบทุก strategy/hook | ทบทวนจาก Part 079 — ถ้าพารามิเตอร์ไม่ตรง unit test จะจับได้ก่อนถึง production |
| Code Review ที่ตรวจสอบความสอดคล้องของ interface โดยเฉพาะ | อาศัยมนุษย์ทดแทนสิ่งที่ compiler ทำให้ไม่ได้ |

### ข้อควรระวัง

- อย่าประเมินความเสี่ยงของการขาดการตรวจสอบสัญญาต่ำเกินไป — ยิ่งระบบมี pattern ที่ซับซ้อนและมี
  จำนวน subprogram ที่ต้อง "implement interface เดียวกัน" มากเท่าไหร่ ความเสี่ยงจากความไม่สอดคล้อง
  กันก็ยิ่งสูงขึ้นตามไปด้วย
- การไม่มี Inheritance ที่แท้จริงใน COBOL หมายความว่าเทคนิคบางอย่างในหนังสือ Design Pattern ดั้งเดิม
  (ที่พึ่งพา class hierarchy อย่างมาก เช่น Decorator Pattern แบบคลาสสิก) จะปรับใช้กับ COBOL ได้ยาก
  กว่า pattern ที่พึ่งพาแค่ "การเรียกผ่าน interface" อย่าง Strategy หรือ Template Method

### แบบฝึกหัดที่ 817.1

**โจทย์**: จงอธิบายว่าทำไมการเขียน Unit Test ให้ครบทุก strategy (ตามแนวทางในตาราง) ถึงเป็นวิธีที่
มีประสิทธิภาพเป็นพิเศษในการชดเชยข้อจำกัดเรื่อง "compiler ไม่ตรวจสอบสัญญา" ของ COBOL

**เฉลย**: เพราะ Unit Test ที่ดี (ตามที่ Part 079 สอน) จะ `CALL` แต่ละ strategy subprogram ด้วยค่า
input จริงและตรวจสอบผลลัพธ์ที่ได้ ถ้า strategy ตัวใดตัวหนึ่งมีจำนวนหรือชนิดพารามิเตอร์ไม่ตรงกับที่
Test Driver คาดหวัง (ซึ่งจำลองการเรียกจากระบบ dispatch จริง) การรัน test จะทำให้เกิด runtime error
(เช่น Segmentation Fault ตามปัญหาคลาสสิกจาก Part 031 ขั้นตอนที่ 304) **ทันทีตอนรัน test** แทนที่จะ
รอไปเจอตอนระบบทำงานจริงบน production ซึ่งอาจสร้างความเสียหายร้ายแรงกว่ามาก การมี Unit Test ที่
ครอบคลุมทุก strategy/hook จึงทำหน้าที่เป็น**เกราะป้องกันทดแทนสิ่งที่ compiler ของภาษาอื่นทำให้ได้
ฟรี ๆ** และเมื่อรวมกับ CI Pipeline (Part 078) ที่รันทดสอบอัตโนมัติทุกครั้งที่มีการแก้ไขโค้ด ก็จะ
กลายเป็นระบบป้องกันที่แข็งแกร่งเทียบเท่าหรือดีกว่าการพึ่งพา compiler เพียงอย่างเดียวในบางแง่มุม
(เพราะ Unit Test ตรวจสอบพฤติกรรมจริง ไม่ใช่แค่ไวยากรณ์)

---

## ขั้นตอนที่ 818: ตัวอย่างรวม — Factory และ Strategy ทำงานร่วมกัน

### สถานการณ์: ระบบคำนวณค่าธรรมเนียมการต่ออายุสมาชิก

ระบบนี้ต้อง (1) **สร้าง record สมาชิกใหม่** พร้อมค่าเริ่มต้นตามประเภท (งาน Factory) แล้ว (2)
**คำนวณค่าธรรมเนียมต่ออายุ**ตามกฎที่ต่างกันไปในแต่ละประเภทสมาชิก (งาน Strategy) — สอง pattern
ทำงานร่วมกันในกระบวนการเดียว

### Factory: สร้าง Record สมาชิก

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MEMBERFACTORY.
       AUTHOR. COBOL-COURSE.

      *> FACTORY step of the combined example: build a membership
      *> record with type-appropriate defaults.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-MEMBER-TYPE           PIC X(1).
       01  LK-YEARS-ACTIVE          PIC 9(2).
       01  LK-MEMBER-RECORD.
           05  LK-MEM-TYPE-OUT      PIC X(1).
           05  LK-BASE-FEE          PIC 9(5)V99.
           05  LK-YEARS-OUT         PIC 9(2).

       PROCEDURE DIVISION USING LK-MEMBER-TYPE LK-YEARS-ACTIVE
               LK-MEMBER-RECORD.
       MAIN-PARA.
           MOVE LK-MEMBER-TYPE TO LK-MEM-TYPE-OUT.
           MOVE LK-YEARS-ACTIVE TO LK-YEARS-OUT.
           EVALUATE LK-MEMBER-TYPE
               WHEN "B"
                   MOVE 500.00 TO LK-BASE-FEE
               WHEN "P"
                   MOVE 2000.00 TO LK-BASE-FEE
               WHEN OTHER
                   MOVE 0.00 TO LK-BASE-FEE
           END-EVALUATE.
           GOBACK.
```

### Strategy: กฎการคำนวณค่าต่ออายุแยกตามประเภท

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RENEWBASIC.
       AUTHOR. COBOL-COURSE.

      *> STRATEGY: basic members always pay the flat base fee.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-BASE-FEE              PIC 9(5)V99.
       01  LK-YEARS-ACTIVE          PIC 9(2).
       01  LK-RENEWAL-FEE           PIC 9(5)V99.

       PROCEDURE DIVISION USING LK-BASE-FEE LK-YEARS-ACTIVE
               LK-RENEWAL-FEE.
       MAIN-PARA.
           MOVE LK-BASE-FEE TO LK-RENEWAL-FEE.
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RENEWPREMIUM.
       AUTHOR. COBOL-COURSE.

      *> STRATEGY: premium members get a loyalty discount of 2% of
      *> the base fee per year active, capped at 20%.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DISCOUNT-PCT          PIC 9V99.
       01  WS-DISCOUNT-AMT          PIC 9(5)V99.

       LINKAGE SECTION.
       01  LK-BASE-FEE              PIC 9(5)V99.
       01  LK-YEARS-ACTIVE          PIC 9(2).
       01  LK-RENEWAL-FEE           PIC 9(5)V99.

       PROCEDURE DIVISION USING LK-BASE-FEE LK-YEARS-ACTIVE
               LK-RENEWAL-FEE.
       MAIN-PARA.
           COMPUTE WS-DISCOUNT-PCT = LK-YEARS-ACTIVE * 0.02.
           IF WS-DISCOUNT-PCT > 0.20
               MOVE 0.20 TO WS-DISCOUNT-PCT
           END-IF.
           COMPUTE WS-DISCOUNT-AMT ROUNDED =
               LK-BASE-FEE * WS-DISCOUNT-PCT.
           COMPUTE LK-RENEWAL-FEE = LK-BASE-FEE - WS-DISCOUNT-AMT.
           GOBACK.
```

### โปรแกรมหลัก: ประกอบ Factory + Strategy เข้าด้วยกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP818MAIN.
       AUTHOR. COBOL-COURSE.

      *> COMBINED EXAMPLE: FACTORY builds the member record with the
      *> right defaults for its type, then STRATEGY dispatches to the
      *> renewal-fee rule that matches that same type. Two patterns,
      *> each responsible for a different concern, working together.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MEMBER-TYPE           PIC X(1).
       01  WS-YEARS-ACTIVE          PIC 9(2).
       01  WS-MEMBER-RECORD.
           05  WS-MEM-TYPE          PIC X(1).
           05  WS-BASE-FEE          PIC 9(5)V99.
           05  WS-YEARS             PIC 9(2).
       01  WS-RENEWAL-FEE           PIC 9(5)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "B" TO WS-MEMBER-TYPE.
           MOVE 3 TO WS-YEARS-ACTIVE.
           PERFORM PROCESS-RENEWAL.

           MOVE "P" TO WS-MEMBER-TYPE.
           MOVE 3 TO WS-YEARS-ACTIVE.
           PERFORM PROCESS-RENEWAL.

           MOVE "P" TO WS-MEMBER-TYPE.
           MOVE 15 TO WS-YEARS-ACTIVE.
           PERFORM PROCESS-RENEWAL.
           STOP RUN.

       PROCESS-RENEWAL.
      *> Step 1: FACTORY creates the record with type-based defaults.
           CALL "MEMBERFACTORY" USING WS-MEMBER-TYPE WS-YEARS-ACTIVE
               WS-MEMBER-RECORD.
      *> Step 2: STRATEGY picks the renewal rule matching that type.
           EVALUATE WS-MEM-TYPE
               WHEN "B"
                   CALL "RENEWBASIC" USING WS-BASE-FEE WS-YEARS
                       WS-RENEWAL-FEE
               WHEN "P"
                   CALL "RENEWPREMIUM" USING WS-BASE-FEE WS-YEARS
                       WS-RENEWAL-FEE
           END-EVALUATE.
           DISPLAY "type=" WS-MEM-TYPE " years=" WS-YEARS
               " base=" WS-BASE-FEE " renewal=" WS-RENEWAL-FEE.
```

**คอมไพล์และรันจริง:**

```bash
cobc -x -o step818main step818main.cob memberfactory.cob renewbasic.cob renewpremium.cob
./step818main
```

**ผลลัพธ์จริง (ยืนยันด้วยการรัน):**

```
type=B years=03 base=00500.00 renewal=00500.00
type=P years=03 base=02000.00 renewal=01880.00
type=P years=15 base=02000.00 renewal=01600.00
```

ตรวจสอบด้วยมือ: สมาชิก Premium 3 ปี → ส่วนลด 3×2% = 6% ของ 2000 = 120.00 → ค่าต่ออายุ 1880.00 ✔
สมาชิก Premium 15 ปี → ส่วนลด 15×2% = 30% แต่ถูกจำกัดเพดานไว้ที่ 20% ของ 2000 = 400.00 → ค่าต่ออายุ
1600.00 ✔

### อธิบายจุดสำคัญ

- `PROCESS-RENEWAL` คือ paragraph ที่**ประสานงานระหว่างสอง pattern**: เรียก Factory ก่อนเพื่อได้
  record ที่สมบูรณ์ แล้วใช้ `WS-MEM-TYPE` จาก record นั้น**เป็นตัวตัดสินใจ**ว่าจะ dispatch ไปยัง
  Strategy ตัวไหนต่อ — นี่คือรูปแบบทั่วไปที่พบในระบบจริง: pattern ต่าง ๆ มักไม่ได้ทำงานโดดเดี่ยว
  แต่**ประกอบกันเป็นกระบวนการทางธุรกิจที่สมบูรณ์**
- สังเกตว่า `RENEWBASIC` และ `RENEWPREMIUM` มี**สัญญาเดียวกันทุกประการ** (`LK-BASE-FEE`,
  `LK-YEARS-ACTIVE` เข้า, `LK-RENEWAL-FEE` ออก) แม้สูตรคำนวณภายในจะต่างกันโดยสิ้นเชิง (คงที่เทียบกับ
  มีส่วนลดแบบขั้นบันได) — ตรงตามหลักการ Strategy Pattern จากขั้นตอนที่ 812 ทุกประการ
- การแยกความรับผิดชอบชัดเจนแบบนี้ (Factory รับผิดชอบแค่ "สร้าง", Strategy รับผิดชอบแค่ "คำนวณค่า
  ต่ออายุ") สอดคล้องกับหลักการ Single Responsibility ที่ Part 031 ขั้นตอนที่ 310 เคยแนะนำไว้ในบริบท
  ของการออกแบบ subprogram ทั่วไป

### ข้อควรระวัง

- เมื่อ pattern หลายตัวทำงานร่วมกัน ความเสี่ยงเรื่อง "สัญญาไม่ตรงกัน" (ตามที่กล่าวในขั้นตอนที่ 817)
  จะทวีคูณขึ้น เพราะตอนนี้มีทั้ง "สัญญาของ Factory" และ "สัญญาของ Strategy" ที่ต้องดูแลพร้อมกัน —
  ยิ่งจำเป็นต้องมี Unit Test ที่ครอบคลุมทั้งสองระดับ (ทดสอบ Factory แยก, ทดสอบแต่ละ Strategy แยก,
  และทดสอบการทำงานร่วมกันทั้งกระบวนการ) ตามแนวทาง Part 079
- อย่าลืมคอมไพล์รวมทุกไฟล์ที่เกี่ยวข้อง (`step818main.cob memberfactory.cob renewbasic.cob
  renewpremium.cob`) — ยิ่งจำนวนไฟล์ที่ประกอบกันมากขึ้น ยิ่งควรพิจารณาใช้ Makefile (ทบทวนจาก Part
  078 ขั้นตอนที่ 776) แทนการพิมพ์รายชื่อไฟล์ทั้งหมดด้วยมือทุกครั้ง

### แบบฝึกหัดที่ 818.1

**โจทย์**: จงอธิบายว่าทำไมการทดสอบยอดเงิน 3 ปี และ 15 ปี ของสมาชิก Premium (ที่ให้ผลลัพธ์ 1880.00
และ 1600.00 ตามลำดับ) ถึงเป็น test case ที่ดีสำหรับพิสูจน์ตรรกะ "เพดานส่วนลด 20%" ใน `RENEWPREMIUM`

**เฉลย**: ปี 3 ให้ส่วนลด 6% ซึ่ง**ยังไม่ถึงเพดาน** 20% เลย (`WS-DISCOUNT-PCT > 0.20` เป็นเท็จ) ทำให้
พิสูจน์ได้ว่าสูตรคำนวณส่วนลดพื้นฐาน (`years × 2%`) ทำงานถูกต้องในกรณีปกติ ในขณะที่ปี 15 ให้ส่วนลด
ตามสูตรพื้นฐานถึง 30% ซึ่ง**เกินเพดาน** 20% ทำให้เงื่อนไข `IF WS-DISCOUNT-PCT > 0.20` เป็นจริงและ
ถูกจำกัดค่าลงเหลือ 20% พอดี การมีทั้งสอง test case นี้คู่กัน**ครอบคลุมทั้งสอง branch ของเงื่อนไข
`IF`** ภายใน `RENEWPREMIUM` (ทั้งกรณีที่เงื่อนไขเป็นจริงและเป็นเท็จ) ซึ่งเป็นหลักการพื้นฐานที่สำคัญ
มากในการออกแบบชุดทดสอบ (เรียกว่า **Branch Coverage** ในศัพท์ทางวิศวกรรมซอฟต์แวร์) — ถ้าทดสอบแค่
กรณีเดียว (เช่น แค่ 3 ปี) จะไม่มีทางรู้เลยว่าตรรกะเรื่องเพดาน 20% ทำงานถูกต้องจริงหรือไม่ เพราะ
เงื่อนไขนั้นไม่เคยถูกกระตุ้นให้ทำงานเลยระหว่างการทดสอบ

---

## ขั้นตอนที่ 819: เมื่อไหร่ไม่ควรใช้ Design Pattern — หลีกเลี่ยง Over-Engineering

### สัญญาณเตือนว่ากำลัง Over-Engineer

| สัญญาณ | ตัวอย่าง | สิ่งที่ควรทำแทน |
|---|---|---|
| มี strategy แค่ 1–2 แบบและไม่มีแนวโน้มจะเพิ่ม | สร้าง Strategy Pattern เต็มรูปแบบสำหรับ `IF` ง่าย ๆ 2 เงื่อนไข | ใช้ `IF`/`EVALUATE` ตรง ๆ ตามที่ Part 081 สอน |
| ใช้ Factory สำหรับ record ที่ไม่มีกฎการสร้างที่ซับซ้อนเลย | สร้าง subprogram Factory สำหรับ record ที่แค่ต้องการ `MOVE SPACES` เริ่มต้น | `MOVE`/`INITIALIZE` ตรง ๆ ในโปรแกรมที่ใช้งาน |
| ใช้ Singleton สำหรับค่าที่ไม่จำเป็นต้องใช้ร่วมกันจริง | เก็บตัวแปรชั่วคราวที่ใช้แค่ภายในโปรแกรมเดียวไว้ใน subprogram แยก | ประกาศเป็น `WORKING-STORAGE` ธรรมดาในโปรแกรมนั้นเอง |
| ใช้ Template Method สำหรับกระบวนการที่มี variant เดียว | สร้างโครงสร้าง hook ทั้งระบบสำหรับรายงานที่มีแบบเดียวตลอดไป | เขียนโปรแกรมตรง ๆ ตามรูปแบบ Driver Paragraph (Part 081 ขั้นตอนที่ 808) |

### ต้นทุนที่แท้จริงของการใช้ Pattern โดยไม่จำเป็น

1. **เพิ่มจำนวนไฟล์ที่ต้องคอมไพล์และดูแล** — ทุก subprogram ที่แยกออกมาต้องมีไฟล์ของตัวเอง
   ต้องเพิ่มเข้า build script/Makefile (Part 078) และมีความเสี่ยงเรื่องพารามิเตอร์ไม่ตรงกันเพิ่มขึ้น
   (Part 031 ขั้นตอนที่ 304)
2. **เพิ่มภาระทางสมองให้ผู้อ่านโค้ดรุ่นถัดไป** — ผู้ที่ไม่คุ้นเคยกับ pattern ที่ใช้ต้องเสียเวลาทำ
   ความเข้าใจโครงสร้างที่ซับซ้อนกว่าที่จำเป็น ขัดกับหลักการ Clean Code ที่ Part 081 เน้นย้ำเรื่อง
   ความอ่านง่ายเป็นอันดับแรก
3. **ทำให้การ debug ยากขึ้น** — การไล่ตาม `CALL` ที่กระจายไปหลายไฟล์ (โดยเฉพาะแบบ data-driven
   dispatch จากขั้นตอนที่ 813 ที่ชื่อโปรแกรมมาจากตัวแปร ไม่ใช่ literal ตรง ๆ) ยากกว่าการไล่ตาม
   `IF`/`EVALUATE` ธรรมดาในไฟล์เดียว

### หลักการตัดสินใจที่แนะนำ

**เริ่มจากโค้ดที่ง่ายที่สุดเสมอ (ตาม Part 081) แล้ว refactor ไปสู่ Design Pattern ก็ต่อเมื่อมี
สัญญาณที่ชัดเจนว่าระบบต้องการความยืดหยุ่นนั้นจริง ๆ** เช่น:

- มี strategy ตั้งแต่ 3 แบบขึ้นไป **และ** มีแนวโน้มว่าจะเพิ่มอีกในอนาคตอันใกล้
- มีหลักฐานว่า business logic การสร้าง record ซับซ้อนขึ้นเรื่อย ๆ จนกระจัดกระจายและซ้ำซ้อนในหลาย
  จุดของระบบ (สัญญาณเดียวกับที่ Part 081 ขั้นตอนที่ 805 ใช้ตัดสินใจ Extract Subprogram)
- มีการร้องขอฟีเจอร์ใหม่ที่ "เพิ่ม variant" ของกระบวนการเดิมซ้ำแล้วซ้ำเล่า

### ข้อควรระวัง

- อย่าใช้ Design Pattern เป็น "เกณฑ์วัดความเก่ง" ของโปรแกรมเมอร์ — โค้ดที่ดีที่สุดคือโค้ดที่**แก้
  ปัญหาได้อย่างเรียบง่ายที่สุดเท่าที่จำเป็น** ไม่ใช่โค้ดที่แสดงความรู้เรื่อง pattern ให้มากที่สุด
- การตัดสินใจใช้หรือไม่ใช้ pattern ควรมาจาก**หลักฐานของความต้องการจริง** (เช่น requirement ที่
  ชัดเจนว่าจะมี variant เพิ่มขึ้น) ไม่ใช่การคาดเดาล่วงหน้าว่า "อนาคตอาจจะต้องการ" (หลักการนี้เรียก
  ว่า YAGNI — "You Aren't Gonna Need It" ในวงการวิศวกรรมซอฟต์แวร์)

### แบบฝึกหัดที่ 819.1

**โจทย์**: จงพิจารณาระบบคำนวณส่วนลดที่มีแค่ 2 กรณี ("สมาชิก" ลด 5%, "ไม่ใช่สมาชิก" ไม่ลด) และไม่มี
แผนจะเพิ่มกรณีอื่นในอนาคตเลย จงอธิบายว่าทำไมการใช้ Strategy Pattern เต็มรูปแบบ (แยกเป็น 2
subprogram + dispatch table) กับกรณีนี้จึงถือเป็น Over-Engineering

**เฉลย**: ด้วยแค่ 2 กรณีที่ไม่มีแนวโน้มจะเพิ่มอีก ต้นทุนของการสร้าง Strategy Pattern เต็มรูปแบบ
(ต้องมี 2 ไฟล์ subprogram แยก, ต้องมี paragraph dispatch, ต้องดูแลให้ "สัญญา" ของทั้งสอง subprogram
ตรงกันตลอดไป) **สูงกว่าประโยชน์ที่ได้รับมาก** เมื่อเทียบกับการเขียน
`IF WS-IS-MEMBER = "Y" COMPUTE WS-DISCOUNT = WS-AMOUNT * 0.05 ELSE MOVE 0 TO WS-DISCOUNT END-IF`
ตรง ๆ เพียงบรรทัดเดียวในโปรแกรมที่ใช้งาน ซึ่งอ่านและเข้าใจได้ทันทีโดยไม่ต้องไล่ตามไฟล์อื่นเลย หาก
ในอนาคตมีความต้องการเพิ่ม variant ที่สาม เมื่อนั้นจึงค่อย refactor จาก `IF` ธรรมดาไปสู่ `EVALUATE`
(Part 081 ขั้นตอนที่ 804) และหากมี variant เพิ่มขึ้นอีกจนเห็นแนวโน้มชัดเจนว่าจะมีเพิ่มเรื่อย ๆ
เมื่อนั้นจึงค่อยพิจารณา refactor ต่อไปสู่ Strategy Pattern เต็มรูปแบบ — การรอให้ความต้องการที่แท้จริง
ปรากฏชัดก่อนค่อยเพิ่มความซับซ้อนของ pattern คือแนวทางที่สอดคล้องกับหลักการ YAGNI และป้องกันการเสีย
เวลาไปกับโครงสร้างที่อาจไม่มีวันถูกใช้ประโยชน์เต็มที่เลย

---

## ขั้นตอนที่ 820: สรุปรวมและแบบฝึกหัดใหญ่ — นำ Pattern ไปประยุกต์กับระบบขนาดย่อม

### สรุปทั้ง 4 Pattern ที่เรียนมาใน Part นี้

| Pattern | กลไกหลักที่ใช้ | ปัญหาที่แก้ | ตัวอย่างที่สอน |
|---|---|---|---|
| Strategy | `EVALUATE` dispatch หรือตาราง + `CALL` ชื่อในตัวแปร | สลับอัลกอริทึมได้โดยผู้เรียกไม่ต้องรู้รายละเอียด | ขั้นตอนที่ 812–813 (Shipping cost) |
| Factory | Subprogram ที่รวมกฎการสร้าง record ตามประเภท | รวมกฎการสร้างข้อมูลที่ซับซ้อนไว้จุดเดียว | ขั้นตอนที่ 814 (AccountFactory) |
| Singleton | WORKING-STORAGE ของ subprogram ที่คงค่าข้าม CALL | มีสถานะเดียวที่ใช้ร่วมกันทั้งระบบ (ในขอบเขต 1 process) | ขั้นตอนที่ 815 (BatchCounter) |
| Template Method | Driver paragraph ตายตัว + `CALL` hook ที่เปลี่ยนได้ | โครงร่างกระบวนการเดิม แต่เนื้อหาแตกต่างกันได้ | ขั้นตอนที่ 816 (Report printing) |

### แบบฝึกหัดใหญ่ที่ 820.1 — ออกแบบระบบคำนวณค่าปรับชำระล่าช้า

**โจทย์**: จงออกแบบระบบขนาดย่อมที่คำนวณ**ค่าปรับชำระบิลล่าช้า** โดยมีข้อกำหนดดังนี้:

1. ลูกค้ามี 2 ประเภท: "N" (ลูกค้าทั่วไป) และ "G" (ลูกค้า Gold ที่ได้รับการยกเว้นบางส่วน)
2. ต้องมี subprogram "โรงงาน" ที่สร้าง record ข้อมูลลูกค้าใหม่พร้อมประเภทและยอดค้างชำระเริ่มต้น
   เป็น 0 (ใช้แนวคิด Factory จากขั้นตอนที่ 814)
3. การคำนวณค่าปรับต้องแยกเป็นคนละ subprogram ตามประเภทลูกค้า (ใช้แนวคิด Strategy จากขั้นตอนที่
   812): ลูกค้าทั่วไปปรับ 2% ของยอดค้างชำระต่อเดือนที่ล่าช้า, ลูกค้า Gold ปรับแค่ 0.5% ต่อเดือน
4. ทดสอบด้วยลูกค้า 2 คน: ลูกค้าทั่วไปยอดค้าง 5000.00 ล่าช้า 3 เดือน, ลูกค้า Gold ยอดค้าง 5000.00
   ล่าช้า 3 เดือนเท่ากัน — เปรียบเทียบผลลัพธ์

**เฉลย**: (โครงสร้างคำตอบ พร้อมโค้ดที่ควรคอมไพล์และทดสอบจริงตามแนวทางที่สอนมาตลอด Part นี้)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CUSTFACTORY.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-CUST-TYPE             PIC X(1).
       01  LK-OVERDUE-AMT           PIC 9(7)V99.
       01  LK-NEW-CUSTOMER.
           05  LK-CUST-TYPE-OUT     PIC X(1).
           05  LK-OVERDUE-OUT       PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-CUST-TYPE LK-OVERDUE-AMT
               LK-NEW-CUSTOMER.
       MAIN-PARA.
           MOVE LK-CUST-TYPE TO LK-CUST-TYPE-OUT.
           MOVE LK-OVERDUE-AMT TO LK-OVERDUE-OUT.
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PENALTYNORMAL.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-OVERDUE-AMT           PIC 9(7)V99.
       01  LK-MONTHS-LATE           PIC 9(2).
       01  LK-PENALTY               PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-OVERDUE-AMT LK-MONTHS-LATE
               LK-PENALTY.
       MAIN-PARA.
           COMPUTE LK-PENALTY ROUNDED =
               LK-OVERDUE-AMT * 0.02 * LK-MONTHS-LATE.
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PENALTYGOLD.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-OVERDUE-AMT           PIC 9(7)V99.
       01  LK-MONTHS-LATE           PIC 9(2).
       01  LK-PENALTY               PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-OVERDUE-AMT LK-MONTHS-LATE
               LK-PENALTY.
       MAIN-PARA.
           COMPUTE LK-PENALTY ROUNDED =
               LK-OVERDUE-AMT * 0.005 * LK-MONTHS-LATE.
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP820MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUST-TYPE             PIC X(1).
       01  WS-OVERDUE-AMT           PIC 9(7)V99.
       01  WS-CUSTOMER.
           05  WS-CUST-TYPE-F       PIC X(1).
           05  WS-OVERDUE-F         PIC 9(7)V99.
       01  WS-MONTHS-LATE           PIC 9(2) VALUE 3.
       01  WS-PENALTY               PIC 9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "N" TO WS-CUST-TYPE.
           MOVE 5000.00 TO WS-OVERDUE-AMT.
           PERFORM PROCESS-CUSTOMER.

           MOVE "G" TO WS-CUST-TYPE.
           MOVE 5000.00 TO WS-OVERDUE-AMT.
           PERFORM PROCESS-CUSTOMER.
           STOP RUN.

       PROCESS-CUSTOMER.
           CALL "CUSTFACTORY" USING WS-CUST-TYPE WS-OVERDUE-AMT
               WS-CUSTOMER.
           EVALUATE WS-CUST-TYPE-F
               WHEN "N"
                   CALL "PENALTYNORMAL" USING WS-OVERDUE-F
                       WS-MONTHS-LATE WS-PENALTY
               WHEN "G"
                   CALL "PENALTYGOLD" USING WS-OVERDUE-F
                       WS-MONTHS-LATE WS-PENALTY
           END-EVALUATE.
           DISPLAY "type=" WS-CUST-TYPE-F " overdue=" WS-OVERDUE-F
               " penalty=" WS-PENALTY.
```

**ผลลัพธ์ที่คาดหวัง (ตรวจสอบด้วยมือ)**: ลูกค้าทั่วไป: 5000.00 × 0.02 × 3 = 300.00; ลูกค้า Gold:
5000.00 × 0.005 × 3 = 75.00 — แสดงให้เห็นว่าลูกค้า Gold ได้รับการยกเว้นค่าปรับส่วนใหญ่ตามเงื่อนไข
ที่กำหนด และโครงสร้างทั้งหมดนี้ประกอบด้วย Factory (`CUSTFACTORY`) และ Strategy
(`PENALTYNORMAL`/`PENALTYGOLD`) ทำงานร่วมกันในรูปแบบเดียวกับที่สอนไว้ในขั้นตอนที่ 818 ทุกประการ

### ข้อควรระวัง

- แบบฝึกหัดนี้ควรถูกนำไปคอมไพล์และรันจริงเพื่อยืนยันผลลัพธ์ด้วยตัวคุณเอง ตามวินัยที่ Part นี้และ
  ทั้งหลักสูตรยึดถือมาตลอด — อย่าเชื่อคำตอบในเอกสารโดยไม่ทดลองรันจริงด้วยตัวเอง (ทบทวนหลักการจาก
  Part 001 ขั้นตอนที่ 10)
- ก่อนนำ pattern เหล่านี้ไปใช้ในระบบจริงขนาดใหญ่ ควรทบทวนขั้นตอนที่ 819 อีกครั้งเสมอ เพื่อยืนยันว่า
  ความซับซ้อนที่เพิ่มขึ้นนั้นคุ้มค่ากับปัญหาที่กำลังแก้ไขจริง ๆ

### แบบฝึกหัดที่ 820.2

**โจทย์**: จงขยายระบบค่าปรับข้างต้นให้รองรับลูกค้าประเภทที่ 3 คือ "V" (VIP) ที่**ไม่มีค่าปรับเลย
ไม่ว่าจะล่าช้ากี่เดือนก็ตาม** โดยใช้เทคนิคที่เหมาะสมที่สุดจากที่เรียนมาทั้ง Part

**เฉลย**: เพิ่ม subprogram ใหม่:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PENALTYVIP.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-OVERDUE-AMT           PIC 9(7)V99.
       01  LK-MONTHS-LATE           PIC 9(2).
       01  LK-PENALTY               PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-OVERDUE-AMT LK-MONTHS-LATE
               LK-PENALTY.
       MAIN-PARA.
           MOVE 0 TO LK-PENALTY.
           GOBACK.
```

แล้วเพิ่ม `WHEN "V" CALL "PENALTYVIP" USING WS-OVERDUE-F WS-MONTHS-LATE WS-PENALTY` เข้าไปใน
`EVALUATE` ของ `PROCESS-CUSTOMER` เหตุผลที่เลือกสร้าง subprogram ใหม่แทนที่จะเพิ่มเงื่อนไข `IF`
พิเศษเข้าไปใน `PENALTYNORMAL` หรือ `PENALTYGOLD` คือการรักษาหลักการ **Single Responsibility**
(แต่ละ strategy รับผิดชอบกฎของประเภทลูกค้าเดียวเท่านั้น) และรักษา**สัญญาที่สอดคล้องกัน**ระหว่างทุก
strategy (รับพารามิเตอร์ชุดเดียวกันเป๊ะ คืนค่า `LK-PENALTY` เหมือนกัน) ตามหลักการ Strategy Pattern
ที่สอนไว้ตลอด Part นี้ — แม้ตรรกะของ `PENALTYVIP` จะเรียบง่ายมาก (คืนค่า 0 เสมอ) แต่การให้มันเป็น
subprogram แยกต่างหากทำให้ระบบยังคง**ขยายได้อย่างสม่ำเสมอ**หากในอนาคตกฎของลูกค้า VIP เปลี่ยนแปลง
ซับซ้อนขึ้น (เช่น "VIP ไม่มีค่าปรับ เว้นแต่ล่าช้าเกิน 12 เดือน") ก็สามารถแก้ไขแค่ใน `PENALTYVIP`
โดยไม่กระทบ strategy อื่นเลยแม้แต่น้อย

---

## สรุปท้ายบท

Part นี้พาคุณสำรวจ Design Patterns คลาสสิก 4 แบบที่ประยุกต์เข้ากับธรรมชาติเชิงกระบวนการของ COBOL
โดยใช้กลไกที่เรียนมาตั้งแต่ต้นหลักสูตร (`CALL`, `EVALUATE`, `WORKING-STORAGE`) และทุกตัวอย่างผ่าน
การคอมไพล์และรันจริงแล้ว:

- แนวคิดพื้นฐานว่า Design Pattern เป็นแนวคิดการแก้ปัญหาที่เป็นนามธรรม ไม่ใช่ไวยากรณ์เฉพาะภาษา OOP
- **Strategy Pattern** ผ่าน `EVALUATE` dispatch (ขั้นตอนที่ 812) และแบบ data-driven ผ่านตาราง +
  `SEARCH` (ขั้นตอนที่ 813) ที่ทำให้เพิ่ม strategy ใหม่โดยไม่ต้องแก้โค้ด dispatch เดิม
- **Factory-like Pattern** ผ่าน subprogram ที่รวมกฎการสร้าง record ตามประเภทไว้จุดเดียว พร้อม
  บทเรียนจริงเรื่องอันตรายของการส่ง literal เข้า `CALL ... USING`
- **Singleton-like Pattern** ผ่านคุณสมบัติที่ WORKING-STORAGE ของ subprogram คงค่าข้ามการเรียก
  พร้อมข้อจำกัดสำคัญเรื่อง `CANCEL` และขอบเขตแค่ภายใน 1 process
- **Template Method Pattern** ผ่าน driver paragraph ที่ตายตัวเรียก hook subprogram ที่เปลี่ยนได้
- การเปรียบเทียบข้อจำกัดของ COBOL เทียบกับภาษา OOP โดยเฉพาะการขาดการตรวจสอบ "สัญญา" โดย compiler
  และแนวทางชดเชยด้วย Unit Test และ Code Review
- ตัวอย่างรวมที่ Factory และ Strategy ทำงานร่วมกันในกระบวนการทางธุรกิจเดียว
- หลักการสำคัญที่สุด: **อย่า Over-Engineer** — ใช้ Pattern เมื่อมีหลักฐานความต้องการจริงเท่านั้น
- แบบฝึกหัดใหญ่ที่ประยุกต์ทั้ง Factory และ Strategy เข้ากับระบบคำนวณค่าปรับชำระล่าช้า

Part นี้เป็น Part สุดท้ายก่อนถึงจุดสิ้นสุดของเฟส 5 (COBOL สมัยใหม่) — เทคนิคทั้งหมดตั้งแต่ Part
071 ถึง Part 082 (Docker, Cloud, CI/CD, Unit Testing, Git, Refactoring, และ Design Patterns) จะ
ถูกนำมาประกอบรวมกันในโปรเจกต์ก่อนจบเฟส Part ถัดไป (**Part 083**) จะเปลี่ยนมุมมองไปสู่การวิเคราะห์
ระบบเก่า: **Legacy System Analysis และ Reverse Engineering** — เทคนิคการทำความเข้าใจโค้ด COBOL
ขนาดใหญ่ที่ไม่มีเอกสารกำกับ ก่อนที่จะนำเทคนิค Refactoring และ Design Pattern จาก Part 081–082 ไป
ประยุกต์ใช้ปรับปรุงระบบเหล่านั้นให้ทันสมัยขึ้นในเฟสถัดไป

**[← กลับไป Part 081: Code Refactoring และ Clean Code สำหรับ COBOL](part-081-refactoring-clean-code.md)** | **[ไปยัง Part 083: Legacy System Analysis และ Reverse Engineering →](part-083-legacy-analysis.md)**
