# 堂數點名簿 — 專案說明（給 Claude Code）

才藝教室（塗鴉班／幼兒班／兒童班／進階班）老師用的 iPhone PWA：點名、調課補課、停課、整期／點數制繳費、收入統計、Excel 匯出。
使用者是教室老師，用繁體中文溝通，UI 文字一律繁體中文（台灣用語）。

## 技術與檔案

- 純 HTML + CSS + 原生 JavaScript，**沒有框架、沒有建置步驟**。整個 App 在 `index.html` 一個檔案裡。
- `sw.js`：Service Worker，網路優先、離線時用快取。**每次改版要把開頭的快取名稱 `const C='rollbook-vN'` 的 N 加 1**，手機才會換新版。
- `manifest.webmanifest`、`icon-*.png`：加入 iPhone 主畫面用。
- 外部資源：Google Fonts（霞鶩文楷 TC、思源黑體 TC、IBM Plex Mono）；SheetJS `xlsx@0.18.5` 由 cdnjs 在匯出時才動態載入。
- 部署：把整個資料夾上傳到 HTTPS 靜態空間（GitHub Pages / Netlify），用 Safari「加入主畫面」。

本機測試：`python3 -m http.server 8000` 後開 `http://localhost:8000`。

## 測試版（改版先放這裡）

- 測試版網址 https://cian0325.github.io/rollbook-test/ （GitHub 專案 `cian0325/rollbook-test`）。**老師沒說要上線，就只更新測試版，不要動這個正式版專案。**
- 測試版由正式版的 index.html 轉出來（`make_test.py`）：因為兩邊同網域、瀏覽器儲存空間共用，所以全部改名隔離——`localStorage` KEY 改 `rollbook-test.v3`、雲端集合改 `test_users`／`test_albums`、Firebase app 名稱 `rbtest`、快取名稱 `rollbook-test-vN`，畫面上有橘色「測試版」緞帶。
- `sw.js` 啟用時只刪 `rollbook-v` 開頭的舊快取，不會刪到測試版的快取。

## 資料（最重要的規則）

- 全部存在 `localStorage['rollbook.v3']`，一個 JSON 物件 `S`。
- **絕對不要改 KEY 或丟掉舊欄位**：老師手機裡有真實資料。新增欄位時在檔尾 `S=load()||seed();` 後面補預設值（現有的遷移寫法：`S.levels=S.levels||defaultLevels();…`）。
- 改動資料格式前，先確認「設定 → 備份與還原」的匯入（`App._import`）也能處理舊備份。

```
S = {
  settings: { studio, unitPrice, haptic, sound, lastBackup },
  levels:   [{ id, name, tone }]                       // 班級類別，tone 決定顏色 0–5
  slots:    [{ id, wd, start:'HH:MM', end, name, level }]   // 每週固定時段，wd 0=日
  terms:    [{ id, name, start:'YYYY-MM-DD', weeks, price, topics?:{levelId:[週主題]} }] // 期別；price = 每堂價格；結束日 = start + weeks*7 - 1
  students: [{ id, name, level, plan:'term'|'points', phone, note, active,
               slotHist:[{ from:'YYYY-MM-DD', slotId }] }]   // 時段調整歷史，依生效日
  records:  [{ id, sid, date, key:slotId, status:'present'|'absent', mk? }] // 點名；mk = 補課/加課的 makeup id
  makeups:  [{ id, sid, absId|null, date, slotId, type?:'credit', termId?, purchaseId? }]
            // absId=補哪一堂沒來的課；absId=null 是加課；type='credit' 是折抵學費（不是實際上課）
  purchases:[{ id, sid, kind:'term'|'points', termId, lessons, amount, adj, adjReason,
               date, paid, paidDate, method, note }]
  closures: [{ id, date, slotId|null, reason }]        // 停課；slotId=null 全天
  confirmed:{ 'YYYY-MM-DD|slotId': { at: timestamp } } // 已按「確認點名」的時段
  sample?: true                                        // 範例資料旗標
}
```

## 學生名單匯入

