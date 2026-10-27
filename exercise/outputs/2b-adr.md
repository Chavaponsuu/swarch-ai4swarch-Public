Agent: Codex; Model: GPT-6

# ADR-0002: แยกงานหลังสั่งซื้อผ่าน RabbitMQ พร้อมเก็บ event อย่างทนทาน

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** TBD (รอระบุชื่อสมาชิกกลุ่มและผู้มีอำนาจตัดสินใจ)
- **Review Result:** Pending Group Review
- **Recommendation:** เห็นด้วยกับทางเลือก 3 เพื่อเสนอให้กลุ่มพิจารณา ตามข้อความที่ผู้ใช้ส่งมา
- **ขอบเขต:** ร่างที่ AI ช่วยเรียบเรียงร่วมกับข้อเสนอที่ผู้ใช้ส่งมา คงหัวข้อ ADR เดิม และแยกคำแนะนำออกจากมติจริง ยังไม่ใช่ ADR ฉบับสุดท้ายที่นักศึกษาเขียน ผลกลุ่มยัง Pending และยังไม่มีผลทดสอบจริง รีวิวใน `exercise/outputs/3-review.md` เป็นรีวิวเบื้องต้นของ AI

## Context and problem

R-12 ให้ Online Boutique ส่งอีเมลผ่าน MailFast แทน mock `emailservice` ให้แต้มผ่าน `loyaltyservice` ใหม่ และบันทึก “order placed” ให้ฝ่ายการตลาด โดยคาดว่าจะเพิ่ม SMS, fraud scoring และการปรับ recommendations ในปีหน้า (`docs/scenario.md:7–14`)

ข้อเท็จจริงของระบบปัจจุบันที่ใช้ในการแก้ไขครั้งนี้คือ `checkoutservice.PlaceOrder` เรียก payment, shipping และ email ตามลำดับผ่าน gRPC; อีเมลล้มเหลวแล้วคำสั่งซื้อยังสำเร็จ (`docs/scenario.md:18`) ไม่ได้เปิด source code หรือตามแหล่งอ้างอิงในร่างเดิมเพื่อตรวจซ้ำ การมี order store, outbox, ledger และความสามารถค้นสถานะ/กันซ้ำของ payment, shipping หรือ MailFast: **Not evidenced in the provided files**

| ประเด็น | ข้อเท็จจริงจากสถานการณ์ |
|---|---|
| เวลา checkout | p95 ของ `PlaceOrder` ≤ 500 ms; ปัจจุบันประมาณ 300 ms (`docs/scenario.md:24`) |
| ปริมาณงาน | พีค 11.11 มี 200 คำสั่งซื้อ/วินาที แต่ไม่ระบุระยะพีคหรืออัตราหลังพีค (`docs/scenario.md:25`) |
| MailFast | p95 1.5 s, p99 8 s, ผิดพลาดประมาณ 0.5% และจำกัด 50 อีเมล/วินาที (`docs/scenario.md:26`) |
| แต้ม | ภายใน 1 นาทีและครั้งเดียวต่อคำสั่งซื้อ (`docs/scenario.md:27`) |
| ความพร้อมใช้งาน | คำสั่งซื้อต้องไม่ล้มเหลวเพราะอีเมล แต้ม หรือ analytics ล่ม (`docs/scenario.md:28`) |
| ทีมและการดูแล | นักพัฒนา 4 คน ถนัด Go/gRPC; 2 คนเคยใช้ RabbitMQ; ใช้ Kubernetes และต้องการ open-source แบบ self-hosted (`docs/scenario.md:29–30`) |

**[Inferred]** การรอ MailFast ใน checkout เสี่ยงต่อ latency แม้เรียกงานขนานกัน; การประเมินนี้อิงตัวเลขข้างต้น ไม่ใช่การบวกค่า p95 หรือผลทดสอบจริง (`docs/scenario.md:24–26`) ทั้งระยะเวลาที่ธุรกิจเรียกว่า “วางคำสั่งซื้อ” และข้อยกเว้น SLA เมื่อ outage เกินหนึ่งนาทียังไม่ได้กำหนด (`docs/scenario.md:27–28`)

## Decision drivers

1. **Availability:** รับคำสั่งซื้อโดยไม่ต้องรอความพร้อมของงานหลังสั่งซื้อ (`docs/scenario.md:28`)
2. **Correctness และ timeliness:** แต้มครั้งเดียวภายใน 60 วินาที พร้อมตรวจจับการละเมิดจริง (`docs/scenario.md:27`)
3. **Performance และ scalability:** p95 ≤ 500 ms ที่ 200 คำสั่งซื้อ/วินาที และเคารพโควตา MailFast (`docs/scenario.md:24–26`)
4. **Modifiability:** เพิ่ม action ปีหน้าโดยลดการเปลี่ยนขั้นตอน checkout (`docs/scenario.md:14`)
5. **Operability และ cost:** ประเมินทั้ง broker และฐานข้อมูลให้เหมาะกับทีม 4 คน (`docs/scenario.md:29–30`)

