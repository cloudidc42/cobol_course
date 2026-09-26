# Part 037: Intrinsic Functions: ข้อความและสถิติ (ขั้นตอนที่ 361–370)

## คำนำของ Part นี้

Part 036 พาเราทำความรู้จักกับ Intrinsic Functions กลุ่มตัวเลขและวันที่ไปแล้ว Part นี้จะสอน
Intrinsic Functions อีกสองกลุ่มที่สำคัญไม่แพ้กัน: **ฟังก์ชันจัดการข้อความ** (String Functions) ที่
ช่วยแปลงตัวพิมพ์ใหญ่-เล็ก, ตัดช่องว่าง, หาความยาว, และแปลงข้อความเป็นตัวเลข — งานที่เราเคยต้อง
เขียนด้วย `STRING`/`UNSTRING`/`INSPECT` (Part 019-021) แบบยาว ๆ ตอนนี้บางกรณีทำได้ในบรรทัด
เดียว และ **ฟังก์ชันสถิติ** (Statistical Functions) เช่น ผลรวม ค่าเฉลี่ย มัธยฐาน และส่วนเบี่ยงเบน
มาตรฐาน ที่มีประโยชน์มากสำหรับงานวิเคราะห์ข้อมูลทางธุรกิจ

สิ่งสำคัญที่ Part นี้เน้นย้ำเป็นพิเศษคือ **ไม่ใช่ทุกฟังก์ชันที่มีชื่ออยู่ในมาตรฐาน COBOL จะถูกรองรับ
เหมือนกันในทุกคอมไพเลอร์** เราได้ทดสอบฟังก์ชันทุกตัวในเอกสารนี้จริงกับ GnuCOBOL 4.0-early-dev
แล้ว และจะระบุให้ชัดเจนว่าฟังก์ชันใดใช้งานได้จริง ฟังก์ชันใดที่ดูเหมือนจะมีปัญหาแต่แท้จริงแล้วเป็นเพราะ
กฎคอลัมน์ 72 (ไม่ใช่ข้อจำกัดของฟังก์ชันเอง) — เป็นบทเรียนสำคัญที่ยืนยันกฎเหล็กของหลักสูตรนี้อีกครั้ง
หนึ่งจากมุมที่ไม่คาดคิด

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน และทุกตัวอย่าง
> ผ่านการคอมไพล์และรันจริงด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง

---

## ขั้นตอนที่ 361: FUNCTION UPPER-CASE และ FUNCTION LOWER-CASE

### แปลงตัวพิมพ์โดยไม่ต้องใช้ INSPECT

ใน Part 021 เราเรียนวิธีแปลงตัวพิมพ์ใหญ่-เล็กด้วย `INSPECT ... CONVERTING` ซึ่งต้องเขียนรายการ
ตัวอักษรทั้งหมดที่ต้องการแปลง Intrinsic Function `UPPER-CASE` และ `LOWER-CASE` ทำสิ่งเดียวกันนี้
ในคำสั่งเดียว โดยไม่ต้องระบุตัวอักษรเอง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP361-UPPER-LOWER.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NAME             PIC X(20) VALUE "Somchai Jaidee".
       01  WS-UPPER            PIC X(20).
       01  WS-LOWER            PIC X(20).

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE FUNCTION UPPER-CASE(WS-NAME) TO WS-UPPER.
           DISPLAY "UPPER-CASE: [" WS-UPPER "]".

           MOVE FUNCTION LOWER-CASE(WS-NAME) TO WS-LOWER.
           DISPLAY "LOWER-CASE: [" WS-LOWER "]".

      *> Common use: case-insensitive comparison.
           IF FUNCTION UPPER-CASE(WS-NAME) = "SOMCHAI JAIDEE      "
               DISPLAY "MATCH FOUND (CASE-INSENSITIVE)"
           END-IF.
           STOP RUN.
```

**ผลลัพธ์:**

```
UPPER-CASE: [SOMCHAI JAIDEE      ]
LOWER-CASE: [somchai jaidee      ]
MATCH FOUND (CASE-INSENSITIVE)
```

### อธิบายจุดสำคัญ

- `FUNCTION UPPER-CASE(WS-NAME)` คืนค่าข้อความเดียวกับ `WS-NAME` ทุกตัวอักษร แต่แปลงเป็นตัวพิมพ์
  ใหญ่ทั้งหมด ความยาวของผลลัพธ์เท่ากับความยาวของฟิลด์ต้นทางเสมอ (รวมช่องว่างท้าย)
- การใช้งานที่พบบ่อยที่สุดคือการเปรียบเทียบข้อความแบบ **ไม่สนตัวพิมพ์ใหญ่-เล็ก** (Case-Insensitive
  Comparison) ดังตัวอย่าง `IF FUNCTION UPPER-CASE(WS-NAME) = "SOMCHAI JAIDEE      "` — ไม่ว่า
  ผู้ใช้จะพิมพ์ชื่อมาด้วยตัวพิมพ์แบบใด การเปรียบเทียบจะสำเร็จเสมอตราบใดที่ตัวอักษรตรงกัน
- สังเกตว่า Literal ที่ใช้เปรียบเทียบ (`"SOMCHAI JAIDEE      "`) ต้องมีช่องว่างท้ายให้ครบ 20 ตัวอักษร
  พอดี (เท่ากับ `PIC X(20)`) เพราะ COBOL เปรียบเทียบข้อความแบบเติมช่องว่างให้เท่ากันก่อนเปรียบเทียบ
  แต่การนับจำนวนช่องว่างเองมักผิดพลาดได้ง่าย ในทางปฏิบัติจะปลอดภัยกว่าถ้าใช้ตัวแปรอีกตัวหนึ่งเป็น
  ฝั่งขวาแทนการพิมพ์ Literal ยาว ๆ เอง

### ข้อควรระวัง

- `UPPER-CASE`/`LOWER-CASE` แปลงเฉพาะตัวอักษร A-Z/a-z ตามมาตรฐาน ASCII เท่านั้น ไม่รองรับ
  ตัวอักษรที่มีเครื่องหมายกำกับเสียง (Accented Characters) ในภาษาอื่นที่ไม่ใช่ภาษาอังกฤษ
- ผลลัพธ์ของฟังก์ชันนี้มีความยาวเท่ากับฟิลด์ต้นทางเสมอ หากนำไปเก็บในฟิลด์ที่มีขนาดเล็กกว่า ข้อความ
  จะถูกตัดท้าย (Truncate) เหมือนการ `MOVE` ปกติทั่วไป

### แบบฝึกหัดที่ 361.1

**โจทย์**: จงเขียนเงื่อนไขตรวจสอบว่าค่าที่ผู้ใช้กรอกมาใน `WS-ANSWER` (สมมติว่าเก็บคำตอบ "yes"
หรือ "YES" หรือ "Yes" ก็ได้) ตรงกับ "YES" หรือไม่ โดยไม่สนตัวพิมพ์ใหญ่-เล็ก

**เฉลย**:

```cobol
       01  WS-ANSWER           PIC X(3) VALUE "Yes".
       ...
           IF FUNCTION UPPER-CASE(WS-ANSWER) = "YES"
               DISPLAY "USER CONFIRMED"
           ELSE
               DISPLAY "USER DID NOT CONFIRM"
           END-IF.
```

---

## ขั้นตอนที่ 362: FUNCTION TRIM — ตัดช่องว่างหัวท้าย

### ตัดช่องว่างโดยไม่ต้องคำนวณความยาวเอง

ก่อนหน้านี้การตัดช่องว่างท้ายข้อความมักต้องใช้ `INSPECT ... TALLYING` หาความยาวจริงก่อน แล้วใช้
Reference Modification ตัด `FUNCTION TRIM` ทำสิ่งนี้ในคำสั่งเดียว โดยรองรับ 3 รูปแบบ:
`FUNCTION TRIM(x)` (ตัดทั้งหัวและท้าย), `FUNCTION TRIM(x LEADING)` (ตัดเฉพาะหัว), และ
`FUNCTION TRIM(x TRAILING)` (ตัดเฉพาะท้าย)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP362-TRIM.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PADDED           PIC X(12) VALUE "  hi there  ".
       01  WS-MARKED           PIC X(20).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Brackets make leading/trailing spaces visible in the output.
           STRING "[" FUNCTION TRIM(WS-PADDED) "]"
               DELIMITED BY SIZE INTO WS-MARKED.
           DISPLAY "TRIM (BOTH SIDES): " WS-MARKED.

           MOVE SPACES TO WS-MARKED.
           STRING "[" FUNCTION TRIM(WS-PADDED LEADING) "]"
               DELIMITED BY SIZE INTO WS-MARKED.
           DISPLAY "TRIM LEADING:      " WS-MARKED.

           MOVE SPACES TO WS-MARKED.
           STRING "[" FUNCTION TRIM(WS-PADDED TRAILING) "]"
               DELIMITED BY SIZE INTO WS-MARKED.
           DISPLAY "TRIM TRAILING:     " WS-MARKED.
           STOP RUN.
```

