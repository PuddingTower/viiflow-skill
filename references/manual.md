# VIIFLOW viiflow 说明书（精炼全文）

来源：官方 `https://viiflow.com/wp-content/uploads/Notebooks/Installation.html` 与
`viiflow-1.9.4` wheel METADATA（2026-10-10 抓取）。本文件为完整参考，SKILL.md 是速查。

## 模块安装

- 64 位 Cython wheel（Windows/Linux）。Python **3.12.x**、numpy **2.4.x**、scipy。
- `pip install viiflow-[...].whl` / `pip install viiflowtools-[...].whl`。
- numpy/scipy 可用 https://github.com/urob/numpy-mkl 的 MKL 版。
- 例子用到 jupyter/其它包，但非必需。

## 重要口径

- **不缩放**：几何不归一到弦长 1。mm 坐标则翼型长 1000；系数按长度 1 算（CL 会到千级）；Re 也指坐标系的 1。多段翼下刻意为之。
- **不检查几何**：必须后缘上表面→后缘下表面的次序；最后两个节点须准确表示后缘切线方向，钝后缘不要底↔顶封口。
- 折角（襟翼舱）用（负）虚位移填平，面板体尽量贴近最终位移体，更准且更易收敛。
- **力矩系数参考坐标系原点**；要绕 1/4 弦就移几何使目标点在原点，或设 `CmRefX=.25`。

## setup 全参数表

| Type | Name | Explanation | Default |
|---|---|---|---|
| double | Re | 雷诺数 | 1E6 |
| double | Ma | 马赫数(<1) | 0 |
| double | Ncrit | 临界放大因子 | 9 |
| int | Itermax | Newton 最大迭代 | 100 |
| bool | IterateWakes | 迭代中重算尾迹形状 | 0 |
| int | Substeps | 两面板节点间步数 | 1 |
| bool | Silent | 不打印迭代信息 | 0 |
| bool | Gradients | 计算梯度 | 0 |
| double | Alpha | 迎角 | 0 |
| double | PitchRate | 绕(0,0)俯仰率 | 0 |
| double | LocusA | G-β 常数 A | 6.75 |
| double | LocusB | G-β 常数 B | 0.83 |
| double | ViscPwrExp | 粘温模型指数 | 0.7 |
| double | WakeLength | 尾迹长度/弦 | 1 |
| double | Tolerance | 收敛判据 | 1e-4 |
| int | IncompressibleBL | >0 时按 Ma=0 算边界层 | 0 |
| double | StepsizeLimit | Newton 步长上限(0..1) | 1 |
| double | ShearLagLambdaWake | 滞后方程 λ(尾迹) | .9 |
| double | ShearLagLambdaFoil | 滞后方程 λ(翼面) | 1 |
| bool | HalfWakes | 尾迹用两条独立边界层 | 0 |
| int | ShearLagType | K_C 函数选择 | 0 |
| double | TransitionStepLimit | 限制转捩移动 | inf |
| bool | IsCascade | 叶栅 | 0 |
| double | CascadeStaggerHeight | 叶栅间距 | 1 |
| double | CascadeStaggerAngle | 叶栅稠度角 | 0 |
| bool | BypassTransition | 旁路转捩 [14] | 1 |
| bool | BypassA | 参数 A [14] | .1 |
| bool | BypassB | 参数 B [14] | .3 |
| int | FreezeWake | 变 α 时冻结尾迹 | 0 |
| int | Mirror | 0无/1地面效应/-1自由面 | 0 |
| double | IntermittencyGrowthRate | 间歇因子增长率 | 10 |
| double | CLTarget | 给定 CL 反求 α | inf |
| int | FreeSurface | 自由面 | 0 |
| double | FroudeNumber | 傅汝德数 | 1.0 |
| double | FreeSurfMultLeft | 自由面可视化长度(迎风) | 4.0 |
| double | FreeSurfMultRight | 自由面可视化长度(背风) | 6.0 |
| double | CmRefX | 力矩参考点 X | 0.0 |
| double | CmRefY | 力矩参考点 Y | 0.0 |

### 参数注解

