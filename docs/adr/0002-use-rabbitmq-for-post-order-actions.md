Agent: Codex; Model: GPT-6

# ADR-0002: แยกงานหลังสั่งซื้อด้วย transactional outbox และ RabbitMQ

- **Status:** Accepted
- **Date:** 2026-10-27
- **Deciders:** กลุ่มตามผลที่ผู้ใช้ยืนยัน: ณัฐเมธีโพเลิศสวัส (สถาปนิก), ชวภณ สุวรรณะ (AI analyst, Product owner และ SRE); สิทธิ์ลงมติรายบุคคลยังไม่ได้แจ้ง
- **ที่มา:** AI ช่วยเรียบเรียงจากหลักฐานและข้อมูลผู้ใช้ยืนยัน: ประชุม 2026-10-27, ณัฐเมธีเป็นสถาปนิก, ชวภณเป็น AI analyst และยืนยันล่าสุดว่ารับบท Product owner กับ SRE เองด้วย และหลังเห็นผลรีวิวอิสระผู้ใช้ยืนยันคง Option 3 พร้อมเงื่อนไขสามข้อก่อนใช้งานจริง
- **ผลทดสอบ:** ยังไม่ได้รัน การตรวจในเอกสารเป็นการวิเคราะห์ข้อเสนอ

## Context and problem

R-12 ต้องส่งอีเมลผ่าน MailFast แทน mock `emailservice` ให้แต้มผ่าน `loyaltyservice` ใหม่ และบันทึก “order placed” สำหรับการตลาด ปีหน้าคาดว่าจะเพิ่ม SMS, fraud scoring และการปรับ recommendations (`docs/scenario.md:7–14`)

ปัจจุบัน `checkoutservice.PlaceOrder` เรียก payment, shipping และ email ตามลำดับ; เมื่ออีเมลล้มเหลวจะบันทึกคำเตือนแล้วตอบสำเร็จ (`system/online-boutique/src/checkoutservice/main.go:252–279`) ทั้งสามใช้ gRPC ผ่าน `chargeCard`, `shipOrder` และ `sendOrderConfirmation` (`main.go:369–393` ใน path เดียวกัน; `docs/scenario.md:18`)

| ประเด็น | ข้อเท็จจริง |
|---|---|
| Checkout | p95 ≤ 500 ms; ปัจจุบันประมาณ 300 ms (`docs/scenario.md:24`) |
| พีค | 200 คำสั่งซื้อ/วินาที ระหว่าง 11.11 (`docs/scenario.md:25`) |
| MailFast | p95 1.5 s, p99 8 s, ผิดพลาดประมาณ 0.5%, จำกัด 50 อีเมล/วินาที (`docs/scenario.md:26`) |
| แต้ม | ภายใน 1 นาที และไม่ให้ซ้ำต่อคำสั่งซื้อ (`docs/scenario.md:27`) |
| ความพร้อมใช้งาน | คำสั่งซื้อต้องไม่ล้มเหลวเพราะ email, loyalty หรือ analytics ล่ม (`docs/scenario.md:28`) |
| ทีมและการปฏิบัติการ | 4 คน ถนัด Go/gRPC; 2 คนเคยใช้ RabbitMQ; ใช้ Kubernetes และต้องการ open-source แบบ self-hosted (`docs/scenario.md:29–30`) |

**[Inferred]** การรอ MailFast ใน request เสี่ยงเกินเป้าหมาย checkout แม้เรียก action ขนานกัน; นี่ไม่ใช่ผลทดสอบหรือการบวก p95 โดยตรง (`docs/scenario.md:24–26`) หากหนึ่ง order ส่งหนึ่งอีเมล backlog ระหว่างพีคต่อเนื่องเพิ่มอย่างน้อย 200 − 50 = 150 รายการ/วินาทีก่อน retry; การเพิ่ม worker ไม่เพิ่มโควตา MailFast

