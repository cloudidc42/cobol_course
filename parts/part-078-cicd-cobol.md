# Part 078: CI/CD Pipeline สำหรับโปรเจกต์ COBOL (ขั้นตอนที่ 771–780)

## คำนำของ Part นี้

Part 076–077 พาเราเรียนรู้การห่อหุ้มโปรแกรม COBOL ด้วย Docker และแนวคิดการย้ายระบบขึ้น Cloud
ถึงตอนนี้เรามีโปรแกรม COBOL ที่รันในสภาพแวดล้อมสมัยใหม่ได้แล้ว แต่คำถามถัดไปคือ: **เมื่อทีมพัฒนา
หลายคนแก้ไขโค้ด COBOL ร่วมกันทุกวัน เราจะมั่นใจได้อย่างไรว่าการแก้ไขแต่ละครั้งไม่ทำให้ระบบพัง?**

คำตอบคือ **CI/CD (Continuous Integration / Continuous Delivery)** — แนวปฏิบัติที่ภาษาโปรแกรมสมัยใหม่
อย่าง Java, Python, JavaScript ใช้กันเป็นมาตรฐานมานานแล้ว แต่หลายคนเข้าใจผิดว่า COBOL "ทำ CI/CD
ไม่ได้" เพราะเป็นภาษาเก่าที่ผูกกับ Mainframe ความจริงคือ COBOL ก็สามารถเข้าสู่กระบวนการ CI/CD ได้
เหมือนภาษาอื่นทุกประการ เพียงแต่เครื่องมือที่ใช้ (compiler, test runner) ต่างออกไป

Part นี้จะพาคุณสร้าง **Pipeline จริงที่ทำงานได้จริง** ตั้งแต่การเขียน shell script คอมไพล์และ
ทดสอบโปรแกรม COBOL อัตโนมัติ ไปจนถึงการเขียนไฟล์ GitHub Actions YAML ที่ถูกต้องตามไวยากรณ์จริง
ทุกสคริปต์ shell ในบทนี้ **รันจริงในสภาพแวดล้อมทดสอบแล้ว** ทั้งกรณีที่ผ่าน (PASS) และกรณีที่ถูก
จงใจทำให้ล้มเหลว (FAIL) เพื่อให้คุณเห็นพฤติกรรมจริงของ pipeline ทั้งสองด้าน

> **หมายเหตุความซื่อตรงเรื่องขอบเขตการทดสอบ**: สภาพแวดล้อมที่ใช้เขียนหลักสูตรนี้ไม่มีทางเชื่อมต่อ
> ไปยัง GitHub Actions จริงเพื่อรันไฟล์ YAML ได้ (ไม่มี GitHub runner ให้เรียกใช้) ดังนั้นไฟล์ YAML
> ในขั้นตอนที่ 778 เป็น **ไวยากรณ์ที่ถูกต้องและใช้งานได้จริงเมื่อ push ขึ้น GitHub repository ที่มี
> Actions เปิดใช้งาน** แต่สิ่งที่ **ทดสอบรันจริงแล้ว 100%** ในบทนี้คือ shell script เทียบเท่า
> (`build.sh`, `test.sh`, `pipeline.sh`) ซึ่งทำหน้าที่เดียวกันกับสิ่งที่ YAML บอกให้ GitHub Actions
> runner ทำทุกประการ — เพียงแต่รันบนเครื่องโดยตรงแทนที่จะรันผ่าน GitHub's cloud runner

---

## ขั้นตอนที่ 771: CI/CD คืออะไร และทำไมสำคัญกับโปรเจกต์ COBOL

### นิยามของ CI และ CD

- **CI (Continuous Integration)**: แนวปฏิบัติที่นักพัฒนาทุกคนใน team นำโค้ดของตัวเองมารวม (integrate)
  เข้ากับโค้ดหลักบ่อย ๆ (เช่น ทุกครั้งที่ push หรือเปิด Pull Request) โดยมี**ระบบอัตโนมัติ**คอยคอมไพล์
  และรันชุดทดสอบทุกครั้งทันที เพื่อจับข้อผิดพลาดให้เร็วที่สุดเท่าที่จะทำได้ (แทนที่จะรอไปเจอตอน
  deploy จริง)
- **CD (Continuous Delivery/Deployment)**: ขั้นตอนถัดจาก CI คือการนำโปรแกรมที่ผ่านการทดสอบแล้ว
  ไป**เตรียมพร้อมส่งมอบ** (Delivery) หรือ**นำไป deploy จริงโดยอัตโนมัติ** (Deployment) โดยไม่ต้องมี
  มนุษย์มานั่งกดปุ่ม build/copy ไฟล์ทีละขั้นตอนด้วยมือ

### ทำไมเรื่องนี้สำคัญเป็นพิเศษกับ COBOL

ทบทวนจาก Part 031 (ขั้นตอนที่ 304): เราพิสูจน์แล้วว่า COBOL **ไม่มีการตรวจสอบข้าม compilation unit**
เลย — ถ้าโปรแกรมหลักส่งพารามิเตอร์ผิดจำนวนไปให้โปรแกรมย่อย compiler จะไม่เตือนอะไรเลย โปรแกรมจะ
compile ผ่านสมบูรณ์แบบแล้วไป crash ตอนรันจริงด้วย Segmentation Fault เท่านั้น

นี่คือเหตุผลที่ **CI สำคัญกับ COBOL มากกว่าภาษาสมัยใหม่หลายภาษาด้วยซ้ำ**: เมื่อ compiler ไม่ช่วย
จับข้อผิดพลาดเชิงโครงสร้างให้ ภาระทั้งหมดตกไปอยู่ที่**การทดสอบอัตโนมัติที่รันทุกครั้งที่มีการแก้ไข**
แทน ถ้าไม่มี CI คอยรันชุดทดสอบให้ทุกครั้ง ทีมอาจไม่รู้เลยว่าการแก้ไข copybook หรือการเปลี่ยนลำดับ
พารามิเตอร์ใน subprogram ตัวหนึ่งทำให้อีก 5 โปรแกรมที่เรียกมันพังไปแล้ว จนกว่าจะไป deploy จริง

### องค์ประกอบพื้นฐานของ Pipeline สำหรับ COBOL

โปรเจกต์ COBOL โดยทั่วไปมี pipeline ที่ประกอบด้วยขั้นตอนหลัก ๆ ดังนี้:

| ขั้นตอน | หน้าที่ | เครื่องมือที่ใช้ในบทนี้ |
|---|---|---|
| 1. Checkout | ดึงโค้ดล่าสุดจาก repository | `git` (Part 080) |
| 2. Build | คอมไพล์ทุกโปรแกรม COBOL ที่เกี่ยวข้อง | `cobc -x` |
| 3. Test | รันโปรแกรมและเทียบผลลัพธ์กับค่าที่คาดหวัง | shell script + `diff` |
| 4. Report | สรุปผล PASS/FAIL ให้ทีมเห็นทันที | exit code + สรุปข้อความ |
| 5. Package/Deploy | บรรจุไฟล์ execute ที่ผ่านการทดสอบเพื่อส่งต่อ | (แนวคิด กล่าวถึงในขั้นตอนที่ 780) |

### โปรแกรมตัวอย่างที่จะใช้ตลอด Part นี้

เพื่อให้ทุกขั้นตอนเชื่อมโยงกัน เราจะใช้โปรแกรม COBOL เดียวกันนี้เป็นกรณีศึกษาหลักตลอดครึ่งแรกของ Part:
โปรแกรมคำนวณดอกเบี้ยอย่างง่ายที่ **มีผลลัพธ์แน่นอนทุกครั้งที่รัน (deterministic)** ซึ่งเป็นคุณสมบัติ
สำคัญที่โปรแกรมต้องมีก่อนจะนำไปเข้าสู่ automated test ได้ (โปรแกรมที่ผลลัพธ์เปลี่ยนไปทุกครั้ง เช่น
ใช้เวลาปัจจุบันหรือค่าสุ่ม จะทดสอบด้วยการเทียบผลลัพธ์ตรง ๆ แบบนี้ไม่ได้):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP771CALC.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRINCIPAL             PIC 9(7)V99 VALUE 10000.00.
       01  WS-RATE                  PIC 9V9999 VALUE 0.0500.
       01  WS-INTEREST              PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-INTEREST ROUNDED = WS-PRINCIPAL * WS-RATE.
           DISPLAY "INTEREST=" WS-INTEREST.
           STOP RUN.
