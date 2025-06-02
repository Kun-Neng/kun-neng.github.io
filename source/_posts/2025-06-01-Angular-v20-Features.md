---
title: Angular v20 Features
date: 2025-06-01 21:50:28
tags:
- Angular
- Web
categories:
- Angular
---

近年來 Angular 團隊採用以下功能開發的三階段流程，在架構上進行重大變革：
1. Vision （願景）：提出早期概念版本
2. Preview （預覽）：收集開發者社群的深入回饋
3. Stable （穩定版）：根據回饋加以完善，以求最大影響力與穩定性

以下是本次 Angular v20 版本的重要更新：

### 吉祥物

Angular 標誌性的盾牌符號很強大，不過在過去幾個月裡，團隊與 Dart 、 Flutter 和 Firebase 吉祥物背後的創意團隊合作，激盪出一些令人驚豔的概念。以下為 Angular 官方吉祥物的提案，由左至右分別為：

|           Angular shaped character           |           Anglerfish           |           Anglerfish variation           |
|:--------------------------------------------:|:------------------------------:|:----------------------------------------:|
|{% asset_img 1_Angular_shaped_character.png %}|{% asset_img 2_Anglerfish.png %}|{% asset_img 3_Anglerfish_variation.png %}|

投票 [RFC - Angular official mascot #61733](https://github.com/angular/angular/discussions/61733)

### Signals

在 Angular v16 引入響應式的 Signal ，它可以用於管理元件的狀態。

Signals 家族
{% asset_img 4_Signals_family.png %}

* #### `resource` API

  考量到使用 Signal 處理異步任務（那些完成時間無法預測的任務），團隊被引導出新的 `resource` API ，一個用於宣告異步依賴的新原語（ primitive ）。使用 `resource` 可以將狀態、值和其他屬性做為 Signal 來存取，無縫地將它們整合到其他基於 Signal 的邏輯和範本中。

* #### 實驗性 `httpResource`

  以響應式的方式發出 HTTP 請求來獲取資料，它建構在 `HttpClient` 上，所以支援所有原本的功能，像是攔截器、模擬及測試。它同時還具備 Signal 的響應式能力。

  ```typescript
  // products() 會是一個包含 HTTP 回應的 Signal
  const products = httpResource(
    () => `/api/product/${productId()}`
  );
  ```

  官方 API [httpResource](https://angular.dev/api/common/http/httpResource)

* #### 實驗性 `resource`

  從 resource 將回應串流回來，對於需要逐步接收回應的場景（如大型語言模型 LLM 的打字機效果）非常有用。

  ```typescript
  characters = resource({
    stream: async () => {
      const data = signal<ResourceStreamItem<string>>({
        value: "",
      });

      getStreamData(data);

      return data;
    },
  });
  ```

  至於 Signal API 家族的其他成員，團隊宣布 `linkedSignal` 、 `effect` 、 `afterNextRender` 、 `afterRenderEffect` 等更多 API 已升級至穩定版，並準備好在生產環境中使用。

  官方指南 [Async reactivity with resources](https://angular.dev/guide/signals/resource)

* #### Signal Forms

  目前團隊正在開發一個新的、基於 Signal 的表單系統進化版，目標是將備受推崇的範本驅動和響應式表單風格的最佳部分，結合成一個統一的表單系統，從最簡單的登入頁面到最複雜的資料表格，都提供卓越的開發者體驗並具備擴展性。一如既往，向後相容性和互通性是團隊的核心考量。

<!-- more -->

### 開發者體驗（ DX ）更新

* #### Host bindings

  Host bindings 允許你在元件和指令中將動態值綁定到你的 host 元素，然而那些表達式沒有經過型別檢查，這可能會導致執行期錯誤。在 Angular 第 20 版中，團隊為 host 屬性以及 `@HostBinding` 和 `@HostListener` 裝飾器都支援型別檢查。語言服務（ Language Service ）也更新支援如語法高亮（ syntax highlighting ）、跳轉到定義（ go to definition ）以及在 host bindings 內部進行自動化重命名（ automated renaming ）等功能。若想試用這個新功能，在專案設定中啟用 `typeCheckHostBindings` 為 `true` 即可。

  ```typescript
  import { Component } from '@angular/core';

  @Component({
    ...
    host: {
      '[style.color]': 'color' // v20 後，若 color 不存在，編譯期會有型別錯誤
    }
  })
  export class AppComponent { }
  ```

* #### 動態元件建立

  版本更新前，必須以命令式的方式手動套用 input 變更，這不甚理想。

  ```typescript
  age = input(30);

  ngOnInit() {
    this.componentRef = this.viewContainerRef.createComponent(ProfileView);
  }

  ngOnChanges(changes: SimpleChanges) {
    if (changes.age) {
      this.componentRef?.setInput('userAge', this.age());
    }
  }
  ```

  Angular v20 可以使用新的 API ，在建立時宣告 input 和 output 綁定，並讓框架處理其餘部分。新的 API 讓 inputs 和 outputs 經由與在範本中綁定它們時相同的機制。

  ```typescript
  createComponent(ProfileView, {
    bindings: [
      // 綁定 Signal 到 input
      inputBinding('userAge', this.age),

      // 綁定到 output
      outputBinding<Result>('onUserAction', result => console.log(result)),
    ],
  });
  ```

  新的 API 更允許將指令應用於動態建立的元件，例如，將 `ActiveUser` 指令套用在 `UserInfo` 元件。

  ```typescript
  createComponent(ProfileView, {
    bindings: [...],
    directives: [
      ActiveUser,
    ]
  });
  ```

  若需要綁定到指令的 inputs 與 outputs ，只要像在元件中加入綁定一樣，使用相同的新宣告式 API 即可將它們套用至指令。

  ```typescript
  createComponent(UserInfoComponent, {
    bindings: [...],
    directives: [
      {
        type: HasColor,
        bindings: [inputBinding('color', () => 'red')]
      }
    ]
  });
  ```

  > 官方 API
  > * [createComponent](https://angular.dev/api/core/createComponent)
  > * [inputBinding](https://angular.dev/api/core/inputBinding)
  > * [outputBinding](https://angular.dev/api/core/outputBinding)

* #### 其他 DX 更新
  * 支援 TypeSript 5.8
  * 支援範本中未標記的範本字串表達式（untagged template literal expressions）
  * 移除了 Angular Material 對 `@angular/animations` 的依賴（若未使用動畫）
  * 預設啟用範本的熱模組替換（HMR）
  * 新增一個 schematic 用於清理專案中未使用的匯入

這些改進旨在讓開發過程更流暢、更安全（型別檢查）、更高效，並減少最終應用程式的打包體積。

### 效能

隨著這個版本中發布的功能， Angular 將效能做為開發者的預設選項，引入基於 Signal 的響應式模型和伺服器端渲染支援等功能，使交付高效能網頁應用程式比以往更容易。伺服器端渲染（ SSR ）幫助 Angular 開發者交付高效能的應用程式。 SSR 允許在伺服器上渲染包含初始頁面狀態的頁面，並將它們傳送到瀏覽器，以實現更快的初始載入和更好的搜尋引擎優化（ SEO ）。

* #### 水合（ Hydration ）

  在 v16 版本中，完整應用程式水合（ full application hydration ）會重複使用伺服器渲染的 DOM 結構，並持久化應用程式狀態，同時傳輸已由伺服器擷取的應用程式資料。如果沒有啟用水合， SSR 應用程式將需要銷毀並重新渲染整個應用程式的 DOM 。增量式水合（ incremental hydration ） 在 v19 版本以開發者預覽的形式登場，其明確目標是幫助開發者交付快速載入的應用程式，因為初始載入對於創造絕佳的終端使用者體驗非常重要。

  憑藉增量式水合的強大功能，開發者可以選擇範本的哪些部分延遲載入和水合，從而減少打包體積和互動時間。增量式水合自 v20 版本起已進入穩定版，這意味著開發者可以放心地享受這個強大新功能帶來的好處。

  {% asset_img 5_incremental_hydration.png %}

  使用步驟：
  1. 加入 `provideClientHydration` 與 `withIncrementalHydration` 到應用程式設定。
  ```typescript
  bootstrapApplication(App, {
    providers: [
      provideClientHydration(withIncrementalHydration())
    ]
  });
  ```
  2. 使用帶有 `hydrate on` 、 `hydrate when` 或 `hydrate never` 等，精確指定哪些使用者事件應該觸發 Angular 載入和水合延遲載入的內容，這有助於顯著改善首次互動時間（ Time To Interaction, TTI ）。
  ```typescript
  @defer (hydrate on interaction) {
    <large-component />
  } @placeholder {
    <div>Larget component placeholder</div>
  }
  ```

  > 官方指南 [Incremental Hydration](https://angular.dev/guide/incremental-hydration)

* #### 路由級別渲染模式（ Route-Level Render Mode ）設定 API

  Angular 的路由級別渲染模式設定 API 在 v20 版本中提升至穩定版，讓開發者完整控制應用程式的渲染，例如明確地定義每個路由元件要由伺服器渲染（ SSR ）、靜態生成（ SSG ）或客戶端渲染（ CSR ）。

  ```typescript
  {
    path: 'profile',
    renderMode: RenderMode.Server,
  },
  {
    path: 'about',
    renderMode: RenderMode.Prerender,
  },
  ```

* #### Zoneless

  在 v20 版本中， Zoneless Angular 由實驗階段進入開發者預覽階段。採用 Zoneless Angular 可以透過消除不必要的變更偵測循環來提高應用程式效能。由於 `zone.js` 用來修補瀏覽器 API ，它通知 Angular 執行變更偵測來響應任何瀏覽器事件，進而更新應用程式的範本，但也導致許多不必要的檢查。因此 Zoneless 的目標是移除 `zone.js` ，並結合 Signals 來明確控制何時與如何觸發變更偵測。

  Signals + Zoneless 的優勢：
  1. 提升效能
  2. 改善除錯體驗
  3. 更好的生態系相容性
  4. 更小的 JavaScript 負載

  啟用 Zoneless 步驟：
  1. 升級到 Angular v20 （ `ng update` ）
  2. 在應用程式設定中加入 `provideZonelessChangeDetection`
  3. 反安裝 ZoneJS （ `npm uninstall zone.js` ）

  > 官方指南 [Angular without ZoneJS (Zoneless)](https://angular.dev/guide/zoneless)

* #### Chrome 開發者工具的 Angular 自訂追蹤

  由 Angular 團隊與 Chrome DevTools 團隊合作開發，在 Chrome 開發者工具加入 Angular 自訂追蹤（ custom track ），使用新 Angular 追蹤的步驟：
  1. 升級到 Angular v20 （ `ng update` ）
  2. 升級到 Chrome v136 或更高版本
  3. 在 Chrome 主控台執行 `ng.enableProfiling()` 來啟用效能數據收集

  在 Chrome DevTools 的 Performance 面板中，會出現一個名為 "Angular" 的自訂軌道，顯示 Angular 特有的效能數據，如變更偵測、元件生命週期等，幫助開發者更容易區分應用程式碼和函式庫程式碼的效能瓶頸。

### 工具鏈與基礎設施

* #### 測試

  隨著 Karma 被棄用， Angular 團隊一直專注於尋找合適的測試替代方案，以滿足三個目標：
  * 提供清晰的遷移路徑
  * 擁有強大的社群支援
  * 改善開發者體驗

  目前提供的實驗性支援：
  * Jest （用於 Node 環境測試）
  * Web Test Runner （用於瀏覽器環境測試）
  * Vitest （ Angular v20 新增的實驗性支援，可同時用於 Node 和瀏覽器環境）

  最終只會選擇一種方案進入穩定版，社群的回饋將是重要參考。

* #### 工具鏈更新

  * Angular DevTools 的路由視覺化（ Router Visualization ）實驗性支援，啟動方式如下：

  {% asset_img 6_enable_router_graph.png %}

  完成後，即可看到執行期的視覺化路由。

  {% asset_img 7_router_visualization.png %}

  * Webpack 已從 `ng new` 移除，意味著簡化建置系統和依賴項，這使得新建立的 Angular 專案的依賴項減少了約 40% 。

* #### 生成內容安全策略（ Auto Content Security Policy, AutoCSP ）

  Angular 團隊與 Google 安全工程團隊合作，加入在生產建置中使用 AutoCSP 的支援，在 Angular v19 中初步實現，而在 v20 中修復錯誤並使其更加穩定。

  啟用 AutoCSP 的方式：
  在 `angular.json` 的 `architect.build.options.security` 中加入 `"autoCsp": true`

  啟用後，在生產建置時會自動生成內容安全策略，並包含在應用程式碼中，有助於防禦跨網站指令碼（ cross-site scripting, XSS ）攻擊。如果惡意行為者將內嵌腳本注入應用程式，自動化的 CSP 設定將阻止該腳本執行。

  工具鏈的簡化和安全性的增強，旨在讓開發者更專注於業務邏輯，同時建構出更安全、更輕量的應用程式。

### AI 整合

[Build with AI](https://angular.dev/ai) 為 AI 與 Angular 的新家，這個新的資源中心旨在幫助 Angular 開發者充分使用 AI ，其內容包含：
* 指引與最佳實踐
* 入門專案
* 程式碼配方
* 引導式教學

網站提供 Google 的 AI 工具整合，如 Vertex AI 、 Firebase 與 Genkit ，這些知識和模式也適用於其他 AI 解決方案。

### 總結

Angular v20 是一個重要的里程碑，帶來了多方面的提升，包括更成熟的 Signal API 、顯著的效能改進（如增量式水合和 Zoneless 的進展）、更強大的開發者工具（如新的 Chrome DevTools 整合和測試方案），以及對 AI 整合的積極佈局。核心目標始終是讓開發者能夠更輕鬆、更高效地打造出使用者喜愛的高品質應用程式。

### 參考連結

官方 YouTube 影片 [Angular v20 Developer Event 2025](https://www.youtube.com/watch?v=FcDamOe1qxA)
