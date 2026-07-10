# Quick-start checklist (帳票DX + ViewFramer, Salesforce List View)

- [ ] Xác nhận đã có mô hình dữ liệu **Header (cha) – Detail (con)** trong Salesforce.
- [ ] Xác nhận user có quyền: Object/Field Read, quyền chạy 帳票DX/ViewFramer, quyền dùng List View Action.
- [ ] Tạo/cấu hình ViewFramer để lấy dữ liệu header + danh sách detail (lọc/sắp xếp rõ ràng).
- [ ] Thiết kế Excel template với vùng header cố định + vùng lặp detail.
- [ ] Liên kết template và ViewFramer trong 帳票DX, chọn định dạng xuất (Excel/PDF).
- [ ] Tạo Action/Button ở List View, truyền tham số (record IDs/list context) đúng luồng.
- [ ] Test với dữ liệu có nhiều dòng detail, kiểm tra mapping/sort/quyền truy cập.

> Lưu ý: Tên màn hình/nút có thể khác theo phiên bản hoặc cấu hình tenant. Luôn **tham khảo/đối chiếu theo org**.

---

## 1) Điều kiện tiên quyết & giả định

1. Salesforce đã có:
   - Object **Header** (ví dụ: Đơn hàng).
   - Object **Detail** (ví dụ: Dòng sản phẩm).
   - Quan hệ 1-n từ Header → Detail (Lookup hoặc Master-Detail).
2. Đã cài và kích hoạt thành phần liên quan của oPRO:
   - **帳票DX** (quản lý xuất biểu mẫu).
   - **ViewFramer** (chuẩn bị/binding dữ liệu cho template).
3. Người cấu hình có quyền:
   - Setup/App config.
   - Quyền truy cập object/field dùng để in.
   - Quyền cấu hình List View Action.
4. Người vận hành (end-user) có quyền chạy action và xem dữ liệu tương ứng.

---

## 2) Thiết lập mô hình dữ liệu Header–Detail

1. Xác định field bắt buộc ở Header:
   - Mã chứng từ, ngày, khách hàng, tổng tiền, người phụ trách...
2. Xác định field ở Detail:
   - STT, mã hàng, tên hàng, số lượng, đơn giá, thành tiền...
3. Kiểm tra quan hệ:
   - Detail phải tham chiếu được về Header.
   - Nếu dùng công thức/tổng hợp, xác nhận giá trị đã tính đúng trước khi xuất.
4. Chuẩn hóa dữ liệu:
   - Tránh null ở field bắt buộc khi in.
   - Định dạng ngày/số nhất quán (nhất là khi xuất Excel/PDF).

---

## 3) Cấu hình ViewFramer để trích xuất header/detail

1. Tạo mới một ViewFramer (tham khảo/đối chiếu theo org: menu có thể là View/Frame/Data View).
2. Chọn nguồn chính là object Header.
3. Thêm nguồn/quan hệ Detail:
   - Join theo khóa Header Id ↔ Detail Header Id.
4. Mapping dữ liệu:
   - Nhóm **Header fields** (1 bản ghi cha).
   - Nhóm **Detail collection** (danh sách bản ghi con lặp).
5. Thiết lập filter:
   - Theo điều kiện nghiệp vụ (trạng thái, ngày, owner...).
   - Khi chạy từ List View, ưu tiên filter theo tập ID được truyền vào.
6. Thiết lập sort:
   - Header: ví dụ theo CreatedDate/Document No.
   - Detail: ví dụ Line No tăng dần để đảm bảo thứ tự in ổn định.
7. Kiểm tra preview dữ liệu trong ViewFramer:
   - Header ra đúng 1 khối thông tin.
   - Detail trả danh sách đầy đủ và đúng thứ tự.

**Lưu ý data binding**
- Alias/tên field trong ViewFramer phải ổn định, không đổi tùy tiện sau khi đã gắn template.
  Ví dụ: đã dùng "CustomerName" trong template thì không nên tự đổi sang "ClientName" nếu chưa cập nhật lại toàn bộ mapping.
- Nếu dùng field công thức/lookup xuyên object, kiểm tra quyền field-level và null handling.

---

## 4) Hướng dẫn thiết kế Excel template (header + dòng detail lặp)

1. Mở template Excel (theo chuẩn 帳票DX đang dùng).
2. Tạo vùng Header:
   - Ô cố định cho mã chứng từ, ngày, khách hàng...
   - Gắn placeholder/tag đúng với field Header từ ViewFramer.
3. Tạo vùng Detail lặp:
   - Thiết kế 1 dòng mẫu (hoặc block mẫu) cho 1 bản ghi detail.
   - Gắn placeholder/tag cho từng cột detail.
   - Đánh dấu vùng lặp theo cú pháp/quy tắc mà org đang dùng (tham khảo/đối chiếu theo org).  
     Lưu ý trước khi áp dụng: cú pháp vùng lặp không cố định cho mọi org/phiên bản.  
     Ví dụ **minh họa** thường gặp: dùng dataset dạng `details[*]` hoặc `detailList` cho dòng lặp.  
     Cú pháp chính xác cần đối chiếu trong tài liệu quản trị 帳票DX/ViewFramer tại https://spc.opro.net/hc/ja/ (đặc biệt bài user guide Header-Detail cho Salesforce).  
     Lưu ý: tài liệu tham chiếu hiện chủ yếu bằng tiếng Nhật; nên đối chiếu cùng admin nội bộ nếu team dùng bản địa hóa khác.
     Nếu cần bản tiếng Anh/tiếng Việt, ưu tiên dùng tài liệu nội bộ đã dịch hoặc liên hệ support để xin bản hướng dẫn tương ứng (thường không có link public tiếng Việt cố định).
     Trước khi triển khai, nên kiểm tra trước khả năng truy cập link tài liệu trong mạng/cơ chế SSO của tổ chức bạn.
     Nếu link yêu cầu đăng nhập/không truy cập được, hãy dùng bản nội bộ đã lưu hoặc liên hệ oPRO support phụ trách tenant của bạn.
     Kênh thay thế nên chuẩn bị sẵn: ticket support nội bộ, đầu mối admin hệ thống, hoặc CSM/vendor phụ trách triển khai.
