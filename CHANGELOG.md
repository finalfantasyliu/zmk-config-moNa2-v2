# moNa2 韌體修改紀錄

## 修改的檔案

- `config/mona2.keymap` — 鍵位配置、層定義、Combo、行為設定
- `boards/shields/mona2/mona2.dtsi` — 軌跡球、自動切層、Encoder 設定
- `boards/shields/mona2/mona2_r.overlay` — 軌跡球 CPI、捲動設定
- `config/mona2_r.conf` — PMW3610 感應器、藍牙設定

---

## 軌跡球

### 方向修正
- **X 軸反轉**：原本 Y_INVERT 改為 X_INVERT，修正上下左右顛倒問題
- 檔案：`mona2.dtsi` `input-processors` 設定

### 靈敏度（CPI）
- **CPI 600 → 1600**：配合 4K 螢幕，提高移動速度
- 檔案：`mona2_r.overlay` `cpi = <1600>`

### 防飄移
- **啟用初始化延遲**：`CONFIG_PMW3610_INIT_POWER_UP_EXTRA_DELAY_MS=300`，給感應器多 300ms 啟動時間
- **啟用最小回報間隔**：`CONFIG_PMW3610_REPORT_INTERVAL_MIN=12`，過濾高頻雜訊
- 檔案：`mona2_r.conf`

### 自動切層（Auto Mouse Layer）
- **啟用 `zip_temp_layer`**：軌跡球移動時自動切到 MOUSE 層（layer 5）
- **Timeout 800ms**：停止移動軌跡球後 800ms 自動退出 MOUSE 層
- **`require-prior-idle-ms = 180`**：打字後 180ms 內不觸發自動切層，防止打字時誤觸
- **`excluded-positions = <14 25>`**：G(14) 和 B(25) 為滑鼠鍵，按下時不會讓自動切層計時器重置
- 檔案：`mona2.dtsi`

### 捲動模式
- **Scroller 層改為 layer 6（SCROLL）**：原本設在 layer 3（MEDIA），改到正確的 SCROLL 層
- **捲動速度 1/8**：`zip_scroll_scaler 1 8`
- 檔案：`mona2_r.overlay`

---

## Encoder（旋轉編碼器）

### 快轉穩定性
- **steps 24 → 80**：匹配 ZMK 官方 4:1 比例，改善快轉時方向判斷錯誤
- **triggers-per-rotation 10 → 20**：每圈觸發 20 次（官方標準值）
- 檔案：`mona2.dtsi`

### 捲動方向
- **上下捲動方向對調**：配合 macOS 自然捲動，順時針=上捲、逆時針=下捲
- 檔案：`mona2.keymap` `scroll_up_down` behavior

### 捲動速度
- **SCRL_VAL 100 → 200**：Encoder 每次捲動量加倍，配合 4K 螢幕
- 檔案：`mona2.keymap` `#define ZMK_POINTING_DEFAULT_SCRL_VAL 200`

---

## 鍵位配置

### Combo（組合鍵）
| Combo | 鍵位 | 功能 |
|-------|------|------|
| W + E | 1 + 2 | Tab |
| J + K | 17 + 18 | Enter |
| S + D | 11 + 12 | 切換輸入法（Ctrl+Space） |

### 新增層

#### Layer 5 — MOUSE（自動切層目標）
- G = 左鍵（MB1）
- B = 右鍵（MB2）
- 其餘全部 `&trans`

#### Layer 6 — SCROLL
- 軌跡球變捲輪模式（透過 overlay scroller 設定）

#### Layer 7 — VIM（按住 Space 啟用）
- H = 左、J = 下、K = 上、L = 右

#### Layer 8 — DESKTOP（長按 W 啟用）
- H = 切左桌面（Ctrl+Left）
- L = 切右桌面（Ctrl+Right）

### 其他鍵位變更
- **Space**：`&lt 5 SPACE` → `&lt 7 SPACE`（改為啟用 VIM 層）
- **W**：`&kp W` → `&lt 8 W`（長按啟用 DESKTOP 層）
- **BLUETOOTH 層**：右下角 BT_CLR / BT_CLR_ALL 改為 `&trans`，防止誤按

---

## Hold-Tap 行為優化

### `&mt`（Mod-Tap）
- **新增 `require-prior-idle-ms = <150>`**：打字快時直接判定 tap，消除黏滯感
- flavor 維持 `balanced`

### `&lt`（Layer-Tap）
- **新增 `require-prior-idle-ms = <150>`**：同上
- flavor 維持 `balanced`
- `quick-tap-ms = <300>`：雙擊同一鍵後按住可長按重複（例如連按 Backspace）

---

## 藍牙連線改善

- **`CONFIG_BT_CTLR_TX_PWR_PLUS_8=y`**：提高藍牙發射功率，耗電差異極小
- **`CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y`**：改善 BLE 配對穩定性、禁用 2M PHY
- 連線間隔維持預設（MIN=6, MAX=12）
- 檔案：`mona2_r.conf`

---

## 參數參考

### 軌跡球相關
| 參數 | 值 | 說明 |
|------|---|------|
| CPI | 1600 | 4K 螢幕用 |
| 自動切層 timeout | 800ms | 停止移動後退出 MOUSE 層 |
| 自動切層 require-prior-idle | 180ms | 打字後多久才允許觸發 |
| 捲動縮放 | 1/8 | 軌跡球捲動模式的速度 |

### Encoder 相關
| 參數 | 值 | 說明 |
|------|---|------|
| steps | 80 | 官方標準，4:1 比例 |
| triggers-per-rotation | 20 | 每圈觸發次數 |
| SCRL_VAL | 200 | 每次捲動量 |

### Hold-Tap 相關
| 參數 | 值 | 說明 |
|------|---|------|
| flavor | balanced | 維持平衡模式 |
| tapping-term-ms | 200（預設） | 判定 tap/hold 的等待時間 |
| quick-tap-ms（lt） | 300 | 雙擊後按住可長按 |
| require-prior-idle-ms | 150 | 快速打字時直接判定 tap |

### 藍牙相關
| 參數 | 值 | 說明 |
|------|---|------|
| TX Power | +8 dBm | 加強訊號 |
| Experimental Conn | 啟用 | 改善穩定性 |
| MIN_INT / MAX_INT | 6 / 12 | 連線間隔（預設值） |
