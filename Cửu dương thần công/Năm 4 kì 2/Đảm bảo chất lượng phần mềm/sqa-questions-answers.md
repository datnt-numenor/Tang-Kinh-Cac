# Báo cáo Trả lời Câu hỏi Bảo vệ/Phỏng vấn BTL SQA

Tài liệu này tổng hợp câu trả lời cho các câu hỏi của giáo viên liên quan đến bài tập lớn môn **Đảm bảo chất lượng phần mềm (SQA)** dựa trên quá trình thực hiện Unit Test các phân hệ **UC25 (Đơn hàng)**, **UC26 (Nhà cung cấp)**, **UC28 (Phiếu nhập)** bằng Jest.

---

## 🛠️ 1. Các kỹ thuật và công cụ sử dụng (Techniques & Tools)

### **a. Công cụ (Tools)**
*   **Jest (Testing Framework):** Thư viện chính chạy kiểm thử đơn vị, hỗ trợ assertion, snapshot testing, mocking mạnh mẽ và thống kê độ bao phủ (coverage report).
*   **Node.js & Express:** Môi trường chạy ứng dụng backend cần được kiểm thử.
*   **Sequelize (ORM):** Công cụ quản lý và truy vấn cơ sở dữ liệu dạng đối tượng trong ứng dụng.

### **b. Kỹ thuật kiểm thử hộp đen áp dụng kèm ví dụ thực tế (Black-Box Testing Techniques & Examples)**

Trong bài tập lớn này, các kỹ thuật kiểm thử hộp đen (Black-box testing) đóng vai trò nòng cốt để phát hiện lỗi logic nghiệp vụ:

#### **Kỹ thuật 1: Phân vùng tương đương (Equivalence Partitioning)**
*   **Mô tả:** Chia miền giá trị đầu vào của hàm thành các lớp dữ liệu hợp lệ và không hợp lệ. Sau đó chọn ra một giá trị đại diện trong mỗi lớp để viết test case.
*   **Ví dụ thực tế từ UC26 (Nhà cung cấp - `supplierService.js`):**
    *   *Miền dữ liệu Email:*
        *   **Lớp Email hợp lệ:** Đại diện là `"supplier@gmail.com"` $\rightarrow$ Test case kỳ vọng trả về `errCode: 0` (Thành công).
        *   **Lớp Email không hợp lệ:** Đại diện là `"invalid_email_format"` (thiếu ký tự `@` và domain) $\rightarrow$ Test case kỳ vọng hệ thống báo lỗi đầu vào `errCode != 0` (Thất bại).
    *   *Miền dữ liệu Số điện thoại:*
        *   **Lớp SĐT hợp lệ:** Đại diện là `"0987654321"` $\rightarrow$ Test case kỳ vọng trả về `errCode: 0`.
        *   **Lớp SĐT không hợp lệ (chứa chữ):** Đại diện là `"0987abc123"` $\rightarrow$ Test case kỳ vọng hệ thống phát hiện lỗi và trả về `errCode != 0`.

#### **Kỹ thuật 2: Phân tích giá trị biên (Boundary Value Analysis)**
*   **Mô tả:** Tập trung kiểm thử tại các biên của miền dữ liệu (giá trị tối thiểu, tối đa, ngay trên biên, ngay dưới biên) vì lập trình viên thường rất dễ mắc lỗi so sánh ($\le$, $\ge$ thay vì $<$, $>$).
*   **Ví dụ thực tế từ UC28 (Phiếu nhập hàng - `receiptService.js`):**
    *   *Biên của số lượng nhập (`quantity`) và đơn giá (`price`):* Theo nghiệp vụ, số lượng nhập kho và đơn giá sản phẩm phải là số nguyên dương ($> 0$). Giá trị biên tối thiểu hợp lệ là `1`.
    *   **Giá trị tại biên hợp lệ:** `quantity = 1` hoặc `price = 1000` $\rightarrow$ Kết quả mong muốn: Ghi nhận thành công.
    *   **Giá trị sát biên không hợp lệ:** `quantity = 0` $\rightarrow$ Kết quả mong muốn: Báo lỗi đầu vào.
    *   **Giá trị dưới biên không hợp lệ:** `quantity = -5` hoặc `price = -1000` (giá trị âm) $\rightarrow$ Kịch bản thực thi gửi tham số âm này và kỳ vọng hệ thống chặn lại và trả về `errCode != 0`.

