# **KẾ HOẠCH TRIỂN KHAI BẢO MẬT DỮ LIỆU CHO CỤM PERCONA MONGODB CE (60GB / 100 TRIỆU BẢN GHI)**

---

## **I. SO SÁNH TỔNG QUAN HAI PHƯƠNG ÁN**

| Tiêu chí | Phương án 1 (Chỉ CSFLE) | Phương án 2 (Data-at-Rest + CSFLE) |
| :--- | :--- | :--- |
| **Mức độ bảo mật** | **Trung bình - Cao**. Chống lộ lọt dữ liệu từ tầng ứng dụng (Hacker/DBA). Tệp `.wt` trên đĩa vật lý vẫn ở dạng thô. | **Tối đa (Gold Standard)**. Chống lộ lọt ở cả tầng ứng dụng lẫn tầng vật lý (đánh cắp ổ cứng, sao chép file DB). |
| **Phạm vi tác động** | Tác động phía **Ứng dụng (Client) và tác động một phần phía cấu hình database** (tạo kho chứa khóa). Không can thiệp sâu vào hạ tầng DB Server. | Tác động cả **Ứng dụng** lẫn **Hạ tầng DB Server** (cấu hình mã hóa đĩa từng node). |
| **Tổng Downtime Hệ thống** | **2 ngày** | **3 ngày** |

---

## **II. PHƯƠNG ÁN 1: CHỈ MÃ HÓA CẤP TRƯỜNG PHÍA KHÁCH (CSFLE)**

### **Giai đoạn A: Chuẩn bị(4 ngày) (Không Downtime)**

| Bước | Ai làm | Làm gì |
| :--- | :--- | :--- |
| A1 | Hạ tầng | Cài đặt và cấu hình cụm **HashiCorp Vault** (HA 3 Node) phục vụ quản lý Master Key cho CSFLE. |
| A2 | Lập trình | Lập trình tính năng **mã hóa dữ liệu đầu vào** và **mã hóa giá trị tìm kiếm** trên ứng dụng (tích hợp MongoDB Driver với CSFLE). |
| A3 | Lập trình | Lập trình **Tool di cư dữ liệu** (Migration Tool) để mã hóa hàng loạt dữ liệu rõ cũ sang dạng nhị phân BinData. Tool phải hỗ trợ xử lý **song song đa luồng** và **gộp lô (Bulk Write)** để đảm bảo tốc độ xử lý 100 triệu bản ghi trong vòng 1 - 2 tiếng. |
| A4 | Kiểm thử | Kiểm thử toàn bộ tính năng mã hóa/giải mã và Migration Tool trên môi trường **Staging/UAT**. |

### **Giai đoạn B: Triển khai(2 ngày, có downtime hệ thống)**

| Bước | Ai làm | Làm gì |
| :--- | :--- | :--- |
| B1 | Hạ tầng | **Dừng toàn bộ ứng dụng** kết nối tới cụm MongoDB (đóng băng dữ liệu, không cho phát sinh ghi mới). |
| B2 | Hạ tầng | **Backup dữ liệu** cụm MongoDB để dự phòng rollback. |
| B3 | Hạ tầng | **Cấu hình CSFLE và kết nối vault**.|
| B4 | Hạ tầng | **Kiểm tra hoạt động cụm và tính năng CSFLE**.|
| B5 | Lập trình | **Chạy Migration Tool** mã hóa toàn bộ dữ liệu rõ cũ (100 triệu bản ghi) thành dạng nhị phân mã hóa BinData Subtype 6. |
| B6 | Lập trình | **Reindex lại Elasticsearch** đối với các trường dữ liệu vừa được mã hóa. |
| B7 | Lập trình | **Deploy phiên bản ứng dụng mới** (đã bật cấu hình tự động mã hóa/giải mã CSFLE). |
| B8 | Hạ tầng | **Kiểm tra kết nối** và **mở lại hệ thống** cho người dùng truy cập. **DOWNTIME KẾT THÚC.** |

---

## **III. PHƯƠNG ÁN 2: MÃ HÓA ĐA TẦNG (DATA-AT-REST + CSFLE)**

