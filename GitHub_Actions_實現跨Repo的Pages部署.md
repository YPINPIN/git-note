# GitHub Actions Workflow 實現跨 repository 的 GitHub Pages 部署

## 1. 產生 Personal Access Token (PAT) -登入你的 GitHub 帳號。

- 點擊右上角你的大頭貼，選擇「Settings」。

- 在左側選單找到「Developer settings」。

- 進入「Personal access tokens」 > 「Tokens (classic)」。

- 點選「Generate new token」>「Generate new token (classic)」。

- 輸入辨識名稱（如：GitHub Actions Deploy Token）。

- 選擇過期期限，可依需求選擇，例如 30 天、90 天或自定義。

- 在「Select scopes」區域勾選：

  - repo（包括寫入權限）

  - 或更細分的 repo:status、repo_deployment、public_repo，視目標 repo 權限需求調整。

- 按「Generate token」。

- 這時會顯示產生的 token 字串，請馬上複製保存，之後無法再看到。

## 2. 新增 Secret 到正式 Repository

- 回到你的正式 `repository` 頁面。

- 點擊「Settings」（專案右上齒輪）。

- 左邊選單點「Secrets and variables」>「Actions」。

- 點「New repository secret」。

- 在「Name」欄輸入：PERSONAL_ACCESS_TOKEN

- 在「Value」欄貼上剛剛複製的 Token 字串。

- 點「Add secret」完成。

## 3. Github 上新增對應 preview 的 repository

記得先開好對應 preview 的 `repository`。

## 4. 新增 workflow yaml 文件

- 在你的專案根目錄底下，建立一個 `.github` 資料夾，裡面再建立 `workflows` 資料夾。

```
your-project/
├── .github/
│   └── workflows/
```

- 在 `workflows` 資料夾中新增一個以 `.yml` 或 `.yaml` 為副檔名的檔案，例如 `deploy.yml`。

  使用說明：

  - 將 6a. Deploy develop 的 `your-username/your-repo-preview` 改成你的目標 `repository` 名稱。

  - 當分支 `develop` 或 `main` 有新的推送，該 Workflow 就會被觸發，並且自動根據分支把 build 內容推送到目標 repo 的 `gh-pages` 分支。

```yaml
name: Deploy GitHub Pages

on:
  push:
    branches:
      - develop
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # 1. Checkout 代碼
      - name: Checkout repository
        uses: actions/checkout@v3
        with:
          fetch-depth: 0

      # 2. Setup Node.js
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '22'

      # 3. 安裝依賴
      - name: Install dependencies
        run: npm ci

      # 4. 設置 DEPLOY_ENV 環境變數
      - name: Set DEPLOY_ENV
        run: echo "DEPLOY_ENV=${GITHUB_REF##*/}" >> $GITHUB_ENV

      # 5. Build
      - name: Build project
        run: npm run build

      # 6a. Deploy develop 分支到另一個 repo（preview）
      - name: Deploy preview (develop branch)
        if: github.ref == 'refs/heads/develop'
        uses: peaceiris/actions-gh-pages@v4
        with:
          personal_token: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
          publish_dir: ./dist
          publish_branch: gh-pages
          external_repository: your-username/your-repo-preview

      # 6b. Deploy main 分支到自己 repo
      - name: Deploy production (main branch)
        if: github.ref == 'refs/heads/main'
        uses: peaceiris/actions-gh-pages@v4
        with:
          personal_token: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
          publish_dir: ./dist
          publish_branch: gh-pages
```

## 5. 修改 vite.config.js 的 base 根據 DEPLOY_ENV 環境變數切換對應 repo

```js
// 依照 DEPLOY_ENV 判斷 base
const repoName =
  process.env.DEPLOY_ENV === 'develop'
    ? 'repo-preview' // develop 分支部署到另一個 repo
    : 'repo'; // main 分支部署到自己 repo

export default defineConfig({
  // base 的寫法:
  // base: '/Repository 的名稱/'
  base: `/${repoName}/`,
  // ...
});
```

## 6. 提交並推送到 GitHub

- 將 `.github/workflows/deploy.yml`、`vite.config.js` 檔案推送到你的 GitHub 遠端 repo。

- GitHub 會自動識別這個 Workflow，並在事件觸發時執行。
