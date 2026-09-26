# Part 011: EVALUATE Statement (เทียบเท่า switch/case) (ขั้นตอนที่ 101–110)

## คำนำของ Part นี้

ใน Part 010 เราจบท้ายด้วยโปรแกรมพิจารณาสินเชื่อที่ใช้ Nested IF ซ้อนกันถึง 4 ชั้น และสังเกตเห็นว่า
ยิ่งเงื่อนไขมากขึ้นเท่าไร ยิ่งต้องนับ `END-IF` ให้ครบและเยื้องโค้ดให้ถูกต้องมากขึ้นเท่านั้น จนโค้ดเริ่มอ่าน
ยากและดูแลรักษายาก Part นี้จะแนะนำคำสั่งที่ COBOL ออกแบบมาเพื่อแก้ปัญหานี้โดยเฉพาะ นั่นคือ
**`EVALUATE`** — คำสั่งที่เทียบเท่ากับ `switch`/`case` ในภาษาโปรแกรมสมัยใหม่อย่าง C, Java หรือ
Python (`match`) แต่ทรงพลังกว่ามาก

`EVALUATE` ไม่ได้เป็นเพียงทางลัดสำหรับเทียบค่าเดียวเหมือน switch ทั่วไปเท่านั้น แต่ยังรองรับการเทียบ
เงื่อนไขแบบ boolean (`EVALUATE TRUE`), การเทียบหลายค่าพร้อมกัน, ช่วงของค่า (`THRU`), และแม้แต่
การเทียบหลายตัวแปรพร้อมกันในคำสั่งเดียว (`ALSO`) ซึ่งเป็นความสามารถที่ภาษาโปรแกรมจำนวนมากไม่มี

Part นี้จะพาคุณไล่เรียงตั้งแต่ไวยากรณ์พื้นฐานของ `EVALUATE`, รูปแบบ `EVALUATE TRUE` ที่ทรงพลัง
ที่สุด, การจับคู่หลายค่า, ช่วงค่าด้วย `THRU`, การเทียบหลายตัวแปรด้วย `ALSO`, `WHEN OTHER`,
การผสานกับ Condition Names, การใช้งานร่วมกับลูป, ไปจนถึงการเปรียบเทียบกับ Nested IF อย่างตรงไปตรงมา
ปิดท้ายด้วยโปรแกรมเมนูร้านอาหารที่ใช้ `EVALUATE` เป็นแกนหลัก

---

## ขั้นตอนที่ 101: EVALUATE พื้นฐาน — เทียบเท่า switch/case

### แนวคิด

รูปแบบพื้นฐานที่สุดของ `EVALUATE` คือการนำตัวแปรหรือค่าหนึ่งตัว (เรียกว่า **subject**) มาเทียบกับ
ค่าที่เป็นไปได้หลายค่า (เรียกว่า **object** ในแต่ละ `WHEN`) ทีละเงื่อนไขตามลำดับจากบนลงล่าง
เมื่อพบค่าที่ตรงกัน จะทำงานเฉพาะบล็อกคำสั่งของ `WHEN` นั้น แล้วออกจาก `EVALUATE` ทันที (ไม่มี
fall-through เหมือน `switch` ในภาษา C ที่ต้องเขียน `break` กำกับทุกครั้ง)

```
EVALUATE <subject>
    WHEN <value-1>
        <statement(s)>
    WHEN <value-2>
        <statement(s)>
    WHEN OTHER
        <statement(s)>
END-EVALUATE
```

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-BASIC-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DAY-NUMBER          PIC 9(1)  VALUE 3.

       PROCEDURE DIVISION.
           EVALUATE WS-DAY-NUMBER
               WHEN 1
                   DISPLAY "MONDAY"
               WHEN 2
                   DISPLAY "TUESDAY"
               WHEN 3
                   DISPLAY "WEDNESDAY"
               WHEN 4
                   DISPLAY "THURSDAY"
               WHEN 5
                   DISPLAY "FRIDAY"
               WHEN OTHER
                   DISPLAY "WEEKEND"
           END-EVALUATE.
           STOP RUN.
```

### อธิบายโค้ด

- `EVALUATE WS-DAY-NUMBER` — กำหนดให้ `WS-DAY-NUMBER` เป็น subject ที่จะถูกนำไปเทียบกับค่า
  ในแต่ละ `WHEN`
- COBOL จะตรวจสอบทีละ `WHEN` จากบนลงล่าง: `WS-DAY-NUMBER` (3) ไม่ตรงกับ 1, ไม่ตรงกับ 2, แต่
  ตรงกับ 3 พอดี จึงทำงานเฉพาะ `DISPLAY "WEDNESDAY"` แล้วออกจาก `EVALUATE` ทันที **ไม่ตรวจสอบ
  `WHEN 4`, `WHEN 5` หรือ `WHEN OTHER` ต่ออีกเลย**
- `END-EVALUATE` เป็น scope terminator (เช่นเดียวกับ `END-IF`) บอกขอบเขตจบของคำสั่ง `EVALUATE`
  อย่างชัดเจน

### ผลลัพธ์ที่ได้จากการรันจริง

```
WEDNESDAY
```

### เปรียบเทียบกับ Nested IF

หากเขียนด้วย Nested IF (แบบที่เรียนใน Part 010) ตรรกะเดียวกันนี้ต้องเขียนถึง 5 ชั้น IF ซ้อนกัน
พร้อม `END-IF` 5 ตัว ในขณะที่ `EVALUATE` ใช้ `END-EVALUATE` เพียงตัวเดียวปิดท้ายทั้งหมด แม้จะมี
กี่ `WHEN` ก็ตาม — นี่คือข้อได้เปรียบสำคัญที่สุดของ `EVALUATE` เมื่อมีทางเลือกจำนวนมาก

### ข้อควรระวัง

- **`EVALUATE` ไม่มี fall-through เหมือน `switch` ในภาษา C** เมื่อพบ `WHEN` ที่ตรงกันแล้ว จะทำงาน
  เฉพาะบล็อกนั้นแล้วออกทันที ไม่ต้องกังวลเรื่องลืมใส่ `break` เหมือนภาษาอื่น (ซึ่งเป็นแหล่งบั๊กที่มีชื่อเสียง
  มากในภาษา C/Java)
- ชนิดของ subject กับ object ใน `WHEN` ควรเป็นชนิดที่เปรียบเทียบกันได้อย่างสมเหตุสมผล (ตัวเลขกับ
  ตัวเลข, ข้อความกับข้อความ) มิเช่นนั้นอาจได้ผลลัพธ์การเปรียบเทียบที่ไม่ตรงกับที่คาดหวัง

### แบบฝึกหัดที่ 101.1

**โจทย์**: จงเขียน `EVALUATE` สำหรับตัวแปร `WS-TRAFFIC-LIGHT PIC X(1)` ที่แสดง "STOP" เมื่อเป็น
"R", "GO" เมื่อเป็น "G", "SLOW DOWN" เมื่อเป็น "Y" และ "UNKNOWN SIGNAL" สำหรับกรณีอื่น

**เฉลย**:
```cobol
EVALUATE WS-TRAFFIC-LIGHT
    WHEN "R"
        DISPLAY "STOP"
    WHEN "G"
        DISPLAY "GO"
    WHEN "Y"
        DISPLAY "SLOW DOWN"
    WHEN OTHER
        DISPLAY "UNKNOWN SIGNAL"
