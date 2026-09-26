# Part 033: COPY Statement และการใช้ Copybooks (ขั้นตอนที่ 321–330)

## คำนำของ Part นี้

ใน Part 031 และ 032 เราได้เรียนรู้วิธีแบ่งโปรแกรมใหญ่ออกเป็น Subprogram ที่เรียกกันด้วย `CALL`
พร้อมส่งพารามิเตอร์ผ่าน `BY REFERENCE`, `BY CONTENT`, และ `BY VALUE` ไปแล้ว แต่ยังมีปัญหาหนึ่งที่
การแบ่ง Subprogram เพียงอย่างเดียวแก้ไม่ได้: เมื่อหลายโปรแกรมต้องใช้ **โครงสร้างข้อมูล (Record
Layout)** แบบเดียวกัน เช่น โปรแกรม A อ่านไฟล์ลูกค้า และโปรแกรม B ก็ต้องอ่านไฟล์ลูกค้าเดียวกันนี้ด้วย
ถ้าเราพิมพ์ `01 CUSTOMER-RECORD.` พร้อมฟิลด์ทั้งหมดซ้ำในทั้งสองโปรแกรม แล้ววันหนึ่งลูกค้าขอเพิ่มฟิลด์
ใหม่ เราจะต้องจำให้ได้ว่ามีโปรแกรมกี่ตัวที่ต้องแก้ไขตาม และถ้าลืมแก้แม้แต่ที่เดียว โปรแกรมสองตัวจะอ่าน
ไฟล์เดียวกันด้วยโครงสร้างที่ไม่ตรงกัน ซึ่งเป็นบั๊กที่อันตรายมากในระบบจริง

Part นี้จะแนะนำคำสั่ง **`COPY`** ซึ่งเป็นทางออกของปัญหานี้: เราเขียนโครงสร้างข้อมูล (หรือแม้แต่
โค้ดส่วนอื่น) ไว้ในไฟล์แยกต่างหากที่เรียกว่า **Copybook** (นามสกุลนิยมใช้ `.cpy`) เพียงครั้งเดียว
แล้วให้ทุกโปรแกรมที่ต้องการใช้โครงสร้างนั้น "คัดลอก" เนื้อหาเข้ามาด้วยคำสั่ง `COPY` ตอนคอมไพล์
เราจะได้เรียนรู้ตั้งแต่การสร้าง Copybook พื้นฐาน การใช้ซ้ำในหลายโปรแกรม การปรับแต่งเนื้อหาด้วย
`REPLACING`, การจัดระเบียบ Copybook เป็นคลัง (Library), ไปจนถึงการนำ Copybook ไปใช้จริงในระบบ
เล็ก ๆ ที่มีหลายโปรแกรมทำงานร่วมกัน

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน และทุกตัวอย่าง
> ผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง

---

## ขั้นตอนที่ 321: ปัญหาโค้ดซ้ำซ้อน และ COPY Statement เบื้องต้น

### ปัญหาที่ COPY เข้ามาแก้ไข

ลองนึกภาพระบบที่มีโปรแกรม COBOL 10 ตัว และทุกตัวต้องอ้างอิงโครงสร้างข้อมูลลูกค้าชุดเดียวกัน
(รหัสลูกค้า, ชื่อ, ยอดคงเหลือ) ถ้าไม่มี `COPY` เราต้องพิมพ์กลุ่มฟิลด์เดิมซ้ำ 10 ครั้งในไฟล์ 10 ไฟล์
วันไหนที่ต้องเพิ่มฟิลด์ "เบอร์โทรศัพท์" เข้าไป เราต้องไปแก้ไขทั้ง 10 ไฟล์ให้ตรงกันเป๊ะ ๆ ทุกตัวอักษร
มิฉะนั้นโปรแกรมที่เขียนไฟล์กับโปรแกรมที่อ่านไฟล์จะเข้าใจโครงสร้าง Record ไม่ตรงกัน ทำให้ข้อมูลเพี้ยน
โดยไม่มี Error ใด ๆ เตือนเราเลย (เพราะ COBOL ไม่รู้ว่าไฟล์จริงมีโครงสร้างอย่างไร มันอ่านตาม PICTURE
ที่เราบอกไว้เท่านั้น)

**`COPY`** แก้ปัญหานี้ด้วยหลักการง่าย ๆ: เขียนโครงสร้างไว้ **ที่เดียว** ในไฟล์ `.cpy` แล้วให้ทุกโปรแกรม
`COPY` ไฟล์นั้นเข้ามา คำสั่งนี้ทำงานตอน **คอมไพล์** (ไม่ใช่ตอนรัน) โดยตัวประมวลผลก่อนคอมไพล์ของ
COBOL (Copy Precompiler / Library Text Preprocessor) จะแทนที่บรรทัด `COPY "ชื่อไฟล์"` ด้วยเนื้อหา
ทั้งหมดของไฟล์นั้น ราวกับเราพิมพ์เนื้อหานั้นด้วยมือตรงตำแหน่งนั้นเอง

### ไวยากรณ์พื้นฐานของ COPY

```
COPY ชื่อ-copybook.
COPY "ชื่อ-copybook".
COPY ชื่อ-copybook OF ชื่อไลบรารี.
COPY ชื่อ-copybook REPLACING ==ข้อความเดิม== BY ==ข้อความใหม่==.
```

ในหลักสูตรนี้เราจะใช้รูปแบบ `COPY "ชื่อไฟล์.cpy".` เป็นหลัก เพราะชัดเจนและตรงกับพฤติกรรมจริงบน
GnuCOBOL มากที่สุด

### ตัวอย่าง: COPY โครงสร้างลูกค้าเข้ามาใน WORKING-STORAGE

ก่อนอื่นเราสร้างไฟล์ copybook ชื่อ `custrec.cpy`:

```cobol
      *> Copybook: CUSTREC.CPY
      *> Shared customer record layout, used by many programs.
       01  CUSTOMER-RECORD.
           05  CUST-ID             PIC X(6).
           05  CUST-NAME           PIC X(20).
           05  CUST-BALANCE        PIC S9(7)V99.
```

สังเกตว่าไฟล์ copybook **ไม่มี** `IDENTIFICATION DIVISION`, `PROGRAM-ID`, หรือ Division ใด ๆ เลย
มันเป็นเพียง "ชิ้นส่วนโค้ด" ที่รอถูกแปะเข้าไปในโปรแกรมอื่น จะแปะไว้ตรงไหนก็ได้ที่ไวยากรณ์ ณ จุดนั้น
ยอมรับ (ในที่นี้คือใน `WORKING-STORAGE SECTION`)

จากนั้นในโปรแกรมหลัก:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP321-COPY-BASICS.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> COPY pulls the entire content of CUSTREC.CPY in right here,
      *> as if we had typed it by hand, before the compiler continues.
           COPY "custrec.cpy".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "C0001 "        TO CUST-ID.
           MOVE "SOMCHAI JAIDEE" TO CUST-NAME.
           MOVE 1500.50          TO CUST-BALANCE.
           DISPLAY "ID: "      CUST-ID.
           DISPLAY "NAME: "    CUST-NAME.
           DISPLAY "BALANCE: " CUST-BALANCE.
           STOP RUN.
