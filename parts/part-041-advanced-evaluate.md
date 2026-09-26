# Part 041: Advanced EVALUATE และเงื่อนไขซับซ้อน (ขั้นตอนที่ 401–410)

## คำนำของ Part นี้

ใน Part 011 เราเรียนรู้ `EVALUATE` ตั้งแต่พื้นฐาน (เทียบค่าเดียว), `EVALUATE TRUE`, การจับคู่หลายค่า,
`THRU`, `EVALUATE ... ALSO` เบื้องต้น (สองตัวแปร), และ `WHEN OTHER` — เพียงพอสำหรับเงื่อนไขทาง
ธุรกิจส่วนใหญ่ที่พบทั่วไป และเราเพิ่งเห็นตัวอย่างการใช้งานจริงของมันในเมนูโปรแกรม `SCREEN SECTION`
ของ Part 040

Part นี้จะพา `EVALUATE` ไปอีกขั้น สำหรับสถานการณ์ทางธุรกิจที่ซับซ้อนขึ้นจริง ๆ: การเทียบ**สาม
subject ขึ้นไปพร้อมกัน**, การเขียน**เงื่อนไขผสม (compound condition) ภายใน `WHEN` เดียว**,
**`EVALUATE` ที่ซ้อนกันหลายชั้น**, การใช้**นิพจน์ทางคณิตศาสตร์และ intrinsic function** เป็น
subject/object, และการเปรียบเทียบกับแนวทาง**ตารางค้นหา (table-driven)** ที่เรียนจาก `SEARCH`
ใน Part 017 ปิดท้ายด้วยโปรแกรมเครื่องมือคำนวณราคา (pricing engine) ที่ผสานทุกเทคนิคเข้าด้วยกัน
ทุกตัวอย่างทดสอบคอมไพล์และรันจริงด้วย GnuCOBOL 4.0 (early-dev) แล้วทั้งหมด รวมถึงกรณี compile
error จริงที่พบระหว่างทดสอบ ซึ่งกลายเป็นบทเรียนสำคัญเรื่องกฎเหล็กคอลัมน์ 72 ที่ย้ำมาตลอดหลักสูตรนี้

---

## ขั้นตอนที่ 401: EVALUATE ... ALSO กับสาม Subject ขึ้นไป

### แนวคิด

Part 011 แสดงให้เห็น `EVALUATE ... ALSO` กับสอง subject แล้ว แนวคิดนี้ขยายไปกี่ subject ก็ได้
เพียงแค่เพิ่ม `ALSO` ต่อกันไปเรื่อย ๆ ทั้งใน `EVALUATE` เองและในทุก `WHEN` — ยิ่งมี subject มาก
เท่าไร ยิ่งสร้าง "ตารางการตัดสินใจ" (decision table) ที่ซับซ้อนขึ้นได้โดยไม่ต้องซ้อน `IF` หลายสิบชั้น

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-TRIPLE-ALSO-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-TYPE   PIC X(1) VALUE "V".
       01  WS-DAY-TYPE        PIC X(1) VALUE "H".
       01  WS-ORDER-AMOUNT    PIC 9(6)V99 VALUE 2000.00.
       01  WS-DISCOUNT        PIC 9(2) VALUE 0.

       PROCEDURE DIVISION.
      *> Three independent subjects at once: customer type,
      *> day type, and a boolean condition on the order amount.
           EVALUATE WS-CUSTOMER-TYPE ALSO WS-DAY-TYPE ALSO TRUE
               WHEN "V" ALSO "H" ALSO WS-ORDER-AMOUNT >= 1000.00
                   MOVE 25 TO WS-DISCOUNT
               WHEN "V" ALSO "N" ALSO WS-ORDER-AMOUNT >= 1000.00
                   MOVE 20 TO WS-DISCOUNT
               WHEN "R" ALSO "H" ALSO WS-ORDER-AMOUNT >= 1000.00
                   MOVE 15 TO WS-DISCOUNT
               WHEN OTHER
                   MOVE 5 TO WS-DISCOUNT
           END-EVALUATE.
           DISPLAY "DISCOUNT: " WS-DISCOUNT "%".
           STOP RUN.
```

### อธิบายโค้ด

- `EVALUATE WS-CUSTOMER-TYPE ALSO WS-DAY-TYPE ALSO TRUE` — ประกาศ subject 3 ตัว: ประเภทลูกค้า,
  ประเภทวัน (วันหยุด/วันธรรมดา), และ `TRUE` (เพื่อให้ตำแหน่งที่ 3 ในแต่ละ `WHEN` เป็นเงื่อนไขเต็มรูป
  แบบเดียวกับ `EVALUATE TRUE` ที่เรียนใน Part 011 ขั้นตอนที่ 102)
- แต่ละ `WHEN` ต้องมี object **ครบ 3 ตัวเสมอ** คั่นด้วย `ALSO` ตรงตามจำนวน subject
- ลูกค้าประเภท "V" (VIP), วันหยุด "H", ยอดสั่งซื้อ 2000.00 (>= 1000.00) ตรงกับ `WHEN` แรกพอดี
  จึงได้ส่วนลด 25%

### ผลลัพธ์ที่ได้จากการรันจริง

```
DISCOUNT: 25%
```

### ข้อควรระวัง

- **จำนวน object ในทุก `WHEN` ต้องเท่ากับจำนวน subject เป๊ะ** (ในที่นี้คือ 3 ตัว) หากขาดหรือเกิน
  จะ compile error ทันที (จะสาธิตข้อผิดพลาดจริงในขั้นตอนที่ 409)
- ยิ่งมี subject หลายตัว จำนวน `WHEN` ที่ต้องเขียนครบทุกกรณีจะเพิ่มขึ้นแบบทวีคูณ (จำนวนกรณีของ
  subject แต่ละตัวคูณกัน) ควรพิจารณาว่าคุ้มค่าจริงหรือไม่ก่อนเพิ่ม subject ตัวที่ 4, 5 ขึ้นไป —
  หากมีปัจจัยมากเกินไป อาจถึงเวลาพิจารณาแนวทางตารางค้นหาแทน (จะเปรียบเทียบในขั้นตอนที่ 408)

### แบบฝึกหัดที่ 401.1

**โจทย์**: จงอธิบายว่าทำไม `WHEN "V" ALSO "N" ALSO WS-ORDER-AMOUNT >= 1000.00` จึงจำเป็นต้อง
เขียนแยกจาก `WHEN "V" ALSO "H" ALSO WS-ORDER-AMOUNT >= 1000.00` ทั้งที่ทั้งคู่เป็นลูกค้า VIP
เหมือนกัน

**เฉลยแนวทาง**: เพราะ `EVALUATE ... ALSO` ต้องการให้**ทุก object ใน WHEN ตรงกับ subject ของ
มันพร้อมกันทั้งหมด** จึงจะถือว่า WHEN นั้นเป็นจริง หาก subject ตัวที่สอง (ประเภทวัน) มีค่าต่างกัน
("H" กับ "N") ก็ถือเป็นคนละกรณีทางตรรกะทันที แม้ subject ตัวอื่นจะเหมือนกันก็ตาม จึงต้องเขียน
`WHEN` แยกกันสำหรับแต่ละชุดค่าผสมที่ให้ผลลัพธ์ต่างกัน — นี่คือธรรมชาติของตารางการตัดสินใจ
(decision table) ที่ต้องระบุทุกชุดค่าผสมอย่างชัดเจน

---

## ขั้นตอนที่ 402: เงื่อนไขผสม (Compound Condition) ภายใน WHEN เดียว

### แนวคิด

แต่ละ object ใน `WHEN` ของ `EVALUATE TRUE` (หรือตำแหน่งที่ใช้ `TRUE` เป็น subject) ไม่จำเป็นต้อง
เป็นเงื่อนไขเดี่ยว ๆ เท่านั้น แต่สามารถเป็น**เงื่อนไขผสม**ที่ใช้ `AND`/`OR`/`NOT` และวงเล็บได้เต็ม
รูปแบบเหมือนใน `IF` (Part 010 ขั้นตอนที่ 94) ทำให้ `WHEN` แต่ละอันสามารถแทนกฎทางธุรกิจที่ซับซ้อน
ได้ในบรรทัดเดียว

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-COMPOUND-WHEN-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-AGE        PIC 9(3) VALUE 25.
       01  WS-INCOME     PIC 9(7)V99 VALUE 45000.00.
       01  WS-RESULT     PIC X(20).

       PROCEDURE DIVISION.
      *> Each WHEN can hold a full compound condition, just like
      *> an IF statement -- not just a single comparison.
           EVALUATE TRUE
               WHEN (WS-AGE >= 20 AND WS-AGE <= 35)
                    AND WS-INCOME >= 30000.00
                   MOVE "TARGET SEGMENT A" TO WS-RESULT
               WHEN WS-AGE < 20 OR WS-AGE > 60
                   MOVE "NOT ELIGIBLE" TO WS-RESULT
               WHEN OTHER
                   MOVE "TARGET SEGMENT B" TO WS-RESULT
           END-EVALUATE.
           DISPLAY WS-RESULT.
           STOP RUN.
```

