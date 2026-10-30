# 遠端冷氣控制系統 — 系統建置與部署手冊

本手冊記錄從零建置遠端冷氣控制系統的完整技術流程，涵蓋開發環境建置、韌體編譯與燒錄、紅外線訊號擷取與協定解碼、小米藍牙加密金鑰 (`Bindkey`) 提取、硬體腳位接線以及 Home Assistant 與 Apple HomeKit 整合步驟。

---

## 第一階段：硬體與材料清單 (BOM)

| 編號 | 設備名稱 | 型號 / 規格 | 用途說明 |
| :--- | :--- | :--- | :--- |
| **1** | **ESP32 開發板** | `ESP32-DevKitC` (板載天線 `ESP32-WROOM-32D`) | 核心控制器，負責 Wi-Fi 通訊、紅外線收發控制與藍牙網關 (`BLE Proxy`) |
| **2** | **紅外線收發模組** | `M5Stack IR Unit (U002)` | 內建 5V 電晶體放大驅動之紅外線發射器 (`IR_TX`) 與 38kHz 紅外線接收器 (`IR_RX`) |
| **3** | **連接線材** | `Grove (HY2.0-4P) 轉 4Pin 2.54mm 杜邦母頭線` | 將 M5Stack U002 的 Grove 介面轉接至 ESP32-DevKitC 排針 |
| **4** | **無線溫濕度計** | `小米藍牙溫濕度計 3 (MJWSD05MMC)` | 部署於冷氣出風口下方，透過 BLE 廣播回傳出風口溫濕度數據 |
| **5** | **供電與傳輸線** | 支援資料傳輸之 `Micro-USB / Type-C 線` + `5V USB 電源供應器` | 初次連接電腦燒錄韌體使用，後續連接插座常時供電 |
| **6** | **家用伺服器** | `Synology NAS` (支援 Docker / Container Manager) | 常時運行 Home Assistant 與 HomeKit Bridge 服務 |

---

## 第二階段：macOS ESPHome 開發環境建置與初次燒錄

> **注意**：初次燒錄韌體時，請勿將杜邦線連接至 ESP32 排針，僅透過 USB 線將開發板單獨連接至電腦即可。

### 2.1 安裝 ESPHome CLI 工具
開啟 macOS 終端機（Terminal），透過 Homebrew 安裝 `esphome`（Homebrew 會自動配置獨立的 Python 執行環境與 PlatformIO 編譯工具鏈）：
```bash
brew install esphome
```
安裝完成後確認版本：
```bash
esphome version
# 輸出範例：Version: 2026.10.0
```

### 2.2 產生通訊金鑰並設定 `secrets.yaml`
1. 進入專案的 `firmware/` 目錄並建立私密設定檔：
   ```bash
   cd firmware
   cp secrets.yaml.example secrets.yaml
   ```
2. 執行以下指令，隨機產生一組 32-byte Base64 格式之 Home Assistant API 加密金鑰：
   ```bash
   openssl rand -base64 32
   ```
3. 編輯 [../firmware/secrets.yaml](../firmware/secrets.yaml)，填入 **2.4GHz Wi-Fi**（ESP32 僅支援 2.4GHz 頻段）之 SSID、密碼與上述產生的 API 金鑰：
   ```yaml
   wifi_ssid: "您的2.4G_WiFi名稱"
   wifi_password: "您的WiFi密碼"
   api_encryption_key: "由 openssl 產生的 32-byte 金鑰"
   ```

### 2.3 識別 USB 序列埠並執行初次燒錄
1. 將 `ESP32-DevKitC` 透過 USB 線連接至 Mac，於終端機列出序列埠裝置：
   ```bash
   ls -la /dev/cu.*
   ```
   確認新增之 USB 轉串口裝置路徑（常見名稱為 `/dev/cu.usbserial-0001` 或 `/dev/cu.SLAB_USBtoUART`）。
2. 驗證韌體設定檔語法：
   ```bash
   esphome config ac-remote-controller.yaml
   ```
3. 執行編譯並透過 USB 燒錄至 ESP32（首次編譯會自動下載 ESP-IDF 框架與工具鏈，約需 2～3 分鐘）：
   ```bash
   esphome compile ac-remote-controller.yaml && \
   esphome upload ac-remote-controller.yaml --device /dev/cu.usbserial-0001
   ```
