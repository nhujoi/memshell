# Tổng hợp Kiến thức: Windows Executive Objects & Kiến trúc Cốt lõi

## 1. Khái niệm Cơ bản: Cấu trúc (Structure) vs Đối tượng Thực thi (Executive Object)
- **Cấu trúc C (C Structure):** Là một khối bộ nhớ nhóm các dữ liệu liên quan lại với nhau (ví dụ: vùng nhớ đệm gói tin mạng của `tcpip.sys`). Bất kỳ trình điều khiển nào cũng có thể tự tạo ra chúng và dùng nội bộ.
- **Đối tượng Thực thi (Executive Object):** Là một cấu trúc C đặc biệt, được cấp phát và quản lý tập trung bởi **Trình quản lý Đối tượng (Object Manager)** của Windows.
- **Điểm phân biệt cốt lõi:** Mỗi Đối tượng Thực thi luôn được hệ điều hành gắn thêm một phần đầu gọi là `_OBJECT_HEADER` (hoạt động như một "tem vận đơn"). Header này chứa các thông tin quản lý chuẩn hóa: Tên, Phân quyền bảo mật (Security Descriptor/ACL), và Số lượng tham chiếu (Reference Count).

## 2. Trình Quản lý Đối tượng (Object Manager)
- Là thành phần cốt lõi nằm trong nhân hệ điều hành (Kernel mode), được triển khai trong mô-đun NT (**`ntoskrnl.exe`**).
- Đóng vai trò là "Thủ thư tối cao", kiểm soát vòng đời của mọi tài nguyên quan trọng trên hệ thống:
  - **Tạo (Create):** Cấp phát vùng nhớ, dán `_OBJECT_HEADER`.
  - **Bảo vệ (Protect):** Dùng Access Control List (ACL) để kiểm tra xem một tiến trình có đủ đặc quyền để thao tác với đối tượng hay không (Ví dụ: Chặn các tiến trình lạ đọc bộ nhớ của LSASS).
  - **Xóa (Delete):** Sử dụng cơ chế đếm tham chiếu (Reference Counting). Chỉ khi không còn tiến trình nào giữ thẻ xử lý (Handle) trỏ đến đối tượng (count = 0), nó mới được giải phóng khỏi RAM.

## 3. Bản chất của Tiến trình (`_EPROCESS`)
- Trong Windows, "Mọi thứ cần quản lý đều là Object" để hệ thống có thể dùng chung một cổng an ninh (Object Manager). Vì vậy, Tiến trình (`_EPROCESS`) cũng là một Đối tượng.
- Tuy nhiên, **`_EPROCESS` đóng vai trò là một "Cái hộp chứa" (Container)**. Bản thân cái vỏ này không thực thi lệnh; nó phải gom các Đối tượng khác lại để tạo thành một môi trường chạy biệt lập:
  - **Không gian địa chỉ ảo:** Vùng RAM riêng biệt để chứa mã lệnh/dữ liệu.
  - **Luồng (`_ETHREAD`):** Đối tượng "công nhân", thực sự chạy các lệnh trên CPU.
  - **Thẻ bảo mật (`_TOKEN`):** Đối tượng định danh quyền hạn (User, Admin, SYSTEM...).
  - **Bảng Thẻ xử lý (Handle Table):** Danh bạ chứa các thẻ xử lý (Handles) trỏ đến các Đối tượng khác (File, Registry, Mutant...) mà tiến trình này đang được phép sử dụng.

## 4. Các Đối tượng Thực thi Phổ biến (Góc nhìn Pháp y Bộ nhớ)
| Tên Đối tượng | Cấu trúc C | Chức năng chính |
| :--- | :--- | :--- |
| **File** | `_FILE_OBJECT` | Đại diện cho tệp đang mở và quyền truy cập vào tệp đó. |
| **Process** | `_EPROCESS` | Bộ chứa tiến trình, ranh giới an toàn cho luồng và bộ nhớ. |
| **Thread** | `_ETHREAD` | Thực thể thực thi lệnh CPU được lập lịch. |
| **Token** | `_TOKEN` | Chứa ngữ cảnh bảo mật (SID, Privileges). |
| **Mutant** | `_KMUTANT` | Cơ chế loại trừ tương hỗ (Mutex), dùng để đồng bộ hóa. |
| **Key** | `_CM_KEY_BODY` | Đại diện cho một khóa Registry đang được mở. |
| **Driver** | `_DRIVER_OBJECT` | Driver chạy ở Kernel-mode đã được nạp vào bộ nhớ. |

## 5. Các Công cụ Khám phá & Phân tích
- **WinObj (Sysinternals):** Giao diện GUI xem toàn bộ "Không gian tên" (Namespace) của Object Manager. Hiển thị thông tin từ `_OBJECT_HEADER` (Tên, Loại, Reference Count, Security). *Lưu ý: Không xem được các Cấu trúc C thông thường.*
- **Process Explorer (Sysinternals):** Giao diện GUI xem các Đối tượng đang được một tiến trình cụ thể sử dụng (thông qua tính năng Lower Pane View > Handles). Có thể xem thông số của `_EPROCESS` như số luồng (Performance) và quyền (Security).
- **Volatility:** Công cụ CLI dùng trong điều tra pháp y (Memory Forensics). Dùng để phân tích trực tiếp file Dump RAM (VD: plugin `windows.pslist` để tìm `_EPROCESS`, `windows.handles` để xem bảng Handle).
- **WinDbg:** Trình gỡ lỗi nhân (Kernel Debugger). Cung cấp góc nhìn sâu nhất, cho phép xem từng byte mã hex cấu trúc C thực tế trong RAM (dùng lệnh `dt nt!_EPROCESS` hoặc `dt nt!_OBJECT_HEADER`).

## 6. Ứng dụng trong An ninh mạng (SOC / DFIR)
- **Tấn công lẩn tránh:** Kẻ tấn công lợi dụng kiến trúc "Container" thông qua kỹ thuật **Process Injection** – ép một `_EPROCESS` hợp lệ (như `explorer.exe`) chứa các đối tượng `_ETHREAD` độc hại để qua mặt Antivirus.
- **Giám sát & Phát hiện:** Mọi hành vi xin quyền truy cập Đối tượng đều phải đi qua Object Manager. Khi mã độc xin cấp Handle để xâm nhập vào tiến trình `lsass.exe`, quá trình "Bảo vệ" của Object Manager sẽ được kích hoạt và có thể được ghi nhận lại (ví dụ: **Event ID 10 - ProcessAccess** trong Sysmon), cung cấp dấu vết quan trọng cho các nhà phân tích SOC.