```

คำสั่งคอมไพล์ (ต้องบอก GnuCOBOL ว่าจะหาไฟล์ `.cpy` จากโฟลเดอร์ไหนด้วยตัวเลือก `-I`):

```
cobc -x -I copybooks -o step321_copy_basics step321_copy_basics.cob
```

**ผลลัพธ์:**

```
ID: C0001 
NAME: SOMCHAI JAIDEE      
BALANCE: +0001500.50
```

### อธิบายจุดสำคัญ

- `COPY "custrec.cpy".` ต้องจบด้วยจุด (`.`) เสมอ เหมือนประโยคอื่น ๆ ใน COBOL
- หลังคอมไพล์เสร็จ ตัวแปร `CUST-ID`, `CUST-NAME`, `CUST-BALANCE` (และกลุ่ม `CUSTOMER-RECORD`)
  ใช้งานได้ใน `PROCEDURE DIVISION` เหมือนกับว่าเราพิมพ์มันไว้ใน `WORKING-STORAGE SECTION` ด้วยมือ
  ตั้งแต่แรก — เพราะ COPY ทำงาน "ก่อน" ขั้นตอนคอมไพล์จริงจะเริ่มวิเคราะห์ไวยากรณ์
- ตัวเลือก `-I copybooks` บอกคอมไพเลอร์ว่าให้ค้นหาไฟล์ที่ระบุใน `COPY` จากโฟลเดอร์ `copybooks`
  เพิ่มเติม (รายละเอียดเรื่องการจัดโฟลเดอร์ Copybook จะอธิบายลึกใน ขั้นตอนที่ 328)

### ข้อควรระวัง

- Copybook ไม่ใช่โปรแกรม COBOL ที่สมบูรณ์ อย่าใส่ `IDENTIFICATION DIVISION` หรือ `PROCEDURE
  DIVISION` เข้าไปในไฟล์ `.cpy` ที่เก็บแค่โครงสร้างข้อมูล
- COPY ทำงานตอนคอมไพล์เท่านั้น หากแก้ไขเนื้อหาใน `.cpy` แล้ว โปรแกรมที่เคยคอมไพล์ไปแล้วจะ**ไม่**
  ได้รับการเปลี่ยนแปลงจนกว่าจะคอมไพล์ใหม่ทุกตัวที่ COPY ไฟล์นั้น — นี่คือทั้งข้อดี (ควบคุมได้ว่าจะ
  อัปเดตโปรแกรมไหนเมื่อไร) และข้อเสีย (ลืมคอมไพล์ใหม่บางตัวแล้วโปรแกรมนั้นยังใช้โครงสร้างเก่าอยู่)
  ที่ต้องระวัง

### แบบฝึกหัดที่ 321.1

**โจทย์**: จงอธิบายว่าทำไมการแก้ไข Copybook หนึ่งไฟล์ ไม่ได้ทำให้โปรแกรมที่เคยคอมไพล์ไปแล้ว
(ไฟล์ `.exe` หรือไฟล์ที่รันได้ที่มีอยู่ก่อนแล้ว) เปลี่ยนพฤติกรรมโดยอัตโนมัติ

**เฉลย**: เพราะ `COPY` เป็นกลไกที่ทำงานเฉพาะตอน**คอมไพล์** (compile-time) เท่านั้น มันจะแทนที่
เนื้อหาไฟล์ `.cpy` ลงในซอร์สโค้ดก่อนที่คอมไพเลอร์จะแปลงเป็นไฟล์รันได้ เมื่อคอมไพล์เสร็จแล้ว ไฟล์
รันได้ (executable) จะไม่มีการอ้างอิงถึงไฟล์ `.cpy` อีกต่อไป — เนื้อหาของมันถูก "ฝัง" ลงในไฟล์รันได้
ไปเรียบร้อยแล้ว การแก้ `.cpy` ภายหลังจึงไม่ส่งผลย้อนกลับไปยังไฟล์ที่คอมไพล์ไปแล้ว ต้องคอมไพล์ทุก
โปรแกรมที่ COPY ไฟล์นั้นใหม่เสมอเพื่อให้การเปลี่ยนแปลงมีผล

---

## ขั้นตอนที่ 322: โครงสร้างของไฟล์ Copybook และการจัดวางฟิลด์

### สิ่งที่ควรอยู่ใน Copybook

Copybook ที่ดีควรมีลักษณะดังนี้:

1. **มีคอมเมนต์อธิบายที่หัวไฟล์เสมอ** — บอกว่าคือ copybook อะไร ใช้ทำอะไร ใครเป็นคนดูแล
2. **มีเฉพาะ Data Description Entries** (`01`, `05`, `10` ฯลฯ พร้อม `PICTURE`) หรือโค้ดส่วนอื่นที่
   ตั้งใจให้ใช้ซ้ำ เช่น กลุ่มคำสั่งใน `PROCEDURE DIVISION` (พบน้อยกว่าแบบ Data Description)
3. **ตั้งชื่อไฟล์ให้สื่อความหมาย** เช่น `custrec.cpy` (Customer Record), `orderfd.cpy`
   (Order File Description) เพื่อให้ทีมงานเข้าใจตรงกันว่าแต่ละไฟล์เก็บอะไร

### ตัวอย่าง: การใช้ Copybook เดียวกันในโปรแกรมที่มีตรรกะต่างกัน (Program A)

โค้ดต่อไปนี้ COPY `custrec.cpy` ตัวเดิมจาก ขั้นตอนที่ 321 แต่ใช้ในบริบทที่ต่างออกไป
(คำนวณยอดคงเหลือใหม่หลังฝากเงินเพิ่ม):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP322-PROGRAM-A.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
           COPY "custrec.cpy".
       01  WS-NEW-BALANCE      PIC S9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "C0010 "        TO CUST-ID.
           MOVE "PRASERT KAEWTA" TO CUST-NAME.
           MOVE 800.00           TO CUST-BALANCE.
           COMPUTE WS-NEW-BALANCE = CUST-BALANCE + 250.00.
           DISPLAY "PROGRAM-A - CUSTOMER: " CUST-NAME.
           DISPLAY "PROGRAM-A - NEW BALANCE: " WS-NEW-BALANCE.
           STOP RUN.
```

**ผลลัพธ์:**

```
PROGRAM-A - CUSTOMER: PRASERT KAEWTA      
PROGRAM-A - NEW BALANCE: +0001050.00
```

### อธิบายจุดสำคัญ

- เราสามารถประกาศตัวแปรของโปรแกรมเอง (`WS-NEW-BALANCE`) ต่อท้าย `COPY` ได้ตามปกติ Copybook
  ไม่ได้ผูกขาดพื้นที่ `WORKING-STORAGE SECTION` ทั้งหมด เป็นเพียงส่วนหนึ่งของมัน
- ฟิลด์ `CUST-ID`, `CUST-NAME`, `CUST-BALANCE` มาจาก Copybook เดียวกันเป๊ะกับ ขั้นตอนที่ 321
  แต่โปรแกรมนี้ใช้มันเพื่อคำนวณยอดใหม่ ซึ่งเป็นตรรกะทางธุรกิจที่แตกต่างไปโดยสิ้นเชิง — นี่คือหัวใจ
  ของ COPY: **โครงสร้างข้อมูลใช้ร่วมกัน แต่ตรรกะการทำงานของแต่ละโปรแกรมเป็นอิสระต่อกัน**

### ข้อควรระวัง

- อย่าใส่ตัวแปรที่ใช้เฉพาะโปรแกรมใดโปรแกรมหนึ่ง (เช่น `WS-NEW-BALANCE` ในตัวอย่างนี้) ปนเข้าไปใน
  Copybook ที่ตั้งใจให้ "ใช้ร่วมกัน" มิฉะนั้น Copybook จะเทอะทะและสร้างความสับสนว่าฟิลด์ไหนเป็น
  ของส่วนกลางจริง ๆ ฟิลด์ไหนเป็นของเฉพาะโปรแกรม
- ถ้า Copybook มีชื่อฟิลด์ที่ชนกับตัวแปรที่โปรแกรมประกาศเองอยู่แล้ว (เช่น มี `01 CUST-ID` อยู่แล้ว
  ก่อน COPY) คอมไพเลอร์จะฟ้อง Error ว่าชื่อซ้ำ (duplicate data name) ต้องตั้งชื่อให้ไม่ชนกัน

### แบบฝึกหัดที่ 322.1

**โจทย์**: จงแก้ไขโปรแกรม `STEP322-PROGRAM-A` ให้คำนวณ `WS-NEW-BALANCE` เป็นยอดคงเหลือหลัง
หักค่าธรรมเนียม 15 บาท แทนที่จะฝากเพิ่ม 250 บาท

**เฉลย**:

```cobol
           COMPUTE WS-NEW-BALANCE = CUST-BALANCE - 15.00.
```

เพียงเปลี่ยนสูตรคำนวณใน `PROCEDURE DIVISION` เท่านั้น โดยไม่ต้องแก้ Copybook เลย เพราะโครงสร้าง
ข้อมูล (`CUST-BALANCE`) เหมือนเดิมทุกประการ มีเพียงตรรกะทางธุรกิจที่เปลี่ยนไป

---

## ขั้นตอนที่ 323: ใช้ Copybook เดียวกันในสองโปรแกรม (พิสูจน์การใช้ซ้ำ)

### หัวใจสำคัญของ COPY: หนึ่งไฟล์ ใช้ได้กับทุกโปรแกรม

ขั้นตอนนี้จะพิสูจน์ให้เห็นชัดว่า Copybook ตัวเดียวกันสามารถถูกใช้โดยโปรแกรมที่ **แยกไฟล์กันโดย
สิ้นเชิง** ได้ โดยไม่ต้องแก้ไข `custrec.cpy` แม้แต่บรรทัดเดียว

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP323-PROGRAM-B.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
           COPY "custrec.cpy".
       01  WS-TAX-AMOUNT       PIC S9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "C0011 "     TO CUST-ID.
           MOVE "SUDA MEECHAI" TO CUST-NAME.
           MOVE 2000.00       TO CUST-BALANCE.
           COMPUTE WS-TAX-AMOUNT = CUST-BALANCE * 0.07.
           DISPLAY "PROGRAM-B - CUSTOMER: " CUST-NAME.
           DISPLAY "PROGRAM-B - TAX (7%): " WS-TAX-AMOUNT.
           STOP RUN.
