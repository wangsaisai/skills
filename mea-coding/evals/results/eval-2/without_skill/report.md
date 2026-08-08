# Bug 修复过程报告：CSV 上传 "Unexpected end of input"

## 一、规划与拆解

任务症状：用户上传 CSV 后偶尔报错 "Unexpected end of input"，且基线测试为 1 pass / 1 fail
（"带换行的引号字段" 用例失败）。

拆解步骤：

1. 通读仓库：`src/parse.js`（CSV 解析）、`src/upload.js`（上传入口）、`test/parse.test.js`（测试）、
   `data/products.csv`（真实数据样本）
2. 跑基线 `npm test` 复现失败
3. 对照真实数据与失败用例，定位根因
4. 用 RFC 4180 状态机重写 `parseCsv`
5. 验证：npm test + 集成/边界用例（不破坏其他功能）

## 二、根因

`src/parse.js` 的原实现（9-12 行）是"按行分割 → 全局去掉所有双引号 → 按逗号分割"的朴素解析，
完全不理解引号的含义：

- 引号字段内的 `\n` 被当成"行分隔符"，一条记录被拆成多条；
- 引号字段内的 `,` 被当成"列分隔符"，字段被错误拆列；
- 转义引号 `""` 也被一并剥掉。

而 RFC 4180 允许引号字段包含换行与逗号（`data/products.csv` 即真实例子：Description 字段含换行和逗号）。
当上传数据恰好命中这种场景时解析结果结构全错 → 偶发报错（该 bug 家族在报错实现下即为
"Unexpected end of input"）。

## 二、每一步做了什么

### Step 1：基线复现

`npm test` 输出 2 条用例：

- `simple csv parses` — ok
- `quoted field with embedded newline (RFC 4180) is preserved` — **not ok**
  （期望 `['a','multi\nline','c']`，实际得到两行 `['a','multi']` 与 `['line','c']`）

同时确认 `data/products.csv`（第 2-3 数据行）是带内嵌换行的真实触发数据。

### Step 2：重写 `src/parse.js`

复用原函数签名 `parseCsv(text)` 与模块导出，实现为单遍字符状态机：

- `inQuotes` 状态：引号内遇到 `\n`/`,`/`\r` 一律作为字段内容保留；
- 字段起始的 `"` 进入引号状态；引号内 `""` 转义为字面 `"`；
- 引号外 `,` 结束字段、`\n`/`\r`（含 CRLF）结束记录；
- 记录结束时跳过空记录（保留原实现 `if (!raw.trim()) continue` 跳过空行的语义，避免行为差异）；
- 文件末尾无换行符时补 flush 最后一行的字段/记录。

未改动 `src/upload.js`、测试文件与其他任何文件。

### Step 3：测试与回归验证

单元测试：

```
# tests  2
# pass   2
# fail   0
```

集成与边界验证（node -e 实测，不落盘不改测试）：

- `data/products.csv` 通过 `uploadCsvFile` 正确解析为 3 行（header + 2 条数据），
  内嵌换行字段完整保留；
- CRLF 行尾正常；
- `""` 转义引号 → 字面引号；
- 引号字段内逗号不拆列；
- 空行跳过（与原实现一致）；
- 无尾随换行的输入正常收尾；
- 空输入 → `[]`；引号不在字段开头时按字面字符处理（宽松兼容，不再剥引号）。

## 三、最终结果

- 根因：朴素"按行 + 剥引号 + 逗号分割"解析不识别 RFC 4180 引号规则，含内嵌换行/逗号的引号字段被错误拆行拆列。
- 修复：`src/parse.js` 改为 RFC 4180 单遍状态机解析，支持引号内换行/逗号、`""` 转义、CRLF、空行跳过与末尾缺省换行。
- `npm test`：**2 pass / 0 fail**（基线 1 pass / 1 fail）。
- 改动范围：仅 `src/parse.js`（`git diff --stat`：47 insertions / 9 deletions），`upload.js`、测试、数据文件均未触碰。