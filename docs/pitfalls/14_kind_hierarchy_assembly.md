# 14. Kind 分类体系与装配边界规范

## 1. 痛点现象 (Symptoms)
- 在 Solaris 视口或 Maya USD 视口中双击选择物体时，无法整选资产，而是直接误选到了一个螺丝钉小零件面片；
- 场景装配层无法正确执行层次化变换选择，或者在导出点云散布 (Point Instancer) 时被报错拒绝。

---

## 2. 根因剖析 (Root Cause)

OpenUSD 的 **Kind 元数据** 决定了 DCC 视口的选择粒度与模型聚合边界（Model Hierarchy）。
Pixar 严格定义了 Kind 的继承关系：
```
model (抽象根类)
    ├── component (单体叶子资产，如一把椅子、一个角色)
    └── group (集合容器)
         └── assembly (大型装配体，如整座建筑、完整街道、摄影棚)
subcomponent (模型内部零件，不可作为 model 独立发布)
```

**核心规约铁律**：
- **Model 层级连续性法则**：一个 `group` 或 `assembly` 的所有父级节点必须全部也是 `group` 或 `assembly`；一个 `model` 内部**严禁**再嵌套另一个 `model`（除非它是 `group` 包含的独立 `component`）。
- 若在单体资产内部把子零件标成了 `component`，会导致视口选择逻辑彻底混乱。

---

## 3. 规约与解决方案 (Solution)

### 3.1 资产与镜头 Kind 标注准则
- 单个道具/角色根节点：`kind = "component"`；
- 镜头 Layout 分组节点：`kind = "group"`；
- 复杂布景/场景大装配：`kind = "assembly"`；
- 内部零部件（如车轮、螺丝、车门）：`kind = "subcomponent"` 或不标注。

### 3.2 自动化 Kind 层级验证脚本

```python
from pxr import Usd, Kind

def validate_model_hierarchy(stage_path: str):
    stage = Usd.Stage.Open(stage_path)
    for prim in stage.Traverse():
        model_api = Usd.ModelAPI(prim)
        kind = model_api.GetKind()
        if kind:
            is_model = model_api.IsModel()
            is_group = model_api.IsGroup()
            print(f"Prim: {prim.GetPath()} | Kind: {kind} | IsModel: {is_model} | IsGroup: {is_group}")
```
