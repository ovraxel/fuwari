---
title: Fuwari 部落格 GitHub Pages 部署指南（免本地環境）
published: 2026-10-21
description: 從 Fork 到上線的完整流程：網站配置、GitHub Actions、文章管理與常見錯誤排除。
image: ''
tags: [Fuwari, Astro, GitHub Pages, 部落格, 教學]
category: 教學
draft: false
---

# Fuwari 部落格 GitHub Pages 部署指南（免本地環境）

本文介紹如何只透過 **GitHub 網頁介面** 建立及維護 Fuwari 部落格，不必在本機安裝 Node.js 或 pnpm。網站由 GitHub Actions 自動建置，並部署至 GitHub Pages。

## 一、建立專案

1. 開啟 [saicaca/fuwari](https://github.com/saicaca/fuwari)。
2. 點選 **Fork**，或使用 **Use this template** 建立倉庫。
3. 記下倉庫網址，例如 `https://github.com/username/fuwari`。

## 二、修改網站配置

### 2.1 `astro.config.mjs`

在倉庫根目錄找到 `astro.config.mjs`，確認站點網址與部署路徑：

```js
export default defineConfig({
  site: "https://username.github.io",
  base: "/fuwari/",
  trailingSlash: "always",
  // 其餘原有配置保留
});
```

以上僅示意需要修改的欄位，請勿刪除原檔案中其他必要配置及匯入。

| 網站類型 | `site` | `base` |
| --- | --- | --- |
| 使用者首頁（倉庫為 `username.github.io`） | `https://username.github.io` | `/` |
| 專案頁面（倉庫為 `fuwari`） | `https://username.github.io` | `/fuwari/` |

專案頁面預設網址為 `https://username.github.io/fuwari/`，不必購買自訂域名。

### 2.2 `src/config.ts`

依照原有配置結構，修改網站標題、副標題、語言（例如 `zh_TW`）、個人簡介、頭像、社群連結及導覽列。不同版本欄位可能有所差異，請以目前倉庫內容為準。

## 三、設定 GitHub Pages 與 Actions

### 3.1 建立部署工作流

如果專案尚無適用的 Pages 工作流，建立 `.github/workflows/deploy.yml`：

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install, build, and upload
        uses: withastro/action@v4
        with:
          package-manager: pnpm@9.14.4

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy
        id: deployment
        uses: actions/deploy-pages@v4
```

**注意：** `pnpm@9.14.4` 是範例，應與倉庫 `package.json` 的 `packageManager` 設定一致。如果專案已有部署工作流，優先修改現有檔案，避免重複部署。

### 3.2 啟用 GitHub Pages

1. 進入倉庫 **Settings → Pages**。
2. 將 **Build and deployment → Source** 設為 **GitHub Actions**。
3. 提交變更後，在 **Actions** 頁面查看建置與部署結果。

### 3.3 pnpm 版本衝突

如果出現：

```text
Multiple versions of pnpm specified
```

請檢查 `package.json` 中的 `packageManager` 與工作流中的 `package-manager` 是否一致；也可移除工作流內手動指定的版本，讓 Action 自動偵測。Node.js 棄用警告不一定代表建置失敗，應以實際錯誤日誌為準。

## 四、刪除示範文章

文章通常位於：

```text
src/content/posts/
```

原始專案可能包含 `markdown.md`、`video.md`、`draft.md` 或 `guide/` 等示範內容，實際以所使用版本為準。

在 GitHub 網頁中逐一開啟不需要的檔案，使用刪除功能並提交變更。Git 不追蹤空資料夾，因此刪除資料夾內全部檔案後，資料夾便不再顯示。重新部署成功後，已刪除的示範文章會從網站消失。

## 五、建立新文章

1. 進入 `src/content/posts/`。
2. 點選 **Add file → Create new file**。
3. 輸入檔名，例如 `my-first-post.md`。
4. 貼上以下內容並提交：

```markdown
---
title: 我的第一篇文章
published: 2026-10-21
description: 這是我的第一篇 Fuwari 部落格文章。
image: ''
tags: [生活, 記錄]
category: 日常
draft: false
---

# 我的第一篇文章

歡迎來到我的部落格。

## 關於本站

本站使用 Fuwari 建立，並透過 GitHub Pages 自動部署。
```

### Frontmatter 欄位說明

| 欄位 | 用途 | 範例 |
| --- | --- | --- |
| `title` | 文章標題 | `我的第一篇文章` |
| `published` | 發布日期 | `2026-10-21` |
| `description` | 文章摘要 | `簡短介紹` |
| `image` | 封面圖片 | `''` |
| `tags` | 標籤陣列 | `[生活, 記錄]` |
| `category` | 文章分類 | `日常` |
| `draft` | 是否為草稿 | `false` |

封面可依專案支援的路徑規則引用本地圖片，或填入可公開存取的完整網址：

```yaml
image: 'https://example.com/images/cover.jpg'
```

## 六、常見錯誤排除

### 6.1 文章資料格式錯誤

每篇文章應包含符合目前專案內容集合 Schema 的 Frontmatter。常見錯誤包括：

- 缺少開頭及結尾的 `---` 分隔線。
- 將 `tags` 寫成單一字串，而非 `[標籤一, 標籤二]`。
- `published` 日期格式不正確。
- 使用中文冒號 `：` 代替 YAML 的英文冒號 `:`。
- 將 `draft: false` 寫成不符合預期的值。

如果標題含有冒號等特殊字元，建議加上引號：

```yaml
title: '閱讀筆記：自由與責任'
```

### 6.2 Astro 建置失敗

若日誌出現：

```text
InvalidContentEntryDataError
```

或堆疊包含：

```text
astro/dist/content/loaders/glob.js
syncData
```

請向上查找第一個實際錯誤，確認是哪篇文章或哪個欄位不符合 Schema，再修改該檔案。最後的 `Process completed with exit code 1` 只表示執行失敗，並非根本原因。

### 6.3 網站樣式或圖片消失

確認 `astro.config.mjs` 中的 `site` 和 `base` 是否與 Pages 網址一致。專案頁面若錯誤地將 `base` 設為 `/`，可能導致靜態資源路徑錯誤。

### 6.4 Actions 成功但網站未更新

確認 Pages 的 Source 是 **GitHub Actions**，最新部署工作流已成功完成，且瀏覽器沒有顯示快取內容。也要確認文章變更已提交至工作流監聽的分支，例如 `main`。

## 七、部署流程總覽

```text
Fork / 建立倉庫
        ↓
修改 astro.config.mjs（site、base）
        ↓
修改 src/config.ts（站點資訊）
        ↓
建立或調整 deploy.yml
        ↓
Settings → Pages → GitHub Actions
        ↓
刪除示範文並建立自己的文章
        ↓
Commit → Actions 自動建置與部署
        ↓
開啟 https://username.github.io/fuwari/
```

往後每次修改文章或配置，只要將變更提交到 `main`，就會觸發自動部署。

## 八、補充說明

- **自訂域名：** 非必要；日後可在 Pages 設定及 DNS 中配置。
- **免本地環境：** 全程可使用 GitHub 網頁編輯及 Actions 建置。
- **本地預覽：** 如果日後有電腦，可依專案指定的套件管理器安裝依賴並啟動開發伺服器。
- **寫文工具：** 可直接使用 GitHub 網頁，也可以先在手機或電腦的 Markdown 編輯器中完成文章，再貼入倉庫。

## 結語

Fuwari 搭配 GitHub Pages，最需要留意的是站點 `site` / `base` 配置、Actions 使用的套件管理器版本，以及每篇文章的 Frontmatter。完成初始設定後，日常維護便可以簡化為：**編輯 Markdown、提交變更、等待自動上線。**
