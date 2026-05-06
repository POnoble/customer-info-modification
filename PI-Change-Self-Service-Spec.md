# Customer Information Modification · Self-Service Spec

**Module:** PI Change Request — Customer Self-Service Portal
**Version:** 2.0
**Last Updated:** May 2026
**Owner:** Customer Experience & Platform Design (Ketkanok Jongcham)

---

## 1. Goal

ลดงานของ Sales/CC ในกระบวนการเปลี่ยนข้อมูลส่วนบุคคล โดยให้ลูกค้ากรอกข้อมูลเองผ่านลิงก์ที่ส่งให้ — ครอบคลุมทั้งลูกค้าที่ยังอยู่ในสัญญา และเจ้าของห้องคนล่าสุดที่เข้าผ่าน Noble ID — กระบวนการเดิม (POS เมนู 2.6) ยังคงเปิดใช้งานเป็น fallback

## 2. Scope

**In scope (V2)** — Web self-service flow 5 ขั้นตอน · ID/Passport + DOB verification · Dual-channel OTP (SMS + Email) · Multi-unit selection · Progressive disclosure form · Auto-generated bilingual PDF · Digital signature · Email routing · POS menu 2.6 integration

**Out of scope (Phase 2)** — Native mobile app · NDID biometric · Co-customer joint signing · LINE Login · Auto address verification API

## 3. User Roles

| Role | สิทธิ์ |
|---|---|
| **Customer (Thai)** | ใช้บัตรประชาชน + วันเกิด · ทำเรื่องผ่านลิงก์ ดูสถานะของตน |
| **Customer (Foreign)** | ใช้ Passport + วันเกิด · ทำเรื่องเป็นภาษาอังกฤษได้ทั้งหมด |
| **Customer (Noble ID Secondary Owner)** | ลูกค้าที่ซื้อต่อจากเจ้าของเดิมและเข้าผ่าน Noble ID |
| **Sales Advisor** | Generate link · ดูคำขอผ่าน POS เมนู 2.6 · ทำเอง (fallback) |
| **Customer Care (CC)** | เหมือน Sales + ขยายอายุลิงก์ + ส่งซ้ำ |
| **Project Reviewer** | Approve/Reject (โครงการที่ยังเปิดขาย) · Push to REM |
| **Application Specialist** | Approve/Reject (โครงการที่ปิดแล้ว) |
| **System Admin** | Audit log · Unlock OTP · Revoke link |

## 4. Five-Step Customer Flow

| Step | Screen | Inputs | Key Outputs |
|---|---|---|---|
| **1.1** | Identity Entry | ID Card / Passport · **วันเกิด (DD/MM/YYYY)** | OTP request |
| **1.2** | OTP Challenge | OTP 6 หลัก + Reference code | Verified session (TTL 30 นาที) |
| **2** | Unit Selection | เลือกห้องที่ต้องการเปลี่ยน | Selected unit IDs |
| **3** | Change Information Form | เลือกฟิลด์ + กรอกค่าใหม่ + แนบเอกสาร | Validated changes |
| **4** | Review & Sign | ตรวจ PDF + วาดลายเซ็น + PDPA consent | Signed REQ |
| **5** | Success | คำขอเข้าระบบ | REQ-XXXX-A/B + PDF download |

### Eligibility Rules

ลูกค้าต้องมีข้อมูลใน CRM โดย `id_card = input AND dob = input` AND อย่างน้อย 1 ห้องอยู่ในสถานะ:
- `Completed` (โอนกรรมสิทธิ์แล้ว) หรือ `Active Contract` (อยู่ระหว่างสัญญา)
- หรือเป็น Noble ID Secondary Owner

## 5. Identity Verification Logic

```
1. Match (id_no, dob) ใน CRM → ถ้าไม่พบ: ปฏิเสธ + แนะนำติดต่อ Sales
2. Generate OTP 6 หลัก + Ref 6 ตัวอักษร · TTL 5 นาที
3. ส่ง OTP **2 channel พร้อมกัน**:
   - SMS gateway → เบอร์ที่ลงทะเบียน
   - Email service → อีเมลที่ลงทะเบียน
4. ลูกค้ากรอก OTP → verify
   - ถูก: mark verified · TTL 30 นาที · โหลดห้อง
   - ผิด ≤ 4 ครั้ง: เพิ่ม counter
   - ผิด ≥ 5 ครั้ง: lockout 30 นาที
```

