# Part 043: Dynamic CALL และ Program Pointers (ขั้นตอนที่ 421–430)

## คำนำของ Part นี้

ใน Part 031 เราเรียนรู้ `CALL "PROGRAM-NAME"` แบบพื้นฐาน ซึ่งชื่อโปรแกรมที่จะเรียกถูกกำหนดตายตัวเป็น
literal ตั้งแต่ตอนเขียนโค้ด (เรียกว่า **Static CALL**) วิธีนี้ใช้งานได้ดีในกรณีทั่วไป แต่มีข้อจำกัดสำคัญ:
โปรแกรมหลักต้อง "รู้" ชื่อ subprogram ทุกตัวที่อาจถูกเรียกตั้งแต่ตอน compile ทำให้ระบบขยายตัวยากเมื่อ
ต้องการเพิ่ม subprogram ใหม่โดยไม่แก้โค้ดโปรแกรมหลัก

Part นี้จะพาคุณไปรู้จักกับ **Dynamic CALL** — การเรียกโปรแกรมย่อยโดยระบุชื่อผ่าน**ตัวแปร** แทนที่จะเป็น
literal ตายตัว ทำให้โปรแกรมสามารถ "ตัดสินใจตอนรันไทม์" ว่าจะเรียกโปรแกรมไหน ซึ่งเป็นรากฐานสำคัญของ
สถาปัตยกรรมแบบปลั๊กอิน (Plugin Architecture), Dispatch Table, และรูปแบบการเขียนโปรแกรมเชิงกลยุทธ์
(Strategy Pattern) ที่ใช้กันจริงในระบบองค์กรขนาดใหญ่ นอกจากนี้เราจะเรียนรู้ **`PROGRAM-POINTER`**
ชนิดข้อมูลพิเศษที่เก็บ "ที่อยู่ของโปรแกรม" ไว้เรียกซ้ำได้อย่างมีประสิทธิภาพ, คำสั่ง `SET ... TO ENTRY`,
และ `CANCEL` สำหรับควบคุมวงจรชีวิตของโปรแกรมย่อยที่โหลดเข้ามาแบบไดนามิก

> **ย้ำกฎเหล็กของหลักสูตร**: โค้ด COBOL ทุกตัวอย่างในเอกสารนี้เป็นภาษาอังกฤษล้วน และผ่านการคอมไพล์
> และรันจริงด้วย GnuCOBOL (`cobc (GnuCOBOL) 4.0-early-dev.0`) แล้วทุกตัวอย่าง รวมถึงกรณีที่ตั้งใจ
> แสดงพฤติกรรมของ error/exception ด้วย

---

## ขั้นตอนที่ 421: ทบทวน Static CALL และข้อจำกัด แนะนำ Dynamic CALL

### ปัญหาของ Static CALL

เมื่อเราเขียน `CALL "SUB-GREET" USING WS-PERSON` ชื่อ `"SUB-GREET"` เป็น **literal** ตัวคอมไพเลอร์
จะผูก (bind) การเรียกนี้กับโปรแกรมชื่อ `SUB-GREET` ตั้งแต่ตอน compile/link เรียกว่า **Static CALL**
วิธีนี้เร็วและปลอดภัย เพราะคอมไพเลอร์ตรวจสอบชื่อโปรแกรมได้ล่วงหน้า แต่มีข้อจำกัดคือ **โปรแกรมที่จะ
ถูกเรียกต้องรู้จักกันตายตัวตั้งแต่ตอนเขียนโค้ด** หากต้องการเปลี่ยนว่าจะเรียกโปรแกรมไหนตามเงื่อนไข
รันไทม์ (เช่น เลือกวิธีคำนวณภาษีตามประเทศที่ลูกค้าอยู่ หรือเลือกรูปแบบรายงานตามการตั้งค่าผู้ใช้) เราไม่
สามารถเปลี่ยน literal ได้โดยไม่ compile โค้ดใหม่

### Dynamic CALL คือทางแก้

**Dynamic CALL** คือการเขียน `CALL identifier` โดย `identifier` เป็น**ตัวแปร** (โดยทั่วไป `PIC X(n)`)
ที่เก็บชื่อโปรแกรมเป็นข้อความ รันไทม์จะอ่านค่าปัจจุบันของตัวแปรนั้นแล้วค้นหาและโหลดโปรแกรมที่มีชื่อ
ตรงกันมาเรียกใช้งาน **ทำให้เราสามารถเปลี่ยนโปรแกรมที่จะถูกเรียกได้ระหว่างการทำงานจริง โดยไม่ต้อง
แก้ไข/compile โค้ดโปรแกรมหลักใหม่เลย**

### ตัวอย่าง: Static CALL เทียบกับ Dynamic CALL

โปรแกรมย่อยสองตัวที่จะใช้ตลอด Part นี้:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SUB-GREET.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-NAME PIC X(20).
       PROCEDURE DIVISION USING LS-NAME.
           DISPLAY "Hello from SUB-GREET, " LS-NAME
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SUB-FAREWELL.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-NAME PIC X(20).
       PROCEDURE DIVISION USING LS-NAME.
           DISPLAY "Goodbye from SUB-FAREWELL, " LS-NAME
           GOBACK.
```

โปรแกรมหลักที่แสดงทั้งสองแบบ:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MAIN1.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PROG-NAME    PIC X(20) VALUE "SUB-GREET".
       01  WS-PERSON       PIC X(20) VALUE "SOMCHAI".
       PROCEDURE DIVISION.
           DISPLAY "Static call:"
           CALL "SUB-GREET" USING WS-PERSON
           DISPLAY "Dynamic call by identifier:"
           CALL WS-PROG-NAME USING WS-PERSON
           MOVE "SUB-FAREWELL" TO WS-PROG-NAME
           CALL WS-PROG-NAME USING WS-PERSON
           STOP RUN.
```

คอมไพล์ (ต้องรวมไฟล์ source ทุกไฟล์ที่เกี่ยวข้องเข้าด้วยกัน):

```
cobc -x -o main1 main1.cob sub-greet.cob sub-farewell.cob
```

**ผลลัพธ์:**

```
Static call:
Hello from SUB-GREET, SOMCHAI
Dynamic call by identifier:
Hello from SUB-GREET, SOMCHAI
Goodbye from SUB-FAREWELL, SOMCHAI
```

### อธิบายจุดสำคัญ

- `CALL "SUB-GREET" USING WS-PERSON`: static call — ชื่อโปรแกรมเป็น literal อยู่ในเครื่องหมายคำพูด
- `CALL WS-PROG-NAME USING WS-PERSON`: dynamic call — `WS-PROG-NAME` เป็นตัวแปร ไม่มีเครื่องหมาย
  คำพูดล้อมรอบ รันไทม์จะไปดูค่าปัจจุบันของ `WS-PROG-NAME` ("SUB-GREET" ในครั้งแรก) แล้วเรียกโปรแกรม
  ที่ชื่อตรงกัน
- เมื่อเรา `MOVE "SUB-FAREWELL" TO WS-PROG-NAME` แล้วเรียก `CALL WS-PROG-NAME` อีกครั้ง โปรแกรมที่ถูก
  เรียกจะเปลี่ยนเป็น `SUB-FAREWELL` ทันที **โดยไม่ต้องแก้โค้ดหรือ compile ใหม่เลย** — นี่คือหัวใจของ
  Dynamic CALL

### ข้อควรระวัง

- ค่าที่เก็บใน identifier ต้อง**ตรงกับชื่อ `PROGRAM-ID`** ของโปรแกรมย่อยแบบตัวพิมพ์ใหญ่-เล็กตามที่
  GnuCOBOL กำหนด (โดยทั่วไปจะไม่สนตัวพิมพ์ใหญ่เล็ก แต่ควรเขียนให้ตรงกันเพื่อความชัดเจน) และต้องไม่มี
  ช่องว่างเกินความจำเป็นปนอยู่ (แนะนำใช้ `FUNCTION TRIM` หรือกำหนดความกว้างตัวแปรให้พอดี)
- ต้อง compile และ link โปรแกรมย่อยทุกตัวที่อาจถูกเรียกแบบ dynamic เข้ากับโปรแกรมหลัก (หรือมีไฟล์
  โปรแกรมที่ compile แยกไว้ในพาธที่ runtime หาเจอ) ไม่เช่นนั้นจะเกิด error ตอนรัน (จะสอนวิธีดักจับใน
  ขั้นตอนที่ 426)

