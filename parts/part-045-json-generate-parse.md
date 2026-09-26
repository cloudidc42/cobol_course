# Part 045: JSON GENERATE และ JSON PARSE ใน COBOL สมัยใหม่ (ขั้นตอนที่ 441–450)

## คำนำของ Part นี้

**JSON** (JavaScript Object Notation) คือรูปแบบข้อมูลที่ใช้แพร่หลายที่สุดในการสื่อสารระหว่างระบบผ่าน
REST API ในปัจจุบัน มาตรฐาน **COBOL-2014** จึงเพิ่มคำสั่ง `JSON GENERATE` (แปลงข้อมูล COBOL เป็น
ข้อความ JSON) และ `JSON PARSE` (แปลงข้อความ JSON กลับเป็นข้อมูล COBOL) เข้ามาเพื่อให้ COBOL ยุคใหม่
เชื่อมต่อกับระบบภายนอกได้โดยตรงโดยไม่ต้องเขียนตัวแปลงเอง — เนื้อหานี้เชื่อมโยงกับสิ่งที่กล่าวถึงไว้
ตั้งแต่ Part 001 (ขั้นตอนที่ 6) ว่า COBOL-2014 เพิ่มความสามารถด้าน JSON เข้ามา

**อย่างไรก็ตาม** ก่อนเขียนเนื้อหาสอน Part นี้ เราได้ทำตามหลักการสำคัญของหลักสูตร: **ทดสอบ feature
จริงก่อนสอน** และผลการทดสอบเผยให้เห็นข้อเท็จจริงที่สำคัญมาก ซึ่งจะอธิบายละเอียดในขั้นตอนที่ 442 —
บิลด์ GnuCOBOL ที่ใช้ในหลักสูตรนี้ **ไม่ได้เปิดใช้งาน JSON library** ทำให้ `JSON GENERATE`/`JSON PARSE`
compile ไม่ผ่านเลย Part นี้จึงออกแบบใหม่ให้สอนทั้ง**ไวยากรณ์มาตรฐาน**ตามที่ COBOL-2014 กำหนด (เพื่อให้
คุณอ่านและเข้าใจโค้ดที่พบในองค์กรที่ใช้คอมไพเลอร์ที่รองรับเต็มรูปแบบ เช่น IBM Enterprise COBOL หรือ
Micro Focus) **และเทคนิคสร้าง/แยกวิเคราะห์ JSON ด้วยมือ**ผ่าน `STRING`/`UNSTRING`/`INSPECT` ที่คุณ
สามารถใช้งานได้จริงทันทีในทุกสภาพแวดล้อม COBOL แม้ไม่มี JSON library ติดตั้งไว้เลยก็ตาม

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน ทุกตัวอย่างที่แสดงว่า
> "compile และรันได้" ผ่านการทดสอบจริงด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้ว ส่วน
> ตัวอย่างที่แสดง error จริงก็เป็นข้อความ error จริงที่ได้จากการ compile จริงเช่นกัน

---

## ขั้นตอนที่ 441: JSON คืออะไร และภาพรวมไวยากรณ์ JSON GENERATE/PARSE ตามมาตรฐาน COBOL-2014

### JSON ในรูปแบบง่าย ๆ

JSON เก็บข้อมูลเป็นคู่ `"key":value` ภายในวงเล็บปีกกา `{ }` สำหรับ object และวงเล็บเหลี่ยม `[ ]`
สำหรับ array ตัวอย่างเช่น:

```json
{"name":"SOMCHAI JAIDEE","age":30,"city":"BANGKOK"}
```

### ไวยากรณ์มาตรฐานของ JSON GENERATE (ตาม COBOL-2014)

ตามมาตรฐาน COBOL-2014 คำสั่ง `JSON GENERATE` มีรูปแบบทั่วไปดังนี้:

```
JSON GENERATE identifier-1 FROM identifier-2
    [ NAME OF identifier-3 IS literal-1 ]...
    [ SUPPRESS [ identifier-4 ] ]
    [ ON EXCEPTION statement-1 ]
    [ NOT ON EXCEPTION statement-2 ]
    [ END-JSON ]
```

โดย `identifier-1` คือตัวแปรข้อความปลายทางที่จะเก็บผลลัพธ์ JSON และ `identifier-2` คือกลุ่มข้อมูล
COBOL ที่จะถูกแปลง ส่วน `JSON PARSE` ก็มีรูปแบบคล้ายกันแต่ทำงานย้อนกลับ (จาก JSON เป็นข้อมูล COBOL):

```
JSON PARSE identifier-1 INTO identifier-2
    [ NAME OF identifier-3 IS literal-1 ]...
    [ WITH DETECTION ]
    [ ON EXCEPTION statement-1 ]
    [ NOT ON EXCEPTION statement-2 ]
    [ END-JSON ]
```

### ตัวอย่างที่ต้องการทดสอบ (แสดงไวยากรณ์อ้างอิงตามมาตรฐาน)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. JSONTEST.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-OUT PIC X(200).
       01  WS-REC.
           05 WS-NAME PIC X(10) VALUE "JOHN".
           05 WS-AGE  PIC 9(3) VALUE 30.
       PROCEDURE DIVISION.
           JSON GENERATE WS-OUT FROM WS-REC
           DISPLAY WS-OUT
           STOP RUN.
```

ทดสอบ compile จริง:

```
cobc -x -o jsontest jsontest.cob
```

### อธิบายจุดสำคัญ

- ไวยากรณ์ `JSON GENERATE target FROM source` เรียบง่ายมาก — ระบุแค่ตัวแปรปลายทางและกลุ่มข้อมูล
  ต้นทาง คอมไพเลอร์ (ที่รองรับ) จะแปลงชื่อ field เป็น key และค่าของ field เป็น value ให้อัตโนมัติ
- `NAME OF identifier IS literal` ใช้เมื่อต้องการตั้งชื่อ key ใน JSON ให้ต่างจากชื่อ field ใน COBOL
  (เช่น field ชื่อ `WS-CUST-NAME` แต่ต้องการ key ใน JSON เป็น `"name"`)
- นี่คือความสามารถที่ทรงพลังมาก **หากคอมไพเลอร์รองรับ** — ไม่ต้องเขียนโค้ดแปลงข้อมูลเองเลย

### ข้อควรระวัง

- ไวยากรณ์นี้เป็นการอ้างอิงตามมาตรฐาน COBOL-2014 **ยังไม่ได้ยืนยันว่าทำงานได้จริงในบิลด์นี้** — ผล
  การทดสอบจริงจะแสดงในขั้นตอนถัดไป โปรดอย่าเพิ่งนำไปใช้ในระบบจริงโดยไม่ทดสอบกับคอมไพเลอร์ที่คุณใช้
  งานจริงก่อนเสมอ

### แบบฝึกหัดที่ 441.1

**โจทย์**: จงเขียนโครงสร้างข้อมูล COBOL (WORKING-STORAGE) สำหรับ record ลูกค้าที่มี field
`CUST-ID` (ตัวเลข 5 หลัก) และ `CUST-NAME` (ข้อความ 20 ตัวอักษร) พร้อมคาดเดาว่าถ้า `JSON GENERATE`
ทำงานได้จริง ผลลัพธ์ JSON จะมีหน้าตาอย่างไร

**เฉลย**:

```cobol
       01  WS-CUSTOMER.
           05  CUST-ID     PIC 9(5) VALUE 12345.
           05  CUST-NAME   PIC X(20) VALUE "SOMCHAI".
