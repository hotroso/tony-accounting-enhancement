# Tony Accounting — Kế hoạch bảo trì & nâng cấp

> File này là bộ nhớ dài hạn của dự án. Mỗi lần quay lại, ĐỌC FILE NÀY TRƯỚC để không quên nhiệm vụ và bối cảnh.

Cập nhật lần cuối: 2026-10-17 (phiên 2)

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

### Giai đoạn 1 — Trích xuất cấu trúc (✅ XONG — bản ĐÃ TỐI ƯU)
- [x] git init tại D:\001.Sac\firebird
- [x] Trích toàn bộ metadata (DDL) từ `DULIEU_PMTONY.TDB` (bản đã tối ưu) -> `schema_full.sql` (3.3 MB, 89.154 dòng).
      Nội dung đã xác minh ĐẦY ĐỦ LOGIC THẬT: 1043 procedure (phần CREATE là stub rỗng theo cơ chế isql,
      logic thật nằm ở 1043 khối ALTER PROCEDURE phía sau — 3625 DECLARE VARIABLE), 147 table, 57 trigger,
      44 index, 91 domain, 2 generator. Đã đọc tận mắt body của INT_CAL_FINANCIAL_BALANCE.
- [x] Backup nhị phân chuẩn bằng gbak: `optimized_metadata.fbk` (6.09 MB, metadata-only `-m`).
      ĐÃ TEST RESTORE thành công -> file hợp lệ, phục hồi 100%. (File .fbk KHÔNG commit git, giữ trên đĩa.)
- [x] git commit lưu cấu trúc bản đã tối ưu làm MỐC SO SÁNH.

> LƯU Ý cơ chế trích: `isql -extract` / gbak sinh ra procedure 2 lần — CREATE (stub `BEGIN SUSPEND; END`
> để phá tham chiếu vòng) rồi ALTER (body thật). Khi đọc file phải nhìn phần ALTER mới thấy logic.

### Giai đoạn 1b — Khi nhận lại DATA GỐC (chưa tối ưu) — CHỜ CHỦ MÁY
- [ ] Chủ máy cung cấp lại file data GỐC (chưa tối ưu).
- [ ] Trích `schema_full.sql` lần nữa từ bản GỐC bằng ĐÚNG lệnh đã dùng (xem mục 8) để điều kiện so sánh đồng nhất.
- [ ] `git diff` giữa bản GỐC và bản ĐÃ TỐI ƯU -> xác định CHÍNH XÁC đã đổi gì (index? viết lại procedure?
      statistics?) khiến Bảng cân đối từ ~30 phút xuống còn vài chục giây.
- [ ] Ghi lại thay đổi đó thành script tái lập được.

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
- 2026-10-17 (phiên 2): Trích xuất XONG bản ĐÃ TỐI ƯU. Xác minh schema_full.sql có đủ logic thật
  (1043 proc có body ở phần ALTER). Tạo + test-restore backup gbak `optimized_metadata.fbk` -> hợp lệ.
  Cập nhật .gitignore (bỏ qua file tạm `_*` và thư mục session). Commit mốc "đã tối ưu".
  Phát hiện: Firebird server (fbserver.exe) đang chạy dạng Windows Service -> trích metadata qua
  `localhost:` ổn, nhưng nếu cần thao tác mức file (copy/gbak offline) thì phải dừng service trước.
  ĐANG CHỜ: data GỐC chưa tối ưu từ chủ máy để đối chiếu diff.

## 8. Lệnh trích xuất CHUẨN (dùng LẠI y hệt cho bản gốc để diff đồng nhất)

Trích DDL đầy đủ (chạy khi app Tony đã tắt):
```
"C:\Program Files\Firebird\Firebird_2_5\bin\isql.exe" -user SYSDBA -password masterkey ^
  -extract "localhost:D:\001.Sac\<FILE_DATA>" -output "D:\001.Sac\firebird\schema_full.sql"
```
Backup metadata nhị phân (test restore để chắc hợp lệ):
```
"C:\Program Files\Firebird\Firebird_2_5\bin\gbak.exe" -b -m -user SYSDBA -password masterkey ^
  "localhost:D:\001.Sac\<FILE_DATA>" "D:\001.Sac\firebird\<ten>.fbk"
```
> Đăng nhập Firebird: SYSDBA / masterkey (mặc định).
