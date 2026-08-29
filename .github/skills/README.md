# Skills 命名規範

建立新的 skill（`.github/skills/` 底下）時，請遵守以下規則：

## 1. 目錄與 `name` 欄位
- 資料夾名稱與 `SKILL.md` frontmatter 中的 `name` 欄位**必須一致**。
- 只能使用**小寫英文字母、數字、連字號 `-`**，不可使用大寫、底線、空白或中文。
  - ✅ `restart-post`、`create-new-post`
  - ❌ `RestartPost`、`Restart_Post`、`restartPost`

## 2. 結構
```
.github/skills/[skill-name]/SKILL.md
```
- 每個可被 `Skill` 工具呼叫的 skill 都必須是獨立資料夾 + `SKILL.md`，並在檔案開頭加上 frontmatter：
  ```yaml
  ---
  name: skill-name
  description: 一句話說明用途與觸發時機
  ---
  ```
- 純參考文件（不需被直接呼叫，只被其他 skill 連結引用）可放在 `.github/skills/` 下的單一 `.md` 檔（如 `BlogPublishWorkflow.md`），不需要 frontmatter 與資料夾。

## 3. `description`
- 需清楚描述「做什麼」與「何時使用」，方便未來判斷是否符合當下任務。

## 建立新 skill 前的檢查清單
- [ ] 資料夾名稱為 kebab-case
- [ ] `name` 欄位與資料夾名稱相同
- [ ] 有 `description`
- [ ] 內容放在該資料夾的 `SKILL.md`