**Rate limits:** 3 OTP / phone+email / hr · 5 attempts / IP / hr · token ลิงก์ TTL 24 ชม. ใช้ได้ 1 ครั้ง

## 6. Two Customer Scenarios at Step 2

### Scenario A · Original Buyer (มีสัญญากับ Noble)
- Lookup: `WHERE buyer_id = customer_id` ใน CRM contracts
- แสดงห้อง + **checkbox เลือกได้** (รองรับ multi-select + Select All)
- ระบบสร้าง **REQ แยกต่อ unit** (5 ฟิลด์ × 2 ห้อง = 10 changes ใน 2 REQ)

### Scenario B · Noble ID Secondary Owner (ซื้อต่อจากเจ้าของเดิม)
- Lookup: `WHERE noble_id_owner = customer_id`
- แสดงห้อง + **checkbox lock** (preselect ทั้งหมด)
- มี info banner อธิบายว่าข้อมูลจะมีผลกับห้องทั้งหมด
- ระบบยังคงสร้าง REQ แยกตาม unit (เพราะ POS เก็บข้อมูลแยกตามโครงการ/ห้องเสมอ)

## 7. Step 3 — Form Field Specs

| Field | Conditional | Required Subfields | Required Doc |
|---|---|---|---|
| ชื่อ-สกุล | All customers | คำนำหน้า/ชื่อ/นามสกุล × TH+EN (6 fields) + reason radio | ปถ.14 (กรณีเปลี่ยนชื่อ) ไม่ต้อง (กรณีแก้คำสะกดผิด) |
| Passport + Country | Foreign only | Passport No. + Country of Issue | สำเนา Passport ทุกหน้าที่มีข้อมูล |
| เลขบัตรประชาชน | Thai only | 13 digits + auto-format | สำเนาบัตรประชาชนใหม่ |
| อีเมล | All | Email + Confirm Email | — (Noble ID member มี Activation flow) |
| เบอร์โทร | All | Sub-type radio (Add/Change) + Phone | — |
| ที่อยู่ตามบัตร | All | Country + 9 sub-fields × TH/EN | สำเนาทะเบียนบ้าน |
| ที่อยู่จัดส่ง | All | Country + 9 sub-fields × TH/EN | — |
| Receipt Type | Active contract only · Paper→E-Receipt only | Radio (E-Receipt) | — |

### Smart Auto Features

- **TH ↔ EN auto-mirror** — พิมพ์ฝั่งใดฝั่งหนึ่ง → เติมอีกฝั่งให้อัตโนมัติ (สำหรับฟิลด์ตัวเลข/ชื่ออาคาร)
- **Foreign customer name** → พิมพ์ EN ได้ TH ตามอัตโนมัติ + Title mapping (Mr.→นาย, Mrs.→นาง, Ms.→นางสาว)
- **Auto postal code** จาก sub-district selection (ลบ/แก้เองได้)
- **Auto-format ID card** เป็น `X-XXXX-XXXXX-XX-X` พร้อม counter `0/13`
- **Cascading dropdown** จังหวัด → อำเภอ → ตำบล (เฉพาะประเทศไทย)
- **Free text** province/district/sub-district เมื่อเลือกประเทศที่ไม่ใช่ไทย (ใส่ "-" ได้)
- **Auto-save** ทุก 30 วินาที

### Validation Rules
- ฟิลด์ที่ติ๊กแล้วต้องกรอกครบ (highlight สีแดงถ้าไม่ครบ)
- Email + Confirm Email ต้องตรงกัน
- ID Card ต้อง 13 หลัก
- บังคับแนบสำเนาบัตร/Passport อย่างน้อย 1 ไฟล์ (ครอสหมายเอกสารตามคำเตือน)

## 8. Noble ID Email Activation Flow

ลูกค้าที่เป็นสมาชิก Noble ID เปลี่ยนอีเมล:
1. หลัง Sales/CC อนุมัติ → ระบบส่ง **Activation Email** ไปอีเมลใหม่
2. ลูกค้าคลิก **Activate link** ในอีเมล (TTL 24 ชม.)
3. ระบบ confirm → อัปเดต Noble ID + REM
4. ถ้าไม่ activate ภายใน 24 ชม. → email ไม่ถูกอัปเดตใน Noble ID · ต้องแจ้งคำขอใหม่

