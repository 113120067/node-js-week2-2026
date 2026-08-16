# Node.js 第二週作業講義

這份講義是給新手看的版本，目標是把整份作業拆開講清楚：這個專案在做什麼、每個檔案負責什麼、程式怎麼運作、以及怎麼驗證結果。

---

## 一、作業主題

這份作業的主題是「教練大頭照上傳 API」。

你要做的是：

- 建立一個 Node.js HTTP server
- 接收 `multipart/form-data` 的圖片上傳
- 使用 `formidable` 解析上傳檔案
- 讀出檔案資訊，例如檔名、大小、副檔名
- 把檔案存到指定資料夾
- 回傳 JSON 結果給前端或測試程式
- 用 Jest 驗證每個功能是否正確

---

## 二、專案結構

這個專案主要有這幾個檔案：

- `index.js`：主要功能都寫在這裡
- `app.js`：啟動伺服器
- `test.js`：測試檔
- `README.md`：原始作業說明
- `fixtures/`：測試用檔案資料夾

---

## 三、安裝與啟動

### 安裝套件

先執行：

```bash
npm install
```

### 啟動 server

```bash
npm start
```

預設會啟動在 `http://localhost:3000`。

### 執行測試

```bash
npm test
```

---

## 四、完整程式

### 1. `index.js`

