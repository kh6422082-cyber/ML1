# VQOS 资源选择器：最小可用版本

## 它负责什么

输入一批已经可编译的 VQOS 候选，例如

\[
a=(\text{mask 数},\text{层数},R_{\rm trace},M_{N,W},\delta),
\]

资源选择器预测这些候选在固定活性空间、固定 \(T_{\max}\) 下的长期误差：

\[
\left\{r_{\rm McL},\kappa(N),\mathrm{coverage},\theta,L,n_H,\mathrm{CNOT},\ldots\right\}
\mapsto
\left\{\mathrm{RMS}[C(t)],\Delta E_s,\Delta w_s\right\}.
\]

它选择最便宜、且在不确定度上界内满足误差阈值的候选。它不接受精确 \(U(t)\)、精确 \(C(t)\) 或精确谱作为推理输入；这些只能在经典开发阶段产生训练标签。

## 当前项目的默认约束

| 项 | 默认阈值 |
|---|---:|
| RMS \(C(t)\) | \(\le 0.02\) |
| 强峰 \(\Delta E_s\) | \(\le5\times10^{-4}\) |
| 强峰 \(\Delta w_s\) | \(\le0.02\) |
| 编码泄漏 | \(\le10^{-12}\) |

默认成本是每个 \(\delta\) 的受控 CNOT 代理；可以加入 \(N,W\) 测量电路数或 trace shots 的权重。

## 决策逻辑

对候选 \(a\)，树集成给出 log-error 均值 \(\hat y\) 和 ensemble spread \(s\)。资源选择器使用：

\[
e^{\rm upper}_j(a)=10^{\hat y_j+z s_j},\qquad z=1\ \text{为当前默认值}.
\]

只有同时满足以下条件才允许部署：

\[
e^{\rm upper}_j(a)\le \tau_j,\quad
\text{code-preserving}=\mathrm{True},\quad
\ell_{\rm leak}\le10^{-12}.
\]

若没有候选通过，结果必须是 `abstain`。此时应增加 masks、层数、trace/直接测量 shots，或先补充校准数据；不能把风险最低的 fallback 说成通过。

## 新同学应完成的四个接口

1. `dataset_generation.py`：扫描分子 Hamiltonian、GBS closure、masks/layers，生成离线标签；
2. `feature_extraction.py`：只保存部署时能取得的 \(N,W\) 残差、条件数、coverage、资源和 pilot 统计；
3. `selector.py`：调用本模块训练、预测、选择和拒绝；
4. `evaluation.py`：按 **molecule_id** 切分训练/留出，而不能把同一分子的候选随机拆分。

## 最小调用示例

```python
from vqos_resource_selector import VQOSResourceSelector, split_by_molecule

# records: list[LabeledCandidate]，标签只在经典开发阶段存在。
train, holdout = split_by_molecule(records, {"lawsone-target"})
selector = VQOSResourceSelector(feature_names, safety_sigma=1.0)
selector.fit(train)

# target_candidates: list[CandidateInput]，没有精确标签。
decision = selector.select(target_candidates)
if decision.status == "certified":
    use_candidate = decision.selected.candidate
else:
    # 升级 ansatz / 增加 trace 或直接测量 shots，而非强行运行。
    print(decision.message)
```

## 接入 M6.2 的映射

`TraceCandidate.features` 可直接写入 `CandidateInput.features`；其
`cnot_per_delta`、`n_measurement_circuits`、`trace_samples` 进入
`ResourceCost`。M6.2 的 `rms`、`d_energy`、`d_intensity` 只在生成训练集时写入 `OfflineLabels`。

当前 `m6_2_trace_probe_wqte_ml.py` 已通过 `selector_input()` 和
`selector_label()` 接入本模块：`target` 分支只调用前者，因而不会把 ED
相关量带入目标分子的选择输入。

当前 R=32 trace-probe 的 **+1\(\sigma\) 原型筛选** 选择 32 masks/2 layers，
与基线一致。这不是 selector 失败，而是保守的无压缩结论：随机 trace 估计下，
更浅候选没有被可靠筛出。上硬件前须用独立分子、多个 trace/shot seed 校准这个
上界；完成校准后才将 `certified` 解释为统计意义上的认证。