- **Substeps (*1)**：默认 ODE 步长=面板节点间距；设 n 把节点间再分 n 段（增大推进成本，不增大全局 Newton）。
- **PitchRate (*2)**：物理俯仰率[rad/s]/来流速度。40°/s、20m/s → `40*pi/180/20`。绕(0,0)，几何需相应摆放。也描述旋转翼 c/R，设 `c/R`。
- **G-β Locus (*3)**：XFOIL (A=6.7,B=.75)、Boeing (6.935,.70)、Green (6.43,.80)、RFOIL/viiflow 默认 (6.75,.83)。
- **ShearLagType (*4)**：0 常数 5.6；1 RFOIL `Kc=4.65-.95*tanh(.275H-3.5)`（大 H 剪切应力更慢到平衡）；2 Ye/Thomas `Kc=3.75*(3H)/(H+2)`（近 XFOIL）。失速附近影响显著。
- **TransitionStepLimit (*5)**：限制单个 Newton 步内转捩移动；不改收敛解，仅助收敛。
- **Cascade (*6)**：无限叶栅叠于竖直轴；StaggerAngle=翼型轴与竖直轴间的角（0=上下对齐），StaggerHeight=沿叠轴间距。
- **FreezeWake (*7)**：用于自定义找 AOA（非内置 α 搜索）时。
- **Mirror (*9)**：沿过 0 且沿来流方向的射线镜像几何。1=地面效应（流平行地面，自由面高 Fr 极限）；-1=在 y=0 强制 Cp=1（自由面低 Fr 极限）。
- **CLTarget (*10)**：把 α 作为额外未知量并入 iter，`x` 长度 +1；`res/grad` 维度不变（仍按常 α）。梯度里可取 α 灵敏度。
- **FreeSurface (*11)**：沿过 0 射线做自由面（闭式积分）；Fr 为 setup 参数；`p.SurfaceVel`/`p.SurfaceNodes` 查看；节点由 FreeSurfMultLeft/Right（×几何 x 总跨度）定义。

## 四个（+空化）方法

- `setup` → 参数结构 s
- `init` → 数据 p、bl、x（含面板算子、初始化边界层）
- `iter` → Newton 迭代到收敛（1.9.4 返回 `(x,flag,res,grad)`）
- `set_forced_transition` → 强制转捩
- `inviscid_velocity` / `viscid_profile` / `viscous_flowfield` → 流场/可视化
- `cav_init` / `cav_solve` → 空化

## 边界层对象 bl

```
bl[0].S / .Sw / .Re / .Ma / .ncrit / .hte / .ST / .sti / .N / .NW / .substeps
bl[0].CD
bl[0].S_trans_forced_up / .S_trans_forced_lo
bl[0].bl_fl.nodes        # 翼面 BL 节点 (recarray)
bl[0].bl_fl.node_tr_up   # 转捩处额外节点
bl[0].bl_fl.node_tr_lo
bl[0].bl_wk.nodes        # 尾迹 BL 节点
```

节点字段（XFOIL 命名）：
```
node.typ    # 1层流 2湍流 3尾迹
node.delta  # 位移厚度
node.theta  # 动量厚度
node.ue     # 边缘速度
node.nct    # 层流:放大因子; 湍流/尾迹:剪切应力系数
node.xi     # 面坐标(从驻点起)
node.UE     # Ma 修正边缘速度
node.H      # 形状因子 delta/theta
node.HK     # 运动学形状参数
node.HS     # 动能形状参数 H*
node.HC     # 密度形状参数 H**
node.RT     # 动量厚度 Re
node.MSQ    # 局部 Ma
node.CF     # 摩阻系数
node.DI     # 耗散系数
node.NX     # 放大因子局部增量
node.DE     # 边界层厚度
node.US     # 滑移速度
node.CQ     # 平衡剪切应力系数
node.xDt    # xi/theta
node.delta_wake
```

## 无粘对象 p

