---
name: viiflow
description: VIIFLOW（viiflow）二维翼型粘性-无粘干扰（VVI）求解器技能。当任务涉及 viiflow/VIIFLOW、翼型极线/升阻力/转捩快速分析、XFOIL/RFOIL 替代或对标、翼型反设计、俯仰/空化/自由面/地面效应/叶栅、流固耦合、翼型参数优化、宿主面板法+积分边界层(IBL) 时自动激活。Triggers: viiflow, VIIFLOW, 粘性无粘干扰, VVI, 面板法, 积分边界层, IBL, 翼型极线, polar, 转捩位置, Ncrit, ShearLagType, RFOIL替代, 反设计, 空化, cavitation, 自由面, 地面效应, 叶栅, cascade, 俯仰翼型, pitch rate, FSI, viiflowtools, repanel, read_selig。
---

# VIIFLOW（viiflow）——二维翼型 VVI 分析技能

> 说明书全文已归档：`references/manual.md`（来自官方 `Installation.html` + 1.9.4 wheel METADATA）。
> 本机工作站已装并冒烟通过，见下「安装事实」。

## 0. 定位

- 作者 Maximilian Ranneberg，AIAA J. 2019《Viiflow—A New Inverse Viscous-Inviscid Interaction Method》。
- **无粘面板法 + 积分边界层(IBL)，Newton 法耦合**（以给定位移厚度解 BL 再耦合），比 XFOIL 在面板多时更快、更稳。
- 输入：翼型坐标 + Re/Ma/α/Ncrit → 输出 **CL、CD、CM、转捩位置、Cp 分布、边界层量（θ/H/ue/cf…）**。
- 适用：亚声速单/多段翼，无强尾流汇合或大分离；俯仰、空化、自由面、地面效应、叶栅、FSI、参数优化、反设计。
- 接口：`import viiflow as vf`（编译 .pyd）+ `import viiflowtools.vf_tools as vft`（读/重分坐标、画图）。**无 GUI，纯脚本**。

## 1. 安装事实（本机工作站 192.168.20.107）

- 版本：**viiflow 1.9.4**（`viiflow-1.9.4-cp312-cp312-win_amd64.whl`）+ **viiflowtools 1.7.4**（`viiflowtools-1.7.4-py3-none-any.whl`）。
- 安装位置：`E:\viiflow_install\`（wheel 与 venv 同目录）。
- 解释器：`E:\viiflow_install\venv\Scripts\python.exe`（venv 用 `--system-site-packages` 创建，复用系统 numpy/scipy/matplotlib）。
- 冒烟脚本：`E:\viiflow_install\smoke.py`（本 skill 副本 `scripts/smoke.py`）。运行：
  `E:\viiflow_install\venv\Scripts\python E:\viiflow_install\smoke.py`
- 2026-10-10 冒烟结果（NACA0012, Re=1e6, Ncrit=9）：α=0→8° 全部收敛，CL 0→0.919/8°（斜率≈0.10/°），CD 0.00543→0.01186。
- **已知偏差**：wheel 声明 `numpy<2.5,>=2.4`，工作站系统 numpy 是 **2.5.3**；实际导入/求解正常（numpy 2.x ABI 兼容）。若后续出现诡异崩溃，在 venv 内装 `numpy>=2.4,<2.5` 复测。
- 环境要求：Python **3.12.x**（cp312 编译）、numpy、scipy、matplotlib（工具包）。**不要**把系统 numpy 降到 2.4 去满足 pin（会连坐 torch 等）。
- 重新下载：官方仅提供 wheel，链接 `https://viiflow.com/download-viiflow`（Windows 直链 `https://viiflow.com/download/486/`，viiflowtools `https://viiflow.com/download/509/`）；下载后**不要改名**。

## 2. 最小用法骨架

```python
import numpy as np
import viiflow as vf
import viiflowtools.vf_tools as vft

# 坐标：Selig 顺序（TE 上表面 → 前缘 → 下表面 → TE），2xN ndarray
X = vft.repanel(vft.read_selig("RAE2822.dat"), 220)   # 或自备 2xN 数组

s = vf.setup(Re=1e6, Ncrit=9.0, Alpha=0.0, Silent=1)
[p, bl, x] = vf.init([X], s)          # Basic: 单段；Pro 才支持多段

for aoa in range(0, 9):
    s.Alpha = float(aoa)
    x, flag = vf.iter(x, bl, p, s)[:2]   # 1.9.4 返回 (x, flag, res, grad)，取前两个
    if flag:                             # flag=1 收敛
        print(aoa, p.CL, bl[0].CD)
```

**注意 1.9.4 API**：`vf.iter(...)` 实际返回 **4 元组** `(x, flag, res, grad)`，官方 Readme 里的
`[x,flag]=vf.iter(...)` 已过时；用 `out=vf.iter(...); x,flag=out[0],out[1]`。

## 3. setup 关键参数（完整表见 `references/manual.md`）

