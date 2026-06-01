# 图像加密超表面 FDTD 数据库设计说明

本文档整理当前矩形 Si3N4 超表面单元样例中已经讨论清楚的关键设置，作为后续建立 PINN 替代电磁仿真数据库的设计依据。

## 1. 当前样例的物理模型

当前样例表示的是周期性超表面单元，而不是孤立散射体。

基本结构为：

```text
空气侧入射
Si3N4 纳米结构
SiO2 基底
```

当前样例参数：

```text
结构材料：Si3N4
基底材料：SiO2
结构高度：800 nm
周期：300 nm
示例结构：矩形柱
示例尺寸：120 nm x 200 nm
示例旋转角：0 deg
波长范围：400-700 nm
波长步长：3 nm
```

## 2. 边界条件设置

周期性单元推荐使用：

```text
x/y 方向：Periodic
z 方向：PML
```

这表示无限周期阵列中的一个单元。

如果把 x/y/z 都设置为 PML，仿真的对象会变成孤立纳米结构散射体，不再是周期超表面单元。

注意：Lumerical GPU FDTD solver 不支持 Periodic 边界。因此当前周期单元样例应使用 CPU solver 运行。

## 3. 光源方向

当前推荐设置：

```text
光源在结构上方
传播方向沿 -z
空气侧入射到 Si3N4/SiO2 结构
```

不要为了让透射率显示为正而随意把光源移动到基底下方。因为：

```text
空气侧入射：空气 -> Si3N4 -> SiO2
基底侧入射：SiO2 -> Si3N4 -> 空气
```

这两个问题通常不等价，尤其结构上下介质不对称时，反射、透射和相位都会不同。

后续如果需要研究正向和反向入射，应把入射侧作为数据库输入参数分别建模，不能混用。

## 4. 透射率为负的问题

FDTD 中 `transmission("T")` 的符号由 monitor 法向和实际功率流方向共同决定。

当前光从上往下传播，透射光沿 -z 穿过下方 T monitor。如果 T monitor 的正法向为 +z，则 FDTD 中直接查看到的透射率可能为负。

这不是物理透射率为负，也不表示仿真错误。

后处理时应保存：

```text
T_raw = transmission("T")
T_spectrum = -T_raw
```

也建议同时保存原始符号数据，便于追溯：

```text
R_raw
T_raw
R_spectrum
T_spectrum
```

## 5. 监视器位置

当前结构坐标可理解为：

```text
Si3N4 柱：z = 0 到 800 nm
SiO2 基底：z = -1000 nm 到 0
空气区域：z > 800 nm
```

推荐监视器顺序：

```text
光源
R monitor
Si3N4 结构
T monitor
PML
```

### 5.1 反射监视器

反射监视器应放在光源和结构之间，但不要和光源重合。

当前样例中：

```text
R monitor: z = 1050 nm
source:    z = 1350 nm
```

这个位置关系是合理的。

### 5.2 透射监视器

如果数据库描述的是：

```text
空气 -> Si3N4 超表面 -> SiO2 基底
```

那么 T monitor 应放在 SiO2 基底内部，而不是基底下方空气中。

原因是放在基底下方会额外引入：

```text
SiO2/空气 背面界面透射
基底内传播相位
背面反射
Fabry-Perot 干涉
```

这些会污染单元本身的响应。

当前样例中：

```text
T monitor: z = -650 nm
SiO2 基底：z = -1000 nm 到 0
```

这个设置适合提取“超表面透射进入 SiO2 后”的单元响应。

如果真实器件是有限厚 SiO2 基底并最终从背面出射到空气，应分两层处理：

```text
单元数据库：T monitor 放在 SiO2 内部，提取单元响应
系统验证：额外加入有限基底传播和背面出射界面
```

### 5.3 近场监视器

近场监视器用于保存结构附近的 E/H 场。

当前样例保存了结构附近三维近场，适合观察和训练局部场分布。

如果后续要做整片超表面近场拼接，建议额外固定一个出射面近场监视器，例如：

```text
z = -100 nm 或 z = -200 nm
```

这样每个单元的出射复场可以在统一参考面上拼接。

## 6. 相位定义

当前样例中没有单独的“相位监视器”。相位来自复电场：