**ผลลัพธ์:**

```
TRIM (BOTH SIDES): [hi there]          
TRIM LEADING:      [hi there  ]        
TRIM TRAILING:     [  hi there]
```

### อธิบายจุดสำคัญ

- `WS-PADDED` เก็บข้อความ `"  hi there  "` (มีช่องว่าง 2 ตัวทั้งหัวและท้าย) ตัวอย่างนี้ใช้ `STRING`
  (Part 019) ครอบผลลัพธ์ด้วยวงเล็บเหลี่ยม `[` และ `]` เพื่อให้เห็นช่องว่างที่เหลืออยู่ได้ชัดเจนใน
  `DISPLAY` (ปกติช่องว่างท้ายจะมองไม่เห็นด้วยตาเปล่า)
- `FUNCTION TRIM(WS-PADDED)` ตัดทั้งช่องว่างหัวและท้ายออกหมด เหลือ `hi there` ล้วน ๆ
- `FUNCTION TRIM(WS-PADDED LEADING)` ตัดเฉพาะช่องว่าง**หัว**ออก แต่ช่องว่างท้ายยังคงอยู่ (สังเกต
  จากผลลัพธ์ `[hi there  ]` ที่มีช่องว่างก่อนวงเล็บปิด)
- `FUNCTION TRIM(WS-PADDED TRAILING)` ตัดเฉพาะช่องว่าง**ท้าย**ออก แต่ช่องว่างหัวยังคงอยู่ (สังเกต
  จากผลลัพธ์ `[  hi there]` ที่มีช่องว่างหลังวงเล็บเปิด)
- คำ `LEADING`/`TRAILING` เขียนต่อท้ายอาร์กิวเมนต์ในวงเล็บเดียวกัน (`FUNCTION TRIM(field
  LEADING)`) ไม่ใช่อาร์กิวเมนต์แยกต่างหาก

### ข้อควรระวัง

- ผลลัพธ์ของ `FUNCTION TRIM` เป็นข้อความที่มีความยาว "ไดนามิก" ตามเนื้อหาจริง เมื่อนำไป `MOVE`
  เข้าฟิลด์ที่มีขนาดคงที่ (เช่น `PIC X(20)`) COBOL จะเติมช่องว่างด้านขวาให้เต็มฟิลด์เสมอ ทำให้เมื่อ
  `DISPLAY` ตัวแปรปลายทางตรง ๆ (ไม่ผ่าน `STRING` ใส่วงเล็บ) จะยังเห็นช่องว่างท้ายอยู่ดี — ต้องเข้าใจ
  ว่า `TRIM` ไม่ได้ "ลดขนาด" ตัวแปรปลายทาง มันแค่ลดความยาวของ**ข้อมูล**ที่ MOVE เข้าไปเท่านั้น
- ห้ามสับสนระหว่าง `FUNCTION TRIM(x LEADING)` กับ `FUNCTION TRIM(x, LEADING)` — ไม่มีจุลภาค
  คั่นระหว่างฟิลด์กับคำ `LEADING`/`TRAILING` ในไวยากรณ์ของ GnuCOBOL

### แบบฝึกหัดที่ 362.1

**โจทย์**: จงอธิบายว่าทำไมผลลัพธ์ของ `FUNCTION TRIM(WS-PADDED LEADING)` เมื่อแสดงผลผ่าน
`STRING` ครอบวงเล็บ ถึงยังมีช่องว่างเหลืออยู่ด้านในวงเล็บปิด

**เฉลย**: เพราะ `LEADING` สั่งให้ตัดเฉพาะช่องว่างที่อยู่**ก่อน**เนื้อหาจริงเท่านั้น ไม่แตะช่องว่างที่อยู่
**หลัง**เนื้อหา ในตัวอย่างนี้ `WS-PADDED` มีช่องว่าง 2 ตัวอยู่ท้ายข้อความด้วย (`"  hi there  "`)
เมื่อ `TRIM ... LEADING` ตัดเฉพาะช่องว่างหน้าออก ผลลัพธ์ที่ได้คือ `"hi there  "` (ยังมีช่องว่างท้าย
2 ตัวติดมาด้วย) เมื่อนำไปแสดงในวงเล็บ `[...]` จึงเห็นช่องว่างนั้นอยู่ก่อนวงเล็บปิดนั่นเอง

---

## ขั้นตอนที่ 363: FUNCTION LENGTH — หาความยาวของข้อมูล

### ความยาว "ที่ประกาศไว้" ไม่ใช่ความยาว "ที่มองเห็น"

