# 矩形 Si3N4 超表面单元 FDTD 样例

这个文件夹包含一个用于 PINN 仿真数据库起步的 Lumerical FDTD 单元结构样例。

## 几何结构

- 结构类型：矩形 Si3N4 纳米柱
- 基底材料：SiO2
- FDTD 内部 SiO2 厚度：500 nm
- 周期：x/y 方向均为 300 nm
- 纳米柱高度：800 nm
- 示例矩形尺寸：120 nm x 200 nm
- 示例旋转角：0 deg
- FDTD z 方向下边界：位于 SiO2 内部，SiO2 基底向下穿过 PML
- 反射监视器位置：距离 Si3N4 顶部 400 nm，大于 300 nm

## 光源设置

- 光源类型：平面波
- 入射方式：法向入射
- 偏振：线偏振
- 波长范围：400-700 nm
- 波长采样点数：101 点，对应 3 nm 间隔

## 边界条件

- x/y 方向：Periodic
- z 方向：PML

这是周期性超表面单元仿真的推荐设置。如果把 x/y/z 都设置为 PML，仿真的将是孤立散射结构，而不是周期阵列单元。

注意：Lumerical 的 GPU FDTD solver 不支持 Periodic 边界。如果使用本样例的周期边界，请使用 CPU solver 运行。

## 文件说明

- `rect_sin_unit_cell_setup.lsf`
  - 自动建立 FDTD 工程，并保存为 `rect_sin_unit_cell.fsp`。
- `rect_sin_unit_cell_export.lsf`
  - 在仿真完成后导出反射谱、透射谱、相位和电磁场数据，保存为 `rect_sin_unit_cell_results.mat`。

## 使用方法

1. 打开 Lumerical FDTD。
2. 运行 `rect_sin_unit_cell_setup.lsf`，生成 `rect_sin_unit_cell.fsp`。
3. 打开或继续使用生成的 `rect_sin_unit_cell.fsp`。
4. 确认资源配置使用 CPU solver，而不是 GPU solver。
5. 运行仿真。
6. 仿真结束后运行 `rect_sin_unit_cell_export.lsf`。

导出的 `.mat` 文件可以作为 PINN 仿真数据库的第一条样本模板。

## 当前导出数据

`rect_sin_unit_cell_export.lsf` 会导出：

- 结构类型
- 材料信息
- 边界条件
- 周期
- 纳米柱高度
- 矩形尺寸
- 旋转角
- 线偏振角
- 波长数组
- 反射谱 `R_spectrum`
- 透射谱 `T_spectrum`
- 反射相位 `phase_R`
- 透射相位 `phase_T`
- 透射面电场 `E_T`
- 反射面电场 `E_R`
- 近场电场 `E_near`
- 近场磁场 `H_near`

## 材料说明

当前脚本默认使用常数折射率材料：

- Si3N4：n = 2.0
- SiO2：n = 1.45

这个设置适合先跑通建模、仿真和数据导出流程。正式生成数据库时，建议替换为 Lumerical 材料库中的色散材料，或者使用实验/文献中的 n,k 数据。
