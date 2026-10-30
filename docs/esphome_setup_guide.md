# macOS ESPHome 開發環境建置與燒錄指南

本文件說明如何在 macOS 環境下安裝與使用 **ESPHome**，將韌體設定檔（`.yaml`）編譯並透過 USB 或 Wi-Fi (OTA) 燒錄至 **ESP32-DevKitC (`ESP32-WROOM-32D`)** 開發板。

---

## 1. 安裝方式說明

在 macOS 上安裝 ESPHome 主要有以下兩種方式：
1. **透過 `Homebrew` 安裝（本專案採用）**：
   * 只需執行 `brew install esphome`。Homebrew 會自動配置獨立的 Python 執行環境與底層編譯工具鏈（`PlatformIO`、`esptool`），避免與系統既有之 Python 環境發生套件版本衝突，且可直接於全系統呼叫 `esphome` 指令。
2. **透過 `Python venv (pip)` 安裝**：
   * 需手動建立虛擬環境並管理 Python 版本相容性。

---

## 2. ESPHome 安裝步驟

### 步驟 2.1：確認 Homebrew 環境
於終端機（Terminal）確認系統已安裝 Homebrew：
```bash
brew --version
```

### 步驟 2.2：安裝 ESPHome CLI 工具
執行以下指令安裝 ESPHome：
```bash
brew install esphome
```
*(安裝過程中，Homebrew 會自動下載 ESPHome 及其依賴之 `platformio` 與 `esptool` 編譯燒錄工具。)*

### 步驟 2.3：驗證安裝結果
安裝完成後，執行以下指令確認版本號：
```bash
esphome version
```
若終端機正常輸出 `Version: 202x.x.x` 即代表安裝完成。

---

## 3. 硬體連接：將 ESP32-DevKitC 連接至 Mac

1. **準備具備資料傳輸功能之 USB 線材**：
   * **注意事項**：請確認使用支援資料傳輸的 USB 線材。部分僅供充電之線材內部無資料線芯，連接電腦後僅電源指示燈亮起，作業系統將無法識別序列埠（Serial Port）。
2. **檢查 macOS 是否識別到 ESP32 序列埠**：
   連接 USB 線後，於終端機輸入以下指令列出 USB 序列埠裝置：
   ```bash
   ls /dev/cu.*
   ```
   * 若出現 `/dev/cu.usbserial-xxxx`、`/dev/cu.SLAB_USBtoUART` 或 `/dev/cu.wchusbserialxxxx` 等裝置節點，即代表系統已正確識別 ESP32 開發板。
   * *(註：ESP32-DevKitC 板載之 USB 轉串口晶片通常為 `CP2102` 或 `CH340`，新版 macOS 已內建相容驅動程式。)*

---

## 4. ESPHome 常用核心指令

本專案之韌體設定檔位於 `../firmware/` 目錄下（詳細規格請參閱 [system_spec.md](./system_spec.md)）：
* `../firmware/secrets.yaml`：存放 Wi-Fi 憑證與 API 通訊加密金鑰（已列入 `.gitignore` 排除版本控制）。
* `../firmware/ac-remote-controller.yaml`：ESP32 主設定檔（定義 Wi-Fi、藍牙網關、紅外線收發腳位與空調溫控實體）。

進入 `firmware/` 目錄後，常用維護指令如下：

| 操作目的 | 終端機指令 | 說明 |
| :--- | :--- | :--- |
| **1. 驗證語法並編譯韌體** | `esphome compile ac-remote-controller.yaml` | 檢查 YAML 語法是否正確，並將設定檔編譯為二進位韌體映像檔。 |
| **2. 編譯、上傳韌體並開啟日誌** | `esphome run ac-remote-controller.yaml` | 依序執行「編譯 ➔ 透過 USB 或 Wi-Fi (OTA) 上傳至開發板 ➔ 顯示即時運作日誌」。 |
| **3. 檢視即時運作日誌 (Logs)** | `esphome logs ac-remote-controller.yaml` | 透過 USB 或 Wi-Fi 即時檢視開發板狀態輸出與紅外線接收解碼結果。 |

---

## 5. 初次燒錄與後續無線更新流程

1. 建立 `../firmware/secrets.yaml`（填入 2.4GHz Wi-Fi 帳號密碼與 API 金鑰）並確認 `../firmware/ac-remote-controller.yaml` 設定。
2. 將 **ESP32-DevKitC** 透過 USB 線連接至 Mac。
3. 執行 `esphome run ac-remote-controller.yaml`，完成初次韌體燒錄並確認設備連上 Wi-Fi。
4. 後續若需修改韌體設定，可直接透過 Wi-Fi 執行 OTA 更新：`esphome upload ac-remote-controller.yaml --device ac-remote-controller.local`。