### อธิบายโค้ด

- `WHEN (WS-AGE >= 20 AND WS-AGE <= 35) AND WS-INCOME >= 30000.00` — เงื่อนไขผสมที่ต้องเป็น
  จริงทั้งสามส่วนพร้อมกัน: อายุอยู่ในช่วง 20-35 **และ** รายได้ตั้งแต่ 30000 ขึ้นไป
- วงเล็บรอบ `(WS-AGE >= 20 AND WS-AGE <= 35)` ช่วยให้อ่านง่ายขึ้นว่าเป็นกลุ่มเงื่อนไขเดียวกัน แม้จะ
  ไม่จำเป็นทางไวยากรณ์ (เพราะ `AND` มีความสำคัญเท่ากันทั้งหมดในที่นี้) แต่ช่วยลดความสับสนเมื่ออ่านโค้ด
- ด้วย `WS-AGE` = 25 (อยู่ในช่วง 20-35) และ `WS-INCOME` = 45000.00 (>= 30000.00) เงื่อนไขแรก
  เป็นจริง จึงได้ผลลัพธ์ "TARGET SEGMENT A"

### ผลลัพธ์ที่ได้จากการรันจริง

```
TARGET SEGMENT A
```

### ข้อควรระวัง

- เงื่อนไขผสมที่ยาวเกินไปใน `WHEN` เดียวอาจทำให้อ่านยาก ควรพิจารณาใช้ 88-level (Part 010) เพื่อ
  ตั้งชื่อเงื่อนไขที่มีความหมายแทนการเขียนเงื่อนไขดิบยาว ๆ ซ้ำ ๆ (เช่น ตั้ง `88 IS-WORKING-AGE
  VALUE 20 THRU 35` แล้วเขียน `WHEN IS-WORKING-AGE AND WS-INCOME >= 30000.00` แทน)
- กฎการใส่วงเล็บและลำดับความสำคัญของ `AND`/`OR`/`NOT` เหมือนกับที่เรียนใน Part 010 ทุกประการ
  — ควรใส่วงเล็บชัดเจนเสมอเมื่อผสม `AND` กับ `OR` ในเงื่อนไขเดียวกัน

### แบบฝึกหัดที่ 402.1

**โจทย์**: จงเขียน `WHEN` ที่ตรวจสอบว่า "ลูกค้าเป็นสมาชิก VIP หรือมียอดสั่งซื้อเกิน 5000 บาท
**และ** ไม่มีประวัติค้างชำระ" (ใช้ตัวแปรสมมติ `WS-IS-VIP PIC X`, `WS-ORDER-AMT PIC 9(6)V99`,
`WS-HAS-OVERDUE PIC X`)

**เฉลย**:
```cobol
WHEN (WS-IS-VIP = "Y" OR WS-ORDER-AMT > 5000.00)
     AND WS-HAS-OVERDUE = "N"
    DISPLAY "ELIGIBLE FOR EXPRESS SHIPPING"
```

---

## ขั้นตอนที่ 403: EVALUATE ซ้อนกัน (Nested EVALUATE)

### แนวคิด

เช่นเดียวกับ `IF` ที่ซ้อนกันได้ (Nested IF จาก Part 010) `EVALUATE` ก็สามารถซ้อนอยู่ภายใน
`WHEN` ของ `EVALUATE` อีกตัวได้เช่นกัน มีประโยชน์เมื่อเงื่อนไขระดับแรกแยกกลุ่มใหญ่ก่อน แล้วแต่ละกลุ่ม
มีเงื่อนไขย่อยที่ต่างกันโดยสิ้นเชิง (ทำให้ `EVALUATE ... ALSO` ตัวเดียวไม่คุ้มค่าเพราะกลุ่มค่าผสมที่เป็น
ไปได้จริงมีน้อยกว่าที่ตารางแบบ ALSO จะบังคับให้เขียน)

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-NESTED-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ORDER-TYPE   PIC X(1) VALUE "D".
       01  WS-COUNTRY      PIC X(2) VALUE "TH".
       01  WS-SHIP-METHOD  PIC X(22).

       PROCEDURE DIVISION.
      *> Nested EVALUATE: an EVALUATE inside a WHEN of another
           EVALUATE WS-ORDER-TYPE
               WHEN "D"
                   EVALUATE WS-COUNTRY
                       WHEN "TH"
                           MOVE "DOMESTIC EXPRESS"
                               TO WS-SHIP-METHOD
                       WHEN OTHER
                           MOVE "INTL DIGITAL DELIVERY"
                               TO WS-SHIP-METHOD
                   END-EVALUATE
               WHEN "P"
                   EVALUATE WS-COUNTRY
                       WHEN "TH"
                           MOVE "DOMESTIC PARCEL"
                               TO WS-SHIP-METHOD
                       WHEN OTHER
                           MOVE "INTL COURIER"
                               TO WS-SHIP-METHOD
                   END-EVALUATE
               WHEN OTHER
                   MOVE "UNKNOWN ORDER TYPE" TO WS-SHIP-METHOD
           END-EVALUATE.
           DISPLAY WS-SHIP-METHOD.
           STOP RUN.
```

### อธิบายโค้ด

- `EVALUATE WS-ORDER-TYPE` ชั้นนอกแยกกลุ่มใหญ่ก่อน: "D" (Digital) กับ "P" (Physical)
- ภายในแต่ละ `WHEN` มี `EVALUATE WS-COUNTRY` ชั้นในแยกย่อยตามประเทศ — สังเกตว่าเงื่อนไขย่อย
  ("TH" กับประเทศอื่น) ไม่เกี่ยวข้องกันเลยระหว่างกลุ่ม "D" กับ "P" (ของ Digital ไม่มี "พัสดุ" และ
  ของ Physical ไม่มี "ดาวน์โหลด") การซ้อน `EVALUATE` จึงเหมาะกว่าการทำ `ALSO` ตารางเดียว ซึ่งจะ
  บังคับให้ต้องคิดคำตอบสำหรับทุกคู่ค่าผสมแม้จะไม่มีความหมายทางธุรกิจจริง
- ด้วย `WS-ORDER-TYPE` = "D" และ `WS-COUNTRY` = "TH" ผลลัพธ์คือ "DOMESTIC EXPRESS"

### กับดักจริงที่พบระหว่างทดสอบ — Nested EVALUATE กับกฎเหล็กคอลัมน์ 72

ระหว่างการทดสอบตัวอย่างนี้ ผู้เขียนเนื้อหาได้ลองเขียนโค้ดฉบับแรกโดยไม่ตัดบรรทัด `MOVE` ให้สั้นลง
เหมือนโค้ดด้านบน ผลคือเกิด **compile error จริง** ดังนี้:

```
ev3.cob:18: error: 'WS-SHIP-METHO' is not defined
```

เมื่อตรวจสอบพบว่าบรรทัดต้นเหตุคือ:
```
                       WHEN OTHER
                           MOVE "INTL DIGITAL DELIVERY" TO WS-SHIP-METHOD
