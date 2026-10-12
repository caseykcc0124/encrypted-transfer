# IntentOS：接手 Agent 安裝指南

目標：在接收者的 Windows 電腦安裝 IntentOS 0.1.0-preview.1，完成啟動與驗證。網站可開啟、Orca 服務連線、Orchestrator 能回覆，是三項不同的驗收結果，請分別回報。

## 1. 下載與解壓

- 適用平台：Windows 10／11 x64。
- [直接下載指定版本的密碼 ZIP](https://raw.githubusercontent.com/caseykcc0124/encrypted-transfer/edcbd3dcd5ad2472927bbf26cf9313cd6ade4610/IntentOS-0.1.0-preview.1-Windows-x64-20261012-113659.zip)。下載不需 GitHub 帳號或 gh。
- SHA256：`08b85c454b641b6b5ee5f555b25dc716471d40c3256e14c34a7e98c2838d6664`。
- 大小：11,750,573 bytes。

不要使用 GitHub 的 Code → Download ZIP；那是儲存庫外層封裝。請下載上述指定檔案，以 SHA256 比對完整性後才執行內容：

```powershell
Get-FileHash -LiteralPath '實際下載位置\IntentOS-release.zip' -Algorithm SHA256
```

此檔使用標準 AES-256 密碼 ZIP。請以 7-Zip 或 WinRAR 完整解壓到新的資料夾，不要直接在壓縮檔內執行程式。密碼由擁有者另行提供，讓使用者在解壓軟體密碼欄輸入；不要寫入命令列、聊天、日誌或 GitHub。Windows 內建解壓功能不一定支援此格式。

解壓後應得到：

```text
IntentOS/
  Install.cmd  Start.cmd  Diagnose.cmd
  install.ps1  start.ps1  diagnose.ps1
  RELEASE.json  README.txt
  app/intentos/  app/web/  app/docs/  app/tools/
  python/python.exe  python/python313._pth  python/LICENSE.txt
```

這是乾淨的新使用者版本，不含作者的意圖、對話、材料、憑證、終端綁定或 Git 歷史。不要從作者電腦複製帳號或終端 ID。

## 2. 盤點前提

已內含 Python 3.13.16。不要另裝 Python、pip 套件，或改用系統 Python 執行此發行版。

Agent 功能另需：

1. 可用的 Git for Windows，服務工作區與 Git 工作樹任務會用到。
2. 已安裝且執行中的 Orca。
3. 使用者選定的 Agent Harness 已安裝、登入，並有可用模型及額度。

先盤點既有環境，避免重複安裝。可用 `git --version` 檢查 Git。Orca 不一定在 PATH，預設原生 CLI 位於 `%LOCALAPPDATA%\Programs\orca\resources\bin\orca.exe`。查證實際路徑後執行 `status --json`，確認 runtime 已就緒。不要將 `.cmd` 包裝器當成原生 CLI。

若 Orca 位於其他位置，在啟動服務的同一程序環境設定：

```powershell
$env:INTENTOS_ORCA_EXE = '實際位置\orca.exe'
& $env:INTENTOS_ORCA_EXE status --json
```

缺少前提時先提出可行的安裝／設定方案，處理一般工程問題。需要登入、付費、信任資料夾或敏感權限時，才請使用者完成必要決定。不要重啟既有業務 Agent 來修復安裝。

## 3. 安裝

一般使用者雙擊 `Install.cmd`。預設安裝到 `%LOCALAPPDATA%\IntentOS\releases\0.1.0-preview.1`，並嘗試建立桌面與開始功能表捷徑，不需管理員權限。

Agent 可用下列指令；請先替換實際解壓位置：

```powershell
$release = '實際解壓位置\IntentOS'
& powershell.exe -NoProfile -ExecutionPolicy Bypass -File (Join-Path $release 'install.ps1')
if ($LASTEXITCODE -ne 0) { throw '安裝失敗，請檢查輸出。' }
$installed = Join-Path $env:LOCALAPPDATA 'IntentOS\releases\0.1.0-preview.1'
```

ExecutionPolicy Bypass 只作用於本次程序，不要更改永久執行政策。自訂位置可使用 `install.ps1 -Destination '指定路徑' -SkipShortcuts`，後續同步修改 `$installed`。

安裝程式會驗證 RELEASE.json 檔案雜湊，拒絕覆寫內容不同的既有檔案。失敗時調查原因或改用新的目的地，不要刪除舊版本或使用者資料來硬過關。

## 4. 啟動

先做不啟動服務的檢查：

```powershell
& powershell.exe -NoProfile -ExecutionPolicy Bypass -File (Join-Path $installed 'start.ps1') -CheckOnly
if ($LASTEXITCODE -ne 0) { throw '啟動前檢查失敗。' }
```

一般使用者點桌面 IntentOS 捷徑，或雙擊安裝目錄的 `Start.cmd`。服務視窗需保持開啟。

Agent 也可在支援保留長時間程序的終端工具執行：

```powershell
& powershell.exe -NoProfile -ExecutionPolicy Bypass -File (Join-Path $installed 'start.ps1') -NoBrowser
```

這條指令會持續執行到服務停止；不要讓工具逾時後誤殺服務。若使用背景啟動，使用隱藏視窗、保留 stdout／stderr 日誌與本次程序身分。不要把未知的既有程序當成本次建立的服務。

預設網址為 `http://127.0.0.1:8767/dashboard.html`。若通訊埠被其他服務占用，不要強制終止它；可明確改用 `start.ps1 -Port 8768 -NoBrowser`。相同版本、相同資料目錄的服務已執行時，啟動器會沿用原服務，第二次啟動的程序可能直接結束。

## 5. 驗證

預設使用者資料位於 `%LOCALAPPDATA%\IntentOS\data`，與版本程式分開。升級或重裝時保留資料。不要自動遷入作者的業務資料。

啟動後有限次輪詢服務，例如每秒一次、最多 60 次；逾時就看日誌，不要無限等待。以下查詢範例在服務就緒後執行；若改過 Port，也需修改 `$base`：

```powershell
$base = 'http://127.0.0.1:8767'
$identity = Invoke-RestMethod -Uri ($base + '/api/desktop-instance') -TimeoutSec 5
$expectedData = [IO.Path]::GetFullPath((Join-Path $env:LOCALAPPDATA 'IntentOS\data'))
if ($identity.product -ne 'IntentOS' -or $identity.data_root -ne $expectedData -or $identity.app_root -ne (Join-Path $installed 'app')) {
    throw '服務身分、資料目錄或程式位置不符。'
}
Invoke-WebRequest -UseBasicParsing -Uri ($base + '/dashboard.html') -TimeoutSec 5 | Select-Object StatusCode
Invoke-RestMethod -Uri ($base + '/api/intents') -TimeoutSec 5
& powershell.exe -NoProfile -ExecutionPolicy Bypass -File (Join-Path $installed 'diagnose.ps1') -Json
if ($LASTEXITCODE -ne 0) { throw '診斷發現異常。' }
```

自訂 DataRoot 時，預期路徑也需同步修改。全新資料的意圖清單為空是正常結果。

分開驗收：

- **網站：**頁面與 API 可用，服務身分及資料位置正確，Diagnose 完整性檢查通過。
- **Orca 服務：**啟動日誌沒有缺 Git／Orca 未就緒訊息；查核資料目錄 `service-pane.json` 記錄的專用終端，使用 Orca 原生工具確認它仍存在、可寫且已連線。這是後端純 Shell 服務身分，不代表 Orchestrator 已開始任務。
- **Orchestrator：**只有使用者同意測試意圖後，才在 UI 建立小型測試意圖，確認實際連線、訊息回覆與任務狀態。不可用靜態頁面、轉圈動畫或已建立終端來替代實際回覆。

各 Harness 的登入、模型與額度要在接收者環境驗證，不能由作者電腦的測試結果推定。

## 6. 故障處理與回報

- 雜湊不符：停止，不執行程式。
- 缺 Git／Orca 未執行：處理前提後重啟此次 IntentOS，保留其他業務程序。
- 服務終端建立結果不明：保留 `service-pane.json` 與日誌，查核 Orca 原生憑據；不要刪檔後反覆建立終端。
- 原服務終端離線：查核／恢復該終端，不盲目改綁其他 Agent。
- 要求登入或信任：請使用者完成必要授權後重新驗證。

最後用繁體中文回報：版本、程式位置、資料位置、網址、網站驗收、Orca 服務連線、是否真的測試過任務，以及尚需使用者處理的事項。不要宣稱未驗證的功能已可用。
