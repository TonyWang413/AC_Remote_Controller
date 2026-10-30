# Synology NAS 部署 Home Assistant 與 HomeKit 整合指南

本指南說明如何在 Synology DSM (`Container Manager`) 環境中，透過本專案之 [docker-compose.yml](../server/docker-compose.yml) 部署 **Home Assistant**，並整合 **ESP32 (`192.168.31.118`)**、**小米藍牙溫濕度計 3** 與 **Apple HomeKit**。

---

## 第一步：安裝「Container Manager」套件
1. 登入 Synology NAS 管理後台（DSM）。
2. 開啟 **「套件中心 (Package Center)」**。
3. 搜尋 **`Container Manager`**（DSM 7.1 以前版本稱為 `Docker`），若尚未安裝請先完成安裝。

---

## 第二步：於 File Station 建立設定檔儲存目錄
> [!IMPORTANT]
> Synology `Container Manager` 不會自動建立不存在的掛載子目錄，請務必先於 `File Station` 手動建立對應資料夾，否則容器啟動時將發生路徑錯誤。

1. 開啟 NAS 上的 **「File Station」**。
2. 進入左側的 **`docker`** 共用資料夾。
3. 點擊上方「新增 ➔ 新增資料夾」，建立名為 **`homeassistant`** 的資料夾。
4. 進入 `homeassistant` 資料夾內，再次點擊「新增資料夾」，建立名為 **`config`** 的子資料夾。
   *(完整目錄路徑為：`docker/homeassistant/config`)*

---

## 第三步：於 Container Manager 匯入 `docker-compose.yml`
1. 開啟 NAS 上的 **「Container Manager」**。
2. 點選左側選單之 **「專案 (Project)」**，並點擊右上角 **「新增 (Create)」**。
3. 於設定視窗中依序填寫：
   * **專案名稱**：輸入 `homeassistant`
   * **路徑**：點擊「設定路徑」，選擇第二步建立之 **`/docker/homeassistant`** 資料夾。
   * **來源**：選擇 **「上傳 docker-compose.yml」**（或選擇「建立 docker-compose.yml」並貼上本專案 [docker-compose.yml](../server/docker-compose.yml) 之內容）。
4. 點擊 **「下一步」** ➔ 網頁入口設定維持預設並點擊 **「下一步」** ➔ 勾選「在專案建立後啟動此專案」並點擊 **「完成」**。
5. 系統將自動下載 Home Assistant 映像檔並啟動容器（約需 2～5 分鐘），當狀態顯示為「專案已啟動 (Running)」即代表部署完成。

---

## 第四步：初始化 Home Assistant 並接入 ESP32

1. 開啟瀏覽器，於網址列輸入：
   ```text
   http://您的NAS區網IP:8123
   ```
   *(例如 NAS IP 為 `192.168.31.100`，則輸入 `http://192.168.31.100:8123`)*
2. 點擊 **「建立我的智慧家庭」**，依畫面引導建立管理員帳號與家庭位置設定。
3. **接入 ESP32 (`192.168.31.118`)**：
   * 由於 `docker-compose.yml` 已配置 `network_mode: host`，進入 Home Assistant 後台並點選左側 **「設定 (Settings)」➔「裝置與服務 (Devices & Services)」**，系統通常會自動探索並顯示 **`AC Remote Controller (ESPHome)`**。
   * *(若未自動顯示，可點擊右下角「新增整合」➔ 搜尋 `ESPHome` ➔ 主機位址輸入 `192.168.31.118`)*。
   * 點擊 **「設定 (Configure)」** 並輸入 `../firmware/secrets.yaml` 中設定之 `api_encryption_key`。
4. 完成驗證後，Home Assistant 將載入 ESP32 上定義之 **`日立冷氣 (Hitachi AC)`** 溫控器與 **`送風防霉 1hr 關機 (Fan 1h)`** 開關，並同時啟用 ESP32 的藍牙網關 (`Bluetooth Proxy`) 轉發功能。

---

## 第五步：將小米藍牙溫濕度計 3 加入 Home Assistant
1. 於 **「設定 (Settings)」➔「裝置與服務 (Devices & Services)」** 的「已探索」區塊找到 **`Xiaomi BLE` (`Temperature/Humidity Sensor 84D3`)**，點擊 **「設定」**。
2. 輸入透過 `Xiaomi Cloud Token Extractor` 提取之 32 碼藍牙加密金鑰 (`Bindkey`)。
3. 等待約 30~60 秒接收藍牙廣播封包後，系統即會建立溫度、濕度與電池電量感測實體。

---

## 第六步：設定 `HomeKit Bridge` 橋接至 Apple 家庭 App

為維持 Apple 家庭介面結構清晰，建立橋接器時僅需包含 **`Climate`（空調溫控器）**、**`Switch`（送風防霉定時開關）** 與 **`Sensor`（溫濕度感測器）** 三個網域類別：

1. 於 Home Assistant 左側選單點選 **「設定 (Settings)」➔「裝置與服務 (Devices & Services)」**。
2. 點擊右下角 **「＋ 新增整合 (Add Integration)」**。
3. 搜尋並選擇 **`Apple` ➔ `HomeKit Bridge`**。
4. **選擇要包含的網域 (Domains to include)**：
   * 取消預設勾選之其他項目，僅勾選以下三項：
     * **`Climate`（空調溫控器）**
     * **`Sensor`（感測器）**
     * **`Switch`（開關）**
   * 點擊 **「傳送 (Submit)」**。
5. **於 iPhone「家庭 App」掃描 QR Code 配對**：
   * 點擊 Home Assistant 左下角之 **「通知 (Notifications)」**，檢視系統產生的 **HomeKit 配對 QR Code**。
   * 開啟 iPhone 內建之 **「家庭 (Home)」App**，點擊右上角 **「＋」➔「加入配件 (Add Accessory)」**，掃描畫面上的 QR Code。
   * 當系統提示「未認證的配件 (Uncertified Accessory)」時，點擊 **「仍然加入 (Add Anyway)」**，並依序將橋接器、溫濕度計與冷氣控制元件指派至目標房間（如「客廳」）。
