**ภาษา:** ภาษาไทย | [English](bonus-recover-decision.en.md)

# โบนัส (ไม่บังคับ ทำที่บ้าน): กู้คืนการตัดสินใจที่ไม่ได้บันทึกไว้ (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T5 (Dhar et al. สร้างการตัดสินใจจากโค้ดและแผนภาพเพื่อลด *การสูญหายของความรู้* หรือ knowledge vaporization), AICH4
> **คำถาม:** เมื่อ AI อธิบายว่า*ทำไม*ระบบจึงถูกสร้างแบบนี้ คำอธิบายส่วนใดมาจากหลักฐาน และส่วนใดเป็นการเดาอย่างมั่นใจ

## ลงมือทำ

เริ่มแชตหรือ session ใหม่ แล้วส่งข้อความนี้ให้ agent

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
language: Thai
```

บรรทัด `language: Thai` ทำให้ ADR ที่กู้คืนเป็นภาษาไทย หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้

agent จะเขียน ADR ที่กู้คืนมา พร้อมตาราง **Claim check** ไว้ที่ [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md)

## ตรวจสอบตาราง claim check

เลือกข้อกล่าวอ้าง**ห้าข้อ**จากตารางของ agent โดยเป็นข้อที่ agent ระบุว่า *Evidenced* (มีหลักฐาน) อย่างน้อยสองข้อ และ *Inferred* (อนุมานเอา) อย่างน้อยสองข้อ แล้วตรวจสอบแต่ละข้อด้วยตัวเอง

| # | ข้อกล่าวอ้าง (ย่อ) | AI ระบุว่า | นิสิตระบุว่า | เหตุผล (path ของไฟล์หรือการให้เหตุผล) |
|---|---|---|---|---|
| C1 | หลาย service definitions อยู่ใน `demo.proto` | Evidenced | Evidenced | `system/online-boutique/protos/demo.proto` (`service CartService`, `service CheckoutService`) |
| C2 | `frontend` รับ HTTP และสร้าง gRPC clients เจ็ดบริการ | Evidenced | Evidenced | `system/online-boutique/src/frontend/main.go` (`r.HandleFunc`, `mustConnGRPC`) |
| C5 | ลำดับ checkout ต้องใช้ผลก่อนตอบ | Inferred | Inferred | เห็นลำดับ RPC ใน `system/online-boutique/src/checkoutservice/main.go` (`PlaceOrder`) แต่คำว่า “ต้อง” เป็นเหตุผลที่อนุมาน |
| C6 | การรักษาชนิดข้อมูลร่วมเป็นแรงจูงใจ | Inferred | Inferred | `demo.proto` มีชนิดข้อมูลร่วม แต่ไม่บอกเหตุผลของผู้ตัดสินใจ |
| C15 | latency และต้นทุนแก้สัญญาอาจเพิ่ม | Inferred | Inferred | เป็นผลกระทบที่เป็นไปได้จาก RPC และไฟล์สัญญาร่วม; ไม่มีผลวัดจริง |

**นับ:** จากข้อกล่าวอ้างทั้งหมด agent ระบุว่า *Evidenced* กี่ข้อ 4 ตามการตรวจหลักฐานในร่างนี้ 4

## สะท้อนคิด

1. ในห้าข้อที่ตรวจ ไม่พบการเดาที่ถูกติดป้าย *Evidenced*: service definitions และการตั้งค่า HTTP/gRPC ของ `frontend` มีอยู่ในไฟล์ที่อ้าง ส่วนแรงจูงใจในการใช้ชนิดข้อมูลร่วมถูกติดป้าย *Inferred*.
2. คนใหม่อาจเข้าใจว่าระบบทั้งหมดใช้ gRPC ทั้งที่ `frontend` รับ HTTP และหลักฐานที่ให้ดูยังไม่ครอบคลุม implementation ทุก service (`src/frontend/main.go`, `http.ListenAndServe`).
3. โค้ดช่วยกู้คืน interface, protocol ที่เห็น และลำดับการเรียกได้ เช่น `grpc.NewClient` กับ `PlaceOrder`. แต่เหตุผลที่ผู้สร้างเลือก `demo.proto` ไฟล์เดียวไม่ปรากฏในโค้ด ต้องถามผู้ตัดสินใจเดิมหรือหา design record.
