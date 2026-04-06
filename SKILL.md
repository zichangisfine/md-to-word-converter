---
name: md-to-word
description: 將 Markdown 文件轉換為 Word (.docx) 文件，使用既定腳本 md_to_word.py
---

# md to Word

你是一個文件轉換工具，負責將 Markdown 文件轉換為 Word 文件。  
⚠️ 不負責自行生成 Word 結構，僅負責準備資料並呼叫既定腳本。
輸入：md 文件  
輸出：Word 文件（.docx），符合使用者指定的欄位結構。

---

## 嚴格限制（非常重要）

- 不得自行產生 Word 文件內容
- 不得使用任何未指定的工具或套件
- 不得修改 md_to_word.py
- 僅允許透過 md_to_word.py 進行轉換
- 嚴格遵守指定的輸入和輸出路徑
---

## 檔案配置
| 項目 | 路徑 |
| --- | --- |
| 輸入路徑 | ./md-to-word/sample.md |
| 輸出路徑 | ./md-to-word/sample.docx |
| 參考程式 | ./md-to-word/md_to_word.py |

---

## 請依照以下流程執行（Pipeline）

### Step 1：檢查 Markdown 結構
- 從輸入路徑讀取 md 文件
- 確認內容包含以下章節（若缺少則補齊）：
  - 專案說明（背景說明）
  - 流程說明
  - 程式調整清單（含功能說明）
  - 測試案例（需對應流程）
- 若為補齊內容，需：
  - 使用 Markdown 格式補上
  - 在段落標註「（此段為自動補齊）」  

### Step 2：執行轉換程式

Run pwsh command:
python ./md-to-word/md_to_word.py

---

### Step 3：驗證輸出
- 確認產生檔案：
  `./md-to-word/sample.docx`
- 若未產生，回報錯誤
- 不得自行用其他方式產生 Word

---

### Step 4：解除檔案鎖定
- 確保 sample.docx 可被開啟與編輯
- 不鎖定 sample.md 與 md_to_word.py
