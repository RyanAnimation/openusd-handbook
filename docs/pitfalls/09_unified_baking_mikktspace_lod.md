# 09. 统一烘焙 MikkTSpace 切线空间与 LOD 分级生成规约

## 1. 痛点现象 (Symptoms)
高精影视资产（高模）烘焙法线贴图并生成多级细节层次 (LOD0~LOD3) 模型导入虚幻或实时视口时：
- 模型硬表面出现不可调和的黑色三角阴影、光影扭曲；
- 换一个 DCC 或烘焙软件（如从 Houdini 转入 Maya 或 Substance），光影破损断裂；
- LOD 切换时（Popping），几何轮廓发生剧烈形变跳跃。

---

## 2. 根因剖析 (Root Cause)

1. **切线空间计算算法不一致**：
   法线贴图是基于模型顶点法线与切线（Tangent/Bitangent）计算的偏移量。如果在烘焙时使用的是传统的平均角切线，而实时引擎（Unreal、Unity、USDView、Karma）在运行时使用的是 **MikkTSpace**（Morten Mikkelsen 切线空间标准算法），两者切线微小的角度偏差会导致表面法线评估出反向光影，形成无法消除的黑斑黑边。
2. **LOD 减面缺乏保形硬约束**：
   传统的简单减面器（Decimator）只追求多边形数量达标，忽视了高模曲面特征线。若没有引入**曲面豪斯多夫距离 (Hausdorff Distance)** 误差控制，关键轮廓处的曲面曲率会发生坍塌。

---

## 3. 规约与解决方案 (Solution)

### 3.1 统一烘焙管线基准
1. **SSOT 法线标准**：
   - 算法：**100% 强制统一为 MikkTSpace**；
   - 坐标系：**OpenGL 标准（X+ 右，Y+ 上，Z+ 外）**，若进入 DirectX 引擎仅对 G 通道取反；
2. **LOD 分级阶梯与误差门禁**：
   - **LOD0**：原始高精/管线资产 (100% 面数，豪斯多夫误差 $\epsilon = 0$)；
   - **LOD1**：次级视距 (面数 50%，最大豪斯多夫误差 $\le 0.05\%$ 资产包围盒对角线)；
   - **LOD2**：中远景 (面数 25%，最大豪斯多夫误差 $\le 0.15\%$)；
   - **LOD3**：超远景极低模 (面数 10%，最大豪斯多夫误差 $\le 0.35\%$)。

### 3.2 自动化 MikkTSpace 与切线检查

```python
from pxr import Usd, UsdGeom

def inspect_mesh_primvars(stage_path: str):
    stage = Usd.Stage.Open(stage_path)
    for prim in stage.Traverse():
        if prim.IsA(UsdGeom.Mesh):
            mesh = UsdGeom.Mesh(prim)
            primvars_api = UsdGeom.PrimvarsAPI(prim)
            
            has_normals = primvars_api.HasPrimvar("normals")
            has_tangents = primvars_api.HasPrimvar("tangents")
            
            print(f"Mesh: {prim.GetPath()} | Normals: {has_normals} | Tangents: {has_tangents}")
            if not has_tangents:
                print(f"  [RECOMMEND] 建议在导出时显式烘焙 MikkTSpace 切线，避免实时引擎重建开销。")
```