```

**คอมไพล์และรัน:**

```bash
cobc -x -o step771calc step771calc.cob
./step771calc
```

**ผลลัพธ์จริง (ยืนยันด้วยการรัน):**

```
INTEREST=0000500.00
```

ผลลัพธ์นี้ (`INTEREST=0000500.00`) จะเป็น "ค่าที่คาดหวัง" (Expected Output) ที่เราจะใช้เทียบใน
ขั้นตอนถัดไป

### ข้อควรระวัง

- CI/CD ไม่ใช่เครื่องมือตัวเดียว แต่เป็น**แนวปฏิบัติ**ที่ประกอบด้วยหลายเครื่องมือทำงานร่วมกัน
  (compiler, test runner, version control, YAML config ของ CI service) — อย่าสับสนว่า "GitHub
  Actions" คือ CI/CD ทั้งหมด มันเป็นเพียง**หนึ่งใน platform** ที่ใช้รัน pipeline เท่านั้น
- โปรแกรมที่จะเข้า CI ได้ต้องมีผลลัพธ์ deterministic เสมอ — ถ้าโปรแกรมพิมพ์วันที่ปัจจุบันหรือใช้
  ค่าสุ่ม การเขียน automated test ต้องแยกส่วนที่ deterministic ออกจากส่วนที่ไม่ deterministic ก่อน
  (เทคนิคนี้จะกล่าวถึงเพิ่มเติมใน Part 079)

### แบบฝึกหัดที่ 771.1

**โจทย์**: จงอธิบายว่าทำไมข้อเท็จจริงที่ว่า "COBOL ไม่ตรวจสอบพารามิเตอร์ข้าม compilation unit"
(จาก Part 031) ถึงทำให้ CI มีความสำคัญกับ COBOL มากกว่าภาษาที่มีระบบ type-checking ข้ามไฟล์ที่
เข้มงวดกว่า

**เฉลย**: ในภาษาที่มีระบบ module/header ตรวจสอบ signature ของฟังก์ชันข้ามไฟล์ตั้งแต่ compile time
(เช่น TypeScript, Java ผ่าน interface) ข้อผิดพลาดเรื่องพารามิเตอร์ผิดชนิด/จำนวนจะถูกจับได้ทันทีตอน
คอมไพล์ โดยไม่ต้องรอให้ automated test ทำงานเลยด้วยซ้ำ แต่ใน COBOL, compiler คอมไพล์แต่ละไฟล์แยกกัน
โดยไม่รู้จักเนื้อหาของไฟล์อื่นเลย ทำให้ข้อผิดพลาดแบบนี้**ไม่มีทางถูกจับได้ตอน compile time**
ต้อง**รันจริง**เท่านั้นถึงจะเจอ (มักจะเจอในรูปแบบ Segmentation Fault) ดังนั้นระบบ CI ที่คอมไพล์และ
**รันทดสอบทุกโปรแกรมที่เกี่ยวข้องโดยอัตโนมัติทุกครั้งที่มีการแก้ไข** จึงเป็นเกราะป้องกันเดียวที่ทีม
COBOL มีอยู่จริงในการจับข้อผิดพลาดประเภทนี้ก่อนที่จะไปถึงระบบ production

---

## ขั้นตอนที่ 772: เขียน Build Script อัตโนมัติตัวแรก

### ทำไมต้องมี Build Script แทนการพิมพ์คำสั่ง cobc เอง

เมื่อโปรเจกต์มีโปรแกรม COBOL หลายสิบหรือหลายร้อยไฟล์ การจำคำสั่ง `cobc` ที่ถูกต้องสำหรับแต่ละโปรแกรม
(flag อะไรบ้าง, ต้องรวมไฟล์ไหนบ้าง) จะกลายเป็นภาระที่ผิดพลาดได้ง่ายมากถ้าทำด้วยมือ **Build Script**
คือสคริปต์ที่บันทึกขั้นตอนการคอมไพล์ไว้เป็นโค้ดที่รันซ้ำได้แน่นอนทุกครั้งเหมือนกัน — เป็นก้าวแรกที่
ขาดไม่ได้ก่อนจะไปสู่ CI เต็มรูปแบบ

### สร้าง build.sh

```bash
#!/bin/bash
# build.sh - compile a single COBOL program and report the result.
# Usage: ./build.sh <program-name-without-extension>
set -u
PROGRAM="$1"

echo "==> Compiling ${PROGRAM}.cob ..."
cobc -x -o "${PROGRAM}" "${PROGRAM}.cob"
BUILD_STATUS=$?

if [ ${BUILD_STATUS} -eq 0 ]; then
    echo "==> BUILD OK: ${PROGRAM}"
else
    echo "==> BUILD FAILED: ${PROGRAM} (cobc exit code ${BUILD_STATUS})"
fi

exit ${BUILD_STATUS}
```

**รันจริง:**

```bash
chmod +x build.sh
./build.sh step771calc
```

**ผลลัพธ์จริง:**

```
==> Compiling step771calc.cob ...
==> BUILD OK: step771calc
```

(exit code ของ script คือ `0` — ตรวจสอบได้ด้วย `echo $?` หลังรัน)

### อธิบายจุดสำคัญ

- `set -u`: บังคับให้ shell แจ้ง error ทันทีถ้าอ้างอิงตัวแปรที่ไม่ได้กำหนดค่า (ป้องกันบั๊กเงียบจาก
  การพิมพ์ชื่อตัวแปรผิด)
- `cobc -x -o "${PROGRAM}" "${PROGRAM}.cob"`: คำสั่งคอมไพล์มาตรฐานเดียวกับที่ใช้มาตลอดหลักสูตร
  เพียงแต่ถูก**ห่อหุ้มเป็นสคริปต์ที่รับชื่อโปรแกรมเป็นพารามิเตอร์**แทนที่จะพิมพ์ตรง ๆ ทุกครั้ง
- `BUILD_STATUS=$?`: เก็บ **exit code** ของคำสั่ง `cobc` ก่อนหน้าไว้ทันที (ต้องเก็บทันทีหลังคำสั่ง
  เพราะ `$?` จะถูกเขียนทับด้วยคำสั่งถัดไปเสมอ) — `cobc` คืนค่า `0` เมื่อคอมไพล์สำเร็จ และค่าอื่นที่
  ไม่ใช่ `0` เมื่อล้มเหลว ตามธรรมเนียม Unix/Linux ทั่วไป
- `exit ${BUILD_STATUS}`: ทำให้ script เองส่งต่อ exit code เดียวกันออกไป — สำคัญมากเพราะ **ระบบ CI
  ทุกตัวตัดสินความสำเร็จ/ล้มเหลวของแต่ละขั้นตอนจาก exit code นี้เท่านั้น** ไม่ได้อ่านข้อความที่
  พิมพ์ออกมาเลย

### ข้อควรระวัง

- ถ้าลืม `exit ${BUILD_STATUS}` ตอนท้าย สคริปต์จะ**คืนค่า exit code ของคำสั่งสุดท้ายที่รันเสมอ**
  (ในที่นี้คือคำสั่ง `if`/`echo` ซึ่งมักจะเป็น `0` เสมอ) ทำให้ script รายงานว่า "สำเร็จ" แม้ตอน compile
  จะ fail จริง ๆ ก็ตาม — เป็นบั๊กที่พบบ่อยมากเวลาเขียน CI script เอง
- ชื่อไฟล์และชื่อโปรแกรมต้องสอดคล้องกัน (`${PROGRAM}.cob` กับ `-o ${PROGRAM}`) — ถ้าโปรเจกต์มี
  โครงสร้างโฟลเดอร์ซับซ้อนกว่านี้ (เช่น src/, bin/ แยกกัน) ต้องปรับ path ให้ตรงกับความเป็นจริงเสมอ

### แบบฝึกหัดที่ 772.1

**โจทย์**: จงแก้ไข `build.sh` ให้ตรวจสอบก่อนว่าไฟล์ `${PROGRAM}.cob` มีอยู่จริงหรือไม่ก่อนเรียก
`cobc` และแสดงข้อความที่ชัดเจนถ้าไม่พบไฟล์

**เฉลย**:

```bash
#!/bin/bash
set -u
PROGRAM="$1"

if [ ! -f "${PROGRAM}.cob" ]; then
    echo "==> ERROR: source file ${PROGRAM}.cob not found"
    exit 2
fi

echo "==> Compiling ${PROGRAM}.cob ..."
cobc -x -o "${PROGRAM}" "${PROGRAM}.cob"
BUILD_STATUS=$?

if [ ${BUILD_STATUS} -eq 0 ]; then
    echo "==> BUILD OK: ${PROGRAM}"
else
    echo "==> BUILD FAILED: ${PROGRAM} (cobc exit code ${BUILD_STATUS})"
fi

