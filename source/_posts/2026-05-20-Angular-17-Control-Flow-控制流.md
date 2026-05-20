---
title: Angular 17 - Control Flow 控制流
date: 2026-05-20 18:33:57
tags: [Angular, Control Flow, 範本語法]
categories: Angular
---

Angular v17 引入了新的 Control Flow （控制流）語法，徹底改變了我們在範本中處理條件判斷和迴圈的方式。讓我們深入探討這個強大功能的語法和優點。

## 介紹

傳統上， Angular 使用 `*ngIf`、`*ngFor` 和 `*ngSwitch` 等結構型指令。雖然這些指令功能完善，但新的 Control Flow 語法提供了更簡潔、更易讀（接近 JavaScript 邏輯）的替代方案來處理範本中的條件渲染與列表迭代，並帶來了性能改進。語法於 Angular v17 引入，並在 v18 成為推薦的開發方式，用以取代傳統的結構型指令。

## Control Flow 語法

以下的＂傳統方式＂代表有別於使用 Control Flow 的語法。

### 1. 條件判斷 `@if`

`@if` 用於根據條件顯示或隱藏 HTML 片段。它支援 `@else if` 與 `@else` 擴充邏輯。

- 傳統方式
```html
<ng-container *ngIf="isVisible; else hidden">
  <div>可見的內容</div>
</ng-container>
<ng-template #hidden>
  <div>隱藏的內容</div>
</ng-template>
```

- `@if` 語法
```html
@if (isVisible) {
  <div>可見的內容</div>
} @else {
  <div>隱藏的內容</div>
}
```

- `@if` 語法（多個條件）
```html
@if (user.role === 'admin') {
  <button>管理員面板</button>
} @else if (user.role === 'editor') {
  <button>編輯內容</button>
} @else {
  <button>查看內容</button>
}
```

#### 使用別名（Alias）

可以使用 `as` 語法將判斷條件的結果存入局部變數，以複用該變數。

```html
@if (items().length; as itemCount) {
  <h2>{{ itemCount }} items to come</h2>
  <!-- displays "2 items to come" -->
}
```

### 2. 列表迭代 `@for`

`@for` 用於渲染集合中的每個項目，並需要加入 `track` （追蹤項目的唯一識別碼） ，有助於 Angular 追蹤項目變動以優化效能。

- 傳統方式
```html
<ul>
  <li *ngFor="let item of items; trackBy: trackItemId; let i = index; let isLast = last">
    {{ i }}: {{ item.name }} {{ isLast ? '(最後一個)' : '' }}
  </li>
</ul>
```
```typescript
trackItemId(index: number, item: Item) {
  return item.id;
}
```

- `@for` 語法
```html
<ul>
  @for (item of items; track item.id; let i = $index; let isLast = $last) {
    <li>{{ i }}: {{ item.name }} {{ isLast ? '(最後一個)' : '' }}</li>
  }
</ul>
```

- `@for` 語法（空列表處理，當集合為空 、 `null` 或 `undefined` 時，會顯示該區塊的內容）
```html
@for (item of items; track item.id) {
  <li>{{ item.name }}</li>
} @empty {
  <li>沒有項目</li>
}
```

#### 隱含變數（Implicit Variables）

- `$index`：目前項目的索引（從 `0` 開始）。
- `$first`：是否為第一個項目。
- `$last`：是否為最後一個項目。
- `$even`：索引是否為偶數。
- `$odd`：索引是否為奇數。

```html
<ul>
  @for (item of items(); track item.id) {
    <li [class.grey]="$even">{{ item.name }}</li>
  }
</ul>
```