END-EVALUATE
```

---

## ขั้นตอนที่ 102: EVALUATE TRUE — เทียบเท่า IF-ELSE IF แบบต่อเนื่อง

### แนวคิด

รูปแบบพื้นฐานในขั้นตอนที่แล้วเหมาะกับการเทียบ**ค่าเดียวกับหลายค่าคงที่** แต่ถ้าเราต้องการตรวจสอบ
**เงื่อนไขที่ต่างกันในแต่ละ WHEN** (เช่น ช่วงคะแนนที่ไม่ได้เป็นค่าคงที่ตายตัว) เราใช้รูปแบบพิเศษที่ทรงพลัง
ที่สุดของ `EVALUATE` คือ **`EVALUATE TRUE`**

เมื่อ subject เป็นคำว่า `TRUE` ตรง ๆ แต่ละ `WHEN` จะเขียน**เงื่อนไข boolean เต็มรูป** (เหมือนที่เขียน
ใน `IF`) แทนที่จะเขียนแค่ค่าคงที่ COBOL จะประเมินเงื่อนไขในแต่ละ `WHEN` ทีละอันจนกว่าจะเจอเงื่อนไข
ที่เป็นจริง (TRUE) พอดี ซึ่งมีพฤติกรรมเทียบเท่ากับ `IF ... ELSE IF ... ELSE IF ...` ต่อเนื่องกันทุกประการ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-TRUE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SCORE               PIC 9(3)  VALUE 78.
       01  WS-GRADE               PIC X(1).

       PROCEDURE DIVISION.
      *> EVALUATE TRUE checks each WHEN's own condition in order,
      *> just like a chain of independent IF / ELSE IF statements.
           EVALUATE TRUE
               WHEN WS-SCORE >= 90
                   MOVE "A" TO WS-GRADE
               WHEN WS-SCORE >= 80
                   MOVE "B" TO WS-GRADE
               WHEN WS-SCORE >= 70
                   MOVE "C" TO WS-GRADE
               WHEN WS-SCORE >= 60
                   MOVE "D" TO WS-GRADE
               WHEN OTHER
                   MOVE "F" TO WS-GRADE
           END-EVALUATE.

           DISPLAY "SCORE " WS-SCORE " GETS GRADE " WS-GRADE.
           STOP RUN.
```

### อธิบายโค้ด

- `EVALUATE TRUE` — subject เป็นค่าคงที่ `TRUE` ตรง ๆ หมายความว่าเราต้องการหา `WHEN` แรกที่
  เงื่อนไขของมันเป็นจริง
- `WHEN WS-SCORE >= 90` — เงื่อนไขเต็มรูป ไม่ใช่ค่าคงที่ธรรมดา ตรวจสอบว่า `WS-SCORE` (78)
  มากกว่าหรือเท่ากับ 90 หรือไม่ — เป็นเท็จ จึงข้ามไปเงื่อนไขถัดไป
- COBOL ตรวจสอบไล่ลงมาเรื่อย ๆ: `>= 90` เท็จ, `>= 80` เท็จ, `>= 70` **เป็นจริง** (78 >= 70)
  จึงทำงาน `MOVE "C" TO WS-GRADE` แล้วออกจาก EVALUATE ทันที ไม่ตรวจสอบ `>= 60` หรือ
  `WHEN OTHER` ต่ออีก

### ผลลัพธ์ที่ได้จากการรันจริง

```
SCORE 078 GETS GRADE C
```

### ทำไม EVALUATE TRUE ถึงสำคัญที่สุด

`EVALUATE TRUE` คือรูปแบบที่ใช้บ่อยที่สุดในโค้ด COBOL ระดับมืออาชีพ เพราะมันครอบคลุมทุกสถานการณ์
ที่ Nested IF ทำได้ แต่เขียนได้กระชับและอ่านง่ายกว่ามาก โดยเฉพาะเมื่อมีมากกว่า 3 เงื่อนไขขึ้นไป
สังเกตว่าตัวอย่างนี้เทียบเท่ากับโค้ด Nested IF ที่เคยเขียนใน Part 010 ขั้นตอนที่ 92 (การหาเกรดจาก
คะแนน) ทุกประการ แต่ไม่ต้องนับ `END-IF` ซ้อนกันเลย

### ข้อควรระวัง

- **ลำดับของ `WHEN` ใน `EVALUATE TRUE` มีความสำคัญมาก** เช่นเดียวกับ Nested IF เพราะ COBOL
  ตรวจสอบจากบนลงล่างและหยุดที่เงื่อนไขแรกที่เป็นจริง หากเรียงลำดับผิด (เช่น เอา `>= 70` ไว้ก่อน
  `>= 90`) ผลลัพธ์จะผิดทันทีเพราะคะแนน 95 จะเข้าเงื่อนไข `>= 70` ก่อนโดยไม่มีโอกาสตรวจสอบ `>= 90`
  เลย
- ต้องมี `WHEN OTHER` เพื่อรองรับกรณีที่ไม่มีเงื่อนไขใดเป็นจริงเลย (จะกล่าวถึงรายละเอียดเพิ่มเติมใน
  ขั้นตอนที่ 106)

### แบบฝึกหัดที่ 102.1

**โจทย์**: จงเขียน `EVALUATE TRUE` เพื่อจำแนกตัวเลข `WS-NUM` ว่าเป็น "POSITIVE", "NEGATIVE"
หรือ "ZERO" (เทียบกับที่เคยเขียนด้วย Nested IF ใน Part 010 แบบฝึกหัด 92.1)

**เฉลย**:
```cobol
EVALUATE TRUE
    WHEN WS-NUM > 0
        DISPLAY "POSITIVE"
    WHEN WS-NUM < 0
        DISPLAY "NEGATIVE"
    WHEN OTHER
        DISPLAY "ZERO"
END-EVALUATE
```

---

## ขั้นตอนที่ 103: การจับคู่หลายค่าใน WHEN เดียวกัน

### แนวคิด

บางครั้งเราต้องการให้หลายค่าที่แตกต่างกันไปยังผลลัพธ์เดียวกัน (เช่น เกรด A และ B ถือว่า "ผ่านเกียรตินิยม")
COBOL ให้เราเขียน `WHEN` ติดกันหลายตัว **โดยไม่มีคำสั่งใด ๆ คั่นระหว่างกัน** เพื่อหมายความว่า
"ค่าใดค่าหนึ่งในกลุ่มนี้ก็ให้ทำงานบล็อกเดียวกัน" ซึ่งแตกต่างจาก `switch` ในภาษา C ที่ใช้ fall-through
(ไม่ใส่ `break`) เพื่อทำสิ่งเดียวกัน — ใน COBOL การเขียนแบบนี้ **ไม่ใช่ fall-through** แต่เป็นไวยากรณ์
ที่ถูกออกแบบมาให้หมายถึง "OR" ของหลายเงื่อนไขอย่างชัดเจนตั้งแต่แรก

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-MULTIVALUE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-LETTER-GRADE        PIC X(1)  VALUE "B".

       PROCEDURE DIVISION.
      *> Stacking several WHEN lines with NO statement between
      *> them means "match ANY one of these values" -- an OR,
      *> without ever falling through like a C switch does.
           EVALUATE WS-LETTER-GRADE
               WHEN "A"
               WHEN "B"
                   DISPLAY "PASSED WITH HONORS"
               WHEN "C"
               WHEN "D"
                   DISPLAY "PASSED"
               WHEN "F"
                   DISPLAY "FAILED"
               WHEN OTHER
                   DISPLAY "UNKNOWN GRADE CODE"
           END-EVALUATE.
           STOP RUN.
