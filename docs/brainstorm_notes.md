# 外出冷氣控制系統 (AC Remote Controller) — 系統規劃與開發紀錄

本文件記錄遠端冷氣控制系統之需求規劃、硬體選型、架構演進與實機驗證歷程。

---

## 1. 專案目標
建置一套穩定、安全的遠端冷氣控制與環境監測系統，讓使用者可於外出返家前透過 Apple 家庭 (Home) App 提前開啟冷氣或切換模式，並藉由冷氣出風口之即時溫度變化確認冷氣實際運轉狀態。

---

## 2. 硬體設備清單 (Confirmed Hardware BOM)
本系統採用之實體硬體設備如下：
1. **核心開發板**：**ESP32-DevKitC 開發板 - 板載天線 (`ESP32-WROOM-32D`)**
   * 同時負責「紅外線收發控制」與「藍牙網關 (`ESPHome Bluetooth Proxy`)」功能。
2. **紅外線收發模組**：**M5Stack 紅外線收發模組 (`U002`)**
   * 內建 **紅外線發射 (`OUT` / `IR_TX`)**（含 5V 電晶體放大驅動電路）與 **38kHz 解調紅外線接收 (`IN` / `IR_RX`)**。
   * 透過 `IR_RX` 接收功能可直接擷取並分析原廠日立遙控器訊號，確認其採用 43 Bytes (344-bit) 之 `hitachi_ac344` 協定。
   * 透過 **「Grove (HY2.0-4P) 轉 4Pin 2.54mm 杜邦母頭線」** 將 U002 的 `5V`、`GND`、`OUT (GPIO19)`、`IN (GPIO18)` 連接至 ESP32-DevKitC 排針。
3. **溫濕度感測器**：**小米藍牙溫濕度計 3 (`MJWSD05MMC`)**
   * 部署於冷氣出風口下方約 20~30 公分處。
   * 採用 MiBeacon 加密藍牙廣播，需先綁定米家 App 並透過 `Xiaomi Cloud Token Extractor` 取得 32 位元十六進位 **`Bindkey`**，再接入 Home Assistant 之 `Xiaomi BLE` 整合。

---

## 3. 系統架構與設計演進 (Architecture & Design Evolution)

### 3.1 硬體與韌體控制端 (ESP32 + ESPHome)
* **初期概念驗證 (v1 Prototype)**：
  * 最初規劃錄製原廠遙控器之「開機」與「關機」兩組訊號，並配置兩個 1 秒自動彈回之點按式開關 (`Momentary Switch`)。
* **正式版架構 (`hitachi_ac344` 原生溫控器 + 送風防霉定時器)**：
  * 經由 `M5Stack U002` 實機擷取日立遙控器紅外線訊號後，確認其完整相容於 ESPHome 內建之 **`hitachi_ac344`** 平台。
  * 最終韌體整合為兩個核心功能實體：
    1. **`日立冷氣 (Hitachi AC)` (`climate.hitachi_ac344`)**：於 Apple 家庭呈現為標準溫控器轉盤，支援 `Off` / `Cool` / `Heat` / `Auto` / `Dry` / `Fan Only` 模式切換、溫度設定與風速調整，並將小米溫濕度計 3 之即時溫濕度綁定至溫控器的當前溫度（`Current`）欄位。
    2. **`送風防霉 1hr 關機 (Fan 1h)` (`switch.template` + `script`)**：針對冷氣關機前需送風吹乾內部冷凝水以預防發霉之需求，點擊後立即切換冷氣為「送風 (`FAN_ONLY`) + 低風速 (`LOW`)」，並由 ESP32 內部計時器 (`script.fan_1hr_timer`) 倒數 60 分鐘後自動發射 `OFF` 關機指令。

