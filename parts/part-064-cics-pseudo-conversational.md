# Part 064: CICS ขั้นสูง: Pseudo-conversational Programming (ขั้นตอนที่ 631–640)

## คำนำของ Part นี้

Part 063 สอนให้เราออกแบบหน้าจอด้วย BMS และเขียนโปรแกรมที่ `SEND MAP`/`RECEIVE MAP` ได้อย่างถูกต้อง
แต่ถ้าสังเกตตัวอย่างในขั้นตอนที่ 629 ให้ดี จะพบคำถามที่ยังไม่มีคำตอบ: **ระหว่างที่โปรแกรมส่งหน้าจอ
ออกไปรอผู้ใช้พิมพ์ข้อมูล (อาจใช้เวลาหลายวินาทีถึงหลายนาที) โปรแกรมทำอะไรอยู่? มันนั่งรอ (wait) เฉย ๆ
ไหม?**

คำตอบคือ **ไม่ควรเลย** — และนี่คือหัวใจสำคัญที่สุดของสถาปัตยกรรม CICS ที่ทำให้มันรองรับผู้ใช้นับหมื่น
คนพร้อมกันบนเครื่องเดียวได้ Part นี้จะพาคุณเข้าใจปัญหาของการเขียนโปรแกรมแบบ **Conversational**
(สนทนาต่อเนื่อง) ที่ดูเหมือนจะเป็นวิธีธรรมชาติที่สุด แต่กลับเป็นหายนะด้านประสิทธิภาพในระบบจริง
และแนะนำวิธีแก้ที่ IBM คิดค้นขึ้น: **Pseudo-conversational Programming** ผ่านกลไก `COMMAREA` และ
`EXEC CICS RETURN TRANSID(...)` ซึ่งเป็นแนวคิดที่แม้จะดูแปลกในตอนแรก แต่จะเผยให้เห็นว่ามันคือบรรพบุรุษ
ทางความคิดของสถาปัตยกรรม **Stateless** ที่ระบบเว็บสมัยใหม่ทั้งหมดใช้อยู่ทุกวันนี้

> **⚠️ ข้อจำกัดของสภาพแวดล้อม**: เช่นเดียวกับ Part 063 เนื้อหาทั้งหมดใน Part นี้ต้องอาศัย CICS
> Transaction Server จริงซึ่ง**ไม่มีอยู่ในสภาพแวดล้อม GnuCOBOL ที่ใช้เรียนหลักสูตรนี้** ตัวอย่างโค้ด
> ทุกตัวอย่างที่มี `EXEC CICS` เป็น **ไวยากรณ์อ้างอิง (Reference Syntax)** ที่ถูกต้องตามมาตรฐาน IBM
> CICS จริง แต่**ไม่สามารถ compile/รันได้ในสภาพแวดล้อมนี้** จะมีป้ายกำกับ
> `⚠️ REFERENCE SYNTAX — ไม่สามารถ compile/รันได้ในสภาพแวดล้อมนี้` กำกับไว้เสมอ

---

## ขั้นตอนที่ 631: ปัญหาของ Conversational Programming — การถือครองทรัพยากรค้าง

### แนวคิด "ธรรมชาติ" ที่กลับกลายเป็นปัญหา

ถ้าให้โปรแกรมเมอร์ที่ไม่เคยรู้จัก CICS มาก่อนเขียนโปรแกรมสนทนากับผู้ใช้ สิ่งที่เขาจะเขียนตามสัญชาตญาณ
คือแบบนี้ (เรียกว่า **Conversational Style**):

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถ compile/รันได้ในสภาพแวดล้อมนี้ (และเป็นรูปแบบที่ไม่ควรใช้จริง)**

```cobol
       PROCEDURE DIVISION.
       MAIN-PARA.
      *> THIS IS THE "WRONG" WAY - conversational style.
      *> The task below sends a screen, then immediately waits
      *> right here inside the SAME task for the user to respond.
           EXEC CICS SEND MAP('CUSTINQ1')
               MAPSET('CUSTMSET')
               ERASE
           END-EXEC.

           EXEC CICS RECEIVE MAP('CUSTINQ1')
               MAPSET('CUSTMSET')
               INTO(CUSTINQ1I)
           END-EXEC.

           PERFORM LOOKUP-CUSTOMER.

           EXEC CICS SEND MAP('CUSTINQ1')
               MAPSET('CUSTMSET')
               DATAONLY
           END-EXEC.

           EXEC CICS RETURN
           END-EXEC.
```

ดูผิวเผินแล้วโค้ดนี้ตรงไปตรงมามาก: ส่งจอ, รอรับข้อมูล, ประมวลผล, ส่งผลลัพธ์, จบ เหมือนเขียนโปรแกรม
console ทั่วไปที่เรียนมาตลอดทั้งหลักสูตร (`DISPLAY`/`ACCEPT` ใน Part 007) แต่ปัญหาใหญ่ซ่อนอยู่ที่คำสั่ง
`RECEIVE MAP` บรรทัดที่สอง

### ปัญหาที่แท้จริง: Task ยังคงถูกจับจองอยู่ตลอดเวลาที่ผู้ใช้ "คิด"

เมื่อ `EXEC CICS RECEIVE MAP` ทำงานในรูปแบบ Conversational **Task (หน่วยงานที่ CICS สร้างขึ้นเพื่อ
รันธุรกรรมนี้) จะถูกค้างไว้ในสถานะ "wait for terminal input" จนกว่าผู้ใช้จะกด Enter** ปัญหาคือ
มนุษย์อ่านหน้าจอและพิมพ์ข้อมูลช้ากว่าคอมพิวเตอร์มหาศาล — อาจใช้เวลา 5 วินาที 30 วินาที หรือแม้แต่
5 นาทีถ้าผู้ใช้ลุกไปชงกาแฟกลางคัน ตลอดเวลานั้น **ทรัพยากรทั้งหมดที่ Task นี้ถืออยู่จะไม่ถูกปล่อยคืน
ให้ระบบเลย**:

1. **หน่วยความจำ (Storage)**: พื้นที่ Working-Storage ทั้งหมดของโปรแกรมยังคงถูกจองอยู่
2. **Task Control Block**: โครงสร้างข้อมูลภายในที่ CICS ใช้ติดตาม Task นี้ยังคงมีอยู่ในระบบ
3. **Lock บนทรัพยากรที่เพิ่งอ่าน/แก้ไข**: ถ้าโปรแกรมเพิ่ง `READ` record จาก VSAM หรือ DB2 มาก่อนส่งจอ
   (เช่น ล็อก record ไว้เพื่อรอแก้ไข) lock นั้นจะยังคงค้างอยู่จนกว่าผู้ใช้จะตอบกลับ — ผู้ใช้คนอื่นที่
   ต้องการเข้าถึง record เดียวกันจะต้องรอ
4. **จำนวน Task พร้อมกันสูงสุด (MXT)**: CICS จำกัดจำนวน Task ที่รันพร้อมกันได้สูงสุด (ค่า `MXT` ใน
   System Initialization Table) ถ้า Task ค้างรอผู้ใช้นานเกินไป จำนวน Task ที่เหลือให้ผู้ใช้คนอื่นใช้
   งานจะลดลงเรื่อย ๆ จนระบบล่ม (system saturation) ในช่วงเวลาที่มีผู้ใช้พร้อมกันมาก

### ทำไมปัญหานี้ถึงร้ายแรงในระบบจริง

ลองจินตนาการธนาคารที่มีพนักงาน 5,000 คนใช้ระบบ CICS พร้อมกันในเวลาทำการ ถ้าแต่ละคนใช้โปรแกรมแบบ
Conversational และใช้เวลาเฉลี่ย 20 วินาทีต่อหน้าจอ (อ่านข้อมูล คิด แล้วพิมพ์) นั่นหมายความว่า
ระบบต้องรองรับ Task ที่ "ค้าง" อยู่พร้อมกันถึง 5,000 Task ตลอดเวลา ทั้งที่ในความเป็นจริง CPU ใช้เวลา
ประมวลผลจริง (การคำนวณ, การอ่านฐานข้อมูล) ของแต่ละธุรกรรมเพียงไม่กี่มิลลิวินาทีเท่านั้น — ทรัพยากร
มหาศาลถูกสูญเปล่าไปกับการ "รอเฉย ๆ"

### ข้อควรระวัง

- Conversational programming **ไม่ใช่ compile error** — โค้ดในขั้นตอนนี้ compile และรันได้ปกติถ้ามี
  CICS จริง ปัญหาคือด้าน **สถาปัตยกรรมและ scalability** ไม่ใช่ด้านไวยากรณ์ ทำให้บั๊กแบบนี้ตรวจจับ
  ยากมากในการทดสอบตอนมีผู้ใช้น้อย แต่จะแสดงอาการรุนแรงเมื่อระบบใช้งานจริงที่มีผู้ใช้จำนวนมาก
- CICS สมัยใหม่มีการตั้งค่า **Runaway Task Interval** ที่จะ cancel task ที่ใช้ CPU นานเกินกำหนดโดย
  อัตโนมัติ แต่การรอ terminal input **ไม่นับเป็นการใช้ CPU** (เป็นสถานะ wait) จึงไม่ถูก runaway
  detection จับได้ ทำให้ปัญหานี้ซ่อนเร้นอยู่ได้นาน

### แบบฝึกหัดที่ 631.1

**โจทย์**: จงอธิบายว่าทำไมการเขียนโปรแกรม CICS แบบ Conversational จึงไม่แสดงปัญหาให้เห็นชัดเจนตอน
ทดสอบด้วยผู้ใช้คนเดียว แต่จะกลายเป็นปัญหาร้ายแรงเมื่อมีผู้ใช้หลายพันคนพร้อมกัน

