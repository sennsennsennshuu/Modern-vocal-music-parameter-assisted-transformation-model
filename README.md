# Multimodal Phonation and Articulation Data Transformation model

本仓库的 Markdown 文档由 **Gemini 4 氢（Gemini 4 Hydrogen）** 编写。

仓库 `sennsennsennshuu/Multimodal-Phonation-and-Articulation-Data-Transformation-model` 仅发布导出的模型权重与许可元数据，包含 `final_pitch/` 与 `final_analyzer/` 两个模型目录，不包含训练源码、`analyzer` Python 包、原始数据集及预处理缓存。

本项目定位为**音高与声学/构音辅助观测模型**：根据 48 kHz 单声道音频逐帧预测基础音高（专用音高模型范围为 50–1200 Hz）、周期/清浊代理、EGG 接触商（CQ）、高频频带能量及 tMRI / UTI / EMA 构音描述符。模型**不包含**任何已验收的现代声乐控制参数预测能力。

## 两个模型对照

| 维度 | `final_pitch/`（专用音高模型） | `final_analyzer/`（主分析器模型） |
|---|---|---|
| ONNX 文件 | `final_pitch/pitch.onnx`（opset 18） | `final_analyzer/analyzer.onnx`（opset 18） |
| 输入签名 | `waveform: float32[B, N]`, `sample_lengths: int64[B]` | `waveform: float32[B, N]`, `sample_lengths: int64[B]` |
| 帧移 / FFT | 480 样本（10 ms）/ 2048 点 | 480 样本（10 ms）/ 2048 点 |
| 音高范围 / 单侧上下文 | 50–1200 Hz；单侧上下文 29,824 样本 | 单侧上下文 121,984 样本 |
| 训练与交付步数 | 训练 21,000 步，交付第 8,000 步 | 训练 40,000 步，交付第 19,000 步 |
| 固定测试集 F0 MAE | 23.349 cents（8,323 条完整原始 WAV） | 71.704 cents（同测试集有效参考区间） |
| 主要输出 | 277 类音高 logits（50–1200 Hz）、F0（Hz / log2 Hz）、周期/清浊代理与局部概率质量 | 119 维声学/构音辅助观测、未校准质量分数及占位控制与分类张量 |

上述 23.349 cents 指标主要来源于语音与元音样本。训练与评测数据中来源明确的真实歌唱仅有一名呼麦歌者的 3 段录音（训练 1 段、开发 1 段、测试 1 段），其中唯一测试录音相对 YIN 代理的 MAE 为 566.949 cents（主模型为 665.944 cents），跨歌者与通用歌唱泛化能力尚未通过验收。

## 目录结构

模型文件与原始交付保持一致。原始许可文件的来源 URL 见 `licenses/sources.json`；原始数据集与预处理缓存不随本仓库发布。

```text
.
├── .gitattributes                           # 仓库文本与二进制文件属性配置
├── LICENSE                                  # 仓库既有 Apache-2.0；上游数据条款另见 DATASET_LICENSES.md
├── MODEL_CARD.md                            # 模型交付事实与能力边界说明
├── DATASET_LICENSES.md                      # 原始训练数据来源与许可详细说明
├── final_pitch/
│   ├── pitch.onnx                           # 专用音高 ONNX 模型（opset 18）
│   ├── weights.pt                           # state dict 与配置包（需匹配架构源码，不可 torch.jit.load）
│   └── metadata.json                        # 导出元数据与张量规范
├── final_analyzer/
│   ├── analyzer.onnx                        # 主分析器 ONNX 模型（opset 18）
│   ├── weights.pt                           # state dict 与配置包（需匹配架构源码，不可 torch.jit.load）
│   └── metadata.json                        # 导出元数据与接口规范
└── licenses/
    ├── sources.json                         # 15 个训练数据目录对应来源与原始 URL
    └── texts/
        ├── CC0-1.0.txt
        ├── CC-BY-4.0.txt
        ├── CC-BY-SA-4.0.txt
        ├── ODbL-1.0.txt
        ├── DbCL-1.0.txt
        ├── CMU-ARCTIC-bdl-COPYING.txt
        ├── CMU-ARCTIC-jmk-COPYING.txt
        └── CMU-ARCTIC-slt-COPYING.txt
```