```js
const http = require('node:http');
const fs = require('node:fs');
const { formidable } = require('formidable');  // formidable v3 用 named import

// ========== 任務一：讀取上傳設定 ==========
/**
 * 從 process.env 讀取上傳相關設定，回傳設定物件。
 *
 * 規則：
 *   - UPLOAD_DIR 未設定 → 預設 '/tmp'
 *   - MAX_FILE_SIZE_MB 未設定 → 預設 5（MB）
 *   - GYM_NAME 未設定 → 預設 '未命名健身房'
 *
 * 回傳物件：
 *   - uploadDir: 上傳目錄（字串）
 *   - maxFileSize: 最大檔案大小（bytes，= MB * 1024 * 1024）
 *   - gymName: 健身房名稱（字串）
 *
 * @returns {{uploadDir: string, maxFileSize: number, gymName: string}}
 *
 * @example
 *   process.env.UPLOAD_DIR = '/tmp/uploads';
 *   process.env.MAX_FILE_SIZE_MB = '10';
 *   process.env.GYM_NAME = 'FitClub';
 *   getUploadConfig();
 *   // { uploadDir: '/tmp/uploads', maxFileSize: 10485760, gymName: 'FitClub' }
 */
function getUploadConfig() {
  const uploadDir = process.env.UPLOAD_DIR || '/tmp';
  const maxFileSizeMB = Number(process.env.MAX_FILE_SIZE_MB || 5);
  const gymName = process.env.GYM_NAME || '未命名健身房';

  return {
    uploadDir,
    maxFileSize: maxFileSizeMB * 1024 * 1024,
    gymName,
  };
}

// ========== 任務二：取副檔名 ==========
/**
 * 從檔名取副檔名，一律回小寫帶 `.`。
 *
 * 規則：
 *   - 'cat.jpg' → '.jpg'
 *   - 'PHOTO.JPG' → '.jpg'（一律小寫）
 *   - 'README' → ''（沒有副檔名）
 *   - 'archive.tar.gz' → '.gz'（只取最後一個）
 *
 * @param {string} filename
 * @returns {string}
 *
 * @example
 *   getFileExtension('cat.jpg');     // '.jpg'
 *   getFileExtension('PHOTO.JPG');   // '.jpg'
 *   getFileExtension('README');      // ''
 */
function getFileExtension(filename) {
  const lastDotIndex = filename.lastIndexOf('.');

  if (lastDotIndex <= 0) {
    return '';
  }

  return filename.slice(lastDotIndex).toLowerCase();
}

// ========== 任務三：解析檔案 metadata ==========
/**
 * 吃 formidable 解析後的 file 物件，回傳整理好的 metadata。
 *
 * formidable 的 file 物件至少有：
 *   - originalFilename: 原始檔名
 *   - size: 檔案 byte 數
 *
 * 回傳：
 *   - filename: 原始檔名
 *   - sizeKB: 檔案大小換成 KB（四捨五入，用 Math.round）
 *   - ext: 副檔名（用任務二的 getFileExtension）
 *
 * @param {{originalFilename: string, size: number}} file
 * @returns {{filename: string, sizeKB: number, ext: string}}
 *
 * @example
 *   parseFileMetadata({ originalFilename: 'leo.jpg', size: 250000 });
 *   // { filename: 'leo.jpg', sizeKB: 244, ext: '.jpg' }
 */
function parseFileMetadata(file) {
  const filename = file.originalFilename || '';

  return {
    filename,
    sizeKB: Math.round(file.size / 1024),
    ext: getFileExtension(filename),
  };
}

// ========== 任務四：產出 upload log 字串 ==========
/**
 * 吃 metadata + config，產出一行 log 字串。
 *
 * 格式：`[{gymName}] Uploaded {filename} ({sizeKB} KB) → {uploadDir}`
 *
 * @param {{filename: string, sizeKB: number}} meta
 * @param {{uploadDir: string, gymName: string}} config
 * @returns {string}
 *
 * @example
 *   formatUploadLog(
 *     { filename: 'leo.jpg', sizeKB: 245, ext: '.jpg' },
 *     { uploadDir: '/tmp/uploads', gymName: 'FitClub' }
 *   );
 *   // '[FitClub] Uploaded leo.jpg (245 KB) → /tmp/uploads'
 */
function formatUploadLog(meta, config) {
  return `[${config.gymName}] Uploaded ${meta.filename} (${meta.sizeKB} KB) → ${config.uploadDir}`;
}

// ========== 任務五：路由分派 ==========
/**
 * 吃 HTTP request / response / config，依 method + url 分派到對應處理邏輯。
 *
 * 規格：
 *   - POST /coaches/avatar：
 *     * 用 formidable 解析 multipart/form-data
 *     * 成功 → 回 200 + JSON { filename, sizeKB, ext, savedPath }
 *     * formidable 解析錯誤（含超過 maxFileSize）→ 回 500 + JSON { error }
 *     * 沒 file 欄位 → 回 400 + JSON { error: 'No file uploaded' }
 *   - 其他路徑 → 回 404 + JSON { error: 'Not Found' }
 *
 * formidable 設定：
 *   - uploadDir / maxFileSize 從 config 取
 *   - keepExtensions: true
 *
 * @param {http.IncomingMessage} req
 * @param {http.ServerResponse} res
 * @param {{uploadDir: string, maxFileSize: number, gymName: string}} config
 * @returns {void} 直接操作 res 回寫、不 return 值
 *
 * @example
 *   // 在 createUploadServer 裡：
 *   http.createServer((req, res) => router(req, res, config))
 */
function router(req, res, config) {
  if (req.method === 'POST' && req.url === '/coaches/avatar') {
    const form = formidable({
      uploadDir: config.uploadDir,
      maxFileSize: config.maxFileSize,
      keepExtensions: true,
    });

    form.parse(req, (err, fields, files) => {
      if (err) {
        res.writeHead(500, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: err.message }));
        return;
      }

      const uploadedFile = files.file?.[0];

      if (!uploadedFile) {
        res.writeHead(400, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: 'No file uploaded' }));
        return;
      }

      const meta = parseFileMetadata(uploadedFile);
      const savedPath = uploadedFile.filepath;

      console.log(formatUploadLog(meta, config));

      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({
        ...meta,
        savedPath,
      }));
    });

    return;
  }

  res.writeHead(404, { 'Content-Type': 'application/json' });
  res.end(JSON.stringify({ error: 'Not Found' }));
}

// ========== 任務六：建立上傳 server ==========
/**
 * 建 http.Server、把每個 request 交給 router。
 *
 * 規格：
 *   - 如果 config.uploadDir 不存在，用 fs.mkdirSync(uploadDir, { recursive: true }) 自動建
 *   - http.createServer(...) 把 request 交給 router(req, res, config)
 *   - 回傳 server instance（不要 server.listen()，那是 app.js 的責任）
 *
 * @param {{uploadDir: string, maxFileSize: number}} config
 * @returns {http.Server}
 *
 * @example
 *   const server = createUploadServer({ uploadDir: '/tmp', maxFileSize: 5 * 1024 * 1024 });
 *   server.listen(3000);  // ← 這行由 app.js 呼叫
 */
function createUploadServer(config) {
  fs.mkdirSync(config.uploadDir, { recursive: true });

  return http.createServer((req, res) => router(req, res, config));
}

module.exports = {
  getUploadConfig,
  getFileExtension,
  parseFileMetadata,
  formatUploadLog,
  router,
  createUploadServer,
};
```

### 2. `app.js`

