# 03. Maya USD 动画导出与变换继承陷阱

## 1. 痛点现象 (Symptoms)
在 Maya 中使用 MayaUSD 插件导出带有蒙皮骨骼动画或多级父子层级变换的角色/道具时，导入 Houdini Solaris 或 Unreal Engine 5 经常发生以下灾难：
- 角色模型原地踏步，而世界位移漂移到了宇宙尽头；
- 模型网格被双重变换 (Double Transform)，导致身体部位缩放拉伸变形；
- 骨骼 Joint 的局部旋转与网格不同步，手臂折叠畸变。

---

## 2. 根因剖析 (Root Cause)

1. **xformOpOrder 的解析差异**：
   Maya 内部默认使用基于矩阵级联的 DAG 评估；而 OpenUSD 强制采用声明式的 `xformOp` 算子列表（如 `xformOp:translate`, `xformOp:rotateXYZ`, `xformOp:scale`）。如果在 Maya 导出选项中没有勾选 `Euler Filter` 或保持原生变换栈，MayaUSD 可能会导出混合了 `xformOp:transform`（矩阵）与独立通道的冗余结构，导致下游 DCC 解析错误。
2. **SkelRoot 边界丢失与 Double Transform**：
   在 USD 规范中，骨骼网格体 (UsdGeomMesh) 与骨骼动画 (UsdSkelSkeleton) 必须严格封装在一个被标记为 `kind = component` 且类型为 `SkelRoot` 的图元之下。
   如果 Maya 导出时将 Mesh 放在 SkelRoot 之外，或者将根节点的位移同时烘焙到了 Mesh 自身的 `xformOp:translate` 和 Skeleton 的 `joints` 根节点上，下游渲染器会叠加计算两次世界矩阵，造成严重的双重变换。

---

## 3. 规约与解决方案 (Solution)

### 3.1 导出规约
- **严格遵循 SkelRoot 封装结构**：
  ```
  /Character (SkelRoot)
      /Meshes (Scope)
          /body_mesh (Mesh, primvars:skel:jointIndices, primvars:skel:jointWeights)
      /Rig (Scope)
          /skeleton (Skeleton, joints, restTransforms, bindTransforms)
      /Animation (SkelAnimation, translations, rotations, scales)
  ```
- **Maya 导出设置建议**：
  - 勾选 `Export Skels = auto` 或 `explicit`；
  - 勾选 `Export Skin = true`；
  - 禁用几何体本身的 Transform 烘焙，将一切位移交给 Skeleton 根骨骼驱动；
  - 导出格式优先选择二进制 `.usdc` 以保证浮点精度与加载效率。

### 3.2 骨骼动画检验脚本

```python
from pxr import Usd, UsdGeom, UsdSkel

def validate_character_skelroot(stage_path: str):
    stage = Usd.Stage.Open(stage_path)
    skel_roots = [p for p in stage.Traverse() if p.IsA(UsdSkel.Root)]
    
    if not skel_roots:
        print("[ERROR] 未检测到任何 UsdSkelRoot 图元！动画将无法被 Karma/Unreal 正确识别。")
        return False
        
    for sr in skel_roots:
        print(f"[OK] 找到有效 SkelRoot: {sr.GetPath()}")
        # 检查其子集是否存在 Mesh 与 Skeleton
        has_mesh = any(p.IsA(UsdGeom.Mesh) for p in Usd.PrimRange(sr))
        has_skel = any(p.IsA(UsdSkel.Skeleton) for p in Usd.PrimRange(sr))
        
        if not (has_mesh and has_skel):
            print(f"[WARNING] SkelRoot {sr.GetPath()} 缺少 Mesh 或 Skeleton 组成部分！")
        else:
            print("  -> 网格与骨骼层级封装合规。")
    return True
```