```

### อธิบายโค้ด

- `WHEN "A"` ตามด้วย `WHEN "B"` ทันที (ไม่มี `DISPLAY` คั่นกลาง) — หมายความว่าถ้า
  `WS-LETTER-GRADE` เป็น "A" **หรือ** "B" ให้ทำงาน `DISPLAY "PASSED WITH HONORS"` ร่วมกัน
- เช่นเดียวกัน `WHEN "C"` กับ `WHEN "D"` ติดกันหมายถึงทั้งคู่ให้ผลลัพธ์ "PASSED" เหมือนกัน
- `WS-LETTER-GRADE` มีค่า "B" ตรงกับกลุ่มแรก จึงแสดง "PASSED WITH HONORS"

### ผลลัพธ์ที่ได้จากการรันจริง

```
PASSED WITH HONORS
```

### ข้อควรระวัง

- **ต้องแยกให้ออกระหว่างรูปแบบนี้กับ fall-through ของภาษา C**: ใน COBOL การเขียน `WHEN` ติดกัน
  หลายตัวโดยไม่มีคำสั่งคั่น เป็นไวยากรณ์ที่ตั้งใจออกแบบมาให้เป็น "OR" อย่างชัดเจน ไม่ใช่ "ลืมใส่ break"
  เหมือนที่มักเป็นบั๊กในภาษา C — เข้าใจความแตกต่างนี้จะช่วยไม่ให้สับสนเมื่อย้ายความรู้ข้ามภาษา
- ห้ามใส่คำสั่งใด ๆ ระหว่าง `WHEN` ที่ต้องการรวมกลุ่ม เพราะถ้าใส่คำสั่งเข้าไป มันจะกลายเป็นคนละกลุ่ม
  ทันที (เช่น `WHEN "A" DISPLAY "X" WHEN "B" DISPLAY "Y"` คือคนละเงื่อนไข ไม่ใช่กลุ่มเดียวกัน)

### แบบฝึกหัดที่ 103.1

**โจทย์**: จงเขียน `EVALUATE` สำหรับ `WS-MONTH-NUMBER PIC 9(2)` ที่แสดง "Q1" สำหรับเดือน 1, 2, 3
และ "Q2" สำหรับเดือน 4, 5, 6 (แสดงเฉพาะสองไตรมาสแรกพอ)

**เฉลย**:
```cobol
EVALUATE WS-MONTH-NUMBER
    WHEN 1
    WHEN 2
    WHEN 3
        DISPLAY "Q1"
    WHEN 4
    WHEN 5
    WHEN 6
        DISPLAY "Q2"
    WHEN OTHER
        DISPLAY "OTHER QUARTER"
END-EVALUATE
```

---

## ขั้นตอนที่ 104: ช่วงค่าด้วย WHEN ... THRU

### แนวคิด

เช่นเดียวกับ Condition Names ที่รองรับช่วงค่าด้วย `THRU` (Part 010 ขั้นตอนที่ 97) `EVALUATE`
ก็รองรับการเทียบ**ช่วงของค่า**โดยตรงในแต่ละ `WHEN` เช่นกัน ทำให้เขียนโค้ดจำแนกช่วงคะแนนหรือช่วง
ตัวเลขได้กระชับกว่าการเขียนเงื่อนไข `>=` และ `<=` คู่กันหลายรอบมาก

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-THRU-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-EXAM-SCORE          PIC 9(3)  VALUE 85.

       PROCEDURE DIVISION.
           EVALUATE WS-EXAM-SCORE
               WHEN 90 THRU 100
                   DISPLAY "GRADE: A"
               WHEN 80 THRU 89
                   DISPLAY "GRADE: B"
               WHEN 70 THRU 79
                   DISPLAY "GRADE: C"
               WHEN 0 THRU 69
                   DISPLAY "GRADE: F"
               WHEN OTHER
                   DISPLAY "INVALID SCORE"
           END-EVALUATE.
           STOP RUN.
```

### อธิบายโค้ด

- `WHEN 90 THRU 100` — ตรงกับค่าตั้งแต่ 90 ถึง 100 (รวมทั้งสองปลาย) เทียบเท่ากับการเขียน
  `EVALUATE TRUE WHEN WS-EXAM-SCORE >= 90 AND WS-EXAM-SCORE <= 100` แต่กระชับกว่ามาก
- `WS-EXAM-SCORE` มีค่า 85 ซึ่งอยู่ในช่วง 80 ถึง 89 พอดี จึงแสดง "GRADE: B"
- สังเกตว่านี่ใช้ subject เป็น `WS-EXAM-SCORE` ตรง ๆ (ไม่ใช่ `TRUE`) เพราะ `THRU` ใช้ได้ทั้งกับ
  รูปแบบพื้นฐานและรูปแบบ `EVALUATE TRUE`

### ผลลัพธ์ที่ได้จากการรันจริง

```
GRADE: B
```

### ข้อควรระวัง

- ช่วงที่กำหนดด้วย `THRU` ในแต่ละ `WHEN` **ไม่ควรทับซ้อนกัน** เหมือนกับ 88-level ที่เรียนใน Part 010
  หากช่วงทับซ้อนกัน COBOL จะใช้ `WHEN` แรกที่ตรงกันเสมอ (ตามลำดับบนลงล่าง) ซึ่งอาจไม่ใช่พฤติกรรม
  ที่ผู้เขียนตั้งใจ
- ลำดับตัวเลขใน `THRU` ต้องเรียงจากน้อยไปมากเสมอ (`90 THRU 100` ถูกต้อง, `100 THRU 90` ผิด
  ไวยากรณ์หรือไม่มีค่าใดตรงกันเลย)

### แบบฝึกหัดที่ 104.1

**โจทย์**: จงเขียน `EVALUATE` สำหรับ `WS-AGE PIC 9(3)` ที่แสดง "CHILD" สำหรับอายุ 0-12,
"TEENAGER" สำหรับ 13-19 และ "ADULT" สำหรับ 20 ปีขึ้นไป (สูงสุด 150)

**เฉลย**:
```cobol
EVALUATE WS-AGE
    WHEN 0 THRU 12
        DISPLAY "CHILD"
    WHEN 13 THRU 19
        DISPLAY "TEENAGER"
    WHEN 20 THRU 150
        DISPLAY "ADULT"
    WHEN OTHER
        DISPLAY "INVALID AGE"
END-EVALUATE
```

---

## ขั้นตอนที่ 105: EVALUATE ... ALSO — เทียบหลายตัวแปรพร้อมกัน

### แนวคิด

ความสามารถที่โดดเด่นที่สุดอย่างหนึ่งของ `EVALUATE` ที่ภาษาโปรแกรมทั่วไปแทบไม่มีคือการเทียบ
**หลาย subject พร้อมกัน** ด้วยคำเชื่อม **`ALSO`** ทำให้เขียนตารางการตัดสินใจ (decision table)
ที่ขึ้นกับหลายปัจจัยพร้อมกันได้อย่างเป็นระเบียบ แทนที่จะต้องซ้อน `IF` ภายใน `IF` หลายชั้น

```
EVALUATE <subject-1> ALSO <subject-2>
    WHEN <value-1a> ALSO <value-2a>
        <statement(s)>
    ...
END-EVALUATE
```

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-ALSO-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-TYPE       PIC X(1)  VALUE "R".
       01  WS-ORDER-AMOUNT        PIC 9(6)V99 VALUE 1500.00.
       01  WS-DISCOUNT-PERCENT    PIC 9(2)  VALUE 0.

       PROCEDURE DIVISION.
      *> EVALUATE ... ALSO ... lets us test TWO (or more)
      *> independent subjects at the same time, like a
      *> two-column decision table.
           EVALUATE WS-CUSTOMER-TYPE ALSO TRUE
               WHEN "V" ALSO WS-ORDER-AMOUNT >= 1000.00
                   MOVE 20 TO WS-DISCOUNT-PERCENT
               WHEN "V" ALSO WS-ORDER-AMOUNT < 1000.00
                   MOVE 15 TO WS-DISCOUNT-PERCENT
               WHEN "R" ALSO WS-ORDER-AMOUNT >= 1000.00
                   MOVE 10 TO WS-DISCOUNT-PERCENT
               WHEN "R" ALSO WS-ORDER-AMOUNT < 1000.00
                   MOVE 5 TO WS-DISCOUNT-PERCENT
               WHEN OTHER
                   MOVE 0 TO WS-DISCOUNT-PERCENT
           END-EVALUATE.

           DISPLAY "CUSTOMER TYPE : " WS-CUSTOMER-TYPE.
           DISPLAY "ORDER AMOUNT  : " WS-ORDER-AMOUNT.
           DISPLAY "DISCOUNT      : " WS-DISCOUNT-PERCENT "%".
           STOP RUN.
