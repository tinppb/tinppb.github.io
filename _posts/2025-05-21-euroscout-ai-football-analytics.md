---
title: "EuroScout AI — Phân tích cầu thủ Big 5 giải VĐQG châu Âu mùa 2025–2026"
date: 2025-05-21 22:00:00 +0700
categories: [Projects, Data Analytics]
tags: [javascript, python, football, data-analytics, sofascore, machine-learning, web-scraping]
---

# EuroScout AI — Interactive Football Player Analytics Dashboard

## Giới thiệu

Trong thế giới bóng đá hiện đại, dữ liệu đóng vai trò ngày càng quan trọng. Từ việc các CLB lớn sử dụng data để tìm kiếm tài năng trẻ, đến việc fan hâm mộ tranh luận ai là cầu thủ xuất sắc nhất — tất cả đều cần số liệu khách quan làm nền tảng.

Với ý tưởng đó, mình đã xây dựng [EuroScout AI](https://euro-scout-ai.vercel.app/) — một hệ thống hoàn chỉnh gồm 2 phần:

1. Data Pipeline: Thu thập tự động 60+ chỉ số thống kê của hàng nghìn cầu thủ từ 5 giải đấu hàng đầu châu Âu (Premier League, La Liga, Serie A, Bundesliga, Ligue 1) mùa giải 2025/2026 thông qua API nội bộ của Sofascore.
2. Analytics Dashboard: Một ứng dụng web SPA (Single Page Application) cho phép tìm kiếm cầu thủ tương tự, so sánh chỉ số head-to-head, khám phá dữ liệu qua biểu đồ scatter, xếp hạng và heatmap — tất cả chạy 100% client-side.

Bài viết này sẽ đi sâu vào cả hai phần: cách thu thập dữ liệu và cách biến dữ liệu thô thành insight thông qua dashboard tương tác.

![EuroScout AI Dashboard](/assets1/img/euroscout/dashboard-demo.png)

---

## Phần 1: Thu thập dữ liệu — Top 5 Leagues Scraper

### 1.1 Bài toán

Các trang thống kê thể thao lớn (Sofascore, FBref, WhoScored,...) đều bảo vệ dữ liệu rất nghiêm ngặt bằng các công nghệ chống Bot như Cloudflare, Akamai. Việc sử dụng Selenium truyền thống gặp nhiều hạn chế: chậm, nặng, và dễ bị phát hiện.

Giải pháp của mình là bỏ qua hoàn toàn việc parse HTML, thay vào đó gọi trực tiếp API nội bộ của Sofascore và bypass hệ thống chống Bot bằng kỹ thuật TLS Fingerprint Impersonation thông qua thư viện `curl_cffi`.

### 1.2 Kiến trúc Pipeline

```text
Terminal Command
     │
     ▼
main.py 
     │  Điều phối 5 giải đấu
     ▼
crawler/full_stats.py
     │  Quét 4 nhóm vị trí: GK → DF → MF → FW
     │  Mỗi nhóm: phân trang offset 0, 20, 40,...
     ▼
Sofascore Internal API
     │  Trả về Raw JSON (60+ fields)
     ▼
Data Transformation (ETL)
     │  Tính per90 / perMatch / total
     │  
     ▼
CSV Output (data/processed/)
     └── Premier_League_PER90_STATS.csv
     └── La_Liga_PER90_STATS.csv
     └── Serie_A_PER90_STATS.csv
     └── ...
```

### 1.3 Kỹ thuật vượt Anti-Bot

Điểm mấu chốt nằm ở việc sử dụng `curl_cffi` với tham số `impersonate="chrome124"`:

```python
from curl_cffi import requests

response = requests.get(url, headers=headers, params=params, impersonate="chrome124")
```

Thư viện này giả lập chính xác TLS fingerprint của trình duyệt Chrome 124 thật, khiến server Sofascore không thể phân biệt request của script với request từ trình duyệt thực tế. Đây là bước tiến lớn so với phương pháp Selenium truyền thống — nhanh hơn gấp 10 lần, nhẹ hơn vì không cần khởi tạo trình duyệt, và ổn định hơn.

### 1.4 Xử lý dữ liệu thông minh

Một thách thức khi thu thập dữ liệu từ Sofascore là việc quy đổi chỉ số. API chỉ trả về tổng số liệu gốc (total), nhưng trong phân tích scouting thực tế, chỉ số `per 90 phút` mới thực sự có ý nghĩa để so sánh công bằng giữa các cầu thủ có thời lượng thi đấu khác nhau.

```python
def calc(val, is_percentage=False):
    if val is None: return 0
    if is_percentage: return val  # Cột % giữ nguyên
    if acc_type == "per90" and minutes_played > 0:
        return round((val / minutes_played) * 90, 2)
    elif acc_type == "perMatch" and appearances > 0:
        return round(val / appearances, 2)
    return round(val, 2) if isinstance(val, float) else val
```

Hàm `calc()` tự động nhận biết cột nào là phần trăm (%) để giữ nguyên, cột nào cần chia trung bình. Điều này đảm bảo dữ liệu đầu ra luôn chính xác về mặt toán học.

### 1.5 Bộ dữ liệu đầu ra

Mỗi file CSV chứa hơn 60 biến số gộp chung, bao gồm:

| Nhóm | Chỉ số tiêu biểu |
|------|-------------------|
| Tấn công | Goals, xG, Total Shots, Shots on Target, Goal Conversion %, Big Chances Missed |
| Phòng ngự | Tackles, Interceptions, Clearances, Blocked Shots, Dribbled Past, Errors → Goal |
| Chuyền bóng | Assists, Key Passes, Big Chances Created, Pass Acc. %, Long Ball %, Crosses |
| Tranh chấp | Total Duels Won %, Ground Duels %, Aerial Duels %, Was Fouled |
| Kỷ luật | Yellow Cards, Red Cards, Fouls, Offsides |

Với mùa giải 2025/2026, bộ dữ liệu bao gồm hơn 2.156 cầu thủ từ 97 đội bóng thuộc 5 giải đấu.

---

## Phần 2: EuroScout AI Dashboard

### 2.1 Công nghệ sử dụng

| Thành phần | Công nghệ |
|---|---|
| Build Tool | Vite 6 |
| Language | Vanilla JavaScript (ES Modules) |
| Styling | Vanilla CSS (Custom Properties, Glassmorphism) |
| Charts| Chart.js 4.4 + chartjs-plugin-datalabels |
| Typography | Google Fonts (Inter, Outfit) |
| Data Source | Sofascore (crawled & processed) |

### 2.2 Tính năng chính

#### Similar Player Finder — Tìm cầu thủ tương tự

Đây là tính năng cốt lõi của EuroScout AI. Bạn chỉ cần nhập tên một cầu thủ (ví dụ: Bruno Fernandes), hệ thống sẽ tìm ra những cầu thủ có phong cách thi đấu tương đồng nhất từ cả 5 giải đấu.

**Thuật toán hoạt động như sau:**

1. Chọn features theo vị trí: Mỗi vị trí (FW/MF/DF/GK) sử dụng một tập chỉ số riêng biệt, phù hợp với đặc thù vai trò:

| Vị trí | Chỉ số được sử dụng |
|---|---|
| FW | Goals, xG, Shots, Dribbles, Key Passes, Aerial % |
| MF | Goals, Assists, Pass Acc.%, Tackles, Interceptions, Long Balls |
| DF | Tackles, Interceptions, Clearances, Aerial %, Duels, Pass Acc.% |
| GK | Clean Sheets, Pass Acc.%, Long Ball %, Aerial % |

2. **Z-score Normalization:** Chuẩn hóa tất cả chỉ số về cùng thang đo, loại bỏ ảnh hưởng của đơn vị khác nhau (ví dụ: Goals dao động 0-2, còn Passes dao động 0-80):

```javascript
// Z-score: z = (x - μ) / σ
for (const f of features) {
  vec[f] = ((p.stats[f] || 0) - means[f]) / stds[f];
}
```

3. **Cosine Similarity:** Tính góc giữa hai vector chỉ số trong không gian đa chiều. Giá trị càng gần 1 (100%) = càng tương đồng:

```javascript
function cosineSimilarity(vecA, vecB, features) {
  let dotProduct = 0, normA = 0, normB = 0;
  for (const f of features) {
    dotProduct += vecA[f] * vecB[f];
    normA += vecA[f] * vecA[f];
    normB += vecB[f] * vecB[f];
  }
  return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
}
```

4. **Bộ lọc linh hoạt:**
   - Cross-League Only: Chỉ tìm cầu thủ từ các giải đấu khác — rất hữu ích cho scouting xuyên giải.
   - Same League Only: Tìm trong cùng giải.
   - Lọc theo vị trí cụ thể.
   - Loại trừ cầu thủ có dưới 270 phút thi đấu (~3 trận).

Ví dụ: Tìm cầu thủ tương tự Lamine Yamal (FW, Barcelona, La Liga) ở chế độ Cross-League → hệ thống sẽ trả về những cầu thủ từ Premier League, Bundesliga, Serie A, Ligue 1 có hồ sơ chỉ số tương đồng.

#### Player Comparison — So sánh Head-to-Head

Cho phép đặt 2 cầu thủ bất kỳ cạnh nhau và so sánh trực tiếp trên:
- 7 nhóm chỉ số: Attacking, Creativity, Passing, Defending, Dribbling, Duels, Discipline
- Radar chart overlay hiển thị percentile rank (so với toàn bộ cầu thủ cùng vị trí)
- Trực quan hóa bằng thanh so sánh với highlight bên nào vượt trội

#### Scatter Explorer — Biểu đồ phân tán

Khám phá mối tương quan giữa các chỉ số qua biểu đồ scatter plot tương tác:
- 8+ presets sẵn có: xG vs Goals, Key Passes vs Assists, Tackles vs Interceptions,...
- Tùy chọn trục X/Y từ 40+ chỉ số bất kỳ
- Lọc theo giải đấu & vị trí
- Hover để xem chi tiết từng cầu thủ (dots)

Một số insight thú vị từ scatter plot:
- xG vs Goals: Cầu thủ nào đang "overperform" (ghi bàn nhiều hơn kỳ vọng)? Cầu thủ nào đang "underperform"?
- Tackles vs Interceptions: Phong cách phòng ngự proactive (nhiều tackle) vs reactive (nhiều interceptions)?
- **Dribbles vs Was Fouled:** Cầu thủ nào tạo ra nhiều tình huống phạm lỗi nhất?

#### Rankings — Bảng xếp hạng

Bảng xếp hạng cầu thủ theo **bất kỳ chỉ số nào** trong 60+ biến số. Với:
- Sắp xếp tăng/giảm
- Lọc theo giải đấu & vị trí
- Phân trang
- Highlight top players

#### Heatmap — Ma trận nhiệt

Ma trận nhiệt so sánh nhiều cầu thủ trên nhiều chỉ số cùng lúc — giúp phát hiện `pattern` và `outlier` trong dữ liệu mà các biểu đồ đơn lẻ khó nhận ra.

### 2.3 Kiến trúc ứng dụng

```text
EuroScout AI/
├── src/
│   ├── data/
│   │   ├── dataStore.js       ← Data store, utilities, stat configs
│   │   └── players.json       ← 4.3MB dataset (2,156 players)
│   ├── engine/
│   │   └── similarity.js      ← Cosine similarity + Z-score engine
│   ├── charts/
│   │   ├── radarChart.js      ← Radar/spider chart
│   │   ├── scatterChart.js    ← Scatter plot
│   │   └── heatmapChart.js    ← Heatmap matrix
│   ├── views/
│   │   ├── dashboard.js       ← Overview với league/position distribution
│   │   ├── similarFinder.js   ← Similar player finder
│   │   ├── comparison.js      ← Head-to-head comparison
│   │   ├── scatterExplorer.js ← Scatter plot explorer
│   │   ├── rankings.js        ← Player rankings table
│   │   └── heatmap.js         ← Heatmap visualization
│   ├── styles/
│   │   └── index.css          ← Full design system (glassmorphism)
│   └── main.js                ← SPA router + global search
├── scripts/
│   └── build-data.mjs         ← CSV → JSON pipeline
└── index.html                 ← Shell HTML
```

### 2.4 Data Pipeline: Từ CSV đến JSON

Dữ liệu từ Phần 1 (CSV) được chuyển đổi sang JSON qua script `build-data.mjs`:

```text
5 file CSV (mỗi giải 1 file)
    │
    ▼
build-data.mjs
    │  - Đọc & gộp 5 file CSV
    │  - Lọc cầu thủ dưới 270 phút (≈ 3 trận)
    │  - Map 44 cột CSV → JSON keys
    │  - Chuẩn hóa vị trí: Forward → FW, Midfielder → MF,...
    │  - Fix encoding đặc biệt
    ▼
players.json (4.3MB)
    └── metadata: { leagues, season, totalPlayers }
    └── players: [ { id, name, team, league, position, stats: {...} } ]
```

---

## Phần 3: Tổng kết & Hướng phát triển

### Những gì đã làm được

| Thành phần | Kết quả |
|---|---|
| Data Collection | 2,156 cầu thủ × 60+ chỉ số × 5 giải đấu |
| Similarity Engine | Cosine Similarity + Z-score, position-aware features |
| Dashboard | 6 views: Dashboard, Similar Finder, Comparison, Scatter, Rankings, Heatmap |
| UX/UI | Dark glassmorphism, responsive, micro-animations, global search |
| Performance | 100% client-side, không cần backend |

### Hướng phát triển

- API Backend (FastAPI): Tích hợp thuật toán nâng cao như PCA (giảm chiều), KMeans clustering để nhóm cầu thủ theo phong cách.
- Theo dõi phong độ theo thời gian: Crawl dữ liệu định kỳ, so sánh sự tiến bộ/suy giảm của cầu thủ.
- Tích hợp dữ liệu chuyển nhượng: Kết hợp giá trị thị trường (Transfermarkt) với chỉ số hiệu suất.
- AI-powered scouting report: Sử dụng LLM để tự động sinh báo cáo scouting bằng ngôn ngữ tự nhiên.

### Source Code

Toàn bộ mã nguồn của dự án được công khai trên GitHub:
- **Data Crawler:** [Top5-Leagues-Scraper-25-26](https://github.com/tinppb/Top5-Leagues-Scraper-25-26)
- **Analytics Dashboard:** [EuroScout-AI](https://github.com/tinppb/EuroScout-AI)

---

## Liên hệ
Nếu có bất kỳ câu hỏi hay đóng góp, hãy liên hệ qua email hoặc tạo issue trên GitHub.

---
## Người phát triển
**Phạm Phước Bảo Tín (tinppb)**  
Email: [baotinphamphuoc@gmail.com](mailto:baotinphamphuoc@gmail.com)  
