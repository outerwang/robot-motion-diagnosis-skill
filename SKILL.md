---
name: robot-motion-diagnosis
description: 机器人运动规划故障诊断工作方法：规划失败与超时分析、运行状态三维度核查、独立验证状态判读、模型版本一致性检查、结构化诊断报告。只组织与解读 robot-motion MCP 工具的真实返回值，不做数值运动学或碰撞计算，不修改任何规划系统。
when-to-use: 当用户询问为什么路径规划失败或搜索超时、某条焊缝转移是否通过独立验证、碰撞发生在哪里、模型或配置版本是否一致、下一步检查什么、历史报告与当前结果是否一致、start_motion_plan 的运行状态，或 CM2B/FR69B/FR76A 任务的验证状态，以及任何需要解读 mcp__robot-motion__ 工具返回值的场合。
user-invocable: false
---

# robot-motion-diagnosis：规划实验故障诊断工作法

## 0. 铁律（每一步都受约束）

1. 只引用工具真实返回值与证据文件；禁止编造任务 ID、Run ID、状态或数值。
2. 四种状态永不混用：**规划器已运行 ≠ 找到候选路径 ≠ 候选通过独立离散检查 ≠ 连续运动/真实执行安全资格**。
3. 搜索超时（TIMEOUT）不是不可行性证明；未验证（NOT_RUN/NOT_VALIDATED）不是通过也不是失败。
4. 历史验证结论只属于其原始记录的路径哈希与输入契约，禁止迁移到新路径。
5. 不建议任何放宽碰撞阈值、关闭 FCL、修改工艺约束或机器人限位的操作。
6. 证据不足时输出 UNRESOLVED 并列出需要工程师补充的具体问题，不猜测。

## 1. 工作流程（按序执行，每步引用工具名）

**第 1 步 确认身份**：确认 Task ID（如 `T1_CM2B`、`ND-SEED-46`、`rm_next_015_fr69b_transfer_expansion/Q1_RRT_E1`）与 Run ID（本地运行 `R-*` 或历史记录）。两者都没有时先调用 `list_tasks` 展示可用任务。

**第 2 步 任务资格**：调用 `get_task_info(task_id)`。检查：工件与模型版本（`model_version`/`input_versions`）、端点、机器人资源、`collision_semantics` 标志、`data_completeness`、`warnings`。版本不一致或数据缺失 → 记为候选故障类 MODEL_VERSION_MISMATCH / INVALID_INPUT。

**第 3 步 运行状态**：
- 本地运行（R-*）：调用 `get_plan_status(run_id)`，读取三个维度：`execution_status`、`plan_result`、`validation_status`，以及 `upstream_integrity`、`warnings`。
- 历史记录：调用 `get_experiment_report(run_id)`，读取 `planner_execution.raw_status`、`independent_validation`、`caveats`。

**第 4 步 独立验证**：当存在候选路径且用户关心其可信度时，调用 `validate_motion(run_id)`。
- PASS：必须同时引用 `candidate_path_hash` 与 `evidence_reference`，说明验证对象与范围（离散网格/状态数），并声明其不等于连续证书。
- NOT_RUN：如实转述 `error_or_incomplete_reason`；对本地新路径，明确"尚未验证"。
- INCOMPLETE / FAIL / ERROR：引用原始 raw_status 与原因。

**第 5 步 故障分类**（必须与真实返回值对应，禁止虚构）：

| 诊断类别 | 判定依据（工具真实字段） |
|---|---|
| INVALID_INPUT | `get_task_info` 数据完整性缺失 / `start_motion_plan` 拒绝信息 / `INPUT_CONTRACT_BLOCKED` |
| IK_FAILURE | 记录中 `q_ok=false`、endpoint IK 失败或 `ENDPOINT_UNRESOLVED` 类字段 |
| SEARCH_TIMEOUT | `execution_status=TIMEOUT` 或 raw_status=`SEARCH_TIMEOUT`/`UNRESOLVED_BUDGET` |
| NO_CANDIDATE_WITHIN_BUDGET | `plan_result=NO_CANDIDATE_WITHIN_BUDGET` 且非超时 |
| CANDIDATE_COLLISION | 验证/参考记录中 `REJECT`、`invalid_states>0`、`REFERENCE_FAIL` |
| REFERENCE_VALIDATION_FAILED | `validate_motion` FAIL 或 `passed=false` 且非预算原因 |
| REFERENCE_INCOMPLETE | INCOMPLETE / `UNVERIFIED_SUCCESS` / `CANDIDATE_VALIDATION_TIMEOUT` |
| TOOL_ERROR | `HARNESS_ERROR` / `PROCESS_FAILURE` / 验证器异常 |
| MODEL_VERSION_MISMATCH | 模型或输入哈希不一致、历史证据与新路径不绑定 |
| EVIDENCE_INSUFFICIENT | 以上类别证据都不充分 |

**第 6 步 结构化诊断报告**（固定 8 节）：
1. 任务身份（Task ID / Run ID / 来源 LOCAL_RUN 或 HISTORICAL_RESEARCH_AUDIT）
2. 当前状态（三个维度分列）
3. 已确认事实（逐条附证据：工具名 + 字段值 + evidence_reference）
4. 证据文件/记录清单
5. 具体失败阶段（输入/IK/搜索/候选终验/独立验证）
6. 尚未确认的问题（UNRESOLVED 明示）
7. 推荐下一步（引用具体工具与参数，不超过三项）
8. 能否宣称"通过验证"：只允许三种回答之一——能（附哈希与范围）/不能/证据不足

**第 7 步 终止条件**：报告输出后停止。不自动重试规划，不自动扩大预算，不启动新规划任务（除非用户明确要求并确认预算）。

## 2. 禁止事项

- 禁止把 `plan_result=PATH_FOUND` 说成"验证通过"。
- 禁止把 TIMEOUT 解释为"路径不存在/任务不可行"。
- 禁止引用 Mock 数据（`mock=true` / `MOCK_PROTOCOL_TEST_ONLY`）作为任何实验证据。
- 禁止把某次运行的临时参数（seed、预算、超时值）沉淀为普遍规则。
- 禁止建议连接真实机器人或发送执行命令。
