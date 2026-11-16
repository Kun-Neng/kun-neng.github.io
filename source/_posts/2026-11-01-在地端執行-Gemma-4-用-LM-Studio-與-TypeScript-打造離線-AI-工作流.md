---
title: 在地端執行 Gemma 4：用 LM Studio 與 TypeScript 打造離線 AI 工作流
date: 2026-11-01 10:41:55
tags: [Gemma, LM Studio, AI, LLM]
categories: AI
description: 介紹 Google 開放權重模型 Gemma 4 的多模態、思考模式、原生函式呼叫與 MoE 架構，並以 LM Studio 啟動 OpenAI 相容 API ，用 TypeScript 實作聊天、收據圖片轉 JSON 與專案助理代理程式，附上結構化輸出與工具呼叫的實測踩雷紀錄。
---

把程式碼、合約或客戶資料貼給雲端 AI 之前，多數團隊都會猶豫一下：資料會不會被留存？離線時還能不能用？每次呼叫的費用加起來是多少？ Google 於 2026 年 4 月推出的開放權重模型 **Gemma 4** ，讓這些問題有了不同的答案——最小的版本能在 4 GB 記憶體的裝置上執行，較大的版本在數學與程式能力上已大幅超越上一代的 Gemma 3 27B ，而且採用可商用的 Apache 2.0 授權。本文整理 Gemma 4 值得關注的功能，並透過 [LM Studio](https://lmstudio.ai/) 在本機啟動 OpenAI 相容的 API 伺服器，以 TypeScript 實作三個實際能派上用場的範例：本機聊天、把收據圖片轉成結構化 JSON ，以及能自己讀取專案檔案的代理程式（ agent ）。

> 本文撰寫時（ 2026 年 11 月）， Gemma 4 為最新一代 Gemma 模型。 Google 原本在 ai.google.dev 的 [LM Studio 整合頁面](https://ai.google.dev/gemma/docs/integrations/lmstudio?hl=zh-tw)已標示為淘汰，新版文件移至 [gemma.ai](https://gemma.ai/deployment/lmstudio/) ；本文的 LM Studio 操作與 API 格式則以 [LM Studio 開發者文件](https://lmstudio.ai/docs/developer)為準。

> 本文的操作畫面與實作結果，以 Windows 11 上的 LM Studio 0.4.28 搭配 `google/gemma-4-e4b` （ GGUF Q4_K_M ）、 RTX 3060 Ti 8 GB 實測。

## Gemma 4 有哪些版本

Gemma 4 依照部署環境分成五種尺寸，從手機到工作站都有對應的選擇（資料來源：[Gemma 4 model card](https://gemma.ai/models/gemma-4/model-card/)）：

| 版本 | 架構 | 參數量 | Context | 輸入模態 |
|------|------|--------|---------|----------|
| E2B | Dense | 有效 2.3B （含 embedding 共 5.1B ） | 128K | 文字、圖片、音訊 |
| E4B | Dense | 有效 4.5B （含 embedding 共 8B ） | 128K | 文字、圖片、音訊 |
| 12B Unified | Dense ，無獨立編碼器 | 11.95B | 256K | 文字、圖片、音訊 |
| 26B A4B | MoE | 總計 25.2B ，每次僅啟用 3.8B | 256K | 文字、圖片 |
| 31B | Dense | 30.7B | 256K | 文字、圖片 |

E2B 、 E4B 、 26B A4B 與 31B 於 2026 年 4 月初釋出， 12B Unified 則於 2026 年 6 月補上，是 Gemma 系列第一個支援原生音訊輸入的中型模型（見 [Google 官方部落格](https://blog.google/innovation-and-ai/technology/developers-tools/introducing-gemma-4-12b/)）。

## 值得關注的五個特色

### 小模型也有大能力

E2B 與 E4B 名稱中的「 E 」代表 effective （有效參數）。它們採用**逐層嵌入**（ Per-Layer Embeddings ， PLE ）設計：每一層 decoder 擁有自己的小型 embedding 表，推論時只做查表、不參與矩陣運算，因此實際運算量遠低於總參數量。根據 [Google Developers Blog](https://developers.googleblog.com/bring-state-of-the-art-agentic-skills-to-the-edge-with-gemma-4/) 的數據， E2B 在部分裝置上只需不到 1.5 GB 記憶體，甚至能在 Raspberry Pi 5 上以每秒 7.6 個 token 的速度生成文字。

### MoE ：大模型的知識、小模型的速度

**26B A4B** 是混合專家（ Mixture of Experts ， MoE ）模型，共有 128 個專家，每個 token 只啟用其中 8 個加上 1 個共享專家，實際參與運算的參數約 3.8B 。換句話說，它需要足以放下 25B 參數的記憶體，但生成速度接近 4B 模型。對於有 16 GB 以上顯示記憶體、又希望回應夠快的開發者，這通常是最划算的選擇。

### 原生多模態

全系列都能理解圖片，並可將影片視為逐格影像處理（最長 60 秒、每秒 1 格）； E2B 、 E4B 與 12B 還能接收最長 30 秒的音訊，用於語音轉文字（實作請見 {% post_link 用-llama-cpp-與-TypeScript-實作-Gemma-4-語音轉文字 '用 llama.cpp 與 TypeScript 實作 Gemma 4 語音轉文字' %} ）。實務上常見的任務包括：

- **OCR 與文件解析**：讀取發票、表格、截圖中的文字
- **GUI 元素定位**：回傳畫面元件的 bounding box （座標以 1000 × 1000 為基準）
- **截圖轉程式碼**：依照網頁截圖產生 HTML

圖片的 [token 預算](https://gemma.ai/models/gemma-4/model-card/#5-variable-image-resolution)可在 70 、 140 、 280 、 560 、 1120 之間調整：分類或簡單描述用低預算即可， OCR 、文件解析則建議用高預算以保留細節。

> [官方建議](https://gemma.ai/models/gemma-4/model-card/#4-modality-order)將**圖片放在文字之前**、音訊放在文字之後，模型的表現會比較穩定。不過筆者在 `llama.cpp` 上實測語音轉文字時，音訊放在文字前面的錯字反而略少，詳見 {% post_link 用-llama-cpp-與-TypeScript-實作-Gemma-4-語音轉文字 '語音轉文字的實作' %} 。

### 可開關的思考模式

Gemma 4 內建可設定的[思考（ thinking ）模式](https://gemma.ai/get-started/capabilities/thinking/)：在 system prompt 開頭加入 `<|think|>` 控制 token 即可啟用，模型會先在 `<|channel>thought ... <channel|>` 區塊中推理，再輸出最終答案。數學、程式除錯等需要多步推理的任務適合開啟；簡單的翻譯或摘要關閉即可節省時間。

要注意的是，執行工具的預設值不一定和模型文件相同。例如在 `llama-server` 上實測， Gemma 4 預設就會進入思考模式，必須在請求中明確關閉（詳見 {% post_link 用-llama-cpp-與-TypeScript-實作-Gemma-4-語音轉文字 '語音轉文字的實作' %} ）。若簡單的問題也回得特別慢，可以先檢查思考模式是否被開啟。

在 LM Studio 中，支援推理的模型會在聊天介面顯示可收合的思考內容；透過 API 呼叫時，可在 Developer 設定中讓伺服器把推理內容分離到 `reasoning_content` 欄位，避免與正式回答混在一起（實際選項名稱請依所使用的 LM Studio 版本確認）。

> 多輪對話時，官方建議**不要**把先前的思考內容放回對話歷史；唯一的例外是包含工具呼叫的回合，該回合的思考內容需要保留。

### 原生函式呼叫與系統提示

Gemma 4 原生支援 `system` 角色與函式呼叫（ function calling ），不必再用 prompt 技巧模擬。搭配 128K 至 256K 的 context window 與 140 種以上語言的預訓練資料，它很適合用來打造在本機執行的代理程式。下表節錄 model card 中 instruction-tuned 版本的部分成績，可以看出世代間的差距：

| Benchmark | Gemma 4 31B | Gemma 4 26B A4B | Gemma 4 E4B | Gemma 3 27B |
|-----------|-------------|-----------------|-------------|-------------|
| MMLU Pro | 85.2% | 82.6% | 69.4% | 67.6% |
| AIME 2026 （無工具） | 89.2% | 88.3% | 42.5% | 20.8% |
| LiveCodeBench v6 | 80.0% | 77.1% | 52.0% | 29.1% |
| GPQA Diamond | 84.3% | 82.3% | 58.6% | 42.4% |

## 用 LM Studio 在本機啟動 Gemma 4

LM Studio 是跨平台（ Windows 、 macOS 、 Linux ）的桌面應用程式，底層使用 `llama.cpp` （ GGUF 格式）與 Apple MLX 引擎，同時提供圖形介面與 `lms` 命令列工具。最重要的是，它能把本機模型包裝成 **OpenAI 相容的 API** ，既有的 OpenAI SDK 程式碼只要改一行 `baseURL` 就能切換到本機模型。

### 依硬體選擇版本

LM Studio 上的 Gemma 4 模型下載大小如下（資料來源：[LM Studio Gemma 4 頁面](https://lmstudio.ai/models/gemma-4)），右欄為依下載大小推估的參考配置：

| Model ID | 下載大小 | 參考配置 |
|----------|----------|----------|
| `google/gemma-4-e2b` | 4.20 GB | 一般筆電、無獨立顯示卡 |
| `google/gemma-4-e4b` | 5.90 GB | 8 GB VRAM 顯示卡 |
| `google/gemma-4-12b` | 7.40 GB | 8 GB VRAM 需部分卸載至 CPU ； 12 GB 以上較順暢 |
| `google/gemma-4-26b-a4b` | 15.60 GB | 16 至 24 GB VRAM ，或 32 GB 統一記憶體的 Mac |
| `google/gemma-4-31b` | 19.00 GB | 24 GB VRAM |

每個版本另有 `-qat` 後綴的變體，代表**量化感知訓練**（ Quantization-Aware Training ）：模型在訓練階段就考慮了低位元量化的誤差，因此在相同大小下通常比一般量化版本保有更好的品質，建議優先選用。

### 方法一：圖形介面

1. 從 [LM Studio 下載頁面](https://lmstudio.ai/download)下載並安裝。頁面最上方的 **Bionic** 是另一個[獨立的代理程式應用](https://lmstudio.ai/docs/bionic)，操作方式與本文不同，請選擇下方的 **Download LM Studio** 。
{% asset_img 1_download_lm_studio.png %}
第一次開啟 LM Studio 會先看到歡迎畫面，按下 **Get Started** 後，它會推薦第一個要下載的模型。以筆者的環境為例，推薦的正是 `google/gemma-4-e4b` ，可以直接下載，或按 **Skip for now** 稍後再下載：
{% asset_img 2_lm_studio_get_started.png %}
{% asset_img 3_lm_studio_first_model.png %}

2. 點選左側側邊欄的 **Model Search** ，在彈出的搜尋視窗中輸入 `Gemma 4 E4B` ，選擇 Google 官方的模型。右側的 **Download Options** 會依照你的硬體推薦合適的量化版本（筆者的環境為 `Q4_K_M` ，並標示 **Full GPU Offload Possible** ，代表整個模型都能放進 GPU ），按下 **Download** 下載：
{% asset_img 4_lm_studio_discover_tab.png %}
{% asset_img 5_lm_studio_gemma_4_e4b.png %}
模型資訊中的 **Capabilities** 標示了它在 LM Studio 中可以使用的能力：
- **Vision** （眼睛圖示）：可以輸入圖片，用於後面的實作二
- **Tool Use** （扳手圖示）：支援工具呼叫，用於後面的實作三
- **Reasoning** ：支援前面介紹的思考模式

> 雖然下方 README 提到小模型支援音訊， Capabilities 卻沒有列出音訊，語音轉文字需要改用其他工具（詳見 {% post_link 用-llama-cpp-與-TypeScript-實作-Gemma-4-語音轉文字 '語音轉文字的實作' %} ）。

3. 下載完成後，右下角會跳出通知，按下 **Load Model** 將模型載入記憶體（之後也可以在 **Model Server** 頁面按 **Load Model** 或 `Ctrl + L` 選擇要載入的模型）。旁邊的 **Use in New Chat** 則是在 LM Studio 內建的聊天介面中試用模型，本文的實作都透過 API 呼叫，因此選擇 **Load Model** ：
{% asset_img 6_lm_studio_load_model.png %}
接著點選左側側邊欄的 **Model Server** ，打開上方的 **Status** 開關，顯示 **Running** 即代表本機伺服器已啟動，位址為 `http://127.0.0.1:1234` 。 **Loaded Models** 中的模型標示為 **READY** 後就可以呼叫，右側的 **API Model Identifier** （ `google/gemma-4-e4b` ）就是程式中 `model` 參數要填入的名稱：
{% asset_img 7_lm_studio_start_server.png %}

> **注意：** 按視窗右上角的 X 只會把 LM Studio 縮到系統匣，伺服器與已載入的模型仍會繼續執行，並持續占用記憶體與 VRAM 。不使用時，請在系統匣的 LM Studio 圖示按右鍵選擇 **Quit** ，或用下方的 `lms unload --all` 與 `lms server stop` 停止。

### 方法二：`lms` 命令列

偏好終端機或需要寫進腳本時，可以使用 `lms` CLI 。 `lms` 隨 LM Studio 一起安裝，不需要另外下載，但[必須先開啟過一次 LM Studio](https://lmstudio.ai/docs/cli) ，它才會把 `lms` 所在的 `~/.lmstudio/bin` 加入 PATH 。已經開著的終端機讀不到新的 PATH ，要重新開啟；在 VS Code 的終端機裡使用時，要把整個 VS Code 重新啟動：

```bash
# 下載模型
lms get google/gemma-4-e4b

# 列出本機已下載的模型
lms ls

# 載入模型，並將 context 長度設為 16K
lms load google/gemma-4-e4b --context-length 16384

# 啟動 API 伺服器（預設埠號 1234）
lms server start

# 查看目前已載入的模型與 context 長度
lms ps

# 用完後卸載模型並停止伺服器
lms unload --all
lms server stop
```

伺服器啟動後，用以下指令確認模型已可供呼叫：

```bash
curl.exe http://localhost:1234/v1/models
```

{% asset_img 8_curl.png %}

> Context 長度越長， KV cache 佔用的記憶體就越多。雖然 Gemma 4 最高支援 256K ，日常使用設定 8K 至 32K 就足夠，需要分析長文件時再調高。從圖形介面按 **Load Model** 載入時，預設 context 長度為 8K （可用 `lms ps` 確認），若要調整，可以在 **Model Server** 頁面右側的 **Load** 分頁修改後重新載入。

{% asset_img 9_lms_ps.png %}

## 實作一：本機聊天

以下範例都使用 Node.js 20 以上版本與官方 [`openai`](https://www.npmjs.com/package/openai) 套件。先建立專案：

```bash
mkdir gemma4-local
cd gemma4-local
npm init -y
npm pkg set type=module   # 啟用 ESM ，才能使用 top-level await
npm i openai
npm i -D tsx @types/node  # 直接執行 TypeScript
```

接著在專案根目錄建立 `tsconfig.json` ：

```json
{
  "compilerOptions": {
    "module": "nodenext",
    "types": ["node"],
    "noEmit": true
  }
}
```

`tsx` 執行時不做型別檢查，這個檔案是給編輯器用的。 TypeScript 6.0 起 `types` 的預設值改為 `[]` ，少了 `"types": ["node"]` ，編輯器會在後面的 `node:fs/promises` 等 import 上顯示找不到名稱的錯誤。

建立 `chat.ts` ：

```typescript
// chat.ts
import OpenAI from 'openai';

const client = new OpenAI({
  baseURL: 'http://localhost:1234/v1', // 指向 LM Studio
  apiKey: 'lm-studio',                 // 本機伺服器不驗證金鑰，填任意字串即可
});

const completion = await client.chat.completions.create({
  model: 'google/gemma-4-e4b',
  messages: [
    { role: 'system', content: '你是資深 Angular 工程師，請以臺灣用語的繁體中文回答。' },
    { role: 'user', content: '用三點說明 Signal 與 RxJS 各自適合的使用時機。' },
  ],
  temperature: 1.0, // 官方建議的取樣參數
  top_p: 0.95,
});

console.log(completion.choices[0].message.content);
```

執行：

```bash
npx tsx chat.ts
```

{% asset_img 10_chat_message_content.png %}

**說明：**
- `baseURL` 與 `apiKey` 是唯一和呼叫 OpenAI 雲端不同的地方，其餘程式碼完全相同。這代表既有的工具鏈（ LangChain 、 Vercel AI SDK 等）大多也能直接接上本機模型。
- `model` 填入 `lms ls` 顯示的 model key 。
- Gemma 4 官方建議的取樣參數為 `temperature = 1.0` 、 `top_p = 0.95` 、 `top_k = 64` ； `top_k` 不在 OpenAI 的標準參數中，可在 LM Studio 的模型設定裡調整。

**實際執行時的注意事項：**
- 輸出中的 `**` 、 `*` 是模型回傳的 Markdown 原文， `console.log` 不會轉換格式，這是正常現象。要在網頁上顯示時，再交給 Markdown 渲染套件處理。
- 提示要求「用三點說明」，筆者執行時模型只列出兩點再加上總結。 4B 等級的模型不一定會精確遵守數量、格式等細節指示，必要時可以重新產生，或改用較大的模型。
- 雖然 system prompt 要求臺灣用語，輸出仍出現「用戶」（臺灣慣用「使用者」）等用語。若用詞很重要，可以在提示中直接列出要使用的詞彙。

## 實作二：收據圖片轉成記帳 JSON

這個範例結合**多模態**與**結構化輸出**：讀入一張收據照片，要求模型依照 JSON Schema 回傳店家、日期、品項與分類，結果可以直接寫入資料庫或記帳 App 。由於整個流程都在本機進行，消費紀錄不會離開你的電腦。

LM Studio 的結構化輸出使用語法約束取樣（ GGUF 模型透過 `llama.cpp` 的 grammar ， MLX 模型透過 Outlines ），能**保證**輸出符合 Schema 格式（見 [Structured Output 文件](https://lmstudio.ai/docs/developer/openai-compat/structured-output)）。

```typescript
// receipt.ts
import { readFile } from 'node:fs/promises';
import OpenAI from 'openai';

const client = new OpenAI({ baseURL: 'http://localhost:1234/v1', apiKey: 'lm-studio' });

// 將圖片轉成 base64 data URL （這裡以 JPEG 為例， PNG 或 WebP 請改成 image/png 、 image/webp ）
const image = await readFile(process.argv[2] ?? './receipt.jpg');
const dataUrl = `data:image/jpeg;base64,${image.toString('base64')}`;

// 定義期望的輸出結構
const receiptSchema = {
  type: 'object',
  properties: {
    store: { type: 'string' },
    date: { type: 'string', format: 'date' }, // 強制輸出 YYYY-MM-DD
    items: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          name: { type: 'string' },
          quantity: { type: 'integer' },
          price: { type: 'number' },
        },
        required: ['name', 'quantity', 'price'],
      },
    },
    total: { type: 'number' },
    category: { type: 'string', enum: ['餐飲', '交通', '購物', '其他'] },
  },
  required: ['store', 'date', 'items', 'total', 'category'],
};

const completion = await client.chat.completions.create({
  model: 'google/gemma-4-e4b',
  messages: [
    {
      role: 'user',
      content: [
        { type: 'image_url', image_url: { url: dataUrl } }, // 圖片放在文字之前
        { type: 'text', text: '請讀取這張收據，整理成記帳資料。' },
      ],
    },
  ],
  response_format: {
    type: 'json_schema',
    json_schema: { name: 'receipt', strict: true, schema: receiptSchema },
  },
});

// 結構化輸出仍以字串形式回傳，需要自行 parse
const receipt = JSON.parse(completion.choices[0].message.content ?? '{}');
console.log(receipt);
```

執行：

```bash
npx tsx receipt.ts ./my-receipt.jpg
```

{% asset_img 11_receipt_format_date.png %}

**說明：**
- `date` 使用 `format: 'date'` ，語法約束會強制輸出 `YYYY-MM-DD` 格式的日期。
- `category` 使用 `enum` 限制可選值，模型只能從四個分類中挑選，後續程式不必處理意料之外的字串。
- `required` 確保每個欄位都會出現，避免下游程式遇到 `undefined` 。
- `content` 改成陣列，即可在同一則訊息中混合圖片與文字。

### 踩雷紀錄：`description` 不會被強制

一開始把 `date` 寫成以下，想用說明文字告訴模型日期格式規範：

```typescript
date: { type: 'string', description: '格式為 YYYY-MM-DD' },
```

結果模型照著收據上的寫法輸出了 `'2024年8月7日'` 。下圖左邊是收據，右下是當時 Gemma 4 E4B 的輸出；右上則是在 [Gemini App](https://gemini.google.com/) 網頁版以 3.5 Flash-Lite 讀取同一張收據的結果，供對照參考：

{% asset_img 12_receipt_message_content.png %}

原因是語法約束只會強制 JSON Schema 中的結構性規則（型別、 `enum` 、 `required` 、 `pattern` 、 `format` 等）， `description` 只是給模型的提示，模型可以不遵守。假設以「請把 date 原樣填入『2024年8月7日』這幾個中文字，絕對不要轉換成其他格式。」的提示測試三種寫法，各執行兩次：

| `date` 的寫法 | 輸出 |
|---|---|
| `description: '格式為 YYYY-MM-DD'` | 兩次都是 `2024年8月7日` |
| `pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}$'` | 兩次都是 `2024-08-07` |
| `format: 'date'` | 兩次都是 `2024-08-07` |

即使提示明確要求不要轉換，使用 `pattern` 或 `format` 時仍輸出規定的格式。因此，**一定要遵守的格式請寫成 Schema 規則，不要只寫在 `description` 裡**。

### 格式正確不等於內容正確

語法約束只保證「格式正確」，不保證「內容正確」。同一張收據、同一個模型，每次執行的結果都可能不同。 LM Studio 文件也提醒 7B 以下的模型在結構化輸出上能力較有限，若辨識錯誤率偏高，可改用 `google/gemma-4-12b` 或 `google/gemma-4-26b-a4b` 。無論使用哪個模型，都建議在程式中加上驗證邏輯，例如檢查品項加總是否等於總計：

```typescript
const sum = receipt.items.reduce((acc: number, item: { quantity: number; price: number }) => acc + item.quantity * item.price, 0);
if (sum !== receipt.total) {
  console.warn(`品項加總 ${sum} 與總計 ${receipt.total} 不符，請人工確認`);
}
```

## 實作三：讀取專案檔案的代理程式

最後一個範例展示 Gemma 4 的**函式呼叫**能力：提供兩個唯讀工具給模型，讓它自行決定何時讀取 `package.json` 或原始碼檔案，再根據結果回答問題。這就是 AI 代理程式最核心的「思考 → 呼叫工具 → 觀察結果 → 再思考」迴圈。

以下三個步驟的程式碼都寫在同一個 `agent.ts` 檔案中，依序接續在前一段程式碼後面。

### 步驟 1：宣告工具

```typescript
// agent.ts
import { readFile } from 'node:fs/promises';
import path from 'node:path';
import OpenAI from 'openai';
import type {
  ChatCompletionMessageParam,
  ChatCompletionTool,
} from 'openai/resources/chat/completions';

const client = new OpenAI({ baseURL: 'http://localhost:1234/v1', apiKey: 'lm-studio' });
const PROJECT_ROOT = path.resolve(process.argv[2] ?? process.cwd());

const tools: ChatCompletionTool[] = [
  {
    type: 'function',
    function: {
      name: 'get_dependencies',
      description: '讀取專案 package.json 中的 dependencies 與 devDependencies',
      parameters: { type: 'object', properties: {} },
    },
  },
  {
    type: 'function',
    function: {
      name: 'read_file',
      description: '讀取專案內指定檔案的內容',
      parameters: {
        type: 'object',
        properties: {
          filePath: { type: 'string', description: '相對於專案根目錄的路徑，例如 src/app/app.config.ts' },
        },
        required: ['filePath'],
      },
    },
  },
];
```

工具的 `description` 就是模型判斷「何時該用哪個工具」的依據，描述越具體，模型越不容易選錯。

### 步驟 2：實作工具

```typescript
// agent.ts （接續步驟 1）
async function runTool(name: string, args: Record<string, string>): Promise<string> {
  try {
    if (name === 'get_dependencies') {
      const pkg = JSON.parse(await readFile(path.join(PROJECT_ROOT, 'package.json'), 'utf-8'));
      return JSON.stringify({ dependencies: pkg.dependencies, devDependencies: pkg.devDependencies });
    }

    if (name === 'read_file') {
      const target = path.resolve(PROJECT_ROOT, args.filePath);
      const relative = path.relative(PROJECT_ROOT, target);
      // 防止模型透過 ../ 讀取專案以外的檔案
      if (relative.startsWith('..') || path.isAbsolute(relative)) {
        return '錯誤：不允許讀取專案目錄以外的檔案';
      }
      return await readFile(target, 'utf-8'); // 要加 await ，讀取失敗時才會進入下方的 catch
    }

    return `錯誤：未知的工具 ${name}`;
  } catch (error) {
    // 找不到檔案、 JSON 格式錯誤、缺少參數等例外，都轉成字串交給模型判斷
    return `錯誤：${(error as Error).message}`;
  }
}
```

工具內發生的任何錯誤都以字串回傳給模型，而不是直接丟出例外：一方面避免整個程式中止，另一方面模型也有機會根據錯誤訊息修正參數再試一次。例如執行時指定的目錄沒有 `package.json` ，模型會收到 `錯誤：ENOENT: no such file or directory, open '.../package.json'` ，並據此回覆找不到檔案，而不是讓程式直接當掉（執行結果見步驟 3 ）。

### 步驟 3：代理迴圈

```typescript
// agent.ts （接續步驟 2）
const messages: ChatCompletionMessageParam[] = [
  { role: 'system', content: '你是 Angular 專案助理。需要資訊時請呼叫工具，不要猜測。以繁體中文回答。' },
  { role: 'user', content: '這個專案使用哪個 Angular 版本？app.config.ts 註冊了哪些 provider？' },
];

const MAX_STEPS = 5; // 避免模型無限呼叫工具

for (let step = 0; step < MAX_STEPS; step++) {
  const { choices } = await client.chat.completions.create({
    model: 'google/gemma-4-e4b',
    messages,
    tools,
  });
  const message = choices[0].message;
  messages.push(message); // 保留含 tool_calls 的回合，模型才知道自己呼叫過什麼

  // 沒有要求呼叫工具，代表模型已經得出最終答案
  if (!message.tool_calls?.length) {
    console.log(message.content);
    break;
  }

  for (const call of message.tool_calls) {
    if (call.type !== 'function') {
      continue;
    }

    console.log(`→ 呼叫 ${call.function.name}(${call.function.arguments})`);

    const args = JSON.parse(call.function.arguments || '{}');
    messages.push({
      role: 'tool',
      tool_call_id: call.id, // 對應到哪一次工具呼叫
      content: await runTool(call.function.name, args),
    });
  }
}
```

對任一 Angular 專案執行：

```bash
npx tsx agent.ts ../my-angular-app
```

{% asset_img 13_tools_result.png %}

如上圖所示，模型先呼叫 `get_dependencies` 取得 `@angular/core` 版本，再呼叫 `read_file` 讀取 `src/app/app.config.ts` ，最後彙整兩者的結果回答。

若指定的目錄不對（例如目錄中沒有 `package.json` ），兩個工具都會回傳錯誤字串，模型會據此說明原因並建議確認路徑，程式不會中止：

{% asset_img 14_tools_file_not_found.png %}

**說明：**
- 這裡沿用前面下載的 `google/gemma-4-e4b` 。多步驟的工具呼叫對模型的推理能力要求較高，若有 16 GB 以上的 VRAM ，建議改用 `google/gemma-4-26b-a4b` ： MoE 架構讓它的速度仍接近 4B 模型，選錯工具或參數的機率也比較低。
- 若偏好 Python ， LM Studio 的 [Python SDK](https://lmstudio.ai/docs/python/agent/act) 提供 `model.act()` ，能直接把 Python 函式當作工具傳入，並自動處理上述迴圈。

## 還能用在哪些地方

除了上述三個範例， Gemma 4 的特性也適合以下情境：

| 情境 | 搭配的特色 | 建議版本 |
|------|------------|----------|
| 內部文件問答、程式碼審查 | 長 context 、資料不出本機 | 26B A4B 、 31B |
| 會議錄音轉逐字稿與摘要 | 音訊輸入 | E2B 、 E4B |
| 網頁截圖轉 HTML 、 UI 自動化測試 | 圖片理解、 GUI 元素定位 | 12B 以上 |
| 手機或 IoT 裝置上的離線助理 | PLE 、低記憶體需求 | E2B 、 E4B |
| 本機 coding agent 的後端模型 | 函式呼叫、思考模式 | 26B A4B |

LM Studio 之外， Gemma 4 也能透過 `llama.cpp` 、 MLX 、 Ollama 或瀏覽器端的 transformers.js 執行；行動裝置則可使用 Google 的 [LiteRT-LM](https://developers.googleblog.com/bring-state-of-the-art-agentic-skills-to-the-edge-with-gemma-4/) 與 Google AI Edge Gallery App 。

## 最佳實踐

- **先選模型，再談 prompt**：依硬體從上方表格挑選，優先使用 `-qat` 版本；模型能完整放進 VRAM 時，速度通常會比部分卸載快上數倍。
- **Context 長度按需設定**：預設值不夠才調高，避免 KV cache 吃光記憶體導致載入失敗。
- **思考模式按任務開關**：推理型任務開啟，簡單任務關閉，並遵守「不把舊思考內容放回歷史」的規則。
- **多模態輸入注意順序**：圖片放在文字前；音訊依官方建議放在文字後，但不同執行工具的實測結果可能不同，最好用自己的資料比較。 OCR 類任務記得提高圖片 token 預算。
- **結構化輸出要再驗證**：格式由語法約束保證，內容仍需商業邏輯檢查（例如品項加總是否等於總額）。
- **工具設計以唯讀、最小權限為原則**：限制可存取的路徑，涉及寫入或刪除的工具應加上人工確認，並設定迴圈的最大步數。
- **留意授權與資料**： Apache 2.0 允許商業使用，但模型輸出的正確性仍需自行把關；地端執行解決了資料外流的疑慮，卻不代表輸出內容可以直接信任。

## 總結

Gemma 4 讓「在自己的電腦上跑一個夠聰明的模型」從實驗變成日常選項： E2B 、 E4B 以 PLE 壓低記憶體需求， 26B A4B 以 MoE 兼顧能力與速度，全系列支援多模態、思考模式與原生函式呼叫。透過 LM Studio 的 OpenAI 相容 API ，現有程式碼只需修改 `baseURL` 就能切換到本機模型，從簡單的聊天、圖片轉 JSON ，到能自行讀取檔案的代理程式都能實現。至於語音轉文字， LM Studio 目前尚無法使用，可改用 `llama.cpp` 實作。

延伸閱讀：

- [Gemma 4 model card](https://gemma.ai/models/gemma-4/model-card/)：完整規格、思考模式格式與 benchmark
- 想把錄音檔轉成逐字稿，可參考 {% post_link 用-llama-cpp-與-TypeScript-實作-Gemma-4-語音轉文字 '用 llama.cpp 與 TypeScript 實作 Gemma 4 語音轉文字' %}
- [Welcome Gemma 4 （ Hugging Face Blog ）](https://huggingface.co/blog/gemma4)：多模態、 GUI 定位與微調範例
- [LM Studio 開發者文件](https://lmstudio.ai/docs/developer)： REST API 、結構化輸出、工具呼叫與 SDK
- 想把本機模型接上開發工具，可參考 {% post_link Angular-CLI-MCP 'Angular CLI MCP' %} 與 {% post_link Angular-WebMCP-讓-AI-代理程式直接操作你的網頁 'Angular WebMCP － 讓 AI 代理程式直接操作你的網頁' %}
- 在 Windows 上建立 Linux 開發環境來執行 `llama.cpp` 等工具，可參考 {% post_link Building-a-Linux-Dev-Environment-with-WSL 'Building a Linux Dev Environment with WSL' %}
- 對代理優先的開發環境有興趣，可參考 {% post_link Introduction-to-the-Google-Antigravity 'Introduction to the Google Antigravity' %}
