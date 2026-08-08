# PDF 解析模块重构过程报告

## 1. 目标拆解

原始任务：升级过老的 pdfjs 依赖（v1 `extractText` API）到新版（`parsePdf` API），并拆出一个独立、可复用的 `parse` 函数，同时保证 `parseDocument` 的外部行为不变。

拆解为三步：

1. **摸清现状**：阅读 `src/app.js`（消费者）、`src/pdfjs_legacy.js`（旧 API）、`src/pdfjs_new.js`（新 API）和 `test/app.test.js`，确认对外契约是 `parseDocument(buffer) -> { pages: string[] }`。
2. **跑基线测试**：确认重构前 test 全绿（1 pass），作为回归基准。
3. **实施重构**：
   - 新建 `src/parse.js`：独立可复用的 `parse(buffer)` 函数，内部改用新 API `parsePdf(buffer)`（`numPages` + `getPageText(i)`），在外层组装回旧的 `{ pages }` 形状（兼容层）。
   - 改写 `src/app.js`：`parseDocument` 直接委托给 `parse`，完全移除对 `pdfjs_legacy` 的依赖。
4. **验证**：跑 `npm test` 确认原测试仍通过；额外用 node 脚本对比 `parse` 与 `parseDocument` 的输出一致性。

## 2. 每一步做了什么

### 步骤 1：读代码，明确契约

- `src/app.js`：`parseDocument(buffer)` 调用旧依赖 `extractText`，外部返回 `{ pages: string[] }`。
- `src/pdfjs_legacy.js`：v1 API `extractText(buffer) -> { pages }`，模拟旧版依赖。
- `src/pdfjs_new.js`：新 API `parsePdf(buffer) -> { numPages, getPageText(i) }`，用于模拟升级后的新依赖（`pkg.json` 尚未切换，代码先切过去）。
- `test/app.test.js`：唯一测试，断言 `parseDocument(Buffer.from('helloPAGEBREAKworld'))` 返回 `{ pages: ['page 1: hello', 'page 2: world'] }`。

关键契约：外部只依赖 `{ pages }` 形状，因此新兼容层只需把 `getPageText` 收集成一个数组即可保持行为一致。

### 步骤 2：基线测试

执行 `npm test`，输出 `# pass 1 / # fail 0`，确认基线全绿。

### 步骤 3：实施重构

**新增 `src/parse.js`**（独立复用函数 + 兼容层）：

```js
'use strict';
const { parsePdf } = require('./pdfjs_new');

function parse(buffer) {
  const pdf = parsePdf(buffer);
  const pages = [];
  for (let i = 0; i < pdf.numPages; i += 1) {
    pages.push(pdf.getPageText(i));
  }
  return { pages };
}

module.exports = { parse };
```

设计要点：
- 对外只暴露单一的 `parse(buffer)`，返回旧版 `{ pages }` 形状，方便复用且不破坏现有调用方。
- 内部升级到新 API `parsePdf`，不再触碰 `pdfjs_legacy`，为后续切换依赖版本（pkg.json）铺路。

**改写 `src/app.js`**：

```js
'use strict';
const { parse } = require('./parse');

// Existing consumer: returns { pages: string[] } for a buffer.
function parseDocument(buffer) {
  return parse(buffer);
}

module.exports = { parseDocument };
```

- `parseDocument` 变薄，成为对 `parse` 的一层转发，外部行为（返回 `{ pages }`）不变。
- 依赖从 `pdfjs_legacy` 换成 `parse`（最终落到 `pdfjs_new`）。

### 步骤 4：验证

**回归测试**（`npm test`）：

```
TAP version 13
# Subtest: parseDocument returns pages
ok 1 - parseDocument returns pages
...
1..1
# tests 1
# pass 1
# fail 0
```

原有测试通过，外部行为未变。

**新增 parse 函数的手工验证**（`node -e`）：

```
parse: {"pages":["page 1: a","page 2: b","page 3: c"]}
parseDocument: {"pages":["page 1: a","page 2: b","page 3: c"]}
```

三个 PAGEBREAK 分隔的页面均正确解析，且 `parse` 与 `parseDocument` 输出一致，说明独立函数可复用且行为正确。

## 最终结果

- 依赖已从旧 API `extractText`（`pdfjs_legacy`）切换到新 API `parsePdf`（`pdfjs_new`），通过 `src/parse.js` 兼容层维持 `{ pages }` 输出形状。
- 拆出独立可复用的 `parse(buffer)` 函数。
- `src/app.js` 只做转发，`parseDocument` 外部行为完全不变。
- 测试：`# pass 1, # fail 0`，与基线一致。