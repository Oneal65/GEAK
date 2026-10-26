# kernel_workflow 深度解析：它如何使能 kernel 优化

> 依据代码：`kernel_workflow/kernel_workflow.js`（540 行，入口/分发器）、`kernel_workflow/kernel_lane.js`
> （1935 行，单语言 worker）、`kernel_workflow/roles/*.md`（12 个角色说明书）、`kernel_workflow/knowledge/*.md`、
> `kernel_workflow/scripts/*`。分析基于仓库 commit `cfd5b589`。

---

## 0. 一句话概括

kernel_workflow 是一个 **"JS 确定性编排 + LLM 角色只做判断"** 的多智能体流水线：

- **控制流**（预算循环、并行扇出、验证门槛、提交/回滚、何时停止）全部写死在 `kernel_lane.js` 里，
  不交给 LLM 去"理解 prose 再执行"；
- **每一个判断**（分析 kernel、规划方向、写代码、测性能、合并补丁、写报告）都由一个读了对应
  `roles/<role>.md` 的子 agent 完成，并且被**强制**以 JSON Schema（StructuredOutput）返回；
- **任何加速比都不相信自报**：每个工程师的补丁会被一个独立的 `verify_engineer` 在干净副本里重测，
  最终结果再由 `director` 从**真正的原始 kernel** 重新构建、重新测一遍。

所以它"使能 kernel 优化"的本质是：**把一个资深 kernel 工程师团队的工作方法（测量契约 → profile →
按瓶颈分派专项 → 独立复核 → 合并 → 再 profile → 迭代）固化成代码，把 LLM 的创造力约束在可验证的
轨道里。**

---

## 1. 分层结构

```
用户自然语言  ──►  Claude Code 主会话  ──►  Workflow 工具
                                              │
                         kernel_workflow.js (dispatcher, level 0)
                           │  mode=optimize | author  → 直接透传
                           │  mode=bakeoff            → Freeze/Discover/多语言并行/Report
                           ▼
                         kernel_lane.js (worker, level 1)   ← 真正的优化流水线
                           │ agent()/parallel()/pipeline()
                           ▼
          子 agent：director / tech_lead / benchmark_engineer / profile_engineer /
                    engineer(algorithm|memory|compute|host_runtime) / deep_engineer /
                    verify_engineer / integrator / researcher / update_experience ...
                           │ Bash/Read/Write（所有文件系统和 GPU 操作都由 agent 做）
                           ▼
          scripts/：gpu_lock.sh、materialize_workspace.sh、profile_kernel.sh、
                    workspace_sources.py、experience_store.py、kb.py ...
```

关键约束（写在代码注释里）：**workflow 脚本本身没有文件系统访问、没有时钟**（`Date.now()`
不可用，否则 resume 会失效）。所以：
- 所有路径都通过 `args` 传入（`workflow_dir` 必须由调用方传，因为脚本读不到自己的路径）；
- 所有 FS/shell 工作都由 agent 执行，脚本只信任 agent 返回的**结构化 JSON**；
- 连"读当前时间"都要派一个 `effort: 'low'` 的小 agent 跑 `date +%s`（见 `secondsLeft()`）。

嵌套层级被刻意限定为 **1 层**：dispatcher(0) → kernel_lane(1)。`e2e_workflow` 和
`kernel_workflow_bmk.js` 直接调 `kernel_lane.js`，不经过 dispatcher，避免超过一层。

---

## 2. 三种模式（`args.mode`）

| mode | 用途 | 行为 |
|---|---|---|
| `optimize`（默认） | 优化已有 kernel | dispatcher 直接 `workflow(kernel_lane.js, {...args})` 透传 |
| `author` | 没有可改的源码（如库 GEMM），从零写一个实现再优化 | `author_engineer` 先写"最简单的正确实现"当种子，再进入同一个优化循环；**分母仍是冻结的线上 kernel**，不是种子 |
| `bakeoff` | 同一个算子，多语言（HIP/Triton/FlyDSL/CK）赛马 | Freeze 一个不可变 oracle → Discover（复用 e2e 的 `op_benchmarker` 探测已有实现 + 离线调 aiter/CK）→ 每种语言一条 `kernel_lane` 并行 → 在**同一个冻结 baseline** 上排名取最快 |

