# mea-coding skill

长任务、多步骤编码任务的"管理-执行-审计"（MEA）工作流。可执行内容见 `SKILL.md`（运行时唯一加载的文件）。

## 目录说明

| 路径 | 用途 | 运行时是否加载 |
|---|---|---|
| `SKILL.md` | 工作流本体 | 是（唯一加载） |
| `evals/evals.json` | 评测 prompt 与各 iteration 结果摘要 | 否 |
| `evals/results/` | iteration-1 原始 A/B 报告留档 | 否 |

## 实测结果（iteration-1，2026-08-08）

3 个 fixture 任务 × 带/不带 skill 的 A/B 对比（隔离工作区 + git 基线）。完整报告见 `evals/results/A-B-comparison.md`，各运行逐轮报告见 `evals/results/eval-{1,2,3}/{with_skill,without_skill}/report.md`。

结论摘要：

- 功能结果持平：6 次运行测试全绿（eval-1 基线 3/1 → with 11/0、without 9/0；eval-2 1/1 → 2/0；eval-3 保持 1/0）。
- 过程差异可控：带 skill 的报告全部含契约清单 + 状态账本 + 每轮审计证据（grep 计数 11–14）；不带 skill 的 0–5。
- 带 skill 差分收益：测试产出更多且纳入 git（eval-1 新增 7 个用例 vs 对照组未跟踪）；重构 diff 更小（eval-3 明确"行为不变"硬性验收）。
- SKILL.md 中失败模式第 7、8 条即来自本轮实测观察。

## 后续迭代

- 跑新 iteration 时：重建隔离 fixture 工作区（复用 `evals/evals.json` 的 prompt），结果摘要写回 `evals.json`，原始报告归档到 `evals/results/iteration-N/`。