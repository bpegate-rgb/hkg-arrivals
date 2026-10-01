# 香港國際機場抵港航班

介面目前採用淺色配色，搭配黃色重點按鈕及清晰的航班狀態標示。

## 最近轉機安檢站

使用者於 2026-10-01 提供以下配對，儲存在 `dist/data/transfer-routes.json`，mock 與正式 API 模式共用。每次刷新按到達閘口精確配對，來源標示「使用者提供」。

| 到達閘口 | 轉機安檢站 |
| --- | --- |
| 1–4 | E2 |
| 5–9 | E1 |
| 10、12、14、18、20、22、24、26、38 | E1 |
| 11、13、15、17、19、21、23、25、37、39 | E2 |
| 27–36 | T3 |
| 40–59 | T5 |
| 60–80 | T6 |

閘口 16 未有提供配對，顯示「待確認」。其他未列閘口或未公布閘口亦顯示「待確認」；取消航班顯示「不適用」。配對來源為使用者，未宣稱經官方確認最短路線或開放狀態。

資料處理在 `dist/js/transfer.js`。新增配對格式：

```json
{"arrivalGate":"23","stationName":"轉機安檢站 E2","location":"閘口對照：使用者提供","userProvided":true}
```

亦可加入經管理者核實的項目，使用 `verified:true` 及 HTTPS `sourceUrl`；此為管理者標記，不代表程式自動核實。路線載入失敗時航班仍正常顯示，安檢位置顯示待確認。

