---
name: create-new-post
description: Create a new post in the repository using Hexo scaffolding.
---

# 建立新文章

## 目的
使用 Hexo 命令快速建立新文章檔案，自動生成標準 Front Matter 。

## 前置要求
- 位於專案根目錄 (`kun-neng.github.io/`)
- Hexo 已正確安裝

## 使用方法

### 基本語法
```bash
hexo new post "[文章標題]"
```

### 範例
```bash
hexo new post "Angular 21 完整升級指南"
```

### 草稿（尚未確定要發佈時）
```bash
hexo new draft "[文章標題]"
```
草稿會建立在 `source/_drafts/`，不會出現在正式站台上；確定要發佈時執行 `hexo publish "[文章標題]"` 將其移至 `source/_posts/`。

## 產生結果
- 檔案會建立在 `source/_posts/[YYYY-MM-DD]-[標題].md`（日期為執行指令當天，可事後於 Front Matter 的 `date` 欄位調整）。
- 標題若包含空白或中文，會直接反映在檔名與網址 slug 上；若需要英文網址，建議使用英文標題，例如 `hexo new post "Angular-21-Migration-Guide"`。
- Hexo 會自動帶入預設 Front Matter（`title`、`date`、`tags`、`categories`），需再手動補上內容與分類。

## 後續步驟
1. 編輯新建的 `.md` 檔案，撰寫內容並補上 `tags` / `categories`。
2. 執行 `hexo server` 本地預覽，確認排版與 Front Matter 解析正確。
3. 需要完整發佈流程（含 SEO、Git 提交、部署）時，參考 [BlogPublishWorkflow](../BlogPublishWorkflow.md)。