### 3.2 伺服器端 (Synology NAS Home Assistant)
* **部署環境**：透過 Synology NAS `Container Manager` (Docker) 運行 Home Assistant Core（配置 `network_mode: host`）。
* **通訊機制**：
  * **ESPHome 整合**：透過區網 Wi-Fi 加密 API 連接 ESP32-DevKitC。
  * **Bluetooth Proxy + Xiaomi BLE**：由 ESP32 接收小米溫濕度計 3 之 BLE 廣播並轉發予 Home Assistant 解密。

### 3.3 使用者控制端與遠端存取 (Apple HomeKit)
* 透過 Home Assistant `HomeKit Bridge` 將空調溫控器、送風防霉定時開關與溫濕度計橋接至 Apple 家庭 App。
* 透過家中 Apple 家庭中樞（HomePod / Apple TV）經由 iCloud 加密通道進行外網遠端控制，無需於路由器設定通訊埠轉發 (Port Forwarding)。

---

## 4. 建置進度與驗證紀錄 (Implementation Checklist)
- [x] **第一階段：確定硬體方案與冷氣控制介面**
    * 完成硬體準備：`ESP32-DevKitC (WROOM-32D)`、`M5Stack 紅外線收發模組 (U002)`、`小米藍牙溫濕度計 3 (MJWSD05MMC)`。
- [x] **第二階段：設計伺服器通訊與 Home Assistant 部署架構**
    * 確立系統架構：Synology NAS 運行 HA ➔ ESPHome (IR + BLE Proxy) + Xiaomi BLE (Bindkey) ➔ HomeKit Bridge。
- [x] **第三階段：制定系統規格書**
    * 完成系統設計規格書 [system_spec.md](./system_spec.md)。
- [x] **第四階段：硬體接線、韌體燒錄與實機驗證**
    * [x] 撰寫 macOS ESPHome 環境建置指南 [esphome_setup_guide.md](./esphome_setup_guide.md) 並透過 Homebrew 安裝 `esphome` (v2026.10.0)。
    * [x] 建立韌體設定檔 [ac-remote-controller.yaml](../firmware/ac-remote-controller.yaml) 與金鑰設定檔 `../firmware/secrets.yaml`（含範本檔 [secrets.yaml.example](../firmware/secrets.yaml.example) 與 [.gitignore](../.gitignore)）。
    * [x] 完成 `ESP32-DevKitC` 初次 USB 燒錄並連上家中 2.4GHz Wi-Fi（IP: `192.168.31.118` / `ac-remote-controller.local`）。
    * [x] 建立 Synology NAS 專用之 [docker-compose.yml](../server/docker-compose.yml) 與部署指南 [synology_ha_setup_guide.md](./synology_ha_setup_guide.md)。
    * [x] 於 Synology NAS 部署 Home Assistant 並接入 ESP32 (`192.168.31.118`)。
    * [x] 提取小米藍牙溫濕度計 3 (`xiaomi.sensor_ht.o3`) 之 `Bindkey` 並接入 Home Assistant (`Xiaomi BLE`)。
    * [x] 完成 ESP32-DevKitC 與 M5Stack U002 接線（確認 U002 外殼標示之 `OUT` 為紅外線發射 `IR_TX` 接 `GPIO19`，`IN` 為紅外線接收 `IR_RX` 接 `GPIO18`）。
    * [x] 透過 M5Stack U002 (`IR_RX`) 接收並解碼日立冷氣 344-bit (`HITACHI_AC344`, 43 bytes) 協定。
    * [x] 整合 `hitachi_ac344` 空調溫控實體 (`Climate`)、串接小米溫濕度計 3 即時室溫，並實作「送風防霉 1hr 關機 (`Fan 1h`)」自動倒數計時功能。
    * [x] 設定 `HomeKit Bridge` 並完成多支家庭成員 iPhone 於 Apple 家庭「客廳」房間之同步控制驗證。
    * [x] 完成專案目錄結構整理（`firmware/`、`server/`、`docs/`）並編撰系統建置手冊 [step_by_step_guide.md](./step_by_step_guide.md) 與專案總覽 [README.md](../README.md)。