**เฉลย**: เพราะทรัพยากรของ CICS (จำนวน Task สูงสุดที่รันพร้อมกันได้ หน่วยความจำที่จองไว้ต่อ Task)
มีจำกัดตามการตั้งค่าระบบ (เช่น `MXT`) เมื่อมีผู้ใช้คนเดียวทดสอบ จำนวน Task ที่ค้างรอพร้อมกันมีแค่ 1
ซึ่งไม่กระทบทรัพยากรของระบบเลย แต่เมื่อมีผู้ใช้จริงหลายพันคนใช้งานพร้อมกัน แต่ละคนก็จะมี Task ของตัวเอง
ที่ค้างรออยู่ตลอดเวลาที่กำลัง "คิด" ทำให้จำนวน Task ที่ค้างรอสะสมขึ้นเรื่อย ๆ จนอาจแตะเพดาน `MXT`
ทำให้ผู้ใช้คนใหม่ที่พยายามเริ่มธุรกรรมไม่สามารถเข้าระบบได้เลย (ระบบดูเหมือน "ค้าง" ทั้งที่ CPU จริง ๆ
แล้วว่างเกือบตลอดเวลา)

---

## ขั้นตอนที่ 632: แนวคิด Pseudo-conversational และ EXEC CICS RETURN TRANSID

### หลักการ: จบ Task ทุกครั้งที่ส่งจอ แล้วค่อยเริ่ม Task ใหม่เมื่อผู้ใช้ตอบกลับ

**Pseudo-conversational Programming** คือรูปแบบที่ CICS ใช้แก้ปัญหาในขั้นตอนที่ 631 ด้วยแนวคิดที่
ฟังดูขัดสามัญสำนึกในตอนแรก: **แทนที่จะให้ Task เดียวคงอยู่ตลอดการสนทนาทั้งหมด ให้ Task จบสมบูรณ์ทันที
หลังจากส่งหน้าจอออกไป (คืนทรัพยากรทั้งหมดกลับสู่ระบบ) แล้วเมื่อผู้ใช้กด Enter ค่อยให้ CICS
เริ่ม Task ใหม่ขึ้นมาทำงานต่อ**

คำสั่งหัวใจสำคัญที่ทำให้สิ่งนี้เกิดขึ้นคือ `EXEC CICS RETURN TRANSID(...)`:

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถ compile/รันได้ในสภาพแวดล้อมนี้**

```cobol
           EXEC CICS RETURN
               TRANSID('CINQ')
               COMMAREA(WS-COMMAREA)
               LENGTH(LENGTH OF WS-COMMAREA)
           END-EXEC.
```

`TRANSID('CINQ')` บอก CICS ว่า **"เมื่อผู้ใช้ที่เทอร์มินัลนี้กด Enter หรือ PF Key ครั้งถัดไป ให้เริ่ม
Transaction 'CINQ' ขึ้นมาใหม่โดยอัตโนมัติ"** — Task ปัจจุบันจะจบสมบูรณ์ทันทีหลังบรรทัดนี้ (เทียบเท่า
`STOP RUN` ของโปรแกรมทั่วไป) ทรัพยากรทุกอย่างที่ Task นี้ถืออยู่ (หน่วยความจำ, lock, task control
block) จะถูกปล่อยคืนสู่ระบบทันที

### วงจรชีวิตของ Pseudo-conversational Task

```
[ผู้ใช้เห็นหน้าจอเปล่า พิมพ์ "INQ" ที่ transaction prompt]
        |
        v
[CICS สร้าง TASK #1 สำหรับ transaction CINQ]
        |
        v
[TASK #1: SEND MAP (ส่งหน้าจอค้นหา) --> EXEC CICS RETURN TRANSID('CINQ')]
        |
        v
[TASK #1 จบสมบูรณ์ - ทรัพยากรทั้งหมดถูกคืน]
        |
        |   <-- ผู้ใช้ใช้เวลาคิด/พิมพ์ข้อมูล (0 Task ใด ๆ ถูกจองไว้เลยช่วงนี้!) -->
        |
        v
[ผู้ใช้กด ENTER]
        |
        v
[CICS สร้าง TASK #2 ใหม่ สำหรับ transaction CINQ (คนละ Task กับ #1 โดยสิ้นเชิง)]
        |
        v
[TASK #2: RECEIVE MAP (รับข้อมูลที่ผู้ใช้พิมพ์) --> ประมวลผล --> SEND MAP ผลลัพธ์
         --> EXEC CICS RETURN TRANSID('CINQ')]
        |
        v
[TASK #2 จบสมบูรณ์ - ทรัพยากรทั้งหมดถูกคืนอีกครั้ง]
```

### จุดที่ต้องทำความเข้าใจให้แจ่มแจ้ง: มันคือคนละ Task กันจริง ๆ

ประเด็นสำคัญที่สุดที่ผู้เริ่มต้นมักเข้าใจผิดคือ **TASK #1 และ TASK #2 ไม่ใช่ Task เดียวกันที่ "ตื่นขึ้น
มา" ทำงานต่อ** แต่เป็น**การรันโปรแกรมใหม่ตั้งแต่ต้นอีกครั้งอย่างสมบูรณ์** (เหมือนเรียก `cobc` compile
โปรแกรมแล้วรันใหม่) หน่วยความจำ Working-Storage ทั้งหมดของ TASK #1 **หายไปหมดแล้ว** เมื่อ TASK #2
เริ่มทำงาน ตัวแปรทุกตัวจะกลับไปเป็นค่าเริ่มต้น (`VALUE` clause) เหมือนโปรแกรมเพิ่งเริ่มทำงานครั้งแรก

**คำถามที่ตามมาทันที**: ถ้าหน่วยความจำหายไปหมดทุกครั้ง แล้วโปรแกรมจะรู้ได้อย่างไรว่า "ตอนนี้กำลังอยู่
ขั้นตอนไหนของการสนทนา" หรือ "ผู้ใช้เพิ่งค้นหาอะไรไปก่อนหน้านี้"? คำตอบคือกลไกที่เรียกว่า **COMMAREA**
ซึ่งจะอธิบายในขั้นตอนถัดไป

### ข้อควรระวัง

- อย่าสับสนระหว่าง "Transaction" กับ "Task" — **Transaction** คือชื่อสี่ตัวอักษร (เช่น `CINQ`) ที่
  ผูกกับโปรแกรมหนึ่งตัวใน Program Control Table ส่วน **Task** คือการรันโปรแกรมนั้นหนึ่งรอบ
  (instance) — Transaction เดียวกันสามารถถูกรันเป็นหลาย Task พร้อมกันได้ (เช่น ผู้ใช้ 100 คนพิมพ์
  `CINQ` พร้อมกัน จะได้ 100 Task ของ transaction เดียวกัน) และ pseudo-conversational หนึ่งการสนทนา
  จะประกอบด้วยหลาย Task ของ transaction เดียวกันเรียงต่อกันตามเวลา
- ถ้าลืมใส่ `TRANSID(...)` ใน `EXEC CICS RETURN` โปรแกรมจะจบการทำงานแบบสมบูรณ์และเทอร์มินัลจะกลับไป
  สู่สถานะว่าง — CICS จะไม่เริ่ม transaction ใดโดยอัตโนมัติเมื่อผู้ใช้กด Enter ครั้งถัดไป (ผู้ใช้ต้อง
  พิมพ์ชื่อ transaction ใหม่เอง) ซึ่งไม่เหมาะกับการสนทนาต่อเนื่องหลายหน้าจอ

### แบบฝึกหัดที่ 632.1

**โจทย์**: จงอธิบายว่าทำไมการที่ TASK #2 เป็น "การรันโปรแกรมใหม่ตั้งแต่ต้น" (ไม่ใช่ Task เดิมที่ตื่น
ขึ้นมา) จึงเป็นทั้งข้อดีและสิ่งที่โปรแกรมเมอร์ต้องระมัดระวังไปพร้อมกัน

**เฉลยแนวทาง**: ข้อดีคือทรัพยากรของระบบ (หน่วยความจำ, task control block, lock) ถูกปล่อยคืนอย่าง
สมบูรณ์ทุกครั้งที่ Task จบ ทำให้ระบบรองรับผู้ใช้จำนวนมากพร้อมกันได้โดยไม่สะสมภาระ แม้จะมีผู้ใช้หลายพัน
คนที่กำลัง "คิด" อยู่พร้อมกัน แต่ในขณะเดียวกัน สิ่งที่ต้องระวังคือโปรแกรมเมอร์ต้อง**ออกแบบวิธีเก็บสถานะ
ที่จำเป็นด้วยตนเอง**อย่างจงใจ (ผ่าน COMMAREA) เพราะไม่มีตัวแปรใดใน Working-Storage ที่จะ "จำ" ค่าจาก
Task ก่อนหน้าได้เลยโดยอัตโนมัติ ต่างจากโปรแกรม Conversational ที่ตัวแปรทุกตัวยังอยู่ในหน่วยความจำเดิม
ตลอดการสนทนาเพราะเป็น Task เดียวกัน

---

## ขั้นตอนที่ 633: COMMAREA — การส่งสถานะข้ามไปยัง Task ถัดไป

### COMMAREA คือ "ความทรงจำ" เพียงหนึ่งเดียวที่ข้าม Task ได้

**COMMAREA (Communication Area)** คือก้อนหน่วยความจำขนาดเล็กที่ CICS **คัดลอกและส่งต่อ**ให้กับ Task
ถัดไปของ transaction เดียวกันบนเทอร์มินัลเดียวกัน มันคือกลไกเดียวที่ทำให้ TASK #2 (ในขั้นตอนที่ 632)
รู้ได้ว่า TASK #1 เคยทำอะไรไปแล้วบ้าง

### วิธีส่งข้อมูลออกไปกับ COMMAREA

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถ compile/รันได้ในสภาพแวดล้อมนี้**

```cobol
       WORKING-STORAGE SECTION.
       01  WS-COMMAREA.
           05  WS-STEP-INDICATOR    PIC X(1).
           05  WS-CUST-ID           PIC 9(5).
           05  WS-RETRY-COUNT       PIC 9(2).

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE "2" TO WS-STEP-INDICATOR.
           MOVE 10001 TO WS-CUST-ID.
           MOVE 0 TO WS-RETRY-COUNT.

           EXEC CICS SEND MAP('CUSTINQ1')
               MAPSET('CUSTMSET')
               ERASE
           END-EXEC.

           EXEC CICS RETURN
               TRANSID('CINQ')
               COMMAREA(WS-COMMAREA)
               LENGTH(LENGTH OF WS-COMMAREA)
           END-EXEC.
```

