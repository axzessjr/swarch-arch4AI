# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** กฤตานน ยวงทอง

**Student ID / รหัสนิสิต:** 6970008621

This is **individual work**: each student completes and submits their own copy, in their own fork.

*งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง*

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|
| t0-large | 498 | 595 | 93.5% | 100% |
| t0-small | 13 | 28 | 72.5% | 0% |

**Q0.** What does the large model buy, and what does it cost?

*large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร*

> Large model ให้ความแม่นยำ (accuracy) ที่สูงขึ้นมากจาก 72.5% เป็น 93.5% จนผ่านเกณฑ์ QR3 (≥ 90%) แต่ต้องแลกด้วย latency ที่ช้าลงเกือบ 20–40 เท่า (p50 จาก 13 ms เป็น 498 ms และ p95 จาก 28 ms เป็น 595 ms) รวมถึงค่าใช้จ่ายและข้อจำกัดด้าน capacity เพราะต้องเรียกใช้ large model ถึง 100% ของทุกรีวิว

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
|---|---|---|---|---|---|---|
| t1-before | 26.5 | 240 | 442 | 20 | 0 | 92.8% |
| t1-after | 9.9 | 414 | 1008 | 0 | 22 | 92.7% |

**Q1.1** In `t1-before`, p50 can look *better* than the baseline. Why is p50 misleading here? Which column tells the real story?

*ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง*

> ค่า p50 (240 ms) ทำให้เข้าใจผิดเพราะคำนวณจากเฉพาะ requests ที่สำเร็จซึ่งทยอยเข้ามาแบบไม่ชนคิวกันหลังจากที่ requests อื่นๆ อีก 10% ค้างยาวไปถึง 20 วินาทีจน client ตัดจบไปก่อน คอลัมน์ที่บอกความจริงคือ `errors` (ล้มเหลวถึง 20 รายการ) และ `run time` ที่ใช้เวลารวมนานถึง 26.5 วินาที (จากปกติ ~10 วินาที)

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

*การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway*

> เพราะขณะที่ request ค้างรอถึง 20 วินาที มันยังคงยึดครองทรัพยากรสำคัญไว้ตลอดเวลา ทั้ง HTTP connection, worker/task slot และหน่วยความจำในทั้ง gateway และ review service (รวมถึงโควตา concurrency ของโมเดล) เมื่อมีการค้างสะสมจึงเกิดการบล็อกคอขวด (resource exhaustion) ที่ลามไปทำให้ request ปกติอื่นๆ ช้าหรือล่มตามไปด้วย

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

*รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น*

> รีวิวที่มีสถานะ `pending` ควรถูกเก็บไว้ในฐานข้อมูลหรือคิว แล้วใช้ background job / worker ดึงไปประมวลผลติด label ย้อนหลังเป็นชุดเมื่อระบบว่าง ซึ่งตรงกับสถาปัตยกรรมแบบ **Asynchronous / batch topology (T4)**

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

*เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที*

> ถ้าตั้งเป็น 0.1 วินาที (100 ms) จะสั้นกว่าเวลาประมวลผลปกติของ large model (~200 ms ต่อ call และ p50 อยู่ที่ 498 ms) ทำให้รีวิวปกติเกือบทั้งหมดถูกตัดจบเป็น `pending` โดยไม่จำเป็น แต่ถ้าตั้งเป็น 10 วินาที จะผิดข้อกำหนด QR1/QR2 (ที่ต้องตอบภายใน 1 วินาที) ทำให้ผู้ใช้ต้องรอนานและทรัพยากรของระบบถูกยึดค้างไว้จนรับโหลดไม่ไหว

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run | threshold | p50 ms | p95 ms | accuracy | large calls |
|---|---|---|---|---|---|
| t2-0.6 | 0.6 | 6 | 555 | 91.5% | 38% |
| t2-0.7 | 0.7 | 365 | 582 | 93.0% | 53% |
| t2-0.8 | 0.8 | 472 | 683 | 93.5% | 64% |
| t2-0.9 | 0.9 | 484 | 624 | 93.5% | 77% |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

*จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป*

