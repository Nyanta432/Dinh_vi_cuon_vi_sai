# 🧲 Lõi Thuật Toán & Mô Hình Vật Lý: Hệ Thống Định Vị Từ Trường 6-DOF
Tài liệu này đi sâu vào giải phẫu **Mô hình Toán học (Forward Model)** và **Thuật toán Tối ưu hóa (Optimization Algorithm)** được sử dụng trong dự án. Điểm nhấn lớn nhất của hệ thống là phương pháp xử lý triệt để điểm kỳ dị (Singularity) sinh ra từ cảm biến cuộn vi sai (Differential Coil) tại vùng trường gần (Near-field).

---

## ⚙️ 1. Cấu trúc Thuật toán (The Algorithm)
Hệ thống giải quyết bài toán định vị 6 bậc tự do (6-DOF) thông qua quy trình 2 giai đoạn khép kín (End-to-End):
1. **Auto-Calibration (Hiệu chuẩn hệ thống):** Tối ưu hóa tọa độ thực tế của 3 cuộn phát (TX) và hệ số khuếch đại điện áp ($K$) dựa trên các điểm mồi (Initial Guesses).
2. **Inverse Kinematics (Động học ngược):** Sử dụng thông số TX và $K$ đã hiệu chuẩn để dò tìm ngược lại quỹ đạo 6D $[x, y, z, roll, pitch, yaw]$ của viên nang theo từng khung hình.

**Cốt lõi Toán học:** 
Cả hai giai đoạn đều được giải quyết bằng thuật toán **Non-linear Least Squares (Bình phương tối thiểu phi tuyến)** thông qua hàm `scipy.optimize.least_squares` với phương pháp **Trust Region Reflective (TRF)**.

**💡 Chìa khóa hội tụ (Jacobian Scaling):** 
Không gian trạng thái chứa các biến có tỷ lệ chênh lệch khổng lồ: Tọa độ ($\sim 0.5\text{ m}$), Góc quay ($\sim 180^\circ$) và Hệ số $K$ ($\sim 0.001$). Để tránh "Bẫy Gradient" (thuật toán kẹt ở cực tiểu cục bộ do đạo hàm của $K$ quá nhỏ), hệ thống kích hoạt cơ chế `x_scale='jac'`. Tính năng này buộc Python tự động co giãn ma trận Jacobian, mô phỏng lại sự thông minh của hàm `lsqnonlin` trong MATLAB, giúp thuật toán bứt phá các góc xoay lớn và bám sát các đỉnh tín hiệu.

---

## 🧲 2. Giải phẫu Forward Model (Cuộn Vi Sai)
**Forward Model** là trái tim của hệ thống, làm nhiệm vụ tính toán tín hiệu điện áp cảm ứng (EMF) lý thuyết khi biết trước tọa độ 6D.
* Phần cứng thu tín hiệu (RX) là một **Cuộn vi sai (Spatial Gradiometer)** gồm 2 nửa cuộn dây quấn ngược chiều, cách nhau khoảng cách $d = 0.25\text{ mm}$.
* Thiết kế này triệt tiêu hoàn toàn nhiễu từ trường đồng pha (như từ trường Trái Đất), nhưng lại sinh ra một bài toán toán học cực kỳ hóc búa ở vùng trường gần.

### Vấn đề kỳ dị và Nguyên nhân thuật toán bị kẹt (The $r^{-4}$ Singularity & Gradient Trap)
Với mô hình "Điểm từ lý tưởng" (Point Dipole), từ trường $\mathbf{B}$ tỉ lệ nghịch với lập phương khoảng cách: $\mathbf{B} \propto \frac{1}{r^3}$.
Do cuộn vi sai đo sự chênh lệch từ trường giữa 2 điểm rất gần nhau, phép đo này tương đương với đạo hàm không gian bậc nhất của $\mathbf{B}$:
$$\Delta \mathbf{B} \approx \frac{\partial \mathbf{B}}{\partial r} \cdot d \propto \frac{d}{r^4}$$

Khi viên nang di chuyển sát vào cuộn phát ($r \to 0$), mô hình lý thuyết $1/r^4$ gây ra sự sụp đổ của toàn bộ thuật toán tối ưu. Quá trình "chết kẹt" này diễn ra qua 3 bước toán học:

