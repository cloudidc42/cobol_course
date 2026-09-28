# Part 079: Unit Testing สำหรับ COBOL (ขั้นตอนที่ 781–790)

## คำนำของ Part นี้

Part 078 สอนให้เราสร้าง CI pipeline ที่คอมไพล์และเทียบ**ผลลัพธ์ทั้งโปรแกรม**กับค่าที่คาดหวัง
(Golden Output Testing) เทคนิคนี้ใช้งานได้ดีในระดับหนึ่ง แต่มีข้อจำกัดสำคัญ: มันบอกได้แค่ว่า
"โปรแกรมทั้งก้อนถูกหรือผิด" แต่ไม่ได้บอกว่า **"ส่วนไหนของตรรกะที่ผิด"** และยิ่งโปรแกรมใหญ่ขึ้น
การเขียน Golden Output ให้ครอบคลุมทุกกรณี (edge case) ก็ยิ่งยากขึ้นเรื่อย ๆ

**Unit Testing** คือการทดสอบ**หน่วยย่อยที่สุดของตรรกะ** (มักเป็นฟังก์ชันหรือ subprogram หนึ่งตัว)
แยกจากส่วนอื่นของระบบโดยสิ้นเชิง เพื่อพิสูจน์ว่าหน่วยนั้น**ทำงานถูกต้องในทุกกรณีที่ควรจะรองรับ**
ก่อนที่มันจะถูกประกอบรวมกับส่วนอื่น ในภาษาสมัยใหม่มีเฟรมเวิร์กสำเร็จรูปมากมาย (JUnit, pytest,
Jest) แต่ COBOL ไม่มีเฟรมเวิร์กมาตรฐานที่ติดตั้งพร้อมใช้แบบนั้น — Part นี้จะพาคุณสำรวจว่ามีเครื่องมือ
อะไรบ้างจริง ๆ ในสภาพแวดล้อมของเรา แล้วสร้าง**วิธีทดสอบหน่วยย่อยด้วยมือ (hand-rolled)** ที่ใช้งาน
ได้จริงและเป็นรากฐานของเครื่องมือเชิงพาณิชย์ทุกตัวที่มีอยู่ในอุตสาหกรรม

> **หมายเหตุความซื่อตรงเรื่องเครื่องมือ**: ก่อนเขียนเนื้อหานี้ ได้ตรวจสอบจริงด้วย `cobc --help`
> และ `apt list --installed` ในสภาพแวดล้อมที่ใช้เขียนหลักสูตรนี้แล้ว **ไม่มีเฟรมเวิร์ก unit testing
> เฉพาะทางสำหรับ COBOL (เช่น funcUT, MFUnit) ติดตั้งอยู่หรือติดตั้งเพิ่มได้ในสภาพแวดล้อมนี้**
> รายละเอียดและผลการตรวจสอบจริงอยู่ในขั้นตอนที่ 782

---

## ขั้นตอนที่ 781: ทำไม Unit Testing กับ COBOL ถึง "ยาก" กว่าภาษาสมัยใหม่

### ธรรมชาติของ COBOL ที่ทำให้ Unit Testing ท้าทาย

1. **ไม่มีแนวคิด "return value" แบบฟังก์ชัน**: ทบทวนจาก Part 031 — `CALL` ใน COBOL ไม่มีการ
   `return x;` เหมือนภาษาอื่น ผลลัพธ์ต้องส่งผ่านพารามิเตอร์ (`LINKAGE SECTION`) เท่านั้น ทำให้การ
   เขียน assertion (`assert result == expected`) ต้องเขียนเองทุกครั้ง ไม่มี syntax สำเร็จรูปให้ใช้
2. **PROCEDURE DIVISION มักผูกกับ I/O โดยตรง**: โปรแกรม COBOL ดั้งเดิมจำนวนมากเขียนให้ `PERFORM`
   ตรรกะทางธุรกิจปนกับ `DISPLAY`, `ACCEPT`, `READ`/`WRITE` ไฟล์ ในย่อหน้าเดียวกัน ทำให้แยกทดสอบ
   เฉพาะ "ตรรกะ" ออกจาก "การอ่าน/เขียนข้อมูล" ได้ยาก
3. **WORKING-STORAGE คงค่าข้ามการเรียกใช้** (พิสูจน์ในขั้นตอนที่ 815 ของ Part 082 เรื่อง Singleton
   pattern): ถ้า subprogram ใช้ตัวแปรร่วม (shared state) ระหว่างการเรียกหลายครั้ง การทดสอบแต่ละ
   กรณีอาจ**ไม่เป็นอิสระจากกัน** (test isolation) ซึ่งขัดกับหลักการพื้นฐานของ unit test ที่ดี
4. **ไม่มีเฟรมเวิร์กมาตรฐานที่ติดตั้งพร้อมใช้**: ต่างจาก Java ที่มี JUnit ติดมากับเกือบทุกโปรเจกต์
   COBOL ไม่มีเครื่องมือแบบนั้นที่เป็นมาตรฐานอุตสาหกรรมเดียวกันทุกที่

### แนวทางที่หลักสูตรนี้จะใช้แก้ปัญหาทั้ง 4 ข้อ

| ปัญหา | แนวทางแก้ |
|---|---|
| ไม่มี return value | ออกแบบ subprogram ให้ "pure function-like" — รับ input, เขียนผลลัพธ์ลงพารามิเตอร์ output ที่แยกชัดเจน ไม่ผสม I/O |
| ตรรกะปนกับ I/O | แยก subprogram คำนวณล้วน ๆ ออกจากโปรแกรมหลักที่จัดการ DISPLAY/READ/WRITE |
| WORKING-STORAGE ค้างข้ามการเรียก | ใช้ `CANCEL` รีเซ็ตสถานะก่อนแต่ละ test case เมื่อจำเป็น (ทบทวนจาก Part 031 ขั้นตอนที่ 309) |
| ไม่มีเฟรมเวิร์กสำเร็จรูป | เขียน "Test Driver" ด้วย COBOL เองเป็นโปรแกรมแยก ที่ CALL subprogram เป้าหมายแล้ว assert ผลลัพธ์ |

### ข้อควรระวัง

- Unit Testing ไม่ได้มาแทนที่ Golden Output Testing จาก Part 078 — ทั้งสองแนวทางทำงานเสริมกัน:
  Golden Output ทดสอบว่า**ระบบทั้งชิ้นทำงานถูกต้องแบบ end-to-end**, ส่วน Unit Test ทดสอบว่า
  **ส่วนประกอบย่อยแต่ละชิ้นถูกต้องในทุกกรณี**ก่อนที่จะนำมาประกอบกัน
- อย่าพยายาม "unit test" โปรแกรมที่ยังผูกตรรกะกับ I/O แน่นเกินไป — ควร**ปรับโครงสร้างโปรแกรม
  (refactor) ให้แยกส่วนออกมาก่อน** ซึ่งเป็นหัวข้อที่ Part 081 จะสอนเจาะลึก

### แบบฝึกหัดที่ 781.1

**โจทย์**: จงอธิบายว่าทำไมการที่ COBOL "ไม่มีแนวคิด return value" ถึงทำให้การออกแบบ subprogram
เพื่อการทดสอบ (testable design) มีความสำคัญมากเป็นพิเศษ เมื่อเทียบกับภาษาที่มี return value

**เฉลย**: ในภาษาที่มี return value เช่น Python (`def calc_tax(amount): return amount * 0.07`)
ฟังก์ชันหนึ่งตัวมักถูกออกแบบให้ "รับ input คืน output" เป็นธรรมชาติอยู่แล้ว ทำให้เขียน unit test
ตรงไปตรงมา (`assert calc_tax(1000) == 70`) แต่ใน COBOL ถ้าโปรแกรมเมอร์ไม่ตั้งใจออกแบบ subprogram
ให้แยกส่วน "คำนวณ" ออกจากส่วน "แสดงผล/บันทึกไฟล์" อย่างชัดเจนตั้งแต่ต้น subprogram นั้นอาจ
`DISPLAY` ผลลัพธ์ออกทางหน้าจอโดยตรงแทนที่จะส่งผ่านพารามิเตอร์ ทำให้ test driver ไม่มีทางอ่านค่า
ผลลัพธ์กลับมาเปรียบเทียบได้เลยนอกจาก capture หน้าจอ (เหมือนที่ Part 078 ทำกับทั้งโปรแกรม) ซึ่งหยาบ
กว่าการ assert ค่าตัวเลขตรง ๆ มาก ดังนั้นวินัยในการออกแบบ "pure function-like subprogram" (รับ
input ผ่าน `LINKAGE SECTION`, ส่ง output ผ่าน `LINKAGE SECTION` เท่านั้น ไม่มี I/O แทรก) จึงเป็น
เงื่อนไขเบื้องต้นที่ COBOL ต้องการเป็นพิเศษก่อนจะทดสอบระดับหน่วยย่อยได้อย่างมีประสิทธิภาพ

---

## ขั้นตอนที่ 782: สำรวจเครื่องมือ Unit Testing ที่มีจริงในสภาพแวดล้อมของเรา

### ตรวจสอบ cobc ว่ามี flag เกี่ยวกับ Unit Test หรือไม่

รันคำสั่งตรวจสอบจริงในสภาพแวดล้อมที่ใช้เขียนหลักสูตรนี้:

```bash
cobc --help | grep -i test
```

**ผลลัพธ์จริง:**

```
  -fhostsign             allow hexadecimal value 'F' for NUMERIC test of signed PACKED DECIMAL field
```

จะเห็นว่า**ไม่มี flag ใดของ `cobc` ที่เกี่ยวข้องกับการ "generate unit test" จริง** — คำว่า "test"
ที่เจอเป็นเพียงส่วนหนึ่งของคำอธิบาย flag `-fhostsign` ที่เกี่ยวกับการตรวจสอบเลขฐานสิบหก ไม่ใช่
เครื่องมือทดสอบแต่อย่างใด

