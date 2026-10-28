# ⚡ 跨裝置極速互傳系統 (CrossDevice DropZone)

一個無伺服器（Serverless）、零維護成本的跨裝置（電腦、手機、平板）檔案與文字快速交換平台。以個人 Google Drive 作為雲端中繼站，突破一般免費工具的檔案大小限制，具備後端密碼驗證與 48 小時自動過期銷毀機制，並透過前後端分離架構完美解決行動端多重帳號衝突與企業防火牆限制。

---

## 壹、 系統規格書 (System Specification)

### 1. 系統架構與技術堆疊

本系統採用 **「GitHub Pages (前端) + Google Apps Script (後端 API) + Google Drive (雲端儲存)」** 的前後端分離架構：

| 模組層級 | 採用技術 / 服務 | 規格與功能說明 |
| :--- | :--- | :--- |
| **前端託管 (Hosting)** | GitHub Pages (`*.github.io`) | 提供乾淨、簡短的固定網址，避開企業防火牆對免費跳板網域（如 `*.pages.dev`）的封鎖，且完全不夾帶 Google 網域 Cookie。 |
| **前端介面 (Frontend)** | HTML5 + Tailwind CSS + Vanilla JS | 響應式網頁設計 (RWD)，支援進度條顯示、單鍵複製與 LocalStorage 密碼記憶功能。 |
| **後端接口 (Backend API)** | Google Apps Script (`doPost`) | 採用 RESTful 風格之 POST 接口，以 `text/plain` 格式接收 JSON 請求，避開 CORS 預檢 (Preflight) 限制並處理密碼驗證與權杖核發。 |
| **儲存中繼站 (Storage)** | Google Drive | 自動於根目錄建立專用資料夾 `CrossDevice_DropZone` 存放所有中繼檔案與文字紀錄。 |
| **大檔傳輸機制** | Google Drive REST API v3 | 採用 `Resumable Upload`（續傳機制），由前端取得暫時性 OAuth Token 後直接將檔案串流至 Google Drive，繞過 GAS 50MB 限制，單檔上限最高支援至 5TB。 |
| **文字交換機制** | JSON 檔案資料庫 | 於專用資料夾內自動維護 `text_clipboard.json`，紀錄文字內容、UUID 與時間戳記。 |
| **資訊安全機制** | 後端密碼驗證 | 公開之 GitHub 前端程式碼不包含任何密碼；所有讀寫與上傳請求皆須通過 GAS 後端 `ACCESS_PASSWORD` 比對。 |
| **自動化排程** | GAS Time-driven Triggers | 每小時自動執行清理腳本 `autoCleanUp`，將超過 48 小時的檔案移至垃圾桶，並於讀寫時自動剔除過期文字。 |

---

### 2. 關鍵架構決策與問題解決紀錄 (Architecture Decision Records)

#### 🛠️ 問題一：Android Chrome 多重 Google 帳號衝突 (`script.google.com/macros/u/1/...` 錯誤)
* **問題現象**：在電腦與 LINE 內建瀏覽器可正常開啟 GAS 網頁，但在登入多個 Google 帳號的 Android Chrome 瀏覽器開啟時，Google 伺服器會強制將網址轉址為 `/macros/u/1/...` 並顯示「很抱歉，目前無法開啟這個檔案」[cite: 3]。
* **根本原因**：Google Apps Script 的 `HtmlService` (`doGet`) 在瀏覽器帶有複數 Google 帳號 Cookie 時，內建的身分路由機制會發生權限混淆而阻擋載入。
* **解決方案**：改採**前後端分離架構**。將 `index.html` 移至外部靜態空間，並將 GAS 改為 `doPost` 純資料 API。手機瀏覽器載入外部網頁時完全不觸發 Google 帳號路由檢查，透過背景 `fetch()` 呼叫 GAS `doPost` 取回 JSON 資料，100% 根治多重帳號衝突。

#### 🛡️ 問題二：企業網路防火牆封鎖與網址簡化
* **問題現象**：GAS 原始部署網址過長不便跨裝置輸入，且部分企業防火牆會封鎖 `*.pages.dev`（Cloudflare）等新興免費代管網域。
* **解決方案**：採用開發者標準網域 **GitHub Pages (`*.github.io`)** 託管前端頁面，兼具「短網址好記」與「企業網路白名單放行」雙重優勢。

---

## 貳、 建置與維運說明書 (Deployment & Maintenance Guide)

