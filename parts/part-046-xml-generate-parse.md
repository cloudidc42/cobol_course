# Part 046: XML GENERATE และ XML PARSE (ขั้นตอนที่ 451–460)

## คำนำของ Part นี้

**XML** (eXtensible Markup Language) เป็นรูปแบบข้อมูลที่ใช้กันมานานในระบบองค์กรขนาดใหญ่ก่อนที่ JSON
จะเข้ามาแพร่หลาย โดยเฉพาะในระบบ Enterprise แบบดั้งเดิม เช่น การแลกเปลี่ยนข้อมูลระหว่างธนาคาร (SWIFT
messages บางรูปแบบ), ระบบ EDI (Electronic Data Interchange), และ Web Services แบบ SOAP ที่ยังใช้งาน
อยู่จริงในหลายองค์กรจนถึงปัจจุบัน มาตรฐาน COBOL เพิ่มความสามารถ `XML GENERATE`/`XML PARSE` เข้ามา
ตั้งแต่ COBOL-2002 (ก่อน JSON เสียอีก) เพื่อรองรับความต้องการนี้โดยตรง

Part นี้จะเดินตามแนวทางเดียวกับ Part 045: **ทดสอบ feature จริงก่อนสอน** ผลปรากฏว่าบิลด์ GnuCOBOL ที่
ใช้ในหลักสูตรนี้**ปิดใช้งาน XML library เช่นเดียวกับ JSON library** (ยืนยันด้วย `cobc --info`) ทำให้
`XML GENERATE`/`XML PARSE` compile ไม่ผ่านเช่นกัน Part นี้จึงสอนทั้งไวยากรณ์มาตรฐานสำหรับใช้อ้างอิงเมื่อ
ทำงานกับคอมไพเลอร์ที่รองรับเต็มรูปแบบ และเทคนิคสร้าง/แยกวิเคราะห์ XML ด้วยมือที่ใช้งานได้จริงทันทีใน
ทุกสภาพแวดล้อม

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน ทุกตัวอย่างที่แสดงว่า
> "compile และรันได้" ผ่านการทดสอบจริงด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้ว

---

## ขั้นตอนที่ 451: XML คืออะไร และภาพรวมไวยากรณ์ XML GENERATE/PARSE ตามมาตรฐาน

### XML ในรูปแบบง่าย ๆ

XML เก็บข้อมูลด้วย "แท็ก" (tag) ที่เปิด-ปิดคู่กัน เช่น `<name>SOMCHAI</name>` และสามารถมี attribute
แนบไว้ในแท็กเปิดได้ เช่น `<item qty="10">APPLE</item>` โครงสร้างสามารถซ้อนกันหลายชั้น (nested) ได้
ตามธรรมชาติ:

```xml
<customer><name>SOMCHAI JAIDEE</name><age>30</age></customer>
```

### ไวยากรณ์มาตรฐานของ XML GENERATE (ตาม COBOL-2002/2014)

```
XML GENERATE identifier-1 FROM identifier-2
    [ COUNT IN identifier-3 ]
    [ NAMESPACE literal-1 [ WITH literal-2 ] ]
    [ ON EXCEPTION statement-1 ]
    [ NOT ON EXCEPTION statement-2 ]
    [ END-XML ]
```

และ `XML PARSE` สำหรับแยกวิเคราะห์ทำงานย้อนกลับ โดยใช้กลไก **PROCESSING PROCEDURE** ที่เรียก
paragraph ของเรากลับมาทุกครั้งที่พบ event ของ XML (เช่น พบแท็กเปิด, พบข้อความ, พบแท็กปิด) ต่างจาก
`JSON PARSE` ที่แปลงเข้าโครงสร้างข้อมูลให้ตรง ๆ:

```
XML PARSE identifier-1
    PROCESSING PROCEDURE procedure-name-1
    [ ON EXCEPTION statement-1 ]
    [ NOT ON EXCEPTION statement-2 ]
    [ END-XML ]
```

### ตัวอย่างที่ต้องการทดสอบ (แสดงไวยากรณ์อ้างอิงตามมาตรฐาน)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. XMLTEST.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-OUT PIC X(200).
       01  WS-REC.
           05 WS-NAME PIC X(10) VALUE "JOHN".
           05 WS-AGE  PIC 9(3) VALUE 30.
       PROCEDURE DIVISION.
           XML GENERATE WS-OUT FROM WS-REC
           DISPLAY WS-OUT
           STOP RUN.
