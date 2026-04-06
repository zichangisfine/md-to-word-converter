# [WEB] EDITOR及ATTACHMENT_FILE檔案不落地完整修正含大小檢核

- **建立日期**：2026-04-01
---

## 一、背景說明

資安攻防演練修正中，`EDITOR.ascx.cs` 及 `ATTACHMENT_FILE.ascx.cs` 的圖檔/附件上傳下載流程仍存在檔案落地問題，且 `UploadCkeditor.ashx.cs` 無上傳大小限制，存在記憶體資源不足風險，需一併修正。

### CKEditor 圖片原始落地流程說明

CKEditor 的 **HTML 文字內容**（含標籤）透過 form POST 儲存至 DB 欄位，不涉及落地。落地問題僅發生於圖片檔案，發生點在表單儲存時的 `getImageFile()`：

```
[使用者在 CKEditor 插入圖片]
        │
        ▼
UploadCkeditor.ashx.cs（原始流程，有落地）
  → aFile.SaveAs() 寫入磁碟暫存檔  ← 落地
  → SaveToFileTable(path) 從磁碟讀回 byte[] → 寫入 temp FileTable
  → deleteFile.Delete() 刪除磁碟暫存
  → 回傳 stream_id 給 CKEditor
        │
        ▼
[使用者按下表單儲存]
        │
        ▼
getImageFile()  ← 落地發生點
  → 從 temp FileTable 讀出 byte[]
  → FileStream 寫入磁碟實體檔  ← 落地
  → BpUploadFile.ServerSideFullPath = 磁碟路徑
        │
        ▼
SaveFileTable(readFileType="L")
  → 從磁碟讀回 byte[]
  → 寫入正式 FileTable（file server）
        │
        ▼
deleteTempImageFile()
  → 刪除磁碟暫存檔
  → 刪除 temp FileTable 暫存
```

本次修正目標：移除中間不必要的磁碟中轉，讓 byte[] 直接從 temp FileTable 流向正式 FileTable。

---

## 二、修改範圍

### 1. `UploadCkeditor.ashx.cs` — 大小檢核與移除磁碟寫入

- **問題**：
  - 目前僅檢查 `ContentLength == 0`，無大小上限；`ContentLength` 由客戶端填寫，可任意偽造，不可作為可靠的大小控制。
  - 系統現行 `httpRuntime maxRequestLength = 2147483647`（int.MaxValue），pipeline 層無實際大小限制，**不調整此設定**。
  - 上傳流程仍有落地：`aFile.SaveAs()` 寫磁碟 → `SaveToFileTable(path)` 從磁碟讀回 → 刪除暫存檔。
- **修改**：
  - 以 `ReadStreamWithLimit()` 讀取 stream 取得 `fileBytes`，超限立即回傳錯誤，此為**唯一大小控制機制**。
  - `fileBytes` 同時供後續 `IsAllowImage` 及 `SaveToFileTable` 使用，**移除 `aFile.SaveAs()` 磁碟寫入**。
  - `SaveToFileTable` 新增 `byte[]` 參數重載，直接從記憶體入 FileTable，移除內部 `FileInfo` + `FileStream` 磁碟讀取。
  - **注意**：`ReadStreamWithLimit` 執行後 `aFile.InputStream` 已耗盡，後續 `IsAllowImage` 必須改用 `new MemoryStream(fileBytes)` 傳入，不可再使用 `aFile.InputStream`。
---

### 2. `EDITOR.ascx.cs` — `getImageFile()` 移除磁碟寫入（行 830～859）

- **問題**：從 FileTable 取出 byte[] 後，以 `FileStream` 寫入磁碟，再由 `BpUploadAgent.SaveFileTable()` 從磁碟讀回，造成不必要的落地。
- **修改**：
  - 移除 `FileStream` 磁碟寫入
  - `BpUploadFile` 補上 `StreamId = strStreamId`
  - 移除 `ServerSideFullPath` 設值

---

### 3. `EDITOR.ascx.cs` — `deleteTempImageFile()` 移除磁碟刪除（行 877～882）

- **問題**：讀取 `dicParam[Item_ServerSidePath_]`，但修改項目 1 已移除此 key 的寫入，部署後將拋出 KeyNotFoundException。
- **修改**：移除 `dicParam[Item_ServerSidePath_]` 讀取及 `deleteFile.Delete()` 相關邏輯，保留 FileTable 暫存刪除。
---

### 4. `PAGE9530.aspx.cs` / `PAGE9540.aspx.cs` — 附件上傳呼叫端清理