```

### อธิบายโค้ด

- `EVALUATE WS-CUSTOMER-TYPE ALSO TRUE` — subject ตัวแรกคือ `WS-CUSTOMER-TYPE` (ประเภท
  ลูกค้า) และ subject ตัวที่สองคือ `TRUE` (เพื่อให้ object ตัวที่สองในแต่ละ `WHEN` เป็นเงื่อนไขเต็มรูป
  ผสมกับ `EVALUATE TRUE` ที่เรียนในขั้นตอนที่ 102)
- `WHEN "V" ALSO WS-ORDER-AMOUNT >= 1000.00` — ตรงกับกรณีที่ `WS-CUSTOMER-TYPE` เป็น "V"
  **และ** `WS-ORDER-AMOUNT >= 1000.00` เป็นจริง**พร้อมกันทั้งคู่**
- ในตัวอย่างนี้ `WS-CUSTOMER-TYPE` เป็น "R" (ลูกค้าทั่วไป) และ `WS-ORDER-AMOUNT` เป็น 1500.00
  (>= 1000.00) จึงตรงกับ `WHEN "R" ALSO WS-ORDER-AMOUNT >= 1000.00` ได้ส่วนลด 10%

### ผลลัพธ์ที่ได้จากการรันจริง

```
CUSTOMER TYPE : R
ORDER AMOUNT  : 001500.00
DISCOUNT      : 10%
```

### เปรียบเทียบกับ Nested IF

หากเขียนตรรกะเดียวกันด้วย Nested IF จะต้องมี IF ซ้อนกัน 2 ชั้น (ชั้นนอกตรวจประเภทลูกค้า ชั้นในตรวจ
ยอดสั่งซื้อ) ซึ่งเมื่อมีปัจจัยมากกว่า 2 อย่าง หรือแต่ละปัจจัยมีหลายกรณี จำนวนชั้นของ Nested IF จะเพิ่ม
ขึ้นอย่างรวดเร็วจนอ่านยากมาก ในขณะที่ `EVALUATE ... ALSO` ยังคงแบนราบ (flat) และอ่านเป็นตาราง
การตัดสินใจได้ชัดเจนแม้จะมีหลายเงื่อนไขก็ตาม

### ข้อควรระวัง

- **จำนวน object ในแต่ละ `WHEN` ต้องตรงกับจำนวน subject ที่ประกาศไว้ใน `EVALUATE` เสมอ**
  (ในตัวอย่างนี้มี 2 subject จึงต้องมี 2 object คั่นด้วย `ALSO` ในทุก `WHEN`) หากใส่ไม่ครบจะเกิด
  compile error ทันที
- ยิ่งมี subject หลายตัวและแต่ละตัวมีหลายกรณี จำนวน `WHEN` ที่ต้องเขียนจะเพิ่มขึ้นแบบทวีคูณ
  (จำนวนกรณีของ subject ตัวแรก × จำนวนกรณีของ subject ตัวที่สอง) ควรพิจารณาความซับซ้อนนี้
  ก่อนเลือกใช้ `ALSO` กับตารางการตัดสินใจที่มีปัจจัยมากเกินไป

### แบบฝึกหัดที่ 105.1

**โจทย์**: จงอธิบายว่าทำไมตัวอย่างข้างต้นจึงต้องมี `WHEN OTHER` ปิดท้าย ทั้งที่ดูเหมือนจะครอบคลุม
ทุกกรณีของ `WS-CUSTOMER-TYPE` ("V" และ "R") กับทุกกรณีของยอดสั่งซื้อ (>= 1000 และ < 1000)
ไปแล้ว

**เฉลยแนวทาง**: เพราะ `WS-CUSTOMER-TYPE` เป็น `PIC X(1)` ที่รับค่าได้ทุกตัวอักษร ไม่ได้จำกัดแค่ "V"
กับ "R" เท่านั้น หากมีค่าอื่นเข้ามา (เช่น "G" สำหรับลูกค้าทั่วไปประเภทใหม่ หรือค่าว่าง/ขยะจากข้อมูล
ที่ไม่ถูกต้อง) จะไม่ตรงกับ `WHEN` ใดเลยในสี่กรณีที่กำหนดไว้ `WHEN OTHER` จึงจำเป็นเสมอเพื่อรองรับ
ค่าที่ไม่คาดคิดเหล่านี้ และป้องกันไม่ให้ `WS-DISCOUNT-PERCENT` เก็บค่าเก่าที่ค้างอยู่โดยไม่ได้ตั้งใจ

---

## ขั้นตอนที่ 106: WHEN OTHER — กรณีเริ่มต้นและความสำคัญของมัน

### แนวคิด

`WHEN OTHER` ทำหน้าที่เหมือน `default` ใน `switch` ของภาษาอื่น คือดักจับทุกกรณีที่ไม่ตรงกับ `WHEN`
ใดเลยก่อนหน้า **สิ่งสำคัญที่ต้องเข้าใจให้ชัดเจนคือ `WHEN OTHER` ไม่ใช่ข้อบังคับทางไวยากรณ์** COBOL
ยอมให้เขียน `EVALUATE` โดยไม่มี `WHEN OTHER` ได้ แต่ผลที่ตามมาคือถ้าไม่มีค่าใดตรงกับ `WHEN` เลย
**`EVALUATE` จะไม่ทำอะไรเลยแม้แต่บรรทัดเดียว** แล้วข้ามไปทำงานหลัง `END-EVALUATE` ต่อทันที
โดยไม่มี error หรือคำเตือนใด ๆ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-OTHER-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STATUS-CODE         PIC X(1)  VALUE "Z".

       PROCEDURE DIVISION.
           EVALUATE WS-STATUS-CODE
               WHEN "A"
                   DISPLAY "STATUS: ACTIVE"
               WHEN "S"
                   DISPLAY "STATUS: SUSPENDED"
               WHEN "C"
                   DISPLAY "STATUS: CLOSED"
               WHEN OTHER
                   DISPLAY "STATUS: UNKNOWN CODE '" WS-STATUS-CODE
                           "' -- PLEASE CHECK DATA SOURCE."
           END-EVALUATE.

      *> WHEN OTHER is NOT required by the compiler, but skipping
      *> it means an unmatched value falls through EVALUATE doing
      *> absolutely nothing -- often a silent bug.
           STOP RUN.
```

### อธิบายโค้ด

`WS-STATUS-CODE` มีค่า "Z" ซึ่งไม่ตรงกับ "A", "S" หรือ "C" ที่กำหนดไว้เลย จึงตกไปที่ `WHEN OTHER`
ซึ่งแสดงข้อความแจ้งเตือนที่ชัดเจนว่าพบรหัสที่ไม่รู้จัก พร้อมแนะนำให้ตรวจสอบแหล่งข้อมูล — นี่คือการ
ออกแบบที่ดี เพราะแทนที่จะปล่อยให้โปรแกรม "เงียบ" ไปเฉย ๆ เมื่อเจอข้อมูลผิดปกติ เราทำให้ปัญหา
"มองเห็นได้" ทันที

### ผลลัพธ์ที่ได้จากการรันจริง

```
STATUS: UNKNOWN CODE 'Z' -- PLEASE CHECK DATA SOURCE.
```

### ทดลองคิดตาม: ถ้าไม่มี WHEN OTHER จะเกิดอะไรขึ้น

หากลบ `WHEN OTHER` ออกจากตัวอย่างข้างต้น แล้วรันด้วยค่า "Z" เหมือนเดิม ผลลัพธ์คือ**ไม่มีข้อความ
ใด ๆ แสดงออกมาเลยจาก EVALUATE นี้** โปรแกรมจะข้ามไปทำงานคำสั่งถัดไปหลัง `END-EVALUATE` ทันที
เสมือนกับว่าไม่มีอะไรเกิดขึ้น ซึ่งในโปรแกรมจริงที่ซับซ้อนกว่านี้ การ "เงียบ" แบบนี้อาจทำให้ตัวแปรผลลัพธ์
ค้างค่าเก่าที่ไม่ถูกต้องไว้ และสร้างบั๊กที่ตรวจจับได้ยากมากในภายหลัง