ไฟล์ที่ได้รับแสดง UUID ใหม่ต่อ request และการสร้าง `orderResult` แต่ไม่แสดง durable order store, outbox หรือ loyalty ledger: **Not evidenced in the provided files** (`system/online-boutique/src/checkoutservice/main.go:230–279`) ส่วนประกอบเหล่านี้ด้านล่างเป็นข้อเสนอใหม่

## Decision drivers

1. **Availability:** แยกความล้มเหลวของงานหลังสั่งซื้อจาก checkout (`docs/scenario.md:28`)
2. **Correctness และ timeliness:** แต้มครั้งเดียวภายใน 1 นาที (`docs/scenario.md:27`)
3. **Performance และ scalability:** p95 ≤ 500 ms ที่ 200 คำสั่งซื้อ/วินาที พร้อมเคารพโควตา MailFast (`docs/scenario.md:24–26`)
4. **Modifiability:** เพิ่มผู้รับงานปีหน้าโดยลดการแก้ checkout (`docs/scenario.md:14`)
5. **Operability และ cost:** ภาระดูแลต้องเหมาะกับทีม 4 คนและแนวทาง self-hosted (`docs/scenario.md:29–30`)

## Considered options

1. เรียกทุก action โดยตรงใน request พร้อม timeout
2. ใช้ PostgreSQL เป็น durable job store สร้างงานใน transaction เดียวกับ order และแยกชุด worker/retry ตาม action โดยไม่เพิ่ม broker
3. ใช้ PostgreSQL เก็บ order/transactional outbox แล้วส่งผ่าน RabbitMQ ไปยังคิว/consumer แยกตาม action

### Pros and cons of the options

ข้อดีข้อเสียเป็น **[Inferred]** จากเป้าหมายใน `docs/scenario.md:14,24–30` ไม่ใช่ผลวัด ทางเลือก 2 และ 3 ต้องมี request idempotency, durable storage และ ledger เช่นเดียวกัน

| ทางเลือก | ข้อดี | ข้อเสีย |
|---|---|---|
| เรียกโดยตรง | ใกล้รูปแบบ gRPC ปัจจุบันและทักษะทีม | MailFast อยู่ใน latency ของ checkout; timeout แล้วข้ามไม่รับประกันทำครบ; เพิ่ม action ต้องแก้ checkout |
| PostgreSQL job store | แยกอีเมลจากแต้มได้และไม่มี broker เพิ่ม; ติดตามงานใน store เดียว | ต้องพัฒนาการจองงาน การกู้ lease และการแจกงาน; polling/การแย่งงานเพิ่มภาระ store; action ใหม่กระทบการสร้างงาน |
| Outbox และ RabbitMQ | กระจาย event เดียวไปหลาย action; แยกการขยายและ retry; ทีมบางส่วนมีประสบการณ์ | เพิ่ม broker นอกเหนือจาก stores; ต้องดูแล routing, confirms, replay และ schema; ไม่แก้ exactly once หรือโควตา MailFast โดยลำพัง |

## Decision

เสนอเลือก **ทางเลือก 3: PostgreSQL transactional outbox + RabbitMQ และ consumer แยกตาม action** เพราะการเพิ่มผู้รับ event ปีหน้าและประสบการณ์ RabbitMQ ของทีมสนับสนุนแนวทางนี้ (**[Inferred]** จาก `docs/scenario.md:14,29`) ยอมรับภาระ broker เพิ่มเพื่อ modifiability; หากแผนปฏิบัติการไม่เหมาะกับทีม ให้กลับมาเทียบทางเลือก 2 โดยคงกลไกป้องกันผลซ้ำไว้ ผู้ใช้ยืนยันว่าประชุมแล้วและคงแบบเดิม Option 3 จึงบันทึกเป็น `Accepted` สำหรับการเลือกแนวทางนี้; ผลรายประเด็นและข้อจำกัดที่ยังไม่ยืนยันอยู่ใน Group Review Summary และ Open questions