## Considered options

ทั้งสามทางเลือกเป็นข้อเสนอใหม่ ข้อดีข้อเสียเป็น **[Inferred]** ไม่ใช่ผลวัด ทางเลือก 2 และ 3 ใช้หลักฐานการรับคำสั่งซื้ออย่างทนทาน กุญแจป้องกัน request ซ้ำ และ ledger แบบเดียวกัน เพื่อไม่ให้ข้อดีของ outbox/การกันซ้ำถูกนับเป็นข้อดีเฉพาะ RabbitMQ

1. เรียก MailFast, loyalty และ analytics โดยตรงใน request พร้อม timeout/การจัดการข้อผิดพลาด
2. ใช้ PostgreSQL เป็น durable job store สร้างงานแยกต่อ action ใน transaction เดียวกับ order ให้ worker แยกชุดตาม action ใช้การจองงานที่หมดอายุได้และ retry แยกกัน โดยไม่เพิ่ม broker
3. ใช้ PostgreSQL เก็บ order/transactional outbox แล้ว relay ไป RabbitMQ แบบ publish/subscribe พร้อมคิวและ consumer แยกตาม action

### Pros and cons of the options

| ทางเลือก | ข้อดี | ข้อเสีย |
|---|---|---|
| 1 — เรียกโดยตรง | **[Inferred]** ใกล้รูปแบบปัจจุบันและทักษะทีม (`docs/scenario.md:18,29`) | **[Inferred]** รอ MailFast เสี่ยงต่อ 500 ms; timeout แล้วข้ามไม่รับประกันงานครบ; เพิ่ม action กระทบ checkout (`docs/scenario.md:14,24–28`) |
| 2 — job store และ worker แยก | **[Inferred]** แยกอีเมลออกจากแต้มได้เช่นเดียวกับทางเลือก 3; ไม่มี broker เพิ่ม; ติดตามงานใน store เดียว (`docs/scenario.md:26–30`) | **[Inferred]** ต้องพัฒนา/ดูแลการจองงาน การกู้ lease การกระจายงานและ retry; polling/การแย่งงานเพิ่มภาระ store; เพิ่ม action ต้องเพิ่มการสร้างงานหรือ dispatcher (`docs/scenario.md:14,25,29`) |
| 3 — outbox และ RabbitMQ | **[Inferred]** กระจาย event ไปหลาย action และแยก consumer/การขยายได้; ทีมสองคนมีประสบการณ์ RabbitMQ (`docs/scenario.md:14,29`) | **[Inferred]** เพิ่ม broker นอกเหนือจาก stores; ต้องดูแล routing, confirms, replay และ schema; ไม่ได้แก้ exactly once หรือโควตา MailFast โดยลำพัง (`docs/scenario.md:26–30`) |

## Decision

เสนอเลือก **ทางเลือก 3: PostgreSQL transactional outbox ร่วมกับ RabbitMQ และ consumer แยกตาม action** เหตุผลหลักคือการเพิ่ม SMS, fraud scoring และ recommendations ปีหน้า ทำให้การกระจาย event เดียวไปยังผู้รับหลายชุดเป็นทิศทางที่ควรพิจารณา และทีมสองคนมีประสบการณ์ RabbitMQ (**[Inferred]** จาก `docs/scenario.md:14,29`) ยอมรับภาระ broker เพิ่มเพื่อเป้าหมาย modifiability โดยไม่อ้างว่า RabbitMQ ทำให้แต้ม exactly once หรือแก้โควตา MailFast

ทางเลือก 2 ยังเป็นทางเลือกสำรองที่สมเหตุผล เพราะ worker แยกชุดก็แยกอีเมลออกจากแต้มได้ ข้อเสนอเลือกทางเลือก 3 จึงมีเงื่อนไขว่าแผนดูแล/กู้คืนต้องทำได้จริงด้วยทีม 4 คน และผล validation ต้องผ่าน หากพิสูจน์ไม่ได้ให้กลับมาเทียบทางเลือก 2 โดยยังคง intent, กุญแจ request และ ledger เดิม (`docs/scenario.md:24–30`)

รายละเอียดต่อไปนี้เป็น **ข้อเสนอการออกแบบใหม่ทั้งหมด** เพื่อรองรับ `docs/scenario.md:24–30` ไม่ใช่ความสามารถที่ยืนยันว่าติดตั้งอยู่แล้ว

**1. Store และการป้องกัน request ซ้ำ**