### ข้อควรระวัง

- **แนวปฏิบัติที่ดีที่สุดคือใส่ `WHEN OTHER` ในทุก `EVALUATE` เสมอ** แม้จะมั่นใจว่าครอบคลุมทุกกรณี
  ที่เป็นไปได้แล้วก็ตาม เพราะข้อมูลจากภายนอก (ผู้ใช้, ไฟล์, ระบบอื่น) มักมีค่าที่ไม่คาดคิดปรากฏขึ้นได้
  เสมอในโลกจริง
- `WHEN OTHER` ต้องเป็น `WHEN` **ตัวสุดท้าย** เสมอ ห้ามวางไว้กลางหรือต้นของรายการ `WHEN`
  (ถ้าวางไว้ก่อน มันจะจับทุกกรณีไปหมดตั้งแต่แรก ทำให้ `WHEN` ที่ตามมาไม่มีโอกาสได้ทำงานเลย)

### แบบฝึกหัดที่ 106.1

**โจทย์**: จงอธิบายผลลัพธ์ที่จะเกิดขึ้น หากมีคนเขียน `EVALUATE` โดยวาง `WHEN OTHER` ไว้เป็น
`WHEN` ตัวแรกสุด (ก่อน `WHEN "A"`)

**เฉลยแนวทาง**: เนื่องจาก `WHEN OTHER` หมายถึง "ทุกกรณีที่ไม่ตรงกับที่ระบุไว้ก่อนหน้า" หากวางไว้
เป็นตัวแรกสุด มันจะกลายเป็น "จับคู่กับทุกค่าที่เป็นไปได้ทั้งหมด" ทันที เพราะยังไม่มี `WHEN` อื่นมาก่อน
เลย ผลคือทุกครั้งที่ `EVALUATE` ทำงาน จะเข้า `WHEN OTHER` เสมอ ไม่ว่า subject จะมีค่าอะไรก็ตาม
ทำให้ `WHEN "A"`, `WHEN "S"`, `WHEN "C"` ที่ตามมาไม่มีโอกาสได้ทำงานเลยแม้แต่ครั้งเดียว
ซึ่งเป็นความผิดพลาดทางตรรกะที่ร้ายแรงมาก

---

## ขั้นตอนที่ 107: EVALUATE ร่วมกับ Condition Names (88-level)

### แนวคิด

`EVALUATE TRUE` ผสานเข้ากับ Condition Names (88-level) ที่เรียนใน Part 010 ได้อย่างสวยงามที่สุด
เพราะแต่ละ `WHEN` สามารถเขียนชื่อ 88-level ตรง ๆ แทนที่จะเขียนเงื่อนไขเปรียบเทียบค่าดิบ ทำให้โค้ด
อ่านเป็นภาษาอังกฤษได้เกือบสมบูรณ์แบบ ตรงตามปรัชญาการออกแบบ COBOL ของ Grace Hopper ที่กล่าวถึง
ตั้งแต่ Part 001

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-CONDITION-NAME-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ORDER-STATUS        PIC X(1)  VALUE "S".
           88  ORDER-IS-PENDING           VALUE "P".
           88  ORDER-IS-SHIPPED           VALUE "S".
           88  ORDER-IS-DELIVERED         VALUE "D".
           88  ORDER-IS-CANCELLED         VALUE "C".

       PROCEDURE DIVISION.
      *> EVALUATE TRUE reads beautifully with 88-level condition
      *> names -- each WHEN becomes a self-documenting sentence.
           EVALUATE TRUE
               WHEN ORDER-IS-PENDING
                   DISPLAY "YOUR ORDER IS BEING PREPARED."
               WHEN ORDER-IS-SHIPPED
                   DISPLAY "YOUR ORDER IS ON THE WAY."
               WHEN ORDER-IS-DELIVERED
                   DISPLAY "YOUR ORDER HAS ARRIVED."
               WHEN ORDER-IS-CANCELLED
                   DISPLAY "YOUR ORDER WAS CANCELLED."
               WHEN OTHER
                   DISPLAY "UNKNOWN ORDER STATUS."
           END-EVALUATE.
           STOP RUN.
```

### อธิบายโค้ด

- `WHEN ORDER-IS-PENDING` — อ่านได้ตรงตัวว่า "เมื่อคำสั่งซื้อกำลังรอดำเนินการ" แทนที่จะต้องเขียน
  `WHEN WS-ORDER-STATUS = "P"` ที่ผู้อ่านต้องไปค้นหาว่า "P" หมายถึงอะไร
- `WS-ORDER-STATUS` มีค่า "S" ตรงกับ `ORDER-IS-SHIPPED` พอดี จึงแสดงข้อความ "YOUR ORDER IS
  ON THE WAY."
- โค้ดทั้งบล็อกนี้อ่านได้ใกล้เคียงประโยคภาษาอังกฤษปกติมาก แม้จะเป็นเงื่อนไขทางธุรกิจที่ซับซ้อนก็ตาม

### ผลลัพธ์ที่ได้จากการรันจริง

```
YOUR ORDER IS ON THE WAY.
```

### ข้อดีระยะยาวของแนวทางนี้

หากในอนาคตต้องเปลี่ยนรหัสสถานะ (เช่น เปลี่ยนจาก "S" เป็น "SH") เราแก้ไขแค่จุดเดียวที่ประกาศ
88-level เท่านั้น โดยไม่ต้องแก้ `EVALUATE` เลยแม้แต่บรรทัดเดียว — นี่คือหลักการ "แก้ไขจุดเดียว
ส่งผลทั่วโปรแกรม" (single point of change) ที่สำคัญมากในการดูแลรักษาโค้ดขนาดใหญ่ในระยะยาว

### ข้อควรระวัง

- Condition Names ที่ใช้ใน `WHEN` ของ `EVALUATE TRUE` ต้องเป็น 88-level ที่ประกาศไว้แล้วใน
  DATA DIVISION เท่านั้น และควรตั้งชื่อให้สื่อความหมายชัดเจนตามหลักการที่เรียนใน Part 010 ขั้นตอนที่ 96
- หาก 88-level หลายตัวมีช่วงค่าทับซ้อนกัน (เช่น ประกาศผิดพลาดจนสองเงื่อนไขเป็นจริงพร้อมกันได้)
  `EVALUATE TRUE` จะเลือกใช้ `WHEN` แรกที่พบเสมอ เช่นเดียวกับกรณี `THRU` ที่ทับซ้อนกันในขั้นตอนที่ 104

### แบบฝึกหัดที่ 107.1

**โจทย์**: จงเพิ่ม 88-level ใหม่ชื่อ `ORDER-IS-RETURNED VALUE "X"` เข้าไปในตัวแปร
`WS-ORDER-STATUS` และเพิ่ม `WHEN` ที่สอดคล้องกันใน EVALUATE ที่แสดงข้อความ "YOUR ORDER WAS
RETURNED."

**เฉลย**:
```cobol
       01  WS-ORDER-STATUS        PIC X(1)  VALUE "S".
           88  ORDER-IS-PENDING           VALUE "P".
           88  ORDER-IS-SHIPPED           VALUE "S".
           88  ORDER-IS-DELIVERED         VALUE "D".
           88  ORDER-IS-CANCELLED         VALUE "C".
           88  ORDER-IS-RETURNED          VALUE "X".
       ...
           EVALUATE TRUE
               WHEN ORDER-IS-PENDING
                   DISPLAY "YOUR ORDER IS BEING PREPARED."
               WHEN ORDER-IS-SHIPPED
                   DISPLAY "YOUR ORDER IS ON THE WAY."
               WHEN ORDER-IS-DELIVERED
                   DISPLAY "YOUR ORDER HAS ARRIVED."
               WHEN ORDER-IS-CANCELLED
                   DISPLAY "YOUR ORDER WAS CANCELLED."
               WHEN ORDER-IS-RETURNED
                   DISPLAY "YOUR ORDER WAS RETURNED."
               WHEN OTHER
                   DISPLAY "UNKNOWN ORDER STATUS."
           END-EVALUATE.