exit ${BUILD_STATUS}
```

การเพิ่มการตรวจสอบไฟล์ก่อน (`[ ! -f ... ]`) ช่วยให้ error message ชัดเจนกว่าปล่อยให้ `cobc` แจ้ง
error เอง (ซึ่งบางครั้งข้อความ error ของ compiler อาจกำกวมสำหรับกรณี "หาไฟล์ไม่เจอ" เทียบกับ
"ไวยากรณ์ผิด")

---

## ขั้นตอนที่ 773: เขียน Test Script — เทียบผลลัพธ์จริงกับผลลัพธ์ที่คาดหวัง

### แนวคิด Golden Output Testing

เทคนิคการทดสอบที่ง่ายและตรงไปตรงมาที่สุดสำหรับโปรแกรม COBOL แบบ batch (ไม่มี interactive input)
คือ **Golden Output Testing** (บางครั้งเรียก Snapshot Testing หรือ Approval Testing): เก็บผลลัพธ์
ที่ถูกต้อง (ที่มนุษย์ตรวจสอบแล้วว่าถูก) ไว้ในไฟล์หนึ่งไฟล์ (เรียกว่า "golden file" หรือ "expected
output") แล้วให้สคริปต์**รันโปรแกรมจริง จับผลลัพธ์ แล้วเทียบกับไฟล์นั้นด้วยคำสั่ง `diff`**

### สร้างไฟล์ expected output

```bash
echo "INTEREST=0000500.00" > expected_output.txt
```

### สร้าง test.sh

```bash
#!/bin/bash
# test.sh - run a compiled COBOL program and compare its output to an
# expected-output file. Prints PASS or FAIL and exits 0/1 accordingly.
set -u
PROGRAM="$1"
EXPECTED_FILE="$2"

echo "==> Running ./${PROGRAM} ..."
./"${PROGRAM}" > actual_output.txt
RUN_STATUS=$?

if [ ${RUN_STATUS} -ne 0 ]; then
    echo "==> TEST FAIL: ${PROGRAM} crashed (exit code ${RUN_STATUS})"
    exit 1
fi

if diff -u "${EXPECTED_FILE}" actual_output.txt > diff_output.txt; then
    echo "==> TEST PASS: ${PROGRAM} output matches ${EXPECTED_FILE}"
    exit 0
else
    echo "==> TEST FAIL: ${PROGRAM} output does not match ${EXPECTED_FILE}"
    echo "----- diff (expected vs actual) -----"
    cat diff_output.txt
    exit 1
fi
```

**รันจริง:**

```bash
chmod +x test.sh
./test.sh step771calc expected_output.txt
```

**ผลลัพธ์จริง:**

```
==> Running ./step771calc ...
==> TEST PASS: step771calc output matches expected_output.txt
```

### อธิบายจุดสำคัญ

- `./"${PROGRAM}" > actual_output.txt`: รันโปรแกรม COBOL จริง แล้ว**redirect ผลลัพธ์**ที่ควรจะขึ้น
  หน้าจอ (จาก `DISPLAY`) ไปเก็บไว้ในไฟล์แทน เพื่อนำไปเทียบในขั้นตอนถัดไป
- ตรวจสอบ `RUN_STATUS` ก่อนเทียบผลลัพธ์เสมอ: ถ้าโปรแกรม crash (เช่น Segmentation Fault จากกับดัก
  Part 031 ขั้นตอนที่ 304) จะไม่มีประโยชน์ที่จะไปเทียบผลลัพธ์ต่อเลย ควรรายงาน FAIL ทันที
- `diff -u "${EXPECTED_FILE}" actual_output.txt`: `-u` คือ unified diff format ที่อ่านง่าย แสดงบรรทัด
  ที่ต่างกันด้วยเครื่องหมาย `-` (ค่าที่คาดหวัง) และ `+` (ค่าที่ได้จริง) — `diff` คืนค่า exit code `0`
  ถ้าไฟล์เหมือนกันทุกประการ และ `1` ถ้าต่างกัน ซึ่งนำมาใช้ตัดสินใจ PASS/FAIL ได้โดยตรงผ่าน `if`

### ข้อควรระวัง

- **ระวังเรื่อง trailing newline และช่องว่างท้ายบรรทัด**: `DISPLAY` ใน COBOL จะเติม newline ท้าย
  บรรทัดเสมอ แต่ถ้าไฟล์ `expected_output.txt` ถูกสร้างด้วยวิธีอื่น (เช่น copy-paste จาก editor บาง
  ตัว) อาจมีหรือไม่มี trailing newline ต่างกัน ทำให้ `diff` รายงานว่าต่างกันทั้งที่ตาเปล่ามองไม่เห็น
  ความต่างเลย — วิธีที่ปลอดภัยที่สุดคือสร้างไฟล์ expected ด้วยการรันโปรแกรมจริงครั้งแรกแล้วตรวจสอบ
  ผลลัพธ์ด้วยมือ (`./program > expected_output.txt` แล้วอ่านตรวจสอบ) แทนที่จะพิมพ์ไฟล์ expected เอง
- Golden Output Testing เหมาะกับโปรแกรมที่ผลลัพธ์ deterministic เท่านั้น (ตามที่กล่าวในขั้นตอนที่
  771) ถ้าโปรแกรมมีการพิมพ์ timestamp หรือค่าที่เปลี่ยนแปลงทุกครั้ง ต้องกรอง (filter) ส่วนนั้นออก
  ก่อนเทียบ หรือแยกทดสอบเฉพาะ subprogram ที่เป็น pure logic แทน (แนวทางนี้จะสอนเต็มรูปแบบใน Part 079)

### แบบฝึกหัดที่ 773.1

**โจทย์**: จงอธิบายว่าทำไม script ต้องตรวจสอบ `RUN_STATUS` (exit code ของการรันโปรแกรม) แยกต่างหาก
จากการเทียบผลลัพธ์ด้วย `diff` แทนที่จะปล่อยให้ไปเทียบผลลัพธ์ตรง ๆ เลย

**เฉลย**: ถ้าโปรแกรม COBOL crash กลางคัน (เช่น SIGSEGV) มันอาจจะทันพิมพ์ผลลัพธ์บางส่วนออกมาก่อน
crash ก็ได้ ซึ่งอาจบังเอิญตรงกับ `expected_output.txt` บางส่วนหรือไม่ตรงเลยก็ได้แล้วแต่กรณี ถ้า
script ไปเทียบผลลัพธ์ด้วย `diff` ทันทีโดยไม่ตรวจ exit code ก่อน อาจได้ผลลัพธ์ที่สับสน (เช่น diff
บอกว่า "ตรงกัน" ทั้งที่โปรแกรม crash ไปแล้วจริง ๆ ถ้าผลลัพธ์บางส่วนที่พิมพ์ทันบังเอิญตรงกัน) การ
ตรวจสอบ exit code ก่อนเสมอทำให้ได้ข้อความ error ที่ตรงประเด็นและชัดเจนกว่า ("โปรแกรม crash" ต่างจาก
"ผลลัพธ์ไม่ตรง" เป็นสาเหตุคนละแบบที่ทีมพัฒนาต้องแก้ไขต่างวิธีกัน)

---

## ขั้นตอนที่ 774: รวม Build และ Test เป็น Pipeline เดียว — กรณีที่ผ่านจริง

### สร้าง pipeline.sh

```bash
#!/bin/bash
# pipeline.sh - a minimal local CI pipeline: build, then test.
# Stops immediately (like a real CI job) if the build stage fails.
set -u
PROGRAM="$1"
EXPECTED_FILE="$2"

./build.sh "${PROGRAM}"
if [ $? -ne 0 ]; then
    echo "==> PIPELINE STOPPED: build stage failed, skipping tests."
    exit 1
fi

./test.sh "${PROGRAM}" "${EXPECTED_FILE}"
TEST_STATUS=$?

if [ ${TEST_STATUS} -eq 0 ]; then
    echo "==> PIPELINE RESULT: SUCCESS"
else
    echo "==> PIPELINE RESULT: FAILURE"
fi