นอกจากนี้ยังตรวจสอบ flag ที่มีชื่อคล้ายกับ "generate C declaration for static call"
(`-fgen-c-decl-static-call` / `-fno-gen-c-decl-static-call`) ซึ่งพบจริงใน `cobc --help` แต่ flag นี้
**ไม่เกี่ยวข้องกับการทดสอบเลย** — มันควบคุมว่า compiler จะ generate function prototype ภาษา C
สำหรับการ `CALL` แบบ static หรือไม่ (เป็นรายละเอียดภายในของการแปลง COBOL เป็น C code ที่ GnuCOBOL
ใช้ ไม่ใช่ฟีเจอร์สร้าง test case ให้อัตโนมัติแต่อย่างใด)

### ตรวจสอบเครื่องมือ Unit Testing เฉพาะทางที่อาจติดตั้งไว้

```bash
which cobcrun cob2unit funcut mfunit
apt list --installed | grep -i cobol
```

**ผลลัพธ์จริง:**

```
/usr/bin/cobcrun
gnucobol4/noble,now 4.0~early~20200606-6.1build1 amd64 [installed]
```

มีเพียง `cobcrun` (เครื่องมือรันโปรแกรม COBOL module แบบ dynamic ที่มากับ GnuCOBOL เอง ไม่ใช่
test framework) และแพ็กเกจ `gnucobol4` เท่านั้นที่ติดตั้งอยู่ **ไม่พบ `funcUT`, `MFUnit`, หรือ
เครื่องมือ unit test เฉพาะทางอื่นใดติดตั้งอยู่ในสภาพแวดล้อมนี้เลย**

### ทำไม funcUT และ MFUnit ถึงไม่ได้ถูกใช้ในหลักสูตรนี้

- **MFUnit**: เป็นเฟรมเวิร์ก unit test ที่พัฒนาโดย Micro Focus (ปัจจุบันคือ OpenText) ผูกกับผลิตภัณฑ์
  Micro Focus COBOL/Visual COBOL โดยเฉพาะ ซึ่งเป็นซอฟต์แวร์เชิงพาณิชย์ที่ต้องมี license — ไม่สามารถ
  ติดตั้งหรือใช้งานร่วมกับ GnuCOBOL แบบ Open Source ที่หลักสูตรนี้ใช้เป็นหลักได้โดยตรง
- **funcUT**: เป็นโปรเจกต์ที่เคยมีการพูดถึงในชุมชน GnuCOBOL สำหรับการสร้าง unit test แบบ Assertion
  แต่ ณ เวลาที่เขียนหลักสูตรนี้ ไม่มีแพ็กเกจที่ติดตั้งผ่าน `apt`/`pip` ได้ในสภาพแวดล้อมมาตรฐาน และ
  ไม่ได้เป็นส่วนหนึ่งของ GnuCOBOL หลักที่ติดตั้งมาด้วย `cobc`
- **ในทางปฏิบัติของอุตสาหกรรมจริง**: ทีม COBOL จำนวนมาก (โดยเฉพาะทีมที่ใช้ GnuCOBOL แบบ Open
  Source) เขียน **Test Driver ด้วย COBOL เอง** เหมือนที่ Part นี้จะสอน เพราะไม่ต้องพึ่งพา license
  หรือเครื่องมือภายนอกใด ๆ เลย เพียงแค่ `cobc` ตัวเดียวก็เพียงพอ — นี่คือเหตุผลที่แนวทางนี้ถูกเลือก
  เป็นแนวทางหลักของหลักสูตร

### ข้อควรระวัง

- ถ้าคุณไปทำงานในองค์กรที่มี license ของ Micro Focus/Visual COBOL อยู่แล้ว ควรศึกษา MFUnit เพิ่มเติม
  เพราะมันมีฟีเจอร์ครบครันกว่าการเขียน Test Driver ด้วยมือมาก (เช่น การสร้างรายงาน HTML อัตโนมัติ)
  — หลักสูตรนี้สอนแนวคิดพื้นฐานที่จะทำให้คุณเข้าใจ MFUnit ได้เร็วขึ้นถ้าต้องใช้งานจริงในอนาคต
- อย่าเข้าใจผิดว่า "ไม่มีเฟรมเวิร์กสำเร็จรูป = COBOL ทดสอบไม่ได้" — Test Driver ที่เขียนด้วย COBOL
  เอง (ตามที่จะสอนในขั้นตอนถัดไป) ทำหน้าที่เดียวกันทุกประการกับ JUnit/pytest เพียงแต่ต้องเขียน
  โครงสร้าง assertion เองแทนที่จะมี `assertEqual()` สำเร็จรูปให้เรียกใช้

### แบบฝึกหัดที่ 782.1

**โจทย์**: จงอธิบายว่าทำไมการตรวจสอบเครื่องมือที่มีจริงในสภาพแวดล้อม (เช่นที่ทำในขั้นตอนนี้) ควรเป็น
ขั้นตอนแรกเสมอก่อนเริ่มออกแบบกลยุทธ์การทดสอบของโปรเจกต์จริง

**เฉลย**: การสันนิษฐานว่ามีเครื่องมือบางอย่างติดตั้งอยู่โดยไม่ตรวจสอบก่อนอาจนำไปสู่การออกแบบ
pipeline ทั้งหมดโดยอ้างอิงเครื่องมือที่ไม่มีอยู่จริงในสภาพแวดล้อม production หรือ CI ขององค์กร
ทำให้ต้องมารื้อออกแบบใหม่ทีหลังเมื่อพบว่าเครื่องมือนั้นติดตั้งไม่ได้ (เช่น ติดปัญหาเรื่อง license
หรือ compatibility กับเวอร์ชัน compiler ที่ใช้งานจริง) การตรวจสอบด้วยคำสั่งง่าย ๆ อย่าง `cobc --help`,
`which`, และ `apt list --installed` ก่อนเสมอ ทำให้ทีมออกแบบกลยุทธ์การทดสอบบนพื้นฐานของสิ่งที่**มี
อยู่จริง**แทนที่จะเป็นสิ่งที่คาดหวังว่าจะมี ซึ่งเป็นหลักการเดียวกับที่วิศวกรซอฟต์แวร์ที่ดีควรทำก่อน
ตัดสินใจเลือกเครื่องมือใด ๆ เสมอ

---

## ขั้นตอนที่ 783: ออกแบบ Subprogram แบบ "Pure Function-like" ให้ทดสอบง่าย

### หลักการออกแบบ

Subprogram ที่ทดสอบง่ายที่สุดคือ subprogram ที่:

1. รับค่า input ทั้งหมดผ่าน `LINKAGE SECTION` เท่านั้น (ไม่อ่านค่าจากไฟล์หรือ `ACCEPT` จากผู้ใช้)
2. เขียนผลลัพธ์ทั้งหมดกลับผ่านพารามิเตอร์ output เท่านั้น (ไม่ `DISPLAY` ผลลัพธ์เอง)
3. ไม่มี side effect อื่นใด (ไม่เปิด/ปิดไฟล์ ไม่แก้ไขตัวแปร global ที่นอกเหนือจากพารามิเตอร์)
4. ใช้ `RETURN-CODE` special register (ทบทวนจาก Part 031 ขั้นตอนที่ 308) เพื่อรายงานสถานะสำเร็จ/
   ล้มเหลว แทนการพิมพ์ error message เอง

### ตัวอย่าง: TAXCALC — subprogram คำนวณภาษีแบบ pure function-like

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TAXCALC.
       AUTHOR. COBOL-COURSE.

      *> TAXCALC is written to be "pure function-like": given an
      *> amount, it computes tax with no other side effects and no
      *> file/screen I/O of its own. That makes it easy to test in
      *> isolation with a driver program, unlike a paragraph buried
      *> inside a large PROCEDURE DIVISION full of DISPLAY/ACCEPT.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-AMOUNT                PIC S9(7)V99.
       01  LK-TAX                   PIC S9(7)V99.

       PROCEDURE DIVISION USING LK-AMOUNT LK-TAX.
       MAIN-PARA.
           IF LK-AMOUNT < 0
               MOVE 0 TO LK-TAX
               MOVE 1 TO RETURN-CODE
           ELSE
               COMPUTE LK-TAX ROUNDED = LK-AMOUNT * 0.07
               MOVE 0 TO RETURN-CODE
           END-IF
           GOBACK.