| 参数 | 含义 | 默认 |
|---|---|---|
| `Re` / `Ma` | 雷诺数 / 马赫数(<1) | 1e6 / 0 |
| `Ncrit` | 转捩 e^N 临界放大因子 | 9 |
| `Alpha` | 迎角(deg) | 0 |
| `Itermax` / `Tolerance` | Newton 最大步 / 收敛判据 | 100 / 1e-4 |
| `Silent` | 关迭代打印 | 0 |
| `Gradients` | 算梯度（Pro） | 0 |
| `ShearLagType` | 0=常数5.6；1=RFOIL Kc；2=XFOIL 型 | 0 |
| `LocusA`/`LocusB` | G-β 湍流轨迹常数 | 6.75 / 0.83 |
| `CLTarget` | 给定 CL 反求迎角(α 变成未知量) | inf |
| `PitchRate` | 俯仰率/来流速度（=c/R 用于旋转翼） | 0 |
| `Mirror` | 1=地面效应(y=0)；-1=自由面 | 0 |
| `FreeSurface`/`FroudeNumber` | 自由面 + 傅汝德数 | 0 / 1.0 |
| `IsCascade`/`CascadeStaggerHeight/Angle` | 叶栅 | 0 / 1 / 0 |
| `CmRefX`/`CmRefY` | 力矩参考点 | 0 / 0 |
| `BypassTransition`/`BypassA`/`BypassB` | 旁路转捩 | 1 / .1 / .3 |

设置 ShearLagType=1 可复现 RFOIL 的近/失速后行为（替代 RFOIL 的最简路径）。

## 4. 结果对象（`init`/`iter` 填入）

- `p`（无粘）：`p.CL` `p.CLi`(无粘) `p.CM` `p.alpha` `p.cp` `p.cp_inviscid` `p.gamma_inviscid` `p.gamma_viscid` `p.mach`；几何 `p.foils[0].X/.S/.normals/.hte`、`p.wakes[0].X`。
- `bl`（边界层，列表，每段一个）：`bl[0].CD` `bl[0].Re` `bl[0].ST`(驻点)；面/尾迹 `bl[0].bl_fl.nodes` / `bl[0].bl_wk.nodes`（numpy recarray）。
- 节点量（XFOIL 命名）：`typ`(1层流/2湍流/3尾迹) `delta` `theta` `ue` `H` `HK` `RT` `CF` `nct`(放大因子/剪切应力系数) `xi` `MSQ` `DI` …
  ```python
  ax.plot(bl[0].bl_fl.nodes.xi, bl[0].bl_fl.nodes.H)   # H(xi)
  ```

## 5. 进阶函数

- `vf.set_forced_transition(bl, p, [x_up], [x_lo])`：固定转捩位置（配 `s.Ncrit=np.inf` 只留强制转捩）。
- 空化（1.9.4 新增）：`pr = vf.cav_init(AF, s)`；`vf.cav_solve(pr, pr.s, sigma, nsubiter, callback, tol, restart, cavWeight)`。给 `pr.CL/CD/CM/CP_noncav`、`pr.cavdrag`、`pr.vd`、`pr.sigma`。不能与 CLTarget 同时用。
- 流场/可视化：`vf.inviscid_velocity(p, Xpts, vbuf)`、`vf.viscid_profile(node, Y, U)`、`vf.viscous_flowfield(bl[0], N)`；工具包 `vfp.plot_geometry(ax,p,bl)`、`vfp.total_flowfield`、`vfp.total_cpfield`、`vfp.calc_streamlines`。
- 梯度（Pro）：`partial.*`（迭代中残差对 α/虚位移的偏导）与 `total.*`（收敛解灵敏度，如 `total.cl_aoa` `total.cd_aoa` `total.cm_aoa`）；FSI 用 partial 驱动联合 Newton。角度单位 **rad**。
- 精细控制迭代：`s.Itermax=0` 后循环 `x,flag,res,grad = vf.iter(x,bl,p,s,res,grad)`，自行 `x -= step*inv(grad)@res`。

## 6. 硬坑（违反必翻车）

1. **不缩放**：几何不改到弦长 1。坐标是 mm 就得自己缩；系数按长度 1 计（会得到千级 CL），Re 也指坐标系里的 1。
2. **坐标顺序/后缘**：必须 Selig 序（TE 上 → 前缘 → TE 下）。钝后缘**不要**用底→顶连线封口，最后两条"腿"应指向后缘切线方向。不检查几何、不给友好报错。
3. **力矩参考点是坐标系原点**：要绕 1/4 弦，就把翼型移到 x=0 在该处，或设 `s.CmRefX=.25`。
4. **Basic 限制**：不支持多段翼、不支持梯度计算。多段/梯度需 Pro。
5. 有折角（如襟翼舱）用**负虚位移**填平，别让 BL 绕折角。
6. 结果口径：`viiflow` 是 2D 单翼型，**给的是气动力而非 T/Q/FM**；进本项目机队时按十段剖面 2D 极线来源使用，不替代内环 BEMT，也不做 CFD 真值。

## 7. 冒烟检查（改动后必跑）

```sh
E:\viiflow_install\venv\Scripts\python scripts\smoke.py
```
期望：导入 OK，α=0→8° 全部 `conv=1`，末尾打印 `ALL_CONVERGED`。

## 8. 版本/许可

- Basic（免费）：单段翼、无梯度。Pro（需向作者报价）：多段 + 梯度等。
- 与 RFOIL 关系：viiflow 自带 RFOIL 型剪切滞后（`ShearLagType=1`），可近似 RFOIL 行为，但**不是** TNO RFOIL 本体。
- 引用：Ranneberg, M. *Viiflow—A New Inverse Viscous-Inviscid Interaction Method.* AIAA J. 57(6):2248-2253, 2019。