# Dữ liệu đo EMF – định vị viên nang nội soi bằng cảm biến từ

File dữ liệu: `conenorot_c_223_data.csv`

Dữ liệu đo EMF của 3 cuộn thu (RX) trong viên nang, khi 3 cuộn phát (TX) cố định phát ở 3 tần số khác nhau. Kèm theo là vị trí và góc Euler của viên nang do hệ Ground Truth (GT) ghi lại tại cùng thời điểm.

## 1. Cấu trúc file

- Định dạng: CSV, **không có dòng tiêu đề**, ngăn cách bằng dấu phẩy.
- Kích thước: **3000 dòng × 15 cột**, không có giá trị thiếu.
- Mỗi dòng là một lần đo.

### Bảng các cột (đánh số từ 0)

| Cột | Ký hiệu | Ý nghĩa | Đơn vị |
|---|---|---|---|
| 0 | TX1–RX1 | EMF cuộn thu RX1 khi TX1 phát (5053 Hz) | V |
| 1 | TX2–RX1 | EMF cuộn thu RX1 khi TX2 phát (6911 Hz) | V |
| 2 | TX3–RX1 | EMF cuộn thu RX1 khi TX3 phát (5932 Hz) | V |
| 3 | TX1–RX2 | EMF cuộn thu RX2 khi TX1 phát | V |
| 4 | TX2–RX2 | EMF cuộn thu RX2 khi TX2 phát | V |
| 5 | TX3–RX2 | EMF cuộn thu RX2 khi TX3 phát | V |
| 6 | TX1–RX3 | EMF cuộn thu RX3 khi TX1 phát | V |
| 7 | TX2–RX3 | EMF cuộn thu RX3 khi TX2 phát | V |
| 8 | TX3–RX3 | EMF cuộn thu RX3 khi TX3 phát | V |
| 9 | x | Vị trí viên nang theo trục x (GT) | mm |
| 10 | y | Vị trí viên nang theo trục y (GT) | mm |
| 11 | z | Vị trí viên nang theo trục z (GT) | mm |
| 12 | roll | **Góc Euler ban đầu của robot** quanh trục x | độ |
| 13 | pitch | **Góc Euler ban đầu của robot** quanh trục y | độ |
| 14 | yaw | **Góc Euler ban đầu của robot** quanh trục z | độ |

Quy tắc chỉ số cột EMF: **`cột = (số TX − 1) + 3 × (số RX − 1)`**.
Ví dụ: TX2–RX3 → `(2−1) + 3×(3−1) = 7`.

### Lưu ý về 3 cột cuối (góc Euler)

- Cột 12, 13, 14 là **góc Euler ban đầu của robot** (roll, pitch, yaw), không phải góc viên nang thay đổi trong lúc đo.
- Trong file này robot **không xoay** khi di chuyển. Ba giá trị gần như không đổi trên toàn bộ file, khoảng (175.82°, −0.79°, 33.15°). Sai khác nhỏ giữa các dòng (vài phần trăm độ) là độ phân giải của hệ GT.
- Vì góc ban đầu khác 0, **không nên trừ góc của mẫu đầu** khi xử lý, nếu không sẽ mất thông tin hướng của viên nang.
- Thứ tự phép xoay dùng trong code: `R = Rz(yaw) · Ry(pitch) · Rx(roll)`.

## 2. Quy ước đo: mỗi vị trí đo 10 lần

Tại mỗi vị trí, viên nang đứng yên và EMF được đo **10 lần liên tiếp** để giảm nhiễu. Vì vậy:

- Cứ **10 dòng liên tiếp** là cùng một vị trí. Dòng 0–9 là vị trí 1, dòng 10–19 là vị trí 2, và cứ thế tiếp tục.
- Trong mỗi nhóm 10 dòng, các cột 9–14 (vị trí và góc) **giống hệt nhau**. Chỉ 9 cột EMF (cột 0–8) khác nhau chút ít giữa các lần đo.
- Tổng cộng: **300 vị trí × 10 lần đo = 3000 dòng**.
- Quỹ đạo theo thứ tự đo: x từ khoảng −140 đến −44 mm, y từ khoảng 334 đến 426 mm, z từ khoảng 147 đến 246 mm.

```
dòng  0..9    → vị trí 1   (10 lần đo)
dòng 10..19   → vị trí 2   (10 lần đo)
...
dòng 2990..2999 → vị trí 300 (10 lần đo)
```

## 3. Hệ trục thiết bị

| Cuộn thu | Hướng trục trong viên nang |
|---|---|
| RX1 | y |
| RX2 | z |
| RX3 | x |

Cuộn TX: 3 cuộn cố định, tần số TX1 = 5053 Hz, TX2 = 6911 Hz, TX3 = 5932 Hz.

## 4. Cách đọc dữ liệu và lấy trung bình 10 lần đo

```python
import numpy as np
import pandas as pd

N_REP = 10  # số lần đo tại mỗi vị trí

raw = pd.read_csv('conenorot_c_223_data.csv', header=None).values   # (3000, 15)
assert raw.shape[0] % N_REP == 0

# Tách theo nhóm: (300 vị trí, 10 lần đo, 15 cột)
groups = raw.reshape(-1, N_REP, raw.shape[1])

# Trung bình 10 lần đo -> 300 mẫu
data = groups.mean(axis=1)               # (300, 15)

emf = data[:, 0:9]                       # 9 kênh EMF
pos_mm = data[:, 9:12]                   # x, y, z (mm)
euler_deg = data[:, 12:15]               # roll, pitch, yaw ban đầu (độ)

# Độ lệch chuẩn giữa 10 lần đo (đánh giá nhiễu)
emf_std = groups[:, :, 0:9].std(axis=1)  # (300, 9)

# Lấy EMF của TX2–RX3 (cột 7)
tx, rx = 2, 3
col = (tx - 1) + 3 * (rx - 1)
emf_tx2_rx3 = emf[:, col]
```

Nếu muốn dùng riêng từng lần đo (ví dụ để phân tích nhiễu), dùng `groups[:, k, :]` với `k = 0..9`.

## 5. Chất lượng dữ liệu (kiểm tra trên file này)

- Độ lệch chuẩn EMF giữa 10 lần đo tại cùng vị trí: khoảng 0.01 V, rất nhỏ so với EMF trung bình (khoảng 0.2 đến 2.3 V).
- Không thấy trôi hệ thống theo thứ tự lần đo trong mỗi nhóm (lần đo thứ 1 đến 10 có trung bình gần như bằng nhau).
- Khoảng giá trị EMF trong file: từ 0.001 đến 12.9 V.

## 6. Lưu ý khi sử dụng

- Đơn vị EMF ghi là V theo cách code đang dùng. Nếu cần so sánh tuyệt đối với mô hình vật lý, cần xác nhận lại hệ số khuếch đại mạch đo.
- Vị trí GT có độ phân giải cỡ vài phần mười mm, đây là sai số nền của mọi kết quả định vị.
- Các tham số hiệu chuẩn (K, pose TX, góc trục RX) tính trên file này đánh giá **trong mẫu**, chưa kiểm tra độc lập trên quỹ đạo khác.
