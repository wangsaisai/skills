# MEA 重构过程报告：PDF 解析模块升级

工作区：`/Users/bamboo/.agents/skills/mea-coding-workspace/iteration-1/eval-3/with_skill/workspace`

## 契约清单（第 0 步）

| 编号 | 需求 | 可核验验收标准 |
|---|---|---|
| R1 | 升级 pdfjs：app 不再依赖 legacy `extractText`，切换到新 API `parsePdf` | `grep` src/app.js 无 `pdfjs_legacy`/`extractText`；src/parse.js 出现 `require('./pdfjs_new')` + `parsePdf` 调用 |
| R2 | 拆出独立 `parse` 函数方便复用 | 存在新文件 src/parse.js 导出 `parse`，可单独 require 直接调用 |
| R3 | 保持现有功能（`parseDocument` 外部行为）不变 | `npm test` 输出 `# pass 1` 且 `# fail 0`（test/app.test.js 未改动，为验收锚点） |

边界：不改 test/app.test.js、src/pdfjs_legacy.js、src/pdfjs_new.js；不加真实 npm 依赖；不改 package.json（无 lockfile、无网络安装流程，vendored 模拟依赖，改动将不可核验）。

## 任务状态账本

| 编号 | 条目 | 状态 | 证据 |
|---|---|---|---|
| R1 | app.js 升级到新 API | ✓ verified | 第 1 轮审计 grep：app.js 无 legacy 引用，parse.js:2 `require('./pdfjs_new')`、:6 `parsePdf(buffer)` |
| R2 | 独立 parse 函数 | ✓ verified | src/parse.js 存在（14 行），`node -e "require('./src/parse').parse(...)"` 直接调用输出 `{"pages":["page 1: a","page 2: b"]}` |
| R3 | 行为不变 | ✓ verified | 第 1 轮审计 `npm test`：`# pass 1 / # fail 0` |
| F1 | 基线：库测试 1 pass / 0 fail | ✓ verified | 开工前 `npm test` 真实输出 |
| F2 | pdfjs 为本地 vendored mock（pkg.json 无依赖、无 lockfile） | ✓ verified | 读 package.json、src 目录 |

## 每轮验收证据

### 第 1 轮（契约 #1：新增 src/parse.js + 改 src/app.js）

`npm test` 真实输出（审计）：

```
> node --test test/
TAP version 13
# Subtest: parseDocument returns pages
ok 1 - parseDocument returns pages
1..1
# tests 1
# pass 1
# fail 0
# cancelled 0
# skipped 0
# todo 0
# duration_ms 141.847898
```

`git status --short` 与 `git diff --stat`：

```
 M src/app.js
?? src/parse.js
src/app.js | 5 ++---
1 file changed, 2 insertions(+), 3 deletions(-)
```

grep 核验（无 legacy 残留、确认接入新 API）：

```
src/parse.js:2: const { parsePdf } = require('./pdfjs_new');
src/parse.js:6: const doc = parsePdf(buffer);
```

parse 函数独立可用性（只读命令实测）：

```
$ node -e "const {parse}=require('./src/parse'); console.log(JSON.stringify(parse(Buffer.from('aPAGEBREAKb','latin1'))))"
{"pages":["page 1: a","page 2: b"]}
```

## 最终 git diff --stat 与测试结果

```
src/app.js | 5 ++---
+”新增 src/parse.js（14 行）→ 合计：2 文件改动，+16 / -3
```

最终 `npm test`：**1 pass / 0 fail**（测试锚点 test/app.test.js 未改动）。

## 结论

- R1/R2/R3 全部 ✓ verified，无 ⚠ blocked、无未决项。
- 方案概要：新增 `src/parse.js` 作为基于新 API（`parsePdf`）的独立可复用解析函数，`src/app.js` 的 `parseDocument` 改为委托 `parse(buffer)`，外部行为（返回 `{pages}`）保持不变。