exit ${TEST_STATUS}
```

**รันจริง:**

```bash
chmod +x pipeline.sh
./pipeline.sh step771calc expected_output.txt
```

**ผลลัพธ์จริง (ทดสอบรันแล้ว):**

```
==> Compiling step771calc.cob ...
==> BUILD OK: step771calc
==> Running ./step771calc ...
==> TEST PASS: step771calc output matches expected_output.txt
==> PIPELINE RESULT: SUCCESS
```

`echo $?` หลังจากนี้จะได้ `0` — นี่คือสิ่งที่ CI service (ไม่ว่าจะเป็น GitHub Actions, GitLab CI,
Jenkins) ใช้ตัดสินว่างานนี้ **"เขียว" (green/passing)** และอนุญาตให้ merge Pull Request ได้

### อธิบายจุดสำคัญ

- Pipeline นี้จำลอง**หลักการสำคัญที่สุดของ CI ที่แท้จริง**: ถ้าขั้นตอนก่อนหน้าล้มเหลว **ให้หยุดทันที**
  ไม่ต้องเสียเวลารันขั้นตอนถัดไป (`if [ $? -ne 0 ]; then ... exit 1; fi` หลัง `build.sh`) — หลักการนี้
  เรียกว่า **"Fail Fast"** ช่วยประหยัดเวลาและทรัพยากรของระบบ CI จริง (ซึ่งมักคิดค่าใช้จ่ายตามเวลาที่ใช้)
- การแยก `build.sh` และ `test.sh` เป็นคนละไฟล์ (แทนที่จะรวมเป็นไฟล์เดียว) ทำให้แต่ละสคริปต์**ทดสอบ
  และเรียกใช้แยกกันได้** เช่น นักพัฒนาอาจต้องการรันแค่ `build.sh` ตอนพัฒนาเพื่อเช็ค syntax เร็ว ๆ
  โดยไม่ต้องรันชุดทดสอบเต็มทุกครั้ง

### ข้อควรระวัง

- ในระบบ CI จริง แนวคิด "Fail Fast" นี้มักถูกเรียกว่า **stage** หรือ **job dependency** — ขั้นตอนที่
  778 (GitHub Actions YAML) จะแสดงให้เห็นว่าแนวคิดเดียวกันนี้ถูกนำไปเขียนเป็น syntax ของ GitHub
  Actions อย่างไร
- อย่าลืมว่าทุกครั้งที่รัน pipeline ไฟล์ `actual_output.txt` และ `diff_output.txt` จะถูกเขียนทับ —
  ถ้าต้องการเก็บประวัติผลการทดสอบแต่ละครั้งไว้ดู ต้อง copy ไฟล์เหล่านี้ไปเก็บที่อื่นก่อน pipeline
  รอบถัดไปจะเขียนทับ

### แบบฝึกหัดที่ 774.1

**โจทย์**: จงอธิบายว่าทำไมหลักการ "Fail Fast" (หยุดทันทีเมื่อขั้นตอนก่อนหน้าล้มเหลว) ถึงสำคัญเป็น
พิเศษกับโปรเจกต์ COBOL ที่มักมีโปรแกรมย่อยหลายสิบไฟล์ที่ต้องคอมไพล์รวมกัน

**เฉลย**: ถ้าโปรแกรมหลักคอมไพล์ไม่ผ่าน (เช่น syntax error) การพยายามรัน test ต่อไปจะไม่มีความหมาย
เลยเพราะไม่มีไฟล์ execute ให้รันด้วยซ้ำ — `test.sh` จะพยายามรัน `./program` ที่ไม่มีอยู่จริง แล้วได้
error ที่**ไม่ตรงประเด็น** (เช่น "command not found" แทนที่จะเป็น "compile error") ทำให้นักพัฒนาเสีย
เวลาสับสนว่าปัญหาจริง ๆ คืออะไร ยิ่งในโปรเจกต์ COBOL ที่มักมีหลายโปรแกรมเชื่อมโยงกันผ่าน `CALL`
(ทบทวน Part 031) การคอมไพล์ไฟล์หนึ่งไฟล์ล้มเหลวอาจทำให้ไม่สามารถ link โปรแกรมทั้งชุดได้เลย การหยุด
ทันทีตั้งแต่ build stage ทำให้ error message ที่ทีมเห็นตรงประเด็นและแก้ไขได้เร็วที่สุด

---

## ขั้นตอนที่ 775: พิสูจน์ Pipeline จริง — จำลองความล้มเหลว 2 รูปแบบ

### รูปแบบที่ 1: Build ผ่าน แต่ผลลัพธ์ผิด (Output Mismatch)

สมมติมีคนแก้ไขอัตราดอกเบี้ยในโปรแกรมโดยไม่ได้ตั้งใจ (บั๊กจริงที่เกิดขึ้นบ่อยในโลกจริง):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP775BUG.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRINCIPAL             PIC 9(7)V99 VALUE 10000.00.
       01  WS-RATE                  PIC 9V9999 VALUE 0.0800.
       01  WS-INTEREST              PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-INTEREST ROUNDED = WS-PRINCIPAL * WS-RATE.
           DISPLAY "INTEREST=" WS-INTEREST.
           STOP RUN.
```

(สังเกตว่าอัตราดอกเบี้ยถูกเปลี่ยนจาก `0.0500` เป็น `0.0800` โดยไม่ได้ตั้งใจ)

**รันจริงผ่าน pipeline.sh:**

```bash
./pipeline.sh step775bug expected_output.txt
```

**ผลลัพธ์จริง (ยืนยันด้วยการรัน — build ผ่าน แต่ test ล้มเหลว):**

```
==> Compiling step775bug.cob ...
==> BUILD OK: step775bug
==> Running ./step775bug ...
==> TEST FAIL: step775bug output does not match expected_output.txt
----- diff (expected vs actual) -----
--- expected_output.txt	2026-09-28 19:14:35.386311828 +0000
+++ actual_output.txt	2026-09-28 19:14:47.116210568 +0000
@@ -1 +1 @@
-INTEREST=0000500.00
+INTEREST=0000800.00
==> PIPELINE RESULT: FAILURE
```

**exit code ที่ได้คือ `1`** — นี่คือสิ่งที่ทำให้ GitHub Actions หรือ CI service ใด ๆ แสดงเครื่องหมาย
กากบาทสีแดง (❌) บน Pull Request ทันที แม้โค้ดจะคอมไพล์ผ่านสมบูรณ์แบบก็ตาม

### รูปแบบที่ 2: Build ล้มเหลวตั้งแต่ต้น (Compile Error)

สมมติมีการพิมพ์ชื่อตัวแปรผิด (ลืมประกาศ หรือพิมพ์ผิด):

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP775SYNTAXERR.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-PRINCIPAL             PIC 9(7)V99 VALUE 10000.00.
       01  WS-INTEREST              PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           COMPUTE WS-INTEREST ROUNDED = WS-PRINCIPAL * WS-RATEXX.
           DISPLAY "INTEREST=" WS-INTEREST.
           STOP RUN.
```

(`WS-RATEXX` ไม่เคยถูกประกาศไว้ใน `WORKING-STORAGE SECTION` เลย — พิมพ์ผิดจาก `WS-RATE`)

**รันจริงผ่าน pipeline.sh:**

```bash
./pipeline.sh step775syntaxerr expected_output.txt
```

**ผลลัพธ์จริง (build ล้มเหลว ทำให้ pipeline หยุดทันทีตามหลัก Fail Fast จากขั้นตอนที่ 774):**

```
==> Compiling step775syntaxerr.cob ...
step775syntaxerr.cob: in paragraph 'MAIN-PARA':
step775syntaxerr.cob:12: error: 'WS-RATEXX' is not defined
==> BUILD FAILED: step775syntaxerr (cobc exit code 1)
==> PIPELINE STOPPED: build stage failed, skipping tests.
```

### อธิบายจุดสำคัญ

- ทั้งสองกรณีทำให้ pipeline จบด้วย **exit code ที่ไม่ใช่ 0** แต่**สาเหตุต่างกันโดยสิ้นเชิง** และ
  ข้อความที่แสดงออกมาก็บอกสาเหตุที่แตกต่างกันชัดเจน — นี่คือประโยชน์ของการแยก build stage กับ
  test stage ออกจากกัน (ตามที่อธิบายในขั้นตอนที่ 774)
- สังเกตว่าในกรณีที่ 1 (`step775bug`) **`cobc` ไม่แจ้ง error ใด ๆ เลย** เพราะ `0.0800` เป็นค่า
  ตัวเลขที่ถูกต้องตามไวยากรณ์ COBOL ทุกประการ — นี่คือเหตุผลที่ **automated test (ไม่ใช่แค่ compiler)
  จำเป็นอย่างยิ่ง** เพราะ compiler ตรวจสอบได้แค่ไวยากรณ์ ไม่ได้ตรวจสอบว่า "ตรรกะทางธุรกิจถูกต้องหรือไม่"

### ข้อควรระวัง

- อย่าพึ่งพา compiler เพียงอย่างเดียวเป็นเกราะป้องกันบั๊ก — กรณีที่ 1 พิสูจน์ชัดเจนว่าโค้ดที่ผิด
  ทางตรรกะ (business logic) แต่ถูกไวยากรณ์ 100% จะผ่าน `cobc` ไปได้อย่างราบรื่นเสมอ
- ทีมที่ไม่มี automated test ที่ครอบคลุมเพียงพอ อาจไม่มีทางรู้เลยว่าเกิดบั๊กแบบกรณีที่ 1 ขึ้น จนกว่า
  จะมีคนไปสังเกตเห็นตัวเลขที่ผิดในรายงานจริง (ซึ่งอาจสายเกินไปแล้วถ้าเป็นระบบการเงิน)

### แบบฝึกหัดที่ 775.1

**โจทย์**: จากสองกรณีความล้มเหลวข้างต้น จงอธิบายว่ากรณีไหน "อันตรายกว่า" ในทางปฏิบัติ และเพราะเหตุใด

**เฉลย**: กรณีที่ 1 (output mismatch จากอัตราดอกเบี้ยผิด) **อันตรายกว่ามาก** แม้จะดู "ไม่รุนแรง" เท่า
กรณีที่ 2 เพราะกรณีที่ 2 (compile error) จะถูกจับได้ทันทีตั้งแต่ขั้นตอน build — ไม่มีทางที่โค้ดที่
คอมไพล์ไม่ผ่านจะหลุดรอดไปถึง production ได้เลย ในขณะที่กรณีที่ 1 **คอมไพล์ผ่านสมบูรณ์แบบ** และถ้าไม่มี
automated test ที่เทียบผลลัพธ์ไว้ล่วงหน้า โค้ดที่คำนวณดอกเบี้ยผิดนี้**อาจหลุดรอดไปถึงระบบ production
ได้จริง** ซึ่งในบริบทของระบบการเงิน (ตามที่ Part 001 อธิบายว่า COBOL เป็นแกนหลักของระบบธนาคาร) การ
คำนวณดอกเบี้ยผิดแม้เพียงเล็กน้อยสามารถสร้างความเสียหายมหาศาลเมื่อคูณด้วยจำนวนธุรกรรมนับล้านรายการ
นี่คือเหตุผลที่แท้จริงที่ automated testing (ไม่ใช่แค่การคอมไพล์สำเร็จ) เป็นหัวใจของ CI

---

## ขั้นตอนที่ 776: Multi-Program Build ด้วย Makefile

### ทำไมต้องใช้ Makefile สำหรับโปรเจกต์ที่มีหลายโปรแกรม

ทบทวนจาก Part 031: โปรแกรม COBOL ที่ใช้ `CALL` ต้องคอมไพล์**หลายไฟล์รวมกัน**ในคำสั่งเดียว
(`cobc -x main.cob sub.cob -o program`) เมื่อโปรเจกต์ใหญ่ขึ้น การจำว่าโปรแกรมไหนต้อง link กับไฟล์
อะไรบ้างจะซับซ้อนมาก **Makefile** คือเครื่องมือมาตรฐานของโลก Unix/Linux ที่ใช้บันทึกกฎการ build
เหล่านี้ไว้เป็นไฟล์เดียว

### โปรแกรมตัวอย่าง: main + subprogram

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP776MAIN.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-AMOUNT                PIC 9(7)V99 VALUE 2000.00.
       01  WS-TAX                   PIC 9(7)V99 VALUE 0.

       PROCEDURE DIVISION.
       MAIN-PARA.
           CALL "STEP776TAX" USING WS-AMOUNT WS-TAX.
           DISPLAY "AMOUNT=" WS-AMOUNT " TAX=" WS-TAX.
           STOP RUN.
```

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. STEP776TAX.
       AUTHOR. COBOL-COURSE.

       DATA DIVISION.
       LINKAGE SECTION.
       01  LK-AMOUNT                PIC 9(7)V99.
       01  LK-TAX                   PIC 9(7)V99.

       PROCEDURE DIVISION USING LK-AMOUNT LK-TAX.
       SUB-MAIN-PARA.
           COMPUTE LK-TAX ROUNDED = LK-AMOUNT * 0.07.
           GOBACK.
