我来为您提供 asari 工具的详细使用教程。让我先查看一下这个工具的最新信息。# Asari 代谢组学数据分析详细教程

我将为您提供一份全面的 asari 使用教程。Asari 是一个用于高分辨率 LC-MS 和 GC-MS 代谢组学数据预处理的 Python 工具。

## 📋 目录
1. 安装与环境配置
2. 数据准备
3. 基本使用流程
4. 高级功能
5. 结果解读
6. 常见问题

---

## 1️⃣ 安装与环境配置

### 安装方法

**方法一：使用 pip 安装（推荐）**
```bash
pip3 install asari-metabolomics
```

**方法二：从源码安装**
```bash
git clone https://github.com/shuzhao-li-lab/asari
cd asari
pip install -e .
```

### 系统要求
- Python 3.8 或更高版本
- 注意：Python 3.12+ 目前与 GC 工作流不兼容

### 验证安装
```bash
asari -h
```

---

## 2️⃣ 数据准备

### 输入文件格式
Asari 需要 **centroid 模式的 mzML 文件**

### 文件转换

**从 Thermo .RAW 文件转换：**
```bash
# 方法一：使用 asari 内置转换
asari convert --input /path/to/raw/files

# 方法二：在处理时自动转换
asari process --mode pos --input /path/to/data --convert_raw True
```

**从其他格式转换：**
使用 ProteoWizard 的 msconvert 工具

### 数据组织结构
建议将所有样本的 mzML 文件放在同一个目录下：
```
my_project/
├── sample_001.mzML
├── sample_002.mzML
├── sample_003.mzML
└── ...
```

---

## 3️⃣ 基本使用流程

### 步骤 1: 单文件分析（可选，用于参数优化）

在处理整个数据集之前，建议先分析单个文件以了解数据特征：

```bash
asari analyze --input /path/to/single_file.mzML
```

这会生成统计描述，帮助您：
- 了解数据质量
- 确定合适的参数
- 评估信噪比

### 步骤 2: 批量处理数据

**基本命令：**
```bash
asari process --mode pos --input /path/to/data_directory
```

**带自动参数优化：**
```bash
asari process --mode pos --input /path/to/data_directory --autoheight True
```

**指定输出目录：**
```bash
asari process --mode pos --input /path/to/data --output /path/to/results
```

### 步骤 3: 查看处理结果

处理完成后会生成一个结果目录，结构如下：
```
projectname_asari_project_XXXXXX/
├── preferred_Feature_table.tsv       # 推荐使用的特征表
├── Feature_annotation.tsv            # 特征注释
├── Annotated_empricalCompounds.json
├── export/
│   ├── full_Feature_table.tsv       # 完整特征表
│   ├── unique_compound__Feature_table.tsv
│   ├── cmap.pickle
│   └── _mass_grid_mapping.csv
├── pickle/                           # 中间文件
└── project.json                      # 项目元数据
```

---

## 4️⃣ 高级功能

### A. 工作流选择

**LC-MS 数据（默认）：**
```bash
asari process --mode pos --input /path/to/data
```

**GC-MS 数据：**
```bash
asari process --workflow GC --input /path/to/data \
  --retention_index_standards /path/to/ri_standards.csv
```

**查看可用工作流：**
```bash
asari list_workflows
```

### B. 质量控制 (QC)

**目标化合物提取：**
```bash
asari extract --input /path/to/data --target target_mzs.txt
```

**生成 QC 报告：**
```bash
asari qc_report --input /path/to/data --spikeins spike_in_list.csv
```

**在处理过程中生成 QC 报告：**
```bash
asari process --mode pos --input /path/to/data \
  --single_file_qc_reports true \
  --spikeins spike_in_list.csv
```

### C. 特征注释

```bash
asari annotate --mode pos --ppm 10 \
  --input /path/to/feature_table.tsv
```

### D. 可视化仪表板

处理完成后启动交互式仪表板：
```bash
asari viz --input /path/to/results_directory
```

这会在浏览器中打开一个交互式界面，可以：
- 查看色谱图
- 检查峰形质量
- 浏览特征表
- 验证峰检测结果

### E. 自定义参数

**创建参数文件（YAML 格式）：**
```yaml
# my_parameters.yaml
mass_precision_ppm: 5
ionization_mode: 'pos'
min_intensity_threshold: 1000
min_peak_height: 5000
```

**使用自定义参数：**
```bash
asari process --parameters my_parameters.yaml --input /path/to/data
```

### F. 负离子模式

```bash
asari process --mode neg --input /path/to/data
```

---

## 5️⃣ 重要参数说明

### 关键参数