```

---

## ขั้นตอนที่ 108: EVALUATE ภายในลูป PERFORM

### แนวคิด

`EVALUATE` มักถูกใช้ร่วมกับลูปในโปรแกรมจริงเสมอ เพื่อประมวลผลเงื่อนไขที่แตกต่างกันสำหรับแต่ละรอบ
ของการวนซ้ำ ขั้นตอนนี้จะแอบยืมคำสั่ง `PERFORM VARYING` (ซึ่งจะสอนอย่างละเอียดใน Part 012-013)
มาสาธิตให้เห็นภาพการทำงานร่วมกัน ผ่านโจทย์คลาสสิกที่รู้จักกันดีในวงการเขียนโปรแกรมคือ **FizzBuzz**

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-IN-LOOP-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNTER             PIC 9(2)  VALUE 1.
       01  WS-FIZZBUZZ-RESULT     PIC X(12).

       PROCEDURE DIVISION.
           PERFORM VARYING WS-COUNTER FROM 1 BY 1
                   UNTIL WS-COUNTER > 9
               EVALUATE TRUE
                   WHEN FUNCTION MOD(WS-COUNTER, 3) = 0
                        AND FUNCTION MOD(WS-COUNTER, 5) = 0
                       MOVE "FIZZBUZZ" TO WS-FIZZBUZZ-RESULT
                   WHEN FUNCTION MOD(WS-COUNTER, 3) = 0
                       MOVE "FIZZ" TO WS-FIZZBUZZ-RESULT
                   WHEN FUNCTION MOD(WS-COUNTER, 5) = 0
                       MOVE "BUZZ" TO WS-FIZZBUZZ-RESULT
                   WHEN OTHER
                       MOVE WS-COUNTER TO WS-FIZZBUZZ-RESULT
               END-EVALUATE
               DISPLAY WS-COUNTER ": " WS-FIZZBUZZ-RESULT
           END-PERFORM.
           STOP RUN.
```

### อธิบายโค้ด

- `PERFORM VARYING WS-COUNTER FROM 1 BY 1 UNTIL WS-COUNTER > 9` — วนลูปให้ `WS-COUNTER`
  มีค่าตั้งแต่ 1 ถึง 9 (จะเรียนไวยากรณ์นี้อย่างละเอียดใน Part 013) ในที่นี้ใช้เพียงเพื่อสาธิตการทำงาน
  ร่วมกับ `EVALUATE` เท่านั้น
- `FUNCTION MOD(WS-COUNTER, 3) = 0` — ใช้ intrinsic function `MOD` หาเศษที่เหลือจากการหาร
  (จะสอนละเอียดใน Part 036) เพื่อตรวจสอบว่าหารด้วย 3 ลงตัวหรือไม่
- `EVALUATE TRUE` ภายในลูปนี้ตรวจสอบทีละรอบว่าตัวเลขปัจจุบันหารด้วยทั้ง 3 และ 5 ลงตัวหรือไม่ก่อน
  (ต้องตรวจสอบเงื่อนไขที่เฉพาะเจาะจงกว่าก่อนเสมอ) จากนั้นจึงค่อยตรวจแยกแค่ 3 หรือแค่ 5

### ผลลัพธ์ที่ได้จากการรันจริง

```
01: 01          
02: 02          
03: FIZZ        
04: 04          
05: BUZZ        
06: FIZZ        
07: 07          
08: 08          
09: FIZZ        
```

ตัวเลขที่หารด้วย 3 ลงตัว (3, 6, 9) แสดง "FIZZ" ตัวเลขที่หารด้วย 5 ลงตัวแสดง "BUZZ" และถ้าลูปนี้
ยาวถึง 15 จะเห็น "FIZZBUZZ" ปรากฏด้วย (15 หารด้วยทั้ง 3 และ 5 ลงตัว)

### ข้อควรระวัง

- **ลำดับของ `WHEN` สำคัญมากในกรณีนี้**: ต้องตรวจสอบเงื่อนไข "หารด้วยทั้ง 3 และ 5" (FIZZBUZZ)
  **ก่อน** เงื่อนไข "หารด้วย 3 อย่างเดียว" (FIZZ) เสมอ เพราะถ้าสลับลำดับ เลข 15 จะเข้าเงื่อนไข FIZZ
  ก่อนโดยไม่มีโอกาสตรวจสอบเงื่อนไข FIZZBUZZ ที่ควรจะเป็นคำตอบที่ถูกต้องกว่าเลย — หลักการนี้เหมือนกับ
  การเรียงลำดับเงื่อนไขใน `EVALUATE TRUE` ทั่วไปที่เรียนในขั้นตอนที่ 102
- `EVALUATE` ที่อยู่ภายในลูปจะถูกประเมินซ้ำใหม่ทุกรอบของการวนซ้ำ ทำให้ผลลัพธ์เปลี่ยนไปตามค่าตัวแปร
  ที่เปลี่ยนแปลงในแต่ละรอบได้อย่างถูกต้อง

### แบบฝึกหัดที่ 108.1

**โจทย์**: จงอธิบายว่าทำไมถ้าสลับลำดับ `WHEN` ในตัวอย่างข้างต้น โดยเอา
`WHEN FUNCTION MOD(WS-COUNTER, 3) = 0` (ตรวจแค่ 3) ไว้ก่อน `WHEN FUNCTION MOD(WS-COUNTER, 3)
= 0 AND FUNCTION MOD(WS-COUNTER, 5) = 0` (ตรวจทั้งคู่) เลข 15 จะได้ผลลัพธ์ผิดพลาดอย่างไร

**เฉลยแนวทาง**: เนื่องจาก `EVALUATE` ตรวจสอบ `WHEN` ตามลำดับบนลงล่างและหยุดที่ตัวแรกที่เป็นจริง
ถ้าเอาเงื่อนไข "หารด้วย 3 อย่างเดียว" ไว้ก่อน เลข 15 (ซึ่งหารด้วย 3 ลงตัวด้วย) จะจับคู่กับเงื่อนไขนั้น
ทันทีและได้ผลลัพธ์ "FIZZ" ทั้งที่ควรจะได้ "FIZZBUZZ" เพราะ 15 หารด้วย 5 ลงตัวเช่นกัน โปรแกรมจะไม่มี
โอกาสตรวจสอบเงื่อนไข FIZZBUZZ ที่ควรจะถูกต้องกว่าเลย นี่คือตัวอย่างที่ชัดเจนว่าทำไมต้องเรียงเงื่อนไข
ที่เฉพาะเจาะจงกว่า (specific) ไว้ก่อนเงื่อนไขที่กว้างกว่า (general) เสมอใน `EVALUATE TRUE`

---

## ขั้นตอนที่ 109: EVALUATE เทียบกับ Nested IF — เมื่อไหร่ควรใช้อันไหน

### แนวคิด