รายละเอียดต่อไปนี้เป็น **ข้อเสนอการออกแบบใหม่ทั้งหมด**:

1. **รับ request และเก็บ intent:** ให้ผู้เรียกใช้ `checkout_attempt_id` เดิมเมื่อ retry ผูกกับตัวตนลูกค้าที่ตรวจสอบแล้ว และบันทึก `order_id`/fingerprint อย่างทนทานก่อน payment/shipping key เดิมกับข้อมูลเดิมต้องกลับไป order/สถานะเดิม; key เดิมกับข้อมูลต่างให้ปฏิเสธ อย่าเก็บข้อมูลบัตรใน intent หรือ event การสร้าง UUID ใหม่ต่อ request ปัจจุบันยังไม่ใช่กลไกนี้ (`system/online-boutique/src/checkoutservice/main.go:233,252`)
2. **ยอมรับ order และสร้าง event:** เมื่อยืนยัน payment/shipping สำเร็จแล้ว ให้ transaction เดียวบันทึก order เป็น `PLACED` พร้อม `OrderPlaced` ใน PostgreSQL outbox ก่อนตอบสำเร็จ ใช้ event ID คงเดิมและ unique key ตาม order/ชนิด event ไม่มี distributed transaction ครอบคลุมผลภายนอก
3. **ส่งต่อ:** outbox relay ส่งแบบ at-least-once ไป exchange/binding/คิวที่เตรียมอย่างทนทาน ใช้ publisher confirms และตรวจข้อความที่ route ไม่ได้ก่อนทำเครื่องหมายส่งแล้ว broker ล่มให้ event รอใน outbox ไม่ให้ checkout รอ broker และไม่ใช้ goroutine หรือ log เพียงอย่างเดียวแทน durable record
4. **แยก action:** ให้ email, loyalty และ analytics มีคิว/consumer/retry แยกกัน เพื่อให้ทุก action ได้ event ของตนเอง ACK หลังบันทึกผลที่จำเป็นอย่างทนทาน; dead-letter คือ “งานยังไม่เสร็จ” การเพิ่มคิวใหม่ไม่ทำให้ได้ event ในอดีตเอง จึงต้องกำหนดวิธี replay และ retention
5. **อีเมลและ analytics:** ย้ายการส่งอีเมลออกจาก checkout ไป consumer ที่เชื่อม MailFast แทน mock จำกัดการส่งรวมทุก replica และ retry ≤ 50/วินาที; ยืนยัน API และนโยบายกรณีรับแล้วคำตอบหายก่อนทำ adapter analytics บันทึก “order placed” พร้อมกันซ้ำตาม order/ชนิด event ก่อน ACK (`docs/scenario.md:10–12,26`)
6. **แต้ม:** ใช้ loyalty ledger ที่แยกทรัพยากร/ความล้มเหลวจาก order store มี unique key ตาม `order_id` กับชนิดการให้แต้ม บันทึกการกันซ้ำและเพิ่มแต้มใน transaction เดียวก่อน ACK เมื่อข้อความซ้ำหลัง commit ให้คืนผลเดิม เป้าหมายคือผลธุรกิจครั้งเดียว ไม่ใช่อ้างว่า broker ส่ง exactly once (`docs/scenario.md:27–28`)
7. **ข้อมูลและการดูแล:** event มี version, event ID, order ID, เวลา commit และข้อมูลขั้นต่ำสำหรับแต่ละ action จำกัดสิทธิ์ข้อมูลติดต่อและไม่แนบข้อมูลบัตร กำหนดผู้ดูแล routing, stores, backup/restore, คิวค้าง และ reconciliation ก่อนใช้งานจริง; ยังไม่มีการมอบหมายบุคคลจริง

