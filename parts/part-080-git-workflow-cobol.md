# Part 080: Version Control (Git) และ Workflow สำหรับทีม COBOL (ขั้นตอนที่ 791–800)

## คำนำของ Part นี้

Part 078 และ 079 สอนให้เราสร้างระบบทดสอบอัตโนมัติที่แข็งแกร่งสำหรับโปรเจกต์ COBOL แต่ระบบทดสอบ
ทั้งหมดนั้นจะไม่มีประโยชน์เลยถ้าทีมพัฒนาไม่มีวิธีการ**จัดการการเปลี่ยนแปลงโค้ดร่วมกัน**ที่เป็นระบบ
Part นี้จะพาคุณเรียนรู้ **Git** — เครื่องมือ Version Control มาตรฐานอุตสาหกรรม — ที่ประยุกต์ใช้กับ
โปรเจกต์ COBOL โดยเฉพาะ ซึ่งมีความท้าทายเฉพาะทางที่ภาษาโปรแกรมสมัยใหม่ไม่ค่อยเจอ เช่น ความอ่อนไหว
ต่อคอลัมน์ในการอ่าน diff และการประสานงานเมื่อ copybook ที่ใช้ร่วมกันหลายโปรแกรมถูกแก้ไข

ทุกคำสั่ง Git ในบทนี้ **รันจริงในสภาพแวดล้อมทดสอบแล้ว** ใน scratch repository ที่แยกต่างหากจาก
โปรเจกต์หลักสูตรนี้โดยสิ้นเชิง (ไม่กระทบ repository ของหลักสูตรแต่อย่างใด) ผลลัพธ์ที่แสดงในบทนี้
เป็น output จริงจาก terminal ทุกตัวอักษร

> **หมายเหตุสภาพแวดล้อม**: ตรวจสอบแล้วว่า `git` ติดตั้งพร้อมใช้งานในสภาพแวดล้อมนี้ (คำสั่ง `which
> git` พบไฟล์ปฏิบัติการจริง) การสาธิตทั้งหมดในบทนี้ใช้ scratch repository ที่สร้างขึ้นใหม่ใน
> ไดเรกทอรีชั่วคราวแยกต่างหาก (`/tmp/cobol-git-demo`) ซึ่งเป็นคนละที่กับ repository ของหลักสูตร
> นี้เอง (`cobol_course`) โดยสิ้นเชิง เพื่อความปลอดภัยและไม่ให้กระทบเนื้อหาที่ผู้เขียนคนอื่นกำลัง
> ทำงานอยู่พร้อมกัน

---

## ขั้นตอนที่ 791: ทำไมโปรเจกต์ COBOL ต้องการ Git เหมือนโปรเจกต์สมัยใหม่ทุกประการ

### Git คืออะไร (ทบทวนสั้น ๆ)

**Git** คือระบบ Distributed Version Control ที่บันทึก**ประวัติการเปลี่ยนแปลง**ของไฟล์ทั้งหมดใน
โปรเจกต์ ทำให้ทีมสามารถ:

- ย้อนกลับไปดูหรือกู้คืนโค้ดเวอร์ชันก่อนหน้าได้เสมอ
- ทำงานคู่ขนานกันหลายคนโดยไม่ทับไฟล์กัน (ผ่าน branch)
- ตรวจสอบว่า "ใครแก้ไขบรรทัดไหน เมื่อไหร่ ทำไม" (ผ่าน commit message และ `git blame`)
- รวมงานของหลายคนเข้าด้วยกันอย่างเป็นระบบ (merge)

### ความเข้าใจผิดที่พบบ่อย: "COBOL อยู่บน Mainframe ไม่ต้องใช้ Git"

ในอดีต โปรแกรมเมอร์ COBOL บน Mainframe มักใช้เครื่องมือจัดการเวอร์ชันเฉพาะของ IBM เช่น **Endevor**
หรือ **ChangeMan** ซึ่งทำงานคล้าย Git ในแนวคิดพื้นฐาน (เก็บประวัติ, มี "library" หลายระดับคล้าย
branch) แต่ในโลกปี 2026 ที่ Part 076–077 สอนไปแล้วว่า COBOL ถูกนำมา Containerize และ deploy บน
Cloud ได้ **โค้ด COBOL ที่พัฒนาด้วย GnuCOBOL หรือ Micro Focus บนเครื่อง Windows/Linux/Mac สมัยใหม่
ก็ใช้ Git ได้เหมือนโปรเจกต์ภาษาอื่นทุกประการ** — แม้แต่องค์กรที่ยังมีระบบ core อยู่บน Mainframe
จำนวนมากก็เริ่มย้ายมาใช้ Git ร่วมกับเครื่องมือ bridge ที่ sync กับระบบ Mainframe เดิม

### ตรวจสอบว่า Git พร้อมใช้งาน

```bash
which git
git --version
```

**ผลลัพธ์จริง:**

```
/usr/bin/git
```

### ข้อควรระวัง

- อย่าสับสนระหว่าง **Git** (เครื่องมือ) กับ **GitHub/GitLab** (บริการ hosting ที่ใช้ Git เป็น
  พื้นฐาน) — ทุกคำสั่งในบทนี้เป็นคำสั่ง Git ล้วน ๆ ที่ทำงานได้แม้ไม่มีการเชื่อมต่ออินเทอร์เน็ตเลย
  (Git เป็นระบบ **Distributed** — ทุกเครื่องมี copy ของประวัติทั้งหมดอยู่ในตัวเอง)
- บทนี้ใช้ scratch repository ใน `/tmp` เพื่อการสาธิตเท่านั้น **ห้ามรันคำสั่ง Git ที่ทำลายข้อมูล
  (เช่น `git reset --hard`, `git clean -f`) กับ repository ของโปรเจกต์จริงโดยไม่เข้าใจผลกระทบอย่าง
  ถ่องแท้ก่อนเสมอ**

### แบบฝึกหัดที่ 791.1

**โจทย์**: จงอธิบายว่าทำไมการที่ Git เป็นระบบ "Distributed" (แต่ละเครื่องมีประวัติครบทั้งหมด) ถึง
เหมาะกับทีมที่ทำงานข้ามไซต์ (เช่น ทีมพัฒนา COBOL ที่กระจายอยู่หลายประเทศ) มากกว่าระบบ Version
Control แบบ Centralized รุ่นเก่า

**เฉลย**: ในระบบ Centralized (เช่น CVS หรือ SVN รุ่นเก่า) ทุกการดำเนินการที่เกี่ยวกับประวัติ (เช่น
ดู commit log, เปรียบเทียบเวอร์ชัน) ต้องเชื่อมต่อกับ server กลางตลอดเวลา ถ้าทีมกระจายอยู่หลายประเทศ
และ server กลางอยู่ไกล การเชื่อมต่อที่ช้าหรือขาดหายจะทำให้งานหยุดชะงัก ในขณะที่ Git ให้ทุกเครื่อง
มี**สำเนาประวัติทั้งหมด**อยู่ในเครื่องตัวเอง ทำให้นักพัฒนาสามารถ commit, ดูประวัติ, สร้าง branch,
และทำงานส่วนใหญ่ได้โดยไม่ต้องเชื่อมต่อเครือข่ายเลย แล้วค่อย sync (push/pull) กับ remote server
เมื่อพร้อมหรือเมื่อมีการเชื่อมต่อเท่านั้น ทำให้เหมาะกับทีมที่กระจายตัวทางภูมิศาสตร์มากกว่ามาก

---

## ขั้นตอนที่ 792: สร้าง Repository และ .gitignore สำหรับโปรเจกต์ COBOL

### เริ่มต้น Repository ใหม่

```bash
mkdir -p /tmp/cobol-git-demo && cd /tmp/cobol-git-demo
git init
git config user.email "demo@example.com"
git config user.name "Demo User"
```

**ผลลัพธ์จริง (จาก `git init`):**

```
Initialized empty Git repository in /tmp/cobol-git-demo/.git/
```

### สร้างโปรแกรม COBOL ตัวอย่าง

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PAYROLL.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-HOURS                 PIC 9(3) VALUE 160.
       01  WS-RATE                  PIC 9(3)V99 VALUE 50.00.
       01  WS-PAY                   PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-PAY = WS-HOURS * WS-RATE.
           DISPLAY "PAY=" WS-PAY.
           STOP RUN.