```

ทดสอบ compile จริง:

```
cobc -x -o xmltest xmltest.cob
```

### อธิบายจุดสำคัญ

- `XML GENERATE target FROM source` มีรูปแบบคล้าย `JSON GENERATE` มาก — ระบุตัวแปรปลายทางและกลุ่ม
  ข้อมูลต้นทาง คอมไพเลอร์จะแปลงชื่อ field เป็นชื่อแท็ก XML และค่าของ field เป็นเนื้อหาในแท็กให้
- `XML PARSE` แตกต่างจาก `JSON PARSE` ตรงที่ใช้กลไก **event-driven** ผ่าน `PROCESSING PROCEDURE` —
  เราต้องเขียน paragraph ที่จะถูกเรียกกลับหลายครั้งระหว่างการแยกวิเคราะห์ แล้วตรวจสอบ special register
  `XML-EVENT` เพื่อรู้ว่าขณะนี้กำลังประมวลผลส่วนไหนของ XML (เปิดแท็ก, ปิดแท็ก, เนื้อหา ฯลฯ)

### ข้อควรระวัง

- เช่นเดียวกับ Part 045 ไวยากรณ์นี้เป็นการอ้างอิงตามมาตรฐาน **ยังไม่ได้ยืนยันว่าทำงานได้จริงในบิลด์
  นี้** — ผลการทดสอบจริงจะแสดงในขั้นตอนถัดไป

### แบบฝึกหัดที่ 451.1

**โจทย์**: จงอธิบายความแตกต่างเชิงแนวคิดระหว่างกลไกการ parse ของ `JSON PARSE` (แปลงตรงเข้าโครงสร้าง
ข้อมูล) กับ `XML PARSE` (event-driven ผ่าน PROCESSING PROCEDURE)

**เฉลย**: `JSON PARSE` ออกแบบมาให้ทำงานแบบ "ตรงไปตรงมา" คือรับ JSON แล้วเติมค่าลงในโครงสร้างข้อมูล
COBOL ที่กำหนดไว้ทันทีในคำสั่งเดียว เหมาะกับ JSON ที่มีโครงสร้างค่อนข้างสอดคล้องกับ record ของ COBOL
โดยตรง ส่วน `XML PARSE` ใช้แนวทาง event-driven เนื่องจาก XML มีความยืดหยุ่นสูงกว่า (attribute, nested
element ที่ซับซ้อน, mixed content) การให้โปรแกรมเมอร์เขียนโค้ดจัดการแต่ละ event เองจึงยืดหยุ่นกว่าใน
การรับมือกับโครงสร้าง XML ที่หลากหลาย แม้จะต้องเขียนโค้ดมากกว่าก็ตาม

---

## ขั้นตอนที่ 452: ทดสอบจริง — เหตุใด XML GENERATE จึง Compile ไม่ผ่านในบิลด์นี้เช่นกัน

### ผลการทดสอบจริง

เมื่อคอมไพล์โค้ดจากขั้นตอนที่ 451 จริงด้วย `cobc (GnuCOBOL) 4.0-early-dev.0` ได้รับ error:

```
xmltest.cob:11: error: compiler is not configured to support XML
```

และผลจาก `cobc --info` ที่แสดงไว้แล้วใน Part 045 ก็ยืนยันสถานะนี้ตรง ๆ:

```
XML library              : disabled
JSON library             : disabled
```

### สาเหตุ: XML ต้องพึ่งพา libxml2 ที่ต้องเชื่อมไว้ตอน Build คอมไพเลอร์

เช่นเดียวกับ JSON, ความสามารถ `XML GENERATE`/`XML PARSE` ของ GnuCOBOL พึ่งพาไลบรารีภายนอกชื่อ
**libxml2** ที่ต้อง link เข้ากับตัว `cobc` ตอน build คอมไพเลอร์เอง ตรวจสอบระบบพบว่า `libxml2` **มี
ติดตั้งอยู่ในระบบจริง** (ผ่านคำสั่ง `ldconfig -p | grep libxml2` ยืนยันว่ามีไฟล์ `libxml2.so.2` อยู่)
แต่ตัว `cobc` ที่ติดตั้งจากแพ็กเกจ `gnucobol4` ของ Ubuntu ในสภาพแวดล้อมนี้**ไม่ได้ถูก build มาให้เชื่อม
กับไลบรารีนี้** ทำให้แม้ระบบจะมีไลบรารีพร้อมใช้งาน แต่ตัวคอมไพเลอร์เองก็ยังไม่สามารถใช้ประโยชน์จากมัน
ได้ นี่คือตัวอย่างที่ชัดเจนว่า **"มีไลบรารีอยู่ในระบบ" กับ "คอมไพเลอร์ถูกตั้งค่าให้ใช้ไลบรารีนั้นได้"
เป็นคนละเรื่องกันโดยสิ้นเชิง**

### อธิบายจุดสำคัญ

- ข้อผิดพลาดทั้ง JSON และ XML มีรูปแบบเดียวกันทุกประการ ("compiler is not configured to support...")
  สะท้อนว่า GnuCOBOL ออกแบบทั้งสอง feature ด้วยสถาปัตยกรรมแบบเดียวกัน คือพึ่งพาไลบรารีภายนอกที่ต้อง
  เปิดใช้งานตอน build
- การที่ libxml2 มีอยู่ในระบบปฏิบัติการไม่ได้ช่วยอะไรเลย เพราะปัญหาอยู่ที่การ**เชื่อมโยง (link)** ไว้
  ตอน build ตัว `cobc` ไม่ใช่การมีไฟล์ไลบรารีอยู่ที่ใดที่หนึ่งในระบบ

### ข้อควรระวัง

- อย่าสันนิษฐานว่า "ติดตั้งไลบรารีที่จำเป็นแล้ว ฟีเจอร์ต้องใช้งานได้" — ต้องตรวจสอบว่า**ตัวโปรแกรมที่
  จะใช้ไลบรารีนั้น** (ในที่นี้คือ `cobc`) ถูกสร้างมาให้เชื่อมกับไลบรารีนั้นจริงหรือไม่ ด้วยคำสั่งที่
  โปรแกรมนั้นมีให้ตรวจสอบโดยตรง (`cobc --info` ในกรณีนี้) ไม่ใช่ตรวจสอบผ่านคำสั่งของระบบปฏิบัติการ
  เพียงอย่างเดียว
- หากต้องการใช้ `XML GENERATE`/`XML PARSE` จริงจัง ทางเลือกเดียวกับที่กล่าวไว้ใน Part 045: เปลี่ยนไป
  ใช้คอมไพเลอร์เชิงพาณิชย์ที่รองรับ, build GnuCOBOL จาก source เองพร้อมเปิดใช้งาน libxml2, หรือใช้
  เทคนิคสร้าง/แยกวิเคราะห์ด้วยมือตามที่จะสอนในขั้นตอนถัดไป

### แบบฝึกหัดที่ 452.1

**โจทย์**: จงอธิบายว่าทำไมการพบว่า `libxml2.so.2` ติดตั้งอยู่ในระบบจึง**ไม่ได้แปลว่า** `XML GENERATE`
จะ compile ผ่านได้

**เฉลย**: การที่ไฟล์ไลบรารี `libxml2.so.2` มีอยู่ในระบบเป็นเพียงเงื่อนไขที่จำเป็น (necessary) แต่ไม่
เพียงพอ (not sufficient) สำหรับให้ `cobc` ใช้ความสามารถ XML ได้ ตัว `cobc` ต้องถูก **compile และ
link เข้ากับ libxml2 ตั้งแต่ตอนสร้างตัวคอมไพเลอร์เอง** ผ่าน configure flag เฉพาะ (เช่น
`--with-xml2`) หากตอน build ไม่ได้เปิดใช้งาน flag นี้ไว้ ตัว `cobc` ที่ได้จะไม่มีโค้ดเรียกใช้
libxml2 อยู่เลย แม้ไฟล์ไลบรารีจะอยู่ในเครื่องพร้อมใช้งานก็ตาม เหมือนมีหนังสือเรียนอยู่ในห้องสมุดแต่ไม่มี
ใครไปหยิบมาอ่าน

---

## ขั้นตอนที่ 453: กลยุทธ์รับมือ — สร้าง XML ด้วยมือผ่าน STRING Statement

### แนวคิด: XML ก็เป็นข้อความที่มีรูปแบบตายตัวเช่นเดียวกับ JSON

เราสามารถสร้างข้อความ XML เองได้ด้วย `STRING` เช่นเดียวกับที่ทำกับ JSON ใน Part 045

### ตัวอย่าง: แปลง Record COBOL เป็น XML ด้วยมือ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. XMAN1.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NAME         PIC X(15) VALUE "SOMCHAI JAIDEE".
       01  WS-AGE          PIC 9(3)  VALUE 30.
       01  WS-AGE-EDIT     PIC ZZ9.
       01  WS-XML          PIC X(120).
       PROCEDURE DIVISION.
           MOVE WS-AGE TO WS-AGE-EDIT
           STRING "<customer>"                        DELIMITED BY SIZE
                   "<name>" DELIMITED BY SIZE
                   FUNCTION TRIM(WS-NAME) DELIMITED BY SIZE
                   "</name>" DELIMITED BY SIZE
                   "<age>" DELIMITED BY SIZE
                   FUNCTION TRIM(WS-AGE-EDIT) DELIMITED BY SIZE
                   "</age>" DELIMITED BY SIZE
                   "</customer>" DELIMITED BY SIZE
               INTO WS-XML
           DISPLAY FUNCTION TRIM(WS-XML)
           STOP RUN.
```

**ผลลัพธ์:**

```
<customer><name>SOMCHAI JAIDEE</name><age>30</age></customer>
```

### อธิบายจุดสำคัญ