```

ผลลัพธ์ JSON ที่คาดว่าจะได้ (ตามมาตรฐาน หากคอมไพเลอร์รองรับ):
`{"CUST-ID":12345,"CUST-NAME":"SOMCHAI"}` (ชื่อ key จะตรงกับชื่อ field COBOL ทุกตัวอักษรโดยค่าเริ่มต้น
รวมถึงเครื่องหมายขีดกลาง เว้นแต่จะระบุ `NAME OF` เพื่อเปลี่ยนชื่อ)

---

## ขั้นตอนที่ 442: ทดสอบจริง — เหตุใด JSON GENERATE จึง Compile ไม่ผ่านในบิลด์นี้

### ผลการทดสอบจริง

เมื่อคอมไพล์โค้ดจากขั้นตอนที่ 441 จริงด้วย `cobc (GnuCOBOL) 4.0-early-dev.0` ได้รับ error ดังนี้:

```
jsontest.cob:11: error [-Werror]: compiler is not configured to support JSON
```

และเมื่อตรวจสอบข้อมูลของคอมไพเลอร์โดยตรงด้วยคำสั่ง:

```
cobc --info
```

พบบรรทัดยืนยันชัดเจน:

```
XML library              : disabled
JSON library             : disabled
```

### สาเหตุ: เป็นการตั้งค่าตอน Build คอมไพเลอร์เอง ไม่ใช่ปัญหาโค้ด

GnuCOBOL รองรับ `JSON GENERATE`/`JSON PARSE` โดยพึ่งพาไลบรารีภายนอก (เช่น cJSON) ที่ต้องถูกเชื่อม
(link) เข้ากับตัว `cobc` **ตั้งแต่ตอน build/compile ตัวคอมไพเลอร์เอง** ผ่าน configure flag เฉพาะ
หากแพ็กเกจ GnuCOBOL ที่ติดตั้งมาไม่ได้เปิดใช้งาน flag นี้ (เหมือนบิลด์ `4.0-early-dev.0` ที่มาจาก
แพ็กเกจ `gnucobol4` ของ Ubuntu ในสภาพแวดล้อมนี้) ความสามารถ JSON ทั้งหมดจะถูกปิดใช้งานสนิท **ไม่ว่า
โค้ด COBOL ของเราจะเขียนถูกต้องตามมาตรฐานแค่ไหนก็ตาม ก็ compile ไม่ผ่านเสมอ**

นี่คือความแตกต่างสำคัญจาก syntax error ทั่วไป — โค้ดในขั้นตอนที่ 441 นั้น**ถูกต้องตามหลักไวยากรณ์
COBOL-2014 ทุกประการ** ปัญหาอยู่ที่ "ความสามารถของตัวคอมไพเลอร์ที่ติดตั้งอยู่" ไม่ใช่ที่ตัวโค้ด

### วิธีตรวจสอบว่าคอมไพเลอร์ของคุณรองรับ JSON หรือไม่ (นำไปใช้ได้ทันที)

```
cobc --info | grep -i json
```

หากได้ผลลัพธ์ `JSON library : disabled` แสดงว่าคอมไพเลอร์ของคุณไม่รองรับเช่นเดียวกับบิลด์นี้ หากได้
`JSON library : cJSON` (หรือชื่อไลบรารีอื่นที่ไม่ใช่ disabled) แสดงว่ารองรับและสามารถใช้ไวยากรณ์จาก
ขั้นตอนที่ 441 ได้ตามปกติ

### อธิบายจุดสำคัญ

- **บทเรียนสำคัญที่สุดของขั้นตอนนี้ไม่ใช่เรื่อง JSON โดยตรง แต่คือวินัยการตรวจสอบ (verify) ความ
  สามารถของสภาพแวดล้อมก่อนใช้งาน feature ใด ๆ ในโปรดักชัน** โดยเฉพาะในโลก COBOL ที่มีคอมไพเลอร์
  หลายยี่ห้อ (GnuCOBOL, IBM Enterprise COBOL, Micro Focus) แต่ละยี่ห้อและแต่ละบิลด์อาจรองรับฟีเจอร์
  แตกต่างกันไป
- ข้อความ error `[-Werror]` บอกใบ้ว่าคำเตือนถูกยกระดับเป็น error (เนื่องจาก compile ด้วย option
  เข้มงวด) แต่ปัญหาแท้จริงคือประโยค "compiler is not configured to support JSON" ซึ่งชี้ชัดเจนว่า
  เป็นเรื่อง configuration ไม่ใช่ syntax

### ข้อควรระวัง

- อย่าลงทุนเวลาศึกษาไวยากรณ์ที่ซับซ้อนของ feature ใดโดยไม่ตรวจสอบก่อนว่าคอมไพเลอร์ที่จะใช้งานจริง
  รองรับหรือไม่ — การตรวจสอบเพียงคำสั่งเดียว (`cobc --info`) ประหยัดเวลาได้มากในระยะยาว
- หากองค์กรของคุณจำเป็นต้องใช้ `JSON GENERATE`/`JSON PARSE` จริง ๆ ทางเลือกคือ (1) เปลี่ยนไปใช้
  คอมไพเลอร์เชิงพาณิชย์ที่รองรับเต็มรูปแบบ (2) build GnuCOBOL จาก source เองพร้อมเปิดใช้งาน JSON
  library หรือ (3) ใช้เทคนิคสร้าง/แยกวิเคราะห์ JSON ด้วยมือตามที่จะสอนในขั้นตอนที่เหลือของ Part นี้

### แบบฝึกหัดที่ 442.1

**โจทย์**: จงอธิบายว่าทำไม error message "compiler is not configured to support JSON" จึงต่างจาก
"syntax error" ทั่วไป และความแตกต่างนี้สำคัญอย่างไรต่อวิธีแก้ปัญหา

**เฉลย**: "Syntax error" หมายความว่าโค้ดเขียนผิดไวยากรณ์ ต้องแก้ที่โค้ด แต่ "compiler is not
configured to support JSON" หมายความว่าโค้ด**ถูกต้อง**ตามไวยากรณ์แล้ว แต่ตัวคอมไพเลอร์ที่ใช้งานอยู่
ไม่มีความสามารถที่จะประมวลผลคำสั่งนั้นได้ (เพราะไม่ได้เชื่อมไลบรารีที่จำเป็นไว้ตอน build) การแก้ไข
จึงไม่ใช่การแก้โค้ด แต่ต้องเปลี่ยนคอมไพเลอร์/บิลด์ที่ใช้งาน หรือเปลี่ยนแนวทางไปใช้เทคนิคอื่นแทน

---

## ขั้นตอนที่ 443: กลยุทธ์รับมือ — สร้าง JSON ด้วยมือผ่าน STRING Statement

### แนวคิด: JSON ก็เป็นแค่ข้อความที่มีรูปแบบตายตัว

เมื่อไม่มี `JSON GENERATE` ให้ใช้ เราสามารถ**สร้างข้อความ JSON เองได้ด้วย `STRING`** ที่เรียนมาแล้วใน
Part 019 เพราะ JSON เป็นเพียงข้อความที่มีรูปแบบ `{"key":"value","key2":value2}` ตายตัว ไม่มีความลับใด
ซ่อนอยู่

### ตัวอย่าง: แปลง Record COBOL เป็น JSON ด้วยมือ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. JMAN1.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NAME         PIC X(15) VALUE "SOMCHAI JAIDEE".
       01  WS-AGE          PIC 9(3)  VALUE 30.
       01  WS-AGE-EDIT     PIC ZZ9.
       01  WS-NAME-TRIM    PIC X(15).
       01  WS-JSON         PIC X(100).
       PROCEDURE DIVISION.
           MOVE FUNCTION TRIM(WS-NAME) TO WS-NAME-TRIM
           MOVE WS-AGE TO WS-AGE-EDIT
           STRING '{"name":"'  DELIMITED BY SIZE
                   FUNCTION TRIM(WS-NAME-TRIM) DELIMITED BY SIZE
                   '","age":' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-AGE-EDIT) DELIMITED BY SIZE
                   '}'        DELIMITED BY SIZE
               INTO WS-JSON
           DISPLAY FUNCTION TRIM(WS-JSON)
           STOP RUN.
```

**ผลลัพธ์:**

```
{"name":"SOMCHAI JAIDEE","age":30}
```

### อธิบายจุดสำคัญ