```
บรรทัดนี้มีความยาว **73 ตัวอักษร** (เกิน column 72 ไป 1 ตัวอักษรพอดี!) เนื่องจากการซ้อน
`EVALUATE` 2 ชั้นทำให้การเยื้องบรรทัด (indentation) ลึกขึ้นเรื่อย ๆ เมื่อรวมกับชื่อตัวแปรที่ยาว
(`WS-SHIP-METHOD`) และข้อความยาว (`"INTL DIGITAL DELIVERY"`) ทำให้ตัวอักษรตัวสุดท้าย `D` ของ
`WS-SHIP-METHOD` หลุดเลย column 72 ไปพอดี คอมไพเลอร์จึงตัดเหลือแค่ `WS-SHIP-METHO` (ไม่มี D)
ซึ่งไม่ตรงกับชื่อตัวแปรที่ประกาศไว้จริง

**นี่คือตัวอย่างที่เป็นรูปธรรมที่สุดของ "กฎเหล็ก" ที่กล่าวถึงตั้งแต่ต้นหลักสูตร**: การซ้อนโครงสร้าง
ควบคุม (Nested IF, Nested EVALUATE) หลายชั้นทำให้การเยื้องบรรทัดลึกขึ้นเรื่อย ๆ และ**เพิ่มความเสี่ยง
ที่บรรทัดจะเกิน column 72 โดยไม่รู้ตัว** แม้จะเป็นภาษาอังกฤษล้วนก็ตาม (ไม่ใช่แค่ปัญหาภาษาไทยที่กล่าวถึง
ในกฎเหล็กหลักเท่านั้น) วิธีแก้ที่ใช้ในโค้ดด้านบนคือ**ตัดบรรทัด `MOVE` ให้ค่าปลายทางอยู่บรรทัดใหม่**
(ย่อหน้าให้พอดีกับ column 72)

### ผลลัพธ์ที่ได้จากการรันจริง

```
DOMESTIC EXPRESS
```

เมื่อเปลี่ยน `WS-COUNTRY` เป็น "US" ผลลัพธ์ที่ทดสอบได้คือ:

```
INTL DIGITAL DELIVERY
```

### ข้อควรระวัง

- **ยิ่งซ้อนโครงสร้างควบคุมลึกเท่าไร ยิ่งต้องระวังความยาวบรรทัดมากขึ้นเท่านั้น** ควรตรวจสอบความยาว
  บรรทัดเสมอเมื่อเขียนโค้ดที่ซ้อนกันมากกว่า 2 ชั้น (เช่น ใช้ editor ที่แสดงเส้น column 72 ให้เห็นชัดเจน
  ตามที่แนะนำใน Part 002)
- Nested `EVALUATE` ที่ซ้อนกันมากกว่า 2-3 ชั้นเริ่มอ่านยากพอ ๆ กับ Nested IF ควรพิจารณาแยกเป็น
  paragraph ย่อย (Part 014) เพื่อลดความลึกของการเยื้องบรรทัดลง

### แบบฝึกหัดที่ 403.1

**โจทย์**: จงอธิบายว่าทำไมข้อผิดพลาด `'WS-SHIP-METHO' is not defined` จึงดูสับสนสำหรับผู้เริ่มต้น
ทั้งที่ตัวแปรที่แท้จริงคือ `WS-SHIP-METHOD` (มี D ต่อท้าย)

**เฉลยแนวทาง**: เพราะข้อความ error ไม่ได้บอกตรง ๆ ว่า "บรรทัดยาวเกิน column 72" แต่รายงานว่า
พบการอ้างอิงตัวแปรชื่อ `WS-SHIP-METHO` ซึ่งไม่มีอยู่จริงในโปรแกรม (เพราะตัวคอมไพเลอร์เห็นเฉพาะ
ตัวอักษรที่อยู่ในคอลัมน์ 8-72 เท่านั้น ตัวอักษร `D` ตัวสุดท้ายที่อยู่เกิน column 72 ไปถูกตัดทิ้งไปเลย
ก่อนที่จะถึงขั้นตอนตรวจสอบชื่อตัวแปร) ทำให้ผู้เขียนโค้ดที่ไม่ทราบกฎเหล็กคอลัมน์ 72 อาจสับสนและคิดว่า
พิมพ์ชื่อตัวแปรผิดที่ไหนสักแห่ง ทั้งที่จริง ๆ แล้วปัญหาคือความยาวบรรทัดล้วน ๆ — จึงเป็นเหตุผลสำคัญที่
ต้องรู้จักกฎนี้ตั้งแต่ต้น เพื่อวินิจฉัย error ประเภทนี้ได้อย่างรวดเร็วในอนาคต

---

## ขั้นตอนที่ 404: EVALUATE กับนิพจน์ทางคณิตศาสตร์เป็น Subject

### แนวคิด

Subject ของ `EVALUATE` ไม่จำเป็นต้องเป็นตัวแปรเดี่ยว ๆ เท่านั้น แต่สามารถเป็น**นิพจน์ทางคณิตศาสตร์**
(arithmetic expression) ได้โดยตรง เช่น ผลรวมของตัวแปรสองตัว COBOL จะคำนวณค่าของนิพจน์ก่อน
แล้วจึงนำผลลัพธ์ไปเทียบกับแต่ละ `WHEN` ตามปกติ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-EXPRESSION-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-QTY-A    PIC 9(3) VALUE 6.
       01  WS-QTY-B    PIC 9(3) VALUE 4.

       PROCEDURE DIVISION.
      *> The subject can be an arithmetic expression, not just
      *> a plain data item.
           EVALUATE WS-QTY-A + WS-QTY-B
               WHEN 0 THRU 5
                   DISPLAY "SMALL COMBINED ORDER"
               WHEN 6 THRU 10
                   DISPLAY "MEDIUM COMBINED ORDER"
               WHEN OTHER
                   DISPLAY "LARGE COMBINED ORDER"
           END-EVALUATE.
           STOP RUN.
```

### อธิบายโค้ด

- `EVALUATE WS-QTY-A + WS-QTY-B` — COBOL คำนวณ `WS-QTY-A + WS-QTY-B` (6 + 4 = 10) ก่อน
  แล้วจึงนำผลลัพธ์ 10 ไปเทียบกับแต่ละ `WHEN` เหมือนกับว่าเขียน `EVALUATE 10` ตรง ๆ
- ผลลัพธ์ 10 อยู่ในช่วง `WHEN 6 THRU 10` พอดี จึงแสดง "MEDIUM COMBINED ORDER"
- เทคนิคนี้มีประโยชน์มากเมื่อต้องจำแนกกลุ่มตามค่าที่คำนวณได้ทันที โดยไม่ต้องสร้างตัวแปรใหม่มาเก็บ
  ผลลัพธ์ก่อนแยกต่างหาก

### ผลลัพธ์ที่ได้จากการรันจริง

```
MEDIUM COMBINED ORDER
```

### ข้อควรระวัง

- นิพจน์ที่ซับซ้อนมากใน subject ของ `EVALUATE` อาจทำให้โค้ดอ่านยากขึ้น หากนิพจน์ซับซ้อนมาก
  ควรพิจารณาคำนวณเก็บไว้ในตัวแปรที่มีชื่อสื่อความหมายก่อน (เช่น `COMPUTE WS-COMBINED-QTY =
  WS-QTY-A + WS-QTY-B` แล้วค่อย `EVALUATE WS-COMBINED-QTY`) เพื่อความชัดเจน
- ชนิดข้อมูลของผลลัพธ์นิพจน์ต้องเข้ากันได้กับค่าที่เปรียบเทียบใน `WHEN` (ตัวเลขกับตัวเลข) เช่นเดียวกับ
  กฎทั่วไปของ `EVALUATE`

### แบบฝึกหัดที่ 404.1

**โจทย์**: จงเขียน `EVALUATE` ที่ใช้นิพจน์ `WS-PRICE * WS-QTY` เป็น subject เพื่อจำแนกยอดสั่งซื้อ
เป็น "SMALL" (น้อยกว่า 1000), "MEDIUM" (1000-4999), "LARGE" (5000 ขึ้นไป)

**เฉลย**:
```cobol
EVALUATE WS-PRICE * WS-QTY
    WHEN 0 THRU 999.99
        DISPLAY "SMALL"
    WHEN 1000 THRU 4999.99
        DISPLAY "MEDIUM"
    WHEN OTHER
        DISPLAY "LARGE"
END-EVALUATE
```

---

## ขั้นตอนที่ 405: EVALUATE TRUE ALSO TRUE — เงื่อนไขอิสระสองตัวพร้อมกัน

### แนวคิด

รูปแบบ `EVALUATE TRUE ALSO TRUE` (หรือมากกว่า) เป็นรูปแบบพิเศษที่ทรงพลังมาก: ทุก subject เป็น
`TRUE` หมายความว่าทุกตำแหน่งใน `WHEN` เป็น**เงื่อนไข boolean อิสระจากกันโดยสิ้นเชิง** ไม่ผูกกับ
ตัวแปรตัวใดตัวหนึ่งเป็นพิเศษ ต่างจากขั้นตอนที่ 401 ที่ subject แรกยังผูกกับตัวแปรค่าคงที่อยู่
รูปแบบนี้เหมาะกับการสร้างตารางการตัดสินใจแบบ "ปัจจัย A เป็นจริงไหม, ปัจจัย B เป็นจริงไหม" อิสระกัน

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-TRUE-ALSO-TRUE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-STOCK-QTY     PIC 9(5) VALUE 0.
       01  WS-PENDING-ORDER PIC 9(5) VALUE 20.

       PROCEDURE DIVISION.
      *> EVALUATE TRUE ALSO TRUE: two fully independent boolean
      *> conditions checked together, like a two-factor rule table.
           EVALUATE TRUE ALSO TRUE
               WHEN WS-STOCK-QTY = 0 ALSO WS-PENDING-ORDER > 0
                   DISPLAY "OUT OF STOCK BUT RESTOCK IS PENDING"
               WHEN WS-STOCK-QTY = 0 ALSO WS-PENDING-ORDER = 0
                   DISPLAY "OUT OF STOCK, NO RESTOCK PLANNED"
               WHEN WS-STOCK-QTY > 0 ALSO WS-PENDING-ORDER > 0
                   DISPLAY "IN STOCK, MORE ARRIVING SOON"
               WHEN OTHER
                   DISPLAY "IN STOCK, NORMAL LEVEL"
           END-EVALUATE.
           STOP RUN.
