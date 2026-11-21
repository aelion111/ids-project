# Phát hiện xâm nhập mạng bằng học máy trên dữ liệu mất cân bằng

Project 3, Trường Công nghệ Thông tin và Truyền thông (SoICT), Đại học Bách khoa Hà Nội.
Giảng viên hướng dẫn: TS. Trần Hoàng Hải. Sinh viên: Trần Quang Huy (20215591).

Phân loại flow mạng của bộ dữ liệu CIC-IDS2017 thành 9 lớp (BENIGN và 8 lớp tấn công). Dữ liệu mất cân bằng mạnh: trong tập huấn luyện, BENIGN có 297.985 mẫu, Heartbleed có 9. Project so sánh 4 cách xử lý mất cân bằng trên 4 mô hình và đánh giá bằng macro-F1 và recall từng lớp thay cho accuracy.

## Dữ liệu

CIC-IDS2017 (Sharafaldin et al., 2018), bản CSV đã trích đặc trưng bằng CICFlowMeter: 8 file, 2.830.743 flow, 78 đặc trưng và nhãn.

1. Tải `MachineLearningCSV.zip` tại https://www.unb.ca/cic/datasets/ids-2017.html.
2. Giải nén vào `data/`, được thư mục `data/MachineLearningCVE/` chứa 8 file CSV.

Dữ liệu không nằm trong repo (khoảng 844 MB).

## Cài đặt

Python 3.12.

