# CLAUDE.md — loan-eligibility-checker

單檔前端工具：問答式健檢 → 逐條核對「微型創業鳳凰貸款」與「就業保險失業者創業貸款」兩方案的資格條件 → 結論卡片（建議申請哪個方案／都不符合的原因／兩者互斥）。無建置步驟、無框架、無 package.json、**無後端**，直接開啟 `index.html`（`file://`）或以靜態伺服器託管即可。所有運算都在瀏覽器本機執行，填答內容不落地儲存（無 localStorage）、不送出到任何伺服器。

**本工具與 `phoenix-loan-generator` 是不同性質的工具，不要混淆：** 這個是「資格自我核對／方案建議」，不做申請書填表；`phoenix-loan-generator` 是「鳳凰貸款申請書填表輔助」，不做資格比對。兩者互為前後道工序（先用這個核對資格 → 確定要申請鳳凰貸款再用那個填表），各自獨立維護。

`manual.html`／`README.md` 沿用其他 generator 專案的創作者資料區塊（與 `phoenix-loan-generator`、`sbir-generator`、`icap-generator` 的 manual.html 為同一份人物資料，更新其中一邊時建議一併同步其餘各邊，但非強制共用檔案）。

## 背景與資料來源

兩份法規皆為 2026 年勞動部令修正、均自民國 115 年 8 月 1 日生效：

- **微型創業鳳凰貸款要點**（115.7.30 勞動發創字第1150509757號令修正）
- **就業保險失業者創業協助辦法**（115.6.10 勞動發創字第1150507318號令修正）

兩份法規皆多次提及第三個互斥方案「失業中高齡者及高齡者創業貸款」，但本專案未取得該法規全文，因此**不單獨提供其資格核對**，僅在「同一事業／負責人是否已獲貸其他方案且未清償」的排除條件中，將其與另一方案一併詢問（因為兩份來源法規本身的條文就是把這兩個方案綁在一起寫的，見下表 R10b/R10c）。

## 架構

單一 `index.html`：內嵌 `<style>` 與 `<script>`，無外部資源、無 `fetch`。核心是一份 `RULES` 陣列（每條規則含 `programs`／`clause`／`text`／`evaluate(answers)`／`reasonFail(answers)`），對兩個方案的判斷完全共用同一套規則清單、依 `programs` 欄位過濾套用對象，而不是各寫一份判斷邏輯——**未來法規異動時，改這份陣列即可，不要另外複製一份判斷函式**。表單沒有用 `data-path` 綁定巢狀 state（像 phoenix-loan-generator 那樣），而是用 `getAnswers()` 直接讀取固定的 `id`/`name` 屬性，因為欄位數量少（14 個問題），沒有巢狀資料的必要。

- 事件綁定在 `#main` 容器上用 `input`/`change` 事件委派，任何欄位變動就整頁重新 `render()`，不做分項局部更新（欄位少，重繪成本可忽略）。
- `computeProgram(key, answers)` 回傳 `{items, overall}`：`overall` 為 `pass`（全部通過）／`fail`（任一條 fail）／`pending`（尚有未填但無 fail）。
- `buildConclusion(ph, ui, bizType)` 依兩個 `overall` 的組合列出對應結論文案，涵蓋「兩者皆不符合」「僅一方符合」「兩者皆符合（互斥說明）」「尚未填答完整」四種情境。
- `loanCap(bizType)`：兩份法規的貸款額度規則完全相同（小規模商業 50 萬；公司/商業/有限合夥/托嬰中心等 200 萬），故兩個方案共用同一個試算函式。

## 資格規則對照表（RULES 陣列 ↔ 法規條文，法規異動時對照更新用）

| 規則 id | 適用方案 | 鳳凰貸款條文 | 就業保險方案條文 | 內容 |
|---|---|---|---|---|
| registration | 共通 | 第四點 | 第六條 | 已完成事業登記／立案登記／稅籍登記 |
| setupYears | 鳳凰限定 | 第四點 | — | 設立登記未超過5年 |
| employeeCount | 鳳凰限定 | 第四點 | — | 員工（不含負責人）未滿5人 |
| isResponsiblePerson | 共通 | 第三點第(二)款 | 第五條第三款 | 登記為負責人並實際經營 |
| concurrentOtherBiz | 鳳凰限定 | 第三點第(三)款 | — | 未同時任其他事業負責人/合夥人/董監事 |
| before14days | 就業保險限定 | — | 第五條第四款 | 登記前14日內無投保紀錄且未任其他事業負責人 |
| consultingDone | 共通 | 第三點第(一)款 | 第五條第一款 | 已接受創業諮詢輔導及適性分析 |
| courseHoursEnough | 共通 | 第三點第(一)款 | 第五條第二款 | 3年內研習課程滿18小時 |
| genderAgeIsland | 鳳凰限定 | 第三點 | — | 女性未滿45／45~65歲／65歲以下離島 三者擇一 |
| isInsured | 就業保險限定 | — | 第二條、第五條 | 具就業保險被保險人身分 |
| creditIssue | 共通 | 第八點第(一)(二)款 | 第十七條第一、二款 | 無票據/信用瑕疵 |
| sameBizOtherLoanUI | 鳳凰限定 | 第八點第(四)款 | — | 同一事業未獲貸就業保險方案或中高齡方案且未清償 |
| sameBizOtherLoanPhoenix | 就業保險限定 | — | 第十七條第五款 | 同一事業未獲貸鳳凰貸款或中高齡方案且未清償 |
| respOtherBizLoan | 共通 | 第八點第(五)款 | 第十七條第六款 | 負責人未以其他事業獲得三方案其一且未清償 |