> เลือก **threshold 0.7** เพราะให้ความแม่นยำสูงถึง 93.0% (ผ่านเกณฑ์ QR3 ≥ 90% และใกล้เคียง large model ที่ 93.5% มาก) ขณะที่ช่วยลดการเรียก large model ลงได้เกือบครึ่งหนึ่งเหลือ 53% และลด p50 ลงเหลือ 365 ms โดยเรายอมแลก (เสีย) **Accuracy** ไปเล็กน้อย (ลดลง 0.5% จาก 93.5% เหลือ 93.0%) เพื่อประหยัด Cost และเพิ่มความเร็ว

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

*ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก*

> เพราะรีวิวส่วนใหญ่เกือบครึ่งหนึ่งมีความชัดเจนจน small model ตอบได้ทันทีในเวลาไม่ถึง 1 ms จึงดึงค่ามัธยฐาน (p50) ลงมาได้อย่างชัดเจน (และยิ่งเห็นชัดเมื่อใช้ร่วมกับ cache หรือที่ threshold 0.6 ซึ่ง p50 เหลือ 6 ms) แต่กลุ่ม 5% ที่ช้าที่สุด (p95 / tail latency) ยังคงเป็นรีวิวที่ยากซึ่งต้องส่งต่อไปรอคิวประมวลผลที่ large model เสมอ ค่า p95 จึงยังคงอยู่ใกล้เคียงเดิมที่ 582 ms

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

*cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้*

> เมื่อรีวิวมีคำปฏิเสธซ้อนหรือประโยคขัดแย้ง เช่น *"not bad"* หรือ *"not cold"* เนื่องจาก small model ดูแค่คำเดี่ยวๆ (bag of words) จึงอาจเห็นคำว่า *"bad"* แล้วมั่นใจสูงมาก (confidence ≥ threshold) ว่าเป็นแง่ลบ ทั้งที่จริงเป็นแง่บวก ส่งผลให้ระบบไม่ส่งต่อให้ large model และติด label ผิด ทำให้เจ้าของร้านเข้าใจผิดหรือได้รับแจ้งเตือนที่คลาดเคลื่อน

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run | p50 ms | p95 ms | accuracy | large calls | cache hits |
|---|---|---|---|---|---|
| t3-cache (threshold 0.7) | 7 | 571 | 93.0% | 38% | 35% |

**Q3.1** Which reviews benefit from the cache, and which never will?

*รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์*

> รีวิวสั้นๆ ยอดนิยมที่ลูกค้ามักพิมพ์ซ้ำกันบ่อยๆ (เช่น *"Delicious!"*, *"Good food"*, *"Too salty"*) จะได้ประโยชน์จาก cache เต็มที่ (ช่วยตอบทันทีจน p50 เหลือ 7 ms และลด large calls ลงเหลือ 38%) ส่วนรีวิวที่เขียนบรรยายยาวและมีข้อความเฉพาะตัวไม่ซ้ำกับใครเลยจะไม่มีวันเกิด cache hit

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

*สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร*

> ต้องล้างข้อมูลเก่าใน cache (cache invalidation / flush) เพื่อไม่ให้ระบบคืนค่าการทำนายเก่าที่มาจากโมเดลรุ่นเดิม และควรออกแบบโดยใส่เลขเวอร์ชันของโมเดล (`model_version`) รวมไว้ใน cache key (เช่น `v2:delicious!`) หรือผูกการล้าง cache อัตโนมัติเข้ากับขั้นตอนการ deploy โมเดลใหม่

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

*ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture*

> ได้เรียนรู้ว่าค่าเฉลี่ยหรือค่ากลางอย่าง `p50` เพียงอย่างเดียวสามารถหลอกตาได้มากเมื่อระบบเกิดอาการค้างบางส่วน (เช่นใน `t1-before` ที่ `p50` ดูเร็วขึ้นเหลือ 240 ms ทั้งที่มี `errors` ถึง 20 รายการ) และการนำแพตเทิร์นหลายตัวมาประกอบกัน (Cascade + Cache + Fallback) ช่วยเสริมจุดอ่อนของกันและกันจนทำให้ทั้งความเร็ว (`p50 = 7 ms`), ความแม่นยำ (`93.0%`) และการใช้ทรัพยากร (`large_calls = 38%`) ผ่านเกณฑ์ทั้งหมดได้จริง
