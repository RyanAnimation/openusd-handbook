# 04. Maya LookdevX 与 MaterialX 着色网络互通避坑

## 1. 痛点现象 (Symptoms)
在 Maya LookdevX 中搭建的 MaterialX (`.mtlx`) 着色网络，导出或引用到 Houdini Solaris (Karma) 或 Unreal Engine 5.5 时：
- 材质显示为粉红色（丢失着色器）；
- 法线贴图异常发黑，或者在接缝处出现明显的高光断层；
- 粗糙度与金属度通道倒错，颜色空间（Color Management）漂移，原本在线性 ACEScg 下的贴图在下游被按 sRGB 错误伽马校正。

---

## 2. 根因剖析 (Root Cause)

1. **着色器类型与实现标准不统一**：
   Maya LookdevX 支持 `standard_surface` (Autodesk/Arnold) 与 `open_pbr_surface` (OpenPBR 1.0)。若在 Maya 中使用了 Arnold 专有的闭合节点（如 `aiImage`、`aiColorCorrect`）而非纯粹的 `ND_image_color3` 等标准 MaterialX 标准节点定义 (NodeDef)，下游未集成 Arnold 编译器的渲染器（如 Karma XPU 或 Unreal Substrate）将完全无法解析这些未注册节点。
2. **法线贴图切线空间与颜色空间元数据缺失**：
   MaterialX 标准规定，贴图节点必须显式标注 `colorspace` 属性（如 `lin_srgb`、`acescg` 或 `raw`）。若缺少该属性，不同 DCC 会回退到各自的默认解释器（Maya 默认为当前工程设置，Karma 默认为 OCIO 规则），从而引发色差。
3. **UsdShade Material Binding 路径相对性错误**：
   MayaUSD 在输出材质绑定关系 (`material:binding`) 时，有时会输出绝对路径 `/World/Materials/M_Wood`，而当该资产作为子引用拼接到下游主场景 `/Shot_01/Props/Table/World/Materials/M_Wood` 时，全局绝对路径失效，导致材质断链。

---

## 3. 规约与解决方案 (Solution)

### 3.1 材质中台规约
1. **只使用 OpenPBR 标准节点**：以 `ND_open_pbr_surface_surfaceshader` 为统一着色模型，弃用特定渲染器私有节点；
2. **贴图节点必填 `colorspace`**：
   - 漫反射/基色：`colorspace="acescg"` 或 `colorspace="srgb_texture"`；
   - 法线/粗糙度/金属度/位移：强行指定 `colorspace="raw"`。
3. **封装材质在资产内部并使用相对路径绑定**：材质必须封装在组件资产内部的 `/Materials` Scope 下，通过相对关系或直接绑定确保重组装不丢材质。

### 3.2 自动化颜色空间与绑定检查

```python
from pxr import Usd, UsdShade

def audit_materialx_bindings(stage_path: str):
    stage = Usd.Stage.Open(stage_path)
    materials = [p for p in stage.Traverse() if p.IsA(UsdShade.Material)]
    print(f"=== 找到 {len(materials)} 个 UsdShadeMaterial ===")
    
    for mat in materials:
        mat_api = UsdShade.Material(mat)
        # 遍历所有 Shader
        for shader_prim in Usd.PrimRange(mat):
            if shader_prim.IsA(UsdShade.Shader):
                shader = UsdShade.Shader(shader_prim)
                shader_id = shader.GetIdAttr().Get()
                print(f"  Shader: {shader_prim.GetPath()} | ID: {shader_id}")
                
                # 检查贴图节点
                if "image" in str(shader_id).lower():
                    cs_input = shader.GetInput("colorspace")
                    if not cs_input.IsValid() or not cs_input.Get():
                        print(f"    [WARNING] 贴图缺少显式 colorspace 标注: {shader_prim.GetPath()}")
```