#### **Kỹ thuật 3: Kiểm thử chuyển trạng thái (State Transition Testing)**
*   **Mô tả:** Kiểm tra hành vi thay đổi trạng thái của thực thể trong hệ thống dưới tác động của các sự kiện và điều kiện ràng buộc đi kèm.
*   **Ví dụ thực tế từ UC25 (Quản lý đơn hàng - `orderService.js`):**
    *   *Vòng đời trạng thái đơn hàng:* Từ trạng thái Xác nhận (`S4`) sang trạng thái Đang giao hàng (`S5`).
    *   *Điều kiện chuyển trạng thái:* Hệ thống chỉ cho phép chuyển sang giao hàng (`S5`) nếu đơn hàng đó **đã được phân công Shipper** (trường `shipperId` khác null).
    *   **Kịch bản kiểm thử trạng thái biên:** Thiết lập đơn hàng ở trạng thái `S4` nhưng gán `shipperId = null`. Gửi yêu cầu chuyển trạng thái đơn hàng sang `S5`. Kỳ vọng mong muốn là hệ thống phát hiện điều kiện thiếu shipper và chặn lại, trả về mã lỗi (`errCode != 0`).

---

## 📊 2. Chuẩn bị dữ liệu kiểm thử (Test Data Preparation)

*   **Sử dụng Dữ liệu Giả lập (Mock Data):**
    *   Toàn bộ dữ liệu phục vụ kiểm thử đơn vị được chuẩn bị trực tiếp bên trong mã nguồn kiểm thử (inline/mock data) bằng các hàm sinh dữ liệu giả lập của Jest (như `mockResolvedValue`, `mockImplementation`).
*   **Giả lập Thực thể Cơ sở dữ liệu (Model Instance Mocking):**
    *   Tự thiết lập các instance giả của Sequelize Models có kèm theo phương thức nguyên mẫu (prototype methods) của Sequelize như `.save()` (ví dụ: `save: jest.fn().mockResolvedValue(true)`) nhằm giả lập hoạt động lưu dữ liệu xuống DB thành công.
*   **Chuẩn bị Dữ liệu liên kết lồng nhau (Complex Nested Mock Data):**
    *   Đối với các nghiệp vụ phức tạp như lấy chi tiết đơn hàng (`getDetailOrderById`), cá nhân tự chuẩn bị các đối tượng liên kết sâu (nested objects) để mô phỏng mối quan hệ giữa Đơn hàng -> Voucher -> Loại Voucher -> Chi tiết đơn hàng -> Địa chỉ người nhận -> Người dùng.

---

## 🚀 3. Những vấn đề khó mà cá nhân đã giải quyết (Challenges & Solutions)

Trong quá trình thực hiện, tôi đã tự mình giải quyết được **3 bài toán khó** sau:

### **Bài toán 1: Cô lập Controller khỏi Database Side-Effects**
*   **Vấn đề:** Khi viết Unit Test cho Controller, Controller sẽ gọi Service, và Service lại truy vấn Database thực. Nếu không xử lý khéo, Unit Test của Controller sẽ bị lỗi do không kết nối được Database hoặc dữ liệu trong DB thay đổi liên tục.
*   **Giải pháp:** Tôi đã áp dụng kỹ thuật `jest.spyOn(orderService, "getAllOrders")` ngay trong hook `beforeEach` của bộ test Controller. Giải pháp này giúp chặn và ép hàm của Service trả về giá trị mock ngay lập tức, ngắt hoàn toàn kết nối xuống DB mà không làm ảnh hưởng đến mã nguồn thực tế.

### **Bài toán 2: Giả lập Sequelize Transaction**
*   **Vấn đề:** Nghiệp vụ lập phiếu nhập hàng (`receiptService.createNewReceipt` - UC28) yêu cầu thực hiện ghi dữ liệu trên nhiều bảng khác nhau dưới dạng một Transaction để đảm bảo tính nguyên tử (Atomicity). Việc giả lập một Transaction của Sequelize (`db.sequelize.transaction`) trong môi trường Unit Test rất khó vì nó chứa các method lồng nhau và các callback phức tạp.
*   **Giải pháp:** Tôi đã tạo ra cấu trúc Mock đối tượng Transaction giả lập trả về một Object chứa các hàm spy: `commit: jest.fn()` và `rollback: jest.fn()`. Nhờ đó, tôi kiểm thử được chính xác xem hệ thống có tự động gọi rollback khi gặp ngoại lệ hay không.

### **Bài toán 3: Kiểm thử phát hiện Lỗi Logic ẩn (Logical Bug)**
*   **Vấn đề:** Các lỗi nghiệp vụ như cho phép đơn hàng sang trạng thái vận chuyển khi chưa gán shipper (UC25), hoặc tạo phiếu nhập có số lượng/đơn giá âm (UC28) hoàn toàn không gây lỗi Runtime (không làm crash app), nên các công cụ quét lỗi tự động thông thường không thể phát hiện.
*   **Giải pháp:** Tôi đã thiết kế các kịch bản kiểm thử cố tình truyền vào các tham số sai lệch nghiệp vụ, nhưng đặt kỳ vọng (assertion) mong muốn là hệ thống phải từ chối và báo lỗi (`expect(res.errCode).not.toBe(0)`). Khi chạy thử, Jest thông báo các test case này bị **FAIL** trên mã nguồn gốc $\rightarrow$ chứng minh thành công sự tồn tại của lỗi logic nghiệp vụ tại Backend.