「設定 → 備份與還原」貼上或選擇 `type:'rollbook-roster'` 的 JSON 會走 `rosterImport()`：依名稱合併類別、依「星期＋開始時間＋類別」合併時段、同名學生略過，不覆蓋既有資料（範例資料時會先清空範例）。格式：
`{type:'rollbook-roster', title, levels:[名稱], slots:[{wd,start,end,level}], students:[{name,level,wd,start,plan?,phone?,note?}], from?, term?:{name,start,weeks,price}, bill?:{paid,date,method,unpaid?:[姓名]}, closures?:[{date,reason}]}`；`term.topics` 是 `{類別名稱:[每週主題]}`，存成 `terms[].topics{levelId:[...]}`，點名頁依週次顯示。匯入預覽可選「加到現有資料」或「全部重建」，`wd:null` 表示時段待確認；有 `term` 會建立期別（同名沿用），有 `bill` 會幫新學生各開一張該期繳費單。
同一時間可以有不同類別的時段（例如週五 19:30 兒童班與進階班）。

## 核心規則（改邏輯前先讀）

- **剩餘堂數** `bal(sid)` = 已購堂數 − 狀態為「到」的紀錄數（正課、補課、加課都扣）。「沒來」不扣。
- **應上堂數** `sessions()` / `termLessons()`：依學生當天的固定時段（`slotOn`）計算，排除停課（`isClosed`）。
- **待補課** `pendingAbs()`：正課「沒來」且沒有對應的補課或折抵（補課又沒來時會回到待補）。
- **續期試算** `renewal(s, T, keep, adj)`：需繳 = 下期應上 − 可結轉（`carryInto`）+ 保留補課的堂數；金額 = 需繳 × 每堂價格 + 調整。`carryInto` 會預扣本期還沒上的課和已排的補課。
- **點名草稿**：按「到／沒來」只改 `UI.drafts`，按「確認點名」（`App._confirm`）才寫入 `records`，並記在 `confirmed`。
- 停課不算應上、不扣堂、不進待補。已確認點名的時段不能設停課。

## 程式結構（index.html 的 `<script>`）

依區塊註解 `/* ---------- 區塊 ---------- */` 排列：工具函式 → storage → 時段 → 班級類別 → 補課 → 堂數計算 → 範例資料 `seed()` → UI → 點名 `vRoll` → 調課 `vMakeup` → 學生 `vStudents` → 作品 `vWorks` → 繳費 `vPay` → 統計 `vStats` → 設定 `vSettings` → 停課 → 期末未補課清單 → 學生查詢 → 按鍵回饋 → 檔案輸出（Excel、.ics）→ `App` 物件（所有 onclick 的動作）。

- 畫面用字串樣板產生 HTML，`render()` 重畫目前分頁；彈出面板用 `sheet(html)`。
- 所有使用者輸入顯示前都要經過 `esc()`。
- 確認對話框用自製的 `confirmSheet()`，不要用 `confirm()`/`alert()`。
- 存檔用 `saveFile()`（iPhone 會跳分享選單）。
- 底部分頁：點名｜調課｜學生｜作品｜帳務。「帳務」內上方切換繳費（`pay`）／統計（`stats`）；設定在頁首右上角齒輪（`#set-btn`）。
- 調課頁的待補課依學生分組，點學生開 `mkStuSheet(sid)` 再選補課或折抵。
- 學生頁預設「依星期」（`UI.sv`）：`vStuWeek` 依星期 → 時段 → 學生（學生歸在 `curSlot` 的時段），「依班級」是原本的清單；搜尋框有字時自動用清單。
- 搜尋框與注音輸入法：組字中（compositionstart～compositionend）不重畫畫面，否則注音會被直接送出。用 document 層級監聽 composition 事件（HTML 的 oncomposition* 屬性無效）。新增會重畫畫面的輸入框時要沿用這個做法。
- **作品功能在正式版先關閉**：index.html 開頭的 `window.HIDE_WORKS=true` 會拿掉「作品」分頁（`WORKS_ON`）。要開放時刪掉那一行，並同時部署 `albums` 的 Firestore 規則、上傳 `album.html`（家長頁）。

## 設計

- 樣式只有一層，數值都用 token：字級 `--fs-xs/sm/md/lg/xl/2xl`（12/13/15/17/20/28）、圓角 `--r-sm/md/lg/full`、陰影 `--sh-1/--sh-2`、間距用 4 的倍數。不要再加寫死的數值。
- 色彩全部用 CSS 變數（`--paper --card --ink --ink-2 --ink-3 --line --soft --indigo --indigo-soft --seal --amber --green --violet`），淺色、深色兩套都要定義。藍色＝主要動作／選中；紅色＝要注意（待補、未收、刪除）；班級顏色只用在小色點 `.lv` 和頭像 `.av`。
- 按鈕只有四種：`.btn`（一般）、`.btn.primary`、`.btn.ghost`、`.btn.danger`，大小加 `.sm`。狀態小標一律 `.pill`。
- 字體：標題 `--f-display`（霞鶩文楷），內文 `--f-body`，數字用內文字體 + tabular-nums；等寬字體 `--f-mono` 只用在家長密碼。
- 手機優先：最窄 360px 不可橫向捲動；注意 `env(safe-area-inset-*)`。
- 尊重 `prefers-reduced-motion`。

