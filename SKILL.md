---
name: he-mri-atlas-subregion-mapping
description: "Build and run an atlas-guided HE–MRI cortical subregion mapping pipeline: learn atlas structure-to-lobe mappings, delineate anatomy-based HE boundaries, correct MRI orientation, search for the most similar MRI plane, create an MRI stack matched to HE, propagate edited HE subregion labels, and produce NIfTI/CSV/QC outputs. Use for ex-vivo HE/T2 alignment and cortical subregion labeling; do not use for unrelated generic image registration or Kluver-only workflows."
---

# HE–MRI 图谱驱动的小脑区映射

## 目标

把图谱、HE 组织学切片和 3D MRI 组织成一条可审计的处理管线：

1. 从图谱和分区表学习“解剖小结构 → 脑叶/小脑区”的映射。
2. 在 HE 灰质中根据沟回、脑回、脑沟和深部结构的空间关系确定脑叶边界与 subregion。
3. 读取 NIfTI 的方向编码和空间 affine，校正 MRI 的旋转/翻转/平移候选。
4. 从 3D MRI 中搜索与每一张 HE 最相似的冠状面，并构建前后相邻的 MRI stack。
5. 将人工修正后的 HE subregion 以最近邻方式映射到 MRI stack。
6. 输出原始空间、stack 空间、标签、组件统计和逐层 QC，并明确报告不确定性。

## 何时使用

适用于：

- Weigert/LFB/HE 等离体组织图谱与 HE 3D 堆叠的脑叶或皮质小区标记；
- HE slice 与 3D T2/T2w MRI 的方向校正、冠状面搜索和 stack 匹配；
- 用户已经编辑粗分区或 subregion label，需要把修正结果传播到原始分辨率或 MRI；
- 需要通过 QC 图逐层确认脑区边界、MRI–HE 匹配和标签映射。

不适用于：只做普通配准而不涉及图谱/解剖分区的任务，也不应把本 skill 当作 Kluver 专用配准范式。

## 执行原则

- 先检查数据结构、NIfTI header、qform/sform、spacing、shape、文件命名和现有脚本，再决定处理顺序。
- 原始数据只读；所有结果写入新建的、带版本号的输出目录。不要覆盖原始 HE、MRI、分割或用户编辑文件。
- 对连续图像可用线性/三次插值；对二值 mask、脑叶、subregion、PVS component ID 等离散标签必须使用最近邻插值。
- 所有“匹配”“映射”“最近距离”都必须在明确的物理坐标或明确的 voxel 坐标转换后进行，不可仅凭数组索引猜测。
- 如果方向、切片号、脑区边界或相似 MRI 层面存在多解，保留候选和理由，不要静默选择并声称确定。

## 标准输入合同

优先解析用户实际提供的路径；若缺少路径，先列出需要的输入而不是猜路径：

- 图谱：PDF、图谱切片图片或已提取的 atlas plates；
- 结构映射表：XLSX/CSV/JSON，至少包含原始小结构名称、缩写、脑叶或 subregion label；
- HE：2D 或 3D NIfTI/TIFF，最好有灰质/白质 mask；
- HE 分区：原始脑叶分割、subregion 分割或用户编辑后的粗分区；
- MRI：3D T2/T2w NIfTI，必要时提供 MRI 的 mask 或 GM segmentation；
- 可选：slice 编号表、手工翻转记录、边界 label 定义、已有 HE–MRI matching 结果。

对每个输入记录绝对路径、shape、spacing、dtype、qform/sform、affine、sha256 和读取方式。

## 管线

### A. 学习图谱与建立映射

1. 从 PDF 或图谱图片提取目标页，并保留原始 PDF 编号与图谱图像编号的对应关系。
2. 识别图谱上的小结构缩写、脑回/脑沟、皮质带和深部白质结构；区分“结构标签”和“边界/纤维/束标签”。
3. 读取用户提供的 Excel 修正版，优先使用用户明确修正过的映射；不得用旧版映射覆盖新版。
4. 建立可追溯的 label dictionary：`label_id`、`code`、中文名、父脑叶、图谱参照结构、定义、置信度、备注。
5. 对“候选区”“待确认”“未细分”保留候选性质，不将其改写为确定的细胞构筑分区。

### B. HE 解剖识别与边界分区

在 HE 灰质 mask 约束内工作。识别顺序应优先使用稳定的空间关系：

1. 先确定切片方位、背腹/内外/前后方向和大脑半球范围。
2. 再利用中央沟、外侧裂、顶枕沟、距状沟、扣带沟、颞上沟、颞下沟、岛盖/岛叶边界等主要沟裂作为脑叶分界候选。
3. 用相邻脑回的连续性、皮质外缘形态、沟底走向、白质扇形结构和深部结构位置判断小区，而不是只按图像颜色或最近点分类。
4. 对每条边界保存：名称、两侧 label、图谱依据、HE 依据、是否完整可见、置信度和需要人工复核的范围。
5. 没有可靠解剖依据时，保持父脑叶或 `未细分/待确认`，不要强行细分。

分区输出须满足：同一 voxel 只有一个离散 label；label 0 表示背景或未标记；非零标签应落在灰质或用户明确允许的皮质范围内。

### C. MRI 方向校正与相似层搜索