```

(บันทึกไว้ที่ `src/payroll.cob`)

### ทำไมต้องมี .gitignore สำหรับโปรเจกต์ COBOL

เมื่อคอมไพล์ `src/payroll.cob` ด้วย `cobc -x -o src/payroll src/payroll.cob` จะได้ไฟล์ execute
(`payroll`) รวมถึงอาจมีไฟล์ชั่วคราวอื่น ๆ เช่น `.o`, `.lst` (ถ้าใช้ flag `-t`) เกิดขึ้น **ไฟล์เหล่านี้
ไม่ควรถูกเก็บใน Git เลย** เพราะ:

1. เป็นไฟล์ที่**สร้างขึ้นใหม่ได้เสมอ**จาก source code (`.cob`) — ไม่ใช่ source of truth
2. ไฟล์ binary แบบนี้**เทียบ diff ไม่ได้อย่างมีความหมาย** (ต่างจากไฟล์ text)
3. ทำให้ repository มีขนาดใหญ่ขึ้นเรื่อย ๆ โดยไม่จำเป็น และอาจขึ้นกับ platform (execute ที่คอมไพล์
   บน Linux รันบน Windows ไม่ได้)

### สร้างไฟล์ .gitignore

```
# Compiled GnuCOBOL executables and intermediate build files
*.exe
payroll
!*.cob
!*.cpy
*.o
*.obj
*.lst
*.trace
listing/
build/
```

### Commit ครั้งแรก

```bash
git add .gitignore src/payroll.cob
git commit -m "Initial commit: add payroll program and gitignore"
git log --oneline
```

**ผลลัพธ์จริง:**

```
aafdb56 Initial commit: add payroll program and gitignore
```

### พิสูจน์ว่า .gitignore ทำงาน

```bash
cobc -x -o src/payroll src/payroll.cob
git status
```

**ผลลัพธ์จริง (สังเกตว่าไฟล์ execute `src/payroll` ที่เพิ่งคอมไพล์ไม่ปรากฏใน git status เลย):**

```
On branch master
nothing to commit, working tree clean
```

### อธิบายจุดสำคัญ

- `payroll` ในไฟล์ `.gitignore` ตรงกับชื่อไฟล์ execute พอดี (ไม่มีนามสกุล เพราะ Linux executable
  ไม่ต้องมีนามสกุล) — บรรทัดนี้ทำให้ Git **มองไม่เห็นไฟล์นี้เลย** แม้จะรันคำสั่ง `cobc` สร้างไฟล์นี้
  ขึ้นมาใหม่กี่ครั้งก็ตาม
- `!*.cob` และ `!*.cpy`: เครื่องหมาย `!` หมายถึง **negation** (ข้อยกเว้น) — บรรทัดนี้ยืนยันชัดเจนว่า
  ไฟล์ source code (`.cob`) และ copybook (`.cpy`) **ต้อง**ถูก track เสมอ แม้จะมี pattern อื่นใน
  `.gitignore` ที่อาจบังเอิญตรงกับชื่อไฟล์เหล่านี้ก็ตาม เป็นแนวปฏิบัติที่ดีเพื่อป้องกันความผิดพลาด
  โดยไม่ตั้งใจ (accidental ignore) ของไฟล์ source ที่สำคัญที่สุด
- `git status` แสดงข้อความ "nothing to commit, working tree clean" แม้ว่าจะมีไฟล์ `src/payroll`
  (ไฟล์ execute ที่คอมไพล์ใหม่) อยู่ในโฟลเดอร์จริง ๆ — นี่คือหลักฐานที่พิสูจน์ว่า `.gitignore`
  ทำงานถูกต้อง

### ข้อควรระวัง

- **`.gitignore` ใช้ได้เฉพาะกับไฟล์ที่ยังไม่เคย `git add` มาก่อนเท่านั้น** — ถ้าไฟล์ execute เคยถูก
  commit เข้า repository ไปแล้วโดยไม่ตั้งใจก่อนที่จะเพิ่ม `.gitignore` การเพิ่ม pattern ใน
  `.gitignore` ภายหลังจะ**ไม่ลบไฟล์นั้นออกจากประวัติที่มีอยู่แล้ว** ต้องใช้คำสั่ง `git rm --cached
  <ไฟล์>` เพื่อหยุด track ไฟล์นั้นอย่างชัดเจนอีกขั้นตอนหนึ่ง
- ทีมควรตกลงรูปแบบ `.gitignore` มาตรฐานสำหรับโปรเจกต์ COBOL ร่วมกันตั้งแต่ต้น (เช่น pattern สำหรับ
  ไฟล์ listing, ไฟล์ trace, โฟลเดอร์ build) เพื่อไม่ให้แต่ละคนใน team เผลอ commit ไฟล์ที่ไม่ควร
  commit เข้าไปโดยไม่ได้ตั้งใจ

### แบบฝึกหัดที่ 792.1

**โจทย์**: จงอธิบายว่าทำไมการเพิ่ม `!*.cob` ใน `.gitignore` (แม้จะดูเหมือนไม่จำเป็นเพราะ `*.cob`
ไม่ได้ถูก ignore อยู่แล้วในตัวอย่างนี้) ถึงยังเป็นแนวปฏิบัติที่ดีสำหรับโปรเจกต์ COBOL ขนาดใหญ่

**เฉลย**: ในโปรเจกต์ COBOL ขนาดใหญ่ที่มี `.gitignore` ซับซ้อนหลายสิบบรรทัด (เช่น ignore ทั้งโฟลเดอร์
`build/`, `*.tmp`, `*.log` และอื่น ๆ อีกมาก) มีความเสี่ยงที่ pattern บางอันอาจ**บังเอิญ**ครอบคลุม
ไฟล์ `.cob` บางไฟล์โดยไม่ได้ตั้งใจ เช่น ถ้ามีคนเพิ่ม pattern `build*` เพื่อ ignore โฟลเดอร์ build
แต่ดันมีไฟล์ source ชื่อ `buildreport.cob` อยู่จริง ไฟล์นั้นจะถูก ignore ไปด้วยโดยไม่มีใครสังเกตเห็น
จนกว่าจะพบว่าไฟล์หายไปจาก repository ตอนที่ต้องการมัน การใส่ `!*.cob` และ `!*.cpy` ไว้เป็น "เกราะ
ป้องกันชั้นสุดท้าย" ทำให้มั่นใจได้ว่า**ไม่ว่า pattern อื่นใน `.gitignore` จะซับซ้อนแค่ไหน ไฟล์
source code และ copybook จะไม่มีวันถูก ignore โดยไม่ตั้งใจเด็ดขาด** ซึ่งเป็นไฟล์ที่สำคัญที่สุดใน
โปรเจกต์ COBOL

---

## ขั้นตอนที่ 793: Branching Workflow — ทำงานคู่ขนานโดยไม่ชนกัน

### ทำไมต้องใช้ Branch

เมื่อทีมมีมากกว่า 1 คน การให้ทุกคนแก้ไขไฟล์เดียวกันบน branch หลัก (`main`) พร้อมกันตลอดเวลาจะทำให้
เกิดความสับสนและข้อผิดพลาดสูงมาก **Branch** คือ "เส้นเวลาคู่ขนาน" ที่แต่ละคน (หรือแต่ละฟีเจอร์)
สามารถแก้ไขโค้ดของตัวเองแยกอิสระ แล้วค่อยนำมา**รวม (merge)** เข้า branch หลักเมื่อพร้อมเท่านั้น

### สร้าง Branch ใหม่สำหรับฟีเจอร์ "คำนวณค่าล่วงเวลา"

```bash
git branch -M main
git checkout -b feature/overtime-pay
git branch
```

**ผลลัพธ์จริง:**

```
* feature/overtime-pay
  main
```

(เครื่องหมาย `*` แสดงว่ากำลังอยู่บน branch `feature/overtime-pay` ในขณะนี้)

### แก้ไขโค้ดบน Branch ใหม่

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PAYROLL.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-HOURS                 PIC 9(3) VALUE 160.
       01  WS-RATE                  PIC 9(3)V99 VALUE 50.00.
       01  WS-OT-HOURS              PIC 9(3) VALUE 10.
       01  WS-OT-RATE               PIC 9(3)V99 VALUE 75.00.
       01  WS-PAY                   PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-PAY = (WS-HOURS * WS-RATE)
               + (WS-OT-HOURS * WS-OT-RATE).
           DISPLAY "PAY=" WS-PAY.
           STOP RUN.
```

### ดู Diff ก่อน Commit

```bash
git diff -- src/payroll.cob
```

**ผลลัพธ์จริง:**

```diff
diff --git a/src/payroll.cob b/src/payroll.cob
index 6fb9243..36e9452 100644
--- a/src/payroll.cob
+++ b/src/payroll.cob
@@ -6,10 +6,13 @@
        WORKING-STORAGE SECTION.
        01  WS-HOURS                 PIC 9(3) VALUE 160.
        01  WS-RATE                  PIC 9(3)V99 VALUE 50.00.
+       01  WS-OT-HOURS              PIC 9(3) VALUE 10.
+       01  WS-OT-RATE               PIC 9(3)V99 VALUE 75.00.
        01  WS-PAY                   PIC 9(7)V99 VALUE 0.

        PROCEDURE DIVISION.
        MAIN-PARA.
-           COMPUTE WS-PAY = WS-HOURS * WS-RATE.
+           COMPUTE WS-PAY = (WS-HOURS * WS-RATE)
+               + (WS-OT-HOURS * WS-OT-RATE).
            DISPLAY "PAY=" WS-PAY.
            STOP RUN.
```

### Commit บน Branch

```bash
git add src/payroll.cob
git commit -m "Add overtime pay calculation"
git log --oneline --all --graph
```

**ผลลัพธ์จริง:**

```
* 76a2685 Add overtime pay calculation
* aafdb56 Initial commit: add payroll program and gitignore
```

### อธิบายจุดสำคัญ

- `git checkout -b feature/overtime-pay`: สร้าง branch ใหม่**และ**สลับไปทำงานบน branch นั้นทันที
  ในคำสั่งเดียว (เทียบเท่ากับ `git branch feature/overtime-pay` ตามด้วย `git checkout
  feature/overtime-pay` แยกกันสองคำสั่ง)
- ชื่อ branch `feature/overtime-pay` ใช้รูปแบบ **`feature/<ชื่อฟีเจอร์>`** ซึ่งเป็นธรรมเนียมที่นิยม
  มากในอุตสาหกรรม (เทียบกับ `bugfix/`, `hotfix/` สำหรับจุดประสงค์อื่น) ทำให้อ่านรายชื่อ branch
  ทั้งหมดแล้วเข้าใจทันทีว่าแต่ละ branch มีจุดประสงค์อะไร
