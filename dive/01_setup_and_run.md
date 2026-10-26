# GEAK 在本机（8× MI308X）上的修复与 kernel_workflow 运行记录

> 日期：2026-10-26　仓库：`/root/serving_agi/GEAK`（commit `cfd5b589`）　harness：Claude Code（`~/.local/bin/tclaude`，内置 Workflow 工具）

## 1. 本机环境

| 项 | 值 |
|---|---|
| GPU | 8 × AMD Instinct MI308X（gfx942 / CDNA3），192 GB HBM，运行前全部空闲 |
| ROCm | pip 版 `rocm_sdk`（ROCm 10.0），位于 `/opt/python/lib/python3.14/site-packages/_rocm_sdk_{core,devel}`，没有 `/opt/rocm` |
| Python | `/opt/python/bin/python3`（3.14） |
| 框架 | torch 2.12.0+rocm10.0.0、triton 3.8、aiter、flydsl、vllm |
| 工具 | `hipcc`、`rocprofv3` 可用；`metrix` 未安装（只是 profiler 优先级里的最后一个回退，不影响使用） |
| API | 已配置 `ANTHROPIC_BASE_URL`、`ANTHROPIC_AUTH_TOKEN`、`ANTHROPIC_CUSTOM_HEADERS` |

## 2. 发现的问题（与 GEAK 代码无关，是 pip 版 ROCm 的问题）

1. **`import torch` 直接报错 `KeyError: 'libamd_smi.so'`**
   amdsmi 只按文件名 `libamd_smi.so` 查找，而磁盘上只有带版本号的 `_rocm_sdk_core/lib/libamd_smi.so.27`。
2. **HIP kernel 编译失败：`hip/hip_runtime_api.h: No such file or directory`**
   没有设置 `ROCM_HOME`，torch 的 `cpp_extension` 找不到 HIP 头文件
   （头文件实际在 `_rocm_sdk_devel/include/hip/`）。

## 3. 修复（已执行，持久化）

```bash
SP=/opt/python/lib/python3.14/site-packages
mkdir -p ~/.local/lib/smifix
ln -sf $SP/_rocm_sdk_core/lib/libamd_smi.so.27 ~/.local/lib/smifix/libamd_smi.so
```

新建 `~/.geak_env.sh`（幂等，重复 source 不会叠加 PATH）：

```bash
# GEAK on pip-installed ROCm (rocm_sdk) — fixes amdsmi lookup + HIP headers for torch cpp_extension
_SP=/opt/python/lib/python3.14/site-packages
export ROCM_HOME=$_SP/_rocm_sdk_devel
export ROCM_PATH=$ROCM_HOME
case ":$LD_LIBRARY_PATH:" in *":$HOME/.local/lib/smifix:"*) ;; *)
  export LD_LIBRARY_PATH=$HOME/.local/lib/smifix:$ROCM_HOME/lib:$_SP/_rocm_sdk_core/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH};;
esac
case ":$PATH:" in *":$HOME/.local/bin:"*) ;; *) export PATH=$HOME/.local/bin:$PATH;; esac
unset _SP
```

在 `~/.bashrc` 末尾追加了一行：`[ -f ~/.geak_env.sh ] && . ~/.geak_env.sh`

### 验证
```text
$ . ~/.geak_env.sh; python3 -c "import torch; ..."
2.12.0+rocm10.0.0 True 8 AMD Instinct MI308X

$ cd examples/tasks/knn
$ python3 scripts/task_runner.py compile      → Compilation: PASS
$ python3 scripts/task_runner.py correctness  → Correctness: PASS
$ python3 scripts/task_runner.py performance  → Perf: 0.3170 ms (shape_4_standard)
                                                 Perf: 0.3268 ms (shape_4_transposed)
                                                 Perf: 0.3447 ms (shape_4_self_query)
```

> 注：knn 的编译过程会用 hipify 原地改写 `src/knn_hip.cpp` 和 `src/knn_hip.hip`，验证后已
> `git checkout` 恢复，保证仓库工作区干净。workflow 本身只在 `exp/` 下的副本里工作，不会动原目录。

