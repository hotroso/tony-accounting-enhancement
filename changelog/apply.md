# Hướng dẫn áp dụng & kiểm thử

> ⚠️ Phần mềm liên quan TIỀN BẠC + PHÁP LÝ. Làm đúng thứ tự. KHÔNG bỏ bước kiểm thử.

## Trạng thái trung thực của script này

- Các script được xây dựng từ **phân tích schema thật** (git `12ec8af` gốc, `8cae432` tham khảo).
- Logic SQL đã được **đối chiếu tĩnh** và chứng minh tương đương (xem các ISSUE).
- **CHƯA chạy thực tế** các script `.sql` này trên database (chưa đụng DB thật).
  => Trước khi dùng production, BẮT BUỘC chạy thử trên BẢN COPY và kiểm thử số liệu.

## Chuẩn bị (BẮT BUỘC)

1. **Tắt hoàn toàn phần mềm Tony Accounting** (Firebird embedded khóa file độc quyền).
   - Nếu có Firebird server chạy service: cân nhắc dừng nếu thao tác mức file.
2. **Backup:**
   - Copy file: `copy "D:\001.Sac\DULIEU_PMTONY.TDB" "D:\001.Sac\DULIEU_PMTONY_backup_<ngay>.TDB"`
   - Backup logic: `gbak -b -user SYSDBA -password masterkey "localhost:D:\...\DULIEU_PMTONY.TDB" "D:\...\truoc_toi_uu.fbk"`
3. **Tạo 1 bản COPY để test** (không test trên file thật lần đầu).

## Thứ tự chạy (trên bản COPY trước)

```
cd D:\001.Sac\firebird\changelog
set ISQL="C:\Program Files\Firebird\Firebird_2_5\bin\isql.exe"
set DB=localhost:D:\001.Sac\DULIEU_PMTONY.TDB

%ISQL% -user SYSDBA -password masterkey %DB% -i 01-create-indexes.sql
%ISQL% -user SYSDBA -password masterkey %DB% -i 02-alter-procedures.sql
```
> PHẢI chạy 01 TRƯỚC 02 (procedure có PLAN ép index -> index phải tồn tại trước).

## KIỂM THỬ ĐỐI CHIẾU SỐ LIỆU (quyết định PASS/FAIL)

Mục tiêu: chứng minh số liệu SAU tối ưu == số liệu GỐC (chỉ nhanh hơn).

Cách làm đề xuất (cần 2 bản DB: 1 gốc nguyên bản, 1 đã áp script):
1. Chọn bộ tham số thực tế: 1 company, vài kỳ, vài ngày chốt (đầu kỳ, giữa kỳ, cuối kỳ).
2. Gọi trực tiếp các procedure trên CẢ 2 bản, cùng tham số, so kết quả:
   ```sql
   SELECT * FROM INT_SD_1_TK_DN(<company>, <account>, '<ngay>');
   SELECT * FROM INT_PS_1_TK(<company>, <account>, '<tu_ngay>', '<den_ngay>');
   -- ... va cac proc con lai
   ```
3. Chạy báo cáo "Bảng cân đối" trên cả 2 bản (qua phần mềm) cho cùng kỳ, XUẤT ra và so TỪNG DÒNG.
4. Chú ý đặc biệt:
   - Tài khoản có đối tượng (OBJECT0/OBJECT1): công nợ khách hàng, nhà cung cấp.
   - Tài khoản hàng hóa (IS_GOODS_ACCOUNT) — điều kiện SUSPEND lọc dòng 0.
   - Nếu DN có giao dịch NGOẠI TỆ: soi kỹ chênh lệch do DOUBLE -> NUMERIC(18,4).

**CHỈ áp lên file thật khi: so khớp 100%, không lệch dù 1 đồng.**

## Nếu cần gỡ bỏ (rollback)
```
%ISQL% -user SYSDBA -password masterkey %DB% -i 99-rollback.sql
```
Hoặc đơn giản nhất: phục hồi từ backup đã tạo ở bước chuẩn bị.

## Đo hiệu năng (ghi lại để đối chứng)
- Trước/sau: thời gian chạy "Bảng cân đối" cùng kỳ. Kỳ vọng: từ ~30 phút -> vài chục giây.
- Có thể dùng `SET STATS ON;` trong isql để xem thời gian khi gọi procedure.
