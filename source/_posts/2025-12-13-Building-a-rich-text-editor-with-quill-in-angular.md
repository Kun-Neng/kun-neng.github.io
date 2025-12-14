---
title: 在 Angular 以 Quill 打造富文字編輯器
date: 2025-12-13 15:55:21
tags:
- Angular
- Quill
- Web
categories:
- Angular
- Quill
- Web
---

### 前言

在現代 Web 開發中，富文字編輯器（ Rich Text Editor ）已成為聊天系統、部落格平台、知識庫編輯器、後台管理系統的基本要素。 Quill 是一款輕量、模組化、可擴充的開源富文字編輯器，對於 Angular 的開發者來說，有兩個套件能讓整合變得簡單：
- `ngx-quill` 提供編輯功能
- `quill-view-html` 僅渲染 HTML ，來顯示富文字內容

本篇文章將從基礎開始，說明如何在 Angular 專案中安裝、設定並運用 Quill 、 `ngx-quill` 與 `quill-view-html` 。


### 什麼是 Quill？

[Quill](https://quilljs.com/) 是一款輕量、模組化、可擴充的開源富文字編輯器，有以下核心功能：
- 客製化主題
  - snow
  - bubble
- toolbar
  - 文字樣式（粗體、斜體、底線、顏色等）與字型大小
  - 程式碼區塊（程式碼語法高亮顯示）
  - 多媒體嵌入（插入圖片、影片和連結）
  - 清單與對齊（有序清單、無序清單及多種文字對齊方式）
  - 支援各種擴充（ mention 與 tag 等）

### 在 Angular 中使用 Quill 編輯器

1. 安裝套件

使用 `npm` 或 `yarn` 安裝 `quill` 及 [`ngx-quill`](https://github.com/KillerCodeMonkey/ngx-quill) 套件

`npm install quill ngx-quill`

> 此處以安裝 `quill@^2.0.3` 與 `ngx-quill@^22.1.2` 版本為例

2. 引入模組（若有 `AppModule` ）

在 Angular 模組中匯入 `QuillModule`

```typescript
import { QuillModule } from 'ngx-quill';

@NgModule({
  imports: [
    QuillModule.forRoot()
  ]
})
```

3. 加入樣式

在 `angular.json` 中引入 Quill 的樣式檔，視需要選定 snow 或 bubble 主題樣式：

```json
"styles": [
  "node_modules/quill/dist/quill.snow.css",
  "src/styles.css",
]
```

或者在全域 SCSS 檔加入

`@import 'node_modules/quill/dist/quill.snow.css';`

4. 加入 toolbar

在 `main.ts` 中提供 toolbar 選項

```typescript
import { provideQuillConfig } from 'ngx-quill/config';

bootstrapApplication(App, {
  providers: [
    provideQuillConfig({
      modules: {
        toolbar: [
          ['bold', 'italic', 'underline', 'strike'],                         // 文字樣式
          [{ 'header': [1, 2, 3, 4, 5, 6, false] }],                         // 字型大小
          ['blockquote', 'code-block'],                                      // 引用與程式碼區塊
          ['link', 'image', 'video', 'formula'],                             // 多媒體嵌入
          [{ 'list': 'ordered'}, { 'list': 'bullet' }, { 'list': 'check' }], // 清單與對齊
          [{ 'script': 'sub'}, { 'script': 'super' }],                       // 上標／下標
          [{ 'indent': '-1'}, { 'indent': '+1' }],                           // 減少縮排／縮排
          [{ 'direction': 'rtl' }],                                          // 文字排列方向
        ];,
      }
    })
  ]
});
```

{% asset_img 1_quill_toolbar.png %}

4. 建立元件

在元件中使用 `quill-editor` 標籤開始編輯，視需求可選擇使用範本驅動表單或響應式表單（如何使用表單可參考[Angular 新手村 - 8. 表單](https://kun-neng.github.io/2025/05/02/Angular-新手村-8-表單)）。

- 範本驅動表單
  ```html
  <quill-editor [(ngModel)]="content"></quill-editor>
  ```

- 響應式表單
  `editor-form.component.html`
  ```html
  <form [formGroup]="form" (ngSubmit)="onSubmit()">
    <quill-editor class="w-full" formControlName="editor"></quill-editor>
    <button type="submit" [disabled]="form.invalid">Submit</button>
  </form>
  ```

  `editor-form.component.ts`
  ```typescript
  import { ChangeDetectionStrategy, Component, inject, OnInit } from '@angular/core';
  import { FormBuilder, FormControl, ReactiveFormsModule, Validators } from '@angular/forms';
  import { QuillEditorComponent } from 'ngx-quill';
  import { debounceTime, distinctUntilChanged } from 'rxjs';

  @Component({
    selector: 'app-editor-form',
    standalone: true,
    imports: [ReactiveFormsModule, QuillEditorComponent],
    templateUrl: './editor-form.component.html',
    changeDetection: ChangeDetectionStrategy.OnPush,
  })
  export class EditorFormComponent implements OnInit {
    fb = inject(FormBuilder);

    form = this.fb.group({
      editor: new FormControl('', Validators.required)
    });

    ngOnInit() {
      this.form.controls.editor.valueChanges.pipe(
        debounceTime(400),
        distinctUntilChanged(),
      ).subscribe(value => {
        console.log('Quill Editor Content:', value);
      });
    }

    onSubmit() {
      if (this.form.valid) {
        console.log('Form Submitted!', this.form.value);
      } else {
        console.log('Form is invalid.');
      }
    }
  }
  ```

<!-- more -->

5. 監聽事件

```html
<quill-editor
  (onEditorCreated)="onEditorCreated($event)"
  (onContentChanged)="onContentChanged($event)">
</quill-editor>
```

主要監聽兩個事件
- `onEditorCreated` ：當 `ngx-quill` 建立完 Quill 編輯器後，會呼叫此事件來取得 Quill editor 實例。在取得實例後，可以偵測 Quill 編輯器的文字變化或操作等
- `onContentChanged` ：當編輯器內容發生變化時觸發（內部行為是 `ngx-quill` 對 Quill 的 `text-change` 事件做包裝，並把內容同步到 Angular ）

```typescript
onEditorCreated(quill: any) {
  // 綁定原生 text-change 事件
  quill.on('text-change', () => {
    console.log('文字變動（來自 Quill 原生）');
  });

  // 取消 autofocus
  quill.blur();
}

onContentChanged(event: any) {
  // 從 event 取得來源為 user 或 api
  if (event.source === 'user') {
    console.log('使用者正在輸入');
  }
}
```

### 在 Angular 中使用 Quill 內容顯示器

`viewer.component.html`
```html
<quill-view-html [content]="editorContent"></quill-view-html>
```

`viewer.component.ts`
```typescript
import { ChangeDetectionStrategy, Component, Input } from '@angular/core';
import { ReactiveFormsModule } from '@angular/forms';
import { QuillViewHTMLComponent } from 'ngx-quill';

@Component({
  selector: 'app-viewer',
  standalone: true,
  imports: [ReactiveFormsModule, QuillViewHTMLComponent],
  templateUrl: './viewer.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class ViewerComponent {
  content = '<p><strong><em><u>Hello World!</u></em></strong></p>';
}
```

### Mention 擴充功能

透過監聽內容變化的事件，開發者能夠在內容變更、選取範圍異動等時機點觸發自訂邏輯。例如當使用者輸入特定字符（例如 `@` ）時，我們可以判斷游標位置顯示提及（ mention ）選單。

1. 安裝套件

`npm install quill-mention`

> 此處以安裝 `quill-mention@^6.1.1` 版本為例

2. 加入樣式

在 `angular.json` 中引入 Quill Mention 的樣式檔：

```json
"styles": [
  "node_modules/quill/dist/quill.snow.css",
  "node_modules/quill-mention/dist/quill.mention.css",
  "src/styles.css",
]
```

或者在全域 SCSS 檔加入

`@import url('node_modules/quill-mention/dist/quill.mention.css');`

3. 加入 mention 至元件

```typescript
import { ChangeDetectionStrategy, Component, inject } from '@angular/core';
import { QuillEditorComponent } from 'ngx-quill';
import Quill from 'quill';
import { Mention, MentionBlot } from "quill-mention";
import { take } from 'rxjs';

Quill.register({
  'modules/mention': Mention,
  'blots/mention': MentionBlot,
}, true);

@Component({
  selector: 'app-mention-form',
  standalone: true,
  imports: [QuillEditorComponent],
  template: `<quill-editor class="w-full" [modules]="modules"></quill-editor>`,
  changeDetection: ChangeDetectionStrategy.OnPush,
})
export class EditorMentionComponent {
  bookService = inject(BookService);
  books$ = this.bookService.books$;

  modules = {
    toolbar: false,
    mention: {
      allowedChars: /^[A-Za-z\sÅÄÖåäö]*$/,
      mentionDenotationChars: ['@'],
      positioningStrategy: 'fixed',
      defaultMenuOrientation: 'top',
      mentionContainerClass: 'w-40 p-2 bg-white shadow-lg border border-gray-300 rounded-lg',
      mentionListClass: 'max-h-60 overflow-y-auto',
      listItemClass: 'p-2 hover:bg-gray-100 cursor-pointer',
      source: (searchTerm: string, renderList: Function) => {
        this.books$.pipe(
          take(1),
        ).subscribe(books => {
          const booksJson = books.map(book => ({
            id: book.id,
            value: book.name,
          }));

          if (searchTerm.length === 0) {
            renderList(booksJson, searchTerm);
          } else {
            const matches = booksJson.filter(bookJson => 
              bookJson.value.toLowerCase().includes(searchTerm.toLowerCase())
            );
            renderList(matches, searchTerm);
          }
        });
      },
      renderItem: (item: { id: string; value: string }) => {
        return item.value;
      },
      onOpen: () => {
        console.log('選單被開啟');
      },
      onSelect: (item: DOMStringMap, insertItem: any) => {
        console.log('項目被選取');
        insertItem(item);
      }
    }
  }
}
```

在輸入框中輸入 `@` 會自動觸發選單，在選取後於編輯器中插入 mention 項目。若想要調整選單位置，可在 `mentionContainerClass` 額外加入自定義的 class 名稱，例如 `ql-mention-list-container` ，並在 `onOpen` 函式中進行調整，例如：

```typescript
onOpen: () => {
  const mentionContainer = document.querySelector('.ql-mention-list-container') as HTMLElement;

  if (!mentionContainer) {
    return;
  }

  const top = mentionContainer.offsetHeight - 4;
  mentionContainer.style.top = `${ top }px`;
}
```

4. 設定 mention 項目樣式

在全域 SCSS 檔加入

```css
.mention {
  color: green;
  font-weight: 600;
  text-decoration: none;
  cursor: pointer;

  &:hover {
    background-color: lightgreen;
    box-shadow: 0 1px 2px rgba(0, 0, 0, 0.06);
  }
}
```

最終呈現效果如下：

{% asset_img 2_quill_mention.png %}
