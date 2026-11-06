# 英伟达开源 CAD 与桌面无叶风扇设计 — 对话整理

> 2026-11-06 · 从 USD BRep 开源新闻一路聊到风扇参数与采购清单

---

## 一、新闻背景：英伟达开源了什么

- **2026-10-08** 英伟达在 GitHub 发布 **USD BRep v1.0.0**（仓库 `NVIDIA-Omniverse/usd-brep`），Apache 2.0 许可，PyPI 同日上架
- 本体是 **SMLib 几何内核**（20+ 年商业历史，英伟达 2022 年收购 Solid Modeling Solutions 所得），一直跑在 ClassCAD 上；底层 NLib 的作者就是《The NURBS Book》（行业"圣经"）的作者
- 开源内容：布尔/圆角/抽壳/曲面求交等建模操作、OpenUSD 双向读写（omniSolid schema）、几何修复、STEP/IGES/JT 转换、桌面 GUI
- 已知缺陷（官方自认）：修复可能静默失败、细分可能产出朝向错误的三角形、暂不接受外部贡献
- 动机：不是做 CAD 软件，而是把精确工程几何喂给 AI 和 Omniverse 数字孪生管线
- 圈内反应：西门子 Parasolid（30+ 年，SolidWorks/Onshape 都在用）不慌；社区快速跟进（smlibpy、Rust 移植 solidcore、SolidCraft 迁移）

### 名称关系

| 名字 | 是什么 |
| --- | --- |
| USD BRep | 英伟达官方开源项目名（仓库 usd-brep），内含 SMLib 内核 |
| SMLib | 内核本名（Solid Modeling Library） |
| smlibpy | ClassCAD 团队（AWV Informatik）晚 4 天发布的 Python 易用封装，**MIT 许可，支持 Mac** |

---

## 二、Mac 上怎么用（路线）

### 官方 usd-brep：macOS 受限

- PyPI 无 macOS wheel（仅 Win/Linux，需 Python 3.12）
- Mac 只能源码编译，且**仅 Apple Silicon**；默认构建无 OpenUSD/无官方 Python API，功能残缺