- เลือก PostgreSQL สำหรับ durable order intent, order record, ผลการเรียกภายนอก และ outbox; ให้ข้อมูล order กับ `OrderPlaced` commit ใน transaction เดียว มี unique key สำหรับ event ตาม `order_id` และชนิด event
- ให้ผู้เรียกเก็บ `checkout_attempt_id` คงเดิมตลอดการ retry ของการสั่งซื้อครั้งเดียว ใช้ร่วมกับตัวตนลูกค้าที่ระบบตรวจสอบแล้วเป็นกุญแจเฉพาะ เก็บ `order_id` และ fingerprint ของข้อมูลธุรกิจโดยไม่เก็บข้อมูลบัตร การใช้ key เดิมกับข้อมูลต่างกันต้องปฏิเสธ; key เดิมกับข้อมูลเดิมคืนผลเดิมหรือสถานะกำลังดำเนินการ
- สร้าง order intent และจองสิทธิ์ประมวลผลอย่างทนทาน **ก่อน** เรียก payment/shipping; ผู้เรียกซ้ำต้องไม่เริ่ม side effect อีกเพียงเพราะ request ก่อนหมดเวลา การส่ง key ใหม่สำหรับการซื้อเดิมยังต้องอาศัยกติกาฝั่งผู้เรียก จึงไม่อ้างว่ากันรายการซ้ำได้จาก key ใดก็ได้
- เสนอให้ loyalty ledger อยู่ PostgreSQL แยก deployment/ทรัพยากรจาก order store เพื่อไม่ให้การล่มของ ledger เป็น dependency โดยตรงของ checkout ทั้งสองส่วนมีแผน backup/restore, replication, monitoring และผู้ดูแลที่ต้องกำหนดก่อนใช้งาน; ไม่มี distributed transaction ระหว่างกัน

**2. ลำดับรับคำสั่งซื้อและ recovery**

| จุดที่บันทึก/เรียก | การทำงานที่เสนอ | เมื่อ process ล่มหรือผลไม่ชัดเจน |
|---|---|---|
| ก่อนผลภายนอก | commit order intent และ operation ID ที่คงเดิม; บันทึกเจตนา payment ก่อนเรียก และเจตนา shipping ก่อนเรียกขั้นนั้น | หากยังไม่บันทึกเจตนา ห้ามเริ่มเรียก; หากบันทึกแล้วแต่ไม่ทราบว่าเรียกสำเร็จหรือไม่ ให้ตรวจสอบสถานะก่อนทำซ้ำ |
| หลังแต่ละบริการตอบ | บันทึกผล/หลักฐานอ้างอิงก่อนเดินขั้นต่อไป | ถ้าบันทึกผลไม่ได้ ให้ถือว่าสถานะไม่แน่นอน; ห้ามสรุปว่าไม่เกิดผลและห้ามเรียกซ้ำอัตโนมัติ |
| payment และ shipping สำเร็จและตรวจสอบได้ | transaction เดียวเปลี่ยนเป็น `PLACED`, บันทึกผลคำสั่งซื้อและ outbox แล้วจึงตอบสำเร็จ | ถ้า commit สำเร็จแต่คำตอบหาย request เดิมต้องคืน order เดิม; ถ้า commit ยังไม่สำเร็จ reconciliation ใช้หลักฐานเดิมเพื่อตัดสินและ commit ต่อ |
| payment สำเร็จ แต่ shipping ล้มเหลว/ไม่ทราบผล | คงสถานะตรวจสอบและเชื่อม operation IDs ไว้กับ order intent | ตรวจผลจริงก่อนเลือกดำเนินต่อหรือยกเลิก/คืนเงินตามนโยบายที่ผู้มีอำนาจรับรอง; ยังไม่สร้าง `OrderPlaced` |

ความสามารถใช้ idempotency key, query สถานะ, ยกเลิก และคืนเงินของ payment/shipping: **Not evidenced in the provided files** ต้องยืนยันสัญญาบริการก่อนเขียนกระบวนการอัตโนมัติ หากค้นสถานะไม่ได้ ให้ส่งเคสพร้อม operation IDs/ประวัติให้ผู้ดูแลตรวจสอบและบันทึกคำตัดสิน ห้ามสมมติว่าการเรียกซ้ำหรือ compensation ปลอดภัย outbox ปิดช่องว่างเฉพาะ order commit กับการส่ง event ไม่ทำให้ผลภายนอกเป็น transaction เดียวกัน