ขั้นตอนนี้เปรียบเทียบโค้ดตรรกะเดียวกันที่เขียนสองแบบเคียงข้างกันโดยตรง เพื่อให้เห็นความแตกต่างของ
ความอ่านง่ายอย่างเป็นรูปธรรม ก่อนสรุปหลักเกณฑ์ว่าเมื่อไหร่ควรเลือกใช้แบบไหน

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-VS-IF-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-SHIPPING-ZONE       PIC 9(1)  VALUE 3.
       01  WS-SHIPPING-COST       PIC 9(4)  VALUE 0.

       PROCEDURE DIVISION.
      *> The same logic written with nested IF (Part 010 style):
           IF WS-SHIPPING-ZONE = 1
               MOVE 50 TO WS-SHIPPING-COST
           ELSE
               IF WS-SHIPPING-ZONE = 2
                   MOVE 80 TO WS-SHIPPING-COST
               ELSE
                   IF WS-SHIPPING-ZONE = 3
                       MOVE 120 TO WS-SHIPPING-COST
                   ELSE
                       MOVE 200 TO WS-SHIPPING-COST
                   END-IF
               END-IF
           END-IF.
           DISPLAY "NESTED IF RESULT     : " WS-SHIPPING-COST.

      *> The exact same logic written with EVALUATE -- flatter,
      *> no cumulative END-IF counting required at all.
           EVALUATE WS-SHIPPING-ZONE
               WHEN 1
                   MOVE 50 TO WS-SHIPPING-COST
               WHEN 2
                   MOVE 80 TO WS-SHIPPING-COST
               WHEN 3
                   MOVE 120 TO WS-SHIPPING-COST
               WHEN OTHER
                   MOVE 200 TO WS-SHIPPING-COST
           END-EVALUATE.
           DISPLAY "EVALUATE RESULT      : " WS-SHIPPING-COST.
           STOP RUN.
```

### อธิบายโค้ด

ทั้งสองบล็อกให้ผลลัพธ์เหมือนกันทุกประการเมื่อ `WS-SHIPPING-ZONE` เป็น 3 (ได้ค่าขนส่ง 120 ทั้งคู่)
แต่สังเกตความแตกต่างของโครงสร้าง:

- **Nested IF**: ต้องเยื้องบรรทัดลึกขึ้นเรื่อย ๆ ทุกครั้งที่เพิ่มเงื่อนไข และต้องปิดด้วย `END-IF`
  จำนวนเท่ากับ `IF` ที่เปิดไว้ (3 ชั้นในตัวอย่างนี้ ต้องมี `END-IF` 3 ตัว)
- **EVALUATE**: โครงสร้างแบนราบ (flat) ทุก `WHEN` อยู่ระดับการเยื้องเดียวกัน และปิดท้ายด้วย
  `END-EVALUATE` เพียงตัวเดียวไม่ว่าจะมีกี่ `WHEN` ก็ตาม

### ผลลัพธ์ที่ได้จากการรันจริง

```
NESTED IF RESULT     : 0120
EVALUATE RESULT      : 0120
```

### หลักเกณฑ์การเลือกใช้

| สถานการณ์ | แนะนำให้ใช้ |
|---|---|
| มีแค่ 1-2 เงื่อนไข ไม่ซับซ้อน | `IF-ELSE` ธรรมดา |
| มีตั้งแต่ 3 เงื่อนไขขึ้นไปที่เทียบค่าเดียวกัน หรือช่วงค่า | `EVALUATE` |
| ต้องเทียบหลายตัวแปรพร้อมกันเป็นตาราง | `EVALUATE ... ALSO` |
| เงื่อนไขแต่ละอันไม่เกี่ยวข้องกัน ตรวจสอบแยกอิสระ (ไม่ใช่ mutually exclusive) | `IF` แยกหลายตัว
  (ไม่ใช่ `EVALUATE` เดียว เพราะ `EVALUATE` ตรวจแค่กรณีแรกที่ตรงแล้วหยุด) |

### ข้อควรระวัง

- **`EVALUATE` ไม่ได้ดีกว่า `IF` เสมอไปในทุกกรณี** สำหรับเงื่อนไขง่าย ๆ เพียง 1-2 อัน การใช้ `IF`
  ธรรมดายังคงเป็นทางเลือกที่อ่านง่ายและตรงไปตรงมากว่า การใช้ `EVALUATE` กับเงื่อนไขเดียวจะดูซับซ้อน
  เกินความจำเป็น
- ทีมพัฒนาจริงหลายทีมมีมาตรฐานโค้ด (coding standard) ของตัวเองว่าจะใช้ `EVALUATE` แทน Nested IF
  เมื่อมีความลึกเกินกี่ชั้น ควรตรวจสอบและปฏิบัติตามมาตรฐานของทีมที่ทำงานด้วยเสมอ

### แบบฝึกหัดที่ 109.1

**โจทย์**: จงอธิบายว่าทำไม Nested IF ที่มี 3 ชั้น (ต้องใช้ `END-IF` 3 ตัว) จึงมีความเสี่ยงต่อบั๊กมากกว่า
`EVALUATE` ที่มี `WHEN` 4 อันแต่ใช้ `END-EVALUATE` เพียงตัวเดียว

**เฉลยแนวทาง**: เพราะใน Nested IF ผู้เขียนต้องนับจำนวน `END-IF` ให้ตรงกับจำนวน `IF` ที่เปิดไว้อย่าง
แม่นยำ และต้องวางตำแหน่งแต่ละ `END-IF` ให้ตรงกับ `IF` ของมันอย่างถูกต้อง (โดยเฉพาะเมื่อมี `ELSE`
ปนอยู่หลายชั้น) ยิ่งซ้อนลึกเท่าไร ยิ่งเสี่ยงต่อการนับผิดหรือวางตำแหน่งผิดมากเท่านั้น ซึ่งอาจทำให้เกิด
"dangling else" หรือโครงสร้างเงื่อนไขที่ผิดเพี้ยนไปจากที่ตั้งใจ (ปัญหาเดียวกับที่กล่าวถึงใน Part 010
ขั้นตอนที่ 95) ในขณะที่ `EVALUATE` ต้องการแค่ `END-EVALUATE` เดียวปิดท้ายไม่ว่าจะมี `WHEN` กี่อัน
ทำให้ความเสี่ยงจากการนับ scope terminator ผิดพลาดลดลงอย่างมาก

---

## ขั้นตอนที่ 110: โปรแกรมรวบยอด — ระบบสั่งอาหารด้วยเมนู (Restaurant Menu System)

### แนวคิด

ปิดท้าย Part นี้ด้วยโปรแกรมที่ผสาน `EVALUATE` เข้ากับ Condition Names และการคำนวณราคารวม
เพื่อจำลองระบบสั่งอาหารแบบเมนูตัวเลขที่พบได้ทั่วไปในระบบ POS (Point of Sale) เบื้องต้น

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. RESTAURANT-MENU-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-MENU-CHOICE         PIC 9(1)  VALUE 2.
       01  WS-ITEM-NAME           PIC X(20).
       01  WS-ITEM-PRICE          PIC 9(4)V99.
       01  WS-QUANTITY            PIC 9(2)  VALUE 3.
       01  WS-LINE-TOTAL          PIC 9(6)V99.
       01  WS-VALID-CHOICE-FLAG   PIC X(1)  VALUE "Y".
           88  CHOICE-IS-VALID            VALUE "Y".

       PROCEDURE DIVISION.
           DISPLAY "===== RESTAURANT MENU =====".
           DISPLAY "1. FRIED RICE       - 60.00".
           DISPLAY "2. PAD THAI         - 70.00".
           DISPLAY "3. TOM YUM SOUP     - 90.00".
           DISPLAY "4. GREEN CURRY      - 85.00".
           DISPLAY "============================".

           EVALUATE WS-MENU-CHOICE
               WHEN 1
                   MOVE "FRIED RICE"    TO WS-ITEM-NAME
                   MOVE 60.00           TO WS-ITEM-PRICE
               WHEN 2
                   MOVE "PAD THAI"      TO WS-ITEM-NAME
                   MOVE 70.00           TO WS-ITEM-PRICE
               WHEN 3
                   MOVE "TOM YUM SOUP"  TO WS-ITEM-NAME
                   MOVE 90.00           TO WS-ITEM-PRICE
               WHEN 4
                   MOVE "GREEN CURRY"   TO WS-ITEM-NAME
                   MOVE 85.00           TO WS-ITEM-PRICE
               WHEN OTHER
                   MOVE "N"             TO WS-VALID-CHOICE-FLAG
           END-EVALUATE.

           IF CHOICE-IS-VALID
               COMPUTE WS-LINE-TOTAL = WS-ITEM-PRICE * WS-QUANTITY
               DISPLAY "YOU ORDERED   : " WS-ITEM-NAME
               DISPLAY "QUANTITY      : " WS-QUANTITY
               DISPLAY "UNIT PRICE    : " WS-ITEM-PRICE
               DISPLAY "TOTAL PRICE   : " WS-LINE-TOTAL
           ELSE
               DISPLAY "INVALID MENU CHOICE. PLEASE TRY AGAIN."
           END-IF.
           STOP RUN.
```

