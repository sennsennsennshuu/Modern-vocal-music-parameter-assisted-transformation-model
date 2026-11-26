# Modern Vocal Music Parameter Assisted Transformation Model 模型卡（Model Card）

本文件说明当前仓库交付的两个声学与构音观测模型的技术规格、实测指标、已知局限与许可边界。更多仓库使用指引见 [README.md](README.md)，上游数据来源与条款汇总见 [DATASET_LICENSES.md](DATASET_LICENSES.md)。

## 模型身份与发布内容

当前仓库是一个纯模型文件仓库，根目录仅包含 [final_pitch/](final_pitch/)、[final_analyzer/](final_analyzer/)、[README.md](README.md)、`MODEL_CARD.md`、[DATASET_LICENSES.md](DATASET_LICENSES.md)、[LICENSE](LICENSE)（Apache-2.0）、`.gitattributes`、[licenses/sources.json](licenses/sources.json) 以及 `licenses/texts/` 下的 8 份原始条款文件。仓库不包含训练代码、`analyzer` Python 包或分块命令行工具，也不发布任何原始训练数据。

两个交付目录各包含 `weights.pt`、ONNX 模型文件（`.onnx`）与 `metadata.json`：

- `final_pitch/`：专用音高与周期/清浊代理估算模型，定位为独立 F0 预测与音高类别及周期/清浊代理输出。模型为 `full` 变体，通道宽度 `192`，音高网格分辨率 `20 cents`，频率范围 `50.0` 至 `1200.0 Hz`。
- `final_analyzer/`：主声学与构音辅助观测分析器，定位为多任务声学观测与传感器描述符预测模型。模型为 `full`（全频）变体，主干宽度 `192`，未启用 `high` 变体的独立高频支路，边界掩码版本 `masking_revision = 2`。

推理接入说明：两个模型导出的 ONNX 文件均采用 opset 18，可直接通过 ONNX Runtime 加载推理。随附的 `weights.pt` 为 state dict 与配置包，并非 `torch.jit` 导出模型，如需直接加载 `.pt` 文件必须自行匹配对应的 PyTorch 网络架构源码。

## 推理接口与输入输出规格

两个模型均要求输入 `48 kHz` 单声道 `float32` 波形，帧移 `hop_samples = 480`（即每 `10 ms` 一帧），STFT 窗口 `fft_size = 2048`。输出总帧数为 `T = floor(N / 480) + 1`，帧中心对齐第 0 个样本（`frame_origin = center_at_sample_0`），帧时间戳公式为 `frame_index * 480 / 48000` 秒，有效帧规则为 `frame_index <= floor(sample_lengths / 480)`。

两个模型的输入张量相同：

- `waveform`：`float32`，形状 `[B, N]`（`final_analyzer` 明确要求有限数值、`B >= 1`、`N >= 1` 的 48 kHz 单声道波形）。
- `sample_lengths`：`int64`，形状 `[B]`（调用方须校验满足 `1 <= length <= N`）。

边界与分块上下文配置如下：

- `final_pitch`：单侧分块上下文 `chunk_context_samples_per_side = 29824`；边界策略为对波形补零并在 LayerNorm 与每个时序模块之后施加掩码。
- `final_analyzer`：单侧分块上下文 `chunk_context_samples_per_side = 121984`；边界策略为第 2 版规则（对波形补零并在 LayerNorm 之后、时序卷积之前施加帧掩码）。

`final_pitch` 输出张量与语义：

| 输出名称 | 形状 | 数据类型 | 说明 |
|---|---|---|---|
| `class_logits` | `[B, T, 277]` | `float32` | 277 个音高类别的未归一化 logits（50.0 至 1200.0 Hz，每 bin 20 cents） |
| `voicing_logit` | `[B, T]` | `float32` | 独立周期/清浊代理判别 logit |
| `log2_f0_hz` | `[B, T]` | `float32` | `argmax ± 2 bins` 局部 softmax 解码的 `log2(Hz)` 音高，不受周期/清浊代理掩码屏蔽，无效填充为 `NaN` |
| `f0_hz` | `[B, T]` | `float32` | 线性 `Hz` 单位音高，与 `log2_f0_hz` 同步且无效填充为 `NaN` |
| `voicing_probability` | `[B, T]` | `float32` | `voicing_logit` 经 sigmoid 得到的周期/清浊代理分数，未作统计校准，不能当成已校准概率 |
| `local_probability_mass` | `[B, T]` | `float32` | `argmax ± 2 bins` 窗口内的分类概率质量，未作统计校准，不代表生理置信度 |
| `frame_times_s` | `[T]` | `float64` | 每帧时间位置（秒） |
| `valid_frames` | `[B, T]` | `bool` | 变长输入的帧网格支持掩码，不代表预测质量或生理置信度 |

