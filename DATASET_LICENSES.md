# 数据集来源与许可说明

核对日期：2026-11-25。本文档说明 Modern Vocal Music Parameter Assisted Transformation Model 训练所用上游数据集的来源、核实后的许可条款、证据限制与署名信息，方便使用者核对合规边界与数据构成。

本仓库根目录仅包含两个模型文件夹、[README.md](README.md)、[MODEL_CARD.md](MODEL_CARD.md)、[DATASET_LICENSES.md](DATASET_LICENSES.md)、既有 Apache-2.0 [LICENSE](LICENSE)、`.gitattributes`、[licenses/sources.json](licenses/sources.json) 以及 [licenses/texts/](licenses/texts) 下的 8 份原始许可条款文件。原始音频、EGG、MRI、UTI、EMA 记录以及预处理缓存均不随本仓库发布。

## 仓库许可与模型边界

仓库既有 [LICENSE](LICENSE) 为 Apache-2.0；上游训练数据条款独立列于本文档与 [licenses/](licenses/)；该 `LICENSE` 不替代或重新许可上游数据，也不能据此推定全部训练权重使用已获得上游授权。鉴于部分上游数据集条款尚未取得完整原文或处于未决状态，本文档既不对模型权重擅自给出统一的商用授权结论，也不直接将个别未决条目推导为全部模型只能非商用或可以商用，具体使用前请结合实际调用场景与上游条款独立评估。

同时，模型能力与训练数据范围存在以下已核实限制，评估时请勿越界解读：

- 现有模型仅输出音高（F0）与声学/构音辅助观测，现代声乐目标的 12 个连续控制轴与 2 个分类头（`registration`、`onset`）均未验收，输出全为 `NaN` 且对应有效掩码全为 `False`。
- 119 维观测中 117 维有训练标签（不等于 117 个声乐控制），2 个颤音维度（其中幅度单位为单边 cents）缺乏训练标签并保持未知；清浊相关输出均为周期/清浊代理，质量与概率分数均未经统计校准，具体阈值须结合使用场景验证。
- 全部训练目录中，来源明确的真实歌唱语料仅有 `Tuvan_Throat_Singing_tMRI` 的 1 名呼麦歌者共 3 段录音，实际划分为训练 1 段、开发 1 段、测试 1 段（仅 1 段作为唯一测试录音）；由于只有一位歌者且样本极少，整体 F0 误差不能代表模型的跨歌者或歌唱泛化能力。
- 模型输出的传感器描述符以及保留逐周期 35% 幅度阈值定义的 EGG 接触商（CQ，不能转为内收真值）均为基于音频的预测观测，不能直接解释为可靠的真实生理测量参数。

## 训练数据集清单（15 个目录 / 13 个来源）

本仓库的模型训练与评测数据总计 `35,545` 条记录（划分为训练集 `19,616` 条、开发集 `7,606` 条、测试集 `8,323` 条），共使用 15 个数据目录，对应 13 个独立上游来源。其中 `SVD` 与 `Singing_EGG_Supplement` 共用同一 Saarbruecken Voice Database 来源，`USC_TIMIT_tMRI` 与 `USC_TIMIT_EMA` 共用同一 USC-TIMIT 来源。各目录记录数、实际使用子集、来源声明许可及证据级别如下表所示：

