# ⚡ 跨裝置極速互傳系統 (CrossDevice DropZone)

一個無伺服器（Serverless）、零維護成本的跨裝置（電腦、手機、平板）檔案與文字快速交換平台。以個人 Google Drive 作為雲端中繼站，突破一般免費工具的檔案大小限制，具備後端密碼驗證、單筆/全部立即清除功能，以及 48 小時自動過期銷毀機制。透過 **GitHub Pages + Google Apps Script (GAS)** 前後端分離架構，徹底解決行動端多重帳號衝突與企業防火牆限制。

---

## 壹、 系統規格書 (System Specification)

### 1. 系統架構與技術堆疊

| 模組層級 | 採用技術 / 服務 | 規格與功能說明 |
| :--- | :--- | :--- |
| **前端託管 (Hosting)** | GitHub Pages (`*.github.io`) | 提供乾淨、簡短的固定網址，穿透企業防火牆限制，並避免夾帶 Google 網域 Cookie。 |
| **前端介面 (Frontend)** | HTML5 + Tailwind CSS + Vanilla JS | 響應式網頁設計 (RWD)，支援上傳進度條、單鍵複製、單筆/全部刪除，以及 `sessionStorage` 分頁級別密碼記憶。 |
| **後端接口 (Backend API)** | Google Apps Script (`doPost`) | 純 RESTful 風格 POST 接口（已移除 GAS 內建 HTML[cite: 5]），以 `text/plain` 接收 JSON 請求避開 CORS 預檢 (Preflight)，統一處理密碼驗證與權杖核發。 |
| **儲存中繼站 (Storage)** | Google Drive | 自動於根目錄建立專用資料夾 `CrossDevice_DropZone` 存放所有中繼檔案與文字紀錄[cite: 2, 4]。 |
| **大檔傳輸機制** | Google Drive REST API v3 | 採用 `Resumable Upload`（續傳機制），由前端取得暫時性 OAuth Token 後直接將檔案串流至 Google Drive，繞過 GAS 50MB 限制，單檔上限最高支援至 5TB。 |
| **文字交換機制** | JSON 檔案資料庫 | 於專用資料夾內自動維護 `text_clipboard.json`[cite: 2, 4]，紀錄文字內容、UUID 與時間戳記[cite: 2]。 |
| **資訊安全機制** | 後端強制驗證 + Session 控制 | 前端公開程式碼不含任何密碼；後端 `doPost` 第一時間強制執行 `verifyPassword`[cite: 4]，前端關閉分頁即自動清除授權狀態。 |
| **資料清理機制** | 手動立即清除 + 48h 自動排程 | 支援網頁端「立即清空全部」與「單筆刪除」；同時配置 GAS 每小時排程 `autoCleanUp` 自動銷毀逾 48 小時之檔案與文字[cite: 1, 2]。 |

---

## 貳、 開發與除錯紀錄 (Development & Troubleshooting Log)

本專案在開發過程中經歷了從「單體式 GAS 架構」演進至「前後端分離架構」的實戰除錯過程，關鍵技術決策紀錄如下：

### 🛠️ 紀錄 1：突破 GAS 50MB 檔案上傳限制
* **挑戰**：Google Apps Script 原生 `google.script.run` 傳遞檔案有 50MB 的嚴格上限，無法滿足大檔案跨裝置互傳需求。
* **解法**：由 GAS 後端透過 `ScriptApp.getOAuthToken()` 生成短效授權碼交給前端[cite: 2]，前端改用 `XMLHttpRequest` 直接對 Google Drive API (`uploadType=resumable`) 發起續傳請求，實現不限大小直傳並即時回報上傳進度百分比。