`final_analyzer` 输出张量与语义：

| 输出名称 | 形状 | 数据类型 | 说明 |
|---|---|---|---|
| `observations` | `[B, T, 119]` | `float32` | 119 维主声学与构音辅助观测输出 |
| `quality_scores` | `[B, T, 119]` | `float32` | 对应 119 维观测的质量分数，未经统计校准，具体阈值须结合使用场景验证 |
| `controls` | `[B, T, 12]` | `float32` | 12 个现代声乐连续控制轴（物理单位），当前全部未验收，全为 `NaN` |
| `control_valid` | `[B, T, 12]` | `bool` | 控制轴有效掩码，当前恒为 `False` |
| `registration_probs` | `[B, T, 4]` | `float32` | 声区分类头概率（`chest`、`head`、`mix`、`falsetto`），无标签可用，全为 `NaN` |
| `registration_valid` | `[B, T]` | `bool` | 声区分类有效掩码，当前恒为 `False` |
| `onset_probs` | `[B, T, 3]` | `float32` | 起音分类头概率（`balanced`、`breathy`、`glottal`），无标签可用，全为 `NaN` |
| `onset_valid` | `[B, T]` | `bool` | 起音分类有效掩码，当前恒为 `False` |
| `frame_times_s` | `[T]` | `float64` | 每帧时间位置（秒） |
| `valid_frames` | `[B, T]` | `bool` | 变长输入的帧网格支持掩码，不代表预测质量或生理置信度 |

## 119 维观测任务映射

`final_analyzer` 的 `observations` 张量按左闭右开区间划分为 10 组观测，共计 119 维。其中 117 维具备训练标签，但不应将其误解为 117 个声乐控制参数；2 个颤音维度（索引 `[3:5]`）没有可靠训练标签并保持未知。

| 索引区间（左闭右开） | 维度 | 字段名 | 物理或观测含义 | 训练标签状态 |
|---|---:|---|---|---|
| `[0:1]` | 1 | `log2_f0_hz` | `log2(Hz)` 音高估算 | 有标签（EGG 周期参考或 YIN 代理） |
| `[1:2]` | 1 | `voicing` | 周期/清浊代理 | 有标签（未校准代理分数） |
| `[2:3]` | 1 | `egg_contact_quotient` | EGG 接触商（逐周期 35% 幅度阈值定义，不能转为内收真值） | 有标签 |
| `[3:4]` | 1 | `vibrato_rate_hz` | 颤音速率（Hz） | 无可靠训练标签 |
| `[4:5]` | 1 | `vibrato_extent_cents` | 颤音幅度（单边 cents） | 无可靠训练标签 |
| `[5:9]` | 4 | `hf_harmonic_db` | 6–8、8–12、12–16、16–20 kHz 高频谐波相对于满幅的 dB 观测（dBFS） | 有标签（受有效带宽掩码约束） |
| `[9:13]` | 4 | `hf_aperiodic_db` | 6–8、8–12、12–16、16–20 kHz 高频栅格间相对于满幅的 dB 观测（dBFS） | 有标签（谐波间能量近似） |
| `[13:37]` | 24 | `tmri_image_profile` | 归一化实时 MRI 图像剖面描述符 | 有标签 |
| `[37:101]` | 64 | `uti_tongue_contour` | 归一化超声舌轮廓坐标描述符 | 有标签 |
| `[101:119]` | 18 | `ema_articulator_xyz` | 电磁构音仪三维传感器坐标（mm） | 有标签 |

## 未验收的声乐控制与分类范围

现代声乐目标共包含 **12 个连续控制轴与 2 个离散分类头**（`registration`、`onset`）。当前已验收的现代声乐控制轴列表为空（`accepted_control_axes = []`），模型在这 12 个连续轴与 2 个分类头上的覆盖率均为 `0`。接口规范中定义了以下 12 个连续控制轴的物理范围与中性参考值（及对应 `[-1, 1]` 归一化映射），但在当前交付模型中，所有未知或无标签维度一律输出 `NaN` 且对应 `control_valid = False`，严禁在下游用中性值补齐缺失预测，也不得以 `egg_contact_quotient`（其保留 35% 幅度阈值定义，不能转为内收 `adduction` 真值）、高频能量或 MRI/UTI/EMA 描述符直接替代这些控制标签：