### วิธีรับข้อมูลกลับมาใน Task ถัดไป: EIBCALEN และ DFHCOMMAREA

Task ใหม่ที่ CICS เริ่มขึ้นมาเมื่อผู้ใช้ตอบกลับจะได้รับ COMMAREA กลับผ่านตัวแปรพิเศษชื่อ
**`DFHCOMMAREA`** ซึ่งต้องประกาศไว้ใน `LINKAGE SECTION` (ทบทวนแนวคิด `LINKAGE SECTION` และการส่ง
พารามิเตอร์ `BY REFERENCE` จาก Part 032 — หลักการเดียวกันเป๊ะ เพียงแต่ CICS เป็นผู้ส่งให้แทนโปรแกรม
อื่นที่ `CALL`):

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถ compile/รันได้ในสภาพแวดล้อมนี้**

```cobol
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-WORK-AREA             PIC X(50).

       LINKAGE SECTION.
       01  DFHCOMMAREA.
           05  LS-STEP-INDICATOR    PIC X(1).
           05  LS-CUST-ID           PIC 9(5).
           05  LS-RETRY-COUNT       PIC 9(2).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> EIBCALEN is a field in the EIB (Execute Interface Block,
      *> a structure CICS supplies automatically to every task -
      *> conceptually similar to EIBAID that Part 062 introduced).
      *> It holds the LENGTH of the COMMAREA received - zero means
      *> "no COMMAREA arrived", which only happens on the VERY
      *> FIRST task of a brand-new conversation.
           IF EIBCALEN = 0
               PERFORM FIRST-TIME-LOGIC
           ELSE
               PERFORM CONTINUING-LOGIC
           END-IF.
```

### ข้อควรระวัง

- **`EIBCALEN = 0` คือสัญญาณเดียวที่บอกว่า "นี่คือจุดเริ่มต้นของการสนทนาใหม่"** โปรแกรมเมอร์ที่ลืม
  ตรวจสอบเงื่อนไขนี้จะเจอปัญหาทันทีในการรันครั้งแรก เพราะ `DFHCOMMAREA` จะยังไม่มีข้อมูลที่มีความหมาย
  ใด ๆ อยู่เลย (การอ่านค่าจากมันโดยไม่ตรวจสอบ `EIBCALEN` ก่อนอาจทำให้โปรแกรม abend หรือได้ค่าขยะ)
- ขนาดของ COMMAREA **มีขีดจำกัดตามการตั้งค่าระบบ** (ปกติสูงสุด 32,763 ไบต์ ในบาง configuration อาจ
  จำกัดน้อยกว่านั้นมาก) ไม่ควรยัด COMMAREA ด้วยข้อมูลขนาดใหญ่ (เช่น ทั้ง record ของลูกค้าที่มีร้อย
  ฟิลด์) — ควรเก็บเฉพาะ "สถานะที่จำเป็นต้องจำข้ามหน้าจอ" เท่านั้น เช่น key ที่ใช้อ่านข้อมูลใหม่จาก
  ฐานข้อมูลอีกครั้งเมื่อจำเป็น แทนที่จะเก็บข้อมูลทั้งหมด
- โครงสร้างของ `WS-COMMAREA` (ฝั่งส่ง) และ `DFHCOMMAREA` (ฝั่งรับ) **ต้องตรงกันทุกไบต์** เพราะ CICS
  แค่คัดลอกข้อมูลดิบไปมา ไม่มีการตรวจสอบชนิดข้อมูลใด ๆ ให้ (เหมือนกับ `BY REFERENCE` ใน Part 032
  ที่เตือนเรื่องนี้ไว้แล้ว)

### แบบฝึกหัดที่ 633.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรมเมอร์ควรเก็บเฉพาะ "รหัสอ้างอิง" (เช่น customer ID) ใน COMMAREA
แทนที่จะเก็บข้อมูลทั้งหมดของลูกค้า (ชื่อ ที่อยู่ ยอดเงิน ประวัติธุรกรรม ฯลฯ)

**เฉลย**: เพราะ COMMAREA มีขีดจำกัดขนาดและการคัดลอกข้อมูลจำนวนมากข้าม Task ทุกครั้งที่ผู้ใช้กด Enter
จะสิ้นเปลืองทรัพยากรระบบโดยไม่จำเป็น (ยิ่งข้อมูลใหญ่ ยิ่งใช้เวลาคัดลอกนานขึ้น) การเก็บเฉพาะรหัสอ้างอิง
(เช่น customer ID 5 หลัก) ทำให้ COMMAREA มีขนาดเล็กมาก และเมื่อ Task ถัดไปต้องใช้ข้อมูลเต็มของลูกค้า
ก็สามารถอ่านใหม่จากฐานข้อมูล (VSAM/DB2) ด้วยรหัสนั้นได้ทันที ซึ่งข้อมูลที่อ่านใหม่ยังรับประกันความ
ถูกต้องล่าสุดด้วย (ไม่ใช่ข้อมูลเก่าที่ค้างมาจากรอบก่อนซึ่งอาจมีคนอื่นแก้ไขไปแล้วระหว่างที่ผู้ใช้คนแรก
กำลังคิดอยู่)

---

## ขั้นตอนที่ 634: โครงสร้าง PROCEDURE DIVISION มาตรฐานสำหรับโปรแกรม Pseudo-conversational

### รูปแบบ (Pattern) ที่พบในโปรแกรม CICS แทบทุกตัว

เมื่อเข้าใจ `EIBCALEN` และ `COMMAREA` แล้ว เราสามารถสรุปโครงสร้างมาตรฐานที่โปรแกรม CICS
pseudo-conversational เกือบทุกตัวในโลกใช้ร่วมกันได้ดังนี้:

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถ compile/รันได้ในสภาพแวดล้อมนี้**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CUSTINQ2.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COMMAREA.
           05  WS-STEP-INDICATOR    PIC X(1).
               88  STEP-INITIAL     VALUE '1'.
               88  STEP-AWAIT-INPUT VALUE '2'.
           05  WS-CUST-ID           PIC 9(5).

       LINKAGE SECTION.
       01  DFHCOMMAREA              PIC X(6).

       PROCEDURE DIVISION.
       MAIN-PARA.
      *> Step 1: figure out WHERE we are in the conversation.
           IF EIBCALEN = 0
               PERFORM INITIALIZE-FIRST-TASK
           ELSE
               MOVE DFHCOMMAREA TO WS-COMMAREA
               EVALUATE TRUE
                   WHEN STEP-AWAIT-INPUT
                       PERFORM PROCESS-USER-INPUT
                   WHEN OTHER
                       PERFORM HANDLE-UNKNOWN-STEP
               END-EVALUATE
           END-IF.

      *> Step 2: every path ends the SAME way - RETURN with a
      *> TRANSID so CICS knows to restart us on the next Enter,
      *> carrying whatever WS-COMMAREA now holds forward.
           EXEC CICS RETURN
               TRANSID('CINQ')
               COMMAREA(WS-COMMAREA)
               LENGTH(LENGTH OF WS-COMMAREA)
           END-EXEC.

       INITIALIZE-FIRST-TASK.
           SET STEP-AWAIT-INPUT TO TRUE.
           MOVE SPACES TO CUSTINQ1O.
           EXEC CICS SEND MAP('CUSTINQ1')
               MAPSET('CUSTMSET')
               ERASE
           END-EXEC.

       PROCESS-USER-INPUT.
           EXEC CICS RECEIVE MAP('CUSTINQ1')
               MAPSET('CUSTMSET')
               INTO(CUSTINQ1I)
           END-EXEC.
      *> Business logic that reads CUSTIDI, looks up the customer,
      *> and SENDs the response map goes here (same pattern as
      *> Part 063, step 629) - omitted for brevity in this step.
           PERFORM LOOKUP-AND-RESPOND.

       HANDLE-UNKNOWN-STEP.
      *> Defensive programming: if WS-STEP-INDICATOR ever holds a
      *> value we do not recognize (data corruption, a bug in an
      *> earlier release), never let the task silently misbehave -
      *> log it and reset the conversation back to the start.
           EXEC CICS SEND TEXT
               FROM('INTERNAL ERROR - RESTARTING')
               ERASE
           END-EXEC.
           SET STEP-AWAIT-INPUT TO TRUE.