### แบบฝึกหัดที่ 421.1

**โจทย์**: จงอธิบายว่าทำไมข้อความ "Hello from SUB-GREET, SOMCHAI" จึงถูกแสดงถึง 2 ครั้งในผลลัพธ์
ของโปรแกรม `MAIN1` ทั้งที่มีคำสั่ง `CALL` อยู่ทั้งหมด 3 ครั้ง

**เฉลย**: `CALL` ครั้งที่ 1 เป็น static call เรียก `SUB-GREET` ตรง ๆ ผ่าน literal ส่วน `CALL` ครั้งที่ 2
เป็น dynamic call ผ่าน `WS-PROG-NAME` ซึ่ง ณ จุดนั้นยังมีค่าเป็น `"SUB-GREET"` อยู่ (ยังไม่ถูกเปลี่ยน)
จึงเรียกโปรแกรมเดียวกันซ้ำ ทำให้ข้อความ "Hello..." ปรากฏ 2 ครั้ง ส่วน `CALL` ครั้งที่ 3 เกิดขึ้นหลังจาก
`MOVE "SUB-FAREWELL" TO WS-PROG-NAME` แล้ว จึงเปลี่ยนไปเรียก `SUB-FAREWELL` แทน แสดงข้อความ
"Goodbye..." ออกมา

---

## ขั้นตอนที่ 422: กลไกเบื้องหลัง Dynamic CALL — Runtime Resolution

### Static CALL ผูกชื่อตอน Compile/Link, Dynamic CALL ค้นหาตอนรัน

ความแตกต่างเชิงเทคนิคที่สำคัญคือ **เวลาที่ระบบค้นหา (resolve) ตำแหน่งของโปรแกรมที่จะเรียก**:

| ประเภท | เวลาที่ resolve ชื่อโปรแกรม | ความเร็ว | ความยืดหยุ่น |
|---|---|---|---|
| Static CALL (`CALL "NAME"`) | ตอน compile/link | เร็วกว่า (ไม่ต้องค้นหาตอนรัน) | ต่ำ — เปลี่ยนไม่ได้ถ้าไม่ compile ใหม่ |
| Dynamic CALL (`CALL identifier`) | ตอนรัน (runtime) ทุกครั้งที่ถูกเรียก | ช้ากว่าเล็กน้อย (มีการค้นหาชื่อ) | สูง — เปลี่ยนพฤติกรรมได้โดยเปลี่ยนค่าตัวแปร |

ใน GnuCOBOL, dynamic call ที่ระบุชื่อโปรแกรมด้วยตัวแปรความยาวคงที่ (fixed-length) จะถูก runtime
library (`libcob`) ค้นหาโปรแกรมที่ตรงกับชื่อนั้นจาก**โปรแกรมที่ compile และ link มาพร้อมกัน**ก่อน
(เหมือนตัวอย่างที่ผ่านมา) หรือจากไฟล์ที่ compile แยกเป็น dynamic module (`.so` บน Linux) ที่วางอยู่ใน
พาธที่กำหนดผ่านตัวแปรสภาพแวดล้อม `COB_LIBRARY_PATH`

### ตัวอย่าง: เลือกโปรแกรมตามเงื่อนไขทางธุรกิจ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MAIN422.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUSTOMER-TYPE    PIC X(10) VALUE "VIP".
       01  WS-PROG-NAME        PIC X(20).
       01  WS-PERSON           PIC X(20) VALUE "MALEE".
       PROCEDURE DIVISION.
           IF WS-CUSTOMER-TYPE = "VIP"
               MOVE "SUB-GREET" TO WS-PROG-NAME
           ELSE
               MOVE "SUB-FAREWELL" TO WS-PROG-NAME
           END-IF
      *> The program to run is decided entirely at runtime,
      *> based on business data, not hard-coded at compile time.
           CALL WS-PROG-NAME USING WS-PERSON
           STOP RUN.
```

คอมไพล์และรัน:

```
cobc -x -o main422 main422.cob sub-greet.cob sub-farewell.cob
```

**ผลลัพธ์:**

```
Hello from SUB-GREET, MALEE
```

### อธิบายจุดสำคัญ

- `WS-PROG-NAME` ถูกกำหนดค่าจาก**ตรรกะทางธุรกิจ** (`IF WS-CUSTOMER-TYPE = "VIP"`) ไม่ใช่ค่าคงที่
  ที่เขียนตายตัว นี่คือรูปแบบการใช้งาน dynamic call ที่พบบ่อยที่สุดในระบบจริง
- โปรแกรมย่อยทั้งสองตัว (`SUB-GREET`, `SUB-FAREWELL`) ยังคงต้องถูก compile และ link เข้ากับโปรแกรม
  หลักเหมือนเดิม — dynamic call เปลี่ยน "ตอนไหนที่ตัดสินใจว่าจะเรียกตัวไหน" ไม่ได้ทำให้ไม่ต้อง
  compile โปรแกรมย่อยเลย

### ข้อควรระวัง

- อย่าสับสนระหว่าง "dynamic call" กับ "การโหลดโปรแกรมจากไฟล์ภายนอกที่ไม่เคย compile ไว้ก่อน" — ใน
  ตัวอย่างข้างต้น โปรแกรมย่อยทุกตัวยังคง**ถูก compile ไว้ล่วงหน้า**เสมอ dynamic call เพียงแค่เลื่อนการ
  ตัดสินใจ "จะเรียกตัวไหน" ไปไว้ที่ runtime เท่านั้น
- Dynamic call มี overhead จากการค้นหาชื่อโปรแกรมทุกครั้งที่เรียก แม้จะเล็กน้อยมากในกรณีทั่วไป แต่ถ้า
  เรียกในลูปที่หมุนหลายล้านรอบ ควรพิจารณาใช้ `PROGRAM-POINTER` (ขั้นตอนที่ 423-424) เพื่อ resolve
  เพียงครั้งเดียวแล้วเรียกซ้ำผ่าน pointer

### แบบฝึกหัดที่ 422.1

**โจทย์**: จงปรับโปรแกรม `MAIN422` ให้เลือกเรียก `SUB-GREET` เมื่อ `WS-CUSTOMER-TYPE` เป็น `"VIP"`
หรือ `"GOLD"` (สองค่า) และเรียก `SUB-FAREWELL` สำหรับกรณีอื่นทั้งหมด

**เฉลย**: เปลี่ยนเงื่อนไข `IF` เป็นการเปรียบเทียบ OR หรือใช้ 88-level:

```cobol
           IF WS-CUSTOMER-TYPE = "VIP" OR WS-CUSTOMER-TYPE = "GOLD"
               MOVE "SUB-GREET" TO WS-PROG-NAME
           ELSE
               MOVE "SUB-FAREWELL" TO WS-PROG-NAME
           END-IF
```

---

## ขั้นตอนที่ 423: PROGRAM-POINTER Data Item และ SET ... TO ENTRY

### ทำไมต้องมี PROGRAM-POINTER

`CALL identifier` (dynamic call ด้วยชื่อข้อความ) สะดวก แต่ทุกครั้งที่เรียก runtime ต้อง**ค้นหาชื่อ
โปรแกรมใหม่**จากค่าตัวแปร ถ้าเรารู้อยู่แล้วว่าจะเรียกโปรแกรมเดิมซ้ำ ๆ หลายครั้ง (เช่นในลูป) การค้นหาซ้ำ
ทุกรอบเป็นการเสียเวลาโดยไม่จำเป็น COBOL จึงมีชนิดข้อมูลพิเศษชื่อ **`PROGRAM-POINTER`** ที่เก็บ
**ที่อยู่ (entry point address)** ของโปรแกรมไว้โดยตรง เมื่อ resolve แล้วครั้งเดียว การเรียกซ้ำผ่าน
pointer จะเร็วกว่าการค้นหาชื่อใหม่ทุกครั้ง

### คำสั่ง SET ... TO ENTRY

```cobol
       SET program-pointer-item TO ENTRY "program-name"