- ใช้เครื่องหมายคำพูดเดี่ยว (`'...'`) เป็นตัวคั่น literal ที่มีเครื่องหมายคำพูดคู่ (`"`) อยู่ข้างใน —
  COBOL ไม่รองรับ backslash escape ในตัวอักษร (`\"` ใช้ไม่ได้) ดังนั้นการใช้ literal delimiter คนละ
  แบบกับเครื่องหมายคำพูดที่ต้องการแทรกจึงเป็นวิธีที่สะดวกที่สุด
- `FUNCTION TRIM(...)` ใช้ตัดช่องว่างส่วนเกินออกจากทั้งชื่อและตัวเลขก่อนนำไปต่อเป็น JSON — เลข
  `WS-AGE-EDIT` แบบ `PIC ZZ9` จะมีช่องว่างนำหน้าถ้าค่าน้อยกว่า 3 หลัก จึงต้อง trim ก่อนเสมอ
- `STRING ... DELIMITED BY SIZE ... INTO WS-JSON`: ต่อชิ้นส่วนข้อความทั้งหมดเข้าด้วยกันเป็นก้อนเดียว
  ตามลำดับที่เขียนไว้ (ทบทวนโดยละเอียดได้จาก Part 019)

### ข้อควรระวัง

- เทคนิคนี้เหมาะกับโครงสร้างข้อมูลแบบเรียบง่าย (flat object) เท่านั้น หากโครงสร้างซับซ้อนขึ้น (nested
  object, array) ต้องเขียนโค้ดเพิ่มเติมตามลำดับขั้นตอนที่จะสอนต่อไป
- ต้องระวังความยาวของตัวแปรปลายทาง (`WS-JSON PIC X(100)`) ให้เพียงพอกับข้อมูลจริงเสมอ มิฉะนั้น
  ข้อความ JSON อาจถูกตัดทอนโดยไม่มีการแจ้งเตือน (`STRING` จะหยุดเงียบ ๆ เมื่อพื้นที่ปลายทางเต็ม เว้น
  แต่จะใช้ `ON OVERFLOW` ตรวจสอบ ซึ่งเรียนไปแล้วใน Part 019)

### แบบฝึกหัดที่ 443.1

**โจทย์**: จงปรับโปรแกรม `JMAN1` ให้เพิ่ม field `"city":"BANGKOK"` เข้าไปในผลลัพธ์ JSON ด้วย

**เฉลย**:

```cobol
           STRING '{"name":"'  DELIMITED BY SIZE
                   FUNCTION TRIM(WS-NAME-TRIM) DELIMITED BY SIZE
                   '","age":' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-AGE-EDIT) DELIMITED BY SIZE
                   ',"city":"BANGKOK"}' DELIMITED BY SIZE
               INTO WS-JSON
```

---

## ขั้นตอนที่ 444: การ Escape อักขระพิเศษใน JSON String

### ปัญหา: ข้อมูลจริงอาจมีเครื่องหมายคำพูดปนอยู่

หากค่าข้อมูลที่จะใส่ใน JSON string มีเครื่องหมายคำพูดคู่ (`"`) ปนอยู่ (เช่น "She said "Hi" to Bob")
การนำไปแทรกใน JSON ตรง ๆ จะทำให้โครงสร้าง JSON เสียหาย (เครื่องหมายคำพูดจะถูกตีความว่าเป็นจุดจบของ
string ก่อนเวลาอันควร) จึงต้อง **escape** เครื่องหมายคำพูดด้วย backslash (`\"`) ก่อนเสมอ

### ตัวอย่าง: Escape เครื่องหมายคำพูดด้วยมือทีละตัวอักษร

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. JMAN2.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RAW          PIC X(30) VALUE 'She said "Hi" to Bob'.
       01  WS-ESCAPED      PIC X(40).
       01  WS-I            PIC 9(2).
       01  WS-J            PIC 9(2) VALUE 1.
       01  WS-CH           PIC X.
       PROCEDURE DIVISION.
           MOVE SPACES TO WS-ESCAPED
           PERFORM VARYING WS-I FROM 1 BY 1
               UNTIL WS-I > FUNCTION LENGTH(FUNCTION TRIM(WS-RAW))
               MOVE WS-RAW(WS-I:1) TO WS-CH
               IF WS-CH = '"'
                   MOVE '\' TO WS-ESCAPED(WS-J:1)
                   ADD 1 TO WS-J
                   MOVE '"' TO WS-ESCAPED(WS-J:1)
                   ADD 1 TO WS-J
               ELSE
                   MOVE WS-CH TO WS-ESCAPED(WS-J:1)
                   ADD 1 TO WS-J
               END-IF
           END-PERFORM
           DISPLAY "Original: " FUNCTION TRIM(WS-RAW)
           DISPLAY "Escaped : " FUNCTION TRIM(WS-ESCAPED)
           STOP RUN.
```

**ผลลัพธ์:**

```
Original: She said "Hi" to Bob
Escaped : She said \"Hi\" to Bob
```

### อธิบายจุดสำคัญ

- ลูป `PERFORM VARYING` เดินตรวจสอบข้อความทีละตัวอักษรด้วย reference modification (`WS-RAW(WS-I:1)`)
  ที่เรียนไปแล้วใน Part 019-021
- เมื่อพบเครื่องหมายคำพูดคู่ (`'"'`) จะแทรก backslash (`'\'`) ตามด้วยเครื่องหมายคำพูดเดิม แทนที่จะ
  copy ตัวอักษรตรง ๆ
- `WS-J` ทำหน้าที่เป็นตัวชี้ตำแหน่งที่จะเขียนถัดไปใน `WS-ESCAPED` — ต้องเพิ่มค่าเองทุกครั้งหลังเขียน
  ตัวอักษร (หรือสองตัวอักษรในกรณี escape)

### ข้อควรระวัง