| 参数 | 说明 | 默认值 | 建议 |
|------|------|--------|------|
| `--mode` | 离子化模式 | pos | pos/neg |
| `--ppm` | m/z 精度 (ppm) | 5 | 根据仪器调整 |
| `--autoheight` | 自动估计最小峰高 | False | 推荐设为 True |
| `--workflow` | 工作流类型 | LC | LC/GC/LC_START |
| `--keep_intermediates` | 保留中间文件 | False | 大数据集建议 False |
| `--storage_format` | 存储格式 | pickle | pickle/json |
| `--compress` | 压缩中间文件 | False | 节省空间时使用 |

### 仪器特定建议

**高分辨率质谱（Orbitrap, FTICR）：**
```bash
asari process --mode pos --ppm 5 --input /path/to/data
```

**TOF 质谱：**
```bash
asari process --mode pos --ppm 10 --input /path/to/data
```

---

## 6️⃣ 结果解读

### 主要输出文件

**1. preferred_Feature_table.tsv**
- 推荐使用的特征表
- 包含跨样本对齐的特征
- 列包括：m/z, RT, 强度等

**2. export/full_Feature_table.tsv**
- 完整的特征表
- 包含所有检测到的峰
- 即使仅在单个样本中出现的特征也会保留

**3. Feature_annotation.tsv**
- 特征的注释信息
- 包含可能的化合物匹配

### 数据质量指标

Asari 追踪以下选择性指标：
- **mSelectivity**: m/z 测量的区分度
- **cSelectivity**: 色谱洗脱峰的区分度

---

## 7️⃣ 典型工作流程示例

### 示例 1: 标准正离子模式分析

```bash
# 步骤 1: 分析单个文件了解数据
asari analyze --input data/sample_001.mzML

# 步骤 2: 处理整个数据集
asari process --mode pos --input data/ \
  --autoheight True \
  --output results/

# 步骤 3: 生成 QC 报告
asari qc_report --input results/ --spikeins qc_standards.csv

# 步骤 4: 启动可视化仪表板
asari viz --input results/
```

### 示例 2: GC-MS 分析

```bash
# 使用 GC 工作流和保留指数标准
asari process --workflow GC \
  --input gc_data/ \
  --retention_index_standards ri_standards.csv \
  --GC_Database MoNA_GCMS \
  --output gc_results/
```

### 示例 3: 大规模研究（1000+ 样本）

```bash
# 使用压缩节省空间，保留中间文件以便重新分析
asari process --mode pos \
  --input large_study/ \
  --compress true \
  --keep_intermediates True \
  --storage_format json \
  --output large_study_results/
```

---

## 8️⃣ 常见问题解决

### Q1: 内存不足
**解决方案：**
- 使用 `--compress true`
- 不使用 `--keep_intermediates`
- 分批处理数据

### Q2: 处理速度慢
**可能原因：**
- 样本数量过多
- 数据复杂度高

**解决方案：**
- 考虑使用分布式处理（Dask）
- 确保有足够的 CPU 资源

### Q3: m/z 值不准确
**解释：**
- Mass tracks 是数据组装的支架
- 真实 m/z 值将通过注释恢复
- 如需更高精度，降低 ppm 值（如 `--ppm 3`）

---

## 9️⃣ 图形界面（GUI）

如果不喜欢命令行，可以使用图形界面：

```bash
asari_gui
```

或使用在线版本：https://asari.app

---

## 🔟 Docker 使用

如果遇到安装问题，可以使用 Docker：

```bash
# 构建镜像
docker build -t asari -f prod.Dockerfile .

# 运行容器（映射数据目录）
docker run -v /local/data:/container/data -ti asari bash

# 在容器内运行 asari
asari process --mode pos --input /container/data
```

---

## 📚 学习资源

- **官方文档**: https://asari.readthedocs.io
- **教程**: https://github.com/shuzhao-li-lab/asari_pcpfm_tutorials
- **论文**: [Nature Communications 14.1 (2023): 4113](https://www.nature.com/articles/s41467-023-39889-1)
- **Web 服务器**: https://asari.app
- **完整流程**: https://pypi.org/project/pcpfm/

---

## 💡 最佳实践建议

1. **数据质量第一**: 处理前确保数据质量良好
2. **参数优化**: 使用 `analyze` 命令了解数据特征
3. **使用自动化**: `--autoheight True` 通常效果很好
4. **保存中间结果**: 大型研究建议保留中间文件
5. **可视化验证**: 使用 `viz` 命令验证结果
6. **版本控制**: 记录使用的 asari 版本和参数

这个教程应该能帮助您开始使用 asari 进行代谢组学数据分析。如果有具体问题，欢迎询问！