```

คอมไพล์และรันแยกจากโปรแกรม A โดยสิ้นเชิง:

```
cobc -x -I copybooks -o step323_program_b step323_program_b.cob
```

**ผลลัพธ์:**

```
PROGRAM-B - CUSTOMER: SUDA MEECHAI        
PROGRAM-B - TAX (7%): +0000140.00
```

### อธิบายจุดสำคัญ

- `STEP322-PROGRAM-A` และ `STEP323-PROGRAM-B` เป็นไฟล์ `.cob` คนละไฟล์ คอมไพล์แยกกันคนละครั้ง
  แต่ทั้งคู่ COPY `custrec.cpy` ไฟล์เดียวกันจากโฟลเดอร์ `copybooks` เดียวกัน
- หากวันหนึ่งต้องเพิ่มฟิลด์ `CUST-PHONE PIC X(10)` เข้าไปในโครงสร้างลูกค้า เราแก้ไข `custrec.cpy`
  เพียงที่เดียว แล้วคอมไพล์ใหม่ทั้ง Program A และ Program B (และโปรแกรมอื่นทุกตัวที่ COPY ไฟล์นี้)
  ทั้งสองโปรแกรมจะได้ฟิลด์ใหม่ทันทีโดยไม่ต้องพิมพ์ซ้ำ

### ข้อควรระวัง

- ในองค์กรจริงที่มีโปรแกรมนับร้อยนับพันตัว COPY ไฟล์เดียวกัน การแก้ไข Copybook จึงต้องผ่าน
  กระบวนการตรวจสอบผลกระทบ (Impact Analysis) อย่างรอบคอบ เพราะการเปลี่ยนแปลงเพียงจุดเดียวอาจ
  ส่งผลกระทบเป็นวงกว้างมาก — นี่คือเหตุผลที่องค์กร Mainframe ขนาดใหญ่มักมีทีมดูแล Copybook Library
  แยกต่างหาก และมีกระบวนการควบคุมเวอร์ชัน (Version Control) อย่างเข้มงวด

### แบบฝึกหัดที่ 323.1

**โจทย์**: หากมีโปรแกรมที่สามชื่อ `STEP323-PROGRAM-C` ต้องการ COPY `custrec.cpy` เช่นกัน แต่
ต้องการคำนวณ "ยอดคงเหลือหลังหักภาษี 7% แล้วบวกดอกเบี้ย 2%" จงร่างโครงส่วน `PROCEDURE DIVISION`
ของโปรแกรมนี้

**เฉลยแนวทาง**:

```cobol
       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "C0012 "  TO CUST-ID.
           MOVE "KANYA SOMSRI" TO CUST-NAME.
           MOVE 5000.00    TO CUST-BALANCE.
           COMPUTE WS-RESULT =
               CUST-BALANCE - (CUST-BALANCE * 0.07)
               + (CUST-BALANCE * 0.02).
           DISPLAY "PROGRAM-C - FINAL BALANCE: " WS-RESULT.
           STOP RUN.
```

จุดสำคัญคือ `COPY "custrec.cpy".` เหมือนเดิมทุกประการ มีเพียงสูตรคำนวณที่ต่างไปตามความต้องการ
ของโปรแกรมนี้เท่านั้น

---

## ขั้นตอนที่ 324: REPLACING Phrase แบบ Pseudo-text

### ปัญหา: Copybook ที่ต้อง "ปรับแต่ง" ตามบริบท

บางครั้งเราอยากเขียน Copybook ที่มีโครงสร้างเหมือนกันทุกประการ แต่ต้องใช้กับ "หน่วยงาน" หรือ
"เอนทิตี" (Entity) ที่ต่างกัน เช่น โครงสร้าง ID + ชื่อ แบบเดียวกันเป๊ะ แต่ใช้ได้ทั้งกับ "ลูกค้า"
(Customer) และ "ซัพพลายเออร์" (Supplier) โดยไม่อยากสร้าง Copybook แยกกันสองไฟล์ที่เนื้อหาซ้ำกัน
เกือบทั้งหมด คำตอบคือใช้ `REPLACING` ร่วมกับ **Pseudo-text** (ข้อความคั่นด้วยเครื่องหมาย `==`)

### สร้าง Copybook แบบ "Generic" ที่มีตัวยึด (Placeholder)

```cobol
      *> Copybook: GENREC.CPY
      *> Generic record layout - :PREFIX: is a placeholder that
      *> the calling program replaces with a real name via REPLACING.
       01  :PREFIX:-RECORD.
           05  :PREFIX:-ID         PIC X(6).
           05  :PREFIX:-NAME       PIC X(20).
```

`:PREFIX:` ในที่นี้เป็นเพียงข้อความธรรมดา (ไม่ใช่คำสงวนพิเศษ) ที่เราตั้งใจให้ `REPLACING` มาแทนที่
ทีหลัง สังเกตว่าถ้า COPY ไฟล์นี้ตรง ๆ โดยไม่ REPLACING โปรแกรมจะคอมไพล์ไม่ผ่าน เพราะ `:PREFIX:`
ไม่ใช่ชื่อตัวแปรที่ถูกต้องตามหลัก COBOL

### ตัวอย่าง: ใช้ REPLACING เพื่อสร้างสองโครงสร้างจาก Copybook เดียว

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP324-REPLACING.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> Same copybook, two different prefixes -> two different
      *> record layouts, both generated from one shared .cpy file.
           COPY "genrec.cpy" REPLACING ==:PREFIX:== BY ==SUPPLIER==.
           COPY "genrec.cpy" REPLACING ==:PREFIX:== BY ==CUSTOMER==.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "S0099 "      TO SUPPLIER-ID.
           MOVE "ACME TRADING" TO SUPPLIER-NAME.
           MOVE "C0099 "      TO CUSTOMER-ID.
           MOVE "SOMCHAI CO."  TO CUSTOMER-NAME.
           DISPLAY "SUPPLIER: " SUPPLIER-ID " " SUPPLIER-NAME.
           DISPLAY "CUSTOMER: " CUSTOMER-ID " " CUSTOMER-NAME.
           STOP RUN.
```

**ผลลัพธ์:**

```
SUPPLIER: S0099  ACME TRADING        
CUSTOMER: C0099  SOMCHAI CO.
```

### อธิบายจุดสำคัญ

- `COPY "genrec.cpy" REPLACING ==:PREFIX:== BY ==SUPPLIER==.` แทนที่ทุกจุดที่มีข้อความ
  `:PREFIX:` ในไฟล์ `genrec.cpy` ด้วยคำว่า `SUPPLIER` ก่อนนำเนื้อหาไปแปะในโปรแกรม ผลลัพธ์คือได้
  กลุ่ม `SUPPLIER-RECORD` พร้อม `SUPPLIER-ID` และ `SUPPLIER-NAME`
- เราเรียก `COPY "genrec.cpy"` **สองครั้ง** ในโปรแกรมเดียวกัน แต่ REPLACING คนละคำ ทำให้ได้
  โครงสร้างสองชุดที่ไม่ชนกัน (`SUPPLIER-RECORD` กับ `CUSTOMER-RECORD`) จาก Copybook ไฟล์เดียว
- เครื่องหมาย `==...==` เรียกว่า **Pseudo-text Delimiter** ใช้ล้อมข้อความที่ต้องการให้ค้นหา/แทนที่
  แบบคำต่อคำ (word-for-word) ไม่ใช่การแทนที่ตัวอักษรธรรมดา

### ข้อควรระวัง

- REPLACING จะแทนที่แบบ "คำ" (token) ไม่ใช่ "ตัวอักษรย่อย" ดังนั้นตัวยึดควรตั้งชื่อให้ไม่ไปพ้องกับ
  คำอื่นในไฟล์โดยไม่ตั้งใจ (การใช้ `:` คร่อมคำ เช่น `:PREFIX:` เป็นธรรมเนียมที่ช่วยลดโอกาสชนกับคำ
  ปกติ)
- ถ้า COPY Copybook ที่มีตัวยึดโดย**ไม่ใส่** `REPLACING` คอมไพเลอร์จะพยายามตีความ `:PREFIX:-ID`
  เป็นชื่อตัวแปรจริง ซึ่งจะทำให้เกิด Syntax Error เพราะ `:` ไม่ใช่อักขระที่ใช้ได้ในชื่อตัวแปร COBOL

### แบบฝึกหัดที่ 324.1

**โจทย์**: จงเพิ่มการ COPY ครั้งที่สามในโปรแกรม `STEP324-REPLACING` เพื่อสร้างโครงสร้างสำหรับ
"พนักงาน" (Employee) จาก `genrec.cpy` ตัวเดิม

**เฉลย**:

```cobol
           COPY "genrec.cpy" REPLACING ==:PREFIX:== BY ==EMPLOYEE==.
```

เพิ่มบรรทัดนี้ต่อจากสองบรรทัดเดิมใน `WORKING-STORAGE SECTION` แล้วจะได้กลุ่ม `EMPLOYEE-RECORD`
พร้อม `EMPLOYEE-ID` และ `EMPLOYEE-NAME` พร้อมใช้งานทันที โดยไม่ต้องแก้ไข `genrec.cpy` เลย

