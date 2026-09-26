# 🍊 CẨM NANG SINH TỒN & ĐÀO TẠO PHỎNG VẤN MySQL

> **Tác giả giả lập:** Senior DBA / Giảng viên IT / Technical Interviewer  
> **Đối tượng:** Sinh viên năm cuối, Fresher/Junior Backend Developer (Java/Spring Boot)  
> **Database xuyên suốt:** Hệ thống Quản lý Cửa hàng Trái cây (Fruit Store)  
> **Triết lý:** *"Hiểu bản chất, không học vẹt cú pháp. Viết được query đúng chưa đủ — phải biết TẠI SAO nó đúng."*

---

# PHẦN 0: KHỞI TẠO PHÒNG THÍ NGHIỆM (LAB SETUP)

> ⚠️ **Lời dặn của thầy:** Copy TOÀN BỘ block SQL bên dưới, paste vào MySQL Workbench hoặc DBeaver, bấm Execute. Nếu chạy lỗi ở bất kỳ dòng nào — dừng lại, đọc lỗi, sửa, chạy lại. Đây là kỹ năng debug đầu tiên em cần có.

## 0.1. Tạo Database

```sql
-- ============================================================
-- FRUIT STORE DATABASE - PHÒNG THÍ NGHIỆM MYSQL
-- ============================================================
-- Xóa database cũ nếu có (CẨN THẬN khi dùng trên production!)
DROP DATABASE IF EXISTS fruit_store;

CREATE DATABASE fruit_store
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;

USE fruit_store;
```

> 💡 **Tại sao `utf8mb4` chứ không phải `utf8`?**  
> `utf8` của MySQL thực chất chỉ hỗ trợ tối đa 3 byte — không đủ để lưu emoji 🍉 hay một số ký tự đặc biệt. `utf8mb4` mới là UTF-8 chuẩn (tối đa 4 byte). **Trong doanh nghiệp, luôn dùng `utf8mb4`.**

## 0.2. Tạo 5 bảng với đầy đủ ràng buộc

```sql
-- ============================================================
-- BẢNG 1: CATEGORY (Danh mục trái cây)
-- ============================================================
CREATE TABLE category (
    category_id   INT           AUTO_INCREMENT,
    category_name VARCHAR(100)  NOT NULL,
    description   VARCHAR(500)  DEFAULT NULL,
    created_at    TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (category_id),
    UNIQUE KEY uk_category_name (category_name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================
-- BẢNG 2: FRUIT (Trái cây)
-- ============================================================
CREATE TABLE fruit (
    fruit_id      INT           AUTO_INCREMENT,
    fruit_name    VARCHAR(150)  NOT NULL,
    category_id   INT           NOT NULL,
    price         DECIMAL(12,2) NOT NULL,
    stock_qty     INT           NOT NULL DEFAULT 0,
    unit          VARCHAR(20)   NOT NULL DEFAULT 'kg',
    origin        VARCHAR(100)  DEFAULT NULL,
    is_seasonal   TINYINT(1)    NOT NULL DEFAULT 0,
    created_at    TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at    TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    PRIMARY KEY (fruit_id),
    CONSTRAINT fk_fruit_category
        FOREIGN KEY (category_id) REFERENCES category(category_id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT,
    INDEX idx_fruit_category (category_id),
    INDEX idx_fruit_price (price)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================
-- BẢNG 3: CUSTOMER (Khách hàng)
-- ============================================================
CREATE TABLE customer (
    customer_id   INT           AUTO_INCREMENT,
    full_name     VARCHAR(150)  NOT NULL,
    email         VARCHAR(200)  NOT NULL,
    phone         CHAR(10)      DEFAULT NULL,
    address       VARCHAR(500)  DEFAULT NULL,
    registered_at TIMESTAMP     NOT NULL DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (customer_id),
    UNIQUE KEY uk_customer_email (email),
    INDEX idx_customer_phone (phone)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================
-- BẢNG 4: `ORDER` (Đơn hàng)
-- Lưu ý: ORDER là từ khóa MySQL, phải đặt trong backtick ``
-- ============================================================
CREATE TABLE `order` (
    order_id      INT           AUTO_INCREMENT,
    customer_id   INT           NOT NULL,
    order_date    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    total_amount  DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    status        ENUM('PENDING','CONFIRMED','SHIPPING','DELIVERED','CANCELLED')
                                NOT NULL DEFAULT 'PENDING',
    note          VARCHAR(1000) DEFAULT NULL,

    PRIMARY KEY (order_id),
    CONSTRAINT fk_order_customer
        FOREIGN KEY (customer_id) REFERENCES customer(customer_id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT,
    INDEX idx_order_customer (customer_id),
    INDEX idx_order_date (order_date),
    INDEX idx_order_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ============================================================
-- BẢNG 5: ORDER_DETAIL (Chi tiết đơn hàng)
-- ============================================================
CREATE TABLE order_detail (
    detail_id     INT           AUTO_INCREMENT,
    order_id      INT           NOT NULL,
    fruit_id      INT           NOT NULL,
    quantity      INT           NOT NULL,
    unit_price    DECIMAL(12,2) NOT NULL,
    line_total    DECIMAL(15,2) GENERATED ALWAYS AS (quantity * unit_price) STORED,

    PRIMARY KEY (detail_id),
    CONSTRAINT fk_detail_order
        FOREIGN KEY (order_id) REFERENCES `order`(order_id)
        ON UPDATE CASCADE
        ON DELETE CASCADE,
    CONSTRAINT fk_detail_fruit
        FOREIGN KEY (fruit_id) REFERENCES fruit(fruit_id)
        ON UPDATE CASCADE
        ON DELETE RESTRICT,
    UNIQUE KEY uk_order_fruit (order_id, fruit_id),
    INDEX idx_detail_fruit (fruit_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

> 🔍 **Điểm đáng chú ý trong thiết kế:**
> | Quyết định | Lý do |
> |---|---|
> | `DECIMAL(12,2)` cho giá | Không bao giờ dùng `FLOAT/DOUBLE` cho tiền — sẽ bị sai số thập phân |
> | `CHAR(10)` cho phone | Số điện thoại VN luôn 10 ký tự — `CHAR` hiệu quả hơn `VARCHAR` khi độ dài cố định |
> | `ENUM` cho status | Giới hạn giá trị hợp lệ ngay ở tầng database, không phụ thuộc hoàn toàn vào code |
> | `ON DELETE CASCADE` ở order_detail | Xóa đơn hàng → tự động xóa chi tiết. Hợp lý vì chi tiết không tồn tại mà thiếu đơn |
> | `ON DELETE RESTRICT` ở fruit | Không cho xóa trái cây nếu đã có đơn đặt hàng. Bảo vệ toàn vẹn dữ liệu |
> | `GENERATED ALWAYS AS` cho line_total | Cột tính toán tự động, luôn chính xác, không cần code xử lý |

## 0.3. Chèn dữ liệu mẫu

```sql
-- ============================================================
-- DỮ LIỆU MẪU - HÀNH TINH TRÁI CÂY
-- ============================================================

-- 1) Danh mục
INSERT INTO category (category_name, description) VALUES
('Trái cây nhiệt đới',  'Các loại trái cây đặc trưng vùng nhiệt đới Đông Nam Á'),
('Trái cây ôn đới',     'Các loại trái cây nhập khẩu từ vùng khí hậu ôn đới'),
('Trái cây họ cam quýt', 'Cam, quýt, bưởi và các loại citrus'),
('Dưa các loại',        'Dưa hấu, dưa lưới, dưa leo trái cây');

-- 2) Trái cây (10 loại)
INSERT INTO fruit (fruit_name, category_id, price, stock_qty, unit, origin, is_seasonal) VALUES
('Sầu riêng Ri6',       1, 180000.00, 25,  'kg',   'Tiền Giang',   1),
('Xoài cát Hòa Lộc',   1, 85000.00,  60,  'kg',   'Đồng Tháp',    1),
('Măng cụt',            1, 65000.00,  40,  'kg',   'Bến Tre',       1),
('Táo Fuji Nhật',       2, 220000.00, 30,  'kg',   'Nhật Bản',      0),
('Nho đỏ Úc',           2, 150000.00, 35,  'kg',   'Úc',            0),
('Cherry Mỹ',           2, 450000.00, 15,  'kg',   'Mỹ',            1),
('Cam sành',            3, 35000.00,  100, 'kg',   'Vĩnh Long',     0),
('Bưởi da xanh',        3, 45000.00,  50,  'trái', 'Bến Tre',       0),
('Dưa hấu',             4, 20000.00,  80,  'kg',   'Long An',       0),
('Dưa lưới Nhật',       4, 120000.00, 20,  'trái', 'Đà Lạt',        1);

-- 3) Khách hàng (7 người)
INSERT INTO customer (full_name, email, phone, address) VALUES
('Nguyễn Văn An',    'an.nguyen@gmail.com',    '0901234567', '123 Lê Lợi, Q.1, TP.HCM'),
('Trần Thị Bình',   'binh.tran@yahoo.com',    '0912345678', '456 Nguyễn Huệ, Q.1, TP.HCM'),
('Lê Hoàng Cường',  'cuong.le@outlook.com',   '0923456789', '789 CMT8, Q.3, TP.HCM'),
('Phạm Minh Dũng',  'dung.pham@gmail.com',    '0934567890', '12 Trần Phú, Q.5, TP.HCM'),
('Hoàng Thị Em',    'em.hoang@gmail.com',     '0945678901', '34 Pasteur, Q.3, TP.HCM'),
('Võ Văn Phúc',     'phuc.vo@company.vn',     '0956789012', '56 Hai Bà Trưng, Q.1, TP.HCM'),
('Đặng Nhật Quỳnh', 'quynh.dang@startup.io',  '0967890123', '78 Võ Văn Tần, Q.3, TP.HCM');

-- 4) Đơn hàng (8 đơn)
INSERT INTO `order` (customer_id, order_date, total_amount, status, note) VALUES
(1, '2024-08-01 09:30:00', 545000.00,  'DELIVERED',  NULL),
(2, '2024-08-03 14:15:00', 780000.00,  'DELIVERED',  'Giao trước 17h'),
(1, '2024-08-10 11:00:00', 350000.00,  'SHIPPING',   NULL),
(3, '2024-08-15 08:45:00', 1350000.00, 'CONFIRMED',  'Đơn công ty, cần hóa đơn VAT'),
(4, '2024-08-20 16:30:00', 200000.00,  'PENDING',    NULL),
(5, '2024-09-01 10:00:00', 900000.00,  'DELIVERED',  NULL),
(6, '2024-09-05 13:20:00', 0.00,       'CANCELLED',  'Khách đổi ý'),
(7, '2024-09-10 17:45:00', 670000.00,  'SHIPPING',   'Giao ngoài giờ hành chính');