### 🛠️ 紀錄 2：解決 Android Chrome 多重 Google 帳號轉址崩潰 (`macros/u/1`)
* **問題現象**：在電腦與 LINE 內建瀏覽器運作正常，但在登入多個 Google 帳號的 Android Chrome 開啟時，網址會被強制轉址為 `script.google.com/macros/u/1/...` 並報錯「很抱歉，目前無法開啟這個檔案」[cite: 3]。
* **根本原因**：直接訪問 GAS `doGet` 網頁時，Google 會讀取瀏覽器內的多重帳號 Cookie 進行身分路由，進而引發權限衝突。
* **解法**：將架構重構為**前後端分離**。前端 `index.html` 獨立移出，後端 GAS 廢除 HTML 渲染，改由 `doPost(e)` 純 API 模式以 `ContentService` 回傳 JSON，徹底繞過 Google 網頁身分轉址機制。

### 🛠️ 紀錄 3：企業網路防火牆穿透 (`pages.dev` 封鎖問題)
* **問題現象**：企業內部網路環境會阻擋 Cloudflare Pages (`*.pages.dev`) 等新興免費代管網域。
* **解法**：改用開發者標準網域 **GitHub Pages (`*.github.io`)** 部署前端靜態網頁，同時解決「GAS 原始網址過長」與「企業防火牆阻擋」兩大痛點。

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

### `[v2.0.0]` - 前後端分離架構重構版
* **Changed**: 將前端網頁遷移至 **GitHub Pages**，後端 GAS 改為 `doPost` 純 JSON API 接口。
* **Fixed**: 徹底根治 Android Chrome 多重 Google 帳號登入狀態下無法開啟網頁 (`macros/u/1`) 的問題[cite: 3]。
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
    if (file.getName() === TEXT_FILE_NAME) {
      file.setContent(JSON.stringify([]));
    } else {
      file.setTrashed(true);
    }
  }
  return { texts: [], files: [] };
}

// 8. 刪除單一檔案
function deleteFileItem(fileId, pwd) {
  verifyPassword(pwd);
  try {
    DriveApp.getFileById(fileId).setTrashed(true);
  } catch (e) {}
  return getItems(pwd);
}

// 9. 刪除單筆文字
function deleteTextItem(textId, pwd) {
  verifyPassword(pwd);
  const folder = getDropFolder();
  const files = folder.getFilesByName(TEXT_FILE_NAME);
  if (files.hasNext()) {
    const file = files.next();
    const content = file.getBlob().getDataAsString();
    if (content) {
      let list = JSON.parse(content);
      list = list.filter(item => item.id !== textId);
      file.setContent(JSON.stringify(list));
    }
  }
  return getItems(pwd);
}

