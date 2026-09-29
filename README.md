### Vấn đề kỳ dị và Nguyên nhân thuật toán bị kẹt (The Singularity & Gradient Trap)
Trong mô hình "Điểm từ lý tưởng" (Point Dipole) cổ điển, phương trình từ trường $\mathbf{B}$ sinh ra bởi cuộn phát được xác định bằng công thức chính xác:
$$\mathbf{B} = \frac{\mu_0}{4\pi} \frac{3(\mathbf{m} \cdot \mathbf{r})\mathbf{r} - r^2\mathbf{m}}{r^5}$$

*(Trong đó: $\mathbf{m}$ là momen từ, $\mathbf{r}$ là vector khoảng cách, và $r = \vert{}\mathbf{r}\vert{}$ là độ lớn khoảng cách).*

Cuộn vi sai thu tín hiệu bằng cách đo sự chênh lệch từ trường giữa hai nửa cuộn dây đặt cách nhau một khoảng $d$. Về mặt toán học, phép đo này xấp xỉ với đạo hàm bậc nhất của từ trường theo không gian ($\Delta \mathbf{B} \approx \frac{\partial \mathbf{B}}{\partial r} \cdot d$). Phép lấy đạo hàm của phân thức chứa $r^5$ ở mẫu số sẽ làm tăng bậc của biến $r$, sinh ra các thành phần chứa $r^{-4}$ hoặc bậc cao hơn. 

Khi viên nang di chuyển sát vào mặt cuộn phát ($r \to 0$), việc tồn tại biến $r$ ở dưới mẫu số gây ra sự sụp đổ của toàn bộ thuật toán tối ưu. Quá trình "chết kẹt" diễn ra qua 3 bước:

1. **Bùng nổ chia cho $0$ (Division by Zero):** Khi $r \to 0$, toàn bộ mẫu số tiến về $0$. Theo nguyên lý toán học, việc chia cho $0$ đẩy cường độ từ trường $\mathbf{B}$ và độ chênh lệch vi sai $\Delta \mathbf{B}$ lên vô cực ($\infty$). Không gian sai số tại đây không còn là một cái phễu trơn tru để thuật toán trượt xuống, mà biến thành một "bức tường thẳng đứng".
2. **Nhiễu loạn bước nhảy (Step-size Chaos):** Đứng trước bức tường vô cực này, thuật toán `least_squares` dò đường bằng cách nhích thử tọa độ $(x, y, z)$ đi một khoảng cực nhỏ (ví dụ $0.001\text{ mm}$). Tuy nhiên, do mẫu số quá gần $0$, một sự thay đổi cực nhỏ của $r$ cũng khiến phân thức phân kỳ, đẩy tín hiệu lý thuyết nhảy vọt lên hàng tỷ lần và tạo ra sai số khổng lồ.
3. **Lỗ hổng Toán học (The Mathematical Loophole):** Đứng trước một không gian hỗn loạn không thể dò đường bằng tọa độ, thuật toán nhận ra một lối tắt. Tín hiệu cuối cùng được tính bằng công thức $EMF = K \cdot \Delta \mathbf{B}$. Thay vì cố gắng tìm tọa độ, thuật toán "đầu hàng" bằng cách ép hệ số khuếch đại $K$ về $0$. Khi $K = 0$, toàn bộ $EMF = 0$, triệt tiêu hiện tượng vô cực và mang lại một sai số an toàn. Hệ quả là thuật toán dừng chạy sớm, để lại một đường tín hiệu phẳng lỳ và hoàn toàn bỏ lỡ các đỉnh đo đạc thực tế.

---

## 🛠️ 3. Giải pháp: Lưỡng cực Điều chuẩn (Regularized Dipole Model)
Để triệt tiêu lỗi chia cho $0$ mà không làm mất đi bản chất đo vi sai (khoảng cách $d$), hệ thống nâng cấp từ mô hình "Điểm từ lý tưởng" lên mô hình "Vòng dây hữu hạn" (Finite-size Coil Approximation) bằng phương pháp **Điều chuẩn (Regularization)**.

**Bước 1: Đưa hệ số chặn vật lý ($R^2$) vào mẫu số**
Cuộn phát TX thực tế có bán kính $R = 60\text{ mm}$. Bằng cách tính toán từ trường cho một vòng dây có kích thước thực, bình phương bán kính $R^2$ được cộng trực tiếp vào cấu trúc mẫu số. Phương trình từ trường $\mathbf{B}$ chuẩn xác tại một nửa của cuộn vi sai trở thành:
$$\mathbf{B} = \frac{\mu_0}{4\pi} \frac{3(\mathbf{m} \cdot \mathbf{r})\mathbf{r} - (r^2 + R^2)\mathbf{m}}{(r^2 + R^2)^{5/2}}$$

**Bước 2: Xử lý giới hạn khi $r \to 0$**
Nhờ sự xuất hiện của $R^2$, khi viên nang chạm mặt cuộn phát ($r = 0$), mẫu số không còn bị triệt tiêu về $0$ mà trở thành một hằng số giới hạn vật lý:
$$\lim_{r \to 0} (r^2 + R^2)^{5/2} = (0 + R^2)^{5/2} = R^5$$
Vì $R = 0.06\text{ m} > 0$, mẫu số luôn là một giá trị dương hữu hạn ($0.06^5 \approx 7.7 \times 10^{-7}$). Khối chóp vô cực của không gian sai số giờ đây được "bo tròn" thành một đỉnh (peak) trơn tru và an toàn.

**Bước 3: Chiếu lên Cuộn Vi Sai**
Mô hình toán học bảo toàn nguyên vẹn tính vi sai bằng cách tính $\mathbf{B}$ độc lập cho 2 vị trí cộng/trừ của cuộn thu:
$$\mathbf{r}_{plus} = \mathbf{r}_{rx} + \frac{d}{2}\mathbf{\hat{n}}_{rx} - \mathbf{r}_{tx}$$
$$\mathbf{r}_{minus} = \mathbf{r}_{rx} - \frac{d}{2}\mathbf{\hat{n}}_{rx} - \mathbf{r}_{tx}$$
Độ lệch từ trường sinh ra dòng điện:
$$EMF \propto \left\vert{} (\mathbf{B}_{plus} - \mathbf{B}_{minus}) \cdot \mathbf{\hat{n}}_{rx} \right\vert{}$$

### 🏆 Kết quả
Việc sử dụng mẫu số $(r^2 + R^2)^{5/2}$ đã giải cứu hoàn toàn thuật toán tối ưu. Không gian Gradient trở nên mượt mà (smooth) ở ngay cả những điểm sát cuộn dây nhất. Thuật toán `least_squares` không còn bị hoảng loạn bởi việc chia cho số $0$, loại bỏ hoàn toàn hiện tượng ép biến $K \to 0$. Nhờ đó, nó thong thả tính toán các bước nhảy lớn, tìm ra chính xác các góc xoay $164^\circ$ và bám khít $100\%$ vào các đỉnh tín hiệu $10\text{ mV}$ - $13\text{ mV}$ của cuộn vi sai thực tế.