没有自然语言模式识别，模式由调用方显式选择（`whenToUse` 写明 "there is NO natural-language mode detection"）。

---

## 3. kernel_lane 主流水线（逐阶段）

```
Setup → [Author] → Analyze → Benchmark → Profile → [Research] → [WarmStart]
  → LOOP{ Plan → (Optimize ‖ Verify 流水线) → [Integrate] → Commit → Reprofile → UpdateMemory → Reclaim }
  → Report → Validate → [UpdateExperience / kb:cite / kb:write]
```

### 3.1 Setup —— director:setup（隔离环境）
`roles/director.md` PHASE=setup：
- 生成防碰撞 run id：`TS=$(date +%Y%m%d_%H%M%S)_$$_${RANDOM}`，因为 agent 的时钟可能是冻结的；
- `EVAL_DIR = <exp_root>/team_<kernel>_<TS>/<kernel>`；
- 用 `materialize_workspace.sh` 把原 kernel 拷成 `baseline/`（原样保留）和 `workspace/`（canonical
  当前最优），排除 `*.so/*.o/aiter/jit`，保留 symlink（Issue #429：曾有 agent 把数 GB 的 .so 拷进每个副本）；
- 在 `workspace/` 里 `git init` + commit "baseline"，**后续所有补丁都是对这个 HEAD 的 git diff**；
- `.gitignore` 排除 build 产物，保证 diff 只含源码；
- **用户原目录只读**（除非 `apply_to_original=true` 且最终 accepted）。

**冻结 baseline 硬门槛**：脚本检查 `setup.baseline_frozen === true || baseline_callable`，否则直接
abort。注释里点名了它要防的 bug：*"optimized-HIP vs naive-HIP = fake 15.7×"*——拿优化后的代码去跟
自己写的朴素版本比，得到虚假加速。

可选 `GEAK_FREEZE_HARNESS=1`：把 harness 文件（task_runner.py 等）复制到只读的 `measure_golden/`，
每次测量前用 `restore.sh` 恢复——因为 agent 是 root，`chmod -w` 挡不住，"测量前恢复"才是硬保证。

### 3.2 Analyze —— tech_lead:analyze
读 kernel 源码、wrapper、测试，产出 `analysis.json`、`codebase_context.md`、`roadmap.md`，返回：
`kernel_type`、`modifiable_files`、`bottleneck_guess`、`candidate_directions`，以及 perf_knowledge 指针
`kk_operator / kk_language / kk_refs`（把这个 kernel 映射到 `perf_knowledge/` 中的算子×后端卡片，
后续工程师只读相关卡片，不必遍历整个知识库）。

### 3.3 Benchmark —— benchmark_engineer（测量契约 COMMANDMENT）
这是整个系统可信度的地基（`roles/benchmark_engineer.md`）：
1. **优先复用已有基础设施**：`config.yaml` 里的 `compile/correctness/performance_command`、
   `scripts/task_runner.py`、`test_*.py`。knn 正是这种情况；
2. 没有才自己写 `test_harness.py`（`--correctness/--benchmark/--full-benchmark/--profile`，CUDA events 计时）；
3. **分母必须是原始实现**，不允许 LLM 写的朴素 PyTorch 参考当 baseline（实在没有才回退，并显式 FLAG）；
4. **源码绑定校验**：往一次性副本的 HIP 源里插入 `#error GEAK_CANDIDATE_SOURCE_PROBE`，要求编译失败。
   如果编译仍然成功，说明 harness 编译的根本不是工作区里的源文件（比如指向了别处的绝对路径），该 harness 无效；
5. 写 `EVAL_DIR/COMMANDMENT.md`：SETUP / CORRECTNESS / BENCHMARK / FULL_BENCHMARK / PROFILE / PARSE /
   METRIC / MODIFIABLE FILES，**所有 GPU 命令必须经 `gpu_lock.sh` 从 workspace 内调用**；
