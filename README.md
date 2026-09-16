# Houdini Shape Grammar · 程序化建筑生成 HDA

> 一套面向 **Houdini 22 / Solaris（LOP）** 的 USD 程序化建筑生成数字资产（HDA）。
>
> 核心思路：把一栋建筑拆成可复用的 **模块（Module）**，再用一条 **形状语法（Shape Grammar）** 描述
> "每一层由哪些模块、按什么顺序、以什么概率排列"，剩下的排版、随机变化、模块贴合、屋顶生成全部交给节点完成。
>
> **本资产改编自 Houdini 官方 SideFX Labs 的 `Labs Building from Patterns` 节点**（详见 [来源与改编说明](#来源与改编说明)）。

---

## 目录

- [这是什么](#这是什么)
- [来源与改编说明](#来源与改编说明)
- [仓库内容](#仓库内容)
- [环境要求](#环境要求)
- [安装方法](#安装方法)
- [节点详解](#节点详解)
  - [1. Building Generator LOP / Floors](#1-building-generator-lop--floors)
  - [2. USD Module Mark](#2-usd-module-mark)
  - [3. USD Rebuild & Rename](#3-usd-rebuild--rename)
- [典型工作流](#典型工作流)
- [数据约定（Primvar）](#数据约定primvar)
- [注意事项与已知限制](#注意事项与已知限制)
- [许可证](#许可证)
- [English](#english)

---

## 这是什么

Houdini 里搭建筑通常有两种做法：手工摆件，或者写一堆循环/COP 逻辑。这套 HDA 走的是第三条路 ——
**形状语法（Shape Grammar）**：

1. 你准备一批"零件"（墙、窗、门、阳台……），每个零件就是一个 USD 模块；
2. 你写一条规则，比如 `[Base]<Wall>`，意思是"先摆底座，再沿外墙排墙模块"；
3. 给规则里的每个符号挂上候选模块、尺寸、权重和随机种子；
4. 生成器读取一个**体块（Blockout）** USD，自动把规则展开、逐层排版、做角度/接缝修正，
   最后输出一栋带屋顶面的完整建筑。

配合作者编写的 USD 前置工具（重建层级 / 重命名 / 打模块元数据），
整条链路可以在 Solaris 里跑成一个纯 USD 的程序化管线。

---

## 来源与改编说明

本仓库的资产**并非从零编写**，而是在 Houdini **官方（SideFX Labs）** 节点的基础上**改编并扩展**而来：

| 官方节点 | 官方定位 | 与本资产的关系 |
| --- | --- | --- |
| **`Labs Building from Patterns`** | *Creates buildings from blockout geometry defined by a pattern of floor modules.* —— 用"楼层模块的排列规则"从体块生成建筑 | **主要改编来源**。本生成器的形状语法思路、`Floor Descriptions`（楼层描述）、体块驱动的工作流与术语体系均沿用自它 |
| **`Labs Building Generator Utility`** | *Creates and configures building modules.* —— 制作并配置建筑模块 | 本资产**内部直接调用**（`labs::building_generator_utility::2.0`）来做模块注册 |
| **`Labs Align and Distribute`** | 把几何体按线性 / 网格排布 | 本资产**内部直接调用**（`labs::align_and_distribute::2.0`）做模块对齐与分布 |

具体来说，[Building Generator LOP](#1-building-generator-lop--floors) 可以理解为
**官方 `Labs Building from Patterns` 的 Solaris / LOP 版本，并在其基础上做了大幅扩展**：

- **沿用**：形状语法的规则表达方式（`[Base]<Wall>` 这类规则 + 展开式）、
  `Floor Descriptions` 楼层描述、体块（Blockout）驱动、模块注册（Module Register）这一整套思路与术语；
- **扩展**：楼层与模块的**权重 / 随机变体**、模块级尺寸覆盖、**成对重叠（Pair Overlap）**、
  整体转角与转角优先级、由外部 SOP 驱动的楼层描述 / 开洞 / 楼层覆盖、
  屋顶面独立输出，以及配套的 USD 模块标注与 USD 重建重命名工具；
- **移植**：从 SOP / 几何上下文迁移到 **Solaris（LOP）/ USD** 上下文，最终产物直接是 USD 而非几何体。

> 资产内部还留有 `WanX::building_from_patterns_overlap::1.0` 这类定义，
> 从命名上即可看出它与官方 `Building from Patterns` 的传承关系。

**因此 SideFX Labs 是本资产的必需依赖，而不是可选项。**

> 版权说明：上游 SideFX Labs 节点及相关代码的版权归 SideFX 所有，遵循其自带许可；
> 本仓库的 MIT 协议仅覆盖本资产中作者自有的部分。

---

## 仓库内容

本仓库只包含可用的 HDA 资产本体，不含任何运行期几何数据或项目文件。

| 文件 | 主要节点类型 | 作用 |
| --- | --- | --- |
| `WanX_Building_Generator_FINAL.hda` | `WanX::building_generator_lop_floors_*`、`WanX::USD_Module_Mark_Extended_*` | 主体资产库：**建筑生成器** + **模块标注工具** |
| `lop_WanX--USB_Rebulit_and_Rename-1.0.hda` | `WanX::USB_Rebulit_and_Rename::1.0` | 前置准备：重建 USD 层级、把根重命名为 `/ShapeGrammar`、写入模块元数据 |

> `WanX_Building_Generator_FINAL.hda` 是一个**资产库**，内部打包了同一族的多个版本/辅助定义，
> 例如 `WanX::module_register*`、`WanX::module_rules::1.0`、`WanX::building_from_patterns_overlap::1.0`
> 以及若干带日期戳的历史快照。日常使用只需要认下面这三个（见 [节点详解](#节点详解)）。

---

## 环境要求

| 项目 | 要求 |
| --- | --- |
| Houdini | **22.0.368**（开发版本；其它 22.x 通常也可用） |
| 工作流 | Solaris / LOP（USD），需要启用 `pxr` Python 模块 |
| 依赖 | **SideFX Labs** —— 本资产的改编基础，**必需**（见 [来源与改编说明](#来源与改编说明)） |
| 平台 | Windows / Linux / macOS 均可（开发环境为 Windows） |

> **SideFX Labs 是必需依赖**，不只是"顺手引用几个节点"：本资产正是从 Labs 的
> `Building from Patterns` / `Building Generator Utility` 改编而来，
> 内部直接调用了 `labs::building_generator_utility::2.0` 与 `labs::align_and_distribute::2.0`。
> 未安装 Labs 时，模块注册与对齐分支会报缺失节点。

---

## 安装方法

任选一种：

**方法 A — 在 Houdini 里安装（推荐）**

1. 打开 Houdini → `File ▸ Install Digital Asset Library…`
2. 选择本仓库里的 `.hda` 文件（两个都要装）

**方法 B — 复制到用户资产目录**

把 `.hda` 放进 `$HOUDINI_USER_PREF_DIR/otls/`（Windows 一般是
`C:\Users\<你>\Documents\houdini22.0\otls\`），重启 Houdini。

**方法 C — 用环境变量扫描目录**

```bash
# 把仓库目录加入资产扫描路径
export HOUDINI_OTLSCAN_PATH="/path/to/this/repo:$HOUDINI_OTLSCAN_PATH"
```

安装后，在 `/stage` 网络里按 <kbd>Tab</kbd> 搜索 `Building Generator` 或 `Module Mark` 即可调出。

---

## 节点详解

### 1. Building Generator LOP / Floors

**节点类型：** `WanX::building_generator_lop_floors_20260910_081730::1.0`
**标签：** `WanX Building Generator LOP / Floors`

这是整套工具的主角：读入体块，按形状语法生成建筑。

**输入**

| 输入槽 | 内容 | 说明 |
| --- | --- | --- |
| Input 1 | Base Modules | 基础模块库（USD） |
| Input 2 | Blockout | 体块 USD，决定建筑的楼层高度与占地轮廓 |
| Input 3 | Extra Modules | 额外模块库（需勾选 `Use Extra Module Input`） |

**主要参数**（完整参数在节点界面上分两个页签）

- **Shape Grammar / 形状语法**
  - `Building Pattern`：建筑级规则，默认 `[Base]<Wall>`。
  - `Floor Descriptions`（多实例）：每层一条 —— 名称、**Expanded Form 展开式**（如 `<Wall>`）、
    层高/宽、整体转角、起止转角、优先级、权重、**Variation 变体与权重**。
  - `Building Modules`（多实例）：按名称/模式匹配模块，可覆盖尺寸、优先级、权重与变体。
  - `Enable Module Transforms and Overlap`：总开关。
  - `Module Transforms`（多实例）：对指定模块做旋转 / 平移修正。
  - `Pair Overlap`（多实例）：指定两个模块之间的重叠量（A/B 模块、重叠值、是否双向）。
  - `Excluded Module Names / Categories`：排除不参与排版的模块。
  - `Variation Settings`：楼层种子、模块种子、是否按层偏移种子、是否强制高度缩放。
  - `External Floor Descriptions`：从外部 SOP 读取楼层描述（而不是用界面参数）。
  - `Cutout / Floor Override`：用外部 SOP 做开洞与楼层覆盖（含覆盖半径）。
  - `Module Sources`：从外部 USD 文件追加模块库。
- **Output / 输出**
  - `Building USD Root`：输出根路径，默认 `/GeneratedBuilding`。
  - `Roof Horizontal Tolerance`：判定"屋顶面"的水平容差。
  - `Align Lower Roofs to Floor Below`：裙房屋顶是否对齐到下一层。

**输出**

生成一栋建筑，挂在 `Building USD Root` 下；屋顶面写成独立子层 `/…/RoofSurfaces`，
并附带 `roof_id`、`roof_height`、`source_prim`、`floor_pattern` 等属性便于后续筛选。

**内部实现（概要）**：`PROCESS`（sopmodify）子网里是一整套 SOP 流水线 ——
导入体块与模块、枚举拓扑、用 `module_register` 注册模块并量取包围盒、
用 VEX/Python 做语法展开与随机挑选、匹配尺寸、局部化、贴合，最后回写为 USD。
其中超过 70 段 VEX / Python 代码块负责语法展开、转角处理、屋顶绕序修正等细节。

---

### 2. USD Module Mark

**节点类型：** `WanX::USD_Module_Mark_Extended_20260909_230316::1.1`
**标签：** `Usd module mark Extended`

给 USD 舞台里的"模块"打标注 —— 决定每个模块叫什么名字、属于哪一类、怎么缩放旋转。

**输入：** 上游 USD 舞台。
**主要参数**

- `Module Root`：模块根路径（默认 `ShapeGrammar`）。根下每个"直接子级且含 Mesh"的 prim 被视作一个模块。
- `Module Mark`（多实例块），每块包含：
  - `Primitive Selection`：用 LOP 选择规则（path pattern）选中一批 prim；
  - `Mark`：类别标签；`Module Name`：模块名（需为合法标识符）；
  - `Register Selection as One Module`：把选中多个 prim 合并注册成一个模块（会自动算包围盒、建 `SGC_<name>` 节点）；
  - `Scale XYZ` / `Rotation Correction` / `Module Width` / `Module Height`。
- **注册模块预览**：`Choose and Show Registered Module` 会弹出列表，
  直接在视窗里单独显示某个已注册模块，`Return to Module Mark` 退出预览。
- 视窗里还有模块拾取：点选场景中的模块，自动把路径填回选择参数。

**它写出的数据**：在每个模块 prim 上写一组 primvar（见 [数据约定](#数据约定primvar)）。

---

### 3. USD Rebuild & Rename

**节点类型：** `WanX::USB_Rebulit_and_Rename::1.0`（"USB" 是 **USD** 的笔误）

管线里的"前置准备"节点。把一批外部 USD 资产**重建**成一个规整的层级，并**重命名**根节点，
使下游的生成器能直接把它当成 `ShapeGrammar` 模块根来用。

内部网络（`hdaroot`）包含：

| 节点 | 作用 |
| --- | --- |
| `PYTHON_REBUILD` | 遍历输入舞台，找到每个模块 prim 上最强的一条 reference/payload；在新根下按**相同的相对路径**重建层级，并重新挂上指向外部资产的 payload。无法重建的 prim 会被记录跳过。 |
| `PYTHON_MODULE_METADATA` / `pythonscript2` | 在模块 Xform 上写入**稳定的、可继承的 constant primvar**（模块 ID、实例 ID、资产名、分类），ID 由内容哈希生成，保证多次运行一致。 |
| `pythonscript1` | 用 `Sdf.BatchNamespaceEdit` 把根路径 `/Rebuilt` 改名为 **`/ShapeGrammar`**。 |
| `fix_black` | 修复某些 Mesh **缺少显式法线**导致的发黑/着色异常；遇到没有显式 normals 的 Mesh 会直接报错提示。 |
| `output0` / `output1` | 输出。 |

---

## 典型工作流

```
外部 USD 资产 / 模块库
        │
        ▼
┌───────────────────────────────┐
│ USD Rebuild & Rename          │  重建层级 · 根重命名为 /ShapeGrammar · 写模块元数据
└───────────────────────────────┘
        │  /ShapeGrammar
        ▼
┌───────────────────────────────┐
│ USD Module Mark               │  给每个模块定名字 / 类别 / 尺寸 / 旋转
└───────────────────────────────┘
        │
        ▼
┌───────────────────────────────┐
│ Building Generator LOP        │  形状语法展开 → 逐层排版 → 贴合 → 屋顶
│  Input1 Base / Input2 Blockout│
└───────────────────────────────┘
        │
        ▼
   /GeneratedBuilding （含 RoofSurfaces）
```

---

## 数据约定（Primvar）

`USD Module Mark` 会为每个模块 prim 写入以下 primvar（constant 插值）：

| Primvar | 类型 | 含义 |
| --- | --- | --- |
| `sg_module` | int | 1 = 模块，0 = 被合并/消耗掉的成员 |
| `sg_source_world` | matrix4d | 模块原始世界变换 |
| `sg_members` | string[] | 合并模块时，成员 prim 列表 |
| `module_type` | string | 类别标签（Mark） |
| `module_group` | string | 分组名 |
| `module_name` | string | 模块名 |
| `module_rotate` | double3 | 旋转修正 |
| `module_scale` | double3 | 缩放 |
| `module_width` | double | 宽度 |
| `module_height` | double | 高度 |

读取时也会参考上游可能存在的 `auto_module_type`（primvar 或同名属性）。

---

## 注意事项与已知限制

- **依赖节点命名/网络结构。** 生成器内部的部分 Python 逻辑引用了写死的场景路径，
  例如 `/stage/sopmodify2/modify/modify`、`/stage/sopmodify2`、`/stage/module_finalize`、
  `/stage/usd_module_mark1`。把它们放进自己的工程时，建议保持相同（或相近）的节点层级与命名；
  否则需要自行改这些常量。
- **版本戳节点名。** 资产类型名带有构建时间戳（如 `_20260910_081730`），
  升级资产后旧场景会引用不到新版本，属正常现象。
- **仅含资产定义。** 本仓库不含 `.hip` 示例文件、示例模型或 cooked 结果，
  需要自备模块库与体块 USD 才能跑通完整流程。
- **开发环境为 Houdini 22.0.368**，更低版本（如 20.x）不保证兼容。

---

## 许可证

MIT License，详见 [LICENSE](LICENSE)。

---

## English

A Houdini 22 / Solaris (LOP) toolkit for **procedural building generation driven by a shape grammar**.

This asset set is **adapted and extended from the official SideFX Labs
`Labs Building from Patterns` node** (together with `Labs Building Generator Utility`),
rather than written from scratch — see the 来源与改编说明 section above for the breakdown.

- **`WanX_Building_Generator_FINAL.hda`** — the main asset library:
  - `building_generator_lop` — reads a USD blockout plus a module library, expands a shape-grammar
    pattern (e.g. `[Base]<Wall>`) per floor, places/rotates/overlaps modules, and emits a full building
    with roof surfaces.
  - `usd_module_mark` — tags USD module prims with metadata primvars (type / group / name / size / rotation),
    with an interactive viewer-based module picker and preview.
- **`lop_WanX--USB_Rebulit_and_Rename-1.0.hda`** — preparation asset: rebuilds an external USD hierarchy,
  renames its root to `/ShapeGrammar`, authors stable module metadata, and repairs meshes that lack normals.

Requires Houdini 22 (developed on 22.0.368), the Solaris/LOP context, and **SideFX Labs** — the official
`Labs Building from Patterns` / `Labs Building Generator Utility` nodes are the foundation this asset was
adapted from, and several Labs utility nodes are called directly. Install via
`File ▸ Install Digital Asset Library…` or by dropping the `.hda`
files into `$HOUDINI_USER_PREF_DIR/otls/`.