| 索引 | 控制轴名称 | 物理范围 | 中性值 | 单位 | 归一化中性值 |
|---:|---|---|---:|---|---:|
| 0 | `breath_support` | `[0, 1]` | `0.5` | `relative` | `0` |
| 1 | `vocal_effort` | `[0, 1]` | `0.5` | `relative` | `0` |
| 2 | `adduction` | `[0, 1]` | `0.5` | `relative` | `0` |
| 3 | `breathiness` | `[0, 1]` | `0.3` | `relative` | `-0.4` |
| 4 | `vocal_weight` | `[0, 1]` | `0.5` | `relative` | `0` |
| 5 | `vowel_openness` | `[0, 1]` | `0.5` | `relative` | `0` |
| 6 | `resonance_brightness` | `[0, 1]` | `0.5` | `relative` | `0` |
| 7 | `twang` | `[0, 1]` | `0.3` | `relative` | `-0.4` |
| 8 | `nasality` | `[0, 1]` | `0.3` | `relative` | `-0.4` |
| 9 | `legato` | `[0, 1]` | `0.7` | `relative` | `0.4` |
| 10 | `vibrato_rate_hz` | `[0, 12]` | `5.5` | `Hz` | `-0.083333` |
| 11 | `vibrato_extent_cents` | `[0, 200]` | `25` | `cents` | `-0.75` |

2 个离散分类头同样因缺乏类别标签而处于未验收的不可用状态：`registration`（类别 `chest`、`head`、`mix`、`falsetto`）与 `onset`（类别 `balanced`、`breathy`、`glottal`）的输出概率均为 `NaN`，且 `registration_valid` 与 `onset_valid` 恒为 `False`。

## 训练数据划分与模型选择策略

训练与评测采用固定数据集，总计 `35,545` 条录音，划分为训练集 `19,616` 条、开发集（验证集）`7,606` 条、测试集 `8,323` 条。数据全部来自既有的 EGG、tMRI、UTI、EMA 及其同步音频。

训练与检查点选择策略如下：

- 主分析器在两个随机种子上共运行 6 组消融与对比训练（比较 `low`、`full` 与 `high` 三种变体），累计 `206,000` 步；最终交付的 `full`（全频）模型实际训练 `40,000` 步，依据开发集指标择优交付第 `19,000` 步检查点。
- 专用音高模型共训练 `21,000` 步，依据开发集指标择优交付第 `8,000` 步检查点。
- 所有候选模型均仅按开发集表现选择，未根据测试集结果重新筛选模型。开发集上的经验平台期仅反映训练过程中的局部收敛情况，既不保证全局最优，也不能证明模型已经排除全部欠拟合或过拟合风险。

## 测试集实测误差与歌唱参考局限

在固定测试集（`8,323` 条原始 WAV 的有效参考区间）上，最终导出 ONNX 模型的实测平均绝对误差（MAE）如下：

| 观测指标 | 最终 ONNX 测试 MAE | 说明与基线对比 |
|---|---:|---|
| 专用音高 F0（`final_pitch`） | `23.349 cents` | 全量 8,323 条测试样本；其中 EGG 周期参考子集为 `11.897 cents`，其余多数样本采用同步 WAV 的 YIN 代理参考 |
| 主分析器 F0（`final_analyzer`） | `71.704 cents` | 同一固定测试集有效参考区间 |
| EGG 接触商（`egg_contact_quotient`） | `0.04364` | 依据逐周期 35% 幅度阈值定义计算，不能转为内收真值 |
| 高频谐波观测（`hf_harmonic_db`） | `1.56162 dB` | 覆盖 6–8、8–12、12–16、16–20 kHz 四频段相对于满幅的 dB 观测，按真实有效带宽掩码评分 |
| 高频栅格间观测（`hf_aperiodic_db`） | `1.63426 dB` | 相对于满幅的谐波间非周期能量近似观测，不能直接等同于气声（`breathiness`）真值 |
| tMRI 描述符（`tmri_image_profile`） | `0.07107` | 归一化图像特征，相对均值基线（`0.07313`）改善幅度有限 |
| UTI 描述符（`uti_tongue_contour`） | `0.02276` | 归一化舌轮廓坐标 |
| EMA 坐标（`ema_articulator_xyz`） | `4.43735 mm` | 三维构音坐标误差，相对均值基线（`4.82930 mm`）改善幅度有限 |