6. FULL_BENCHMARK 跑 **3 次**，要求各 case 差异 ≈5% 以内，写 `baseline_timing.json`。

之后**每一个 agent 都只按 COMMANDMENT 测量**，这保证了不同工程师、不同轮次的数字可比。

### 3.4 Profile —— profile_engineer:baseline
- 先识别本机卡（`rocminfo` → gfx、CU 数），按 gfx 选 `amd_instinct.md`（gfx94*/gfx95*）或
  `amd_ryzen.md`（gfx11*）——**卡是探测出来的，不假设 MI300X**；
- 跑 `scripts/profile_kernel.sh`，按优先级回退：`rocprof-compute → omniperf → rocprofv3 → rocprof → metrix`；
  失败时输出 `!!! PROFILER FAILED` 块并给出该覆盖哪个环境变量，**不允许静默降级**；
- 按 `profiling_guide.md` 决策树分类：compute / memory / latency（再细分 C1 依赖等待 vs C2 发射等待）/
  lds / balanced，以及 **overhead-bound**（不同规模的 case 延迟相近、或每次调用 dispatch 数 > 1）；
- 做"廉价体检"：roofline 效率 >100% 说明峰值标定错；`Grid/Workgroup < CU 数` 说明 GPU 都没填满。

### 3.5 Research（可选，`dra_enabled=true`）
`researcher` 角色：提取事实 + 生成排序后的研究问题 → **每个问题一个 agent 并行**用 WebSearch/WebFetch
调研 → 可选 blindspot 第二波 → 合成 `deep_search_brief.md`（2–4KB 排序方向）。TechLead 规划时
**必须先做自己的 profile 分析，再看 brief**，且 brief 最多占 `BUDGET_REMAINING − 1` 个方向，
至少留一个"自由探索"槽——这是为了修复 v3 的"锚定"失败（所有工程师都扑向同一个主题）。

### 3.6 WarmStart（经验库热启动，默认 on）
按 `(kernel, language, gfx)` 在经验库里检索 top-3 历史最佳补丁（远端 KB Store 服务优先，凭据缺失则
回退本地 `kb_artifacts/` / `kb_store_local`）。**历史记录的加速比只用于排序**；每个候选都要先
`experience_store.py remap` 把路径映射到当前工作区，再走**和普通候选完全相同的 verify_engineer 门槛**，
第一个通过的被 adopt 并 commit，`cumulative` 从这个 Nx 起算。最后用 `attest` 把本机的复现结论
（validated/failed/not_reproduced/inapplicable）写回账本。返回值里会把 `incremental_speedup`
单独拆出来，防止把知识库带来的收益算成本次运行自己的功劳。

### 3.7 优化主循环（核心）

```js
while (!skipLoop && dispatched < BUDGET && noImprove < MAX_NO_IMPROVE) { ... }
```

每一轮：

**(a) Plan —— tech_lead:plan_round**
输入：剩余预算、当前累计加速、最新 profile、当前最优 per-case 表、HISTORY（洞察黑板 + 假设账本）、
perf_knowledge 指针、可选 DRA brief、learned KB 预算。返回 `{stop, reasoning, directions[]}`，
每个 direction 有 `specialty ∈ {algorithm, memory, compute, host_runtime, deep_explore}`、
`focus_files`、`expected_speedup`、具体 `prompt`。

`tech_lead.md` 中硬编码的规划规则（摘要）：
- **默认用完预算**，"pace, don't dump"：每轮 2–3 个方向，留预算给"集成后新出现的瓶颈"；
- **只有同时满足三条硬门槛才允许 stop**：(a) 重复调用型 harness 的 launch floor 已用 wrapper 级
  graph capture 攻过；(b) 没有 compute-bound case 仍 ≳3× floor；(c) 上一轮最佳验证收益 <~3%；