-- 5) Chi tiết đơn hàng (15 dòng)
INSERT INTO order_detail (order_id, fruit_id, quantity, unit_price) VALUES
-- Đơn 1: Anh An mua sầu riêng + xoài
(1, 1, 2, 180000.00),   -- 2kg sầu riêng = 360,000
(1, 2, 1, 85000.00),    -- 1kg xoài = 85,000
(1, 9, 5, 20000.00),    -- 5kg dưa hấu = 100,000
-- Đơn 2: Chị Bình mua cherry + táo
(2, 6, 1, 450000.00),   -- 1kg cherry = 450,000
(2, 4, 1, 220000.00),   -- 1kg táo Fuji = 220,000
(2, 7, 3, 35000.00),    -- 3kg cam sành = 105,000 (tặng chồng 😄)
-- Đơn 3: Anh An quay lại mua thêm
(3, 5, 2, 150000.00),   -- 2kg nho đỏ = 300,000
(3, 8, 1, 45000.00),    -- 1 trái bưởi = 45,000
-- Đơn 4: Anh Cường mua sỉ cho công ty
(4, 1, 5, 180000.00),   -- 5kg sầu riêng = 900,000
(4, 6, 1, 450000.00),   -- 1kg cherry = 450,000
-- Đơn 5: Anh Dũng mua dưa
(5, 9, 10, 20000.00),   -- 10kg dưa hấu = 200,000
-- Đơn 6: Chị Em mua combo
(6, 1, 3, 180000.00),   -- 3kg sầu riêng = 540,000
(6, 3, 2, 65000.00),    -- 2kg măng cụt = 130,000
(6, 10, 2, 120000.00),  -- 2 trái dưa lưới = 240,000 (tổng = 910,000 ≈ 900,000 sau giảm giá)
-- Đơn 8: Chị Quỳnh
(8, 2, 3, 85000.00),    -- 3kg xoài = 255,000
(8, 3, 5, 65000.00);    -- 5kg măng cụt = 325,000 (tổng = 580,000 ≈ 670,000 tính phí ship)
```

> 🎯 **Kiểm tra nhanh:** Sau khi chạy xong, thực thi lệnh sau để đảm bảo dữ liệu đã vào:
> ```sql
> SELECT 'category' AS tbl, COUNT(*) AS rows_count FROM category
> UNION ALL SELECT 'fruit', COUNT(*) FROM fruit
> UNION ALL SELECT 'customer', COUNT(*) FROM customer
> UNION ALL SELECT 'order', COUNT(*) FROM `order`
> UNION ALL SELECT 'order_detail', COUNT(*) FROM order_detail;
> ```
> Kết quả kỳ vọng: `category=4`, `fruit=10`, `customer=7`, `order=8`, `order_detail=16`.

---

# PHẦN 1: TƯ DUY THIẾT KẾ CƠ SỞ DỮ LIỆU & KIỂU DỮ LIỆU

> *"Một cơ sở dữ liệu thiết kế tốt là cơ sở dữ liệu mà em không bao giờ cần phải đập đi làm lại."*

## 1.1. Ba dạng chuẩn hóa (Normalization): Không phải lý thuyết — là kỹ năng sống còn

### 🔴 1NF (First Normal Form) — "Mỗi ô chỉ chứa MỘT giá trị"

**Nguyên tắc:** Mỗi cột chỉ chứa giá trị nguyên tử (atomic), không có danh sách, không có nhóm lặp.

**❌ Vi phạm 1NF — Bảng thiết kế sai:**

| order_id | customer_name | fruits_bought |
|---|---|---|
| 1 | Nguyễn Văn An | Sầu riêng Ri6, Xoài cát Hòa Lộc, Dưa hấu |
| 2 | Trần Thị Bình | Cherry Mỹ, Táo Fuji Nhật |

**Hậu quả thảm khốc:**
- Muốn tìm "ai mua Sầu riêng?" → phải dùng `LIKE '%Sầu riêng%'` → **chậm kinh khủng**, không dùng được Index.
- Muốn đếm "bán được bao nhiêu loại trái cây?" → phải tách chuỗi bằng code → **lỗi tiềm ẩn**.
- Muốn xóa "Dưa hấu" khỏi đơn 1 → phải xử lý chuỗi → **cực kỳ dễ sai**.

**✅ Tuân thủ 1NF — Tách ra bảng `order_detail`:**

Đây chính xác là thiết kế chúng ta đã làm ở Phần 0. Mỗi dòng trong `order_detail` chỉ chứa MỘT loại trái cây.

```sql
-- Truy vấn giờ trở nên đơn giản và NHANH:
SELECT c.full_name, f.fruit_name
FROM order_detail od
JOIN `order` o ON o.order_id = od.order_id
JOIN customer c ON c.customer_id = o.customer_id
JOIN fruit f ON f.fruit_id = od.fruit_id
WHERE f.fruit_name = 'Sầu riêng Ri6';
```

### 🟡 2NF (Second Normal Form) — "Mọi cột phải phụ thuộc vào TOÀN BỘ khóa chính"

**Tiền đề:** Bảng đã đạt 1NF + khóa chính là khóa tổ hợp (composite key).

**❌ Vi phạm 2NF — Ví dụ sai với Fruit Store:**

Giả sử ta thiết kế bảng `order_detail` nhưng **nhúng luôn thông tin trái cây** vào:

| order_id | fruit_id | quantity | unit_price | fruit_name | origin |
|---|---|---|---|---|---|
| 1 | 1 | 2 | 180000 | Sầu riêng Ri6 | Tiền Giang |
| 1 | 2 | 1 | 85000 | Xoài cát Hòa Lộc | Đồng Tháp |

Khóa chính: `(order_id, fruit_id)`.  
Vấn đề: `fruit_name` và `origin` chỉ phụ thuộc vào `fruit_id` — **KHÔNG** phụ thuộc vào `order_id`. Đây gọi là **phụ thuộc hàm bộ phận (partial dependency)**.

**Hậu quả:**
- **Update Anomaly:** Đổi tên "Sầu riêng Ri6" → phải sửa ở TẤT CẢ các dòng order_detail có fruit_id = 1. Sót một dòng = dữ liệu mâu thuẫn.
- **Insertion Anomaly:** Muốn thêm trái cây mới nhưng chưa có ai đặt hàng → không insert được vì thiếu `order_id`.

**✅ Tuân thủ 2NF:** Tách `fruit_name`, `origin` sang bảng riêng `fruit` (đúng như thiết kế của chúng ta).

### 🟢 3NF (Third Normal Form) — "Không có phụ thuộc bắc cầu"

**Tiền đề:** Đã đạt 2NF.

**❌ Vi phạm 3NF — Ví dụ sai:**

Giả sử bảng `fruit` có thêm cột `category_name`:

| fruit_id | fruit_name | category_id | category_name | price |
|---|---|---|---|---|
| 1 | Sầu riêng Ri6 | 1 | Trái cây nhiệt đới | 180000 |
| 7 | Cam sành | 3 | Trái cây họ cam quýt | 35000 |

Chuỗi phụ thuộc: `fruit_id` → `category_id` → `category_name`.  
`category_name` phụ thuộc vào `category_id` (không phải trực tiếp vào khóa chính `fruit_id`). Đây là **phụ thuộc bắc cầu (transitive dependency)**.

**Hậu quả:** Đổi tên danh mục "Trái cây nhiệt đới" → phải sửa ở MỌI dòng fruit thuộc danh mục đó.

**✅ Tuân thủ 3NF:** Tách `category_name` sang bảng `category`, bảng `fruit` chỉ giữ `category_id` làm Foreign Key. **Đây chính xác là thiết kế Phần 0.**

> 📌 **Quy tắc vàng nhớ đời:**  
> *"Mỗi cột phải phụ thuộc vào Khóa (1NF), toàn bộ Khóa (2NF), và không gì ngoài Khóa (3NF)."*  
> — Câu nói kinh điển của Bill Kent.

---

## 1.2. Kỹ thuật chọn kiểu dữ liệu — Quyết định nhỏ, hậu quả lớn

### VARCHAR vs CHAR — Khi nào dùng gì?

| Tiêu chí | CHAR(n) | VARCHAR(n) |
|---|---|---|
| **Bản chất lưu trữ** | Luôn chiếm đúng `n` byte, đệm khoảng trắng nếu thiếu | Chiếm `độ dài thực + 1-2 byte` ghi length prefix |
| **Hiệu năng** | Nhanh hơn khi tất cả giá trị có độ dài gần bằng nhau | Tốt hơn khi độ dài biến thiên lớn |
| **Ví dụ Fruit Store** | `phone CHAR(10)` — SĐT VN luôn 10 số | `fruit_name VARCHAR(150)` — "Dưa hấu" ≠ "Xoài cát Hòa Lộc" |

```sql
-- Minh họa: So sánh dung lượng thực tế
-- CHAR(10) lưu "0901234567" → chiếm 10 byte ✓
-- CHAR(10) lưu "abc"        → chiếm 10 byte (đệm 7 khoảng trắng) ✗ lãng phí
-- VARCHAR(10) lưu "abc"     → chiếm 3 + 1 = 4 byte ✓ tiết kiệm
```

> 🎯 **Nguyên tắc thực tế trong doanh nghiệp:**  
> — Nếu giá trị có độ dài **cố định** (mã bưu điện, mã tỉnh, SĐT) → dùng `CHAR`.  
> — Mọi trường hợp khác → dùng `VARCHAR`.  
> — **Đừng bao giờ** set VARCHAR(10000) "cho chắc" — nó ảnh hưởng đến bộ nhớ tạm khi MySQL sort.

### DECIMAL — Tại sao lưu tiền PHẢI dùng DECIMAL?

```sql
-- THÍ NGHIỆM: Chứng minh FLOAT bị lỗi sai số
SELECT CAST(0.1 + 0.2 AS DECIMAL(10,2)) AS decimal_result,
       0.1e0 + 0.2e0                     AS float_result;
-- Kết quả:
-- decimal_result = 0.30    ✅ Chính xác
-- float_result   = 0.30000000000000004  ❌ SAI SỐ!
```

**Hậu quả thực tế:** Giả sử cửa hàng trái cây bán 1 triệu đơn, mỗi đơn sai 0.01 VNĐ → sai lệch 10,000 VNĐ. Với ngoại tệ (USD), hàng triệu giao dịch → sai lệch hàng trăm/ngàn đô. **Kiểm toán sẽ không bao giờ chấp nhận.**

```sql
-- CÁCH ĐÚNG trong Fruit Store:
-- price         DECIMAL(12,2)  → tối đa 9,999,999,999.99 VNĐ
-- total_amount   DECIMAL(15,2)  → tối đa 9,999,999,999,999.99 VNĐ
```

| Kiểu | Khi nào dùng | Khi nào KHÔNG dùng |
|---|---|---|
| `DECIMAL(p,s)` | Tiền, thuế, chiết khấu, tỷ lệ lãi suất | Tính toán khoa học cần hiệu năng cực cao |
| `FLOAT/DOUBLE` | Tọa độ GPS, dữ liệu cảm biến IoT | **BẤT KỲ** khi nào liên quan đến tiền |

### DATETIME vs TIMESTAMP — Cuộc chiến thời gian

| Tiêu chí | DATETIME | TIMESTAMP |
|---|---|---|
| **Phạm vi** | `1000-01-01` → `9999-12-31` | `1970-01-01 00:00:01` → `2038-01-19 03:14:07` |
| **Dung lượng** | 8 byte | 4 byte |
| **Timezone** | Lưu **nguyên giá trị**, không chuyển đổi | Lưu theo **UTC**, tự chuyển đổi khi đọc dựa vào timezone session |
| **Default** | Không có auto-update | Hỗ trợ `DEFAULT CURRENT_TIMESTAMP` và `ON UPDATE CURRENT_TIMESTAMP` |

```sql
-- THÍ NGHIỆM: Chứng minh sự khác biệt timezone
SET time_zone = '+07:00';  -- Múi giờ Việt Nam
INSERT INTO fruit (fruit_name, category_id, price, stock_qty)
VALUES ('Test Timezone', 1, 10000.00, 1);