### smlibpy：Mac 的实际入口

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
mkdir ~/fan-project && cd ~/fan-project
uv init && uv add smlibpy
uv run python -c "import smlibpy; print(smlibpy.__version__)"
```

- wheel 覆盖 macOS（Apple Silicon + Intel）、Python 3.10–3.15，8-11MB，仅依赖 NumPy
- 建模示例：

```python
from smlibpy import *
plate = Box(100, 70, 4)
plate = plate.fillet(plate.edges().parallel_to(Z), 6)
plate -= GridLocations(58, 49, 2, 2) * Cylinder(1.4, 4)
export_step(plate, "plate.step")
```

### 连 Claude Code

- 路线 A：Claude Code 直接写脚本调用（装完即用）
- 路线 B：MCP 模式（官方支持）：

```
claude plugin marketplace add awv-informatik/smlibpy
claude plugin install smlibpy@smlibpy
```

- 其他宿主：`uvx --from "smlibpy[mcp]" smlibpy-mcp`
- 联网需求：安装要联网，**建模计算纯本地 CPU**，不传数据

### 能做什么 / 输出什么

- 实体建模全套 + 钣金折弯展开 + 点云拟合曲面 + 文字刻字 + 自由变形晶格（0.6 版）
- 输出：STEP / STL / GLB / DXF / USD（.smb/.smp 私有格式）
- `render()`：工程渲染（剖视、X 光、斑马纹、曲率云图、技术图纸、diff 对比），无灯光/背景控制
- `show()`：本地起服务弹浏览器，可交互 3D 查看（localhost，不出本机）

### 查看器（Mac）

- 空格预览（USDZ）、Reality Converter（苹果官方免费）、Blender（Cycles 照片级渲染，免费）、usdview（皮卡官方，能看场景图层结构）
- Omniverse **不支持 Mac**

---

## 三、USD 格式与 AI 管线

- USD 是皮卡开源的 3D 场景「集装箱」：分层组装（透明胶片式叠加）、跨软件收模型、分发给仿真/渲染/VR/机器人
- 默认货物是多边形网格；usd-brep 加了 omniSolid 舱位让精确 NURBS 原样上船
- 支持 USD 的 AI 管线：NVIDIA Cosmos（世界模型）、Isaac Sim（机器人）、fVDB（空间智能）、USD NIM（AI 出 USD 资产）、苹果 visionOS（USDZ）、Unity USD Toolkit、虚幻引擎
- 工程出厂仍走 STEP；USD 是数字孪生/XR/机器人圈的普通话

---

## 四、开源仿真生态（nTop 对标）

| 层 | 开源选项 |
| --- | --- |
| 隐式建模/晶格 | OpenVCAD（学术）、F13LD（f13ld.app，网页免费，TPMS 晶格）、libfive |
| 拓扑优化 | TopOpt（DTU 免费）、OpenLSTO、OpenPISCO、FEniTop、Plato |
| FEA | CalculiX、Code\_Aster、Elmer、FEniCS、Kratos、MOOSE |
| CFD | OpenFOAM（Docker 最省事）、SU2（brew 可装） |
| 英伟达 | Warp（可微分仿真）、Modulus（物理 ML） |

### Elmer（Mac 原生 CFD/多物理场）

- 芬兰 CSC 出品，GPL，有 macOS 安装包 + GUI（ElmerGUI）
- 可算流体/结构/热/电磁耦合；CFD 弱于 OpenFOAM（复杂湍流吃力）
- 案例本质是 case.sif 文本文件，Claude Code 可直接读写替人"填表"
- 适合当仿真第一课

### 其他 Mac CFD

- SimScale（浏览器云端，免费档）⭐ 推荐入门
- AirShaper（外部气流报告，按次付费）
- COMSOL（Mac 原生但贵）
- Ansys/Fluent/Star-CCM+ 均不出 Mac 版

---

## 五、桌面无叶风扇设计结论（核心成果）

### 结构定案

**超椭圆拱形（非闭合环，像拱桥）+ 双侧鼓风机**

- 拱腿立在底座上 = 进风道；拱身 = 均压管（空心截面 90/80mm）；拱背外缘开窄缝出风
- 风路减半、进风直达、结构稳固
- 拱顶内置**分流脊**（防两股风对撞湍流）
- 进风口开在底座**背侧或底部垫高**（别朝前，会吸回自己吹出的风）；进气面积 ≥ 鼓风机进风口 2-3 倍

### 气动计算（桌面版）

| 方案 | 缝宽 | 评价 |
| --- | --- | --- |
| 2×40mm + 跨度 320 | 0.7-1.1mm | ❌ 缝 FDM 打不出，满转啸叫 |
| **2×50mm (5015) + 跨度 240×拱高 135** | **1.2-1.8mm** | ✅ 推荐甜点 |
| 2×60mm | 1.8-2.7mm | ✅ 余量最大 |

预期效果：缝速 10-12 m/s，卷吸后总出风 **60-100 L/s**（落地扇中速档体感），中段转速即可、安静。

### 关键公式（改尺寸重算用）

- 拱弧长 ≈ 跨度 × 1.6；缝面积 = 弧长 × 缝宽
- 一次流量 Q = 缝面积 × 缝速
- 所需静压（良好唇口设计）≈ 0.5·ρ·(缝速/0.62)²·0.8 ≈ 80-180 Pa
- 卷吸倍增 5-10 倍

### 采购清单

1. **5015 离心鼓风机 ×2**（滚珠轴承版，12V/5V，\~10-20 元/个）
2. 可调压电源（按风机功率 ×1.5 冗余）
3. 热线风速计（\~30 元，验收缝速）
4. 备选：二手戴森拆机鼓风机（备用方案）

### 省电三件套（缝的设计）

1. 缝入口圆角 R0.5
2. 缝唇口 0.5-1mm 渐缩薄唇
3. 内腔光滑渐扩

### 实施顺序

1. 买 5015×2 + 风速计
2. 纸箱粗原型实测缝速（10 分钟验证风量压头）
3. 锁定参数表 → Claude Code + smlibpy 建模（拱放样→抽壳→切缝）
4. render 剖视验收（缝唇、分流脊）→ 打印
5. （可选）SimScale/Elmer 做 2D 截面 CFD 复核

---

## 六、相关链接

- usd-brep 仓库: github.com/NVIDIA-Omniverse/usd-brep
- smlibpy: github.com/awv-informatik/smlibpy · 文档 smlibpy.classcad.ch
- 网页版内核: npm @classcad/smlibwasm（浏览器可交互）
- TPMS 晶格工具: f13ld.app
- SimScale: simscale.com · Elmer: elmerfem.org
