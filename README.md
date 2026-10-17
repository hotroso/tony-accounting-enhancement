# Tony Accounting — Enhancement & Maintenance

Dự án bảo trì và tối ưu phần mềm kế toán **Tony Accounting** (Công ty TNHH Phần mềm T.O.N.Y).
Nhà cung cấp đã ngừng hoạt động; dự án này ghi lại việc **tối ưu hiệu năng** và **rà soát cập nhật
pháp lý** cho những ai còn đang sử dụng phần mềm.

> ⚠️ **PHẦN MỀM LIÊN QUAN TIỀN BẠC VÀ PHÁP LÝ (kế toán, thuế, báo cáo tài chính).**
> Mọi thay đổi database phải được backup đầy đủ và kiểm thử đối chiếu số liệu trước khi dùng thật.

## Bối cảnh kỹ thuật

- Ứng dụng desktop .NET Framework 2.0 (Windows Forms + DevExpress), đã biên dịch — KHÔNG có mã nguồn DLL.
- CSDL: **Firebird embedded 2.5** (file `.TDB`).
- **Phần lớn logic nghiệp vụ nằm trong DATABASE** (1043 stored procedure, 147 bảng, 57 trigger).
  => Có thể tối ưu/cập nhật qua SQL mà không cần mã nguồn DLL.

## Nội dung repo

| Đường dẫn | Mô tả |
|---|---|
| `PLAN.md` | Kế hoạch tổng + nhật ký tiến độ (đọc trước) |
| `schema_full.sql` | Schema đầy đủ (baseline, trích từ bản gốc) |
| `CAP_NHAT_PHAP_LY.md` | Rà soát thông tư/nghị định 2018→2026 & khả năng xử lý |
| `DIEU_TRA_TOI_UU.md` | Điều tra chuyên sâu 6 procedure tối ưu (chứng minh tương đương) |
| `changelog/` | Script update tối ưu + tài liệu chứng minh từng thay đổi + rollback |

## Tối ưu "Bảng cân đối" (tóm tắt)

Từ ~30 phút xuống còn vài chục giây, bằng:
1. **Thêm 10 index tổ hợp** trên các bảng giao dịch lớn (BALANCES, CREDIT_DEBIT, DOCUMENTS, PERIODS).
2. **Viết lại 6 procedure** tính phát sinh & số dư (gộp quét, JOIN phẳng thay EXISTS lồng, chốt sớm).

Chi tiết + cách áp dụng an toàn: xem `changelog/` (đọc `changelog/README.md` và `changelog/apply.md`).

## ⚠️ Trạng thái

- Phân tích tĩnh: 6/6 procedure tương đương số liệu với bản gốc.
- **Kiểm thử số thực: CHƯA chạy.** Bắt buộc đối chiếu số liệu trước khi dùng production.

## Bản quyền / Lưu ý pháp lý

Phần mềm Tony Accounting và thiết kế cơ sở dữ liệu gốc thuộc bản quyền của **Công ty TNHH Phần mềm
T.O.N.Y**. Repo này phục vụ mục đích **bảo trì và chia sẻ kỹ thuật phi thương mại** cho người đang
sử dụng phần mềm. File `schema_full.sql` là cấu trúc được trích xuất từ sản phẩm gốc — cân nhắc kỹ
quyền sở hữu trí tuệ trước khi phân phối lại công khai. Phần đóng góp mới (changelog, tài liệu phân
tích) là của người bảo trì.

Đây là tài liệu kỹ thuật, KHÔNG phải tư vấn pháp lý/thuế. Số liệu và hiệu lực văn bản cần kế toán
của đơn vị xác nhận với cơ quan thuế.
