# blog

個人網站，Jekyll + [minimal-mistakes](https://mmistakes.github.io/minimal-mistakes/) 主題，部署在 Vercel。

## 環境需求

- Ruby >= 3.0（系統內建的可能太舊，建議用 `brew install ruby`，本機用的是 `/opt/homebrew/opt/ruby`）
- Bundler

## 本機開發

```bash
export PATH="/opt/homebrew/opt/ruby/bin:$PATH"
export GEM_HOME="$HOME/.gem-blog"
export PATH="$GEM_HOME/bin:$PATH"

bundle install          # 第一次或 Gemfile 有變動時
bundle exec jekyll serve  # http://localhost:4000 預覽
bundle exec jekyll build  # 只 build，輸出到 _site/
```

## 部署

Push 到 `main` 後 Vercel 會自動重新 build 並部署（設定見 [vercel.json](vercel.json)）。
本機 build 不是 push 的必要步驟，純粹是拿來先抓 YAML/Liquid 語法錯誤、預覽畫面用的。

## 專案結構

```
_config.yml       站台設定（標題、作者、外掛、導覽用的 archive 路徑）
_data/navigation.yml  導覽列選單
_pages/           固定頁面（About、Categories 等），frontmatter 要有 permalink
_posts/           文章，檔名格式 YYYY-MM-DD-slug.md
assets/images/    圖片
assets/css/main.scss  主題樣式覆寫（字體大小、頁面寬度等變數要寫在 @import "minimal-mistakes" 之前）
```

## 常見操作

### 加一篇新文章

在 `_posts/` 新增 `YYYY-MM-DD-標題.md`：

```yaml
---
title: "標題"
categories:
  - 分類名稱
header:
  teaser: /assets/images/xxx-teaser.jpg   # 選填，首頁列表縮圖
---

![](/assets/images/xxx.jpg)   # 選填，內文開頭放圖

內文...
```

分類（category）目前有啟用，會自動出現在 `/categories/` 頁面（[_pages/category-archive.md](_pages/category-archive.md)）。標籤（tags）目前沒有啟用，因為文章量還少，用不到。

### 加圖片

丟進 `assets/images/`，文章裡用 `![說明文字](/assets/images/檔名.jpg)` 引用。
大圖建議先壓縮/縮小再放進去（可以用 macOS 內建的 `sips`，例如 `sips --resampleWidth 2000 -s formatOptions 70 圖檔.jpg`），避免 build 出來的網站太肥。

### 加固定頁面（像 About）

在 `_pages/` 新增 `.md`，frontmatter 要有 `permalink`，需要的話手動加進 [_data/navigation.yml](_data/navigation.yml)。

### 調整外觀（字體大小、版面寬度、配色等）

改 [assets/css/main.scss](assets/css/main.scss)，在 `@import "minimal-mistakes";` **之前**用 `!default` 變數覆寫，例如：

```scss
$doc-font-size: 15px !default;
$max-width: 1600px !default;
```

主題完整的變數清單在 gem 裡的 `_sass/minimal-mistakes/_variables.scss`（本機路徑：`$GEM_HOME/gems/minimal-mistakes-jekyll-*/`）。

## 什麼時候才需要查官方文件

日常寫文章、加圖片、加分類、微調樣式，照上面的模式做就好。真的要查 [minimal-mistakes 文件](https://mmistakes.github.io/minimal-mistakes/) 的情況通常是：

- 想用主題還沒設定的功能（例如留言系統、搜尋、多語系、Table of Contents 側欄）
- 想換版型 skin、大幅調整 layout 結構
- 遇到 build 錯誤，訊息裡提到某個沒看過的主題變數/檔案