**การกู้คืนผลภายนอก:** โค้ดปัจจุบันเรียก payment/shipping ก่อนสร้างผล order (`system/online-boutique/src/checkoutservice/main.go:252–279`) outbox ไม่ปิดช่องว่างนี้ทั้งหมด จึงเสนอขั้นตอนต่อไปนี้:

| กรณี | การจัดการที่เสนอ |
|---|---|
| ก่อนแต่ละ external call | บันทึก intent และ operation ID คงเดิมก่อนเรียก |
| ไม่ทราบผลหรือบันทึกผลไม่ได้ | คงสถานะตรวจสอบ ห้ามถือว่าไม่เกิดผลและ retry โดยเดา; ค้นผลจากบริการก่อน |
| payment/shipping ยืนยันสำเร็จ | commit `PLACED`/outbox ต่อ; request เดิมกลับมาที่ order เดิม |
| payment สำเร็จ แต่ shipping ล้มเหลว/ไม่ทราบผล | ให้ผู้รับผิดชอบ reconciliation ตรวจหลักฐานและผู้มีอำนาจตัดสินการดำเนินต่อ/ยกเลิก/คืนเงินตามนโยบาย |

ความสามารถ idempotency, query สถานะ และ compensation ของบริการภายนอก **Not evidenced in the provided files** หากยังยืนยันไม่ได้ ให้คงเคสตรวจสอบพร้อม operation IDs และผู้รับผิดชอบ ไม่อ้างว่าการทำซ้ำหรือคืนเงินอัตโนมัติปลอดภัย

**SLA และขอบเขต:** **[Assumption]** เสนอเริ่มนับเมื่อ commit `PLACED`/outbox จน loyalty commit รวม relay, คิว และ retry ต้อง ≤ 60 s; ต้องให้ธุรกิจรับรองนิยามนี้ และวัด latency ของ request ตั้งแต่รับ request จริงแยกกัน ห้ามเลื่อนจุดเริ่มเพื่อซ่อนความล่าช้าก่อน commit ไม่กำหนดงบเวลาย่อยขึ้นเองจากข้อมูลที่ไม่มี

**[Inferred]** หากเส้นทางแต้มล่มเกินหนึ่งนาที จะรักษาทั้งการรับ order ต่อและแต้มทันเวลาสำหรับทุก order ไม่ได้ ข้อเสนอคือรับ order ต่อเมื่อ order store พร้อมและมีพื้นที่ เก็บงานและตามให้แต้มเมื่อฟื้นตัว โดยรายงานกรณีช้าเป็นการละเมิด SLA เดิมจนกว่าธุรกิจจะตัดสินข้อยกเว้น (`docs/scenario.md:27–28`) พื้นที่เก็บจำกัดไม่รับประกันการรองรับ outage ไม่จำกัดเวลา

**ความจุ:** ใช้ระยะพีค `T_peak` และอัตราส่งสำเร็จหลัง retry `mu_email ≤ 50/s` ที่วัด/ยืนยันจริง ประมาณ backlog เป็น `Q_end ≈ Q0 + (200 − mu_email) × T_peak`; จะระบายได้เมื่ออัตราใหม่ต่ำกว่าอัตราสำเร็จ ขณะ broker ล่ม outbox อาจเพิ่ม 200 event/วินาที ไม่มีข้อมูลระยะพีค/อัตราหลังพีค/ขนาด event จึงยังไม่ระบุความจุ production (`docs/scenario.md:25–26`)

## Group Review Summary

**วันที่ประชุม:** 2026-10-27 ตามคำตอบผู้ใช้ว่า “วันนี้”  
**แหล่งข้อมูล:** ผู้ใช้ยืนยันว่าประชุมแล้ว คงแบบเดิม Option 3 และให้บทบาทพร้อมความเห็นในการทบทวนครั้งนี้ เนื้อหาด้านล่างใช้ข้อมูลที่ผู้ใช้ยืนยัน ไม่ใช้บทสนทนาจากไฟล์ตัวอย่างจำลอง

