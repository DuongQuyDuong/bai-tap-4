# BÀI NỘP AUTORIDE – THIẾT KẾ LẠI ERD VÀ MYSQL

Tách các khối nội dung dưới đây thành các file tương ứng trước khi đẩy lên GitHub.

---

## autoride_db.sql

```sql
-- AutoRide: chạy trên MySQL 8.0.16+; sử dụng database thực hành
CREATE DATABASE IF NOT EXISTS autoride_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE autoride_db;

-- Tạo schema cũ để có thể thực hành ALTER TABLE trên CSDL mới.
CREATE TABLE IF NOT EXISTS Cars (
 car_id INT AUTO_INCREMENT PRIMARY KEY,
 model_name VARCHAR(100) NOT NULL,
 license_plate VARCHAR(20) NOT NULL UNIQUE
) ENGINE=InnoDB;
CREATE TABLE IF NOT EXISTS Rentals (
 rental_id INT AUTO_INCREMENT PRIMARY KEY,
 car_id INT,
 customer_name VARCHAR(100) NOT NULL,
 rent_date DATETIME NOT NULL,
 return_date DATETIME,
 status VARCHAR(50) DEFAULT 'BOOKED',
 CONSTRAINT fk_rentals_car FOREIGN KEY (car_id) REFERENCES Cars(car_id)
) ENGINE=InnoDB;

-- Chạy phần ALTER này một lần trên schema legacy (nếu đã có cột thì bỏ qua).
ALTER TABLE Rentals
 MODIFY COLUMN car_id INT NOT NULL,
 MODIFY COLUMN status ENUM('BOOKED','ACTIVE','COMPLETED','CANCELLED') NOT NULL DEFAULT 'BOOKED',
 ADD COLUMN security_deposit DECIMAL(15,2) NOT NULL DEFAULT 0.00,
 ADD COLUMN late_fee DECIMAL(15,2) NOT NULL DEFAULT 0.00,
 ADD COLUMN damage_fee DECIMAL(15,2) NOT NULL DEFAULT 0.00,
 ADD CONSTRAINT chk_deposit_nonnegative CHECK (security_deposit >= 0),
 ADD CONSTRAINT chk_late_nonnegative CHECK (late_fee >= 0),
 ADD CONSTRAINT chk_damage_nonnegative CHECK (damage_fee >= 0);

CREATE TABLE IF NOT EXISTS Inspections (
 inspection_id INT AUTO_INCREMENT PRIMARY KEY,
 rental_id INT NOT NULL,
 inspection_date DATETIME NOT NULL,
 damage_description TEXT,
 inspector_name VARCHAR(100) NOT NULL,
 CONSTRAINT fk_inspection_rental FOREIGN KEY (rental_id)
   REFERENCES Rentals(rental_id) ON DELETE RESTRICT
) ENGINE=InnoDB;

-- DML: mô phỏng khách trả trễ 2 ngày, vỡ đèn pha trái.
-- Các mức tiền chỉ mang tính minh họa, không có quy tắc tính phí cố định trong đề.
INSERT INTO Cars(model_name, license_plate) VALUES ('Toyota Vios', '20A-12345');
SET @car_id = LAST_INSERT_ID();
INSERT INTO Rentals(car_id, customer_name, rent_date, return_date, status, security_deposit)
VALUES (@car_id, 'Nguyen Van A', '2026-10-10 08:00:00', '2026-10-12 08:00:00', 'BOOKED', 10000000.00);
SET @rental_id = LAST_INSERT_ID();
UPDATE Rentals SET status = 'ACTIVE' WHERE rental_id = @rental_id;

START TRANSACTION;
INSERT INTO Inspections(rental_id, inspection_date, damage_description, inspector_name)
VALUES (@rental_id, '2026-10-14 08:00:00', 'Vỡ đèn pha trái', 'Tran Thi B');
UPDATE Rentals
SET status = 'COMPLETED', late_fee = 1000000.00, damage_fee = 2000000.00
WHERE rental_id = @rental_id;
COMMIT;

SELECT r.rental_id, r.customer_name, r.status, r.security_deposit,
       r.late_fee, r.damage_fee,
       (r.security_deposit - r.late_fee - r.damage_fee) AS refund_amount,
       i.damage_description, i.inspector_name
FROM Rentals r
JOIN Inspections i ON i.rental_id = r.rental_id
WHERE r.rental_id = @rental_id;
-- Kết quả giả định: 10.000.000 - 1.000.000 - 2.000.000 = 7.000.000 VNĐ.
-- Theo kịch bản đơn giản ở đề: late_fee = 0 => hoàn 8.000.000 VNĐ.

```