## 4. 运行 kernel_workflow

### 4.1 调用方式
在 Claude Code 会话中直接调用 Workflow 工具（等价于对 Claude 说
`use /root/serving_agi/GEAK/kernel_workflow to optimize /root/serving_agi/GEAK/examples/tasks/knn, budget 4`）：

```js
Workflow({
  scriptPath: "/root/serving_agi/GEAK/kernel_workflow/kernel_workflow.js",
  args: {
    kernel_path:  "/root/serving_agi/GEAK/examples/tasks/knn",
    workflow_dir: "/root/serving_agi/GEAK/kernel_workflow",
    mode: "optimize",
    budget: 4,                 // 默认 6；本次为缩短时长设为 4
    gpu_ids: "0,1,2,3",        // pool 模式，gpu_lock.sh 动态分配空闲卡
    task: "ENV NOTE (mandatory): ... 每条 COMMANDMENT 命令必须带 `source ~/.geak_env.sh &&` 前缀 ..."
  }
})
```

**为什么要用 `task` 注入环境说明**：子 agent 的 Bash 是非交互 shell，不一定会加载 `~/.bashrc`。
COMMANDMENT 是所有 agent 唯一的测量入口，只要 benchmark_engineer 把 `source ~/.geak_env.sh &&`
写进 COMMANDMENT，整个团队就都会在正确的环境里编译和测量。

Run ID：`wf_7f5c1d46-f90`
输出目录：`/root/serving_agi/GEAK/exp/team_knn_20261026_120638_2238851_5312/knn/`

### 4.2 过程

总耗时约 **2 小时 21 分**（12:06 → 14:27），共 **30 个子 agent**，0 个出错，418 次工具调用，约 168 万 subagent token。

| # | 阶段 / agent | 关键产出 |
|---|---|---|
| 1 | `director:setup` | 建立 `EVAL_DIR`，`baseline/` + `workspace/`（git commit "baseline"）。识别为 hip2hip 任务，唯一需要优化的源是 `src/knn_cuda.hip`；冻结 baseline = 原始拷贝 |
| 2 | `tech_lead:analyze` | `kernel_type=hip`，可改文件：`src/knn_cuda.hip, src/knn.cpp, knn_wrapper.py, kernel_loader.py`；初判 latency-bound |
| 3 | `benchmark_engineer` | 复用 `task_runner.py` 生成 COMMANDMENT（已带 ROCm 环境变量，全部经 `gpu_lock.sh`），baseline 跑 3 次；15 个 case（5 shape × standard/transposed/self_query），baseline geomean **0.4087 ms** |
| 4 | `profile_engineer:baseline` | rocprofv3：**latency-bound**。每线程串行扫描 N、816 B/lane scratch、只有 1–16 个 WG（80 CU）、每次调用 5–7 次 dispatch、78% 时间在依赖等待；`knn_kernel` 占 GPU 时间 97% |
| 5 | `warm_start:resolve` | `no_such_store`，本机无经验库，冷启动 |
| 6 | `tech_lead:plan r1` | 2 个方向：algorithm（wave 协作 + 寄存器 top-k）、host_runtime（单次 dispatch 的 C++ 绑定） |
| 7 | `eng r1_d0` → `verify` | **7.72×**（verify 独立复测） |
| 8 | `eng r1_d1` → `verify` | **1.055×**（kernel 主导时 host 收益被掩盖） |
| 9 | `integrate r1` | 原样叠加 17.07×；**手工合并成单次 dispatch 31.33×** → 提交 |
| 10 | `reprofile r1` | dispatch 1 次；分化：self_query 大 case 变 VALU-issue-bound（每对点 40–50 条 VALU），小 case 落在 5–9 µs 地板 |
| 11 | `tech_lead:plan r2` | 剩余预算 2，明确拒绝 stop（三条停止门槛都不满足），再派 algorithm + host_runtime |
| 12 | `eng r2_d0` → `verify` | **43.21×**：radix-select 阈值预筛 + 零 VALU 的 buffer 寻址 + packed fp32，VALU/对从 40–50 降到 13–14 |
| 13 | `eng r2_d1` → `verify` | **33.82×**：CPython METH_FASTCALL 快路径，host 7.2–8.1 µs → 5.1–5.5 µs；HIP graph replay 实测 15/15 case 都更慢，已放弃 |
| 14 | `integrate r2` | 两补丁改的是不相交文件，干净叠加 **54.09×** → 提交（需手工应用，见下方问题 3） |
| 15 | `reprofile r2` / `memory r2` | 剩余三种瓶颈形态：host 地板（~4.7 µs）、under-fill、VALU+vL1 共同受限 |
| 16 | `tech_lead:report` | `tech_lead_report.md`、`final_patch.diff` |
| 17 | `director:validate` | 从原始目录重建 → 打补丁 → 正确性 PASS → 重测：**53.97×**，与 TechLead 的 54.09× 差 0.2% → **accepted** |
| 18 | `update_experience` / `kb:cite` / `kb:write` | 经验卡片放进 `knowledge/learned/_inbox/`（需人工 `drain --apply` 才正式入库）；引用账本 2 条；补丁写入 `kb_artifacts/` + `kb_store_local/`（远端无凭据，跳过） |