- ในการใช้งานจริง ควร escape อักขระพิเศษอื่น ๆ ของ JSON ด้วยเช่นกัน ได้แก่ backslash เอง (`\` →
  `\\`), newline (`\n`), tab (`\t`), และอักขระควบคุมอื่น ๆ — ตัวอย่างนี้สาธิตเฉพาะเครื่องหมายคำพูด
  เพื่อความกระชับ แต่หลักการเดียวกันนำไปขยายกับอักขระอื่นได้โดยเพิ่มเงื่อนไข `IF`/`EVALUATE` เพิ่มเติม
- ควรกำหนดความกว้าง `WS-ESCAPED` ให้เผื่อพื้นที่มากกว่าความยาวต้นฉบับเสมอ เพราะการ escape ทำให้ความ
  ยาวข้อความเพิ่มขึ้น (แต่ละอักขระที่ escape จะกลายเป็น 2 ตัวอักษร)

### แบบฝึกหัดที่ 444.1

**โจทย์**: จงขยายโปรแกรม `JMAN2` ให้ escape เครื่องหมาย backslash (`\`) ด้วยเช่นกัน (กลายเป็น `\\`)

**เฉลย**: เพิ่มเงื่อนไขใน `EVALUATE`/`IF`:

```cobol
               EVALUATE WS-CH
                   WHEN '"'
                       MOVE '\' TO WS-ESCAPED(WS-J:1)
                       ADD 1 TO WS-J
                       MOVE '"' TO WS-ESCAPED(WS-J:1)
                       ADD 1 TO WS-J
                   WHEN '\'
                       MOVE '\' TO WS-ESCAPED(WS-J:1)
                       ADD 1 TO WS-J
                       MOVE '\' TO WS-ESCAPED(WS-J:1)
                       ADD 1 TO WS-J
                   WHEN OTHER
                       MOVE WS-CH TO WS-ESCAPED(WS-J:1)
                       ADD 1 TO WS-J
               END-EVALUATE
```

---

## ขั้นตอนที่ 445: แปลงตาราง (OCCURS) เป็น JSON Array ด้วยมือ

### แนวคิด: JSON Array คือรายการ Object คั่นด้วย Comma ใน [ ]

เมื่อข้อมูลของเราเป็นตาราง (`OCCURS`) แทนที่จะเป็น record เดี่ยว เราต้องวนลูปสร้าง object ย่อยแต่ละตัว
แล้วคั่นด้วย comma และครอบทั้งหมดด้วยวงเล็บเหลี่ยม `[ ]`

### ตัวอย่าง: แปลงตารางสินค้าเป็น JSON Array

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. JMAN3.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-ITEMS.
           05  WS-ITEM OCCURS 3 TIMES PIC X(8).
       01  WS-QTY.
           05  WS-QTY-ITEM OCCURS 3 TIMES PIC 9(3).
       01  WS-I            PIC 9(2).
       01  WS-QTY-EDIT     PIC ZZ9.
       01  WS-JSON         PIC X(150).
       01  WS-JSON-LEN     PIC 9(4) VALUE 1.
       PROCEDURE DIVISION.
           MOVE "APPLE   " TO WS-ITEM(1)
           MOVE "BANANA  " TO WS-ITEM(2)
           MOVE "CHERRY  " TO WS-ITEM(3)
           MOVE 10 TO WS-QTY-ITEM(1)
           MOVE 25 TO WS-QTY-ITEM(2)
           MOVE 7  TO WS-QTY-ITEM(3)

           MOVE SPACES TO WS-JSON
           STRING "[" DELIMITED BY SIZE
               INTO WS-JSON WITH POINTER WS-JSON-LEN

           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 3
               MOVE WS-QTY-ITEM(WS-I) TO WS-QTY-EDIT
               IF WS-I > 1
                   STRING "," DELIMITED BY SIZE
                       INTO WS-JSON WITH POINTER WS-JSON-LEN
               END-IF
               STRING '{"name":"' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-ITEM(WS-I)) DELIMITED BY SIZE
                   '","qty":' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-QTY-EDIT) DELIMITED BY SIZE
                   "}" DELIMITED BY SIZE
                   INTO WS-JSON WITH POINTER WS-JSON-LEN
           END-PERFORM
           STRING "]" DELIMITED BY SIZE
               INTO WS-JSON WITH POINTER WS-JSON-LEN

           DISPLAY FUNCTION TRIM(WS-JSON)
           STOP RUN.
```

**ผลลัพธ์:**

```
[{"name":"APPLE","qty":10},{"name":"BANANA","qty":25},{"name":"CHERRY","qty":7}]
```

### อธิบายจุดสำคัญ

- `STRING ... INTO WS-JSON WITH POINTER WS-JSON-LEN`: การใช้ `WITH POINTER` ทำให้เราสามารถเรียก
  `STRING` ได้หลายครั้งต่อเนื่องกัน โดยแต่ละครั้งจะต่อท้ายจากตำแหน่งที่ `WS-JSON-LEN` ชี้ไว้ล่าสุด
  แทนที่จะเขียนทับตั้งแต่ต้นทุกครั้ง (ทบทวนราบละเอียดของ `WITH POINTER` ได้จาก Part 019)
- `IF WS-I > 1` ก่อนใส่ comma: เทคนิคมาตรฐานสำหรับ "ใส่ comma คั่นระหว่างรายการ แต่ไม่ใส่ก่อนรายการ
  แรก" ที่ใช้ได้กับการสร้างรายการคั่นด้วยเครื่องหมายใด ๆ ก็ตาม
- โครงสร้างผลลัพธ์ `[{...},{...},{...}]` คือ JSON Array ของ 3 Object ตามที่ COBOL-2014 มาตรฐานจะสร้าง
  ให้เมื่อ `JSON GENERATE` ทำงานกับกลุ่มข้อมูลที่มี `OCCURS`

### ข้อควรระวัง

- ต้องเริ่มต้น `WS-JSON-LEN` เป็น 1 เสมอก่อนเริ่มสร้างข้อความใหม่ (ไม่ใช่ค่าที่หลงเหลือจากการใช้งาน
  ครั้งก่อน) มิฉะนั้นข้อความใหม่จะไปต่อท้ายข้อความเก่าที่ไม่ได้ล้างทิ้ง
- ตรวจสอบว่าความกว้างของ `WS-JSON` (`PIC X(150)` ในตัวอย่าง) เพียงพอกับจำนวนรายการสูงสุดที่อาจเกิด
  ขึ้นจริง หากตารางมีขนาดใหญ่ขึ้น ต้องคำนวณความกว้างที่เหมาะสมล่วงหน้า

### แบบฝึกหัดที่ 445.1

**โจทย์**: จงอธิบายว่าทำไมต้องตรวจสอบ `IF WS-I > 1` ก่อนใส่ comma แทนที่จะใส่ comma หลังทุก object
แล้วค่อยลบตัวสุดท้ายทิ้งภายหลัง

**เฉลย**: การตรวจสอบก่อนใส่ (`IF WS-I > 1`) เป็นวิธีที่ตรงไปตรงมาและมีประสิทธิภาพกว่า เพราะไม่ต้อง
เสียเวลาแก้ไขข้อความภายหลัง (ซึ่งต้องคำนวณตำแหน่ง comma ตัวสุดท้ายแล้วลบออกอีกที เพิ่มความซับซ้อนของ
โค้ดโดยไม่จำเป็น) การเช็คเงื่อนไขก่อนเขียนแต่ละครั้งทำให้โค้ดอ่านง่ายและมีจุดที่อาจเกิดบั๊กน้อยกว่า

---

## ขั้นตอนที่ 446: การ Parse JSON แบบง่ายด้วยมือผ่าน UNSTRING (Flat Object)

### แนวคิด: แยกข้อความ JSON เป็นชิ้นส่วนด้วย Delimiter ที่รู้จัก

การ parse JSON object แบบเรียบง่าย (ไม่มี nested object หรือ array ซ้อนอยู่ข้างใน) ทำได้โดยแยกทีละ
ขั้นตอน: (1) ตัดวงเล็บปีกกาครอบนอกออก (2) แยกแต่ละคู่ `"key":value` ด้วย comma (3) แยก key กับ value
แต่ละคู่ด้วย colon (4) ตัดเครื่องหมายคำพูดออกจาก key และ value ที่เป็นข้อความ

### ตัวอย่าง: Parser สำหรับ Flat JSON Object

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. JPARSE2.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-JSON-IN      PIC X(60)
           VALUE '{"name":"SOMCHAI","age":30,"city":"BANGKOK"}'.
       01  WS-BODY         PIC X(60).
       01  WS-LEN          PIC 9(4).
       01  WS-PARTS.
           05  WS-PART OCCURS 3 TIMES PIC X(30).
       01  WS-KEY          PIC X(15).
       01  WS-VALUE        PIC X(15).
       01  WS-I            PIC 9(2).
       PROCEDURE DIVISION.
      *> Step 1: strip the outer { and } via reference
      *> modification, sized from the trimmed length.
           COMPUTE WS-LEN = FUNCTION LENGTH(FUNCTION TRIM(WS-JSON-IN))
           MOVE SPACES TO WS-BODY
           MOVE WS-JSON-IN(2:WS-LEN - 2) TO WS-BODY

      *> Step 2: split the 3 "key":value pairs on comma.
           UNSTRING WS-BODY DELIMITED BY ","
               INTO WS-PART(1) WS-PART(2) WS-PART(3)

           DISPLAY "Parsed fields:"
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 3
      *> Step 3: split each "key":value on the first colon.
               UNSTRING WS-PART(WS-I) DELIMITED BY ":"
                   INTO WS-KEY WS-VALUE
      *> Step 4: strip surrounding quotes from key and value
      *> (numeric values like age have no quotes, so this is
      *> a no-op for them).
               INSPECT WS-KEY REPLACING ALL '"' BY SPACE
               INSPECT WS-VALUE REPLACING ALL '"' BY SPACE
               DISPLAY "  " FUNCTION TRIM(WS-KEY)
                   " = " FUNCTION TRIM(WS-VALUE)
           END-PERFORM
           STOP RUN.
```

**ผลลัพธ์:**

```
Parsed fields:
  name = SOMCHAI
  age = 30
  city = BANGKOK
```

### อธิบายจุดสำคัญ

- `WS-JSON-IN(2:WS-LEN - 2)`: reference modification ตัดตัวอักษรตัวแรก (`{`) และตัวสุดท้าย (`}`)
  ออก โดยเริ่มอ่านจากตำแหน่งที่ 2 เป็นความยาว `WS-LEN - 2` ตัวอักษร (ทบทวนเทคนิคนี้ได้จาก Part 019)
- `UNSTRING WS-BODY DELIMITED BY ","`: แยกข้อความออกเป็น 3 ส่วนตาม comma แต่ละส่วนยังอยู่ในรูปแบบ
  `"key":value` ที่ต้องแยกต่ออีกชั้น
- `UNSTRING WS-PART(WS-I) DELIMITED BY ":"`: แยก key กับ value ออกจากกันด้วย colon
- `INSPECT ... REPLACING ALL '"' BY SPACE`: ลบเครื่องหมายคำพูดออกจากทั้ง key และ value (ถ้าไม่มี
  เครื่องหมายคำพูดอยู่แล้ว เช่นค่าตัวเลข `30` คำสั่งนี้จะไม่มีผลใด ๆ — เป็น no-op ที่ปลอดภัย)

### ข้อควรระวัง

- **เทคนิคนี้ใช้ได้เฉพาะกับ flat object ที่ไม่มี nested object/array และไม่มี comma หรือ colon
  ปะปนอยู่ภายในค่าข้อความเอง** (เช่น ถ้า value เป็น `"Bangkok, Thailand"` ที่มี comma อยู่ในนั้น จะ
  ทำให้การแยกด้วย `UNSTRING DELIMITED BY ","` ผิดพลาดทันที) เป็นข้อจำกัดสำคัญที่ parser แบบมือเขียน
  เต็มรูปแบบ (state machine) เท่านั้นที่จะแก้ปัญหานี้ได้อย่างสมบูรณ์
- จำนวนฟิลด์ต้องรู้ล่วงหน้า (`OCCURS 3 TIMES` ในตัวอย่าง) — หากจำนวนฟิลด์ใน JSON เปลี่ยนแปลงไม่แน่นอน
  ต้องออกแบบให้ยืดหยุ่นกว่านี้ (ดูตัวอย่าง array ที่นับจำนวนจริงในขั้นตอนที่ 447)

### แบบฝึกหัดที่ 446.1

**โจทย์**: จงปรับ `JPARSE2` ให้รองรับ JSON ที่มีเพียง 2 field แทน 3 field (เช่น
`{"name":"MALEE","age":25}`)

**เฉลย**: เปลี่ยน `OCCURS 3 TIMES` เป็น `OCCURS 2 TIMES`, `UNSTRING WS-BODY DELIMITED BY ","` ให้
`INTO WS-PART(1) WS-PART(2)` เท่านั้น และปรับลูป `PERFORM VARYING ... UNTIL WS-I > 2` แทน

---

## ขั้นตอนที่ 447: การ Parse JSON Array กลับเข้าตาราง (OCCURS) ด้วยมือ

### แนวคิด: แยก Object แต่ละตัวในวงเล็บเหลี่ยมออกจากกันก่อน

การ parse JSON array ต้องแยกที่ระดับ object ก่อน (คั่นด้วย `},{`) แล้วจึงค่อย parse แต่ละ object
ต่อด้วยเทคนิคเดียวกับขั้นตอนที่ 446

### ตัวอย่าง: Parser สำหรับ JSON Array กลับเข้าตาราง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. JPARSE4.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-JSON-IN      PIC X(100) VALUE
           '[{"name":"APPLE","qty":10},{"name":"BANANA","qty":25}]'.
       01  WS-BODY         PIC X(100).
       01  WS-LEN          PIC 9(4).
       01  WS-OBJECTS.
           05  WS-OBJECT OCCURS 5 TIMES PIC X(40).
       01  WS-COUNT        PIC 9(2) VALUE 0.
       01  WS-I            PIC 9(2).
       PROCEDURE DIVISION.
           COMPUTE WS-LEN = FUNCTION LENGTH(FUNCTION TRIM(WS-JSON-IN))
           MOVE SPACES TO WS-BODY
           MOVE WS-JSON-IN(2:WS-LEN - 2) TO WS-BODY

      *> TALLYING IN counts how many pieces UNSTRING actually
      *> produced, so we know how many table entries are filled.
           UNSTRING WS-BODY DELIMITED BY "},{"
               INTO WS-OBJECT(1) WS-OBJECT(2) WS-OBJECT(3)
                    WS-OBJECT(4) WS-OBJECT(5)
               TALLYING IN WS-COUNT

           DISPLAY "Number of objects found: " WS-COUNT
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > WS-COUNT
               INSPECT WS-OBJECT(WS-I) REPLACING ALL '{' BY SPACE
               INSPECT WS-OBJECT(WS-I) REPLACING ALL '}' BY SPACE
               INSPECT WS-OBJECT(WS-I) REPLACING ALL '"' BY SPACE
               DISPLAY "Object " WS-I ": "
                   FUNCTION TRIM(WS-OBJECT(WS-I))
           END-PERFORM
           STOP RUN.
```

**ผลลัพธ์:**

```
Number of objects found: 02
Object 01: name : APPLE , qty :10
Object 02: name : BANANA , qty :25
```

### อธิบายจุดสำคัญ

- `UNSTRING ... DELIMITED BY "},{"`: ใช้ลำดับอักขระ `},{ ` (ที่คั่นระหว่าง object แต่ละตัวใน array)
  เป็น delimiter — เทคนิคนี้ใช้ได้เพราะเราตัดวงเล็บเหลี่ยมครอบนอกออกไปแล้วตั้งแต่ต้น (ด้วย reference
  modification) เหลือแต่เนื้อหาภายใน `[ ]`
- `TALLYING IN WS-COUNT`: นับจำนวนชิ้นส่วนที่ `UNSTRING` แยกออกมาได้จริง ทำให้เรารู้ว่าตารางถูกเติม
  ข้อมูลไปกี่แถว โดยไม่ต้องรู้จำนวน object ล่วงหน้าตายตัว (ทบทวน `TALLYING` เพิ่มเติมได้จาก Part 020)
- ในตัวอย่างนี้ผลลัพธ์การแสดงผลของแต่ละ object ยังไม่ได้แยก key/value ออกจากกันอย่างสมบูรณ์ (เพียง
  ลบวงเล็บและเครื่องหมายคำพูดออก) หากต้องการแยกละเอียดถึงระดับ field สามารถนำแต่ละ `WS-OBJECT(WS-I)`
  ไปผ่านเทคนิค parse แบบ flat object จากขั้นตอนที่ 446 ต่อได้ทันที

### ข้อควรระวัง

- ต้องกำหนดขนาด `OCCURS` ของตารางปลายทาง (`WS-OBJECT OCCURS 5 TIMES` ในตัวอย่าง) ให้มากพอรองรับ
  จำนวน object สูงสุดที่อาจเกิดขึ้นจริงเสมอ มิฉะนั้น `UNSTRING` จะละทิ้งข้อมูลส่วนเกินอย่างเงียบ ๆ
- เทคนิคการแยกด้วย `},{ ` มีข้อจำกัดเดียวกับขั้นตอนที่ 446: ใช้ไม่ได้หากมี nested object/array ซ้อน
  อยู่ภายใน object แต่ละตัว

### แบบฝึกหัดที่ 447.1

**โจทย์**: จงต่อยอดโปรแกรม `JPARSE4` ให้แยก `name` และ `qty` ของแต่ละ object ออกมาแสดงเป็นสอง
คอลัมน์แยกกันชัดเจน โดยใช้เทคนิคจากขั้นตอนที่ 446 ร่วมด้วย

**เฉลย** (แนวทาง): หลังจากได้ `WS-OBJECT(WS-I)` ที่ลบวงเล็บและเครื่องหมายคำพูดออกแล้ว (เช่น
`name : APPLE , qty :10`) ให้ใช้ `UNSTRING WS-OBJECT(WS-I) DELIMITED BY ","` แยกเป็นสองส่วนก่อน
แล้วใช้ `UNSTRING ... DELIMITED BY ":"` แยก key/value ของแต่ละส่วนต่อ ตามรูปแบบเดียวกับ `JPARSE2`
ในขั้นตอนที่ 446

---

## ขั้นตอนที่ 448: ตัวเลข, ค่าว่าง (null) และการจัดรูปแบบข้อมูลตัวเลขใน JSON

### ความแตกต่างระหว่างค่าตัวเลขและค่าข้อความใน JSON

ใน JSON ค่าตัวเลข**ไม่มีเครื่องหมายคำพูดล้อมรอบ** (เช่น `"age":30`) ในขณะที่ค่าข้อความ**ต้องมี
เครื่องหมายคำพูดล้อมรอบเสมอ** (เช่น `"name":"SOMCHAI"`) การสร้าง JSON ด้วยมือจึงต้องระวังเรื่องนี้
ให้ถูกต้อง — สิ่งที่ตรงกันข้ามกับความคุ้นเคยจาก COBOL ที่ field ตัวเลขมักถูกจัดรูปแบบด้วยเครื่องหมาย
คำพูดเมื่อพิมพ์แสดงผล

### ตัวอย่าง: จัดการตัวเลขทศนิยมและค่า null ให้ถูกต้องตามกฎ JSON

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. JMAN4.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRICE        PIC 9(5)V99 VALUE 149.50.
       01  WS-PRICE-EDIT   PIC Z(4)9.99.
       01  WS-HAS-DISCOUNT PIC X VALUE "N".
       01  WS-DISCOUNT-PART PIC X(30).
       01  WS-JSON         PIC X(100).
       PROCEDURE DIVISION.
           MOVE WS-PRICE TO WS-PRICE-EDIT
           IF WS-HAS-DISCOUNT = "Y"
               MOVE '"discount":10' TO WS-DISCOUNT-PART
           ELSE
      *> JSON's null (no quotes) represents "no value", distinct
      *> from an empty string "" or the number zero.
               MOVE '"discount":null' TO WS-DISCOUNT-PART
           END-IF
           STRING '{"price":' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-PRICE-EDIT) DELIMITED BY SIZE
                   ',' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-DISCOUNT-PART) DELIMITED BY SIZE
                   '}' DELIMITED BY SIZE
               INTO WS-JSON
           DISPLAY FUNCTION TRIM(WS-JSON)
           STOP RUN.
