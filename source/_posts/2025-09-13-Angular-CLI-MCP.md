---
title: Angular CLI MCP 介紹
date: 2025-09-13 21:11:07
tags:
- Angular
- MCP
- Web
categories:
- Angular
- MCP
---

在 [Angular CLI 20.1.0](https://github.com/angular/angular-cli/releases/tag/20.1.0) 加入了 MCP 伺服器，協助開發者基於 Angular 提供的官方 API 進行快速、準確地建立應用程式各項功能。

{% asset_img 1_angular_cli_add_initial_mcp_server.png %}

### [Model Context Protocol (MCP)](https://modelcontextprotocol.io/docs/getting-started/intro)

想像有一個 AI 模型，它要跟外部世界（例如資料庫、 API 、檔案系統、甚至公司內部系統）互動，但不同系統的介面和規則都不一樣。如果要讓 AI 模型連接不同的工具或資料來源，開發者常常需要寫一堆客製化程式，像是自己設計 API 、格式轉換、錯誤處理等等，耗時且不易共享。

MCP 的出現，解決了上述問題，因為 MCP 像一個結合翻譯官與溝通規範的媒介，讓 AI 模型與各種不同來源的資料或工具，用統一的方式來交談。

它具備以下好處：
* 降低整合難度：只要大家都用 MCP ，模型就能更輕鬆地接取不同工具。
* 可移植性：一個 MCP 工具可以同時使用在不同的 AI 模型或平台，減少重工。
* 安全性：MCP 定義了更明確的存取規範，避免 AI 模型隨便存取敏感資料。

### 如何在 Angular 使用 MCP ？

1. 首先建立一個 Angular 20.1.0 版本（或以上）的專案，以專案名稱 `angular-cli-mcp` 為例：
   `npx @angular/cli@20.1.0 new angular-cli-mcp`

2. 使用以下指令來啟動 MCP 伺服器，執行結束會有指引說明如何設定本機環境：
   `ng mcp`
   {% asset_img 2_ng_mcp.png %}

3. 在專案根目錄建立 `.vscode/mcp.json` 檔案（以 VS Code 為例），並加入以下設定：
   ```
   {
     "servers": {
       "angular-cli": {
         "command": "npx",
         "args": ["-y", "@angular/cli", "mcp"]
       }
     }
   }
   ```
   目前提供兩個選項，可加入至 `args` 參數：
   - `--read-only` 僅註冊不會對專案進行改動的工具。編輯器或程式碼代理（ agent ）仍可能執行編輯。預設為 `false` 。
   - `--local-only` 僅註冊不需要網路連線的工具。編輯器或程式碼代理仍可能透過網路傳送資料。預設為 `false` 。

4. 啟動 MCP 伺服器的方式（以下二擇一）
   1. 在 `mcp.json` 檔案中可看到在 `angular-cli` MCP 伺服器加入後出現的選項：
   {% asset_img 3_mcp_configurations.png %}
   點選＂開始＂後即可啟動 MCP 伺服器：
   {% asset_img 4_mcp_is_running.png %}
   2. 在延伸模組中的 MCP 伺服器選取 `angular-cli` 後，手動啟動伺服器：
   {% asset_img 5_extension.png %}

5. 在 agent 模式下，以 prompt 進行設定檢查：
   {% asset_img 6_angular_cli_mcp_tools.png %}
   目前 MCP 伺服器提供三個工具：
   - `search_documentation` 搜尋在 [https://angular.dev](https://angular.dev) 的 Angular 官方文件，提供 API 使用說明與開發範例等（下圖僅擷取部分回應內容）。
     {% asset_img 7_search_documentation.png %}
   - `list_projects` 列出 `angular.json` 檔案內所定義的應用程式（ applications ）和函式庫（ libraries ）。
     {% asset_img 8_list_projects.png %}
   - `get_best_practices` 使用 `#angular-cli` 提供 Angular 官方完整的最佳實踐指南（下圖僅擷取部分回應內容）。
     {% asset_img 9_get_best_practices.png %}

<!-- more -->

### 使用範例

* 在某個資料夾下建立服務，以取得後端 API 資料
   > 資料夾為 `posts`
   > 服務名稱為 `post`
   > 後端 URL 為 `https://jsonplaceholder.typicode.com/posts`

   在送出 prompt 後，我們可以經由以下的 response 檢查它的思考邏輯是否正確：
   1. 建立資料夾 `posts` 與服務 `post`
   {% asset_img 10_1_prompt.png %}
   2. 由於需要取得後端 API 資料，需要提供 `HttpClient`
   {% asset_img 10_2_providehttpclient.png %}
   3. 參考 Angular 官方命名規則，修改服務名稱為 `post.service.ts`
   {% asset_img 10_3_post_service.png %}
   4. 顯示 `post.service.ts` 實作內容
   {% asset_img 10_4_post_service.png %}

* 在 `posts` 資料夾內建立元件，以顯示取得的後端資料
   > 資料夾為 `posts`
   > 元件名稱為 `posts.component.ts`

   在送出 prompt 後，我們可以經由以下的 response 檢查它的思考邏輯是否正確：
   1. 任務拆解，並建立 `posts` 元件
   {% asset_img 11_1_prompt.png %}
   2. 設定並更新路由
   {% asset_img 11_2_routes.png %}
   3. 在調整路由的過程中，會持續檢查錯誤並修正
   {% asset_img 11_3_routes_and_modification.png %}
   4. 在元件注入服務，並修正錯誤
   {% asset_img 11_4_dependency_injection.png %}
   5. 更新 `posts.component.html` 內容，以顯示後端資料
   {% asset_img 11_5_template_updated.png %}
   6. 清理 `app.component.html` 內容，將路由導到 `posts` 元件
   {% asset_img 11_6_template_updated.png %}

### 參考來源
* 官方文件 [Angular CLI MCP Server setup](https://angular.dev/ai/mcp)