`FUNCTION LENGTH(x)` คืนค่าจำนวนไบต์ (ตัวอักษร) ของ `x` **ตามขนาดที่ประกาศไว้ใน PICTURE**
เสมอ ไม่ใช่ความยาวของ "เนื้อหาที่มีความหมาย" หากต้องการความยาวของเนื้อหาจริง (ไม่รวมช่องว่างท้าย)
ต้องใช้ร่วมกับ `FUNCTION TRIM` จาก ขั้นตอนที่ 362 ก่อน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP363-LENGTH.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FIELD            PIC X(20) VALUE "  hello  ".
       01  WS-LEN-FULL         PIC 9(4).
       01  WS-LEN-TRIMMED      PIC 9(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> FUNCTION LENGTH always returns the DECLARED size of the
      *> field, including trailing spaces - not the "visible" text.
           COMPUTE WS-LEN-FULL = FUNCTION LENGTH(WS-FIELD).
           DISPLAY "LENGTH OF WS-FIELD (PIC X(20)): " WS-LEN-FULL.

      *> Combine with TRIM to get the length of just the content.
           COMPUTE WS-LEN-TRIMMED =
               FUNCTION LENGTH(FUNCTION TRIM(WS-FIELD)).
           DISPLAY "LENGTH AFTER TRIM: " WS-LEN-TRIMMED.
           STOP RUN.
```

**ผลลัพธ์:**

```
LENGTH OF WS-FIELD (PIC X(20)): 0020
LENGTH AFTER TRIM: 0005
```

### อธิบายจุดสำคัญ

- `FUNCTION LENGTH(WS-FIELD)` คืนค่า `20` เสมอ ไม่ว่าเนื้อหาจริงข้างในจะสั้นแค่ไหน เพราะ `WS-FIELD`
  ถูกประกาศเป็น `PIC X(20)` — นี่คือพฤติกรรมที่ผู้เริ่มต้นมักเข้าใจผิดบ่อยที่สุด: คิดว่า `LENGTH` จะ
  "ฉลาด" พอที่จะนับแค่ตัวอักษรที่มีความหมาย แต่จริง ๆ แล้วมันนับตามขนาดฟิลด์เสมอ
- `FUNCTION LENGTH(FUNCTION TRIM(WS-FIELD))` ซ้อนฟังก์ชัน (จาก Part 036 ขั้นตอนที่ 359) เพื่อ
  หาความยาวของเนื้อหาจริงหลังตัดช่องว่างหัวท้ายแล้ว ได้ผลลัพธ์ `5` (ความยาวของคำว่า "hello")
- รูปแบบ `FUNCTION LENGTH(FUNCTION TRIM(x))` เป็นสำนวนที่ใช้บ่อยมากในการตรวจสอบว่าผู้ใช้กรอก
  ข้อมูลมาหรือไม่ (เช่น ตรวจสอบว่าความยาวหลัง TRIM เป็น 0 หรือไม่ เพื่อรู้ว่าฟิลด์นั้นว่างเปล่าจริง ๆ
  หรือมีแค่ช่องว่าง)

### ข้อควรระวัง

- อย่าใช้ `FUNCTION LENGTH` เพียงลำพังเพื่อตรวจสอบว่า "ผู้ใช้กรอกข้อมูลมาหรือไม่" เพราะมันจะคืนค่า
  ขนาดฟิลด์เสมอไม่ว่าจะมีเนื้อหาจริงหรือไม่ ต้องใช้คู่กับ `FUNCTION TRIM` เสมอสำหรับวัตถุประสงค์นี้
- `FUNCTION LENGTH` ใช้ได้กับข้อมูลทุกประเภท (ตัวเลข, ตัวอักษร, กลุ่ม) ไม่ใช่แค่ข้อความ แต่ในบริบท
  ของ Part นี้เราเน้นการใช้กับข้อความเป็นหลัก

### แบบฝึกหัดที่ 363.1

**โจทย์**: จงเขียนโค้ดตรวจสอบว่า `WS-INPUT-NAME` (PIC X(30)) ที่รับมาจากผู้ใช้ ว่างเปล่าหรือไม่
(สมมติว่าผู้ใช้กรอกแค่ช่องว่างมาโดยไม่ตั้งใจ)

**เฉลย**:

```cobol
       01  WS-INPUT-NAME       PIC X(30) VALUE SPACES.
       01  WS-NAME-LEN         PIC 9(4).
       ...
           COMPUTE WS-NAME-LEN =
               FUNCTION LENGTH(FUNCTION TRIM(WS-INPUT-NAME)).
           IF WS-NAME-LEN = 0
               DISPLAY "ERROR: NAME FIELD IS EMPTY"
           ELSE
               DISPLAY "NAME LENGTH: " WS-NAME-LEN
           END-IF.
```

---

## ขั้นตอนที่ 364: FUNCTION NUMVAL และ FUNCTION NUMVAL-C — แปลงข้อความเป็นตัวเลข

### เมื่อตัวเลขมาในรูปแบบข้อความ

ข้อมูลที่รับมาจากไฟล์ภายนอก, หน้าจอผู้ใช้, หรือ API มักอยู่ในรูปแบบข้อความ (Alphanumeric) แม้จะ
"ดูเหมือน" ตัวเลขก็ตาม เช่น `"1234.56"` การจะนำไปคำนวณต้องแปลงเป็นข้อมูลตัวเลขจริงก่อน
`FUNCTION NUMVAL` และ `FUNCTION NUMVAL-C` ทำหน้าที่นี้

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP364-NUMVAL.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-TEXT-PLAIN       PIC X(10) VALUE "  1234.56".
       01  WS-TEXT-CURRENCY    PIC X(12) VALUE "$1,234.56".
       01  WS-TEXT-NEGATIVE    PIC X(10) VALUE "-99.90".
       01  WS-NUM              PIC S9(6)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> FUNCTION NUMVAL converts a plain numeric-looking string
      *> (digits, optional sign, optional decimal point) to a number.
           COMPUTE WS-NUM = FUNCTION NUMVAL(WS-TEXT-PLAIN).
           DISPLAY "NUMVAL PLAIN:    " WS-NUM.

           COMPUTE WS-NUM = FUNCTION NUMVAL(WS-TEXT-NEGATIVE).
           DISPLAY "NUMVAL NEGATIVE: " WS-NUM.

      *> FUNCTION NUMVAL-C also understands currency symbols and
      *> thousands separators - plain NUMVAL would reject them.
           COMPUTE WS-NUM = FUNCTION NUMVAL-C(WS-TEXT-CURRENCY).
           DISPLAY "NUMVAL-C CURRENCY: " WS-NUM.
           STOP RUN.
```

**ผลลัพธ์:**

```
NUMVAL PLAIN:    +001234.56
NUMVAL NEGATIVE: -000099.90
NUMVAL-C CURRENCY: +001234.56
```

### อธิบายจุดสำคัญ

- `FUNCTION NUMVAL` แปลงข้อความที่มีเฉพาะตัวเลข เครื่องหมายบวก/ลบ และจุดทศนิยม ให้เป็นค่าตัวเลข
  จริง โดยไม่สนใจช่องว่างหน้า/หลัง (`"  1234.56"` แปลงได้ปกติ)
- `FUNCTION NUMVAL-C` (C ย่อมาจาก Currency) รองรับเพิ่มเติม: เครื่องหมายสกุลเงิน (`$`) และตัวคั่น
  หลักพัน (`,`) ทำให้แปลง `"$1,234.56"` เป็น `1234.56` ได้สำเร็จ ในขณะที่ `FUNCTION NUMVAL`
  ธรรมดาจะทำไม่ได้กับข้อความรูปแบบนี้ (ดูตัวอย่างความล้มเหลวด้านล่าง)

### ข้อควรระวัง: NUMVAL กับข้อความสกุลเงิน คืนค่าศูนย์เงียบ ๆ (เหมือน SQRT ติดลบ)

ลองดูสิ่งที่เกิดขึ้นเมื่อใช้ `FUNCTION NUMVAL` (ไม่ใช่ `NUMVAL-C`) กับข้อความที่มีเครื่องหมาย
สกุลเงินปน:

```cobol
           COMPUTE WS-NUM = FUNCTION NUMVAL(WS-TEXT-CURRENCY).
           DISPLAY "NUMVAL ON CURRENCY TEXT: " WS-NUM.
```

**ผลลัพธ์:**

```
NUMVAL ON CURRENCY TEXT: +000000.00
```

เช่นเดียวกับ `FUNCTION SQRT` ของค่าติดลบใน Part 036 ขั้นตอนที่ 352 ฟังก์ชันนี้**ไม่แจ้ง Error
ใด ๆ** เมื่อรับข้อความที่แปลงไม่ได้ มันเพียงคืนค่า `0` เงียบ ๆ นี่เป็นอันตรายอย่างยิ่งในระบบที่ประมวลผล
ข้อมูลจากภายนอก (เช่น ไฟล์ CSV หรือ Feed จากระบบอื่น) เพราะข้อมูลผิดรูปแบบจะกลายเป็น `0`
โดยไม่มีการแจ้งเตือน อาจทำให้ยอดเงินในรายงานผิดพลาดโดยไม่มีใครสังเกตเห็น

**แนวทางป้องกัน**: หากไม่แน่ใจว่าข้อความจะมีสัญลักษณ์สกุลเงินปนหรือไม่ ให้ใช้ `FUNCTION NUMVAL-C`
เป็นค่าเริ่มต้นเสมอ เพราะมันรองรับทั้งข้อความตัวเลขธรรมดาและข้อความสกุลเงิน (ผลลัพธ์ของ `NUMVAL`
ธรรมดากับ `NUMVAL-C` จะเหมือนกันทุกประการเมื่อข้อความไม่มีสัญลักษณ์สกุลเงินปนอยู่)

### แบบฝึกหัดที่ 364.1

**โจทย์**: จงเขียนโค้ดแปลงข้อความ `"1,000.00"` (มีตัวคั่นหลักพันแต่ไม่มีเครื่องหมายสกุลเงิน) เป็น
ตัวเลข แล้วอธิบายว่าทำไมต้องใช้ `NUMVAL-C` ไม่ใช่ `NUMVAL`

**เฉลย**:

```cobol
       01  WS-TEXT-THOUSAND    PIC X(10) VALUE "1,000.00".
       01  WS-RESULT           PIC S9(7)V99.
       ...
           COMPUTE WS-RESULT = FUNCTION NUMVAL-C(WS-TEXT-THOUSAND).
           DISPLAY "RESULT: " WS-RESULT.
```

ต้องใช้ `NUMVAL-C` เพราะเครื่องหมายจุลภาค (`,`) ที่คั่นหลักพันไม่ใช่ตัวอักษรที่ `FUNCTION NUMVAL`
ธรรมดารองรับ (มันรองรับเฉพาะตัวเลข เครื่องหมาย +/- และจุดทศนิยมเท่านั้น) ในขณะที่ `NUMVAL-C`
ถูกออกแบบมาให้เข้าใจรูปแบบตัวเลขที่ใช้ในทางการเงิน ซึ่งรวมถึงตัวคั่นหลักพันด้วย

---

## ขั้นตอนที่ 365: FUNCTION SUM — รวมยอดโดยไม่ต้องวนลูปเอง

### รวมค่าจากหลายฟิลด์หรือทั้งตารางในคำสั่งเดียว

`FUNCTION SUM` รับอาร์กิวเมนต์ได้หลายตัว (ทั้งตัวแปรเดี่ยวและสมาชิกของตาราง) แล้วรวมผลทั้งหมดเข้า
ด้วยกัน ช่วยลดความจำเป็นในการเขียน `PERFORM VARYING` วนบวกทีละตัวสำหรับกรณีที่ไม่ซับซ้อนมาก

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP365-SUM.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  SALES-TABLE.
           05  SALES-AMT       PIC S9(6)V99 OCCURS 5 TIMES.
       01  WS-TOTAL            PIC S9(8)V99.
       01  WS-IDX              PIC 9.

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 1200.50 TO SALES-AMT(1).
           MOVE  850.00 TO SALES-AMT(2).
           MOVE 3000.75 TO SALES-AMT(3).
           MOVE  199.99 TO SALES-AMT(4).
           MOVE  500.00 TO SALES-AMT(5).

      *> FUNCTION SUM accepts a list of items directly - no PERFORM
      *> loop needed to add them up.
           COMPUTE WS-TOTAL = FUNCTION SUM(SALES-AMT(1)
               SALES-AMT(2) SALES-AMT(3) SALES-AMT(4) SALES-AMT(5)).
           DISPLAY "TOTAL SALES (LISTED ARGS): " WS-TOTAL.

      *> SUM can mix table items and plain literals in one call.
           COMPUTE WS-TOTAL = FUNCTION SUM(SALES-AMT(1) 100).
           DISPLAY "SUM WITH A LITERAL ADDED: " WS-TOTAL.
           STOP RUN.
```

**ผลลัพธ์:**

```
TOTAL SALES (LISTED ARGS): +00005751.24
SUM WITH A LITERAL ADDED: +00001300.50
```

### อธิบายจุดสำคัญ

- `FUNCTION SUM(SALES-AMT(1) SALES-AMT(2) ... SALES-AMT(5))` รวมสมาชิกทั้ง 5 ตัวของตาราง
  `SALES-AMT` ให้ผลลัพธ์ `5751.24` (1200.50 + 850.00 + 3000.75 + 199.99 + 500.00) ในคำสั่ง
  เดียว ไม่ต้องเขียน `PERFORM VARYING ... ADD SALES-AMT(WS-IDX) TO WS-TOTAL` เอง
- `FUNCTION SUM` รับได้ทั้งตัวแปรและ Literal ปนกัน (`SALES-AMT(1) 100`) แสดงให้เห็นความยืดหยุ่น
  ของอาร์กิวเมนต์
- ข้อจำกัดที่พบจากการทดสอบจริง: ไวยากรณ์แบบ `SALES-AMT(ALL)` (ที่หลายคนคาดหวังว่าจะรวมสมาชิก
  ทั้งหมดของตารางโดยไม่ต้องระบุทีละตัว) **ไม่ได้รับการรองรับ** ใน GnuCOBOL เวอร์ชันนี้ — ต้องระบุ
  ดัชนี (Subscript) ของแต่ละสมาชิกที่ต้องการรวมอย่างชัดเจนเสมอ

### ข้อควรระวัง

- สำหรับตารางขนาดใหญ่หรือขนาดที่ไม่แน่นอน (เช่น จำนวนสมาชิกที่ใช้งานจริงเปลี่ยนแปลงตามข้อมูล
  นำเข้า) `FUNCTION SUM` ที่ต้องระบุทุก Subscript เองไม่สะดวกเท่ากับการใช้ `PERFORM VARYING`
  วนบวกตามจำนวนสมาชิกจริง (`WS-ITEM-COUNT`) ควรเลือกใช้ให้เหมาะกับสถานการณ์: `FUNCTION SUM`
  เหมาะกับตารางขนาดคงที่และทราบจำนวนแน่นอน ส่วนตารางที่ขนาดแปรผันควรใช้ลูป
- ผลลัพธ์จาก `FUNCTION SUM` ควรถูกกำหนดขนาดฟิลด์ปลายทางให้ใหญ่พอรองรับผลรวมสูงสุดที่เป็นไปได้
  เพื่อป้องกัน Overflow

### แบบฝึกหัดที่ 365.1

**โจทย์**: จงเขียนโค้ดหาผลรวมของสมาชิกเพียง 3 ตัวแรกของ `SALES-AMT` เท่านั้น (ไม่รวมตัวที่ 4
และ 5)

**เฉลย**:

```cobol
           COMPUTE WS-TOTAL =
               FUNCTION SUM(SALES-AMT(1) SALES-AMT(2) SALES-AMT(3)).
           DISPLAY "TOTAL OF FIRST 3: " WS-TOTAL.
```

ผลลัพธ์ที่คาดไว้: `TOTAL OF FIRST 3: +00005051.25` (1200.50 + 850.00 + 3000.75)

---

## ขั้นตอนที่ 366: ฟังก์ชันสถิติ — MEAN, MEDIAN, VARIANCE, STANDARD-DEVIATION, RANGE

### เมื่อ COBOL ทำงานวิเคราะห์ข้อมูลเชิงสถิติได้ในตัว

COBOL มาตรฐานสมัยใหม่มีฟังก์ชันสถิติพื้นฐานติดตัวมาให้ครบถ้วน เหมาะกับงานวิเคราะห์ข้อมูลทางธุรกิจ
เช่น หาค่าเฉลี่ยยอดขาย หรือดูการกระจายตัวของข้อมูล เราได้ทดสอบทุกฟังก์ชันในหัวข้อนี้จริงแล้วและ
ยืนยันว่าใช้งานได้ทั้งหมดใน GnuCOBOL

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP366-STATS.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RESULT           PIC S9(8)V999.

       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-RESULT = FUNCTION MEAN(10 20 30 40).
           DISPLAY "MEAN(10,20,30,40)     = " WS-RESULT.

           COMPUTE WS-RESULT = FUNCTION MEDIAN(10 20 30 40).
           DISPLAY "MEDIAN(10,20,30,40)   = " WS-RESULT.

           COMPUTE WS-RESULT = FUNCTION VARIANCE(2 4 4 4 5 5 7 9).
           DISPLAY "VARIANCE(...)         = " WS-RESULT.

      *> FUNCTION STANDARD-DEVIATION is the square root of the
      *> variance - notice the COMPUTE line must be split across
      *> two lines to stay inside column 72 (the function name
      *> alone is long).
           COMPUTE WS-RESULT =
               FUNCTION STANDARD-DEVIATION(2 4 4 4 5 5 7 9).
           DISPLAY "STANDARD-DEVIATION(...) = " WS-RESULT.

           COMPUTE WS-RESULT = FUNCTION RANGE(3 88 21 6).
           DISPLAY "RANGE(3,88,21,6)      = " WS-RESULT.
           STOP RUN.
```

**ผลลัพธ์:**

```
MEAN(10,20,30,40)     = +00000025.000
MEDIAN(10,20,30,40)   = +00000025.000
VARIANCE(...)         = +00000004.000
STANDARD-DEVIATION(...) = +00000002.000
RANGE(3,88,21,6)      = +00000085.000
```

### อธิบายจุดสำคัญ

- `FUNCTION MEAN(10 20 30 40) = 25` — ค่าเฉลี่ยเลขคณิตธรรมดา ((10+20+30+40)/4 = 25)
- `FUNCTION MEDIAN(10 20 30 40) = 25` — ค่ามัธยฐาน (เมื่อจำนวนสมาชิกเป็นเลขคู่ จะใช้ค่าเฉลี่ยของ
  สองตัวกลาง คือ 20 และ 30 ได้ผล 25 พอดีในกรณีนี้)
- `FUNCTION VARIANCE(2 4 4 4 5 5 7 9) = 4` และ `FUNCTION STANDARD-DEVIATION(...) = 2` —
  สังเกตว่า Standard Deviation คือรากที่สองของ Variance พอดี (√4 = 2) ตรงตามหลักสถิติพื้นฐาน
- `FUNCTION RANGE(3 88 21 6) = 85` — ผลต่างระหว่างค่ามากที่สุด (88) กับค่าน้อยที่สุด (3) เทียบเท่า
  กับการเขียน `FUNCTION MAX(...) - FUNCTION MIN(...)` เอง แต่กระชับกว่า

### ข้อควรระวัง: ชื่อฟังก์ชันยาว ต้องแยกบรรทัดเสมอ

สังเกตในโค้ดตัวอย่างว่า `FUNCTION STANDARD-DEVIATION` ถูกเขียนแยกบรรทัดจาก `COMPUTE
WS-RESULT =` ในขณะที่ฟังก์ชันอื่น (`MEAN`, `MEDIAN`, `VARIANCE`, `RANGE`) เขียนในบรรทัดเดียวกัน
ได้ นี่ไม่ใช่ความชอบส่วนตัว แต่เป็นเพราะชื่อ `STANDARD-DEVIATION` ยาวถึง 19 ตัวอักษร เมื่อรวมกับ
`COMPUTE WS-RESULT = FUNCTION ` และอาร์กิวเมนต์ `(2 4 4 4 5 5 7 9).` แล้ว **ความยาวรวมจะเกิน
คอลัมน์ 72 ทันที** จากการทดสอบจริงพบว่าบรรทัดเดียวกันนี้ (ไม่แยกบรรทัด) มีความยาว 76 ไบต์ ทำให้
คอมไพเลอร์ Error ว่า `syntax error, unexpected DISPLAY` เนื่องจากส่วนที่เกินคอลัมน์ 72 ถูกตัดทิ้ง
ไปตามกฎ Fixed-Format

**บทเรียนสำคัญ**: ก่อนสรุปว่าฟังก์ชันใดฟังก์ชันหนึ่ง "ใช้ไม่ได้" ในคอมไพเลอร์ ให้ตรวจสอบความยาว
บรรทัดก่อนเสมอ โดยเฉพาะเมื่อฟังก์ชันมีชื่อยาว เพราะ Error message ที่ได้ (`syntax error, unexpected
DISPLAY`) ไม่ได้บอกตรง ๆ ว่าปัญหาคือคอลัมน์ 72 เกิน ทำให้เข้าใจผิดได้ง่ายว่าฟังก์ชันนั้นไม่รองรับ

### แบบฝึกหัดที่ 366.1

**โจทย์**: จงคำนวณ Coefficient of Variation (สัมประสิทธิ์การแปรผัน = Standard Deviation หารด้วย
Mean) ของชุดข้อมูล `2 4 4 4 5 5 7 9` โดยใช้ฟังก์ชันสถิติที่เรียนมา

**เฉลย**:

```cobol
       01  WS-SD               PIC S9(8)V999.
       01  WS-MEAN-VAL         PIC S9(8)V999.
       01  WS-CV               PIC S9(4)V9999.
       ...
           COMPUTE WS-SD =
               FUNCTION STANDARD-DEVIATION(2 4 4 4 5 5 7 9).
           COMPUTE WS-MEAN-VAL = FUNCTION MEAN(2 4 4 4 5 5 7 9).
           COMPUTE WS-CV = WS-SD / WS-MEAN-VAL.
           DISPLAY "COEFFICIENT OF VARIATION: " WS-CV.
```

ผลลัพธ์ที่คาดไว้: SD = 2, Mean = 5 ดังนั้น CV = 2/5 = 0.4

---

## ขั้นตอนที่ 367: FUNCTION ORD, FUNCTION ORD-MAX, FUNCTION ORD-MIN

### ORD ไม่ใช่ ASCII Code โดยตรง แต่เป็นตำแหน่งในลำดับ

`FUNCTION ORD(x)` คืนค่า **ตำแหน่งลำดับ (Ordinal Position)** ของตัวอักษร `x` ในชุดอักขระ
(Collating Sequence) ของระบบ **โดยนับเริ่มจาก 1** (ไม่ใช่ 0) ต่างจาก ASCII Code ที่คุ้นเคย (ซึ่ง
เริ่มนับจาก 0) อยู่เสมอ 1 หน่วย ส่วน `FUNCTION ORD-MAX`/`ORD-MIN` คืนค่า **ตำแหน่งของอาร์กิวเมนต์**
ที่มีค่ามากที่สุด/น้อยที่สุด (ไม่ใช่ค่านั้นเอง)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP367-ORD.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-POS              PIC 9(4).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> FUNCTION ORD returns the ordinal position of a character
      *> in the program's collating sequence (1-based, NOT the
      *> ASCII code itself).
           COMPUTE WS-POS = FUNCTION ORD("A").
           DISPLAY "ORD(A) = " WS-POS.
           COMPUTE WS-POS = FUNCTION ORD("B").
           DISPLAY "ORD(B) = " WS-POS.
           COMPUTE WS-POS = FUNCTION ORD("a").
           DISPLAY "ORD(a) = " WS-POS.

      *> ORD-MAX / ORD-MIN return the POSITION (1st, 2nd, 3rd...)
      *> of the highest / lowest argument, not the value itself.
           MOVE FUNCTION ORD-MAX("A" "Z" "M") TO WS-POS.
           DISPLAY "ORD-MAX(A,Z,M), POSITION OF Z = " WS-POS.
           MOVE FUNCTION ORD-MIN("A" "Z" "M") TO WS-POS.
           DISPLAY "ORD-MIN(A,Z,M), POSITION OF A = " WS-POS.
           STOP RUN.