| 训练目录 | 记录数 | 实际使用子集 | 来源链接 | 来源声明许可 | 证据级别与限制 |
|---|---:|---|---|---|---|
| `SVD` | 9,565 | `healthy.zip`；健康语音与元音子集（681 名受试者，50 kHz） | [Zenodo 16874898](https://zenodo.org/records/16874898) | CC BY 4.0 | `download_source_record`；与 `Singing_EGG_Supplement` 共源 |
| `Singing_EGG_Supplement` | 266 | `Gesangsstimme / Sängerstimme`；元音与朗读，非歌唱语料（19 名受试者，50 kHz） | [Zenodo 16874898](https://zenodo.org/records/16874898) | CC BY 4.0 | `download_source_record`；与 `SVD` 共源 |
| `CMU_Arctic_EGG` | 3,377 | `bdl` / `jmk` / `slt` 三名发音人（32 kHz） | [Festvox CMU_ARCTIC](http://www.festvox.org/cmu_arctic/) | CMU 原始 `COPYING`（宽松自定义条款） | `official_COPYING`；保留原条款，不改称标准 BSD |
| `PTDB_TUG` | 4,718 | 官方同步 `MIC / LAR`（20 名受试者，48 kHz） | [TU Graz PTDB-TUG](https://www.spsc.tugraz.at/databases-and-tools/ptdb-tug-pitch-tracking-database-from-graz-university-of-technology.html) | ODbL 1.0（数据库）＋ DbCL 1.0（单项内容） | `official_license_statement`；两项条款需同时列出 |
| `KEELE` | 10 | 10 名受试者记录（20 kHz） | [Zenodo 3920591](https://zenodo.org/records/3920591) | 分发记录 `other-nc`（非商业用途） | `distributor_record_only`；未取得原作者完整条款，非标准 CC 许可 |
| `APLAWD` | 10,979 | 原始包内 10 名受试者记录（20 kHz） | [aplawdw.zip](https://www.commsp.ee.ic.ac.uk/~sap/uploads/data/aplawdw.zip) | 未确认明确授权条款 | `unresolved_no_explicit_license_found`；包内说明无授权声明，状态未决 |
| `Tuvan_Throat_Singing_tMRI` | 3 | `MRIaudio / MRImoviesC`；1 名歌者 3 段呼麦（训练 1 段 / 开发 1 段 / 测试 1 段，44.1 kHz） | [Zenodo 4966580](https://zenodo.org/records/4966580) | CC0 1.0 | `download_source_record`；唯一来源明确的歌唱数据集，仅一名歌者 |
| `Vocal_Pitch_Resonance_tMRI` | 16 | 实际为口哨研究；`filtered_audio / video`（6 名受试者，44.1 kHz） | [Zenodo 5026872](https://zenodo.org/records/5026872) | CC0 1.0 | `download_source_record` |
| `USC_TIMIT_tMRI` | 851 | 10 名发音人的 MRI 语音（20 kHz） | [Zenodo 19422914](https://zenodo.org/records/19422914) | CC BY 4.0 | `download_source_record`；与 `USC_TIMIT_EMA` 共源 |
| `Connacht_Palatalization_UTI` | 735 | 爱尔兰语 Connacht 方言子集（5 名受试者，44.1 kHz） | [Zenodo 4934835](https://zenodo.org/records/4934835) | CC BY 4.0 | `download_source_record` |
| `Ultraspeech_UTI` | 1,108 | 仅 `PB` 单发音人子集（44.1 kHz） | [Zenodo 1194786](https://zenodo.org/records/1194786) | CC BY 4.0 | `download_source_record` |
| `SAIT_EMA_Mandarin` | 1,924 | 仅 `A_F001 / A_M001` 两名发音人子集（48 kHz） | [Zenodo 17506802](https://zenodo.org/records/17506802) | CC BY 4.0 | `download_source_record` |
| `BY2014_EMA` | 672 | 单发音人 EMA 子集（22.05 kHz） | [Zenodo 154083](https://zenodo.org/records/154083) | CC BY-SA 4.0 | `download_source_record`；含相同方式共享条款 |
| `EMA_5EMO_Expressive` | 953 | 3 名发音人的情绪语音子集（22.05 kHz） | [Zenodo 19303746](https://zenodo.org/records/19303746) | CC BY 4.0 | `download_source_record` |
| `USC_TIMIT_EMA` | 368 | 4 名发音人 EMA 子集（22.05 kHz） | [Zenodo 19422914](https://zenodo.org/records/19422914) | CC BY 4.0 | `download_source_record`；与 MRI 共源但不冒称同步配对 |

## 未决或受限来源核查细节

### KEELE

实际下载来源为 [Zenodo 3920591](https://zenodo.org/records/3920591) 中的 `KEELE.zip`。该合集记录的总许可元数据为 `other-nc`，说明文字逐项注明 KEELE 属于“free for noncommercial use”。这仅代表第三方分发者在托管页面填写的记录，不能替代 Keele 原作者发布的完整许可全文，也不能将其归类为标准知识共享（CC）许可或把同一合集中其他语料库的条款套用给 KEELE。

### APLAWD

实际下载压缩包为 `aplawdw.zip`。已直接检查包内的 `APLAWDW/doc/aplawdw.txt` 说明文档：该文档详细描述了采集条件、数据结构以及归属信息（Mike Brookes，August 2015），但全文未包含任何明确的许可授予条款。因此 APLAWD 的数据使用与再分发授权仍标记为未决，不能将“可公开下载”等同于“已授予开放或商业许可”。

## 已收录的原始许可正文与来源元数据

仓库已将核实所得的 8 份原始许可正文完整保存于 [licenses/texts/](licenses/texts) 目录，未对任何原始条款作改写或简化：

- [CC0 1.0](licenses/texts/CC0-1.0.txt)：适用于 `Tuvan_Throat_Singing_tMRI` 与 `Vocal_Pitch_Resonance_tMRI`。
- [CC BY 4.0](licenses/texts/CC-BY-4.0.txt)：适用于表格中对应声明 `cc-by-4.0` 的 8 个 Zenodo 训练目录。
- [CC BY-SA 4.0](licenses/texts/CC-BY-SA-4.0.txt)：适用于 `BY2014_EMA`，保留原始相同方式共享条款，其具体适用范围需按原文自行判断。
- [ODbL 1.0](licenses/texts/ODbL-1.0.txt) 与 [DbCL 1.0](licenses/texts/DbCL-1.0.txt)：`PTDB_TUG` 官方分别用于数据库整体与单项内容，两者须同时并列说明。
- CMU-ARCTIC 原始条款：[bdl COPYING](licenses/texts/CMU-ARCTIC-bdl-COPYING.txt)、[jmk COPYING](licenses/texts/CMU-ARCTIC-jmk-COPYING.txt)、[slt COPYING](licenses/texts/CMU-ARCTIC-slt-COPYING.txt)。三份文件均为 CMU 自定义宽松条款，保留原始版权、作者、修改标识与免责声明，不擅自改称为标准 BSD 许可。

已取得的来源地址、作者或 DOI、核对日期及实际子集见 [licenses/sources.json](licenses/sources.json)（不同来源在元数据中的字段完整度存在差异，该文件也不包含数据集划分统计）。

## 数据预处理简述与原作者署名

在模型训练前，项目对上述已取得的同步录音与传感器数据执行了统一预处理，包括音频解码、单声道转换、抗混叠重采样、时间对齐、有效区间掩码以及基于方法定义的辅助标签提取；其中 MRI 与 UTI 图像描述符由算法提取，EMA 坐标经过既定流程处理。模型训练包含以传感器目标或声学辅助特征监督音频侧预测的过程，用于观测声学与构音关联，并不代表模型在推理部署时直接测量人体生理信号。

原始数据本身的著作权及相关权利仍归各自的原作者与权利人所有。若您的研究或工程工作基于本仓库成果，请参考 [licenses/sources.json](licenses/sources.json) 中已记录的来源地址、作者或 DOI 信息规范引用对应原始数据集，并特别留意遵守 `PTDB_TUG` 等来源在官方页面提出的引用要求。