- แต่ละ field COBOL แปลงเป็นคู่แท็กเปิด-ปิด `<fieldname>value</fieldname>` เรียงต่อกันตามลำดับ —
  หลักการเดียวกับที่ `XML GENERATE` จะทำให้อัตโนมัติหากคอมไพเลอร์รองรับ
- ต่างจาก JSON ที่ใช้เครื่องหมายคำพูดคู่ล้อมค่าข้อความ XML ใช้**แท็กปิด**ที่มีชื่อตรงกับแท็กเปิด (นำ
  หน้าด้วย `/`) เป็นตัวบอกจุดสิ้นสุดของค่าแทน ทำให้ไม่ต้องกังวลเรื่องเครื่องหมายคำพูดในค่าข้อมูลเหมือน
  JSON แต่ต้องระวังอักขระพิเศษของ XML แทน (จะสอนในขั้นตอนที่ 454)

### ข้อควรระวัง

- เทคนิคนี้เหมาะกับโครงสร้างเรียบง่ายเช่นเดียวกับ JSON ในบทก่อน หากต้องการ nested element ที่ซับซ้อน
  ต้องเขียนโค้ดเพิ่มเติมตามลำดับชั้นเอง
- ต้องตรวจสอบให้แน่ใจว่าทุกแท็กที่เปิดมีแท็กปิดคู่กันเสมอ (well-formed XML) มิฉะนั้นโปรแกรมปลายทางที่
  อ่าน XML นี้จะ parse ไม่สำเร็จ — การเขียนด้วยมือมีความเสี่ยงสูงกว่าการใช้ `XML GENERATE` ที่รับประกัน
  ความถูกต้องของโครงสร้างโดยอัตโนมัติ

### แบบฝึกหัดที่ 453.1

**โจทย์**: จงปรับโปรแกรม `XMAN1` ให้เพิ่ม element `<city>BANGKOK</city>` เข้าไปก่อนแท็กปิด
`</customer>`

**เฉลย**: เพิ่มส่วน `STRING` เข้าไปก่อน `"</customer>"`:

```cobol
           STRING "<customer>"                        DELIMITED BY SIZE
                   "<name>" DELIMITED BY SIZE
                   FUNCTION TRIM(WS-NAME) DELIMITED BY SIZE
                   "</name>" DELIMITED BY SIZE
                   "<age>" DELIMITED BY SIZE
                   FUNCTION TRIM(WS-AGE-EDIT) DELIMITED BY SIZE
                   "</age>" DELIMITED BY SIZE
                   "<city>BANGKOK</city>" DELIMITED BY SIZE
                   "</customer>" DELIMITED BY SIZE
               INTO WS-XML
```

---

## ขั้นตอนที่ 454: การ Escape อักขระพิเศษใน XML

### อักขระพิเศษ 5 ตัวที่ XML สงวนไว้

XML สงวนอักขระ 5 ตัวไว้เป็นพิเศษ เพราะมีความหมายทางโครงสร้าง (บอกจุดเริ่ม/จบแท็ก หรือ attribute)
หากค่าข้อมูลจริงมีอักขระเหล่านี้ปนอยู่ ต้อง escape ก่อนเสมอ:

| อักขระ | ความหมายใน XML | รูปแบบ Escape |
|---|---|---|
| `&` | เริ่มต้น entity reference | `&amp;` |
| `<` | เริ่มต้นแท็ก | `&lt;` |
| `>` | จบแท็ก | `&gt;` |
| `"` | ล้อม attribute value | `&quot;` |
| `'` | ล้อม attribute value (แบบเดี่ยว) | `&apos;` |

### ตัวอย่าง: Escape อักขระพิเศษด้วยมือทีละตัวอักษร

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. XMAN2.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RAW          PIC X(30) VALUE "Tom & Jerry <best friends>".
       01  WS-ESCAPED      PIC X(60).
       01  WS-LEN          PIC 9(4).
       01  WS-I            PIC 9(2).
       01  WS-J            PIC 9(4) VALUE 1.
       01  WS-CH           PIC X.
       PROCEDURE DIVISION.
           COMPUTE WS-LEN = FUNCTION LENGTH(FUNCTION TRIM(WS-RAW))
           MOVE SPACES TO WS-ESCAPED
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > WS-LEN
               MOVE WS-RAW(WS-I:1) TO WS-CH
               EVALUATE WS-CH
                   WHEN "&"
                       STRING "&amp;" DELIMITED BY SIZE
                           INTO WS-ESCAPED WITH POINTER WS-J
                   WHEN "<"
                       STRING "&lt;" DELIMITED BY SIZE
                           INTO WS-ESCAPED WITH POINTER WS-J
                   WHEN ">"
                       STRING "&gt;" DELIMITED BY SIZE
                           INTO WS-ESCAPED WITH POINTER WS-J
                   WHEN OTHER
                       STRING WS-CH DELIMITED BY SIZE
                           INTO WS-ESCAPED WITH POINTER WS-J
               END-EVALUATE
           END-PERFORM
           DISPLAY "Original: " FUNCTION TRIM(WS-RAW)
           DISPLAY "Escaped : " FUNCTION TRIM(WS-ESCAPED)
           STOP RUN.
```

**ผลลัพธ์:**

```
Original: Tom & Jerry <best friends>
Escaped : Tom &amp; Jerry &lt;best friends&gt;
```

### อธิบายจุดสำคัญ

- `EVALUATE WS-CH` ตรวจสอบทีละอักขระว่าตรงกับอักขระพิเศษตัวใดหรือไม่ ถ้าตรงจะแทนที่ด้วย entity
  reference ที่เหมาะสม (`&amp;`, `&lt;`, `&gt;`) ถ้าไม่ตรงกับตัวใดเลย (`WHEN OTHER`) จะ copy อักขระ
  เดิมตรง ๆ
- `STRING ... WITH POINTER WS-J`: ใช้เทคนิคเดียวกับที่เรียนใน Part 045 (ขั้นตอนที่ 445) เพื่อต่อ
  ข้อความหลายส่วนเข้าด้วยกันโดยรักษาตำแหน่งที่เขียนล่าสุดไว้

### ข้อควรระวัง

- ตัวอย่างนี้สาธิตเฉพาะ `&`, `<`, `>` เพื่อความกระชับ ในการใช้งานจริงควร escape `"` และ `'` ด้วย
  เช่นกันหากค่านั้นจะถูกนำไปใส่ใน attribute (เพิ่ม `WHEN` อีกสองกรณีตามรูปแบบเดียวกัน)
- ลำดับการ escape สำคัญมาก: **ต้อง escape `&` ก่อนอักขระอื่นเสมอ** เพราะถ้า escape ตัวอื่นก่อนแล้ว
  ค่อย escape `&` ทีหลัง จะไป escape ซ้ำ entity reference ที่เพิ่งสร้างไป (เช่น `&lt;` จะกลายเป็น
  `&amp;lt;` ผิดเพี้ยน) ตัวอย่างข้างต้นประมวลผลทีละตัวอักษรในรอบเดียว จึงไม่มีปัญหานี้ แต่หากออกแบบ
  ด้วยวิธี "แทนที่ทั้งสตริง" (เช่นด้วย `INSPECT REPLACING`) หลายรอบต่อเนื่องกัน ต้องเรียง `&` ไว้เป็น
  อันดับแรกเสมอ