```

### อธิบายโค้ด

- `EVALUATE TRUE ALSO TRUE` — ทั้งสอง subject เป็น `TRUE` หมายความว่าทั้งสองตำแหน่งในแต่ละ
  `WHEN` เป็นเงื่อนไข boolean อิสระ ไม่ใช่การเทียบค่ากับตัวแปรตัวเดียวซ้ำ ๆ
- `WHEN WS-STOCK-QTY = 0 ALSO WS-PENDING-ORDER > 0` — ตรวจสอบสองเงื่อนไขที่ไม่เกี่ยวข้องกันทาง
  ไวยากรณ์เลย (คนละตัวแปร คนละตัวดำเนินการเปรียบเทียบ) แต่ทั้งคู่ต้องเป็นจริงพร้อมกัน
- ด้วย `WS-STOCK-QTY` = 0 และ `WS-PENDING-ORDER` = 20 (> 0) ตรงกับ `WHEN` แรกพอดี

### ผลลัพธ์ที่ได้จากการรันจริง

```
OUT OF STOCK BUT RESTOCK IS PENDING
```

### เปรียบเทียบกับ IF ซ้อนกัน

ตรรกะเดียวกันนี้เขียนด้วย Nested IF ได้เช่นกัน (`IF WS-STOCK-QTY = 0 ... IF WS-PENDING-ORDER >
0 ...`) แต่ `EVALUATE TRUE ALSO TRUE` ให้ภาพรวมของ "ตารางการตัดสินใจ 2 มิติ" ที่ชัดเจนกว่ามาก
เมื่ออ่านทีละ `WHEN` เพราะทุกกรณีถูกจัดเรียงเป็นแถวที่เห็นครบทุกชุดค่าผสมในที่เดียว ไม่ต้องไล่ตาม
โครงสร้างการซ้อนที่ลึกลงไป

### ข้อควรระวัง

- ต้องระวังอย่าให้กรณีใดกรณีหนึ่งตกหล่น (เช่น ในตัวอย่างนี้ไม่ได้เขียนกรณี
  `WS-STOCK-QTY > 0 ALSO WS-PENDING-ORDER = 0` แยกไว้ตรง ๆ) ซึ่งจะตกไปที่ `WHEN OTHER`
  โดยอัตโนมัติ — ต้องตรวจสอบให้แน่ใจว่า `WHEN OTHER` ให้ผลลัพธ์ที่ถูกต้องสำหรับทุกกรณีที่เหลือจริง ๆ
- ยิ่งมีปัจจัยอิสระมากเท่าไร (subject มากขึ้น) จำนวนชุดค่าผสมที่เป็นไปได้ยิ่งเพิ่มแบบทวีคูณ (2 ปัจจัย
  = 4 กรณี, 3 ปัจจัย = 8 กรณี) ควรพิจารณาว่าคุ้มค่าที่จะเขียนทุกกรณีจริงหรือไม่

### แบบฝึกหัดที่ 405.1

**โจทย์**: จงเพิ่ม `WHEN` ที่ขาดหายไปสำหรับกรณี `WS-STOCK-QTY > 0 ALSO WS-PENDING-ORDER = 0`
พร้อมข้อความ "IN STOCK, NO MORE COMING" แล้ววาง `WHEN` นี้ในตำแหน่งที่ถูกต้อง

**เฉลยแนวทาง**:
```cobol
           EVALUATE TRUE ALSO TRUE
               WHEN WS-STOCK-QTY = 0 ALSO WS-PENDING-ORDER > 0
                   DISPLAY "OUT OF STOCK BUT RESTOCK IS PENDING"
               WHEN WS-STOCK-QTY = 0 ALSO WS-PENDING-ORDER = 0
                   DISPLAY "OUT OF STOCK, NO RESTOCK PLANNED"
               WHEN WS-STOCK-QTY > 0 ALSO WS-PENDING-ORDER > 0
                   DISPLAY "IN STOCK, MORE ARRIVING SOON"
               WHEN WS-STOCK-QTY > 0 ALSO WS-PENDING-ORDER = 0
                   DISPLAY "IN STOCK, NO MORE COMING"
               WHEN OTHER
                   DISPLAY "IN STOCK, NORMAL LEVEL"
           END-EVALUATE
```
เมื่อเขียนครบทั้ง 4 กรณีแล้ว `WHEN OTHER` จะไม่มีทางถูกเรียกใช้งานเลย (เพราะครอบคลุมทุกชุดค่าผสม
ที่เป็นไปได้ของทั้งสองเงื่อนไขแล้ว) แต่ยังคงควรเก็บ `WHEN OTHER` ไว้เสมอตามหลักการที่เรียนใน
Part 011 ขั้นตอนที่ 106

---

## ขั้นตอนที่ 406: EVALUATE กับ Intrinsic Function

### แนวคิด

ทั้ง subject และ object ของ `EVALUATE` สามารถเป็นผลลัพธ์จาก **`FUNCTION`** (intrinsic function
ที่เรียนใน Part 036-037) ได้โดยตรง มีประโยชน์มากเมื่อต้องแปลงค่าก่อนเปรียบเทียบ เช่น แปลงตัวอักษร
เป็นตัวพิมพ์ใหญ่ก่อนเทียบ เพื่อไม่ให้การพิมพ์ตัวพิมพ์เล็ก-ใหญ่ต่างกันทำให้ผลลัพธ์ผิดพลาด

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-FUNCTION-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NAME    PIC X(10) VALUE "somchai".

       PROCEDURE DIVISION.
      *> Intrinsic FUNCTION calls can appear as the subject
      *> or as an object inside WHEN.
           EVALUATE FUNCTION UPPER-CASE(WS-NAME)
               WHEN "SOMCHAI"
                   DISPLAY "WELCOME BACK, SOMCHAI"
               WHEN OTHER
                   DISPLAY "UNKNOWN USER"
           END-EVALUATE.
           STOP RUN.
```

### อธิบายโค้ด

- `EVALUATE FUNCTION UPPER-CASE(WS-NAME)` — แปลง `WS-NAME` ("somchai" ตัวพิมพ์เล็ก) เป็น
  ตัวพิมพ์ใหญ่ทั้งหมดก่อน ("SOMCHAI") แล้วจึงนำผลลัพธ์ไปเทียบกับ `WHEN`
- วิธีนี้ทำให้ผู้ใช้พิมพ์ชื่อผู้ใช้ด้วยตัวพิมพ์เล็ก, ใหญ่, หรือผสมก็ยังได้ผลลัพธ์ถูกต้องเสมอ โดยไม่ต้อง
  เขียน `WHEN` แยกสำหรับทุกรูปแบบการพิมพ์ที่เป็นไปได้ (เช่น "SOMCHAI", "somchai", "Somchai" ฯลฯ)

### ผลลัพธ์ที่ได้จากการรันจริง

```
WELCOME BACK, SOMCHAI
```

### ข้อควรระวัง

- การเรียก `FUNCTION` ซ้ำหลายครั้งใน `EVALUATE` เดียว (เช่น ถ้ามีหลาย `WHEN` ที่ต้องเรียก
  FUNCTION กับค่าเดิมซ้ำ ๆ) อาจไม่มีประสิทธิภาพเท่ากับการคำนวณค่าไว้ในตัวแปรก่อนหนึ่งครั้ง — แต่
  ในกรณีของ subject (ที่คำนวณเพียงครั้งเดียวตอนต้น `EVALUATE`) ไม่มีปัญหานี้เพราะ COBOL คำนวณ
  ค่าของ subject เพียงครั้งเดียวเท่านั้น
- ต้องแน่ใจว่าค่าที่ FUNCTION คืนกลับมามีชนิดข้อมูลที่เข้ากันได้กับค่าที่เปรียบเทียบใน `WHEN`
  (ในที่นี้ `FUNCTION UPPER-CASE` คืนค่าเป็นข้อความ จึงเทียบกับ string literal ได้ตรงตามชนิด)