```

**ผลลัพธ์:**

```
{"price":149.50,"discount":null}
```

### อธิบายจุดสำคัญ

- `PIC Z(4)9.99` ใช้แสดงตัวเลขทศนิยมโดยมีจุดทศนิยมจริง (`.`) ปรากฏในผลลัพธ์ (ต่างจาก `PIC 9(5)V99`
  ที่ `V` เป็นจุดทศนิยม**สมมติ** ไม่ปรากฏจริงในหน่วยความจำ) — จำเป็นสำหรับ JSON ที่ต้องการจุดทศนิยม
  จริงในข้อความ (ทบทวน editing picture ได้จาก Part 006)
- ค่า `null` ใน JSON เขียนโดย**ไม่มีเครื่องหมายคำพูด** และไม่ใช่คำว่า `"null"` (ที่มีเครื่องหมายคำพูด
  จะกลายเป็นข้อความ "null" ธรรมดา ไม่ใช่ค่าว่างพิเศษของ JSON) ต้องเขียนตัวอักษร `null` ตรง ๆ ในเนื้อ
  ข้อความ JSON

### ข้อควรระวัง

- ระวังอย่าใส่เครื่องหมายคำพูดรอบตัวเลขหรือ `null`/`true`/`false` โดยไม่ตั้งใจ เพราะจะเปลี่ยนความหมาย
  ของค่าจาก "ตัวเลข/ค่าพิเศษ" กลายเป็น "ข้อความธรรมดา" ทันที ซึ่งฝั่งรับข้อมูล (เช่น เว็บแอปพลิเคชัน
  ที่เขียนด้วย JavaScript) อาจตีความข้อมูลผิดพลาดได้
- `PIC Z(4)9.99` เมื่อค่าน้อยกว่า 100000.00 จะมีช่องว่างนำหน้า ต้อง `FUNCTION TRIM` ก่อนนำไปต่อ JSON
  เสมอ เหมือนกับตัวอย่างในขั้นตอนก่อนหน้า

### แบบฝึกหัดที่ 448.1

**โจทย์**: จงอธิบายว่าทำไม `"discount":"null"` (มีเครื่องหมายคำพูดล้อม null) กับ `"discount":null`
(ไม่มีเครื่องหมายคำพูด) จึงมีความหมายต่างกันโดยสิ้นเชิงสำหรับโปรแกรมที่อ่านค่า JSON นี้

**เฉลย**: `"discount":"null"` หมายความว่า field `discount` มีค่าเป็น**ข้อความ** "null" (สตริง 4
ตัวอักษร) ซึ่งโปรแกรมที่อ่านค่าจะเห็นเป็นข้อความจริง ๆ ไม่ใช่ค่าว่าง ในขณะที่ `"discount":null`
หมายความว่า field `discount` **ไม่มีค่า** เลย (JSON's null คือค่าพิเศษที่แทน "ไม่มีข้อมูล") โปรแกรม
ส่วนใหญ่ (เช่น JavaScript) จะตรวจสอบสองกรณีนี้ด้วยวิธีต่างกันโดยสิ้นเชิง หากสร้าง JSON ผิดรูปแบบจะทำ
ให้ระบบปลายทางตีความข้อมูลผิดพลาดได้

---

## ขั้นตอนที่ 449: ข้อจำกัดของเทคนิคสร้างเอง เทียบกับ JSON GENERATE/PARSE จริง

### สรุปข้อจำกัดที่ต้องตระหนัก

เทคนิคที่สอนมาตลอด Part นี้ (`STRING`/`UNSTRING`/`INSPECT`) ใช้งานได้จริงและเพียงพอสำหรับหลาย
สถานการณ์ แต่มีข้อจำกัดสำคัญเมื่อเทียบกับ `JSON GENERATE`/`JSON PARSE` ที่มีในคอมไพเลอร์ที่รองรับเต็ม
รูปแบบ:

| ประเด็น | เทคนิคสร้างเอง (STRING/UNSTRING) | JSON GENERATE/PARSE (มาตรฐาน) |
|---|---|---|
| Object/Array ซ้อนกันหลายชั้น (nested) | ทำได้ยาก ต้องเขียนโค้ดแยกแต่ละชั้นเอง | รองรับอัตโนมัติตามโครงสร้างกลุ่มข้อมูล COBOL |
| ค่าข้อความที่มี comma/colon อยู่ภายใน | เสี่ยง parse ผิดพลาดสูง (ตามที่เห็นในขั้นตอนที่ 446) | รองรับถูกต้องเสมอ (ใช้ state machine ประมวลผล JSON จริง) |
| Escape อักขระพิเศษครบถ้วน (Unicode, control chars) | ต้องเขียนโค้ดจัดการเองทั้งหมด | จัดการให้อัตโนมัติตามมาตรฐาน JSON |
| ความเร็วในการพัฒนา | ช้ากว่า ต้องเขียนและทดสอบโค้ดเอง | เร็วกว่ามาก เพียงกำหนดโครงสร้างข้อมูล COBOL |
| ประสิทธิภาพการทำงาน | พอใช้สำหรับข้อมูลขนาดเล็ก-กลาง | มักผ่านการปรับแต่งประสิทธิภาพมาอย่างดีจากผู้พัฒนาคอมไพเลอร์ |

### ไวยากรณ์อ้างอิงแบบเต็มของ JSON GENERATE (สำหรับใช้เมื่อคอมไพเลอร์รองรับ)

**หมายเหตุ**: โค้ดต่อไปนี้เป็นไวยากรณ์อ้างอิงตามมาตรฐาน COBOL-2014 **ยังไม่ได้ทดสอบว่าคอมไพล์ผ่านใน
บิลด์ปัจจุบัน** (ยืนยันแล้วว่า compile ไม่ผ่านตามขั้นตอนที่ 442) แสดงไว้เพื่อให้คุณจดจำรูปแบบไปใช้เมื่อ
ทำงานกับคอมไพเลอร์ที่รองรับ (เช่น IBM Enterprise COBOL, Micro Focus, หรือ GnuCOBOL ที่ build พร้อม
เปิดใช้งาน JSON library):

```
      *> Reference syntax only - NOT runnable on this build.
      *> Confirmed to fail with: "compiler is not configured
      *> to support JSON" (see step 442).
       01  WS-CUSTOMER.
           05  CUST-ID     PIC 9(5).
           05  CUST-NAME   PIC X(20).
       01  WS-JSON-OUT     PIC X(200).

       JSON GENERATE WS-JSON-OUT FROM WS-CUSTOMER
           NAME OF CUST-ID   IS "id"
           NAME OF CUST-NAME IS "name"
           ON EXCEPTION
               DISPLAY "JSON GENERATE failed"
       END-JSON

       JSON PARSE WS-JSON-OUT INTO WS-CUSTOMER
           NAME OF CUST-ID   IS "id"
           NAME OF CUST-NAME IS "name"
           ON EXCEPTION
               DISPLAY "JSON PARSE failed"
       END-JSON