**[Assumption] บทบาทที่เสนอสำหรับ reconciliation:** ให้ผู้ดูแล `checkoutservice` เป็นเจ้าของเคสตั้งแต่พบผลไม่ทราบจนปิดเคส ผู้ดูแล payment/shipping ช่วยยืนยันผลจากหลักฐานของบริการ ผู้ดูแล broker/stores ดูแลงานค้างและการ restore และให้ผู้แทนธุรกิจตัดสินการยกเลิก/คืนเงิน บทบาทเหล่านี้ยังไม่ได้มอบหมายให้บุคคลจริง ต้องระบุชื่อผู้รับผิดชอบและผู้แทนก่อนยอมรับแผน (`docs/scenario.md:29–30`)

ขั้นตอนที่เสนอคือเก็บ request key, order/operation IDs, เวลาและหลักฐานผลไว้ในเคส; ยืนยันสำเร็จทั้ง payment/shipping แล้วจึง commit `PLACED`/outbox ต่อ; หาก payment สำเร็จแต่ shipping ไม่สำเร็จให้ผู้มีอำนาจตัดสินก่อนดำเนินต่อหรือชดเชย; ถ้ายังยืนยันผลไม่ได้ให้คงสถานะตรวจสอบและไม่เรียกซ้ำอัตโนมัติ บันทึกผู้ตัดสิน เหตุผล และผลหลังดำเนินการทุกครั้ง การทดสอบต้องแสดงว่า request เดิมกลับมาดูเคส/order เดิมได้แม้ยังไม่ปิดเคส

**3. การส่ง event และงานหลังสั่งซื้อ**

- outbox relay ส่ง event ID เดิมเมื่อ retry ไปยัง exchange และคิวที่เตรียมไว้อย่างทนทาน ใช้ publisher confirms และตรวจข้อความที่ route ไม่ได้ก่อนทำเครื่องหมายส่งแล้ว broker ล่มให้ค้างที่ outbox; การส่งเป็นแบบ at-least-once
- แยกคิว/consumer/retry สำหรับอีเมล loyalty และ analytics เพื่อให้ทุก action ได้ event ของตนเอง ACK หลังบันทึกผลอย่างทนทาน แยกงานผิดพลาดถาวรไว้ตรวจสอบ; dead-letter ยังถือเป็นงานไม่เสร็จ
- consumer อีเมลเชื่อม MailFast แทน mock และจำกัดอัตรารวมทุก replica รวม retry ≤ 50/วินาที (`docs/scenario.md:10,26`) ต้องยืนยัน protocol, idempotency และการค้นสถานะก่อนทำ adapter; หากรับงานแล้วคำตอบหาย ต้องใช้นโยบาย retry/ความเสี่ยงอีเมลซ้ำที่ตกลงไว้
- loyalty ใช้ unique key ตาม `order_id` กับชนิดการให้แต้ม บันทึกการกันซ้ำและเพิ่มแต้มใน transaction เดียว ก่อน ACK; duplicate หลัง commit คืนผลเดิม การกันซ้ำของ request ในข้อ 1 และของ ledger ทำหน้าที่คนละช่วง (`docs/scenario.md:27`)
- analytics บันทึก event อย่างทนทานและกันซ้ำตาม `order_id` กับชนิด event ก่อน ACK; ปลายทาง/schema ยังต้องตกลง (`docs/scenario.md:12`)
- event มี version, event ID คงเดิม, `order_id`, เวลา commit และข้อมูลขั้นต่ำที่แต่ละ action ต้องใช้; ต้องตกลงตัวตนสมาชิก/สูตรแต้ม จำกัดสิทธิ์ข้อมูลติดต่อ และไม่คัดลอกข้อมูลบัตรลง event
- retention ของกุญแจกันซ้ำต้องครอบคลุมช่วง retry/replay ที่อนุญาต; ห้ามลบ event ที่ยังส่งไม่ครบเงียบ ๆ แผน replay สำหรับ consumer ใหม่ต้องระบุแยก ไม่อ้างว่าเพิ่มคิวแล้วจะได้ event ในอดีตเอง

**4. ความหมาย SLA และกรณี outage**