- `git diff` ก่อน commit เป็นขั้นตอนสำคัญมากที่**ไม่ควรข้าม** — ช่วยให้ตรวจสอบว่าการแก้ไขตรงตามที่
  ตั้งใจจริง ๆ ก่อนบันทึกลงประวัติถาวร (โดยเฉพาะกับโค้ด COBOL ที่การเยื้อง column มีความหมายสำคัญ
  ตามที่จะกล่าวถึงในขั้นตอนที่ 796)

### ข้อควรระวัง

- การ commit บ่อย ๆ ด้วยข้อความที่ชัดเจน (เช่น "Add overtime pay calculation" ไม่ใช่ "fix" หรือ
  "update") ช่วยให้ `git log` กลายเป็นเอกสารประวัติที่มีประโยชน์จริง แทนที่จะเป็นแค่ backup ที่อ่าน
  ไม่รู้เรื่อง
- อย่าลืมว่า branch ใน Git เป็นเพียง**ตัวชี้ (pointer)** ไปยัง commit หนึ่ง ๆ เท่านั้น ไม่ใช่การ
  copy ไฟล์ทั้งหมด — ทำให้การสร้าง branch ใหม่ใน Git **เร็วมากและแทบไม่เสียพื้นที่เพิ่ม** ต่างจาก
  ระบบ Version Control รุ่นเก่าบางระบบที่การสร้าง branch มีค่าใช้จ่ายสูง

### แบบฝึกหัดที่ 793.1

**โจทย์**: จงอธิบายว่าทำไมทีมพัฒนา COBOL ควรสร้าง branch แยกสำหรับแต่ละฟีเจอร์ (เช่น
`feature/overtime-pay`) แทนที่จะแก้ไขทุกอย่างบน branch `main` โดยตรง โดยเชื่อมโยงกับความเสี่ยงที่
เคยเรียนมาใน Part 031 เรื่องการที่ COBOL compiler ไม่ตรวจสอบพารามิเตอร์ข้ามไฟล์

**เฉลย**: ถ้าทุกคนแก้ไข `main` โดยตรง และมีคนหนึ่งกำลังแก้ไข subprogram ที่ยังไม่เสร็จสมบูรณ์ (เช่น
เปลี่ยนจำนวนพารามิเตอร์ของ `CALL` แต่ยังแก้ไขโปรแกรมที่เรียกใช้ไม่ครบทุกจุด) คนอื่นในทีมที่ pull
โค้ดจาก `main` มาในช่วงเวลานั้นอาจได้โค้ดที่**คอมไพล์ผ่านแต่ crash ตอนรันจริง** (ตามปัญหาคลาสสิกจาก
Part 031 ขั้นตอนที่ 304) โดยไม่รู้ตัวเลยว่าเป็นเพราะการเปลี่ยนแปลงที่ยังไม่เสร็จของคนอื่น การแยก
งานที่ยังไม่เสร็จไว้บน feature branch ทำให้ `main` **คงสถานะที่ใช้งานได้เสมอ (always deployable)**
— คนอื่นในทีมที่ pull จาก `main` จะได้โค้ดที่ผ่านการทดสอบและรวม (merge) เรียบร้อยแล้วเท่านั้น
ไม่ใช่โค้ดที่ยังอยู่ระหว่างการพัฒนา นี่คือเหตุผลที่ branching workflow มีความสำคัญเป็นพิเศษกับ COBOL
ที่ compiler ไม่ช่วยจับข้อผิดพลาดเชิงโครงสร้างข้ามไฟล์ให้

---

## ขั้นตอนที่ 794: Merge Conflict จริง — เมื่อสองคนแก้ไขไฟล์เดียวกัน

### จำลองสถานการณ์: คนที่สองแก้ไข main พร้อมกัน

ในขณะที่ `feature/overtime-pay` (จากขั้นตอนที่ 793) กำลังพัฒนาอยู่ สมมติมีเพื่อนร่วมทีมอีกคนแก้ไข
`main` โดยตรง (ปรับอัตราค่าแรงต่อชั่วโมงขึ้น):

```bash
git checkout main
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PAYROLL.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-HOURS                 PIC 9(3) VALUE 160.
       01  WS-RATE                  PIC 9(3)V99 VALUE 55.00.
       01  WS-PAY                   PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-PAY = WS-HOURS * WS-RATE.
           DISPLAY "PAY=" WS-PAY.
           STOP RUN.
```

(สังเกตว่า `WS-RATE` เปลี่ยนจาก `50.00` เป็น `55.00`)

```bash
git add src/payroll.cob
git commit -m "Bump base hourly rate to 55.00 on main"
```

### พยายาม Merge — เกิด Conflict จริง

```bash
git merge feature/overtime-pay --no-edit
```

**ผลลัพธ์จริง:**

```
Auto-merging src/payroll.cob
CONFLICT (content): Merge conflict in src/payroll.cob
Automatic merge failed; fix conflicts and then commit the result.
```

### ดูเนื้อหาไฟล์ที่มี Conflict Marker

```bash
cat src/payroll.cob
```

**ผลลัพธ์จริง:**

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PAYROLL.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-HOURS                 PIC 9(3) VALUE 160.
<<<<<<< HEAD
       01  WS-RATE                  PIC 9(3)V99 VALUE 55.00.
=======
       01  WS-RATE                  PIC 9(3)V99 VALUE 50.00.
       01  WS-OT-HOURS              PIC 9(3) VALUE 10.
       01  WS-OT-RATE               PIC 9(3)V99 VALUE 75.00.
>>>>>>> feature/overtime-pay
       01  WS-PAY                   PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-PAY = (WS-HOURS * WS-RATE)
               + (WS-OT-HOURS * WS-OT-RATE).
            DISPLAY "PAY=" WS-PAY.
           STOP RUN.
```

### แก้ไข Conflict ด้วยมือ

ในกรณีนี้ทั้งสองการเปลี่ยนแปลง**ต้องการเก็บไว้ทั้งคู่** (อัตราค่าแรงใหม่ 55.00 **และ** ฟีเจอร์
overtime) — เราแก้ไขไฟล์ให้รวมทั้งสองส่วนเข้าด้วยกัน ลบเครื่องหมาย conflict marker
(`<<<<<<<`, `=======`, `>>>>>>>`) ออกทั้งหมด:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. PAYROLL.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-HOURS                 PIC 9(3) VALUE 160.
       01  WS-RATE                  PIC 9(3)V99 VALUE 55.00.
       01  WS-OT-HOURS              PIC 9(3) VALUE 10.
       01  WS-OT-RATE               PIC 9(3)V99 VALUE 75.00.
       01  WS-PAY                   PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-PAY = (WS-HOURS * WS-RATE)
               + (WS-OT-HOURS * WS-OT-RATE).
           DISPLAY "PAY=" WS-PAY.
           STOP RUN.
```

### สำคัญที่สุด: คอมไพล์และรันทดสอบก่อน commit การแก้ไข conflict เสมอ

```bash
cobc -x -o /tmp/payroll_check src/payroll.cob
/tmp/payroll_check
```

**ผลลัพธ์จริง:**

```
PAY=0009550.00
```

ตรวจสอบด้วยมือ: `(160 × 55.00) + (10 × 75.00) = 8800.00 + 750.00 = 9550.00` ✔ ถูกต้อง

### Commit ผลการแก้ไข Conflict

```bash
git add src/payroll.cob
git commit -m "Merge feature/overtime-pay into main, keep 55.00 base rate"
git log --oneline --all --graph
```

**ผลลัพธ์จริง:**

```
*   2badaf0 Merge feature/overtime-pay into main, keep 55.00 base rate
|\
| * 76a2685 Add overtime pay calculation
* | 9c0c238 Bump base hourly rate to 55.00 on main
|/
* aafdb56 Initial commit: add payroll program and gitignore
```

### อธิบายจุดสำคัญ

- Git **merge บางส่วนของไฟล์ได้อัตโนมัติ** (สังเกตว่าส่วน `PROCEDURE DIVISION` ถูกรวมให้อัตโนมัติ
  โดยไม่มี conflict เพราะ `main` ไม่ได้แก้ไขส่วนนั้นเลย) แต่**conflict เกิดเฉพาะจุดที่ทั้งสอง branch
  แก้ไขบรรทัดเดียวกันหรือใกล้กันมากเท่านั้น** (ในที่นี้คือบรรทัด `WS-RATE`)
- ข้อความ `CONFLICT (content): Merge conflict in src/payroll.cob` **ไม่ใช่ error ที่ต้องกลัว**
  แต่เป็นกลไกความปลอดภัยของ Git ที่**ปฏิเสธจะเดาเองว่าควรเก็บฝั่งไหน**เมื่อทั้งสองฝั่งขัดแย้งกันตรง
  จุดเดียวกัน แล้วให้มนุษย์เป็นผู้ตัดสินใจแทน
- **การคอมไพล์และรันทดสอบก่อน commit ผลการแก้ไข conflict คือขั้นตอนที่ขาดไม่ได้เด็ดขาด** — การแก้ไข
  conflict ด้วยมือมีความเสี่ยงสูงมากที่จะพิมพ์ผิดหรือลืมลบ conflict marker ตกค้างไว้ ซึ่งจะทำให้
  compile error ทันที (ทบทวน syntax ผิดปกติแบบนี้จาก Part ต่าง ๆ ที่ผ่านมา)

### ข้อควรระวัง

