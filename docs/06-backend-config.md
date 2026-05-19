# 06 · 後臺設定

## 技術架構說明

- 遊玩次數限制與冷卻倒數由 **server-side** 控制（每日次數依 VIP × 國家矩陣）
- 每次 spin 結果由 server 計算，client 僅負責演出
- 資料記錄使用 MSSQL 與 MongoDB 雙寫架構

## 分析需求

| Why（業務問題） | What（指標） | How（計算方式） |
|----------------|-------------|----------------|
| **TBD（由數據分析師定義）** | **TBD** | **TBD** |

> **備註**：分析需求尚未填寫，待數據分析師補充業務問題、指標與計算方式。

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
| `daily_spin_limit` | **TBD** | 每日可遊玩次數（VIP × 國家矩陣） | 🔴 由數值企劃設定 |
| `bet_options` | **TBD** | 可選押注額列表 | 🔴 由數值企劃設定 |
| `cooldown_duration` | **TBD** | 次數歸零後冷卻時間 | 🔴 由數值企劃設定 |
| `auto_spin_options` | **TBD** | 自動旋轉手數選項（最多 4 個） | 🔴 由數值企劃設定 |
| `long_press_threshold` | 約 2 秒 | 長按觸發自動旋轉閾值 | ✅ 已確認 |
| `reel_reward_multipliers` | **TBD** | 轉盤各格子對應押注額的寶石獎勵倍率 | 🔴 由數值企劃設定 |
| `chance_code` | **TBD** | 機率代碼（控制中獎機率） | 🔴 由數值企劃設定 |
| `game_center_game_id` | **TBD** | 娛樂館遊戲 ID | 🔴 待確認 |

## 待確認事項彙整

### 數值企劃

- [ ] 各 VIP 等級 × 國家的每日可遊玩次數矩陣
- [ ] 可選押注額選項列表
- [ ] 轉盤各格子的寶石獎勵對照表
- [ ] 冷卻倒數時間
- [ ] 自動旋轉手數選項（最多 4 個）
- [ ] 機率代碼定義

### UI / 美術

- [ ] 寶石小瑪莉本體畫面
- [ ] 各格壓黑圖
- [ ] 寶石飛入動畫效果
- [ ] 自動旋轉次數選擇面板
- [ ] 金幣不足時的提示 UI

### 後端 / 數據

- [ ] SessionGameCenterGameLog 17 個欄位建置
- [ ] SessionGemLog 9 個欄位建置
- [ ] GemToCoinGiftAward