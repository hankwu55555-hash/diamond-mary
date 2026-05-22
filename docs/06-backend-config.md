# 06 · 後臺設定

## 技術架構說明

- 遊玩次數限制與冷卻倒數由 **server-side** 控制（每日次數依 VIP × 國家矩陣）
- 每次 spin 結果由 server 計算，client 僅負責演出
- 資料記錄使用 MSSQL 與 MongoDB 雙寫架構

## 分析需求（⚠️ 建議版本，待數據分析師確認）

| Why（業務問題） | What（指標） | How（計算方式） |
|----------------|-------------|----------------|
| 玩家對小瑪莉的參與度如何？ | 日活躍遊玩人數（DAU） | COUNT(DISTINCT UserID) FROM SessionGameCenterGameLog WHERE DATE(StartEventTime) = 目標日 |
| 玩家平均每日遊玩幾手？ | 人均每日旋轉次數 | SUM(TotalBetTimes) / COUNT(DISTINCT UserID)，按日分組 |
| 金幣消耗速度是否健康？ | 日均金幣投入量 | SUM(TotalCoinBet) GROUP BY DATE(StartEventTime) |
| 寶石產出是否在預期範圍內？ | 日均寶石產出量 | SUM(TotalGemWin) GROUP BY DATE(StartEventTime) |
| 不同 VIP 等級的遊玩行為差異？ | 各 VIP 等級人均旋轉次數與寶石產出 | AVG(TotalBetTimes), AVG(TotalGemWin) GROUP BY VipLV |
| 各押注額的使用比例？ | 押注額分佈比 | 依 TotalCoinBet / TotalBetTimes 推算單手押注額，統計 50M/100M/500M 各佔比 |
| 自動旋轉功能使用率？ | 自動旋轉使用比例 | 具備連續多手（≥10 手間隔 < 3 秒）特徵的 session 佔比 |
| 每日次數上限是否合理？ | 次數耗盡率 | 當日使用次數 = 上限的 UserID 數 / 當日遊玩 UserID 數 |

> **備註**：以上分析需求為根據資料表結構推導的建議版本，實際業務問題與指標定義請由數據分析師確認調整。

## 埋點需求總覽

| 需求時間 | 異動 Table | 需求類型 | 內容說明 | 負責軟體 | 數據驗證 |
|----------|-----------|----------|----------|----------|----------|
| 2026.05.19 | EventLog | 欄位值擴充 | EventID 欄位新增分類值（參考 DimEventID） | 承諭 | 育瑄 |
| 2026.05.19 | SessionGameCenterGameLog | 新增欄位 | 新增 17 個欄位追蹤 session 遊玩數據 | 又豪 | 育瑄 |

## 資料表結構

### SessionGameCenterGameLog（Session 遊玩記錄）

| 中文說明 | 欄位名稱 | MSSQL 型別 | MongoDB 型別 | 備註 |
|----------|----------|-----------|-------------|------|
| SessionID | SessionID | varchar(32) | BsonString | |
| 該 session 第一次 spin 時間 | StartEventTime | int | BsonInt32 | Unix timestamp（秒） |
| 該 session 最後一次 spin 時間 | LastEventTime | int | BsonInt32 | Unix timestamp（秒） |
| 娛樂館遊戲 ID | GameCenterGameID | smallint | BsonInt32 | Refer to definition table |
| 玩家 ID | UserID | int | BsonInt32 | |
| 總押注次數 | TotalBetTimes | int | BsonInt32 | |
| 總押注額金幣 | TotalCoinBet | decimal(18,2) | BsonInt64 | |
| 總押注額寶石 | TotalGemBet | int | BsonInt32 | |
| 總 Win 次數 | TotalWinTimes | int | BsonInt32 | |
| 總金幣贏分 | TotalCoinWin | decimal(18,2) | BsonInt64 | |
| 總寶石贏分 | TotalGemWin | int | BsonInt32 | |
| session 結束時玩家金幣餘額 | EndBalance | decimal(18,2) | BsonInt64 | |
| session 結束時玩家寶石餘額 | EndGem | int | BsonInt32 | |
| 設備識別 ID | UDID | nvarchar(36) | BsonString | 取得資料若為空白或不符格式，預設值 'Unknown' |
| VIP Level | VipLV | int | BsonInt32 | |
| 所在地區（國別） | Country | char(2) | BsonString | ISO 3166-1 alpha-2（如 TW、US、CA） |
| 機率代碼 | ChanceCode | int | BsonInt32 | Refer to definition table |