```

คำสั่งนี้จะค้นหาโปรแกรมชื่อที่ระบุ (literal หรือ identifier ก็ได้) แล้วเก็บที่อยู่ของมันไว้ใน
`PROGRAM-POINTER` ที่ระบุ

### ตัวอย่าง: ประกาศและใช้งาน PROGRAM-POINTER

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MAIN2.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PERSON       PIC X(20) VALUE "MALEE".
       01  WS-GREET-PTR    USAGE IS PROGRAM-POINTER.
       01  WS-BYE-PTR      USAGE IS PROGRAM-POINTER.
       PROCEDURE DIVISION.
           SET WS-GREET-PTR TO ENTRY "SUB-GREET"
           SET WS-BYE-PTR   TO ENTRY "SUB-FAREWELL"
           CALL WS-GREET-PTR USING WS-PERSON
           CALL WS-BYE-PTR   USING WS-PERSON
           STOP RUN.
```

**ผลลัพธ์:**

```
Hello from SUB-GREET, MALEE
Goodbye from SUB-FAREWELL, MALEE
```

### อธิบายจุดสำคัญ

- `01 WS-GREET-PTR USAGE IS PROGRAM-POINTER.`: ประกาศตัวแปรชนิด `PROGRAM-POINTER` — สังเกตว่าไม่มี
  `PICTURE` clause เพราะเป็นชนิดข้อมูลพิเศษที่เก็บที่อยู่หน่วยความจำ ไม่ใช่ตัวเลขหรือข้อความปกติ
- `SET WS-GREET-PTR TO ENTRY "SUB-GREET"`: ค้นหาโปรแกรม `SUB-GREET` แล้วเก็บที่อยู่เข้าจุดเริ่มต้น
  (entry point) ของมันไว้ใน `WS-GREET-PTR` — การค้นหานี้เกิดขึ้น**เพียงครั้งเดียว** ณ บรรทัดนี้
- `CALL WS-GREET-PTR USING WS-PERSON`: เรียกโปรแกรมผ่าน pointer โดยตรง ไม่ต้องค้นหาชื่อซ้ำอีก

### ข้อควรระวัง

- `PROGRAM-POINTER` ต้องถูก `SET` ให้มีค่าก่อนใช้งานเสมอ หากเรียก `CALL` ผ่าน pointer ที่ยังไม่เคย
  `SET` (มีค่า garbage หรือ NULL) โปรแกรมจะ crash หรือมีพฤติกรรมไม่แน่นอน
- `ENTRY "program-name"` ต้องเป็นชื่อโปรแกรมที่มีอยู่จริงและถูก link เข้ามาแล้ว มิฉะนั้นจะเกิด error
  ตอนรัน (เรียนวิธีดักจับใน ขั้นตอนที่ 426)

### แบบฝึกหัดที่ 423.1

**โจทย์**: จงอธิบายความแตกต่างระหว่าง `CALL "SUB-GREET" USING ...`, `CALL WS-PROG-NAME USING ...`
(โดย `WS-PROG-NAME` เป็น `PIC X(20)`), และ `CALL WS-GREET-PTR USING ...` (โดย `WS-GREET-PTR` เป็น
`PROGRAM-POINTER`) ทั้งสามรูปแบบ

**เฉลย**: แบบแรกเป็น static call — ชื่อโปรแกรมผูกตายตัวตอน compile แบบที่สองเป็น dynamic call ด้วย
ชื่อข้อความ — runtime ค้นหาชื่อโปรแกรมใหม่จากค่าตัวแปรทุกครั้งที่ถูกเรียก แบบที่สามเป็น dynamic call
ผ่าน pointer ที่ resolve ที่อยู่ไว้ล่วงหน้าแล้วด้วย `SET ... TO ENTRY` การเรียกซ้ำจึงเร็วกว่าแบบที่สอง
เพราะไม่ต้องค้นหาชื่อใหม่ทุกครั้ง

---

## ขั้นตอนที่ 424: เปรียบเทียบประสิทธิภาพและกรณีการใช้งานที่เหมาะสม

### เมื่อไหร่ควรใช้แบบไหน

| สถานการณ์ | รูปแบบที่แนะนำ |
|---|---|
| รู้ชื่อโปรแกรมตายตัวตั้งแต่ตอนเขียนโค้ด ไม่เปลี่ยนแปลง | Static CALL (`CALL "NAME"`) |
| ต้องเลือกโปรแกรมตามเงื่อนไขทางธุรกิจ เรียกไม่บ่อย | Dynamic CALL ด้วยชื่อ (`CALL identifier`) |
| ต้องเรียกโปรแกรมเดิมซ้ำ ๆ จำนวนมาก (เช่นในลูป) และรู้ชื่อโปรแกรมล่วงหน้า | `PROGRAM-POINTER` + `SET ... TO ENTRY` แล้วเรียกผ่าน pointer |
| ต้องส่งต่อ "ความสามารถในการเรียกกลับ" ให้โปรแกรมอื่น (callback) | `PROGRAM-POINTER` ส่งเป็นพารามิเตอร์ (ขั้นตอนที่ 428) |

### ตัวอย่าง: วัดผลเชิงแนวคิดด้วยการเรียกซ้ำในลูป

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MAIN424.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PERSON       PIC X(20) VALUE "NIDA".
       01  WS-PROG-NAME    PIC X(20) VALUE "SUB-GREET".
       01  WS-GREET-PTR    USAGE PROGRAM-POINTER.
       01  WS-I            PIC 9(1).
       PROCEDURE DIVISION.
           DISPLAY "=== Using name-based dynamic CALL 3 times ==="
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 3
      *> Each iteration re-resolves the program name.
               CALL WS-PROG-NAME USING WS-PERSON
           END-PERFORM

           DISPLAY "=== Using PROGRAM-POINTER, resolved once ==="
           SET WS-GREET-PTR TO ENTRY WS-PROG-NAME
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 3
      *> The pointer is already resolved; no name lookup here.
               CALL WS-GREET-PTR USING WS-PERSON
           END-PERFORM
           STOP RUN.
```

**ผลลัพธ์:**

```
=== Using name-based dynamic CALL 3 times ===
Hello from SUB-GREET, NIDA
Hello from SUB-GREET, NIDA
Hello from SUB-GREET, NIDA
=== Using PROGRAM-POINTER, resolved once ===
Hello from SUB-GREET, NIDA
Hello from SUB-GREET, NIDA
Hello from SUB-GREET, NIDA
```

### อธิบายจุดสำคัญ

- ผลลัพธ์ที่แสดงออกมาเหมือนกันทุกประการทั้งสองแบบ — ความแตกต่างอยู่ที่ **สิ่งที่เกิดขึ้นภายใน** ไม่ใช่
  ผลลัพธ์ที่มองเห็น: แบบแรกค้นหาชื่อโปรแกรมใหม่ทุกรอบลูป (3 ครั้ง) แบบที่สองค้นหาเพียงครั้งเดียวก่อน
  เข้าลูป แล้วเรียกผ่านที่อยู่ที่จำไว้แล้วอีก 3 ครั้ง
- `SET WS-GREET-PTR TO ENTRY WS-PROG-NAME`: `ENTRY` รับได้ทั้ง literal และ identifier — ในที่นี้ใช้
  ค่าจากตัวแปร `WS-PROG-NAME` ได้เช่นกัน

### ข้อควรระวัง

- ในโปรแกรมขนาดเล็กความแตกต่างด้านความเร็วนี้แทบไม่มีผลที่สังเกตได้ แต่ในระบบ Batch ที่ประมวลผล
  หลายล้าน record ต่อวัน การลด overhead การค้นหาชื่อโปรแกรมซ้ำ ๆ ในลูปสามารถส่งผลต่อประสิทธิภาพรวม
  ได้อย่างมีนัยสำคัญ — นี่คือเหตุผลเชิงวิศวกรรมที่แท้จริงเบื้องหลัง `PROGRAM-POINTER`

### แบบฝึกหัดที่ 424.1

**โจทย์**: จงอธิบายว่าทำไมการ `SET WS-GREET-PTR TO ENTRY WS-PROG-NAME` ควรอยู่**นอกลูป** ไม่ใช่ในลูป

**เฉลย**: เพราะจุดประสงค์หลักของ `PROGRAM-POINTER` คือการค้นหา (resolve) ที่อยู่ของโปรแกรมเพียงครั้ง
เดียวแล้วนำไปใช้ซ้ำ หากนำ `SET ... TO ENTRY` ไปไว้ในลูป จะเท่ากับค้นหาชื่อโปรแกรมใหม่ทุกรอบเหมือนกับ
การใช้ dynamic call ด้วยชื่อธรรมดา ทำให้เสียประโยชน์ด้านประสิทธิภาพที่ `PROGRAM-POINTER` ควรมอบให้
ไปโดยสิ้นเชิง

---

## ขั้นตอนที่ 425: CANCEL Statement — รีเซ็ตสถานะโปรแกรมย่อย

### ปัญหา: โปรแกรมย่อยที่ถูกเรียกซ้ำจะ "จำ" ค่าเดิมไว้

เมื่อโปรแกรมย่อยถูกเรียกครั้งแรก ค่าตัวแปรใน `WORKING-STORAGE SECTION` ของมันจะถูกกำหนดค่าเริ่มต้น
(`VALUE` clause หรือค่าว่าง) ตามปกติ แต่**ถ้าถูกเรียกซ้ำอีกครั้งโดยไม่มีการ `CANCEL`** ค่าตัวแปรใน
`WORKING-STORAGE` ของโปรแกรมย่อยจะยังคงเป็นค่าที่เหลือจากการเรียกครั้งก่อนหน้า (ไม่ถูกรีเซ็ตกลับไปที่
`VALUE` เริ่มต้น) พฤติกรรมนี้มีประโยชน์เมื่อเราต้องการให้ subprogram "จำสถานะ" ข้ามการเรียกแต่ละครั้ง
(เช่น ตัวนับที่ต้องสะสมค่า) แต่ก็อาจเป็นบั๊กที่ไม่คาดคิดถ้าเราคาดหวังให้โปรแกรมย่อยเริ่มต้นใหม่ทุกครั้ง

### CANCEL คือคำสั่งรีเซ็ต

```cobol
       CANCEL program-name-or-identifier