頁面附有[官方轉機區位置](https://www.hongkongairport.com/en/passenger-guide/airport-facilities-services/transfer-area)連結。

免建置、無第三方前端依賴的 HTML / CSS / JavaScript 專案。介面為繁體中文，所有時間固定使用 Asia/Hong_Kong。**預設全為虛構模擬資料，不可用於接機或航班決策。**

## 啟動

安裝 Node.js 20 或以上，在本資料夾開啟終端機：

```sh
npm start
```

瀏覽 http://127.0.0.1:4173 。不需要 npm install、不需打包。Ctrl+C 停止服務。亦可用任何靜態 HTTP 伺服器服務 `dist/`；不要直接以 file:// 雙擊 HTML，瀏覽器會限制 ES modules / fetch。`server.mjs` 是本機示範伺服器，並非正式 API 後端。

## 檔案結構

```text
hkg-arrivals/
├── README.md
├── package.json
├── server.mjs                 本機靜態伺服器
├── tests/core.test.mjs        資料與搜尋測試
└── dist/                     可直接部署的網頁
    ├── index.html            UI 結構
    ├── styles.css            淺色及響應式樣式
    ├── data/mock-flights.json 獨立模擬資料
    └── js/
        ├── app.js            畫面狀態、事件、更新排程
        ├── ui.js             安全 DOM 輸出、香港時間格式
        ├── config.js         provider / endpoint 設定
        ├── api.js            資料來源抽象層
        ├── processing.js     驗證、排序、統計
        └── filters.js        搜尋與狀態篩選
```

## 功能與定義

12 個虛構航班涵蓋準時、已抵達、延誤、取消、未公布閘口。模擬時間以本次載入頁面的整點為基準加上 JSON 的分鐘 offset，刷新不移動基準。資料不模擬真實飛行進度，重新載入會建立新時間基準。

預設每 2 分鐘刷新，可改為 1 / 5 分鐘或暫停；手動刷新永遠可用（請求進行中除外）。每次請求完成後安排下次刷新，不重疊發送。背景分頁可能受瀏覽器節流，回到分頁時若已到更新時間會補刷新。

即將抵達 = 準時 + 延誤；並非額外一種狀態。總數及摘要為全部取得航班，搜尋結果數則反映目前搜尋與篩選。以實際 → 預計 → 原定時間排序，使用完整日期避免跨日錯序。取消航班應由後端將實際及預計時間設為 null，以原定時間排序。每筆顯示日期及時間、原定時間、航班、航空公司、出發城市／機場、狀態、到達閘口。缺少閘口顯示「待公布」。

更新失敗時保留上次成功資料及最後成功更新時間，顯示錯誤；首次失敗不會顯示成正常空資料。API 模式失敗不會偷偷切換到模擬資料。介面支援鍵盤、表格標題、狀態按鈕 aria-pressed、loading 及錯誤通知。手機改用雙欄航班卡片。

## API 接入設定

修改 `dist/js/config.js`：

```js
export const config = {
  provider: 'api',
  endpoint: '/api/arrivals',
  timeoutMs: 10000,
  refreshMs: 120000,
  mockLatencyMs: 650
};
```

`/api/arrivals` 是**待實作的自有後端介面**，不是已可用的供應商 API。第一版只提供靜態網頁；設定 api 前須另行建置後端。正式流程：瀏覽器 → 自有 HTTPS 後端 → 已授權供應商 JSON API。由後端保管金鑰、設定存取權限、快取、限流、重試及供應商映射。切勿把供應商密鑰寫入 config.js 或任何公開檔案。跨網域後端須允許正確 CORS origin；優先同源部署。

供應商 adapter 應把資料轉為以下格式；前端共用入口是 `api.js` 的 `fetchFlights()`，供應商差異不應流入 UI。

```json
{
  "flights": [{
    "id": "2026-10-01-CX123-HKG",
    "flightNumber": "CX123",
    "airline": "國泰航空",
    "originCity": "東京",
    "originAirport": "成田",
    "originCode": "NRT",
    "scheduledArrival": "2026-10-01T19:00:00+08:00",
    "estimatedArrival": "2026-10-01T19:30:00+08:00",
    "actualArrival": null,
    "status": "delayed",
    "gate": null
  }]
}
```

必填文字：id（唯一，應含航班日期）、flightNumber、airline、originCity、originAirport、originCode、scheduledArrival、status。時間為含 Z 或時區偏移的 ISO 8601；estimatedArrival、actualArrival、gate 可 null。status 限 scheduled / arrived / delayed / cancelled；arrived 必須有 actualArrival。後端只回傳 HKG 抵港航班，決定時間範圍及共掛班號去重規則。閘口、停機位、行李帶、抵港大堂是不同欄位，不可互相冒充；無到達閘口資料回傳 null。非 2xx、無效 JSON、格式錯誤、逾時均顯示錯誤並保留舊資料。

## 資料來源研究（2026-10-01 查閱）

1. **香港機場管理局 / DATA.GOV.HK 官方公開資料**：[資料集](https://data.gov.hk/en-data/dataset/aahk-team1-flight-info)、[官方 REST 規格](https://www.hongkongairport.com/iwov-resources/misc/opendata/Flight_Information_DataSpec_en.pdf)。政府資料頁標示每日更新至前一曆日；REST 的 `/flightinfo-rest/rest/flights/past` 提供歷史 JSON，參數包括 date、arrival=true、cargo=false、lang=zh_HK。這不是已確認的即時 API。規格的抵港資料列有 stand（停機位），不應當成 gate。適合歷史展示／分析，不足以滿足即時接機。正式使用須閱讀資料頁的適用使用條款並保留來源標示。
2. **FlightAware AeroAPI（商業）**：[官方產品及端點介紹](https://www.flightaware.com/commercial/aeroapi/)。提供機場抵港、預定抵港及航班狀態等資料；須確認所購方案的 HKG 覆蓋、預計／實際時間、到達閘口完整度、費用、呼叫限制及公開展示／再分發授權。需要後端金鑰及 adapter。
3. **Cirium（商業）**：[官方產品](https://www.cirium.com/data/aviation-api/)、[Flight Status API 文件](https://developer.cirium.com/apis/cirium-sky-api/flight-status)。可評估機場／日期航班狀態查詢。須向供應商確認香港覆蓋、閘口資料、服務等級、費用及展示授權，同樣透過後端接入。

上述為官方文件研究，未以付費帳戶驗證端點或即時 HKG 資料品質，未宣稱所有欄位均可取得。不抓取機場 HTML，也不依賴網站內部未公開介面。取得供應商授權及驗證資料完整度後，再把 provider 改為 api。

## 檢查與狀態示範

```sh
npm test
```

測試涵蓋跨日／實際時間排序、搜尋正規化、狀態與統計、空資料及格式錯誤。

- 正常：`http://127.0.0.1:4173/`
- 空資料：`http://127.0.0.1:4173/?scenario=empty`
- 連線錯誤：`http://127.0.0.1:4173/?scenario=error`
- Loading：每次模擬請求預設等待 650ms，可在 config 增加 mockLatencyMs。
- 舊資料保留：先正常載入，再在瀏覽器開發工具將網路切離線並按刷新。
- 搜尋不存在的航班可查看搜尋 Empty State，按「清除搜尋與篩選」恢復。

scenario 只在 mock 模式生效，正式 API 不會被網址參數切成模擬資料。無外部字型、CDN 或追蹤工具。

## 儲存到 GitHub（網頁上傳）

1. 登入 GitHub，建立新 repository，名稱可用 `hkg-arrivals`，按需要選 Public 或 Private，勾選建立 README。
2. 解壓專案 ZIP，開啟新 repository，選 Add file → Upload files。
3. 把解壓後的 `dist/`、`tests/`、`package.json`、`server.mjs`、`README.md`、`VERIFICATION.md`、`.gitignore` 上傳到 repository 根目錄，保留資料夾結構。上傳原始檔案，不是只上傳 ZIP。
4. 填入提交說明，例如「新增淺色抵港航班專案」，按 Commit changes。

完成後可在 GitHub 查看及下載整個專案。儲存原始碼不等於已將網頁上線；本機仍使用 `npm start` 啟動。

參考：[GitHub 官方上傳文件](https://docs.github.com/en/repositories/working-with-files/managing-files/adding-a-file-to-a-repository)。