แสดง notice ที่ Step 3 (Email field) อธิบาย flow นี้ให้ลูกค้าทราบล่วงหน้า

## 9. Step 4 — Document & Signature

- ระบบ generate **PDF อัตโนมัติ** ตามรูปแบบ Personal Information Change Form ของ Noble (ขีดสีดำ-เทา-ส้ม-สาน-เหล็กบนล่าง · 2 ภาษา TH+EN ในไฟล์เดียว)
- **1 PDF ต่อ 1 unit** เรียงต่อกันแนวตั้งใน viewer (ไม่ใช้ tabs)
- Watermark "Draft" จนกว่าจะเซ็น
- รองรับ **Digital signature** ผ่าน HTML5 Canvas (ลากเม้าส์/นิ้ว) — ลายเซ็น apply ไปทุก PDF อัตโนมัติ
- กลับไปแก้ข้อมูลที่ Step 3 → **ลายเซ็นถูกล้างอัตโนมัติ** ต้องเซ็นใหม่
- บังคับติ๊ก 2 checkbox ก่อน Submit:
  1. ตรวจสอบเอกสารถูกต้องแล้ว
  2. ยินยอม PDPA Consent (มีลิงก์ไป `https://www.noblehome.com/th/policy`)

## 10. Step 5 — Success & PDF Download

- แสดง REQ-XXXX-A/B (1 ต่อ unit)
- Download All — รวมเอกสารคำขอที่เซ็นแล้ว + ไฟล์แนบทั้งหมดเป็น PDF เดียว ใช้ `pdf-lib`
- ส่งสรุปทาง email ลูกค้าอัตโนมัติ
- routing email แจ้งทีม:
  - **โครงการเปิดขาย** → `<project>@noblehome.com` + CC team
  - **โครงการปิดแล้ว** → `app-specialist@noblehome.com`

## 11. Bilingual UI (TH/EN Toggle)

- ทุก label, button, hint, warning รองรับ 2 ภาษา (~70 i18n keys)
- ลูกค้าสลับภาษาได้ที่ header ทุกเมื่อ → re-render ทันที
- **ฟิลด์ใน Form (Step 3) ยังคงเป็น 2 ภาษาเสมอ** (TH + EN cards) เพราะเอกสารราชการต้องการครบทั้ง 2 ภาษา
- เอกสาร PDF เป็น 2 ภาษาในไฟล์เดียวเสมอ

## 12. State Machine

```
LinkGenerated → Verifying → Drafting (auto-save 30s)
                ↓                    ↓
            Locked (30m)      PendingReview → Approved → PushedToREM → Completed
                                  ↓                            ↓
                              NeedInfo                      PushFailed → (retry)
                                  ↓
                              Drafting → ...
                              Rejected
```

## 13. New Database Tables (suggested)

| Table | Key columns |
|---|---|
| `pi_change_link` | id, token (unique), customer_id, generated_by, expires_at, used_at |
| `pi_change_request` | req_no, link_id, customer_id, unit_id, status, fields_changed (JSONB), attachments (JSONB), pdpa_consent_at, signature_hash, ip, user_agent, submitted_at |
| `otp_session` (Redis) | session_id, otp_hash, ref, attempts, channels_sent[], expires_at |
| `pi_audit_log` (append-only) | id, ts, event_type, actor_type, actor_id, target_id, ip, user_agent, payload (JSONB) |
| `noble_id_email_activation` | id, customer_id, old_email, new_email, token, sent_at, activated_at |

## 14. APIs (sketch)

```
POST   /api/v1/pi-change/link              (Sales/CC) → { url, token, expires_at }
GET    /api/v1/pi-change/link/{token}      (Customer) → validate
POST   /api/v1/pi-change/identity/lookup   { id_no, dob } → 200 { masked_phone, masked_email, otp_ref } | 404
POST   /api/v1/pi-change/identity/resend   { session } → resend SMS+Email
POST   /api/v1/pi-change/identity/verify   { session, otp } → { units }
GET    /api/v1/pi-change/customer/units    → list (Original Buyer + Noble ID Secondary)
POST   /api/v1/pi-change/request           { unit_ids[], fields, attachments, signature } → { req_nos[] }
GET    /api/v1/pi-change/request/{req_no}/pdf  → merged PDF (form + attachments)
POST   /api/v1/pi-change/request/{req_no}/review  (Reviewer) { decision, note }
POST   /api/v1/noble-id/email/activate     { token } → confirm new email
```