```
p.NS p.NW p.alpha p.pitch_rate p.IsCascade p.CascadeStaggerHeight p.CascadeStaggerAngle
p.mach p.CL p.CLi p.CM
p.CascadeDeflectionAngle p.CascadeDownstreamVelocity
p.CascadeDeflectionAngleInviscid p.CascadeDownstreamVelocityInviscid
p.gamma_inviscid_0 p.gamma_inviscid_90 p.gamma_inviscid_pitch p.gamma_inviscid
p.cp p.cp_inviscid p.gamma_viscid p.max_thickness p.min_thickness
p.foils[0].X/.S/.normals/.N/.sharp/.VD/.delta/.hte
p.wakes[0].X/.S/.normals/.N/.delta
```

## 梯度（Pro）

partial（迭代中偏导）：
```
partial.res_vd (NX,NVD)   partial.res_aoa (NX,1)
partial.gam_x (N,NX)      partial.gam_aoa (N,1)     partial.gam_vd (N,NVD)
partial.cl_x (1,NX)       partial.cd_x (NF,NX)      partial.cm_x (1,NX)
partial.cm_aoa (1,1)      partial.cl_vd (1,NVD)     partial.cm_vd (1,NVD)
```
total（收敛解灵敏度）：
```
total.gam_aoa (N,1)  total.cl_aoa (1,1)  total.cm_aoa (1,1)  total.cd_aoa (NF,1)
total.delta_vd (NX,NVD)  total.gam_vd (NX,NVD)
total.cl_vd (1,NVD)  total.cm_vd (1,NVD)  total.cd_vd (NF,NVD)
```
`_vd` 对虚位移，`_aoa` 对迎角（**rad**），`x` 对 viiflow 变量。

## 空化

- `problem = vf.cav_init(AF, s)`；`flag = vf.cav_solve(problem, problem.s, sig, nsubiter, callback, tol, restart, cavWeight)`。
- 不能与 CLTarget 同用（α 固定）。
- `problem` 字段：`p,bl,x,flag`（含镜像空泡的虚位移）、`vd`（空泡虚位移）、`sigma`、`cavdrag`（附加压差阻力，**非**全部空化阻力；主阻力来自空泡后的分离）、`CL/CD/CM/CP_noncav`、`itr`（CavResidual/Step/NormalResidual）。
- `nsubiter` 最大迭代；`tol` 默认 1e-4；`callback` 每内部迭代回调（可 None）；`restart=True` 先解无空化；`cavWeight` 默认 1，用于平衡 `cp>cp_cav` 与 `cp=cp_cav`（取值 .1~20）。

## 版本/许可

- Basic：不含多段几何、不含梯度计算。Pro：含。
- Changelog 要点：1.9.4 增空化求解 + 全隐边界层；1.9 增地面效应/反镜像/自由面、间歇模型、CLTarget；1.7 复数变量面板法 + 叶栅；1.6 增 ShearLagType/TransitionStepLimit，参数改 CamelCase。

## 参考文献

[1] Ranneberg, M. *Viiflow—A New Inverse Viscous-Inviscid Interaction Method.* AIAA J. 57(6):2248-2253, 2019.
[2] Drela & Giles, AIAA J. 25(10):1347-1355, 1987.
[3] Van Rooij, R.P.J.O.M. *Modification of the boundary layer calculation in RFOIL…* IW-96087R, TU-Delft, 1996.
[4] Drela, *Integral boundary layer formulation for blunt trailing edges*, 1989.
[5] Clauser, J. Aeronaut. Sci. 21(2):91-108, 1954.
[6] Green, Weeks & Brooman, ARC R&M 3791, 1973.
[7] Rona, Monti & Airiau, Aeronaut. J. 116(1180):569-598, 2012.
[8] van der Horst et al., 34th Wind Energy Symposium, 2016.
[9] Ye, B. *The Modeling of Laminar-to-turbulent Transition…* MSc, TU-Delft, 2015.
[10] Thomas, J. 17th Fluid Dynamics Conf., 1984.
[11] Mokry, M. BEM 42:211, 2006.
[12] Hess & Smith, Prog. Aerosp. Sci. 8:1-138, 1967.
[13] Drela, *XFOIL…*, Springer LNA, 1989.
[14] Drela, *MISES… modified Abu-Ghannam/Shaw transition*, 1995.
[15] Chen & Price, Appl. Math. Mech. 33:1467-1480, 2012.