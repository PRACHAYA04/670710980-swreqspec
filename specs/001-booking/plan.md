# Plan: จองคิวตรวจสุขภาพ (Booking)

## 1. สรุปแนวทาง

ฟีเจอร์นี้ให้ผู้รับบริการที่ยืนยันตัวตนแล้วเลือกแพ็กเกจ วัน และช่วงเวลาตรวจสุขภาพ แล้วได้หมายเลขคิวในระบบ booking สำหรับการตรวจสุขภาพ โรงพยาบาลต้องป้องกันจองซ้ำในวันเดียวกัน ให้แสดงช่วงเวลาว่างแบบ real-time และจัดการกรณีช่วงเวลาเต็มและข้อความยืนยันที่ไม่สำเร็จโดยไม่ทำให้การจองเสียหาย ผู้ใช้หลักคือผู้รับบริการและทีมบริการแจ้งเตือน/เวชระเบียนที่ต้องดูข้อมูล booking และ audit log

## 2. เทคโนโลยีที่ใช้

| สิ่งที่เลือก | มาจาก | หมายเหตุ |
|---|---|---|
| MySQL | CON-TECH-01 | ใช้เป็นฐานข้อมูลหลักตามข้อกำหนดของโรงพยาบาล |
| React (Vite) | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สำหรับหน้าจอการเลือกแพ็กเกจและช่วงเวลา |
| Python FastAPI | ทีมเลือกเอง ไม่ได้มาจาก spec | ใช้สำหรับ API การจองและการคำนวณช่วงว่าง |
| Redis หรือ queue ระยะสั้น | IF-NOT-01 | ใช้สำหรับเก็บข้อความยืนยันที่ส่งไม่สำเร็จและส่งซ้ำภายใน 5 นาที |
| Audit log table | DOM-PDPA-01 | ใช้บันทึกผู้เข้าถึง เวลา และรหัสผู้รับบริการที่ใช้ภายในระบบ |

## 3. โมเดลข้อมูล

| Entity | ฟิลด์หลัก | รองรับ FR / Constraint |
|---|---|---|
| PatientProfile | HN, citizen_id_hash, verified_at | IF-IDP-01, IF-HIS-01 |
| Booking | booking_id, patient_hn, package_id, service_date, slot_id, queue_number, status, created_at, updated_at | FR-BKG-02, FR-BKG-04, FR-BKG-05 |
| SlotTemplate | slot_id, date, start_time, end_time, capacity, package_id | FR-BKG-01, FR-BKG-06 |
| BookingAuditLog | log_id, accessed_by, patient_hn, access_time, action | DOM-PDPA-01 |
| NotificationMessage | message_id, booking_id, channel, payload, status, retry_count, next_retry_at | FR-BKG-05, NFR-REL-02, IF-NOT-01 |
| PackageOption | package_id, name, valid_days, description | FR-BKG-06 |

หมายเหตุ: ระบบจะไม่เก็บเลขบัตรประชาชนในตารางการจองตาม IF-HIS-01 และจะใช้ HN เป็นรหัสผู้รับบริการภายในระบบสำหรับ audit log ตามข้อตกลงที่ระบุใน spec

## 4. API / หน้าจอ

| รายการ | รายละเอียด |
|---|---|
| GET /api/bookings/slots | แสดงช่วงเวลาว่างภายใน 30 วัน พร้อมจำนวนที่นั่งคงเหลือ ใช้รองรับ FR-BKG-01 และ FR-BKG-06 |
| GET /api/bookings/check-existing | ตรวจว่าผู้รับบริการมีคิวที่ยังไม่ได้ใช้ในวันเดียวกันหรือไม่ ใช้รองรับ FR-BKG-02 |
| POST /api/bookings/validate | Validate slot availability ก่อนยืนยันการจอง ใช้รองรับ FR-BKG-03 |
| POST /api/bookings | สร้างการจองและส่งคำขอแจ้งเตือน ใช้รองรับ FR-BKG-04 |
| POST /api/bookings/{id}/retry-notification | ส่งข้อความยืนยันซ้ำเมื่อส่งไม่สำเร็จ ใช้รองรับ FR-BKG-05 และ NFR-REL-02 |
| GET /api/bookings/{id} | แสดงหมายเลขคิวและสถานะการจอง ใช้รองรับ FR-BKG-02 และ FR-BKG-05 |
| หน้าเลือกแพ็กเกจ | เลือกแพ็กเกจและวันที่/ช่วงเวลา สำหรับผู้รับบริการ |
| หน้าแสดงผลการจอง | แสดงหมายเลขคิวและข้อความยืนยันที่ถูกสร้าง |

## 5. ตารางตรวจ Constraints

