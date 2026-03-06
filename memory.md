# Memory: 口咽肌肉功能訓練 Webcam 偵測系統

## 專案緣起

使用者（Che-Wei）擁有一組來自成功大學附設醫院的口咽肌肉功能訓練衛教影片（2024 最新版本，鏡像版本），希望開發一個 webcam 即時偵測系統，讓患者在家訓練時可以自動判斷動作是否完成、並計算次數。

## 部署資訊

- **GitHub**：https://github.com/elfj/oral-exercise
- **Vercel**：https://oral-exercise-vercel.vercel.app
- **GitHub 帳號**：elfj

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

**偵測架構：**
- `Smoother` 類別：EMA 平滑（預設 alpha=0.35，可由各動作覆蓋）
- `ExerciseDetector` 類別：狀態機 + 遲滯（enter/exit 不同閾值）+ cooldown 防重複計數
- 支援每個動作獨立設定 `smoothAlpha` 和 `cooldownMs`
- toggle 型：偵測值上升超過 enterThreshold → 下降低於 exitThreshold = 1 次
- hold 型：偵測值持續超過 enterThreshold 達 holdMs 毫秒 = 1 次

**增加功能：動畫示範面板**
- 使用者反映只看文字說明不知道怎麼做動作
- 為每個動作建立 SVG 動畫（SMIL 原生動畫），顯示在畫面左側

### 第三階段：部署

- GitHub repo: https://github.com/elfj/oral-exercise (master branch)
- Vercel 靜態網站: https://oral-exercise-vercel.vercel.app
- `vercel.json` 設定 `Permissions-Policy: camera=self` header

### 第四階段：演算法校準（基於實測影片）

使用者錄製實際操作影片，用 ffmpeg 提取畫面分析 debug 面板的 blendshape 數值，逐一校準每個動作的偵測演算法。

**關鍵發現：多個 MediaPipe blendshape 對此使用者回傳 0，需改用替代指標。**

#### 河豚鼓鼓（pufferfish）— 3 輪修正

**問題**：`cheekPuff` blendshape 永遠回傳 0.000。

- **第 1 輪**：改用 mSmileLeft/Right 複合分數 → 失敗（mSmile 也是 0）
- **第 2 輪**：改用 `mRollUpper*0.6 + mClose*0.4`，exitThreshold 0.18 → 失敗（運動間 mClose 維持 0.290，分數 0.221 > 0.18，toggle 無法完成）
- **第 3 輪（最終）**：降低 mClose 權重 → `mRollUpper*0.8 + mClose*0.2`，exitThreshold 提高至 0.22，新增 smoothAlpha 0.6 + cooldownMs 600
  - 運動間分數 0.198 < 0.22 exit ✓

#### 戽斗星球（underbite）

**問題**：`jawForward` blendshape 永遠回傳 0.000（最高 0.010）。

- **分析**：mouthRollLower 從靜止 ~0.1 跳到下顎前突時 ~0.97
- **修正**：改用 `mouthRollLower`（保留 jawForward > 0.08 的 fallback），enterThreshold 0.35，exitThreshold 0.20

#### 青蛙捕食（frog）

**問題**：舊公式 `jawOpen*0.5 + mLowerDown*0.3 + mShrugLower*0.2` 中，jawOpen 在休息時仍高達 0.41-0.63，導致分數永遠 > exitThreshold 0.10，toggle 無法完成。

- **分析**：mouthLowerDownLeft/Right 是真正的差異指標
  - 舌頭伸出：0.37-0.68
  - 休息：0.005-0.04
- **修正**：改用純 `(mouthLowerDownLeft + mouthLowerDownRight) / 2`，enterThreshold 0.15，exitThreshold 0.06，smoothAlpha 0.5，cooldownMs 400

## 目前偵測設定（最新）

```javascript
// 戽斗星球 — hold 3 秒
underbite: {
  getScore: jawForward > 0.08 ? jawForward : mouthRollLower,
  enterThreshold: 0.35, exitThreshold: 0.20
}

// 青蛙捕食 — toggle
frog: {
  getScore: (mouthLowerDownLeft + mouthLowerDownRight) / 2,
  enterThreshold: 0.15, exitThreshold: 0.06,
  smoothAlpha: 0.5, cooldownMs: 400
}

// 章魚吸盤 — toggle
octopus: {
  blendshape: 'mouthPucker',
  enterThreshold: 0.25, exitThreshold: 0.10
}

// 河豚鼓鼓 — toggle
pufferfish: {
  getScore: cheekPuff > 0.08 ? cheekPuff : mouthRollUpper * 0.8 + mouthClose * 0.2,
  enterThreshold: 0.28, exitThreshold: 0.22,
  smoothAlpha: 0.6, cooldownMs: 600
}

// 一笑呷百二 — hold 3 秒
smile: {
  getScore: (mouthSmileLeft + mouthSmileRight) / 2,
  enterThreshold: 0.28, exitThreshold: 0.12
}
```

## 校準方法論

1. 使用者錄製實際操作影片（MOV）
2. 用 ffmpeg 提取畫面（`fps=2`），裁切右半部（debug 面板）
3. 讀取每幀 blendshape 數值，區分「動作中」vs「休息」
4. 找出分離度最佳的 blendshape 指標
5. 設定 enterThreshold / exitThreshold 確保兩狀態間有足夠間距
6. 部署後使用者實測驗證

**重要教訓：**
- 靜態擺拍的基準值 ≠ 運動中的基準值（運動中肌肉張力較高）
- 必須分析實際運動影片，不能只看靜態截圖
- MediaPipe 某些 blendshape 對特定使用者可能完全無反應（回傳 0），需要替代指標

## 檔案結構

```
影片/
├── 【衛教影片】*.mp4              ← 16 支原始衛教影片
├── 口咽肌肉訓練_Webcam偵測分析報告.md  ← 可行性分析報告
├── oral_exercise_trainer.html     ← 主程式（本地版，與 Vercel 版同步）
├── 啟動訓練系統.command            ← macOS 本地啟動腳本
└── oral-exercise-vercel/          ← Vercel 部署用（GitHub repo）
    ├── vercel.json
    ├── memory.md                  ← 本檔案
    ├── README.md
    └── public/
        └── index.html             ← 主程式
```

## 待辦 / 已知問題

- [ ] 章魚吸盤、一笑呷百二尚未經過實測影片校準
- [ ] MediaPipe 標準 52 blendshape 不含 `tongueOut`，青蛙捕食使用 mouthLowerDown 間接偵測
- [ ] 河豚鼓鼓無法區分左右臉頰（cheekPuff 只有一個總值），只計整體鼓起次數
- [ ] debug 面板目前常駐顯示，可考慮加開關
- [ ] 完整課程模式的動作間休息時間可加長
- [ ] 可考慮加入音效回饋