> 如果資料沒有唯一識別碼，也可以選擇使用 `track $index`（追蹤索引），但使用唯一識別碼始終是效能最佳的做法。因為當資料陣列發生變動（例如：新增、刪除、排序或過濾）時，使用唯一識別碼和 `$index` 會帶來截然不同的底層操作結果：
> - 使用唯一識別碼（如 `track item.id` ）
>   - 精準更新： Angular 能精準對應資料模型與 DOM 節點
>   - 效能極佳：當陣列順序改變或從中間刪除項目時， Angular 會移動對應的 DOM 元素，而不會銷毀並重新創建它們。
>   - 狀態保留：DOM 節點（例如 `<input>` 輸入框的焦點、未送出的文字、展開的下拉選單）會跟著資料一起移動，保持原有狀態。
> - 使用 `$index` （如 `track $index` ）
>   - 依賴位置而非內容： Angular 只依賴＂第幾個位置＂來識別 DOM 。
>   - 效能低落與畫面重繪：當刪除陣列的第一筆資料時， Angular 看到的是＂索引 0 的資料變了＂，它會強迫銷毀索引 0 的 DOM 、創建新的 DOM ，並更新後續所有的 DOM 節點。
>   - 狀態丟失：因為舊的 DOM 被銷毀並創建了新的 DOM 元素，使用者在畫面上輸入到一半的資料或焦點會全部遺失。

<!-- more -->

### 3. 多重選擇 `@switch`

`@switch` 根據表達式的值，在不同的範本之間切換。

- 傳統方式
```html
<ng-container [ngSwitch]="status">
  <div *ngSwitchCase="'success'">成功!</div>
  <div *ngSwitchCase="'error'">錯誤!</div>
  <div *ngSwitchCase="'loading'">載入中...</div>
  <div *ngSwitchDefault>未知狀態</div>
</ng-container>
```

- `@switch` 語法
```html
@switch (status) {
  @case ('success') {
    <div>成功！</div>
  }
  @case ('error') {
    <div>錯誤！</div>
  }
  @case ('loading') {
    <div>載入中...</div>
  }
  @default {
    <div>未知狀態</div>
  }
}
```

#### 多條件比對

Angular 21.1 （Release Date 2026-01-12）開始支援在單個 `@case` 中設定多個比對值。

在 21.1 之前的版本，必須為每個值寫一個 `@case` ：

```html
@switch (role) {
  @case ('admin') {
    <div>擁有完整權限</div>
  }
  @case ('manager') {
    <div>擁有完整權限</div>
  }
  @default {
    <div>一般使用者</div>
  }
}
```

在 21.1 之後的版本，只需要將多個比對值以逗號分隔，集中在同一個 `@case` 內即可：

```html
@switch (role) {
  @case ('admin', 'manager') {
    <div>擁有完整權限</div>
  }
  @default {
    <div>一般使用者</div>
  }
}
```

### 4. 延遲載入 `@defer`

雖然 `@defer` 常用於效能優化，但它也是控制流的一環。它允許延遲載入頁面的特定區域（及其相關組件），直到滿足特定觸發條件（如以下的 `on hover` ）。

```html
@defer (on hover) {
  <!-- 真正重量級的元件 -->
  <app-car-panel></app-car-panel>
} @loading (minimum 1000ms) {
  <!-- 骨架屏載入中效果 -->
  <app-loading></app-loading>
} @placeholder {
  <!-- 在內容載入前顯示的佔位符 -->
  <app-placeholder></app-placeholder>
}
```

註： `@placeholder` 區塊通常用於顯示骨架屏（ghost elements），避免載入時畫面跳動或出現空白，直到實際內容載入完成。使用 `minimum 1000ms` 確保骨架屏至少顯示 1 秒，避免網路太快時的顯示閃爍，提升視覺平滑度。

## 優點

1. **更好的可讀性**
新語法看起來像原生的控制流程，與 JavaScript 中的 `if/else` 和 `for` 迴圈相似，降低了學習曲線。

2. **性能改進**
- 強制加入 `track` ，以增進效能（ `trackBy` 為選擇性，不設定時容易導致效能損耗）
- 減少了不必要的變更偵測
- 更小的打包體積（為編譯器內建，不需要導入額外模組 `CommonModule` 或特定的指令如 `NgIf` ）

3. **更少的模板複雜度**
- 無需 `ng-container` 與 `ng-template` 標籤
- 無需 `let` 變數綁定的複雜語法
- 程式碼結構更清晰