| เรื่อง | ข้อมูล/ผลที่ผู้ใช้ยืนยัน | ขอบเขตที่ยังเปิดอยู่ |
|---|---|---|
| Architecture | ผู้ใช้ยืนยันหลังเห็นรีวิวอิสระว่าคง Option 3 พร้อมเงื่อนไขสามข้อก่อนใช้งานจริง | การยืนยันแบบไม่ใช่ผลทดสอบหรือการปิด Open questions |
| Loyalty SLA | ชวภณในบทบาท AI analyst เลือกประเด็นแต้มภายใน 1 นาทีขัดกับ availability เมื่อ loyalty ล่มนาน ว่าเป็นข้อค้นพบที่มีประโยชน์ที่สุด | ไม่ใช่การรับรองนิยาม t0 หรือการเปลี่ยน SLA เดิม |
| Loyalty outage > 60 s | ความเห็นที่ผู้ใช้เลือกตระหนักถึงข้อขัดแย้งระหว่างสองเป้าหมาย | ยังไม่แจ้งมติข้อยกเว้น/การชดเชย ต้องคงรายงาน SLA breach ตามข้อกำหนดเดิม |
| Payment/Shipping recovery | คงข้อเสนอเดิมตามมติเลือก Option 3 โดยรวม | ไม่ได้รับรายละเอียดอภิปรายหรือการยืนยัน API/reconciliation รายประเด็น |
| Capacity & operations | ชวภณเลือกว่าจุดอ่อนของรีวิวคือยังประเมินภาระดูแลทีม 4 คนไม่ได้จากข้อมูลจริง | ยังไม่มีข้อมูลขนาด production ผลทดลอง หรือผู้รับผิดชอบเพิ่ม |

### Deciders / Attendance Record

วันที่และบทบาทบันทึกตามคำยืนยันของผู้ใช้ ไม่สมมติรายชื่อหรือบทบาทเพิ่มเติม

| ชื่อ | บทบาทที่ผู้ใช้ยืนยัน | วันที่ประชุม | สิทธิ์ลงมติ |
|---|---|---|---|
| ชวภณ สุวรรณะ | AI analyst, Product owner และ SRE | 2026-10-27 | ยังไม่ได้แจ้ง |
| ณัฐเมธีโพเลิศสวัส | สถาปนิก | 2026-10-27 | ยังไม่ได้แจ้ง |

ผู้ใช้ยืนยันล่าสุดว่าชวภณ สุวรรณะรับบท Product owner และ SRE เองทั้งคู่ เพิ่มจาก AI analyst รายชื่อที่แจ้งมี 2 คน ไม่เพิ่มผู้เข้าร่วมหรือบทสนทนาที่ไม่ได้รายงาน

### Decision Record

- **Proposed Decision:** Option 3 — PostgreSQL transactional outbox + RabbitMQ
- **Final Decision:** คง Option 3 โดยมีเงื่อนไขก่อนใช้งานจริง: (1) ให้ธุรกิจรับรองนิยาม SLA แต้มและนโยบายเมื่อ loyalty ล่มนาน (2) ระบุความจุ ผู้ดูแล และแผนกู้คืนโดยเทียบภาระกับทางเลือก 2 (3) เก็บหลักฐานทดสอบ 200 order/s รวม crash ข้อความซ้ำ และผลภายนอกไม่แน่นอนก่อนอ้างว่าผ่าน
- **Reason:** ผู้ใช้ยืนยันแบบเดิมและเงื่อนไขทั้งสามหลังเห็นผลรีวิวในแชตใหม่ เหตุผลเทคนิคของข้อเสนออยู่ใน Decision ไม่เพิ่มคำพูดหรือเหตุผลของสมาชิกที่ไม่ได้แจ้ง
- **Decided By:** กลุ่มตามคำยืนยันผู้ใช้; รายชื่อที่แจ้งคือ ณัฐเมธีโพเลิศสวัส (สถาปนิก) และ ชวภณ สุวรรณะ (AI analyst, Product owner และ SRE); ไม่ได้แจ้งสิทธิ์ลงมติรายบุคคล
- **Decision Date:** 2026-10-27
- **ADR Status:** Accepted — การเลือกแนวทางมีเงื่อนไขก่อนใช้งานจริงตามที่ผู้ใช้ยืนยัน ไม่ใช่หลักฐานว่าระบบผ่าน SLA
- **Review Evidence:** ผลรีวิวอิสระ `exercise/outputs/3-review.md`, ฉบับที่รีวิว `exercise/outputs/adr-reviewed-snapshot.md` และคำยืนยันหลังรีวิวของผู้ใช้ใน `exercise/04-review-board.md` / `exercise/00-prompt-log.md`