```

### อธิบายจุดสำคัญ

- **ทุกเส้นทางของโปรแกรมจบลงที่ `EXEC CICS RETURN` เดียวกันเสมอ** — นี่คือรูปแบบบังคับของ
  pseudo-conversational ทำให้แน่ใจว่าไม่ว่าจะเกิดอะไรขึ้นระหว่างทาง Task จะจบและคืนทรัพยากรสมบูรณ์
  เสมอ
- **`WS-STEP-INDICATOR`** ทำหน้าที่เป็น "ตัวแปรสถานะของเครื่องจักรสถานะ (state machine)" — บอกว่า
  ตอนนี้การสนทนาอยู่ที่ขั้นไหน คล้ายกับ `88-level` Condition Names ที่เรียนใน Part 010 เพียงแต่ค่า
  ของมันถูกเก็บรักษาข้าม Task ผ่าน COMMAREA
- **`HANDLE-UNKNOWN-STEP`** คือแนวคิด Defensive Programming ที่สำคัญมาก — เพราะ COMMAREA เป็นข้อมูล
  ดิบที่ไม่มีการตรวจสอบชนิดจาก CICS จึงควรมี branch ดักค่าที่ไม่คาดคิดไว้เสมอ

### ข้อควรระวัง

- โครงสร้างนี้หมายความว่า **หนึ่งโปรแกรม COBOL หนึ่งตัวต้องรับผิดชอบทุกขั้นตอนของการสนทนาทั้งหมด**
  (ไม่ใช่แยกโปรแกรมคนละตัวสำหรับแต่ละหน้าจอ) เพราะ transaction เดียวกัน (`CINQ`) ผูกกับโปรแกรมเดียว
  ใน Program Control Table เสมอ
- ถ้าลืม `SET STEP-AWAIT-INPUT TO TRUE` ก่อน `EXEC CICS RETURN` ใน `INITIALIZE-FIRST-TASK` Task
  ถัดไปจะไม่รู้ว่าต้องประมวลผลแบบไหน (`WS-STEP-INDICATOR` จะมีค่าว่างเปล่าตาม default ของ
  `WORKING-STORAGE`) ทำให้ตกไปที่ `HANDLE-UNKNOWN-STEP` โดยไม่ตั้งใจ

### แบบฝึกหัดที่ 634.1

**โจทย์**: จงอธิบายว่าทำไมโปรแกรม CICS pseudo-conversational เกือบทุกตัวต้องมีการตรวจสอบ
`IF EIBCALEN = 0` เป็นคำสั่งแรกสุดใน `PROCEDURE DIVISION` เสมอ

**เฉลย**: เพราะ `EIBCALEN = 0` คือหลักฐานเดียวที่บอกได้อย่างแน่ชัดว่า Task ที่กำลังรันอยู่นี้เป็น
"จุดเริ่มต้นของการสนทนาใหม่" (ยังไม่มี COMMAREA ใด ๆ ส่งมาก่อนหน้า) หากไม่ตรวจสอบเงื่อนไขนี้ก่อน
โปรแกรมอาจพยายามอ่านค่าจาก `DFHCOMMAREA` ที่ยังไม่มีข้อมูลจริง ซึ่งอาจทำให้เกิด error รุนแรง (เช่น
storage violation) หรือได้ค่าที่ไม่มีความหมายมาประมวลผลต่อ การตรวจสอบนี้เป็นคำสั่งแรกสุดเสมอเพราะ
เป็นการตัดสินใจพื้นฐานที่สุดว่าโปรแกรมควรทำงานในเส้นทางไหนต่อไป

---

## ขั้นตอนที่ 635: ออกแบบ State Machine ด้วยตัวแปรสถานะใน COMMAREA

### การสนทนาหลายขั้นตอนต้องการมากกว่าแค่ "ครั้งแรก" กับ "ครั้งถัดไป"

ตัวอย่างในขั้นตอนที่ 634 มีแค่ 2 สถานะ (`STEP-INITIAL` และ `STEP-AWAIT-INPUT`) แต่ระบบจริงมักมีฟอร์ม
หลายหน้าจอต่อเนื่องกัน เช่น ระบบเปิดบัญชีลูกค้าใหม่ที่มี 3 หน้าจอ: (1) กรอกข้อมูลส่วนตัว (2) กรอก
ข้อมูลที่อยู่ (3) ยืนยันและบันทึก — แต่ละหน้าจอต้อง "จำ" ว่าตัวเองอยู่ขั้นไหน และข้อมูลจากขั้นก่อนหน้า
ทั้งหมดต้องถูกพกพาต่อไปเรื่อย ๆ ผ่าน COMMAREA จนกว่าจะถึงขั้นตอนสุดท้ายที่บันทึกข้อมูลจริง

### ออกแบบ COMMAREA สำหรับ State Machine หลายขั้นตอน

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถ compile/รันได้ในสภาพแวดล้อมนี้**

```cobol
       01  WS-COMMAREA.
           05  WS-CURRENT-STEP      PIC 9(1).
               88  STEP-PERSONAL-INFO   VALUE 1.
               88  STEP-ADDRESS-INFO    VALUE 2.
               88  STEP-CONFIRMATION    VALUE 3.
      *> Data accumulated across screens - each screen only ADDS
      *> to this area, never erases what a previous screen filled.
           05  WS-CUST-NAME         PIC X(20).
           05  WS-CUST-DOB          PIC 9(8).
           05  WS-CUST-ADDR-1       PIC X(30).
           05  WS-CUST-ADDR-2       PIC X(30).
           05  WS-CUST-POSTAL       PIC 9(5).

       PROCEDURE DIVISION.
       MAIN-PARA.
           IF EIBCALEN = 0
               PERFORM SHOW-PERSONAL-INFO-SCREEN
           ELSE
               MOVE DFHCOMMAREA TO WS-COMMAREA
               EVALUATE TRUE
                   WHEN STEP-PERSONAL-INFO
                       PERFORM RECEIVE-PERSONAL-INFO
                       PERFORM SHOW-ADDRESS-SCREEN
                   WHEN STEP-ADDRESS-INFO
                       PERFORM RECEIVE-ADDRESS-INFO
                       PERFORM SHOW-CONFIRMATION-SCREEN
                   WHEN STEP-CONFIRMATION
                       PERFORM RECEIVE-CONFIRMATION
                       PERFORM SAVE-NEW-CUSTOMER
               END-EVALUATE
           END-IF.
           EXEC CICS RETURN
               TRANSID('CNEW')
               COMMAREA(WS-COMMAREA)
               LENGTH(LENGTH OF WS-COMMAREA)
           END-EXEC.
```

### หลักการออกแบบที่สำคัญ

1. **แต่ละหน้าจอ "รับ" ข้อมูลของหน้าจอก่อนหน้าเข้ามาใน COMMAREA แล้ว "เพิ่ม" ข้อมูลของตัวเองต่อท้าย**
   — สังเกตว่า `RECEIVE-ADDRESS-INFO` ไม่ได้ล้างค่า `WS-CUST-NAME` ที่มาจากขั้นตอนก่อนหน้าทิ้งเลย
2. **ค่า `WS-CURRENT-STEP` ต้องถูกอัปเดตก่อนหรือระหว่าง `SHOW-*-SCREEN`** ของขั้นตอนถัดไปเสมอ เพื่อให้
   Task ถัดไปรู้ว่าต้องประมวลผลอะไรเมื่อ COMMAREA กลับเข้ามา
3. **ขั้นตอนสุดท้าย (`STEP-CONFIRMATION`) คือจุดเดียวที่ข้อมูลจะถูกเขียนลงฐานข้อมูลจริง** (ผ่าน VSAM
   หรือ DB2) — ขั้นตอนก่อนหน้าทั้งหมดเป็นแค่การ "สะสม" ข้อมูลไว้ใน COMMAREA เท่านั้น ยังไม่กระทบข้อมูล
   จริงในระบบ ทำให้ผู้ใช้ยกเลิกกลางทางได้โดยไม่มีข้อมูลค้างคาในฐานข้อมูล

### ข้อควรระวัง

- COMMAREA ที่สะสมข้อมูลหลายหน้าจอจะมีขนาดใหญ่ขึ้นเรื่อย ๆ ตามจำนวนฟิลด์ที่ต้องจำ ต้องตรวจสอบว่า
  ยังไม่เกินขีดจำกัดขนาด COMMAREA ของระบบเสมอ (ทบทวนจากขั้นตอนที่ 633)
- ถ้าผู้ใช้กด **PF3 (ยกเลิก)** กลางทาง โปรแกรมต้องมีตรรกะจัดการแยกต่างหาก (ตรวจสอบ `EIBAID` ทบทวนจาก
  Part 062) เพื่อไม่ให้ตกลงไปในสถานะถัดไปโดยไม่ตั้งใจทั้งที่ผู้ใช้ต้องการยกเลิก

### แบบฝึกหัดที่ 635.1

**โจทย์**: จงอธิบายว่าทำไมการเขียนข้อมูลลงฐานข้อมูลจริง (VSAM/DB2) ควรเกิดขึ้นที่ขั้นตอนสุดท้าย
(`STEP-CONFIRMATION`) เท่านั้น แทนที่จะเขียนบางส่วนทีละหน้าจอไปเรื่อย ๆ

**เฉลย**: เพราะถ้าเขียนข้อมูลบางส่วนลงฐานข้อมูลทันทีตั้งแต่หน้าจอแรก (เช่น สร้าง record ลูกค้าใหม่
ด้วยแค่ชื่อ-วันเกิด) แล้วผู้ใช้ตัดสินใจยกเลิกกลางทาง (ปิดเทอร์มินัลไปเฉย ๆ หรือกด PF3) ฐานข้อมูลจะมี
record ที่ไม่สมบูรณ์ (ไม่มีที่อยู่) ค้างอยู่ ซึ่งอาจก่อปัญหาให้ระบบอื่นที่พึ่งพาข้อมูลนี้ในภายหลัง
การสะสมข้อมูลไว้ใน COMMAREA ก่อน แล้วเขียนลงฐานข้อมูลเป็นก้อนเดียวสมบูรณ์ที่ขั้นตอนสุดท้ายเท่านั้น
ทำให้มั่นใจได้ว่าข้อมูลที่เข้าสู่ฐานข้อมูลจริงจะสมบูรณ์เสมอ (all-or-nothing) — แนวคิดนี้ใกล้เคียงกับ
หลักการ Transaction ใน DB2 ที่จะเรียนเพิ่มเติมเรื่อง COMMIT ใน Part 066 (Commit Interval) เช่นกัน

---

## ขั้นตอนที่ 636: ตัวอย่างสมบูรณ์ — Wizard เปิดบัญชีลูกค้าใหม่แบบ 3 หน้าจอ

### รวมทุกอย่างเข้าด้วยกัน

มาดูตัวอย่างโปรแกรมที่สมบูรณ์กว่าขั้นตอนที่ 635 โดยใส่รายละเอียดการรับ/ส่งจอครบทุกขั้นตอน (ย่อ BMS
Symbolic Map ให้เป็นชื่อฟิลด์ทั่วไป สมมติว่ามี Map แยกกัน 3 หน้าจอตามหลักการ Part 063: `PERSMAP`,
`ADDRMAP`, `CONFMAP`):

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถ compile/รันได้ในสภาพแวดล้อมนี้**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. NEWCUST.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-COMMAREA.
           05  WS-CURRENT-STEP      PIC 9(1).
               88  STEP-PERSONAL    VALUE 1.
               88  STEP-ADDRESS     VALUE 2.
               88  STEP-CONFIRM     VALUE 3.
           05  WS-CUST-NAME         PIC X(20).
           05  WS-CUST-DOB          PIC 9(8).
           05  WS-CUST-ADDR-1       PIC X(30).
           05  WS-CUST-ADDR-2       PIC X(30).
           05  WS-CUST-POSTAL       PIC 9(5).
           05  WS-NEW-CUST-ID       PIC 9(5).

       LINKAGE SECTION.
       01  DFHCOMMAREA              PIC X(97).

       PROCEDURE DIVISION.
       MAIN-PARA.
           IF EIBCALEN = 0
               PERFORM SHOW-PERSONAL-SCREEN
           ELSE
               MOVE DFHCOMMAREA TO WS-COMMAREA
      *> PF3 cancels the wizard from ANY step - checked before the
      *> state machine so it always works no matter where we are.
               IF EIBAID = DFHPF3
                   PERFORM CANCEL-WIZARD
               ELSE
                   EVALUATE TRUE
                       WHEN STEP-PERSONAL
                           PERFORM RECEIVE-PERSONAL
                           PERFORM SHOW-ADDRESS-SCREEN
                       WHEN STEP-ADDRESS
                           PERFORM RECEIVE-ADDRESS
                           PERFORM SHOW-CONFIRM-SCREEN
                       WHEN STEP-CONFIRM
                           PERFORM RECEIVE-CONFIRM
                           PERFORM SAVE-AND-FINISH
                   END-EVALUATE
               END-IF
           END-IF.
           EXEC CICS RETURN
               TRANSID('CNEW')
               COMMAREA(WS-COMMAREA)
               LENGTH(LENGTH OF WS-COMMAREA)
           END-EXEC.

       SHOW-PERSONAL-SCREEN.
           SET STEP-PERSONAL TO TRUE.
           MOVE SPACES TO PERSMAPO.
           EXEC CICS SEND MAP('PERSMAP') MAPSET('NEWCUST')
               ERASE
           END-EXEC.

       RECEIVE-PERSONAL.
           EXEC CICS RECEIVE MAP('PERSMAP') MAPSET('NEWCUST')
               INTO(PERSMAPI)
           END-EXEC.
           MOVE NAMEI TO WS-CUST-NAME.
           MOVE DOBI TO WS-CUST-DOB.

       SHOW-ADDRESS-SCREEN.
           SET STEP-ADDRESS TO TRUE.
           MOVE SPACES TO ADDRMAPO.
           EXEC CICS SEND MAP('ADDRMAP') MAPSET('NEWCUST')
               ERASE
           END-EXEC.

       RECEIVE-ADDRESS.
           EXEC CICS RECEIVE MAP('ADDRMAP') MAPSET('NEWCUST')
               INTO(ADDRMAPI)
           END-EXEC.
           MOVE ADDR1I TO WS-CUST-ADDR-1.
           MOVE ADDR2I TO WS-CUST-ADDR-2.
           MOVE POSTALI TO WS-CUST-POSTAL.

       SHOW-CONFIRM-SCREEN.
           SET STEP-CONFIRM TO TRUE.
           MOVE SPACES TO CONFMAPO.
      *> Echo everything gathered so far back to the user for
      *> review - this is the whole point of a confirmation step.
           MOVE WS-CUST-NAME TO CNAMEO OF CONFMAPO.
           MOVE WS-CUST-ADDR-1 TO CADDR1O OF CONFMAPO.
           MOVE WS-CUST-ADDR-2 TO CADDR2O OF CONFMAPO.
           EXEC CICS SEND MAP('CONFMAP') MAPSET('NEWCUST')
               ERASE
           END-EXEC.

       RECEIVE-CONFIRM.
           EXEC CICS RECEIVE MAP('CONFMAP') MAPSET('NEWCUST')
               INTO(CONFMAPI)
           END-EXEC.

       SAVE-AND-FINISH.
      *> This is the ONLY place in the whole wizard that touches
      *> the real customer master file - see step 635's rationale.
           PERFORM WRITE-NEW-CUSTOMER-RECORD.
           EXEC CICS SEND TEXT
               FROM('CUSTOMER CREATED SUCCESSFULLY')
               ERASE
           END-EXEC.
           EXEC CICS RETURN
           END-EXEC.

       CANCEL-WIZARD.
           EXEC CICS SEND TEXT
               FROM('CANCELLED - NO DATA WAS SAVED')
               ERASE
           END-EXEC.
           EXEC CICS RETURN
           END-EXEC.
```