- **[Assumption] ข้อเสนอรอธุรกิจรับรอง:** ให้ `t0` เป็น commit แรกที่ order เปลี่ยนเป็น `PLACED` พร้อม outbox และ `t1` เป็น commit แต้มสำเร็จ ต้องมี `t1 − t0 ≤ 60 s`; ไม่เริ่มเวลาใหม่เมื่อ relay/retry พร้อมเก็บเวลารับ request และเวลาผลภายนอกเพื่อตรวจความล่าช้าก่อน commit นิยามนี้ยังไม่ใช่การอนุมัติเปลี่ยนข้อกำหนด (`docs/scenario.md:27`)
- **ขอบเขตเวลา 60 s:** นับ end-to-end ตั้งแต่ `t0` ถึง `t1` รวม outbox relay, broker/คิว, retry/backoff และการประมวลผล/commit ของ consumer ไม่ใช่ให้ consumer มีเวลาเพิ่มอีก 60 s หลังรับข้อความ (`docs/scenario.md:27`)
- **[Assumption] งบเวลาเริ่มต้นสำหรับทดสอบ:** relay/ส่งต่อ ≤ 5 s, รอคิวรวม retry ≤ 45 s, ประมวลผลและ commit ledger ≤ 10 s รวม ≤ 60 s ตัวเลขย่อยเป็นข้อเสนอ ไม่ใช่ผลวัดหรือการบวกค่า p95; ต้องวัด end-to-end ทุกคำสั่งซื้อ รวมงานที่ยังไม่เสร็จเมื่อครบกำหนด
- **[Inferred]** เมื่อเส้นทางแต้มล่มเกินหนึ่งนาที จะรักษาทั้งการรับ order ต่อและกำหนดแต้มเดิมสำหรับทุก order ไม่ได้ (`docs/scenario.md:27–28`) ข้อเสนอคือรับ order ต่อ เก็บงาน แจ้ง SLA breach และตามให้แต้มหลังฟื้นตัว; ทุกกรณีเกินเวลา **ยังถือว่าไม่ผ่านข้อกำหนดเดิม** จนกว่าธุรกิจจะรับรองข้อยกเว้น/การชดเชย
- order/outbox store เป็น dependency ใหม่ที่ต้องมีความพร้อมใช้งานของตนเอง รวมเวลา commit ในงบ 500 ms; broker/consumer ไม่ใช่เงื่อนไขของคำตอบสำเร็จ ต้องทดสอบการล่มและพื้นที่เต็มด้วย ไม่อ้างว่าพื้นที่จำกัดรับ outage ไม่จำกัดเวลาได้

**5. ความจุและการระบาย backlog**

**[Inferred]** หากหนึ่งคำสั่งซื้อต้องส่งหนึ่งอีเมล ให้ `T_peak` เป็นวินาทีของพีค, `Q0` เป็นงานค้างเริ่มต้น และ `mu_email` เป็นอัตราอีเมลสำเร็จจริงซึ่งไม่เกิน 50/วินาทีและอาจต่ำลงจาก retry:

- `Q_end ≈ Q0 + (200 − mu_email) × T_peak`; ก่อนนับ retry backlog เพิ่มอย่างน้อย 150 รายการ/วินาที (`docs/scenario.md:25–26`)
- ถ้าอัตราใหม่หลังพีคคือ `lambda_after < mu_email` เวลาระบายกลับระดับเดิมประมาณ `(Q_end − Q0) / (mu_email − lambda_after)`; หากอัตราใหม่ไม่ต่ำกว่าอัตราสำเร็จ คิวจะไม่ระบายหมด
- **[Assumption] ตัวอย่างคำนวณเท่านั้น:** พีค 10 นาที, `Q0 = 0`, ไม่มี retry และส่งสำเร็จเต็ม 50/วินาที จะค้าง 90,000 รายการ; ถ้าหลังพีคเหลือ 20 คำสั่งซื้อ/วินาที จะใช้ 3,000 วินาทีหรือ 50 นาทีเพื่อระบายคิวหมด ตัวเลข 10 นาทีและ 20/วินาทีไม่ใช่ข้อเท็จจริงจากสถานการณ์
- broker ล่มต่างจาก MailFast ช้า: ที่พีค outbox อาจสะสม 200 event/วินาที หากไม่มีการส่งออก ตัวอย่างล่ม 10 นาทีเท่ากับ 120,000 event ก่อนนับข้อมูลค้างเดิม
- กำหนดพื้นที่จากจำนวนค้าง × ขนาดข้อมูลที่วัดจริง แล้วรวม indexes, retry, retention และ replication; กำหนด headroom จากช่วง outage/การกู้คืนที่ตกลง ห้ามใช้ตัวอย่างข้างต้นเป็นขนาด production
- แยกแจ้งเตือนอายุงานแต้มเทียบ 60 s, อายุอีเมลเทียบเวลาที่ธุรกิจยอมรับ และพื้นที่ที่เหลือเทียบเวลาที่ผู้ดูแลต้องใช้กู้คืน หากคิวเต็มจนทำให้ checkout ล้มเหลวจาก downstream outage ต้องรายงานว่าไม่ผ่านเป้าหมาย availability (`docs/scenario.md:27–28`)

**6. Group Review Summary**

ข้อความส่วนนี้เรียบเรียงจากข้อเสนอที่ผู้ใช้ส่งมา โดยแยกคำแนะนำว่า “เห็นด้วยกับทางเลือก 3” ออกจากผลพิจารณาของกลุ่มที่ยังเป็น `Pending Group Review` ข้อเท็จจริงประกอบเหตุผลอ้างอิง `docs/scenario.md`; ข้อมูลที่ยังไม่มีคงเป็น **Assumption** หรือคำถามเปิด

