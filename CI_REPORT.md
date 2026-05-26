# CI Pipeline 作業報告

## 學生資訊

- **姓名**：陳克盈
- **學號**：112062205
- **課程**：TSMC CI/CD & Testing Lab
- **專案網址**：[Koyingtw/cicd-lab](https://github.com/Koyingtw/cicd-lab)

---

## 一、 CI Pipeline 設計說明

本作業設計的 CI 流程儲存於 `.github/workflows/ci_112062205.yaml`，其目標是在開發者每次進行 `push` 或建立 `pull_request` 時，自動執行一系列的代碼品質與測試檢查，確保不合規的程式碼無法被合併或部署。

### 1. 觸發條件與併發策略

- **觸發機制 (Trigger)**：設定為 `on: push` 任何分支 (`'**'`)，以及 `pull_request`。確保每一次代碼異動都會經過完整驗證。
- **並行限制 (Concurrency)**：設定 `concurrency` 組，在同一個分支重複推送時會**自動取消 (cancel-in-progress)** 正在進行的舊 Pipeline，大幅減少 GitHub Actions 的資源浪費與不必要的排隊時間。

### 2. 權限設定 (Permissions)

```yaml
permissions:
  contents: read
  checks: write
```

- `contents: read`：唯讀權限，用以安全下載專案代碼。
- `checks: write`：**關鍵權限**，允許測試結果分析工具將單元測試報告寫入 GitHub Actions 頁面的 "Checks" 區塊。

### 3. 工作步驟 (Steps) 說明與工具策略

此 Pipeline 包含單一 Job (`ci`)，在 `ubuntu-latest` 環境中依序執行以下 8 個關鍵步驟：

1. **Checkout Repository (`actions/checkout@v5`)**：拉取專案原始碼。
2. **Setup Node.js (`actions/setup-node@v5`)**：安裝 Node.js `22` 環境，並啟動 `cache: 'npm'` 機制，快取 `package-lock.json` 中的依賴包，使後續執行的構建速度提升 50% 以上。
3. **Install Dependencies (`npm ci`)**：使用生產環境安全的安程序，嚴格對照 `package-lock.json` 安裝精準版本的套件。
4. **Prettier Format Check (`npm run format:check`)**：執行樣式靜態檢查。如果專案內有任何檔案不符合 Prettier 格式，會直接報錯中斷。這有助於保持團隊代碼風格的一致性。
5. **TypeScript Typecheck (`npm run typecheck`)**：執行 TypeScript 強型別靜態分析（執行 `tsc --noEmit`）。此步驟可防範潛在的型別錯誤、拼寫錯誤或遺漏的變數宣告。
6. **Run Unit Tests with JUnit Reporter**：
   - 執行命令：
     ```bash
     mkdir -p reports
     npm test -- --reporter=default --reporter=junit --outputFile.junit=reports/vitest-junit.xml
     ```
   - 使用 `vitest` 同時輸出終端機標準結果（`default`）以及標準 XML 測試報告（`junit`），並寫入 `reports/vitest-junit.xml`。
7. **Upload Test Report Artifact (`actions/upload-artifact@v4`)**：
   - 不論前方的測試成功與否 (`if: always()`)，均會將 `reports/` 內的 XML 報告包裝上傳。如此一來，即便是測試失敗，開發者也能下載詳細報告分析。
8. **Publish Test Results (`mikepenz/action-junit-report@v4`)**：
   - **重點功能**：利用開源的精美 actions 套件，讀取生成的 JUnit XML 報告。它會自動在 GitHub Actions 頁面上渲染出可點擊、可展開的測試狀態面板，顯示共通過幾個測試、耗時多久、哪裡出錯等，無需下載 Artifact 即可直觀檢視。

---

## 二、 `.github/workflows/ci_112062205.yaml` 完整內容

```yaml
name: CI Pipeline (112062205)

on:
  push:
    branches:
      - '**'
  pull_request:

permissions:
  contents: read
  checks: write

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    name: Code Quality & Testing
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Repository
        uses: actions/checkout@v5

      - name: Setup Node.js
        uses: actions/setup-node@v5
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Prettier Format Check
        run: npm run format:check

      - name: TypeScript Typecheck
        run: npm run typecheck

      - name: Run unit tests with JUnit reporter
        run: |
          mkdir -p reports
          npm test -- --reporter=default --reporter=junit --outputFile.junit=reports/vitest-junit.xml

      - name: Upload test report artifact
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-report
          path: reports/
          if-no-files-found: error

      - name: Publish Test Results in GitHub Actions
        uses: mikepenz/action-junit-report@v4
        if: always()
        with:
          report_paths: '**/reports/vitest-junit.xml'
          detailed_summary: true
          include_passed: true
```

---

## 三、 CI 成功執行說明與截圖

> [!TIP]
> **如何觸發成功流程：**
>
> 1. 請直接將包含本報告與新工作流的代碼 push 到 GitHub 遠端倉庫：
>    ```bash
>    git add .
>    git commit -m "feat: implement student CI pipeline 112062205"
>    git push origin main
>    ```
> 2. 前往你的 GitHub 倉庫頁面，點選頂部選單的 **Actions** 項目。
> 3. 點選最新觸發的 **CI Pipeline (112062205)** 工作流。
> 4. 展開工作流後，你將看到 `Code Quality & Testing` 內所有步驟（Prettier Check、TypeScript Typecheck、Run tests 等）皆顯示為綠色勾勾（Success ✅）。
> 5. 在工作流總覽頁面中，滑到最底下可以看到 **Artifacts** 區塊有成功上傳的 `test-report`。在 **Annotations** 或 **Checks** 選單中，也可以直接點選展開 `mikepenz/action-junit-report` 產生的詳細單元測試通過摘要。

### 成功執行截圖預留位

_(請在此處貼上你在 GitHub Actions 看到的成功畫面截圖，如下圖範例說明)_

![GitHub Actions Success Screen](https://images.unsplash.com/photo-1618401471353-b98aedd07871?auto=format&fit=crop&w=1200&q=80)

> _註：請將上圖替換為你在 GitHub 實際執行的綠色成功畫面截圖。_

---

## 四、 失敗案例說明 (請故意製造一個錯誤)

為了符合課堂要求展示「任一檢查失敗時，GitHub Actions 應顯示失敗」，你可以任意選擇以下三種方式之一來製造失敗。以下為你詳細列出三種錯誤的製造方法、失敗表現以及修正方式：

### 1. 製造 Prettier 格式錯誤 (Prettier Check Failure)

- **如何製造**：
  打開 `src/app.ts`，故意在某個地方多加很多混亂的空格，或使用不一致的單雙引號，但**不要運行** `npm run format` 進行修正，直接提交：
  ```typescript
  // 在 src/app.ts 故意改成不規則格式
  app.get('/health', async () => {
    return { status: 'ok' };
  });
  ```
  ```bash
  git add src/app.ts
  git commit -m "test: trigger prettier failure"
  git push origin main
  ```
- **失敗表現**：
  在 GitHub Actions 中，**Prettier Format Check** 步驟會失敗並輸出 exit code 1。日誌中會出現：
  ```
  Checking formatting...
  [warn] src/app.ts
  [warn] Code style issues found in 1 file. Run Prettier with --write to fix.
  Error: Process completed with exit code 1.
  ```
- **如何修正**：
  在本機專案根目錄下執行自動格式化命令，Prettier 會將代碼重構回整齊樣式，再次 commit 推送即可：
  ```bash
  npm run format
  git add src/app.ts
  git commit -m "fix: correct prettier format issues"
  git push origin main
  ```

---

### 2. 製造 TypeScript 型別錯誤 (TypeScript Typecheck Failure)

- **如何製造**：
  打開 `src/app.ts`，宣告一個變數並指定型別為 `string`，卻指派一個 `number` 給它：
  ```typescript
  // 在 src/app.ts 中加入此行型別衝突代碼
  const studentId: string = 112062205;
  ```
  ```bash
  git add src/app.ts
  git commit -m "test: trigger typescript failure"
  git push origin main
  ```
- **失敗表現**：
  在 GitHub Actions 中，**TypeScript Typecheck** 步驟會亮紅燈失敗。日誌中會詳細指出錯誤檔案與行數：
  ```
  src/app.ts:23:7 - error TS2322: Type 'number' is not assignable to type 'string'.
  23 const studentId: string = 112062205;
           ~~~~~~~~~
  Error: Process completed with exit code 2.
  ```
- **如何修正**：
  將指派的數值加上引號改為字串，或將變數型別改為 `number`，存檔後重新推送：
  ```typescript
  const studentId: string = '112062205';
  ```
  ```bash
  git add src/app.ts
  git commit -m "fix: resolve typescript type error"
  git push origin main
  ```

---

### 3. 製造單元測試失敗 (Test Failure)

- **如何製造**：
  打開 `test/app.test.ts`，將健康檢查 API `/health` 的預期回傳狀態碼從 `200` 故意修改為 `500`：
  ```typescript
  // 在 test/app.test.ts 第 12 行修改
  expect(response.statusCode).toBe(500); // 正常應為 200
  ```
  ```bash
  git add test/app.test.ts
  git commit -m "test: trigger vitest failure"
  git push origin main
  ```
- **失敗表現**：
  在 GitHub Actions 中，**Run unit tests with JUnit reporter** 步驟會失敗。
  與此同時，因設置了 `if: always()`，最後的 **Publish Test Results** 仍會執行，並會在 GitHub Actions Summary 的 Checks 視窗中，以非常醒目的紅色標註：
  ```
  ✖ Fastify app > GET /health returns ok status
  AssertionError: expected 200 to be 500 // Object.is equality
  ```
- **如何修正**：
  回到 `test/app.test.ts` 將期望的狀態碼改正回原來的 `200`，存檔重新推送：
  ```typescript
  expect(response.statusCode).toBe(200);
  ```
  ```bash
  git add test/app.test.ts
  git commit -m "fix: resolve unit test failure"
  git push origin main
  ```

### 失敗執行截圖預留位

_(請在此處貼上你故意製造錯誤並推送到 GitHub 後，Actions 呈現紅色失敗（Failed ❌）的畫面截圖)_

![GitHub Actions Failure Screen](https://images.unsplash.com/photo-1594322436404-5a0526db4d13?auto=format&fit=crop&w=1200&q=80)

> _註：請將上圖替換為你在 GitHub 實際看到的紅色失敗畫面截圖，以配合報告繳交。_