---

## er_activity_mapping.md

```markdown
# AutoRide – Đối chiếu Activity Diagram và ERD

## Các khoảng trống dữ liệu

| Điểm chạm nghiệp vụ | Thiếu sót trong Legacy ERD | Hệ quả | Cách khắc phục |
|---|---|---|---|
| Đóng tiền cọc | Không có `security_deposit` | Không biết số phải hoàn | Thêm `DECIMAL(15,2)` |
| Trả xe trễ | Không có `late_fee` | Không ghi nhận phí trễ | Thêm `late_fee` |
| Xe hỏng | Không có `damage_fee` | Không thu phí sửa chữa | Thêm `damage_fee` |
| Kiểm tra xe | Không có bảng `Inspections` | Không lưu hư hại, người kiểm tra, thời gian | Tạo bảng với FK tới `Rentals` |
| Vòng đời hợp đồng | `status VARCHAR` tự do | Có thể nhập trạng thái sai | Đổi sang `ENUM` |

## Phân tích ngắn (dưới 200 từ)

Trong Activity Diagram, bước trả xe có các nhánh: trả đúng/trễ và xe bình thường/hư hỏng. Bảng Rentals cũ không có `damage_fee`, nên không thể tính đúng tiền hoàn cọc khi xe bị hỏng. Cột này cần dùng `DECIMAL`, không dùng `FLOAT`, để tránh sai số biểu diễn tiền tệ. `security_deposit` và `late_fee` cũng cần được lưu để tính `refund = security_deposit - late_fee - damage_fee`. Bảng `Inspections` riêng lưu thời gian, người kiểm tra và mô tả hư hỏng, cho phép một hợp đồng có nhiều lượt kiểm tra nếu nghiệp vụ mở rộng. Khóa ngoại `ON DELETE RESTRICT` ngăn xóa hợp đồng khi còn biên bản. `ENUM` giới hạn trạng thái hợp lệ, còn `CHECK` ngăn ghi phí âm. Những thay đổi này nối các nhánh quy trình với dữ liệu, giúp kế toán đối soát và tránh hoàn cọc sai.

```

---

## ai_prompt_log.md

```markdown
# Nhật ký tương tác AI – AutoRide

> Bản ghi tổng hợp từ yêu cầu trong cuộc trao đổi hiện tại; không khẳng định đã thực hiện kiểm thử MySQL.

1. **Câu hỏi:** Nên dùng `DECIMAL` hay `FLOAT` cho tiền cọc và phí?  
   **Tổng hợp phản hồi:** Dùng `DECIMAL(15,2)` để lưu số tiền thập phân chính xác, tránh sai số nhị phân từ `FLOAT`.
2. **Câu hỏi:** Vì sao tách `Inspections` khỏi `Rentals`?  
   **Tổng hợp phản hồi:** Quan hệ 1–N hỗ trợ nhiều biên bản/lượt kiểm tra cho một hợp đồng; dữ liệu kiểm tra có người lập và ngày riêng, giảm lặp thông tin.
3. **Câu hỏi:** Phép tính hoàn cọc xử lý `NULL` thế nào?  
   **Tổng hợp phản hồi:** Đặt các khoản phí `NOT NULL DEFAULT 0` hoặc dùng `COALESCE(amount,0)` khi tính; không nên để toàn bộ biểu thức thành `NULL`.
4. **Câu hỏi:** Chặn lập biên bản khi hợp đồng `BOOKED` ra sao?  
   **Tổng hợp phản hồi:** Có thể dùng `BEFORE INSERT` trigger kiểm tra trạng thái của `Rentals` rồi `SIGNAL SQLSTATE '45000'` nếu không đủ điều kiện; một `CHECK` đơn giản không kiểm tra trạng thái ở bảng khác.

**Giới hạn:** Kịch bản trả trễ 2 ngày không nêu đơn giá phạt/ngày; ví dụ SQL chọn 500.000 VNĐ/ngày để minh họa, không phải quy tắc đã được cung cấp.

```

---

## Hướng dẫn nộp GitHub

Tạo repository `autoride-database-design`, đưa 3 file trên vào repo, rồi chạy:

```bash
git init
git add .
git commit -m "feat: redesign AutoRide database and map activities"
git branch -M main
git remote add origin https://github.com/USERNAME/autoride-database-design.git
git push -u origin main
```

Thay `USERNAME` bằng tài khoản GitHub. Chạy SQL bằng MySQL 8.0.16+ trên database thực hành, không chạy lại nguyên file hai lần vì ALTER thêm cột sẽ báo trùng; không chạy trên dữ liệu thật khi chưa sao lưu.