### อธิบายจุดสำคัญ

- **`SAVE-AND-FINISH` และ `CANCEL-WIZARD` จบด้วย `EXEC CICS RETURN` ที่ไม่มี `TRANSID`** — นี่คือ
  ความแตกต่างสำคัญ: เมื่อการสนทนา "จบสมบูรณ์" (ไม่ว่าจะสำเร็จหรือถูกยกเลิก) โปรแกรมไม่ต้องการให้
  CICS เริ่ม transaction ใหม่โดยอัตโนมัติอีกต่อไป เทอร์มินัลจะกลับสู่สถานะว่างพร้อมให้ผู้ใช้พิมพ์
  transaction ใหม่เอง
- **การตรวจสอบ `EIBAID = DFHPF3` อยู่ก่อน `EVALUATE TRUE` ของ state machine เสมอ** — ทำให้ผู้ใช้
  ยกเลิกได้จากทุกขั้นตอน ไม่ว่าจะอยู่ที่หน้าจอไหนก็ตาม โดยไม่ต้องเขียนโค้ดตรวจสอบซ้ำในแต่ละ `WHEN`
- `DFHCOMMAREA PIC X(97)` ใน `LINKAGE SECTION` ต้องมีขนาด**เท่ากับหรือมากกว่า** `WS-COMMAREA`
  (1+20+8+30+30+5+5 = 99 ไบต์ ในตัวอย่างนี้ควรตรวจนับให้ตรงจริง ๆ ก่อนใช้งานจริงเสมอ)

### ข้อควรระวัง

- ต้องระวังเรื่องความยาว `DFHCOMMAREA` ที่ประกาศใน `LINKAGE SECTION` กับความยาวจริงที่ถูกส่งมา
  (`EIBCALEN`) — ถ้า COMMAREA จริงสั้นกว่าที่ประกาศไว้ (เช่น รุ่นเก่าของโปรแกรมส่ง COMMAREA สั้นกว่า)
  การอ้างอิงฟิลด์ท้าย ๆ อาจอ่านหน่วยความจำที่ไม่ได้เป็นของโปรแกรมจริง ควรตรวจสอบ `EIBCALEN` เทียบกับ
  ขนาดที่คาดหวังก่อนเสมอในระบบที่มีหลายเวอร์ชันทำงานร่วมกัน

### แบบฝึกหัดที่ 636.1

**โจทย์**: จงอธิบายว่าทำไมการตรวจสอบ `EIBAID = DFHPF3` (ปุ่มยกเลิก) จึงต้องอยู่**ก่อน**
`EVALUATE TRUE ... WHEN STEP-PERSONAL ...` ไม่ใช่อยู่**ภายใน**แต่ละ `WHEN`

**เฉลย**: ถ้าตรวจสอบ PF3 อยู่ภายในแต่ละ `WHEN` แยกกัน จะต้องเขียนโค้ดตรวจสอบซ้ำ 3 ครั้ง (ครั้งละ
ขั้นตอน) เพิ่มความเสี่ยงที่จะลืมใส่ในบาง `WHEN` (เช่น ลืมใส่ใน `STEP-CONFIRM` ทำให้ผู้ใช้กด PF3 ที่
หน้าจอยืนยันไม่ได้) การตรวจสอบก่อนเข้า `EVALUATE` ทำให้โค้ดสำหรับ "ยกเลิกจากทุกที่" เขียนเพียงครั้งเดียว
รับประกันว่าผู้ใช้ยกเลิกได้จากทุกขั้นตอนอย่างสม่ำเสมอ และยังทำให้โค้ดสั้นกระชับกว่าด้วย

---

## ขั้นตอนที่ 637: EXEC CICS RETURN เทียบกับ XCTL และ LINK

### สามวิธีที่โปรแกรม CICS ส่งการควบคุมไปยังโปรแกรมอื่น

นอกจาก `EXEC CICS RETURN` ที่เรียนมาตลอด Part นี้ ยังมีอีกสองคำสั่งที่ใช้ "โอนการควบคุม" ไปยังโปรแกรม
อื่นในโลก CICS ซึ่งมีพฤติกรรมต่างกันโดยสิ้นเชิง — เข้าใจความแตกต่างนี้สำคัญมากเพราะเลือกผิดจะทำให้
ออกแบบระบบผิดพลาดทั้งด้านทรัพยากรและด้านตรรกะ

| คำสั่ง | พฤติกรรม | เทียบเคียงได้กับ |
|---|---|---|
| `EXEC CICS RETURN` | จบ Task ปัจจุบันสมบูรณ์ คืนการควบคุมกลับไปที่ CICS (และอาจสั่งให้เริ่ม transaction ใหม่ผ่าน `TRANSID`) | `STOP RUN` ของโปรแกรม standalone |
| `EXEC CICS XCTL` | **โอน**การควบคุมไปยังโปรแกรมอื่นทันที **ภายใน Task เดียวกัน** โดยโปรแกรมปัจจุบันจะไม่ทำงานต่ออีกเลย (คล้าย "เปลี่ยนตัว" กลางเรื่อง) | หลักการคล้าย `exec()` ใน Unix ที่แทนที่โปรเซสปัจจุบันด้วยโปรแกรมใหม่ |
| `EXEC CICS LINK` | **เรียก**โปรแกรมอื่นเหมือน subroutine แล้ว**กลับมาทำงานต่อ**ที่โปรแกรมเดิมหลังโปรแกรมที่ถูกเรียกจบ | `CALL` statement ที่เรียนใน Part 031 |

### ตัวอย่างเปรียบเทียบ

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถ compile/รันได้ในสภาพแวดล้อมนี้**