```js
require('dotenv').config();

const {
  getUploadConfig,
  createUploadServer,
} = require('./index');

const config = getUploadConfig();
const PORT = Number(process.env.PORT) || 3000;

console.log('========================================');
console.log(`🏋️  ${config.gymName} 大頭照上傳 API`);
console.log(`📁 上傳目錄：${config.uploadDir}`);
console.log(`📏 最大檔案：${config.maxFileSize / 1024 / 1024} MB`);
console.log('========================================');

const server = createUploadServer(config);

server.listen(PORT, () => {
  console.log(`✅ Server listening on http://localhost:${PORT}`);
  console.log('');
  console.log('測試方式：');
  console.log(`  curl -F "file=@fixtures/sample.jpg" http://localhost:${PORT}/coaches/avatar`);
  console.log('');
});
```

---

## 五、每個函式在做什麼

### 1. `getUploadConfig()`

這個函式負責從環境變數讀資料。

它會讀：

- `UPLOAD_DIR`
- `MAX_FILE_SIZE_MB`
- `GYM_NAME`

如果沒有設定，就用預設值。

### 2. `getFileExtension(filename)`

這個函式負責從檔名取副檔名。

例如：

- `cat.jpg` 會回傳 `.jpg`
- `PHOTO.JPG` 會回傳 `.jpg`
- `README` 會回傳空字串

重點是：只取最後一個點後面的部分，而且轉成小寫。

### 3. `parseFileMetadata(file)`

這個函式接收 `formidable` 解析後的檔案物件。

它會整理出三個欄位：

- `filename`：原始檔名
- `sizeKB`：檔案大小換成 KB
- `ext`：副檔名

### 4. `formatUploadLog(meta, config)`

這個函式只是把資訊組成一行 log。

格式像這樣：

```text
[TestGym] Uploaded sample.jpg (2 KB) → C:\Temp
```

### 5. `router(req, res, config)`

這是整個 server 的路由核心。

它會判斷：

- 如果是 `POST /coaches/avatar`，就進行檔案上傳處理
- 如果不是，就回 `404 Not Found`

上傳成功時會回傳 JSON：

```json
{
  "filename": "sample.jpg",
  "sizeKB": 2,
  "ext": ".jpg",
  "savedPath": "C:\\..."
}
```

如果沒有檔案，就回 `400`。

如果檔案太大或解析失敗，就回 `500`。

### 6. `createUploadServer(config)`

這個函式負責建立 HTTP server。

它會先確認上傳資料夾存在，不存在就建立。

然後回傳 `http.createServer(...)`，讓 `app.js` 自己去 `listen()`。

---

## 六、程式流程圖

```mermaid
flowchart TD
  A[app.js 啟動] --> B[讀取環境變數 getUploadConfig]
  B --> C[建立 server createUploadServer]
  C --> D[收到 request]
  D --> E{是不是 POST /coaches/avatar?}
  E -- 是 --> F[formidable 解析上傳檔案]
  F --> G{有沒有錯誤?}
  G -- 有 --> H[回 500]
  G -- 沒有 --> I{有沒有 file?}
  I -- 沒有 --> J[回 400]
  I -- 有 --> K[整理 metadata]
  K --> L[回 200 JSON]
  E -- 不是 --> M[回 404]
```

---

## 七、測試內容怎麼看

`test.js` 主要在驗證這幾件事：

- 環境變數有沒有正確讀到
- 副檔名有沒有抓對
- 檔案 metadata 有沒有整理正確
- log 字串有沒有組對
- server 路由與上傳功能有沒有符合規格

最後如果看到：

```text
Tests: 14 passed, 14 total
```

就代表全部通過。

---

## 八、常見錯誤

### 1. `files.file[0]` 找不到

因為 `formidable` v3 在 multipart 上傳時，常會把同名欄位包成陣列。

所以你要取：

```js
files.file[0]
```

### 2. `curl` 上傳失敗

確認欄位名稱是不是 `file`。

正確範例：

```bash
curl -F "file=@fixtures/sample.jpg" http://localhost:3000/coaches/avatar
```

### 3. 測試 timeout

如果 `Jest` 卡住，通常是某個 response 沒有 `res.end()`。

只要每個分支都有正確結束 response，通常就能避免這個問題。

---

## 九、結論

這份作業的核心概念其實很單純：

1. 先讀設定
2. 建立 HTTP server
3. 收到上傳請求後交給 `formidable`
4. 整理檔案資訊
5. 回傳 JSON 給呼叫端
6. 用測試確認每個功能都正確

如果你把這份講義理解清楚，這類 Node.js 上傳 API 題目就能自己寫出來了。
