# 上游 umaai-rs 有什么算法

> 上游仓库：[xulai1001/umaai-rs](https://github.com/xulai1001/umaai-rs)
> 一句话：这是赛马娘育成 AI 的 Rust 主仓，模拟器 + MCTS 搜索 + NN 评估 + 多教师训练一整套。模拟器是黑盒考场，我们不能改它，只能在外面换算法。

## 仓库结构

```
crates/
  umasim/        模拟器 + 搜索 + 评估 + 训练（核心就在这里）
  umaai/         上层的决策/协议/拉面杯逻辑（protocol、scenario、ramen_nn）
  umaai_review/  复盘评审工具
scripts/ramen_nn/   Python 侧的 NN 训练脚本（model/train/export_onnx）
```

关键依赖：`tract-onnx`（ONNX 推理，跑 NN 用）、`rayon`（并行放大批量评估）、`rand`/`rand_distr`（随机种子）。

---

## 一、搜索层：crates/umasim/src/search/

| 文件 | 是什么 |
|---|---|
| `flat_search.rs` | **扁平搜索**——不上完整树，只做一层前瞻，然后直接"快进"到终局用叶子评估器打分。这就是我们 GBDT 要接的那个"空叶子"搜索骨架 |
| `search/config.rs` | 搜索参数（探索常数、rollout 次数、种子等） |
| `search/terminal.rs` / `ramen_terminal.rs` | 终局判定（育成结束条件） |
| `search/searchable.rs` / `seeds.rs` | 搜索接口抽象 + 种子派生 |

**核心结论**：上游已经铺好了"搜索框架 + 叶子评估空着"的骨架，缺的就是一个"看状态报分"的叶子评估器。这也是为什么 GBDT 叶子评估器是当前性价比最高的下一步——不是从零造搜索，是往空槽里塞一块。

## 二、评估层（叶子估值）

| 类型 | 位置 | 说明 |
|---|---|---|
| 手写策略 | `game/ramen/policy.rs`、`policy_schema.rs` | 35+70 参数的规则打分，GA 拧过的那套 |
| NN 评估器 | `neural/neural_net_evaluator.rs`、`value_output.rs` | 用 tract-onnx 推理，输出胜率/价值 |
| 集成 NN | `scripts/ramen_nn/export_ensemble_onnx.py` | 支持把多个 NN 集成（ensemble）导出成一个 ONNX |
| rollout 终端 | `search/terminal.rs` | 叶子处用终局结果当评估（纯粹的 rollout 价值） |

NN 结构：`ramen_nn/whole.rs`（整体价值）+ `ramen_nn/hint.rs`（启发式/局部提示），配合 Python 侧 `model.py`。`DESIGN.md` 里有完整设计文档。

## 三、训练层：crates/umasim/src/trainer/

一大排训练器，说明上游走的是"多教师"路线：

| Trainer | 干什么 |
|---|---|
| `mcts_trainer.rs` / `ramen_mcts_trainer.rs` | MCTS 打谱，产出 (状态, 策略, 价值) 训练样本 |
| `neural_net_trainer.rs` / `ramen_nn_trainer.rs` | 用样本训练 NN 评估器 |
| `handwritten_trainer.rs` / `ramen_handwritten_trainer.rs` | 用样本反推/校准手写策略 |
| `collector_trainer.rs` / `mean_filter_collector_trainer.rs` | 数据采集 + 均值过滤（去噪） |
| `composition_ramen_trainer.rs` | 组合多个信号（NN + 手写 + MCTS） |
| `ramen_rollout_trainer.rs` | 纯 rollout 采样 |
| `ramen_special_root.rs` | 根节点特殊处理 |

**核心结论**：这就是"蒸馏闭环"的雏形——MCTS 打谱 → 训练学生（NN/手写）→ 更快的搜索 → 再打谱。我们后续的 Expert Iteration 正式化（tier3 #11），就是把它做成自动循环。

## 四、数据采集工具：tools/data_collection/

- `calc_training_value_microbench.rs`：训练价值计算基准
- `mcts_rollout_switch_verify.rs`：MCTS ↔ rollout 切换验证
- `nn_rollout_probe.rs`：NN rollout 探测
- `ramen_teacher_collect.rs`：**教师采集（5.6 万局数据就是这里产的）**
- `trainer_overhead.rs`：训练开销统计

## 五、设计文档（.trae/documents/）

- `nn_mcts_legacy_comparison.md`：NN vs MCTS vs 传统 对照
- `nn_rollout_handoff.md`：NN 与 rollout 的交接
- `cpu_search_optimization_plan.md`：CPU 搜索优化计划
- `good_bad_labels_draft.md`：好/坏标签草稿（用于二分类标签生成）
- `handwritten_base_policy_plan.md`：手写基策计划

---

## 一句话总结

上游已经拥有：**MCTS 搜索框架 + flat_search 空槽 + NN（tract-onnx，可集成）+ 手写策略 + 多教师训练器 + 5.6 万局数据采集**。

它缺的、也正是我们算法实验室要补的，是**决策面的"估值"和"前瞻"没接满**——flat_search 的叶子评估器还空着，这就是 GBDT / NNUE 增量评估 / Expert Iteration 的切入点。