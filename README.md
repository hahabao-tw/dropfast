# ⚡ DropZone 跨裝置極速互傳系統

一個無伺服器（Serverless）、零維護成本的跨裝置（電腦、手機、平板）檔案與文字快速交換平台。以個人 Google Drive 作為雲端中繼站，單檔支援高達 **1TB** 直傳，具備**單筆項目獨立密碼鎖**、**單筆/全部立即清除功能**，以及 **6 小時自動過期銷毀機制**。透過 **GitHub Pages + Google Apps Script (GAS)** 前後端分離架構，徹底解決行動端多重帳號衝突與企業防火牆限制。

---

## 壹、 系統規格書 (System Specification)

| 模組層級 | 採用技術 / 服務 | 規格與功能說明 |
| :--- | :--- | :--- |
| **前端託管 (Hosting)** | GitHub Pages (`*.github.io`) | 提供簡短固定網址，穿透企業防火牆限制，並避免瀏覽器載入時觸發 Google 多重帳號 Cookie 路由衝突。 |
| **前端介面 (Frontend)** | HTML5 + Tailwind CSS + Apple 風格模組 | 採用 Apple Control Center 設計語言，以太空灰導覽列搭配「Apple 藍（文字專區）」與「翡翠綠（檔案專區）」高對比卡片，內建容量單位自動換算 (`B` ~ `TB`)。 |
| **後端接口 (Backend API)** | Google Apps Script (`doPost`) | 純 RESTful 風格 POST 接口，以 `text/plain` 接收 JSON 請求避開 CORS 預檢 (Preflight)，統一由 `metadata.json` 管理所有項目狀態。 |
| **大檔傳輸與上限** | Google Drive REST API v3 | 採用 `Resumable Upload`（續傳機制）前端直傳 Google Drive，繞過 GAS 50MB 限制；前後端雙重驗證**單檔上限 1TB**。 |
| **獨立密碼鎖機制** | 單筆項目加密驗證 (`metadata.json`) | 免全站登入即可快速使用；可針對單筆文字或單一檔案設定專屬密碼。受保護文字預設不回傳明文，需輸入密碼當場解鎖；受保護檔案需驗證密碼始可取得下載連結。 |
| **跨裝置無障礙下載** | DriveApp 動態權限核發 | 下載檔案通過密碼驗證時，後端自動設定 `ANYONE_WITH_LINK` 權限並回傳 `uc?export=download` 直連網址，根治跨裝置 Google 帳號 403 權限阻擋問題[cite: 7]。 |
| **資料清理機制** | 手動立即清除 + 6h 自動排程 | 支援網頁端「🗑️ 清空全部」與「單筆刪除」；同時於每次讀寫及每小時排程 `autoCleanUp` 自動銷毀逾 **6 小時**之過期檔案與文字[cite: 1, 2]。 |

---

## 貳、 版本變更紀錄 (Changelog)

### `[v3.1.0]` - 容量規範與視覺強化版 (Current)
* **Added**: 新增單檔上傳 **1TB (`1,099,511,627,776 Bytes`)** 上限檢查（前端上傳前預先攔截 + 後端 `registerFile` 雙重驗證）。
* **Added**: 新增 `formatBytes()` 智慧容量單位轉換函式，自動依檔案大小與上傳進度顯示 `B / KB / MB / GB / TB`。
* **Changed**: 強化視覺層次與區塊辨識度，將背景加深為霧灰藍 (`#e2e8f0`)，並為「文字剪貼簿專區（Apple 藍）」與「檔案傳輸專區（翡翠綠）」加上頂部識別色條與獨立內艙設計。
* **Fixed**: 修復加密文字解鎖後因重新拉取列表導致明文再次被隱藏的問題（新增前端 `unlockedTextsCache` 快取展開機制）。
* **Fixed**: 修復跨裝置或未登入 Google 帳號下載檔案時出現 `403 Forbidden` 無權限畫面的問題（後端於驗證通過後動態開啟 `ANYONE_WITH_LINK` 檢視權限）[cite: 7]。

### `[v3.0.0]` - 單筆項目密碼鎖與 Apple 風格改版
* **Changed**: 取消全站登入密碼門檻，改為**針對單次傳送之文字或檔案個別設定密碼鎖（選填）**。
* **Changed**: 自動銷毀時限由 48 小時縮短為 **6 小時**，提升雲端中繼空間周轉率與隱私安全性。
* **Changed**: 後端儲存結構由純文字檔升級為 `metadata.json` 元數據管理結構，整合紀錄檔案 ID、容量、時間戳記與個別密碼。
* **Changed**: 全站 UI 重新設計，導入 Apple 官方極簡設計語言。

### `[v2.1.0]` - 手動清理與安全性加固版
* **Added**: 新增「🗑️ 立即清空全部」按鈕與單筆文字/檔案個別「刪除」按鈕。
* **Fixed**: 修復首頁因 `localStorage` 殘留導致等待 3 秒自動跳轉解鎖的問題。

### `[v2.0.0]` - GitHub Pages 前後端分離架構重構版
* **Changed**: 將前端網頁遷移至 **GitHub Pages**，後端 GAS 改為 `doPost` 純 JSON API 接口。
* **Fixed**: 徹底根治 Android Chrome 多重 Google 帳號登入狀態下無法開啟網頁 (`script.google.com/macros/u/1/...`) 的問題[cite: 3]。
* **Fixed**: 避開企業網路防火牆對免費代管網域 (`*.pages.dev`) 的封鎖。

### `[v1.0.0]` - 初始原型版
* **Added**: 完成基於 Google Apps Script + Google Drive Resumable Upload 的跨裝置文字與檔案互傳原型，並配置 `autoCleanUp` 自動清理排程[cite: 1, 2]。