```

### Makefile

```makefile
# Makefile - builds the multi-program COBOL pipeline demo and runs its test.
PROGRAM = step776main
OBJS    = step776main.cob step776tax.cob

.PHONY: all build test clean

all: build test

build:
	cobc -x -o $(PROGRAM) $(OBJS)

test: build
	./$(PROGRAM) > actual_output_776.txt
	diff -u expected_output_776.txt actual_output_776.txt && echo "MAKE TEST: PASS"

clean:
	rm -f $(PROGRAM) *.o actual_output_776.txt
```

**รันจริง (`expected_output_776.txt` มีข้อความ `AMOUNT=0002000.00 TAX=0000140.00`):**

```bash
make
```

**ผลลัพธ์จริง:**

```
cobc -x -o step776main step776main.cob step776tax.cob
./step776main > actual_output_776.txt
diff -u expected_output_776.txt actual_output_776.txt && echo "MAKE TEST: PASS"
MAKE TEST: PASS
```

### อธิบายจุดสำคัญ

- `.PHONY: all build test clean`: บอก `make` ว่า `all`, `build`, `test`, `clean` เป็น**ชื่อคำสั่ง**
  ไม่ใช่ชื่อไฟล์จริงที่จะถูกสร้างขึ้น (ป้องกันปัญหาถ้าบังเอิญมีไฟล์ชื่อ `test` อยู่ในโฟลเดอร์เดียวกัน)
- `test: build`: บอกว่าเป้าหมาย `test` **ขึ้นอยู่กับ** (depends on) เป้าหมาย `build` เสมอ — `make`
  จะรัน `build` ให้ก่อนโดยอัตโนมัติทุกครั้งที่สั่ง `make test` ทำให้ไม่มีทางลืมคอมไพล์ก่อนรันทดสอบ
- **ข้อควรระวังเรื่อง Tab**: บรรทัดคำสั่งใต้แต่ละเป้าหมาย (เช่น `cobc -x ...`) **ต้องขึ้นต้นด้วย
  ตัวอักษร Tab เท่านั้น** ห้ามใช้ Space แทนเด็ดขาด — เป็นกับดักคลาสสิกของ Makefile ที่ทำให้เกิด
  error `*** missing separator` ถ้า editor แปลง Tab เป็น Space อัตโนมัติโดยไม่รู้ตัว

### ข้อควรระวัง

- `Makefile` เหมาะกับการ build โปรเจกต์ที่มีหลายไฟล์และมี dependency ระหว่างกันชัดเจน แต่สำหรับ
  pipeline ที่ซับซ้อนกว่านี้มาก (หลายสิบโปรแกรม, ต้องมี logic การเลือกไฟล์แบบมีเงื่อนไข) ทีมจำนวนมาก
  เลือกใช้ shell script (แบบขั้นตอนที่ 772–775) หรือเครื่องมือ build เฉพาะทางแทน เพราะ syntax ของ
  Makefile ที่ซับซ้อนขึ้นอาจอ่านยากกว่า shell script ธรรมดา
- อย่าลืม `make clean` เพื่อล้างไฟล์ execute เก่าก่อน commit เข้า git (เชื่อมโยงกับ `.gitignore`
  ที่จะสอนใน Part 080)

### แบบฝึกหัดที่ 776.1

**โจทย์**: จงเพิ่มเป้าหมายใหม่ชื่อ `rebuild` ใน Makefile ข้างต้น ที่ทำหน้าที่ `clean` แล้วตามด้วย
`build` ทันทีในคำสั่งเดียว

**เฉลย**:

```makefile
.PHONY: all build test clean rebuild