`weights.pt` 仅保存权重 state dict 与配置包，重建 PyTorch 网络需要配套的架构源码，**不能直接通过 `torch.jit.load` 加载**；直接推理请使用 ONNX 模型。

## 48 kHz 单声道输入与 10 ms 帧网格

两个模型共享同一套输入与帧网格规范：

- **音频规格**：`48000` Hz 采样率、单声道（`channels=1`）、`float32` 有限值波形；批量大小 `B >= 1`、采样点数 `N >= 1`。调用方需保证 `1 <= sample_lengths[b] <= N`。
- **10 ms 帧网格**：`hop_samples = 480`（即每帧 10 ms），`fft_size = 2048`，帧中心对齐第 0 个样本（`center_at_sample_0`）。
- **帧数与时间戳公式**：总帧数 `T = floor(N / 480) + 1`，第 `i` 帧时间戳 `frame_times_s[i] = i * 480 / 48000`（单位：秒，dtype 为 `float64`）。
- **有效帧掩码**：当 `frame_index <= floor(sample_lengths / 480)` 时 `valid_frames` 为 `True`。`valid_frames` 仅表示变长输入的帧网格支持，不代表预测质量或生理置信度；部分无效输出位为 `NaN`。

## ONNX Runtime 推理示例

安装最小运行依赖（使用清华 PyPI 镜像）：

```bash
python -m pip install -i https://pypi.tuna.tsinghua.edu.cn/simple numpy soundfile onnxruntime
```

以下脚本使用相对路径读取当前目录下的 `example.wav`，执行输入校验后调用 `final_pitch/pitch.onnx`，并按输出名保存为 `.npz`：

```python
from pathlib import Path
import numpy as np
import onnxruntime as ort
import soundfile as sf

# 读取单声道 48 kHz 音频，并检查非空与有限值
audio, sample_rate = sf.read(Path("example.wav"), dtype="float32", always_2d=False)
if sample_rate != 48000:
    raise ValueError(f"仅支持 48000 Hz 采样率，当前为 {sample_rate} Hz")
if audio.ndim != 1:
    raise ValueError(f"仅支持单声道输入，当前通道形状为 {audio.shape}")
if audio.size == 0 or not np.all(np.isfinite(audio)):
    raise ValueError("音频为空或包含 NaN/Inf 非有限值")

# 构造批大小 B=1 的 float32[1, N] 波形，以及形状为 [1] 的一维 int64 sample_lengths
num_samples = audio.shape[0]
waveform = np.ascontiguousarray(audio[np.newaxis, :], dtype=np.float32)
sample_lengths = np.array([num_samples], dtype=np.int64)

# 加载专用音高模型并按输出张量名称保存结果
session = ort.InferenceSession("final_pitch/pitch.onnx", providers=["CPUExecutionProvider"])
output_names = [out.name for out in session.get_outputs()]
outputs = session.run(
    output_names,
    {"waveform": waveform, "sample_lengths": sample_lengths},
)
np.savez("pitch_outputs.npz", **dict(zip(output_names, outputs)))
```

将上述加载路径替换为 `final_analyzer/analyzer.onnx` 并保持相同的 `waveform` 与 `sample_lengths` 输入，即可获得主分析器的全部输出字段（`observations`、`quality_scores`、`controls`、`control_valid`、`registration_probs`、`registration_valid`、`onset_probs`、`onset_valid`、`frame_times_s`、`valid_frames`）。

本示例面向单段内存音频推理，未实现长音频分块。长录音分块推理需结合模型单侧上下文长度（`final_pitch` 为 29,824 样本，`final_analyzer` 为 121,984 样本）与全局 480 样本 hop 网格对齐，本仓库不提供分块封装或内置 CLI 命令。

## 观测字段解释

### `final_pitch/pitch.onnx` 输出

