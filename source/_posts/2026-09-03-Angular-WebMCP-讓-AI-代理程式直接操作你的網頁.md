---
title: Angular WebMCP - 讓 AI 代理程式直接操作你的網頁
date: 2026-09-03 20:21:26
tags: [Angular, WebMCP, AI, MCP]
categories: Angular
---

當 AI 瀏覽器代理程式（ agent ）要幫使用者填表單、查訂單時，傳統做法是對畫面截圖，再用視覺模型猜測哪個矩形是輸入欄位、哪個按鈕能送出——這種方式既不精準也不穩定。WebMCP（ Web Model Context Protocol ）提供了另一條路：讓網頁主動向瀏覽器內的 AI 代理程式宣告**這個網站能做什麼**，並以結構化的工具（ tool ）暴露操作介面。本文整理 WebMCP 的概念與規格現況，說明如何在 Angular 專案中以官方的實驗性 API 與手動整合 `document.modelContext` 兩種方式實作工具，最後串接 Gemini API ，實際用自然語言下 prompt 驗證代理程式能不能正確呼叫這些工具。

## 什麼是 WebMCP

WebMCP 是由 [W3C Web Machine Learning Community Group](https://www.w3.org/community/webmachinelearning/) 主導的網頁標準草案。核心概念可以理解為：網站本身變成一個 MCP Server，只是傳輸層不是 HTTP + JSON-RPC，而是瀏覽器的 JavaScript 環境本身。頁面可以註冊具名的函式工具，並為每個工具定義輸入結構，供同一瀏覽器內運作的 AI 代理程式（側邊欄助理、內建瀏覽器代理或擴充功能）直接呼叫，由於工具的定義方式與 Model Context Protocol （ MCP ） 十分相似而得名，差別只在於傳輸層換成了瀏覽器的 JavaScript heap 。

**主要優點：**
- 精準：AI 助手可以直接透過結構化工具與應用程式互動（例如直接建立使用者），不需要依賴複雜的 DOM UI 操作或猜測。
- 免部署：工具在客戶端執行，自動繼承使用者已登入的 session 與應用程式狀態，不需要額外部署伺服器，也不用分享 API key 。

**常見應用場景：**
- 複雜表單填寫：區分＂全名＂欄位或＂姓＂＂名＂分開的欄位，比瀏覽器自動填入更精準。
- 難以視覺辨識的介面：如日曆選擇器這類對人類直覺、但對視覺模型較困難的元件。
- 開發者除錯工具：在設定頁建立診斷工具，讓代理程式觸發原本藏在多層選單裡的修正動作。
- 電商／訂單查詢：讓代理程式直接查詢訂單狀態、物流進度等結構化資料。
- UI 元件庫整合：把清單分頁、篩選等操作包成工具，取代逐一點擊 UI 元件。

## 規格現況與限制

- 在 Chrome（149+）提供實驗性原生支援，需要在 `chrome://flags/#enable-webmcp-testing` 開啟才能使用（參考下圖步驟，在設為 `Enabled` 後重新啟動瀏覽器即可）；其他瀏覽器則需依賴 polyfill（如 [`@mcp-b/webmcp-polyfill`](https://www.npmjs.com/package/@mcp-b/webmcp-polyfill) ）。[Microsoft 是規格共同作者之一](https://github.com/webmachinelearning/webmcp) ， Edge 未來支援機會較高，但尚無正式時程。

  {% asset_img 1_enable_webmcp_testing.png %}

- WebMCP 目前仍是早期草案（ Chrome 官方直到 [2026 年 5 月才發布開發者文件](https://developer.chrome.com/docs/ai/webmcp?hl=zh-tw) 正式介紹）， API 仍可能變動，其中一項正在進行的是 API 掛載位置從 `navigator.modelContext` 轉往 `document.modelContext` ，因此會在 DevTools 出現以下錯誤：

  {% asset_img 2_deprecated_navigator_modelContext.png %}

- 工具邏輯執行於瀏覽器 JavaScript 環境中，無法在沒有瀏覽器的伺服器端場景直接使用。
- [規格](https://webmachinelearning.github.io/webmcp/#permissions-policy)透過 Permissions Policy 定義了一個名為 `tools` 的功能，預設允許清單只有 `'self'`，也就是說跨來源的 iframe 預設無法註冊或存取工具；子頁面若要跨來源使用，父層需要在 `<iframe>` 上明確加上 `allow` 屬性委派權限，例如：

  ```html
  <!-- 允許所有來源的子頁面使用 -->
  <iframe src="https://other-origin.com" allow="tools *"></iframe>

  <!-- 只允許特定來源的子頁面使用 -->
  <iframe src="https://trusted-origin.com" allow="tools https://trusted-origin.com"></iframe>
  ```

## Angular 官方實驗性支援

Angular 目前提供的 WebMCP 支援是實驗性質（ Experimental ）， API 隨時可能變動，但已經涵蓋應用程式、路由、服務與 Signal Form 等不同層級的整合方式。

> - 新建 Angular 22 版本的專案，可以使用 `npx @angular/cli@22 new <專案名稱>`
> - 需要注意 Angular 22 有 Node.js 最低版本支援（ `v22.22.3` or `v24.15.0` or `v26.0.0` ），可以使用 nvm 套件協助升級（使用方式可參考 [NVM 簡介](https://kun-neng.github.io/2025/09/07/NVM-Introduction/)）。

## 工具安裝

Google 官方針對 WebMCP 提供的擴充套件為 [WebMCP - Model Context Tool Inspector](https://chromewebstore.google.com/detail/webmcp-model-context-tool/gbpdfapgefenggkahomfgkhfehlcenpd)，由 GoogleChromeLabs 團隊開發，其原始碼託管於 GitHub 的 [webmcp-tools](https://github.com/GoogleChromeLabs/webmcp-tools) 專案中。

{% asset_img 3_Model_Context_Tool_Inspector.png %}

安裝完畢後，可以自擴充功能點開操作畫面，看起來什麼都沒有，是因為網頁跟它還沒有建立連線，也沒有註冊任何工具。它主要分兩個部分，上半部會顯示可使用的工具，下半部可以與網頁進行互動。

{% asset_img 4_Model_Context_Tool_Inspector.png %}

### 應用程式層級

在應用程式配置中使用 `provideExperimentalWebMcpTools` 註冊。工具的 `execute` 回呼函數會在依賴注入環境（ Injection Context ）中執行，代表你可以直接注入其他服務。

> 在 Angular 22 以前的版本，匯入 `provideExperimentalWebMcpTools` 時 IDE 會報錯：
> {% asset_img 5_provideExperimentalWebMcpTools_error.png %}

```typescript
import { Service, inject, provideExperimentalWebMcpTools } from '@angular/core';
import { bootstrapApplication } from '@angular/platform-browser';
import { AppRoot } from './app-root';

@Service()
class Greeter {
  sayHello(): string {
    return 'Hello agent!';
  }
}

bootstrapApplication(AppRoot, {
  providers: [
    provideExperimentalWebMcpTools([
      {
        name: 'greet',
        description: 'Greets the agent.',
        inputSchema: {
          type: 'object',
          properties: {},
        },
        execute: () => {
          const greeter = inject(Greeter);
          return {
            content: [
              {
                type: 'text',
                text: greeter.sayHello(),
              }
            ]
          };
        },
      },
    ]),
  ],
});
```

網頁啟動後，打開 `WebMCP - Model Context Tool Inspector` 擴充工具可看到已經有工具存在，而 `description` 、 `inputSchema` 和 `name` 會顯示在上半部的 WebTools 列表中。經由點擊＂Execute Tool＂會執行該工具提供的功能並回傳特定內容：

{% asset_img 6_execute_tool.png %}

<!-- more -->

### 帶有輸入參數的工具

透過 JSON Schema 語法在 `inputSchema` 中定義參數，規範輸入的型別， Angular 會自動在 `execute` 中推斷傳入參數的 TypeScript 型別。

```typescript
provideExperimentalWebMcpTools([
  {
    name: 'searchCatalog',
    description: 'Searches the store catalog for products matching a query.',
    inputSchema: {
      type: 'object',
      properties: {
        query: {
          type: 'string',
          description: 'The search keywords.'
        },
        maxResults: {
          type: 'number',
          description: 'Maximum number of results to return.'
        },
      },
      required: ['query'],
      additionalProperties: false,
    },
    execute: ({ query, maxResults }) => {
      // 因為 AI 傳入的資料未必絕對符合 Schema ，建議在 runtime 進行驗證
      if (typeof query !== 'string') {
        throw new Error(`Bad query: ${query}`);
      }

      if (typeof maxResults !== 'number' && maxResults !== undefined) {
        throw new Error(`Bad maxResults: ${maxResults}`);
      }

      const limit = maxResults ?? 5;
      return {
        content: [{type: 'text', text: `Returning up to ${limit} results for "${query}".`}],
      };
    },
  },
]);
```

在 `WebMCP - Model Context Tool Inspector` 選擇 `searchCatalog` 工具，並試圖給予不合法的輸入值，可以在 DevTools 看到對應的錯誤訊息：

{% asset_img 7_invalid_inputs.png %}

若給予合法的輸入值，不會有錯誤訊息，而是獲得回應：

{% asset_img 8_valid_input.png %}

### 路由層級

如果工具只在特定頁面有效，可以直接在路由中提供。

```typescript
import { provideExperimentalWebMcpTools } from '@angular/core';
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'dashboard',
    loadComponent: () => import('./dashboard').then((m) => m.Dashboard),
    providers: [
      provideExperimentalWebMcpTools([
        {
          name: 'exportDashboardReports',
          description: 'Exports the current dashboard analytics.',
          inputSchema: { type: 'object', properties: {} },
          execute: () => ({
            content: [
              {
                type: 'text',
                text: 'Dashboard export successfully triggered.'
              }
            ]
          }),
        },
      ]),
    ],
  },
];
```

在 `app.config.ts` 搭配 `withExperimentalAutoCleanupInjectors()` 確保切換路由時工具會自動註銷。

```typescript
export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes, withExperimentalAutoCleanupInjectors())
  ],
};
```

此處我們加入路由功能（細節請參考 [Angular 新手村 - 3. 路由](https://kun-neng.github.io/2024/12/29/Angular-%E6%96%B0%E6%89%8B%E6%9D%91-3-%E8%B7%AF%E7%94%B1)），可以在首頁與儀表板（ dashboard ）之間切換。

在首頁時，只會有原本的兩個工具：

{% asset_img 9_home.png %}

切換到儀表板後，有額外的工具出現：

{% asset_img 10_dashboard.png %}

### 服務層級

針對動態情境，可以使用 `declareExperimentalWebMcpTool` 在任何依賴注入環境（例如元件或服務的建構子）內註冊工具，當該環境被銷毀時，工具也會一併自動清理。

```typescript
import { declareExperimentalWebMcpTool, Service, signal } from '@angular/core';

@Service()
export class Counter {
  readonly count = signal(0);

  constructor() {
    declareExperimentalWebMcpTool({
      name: 'getCounter',
      description: 'Reads the global counter.',
      inputSchema: { type: 'object', properties: {} },
      execute: () => ({
        content: [
          {
            type: 'text',
            text: `The count is: ${this.count()}.`
          }
        ],
      }),
    });

    declareExperimentalWebMcpTool({
      name: 'incrementCounter',
      description: 'Increments the global counter by 1.',
      inputSchema: { type: 'object', properties: {} },
      execute: () => ({
        content: [
          {
            type: 'text',
            text: `The count is now: ${this.increment()}.`
          }
        ],
      }),
    });

    declareExperimentalWebMcpTool({
      name: 'decrementCounter',
      description: 'Decrements the global counter by 1.',
      inputSchema: { type: 'object', properties: {} },
      execute: () => ({
        content: [
          {
            type: 'text',
            text: `The count is now: ${this.decrement()}.`
          }
        ],
      }),
    });
  }

  increment(): number {
    this.count.update((value) => value + 1);
    return this.count();
  }

  decrement(): number {
    this.count.update((value) => value - 1);
    return this.count();
  }
}
```

我們可以建立一個元件進行計數器數值的增減，也可以透過呼叫 `getCounter` /`incrementCounter` / `decrementCounter` 三個工具針對同一個計數狀態進行互動。

### 最佳實踐與注意事項

- **命名衝突**：WebMCP 的工具名稱必須是全域唯一的，因此避免將工具註冊在會被多次重複渲染的元件中，造成重複註冊而拋出錯誤。優先考慮註冊在應用程式層級、路由層級或根服務層級裡。
- **驗證輸入**：雖然定義了 JSON Schema ，但 Angular 不會攔截或驗證 AI 實際傳來的資料，開發者應當在 `execute` 函數內手動驗證資料型別，以確保系統穩定。
- **測試**：若要進行單元測試，官方建議使用 `@mcp-b/webmcp-polyfill` 來做 Mock 實作。

## 手動整合 `document.modelContext`

如果專案目前還停留在 Angular 22 以前版本， `provideExperimentalWebMcpTools` 這類官方 API 雖然無法直接使用，但仍然可以透過原生瀏覽器 API 來實作，因為 WebMCP 本質上是建構在瀏覽器的 Model Context API 之上， Angular 官方的封裝只是讓這層 API 更好用，並非必要條件；換言之，只要瀏覽器（或搭配 polyfill）支援，就能繞過 Angular 版本限制，手動調用底層 API 。

同樣地，即使專案已升級到 Angular 22 ，遇到官方實驗性 API 尚未涵蓋的場景，或是需要更貼近瀏覽器原生行為時，也可以直接操作 `document.modelContext` ，並把 `registerTool` / `unregisterTool` 放進元件的生命週期，讓 `execute` 內部呼叫既有的 service 或更新 Signal。

> 使用 `registerTool` 註冊工具時，需要給定剛才提到的 `name` 、 `description` 、 `inputSchema` 、 `execute` 四個要素。

以下用一個訂單管理頁面（ `OrderListComponent` ）當作場景：使用者可以在頁面上查詢訂單狀態、依狀態篩選列表，而這些操作我們也希望開放給 AI 代理程式使用。接下來會示範三種手動整合的寫法——直接在元件生命週期註冊、搭配 Signal 用 `effect()` 動態註冊、以及把重複邏輯封裝成共用的 `WebMcpService` ，並逐步比較優缺點。

後續範例會共用以下這個簡化的 `OrderService` ，實務上 `getOrders` 會呼叫後端 API ，這裡先以假資料模擬：

```typescript
import { Injectable } from '@angular/core';

export interface Order {
  id: string;
  status: 'pending' | 'shipped' | 'completed';
  orderedAt: string;
}

@Injectable({
  providedIn: 'root'
})
export class OrderService {
  async getOrders(timeframe: string): Promise<Order[]> {
    return [
      { id: 'A001', status: 'shipped', orderedAt: daysAgo(1) },
      { id: 'A002', status: 'pending', orderedAt: daysAgo(3) },
      { id: 'A003', status: 'completed', orderedAt: daysAgo(6) },
      { id: 'A004', status: 'shipped', orderedAt: daysAgo(9) },
      { id: 'A005', status: 'pending', orderedAt: daysAgo(11) },
    ];
  }
}
```

### 元件生命週期：`ngOnInit` / `ngOnDestroy`

第一個步驟，最直接的做法就是把 `registerTool` / `unregisterTool` 分別放進元件的 `ngOnInit` / `ngOnDestroy` ，讓工具的生命週期跟著元件走：頁面顯示時開放代理程式查詢訂單狀態，離開頁面後就收回這個能力，避免工具殘留造成命名衝突或呼叫到已銷毀的元件。

```typescript
export class OrderListComponent implements OnInit, OnDestroy {
  private orderService = inject(OrderService);

  private unregisterTool?: () => void;

  orders = signal<Order[]>([]);

  async ngOnInit() {
    this.orders.set(await this.orderService.getOrders(21));

    const unregister = document.modelContext?.registerTool({
      name: 'get_order_status',
      description: 'Look up orders placed within the last N days, optionally filtered by status.',
      inputSchema: {
        type: 'object',
        properties: {
          daysAgo: {
            type: 'number',
            description: 'How many days back to look, e.g. 7'
          },
          status: {
            type: 'string',
            enum: ['pending', 'shipped', 'completed']
          }
        },
        required: ['daysAgo']
      },
      execute: async ({ daysAgo, status }) => {
        let orders = await this.orderService.getOrders(daysAgo);
        if (status) {
          orders = orders.filter((order) => order.status === status);
        }
        this.orders.set(orders);
        return {
          content: [
            {
              type: 'text',
              text: JSON.stringify(orders)
            }
          ]
        };
      }
    });

    if (typeof unregister === 'function') {
      this.unregisterTool = unregister;
    }
  }

  ngOnDestroy() {
    this.unregisterTool?.();
  }
}
```

### Signal 驅動：用 `effect()` 動態註冊與清理

除了查詢，這個訂單頁面也希望讓代理程式能直接更新訂單狀態（例如標記為已出貨），並讓畫面上的 `orders` Signal 同步反映最新結果。這種需要在 `execute` 內同時操作 Signal 的情境，比起在 `ngOnInit` / `ngOnDestroy` 手動管理，改用 `effect()` 搭配 `onCleanup` 註冊會更簡潔，也不容易漏掉。

```typescript
export class OrderListComponent {
  private orderService = inject(OrderService);

  orders = signal<Order[]>([]);

  constructor() {
    effect((onCleanup) => {
      this.orderService.getOrders(21).then((orders) => this.orders.set(orders));

      const unregister = document.modelContext?.registerTool({
        name: 'update_order_status',
        description: 'Update the status of an existing order (e.g. mark it as shipped or completed).',
        inputSchema: {
          type: 'object',
          properties: {
            orderId: {
              type: 'string',
              description: 'The order ID to update'
            },
            status: {
              type: 'string',
              enum: ['pending', 'shipped', 'completed']
            },
          },
          required: ['orderId', 'status'],
        },
        execute: async ({ orderId, status }) => {
          const newStatus = status as Order['status'];
          await this.orderService.updateStatus(orderId, newStatus);

          this.orders.update((list) =>
            list.map((order) => (order.id === orderId ? { ...order, status: newStatus } : order)),
          );

          return {
            content: [
              {
                type: 'text',
                text: `Order ${orderId} updated to ${status}`,
              },
            ],
          };
        },
      });

      onCleanup(() => unregister?.());
    });
  }
}
```

### 服務封裝：共用 `WebMcpService`

前面兩種寫法都得在每個元件裡重複判斷 `document.modelContext` 是否存在、手動處理瀏覽器不支援的情況，一旦頁面變多，這些樣板程式碼就會到處複製貼上。比較好的做法是把這層邏輯抽成共用的 `WebMcpService` ，元件只需要專注在工具的 `name` 、 `description` 、 `inputSchema` 、 `execute` 上，其餘交給服務處理，同時集中管理以下幾個設計重點：

- **封裝成 Service**：寫一個 `WebMcpService` ， `register(tool)` 回傳 `registerTool` 給的 unregister 閉包，內部處理瀏覽器不支援時的 fallback（ `document.modelContext` 可能是 `undefined` ），避免每個 component 都要判斷。
- **型別安全**：用 TypeScript interface 定義 `inputSchema` 對應的參數型別， `execute` 的參數型別跟 schema 手動對齊（目前沒有官方型別套件會自動生成）。
- **相容性**：由於原生支援僅 Chrome 且需開 flag ，正式導入前建議用 `@mcp-b/global` 之類的 polyfill，或至少做 feature detection 讓功能優雅降級。
- **命名工具**：`name` 必須符合 `^[A-Za-z0-9_.-]+$` 且長度 ≤128 字元，用 route 或 feature 前綴（如 `orders.get_status` ）避免不同模組的工具撞名。

`WebMcpService` 的實作：

```typescript
@Injectable({
  providedIn: 'root'
})
export class WebMcpService {
  readonly isSupported = isPlatformBrowser(inject(PLATFORM_ID)) && !!document.modelContext;

  register(tool: WebMcpTool): () => void {
    if (!this.isSupported) {
      return () => {};
    }

    const unregister = document.modelContext?.registerTool(tool);
    return typeof unregister === 'function' ? unregister : () => {};
  }
}
```

> `isSupported` 那段可不加，因為 `PLATFORM_ID` 用來判斷是否在瀏覽器環境，避免 SSR （ Angular Universal ）下不會出錯。加的話，可以拿來做 UI 上的 feature detection ，例如顯示＂AI 助理可操作此頁面＂的提示。

匯入 `WebMcpService` 後，元件修正如下：

```typescript
export class OrderListComponent {
  private orderService = inject(OrderService);
  private webMcpService = inject(WebMcpService); // <- 匯入 `WebMcpService`

  orders = signal<Order[]>([]);

  constructor() {
    effect((onCleanup) => {
      this.orderService.getOrders(21).then((orders) => this.orders.set(orders));

      const unregister = this.webMcpService.register({ // <- 使用 `WebMcpService` 進行註冊
        name: 'update_order_status',
        description: 'Update the status of an existing order (e.g. mark it as shipped or completed).',
        inputSchema: {
          type: 'object',
          properties: {
            orderId: {
              type: 'string',
              description: 'The order ID to update'
            },
            status: {
              type: 'string',
              enum: ['pending', 'shipped', 'completed']
            },
          },
          required: ['orderId', 'status'],
        },
        execute: async ({ orderId, status }) => {
          const newStatus = status as Order['status'];
          await this.orderService.updateStatus(orderId, newStatus);

          this.orders.update((list) =>
            list.map((order) => (order.id === orderId ? { ...order, status: newStatus } : order)),
          );

          return {
            content: [
              {
                type: 'text',
                text: `Order ${orderId} updated to ${status}`,
              },
            ],
          };
        },
      });

      onCleanup(() => unregister());
    });
  }
}
```

> `register` 回傳 `registerTool` 給的 `unregister` 閉包，讓呼叫端（例如 `effect()` 的 `onCleanup` ）自行決定何時清理， `WebMcpService` 本身不需要碰觸 `DestroyRef` 或依賴注入環境的限制。

## 設定與使用 Gemini API

`Model Context Tool Inspector` 只能手動點擊執行工具，若想改用自然語言下 prompt 、讓 AI 判斷該呼叫哪個工具，可以使用 Gemini API 串接 WebMCP 工具。

1. 前往 [Google AI Studio](https://aistudio.google.com/apikey) 取得 API key 。
2. 點選＂Set Gemini API key＂開啟對話視窗，將 API key 貼入後點＂確定＂。
  {% asset_img 11_set_Gemini_API_key.png %}
  設定成功後，＂Send＂與＂Reset＂按鈕就可以使用了。
  {% asset_img 12_Gemini_API_key_is_set.png %}

3. 在＂User Prompt＂處輸入自然語言，將 `A008` 這項訂單的狀態改為 `Shipped` 後送出。
  {% asset_img 13_user_prompt_1.png %}
  {% asset_img 13_user_prompt_2.png %}

4. AI 代理程式會選擇適合的工具，並傳入對應參數、變更訂單狀態後，回傳結果。
  {% asset_img 14_result.png %}

## 總結

WebMCP 把 **AI 代理程式如何操作網頁**這件事從猜測 DOM 換成了明確宣告的工具介面，對開發者而言等於多了一種面向 AI 使用者的 API 層。 Angular 目前提供兩種開發方式：
1. 使用 Angular 22 以上，涵蓋應用程式／路由／服務等各層級的官方實驗性 API（ `provideExperimentalWebMcpTools` 、 `declareExperimentalWebMcpTool` ）。
2. 不受 Angular 版本限制、直接操作瀏覽器原生的 `document.modelContext` ，從元件生命週期註冊開始，需要跟著 Signal 動態調整時改用 `effect()` 搭配 `onCleanup` 等，到多個元件都要重複這套邏輯，再封裝成共用的 `WebMcpService` 統一管理。

不論用哪種方式註冊，最後都可以串接 Gemini API 這類 LLM ，讓使用者只需要下一句自然語言 prompt ，就能由代理程式自行判斷該呼叫哪個工具、帶入什麼參數，避開痛點：**不再需要對畫面截圖、用視覺模型猜測按鈕位置**。由於規格與 API 都仍在快速變動中（目前僅 Chrome 需開 flag 支援），現階段適合以實驗心態，在非核心流程中小範圍導入，並持續關注 `document.modelContext` 等後續規格演進。

## 參考資料
- [WebMCP](https://developer.chrome.com/docs/ai/webmcp?hl=zh-tw)
- [Angular WebMCP](https://angular.dev/ai/webmcp)