---

## ขั้นตอนที่ 325: COPY ใน FILE SECTION สำหรับ FD Record (ใช้ร่วมกันระหว่างโปรแกรมเขียน/อ่าน)

### ทำไม FD Record ถึงสำคัญที่สุดสำหรับ COPY

ย้อนกลับไปที่ Part 024-026 เราเรียนรู้ว่า `FD` (File Description) ต้องมี Record Layout ที่ตรงกัน
เป๊ะกับไฟล์จริงบนดิสก์ ถ้าโปรแกรมที่ **เขียน** ไฟล์ กับโปรแกรมที่ **อ่าน** ไฟล์นั้นมี Record Layout
ไม่ตรงกันแม้แต่ 1 ไบต์ ข้อมูลจะเพี้ยนทันทีโดยไม่มี Error เตือน (เพราะ COBOL File I/O ระดับพื้นฐาน
ไม่ได้ตรวจสอบโครงสร้างข้อมูลจริงในไฟล์ มันอ่าน/เขียนตามที่ `FD` บอกไว้เท่านั้น) นี่คือจุดที่ COPY
มีคุณค่ามากที่สุด: การบังคับให้โปรแกรมเขียนและโปรแกรมอ่าน**ใช้ Copybook เดียวกัน** รับประกันว่า
โครงสร้างจะตรงกันเสมอ

### Copybook สำหรับ FD Record

```cobol
      *> Copybook: ORDERFD.CPY
      *> FD record layout for the order file, shared by the writer
      *> and reader programs so both always agree on the format.
       01  ORDER-RECORD.
           05  ORD-ID              PIC X(5).
           05  ORD-STATUS          PIC X(1).
               88  ORD-PENDING     VALUE "P".
               88  ORD-SHIPPED     VALUE "S".
               88  ORD-CANCELLED   VALUE "C".
           05  ORD-AMOUNT          PIC S9(7)V99.
```

### โปรแกรมที่ 1: ผู้เขียนไฟล์ (Writer)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP325-ORDER-WRITER.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ORDER-FILE ASSIGN TO "ORDERS325.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  ORDER-FILE.
           COPY "orderfd.cpy".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT ORDER-FILE.
           MOVE "O0001" TO ORD-ID.
           SET ORD-PENDING TO TRUE.
           MOVE 250.00 TO ORD-AMOUNT.
           WRITE ORDER-RECORD.

           MOVE "O0002" TO ORD-ID.
           SET ORD-SHIPPED TO TRUE.
           MOVE 999.50 TO ORD-AMOUNT.
           WRITE ORDER-RECORD.
           CLOSE ORDER-FILE.
           DISPLAY "WRITER: 2 ORDERS WRITTEN".
           STOP RUN.
```

### โปรแกรมที่ 2: ผู้อ่านไฟล์ (Reader) — COPY Copybook เดียวกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP325-ORDER-READER.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT ORDER-FILE ASSIGN TO "ORDERS325.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  ORDER-FILE.
      *> Same copybook as the writer - the record layout can
      *> never drift out of sync between the two programs.
           COPY "orderfd.cpy".

       WORKING-STORAGE SECTION.
       01  WS-EOF              PIC X VALUE "N".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN INPUT ORDER-FILE.
           PERFORM UNTIL WS-EOF = "Y"
               READ ORDER-FILE
                 AT END MOVE "Y" TO WS-EOF
                 NOT AT END
                   IF ORD-PENDING
                     DISPLAY ORD-ID " PENDING " ORD-AMOUNT
                   ELSE IF ORD-SHIPPED
                     DISPLAY ORD-ID " SHIPPED " ORD-AMOUNT
                   END-IF
               END-READ
           END-PERFORM.
           CLOSE ORDER-FILE.
           STOP RUN.
```

**ผลลัพธ์ (รัน Writer ก่อน แล้วรัน Reader):**

```
WRITER: 2 ORDERS WRITTEN
O0001 PENDING +0000250.00
O0002 SHIPPED +0000999.50
```

### อธิบายจุดสำคัญ

- ทั้งสองโปรแกรม COPY `orderfd.cpy` ตัวเดียวกันเข้าไปใน `FD ORDER-FILE.` ของตัวเอง ทำให้
  `ORDER-RECORD` ในทั้งสองโปรแกรมมีโครงสร้างไบต์ต่อไบต์ที่ตรงกันเป๊ะโดยอัตโนมัติ
- หากวันหนึ่งต้องเพิ่มฟิลด์ `ORD-CUSTOMER-ID PIC X(6)` เข้าไปในไฟล์ Order เราแก้ที่ `orderfd.cpy`
  ที่เดียว แล้วคอมไพล์ Writer และ Reader ใหม่ทั้งคู่ ทั้งสองโปรแกรมจะเข้าใจไฟล์รูปแบบใหม่ตรงกันทันที
- สังเกตว่าโค้ดในย่อหน้าเงื่อนไข (`IF ORD-PENDING ... ELSE IF ORD-SHIPPED ... END-IF`) เขียนแบบ
  ย่อหน้าตื้น ๆ โดยตั้งใจ เพราะถ้าเขียนย่อหน้าลึกเกินไป (เช่นย่อหน้าตามธรรมเนียมภาษาอื่นที่นิยมเยื้อง
  ครั้งละ 4-8 ช่องต่อชั้น) ความยาวบรรทัดจะเกิน**คอลัมน์ 72** ได้ง่ายมากเมื่อมีตัวแปรชื่อยาวและมีทั้ง
  `IF`/`ELSE IF`/`DISPLAY` ซ้อนกันหลายชั้น — เป็นเรื่องที่ต้องระวังทุกครั้งที่เขียนโค้ดซ้อนเงื่อนไขลึก ๆ

### ข้อควรระวัง

- Copybook สำหรับ `FD` **ต้อง** มีโครงสร้างตรงกับไฟล์จริงบนดิสก์แบบไบต์ต่อไบต์เสมอ การแก้ไข
  Copybook โดยไม่ migrate ข้อมูลเก่าในไฟล์ให้ตรงกับโครงสร้างใหม่ จะทำให้อ่านไฟล์เก่าผิดพลาดทันที
- ห้ามลืมคอมไพล์โปรแกรมที่เกี่ยวข้อง**ทุกตัว**ใหม่หลังแก้ Copybook ของ FD — ถ้า Writer คอมไพล์ใหม่
  แต่ Reader ยังเป็นเวอร์ชันเก่า ทั้งสองจะเข้าใจไฟล์คนละแบบ

### แบบฝึกหัดที่ 325.1

**โจทย์**: จงอธิบายว่าทำไมการ COPY Copybook เดียวกันในทั้ง Writer และ Reader จึงปลอดภัยกว่าการ
พิมพ์ `01 ORDER-RECORD.` แยกกันในสองโปรแกรม แม้จะพิมพ์เหมือนกันทุกตัวอักษรในตอนแรกก็ตาม

**เฉลย**: เพราะแม้ตอนแรกจะพิมพ์เหมือนกันทุกตัวอักษร แต่เมื่อเวลาผ่านไปและมีการแก้ไขโครงสร้าง
ข้อมูล (เช่น เพิ่มฟิลด์ใหม่) โปรแกรมเมอร์อาจแก้ไขโปรแกรมหนึ่งแล้วลืมแก้อีกโปรแกรมหนึ่ง เพราะมันเป็น
คนละไฟล์ คนละที่ในระบบ ไม่มีอะไรบังคับให้ทั้งสองที่ตรงกันเสมอ แต่เมื่อใช้ `COPY` จากไฟล์เดียวกัน
การแก้ไข Copybook เพียงจุดเดียวจะบังคับ (โดยอัตโนมัติผ่านกระบวนการคอมไพล์ใหม่) ให้ทุกโปรแกรมที่
COPY ไฟล์นั้นได้โครงสร้างใหม่ตรงกันเสมอ ลดความเสี่ยงจากความผิดพลาดของมนุษย์ (Human Error) ที่
อาจเกิดจากการแก้ไขแยกกันในหลายที่

---

## ขั้นตอนที่ 326: Copybook ที่มี 88-level Condition Names

### Copybook ไม่ได้เก็บแค่ PICTURE แต่เก็บ "ความหมายทางธุรกิจ" ได้ด้วย

จาก ขั้นตอนที่ 325 สังเกตว่า `orderfd.cpy` ไม่ได้มีแค่ `PIC X(1)` ธรรมดาสำหรับ `ORD-STATUS` แต่มี
**88-level Condition Names** (`ORD-PENDING`, `ORD-SHIPPED`, `ORD-CANCELLED`) ติดมาด้วย นี่เป็น
เทคนิคสำคัญมาก: เมื่อ Copybook เก็บทั้งโครงสร้างข้อมูล**และ**ความหมายทางธุรกิจของค่าต่าง ๆ ไว้ด้วย
ทุกโปรแกรมที่ COPY ไฟล์นี้จะได้ใช้ชื่อเงื่อนไขที่มีความหมาย (เช่น `ORD-PENDING`) แทนที่จะต้องจำเอง
ว่า `"P"` แปลว่าอะไร — ลดความเสี่ยงจากการพิมพ์ค่า Literal ผิดในแต่ละโปรแกรม