```

**ผลลัพธ์:**

```
ORD(A) = 0066
ORD(B) = 0067
ORD(a) = 0098
ORD-MAX(A,Z,M), POSITION OF Z = 0002
ORD-MIN(A,Z,M), POSITION OF A = 0001
```

### อธิบายจุดสำคัญ

- `FUNCTION ORD("A") = 66` — ASCII Code จริงของ `"A"` คือ `65` แต่ `ORD` คืนค่า `66` เพราะนับ
  ลำดับเริ่มจาก **1** ไม่ใช่ **0** (ตำแหน่งแรกสุดในชุดอักขระคือค่า ASCII `0` ซึ่งมีลำดับที่ `1`
  ดังนั้นตัวอักษรที่มี ASCII Code `65` จึงอยู่ในลำดับที่ `66`) กฎง่าย ๆ คือ **`ORD = ASCII Code +
  1` เสมอ**
- `FUNCTION ORD("B") = 67` และ `FUNCTION ORD("a") = 98` ยืนยันกฎเดียวกัน (ASCII ของ B คือ 66,
  ของ a ตัวเล็กคือ 97)
- `FUNCTION ORD-MAX("A" "Z" "M")` หาว่าอาร์กิวเมนต์ตัวไหนมีค่ามากที่สุด (คือ `"Z"`) แล้วคืนค่า
  **ตำแหน่งของมันในรายการอาร์กิวเมนต์** (ไม่ใช่ค่าของมันเอง) — `"Z"` อยู่ในตำแหน่งที่ 2 ของรายการ
  `("A" "Z" "M")` จึงได้ผลลัพธ์ `2` ไม่ใช่ค่า ORD ของ Z
- ในทำนองเดียวกัน `FUNCTION ORD-MIN("A" "Z" "M")` ได้ `1` เพราะ `"A"` (ค่าน้อยที่สุด) อยู่ในตำแหน่ง
  ที่ 1 ของรายการ

### ข้อควรระวัง

- ข้อผิดพลาดที่พบบ่อยที่สุดคือคิดว่า `ORD-MAX`/`ORD-MIN` คืน "ค่า" ที่มากที่สุด/น้อยที่สุด (เหมือน
  `FUNCTION MAX`/`MIN` จาก Part 036) แต่จริง ๆ แล้วมันคืน**ตำแหน่ง (Index)** ของอาร์กิวเมนต์นั้น
  ในรายการที่ส่งเข้าไป ถ้าต้องการค่าจริงต้องใช้ `FUNCTION MAX`/`MIN` แทน
- อย่าลืมว่า `ORD` นับเริ่มจาก 1 ไม่ใช่ 0 หากนำไปคำนวณต่อโดยลืมจุดนี้ (เช่น พยายามใช้ผลลัพธ์เป็น
  ASCII Code ตรง ๆ) จะได้ค่าที่คลาดเคลื่อนไป 1 หน่วยเสมอ

### แบบฝึกหัดที่ 367.1

**โจทย์**: จงเขียนโค้ดหาตำแหน่ง (Ordinal Position) ของตัวอักษร `"Z"` แล้วคำนวณย้อนกลับเป็น
ASCII Code จริงของมัน

**เฉลย**:

```cobol
       01  WS-ORD-POS          PIC 9(4).
       01  WS-ASCII-CODE       PIC 9(4).
       ...
           COMPUTE WS-ORD-POS = FUNCTION ORD("Z").
           COMPUTE WS-ASCII-CODE = WS-ORD-POS - 1.
           DISPLAY "ORD(Z) = " WS-ORD-POS.
           DISPLAY "ASCII CODE OF Z = " WS-ASCII-CODE.
