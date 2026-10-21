```markdown
---
title: Fuwari 部落格 GitHub Pages 部署指南（免本地環境）
published: 2026-10-21
description: 從 Fork 到上線的完整流程：改配置、GitHub Actions、刪示範文、寫文章與常見錯誤排除。
image: ''
tags: [Fuwari, Astro, GitHub Pages, 部落格, 教學]
category: 教學
draft: false
---

# Fuwari 部落格 GitHub Pages 部署指南（免本地環境）

本篇整理自實際部署過程，目標是：**只在 GitHub 網頁操作**，不需本機安裝 Node / pnpm，也能完成配置、寫文與自動部署。

---

## 一、取得專案

1. 開啟 [saicaca/fuwari](https://github.com/saicaca/fuwari)
2. 點 **Fork**（或使用 **Use this template** 建立新倉庫）
3. 記住你的倉庫網址，例如：  
   `https://github.com/你的使用者名稱/fuwari`

---

## 二、必改配置檔

### 1. `astro.config.mjs`（站點網址）

路徑：倉庫根目錄的 `astro.config.mjs`

重點修改：

```js
export default defineConfig({
  site: "https://你的使用者名稱.github.io",
  base: "/倉庫名/",   // 若倉庫名是 username.github.io 則用 "/"
  trailingSlash: "always",
  // ...其餘保持
});
```

說明：

| 情境 | `site` 範例 | `base` |
|------|-------------|--------|
| 專案頁（倉庫名 ≠ username.github.io） | `https://username.github.io` | `/倉庫名/` |
| 使用者首頁（倉庫名 = username.github.io） | `https://username.github.io` | `/` |

**不需要自訂域名也能用**，預設 `https://username.github.io/倉庫名/` 即可。

### 2. `src/config.ts`（站點資訊）

路徑：`src/config.ts`

建議至少改：

- 站點標題、副標
- 語言（如 `zh_TW` / `zh_CN`）
- 個人簡介、頭像、社群連結
- 導覽列項目

---

## 三、GitHub Pages + Actions 部署

### 1. 建立工作流檔

路徑：`.github/workflows/deploy.yml`（沒有就新建）

範例內容（注意 **pnpm 版本** 要與專案一致）：

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
          # 若 package.json 裡 pnpm 版本不同，請改成與 package.json 一致

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

### 2. 開啟 GitHub Pages

1. 倉庫 → **Settings** → **Pages**
2. **Source** 選 **GitHub Actions**
3. 推送後到 **Actions** 查看建置是否成功

### 3. 常見建置錯誤：pnpm 版本衝突

錯誤類似：

> Multiple versions of pnpm specified

處理方式：

- 在 `deploy.yml` 的 `withastro/action` 裡明確指定：  
  `package-manager: pnpm@9.14.4`  
  （版本以你倉庫 `package.json` 為準）
- 或拿掉手動指定，改讓 Action 自動偵測

Node.js 棄用警告通常不影響建置，可先忽略。

---

## 四、刪除示範文章

示範文在：

```text
src/content/posts/
```

常見檔案：`markdown.md`、`video.md`、`draft.md` 等，以及可能有的 `guide/` 資料夾。

**在 GitHub 網頁刪除：**

1. 進入 `src/content/posts/`
2. 點開檔案 → 右上角垃圾桶 → Commit  
3. 資料夾可整包刪除

刪完並成功部署後，首頁示範文就會消失。

---

## 五、新建文章（網頁操作）

1. `src/content/posts/` → **Add file** → **Create new file**
2. 檔名建議英文，例如：`my-first-post.md`
3. 開頭必須有 **frontmatter**，範例：

```yaml
---
title: 我的第一篇文章
published: 2026-10-21
description: 這是摘要，會顯示在列表上
image: ''
tags: [生活, 記錄]
category: 日常
draft: false
---

正文從這裡開始，支援 Markdown。
```

### Frontmatter 重點

| 欄位 | 說明 | 注意 |
|------|------|------|
| `title` | 標題 | 必填 |
| `published` | 日期 | 必填，建議 `YYYY-MM-DD` |
| `description` | 摘要 | 建議填 |
| `image` | 封面 | 可空字串 `''` |
| `tags` | 標籤 | 必須是陣列：`[A, B]` |
| `category` | 分類 | 字串 |
| `draft` | 草稿 | `true` 不會公開 |

### 封面圖

```yaml
image: ./cover.jpg
```

圖片可放在與文章同一目錄，或使用完整 URL。

### 常見建置失敗原因（文章相關）

- **沒有 frontmatter**（整篇只有 `# 標題`）→ 必加 `---` 區塊  
- `tags` 寫成字串而非陣列  
- `published` 格式錯誤  
- YAML 用了中文冒號 `：`  

錯誤堆疊常出現在：

```text
.../astro/dist/content/loaders/glob.js ... syncData
```

請往日誌上方找 `InvalidContentEntryDataError` 或 schema 相關訊息，對應修改該篇 frontmatter。

---

## 六、部署流程總覽

```text
Fork / 建立倉庫
    ↓
改 astro.config.mjs（site、base）
    ↓
改 src/config.ts（站點資訊）
    ↓
新增 .github/workflows/deploy.yml
    ↓
Settings → Pages → Source: GitHub Actions
    ↓
刪示範文、寫自己的文章（含正確 frontmatter）
    ↓
Commit / Push → Actions 自動建置部署
    ↓
開啟 https://username.github.io/倉庫名/
```

之後每次改文章或配置並推送到 `main`，就會自動重新部署。

---

## 七、補充說明

- **自訂域名**：非必須；要用再在 Pages 設定與 DNS 綁定即可。  
- **免本地預覽**：全程用 GitHub 網頁編輯 + Actions 即可；有電腦時再本機 `pnpm install` / `pnpm dev` 預覽更方便。  
- **寫文工具**：可用 GitHub 網頁、或本機 / 手機 Markdown 編輯器寫好後貼上。

---

## 結語

Fuwari 搭配 GitHub Pages 的門檻主要在：站點 `site`/`base`、Actions 的 pnpm 版本，以及每篇文章的 frontmatter。把這三點處理好，就能穩定用「只改網頁、自動上線」的方式維護部落格。
```