| Constraint ID | ถูกนำไปใช้ที่ไหนใน plan | สถานะ |
|---|---|---|
| CON-TECH-01 | MySQL เป็นฐานข้อมูลหลัก | ใช้แล้ว |
| DOM-PDPA-01 | BookingAuditLog และการบันทึกผู้เข้าถึง เวลา และ HN | ใช้แล้ว |
| IF-IDP-01 | Precondition และ validation ก่อนเปิดข้อมูลผู้รับบริการ | ใช้แล้ว |
| IF-HIS-01 | PatientProfile ใช้ HN จาก HIS และไม่เก็บเลขบัตรประชาชนใน Booking | ใช้แล้ว |
| IF-NOT-01 | NotificationMessage เป็น queue แบบ asynchronous และไม่บล็อกคิวหลัก | ใช้แล้ว |

## 6. แผนทดสอบจาก Acceptance Criteria

| AC ID | ชื่อ test | ทดสอบอย่างไร |
|---|---|---|
| AC-BKG-01 | test_AC_BKG_01_booking_success_and_slot_reduction | ตั้งค่าช่วงเวลา 09.00 มีที่นั่งว่าง 1 ที่ แล้วยืนยันการจอง ตรวจว่าบันทึกสำเร็จ แสดงหมายเลขคิว และที่นั่งคงเหลือเป็น 0 |
| AC-BKG-02 | test_AC_BKG_02_reject_existing_same_day_booking | สร้างคิวที่ยังไม่ได้ใช้ในวันเดียวกัน แล้วพยายามจองใหม่ ตรวจว่าระบบปฏิเสธและแสดงหมายเลขคิวเดิม |
| AC-BKG-03 | test_AC_BKG_03_offer_alternative_slots_when_full | จำลองช่วงเวลาเต็มและมีคนยืนยันก่อน แล้วตรวจว่าระบบแจ้ง “ช่วงเวลาเต็ม” และเสนอ 3 ตัวเลือกจากวันเดียวกันก่อน ถ้าไม่มีให้ใช้วันถัดไป |
| AC-BKG-04 | test_AC_BKG_04_notification_failure_does_not_rollback_booking | จำลอง SMS/LINE ไม่ตอบ ได้รับสถานะ fail ตรวจว่าการจองยังบันทึกอยู่ และมีรายการใน retry queue ภายใน 5 นาที |
| AC-BKG-05 | test_AC_BKG_05_slot_search_performance_200_users | จำลอง concurrency 200 คน ค้นหาช่วงเวลาว่าง แล้วตรวจ p95 <= 2 วินาที |
| AC-BKG-06 | test_AC_BKG_06_audit_log_recorded | เปิดดูข้อมูลการจองจากผู้เข้าถึง ตรวจว่า audit log มีผู้เข้าถึง เวลา และ HN |

## 7. ลำดับงาน

1. จัดทำ schema ฐานข้อมูลสำหรับ Booking, SlotTemplate, NotificationMessage และ AuditLog ตาม FR-BKG-01, FR-BKG-04, DOM-PDPA-01
2. สร้าง API ค้นหาช่วงเวลาว่างและจำนวนที่นั่งคงเหลือ และเชื่อมกับการคำนวณจาก package/slot ตาม FR-BKG-01 และ FR-BKG-06
3. สร้าง rule ตรวจคิวที่ยังไม่ได้ใช้ในวันเดียวกันเพื่อปฏิเสธการจองซ้ำตาม FR-BKG-02 และ AC-BKG-02
4. สร้าง flow การยืนยันและแก้ไขกรณีช่วงเวลาเต็ม โดยเสนอ 3 ตัวเลือกจากวันเดียวกันก่อน ตาม FR-BKG-03 และ AC-BKG-03
5. สร้าง flow บันทึกการจองและออกหมายเลขคิว พร้อมส่งคำขอแจ้งเตือนตาม FR-BKG-04 และ AC-BKG-01
6. สร้าง queue หรือตัวจัดการ retry สำหรับการส่งข้อความยืนยันที่ล้มเหลวตาม FR-BKG-05, NFR-REL-02 และ AC-BKG-04
7. สร้าง audit log และ validation ของผู้เข้าถึงข้อมูลตาม DOM-PDPA-01 และ AC-BKG-06
8. ทดสอบประสิทธิภาพ concurrency และ acceptance test ครบตาม AC-BKG-05 และ AC ทั้งหมด

## 8. สิ่งที่ยังไม่ทำ

- Q-02 หมายเลขคิวรีเซ็ตรายวัน หรือนับต่อเนื่อง? -> ถามเจ้าหน้าที่เวชระเบียน

ส่วนนี้ยังไม่สร้างจนกว่าจะได้คำตอบ เพราะผลลัพธ์จะมีผลต่อการออกแบบหมายเลขคิวและการทดสอบการแสดง queue number ของแต่ละวัน