-- Xem created_at (TIMESTAMP) → hiển thị giờ VN
SELECT fruit_name, created_at FROM fruit WHERE fruit_name = 'Test Timezone';

SET time_zone = '+00:00';  -- Chuyển sang UTC
-- Xem lại → created_at tự động trừ 7 tiếng!
SELECT fruit_name, created_at FROM fruit WHERE fruit_name = 'Test Timezone';

-- Dọn dẹp
DELETE FROM fruit WHERE fruit_name = 'Test Timezone';
SET time_zone = '+07:00';
```

> 🏢 **Quyết định trong doanh nghiệp:**
> - Cột `created_at`, `updated_at` → dùng **TIMESTAMP** (tự động chuyển timezone, nhẹ hơn).
> - Cột `order_date`, `birth_date`, ngày sự kiện → dùng **DATETIME** (không muốn timezone can thiệp, cần lưu chính xác giờ local).
> - **Vấn đề năm 2038:** TIMESTAMP sẽ hết phạm vi vào năm 2038. MySQL 8.0.28+ đã xử lý bằng cách mở rộng internal storage, nhưng đây là câu hỏi phỏng vấn kinh điển!

---

## 1.3. 🎤 GÓC PHỎNG VẤN — PHẦN 1

### Câu 1: "Giải thích sự khác nhau giữa 1NF, 2NF và 3NF. Cho ví dụ vi phạm từng dạng."

> **Câu trả lời sắc sảo:**
>
> Ba dạng chuẩn hóa là quy trình loại bỏ dần sự dư thừa dữ liệu:
>
> - **1NF** yêu cầu mỗi ô chỉ chứa giá trị nguyên tử. Ví dụ: nếu cột `fruits_bought` chứa "Sầu riêng, Xoài, Dưa hấu" trong một ô → vi phạm 1NF. Phải tách ra bảng riêng, mỗi dòng một loại quả.
>
> - **2NF** yêu cầu mọi cột non-key phải phụ thuộc vào **toàn bộ** khóa chính. Nếu bảng `order_detail` có composite key `(order_id, fruit_id)` nhưng cột `fruit_name` chỉ phụ thuộc vào `fruit_id` → vi phạm 2NF vì đó là phụ thuộc bộ phận. Phải tách `fruit_name` sang bảng `fruit`.
>
> - **3NF** loại bỏ phụ thuộc bắc cầu. Nếu bảng `fruit` chứa cả `category_id` lẫn `category_name`, thì `category_name` phụ thuộc bắc cầu qua `category_id`. Phải tách sang bảng `category`.
>
> Nếu không chuẩn hóa, sẽ gặp 3 loại anomaly: **Update** (cập nhật thiếu sót), **Insertion** (không thêm được dữ liệu), **Deletion** (xóa mất dữ liệu liên quan).

### Câu 2: "Tại sao không dùng FLOAT để lưu giá tiền trong cột `price`?"

> **Câu trả lời sắc sảo:**
>
> FLOAT và DOUBLE lưu trữ số thập phân theo chuẩn IEEE 754 (dạng nhị phân floating-point), nên **không thể biểu diễn chính xác** một số giá trị thập phân. Ví dụ: `0.1 + 0.2` sẽ ra `0.30000000000000004` thay vì `0.30`.
>
> Trong hệ thống Fruit Store, nếu dùng FLOAT cho `price`:
> - Mỗi đơn hàng có thể sai lệch vài đồng.
> - Với hàng triệu giao dịch, sai số tích lũy lên hàng chục/trăm nghìn đồng.
> - Kiểm toán tài chính sẽ phát hiện bất khớp giữa sổ sách và database.
>
> **DECIMAL** lưu trữ theo exact arithmetic (số thập phân chính xác), không có sai số. Trong cửa hàng trái cây, tôi dùng `DECIMAL(12,2)` cho giá trái cây và `DECIMAL(15,2)` cho tổng đơn hàng.

### Câu 3: "DATETIME và TIMESTAMP khác nhau thế nào? Khi nào dùng cái nào?"

> **Câu trả lời sắc sảo:**
>
> Có 3 khác biệt cốt lõi:
>
> 1. **Timezone:** TIMESTAMP lưu nội bộ theo UTC, khi đọc sẽ tự chuyển theo timezone của session. DATETIME lưu nguyên giá trị, không chuyển đổi.
>
> 2. **Phạm vi:** TIMESTAMP chỉ lưu được từ 1970 → 2038 (4 byte). DATETIME lưu từ 1000 → 9999 (8 byte).
>
> 3. **Auto-update:** TIMESTAMP hỗ trợ `DEFAULT CURRENT_TIMESTAMP` và `ON UPDATE CURRENT_TIMESTAMP` trực tiếp (DATETIME cũng hỗ trợ từ MySQL 5.6.5+, nhưng TIMESTAMP là lựa chọn tự nhiên hơn cho audit columns).
>
> **Quy tắc thực tế:** Trong Fruit Store, tôi dùng TIMESTAMP cho `created_at` và `updated_at` vì chúng là cột audit, cần tự động cập nhật và nhẹ hơn 50% dung lượng. Tôi dùng DATETIME cho `order_date` vì đây là thời điểm nghiệp vụ cần lưu chính xác theo giờ local, không muốn timezone can thiệp khi hệ thống mở rộng ra nhiều vùng.

---

# PHẦN 2: KỸ NĂNG TRUY VẤN XUYÊN BẢNG & XỬ LÝ DỮ LIỆU LỚN

> *"SELECT * FROM table — ai cũng viết được. Nhưng viết một query JOIN 5 bảng, chạy trong 50ms trên 10 triệu dòng — đó mới là lập trình viên."*

## 2.1. Giải phẫu các loại JOIN

### Bản chất hình học

Hãy tưởng tượng 2 bảng là 2 hình tròn trong biểu đồ Venn:

| Loại JOIN | Kết quả trả về | Ẩn dụ |
|---|---|---|
| **INNER JOIN** | Chỉ lấy phần **giao** — dòng khớp ở CẢ HAI bảng | "Chỉ lấy trái cây **đã từng được đặt hàng**" |
| **LEFT JOIN** | Toàn bộ bảng **trái** + phần giao (bảng phải NULL nếu không khớp) | "Lấy TẤT CẢ trái cây, kể cả loại **chưa ai mua**" |
| **RIGHT JOIN** | Toàn bộ bảng **phải** + phần giao (bảng trái NULL nếu không khớp) | Ít dùng — thường viết lại bằng LEFT JOIN cho dễ đọc |

### Minh họa bằng Fruit Store

```sql
-- ============================================================
-- INNER JOIN: Chỉ lấy trái cây ĐÃ từng có trong đơn hàng
-- ============================================================
SELECT DISTINCT f.fruit_name, f.price
FROM fruit f
INNER JOIN order_detail od ON f.fruit_id = od.fruit_id;
-- Kết quả: Chỉ trái cây xuất hiện trong order_detail
-- "Táo Fuji Nhật" có vì đơn 2 mua. Nhưng trái nào CHƯA ai mua?

-- ============================================================
-- LEFT JOIN: Lấy TẤT CẢ trái cây, kể cả loại chưa ai đặt
-- ============================================================
SELECT f.fruit_name, f.price, od.detail_id
FROM fruit f
LEFT JOIN order_detail od ON f.fruit_id = od.fruit_id
WHERE od.detail_id IS NULL;
-- Kết quả: Những trái cây "ế" — tồn kho nhưng chưa có đơn nào

-- ============================================================
-- Giải thích cho người mới:
-- LEFT JOIN luôn giữ TẤT CẢ dòng bên trái (fruit).
-- Nếu không tìm thấy dòng khớp bên phải (order_detail) → cột phải = NULL.
-- Thêm WHERE od.detail_id IS NULL → lọc ra "chỉ trái cây chưa ai mua".
-- ============================================================
```

### 🔥 Query kết hợp 5 bảng — Báo cáo doanh thu chi tiết

```sql
-- ============================================================
-- BÁO CÁO: Chi tiết mỗi giao dịch thành công
-- Kết hợp: customer → order → order_detail → fruit → category
-- ============================================================
SELECT
    c.full_name                          AS ten_khach_hang,
    o.order_id                           AS ma_don,
    o.order_date                         AS ngay_dat,
    o.status                             AS trang_thai,
    cat.category_name                    AS danh_muc,
    f.fruit_name                         AS trai_cay,
    f.origin                             AS xuat_xu,
    od.quantity                          AS so_luong,
    od.unit_price                        AS don_gia,
    od.line_total                        AS thanh_tien
FROM customer c
    INNER JOIN `order` o        ON c.customer_id = o.customer_id
    INNER JOIN order_detail od  ON o.order_id    = od.order_id
    INNER JOIN fruit f          ON od.fruit_id   = f.fruit_id
    INNER JOIN category cat     ON f.category_id = cat.category_id
WHERE o.status != 'CANCELLED'
ORDER BY o.order_date DESC, od.line_total DESC;
```

> 💡 **Mẹo viết query nhiều JOIN:**
> 1. **Bắt đầu từ bảng trung tâm** — ở đây là `order` (kết nối customer ở trên, order_detail ở dưới).
> 2. **Đặt alias ngắn gọn** — `c`, `o`, `od`, `f`, `cat` — dễ đọc, dễ gõ.
> 3. **Viết mỗi JOIN trên một dòng** — dễ debug, dễ comment từng JOIN khi test.
> 4. **JOIN theo thứ tự logic** — đi từ customer → đơn hàng → chi tiết → trái cây → danh mục.

---

## 2.2. GROUP BY và HAVING — Gom nhóm và lọc kết quả tổng hợp

### Bài toán: "Tìm Top 3 loại trái cây mang lại doanh thu cao nhất"

```sql
-- ============================================================
-- TOP 3 TRÁI CÂY DOANH THU CAO NHẤT
-- (Chỉ tính đơn hàng không bị hủy)
-- ============================================================
SELECT
    f.fruit_name                         AS trai_cay,
    cat.category_name                    AS danh_muc,
    SUM(od.quantity)                     AS tong_so_luong_ban,
    SUM(od.line_total)                   AS tong_doanh_thu,
    COUNT(DISTINCT od.order_id)          AS so_don_hang
FROM order_detail od
    INNER JOIN `order` o    ON od.order_id    = o.order_id
    INNER JOIN fruit f      ON od.fruit_id    = f.fruit_id
    INNER JOIN category cat ON f.category_id  = cat.category_id
WHERE o.status != 'CANCELLED'
GROUP BY f.fruit_id, f.fruit_name, cat.category_name
ORDER BY tong_doanh_thu DESC
LIMIT 3;
```

**Giải thích từng dòng:**

| Dòng | Vai trò |
|---|---|
| `SUM(od.quantity)` | Tổng số lượng đã bán qua mọi đơn hàng |
| `SUM(od.line_total)` | Tổng doanh thu (đã nhân quantity × unit_price) |
| `COUNT(DISTINCT od.order_id)` | Đếm số đơn hàng khác nhau chứa trái cây này |
| `GROUP BY f.fruit_id, f.fruit_name, ...` | Nhóm theo trái cây — mỗi loại quả = 1 dòng kết quả |
| `HAVING` (xem bên dưới) | Lọc **sau** khi đã gom nhóm |
| `LIMIT 3` | Chỉ lấy top 3 |

### WHERE vs HAVING — Sự khác biệt sinh tử

```sql
-- ============================================================
-- HAVING: Chỉ lấy trái cây có tổng doanh thu > 300,000 VNĐ
-- ============================================================
SELECT
    f.fruit_name,
    SUM(od.line_total) AS tong_doanh_thu