```

`CANCEL` จะ "คืน" โปรแกรมย่อยกลับสู่สถานะเริ่มต้น ครั้งต่อไปที่ถูกเรียก ค่าตัวแปรใน
`WORKING-STORAGE SECTION` ของมันจะถูกกำหนดค่าเริ่มต้นใหม่ตาม `VALUE` clause อีกครั้ง

### ตัวอย่าง: ตัวนับสะสมค่า และผลของ CANCEL

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SUBCTR.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COUNTER PIC 9(4) VALUE 0.
       LINKAGE SECTION.
       01  LS-OUT PIC 9(4).
       PROCEDURE DIVISION USING LS-OUT.
           ADD 1 TO WS-COUNTER
           MOVE WS-COUNTER TO LS-OUT
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MAINCANCEL.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-RESULT PIC 9(4).
       01  WS-PROG   PIC X(10) VALUE "SUBCTR".
       PROCEDURE DIVISION.
           CALL WS-PROG USING WS-RESULT
           DISPLAY "1st call, counter = " WS-RESULT
           CALL WS-PROG USING WS-RESULT
           DISPLAY "2nd call, counter = " WS-RESULT
           CANCEL WS-PROG
           CALL WS-PROG USING WS-RESULT
           DISPLAY "3rd call after CANCEL, counter = " WS-RESULT
           STOP RUN.
```

**ผลลัพธ์:**

```
1st call, counter = 0001
2nd call, counter = 0002
3rd call after CANCEL, counter = 0001
```

### อธิบายจุดสำคัญ

- การเรียกครั้งที่ 1 และ 2 ทำให้ `WS-COUNTER` ภายใน `SUBCTR` สะสมค่าเพิ่มขึ้นเรื่อย ๆ (0001, 0002)
  เพราะไม่มีการ `CANCEL` ระหว่างการเรียกทั้งสองครั้ง โปรแกรมย่อยจึง "จำ" ค่าตัวนับไว้ข้ามการเรียก
- หลัง `CANCEL WS-PROG` ตัวนับกลับไปเริ่มที่ 0001 ใหม่ในการเรียกครั้งที่ 3 — เพราะ `CANCEL` สั่งให้
  runtime ล้างสถานะภายในของโปรแกรมย่อยทิ้ง ครั้งต่อไปที่ `CALL` จึงเหมือนเป็นการเริ่มโปรแกรมนั้นใหม่
  ทั้งหมด (คล้ายกับการโหลดโปรแกรมเข้าหน่วยความจำครั้งแรก)

### ข้อควรระวัง

- `CANCEL` ใช้ได้กับโปรแกรมที่ถูกเรียกผ่าน dynamic call (ทั้งด้วยชื่อและด้วย `PROGRAM-POINTER`) เป็น
  หลัก — มาตรฐาน COBOL ไม่รับประกันผลลัพธ์ที่แน่นอนหากใช้ `CANCEL` กับโปรแกรมที่กำลังทำงานอยู่ (เช่น
  `CANCEL` ตัวเอง หรือ `CANCEL` โปรแกรมที่อยู่ใน call chain ปัจจุบัน) จึงควรหลีกเลี่ยง
- อย่าลืมว่า `CANCEL` มีผลเฉพาะกับ**สถานะข้อมูลภายใน** (`WORKING-STORAGE` ของ subprogram) ไม่ใช่การ
  ลบไฟล์โปรแกรมออกจากดิสก์หรือหน่วยความจำถาวรแต่อย่างใด

### แบบฝึกหัดที่ 425.1

**โจทย์**: หากต้องการให้ `SUBCTR` นับต่อเนื่องจากค่าก่อนหน้าเสมอ (ไม่ต้องการให้ใครมา `CANCEL` โดย
ไม่ตั้งใจ) ควรออกแบบโค้ดส่วนไหนเป็นพิเศษ

**เฉลย**: ควรเก็บค่าตัวนับไว้ในที่ที่ไม่ถูกกระทบจาก `CANCEL` เช่น เขียนค่าลงไฟล์ (persist สถานะ) แทน
การพึ่งพา `WORKING-STORAGE` ของ subprogram ล้วน ๆ หรือถ้าจำเป็นต้องพึ่ง in-memory state จริง ๆ ควร
มีเอกสารกำกับชัดเจนในโค้ดว่าห้ามเรียก `CANCEL` กับโปรแกรมนี้ และควบคุมจุดที่มีสิทธิ์เรียก `CANCEL`
ให้อยู่ในที่เดียวเท่านั้นเพื่อป้องกันความผิดพลาด

---

## ขั้นตอนที่ 426: ON EXCEPTION / NOT ON EXCEPTION — จัดการเมื่อหาโปรแกรมไม่เจอ

### ปัญหา: ถ้าเรียกโปรแกรมที่ไม่มีอยู่จริง

Dynamic CALL เปิดโอกาสให้ชื่อโปรแกรมมาจากตัวแปรที่อาจมีค่าผิดพลาด (พิมพ์ผิด, ข้อมูล configuration
เพี้ยน ฯลฯ) หากเรียก `CALL identifier` ด้วยชื่อที่ไม่มีโปรแกรมใดตรงกันเลย โปรแกรมจะ crash ทันทีถ้าไม่
มีการดักจับ COBOL จึงมี clause `ON EXCEPTION` ให้แนบไว้กับ `CALL` เพื่อดักจับกรณีนี้

### ตัวอย่าง: ดักจับ CALL ที่หาโปรแกรมไม่เจอ

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MAIN3B.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PERSON       PIC X(20) VALUE "NIDA".
       01  WS-PROG-NAME    PIC X(20) VALUE "NO-SUCH-PROGRAM".
       PROCEDURE DIVISION.
           CALL WS-PROG-NAME USING WS-PERSON
               ON EXCEPTION
                   DISPLAY "ERROR: cannot find program "
                       WS-PROG-NAME
           END-CALL
           DISPLAY "Program continues after failed dynamic CALL"
           STOP RUN.