### แบบฝึกหัดที่ 454.1

**โจทย์**: จงอธิบายว่าทำไมการ escape `&` ก่อนอักขระอื่นจึงสำคัญ โดยยกตัวอย่างประกอบ

**เฉลย**: สมมติมีข้อความต้นฉบับ `<tag>` และเรา escape `<` ก่อนเป็น `&lt;tag&gt;` แล้วจึงค่อย escape
`&` ทีหลัง จะได้ผลลัพธ์เป็น `&amp;lt;tag&amp;gt;` ซึ่งผิดพลาด เพราะ `&` ที่เพิ่งสร้างขึ้นจากการ
escape `<` ถูกนำไป escape ซ้ำอีกรอบ กลายเป็นข้อความที่แสดงผลผิดเพี้ยนเมื่อถูก parse กลับ (จะเห็น
`&lt;tag&gt;` เป็นข้อความตรง ๆ แทนที่จะถูกตีความกลับเป็น `<tag>`) ดังนั้นต้อง escape `&` เป็นลำดับแรก
เสมอเพื่อป้องกันการ escape ซ้ำซ้อนแบบนี้

---

## ขั้นตอนที่ 455: แปลงตาราง (OCCURS) เป็น XML Repeating Elements ด้วยมือ

### แนวคิด: XML ไม่มี Array แบบ JSON แต่ใช้ Element ซ้ำชื่อเดิม

ต่างจาก JSON ที่มี syntax `[ ]` เฉพาะสำหรับ array, XML แสดงรายการซ้ำด้วยการเขียน element ชื่อเดิมซ้ำ
กันหลายครั้งเรียงต่อกัน เช่น `<item>...</item><item>...</item><item>...</item>` และมักใช้
**attribute** เก็บข้อมูลเสริมประกอบแต่ละ element ด้วย

### ตัวอย่าง: แปลงตารางสินค้าเป็น XML พร้อม Attribute

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. XMAN3.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ITEMS.
           05  WS-ITEM OCCURS 3 TIMES PIC X(8).
       01  WS-QTY.
           05  WS-QTY-ITEM OCCURS 3 TIMES PIC 9(3).
       01  WS-QTY-EDIT     PIC ZZ9.
       01  WS-I            PIC 9(2).
       01  WS-XML          PIC X(200).
       01  WS-PTR          PIC 9(4) VALUE 1.
       PROCEDURE DIVISION.
           MOVE "APPLE   " TO WS-ITEM(1)
           MOVE "BANANA  " TO WS-ITEM(2)
           MOVE "CHERRY  " TO WS-ITEM(3)
           MOVE 10 TO WS-QTY-ITEM(1)
           MOVE 25 TO WS-QTY-ITEM(2)
           MOVE 7  TO WS-QTY-ITEM(3)

           MOVE SPACES TO WS-XML
           STRING "<items>" DELIMITED BY SIZE
               INTO WS-XML WITH POINTER WS-PTR
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 3
               MOVE WS-QTY-ITEM(WS-I) TO WS-QTY-EDIT
               STRING '<item qty="' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-QTY-EDIT) DELIMITED BY SIZE
                   '">' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-ITEM(WS-I)) DELIMITED BY SIZE
                   "</item>" DELIMITED BY SIZE
                   INTO WS-XML WITH POINTER WS-PTR
           END-PERFORM
           STRING "</items>" DELIMITED BY SIZE
               INTO WS-XML WITH POINTER WS-PTR
           DISPLAY FUNCTION TRIM(WS-XML)
           STOP RUN.
```

**ผลลัพธ์:**

```
<items><item qty="10">APPLE</item><item qty="25">BANANA</item><item qty="7">CHERRY</item></items>
```

### อธิบายจุดสำคัญ

- `'<item qty="'` ... `'">'`: ใช้เครื่องหมายคำพูดเดี่ยวล้อม literal ที่มีเครื่องหมายคำพูดคู่อยู่ข้างใน
  (สำหรับล้อมค่า attribute) — เทคนิคเดียวกับที่ใช้ตอนสร้าง JSON ใน Part 045
- แต่ละ `<item>` มี attribute `qty` เก็บจำนวน และมีเนื้อหาข้างในเป็นชื่อสินค้า — แสดงให้เห็นว่า XML
  รองรับการเก็บข้อมูลได้สองรูปแบบพร้อมกันในหนึ่ง element (attribute และ content) ซึ่ง JSON ไม่มี
  แนวคิดที่ตรงกันโดยตรง
- Element `<item>` ถูกเขียนซ้ำ 3 ครั้งเรียงต่อกันภายใน `<items>...</items>` แทนการใช้ syntax พิเศษ
  แบบ array

### ข้อควรระวัง

- ค่าที่นำไปใส่ใน attribute (`qty="..."`) ควร escape เครื่องหมายคำพูดคู่หากมีความเป็นไปได้ที่ค่านั้น
  จะมีเครื่องหมายคำพูดปนอยู่ (ในตัวอย่างนี้เป็นตัวเลขล้วนจึงไม่มีความเสี่ยง แต่ถ้าเป็นค่าข้อความที่มา
  จากผู้ใช้ ต้องระวังเสมอ)
- ต้องตรวจสอบว่าทุก element ปิดถูกต้องครบถ้วน โดยเฉพาะเมื่อสร้างด้วยลูปที่อาจมีเงื่อนไขซับซ้อน

### แบบฝึกหัดที่ 455.1

**โจทย์**: จงปรับโปรแกรม `XMAN3` ให้เพิ่ม attribute ตัวที่สองชื่อ `unit` ที่มีค่าเป็น `"kg"` สำหรับ
ทุก item

**เฉลย**: แก้ไขส่วน `STRING` ในลูปให้เพิ่ม attribute เข้าไป:

```cobol
               STRING '<item qty="' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-QTY-EDIT) DELIMITED BY SIZE
                   '" unit="kg">' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-ITEM(WS-I)) DELIMITED BY SIZE
                   "</item>" DELIMITED BY SIZE
                   INTO WS-XML WITH POINTER WS-PTR
```

---

## ขั้นตอนที่ 456: การ Parse XML แบบง่ายด้วยมือผ่าน UNSTRING (Tag Extraction)

### แนวคิด: แยกข้อความระหว่างแท็กเปิดและแท็กปิด

การ parse XML แบบเรียบง่าย (element เดียว ไม่มีการซ้อนซับซ้อน) ทำได้โดยแยกข้อความด้วยแท็กเปิดก่อน
แล้วแยกด้วยแท็กปิดอีกที เพื่อให้เหลือแต่เนื้อหาที่อยู่ตรงกลาง

### ตัวอย่าง: Parser สำหรับดึงค่าจาก XML Element

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. XPARSE1.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-XML-IN       PIC X(80) VALUE
           "<customer><name>SOMCHAI</name><age>30</age></customer>".
       01  WS-JUNK         PIC X(40).
       01  WS-NAME-OUT     PIC X(20).
       01  WS-AGE-OUT      PIC X(10).
       PROCEDURE DIVISION.
      *> Extract the text between <name> and </name>: split on
      *> the opening tag first, then the closing tag.
           UNSTRING WS-XML-IN DELIMITED BY "<name>"
               INTO WS-JUNK WS-NAME-OUT
           UNSTRING WS-NAME-OUT DELIMITED BY "</name>"
               INTO WS-NAME-OUT

           UNSTRING WS-XML-IN DELIMITED BY "<age>"
               INTO WS-JUNK WS-AGE-OUT
           UNSTRING WS-AGE-OUT DELIMITED BY "</age>"
               INTO WS-AGE-OUT

           DISPLAY "name = " FUNCTION TRIM(WS-NAME-OUT)
           DISPLAY "age  = " FUNCTION TRIM(WS-AGE-OUT)
           STOP RUN.
```