- `class_logits`（`float32[B, T, 277]`）与 `log2_f0_hz` / `f0_hz`（`float32[B, T]`）：对应 50–1200 Hz 训练与解码范围，通过 `argmax +/- 2 bins` 局部 softmax 解码得到对数音高与 Hz 音高。F0 输出未经过周期/清浊代理掩码过滤（`masked_by_predicted_voicing=false`），F0 无效填充为 `NaN`，且不属于具名控制参数。
- `voicing_logit` / `voicing_probability`（`float32[B, T]`）：独立周期/清浊代理的 logit 及其 sigmoid 分数。该分数未作统计校准，不能当成已校准概率或可靠的生理清浊判断；如用于阈值筛选，具体阈值须结合使用场景验证。
- `local_probability_mass`（`float32[B, T]`）：`argmax +/- 2 bins` 内的分类概率质量，未作统计校准，不能当成已校准概率，也不代表生理置信度；如设置阈值须结合使用场景验证。
- `frame_times_s`（`float64[T]`）与 `valid_frames`（`bool[B, T]`）：帧时间轴与变长输入的帧网格支持掩码（不代表预测质量或生理置信度）。

### `final_analyzer/analyzer.onnx` 输出

- `observations`（`float32[B, T, 119]`）与 `quality_scores`（`float32[B, T, 119]`）：119 维声学与构音辅助观测及未校准质量分数。这些维度均为纯音频预测的辅助观测，**不等于直接仪器测量或具名声乐控制参数**；其中 117 维存在训练标签（不等于 117 个声乐控制），2 维颤音指标无训练标签并保持未知。
- 119 维任务映射表（索引采用 Python 左闭右开区间；四个高频频段分别为 6–8、8–12、12–16、16–20 kHz）：

| 索引区间 | 字段名 | 维度 | 单位与定义 |
|---|---|---:|---|
| `[0:1]` | `log2_f0_hz` | 1 | `log2(Hz)` |
| `[1:2]` | `voicing` | 1 | 周期/清浊代理（未校准分数） |
| `[2:3]` | `egg_contact_quotient` | 1 | 逐周期 35% 幅度阈值定义（不能转为内收真值） |
| `[3:4]` | `vibrato_rate_hz` | 1 | Hz（无训练标签，保持未知） |
| `[4:5]` | `vibrato_extent_cents` | 1 | 单边 cents（无训练标签，保持未知） |
| `[5:9]` | `hf_harmonic_db` | 4 | 四高频段（6–8、8–12、12–16、16–20 kHz）相对于满幅的 dB 观测（dBFS） |
| `[9:13]` | `hf_aperiodic_db` | 4 | 四高频段谐波栅格间相对于满幅的近似 dB 观测（dBFS） |
| `[13:37]` | `tmri_image_profile` | 24 | 24 维归一化图像描述符 |
| `[37:101]` | `uti_tongue_contour` | 64 | 64 维归一化坐标 |
| `[101:119]` | `ema_articulator_xyz` | 18 | 18 维 mm |

- 现代声乐目标共包含 **12 个连续控制轴与 2 个分类头**，当前均未通过验收且全部未知：
  - `controls`（`float32[B, T, 12]`）与 `control_valid`（`bool[B, T, 12]`）：按接口规范保留的 12 个占位连续控制轴（`breath_support`、`vocal_effort`、`adduction`、`breathiness`、`vocal_weight`、`vowel_openness`、`resonance_brightness`、`twang`、`nasality`、`legato`、`vibrato_rate_hz`、`vibrato_extent_cents`）。当前 `accepted_control_axes` 为空，输出全为 `NaN` 且 `control_valid` 全为 `False`，不可将缺失输出替换为 0.5 或 0 等中性值作为控制结果使用。
  - `registration_probs`（`float32[B, T, 4]`，`chest` / `head` / `mix` / `falsetto`）与 `registration_valid`（`bool[B, T]`）、`onset_probs`（`float32[B, T, 3]`，`balanced` / `breathy` / `glottal`）与 `onset_valid`（`bool[B, T]`）：2 个离散分类头均缺少类别标签，均未验收，输出全为 `NaN` 且对应 valid 掩码全为 `False`。