4. 燒錄完成後，驗證 ESP32 是否已連上區域網路：
   ```bash
   ping -c 2 ac-remote-controller.local
   ```
   若成功回應區網 IP（如 `192.168.31.118`），後續韌體更新與日誌檢視皆可透過 Wi-Fi (OTA) 執行，無需再連接電腦 USB。

---

## 第三階段：連接 M5Stack U002 紅外線模組與訊號解碼

### 3.1 硬體接線與腳位配置
1. **中斷 ESP32 的 USB 電源供應**。
2. 將 `Grove 轉 4Pin 杜邦母頭線` 連接至 **M5Stack U002**，並對照 U002 外殼標示之 4 個腳位（`GND`、`5V`、`IN`、`OUT`）連接至 `ESP32-DevKitC`：

| M5Stack U002 標示 | 杜邦線顏色 | 連接至 ESP32-DevKitC 腳位 | 訊號方向與硬體注意事項 |
| :--- | :--- | :--- | :--- |
| **`GND`** | 黑色 (Black) | **`GND`**（建議接右排 `19` 號腳附近的 `GND`） | 系統共地線 |
| **`5V`** | 紅色 (Red) | **`5V`**（左排最左下角第一根腳位） | **注意**：`5V` 正上方相鄰腳位為 **`CMD`（內部 SPI Flash 專用腳位）**，請勿誤接，否則將導致 ESP32 無法讀取 Flash 或無法開機 |
| **`OUT` (`IR_TX`)** | 白色 (White) / 依線材對照 | **`GPIO 19`**（板子印 `19`） | **注意**：U002 外殼標示之 `OUT` 代表 **紅外線發射輸出 (`IR OUT / TX`)**，需連接至 `remote_transmitter` (`GPIO 19`) |
| **`IN` (`IR_RX`)** | 黃色 (Yellow) / 依線材對照 | **`GPIO 18`**（板子印 `18`） | U002 外殼標示之 `IN` 代表 **紅外線接收輸入 (`IR IN / RX`)**，需連接至 `remote_receiver` (`GPIO 18`) |

3. 確認接線無誤後，重新接上 USB 電源。

### 3.2 紅外線遙控器訊號擷取與解碼流程
若需擷取並解碼冷氣遙控器訊號，可依下列三個步驟進行：

1. **配置 `remote_receiver` 關鍵參數**：
   於 [../firmware/ac-remote-controller.yaml](../firmware/ac-remote-controller.yaml) 中設定：
   * `inverted: true`：M5Stack U002 內建之 38kHz 解調接收器為低電位觸發（Active-Low），平時維持高電位，接收紅外線載波時拉低電位。
   * `idle: 25ms`：ESP32 內部 RMT 15-bit 硬體計數器於 1MHz 時脈下之上限為 `32.767ms`，因此 `idle` 必須小於 `32ms`（若設定 `50ms` 將導致 `rmt_receive` 初始化失敗）。
   * `rmt_symbols: 256`：空調紅外線封包長度較長（日立協定為 344 bits，約 692 個脈衝邊緣），需提高 RMT 記憶體符號上限以避免緩衝區溢位截斷。
2. **啟動無線日誌監聽**：
   於終端機執行：
   ```bash
   esphome logs ac-remote-controller.yaml --device ac-remote-controller.local
   ```
3. **擷取並分析原廠遙控器封包**：
   將日立原廠遙控器對準 U002 接收視窗按下按鍵，終端機將輸出接收之脈衝時序與協定分析結果。
   * 實測台灣日立冷氣發送 43 Bytes（344 bits）長封包，標頭為 `0x01, 0x10, 0x00, 0x40, 0xBF...`，自第 3 Byte 起每兩個 Byte 互為位元反相校驗（`Byte[2k+1] ^ Byte[2k+2] == 0xFF`）。
   * 此協定對應 ESPHome 內建之 **`hitachi_ac344`** 平台，可直接於設定檔啟用 `platform: hitachi_ac344` 提供完整的溫控與模式切換功能。

---

## 第四階段：提取「小米藍牙溫濕度計 3」加密金鑰 (`Bindkey`)

小米藍牙溫濕度計 3 (`MJWSD05MMC`) 的 BLE 廣播採用 MiBeacon 加密。為使「米家 App」與「Home Assistant」可同時讀取資料，需由小米雲端提取配對時產生的 32 碼十六進位 `Bindkey`：