**ผลลัพธ์:**

```
name = SOMCHAI
age  = 30
```

### อธิบายจุดสำคัญ

- ขั้นแรก `UNSTRING WS-XML-IN DELIMITED BY "<name>"` แยกข้อความออกเป็น 2 ส่วน: ส่วนก่อนแท็ก
  `<name>` (เก็บทิ้งใน `WS-JUNK`) และส่วนหลังแท็ก `<name>` (เก็บใน `WS-NAME-OUT` ซึ่งตอนนี้ยังมี
  `</name>` และข้อมูลอื่นต่อท้ายอยู่)
- ขั้นที่สอง `UNSTRING WS-NAME-OUT DELIMITED BY "</name>"` แยกเอาเฉพาะส่วนก่อนแท็กปิดออกมา ได้ค่า
  "SOMCHAI" ที่สะอาดจริง ๆ
- ทำซ้ำรูปแบบเดียวกันสำหรับแต่ละ field ที่ต้องการดึงค่า (`<age>...</age>` ในตัวอย่างนี้)

### ข้อควรระวัง

- เทคนิคนี้ใช้ได้กับ XML ที่มีโครงสร้างเรียบง่ายและ**ไม่มี nested element ที่ใช้ชื่อแท็กซ้ำกัน**
  หากมี `<name>` ปรากฏมากกว่าหนึ่งครั้งในเอกสาร (เช่นซ้อนอยู่ในหลาย parent element) วิธีนี้จะดึงเฉพาะ
  ค่าแรกที่พบเท่านั้น ต้องออกแบบ parser ที่ซับซ้อนกว่านี้สำหรับกรณีนั้น
- ต้องมั่นใจว่าแท็กที่ระบุใน `DELIMITED BY` สะกดตรงกับ XML ต้นฉบับทุกตัวอักษร รวมถึงตัวพิมพ์ใหญ่-เล็ก
  (XML เป็น case-sensitive ต่างจาก COBOL keyword ที่ไม่สนตัวพิมพ์ใหญ่เล็ก)

### แบบฝึกหัดที่ 456.1

**โจทย์**: จงเพิ่มการดึงค่า `<city>` จาก XML `<customer><name>MALEE</name><city>CHIANGMAI</city>
</customer>` ด้วยเทคนิคเดียวกัน

**เฉลย**:

```cobol
       01  WS-CITY-OUT     PIC X(20).
       ...
           UNSTRING WS-XML-IN DELIMITED BY "<city>"
               INTO WS-JUNK WS-CITY-OUT
           UNSTRING WS-CITY-OUT DELIMITED BY "</city>"
               INTO WS-CITY-OUT
           DISPLAY "city = " FUNCTION TRIM(WS-CITY-OUT)
```

---

## ขั้นตอนที่ 457: การ Parse ค่าจาก XML Attribute ด้วยมือ

### แนวคิด: Attribute อยู่ในแท็กเปิด ต้องแยกคนละวิธีกับเนื้อหา Element

การดึงค่า attribute (เช่น `qty` จาก `<item qty="10">APPLE</item>`) ต้องแยกที่ตำแหน่ง `qty="` แล้ว
แยกต่อด้วยเครื่องหมายคำพูดปิด `"`

### ตัวอย่าง: Parser สำหรับดึงค่า Attribute และเนื้อหา Element

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. XATTR1.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TAG          PIC X(40)
           VALUE '<item qty="10">APPLE</item>'.
       01  WS-JUNK         PIC X(40).
       01  WS-QTY-OUT      PIC X(10).
       01  WS-NAME-OUT     PIC X(20).
       PROCEDURE DIVISION.
      *> Get the attribute value: split on qty=" then on the
      *> closing quote.
           UNSTRING WS-TAG DELIMITED BY 'qty="'
               INTO WS-JUNK WS-QTY-OUT
           UNSTRING WS-QTY-OUT DELIMITED BY '"'
               INTO WS-QTY-OUT

      *> Get the element text: split on > then on </item>.
           UNSTRING WS-TAG DELIMITED BY '">'
               INTO WS-JUNK WS-NAME-OUT
           UNSTRING WS-NAME-OUT DELIMITED BY "</item>"
               INTO WS-NAME-OUT

           DISPLAY "qty  = " FUNCTION TRIM(WS-QTY-OUT)
           DISPLAY "name = " FUNCTION TRIM(WS-NAME-OUT)
           STOP RUN.
```

**ผลลัพธ์:**

```
qty  = 10
name = APPLE
```

### อธิบายจุดสำคัญ

- การดึง attribute ใช้หลักการเดียวกับการดึงเนื้อหา element (แยกสองรอบ) เพียงแต่ delimiter เปลี่ยนเป็น
  `qty="` (จุดเริ่มค่า attribute) และ `"` (เครื่องหมายคำพูดปิด attribute) แทน
- การดึงเนื้อหา element ในตัวอย่างนี้ใช้ `'">'` (เครื่องหมายคำพูดปิด attribute ตามด้วย `>` ที่ปิดแท็ก
  เปิด) เป็นจุดเริ่มต้น เพื่อข้ามส่วน attribute ทั้งหมดไปแล้วเริ่มอ่านเนื้อหาจริงของ element

### ข้อควรระวัง

- เทคนิคนี้ตั้งสมมติฐานว่าลำดับ attribute และรูปแบบเครื่องหมายคำพูด (คู่ ไม่ใช่เดี่ยว) เป็นไปตาม
  รูปแบบที่คาดไว้เป๊ะ หาก XML ต้นฉบับใช้เครื่องหมายคำพูดเดี่ยว (`qty='10'`) หรือมีลำดับ attribute
  ต่างไป ต้องปรับ delimiter ให้สอดคล้องกัน
- หากมีหลาย attribute ในแท็กเดียว (เช่น `<item qty="10" unit="kg">`) ต้องแยกทีละ attribute ด้วย
  delimiter ที่ตรงกับตำแหน่งของแต่ละตัว งานนี้จะซับซ้อนขึ้นเรื่อย ๆ ตามจำนวน attribute — เป็นเหตุผล
  สำคัญที่ `XML PARSE` มาตรฐาน (เมื่อคอมไพเลอร์รองรับ) จะสะดวกกว่ามากในกรณีที่ซับซ้อน

### แบบฝึกหัดที่ 457.1

**โจทย์**: จากตัวอย่าง `XATTR1` จงเพิ่มการดึงค่า attribute ตัวที่สองชื่อ `unit` จาก
`'<item qty="10" unit="kg">APPLE</item>'`

**เฉลย**:

```cobol
       01  WS-UNIT-OUT     PIC X(10).
       ...
           UNSTRING WS-TAG DELIMITED BY 'unit="'
               INTO WS-JUNK WS-UNIT-OUT
           UNSTRING WS-UNIT-OUT DELIMITED BY '"'
               INTO WS-UNIT-OUT
           DISPLAY "unit = " FUNCTION TRIM(WS-UNIT-OUT)
