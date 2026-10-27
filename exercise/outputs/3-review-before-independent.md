Agent: Codex; Model: GPT-6

# รีวิว ADR ฉบับปรับปรุงก่อนการประชุม

เป้าหมาย: `docs/adr/0002-use-rabbitmq-for-post-order-actions.md` สถานะ `Proposed` ซึ่ง AI ช่วยร่างตามคำขอ ยังรอนิสิตตรวจรับและกลุ่มยังไม่ได้ประชุม รีวิวนี้ทำในแชตเดียวกับการแก้เอกสาร จึงไม่ใช่การประเมินอิสระใน session ใหม่ตามขั้นตอนใบงาน และไม่ใช่มติกลุ่ม

หลักฐานของข้อค้นพบ: **ADR** = เอกสารเป้าหมายข้างต้น; **S** = `docs/scenario.md` ไม่มีผลทดสอบจริง ผลรีวิวเบื้องต้นเดิมเก็บที่ `exercise/outputs/3-review-preliminary.md`

## Quality-attribute scenarios

เรียงตาม แหล่ง · สิ่งกระตุ้น · สภาพแวดล้อม · การตอบสนอง · เกณฑ์วัด

1. **latency:** ลูกค้า · ส่ง 200 order/s · พีค 11.11 ขณะ MailFast มี p95 1.5 s / p99 8 s / ผิดพลาด 0.5% · checkout เก็บ intent/order/outbox แล้วตอบโดยไม่รอ consumer · p95 request รวมการเขียนทั้งหมด ≤ 500 ms และส่งอีเมลรวม retry ≤ 50/s (S:24–26; ADR:Decision, Validation)
2. **แต้มและผลซ้ำ:** ผู้เรียก/relay · retry request key เดิมและส่ง event ซ้ำหลัง ledger commit ก่อน ACK · โหลดพีค · คืน order เดิมและไม่เพิ่มแต้มซ้ำ · แต้มครั้งเดียวต่อ order ภายใน 60 s รวม relay/คิว/retry โดยจุดเริ่มยังรอธุรกิจรับรอง (S:25,27; ADR:Decision, Validation)
3. **availability:** ระบบ loyalty · ใช้ไม่ได้เกิน 1 นาที · order store ยังพร้อมและมีพื้นที่ · รับ order เก็บงานและตามให้แต้มหลังฟื้นตัว · order ไม่ล้มเหลวเพราะ loyalty และรายงานแต้มเกินเวลาเป็น SLA breach เดิม ไม่สรุปว่าผ่านเงื่อนไขแต้ม (S:27–28; ADR:Decision)

## Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | **[Inferred]** แยก request idempotency, outbox และ ledger transaction พร้อมแยกความล้มเหลวของ loyalty store (ADR:Decision ข้อ 1–6) | ครอบคลุม request ซ้ำและ event ซ้ำคนละช่วง โดยไม่ทำให้ ledger เป็น dependency ตรงของ checkout; ยังต้องพิสูจน์ด้วย validation |
| Risk | แผน reconciliation ต้องใช้หลักฐานผลภายนอก แต่ API ค้นผล/กันซ้ำและผู้รับผิดชอบยังไม่ยืนยัน (ADR:การกู้คืนผลภายนอก, Open questions) | หลังชำระเงินแล้วแต่ผลไม่ชัด เคสอาจค้างก่อนเกิด event; outbox ไม่แก้ช่องว่างนี้เอง |
| Sensitivity point | ระยะพีค อัตราหลังพีค และอัตราส่งสำเร็จหลัง retry ยังไม่ทราบ (ADR:ความจุ; S:25–26) | 200 order/s กับโควตา 50 อีเมล/s ทำให้คิวสะสมอย่างน้อย 150/s ก่อน retry จึงยังเลือกความจุ/ระยะกู้คืนจริงไม่ได้ |
| Trade-off point | รับ order ต่อเมื่อ loyalty ล่มรักษา availability แต่แต้มอาจเกิน 1 นาที (ADR:SLA และขอบเขต; S:27–28) | การไม่สูญหายไม่เท่ากับทันเวลา ต้องให้ธุรกิจตัดสินข้อขัดแย้งโดยไม่เปลี่ยน SLA เอง |

## Missing or weak

- ยังไม่มีผลวัด checkout ที่รวมการเขียน intent/order/outbox และไม่มีผลทดลองดูแล PostgreSQL/RabbitMQ เทียบ job store สำหรับทีม 4 คน (ADR:Validation; S:24,29–30)
- API ภายนอก สูตรแต้ม นโยบายข้อมูลส่วนบุคคล ผู้ดูแล และ topology/retention ยังเปิดไว้ การอธิบายแผนไม่ใช่หลักฐานว่าติดตั้งหรือทดสอบสำเร็จ (ADR:Open questions)
- เอกสารเปิดเผยความไม่แน่นอนแล้ว แต่ยังไม่สามารถอ้างว่าผ่าน R-12 ทุกกรณี โดยเฉพาะ outage เกินกำหนดและพื้นที่เต็ม (ADR:Decision, Consequences)

## Verdict

**accept with changes — สำหรับข้อเสนอเพื่อนำเข้าประชุม** ยังไม่ใช่การอนุมัติใช้งานจริง สามเรื่องที่ต้องนำไปตัดสินคือ:

1. ให้ธุรกิจยืนยันจุดเริ่มเวลาและข้อขัดแย้งแต้มภายใน 1 นาทีกับ availability พร้อมผู้ตัดสินและเหตุผล (S:27–28)
2. ยืนยันสัญญา payment/shipping/MailFast และผู้รับผิดชอบ reconciliation ก่อนอ้างว่ากู้ผลไม่แน่นอนได้ปลอดภัย (ADR:Decision, Open questions)
3. ยืนยันระยะพีค/อัตราหลังพีคและแผนดูแลของทีม เปรียบเทียบต้นทุนกับทางเลือก 2 และเก็บผล validation จริงก่อนกล่าวว่าผ่านเป้าหมาย (S:24–30; ADR:Validation)
