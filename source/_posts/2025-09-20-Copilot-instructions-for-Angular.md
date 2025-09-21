---
title: 替 Angular 提供客製化的 Copilot 指示
date: 2025-09-20 20:55:23
tags:
- Angular
- AI
- LLM
- Web
categories:
- Angular
- AI
- LLM
---

大型語言模型（ large language model, LLM ）已成為近年來開發者投入研究的議題，它已在基礎上改變了＂軟體如何開發＂與＂開發者如何工作＂這兩件事。雖然 LLM 能夠產生程式碼，但為了 Angular 等不斷發展的框架產生程式碼就可能是一個挑戰。指示（ instruction ）和提示（ prompt ）是一種新興的標準，用於生成支援具有特定領域細節的現代程式碼。

以下指示內容是用來幫助 LLM 根據 Angular 最佳實踐（[由此複製](https://angular.dev/ai/develop-with-ai#custom-prompts-and-system-instructions) ）生成符合規範的程式碼：

```markdown
You are an expert in TypeScript, Angular, and scalable web application development. You write maintainable, performant, and accessible code following Angular and TypeScript best practices.
## TypeScript Best Practices
- Use strict type checking
- Prefer type inference when the type is obvious
- Avoid the `any` type; use `unknown` when type is uncertain
## Angular Best Practices
- Always use standalone components over NgModules
- Must NOT set `standalone: true` inside Angular decorators. It's the default.
- Use signals for state management
- Implement lazy loading for feature routes
- Do NOT use the `@HostBinding` and `@HostListener` decorators. Put host bindings inside the `host` object of the `@Component` or `@Directive` decorator instead
- Use `NgOptimizedImage` for all static images.
  - `NgOptimizedImage` does not work for inline base64 images.
## Components
- Keep components small and focused on a single responsibility
- Use `input()` and `output()` functions instead of decorators
- Use `computed()` for derived state
- Set `changeDetection: ChangeDetectionStrategy.OnPush` in `@Component` decorator
- Prefer inline templates for small components
- Prefer Reactive forms instead of Template-driven ones
- Do NOT use `ngClass`, use `class` bindings instead
- DO NOT use `ngStyle`, use `style` bindings instead
## State Management
- Use signals for local component state
- Use `computed()` for derived state
- Keep state transformations pure and predictable
- Do NOT use `mutate` on signals, use `update` or `set` instead
## Templates
- Keep templates simple and avoid complex logic
- Use native control flow (`@if`, `@for`, `@switch`) instead of `*ngIf`, `*ngFor`, `*ngSwitch`
- Use the async pipe to handle observables
## Services
- Design services around a single responsibility
- Use the `providedIn: 'root'` option for singleton services
- Use the `inject()` function instead of constructor injection
```

### Copilot 指示介紹

指示是幫助我們客製化與 LLM 的對話，來配合滿足程式碼的最佳實踐與專案需求。經由撰寫一致配置檔，可以自動將我們偏好的上下文（ context ）、工具和指南套用至每一次的對話，以節省時間並確保回應的一致性，而不需要在每個聊天請求中，手動提供相同的資訊。以 VS Code 為例，有五種主要的方法客製化對話，這些選項可以單獨存在或彼此結合。本篇僅介紹客製化指示的使用方式。

| 項目 | 說明 | 適用情境 | 優缺點 |
| --- | --- | ------- | ----- |
| **[Custom instructions](https://code.visualstudio.com/docs/copilot/customization/overview#_custom-instructions)<br>客製化指示** | 使用 Markdown 檔案定義在程式碼生成、程式碼審查、訊息提交等常見任務時的規範或指南 | - 想在每次對話都自動套用程式碼風格、訊息提交的格式、專案內部的程式碼審查標準等<br>- 在大型團隊、專案規模大、多人合作的時候特別適用 | 👍 針對專案設定 glob pattern （例如只針對某些檔案類型或資料夾），達到一致性<br>👎 初期要耗時寫規範<br>👎 如果規範太多或太嚴格，可能造成 AI 回應速度過慢或回應被限制而不夠靈活 |
| **[Prompt files](https://code.visualstudio.com/docs/copilot/customization/overview#_prompt-files)<br>提示檔案** | 使用 Markdown 檔案定義可重複使用的（ reusable ）的提示（ prompt ）來處理常見或重複的開發任務 | - 利用共享提示，讓大家對某些任務的預期結果一致<br>- 有許多重複性工作或標準化流程時，例如專案中常常要搭建新元件、寫某種格式的文件、生成測試案例等任務 | 👍 可複用，節省時間，且容易維護重複任務的品質一致性<br>👎 初期要耗時設計模板<br>👎 若提示定義不夠清楚或文件沒維護好，可能導致輸出結果不穩定 |
| **[Chat modes](https://code.visualstudio.com/docs/copilot/customization/overview#_chat-modes)<br>聊天模式** | 使用 Markdown 檔案建立特定角色或任務助理，例如資料庫管理、前端開發、規劃研究等，在這個模式可以定義範疇與能力、哪一些工具可以被存取、首選的語言模型等 | 針對專案不同階段（設計階段、測試階段、維護階段）切換不同風格與需求，或者以不同人員（前端、後端、安全審查等）進行此模式，預期 AI 會有不同的回應偏好和工具權限 | 👍 能夠更精細控制 AI 在不同情境中的行為與工具使用，提高專業任務中的效率與精準度<br>👎 設定的複雜度較高，需要維護多個模式與文件。若模式切換頻繁，在管理上較麻煩 |
| **[Language models](https://code.visualstudio.com/docs/copilot/customization/overview#_language-models)<br>語言模型選擇** | 在不同任務間切換模型，選擇最適合的模型來處理特定需求（例如：開發速度快，但能力中等或者進行複雜推理與視覺處理等），允許自帶 API key 存取不同模型或在本地佈署模型 | - 當任務強度、性能或需求精確度不一樣時很有用（例如：簡單重構時希望響應快，而進行架構設計或安全審查時希望使用更強大的模型）<br>- 適合有成本考量或想控制資源時使用 | 👍 靈活根據任務調整模型，以節省時間或資源；若有客製化模型或引入雲端模型，更具拓展的可能性<br>👎 模型間切換可能導致風格、輸出一致性有差異<br>👎 較好的模型通常成本較高，而本地部署或 API key 的管理需注意安全性與版本控制 |
| **[Model Context Protocol (MCP) 與工具整合](https://code.visualstudio.com/docs/copilot/customization/overview#_mcp-and-tools)** | 使用 MCP 連接外部服務或專用工具，讓對話不只在程式碼內部操作，也能查詢資料庫、呼叫 API 、取得即時資訊或執行其它開發工具 | 當專案需要外部資源或工具整合，例如進行跨工具或跨領域的任務時非常有用 | 👍 能整合更多上下文與資源，提高回應的實用性與全面性<br>👍 把 workflow 串起來，不用跳出 VS Code<br>👎 安全與權限需注意（例如存取外部服務憑證與 API key 等）<br>👎 工具整合、維護成本高，可能有相依性或工具不穩定問題 |

<!-- more -->

### 如何使用客製化指示？

VS Code 可支援多個基於 Markdown 的指示文件，並會將其組合在一起加入至對話上下文中。指示檔的用途並不是讓 VS Code 自動生成 Angular 程式碼，而是讓編輯器**具備 AI 輔助功能**，因此需要搭配 VS Code 的延伸模組 GitHub Copilot 與 GitHub Copilot Chat 來使用。準備指示文件的方式主要有以下兩種：

* 單個 `.github/copilot-instructions.md` 檔
  1. 確保啟用 [`github.copilot.chat.codeGeneration.useInstructionFiles`](vscode://settings/github.copilot.chat.codeGeneration.useInstructionFiles) 設定
  2. 在工作區的根目錄建立 `.github/copilot-instructions.md` 檔，並撰寫客製化的指示。 VS Code 會自動將指示內容套用到工作區的對話需求中

* 一或多個 `.instructions.md` 檔（可針對不同的程式語言或框架，準備不同的指示文件），它們只會在建立與修改檔案時使用，以 Angular 框架開發為例：
  1. 準備一個 Angular 提供的指示內容寫入 `.instructions.md`	並置於專案根目錄下
  > 檔名可以更名為 `Angular.instructions.md` 以便於選取時進行辨認
  2. 若有多個指示資料夾，可以在工作區的 [chat.instructionsFilesLocations](vscode://settings/chat.instructionsFilesLocations) 進行設定，或在 `settings.json` 加入指示檔所在的資料夾路徑：
  ```json
  "chat.instructionsFilesLocations": {
    ".github/instructions": true,
  }
  ```
  {% asset_img 1_instructions_file.png %}

  另外一種方式為
    1. 點選「設定聊天...」按鈕
    2. 選擇「提示」選項
    3. 建立新的指示檔案或查看現有的指示檔
  {% asset_img 2_instructions_setting.png %}
  > 若選擇「新的指示檔案...」，則可選擇指示檔案的存放路徑
  3. 有兩種方式使用指示：
    * 將 `applyTo` 屬性加到檔案標頭，指定哪些檔案需要套用指示。
    * 手動將指示檔附加到對話中

### 使用範例

* 在某個資料夾下建立服務，以取得後端 API 資料
   > 資料夾為 `posts`
   > 服務名稱為 `post`
   > 後端 URL 為 `https://jsonplaceholder.typicode.com/posts`

   在送出 prompt 後，我們可以經由以下的 response 檢查它的思考邏輯是否正確：
   1. 建立資料夾 `posts` 與服務 `post.service.ts`
   {% asset_img 3_prompt_using_instruction.png %}
   2. 分析服務 `post.service.ts` 需要的模組與介面，由 LLM 的思維過程中可以看到其遵照指示的規範來生成程式碼
   {% asset_img 4_create_service.png %}
   3. 加入 `provideHttpClient()` 讓服務 `post.service.ts` 能使用 `HttpClient` 取得後端 API 資料
   {% asset_img 5_dependency_injection.png %}
   4. 修改 `app.ts` 來顯示後端資料
   {% asset_img 6_modify_app_component.png %}
   5. 整理變更摘要
   {% asset_img 7_summary.png %}

### 總結

使用指示雖然可以幫助我們產生符合規範的程式碼，但是在技術快速更迭的時代，這些依據過時規範而產出的程式碼或 APIs 可能會不敷使用，甚至開發者需要花更多時間來除錯或重構某些很簡單的問題。若想要生成符合最新規範的程式碼，可以選擇使用 MCP (Model Context Protocol) 的方式來進行，細節可參考 [Angular CLI MCP 介紹](https://kun-neng.github.io/2025/09/13/Angular-CLI-MCP/)。

### 程式碼
* [GitHub repo](https://github.com/Kun-Neng/angular-cli-mcp)

### 參考來源
* 官方文件 [LLM prompts and AI IDE setup](https://angular.dev/ai/develop-with-ai)
