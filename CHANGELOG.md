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
- **啟用 `zip_temp_layer`**：軌跡球移動時自動切到 MOUSE 層（layer 5），[ZMK Temp Layer 文件](https://zmk.dev/docs/keymaps/input-processors/overview)
- **基礎 Timeout 800ms → 1500ms**：停止移動軌跡球後 1.5 秒自動退出 MOUSE 層（原 800ms 太短，滑鼠操作來不及完成）
- **`mkp_input_listener` 點擊延長至 5 秒**：按滑鼠鍵時超時延長，確保雙擊和連續操作不會被中斷
- **`require-prior-idle-ms = 180`**：打字後 180ms 內不觸發自動切層，防止打字時手掌誤觸軌跡球
- **`excluded-positions = <14 25>`**：G(14)=MB1 和 B(25)=MB2，按滑鼠鍵時不退出 MOUSE 層
- 檔案：`mona2.dtsi`（基礎設定）、`mona2.keymap`（mkp_input_listener）

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
| Combo | 鍵位 | 功能 | 特殊設定 |
|-------|------|------|---------|
| W + E | 1 + 2 | Tab | `slow-release` |
| J + K | 17 + 18 | Enter | `slow-release` |
| S + D | 11 + 12 | 切換輸入法 | `slow-release` + `&input_tap` macro |

**`slow-release`**：兩個鍵都放開後才結束 combo，防止放開時機不同導致重複觸發。

**`&input_tap` macro**：輸入法切換原本用 `&kp LC(SPACE)`，按住期間會持續送出 Ctrl+Space，macOS 會不斷循環切換輸入法。改用 macro 後不管按多久只送一次快速點按。

### SYM 層新增 C++ 符號
| 位置 | 符號 | 用途 |
|------|------|------|
| C 位（`*` 下方） | `::` | scope resolution（如 `std::vector`） |
| V 位（`\` 下方） | `->` | arrow operator（如 `ptr->member`） |

用 macro 實現，按一下輸出兩個字元。

### 新增層

#### Layer 5 — MOUSE（自動切層目標）
- G = 左鍵（MB1）
- B = 右鍵（MB2）
- 其餘全部 `&trans`

#### Layer 6 — SCROLL
- 軌跡球變捲輪模式（透過 overlay scroller 設定）

#### Layer 7 — VIM（按住 Space 啟用）
- 右手 home row：H=左、J=下、K=上、L=右
- 右手頂排：Y=Home、U=PgDn、I=PgUp、O=End

#### Layer 8 — DESKTOP（長按 W 啟用）
- H = 切左桌面（Ctrl+Left）
- J = App Exposé（Ctrl+Down）
- K = Mission Control（Ctrl+Up）
- L = 切右桌面（Ctrl+Right）

### BLUETOOTH 層改動
- **移除右手 BT_SEL 0-4**（左手已有，不需要兩邊重複）
- **右手加方向鍵**：K=UP、M=LEFT、,=DOWN、.=RIGHT（方便在 BT 層做基本導航）
- **B 位**：`&ind_bat` 手動查看電量

### 其他鍵位變更
- **Space**：`&lt 5 SPACE` → `&lt 7 SPACE`（改為啟用 VIM 層）
- **W**：`&kp W` → `&lt 8 W`（長按啟用 DESKTOP 層）
- **右下角**：`&lt 4 SPACE` → `&lt 4 DELETE`（Space 用處不大，改為 Delete）

---

## Hold-Tap 行為優化

### `&mt`（Mod-Tap）
- **`require-prior-idle-ms = <150>`**：打字快時直接判定 tap，防止打字中誤觸 Shift
- flavor = `balanced`

> **什麼是 require-prior-idle-ms？** 如果你在過去 150ms 內按過任何鍵（正在打字），hold-tap 會直接判定為 tap，不會觸發 hold 功能。這防止打字時意外按住 Z 觸發 Shift。[ZMK 文件](https://zmk.dev/docs/keymaps/behaviors/hold-tap#require-prior-idle-ms)

### `&lt`（Layer-Tap）
- **移除 `require-prior-idle-ms`**（原本 150，現在不設）：讓切層隨時可用，不受打字速度影響
- flavor = `balanced`
- `quick-tap-ms = <200>`：200ms 內連按同一鍵直接判定 tap（例如快速連按 Backspace 刪字）
- `tapping-term-ms = <200>`：按住超過 200ms 判定為 hold（切層）

> **為什麼 `&mt` 保留 idle 門檻但 `&lt` 移除？**
> - `&mt` 管的是修飾鍵（Shift on Z），打字時常碰到，需要防誤觸
> - `&lt` 管的是層切換（Backspace→SYM 層），是你主動要用的，加門檻反而讓切層不順
>
> [ZMK Hold-Tap 文件](https://zmk.dev/docs/keymaps/behaviors/hold-tap)

### `mkp_input_listener`（滑鼠鍵點擊延長超時）

```c
&mkp_input_listener { input-processors = <&zip_temp_layer MOUSE 5000>; };
```

按下 G(MB1) 或 B(MB2) 時，MOUSE 層超時延長到 5 秒。

> **為什麼需要？** 沒有這個設定時，MOUSE 層超時只有 1.5 秒（靠軌跡球移動重置）。雙擊時第一次點擊後如果軌跡球沒在動，1.5 秒到了層就退出，第二次點擊會輸出字母 g 而不是 MB1。加了這個後，每次點擊都會重置超時到 5 秒，確保連續操作不會中斷。
>
> [ZMK Input Processor 文件](https://zmk.dev/docs/keymaps/input-processors/overview)

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
| CPI | 1600 | 4K 螢幕用，[ZMK PMW3610](https://github.com/inorichi/zmk-pmw3610-driver) |
| 自動切層 timeout | **1500ms** | 停止移動後退出 MOUSE 層（原 800ms 太短） |
| mkp_input_listener timeout | **5000ms** | 點擊滑鼠鍵後延長超時，防止雙擊中斷 |
| 自動切層 require-prior-idle | 180ms | 打字後多久才允許觸發 |
| excluded-positions | 14, 25 | G 和 B（滑鼠鍵），按下不退出 MOUSE 層 |
| 捲動縮放 | 1/8 | 軌跡球捲動模式的速度 |

### Encoder 相關
| 參數 | 值 | 說明 |
|------|---|------|
| steps | 80 | 官方標準，4:1 比例 |
| triggers-per-rotation | 20 | 每圈觸發次數 |
| SCRL_VAL | 200 | 每次捲動量，[ZMK Pointing](https://zmk.dev/docs/keymaps/behaviors/pointing) |

### Hold-Tap 相關
| 參數 | 行為 | 值 | 說明 |
|------|------|---|------|
| flavor | mt / lt | balanced | 按住+按其他鍵=hold，單獨放開=tap，[文件](https://zmk.dev/docs/keymaps/behaviors/hold-tap#flavors) |
| tapping-term-ms | lt | 200 | 按住超過 200ms 判定 hold，[文件](https://zmk.dev/docs/keymaps/behaviors/hold-tap#tapping-term-ms) |
| quick-tap-ms | lt | 200 | 200ms 內連按同鍵=tap（連打 Backspace），[文件](https://zmk.dev/docs/keymaps/behaviors/hold-tap#quick-tap-ms) |
| require-prior-idle-ms | **僅 mt** | 150 | 打字中直接判定 tap（lt 不設，讓切層隨時響應），[文件](https://zmk.dev/docs/keymaps/behaviors/hold-tap#require-prior-idle-ms) |

### 藍牙相關
| 參數 | 值 | 說明 |
|------|---|------|
| TX Power | +8 dBm | 加強訊號，[ZMK BLE 文件](https://zmk.dev/docs/troubleshooting/connection-issues) |
| Experimental Conn | 啟用 | 改善穩定性 |
| MIN_INT / MAX_INT | 6 / 12 | 連線間隔（預設值） |

---

## 電量 LED 指示

### 按鍵查看電量
- **BLUETOOTH 層按 B（`&ind_bat`）**：LED 閃爍顯示當前電量
- 需要 `#include <behaviors/rgbled_widget.dtsi>` 在 keymap 中
- 分體鍵盤左右各自顯示自己的電量

### 電量顏色等級（4 級）
| 電量 | 顏色 | COLOR 代碼 |
|------|------|-----------|
| > 70% | 綠色 | 2 (GREEN) |
| 30-70% | 黃色 | 3 (YELLOW) |
| 10-30% | 洋紅 | 5 (MAGENTA) |
| < 10% | 紅色 | 1 (RED) |

### 自動顯示行為
| 觸發時機 | 行為 |
|---------|------|
| 開機 / 喚醒 | 閃一下對應顏色 |
| 電量低於 10%（critical） | 每次電量變化時自動閃紅色 |
| Peripheral 斷線 | 閃洋紅色 |

### 可用顏色代碼參考（zmk-rgbled-widget）
| 代碼 | 顏色 |
|------|------|
| 0 | 黑（不亮） |
| 1 | 紅 |
| 2 | 綠 |
| 3 | 黃 |
| 4 | 藍 |
| 5 | 洋紅 |
| 6 | 青 |
| 7 | 白 |

---

## 注意事項

### .conf 檔案格式
- Kconfig 的 `.conf` 檔**不支援行內註解**
- `CONFIG_XXX=70  # 註解` 會導致 build 失敗
- 註解必須放在**獨立一行**，以 `#` 開頭