การเลือกแนวทางไม่ใช่ผลทดสอบ การยืนยัน API หรือการปิด Open questions ทุกข้อ ผลรีวิวของ AI แยกจากมติของผู้ใช้ และไฟล์ประชุมจำลองไม่ใช้เป็นหลักฐานเหตุการณ์จริง

## Consequences

- **Positive:** **[Inferred]** ลดการรอ downstream ใน checkout และแยกความล้มเหลวของ consumer (**performance, availability**; `docs/scenario.md:24,26,28`) แต่ต้องวัดต้นทุนการเขียน intent/order/outbox ทั้งหมด
- **Positive:** **[Inferred]** เพิ่ม action และขยายแต้มแยกจากอีเมลได้ (**modifiability, scalability**; `docs/scenario.md:14,25–27`)
- **Negative:** **[Inferred]** broker และ stores เพิ่มงานดูแลให้ทีม 4 คน; อีเมลสะสมและแต้มอาจช้าระหว่าง outage (**operability, cost, timeliness**; `docs/scenario.md:25–30`)
- **Risks and mitigations:** ป้องกัน request ซ้ำด้วย intent/key และ event ซ้ำด้วย ledger transaction; retention ต้องครอบคลุม replay และทดสอบ crash หลัง commit ก่อน ACK (**correctness, durability**; เป้าหมาย `docs/scenario.md:27`; วิธีเป็นข้อเสนอ)
- **Risks and mitigations:** การสูญหาย/คิวเต็ม/route ไม่ถึงต้องมีการตรวจนับงานค้าง แจ้งเตือนและแผน restore; order store เป็น dependency ใหม่ที่ต้องมีแผนความพร้อมใช้งาน (**availability, durability**; เป้าหมาย `docs/scenario.md:28–30`; วิธีเป็นข้อเสนอ)
- **Risks and mitigations:** MailFast รับแล้วไม่ตอบอาจทำให้อีเมลซ้ำ ต้องยืนยัน idempotency/query สถานะหรือตกลงนโยบายก่อนใช้ (**reliability, user experience**; `docs/scenario.md:26` ไม่ได้ให้รายละเอียด API)

## Validation

เป็นแผนที่ยังไม่ได้รัน ไม่ใช้ผลคำนวณแทนผลระบบจริง และไม่เรียกชำระเงิน/จัดส่งจริงในการทดลองโดยพลการ