```

### อธิบายจุดสำคัญ

- `NAME OF field IS "json-key"` คือ clause สำคัญที่ทำให้เราแมปชื่อ field COBOL (ที่มักมีขีดกลางหรือ
  ตัวพิมพ์ใหญ่ตามธรรมเนียม COBOL) เข้ากับชื่อ key ใน JSON ตามธรรมเนียม camelCase หรือ snake_case
  ที่นิยมในโลก JSON/JavaScript
- `ON EXCEPTION` ใน `JSON GENERATE`/`JSON PARSE` ทำหน้าที่คล้ายกับที่เราเห็นใน `CALL` (Part 043) —
  ดักจับข้อผิดพลาดโดยไม่ทำให้โปรแกรม crash

### ข้อควรระวัง

- **อย่านำโค้ดในกล่องอ้างอิงข้างต้นไป compile ในสภาพแวดล้อมนี้โดยตรง** เพราะจะได้ error เหมือนที่แสดง
  ในขั้นตอนที่ 442 ทุกประการ — ก่อนใช้งานในโปรเจกต์จริงเสมอต้องตรวจสอบด้วย `cobc --info | grep -i
  json` ก่อน
- หากทำงานกับระบบที่คอมไพเลอร์รองรับ JSON จริง แนะนำให้ใช้ `JSON GENERATE`/`JSON PARSE` แทนเทคนิคสร้าง
  เองเสมอ เพราะปลอดภัยและดูแลรักษาง่ายกว่ามาก เทคนิคสร้างเองควรสงวนไว้สำหรับกรณีที่ยืนยันแล้วว่าไม่มี
  ทางเลือกอื่นเท่านั้น

### แบบฝึกหัดที่ 449.1

**โจทย์**: จงอธิบายว่าเหตุใดหลักสูตรนี้จึงยังคงสอนไวยากรณ์ของ `JSON GENERATE`/`JSON PARSE` ทั้งที่
compile ไม่ผ่านในสภาพแวดล้อมที่ใช้ฝึกจริง

**เฉลย**: เพราะไวยากรณ์นี้เป็นส่วนหนึ่งของมาตรฐาน COBOL-2014 ที่คอมไพเลอร์เชิงพาณิชย์หลายตัว (IBM
Enterprise COBOL, Micro Focus) รองรับเต็มรูปแบบ ผู้เรียนที่จะทำงานกับระบบองค์กรจริงในอนาคตมีโอกาสสูง
ที่จะพบโค้ดลักษณะนี้ในระบบที่ใช้คอมไพเลอร์เหล่านั้น การเข้าใจไวยากรณ์จึงมีค่าแม้จะไม่สามารถทดสอบรันจริง
ในสภาพแวดล้อมของหลักสูตรได้ ในขณะเดียวกันหลักสูตรก็ยึดหลักความซื่อสัตย์ทางเทคนิค: จะไม่บอกว่าโค้ดนี้
"รันได้" ในบิลด์ที่ใช้อยู่หากไม่เป็นความจริง จึงระบุไว้อย่างชัดเจนว่าเป็นไวยากรณ์อ้างอิงเท่านั้น

---

## ขั้นตอนที่ 450: ตัวอย่างรวม — โปรแกรมส่งออก Record ลูกค้าเป็นไฟล์ JSON

### ภาพรวมโปรแกรมสุดท้ายของ Part นี้

เราจะรวมเทคนิคทั้งหมดที่เรียนมา: การสร้าง JSON ด้วย `STRING`, การจัดรูปแบบตัวเลข, และการเขียนผลลัพธ์
ลงไฟล์จริงด้วย `WRITE` (ทบทวนจาก Part 023-025) เพื่อสร้างโปรแกรมส่งออกข้อมูลลูกค้าเป็นไฟล์ `.json`
ที่ใช้งานได้จริง

### ⚠️ ข้อควรระวังสำคัญที่ตรวจพบระหว่างการทดสอบ: ต้องล้าง Record ด้วย SPACES ก่อนเสมอ

ระหว่างทดสอบโปรแกรมสำหรับขั้นตอนนี้ พบปัญหาจริงที่สำคัญมาก: หากเขียนค่าเข้า record ของไฟล์
(`FD` record) ด้วย `STRING` โดย**ไม่ล้างค่าตัวแปรด้วย `SPACES` ก่อน** ตัวอักษรขยะ (ค่าที่ยังไม่ถูก
กำหนดค่าเริ่มต้น) ที่หลงเหลืออยู่ในส่วนที่ `STRING` ไม่ได้เขียนทับ จะทำให้ `WRITE` ล้มเหลวด้วย
**File Status 71 (boundary violation)** ทันที:

```
libcob: error: unknown file error (status = 71) for file OUT-FILE ('TEST5.DAT')
```

หลังจากเพิ่ม `MOVE SPACES TO OUT-LINE` ก่อน `STRING` ปัญหาหายไปทันทีและไฟล์ถูกเขียนสำเร็จ — นี่คือ
กับดักที่พบได้จริงจากการทดสอบ ไม่ใช่แค่ทฤษฎี จึงนำมาสอนไว้ในตัวอย่างสมบูรณ์ด้านล่าง

### โปรแกรมสมบูรณ์: ส่งออก Record ลูกค้าเป็นไฟล์ JSON

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. JFILE7.
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT OUT-FILE ASSIGN TO "CUSTOMER450.JSON"
               ORGANIZATION IS LINE SEQUENTIAL.
       DATA DIVISION.
       FILE SECTION.
       FD  OUT-FILE.
       01  OUT-LINE            PIC X(120).
       WORKING-STORAGE SECTION.
       01  WS-NAME             PIC X(15) VALUE "SOMCHAI JAIDEE".
       01  WS-AGE              PIC 9(3)  VALUE 30.
       01  WS-AGE-EDIT         PIC ZZ9.
       PROCEDURE DIVISION.
           MOVE WS-AGE TO WS-AGE-EDIT
           OPEN OUTPUT OUT-FILE
      *> CRITICAL: clear the FD record before STRING fills only
      *> part of it. Without this, leftover uninitialized bytes
      *> caused a real "status = 71" boundary violation on WRITE
      *> when tested - confirmed during development of this part.
           MOVE SPACES TO OUT-LINE
           STRING '{"name":"' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-NAME) DELIMITED BY SIZE
                   '","age":' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-AGE-EDIT) DELIMITED BY SIZE
                   "}" DELIMITED BY SIZE
               INTO OUT-LINE
           WRITE OUT-LINE
           CLOSE OUT-FILE
           DISPLAY "Wrote CUSTOMER450.JSON"
           STOP RUN.
```

