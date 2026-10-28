# ⚡ 跨裝置極速互傳系統 (CrossDevice DropZone)

一個無伺服器（Serverless）、零維護成本的跨裝置（電腦、手機、平板）檔案與文字快速交換平台。以個人 Google Drive 作為雲端中繼站，突破一般免費工具的檔案大小限制，具備後端密碼驗證、單筆/全部立即清除功能，以及 48 小時自動過期銷毀機制。透過 **GitHub Pages + Google Apps Script (GAS)** 前後端分離架構，徹底解決行動端多重帳號衝突與企業防火牆限制。

---

## 壹、 系統規格書 (System Specification)

### 1. 系統架構與技術堆疊

| 模組層級 | 採用技術 / 服務 | 規格與功能說明 |
| :--- | :--- | :--- |
| **前端託管 (Hosting)** | GitHub Pages (`*.github.io`) | 提供乾淨、簡短的固定網址，穿透企業防火牆限制，並避免瀏覽器載入網頁時觸發 Google 多重帳號 Cookie 路由衝突。 |
| **前端介面 (Frontend)** | HTML5 + Tailwind CSS + Vanilla JS | 響應式網頁設計 (RWD)，支援上傳進度條、單鍵複製、單筆/全部刪除，以及 `sessionStorage` 分頁級別密碼記憶。 |
| **後端接口 (Backend API)** | Google Apps Script (`doPost`) | 純 RESTful 風格 POST 接口（已移除 GAS 內建 HTML[cite: 5]），以 `text/plain` 接收 JSON 請求避開 CORS 預檢 (Preflight)，統一處理密碼驗證與權杖核發。 |
| **儲存中繼站 (Storage)** | Google Drive | 自動於根目錄建立專用資料夾 `CrossDevice_DropZone` 存放所有中繼檔案與文字紀錄[cite: 2, 4]。 |
| **大檔傳輸機制** | Google Drive REST API v3 | 採用 `Resumable Upload`（續傳機制），由前端取得暫時性 OAuth Token 後直接將檔案串流至 Google Drive，繞過 GAS 50MB 限制，單檔上限最高支援至 5TB。 |
| **文字交換機制** | JSON 檔案資料庫 | 於專用資料夾內自動維護 `text_clipboard.json`[cite: 2, 4]，紀錄文字內容、UUID 與時間戳記[cite: 2]。 |
| **資訊安全機制** | 後端強制驗證 + Session 控制 | 前端公開程式碼不含任何密碼；後端 `doPost` 第一時間強制執行 `verifyPassword`[cite: 4]，前端關閉分頁即自動清除授權狀態。 |
| **資料清理機制** | 手動立即清除 + 48h 自動排程 | 支援網頁端「立即清空全部」與「單筆刪除」；同時配置 GAS 每小時排程 `autoCleanUp` 自動銷毀逾 48 小時之檔案與文字[cite: 1, 2]。 |

---

## 貳、 開發與除錯紀錄 (Development & Troubleshooting Log)

本專案在開發過程中經歷了從「單體式 GAS 架構」演進至「GitHub Pages + GAS 前後端分離架構」的實戰除錯過程，關鍵技術決策紀錄如下：

### 🛠️ 紀錄 1：突破 GAS 50MB 檔案上傳限制
* **挑戰**：Google Apps Script 原生 `google.script.run` 傳遞檔案有 50MB 的嚴格上限，無法滿足大檔案跨裝置互傳需求。
* **解法**：由 GAS 後端透過 `ScriptApp.getOAuthToken()` 生成短效授權碼交給前端[cite: 2]，前端改用 `XMLHttpRequest` 直接對 Google Drive API (`uploadType=resumable`) 發起續傳請求，實現不限大小直傳並即時回報上傳進度百分比。

### 🛠️ 紀錄 2：改用 GitHub Pages 解決 Android Chrome 多重 Google 帳號崩潰 (`macros/u/1`)
* **問題現象**：在電腦與 LINE 內建瀏覽器運作正常，但在登入多個 Google 帳號的 Android Chrome 開啟時，網址會被強制轉址為 `script.google.com/macros/u/1/...` 並報錯「很抱歉，目前無法開啟這個檔案」[cite: 3]。
* **根本原因**：直接訪問 `script.google.com` 的 GAS `doGet` 網頁時，Google 伺服器會讀取手機 Chrome 內的多重帳號 Cookie 進行身分路由，引發權限衝突而阻擋載入。
* **解法**：將前端網頁 `index.html` 移轉至 **GitHub Pages (`*.github.io`)** 獨立託管，並將後端 GAS 改為 `doPost(e)` 純 API 模式回傳 JSON。使用者透過手機 Chrome 訪問 GitHub Pages 網址時，完全不會經過 Google 的網頁帳號轉址機制，同時獲得簡短易記的固定網址。

