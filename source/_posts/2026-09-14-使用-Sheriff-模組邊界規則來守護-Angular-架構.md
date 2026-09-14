---
title: 使用 Sheriff 模組邊界規則來守護 Angular 架構
date: 2026-09-14 21:23:40
tags: [Angular, TypeScript, ESLint, 架構]
categories: Angular
---

Angular 專案一旦成長到多個 feature 、多個團隊共同維護，很容易出現「元件互相亂 import 」的狀況：訂單模組直接引用客戶模組內部的元件、共用模組反過來依賴某個 feature ，久而久之依賴關係變成一張網，重構時牽一髮動全身。這類問題光靠 code review 很難長期把關，因為 TypeScript 編譯器本身並不理解「資料夾邊界」的概念，只要路徑對得上就能 import 。[Sheriff](https://github.com/softarc-consulting/sheriff) 是一套用來檢查模組邊界與依賴規則的工具，能把架構規範寫成設定檔，讓違規在 commit 前、甚至輸入程式碼的當下就被抓出來。本文說明如何安裝、初始化與使用 Sheriff ，並以 Angular 專案示範如何把它接上 ESLint ，讓 IDE 即時標紅違規的 import 。

> 以下所有指令與輸出結果，皆在一個以 `ng new` 建立的 Angular 22 專案中實際執行驗證。

## 為什麼需要模組邊界工具

Sheriff 的核心概念是「幫每個資料夾貼標籤（ tag ），再定義標籤之間允許的依賴方向」。舉例來說，若將 `orders` 與 `customers` 兩個 feature 分別標記為 `domain:orders` 與 `domain:customers`，並宣告兩者互不相依，那麼只要有人在 `orders` 底下的檔案 import 了 `customers` 底下的元件，Sheriff 就能在靜態分析階段抓到這個違規，不需要等到執行期或跑過所有測試才發現架構被破壞。

這與 Nx 的 module boundary 規則概念類似，但 Sheriff 不綁定特定的 monorepo 工具，單一 Angular 專案（甚至沒有使用 Nx ）也能直接套用。

## 安裝

在 Angular 專案根目錄安裝核心套件與 ESLint 外掛：

```bash
npm i -D @softarc/sheriff-core @softarc/eslint-plugin-sheriff
```

- `@softarc/sheriff-core`：提供設定檔解析、依賴圖分析與 `sheriff` CLI 。
- `@softarc/eslint-plugin-sheriff`：把 Sheriff 的規則包成 ESLint plugin ，讓違規能在 `eslint` 執行時（含 IDE 即時提示）顯示出來。

> 兩個套件版本必須一致（本文驗證時為 `0.19.6`），因為 `eslint-plugin-sheriff` 內部會直接依賴對應版本的 `sheriff-core` 。

## 初始化設定檔

```bash
npx sheriff init
```

執行後會在專案根目錄產生 `sheriff.config.ts`：

```typescript
import { SheriffConfig } from '@softarc/sheriff-core';

/**
  * Minimal configuration for Sheriff
  * Assigns the 'noTag' tag to all modules and
  * allows all modules to depend on each other.
  */

export const config: SheriffConfig = {
  enableBarrelLess: true,
  modules: {}, // apply tags to your modules
  depRules: {
    // root is a virtual module, which contains all files not being part
    // of any module, e.g. application shell, main.ts, etc.
    'root': 'noTag',
    'noTag': 'noTag',

    // add your dependency rules here
  },
};
```

預設情況下所有檔案都被歸類為 `noTag`，且互相允許依賴，等於還沒有任何限制。要讓它真正發揮作用，需要修改 `modules` 與 `depRules` 這兩個欄位：

- `modules`：用路徑樣式（可含 `<placeholder>`）幫資料夾貼標籤。
- `depRules`：定義每個標籤「可以依賴」哪些標籤，未列出的方向一律視為違規。

以一個有 `orders` 與 `customers` 兩個 feature 資料夾（`src/app/features/orders`、`src/app/features/customers`）的專案為例：

```typescript
export const config: SheriffConfig = {
  enableBarrelLess: true,
  modules: {
    'src/app/features/<domain>': 'domain:<domain>',
  },
  depRules: {
    'root': ['domain:orders', 'domain:customers'],
    'domain:orders': [],
    'domain:customers': [],
  },
};
```

這代表：`orders` 與 `customers` 都不能依賴任何其他 domain （空陣列），只有應用程式殼層（`root`）可以依賴兩者去組裝畫面。

## 用 CLI 驗證

修改設定檔後，先確認目前的程式碼是否合規：

```bash
npx sheriff verify src/app/features/orders/order-list.component.ts
```

通過時會顯示：

```
Verification Report

No issues found. Well done!
```

接著刻意讓 `order-list.component.ts` import `customers` 底下的元件（模擬跨界依賴）：

```typescript
import { Component } from '@angular/core';
import { CustomerListComponent } from '../customers/customer-list.component'; // <--

@Component({
  selector: 'app-order-list',
  imports: [CustomerListComponent], // <--
  template: `<p>Order list</p><app-customer-list />`,
})
export class OrderListComponent {}
```

再次執行驗證，這次 Sheriff 會明確指出違規的檔案、違反的方向：

```
Verification Report

Issues found:
  Total Invalid Files: 1
  Total Encapsulation Violations: 0
  Total Dependency Rule Violations: 1
----------------------------------

|-- src\app\features\orders\order-list.component.ts
|   |-- Dependency Rule Violations
|   |   |-- from tag domain:orders to tags domain:customers
```

`from tag domain:orders to tags domain:customers` 這行訊息，精確標出是哪個標籤（也就是哪個 feature ）試圖依賴哪個標籤，比單純的 lint 錯誤訊息更容易對應回架構圖，方便和團隊溝通「這條依賴為什麼不該存在」。

> 若要一次檢查整個專案而不是單一檔案，可以省略路徑參數，直接執行 `npx sheriff verify` 即可；
> CI pipeline 通常會這樣使用，當成 PR 合併前的自動化架構檢查。

<!-- more -->

## 接上 ESLint / IDE 即時提示

到這一步，`sheriff verify` 已經能在 CLI 或 CI 抓到違規，但這是「額外執行一個指令」才能看得到的結果。多數 Angular 專案裝好 `@softarc/eslint-plugin-sheriff` 後，會直覺以為已經能在編輯器裡即時標紅違規的 import ，但是如果專案原本沒有 ESLint 設定（沒有 `eslint.config.js`），這個 plugin 其實還沒有被接進任何 lint pipeline ，`npm run lint` 也無從執行，所以要在打字當下就被提醒，需要先有 ESLint 的 flat config ，再把 Sheriff 的規則疊上去。我們先在 VS Code 中搜尋並安裝微軟官方的 [`ESLint` 延伸模組](https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint)，就可以在開發中看到呈現效果。

{% asset_img 1_eslint_extension.png %}

### 用官方 schematic 建立 ESLint 設定

若 Angular 專案還沒裝過 ESLint ，可以用官方 schematic 一次建立好基礎設定：

```bash
ng add @angular-eslint/schematics
```

在單一專案、尚未有任何 linter 的 workspace 執行後，它會自動：

- 安裝 `eslint`、`@eslint/js`、`typescript-eslint`、`angular-eslint` 等套件
- 建立 `eslint.config.js`（ flat config ）
- 在 `angular.json` 加入 `lint` architect target ，之後可用 `ng lint` 執行
- 在 `package.json` 補上 `"lint": "ng lint"` script

也就是說，這是專案第一次擁有 ESLint 設定，`package.json`、`package-lock.json`、`angular.json` 三個檔案都會因此變動。

### 版本衝突：angular-eslint 22 拉 ESLint 10 ，sheriff 只支援到 9

實際跑一次會發現，`angular-eslint@22`（對應 Angular 22 ）預設會安裝 **ESLint 10**（`eslint@10.x`、`@eslint/js@10.x`），因為它的 peer dependency 是 `eslint@^9.0.0 || ^10.0.0`，npm 會選擇符合範圍的最新版本。但目前 `@softarc/eslint-plugin-sheriff@0.19.6` 的 peer dependency 只到 `eslint@^8.0.0 || ^9.0.0`，還沒有支援 ESLint 10 。若在裝好 `@angular-eslint/schematics` 之後才安裝 Sheriff ，會直接撞到 `ERESOLVE` 錯誤：

```
npm error ERESOLVE unable to resolve dependency tree
npm error
npm error While resolving: sheriff-demo@0.0.0
npm error Found: eslint@10.10.0
npm error node_modules/eslint
npm error   dev eslint@"^10.9.1" from the root project
...
npm error Could not resolve dependency:
npm error peer eslint@"^8.0.0 || ^9.0.0" from @softarc/eslint-plugin-sheriff@0.19.6
```

解法是把 `eslint`、`@eslint/js` 都改釘在相容的 9.x 版本，讓兩邊的 peer dependency 都能滿足：

```json
{
  "devDependencies": {
    "eslint": "^9.39.5",
    "@eslint/js": "^9.39.5"
  }
}
```

改完後重新 `npm install`，再安裝 Sheriff 兩個套件，就不會再出現 `ERESOLVE` 錯誤。

> 在 Sheriff 支援 ESLint 10 之前，`angular-eslint` 專案若要同時使用 Sheriff ，得暫時把 ESLint 釘在 9.x 。之後若 `eslint-plugin-sheriff` 更新 peer dependency ，可以再拿掉這個限制版本，改回 `angular-eslint` 預設帶入的版本。

### 把 Sheriff 規則接進 ESLint

在 `eslint.config.js` 中匯入 `@softarc/eslint-plugin-sheriff`，並把 `sheriff.configs.all` 加進 `extends`：

```javascript
// @ts-check
const eslint = require('@eslint/js');
const { defineConfig } = require('eslint/config');
const tseslint = require('typescript-eslint');
const angular = require('angular-eslint');
const sheriff = require('@softarc/eslint-plugin-sheriff');

module.exports = defineConfig([
  {
    files: ['**/*.ts'],
    extends: [
      eslint.configs.recommended,
      tseslint.configs.recommended,
      tseslint.configs.stylistic,
      angular.configs.tsRecommended,
      sheriff.configs.all, // <--
    ],
    processor: angular.processInlineTemplates,
    rules: {
      '@angular-eslint/directive-selector': [
        'error',
        {
          type: 'attribute',
          prefix: 'app',
          style: 'camelCase',
        },
      ],
      '@angular-eslint/component-selector': [
        'error',
        {
          type: 'element',
          prefix: 'app',
          style: 'kebab-case'
        },
      ],
    },
  },
  {
    files: ['**/*.html'],
    extends: [
      angular.configs.templateRecommended,
      angular.configs.templateAccessibility,
    ],
    rules: {},
  },
]);
```

`sheriff.configs.all` 會啟用 Sheriff 提供的三條規則：
- `dependency-rule` （依賴方向檢查）
- `encapsulation` （模組內部檔案不得被外部直接 import ，只能從該模組的 `index.ts` 引入）
- `deep-import` （與 encapsulation 相輔相成，防堵透過相對路徑進行跨模組的深層抓取）

### 驗證 IDE / lint 層級也能抓到違規

沿用前面 `orders` import `customers` 的例子，直接對該檔案跑 `eslint`：

```bash
npx eslint src/app/features/orders/order-list.component.ts
```

輸出結果：

```
src/app/features/orders/order-list.component.ts
  2:1  error  module \src\app\features\orders cannot access \src\app\features\customers.
              Tag domain:orders has no clearance for tags domain:customers  @softarc/sheriff/dependency-rule

✖ 1 problem (1 error, 0 warnings)
```

這次錯誤是由 `@softarc/sheriff/dependency-rule` 這條 ESLint 規則直接報出來的，代表只要編輯器有啟用 ESLint 延伸模組，寫下違規 import 的當下就會立刻出現紅色波浪線，不用額外執行 `sheriff verify` 或等 CI 才發現； `npm run lint`（也就是 `ng lint`）也會因為這條規則而失敗，我們可以將它加入至 CI 的檢查步驟。測試再把違規的 import 還原後， `eslint` 與 `sheriff verify` 都會再次回報無問題。

## 最佳實踐

- 從寬鬆的標籤開始，逐步收緊：
  一開始可以先用較粗的分類（例如 `feature` / `shared` / `core`），確認整體依賴方向正確後，再細分到每個 domain ，避免一開始規則太嚴格導致大量既有程式碼被擋下來。
- `sheriff verify` 用於 CI ，ESLint 用於開發當下：
  兩者互補而非取代， `sheriff verify` 可以檢查整個專案（含尚未被 lint 涵蓋到的檔案），適合放進 CI 的必要檢查； ESLint 則提供即時回饋，讓開發者在寫程式碼時就能修正，不需要等到推送後才發現。
- `enableBarrelLess: true` 讓模組不需要 barrel 檔：
  較舊版本的 Sheriff 要求每個模組要有 `index.ts` 作為對外窗口，`sheriff init` 產生的新設定預設已改用 barrel-less 模式，可以直接依實際檔案路徑判斷模組邊界，減少維護 barrel 檔的成本。
- 鎖定套件版本，定期檢查相容性：
  `angular-eslint` 與 `@softarc/eslint-plugin-sheriff` 對 ESLint 主版本的支援時程不一定同步，升級 Angular 或 ESLint 前，建議先用 `npm view <package> peerDependencies` 確認相容範圍，避免升級後才卡在依賴衝突。

## 總結

Sheriff 讓「模組邊界」從團隊口頭約定變成可以自動驗證的規則：`sheriff init` 建立設定檔、用 `modules` 與 `depRules` 描述允許的依賴方向、`sheriff verify` 提供 CLI 與 CI 層級的把關，但要讓違規在寫程式碼的當下就被標紅，還需要專案本身已有 ESLint flat config ，並把 `sheriff.configs.all` 加入至 `extends` 。若專案還沒有 ESLint ，`ng add @angular-eslint/schematics` 是最快的起點，只是要注意目前（本文發佈時） `angular-eslint` 與 `eslint-plugin-sheriff` 對 ESLint 主版本的支援有落差，需要暫時把 `eslint` 與 `@eslint/js` 釘在 9.x 才能兩邊相容。

## 參考資料
- Sheriff 官方 GitHub ：https://github.com/softarc-consulting/sheriff
- angular-eslint 官方 GitHub ：https://github.com/angular-eslint/angular-eslint
