---
title: Introduction to the Google Antigravity
date: 2025-12-28 13:16:26
tags:
- Antigravity
- IDE
- AI
categories:
- Antigravity
- IDE
- AI
---

Google Antigravity IDE 是 Google 於 2025 年推出的下一代 AI 整合開發平台（ IDE ），定位為代理優先（ Agent-First ）開發環境，藉由強大的 AI 代理（ AI Agents ）協助開發者進行整個軟體開發流程，包含分析任務 → 規劃 → 拆解任務 → 生成程式碼 → 測試與審查 → 反饋迴圈甚至最後於瀏覽器驗證。

本文發佈時， Antigravity 為公開預覽（ public preview ）階段，提供個人免費使用，而內文以版本 `1.13.3` 為例。

> Antigravity 根據該團隊的建議之發音為 /æntɪˈɡrævɪti/ ，縮寫為 AGY 。

### 核心理念

Google Antigravity 的設計理念，是將傳統的 IDE 體驗提升至代理優先，達到
- AI 代理自主運作：不只是提供 autocomplete 或簡單的建議，而是能夠獨立規劃、執行、驗證複雜任務。
- 跨工具一體化作業：代理可同時橫跨編輯器（ editor ）、終端機（ terminal ）與瀏覽器（ browser ），執行端到端流程。
- 以任務為中心的工作流程：使用自然語言定義目標， AI 將其拆解成可執行步驟並自動完成。

### 功能介紹

