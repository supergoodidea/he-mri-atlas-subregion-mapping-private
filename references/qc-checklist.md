# QC 与交付检查清单

## 图谱与 HE

- [ ] 图谱页码/section 编号可追溯。
- [ ] 每个 subregion 都能追溯到图谱小结构或明确的空间依据。
- [ ] HE 灰度背景、GM mask、label overlay 同时可见。
- [ ] 主要脑叶分界线有名称，边界不清处有标记。
- [ ] 没有把白质束、深部核团或图谱注释线误当作皮质 subregion。

## MRI 匹配

- [ ] qform/sform、spacing 和 affine 已检查。
- [ ] 方向变换可复现，并保留原始 MRI 不变。
- [ ] 每个 HE slice 有最佳 MRI 层和相邻 stack 层索引。
- [ ] 显示最佳候选与次佳候选，低分或并列结果需人工确认。
- [ ] stack label 与 HE label 使用最近邻，不产生非整数标签。

## 数值与文件

- [ ] label 值属于定义表，背景为 0。
- [ ] 输出 NIfTI 可以重新读取，空间信息与 manifest 一致。
- [ ] PVS 体积按实际物理 voxel volume 计算。
- [ ] 原始文件 sha256 未改变。
- [ ] CSV、NIfTI、QC 图和 README 互相引用且版本号一致。
- [ ] 明确列出未匹配 slice、未分区区域、模糊边界和超阈值距离。
