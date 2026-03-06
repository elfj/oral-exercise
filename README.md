# 口咽肌肉功能訓練 — Webcam 即時偵測系統

透過 Webcam 即時偵測臉部動作，引導使用者完成口咽肌肉功能訓練，並自動計算完成次數。

## Demo

**線上版**：https://oral-exercise-vercel.vercel.app

直接在瀏覽器開啟即可使用（需允許相機權限）。

## 支援的訓練動作

| 動作 | 偵測方式 | 計次邏輯 |
|------|---------|---------|
| 🌍 戽斗星球 | 下顎前突（mouthRollLower） | 維持 3 秒 = 1 次 |
| 🐸 青蛙捕食 | 下唇下拉程度（mouthLowerDown） | 伸舌→收回 = 1 次 |
| 🐙 章魚吸盤 | 嘟嘴（mouthPucker） | 嘟嘴→放鬆 = 1 次 |
| 🐡 河豚鼓鼓 | 上唇翻捲 + 嘴巴閉合（mouthRollUpper + mouthClose） | 鼓起→放鬆 = 1 次 |
| 😁 一笑呷百二 | 左右嘴角上揚（mouthSmile） | 維持 3 秒 = 1 次 |

> **註**：部分 MediaPipe blendshape（如 `cheekPuff`、`jawForward`）對某些使用者回傳值極低，因此使用替代的複合指標偵測。

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

- **EMA 平滑**：預設 alpha=0.35，各動作可獨立設定 `smoothAlpha`
- **遲滯閾值**：enter 與 exit 使用不同閾值，防止邊界抖動
- **Cooldown**：計次後短暫冷卻期（預設 300ms，各動作可獨立設定 `cooldownMs`），防止重複計算
- **Toggle 型**（青蛙、章魚、河豚）：偵測值超過 enterThreshold → 低於 exitThreshold = 1 次
- **Hold 型**（戽斗、笑）：偵測值持續超過 enterThreshold 達 3 秒 = 1 次

### 演算法校準紀錄

偵測閾值經過實測影片逐一校準。校準方法：

1. 使用者錄製實際操作影片
2. 用 ffmpeg 提取畫面，讀取 debug 面板的 blendshape 數值
3. 比較「動作中」vs「休息」的數值差異，選擇分離度最佳的指標
4. 設定 enterThreshold / exitThreshold 確保兩狀態間有足夠間距

詳細校準歷程與數據見 [memory.md](memory.md)。

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

- MediaPipe 標準 52 blendshape 不含 `tongueOut`，青蛙捕食使用 mouthLowerDown 間接偵測
- 河豚鼓鼓無法區分左右臉頰（`cheekPuff` 僅提供總值），使用者需自行交替
- 部分 blendshape（cheekPuff、jawForward）對某些使用者無反應，需使用替代指標
- 偵測閾值可能因個人臉部特徵、光線、相機距離而需微調
- 需要支援 WebGL 的現代瀏覽器（Chrome、Edge、Safari、Firefox）

## License

MIT