rebuild: clean build
```

เมื่อรัน `make rebuild`, `make` จะรัน `clean` ก่อน (ลบไฟล์ execute เก่าทิ้ง) แล้วตามด้วย `build`
โดยอัตโนมัติตามลำดับ dependency ที่ระบุไว้ (`clean build` หลังเครื่องหมาย `:`) เป็นประโยชน์เวลาต้องการ
บังคับให้คอมไพล์ใหม่ทั้งหมดโดยไม่พึ่งพา timestamp ของไฟล์ที่ `make` ใช้ตัดสินใจตามปกติว่าต้อง
build ใหม่หรือไม่

---

## ขั้นตอนที่ 777: แนวคิดพื้นฐานของ GitHub Actions

### GitHub Actions คืออะไร

**GitHub Actions** คือบริการ CI/CD ที่มากับ GitHub โดยตรง ทำงานโดยอ่านไฟล์ **YAML** ที่วางไว้ใน
โฟลเดอร์พิเศษ `.github/workflows/` ของ repository แล้วรันคำสั่งที่ระบุไว้บน **"runner"** (เครื่อง
เสมือนที่ GitHub เตรียมไว้ให้ฟรีในปริมาณจำกัดสำหรับ repository สาธารณะ) โดยอัตโนมัติทุกครั้งที่มี
เหตุการณ์ที่กำหนดไว้เกิดขึ้น (เช่น push โค้ด, เปิด Pull Request)

### โครงสร้างพื้นฐานของไฟล์ YAML สำหรับ Workflow

| ส่วนประกอบ | ความหมาย |
|---|---|
| `name` | ชื่อของ workflow ที่จะแสดงบนหน้า GitHub |
| `on` | เหตุการณ์ที่จะ trigger ให้ workflow นี้ทำงาน (เช่น `push`, `pull_request`) |
| `jobs` | รายการงานที่จะรัน แต่ละงานรันบน runner แยกกัน (ค่าเริ่มต้นรันพร้อมกัน) |
| `runs-on` | ระบุชนิดของเครื่อง runner ที่จะใช้ (เช่น `ubuntu-latest`) |
| `steps` | ลำดับคำสั่งที่จะรันภายในแต่ละ job (รันตามลำดับบนสุดลงล่าง) |
| `uses` | เรียกใช้ "Action" สำเร็จรูปที่คนอื่นเขียนไว้แล้ว (เช่น `actions/checkout`) |
| `run` | รันคำสั่ง shell ตรง ๆ |

### ความสัมพันธ์กับสิ่งที่เราสร้างมาแล้ว

**ข่าวดี**: GitHub Actions ไม่ได้ต้องการอะไรที่พิเศษไปกว่าสิ่งที่เราสร้างไว้แล้วในขั้นตอนที่ 772–775
เลย — YAML เป็นเพียง**เปลือกที่บอก runner ว่าให้ติดตั้ง GnuCOBOL แล้วรัน `build.sh`/`test.sh` ของเรา**
เท่านั้น ตรรกะการคอมไพล์และทดสอบทั้งหมดยังคงเป็น shell script เดิมที่เราทดสอบรันจริงมาแล้วทุกประการ

### ข้อควรระวัง

- อย่าเข้าใจผิดว่า "ต้องเขียน logic การทดสอบใหม่หมดเป็นภาษา YAML" — YAML ทำหน้าที่แค่**ประสานงาน**
  (orchestration): บอกว่าจะรันอะไร เมื่อไหร่ บนเครื่องแบบไหน ส่วน**ตรรกะจริง**ยังคงเป็น shell
  script/COBOL เหมือนเดิม
- `on:` เป็นคำสงวนใน YAML บางเวอร์ชันที่อาจถูกตีความเป็นค่า boolean (`true`) โดย YAML parser บาง
  ตัวถ้าไม่ใส่เครื่องหมาย quote — GitHub Actions มี parser ของตัวเองที่จัดการเรื่องนี้ได้ถูกต้อง
  แต่ถ้านำไฟล์ไปประมวลผลด้วยเครื่องมือ YAML ทั่วไปอื่น ๆ ควรระวังจุดนี้ไว้

### แบบฝึกหัดที่ 777.1

**โจทย์**: จงจับคู่คำสั่งที่เราเขียนมาแล้วในขั้นตอนที่ 772–775 (`build.sh`, `test.sh`) กับส่วนของ
GitHub Actions YAML ที่ควรจะเรียกใช้มัน

**เฉลย**: `build.sh` และ `test.sh` ควรถูกเรียกผ่าน `run:` ภายใน `steps:` ของ job หนึ่ง โดยแต่ละ
`run:` step หนึ่งอันจะเทียบเท่ากับการรันคำสั่งนั้นในเทอร์มินัลของ runner โดยตรง เช่น
`run: ./build.sh step771calc` และ `run: ./test.sh step771calc expected_output.txt` ตามลำดับ —
เหมือนกับที่เราพิมพ์คำสั่งเหล่านี้เองในเทอร์มินัลทุกประการ เพียงแต่ GitHub Actions เป็นผู้รันให้
อัตโนมัติแทนเราทุกครั้งที่มีการ push

---

## ขั้นตอนที่ 778: เขียนไฟล์ GitHub Actions Workflow ที่สมบูรณ์

### ไฟล์ .github/workflows/cobol-ci.yml

```yaml
name: COBOL CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Install GnuCOBOL
        run: |
          sudo apt-get update
          sudo apt-get install -y gnucobol4 || sudo apt-get install -y gnucobol

      - name: Build COBOL program
        run: |
          chmod +x build.sh test.sh
          ./build.sh step771calc

      - name: Run automated test
        run: ./test.sh step771calc expected_output.txt

      - name: Upload test artifacts
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: cobol-test-results
          path: |
            actual_output.txt
            diff_output.txt
