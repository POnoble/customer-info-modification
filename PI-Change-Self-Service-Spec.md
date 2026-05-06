# PI Change · Self-Service System Spec

**Module:** A · PI Change Request
**Version:** 1.0 (Draft for review)
**Date:** April 2026
**Owner:** Customer Experience & Platform Design

---

## 1. Goal

ลดงานของ Sales/CC ในกระบวนการเปลี่ยนข้อมูลส่วนบุคคลของลูกค้า โดยให้ลูกค้ากรอกฟอร์มเอง ผ่านลิงค์ที่ Sales/CC ส่งให้ — โดยที่กระบวนการเดิม (POS เมนู 2.8) ยังคงเปิดไว้สำหรับ fallback

## 2. Scope (In / Out)

**In scope** — Web self-service flow (A1 Identity Verify → A2 Form → A3 Review/PDPA → A4 Done), Sales/CC trigger UI, audit trail, fallback to existing POS menu 2.8.

**Out of scope (Phase 1)** — In-app native flow, biometric verification, e-Signature with NDID, foreign customer onboarding (passport flow planned in Phase 2).

## 3. Roles

| Role | สิทธิ์ |
|---|---|
| Customer | เปิดลิงค์ · ยืนยันตัวตน · เลือกห้อง · กรอก/แก้/ส่งคำขอ · ดูสถานะของตน |
| Sales Advisor | Generate link · ดูคำขอที่ลูกค้าส่ง · ทำผ่าน POS เมนู 2.8 (fallback) |
| CC | เหมือน Sales + ขยายอายุลิงค์ + ส่งซ้ำ |
| Reviewer | Approve/Reject · Push to REM |
| System Admin | Audit · Unlock · Revoke link |

## 4. Critical UX Change — A1 Redesign

A1 เดิมเป็นหน้า "เลือกห้อง" ตรง ๆ → เปลี่ยนเป็น **Identity Verification Gate** ก่อน

| Sub-step | Screen | Inputs | Outputs |
|---|---|---|---|
| A1.1 | Identity Entry | ID Card / Passport · Phone | สั่งส่ง OTP |
| A1.2 | OTP Challenge | OTP 6 หลัก + Ref Code | Verified session |
| A1.3 | Unit Selection | เลือก 1 ห้องจาก list ของลูกค้า | session + unit_id |

**Eligibility Rule:** ลูกค้าที่ verify ผ่านได้ ต้องมีข้อมูลในตาราง `customer` AND มีสัญญา `status IN (Completed, Active Rental)` AND `phone` ตรงกับที่ลงทะเบียนไว้ใน CRM

## 5. Identity Verification Logic

```
1. Match (id_no, phone) ใน CRM → ถ้าไม่พบ: ปฏิเสธ + ติดต่อ Sales
2. Generate OTP 6 หลัก + Ref 6 ตัวอักษร · TTL 5 นาที · เก็บ bcrypt hash ใน Redis
3. ส่ง OTP ทาง SMS gateway → คืน ref ให้ frontend
4. ลูกค้ากรอก OTP → verify hash
   - ถูก: mark session verified (TTL 30 นาที)
   - ผิด: เพิ่ม counter (TTL 30 นาที)
   - ผิด ≥ 5: lockout 30 นาที
5. โหลด units WHERE owner_id = customer_id (รวม Tenant)
```

**Rate limits:** 3 OTP / phone / hr · 5 OTP / IP / hr · token ลิงค์ใช้ได้ 1 ครั้ง · TTL 24 ชม.

## 6. State Machine

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

## 7. New Database Tables

| Table | Key columns |
|---|---|
| `pi_change_link` | id, token (unique), customer_id, generated_by, expires_at, used_at |
| `pi_change_request` | req_no, link_id, customer_id, unit_id, status, fields_changed (JSONB), attachments (JSONB), pdpa_consent_at, submitted_at |
| `otp_session` (Redis) | session_id, otp_hash, ref, attempts, expires_at |
| `pi_audit_log` (append-only) | id, ts, event_type, actor_type, actor_id, target_id, ip, user_agent, payload (JSONB) |