```

### อธิบายจุดสำคัญ

- **ไม่มี `DISPLAY` แม้แต่บรรทัดเดียวในโปรแกรมนี้** — นี่คือลักษณะสำคัญที่สุดของการออกแบบแบบ pure
  function-like: subprogram นี้**ไม่สนใจเลยว่าใครเป็นผู้เรียก** อาจเป็นโปรแกรมหลักที่ทำงานจริง
  (production) หรือ test driver ก็ตาม มันทำหน้าที่เดียวคือคำนวณและคืนผลลัพธ์เท่านั้น
- `PIC S9(7)V99` (มี `S` นำหน้า = Signed): เลือกใช้ signed field เพราะเราต้องการรองรับกรณี
  `LK-AMOUNT` เป็นค่าลบเพื่อทดสอบเงื่อนไข validation (`IF LK-AMOUNT < 0`) — ถ้าใช้ unsigned field
  (`PIC 9(7)V99` ธรรมดา) COBOL จะไม่สามารถเก็บค่าลบได้เลยตั้งแต่แรก ทำให้ทดสอบกรณีนี้ไม่ได้
- `MOVE 1 TO RETURN-CODE` เมื่อ input ไม่ถูกต้อง: เป็นวิธีมาตรฐานที่ subprogram ใน COBOL ใช้
  "ส่งสัญญาณ error" กลับไปให้ผู้เรียกโดยไม่ต้องพึ่งพา exception handling แบบภาษาสมัยใหม่ (COBOL
  ไม่มี try/catch มาตรฐาน) — ผู้เรียกสามารถตรวจสอบค่า `RETURN-CODE` ได้ทันทีหลัง `CALL` กลับมา

### ข้อควรระวัง

- การออกแบบแบบนี้หมายความว่า**การตรวจสอบ input ที่ไม่ถูกต้อง (validation) เป็นความรับผิดชอบของ
  ตัว subprogram เอง** ไม่ใช่ผู้เรียก — นี่คือหลักการเดียวกับ "Defensive Programming" ในภาษาสมัยใหม่
- ระวังอย่าให้ subprogram แบบนี้ "ลืม" ตั้งค่า `RETURN-CODE` ในบาง path ของ `IF`/`EVALUATE` เพราะ
  ค่าที่ค้างอยู่จากการเรียกครั้งก่อนหน้าอาจรั่วไหลมาถึงผู้เรียกโดยไม่ตั้งใจ (ปัญหานี้เกี่ยวข้องกับ
  ธรรมชาติที่ WORKING-STORAGE คงค่าข้ามการเรียก ตามที่กล่าวในขั้นตอนที่ 781) — ในตัวอย่างข้างบน
  ทั้งสอง branch ของ `IF` ตั้งค่า `RETURN-CODE` ครบทุกกรณีโดยตั้งใจเพื่อป้องกันปัญหานี้

### แบบฝึกหัดที่ 783.1

**โจทย์**: จงอธิบายว่าทำไมการที่ `TAXCALC` ไม่มี `DISPLAY` เลยแม้แต่บรรทัดเดียว ถึงทำให้มันสามารถ
ถูกเรียกใช้ได้ทั้งจากโปรแกรม production จริง และจาก test driver โดยไม่ต้องแก้ไขโค้ดของ `TAXCALC`
เองเลย

**เฉลย**: เพราะ `TAXCALC` สื่อสารกับโลกภายนอกผ่านพารามิเตอร์ (`LK-AMOUNT` เข้า, `LK-TAX` ออก,
`RETURN-CODE` ออก) เท่านั้น ไม่ได้ผูกติดกับช่องทาง I/O เฉพาะเจาะจงใด ๆ (เช่นหน้าจอ terminal) ทำให้
ผู้เรียก**ไม่ว่าจะเป็นใคร**ก็สามารถอ่านผลลัพธ์จากพารามิเตอร์ที่ตัวเองส่งเข้าไปได้เหมือนกันทุก
ประการ — โปรแกรม production จริงอาจนำ `LK-TAX` ไปแสดงผลบนรายงาน หรือบันทึกลงไฟล์ ในขณะที่ test
driver นำ `LK-TAX` ไปเทียบกับค่าที่คาดหวังด้วย `IF` เอง ทั้งสองกรณีเรียก `TAXCALC` ด้วยวิธีเดียวกัน
เป๊ะ (`CALL "TAXCALC" USING ...`) และ `TAXCALC` เองไม่มีทางรู้เลยว่าใครเป็นผู้เรียกอยู่ นี่คือหัวใจ
ของการออกแบบที่แยก "ตรรกะ" ออกจาก "การนำผลลัพธ์ไปใช้งานต่อ" อย่างสมบูรณ์

---

## ขั้นตอนที่ 784: เขียน Test Driver — CALL หลายกรณีพร้อม Assert

### โครงสร้างของ Test Driver

Test Driver คือโปรแกรม COBOL ธรรมดาที่ทำหน้าที่:

1. ตั้งค่า input ของแต่ละกรณีทดสอบ (test case)
2. `CALL` subprogram เป้าหมายด้วย input นั้น
3. เปรียบเทียบผลลัพธ์ที่ได้กับค่าที่คาดหวัง (expected)
4. พิมพ์ `[PASS]` หรือ `[FAIL]` พร้อมนับจำนวนรวม

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TESTDRIVER.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TEST-NAME             PIC X(40).
       01  WS-TEST-AMOUNT           PIC S9(7)V99.
       01  WS-TEST-TAX              PIC S9(7)V99.
       01  WS-EXPECTED-TAX          PIC S9(7)V99.
       01  WS-EXPECTED-RC           PIC 9(2).
       01  WS-PASS-COUNT            PIC 9(3) VALUE 0.
       01  WS-FAIL-COUNT            PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "==== TAXCALC unit test suite ====".
           PERFORM TEST-NORMAL-AMOUNT.
           PERFORM TEST-ROUNDING-CASE.
           PERFORM TEST-ZERO-AMOUNT.
           PERFORM TEST-NEGATIVE-AMOUNT.
           DISPLAY "==================================".
           DISPLAY "TOTAL PASS=" WS-PASS-COUNT " FAIL=" WS-FAIL-COUNT.
           IF WS-FAIL-COUNT > 0
               MOVE 1 TO RETURN-CODE
           ELSE
               MOVE 0 TO RETURN-CODE
           END-IF
           STOP RUN.

       TEST-NORMAL-AMOUNT.
           MOVE "tax on 1000.00 -> 70.00" TO WS-TEST-NAME.
           MOVE 1000.00 TO WS-TEST-AMOUNT.
           MOVE 70.00 TO WS-EXPECTED-TAX.
           MOVE 0 TO WS-EXPECTED-RC.
           CALL "TAXCALC" USING WS-TEST-AMOUNT WS-TEST-TAX.
           PERFORM ASSERT-TAX-AND-RC.

       TEST-ROUNDING-CASE.
           MOVE "tax on 2500.50 -> 175.04 (rounded)" TO WS-TEST-NAME.
           MOVE 2500.50 TO WS-TEST-AMOUNT.
           MOVE 175.04 TO WS-EXPECTED-TAX.
           MOVE 0 TO WS-EXPECTED-RC.
           CALL "TAXCALC" USING WS-TEST-AMOUNT WS-TEST-TAX.
           PERFORM ASSERT-TAX-AND-RC.

       TEST-ZERO-AMOUNT.
           MOVE "tax on 0.00 -> 0.00" TO WS-TEST-NAME.
           MOVE 0.00 TO WS-TEST-AMOUNT.
           MOVE 0.00 TO WS-EXPECTED-TAX.
           MOVE 0 TO WS-EXPECTED-RC.
           CALL "TAXCALC" USING WS-TEST-AMOUNT WS-TEST-TAX.
           PERFORM ASSERT-TAX-AND-RC.

       TEST-NEGATIVE-AMOUNT.
           MOVE "negative amount -> rejected (RC=1)" TO WS-TEST-NAME.
           MOVE -500.00 TO WS-TEST-AMOUNT.
           MOVE 0.00 TO WS-EXPECTED-TAX.
           MOVE 1 TO WS-EXPECTED-RC.
           CALL "TAXCALC" USING WS-TEST-AMOUNT WS-TEST-TAX.
           PERFORM ASSERT-TAX-AND-RC.

       ASSERT-TAX-AND-RC.
           IF WS-TEST-TAX = WS-EXPECTED-TAX
               AND RETURN-CODE = WS-EXPECTED-RC
               DISPLAY "[PASS] " WS-TEST-NAME
               ADD 1 TO WS-PASS-COUNT
           ELSE
               DISPLAY "[FAIL] " WS-TEST-NAME
               DISPLAY "       expected tax=" WS-EXPECTED-TAX
                   " actual tax=" WS-TEST-TAX
               DISPLAY "       expected rc=" WS-EXPECTED-RC
                   " actual rc=" RETURN-CODE
               ADD 1 TO WS-FAIL-COUNT
           END-IF.
```

**คอมไพล์ (รวม testdriver.cob กับ taxcalc.cob):**

```bash
cobc -x -o testdriver testdriver.cob taxcalc.cob
```

### อธิบายจุดสำคัญ

- แต่ละ `TEST-xxx` paragraph ทำ 4 อย่างเสมอ: ตั้งชื่อ test (`WS-TEST-NAME`), ตั้งค่า input, ตั้งค่า
  expected, แล้ว `CALL` — รูปแบบนี้เรียกว่า **AAA Pattern (Arrange-Act-Assert)** ซึ่งเป็นโครงสร้าง
  มาตรฐานของ unit test ในทุกภาษา เพียงแต่ใน COBOL เราต้องเขียนด้วยมือเองผ่าน paragraph แยกกัน
- `ASSERT-TAX-AND-RC` เป็น **paragraph ตัวช่วย (helper)** ที่ถูกเรียกซ้ำจากทุก test case — ลดความ
  ซ้ำซ้อนของโค้ดเปรียบเทียบผลลัพธ์ (หลักการนี้จะขยายความเพิ่มเติมใน Part 081 เรื่อง Clean Code)
- `RETURN-CODE = WS-EXPECTED-RC` ใน `IF`: เราสามารถเปรียบเทียบ special register `RETURN-CODE`
  ได้เหมือนตัวแปรธรรมดาทันทีหลัง `CALL` กลับมา เพราะ COBOL เก็บค่าที่ subprogram ตั้งไว้ล่าสุดไว้ให้
  ผู้เรียกอ่านได้เสมอ

### ผลลัพธ์จริงเมื่อรัน (ทุก test case ผ่าน)

```
==== TAXCALC unit test suite ====
[PASS] tax on 1000.00 -> 70.00
[PASS] tax on 2500.50 -> 175.04 (rounded)
[PASS] tax on 0.00 -> 0.00
[PASS] negative amount -> rejected (RC=1)
==================================
TOTAL PASS=004 FAIL=000
```

(exit code ที่ได้คือ `0` เพราะ `WS-FAIL-COUNT` เท่ากับ 0)

### ข้อควรระวัง

- **ระวังความกว้างของ `PIC X` สำหรับชื่อ test**: ถ้ากำหนด `WS-TEST-NAME` แคบเกินไป (เช่น `PIC
  X(30)`) ชื่อ test ที่ยาวกว่านั้นจะถูก**ตัดท้ายอย่างเงียบ ๆ** โดยไม่มี error หรือ warning ใด ๆ เลย
  (เหมือนปัญหาที่ Part 031 ขั้นตอนที่ 304 เคยพิสูจน์กับพารามิเตอร์) ทำให้ข้อความสรุปผลอ่านไม่ครบ —
  ควรกำหนดความกว้างให้มากพอสำหรับข้อความ test ที่ยาวที่สุดเสมอ (ในตัวอย่างนี้ใช้ `PIC X(40)`)