### ตัวอย่าง: ใช้ Condition Names จาก Copybook ร่วมกับ EVALUATE

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP326-CONDITION-NAMES.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> The copybook brings in ORD-STATUS plus its 88-level
      *> condition names (ORD-PENDING, ORD-SHIPPED, ORD-CANCELLED)
      *> ready to use, with zero extra typing in this program.
           COPY "orderfd.cpy".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "O0100" TO ORD-ID.
           MOVE 500.00  TO ORD-AMOUNT.
           SET ORD-CANCELLED TO TRUE.

           EVALUATE TRUE
               WHEN ORD-PENDING
                   DISPLAY ORD-ID ": still pending, awaiting stock"
               WHEN ORD-SHIPPED
                   DISPLAY ORD-ID ": already shipped"
               WHEN ORD-CANCELLED
                   DISPLAY ORD-ID ": cancelled, refund " ORD-AMOUNT
               WHEN OTHER
                   DISPLAY ORD-ID ": unknown status"
           END-EVALUATE.
           STOP RUN.
```

**ผลลัพธ์:**

```
O0100: cancelled, refund +0000500.00
```

### อธิบายจุดสำคัญ

- โปรแกรมนี้ไม่เคยพิมพ์ `88 ORD-PENDING VALUE "P".` เองเลย — ทุกอย่างมาจาก `COPY "orderfd.cpy"`
  ทั้งหมด แต่สามารถใช้ `SET ORD-CANCELLED TO TRUE` และ `EVALUATE TRUE WHEN ORD-PENDING ...`
  ได้ทันทีราวกับพิมพ์เอง
- นี่คือประโยชน์สำคัญของการฝัง Condition Names ไว้ใน Copybook: ทุกโปรแกรมในระบบที่ต้องตรวจสอบ
  สถานะออเดอร์จะใช้ชื่อเดียวกัน ความหมายเดียวกันเสมอ ไม่มีทางที่โปรแกรมหนึ่งจะเข้าใจว่า `"P"` แปลว่า
  "Paid" ในขณะที่อีกโปรแกรมเข้าใจว่า "Pending" เพราะทุกคนอ้างอิงจาก Copybook เดียวกัน

### ข้อควรระวัง

- ถ้าต้องเพิ่มสถานะใหม่ (เช่น `"R"` สำหรับ Returned) ต้องเพิ่ม 88-level ใหม่ใน Copybook แล้ว
  คอมไพล์ทุกโปรแกรมที่เกี่ยวข้องใหม่ — โปรแกรมเก่าที่ยังไม่ได้คอมไพล์ใหม่จะไม่รู้จักสถานะนี้ และจะตก
  ไปอยู่ใน `WHEN OTHER` เสมอ (ซึ่งอาจไม่ใช่พฤติกรรมที่ถูกต้องทางธุรกิจ)
- ควรตั้งชื่อ Condition Names ให้สื่อความหมายชัดเจนและไม่กำกวม (`ORD-PENDING` ดีกว่า `ORD-P`)
  เพราะ Copybook นี้จะถูกอ่านและใช้งานโดยโปรแกรมเมอร์หลายคนในหลายโปรแกรมตลอดอายุของระบบ

### แบบฝึกหัดที่ 326.1

**โจทย์**: จงจำลอง Copybook `orderfd.cpy` เวอร์ชันใหม่ที่เพิ่มสถานะ `"R"` (Returned) พร้อมชื่อ
เงื่อนไข `ORD-RETURNED` แล้วเขียนส่วน `EVALUATE` เพิ่มเติมให้รองรับสถานะนี้

**เฉลย**: เพิ่มบรรทัดต่อไปนี้ในไฟล์ `orderfd.cpy` ต่อจาก `88 ORD-CANCELLED VALUE "C".`:

```cobol
               88  ORD-RETURNED    VALUE "R".
```

แล้วเพิ่ม `WHEN` ใหม่ในโปรแกรมที่ใช้ `EVALUATE TRUE`:

```cobol
               WHEN ORD-RETURNED
                   DISPLAY ORD-ID ": returned by customer"
```

หลังจากนั้นต้องคอมไพล์**ทุกโปรแกรม**ที่ COPY `orderfd.cpy` ใหม่ เพื่อให้รู้จักสถานะ `ORD-RETURNED`

---

## ขั้นตอนที่ 327: Copybook ค่าคงที่ระดับ 78 (Constants)

### ทำไมค่าคงที่ควรอยู่ใน Copybook เช่นกัน

นอกจากโครงสร้าง Record แล้ว Copybook ยังเหมาะมากสำหรับเก็บ **ค่าคงที่ระดับองค์กร** (Application
Constants) เช่น อัตราภาษี, รหัสบริษัท, ค่าสูงสุดต่าง ๆ ที่หลายโปรแกรมต้องใช้ค่าเดียวกัน COBOL มี
**Level 78** ซึ่งเป็น Level Number พิเศษสำหรับประกาศค่าคงที่ (ไม่ใช่ตัวแปรที่แก้ไขค่าได้)

### ตัวอย่าง: Copybook ค่าคงที่

```cobol
      *> Copybook: CONSTANTS.CPY
      *> Application-wide constants (78-level items). Change the
      *> value in ONE place; every program that copies this file
      *> picks up the new value on its next compile.
       78  MAX-CUSTOMERS           VALUE 1000.
       78  TAX-RATE-PERCENT        VALUE 7.
       78  COMPANY-CODE            VALUE "TH01".
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP327-CONSTANTS.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
           COPY "constants.cpy".
       01  WS-SALE-AMOUNT      PIC S9(7)V99 VALUE 1000.00.
       01  WS-TAX-AMOUNT       PIC S9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "COMPANY CODE: "  COMPANY-CODE.
           DISPLAY "MAX CUSTOMERS: " MAX-CUSTOMERS.
           COMPUTE WS-TAX-AMOUNT =
               WS-SALE-AMOUNT * TAX-RATE-PERCENT / 100.
           DISPLAY "TAX ON " WS-SALE-AMOUNT ": " WS-TAX-AMOUNT.
           STOP RUN.
```

**ผลลัพธ์:**

```
COMPANY CODE: TH01
MAX CUSTOMERS: 1000
TAX ON +0001000.00: +0000070.00
```

### อธิบายจุดสำคัญ

- `78 MAX-CUSTOMERS VALUE 1000.` ประกาศชื่อ `MAX-CUSTOMERS` ให้แทนค่า `1000` ได้เลย โดย
  **ไม่ต้องมี `PICTURE`** เพราะ Level 78 ไม่ได้จองพื้นที่หน่วยความจำเหมือนตัวแปรทั่วไป มันเป็นเพียง
  "ชื่อเรียกแทนค่าคงที่" ที่คอมไพเลอร์จะแทนที่ด้วยค่าจริงตอนคอมไพล์
- ใช้ `MAX-CUSTOMERS`, `TAX-RATE-PERCENT`, `COMPANY-CODE` ใน `PROCEDURE DIVISION` ได้เหมือน
  ตัวแปรทั่วไป (อ่านค่าได้) แต่ **ไม่สามารถ `MOVE` ค่าใหม่ไปใส่มันได้** เพราะมันคือค่าคงที่
- ถ้าวันหนึ่งอัตราภาษีเปลี่ยนจาก 7% เป็น 8% เราแก้ที่ `constants.cpy` บรรทัดเดียว แล้วคอมไพล์ทุก
  โปรแกรมที่เกี่ยวข้องใหม่ ไม่ต้องไล่หาเลข `7` หรือ `0.07` ที่กระจัดกระจายอยู่ทั่วซอร์สโค้ดนับสิบไฟล์

### ข้อควรระวัง

- ห้ามพยายาม `MOVE ค่าใหม่ TO MAX-CUSTOMERS` เด็ดขาด เพราะ Level 78 ไม่ใช่ตัวแปรที่แก้ไขได้
  คอมไพเลอร์จะฟ้อง Error ทันที
- ควรตั้งชื่อค่าคงที่ให้สื่อความหมายชัดเจนกว่าตัวแปรทั่วไปด้วยซ้ำ เพราะค่าคงที่มักถูกอ้างอิงจากหลาย
  จุดในระบบเป็นเวลานาน ชื่อที่กำกวมจะสร้างความสับสนสะสมไปเรื่อย ๆ

### แบบฝึกหัดที่ 327.1

**โจทย์**: จงเพิ่มค่าคงที่ใหม่ชื่อ `DISCOUNT-RATE-PERCENT` ที่มีค่า `5` ลงใน `constants.cpy`
แล้วเขียนโค้ดคำนวณราคาหลังหักส่วนลดจากยอดขาย 1000.00

**เฉลย**: เพิ่มบรรทัดนี้ในไฟล์ `constants.cpy`:

```cobol
       78  DISCOUNT-RATE-PERCENT   VALUE 5.
