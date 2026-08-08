# MEA skill A/B 对比报告（iteration-1）

日期：2026-08-08
方法：3 个真实任务 fixture × 2 组（with-skill 遵循 `~/.agents/skills/mea-coding/SKILL.md` / without-skill 常规习惯），
隔离工作区 + git init 基线，各自独立完成并写 `outputs/report.md`。

## 结果总览

| eval | 任务 | 基线 | with-skill | without-skill |
|---|---|---|---|---|
| 1 | remember-me 登录功能 | 3 pass/1 fail | 11 pass/0 fail | 9 pass/0 fail |
| 2 | CSV 引号内换行 bug 定位修复 | 1 pass/1 fail | 2 pass/0 fail | 2 pass/0 fail |
| 3 | PDF 模块重构（行为不变） | 1 pass/0 fail | 1 pass/0 fail | 1 pass/0 fail |

功能性指标：三组全部修复成功、全绿。差异在过程质量。

## 过程质量对比

| 维度 | with-skill | without-skill |
|---|---|---|
| 契约/验收标准/审计证据 出现次数（报告内 grep 计数） | eval-1: 14、eval-2: 12、eval-3: 11 | eval-1: 0、eval-2: 5、eval-3: 3 |
| 开工前是否先立可核验验收标准 | 是（表格化 R1-R3 + 边界约束） | 否，直接列拆解步骤 |
| 每轮是否留下独立验收证据 | 是（每轮附真实 npm test 输出、git diff --stat、边界断言输出） | 部分（mostly 结尾一次性验证） |
| 是否有"状态账本/审计"结构 | 是（任务状态账本 + 总审计） | 否 |
| 改动范围控制 | eval-1: 3 files + 新增 remember-me.test.js（契约外测试）；eval-2: 仅 parse.js；eval-3: app.js + 新增 parse.js | eval-1: 3 files + login.test.js（未跟踪）；eval-2: 仅 parse.js；eval-3: app.js + parse.js（未跟踪） |
| 额外产出 | 新增完整集成测试 remember-me.test.js（7 个新用例） | 新增 login.test.js（未跟踪） |

## 关键观察

1. **功能结果趋同**：任务可完成性上两者持平——这三个 fixture 的 bug 面较小，常规 agent 也能搞定。skill 的价值不在"能否修好"，而在**过程可核验性**。
2. **skill 的标记行为确实触发**：with-skill 组报告全部呈现契约清单、状态账本、每轮审计证据；without-skill 组 0-5 处，且多为顺带提"验证了"，没有结构化账本。
3. **有意义的测试差异**：eval-1 with-skill 写了 7 个新集成用例（remember-me.test.js）并全部纳入 git；without-skill 的 login.test.js 未跟踪且用例更少（11 vs 9 pass）。eval-3 with-skill 明确列出"行为不变"硬性验收且 diff 最小化。
4. **置信度问题**：没有收集 token 计数 / 时长（本轮未做计时遥测），无法判断效率差异。

## 对 SKILL.md 的后续建议（待用户确认）

- 优点已验证：契约→执行→审计 结构可被模型稳定遵循。
- 弱点待补：契约外新增测试文件应显式纳入契约（with-skill 的 remember-me.test.js 其实是契约外变更，但被接受为合理）。
- 可选：加入"每轮预算/轮数上限"，防止长任务跑飞。