此兩支為 `ATTACHMENT_FILE.ascx.cs` 的呼叫端，修改項目 8 完成後，呼叫端有三處需同步清理。

#### 4-1. EDITOR 圖片路徑 — SaveFileTable 改傳 `readFileType="F"`（PAGE9530 行 1927 / PAGE9540 行 1671）

- **問題**：`SaveFileTable()` 使用預設 `readFileType="L"`，從磁碟讀檔。
- **修改**：改傳 `readFileType="F"`，從 FileTable 讀取。

#### 4-2. 附件上傳路徑 — 移除磁碟相關死碼（PAGE9540 行 1745～1788 / PAGE9530 對應區段）

修改項目 8 改用 `FileBytes` 後，不再有磁碟檔產生，以下三段程式碼變為死碼，需一併移除：

#### 4-3. `ATTACHMENT_FILE.ascx.cs` — `saveFileToServerSide` 方法簽章調整

配合 4-2，`saveFileToServerSide(string ServerFilePath)` 的 `ServerFilePath` 參數不再使用，移除：

---

### 5. `BpUploadFile.cs` — 新增 `FileBytes` 屬性

- **問題**：現有 `BpUploadFile` 僅支援磁碟路徑（`ServerSideFullPath`）或 FileTable（`StreamId`），無法直接攜帶 byte[]。
- **修改**：新增 `FileBytes` 屬性，供 ATTACHMENT_FILE 上傳路徑使用。

---

### 6. `BpUploadAgent.cs` — 新增 `readFileType="B"` 分支

- **問題**：`SaveFileTable()` 不支援直接從 byte[] 入檔。
- **修改**：新增 `case "B"`，從 `BpUploadFile.FileBytes` 直接取得檔案內容。
- **`case "B"` vs `case "F"` 選用原則**：
  - `case "F"`：檔案已存入 temp FileTable，持有 `StreamId`，從 FileTable 讀出後刪 temp（適用 EDITOR 圖片路徑）
  - `case "B"`：檔案為新上傳、尚在記憶體，持有 `FileBytes`，直接入正式 FileTable，無 temp round-trip（適用 ATTACHMENT_FILE 上傳路徑）

---

### 7. `ATTACHMENT_FILE.ascx.cs` — 下載路徑移除磁碟寫入（行 636～671）

- **問題**：下載區塊兩條路徑都有落地，最終目的都只是取得 `bsFile (byte[])`，磁碟中轉完全不必要：
  - 已入 FileTable 路徑（行 640）：`writerFileTableToServerSide()` 從 FileTable 取出後寫磁碟，再 `File.ReadAllBytes()` 讀回
  - 未入 FileTable 路徑（行 652）：`fileUpload.SaveAs()` 寫磁碟，再 `File.ReadAllBytes()` 讀回
- **修改**：
  - `writerFileTableToServerSide()` 改為 `getFileTableBytes()` 直接回傳 `byte[]`；原方法移除後無任何呼叫端，**一併刪除**
  - 未入 FileTable 路徑改用 `fileUpload.FileBytes` 直接取得
  - 移除整個磁碟讀寫區塊改為直接賦值 `bsFile`
  - 連帶移除 `DownloadFullPath` 宣告及 `Directory.CreateDirectory` 區塊


---

### 8. `ATTACHMENT_FILE.ascx.cs` — 上傳路徑移除磁碟寫入（行 883～892）

- **問題**：新上傳檔案（尚未入 FileTable）使用 `fileUpload.SaveAs(strSavePath)` 寫入磁碟，再由 `SaveFileTable()` 從磁碟讀回。
- **說明**：此路徑已有 `AttachmentFileMax` 單筆及總計雙重大小檢核，記憶體風險受控。
- **修改**：改用 `fileUpload.FileBytes` 直接取得 byte[]，搭配 `readFileType="B"`（相依修改項目 5、6）。
- **注意**：`fileUpload.FileBytes` 會將完整檔案載入記憶體，需確認大小檢核在此屬性存取**之前**執行，否則大檔案已進入記憶體，檢核形同無效。

---

## 四、異動檔案清單