```text
phase_T = angle(Epol_T)
phase_R = angle(Epol_R)
```

相位会随监视器 z 位置变化，因为传播会引入：

```text
phase(z) = phase0 + k z
```

因此数据库中相位必须在固定参考面上定义。

“全相位覆盖”的意义不是绝对相位天然唯一，而是：

```text
在同一个参考面、同一个入射方向、同一个相位定义下，
不同结构产生的相对相位能够覆盖 0 到 2π。
```

更严谨的相位定义应扣除参考传播相位：

```text
phase_corrected = angle(t_with_meta) - angle(t_reference)
```

其中 `t_reference` 可由参考仿真得到：

```text
只有 SiO2 基底
没有 Si3N4 纳米结构
```

正式数据库建议保存：

```text
phase_raw
phase_unwrapped
phase_corrected
```

## 7. 图像加密任务对数据库的要求

图像加密超表面通常需要的不只是普通透射率和反射率，而是复振幅响应：

```text
幅度
相位
偏振转换
```

因此正式数据库应优先提取 Jones 矩阵。

线偏振基下：

```text
t_xx, t_xy
t_yx, t_yy
```

含义为：

```text
Ex 入射 -> Ex/Ey 出射
Ey 入射 -> Ex/Ey 出射
```

如果要做圆偏振图像加密，还应转换到圆偏振基：

```text
t_RR, t_RL
t_LR, t_LL
```

这样数据库才能支持：

```text
线偏振加密
圆偏振加密
偏振复用图像
波长复用图像
相位型全息加密
```

## 8. 推荐的数据库输入参数

PINN 数据库输入建议包含：

```text
shape_type
geometry_params
height
period
rotation_angle
wavelength
incident_polarization
incident_side
pillar_material
substrate_material
boundary_condition
monitor_reference_plane
```

对于不同结构，几何参数可以定义为：

```text
矩形：Lx, Ly, theta
圆形：radius
平行四边形：Lx, Ly, shear_angle, theta
H 形：bar_width, gap, arm_length, theta
十字形：arm_length_x, arm_length_y, arm_width_x, arm_width_y, theta
```

圆形结构旋转后几何不变，因此旋转角通常不是有效自由度。

## 9. 推荐的数据库输出参数

基础输出：

```text
R_spectrum
T_spectrum
phase_R
phase_T
```

图像加密推荐输出：

```text
complex Jones matrix
co-polarized transmission
cross-polarized transmission
co-polarized reflection
cross-polarized reflection
co-polarized phase
cross-polarized phase
```

可选重数据：

```text
E_near
H_near
E_transmission_plane
H_transmission_plane
```

## 10. 建议的后续开发顺序

不要马上扩展所有形状。建议先把当前矩形样例升级为 Jones 矩阵提取样例。

推荐顺序：

```text
1. 当前单个矩形样例跑通
2. 修正导出脚本，保存 R_raw/T_raw 和物理 R/T
3. 增加参考仿真，扣除传播相位
4. 增加 Ex 入射和 Ey 入射，提取 Jones 矩阵
5. 增加旋转角 sweep
6. 增加矩形尺寸 sweep
7. 扩展到 H 形、十字形、平行四边形
8. 增加圆偏振基响应
9. 最后扩展到 400-700 nm 全波段数据库
```

对于图像加密早期验证，可以先选择少量代表波长：

```text
450 nm
532 nm
633 nm
```

确认相位覆盖、偏振转换和图像重建流程后，再扩展到 400:3:700 nm 的宽带数据库。

## 11. 当前矩形样例的问题和限制

当前矩形样例已经完成了第一阶段目标：

```text
可以建模
可以运行 FDTD
可以导出 R/T/phase/E/H
可以作为后续数据库脚本模板
```

但它仍然只是一个起步样例，不应直接作为正式数据库的最终格式。

### 11.1 R/T 提取方式仍是临时检查方法

当前导出脚本使用普通 power monitor：

```lsf
T_raw = transmission("T");
R_raw = transmission("R");

T_spectrum = -T_raw;
R_spectrum = 1 + R_raw;
```

这个公式适用于当前布局：

```text
光源在上方
R monitor 位于光源和结构之间
T monitor 位于结构下方的 SiO2 基底中
光沿 -z 方向传播
```