### อธิบายโค้ด

- โปรแกรมแสดงเมนูอาหาร 4 รายการก่อน แล้วใช้ `EVALUATE WS-MENU-CHOICE` เพื่อแปลงหมายเลขที่
  ลูกค้าเลือก (จำลองด้วย `VALUE 2`) ให้เป็นชื่อรายการอาหารและราคา
- แต่ละ `WHEN` มีคำสั่งมากกว่าหนึ่งบรรทัด (ทั้ง `MOVE` ชื่อและ `MOVE` ราคา) ซึ่งเป็นไปได้ตามปกติ
  เหมือนกับที่ `IF` รองรับหลายคำสั่งภายในหนึ่งเงื่อนไข
- `WHEN OTHER` กำหนดค่า `WS-VALID-CHOICE-FLAG` เป็น "N" เพื่อบอกว่าตัวเลือกไม่ถูกต้อง แทนที่จะ
  พยายามคำนวณราคาจากข้อมูลที่ไม่มีอยู่จริง — เป็นการป้องกันข้อผิดพลาดที่ดีตามหลักการจาก
  ขั้นตอนที่ 106
- `IF CHOICE-IS-VALID` — ใช้ Condition Name ตรวจสอบก่อนคำนวณและแสดงผลสรุปคำสั่งซื้อ

### ผลลัพธ์ที่ได้จากการรันจริง

```
===== RESTAURANT MENU =====
1. FRIED RICE       - 60.00
2. PAD THAI         - 70.00
3. TOM YUM SOUP     - 90.00
4. GREEN CURRY      - 85.00
============================
YOU ORDERED   : PAD THAI            
QUANTITY      : 03
UNIT PRICE    : 0070.00
TOTAL PRICE   : 000210.00
```

ลูกค้าเลือกเมนู 2 (PAD THAI ราคา 70.00 บาท) จำนวน 3 จาน รวมเป็น 70.00 × 3 = 210.00 บาท
ตรงกับผลลัพธ์ `TOTAL PRICE : 000210.00` พอดี

### ข้อควรระวัง

- โปรแกรมนี้กำหนด `WS-MENU-CHOICE` และ `WS-QUANTITY` ด้วย `VALUE` ตายตัวเพื่อความง่ายในการ
  สาธิต ในระบบจริงควรใช้ `ACCEPT` (ตามที่เรียนใน Part 007) รับค่าจากผู้ใช้แทน พร้อมตรวจสอบด้วย
  `IS NUMERIC` ก่อนนำไปใช้ใน `EVALUATE`
- สังเกตว่าการรวมคำสั่ง `MOVE` สองคำสั่งไว้ในหนึ่ง `WHEN` ทำได้อย่างอิสระ ไม่มีข้อจำกัดเรื่องจำนวน
  คำสั่งต่อหนึ่ง `WHEN` เหมือนกับ `IF` ทุกประการ

### แบบฝึกหัดที่ 110.1

**โจทย์**: จงต่อยอดโปรแกรมเมนูร้านอาหารข้างต้น โดยเพิ่มเมนูที่ 5 "MANGO STICKY RICE" ราคา 55.00
บาท เข้าไปในทั้งส่วนแสดงเมนูและ `EVALUATE`

**เฉลยแนวทาง**:
```cobol
           DISPLAY "5. MANGO STICKY RICE - 55.00".
           ...
           EVALUATE WS-MENU-CHOICE
               WHEN 1
                   MOVE "FRIED RICE"    TO WS-ITEM-NAME
                   MOVE 60.00           TO WS-ITEM-PRICE
               WHEN 2
                   MOVE "PAD THAI"      TO WS-ITEM-NAME
                   MOVE 70.00           TO WS-ITEM-PRICE
               WHEN 3
                   MOVE "TOM YUM SOUP"  TO WS-ITEM-NAME
                   MOVE 90.00           TO WS-ITEM-PRICE
               WHEN 4
                   MOVE "GREEN CURRY"   TO WS-ITEM-NAME
                   MOVE 85.00           TO WS-ITEM-PRICE
               WHEN 5
                   MOVE "MANGO STICKY RICE" TO WS-ITEM-NAME
                   MOVE 55.00               TO WS-ITEM-PRICE
               WHEN OTHER
                   MOVE "N"             TO WS-VALID-CHOICE-FLAG
           END-EVALUATE.
```
อย่าลืมว่า `WS-MENU-CHOICE PIC 9(1)` รองรับตัวเลข 0-9 อยู่แล้ว จึงไม่ต้องแก้ขนาดฟิลด์นี้เพิ่มเติม

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้ `EVALUATE` Statement ซึ่งเป็นคำสั่งเทียบเท่า switch/case ที่ทรงพลังที่สุด
ตัวหนึ่งของ COBOL อย่างครบถ้วน:

- `EVALUATE` พื้นฐาน — เทียบค่าเดียวกับหลายค่าคงที่ ไม่มี fall-through เหมือน switch ในภาษา C
- `EVALUATE TRUE` — เทียบเท่า IF-ELSE IF ต่อเนื่องกัน เป็นรูปแบบที่ใช้บ่อยที่สุดในโค้ดจริง
- การจับคู่หลายค่าใน `WHEN` เดียวกันด้วยการเขียน `WHEN` ติดกันหลายตัว (เป็น OR ไม่ใช่ fall-through)
- ช่วงค่าด้วย `WHEN ... THRU` สำหรับจำแนกช่วงตัวเลขอย่างกระชับ
- `EVALUATE ... ALSO` สำหรับเทียบหลายตัวแปรพร้อมกันแบบตารางการตัดสินใจ
- `WHEN OTHER` และความสำคัญของการใส่ไว้เสมอเพื่อป้องกันกรณีที่ไม่มีเงื่อนไขใดตรงกันเลย
- การผสาน `EVALUATE TRUE` กับ Condition Names (88-level) ให้โค้ดอ่านเป็นภาษาอังกฤษได้เกือบ
  สมบูรณ์แบบ
- การใช้ `EVALUATE` ร่วมกับลูป `PERFORM VARYING` ผ่านตัวอย่าง FizzBuzz
- การเปรียบเทียบ `EVALUATE` กับ Nested IF โดยตรง พร้อมหลักเกณฑ์การเลือกใช้ที่เหมาะสม
- โปรแกรมรวบยอดระบบสั่งอาหารด้วยเมนูที่ผสาน EVALUATE, Condition Names และการคำนวณเข้าด้วยกัน

`EVALUATE` คือเครื่องมือสำคัญที่ทำให้โค้ด COBOL ที่มีเงื่อนไขซับซ้อนหลายทางเลือกยังคงอ่านง่ายและ
ดูแลรักษาได้ ใน Part ถัดไปเราจะกลับไปสู่แนวคิดพื้นฐานสำคัญอีกอันของการเขียนโปรแกรมเชิงกระบวนการ
(Procedural Programming) ที่กล่าวถึงใน Part 001 นั่นคือ **Iteration (การวนซ้ำ)** ผ่านคำสั่ง
**`PERFORM`** พื้นฐานและ **`PERFORM UNTIL`** ซึ่งเป็นกลไกการทำลูปหลักของ COBOL

**[← กลับไป Part 010](part-010-if-else-condition-names.md)** | **[ไปยัง Part 012: PERFORM UNTIL →](part-012-perform-until.md)**