```cobol
      *> XCTL: hand off completely to another program. The CALLING
      *> program's Working-Storage is GONE after this - there is
      *> no coming back to it. Commonly used to move from a menu
      *> program into the specific transaction the user picked.
           EXEC CICS XCTL
               PROGRAM('CUSTINQ')
               COMMAREA(WS-COMMAREA)
               LENGTH(LENGTH OF WS-COMMAREA)
           END-EXEC.

      *> LINK: call another program as a subroutine and return
      *> here afterward. Same idea as CALL (Part 031), but crossing
      *> into a separately-compiled CICS program rather than a
      *> subprogram linked into the same load module.
           EXEC CICS LINK
               PROGRAM('VALIDATE-CUST-ID')
               COMMAREA(WS-VALIDATION-AREA)
               LENGTH(LENGTH OF WS-VALIDATION-AREA)
           END-EXEC.
      *> Execution resumes HERE once VALIDATE-CUST-ID returns.
           IF WS-VALIDATION-RESULT = 'INVALID'
               PERFORM SHOW-ERROR-MESSAGE
           END-IF.
```

### เมื่อไรควรใช้อะไร

- ใช้ **`RETURN` + `TRANSID`** เมื่อ Task ต้องจบเพื่อรอผู้ใช้ตอบกลับ (หัวใจของ pseudo-conversational
  ที่เรียนมาตลอด Part นี้)
- ใช้ **`XCTL`** เมื่อต้องการเปลี่ยนไปทำงานกับโปรแกรมอื่นอย่างถาวรภายใน Task เดียวกัน (เช่น จากเมนู
  หลักไปสู่โปรแกรมที่ผู้ใช้เลือก) โดยไม่ต้องการกลับมาที่โปรแกรมเดิมอีก — ประหยัดหน่วยความจำเพราะ
  โปรแกรมเดิมถูกปลดออกจากหน่วยความจำทันที
- ใช้ **`LINK`** เมื่อต้องการเรียกใช้ตรรกะที่ใช้ร่วมกันหลายที่ (เช่น โปรแกรมตรวจสอบความถูกต้องของ
  รหัสลูกค้าที่ถูกเรียกจากหลายจุดในระบบ) แล้วต้องการผลลัพธ์กลับมาทำงานต่อ

### ข้อควรระวัง

- **`XCTL` ไม่ใช่ `RETURN`** — Task ยังคงทำงานต่อเนื่องอยู่ (ไม่ได้จบและคืนทรัพยากร) เพียงแค่เปลี่ยน
  ว่ากำลังรันโค้ดของโปรแกรมไหนอยู่เท่านั้น ถ้าใช้ `XCTL` แทน `RETURN` โดยเข้าใจผิดว่าเป็นการ "จบ" การ
  สนทนา จะทำให้ Task ยังคงค้างรอ (ถ้าโปรแกรมใหม่ทำงานแบบ Conversational) กลับไปเจอปัญหาแบบขั้นตอนที่
  631 อีกครั้ง
- **`LINK` ซ้อนกันหลายชั้นเกินไปจะกินทรัพยากร Stack ของ CICS** เหมือนกับการเรียก subprogram ซ้อนกัน
  ลึกเกินไปในโปรแกรมทั่วไป (Part 031/034) ควรออกแบบให้มีความลึกของการ `LINK` ที่สมเหตุสมผล

### แบบฝึกหัดที่ 637.1

**โจทย์**: ระบบมีโปรแกรมเมนูหลัก (`MAINMENU`) ที่ให้ผู้ใช้เลือกว่าจะเข้าโปรแกรมค้นหาลูกค้า
(`CUSTINQ`) หรือโปรแกรมเปิดบัญชีใหม่ (`NEWCUST`) จงอธิบายว่าควรใช้ `XCTL` หรือ `LINK` ในการส่งต่อ
การควบคุมจากเมนูไปยังโปรแกรมที่ผู้ใช้เลือก พร้อมเหตุผล

**เฉลย**: ควรใช้ **`XCTL`** เพราะเมื่อผู้ใช้เลือกเมนูแล้ว ไม่มีความจำเป็นต้อง "กลับมา" ทำงานต่อที่
โปรแกรมเมนูหลักอีก (การกลับสู่เมนูหลักในภายหลังมักทำผ่านการเริ่ม transaction ของเมนูใหม่ ไม่ใช่การ
`RETURN` จากโปรแกรมย่อยกลับมา) การใช้ `XCTL` ทำให้โปรแกรมเมนูหลักถูกปลดออกจากหน่วยความจำทันทีที่
โอนการควบคุมไป ประหยัดทรัพยากรมากกว่าการใช้ `LINK` ซึ่งจะทำให้โปรแกรมเมนูหลักยังคงค้างอยู่ในหน่วยความ
จำรอการกลับมาโดยไม่มีประโยชน์ใด ๆ

---

## ขั้นตอนที่ 638: ข้อผิดพลาดที่พบบ่อยและข้อจำกัดของ Pseudo-conversational

### กับดักที่โปรแกรมเมอร์มือใหม่เจอบ่อยที่สุด

1. **ขนาด COMMAREA เกินขีดจำกัด** — ถ้าพยายามยัดข้อมูลจำนวนมากเข้า COMMAREA (เช่น ตารางข้อมูลทั้งหมด
   ของหน้าจอ grid ที่มี 100 แถว) จะเกิด error `LENGERR` เมื่อพยายาม `RETURN` พร้อม COMMAREA ที่ใหญ่
   เกินกว่าที่ระบบกำหนด วิธีแก้คือเก็บเฉพาะ key ที่จำเป็น แล้วอ่านข้อมูลจริงใหม่จากฐานข้อมูลทุกครั้ง
   (ทบทวนหลักการจากขั้นตอนที่ 633)
2. **ลืมว่า Task ใหม่ "ไม่มีความทรงจำ" นอกเหนือจาก COMMAREA** — ตัวแปรใด ๆ ใน `WORKING-STORAGE` ที่
   ไม่ได้ถูกใส่ไว้ใน COMMAREA จะกลับไปเป็นค่า `VALUE` เริ่มต้นเสมอในทุก Task ใหม่ ผู้เริ่มต้นมักลืม
   จุดนี้และคาดหวังว่าตัวแปรจะ "จำ" ค่าไว้เหมือนโปรแกรม batch ทั่วไป
3. **Task Timeout**: ถ้าผู้ใช้ปล่อยหน้าจอทิ้งไว้นานเกินไปโดยไม่ตอบกลับ (เช่น ลืมทำงานต่อทั้งวัน)
   CICS มักตั้งค่า **DTIMOUT (Deferred Terminal Timeout)** ให้ตัดการเชื่อมต่อ pseudo-conversational
   ที่ค้างนานเกินกำหนดโดยอัตโนมัติ (ค่านี้ตั้งใน Transaction Definition) เพื่อป้องกันไม่ให้เทอร์มินัล
   ที่ไม่มีคนใช้แล้วครอบครองทรัพยากรค้างไว้ (แม้ pseudo-conversational จะคืนทรัพยากรของ Task แล้วก็
   ตาม แต่ยังมีความสัมพันธ์ระหว่างเทอร์มินัลกับ transaction ที่ค้างรอ ซึ่งต้องจัดการเช่นกัน)
4. **ไม่จัดการ `EIBAID` ให้ครบทุกปุ่มที่เป็นไปได้** — ทบทวนจาก Part 062: `EIBAID` บอกว่าผู้ใช้กดปุ่ม
   ไหน (Enter, PF1-PF24, Clear) ถ้าโปรแกรมตรวจสอบแค่บางปุ่มแล้วไม่มี `WHEN OTHER` ดักไว้ ผู้ใช้ที่กด
   ปุ่มที่ไม่คาดคิด (เช่น PF7 ที่โปรแกรมไม่รองรับ) อาจทำให้ตรรกะทำงานผิดพลาด

### ตารางสรุปข้อผิดพลาดและวิธีป้องกัน

| ข้อผิดพลาด | อาการที่พบ | วิธีป้องกัน |
|---|---|---|
| COMMAREA ใหญ่เกินไป | `EIBRESP` = `LENGERR` ตอน RETURN | เก็บแค่ key/state จำเป็น อ่านข้อมูลเต็มใหม่ทุกครั้ง |
| ลืมตรวจ `EIBCALEN = 0` | Task แรกอ่านค่าขยะจาก DFHCOMMAREA | ตรวจสอบเป็นคำสั่งแรกสุดเสมอ |
| ลืมอัปเดต state indicator | State machine ค้างวนซ้ำขั้นตอนเดิม | อัปเดตก่อน RETURN ทุกเส้นทางเสมอ |
| ไม่จัดการปุ่มที่ไม่คาดคิด | ตรรกะพังเมื่อผู้ใช้กดปุ่มแปลก | มี `WHEN OTHER`/default case ดักเสมอ |
| Task ค้างนานเกินไป (ผู้ใช้ไม่ตอบ) | เทอร์มินัลถูกตัดโดย DTIMOUT | ตั้งค่า DTIMOUT ให้เหมาะสมกับลักษณะงาน |

### ข้อควรระวัง

- อย่าพยายาม "แก้ปัญหา" ขนาด COMMAREA ด้วยการใช้ **TSQ (Temporary Storage Queue)** หรือ
  **TDQ (Transient Data Queue)** แทนโดยไม่เข้าใจ trade-off — สิ่งเหล่านี้เป็นกลไกเก็บข้อมูลชั่วคราว
  ที่ใหญ่กว่า COMMAREA ได้ แต่มีค่าใช้จ่ายด้าน I/O เพิ่มเติม (เนื้อหาขั้นสูงเหล่านี้อยู่นอกขอบเขตของ
  หลักสูตรนี้ แต่ควรรู้จักชื่อไว้)
- การทดสอบโปรแกรม pseudo-conversational ด้วยการรันครั้งเดียวจบ (แบบ Conversational ปลอม ๆ ในเครื่อง
  ทดสอบ) อาจไม่จับปัญหาเรื่อง state machine ได้ครบ ควรทดสอบด้วยการ "จำลอง" การกด Enter หลายรอบจริง ๆ
  ผ่านเทอร์มินัลจำลอง (3270 emulator) เสมอ

### แบบฝึกหัดที่ 638.1