1. 读取 MRI 的 qform/sform 和 affine；同时记录数组轴与物理轴的对应关系，不要默认 `axis 2` 就是冠状面。
2. 生成有限的方向候选：轴置换、左右翻转、上下翻转、前后翻转以及必要的 90/180 度面内旋转。候选必须记录为可复现的变换，而不是直接修改原始文件。
3. 为提高搜索速度，可对 MRI 降采样或裁剪到脑组织 bounding box；最终映射必须回到原始 MRI 网格或明确的 stack 网格。
4. 以 HE 的 GM mask、外轮廓、灰白质比例、主要沟回形态和强度轮廓作为匹配依据。原始强度只作为辅助，不能替代解剖结构匹配。
5. 对每个 HE slice 在候选 MRI 冠状面中计算相似度并保留前若干候选；检查匹配是否跨越多个 MRI 层面、是否存在 slice 编号错配或空层面。
6. 选择最相似层面作为中心，再在物理距离或固定层数范围内建立 MRI stack；将 `HE slice → MRI center plane → MRI stack planes → original MRI indices` 写入 manifest。
7. 若多方向或多层面得分接近，输出候选排序和 QC，而不是静默定论。

### D. HE subregion 修正传播到 MRI

当用户在降采样 HE subregion 文件中用正确 label 覆盖错误区域时：

1. 先把编辑文件最近邻插值回原始 HE 网格；保留原始 HE subregion 作为对照。
2. 仅在 HE GM mask 内应用编辑 label；编辑文件之外的区域不自动改变。
3. 根据已确认的 HE–MRI stack 变换，将 HE subregion 映射到对应 MRI stack。若 stack 每层对应原始 MRI 体素，优先使用显式 `stack_to_original_T2_voxel` 映射。
4. MRI 上的离散 label 统一使用最近邻，不使用线性插值产生伪标签。
5. 每个 MRI stack 输出：GM/subregion label NIfTI、HE 对照 label、slice 映射表和标签计数。

### E. PVS 或其他独立目标的脑区分配（可选）

如果任务包含 PVS：

1. 将二值 PVS 进行 2D/3D 连通域标记，明确连通性规则、体素 spacing 和是否跨层连接。
2. 每个独立 PVS 通过其 voxel 集合或质心到非零 subregion 的物理距离分配标签；记录最近距离、候选 label 和是否超出距离阈值。
3. 若用户要求“补齐连通区域”，只在明确的二值 PVS mask 内进行受限填充，并保留填充前后体素数和组件数。
4. 体积使用实际物理体素体积 `abs(det(affine[:3,:3]))` 或 header zooms 的乘积，不把 voxel count 当作 mm³。
5. 统计时明确区分：原始 MRI 空间、MRI stack 空间、HE 空间、中心层面积或 3D 体积。不要混用这些量。

## QC 最低要求

至少生成以下 QC：

- 图谱页与映射表的结构—subregion 对照；
- HE 原图、GM mask、脑叶/subregion overlay，标出边界名称；
- MRI 原始/旋转后/最佳匹配层面的并排或叠加图；
- 每个 HE slice 对应的 MRI stack 中心层和前后层；
- 用户编辑前后 subregion 差异图；
- MRI stack label 与 HE label 的逐层 QC；
- 若有 PVS，显示 PVS component、subregion label 和距离分配结果；
- 所有异常：无匹配层、方向不确定、边界不可见、未分区 GM、距离过大、标签超出定义集合。

QC 图应标注 subject、HE slice、MRI index/stack index、方向变换、label 名称和图谱依据。不要只输出彩色标签而不提供原始灰度或结构背景。

## 输出组织

建议每次运行使用版本化目录，例如：

```text
<output_root>/
├── inputs_manifest.json
├── label_definitions.json
├── atlas_mapping.csv
├── he_to_mri_stack_manifest.csv
├── HE_space/
├── MRI_space/
├── MRI_stack/
├── tables/
├── qc/
├── validation.json
└── README_先读.md
```

每个 NIfTI 输出应说明：空间、shape、spacing、affine、插值方法、来源文件和生成脚本。结果文件名要体现空间和版本，例如 `*_HEspace_v2.nii.gz`、`*_MRIstack_v1.nii.gz`。

## 验证与停止条件

完成前至少检查：

- 原始输入 sha256 未改变；
- 输出 NIfTI 能被重新读取，shape/spacing/affine 符合预期；
- 所有离散输出标签均属于 label dictionary，背景为 0；
- HE/MRI 的映射方向与 manifest 一致；
- MRI stack 的中心层和相邻层有明确原始 MRI 索引；
- 对空层、低相似度、边界不清或距离超阈值对象给出报告；
- 体积、面积、组件数和标签计数经过独立数值检查。

遇到以下情况应停止并报告，而不是继续自动细分：NIfTI 方向信息相互矛盾；所有 MRI 候选相似度都低；HE slice 与 MRI 的组织范围明显不同；用户编辑 label 无法映射到定义表；边界在当前切片不可见；或多个方向/层面无法区分。

## 参考资料

- 具体坐标变换、MRI stack manifest、标签插值和验证字段见 [references/pipeline-details.md](references/pipeline-details.md)。
- QC 图和交付物检查清单见 [references/qc-checklist.md](references/qc-checklist.md)。