- **正交性**：每个方向 specialty+策略不同、主 `focus_files` 不同 → 互不冲突且可合并（替代了单独的去重步骤）；
- 多个 case overhead-bound 时**必须**包含一个 `host_runtime` 方向；必须有方向专攻最差的 case；
- **floor-aware**：大小差很多的 case 延迟接近 → 在 launch floor 上；floor 不是"做完了"，要用
  HIP graph capture 去压；
- 模式触发：单线程扫大数组 → 第一轮必须有 warp-cooperative 的 algorithm 方向；
- `deep_explore`：在 specialist 平台期 / 需要推倒重写 / 冲 roofline 时使用，**独占一轮**，花费
  `deep_cost`（默认 2）预算，只给目标不给方法。

脚本层面的约束：`plan.directions.slice(0, remaining)` 硬截断到剩余预算；一旦含 `deep_explore`，
本轮只保留它；`dispatched += roundCost`。

**(b,c) Optimize ‖ Verify —— `pipeline()` 流水线**
```js
const results = await pipeline(directions,
  d    => agentT(engineer prompt ...),      // 阶段1：工程师优化
  prev => agentT(verify_engineer ...))      // 阶段2：该方向一完成就立即独立验证
```
- 每个工程师先用 `materialize_workspace.sh` 从 CANONICAL 复制出**私有工作区**
  `round_N/engineer_i/workspace`，完全隔离；
- 只加载自己 specialty 的知识（algorithm → hip/triton_optimization P0/P1 + geomean_levers；memory →
  P1/P2 + 硬件参考；compute → P3/P4 + 占用率/VGPR 表；host_runtime → wrapper_optimization +
  geomean_levers，可以改 Python wrapper 和 C++ binding）——**上下文聚焦，结果自然正交**；
- 遵守 `self_monitoring.md` 的守护信号（停滞检测、死路检测、天花板检测、崩溃循环恢复、多样性）；
- 只有 geomean > `candidate_floor`(1.0) 才写 `best_patch.diff`；
- **pipeline 的意义**：不必等最慢的工程师，谁先完成谁先验证。

**验证的"测量锚定"而非"返回值锚定"**：只有"工程师正常跑完、诚实报告 ≤ floor、measurement_valid=true"
这一种情况才跳过 verify。工程师崩溃、超时、StructuredOutput 丢失时，**仍把磁盘上的
`best_patch.diff` 交给 verify**（注释里记录了一次 bake-off 中 1.56× 的 Triton 补丁因返回值丢失被静默丢弃）。

**verify_engineer**（"trust anchor"）：在 CANONICAL 的干净副本上 `git apply` → 跑 COMMANDMENT 的
correctness → FULL_BENCHMARK **跑两次**（差 >5% 取更好/中位并记录方差）→ 拿 `BASELINE_PER_CASE`
算 per-case 加速、geomean、算术平均。修改 harness、回归（≤1.0）、apply 失败、正确性失败都会被拒。
**脚本只信任 verify 的数字**：
```js
verified = clean.filter(r => says(r.ver.status,'verified') && says(r.ver.correctness,'pass')
                             && primSpeedup(r.ver) > CANDIDATE_FLOOR)
```
（`says()` 匹配开头单词，避免 agent 回答 "PASS - 15/15 draws" 时被 `=== 'pass'` 误判。）

**(d) Integrate —— integrator（≥2 个验证通过时，不耗预算）**
按验证加速排序，先 apply 最好的，再逐个叠加，每加一个都 correctness + benchmark，只保留"正确且更快"
的叠加；`git apply` 冲突时**手工合并**两种思路（注释说：最好的结果常常是手工合并，而不是 diff 堆叠）。
只有 `conclusion==='improved'` 且超过最佳单个候选，才作为额外候选加入。

**(e) Commit 门槛**
```js
const improved = winner && winner.geomean > cumulative * (1 + MIN_IMPROVE)   // 默认 2%
```
超过累计最优 2% 才 commit 进 CANONICAL（`git apply || git apply --3way || 手工+重跑 correctness`），
并写 `current_best.diff`。**下一轮在累计最优之上继续**。注意加速比永远是 **相对真正原始 baseline 的
绝对延迟比**：`geomean(baseline_ms / optimized_ms)`，而不是相对上一轮的。