**ผลลัพธ์:**

```
Wrote CUSTOMER450.JSON
```

เนื้อหาไฟล์ `CUSTOMER450.JSON` ที่ได้:

```
{"name":"SOMCHAI JAIDEE","age":30}
```

### อธิบายจุดสำคัญ

- `MOVE SPACES TO OUT-LINE` ก่อน `STRING`: บรรทัดนี้**จำเป็นเสมอ**สำหรับ FD record ที่ `STRING`
  จะเขียนทับเพียงบางส่วน (ความยาว JSON จริงมักสั้นกว่าความกว้าง record ที่กำหนดไว้เผื่อ) มิฉะนั้นจะ
  เกิด error ตามที่อธิบายไว้ข้างต้น
- `ORGANIZATION IS LINE SEQUENTIAL`: เลือกใช้ไฟล์ text ธรรมดา (ทบทวนจาก Part 023) เพื่อให้ผลลัพธ์
  เปิดอ่านด้วยโปรแกรม text editor ทั่วไปได้โดยตรง
- โครงสร้างโปรแกรมนี้เป็นแม่แบบที่นำไปต่อยอดได้จริง: เปลี่ยนข้อมูลต้นทางเป็นการอ่านจากไฟล์หรือ
  ฐานข้อมูลแทนค่าคงที่ แล้ววนลูปเขียนหลาย record ลงไฟล์ JSON เดียวกัน (เช่น สร้างเป็น JSON Array
  ตามเทคนิคขั้นตอนที่ 445 ผสมกับการเขียนไฟล์ในขั้นตอนนี้) ก็จะได้ระบบส่งออกข้อมูลเป็น JSON ไฟล์ที่
  ใช้งานได้จริงในระดับเบื้องต้น