其中：

```text
T_spectrum = -T_raw
```

是因为透射光沿 -z 方向穿过下方 T monitor。

```text
R_spectrum = 1 + R_raw
```

是因为 R monitor 测到的是入射光和反射光的净功率：

```text
R_raw = -incident + reflected = -1 + R
```

因此：

```text
R = 1 + R_raw
```

这种方法适合快速检查仿真是否正常，但不是周期性超表面单元库的最佳最终标签。

### 11.2 正式数据库建议改用 S-parameter / grating 提取

官方 metalens 案例的单元仿真不是直接用普通 power monitor 的 `transmission()` 作为最终结果，而是用 S 参数和光栅级次结果：

```lsf
S = getsweepresult(sname,"S");
T = getsweepresult(sname,"T");

phase = angle(S.S21_Gn);
T0 = T.T_Gn;
```

也就是：

```text
相位来自零级复透射系数 S21
透射率来自零级光栅级次功率 T_Gn
```

这种方式更适合周期单元，因为它可以区分：

```text
零级透射
高阶衍射透射
总透射
```

当前样例的普通 monitor 结果可用于 sanity check，但后续正式数据库应增加：

```text
S-parameter analysis group
grating projection / grating order extraction
```

并优先保存：

```text
S21
S11
phase_S21
phase_S11
T_Gn
R_Gn
```

### 11.3 当前相位定义依赖监视器位置

当前导出的：

```text
phase_T
phase_R
```

来自监视器中心点复电场：

```lsf
phase_T = angle(Epol_T);
phase_R = angle(Epol_R);
```

这个相位会随 monitor 的 z 位置变化，因此只适合当前样例内部比较。

正式数据库应使用统一参考面，并建议保存：

```text
phase_raw
phase_unwrapped
phase_corrected
```

更好的做法是用：

```text
phase_S21 = angle(S21)
```

或扣除参考结构传播相位：

```text
phase_corrected = angle(t_with_meta) - angle(t_reference)
```

### 11.4 当前 .mat 文件过大

当前完整导出的单个样本接近 1 GB，主要原因是保存了完整三维近场：

```text
E_near
H_near
```

当前样例中：

```text
E_near 约 485 MB
H_near 约 485 MB
```

而光谱和几何参数本身只占很小空间。

批量建库时不应每个样本都保存完整三维场。

建议拆分为两个导出版本：

```text
轻量版：只保存几何参数、波长、R/T、phase、原始 R/T
完整版：额外保存 E_T、E_R、E_near、H_near
```

轻量版建议保存：

```text
shape_type
period
pillar_height
geometry_params
theta_deg
pol_angle_deg
wavelength
R_raw
T_raw
R_spectrum
T_spectrum
phase_R
phase_T
```

完整版才保存：

```text
E_T
E_R
E_near
H_near
```

### 11.5 当前 R+T 在少数短波点有偏差

当前修正后的 `.mat` 中，绝大多数波长点的能量检查是合理的：

```text
R_spectrum + T_spectrum 平均约为 0.993
```

但少数短波点偏离较明显，例如：

```text
403 nm: R+T 约 0.937
418 nm: R+T 约 0.703
430 nm: R+T 约 0.639
```

这可能与以下因素有关：

```text
周期 300 nm 在短波端接近或进入衍射条件
普通 power monitor 不能区分不同光栅级次
网格精度或仿真收敛不足
监视器位置和近场/衍射场混合
```

这也是后续应改用 S-parameter / grating 提取的原因之一。

### 11.6 当前材料模型只是常数折射率近似

当前脚本默认：

```text
Si3N4: n = 2.0
SiO2:  n = 1.45
```

这适合跑通流程，但不适合作为最终物理数据库。

正式数据库应替换为：

```text
Lumerical 材料库中的色散材料
或实验/文献 n,k 数据
```

### 11.7 当前 GPU solver 不能用于周期边界

当前周期单元边界条件为：

```text
x/y: Periodic
z: PML
```

Lumerical GPU FDTD solver 不支持 Periodic 边界，因此当前样例应使用 CPU solver。

如果强行改成全 PML 虽然可能可以使用 GPU，但物理模型会变成孤立散射体，不再是周期超表面单元。