**(f) Reprofile** —— commit 后重新 profile，写 BEFORE→AFTER 的瓶颈迁移分析，供下一轮规划。

**UpdateMemory —— tech_lead:update_memory（跨轮记忆）**
把本轮每个方向的 expected/claimed/verified/status 喂回 TechLead，蒸馏成：
- `insights`（洞察黑板，注入下一轮工程师 prompt 的 `INSIGHTS`）
- `ledger`（假设账本：哪些方向是死路，不再重试）
- `bottleneck_now`、`suggest_next`

**停止条件**：预算用尽；或连续 `MAX_NO_IMPROVE`（默认 2）轮无进展；或 TechLead `stop=true`
（且通过了上面三条硬门槛）；或可选 `deadline_epoch` 到期（且在窗口未用完时会**拒绝**
TechLead 的 stop，强制重规划，最多 `max_forced_replans` 次）。

**Reclaim** —— 每轮结束跑 `reclaim_eval_artifacts.sh` 回收旧轮的重型副本（软上限 32GiB、硬上限 64GiB）。

### 3.8 Report —— tech_lead:report
写 `tech_lead_report.md`（逐轮叙述 + 最终 per-case 表）和 `final_patch.diff`（canonical 相对初始 commit 的累计 diff）。

### 3.9 Validate —— director:validate（独立仲裁）
- **不信任 TechLead 的数字**：从**用户原始路径**重新 materialize 一个 `validation_workspace`，
  `git apply final_patch.diff`，按 COMMANDMENT 跑 correctness 和 FULL_BENCHMARK（gpu_lock）；
- 用 `BASELINE_TIMING` 重算 geomean；
- 仲裁：Director 结果在 TechLead 声称值 10% 以内或更高 → `accepted`；低 >10% → `flagged`
  （以 Director 实测为官方值）；正确性失败或补丁无法应用 → `flagged`；
- 标注 `timing_basis`（clean / host_bound / unprimed / unknown / not_applicable）——普通 optimize
  模式没有冻结 oracle 的 timing receipt，标为 `not_applicable`，这是路径的形态而不是故障；
- `apply_to_original=true` 且 accepted 时才把补丁写回原目录。

### 3.10 经验沉淀（自我进化）
- **UpdateExperience**：仅当 `validation_status==='accepted'` 且最终加速 >1.0 且机器不拥挤、非 held-out
  时，TechLead 往 `knowledge/learned/` **最多蒸馏一张卡片**（ADD-only、curate/merge 而非盲目追加；
  只记录相对数字，不记 ms/TFLOPs；记录 `symptom → root cause → fix` 的坑）。`INDEX.md` 由
  `kb.py index` 从卡片 frontmatter 生成。目前库里已有约 139 个文件；
- **kb:cite**：把"本轮规划引用了哪张卡 → verify 实测结果 → 是否成为赢家"写入引用账本——
  **对所有运行都写**（包括失败的），否则只把成功样本放进反馈数据，计数会显得"虚高"；
- **kb:write**：把最终补丁写回经验库（本地树 + 本地 store + 有凭据时远端服务），供下一次 WarmStart 使用。

为了防止知识库"把规划变成笼子"，代码中有一段实测数据：给规划者读完整索引自由挑选，16 个 kernel 的
geomean 从 4.45× 降到 3.44×（-22.7%）。因此加了 `KB_DIR_CAP=1`（每轮最多 1 个方向由卡片引导）和
`KB_COLD_DIRECTION`（每轮至少 1 个纯 profile 驱动的方向作为对照组），并统计 `kb_cap_bound_rounds`、
`direction_entropy`（单一化的"金丝雀"指标）。

---

## 4. 让优化"可信"的工程机制汇总