停止原因：`stopped_by: "budget"`（4/4 用完，2 轮）。

**运行中发现的问题**
1. **每轮的磁盘回收被权限系统拦截**：`reclaim_eval_artifacts.sh` 内含 `rm -rf`，被 auto 模式的分类器判为不可逆删除。本次 `exp/` 只有 180 MB，没有影响。若长期运行想自动回收，可给该脚本加一条 Bash 允许规则。
2. **合并时 `git apply` 静默跳过补丁**（round 1）：`exp/` 位于外层 GEAK 仓库内且被 gitignore，`git apply` 在嵌套位置会悄悄跳过文件而不报错。integrator 发现后手工合并，所以没造成损失，但这是一个潜在的 GEAK bug（可考虑把 `exp_root` 放到仓库外，或在 workspace 内用 `git -C` / `--directory`）。
3. **round 2 提交需手工应用**：`src/knn_hip.hip` 是 `knn_cuda.hip` 的 hipify 产物，编译时被重新生成，导致 integrate 工作区和 canonical 中该文件内容不一致，`git apply` / `--3way` 都失败。commit agent 手工应用并重跑正确性后通过。
4. **经验库语言标签错误**：optimize 模式下 `target_language` 默认是 `triton`，所以这次 HIP kernel 的经验被记成 `knn__triton__gfx942`（路径在 `kb_artifacts/gfx942/hip/` 下，但 slug 和 canonical id 是 triton）。下次如果要命中热启动，需要传同样的 `target_language`；更好的做法是调用时显式传 `target_language: "hip"`。

### 4.3 结果

**Director 独立验证：geomean 53.97×，算术平均 76.38×，正确性 PASS，状态 accepted**
（TechLead 报告 54.09×，差 0.2%；`timing_basis: not_applicable` 表示 optimize 模式下没有冻结 oracle 的 timing receipt，分母来自本 lane 自己的 `baseline_timing.json`）。

| case | baseline ms | 优化后 ms | 加速 |
|---|---|---|---|
| shape_0_standard | 0.0542 | 0.0044 | 12.3× |
| shape_0_transposed | 0.0635 | 0.0045 | 14.1× |
| shape_0_self_query | 0.0664 | 0.0048 | 13.8× |
| shape_1_standard | 0.2935 | 0.0049 | 59.9× |
| shape_1_transposed | 0.3055 | 0.0049 | 62.3× |
| shape_1_self_query | 0.3712 | 0.0050 | 74.2× |
| shape_2_standard | 0.7547 | 0.0089 | 84.8× |
| shape_2_transposed | 0.7633 | 0.0097 | 78.7× |
| shape_2_self_query | 0.8821 | 0.0247 | 35.7× |
| shape_3_standard | 2.1738 | 0.0086 | 252.8× |
| shape_3_transposed | 2.1655 | 0.0099 | 218.7× |
| shape_3_self_query | 2.2947 | 0.0261 | 87.9× |
| shape_4_standard | 0.3169 | 0.0062 | 51.1× |
| shape_4_transposed | 0.3266 | 0.0073 | 44.7× |
| shape_4_self_query | 0.3428 | 0.0063 | 54.4× |