| เรื่อง | ข้อเสนอและเหตุผล | ผลพิจารณาจริง |
|---|---|---|
| ทางเลือกสถาปัตยกรรม | เห็นด้วยกับทางเลือก 3: PostgreSQL transactional outbox + RabbitMQ เพื่อเสนอให้กลุ่มพิจารณา เพราะแยก consumer และรองรับ action เพิ่มได้; ยอมรับภาระ broker/stores และเก็บทางเลือก 2 เป็นตัวเลือกสำรอง (**[Inferred]** จาก `docs/scenario.md:14,29–30`) | Pending |
| SLA แต้ม | เสนอเริ่มนับจาก commit แรกของ `PLACED`/outbox จนถึง commit แต้ม รวม relay, เวลาค้างคิว, retry และ consumer ไม่เกิน 60 s; ใช้ ledger unique key และ transaction เพื่อป้องกันผลซ้ำ (`docs/scenario.md:27`) | Pending |
| loyalty ล่มเกิน 60 s | เสนอรับ order ต่อเมื่อ order store ยังพร้อมและมีพื้นที่ เก็บ event และตามให้แต้มเมื่อฟื้นตัว โดยรายงานกรณีล่าช้าเป็น SLA breach เดิม ไม่ถือว่าได้รับข้อยกเว้นหรืออนุมัติการชดเชยแล้ว (`docs/scenario.md:27–28`) | Pending |
| payment/shipping recovery | บันทึก intent/operation ID ก่อนเรียก; ตรวจสถานะก่อน retry เมื่อไม่ทราบผล และใช้ idempotency key เฉพาะบริการที่ยืนยันว่ารองรับ; ให้ reconciliation ตัดสินเคสและสร้าง `OrderPlaced` หลังยืนยันผลสำเร็จและ commit order/outbox ตามข้อ 2 | Pending |
| ความจุและการดูแล | แยกคิว จำกัดอัตราอีเมลรวม retry ≤ 50/วินาที ติดตามความยาว/อายุคิวและ dead-letter ที่ยังไม่เสร็จ; ใช้ระยะพีค 10 นาที/หลังพีค 20 ครั้งต่อวินาทีเป็นเพียงตัวอย่างคำนวณและชุดทดลอง ไม่ใช่ขนาด production (`docs/scenario.md:25–26,29–30`) | Pending |

**Decision Record**

- **Final Decision:** Pending
- **Reason:** รอการพิจารณาจากกลุ่ม
- **Decided By:** TBD
- **Decision Date:** TBD
- **ADR Status:** Proposed

**ข้อสรุปที่เสนอ:** **[Inferred]** ทางเลือก 3 มีเหตุผลเพียงพอให้นำเข้าพิจารณา โดยแลกความสามารถเพิ่ม action กับภาระดูแล broker และ stores แต่ยังต้องยืนยันนิยาม SLA, สัญญาบริการภายนอก, สมมติฐานความจุ และผู้รับผิดชอบตาม `Open questions` (`docs/scenario.md:14,24–30`)

เมื่อมีผลพิจารณาจริง ให้เติมข้อสรุป เหตุผล ผู้ตัดสินและวันที่ ทั้งในตารางและ Decision Record แล้วแก้เนื้อหา ADR ให้สอดคล้องกัน ระหว่างนี้คง `Proposed`; แผนใน Validation ยังไม่ได้รัน และคำแนะนำเห็นด้วยไม่ใช่การอนุมัติจากกลุ่ม

## Consequences