```

**ผลลัพธ์:**

```
ERROR: cannot find program NO-SUCH-PROGRAM
Program continues after failed dynamic CALL
```

### อธิบายจุดสำคัญ

- `ON EXCEPTION` เป็น clause ที่แนบไว้ท้าย `CALL` — เมื่อ runtime หาโปรแกรมที่ระบุไม่เจอ (หรือเกิด
  ข้อผิดพลาดในการเรียกโปรแกรม) มันจะรันคำสั่งภายใน `ON EXCEPTION` แทนที่จะปล่อยให้โปรแกรม crash
- `END-CALL` คือ scope terminator ปิดท้าย `CALL` ที่มี clause พิเศษแนบมา (จำเป็นต้องมีเมื่อใช้
  `ON EXCEPTION`)
- หลังจาก `ON EXCEPTION` ทำงานเสร็จ โปรแกรมหลักจะทำงาน**ต่อไปตามปกติ** (ไม่ crash) ดังจะเห็นจาก
  บรรทัด "Program continues..." ที่ยังคงแสดงผลออกมา

### ⚠️ ข้อจำกัดที่ตรวจพบจริงในบิลด์นี้: NOT ON EXCEPTION ทำงานไม่ถูกต้อง

มาตรฐาน COBOL ยังอนุญาตให้แนบ clause `NOT ON EXCEPTION` คู่กับ `ON EXCEPTION` เพื่อระบุคำสั่งที่จะ
รันเมื่อ `CALL` **สำเร็จ** (ตรงข้ามกับ `ON EXCEPTION`) อย่างไรก็ตาม จากการทดสอบจริงกับ
`cobc (GnuCOBOL) 4.0-early-dev.0` พบว่า **`NOT ON EXCEPTION` ไม่ทำงานตามที่คาดหวัง**:

```cobol
      *> Tested and confirmed NOT reliable on this build:
           CALL WS-PROG-NAME USING WS-PERSON
               ON EXCEPTION
                   DISPLAY "ERROR: cannot find program"
               NOT ON EXCEPTION
                   DISPLAY "Call succeeded"
           END-CALL
```

เมื่อทดสอบกับโปรแกรมที่ **หาไม่เจอ** ผลลัพธ์ที่ได้กลับแสดงทั้งสองข้อความ (ทั้ง `ON EXCEPTION` และ
`NOT ON EXCEPTION` ทำงาน) และเมื่อทดสอบกับโปรแกรมที่**เรียกสำเร็จ** ข้อความจาก `NOT ON EXCEPTION`
กลับไม่ปรากฏเลย พฤติกรรมนี้ตรงข้ามและไม่สอดคล้องกันในทั้งสองกรณี แสดงว่า `NOT ON EXCEPTION` ของ
`CALL` มีบั๊กในบิลด์ early-dev นี้ **จึงไม่แนะนำให้ใช้ `NOT ON EXCEPTION` กับ `CALL` ในบิลด์นี้**

**วิธีแก้ที่ตรวจสอบแล้วว่าใช้งานได้จริง**: ใช้ `ON EXCEPTION` เพียงอย่างเดียวคู่กับ flag ของเราเอง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MAIN426.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PERSON        PIC X(20) VALUE "NIDA".
       01  WS-PROG-NAME     PIC X(20) VALUE "SUB-GREET".
       01  WS-CALL-FAILED   PIC X VALUE "N".
       PROCEDURE DIVISION.
           MOVE "N" TO WS-CALL-FAILED
           CALL WS-PROG-NAME USING WS-PERSON
               ON EXCEPTION
                   MOVE "Y" TO WS-CALL-FAILED
                   DISPLAY "ERROR: cannot find program "
                       WS-PROG-NAME
           END-CALL
           IF WS-CALL-FAILED = "N"
               DISPLAY "Call succeeded"
           END-IF
           STOP RUN.
```

**ผลลัพธ์ (เมื่อ SUB-GREET มีอยู่จริง):**

```
Hello from SUB-GREET, NIDA
Call succeeded
```

### ข้อควรระวัง

- ยึดหลัก **ใช้ `ON EXCEPTION` เพียงอย่างเดียว** แล้วใช้ตรรกะ `IF` ปกติเพื่อจัดการ "กรณีสำเร็จ" แทนการ
  พึ่งพา `NOT ON EXCEPTION` — เทคนิคนี้ยืนยันแล้วว่าให้ผลลัพธ์ถูกต้อง 100% ในบิลด์นี้
- ก่อนใช้ feature ใด ๆ ที่มีหลาย clause ร่วมกัน ควรเขียนโปรแกรมทดสอบเล็ก ๆ (เหมือนที่ทำในขั้นตอนนี้)
  เพื่อยืนยันพฤติกรรมจริงบนบิลด์คอมไพเลอร์ที่ใช้งานอยู่ ก่อนนำไปใช้ในระบบจริง — เป็นวินัยสำคัญของงาน
  Legacy/Enterprise ที่คอมไพเลอร์แต่ละเวอร์ชันอาจมีพฤติกรรมต่างกันเล็กน้อย

### แบบฝึกหัดที่ 426.1

**โจทย์**: จงเขียนโปรแกรมที่พยายามเรียก `"MISSING-PROGRAM"` แบบ dynamic call และแสดงข้อความ
"Using fallback program instead" แล้วเรียก `SUB-GREET` แทน หากพบว่าเรียกไม่สำเร็จ

**เฉลย**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. FALLBACK.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PERSON        PIC X(20) VALUE "BOONMEE".
       01  WS-PROG-NAME     PIC X(20) VALUE "MISSING-PROGRAM".
       01  WS-CALL-FAILED   PIC X VALUE "N".
       PROCEDURE DIVISION.
           CALL WS-PROG-NAME USING WS-PERSON
               ON EXCEPTION
                   MOVE "Y" TO WS-CALL-FAILED
           END-CALL
           IF WS-CALL-FAILED = "Y"
               DISPLAY "Using fallback program instead"
               CALL "SUB-GREET" USING WS-PERSON
           END-IF
           STOP RUN.
```

---

## ขั้นตอนที่ 427: รูปแบบ Dispatch Table — เลือก Subprogram จากตารางข้อมูล

### แนวคิด Table-Driven Dispatch

เมื่อจำนวนโปรแกรมย่อยที่ต้องเลือกเรียกมีมากขึ้น การเขียน `IF/ELSE IF` ยาว ๆ จะดูแลรักษายาก แนวทางที่
นิยมในระบบจริงคือการสร้าง **ตารางจับคู่** ระหว่าง "รหัสคำสั่ง" กับ "ชื่อโปรแกรม" แล้วค้นหาในตารางนั้น
แทน วิธีนี้ทำให้การเพิ่ม/แก้ไขคำสั่งใหม่ทำได้ง่ายเพียงแก้ที่ตารางข้อมูล โดยไม่ต้องแตะตรรกะหลัก

### ตัวอย่าง: Dispatch Table สำหรับเครื่องคิดเลขอย่างง่าย

โปรแกรมย่อยสำหรับคำนวณ:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CALC-ADD.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-A            PIC 9(5).
       01  LS-B            PIC 9(5).
       01  LS-RESULT       PIC 9(6).
       PROCEDURE DIVISION USING LS-A LS-B LS-RESULT.
           COMPUTE LS-RESULT = LS-A + LS-B
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CALC-MUL.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-A            PIC 9(5).
       01  LS-B            PIC 9(5).
       01  LS-RESULT       PIC 9(6).
       PROCEDURE DIVISION USING LS-A LS-B LS-RESULT.
           COMPUTE LS-RESULT = LS-A * LS-B
           GOBACK.
```

โปรแกรมหลักพร้อม dispatch table:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DISPATCH.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-OP-TABLE.
           05  FILLER PIC X(10) VALUE "ADD".
           05  FILLER PIC X(10) VALUE "CALC-ADD".
           05  FILLER PIC X(10) VALUE "MUL".
           05  FILLER PIC X(10) VALUE "CALC-MUL".
       01  WS-OP-ARRAY REDEFINES WS-OP-TABLE.
           05  WS-OP-ROW OCCURS 2 TIMES.
               10  WS-OP-CODE      PIC X(10).
               10  WS-OP-PROGRAM   PIC X(10).
       01  WS-REQUEST      PIC X(10) VALUE "MUL".
       01  WS-A            PIC 9(5) VALUE 6.
       01  WS-B            PIC 9(5) VALUE 7.
       01  WS-RESULT       PIC 9(6).
       01  WS-I            PIC 9(2).
       01  WS-PROGRAM      PIC X(10).
       PROCEDURE DIVISION.
           MOVE SPACES TO WS-PROGRAM
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 2
               IF WS-OP-CODE(WS-I) = WS-REQUEST
                   MOVE WS-OP-PROGRAM(WS-I) TO WS-PROGRAM
               END-IF
           END-PERFORM

           IF WS-PROGRAM = SPACES
               DISPLAY "Unknown operation: " WS-REQUEST
           ELSE
               CALL WS-PROGRAM USING WS-A WS-B WS-RESULT
               DISPLAY WS-REQUEST "(" WS-A ", " WS-B ") = "
                   WS-RESULT
           END-IF
           STOP RUN.
