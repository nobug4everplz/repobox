# iOS 捷徑：收進 RepoBox

iOS 的分享選單不支援網頁的 `share_target`，所以 iPhone 改用「捷徑」App。這個捷徑做的事和收藏網頁送出時一樣：找出 GitHub repo 網址，用 Contents API 在收藏庫寫一個 `inbox/<id>.json`。

> 這份步驟依 `web/app.js` 的送出格式與 `action/scripts/items.py` 的檢查規則寫成，尚未在實機上驗證。動作名稱以繁體中文介面為主，括號內是英文介面的名稱；在捷徑 App 搜尋任一個都找得到。

## 事前準備

- 收藏庫已照 README 第 2 步建好（下文以 `<帳號>/repobox-library` 表示）。
- 一把 fine-grained token：只選 `repobox-library`，權限只開 **Contents: Read and write**。建議手機另外建一把，遺失手機時只要撤銷這一把。

## 建立捷徑

在「捷徑」App 按 **＋** 新增，命名為「收進 RepoBox」，依序加入下列動作。`[名稱]` 表示插入前面步驟的結果（點欄位後從變數列選）。

1. **捷徑設定**：按上方的 ⓘ，開啟「**在分享表單中顯示**」，接收類型只勾 **URL** 和 **文字**。「如果沒有輸入」選「**取得剪貼板**」，這樣從主畫面直接執行時，會讀剪貼簿裡複製好的連結。
2. **URL 編碼**（URL Encode）：輸入選「捷徑輸入」，模式改成 **解碼**。社群 App 常把網址包成 `l.threads.com/?u=https%3A%2F%2Fgithub.com...`，解碼後才找得到。
3. **比對文字**（Match Text）：比對步驟 2 的結果，樣式填：
   ```
   github\.com/([A-Za-z0-9][A-Za-z0-9-]*)/([A-Za-z0-9._-]+)
   ```
4. **如果**（If）：步驟 3 的「符合項目」**沒有任何值** →
   - **顯示通知**（Show Notification）：`沒找到 GitHub repo 網址`
   - **停止此捷徑**（Stop This Shortcut）
   - 「否則」留空，「結束如果」之後繼續。
5. **取得清單中的項目**（Get Item from List）：從「符合項目」取 **第一個項目**。
6. **取得符合項目的群組**（Get Group from Matched Text）：從步驟 5 取 **索引 1**，把結果重新命名為 `Owner`。
7. **取得符合項目的群組**：同樣從步驟 5 取 **索引 2**。
8. **取代文字**（Replace Text）：在步驟 7 的結果中，把 `\.git$` 取代為空白，展開選項開啟 **正規表示式**。結果重新命名為 `Repo`。
9. **格式化日期**（Format Date）：日期選「目前日期」，格式選「自訂」，填 `yyyyMMdd-HHmmss`，結果命名為 `時間ID`。
10. **格式化日期**：日期選「目前日期」，格式選 **ISO 8601**，並開啟「包含時間」，結果命名為 `收藏時間`。
11. **隨機數字**（Random Number）：最小 `1000`、最大 `9999`。
12. **文字**（Text），內容如下（引號、逗號照打）：
    ```
    {
      "v": 1,
      "url": "https://github.com/[Owner]/[Repo]",
      "captured_at": "[收藏時間]",
      "source": "ios",
      "page_url": "",
      "note": "",
      "client": "shortcut@1"
    }
    ```
13. **Base64 編碼**（Base64 Encode）：輸入是步驟 12 的文字，「換行」選 **無**。
14. **取得 URL 內容**（Get Contents of URL）：
    - URL：`https://api.github.com/repos/<帳號>/repobox-library/contents/inbox/[時間ID]-[隨機數字].json`
    - 方法：**PUT**
    - 標頭（Headers）三個：
      - `Authorization`：`Bearer <你的 token>`
      - `Accept`：`application/vnd.github+json`
      - `X-GitHub-Api-Version`：`2022-11-28`
    - 要求內文（Request Body）選 **JSON**，加兩個「文字」欄位：
      - `message`：`capture: [Owner]/[Repo]`
      - `content`：步驟 13 的「Base64 編碼結果」
15. **取得字典值**（Get Dictionary Value）：從步驟 14 的結果取鍵 `content` 的值。
16. **如果**：步驟 15 的結果 **有任何值** →
    - **顯示通知**：`已收：[Owner]/[Repo]`
    - 「否則」→ **顯示通知**：`沒收到：[取得 URL 內容]`（插入步驟 14 的結果，會顯示 GitHub 的錯誤訊息）

## 使用方式

- **Threads／X 貼文裡的連結**：分享整篇貼文只會拿到貼文網址，找不到 GitHub 連結。改成長按貼文裡的 GitHub 連結 → 分享 → 收進 RepoBox；或先「拷貝連結」，再從主畫面或小工具執行捷徑（會讀剪貼簿）。
- **Safari 或 GitHub App 開著的 repo 頁**：直接按分享 → 收進 RepoBox。
- 收到通知「已收」後，分析約需幾分鐘，完成後到收藏網頁的書庫查看。

## 注意事項

- token 寫在捷徑裡。**不要分享這個捷徑**，分享出去等於交出 token；要給別人用，先把 Authorization 欄位清空。
- 網址只取 `owner/repo`，`/tree/main`、`?tab=readme` 等後綴會自動去掉。比對到的若不是真的 repo（例如 `github.com/settings/tokens`），分析流程查不到它，會把那筆移到 `inbox/failed/`。
- 錯誤 `401`：token 過期或貼錯；`404`：repo 名稱寫錯，或 token 沒授權這個 repo；`422`：同一秒內重複送出，再執行一次即可。