- **Positive:** **[Inferred]** checkout ไม่รอ MailFast/loyalty/analytics และแต่ละ action ขยายแยกกันได้ (**performance, availability, scalability**; `docs/scenario.md:24–28`) แต่ผล 500 ms ยังต้องวัดรวมการเขียน intent และ commit ใหม่ทั้งหมด
- **Positive:** **[Inferred]** consumer ใหม่รองรับ action เพิ่มได้โดยไม่เพิ่มการเรียก downstream ใน checkout เมื่อ event มีข้อมูลเพียงพอ (**modifiability**; `docs/scenario.md:14`); หากต้องแก้ schema หรือ fraud เป็นเงื่อนไขก่อนรับ order ต้องทบทวนแบบ
- **Negative:** **[Inferred]** PostgreSQL สำหรับ order/outbox และ ledger ที่แยกทรัพยากร รวม RabbitMQ เพิ่มภาระ backup/restore, monitoring และ on-call ให้ทีม 4 คน; ต้องเทียบทางเลือก 2 ก่อนอนุมัติ (**operability, cost, availability**; `docs/scenario.md:29–30`)
- **Negative:** **[Inferred]** อีเมลสะสมระหว่างพีค และแต้มอาจเกินกำหนดระหว่าง outage แม้ไม่สูญหาย; เพิ่ม worker ไม่เพิ่มโควตา MailFast (**timeliness, user experience**; `docs/scenario.md:25–28`)
- **Risks and mitigations:** duplicate/replay อาจให้แต้มซ้ำ จึงใช้ทั้งกุญแจ request และ ledger transaction พร้อม retention ครอบคลุม replay; กรณีผล payment/shipping ไม่ทราบต้อง reconciliation ตามข้อ 2 (**correctness, durability**; เป้าหมาย `docs/scenario.md:27`; วิธีลดความเสี่ยงเป็นข้อเสนอ)
- **Risks and mitigations:** outbox/คิวสูญหาย เต็ม หรือ route ไม่ถึงอาจทำงานไม่ครบ จึงทดสอบ routing, confirms, backup/restore และติดตาม event จนแต่ละ action เสร็จ (**availability, durability, operability**; เป้าหมาย `docs/scenario.md:10–12,28`)
- **Risks and mitigations:** MailFast รับแล้วไม่ตอบอาจทำให้อีเมลซ้ำ; ต้องยืนยัน API และนโยบายกรณีไม่ทราบผล ความสามารถกันซ้ำยัง **Not evidenced in the provided files** (**reliability, user experience**; `docs/scenario.md:26`)

## Validation

ทั้งหมดเป็น **แผนทดสอบ ยังไม่ได้รันหรือยืนยันผล** ต้องบันทึกสภาพแวดล้อม ระยะทดสอบ ผลวัด และกรณีที่ไม่ผ่าน; ใช้ข้อมูลจำลองสำหรับการทดลอง ไม่เรียกชำระเงิน/จัดส่งจริงโดยพลการ

**[Assumption] ชุดทดลองเริ่มต้นที่เสนอ:** ใช้หนึ่งอีเมลต่อ order, เริ่มจากคิวว่าง, พีค 200 คำสั่งซื้อ/วินาทีเป็นเวลา 10 นาที แล้วลดเหลือ 20/วินาทีและติดตามจนคิวกลับระดับเดิม; ตัวอย่างไม่มี retry ควรค้าง 90,000 อีเมลและระบาย 50 นาที แต่เมื่อจำลองความผิดพลาดต้องใช้ผลวัดจริง ทดลอง broker ล่ม 10 นาทีแยกอีกกรณีเพื่อเทียบ outbox กับตัวอย่าง 120,000 event และทดลอง loyalty ล่ม 120 s เพื่อให้เห็น SLA breach อย่างชัดเจน ตัวเลขระยะเวลา/โหลดหลังพีคเป็นค่าทดลองที่เสนอ ไม่ใช่ข้อมูลธุรกิจหรือผลการรัน (`docs/scenario.md:24–28`)