Phương án 2 bao gồm **toàn bộ các bước của Phương án 1**, đồng thời **bổ sung thêm quy trình mã hóa dữ liệu tĩnh (Data-at-Rest)** tại tầng lưu trữ vật lý của cụm MongoDB.

### **Giai đoạn A: Chuẩn bị(4 ngày, không downtime)**

Thực hiện toàn bộ các bước **A1, A2, A3, A4** của Phương án 1, đồng thời bổ sung:

| Bước | Ai làm | Làm gì |
| :--- | :--- | :--- |
| A5 | Hạ tầng | Cấu hình thêm trên HashiCorp Vault: Tạo **Master Key cho mã hóa đĩa tĩnh** (Vault KV Secrets Engine v2). Tạo Token xác thực cho từng node MongoDB. |

### **Giai đoạn B: Triển khai (3 ngày, có downtime hệ thống)**

| Bước | Ai làm | Làm gì |
| :--- | :--- | :--- |
| B1 | Hạ tầng | **Dừng toàn bộ ứng dụng** kết nối tới cụm MongoDB (đóng băng dữ liệu). |
| B2 | Hạ tầng | **Backup dữ liệu** cụm MongoDB để dự phòng rollback. |
| **B3** | **Hạ tầng** | **Mã hóa đĩa tĩnh Node 3 (Secondary):** Dừng dịch vụ MongoDB trên Node 3 → Cấu hình kích hoạt mã hóa đĩa tĩnh tích hợp Vault → Xóa thư mục dữ liệu → Khởi động lại → Chờ Node 3 tự động đồng bộ (Initial Sync) từ Primary và chuyển sang trạng thái `SECONDARY`. |
| **B4** | **Hạ tầng** | **Mã hóa đĩa tĩnh Node 2 (Secondary):** Thực hiện tương tự Node 3. Chờ Node 2 đồng bộ xong và trạng thái `SECONDARY`. |
| **B5** | **Hạ tầng** | **Chuyển giao Primary:** Chạy lệnh `rs.stepDown()` trên Node 1 (Primary hiện tại) để một trong hai node đã mã hóa lên làm Primary mới. |
| **B6** | **Hạ tầng** | **Mã hóa đĩa tĩnh Node 1 (Primary cũ, giờ là Secondary):** Thực hiện tương tự Node 3 & Node 2. Chờ đồng bộ xong. **Cả 3 node đã được mã hóa đĩa tĩnh 100%.** |
| B7 | Lập trình | **Chạy Migration Tool** mã hóa toàn bộ dữ liệu rõ cũ (CSFLE). |
| B8 | Lập trình | **Reindex lại Elasticsearch.** |
| B9 | Lập trình | **Deploy phiên bản ứng dụng mới** (đã bật CSFLE). |
| B10 | Hạ tầng | **Kiểm tra kết nối** và **mở lại hệ thống.** **DOWNTIME KẾT THÚC.** |

---

## **IV. KỊCH BẢN ROLLBACK (PHÒNG NGỪA RỦI RO)**

| Sự cố | Cách xử lý |
| :--- | :--- |
| **Node đồng bộ đĩa tĩnh bị lỗi (PA2)** | Dừng node lỗi → Khôi phục cấu hình gốc (bỏ cấu hình mã hóa) → Xóa thư mục dữ liệu → Khởi động lại để node tự đồng bộ dạng không mã hóa từ Primary cũ. Hệ thống trở về trạng thái ban đầu. |
| **Migration Tool bị lỗi giữa chừng** | Dừng tool → Restore lại collection bị lỗi từ bản backup đã tạo ở bước B2 → Deploy lại phiên bản ứng dụng cũ → Phân tích log để sửa lỗi tool trước khi lên lịch bảo trì lần sau. |
| **Vault bị sập trong quá trình thi công** | Chờ khôi phục cụm Vault (HA 3 Node) → Tiếp tục quy trình từ bước đang dở. Nếu không khôi phục được Vault, thực hiện rollback toàn bộ về trạng thái ban đầu bằng bản backup. |