- **conflict marker ที่ลืมลบออก** (`<<<<<<<`, `=======`, `>>>>>>>`) เป็นข้อผิดพลาดที่พบบ่อยมากของ
  มือใหม่ — เครื่องหมายเหล่านี้ไม่ใช่ไวยากรณ์ COBOL เลย จะทำให้ `cobc` แจ้ง syntax error ทันทีถ้า
  หลงเหลืออยู่ในไฟล์ตอน commit (ซึ่งเป็นเหตุผลสำคัญที่ควรคอมไพล์ทดสอบก่อน commit เสมอตามที่กล่าวไว้)
- ในโปรเจกต์ COBOL ที่ใช้ fixed-format column-sensitive (ตามกฎเหล็กของหลักสูตรนี้) การแก้ไข
  conflict ด้วยมือต้องระวังเรื่อง**การเยื้อง column ให้ถูกต้องตามตำแหน่งเดิม**เป็นพิเศษ เพราะ editor
  บางตัวอาจแปลง Tab เป็น Space หรือในทางกลับกันโดยไม่ตั้งใจระหว่างกระบวนการแก้ไข ทำให้เกิด column
  overflow error ที่ไม่คาดคิด

### แบบฝึกหัดที่ 794.1

**โจทย์**: จงอธิบายว่าทำไมการที่ `git merge` รวมส่วน `PROCEDURE DIVISION` ให้อัตโนมัติโดยไม่มี
conflict (แม้ว่า `feature/overtime-pay` จะแก้ไขส่วนนั้นด้วย) ถึงอาจเป็นอันตรายในบางกรณี แม้ Git จะ
ไม่รายงาน error ใด ๆ เลยก็ตาม

**เฉลย**: Git ตัดสินใจว่าไม่มี conflict โดยพิจารณาแค่ว่า**บรรทัดเดียวกันถูกแก้ไขโดยทั้งสองฝั่งหรือ
ไม่**เท่านั้น — มันไม่มีความเข้าใจเรื่อง**ความหมายทางตรรกะ**ของโค้ด COBOL เลยแม้แต่น้อย ในกรณีนี้
โชคดีที่ `main` ไม่ได้แตะ `PROCEDURE DIVISION` เลย ทำให้ merge อัตโนมัติปลอดภัย แต่ถ้าสมมติว่า
`main` เคยแก้ไขสูตรคำนวณในส่วนนั้นเป็นสูตรอื่นที่ไม่เกี่ยวกับ overtime (เช่น เพิ่มการหักภาษี) และ
`feature/overtime-pay` ก็แก้ไขสูตรเดียวกันแต่คนละบรรทัด (เช่น เพิ่ม `COMPUTE` แยกบรรทัดใหม่) Git
อาจจะ merge ทั้งสองการเปลี่ยนแปลงเข้าด้วยกันโดยไม่มี conflict แจ้งเตือนเลย ทั้งที่ผลลัพธ์ทางตรรกะ
ของโปรแกรมหลัง merge อาจผิดพลาดโดยสิ้นเชิง (เช่น สูตรคำนวณสองอันที่ทับซ้อนกันโดยไม่ตั้งใจ) — นี่
คือเหตุผลที่ **automated test (Part 078–079) ต้องรันทันทีหลัง merge ทุกครั้งเสมอ** ไม่ใช่แค่พึ่งพา
การที่ Git "merge สำเร็จโดยไม่มี conflict" เป็นเครื่องยืนยันว่าโค้ดถูกต้อง เพราะทั้งสองเรื่องเป็น
คนละมิติกันโดยสิ้นเชิง

---

## ขั้นตอนที่ 795: Code Review เฉพาะทางของ COBOL — ปัญหา Column-Sensitive Diff

### ปัญหา: การเยื้อง (Re-indent) เพียงเล็กน้อยทำให้ diff ดูเหมือนเปลี่ยนทั้งบรรทัด

โค้ด COBOL แบบ Fixed-Format มีกฎเรื่องคอลัมน์ที่เข้มงวด (ทบทวนจาก Part 003) ทำให้แม้แต่การขยับ
ตำแหน่งเยื้องเพียง 1 ช่องว่าง (โดยไม่ได้เปลี่ยนตรรกะเลยแม้แต่นิดเดียว) ก็ทำให้ Git มองว่า "ทั้ง
บรรทัดถูกแก้ไข"

**ทดลองจริง**: นำไฟล์ `src/payroll.cob` ปัจจุบันมาเยื้องบรรทัด `COMPUTE WS-PAY` ให้เลื่อนไปทางขวา
1 ช่องว่าง แล้วเทียบ diff:

```bash
diff -u src/payroll.cob /tmp/payroll_reindent_demo.cob
```

**ผลลัพธ์จริง:**

```diff
--- src/payroll.cob	2026-09-28 19:19:15.488609677 +0000
+++ /tmp/payroll_reindent_demo.cob	2026-09-28 19:19:29.784091674 +0000
@@ -12,7 +12,7 @@
 
        PROCEDURE DIVISION.
        MAIN-PARA.
-           COMPUTE WS-PAY = (WS-HOURS * WS-RATE)
+            COMPUTE WS-PAY = (WS-HOURS * WS-RATE)
                + (WS-OT-HOURS * WS-OT-RATE).
            DISPLAY "PAY=" WS-PAY.
            STOP RUN.
```

สังเกตว่า diff แสดงว่าบรรทัด `COMPUTE WS-PAY` ทั้งบรรทัด "ถูกลบและเพิ่มใหม่" (`-` แล้วตามด้วย `+`)
ทั้งที่ความแตกต่างจริงคือ**ช่องว่างนำหน้าเพียง 1 ตัวอักษรเท่านั้น**

### ผลกระทบต่อกระบวนการ Code Review

เมื่อ Reviewer เห็น diff แบบนี้ในโค้ดจริงที่มีหลายร้อยบรรทัด อาจ:

1. **เสียเวลาอ่านและวิเคราะห์บรรทัดที่ "เปลี่ยน"** ทั้งที่ความหมายไม่เปลี่ยนเลย
2. **พลาดการเปลี่ยนแปลงที่สำคัญจริง** ที่ซ่อนอยู่ท่ามกลาง diff จำนวนมากที่เกิดจากการ re-indent
   ล้วน ๆ (เรียกปรากฏการณ์นี้ว่า **"Noisy Diff"**)
3. ถ้าใช้ `git blame` เพื่อดูว่า "บรรทัดนี้ใครเป็นคนเขียนตรรกะนี้" จะเจอชื่อของคนที่แค่ re-indent
   แทนที่จะเป็นคนที่เขียนตรรกะจริง ๆ ทำให้ประวัติความรับผิดชอบ (accountability) คลาดเคลื่อน

### วิธีลดปัญหานี้ในทางปฏิบัติ

| แนวทาง | รายละเอียด |
|---|---|
| กำหนดมาตรฐานการเยื้อง column ที่ชัดเจนตั้งแต่ต้นทีม | ลดโอกาสที่แต่ละคนจะเยื้องต่างกันโดยไม่ตั้งใจ |
| ใช้ `git diff -w` (ignore whitespace) เวลา review | มองข้ามความต่างที่เป็นแค่ช่องว่างเมื่อต้องการดูเฉพาะการเปลี่ยนแปลงเชิงตรรกะ |
| แยก commit "reformat only" ออกจาก commit ที่เปลี่ยนตรรกะ | ทำให้ reviewer รู้ทันทีว่า commit ไหนควรอ่านละเอียด commit ไหนแค่ดูผ่าน ๆ |
| ใช้เครื่องมือ formatter อัตโนมัติที่ตั้งค่าเดียวกันทั้งทีม | ลดการเยื้องที่ไม่สอดคล้องกันตั้งแต่ต้น (ถ้ามีเครื่องมือรองรับ fixed-format COBOL) |

**สาธิตการใช้ `git diff -w`:**

```bash
git diff -w -- src/payroll.cob
```

เมื่อความต่างเป็นแค่ whitespace ล้วน ๆ คำสั่งนี้จะไม่แสดงผลต่างใด ๆ เลย (ต่างจาก `git diff` ปกติ
ที่แสดงว่าทั้งบรรทัดเปลี่ยน) ช่วยให้ reviewer มองเห็นเฉพาะการเปลี่ยนแปลงที่มีความหมายจริงเท่านั้น

### ข้อควรระวัง

- `git diff -w` มีประโยชน์มากสำหรับ**การอ่าน review** แต่**ไม่ควรใช้ตัดสินใจว่าจะ commit อะไรบ้าง**
  — เมื่อจะ `git add`/`git commit` ต้องพิจารณาทุกการเปลี่ยนแปลงจริง (รวม whitespace) เพราะ
  whitespace ที่ผิดตำแหน่งใน fixed-format COBOL อาจทำให้ column overflow ได้จริงตามกฎเหล็กของ
  หลักสูตรนี้
- ปัญหา Noisy Diff รุนแรงขึ้นมากถ้าทีมใช้ editor ที่ตั้งค่า auto-format ต่างกัน (เช่น บาง editor
  แปลง Tab เป็น Space อัตโนมัติ) ทีม COBOL มืออาชีพควรตกลงและบังคับใช้การตั้งค่า editor ที่เหมือนกัน
  ทั้งทีมอย่างเข้มงวด (เช่น ผ่านไฟล์ `.editorconfig`) เพื่อป้องกันปัญหานี้ตั้งแต่ต้น

### แบบฝึกหัดที่ 795.1

**โจทย์**: จงอธิบายว่าทำไมการแยก commit ที่ "reformat เท่านั้น" ออกจาก commit ที่ "เปลี่ยนตรรกะ"
(ตามแนวทางที่สามในตารางข้างต้น) ถึงช่วย reviewer ได้มากกว่าการใช้ `git diff -w` เพียงอย่างเดียว