### 🛠️ 紀錄 3：企業網路防火牆穿透 (`pages.dev` 封鎖問題)
* **問題現象**：企業內部網路環境會阻擋 Cloudflare Pages (`*.pages.dev`) 等新興免費代管網域。
* **解法**：選用開發者標準網域 **GitHub Pages (`*.github.io`)** 部署前端靜態網頁，確保在公司內部網路與外部行動網路皆能暢行無阻。

### 🛠️ 紀錄 4：修復首頁 3 秒自動解鎖漏洞與強化密碼驗證
* **問題現象**：初次進入首頁時，未輸入密碼等待約 3 秒會自動跳轉進入主畫面。
* **根本原因**：
  1. 早期版本使用 `localStorage` 永久記憶密碼，網頁載入時於背景自動呼叫 GAS 驗證，因 GAS 冷啟動約需 3 秒，造成「停留首頁 3 秒後自己解鎖」的現象。
  2. 前端未嚴格校驗回傳資料結構，若後端未更新版本誤回傳非預期內容時容易誤判放行。
* **解法**：
  1. 後端 `doPost` 入口第一行強制執行 `verifyPassword(pwd)`，無密碼或密碼錯誤直接拋出例外阻斷[cite: 4]。
  2. 前端廢除 `localStorage` 並自動清除舊有殘留值，改採 `sessionStorage`（僅於當前分頁有效，關閉分頁即鎖定），並加上 `Array.isArray(data.files)` 雙重結構校驗。

---

## 參、 版本變更紀錄 (Changelog)

### `[v2.1.0]` - 最終穩定版 (Current)
* **Added**: 新增「🗑️ 立即清空全部」按鈕，可一鍵將雲端中繼資料夾內所有檔案移至垃圾桶並清空文字紀錄。
* **Added**: 新增單筆文字與單一檔案的個別「刪除」按鈕。
* **Fixed**: 修復首頁因 `localStorage` 殘留導致等待 3 秒自動解鎖的問題，改用 `sessionStorage` 分頁級別記憶機制。
* **Security**: 強化後端 `doPost` 驗證邏輯，所有 API 操作均強制於第一行驗證密碼[cite: 4]；清理 GAS 專案中不再使用的 `index.html` 檔案[cite: 5]。

### `[v2.0.0]` - GitHub Pages 前後端分離架構重構版
* **Changed**: 將前端網頁遷移至 **GitHub Pages**，後端 GAS 改為 `doPost` 純 JSON API 接口。
* **Fixed**: 透過 GitHub Pages 外部託管徹底根治 Android Chrome 多重 Google 帳號登入狀態下無法開啟網頁 (`macros/u/1`) 的問題[cite: 3]。
* **Fixed**: 避開企業防火牆對 `*.pages.dev` 網域的封鎖，並達成短網址效果。

### `[v1.1.0]` - 安全性升級版
* **Added**: 新增後端密碼保護機制 (`ACCESS_PASSWORD`) 與前端解鎖/鎖定畫面[cite: 4]。

### `[v1.0.0]` - 初始原型版
* **Added**: 完成基於 Google Apps Script + Google Drive 的跨裝置文字（JSON 儲存）與不限大小檔案（Resumable Upload）互傳功能[cite: 2]。
* **Added**: 實作 `autoCleanUp` 48 小時自動清除過期檔案與文字排程[cite: 1, 2]。

---

## 肆、 建置與維運說明 (Deployment & Maintenance)

