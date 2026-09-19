# 05. Katana USD 工作流与毛发 (XGen) 资产导出限制

## 1. 痛点现象 (Symptoms)
在 Maya 中使用 XGen 交互式修饰毛发 (Interactive Groom Splines) 制作的角色毛发或生物绒毛，通过 USD 流程导入 Foundry Katana 并在 RenderMan/Arnold 渲染时：
- 毛发曲线完全无法被 Katana 识别，或被当作静态面片网格 (Polygon Mesh) 处理；
- 曲线宽度 (width) 丢失，毛发在视口中粗大如铁丝；
- 动画插值抽搐，毛发控制顶点 (Control Points) 随帧数变动导致时间步采样错位。

---

## 2. 根因剖析 (Root Cause)

1. **UsdGeomBasisCurves 几何规范要求**：
   OpenUSD 使用 `UsdGeomBasisCurves` 图元表达工业级毛发与曲线。它要求明确声明：
   - `type`：`cubic`（三次样条）或 `linear`（线性折线）；
   - `basis`：`bspline`、`catmullRom` 或 `bezier`；
   - `curveVertexCounts`：每条曲线包含的顶点数量数组；
   - `widths`：沿曲线的半径分布（必须标明 `interpolation = "vertex"` 或 `"varying"`）。
   Maya XGen 原生是基于隐式生成器的描述，若未经过显式几何烘焙而直接输出为通用 Mesh，Katana 的 USD 插件 (FnUsdIn) 会按照多边形图元对待，失去曲线原生细分与着色特性。
2. **点数恒定性 (Topology Invariance)**：
   若动力学解算过程中由于碰撞截断导致帧与帧之间顶点数发生变化，USD 会因为拓扑不匹配而无法插值生成运动模糊 (Motion Blur)。

---

## 3. 规约与解决方案 (Solution)

### 3.1 工业导出流程规范
1. **Maya 侧转烘焙**：在 Maya 中使用 `XGen -> Interactive Groom -> Convert to Curves`，或使用 MayaUSD 插件的专有毛发曲线转换器；
2. **强制写入 UsdGeomBasisCurves**：
   - 曲线基底采用 `basis = "catmullRom"` 或 `bspline`；
   - 明确标注 `wrap = "nonperiodic"`；
   - 曲线根部至发梢宽度通过一维数组线性插值。
3. **Katana 侧节点配置**：
   - 使用 `UsdIn` 加载毛发资产；
   - 检查 Katana 的 `Geometry.type` 是否自动识别为 `curves`；
   - 挂载专有毛发着色器 (如 `Arnold standard_hair` 或 `Karma Fur`)。

### 3.2 自动化曲线有效性核查脚本

```python
from pxr import Usd, UsdGeom

def validate_usd_hair_curves(stage_path: str):
    stage = Usd.Stage.Open(stage_path)
    curves = [p for p in stage.Traverse() if p.IsA(UsdGeom.BasisCurves)]
    
    if not curves:
        print("[ERROR] 场景中未发现合规的 UsdGeomBasisCurves 图元！")
        return False
        
    for c_prim in curves:
        c = UsdGeom.BasisCurves(c_prim)
        curve_type = c.GetTypeAttr().Get()
        basis = c.GetBasisAttr().Get()
        counts = c.GetCurveVertexCountsAttr().Get()
        points = c.GetPointsAttr().Get()
        widths = c.GetWidthsAttr().Get()
        
        print(f"[OK] 毛发图元: {c_prim.GetPath()}")
        print(f"  曲线类型: {curve_type}, 基底: {basis}")
        print(f"  曲线总数: {len(counts) if counts else 0}, 控制点总数: {len(points) if points else 0}")
        
        if not widths:
            print(f"  [CRITICAL] 缺少 widths 宽度属性！渲染将显示异常。")
    return True
```
