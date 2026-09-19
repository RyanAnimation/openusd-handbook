# OpenUSD 跨 DCC 工业管线避坑手册 (Survival Guide)

> 💡 **编写准则**：本手册借鉴 [Matt Estela (cgwiki)](https://tokeru.com/cgwiki) 范式，坚持纯静态、零广告、高信息密度与开源共享。所有词条均源自真实影视动画与游戏工业生产中的实际踩坑案例与自动化工程实践。

---

## 为什么需要这本手册？

在现代影视视效 (VFX)、三维动画与高品质虚幻实时引擎管线中，**OpenUSD (Universal Scene Description)** 正在迅速取代传统的 FBX/Alembic 孤立缓存，成为跨软件生态的数据真理源 (Single Source of Truth, SSOT)。

然而，官方文档（Pixar USD API 文档、Autodesk Maya USD 文档、SideFX Solaris 文档）通常只交代“概念与标准 API 用法”，极少提及跨 DCC 混用时的边界条件与底层暗坑：

- 为什么在 Maya 中导出的角色动画在 Solaris 视口中变形塌陷？
- 为什么在 Katana 中引用的 USD 材质覆写无法传递到 Karma XPU？
- 为什么在超大规模资产场景中，Karma 渲染会遭遇 GPU 显存 OOM 崩溃？
- 什么是 Dangling Over（悬空孤立覆写），它如何悄悄造成百兆级的图元垃圾膨胀？
- 如何通过 Asset Resolver 2.0 彻底治理跨工作室团队的软链接与绝对路径断链？

本手册致力于为一线**工业管线 TD (Pipeline TD)**、**技术美术 (Technical Artist)** 与 **CG 开发者** 提供一本即查即用的案头实战排错字典。

---

## 避坑词条快速检索目录

| 编号 | 核心词条 | 涉及 DCC / 技术栈 | 痛点现象与规约重点 |
| :---: | :--- | :--- | :--- |
| [**01**](pitfalls/01_liverps_composition_arcs.md) | **LIVERPS 组合弧与 Over 覆写陷阱** | 全 DCC 通用 / pxr.Usd | 理解 7 阶优先级裁决，规避强组合弧覆盖失效 |
| [**02**](pitfalls/02_dangling_over_cleanup.md) | **SdfPath 路径与 Dangling Over 清洗** | 全 DCC 通用 / Python | 图元重命名后覆写孤立导致的显存与图元膨胀修复 |
| [**03**](pitfalls/03_maya_usd_animation_transform.md) | **Maya USD 动画导出与变换继承** | Maya 2024+ / MayaUSD | 根骨骼 Transform 继承失效与世界矩阵重算避坑 |
| [**04**](pitfalls/04_maya_lookdevx_materialx.md) | **Maya LookdevX 与 MaterialX 互通** | Maya LookdevX / USDShade | 材质网络跨 DCC 解析断链与颜色空间丢失修复 |
| [**05**](pitfalls/05_katana_usd_xgen_hair.md) | **Katana USD 工作流与毛发 (XGen) 限制** | Katana 6.0+ / Maya / Arnold | 程序化毛发生长曲线在 Katana Stage 中的层级映射 |
| [**06**](pitfalls/06_solaris_component_builder.md) | **Solaris Component Builder 规范封装** | Houdini Solaris / LOPs | 几何体、材质与变体集标准化资产结构封装 SOP |
| [**07**](pitfalls/07_solaris_payload_delayed_loading.md) | **Solaris Payload 延迟流式加载** | Houdini 20+ / Karma XPU | 超大规模资产 >50,000 面强制 Payload 解耦与视锥流式加载 |
| [**08**](pitfalls/08_materialx_unreal_substrate.md) | **MaterialX 与 Unreal 5.5 Substrate 映射** | MaterialX / UE 5.5 Substrate | OpenPBR 到 Substrate 模块化着色层级映射与黑边修复 |
| [**09**](pitfalls/09_unified_baking_mikktspace_lod.md) | **统一烘焙 MikkTSpace 切线与 LOD** | Houdini / Karma / Unreal | MikkTSpace 切线对齐与曲面豪斯多夫距离分级自动化 |
| [**10**](pitfalls/10_asset_resolver_context_pinning.md) | **Asset Resolver 2.0 与 Context Pinning** | pxr.Ar / C++ / Python | 跨多工作室 `asset://` URI 动态解析与版本锁定机制 |
| [**11**](pitfalls/11_python_usd_api_patterns.md) | **Python USD API 核心算子与批量巡检** | Python / pxr.Usd | 避免 Stage.Reload 阻塞，使用 SdfChangeBlock 批量更新 |
| [**12**](pitfalls/12_stage_assembly_relative_paths.md) | **跨 DCC Stage 组装根锚定与相对路径** | 全 DCC 通用 / Git / NAS | 根锚定相对路径规约，消除网络盘驱动器绝对路径依赖 |
| [**13**](pitfalls/13_purpose_token_viewport_culling.md) | **Purpose 分流与视口减负规约** | Solaris / Maya / UE | default/render/proxy/guide 四级分流与极速视口预览 |
| [**14**](pitfalls/14_kind_hierarchy_assembly.md) | **Kind 分类体系与装配规范** | 全 DCC 通用 / Stage 装配 | component/group/assembly 语义约束与选择边界 |
| [**15**](pitfalls/15_variantset_switching_traps.md) | **USD 变体集 (VariantSet) 避坑** | 全 DCC 通用 / LOPs | 跨层级多重变体覆写冲突与默认变体回退机制 |

---

## 配套开源工具链

本手册提供全套生产级开源排查脚本：
👉 **GitHub 仓库**：[`ryan-cg/openusd-production-tools`](https://github.com/ryan-cg/openusd-production-tools)
包含孤立覆写清理、超大资产 Payload 延迟加载重构、URI 解析门禁等即插即用脚本。

## 贡献与交流

本手册持续由一线工业实践反哺更新。若您在实际生产中遇到新的跨 DCC 疑难杂症，欢迎提交 Issue 或 Pull Request。如需企业级管线架构咨询或内部工作坊培训，请查阅 [顾问咨询通道](consulting.md)。