- Test driver ต้องคอมไพล์รวมกับ subprogram เป้าหมายเสมอ (เหมือนโปรแกรมหลัก+ย่อยทั่วไปจาก Part 031)
  — ถ้าลืมใส่ `taxcalc.cob` ในคำสั่ง `cobc` จะได้ linker error ทันที

### แบบฝึกหัดที่ 784.1

**โจทย์**: จงอธิบายว่าทำไมการรวม assertion ของ `WS-TEST-TAX` และ `RETURN-CODE` ไว้ใน `IF` เดียว
(`IF WS-TEST-TAX = WS-EXPECTED-TAX AND RETURN-CODE = WS-EXPECTED-RC`) อาจทำให้ debug ยากกว่าการแยก
ตรวจสอบทีละเงื่อนไข ในกรณีที่ test ล้มเหลว

**เฉลย**: เมื่อ `IF` แบบรวมเงื่อนไขด้วย `AND` ล้มเหลว เราจะรู้แค่ว่า "test นี้ fail" แต่ไม่รู้ทันที
ว่า fail เพราะค่า tax ผิด, RETURN-CODE ผิด, หรือผิดทั้งคู่ ต้องอ่านข้อความ debug เพิ่มเติม (ซึ่ง
โค้ดตัวอย่างนี้แก้ปัญหาบางส่วนด้วยการพิมพ์ทั้งค่า tax และ RETURN-CODE ที่คาดหวัง/ได้จริงออกมาใน
branch `ELSE` เสมอ ไม่ว่าจะผิดจากเงื่อนไขไหน) วิธีที่ละเอียดกว่านี้คือแยกตรวจสอบทีละเงื่อนไขด้วย
`IF` สองอันแยกกัน แล้วรายงานผลแยกกันชัดเจนว่า "tax ไม่ตรง" หรือ "RETURN-CODE ไม่ตรง" ซึ่งจะทำให้
debug เร็วขึ้นในกรณีที่มี test case จำนวนมากและหลาย field ต้องตรวจสอบพร้อมกัน — นี่คือ trade-off
ระหว่างความกระชับของโค้ดกับความละเอียดของข้อความ error ที่ทีมต้องตัดสินใจเองตามความเหมาะสม

---

## ขั้นตอนที่ 785: รันชุดทดสอบจริง — ยืนยันผล PASS ทั้งหมด

### รันจริงและตรวจสอบ exit code

```bash
cobc -x -o testdriver testdriver.cob taxcalc.cob
./testdriver
echo "Exit code: $?"
```

**ผลลัพธ์จริง (คัดลอกจากการรันจริงทุกตัวอักษร):**

```
==== TAXCALC unit test suite ====
[PASS] tax on 1000.00 -> 70.00
[PASS] tax on 2500.50 -> 175.04 (rounded)
[PASS] tax on 0.00 -> 0.00
[PASS] negative amount -> rejected (RC=1)
==================================
TOTAL PASS=004 FAIL=000
Exit code: 0
```

### วิเคราะห์แต่ละกรณีทดสอบว่าครอบคลุมอะไรบ้าง

| Test Case | สิ่งที่ทดสอบ | ทำไมสำคัญ |
|---|---|---|
| tax on 1000.00 → 70.00 | เส้นทาง (path) ปกติของการคำนวณ | Happy path พื้นฐานที่สุด |
| tax on 2500.50 → 175.04 | การปัดเศษด้วย `ROUNDED` | 2500.50 × 0.07 = 175.035 พอดี ซึ่งเป็นค่า "กึ่งกลาง" ที่พิสูจน์ว่ากฎการปัดเศษทำงานถูกต้อง |
| tax on 0.00 → 0.00 | ค่าขอบเขตต่ำสุด (boundary: zero) | ค่า 0 มักเป็นจุดที่โปรแกรมเมอร์ลืมพิจารณา |
| negative amount → RC=1 | เส้นทาง error/validation | พิสูจน์ว่า subprogram ปฏิเสธ input ที่ไม่ถูกต้องอย่างถูกต้อง |

การที่ทั้ง 4 กรณีนี้ผ่านหมด **ไม่ได้แปลว่าโปรแกรมไม่มีบั๊กเลย** (unit test พิสูจน์ได้แค่ว่ากรณีที่
เขียนไว้ทำงานถูกต้อง ไม่ใช่ทุกกรณีที่เป็นไปได้ทั้งหมด) แต่มันให้ **ความมั่นใจในระดับสูงมาก** ว่า
เส้นทางหลักทั้งหมดของตรรกะทำงานตามที่ตั้งใจไว้

### ข้อควรระวัง

- อย่าหยุดแค่ "test ผ่านหมดแล้ว = จบ" — คำถามที่ควรถามต่อเสมอคือ **"มีกรณีไหนที่ยังไม่ได้ทดสอบอีก
  หรือไม่"** (ขั้นตอนที่ 788 จะสอนวิธีคิดเรื่อง edge case เพิ่มเติมอย่างเป็นระบบ)
- จำนวน `[PASS]` ที่มากไม่ได้แปลว่าคุณภาพของ test ดี — test 4 กรณีที่ครอบคลุม edge case สำคัญ
  มีค่ามากกว่า test 40 กรณีที่ทดสอบแต่ happy path ซ้ำ ๆ กันในรูปแบบเดิม

### แบบฝึกหัดที่ 785.1

**โจทย์**: จงอธิบายว่าทำไม test case "tax on 2500.50 → 175.04" ถึงมีค่ามากกว่า test case ธรรมดาที่
ใช้ตัวเลขกลม ๆ อย่าง "tax on 1000.00 → 70.00" ในแง่ของการจับบั๊ก

**เฉลย**: 2500.50 × 0.07 = 175.035 พอดี ซึ่งเป็นค่าที่ตัวเลขหลังจุดทศนิยมตำแหน่งที่ 3 คือ `5`
พอดี — นี่คือ**จุดกึ่งกลางของการปัดเศษ (rounding boundary)** ที่ทดสอบว่ากฎ "ปัดขึ้นเมื่อหลักถัดไป
คือ 5 ขึ้นไป" (ตามที่ Part 009 สอนเรื่องเลขคณิตพื้นฐาน) ทำงานถูกต้องจริงหรือไม่ ถ้า subprogram มี
บั๊กเกี่ยวกับการปัดเศษ (เช่น ลืมใส่ `ROUNDED` หรือใส่ผิดตำแหน่ง) test case แบบตัวเลขกลม ๆ อย่าง
1000.00 × 0.07 = 70.00 พอดี (ไม่มีเศษให้ปัดเลย) จะไม่มีทางจับบั๊กนี้ได้เลย เพราะผลลัพธ์จะถูกต้อง
ไม่ว่าจะมี `ROUNDED` หรือไม่ก็ตาม การเลือก test case ที่ตั้งใจ "เข้าใกล้จุดที่พฤติกรรมอาจเปลี่ยน"
แบบนี้คือทักษะสำคัญของการออกแบบชุดทดสอบที่ดี

---

## ขั้นตอนที่ 786: พิสูจน์ Test Driver จริง — จำลองบั๊กแล้วดู FAIL จริง

### จงใจใส่บั๊กเข้าไปใน TAXCALC

สมมติมีคนแก้ไขอัตราภาษีผิดพลาด (เปลี่ยนจาก 0.07 เป็น 0.10 โดยไม่ตั้งใจ หรือเข้าใจผิดว่าอัตราภาษี
เปลี่ยนแล้ว):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TAXCALCBUGGY.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-AMOUNT                PIC S9(7)V99.
       01  LK-TAX                   PIC S9(7)V99.

       PROCEDURE DIVISION USING LK-AMOUNT LK-TAX.
       MAIN-PARA.
           IF LK-AMOUNT < 0
               MOVE 0 TO LK-TAX
               MOVE 1 TO RETURN-CODE
           ELSE
               COMPUTE LK-TAX ROUNDED = LK-AMOUNT * 0.10
               MOVE 0 TO RETURN-CODE
           END-IF
           GOBACK.
```

(สังเกตว่ามีการเปลี่ยน `PROGRAM-ID` เป็น `TAXCALCBUGGY` และแก้ test driver ให้ `CALL
"TAXCALCBUGGY"` แทน เพื่อไม่ให้กระทบไฟล์เดิมที่ถูกต้องอยู่แล้ว)

**คอมไพล์และรันจริง:**

```bash
cobc -x -o testdriver_buggy testdriver_buggy.cob taxcalc_buggy.cob
./testdriver_buggy
echo "Exit code: $?"
```

**ผลลัพธ์จริง (ยืนยันด้วยการรัน — FAIL จริง 2 จาก 4 กรณี):**

```
==== TAXCALC unit test suite ====
[FAIL] tax on 1000.00 -> 70.00
       expected tax=+0000070.00 actual tax=+0000100.00
       expected rc=00 actual rc=+000000000
[FAIL] tax on 2500.50 -> 175.04 (rounded)
       expected tax=+0000175.04 actual tax=+0000250.05
       expected rc=00 actual rc=+000000000