```

> **หมายเหตุความซื่อตรง**: ไฟล์ YAML นี้ผ่านการตรวจสอบว่าเป็น **YAML syntax ที่ถูกต้อง** ด้วย
> parser จริง (`python3` + `pyyaml`) แล้ว และ syntax ของแต่ละ key (`on`, `jobs`, `runs-on`, `steps`,
> `uses`, `run`) ตรงตามเอกสารทางการของ GitHub Actions ทุกประการ **แต่สภาพแวดล้อมที่ใช้เขียนหลักสูตร
> นี้ไม่มี GitHub runner จริงให้เรียกใช้** จึงไม่สามารถยืนยันได้ 100% ว่า workflow นี้จะรันผ่านบน
> GitHub จริงโดยไม่มีปัญหาเรื่องสภาพแวดล้อมอื่น ๆ (เช่น เวอร์ชัน Ubuntu ของ runner ที่เปลี่ยนไป หรือ
> ชื่อแพ็กเกจ `gnucobol4` ที่อาจเปลี่ยนตามเวอร์ชันของ distro) สิ่งที่ **ทดสอบรันจริงแล้ว 100%** คือ
> `build.sh` และ `test.sh` ที่ workflow นี้เรียกใช้ ซึ่งเป็นตรรกะเดียวกันทุกประการกับที่ runner
> จะรัน

### อธิบายทีละส่วน

- `on.push.branches` / `on.pull_request.branches`: กำหนดให้ workflow นี้ทำงานทุกครั้งที่มีคน push
  โค้ดเข้า branch `main` โดยตรง หรือทุกครั้งที่มีคนเปิด/อัปเดต Pull Request ที่จะ merge เข้า `main`
  — นี่คือหัวใจของ **Continuous Integration**: ทดสอบทุกครั้งที่มีการเปลี่ยนแปลงโค้ด ไม่ใช่แค่ตอน
  จะ deploy เท่านั้น
- `runs-on: ubuntu-latest`: เลือกเครื่อง runner ที่รัน Ubuntu Linux เวอร์ชันล่าสุดที่ GitHub เตรียม
  ไว้ให้ — เครื่องนี้เป็นเครื่องเปล่าใหม่ทุกครั้งที่ workflow รัน (ไม่มีการเก็บ state ข้ามรอบ) จึงต้อง
  ติดตั้ง GnuCOBOL ใหม่ทุกครั้งด้วย step `Install GnuCOBOL`
- `uses: actions/checkout@v4`: เรียกใช้ Action สำเร็จรูปอย่างเป็นทางการของ GitHub ที่ทำหน้าที่
  ดึงโค้ดจาก repository มาไว้บน runner ก่อน — จำเป็นเสมอเป็น step แรกของเกือบทุก workflow
- `sudo apt-get install -y gnucobol4 || sudo apt-get install -y gnucobol`: ใช้ `||` (OR) เพื่อลอง
  ติดตั้งชื่อแพ็กเกจ `gnucobol4` ก่อน ถ้าไม่พบให้ลองชื่อ `gnucobol` แทน — เป็นวิธีป้องกันความเสี่ยง
  จากชื่อแพ็กเกจที่อาจต่างกันไปตามเวอร์ชันของ Ubuntu ที่ runner ใช้
- `if: always()` บน step สุดท้าย: บังคับให้ step นี้ (อัปโหลดไฟล์ผลการทดสอบเป็น artifact ให้ดู
  ย้อนหลังได้) ทำงาน**เสมอ แม้ step ก่อนหน้าจะ fail ก็ตาม** — มีประโยชน์มากเพราะเมื่อ test ล้มเหลว
  เรายิ่งต้องการเห็นไฟล์ `diff_output.txt` เพื่อวิเคราะห์สาเหตุ ไม่ใช่แค่ตอนที่ทุกอย่างผ่านเท่านั้น

### ข้อควรระวัง

- ต้องวางไฟล์นี้ที่ path `.github/workflows/cobol-ci.yml` เป๊ะ ๆ (ชื่อโฟลเดอร์สะกดผิดแม้แต่ตัวเดียว
  GitHub จะไม่รู้จัก workflow นี้เลยโดยไม่มี error ใด ๆ แจ้งเตือน)
- indentation ใน YAML **มีความหมาย** เหมือนกับ Python — ต้องใช้ space สม่ำเสมอ (ห้ามผสม Tab)
  มิฉะนั้นจะเกิด parsing error ทันที

### แบบฝึกหัดที่ 778.1

**โจทย์**: จงอธิบายว่าทำไมการรัน `build.sh` และ `test.sh` ที่เราทดสอบจริงแล้วในขั้นตอนที่ 772–775
บนเครื่องของเราเอง ถึงยัง**ไม่รับประกัน 100%** ว่า workflow ใน GitHub Actions runner จะทำงานสำเร็จ
เหมือนกันทุกประการ

**เฉลย**: แม้ตรรกะของ script จะเหมือนกันทุกประการ แต่ **สภาพแวดล้อม (environment)** ของ GitHub
Actions runner อาจแตกต่างจากเครื่องที่เราทดสอบ เช่น เวอร์ชันของ GnuCOBOL ที่ `apt-get` ติดตั้งให้บน
runner อาจไม่ตรงกับเวอร์ชันที่ใช้ทดสอบ (`4.0-early-dev` ในหลักสูตรนี้), locale/encoding ของระบบอาจ
ต่างกัน, หรือชื่อแพ็กเกจ `gnucobol4`/`gnucobol` อาจถูกเปลี่ยนชื่อในอนาคต นี่คือเหตุผลที่ควรมีขั้นตอน
`Install GnuCOBOL` ที่ตรวจสอบและติดตั้งอย่างชัดเจนใน workflow เสมอ (ไม่พึ่งพาว่า runner "น่าจะมี"
เครื่องมือติดตั้งไว้แล้ว) และเป็นเหตุผลที่หลักสูตรนี้ระบุอย่างตรงไปตรงมาว่าสิ่งที่พิสูจน์แล้วจริง
100% คือการรันบนเครื่องทดสอบโดยตรง ไม่ใช่การรันผ่าน GitHub Actions runner จริง (ซึ่งเป็นประเด็นที่
กล่าวถึงในหัวข้อ "Environment Drift" ที่จะขยายความในขั้นตอนที่ 779)

---

## ขั้นตอนที่ 779: ข้อควรระวังเฉพาะทางของ CI/CD สำหรับ COBOL

### กับดักที่ 1: Column Sensitivity ในสคริปต์ที่ฝังโค้ด COBOL

ทบทวนกฎเหล็กจาก `docs/COURSE-OUTLINE.md`: GnuCOBOL แบบ Fixed-Format นับคอลัมน์ 72 เป็น**ไบต์**
ถ้าทีมมี script ที่สร้างไฟล์ COBOL จาก template (เช่น เพื่อ generate โปรแกรมซ้ำ ๆ อัตโนมัติใน CI)
ต้องระวังว่าตัวแปรที่ถูกแทรกเข้าไปในคอลัมน์ที่ใกล้ขอบ 72 อาจทำให้บรรทัดยาวเกินโดยไม่รู้ตัว
โดยเฉพาะถ้าเป็นการ generate จาก JSON/YAML config ที่ไม่ได้ตรวจสอบความยาวบรรทัดก่อน

### กับดักที่ 2: Environment Drift (สภาพแวดล้อมของ runner ต่างจากเครื่อง dev)

ตามที่กล่าวในขั้นตอนที่ 778 — เวอร์ชันของ `cobc` ที่ต่างกันแม้เพียงเล็กน้อยอาจมีพฤติกรรมต่างกันได้
โดยเฉพาะเรื่อง default compiler flag หรือ dialect ที่ใช้ (ทบทวน Part 049 เรื่อง Compiler
Directives) วิธีป้องกันที่ดีที่สุดคือ**ระบุเวอร์ชันของ image/container ที่ใช้ให้ตายตัว** (เช่น ใช้
Docker image ที่ build ไว้ล่วงหน้าจาก Part 076 แทนการ `apt-get install` แบบดึงเวอร์ชันล่าสุดเสมอ)

### กับดักที่ 3: Copybook Path ใน CI Runner

โปรแกรมที่ใช้ `COPY "customer.cpy"` (Part 033) ต้องการ compiler flag `-I <directory>` เพื่อบอก
path ที่จะค้นหาไฟล์ copybook ถ้า path นี้ถูก hardcode ไว้ตาม path บนเครื่อง dev ส่วนตัว (เช่น
`/home/somchai/project/copybooks`) แต่ runner ของ CI มี path ที่ต่างไปโดยสิ้นเชิง (`/home/runner/work/...`)
การ build จะล้มเหลวทันทีด้วย error "copybook not found" ควรใช้ **relative path** เทียบกับ root ของ
repository เสมอในสคริปต์ CI

### กับดักที่ 4: Locale และ Encoding

ระบบ COBOL จำนวนมากมีประวัติเกี่ยวข้องกับ EBCDIC (ตามที่กฎเหล็กของหลักสูตรอธิบาย) แม้ GnuCOBOL บน
Linux จะใช้ ASCII/UTF-8 เป็นปกติ แต่ถ้าทีมมีการแปลงไฟล์ข้อมูลระหว่าง encoding ต่างกันในขั้นตอน
CI/CD (เช่น รับไฟล์จากระบบ Mainframe จริงมาทดสอบ) ต้องระบุ locale ของ runner ให้ชัดเจนเสมอ ไม่พึ่ง
ค่า default ที่อาจเปลี่ยนแปลงได้ระหว่าง runner แต่ละรอบ

### ตารางสรุปกับดักและวิธีป้องกัน

| กับดัก | ผลกระทบ | วิธีป้องกัน |
|---|---|---|
| Column sensitivity ใน generated code | Compile error ที่ไม่คาดคิด | ตรวจสอบความยาวบรรทัดก่อน generate เสมอ |
| Environment drift | พฤติกรรมต่างกันระหว่าง dev กับ CI | Pin เวอร์ชัน compiler/ใช้ Docker image คงที่ (Part 076) |
| Copybook path hardcode | "copybook not found" บน CI | ใช้ relative path + `-I` flag เทียบ root repo |
| Locale/Encoding | ข้อมูลผิดเพี้ยนเงียบ ๆ | ระบุ locale ชัดเจนใน script/YAML |

### ข้อควรระวัง

- ปัญหาทั้ง 4 ข้อข้างต้นมีลักษณะร่วมกันคือ **"ทำงานได้ปกติบนเครื่อง dev ของตัวเอง แต่ล้มเหลวบน CI"**
  ซึ่งเป็นอาการคลาสสิกที่เรียกว่า **"works on my machine"** — นี่คือเหตุผลสำคัญที่สุดข้อหนึ่งที่
  ควรมี CI ตั้งแต่ต้นโปรเจกต์ เพราะมันบังคับให้ทีมต้องแก้ปัญหาเรื่องความสอดคล้องของสภาพแวดล้อมตั้งแต่
  เนิ่น ๆ แทนที่จะไปเจอตอน deploy จริง

### แบบฝึกหัดที่ 779.1

**โจทย์**: จงยกตัวอย่างการแก้ปัญหา "Environment Drift" ด้วยแนวคิดที่เคยเรียนมาแล้วใน Part 076
(Containerizing COBOL ด้วย Docker)

**เฉลย**: แทนที่จะให้ CI runner ติดตั้ง GnuCOBOL ใหม่ทุกครั้งด้วย `apt-get install` (ซึ่งอาจได้
เวอร์ชันที่ต่างกันไปในแต่ละครั้งที่ package repository อัปเดต) ทีมสามารถ build Docker image ที่มี
GnuCOBOL เวอร์ชันที่ต้องการติดตั้งไว้แน่นอนเรียบร้อยแล้ว (ตามที่ Part 076 สอน) แล้วให้ GitHub Actions
workflow ใช้ image นั้นเป็นสภาพแวดล้อมของ job โดยตรง (ผ่าน `container:` key ใน job) วิธีนี้รับประกัน
ว่าทุกครั้งที่ CI รัน จะใช้เวอร์ชัน compiler เดียวกันเป๊ะกับที่ทีมตั้งใจไว้ ไม่ขึ้นกับว่า package
repository ของ Ubuntu จะมีเวอร์ชันอะไรอยู่ในวันนั้น ๆ

---

## ขั้นตอนที่ 780: Capstone — Pipeline รายงานผลรวมหลายโปรแกรม

### เป้าหมาย

รวบยอดทุกเทคนิคจาก Part นี้เข้าด้วยกัน: สร้างสคริปต์ที่ build และ test **หลายโปรแกรม** แล้วสรุปผล
เป็นรายงานเดียว เหมือนที่ CI service จริงแสดงผลสรุปให้เห็นในหน้าเดียวหลังจบ pipeline

### ci_report.sh

```bash
#!/bin/bash
# ci_report.sh - capstone pipeline: build + test several COBOL programs
# and print a final PASS/FAIL summary report, like a CI job summary.
set -u
declare -A PROGRAMS
PROGRAMS[step771calc]=expected_output.txt

PASS_COUNT=0
FAIL_COUNT=0

echo "================ CI PIPELINE REPORT ================"
for PROGRAM in "${!PROGRAMS[@]}"; do
    EXPECTED_FILE="${PROGRAMS[$PROGRAM]}"
    ./build.sh "${PROGRAM}" > /tmp/build_log_$$.txt 2>&1
    if [ $? -ne 0 ]; then
        echo "[FAIL] ${PROGRAM} - build error"
        FAIL_COUNT=$((FAIL_COUNT+1))
        continue
    fi
    ./test.sh "${PROGRAM}" "${EXPECTED_FILE}" > /tmp/test_log_$$.txt 2>&1
    if [ $? -eq 0 ]; then
        echo "[PASS] ${PROGRAM}"
        PASS_COUNT=$((PASS_COUNT+1))
    else
        echo "[FAIL] ${PROGRAM} - output mismatch"
        FAIL_COUNT=$((FAIL_COUNT+1))
    fi
done
rm -f /tmp/build_log_$$.txt /tmp/test_log_$$.txt

echo "======================================================"
echo "TOTAL: ${PASS_COUNT} passed, ${FAIL_COUNT} failed"
if [ ${FAIL_COUNT} -eq 0 ]; then
    echo "OVERALL RESULT: SUCCESS"
    exit 0
else
    echo "OVERALL RESULT: FAILURE"
    exit 1