FROM order_detail od
    INNER JOIN `order` o ON od.order_id = o.order_id
    INNER JOIN fruit f   ON od.fruit_id = f.fruit_id
WHERE o.status != 'CANCELLED'            -- (1) Lọc TRƯỚC khi gom nhóm
GROUP BY f.fruit_id, f.fruit_name
HAVING tong_doanh_thu > 300000           -- (2) Lọc SAU khi gom nhóm
ORDER BY tong_doanh_thu DESC;
```

> 📌 **Quy tắc nhớ:**
> - `WHERE` lọc **từng dòng riêng lẻ** → chạy **TRƯỚC** GROUP BY.
> - `HAVING` lọc **kết quả đã gom nhóm** → chạy **SAU** GROUP BY.
> - Không thể viết `WHERE SUM(od.line_total) > 300000` — MySQL sẽ báo lỗi vì WHERE không nhận aggregate function.

### Bài toán nâng cao: Doanh thu theo danh mục theo tháng

```sql
-- ============================================================
-- DOANH THU THEO DANH MỤC, THEO THÁNG
-- ============================================================
SELECT
    cat.category_name                              AS danh_muc,
    DATE_FORMAT(o.order_date, '%Y-%m')             AS thang,
    COUNT(DISTINCT o.order_id)                     AS so_don,
    SUM(od.line_total)                             AS doanh_thu
FROM order_detail od
    INNER JOIN `order` o    ON od.order_id    = o.order_id
    INNER JOIN fruit f      ON od.fruit_id    = f.fruit_id
    INNER JOIN category cat ON f.category_id  = cat.category_id
WHERE o.status IN ('DELIVERED', 'SHIPPING', 'CONFIRMED')
GROUP BY cat.category_id, cat.category_name, DATE_FORMAT(o.order_date, '%Y-%m')
ORDER BY thang ASC, doanh_thu DESC;
```

---

## 2.3. Phân trang (Pagination) với LIMIT và OFFSET

### Cách cơ bản (dùng trong project Spring Boot)

```sql
-- ============================================================
-- PHÂN TRANG: Trang 1, mỗi trang 3 trái cây
-- ============================================================
SELECT fruit_id, fruit_name, price, stock_qty
FROM fruit
ORDER BY price DESC
LIMIT 3 OFFSET 0;     -- Trang 1: bỏ qua 0, lấy 3

-- Trang 2:
-- LIMIT 3 OFFSET 3;  -- Bỏ qua 3 dòng đầu, lấy 3 dòng tiếp

-- Trang 3:
-- LIMIT 3 OFFSET 6;  -- Bỏ qua 6, lấy 3
```

**Công thức:** `OFFSET = (page_number - 1) × page_size`

### ⚠️ Vấn đề hiệu năng OFFSET lớn

```sql
-- Trang 10,000 với 10 items/page:
SELECT * FROM fruit ORDER BY fruit_id LIMIT 10 OFFSET 99990;
-- MySQL phải ĐỌC 100,000 dòng, vứt bỏ 99,990 dòng, chỉ trả về 10. CỰC KỲ CHẬM!
```

### ✅ Giải pháp: Keyset Pagination (Cursor-based)

```sql
-- Thay vì OFFSET, dùng điều kiện WHERE trên primary key:
-- Giả sử trang trước kết thúc ở fruit_id = 99990
SELECT * FROM fruit
WHERE fruit_id > 99990       -- "Nhảy" thẳng đến vị trí cần
ORDER BY fruit_id
LIMIT 10;
-- MySQL dùng Index trên primary key → tìm ngay vị trí fruit_id > 99990 → NHANH!
```

> 🏢 **Góc doanh nghiệp:**  
> - **OFFSET pagination** phù hợp với admin panel (ít dữ liệu, UI cần "trang 1, 2, 3…").  
> - **Keyset pagination** bắt buộc dùng cho API có hàng triệu records (newsfeed, danh sách sản phẩm).  
> - **Spring Data JPA** mặc định dùng OFFSET pagination qua `Pageable`. Khi dữ liệu lớn, em phải tự viết custom query với keyset.

---

## 2.4. Subquery và một số kỹ thuật nâng cao

```sql
-- ============================================================
-- SUBQUERY: Tìm khách hàng có tổng chi tiêu cao nhất
-- ============================================================
SELECT c.full_name, c.email, sub.tong_chi_tieu
FROM customer c
INNER JOIN (
    SELECT
        o.customer_id,
        SUM(o.total_amount) AS tong_chi_tieu
    FROM `order` o
    WHERE o.status != 'CANCELLED'
    GROUP BY o.customer_id
) sub ON c.customer_id = sub.customer_id
ORDER BY sub.tong_chi_tieu DESC
LIMIT 1;

-- ============================================================
-- EXISTS: Tìm danh mục có ÍT NHẤT 1 trái cây tồn kho > 50
-- ============================================================
SELECT cat.category_name
FROM category cat
WHERE EXISTS (
    SELECT 1
    FROM fruit f
    WHERE f.category_id = cat.category_id
      AND f.stock_qty > 50
);
```

> 💡 **EXISTS vs IN:**  
> - `EXISTS` dừng ngay khi tìm thấy dòng đầu tiên → nhanh hơn khi subquery trả về nhiều dòng.  
> - `IN` phải tính toàn bộ subquery trước → tốt hơn khi subquery nhỏ.  
> - Trong thực tế doanh nghiệp, **hầu hết các trường hợp JOIN sẽ tốt hơn subquery** về hiệu năng. Dùng subquery khi logic quá phức tạp để viết bằng JOIN.

---

## 2.5. 🎤 GÓC PHỎNG VẤN — PHẦN 2

### Câu 1: "Phân biệt INNER JOIN, LEFT JOIN, RIGHT JOIN. Khi nào dùng LEFT JOIN?"

> **Câu trả lời sắc sảo:**
>
> - **INNER JOIN** chỉ trả về dòng khớp ở **cả hai** bảng. Ví dụ: JOIN `fruit` với `order_detail` chỉ ra trái cây đã bán.
>
> - **LEFT JOIN** trả về **tất cả** dòng bảng trái, dù bảng phải không có dòng khớp (cột bảng phải = NULL). Ví dụ: LEFT JOIN `fruit` với `order_detail` sẽ liệt kê tất cả trái cây — kể cả loại chưa ai mua (cột `order_id` sẽ là NULL).
>
> - **RIGHT JOIN** ngược lại với LEFT JOIN. Trong thực tế, ta hiếm khi dùng RIGHT JOIN vì có thể viết lại bằng LEFT JOIN bằng cách đổi thứ tự bảng, giúp code dễ đọc hơn.
>
> **Khi nào dùng LEFT JOIN:** Khi cần hiển thị "tất cả bản ghi một bên, kể cả không có dữ liệu liên quan". Trong Fruit Store, tôi dùng LEFT JOIN để tìm trái cây tồn kho chưa ai mua, hoặc khách hàng đã đăng ký nhưng chưa từng đặt hàng.

### Câu 2: "WHERE và HAVING khác nhau thế nào?"

> **Câu trả lời sắc sảo:**
>
> - `WHERE` lọc **từng dòng riêng lẻ**, thực thi **TRƯỚC** GROUP BY. Nó không chấp nhận aggregate function (SUM, COUNT, AVG…).
>
> - `HAVING` lọc **kết quả đã gom nhóm**, thực thi **SAU** GROUP BY. Nó chấp nhận aggregate function.
>
> Ví dụ thực tế trong Fruit Store: Tôi muốn tìm trái cây có doanh thu trên 300,000 VNĐ, chỉ tính đơn đã giao.
> - `WHERE o.status = 'DELIVERED'` → lọc bỏ đơn hủy **trước** khi tính tổng.
> - `HAVING SUM(od.line_total) > 300000` → sau khi tính tổng, chỉ giữ trái cây vượt ngưỡng.
>
> Nếu viết `WHERE SUM(...) > 300000`, MySQL sẽ báo lỗi ngay vì WHERE chạy trước GROUP BY — lúc đó chưa có nhóm nào để SUM.

### Câu 3: "OFFSET pagination có vấn đề gì khi dữ liệu lớn? Giải pháp là gì?"

> **Câu trả lời sắc sảo:**
>
> **Vấn đề:** Khi OFFSET lớn (ví dụ `LIMIT 10 OFFSET 99990`), MySQL phải đọc 100,000 dòng từ đĩa, sắp xếp, rồi vứt bỏ 99,990 dòng, chỉ trả về 10. Càng lùi về sau, càng chậm — độ phức tạp O(N).
>
> **Giải pháp: Keyset Pagination (Cursor-based).** Thay vì dùng OFFSET, ta dùng điều kiện WHERE trên cột đã đánh Index (thường là primary key hoặc cột sort). Ví dụ:
> ```sql
> SELECT * FROM fruit WHERE fruit_id > :lastSeenId ORDER BY fruit_id LIMIT 10;
> ```
> MySQL dùng B-Tree Index nhảy thẳng đến vị trí `fruit_id > lastSeenId` → O(log N), nhanh hơn nhiều lần.
>
> **Hạn chế của Keyset:** Không thể nhảy đến trang bất kỳ (trang 50 chẳng hạn), chỉ đi "trang tiếp theo/trang trước". Phù hợp cho infinite scroll, API feed, không phù hợp cho UI có nút chọn trang. Trong Spring Boot, cần tự viết custom repository thay vì dùng `Pageable` mặc định.

---

# PHẦN 3: TỐI ƯU HÓA HIỆU NĂNG (BÍ QUYẾT ĐÀM PHÁN LƯƠNG CAO)

> *"Junior biết viết query. Senior biết viết query chạy nhanh gấp 100 lần. Đó là khoảng cách lương 15 triệu và 40 triệu."*

## 3.1. B-Tree Index — Cơ chế hoạt động

### Index là gì? Tại sao nhanh?

Hãy tưởng tượng bảng `fruit` có 1 triệu dòng. Khi chạy:

```sql
SELECT * FROM fruit WHERE price = 180000;
```

**Không có Index:** MySQL phải quét **từng dòng một** (Full Table Scan) — đọc 1 triệu dòng → O(N). Như lật từng trang sách 1,000 trang để tìm từ "Sầu riêng".

**Có Index trên `price`:** MySQL dùng cấu trúc B-Tree (cây cân bằng) → tìm trong ~20 bước so sánh (log₂(1,000,000) ≈ 20) → O(log N). Như dùng **mục lục** ở cuối sách.

### B-Tree hoạt động ra sao?

```
                         [85000 | 150000]
                        /        |        \
              [20000|35000|45000] [65000|85000|120000] [180000|220000|450000]
              /    |    |    \      /   |   |    \       /     |      |     \
           [data] [data] ...   [data] [data] ...     [data]  [data]  [data]
