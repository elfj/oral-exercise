# 口咽肌肉功能訓練 — Webcam 即時偵測系統

透過 Webcam 即時偵測臉部動作，引導使用者完成口咽肌肉功能訓練，並自動計算完成次數。

## Demo

部署後直接在瀏覽器開啟即可使用（需允許相機權限）。

## 支援的訓練動作

| 動作 | 偵測方式 | 計次邏輯 |
|------|---------|---------|
| 🌍 戽斗星球 | 下顎前突（jawForward） | 維持 3 秒 = 1 次 |
| 🐸 青蛙捕食 | 張嘴 + 下唇下拉 + 下唇伸展（複合分數） | 張嘴→閉嘴 = 1 次 |
| 🐙 章魚吸盤 | 嘟嘴（mouthPucker） | 嘟嘴→放鬆 = 1 次 |
| 🐡 河豚鼓鼓 | 臉頰鼓起（cheekPuff） | 鼓起→放鬆 = 1 次 |
| 😁 一笑呷百二 | 左右嘴角上揚（mouthSmile） | 維持 3 秒 = 1 次 |

## 功能

- **兩種模式**：單項練習（自由選擇動作）和完整課程（依序完成全部 5 個動作）
- **即時偵測**：MediaPipe Face Landmarker blendshape 分析，30fps 偵測
- **動畫示範**：每個動作附有 SVG 動畫，邊看邊做
- **視覺回饋**：偵測狀態指示燈、分數條、維持時間進度條、完成閃光
- **成績統計**：完成次數與用時統計
- **Debug 面板**：即時顯示所有嘴巴相關 blendshape 數值（開發 / 校準用）

## 技術架構

- **前端框架**：純 HTML / CSS / JavaScript（單一檔案，零依賴）
- **臉部偵測**：[MediaPipe Face Landmarker](https://ai.google.dev/edge/mediapipe/solutions/vision/face_landmarker)（tasks-vision v0.10.14）
- **偵測資料**：52 個 ARKit 相容 blendshape
- **動畫**：SVG SMIL 原生動畫
- **部署**：Vercel（靜態網站託管）

### 偵測流程

```
Webcam → MediaPipe Face Landmarker → 52 Blendshapes
    → Smoother（EMA 平滑）→ ExerciseDetector（狀態機）
    → 計次 + UI 更新
```

### 偵測機制

- **EMA 平滑**：alpha=0.5，降低逐幀雜訊
- **遲滯閾值**：enter 與 exit 使用不同閾值，防止邊界抖動
- **Cooldown**：計次後短暫冷卻期，防止重複計算
- **Toggle 型**（青蛙、章魚、河豚）：偵測值超過 enterThreshold → 低於 exitThreshold = 1 次
- **Hold 型**（戽斗、笑）：偵測值持續超過 enterThreshold 達 3 秒 = 1 次

## 本地開發

由於瀏覽器不允許 `file://` 協定存取相機，本地開發需要 HTTP server：

```bash
cd public
python3 -m http.server 8080
# 然後開啟 http://localhost:8080
```

## 部署到 Vercel

### 方式一：GitHub 整合（推薦）

1. Push 此 repo 到 GitHub
2. 到 [vercel.com](https://vercel.com) 登入
3. Import GitHub repo
4. Framework Preset 選 **Other**
5. Deploy

之後每次 push 都會自動重新部署。

### 方式二：CLI

```bash
npx vercel --prod
```

## 專案結構

```
├── README.md
├── memory.md          ← 開發歷程與決策紀錄
├── vercel.json        ← Vercel 設定（含 camera 權限 header）
└── public/
    └── index.html     ← 完整的訓練系統（單一 HTML 檔案）
```

## 背景

此系統基於成功大學附設醫院的口咽肌肉功能訓練衛教影片（2024 最新版本）開發。原始影片包含 16 個訓練動作，經過可行性分析後，選出 5 個最適合 webcam 偵測的動作實作。

## 已知限制

- MediaPipe 標準 52 blendshape 不含 `tongueOut`，青蛙捕食使用間接指標偵測（張嘴程度）
- 河豚鼓鼓無法區分左右臉頰（`cheekPuff` 僅提供總值），使用者需自行交替
- 偵測閾值可能因個人臉部特徵、光線、相機距離而需微調
- 需要支援 WebGL 的現代瀏覽器（Chrome、Edge、Safari、Firefox）

## License

MIT