### SessionGemLog（寶石異動記錄，IGS only）

| 中文說明 | 欄位名稱 | MSSQL 型別 | MongoDB 型別 | 備註 |
|----------|----------|-----------|-------------|------|
| 該 session 第一次事件發生時間 | StartEventTime | int | BsonInt32 | Unix timestamp（秒） |
| 該 session 最後一次事件發生時間 | LastEventTime | int | BsonInt32 | Unix timestamp（秒） |
| 事件代號 | GemReason | smallint | BsonInt32 | DimGemReason |
| 玩家代號 | UserID | int | BsonInt32 | |
| 總 Event 次數 | TotalEventTimes | int | BsonInt32 | |
| 總獲得寶石量 | TotalGemAwarded | int | BsonInt64 | |
| session 結束時玩家寶石剩餘數量 | EndBalance | int | BsonInt64 | |
| 設備識別 ID | UDID | nvarchar(36) | BsonString | 取得資料若為空白或不符格式，預設值 'Unknown' |
| SessionID | SessionID | varchar(50) | BsonString | |

### GemToCoinGiftAwardLog（寶石贈金幣收取 Log）

| 中文說明 | 欄位名稱 | MSSQL 型別 | MongoDB 型別 | 備註 |
|----------|----------|-----------|-------------|------|
| 事件發生時間 | EventTime | int | BsonInt32 | Unix timestamp（秒） |
| 禮物 ID | GiftID | varchar(50) | BsonString | |
| 收禮者 ID | ReceiverID | int | BsonInt32 | |
| 收禮者事件前金幣 | ReceiverCoinBefore | decimal(18,2) | BsonInt32 | |
| 收禮者事件後金幣 | ReceiverCoinAfter | decimal(18,2) | BsonInt32 | |
| 獲取金幣 | ReceiverCoinAward | decimal(18,2) | BsonInt32 | 可能為 0，代表收的是寶石 |
| 收禮者事件前寶石 | ReceiverGemBefore | decimal(18,2) | BsonInt32 | |
| 收禮者事件後寶石 | ReceiverGemAfter | decimal(18,2) | BsonInt32 | |
| 獲取寶石 | ReceiverGemAward | decimal(18,2) | BsonInt32 | 可能為 0，代表收的是金幣 |
| 收禮者 Vip 等級 | ReceiverVip | int | BsonInt32 | |
| 贈禮者 ID | SenderID | int | BsonInt32 | |
| 送出的物品類型 | SentType | int | BsonInt32 | 1: 金幣，2: 寶石 |

## 需求類型定義

| 需求類型（中文） | 需求類型（英文） | 說明 |
|-----------------|-----------------|------|
| 新增資料表 | Create Table | 建立一張全新的資料表，用於承載新的資料結構與內容 |
| 新增欄位 | Add Column | 在既有資料表中新增一個欄位，擴充資料結構 |
| 欄位值擴充 | Field Value Expansion | 資料表結構不變，僅新增某欄位允許使用的分類值 |
| 調整欄位定義 | Alter Column | 修改既有欄位的結構定義（資料型態、長度、NULL、預設值等） |
| 對應驗證 | Mapping Validation | 驗證新資料是否符合既有資料規則與分類邏輯，不涉及結構異動 |

## 可調參數總覽表

| 參數名稱 | 當前值 | 說明 | 狀態 |
|----------|--------|------|------|
| `daily_spin_limit` | VIP1=0, VIP2=0, VIP3=20, VIP4=50, VIP5=100, VIP6=150 | 每日可遊玩次數（依 VIP 等級，1=銅 6=鑽石） | ✅ 已確認 |
| `bet_options` | [50000000, 100000000, 500000000] | 可選押注額列表（單位：金幣） | ✅ 已確認（影片實測） |
| `cooldown_duration` | 等待至伺服器午夜 00:00 重置 | 次數歸零後冷卻時間（按鈕顯示 RESET 倒數至午夜） | ⚠️ 建議值（依 RESET 文案推論） |
| `auto_spin_options` | [10, 50, 100, ∞] | 自動旋轉手數選項 | ✅ 已確認（影片實測） |
| `long_press_threshold` | 約 2 秒 | 長按觸發自動旋轉閾值 | ✅ 已確認 |
| `base_reward_table` | 見 01-基礎規格 | 50M 押注時的格子獎勵基準值（1K～10K） | ✅ 已確認（影片實測） |
| `reward_scale_factor` | 與押注額等比 | 獎勵隨押注額線性縮放（×2、×10） | ✅ 已確認（影片實測） |
| `chance_code` | 見下方建議結構 | 機率代碼（控制各格子中獎機率） | ⚠️ 建議結構（實際權重由數值企劃設定） |
| `game_center_game_id` | 待系統分配 | 娛樂館遊戲 ID（新遊戲需由後端團隊在 DimGameCenterGame 表新增） | ⚠️ 待後端分配 |