**歌唱数据极度稀疏与代理参考分歧警示**：现有数据中来源明确的真实歌唱仅包含一名呼麦歌者的 3 段录音，实际划分为训练 1 段、开发 1 段、测试 1 段，仅 1 段作为唯一测试录音。在该唯一呼麦测试录音上，模型预测相对于固定 YIN 代理参考的 F0 MAE 极大：`final_pitch` 为 `566.949 cents`，`final_analyzer` 为 `665.944 cents`。由于仅有一名歌者的三段录音，整体测试集 F0 误差绝不能替代歌唱泛化能力验收，也不能仅凭模型与 YIN 代理之间的分歧判定双方何者正确。当前模型的通用歌唱与跨歌者泛化能力均未通过验收。

## 高频观测消融与适用范围

`final_analyzer` 输出覆盖 `6–8`、`8–12`、`12–16` 与 `16–20 kHz` 四个频段的高频声学观测。六图消融实验在两个随机种子上比较了 `low`、`full` 与 `high` 三种变体，结果表明全频输入（`full`）相较低频基线（`low`）体现了高频声学观测收益；面向最终主分析器的实际 ONNX 高频消融评测覆盖了 `4,781` 条测试录音与 `131` 个既定分组，验证了 6 kHz 以上观测相比低频基线的误差改善。最终交付模型依据开发集指标选定为 `full` 变体，该变体未启用 `high` 变体的独立高频支路，上述收益来自全频输入对高频声学特征的建模，而不应归因于额外的独立高频支路。

该声学误差改善存在明确边界：它只代表声学频段观测能力的提升，不能外推为气声（`breathiness`）、`twang`、鼻化（`nasality`）等声乐控制感知能力，也不能推论具备跨歌者的歌唱解析能力。训练与评测中严格依赖原始采样带宽与质量有效掩码；将低采样率音频重采样至 `48 kHz` 并不能凭空创造真实的高频谐波或非周期信息，实际部署时必须保留对输入有效带宽的约束认知。

## 推理用途、限制与数据许可说明

适用用途与工程限制：

- 本仓库模型仅适用于针对语音及既有实验室数据集特征域的 F0 估算、周期/清浊代理评估与声学/构音辅助观测原型研究。
- 所有质量与概率字段（`quality_scores`、`voicing_probability`、`local_probability_mass`）均未进行统计校准，不能当成已校准概率或临床/生理置信指标；若下游需要采用阈值筛选或门控，具体阈值须结合使用场景自行验证。
- `valid_frames` 仅表示变长输入的帧网格支持，不代表预测质量或生理置信度。
- `tmri_image_profile`、`uti_tongue_contour`、`ema_articulator_xyz` 以及保留逐周期 35% 幅度阈值定义的 `egg_contact_quotient`（不能转为内收真值）均是从单声道音频间接预测的观测指标，尤其是 tMRI 与 EMA 相对均值基线增益较小，不可直接解释为可靠的真实人体生理测量参数。

许可与合规边界：

- 仓库既有 [LICENSE](LICENSE) 为 Apache-2.0；上游训练数据条款独立列于 [DATASET_LICENSES.md](DATASET_LICENSES.md) 与 [licenses/](licenses/)；该 `LICENSE` 不替代或重新许可上游数据，也不能据此推定全部训练权重使用已获得上游授权。
- 本仓库不包含原始训练数据集，所有上游数据源的版权与限制见 [DATASET_LICENSES.md](DATASET_LICENSES.md)、[licenses/sources.json](licenses/sources.json) 及 `licenses/texts/` 下的 8 份原始条款文本。
- 上游数据集中 `KEELE`（仅记录分发标记 `other-nc`，原始完整许可条款尚未取得）与 `APLAWD`（未找到明确授权声明，状态未决）均存在许可未决状态。鉴于上游条款包含非商业、未明示再分发或相同方式共享等差异，本模型卡不对交付模型权重的统一商业化授权作出任何担保或结论；任何商业或衍生使用须由使用方自行核验上游条款并承担合规责任。