```

**ผลลัพธ์:**

```
MUL       (00006, 00007) = 000042
```

### อธิบายจุดสำคัญ

- `WS-OP-TABLE` เก็บข้อมูลดิบเป็นคู่ ๆ ("ADD"→"CALC-ADD", "MUL"→"CALC-MUL") แล้วใช้ `REDEFINES` เพื่อ
  มองข้อมูลก้อนเดียวกันเป็นตาราง `OCCURS 2 TIMES` ที่มี 2 คอลัมน์ (รหัสคำสั่ง, ชื่อโปรแกรม) — เทคนิค
  นี้เรียนไปแล้วใน Part 016 และ Part 022
- ลูป `PERFORM VARYING` ค้นหาแถวที่ `WS-OP-CODE` ตรงกับ `WS-REQUEST` แล้วดึงชื่อโปรแกรมที่ตรงกันออกมา
- `CALL WS-PROGRAM` เป็น dynamic call ตามชื่อที่ค้นเจอจากตาราง — เพิ่มคำสั่งใหม่ทำได้ง่ายเพียงเพิ่ม
  แถวใน `WS-OP-TABLE` โดยไม่ต้องแก้ตรรกะการค้นหาหรือการเรียกเลย

### ข้อควรระวัง

- ตัวอย่างนี้ใช้การค้นหาแบบ Linear Search ซึ่งเหมาะกับตารางขนาดเล็ก ถ้าตารางมีคำสั่งจำนวนมาก ควร
  พิจารณาใช้ `SEARCH ALL` กับตารางที่เรียงลำดับแล้ว (ทบทวนได้ใน Part 017) เพื่อประสิทธิภาพที่ดีกว่า
- ควรตรวจสอบกรณี "ไม่พบรหัสคำสั่ง" เสมอ (`WS-PROGRAM = SPACES` ในตัวอย่าง) เพื่อป้องกันการเรียก
  `CALL` ด้วยชื่อว่างเปล่าซึ่งจะทำให้เกิด error

### แบบฝึกหัดที่ 427.1

**โจทย์**: จงเพิ่มแถวใหม่ในตาราง `WS-OP-TABLE` สำหรับคำสั่ง `"SUB"` ที่เรียกโปรแกรม `"CALC-SUB"`
โดยไม่ต้องแก้ไขตรรกะการค้นหาหรือการเรียกใด ๆ

**เฉลย**: เพิ่ม `FILLER` สองบรรทัดในกลุ่มข้อมูล และเปลี่ยน `OCCURS 2 TIMES` เป็น `OCCURS 3 TIMES`
พร้อมปรับเงื่อนไข `UNTIL WS-I > 2` เป็น `UNTIL WS-I > 3`:

```cobol
       01  WS-OP-TABLE.
           05  FILLER PIC X(10) VALUE "ADD".
           05  FILLER PIC X(10) VALUE "CALC-ADD".
           05  FILLER PIC X(10) VALUE "MUL".
           05  FILLER PIC X(10) VALUE "CALC-MUL".
           05  FILLER PIC X(10) VALUE "SUB".
           05  FILLER PIC X(10) VALUE "CALC-SUB".
       01  WS-OP-ARRAY REDEFINES WS-OP-TABLE.
           05  WS-OP-ROW OCCURS 3 TIMES.
               10  WS-OP-CODE      PIC X(10).
               10  WS-OP-PROGRAM   PIC X(10).
```

---

## ขั้นตอนที่ 428: ส่ง PROGRAM-POINTER เป็นพารามิเตอร์ — รูปแบบ Callback

### แนวคิด Callback ใน COBOL

หนึ่งในความสามารถที่ทรงพลังที่สุดของ `PROGRAM-POINTER` คือการ**ส่งมันเป็นพารามิเตอร์**ให้โปรแกรมอื่น
เพื่อให้โปรแกรมนั้นเรียกกลับมาทีหลัง (เรียกว่า **Callback** ในภาษาโปรแกรมสมัยใหม่หลายภาษา) รูปแบบนี้
ทำให้เราเขียนโปรแกรม "แม่แบบ" (template) ที่ทำงานร่วมกับ logic ที่หลากหลายได้ โดยไม่ต้องรู้ล่วงหน้าว่า
logic นั้นคืออะไร

### ตัวอย่าง: โปรแกรม INVOKER รับ Callback มาเรียกใช้

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. INVOKER.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-CALLBACK     USAGE PROGRAM-POINTER.
       01  LS-VALUE        PIC 9(5).
       PROCEDURE DIVISION USING LS-CALLBACK LS-VALUE.
           DISPLAY "INVOKER: about to call back with value "
               LS-VALUE
           CALL LS-CALLBACK USING LS-VALUE
           DISPLAY "INVOKER: callback finished"
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. DOUBLER.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-VALUE        PIC 9(5).
       PROCEDURE DIVISION USING LS-VALUE.
           COMPUTE LS-VALUE = LS-VALUE * 2
           DISPLAY "DOUBLER: new value = " LS-VALUE
           GOBACK.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MAINCB.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DOUBLER-PTR  USAGE PROGRAM-POINTER.
       01  WS-NUMBER       PIC 9(5) VALUE 21.
       PROCEDURE DIVISION.
           SET WS-DOUBLER-PTR TO ENTRY "DOUBLER"
           CALL "INVOKER" USING WS-DOUBLER-PTR WS-NUMBER
           DISPLAY "MAIN: final value = " WS-NUMBER
           STOP RUN.
```

**ผลลัพธ์:**

```
INVOKER: about to call back with value 00021
DOUBLER: new value = 00042
INVOKER: callback finished
MAIN: final value = 00042
```

### อธิบายจุดสำคัญ

- `MAINCB` ไม่ได้เรียก `DOUBLER` ตรง ๆ — มันเรียก `INVOKER` โดย**ส่ง pointer ของ `DOUBLER`** ไปเป็น
  พารามิเตอร์ตัวแรก (`LS-CALLBACK` ใน `INVOKER`)
- `INVOKER` ไม่รู้จัก `DOUBLER` เลยตอน compile — มันแค่รู้ว่าจะได้รับ `PROGRAM-POINTER` มาตัวหนึ่งแล้ว
  `CALL LS-CALLBACK USING LS-VALUE` เพื่อเรียกโปรแกรมที่ pointer นั้นชี้ไป
- นี่คือ **Strategy Pattern** แบบ COBOL: `INVOKER` เป็นโปรแกรม "แม่แบบ" ที่ไม่ผูกติดกับ logic เฉพาะ
  ใด ๆ สามารถส่ง pointer ของโปรแกรมอื่น (เช่น `TRIPLER`, `SQUARER`) เข้าไปแทนที่ `DOUBLER` ได้โดยไม่
  ต้องแก้ไขโค้ด `INVOKER` เลยแม้แต่บรรทัดเดียว
- `WS-NUMBER` เปลี่ยนค่าจาก 21 เป็น 42 ได้เพราะส่งผ่าน `USING` แบบ `BY REFERENCE` (ค่าเริ่มต้นของ
  COBOL ทบทวนได้ใน Part 032) ทำให้การแก้ไขค่าใน `DOUBLER` สะท้อนกลับมาถึง `MAINCB` ได้

### ข้อควรระวัง

- ผู้เขียนโปรแกรม "แม่แบบ" (เช่น `INVOKER`) ต้องกำหนดสัญญา (contract) ที่ชัดเจนว่าโปรแกรม callback
  ต้องรับพารามิเตอร์แบบไหน (จำนวน, ชนิด, ลำดับ) เพราะ COBOL ไม่มีการตรวจสอบชนิดพารามิเตอร์ข้าม
  โปรแกรมที่เข้มงวดเหมือนภาษาที่มี type system แน่นหนา หากโปรแกรม callback รับพารามิเตอร์ไม่ตรงกับ
  ที่ผู้เรียกส่งมา อาจเกิดพฤติกรรมที่ไม่คาดคิดโดยไม่มี error แจ้งเตือนชัดเจน
- ควร `SET` pointer ให้มีค่าที่ถูกต้องก่อนส่งเป็นพารามิเตอร์เสมอ (ทบทวนข้อควรระวังจากขั้นตอนที่ 423)

### แบบฝึกหัดที่ 428.1

**โจทย์**: จงเขียนโปรแกรม `TRIPLER` ที่คูณค่าด้วย 3 แล้วปรับ `MAINCB` ให้เรียก `INVOKER` ด้วย
`TRIPLER` แทน `DOUBLER` โดยไม่แก้ไขโค้ด `INVOKER` เลย