```

- Mỗi nút (node) chứa nhiều key, được **sắp xếp tăng dần**.
- Tìm `price = 180000`: so sánh với root → đi phải → so sánh → tìm thấy leaf → trỏ về dòng dữ liệu.
- **Mỗi nút = 1 page trên đĩa (16KB default)**. Ít I/O hơn full scan rất nhiều.

### Cột nào PHẢI đánh Index?

| Loại cột | Ví dụ trong Fruit Store | Lý do |
|---|---|---|
| **Primary Key** | `fruit_id`, `order_id` | MySQL InnoDB tự động tạo Clustered Index |
| **Foreign Key** | `fruit.category_id`, `order.customer_id` | JOIN sẽ nhanh hơn rất nhiều lần |
| **Cột WHERE thường xuyên** | `order.status`, `order.order_date` | Điều kiện lọc cần Index để tránh full scan |
| **Cột ORDER BY** | `fruit.price` | Sort sẽ dùng Index thay vì filesort |
| **Cột UNIQUE** | `customer.email` | Đảm bảo tính duy nhất + tự động tạo Index |

### Cột nào TUYỆT ĐỐI KHÔNG nên đánh Index?

| Tình huống | Ví dụ | Lý do |
|---|---|---|
| **Cột có ít giá trị phân biệt (low cardinality)** | `fruit.is_seasonal` (chỉ có 0 hoặc 1) | Index chẳng giúp lọc bớt bao nhiêu — MySQL optimizer sẽ chọn full scan |
| **Bảng rất nhỏ** | `category` (chỉ 4 dòng) | Full scan 4 dòng nhanh hơn tra Index |
| **Cột thường xuyên UPDATE** | Cột `stock_qty` nếu cập nhật liên tục | Mỗi lần UPDATE = phải cập nhật cả Index → overhead lớn |
| **Cột TEXT/BLOB dài** | `order.note` | Index trên text dài rất tốn bộ nhớ, hiệu quả thấp |

> ⚠️ **Sai lầm phổ biến của Fresher:** "Đánh Index hết tất cả các cột cho chắc."  
> → **SAI.** Mỗi Index tốn dung lượng đĩa và RAM. Mỗi INSERT/UPDATE/DELETE phải cập nhật TẤT CẢ Index liên quan. Đánh Index bừa bãi = ghi dữ liệu chậm đi gấp đôi.

---

## 3.2. Đọc hiểu lệnh EXPLAIN — "Siêu âm" cho query

### Cách dùng

```sql
-- Thêm EXPLAIN trước bất kỳ SELECT nào:
EXPLAIN SELECT f.fruit_name, od.quantity
FROM fruit f
INNER JOIN order_detail od ON f.fruit_id = od.fruit_id
WHERE f.price > 100000;
```

### Giải thích các cột quan trọng

| Cột | Ý nghĩa | Giá trị tốt | Giá trị xấu (🚩) |
|---|---|---|---|
| **type** | Kiểu truy cập dữ liệu | `const`, `eq_ref`, `ref`, `range` | `ALL` (Full Table Scan!) |
| **key** | Index thực sự được dùng | Tên Index cụ thể | `NULL` (không dùng Index!) |
| **rows** | Số dòng MySQL **ước tính** phải đọc | Số nhỏ | Bằng tổng số dòng bảng |
| **Extra** | Thông tin bổ sung | `Using index` (covering index) | `Using filesort`, `Using temporary` |

**Bảng tra cứu `type` — từ tốt nhất → tệ nhất:**

| type | Ý nghĩa | Hiệu năng |
|---|---|---|
| `system` / `const` | Bảng chỉ có 1 dòng / truy vấn bằng Primary Key | ⚡ Tốt nhất |
| `eq_ref` | JOIN bằng Primary Key hoặc Unique Key, mỗi dòng bảng trái khớp đúng 1 dòng bảng phải | ⚡ Rất tốt |
| `ref` | JOIN/WHERE bằng Index (non-unique), có thể khớp nhiều dòng | ✅ Tốt |
| `range` | Dùng Index để quét một khoảng (>, <, BETWEEN, IN) | ✅ Chấp nhận được |
| `index` | Quét toàn bộ Index (tốt hơn quét bảng, nhưng vẫn đọc nhiều) | ⚠️ Cẩn thận |
| `ALL` | **Full Table Scan** — đọc TOÀN BỘ bảng | 🔴 BÁO ĐỘNG |

### Ví dụ thực hành

```sql
-- ============================================================
-- CASE 1: Query KHÔNG có Index → Full Table Scan
-- ============================================================
EXPLAIN SELECT * FROM fruit WHERE origin = 'Bến Tre';
-- type = ALL, key = NULL → MySQL quét toàn bộ bảng fruit!

-- ============================================================
-- SỬA: Thêm Index cho cột origin
-- ============================================================
ALTER TABLE fruit ADD INDEX idx_fruit_origin (origin);

-- Chạy lại EXPLAIN:
EXPLAIN SELECT * FROM fruit WHERE origin = 'Bến Tre';
-- type = ref, key = idx_fruit_origin → Giờ dùng Index rồi!

-- ============================================================
-- CASE 2: Query dùng Index nhưng EXPLAIN cảnh báo
-- ============================================================
EXPLAIN SELECT f.fruit_name, cat.category_name
FROM fruit f
INNER JOIN category cat ON f.category_id = cat.category_id
ORDER BY f.fruit_name;
-- Extra = "Using filesort" → MySQL phải sort lại kết quả trên đĩa
-- → Có thể thêm Index trên fruit_name nếu query này chạy thường xuyên
```

> 🎯 **Quy trình tối ưu query trong doanh nghiệp:**
> 1. Viết query hoàn chỉnh.
> 2. Chạy `EXPLAIN` (hoặc `EXPLAIN ANALYZE` trong MySQL 8.0.18+).
> 3. Xem cột `type` — có `ALL` không?
> 4. Xem cột `key` — có `NULL` không?
> 5. Xem cột `Extra` — có `Using filesort` hoặc `Using temporary` không?
> 6. Thêm/sửa Index nếu cần → chạy lại EXPLAIN → so sánh.

---

## 3.3. Bắt bệnh lỗi N+1 Query trong Spring Data JPA

### N+1 Query là gì?

Đây là lỗi hiệu năng **phổ biến nhất** khi dùng ORM (JPA/Hibernate).

**Kịch bản Fruit Store:**

Giả sử em có Entity `Order` và `OrderDetail`:

```java
// ❌ ENTITY GÂY LỖI N+1
@Entity
@Table(name = "`order`")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer orderId;

    @ManyToOne(fetch = FetchType.EAGER)  // ← Mặc định ManyToOne là EAGER
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderDetail> orderDetails;

    // getters, setters...
}
```

**Khi gọi `orderRepository.findAll()`:**

```
-- Hibernate phát ra:
-- Query 1: SELECT * FROM `order`                         ← 1 query lấy 8 đơn hàng
-- Query 2: SELECT * FROM customer WHERE customer_id = 1   ← Lấy info khách đơn 1
-- Query 3: SELECT * FROM customer WHERE customer_id = 2   ← Lấy info khách đơn 2
-- Query 4: SELECT * FROM customer WHERE customer_id = 1   ← LẠI lấy khách đơn 3 (trùng!)
-- ...
-- Query 9: SELECT * FROM customer WHERE customer_id = 7   ← Lấy info khách đơn 8
```

**Tổng cộng: 1 + N = 1 + 8 = 9 queries!** Với 10,000 đơn hàng → 10,001 queries → database sập.

### ✅ Giải pháp 1: JOIN FETCH trong JPQL

```java
// ✅ REPOSITORY VỚI JOIN FETCH
@Repository
public interface OrderRepository extends JpaRepository<Order, Integer> {

    @Query("SELECT o FROM Order o " +
           "JOIN FETCH o.customer " +
           "WHERE o.status != 'CANCELLED'")
    List<Order> findAllWithCustomer();

    // JOIN FETCH nhiều quan hệ:
    @Query("SELECT DISTINCT o FROM Order o " +
           "JOIN FETCH o.customer " +
           "JOIN FETCH o.orderDetails od " +
           "JOIN FETCH od.fruit " +
           "WHERE o.status = :status")
    List<Order> findByStatusWithDetails(@Param("status") String status);
}
```

**Kết quả:** Hibernate phát ra **DUY NHẤT 1 query** với SQL JOIN:

```sql
SELECT o.*, c.*, od.*, f.*
FROM `order` o
INNER JOIN customer c ON o.customer_id = c.customer_id
INNER JOIN order_detail od ON o.order_id = od.order_id
INNER JOIN fruit f ON od.fruit_id = f.fruit_id
WHERE o.status = 'DELIVERED';
```

### ✅ Giải pháp 2: @EntityGraph

```java
@Repository
public interface OrderRepository extends JpaRepository<Order, Integer> {

    @EntityGraph(attributePaths = {"customer", "orderDetails", "orderDetails.fruit"})
    @Query("SELECT o FROM Order o WHERE o.status != 'CANCELLED'")
    List<Order> findAllWithGraph();
}
```

### ✅ Giải pháp 3: @BatchSize (giảm N+1 thành N/batch + 1)

```java
@Entity
@Table(name = "customer")
public class Customer {
    // ...
}