```bash
python3.12 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Trên macOS, LightGBM cần OpenMP: `brew install libomp`.

## Chạy

Mở các notebook trong thư mục `notebooks/` bằng Jupyter hoặc VS Code (kernel `.venv`) và chạy theo thứ tự:

| Notebook | Nội dung | Đầu ra |
|---|---|---|
| `01_data_exploration` | Phân phối 15 nhãn gốc, thống kê lỗi dữ liệu | — |
| `02_cleaning_and_split` | Làm sạch, gộp thành 9 lớp, chia train/test | `data/processed/train.pkl`, `test.pkl` |
| `03_baseline` | Random Forest không xử lý mất cân bằng | — |
| `04_experiments` | Ma trận thí nghiệm 4 cách xử lý × 4 mô hình | `results/results.csv`, `results/predictions/` |
| `05_analysis` | Phân tích kết quả (đang thực hiện) | Hình vẽ |

Notebook 02 cần khoảng 4 GB RAM. Mọi bước ngẫu nhiên dùng `random_state=42`, nên chạy lại cho cùng kết quả.

## Phương pháp

### Tiền xử lý (notebook 02)

- Gộp 15 nhãn thành 9 lớp: 4 kiểu DoS thành `DoS`, FTP-Patator và SSH-Patator thành `Brute Force`, 3 kiểu Web Attack thành `Web Attack`.
- Bỏ `Destination Port`, một cột bản sao và 8 cột hằng số, còn 68 đặc trưng. Cổng đích phản ánh cách dựng kịch bản tấn công trong lab (toàn bộ Heartbleed và Infiltration ở cổng 444), không phải hành vi của flow.
- Xóa 2.867 dòng có NaN hoặc Infinity, đều là flow có thời lượng bằng 0.
- Xóa dòng trùng và 309 nhóm dòng giống hệt nhau về đặc trưng nhưng khác nhãn, **trước khi chia** train/test. Do đã bỏ cột cổng, PortScan giảm từ 158.930 xuống 1.850 mẫu: các flow quét cổng chỉ khác nhau ở số cổng.
- Giữ ngẫu nhiên 20% BENIGN và toàn bộ mẫu tấn công, còn 708.665 flow.
- Chia train/test 80/20, phân tầng theo lớp. Không dòng test nào giống hệt một dòng train.

| Lớp | Dữ liệu gốc | Train | Test |
|---|---:|---:|---:|
| BENIGN | 2.273.097 | 297.985 | 74.497 |
| DoS | 252.661 | 154.868 | 38.717 |
| DDoS | 128.027 | 102.409 | 25.602 |
| Brute Force | 13.835 | 7.320 | 1.830 |
| Web Attack | 2.180 | 1.714 | 429 |
| PortScan | 158.930 | 1.480 | 370 |
| Bot | 1.966 | 1.118 | 279 |
| Infiltration | 36 | 29 | 7 |
| Heartbleed | 11 | 9 | 2 |
| **Tổng** | 2.830.743 | 566.932 | 141.733 |

### Thí nghiệm (notebook 04)

Mỗi cấu hình là một `Pipeline` của imbalanced-learn: `StandardScaler` → bước lấy mẫu lại (nếu có) → mô hình. Bước lấy mẫu lại chỉ chạy khi huấn luyện, tập test không bị thay đổi.

| Cách xử lý | Thực hiện |
|---|---|
| Không xử lý | Huấn luyện trên dữ liệu gốc |
| Class weight | `class_weight="balanced"` (KNN không hỗ trợ) |
| Undersampling | `RandomUnderSampler`, mỗi lớp tối đa 50.000 mẫu (cắt BENIGN, DoS, DDoS) |
| SMOTE | Mỗi lớp tối thiểu 5.000 mẫu (bổ sung PortScan, Web Attack, Bot, Infiltration, Heartbleed) |

Mô hình: Logistic Regression (`max_iter=1000`), KNN (k = 5), Random Forest (100 cây), LightGBM (`max_delta_step=1`, vì với tham số mặc định mô hình phân kỳ trên dữ liệu này). Các tham số khác để mặc định. Tổng cộng 15 cấu hình.

Chỉ số chính là macro-F1 và recall từng lớp. Accuracy chỉ để tham khảo vì BENIGN, DoS và DDoS chiếm khoảng 98% tập test.

## Kết quả

Đã chạy 8 cấu hình của LightGBM và Random Forest; Logistic Regression và KNN chưa chạy. Heartbleed được nhận đúng 2/2 ở mọi cấu hình.

| Mô hình | Cách xử lý | Macro-F1 | Recall Web Attack | Recall Bot | Precision Bot | Infiltration | Train (giây) |
|---|---|---:|---:|---:|---:|---:|---:|
| LightGBM | Không xử lý | 0,9801 | 0,9907 | 0,9462 | 0,8980 | 6/7 | 20,9 |
| LightGBM | Class weight | 0,9797 | 0,9977 | 0,9964 | 0,8554 | 6/7 | 21,2 |
| LightGBM | Undersampling | 0,9791 | 0,9953 | 0,9857 | 0,8594 | 6/7 | 11,3 |
| LightGBM | SMOTE | 0,9684 | 0,9930 | 0,9642 | 0,8622 | 5/7 | 23,8 |
| Random Forest | Không xử lý | 0,9633 | 0,9534 | 0,8315 | 0,9317 | 5/7 | 22,2 |
| Random Forest | Class weight | 0,9538 | 0,9837 | 0,9785 | 0,5460 | 6/7 | 13,3 |
| Random Forest | Undersampling | 0,9651 | 0,9720 | 0,9211 | 0,8712 | 5/7 | 6,6 |
| Random Forest | SMOTE | 0,9752 | 0,9580 | 0,8817 | 0,9111 | 6/7 | 31,1 |

Thời gian đo trên MacBook 16 GB RAM.

Khi so sánh macro-F1, lưu ý tập test chỉ có 7 mẫu Infiltration: nhận đúng thêm một mẫu (5/7 lên 6/7) làm macro-F1 tăng khoảng 0,01. Chênh lệch cỡ đó giữa hai cấu hình có thể chỉ do một flow.

## Hạn chế

- Lớp hiếm có rất ít mẫu test (Infiltration 7, Heartbleed 2), nên recall của chúng dao động mạnh.
- Tập test chỉ giữ 20% BENIGN, nên precision của các lớp tấn công lạc quan hơn khi triển khai thực tế.
- Tham số mô hình, mức trần và mức sàn không được tinh chỉnh; kết quả dựa trên một lần chia dữ liệu và một bộ dữ liệu.

## Cấu trúc thư mục

```
├── data/                  dữ liệu (không đưa lên Git)
│   ├── MachineLearningCVE/    8 file CSV gốc
│   └── processed/             train.pkl, test.pkl
├── notebooks/             01–05
├── results/
│   ├── results.csv            mỗi cấu hình một dòng
│   └── predictions/           nhãn dự đoán trên tập test (không đưa lên Git)
└── requirements.txt
```

## Tài liệu tham khảo

- I. Sharafaldin, A. H. Lashkari, A. A. Ghorbani, "Toward Generating a New Intrusion Detection Dataset and Intrusion Traffic Characterization," ICISSP 2018, pp. 108–116. doi:10.5220/0006639801080116
- G. Engelen, V. Rimmer, W. Joosen, "Troubleshooting an Intrusion Detection Dataset: the CICIDS2017 Case Study," IEEE SPW 2021, pp. 7–12. doi:10.1109/SPW53761.2021.00009
- H. He, E. A. Garcia, "Learning from Imbalanced Data," IEEE TKDE 21(9):1263–1284, 2009. doi:10.1109/TKDE.2008.239
- N. V. Chawla et al., "SMOTE: Synthetic Minority Over-sampling Technique," JAIR 16:321–357, 2002. doi:10.1613/jair.953
- G. Lemaître, F. Nogueira, C. K. Aridas, "Imbalanced-learn: A Python Toolbox to Tackle the Curse of Imbalanced Datasets in Machine Learning," JMLR 18(17):1–5, 2017.
- L. Breiman, "Random Forests," Machine Learning 45(1):5–32, 2001. doi:10.1023/A:1010933404324
- G. Ke et al., "LightGBM: A Highly Efficient Gradient Boosting Decision Tree," NeurIPS 2017, pp. 3146–3154.