[PASS] tax on 0.00 -> 0.00
[PASS] negative amount -> rejected (RC=1)
==================================
TOTAL PASS=002 FAIL=002
Exit code: 1
```

### วิเคราะห์ผลลัพธ์

- **สองกรณีแรกล้มเหลวทันที** เพราะ 1000.00 × 0.10 = 100.00 (ไม่ใช่ 70.00) และ 2500.50 × 0.10 =
  250.05 (ไม่ใช่ 175.04) — test driver จับบั๊กนี้ได้ทันทีและชัดเจน พร้อมแสดงค่าที่คาดหวังกับค่าที่
  ได้จริงเทียบกันให้เห็นตรง ๆ
- **สองกรณีหลังยังผ่านอยู่** เพราะ `TEST-ZERO-AMOUNT` (0.00 × 0.10 ยังคงเป็น 0.00) และ
  `TEST-NEGATIVE-AMOUNT` (ตรวจสอบแค่ path ของ validation ที่ไม่เกี่ยวกับอัตราภาษีเลย) ไม่ถูกกระทบ
  จากบั๊กนี้ — นี่แสดงให้เห็นว่า **การมี test case หลากหลายช่วยระบุได้ชัดเจนว่าบั๊กกระทบส่วนไหนบ้าง**
  แทนที่จะรู้แค่ว่า "มีอะไรผิดพลาดสักที่" แบบ Golden Output Testing ทั้งโปรแกรม
- สังเกตรูปแบบการแสดงผล `RETURN-CODE`: `expected rc=00` (จาก `WS-EXPECTED-RC` ที่เป็น `PIC 9(2)`)
  เทียบกับ `actual rc=+000000000` (จาก special register `RETURN-CODE` ที่ GnuCOBOL กำหนด PICTURE
  ภายในเป็นเลขมีเครื่องหมายความกว้างต่างไปจากที่เราประกาศเอง) — ทั้งสองค่านี้**เท่ากันในทางตรรกะ**
  (ศูนย์ทั้งคู่) ที่ทำให้ `IF ... RETURN-CODE = WS-EXPECTED-RC` ยังคงเปรียบเทียบถูกต้อง แม้รูปแบบ
  การแสดงผลจะดูต่างกันก็ตาม

### ข้อควรระวัง

- **รูปแบบการแสดงผลของ `RETURN-CODE` เป็นรายละเอียดที่ขึ้นกับ compiler แต่ละตัว** (implementation-
  defined) — ในตัวอย่างนี้ GnuCOBOL แสดงเป็นเลขมีเครื่องหมายความกว้าง 9 หลัก แม้เราจะไม่เคยประกาศ
  ขนาดของมันเองเลยก็ตาม อย่าเขียนโค้ดที่พึ่งพารูปแบบการแสดงผลที่แน่นอนของ `RETURN-CODE` ข้าม
  compiler ต่างยี่ห้อกัน (ควร `MOVE RETURN-CODE TO` ตัวแปรที่มี PICTURE ชัดเจนของเราเองก่อนแสดงผล
  ถ้าต้องการรูปแบบที่แน่นอน)
- การทดสอบนี้พิสูจน์ประเด็นสำคัญจาก Part 078 อีกครั้ง: **บั๊กแบบนี้ (เปลี่ยนค่าคงที่ผิด) จะไม่มีวัน
  ถูก `cobc` จับได้เลย** เพราะ `0.10` เป็นค่าตัวเลขที่ถูกต้องตามไวยากรณ์ทุกประการ — มีแค่ automated
  test เท่านั้นที่จับได้

### แบบฝึกหัดที่ 786.1

**โจทย์**: จงอธิบายว่าทำไม test case "negative amount → RC=1" ยังคงผ่าน (`[PASS]`) แม้ว่า
`TAXCALCBUGGY` จะมีบั๊กเรื่องอัตราภาษีอยู่ก็ตาม

**เฉลย**: บั๊กที่ถูกใส่เข้าไปอยู่ใน branch `ELSE` ของ `IF LK-AMOUNT < 0` เท่านั้น (ส่วนที่คำนวณ
`COMPUTE LK-TAX ROUNDED = LK-AMOUNT * 0.10`) ส่วน branch `IF LK-AMOUNT < 0` (ที่ตั้งค่า `LK-TAX`
เป็น 0 และ `RETURN-CODE` เป็น 1) ไม่ได้ถูกแก้ไขเลย เมื่อ test case ส่ง `WS-TEST-AMOUNT = -500.00`
เข้าไป โปรแกรมจะเข้า branch `IF LK-AMOUNT < 0` ซึ่งยังคงทำงานถูกต้องเหมือนเดิมทุกประการ ทำให้
test case นี้ผ่าน — นี่คือตัวอย่างที่ดีของการที่ **test case หลายกรณีที่ครอบคลุมหลาย branch ของ
ตรรกะ ช่วยระบุตำแหน่งที่แท้จริงของบั๊กได้แม่นยำกว่า** (บั๊กอยู่ที่การคำนวณอัตราภาษี ไม่ใช่ที่การ
ตรวจสอบ validation) ซึ่งช่วยให้นักพัฒนาแก้ไขโค้ดได้ตรงจุดเร็วกว่าการรู้แค่ว่า "มีบางอย่างผิดพลาด"

---

## ขั้นตอนที่ 787: ลดความซ้ำซ้อนด้วย Assert Helper Paragraph

### ปัญหาของการเขียน Assertion ซ้ำ ๆ

สังเกตจากขั้นตอนที่ 784 ว่า `ASSERT-TAX-AND-RC` ถูกเรียกซ้ำจากทุก `TEST-xxx` paragraph อยู่แล้ว
แต่ถ้าเรามี subprogram หลายตัวที่ต้องทดสอบ (ไม่ใช่แค่ `TAXCALC` ตัวเดียว) การเขียน assertion แยก
เฉพาะของแต่ละ subprogram จะเริ่มซ้ำซ้อนกันมาก ขั้นตอนนี้แสดงวิธี**ออกแบบ assertion helper ที่ใช้
ร่วมกันได้ทั่วไป** โดยใช้ field ชื่อกลาง ๆ แทนชื่อเฉพาะของ subprogram ใดตัวหนึ่ง

### รูปแบบ Generic Assert (ตัวอย่างแนวคิด)

```cobol
       WORKING-STORAGE SECTION.
       01  WS-TEST-NAME             PIC X(40).
       01  WS-ACTUAL                PIC S9(7)V99.
       01  WS-EXPECTED              PIC S9(7)V99.
       01  WS-PASS-COUNT            PIC 9(3) VALUE 0.
       01  WS-FAIL-COUNT            PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
      *> ... (test cases move values into WS-ACTUAL / WS-EXPECTED
      *>      after each CALL, then PERFORM ASSERT-EQUAL) ...

       ASSERT-EQUAL.
           IF WS-ACTUAL = WS-EXPECTED
               DISPLAY "[PASS] " WS-TEST-NAME
               ADD 1 TO WS-PASS-COUNT
           ELSE
               DISPLAY "[FAIL] " WS-TEST-NAME
                   " expected=" WS-EXPECTED " actual=" WS-ACTUAL
               ADD 1 TO WS-FAIL-COUNT
           END-IF.
```

ในขั้นตอนที่ 789 เราจะเห็นรูปแบบ `ASSERT-RESULT` ที่เป็น generic assert แบบนี้ถูกใช้จริงกับ
subprogram สองตัวที่ต่างกัน (`TAXCALC` และ `DISCOUNTCALC`) ในไฟล์ test runner เดียวกัน พิสูจน์ว่า
การตั้งชื่อตัวแปรกลาง ๆ (`WS-ACTUAL`, `WS-EXPECTED` แทนที่จะเป็น `WS-TEST-TAX`) ทำให้ paragraph
assertion เดียวใช้ร่วมกันข้าม subprogram ได้จริง

### อธิบายจุดสำคัญ

- การเปลี่ยนชื่อจาก `WS-TEST-TAX`/`WS-EXPECTED-TAX` (เฉพาะเจาะจงกับ TAXCALC) เป็น `WS-ACTUAL`/
  `WS-EXPECTED` (ชื่อกลาง ๆ) คือหัวใจของการทำให้ assertion **นำกลับมาใช้ซ้ำได้ (reusable)**
- แนวคิดนี้คือ**การจำลองสิ่งที่เฟรมเวิร์ก unit test ในภาษาอื่นทำให้อัตโนมัติ** (เช่น
  `assertEqual(actual, expected)` ใน Python's `unittest`) — ใน COBOL เราต้องสร้างมันขึ้นมาเองในรูป
  ของ paragraph ที่ใช้ตัวแปร WORKING-STORAGE ร่วมกันแทน

### ข้อควรระวัง

- Generic assert แบบนี้เหมาะกับกรณีที่เปรียบเทียบ**ค่าเดียว** (single value) เท่านั้น ถ้าต้องการ
  เปรียบเทียบ group item ทั้งก้อน (หลาย field พร้อมกัน) ต้องออกแบบ assertion เพิ่มเติมที่เทียบทีละ
  field หรือใช้เทคนิค `MOVE CORRESPONDING`/การเทียบทั้ง group โดยตรงถ้าโครงสร้างตรงกันทุกประการ
- อย่าลืมว่า `WS-ACTUAL`/`WS-EXPECTED` เป็น**ตัวแปรที่ใช้ร่วมกันทุก test case** — ต้องตั้งค่าใหม่
  ทุกครั้งก่อน `PERFORM ASSERT-EQUAL` มิฉะนั้นค่าที่ค้างจาก test case ก่อนหน้าอาจทำให้ผลลัพธ์ผิดเพี้ยน

### แบบฝึกหัดที่ 787.1

**โจทย์**: จงอธิบายข้อจำกัดของแนวทาง "generic assert ด้วยตัวแปรกลาง" เมื่อเทียบกับเฟรมเวิร์ก unit
test ในภาษาสมัยใหม่ที่รองรับ generic type หรือ function overloading

**เฉลย**: ใน COBOL, `WS-ACTUAL`/`WS-EXPECTED` ต้องถูกประกาศด้วย PICTURE ที่ตายตัวไว้ล่วงหน้า (เช่น
`PIC S9(7)V99`) ทำให้ assertion helper นี้ใช้ได้เฉพาะกับค่าที่มีชนิดและความกว้างตรงกับที่ประกาศไว้
เท่านั้น ถ้าต้องการเทียบค่าที่เป็น `PIC X` (ข้อความ) หรือ `PIC 9(3)` (ความกว้างต่างออกไป) จะต้อง
สร้าง assertion helper แยกอีกชุดหนึ่ง (เช่น `ASSERT-EQUAL-TEXT`, `ASSERT-EQUAL-SMALL-NUM`) ต่างจาก
ภาษาที่รองรับ Generic/Template (เช่น Java generics หรือ Python ที่ไม่มีการประกาศชนิดตายตัว) ที่
ฟังก์ชัน `assertEqual()` ตัวเดียวสามารถรับค่าได้ทุกชนิดโดยอัตโนมัติ นี่คือข้อจำกัดที่มาจากธรรมชาติ
ของ COBOL ที่เป็นภาษาที่มีการประกาศชนิดข้อมูลอย่างเข้มงวด (strongly and statically typed ในความหมาย
ของ PICTURE Clause) ซึ่งทีมพัฒนาต้องยอมรับ trade-off นี้เมื่อออกแบบชุดเครื่องมือทดสอบของตัวเอง

---

## ขั้นตอนที่ 788: ทดสอบ Boundary และ Edge Case อย่างเป็นระบบ

### แนวคิด Boundary Value Analysis

**Boundary Value Analysis** คือเทคนิคการออกแบบ test case ที่เน้นทดสอบ**ค่าขอบเขต** ของช่วงข้อมูล
ที่โปรแกรมรองรับ เพราะบั๊กส่วนใหญ่มักซ่อนอยู่ที่ขอบเขตเหล่านี้ (เช่น ค่าศูนย์, ค่าสูงสุดที่ PICTURE
รองรับได้, จุดที่การปัดเศษเปลี่ยนทิศทาง) ไม่ใช่ตรงกลางของช่วงข้อมูลปกติ

### ทดสอบขอบเขตจริงกับ TAXCALC

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP788BOUNDARY.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-AMOUNT                PIC S9(7)V99.
       01  WS-TAX                   PIC S9(7)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 7.50 TO WS-AMOUNT.
           CALL "TAXCALC" USING WS-AMOUNT WS-TAX.
           DISPLAY "amount=" WS-AMOUNT " tax=" WS-TAX
               " (rounding boundary .525 -> .53)".

           MOVE 9999999.99 TO WS-AMOUNT.
           CALL "TAXCALC" USING WS-AMOUNT WS-TAX.
           DISPLAY "amount=" WS-AMOUNT " tax=" WS-TAX
               " (max PIC S9(7)V99 value)".

           MOVE 0.01 TO WS-AMOUNT.
           CALL "TAXCALC" USING WS-AMOUNT WS-TAX.
           DISPLAY "amount=" WS-AMOUNT " tax=" WS-TAX
               " (smallest positive unit)".
           STOP RUN.
```

