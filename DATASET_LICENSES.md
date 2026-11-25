# 原始数据集来源与许可

核对日期：2026-11-25。本仓库只发布训练后的模型、接口元数据与说明，不包含原始音频、EGG、MRI、UTI、EMA 记录或预处理缓存。

现有仓库根目录的 Apache-2.0 `LICENSE` 保留不变。它不替代、撤销或重新许可任何原始数据集条款；本表也不是对训练权重是否触发上游条款、是否可商用的统一法律结论。

15 个训练数据目录对应 13 个不同来源；SVD 两个目录共享一个来源，USC-TIMIT 的 MRI / EMA 共享一个来源。下面按实际训练目录逐一列出。

| 训练数据目录 | 记录数 | 已核实的来源条款 | 实际使用范围与限制 |
|---|---:|---|---|
| [SVD](https://zenodo.org/records/16874898) | 9,565 | CC BY 4.0 | healthy.zip；健康语音/元音子集 |
| [Singing_EGG_Supplement](https://zenodo.org/records/16874898) | 266 | CC BY 4.0 | Gesangsstimme / Sängerstimme；元音与朗读，非歌唱语料 |
| [CMU_Arctic_EGG](http://www.festvox.org/cmu_arctic/) | 3,377 | CMU 原始 COPYING（宽松自定义条款） | bdl / jmk / slt 三名发音人 |
| [PTDB_TUG](https://www.spsc.tugraz.at/databases-and-tools/ptdb-tug-pitch-tracking-database-from-graz-university-of-technology.html) | 4,718 | ODbL 1.0（数据库）＋ DbCL 1.0（单项内容） | 官方同步 MIC / LAR |
| [KEELE](https://zenodo.org/records/3920591) | 10 | 分发记录：other-nc；非商业用途 | 原始许可全文未取得；不能称为标准 CC 许可 |
| [APLAWD](https://www.commsp.ee.ic.ac.uk/~sap/uploads/data/aplawdw.zip) | 10,979 | 未确认明确的授权条款 | 原始包说明文件未声明许可；公开下载不等于开放授权 |
| [Tuvan_Throat_Singing_tMRI](https://zenodo.org/records/4966580) | 3 | CC0 1.0 | MRIaudio / MRImoviesC；一名歌者三段呼麦 |
| [Vocal_Pitch_Resonance_tMRI](https://zenodo.org/records/5026872) | 16 | CC0 1.0 | 实际为口哨研究；filtered_audio / video |
| [USC_TIMIT_tMRI](https://zenodo.org/records/19422914) | 851 | CC BY 4.0 | 10名发音人的 MRI 语音 |
| [Connacht_Palatalization_UTI](https://zenodo.org/records/4934835) | 735 | CC BY 4.0 | 爱尔兰语 Connacht 子集 |
| [Ultraspeech_UTI](https://zenodo.org/records/1194786) | 1,108 | CC BY 4.0 | 仅 PB 一名发音人子集 |
| [SAIT_EMA_Mandarin](https://zenodo.org/records/17506802) | 1,924 | CC BY 4.0 | 仅 A_F001 / A_M001 两人子集 |
| [BY2014_EMA](https://zenodo.org/records/154083) | 672 | CC BY-SA 4.0 | 单发音人 EMA 子集 |
| [EMA_5EMO_Expressive](https://zenodo.org/records/19303746) | 953 | CC BY 4.0 | 三名发音人的情绪语音子集 |
| [USC_TIMIT_EMA](https://zenodo.org/records/19422914) | 368 | CC BY 4.0 | 四名发音人 EMA 子集；与 MRI 不冒称同步配对 |

## 已收录的原始条款正文

- [CC0 1.0](licenses/texts/CC0-1.0.txt)：Tuvan 与口哨 MRI 来源。
- [CC BY 4.0](licenses/texts/CC-BY-4.0.txt)：对应表内来源记录。
- [CC BY-SA 4.0](licenses/texts/CC-BY-SA-4.0.txt)：BY2014；保留其相同方式共享条款，具体适用对象按原文判断。
- [ODbL 1.0](licenses/texts/ODbL-1.0.txt) 与 [DbCL 1.0](licenses/texts/DbCL-1.0.txt)：PTDB-TUG 官方分别用于数据库与单项内容，不能只写其中一种。
- CMU-ARCTIC 的 [bdl](licenses/texts/CMU-ARCTIC-bdl-COPYING.txt)、[jmk](licenses/texts/CMU-ARCTIC-jmk-COPYING.txt)、[slt](licenses/texts/CMU-ARCTIC-slt-COPYING.txt) 原始 COPYING：保留原版权、作者、修改标识与免责声明。

所有来源记录、作者署名、DOI 和核对级别见 [licenses/sources.json](licenses/sources.json)。正文文件来自 Creative Commons、Open Data Commons 或 Festvox 官方地址；并未将自定义条款简化改写成 BSD-3-Clause 等标准许可。

## 尚未取得完整许可原文的来源

### KEELE

实际下载来源为 [Zenodo 3920591](https://zenodo.org/records/3920591) 的 KEELE.zip。该记录的总许可字段为 `other-nc`，描述逐项注明 KEELE “free for noncommercial use”。这是分发者记录，不能替代 Keele 原作者完整条款；本次未取得原始许可全文，不把同记录中其他数据库的许可套用给 KEELE。

### APLAWD

已检查实际来源 `aplawdw.zip` 中的 `APLAWDW/doc/aplawdw.txt` 说明文件；该文件描述采集条件、数据结构和 Mike Brookes（August 2015），未出现明确许可授予。未确认完整数据使用/再分发授权，标为未决；不能因为文件可公开下载就宣称允许商用或自由再分发。原始录音和文本不随模型仓库上传。

## 数据处理说明与署名

训练前对已取得的同步录音进行解码、单声道处理、抗混叠重采样、时间对齐、有效区间掩码和方法定义的辅助标签提取；MRI / UTI 图像描述符为算法提取，EMA 坐标经既定处理。模型训练包括传感器目标到音频预测的监督，不代表部署时直接测量人体生理信号。

上述处理和训练由本项目完成，原始数据的作者及权利归其各自权利人。请按每个来源记录中的作者与 DOI 引用原始工作，尤其遵守 PTDB-TUG 官方引用要求。