| 机制 | 位置 | 防的是什么 |
|---|---|---|
| 冻结 baseline 硬门槛 | `kernel_lane.js` Setup 后 | 拿优化版跟自己写的朴素版比（假加速） |
| COMMANDMENT 测量契约 | benchmark_engineer | 不同 agent 用不同方式测，数字不可比 |
| 源码绑定 probe（`#error` 注入） | benchmark_engineer + `workspace_sources.py` | harness 实际编译的是别处的源码 |
| `gpu_lock.sh` | 所有 GPU 命令 | 并行工程师抢 GPU；共享 torch_extensions 缓存导致测到别人的 .so；为 9 种 arch 编译 |
| gpu_lock 退出时源码复检（exit 86） | `gpu_lock.sh` | 命令执行过程中被 builder 重建的陈旧 overlay |
| 私有工作区 + git diff 补丁 | engineer/verify/integrator | 工程师相互干扰；补丁不可复现 |
| 独立 verify（两次测量） | verify_engineer | 工程师自报加速有噪声/有误 |
| 测量锚定的 harvest | `pipeline` 第二阶段 | 返回值丢失导致真实赢家被丢弃 |
| 2% commit 门槛 | `MIN_IMPROVE` | 把噪声当作进步提交 |
| Director 从原始路径重建验证 | director:validate | TechLead 汇报失真；harness 被改 |
| `agentT()` 超时 + 重试 | 所有 agent 调用 | 单个 agent 卡死阻塞整轮 barrier；网关瞬时故障导致整个 workflow 崩溃 |
| StructuredOutput + JSON Schema | 所有角色 | LLM 输出格式漂移，脚本无法解析 |
| 可选 `GEAK_FREEZE_HARNESS` | director/benchmark | root agent 改 harness 刷分 |

---

## 5. 让优化"有效"的方法论（knowledge/）

- `optimization_strategies.md`：优先级层级 + 关键模式检测 + 按瓶颈选策略（memory/compute/latency/LDS/
  balanced/overhead-bound）+ 可叠加/互斥的组合策略表（integrator 依赖它判断兼容性）；
- `geomean_levers.md`（**每轮必读**）：核心观点 **"kernel-fast ≠ wall-clock-fast"**——kernel 本身提速
  40–60× 后，小 shape 的墙钟被固定开销（Python 分发、`.contiguous()`、`torch.zeros`、每次 launch
  5–15µs × dispatch 数）托底。六个杠杆：合并 dispatch、消除 host 开销、原生布局输出、
  攻最慢的 case、persistent/grid-right kernel、HIP graph capture；以及"floor-dominated 信号陷阱"；
- `hip_optimization.md` / `triton_optimization.md`：P0–P4 分级技巧；
- `wrapper_optimization.md`：host/runtime 模式；
- `amd_instinct.md` / `amd_ryzen.md`：按 gfx 的硬件参数（CU、VGPR/AGPR、占用率、带宽）；
- `profiling_guide.md`：profiler 用法、故障阶梯、瓶颈决策树；
- `self_monitoring.md`：工程师自我监控（停滞/死路/天花板/崩溃循环/多样性）、补丁保存规则、benchmark 纪律；
- `../perf_knowledge/`：算子×后端 SOTA 卡片（REFERENCE ONLY：只提供事实/做法，绝不替代测量）；
- `knowledge/learned/`：历史运行蒸馏出的经验卡片（advisory prior，永远被实测推翻）。

**为什么要"专项工程师"**：每个 specialty 只加载自己的知识 → 上下文更聚焦；各自改不同文件 →
补丁天然正交、可被 integrator 合并；`host_runtime` 被提升为一等公民，因为注释指出"最后 1.5–3× 的
geomean 常常在这里"。

---

## 6. 预算语义

`budget` = 整个运行中 TechLead 可以派出的**优化方向总数**。只有优化方向的工程师计入预算；
benchmark / profile / verify / integrate / commit / validate 都**不**计入。`deep_explore` 每个计
`deep_cost`（默认 2）。例：budget=6 → 第 1 轮 3 个 + 第 2 轮 3 个；或 4+2；或 4 个后 stop。