貸款額度（兩方案相同）：鳳凰貸款第六點／就業保險方案第七條——小規模商業 50 萬；公司/商業/有限合夥或托嬰中心等 200 萬。增貸規則（兩方案相同）：鳳凰貸款第九點／就業保険方案第九條——最多增貸2次，首次＋二次＋三次核給金額合計不得超過上限；此工具僅以頁面底部勾選框顯示提醒文字，不做完整分支判斷邏輯。

## 加入主畫面（PWA，2026-08-14 新增）

比照工作區其餘 10 個已加裝 PWA 的線上工具（`expense-tracker-pwa` 為原始版本，見該專案 CLAUDE.md）：`manifest.json`＋`icons/`（淺灰藍 `#f4f6fb` 背景、藍色 `#2563EB`「貸」字圖示，對應 `--bg`／`--blue`）＋`service-worker.js`（network-first＋同源快取備援；本工具本來就無後端無 `fetch`，SW 純粹是安裝資格判定用）。安裝按鈕（`#installBtn`）放在 `.actions`（跟「操作手冊」「重新填寫」同排、同 `.secondary` 樣式）。

**這次是九個工具做完後才發現漏掉的第十個**：這個工具有公開 GitHub Pages 部署（<https://m255525.github.io/loan-eligibility-checker/>），但本檔（CLAUDE.md）此前完全沒提過「GitHub Pages」或「github.io」字樣，先前用文字搜尋工作區文件找「還有哪些網站要加裝」時因此漏掉——**之後要盤點工作區已上線網站，不能只靠 grep CLAUDE.md/README.md 找關鍵字，要用 `gh repo list M255525` 列出全部公開 repo、逐一 `gh api repos/M255525/<repo>/pages` 查詢才準確**。已從一開始就內建 iOS／iPadOS／macOS 相容性（`isIOSDevice`／`isMacDesktop && isSafariEngine`／`isStandalone` 三種判斷＋對應指引文字，`apple-touch-icon` 180×180＋`mobile-web-app-capable`／`apple-mobile-web-app-capable` 雙標籤），不像前 9 個工具是先做完再回頭補一輪，細節見 [[pwa-install-rollout]]。已用 Playwright 實測 Chromium 觸發 `beforeinstallprompt`、SW 註冊成功。

**回饋機制與快取踩坑修正（2026-08-14，其他姊妹專案使用者實測回報「加入主畫面沒有功能」才發現兩層問題，這個工具是同一批一起修）**：(1) 無 `showToast` 時原本用「暫時置換按鈕文字」當提示，在工具列裡太不明顯——改成 `window.alert(fallbackMessage())`，`deferredPrompt.prompt()` 也包 try/catch。(2) `service-worker.js` 的 `fetch(event.request)` 沒有繞過瀏覽器 HTTP 快取——GitHub Pages 對回應下 `Cache-Control: max-age=600`，10 分鐘內「network-first」名不符實，可能吃到舊版內容重新存進 Cache Storage。改成 `fetch(event.request, {cache:'reload'})` 強制略過 HTTP 快取，`CACHE_NAME` 同步升版 v1→v2。細節見 [[pwa-install-rollout]]。

## 指令

無建置/測試指令。修改 `index.html` 後直接用瀏覽器開啟驗證即可（若本機未連接 Preview MCP，可用 `python -m http.server <port> --directory 政府補助認證產生器/loan-eligibility-checker` 暫時起一個靜態伺服器測試，測完記得關閉）。

改動 `RULES` 陣列後，建議在瀏覽器 console 用 `document.getElementById(...)`／`document.querySelector('input[name=...]')` 手動賦值＋觸發 `input`/`change` 事件，確認對應 checklist 與結論即時更新且判斷正確，尤其留意「兩者皆符合」與「共通排除條件命中」這兩種較少見的組合是否仍正確顯示。
