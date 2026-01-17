# PCPFM代谢组学数据分析详细教程

## 目录
1. [简介](#简介)
2. [安装配置](#安装配置)
3. [工作流程概览](#工作流程概览)
4. [详细分析步骤](#详细分析步骤)
5. [进阶功能](#进阶功能)
6. [常见问题](#常见问题)

---

## 简介

PCPFM(Python-Centric Pipeline for Metabolomics)是一个全面的LC-MS代谢组学数据预处理流程,可以:
- 将原始数据转换为特征表
- 进行质量控制和标准化
- 批次校正和数据归一化
- MS1和MS2代谢物注释
- 输出标准化格式数据供下游分析

### 核心引用文献
- Mitchell et al., 2024. PLOS Computational Biology
- Li et al., 2023. Nature Communications (Asari软件)

---

## 安装配置

### 1. 基础安装

```bash
# 使用pip安装
pip install pcpfm

# 或从源码安装
git clone https://github.com/shuzhao-li-lab/PythonCentricPipelineForMetabolomics.git
cd PythonCentricPipelineForMetabolomics
pip install -e .
```

### 2. 下载额外资源

```bash
# 下载HMDB、LMSD和MoNA数据库
pcpfm download_extras
```

**注意**: HMDB数据库仅限非商业用途使用

### 3. 系统要求

- Python 3.7+
- 对于.raw文件转换需要mono和ThermoRawFileParser
- 足够的磁盘空间存储原始数据和中间结果

---

## 工作流程概览

```
原始数据(.raw/.mzML)
    ↓
格式转换(如需要)
    ↓
特征提取(Asari)
    ↓
质量控制
    ↓
数据整理(空白掩蔽、归一化、批次校正)
    ↓
经验化合物(EmpCpd)构建
    ↓
注释(MS1/MS2)
    ↓
最终输出
```

---

## 详细分析步骤

### 步骤1: 准备实验元数据

#### 1.1 手动创建序列文件

创建CSV文件包含至少以下字段:

| Sample Type | Name | Filepath |
|------------|------|----------|
| Blank | Sample_01 | path/to/Sample_01.raw |
| QC | Sample_02 | path/to/Sample_02.raw |
| Unknown | Sample_03 | path/to/Sample_03.raw |

**推荐字段**:
- `Sample Type`: 样品类型(Blank/QC/Unknown/STD)
- `Name`: 样品名称
- `Filepath`: 文件路径
- `Batch`: 批次信息(用于批次校正)
- 其他自定义字段

#### 1.2 使用预处理命令(可选)

```bash
pcpfm preprocess \
    -s ./Sequence.csv \
    --new_csv_path ./NewSequence.csv \
    --name_field='Name' \
    --path_field='Path' \
    --preprocessing_config ./preprocessing.json
```

**预处理配置示例** (preprocessing.json):
```json
{
    "sample_type": {
        "qstd": {
            "substrings": ["qstd", "QSTD"],
            "search": ["File Name", "Sample ID"]
        },
        "blank": {
            "substrings": ["blank", "BLANK"],
            "search": ["File Name"]
        }
    }
}
```

---

### 步骤2: 组装实验项目

```bash
pcpfm assemble \
    -s ./sequence.csv \
    --name_field='Name' \
    --path_field='Filepath' \
    -o ./projects \
    -j my_experiment
```

**参数说明**:
- `-s`: 序列文件路径
- `--name_field`: 样品名称字段
- `--path_field`: 文件路径字段
- `-o`: 输出目录
- `-j`: 项目名称

**可选过滤器**:

使用JSON过滤器选择特定样品:
```json
{
    "Method": {
        "includes": ["HILICpos"]
    }
}
```

```bash
# 应用过滤器
pcpfm assemble ... --filter filter.json

# 或排除特定样品
pcpfm assemble ... --skip_list excluded_samples.txt
```

**项目目录结构**:
```
my_experiment/
├── annotations/           # 注释结果
├── asari_results/        # Asari特征提取结果
├── converted_acquisitions/ # mzML文件
├── feature_tables/       # 用户创建的特征表
├── results/              # 最终输出
├── QAQC_figs/           # 质控图表
├── raw_acquisitions/     # 原始.raw文件
└── experiment.json       # 项目配置
```

---

### 步骤3: 数据格式转换

**仅当使用.raw文件时需要**

```bash
pcpfm convert \
    -i ./my_experiment \
    --conversion_command "mono ThermoRawFileParser.exe -f=1 -i \$RAW_PATH -b \$OUT_PATH"
```

**注意**:
- 确保已安装mono
- ThermoRawFileParser会在`download_extras`时自动下载
- `$RAW_PATH`和`$OUT_PATH`是占位符,会被自动替换

---

### 步骤4: 特征提取(Asari)

```bash
# 基础命令(自动推断离子化模式)
pcpfm asari -i ./my_experiment

# 带额外参数
pcpfm asari \
    -i ./my_experiment \
    --extra_asari "--mode pos --ppm 5"
```

**生成的特征表**:
- `full`: 完整特征表
- `preferred`: 首选特征表(推荐用于下游分析)

**Asari核心参数**:
- `--mode`: 离子化模式(pos/neg)
- `--ppm`: 质量精度(默认5ppm)
- 更多参数见Asari文档

---

### 步骤5: 质量控制(QAQC)

```bash
# 运行所有QC检查
pcpfm QAQC \
    -i ./my_experiment \
    --table_moniker preferred \
    --all true

# 或选择特定分析
pcpfm QAQC \
    -i ./my_experiment \
    --table_moniker preferred \
    --pca --pearson --tsne
```

**支持的QC分析**:
- `--pca`: 主成分分析
- `--tsne`: t-SNE降维
- `--pearson`: Pearson相关性
- `--spearman`: Spearman相关性
- `--kendall`: Kendall tau相关性
- `--missing_feature_percentiles`: 缺失特征分布
- `--median_correlation_outlier_detection`: 基于相关性的异常值检测
- `--intensity_analysis`: 强度分析

**输出**:
- 图表保存在`QAQC_figs/`目录
- 结果记录在`experiment.json`

---

### 步骤6: 数据整理和标准化

#### 6.1 空白掩蔽

去除背景离子和污染物:

```bash
pcpfm blank_masking \
    -i ./my_experiment \
    --table_moniker preferred \
    --new_moniker preferred_blank_masked \
    --blank_value Blank \
    --sample_value Unknown \
    --query_field sample_type \
    --blank_intensity_ratio 3
```

**参数说明**:
- `--blank_intensity_ratio`: 样品强度必须是空白的N倍才保留(默认3)
- `--query_field`: 元数据字段名
- `--blank_value`: 空白样品标识
- `--sample_value`: 实验样品标识

#### 6.2 删除不需要的样品

**按元数据字段删除**:
```bash
pcpfm drop_samples \
    -i ./my_experiment \
    --table_moniker preferred_blank_masked \
    --new_moniker preferred_masked_unknown \
    --drop_field sample_type \
    --drop_value QC
```

**反转逻辑(只保留匹配的)**:
```bash
pcpfm drop_samples \
    -i ./my_experiment \
    --table_moniker preferred_blank_masked \
    --new_moniker preferred_masked_unknown \
    --drop_field sample_type \
    --drop_value Unknown \
    --drop_others true
```

**按样品名删除**:
```bash
pcpfm drop_samples \
    -i ./my_experiment \
    --table_moniker preferred_blank_masked \
    --new_moniker cleaned \
    --drop_name sample_001
```

**基于QC结果删除**:
```bash
pcpfm drop_samples \
    -i ./my_experiment \
    --table_moniker preferred_blank_masked \
    --new_moniker cleaned \
    --qaqc_filter qaqc_filter.json
```

qaqc_filter.json示例:
```json
{
    "median_correlation_outlier_detection": {
        "threshold": -2.0,
        "metric": "z_score"
    }
}
```

#### 6.3 归一化

校正样品间仪器变异:

```bash
pcpfm normalize \
    -i ./my_experiment \
    --table_moniker preferred_masked_unknown \
    --new_moniker preferred_normalized \
    --TIC_normalization_percentile 0.90 \
    --normalize_value median
```

**参数说明**:
- `--TIC_normalization_percentile`: 特征必须存在于该比例样品中才用于计算归一化因子(默认0.90)
- `--normalize_value`: median或mean

**批次感知归一化**:
```bash
pcpfm normalize \
    -i ./my_experiment \
    --table_moniker preferred_masked_unknown \
    --new_moniker preferred_normalized \
    --by_batch Batch \
    --TIC_normalization_percentile 0.90
```

#### 6.4 删除低频特征

```bash
pcpfm drop_missing_features \
    -i ./my_experiment \
    --table_moniker preferred_normalized \
    --new_moniker preferred_drop_missing \
    --feature_retention_percentile 0.25
```

**参数说明**:
- `--feature_retention_percentile`: 特征必须存在于该比例样品中才保留(默认0.50)

#### 6.5 缺失值填补

```bash
pcpfm impute \
    -i ./my_experiment \
    --table_moniker preferred_drop_missing \
    --new_moniker preferred_imputed \
    --interpolation_ratio 0.5
```

**参数说明**:
- `--interpolation_ratio`: 最小值的倍数(默认0.5)

#### 6.6 批次校正

```bash
pcpfm batch_correct \
    -i ./my_experiment \
    --table_moniker preferred_imputed \
    --new_moniker batch_corrected \
    --by_batch Batch
```

**注意**:
- 批次校正对缺失值敏感,建议先填补
- 可能需要调整参数以获得最佳结果

#### 6.7 对数转换

```bash
pcpfm log_transform \
    -i ./my_experiment \
    --table_moniker batch_corrected \
    --new_moniker for_analysis \
    --log_transform log2
```

**可选值**:
- `log2`: 以2为底(默认)
- `log10`: 以10为底

---

### 步骤7: 注释

#### 7.1 构建经验化合物(EmpCpd)

将相关特征(同位素、加合物)分组:

```bash
pcpfm build_empCpds \
    -i ./my_experiment \
    --table_moniker preferred \
    --empCpd_moniker preferred
```

**推荐**: 使用`full`或`preferred`特征表构建

**可选参数**:
- `--khipu_mz_tolerance`: ppm容差(默认5)
- `--khipu_rt_tolerance`: 保留时间容差,秒(默认2)
- `--khipu_adducts_pos`: 正离子模式加合物配置
- `--khipu_adducts_neg`: 负离子模式加合物配置

#### 7.2 MS1注释 - Level 4

基于分子式和m/z的数据库匹配:

```bash
# 使用默认数据库(HMDB + LMSD)
pcpfm l4_annotate \
    -i ./my_experiment \
    --empCpd_moniker preferred \
    --new_moniker MS1_annotated

# 使用自定义数据库
pcpfm l4_annotate \
    -i ./my_experiment \
    --empCpd_moniker preferred \
    --new_moniker MS1_annotated \
    --targets my_database.json
```

**注意**: Level 4注释不适用于单峰(singleton)

#### 7.3 MS1注释 - Level 1b

基于标准品库的RT和m/z匹配:

```bash
pcpfm l1b_annotate \
    -i ./my_experiment \
    --empCpd_moniker preferred \
    --new_moniker MS1_auth_std_annotate \
    --targets authentic_standards.csv \
    --annot_mz_tolerance 5 \
    --annot_rt_tolerance 30
```

**标准品CSV格式要求**:
- `Confirm Precursor`: m/z值
- `RT`: 保留时间(分钟)
- `CompoundName`: 化合物名称

#### 7.4 MS2映射(必需步骤)

将实验MS2谱图映射到特征:

```bash
pcpfm map_ms2 \
    -i ./my_experiment \
    --empCpd_moniker MS1_annotated \
    --new_moniker MS2_mapped \
    --annot_mz_tolerance 5 \
    --annot_rt_tolerance 30
```

**外部MS2数据**:
```bash
pcpfm map_ms2 \
    -i ./my_experiment \
    --empCpd_moniker MS1_annotated \
    --new_moniker MS2_mapped \
    --ms2_dir /path/to/additional/mzml
```

#### 7.5 MS2注释 - Level 2

基于参考谱图库的谱图匹配:

```bash
# 使用默认MoNA数据库
pcpfm l2_annotate \
    -i ./my_experiment \
    --empCpd_moniker MS2_mapped \
    --new_moniker MS2_MS1_annotated

# 使用自定义.msp文件
pcpfm l2_annotate \
    -i ./my_experiment \
    --empCpd_moniker MS2_mapped \
    --new_moniker MS2_MS1_annotated \
    --msp_files_pos custom_pos.msp \
    --msp_files_neg custom_neg.msp \
    --ms2_min_peaks 5
```

**参数说明**:
- `--ms2_similarity_metric`: 相似度算法
- `--ms2_min_peaks`: 最小峰数(默认5)

#### 7.6 MS2注释 - Level 1a

基于标准品MS2谱图的匹配:

```bash
pcpfm l1a_annotate \
    -i ./my_experiment \
    --empCpd_moniker MS2_mapped \
    --new_moniker MS2_MS1_annotated \
    --targets compound_discoverer_export.csv \
    --annot_mz_tolerance 5 \
    --annot_rt_tolerance 30
```

---

### 步骤8: 输出结果

#### 8.1 生成三表输出

```bash
pcpfm generate_output \
    -i ./my_experiment \
    -em MS2_MS1_annotated \
    -tm for_analysis
```

**输出文件**(位于`results/`目录):
1. **annotation_table**: 注释信息表
2. **feature_table**: 特征强度表
3. **sample_annotation_table**: 样品元数据表

#### 8.2 生成PDF报告

```bash
pcpfm report \
    -i ./my_experiment \
    --color_by '["genotype"]' \
    --marker_by '["treatment"]' \
    --text_by '["sample_id"]'
```

**可选参数**:
- `--color_by`: 按字段着色
- `--marker_by`: 按字段选择标记
- `--text_by`: 按字段添加标签

#### 8.3 项目摘要

```bash
pcpfm summarize -i ./my_experiment
```

显示所有特征表和EmpCpd及其路径

---

## 进阶功能

### 1. 完整分析流程示例

```bash
# 1. 组装项目
pcpfm assemble -s sequence.csv --name_field Name --path_field Filepath -o . -j my_project

# 2. 转换(如需要)
pcpfm convert -i ./my_project

# 3. 特征提取
pcpfm asari -i ./my_project

# 4. 质控
pcpfm QAQC -i ./my_project --table_moniker preferred --all true

# 5. 数据整理流程
pcpfm blank_masking -i ./my_project --table_moniker preferred --new_moniker step1 \
    --blank_value Blank --sample_value Unknown --query_field sample_type

pcpfm drop_samples -i ./my_project --table_moniker step1 --new_moniker step2 \
    --drop_field sample_type --drop_value QC

pcpfm normalize -i ./my_project --table_moniker step2 --new_moniker step3 \
    --TIC_normalization_percentile 0.90

pcpfm drop_missing_features -i ./my_project --table_moniker step3 --new_moniker step4 \
    --feature_retention_percentile 0.25

pcpfm impute -i ./my_project --table_moniker step4 --new_moniker step5

pcpfm batch_correct -i ./my_project --table_moniker step5 --new_moniker step6 \
    --by_batch Batch

pcpfm log_transform -i ./my_project --table_moniker step6 --new_moniker final

# 6. 注释
pcpfm build_empCpds -i ./my_project --table_moniker preferred --empCpd_moniker empCpd1

pcpfm l4_annotate -i ./my_project --empCpd_moniker empCpd1 --new_moniker empCpd2

pcpfm map_ms2 -i ./my_project --empCpd_moniker empCpd2 --new_moniker empCpd3

pcpfm l2_annotate -i ./my_project --empCpd_moniker empCpd3 --new_moniker empCpd_final

# 7. 输出
pcpfm generate_output -i ./my_project -em empCpd_final -tm final

pcpfm report -i ./my_project
```

### 2. 使用Nextflow

项目包含Nextflow流程文件用于批量处理:

```bash
nextflow run pcpfm_nextflow.nf \
    --input_dir ./data \
    --output_dir ./results
```

### 3. 重置项目

删除所有用户生成的表和EmpCpd,保留Asari结果:

```bash
# 交互式确认
pcpfm reset -i ./my_experiment

# 强制重置
pcpfm reset -i ./my_experiment --force
```

---

## 常见问题

### Q1: 样品名称与文件名不匹配怎么办?
**A**: 确保在序列文件中正确设置`--name_field`和`--path_field`。使用`preprocess`命令可以帮助推断路径。

### Q2: 如何处理多批次数据?
**A**: 
1. 在元数据中添加`Batch`字段
2. 使用`--by_batch Batch`进行批次感知的归一化和批次校正

### Q3: 内存不足怎么办?
**A**:
- 减少并行处理的样品数
- 使用`preferred`而非`full`特征表
- 增加系统虚拟内存

### Q4: 如何自定义加合物和同位素?
**A**: 创建自定义JSON文件指定加合物信息,使用`--khipu_adducts_pos`和`--khipu_adducts_neg`参数。

### Q5: 注释结果不满意怎么办?
**A**:
1. 调整m/z和RT容差参数
2. 使用更合适的参考数据库
3. 组合多种注释策略(Level 1a/1b/2/4)

### Q6: 批次校正失败?
**A**:
- 确保先进行缺失值填补
- 检查批次信息是否正确
- 尝试调整特征保留百分位

### Q7: 如何导出用于MetaboAnalyst的数据?
**A**: 使用`generate_output`命令生成的三个表可直接导入MetaboAnalyst或其他分析工具。

---

## 技术支持

- **GitHub Issues**: https://github.com/shuzhao-li-lab/PythonCentricPipelineForMetabolomics/issues
- **文档**: https://pythoncentricpipelineformetabolomics.readthedocs.io
- **示例教程**: https://github.com/shuzhao-li-lab/asari_pcpfm_tutorials

---

## 最佳实践建议

1. **数据组织**: 始终使用清晰的命名约定和元数据
2. **质量控制**: 在每个主要步骤后运行QAQC
3. **moniker命名**: 使用描述性的moniker名称追踪分析流程
4. **参数记录**: 记录所有使用的参数以确保可重复性
5. **版本控制**: 使用Git追踪分析脚本
6. **数据备份**: 定期备份项目目录

---

**最后更新**: 2026年1月
**PCPFM版本**: 最新稳定版
