# 06. Houdini Solaris Component Builder 规范封装

## 1. 痛点现象 (Symptoms)
在 Houdini Solaris 中制作资产时，如果手动使用多个 `Configure Primitive`、`Reference` 和 `Graft` 节点零散组装，经常出现以下问题：
- 资产层级深浅不一，缺乏统一的标准接口，下游镜头装配人员在引用资产时不知道哪个是根节点；
- 材质绑定混乱，有些绑定在 Mesh 上，有些绑定在 Scope 上，导致变体切换时着色器错乱；
- 缩略图、默认变体与 Payload 文件散落在外部，缺少自包含性，导致提交农场渲染时发生绝对路径丢失。

---

## 2. 根因剖析 (Root Cause)

SideFX 在 Houdini 18.5+ 引入了标准的 **Component Builder LOP** 工作流（由 `componentgeometry`、`componentmaterial`、`componentoutput` 组成的标准三节点管线）。
如果绕过标准工作流随意装配：
1. **缺少标准元数据**：如 `kind = "component"`、`defaultPrim` 未定义；
2. **Payload 结构松散**：未能将实际高模几何体正确剥离为独立的 Payload 层；
3. **变体集命名不统一**：没有遵循业界通用的 `modelingVariant`、`shadingVariant` 标准命名约定。

---

## 3. 规约与解决方案 (Solution)

### 3.1 Component Builder 标准四层结构
使用标准封装输出的资产，其 Sdf 层级结构必须严格保证如下四层骨架：

```
/ASSET_NAME (Xform, kind=component, defaultPrim)
    /geo (Scope)
        /render (Mesh, purpose=render)
        /proxy (Mesh, purpose=proxy)
    /mtl (Scope)
        /M_Material_01 (Material)
        /M_Material_02 (Material)
```

### 3.2 节点流水线规约
1. **Component Geometry LOP**：
   - 导入 SOP 几何体，在内部拆分为 Render 几何体与 Proxy 低模代理；
   - 自动添加 `purpose` 标记。
2. **Component Material LOP**：
   - 批量导入 MaterialX 材质；
   - 将材质直接绑定到 Geometry Scope 或 Mesh 路径，定义材质变体。
3. **Component Output LOP**：
   - 开启 `Separate Payload Layer`；
   - 将几何体剥离输出为 `asset_name_payload.usd`；
   - 主文件 `asset_name.usd` 仅保留 Xform 锚点、材质定义与 Payload 组合弧；
   - 开启 `Thumbnail Generation`，内嵌预览缩略图。

---

## 4. 工业生产自动化验收检查

```python
from pxr import Usd, UsdGeom

def inspect_component_asset(usd_file_path: str):
    stage = Usd.Stage.Open(usd_file_path)
    root_prim = stage.GetDefaultPrim()
    
    if not root_prim.IsValid():
        print(f"[ERROR] 缺少 defaultPrim 根锚点！")
        return False
        
    model_api = Usd.ModelAPI(root_prim)
    kind = model_api.GetKind()
    if kind != "component":
        print(f"[WARNING] 根节点 Kind 属性为 '{kind}'，标准单体资产应为 'component'")
        
    # 检查是否有 Payload 组合弧
    has_payload = root_prim.HasPayload()
    print(f"[OK] 资产根图元: {root_prim.GetPath()} | Kind: {kind} | 含 Payload: {has_payload}")
    return True
```
