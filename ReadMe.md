# MyAISkill

AI Coding Agent 自訂 Skill 集合，適用於 Claude Code 與 GitHub Copilot。

## 安裝與更新

使用 [vercel-labs/skills](https://github.com/vercel-labs/skills) CLI 工具，支援 40+ 種 AI Coding Agent（Claude Code、GitHub Copilot、Cursor 等），自動安裝至對應路徑。

**安裝**

```bash
npx skills add kkman021/MyAISkill
```

執行後以互動方式選擇要安裝的 skill 與目標 agent。

**檢查可用更新**

```bash
npx skills check
```

**更新所有已安裝的 skills**

```bash
npx skills update
```

**查看已安裝清單**

```bash
npx skills list
```

---

## Skills

### ado-task-analyzer

從 Azure DevOps 查詢中抓取 Work Items，分析描述內容，探索 codebase 定位問題，最後將執行計畫或釐清請求以 comment 形式回覆至 ADO。

#### 觸發時機

以下情境會自動觸發此 skill：

- 提及 Azure DevOps task、ADO work items、DevOps 查詢結果
- 要求分析 ADO bug、調查 work items、回覆 ADO issue
- 中文口語如「幫我看 ADO task」、「分析 DevOps 的工作項目」、「去 Azure DevOps 抓 task」

也可直接使用 slash command：`/ado-task-analyzer`

#### 環境變數

| 變數 | 必要 | 說明 |
|---|---|---|
| `AzureDevOps_PAT` | 是 | Personal Access Token，需有 Work Items 讀寫權限 |
| `AzureDevOps_ORG` | 是 | 組織名稱（例：`mycompany`） |
| `AzureDevOps_PROJECT` | 是 | 專案名稱（例：`MyProject`） |
| `AzureDevOps_QUERY_ID` | 條件性 | 已儲存查詢的 GUID；若直接指定 Work Item ID 則不需要 |

#### 環境變數設定

<details>
<summary><strong>Claude Code</strong></summary>

在 `~/.claude/settings.json` 中設定：

```json
{
  "env": {
    "AzureDevOps_PAT": "your-pat-token",
    "AzureDevOps_ORG": "your-org",
    "AzureDevOps_PROJECT": "your-project",
    "AzureDevOps_QUERY_ID": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
  }
}
```

</details>

<details>
<summary><strong>GitHub Copilot</strong></summary>

Copilot 無 settings.json 可設定環境變數，需透過系統環境變數提供。

**Windows** — 在系統環境變數中新增：

```
AzureDevOps_PAT=your-pat-token
AzureDevOps_ORG=your-org
AzureDevOps_PROJECT=your-project
AzureDevOps_QUERY_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

設定路徑：系統設定 > 系統 > 進階系統設定 > 環境變數，或透過 PowerShell：

```powershell
[System.Environment]::SetEnvironmentVariable('AzureDevOps_PAT', 'your-pat-token', 'User')
[System.Environment]::SetEnvironmentVariable('AzureDevOps_ORG', 'your-org', 'User')
[System.Environment]::SetEnvironmentVariable('AzureDevOps_PROJECT', 'your-project', 'User')
[System.Environment]::SetEnvironmentVariable('AzureDevOps_QUERY_ID', 'xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx', 'User')
```

設定後需重啟 VS Code 使環境變數生效。

</details>

#### 使用方式

**方式一：透過查詢抓取所有 Work Items**

```
幫我看 ADO task
```

Skill 會使用 `AzureDevOps_QUERY_ID` 抓取查詢結果中的所有 work items。

**方式二：指定特定 Work Item ID**

```
分析 ADO work item #12345 和 #12346
```

此方式不需要設定 `AzureDevOps_QUERY_ID`。

#### 處理流程

1. **驗證環境變數** — 缺少必要變數時停止並提示
2. **抓取 Work Items** — 從查詢或指定 ID 取得項目
3. **分析意圖** — 判斷描述是否足夠清楚
4. **探索 Codebase** — 搜尋相關程式碼，定位問題
5. **產出結果** — 根據分析結果產生以下三種回應之一：
   - **執行計畫**：問題已定位，產出含根因分析與修改步驟的計畫
   - **調查報告**：意圖清楚但無法在 codebase 中定位問題
   - **釐清請求**：描述不足，列出需要補充的資訊
6. **回覆 ADO** — 將結果以 Markdown comment 發佈至對應 work item
7. **輸出摘要表** — 列出所有處理項目及其結果

#### 注意事項

- Skill 僅負責分析與回覆 comment，不會自動修改程式碼
- 實作變更需另外明確指示
- PAT 需有足夠權限（Work Items Read & Write）

---

### playwright-cli

透過 `playwright-cli` CLI 工具控制瀏覽器，執行網頁自動化操作，包含導覽、表單填寫、截圖、資料擷取與 E2E 測試。

#### 觸發時機

以下情境會自動觸發此 skill：

- 需要導覽網頁、點擊元素、填寫表單、截圖
- 執行 E2E 測試或網頁自動化流程
- 從網頁擷取資料或驗證頁面狀態

#### 安裝 playwright-cli 工具

`playwright-cli` 是獨立的 npm 套件，需先安裝才能使用。

**全域安裝（建議）**

```bash
npm install -g @playwright/cli
```

安裝後即可直接執行 `playwright-cli`。

**臨時使用（無需安裝）**

```bash
npx @playwright/cli open https://example.com
```

> 舊套件名稱 `playwright-cli` 已棄用，現行套件為 `@playwright/cli`，但執行的二進位檔名稱仍為 `playwright-cli`。

**確認安裝**

```bash
playwright-cli --version
```

#### 主要指令

```bash
playwright-cli open https://example.com   # 開啟瀏覽器並導覽
playwright-cli snapshot                   # 取得頁面快照（含元素 ref）
playwright-cli click e3                   # 點擊元素
playwright-cli fill e5 "value"            # 填入文字
playwright-cli screenshot                 # 截圖
playwright-cli close                      # 關閉瀏覽器
```

若全域 `playwright-cli` 無法執行，改用 `npx playwright-cli`。

#### 功能涵蓋

- 核心操作：點擊、輸入、拖曳、選擇、上傳、checkbox
- 鍵盤與滑鼠事件
- 多 Tab 管理
- Cookie / LocalStorage / SessionStorage 操作
- Network request mocking
- DevTools：console、network log、tracing、video recording
- 多瀏覽器：Chrome、Firefox、WebKit、Edge

---

### gdpr-cookieyes-audit

模擬首次訪客（無既有 consent 狀態），偵測在使用者點擊同意按鈕前即已執行的第三方追蹤 script、iframe 與網路請求，稽核是否違反 GDPR Prior Consent 規範。

#### 觸發時機

以下情境會自動觸發此 skill：

- 提供 URL 並詢問 cookie 合規或「偷跑」追蹤問題
- 需要 GDPR 稽核、CookieYes 防護缺口分析
- 提及 GTM、Hotjar、VWO、YouTube embed 等可能在未獲授權前載入的追蹤工具

#### 稽核流程

1. 以 in-memory profile 開啟瀏覽器（確保無既有 consent cookie）
2. 攔截所有 network request
3. 掃描 `<script src>` — 偵測無 `data-cookieyes` 的已知追蹤域
4. 掃描 `<iframe>` — 偵測有 live `src` 且無 `data-cookieyes` 的嵌入內容
5. 稽核 GTM consent default 順序 — `consent default` 須在 `gtm.start` 之前
6. 確認 CookieYes banner 出現（健全性檢查）

#### 違規分類

| 類別 | 說明 |
|---|---|
| Cat 1 — Hardcoded script | 外部追蹤 script 無 `data-cookieyes`，頁面載入即執行 |
| Cat 2 — GTM 無 consent default | GTM 在未設定 consent default 前即啟動 |
| Cat 3 — Iframe | 含 live `src` 的 embed iframe 無 `data-cookieyes` |

#### 輸出格式

輸出違規表格，分為 **Action A**（需立即修正）與 **Action B**（需法務審查），並列出已通過檢查的項目。

---

### temp-email

透過 TempMail API 管理臨時/一次性 email 信箱，適用於 E2E 測試工作流程——註冊驗證、邀請接受、密碼重設流程等需要拋棄式 email 的情境。

#### 觸發時機

以下情境會自動觸發此 skill：

- 需要建立臨時信箱、取得測試用 email 地址
- 查收驗證信、讀取 email 內容、擷取驗證連結或驗證碼
- 口語如「建立臨時信箱」、「temp mail」、「收驗證信」、「建一個測試信箱」
- E2E 測試流程中需要 disposable email

也可直接使用 slash command：`/temp-email`

#### 申請 API 金鑰

1. 前往 [https://rapidapi.com/Privatix/api/temp-mail](https://rapidapi.com/Privatix/api/temp-mail)
2. 登入或註冊 RapidAPI 帳號
3. 訂閱該 API（提供免費方案）
4. 進入 **Endpoints** 頁面，點選任一端點後在右側 **Header Parameters** 欄位可找到：
   - `X-RapidAPI-Key` — 即 `TempMail_RapidApiKey` 的值
5. `TempMail_Jwt` 為呼叫 API 回傳的 Bearer token，首次呼叫建立信箱時由 API 核發

#### 環境變數

| 變數 | 必要 | 說明 |
|---|---|---|
| `TempMail_Jwt` | 是 | API Bearer token（由 API 核發） |
| `TempMail_RapidApiKey` | 是 | RapidAPI 訂閱金鑰（來自 RapidAPI 後台） |

#### 環境變數設定

<details>
<summary><strong>Claude Code</strong></summary>

在 `~/.claude/settings.json` 中設定：

```json
{
  "env": {
    "TempMail_Jwt": "your-jwt-token",
    "TempMail_RapidApiKey": "your-rapidapi-key"
  }
}
```

</details>

<details>
<summary><strong>GitHub Copilot</strong></summary>

Copilot 無 settings.json 可設定環境變數，需透過系統環境變數提供。

**Windows** — 透過 PowerShell：

```powershell
[System.Environment]::SetEnvironmentVariable('TempMail_Jwt', 'your-jwt-token', 'User')
[System.Environment]::SetEnvironmentVariable('TempMail_RapidApiKey', 'your-rapidapi-key', 'User')
```

設定後需重啟 VS Code 使環境變數生效。

</details>

#### 功能涵蓋

- 建立臨時信箱（可自訂 local part 與存活時間）
- 列出收件匣中的所有郵件
- 讀取完整郵件內容（純文字 / HTML）
- 擷取驗證連結或數字驗證碼
- E2E 測試：輪詢等待驗證信到達（15 秒間隔，最多重試 3 次）

#### 注意事項

- 建立的信箱記錄會寫入當前工作目錄的 `temp-mails.md`
- 預設信箱存活時間為 300 秒
- 使用 RapidAPI 上的 [Temp Mail](https://rapidapi.com/Privatix/api/temp-mail) 服務