### แบบฝึกหัดที่ 406.1

**โจทย์**: จงเขียน `EVALUATE` ที่ใช้ `FUNCTION LENGTH(WS-INPUT)` เป็น subject เพื่อแยกความยาว
ข้อความเป็น "TOO SHORT" (น้อยกว่า 3), "OK" (3-20), "TOO LONG" (มากกว่า 20)

**เฉลย**:
```cobol
EVALUATE FUNCTION LENGTH(FUNCTION TRIM(WS-INPUT))
    WHEN 0 THRU 2
        DISPLAY "TOO SHORT"
    WHEN 3 THRU 20
        DISPLAY "OK"
    WHEN OTHER
        DISPLAY "TOO LONG"
END-EVALUATE
```
(ใช้ `FUNCTION TRIM` ร่วมด้วยเพื่อไม่ให้ช่องว่างท้ายข้อความที่ `PIC X` เติมมาให้ ถูกนับรวมเป็นความยาว
ข้อความโดยไม่ได้ตั้งใจ)

---

## ขั้นตอนที่ 407: กรณีศึกษาธุรกิจจริง — กฎการคิดค่าจัดส่งหลายปัจจัย

### แนวคิด

ขั้นตอนนี้รวมเทคนิคจากขั้นตอนที่ 401 (หลาย subject) และขั้นตอนที่ 402 (เงื่อนไขผสม) เข้าด้วยกัน
ในสถานการณ์ธุรกิจจริงที่ซับซ้อนขึ้น: การคิดค่าจัดส่งที่ขึ้นกับประเภทคำสั่งซื้อ, ความภักดีของลูกค้า
ผสมกับยอดสั่งซื้อ (compound), และการที่สินค้าเปราะบางหรือไม่

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-BUSINESS-CASE-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ORDER-TYPE     PIC X(1) VALUE "R".
       01  WS-CUSTOMER-YEARS PIC 9(2) VALUE 5.
       01  WS-ORDER-AMOUNT   PIC 9(7)V99 VALUE 12000.00.
       01  WS-IS-FRAGILE     PIC X(1) VALUE "Y".
       01  WS-SHIPPING-FEE   PIC 9(5)V99 VALUE 0.

       PROCEDURE DIVISION.
      *> A realistic combined business rule: shipping fee depends
      *> on order type AND a compound loyalty/amount condition,
      *> AND whether the item is fragile.
           EVALUATE WS-ORDER-TYPE ALSO TRUE ALSO WS-IS-FRAGILE
               WHEN "R" ALSO (WS-CUSTOMER-YEARS >= 3
                               AND WS-ORDER-AMOUNT >= 10000.00)
                        ALSO "Y"
                   MOVE 50.00 TO WS-SHIPPING-FEE
               WHEN "R" ALSO (WS-CUSTOMER-YEARS >= 3
                               AND WS-ORDER-AMOUNT >= 10000.00)
                        ALSO "N"
                   MOVE 0.00 TO WS-SHIPPING-FEE
               WHEN "R" ALSO TRUE ALSO "Y"
                   MOVE 80.00 TO WS-SHIPPING-FEE
               WHEN "R" ALSO TRUE ALSO "N"
                   MOVE 30.00 TO WS-SHIPPING-FEE
               WHEN OTHER
                   MOVE 100.00 TO WS-SHIPPING-FEE
           END-EVALUATE.
           DISPLAY "SHIPPING FEE: " WS-SHIPPING-FEE.
           STOP RUN.
```

### อธิบายโค้ด

- Subject ตัวที่ 2 เป็น `TRUE` ทำให้ตำแหน่งที่ 2 ในแต่ละ `WHEN` เขียนเป็นเงื่อนไขผสมเต็มรูปได้
  `(WS-CUSTOMER-YEARS >= 3 AND WS-ORDER-AMOUNT >= 10000.00)` — ผสมเทคนิคจากขั้นตอนที่ 401
  (หลาย subject) กับขั้นตอนที่ 402 (compound condition ภายใน object) เข้าด้วยกันได้อย่างลงตัว
- ลำดับ `WHEN` สำคัญมาก: กรณี "ลูกค้าประจำที่ซื้อเยอะ" ต้องตรวจสอบ**ก่อน**กรณี "ลูกค้าทั่วไป" เสมอ
  (เหมือนหลักการเรียงเงื่อนไขเฉพาะเจาะจงก่อนเงื่อนไขกว้างที่เรียนใน Part 011 ขั้นตอนที่ 108)
- ด้วย order type "R", สมาชิก 5 ปี (>= 3), ยอด 12000.00 (>= 10000.00), สินค้าเปราะบาง "Y"
  ตรงกับ `WHEN` แรกพอดี ค่าจัดส่ง 50.00

### ผลลัพธ์ที่ได้จากการรันจริง

```
SHIPPING FEE: 00050.00
```

เมื่อทดสอบเปลี่ยน `WS-CUSTOMER-YEARS` เป็น 1 (ยังไม่ครบเงื่อนไขความภักดี) ผลลัพธ์ที่ทดสอบได้คือ
ตกไปที่ `WHEN "R" ALSO TRUE ALSO "Y"` ให้ค่าจัดส่ง **80.00** แทน (สินค้าเปราะบางแต่ไม่ใช่ลูกค้า
ประจำที่ซื้อเยอะพอ)

### ข้อควรระวัง

- เมื่อผสมหลาย subject กับเงื่อนไขผสมภายใน object ควรจัดรูปแบบโค้ด (ตัดบรรทัด, จัดวงเล็บ) ให้อ่าน
  ง่ายที่สุดเท่าที่จะทำได้ เพราะความซับซ้อนสะสมขึ้นเร็วมาก
- ควรทดสอบทุกเส้นทาง (path) ของกฎธุรกิจด้วยชุดข้อมูลที่ครอบคลุมทุกกรณี ก่อนนำไปใช้งานจริง เพราะ
  การพลาดเรียงลำดับ `WHEN` ผิดในกฎที่ซับซ้อนขนาดนี้อาจตรวจจับได้ยากมากด้วยตาเปล่า

### แบบฝึกหัดที่ 407.1

**โจทย์**: จงอธิบายว่าทำไมต้องตรวจสอบ `WHEN "R" ALSO (WS-CUSTOMER-YEARS >= 3 AND
WS-ORDER-AMOUNT >= 10000.00) ALSO "Y"` **ก่อน** `WHEN "R" ALSO TRUE ALSO "Y"`

**เฉลยแนวทาง**: เพราะ `EVALUATE` ตรวจสอบ `WHEN` ตามลำดับบนลงล่างและหยุดที่ตัวแรกที่ตรงกันทั้งหมด
หาก `WHEN "R" ALSO TRUE ALSO "Y"` (เงื่อนไขกว้างกว่า เพราะ `TRUE` ตรงกับทุกกรณี) ถูกวางไว้ก่อน
ลูกค้าประจำที่ซื้อเยอะและสินค้าเปราะบางจะตรงกับ `WHEN` นี้ทันทีและได้ค่าจัดส่ง 80.00 อย่างผิดพลาด
ทั้งที่ควรได้รับส่วนลดพิเศษเหลือ 50.00 ตามเงื่อนไขที่เฉพาะเจาะจงกว่า — หลักการนี้เหมือนกับที่เรียนใน
Part 011 ขั้นตอนที่ 108 เรื่อง FizzBuzz ทุกประการ: ต้องเรียงเงื่อนไขเฉพาะเจาะจงกว่าไว้ก่อนเสมอ

---

## ขั้นตอนที่ 408: EVALUATE เทียบกับแนวทาง Table-Driven (SEARCH)

### แนวคิด

เมื่อจำนวนกรณีใน `EVALUATE` เพิ่มขึ้นมาก (เช่น มากกว่า 15-20 `WHEN`) โดยเฉพาะกรณีที่เป็นการ
**ค้นหาค่าคงที่แบบตรง ๆ** (ไม่ใช่ช่วงหรือเงื่อนไขผสมซับซ้อน) อาจถึงเวลาพิจารณาแนวทางอื่นที่ยืดหยุ่น
กว่า คือ **ตารางค้นหา (table-driven lookup)** ด้วย `OCCURS` และ `SEARCH` ที่เรียนใน Part 016-017
ขั้นตอนนี้จะเปรียบเทียบทั้งสองแนวทางเคียงข้างกัน

### ตัวอย่างโค้ด — แนวทาง EVALUATE (ยาวขึ้นเรื่อย ๆ ตามจำนวนกรณี)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-DAY-LOOKUP-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DAY-NUM  PIC 9(1) VALUE 3.
       01  WS-DAY-NAME PIC X(10).

       PROCEDURE DIVISION.
           EVALUATE WS-DAY-NUM
               WHEN 1 MOVE "MONDAY"    TO WS-DAY-NAME
               WHEN 2 MOVE "TUESDAY"   TO WS-DAY-NAME
               WHEN 3 MOVE "WEDNESDAY" TO WS-DAY-NAME
               WHEN 4 MOVE "THURSDAY"  TO WS-DAY-NAME
               WHEN 5 MOVE "FRIDAY"    TO WS-DAY-NAME
               WHEN 6 MOVE "SATURDAY"  TO WS-DAY-NAME
               WHEN 7 MOVE "SUNDAY"    TO WS-DAY-NAME
               WHEN OTHER MOVE "INVALID"  TO WS-DAY-NAME
           END-EVALUATE.
           DISPLAY "DAY: " WS-DAY-NAME.
           STOP RUN.
```