**ผลลัพธ์จริง (ทดสอบรันแล้ว):**

```
amount=+0000007.50 tax=+0000000.53 (rounding boundary .525 -> .53)
amount=+9999999.99 tax=+0700000.00 (max PIC S9(7)V99 value)
amount=+0000000.01 tax=+0000000.00 (smallest positive unit)
```

### วิเคราะห์แต่ละ boundary case

| Case | ค่าที่ทดสอบ | สิ่งที่พิสูจน์ |
|---|---|---|
| Rounding boundary | 7.50 → tax คำนวณได้ 0.525 พอดี | ยืนยันว่า `ROUNDED` ปัดขึ้นเป็น 0.53 ถูกต้องตามกฎ round-half-up |
| Maximum value | 9999999.99 (ค่าสูงสุดที่ `PIC S9(7)V99` เก็บได้) | ยืนยันว่าไม่เกิด overflow หรือพฤติกรรมผิดปกติที่ค่าขอบบนสุด |
| Smallest positive unit | 0.01 (หน่วยที่เล็กที่สุดที่ไม่ใช่ศูนย์) | ยืนยันว่า 0.01 × 0.07 = 0.0007 ถูกปัดลงเป็น 0.00 อย่างถูกต้อง (ไม่ปัดขึ้นผิดพลาดเป็น 0.01) |

สังเกตกรณีสุดท้ายเป็นพิเศษ: 0.01 × 0.07 = 0.0007 ซึ่งน้อยกว่าครึ่งหนึ่งของหน่วยที่เล็กที่สุด (0.005)
มาก ทำให้ `ROUNDED` ปัดลงเป็น 0.00 อย่างถูกต้องตามกฎ — นี่คือตัวอย่างของ "edge case ที่ค่าใกล้ศูนย์
มาก" ที่บางทีมมักลืมทดสอบ เพราะดูเหมือนเป็นกรณีที่ "ไม่น่ามีปัญหาอะไร"

### ข้อควรระวัง

- **การหาค่าขอบเขตสูงสุดที่ PICTURE clause รองรับได้ต้องคำนวณเอง**: `PIC S9(7)V99` รองรับตัวเลข
  จำนวนเต็ม 7 หลักและทศนิยม 2 หลัก ทำให้ค่าสูงสุดคือ `9999999.99` — ถ้าลองใส่ค่าที่เกินกว่านี้
  (เช่น `10000000.00`) COBOL จะ**ตัดค่าให้พอดีกับความกว้างที่ประกาศไว้อย่างเงียบ ๆ** (truncation)
  โดยไม่มี error แจ้งเตือน (ทบทวนพฤติกรรมนี้จาก Part 008 เรื่อง MOVE Rules) จึงควรมี test case ที่
  ทดสอบพฤติกรรมนี้ไว้ด้วยถ้าโปรแกรมมีความเสี่ยงที่ input จะเกินขอบเขตได้จริง
- อย่าลืมทดสอบ**ค่าติดลบที่ใกล้ศูนย์มาก** เช่น -0.01 ด้วย เพื่อยืนยันว่าเงื่อนไข `IF LK-AMOUNT < 0`
  ทำงานถูกต้องแม้กับค่าที่ใกล้ขอบเขตของเงื่อนไขมากที่สุด (ไม่ใช่แค่ค่าติดลบมาก ๆ อย่าง -500.00
  ที่ทดสอบไปแล้วในขั้นตอนที่ 784)

### แบบฝึกหัดที่ 788.1

**โจทย์**: จงออกแบบ test case เพิ่มเติมอีก 1 กรณีที่ทดสอบค่าขอบเขต "ติดลบที่ใกล้ศูนย์ที่สุด" (-0.01)
กับ `TAXCALC` แล้วอธิบายว่าคาดว่าผลลัพธ์จะเป็นอย่างไร

**เฉลย**:

```cobol
           MOVE -0.01 TO WS-AMOUNT.
           CALL "TAXCALC" USING WS-AMOUNT WS-TAX.
           DISPLAY "amount=" WS-AMOUNT " tax=" WS-TAX
               " (smallest negative -> rejected)".
```

ผลลัพธ์ที่คาดหวังคือ `WS-TAX` จะถูกตั้งเป็น `0.00` และ `RETURN-CODE` จะถูกตั้งเป็น `1` เพราะเงื่อนไข
`IF LK-AMOUNT < 0` ใน `TAXCALC` ตรวจสอบแค่ว่าค่าน้อยกว่าศูนย์หรือไม่ โดยไม่สนใจว่าค่านั้นจะใกล้ศูนย์
แค่ไหน — แม้ -0.01 จะเป็นค่าติดลบที่น้อยที่สุดเท่าที่ `PIC S9(7)V99` เก็บได้ (ใกล้ศูนย์ที่สุด) มันก็
ยังคง "น้อยกว่าศูนย์" อยู่ดีตามตรรกะ ทำให้ยังถูกปฏิเสธเหมือนกับ -500.00 ทุกประการ test case นี้มี
ประโยชน์ในการยืนยันว่าเงื่อนไข `< 0` ไม่มี "ช่องโหว่" ที่ค่าใกล้ศูนย์มาก ๆ จะหลุดผ่านไปได้โดยไม่ได้
ตั้งใจ (เช่น ถ้ามีใครเขียนเงื่อนไขผิดเป็น `IF LK-AMOUNT < -1` โดยไม่ตั้งใจ test case นี้จะจับได้ทันที)

---

## ขั้นตอนที่ 789: จัดระเบียบ Test Suite ที่ครอบคลุมหลาย Subprogram

### เพิ่ม subprogram ตัวที่สอง: DISCOUNTCALC

เพื่อแสดงให้เห็นการจัดระเบียบ test suite ที่ครอบคลุม**หลาย subprogram**พร้อมกัน เราจะเพิ่ม
subprogram ใหม่ที่คำนวณส่วนลดตามประเภทลูกค้า:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DISCOUNTCALC.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-AMOUNT                PIC S9(7)V99.
       01  LK-CUST-TYPE             PIC X(1).
       01  LK-DISCOUNT              PIC S9(7)V99.

       PROCEDURE DIVISION USING LK-AMOUNT LK-CUST-TYPE LK-DISCOUNT.
       MAIN-PARA.
           EVALUATE LK-CUST-TYPE
               WHEN "V"
                   COMPUTE LK-DISCOUNT ROUNDED = LK-AMOUNT * 0.10
                   MOVE 0 TO RETURN-CODE
               WHEN "R"
                   COMPUTE LK-DISCOUNT ROUNDED = LK-AMOUNT * 0.02
                   MOVE 0 TO RETURN-CODE
               WHEN OTHER
                   MOVE 0 TO LK-DISCOUNT
                   MOVE 1 TO RETURN-CODE
           END-EVALUATE
           GOBACK.
