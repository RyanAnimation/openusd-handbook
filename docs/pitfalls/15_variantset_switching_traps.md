# 15. USD 变体集 (VariantSet) 避坑与切换冲突治理

## 1. 痛点现象 (Symptoms)
- 在镜头层切换资产的变体（例如将 `shadingVariant` 从 `clean` 切到 `dirty`），视口纹理毫无变化；
- 嵌套多个 VariantSet 时（如同时包含 `damageVariant` 与 `displayVariant`），切换其中一个导致另一个被强制重置回初始状态。

---

## 2. 根因剖析 (Root Cause)

1. **变体观点与 Local Over 覆写冲突**：
   回顾 LIVERPS 规则：**Local Opinions (L) > VariantSets (V)**。
   如果下游镜头曾经针对图元直接做过局部材质覆写（产生了 Local Opinion），那么无论下游怎么切换 VariantSet，由于 Local 观点强度高于 Variant 内部观点，变体切换所带来的属性变动会被当前层的 Local 强行掩盖。
2. **多重变体定义顺序混乱**：
   VariantSet 内部嵌套另一个 VariantSet 时，若内外顺序颠倒，会导致外部无法独立评估内部变体分支。

---

## 3. 规约与解决方案 (Solution)

### 3.1 变体集正交化原则 (Orthogonality)
1. **职责单一化**：
   - 几何形态切换统一放入 `modelingVariant`；
   - 材质外观切换统一放入 `shadingVariant`；
   - 质量等级切换统一放入 `lodVariant`。
2. **严禁在 Variant 内部定义非正交属性**：
   `shadingVariant` 内部只允许包含 Material Binding 或 Shader 参数覆写，绝对禁止在其中定义新的几何体拓扑或骨骼结构。

### 3.2 变体状态诊断脚本

```python
from pxr import Usd

def diagnose_variant_selection(prim: Usd.Prim, vset_name: str):
    vsets = prim.GetVariantSets()
    if not vsets.HasVariantSet(vset_name):
        print(f"[ERROR] 图元 {prim.GetPath()} 不包含变体集 {vset_name}")
        return
        
    vset = vsets.GetVariantSet(vset_name)
    current_sel = vset.GetVariantSelection()
    options = vset.GetVariantNames()
    print(f"图元: {prim.GetPath()} | 变体集: {vset_name}")
    print(f"  当前选中: {current_sel} | 可选列表: {options}")
```