// Trong application.yml:
// spring:
//   jpa:
//     properties:
//       hibernate:
//         default_batch_fetch_size: 20
```

Với `batch_size = 20`: Thay vì 8 query riêng lẻ, Hibernate gom thành:
```sql
SELECT * FROM customer WHERE customer_id IN (1, 2, 3, 4, 5, 6, 7);  -- 1 query duy nhất
```

> ⚠️ **Quy tắc vàng trong doanh nghiệp:**
> 1. **LUÔN** set `FetchType.LAZY` cho `@ManyToOne` và `@OneToMany`.
> 2. **LUÔN** bật `hibernate.default_batch_fetch_size: 20` (hoặc 50) trong production.
> 3. Khi cần dữ liệu related → viết `JOIN FETCH` cụ thể thay vì để Hibernate tự quyết.
> 4. Bật `spring.jpa.show-sql=true` trong dev → kiểm tra số lượng query phát ra.

---

## 3.4. 🎤 GÓC PHỎNG VẤN — PHẦN 3

### Câu 1: "B-Tree Index hoạt động như thế nào? Tại sao nó nhanh hơn Full Table Scan?"

> **Câu trả lời sắc sảo:**
>
> B-Tree Index tổ chức dữ liệu theo cấu trúc cây cân bằng. Mỗi nút chứa nhiều key được sắp xếp, và mỗi nút tương ứng với một page trên đĩa (mặc định 16KB trong InnoDB).
>
> Khi tìm `price = 180000`:
> - **Full Table Scan** đọc từng dòng từ đầu đến cuối → O(N). Với 1 triệu dòng = 1 triệu lần so sánh.
> - **B-Tree** bắt đầu từ root, so sánh key, đi theo nhánh phù hợp → O(log N). Với 1 triệu dòng ≈ 20 lần so sánh = 20 lần đọc đĩa.
>
> Thêm vào đó, InnoDB trong MySQL dùng **Clustered Index** trên Primary Key — dữ liệu bảng thực sự **được sắp xếp** theo PK trên đĩa. Nên truy vấn theo PK nhanh nhất. Index phụ (Secondary Index) lưu giá trị cột + con trỏ về PK, rồi tra ngược về Clustered Index để lấy dữ liệu đầy đủ.
>
> **Tuy nhiên,** Index không phải miễn phí: mỗi INSERT/UPDATE/DELETE phải cập nhật tất cả Index liên quan, tốn thêm I/O. Nên chỉ đánh Index cho cột thực sự cần.

### Câu 2: "Đọc output EXPLAIN, em nhìn vào cột nào đầu tiên?"

> **Câu trả lời sắc sảo:**
>
> Tôi xem theo thứ tự ưu tiên:
>
> 1. **`type`** — nếu thấy `ALL` → Full Table Scan → cần kiểm tra ngay. Giá trị tốt là `const`, `eq_ref`, `ref`, `range`.
>
> 2. **`key`** — nếu `NULL` nghĩa là không có Index nào được sử dụng, dù có thể đã tạo Index (do MySQL optimizer quyết định không dùng).
>
> 3. **`rows`** — số dòng MySQL ước tính phải đọc. Nếu con số này bằng tổng số dòng bảng → full scan.
>
> 4. **`Extra`** — `Using filesort` (phải sort trên đĩa, chậm), `Using temporary` (phải tạo bảng tạm, chậm hơn nữa), `Using index` (covering index, tốt nhất — không cần tra ngược về bảng).
>
> Trong dự án Fruit Store, khi tôi thấy EXPLAIN của query TOP 3 doanh thu hiển thị `type = ALL` trên bảng `order_detail` → tôi biết cần thêm composite index `(fruit_id, order_id)` hoặc kiểm tra lại điều kiện JOIN.

### Câu 3: "N+1 Query là gì? Em xử lý thế nào trong Spring Data JPA?"

> **Câu trả lời sắc sảo:**
>
> N+1 Query xảy ra khi ORM (Hibernate) phát ra 1 query để lấy danh sách entity cha, rồi phát thêm N query riêng lẻ để lấy entity con/liên quan của từng dòng.
>
> Ví dụ: `orderRepository.findAll()` phát 1 query lấy 8 đơn hàng, rồi 8 query riêng lấy customer của từng đơn → tổng 9 query. Với 10,000 đơn → 10,001 query.
>
> **3 cách xử lý:**
> 1. **JOIN FETCH trong JPQL:** `SELECT o FROM Order o JOIN FETCH o.customer` → Hibernate sinh 1 SQL JOIN duy nhất.
> 2. **@EntityGraph:** Khai báo `attributePaths` để Hibernate tự JOIN.
> 3. **@BatchSize / `default_batch_fetch_size`:** Gom N query thành N/batch query bằng `WHERE id IN (...)`.
>
> **Quy tắc sống còn:** Luôn dùng `FetchType.LAZY`, không bao giờ dùng `EAGER`. Bật `show-sql` trong dev để phát hiện sớm. Trong production, dùng Spring Boot Actuator + Hibernate Statistics để monitor số query.

---

# PHẦN 4: BẢO VỆ DỮ LIỆU — TRANSACTION & CONCURRENCY

> *"Database không chỉ cần nhanh — nó cần ĐÚNG. Mất 0.01 VNĐ do sai số là chấp nhận được. Mất 1 đơn hàng do race condition là không thể tha thứ."*

## 4.1. ACID — 4 thuộc tính qua kịch bản thanh toán đơn hàng

### Kịch bản: Anh An mua 2kg Sầu riêng Ri6

Quá trình thanh toán bao gồm **3 thao tác** phải xảy ra **đồng thời hoặc không xảy ra gì cả**:

1. Tạo dòng mới trong `order_detail` (ghi nhận mua 2kg sầu riêng).
2. Giảm `stock_qty` trong `fruit` (trừ tồn kho 2kg).
3. Cập nhật `total_amount` trong `order` (cộng thêm 360,000 VNĐ).

```sql
-- ============================================================
-- TRANSACTION THANH TOÁN — MINH HỌA ACID
-- ============================================================
START TRANSACTION;

-- Bước 1: Tạo chi tiết đơn hàng
INSERT INTO order_detail (order_id, fruit_id, quantity, unit_price)
VALUES (5, 1, 2, 180000.00);

-- Bước 2: Giảm tồn kho
UPDATE fruit
SET stock_qty = stock_qty - 2
WHERE fruit_id = 1 AND stock_qty >= 2;  -- Kiểm tra đủ hàng!

-- Bước 3: Cập nhật tổng đơn hàng
UPDATE `order`
SET total_amount = total_amount + 360000.00
WHERE order_id = 5;

-- Nếu tất cả thành công:
COMMIT;

-- Nếu BẤT KỲ bước nào thất bại:
-- ROLLBACK;
```

### 4 thuộc tính ACID giải thích qua kịch bản trên

| Thuộc tính | Ý nghĩa | Áp dụng vào kịch bản |
|---|---|---|
| **A — Atomicity** (Nguyên tử) | Tất cả hoặc không gì cả | Nếu bước 2 thất bại (hết hàng), bước 1 cũng bị hủy (ROLLBACK). Không có chuyện "ghi nhận mua nhưng không trừ kho". |
| **C — Consistency** (Nhất quán) | Database luôn ở trạng thái hợp lệ trước và sau transaction | Trước: tổng kho = 25. Sau: tổng kho = 23, đơn hàng tăng 360K. Tổng giá trị không "bốc hơi". |
| **I — Isolation** (Cô lập) | Transaction này không nhìn thấy dữ liệu "dở dang" của transaction khác | Anh An đang mua → chị Bình đồng thời xem tồn kho → chị Bình thấy 25 hoặc 23, **KHÔNG BAO GIỜ** thấy giá trị trung gian sai lệch. |
| **D — Durability** (Bền vững) | Một khi COMMIT xong, dữ liệu tồn tại vĩnh viễn | Ngay sau COMMIT, dù server mất điện → khi restart, đơn hàng của anh An vẫn còn. MySQL dùng redo log (WAL) để đảm bảo. |

---

## 4.2. 4 mức độ cách ly (Isolation Levels)

### Các hiện tượng bất thường

| Hiện tượng | Mô tả | Ví dụ Fruit Store |
|---|---|---|
| **Dirty Read** | Đọc dữ liệu mà transaction khác **chưa COMMIT** | Chị Bình thấy tồn kho sầu riêng = 23 (anh An đang mua nhưng chưa COMMIT). Anh An ROLLBACK → tồn kho thực tế vẫn 25. Chị Bình đã hành động sai dựa trên dữ liệu "bẩn". |
| **Non-Repeatable Read** | Đọc cùng 1 dòng 2 lần trong 1 transaction, kết quả **khác nhau** | Chị Bình đọc giá sầu riêng = 180,000. Trong khi đó, admin sửa giá thành 200,000 và COMMIT. Chị Bình đọc lại → thấy 200,000. Cùng 1 transaction mà đọc ra 2 giá khác nhau! |
| **Phantom Read** | Query trả về **số lượng dòng khác nhau** khi chạy 2 lần | Chị Bình đếm số trái cây giá > 100K → ra 5 loại. Admin thêm "Vải thiều" giá 120K rồi COMMIT. Chị Bình đếm lại → ra 6 loại. Dòng mới xuất hiện như "bóng ma" (phantom). |

### 4 mức độ Isolation

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Hiệu năng |
|---|---|---|---|---|
| **READ UNCOMMITTED** | ❌ Xảy ra | ❌ Xảy ra | ❌ Xảy ra | ⚡ Nhanh nhất |
| **READ COMMITTED** | ✅ Chặn | ❌ Xảy ra | ❌ Xảy ra | Nhanh |
| **REPEATABLE READ** (MySQL default) | ✅ Chặn | ✅ Chặn | ⚠️ Xảy ra* | Trung bình |
| **SERIALIZABLE** | ✅ Chặn | ✅ Chặn | ✅ Chặn | 🐢 Chậm nhất |

> ⚠️ **Lưu ý quan trọng về MySQL InnoDB:**  
> MySQL InnoDB ở mức REPEATABLE READ sử dụng cơ chế **MVCC (Multi-Version Concurrency Control)** kết hợp **gap locking**, nên trong thực tế **phần lớn trường hợp Phantom Read KHÔNG xảy ra** — khác với tiêu chuẩn SQL lý thuyết. Đây là điểm ghi thêm "điểm cộng" nếu nói trong phỏng vấn.

```sql
-- ============================================================
-- THÍ NGHIỆM ISOLATION LEVELS
-- ============================================================

-- Terminal 1 (Anh An):
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
UPDATE fruit SET price = 999999.00 WHERE fruit_id = 1;
-- CHƯA COMMIT!

-- Terminal 2 (Chị Bình):
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
START TRANSACTION;
SELECT price FROM fruit WHERE fruit_id = 1;
-- Kết quả: 999999.00 ← DIRTY READ! Đọc được dữ liệu chưa COMMIT!

-- Terminal 1:
ROLLBACK;
-- Giá thực tế vẫn là 180000. Chị Bình đã đọc sai!

-- ============================================================
-- BÂY GIỜ THỬ VỚI READ COMMITTED:
-- ============================================================

-- Terminal 2:
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
SELECT price FROM fruit WHERE fruit_id = 1;
-- Kết quả: 180000.00 ← ĐÚNG! Không thấy dữ liệu chưa COMMIT.
COMMIT;
```

---

## 4.3. Bài toán Concurrency: 2 khách hàng cùng mua quả sầu riêng cuối cùng

### Kịch bản: `stock_qty = 1` (còn đúng 1kg)

Anh An và chị Bình **đồng thời** bấm nút "Mua ngay" trên app.

**❌ Nếu không có cơ chế khóa:**

| Thời điểm | Anh An (Thread 1) | Chị Bình (Thread 2) |
|---|---|---|
| T1 | `SELECT stock_qty FROM fruit WHERE fruit_id = 1` → Đọc: **1** | |
| T2 | | `SELECT stock_qty FROM fruit WHERE fruit_id = 1` → Đọc: **1** |
| T3 | `stock_qty >= 1` → OK → Tiến hành mua | |
| T4 | | `stock_qty >= 1` → OK → Tiến hành mua |
| T5 | `UPDATE SET stock_qty = stock_qty - 1` → **stock = 0** | |
| T6 | | `UPDATE SET stock_qty = stock_qty - 1` → **stock = -1** 🔴 |

**Kết quả:** Tồn kho = -1. Bán 2kg nhưng chỉ có 1kg. **Thảm họa.**

### ✅ Giải pháp 1: Pessimistic Lock (Khóa bi quan)

> **Triết lý:** "Tôi nghĩ CHẮC CHẮN sẽ có người khác cạnh tranh → khóa dòng lại trước khi làm."

```sql
-- ============================================================
-- PESSIMISTIC LOCK: SELECT ... FOR UPDATE
-- ============================================================

-- Anh An (Thread 1):
START TRANSACTION;

-- Khóa dòng fruit_id = 1, không cho ai đọc/ghi cho đến khi COMMIT
SELECT stock_qty FROM fruit WHERE fruit_id = 1 FOR UPDATE;
-- Kết quả: stock_qty = 1

-- Kiểm tra:
-- IF stock_qty >= 1 THEN:
UPDATE fruit SET stock_qty = stock_qty - 1 WHERE fruit_id = 1;
INSERT INTO order_detail (order_id, fruit_id, quantity, unit_price) VALUES (...);

COMMIT;  -- Giải phóng lock

-- ============================================================
-- Chị Bình (Thread 2) — xảy ra ĐỒNG THỜI:
-- ============================================================
START TRANSACTION;