### ตัวอย่างโค้ด — แนวทาง Table-Driven ด้วย SEARCH

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SEARCH-DAY-LOOKUP-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DAY-NUM  PIC 9(1) VALUE 3.
       01  WS-DAY-NAME PIC X(10).

       01  DAY-TABLE.
           05  DAY-ENTRY OCCURS 7 TIMES INDEXED BY DAY-IDX.
               10  DAY-NUM   PIC 9(1).
               10  DAY-NAME  PIC X(10).

       PROCEDURE DIVISION.
           MOVE 1 TO DAY-NUM(1). MOVE "MONDAY"    TO DAY-NAME(1).
           MOVE 2 TO DAY-NUM(2). MOVE "TUESDAY"   TO DAY-NAME(2).
           MOVE 3 TO DAY-NUM(3). MOVE "WEDNESDAY" TO DAY-NAME(3).
           MOVE 4 TO DAY-NUM(4). MOVE "THURSDAY"  TO DAY-NAME(4).
           MOVE 5 TO DAY-NUM(5). MOVE "FRIDAY"    TO DAY-NAME(5).
           MOVE 6 TO DAY-NUM(6). MOVE "SATURDAY"  TO DAY-NAME(6).
           MOVE 7 TO DAY-NUM(7). MOVE "SUNDAY"    TO DAY-NAME(7).

      *> Table-driven lookup with SEARCH instead of a long
      *> EVALUATE with one WHEN per possible value.
           SET DAY-IDX TO 1.
           SEARCH DAY-ENTRY
               WHEN DAY-NUM(DAY-IDX) = WS-DAY-NUM
                   MOVE DAY-NAME(DAY-IDX) TO WS-DAY-NAME
           END-SEARCH.

           DISPLAY "DAY: " WS-DAY-NAME.
           STOP RUN.
```

### อธิบายโค้ด

ทั้งสองโปรแกรมให้ผลลัพธ์เหมือนกันทุกประการ แต่แนวทาง `SEARCH` เก็บข้อมูล (ตัวเลขวัน ↔ ชื่อวัน)
ไว้ใน**ตาราง**แทนที่จะฝังอยู่ในโครงสร้างโค้ดโดยตรง ทำให้:

- เพิ่ม/แก้ไข/ลบรายการทำได้ที่จุดเดียว (แก้ตาราง) โดยไม่ต้องแก้โครงสร้าง `EVALUATE`/`SEARCH`
- ถ้าข้อมูลมาจากไฟล์หรือฐานข้อมูลภายนอก สามารถโหลดเข้าตารางแบบไดนามิกได้ (ต่างจาก `EVALUATE`
  ที่ต้องเขียน `WHEN` ตายตัวในโค้ด compile-time)
- เมื่อจำนวนรายการเพิ่มขึ้นมาก (หลักร้อย/พัน) `SEARCH ALL` (Binary Search จาก Part 017) จะเร็ว
  กว่า `EVALUATE` ที่ต้องไล่ทีละ `WHEN` จากบนลงล่างมาก

### ผลลัพธ์ที่ได้จากการรันจริง

ทั้งสองโปรแกรมให้ผลลัพธ์เหมือนกัน:
```
DAY: WEDNESDAY
```

### เมื่อไหร่ควรใช้แบบไหน

| สถานการณ์ | แนะนำให้ใช้ |
|---|---|
| จำนวนกรณีน้อย (< 10-15) และแต่ละกรณีมีตรรกะต่างกัน | **EVALUATE** — อ่านง่าย ชัดเจนในโค้ดทันที |
| มีเงื่อนไขช่วง (`THRU`) หรือเงื่อนไขผสมซับซ้อน | **EVALUATE** — `SEARCH` ไม่เหมาะกับเงื่อนไขซับซ้อนเท่า |
| จำนวนกรณีมาก (ค้นหาค่าคงที่ตรง ๆ) หรือข้อมูลมาจากภายนอก | **SEARCH/ตาราง** — แก้ไขง่ายกว่า ไม่ต้องแก้โค้ด |
| ต้องการความเร็วสูงสุดกับข้อมูลจำนวนมากที่เรียงลำดับแล้ว | **SEARCH ALL (Binary Search)** |

### ข้อควรระวัง

- การเปลี่ยนจาก `EVALUATE` เป็น `SEARCH` ไม่ได้แปลว่าดีกว่าเสมอไป — สำหรับกรณีน้อย ๆ ที่มีตรรกะ
  ต่างกันชัดเจน `EVALUATE` ยังคงอ่านง่ายกว่ามาก การสร้างตารางเพิ่มความซับซ้อนของโค้ดโดยไม่จำเป็น
  หากมีเพียงไม่กี่กรณี
- ทบทวน `SEARCH`/`SEARCH ALL`/`OCCURS`/`INDEXED BY` โดยละเอียดได้ที่ Part 016-017

### แบบฝึกหัดที่ 408.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมที่แปลงรหัสประเทศ 2 ตัวอักษร (เช่น "TH", "US", "JP", ...) เป็น
ชื่อประเทศเต็ม ที่มีรหัสประเทศมากกว่า 100 ประเทศ จึงควรใช้แนวทาง Table-Driven มากกว่า `EVALUATE`

**เฉลยแนวทาง**: เพราะการเขียน `EVALUATE` ที่มี 100 กว่า `WHEN` จะทำให้โค้ดยาวมากและอ่านยาก
มาก อีกทั้งหากต้องการเพิ่มประเทศใหม่หรือแก้ไขชื่อประเทศ ต้องแก้โค้ดและคอมไพล์ใหม่ทุกครั้ง ในขณะที่
แนวทาง Table-Driven สามารถเก็บข้อมูลรหัสประเทศ-ชื่อประเทศไว้ในตาราง (หรือแม้แต่โหลดจากไฟล์
ภายนอกตอนเริ่มโปรแกรม) ทำให้แก้ไขข้อมูลได้โดยไม่ต้องแก้โค้ดหรือคอมไพล์ใหม่เลย และเมื่อใช้
`SEARCH ALL` (Binary Search) กับข้อมูล 100 กว่ารายการที่เรียงลำดับแล้ว จะค้นหาได้เร็วกว่าการไล่
เทียบทีละ `WHEN` ใน `EVALUATE` อย่างมีนัยสำคัญ

---

## ขั้นตอนที่ 409: ข้อผิดพลาดที่พบบ่อยใน Advanced EVALUATE

### แนวคิด

ขั้นตอนนี้รวบรวมข้อผิดพลาดที่พบบ่อยที่สุดเมื่อใช้ `EVALUATE` ขั้นสูง พร้อม**หลักฐาน compile error
จริง**ที่ทดสอบได้จากการพยายามเขียนโค้ดผิดโดยตั้งใจ เพื่อให้คุณจดจำอาการของข้อผิดพลาดเหล่านี้ได้ทันที
เมื่อพบเจอในอนาคต

### ตัวอย่างโค้ดที่ผิดพลาดจริง — จำนวน Object ไม่ตรงกับจำนวน Subject

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. EVALUATE-MISMATCH-ERROR-DEMO.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-A   PIC X(1) VALUE "A".
       01  WS-B   PIC X(1) VALUE "B".

       PROCEDURE DIVISION.
      *> BUG: EVALUATE declares 2 subjects (WS-A ALSO WS-B) but
      *> the WHEN below provides only 1 object.
           EVALUATE WS-A ALSO WS-B
               WHEN "A"
                   DISPLAY "MISSING SECOND OBJECT"
           END-EVALUATE.
           STOP RUN.
```

