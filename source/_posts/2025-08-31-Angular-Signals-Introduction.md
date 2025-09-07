---
title: Angular Signals 演進說明
date: 2025-08-31 10:41:55
tags:
- Angular
- Signal
- Web
categories:
- Angular
- Signal
---

Angular Signals 是 Angular 框架在開發過程中一項重要的演進，旨在為應用程式的狀態管理提供更優雅、更高效的解決方案。以下是 Signals 在 Angular 各主要版本從提案到正式發佈的演進：

|年.月   |版本|演進說明|
|-------|---------------------------|-----------------------------------|
|2022.06|&nbsp;&nbsp;v14&nbsp;&nbsp;|**RFC (Request for Comments) 提案階段**<br>目標是找出能解決 Angular 變更偵測（change detection）痛點的新方案，能夠與 RxJS 等現有工具鏈整合，並減少對 ZoneJS 函式庫的依賴。|
|2022.11|&nbsp;&nbsp;v15&nbsp;&nbsp;|**實驗性階段**<br>Angular 內部透過 [`createSignal()`](https://github.com/angular/angular/blob/main/packages/core/src/render3/reactivity/signal.ts#L79) 函數來建立 Signal ，此階段不推薦用於生產環境。<br><br>參考連結：[Angular v15 is now available!](https://blog.angular.dev/angular-v15-is-now-available-df7be7f2f4c8)|
|2023.05|&nbsp;&nbsp;v16&nbsp;&nbsp;|**Signals 革命性發展的里程碑**<br>Signals 被正式發布，不再是實驗性功能，意謂 Signals 已經足夠穩定和成熟，可以安全地在生產環境中使用。<br><br>引入幾個關鍵的 Signals 相關 APIs：<br>1. `signal()`：用於創建一個 writable Signal ，使用 `.set()` 來賦值<br>2. `computed()`：創建一個依賴於其他 Signals ，並且會自動更新的唯讀 Signal （亦即，不可使用 `.set()` 來賦值）<br>3. `effect()`：用於在 Signal 的值改變時觸發副作用（side effects），例如更新 DOM 或觸發 API 呼叫<br><br>在 Observable 與 Signal 之間進行轉換：<br>Signal -> Observable：`import { toObservable } from '@angular/core/rxjs-interop';`<br>Observable -> Signal：`import { toSignal } from '@angular/core/rxjs-interop';`<br><br>參考連結：[Angular v16 is here!](https://blog.angular.dev/angular-v16-is-here-4d7a28ec680d)|
|2023.11|&nbsp;&nbsp;v17&nbsp;&nbsp;|**深度整合新功能，成為新功能的核心**<br>這個版本中最重要的相關更新是 Signals 引入 `@if` 、 `@for` 、 `@switch` 等內建控制流語法。這些新的語法在內部完全依賴 Signals 進行變更偵測，大大簡化了模板的變更偵測機制，並且不需要依賴 `zone.js` 也能正常運作，這對於未來實現 Zoneless 的應用程式非常有幫助。<br><br>參考連結：[Introducing Angular v17](https://blog.angular.dev/introducing-angular-v17-4d7033312e4b)|
|2024.05|&nbsp;&nbsp;v18&nbsp;&nbsp;|**雙向繫結與新 API**<br>引入幾個重要的 Signals 相關 APIs ，簡化元件間的資料傳遞：<br>1. `input()` 讓父元件可以傳遞一個 Signal 的值給子元件<br>2. `output()` 讓子元件可以發出事件，並在父元件中以 Signal 的方式訂閱這些事件<br>3. `model()`：結合了 `@Input()` 和 `@Output()` 的功能，讓開發者可以用更簡潔的方式實現雙向資料繫結。<br><br>參考連結：[Angular v18 is now available!](https://blog.angular.dev/angular-v18-is-now-available-e79d5ac0affe)|
|2024.11|&nbsp;&nbsp;v19&nbsp;&nbsp;|**反應式原語（Reactivity Primitives）**<br>引入新的 Signals 實驗性 APIs 以取得開發者回饋：<br>1. `linkedSignal()`：創建一個類似於 `computed()` ，而且可以依賴於其他狀態的 writable Signal<br>2. `resource()`：執行非同步操作來取得資源<br><br>參考連結：[Meet Angular v19](https://blog.angular.dev/meet-angular-v19-7b29dfd05b84)|
|2025.05|&nbsp;&nbsp;v20&nbsp;&nbsp;|**穩定階段**<br>引入新的 Signals 實驗性 API：<br>`httpResource()` 使用 `HttpClient` 取得 HTTP 資源並回傳 `HttpResourceRef` ，其內容包含 `value` 屬性的 Signal ，可直接在範本中使用<br><br>參考連結：[Announcing Angular v20](https://blog.angular.dev/announcing-angular-v20-b5c9c06cf301)|

官網介紹：
- [Angular Signals](https://angular.dev/guide/signals)
