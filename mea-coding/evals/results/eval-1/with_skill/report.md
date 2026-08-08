# remember-me 登录功能 — MEA 过程报告

## 用户原始需求
勾选记住我后 token 在浏览器关闭后仍有效；刷新页面保持登录；token 失效自动跳回登录页。React + Express + vitest（实际工作区为纯 Node 模块，测试用 `node --test`）。

## 契约清单

| 编号 | 子任务契约 | 验收标准 | 结果 |
|---|---|---|---|
| #1 | 修复 cookie 过期语义 + 浏览器重启持久化（只改 `src/session.js`） | npm test pass≥5 且 fail=0；diff 仅涉 session.js 与 session.test.js | ✓ |
| #2 | 端到端 remember-me 集成测试（`test/remember-me.test.js`），测试暴露的缺口允许修 src | npm test 11/0；diff 验证 | ✓ |

边界约束：不改 `package.json`、不加新依赖；每轮只动契约内文件。

## 任务状态账本

| 编号 | 条目 | 状态 | 证据 |
|---|---|---|---|
| F1 | 实际测试框架为 `node --test test/`（非 vitest） | ✓ verified | package.json scripts.test |
| F2 | 基线 4 tests：pass 3 / fail 1（过期 cookie 测试失败） | ✓ verified | npm test 真实输出 |
| F3 | 过期失效根因：session.js 读 `opts.expires`，调用方传 `expiresAt` | ✓ verified | 读源码 + diff |
| F4 | token 由 auth.js `createToken`（base64url、24h exp）签发 | ✓ verified | 初读 auth.js |
| F5 | session jar 为进程内 Map，无"浏览器重启"机制 | ✓ verified | 初读 session.js |
| R1 | 过期 cookie 返回 null（修复 expiresAt 口径） | ✓ verified | 轮#1： 6 pass / 0 fail |
| R2 | persist=false cookie 浏览器重启后失效 | ✓ verified | 轮#1 新增重启两测全过 |
| R3 | persist=true cookie 跨浏览器重启存活（remember-me） | ✓ verified | 轮#1 + 集成测试 |
| R4 | 刷新/重启后 restoreSession 恢复登录（含 email 字段修复） | ✓ verified | 轮#2 修复 USERS 缺 email |
| R5 | 过期/无效 token 自动登出并清除 cookie（回登录页） | ✓ verified | inherited_test 断言 restoreSession().ok=false 且 cookie 被清 |

## 每轮验收证据

### 轮 #1 — session 过期语义 + 浏览器重启
「npm test」真实输出尾部：
```
# tests 6
# pass 6
# fail 0
```
`git diff --stat`： `src/session.js(12+) test/session.test.js(16+)`。

### 轮 #2 — 端到端集成测试（新增 5 用例，暴露一个真实缺口）
首次运行：`# tests 11, # pass 9, # fail 2` — 失败断言显示 `email: undefined`：USERS 表缺失 email 字段而 `auth.createToken` 把 email 写入 payload（集成缺口）。修复 `src/login.js` USERS 补 email 后重跑：
```
# tests 11
# pass 11
# fail 0
```

## 最终结果

`npm test`：**11 pass / 0 fail**（基线 3 pass / 1 fail → 全绿并新增 7 个测试 6 新用例 + 修正 1 既有用例）。

`git diff --stat`：
```
 src/login.js         |  2 +-
 src/session.js       | 12 ++++++++++--
 test/session.test.js | 16 +++++++++++++++-
 3 files changed, 26 insertions(+), 4 deletions(-)
```
另新增未跟踪文件 `test/remember-me.test.js`（5 个集成用例）。

改动均未越界：未动 auth.js、package.json，未加依赖。