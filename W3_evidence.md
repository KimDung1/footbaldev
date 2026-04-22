# W3 Evidence — Data Access Log

## Tổng quan

Tài liệu này mô tả 3 access pattern chính của hệ thống đặt sân bóng, dựa trên database PostgreSQL đã thiết kế.

Bao gồm:

* Tìm sân
* Đặt sân
* Xem lịch sân

---

# 1. Access Pattern: Tìm sân

## Use Case

Người dùng tìm sân theo:

* Thành phố / quận (locations)
* Ngày
* Khung giờ

---

## Query Pattern

```sql
SELECT f.id, f.name, f.price_per_hour, l.address_text
FROM fields f
JOIN locations l ON f.location_id = l.id
JOIN subfields sf ON sf.field_id = f.id
JOIN timeslots t ON t.sub_field_id = sf.id
WHERE l.city = 'Da Nang'
  AND t.date = '2026-04-25'
  AND t.start_time >= '18:00'
  AND t.end_time <= '20:00'
  AND t.status = 'available';
```

---

## Giải thích theo database

* `fields` → thông tin sân
* `locations` → địa chỉ sân
* `subfields` → sân nhỏ (5vs5, 7vs7)
* `timeslots` → khung giờ

Query này join qua 4 bảng để tìm sân còn trống theo giờ

---

## Tối ưu

### Index nên có

```sql
CREATE INDEX idx_location_city ON locations(city);
CREATE INDEX idx_timeslots_search 
ON timeslots(sub_field_id, date, start_time, end_time, status);
```

---

## Ghi chú

* Đây là query đọc nhiều (read-heavy)
* Có thể cache bằng Redis (ElastiCache)
* Có thể thêm filter:

  * giá (price_per_hour)
  * loại sân (field_type)

---

# 2. Access Pattern: Đặt sân

## Use Case

Người dùng chọn timeslot và tạo booking

---

## Query Pattern (Transaction)

```sql
BEGIN;

-- 1. Lock timeslot để tránh double booking
SELECT *
FROM timeslots
WHERE id = 'timeslot_id'
FOR UPDATE;

-- 2. Update trạng thái timeslot
UPDATE timeslots
SET status = 'booked',
    booking_id = 'booking_id'
WHERE id = 'timeslot_id'
  AND status = 'available';

-- 3. Tạo booking
INSERT INTO bookings (
  id,
  booking_date,
  total_price,
  deposit_amount,
  remaining_amount,
  user_id,
  created_at,
  updated_at
)
VALUES (
  'booking_id',
  NOW(),
  500000,
  100000,
  400000,
  'user_id',
  NOW(),
  NOW()
);

COMMIT;
```

---

## Giải thích theo database

* `timeslots.booking_id` → liên kết tới booking
* `bookings` → chứa thông tin tổng giao dịch
* `payments` → xử lý thanh toán riêng

---

## Tối ưu

### Concurrency control

* Dùng `SELECT ... FOR UPDATE`
* Dùng transaction để đảm bảo atomic

---

## Ghi chú

* Đây là operation quan trọng nhất (write-critical)
* Phải đảm bảo:
   không bị double booking

---

# 3. Access Pattern: Xem lịch sân

## Use Case

Người dùng xem tất cả khung giờ của một sân trong ngày

---

## Query Pattern

```sql
SELECT t.start_time, t.end_time, t.status
FROM timeslots t
JOIN subfields sf ON t.sub_field_id = sf.id
WHERE sf.field_id = 'field_id'
  AND t.date = '2026-04-25'
ORDER BY t.start_time;
```

---

## Giải thích theo database

* `subfields` → gom các sân con của 1 sân lớn
* `timeslots` → hiển thị trạng thái:

  * available
  * booked
  * maintenance

---

## Tối ưu

### Index nên có

```sql
CREATE INDEX idx_timeslots_schedule 
ON timeslots(sub_field_id, date);
```

---

## Ghi chú

* Đây là query đọc nhiều (high-frequency read)
* Có thể cache theo:

  * key: field_id + date
* Có thể preload trước để tăng tốc

---

# Kết luận

3 access pattern chính:

1. Tìm sân → nhiều join, read-heavy
2. Đặt sân → transaction, chống race condition
3. Xem lịch → đọc nhiều, cần tối ưu index + cache

---