**เฉลย**:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. TRIPLER.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-VALUE        PIC 9(5).
       PROCEDURE DIVISION USING LS-VALUE.
           COMPUTE LS-VALUE = LS-VALUE * 3
           DISPLAY "TRIPLER: new value = " LS-VALUE
           GOBACK.
```

แล้วแก้ `MAINCB` เพียงบรรทัดเดียว: `SET WS-DOUBLER-PTR TO ENTRY "TRIPLER"` (หรือเปลี่ยนชื่อตัวแปรให้
สื่อความหมายเป็น `WS-CALLBACK-PTR` เพื่อความชัดเจน) — `INVOKER` ทำงานได้ทันทีโดยไม่ต้องแก้ไขอะไรเลย
เพราะมันไม่เคยรู้จัก `DOUBLER` หรือ `TRIPLER` เจาะจงตั้งแต่แรก

---

## ขั้นตอนที่ 429: ข้อควรระวังและข้อจำกัดของ Dynamic CALL โดยรวม

### สรุปข้อควรระวังสำคัญที่ต้องจำ

1. **ชื่อโปรแกรมต้องตรงกันเป๊ะ**: ค่าใน identifier ที่ใช้ dynamic call ต้องตรงกับ `PROGRAM-ID` ของ
   โปรแกรมย่อยจริง ๆ รวมช่องว่างท้ายข้อความที่อาจก่อปัญหาถ้าความกว้างตัวแปรไม่พอดี ควรใช้
   `FUNCTION TRIM` หรือกำหนดความกว้างให้ตรงเสมอ
2. **NOT ON EXCEPTION ไม่น่าเชื่อถือในบิลด์นี้**: ตามที่พิสูจน์แล้วในขั้นตอนที่ 426 ให้ใช้
   `ON EXCEPTION` ร่วมกับ flag ของเราเองแทน
3. **PROGRAM-POINTER ต้อง SET ก่อนใช้เสมอ**: การเรียก `CALL` ผ่าน pointer ที่ไม่เคยถูก `SET` (หรือ
   ถูก `SET TO NULL`) จะทำให้โปรแกรม crash
4. **CANCEL มีผลต่อสถานะภายในเท่านั้น**: ไม่ใช่การลบโปรแกรมออกจากระบบถาวร และควรใช้ด้วยความระมัดระวัง
   กับโปรแกรมที่อยู่ระหว่างการทำงาน
5. **Dynamic call มี overhead เล็กน้อยกว่า static call**: ในงานที่ต้องเรียกซ้ำจำนวนมาก ควรพิจารณาใช้
   `PROGRAM-POINTER` เพื่อลด overhead การค้นหาชื่อซ้ำ
6. **ต้อง compile/link โปรแกรมย่อยทุกตัวที่อาจถูกเรียกไว้ล่วงหน้าเสมอ**: dynamic call ไม่ได้แปลว่า
   ไม่ต้อง compile subprogram — มันเปลี่ยนแค่ "เวลาที่ตัดสินใจว่าจะเรียกตัวไหน" เท่านั้น

### ตัวอย่าง: ตรวจสอบผลกระทบของช่องว่างท้ายชื่อโปรแกรม

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MAIN429.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PERSON        PIC X(20) VALUE "ANONG".
       01  WS-PROG-WIDE     PIC X(30) VALUE "SUB-GREET".
       01  WS-CALL-FAILED   PIC X VALUE "N".
       PROCEDURE DIVISION.
      *> WS-PROG-WIDE is PIC X(30) but only holds "SUB-GREET"
      *> (9 characters) padded with 21 trailing spaces. GnuCOBOL
      *> correctly trims trailing spaces for program name
      *> resolution, so this still works as expected.
           CALL WS-PROG-WIDE USING WS-PERSON
               ON EXCEPTION
                   MOVE "Y" TO WS-CALL-FAILED
           END-CALL
           IF WS-CALL-FAILED = "Y"
               DISPLAY "Call failed unexpectedly"
           ELSE
               DISPLAY "Call succeeded despite trailing spaces"
           END-IF
           STOP RUN.
```

**ผลลัพธ์:**

```
Hello from SUB-GREET, ANONG
Call succeeded despite trailing spaces
```

### อธิบายจุดสำคัญ

- GnuCOBOL จัดการช่องว่างท้ายข้อความ (trailing spaces) ในชื่อโปรแกรมให้อัตโนมัติเมื่อค้นหาโปรแกรม
  ดังนั้นการที่ตัวแปรมีความกว้างมากกว่าชื่อโปรแกรมจริง (เช่น `PIC X(30)` เก็บชื่อยาว 9 ตัวอักษร) จะไม่
  เป็นปัญหา อย่างไรก็ตาม **ช่องว่างตรงกลางหรือหน้าข้อความ** (leading/embedded spaces) จากข้อมูลป้อน
  เข้าที่ไม่สะอาดยังคงเป็นสาเหตุที่พบบ่อยของบั๊กจริงในระบบ Production ควรตรวจสอบข้อมูลก่อนใช้เสมอ

### ข้อควรระวัง

- อย่าพึ่งพาพฤติกรรม "trim ให้อัตโนมัติ" ของ compiler เป็นข้อสมมติฐานหลักในการออกแบบระบบ เพราะ
  compiler แต่ละยี่ห้อ/เวอร์ชันอาจมีรายละเอียดต่างกัน ควร `FUNCTION TRIM` หรือควบคุมข้อมูลให้สะอาด
  ตั้งแต่ต้นทางเสมอเพื่อความชัดเจนและพกพาข้ามคอมไพเลอร์ได้ง่ายกว่า

### แบบฝึกหัดที่ 429.1

**โจทย์**: จงเขียนรายการ (checklist) 3 ข้อที่ควรตรวจสอบก่อนนำโค้ดที่ใช้ dynamic CALL ขึ้นใช้งานจริง
(production)

**เฉลย** (ตัวอย่างคำตอบที่ดี):
1. ตรวจสอบว่าชื่อโปรแกรมทุกตัวที่อาจถูกเรียกแบบ dynamic ถูก compile และ link/deploy ไว้ครบถ้วนแล้ว
2. มีการดักจับ `ON EXCEPTION` ครอบคลุมทุกจุดที่ทำ dynamic call เพื่อป้องกันโปรแกรม crash กรณีหา
   โปรแกรมไม่เจอ
3. ข้อมูลที่นำมาใช้เป็นชื่อโปรแกรม (โดยเฉพาะจากภายนอก เช่น ไฟล์ configuration หรือฐานข้อมูล) ผ่าน
   การตรวจสอบความสะอาด (validate/trim) ก่อนนำไปใช้ใน `CALL` เสมอ

---

## ขั้นตอนที่ 430: ตัวอย่างรวม — ระบบประมวลผลคำสั่งแบบปลั๊กอิน (Plugin-Style Command Processor)

### ภาพรวมโปรแกรมสุดท้ายของ Part นี้

เราจะรวมทุกเทคนิคที่เรียนมาใน Part นี้เข้าด้วยกัน: **Dispatch Table** (ขั้นตอนที่ 427) +
**Dynamic CALL** (ขั้นตอนที่ 421-422) + **การจัดการ Exception ที่ถูกต้อง** (ขั้นตอนที่ 426) เพื่อสร้าง
ระบบประมวลผลรายการคำสั่งที่รองรับการเพิ่มคำสั่งใหม่ได้ง่าย และไม่ล่มแม้เจอคำสั่งที่ไม่รู้จัก

### โปรแกรมย่อยที่ใช้ (เพิ่มเติมจากขั้นตอนที่ 427)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CALC-SUB.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-A            PIC 9(5).
       01  LS-B            PIC 9(5).
       01  LS-RESULT       PIC 9(6).
       PROCEDURE DIVISION USING LS-A LS-B LS-RESULT.
           COMPUTE LS-RESULT = LS-A - LS-B
           GOBACK.