```

ผลลัพธ์ที่คาดไว้: `ORD(Z) = 0091` และ `ASCII CODE OF Z = 0090` (ASCII จริงของ Z คือ 90)

---

## ขั้นตอนที่ 368: FUNCTION REVERSE และ FUNCTION CONCATENATE

### REVERSE — กลับลำดับตัวอักษร

`FUNCTION REVERSE(x)` คืนค่าข้อความที่กลับลำดับตัวอักษรทั้งหมดของ `x` (รวมช่องว่างด้วย) มี
ประโยชน์ในงานตรวจสอบ Palindrome หรือการแสดงผลบางรูปแบบพิเศษ

### CONCATENATE — ต่อข้อความหลายส่วนเข้าด้วยกัน

`FUNCTION CONCATENATE(x y z ...)` ต่อข้อความหลายฟิลด์เข้าด้วยกันเป็นค่าเดียว คล้ายกับ `STRING`
(Part 019) แต่ใช้ในรูปแบบนิพจน์ (Expression) ได้โดยตรงโดยไม่ต้องมี `INTO` แยกบรรทัด

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP368-REVERSE-CONCAT.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-NAME             PIC X(15) VALUE "SOMCHAI".
       01  WS-REV              PIC X(15).
       01  WS-FIRST            PIC X(15) VALUE "  INVOICE  ".
       01  WS-SECOND           PIC X(10) VALUE "-2026-09".
       01  WS-CONCAT           PIC X(30).

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE FUNCTION REVERSE(WS-NAME) TO WS-REV.
           DISPLAY "REVERSE(SOMCHAI) = [" WS-REV "]".

      *> FUNCTION CONCATENATE joins several fields into one.
           MOVE FUNCTION CONCATENATE(WS-FIRST WS-SECOND)
               TO WS-CONCAT.
           DISPLAY "CONCATENATE (WITH PADDING) = [" WS-CONCAT "]".

      *> A FUNCTION call CAN be nested inside CONCATENATE's own
      *> argument list - the compiler accepts this fine, as long
      *> as the line is split so it never crosses column 72.
           MOVE FUNCTION CONCATENATE(FUNCTION TRIM(WS-FIRST)
               WS-SECOND) TO WS-CONCAT.
           DISPLAY "CONCATENATE (TRIMMED FIRST) = [" WS-CONCAT "]".
           STOP RUN.
```