## 8. APIs (sketch)

```
POST   /api/v1/pi-change/link              (Sales/CC) → { url, token, expires_at }
GET    /api/v1/pi-change/link/{token}      (Customer) → validate token
POST   /api/v1/pi-change/identity/lookup   { id_no, phone } → 200 { otp_ref } | 404
POST   /api/v1/pi-change/identity/verify   { session, otp } → 200 { units } | 400
POST   /api/v1/pi-change/request           { unit_id, fields, attachments } → { req_no }
GET    /api/v1/pi-change/request/{req_no}  → status detail
POST   /api/v1/pi-change/request/{req_no}/review  (Reviewer) { decision, note }
```

## 9. Security Controls

- HTTPS + HSTS · TLS 1.3 only
- JWT (RS256) สำหรับ link token + jti blacklist หลังใช้
- OTP เก็บ bcrypt hash · ห้ามเก็บ plaintext
- WAF + rate limit (Cloudflare หรือเทียบเท่า)
- File upload: virus scan + ขนาด ≤ 5MB/ไฟล์ · ≤ 25MB total · เก็บ private S3 + signed URL
- Audit log: append-only · เก็บ ≥ 7 ปี ตาม PDPA
- Mask sensitive data ในหน้าจอ (ชื่อ ส****ี · เบอร์ 081-XXX-5678)

## 10. PDPA / Compliance

- แสดง Consent text ก่อน Submit ทุกครั้ง (Section A3)
- Privacy Policy link ที่ footer ทุกหน้า
- ลูกค้ามีสิทธิ์ขอดูประวัติคำขอของตนเอง (Self-service portal)
- เก็บลายเซ็นดิจิทัล + IP + timestamp เป็นหลักฐานการให้ Consent

## 11. Fallback Path

ลูกค้ากลุ่มไหนยังต้องใช้ POS เมนู 2.8 เดิม?

- ลูกค้าที่เบอร์ใน CRM ไม่ตรง / ไม่มีเบอร์
- ลูกค้าที่ไม่สะดวกใช้มือถือ (สูงอายุ / ไม่มี smartphone)
- ลูกค้าต่างชาติที่ใช้ passport (Phase 2)
- ลูกค้าที่ verify ไม่ผ่าน 5 ครั้ง → ระบบล็อก → ต้องไป POS

Sales/CC ยังเข้า POS เมนู 2.8 ได้เหมือนเดิม · ผลลัพธ์ส่ง REM ทางเดียวกัน

## 12. Rollout

| Phase | เดือน | Scope |
|---|---|---|
| 1 — Foundation | M+1 | Data cleansing เบอร์ + ID, สร้าง Identity Service, SMS gateway integration |
| 2 — Pilot | M+2 | 1 โครงการ (Nue Core คูคต) · เฉพาะ field Email + Phone |
| 3 — Full Launch | M+4 | ทุกโครงการ · ทุก field · LINE OA channel · Tenant flow |

## 13. Success Metrics (6 เดือนหลัง launch)

- Self-service completion rate ≥ 70%
- Time-to-REM < 4 ชั่วโมง (จากเดิม 1–3 วัน)
- Sales effort ต่อ 1 คำขอ < 5 นาที (จากเดิม ~25 นาที)
- Data error rate < 1% (จากเดิม ~6%)
- OTP first-try success ≥ 92%

## 14. Open Questions

1. ใช้ SMS gateway เจ้าไหน? (Thaibulksms, AIS Cloud, Twilio?)
2. Reviewer คือใคร? Sales Admin / CC Lead / Compliance Team?
3. PDPA Officer ต้อง sign-off design หรือไม่?
4. ต้องการ LINE Login เป็นทางเลือกของ OTP หรือไม่?
5. Token ลิงค์ — ส่งทาง email อย่างเดียว หรือ SMS ด้วย?

---

*ดูเอกสารประกอบ:*
- `A1-Identity-Verify-Mockup.html` — High-fidelity mockup 3 states
- `System-Architecture.html` — Full architecture, sequence diagram, state machine, rollout plan