### 1. Google Apps Script 後端建置步驟
1. 前往 [script.google.com](https://script.google.com/) 建立新專案，將後端程式碼貼入 `程式碼.gs`[cite: 1]，並修改頂端的 `ACCESS_PASSWORD`。
2. **初次授權**：於頂端工具列選擇 `getDropFolder` 並點擊「執行」，核准 Google Drive 存取權限。
3. **設定 48 小時自動清除排程**：
   * 點選左側「**觸發條件（鬧鐘圖示）**」[cite: 1] ➔「**新增觸發條件**」。
   * 執行函式選擇 `autoCleanUp`[cite: 1, 2] ➔ 活動來源選 `時間驅動` ➔ 類型選 `小時基準計時器` ➔ `每小時`。
4. **部署 API 端點**：
   * 點擊右上角「部署」[cite: 1] ➔「新增部署作業」➔ 類型選「**網頁應用程式**」。
   * **執行身分**：`我`
   * **誰可以存取**：`所有人`
   * 部署後複製產生的 `https://script.google.com/macros/s/.../exec` 網址。
   * *(註：日後若修改後端程式碼或密碼，請由「部署」[cite: 1] ➔「管理部署作業」➔「編輯 ✏️」➔「建立新版本」更新，以維持 `/exec` 網址不變。)*

### 2. GitHub Pages 前端部署步驟
1. 於 GitHub 建立一個 Public Repository（例如命名為 `drop`）。
2. 建立 `index.html`，將前端程式碼貼入，並將程式碼中的 `GAS_URL` 變數替換為步驟 1 取得的 `/exec` 網址。
3. 進入 Repository 的 **Settings** ➔ **Pages**，於 **Branch** 選擇 `main` / `/ (root)` 並點擊 **Save**。
4. 等待約 1 分鐘即可透過 `https://<GitHub帳號>.github.io/<Repo名稱>/` 存取系統。

---

## 參、 使用說明書 (User Manual)

1. **快速開啟**：
   * **電腦端**：開啟 GitHub Pages 網址並加入書籤列。
   * **手機端**：於 Chrome / Safari 開啟網址後，點選瀏覽器選單「**加到主畫面**」，即可作為桌面 App 使用。
2. **安全解鎖與鎖定**：
   * 首次進入請輸入通行密碼，系統驗證成功後會自動記憶於該裝置瀏覽器中，下次開啟免重複輸入。
   * 若於公用電腦使用完畢，請點擊畫面右上角紅色的「**鎖定**」按鈕清除記憶狀態。
3. **文字互傳**：
   * 輸入文字後點擊「傳送文字」即可同步至雲端；於另一台裝置點擊「重新整理」並按下「複製」即可取用。
4. **不限大小檔案互傳**：
   * 選取檔案後點擊「開始上傳」，待進度條達 100% 即完成直傳；於另一台裝置點擊「下載 / 開啟」即可取得檔案。
5. **實體路徑與自動銷毀**：
   * 所有中繼資料皆位於 Google Drive 根目錄下的 `CrossDevice_DropZone` 資料夾中，滿 48 小時後由排程自動移至垃圾桶[cite: 2]。

---

## 肆、 專案最終原始碼備份 (Source Code)

### 1. 後端程式碼 (`程式碼.gs` - 存放於 Google Apps Script)[cite: 1]

```javascript
// ================= 系統參數設定 =================
const FOLDER_NAME = "CrossDevice_DropZone"; // Google Drive 中繼資料夾名稱
const TEXT_FILE_NAME = "text_clipboard.json"; // 文字紀錄檔名稱
const EXPIRE_HOURS = 48; // 自動銷毀時限（小時）
const ACCESS_PASSWORD = "8888"; // 通行密碼（請自行修改）
// ================================================

function verifyPassword(pwd) {
  if (pwd !== ACCESS_PASSWORD) {
    throw new Error("密碼錯誤！");
  }
}

// 保留 doGet 供直接測試，並開啟 iframe 相容性
function doGet() {
  return HtmlService.createHtmlOutputFromFile('index')
    .setTitle('跨裝置極速互傳')
    .addMetaTag('viewport', 'width=device-width, initial-scale=1')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

// 供 GitHub Pages 外部前端呼叫的純 API 通道（根治 Google 多重帳號 Bug）
function doPost(e) {
  try {
    const params = JSON.parse(e.postData.contents);
    const action = params.action;
    const pwd = params.pwd;
    let result;

    if (action === 'getItems') {
      result = getItems(pwd);
    } else if (action === 'saveTextItem') {
      result = saveTextItem(params.text, pwd);
    } else if (action === 'getUploadConfig') {
      result = getUploadConfig(pwd);
    }

    return ContentService.createTextOutput(JSON.stringify({ status: 'ok', data: result }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({ status: 'error', message: err.message }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function getDropFolder() {
  const folders = DriveApp.getFoldersByName(FOLDER_NAME);
  if (folders.hasNext()) {
    return folders.next();
  }
  return DriveApp.createFolder(FOLDER_NAME);
}

function getUploadConfig(pwd) {
  verifyPassword(pwd);
  const folder = getDropFolder();
  return {
    token: ScriptApp.getOAuthToken(),
    folderId: folder.getId()
  };
}

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

function autoCleanUp() {
  const folder = getDropFolder();
  const files = folder.getFiles();
  const now = new Date().getTime();
  const expireMs = EXPIRE_HOURS * 3600 * 1000;

  while (files.hasNext()) {
    const file = files.next();
    if (file.getName() === TEXT_FILE_NAME) continue;
    
    const createdTime = file.getDateCreated().getTime();
    if (now - createdTime > expireMs) {
      file.setTrashed(true);
    }
  }
}