## 测试结果与局限

以下为两个最终交付 ONNX 模型在测试集上的实测 MAE，详情另见 `MODEL_CARD.md`：

| 观测指标 | 最终 ONNX 测试 MAE | 说明与条件 |
|---|---:|---|
| 专用音高 F0 | 23.349 cents | 50–1200 Hz；8,323 条完整原始 WAV（EGG 周期参考子集 11.897 cents，其余多数为 YIN 代理参考） |
| 主分析器 F0 | 71.704 cents | 同一固定测试集的有效参考区间 |
| 呼麦测试音高（能力边界） | 566.949 cents / 665.944 cents | 专用音高 566.949 cents、主分析器 665.944 cents（相对固定 YIN 代理）；真实歌唱仅一名呼麦歌者三段录音（训练 1 / 开发 1 / 测试 1），此为唯一测试录音结果 |
| EGG 接触商（CQ） | 0.04364 | 逐周期 35% 幅度阈值定义，不能转为内收真值 |
| 高频谐波观测 | 1.56162 dB | 6–8 / 8–12 / 12–16 / 16–20 kHz 相对于满幅的 dB 观测，按真实覆盖掩码评分 |
| 高频栅格间观测 | 1.63426 dB | 相对于满幅的谐波间近似能量观测，不能直接视为气声真值 |
| tMRI 描述符 | 0.07107 | 归一化图像特征，均值基线 0.07313，收益有限 |
| UTI 描述符 | 0.02276 | 归一化舌轮廓坐标 |
| EMA 坐标 | 4.43735 mm | 均值基线 4.82930 mm，收益有限 |

能力边界与使用局限：

- **歌唱泛化未验收**：整体 23.349 cents 指标主要反映语音与元音样本表现；来源明确的真实歌唱仅有 1 名呼麦歌者的 3 段录音（训练 1 段、开发 1 段、测试 1 段），唯一测试录音相对 YIN 代理的 MAE 为 566.949 cents（主分析器 665.944 cents）。仅凭模型与代理参考的分歧无法判定真值归属，且只有一位歌者，跨歌者与通用歌唱泛化能力尚未通过验收。
- **高频成分与带宽边界**：六图消融在两个随机种子上比较了 `low`、`full` 与 `high` 三种变体，全频输入（`full`）相较低频基线（`low`）体现高频声学观测收益；最终交付模型按开发集选定为 `full`（未启用 `high` 变体的独立高频支路）。面向最终主模型的实际 ONNX 高频消融覆盖 4,781 条测试录音与 131 个既定分组，6 kHz 以上成分预测经过测试并体现声学观测收益，但不能据此推论具名声乐控制能力；将低采样率音频重采样至 48 kHz 也不会增加源信号真实带宽。
- **训练收敛说明**：主模型采用两随机种子共训练六组、累计 206,000 步；最终交付模型依据开发集选出（主分析器 40,000 步选第 19,000 步，专用音高 21,000 步选第 8,000 步）。经验平台期不代表全局最优或已排除全部欠拟合与过拟合。

## 训练数据来源与许可证

数据集总计 **35,545** 条记录（划分为 `train` **19,616** / `validation` **7,606** / `test` **8,323**），主体为语音与元音语料，另有口哨（16 条）及一名歌者的呼麦（3 条：训练 1 条、开发 1 条、测试 1 条）。训练仅使用既有 EGG、tMRI、UTI、EMA 及其同步音频，对应 15 个训练数据目录、13 个上游来源（`SVD` 与 `Singing_EGG_Supplement` 共用同一 Zenodo 记录，`USC_TIMIT_tMRI` 与 `USC_TIMIT_EMA` 共用同一 Zenodo 记录）。