### ข้อควรระวัง

- ตรวจสอบให้แน่ใจเสมอว่า `MOVE SPACES TO record-name` มาก่อน `STRING ... INTO record-name` ทุกครั้ง
  ที่ record ปลายทางเป็นส่วนหนึ่งของ `FD` (ไฟล์) — นี่ไม่ใช่แค่นิสัยที่ดี แต่เป็น**ข้อกำหนดที่จำเป็น**
  เพื่อป้องกัน File Status 71 ตามที่พิสูจน์จากการทดสอบจริง
- สำหรับระบบจริงที่ต้องส่งออกข้อมูลจำนวนมากเป็น JSON Array ควรพิจารณาว่าคอมไพเลอร์ปลายทางรองรับ
  `JSON GENERATE` หรือไม่ก่อนเสมอ (ขั้นตอนที่ 442) เพราะจะประหยัดเวลาพัฒนาได้มากกว่าการเขียนเทคนิค
  ด้วยมือทั้งหมด หากรองรับได้จริง

### แบบฝึกหัดที่ 450.1

**โจทย์**: จงปรับโปรแกรม `JFILE7` ให้เขียน record ลูกค้า 2 คนลงไฟล์เดียวกันในรูปแบบ JSON Array
(`[{...},{...}]`)

**เฉลย** (แนวทาง): รวมเทคนิคจากขั้นตอนที่ 445 (สร้าง JSON Array ด้วยลูป) เข้ากับการเขียนไฟล์:

```cobol
       WORKING-STORAGE SECTION.
       01  WS-NAMES.
           05  FILLER PIC X(15) VALUE "SOMCHAI JAIDEE".
           05  FILLER PIC X(15) VALUE "MALEE SRISUK".
       01  WS-NAME-TABLE REDEFINES WS-NAMES.
           05  WS-CUST-NAME OCCURS 2 TIMES PIC X(15).
       01  WS-JSON             PIC X(150).
       01  WS-PTR              PIC 9(4) VALUE 1.
       01  WS-I                PIC 9(2).
       PROCEDURE DIVISION.
           MOVE SPACES TO WS-JSON
           STRING "[" DELIMITED BY SIZE
               INTO WS-JSON WITH POINTER WS-PTR
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 2
               IF WS-I > 1
                   STRING "," DELIMITED BY SIZE
                       INTO WS-JSON WITH POINTER WS-PTR
               END-IF
               STRING '{"name":"' DELIMITED BY SIZE
                   FUNCTION TRIM(WS-CUST-NAME(WS-I))
                       DELIMITED BY SIZE
                   '"}' DELIMITED BY SIZE
                   INTO WS-JSON WITH POINTER WS-PTR
           END-PERFORM
           STRING "]" DELIMITED BY SIZE
               INTO WS-JSON WITH POINTER WS-PTR
           MOVE SPACES TO OUT-LINE
           MOVE WS-JSON TO OUT-LINE
           WRITE OUT-LINE.
```

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้เรื่อง JSON ใน COBOL อย่างครบถ้วนและซื่อตรงต่อข้อเท็จจริง:

- ภาพรวมไวยากรณ์มาตรฐานของ `JSON GENERATE`/`JSON PARSE` ตาม COBOL-2014
- **การทดสอบจริงและข้อค้นพบสำคัญ**: บิลด์ GnuCOBOL ที่ใช้ในหลักสูตรนี้ปิดใช้งาน JSON library ทำให้
  `JSON GENERATE`/`JSON PARSE` compile ไม่ผ่านเลย (ยืนยันด้วย `cobc --info` และข้อความ error จริง)
- วิธีตรวจสอบว่าคอมไพเลอร์ของคุณรองรับ feature ใด ๆ ก่อนนำไปใช้งานจริง — วินัยสำคัญที่ควรติดตัวไปใช้
  ตลอดสายอาชีพ
- เทคนิคสร้าง JSON ด้วยมือผ่าน `STRING`: object เดี่ยว, escape เครื่องหมายคำพูด, array จากตาราง
  `OCCURS`, การจัดรูปแบบตัวเลขและ `null` ให้ถูกต้อง
- เทคนิค parse JSON ด้วยมือผ่าน `UNSTRING`/`INSPECT`: flat object และ array กลับเข้าตาราง พร้อม
  ข้อจำกัดที่ต้องตระหนัก (ใช้ไม่ได้กับ nested structure หรือค่าที่มี comma/colon ปะปนอยู่ภายใน)
- ตารางเปรียบเทียบข้อจำกัดของเทคนิคสร้างเองกับ `JSON GENERATE`/`JSON PARSE` มาตรฐาน
- ตัวอย่างรวมส่งออก record ลูกค้าเป็นไฟล์ JSON พร้อม**ข้อควรระวังที่ตรวจพบจริงระหว่างการพัฒนา**:
  ต้อง `MOVE SPACES` ล้าง record ก่อน `STRING` เสมอ มิฉะนั้นจะเกิด File Status 71

Part นี้แสดงให้เห็นว่าแม้ feature ระดับมาตรฐานอย่าง JSON อาจไม่พร้อมใช้งานในทุกสภาพแวดล้อม แต่ COBOL
ที่มีเครื่องมือพื้นฐานอย่าง `STRING`/`UNSTRING` ก็ยังสามารถทำงานร่วมกับระบบสมัยใหม่ได้จริงเสมอ ใน Part
ถัดไปเราจะพบสถานการณ์คล้ายกันกับ **XML** ผ่าน `XML GENERATE`/`XML PARSE` ซึ่งเป็นอีกหนึ่งฟีเจอร์สำคัญ
ของ COBOL สมัยใหม่สำหรับการแลกเปลี่ยนข้อมูลกับระบบ Enterprise แบบดั้งเดิม

**[← กลับไป Part 044: Based Storage และ POINTER Data Item](part-044-based-storage-pointer.md)** | **[ไปยัง Part 046: XML GENERATE และ XML PARSE →](part-046-xml-generate-parse.md)**