| การตรวจ | เกณฑ์และหลักฐาน |
|---|---|
| โหลด checkout/MailFast | 200 order/s ตลอดระยะพีคที่ธุรกิจยืนยัน; p95 ≤ 500 ms รวมการเขียนทั้งหมด; จำลอง MailFast p95 1.5 s, p99 8 s, ผิดพลาด 0.5%; ส่งรวม retry ≤ 50/s (`docs/scenario.md:24–26`) |
| แต้มและผลซ้ำ | วัดทุก order จากจุดเริ่มที่ธุรกิจรับรองถึง loyalty commit; แต้มครั้งเดียวและ ≤ 60 s ตามเงื่อนไขที่ประกาศว่าผ่าน รวมงานที่ยังค้างเมื่อครบกำหนด ไม่ใช้เพียงค่าเฉลี่ย/p95 (`docs/scenario.md:27`) |
| Request/event ซ้ำ | key เดิมคืน order/สถานะเดิม; ข้อมูลต่างกับ key เดิมถูกปฏิเสธ; crash หลัง ledger commit ก่อน ACK ต้องไม่เพิ่มแต้มซ้ำ; analytics ไม่เพิ่มรายการธุรกิจซ้ำ |
| External effects | แทรก crash ก่อน/หลัง payment/shipping และการ commit; ตรวจ intent/operation IDs และขั้นตอน reconciliation; ไม่มี retry โดยเดาผล; order ที่ยืนยันสำเร็จมี event ที่ส่งต่อได้ |
| Broker/consumer/outage | ปิดแต่ละส่วนแยกกันเมื่อ order store พร้อม; checkout ไม่ล้มเหลวเพราะส่วนนั้น หลังฟื้นตัวทำงานครบ; outage แต้มเกิน 60 s ต้องแจ้งและนับ SLA breach ไม่สรุปว่าผ่านข้อกำหนดเดิม (`docs/scenario.md:27–28`) |
| ความจุ/การดูแล/action ใหม่ | วัดขนาด event และอัตราระบายจากระยะพีคที่ยืนยัน ทดสอบ routing, พื้นที่เต็มและ restore พร้อมนับงานหาย/ซ้ำ ให้ทีมทดลองกู้คืนและเพิ่ม consumer แล้วเทียบภาระกับทางเลือก 2 (`docs/scenario.md:14,25,29–30`) |

## Open questions

- ผู้ใช้ยืนยันวันที่ประชุมและบทบาทของ ณัฐเมธีโพเลิศสวัส (สถาปนิก) กับ ชวภณ สุวรรณะ (AI analyst, Product owner และ SRE) แล้ว โดยชวภณรับบท Product owner และ SRE ด้วย; ยังไม่แจ้งสิทธิ์ลงมติรายบุคคล และเจ้าของ order store, ledger, broker หรือ reconciliation ในระบบที่จะใช้งานจริง
- ธุรกิจนิยามเวลา “วางคำสั่งซื้อ” อย่างไร และจะตัดสินข้อขัดแย้ง SLA แต้มกับ availability เมื่อ outage ยาวอย่างไร? (`docs/scenario.md:27–28`)
- ระยะพีค อัตราหลังพีค อายุอีเมลที่ยอมรับ และช่วง outage/replay ที่ต้องรองรับเป็นเท่าใด? (`docs/scenario.md:25–26`)
- payment/shipping และ MailFast รองรับ operation ID, idempotency, query สถานะ และการจัดการผลไม่แน่นอนอย่างไร? **Not evidenced in the provided files**
- สูตรแต้ม บัญชีสมาชิก การคืน/ยกเลิกแต้ม และนโยบาย order ที่ยังไม่ `PLACED` คืออะไร? (`docs/scenario.md:11,27`)
- analytics ใช้ schema/ที่เก็บใด ต้องบันทึกภายในเวลาเท่าใด และข้อมูลส่วนบุคคลใดที่แต่ละ consumer ได้รับ? (`docs/scenario.md:12`)
- topology, replication, retention, backup/restore และงบทรัพยากรของ PostgreSQL/RabbitMQ เป็นอย่างไร? ยังไม่มีตัวเลขสำหรับยืนยันว่าเหมาะกับทีม (`docs/scenario.md:29–30`)
- fraud scoring ปีหน้าจะยังเป็นงานหลังสั่งซื้อหรือเงื่อนไขก่อนรับ order? หากเป็นอย่างหลังต้องทบทวนขอบเขต (`docs/scenario.md:14`)