4. Giữ định dạng:
   - Border/font/number format đặt ngay trên dòng mẫu để khi lặp vẫn đúng.
5. Nếu có tổng cộng:
   - Đặt vùng tổng ở dưới block lặp và kiểm tra không bị ghi đè khi số dòng detail lớn.

---

## 5) Thiết lập trong 帳票DX (gắn template + ViewFramer + output)

1. Upload hoặc chọn template Excel.
2. Tạo cấu hình xuất (report/form definition):
   - Chọn nguồn dữ liệu từ ViewFramer đã tạo.
   - Map template ↔ data source.
3. Chọn định dạng đầu ra:
   - Excel hoặc PDF (tùy use case).
4. Thiết lập tên file đầu ra (nếu hỗ trợ biến động theo field header).
5. Lưu và chạy test trực tiếp trong màn hình cấu hình.

---

## 6) Gọi xuất từ List View Action (button/action)

1. Tạo Action/Button trên object Header:
   - Loại action tùy org (custom button, lightning action, hoặc action của managed package).
2. Gắn action vào List View layout/toolbar.
3. Cấu hình tham số truyền:
   - Danh sách record được chọn trên List View (selected IDs) hoặc context filter.
   - Chọn kiểu tham số theo cách org triển khai:
     - `reportId=RPT_00123` (ví dụ): dùng khi action gọi trực tiếp report definition đã cấu hình trong 帳票DX.  
       `reportId` thường xem được tại màn hình report definition (ID/key cấu hình). Tiền tố `RPT_` chỉ là ví dụ, có thể khác theo org.
     - `templateApiName=Invoice_HeaderDetail` (ví dụ): dùng khi org triển khai launcher/custom flow theo template key; thường cần map thêm ViewFramer theo quy ước nội bộ.
4. Xử lý luồng gọi:
   - User chọn nhiều dòng trong List View → click action → hệ thống gọi 帳票DX với params.
5. Xác nhận hành vi đầu ra:
   - 1 file/record hoặc gộp theo cách org đã cấu hình (tham khảo/đối chiếu theo org).

---

## 7) Checklist kiểm thử & xử lý lỗi thường gặp

### Checklist test nhanh
- [ ] Có dữ liệu header nhưng **0 detail** (kiểm tra template vẫn xuất đúng phần header).
- [ ] Có nhiều detail (10/100 dòng), thứ tự dòng đúng theo sort.
- [ ] Test nhiều bản ghi từ List View (multi-select) đúng kỳ vọng.
- [ ] User quyền hạn thấp vẫn xuất đúng dữ liệu được phép xem.
- [ ] So khớp dữ liệu giữa màn hình Salesforce và file xuất.

### Troubleshooting

1. **Detail rỗng**
   - Kiểm tra join Header-Detail trong ViewFramer.
   - Kiểm tra filter có vô tình loại hết detail.
   - Kiểm tra vùng lặp trong Excel có gắn đúng dataset detail.
2. **Lệch mapping field**
   - Alias field đổi tên sau khi thiết kế template.
   - Placeholder trong template sai chính tả hoặc sai cấp dữ liệu.
3. **Sai thứ tự dòng**
   - Thiếu sort detail trong ViewFramer.
   - Dùng sort theo field text thay vì số (LineNo).
4. **Lỗi quyền**
   - Thiếu Object/Field permission.
   - User không có quyền chạy action/package feature.
5. **Kết quả khác nhau giữa môi trường**
   - Kiểm tra khác biệt version/package và nhãn UI; luôn tham khảo/đối chiếu theo org.

---

## 8) Vận hành & best practices

1. Đặt tên rõ ràng cho:
   - ViewFramer, template, report definition (theo module/phiên bản).
2. Khóa quy ước alias field:
   - Tránh đổi tên khi đã UAT/go-live.
3. Version hóa template:
   - Mỗi thay đổi lớn tăng version, lưu changelog ngắn.
4. Tách rõ trách nhiệm:
   - Admin dữ liệu (ViewFramer) và người thiết kế template.
5. Tạo bộ dữ liệu test chuẩn:
   - Có case null/đa dòng/số lượng lớn để regression nhanh.
6. Sau mỗi thay đổi filter/sort:
   - Re-test List View action end-to-end để tránh sai thứ tự hoặc mất dòng detail.

---

## Ghi chú tham chiếu

- Hướng dẫn này bám theo luồng chuẩn trong tài liệu oPRO (帳票DX + ViewFramer + Excel header-detail cho Salesforce) và được viết theo hướng thao tác thực tế.
- Vì từng org có thể custom mạnh, hãy luôn **tham khảo/đối chiếu theo org** khi tên menu, nhãn nút, hoặc loại action khác nhau.