**ผลลัพธ์:**

```
REVERSE(SOMCHAI) = [        IAHCMOS]
CONCATENATE (WITH PADDING) = [  INVOICE      -2026-09       ]
CONCATENATE (TRIMMED FIRST) = [INVOICE-2026-09               ]
```

### อธิบายจุดสำคัญ

- `FUNCTION REVERSE(WS-NAME)` กลับข้อความ `"SOMCHAI         "` (มีช่องว่างท้ายเพราะฟิลด์เป็น
  `PIC X(15)`) เป็น `"        IAHCMOS"` — สังเกตว่าช่องว่างท้ายเดิม กลายเป็นช่องว่าง**หน้า**หลังกลับ
  ลำดับ เพราะ `REVERSE` กลับ**ทุกไบต์**ในฟิลด์ ไม่ได้ฉลาดพอที่จะรู้ว่าช่องว่างท้ายไม่ใช่เนื้อหา
- `FUNCTION CONCATENATE(WS-FIRST WS-SECOND)` ต่อ `WS-FIRST` (ที่มีช่องว่างหัวท้ายเต็ม 15 ตัวอักษร)
  กับ `WS-SECOND` เข้าด้วยกันตรง ๆ ทำให้เห็นช่องว่างตรงกลางที่มาจากขนาดฟิลด์ `WS-FIRST` เต็มจำนวน
- เมื่อต้องการต่อข้อความแบบ "แน่น" (ไม่มีช่องว่างเกินความจำเป็น) ต้อง `FUNCTION TRIM` ฟิลด์ก่อน
  ส่งเข้า `CONCATENATE` เสมอ ดังตัวอย่างที่สาม `FUNCTION CONCATENATE(FUNCTION TRIM(WS-FIRST)
  WS-SECOND)` ที่ได้ผลลัพธ์ `"INVOICE-2026-09"` แน่นสนิทไม่มีช่องว่างคั่นกลาง

### ข้อควรระวัง: จุดที่เคยเข้าใจผิดว่าเป็นข้อจำกัดของ CONCATENATE

ระหว่างการพัฒนาบทเรียนนี้ พบว่าการเขียน `FUNCTION CONCATENATE(FUNCTION TRIM(x) y)` **ใน
บรรทัดเดียว** (ไม่ตัดบรรทัด) ทำให้เกิด Compile Error `syntax error, unexpected DISPLAY, expecting
TO` ซึ่งตอนแรกดูเหมือนว่า `CONCATENATE` ไม่รองรับการซ้อนฟังก์ชันเป็นอาร์กิวเมนต์ของตัวเอง แต่เมื่อ
ตรวจสอบความยาวบรรทัดจริงพบว่ามันยาวถึง **85 ไบต์** — เกินคอลัมน์ 72 ไปมาก! เมื่อแยกนิพจน์เดียวกัน
ออกเป็น 2 บรรทัด (ดังในโค้ดตัวอย่างข้างบน) ก็คอมไพล์และรันได้ปกติทันที

**บทเรียนสำคัญที่สุดของ Part นี้**: `FUNCTION CONCATENATE` **รองรับ**การซ้อนฟังก์ชันอื่นเป็น
อาร์กิวเมนต์ของมันได้ตามปกติทุกประการ ปัญหาที่แท้จริงไม่เคยเป็นเรื่องของฟังก์ชันเลย แต่เป็นกฎเหล็ก
เรื่องคอลัมน์ 72 ของหลักสูตรนี้ที่ปรากฏขึ้นอีกครั้งในจุดที่ไม่คาดคิด — นี่คือเหตุผลที่ผู้เขียนโปรแกรม
COBOL มืออาชีพต้องนับความยาวบรรทัดเป็นนิสัย โดยเฉพาะเมื่อซ้อนฟังก์ชันที่มีชื่อยาวเข้าด้วยกันหลายชั้น

### แบบฝึกหัดที่ 368.1

**โจทย์**: จงเขียนโค้ดตรวจสอบว่าข้อความใน `WS-NAME` (ที่ผ่าน `FUNCTION TRIM` แล้ว) เป็น
Palindrome หรือไม่ (คำที่อ่านจากหน้าไปหลังกับหลังมาหน้าเหมือนกัน) โดยใช้ `FUNCTION REVERSE`

**เฉลย**:

```cobol
       01  WS-TEST-WORD        PIC X(10) VALUE "LEVEL".
       01  WS-REVERSED         PIC X(10).
       ...
           MOVE FUNCTION REVERSE(WS-TEST-WORD) TO WS-REVERSED.
           IF WS-TEST-WORD = WS-REVERSED
               DISPLAY "PALINDROME: YES"
           ELSE
               DISPLAY "PALINDROME: NO"
           END-IF.
```

ผลลัพธ์ที่คาดไว้: `PALINDROME: YES` (เพราะ "LEVEL" อ่านกลับก็ยังเป็น "LEVEL" — ทั้งสองฟิลด์มี
ขนาดเท่ากันคือ `PIC X(10)` ทำให้ช่องว่างท้ายที่เท่ากันไม่ทำให้การเปรียบเทียบผิดพลาด)

---

## ขั้นตอนที่ 369: การผสมผสานฟังก์ชันข้อความเพื่อทำความสะอาดข้อมูล (Data Cleaning)

### สถานการณ์จริง: ข้อมูลนำเข้าที่ไม่เป็นระเบียบ

งานที่พบบ่อยที่สุดในระบบธุรกิจจริงคือการรับข้อมูลจากแหล่งภายนอก (ไฟล์ที่ผู้ใช้กรอกเอง, ข้อมูลนำเข้า
จากระบบเก่า) ที่มักมีช่องว่างเกิน ตัวพิมพ์ไม่สม่ำเสมอ ปะปนกันอยู่ ขั้นตอนนี้จะสาธิตการผสมผสาน
`FUNCTION TRIM`, `FUNCTION UPPER-CASE`, และ `FUNCTION LENGTH` เข้าด้วยกันเพื่อทำความสะอาด
ข้อมูลชุดหนึ่งให้อยู่ในรูปแบบมาตรฐานเดียวกัน

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP369-DATA-CLEANING.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  RAW-NAMES.
           05  FILLER          PIC X(20) VALUE "  somchai jaidee".
           05  FILLER          PIC X(20) VALUE "SUDA MEECHAI  ".
           05  FILLER          PIC X(20) VALUE "  Prasert Kaewta ".
       01  RAW-NAME-TABLE REDEFINES RAW-NAMES.
           05  RAW-NAME        PIC X(20) OCCURS 3 TIMES.
       01  WS-CLEAN-NAME       PIC X(20).
       01  WS-IDX              PIC 9.

       PROCEDURE DIVISION.
       MAIN-PARA.
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 3
      *> One pass cleans messy input: trim spaces, force upper
      *> case, ready for a consistent report or a key comparison.
               MOVE FUNCTION UPPER-CASE(FUNCTION TRIM(
                   RAW-NAME(WS-IDX))) TO WS-CLEAN-NAME
               DISPLAY "CLEANED: [" WS-CLEAN-NAME "] LEN="
                   FUNCTION LENGTH(FUNCTION TRIM(RAW-NAME(WS-IDX)))
           END-PERFORM.
           STOP RUN.