```

### ALLSUITES — Test Runner ที่รวมหลาย Suite เข้าด้วยกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. ALLSUITES.
       AUTHOR. COBOL-COURSE.

      *> A single test-runner program that groups MULTIPLE
      *> subprograms' test suites together and prints one grand
      *> total, the way a real CI job reports "12 suites, 47 tests".
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TEST-NAME             PIC X(40).
       01  WS-AMOUNT                PIC S9(7)V99.
       01  WS-CUST-TYPE             PIC X(1).
       01  WS-ACTUAL                PIC S9(7)V99.
       01  WS-EXPECTED              PIC S9(7)V99.
       01  WS-EXPECTED-RC           PIC 9(2).
       01  WS-PASS-COUNT            PIC 9(3) VALUE 0.
       01  WS-FAIL-COUNT            PIC 9(3) VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "== Suite: TAXCALC ==".
           PERFORM TAX-TEST-1.
           PERFORM TAX-TEST-2.
           DISPLAY "== Suite: DISCOUNTCALC ==".
           PERFORM DISC-TEST-1.
           PERFORM DISC-TEST-2.
           PERFORM DISC-TEST-3.
           DISPLAY "=========================".
           DISPLAY "GRAND TOTAL PASS=" WS-PASS-COUNT
               " FAIL=" WS-FAIL-COUNT.
           IF WS-FAIL-COUNT > 0
               MOVE 1 TO RETURN-CODE
           ELSE
               MOVE 0 TO RETURN-CODE
           END-IF
           STOP RUN.

       TAX-TEST-1.
           MOVE "TAXCALC: 1000.00 -> 70.00" TO WS-TEST-NAME.
           MOVE 1000.00 TO WS-AMOUNT.
           MOVE 70.00 TO WS-EXPECTED.
           MOVE 0 TO WS-EXPECTED-RC.
           CALL "TAXCALC" USING WS-AMOUNT WS-ACTUAL.
           PERFORM ASSERT-RESULT.

       TAX-TEST-2.
           MOVE "TAXCALC: negative -> RC=1" TO WS-TEST-NAME.
           MOVE -10.00 TO WS-AMOUNT.
           MOVE 0.00 TO WS-EXPECTED.
           MOVE 1 TO WS-EXPECTED-RC.
           CALL "TAXCALC" USING WS-AMOUNT WS-ACTUAL.
           PERFORM ASSERT-RESULT.

       DISC-TEST-1.
           MOVE "DISCOUNTCALC: VIP 1000.00 -> 100.00" TO WS-TEST-NAME.
           MOVE 1000.00 TO WS-AMOUNT.
           MOVE "V" TO WS-CUST-TYPE.
           MOVE 100.00 TO WS-EXPECTED.
           MOVE 0 TO WS-EXPECTED-RC.
           CALL "DISCOUNTCALC" USING WS-AMOUNT WS-CUST-TYPE WS-ACTUAL.
           PERFORM ASSERT-RESULT.

       DISC-TEST-2.
           MOVE "DISCOUNTCALC: Regular 1000.00 -> 20.00" TO
               WS-TEST-NAME.
           MOVE 1000.00 TO WS-AMOUNT.
           MOVE "R" TO WS-CUST-TYPE.
           MOVE 20.00 TO WS-EXPECTED.
           MOVE 0 TO WS-EXPECTED-RC.
           CALL "DISCOUNTCALC" USING WS-AMOUNT WS-CUST-TYPE WS-ACTUAL.
           PERFORM ASSERT-RESULT.

       DISC-TEST-3.
           MOVE "DISCOUNTCALC: unknown type -> RC=1" TO WS-TEST-NAME.
           MOVE 1000.00 TO WS-AMOUNT.
           MOVE "X" TO WS-CUST-TYPE.
           MOVE 0.00 TO WS-EXPECTED.
           MOVE 1 TO WS-EXPECTED-RC.
           CALL "DISCOUNTCALC" USING WS-AMOUNT WS-CUST-TYPE WS-ACTUAL.
           PERFORM ASSERT-RESULT.

       ASSERT-RESULT.
           IF WS-ACTUAL = WS-EXPECTED AND RETURN-CODE = WS-EXPECTED-RC
               DISPLAY "[PASS] " WS-TEST-NAME
               ADD 1 TO WS-PASS-COUNT
           ELSE
               DISPLAY "[FAIL] " WS-TEST-NAME
                   " expected=" WS-EXPECTED " actual=" WS-ACTUAL
               ADD 1 TO WS-FAIL-COUNT
           END-IF.
```

**คอมไพล์และรันจริง:**

```bash
cobc -x -o allsuites allsuites.cob taxcalc.cob discountcalc.cob
./allsuites
```

**ผลลัพธ์จริง:**

```
== Suite: TAXCALC ==
[PASS] TAXCALC: 1000.00 -> 70.00
[PASS] TAXCALC: negative -> RC=1
== Suite: DISCOUNTCALC ==
[PASS] DISCOUNTCALC: VIP 1000.00 -> 100.00
[PASS] DISCOUNTCALC: Regular 1000.00 -> 20.00
[PASS] DISCOUNTCALC: unknown type -> RC=1
=========================
GRAND TOTAL PASS=005 FAIL=000
```

### อธิบายจุดสำคัญ

- นี่คือตัวอย่างจริงของ **generic assert** ที่กล่าวถึงในขั้นตอนที่ 787: `ASSERT-RESULT` ใช้
  `WS-ACTUAL`/`WS-EXPECTED` (ชื่อกลาง ๆ) ทำให้ paragraph เดียวนี้**ใช้ร่วมกันได้ทั้งกับผลลัพธ์จาก
  `TAXCALC` และ `DISCOUNTCALC`** แม้ทั้งสอง subprogram จะมี interface ที่ต่างกัน (`DISCOUNTCALC`
  รับพารามิเตอร์ 3 ตัว ในขณะที่ `TAXCALC` รับแค่ 2 ตัว) เพราะ assertion สนใจแค่ผลลัพธ์สุดท้าย
  ที่ถูก `MOVE` เข้ามาใน `WS-ACTUAL`/`WS-EXPECTED` เท่านั้น
- การจัดกลุ่มด้วย `DISPLAY "== Suite: xxx =="` ก่อนแต่ละกลุ่ม test ช่วยให้อ่านผลลัพธ์ง่ายขึ้นมาก
  เมื่อจำนวน test case เพิ่มขึ้นเรื่อย ๆ — เทียบเท่ากับแนวคิด "test suite" หรือ "test class" ใน
  เฟรมเวิร์กสมัยใหม่ที่จัดกลุ่ม test case ที่เกี่ยวข้องกันไว้ด้วยกัน

### ข้อควรระวัง

- เมื่อจำนวน subprogram และ test case เพิ่มขึ้นมาก (หลักร้อยหรือหลักพัน) การเขียนทุก test case
  เป็น paragraph แยกในไฟล์เดียวแบบนี้จะเริ่มไม่สะดวก ทีมขนาดใหญ่มักแยกไฟล์ test driver ตาม
  subprogram หรือตามโมดูลทางธุรกิจ แล้วใช้ script ภายนอก (เหมือน `ci_report.sh` จาก Part 078
  ขั้นตอนที่ 780) รวบรวมผลจากหลายไฟล์ test driver เข้าเป็นรายงานเดียว
- ระวังอย่าให้ `WS-EXPECTED-RC` และตัวแปรกลางอื่น ๆ **ถูกใช้ผิดลำดับ** ระหว่าง test case ที่ติดกัน
  — เช่นถ้าลืม `MOVE` ค่าใหม่ให้ `WS-EXPECTED-RC` ก่อน test case ถัดไป ค่าเก่าจากกรณีก่อนหน้าจะยัง
  ค้างอยู่และทำให้ assertion ผิดพลาดอย่างเงียบ ๆ (นี่คือเหตุผลที่ตัวอย่างข้างต้นกำหนดค่าทุก field
  ที่เกี่ยวข้องใหม่ทุกครั้งใน**ทุก** test case โดยไม่มีข้อยกเว้น)

### แบบฝึกหัดที่ 789.1

**โจทย์**: จงเพิ่ม test case ใหม่ชื่อ `DISC-TEST-4` ที่ทดสอบ `DISCOUNTCALC` ด้วยลูกค้าประเภท "R"
(Regular) และจำนวนเงิน 0.00 บาท คาดว่าผลลัพธ์ควรเป็นอย่างไร

**เฉลย**:

```cobol
       DISC-TEST-4.
           MOVE "DISCOUNTCALC: Regular 0.00 -> 0.00" TO WS-TEST-NAME.
           MOVE 0.00 TO WS-AMOUNT.
           MOVE "R" TO WS-CUST-TYPE.
           MOVE 0.00 TO WS-EXPECTED.
           MOVE 0 TO WS-EXPECTED-RC.
           CALL "DISCOUNTCALC" USING WS-AMOUNT WS-CUST-TYPE WS-ACTUAL.
           PERFORM ASSERT-RESULT.
```

ผลลัพธ์ที่คาดหวังคือ `[PASS]` เพราะ 0.00 × 0.02 = 0.00 พอดี และ `LK-CUST-TYPE = "R"` เป็นค่าที่
`DISCOUNTCALC` รู้จักอยู่แล้ว (ไม่ใช่ `OTHER`) จึงทำให้ `RETURN-CODE` เป็น `0` ตามปกติ (ไม่ใช่ error
case) — ต้องอย่าลืม `PERFORM DISC-TEST-4.` เพิ่มใน `MAIN-PARA` ด้วย มิฉะนั้น test case ที่เขียนไว้
จะไม่ถูกรันเลย (เป็นข้อผิดพลาดที่พบบ่อยเมื่อเพิ่ม test case ใหม่แล้วลืมเรียกใช้)

---

## ขั้นตอนที่ 790: เชื่อมต่อ Unit Test เข้ากับ CI Pipeline

### สร้าง run_unit_tests.sh

ทบทวนแนวคิดจาก Part 078: exit code ของโปรแกรม COBOL (ผ่าน `RETURN-CODE` ที่ `STOP RUN` ส่งต่อออก
มาเป็น exit code ของ process) สามารถนำมาใช้ตัดสินใจใน shell script ได้โดยตรง