## 15. Security Controls

- HTTPS + HSTS · TLS 1.3 only
- JWT (RS256) สำหรับ link token + jti blacklist หลังใช้
- OTP เก็บ bcrypt hash · ห้ามเก็บ plaintext
- WAF + rate limit (Cloudflare หรือเทียบเท่า)
- File upload: virus scan + ขนาด ≤ 10MB/ไฟล์ · ≤ 50MB total · เก็บ private S3 + signed URL
- Audit log: append-only · เก็บ ≥ 7 ปี ตาม PDPA
- Mask sensitive data ในหน้าจอ (ชื่อ ส****ี · เบอร์ 081-XXX-5678 · อีเมล o****e@gmail.com)
- Cross-mark requirement บนเอกสารแนบ (เพิ่ม layer ป้องกันการนำเอกสารไปใช้ผิดวัตถุประสงค์)

## 16. PDPA Compliance

- แสดง Consent text ในเอกสาร PDF ทุกใบ
- ลิงก์ Privacy Policy: `https://www.noblehome.com/th/policy` (ทุกหน้าที่มี consent)
- ลูกค้ามีสิทธิ์ขอดู / แก้ / ลบ ข้อมูลตนเองได้ (ผ่าน Contact Center)
- เก็บลายเซ็นดิจิทัล + IP + User Agent + Timestamp เป็นหลักฐาน
- รองรับ พ.ร.บ. ว่าด้วยธุรกรรมทางอิเล็กทรอนิกส์ พ.ศ. 2544

## 17. Fallback Path

ลูกค้ากลุ่มไหนยังต้องผ่าน POS เมนู 2.6 (Sales กรอกแทน):

- ลูกค้าที่ข้อมูลใน CRM ไม่ครบ (ไม่มีอีเมล/วันเกิด/เบอร์)
- ลูกค้าที่ verify OTP ไม่ผ่าน 5 ครั้ง → ระบบล็อก
- ลูกค้าที่ไม่สะดวกใช้ smartphone (สูงอายุ)
- กรณีพิเศษที่ต้องการเอกสารแนบเฉพาะ

POS เมนู 2.6 เพิ่ม column **Source** (📱 Customer / 👤 Sales) และ Status filter ใหม่ (Pending Review / Need Info / Approved / Rejected / Failed REM)

## 18. Rollout

| Phase | เดือน | Scope |
|---|---|---|
| 1 — Foundation | M+1 | Data cleansing (เบอร์ + email + DOB ใน CRM), SMS+Email gateway, Identity service |
| 2 — Customer Portal | M+2 | Build 5 screens · TH/EN i18n · Auto-features |
| 3 — POS Integration | M+3 | Source column, Approval flow, REM push, Email routing |
| 4 — Pilot | M+4 | 1-2 โครงการ · เก็บ feedback · KPI tracking |
| 5 — Full Launch | M+5 | ทุกโครงการ · LINE OA channel · Tenant flow |

## 19. Success Metrics (6 เดือนหลัง launch)

- Self-service completion rate ≥ 70%
- Time-to-REM < 4 ชม. (จากเดิม 1–3 วัน)
- Sales effort/คำขอ < 5 นาที (จากเดิม ~25 นาที)
- Data error rate < 1% (จากเดิม ~6%)
- OTP first-try success ≥ 92%
- Noble ID Email activation rate ≥ 85%
- TH/EN usage ratio ~ 95% TH / 5% EN

## 20. Open Questions

1. SMS gateway — Thaibulksms / AIS Cloud / Twilio?
2. Email service — SendGrid / AWS SES / on-premise?
3. PDPA Officer ต้อง sign-off design ก่อน production หรือไม่?
4. Co-customer (เจ้าของห้องร่วม 2+ คน) ต้องเซ็นทุกคนหรือคนเดียวพอ? — Phase 2
5. ระยะเวลาที่ลูกค้าจะ activate email ของ Noble ID — ถ้าหมดเวลาควรทำอย่างไร?

---

*ดูเอกสารประกอบ:*
- `PI-Customer-App-V2.html` — Working prototype (5 screens · all features)
- `System-Architecture.html` — Architecture diagrams · sequence flow · state machine
- `index.html` — Landing page รวม link