```

---

## ขั้นตอนที่ 458: แนวคิด Namespace และ Nested Element ใน XML

### XML Namespace คืออะไร (แนวคิดเชิงทฤษฎี)

ระบบ XML องค์กรขนาดใหญ่มักใช้ **Namespace** เพื่อป้องกันชื่อ element ชนกันเมื่อรวมเอกสาร XML จาก
หลายแหล่งเข้าด้วยกัน โดยใส่ prefix นำหน้าชื่อ element เช่น:

```xml
<bank:customer xmlns:bank="http://example.com/banking">
    <bank:name>SOMCHAI</bank:name>
    <bank:account>1234567890</bank:account>
</bank:customer>
```

`xmlns:bank="..."` ประกาศว่า prefix `bank:` ผูกกับ URI เฉพาะ ทำให้ element `<bank:customer>` ไม่
ปะปนกับ `<customer>` จาก namespace อื่นแม้จะอยู่ในเอกสารเดียวกัน มาตรฐาน `XML GENERATE` รองรับการสร้าง
namespace ผ่าน clause `NAMESPACE` ที่แสดงไว้ในขั้นตอนที่ 451 (เมื่อคอมไพเลอร์รองรับ)

### ตัวอย่าง: Parse ค่าจาก Element ที่มี Namespace Prefix ด้วยมือ

แม้จะมี prefix นำหน้า เทคนิคการดึงค่าด้วย `UNSTRING` ยังคงใช้หลักการเดิมได้ เพียงระบุชื่อแท็กให้รวม
prefix ด้วย:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. XNAMESPACE.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-XML-IN       PIC X(120) VALUE
           "<bank:customer><bank:name>SOMCHAI</bank:name>"
           & "<bank:account>1234567890</bank:account>"
           & "</bank:customer>".
       01  WS-JUNK         PIC X(60).
       01  WS-NAME-OUT     PIC X(20).
       01  WS-ACCT-OUT     PIC X(20).
       PROCEDURE DIVISION.
           UNSTRING WS-XML-IN DELIMITED BY "<bank:name>"
               INTO WS-JUNK WS-NAME-OUT
           UNSTRING WS-NAME-OUT DELIMITED BY "</bank:name>"
               INTO WS-NAME-OUT

           UNSTRING WS-XML-IN DELIMITED BY "<bank:account>"
               INTO WS-JUNK WS-ACCT-OUT
           UNSTRING WS-ACCT-OUT DELIMITED BY "</bank:account>"
               INTO WS-ACCT-OUT

           DISPLAY "name    = " FUNCTION TRIM(WS-NAME-OUT)
           DISPLAY "account = " FUNCTION TRIM(WS-ACCT-OUT)
           STOP RUN.
```

**ผลลัพธ์:**

```
name    = SOMCHAI
account = 1234567890
```

### อธิบายจุดสำคัญ

- `&` ที่ต้นบรรทัดต่อเนื่องใน COBOL literal เป็นตัวเชื่อม literal หลายบรรทัดเข้าด้วยกันเป็นข้อความ
  เดียว (concatenation ที่ระดับ source code ไม่ใช่ runtime) ใช้เมื่อ literal ยาวเกิน 1 บรรทัดในพื้นที่
  fixed-format คอลัมน์ 8-72
- เทคนิค `UNSTRING` แบบเดิมยังใช้ได้กับชื่อแท็กที่มี prefix เพียงแค่ระบุ delimiter ให้รวม prefix เข้า
  ไปด้วย (`"<bank:name>"` แทน `"<name>"`) — แสดงให้เห็นว่าหลักการพื้นฐานไม่เปลี่ยน เพียงต้องรู้ชื่อแท็ก
  ที่แน่นอนล่วงหน้าเท่านั้น

### ข้อควรระวัง

- เทคนิคนี้ยังคงมีข้อจำกัดเดิม (ใช้ไม่ได้กับ nested structure ที่ซับซ้อนมาก หรือเมื่อไม่ทราบ prefix
  ที่แน่นอนล่วงหน้า) ในระบบจริงที่ต้องจัดการ Namespace อย่างเข้มงวด (ตรวจสอบ URI, รองรับ prefix ที่
  เปลี่ยนได้) จำเป็นต้องใช้ XML parser แบบเต็มรูปแบบ (เช่น `XML PARSE` มาตรฐาน หรือไลบรารีภายนอก)
- การเขียน XML ด้วยมือที่มี Namespace ต้องระบุ `xmlns:prefix="URI"` ในแท็กราก (root element) เองด้วย
  ตัวอย่างข้างต้นเน้นการ parse เท่านั้น ไม่ได้แสดงการสร้าง XML ที่มี namespace declaration ครบถ้วน

### แบบฝึกหัดที่ 458.1

**โจทย์**: จงอธิบายว่าทำไมการมี Namespace (เช่น prefix `bank:`) จึงมีประโยชน์เมื่อต้องรวมเอกสาร XML
จากหลายระบบเข้าด้วยกัน

**เฉลย**: หากสองระบบต่างมี element ชื่อ `<customer>` ที่มีความหมายต่างกัน (เช่น ระบบธนาคารกับระบบ
ประกันภัย) การรวมเอกสาร XML จากทั้งสองระบบเข้าด้วยกันโดยไม่มี Namespace จะทำให้เกิดความกำกวมว่า
`<customer>` ตัวไหนเป็นของระบบใด การใส่ prefix (เช่น `<bank:customer>` และ `<insurance:customer>`)
ที่ผูกกับ Namespace URI ที่ต่างกันทำให้แยกความหมายได้ชัดเจน แม้ชื่อ element ท้องถิ่น (local name)
จะซ้ำกันก็ตาม ป้องกันปัญหาการชนกันของชื่อ (naming collision) เมื่อระบบขนาดใหญ่ต้องบูรณาการข้อมูลจาก
หลายแหล่ง

---

## ขั้นตอนที่ 459: ข้อจำกัดของเทคนิค Manual เทียบกับ XML GENERATE/PARSE จริง

### สรุปข้อจำกัดที่ต้องตระหนัก