fi
```

**รันจริง:**

```bash
chmod +x ci_report.sh
./ci_report.sh
```

**ผลลัพธ์จริง (ทดสอบรันแล้ว):**

```
================ CI PIPELINE REPORT ================
[PASS] step771calc
======================================================
TOTAL: 1 passed, 0 failed
OVERALL RESULT: SUCCESS
```

(exit code สุดท้ายคือ `0` — ยืนยันด้วย `echo $?`)

### อธิบายจุดสำคัญ

- `declare -A PROGRAMS`: ประกาศ **associative array** ใน bash ที่เก็บคู่ "ชื่อโปรแกรม → ชื่อไฟล์
  expected output" — ทำให้เพิ่มโปรแกรมใหม่เข้า pipeline ได้ง่ายเพียงเพิ่มบรรทัดเดียว โดยไม่ต้อง
  แก้ไข loop logic เลย เป็นการออกแบบที่**ขยายได้ (scalable)** ไปสู่โปรเจกต์ที่มีหลายสิบโปรแกรมจริง
- `> /tmp/build_log_$$.txt 2>&1`: เก็บ log ของแต่ละโปรแกรมแยกเป็นไฟล์ชั่วคราว (`$$` คือ process ID
  ของ script เอง ทำให้ชื่อไฟล์ไม่ชนกันถ้ามีหลาย instance รันพร้อมกัน) เพื่อไม่ให้ log ของแต่ละ
  โปรแกรมปนกันในหน้าจอสรุปผล — ในระบบ CI จริง log แบบนี้มักถูกเก็บเป็น "artifact" ให้ดูย้อนหลังได้
  (เหมือน step `Upload test artifacts` ในขั้นตอนที่ 778)
- โครงสร้างสรุปผลท้ายสุด (`TOTAL: X passed, Y failed`) เป็นรูปแบบมาตรฐานที่ CI dashboard ส่วนใหญ่
  ใช้แสดงผล — การออกแบบให้ script ของเราพิมพ์รูปแบบคล้ายกันช่วยให้ทีมอ่านผลลัพธ์ได้ทันทีโดยไม่ต้อง
  เรียนรู้รูปแบบใหม่

### ภาพรวม Pipeline แบบเต็มที่ประกอบเข้าด้วยกันทั้งหมดใน Part นี้

```
git push / Pull Request (Part 080)
        │
        ▼
GitHub Actions runner เริ่มทำงาน (ขั้นตอนที่ 777-778)
        │
        ▼
Checkout โค้ด (actions/checkout)
        │
        ▼
ติดตั้ง GnuCOBOL บน runner
        │
        ▼
ci_report.sh: build.sh ทุกโปรแกรม (ขั้นตอนที่ 772, 776)
        │
   ┌────┴────┐
   ▼         ▼
สำเร็จ     ล้มเหลว → หยุดทันที (Fail Fast, ขั้นตอนที่ 774)
   │
   ▼
ci_report.sh: test.sh ทุกโปรแกรม (ขั้นตอนที่ 773, 775)
        │
        ▼
สรุปผล PASS/FAIL + exit code (ขั้นตอนที่ 780)
        │
        ▼
GitHub แสดง ✅ หรือ ❌ บน Pull Request
        │
        ▼
(CD ขั้นถัดไป: package + deploy — แนวคิดต่อยอดในเฟส 5 บทหลัง ๆ)
```

### ข้อควรระวัง

- Capstone script นี้ยังเป็นเวอร์ชันเรียบง่าย — ในโปรเจกต์จริงควรเพิ่มการรัน**ชุดทดสอบแบบ unit test**
  ที่ครอบคลุมกว่าการเทียบผลลัพธ์ทั้งโปรแกรม (Golden Output) เพียงอย่างเดียว ซึ่งเป็นหัวข้อที่ Part
  079 จะสอนแบบเจาะลึกต่อไปทันที
- การเก็บ log แยกไฟล์ด้วย `$$` ใช้ได้ดีสำหรับสคริปต์ทดสอบในเครื่อง แต่บน CI service จริงส่วนใหญ่มี
  ระบบจัดการ log ของตัวเองอยู่แล้ว (แสดงผลแบบ real-time พร้อม timestamp) จึงมักไม่จำเป็นต้องเขียน
  กลไกนี้เองซ้ำอีกเมื่อย้ายไปใช้ GitHub Actions จริง

### แบบฝึกหัดที่ 780.1

**โจทย์**: จงเพิ่มโปรแกรม `step776main` (จากขั้นตอนที่ 776) เข้าไปใน `ci_report.sh` โดยต้องคอมไพล์
รวมกับ `step776tax.cob` ด้วย — อธิบายว่าทำไมโครงสร้าง `declare -A PROGRAMS` แบบเดิมจึง**ไม่รองรับ**
กรณีนี้ได้ทันทีโดยไม่ต้องแก้ไขเพิ่มเติม

**เฉลย**: โครงสร้าง `declare -A PROGRAMS` เดิมและ `build.sh` ที่เขียนไว้ในขั้นตอนที่ 772 รองรับแค่
**โปรแกรมไฟล์เดียว** (`cobc -x -o "${PROGRAM}" "${PROGRAM}.cob"`) แต่ `step776main` ต้องการคอมไพล์
รวมกับ `step776tax.cob` อีกไฟล์หนึ่งด้วย (ตามที่ขั้นตอนที่ 776 อธิบาย) ดังนั้นต้องปรับ `build.sh`
ให้รับรายชื่อไฟล์ประกอบเพิ่มเติมได้ เช่น เปลี่ยนโครงสร้างข้อมูลเป็น
`PROGRAMS[step776main]="step776main.cob step776tax.cob|expected_output_776.txt"` แล้วแก้ loop ให้
แยกส่วน "รายชื่อไฟล์ source" กับ "ไฟล์ expected output" ออกจากกันด้วยเครื่องหมาย `|` ก่อนส่งต่อให้
`cobc` นี่คือตัวอย่างจริงว่าทำไม script อัตโนมัติที่เริ่มจากง่าย ๆ มักต้องถูกออกแบบใหม่เรื่อย ๆ
เมื่อโปรเจกต์เติบโตซับซ้อนขึ้น — เป็นเหตุผลที่ทีมขนาดใหญ่มักย้ายจาก shell script ไปใช้เครื่องมือ
build ที่ออกแบบมาให้รองรับ dependency ซับซ้อนแบบนี้โดยเฉพาะ (เช่น Makefile จากขั้นตอนที่ 776 หรือ
เครื่องมือ build เฉพาะทางของแต่ละ CI platform)

---

## สรุปท้ายบท

Part นี้พาคุณสร้าง CI/CD pipeline สำหรับโปรเจกต์ COBOL ตั้งแต่ต้นจนจบ โดยทุกสคริปต์**รันจริงและ
พิสูจน์ผลลัพธ์แล้วทั้งกรณีผ่านและกรณีล้มเหลว**:

- แนวคิด CI/CD และเหตุผลที่สำคัญเป็นพิเศษกับ COBOL เพราะ compiler ไม่ตรวจสอบพารามิเตอร์ข้ามไฟล์
- การเขียน `build.sh` ที่คอมไพล์โปรแกรมและส่งต่อ exit code อย่างถูกต้อง
- การเขียน `test.sh` ที่เทียบผลลัพธ์จริงกับ Golden Output ด้วย `diff`
- การรวม build+test เป็น `pipeline.sh` ตามหลัก Fail Fast
- การพิสูจน์ pipeline ด้วยการจำลองความล้มเหลว 2 แบบ: output mismatch และ compile error
- การใช้ Makefile จัดการ build ที่มีหลายโปรแกรมเชื่อมโยงกันผ่าน `CALL`
- แนวคิดพื้นฐานและโครงสร้างของ GitHub Actions YAML
- ไฟล์ GitHub Actions workflow ที่ถูกต้องตามไวยากรณ์จริง พร้อมคำเตือนที่ซื่อตรงเรื่องขอบเขตการทดสอบ
- กับดักเฉพาะทางของ CI/CD สำหรับ COBOL: column sensitivity, environment drift, copybook path,
  locale/encoding
- Capstone script ที่รวมหลายโปรแกรมเข้าเป็นรายงานสรุปผลเดียว

Part ถัดไป (**Part 079**) จะเจาะลึกสิ่งที่ Part นี้แตะไว้เพียงผิวเผิน (Golden Output Testing ทั้ง
โปรแกรม) ไปสู่ **Unit Testing ระดับ subprogram**: การออกแบบ subprogram ให้ทดสอบง่าย การเขียน test
driver ที่ assert ค่าทีละกรณี และการสร้างชุดทดสอบที่ครอบคลุม edge case ต่าง ๆ — เทคนิคที่จะทำให้
`ci_report.sh` ในขั้นตอนที่ 780 มีความหมายมากขึ้นไปอีกขั้น

**[← กลับไป Part 077](part-077-cobol-cloud-modernization.md)** | **[ไปยัง Part 079: Unit Testing สำหรับ COBOL →](part-079-unit-testing-cobol.md)**