| การตรวจ | เกณฑ์ตรวจรับและหลักฐานที่ต้องเก็บ |
|---|---|
| โหลด checkout | 200 คำสั่งซื้อ/วินาที ตลอดระยะพีคที่ธุรกิจรับรอง; p95 `PlaceOrder` ≤ 500 ms รวมทุกการเขียน store/การประมวลผลใน request รายงาน errors/timeouts แยกด้วย จำลอง MailFast p95 1.5 s, p99 8 s และผิดพลาด 0.5%; ส่งรวม retry ≤ 50/วินาที (`docs/scenario.md:24–26`) |
| แต้มตามเวลา | วัด `t1 − t0` ของทุก order และตรวจงานค้างเมื่อครบ 60 s; ทุก order ในเงื่อนไขทดสอบที่อ้างว่าผ่านต้องได้แต้มครั้งเดียวภายใน 60 s รายงานจำนวน/อายุที่เกิน ห้ามสรุปจากค่าเฉลี่ยหรือ p95 เท่านั้น (`docs/scenario.md:27`) |
| กุญแจ request และ event ซ้ำ | ส่ง request key เดิมซ้ำพร้อมกันและหลังคำตอบหาย ต้องได้ order เดิม/สถานะเดิม; key เดิมแต่ข้อมูลต่างต้องปฏิเสธ ทดสอบ event ซ้ำและ crash หลัง ledger commit ก่อน ACK: แต้มเพิ่มครั้งเดียว; analytics หนึ่งรายการต่อกุญแจ |
| ช่องว่าง payment/shipping/commit | แทรก crash ก่อน/หลังแต่ละ external call และก่อน/หลังบันทึกผล/commit; มี intent ตรวจสอบได้ ไม่มีการเรียกซ้ำแบบเดาผล เคสไม่ทราบผลเข้าสู่ reconciliation ที่มีหลักฐาน/ผู้ตัดสิน; order ที่ยืนยันสำเร็จมี outbox เพียงหนึ่ง event เชิงธุรกิจ |
| broker และ consumer ล่ม | ปิดแยกแต่ละส่วนเมื่อ order store ยังทำงาน; checkout ไม่ล้มเหลวเพราะเหตุนั้น หลังฟื้นตัวติดตามทุก event จนงานครบ ทดสอบ relay ล่มหลัง confirm ก่อนทำเครื่องหมายด้วย (`docs/scenario.md:28`) |
| outage แต้มเกิน 60 s | order ยังรับได้ งานคงอยู่และได้แต้มหลังฟื้นตัว; รายงานเป็น SLA breach ตามข้อกำหนดเดิม แยกจากผลผ่านกรณีปกติ; การแจ้งเตือนทำงานไม่ทำให้กรณีนี้ผ่าน `docs/scenario.md:27` |
| backlog และการกู้คืน | วัดขนาด event, อัตราสำเร็จหลัง retry และอัตราระบายจริงที่ระยะพีค/โหลดหลังพีคที่ยืนยัน; เทียบสูตรกับผล ทดสอบพื้นที่เต็มและ restore พร้อมนับงานที่หาย/ซ้ำ; ตัวอย่างพีค 10 นาทีเป็นเพียงชุดทดลอง ไม่ใช่ตัวแทนพีคจริงโดยอัตโนมัติ |
| ภาระดูแลและ action ใหม่ | ระบุผู้ดูแล/ขั้นตอนกู้ broker และ stores ให้ทีม 4 คนทดลอง; เพิ่ม consumer ใหม่โดยไม่เพิ่มการเรียกใน checkout พร้อมบันทึกงาน routing/schema ที่ต้องแก้ แล้วเทียบผลกับทางเลือก 2 (`docs/scenario.md:14,29–30`) |

## Open questions

- ใครเป็นผู้อนุมัติ ADR และเจ้าของ broker, order store, loyalty ledger และ reconciliation? ต้องมีผู้รับผิดชอบจริงก่อนยอมรับแผนดูแล (`docs/scenario.md:29–30`)
- ธุรกิจยอมรับ `t0` ที่เสนอหรือไม่? หากกำหนดให้เริ่มก่อน commit ต้องปรับการวัด/งบเวลา และจะตัดสินข้อขัดแย้ง SLA แต้มกับ availability เมื่อ outage ยาวอย่างไร? ระหว่างนี้ข้อกำหนดเดิมยังใช้ (`docs/scenario.md:27–28`)
- ระยะพีค อัตราหลังพีค ระยะ outage ที่ต้องรองรับ และอายุอีเมลสูงสุดที่ยอมรับคือเท่าใด? ยังไม่สามารถล็อกขนาด production จาก `docs/scenario.md:25–26` ได้
- payment/shipping รองรับ operation ID คงเดิม, idempotency และค้นสถานะอย่างไร? ใครตัดสินการคืนเงิน/ยกเลิกในกรณีค้าง? **Not evidenced in the provided files**
- วิธีผูกตัวตนลูกค้ากับ `checkout_attempt_id`, ช่วง retry/replay ที่ต้องเก็บกุญแจ และการแสดงสถานะค้างให้ผู้เรียกเป็นอย่างไร? ข้อเสนอไม่ถือว่าความสามารถนี้มีอยู่แล้ว
- สูตรแต้ม บัญชีสมาชิก การยกเลิก/คืนแต้ม และการจัดการ order ที่ยังไม่ `PLACED` ต้องเป็นอย่างไร? `docs/scenario.md:11,27` ไม่ได้ระบุ
- MailFast มี API/protocol, idempotency key, query สถานะและข้อผิดพลาดถาวรแบบใด? analytics เก็บที่ใด/schema ใด/ภายในเวลาเท่าใด? **Not evidenced in the provided files** นอกเหนือจากข้อกำหนดใน `docs/scenario.md:10–12,26`
- PostgreSQL/RabbitMQ จะใช้ topology, replication, backup/restore, retention และงบทรัพยากรเท่าใด? ต้องกำหนดจากขอบเขต outage และผลทดลองก่อนอนุมัติ ไม่อ้างว่าการระบุชื่อ store เพียงอย่างเดียวปิดเรื่อง availability แล้ว
- fraud scoring ปีหน้าจะเป็นงานหลังสั่งซื้อหรือเงื่อนไขก่อนยอมรับคำสั่งซื้อ? หากเป็นเงื่อนไขก่อน ต้องทบทวนขอบเขตการตัดสินใจนี้ (`docs/scenario.md:14`)