```

แล้วในโปรแกรม:

```cobol
       01  WS-DISCOUNT-AMOUNT  PIC S9(7)V99.
       01  WS-FINAL-PRICE      PIC S9(7)V99.
       ...
           COMPUTE WS-DISCOUNT-AMOUNT =
               WS-SALE-AMOUNT * DISCOUNT-RATE-PERCENT / 100.
           COMPUTE WS-FINAL-PRICE =
               WS-SALE-AMOUNT - WS-DISCOUNT-AMOUNT.
           DISPLAY "FINAL PRICE: " WS-FINAL-PRICE.
```

---

## ขั้นตอนที่ 328: การจัดระเบียบ Copybook Library และตัวเลือก -I

### เก็บ Copybook ไว้ที่ไหนดี

ในโปรเจกต์จริง Copybook มักถูกแยกเก็บไว้ในโฟลเดอร์ของตัวเอง (เช่น `copybooks/`) แยกจากไฟล์
`.cob` ของโปรแกรม เพื่อให้จัดการง่าย และสามารถแชร์โฟลเดอร์นี้ให้หลายโปรเจกต์ใช้ร่วมกันได้ GnuCOBOL
มีตัวเลือก `-I` (Include path) ที่บอกคอมไพเลอร์ว่าให้ค้นหาไฟล์ที่ระบุใน `COPY` จากโฟลเดอร์ใดบ้าง
(สามารถระบุ `-I` ได้หลายครั้งเพื่อค้นหาจากหลายโฟลเดอร์)

### ตัวอย่าง: คอมไพล์โดยใช้ -I

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP328-LIBRARY-PATH.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> No path in the COPY statement itself - the compiler finds
      *> custrec.cpy by searching the directories given with -I.
           COPY "custrec.cpy".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "C0500 "    TO CUST-ID.
           MOVE "FOUND VIA -I" TO CUST-NAME.
           DISPLAY "COPYBOOK LOADED OK: " CUST-ID " " CUST-NAME.
           STOP RUN.
```

คำสั่งคอมไพล์ที่ **ใช้** `-I` ชี้ไปที่โฟลเดอร์ `copybooks`:

```
cobc -x -I copybooks -o step328_library_path step328_library_path.cob
```

**ผลลัพธ์:**

```
COPYBOOK LOADED OK: C0500  FOUND VIA -I        
```

### สิ่งที่เกิดขึ้นเมื่อ**ไม่** ใช้ -I

```
cobc -x -o step328_fail step328_library_path.cob
```

**ผลลัพธ์ (Error จริงจาก GnuCOBOL):**

```
step328_library_path.cob:9: error: custrec.cpy: No such file or directory
step328_library_path.cob: in paragraph 'MAIN-PARA':
step328_library_path.cob:13: error: 'CUST-ID' is not defined
step328_library_path.cob:14: error: 'CUST-NAME' is not defined
step328_library_path.cob:15: error: 'CUST-ID' is not defined
step328_library_path.cob:15: error: 'CUST-NAME' is not defined
```

### อธิบายจุดสำคัญ

- เมื่อคอมไพเลอร์หาไฟล์ `.cpy` ไม่เจอ มันจะรายงาน Error ที่บรรทัด `COPY` ก่อน แล้วจึง Error ต่อเนื่อง
  ที่ทุกจุดในโปรแกรมที่อ้างอิงถึงฟิลด์ซึ่งควรจะมาจาก Copybook นั้น (เพราะฟิลด์เหล่านั้นไม่เคยถูก
  ประกาศจริง) นี่เป็นรูปแบบ Error ที่พบบ่อยมากเมื่อทำงานกับ Copybook — เห็น Error เพียบแต่ต้นตอ
  จริง ๆ มักเป็นบรรทัดแรกสุดเสมอ
- อีกทางเลือกหนึ่งที่หลายองค์กร (และ GnuCOBOL) รองรับคือตัวแปรสภาพแวดล้อม `COBCPY` ซึ่งกำหนด
  รายการโฟลเดอร์ให้ค้นหา Copybook โดยไม่ต้องพิมพ์ `-I` ทุกครั้งที่คอมไพล์ (ตั้งค่าไว้ครั้งเดียวใน
  Environment ของเครื่อง Build)

### ข้อควรระวัง

- ทีมพัฒนาควรตกลงกันเรื่องโครงสร้างโฟลเดอร์ Copybook และตัวเลือก `-I`/`COBCPY` ให้เป็นมาตรฐาน
  เดียวกันทั้งทีม มิฉะนั้นสคริปต์ Build ของแต่ละคนอาจหาไฟล์เจอไม่เหมือนกัน ทำให้ผลการคอมไพล์ไม่
  สอดคล้องกันระหว่างเครื่อง
- Error ข้อความ `'CUST-ID' is not defined` ที่ตามมาหลัง Error `No such file or directory`
  มักทำให้ผู้เริ่มต้นสับสนและไปแก้ไขผิดจุด (เช่น คิดว่าตัวแปรพิมพ์ผิด) ทั้งที่ต้นตอจริงคือหาไฟล์
  Copybook ไม่เจอ ควรอ่าน Error ข้อความแรกสุดเสมอเมื่อเจอปัญหาเกี่ยวกับ COPY

### แบบฝึกหัดที่ 328.1

**โจทย์**: จงอธิบายว่าทำไม Error message ที่ได้เมื่อไม่ใช้ `-I` จึงมี Error หลายบรรทัด ทั้งที่ปัญหา
จริงมีเพียงจุดเดียวคือหาไฟล์ `.cpy` ไม่เจอ

**เฉลย**: เพราะเมื่อคอมไพเลอร์หาไฟล์ `custrec.cpy` ไม่เจอ มันไม่สามารถประกาศตัวแปร `CUST-ID`
และ `CUST-NAME` ได้เลย (เนื่องจากตัวแปรเหล่านี้ควรถูกประกาศผ่านเนื้อหาที่ COPY เข้ามา) ดังนั้นเมื่อ
คอมไพเลอร์อ่านต่อไปถึงส่วน `PROCEDURE DIVISION` ที่มีการอ้างอิงถึง `CUST-ID` และ `CUST-NAME`
มันจะพบว่าชื่อเหล่านี้ไม่เคยถูกประกาศไว้เลยในโปรแกรม (undefined) จึง Error ซ้ำทุกจุดที่ใช้ชื่อเหล่านั้น
Error ทั้งหมดจึงเป็นผลกระทบต่อเนื่อง (Cascading Errors) จากต้นตอเดียวคือบรรทัด `COPY` ที่ล้มเหลว

---

## ขั้นตอนที่ 329: Copybook ซ้อน Copybook (Nested COPY) และข้อจำกัดเรื่อง Level Number

### Copybook สามารถ COPY Copybook อื่นได้

ฟีเจอร์ที่มีประโยชน์อีกอย่างของ `COPY` คือมันสามารถใช้ **ซ้อนกันได้** — ไฟล์ `.cpy` หนึ่งไฟล์
สามารถมีคำสั่ง `COPY` เรียกไฟล์ `.cpy` อื่นเข้ามาประกอบภายในตัวมันเองได้ เหมาะสำหรับกรณีที่มี
"บล็อกข้อมูลย่อย" ที่ใช้ซ้ำได้ในหลาย Record (เช่น บล็อกที่อยู่ ใช้ได้ทั้งในระเบียนลูกค้าและระเบียน
ซัพพลายเออร์)

### Copybook ที่อยู่ (นำไปฝังในหลาย Record ได้)

```cobol
      *> Copybook: ADDR.CPY
      *> Address fields, meant to be COPY'd inside another copybook.
           05  ADDR-LINE1          PIC X(20).
           05  ADDR-CITY           PIC X(15).
```

### Copybook ลูกค้าเต็มรูปแบบ ที่ซ้อน COPY ที่อยู่เข้าไป

```cobol
      *> Copybook: CUSTFULL.CPY
      *> A copybook that itself uses COPY - it nests ADDR.CPY to
      *> pull in address fields as siblings under this record.
       01  CUSTOMER-FULL-RECORD.
           05  CF-ID               PIC X(6).
           05  CF-NAME             PIC X(20).
           COPY "addr.cpy".
```

### โปรแกรมที่ใช้ Copybook ซ้อนกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP329-NESTED-COPY.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> COPY here pulls in CUSTFULL.CPY, which in turn pulls in
      *> ADDR.CPY - the compiler expands both, layer by layer.
           COPY "custfull.cpy".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "C0077 "     TO CF-ID.
           MOVE "SOMCHAI"     TO CF-NAME.
           MOVE "123 MAIN RD" TO ADDR-LINE1.
           MOVE "BANGKOK"     TO ADDR-CITY.
           DISPLAY CF-ID " " CF-NAME " " ADDR-LINE1 " " ADDR-CITY.
           STOP RUN.