**โจทย์**: จงอธิบายว่าทำไมการตั้งค่า `DTIMOUT` (ระยะเวลาก่อนตัดการเชื่อมต่อเทอร์มินัลที่ไม่มีการตอบกลับ)
จึงยังจำเป็น แม้ pseudo-conversational จะคืนทรัพยากรของ Task ไปแล้วตั้งแต่ส่งหน้าจอออกไป

**เฉลย**: แม้ Task จะจบและคืนทรัพยากรของระบบ (หน่วยความจำ, task control block) ไปแล้ว แต่ CICS ยังคง
ต้อง "จดจำ" ว่าเทอร์มินัลนี้กำลังรอ transaction ใดอยู่ (ผ่านข้อมูลที่ผูกกับเทอร์มินัล เพื่อให้รู้ว่า
ต้องเริ่ม transaction ไหนเมื่อผู้ใช้ตอบกลับ) ถ้าผู้ใช้ไม่ตอบกลับเลยเป็นเวลานานมาก (เช่น ปิดคอมพิวเตอร์
ไปโดยไม่ log off) ความสัมพันธ์นี้จะค้างอยู่โดยไม่มีประโยชน์ `DTIMOUT` จึงทำหน้าที่เป็นตัวจำกัดเวลาที่
ระบบจะรอเทอร์มินัลที่ไม่ตอบสนองอีกต่อไป และยกเลิกความสัมพันธ์ที่ค้างอยู่นั้นโดยอัตโนมัติ เพื่อรักษา
ความสะอาดของทรัพยากรระบบในระยะยาว

---

## ขั้นตอนที่ 639: เปรียบเทียบเชิงแนวคิดกับสถาปัตยกรรม Stateless ของเว็บสมัยใหม่

### ทำไมแนวคิดที่ดูเก่าแก่นี้กลับมี "ญาติ" ในโลกเว็บสมัยใหม่

นักเรียนหลายคนที่มาถึงจุดนี้อาจรู้สึกว่า Pseudo-conversational เป็นแนวคิดที่ซับซ้อนและแปลกประหลาด
แต่ความจริงที่น่าทึ่งคือ **หลักการเดียวกันนี้คือรากฐานของสถาปัตยกรรมเว็บสมัยใหม่ทั้งหมด** ที่หลักสูตรนี้
จะสอนเพิ่มเติมใน Part 073 (REST API) เมื่อถึงเฟส 5

### ตารางเปรียบเทียบ

| แง่มุม | CICS Pseudo-conversational | HTTP/REST API สมัยใหม่ |
|---|---|---|
| การ "จบ" หลังตอบสนอง | `EXEC CICS RETURN` จบ Task ทันทีหลังส่งจอ | Web server ปิด connection/จบ request ทันทีหลังส่ง response |
| การเก็บสถานะระหว่างการร้องขอ | `COMMAREA` ส่งสถานะไปกับ Task ถัดไป | Session ID / JWT Token / Cookie ส่งสถานะไปกับ request ถัดไป |
| ทรัพยากรระหว่างรอผู้ใช้ | Task ไม่ถูกจองไว้เลยระหว่างรอผู้ใช้ตอบกลับ | Server thread/connection ไม่ถูกจองไว้เลยระหว่างรอผู้ใช้คลิก |
| ประโยชน์หลัก | รองรับผู้ใช้เทอร์มินัลหลายพันคนพร้อมกันด้วยทรัพยากรจำกัด | รองรับผู้ใช้เว็บหลายล้านคนพร้อมกันด้วยทรัพยากรจำกัด |
| ชื่อเรียกแนวคิดนี้ในแต่ละโลก | "Pseudo-conversational" | "Stateless" (ไร้สถานะ) |

### บทเรียนที่ลึกซึ้งกว่านั้น

สิ่งที่ CICS ค้นพบในยุค 1970 (ก่อนอินเทอร์เน็ตจะถือกำเนิดด้วยซ้ำ) คือหลักการที่ทุกระบบขนาดใหญ่ที่ต้อง
รองรับผู้ใช้จำนวนมากพร้อมกันต้องยึดถือ: **อย่าผูกทรัพยากรระบบไว้กับสิ่งที่ควบคุมไม่ได้ (พฤติกรรมมนุษย์)**
เซิร์ฟเวอร์เว็บสมัยใหม่ที่ใช้ Node.js, Python (Flask/Django), หรือ Java (Spring Boot) ล้วนออกแบบมา
บนหลักการเดียวกัน: แต่ละ HTTP request ประมวลผลเสร็จแล้ว "จบ" ทันที ไม่มี thread ใดถูกจองรอผู้ใช้
คลิกปุ่มถัดไป — สถานะทั้งหมดที่ต้องจำถูกเก็บไว้ที่อื่น (ฐานข้อมูล, Redis, JWT Token ที่ฝั่ง client
ถืออยู่) แล้วส่งกลับมาพร้อมกับ request ถัดไป เหมือนกับที่ `COMMAREA` ทำหน้าที่นี้ให้กับ CICS

การเข้าใจ Pseudo-conversational อย่างถ่องแท้ในวันนี้ จึงไม่ใช่แค่การเรียนรู้เทคโนโลยีเก่าเพื่อดูแล
ระบบ Legacy เท่านั้น แต่ยังเป็นการปูพื้นฐานความเข้าใจเรื่อง **Stateless Architecture** ที่จำเป็นต่อการ
ออกแบบระบบ Modernization ใน Part 073–077 ของหลักสูตรนี้ด้วย

### ข้อควรระวัง

- อย่าสรุปว่า COMMAREA "เหมือนกันเป๊ะ" กับ Session/Cookie ของเว็บ — มีความแตกต่างสำคัญ เช่น COMMAREA
  ผูกกับเทอร์มินัลเฉพาะเครื่องโดยตรงผ่านกลไกภายในของ CICS ในขณะที่ Session ของเว็บมักอาศัย
  ตัวระบุ (session ID) ที่ส่งไปมาทาง HTTP header/cookie อย่างชัดเจน แต่ **หลักการพื้นฐาน** (ไม่ผูก
  ทรัพยากรไว้รอผู้ใช้, ส่งสถานะไปมาแทนการจำในหน่วยความจำ) เหมือนกัน

### แบบฝึกหัดที่ 639.1

**โจทย์**: จงอธิบายด้วยคำพูดของตัวเองว่าทำไม "การไม่ผูกทรัพยากรไว้รอผู้ใช้" จึงเป็นหลักการสำคัญร่วมกัน
ระหว่าง CICS ยุค 1970 กับเว็บเซิร์ฟเวอร์สมัยใหม่ ทั้งที่เทคโนโลยีทั้งสองห่างกันหลายสิบปี

**เฉลยแนวทาง**: เพราะปัญหาพื้นฐานที่ทั้งสองระบบต้องเผชิญเหมือนกันทุกประการ คือ "มีผู้ใช้จำนวนมากที่ใช้
เวลาไม่แน่นอนในการตัดสินใจ/พิมพ์ข้อมูล แต่ทรัพยากรของเซิร์ฟเวอร์ (CPU, หน่วยความจำ, การเชื่อมต่อ) มี
จำกัดเสมอ" ถ้าออกแบบให้ทรัพยากรถูกจองไว้ตลอดเวลาที่รอผู้ใช้ (ไม่ว่าจะเป็น CICS Task หรือ web server
thread) ระบบจะรองรับผู้ใช้พร้อมกันได้น้อยมาก เพราะทรัพยากรจะถูกใช้ไปกับการ "รอเฉย ๆ" แทนที่จะเป็นการ
"ประมวลผลจริง" หลักการ "จบการทำงานทันทีที่ส่งคำตอบ แล้วเก็บสถานะไว้ที่อื่นเพื่อดึงกลับมาใช้เมื่อจำเป็น"
จึงเป็นคำตอบที่สมเหตุสมผลที่สุดสำหรับปัญหาประเภทนี้ ไม่ว่าเทคโนโลยีรอบข้างจะเปลี่ยนไปแค่ไหนก็ตาม

---

## ขั้นตอนที่ 640: ตัวอย่างบูรณาการ — รวม BMS (Part 063) เข้ากับ Pseudo-conversational (Part นี้)

### ภาพรวมสุดท้าย: โปรแกรม CICS ฉบับสมบูรณ์

มาปิดท้าย Part นี้ด้วยการรวมทุกความรู้จาก Part 063 (BMS Maps) เข้ากับ Part 064 (Pseudo-conversational)
เป็นโปรแกรมเดียวที่สมบูรณ์ที่สุดเท่าที่หลักสูตรนี้จะแสดงได้ — โปรแกรมสอบถามและแก้ไขยอดเงินลูกค้า
ที่มี 2 หน้าจอทำงานต่อเนื่องกัน