| ประเด็น | เทคนิคสร้างเอง (STRING/UNSTRING) | XML GENERATE/PARSE (มาตรฐาน) |
|---|---|---|
| Nested element หลายชั้นที่ซับซ้อน | ทำได้ยากมาก ต้องเขียนโค้ดแยกแต่ละชั้นเอง | รองรับอัตโนมัติตามโครงสร้างกลุ่มข้อมูล COBOL |
| Namespace ที่ซับซ้อน (หลาย prefix, default namespace) | ต้องจัดการเองทั้งหมด เสี่ยงผิดพลาดสูง | รองรับผ่าน clause `NAMESPACE` |
| การตรวจสอบ well-formedness (โครงสร้างถูกต้อง) | ไม่มีการตรวจสอบอัตโนมัติ | ตรวจสอบและแจ้ง `ON EXCEPTION` หากผิดรูปแบบ |
| Escape/Unescape อักขระพิเศษครบถ้วน | ต้องเขียนโค้ดจัดการเองทั้งหมด | จัดการอัตโนมัติตามมาตรฐาน XML |
| CDATA sections, comments, processing instructions | ไม่รองรับโดยง่าย | รองรับตามมาตรฐาน XML เต็มรูปแบบ |

### ไวยากรณ์อ้างอิงแบบเต็มของ XML PARSE (สำหรับใช้เมื่อคอมไพเลอร์รองรับ)

**หมายเหตุ**: โค้ดต่อไปนี้เป็นไวยากรณ์อ้างอิงตามมาตรฐาน **ยังไม่ได้ทดสอบว่าคอมไพล์ผ่านในบิลด์ปัจจุบัน**
(ยืนยันแล้วว่า compile ไม่ผ่านตามขั้นตอนที่ 452) แสดงไว้เพื่อการอ้างอิงเมื่อทำงานกับคอมไพเลอร์ที่รองรับ:

```
      *> Reference syntax only - NOT runnable on this build.
      *> Confirmed to fail with: "compiler is not configured
      *> to support XML" (see step 452).
       01  WS-XML-DOC      PIC X(200).
       01  WS-CUSTOMER.
           05  CUST-NAME   PIC X(20).
           05  CUST-AGE    PIC 9(3).

       XML PARSE WS-XML-DOC
           PROCESSING PROCEDURE HANDLE-XML-EVENT
           ON EXCEPTION
               DISPLAY "XML PARSE failed"
       END-XML

       HANDLE-XML-EVENT.
      *> XML-EVENT is a special register telling us what kind
      *> of XML token was just encountered (start of element,
      *> end of element, character data, etc.)
           EVALUATE XML-EVENT
               WHEN "START-OF-ELEMENT"
                   DISPLAY "Entering element: " XML-TEXT
               WHEN "CONTENT-CHARACTERS"
                   DISPLAY "Content: " XML-TEXT
               WHEN "END-OF-ELEMENT"
                   DISPLAY "Leaving element: " XML-TEXT
           END-EVALUATE.
```

### อธิบายจุดสำคัญ

- `PROCESSING PROCEDURE HANDLE-XML-EVENT`: ระบุชื่อ paragraph ที่จะถูกเรียกกลับซ้ำ ๆ ระหว่างการ
  parse — แนวคิด callback ที่คล้ายกับที่เรียนไปแล้วใน Part 043 (ขั้นตอนที่ 428) แต่ในบริบทนี้เป็นกลไก
  ภายในของคอมไพเลอร์เอง ไม่ใช่ที่เราสร้างเอง
- `XML-EVENT` และ `XML-TEXT` เป็น special register ที่ COBOL เตรียมไว้ให้อัตโนมัติเมื่อใช้
  `XML PARSE` — บอกประเภทของ event ที่เพิ่งเกิดขึ้นและข้อความที่เกี่ยวข้อง

### ข้อควรระวัง

- **อย่านำโค้ดในกล่องอ้างอิงไป compile ในสภาพแวดล้อมนี้โดยตรง** จะได้ error เหมือนขั้นตอนที่ 452
- กลไก event-driven ของ `XML PARSE` ซับซ้อนกว่า `JSON PARSE` พอสมควร (ต้องเขียน paragraph จัดการ
  event เอง) แต่ก็ยืดหยุ่นกว่าในการรับมือกับโครงสร้าง XML ที่หลากหลายและซับซ้อน หากมีโอกาสทำงานกับ
  คอมไพเลอร์ที่รองรับจริง ควรศึกษา special register ที่เกี่ยวข้องเพิ่มเติม (`XML-EVENT`, `XML-TEXT`,
  `XML-NTEXT`, `XML-CODE` ฯลฯ) จากคู่มือของคอมไพเลอร์นั้น ๆ

### แบบฝึกหัดที่ 459.1

**โจทย์**: จงเปรียบเทียบว่าการเขียน parser ด้วยมือ (`UNSTRING`) กับกลไก `PROCESSING PROCEDURE` ของ
`XML PARSE` มาตรฐาน แบบไหนเหมาะกับ XML ที่มีโครงสร้างเปลี่ยนแปลงบ่อยมากกว่ากัน เพราะเหตุใด

**เฉลย**: กลไก `PROCESSING PROCEDURE` ของ `XML PARSE` มาตรฐานเหมาะกับ XML ที่มีโครงสร้างเปลี่ยนแปลง
บ่อยหรือซับซ้อนมากกว่า เพราะมันประมวลผลทีละ event โดยไม่ผูกติดกับตำแหน่งที่ตายตัวของแท็กในข้อความ
(ไม่ต้องรู้ล่วงหน้าว่าแท็กอยู่ตำแหน่งไหน) ในขณะที่เทคนิค `UNSTRING` ด้วยมือต้องอาศัยการรู้ตำแหน่ง/ลำดับ
ของแท็กที่ค่อนข้างตายตัวในการเขียน delimiter ให้ตรง หากโครงสร้าง XML เปลี่ยนไป (เช่น ลำดับ element
สลับกัน หรือมี element ใหม่แทรกเข้ามา) โค้ด `UNSTRING` ที่เขียนไว้อาจต้องแก้ไขใหม่ทั้งหมด ในขณะที่
`PROCESSING PROCEDURE` จะยังคงทำงานถูกต้องตราบใดที่ logic การจัดการ event ครอบคลุมกรณีที่เป็นไปได้
ทั้งหมด

---

## ขั้นตอนที่ 460: ตัวอย่างรวม — โปรแกรมส่งออก Record เป็นไฟล์ XML

### ภาพรวมโปรแกรมสุดท้ายของ Part นี้

เราจะรวมเทคนิคทั้งหมด: การสร้าง XML ด้วย `STRING`, การ escape อักขระพิเศษ, และการเขียนผลลัพธ์ลงไฟล์
จริง — โดยนำ**บทเรียนสำคัญจาก Part 045** (ต้อง `MOVE SPACES` ล้าง FD record ก่อน `STRING` เสมอ
เพื่อป้องกัน File Status 71) มาใช้ที่นี่ด้วยเช่นกัน เนื่องจากเป็นข้อกำหนดทั่วไปของ GnuCOBOL ไม่ได้
จำกัดเฉพาะการเขียน JSON เท่านั้น