1. **Bùng nổ Gradient (Gradient Explosion):** Thuật toán `least_squares` hoạt động bằng cách tính ma trận Jacobian (đạo hàm riêng của sai số theo từng biến) để dò tìm hướng dốc đi xuống. Đạo hàm của $r^{-4}$ là $-4r^{-5}$. Khi $r \to 0$, đạo hàm này tiến tới vô cực với tốc độ khủng khiếp. Không gian sai số tại đây không còn là một cái phễu trơn tru mà biến thành một "bức tường thẳng đứng".
2. **Nhiễu loạn bước nhảy (Step-size Chaos):** Đứng trước bức tường vô cực này, chỉ cần thuật toán nhích thử tọa độ $(x, y, z)$ đi một khoảng cực nhỏ (ví dụ $0.001\text{ mm}$), tín hiệu lý thuyết $\Delta \mathbf{B}$ lập tức nhảy vọt lên hàng tỷ lần. Điều này tạo ra một sai số khổng lồ so với tín hiệu thực tế (chỉ khoảng vài chục mV).
3. **Lỗ hổng Toán học và Sự "Bỏ cuộc" (The Mathematical Loophole):** Đứng trước một không gian hỗn loạn không thể dò đường bằng tọa độ, thuật toán nhận ra một lối tắt. Vì tín hiệu cuối cùng được tính bằng công thức $EMF = K \cdot \Delta \mathbf{B}$, thuật toán quyết định "đầu hàng" việc dò tìm tọa độ và lập tức ép hệ số khuếch đại $K$ tiến sát về $0$. Khi $K = 0$, toàn bộ $EMF = 0$, triệt tiêu sự bùng nổ của $\Delta \mathbf{B}$ và mang lại một sai số hữu hạn an toàn. 

**Hệ quả:** Thuật toán dừng chạy và báo "tối ưu thành công" từ rất sớm, để lại kết quả là một đường tín hiệu phẳng lỳ (flatline) nằm bẹt dưới trục hoành, hoàn toàn bỏ lỡ các đỉnh tín hiệu đo đạc phần cứng.

---

## 🛠️ 3. Giải pháp: Lưỡng cực Điều chuẩn (Regularized Dipole Model)
Để triệt tiêu điểm kỳ dị mà không làm mất đi bản chất đo vi sai (khoảng cách $d$), hệ thống nâng cấp từ mô hình "Điểm từ lý tưởng" lên mô hình "Vòng dây hữu hạn" (Finite-size Coil Approximation) bằng phương pháp **Điều chuẩn (Regularization)**.

**Bước 1: Đưa hệ số chặn vật lý ($R^2$) vào mẫu số**
Cuộn phát TX thực tế có bán kính $R = 60\text{ mm}$. Hệ thống tích hợp $R$ trực tiếp vào mẫu số của thế vô hướng từ. Qua quá trình lấy Gradient, phương trình từ trường $\mathbf{B}$ chuẩn xác tại một nửa của cuộn vi sai trở thành:
$$\mathbf{B} = \frac{\mu_0}{4\pi} \frac{3(\mathbf{m} \cdot \mathbf{r})\mathbf{r} - (r^2 + R^2)\mathbf{m}}{(r^2 + R^2)^{5/2}}$$

*(Trong đó $\mathbf{m}$ là momen từ, $\mathbf{r}$ là vector khoảng cách).*

**Bước 2: Xử lý giới hạn khi $r \to 0$**
Nhờ có $R^2$, khi viên nang chạm mặt cuộn phát ($r = 0$), mẫu số không bị triệt tiêu mà trở thành một hằng số giới hạn vật lý:
$$\lim_{r \to 0} (r^2 + R^2)^{5/2} = (0 + R^2)^{5/2} = R^5$$
Khối chóp vô cực giờ đây được "bo tròn" thành một đỉnh (peak) trơn tru và mềm mại.

**Bước 3: Chiếu lên Cuộn Vi Sai**
Mô hình toán học bảo toàn nguyên vẹn tính vi sai bằng cách tính $\mathbf{B}$ độc lập cho 2 vị trí cộng/trừ của cuộn thu:
$$\mathbf{r}_{plus} = \mathbf{r}_{rx} + \frac{d}{2}\mathbf{\hat{n}}_{rx} - \mathbf{r}_{tx}$$
$$\mathbf{r}_{minus} = \mathbf{r}_{rx} - \frac{d}{2}\mathbf{\hat{n}}_{rx} - \mathbf{r}_{tx}$$
Độ lệch từ trường sinh ra dòng điện:
$$EMF \propto \left\vert{} (\mathbf{B}_{plus} - \mathbf{B}_{minus}) \cdot \mathbf{\hat{n}}_{rx} \right\vert{}$$

### 🏆 Kết quả
Việc sử dụng mẫu số $(r^2 + R^2)^{5/2}$ đã giải cứu hoàn toàn thuật toán tối ưu. Không gian Gradient trở nên mượt mà (smooth) ở ngay cả những điểm sát cuộn dây nhất. Thuật toán `least_squares` không còn bị hoảng loạn bởi bùng nổ đạo hàm, loại bỏ hoàn toàn hiện tượng ép biến $K \to 0$. Nhờ đó, nó thong thả tính toán các bước nhảy lớn, tìm ra chính xác các góc xoay $164^\circ$ và bám khít $100\%$ vào các đỉnh tín hiệu $10\text{ mV}$ - $13\text{ mV}$ của cuộn vi sai thực tế.