> ⚠️ **REFERENCE SYNTAX — ไม่สามารถ compile/รันได้ในสภาพแวดล้อมนี้ ผสานความรู้จาก Part 063 ทั้งหมด**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CUSTUPD.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
      *> Symbolic maps from Part 063's BMS design (two screens:
      *> inquiry and update-confirmation), COPYd in as usual.
           COPY CUSTMSET.
           COPY UPDMSET.

       01  WS-COMMAREA.
           05  WS-CURRENT-STEP      PIC 9(1).
               88  STEP-SHOW-INQUIRY    VALUE 1.
               88  STEP-SHOW-UPDATE     VALUE 2.
           05  WS-CUST-ID           PIC 9(5).
           05  WS-CUST-NAME         PIC X(20).
           05  WS-OLD-BALANCE       PIC 9(9)V99.

       01  WS-NEW-BALANCE           PIC 9(9)V99.
       01  WS-DISPLAY-BAL           PIC ZZZ,ZZZ,ZZ9.99.

       LINKAGE SECTION.
       01  DFHCOMMAREA              PIC X(39).

       PROCEDURE DIVISION.
       MAIN-PARA.
           IF EIBCALEN = 0
               PERFORM SHOW-INQUIRY-SCREEN
           ELSE
               MOVE DFHCOMMAREA TO WS-COMMAREA
               EVALUATE TRUE
                   WHEN STEP-SHOW-INQUIRY
                       PERFORM RECEIVE-INQUIRY-AND-LOOKUP
                   WHEN STEP-SHOW-UPDATE
                       PERFORM RECEIVE-UPDATE-AND-SAVE
               END-EVALUATE
           END-IF.
           EXEC CICS RETURN
               TRANSID('CUPD')
               COMMAREA(WS-COMMAREA)
               LENGTH(LENGTH OF WS-COMMAREA)
           END-EXEC.

      *> ---- SCREEN 1: customer ID inquiry (BMS map CUSTINQ1) ----
       SHOW-INQUIRY-SCREEN.
           SET STEP-SHOW-INQUIRY TO TRUE.
           MOVE SPACES TO CUSTINQ1O.
           EXEC CICS SEND MAP('CUSTINQ1') MAPSET('CUSTMSET')
               ERASE
           END-EXEC.

       RECEIVE-INQUIRY-AND-LOOKUP.
           EXEC CICS RECEIVE MAP('CUSTINQ1') MAPSET('CUSTMSET')
               INTO(CUSTINQ1I)
           END-EXEC.
           IF CUSTIDL > 0
               MOVE CUSTIDI TO WS-CUST-ID
               PERFORM LOOKUP-CUSTOMER
               PERFORM SHOW-UPDATE-SCREEN
           ELSE
               MOVE SPACES TO MSGO
               STRING 'ENTER A CUSTOMER ID' DELIMITED BY SIZE
                   INTO MSGO
               EXEC CICS SEND MAP('CUSTINQ1') MAPSET('CUSTMSET')
                   DATAONLY
               END-EXEC
           END-IF.

       LOOKUP-CUSTOMER.
      *> Real system: VSAM READ (Part 055) keyed by WS-CUST-ID.
           MOVE "SOMCHAI JAIDEE" TO WS-CUST-NAME.
           MOVE 5000.00 TO WS-OLD-BALANCE.

      *> ---- SCREEN 2: confirm and enter new balance (BMS UPDMAP) --
       SHOW-UPDATE-SCREEN.
           SET STEP-SHOW-UPDATE TO TRUE.
           MOVE SPACES TO UPDMAPO.
           MOVE WS-CUST-NAME TO UNAMEO.
           MOVE WS-OLD-BALANCE TO UOLDBALO.
           EXEC CICS SEND MAP('UPDMAP') MAPSET('UPDMSET')
               ERASE
           END-EXEC.

       RECEIVE-UPDATE-AND-SAVE.
           EXEC CICS RECEIVE MAP('UPDMAP') MAPSET('UPDMSET')
               INTO(UPDMAPI)
           END-EXEC.
           IF UNEWBALL > 0
               MOVE UNEWBALI TO WS-NEW-BALANCE
      *> Real system: VSAM REWRITE (Part 055/056) keyed by
      *> WS-CUST-ID, replacing WS-OLD-BALANCE with WS-NEW-BALANCE.
               MOVE SPACES TO UMSGO
               STRING 'BALANCE UPDATED SUCCESSFULLY'
                   DELIMITED BY SIZE INTO UMSGO
               EXEC CICS SEND MAP('UPDMAP') MAPSET('UPDMSET')
                   DATAONLY FREEKB
               END-EXEC
               EXEC CICS RETURN
               END-EXEC
           ELSE
               MOVE SPACES TO UMSGO
               STRING 'ENTER A NEW BALANCE' DELIMITED BY SIZE
                   INTO UMSGO
               EXEC CICS SEND MAP('UPDMAP') MAPSET('UPDMSET')
                   DATAONLY
               END-EXEC
           END-IF.
```

### อธิบายภาพรวม

- โปรแกรมนี้เป็น **transaction เดียว (`CUPD`) ที่ครอบคลุมทั้ง 2 หน้าจอ** ตามหลักการจากขั้นตอนที่ 634
- `WS-COMMAREA` พก `WS-CUST-ID`, `WS-CUST-NAME`, และ `WS-OLD-BALANCE` ข้ามจากหน้าจอที่ 1 ไปยังหน้าจอ
  ที่ 2 — ทำให้หน้าจอที่ 2 ไม่ต้องค้นหาลูกค้าซ้ำอีกครั้ง (แม้จะเป็นคนละ Task กันโดยสิ้นเชิง)
- เมื่อบันทึกสำเร็จ (`RECEIVE-UPDATE-AND-SAVE` กรณี `UNEWBALL > 0`) โปรแกรมจบด้วย
  `EXEC CICS RETURN` **แบบไม่มี `TRANSID`** เพราะการสนทนาจบสมบูรณ์แล้ว
- ถ้าผู้ใช้ยังไม่กรอกยอดใหม่ (`UNEWBALL` ไม่มากกว่า 0 ทบทวนความหมายค่า -1 จากขั้นตอนที่ 626) โปรแกรม
  จะส่งจอเดิมกลับไปพร้อมข้อความเตือน และ **ต้องคง `TRANSID('CUPD')` ไว้ที่ `EXEC CICS RETURN` ท้าย
  `MAIN-PARA`** เพื่อให้วนกลับมาที่ `STEP-SHOW-UPDATE` อีกครั้งเมื่อผู้ใช้ตอบกลับ

### ข้อควรระวัง

- สังเกตว่า `RECEIVE-UPDATE-AND-SAVE` มี `EXEC CICS RETURN` อยู่ **ภายใน** paragraph ของมันเองในกรณี
  สำเร็จ (ไม่รอให้ไปถึง `EXEC CICS RETURN` ที่ท้าย `MAIN-PARA`) นี่เป็นเทคนิคที่ต้องระวังเรื่อง flow
  ให้ดี เพราะถ้า `EXEC CICS RETURN` ถูกเรียกจากกลาง paragraph โค้ดหลังจากนั้นในโปรแกรมจะไม่ทำงานอีก
  เลย (เทียบเท่า `STOP RUN` ทบทวนจาก Part 012) ในทางปฏิบัติจริงหลายทีมเลือกให้มีจุด
  `EXEC CICS RETURN` เดียวที่ท้ายโปรแกรมเสมอ (ตั้งค่าตัวแปร flag แล้วให้ logic หลักตรวจสอบว่าจะใส่
  `TRANSID` หรือไม่) เพื่อลดความสับสนเรื่อง control flow แบบนี้

### แบบฝึกหัดที่ 640.1

**โจทย์**: จงอธิบายภาพรวมทั้งหมดของ Part นี้เป็นข้อความสั้น ๆ 3-4 ประโยค โดยครอบคลุม: ปัญหาที่
Pseudo-conversational แก้, กลไกหลักที่ใช้แก้ปัญหานั้น, และความเชื่อมโยงกับสถาปัตยกรรมสมัยใหม่

**เฉลยแนวทาง**: Conversational programming ทำให้ CICS Task ถูกจับจองทรัพยากรไว้ตลอดเวลาที่รอผู้ใช้
พิมพ์ข้อมูล ซึ่งไม่สามารถรองรับผู้ใช้จำนวนมากพร้อมกันได้ Pseudo-conversational แก้ปัญหานี้โดยให้ Task
จบสมบูรณ์ทันทีหลังส่งหน้าจอ (ผ่าน `EXEC CICS RETURN TRANSID(...)`) แล้วใช้ `COMMAREA` เป็นกลไกส่ง
สถานะที่จำเป็นไปยัง Task ใหม่ที่ CICS จะสร้างขึ้นเมื่อผู้ใช้ตอบกลับ ทำให้ทรัพยากรของระบบไม่ถูกจองไว้
เลยระหว่างที่ผู้ใช้กำลัง "คิด" หลักการนี้เป็นต้นแบบทางความคิดของสถาปัตยกรรม Stateless ที่เว็บเซิร์ฟเวอร์
สมัยใหม่ทุกตัวใช้อยู่ในปัจจุบัน แสดงให้เห็นว่าปัญหาพื้นฐานเรื่อง scalability นั้นไม่เปลี่ยนแปลงไปตาม
ยุคสมัย แม้เทคโนโลยีจะเปลี่ยนไปมากแค่ไหนก็ตาม

---

## สรุปท้ายบท

Part นี้พาคุณเจาะลึกแนวคิดที่ท้าทายและสำคัญที่สุดอย่างหนึ่งของการเขียนโปรแกรม CICS:

- ปัญหาของ **Conversational Programming** ที่ผูกทรัพยากรระบบไว้กับ Task ตลอดเวลาที่รอผู้ใช้ตอบกลับ
- แนวคิด **Pseudo-conversational** ที่ให้ Task จบทันทีหลังส่งหน้าจอ ผ่าน
  `EXEC CICS RETURN TRANSID(...)`
- **COMMAREA** และ **EIBCALEN** กลไกที่ทำให้ Task ใหม่ "จำ" สถานะจาก Task ก่อนหน้าได้
- โครงสร้าง `PROCEDURE DIVISION` มาตรฐานสำหรับโปรแกรม pseudo-conversational และการออกแบบ State
  Machine หลายขั้นตอนด้วยตัวแปรสถานะใน COMMAREA
- ความแตกต่างระหว่าง `RETURN`, `XCTL`, และ `LINK`
- ข้อผิดพลาดที่พบบ่อยและวิธีป้องกัน
- ความเชื่อมโยงเชิงแนวคิดกับสถาปัตยกรรม **Stateless** ของเว็บสมัยใหม่ ที่จะเรียนเพิ่มเติมใน Part 073

ความรู้เรื่อง CICS ทั้ง Part 061–064 ที่ผ่านมาเป็นรากฐานสำคัญของโลก Mainframe Transaction Processing
ใน Part 065 เราจะเปลี่ยนไปสำรวจเทคโนโลยี Mainframe อีกแขนงหนึ่งที่เก่าแก่ไม่แพ้กัน แต่แก้ปัญหาคนละ
มุม: **IMS (Information Management System)** ฐานข้อมูลเชิงลำดับชั้น (Hierarchical Database) และ
ระบบจัดการธุรกรรมของ IBM ที่ยังคงใช้งานอยู่จริงในสถาบันการเงินขนาดใหญ่หลายแห่งทั่วโลกจนถึงทุกวันนี้

**[ไปยัง Part 065: IMS DB/DC เบื้องต้น →](part-065-ims-intro.md)**