### ผลการทดสอบคอมไพล์จริง

```
error: wrong number of WHEN parameters
```

คอมไพเลอร์ตรวจพบทันทีว่า `EVALUATE WS-A ALSO WS-B` ประกาศ subject ไว้ 2 ตัว แต่ `WHEN "A"`
ให้ object มาเพียง 1 ตัว จึง compile error ทันที **นี่เป็นข้อผิดพลาดที่ดี**เพราะคอมไพเลอร์จับได้
ตั้งแต่ขั้นตอนคอมไพล์ ไม่ปล่อยให้กลายเป็นบั๊กที่ซ่อนอยู่ตอนรันจริง

### วิธีแก้ไข

```cobol
           EVALUATE WS-A ALSO WS-B
               WHEN "A" ALSO "B"
                   DISPLAY "BOTH OBJECTS PROVIDED CORRECTLY"
               WHEN OTHER
                   DISPLAY "NO MATCH"
           END-EVALUATE.
```

ทดสอบคอมไพล์และรันแล้ว **ผ่านสำเร็จ** ให้ผลลัพธ์ "BOTH OBJECTS PROVIDED CORRECTLY" เมื่อ
`WS-A` = "A" และ `WS-B` = "B" ตรงกัน

### สรุปข้อผิดพลาดที่พบบ่อยอื่น ๆ (อ้างอิงจากขั้นตอนก่อนหน้าใน Part 011 และ Part นี้)

| ข้อผิดพลาด | ผลที่ตามมา | อ้างอิง |
|---|---|---|
| จำนวน object ไม่ตรงกับจำนวน subject | Compile error `wrong number of WHEN parameters` | ขั้นตอนนี้ |
| เรียงลำดับ `WHEN` ผิด (เงื่อนไขกว้างไว้ก่อนเงื่อนไขแคบ) | รันผ่านแต่ผลลัพธ์ผิดแบบเงียบ ๆ | Part 011 ขั้นตอนที่ 102, 108; Part 041 ขั้นตอนที่ 407 |
| ไม่มี `WHEN OTHER` | ไม่มีอะไรเกิดขึ้นเมื่อไม่ตรงกับ WHEN ใดเลย | Part 011 ขั้นตอนที่ 106 |
| วาง `WHEN OTHER` ไม่ใช่ตัวสุดท้าย | `WHEN` ที่ตามมาไม่มีโอกาสทำงานเลย | Part 011 ขั้นตอนที่ 106 |
| บรรทัดเกิน column 72 จากการเยื้องลึกของ Nested EVALUATE | Compile error ชื่อตัวแปรถูกตัด (`is not defined`) | Part 041 ขั้นตอนที่ 403 |

### ข้อควรระวัง

- ข้อผิดพลาดที่คอมไพเลอร์จับได้ (เช่น จำนวน object ไม่ตรง) ยังถือว่า "โชคดี" เพราะรู้ปัญหาทันที
  ข้อผิดพลาดที่**อันตรายกว่ามาก**คือข้อผิดพลาดเชิงตรรกะที่คอมไพล์ผ่านปกติแต่ผลลัพธ์ผิด (เช่น เรียง
  ลำดับ `WHEN` ผิด) ซึ่งต้องอาศัยการทดสอบด้วยชุดข้อมูลที่ครอบคลุมทุกกรณีเท่านั้นจึงจะจับได้
- ควรเขียนแบบทดสอบ (test case) ที่ครอบคลุมทุกเส้นทางของ `EVALUATE` ที่ซับซ้อน โดยเฉพาะเมื่อมี
  หลาย subject หรือเงื่อนไขผสมซ้อนกันหลายชั้น

### แบบฝึกหัดที่ 409.1

**โจทย์**: จงอธิบายว่าทำไม compile error `wrong number of WHEN parameters` จึงถือเป็น "ข้อดี"
ของ COBOL เมื่อเทียบกับภาษาที่ไม่มีการตรวจสอบลักษณะนี้

**เฉลยแนวทาง**: เพราะข้อผิดพลาดถูกตรวจพบ**ตั้งแต่ขั้นตอนคอมไพล์** ก่อนที่โปรแกรมจะถูกนำไปใช้งาน
จริงเลยด้วยซ้ำ ทำให้โปรแกรมเมอร์แก้ไขได้ทันทีในขั้นตอนพัฒนา แทนที่จะปล่อยให้กลายเป็นบั๊กที่ซ่อนอยู่
จนกว่าจะมีข้อมูลจริงมาทดสอบ (หรือแย่กว่านั้นคือถูกค้นพบหลังนำไปใช้งานจริงแล้ว) ภาษาโปรแกรมที่มีการ
ตรวจสอบชนิดข้อมูลและโครงสร้างไวยากรณ์อย่างเข้มงวดตั้งแต่ compile-time (เช่น COBOL ในกรณีนี้)
จึงช่วยลดความเสี่ยงของบั๊กที่ไปโผล่ตอน production ได้มากกว่าภาษาที่ตรวจสอบหลวมกว่า

---

## ขั้นตอนที่ 410: โปรแกรมรวบยอด — เครื่องมือคำนวณราคาสินค้า (Pricing Engine)

### แนวคิด

ปิดท้าย Part นี้ด้วยโปรแกรมที่ผสานทุกเทคนิคขั้นสูงของ `EVALUATE`: หลาย subject ด้วย `ALSO`,
Condition Names (88-level จาก Part 010), ช่วงค่าด้วย `THRU`, และ Nested `EVALUATE` — จำลอง
เครื่องมือคำนวณส่วนลดของระบบขายสินค้าจริงที่ต้องพิจารณาทั้งประเภทลูกค้า, ยอดสั่งซื้อ, และวันพิเศษ

### ตัวอย่างโค้ด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PRICING-ENGINE-CAPSTONE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-TYPE    PIC X(1) VALUE "R".
           88  CUST-VIP                 VALUE "V".
           88  CUST-REGULAR              VALUE "R".
           88  CUST-GUEST                VALUE "G".
       01  WS-ORDER-AMOUNT     PIC 9(7)V99 VALUE 6000.00.
       01  WS-IS-HOLIDAY       PIC X(1) VALUE "Y".
           88  TODAY-IS-HOLIDAY          VALUE "Y".
       01  WS-DISCOUNT-PCT     PIC 9(2)V9 VALUE 0.
       01  WS-FINAL-AMOUNT     PIC 9(7)V99.

       PROCEDURE DIVISION.
      *> Capstone: a pricing engine combining EVALUATE ALSO,
      *> THRU ranges, condition-names and a nested EVALUATE.
           EVALUATE TRUE ALSO TRUE
               WHEN CUST-VIP ALSO TRUE
                   EVALUATE TRUE
                       WHEN WS-ORDER-AMOUNT >= 5000.00
                           MOVE 30.0 TO WS-DISCOUNT-PCT
                       WHEN WS-ORDER-AMOUNT >= 1000.00
                           MOVE 20.0 TO WS-DISCOUNT-PCT
                       WHEN OTHER
                           MOVE 10.0 TO WS-DISCOUNT-PCT
                   END-EVALUATE
               WHEN CUST-REGULAR ALSO TODAY-IS-HOLIDAY
                   MOVE 15.0 TO WS-DISCOUNT-PCT
               WHEN CUST-REGULAR ALSO TRUE
                   EVALUATE TRUE
                       WHEN WS-ORDER-AMOUNT >= 1000.00
                           MOVE 10.0 TO WS-DISCOUNT-PCT
                       WHEN OTHER
                           MOVE 5.0 TO WS-DISCOUNT-PCT
                   END-EVALUATE
               WHEN CUST-GUEST ALSO TODAY-IS-HOLIDAY
                   MOVE 5.0 TO WS-DISCOUNT-PCT
               WHEN OTHER
                   MOVE 0.0 TO WS-DISCOUNT-PCT
           END-EVALUATE.

           COMPUTE WS-FINAL-AMOUNT =
               WS-ORDER-AMOUNT
               - (WS-ORDER-AMOUNT * WS-DISCOUNT-PCT / 100).

           DISPLAY "CUSTOMER TYPE : " WS-CUSTOMER-TYPE.
           DISPLAY "ORDER AMOUNT  : " WS-ORDER-AMOUNT.
           DISPLAY "DISCOUNT %    : " WS-DISCOUNT-PCT.
           DISPLAY "FINAL AMOUNT  : " WS-FINAL-AMOUNT.
           STOP RUN.
