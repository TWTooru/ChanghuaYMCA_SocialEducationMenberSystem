# 帳號管理頁面 HTTPS 網址變更與 Supabase 環境設定規格

日期：2026-09-25  
狀態：已完成對話設計核准，待書面規格審閱

## 背景與問題診斷

彰化 YMCA 社教學員資訊管理系統的前端站點網址由原本的 HTTP 調整為 HTTPS：
- 正式完整站點網址：`https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/`
- 瀏覽器發送請求之 Origin：`https://www.changhua.ymca.org.tw`

在系統各項功能中，「數據概覽」、「學員清單與查詢」、「歷年選課歷程」、「課程管理」等頁籤皆透過 `@supabase/supabase-js` 客戶端直接以 HTTPS 呼叫 Supabase REST API 與 RPC，不受站台網址通訊協定轉換影響。

然而「帳號管理」頁（`AdminAccountPage`）涉及身分治理特權（邀請帳號、指派角色、寄送密碼重設、永久刪除），為落實授權隔離，由後端 Supabase Edge Function（`admin-users`）統一承接。當前端以 HTTPS 網址開啟時，瀏覽器對 Edge Function 發送帶有 `Origin: https://www.changhua.ymca.org.tw` 的 Preflight 及 POST 請求；若 Edge Function 的 CORS 設定仍為 HTTP 或本機預設值，瀏覽器將強制阻擋跨來源請求，導致帳號管理頁面顯示「帳號管理操作失敗，請稍後再試。」

此外，邀請使用者（`inviteUser`）與重設密碼（`sendPasswordReset`）信件中包含的導向網址若未同步為 HTTPS，使用者點擊信件連結時將被重導向至舊的 HTTP 網址或遭 Supabase Auth 重新導向白名單拒絕。

## 目標

1. 透過 Supabase CLI 與專案後台設定，將 Edge Function `admin-users` 之 `ALLOWED_ORIGIN` 設為 `https://www.changhua.ymca.org.tw`。
2. 將信件重新導向環境變數 `INVITE_REDIRECT_URL` 與 `PASSWORD_RESET_REDIRECT_URL` 設為 `https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/`。
3. 確保 Supabase Auth 之 `site_url` 與 `additional_redirect_urls` 允許上述 HTTPS 站點。
4. 不修改現有前端與 Edge Function 核心程式碼，落實純環境組態調整（方案 B）。
5. 提供完整檢查與驗證流程，確保帳號管理頁面正常載入且無 CORS 阻擋。

## 非目標

- 不更動 `supabase/functions/admin-users/index.ts` 中的業務邏輯與驗證機制。
- 不改動前端 UI 元件或狀態管理（`useAdminUsers`、`AdminAccountPage`）。
- 不替換現有 Supabase 資料庫 schema 或 trigger。

## 詳細設定規範

### 1. Edge Function Secrets（環境變數）

Edge Function 在 Deno 環境中執行時，由 `requiredEnv` 與 `createRuntimeHandler` 讀取系統環境變數。需在遠端 Supabase 專案中寫入以下 Secrets：

| 變數名稱 | 設定值 | 說明與安全規範 |
| :--- | :--- | :--- |
| `ALLOWED_ORIGIN` | `https://www.changhua.ymca.org.tw` | **嚴格規範**：只包含協定與網域名稱，**絕不可**包含結尾斜線（`/`）或子目錄路徑（`/YMCA_...`）。瀏覽器比對 Origin 時採精準比對。 |
| `INVITE_REDIRECT_URL` | `https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/` | 新受邀者點擊邀請信連結後的登陸頁面。 |
| `PASSWORD_RESET_REDIRECT_URL` | `https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/` | 使用者點擊重設密碼信連結後的登陸頁面。 |

### 2. Supabase Auth URL Configuration（驗證重新導向白名單）

於 Supabase 專案後台之 **Authentication -> URL Configuration** 確保以下設定生效：

- **Site URL**：`https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/`
- **Redirect URLs**：
  - `https://www.changhua.ymca.org.tw/**`
  - `https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/**`

### 3. 本機組態同步（`supabase/config.toml`）

為避免本機開發環境與設定 drift，在 `supabase/config.toml` 中同步備份站點與重新導向設定：
- `auth.site_url = "https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/"`（可並存於本機組態或保留註解）
- `auth.additional_redirect_urls` 納入正式 HTTPS 網址。

## 執行步驟（Supabase CLI）

1. **檢查 Supabase CLI 登入與專案綁定狀態**：
   - 執行 `npx supabase projects list` 檢視已綁定之專案 ID（例如 `hkmbzvyxynpqsxfsinht` 或 `nychrjrezpuxwuurpcvh`）。
   - 若未綁定，執行 `npx supabase link --project-ref <PROJECT_ID>`。
2. **推送 Edge Function Secrets**：
   - 執行以下指令設定遠端環境變數：
     ```bash
     npx supabase secrets set ALLOWED_ORIGIN="https://www.changhua.ymca.org.tw" INVITE_REDIRECT_URL="https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/" PASSWORD_RESET_REDIRECT_URL="https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/"
     ```
   - 執行 `npx supabase secrets list` 確認三個變數已成功寫入。
3. **重新部署 Edge Function（如需使 Secrets 即時生效）**：
   - 執行 `npx supabase functions deploy admin-users` 確保 Edge Function 重新載入最新環境變數。
4. **檢查 / 提示 Auth URL Configuration**：
   - 指引使用者確認 Supabase 控制台上的 Site URL 與 Redirect URLs 設定。

## 驗證方式與驗收標準

1. **CORS 預檢與請求驗證**：
   - 使用瀏覽器或 `curl` 對 Edge Function 發送 OPTIONS 預檢請求：
     - Request Header: `Origin: https://www.changhua.ymca.org.tw`
     - Response Header 必須包含：
       - `Access-Control-Allow-Origin: https://www.changhua.ymca.org.tw`
       - `Access-Control-Allow-Credentials: true`
2. **帳號管理頁面載入**：
   - 以管理員身分登入 `https://www.changhua.ymca.org.tw/YMCA_SocialEducationMenberSystem/`。
   - 切換至「帳號管理」頁籤，應正常顯示帳號清單與稽核紀錄，無紅字錯誤提示。
3. **邀請與密碼重設**：
   - 觸發重寄邀請或密碼重設，檢查信件連結之重導向位址正確帶有 `https://...` 且可跳轉回系統完成認證。
