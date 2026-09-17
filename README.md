# 3D Magnetic Tracking System - Endoscopic Capsule Localization

> Hệ thống định vị 3D không dây cho viên nang nội soi sử dụng mô hình lưỡng cực từ (Magnetic Dipole Modeling) và thuật toán tối ưu hóa phi tuyến trên nền tảng Python (Google Colab).

---

## 📌 Tổng quan dự án (Overview)
Đề tài tập trung giải quyết **bài toán ngược (Inverse Problem)** trong hệ thống định vị y tế không dây: Dịch ngược tín hiệu điện áp cảm ứng ($\text{EMF}$) thu được từ phần cứng thành tọa độ không gian 3D thực tế $(X, Y, Z)$ và góc quay định hướng ($\text{Roll, Pitch, Yaw}$) của viên nang nội soi.

Mã nguồn được thiết kế dạng mô-đun (chia thành các Section độc lập trên Jupyter Notebook / Google Colab) để dễ dàng kiểm chứng mô hình vật lý, hiệu chuẩn phần cứng và trực quan hóa kết quả thực nghiệm.

---

## ⚙️ Cấu hình phần cứng hệ thống (System Architecture)
* **Hệ thống Phát ($\text{TX}$):** 3 cuộn dây phát từ trường độc lập ($N_{tx} = 300$ vòng, bán kính $R = 6\text{ cm}$), kích thích ở các tần số khác nhau $[5050, 6910, 5930]\text{ Hz}$.
* **Hệ thống Thu ($\text{RX}$):** Cảm biến vi sai gồm 2 nửa cuộn dây cách nhau khoảng vi sai $d = 0.25\text{ mm}$.
  * *Đặc thù phần cứng:* Cảm biến $\text{RX}_2$ được bố trí nằm ngang hướng thẳng lên trục $Z$, trong khi $\text{RX}_3$ hướng theo trục $Y$ và $\text{RX}_1$ theo trục $X$.
* **Dữ liệu thực nghiệm (`conenorot_data.csv`):** Chứa tín hiệu điện áp đo đạc thô ở 9 kênh ($3\text{ TX} \times 3\text{ RX}$) kết hợp với dữ liệu chuẩn Ground Truth từ thiết bị cơ khí.

---

## 📂 Kiến trúc mã nguồn chi tiết (Source Code Breakdown)
Mã nguồn được phân chia thành 7 khối (Sections) rành mạch. Dưới đây là chức năng và ý nghĩa kỹ thuật của từng khối:

### Section 1: Cấu hình thông số vật lý (Physical Parameters Setup)
* **Nhiệm vụ:** Khai báo toàn bộ các hằng số vật lý cốt lõi của phần cứng hệ thống (số vòng dây phát $N_{tx}$, bán kính cuộn $R$, tần số kích thích $f$, dòng điện kích thích $I$, khoảng cách vi sai $d = 0.25\text{ mm}$, diện tích mặt cắt $AK$). Đặc biệt là nạp ma trận tọa độ và góc đặt nghiêng thực tế của 3 cuộn phát ($\text{TX}$).
* **Ý nghĩa:** Cung cấp "luật chơi" vật lý chuẩn xác, đóng vai trò nền tảng cho toàn bộ mô hình toán học ở các bước sau.

### Section 2: Tải và trích xuất dữ liệu (Data Loading & Pre-processing)
* **Nhiệm vụ:** Đọc file thực nghiệm `conenorot_data.csv`. Trích xuất dữ liệu thành 3 ma trận độc lập: 9 cột tín hiệu điện áp đo thực tế ($\text{EMF}$), 3 cột tọa độ thực tế (`GT_pos`), và 3 cột góc quay (`GT_ang`). Code tích hợp cơ chế kiểm tra lỗi tệp (File Not Found) và **tự động quy đổi đơn vị tọa độ** từ $\text{mm}$ sang mét ($\text{m}$) chuẩn SI.
* **Ý nghĩa:** Đảm bảo đồng nhất hệ đơn vị đo lường giữa phần cứng và phương trình Maxwell, tránh triệt tiêu từ trường về $0$ do lệch đơn vị.