```

### โปรแกรมหลัก: FINAL430

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. FINAL430.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-REQUESTS.
           05  FILLER PIC X(10) VALUE "ADD".
           05  FILLER PIC X(10) VALUE "MUL".
           05  FILLER PIC X(10) VALUE "DIV".
           05  FILLER PIC X(10) VALUE "SUB".
       01  WS-REQ-TABLE REDEFINES WS-REQUESTS.
           05  WS-REQ-CODE OCCURS 4 TIMES PIC X(10).
       01  WS-PROG-NAME    PIC X(10).
       01  WS-A            PIC 9(5) VALUE 20.
       01  WS-B            PIC 9(5) VALUE 5.
       01  WS-RESULT       PIC 9(6).
       01  WS-CALL-FAILED  PIC X VALUE "N".
       01  WS-I            PIC 9(2).
       PROCEDURE DIVISION.
           PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 4
      *> Step 1: look up the program name for this request code.
      *> Adding a new operation only means adding one WHEN branch
      *> here and one more row in WS-REQUESTS - the CALL and
      *> error-handling logic below never changes.
               EVALUATE WS-REQ-CODE(WS-I)
                   WHEN "ADD"
                       MOVE "CALC-ADD" TO WS-PROG-NAME
                   WHEN "SUB"
                       MOVE "CALC-SUB" TO WS-PROG-NAME
                   WHEN "MUL"
                       MOVE "CALC-MUL" TO WS-PROG-NAME
                   WHEN OTHER
                       MOVE SPACES TO WS-PROG-NAME
               END-EVALUATE

               IF WS-PROG-NAME = SPACES
                   DISPLAY "Skip unsupported operation: "
                       WS-REQ-CODE(WS-I)
               ELSE
                   MOVE "N" TO WS-CALL-FAILED
      *> Step 2: dynamic CALL with proper exception handling
      *> (ON EXCEPTION only - see step 426 for why).
                   CALL WS-PROG-NAME USING WS-A WS-B WS-RESULT
                       ON EXCEPTION
                           MOVE "Y" TO WS-CALL-FAILED
                           DISPLAY "Program not found: "
                               WS-PROG-NAME
                   END-CALL
                   IF WS-CALL-FAILED = "N"
                       DISPLAY WS-REQ-CODE(WS-I) ": "
                           WS-A " , " WS-B " -> " WS-RESULT
                   END-IF
               END-IF
           END-PERFORM
           STOP RUN.
```

คอมไพล์:

```
cobc -x -o final430 final430.cob subcalc-add.cob subcalc-mul.cob subcalc-sub.cob
```

**ผลลัพธ์:**

```
ADD       : 00020 , 00005 -> 000025
MUL       : 00020 , 00005 -> 000100
Skip unsupported operation: DIV
SUB       : 00020 , 00005 -> 000015
```

### อธิบายจุดสำคัญ

- คำขอ 4 รายการถูกประมวลผลตามลำดับ: `ADD`, `MUL`, `DIV`, `SUB` — เนื่องจากไม่มีโปรแกรม `CALC-DIV`
  compile ไว้ (ตั้งใจไม่สร้างเพื่อจำลองสถานการณ์คำสั่งที่ยังไม่รองรับ) ตรรกะ `EVALUATE` จึงตรวจพบว่า
  ไม่รู้จักคำสั่งนี้ (`WS-PROG-NAME = SPACES`) และข้ามไปอย่างปลอดภัยโดยไม่พยายามเรียก `CALL` เลย
- คำสั่งที่เหลือ (`ADD`, `MUL`, `SUB`) ถูกเรียกผ่าน dynamic CALL ตามชื่อโปรแกรมที่แปลงมาจากรหัสคำสั่ง
  และแสดงผลลัพธ์การคำนวณได้ถูกต้อง
- โครงสร้างนี้แสดงให้เห็นว่าการรวม **Dispatch Table + Dynamic CALL + Exception Handling ที่ถูกต้อง**
  ทำให้ได้ระบบที่ขยายง่าย (เพิ่มคำสั่งใหม่โดยแก้เพียงจุดเดียว) และทนทานต่อข้อผิดพลาด (ไม่ล่มแม้เจอ
  คำสั่งหรือโปรแกรมที่ไม่มีอยู่จริง) ซึ่งเป็นรูปแบบที่พบได้จริงในระบบประมวลผลธุรกรรมขนาดใหญ่จำนวนมาก

### ข้อควรระวัง

- ตัวอย่างนี้เพื่อการศึกษาใช้ literal ตายตัวสำหรับรายการคำสั่ง (`WS-REQUESTS`) ในระบบจริงรายการคำสั่ง
  มักมาจากไฟล์ input, คิวข้อความ (message queue), หรือฐานข้อมูล ซึ่งต้องผ่านการตรวจสอบความถูกต้อง
  ของข้อมูลอย่างเข้มงวดกว่านี้มากก่อนนำไปใช้เป็นชื่อโปรแกรมใน `CALL`

### แบบฝึกหัดที่ 430.1

**โจทย์**: จงเพิ่มโปรแกรม `CALC-DIV` (หารสองจำนวน) และปรับ `FINAL430` ให้รองรับคำสั่ง `"DIV"` ได้
สำเร็จ (ไม่ต้อง handle การหารด้วยศูนย์ในเวอร์ชันนี้)

**เฉลย**: เพิ่มโปรแกรมย่อย:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CALC-DIV.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-A            PIC 9(5).
       01  LS-B            PIC 9(5).
       01  LS-RESULT       PIC 9(6).
       PROCEDURE DIVISION USING LS-A LS-B LS-RESULT.
           COMPUTE LS-RESULT = LS-A / LS-B
           GOBACK.
```

แล้วเพิ่ม `WHEN "DIV" MOVE "CALC-DIV" TO WS-PROG-NAME` ใน `EVALUATE` และเพิ่ม `subcalc-div.cob` เข้า
คำสั่ง compile ให้ครบทุกไฟล์ — โครงสร้างส่วนที่เหลือของโปรแกรมไม่ต้องแก้ไขเลย

---

## สรุปท้ายบท

ใน Part นี้ เราได้เรียนรู้การเรียกโปรแกรมย่อยแบบไดนามิกอย่างครบถ้วน:

- ความแตกต่างระหว่าง Static CALL (`CALL "NAME"`) กับ Dynamic CALL (`CALL identifier`)
- กลไกเบื้องหลัง: static call ผูกชื่อตอน compile/link ส่วน dynamic call ค้นหาตอนรัน
- `PROGRAM-POINTER` และ `SET ... TO ENTRY` สำหรับเก็บที่อยู่โปรแกรมไว้เรียกซ้ำอย่างมีประสิทธิภาพ
- `CANCEL` สำหรับรีเซ็ตสถานะภายในของโปรแกรมย่อยที่ถูกเรียกแบบ dynamic
- `ON EXCEPTION` สำหรับดักจับกรณีเรียกโปรแกรมไม่เจอ และ**ข้อจำกัดที่ยืนยันจากการทดสอบจริง**ว่า
  `NOT ON EXCEPTION` ใช้งานไม่ได้อย่างน่าเชื่อถือใน GnuCOBOL 4.0-early-dev บิลด์นี้
- รูปแบบ Dispatch Table สำหรับเลือกโปรแกรมจากตารางข้อมูลแทนการเขียน `IF/ELSE` ยาว ๆ
- รูปแบบ Callback ด้วยการส่ง `PROGRAM-POINTER` เป็นพารามิเตอร์ให้โปรแกรมอื่นเรียกกลับ
- ตัวอย่างรวมระบบประมวลผลคำสั่งแบบปลั๊กอินที่ผสานทุกเทคนิคเข้าด้วยกัน

Dynamic CALL และ PROGRAM-POINTER เป็นรากฐานสำคัญที่ทำให้ COBOL สามารถสร้างสถาปัตยกรรมที่ยืดหยุ่นและ
ขยายตัวได้ ซึ่งเป็นทักษะที่พบบ่อยในระบบ Enterprise ขนาดใหญ่ ใน Part ถัดไปเราจะเจาะลึกอีกขั้นเรื่อง
**หน่วยความจำแบบไดนามิก**: `POINTER` data item ทั่วไป, `BASED` storage, และคำสั่ง `ALLOCATE`/`FREE`
สำหรับการจัดสรรหน่วยความจำระหว่างการทำงาน ซึ่งเป็นเทคนิคขั้นสูงที่ COBOL-2002/2014 เพิ่มเข้ามาเพื่อ
รองรับโครงสร้างข้อมูลแบบไดนามิก เช่น Linked List

**[← กลับไป Part 042](part-042-class-condition-sign-condition.md)** | **[ไปยัง Part 044: Based Storage และ POINTER Data Item →](part-044-based-storage-pointer.md)**
