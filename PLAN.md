# Tony Accounting — Kế hoạch bảo trì & nâng cấp

> File này là bộ nhớ dài hạn của dự án. Mỗi lần quay lại, ĐỌC FILE NÀY TRƯỚC để không quên nhiệm vụ và bối cảnh.

Cập nhật lần cuối: 2026-10-17

---

## 1. Bối cảnh

- Phần mềm kế toán **Tony Accounting** của Công ty TNHH Phần mềm T.O.N.Y. Nhà cung cấp **đã phá sản / không liên hệ được**. Không có mã nguồn (source code). Công ty vợ của chủ máy đang dùng thật.
- Thư mục cài đặt: `C:\Program Files (x86)\Tony Software\Tony Accounting`
- Kiến trúc:
  - Giao diện: .NET Framework 2.0 (runtime v2.0.50727), Windows Forms + DevExpress v8.2 / v11.2 (đã biên dịch, KHÔNG sửa được nếu thiếu source).
  - Framework nghiệp vụ: CSLA (`Csla.DLL`).
  - CSDL: **Firebird embedded 2.5.8** (`fbembed.dll` = WI-V2.5.8.27089).
  - **QUAN TRỌNG:** Phần lớn logic nghiệp vụ nằm trong CSDL (1044 stored procedures, 146 tables, 57 triggers — xem `Creator.cql`), KHÔNG nằm trong DLL. => Có thể cập nhật/tối ưu nghiệp vụ qua SQL (procedure/trigger/index) mà KHÔNG cần source code DLL. Đây là con đường khả thi.
  - Cơ chế update gốc của phần mềm: các file `.uql` (SQL migration) trong thư mục `Update\`, mới nhất ~2018. `Creator.cql` = script tạo schema (procedure trong đó là STUB rỗng `BEGIN SUSPEND; END`; logic thật nằm trong file DB đang chạy).

## 2. Môi trường & công cụ (đã xác minh)

- Firebird 2.5 đã cài: `C:\Program Files\Firebird\Firebird_2_5\bin\` (isql.exe, gbak.exe, gstat.exe, gfix.exe...). Tương thích ODS 2.5 với DB của phần mềm.
- Thư mục làm việc (code + git): `D:\001.Sac\firebird`
- File dữ liệu:
  - `D:\001.Sac\DULIEU_PMTONY.TDB` — bản ĐÃ tối ưu (optimized) một lần trước đó. `.TDB` = file Firebird đổi đuôi.
  - Chủ máy sẽ cung cấp lại **DATA GỐC** (chưa tối ưu) để tái hiện lỗi "Bảng cân đối chạy chậm ~30 phút".
- Git: dùng để lưu lịch sử cấu trúc DB và mọi thay đổi.

## 3. Nhiệm vụ của chủ máy (yêu cầu gốc)

1. Xem khả năng cập nhật tính năng / cập nhật theo thông tư, nghị định mới (từ 2017 → nay).
2. Tra cứu thông tư/nghị định mới trên web (KHÔNG cần dựng MCP riêng — web search built-in đã đủ).
3. **Tối ưu chức năng "Bảng cân đối"** — hiện chạy chậm ~30 phút trên data gốc. Cần xem giải pháp TRƯỚC khi sửa.

## 4. Các procedure liên quan "Bảng cân đối" (trong Creator.cql, bản stub)

- `RPT_BANG_CAN_DOI_TAI_KHOAN` (+ biến thể `_ALL`, `_ALLK`, `_C1`, `_C1K`, `_C2`, `_C2K`, `_CON`, `_CONK`)
- `INT_BANG_CAN_DOI_TAI_KHOAN_C`
- `INT_CAL_FINANCIAL_BALANCE`  <- nghi là phần tính toán nặng nhất
- Nhóm BALANCE khác: `CMD_GET_BALANCE_ACCOUNT`, `COPY_BALANCES`, `FINANCIAL_BALANCE_S_EC`, `ENTRY_BALANCE_S_RS`, `DOCUMENT_BALANCE_S_RS`...
> Lưu ý: logic THẬT phải trích từ file DB, KHÔNG dùng bản stub trong Creator.cql.

## 5. Lộ trình (giai đoạn)

### Giai đoạn 0 — An toàn (BẮT BUỘC trước mọi can thiệp)
- [ ] Phần mềm phải TẮT khi thao tác DB (Firebird embedded khóa file độc quyền).
- [ ] Backup file DB (copy) + backup logic (gbak) trước khi sửa bất cứ gì.

### Giai đoạn 1 — Trích xuất cấu trúc (ĐANG LÀM)
- [ ] git init tại D:\001.Sac\firebird
- [ ] Trích toàn bộ metadata (DDL): tables, domains, procedures (logic thật), triggers, indexes, views, generators, constraints -> file .sql
- [ ] git commit lưu cấu trúc

### Giai đoạn 2 — Chẩn đoán "Bảng cân đối" chậm (chờ data gốc)
- [ ] Nhận lại data GỐC từ chủ máy.
- [ ] Trích logic thật của các RPT_BANG_CAN_DOI_* và INT_CAL_FINANCIAL_BALANCE.
- [ ] Chạy PLAN / phân tích: thiếu index? cursor lồng? quét toàn bảng? thống kê lỗi thời?
- [ ] BÁO CÁO nguyên nhân + phương án cho chủ máy, xin duyệt TRƯỚC khi sửa.

### Giai đoạn 3 — Tối ưu
- [ ] Áp index / viết lại procedure / cập nhật statistics, sau khi backup.
- [ ] Đo lại thời gian chạy, so sánh trước/sau.

### Giai đoạn 4 — Cập nhật thông tư/nghị định
- [ ] Lập danh mục văn bản 2017→nay (TT133/2016, TT200/2014 sửa đổi, Nghị định 123/2020 + TT78/2021 hóa đơn điện tử, mẫu BCTC...).
- [ ] Đối chiếu khả năng: sửa qua SP / sửa report .repx / phải làm ngoài (HTKK) / không làm được.

## 6. Nguyên tắc an toàn
- LUÔN backup trước khi can thiệp DB.
- LUÔN thao tác khi phần mềm đã tắt.
- Mọi thay đổi SQL lưu thành script có thể revert, commit vào git.
- Không sửa DLL đã biên dịch.
- Không tự ý sửa file quyền của Kiro (~/.kiro/settings/permissions.yaml) — chủ máy tự sửa.

## 7. Nhật ký tiến độ
- 2026-10-17: Khảo sát xong kiến trúc. Xác định logic nằm trong DB (1044 SP). Tìm thấy Firebird 2.5 tools. Lập plan, git init, chuẩn bị trích cấu trúc từ bản data đã tối ưu.