### Section 3: Trực quan hóa cơ sở dữ liệu (Database Visualization)
* **Nhiệm vụ:** Sử dụng thư viện `matplotlib` để render biểu đồ không gian 3D, tái hiện lại quỹ đạo di chuyển thực tế (Ground Truth) của viên nang nội soi từ điểm xuất phát (màu xanh) đến điểm kết thúc (màu đỏ).
* **Ý nghĩa:** Giúp người nghiên cứu có cảm quan trực quan về hình thù đường đi của viên nang trước khi đối chiếu với thuật toán giải mã.

### Section 4: Mô hình vật lý thuận (Forward Model - Discrete Method)
* **Nhiệm vụ:** Xây dựng hàm toán học mô phỏng quá trình sinh từ trường $\mathbf{B}$ của lưỡng cực từ. Thay vì dùng xấp xỉ đạo hàm vi phân, mô hình áp dụng phương pháp Rời rạc (Discrete): Tính toán cảm ứng từ trực tiếp tại hai nửa cuộn thu cách nhau khoảng $d$, sau đó lấy hiệu số $\Delta B$. Thuật toán ánh xạ chính xác thiết kế phần cứng ($\text{RX}_2$ theo trục $Z$, $\text{RX}_3$ theo trục $Y$, $\text{RX}_1$ theo trục $X$).
* **Ý nghĩa:** "Đóng vai" phần cứng thực nghiệm, giúp tính toán ngược ra mức điện áp lý tưởng thu được dựa trên một bộ tọa độ viên nang cho trước.

### Section 5: Hiệu chuẩn biên độ (Scale Calibration - Closed Form)
* **Nhiệm vụ:** Chạy mô hình lý thuyết qua toàn bộ các điểm quỹ đạo Ground Truth. Sau đó áp dụng phương pháp Hồi quy tuyến tính đóng (Closed-form Least Squares) nhằm tính ra mảng 9 hệ số tỷ lệ (`scale_factors`) cho 9 kênh đo.
* **Ý nghĩa:** Đây là bước tối quan trọng để bù đắp các suy hao biên độ tín hiệu, nhiễu điện trở mạch, hoặc sai số gia công linh kiện giữa môi trường lý tưởng và thực tế.

### Section 6: Giải mã tọa độ bằng tối ưu hóa (Localization Optimization)
* **Nhiệm vụ:** Đây là "trái tim" của hệ thống phần mềm. Vòng lặp giải quyết Bài toán ngược (Inverse Problem) cho từng mẫu đo bằng hàm tối ưu phi tuyến `scipy.optimize.least_squares` với thuật toán `trust-region-reflective`. Hàm mục tiêu ép phần dư giữa [EMF Lý thuyết $\times$ Scale] và [EMF Đo đạc] về mốc $0$. Mỗi nghiệm được mồi từ Ground Truth và giới hạn nghiêm ngặt trong hộp không gian $\pm 3\text{ cm}$.
* **Ý nghĩa:** Dịch ngược tín hiệu điện áp đo được thành không gian 3D thực tế. 

### Section 7: Kiểm chứng & Trực quan hóa kết quả (Evaluation & Visualization)
* **Nhiệm vụ:** Tính toán sai số lệch chuẩn toàn cục ($\text{RMSE}$) tính bằng milimet. Xuất 2 bộ đồ thị phân tích: Đồ thị chồng chập quỹ đạo 3D (Thực tế vs Ước lượng) và Lưới đồ thị phân tích tín hiệu $3 \times 3$.
* **Ý nghĩa:** Lưới $3 \times 3$ so sánh trực tiếp đường cong điện áp Đo đạc (Đen) và Lý thuyết (Đỏ), cung cấp bằng chứng khoa học tuyệt đối về mức độ hội tụ của hệ thống mô hình.

---
## 📊 Kết quả đánh giá hệ thống
* **Định lượng:** Báo cáo mức độ bám sát thực nghiệm qua sai số vị trí tổng thể $\text{RMSE}$.
* **Định tính:** Minh chứng sự đồng bộ pha và biên độ hoàn hảo giữa cảm biến thực tế và mô hình toán học qua biểu đồ đa kênh $3 \times 3$.
