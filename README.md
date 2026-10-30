# 外出冷氣控制系統 (AC Remote Controller)

本專案為基於 **ESP32-DevKitC (`ESPHome`)**、**M5Stack U002 紅外線收發模組**、**小米藍牙溫濕度計 3** 與 **Synology NAS (`Home Assistant` + `Apple HomeKit`)** 建置之遠端冷氣控制與環境監測系統。

系統整合至 iOS / macOS **「Apple 家庭 (Home)」App** 與 **Siri**，支援遠端空調模式切換、溫度調節、即時出風口溫濕度監測，以及 **「送風防霉 1 小時定時自動關機」** 功能。

---

## 專案目錄結構 (Directory Structure)

```text
AC_Remote_Controller/
├── README.md                        # 專案總覽與文件索引
├── .clinerules                      # 專案文件與開發維護規範
├── .gitignore                       # Git 忽略清單 (排除私密金鑰與編譯快取)
│
├── firmware/                        # ESP32 韌體設定目錄 (ESPHome)
│   ├── ac-remote-controller.yaml    # ESP32 韌體主設定檔 (BLE Proxy + Hitachi AC344 + 1hr 送風定時)
│   ├── secrets.yaml                 # Wi-Fi 憑證與 API 加密金鑰 (由 .gitignore 排除不上傳)
│   └── secrets.yaml.example         # 私密金鑰設定檔範本
│
├── server/                          # 伺服器部署目錄 (Home Assistant)
│   └── docker-compose.yml           # Synology Container Manager (Docker) 部署設定檔
│
└── docs/                            # 系統規格與建置文件
    ├── step_by_step_guide.md        # 系統建置與部署手冊 (完整端到端建置流程)
    ├── system_spec.md               # 系統設計規格書 (硬體接線、系統架構、韌體規格)
    ├── esphome_setup_guide.md       # macOS ESPHome 開發環境建置與指令指南
    ├── synology_ha_setup_guide.md   # Synology NAS Home Assistant 與 HomeKit Bridge 設定指南
    └── brainstorm_notes.md          # 系統規劃與開發紀錄
```

---

## 文件導覽 (Documentation Index)

若需從零開始建置本系統，建議依下列順序閱讀相關文件：

1. **[系統建置與部署手冊 (`./docs/step_by_step_guide.md`)](./docs/step_by_step_guide.md)**
   * 涵蓋硬體材料清單、macOS 安裝 ESPHome、產生通訊金鑰、初次 USB 燒錄與後續 Wi-Fi OTA 更新、M5Stack U002 腳位接線與注意事項、`remote_receiver` 擷取與解碼日立 344-bit 紅外線協定、提取小米溫濕度計 3 `Bindkey`，以及 Apple HomeKit 整合步驟。
2. **[系統設計規格書 (`./docs/system_spec.md`)](./docs/system_spec.md)**
   * 定義系統架構圖、ESP32-DevKitC 與 M5Stack U002 腳位對應關係、ESPHome 韌體規格、Home Assistant 實體對應與測試驗證計畫。
3. **[macOS ESPHome 開發環境指南 (`./docs/esphome_setup_guide.md`)](./docs/esphome_setup_guide.md)**
   * 說明於 macOS 安裝 `esphome`、檢查 USB 序列埠 (`/dev/cu.*`) 及常用維護指令（`compile`、`upload`、`logs`）。
4. **[Synology NAS Home Assistant 部署指南 (`./docs/synology_ha_setup_guide.md`)](./docs/synology_ha_setup_guide.md)**
   * 說明 Synology NAS `Container Manager` 匯入 [./server/docker-compose.yml](./server/docker-compose.yml) 流程、接入 ESP32 與小米溫濕度計 3、設定 `HomeKit Bridge` 及家庭成員共享權限設定。
5. **[系統規劃與開發紀錄 (`./docs/brainstorm_notes.md`)](./docs/brainstorm_notes.md)**
   * 記錄本專案之硬體選型、架構決策與實機驗證歷程。

---

## 韌體更新與維護指令 (Quick Commands)

修改 [./firmware/ac-remote-controller.yaml](./firmware/ac-remote-controller.yaml) 後，可於終端機執行以下指令，透過區域網路 Wi-Fi (OTA) 直接更新 ESP32 韌體：

```bash
cd firmware
esphome compile ac-remote-controller.yaml && \
esphome upload ac-remote-controller.yaml --device ac-remote-controller.local
```