GPU 调度默认 `gpu_mode=pool`：每个消费者拿到整个 GPU 池（如 `"0,1,2,3"`），由 `gpu_lock.sh`
在获取时 flock 一张**空闲**的卡。注释给出的理由：任务耗时差异可达 23×，固定绑卡会让一轮被最慢的
"卡"而不是最慢的"任务"拖住，4 路时会闲置约 43% 的 GPU 时间。

---

## 7. 产物目录（每次运行）

```
exp/team_<kernel>_<TS>/<kernel>/
├── baseline/                 原始拷贝（只读参考）
├── workspace/                canonical 当前最优（git 仓库，每轮赢家一个 commit）
├── COMMANDMENT.md            测量契约
├── baseline_timing.json      3 次 baseline
├── analysis.json / codebase_context.md / roadmap.md
├── baseline_metrics.json / profiling_summary.md / profile_output/
├── round_N/engineer_i/{workspace/, worker_result.json, report.md, best_patch.diff, verify/}
├── round_N/integrate/
├── round_N_metrics.json / round_N_shift_analysis.md
├── insight_log.md / current_best.diff
├── tech_lead_report.md       TechLead 的逐轮叙述 + 最终表
├── final_patch.diff          官方补丁
├── validation_workspace/     Director 从原始路径重建
└── director_validation.json  官方验证结果
```

---

## 8. 一个观察：本机 pip 版 ROCm 的适配点

workflow 本身不安装工具链，"kernel 环境必须先建好"（README 警告）。本机是 pip 安装的
`rocm_sdk`（ROCm 10.0），需要 `~/.geak_env.sh` 设置 `ROCM_HOME` 和 `libamd_smi.so` 软链。子 agent 的
Bash 是非交互 shell，未必会 source `~/.bashrc`，所以本次运行通过 `args.task` 要求 benchmark_engineer
把 `source ~/.geak_env.sh &&` 写进 COMMANDMENT 的每条命令里——COMMANDMENT 是所有 agent 唯一的
测量入口，所以只要它带上前缀，整个团队就都在正确的环境里运行。详见 `01_setup_and_run.md`。

---

## 9. 用本次 knn 运行印证上述机制

| 机制 | 本次实际表现 |
|---|---|
| 独立 verify | 工程师自报 r2_d1 34.40×，verify 复测 33.82×；所有晋级数字都来自 verify |
| Integrator 手工合并 | round 1 两补丁原样叠加 17.07×，手工合并成单次 dispatch 31.33×——印证了 "algorithm 与 host 收益只有共用一次 dispatch 时才相乘" |
| Reprofile 驱动下一轮 | r1 后瓶颈从"串行 + under-fill"变为"VALU-issue-bound + host 地板"，r2 的方向正是针对这两点 |
| 停止门槛 | r2 规划时 TechLead 逐条核对 (a)(b)(c) 三条门槛，均不满足，于是拒绝 stop 并用完预算 |
| geomean_levers 的 graph-capture 杠杆 | 被规划、被实测、被否定（单 dispatch 算子 replay ≈ 裸 launch，15/15 更慢），作为 dead-end 写进 ledger 和报告——知识是先验，实测说了算 |
| Director 仲裁 | 从原始目录重建复测 53.97× vs 报告 54.09×（差 0.2% < 10%）→ accepted |
| 经验沉淀 | 卡片进 `_inbox/` 等人工 drain；补丁写入本地经验库，下次同名 kernel 会在 WarmStart 中作为候选并重新验证 |
| learned KB 预算 | 两轮各有 1 个 host_runtime 方向引用了历史卡片（均来自 gfx950 attention decode），未超 `KB_DIR_CAP`，`kb_cap_bound_rounds=0` |

暴露的鲁棒性问题（详见 `01_setup_and_run.md` §4.2）：嵌套仓库导致 `git apply` 静默跳过、hipify 生成文件引起补丁漂移、`target_language` 默认 triton 导致 HIP 经验被错误标注、reclaim 的 `rm -rf` 在 auto 权限模式下被拦截。前两个都被下游 agent 发现并手工修复——这也说明"每一步都重跑正确性"的设计让系统对 agent 自身的失误有容错能力。