| 数据目录 | 样本数 | 使用范围说明 | 报告条款 / 状态 | 上游来源 |
|---|---:|---|---|---|
| `SVD` | 9,565 | `healthy.zip`；健康语音/元音子集 | CC BY 4.0 | https://zenodo.org/records/16874898 |
| `Singing_EGG_Supplement` | 266 | Gesangsstimme / Sängerstimme；元音与朗读，非歌唱语料 | CC BY 4.0 | https://zenodo.org/records/16874898 |
| `CMU_Arctic_EGG` | 3,377 | `bdl` / `jmk` / `slt` 三名发音人 | CMU 原始 `COPYING`（宽松自定义条款） | http://www.festvox.org/cmu_arctic/ |
| `PTDB_TUG` | 4,718 | 官方同步 MIC / LAR | ODbL 1.0（数据库）+ DbCL 1.0（单项内容） | https://www.spsc.tugraz.at/databases-and-tools/ptdb-tug-pitch-tracking-database-from-graz-university-of-technology.html |
| `KEELE` | 10 | 原始许可全文未取得；不能称为标准 CC 许可 | 分发记录：`other-nc`；非商业用途 | https://zenodo.org/records/3920591 |
| `APLAWD` | 10,979 | 原始包说明文件未声明许可；公开下载不等于开放授权 | 未确认明确的授权条款 | https://www.commsp.ee.ic.ac.uk/~sap/uploads/data/aplawdw.zip |
| `Tuvan_Throat_Singing_tMRI` | 3 | `MRIaudio` / `MRImoviesC`；一名歌者三段呼麦（训练 1 / 开发 1 / 测试 1） | CC0 1.0 | https://zenodo.org/records/4966580 |
| `Vocal_Pitch_Resonance_tMRI` | 16 | 实际为口哨研究；`filtered_audio` / `video` | CC0 1.0 | https://zenodo.org/records/5026872 |
| `USC_TIMIT_tMRI` | 851 | 10 名发音人的 MRI 语音 | CC BY 4.0 | https://zenodo.org/records/19422914 |
| `Connacht_Palatalization_UTI` | 735 | 爱尔兰语 Connacht 子集 | CC BY 4.0 | https://zenodo.org/records/4934835 |
| `Ultraspeech_UTI` | 1,108 | 仅 PB 一名发音人子集 | CC BY 4.0 | https://zenodo.org/records/1194786 |
| `SAIT_EMA_Mandarin` | 1,924 | 仅 `A_F001` / `A_M001` 两人子集 | CC BY 4.0 | https://zenodo.org/records/17506802 |
| `BY2014_EMA` | 672 | 单发音人 EMA 子集 | CC BY-SA 4.0 | https://zenodo.org/records/154083 |
| `EMA_5EMO_Expressive` | 953 | 三名发音人的情绪语音子集 | CC BY 4.0 | https://zenodo.org/records/19303746 |
| `USC_TIMIT_EMA` | 368 | 四名发音人 EMA 子集；与 MRI 不冒称同步配对 | CC BY 4.0 | https://zenodo.org/records/19422914 |

许可边界说明：

- 仓库既有 `LICENSE` 为 **Apache-2.0**；上游训练数据条款独立列于 `DATASET_LICENSES.md` 与 `licenses/`；该 `LICENSE` 不替代或重新许可上游数据，也不能据此推定全部训练权重使用已获得上游授权。
- `PTDB_TUG` 适用 ODbL 1.0 + DbCL 1.0；`BY2014_EMA` 适用 CC BY-SA 4.0；`CMU_Arctic_EGG` 保留官方原始 `COPYING` 条款（未改写为标准 BSD）。
- `KEELE` 仅记录分发标记为 `other-nc`（非商业用途），原始完整许可条款尚未取得；`APLAWD` 尚未找到明确授权声明。
- 完整许可梳理、上游原始 URL 及条款文本分别见 [DATASET_LICENSES.md](DATASET_LICENSES.md)、[licenses/sources.json](licenses/sources.json) 与 [licenses/texts/](licenses/texts/)。

## 致谢

感谢上述 13 个上游公开语料记录（覆盖 EGG、tMRI、UTI 与 EMA 同步观测数据）的原始采集作者与维护机构提供研究数据，同时感谢 ONNX Runtime、NumPy 与 soundfile 社区提供稳定的本地推理与音频基础工具。
