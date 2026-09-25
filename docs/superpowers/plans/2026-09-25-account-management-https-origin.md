# 帳號管理頁面 HTTPS 網址變更與 Supabase 環境設定實作計畫

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 將「帳號管理」依賴之 Supabase Edge Function (`admin-users`) CORS 允許來源與郵件重新導向設定更新為正式 HTTPS 網址，排除跨來源連線阻擋問題。

**Architecture:** 依據設計規格方案 B，不修改前端與 Edge Function 核心程式碼，透過 Supabase CLI 將遠端專案 `nychrjrezpuxwuurpcvh` 的 Secrets (`ALLOWED_ORIGIN`、`INVITE_REDIRECT_URL`、`PASSWORD_RESET_REDIRECT_URL`) 更新為正式站點 HTTPS 網址，並重新部署 Edge Function 使設定即時生效。

**Tech Stack:** Supabase CLI, Deno, Supabase Edge Functions (`admin-users`), PowerShell / curl

## Global Constraints

- 遠端 Supabase 專案 ID：`nychrjrezpuxwuurpcvh`（YMCA社會教育學員資料庫）。
- `ALLOWED_ORIGIN` 嚴格設定值：`https://www.changhua.ymca.org.tw`（不得包含結尾斜線或子目錄路徑）。
- `INVITE_REDIRECT_URL` 嚴格設定值：`https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/`。
- `PASSWORD_RESET_REDIRECT_URL` 嚴格設定值：`https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/`。
- 不修改前端與 Edge Function 現有商業邏輯與授權機制。

---

### Task 1: 透過 Supabase CLI 更新遠端 Edge Function Secrets

**Files:**
- 無本地程式碼修改（配置遠端專案 Secrets）。

**Interfaces:**
- Consumes: Supabase CLI authenticated session & project ref `nychrjrezpuxwuurpcvh`
- Produces: 遠端環境變數 `ALLOWED_ORIGIN`, `INVITE_REDIRECT_URL`, `PASSWORD_RESET_REDIRECT_URL`

- [ ] **Step 1: 執行 Secrets 更新指令**

執行以下 PowerShell 指令將三個環境變數寫入遠端專案：

```powershell
npx supabase secrets set --project-ref nychrjrezpuxwuurpcvh ALLOWED_ORIGIN="https://www.changhua.ymca.org.tw" INVITE_REDIRECT_URL="https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/" PASSWORD_RESET_REDIRECT_URL="https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/"
```

- [ ] **Step 2: 驗證 Secrets 清單已更新**

執行以下指令確認變數已正確更新：

```powershell
npx supabase secrets list --project-ref nychrjrezpuxwuurpcvh
```

預期輸出：
- JSON / 清單中包含 `ALLOWED_ORIGIN`、`INVITE_REDIRECT_URL`、`PASSWORD_RESET_REDIRECT_URL`，且 `updated_at` 時間戳記為最新當前時間。

---

### Task 2: 重新部署 `admin-users` Edge Function

**Files:**
- 目標函式：`supabase/functions/admin-users/`

**Interfaces:**
- Consumes: `supabase/functions/admin-users/index.ts`
- Produces: 遠端 `admin-users` function 容器重啟並載入最新 Secrets

- [ ] **Step 1: 部署 Edge Function**

執行以下指令重新發布 `admin-users`：

```powershell
npx supabase functions deploy admin-users --project-ref nychrjrezpuxwuurpcvh
```

- [ ] **Step 2: 確認部署成功**

確認命令回傳代碼為 0，且輸出顯示：
`Deployed Function admin-users on project nychrjrezpuxwuurpcvh`

---

### Task 3: 驗證 CORS 預檢回應與遠端 Edge Function 回應標頭

**Files:**
- 無

**Interfaces:**
- Consumes: `https://nychrjrezpuxwuurpcvh.supabase.co/functions/v1/admin-users`
- Produces: HTTP preflight OPTIONS response headers

- [ ] **Step 1: 發送 CORS OPTIONS 預檢請求**

使用 `curl` 模擬來自 `https://www.changhua.ymca.org.tw` 的瀏覽器預檢請求：

```powershell
curl -i -X OPTIONS "https://nychrjrezpuxwuurpcvh.supabase.co/functions/v1/admin-users" `
  -H "Origin: https://www.changhua.ymca.org.tw" `
  -H "Access-Control-Request-Method: POST" `
  -H "Access-Control-Request-Headers: authorization, content-type"
```

- [ ] **Step 2: 檢查回應標頭**

預期檢查點：
1. HTTP 狀態碼為 `204 No Content` 或 `200 OK`。
2. 包含標頭 `access-control-allow-origin: https://www.changhua.ymca.org.tw`。
3. 包含標頭 `access-control-allow-credentials: true`。

---

### Task 4: 同步更新本機 `supabase/config.toml` 與建立提交

**Files:**
- 修改：`supabase/config.toml:159-163`

**Interfaces:**
- Consumes: 現有 `supabase/config.toml`
- Produces: 符合正式站 HTTPS 的輔助導向設定

- [ ] **Step 1: 在 `supabase/config.toml` 中新增 HTTPS 重新導向網址**

將 `additional_redirect_urls` 新增正式站 HTTPS 網址：

```toml
additional_redirect_urls = [
  "https://127.0.0.1:5173",
  "https://www.changhua.ymca.org.tw/**",
  "https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/**"
]
```

- [ ] **Step 2: 提交設定檔案變更**

```powershell
git add supabase/config.toml
git commit -m "chore(config): record https redirect urls in supabase config"
```

---

### Task 5: 提醒使用者核對 Supabase Auth 後台 URL Configuration

**Files:**
- 無

- [ ] **Step 1: 提供 Supabase Auth 後台核對步驟**

引導使用者確認 Supabase Dashboard -> Authentication -> URL Configuration 中的：
1. **Site URL**: `https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/`
2. **Redirect URLs**: 確保有 `https://www.changhua.ymca.org.tw/**`
