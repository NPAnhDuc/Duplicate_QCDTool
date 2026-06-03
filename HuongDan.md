# 📘 Hướng Dẫn Sử Dụng — QCD Tool Vector DB

> **Công cụ phát hiện bug ticket trùng lặp** cho dự án VinFast  
> Sử dụng ChromaDB + Gemini Embedding + AI Reranking

---

## 📋 Mục Lục

1. [Yêu cầu cài đặt](#1-yêu-cầu-cài-đặt)
2. [Cấu hình file .env](#2-cấu-hình-file-env)
3. [Khởi động ứng dụng](#3-khởi-động-ứng-dụng)
4. [Lần đầu sử dụng — Đồng bộ Database](#4-lần-đầu-sử-dụng--đồng-bộ-database)
5. [Chạy phân tích tương đồng](#5-chạy-phân-tích-tương-đồng)
6. [Đọc hiểu kết quả](#6-đọc-hiểu-kết-quả)
7. [Cấu hình nâng cao (Sidebar)](#7-cấu-hình-nâng-cao-sidebar)
8. [Quản lý Database](#8-quản-lý-database)
9. [Xử lý lỗi thường gặp](#9-xử-lý-lỗi-thường-gặp)
10. [Cơ chế hoạt động](#10-cơ-chế-hoạt-động)

---

## 1. Yêu cầu cài đặt

### Môi trường
- Python **3.9+**
- Kết nối **VPN VinFast** (bắt buộc để truy cập Jira `tms-uat.vinfast.vn`)

### Cài thư viện
```bash
pip install streamlit google-generativeai chromadb pandas openpyxl requests python-dotenv
```

### Cấu trúc thư mục
```
project/
├── appVTDB.py      ← file ứng dụng chính
├── .env                 ← file cấu hình API keys (tự tạo)
└── chroma_db/           ← tự động tạo khi sync lần đầu
```

---

## 2. Cấu hình file .env

Tạo file `.env` cùng thư mục với `app_appVTDB.py`:

```env
GEMINI_API_KEY=AIzaSy...your_key_here
JIRA_API_TOKEN=your_jira_bearer_token
JIRA_URL=https://tms-uat.vinfast.vn
JIRA_PROJECT_KEY=VF6
```

| Biến | Mô tả | Lấy ở đâu |
|------|-------|-----------|
| `GEMINI_API_KEY` | API Key của Google Gemini | [Google AI Studio](https://aistudio.google.com/apikey) |
| `JIRA_API_TOKEN` | Bearer token Jira VinFast | Jira → Profile → Personal Access Tokens |
| `JIRA_URL` | URL Jira server | `https://tms-uat.vinfast.vn` |
| `JIRA_PROJECT_KEY` | Project key Jira | Ví dụ: `VF6` |

> ⚠️ **Bảo mật:** Không bao giờ commit file `.env` lên Git. Thêm `.env` vào `.gitignore`.

---

## 3. Khởi động ứng dụng

```bash
# Di chuyển vào thư mục project
cd /path/to/project

# Chạy ứng dụng
streamlit run appVTDB.py
```

Trình duyệt sẽ tự mở tại `http://localhost:8501`

---

## 4. Lần đầu sử dụng — Đồng bộ Database

> Bước này **bắt buộc** trước khi chạy phân tích. Chỉ cần làm **1 lần** (hoặc khi cần cập nhật master list).

### Bước 4.1 — Vào tab Quản lý Database

Nhấn tab **🗄️ Quản lý Database** ở trên cùng.

### Bước 4.2 — Kiểm tra JQL Query

Ở sidebar bên trái, mục **JQL Query**, mặc định là:
```
project = "VF6" ORDER BY created DESC
```

Có thể tùy chỉnh để lọc theo sprint, ngày, loại issue, v.v.:
```
project = "VF6" AND issuetype = Bug AND created >= -90d ORDER BY created DESC
```

### Bước 4.3 — Tải dữ liệu từ Jira

Nhấn nút **⬇️ Tải dữ liệu Jira (từ JQL)**.

- Ứng dụng sẽ kết nối Jira qua API và tải toàn bộ tickets khớp JQL
- Dữ liệu hiển thị dạng bảng để xem trước
- Có thể nhấn **📥 Tải về Excel** để lưu bản sao

### Bước 4.4 — Đồng bộ vào Vector DB

Nhấn nút **🔄 Đồng bộ dữ liệu đã tải → DB (upsert)**.

- Ứng dụng tự động tách `Description` mỗi ticket thành các vùng: **Actual Result**, **Expected**, **Steps**
- Tạo embedding vector bằng Gemini và lưu vào ChromaDB
- Quá trình mất khoảng **1–3 phút** tùy số lượng ticket

✅ Khi hoàn tất sẽ hiện: `Upserted X tickets. DB hiện có X tickets.`

> 💡 **Tip:** Chạy lại upsert bất cứ lúc nào để cập nhật ticket mới — ticket cũ không bị trùng (ID = Jira Key).

---

## 5. Chạy phân tích tương đồng

### Bước 5.1 — Chuẩn bị file Excel đầu vào

File Excel cần có các cột sau (không phân biệt hoa/thường):

| Cột | Bắt buộc | Mô tả |
|-----|----------|-------|
| `Summary` | ✅ | Tiêu đề bug ticket mới |
| `Description` | ✅ | Mô tả chi tiết (Actual/Expected/Steps) |
| `Market` hoặc `Markets` | ➖ Khuyến nghị | Thị trường: KZ, VN, EU, ME, UAE… |

> Nếu không có cột Market, tool sẽ tự trích xuất từ Summary/Description nếu có tag `[EU]`, `[VN]`...

### Bước 5.2 — Upload file

Ở sidebar bên trái, nhấn **📂 Upload New List** và chọn file `.xlsx` hoặc `.csv`.

### Bước 5.3 — Điều chỉnh tham số (tuỳ chọn)

| Tham số | Mặc định | Ý nghĩa |
|---------|----------|---------|
| **Ngưỡng tương đồng** | 60% | Chỉ hiện kết quả có score ≥ ngưỡng |
| **Top-K candidates** | 5 | Số lượng ticket từ DB đưa vào AI xét |

### Bước 5.4 — Chạy phân tích

Nhấn nút **🚀 Chạy Phân Tích** ở sidebar.

Tiến trình hiển thị theo từng ticket. Thời gian xử lý: ~**3–8 giây/ticket** (phụ thuộc Gemini API).

---

## 6. Đọc hiểu kết quả

### 6.1 — Bảng tổng quan (Metrics)

| Thẻ | Màu | Ý nghĩa |
|-----|-----|---------|
| 🔴 **DUPLICATE** | Đỏ | Score ≥ 88% — Rất nhiều khả năng đã có trong DB |
| 🟡 **NEAR DUP** | Vàng | Score 70–87% — Tương tự, cần xem xét kỹ |
| 🔵 **SIMILAR** | Xanh dương | Score 50–69% — Có điểm chung, nhưng khác biệt rõ |
| 🟢 **NEW (Sạch)** | Xanh lá | Score < ngưỡng — Chưa có trong DB, đủ điều kiện tạo mới |

### 6.2 — Chi tiết từng ticket

Nhấn vào từng dòng để xem chi tiết:

```
🔴 DUPLICATE  [88%]  [VN] MHU display wrong speed after IGN cycle

  🆕 NEW Ticket                    🔗 JIRA Match: VF6LHD-12345
  Summary: ...                     Market (Jira): VN
  Market: VN                       Score: 88%  Classification: DUPLICATE
  Observation: ...

  📋 Lý do match:
  [Actual Result]: MHU hiển thị sai tốc độ ≈ MHU show incorrect speed value
  [Expected Result]: Speed displayed correctly after IGN ON
  [Keywords matched]: MHU, speed, IGN, display, incorrect
  [Conclusion]: DUPLICATE — cùng component MHU, cùng hành vi hiển thị sai tốc độ sau IGN
```

### 6.3 — Score có điều chỉnh Market

Nếu ticket mới và ticket Jira **khác market**, score gốc bị trừ 20 điểm:

```
Score: 72% (gốc 92% -20pts market)
```

> Nghĩa là AI đánh 92% tương đồng về kỹ thuật, nhưng khác market → hạ xuống 72% (NEAR_DUP).

### 6.4 — Tải kết quả

Nhấn **📥 Tải kết quả (.xlsx)** ở cuối trang để export toàn bộ ra file Excel.

---

## 7. Cấu hình nâng cao (Sidebar)

### Ngưỡng tương đồng (%)

- **40–55%**: Bắt nhiều hơn, dễ có false positive
- **60%** *(mặc định)*: Cân bằng giữa bắt trùng và độ chính xác
- **70–90%**: Chỉ hiện kết quả rất chắc chắn

### Top-K candidates

- **3**: Nhanh hơn, ít nhiễu hơn, có thể bỏ sót
- **5** *(mặc định)*: Cân bằng
- **8–10**: Phủ rộng hơn, AI tốn thêm thời gian rerank

> 💡 Với ticket có description ngắn hoặc thiếu Actual Result, nên tăng Top-K lên 7–8.

---

## 8. Quản lý Database

### Xem trạng thái DB

Tab **🗄️ Quản lý Database** hiển thị:
- Số lượng tickets đang lưu
- Thời điểm sync gần nhất
- Mẫu 3 Jira Key gần nhất

### Xem mẫu dữ liệu

Nhấn **👁️ Xem mẫu dữ liệu trong DB** để kiểm tra 10 records đầu tiên — hữu ích để xác nhận dữ liệu đã sync đúng.

### Kiểm tra Embedding Models

Nếu gặp lỗi `404` liên quan embedding, nhấn **🔎 Kiểm tra Embedding Models khả dụng** → **Liệt kê models** để xem model nào đang hoạt động với API Key hiện tại.

### Xóa và Rebuild DB

Nhấn **🗑️ Xoá DB** khi cần:
- Đổi Gemini API Key (embedding dimension có thể thay đổi)
- Dữ liệu Jira đã thay đổi lớn và cần sync lại toàn bộ
- Phát hiện dữ liệu bị corrupt

> ⚠️ Sau khi xóa, bắt buộc phải sync lại từ đầu (Bước 4).

---

## 9. Xử lý lỗi thường gặp

### ❌ `Thiếu GEMINI_API_KEY hoặc JIRA_API_TOKEN`
- Kiểm tra file `.env` có đúng thư mục không
- Mở terminal mới và chạy lại `streamlit run app_vecterDB.py`

### ❌ `Jira API lỗi 401 / 403`
- Token Jira đã hết hạn → vào Jira tạo token mới, cập nhật `.env`
- Kiểm tra kết nối VPN VinFast

### ❌ `Jira API lỗi 404`
- Kiểm tra `JIRA_URL` trong `.env` — không có dấu `/` ở cuối
- Kiểm tra `JIRA_PROJECT_KEY` có đúng không

### ❌ `Không có model embedding nào hoạt động`
- Vào tab **🗄️ Quản lý Database** → **Kiểm tra Embedding Models**
- Xem model nào khả dụng và cập nhật `MODELS_TO_TRY` trong code nếu cần

### ⚠️ `Hết Quota (429)` — AI rerank
- Gemini API đạt giới hạn request/phút
- Tool tự động đợi và retry — không cần can thiệp
- Nếu thường xuyên xảy ra: giảm Top-K xuống 3–4

### ⚠️ Vector DB trả kết quả nhiễu / score quá cao
- Xóa DB và sync lại (embedding format đã được cập nhật)
- Giảm Top-K xuống 3
- Tăng Ngưỡng tương đồng lên 65–70%

---

## 10. Cơ chế hoạt động

```
File Excel mới
      │
      ▼
[1] Trích xuất vùng văn bản
    extract_zones() — tách Actual Result, Expected, Steps từ Description

      │
      ▼
[2] Tạo Query Vector
    build_vector_document() — chỉ dùng Actual Result + Summary
    → Gemini Embedding API → vector 768 chiều

      │
      ▼
[3] Vector Search (ChromaDB)
    Query top-K candidates có cosine distance ≤ 0.45
    Ưu tiên filter cùng Market trước

      │
      ▼
[4] AI Reranking (Gemini LLM)
    So sánh chi tiết NEW ticket vs từng candidate
    Chấm điểm 0–100 dựa trên:
      • Actual Result (55%)
      • Summary (20%)
      • Expected (15%)
      • Procedure (10%)

      │
      ▼
[5] Điều chỉnh & Phân loại
    • Trừ 20 điểm nếu khác Market
    • DUPLICATE ≥ 88% | NEAR_DUP ≥ 70% | SIMILAR ≥ 50% | NEW < 50%

      │
      ▼
[6] Kết quả & Export Excel
```

### Tại sao chỉ embed Actual Result?

`Expected Result` và `Steps to Reproduce` thường **giống nhau** giữa nhiều ticket cùng feature (ví dụ cùng test case HWA). Nếu embed cả 3 trường, cosine similarity tăng giả tạo → nhiễu. Chỉ embed `Actual Result` + `Summary` đảm bảo vector phản ánh đúng **hành vi lỗi thực sự**.

---

*Cập nhật lần cuối: 2026-06-03 | Phiên bản: appVTDB v2 (patched)*