## chance_code 建議結構（⚠️ 建議版本）

`chance_code` 控制轉盤 22 格的中獎機率分佈。建議採用加權隨機模型：

```json
{
  "chance_code": 1,
  "description": "標準機率配置",
  "weights": {
    "pos_1":  2,   // 猴爺 10K — 低權重（稀有大獎）
    "pos_2":  10,  // 1K
    "pos_3":  8,   // 1.5K
    "pos_4":  5,   // 3K
    "pos_5":  7,   // 2K
    "pos_6":  10,  // 1K
    "pos_7":  8,   // 1.5K
    "pos_8":  3,   // 6K — 低權重
    "pos_9":  7,   // 2K
    "pos_10": 8,   // 1.5K
    "pos_11": 2,   // 聚寶盆 8K — 低權重（次大獎）
    "pos_12": 10,  // 1K
    "pos_13": 8,   // 1.5K
    "pos_14": 5,   // 3K
    "pos_15": 8,   // 1.5K
    "pos_16": 10,  // 1K
    "pos_17": 8,   // 1.5K
    "pos_18": 10,  // 1K
    "pos_19": 7,   // 2K
    "pos_20": 4,   // 4K
    "pos_21": 7,   // 2K
    "pos_22": 8    // 1.5K
  }
}
```

> **計算原則**：高獎勵格（猴爺、聚寶盆、6K）設定較低權重，低獎勵格（1K、1.5K）設定較高權重。權重總和 = 155，各格中獎機率 = weight / 155。實際權重值由數值企劃根據目標 RTP（Return to Player）調整。

> **多組 chance_code**：可配置多組不同的機率代碼，透過後臺切換或依玩家條件（VIP、國家等）動態選擇。

## 待確認事項彙整

### 數值企劃

- [x] 可選押注額選項列表 — ✅ 已確認（50M / 100M / 500M）
- [x] 轉盤各格子的寶石獎勵對照表 — ✅ 已確認（影片逐格分析）
- [x] 自動旋轉手數選項（最多 4 個）— ✅ 已確認（10 / 50 / 100 / ∞）
- [x] 冷卻倒數時間 — ⚠️ 已提供建議值（等待至伺服器午夜重置）
- [x] 機率代碼定義 — ⚠️ 已提供建議結構（加權隨機模型）
- [x] 各 VIP 等級每日可遊玩次數 — ✅ 已確認（VIP1=0, VIP2=0, VIP3=20, VIP4=50, VIP5=100, VIP6=150）

### UI / 美術

- [ ] 寶石小瑪莉本體畫面 — 🔴 待製作（已提供 12 張參考圖）
- [ ] 各格壓黑圖 — 🔴 待製作
- [ ] 寶石飛入動畫效果 — 🔴 待製作（動畫規格已定義於 05-animations）
- [ ] 自動旋轉次數選擇面板 — 🔴 待製作（面板規格已定義於 04-ui-display）
- [x] 金幣不足時的提示 UI — ⚠️ 已提供建議規格（Toast 樣式 + 7 語系文案）

### 後端 / 數據

- [ ] SessionGameCenterGameLog 17 個欄位建置 — 🔴 待開發
- [ ] SessionGemLog 9 個欄位建置 — 🔴 待開發
- [ ] GemToCoinGiftAwardLog 12 個欄位建置 — 🔴 待開發
- [ ] EventLog EventID 分類值擴充 — 🔴 待開發
- [ ] GameCenterGameID 定義 — ⚠️ 待後端團隊分配 ID
- [x] 分析需求（Why/What/How）— ⚠️ 已提供建議版本（8 項指標）

## 確認進度追蹤表

| 部門 | 總項目 | 已完成/已建議 | 待處理 | 狀態 |
|------|--------|--------------|--------|------|
| 數值企劃 | 6 | 5（3 已確認 + 2 建議值） | 1 | 🟡 大部分已處理 |
| UI / 美術 | 5 | 1（建議值） | 4 | 🔴 待製作 |
| 後端 / 數據 | 6 | 1（建議值） | 5 | 🔴 待開發 |
