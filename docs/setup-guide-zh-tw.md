# Open Agents — 安裝與使用指南（繁體中文）

> **目錄**
> - [前置需求](#前置需求)
> - [儲存庫設定](#儲存庫設定)
> - [環境變數](#環境變數)
> - [PostgreSQL / Neon 資料庫](#postgresql--neon-資料庫)
> - [Vercel OAuth 整合](#vercel-oauth-整合)
> - [GitHub App 整合](#github-app-整合)
> - [本地開發](#本地開發)
> - [部署到 Vercel](#部署到-vercel)
> - [使用應用程式](#使用應用程式)
> - [常用指令速查](#常用指令速查)
> - [專案結構](#專案結構)
> - [疑難排解](#疑難排解)

---

## 前置需求

開始之前，請確認以下工具已安裝於你的機器：

| 工具 | 最低版本 | 安裝方式 |
|------|---------|---------|
| **Bun** | 1.2.14+ | `curl -fsSL https://bun.sh/install \| bash` |
| **Git** | 任意版本 | [git-scm.com](https://git-scm.com/) |
| **Node.js** | 20+（選用，僅供部分工具相容性） | [nodejs.org](https://nodejs.org/) |
| **Vercel CLI**（選用） | 最新版 | `bun add -g vercel` |

> ⚠️ 本專案**僅使用 Bun**。請**勿**使用 `npm`、`yarn` 或 `pnpm`，否則會產生錯誤的鎖定檔案，導致依賴安裝失敗。

---

## 儲存庫設定

### 1. Fork 儲存庫

前往 <https://github.com/vercel-labs/open-agents>，點擊 **Fork** 以建立你自己的副本。

### 2. Clone 你的 Fork

```bash
git clone https://github.com/<YOUR_USERNAME>/open-agents.git
cd open-agents
```

### 3. 安裝依賴

```bash
bun install
```

這會安裝所有定義於 `apps/*` 與 `packages/*` 的工作區套件。

---

## 環境變數

應用程式從 `apps/web/.env` 讀取設定。範本已提供，執行以下指令建立：

```bash
cp apps/web/.env.example apps/web/.env
```

接著開啟 `apps/web/.env`，依以下說明填入各項目。

### 最低啟動所需

```env
POSTGRES_URL=          # PostgreSQL 連線字串（參見 Neon 章節）
JWE_SECRET=            # 32 位元組的 URL-safe base64 字串
```

產生 `JWE_SECRET`：

```bash
openssl rand -base64 32 | tr '+/' '-_' | tr -d '=\n'
```

### 登入所需

```env
ENCRYPTION_KEY=                        # 32 位元組十六進位字串（64 個十六進位字元）
NEXT_PUBLIC_VERCEL_APP_CLIENT_ID=      # Vercel OAuth App 的 Client ID
VERCEL_APP_CLIENT_SECRET=              # Vercel OAuth App 的 Client Secret
```

產生 `ENCRYPTION_KEY`：

```bash
openssl rand -hex 32
```

### GitHub 儲存庫存取、Push 與 PR 所需

```env
NEXT_PUBLIC_GITHUB_CLIENT_ID=   # GitHub App Client ID
GITHUB_CLIENT_SECRET=           # GitHub App Client Secret
GITHUB_APP_ID=                  # GitHub App 數字 ID
GITHUB_APP_PRIVATE_KEY=         # PEM 內容（用 \n 取代換行）或 base64 編碼的 PEM
NEXT_PUBLIC_GITHUB_APP_SLUG=    # GitHub App 的 URL slug
GITHUB_WEBHOOK_SECRET=          # 你在建立 App 時自訂的 Webhook 密鑰
```

### 選用項目

```env
REDIS_URL=                                     # Redis，用於技能中繼資料快取
KV_URL=                                        # Vercel KV 替代方案
VERCEL_PROJECT_PRODUCTION_URL=                 # 正式環境的標準 URL
NEXT_PUBLIC_VERCEL_PROJECT_PRODUCTION_URL=     # 同上，對瀏覽器公開
VERCEL_SANDBOX_BASE_SNAPSHOT_ID=               # 覆寫預設沙盒快照
ELEVENLABS_API_KEY=                            # 語音轉文字功能
```

---

## PostgreSQL / Neon 資料庫

### 方案 A — 使用 Neon（推薦）

1. 前往 <https://neon.tech/> 建立免費帳號。
2. 建立新的 **Project** 與其中的 **Database**。
3. 複製連線字串（格式為 `postgresql://…`）。
4. 將其貼至 `.env` 中的 `POSTGRES_URL`。

> Neon 的**資料庫分支（Branching）**功能與 Vercel 整合：每個預覽部署會自動取得一個隔離的資料庫分支，確保預覽與正式資料完全不互通。

### 方案 B — 使用任意 PostgreSQL 伺服器

任何 PostgreSQL 14+ 均可使用，將 `POSTGRES_URL` 設為正確的連線字串即可。

### 資料庫遷移

遷移會在建置時（`bun run build`）**自動執行**。每次 Vercel 部署（預覽或正式）都會對對應的資料庫套用待執行的遷移，不需要在正式環境手動執行。

本地開發時執行：

```bash
bun run --cwd apps/web db:generate   # 修改 schema.ts 後產生遷移檔
bun run --cwd apps/web db:migrate    # 在本地套用待執行遷移
```

> 每次修改 `schema.ts` 時，請將產生的 `.sql` 遷移檔一起提交（commit）。

---

## Vercel OAuth 整合

Open Agents 使用 **Vercel OAuth** 進行使用者登入。

### 1. 建立 Vercel OAuth App

1. 前往 <https://vercel.com/account/tokens> → **Integrations** → **OAuth Apps** → **Create**。
2. 設定 **Callback URL**：
   - 正式環境：`https://YOUR_DOMAIN/api/auth/vercel/callback`
   - 本地開發：`http://localhost:3000/api/auth/vercel/callback`

### 2. 複製憑證

```env
NEXT_PUBLIC_VERCEL_APP_CLIENT_ID=<Client ID>
VERCEL_APP_CLIENT_SECRET=<Client Secret>
```

---

## GitHub App 整合

Open Agents **不需要**獨立的 GitHub OAuth App。它使用 **GitHub App 的使用者授權流程**同時處理儲存庫存取與使用者身份識別。

### 1. 建立 GitHub App

前往 <https://github.com/settings/apps/new> 進行設定：

| 設定項目 | 值 |
|---------|---|
| **Homepage URL** | `https://YOUR_DOMAIN`（本地開發填 `http://localhost:3000`） |
| **Callback URL** | `https://YOUR_DOMAIN/api/github/app/callback` |
| **Setup URL** | `https://YOUR_DOMAIN/api/github/app/callback` |
| **Webhook URL** | `https://YOUR_DOMAIN/api/github/webhook` |
| **Webhook Secret** | 自訂的隨機密鑰 |
| 安裝時要求使用者授權（Request user authorization during install） | ✅ 啟用 |
| 將 App 設為公開（Make App public） | ✅ 推薦啟用（以支援組織安裝） |

#### 所需權限

- **Repository**：Contents（讀寫）、Pull requests（讀寫）、Metadata（唯讀）
- **Account**：Email addresses（唯讀）

### 2. 產生私鑰

在 App 設定頁面，捲動至 **Private Keys**，點擊 **Generate a private key**，下載 `.pem` 檔案。

將私鑰編碼為環境變數：

```bash
# 方式一 — base64（推薦）
base64 -w 0 your-app.YYYY-MM-DD.private-key.pem

# 方式二 — 跳脫換行字元（將 PEM 中的換行替換為 \n 後貼上）
```

### 3. 複製憑證

```env
NEXT_PUBLIC_GITHUB_CLIENT_ID=<App Client ID>
GITHUB_CLIENT_SECRET=<App Client Secret>
GITHUB_APP_ID=<數字 App ID>
GITHUB_APP_PRIVATE_KEY=<base64 編碼的 PEM 或跳脫換行的 PEM>
NEXT_PUBLIC_GITHUB_APP_SLUG=<App 設定頁顯示的 URL slug>
GITHUB_WEBHOOK_SECRET=<你自訂的 Webhook 密鑰>
```

---

## 本地開發

```bash
# 1. 安裝依賴（若尚未執行）
bun install

# 2. 複製並填寫環境變數檔案
cp apps/web/.env.example apps/web/.env
# 用你的值編輯 apps/web/.env

# 3. 啟動 Web 應用程式
bun run web
```

應用程式現在可在 <http://localhost:3000> 存取。

### 品質檢查

提交任何變更前請執行：

```bash
bun run ci          # 格式檢查 + 程式碼規範 + 型別檢查 + 測試 + 資料庫 schema 驗證
bun run fix         # 自動修正格式與程式碼規範問題
bun run typecheck   # TypeScript 型別檢查所有套件
bun test            # 執行所有測試
```

---

## 部署到 Vercel

### 逐步說明

1. **Fork** 儲存庫並推送到你的 GitHub 帳號。
2. 前往 <https://vercel.com/new> → **Import** 你的 Fork。
3. 在 **Environment Variables** 中至少加入：

   ```env
   POSTGRES_URL=
   JWE_SECRET=
   ENCRYPTION_KEY=
   ```

4. 點擊 **Deploy**。第一次部署完成後，記下你的正式環境 URL。
5. 加入 Vercel OAuth 憑證（`NEXT_PUBLIC_VERCEL_APP_CLIENT_ID`、`VERCEL_APP_CLIENT_SECRET`）並重新部署。
6. 加入 GitHub App 憑證並重新部署。
7. （選用）加入 `REDIS_URL` / `KV_URL`、`VERCEL_PROJECT_PRODUCTION_URL` 與 `ELEVENLABS_API_KEY`。

> 若在 import 專案時加入 Neon 整合，Vercel 可自動佈建 `POSTGRES_URL`。

### 一鍵部署按鈕

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?project-name=open-agents&repository-name=open-agents&repository-url=https%3A%2F%2Fgithub.com%2Fvercel-labs%2Fopen-agents)

---

## 使用應用程式

### 登入

1. 開啟部署後的 URL（或本地的 `http://localhost:3000`）。
2. 點擊 **Sign in with Vercel**，授權 OAuth App。

### 連接你的 GitHub 帳號

1. 登入後，前往 **Settings** → **Integrations**。
2. 點擊 **Install GitHub App**，選擇要讓 Agent 存取的儲存庫或組織。

### 開始新的工作階段（與 Agent 對話）

1. 在首頁點擊 **New session**。
2. 輸入你想讓 Agent 執行的提示詞，例如：

   ```
   在這個儲存庫的 Express 應用程式中新增 /health 端點，並撰寫相對應的測試。
   ```

3. 選擇性地選取 Agent 應操作的 **GitHub 儲存庫**。
4. 按下 **Enter** 或點擊 **Send**。

### Agent 能力一覽

| 工具 | 功能說明 |
|------|---------|
| **file** | 在沙盒中讀取、寫入、搜尋檔案 |
| **shell** | 執行 Shell 指令（`npm install`、`git` 等） |
| **search** | 語意與正則表達式程式碼搜尋 |
| **task** | 派生子 Agent 進行平行處理 |
| **skill** | 從技能登錄檔載入可重複使用的技能提示 |
| **web** | 擷取網頁內容（文件、Issues 等） |

### 工作階段分享

複製任何工作階段的 URL 並分享。接收方可以**唯讀**瀏覽該工作階段（訊息與檔案差異），但無法發送新訊息。

### 自動 Commit / 自動 PR

在工作階段設定中啟用 **Auto-commit** 或 **Auto-PR**，Agent 在成功執行後會自動推送分支並建立 Pull Request。

### 語音輸入（選用）

若設定了 `ELEVENLABS_API_KEY`，聊天輸入框會出現麥克風按鈕，點擊即可錄製語音提示。

---

## 常用指令速查

```bash
bun run web                          # 啟動本地開發伺服器（http://localhost:3000）
bun run ci                           # 完整 CI 檢查（格式 + 程式碼規範 + 型別 + 測試 + 資料庫）
bun run check                        # 僅進行程式碼規範與格式檢查
bun run fix                          # 自動修正程式碼規範與格式
bun run typecheck                    # TypeScript 型別檢查所有套件
bun test                             # 執行所有測試
bun test path/to/file.test.ts        # 執行特定測試檔案
bun run --cwd apps/web db:generate   # 產生 Drizzle 遷移檔
bun run --cwd apps/web db:migrate    # 在本地套用遷移
bun run --cwd apps/web db:check      # 驗證 schema 與遷移是否一致
bun run sandbox:snapshot-base        # 在 Vercel 上刷新基礎沙盒快照
```

---

## 專案結構

```text
open-agents/
├── apps/
│   └── web/               # Next.js 應用程式 — 聊天 UI、驗證、工作流程、API 路由
│       ├── app/           # App Router 頁面與版面配置
│       ├── lib/           # 伺服器工具、資料庫 Schema（Drizzle）、驗證輔助函式
│       └── .env.example   # 環境變數範本
├── packages/
│   ├── agent/             # Agent 執行環境 — 工具、子 Agent、技能載入器
│   ├── sandbox/           # 沙盒抽象層與 Vercel 沙盒整合
│   ├── shared/            # 共用型別與工具函式
│   └── tsconfig/          # 共用 TypeScript 設定
├── docs/
│   └── agents/            # 架構、程式碼風格、與 Lessons Learned 文件
├── scripts/               # 輔助腳本（快照刷新、隔離測試執行器）
├── turbo.json             # Turborepo Pipeline 設定
├── package.json           # 根工作區 + 腳本
└── bun.lock               # Bun 鎖定檔（請一起提交）
```

---

## 疑難排解

| 問題 | 解決方式 |
|------|---------|
| `bun install` 失敗 | 確認 Bun 版本 ≥ 1.2.14。執行 `bun upgrade`。 |
| 應用程式啟動但無法登入 | 檢查 `NEXT_PUBLIC_VERCEL_APP_CLIENT_ID` 與 `VERCEL_APP_CLIENT_SECRET`。確認 OAuth Callback URL 完全一致。 |
| GitHub 功能無法使用 | 確認所有 `GITHUB_*` 環境變數已設定，且 GitHub App 已安裝在至少一個儲存庫。 |
| 資料庫遷移失敗 | 確認 `POSTGRES_URL` 正確，且資料庫使用者具有 `CREATE TABLE` 權限。 |
| 沙盒無法啟動 | `VERCEL_SANDBOX_BASE_SNAPSHOT_ID` 可能需要更新。執行 `bun run sandbox:snapshot-base`。 |
| 拉取最新程式碼後出現型別錯誤 | 先執行 `bun install`，再執行 `bun run typecheck`。 |
