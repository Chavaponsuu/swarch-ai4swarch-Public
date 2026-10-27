Agent: Codex; Model: GPT-6

# ผลตรวจหลักฐานจริงเพิ่มเติม

**วันที่ตรวจ:** 2026-10-27  
**วิธีตรวจ:** อ่านไฟล์จริงด้วย file APIs หลังจบ run เดิม ไม่รัน terminal ไม่รันระบบ และไม่อ้างว่าการตรวจนี้ทำโดยนิสิตเอง

| สิ่งที่ตรวจ | ผลที่พบ | หลักฐาน |
|---|---|---|
| สัญญาบริการ | 9 service contracts และ 15 RPC | `system/online-boutique/protos/demo.proto`: service/RPC declarations |
| frontend → productcatalogservice | ปลายทาง productcatalogservice:3550 ตรงกับ Service/targetPort 3550 | `system/online-boutique/kubernetes-manifests/frontend.yaml:69–70`; `productcatalogservice.yaml:79–81` ในโฟลเดอร์เดียวกัน |
| frontend → cartservice | cartservice:7070 ตรงกับ Service/targetPort 7070 | `system/online-boutique/kubernetes-manifests/frontend.yaml:73–74`; `cartservice.yaml:80–82` |
| checkoutservice → paymentservice | paymentservice:50051 ตรงกับ Service/targetPort 50051 | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:61–62`; `paymentservice.yaml:79–81` |
| checkoutservice → emailservice | emailservice:5000 ถูกส่งต่อไป targetPort 8080 | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:63–64`; `emailservice.yaml:81–83` |
| recommendationservice → productcatalogservice | productcatalogservice:3550 ตรงกับ Service ของ catalog | `system/online-boutique/kubernetes-manifests/recommendationservice.yaml:60–61`; `productcatalogservice.yaml:79–81` |
| cartservice → redis-cart | redis-cart:6379 ตรงกับ Redis Service/targetPort 6379 และ image redis:alpine | `system/online-boutique/kubernetes-manifests/cartservice.yaml:50–51,118,153–156` |
| อีเมลล้มเหลวใน checkout | บันทึก Warn แล้วตอบ order สำเร็จ ไม่คืน error จากการส่งอีเมล | `system/online-boutique/src/checkoutservice/main.go:273–279` |
| เป้าหมายธุรกิจที่ใช้ใน ADR | checkout p95 ≤ 500 ms, พีค 200 order/s, MailFast 1.5 s/8 s/0.5%/50 ต่อวินาที, แต้มภายในหนึ่งนาที | `docs/scenario.md:24–28` |

ตาราง dependency ยืนยัน configuration กับปลายทาง ไม่ยืนยันว่าทุก RPC ถูกเรียกจริงหรือเรียกตามลำดับใด การตรวจ main.go เพิ่มเป็นขั้นตรวจหลัง recovery ไม่ได้ย้อนเปลี่ยนขอบเขตหลักฐานของ Run A/B

การตรวจเปรียบเทียบ ADR อ่านไฟล์ `exercise/outputs/2a-adr.md` และ `exercise/outputs/2b-adr.md` ฉบับปัจจุบันจริง: Run a ใช้ requirement และเปิดข้อมูลที่ไม่ทราบเป็นคำถาม; Run b ใช้ตัวเลข scenario และเสนอ PostgreSQL/RabbitMQ โดยไม่อ้างว่าเป็นส่วนประกอบเดิม ทั้งสองไฟล์เป็นข้อเสนอ ไม่ใช่ implementation หรือผลทดสอบ