0. 於[官方網站](https://antigravity.google/download)下載並安裝，進行個人化設定後以 Google 帳號登入

1. Agent Manager
   - 同時啟動與監控多個 AI 代理
   - 指派任務並追蹤進度
   - 管理跨專案與跨工作空間的代理活動
   - 以任務清單與實作計畫展示代理執行情況
  \
  完成設定並登入 Google 帳號後，畫面呈現類似任務控制面板，如下圖所示。
  {% asset_img 1_0_agent_manager.png %}
  點選 Next 後，如 VS Code 的操作，可以選擇開啟本地現有的專案。另一個選項需要使用 Remote SSH 來開啟遠端的專案。
  {% asset_img 1_1_agent_manager_set_up_workspace.png %}
  > 若選擇開啟現有專案，它會偵測專案內的語言或函式庫，自動推薦適合的 Extensions 進行安裝。
  選定專案後，可以開始針對該專案設定代理工作。
  {% asset_img 1_2_agent_manager_start_conversation.png %}
  介面各區塊說明如下：
     1. Inbox
        集中管理所有對話的功能。當您指派任務給 Agent 後，這些任務會顯示在收件匣中，方便查看所有當前對話的清單。點擊任一對話即可檢視完整的訊息往來、任務狀態、 Agent 生成的內容，甚至是待核准的工作，讓您能回顧與處理先前的任務。
     2. Start Conversation
        點擊可開啟新對話，系統會直接跳轉至輸入框，讓您快速開始互動。
     3. Workspaces
        提供多工作區的支援，可以隨時新增工作區，並在開始對話時切換至所需的工作區進行操作。
     4. Playground
        與代理程式進行初步對話，並根據需要將對話轉換為正式的工作區，方便管理檔案與任務。
     5. Editor View
        切換至編輯器，查看工作區資料夾與生成的檔案。進入此模式後可直接編輯檔案，並在編輯器中提供指引或指令，讓 Agent 根據您的修改執行動作或進行調整（參考功能項目 2 ）。
     6. Browser
        Antigravity 的一大特色是與 Chrome 瀏覽器的深度整合（參考功能項目 4 ）。

2. Editor（編輯器）
   - 程式碼智慧建議與補全
   - 自然語言命令與指令
  \
  編輯器與 VS Code 極為類似，我們可以開啟 Agent 操作頁面（可點選右上方的按鈕進行開關）
  {% asset_img 2_0_editor.png %}
  \
  可以在輸入框進行以下操作：
  - 輸入以下文字
    - `@` 加入背景資訊（如檔案、目錄、 MCP 伺服器）
    - `/` 參考工作流程（選擇已儲存的 instruction ）
  - 點擊按鈕加入圖片或上述功能
  {% asset_img 2_1_add_context.png %}
  - 選擇兩種 AI 對話模式（此為 Antigravity 剛推出的亮眼特色）
    - `Planning` 適用於深入研究、複雜工作或協作等較複雜的工作，代理程式會規劃步驟並核准執行
    - `Fast` 適合於快速完成簡單的工作
    {% asset_img 2_2_conversation_mode.png %}
  > 可以在右上方的 User Settings 設定是否自動核准以下項目：
  > **Artifact Review Policy**
  > - `Always Proceed` 一律核准
  > - `Agent Decides` 交由代理決定
  > - `Request Review` 要求使用者核准
  > {% asset_img 2_0_settings_artifacts_review_policy.png %}
  > **Terminal Command Auto Execution**
  > - `Always Proceed` 一律核准
  > - `Request Review` 要求使用者核准
  > {% asset_img 2_0_settings_terminal_command_auto_execution.png %}
  - 切換多種最新模型
    - Google Gemini 3 Pro 、 Flash
    - Anthropic Claude Sonnet 4.5
    - 開源 GPT-OSS
    {% asset_img 2_3_model.png %}

<!-- more -->

3. 構件（ Artifacts ）
   在規劃或執行工作的過程中產生可驗證文件：
   - 任務清單（ Task Lists ）
   - 實作計畫（ Implementation Plan ）
   - 逐步操作（ Walkthrough ）
   - 程式碼差異、瀏覽器截圖與互動等
  \
  上述這些產出物可用於審核、驗證與回顧 AI 代理工作的行為。
  \
  例如：在 `Planning` 模式輸入 `參考 angular-chat-app 所需要的資料，在 nest-chat-api 建立對應的 API` 後，過程中會依序產生以下檔案。
   - 包含任務清單的 `Task` 檔
   {% asset_img 3_1_task.png %}
   > 可以在以下藍色箭頭處加入修改評論，讓模型能生成更貼近需求的程式碼
   > {% asset_img 3_1_task_comment.png %}
   - 包含實作計畫的 `Implementation Plan` 檔
   {% asset_img 3_2_implementation_plan.png %}
   - 包含執行步驟的 `Walkthrough` 檔
   {% asset_img 3_3_walkthrough.png %}
  上述檔案都會留存在 Inbox 中，往後可再開啟檢視。

4. 瀏覽器集成與自動測試
   - 直接在瀏覽器中啟動與測試應用程式
   - 透過瀏覽器互動執行 UI 驗證
   - 捕捉並記錄測試結果
   - 對畫面截圖加上評論

5. 規則和工作流程
   - `Rules` 系統指令：引導代理程式遵循所提供這些指引（例如特定程式碼樣式或一律記錄方法），觸發條件可選擇
     - `Manual`
     - `Always On`
     - `Model Decision`
     - `Glob`
   - `Workflows` 已儲存的提示：在與代理互動時以 `/` 觸發
  \
  經由右上方的選項進入
  {% asset_img 4_1_customizations.png %}
  設定規則
  {% asset_img 4_2_rules.png %}
  設定工作流程
  {% asset_img 4_3_workflows.png %}
  > 兩者都可以套用至全域（使用者目錄）或個別工作區（專案根目錄），存放路徑分別如下
  > - 全域規則： `~/.gemini/GEMINI.md`
  > - 工作區規則： `your-workspace/.agent/rules/`
  > - 全域工作流程： `~/.gemini/antigravity/global_workflows/global-workflow.md`
  > - 工作區工作流程： `your-workspace/.agent/workflows/`

### `Playground` 範例

範例一：點選 `Playground` 旁的加號，在輸入框輸入 `go to antigravity.google` ，會引導開啟瀏覽器與說明頁面。
  1. 要求安裝 Chrome 瀏覽器套件 Antigravity Browser Extension
  {% asset_img 5_0_antigravity_browser_control.png %}
  2. 回到 `Playground` 同意 Antigravity 在瀏覽器開啟 URL
  {% asset_img 5_1_permission.png %}
  3. 瀏覽器正在載入畫面
  {% asset_img 5_2_getting_dom.png %}
  4. 若輸入 `Translate the text to traditional Chinese` 並提交， Antigravity 會持續進行修改並要求核准確認
  {% asset_img 5_3_confirmation.png %}
  > 若使用者畫面不在 Agent Manager ，瀏覽器會顯示＂待核准＂的提醒
  > {% asset_img 5_4_notification.png %}
  5. 修改後的結果呈現於瀏覽器
  {% asset_img 5_5_result.png %}

範例二：點選 `Playground` 旁的加號，在輸入框輸入 `建立一款具備現代風格且簡潔美觀的番茄鐘`
  1. 在使用者目錄下建立專案
  {% asset_img 5_6_start_a_new_project.png %}
  2. 經歷數次的核准確認
  {% asset_img 5_7_confirmation.png %}
  3. 結果呈現於瀏覽器
  {% asset_img 5_8_result.png %}

### 監控模型用量

安裝 `Antigravity Cockpit` 套件會在右下方顯示模型用量，滑鼠停留其上會顯示詳細資訊。

{% asset_img antigravity_cockpit.png %}

### 總結

Google Antigravity IDE 是一款革命性的 AI 驅動開發工具，代表了一種全新的代理優先開發流程。它把 AI 代理視為核心工作單位，不只是提示補全，而是能夠在編輯器、終端機與瀏覽器間自主完成任務、測試與反饋迴圈。這種模式極大化提高生產力、降低重複工作量，並重塑未來軟體開發方式。

### 參考資料
* [官方網站](https://antigravity.google)
* [使用教學](https://codelabs.developers.google.com/getting-started-google-antigravity?hl=zh-tw)