```

**ผลลัพธ์:**

```
CLEANED: [SOMCHAI JAIDEE      ] LEN=14
CLEANED: [SUDA MEECHAI        ] LEN=12
CLEANED: [PRASERT KAEWTA      ] LEN=14
```

### อธิบายจุดสำคัญ

- `RAW-NAME-TABLE REDEFINES RAW-NAMES` (เทคนิคจาก Part 022) ใช้แปลงกลุ่มฟิลด์ `FILLER` สาม
  ฟิลด์ให้กลายเป็นตาราง `RAW-NAME OCCURS 3 TIMES` เพื่อให้วนลูปประมวลผลได้สะดวก
- ในลูปเดียว เราทำความสะอาดข้อมูลสามขั้นตอนพร้อมกัน: (1) `FUNCTION TRIM` ตัดช่องว่างหัวท้าย
  (ทั้งข้อมูลตัวที่ 1 ที่มีช่องว่างหน้า ตัวที่ 2 ที่มีช่องว่างท้าย และตัวที่ 3 ที่มีทั้งสองแบบ) (2)
  `FUNCTION UPPER-CASE` แปลงเป็นตัวพิมพ์ใหญ่ทั้งหมดเพื่อความสม่ำเสมอ (3) `FUNCTION LENGTH`
  ร่วมกับ `TRIM` แสดงความยาวจริงของแต่ละชื่อ (ไม่รวมช่องว่าง)
- ผลลัพธ์แสดงให้เห็นว่าไม่ว่าข้อมูลต้นทางจะมีช่องว่างวางแบบใด (หน้า, หลัง, หรือทั้งสองแบบ) หลังผ่าน
  กระบวนการนี้แล้วทุกชื่อจะอยู่ในรูปแบบมาตรฐานเดียวกัน (ตัวพิมพ์ใหญ่, ไม่มีช่องว่างหัว)

### ข้อควรระวัง

- การซ้อนฟังก์ชัน 2-3 ชั้นในลูปแบบนี้ ทำให้ต้องตัดบรรทัดหลายจุดเพื่อรักษากฎคอลัมน์ 72 ควรวางแผน
  การตัดบรรทัดให้อ่านง่าย (เช่น ตัดตรงจุดที่เป็นวงเล็บเปิดฟังก์ชันชั้นใน เหมือนตัวอย่างข้างบน) แทนที่
  จะตัดแบบสุ่มซึ่งจะทำให้อ่านโค้ดยากขึ้นมาก
- เมื่อทำความสะอาดข้อมูลจำนวนมาก (เช่น อ่านจากไฟล์นับพันระเบียน) ควรพิจารณาประสิทธิภาพด้วย
  การเรียกฟังก์ชันซ้อนกันหลายชั้นในลูปขนาดใหญ่มีค่าใช้จ่ายด้านการประมวลผลสะสม แม้จะไม่มากในระดับ
  ที่สังเกตเห็นได้สำหรับข้อมูลขนาดปกติก็ตาม

### แบบฝึกหัดที่ 369.1

**โจทย์**: จงขยายโปรแกรมข้างบนให้ตรวจสอบด้วยว่าชื่อใดมีความยาว (หลัง TRIM) น้อยกว่า 10
ตัวอักษร แล้วแสดงข้อความเตือนพิเศษ

**เฉลย**:

```cobol
       01  WS-NAME-LEN         PIC 9(4).
       ...
               COMPUTE WS-NAME-LEN =
                   FUNCTION LENGTH(FUNCTION TRIM(RAW-NAME(WS-IDX)))
               IF WS-NAME-LEN < 10
                   DISPLAY "  WARNING: NAME SEEMS TOO SHORT"
               END-IF
```

เพิ่มโค้ดนี้ต่อท้ายภายในลูป `PERFORM VARYING` เดิม

---

## ขั้นตอนที่ 370: สรุปรวม — โปรแกรมตรวจสอบและแปลงข้อมูล (Data Validator)

### รวมฟังก์ชันข้อความและสถิติทั้งหมดของ Part นี้เข้าด้วยกัน

ขั้นตอนสุดท้ายของ Part นี้จะรวมฟังก์ชันข้อความ (`TRIM`, `UPPER-CASE`, `LENGTH`, `NUMVAL-C`)
และฟังก์ชันสถิติ (`SUM`, `MEAN`, `MAX`) เข้าด้วยกันในโปรแกรมเดียว จำลองสถานการณ์จริง: รับ
ข้อมูลชื่อลูกค้าและยอดเงินจากแหล่งภายนอกที่มีรูปแบบไม่เป็นระเบียบ (มีช่องว่างเกิน, มีสัญลักษณ์
สกุลเงินและตัวคั่นหลักพัน) แล้วแปลงให้เป็นข้อมูลที่ใช้งานได้จริงพร้อมสรุปผลทางสถิติ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP370-DATA-VALIDATOR.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RAW-NAME         PIC X(20) VALUE "  somchai jaidee  ".
       01  WS-CLEAN-NAME       PIC X(20).
       01  WS-NAME-LEN         PIC 9(4).
       01  WS-RAW-AMOUNTS.
           05  FILLER          PIC X(10) VALUE " 1,250.00 ".
           05  FILLER          PIC X(10) VALUE "   999.50 ".
           05  FILLER          PIC X(10) VALUE " 3,000.75 ".
       01  WS-AMOUNT-TABLE REDEFINES WS-RAW-AMOUNTS.
           05  WS-RAW-AMT      PIC X(10) OCCURS 3 TIMES.
       01  WS-NUM-AMOUNTS.
           05  WS-AMT          PIC S9(7)V99 OCCURS 3 TIMES.
       01  WS-IDX              PIC 9.
       01  WS-TOTAL            PIC S9(8)V99.
       01  WS-AVERAGE          PIC S9(8)V99.
       01  WS-HIGHEST          PIC S9(8)V99.

       PROCEDURE DIVISION.
       MAIN-PARA.
           DISPLAY "==== DATA VALIDATOR (STEP 370) ====".

      *> Step 1: clean up the free-text name field.
           MOVE FUNCTION UPPER-CASE(FUNCTION TRIM(WS-RAW-NAME))
               TO WS-CLEAN-NAME.
           COMPUTE WS-NAME-LEN =
               FUNCTION LENGTH(FUNCTION TRIM(WS-RAW-NAME)).
           DISPLAY "CLEAN NAME: [" WS-CLEAN-NAME "]".
           DISPLAY "NAME LENGTH: " WS-NAME-LEN.

      *> Step 2: convert every currency-formatted amount to a
      *> real numeric field.
           PERFORM VARYING WS-IDX FROM 1 BY 1 UNTIL WS-IDX > 3
               COMPUTE WS-AMT(WS-IDX) =
                   FUNCTION NUMVAL-C(WS-RAW-AMT(WS-IDX))
           END-PERFORM.

      *> Step 3: run the numbers through the statistical functions.
           COMPUTE WS-TOTAL =
               FUNCTION SUM(WS-AMT(1) WS-AMT(2) WS-AMT(3)).
           COMPUTE WS-AVERAGE =
               FUNCTION MEAN(WS-AMT(1) WS-AMT(2) WS-AMT(3)).
           COMPUTE WS-HIGHEST =
               FUNCTION MAX(WS-AMT(1) WS-AMT(2) WS-AMT(3)).
           DISPLAY "TOTAL AMOUNT:   " WS-TOTAL.
           DISPLAY "AVERAGE AMOUNT: " WS-AVERAGE.
           DISPLAY "HIGHEST AMOUNT: " WS-HIGHEST.
           STOP RUN.
```

**ผลลัพธ์:**

```
==== DATA VALIDATOR (STEP 370) ====
CLEAN NAME: [SOMCHAI JAIDEE      ]
NAME LENGTH: 0014
TOTAL AMOUNT:   +00005250.25
AVERAGE AMOUNT: +00001750.08
HIGHEST AMOUNT: +00003000.75
```

### อธิบายจุดสำคัญ

- **ขั้นที่ 1** ทำความสะอาดชื่อลูกค้าด้วยเทคนิคเดียวกับ ขั้นตอนที่ 369: `TRIM` ตัดช่องว่าง แล้ว
  `UPPER-CASE` แปลงตัวพิมพ์ ได้ชื่อที่สะอาดพร้อมนำไปเก็บหรือเปรียบเทียบต่อ
