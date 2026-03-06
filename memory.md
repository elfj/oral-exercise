# Memory: 口咽肌肉功能訓練 Webcam 偵測系統

## 專案緣起

使用者（Che-Wei）擁有一組來自成功大學附設醫院的口咽肌肉功能訓練衛教影片（2024 最新版本，鏡像版本），希望開發一個 webcam 即時偵測系統，讓患者在家訓練時可以自動判斷動作是否完成、並計算次數。

## 開發歷程

### 第一階段：影片分析與可行性評估

1. 從使用者的「影片」資料夾讀取全部 16 支 mp4 衛教影片
2. 用 ffmpeg 從每支影片提取 5 張關鍵畫面（共 80 張截圖）
3. 逐一查看所有截圖，分析每個動作的視覺特徵

**影片列表：**

- 五大動作系列（4 支）：舌頭往上、往下、往前、往左 — 全部使用壓舌板
- 十三套運動系列（12 支）：01 鬚鯨濾食、02 金銅秤、03 戽斗星球、04 千斤頂、05 青蛙捕食、06 摩天輪、07 章魚吸盤、09 河豚鼓鼓、10 一笑呷百二、11 蛇吞象、12 老虎示威、13 我是演唱家（缺 08）

**分析結論：** 根據 webcam + 電腦視覺的能力限制，將 16 個動作分成三個梯隊：

- 第一梯隊（強烈推薦）：河豚鼓鼓、一笑呷百二、青蛙捕食、戽斗星球、章魚吸盤
- 第二梯隊（需搭配麥克風）：老虎示威、我是演唱家、鬚鯨濾食
- 第三梯隊（不推薦）：五大動作全部、千斤頂、摩天輪、金銅秤、蛇吞象

產出：`口咽肌肉訓練_Webcam偵測分析報告.md`

### 第二階段：Web App 開發

**技術選型討論：** 使用者詢問由 Claude 寫 vs 用 Google AI Studio。結論是由 Claude 寫，因為已經深入分析過影片特徵，且 MediaPipe 是純前端方案不需後端。

**需求確認：**
- 使用流程：兩種模式（單項練習 + 完整課程）
- 每個動作 5 次
- 全中文介面

**技術架構：**
- 單一 HTML 檔案（~860 行）
- MediaPipe Face Landmarker tasks-vision API（v0.10.14）
- 從 CDN 載入 WASM + 模型，零後端
- 使用 52 個 ARKit 相容 blendshape 進行偵測

**5 個動作的偵測對應：**

| 動作 | Blendshape | 類型 |
|------|-----------|------|
| 戽斗星球 | `jawForward` | hold（維持 3 秒） |
| 青蛙捕食 | `jawOpen` × 0.5 + `mouthLowerDownLeft/Right` × 0.3 + `mouthShrugLower` × 0.2（複合分數） | toggle |
| 章魚吸盤 | `mouthPucker` | toggle |
| 河豚鼓鼓 | `cheekPuff` | toggle |
| 一笑呷百二 | (`mouthSmileLeft` + `mouthSmileRight`) / 2 | hold（維持 3 秒） |

**偵測架構：**
- `Smoother` 類別：EMA 平滑（alpha=0.5）
- `ExerciseDetector` 類別：狀態機 + 遲滯（enter/exit 不同閾值）+ cooldown 防重複計數
- toggle 型：偵測值上升超過 enterThreshold → 下降低於 exitThreshold = 1 次
- hold 型：偵測值持續超過 enterThreshold 達 holdMs 毫秒 = 1 次

### 第三階段：問題修復與迭代

**問題 1：本地檔案無法存取相機**
- 原因：瀏覽器不允許 `file://` 協定使用 `getUserMedia`
- 解決：建立 `啟動訓練系統.command` 腳本，自動啟動 Python HTTP server + 開啟瀏覽器

**問題 2：偵測閾值太高，動作難以觸發**
- 原因：初始閾值設得太保守
- 解決：全面降低閾值（降幅 38%~60%），提高 smoothing 響應度（alpha 0.35→0.5）
- 特別是青蛙捕食：從單一 `jawOpen`（0.55）改為 3 個 blendshape 的複合分數（0.22）

**問題 3：使用者反映青蛙捕食仍然偵測不到**
- 解決：加入即時 blendshape debug 面板（右上角顯示所有嘴巴相關 blendshape 數值），用於診斷和校準
- 目前狀態：等待使用者測試回報具體數值

**增加功能：動畫示範面板**
- 使用者反映只看文字說明不知道怎麼做動作
- 為每個動作建立 SVG 動畫（SMIL 原生動畫），顯示在畫面左側
- 5 個動畫：下顎前推、張嘴伸舌、嘟嘴放鬆、臉頰交替鼓、大笑露齒

### 第四階段：部署

- 使用者希望部署到雲端（Vercel + GitHub）
- 準備了 Vercel 部署用的專案結構：`vercel.json` + `public/index.html`
- 因 Cowork 環境限制無法直接 push，提供了完整的 GitHub → Vercel 部署步驟

## 目前閾值設定

```javascript
underbite: enterThreshold: 0.10, exitThreshold: 0.05
frog:      enterThreshold: 0.22, exitThreshold: 0.10  (複合分數)
octopus:   enterThreshold: 0.25, exitThreshold: 0.10
pufferfish: enterThreshold: 0.12, exitThreshold: 0.04
smile:     enterThreshold: 0.28, exitThreshold: 0.12
```

## 檔案結構

```
影片/
├── 【衛教影片】*.mp4              ← 16 支原始衛教影片
├── 口咽肌肉訓練_Webcam偵測分析報告.md  ← 可行性分析報告
├── oral_exercise_trainer.html     ← 主程式（含 debug 面板）
├── 啟動訓練系統.command            ← macOS 本地啟動腳本
└── oral-exercise-vercel/          ← Vercel 部署用
    ├── vercel.json
    ├── memory.md
    ├── README.md
    └── public/
        └── index.html             ← 主程式副本
```

## 待辦 / 已知問題

- [ ] 青蛙捕食偵測仍需根據實測數據校準
- [ ] MediaPipe 標準 52 blendshape 不含 `tongueOut`，伸舌偵測只能用間接指標
- [ ] 河豚鼓鼓目前無法區分左右臉頰（cheekPuff 只有一個總值），只計整體鼓起次數
- [ ] debug 面板上線後應可移除或加開關
- [ ] 完整課程模式的動作間休息時間可加長
- [ ] 可考慮加入音效回饋
