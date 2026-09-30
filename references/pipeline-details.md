# 管线细节与数据字段

## 推荐 manifest 字段

### 输入记录

```text
subject
path
role
space
shape
spacing_mm
qform_code
sform_code
affine
sha256
```

`space` 至少区分 `HE`, `HE_coarse`, `MRI_original`, `MRI_stack`, `atlas_plate`。

### HE–MRI stack 记录

```text
subject
he_slice
he_array_index
best_mri_plane
best_score
orientation_code
inplane_transform
stack_indices
stack_to_original_mri_voxel
status
notes
```

`orientation_code` 应是可读字符串，例如 `swap_xy|flip_x|rot90`，而不是只保存一个无法解释的整数。

## 坐标与插值

对于 NIfTI voxel 坐标 `v`，用 affine 得到物理坐标：

```text
x_world = affine @ [v_x, v_y, v_z, 1]
```

如果将 label 从 source 网格重采样到 target 网格，目标 voxel 中心先通过 `target_affine^-1` 转到 source voxel 坐标，再用最近邻取值。不得用图像数组行列的直觉替代 affine。

对于每个 PVS 或小结构 label，优先使用其所有 voxel 到候选 subregion voxel 的最小物理距离；若计算量过大，可用质心作为初筛，再对候选区域做精确距离计算。

## 匹配得分建议

搜索 MRI 冠状面时，可将以下分量加权组合，权重必须写入配置或结果：

- GM mask 的 Dice/IoU；
- 脑组织外轮廓相似度；
- 灰白质边界或皮质带强度轮廓；
- 主要沟回/裂隙的边缘或距离变换相似度；
- MRI 强度相关性，仅作为辅助。

匹配分数不是解剖学真值。QC 必须展示最佳层、次佳层和差异，尤其是分数差很小时。

## 标签约定

- `0`：背景/未标记；
- 父脑叶标签与 subregion 标签不能在未说明的情况下混用；
- `candidate`、`uncertain`、`unsegmented` 要保留在名称或状态字段中；
- label dictionary 是唯一的标签解释来源，所有 CSV 和 QC 使用同一套名称。

## PVS 统计字段

建议的 component-level 表：

```text
subject
space
slice_or_stack
component_id
subregion_label
voxel_count
volume_mm3
centroid_x_mm
centroid_y_mm
centroid_z_mm
nearest_distance_mm
assignment_status
```

相关性分析应注明配对单位：跨 HE slice 的总量、中心层面积或 component-level 量。若有效非零配对少于预设阈值，输出 `insufficient_positive_pairs`，不要报告看似完美的相关系数。