```

### อธิบายโค้ด

- `EVALUATE TRUE ALSO TRUE` ชั้นนอกใช้ **Condition Names** (`CUST-VIP`, `CUST-REGULAR`,
  `CUST-GUEST`, `TODAY-IS-HOLIDAY` จาก Part 010) เป็น object แทนการเขียนเทียบค่าดิบตรง ๆ
  ทำให้โค้ดอ่านเป็นภาษาอังกฤษได้เกือบสมบูรณ์ (ผสานปรัชญาของ Grace Hopper ที่กล่าวถึงตั้งแต่ Part 001)
- เมื่อเป็น `CUST-VIP` หรือ `CUST-REGULAR` (กรณีไม่ใช่วันหยุด) จะมี **Nested EVALUATE** ชั้นใน
  ตรวจสอบช่วงยอดสั่งซื้อ (`WS-ORDER-AMOUNT >= 5000.00` เป็นต้น) เพื่อกำหนดเปอร์เซ็นต์ส่วนลด
  ตามระดับยอดซื้อ
- ลำดับ `WHEN` เรียงจากเฉพาะเจาะจงไปกว้าง: ตรวจ VIP ก่อน, ตรวจ Regular+Holiday (เฉพาะเจาะจง
  กว่า Regular ธรรมดา) ก่อน Regular ทั่วไป, ตรวจ Guest+Holiday ก่อนตกไปที่ `WHEN OTHER`

### ผลลัพธ์ที่ได้จากการรันจริง (ทดสอบ 3 กรณี)

**กรณีที่ 1** — ลูกค้า VIP ("V"), ยอดสั่งซื้อ 6000.00:
```
CUSTOMER TYPE : V
ORDER AMOUNT  : 0006000.00
DISCOUNT %    : 30.0
FINAL AMOUNT  : 0004200.00
```

**กรณีที่ 2** — ลูกค้าประจำ ("R") ในวันหยุด ("Y"), ยอดสั่งซื้อ 6000.00:
```
CUSTOMER TYPE : R
ORDER AMOUNT  : 0006000.00
DISCOUNT %    : 15.0
FINAL AMOUNT  : 0005100.00
```

**กรณีที่ 3** — ลูกค้าทั่วไป ("G") ในวันธรรมดา (ไม่ใช่วันหยุด), ยอดสั่งซื้อ 3500.00:
```
CUSTOMER TYPE : G
ORDER AMOUNT  : 0003500.00
DISCOUNT %    : 00.0
FINAL AMOUNT  : 0003500.00
```

ทั้ง 3 กรณีทดสอบยืนยันแล้วว่าตรงตามตรรกะที่ออกแบบไว้ทุกประการ

### ข้อควรระวัง

- โปรแกรมนี้แสดงให้เห็นว่าการผสาน **หลาย subject** + **Condition Names** + **Nested EVALUATE**
  เข้าด้วยกันสามารถแทนกฎธุรกิจที่ซับซ้อนได้อย่างเป็นระเบียบ แต่ก็ต้องแลกมาด้วยความยาวและความลึก
  ของโค้ดที่เพิ่มขึ้น ควรพิจารณาแยกส่วนคำนวณส่วนลดออกเป็น paragraph หรือ subprogram แยกต่างหาก
  (Part 014, Part 031) หากตรรกะซับซ้อนขึ้นไปอีกในระบบจริง
- ทดสอบทุก path ของกฎธุรกิจนี้อย่างละเอียดก่อนใช้งานจริงเสมอ ตามที่เน้นย้ำในขั้นตอนที่ 409 — ในที่นี้
  ทดสอบยืนยันแล้ว 3 เส้นทางหลัก แต่ระบบจริงควรทดสอบให้ครบทุกชุดค่าผสมที่เป็นไปได้

### แบบฝึกหัดที่ 410.1

**โจทย์**: จงต่อยอดโปรแกรมข้างต้น เพิ่มเงื่อนไขพิเศษ: ถ้าเป็นลูกค้า VIP **และ** เป็นวันหยุด **และ**
ยอดสั่งซื้อ >= 5000.00 ให้ได้ส่วนลดพิเศษ 35% แทนที่ 30% ปกติ (ต้องเพิ่มเงื่อนไขนี้ไว้**ก่อน**เงื่อนไข
VIP เดิม เพราะเป็นกรณีที่เฉพาะเจาะจงกว่า)

**เฉลยแนวทาง**:
```cobol
           EVALUATE TRUE ALSO TRUE
               WHEN CUST-VIP ALSO (TODAY-IS-HOLIDAY
                                    AND WS-ORDER-AMOUNT >= 5000.00)
                   MOVE 35.0 TO WS-DISCOUNT-PCT
               WHEN CUST-VIP ALSO TRUE
                   EVALUATE TRUE
                       WHEN WS-ORDER-AMOUNT >= 5000.00
                           MOVE 30.0 TO WS-DISCOUNT-PCT
                       WHEN WS-ORDER-AMOUNT >= 1000.00
                           MOVE 20.0 TO WS-DISCOUNT-PCT
                       WHEN OTHER
                           MOVE 10.0 TO WS-DISCOUNT-PCT
                   END-EVALUATE
               ...
```
สังเกตว่าเงื่อนไขใหม่ต้องอยู่**ก่อน** `WHEN CUST-VIP ALSO TRUE` เดิมเสมอ เพราะ `TRUE` ในเงื่อนไข
เดิมจะครอบคลุมทุกกรณีของ VIP ไว้หมดแล้ว (รวมถึงกรณีวันหยุด+ยอดสูงด้วย) หากไม่ย้ายเงื่อนไขใหม่ขึ้น
ก่อน เงื่อนไขพิเศษ 35% จะไม่มีทางถูกเรียกใช้งานเลย — เป็นการทบทวนหลักการเรียงลำดับเงื่อนไขเฉพาะ
เจาะจงก่อนกว้างอีกครั้งจากขั้นตอนที่ 407

---

## สรุปท้ายบท

ใน Part นี้ เราได้เจาะลึก `EVALUATE` ในระดับขั้นสูงอย่างครบถ้วน โดยทดสอบคอมไพล์และรันจริงทุก
ตัวอย่างด้วย GnuCOBOL 4.0 (early-dev):

- `EVALUATE ... ALSO` กับสาม subject ขึ้นไป สำหรับตารางการตัดสินใจหลายมิติ
- เงื่อนไขผสม (`AND`/`OR`/`NOT`) ภายใน `WHEN` เดียว
- **Nested EVALUATE** พร้อมกับดักจริงเรื่องกฎเหล็กคอลัมน์ 72 ที่พิสูจน์ด้วย compile error จริง
- นิพจน์ทางคณิตศาสตร์เป็น subject ของ `EVALUATE`
- `EVALUATE TRUE ALSO TRUE` สำหรับเงื่อนไขอิสระหลายตัวพร้อมกัน
- `EVALUATE` ร่วมกับ Intrinsic Function (`FUNCTION UPPER-CASE`, `FUNCTION LENGTH`)
- กรณีศึกษาธุรกิจจริงที่ผสานหลาย subject กับเงื่อนไขผสม
- การเปรียบเทียบ `EVALUATE` กับแนวทาง Table-Driven ด้วย `SEARCH`
- ข้อผิดพลาดที่พบบ่อยพร้อมหลักฐาน compile error จริง (`wrong number of WHEN parameters`)
- โปรแกรมรวบยอดเครื่องมือคำนวณราคาสินค้าที่ผสาน Condition Names, หลาย subject, และ Nested
  EVALUATE เข้าด้วยกัน ทดสอบยืนยันแล้ว 3 เส้นทางหลัก

`EVALUATE` ขั้นสูงคือเครื่องมือสำคัญสำหรับเขียนกฎธุรกิจที่ซับซ้อนให้อ่านง่ายและดูแลรักษาได้ แต่ต้องใช้
ด้วยความระมัดระวังเรื่องการเรียงลำดับเงื่อนไขและความยาวบรรทัดเสมอ ใน **Part 042** เราจะกลับไป
เจาะลึกเงื่อนไขอีกประเภทหนึ่งที่กล่าวถึงเพียงผิวเผินใน Part 010 คือ **Class Condition**
(`IS NUMERIC`, `IS ALPHABETIC`) และ **Sign Condition** (`IS POSITIVE`, `IS NEGATIVE`,
`IS ZERO`) ในระดับที่ลึกและครอบคลุมมากขึ้น พร้อมกรณีทดสอบจริงที่เผยพฤติกรรมที่คาดไม่ถึงของ
`IS NUMERIC` กับข้อมูลที่มีช่องว่าง

**[← กลับไป Part 040](part-040-screen-section.md)** | **[ไปยัง Part 042: Class Condition และ Sign Condition →](part-042-class-sign-condition.md)**