SELECT stock_qty FROM fruit WHERE fruit_id = 1 FOR UPDATE;
-- ⏳ BỊ BLOCK! Phải ĐỢI Anh An COMMIT xong mới chạy được!

-- Sau khi Anh An COMMIT:
-- Kết quả: stock_qty = 0
-- IF stock_qty >= 1 → FALSE → Từ chối đơn hàng → Thông báo "Hết hàng"

ROLLBACK;
```

**Trong Spring Data JPA:**

```java
@Repository
public interface FruitRepository extends JpaRepository<Fruit, Integer> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT f FROM Fruit f WHERE f.fruitId = :id")
    Optional<Fruit> findByIdForUpdate(@Param("id") Integer id);
}
```

```java
@Service
@Transactional
public class OrderService {

    public void purchaseFruit(int fruitId, int quantity) {
        // 1. Lock dòng fruit
        Fruit fruit = fruitRepository.findByIdForUpdate(fruitId)
            .orElseThrow(() -> new NotFoundException("Fruit not found"));

        // 2. Kiểm tra tồn kho
        if (fruit.getStockQty() < quantity) {
            throw new OutOfStockException("Không đủ hàng!");
        }

        // 3. Trừ kho
        fruit.setStockQty(fruit.getStockQty() - quantity);
        fruitRepository.save(fruit);

        // 4. Tạo đơn hàng...
    }
}
```

### ✅ Giải pháp 2: Optimistic Lock (Khóa lạc quan)

> **Triết lý:** "Tôi nghĩ ÍT KHI có người khác cạnh tranh → không khóa, nhưng kiểm tra trước khi ghi."

**Cơ chế:** Thêm cột `version` vào bảng. Mỗi lần UPDATE, kiểm tra version có trùng không.

```sql
-- Thêm cột version vào bảng fruit
ALTER TABLE fruit ADD COLUMN version INT NOT NULL DEFAULT 0;
```

```sql
-- ============================================================
-- OPTIMISTIC LOCK: Dùng cột version
-- ============================================================

-- Anh An đọc:
SELECT fruit_id, stock_qty, version FROM fruit WHERE fruit_id = 1;
-- Kết quả: stock_qty = 1, version = 0

-- Anh An cập nhật (kèm điều kiện version):
UPDATE fruit
SET stock_qty = stock_qty - 1, version = version + 1
WHERE fruit_id = 1 AND version = 0;
-- affected_rows = 1 → Thành công! version giờ = 1

-- Chị Bình cũng đọc được version = 0 (trước khi Anh An commit):
UPDATE fruit
SET stock_qty = stock_qty - 1, version = version + 1
WHERE fruit_id = 1 AND version = 0;
-- affected_rows = 0 → THẤT BẠI! Version đã thay đổi!
-- → Throw OptimisticLockException → Retry hoặc thông báo "Hết hàng"
```

**Trong Spring Data JPA:**

```java
@Entity
public class Fruit {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer fruitId;

    @Version  // ← Hibernate tự quản lý version
    private Integer version;