4. **更強大的表達能力**
- `track` 函數優化了列表渲染
- 內置的 `@empty` 塊簡化了空狀態處理
- 更靈活的條件鏈式處理

5. **改進的開發體驗**
- IDE 的自動補全支援更好
- 類型檢查更準確
- 錯誤報告更清晰

## 實際範例

### 用戶列表

```html
<div class="user-list">
  @if (users.length > 0) {
    <h2>用戶列表 ({{ users.length }} 個)</h2>
    @for (user of users; track user.id; let idx = $index) {
      <div class="user-card">
        <div class="user-number">{{ idx + 1 }}</div>
        <h3>{{ user.name }}</h3>

        @switch (user.status) {
          @case ('active') {
            <span class="badge badge-success">活躍</span>
          }
          @case ('inactive') {
            <span class="badge badge-warning">非活躍</span>
          }
          @case ('blocked') {
            <span class="badge badge-danger">已封禁</span>
          }
        }

        @if (user.isAdmin) {
          <span class="admin-badge">管理員</span>
        }
      </div>
    }
  } @else if (isLoading()) {
    <div class="loading">載入中...</div>
  } @else {
    <div class="empty-state">沒有用戶</div>
  }
</div>
```

```typescript
import { Component } from '@angular/core';

interface User {
  id: number;
  name: string;
  status: 'active' | 'inactive' | 'blocked';
  isAdmin: boolean;
}

@Component({
  selector: 'app-user-list',
  templateUrl: './user-list.component.html',
  styleUrls: ['./user-list.component.css']
})
export class UserListComponent {
  users: User[] = [
    { id: 1, name: '張三', status: 'active', isAdmin: true },
    { id: 2, name: '李四', status: 'active', isAdmin: false },
    { id: 3, name: '王五', status: 'inactive', isAdmin: false }
  ];

  isLoading = signal(false);
}
```

## [遷移工具](https://angular.dev/reference/migrations/control-flow)

針對現存的結構型指令，想要遷移至 Control Flow 語法，可使用
- Angular CLI
  `ng g @angular/core:control-flow`
- Nx CLI
  `yarn nx g @angular/core:control-flow`

## 瀏覽器兼容性

Control Flow 語法在 Angular v17+ 中可用，並支援所有現代瀏覽器。如果需要支援舊版本，可以繼續使用傳統的結構型指令。

可以在 ESLint 配置（ `.eslintrc.json` 或 `eslintrc.js` ）中添加 `@angular-eslint` 規則來強制使用新的 Control Flow 語法，而不是舊的結構型指令（ `*ngIf` 、 `*ngFor` 、 `*ngSwitch` ）：
```json
{
  "rules": {
    "@angular-eslint/prefer-control-flow": "warn"
  }
}
```

`eslint.config.js` 或 `eslint.config.mjs`

```javascript
export default [
  // ...
  {
    files: ['**/*.html'],
    // 載入 Angular template plugin
    plugins: {
      '@angular-eslint/template': require('@angular-eslint/eslint-plugin-template')
    },
    rules: {
      // 強制使用 Control Flow
      '@angular-eslint/template/prefer-control-flow': 'error',
    },
  },
];
```

## 結論

Angular 17 的 Control Flow 語法代表了框架現代化的一個重要步驟。它不僅提供了更簡潔、更易讀的程式碼，還帶來了性能改進和更好的開發體驗。

## 相關資源

- [Angular Control Flow 新版官方文件](https://angular.dev/guide/templates/control-flow)
- [Angular Control Flow 舊版官方文件](https://v17.angular.io/guide/control_flow)
- 效能探討
  - [Unlocking Performance: A Guide to Angular 17’s New Control Flow](https://javascript.plainenglish.io/unlocking-performance-a-guide-to-angular-17s-new-control-flow-68432c265dde)
  - [A Developer’s Guide To Control Flow In Angular](https://medium.com/@himanshusaraswat/a-developers-guide-to-control-flow-in-angular-d220edf57c59)

---

如果您有任何問題或想要了解更多細節，歡迎留言討論！
