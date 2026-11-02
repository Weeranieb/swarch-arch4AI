# Tutorial 6 worksheet: serving patterns

**Name / ชื่อ:** วีรชัย แป้นถึง

**Student ID / รหัสนิสิต:** 6970279321

This is **individual work**: each student completes and submits their own copy, in their own fork.

_งานนี้เป็น**งานเดี่ยว** นิสิตแต่ละคนทำและส่งงานของตัวเอง โดยใช้ forked repository ของตัวเอง_

Copy result rows from `results/runs.csv` (or the load test's output) into the tables. Keep answers short: one to three sentences each. Answer in Thai or English.

ตอนที่ตอบ ให้นิสิตคัดลอกผลจาก `results/runs.csv` (หรือจาก output ของ load test) ลงในตาราง ตอบสั้นๆ ข้อละ 1–3 ประโยค จะตอบเป็นภาษาไทยหรือภาษาอังกฤษก็ได้

## Task 0: Baseline (copy the baseline results from `results/runs.csv` here / copy ผลลัพธ์ของ baseline จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls |
| -------- | ------ | ------ | -------- | ----------- |
| t0-large | 490    | 599    | 93.5%    | 100%        |
| t0-small | 15     | 29     | 72.5%    | 0%          |

**Q0.** What does the large model buy, and what does it cost?

_large model ให้อะไรเพิ่ม และต้องแลกด้วยอะไร_

> large-model ให้ความแม่นยำจาก model เล็ก 72.5 -> 93.5 % ซึ่งถือว่ามากขึ้นอย่างเห็นได้ชัดแต่ก็แลกกับ เวลาที่ใช้จาก 29 ms -> 599 ms (ที่ p95 ซึ่งเป็นประมาณ 20 เท่า) ดังนั้นจึงต้อง reconsider ถึง balance ของ latency ของระบบด้วยและ accuracy model

## Task 1: Fallback prediction (copy the Task 1 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 1 จาก `results/runs.csv` มาวางที่นี่)

| Run       | run time (s) | p50 ms | p95 ms | errors | pending | accuracy |
| --------- | ------------ | ------ | ------ | ------ | ------- | -------- |
| t1-before | 26.2         | 223    | 344    | 18     | 0       | 93.4%    |
| t1-after  | 9.5          | 404    | 1006   | 0      | 19      | 92.8%    |

**Q1.1** In `t1-before`, p50 can look _better_ than the baseline. Why is p50 misleading here? Which column tells the real story?

_ใน `t1-before` ค่า p50 อาจดู "ดีกว่า" baseline ทำไม p50 จึงทำให้เข้าใจผิดในกรณีนี้ และ column ไหนบอกความจริง_

> p50 ทำให้เข้าใจผิดเพราะนับเฉพาะ request ที่สำเร็จ ส่วน request ที่ค้างแล้วกลายเป็น error จะไม่ถูกนับใน latency เลย ทำให้เหลือแต่ตัวที่ผ่านเร็ว p50 จึงดูดี ทั้งที่ระบบแย่ลง คอลัมน์ที่บอกความจริงคือ `errors` (ของเรา 18 รายการ) และเวลาทั้งรอบที่ยาวขึ้นมาก

**Q1.2** The hang affects only 10% of large-model calls. Why is it worse for the system than 10% of reviews being slow? Think about what a waiting request holds on to in the review service and the gateway.

_การค้างเกิดกับ calls ไปยัง large model แค่ 10% ทำไมจึงส่งผลต่อระบบแย่กว่าการที่รีวิว 10% ช้าลงเฉยๆ ลองคิดว่า request ที่รออยู่ถือทรัพยากรอะไรไว้ใน review service และ gateway_

> เพราะ request ที่รอ AI ค้างจะยึด connection/worker ของ gateway และ review service ไว้ ทำให้ request อื่นเข้าไม่ได้ ผลกระทบเลยกระจายเกิน 10% ไม่เหมือนกรณีช้าเฉยๆ ที่จบแล้วคืนทรัพยากร หลังแก้ใช้ timeout 1 วินาทีแล้วคืน pending จึงปล่อยช่องเร็ว errors 18→0 และเวลาทั้งรอบ 26s→9.5s

**Q1.3** Pending reviews have no label. Suggest what should happen to them, and name the topology from the lecture that does it.

_รีวิวที่ pending ยังไม่มี label ควรจัดการกับรีวิวเหล่านี้อย่างไร และ topology ใดจาก lecture ที่ทำแบบนั้น_

> ควรให้ batch job ดึงรีวิว pending ไปติด label ทีหลังเป็นชุด เมื่อ inference พร้อมแล้ว topology ที่ทำแบบนี้คือ Asynchronous / batch (T4)

**Q1.4** We chose a 1-second timeout. What goes wrong if it is 0.1 s? And if it is 10 s?

_เราเลือก timeout 1 วินาที จะเกิดปัญหาอะไรถ้าตั้งเป็น 0.1 วินาที และถ้าตั้งเป็น 10 วินาที_

> ถ้าเร็วไป (0.1 s) จะตัดก่อน large model ตอบทัน (~200 ms) ทำให้ pending สูงโดยไม่จำเป็น ต้องให้ batch มาติด label ใหม่ทีหลัง ถ้าช้าไป (10 s) จะยึด connection นานเหมือนเดิม ส่งผลกระทบทั้งระบบเมื่อ inference ค้าง

## Task 2: Cascade (copy the Task 2 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 2 จาก `results/runs.csv` มาวางที่นี่)

| Run    | threshold | p50 ms | p95 ms | accuracy | large calls |
| ------ | --------- | ------ | ------ | -------- | ----------- |
| t2-0.6 | 0.6       | 6      | 569    | 91.5%    | 38%         |
| t2-0.7 | 0.7       | 363    | 583    | 93.0%    | 53%         |
| t2-0.8 | 0.8       | 491    | 664    | 93.5%    | 64%         |
| t2-0.9 | 0.9       | 483    | 600    | 93.5%    | 77%         |

**Q2.1** Which threshold would you deploy for CampusEats, and why? Name the quality you give up.

_จะเลือก threshold ใดสำหรับ deploy ใน CampusEats เพราะอะไร และต้องยอมเสีย quality ใดไป_

> เลือก 0.6 เพราะ accuracy ยังเกิน 90% (91.5%) และ p50 แค่ 6 ms พร้อม large calls ต่ำสุด (38%) ถ้าเพิ่ม threshold ความแม่นขึ้นเล็กน้อยแต่ latency/cost สูงขึ้นชัด ยอมเสีย accuracy เล็กน้อยเมื่อเทียบกับ large ล้วนๆ (93.5%)

**Q2.2** Look at p50 and p95 for your threshold. Why does the cascade help p50 much more than p95?

_ดูค่า p50 และ p95 ของ threshold ที่เลือก ทำไม cascade จึงช่วย p50 ได้มากกว่า p95 มาก_

> เพราะ request ส่วนใหญ่ (ฝั่ง p50) ใช้คำตอบ small model ได้เลย ไม่ต้องรอ large จึงเร็วมาก (p50 = 6 ms) ส่วน p95 ยังรวมเคสที่ยากแล้ว escalate ไป large (~200 ms) เลยยังสูงอยู่ (569 ms)

**Q2.3** The cascade trusts the small model's confidence. Give one situation where that trust is misplaced, and its effect on users.

_cascade เชื่อ confidence ของ small model ยกตัวอย่างหนึ่งสถานการณ์ที่ไม่ควรเชื่อ และผลที่เกิดกับผู้ใช้_

> เช่นประโยคปฏิเสธอย่าง "not bad" ที่ small model มั่นใจแต่ตีความผิด เจ้าของร้านอาจได้ label ผิด (คิดว่าลูกค้าไม่พอใจทั้งที่พอใจ หรือกลับกัน) แล้วตัดสินใจผิด

## Task 3: Cached predictions (copy the Task 3 results from `results/runs.csv` here / copy ผลลัพธ์ของ Task 3 จาก `results/runs.csv` มาวางที่นี่)

| Run      | p50 ms | p95 ms | accuracy | large calls | cache hits |
| -------- | ------ | ------ | -------- | ----------- | ---------- |
| t3-cache | 4      | 550    | 93.0%    | 38%         | 35%        |

**Q3.1** Which reviews benefit from the cache, and which never will?

_รีวิวแบบไหนได้ประโยชน์จาก cache และแบบไหนไม่มีวันได้ประโยชน์_

> ได้ประโยชน์คือรีวิวที่ข้อความซ้ำหรือต่างกันแค่ตัวพิมพ์/ช่องว่าง (เช่น "Delicious!" กับ " delicious! ") เพราะ cache จำคำตอบไว้แล้ว ในรันเรา hit ราว 35% ส่วนรีวิวข้อความใหม่ที่ไม่เคยมีและไม่ซ้ำกับใคร จะไม่มีวันได้ประโยชน์จาก cache

**Q3.2** Next week you deploy a better large model. What must happen to the cache, and how would you design for it?

_สัปดาห์หน้าจะ deploy large model ตัวใหม่ที่ดีกว่าเดิม ต้องทำอะไรกับ cache และจะออกแบบรองรับเรื่องนี้อย่างไร_

> ต้องเคลียร์หรือ invalidate cache ไม่งั้นจะยังตอบด้วยผลจากโมเดลเก่า ออกแบบได้โดยใส่ model version ไว้ใน cache key (เปลี่ยนโมเดลแล้ว key ใหม่ไม่ชนของเก่า) และ/หรือ flush cache ตอน deploy

## Debrief

**Q4.** Name one thing you learned from these measurements that the lecture did not cover.

_ยกตัวอย่างหนึ่งสิ่งที่ได้เรียนรู้จากการวัดผลครั้งนี้ ที่ไม่มีพูดถึงใน lecture_

> ทำให้รู้ว่า AI app ก็ยังต้องการบาง model ที่เน้น low latency และมีการ balance ด้วยวิธีต่างๆ เพื่อให้ตอบสนองต่อ user ได้เร็วพร้อมกับรักษา Accuracy ไว้ นั่นแปลว่า Architecture for AI จะเข้ามามีบทบาท และในฐานะ Software Architect จำเป็นจะต้องรู้สถาปัตยกรรมที่ใช้บ่อยในวงการ AI เพื่อตอบโจทย์ทาง business
