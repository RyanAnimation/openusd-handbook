# 13. Purpose 分流与视口减负规约

## 1. 痛点现象 (Symptoms)
在视口中操作复杂场景时，视口交互极其卡顿；或者在最终离线渲染时，视口专用的辅助骨骼线框、碰撞体网格与粗模代理意外出现在最终渲染成片中。

---

## 2. 根因剖析 (Root Cause)

OpenUSD 定义了标准图元用途标记属性：`purpose` (Token 属性)。
标准支持的四大 Purpose：
- `default`：默认可见性（未显式声明时的常规图元）；
- `render`：最终高精渲染图元（视口可隐藏，离线渲染时强制显示）；
- `proxy`：低模代理（仅供视口交互实时显示，离线渲染时强制忽略）；
- `guide`：骨骼、参考线、辅助网格与相机标尺。

如果艺术家在建模或装配时没有显式为子网格设置 `purpose = "proxy"` 或 `purpose = "render"`，渲染器会将所有几何体（甚至碰撞包围盒）作为 `default` 一并打入光线追踪加速结构 (BVH)，造成渲染变慢与模型重叠穿模。

---

## 3. 规约与解决方案 (Solution)

### 3.1 标准资产分流架构
在资产的 `/geo` 目录下明确分离：
```
/geo
    /render (Mesh, purpose = "render") -> 400,000 面高精度曲面
    /proxy  (Mesh, purpose = "proxy")  -> 2,000 面低模视口代理
    /guide  (BasisCurves, purpose = "guide") -> 辅助定位轴
```

### 3.2 批量打标 Python 示例

```python
from pxr import Usd, UsdGeom

def enforce_purpose_standards(stage: Usd.Stage):
    for prim in stage.Traverse():
        if prim.IsA(UsdGeom.Imageable):
            img_api = UsdGeom.Imageable(prim)
            name = prim.GetName().lower()
            if "proxy" in name:
                img_api.GetPurposeAttr().Set(UsdGeom.Tokens.proxy)
                print(f"[PURPOSE] 标记为 PROXY: {prim.GetPath()}")
            elif "render" in name or "high" in name:
                img_api.GetPurposeAttr().Set(UsdGeom.Tokens.render)
                print(f"[PURPOSE] 标记为 RENDER: {prim.GetPath()}")
```