### 1. Google Apps Script 後端設定
1. 於 [script.google.com](https://script.google.com/) 專案中僅需保留 `程式碼.gs` 一個檔案[cite: 5]，貼入後方備份之完整後端程式碼，並自訂 `ACCESS_PASSWORD`[cite: 4]。
2. **觸發條件設定**：左側「觸發條件（鬧鐘圖示）」[cite: 1] ➔ 新增觸發條件 ➔ 執行函式選 `autoCleanUp`[cite: 1, 2] ➔ 時間驅動 ➔ `每小時`。
3. **部署與更新**：
   * 首次部署：右上角「部署」[cite: 1, 4] ➔「新增部署作業」➔ 網頁應用程式（執行身分：**我** / 誰可以存取：**所有人**）。
   * 版本更新：右上角「部署」[cite: 1, 4] ➔「管理部署作業」➔「編輯 ✏️」➔ 版本選「**建立新版本**」➔「部署」（維持 `/exec` 網址不變）。

### 2. GitHub Pages 前端設定
1. 將後方備份之 `index.html` 上傳至 GitHub Public Repository。
2. 修改 `index.html` 內的 `GAS_URL` 變數，填入 GAS 部署取得的 `/exec` 網址。
3. 於 Repository 的 **Settings** ➔ **Pages** 啟用 `main` 分支部署，即可透過 `https://<帳號>.github.io/<專案名>/` 跨裝置使用。

---

## 伍、 最終版原始碼備份 (Final Source Code)

### 1. 後端程式碼 (`程式碼.gs` — 存放於 Google Apps Script)[cite: 5]

```javascript
// ================= 系統參數設定 =================
const FOLDER_NAME = "CrossDevice_DropZone"; // Google Drive 中繼資料夾名稱
const TEXT_FILE_NAME = "text_clipboard.json"; // 文字紀錄檔名稱
const EXPIRE_HOURS = 48; // 自動銷毀時限（小時）
const ACCESS_PASSWORD = "8888"; // 👈 通行密碼（請自行修改）
// ================================================

// 驗證密碼輔助函式
function verifyPassword(pwd) {
  if (!pwd || pwd !== ACCESS_PASSWORD) {
    throw new Error("密碼錯誤！");
  }
}

// 1. 狀態檢查入口 (前端已移至 GitHub Pages)
function doGet() {
  return ContentService.createTextOutput("CrossDevice DropZone API 運作正常");
}

// 2. 供 GitHub Pages 外部前端呼叫的純 API 通道
function doPost(e) {
  try {
    const params = JSON.parse(e.postData.contents);
    const action = params.action;
    const pwd = params.pwd;
    
    // 🔒 強制第一時間驗證密碼
    verifyPassword(pwd);

    let result;
    if (action === 'getItems') {
      result = getItems(pwd);
    } else if (action === 'saveTextItem') {
      result = saveTextItem(params.text, pwd);
    } else if (action === 'getUploadConfig') {
      result = getUploadConfig(pwd);
    } else if (action === 'clearAllItems') {
      result = clearAllItems(pwd);
    } else if (action === 'deleteFileItem') {
      result = deleteFileItem(params.fileId, pwd);
    } else if (action === 'deleteTextItem') {
      result = deleteTextItem(params.textId, pwd);
    } else {
      throw new Error("未知的操作請求");
    }

    return ContentService.createTextOutput(JSON.stringify({ status: 'ok', data: result }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({ status: 'error', message: err.message }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

// 3. 取得或建立專用資料夾
function getDropFolder() {
  const folders = DriveApp.getFoldersByName(FOLDER_NAME);
  if (folders.hasNext()) {
    return folders.next();
  }
  return DriveApp.createFolder(FOLDER_NAME);
}

// 4. 供前端取得 OAuth Token 與資料夾 ID (用於前端不限大小直傳)
function getUploadConfig(pwd) {
  verifyPassword(pwd);
  const folder = getDropFolder();
  return {
    token: ScriptApp.getOAuthToken(),
    folderId: folder.getId()
  };
}

// 5. 儲存文字訊息
function saveTextItem(text, pwd) {
  verifyPassword(pwd);
  const folder = getDropFolder();
  const files = folder.getFilesByName(TEXT_FILE_NAME);
  let list = [];
  let file;

  if (files.hasNext()) {
    file = files.next();
    const content = file.getBlob().getDataAsString();
    if (content) list = JSON.parse(content);
  }

  list.unshift({
    id: Utilities.getUuid(),
    text: text,
    timestamp: new Date().getTime()
  });

  const now = new Date().getTime();
  list = list.filter(item => (now - item.timestamp) < EXPIRE_HOURS * 3600 * 1000);

  const jsonString = JSON.stringify(list);
  if (file) {
    file.setContent(jsonString);
  } else {
    folder.createFile(TEXT_FILE_NAME, jsonString, MimeType.PLAIN_TEXT);
  }
  return list;
}

// 6. 讀取目前的檔案列表與文字列表
function getItems(pwd) {
  verifyPassword(pwd);
  const folder = getDropFolder();
  const now = new Date().getTime();
  
  let texts = [];
  const textFiles = folder.getFilesByName(TEXT_FILE_NAME);
  if (textFiles.hasNext()) {
    const content = textFiles.next().getBlob().getDataAsString();
    if (content) {
      texts = JSON.parse(content).filter(item => (now - item.timestamp) < EXPIRE_HOURS * 3600 * 1000);
    }
  }

  const files = folder.getFiles();
  const fileList = [];
  while (files.hasNext()) {
    const f = files.next();
    if (f.getName() !== TEXT_FILE_NAME && !f.isTrashed()) {
      fileList.push({
        id: f.getId(),
        name: f.getName(),
        size: f.getSize(),
        created: f.getDateCreated().getTime(),
        url: f.getDownloadUrl() || f.getUrl()
      });
    }
  }
  
  fileList.sort((a, b) => b.created - a.created);
  return { texts: texts, files: fileList };
}

// 7. 立即清空所有檔案與文字
function clearAllItems(pwd) {
  verifyPassword(pwd);
  const folder = getDropFolder();
  const files = folder.getFiles();

  while (files.hasNext()) {
    const file = files.next();
    if (file.