### โปรแกรมสมบูรณ์: ส่งออก Record ลูกค้าเป็นไฟล์ XML

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. XFILE1.
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT OUT-FILE ASSIGN TO "CUSTOMER460.XML"
               ORGANIZATION IS LINE SEQUENTIAL.
       DATA DIVISION.
       FILE SECTION.
       FD  OUT-FILE.
       01  OUT-LINE            PIC X(150).
       WORKING-STORAGE SECTION.
       01  WS-NAME             PIC X(15) VALUE "SOMCHAI JAIDEE".
       01  WS-AGE              PIC 9(3)  VALUE 30.
       01  WS-AGE-EDIT         PIC ZZ9.
       PROCEDURE DIVISION.
           MOVE WS-AGE TO WS-AGE-EDIT
           OPEN OUTPUT OUT-FILE
      *> Clear the FD record before STRING fills only part of
      *> it - required to avoid File Status 71 (see Part 045,
      *> step 450, for the full explanation of this pitfall).
           MOVE SPACES TO OUT-LINE
           STRING "<customer>" DELIMITED BY SIZE
                   "<name>" DELIMITED BY SIZE
                   FUNCTION TRIM(WS-NAME) DELIMITED BY SIZE
                   "</name>" DELIMITED BY SIZE
                   "<age>" DELIMITED BY SIZE
                   FUNCTION TRIM(WS-AGE-EDIT) DELIMITED BY SIZE
                   "</age>" DELIMITED BY SIZE
                   "</customer>" DELIMITED BY SIZE
               INTO OUT-LINE
           WRITE OUT-LINE
           CLOSE OUT-FILE
           DISPLAY "Wrote CUSTOMER460.XML"
           STOP RUN.
```

**ผลลัพธ์:**

```
Wrote CUSTOMER460.XML
```

เนื้อหาไฟล์ `CUSTOMER460.XML` ที่ได้:

```
<customer><name>SOMCHAI JAIDEE</name><age>30</age></customer>
```

### อธิบายจุดสำคัญ

- โครงสร้างโปรแกรมเหมือนกับตัวอย่างสรุปของ Part 045 (ขั้นตอนที่ 450) ทุกประการ เปลี่ยนเพียงรูปแบบ
  ข้อความที่สร้างจาก JSON เป็น XML — แสดงให้เห็นว่าหลักการพื้นฐาน (สร้างข้อความด้วย `STRING`, ล้าง
  record ก่อนเขียนไฟล์) ใช้ร่วมกันได้ไม่ว่าจะเป็นรูปแบบข้อมูลใดก็ตาม
- ไฟล์ผลลัพธ์ที่ได้เป็น XML ที่ถูกต้องตามหลัก well-formed (ทุกแท็กเปิดมีแท็กปิดคู่กัน) พร้อมนำไปใช้
  งานต่อกับระบบอื่นที่รับข้อมูลรูปแบบ XML ได้ทันที

### ข้อควรระวัง

- ตัวอย่างนี้ไม่มีการประกาศ XML declaration (`<?xml version="1.0" encoding="UTF-8"?>`) ซึ่งระบบ
  ปลายทางบางระบบอาจต้องการ หากจำเป็นสามารถเพิ่มเป็นบรรทัด `STRING` แรกก่อนเนื้อหาหลักได้
- เช่นเดียวกับข้อควรระวังใน Part 045: หากคอมไพเลอร์ปลายทางที่ใช้งานจริงรองรับ `XML GENERATE` ควรใช้
  คำสั่งมาตรฐานแทนเทคนิคด้วยมือเสมอ เพื่อความถูกต้องและดูแลรักษาง่ายกว่า

### แบบฝึกหัดที่ 460.1

**โจทย์**: จงปรับโปรแกรม `XFILE1` ให้เพิ่มบรรทัด XML declaration `<?xml version="1.0"
encoding="UTF-8"?>` ไว้ก่อนเนื้อหา `<customer>`

**เฉลย**:

```cobol
           MOVE SPACES TO OUT-LINE
           STRING '<?xml version="1.0" encoding="UTF-8"?>'
                   DELIMITED BY SIZE
                   "<customer>" DELIMITED BY SIZE
                   "<name>" DELIMITED BY SIZE
                   FUNCTION TRIM(WS-NAME) DELIMITED BY SIZE
                   "</name>" DELIMITED BY SIZE
                   "<age>" DELIMITED BY SIZE
                   FUNCTION TRIM(WS-AGE-EDIT) DELIMITED BY SIZE
                   "</age>" DELIMITED BY SIZE
                   "</customer>" DELIMITED BY SIZE
               INTO OUT-LINE
```

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้เรื่อง XML ใน COBOL อย่างครบถ้วนและซื่อตรงต่อข้อเท็จจริงเช่นเดียวกับ Part 045:

- ภาพรวมไวยากรณ์มาตรฐานของ `XML GENERATE`/`XML PARSE` รวมถึงกลไก event-driven ผ่าน
  `PROCESSING PROCEDURE` ที่แตกต่างจาก `JSON PARSE`
- **การทดสอบจริงและข้อค้นพบสำคัญ**: บิลด์ GnuCOBOL ที่ใช้ในหลักสูตรนี้ปิดใช้งาน XML library เช่นเดียว
  กับ JSON library แม้ระบบจะมี `libxml2` ติดตั้งอยู่จริงก็ตาม — เป็นบทเรียนสำคัญว่า "มีไลบรารีในระบบ"
  กับ "คอมไพเลอร์ถูกตั้งค่าให้ใช้ไลบรารีนั้นได้" เป็นคนละเรื่องกัน
- เทคนิคสร้าง XML ด้วยมือผ่าน `STRING`: element เดี่ยว, escape อักขระพิเศษ 5 ตัวของ XML, repeating
  element พร้อม attribute จากตาราง `OCCURS`
- เทคนิค parse XML ด้วยมือผ่าน `UNSTRING`: การดึงเนื้อหา element, การดึงค่า attribute, และการรับมือ
  กับ Namespace prefix เบื้องต้น
- ตารางเปรียบเทียบข้อจำกัดของเทคนิคสร้างเองกับ `XML GENERATE`/`XML PARSE` มาตรฐาน
- ตัวอย่างรวมส่งออก record ลูกค้าเป็นไฟล์ XML โดยนำบทเรียนเรื่อง `MOVE SPACES` ก่อน `STRING` จาก
  Part 045 มาประยุกต์ใช้ซ้ำ

Part นี้ปิดท้ายเรื่องการแลกเปลี่ยนข้อมูลกับระบบภายนอกในรูปแบบมาตรฐาน (JSON และ XML) ใน Part ถัดไปเรา
จะกลับมาสู่หัวข้อพื้นฐานที่ทำงานได้เต็มรูปแบบในทุกบิลด์ของ GnuCOBOL: **การจัดการวันที่และเวลาขั้นสูง**
ซึ่งต่อยอดจาก Intrinsic Functions ด้านวันที่ที่เรียนไปแล้วใน Part 036 ไปสู่การตรวจสอบความถูกต้องของ
วันที่ การคำนวณผลต่างระหว่างวันที่ การตรวจสอบปีอธิกสุรทิน และการจัดรูปแบบวันที่สำหรับแสดงผล

**[← กลับไป Part 045: JSON GENERATE และ JSON PARSE](part-045-json-generate-parse.md)** | **[ไปยัง Part 047: การจัดการวันที่และเวลาขั้นสูง →](part-047-advanced-date-time.md)**