1. 開啟手機 **「米家 App」**，將「小米藍牙溫濕度計 3」新增並綁定至米家帳號。
2. 於 Mac 終端機下載並執行開源工具 **`Xiaomi Cloud Token Extractor`**：
   ```bash
   mkdir -p /tmp/token_extractor && cd /tmp/token_extractor
   curl -sL https://github.com/PiotrMachowski/Xiaomi-cloud-tokens-extractor/releases/latest/download/token_extractor.zip -o token_extractor.zip
   unzip -o token_extractor.zip
   python3 -m venv venv
   ./venv/bin/pip install -r token_extractor/requirements.txt
   ./venv/bin/python token_extractor/token_extractor.py
   ```
3. **透過 QR Code 授權登入**：
   * 當工具提示選擇登入方式 `p/q:` 時，輸入 **`q`**（QR Code 登入）。
   * 於瀏覽器開啟終端機顯示之本地網址 `http://127.0.0.1:31415`。
   * 開啟手機 **「米家 App」右上角「＋」➔「掃一掃」**，掃描網頁上的 QR Code 並確認授權。
   * 提示 `Select server:` 時直接按 `Enter`（自動檢索包含 `tw` 在內之區域伺服器）。
4. **記錄設備之 `BLE KEY`**：
   於終端機輸出清單中找到 `Xiaomi Smart Temperature and Humidity Monitor 3 (xiaomi.sensor_ht.o3)`：
   ```text
   NAME:     Xiaomi Smart Temperature and Humidity Monitor 3
   MAC:      A4:C1:38:XX:XX:XX
   BLE KEY:  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx  <-- 32 碼 Bindkey
   ```

---

## 第五階段：於 Synology NAS 部署 Home Assistant 與整合設定

詳細操作步驟請參閱 [./synology_ha_setup_guide.md](./synology_ha_setup_guide.md)，核心設定流程如下：

1. **建立配置目錄與啟動容器**：
   * 於 NAS `File Station` 建立 `/docker/homeassistant/config` 目錄。
   * 於 `Container Manager` 新增專案並匯入 [../server/docker-compose.yml](../server/docker-compose.yml)（使用 `network_mode: host` 以支援 mDNS 自動探索與 HomeKit 通訊協定）。
2. **於 Home Assistant 接入 ESP32 與溫濕度計**：
   * 開啟 `http://NAS區網IP:8123` ➔「設定 ➔ 裝置與服務」。
   * 點選自動探索之 **`AC Remote Controller (ESPHome)`**，輸入 `secrets.yaml` 中的 `api_encryption_key`。
   * 點選自動探索之 **`Temperature/Humidity Sensor (Xiaomi BLE)`**，輸入第四階段取得的 32 碼 `Bindkey`。
3. **將溫濕度感測器綁定至空調溫控實體**：
   * 於 Home Assistant 查詢溫濕度計之溫度與濕度 `Entity ID`（例如 `sensor.ke_ting_temperature_humidity_sensor_84d3_temperature`）。
   * 將其填入 [../firmware/ac-remote-controller.yaml](../firmware/ac-remote-controller.yaml) 的 `sensor:` 區塊中，並執行 OTA 更新：
     ```bash
     cd firmware
     esphome compile ac-remote-controller.yaml && \
     esphome upload ac-remote-controller.yaml --device ac-remote-controller.local
     ```
   * 完成後，Apple 家庭空調控制介面上的當前溫度（`Current`）即會即時顯示小米溫濕度計 3 所測得之真實環境溫度。

---

## 第六階段：設定 Apple HomeKit 橋接與家庭成員共享

1. **建立 `HomeKit Bridge`**：
   * 於 Home Assistant「設定 ➔ 裝置與服務 ➔ 新增整合」選擇 **`Apple ➔ HomeKit Bridge`**。
   * 勾選 **`Climate`（空調溫控器）**、**`Switch`（送風防霉 1hr 定時開關）** 與 **`Sensor`（溫濕度感測器）** 三個網域類別。
   * 開啟 Home Assistant 左下角「通知」，使用主控端 iPhone 之「家庭 App」掃描 QR Code 完成配對，並將配件指派至目標房間（如「客廳」）。
2. **多裝置家庭成員共享設定**：
   * 請勿使用第二支手機重複掃描 Home Assistant 的配對 QR Code。
   * 於主控端 iPhone「家庭 App」點選右上角 **`…` ➔「家庭設定」➔「邀請他人」**，輸入家庭成員之 Apple ID。
   * 確認新加入之配件已由「預設房間 (Default Room)」移至目標房間（如「客廳」），所有家庭成員之裝置即可自動同步顯示與操作。
