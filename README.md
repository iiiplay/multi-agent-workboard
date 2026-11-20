# Multi-Agent Workboard

一個人同時使用多個 AI 編碼 agent（Claude Code、多組 Codex、Gemini CLI）時的本機任務看板。把「誰在做什麼、用了多少、要交代誰做什麼」集中在一個網頁裡，零依賴、單一 Node 進程、任務即 Markdown 檔。

## 為什麼要這個

訂閱型的 AI CLI(Claude Code / Codex / Gemini CLI)各自獨立跑,沒有一個地方能同時看到:
- 每個 agent 今天/這週/累計用了多少 token(估算費用)
- 手上有哪些任務、指派給誰、進度到哪
- 一鍵把某個任務交給某個 agent 的非互動模式去做,並看它做得怎樣

這個專案就是解決這件事,純粹自用工具,不是要給別人用的產品。

## 快速開始

```bash
node tools/server.mjs        # 預設 http://localhost:4173,可傳 port 覆蓋
# 或
npm start
```

打開 `http://localhost:4173`,看板就能用了。改 `tools/*.mjs` 後要重啟 server 才會生效;改 `web/index.html` 重新整理頁面即可,不用重啟。

## 功能

- **用量卡片**:依 agent 解析各 CLI 本機的 log/session 檔,顯示今日/7日/累計 token 與「API 等值估算」費用(使用者是訂閱制,這數字純參考,不代表實際扣款)。
- **五欄看板**(backlog / todo / doing / review / done):可拖曳改狀態、下拉選單指派 agent、新增/刪除任務。
- **一鍵派工**:對已指派 agent 的任務按「▶ 派工」,會用該 CLI 的非互動模式 spawn 一個行程去讀 `AGENTS.md` 並完成這張任務,在 Windows 上也能正確啟動 `.cmd`/`.bat` 這類 CLI 執行檔。
- **派工狀態可見 + 完整 log viewer**:派工中的任務能看到「執行中/已完成/失敗/已停止」,點「📜 log」能看完整輸出,3 秒自動輪詢更新。
- **停止 / 刪除**:派工中可以按「■ 停止」真的砍掉整棵行程樹(Windows 上用 `taskkill /pid <pid> /T /F`);任意任務可以「🗑 刪除」,正在跑的話會先停止再刪。
- **重啟不丟狀態**:派工中的行程狀態會存進 `board/runs/state.json`,server 重啟後用 pid 存活檢查還原,不會顯示成「狀態未知」。
- **桌面通知**:任務變成 review、或派工失敗/被中斷時,右上角按鈕開啟後會跳桌面通知 + 提示音,不用一直盯著畫面。
- **檔案衝突防呆**:任務可以選填 `files`(這張任務會動到的檔案/目錄),如果要派工的任務跟目前正在執行中的另一張任務宣告的檔案重疊,派工前會示警,避免兩個 agent 同時改到同一份檔案互相蓋掉;可以選擇強制派工。
- **每個任務的用量估算**:log viewer 裡會顯示「這次派工大概用了多少 token / 大概多少錢」,用該次派工的起訖時間去比對該 agent 的原始 log,是時間窗估算,不是精確逐筆歸因(已知限制,見下方)。

## 架構

零依賴(只用 Node 內建模組)、檔案即資料庫(任務是 Markdown,agent 跟網頁操作同一份檔案)、唯讀解析各 CLI 的 log(絕不修改它們的設定或 log)。詳細檔案分工、多 agent 協作規則、開發這個 repo 的注意事項都寫在 [AGENTS.md](AGENTS.md)——所有要在這個看板上工作的 agent(包括你自己手動操作)開工前都應該先讀那份。

```
agents.json        agent 清單、看板欄位定義
board/tasks/*.md    一件任務一個檔(frontmatter: title/status/agent/priority/files/updated)
board/runs/          派工的 log 與行程狀態側檔(已 gitignore)
tools/server.mjs    本機 HTTP server(REST API + 提供 web/)
tools/usage.mjs      各 CLI 用量解析
tools/dispatch.mjs   spawn CLI、追蹤行程、砍行程
web/index.html       單頁前端
apps/todoapp/        獨立的示範小專案(驗證派工機制用的 todo app,跟看板本體無關)
```

## 已知限制

- 用量與費用都是「API 等值估算」,不是真的計費依據。
- 每個任務的用量估算是「時間窗」比對,同一個 agent 若在派工期間也被人手動互動使用,那部分也會被算進去,無法區分來源。
- Codex 的 session 資料只有累計值沒有逐訊息時間戳,時間窗估算的顆粒度比 Claude/Gemini 粗。
- 檔案衝突防呆只比對「有宣告 `files` 且目前正在執行中」的任務,不是完整的靜態分析,也不會擋 `todo` 排隊中的任務。
- 純本機單人工具,沒有帳號權限控管,不要對外公開這個 server。

## 開發

見 [AGENTS.md](AGENTS.md) 的「開發這個 repo」段落。