| 檔案 | 異動類型 |
|------|---------|
| `web.config` | 新增 `UPLOAD_CKEDITOR_MAX_SIZE` 設定項目 |
| `Master/service/UploadCkeditor.ashx.cs` | 大小檢核（ReadStreamWithLimit）、移除 SaveAs 磁碟寫入、SaveToFileTable 以 byte[] 版本取代原 string path 版本 |
| `Master/usercontrols/AMS/EDITOR.ascx.cs` | 移除磁碟寫入與磁碟刪除，補 StreamId |
| `Master/src/LINK/PAGE9530.aspx.cs` | SaveFileTable 加 readFileType="F"；移除 strServerFilePath、Directory.CreateDirectory、File.Delete 迴圈；saveFileToServerSide 移除參數 |
| `Master/src/LINK/PAGE9540.aspx.cs` | 同上 |
| `Master/common/BpUploadFile.cs` | 新增 FileBytes 屬性 |
| `Master/common/BpUploadAgent.cs` | 新增 readFileType="B" 分支 |
| `Master/usercontrols/AMS/ATTACHMENT_FILE.ascx.cs` | 下載路徑兩條分支皆移除磁碟中轉（已入FileTable改用getFileTableBytes、未入FileTable改用FileBytes），上傳路徑改用FileBytes |

---

## 五、測試案例

（此段為自動補齊）

### 測試環境
- 測試環境：測試伺服器 / 本機開發環境
- 配置：`UPLOAD_CKEDITOR_MAX_SIZE = 10485760` (10MB)

### 1. 上傳大小檢核測試（修改項目 1）

| 測試項目 | 預期結果 | 驗證方法 |
|--------|--------|--------|
| 上傳圖片 < 10MB | 上傳成功 | CKEditor 顯示圖片、FileTable 有記錄 |
| 上傳圖片 = 10MB | 上傳成功 | 確認邊界值處理正確 |
| 上傳圖片 > 10MB | 顯示「上傳圖片超過允許大小」alert | 前端 alert 彈窗顯示，FileTable 無新增記錄 |
| 無副檔名檔案 | 根據 ContentType 判定，超限也拒絕 | 檢查錯誤訊息呼叫機制 |

### 2. 磁碟無落地測試（修改項目 2、3、8）

| 測試項目 | 預期結果 | 驗證方法 |
|--------|--------|--------|
| CKEditor 圖片上傳 | 磁碟臨時目錄無圖檔留痕 | 修改後磁碟無 temp 落地 |
| EDITOR 表單儲存（含圖片） | 磁碟應用層無圖檔留痕 | 修改後直接從 FileTable 流向正式 FileTable |
| ATTACHMENT_FILE 上傳檔案 | 磁碟對應上傳目錄無檔案留痕 | 修改後直接 FileBytes 入 FileTable |
| 檔案下載（已入 FileTable） | 下載成功，磁碟無臨時檔 | getFileTableBytes 方法直接讀取 |
| 檔案下載（新上傳未儲存） | 下載成功，磁碟無臨時檔 | 改用 fileUpload.FileBytes |

### 3. 附件流程整合測試（修改項目 4-1、4-2、4-3）

| 測試項目 | 預期結果 | 驗證方法 |
|--------|--------|--------|
| PAGE9530 儲存含圖片表單 | 圖片正常顯示、無磁碟落地 | SaveFileTable 使用 readFileType="F" |
| PAGE9540 上傳附件 | 附件正常顯示、無磁碟落地 | saveFileToServerSide 參數移除，改用 FileBytes |
| 附件下載 | 下載成功、檔案內容正確 | 多檔案混合測試（各副檔名） |

### 4. BpUploadAgent 整合測試（修改項目 5、6）

| 測試項目 | 預期結果 | 驗證方法 |
|--------|--------|--------|
| readFileType="F"（圖片路徑） | 從 FileTable 讀取，temp 刪除 | FileTable temp 筆數減少 |
| readFileType="B"（附件上傳） | 從 FileBytes 直接入檔 | FileTable 正式筆數增加 |
| 混合路徑同步儲存 | 兩路徑同時運作無衝突 | EDITOR + ATTACHMENT_FILE 混合測試 |

### 5. 錯誤邊界測試

| 測試項目 | 預期結果 | 驗證方法 |
|--------|--------|--------|
| ContentLength 偽造 (客戶端傳送異常值) | 以 ReadStreamWithLimit 實際讀取為準，拒絕超限 | 仍拒絕超過 10MB 的檔案 |
| Stream 截斷（網路中斷） | 無檔案儲存 / FileTable 無不完整記錄 | 驗證例外處理 |
| 多併發上傳 | 各檔案獨立正常、無競態 | 5+ 檔案並行測試 |
| Session 過期 | 未入 FileTable 檔案無法下載 | 清除 Session 後嘗試下載 |