**เฉลย**: `git diff -w` ช่วยได้แค่**ตอนที่ reviewer เลือกใช้ flag นี้เอง**ตอนดู diff แต่ถ้า commit
หนึ่งรวมทั้งการ reformat และการเปลี่ยนตรรกะเข้าด้วยกัน ประวัติ (`git log`) ของ commit นั้นจะดูสับสน
ตลอดไป ไม่ว่าจะย้อนกลับมาดูกี่ครั้งก็ตาม (ต้องใช้ `-w` ทุกครั้งที่ดู) ในขณะที่การแยก commit ตั้งแต่
ต้น (เช่น commit แรก "Reformat payroll.cob to standard indentation" ตามด้วย commit ที่สอง "Add
overtime pay calculation") ทำให้ **ประวัติทั้งหมดชัดเจนถาวร** — ใครก็ตามที่มาดูประวัติภายหลัง
(แม้จะไม่รู้จักหรือไม่ได้ใช้ flag `-w` เลย) จะเห็นทันทีว่า commit ไหนเป็นแค่ reformat (ดู diff ผ่าน ๆ
ได้) และ commit ไหนมีการเปลี่ยนแปลงตรรกะจริงที่ต้อง review อย่างละเอียด วิธีนี้จึงเป็นแนวทางที่
ยั่งยืนกว่าการพึ่งพา flag ของเครื่องมือที่ต้องเลือกใช้เองทุกครั้ง

---

## ขั้นตอนที่ 796: การประสานงานเมื่อแก้ไข Copybook ที่ใช้ร่วมกันหลายโปรแกรม

### ทบทวน: Copybook คืออะไร (จาก Part 033)

**Copybook** (`.cpy`) คือไฟล์ที่เก็บโครงสร้างข้อมูลหรือโค้ดที่ใช้ร่วมกันหลายโปรแกรม ผ่านคำสั่ง
`COPY` — ปัญหาเฉพาะทางของ COBOL ที่ไม่ค่อยพบในภาษาสมัยใหม่ (ที่มักมีระบบ module/import ตรวจสอบ
ความสอดคล้องข้ามไฟล์ให้อัตโนมัติ) คือ **การแก้ไข copybook หนึ่งไฟล์อาจกระทบหลายสิบโปรแกรมที่
`COPY` มันเข้าไปใช้ โดยที่ compiler ไม่มีทางเตือนได้ว่ามีโปรแกรมไหนบ้างที่ได้รับผลกระทบ**

### สร้าง Copybook ที่ใช้ร่วมกัน

```cobol
      *> Shared customer record layout - included by multiple
      *> programs via COPY. Any change here must be coordinated
      *> across every program that copies it.
       01  CUSTOMER-RECORD.
           05  CUST-ID              PIC 9(5).
           05  CUST-NAME            PIC X(20).
           05  CUST-BALANCE         PIC 9(7)V99.
```

(บันทึกไว้ที่ `copybooks/customer.cpy`)

### สองโปรแกรมที่ COPY copybook นี้

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. BILLING.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       COPY "customer.cpy".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 10001 TO CUST-ID.
           MOVE "SOMCHAI JAIDEE" TO CUST-NAME.
           MOVE 500.00 TO CUST-BALANCE.
           DISPLAY "BILLING: " CUST-ID " " CUST-NAME " "
               CUST-BALANCE.
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STATEMENTS.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       COPY "customer.cpy".

       PROCEDURE DIVISION.
       MAIN-PARA.
           MOVE 20002 TO CUST-ID.
           MOVE "PRASERT KAEWTA" TO CUST-NAME.
           MOVE 1200.00 TO CUST-BALANCE.
           DISPLAY "STATEMENT: " CUST-ID " " CUST-NAME " "
               CUST-BALANCE.
           STOP RUN.
```

**คอมไพล์และรันจริง (ต้องใช้ flag `-I` ชี้ไปยังโฟลเดอร์ copybook — ทบทวนจาก Part 033):**

```bash
cobc -x -I copybooks -o billing src/billing.cob
./billing
cobc -x -I copybooks -o statements src/statements.cob
./statements
```

**ผลลัพธ์จริง:**

```
BILLING: 10001 SOMCHAI JAIDEE       0000500.00
STATEMENT: 20002 PRASERT KAEWTA       0001200.00
```

### ตรวจสอบว่าใครใช้ copybook นี้บ้างก่อนแก้ไข (แนวปฏิบัติที่สำคัญมาก)

ก่อนแก้ไข copybook ที่ใช้ร่วมกัน ควรค้นหาก่อนเสมอว่ามีโปรแกรมใดบ้าง `COPY` มันอยู่:

```bash
grep -rl "customer.cpy" src/
```

**ผลลัพธ์จริง:**

```
src/billing.cob
src/statements.cob
```

คำสั่งนี้บอกทันทีว่า **การแก้ไข `customer.cpy` จะกระทบทั้ง `billing.cob` และ `statements.cob`**
ก่อนที่จะลงมือแก้ไขจริง — เป็นขั้นตอนที่ Git เองไม่ได้ทำให้อัตโนมัติ (Git ไม่รู้จัก `COPY` statement
ของ COBOL) แต่เป็นวินัยที่ทีมต้องสร้างขึ้นเอง

### พิสูจน์ผลกระทบจริง: ลดความกว้าง CUST-NAME โดยไม่ประสานงาน

สมมติมีคนแก้ไข `customer.cpy` ลดความกว้าง `CUST-NAME` จาก `PIC X(20)` เป็น `PIC X(10)` โดยคิดว่า
"ประหยัดพื้นที่" โดยไม่ได้ตรวจสอบก่อนว่ามีโปรแกรมไหนใช้ชื่อที่ยาวกว่า 10 ตัวอักษรอยู่:

```diff
--- customer_before.cpy
+++ customer.cpy
@@ -3,5 +3,5 @@
        01  CUSTOMER-RECORD.
            05  CUST-ID              PIC 9(5).
-           05  CUST-NAME            PIC X(20).
+           05  CUST-NAME            PIC X(10).
            05  CUST-BALANCE         PIC 9(7)V99.
```

**คอมไพล์ใหม่และรันจริง (ไม่มี error ใด ๆ เลยตอน compile):**

```bash
cobc -x -I copybooks -o billing_v2 src/billing.cob
./billing_v2
```

**ผลลัพธ์จริง (ยืนยันปัญหา — ชื่อลูกค้าถูกตัดทอนอย่างเงียบ ๆ):**

```
BILLING: 10001 SOMCHAI JA 0000500.00
```

สังเกตว่า `"SOMCHAI JAIDEE"` (14 ตัวอักษร) ถูกตัดเหลือแค่ `"SOMCHAI JA"` (10 ตัวอักษรพอดี) **โดยไม่มี
compile error หรือ warning ใด ๆ เตือนเลย** — นี่คือกับดักคลาสสิกของการแก้ไข copybook ที่ใช้ร่วมกัน
โดยไม่ประสานงานกับทุกโปรแกรมที่ใช้มัน

### อธิบายจุดสำคัญ

- `grep -rl "customer.cpy" src/` เป็นเทคนิคง่าย ๆ แต่ **จำเป็นอย่างยิ่ง** ก่อนแก้ไข copybook ใด ๆ
  ในโปรเจกต์จริง — ทีมขนาดใหญ่มักเขียน script อัตโนมัติที่รันคำสั่งลักษณะนี้เป็นส่วนหนึ่งของ CI
  pipeline (Part 078) เพื่อ**แสดงรายชื่อโปรแกรมที่ได้รับผลกระทบทุกครั้งที่มี Pull Request แก้ไข
  copybook** ให้ reviewer เห็นชัดเจนก่อนอนุมัติ
- ปัญหานี้เชื่อมโยงโดยตรงกับ Part 031 ขั้นตอนที่ 307 ที่สอนเรื่องการส่ง Group Item เป็นพารามิเตอร์
  — ถ้าโครงสร้างของ copybook เปลี่ยนไปแต่โปรแกรมที่ใช้ยังคาดหวังโครงสร้างเดิม จะเกิดปัญหาการตีความ
  ข้อมูลผิดเพี้ยนแบบเงียบ ๆ เหมือนกันทุกประการ

### ข้อควรระวัง

- **Copybook คือจุดที่มีความเสี่ยงสูงที่สุดจุดหนึ่งในการ Code Review ของโปรเจกต์ COBOL** เพราะ
  ผลกระทบของการแก้ไขไม่ได้จำกัดอยู่แค่ไฟล์ที่ diff แสดงเท่านั้น แต่กระจายไปถึงทุกโปรแกรมที่ `COPY`
  มันด้วย Reviewer ที่เห็น Pull Request แก้ไข `.cpy` ควรตรวจสอบรายชื่อโปรแกรมที่ได้รับผลกระทบเสมอ
  ก่อนอนุมัติ ไม่ใช่ดูแค่ diff ของไฟล์ `.cpy` เพียงไฟล์เดียว
- การเพิ่ม field ใหม่ท้าย copybook (เช่น เพิ่ม `05 CUST-EMAIL PIC X(30).` ต่อท้าย) มักปลอดภัยกว่า
  การเปลี่ยนขนาดหรือลำดับ field ที่มีอยู่เดิม เพราะโปรแกรมเดิมที่ยังไม่ถูกแก้ไขจะยังทำงานถูกต้อง
  เหมือนเดิม (ตราบใดที่ไม่มีการใช้ `REDEFINES` ที่พึ่งพาขนาดรวมของ group item — ทบทวนจาก Part 022)

### แบบฝึกหัดที่ 796.1

**โจทย์**: จงเสนอแนวปฏิบัติ 2 ข้อที่ทีม COBOL ควรนำมาใช้เป็นส่วนหนึ่งของ workflow การ Code Review
เพื่อลดความเสี่ยงจากการแก้ไข copybook ที่ใช้ร่วมกัน

**เฉลย**: (1) **บังคับให้ CI pipeline รัน `grep -rl "<ชื่อ copybook>" src/` โดยอัตโนมัติทุกครั้งที่
Pull Request แตะไฟล์ `.cpy`** แล้วโพสต์รายชื่อโปรแกรมที่ได้รับผลกระทบเป็นความคิดเห็นบน Pull Request
ให้ reviewer เห็นทันทีโดยไม่ต้องค้นหาเอง (2) **บังคับให้ CI คอมไพล์และรันชุดทดสอบ (Part 078–079)
ของ**ทุกโปรแกรม**ที่ `COPY` copybook ที่ถูกแก้ไข** ไม่ใช่แค่โปรแกรมที่ Pull Request แก้ไขไฟล์ตรง ๆ
เท่านั้น เพราะอย่างที่พิสูจน์ในขั้นตอนนี้ การเปลี่ยนแปลง copybook สามารถทำให้โปรแกรมอื่นพังได้โดยที่
Pull Request นั้นไม่ได้แตะไฟล์ของโปรแกรมนั้นเลยแม้แต่บรรทัดเดียว ทั้งสองแนวทางนี้ช่วยเปลี่ยนความ
เสี่ยงที่ซ่อนอยู่ (ซึ่ง Git และ compiler มองไม่เห็น) ให้กลายเป็นสิ่งที่ทีมมองเห็นได้ชัดเจนก่อนที่จะ
merge เข้า main

---

## ขั้นตอนที่ 797: Commit Message และประวัติที่มีความหมายสำหรับโค้ด COBOL

### ทำไม Commit Message มีความสำคัญเป็นพิเศษกับ COBOL

เนื่องจาก COBOL มักเกี่ยวข้องกับระบบที่มีอายุการใช้งานยาวนานหลายสิบปี (ทบทวนจาก Part 001) การอ่าน
ประวัติ `git log` ย้อนหลังหลายปีเพื่อทำความเข้าใจว่า "ทำไมโค้ดส่วนนี้ถึงเขียนแบบนี้" เป็นสิ่งที่
เกิดขึ้นบ่อยกว่าในโปรเจกต์ภาษาอื่นที่มักถูก rewrite ใหม่บ่อยกว่า

### รูปแบบ Commit Message ที่แนะนำ

```
<ประเภท>: <สรุปสั้น ๆ ไม่เกิน 50 ตัวอักษร>

<รายละเอียดเพิ่มเติม (ถ้าจำเป็น) อธิบายว่า "ทำไม" ไม่ใช่แค่ "อะไร">
```

ตัวอย่างประเภทที่นิยมใช้: `feat` (ฟีเจอร์ใหม่), `fix` (แก้บั๊ก), `refactor` (ปรับโครงสร้างโดยไม่
เปลี่ยนพฤติกรรม — ทบทวนจาก Part 081), `test` (เพิ่ม/แก้ไขชุดทดสอบ), `docs` (เอกสาร)

### ดูประวัติทั้งหมดของ repository ตัวอย่าง

```bash
git log --oneline --all --graph
```

**ผลลัพธ์จริง (จากทุกขั้นตอนที่ผ่านมาในบทนี้):**

```
* 04efb92 Add shared customer copybook and two programs that use it
*   2badaf0 Merge feature/overtime-pay into main, keep 55.00 base rate
|\
| * 76a2685 Add overtime pay calculation
* | 9c0c238 Bump base hourly rate to 55.00 on main
|/
* aafdb56 Initial commit: add payroll program and gitignore
```

### ตัวอย่างการเขียน Commit Message ที่ดีเทียบกับที่ไม่ดี

| ไม่ดี | ดี | เหตุผล |
|---|---|---|
| `fix` | `fix: correct overtime rate calculation for hours > 160` | บอกว่า "แก้อะไร" ชัดเจน ค้นหาย้อนหลังได้ |
| `update payroll.cob` | `feat: add overtime pay support to payroll calculation` | บอกเจตนาของการเปลี่ยนแปลง ไม่ใช่แค่บอกว่าไฟล์ไหนถูกแก้ |
| `wip` | (ไม่ควร commit สถานะที่ยังไม่สมบูรณ์เข้า main เลย — ใช้ branch แยกแทน) | commit message ที่บอกว่า "งานยังไม่เสร็จ" ไม่ควรอยู่ในประวัติถาวรของ main |

### ข้อควรระวัง

- ใน COBOL ที่โครงสร้างข้อมูล (PICTURE, ความกว้าง field) มีความสำคัญมาก การเขียน commit message
  ที่ระบุ**ผลกระทบต่อโครงสร้างข้อมูล**อย่างชัดเจน (เช่น "changes CUST-NAME width from X(20) to
  X(10) - coordinate with billing.cob and statements.cob") มีคุณค่ามากกว่าภาษาอื่นที่การเปลี่ยน
  ชนิดข้อมูลมักถูก compiler ตรวจสอบให้อัตโนมัติอยู่แล้ว
- อย่าใช้ commit message ภาษาไทยปนกับโค้ด COBOL ที่ต้องเป็นภาษาอังกฤษล้วนตามกฎเหล็กของหลักสูตร —
  แม้ commit message จะไม่ใช่ส่วนหนึ่งของไฟล์ `.cob` โดยตรง แต่ทีมควรตกลงภาษาที่ใช้ให้สอดคล้องกัน
  ทั้งโปรเจกต์เพื่อให้ทุกคนในทีมนานาชาติอ่านประวัติเข้าใจตรงกัน

### แบบฝึกหัดที่ 797.1

**โจทย์**: จงเขียน commit message ที่ดีสำหรับการเปลี่ยนแปลง copybook ในขั้นตอนที่ 796 (การลดความ
กว้าง `CUST-NAME` จาก `PIC X(20)` เป็น `PIC X(10)`) โดยระบุผลกระทบที่ต้องประสานงานให้ชัดเจน

**เฉลย**:

```
fix: reduce CUST-NAME width in customer.cpy from X(20) to X(10)

BREAKING CHANGE: this affects billing.cob and statements.cob which
both COPY this copybook. Verified with `grep -rl "customer.cpy" src/`
before making this change. Names longer than 10 characters will be
silently truncated - see updated test cases in billing_test.cob.
```

Commit message นี้ทำหน้าที่ 3 อย่างพร้อมกัน: (1) สรุปการเปลี่ยนแปลงในบรรทัดแรกสั้น ๆ (2) ทำเครื่องหมาย
`BREAKING CHANGE` ชัดเจนเพื่อเตือนทุกคนที่มาอ่านประวัติภายหลังว่านี่ไม่ใช่การเปลี่ยนแปลงเล็กน้อย
(3) ระบุรายชื่อโปรแกรมที่ได้รับผลกระทบและวิธีที่ตรวจสอบมา (คำสั่ง `grep`) ทำให้ผู้ที่ทำ Code Review
หรือผู้ที่มาแก้ไข bug ในอนาคตเข้าใจบริบทได้ทันทีโดยไม่ต้องไปสืบค้นเพิ่มเติมเอง

---

## ขั้นตอนที่ 798: Pull Request Workflow และ Branch Protection

### วงจรชีวิตของ Pull Request สำหรับทีม COBOL

1. นักพัฒนาสร้าง feature branch (เช่น `feature/overtime-pay` จากขั้นตอนที่ 793)
2. พัฒนาและ commit การเปลี่ยนแปลงบน branch นั้น พร้อมรัน unit test/golden output test ในเครื่อง
   ตัวเองก่อนเสมอ (Part 079)
3. Push branch ขึ้น remote repository (เช่น GitHub) แล้วเปิด Pull Request เพื่อขอ merge เข้า `main`
4. **CI pipeline (Part 078) รันอัตโนมัติทันที** — build + test ทุกโปรแกรมที่เกี่ยวข้อง
5. เพื่อนร่วมทีมทำ Code Review (พิจารณาประเด็นเฉพาะทางจากขั้นตอนที่ 795–796)
6. เมื่อ CI ผ่านและได้รับการอนุมัติจาก reviewer แล้ว จึง merge เข้า `main`

### Branch Protection Rule ที่แนะนำสำหรับโปรเจกต์ COBOL

| กฎ | เหตุผล |
|---|---|
| ห้าม push ตรงเข้า `main` — ต้องผ่าน Pull Request เท่านั้น | บังคับให้ทุกการเปลี่ยนแปลงผ่าน Code Review |
| บังคับให้ CI (build + unit test + golden output test) ผ่านก่อน merge ได้ | ป้องกันไม่ให้โค้ดที่คอมไพล์ไม่ผ่านหรือ test ล้มเหลวเข้า main |
| บังคับให้มี reviewer อนุมัติอย่างน้อย 1 คนก่อน merge | เพิ่มโอกาสจับปัญหาเฉพาะทาง เช่น copybook coordination (ขั้นตอนที่ 796) |
| แจ้งเตือนอัตโนมัติเมื่อ Pull Request แตะไฟล์ `.cpy` | เตือน reviewer ให้ตรวจสอบผลกระทบข้ามโปรแกรมเป็นพิเศษ |

### ความเชื่อมโยงกับ Part 078

Branch protection rule ข้อที่สอง ("บังคับให้ CI ผ่านก่อน merge") คือจุดที่ทำให้ pipeline ที่เรา
สร้างใน Part 078 (`pipeline.sh`, `ci_report.sh`) และ Part 079 (`run_unit_tests.sh`) **มีความหมาย
จริงจังในทางปฏิบัติ** — มันไม่ใช่แค่สคริปต์ที่รันแล้วดูผลลัพธ์เฉย ๆ แต่กลายเป็น**ประตูที่ขวางกั้น
ไม่ให้โค้ดที่มีปัญหาเข้าสู่ `main` ได้เลย** โดยอัตโนมัติ ไม่ต้องพึ่งพาความจำหรือวินัยของมนุษย์ล้วน ๆ

### ข้อควรระวัง

- Branch protection ที่เข้มงวดเกินไป (เช่น บังคับ reviewer หลายคนสำหรับการแก้ไขเล็กน้อยมาก) อาจทำให้
  ทีมทำงานช้าลงโดยไม่จำเป็น ควรปรับระดับความเข้มงวดให้เหมาะสมกับความเสี่ยงจริงของแต่ละส่วนของโค้ด
  (เช่น การแก้ไข copybook หรือโปรแกรมที่เกี่ยวกับการเงินโดยตรง ควรเข้มงวดกว่าการแก้ไขข้อความ
  `DISPLAY` ธรรมดา)
- อย่าลืมว่า Branch Protection Rule เป็นฟีเจอร์ของ**บริการ hosting** (เช่น GitHub, GitLab) ไม่ใช่
  ของ Git เอง — Git แกนกลางไม่มีแนวคิดเรื่อง "ห้าม push เข้า branch นี้" อยู่ในตัว การตั้งค่านี้ต้อง
  ทำผ่านการตั้งค่าของ repository บนบริการ hosting ที่ทีมเลือกใช้

### แบบฝึกหัดที่ 798.1

**โจทย์**: จงอธิบายว่าทำไม branch protection rule ที่ "บังคับให้ CI ผ่านก่อน merge ได้" ถึงสำคัญ
กว่าการพึ่งพาแค่ "นักพัฒนาสัญญาว่าจะรัน test เองในเครื่องก่อน push" เพียงอย่างเดียว

**เฉลย**: การพึ่งพาวินัยของมนุษย์ล้วน ๆ มีความเสี่ยงที่จะผิดพลาดได้เสมอ ไม่ว่าจะด้วยความรีบเร่ง
ลืมรัน test ก่อน push หรือแม้แต่รัน test บนเครื่องตัวเองที่มีสภาพแวดล้อมต่างจากเครื่องอื่นในทีม
(ปัญหา "works on my machine" ที่กล่าวถึงใน Part 078 ขั้นตอนที่ 779) การให้ CI รันอัตโนมัติบน
สภาพแวดล้อมที่ควบคุมได้และเหมือนกันทุกครั้ง (เช่น GitHub Actions runner ที่กำหนดไว้แน่นอน) และ
ผูกผลของมันเข้ากับ branch protection rule โดยตรง ทำให้**ไม่มีทางที่โค้ดที่ยังไม่ผ่านการทดสอบจะเข้า
`main` ได้เลย ไม่ว่านักพัฒนาคนนั้นจะตั้งใจหรือไม่ตั้งใจก็ตาม** เปลี่ยนจาก "ความหวังว่าทุกคนจะทำตาม
กระบวนการ" ให้กลายเป็น "กฎที่ระบบบังคับใช้เองโดยอัตโนมัติ" ซึ่งเชื่อถือได้มากกว่าในระยะยาวสำหรับ
ทีมที่มีขนาดใหญ่ขึ้นเรื่อย ๆ

---

## ขั้นตอนที่ 799: กลยุทธ์ Branching ที่นิยมใช้กับโปรเจกต์ COBOL ขนาดใหญ่

### เปรียบเทียบกลยุทธ์ Branching หลัก 2 แบบ

| กลยุทธ์ | ลักษณะ | เหมาะกับ |
|---|---|---|
| **GitHub Flow** (เรียบง่าย) | มี `main` เดียวที่ deploy ได้เสมอ + feature branch สั้น ๆ ที่ merge บ่อย | ทีมที่ deploy บ่อย (Continuous Deployment) |
| **Git Flow** (ซับซ้อนกว่า) | มี `main` (production), `develop` (รวมงานที่พัฒนาเสร็จรอ release), `release/*`, `hotfix/*` แยกกันชัดเจน | ทีมที่ release เป็นรอบ (เช่น ทุกไตรมาส) ต้องการควบคุมเวอร์ชันที่ชัดเจน |

### ทำไม Git Flow มักเหมาะกับระบบ COBOL ระดับองค์กรมากกว่า GitHub Flow

ระบบ COBOL ระดับองค์กร (เช่น ระบบธนาคาร, ประกันภัย ตามที่ Part 001 อธิบาย) มักมี**รอบการ release
ที่เป็นทางการและมีการทดสอบ regression อย่างละเอียดก่อน deploy จริงเสมอ** (ต่างจากสตาร์ทอัพที่อาจ
deploy วันละหลายครั้ง) ทำให้แนวคิดของ **Git Flow** ที่แยก `develop` (พื้นที่รวมงานที่เสร็จแล้วรอ
ทดสอบ) ออกจาก `main`/`master` (โค้ดที่ผ่าน production release แล้วจริง ๆ) และมี `hotfix/*` แยก
สำหรับแก้ไขด่วนบนระบบ production โดยไม่ต้องรอรวมกับงานที่ยังพัฒนาไม่เสร็จ **สอดคล้องกับวงจรการ
ทำงานจริงของทีม COBOL องค์กรขนาดใหญ่มากกว่า**

### ตัวอย่างโครงสร้าง Branch แบบ Git Flow สำหรับโปรเจกต์ COBOL

```
main        ──●────────────●─────────────●──────────►  (production releases เท่านั้น)
              │             │             │
release/2.1  ─┴──●───●──────┘             │
                                          │
hotfix/urgent-tax-fix ────────────────────┴──►  (แก้ด่วนจาก main โดยตรง)

develop     ──●───●───●───●───●───●───●──────────────►  (รวมงานทุก feature)
              │       │       │
feature/overtime  ────┘       │
feature/discount  ────────────┘
```

### ข้อควรระวัง

- ไม่มีกลยุทธ์ branching ใดที่ "ถูกต้องที่สุด" ตายตัว — ทีมต้องเลือกให้เหมาะกับ**จังหวะการ release
  จริง**ของตัวเอง ทีมที่เลือก Git Flow ทั้งที่ deploy บ่อยมากอาจรู้สึกว่ากระบวนการซับซ้อนเกินความ
  จำเป็น ในขณะที่ทีมที่เลือก GitHub Flow ทั้งที่ต้อง release เป็นรอบทางการอาจขาดความชัดเจนเรื่อง
  "เวอร์ชันไหนกำลัง deploy อยู่จริงที่ production"
- ไม่ว่าจะเลือกกลยุทธ์ใด **หลักการ CI ต้องรันทุกครั้งที่มีการ merge เข้า branch สำคัญเสมอ** (ทั้ง
  `main` และ `develop` ถ้าใช้ Git Flow) ไม่ใช่แค่ตอน merge เข้า `main` เพียงอย่างเดียว

### แบบฝึกหัดที่ 799.1

**โจทย์**: จงอธิบายว่าทำไม branch `hotfix/*` ใน Git Flow ถึงถูกออกแบบให้แตกออกมาจาก `main`
โดยตรง แทนที่จะแตกออกมาจาก `develop`

**เฉลย**: `develop` เป็นที่รวมงานของ feature ต่าง ๆ ที่**ยังไม่ผ่านการทดสอบ release เต็มรูปแบบ**
และอาจมีการเปลี่ยนแปลงที่ยังไม่เสถียรอยู่ ในขณะที่ `main` คือโค้ดที่ผ่านการ release และรันอยู่บน
production จริง ถ้าเกิดปัญหาด่วนบน production (เช่น พบว่าสูตรคำนวณภาษีผิดพลาดในระบบจริง ต้องแก้ไข
ด่วนที่สุด) การแตก branch จาก `develop` จะทำให้ hotfix นั้น**ติดพ่วงเอาการเปลี่ยนแปลงอื่น ๆ ที่ยัง
ไม่เสถียรจาก `develop` มาด้วยโดยไม่ตั้งใจ** ซึ่งเสี่ยงมากที่จะทำให้เกิดปัญหาใหม่ตามมาในสถานการณ์ที่
ต้องการความเร็วและความปลอดภัยสูงสุด การแตกจาก `main` โดยตรงรับประกันว่า hotfix มีแค่การแก้ไขที่
จำเป็นจริง ๆ เพียงอย่างเดียว บนพื้นฐานของโค้ดเวอร์ชันที่กำลังรันบน production อยู่จริง ทำให้ทดสอบ
และ deploy กลับได้เร็วและปลอดภัยที่สุดเท่าที่จะเป็นไปได้ (หลังจากนั้นค่อย merge hotfix กลับเข้า
`develop` ด้วยเพื่อไม่ให้การแก้ไขนี้หายไปตอน release รอบถัดไป)

---

## ขั้นตอนที่ 800: Capstone — ประกอบทุกเทคนิคเข้าด้วยกัน

### สรุปประวัติทั้งหมดของ Scratch Repository ในบทนี้

```bash
cd /tmp/cobol-git-demo
git log --oneline --all --graph
git status
```

**ผลลัพธ์จริง (สถานะสุดท้ายหลังทุกขั้นตอนในบทนี้):**

```
* 04efb92 Add shared customer copybook and two programs that use it
*   2badaf0 Merge feature/overtime-pay into main, keep 55.00 base rate
|\
| * 76a2685 Add overtime pay calculation
* | 9c0c238 Bump base hourly rate to 55.00 on main
|/
* aafdb56 Initial commit: add payroll program and gitignore
```

```
On branch main
nothing to commit, working tree clean
```

### Checklist สำหรับทีม COBOL ที่จะนำ Git Workflow ไปใช้จริง

| หัวข้อ | สิ่งที่ต้องทำ | Part ที่เกี่ยวข้อง |
|---|---|---|
| `.gitignore` | ครอบคลุมไฟล์ execute, `.o`, `.lst` พร้อม negation ป้องกัน `.cob`/`.cpy` | ขั้นตอนที่ 792 |
| Branching | เลือกกลยุทธ์ที่เหมาะกับจังหวะ release จริงของทีม | ขั้นตอนที่ 793, 799 |
| Merge Conflict | มีวินัยคอมไพล์+ทดสอบก่อน commit ผลการแก้ไข conflict เสมอ | ขั้นตอนที่ 794 |
| Code Review | ใช้ `git diff -w` ลด noise, แยก commit reformat ออกจาก commit ตรรกะ | ขั้นตอนที่ 795 |
| Copybook Coordination | `grep -rl` หาโปรแกรมที่ได้รับผลกระทบก่อนแก้ไข copybook ทุกครั้ง | ขั้นตอนที่ 796 |
| Commit Message | ระบุ "ทำไม" ไม่ใช่แค่ "อะไร" โดยเฉพาะการเปลี่ยนโครงสร้างข้อมูล | ขั้นตอนที่ 797 |
| Pull Request + CI | บังคับ CI ผ่านก่อน merge เสมอ (เชื่อม Part 078–079) | ขั้นตอนที่ 798 |

### ภาพรวม Workflow แบบเต็มที่เชื่อมกับ Part 078–079

```
นักพัฒนาสร้าง feature branch (799)
        │
        ▼
แก้ไขโค้ด + ตรวจสอบผลกระทบ copybook ด้วย grep (796)
        │
        ▼
รัน unit test + golden output test ในเครื่อง (078-079)
        │
        ▼
Commit ด้วยข้อความที่ชัดเจน (797)
        │
        ▼
Push + เปิด Pull Request
        │
        ▼
CI รันอัตโนมัติ (078-079) ──► ล้มเหลว ──► แก้ไขแล้วกลับไปด้านบน
        │ ผ่าน
        ▼
Code Review (795-796) ──► ขอแก้ไขเพิ่ม ──► แก้ไขแล้วกลับไปด้านบน
        │ อนุมัติ
        ▼
Merge เข้า main (branch protection บังคับ CI+review ผ่านแล้วเท่านั้น) (798)
```

### ข้อควรระวัง

- Workflow ทั้งหมดนี้เป็น**เครื่องมือที่ช่วยลดความเสี่ยง** ไม่ใช่เกราะป้องกันที่สมบูรณ์แบบ 100% —
  ทีมยังต้องมีวินัยและความเข้าใจในหลักการเบื้องหลังของแต่ละขั้นตอนอยู่เสมอ การทำตาม checklist โดย
  ไม่เข้าใจเหตุผล (เช่น รัน `grep` แต่ไม่เข้าใจว่าทำไมต้องรัน) มีความเสี่ยงที่จะพลาดกรณีที่ checklist
  ไม่ได้ครอบคลุมไว้
- อย่าลืมว่า scratch repository ใน `/tmp/cobol-git-demo` ที่ใช้สาธิตทั้งบทนี้เป็นเพียงตัวอย่าง
  การเรียนรู้เท่านั้น **ไม่ใช่ส่วนหนึ่งของ repository หลักสูตรนี้แต่อย่างใด** และจะหายไปเมื่อระบบ
  ล้างไฟล์ชั่วคราว — ถ้าต้องการทดลองด้วยตัวเอง ให้สร้าง scratch repository ใหม่ในเครื่องของคุณเอง
  ตามขั้นตอนในบทนี้

### แบบฝึกหัดที่ 800.1

**โจทย์**: จากทุกเทคนิคใน Part นี้ จงเรียงลำดับความสำคัญ (จากมากไปน้อย) ของ 3 เทคนิคที่คุณคิดว่า
**เฉพาะเจาะจงกับ COBOL มากที่สุด** (ไม่ใช่แนวปฏิบัติ Git ทั่วไปที่ใช้กับภาษาใดก็ได้) พร้อมให้เหตุผล

**เฉลย**: คำตอบอาจแตกต่างกันได้ตามมุมมอง แต่แนวทางที่สมเหตุสมผลคือ (1) **การประสานงานเมื่อแก้ไข
Copybook** (ขั้นตอนที่ 796) — เป็นปัญหาที่เกิดจากธรรมชาติเฉพาะของ `COPY` statement และการที่
compiler ไม่ตรวจสอบผลกระทบข้ามไฟล์ให้ ซึ่งภาษาสมัยใหม่ส่วนใหญ่ที่มีระบบ module/import ที่ตรวจสอบ
ได้ไม่ต้องกังวลเรื่องนี้มากเท่า (2) **Column-Sensitive Diff** (ขั้นตอนที่ 795) — เกิดจากกฎ
Fixed-Format ของ COBOL โดยตรงตามกฎเหล็กของหลักสูตรนี้ ซึ่งเป็นข้อจำกัดที่ภาษาสมัยใหม่ส่วนใหญ่ไม่มี
(3) **.gitignore สำหรับไฟล์ที่คอมไพล์แล้ว** (ขั้นตอนที่ 792) — แม้แนวคิด `.gitignore` จะเป็นสากล
กับทุกภาษา แต่รายละเอียดของไฟล์ที่ต้อง ignore (`.lst`, `.trace` จากการคอมไพล์ COBOL) เป็นรายละเอียด
เฉพาะทางที่ทีมต้องรู้จักเครื่องมือ `cobc` เป็นอย่างดีจึงจะตั้งค่าได้ถูกต้อง ส่วนเทคนิคอื่น ๆ ในบทนี้
เช่น Branch Protection, Pull Request Workflow, Git Flow เป็นแนวปฏิบัติ Git ทั่วไปที่ใช้ได้กับ
โปรเจกต์ภาษาใดก็ได้เหมือนกัน ไม่ได้เฉพาะเจาะจงกับ COBOL

---

## สรุปท้ายบท

Part นี้พาคุณเรียนรู้ Git และ Workflow การทำงานร่วมกันของทีม โดยเน้นประเด็นเฉพาะทางที่โปรเจกต์
COBOL เจอจริง และทุกคำสั่ง Git ที่แสดงในบทนี้**รันจริงใน scratch repository แยกต่างหากแล้วทั้งหมด**:

- เหตุผลที่ COBOL ยุคใหม่ (ที่พัฒนาด้วย GnuCOBOL บนเครื่องสมัยใหม่) ใช้ Git ได้เหมือนภาษาอื่นทุก
  ประการ แม้จะมีประวัติผูกกับเครื่องมือเฉพาะของ Mainframe มาก่อน
- การสร้าง `.gitignore` ที่เหมาะสมสำหรับไฟล์ execute และไฟล์ชั่วคราวจากการคอมไพล์ COBOL พร้อม
  negation pattern ป้องกันไฟล์ source สำคัญ
- Branching Workflow และการแก้ไข Merge Conflict จริง พร้อมวินัยคอมไพล์+ทดสอบก่อน commit เสมอ
- ปัญหา Column-Sensitive Diff ที่เฉพาะเจาะจงกับ COBOL Fixed-Format และวิธีลดผลกระทบต่อ Code Review
- การประสานงานเมื่อแก้ไข Copybook ที่ใช้ร่วมกันหลายโปรแกรม พร้อมการพิสูจน์ผลกระทบจริงจากการแก้ไข
  โดยไม่ประสานงาน (ชื่อลูกค้าถูกตัดทอนอย่างเงียบ ๆ)
- แนวทางการเขียน Commit Message ที่มีความหมายสำหรับระบบที่มีอายุการใช้งานยาวนาน
- Pull Request Workflow และ Branch Protection ที่เชื่อมโยง CI จาก Part 078–079 เข้ากับกระบวนการ
  merge จริง
- การเปรียบเทียบกลยุทธ์ Branching (GitHub Flow vs Git Flow) และเหตุผลที่ Git Flow มักเหมาะกับ
  ระบบ COBOL ระดับองค์กรมากกว่า

Part ถัดไป (**Part 081**) จะเปลี่ยนมุมมองกลับไปที่**ตัวโค้ดเอง**: Code Refactoring และ Clean Code
สำหรับ COBOL — การแยก paragraph ออกเป็น subprogram, การกำจัด GO TO แบบ spaghetti ด้วย PERFORM
โครงสร้าง, และการลด nested IF ด้วย EVALUATE โดยทุกเทคนิคจะพิสูจน์ด้วยการคอมไพล์และรันทั้งก่อนและ
หลัง refactor เพื่อยืนยันว่าพฤติกรรมของโปรแกรมไม่เปลี่ยนแปลง — ซึ่งเป็นสิ่งที่ Git Workflow ใน Part
นี้ (โดยเฉพาะ Branch + Pull Request + CI) จะรองรับกระบวนการ refactor อย่างปลอดภัยได้อย่างสมบูรณ์

**[← กลับไป Part 079: Unit Testing สำหรับ COBOL](part-079-unit-testing-cobol.md)** | **[ไปยัง Part 081: Code Refactoring และ Clean Code สำหรับ COBOL →](part-081-refactoring-clean-code.md)**