（全部为 Director 在重建工作区上的实测值，每个 case 跑 3 次取中位，来自 `director_validation.json`。）

**加速轨迹**：baseline 1× → round 1 合并 31.33× → round 2 合并 54.09× → Director 复测 53.97×。

**起作用的优化**
1. 每个 query 一个 wave，lane 协作，top-k 放寄存器：scratch 816 B/lane → 0，填满 GPU（单独 7.7×）
2. 单次 dispatch + 原生布局：直接读 (B,N,3)/(B,3,N)，直接写 int32 (B,k,M)，去掉 dist2 缓冲、fill、transpose、cast（与 1 合并后 31.3×）
3. radix-select 阈值预筛：fp32 非负距离的位模式与 uint32 同序，用 SGPR 整数比较，每 tile 插入数从 64 降到 3–11
4. 零 VALU buffer 寻址 + packed fp32：VALU/对 40–50 → 13–14，self_query 快 2×
5. 精简 host 路径（METH_FASTCALL builtin）：每次调用省约 2.5 µs

**没起作用的**：HIP graph capture/replay（单 dispatch 算子没有可摊销的 launch 链，15/15 更慢）；self_query 多 query/wave（VGPR 翻倍，占用率下降）；两补丁原样叠加（17× vs 手工合并 31×）。

**剩余空间（TechLead 建议）**：s2/s3/s4 的 standard/transposed 把 N 拆到 2–4 个 wave 再用 LDS/DPP 合并 top-k；self_query 用 LDS 共享点 tile；transposed 路径用 b64/b128 读取。s0/s1 已到 host 地板。

**产物位置**
- 官方补丁：`exp/team_knn_20261026_120638_2238851_5312/knn/final_patch.diff`
- 报告：同目录 `tech_lead_report.md`、`director_validation.json`
- 原目录 `examples/tasks/knn` **未被修改**（`apply_to_original` 默认 false）。要应用：`cd examples/tasks/knn && git apply <final_patch.diff>`
- 新增的仓库文件：`kernel_workflow/knowledge/learned/_inbox/` 下 2 个 json（经验卡片 + 引用账本）；`kb_artifacts/`、`kb_store_local/`（本地经验库）

## 5. 以后怎么用

```bash
source ~/.geak_env.sh            # 新 shell 会通过 ~/.bashrc 自动加载
IS_SANDBOX=1 tclaude --dangerously-skip-permissions
```
然后用自然语言下达任务：
- 单 kernel：`use /root/serving_agi/GEAK/kernel_workflow to optimize <kernel_dir>[, budget N][, focus on ...]`
- 多语言赛马：在上面的基础上加 `mode bakeoff, backends hip,triton,flydsl, gpus 0,1,2`
- 整机推理（本机装的是 vllm）：`use /root/serving_agi/GEAK/e2e_workflow to optimize inference for <model>, vllm, ISL/OSL=1024, conc=64, gpus 0,1,2,3`

可选：`cd /root/serving_agi/GEAK && GEAK_SKIP_BOOTSTRAP=1 pip install -e .`
（会从 GitHub 安装 `metrix`；`GEAK_SKIP_BOOTSTRAP=1` 表示不重复 clone 仓库，也不重复安装 Claude CLI）。

kernel_workflow 的原理分析见 [`02_kernel_workflow_deep_dive.md`](02_kernel_workflow_deep_dive.md)。