- **ขั้นที่ 2** แปลงข้อความยอดเงินที่มีรูปแบบสกุลเงินจริง (มีช่องว่างรอบ ๆ, มีตัวคั่นหลักพัน) เป็นค่า
  ตัวเลขจริงด้วย `FUNCTION NUMVAL-C` ในลูป `PERFORM VARYING` — สังเกตว่า `NUMVAL-C` จัดการ
  ทั้งช่องว่างและตัวคั่นหลักพันได้ในฟังก์ชันเดียว ไม่ต้อง `TRIM` ก่อนแยกต่างหาก
- **ขั้นที่ 3** นำค่าตัวเลขที่แปลงแล้วไปผ่านฟังก์ชันสถิติทั้งสาม (`SUM`, `MEAN`, `MAX`) ได้ผลรวม
  ค่าเฉลี่ย และค่าสูงสุดของยอดเงินทั้งสามรายการในคำสั่งสั้น ๆ เพียงไม่กี่บรรทัด
- โปรแกรมนี้แสดงให้เห็นภาพรวมของ Part 036-037 ทั้งสอง Part ที่ทำงานร่วมกันได้อย่างกลมกลืน:
  ฟังก์ชันข้อความจัดการความไม่เป็นระเบียบของข้อมูลนำเข้า และฟังก์ชันตัวเลข/สถิติสรุปผลลัพธ์สุดท้าย

### ข้อควรระวังสรุปรวมทั้ง Part

- ฟังก์ชันที่ยืนยันว่าใช้งานได้จริงใน GnuCOBOL เวอร์ชันนี้ (ทดสอบคอมไพล์และรันจริงแล้วทั้งหมด):
  `UPPER-CASE`, `LOWER-CASE`, `TRIM` (พร้อม `LEADING`/`TRAILING`), `LENGTH`, `NUMVAL`,
  `NUMVAL-C`, `SUM`, `MEAN`, `MEDIAN`, `VARIANCE`, `STANDARD-DEVIATION`, `RANGE`, `ORD`,
  `ORD-MAX`, `ORD-MIN`, `REVERSE`, `CONCATENATE`
- `FUNCTION NUMVAL` (ไม่ใช่ `NUMVAL-C`) คืนค่า `0` เงียบ ๆ เมื่อรับข้อความที่มีสัญลักษณ์สกุลเงินหรือ
  ตัวคั่นหลักพัน — ควรใช้ `NUMVAL-C` เป็นค่าเริ่มต้นเมื่อไม่แน่ใจรูปแบบข้อมูลนำเข้า
- `FUNCTION SUM(x(ALL))` (รวมทั้งตารางโดยไม่ระบุ Subscript ทีละตัว) ไม่ได้รับการรองรับใน
  GnuCOBOL เวอร์ชันนี้ ต้องระบุ Subscript ของแต่ละสมาชิกอย่างชัดเจน หรือใช้ `PERFORM VARYING`
  วนบวกเองสำหรับตารางขนาดแปรผัน
- `FUNCTION ORD` นับเริ่มจาก 1 (เท่ากับ ASCII Code + 1) และ `ORD-MAX`/`ORD-MIN` คืน**ตำแหน่ง**
  ของอาร์กิวเมนต์ที่มีค่ามากสุด/น้อยสุด ไม่ใช่ค่านั้นเอง
- ข้อค้นพบสำคัญที่สุด: กรณีที่ดูเหมือนฟังก์ชัน (`STANDARD-DEVIATION`, `CONCATENATE` กับฟังก์ชัน
  ซ้อน) "ใช้ไม่ได้" ในการทดสอบเบื้องต้น แท้จริงแล้วเป็นเพราะ**ความยาวบรรทัดเกินคอลัมน์ 72**
  ทั้งหมด ไม่ใช่ข้อจำกัดของฟังก์ชันเอง — เมื่อตัดบรรทัดให้ถูกต้องแล้ว ทุกฟังก์ชันทำงานได้ตามที่มาตรฐาน
  ระบุไว้ นี่คือบทเรียนเชิงประจักษ์ที่ตอกย้ำกฎเหล็กของหลักสูตรนี้อีกครั้งหนึ่งอย่างชัดเจนที่สุด

### แบบฝึกหัดที่ 370.1

**โจทย์**: จงขยายโปรแกรม `STEP370-DATA-VALIDATOR` ให้แสดง Standard Deviation ของยอดเงิน
ทั้งสามรายการด้วย (นอกเหนือจาก Total, Average, Highest ที่มีอยู่แล้ว)

**เฉลย**:

```cobol
       01  WS-STDDEV           PIC S9(8)V999.
       ...
           COMPUTE WS-STDDEV =
               FUNCTION STANDARD-DEVIATION(WS-AMT(1) WS-AMT(2)
                   WS-AMT(3)).
           DISPLAY "STANDARD DEVIATION: " WS-STDDEV.
```

เพิ่มการประกาศตัวแปรและโค้ดคำนวณนี้ต่อจากส่วนคำนวณ `WS-HIGHEST` เดิม สังเกตว่าบรรทัด
`COMPUTE` ต้องแยกเป็น 2-3 บรรทัดเพื่อรักษากฎคอลัมน์ 72 ตามที่เรียนมาตลอด Part นี้

---

## สรุปท้ายบท

Part นี้พาเราสำรวจ Intrinsic Functions กลุ่มข้อความและสถิติอย่างละเอียด พร้อมทดสอบจริงทุกฟังก์ชัน
กับ GnuCOBOL ได้แก่:

- `FUNCTION UPPER-CASE`/`LOWER-CASE` สำหรับแปลงตัวพิมพ์และเปรียบเทียบแบบไม่สนตัวพิมพ์
- `FUNCTION TRIM` พร้อมตัวเลือก `LEADING`/`TRAILING` สำหรับตัดช่องว่างหัวท้าย
- `FUNCTION LENGTH` และข้อควรระวังเรื่องความยาว "ที่ประกาศไว้" กับ "เนื้อหาจริง"
- `FUNCTION NUMVAL`/`NUMVAL-C` สำหรับแปลงข้อความเป็นตัวเลข และอันตรายของการคืนค่า 0 เงียบ ๆ
- `FUNCTION SUM` สำหรับรวมยอดโดยไม่ต้องวนลูปเอง
- ฟังก์ชันสถิติ `MEAN`, `MEDIAN`, `VARIANCE`, `STANDARD-DEVIATION`, `RANGE` ที่ยืนยันแล้วว่า
  รองรับทั้งหมดใน GnuCOBOL
- `FUNCTION ORD`, `ORD-MAX`, `ORD-MIN` และความแตกต่างสำคัญระหว่าง "ตำแหน่ง" กับ "ค่า"
- `FUNCTION REVERSE` และ `FUNCTION CONCATENATE` พร้อมบทเรียนสำคัญที่สุดของ Part นี้: ปัญหา
  ที่ดูเหมือนข้อจำกัดของฟังก์ชัน แท้จริงคือกฎคอลัมน์ 72 ที่ต้องระวังเสมอ
- การผสมผสานฟังก์ชันข้อความเพื่อทำความสะอาดข้อมูลจริง และโปรแกรมตรวจสอบ/แปลงข้อมูลแบบเต็ม
  รูปแบบที่รวมทุกเทคนิคของ Part 036-037 เข้าด้วยกัน

เราได้เรียนรู้ Intrinsic Functions ครบทั้งสี่กลุ่มหลักแล้ว (ตัวเลข, วันที่, ข้อความ, สถิติ) ซึ่งเป็น
เครื่องมือที่ทำให้โค้ด COBOL สมัยใหม่กระชับ แม่นยำ และมีบั๊กน้อยลงอย่างมากเมื่อเทียบกับการเขียน
ตรรกะคำนวณเองทั้งหมด ใน **Part 038** เราจะเปลี่ยนไปสำรวจเครื่องมือสร้างรายงานที่ทรงพลังของ
COBOL: **Report Writer Feature** ซึ่งช่วยให้เราสร้างรายงานที่มีหัวกระดาษ ท้ายกระดาษ และการจัดกลุ่ม
ข้อมูลอัตโนมัติ โดยเขียนโค้ดน้อยกว่าการควบคุมการพิมพ์รายงานด้วยตัวเองทั้งหมด

**[กลับไปยัง Part 036: Intrinsic Functions ตัวเลขและวันที่ →](part-036-intrinsic-functions-numeric-date.md)**

**[ไปยัง Part 038: Report Writer Feature เบื้องต้น →](part-038-report-writer-basics.md)**