```

**ผลลัพธ์:**

```
C0077  SOMCHAI              123 MAIN RD          BANGKOK        
```

### ข้อจำกัดสำคัญ: Level Number ต้องวางแผนให้ถูก (พิสูจน์ด้วย Error จริง)

จุดที่ต้องระวังมากคือ **Level Number** ของฟิลด์ในไฟล์ที่ถูกซ้อนต้องสอดคล้องกับตำแหน่งที่ต้องการ
ใช้งานจริง ลองดูโครงสร้างที่ผิดพลาด (ตั้งใจให้ `ADDR-LINE1`/`ADDR-CITY` เป็น "ลูก" ของกลุ่ม
`CF-ADDRESS` แต่ทำไม่ถูกวิธี):

```cobol
       01  CUSTOMER-FULL-RECORD.
           05  CF-ID               PIC X(6).
           05  CF-NAME             PIC X(20).
           05  CF-ADDRESS.
               COPY "addr.cpy".
```

`addr.cpy` มีฟิลด์ระดับ `05` (`ADDR-LINE1`, `ADDR-CITY`) เมื่อถูกแปะเข้ามาใต้ `05 CF-ADDRESS.`
ฟิลด์เหล่านี้จะกลายเป็น **พี่น้อง (sibling)** ของ `CF-ADDRESS` ไม่ใช่ **ลูก (child)** ของมัน
(เพราะ Level Number เท่ากันคือ `05` ทั้งคู่) ทำให้ `CF-ADDRESS` กลายเป็นกลุ่มที่ไม่มีลูกเลย และไม่มี
`PICTURE` ของตัวเอง คอมไพเลอร์จะฟ้อง Error ทันที:

```
copybooks/custfull_bad.cpy:4: error: PICTURE clause required for 'CF-ADDRESS'
```

### อธิบายจุดสำคัญ

- COPY เป็นเพียงการ "แปะข้อความ" ไม่ได้ปรับ Level Number ให้อัตโนมัติ ผู้เขียน Copybook ต้อง
  ออกแบบ Level Number ของไฟล์ที่จะถูกซ้อนให้เข้ากับบริบทที่ตั้งใจใช้งานเสมอ
- วิธีแก้ที่ถูกต้องสำหรับตัวอย่างนี้ (ตามที่แสดงในหัวข้อก่อนหน้า) คือให้ `COPY "addr.cpy".` อยู่ใน
  ตำแหน่งเดียวกับ `05 CF-ID` และ `05 CF-NAME` โดยตรง (เป็นพี่น้องกันภายใต้ `01
  CUSTOMER-FULL-RECORD` เดียวกัน) แทนที่จะพยายามให้เป็นลูกของกลุ่มย่อยอีกชั้นหนึ่ง

### ข้อควรระวัง

- ก่อนออกแบบ Copybook ที่ตั้งใจให้ถูกซ้อนใน Copybook อื่น ควรทดสอบคอมไพล์จริงเสมอ อย่าคาดเดา
  โครงสร้างเอาเองเพียงจากการอ่านโค้ด เพราะ COBOL Preprocessor (ตัวจัดการ COPY) ทำงาน "ก่อน"
  ตัว Parser วิเคราะห์ Level Number ทำให้ปัญหาบางอย่างมองเห็นได้ยากจากการอ่านไฟล์ `.cpy` เพียง
  ไฟล์เดียวโดด ๆ ต้องดูผลลัพธ์หลังขยาย COPY แล้วเท่านั้นจึงจะเห็นปัญหาชัดเจน

### แบบฝึกหัดที่ 329.1

**โจทย์**: จงอธิบายว่าทำไมโครงสร้างที่ถูกต้อง (`COPY "addr.cpy".` วางเป็นพี่น้องของ `CF-ID` และ
`CF-NAME`) ถึงทำงานได้ ในขณะที่โครงสร้างที่ผิด (วางไว้ใต้ `05 CF-ADDRESS.`) กลับ Error

**เฉลย**: ในโครงสร้างที่ถูกต้อง `ADDR-LINE1` และ `ADDR-CITY` (Level `05`) ถูกวางในตำแหน่งเดียวกับ
`CF-ID` และ `CF-NAME` (Level `05` เช่นกัน) ทั้งหมดจึงเป็นฟิลด์ลูกโดยตรงของ `01
CUSTOMER-FULL-RECORD` ซึ่งถูกต้องตามหลักการซ้อน Level ของ COBOL ส่วนในโครงสร้างที่ผิด ผู้เขียน
ตั้งใจให้ `CF-ADDRESS` (Level `05`) เป็นกลุ่มแม่ที่มี `ADDR-LINE1`/`ADDR-CITY` เป็นลูก แต่เนื่องจาก
ฟิลด์ทั้งสองใน `addr.cpy` มี Level `05` เท่ากับ `CF-ADDRESS` เอง (ไม่ใช่ Level ที่ลึกกว่า เช่น `10`)
COBOL จึงตีความว่าทั้งสามฟิลด์เป็นพี่น้องกัน ทำให้ `CF-ADDRESS` กลายเป็นกลุ่มว่างเปล่าไม่มีลูก และ
เนื่องจากมันไม่มี `PICTURE` ของตัวเองด้วย (เพราะตั้งใจให้เป็นกลุ่ม ไม่ใช่ Elementary Item) คอมไพเลอร์
จึงฟ้อง Error ว่าต้องการ `PICTURE clause` สำหรับมัน

---

## ขั้นตอนที่ 330: แนวปฏิบัติที่ดี สรุปรวม และตัวอย่างระบบเล็ก ๆ ที่ใช้ Copybook ประกอบกัน

### แนวปฏิบัติที่ดีสำหรับการใช้ Copybook (Best Practices)

1. **ตั้งชื่อไฟล์และคอมเมนต์หัวไฟล์ให้ชัดเจนเสมอ** — บอกว่า Copybook นี้คืออะไร ใครใช้บ้าง
2. **แยก Copybook ตามหน้าที่**: โครงสร้าง Record หนึ่งไฟล์ ค่าคงที่อีกไฟล์ ไม่ควรปนกันจนสับสน
3. **เก็บ Copybook ทั้งหมดไว้ในโฟลเดอร์กลาง** (เช่น `copybooks/`) และใช้ `-I` หรือ `COBCPY`
   อ้างอิงเสมอ อย่า COPY ด้วย Path แบบเจาะจงเครื่อง (Absolute Path) เพราะย้ายเครื่องแล้วจะพัง
4. **ทดสอบคอมไพล์ Copybook ใหม่ทุกครั้งกับโปรแกรมจริงอย่างน้อยหนึ่งตัว** ก่อนประกาศว่าใช้งานได้
5. **ควบคุมเวอร์ชันของ Copybook ด้วย Git เหมือนไฟล์ `.cob`** เพราะมันคือส่วนหนึ่งของซอร์สโค้ด
6. **ระวังผลกระทบวงกว้าง**: ก่อนแก้ Copybook ที่มีโปรแกรมจำนวนมาก COPY อยู่ ควรค้นหา (grep) ว่ามี
   โปรแกรมใดบ้างที่ใช้ Copybook นี้ แล้ววางแผน Regression Test ให้ครอบคลุมทุกโปรแกรมนั้น

### ตัวอย่างระบบเล็ก ๆ: ระบบสินค้า ที่ใช้ 2 Copybook ร่วมกันระหว่าง 2 โปรแกรม

Copybook โครงสร้างสินค้า:

```cobol
      *> Copybook: PRODREC.CPY
      *> Product master record, shared between the maintenance
      *> program and the reporting program.
       01  PRODUCT-RECORD.
           05  PROD-CODE           PIC X(6).
           05  PROD-DESC           PIC X(20).
           05  PROD-PRICE          PIC S9(5)V99.
           05  PROD-QTY-ON-HAND    PIC 9(5).
```

โปรแกรมที่ 1 — เพิ่มสินค้าใหม่ (ใช้ทั้ง `prodrec.cpy` สำหรับ FD และ `constants.cpy` จาก
ขั้นตอนที่ 327 สำหรับแสดงรหัสบริษัท):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP330-ADD-PRODUCT.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PRODUCT-FILE ASSIGN TO "PRODUCTS330.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-FILE.
           COPY "prodrec.cpy".

       WORKING-STORAGE SECTION.
           COPY "constants.cpy".

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN OUTPUT PRODUCT-FILE.
           MOVE "P0001 " TO PROD-CODE.
           MOVE "USB CABLE"  TO PROD-DESC.
           MOVE 99.00         TO PROD-PRICE.
           MOVE 500           TO PROD-QTY-ON-HAND.
           WRITE PRODUCT-RECORD.

           MOVE "P0002 " TO PROD-CODE.
           MOVE "WIRELESS MOUSE" TO PROD-DESC.
           MOVE 350.00              TO PROD-PRICE.
           MOVE 120                 TO PROD-QTY-ON-HAND.
           WRITE PRODUCT-RECORD.
           CLOSE PRODUCT-FILE.
           DISPLAY "COMPANY " COMPANY-CODE ": 2 PRODUCTS ADDED".
           STOP RUN.
```

