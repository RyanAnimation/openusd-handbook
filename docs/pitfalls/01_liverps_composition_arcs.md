# 01. LIVERPS 组合弧与 Over 覆写陷阱

## 1. 痛点现象 (Symptoms)
在 Houdini Solaris 或 Maya USD 中，下游灯光或镜头艺术家尝试通过 `over` 节点覆写某个资产的材质属性或变换矩阵，但在渲染或回放时发现：
- 覆写属性完全不生效，仍然显示上游资产的原始材质；
- 或者在切换变体 (Variant) 时，覆写属性突然被重置回默认值；
- 场景中多层 Layer Stack 之间顺序看似正确，但特定图元的某些属性始终被上一层强势“锁死”。

---

## 2. 根因剖析 (Root Cause)

OpenUSD 的组合弧 (Composition Arcs) 遵循严格的裁决强度顺序，业内缩写为 **LIVERPS**：

```
L (Local Opinions)           -- 本层 Local 观点（最高优先级）
I (Inherits)                 -- 继承类
V (VariantSets)              -- 变体集选择
E (Specializes)              -- 专化类
R (References)               -- 外部引用
P (Payloads)                 -- 外部大资产负载
S (SubLayers)                -- 子层嵌套（最低优先级）
```

**核心陷阱**：
很多初学者误以为“只要位于最顶层 SubLayer，就能覆写底层一切属性”。
**实际上，SubLayers 的优先级仅在同级弧内生效**。如果下游覆写是在一个 SubLayer 中对某个已经跨跨 Reference 引入的图元进行修改，而上游文件在图元内部的 VariantSet 中硬编码了该属性，那么由于 **VariantSets (V) > References (R) > SubLayers (S)**，底层 VariantSet 中的内部 Local 意见在特定组合下会表现出超预期的抗覆写特性。

此外，使用 `over` 声明时，如果该 PrimPath 在合成后的 Stage 中并不存在（例如父级 Prim 尚未被 `def`），该 `over` 将成为无效的“悬空孤立覆写”（Dangling Over），被 Stage 自动忽略。

---

## 3. 规约与解决方案 (Solution)

### 3.1 黄金规则
1. **Def 负责生，Over 负责养**：永远不要在底层资产库中使用 `over` 创建几何体核心图元；资产库只输出带有标准 `def Xform` / `def Mesh` 的资产。
2. **覆写必须指向明确存在的 SdfPath**：覆写前，确保上游已经由 Reference 或 Payload 将该路径加载进入 Stage。
3. **分清 LayerStack 与 Reference 的强弱边界**：若需对 Reference 资产属性进行最高权限覆写，直接在当前 Root Layer 的 Local Opinion（即本层）写入，而不是包裹在深层子引用中。

### 3.2 自动化 Python 检测与修复示例

```python
from pxr import Usd, Sdf

def inspect_property_strength(stage: Usd.Stage, prim_path: str, prop_name: str):
    prim = stage.GetPrimAtPath(prim_path)
    if not prim.IsValid():
        print(f"[ERROR] 目标图元不存在: {prim_path}")
        return
    
    prop = prim.GetProperty(prop_name)
    if not prop.IsValid():
        print(f"[ERROR] 属性不存在: {prop_name}")
        return
        
    # 查询属性观点强度堆栈 (Property Stack)
    prop_stack = prop.GetPropertyStack()
    print(f"=== 属性 {prop_name} 强度裁决堆栈 (自强至弱) ===")
    for idx, prop_spec in enumerate(prop_stack):
        layer = prop_spec.layer
        print(f"  [{idx}] Layer: {layer.identifier} | Value: {prop_spec.default}")

# 用法示例
# stage = Usd.Stage.Open("shot_01.usd")
# inspect_property_strength(stage, "/World/Asset/geo", "xformOp:translate")
```

---

## 4. 工业生产避坑速查表
- 避免在 Class (`_class_`) 继承链中定义高频变动属性；
- 如果必须在镜头层强行覆写被底层 Variant 锁死的属性，请在当前 Stage Root Layer 的最高优先级 Local 根层执行强行覆盖，或通过编辑目标 (`stage.SetEditTarget`) 显式指定写入层。
