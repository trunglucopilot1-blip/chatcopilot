# Hướng dẫn Xuất Báo Cáo Header–Detail (帳票) dạng PDF bằng ViewFramer từ Salesforce List View Action

> **Nguồn tham khảo:** [spc.opro.net/hc/ja](https://spc.opro.net/hc/ja/)  
> **Phiên bản tài liệu:** 1.0  
> **Ngôn ngữ:** Tiếng Việt  

---

## ✅ Checklist Khởi Động Nhanh

Trước khi bắt đầu, hãy xác nhận đã hoàn thành các mục sau:

- [ ] Đã cài đặt package **帳票DX** (SPC) trên Salesforce org
- [ ] Đã có quyền System Administrator hoặc quyền tương đương (Permission Set của 帳票DX)
- [ ] Đã xác định object **Header** (cha) và object **Detail** (con) trong Salesforce
- [ ] Đã có quan hệ Lookup hoặc Master-Detail giữa Header và Detail
- [ ] Đã tạo **ViewFramer view** nối dữ liệu Header–Detail
- [ ] Đã thiết kế **template Excel/Word** với vùng header và vùng lặp detail
- [ ] Đã upload template và liên kết với ViewFramer trong 帳票DX
- [ ] Đã cấu hình định dạng xuất là **PDF**
- [ ] Đã tạo **List View Button** trên Salesforce và liên kết với 帳票DX action
- [ ] Đã chạy test end-to-end thành công

---

## 1. Điều Kiện Tiên Quyết (Prerequisites)

### 1.1 Quyền Salesforce

| Yêu cầu | Ghi chú |
|---------|---------|
| Profile: System Administrator hoặc Custom Profile có đủ quyền | Tối thiểu cần Create/Read/Edit trên object liên quan |
| Permission Set 帳票DX (do OPRO cung cấp) | Cần gán cho user trước khi sử dụng |
| Quyền "Customize Application" | Để tạo Button/Action trên List View |
| Quyền "Modify All Data" hoặc quyền Read trên các object Header/Detail | Tùy theo phạm vi bản ghi cần xuất |

### 1.2 Thành Phần App/Package Cần Có

- **帳票DX (SPC package):** Package chính, bao gồm ViewFramer engine, 帳票DX Designer/Worker.
- **ViewFramer:** Tính năng trong 帳票DX dùng để định nghĩa "view" dữ liệu từ nhiều object/app.  
  > **Lưu ý:** ViewFramer là **trung tâm xử lý dữ liệu** cho báo cáo Header–Detail. Không có ViewFramer, không thể xử lý dữ liệu đa tầng (nhiều dòng detail) đúng cách.
- **XAデザイナー (XA Designer):** Công cụ thiết kế template (Excel/Word) tích hợp với 帳票DX.  
  *(Tên giao diện có thể khác nhau tùy phiên bản/org — tham khảo/đối chiếu theo org)*

### 1.3 Các Object và Field Cần Chuẩn Bị

**Object Header (Cha)** — Ví dụ: `Opportunity` (Cơ hội bán hàng) hoặc custom object `Order__c`

| Field | Loại | Vai trò |
|-------|------|---------|
| `Name` | Text | Tên đơn hàng |
| `Account__c` | Lookup(Account) | Khách hàng |
| `CloseDate` | Date | Ngày chốt |
| `TotalAmount__c` | Currency | Tổng tiền |

**Object Detail (Con)** — Ví dụ: `OpportunityLineItem` hoặc custom object `OrderItem__c`

| Field | Loại | Vai trò |
|-------|------|---------|
| `Order__c` | Master-Detail / Lookup → Header | Quan hệ cha–con |
| `ProductName__c` | Text | Tên sản phẩm |
| `Quantity__c` | Number | Số lượng |
| `UnitPrice__c` | Currency | Đơn giá |
| `LineTotal__c` | Formula(Currency) | Thành tiền |

### 1.4 Giả Định

- Mỗi record Header có thể có **nhiều record Detail** (quan hệ 1:N).
- Người dùng sẽ **chọn một hoặc nhiều record** từ List View của object Header, sau đó click button để xuất PDF.
- Template được thiết kế dưới dạng **Excel (.xlsx)** và 帳票DX sẽ render thành **PDF**.

---

## 2. Mô Hình Dữ Liệu Header–Detail trong Salesforce cho ViewFramer

### 2.1 Cấu Trúc Quan Hệ Cha–Con

```
[Header Object]  ──1:N──>  [Detail Object]
     (Cha)                      (Con)
   Order__c               OrderItem__c
       |                        |
   Id (PK)  <──  Order__c (FK/Lookup/MD)
```

**ViewFramer hoạt động theo cơ chế:**
- Lấy dữ liệu từ object Header (1 record) làm **vùng thông tin tĩnh**.
- Lấy danh sách Detail (nhiều records) liên quan đến Header đó làm **vùng lặp**.
- Join hai tập dữ liệu theo key trường quan hệ (`Order__c` trên Detail trỏ tới `Id` của Header).

### 2.2 Các Loại Quan Hệ Được Hỗ Trợ

| Loại quan hệ | Salesforce Field Type | Ghi chú |
|-------------|----------------------|---------|
| Master-Detail | Master-Detail Field | Khuyến nghị — cascade delete, rollup summary |
| Lookup | Lookup Field | Linh hoạt hơn nhưng cần cấu hình join tay trong ViewFramer |

> ⚠️ **Lưu ý:** Với quan hệ Lookup, bạn phải chỉ định rõ trường join trong ViewFramer (xem Bước 4).

---

## 3. Tạo và Cấu Hình ViewFramer

> ViewFramer là **thành phần cốt lõi** của toàn bộ quy trình. Cấu hình ViewFramer quyết định dữ liệu nào được đổ vào template và theo thứ tự nào.

### Bước 3.1: Truy Cập ViewFramer

1. Vào **App Launcher** (lưới ô vuông góc trên trái Salesforce).
2. Tìm và mở ứng dụng **「帳票DX」** (hoặc tên hiển thị tùy theo cấu hình org — *tham khảo/đối chiếu theo org*).
3. Trong menu, chọn **「ViewFramer」** hoặc tab **「ビューフレーマー」**.  
   *(Tên tab/menu có thể khác nhau theo phiên bản — tham khảo/đối chiếu theo org)*

### Bước 3.2: Tạo Mới ViewFramer View

1. Click **「新規作成」** (Tạo mới) hoặc nút **「+」**.
2. Điền thông tin cơ bản:
   - **View 名 (Tên View):** Ví dụ `注文帳票_ヘッダー明細` (Đặt tên dễ nhận diện, không dấu cách).
   - **対象オブジェクト (Object đích):** Chọn object **Header** (ví dụ: `Order__c`).
   - **説明 (Mô tả):** Tóm tắt mục đích view.
3. Click **「保存」** (Lưu) để tạo view.

### Bước 3.3: Thêm Object Detail (Join / Relationship)

1. Trong màn hình chi tiết View vừa tạo, tìm mục **「関連オブジェクト」** (Object liên quan) hoặc **「結合設定」** (Cấu hình Join).  
   *(Nhãn UI có thể khác — tham khảo/đối chiếu theo org)*
2. Click **「追加」** (Thêm) để thêm object con:
   - **Object:** Chọn `OrderItem__c` (object Detail).
   - **結合フィールド (Join Field):** Chọn trường `Order__c` trên Detail — trỏ tới `Id` của Header.
   - **結合タイプ (Join Type):** Chọn **Left Join** (giữ Header ngay cả khi không có Detail) hoặc **Inner Join** (chỉ lấy Header có Detail).
3. Click **「保存」** để xác nhận cấu hình join.

> 💡 **Mẹo:** Nếu cần join thêm object khác (ví dụ: Account), lặp lại bước này với object tương ứng.

### Bước 3.4: Chọn Field và Đặt Tên Alias (Field Mapping)

1. Trong tab **「フィールド設定」** (Cấu hình Field) của View, chọn các field cần xuất:

**Field từ Header:**

| API Name (Salesforce) | Alias trong ViewFramer | Hiển thị trong template |
|----------------------|------------------------|------------------------|
| `Name` | `header_name` | Tên đơn hàng |
| `Account__r.Name` | `customer_name` | Tên khách hàng |
| `CloseDate` | `close_date` | Ngày chốt |
| `TotalAmount__c` | `total_amount` | Tổng tiền |

**Field từ Detail:**

| API Name (Salesforce) | Alias trong ViewFramer | Hiển thị trong template |
|----------------------|------------------------|------------------------|
| `ProductName__c` | `item_product` | Tên sản phẩm |
| `Quantity__c` | `item_qty` | Số lượng |
| `UnitPrice__c` | `item_price` | Đơn giá |
| `LineTotal__c` | `item_total` | Thành tiền |

> ⚠️ **Quan trọng:** Alias (tên alias) trong ViewFramer **phải khớp chính xác** với placeholder/tag trong template Excel/Word. Sai một ký tự sẽ dẫn đến field trống trong PDF.

### Bước 3.5: Cấu Hình Filter (Điều Kiện Lọc)

1. Chuyển sang tab **「フィルター」** (Filter).
2. Thêm điều kiện lọc nếu cần (ví dụ: chỉ lấy Detail có `Status__c = 'Active'`).
3. Để xuất từ List View theo ID được chọn, đặt filter:
   - **Field:** `Id` (của object Header)
   - **Operator:** `IN` hoặc `=`
   - **Value:** `{RECORD_IDS}` *(placeholder — tên biến thực tế tùy theo cấu hình 帳票DX/org, tham khảo tài liệu chính thức của OPRO)*
4. Click **「保存」**.

### Bước 3.6: Cấu Hình Sort (Thứ Tự Sắp Xếp)

1. Chuyển sang tab **「ソート」** (Sort).
2. Thêm điều kiện sắp xếp cho Detail records:
   - **Field:** `LineNumber__c` (hoặc trường thứ tự dòng).
   - **Thứ tự:** Tăng dần (昇順 / ASC).
3. Click **「保存」**.

> ⚠️ **Lưu ý:** Nếu không cấu hình Sort, thứ tự dòng detail trong PDF **có thể không ổn định** (Salesforce không đảm bảo thứ tự mặc định).

### Bước 3.7: Xem Trước Dữ Liệu ViewFramer

1. Trong màn hình View, click **「プレビュー」** (Xem trước) hoặc **「データ確認」** (Kiểm tra dữ liệu).
2. Chọn một record Header để test.
3. Xác nhận:
   - Dữ liệu header hiển thị đúng.
   - Danh sách detail hiển thị đúng số dòng và đúng thứ tự.
   - Alias field không bị null/trống bất thường.

---

## 4. Thiết Kế Template Excel cho PDF Output

### Bước 4.1: Tải và Mở XA Designer (hoặc Template Editor)

1. Tải về XA Designer từ trang OPRO hoặc qua App Launcher trong Salesforce.  
   *(Tên và cách truy cập có thể khác — tham khảo/đối chiếu theo org)*
2. Tạo file Excel mới hoặc mở template hiện có.

### Bước 4.2: Thiết Kế Vùng Header

Đặt thông tin tĩnh của Header ở phần đầu sheet:

```
┌─────────────────────────────────────────┐
│  PHIẾU ĐẶT HÀNG                         │
│  Số đơn:  {{header_name}}               │
│  Khách:   {{customer_name}}             │
│  Ngày:    {{close_date}}                │
│  Tổng:    {{total_amount}}              │
└─────────────────────────────────────────┘
```

**Cú pháp placeholder:** `{{tên_alias}}` — tên alias phải **giống hệt** alias đã đặt trong ViewFramer (Bước 3.4).  
*(Cú pháp tag thực tế có thể là `{tên_alias}` hoặc `[tên_alias]` — tham khảo/đối chiếu theo phiên bản 帳票DX)*

### Bước 4.3: Thiết Kế Vùng Lặp Detail (Repeat Block)

1. Tạo một hàng **mẫu dòng detail** trong bảng:

```
┌───┬──────────────────┬──────┬──────────┬──────────┐
│ # │ Tên sản phẩm     │ SL   │ Đơn giá  │ Thành tiền│
├───┼──────────────────┼──────┼──────────┼──────────┤
│ 1 │ {{item_product}} │{{item_qty}}│{{item_price}}│{{item_total}}│
└───┴──────────────────┴──────┴──────────┴──────────┘
```

2. Đánh dấu hàng này là **vùng lặp (Repeat Row)** trong XA Designer:
   - Chọn toàn bộ hàng mẫu.
   - Click **「繰り返し設定」** (Cấu hình lặp) hoặc Insert Repeat.  
     *(Tên chức năng có thể khác — tham khảo/đối chiếu theo org)*
   - Liên kết vùng lặp với **object Detail** trong ViewFramer.
3. Số hàng sẽ **tự động mở rộng** theo số lượng detail records khi render.

### Bước 4.4: Lưu Ý Phân Trang (Pagination)

- Nếu số dòng detail vượt quá 1 trang A4, 帳票DX sẽ **tự động ngắt trang**.
- Để lặp lại tiêu đề bảng ở đầu mỗi trang mới, sử dụng chức năng **「ページヘッダー」** (Page Header) trong template.  
  *(Tham khảo/đối chiếu theo org)*
- Kiểm tra margin/lề trang để tránh nội dung bị cắt.

### Bước 4.5: Lưu Template

1. Lưu file Excel với tên rõ ràng, ví dụ: `order_report_template.xlsx`.
2. **Không** thay đổi tên alias sau khi đã upload vào 帳票DX (sẽ cần upload lại nếu thay đổi).

---

## 5. Cấu Hình 帳票DX — Liên Kết Template + ViewFramer, Thiết Lập PDF

### Bước 5.1: Upload Template vào 帳票DX

1. Vào 帳票DX trong Salesforce (App Launcher → 帳票DX).
2. Chọn tab **「帳票設定」** (Cấu hình Biểu mẫu) hoặc **「テンプレート管理」** (Quản lý Template).  
   *(Tên tab có thể khác — tham khảo/đối chiếu theo org)*
3. Click **「新規登録」** (Đăng ký mới) hoặc **「アップロード」** (Upload).
4. Điền:
   - **帳票名 (Tên Biểu mẫu):** Ví dụ `注文帳票PDF`.
   - **ファイル (File):** Chọn file `order_report_template.xlsx` đã thiết kế.
5. Click **「保存」**.

### Bước 5.2: Liên Kết ViewFramer với Template

1. Trong màn hình chi tiết Biểu mẫu vừa tạo (hoặc tạo **「出力設定」** — Output Settings mới):
2. Tìm mục **「ViewFramer設定」** hoặc **「データソース」** (Data Source).
3. Chọn ViewFramer View đã tạo ở Bước 3 (ví dụ: `注文帳票_ヘッダー明細`).
4. Xác nhận mapping giữa alias ViewFramer và tag trong template (hệ thống có thể tự detect).

### Bước 5.3: Thiết Lập Định Dạng Xuất PDF

1. Trong **「出力設定」** (Output Settings), tìm mục **「出力形式」** (Output Format).
2. Chọn **「PDF」**.
3. Cấu hình thêm nếu cần:
   - **用紙サイズ (Khổ giấy):** A4 (phổ biến).
   - **向き (Hướng giấy):** Dọc (縦) hoặc Ngang (横).
   - **パスワード (Mật khẩu PDF):** Tùy chọn bảo mật.
4. Click **「保存」**.

### Bước 5.4: Cấu Hình Action Xuất từ List View

1. Trong 帳票DX, tìm tab **「アクション設定」** (Action Settings) hoặc **「ボタン設定」** (Button Settings).
2. Tạo action mới:
   - **アクション名 (Tên Action):** Ví dụ `注文PDF出力`.
   - **対象オブジェクト (Object đích):** Object Header (`Order__c`).
   - **対象帳票 (Biểu mẫu):** Chọn `注文帳票PDF` đã tạo.
   - **トリガー (Trigger):** Chọn **「リストビュー」** (List View) — xuất từ danh sách.
   - **選択方法 (Cách chọn):** Single record hoặc Multi-record tùy yêu cầu.
3. Click **「保存」**.

> 💡 **Lưu ý Multi-record:** Khi người dùng chọn nhiều record từ List View, 帳票DX sẽ truyền danh sách ID vào ViewFramer. ViewFramer xử lý từng record Header và gộp detail tương ứng — kết quả là file PDF nhiều trang (1 biểu mẫu/Header) hoặc 1 PDF gộp tùy cấu hình.

---

## 6. Cấu Hình List View Button trên Salesforce

### Bước 6.1: Tạo Button trên Object Header

1. Vào **Setup** (Thiết lập) → **Object Manager** → Chọn object Header (ví dụ: `Order__c`).
2. Chọn **「Buttons, Links, and Actions」** (Nút, Liên kết và Hành động).
3. Click **「New Button or Link」** (Nút hoặc Liên kết Mới).
4. Cấu hình:
   - **Label (Nhãn):** `PDF出力` hoặc `Xuất PDF Đơn Hàng`.
   - **Name (Tên API):** `Export_PDF_Order` *(không dấu, không khoảng trắng)*.
   - **Display Type:** Chọn **「List Button」** *(bắt buộc để hiện ở List View)*.
   - **Behavior:** Chọn **「Execute JavaScript」** hoặc **「Display in existing window without sidebar or header」** tùy theo cách 帳票DX cung cấp.  
     *(Tham khảo/đối chiếu theo phiên bản — một số phiên bản 帳票DX dùng URL-based action, một số dùng LWC/VF page)*
   - **Content Source:** Nhập URL hoặc đoạn code do 帳票DX cung cấp.  
     Ví dụ URL mẫu (tham khảo — *đối chiếu theo org*):
     ```
     /apex/SPC__ExportPage?actionId={ACTION_ID}&recordIds={!GETRECORDIDS(Order__c)}
     ```
     - `{ACTION_ID}`: ID của action đã tạo trong 帳票DX (Bước 5.4).
     - `{!GETRECORDIDS(Order__c)}`: Salesforce merge field lấy danh sách ID được chọn.
5. Click **「Save」**.

### Bước 6.2: Thêm Button vào List View Layout

1. Từ **Setup** → **Object Manager** → Object Header → **Search Layouts** (Layout Tìm kiếm).
2. Click **「Edit」** bên cạnh **「List View」**.
3. Trong mục **「Custom Buttons」** (Nút tùy chỉnh), kéo `Export_PDF_Order` sang cột **「Selected Buttons」**.
4. Click **「Save」**.

### Bước 6.3: Hướng Dẫn Người Dùng Cuối (End User)

1. Mở List View của object Header (ví dụ: danh sách Đơn hàng).
2. **Chọn một hoặc nhiều record** bằng cách tick checkbox bên trái.
3. Click nút **「PDF出力」** (Xuất PDF) xuất hiện ở thanh toolbar trên List View.
4. 帳票DX sẽ xử lý và tự động tải xuống file PDF (hoặc mở trong tab mới tùy cấu hình).

> 💡 **Lưu ý:** Nếu nút không hiển thị, kiểm tra lại:  
> - Button đã được thêm vào Search Layout chưa.  
> - User có Permission Set 帳票DX chưa.  
> - List View có đang ở chế độ hiển thị đúng chưa.

---

## 7. Luồng Truyền Tham Số và Cơ Chế Hoạt Động

```
[User chọn records trên List View]
        │
        ▼
[Salesforce gọi URL/VF Page với recordIds={ID1,ID2,...}]
        │
        ▼
[帳票DX nhận recordIds → gọi ViewFramer với điều kiện Id IN (ID1, ID2,...)]
        │
        ▼
[ViewFramer query Header records + join Detail records]
        │
        ▼
[ViewFramer trả dữ liệu đã join về 帳票DX]
        │
        ▼
[帳票DX bind dữ liệu vào template Excel → render PDF]
        │
        ▼
[PDF được download về máy user / lưu vào Salesforce Files]
```

---

## 8. Quy Trình Kiểm Thử End-to-End

### Bước 8.1: Test ViewFramer Độc Lập

1. Trong ViewFramer, dùng chức năng **「プレビュー」** (Preview).
2. Nhập ID của record Header cần test.
3. Xác nhận:
   - ✅ Dữ liệu Header hiển thị đúng (name, ngày, tổng tiền...).
   - ✅ Danh sách Detail hiển thị đủ số dòng và đúng thứ tự.
   - ✅ Không có field alias nào bị null bất ngờ.

### Bước 8.2: Test Template Độc Lập

1. Trong 帳票DX, dùng chức năng xuất thử với **「テスト出力」** (Test Output).
2. Chọn record Header.
3. Kiểm tra file PDF đầu ra:
   - ✅ Thông tin header đúng, đủ.
   - ✅ Các dòng detail hiển thị đúng, không bị thiếu dòng.
   - ✅ Phân trang đúng, không bị cắt nội dung.
   - ✅ Format số, ngày tháng hiển thị đúng định dạng.

### Bước 8.3: Test từ List View Button

1. Vào List View của object Header.
2. Chọn **1 record** → Click nút `PDF出力` → Kiểm tra PDF output.
3. Chọn **nhiều record** (2–3 records) → Click nút → Kiểm tra:
   - ✅ PDF chứa đúng số biểu mẫu tương ứng số record được chọn.
   - ✅ Mỗi biểu mẫu có detail của riêng record đó (không bị lẫn dữ liệu).
4. Test với record Header **không có Detail** → Kiểm tra xử lý trường hợp này.

### Bước 8.4: Test Phân Quyền

1. Đăng nhập bằng user thông thường (không phải Admin).
2. Thực hiện xuất PDF → Xác nhận không có lỗi quyền truy cập.
3. Thử với record ngoài phạm vi quyền của user → Xác nhận hệ thống xử lý đúng.

---

## 9. Xử Lý Sự Cố (Troubleshooting)

### 9.1 ❌ Dòng Detail Bị Trống / Thiếu

| Nguyên nhân | Giải pháp |
|-------------|-----------|
| Cấu hình Join sai trong ViewFramer | Kiểm tra lại Join Field — phải khớp với trường quan hệ trong Salesforce |
| Record Detail không thỏa điều kiện Filter | Kiểm tra lại Filter trong ViewFramer, thử xóa Filter để test |
| User không có quyền đọc object Detail | Cấp quyền Read trên object/field Detail cho Profile/Permission Set |
| Object Detail không có record liên quan | Đây là trường hợp bình thường nếu record Header chưa có Detail |

### 9.2 ❌ Tag/Placeholder Không Được Thay Thế (Hiển thị `{{header_name}}` thô)

| Nguyên nhân | Giải pháp |
|-------------|-----------|
| Alias trong ViewFramer không khớp với tag trong template | Đối chiếu từng alias — phân biệt chữ hoa/thường, ký tự đặc biệt |
| Sai cú pháp tag (dùng `{{}}` nhưng cần `{}` hoặc `[]`) | Kiểm tra cú pháp tag theo tài liệu OPRO cho phiên bản đang dùng |
| Template chưa được liên kết với đúng ViewFramer | Vào Output Settings → kiểm tra Data Source |
| Field không được chọn trong ViewFramer Field Settings | Thêm field vào danh sách field của ViewFramer |

### 9.3 ❌ Thứ Tự Dòng Detail Không Đúng

| Nguyên nhân | Giải pháp |
|-------------|-----------|
| Chưa cấu hình Sort trong ViewFramer | Thêm Sort điều kiện trong tab Sort của ViewFramer |
| Sort được cấu hình nhưng field sort không có giá trị | Kiểm tra dữ liệu thực tế trong Salesforce |

### 9.4 ❌ Lỗi Quyền / Permission

| Nguyên nhân | Giải pháp |
|-------------|-----------|
| User chưa được gán Permission Set 帳票DX | Vào Setup → Permission Sets → Gán cho user |
| Lỗi "Insufficient Privileges" khi click button | Kiểm tra Profile có quyền "Customize Application" và quyền trên custom object |
| Lỗi khi truy cập URL action | Kiểm tra URL button có chứa đúng ACTION_ID chưa |

### 9.5 ❌ PDF Bị Lỗi Layout / Cắt Nội Dung

| Nguyên nhân | Giải pháp |
|-------------|-----------|
| Vùng lặp detail tràn sang trang tiếp theo bị cắt | Kiểm tra cài đặt Page Header, đảm bảo tiêu đề bảng được lặp |
| Margin quá nhỏ hoặc quá lớn | Điều chỉnh margin trong template Excel và Output Settings |
| Font không tương thích | Dùng font phổ biến (Arial, MS Gothic) có sẵn trong môi trường render |
| Cột quá rộng/hẹp | Điều chỉnh độ rộng cột trong template, test lại |

### 9.6 ❌ Nút Không Hiển Thị Trên List View

| Nguyên nhân | Giải pháp |
|-------------|-----------|
| Button chưa được thêm vào Search Layout | Setup → Object Manager → Search Layouts → List View → Thêm button |
| Button loại không phải "List Button" | Tạo lại button với Display Type = List Button |
| Cache trình duyệt | Xóa cache hoặc dùng Incognito/Private Mode |

---

## 10. Best Practices và Lưu Ý Vận Hành

### 10.1 Quản Lý ViewFramer

- **Đặt tên có quy tắc:** Dùng tiền tố theo object hoặc chức năng, ví dụ `ORD_HeaderDetail_v1`.
- **Tài liệu hóa alias:** Lưu danh sách alias field vào tài liệu nội bộ — dùng khi cần chỉnh sửa template.
- **Tránh xóa/đổi alias đang dùng:** Sẽ làm hỏng template đang sử dụng.
- **Test sau mỗi thay đổi:** Dù chỉ thêm/bỏ 1 field trong ViewFramer, luôn test lại output.

### 10.2 Quản Lý Template

- **Versioning template:** Lưu các phiên bản template với số version trong tên file (`_v1`, `_v2`).
- **Backup template Excel gốc:** Luôn giữ file `.xlsx` gốc trước khi upload lên 帳票DX.
- **Test template với dữ liệu biên (edge case):** Thử với record có detail rất nhiều dòng (>50), detail 0 dòng, text rất dài...

### 10.3 Hiệu Năng và Giới Hạn

- **Giới hạn số record khi xuất hàng loạt:** Kiểm tra tài liệu OPRO cho giới hạn số record tối đa khi dùng multi-select từ List View.
- **Thời gian xử lý:** PDF có nhiều dòng detail hoặc nhiều biểu mẫu sẽ mất nhiều thời gian hơn — thông báo cho người dùng.
- **Tránh field công thức phức tạp trong ViewFramer:** Nếu có thể, tính toán trước trong Salesforce (Formula/Rollup Summary) và dùng kết quả trong ViewFramer.

### 10.4 Bảo Mật

- **Không cấp quyền dư thừa:** Chỉ gán Permission Set 帳票DX cho user cần xuất báo cáo.
- **Review Field-Level Security:** Đảm bảo các field nhạy cảm (giá, chiết khấu...) có Field-Level Security phù hợp.
- **Log xuất PDF:** Kiểm tra xem 帳票DX có lưu log xuất file không — hữu ích cho audit.

### 10.5 Vận Hành Dài Hạn

- **Kiểm tra sau mỗi lần update 帳票DX package:** Tên tab/menu và hành vi có thể thay đổi.
- **Giữ liên lạc với OPRO support:** Khi gặp vấn đề về render PDF hoặc tích hợp phức tạp.
- **Đào tạo người dùng cuối:** Hướng dẫn cách chọn record và sử dụng nút List View Action đúng cách.

---

## Tài Liệu Tham Khảo

| Tài liệu | URL |
|----------|-----|
| 帳票DX User Guide (tiếng Nhật) | https://spc.opro.net/hc/ja/ |
| ViewFramer — TECH COLUMN | https://spc.opro.net/hc/ja/articles/360007922494 |
| Header–Detail ViewFramer Guide | https://spc.opro.net/hc/ja/articles/46050262559129 |
| 帳票DX System Requirements | https://spc.opro.net/hc/ja/articles/23655145322649 |

> **Tuyên bố từ chối:** Tài liệu này được viết dựa trên tài liệu kỹ thuật OPRO (spc.opro.net/hc/ja). Tên nhãn UI, tên tab, cú pháp tag và hành vi chính xác **có thể khác nhau tùy phiên bản 帳票DX và cấu hình org**. Luôn **đối chiếu với tài liệu chính thức của OPRO** và kiểm tra với admin của org khi có sự khác biệt.

---

*Tài liệu này được tạo cho repository [trunglucopilot1-blip/chatcopilot](https://github.com/trunglucopilot1-blip/chatcopilot). Cập nhật lần cuối: 2026-07.*
