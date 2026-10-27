**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

> **ที่มาของคำตอบ:** AI ช่วยเรียบเรียงและตรวจซ้ำจากไฟล์จริงวันที่ 2026-10-27 พบ configuration ตรงกัน 6 dependency และตรวจพฤติกรรมอีเมลใน main.go เพิ่มหลังจบ run เดิม ผลนี้เป็นการตรวจจริงโดย AI ไม่อ้างว่านิสิตได้เปิดตรวจไฟล์ด้วยตนเอง ดู [หลักฐานการตรวจ](outputs/verification-evidence.md)

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| `frontend` → `productcatalogservice` | `system/online-boutique/kubernetes-manifests/frontend.yaml:69–70` (`PRODUCT_CATALOG_SERVICE_ADDR`); `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml:79–81` (`grpc`, `port`, `targetPort`) | ✅ โดย AI: ยืนยัน configuration | [ผลตรวจ AI] ค่าปลายทางคือ `productcatalogservice:3550` ตรงกับ Service พอร์ต 3550 และ targetPort 3550; ยังไม่ได้ตรวจ RPC call sites |
| `frontend` → `cartservice` | `system/online-boutique/kubernetes-manifests/frontend.yaml:73–74` (`CART_SERVICE_ADDR`); `system/online-boutique/kubernetes-manifests/cartservice.yaml:80–82` | ✅ โดย AI: ยืนยัน configuration | [ผลตรวจ AI] ค่าปลายทางคือ `cartservice:7070` ตรงกับ Service พอร์ต 7070 และ targetPort 7070 |
| `checkoutservice` → `paymentservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:61–62` (`PAYMENT_SERVICE_ADDR`); `system/online-boutique/kubernetes-manifests/paymentservice.yaml:79–81` | ✅ โดย AI: ยืนยัน configuration | [ผลตรวจ AI] ค่าปลายทางคือ `paymentservice:50051` ตรงกับ Service พอร์ต 50051 และ targetPort 50051; หลักฐานนี้ยังไม่ยืนยันการชำระเงินจริงกับผู้ให้บริการภายนอก |
| `checkoutservice` → `emailservice` | `system/online-boutique/kubernetes-manifests/checkoutservice.yaml:63–64` (`EMAIL_SERVICE_ADDR`); `system/online-boutique/kubernetes-manifests/emailservice.yaml:81–83` | ✅ โดย AI: ยืนยัน configuration | [ผลตรวจ AI] ผู้เรียกใช้ `emailservice:5000`; Kubernetes Service รับที่พอร์ต 5000 แล้วส่งไป targetPort 8080 จึงไม่มีความขัดแย้งระหว่างสองพอร์ต |
| `recommendationservice` → `productcatalogservice` | `system/online-boutique/kubernetes-manifests/recommendationservice.yaml:60–61` (`PRODUCT_CATALOG_SERVICE_ADDR`); `system/online-boutique/kubernetes-manifests/productcatalogservice.yaml:79–81` | ✅ โดย AI: ยืนยัน configuration | [ผลตรวจ AI] ค่าปลายทางคือ `productcatalogservice:3550` ตรงกับ Service ของ catalog |
| `cartservice` → `redis-cart` | `system/online-boutique/kubernetes-manifests/cartservice.yaml:50–51` (`REDIS_ADDR`), `118–120` (`image: redis:alpine`), `143–156` (`redis-cart` Service) | ✅ โดย AI: ยืนยัน configuration | [ผลตรวจ AI] ค่าปลายทางคือ `redis-cart:6379` ตรงกับ Redis Service พอร์ต 6379 และ targetPort 6379; การตั้งค่านี้ยังไม่บอก key schema หรือ TTL |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): **[ผลตรวจ AI]** ไม่พบปลายทางหรือพอร์ตที่ผิดใน dependency 6 รายการที่ตรวจข้างต้น การตรวจนี้ไม่ได้ยืนยัน dependency ทุกแถวหรือการทำงานจริงขณะรันระบบ รายงาน Run B ระบุข้อจำกัดเรื่อง actual calls ไว้ในหัวข้อ Gaps (`exercise/outputs/1B-architecture.md:129`)
  - **จุดที่ยังต้องตรวจเพิ่ม:** `frontend` มี `SHOPPING_ASSISTANT_SERVICE_ADDR=shoppingassistantservice:80` อยู่จริง แต่บรรทัดตั้งค่า `ENABLE_ASSISTANT` ถูก comment (`system/online-boutique/kubernetes-manifests/frontend.yaml:83–94`) จึงยังสรุปไม่ได้ว่าฟีเจอร์นี้เปิดใช้งานหรือมี runtime dependency รายงานแยกประเด็นนี้เป็น unresolved endpoint ใน Gaps (`exercise/outputs/1B-architecture.md:130`) การยืนยันต้องขอหลักฐานเพิ่ม; รอบตรวจนี้ยังไม่ได้อ่านไฟล์เปิดใช้งานหรือ implementation

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

   **[AI ช่วยเรียบเรียงจากหลักฐานที่ตรวจจริง]** Run A ระบุชื่อ service contracts และ RPC ได้จาก `system/online-boutique/protos/demo.proto` แต่ไฟล์นี้ไม่แสดง deployment topology, ผู้เรียกแต่ละบริการ, ภาษา implementation หรือที่เก็บข้อมูล. Run B มีหลักฐาน configuration เพิ่ม เช่น `frontend.yaml:73–74` แสดงปลายทาง `cartservice:7070` และ `cartservice.yaml:50–51,118–120` แสดงการตั้งค่าเชื่อมไป Redis โดยทั้งสองไฟล์อยู่ใต้ `system/online-boutique/kubernetes-manifests/`. อย่างไรก็ตาม environment variables ยืนยันเพียงปลายทางที่ตั้งค่าไว้ ส่วนลำดับการเรียก RPC และพฤติกรรมจริงยังต้องตรวจ implementation เพิ่ม

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

   **[AI ช่วยเรียบเรียงจากหลักฐานที่ตรวจจริง]** ข้อผิดพลาดที่เสี่ยงต่อความเข้าใจคือการตีความลูกศร checkoutservice → emailservice ว่าเมื่อส่งอีเมลล้มเหลวแล้วคำสั่งซื้อจะต้องล้มเหลว ซึ่งไฟล์ deployment ยืนยันพฤติกรรมนี้ไม่ได้. เมื่อเปิด `system/online-boutique/src/checkoutservice/main.go:273–279` ตรวจจริง พบว่าระบบบันทึกคำเตือนแล้วคืน order สำเร็จ. ดังนั้นควรแยก configuration dependency ออกจาก failure behavior และตรวจ implementation เพิ่มก่อนสรุป โดยไม่กล่าวหาว่า Run B อ้างพฤติกรรมผิดอยู่แล้ว