โปรแกรมที่ 2 — รายงานมูลค่าสต๊อก (COPY `prodrec.cpy` ตัวเดิม):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP330-REPORT-PRODUCT.
       AUTHOR. COBOL-COURSE.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT PRODUCT-FILE ASSIGN TO "PRODUCTS330.DAT"
               ORGANIZATION IS LINE SEQUENTIAL.

       DATA DIVISION.
       FILE SECTION.
       FD  PRODUCT-FILE.
      *> Same PRODREC.CPY as the maintenance program - the report
      *> can never read a mismatched layout.
           COPY "prodrec.cpy".

       WORKING-STORAGE SECTION.
       01  WS-EOF              PIC X VALUE "N".
       01  WS-TOTAL-VALUE      PIC S9(9)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN INPUT PRODUCT-FILE.
           PERFORM UNTIL WS-EOF = "Y"
               READ PRODUCT-FILE
                 AT END MOVE "Y" TO WS-EOF
                 NOT AT END
                   DISPLAY PROD-CODE " " PROD-DESC " QTY="
                       PROD-QTY-ON-HAND
                   COMPUTE WS-TOTAL-VALUE = WS-TOTAL-VALUE +
                       PROD-PRICE * PROD-QTY-ON-HAND
               END-READ
           END-PERFORM.
           CLOSE PRODUCT-FILE.
           DISPLAY "TOTAL STOCK VALUE: " WS-TOTAL-VALUE.
           STOP RUN.
```

**ผลลัพธ์ (รัน ADD-PRODUCT ก่อน แล้วรัน REPORT-PRODUCT):**

```
COMPANY TH01: 2 PRODUCTS ADDED
P0001  USB CABLE            QTY=00500
P0002  WIRELESS MOUSE       QTY=00120
TOTAL STOCK VALUE: +000091500.00
```

### อธิบายจุดสำคัญ

- ระบบเล็ก ๆ นี้แสดงให้เห็นการผสมผสาน Copybook สองชนิดที่เรียนมาตลอด Part นี้: `prodrec.cpy`
  (โครงสร้าง FD ที่ใช้ร่วมกันระหว่างโปรแกรมเขียนและโปรแกรมอ่าน จาก ขั้นตอนที่ 325) และ
  `constants.cpy` (ค่าคงที่ระดับองค์กร จาก ขั้นตอนที่ 327) ทำงานร่วมกันในโปรแกรมเดียว
- ทั้งสองโปรแกรมไม่มีทางที่จะเข้าใจโครงสร้างไฟล์ `PRODUCTS330.DAT` ไม่ตรงกัน เพราะทั้งคู่ COPY
  `prodrec.cpy` ไฟล์เดียวกันเสมอ นี่คือรากฐานสำคัญที่ระบบ COBOL ขนาดใหญ่ในองค์กรจริงยึดถือมาตลอด
  หลายสิบปี

### ข้อควรระวังสรุปรวมทั้ง Part

- อย่าลืม `-I` หรือ `COBCPY` เวลาคอมไพล์ มิฉะนั้นจะเจอ Error ต่อเนื่องจำนวนมาก
- Copybook สำหรับ `FD` ต้องตรงกับไฟล์จริงบนดิสก์เสมอ แก้ไขแล้วต้องคอมไพล์ทุกโปรแกรมที่เกี่ยวข้อง
- Level 78 ใช้เก็บค่าคงที่ อ่านได้อย่างเดียว ห้าม `MOVE` ค่าใหม่เข้าไป
- REPLACING ทำงานแบบแทนที่คำ (word-level) ผ่าน Pseudo-text `==...==`
- การซ้อน COPY ต้องวางแผน Level Number ให้ถูกต้อง มิฉะนั้นจะได้ Error ที่อาจดูไม่เกี่ยวกับ COPY
  โดยตรง (เช่น "PICTURE clause required")

### แบบฝึกหัดที่ 330.1

**โจทย์**: จงออกแบบ Copybook ใหม่ชื่อ `EMPRATE.CPY` ที่เก็บ 78-level Constants สำหรับ "อัตรา
ค่าล่วงเวลา" (`OVERTIME-RATE` = 1.5) และ "ชั่วโมงทำงานมาตรฐานต่อวัน" (`STANDARD-HOURS` = 8)
แล้วเขียนโปรแกรมสั้น ๆ ที่ COPY ไฟล์นี้เพื่อคำนวณค่าล่วงเวลาของพนักงานที่ทำงาน 10 ชั่วโมงในวันนั้น
ด้วยค่าแรงต่อชั่วโมง 100 บาท

**เฉลย**:

Copybook `EMPRATE.CPY`:

```cobol
      *> Copybook: EMPRATE.CPY
       78  STANDARD-HOURS          VALUE 8.
       78  OVERTIME-RATE           VALUE 1.5.
```

โปรแกรม:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP330-OT-CALC.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
           COPY "EMPRATE.CPY".
       01  WS-HOURLY-RATE      PIC 9(5)V99 VALUE 100.00.
       01  WS-HOURS-WORKED     PIC 9(3)    VALUE 10.
       01  WS-OT-HOURS         PIC 9(3).
       01  WS-OT-PAY           PIC 9(7)V99.
       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-OT-HOURS =
               WS-HOURS-WORKED - STANDARD-HOURS.
           COMPUTE WS-OT-PAY =
               WS-OT-HOURS * WS-HOURLY-RATE * OVERTIME-RATE.
           DISPLAY "OVERTIME HOURS: " WS-OT-HOURS.
           DISPLAY "OVERTIME PAY: " WS-OT-PAY.
           STOP RUN.
```

ผลลัพธ์ที่คาดไว้: `OVERTIME HOURS: 002` และ `OVERTIME PAY: 0000300.00` (2 ชั่วโมง × 100 บาท ×
1.5 เท่า)

---

## สรุปท้ายบท

ใน Part นี้เราได้เรียนรู้คำสั่ง `COPY` และการใช้งาน Copybook อย่างครบวงจร ได้แก่:

- ปัญหาโค้ดซ้ำซ้อนที่ `COPY` เข้ามาแก้ไข และไวยากรณ์พื้นฐานของ `COPY "ชื่อไฟล์.cpy".`
- โครงสร้างที่ดีของไฟล์ Copybook และการใช้ Copybook เดียวกันซ้ำในหลายโปรแกรมที่มีตรรกะต่างกัน
- การปรับแต่ง Copybook ด้วย `REPLACING` และ Pseudo-text (`==...==`) เพื่อสร้างหลายโครงสร้างจาก
  ไฟล์เดียว
- การใช้ Copybook ใน `FILE SECTION` เพื่อบังคับให้โปรแกรมเขียนและโปรแกรมอ่านไฟล์เข้าใจ Record
  Layout ตรงกันเสมอ
- การฝัง 88-level Condition Names ไว้ใน Copybook เพื่อแชร์ความหมายทางธุรกิจร่วมกัน
- การใช้ Level 78 เก็บค่าคงที่ระดับองค์กรใน Copybook
- การจัดระเบียบ Copybook เป็นคลัง (Library) ด้วยตัวเลือก `-I` และตัวแปรสภาพแวดล้อม `COBCPY`
- การซ้อน COPY (Copybook เรียก Copybook อื่น) และข้อจำกัดเรื่อง Level Number ที่ต้องวางแผนให้ถูก
- แนวปฏิบัติที่ดีสำหรับการดูแล Copybook ในโปรเจกต์ขนาดใหญ่ และตัวอย่างระบบเล็ก ๆ ที่ผสมผสาน
  Copybook หลายชนิดเข้าด้วยกัน

`COPY` เป็นเครื่องมือพื้นฐานที่สุดตัวหนึ่งที่ทำให้ระบบ COBOL ขนาดหลายล้านบรรทัดในองค์กรจริงยังคง
ดูแลรักษาได้ตลอดหลายสิบปี ใน **Part 034** เราจะเรียนรู้อีกวิธีหนึ่งในการจัดโครงสร้างโปรแกรมขนาดใหญ่:
**Nested Programs** — การเขียนหลายโปรแกรมซ้อนกันในไฟล์ซอร์สเดียว พร้อมกฎเรื่อง `END PROGRAM`,
ขอบเขตของข้อมูล (Data Scope), และคำสั่งพิเศษอย่าง `COMMON` และ `INITIAL`

**[กลับไปยัง Part 032: การส่งพารามิเตอร์ →](part-032-parameter-passing.md)**

**[ไปยัง Part 034: Nested Programs และ END PROGRAM →](part-034-nested-programs.md)**