    private Integer stockQty;
    // ...
}
```

> Khi 2 transaction cùng đọc version = 0, người COMMIT sau sẽ nhận `ObjectOptimisticLockingFailureException`. Em cần catch exception này và retry hoặc trả lỗi cho user.

### So sánh 2 loại Lock

| Tiêu chí | Pessimistic Lock | Optimistic Lock |
|---|---|---|
| **Cơ chế** | Khóa dòng bằng `SELECT ... FOR UPDATE` | Kiểm tra `version` khi UPDATE |
| **Khi nào dùng** | Xung đột thường xuyên (flash sale, đặt vé) | Xung đột hiếm khi (cập nhật profile, giỏ hàng) |
| **Ưu điểm** | Đảm bảo tuyệt đối, không cần retry | Không khóa → throughput cao hơn |
| **Nhược điểm** | Giảm throughput vì thread phải chờ. Risk: deadlock | Phải xử lý retry khi conflict. UX có thể bị ảnh hưởng |
| **Fruit Store** | Giảm tồn kho khi đặt hàng (xung đột cao) | Cập nhật thông tin trái cây (xung đột thấp) |

---

## 4.4. 🎤 GÓC PHỎNG VẤN — PHẦN 4

### Câu 1: "Giải thích 4 thuộc tính ACID. Cho ví dụ thực tế."

> **Câu trả lời sắc sảo:**
>
> ACID là 4 đảm bảo mà mọi database transaction phải tuân thủ:
>
> - **Atomicity:** Tất cả hoặc không gì cả. Ví dụ trong Fruit Store: khi khách mua sầu riêng, phải đồng thời INSERT order_detail, UPDATE trừ kho, UPDATE tổng đơn. Nếu trừ kho thất bại → toàn bộ transaction ROLLBACK, không có chuyện "ghi nhận mua nhưng không trừ kho".
>
> - **Consistency:** Database luôn ở trạng thái hợp lệ. Trước transaction: kho = 25. Sau: kho = 23, đơn hàng +360K. Không mất mát dữ liệu, mọi constraint (Foreign Key, CHECK) được đảm bảo.
>
> - **Isolation:** Transaction chạy song song không ảnh hưởng lẫn nhau. Anh An đang mua sầu riêng → chị Bình xem tồn kho → thấy 25 (chưa commit) hoặc 23 (đã commit), không bao giờ thấy giá trị trung gian.
>
> - **Durability:** Sau COMMIT, dữ liệu tồn tại vĩnh viễn kể cả khi server crash. MySQL InnoDB đảm bảo bằng redo log (Write-Ahead Logging) — ghi log trước khi ghi dữ liệu thực.

### Câu 2: "Dirty Read, Non-Repeatable Read, Phantom Read là gì? MySQL mặc định chặn được loại nào?"

> **Câu trả lời sắc sảo:**
>
> - **Dirty Read:** Đọc dữ liệu chưa COMMIT. Nguy hiểm nhất vì dữ liệu đó có thể bị ROLLBACK.
> - **Non-Repeatable Read:** Đọc cùng 1 dòng 2 lần trong 1 transaction, ra kết quả khác (do transaction khác UPDATE và COMMIT).
> - **Phantom Read:** Chạy cùng 1 query 2 lần, số lượng dòng trả về khác nhau (do transaction khác INSERT/DELETE và COMMIT).
>
> MySQL InnoDB mặc định dùng **REPEATABLE READ** → chặn Dirty Read và Non-Repeatable Read. Về lý thuyết SQL chuẩn, Phantom Read vẫn có thể xảy ra ở mức này. **Tuy nhiên**, InnoDB dùng MVCC kết hợp gap locking nên trong thực tế hầu hết Phantom Read cũng được ngăn chặn — đây là điểm đặc biệt của MySQL so với PostgreSQL hay Oracle.

### Câu 3: "Optimistic Lock và Pessimistic Lock khác nhau thế nào? Khi nào dùng cái nào?"

> **Câu trả lời sắc sảo:**
>
> - **Pessimistic Lock** giả định xung đột **chắc chắn xảy ra** → khóa dòng trước bằng `SELECT ... FOR UPDATE`. Thread khác phải **đợi** đến khi lock được giải phóng. **Dùng khi:** tần suất xung đột cao (flash sale, giảm tồn kho, đặt vé máy bay).
>
> - **Optimistic Lock** giả định xung đột **hiếm khi xảy ra** → không khóa, thêm cột `version`. Khi UPDATE, kiểm tra `WHERE version = :expectedVersion`. Nếu version đã thay đổi → reject → retry. **Dùng khi:** tần suất xung đột thấp (cập nhật profile, sửa mô tả sản phẩm).
>
> **Ví dụ Fruit Store:** Khi 2 khách cùng mua quả sầu riêng cuối cùng:
> - Pessimistic: Người đầu tiên lock dòng, người thứ hai **bị block** cho đến khi người đầu commit. Đảm bảo tuyệt đối nhưng giảm throughput.
> - Optimistic: Cả hai đều đọc `version = 0`. Người commit trước thành công (`version → 1`). Người commit sau thấy version đã thay đổi → bị reject.
>
> Trong Spring Data JPA: Pessimistic dùng `@Lock(LockModeType.PESSIMISTIC_WRITE)`, Optimistic dùng `@Version` trên entity.

---

# PHẦN 5: BÀI TEST PHỎNG VẤN TỔNG HỢP (MOCK INTERVIEW)

> *"Đây là phần quyết định em có nhận offer hay không. Tôi không cần câu trả lời hoàn hảo — tôi cần thấy em BIẾT TƯ DUY."*
>
> **Hướng dẫn:** Đọc từng tình huống, suy nghĩ 5 phút, rồi viết câu trả lời theo chuẩn **STAR** (Situation → Task → Action → Result) trước khi xem đáp án gợi ý.

---

## Tình huống 1: "API danh sách trái cây đột ngột chậm — Response time tăng từ 200ms lên 15 giây"

> **Bối cảnh:** Em là backend developer phụ trách hệ thống Fruit Store. Sáng thứ Hai, team monitoring báo API `GET /api/fruits` (có phân trang, mỗi trang 20 items) response time tăng vọt từ 200ms lên 15 giây. Không có thay đổi code nào cuối tuần. Database có 500,000 trái cây.

**Yêu cầu:** Suy luận nguyên nhân có thể và đưa ra quy trình debug.

> **Đáp án gợi ý theo STAR:**
>
> **Situation:** API phân trang trái cây bất ngờ chậm 75 lần, không có code change.
>
> **Task:** Xác định root cause và khôi phục hiệu năng.
>
> **Action:**
> 1. **Kiểm tra slow query log** (`SET GLOBAL slow_query_log = 'ON'`) → xác định query nào chậm.
> 2. **Chạy EXPLAIN** trên query chậm → nếu `type = ALL` → full table scan → Index có thể bị drop hoặc corrupt.
> 3. **Kiểm tra Index:** `SHOW INDEX FROM fruit` → xác nhận Index còn tồn tại.
> 4. **Kiểm tra bảng thống kê:** `ANALYZE TABLE fruit` → cập nhật lại statistics cho optimizer.
> 5. **Khả năng cao:** Bảng đã vượt ngưỡng — 500K rows, buffer pool không đủ → dữ liệu phải đọc từ đĩa thay vì RAM. Kiểm tra `SHOW GLOBAL STATUS LIKE 'Innodb_buffer_pool_reads'`.
> 6. **Hoặc:** Trang cuối dùng `OFFSET 499980 LIMIT 20` → MySQL đọc gần 500K rows rồi vứt → chuyển sang keyset pagination.
>
> **Result:** Sau khi ANALYZE TABLE + tăng `innodb_buffer_pool_size` + chuyển sang keyset pagination → response time về lại 50ms.

---

## Tình huống 2: "Flash sale Sầu riêng — Tồn kho bị âm, khách hàng khiếu nại"

> **Bối cảnh:** Cửa hàng chạy flash sale Sầu riêng Ri6, giới hạn 100kg. Sau 30 phút, kiểm tra database thấy `stock_qty = -17`. 117 kg đã được bán ra dù chỉ có 100kg. 17 khách hàng nhận mail "Hết hàng, hoàn tiền."

**Yêu cầu:** Phân tích nguyên nhân và thiết kế giải pháp không tái diễn.

> **Đáp án gợi ý theo STAR:**
>
> **Situation:** Flash sale gây overselling — tồn kho bị âm vì race condition.
>
> **Task:** Ngăn chặn overselling, đảm bảo tồn kho không bao giờ âm.
>
> **Action:**
> 1. **Nguyên nhân:** Code kiểm tra `if (stockQty >= quantity)` rồi mới `UPDATE stock_qty = stock_qty - quantity` → giữa 2 câu lệnh, nhiều thread cùng đọc `stock_qty = 1` → cùng thấy "đủ hàng" → cùng trừ kho → âm.
>
> 2. **Giải pháp tầng 1 — Atomic UPDATE với điều kiện:**
>    ```sql
>    UPDATE fruit
>    SET stock_qty = stock_qty - :quantity
>    WHERE fruit_id = :id AND stock_qty >= :quantity;
>    -- Nếu affected_rows = 0 → hết hàng → từ chối đơn
>    ```
>    Gộp kiểm tra và trừ kho vào 1 câu lệnh duy nhất (atomic).
>
> 3. **Giải pháp tầng 2 — Pessimistic Lock:**
>    ```java
>    @Lock(LockModeType.PESSIMISTIC_WRITE)
>    @Query("SELECT f FROM Fruit f WHERE f.fruitId = :id")
>    Optional<Fruit> findByIdForUpdate(@Param("id") Integer id);
>    ```
>
> 4. **Giải pháp tầng 3 — Database constraint:**
>    ```sql
>    ALTER TABLE fruit ADD CONSTRAINT chk_stock_positive CHECK (stock_qty >= 0);
>    ```
>    Hàng rào cuối cùng: database tự reject nếu stock_qty < 0.
>
> **Result:** 3 tầng phòng thủ: atomic query + pessimistic lock + CHECK constraint. Tồn kho không bao giờ âm.

---

## Tình huống 3: "Deadlock xảy ra khi 2 đơn hàng xử lý đồng thời"

> **Bối cảnh:** Log hệ thống báo `Deadlock found when trying to get lock; try restarting transaction`. Xảy ra khi 2 khách hàng đặt hàng cùng lúc, mỗi đơn chứa 2 loại trái cây khác nhau nhưng có 1 loại trùng.

**Yêu cầu:** Giải thích deadlock xảy ra thế nào và cách phòng tránh.

> **Đáp án gợi ý theo STAR:**
>
> **Situation:** Deadlock giữa 2 transaction cùng lock 2 trái cây nhưng theo thứ tự ngược nhau.
>
> **Task:** Hiểu cơ chế deadlock và thiết kế giải pháp.
>
> **Action:**
> 1. **Phân tích deadlock:**
>    | Thời điểm | Thread 1 (Đơn A: Sầu riêng + Cherry) | Thread 2 (Đơn B: Cherry + Sầu riêng) |
>    |---|---|---|
>    | T1 | Lock fruit_id = 1 (Sầu riêng) ✅ | |
>    | T2 | | Lock fruit_id = 6 (Cherry) ✅ |
>    | T3 | Lock fruit_id = 6 (Cherry) → ⏳ ĐỢI Thread 2 | |
>    | T4 | | Lock fruit_id = 1 (Sầu riêng) → ⏳ ĐỢI Thread 1 |
>    | | **🔴 DEADLOCK!** Cả hai chờ nhau mãi mãi | |
>
> 2. **Giải pháp cốt lõi — Lock theo thứ tự cố định:**
>    ```java
>    // LUÔN lock theo fruit_id tăng dần, bất kể thứ tự trong đơn hàng
>    List<Integer> fruitIds = orderRequest.getFruitIds();
>    Collections.sort(fruitIds);  // Sắp xếp trước khi lock!
>
>    for (Integer fruitId : fruitIds) {
>        Fruit fruit = fruitRepository.findByIdForUpdate(fruitId);
>        // Xử lý...
>    }
>    ```
>    Thread 1 và Thread 2 đều lock fruit_id = 1 trước, rồi fruit_id = 6 → không bao giờ deadlock.
>
> 3. **Giải pháp bổ sung:**
>    - Set `innodb_lock_wait_timeout = 5` (tối đa chờ 5 giây).
>    - Catch `DeadlockLoserDataAccessException` trong Spring → retry tự động (tối đa 3 lần).
>
> **Result:** Lock theo thứ tự cố định triệt tiêu deadlock. Retry mechanism xử lý edge case.

---

## Tình huống 4: "Báo cáo doanh thu cuối tháng chạy 45 phút, block toàn bộ hệ thống"

> **Bối cảnh:** Cuối mỗi tháng, team kế toán chạy báo cáo doanh thu tổng hợp. Query JOIN 5 bảng, GROUP BY theo danh mục + tháng, trên 2 triệu dòng order_detail. Query mất 45 phút và trong thời gian đó, API khách hàng bị timeout.

**Yêu cầu:** Thiết kế giải pháp để báo cáo không ảnh hưởng hệ thống chính.

> **Đáp án gợi ý theo STAR:**
>
> **Situation:** Report query nặng block hệ thống OLTP (giao dịch hàng ngày).
>
> **Task:** Tách workload analytics ra khỏi production traffic.
>
> **Action:**
> 1. **Giải pháp 1 — Read Replica:**
>    - Tạo MySQL Replica (slave) → đồng bộ dữ liệu từ Primary (master).
>    - Chạy report query trên Replica → không ảnh hưởng Primary.
>    - Spring Boot config: dùng `@Transactional(readOnly = true)` routing đến Replica.
>
> 2. **Giải pháp 2 — Materialized View / Summary Table:**
>    ```sql
>    -- Tạo bảng tổng hợp, cập nhật hàng đêm bằng scheduled job
>    CREATE TABLE monthly_revenue_summary (
>        summary_id    INT AUTO_INCREMENT PRIMARY KEY,
>        category_id   INT NOT NULL,
>        month_year    CHAR(7) NOT NULL,  -- '2024-08'
>        total_orders  INT NOT NULL,
>        total_revenue DECIMAL(18,2) NOT NULL,
>        updated_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
>        UNIQUE KEY uk_cat_month (category_id, month_year)
>    );
>
>    -- Scheduled job chạy lúc 2h sáng:
>    INSERT INTO monthly_revenue_summary (category_id, month_year, total_orders, total_revenue)
>    SELECT
>        f.category_id,
>        DATE_FORMAT(o.order_date, '%Y-%m'),
>        COUNT(DISTINCT o.order_id),
>        SUM(od.line_total)
>    FROM order_detail od
>        JOIN `order` o ON od.order_id = o.order_id
>        JOIN fruit f ON od.fruit_id = f.fruit_id
>    WHERE o.status != 'CANCELLED'
>    GROUP BY f.category_id, DATE_FORMAT(o.order_date, '%Y-%m')
>    ON DUPLICATE KEY UPDATE
>        total_orders = VALUES(total_orders),
>        total_revenue = VALUES(total_revenue);
>    ```
>    Report query giờ chỉ đọc bảng summary (vài chục dòng) → < 100ms.
>
> 3. **Giải pháp 3 — Tối ưu query gốc:**
>    - Thêm composite index: `CREATE INDEX idx_od_cover ON order_detail (order_id, fruit_id, quantity, unit_price)` → Covering Index, không cần tra bảng.
>    - Partition bảng `order` theo tháng → query chỉ scan partition cần thiết.
>
> **Result:** Read Replica cho short-term, Summary Table cho long-term, Covering Index cho tối ưu tức thì. Report chạy trong < 5 giây.

---

## Tình huống 5: "Database bất ngờ chiếm 95% dung lượng ổ đĩa"

> **Bối cảnh:** Alert lúc 3h sáng: ổ đĩa server database còn 2GB free (tổng 100GB). 1 tháng trước còn 40GB free. Không có thay đổi lượng truy cập đáng kể. Hệ thống Fruit Store vẫn hoạt động bình thường nhưng có thể crash bất cứ lúc nào.

**Yêu cầu:** Quy trình xử lý sự cố và phòng tránh.

> **Đáp án gợi ý theo STAR:**
>
> **Situation:** Dung lượng đĩa tăng bất thường 38GB trong 1 tháng, risk database crash.
>
> **Task:** Xác định nguyên nhân, giải phóng dung lượng, thiết lập monitoring.
>
> **Action:**
> 1. **Tìm bảng/file chiếm nhiều nhất:**
>    ```sql
>    SELECT
>        table_name,
>        ROUND(data_length / 1024 / 1024, 2) AS data_mb,
>        ROUND(index_length / 1024 / 1024, 2) AS index_mb,
>        ROUND(data_free / 1024 / 1024, 2) AS fragmented_mb,
>        table_rows
>    FROM information_schema.tables
>    WHERE table_schema = 'fruit_store'
>    ORDER BY (data_length + index_length) DESC;
>    ```
>
> 2. **Nguyên nhân phổ biến:**
>    - **Binary log tích tụ:** `SHOW BINARY LOGS` → log cũ không được purge.
>      → Fix: `PURGE BINARY LOGS BEFORE '2024-08-01'` + set `expire_logs_days = 7`.
>    - **General/Slow query log quá lớn:** Log ghi toàn bộ query vào file.
>      → Fix: Tắt general log trong production: `SET GLOBAL general_log = 'OFF'`.
>    - **Fragmentation:** DELETE nhiều nhưng InnoDB không tự giải phóng dung lượng.
>      → Fix: `ALTER TABLE order_detail ENGINE = InnoDB` (rebuild bảng, giải phóng fragment).
>    - **Bảng audit/log phình to:** Nếu có bảng log ghi mọi request.
>      → Fix: Partition theo tháng, DROP partition cũ.
>
> 3. **Phòng tránh dài hạn:**
>    - Monitoring: Alert khi đĩa > 70% (warning), > 85% (critical).
>    - Automated cleanup job cho log files.
>    - Partition bảng lớn theo thời gian.
>    - Đánh giá: nếu data thực sự lớn → plan migration lên storage lớn hơn hoặc archive dữ liệu cũ.
>
> **Result:** Giải phóng ~35GB từ binary log + fragmentation. Thiết lập monitoring + auto-purge policy. Hệ thống ổn định.

---

# LỜI KẾT

> 🎓 **Từ thầy:**
>
> Em đã đi qua một hành trình từ `CREATE TABLE` đến `Deadlock Resolution`. Đây không phải toàn bộ kiến thức MySQL — nhưng là **nền tảng vững chắc** để:
>
> 1. **Làm tốt khóa luận** — Thiết kế database chuẩn, viết query hiệu quả, xử lý concurrency.
> 2. **Vượt qua phỏng vấn** — Trả lời được 90% câu hỏi MySQL dành cho Fresher/Junior.
> 3. **Làm việc thực tế** — Biết EXPLAIN, biết N+1, biết khi nào dùng Pessimistic vs Optimistic Lock.
>
> **Bước tiếp theo em nên làm:**
> - [ ] Chạy TOÀN BỘ code SQL trong tài liệu này trên MySQL Workbench.
> - [ ] Tự viết lại query không nhìn tài liệu.
> - [ ] Tự thí nghiệm Isolation Level (mở 2 terminal MySQL).
> - [ ] Áp dụng vào entity JPA trong khóa luận Spring Boot.
> - [ ] Đọc thêm: MySQL Official Documentation → InnoDB Locking and Transaction Model.
>
> *"Kiến thức không phải là thứ em đọc được — là thứ em VIẾT được khi không có tài liệu bên cạnh."*
>
> — Chúc em tốt nghiệp xuất sắc và nhận offer từ buổi phỏng vấn đầu tiên. 💪