// 10. 48小時自動清理排程 (每小時觸發)
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
```

### 2. 前端程式碼 (`index.html` — 存放於 GitHub Repository)

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>跨裝置極速互傳</title>
  <script src="[https://cdn.tailwindcss.com](https://cdn.tailwindcss.com)"></script>
</head>
<body class="bg-slate-100 min-h-screen p-4 md:p-8">

  <!-- 🔒 密碼登入畫面 -->
  <div id="loginScreen" class="max-w-md mx-auto mt-20 bg-white p-8 rounded-xl shadow-md space-y-5">
    <div class="text-center">
      <h1 class="text-2xl font-bold text-slate-800">🔒 請輸入通行密碼</h1>
      <p class="text-sm text-slate-500 mt-1">跨裝置檔案與文字中繼站</p>
    </div>
    <input type="password" id="pwdInput" class="w-full border rounded-lg p-3 text-center text-lg tracking-widest focus:outline-none focus:ring-2 focus:ring-blue-500" placeholder="••••" onkeydown="if(event.key==='Enter') login()"/>
    <p id="loginError" class="text-xs text-red-500 text-center hidden">密碼錯誤，請重新輸入！</p>
    <button onclick="login()" id="loginBtn" class="w-full py-3 bg-slate-800 text-white font-medium rounded-lg hover:bg-slate-700">解鎖進入</button>
  </div>

  <!-- ⚡ 主畫面 -->
  <div id="mainApp" class="max-w-3xl mx-auto space-y-6 hidden">
    <div class="bg-white p-6 rounded-xl shadow-sm flex flex-col sm:flex-row justify-between items-start sm:items-center gap-4">
      <div>
        <h1 class="text-2xl font-bold text-slate-800">⚡ 跨裝置檔案 / 文字中繼站</h1>
        <p class="text-sm text-slate-500">檔案直傳 Google Drive・48小時自動銷毀</p>
      </div>
      <div class="flex flex-wrap gap-2 w-full sm:w-auto justify-end">
        <button onclick="loadData()" id="refreshBtn" class="px-3.5 py-2 bg-slate-800 text-white rounded-lg text-sm hover:bg-slate-700">重新整理</button>
        <button onclick="clearAll()" id="clearAllBtn" class="px-3.5 py-2 bg-rose-600 text-white rounded-lg text-sm hover:bg-rose-700">🗑️ 立即清空全部</button>
        <button onclick="logout()" class="px-3 py-2 bg-slate-100 text-slate-600 border rounded-lg text-sm hover:bg-slate-200">鎖定</button>
      </div>
    </div>

    <!-- 文字互傳區 -->
    <div class="bg-white p-6 rounded-xl shadow-sm space-y-4">
      <h2 class="font-semibold text-slate-700">📝 快速傳送文字</h2>
      <textarea id="textInput" rows="3" class="w-full border rounded-lg p-3 text-sm focus:outline-none focus:ring-2 focus:ring-blue-500" placeholder="在此貼上網址、驗證碼或筆記..."></textarea>
      <button onclick="sendText()" id="sendTextBtn" class="w-full py-2.5 bg-blue-600 text-white font-medium rounded-lg hover:bg-blue-700">傳送文字</button>
      <div id="textList" class="space-y-2 pt-2"></div>
    </div>

    <!-- 檔案上傳區 -->
    <div class="bg-white p-6 rounded-xl shadow-sm space-y-4">
      <h2 class="font-semibold text-slate-700">📁 傳送檔案 (不限大小)</h2>
      <input type="file" id="fileInput" class="block w-full text-sm text-slate-500 file:mr-4 file:py-2 file:px-4 file:rounded-lg file:border-0 file:text-sm file:font-semibold file:bg-blue-50 file:text-blue-700 hover:file:bg-blue-100"/>
      
      <div id="progressContainer" class="hidden space-y-1">
        <div class="w-full bg-slate-200 rounded-full h-2.5">
          <div id="progressBar" class="bg-blue-600 h-2.5 rounded-full" style="width: 0%"></div>
        </div>
        <p id="progressText" class="text-xs text-slate-500 text-right">0%</p>
      </div>

      <button onclick="uploadFile()" id="uploadBtn" class="w-full py-2.5 bg-emerald-600 text-white font-medium rounded-lg hover:bg-emerald-700">開始上傳</button>
      <div id="fileList" class="space-y-2 pt-2"></div>
    </div>
  </div>

  <script>
    // 👇 請替換為您的 GAS 網頁應用程式網址 (/exec 結尾)
    const GAS_URL = "[https://script.google.com/macros/s/請替換為您的GAS網址代碼/exec](https://script.google.com/macros/s/請替換為您的GAS網址代碼/exec)";
    
    // 使用 sessionStorage：同一分頁整理時保留，關閉分頁即自動清除密碼
    let currentPwd = sessionStorage.getItem('dropzone_pwd') || '';

    async function callGasApi(payload) {
      const res = await fetch(GAS_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'text/plain;charset=utf-8' },
        body: JSON.stringify(payload)
      });
      
      const text = await res.text();
      let json;
      try {
        json = JSON.parse(text);
      } catch (e) {
        throw new Error("後端未正確回應 JSON，請確認 GAS 已建立新版本部署");
      }

      if (json.status !== 'ok' || !json.data) {
        throw new Error(json.message || "驗證失敗");
      }
      return json.data;
    }

    window.onload = () => {
      localStorage.removeItem('dropzone_pwd'); // 清除舊版殘留密碼
      if (currentPwd && currentPwd.trim() !== '') {
        document.getElementById('loginBtn').innerText = '自動登入驗證中...';
        document.getElementById('loginBtn').disabled = true;
        verifyAndLoad(currentPwd);
      }
    };

    function login() {
      const pwd = document.getElementById('pwdInput').value.trim();
      if (!pwd) return;
      document.getElementById('loginBtn').innerText = '驗證中...';
      document.getElementById('loginBtn').disabled = true;
      document.getElementById('loginError').classList.add('hidden');
      verifyAndLoad(pwd);
    }

    async function verifyAndLoad(pwd) {
      try {
        if (!pwd) throw new Error("密碼不可為空");
        const data = await callGasApi({ action: 'getItems', pwd: pwd });
        
        if (!data || !Array.isArray(data.files)) {
          throw new Error("資料驗證異常");
        }

        currentPwd = pwd;
        sessionStorage.setItem('dropzone_pwd', pwd);
        
        document.getElementById('loginScreen').classList.add('hidden');
        document.getElementById('mainApp').classList.remove('hidden');
        document.getElementById('loginBtn').innerText = '解鎖進入';
        document.getElementById('loginBtn').disabled = false;
        renderData(data);
      } catch (err) {
        sessionStorage.removeItem('dropzone_pwd');
        currentPwd = '';
        document.getElementById('loginScreen').classList.remove('hidden');
        document.getElementById('mainApp').classList.add('hidden');
        document.getElementById('loginBtn').innerText = '解鎖進入';
        document.getElementById('loginBtn').disabled = false;
        document.getElementById('loginError').classList.remove('hidden');
      }
    }

    function logout() {
      sessionStorage.removeItem('dropzone_pwd');
      localStorage.removeItem('dropzone_pwd');
      currentPwd = '';
      document.getElementById('pwdInput').value = '';
      document.getElementById('mainApp').classList.add('hidden');
      document.getElementById('loginScreen').classList.remove('hidden');
    }

    async function loadData() {
      const btn = document.getElementById('refreshBtn');
      btn.innerText = '載入中...';
      const data = await callGasApi({ action: 'getItems', pwd: currentPwd });
      renderData(data);
      btn.innerText = '重新整理';
    }

    async function clearAll() {
      if (!confirm('確定要立即清除雲端中繼站上的「所有檔案與文字紀錄」嗎？')) return;
      const btn = document.getElementById('clearAllBtn');
      btn.innerText = '清除中...';
      btn.disabled = true;
      try {
        const data = await callGasApi({ action: 'clearAllItems', pwd: currentPwd });
        renderData(data);
      } catch (err) {
        alert('清除失敗：' + err.message);
      }
      btn.innerText = '🗑️ 立即清空全部';
      btn.disabled = false;
    }

    async function deleteFile(fileId) {
      if (!confirm('確定要刪除此檔案嗎？')) return;
      const data = await callGasApi({ action: 'deleteFileItem', fileId: fileId, pwd: currentPwd });
      renderData(data);
    }

    async function deleteText(textId) {
      const data = await callGasApi({ action: 'deleteTextItem', textId: textId, pwd: currentPwd });
      renderData(data);
    }

    function renderData(data) {
      const textContainer = document.getElementById('textList');
      if (data.texts.length === 0) {
        textContainer.innerHTML = `<p class="text-xs text-slate-400 text-center py-2">目前無文字紀錄</p>`;
      } else {
        textContainer.innerHTML = data.texts.map(item => `
          <div class="p-3 bg-slate-50 rounded-lg border flex justify-between items-start gap-2">
            <pre class="text-sm text-slate-800 whitespace-pre-wrap break-all font-sans">${escapeHtml(item.text)}</pre>
            <div class="flex gap-1.5 shrink-0">
              <button onclick="navigator.clipboard.writeText(\`${escapeHtml(item.text)}\`)" class="text-xs px-2.5 py-1 bg-white border rounded hover:bg-slate-100">複製</button>
              <button onclick="deleteText('${item.id}')" class="text-xs px-2 py-1 bg-white border border-rose-200 text-rose-600 rounded hover:bg-rose-50">刪除</button>
            </div>
          </div>
        `).join('');
      }

      const fileContainer = document.getElementById('fileList');
      if (data.files.length === 0) {
        fileContainer.innerHTML = `<p class="text-xs text-slate-400 text-center py-2">目前無暫存檔案</p>`;
      } else {
        fileContainer.innerHTML = data.files.map(f => `
          <div class="p-3 bg-slate-50 rounded-lg border flex justify-between items-center gap-2">
            <div class="truncate pr-2">
              <p class="text-sm font-medium text-slate-800 truncate">${escapeHtml(f.name)}</p>
              <p class="text-xs text-slate-400">${(f.size / 1024 / 1024).toFixed(2)} MB</p>
            </div>
            <div class="flex gap-1.5 shrink-0">
              <a href="${f.url}" target="_blank" class="text-xs px-3 py-1.5 bg-blue-600 text-white rounded hover:bg-blue-700">下載 / 開啟</a>
              <button onclick="deleteFile('${f.id}')" class="text-xs px-2.5 py-1.5 bg-white border border-rose-200 text-rose-600 rounded hover:bg-rose-50">刪除</button>
            </div>
          </div>
        `).join('');
      }
    }

    async function sendText() {
      const input = document.getElementById('textInput');
      const text = input.value.trim();
      if (!text) return;
      document.getElementById('sendTextBtn').innerText = '傳送中...';
      await callGasApi({ action: 'saveTextItem', text: text, pwd: currentPwd });
      input.value = '';
      document.getElementById('sendTextBtn').innerText = '傳送文字';
      loadData();
    }

    async function uploadFile() {
      const file = document.getElementById('fileInput').files[0];
      if (!file) return alert('請選擇檔案');

      document.getElementById('uploadBtn').disabled = true;
      document.getElementById('progressContainer').classList.remove('hidden');

      const config = await callGasApi({ action: 'getUploadConfig', pwd: currentPwd });
      const metadata = {
        name: file.name,
        mimeType: file.type || 'application/octet-stream',
        parents: [config.folderId]
      };

      const res = await fetch('[https://www.googleapis.com/upload/drive/v3/files?uploadType=resumable](https://www.googleapis.com/upload/drive/v3/files?uploadType=resumable)', {
        method: 'POST',
        headers: {
          'Authorization': 'Bearer ' + config.token,
          'Content-Type': 'application/json; charset=UTF-8',
          'X-Upload-Content-Type': file.type || 'application/octet-stream',
          'X-Upload-Content-Length': file.size
        },
        body: JSON.stringify(metadata)
      });

      const uploadUrl = res.headers.get('Location');
      const xhr = new XMLHttpRequest();
      xhr.open('PUT', uploadUrl, true);
      xhr.upload.onprogress = (e) => {
        if (e.lengthComputable) {
          const percent = Math.round((e.loaded / e.total) * 100);
          document.getElementById('progressBar').style.width = percent + '%';
          document.getElementById('progressText').innerText = percent + '%';
        }
      };
      xhr.onload = () => {
        document.getElementById('uploadBtn').disabled = false;
        document.getElementById('progressContainer').classList.add('hidden');
        document.getElementById('fileInput').value = '';
        loadData();
      };
      xhr.send(file);
    }

    function escapeHtml(str) {
      return str.replace(/[&<>'"]/g, tag => ({ '&': '&amp;', '<': '&lt;', '>': '&gt;', "'": '&#39;', '"': '&quot;' }[tag]));
    }
  </script>
</body>
</html>
```