```bash
#!/bin/bash
# run_unit_tests.sh - compile the ALLSUITES test runner together with the
# subprograms it tests, then run it. The COBOL program's own RETURN-CODE
# becomes this script's exit code, so a CI system sees PASS/FAIL directly.
set -u
echo "==> Building test runner (allsuites + taxcalc + discountcalc) ..."
cobc -x -o allsuites allsuites.cob taxcalc.cob discountcalc.cob
if [ $? -ne 0 ]; then
    echo "==> BUILD FAILED"
    exit 1
fi

echo "==> Running unit test suite ..."
./allsuites
TEST_STATUS=$?

if [ ${TEST_STATUS} -eq 0 ]; then
    echo "==> UNIT TESTS: ALL PASSED"
else
    echo "==> UNIT TESTS: FAILURES DETECTED"
fi
exit ${TEST_STATUS}
```

**รันจริง:**

```bash
chmod +x run_unit_tests.sh
./run_unit_tests.sh
```

**ผลลัพธ์จริง (ทดสอบรันแล้ว):**

```
==> Building test runner (allsuites + taxcalc + discountcalc) ...
==> Running unit test suite ...
== Suite: TAXCALC ==
[PASS] TAXCALC: 1000.00 -> 70.00
[PASS] TAXCALC: negative -> RC=1
== Suite: DISCOUNTCALC ==
[PASS] DISCOUNTCALC: VIP 1000.00 -> 100.00
[PASS] DISCOUNTCALC: Regular 1000.00 -> 20.00
[PASS] DISCOUNTCALC: unknown type -> RC=1
=========================
GRAND TOTAL PASS=005 FAIL=000
==> UNIT TESTS: ALL PASSED
```

(exit code สุดท้าย: `0`)

### การผสานเข้ากับ GitHub Actions YAML จาก Part 078

ขั้นตอน `Run automated test` ในไฟล์ `.github/workflows/cobol-ci.yml` (Part 078 ขั้นตอนที่ 778)
สามารถขยายให้เรียก `run_unit_tests.sh` เพิ่มเติมจาก Golden Output Test เดิมได้ทันที:

```yaml
      - name: Build COBOL programs
        run: |
          chmod +x build.sh test.sh run_unit_tests.sh
          ./build.sh step771calc

      - name: Run Golden Output test
        run: ./test.sh step771calc expected_output.txt

      - name: Run unit test suite
        run: ./run_unit_tests.sh
```

ทั้งสองขั้นตอน (`Run Golden Output test` และ `Run unit test suite`) ทำงาน**เสริมกัน**: ถ้าขั้นตอน
ไหน exit code ไม่เป็น `0`, GitHub Actions จะแสดง step นั้นเป็น ❌ ทันที และ job ทั้งหมดจะถูกทำเครื่องหมาย
ว่าล้มเหลว ทำให้ Pull Request ไม่สามารถ merge ได้จนกว่าจะแก้ไข (ถ้าทีมตั้งค่า branch protection
rule ไว้ตามนั้น)

### ภาพรวมกลยุทธ์การทดสอบแบบเต็มรูปแบบ

```
                    ┌─────────────────────────┐
                    │   git push / PR (080)    │
                    └────────────┬─────────────┘
                                 ▼
                    ┌─────────────────────────┐
                    │  Build ทุกโปรแกรม (078)  │
                    └────────────┬─────────────┘
                                 ▼
              ┌──────────────────┴──────────────────┐
              ▼                                      ▼
   ┌─────────────────────┐              ┌──────────────────────────┐
   │ Golden Output Test    │              │  Unit Test Suite (079)    │
   │ (ทดสอบทั้งโปรแกรม)    │              │  (ทดสอบ subprogram ย่อย)   │
   │ Part 078              │              │  ครอบคลุม edge case ต่างๆ  │
   └──────────┬───────────┘              └─────────────┬────────────┘
              └──────────────────┬─────────────────────┘
                                 ▼
                    ┌─────────────────────────┐
                    │   สรุปผล PASS/FAIL รวม    │
                    └─────────────────────────┘
```

### ข้อควรระวัง

- ควรรัน **Unit Test ก่อน Golden Output Test เสมอ** ในทางปฏิบัติ (แม้ตัวอย่างข้างบนจะเรียงแบบใดก็ได้)
  เพราะ Unit Test รันเร็วกว่ามาก (ไม่ต้องรอ I/O หรือ process ใหญ่) และให้ข้อมูล debug ที่ละเอียดกว่า
  — ถ้า Unit Test ล้มเหลวแล้ว มักไม่มีประโยชน์ที่จะเสียเวลารัน Golden Output Test ทั้งระบบต่อ (หลัก
  Fail Fast เดิมจาก Part 078 ขั้นตอนที่ 774)
- การมีทั้งสองชั้นการทดสอบ (unit + golden output) ไม่ได้แปลว่าปลอดภัย 100% เสมอไป — ยังมีสิ่งที่
  ทดสอบไม่ครอบคลุม เช่น performance ภายใต้ load หนัก หรือพฤติกรรมเมื่อรันพร้อมกันหลาย process
  (concurrency) ซึ่งเป็นหัวข้อขั้นสูงกว่าที่อยู่นอกเหนือขอบเขตของ Part นี้

### แบบฝึกหัดที่ 790.1

**โจทย์**: จงอธิบายว่าทำไมการรัน Unit Test ก่อน Golden Output Test (ตามที่แนะนำในข้อควรระวังข้างต้น)
ถึงเป็นกลยุทธ์ที่ประหยัดเวลาของทีมพัฒนามากกว่าการรันในลำดับกลับกัน

**เฉลย**: Unit Test ทดสอบ subprogram แต่ละตัวแยกกันในระดับที่เล็กที่สุด ทำให้รันเร็วมากและระบุ
ตำแหน่งบั๊กได้ชัดเจนทันที (เช่น "TAXCALC คำนวณผิด") ในขณะที่ Golden Output Test ต้องรันโปรแกรมทั้ง
ระบบ ซึ่งมักใช้เวลานานกว่าและอาจเกี่ยวข้องกับการอ่าน/เขียนไฟล์จริง (I/O) หากมี Unit Test ที่ล้มเหลว
อยู่แล้ว การไปรัน Golden Output Test ต่อแทบจะรับประกันได้เลยว่าจะล้มเหลวตามไปด้วย (เพราะ subprogram
ที่มีบั๊กเป็นส่วนหนึ่งของระบบทั้งชิ้นอยู่แล้ว) ทำให้เสียเวลารันระบบทั้งหมดไปโดยไม่ได้ข้อมูลใหม่ที่มี
ประโยชน์เพิ่มเติมจาก Unit Test ที่ล้มเหลวไปแล้ว การจัดลำดับ "เร็วและละเอียดก่อน ช้าและกว้างทีหลัง"
คือหลักการ Fail Fast ที่ประยุกต์ใช้กับการออกแบบลำดับขั้นตอนใน pipeline โดยตรง ช่วยให้ทีมได้รับ
feedback ว่าโค้ดมีปัญหาเร็วที่สุดเท่าที่เป็นไปได้เสมอ

---

## สรุปท้ายบท

Part นี้พาคุณสร้างกลยุทธ์ Unit Testing สำหรับ COBOL ตั้งแต่ศูนย์ โดยไม่ต้องพึ่งพาเฟรมเวิร์กเชิง
พาณิชย์ใด ๆ เลย และทุกตัวอย่างรันจริงพิสูจน์ผลลัพธ์แล้วทั้งกรณี PASS และ FAIL:

- เหตุผลที่ COBOL ทดสอบระดับหน่วยย่อยได้ยากกว่าภาษาสมัยใหม่ (ไม่มี return value, I/O ปนกับตรรกะ,
  WORKING-STORAGE คงค่าข้ามการเรียก, ไม่มีเฟรมเวิร์กมาตรฐาน)
- ผลการตรวจสอบจริงว่าไม่มีเครื่องมือ unit test เฉพาะทาง (funcUT, MFUnit) ติดตั้งในสภาพแวดล้อมนี้
  และเหตุผลที่หลักสูตรเลือกแนวทาง Test Driver ที่เขียนด้วย COBOL เอง
- การออกแบบ subprogram แบบ "pure function-like" ที่ทดสอบง่าย (`TAXCALC`)
- การเขียน Test Driver ที่ CALL หลายกรณีพร้อม assert ผลลัพธ์ (AAA Pattern)
- การรันจริงยืนยันผล PASS ทั้ง 4 กรณี และวิเคราะห์ว่าแต่ละกรณีทดสอบอะไร
- การจงใจใส่บั๊กแล้วพิสูจน์ว่า test driver จับ FAIL ได้จริง พร้อมวิเคราะห์ว่าทำไมบาง test case
  ยังผ่านอยู่
- การลด duplicate code ด้วย Assert Helper Paragraph แบบ generic
- Boundary Value Analysis: การออกแบบ test case ที่ทดสอบค่าขอบเขต (ศูนย์, ค่าสูงสุด, จุดปัดเศษ)
- การจัดระเบียบ test suite ที่ครอบคลุมหลาย subprogram พร้อมกันในไฟล์เดียว (`ALLSUITES`)
- การผสาน Unit Test เข้ากับ CI pipeline จาก Part 078 พร้อมกลยุทธ์การจัดลำดับที่ประหยัดเวลา

Part ถัดไป (**Part 080**) จะเปลี่ยนมุมมองไปที่**กระบวนการทำงานร่วมกันของทีม**: Version Control
ด้วย Git ที่ประยุกต์ใช้กับโปรเจกต์ COBOL โดยเฉพาะ — `.gitignore` สำหรับไฟล์ execute ที่คอมไพล์แล้ว,
Branching Workflow, และประเด็นเฉพาะทางของ Code Review ในโค้ด COBOL เช่นความอ่อนไหวต่อคอลัมน์ใน
diff และการประสานงานเมื่อแก้ไข copybook ที่ใช้ร่วมกันหลายโปรแกรม

**[← กลับไป Part 078: CI/CD Pipeline สำหรับโปรเจกต์ COBOL](part-078-cicd-cobol.md)** | **[ไปยัง Part 080: Version Control (Git) และ Workflow สำหรับทีม COBOL →](part-080-git-workflow-cobol.md)**