## 雲端同步（已完成，Firebase）

- 程式在 `/* ---------- 雲端同步（Firebase Firestore） ---------- */` 區塊。SDK `firebase-*-compat.js` v10.12.2 從 gstatic 動態載入；沒設定 Firebase 時完全不載入，App 照常只用本機。
- 設定來源：`localStorage['rollbook.cloud'].cfg`（在「設定 → 雲端同步」貼上 firebaseConfig），或在 index.html 定義 `window.FIREBASE_CONFIG`。
- 結構：`users/{uid}/parts/core`（records 以外的全部 S，JSON 字串）與 `users/{uid}/parts/r-YYYY-MM`（當月 records）。每份 `{json, rev, device, updatedAt}`。
- 本機仍是主資料（localStorage），`save()` 後 1.5 秒上傳有變動的段落。上傳用 transaction 比對 rev，不同時用 `merge3(上次同步, 這台, 雲端)` 三方合併（id 陣列逐筆合併、物件逐鍵合併、雙方都改以這台為準）。上次同步內容存在 `localStorage['rollbook.cloud.base']`。
- `onSnapshot` 即時接收其他裝置的修改；這台沒改過的段落直接套用。
- 第一次登入：雲端空 → 上傳；這台是範例／空 → 載入雲端；兩邊都有資料 → 讓使用者選。
- Firestore 規則只允許 `request.auth.uid == uid` 讀寫 `users/{uid}/**`。
- 資安設定（2026-11）：Firebase 金鑰在 Google Cloud 限制只能從 `https://cian0325.github.io/*`、`https://rollbook-cb3af.firebaseapp.com/*` 使用；Authentication 已關閉「建立（註冊）」，只有老師的帳號。換網域時要先把新網域加進金鑰限制，否則雲端會連不上。
- 加密預設名單 `seed.enc.json` 已從 GitHub 移除（公開檔案可被離線猜密碼）。程式找不到檔案時會略過，不要再把名單放上公開專案。
- `sw.js` 只快取同網域與 gstatic/cdnjs/字型，Firestore 連線不經過快取。
- 注意：資料長大後 localStorage（約 5MB）會同時存資料與同步基準，數年後可能需要改用 IndexedDB。

## 下一步計畫（老師已同意的方向）

1. ~~雲端同步~~：已完成（Firebase，見上方）。
2. **作品分頁**：程式已完成、只在測試版開放（正式版 `HIDE_WORKS`）。選學生 → 選期別與作品（每週主題）→ 拍照，自動偵測畫紙四角（`detectPaper`：邊框顏色分群＋霍夫直線＋邊緣精修，回傳信心分數）並拉正，全部先給老師看一眼再上傳。照片存 Firestore `albums/{aid}`（`aid` = 家長密碼的 SHA-256；`photos` 縮圖＋資料、`full` 原圖 ≤ 88 萬字元）。可換家長密碼（`wkRekeyDo`，搬照片到新 aid、舊的刪除）。
3. **家長作品頁** `album.html`：只在測試版。家長輸入 8 碼密碼（不用帳號），只讀得到那一本；有隱私說明。規則：`albums/{aid}` get 可、list 不可，寫入限 owner。
   - 已知限制：Firestore 免費 1GB 約放 2,400 張，正式開放前要改用 Firebase Storage 或其他儲存方式。
4. 選做：每位學生一張 QR Code 小卡，拍作品時掃描自動分類。

**不做人臉辨識**：兒童生物特徵屬敏感個資，辨識也不穩定。改用「先選學生再拍照」或 QR Code。上傳兒童照片前需取得家長同意，家長頁要有隱私說明。

## 其他待辦想法

剩餘堂數提醒＋續費訊息、各類別不同價格、家長聯絡資訊、自動備份、專班不固定時段、時段人數上限、材料費與支出記帳、體驗課／候補名單、升班紀錄、生日提醒。
