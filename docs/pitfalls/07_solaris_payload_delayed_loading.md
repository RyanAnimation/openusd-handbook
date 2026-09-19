# 07. Solaris Payload 延迟流式加载与 GPU 显存保护

## 1. 痛点现象 (Symptoms)
在 Houdini Solaris 中打开包含成千上万植被、建筑与高精数字道具的大规模城市/森林场景时：
- 打开 USD 场景耗时数分钟，内存直接飙升至 64GB+；
- 切换到 Karma XPU 渲染时，显卡显存直接打满崩溃（CUDA Out of Memory, OOM）；
- 视口导航极其卡顿，帧率降至 1~2 FPS。

---

## 2. 根因剖析 (Root Cause)

**References 与 Payloads 的本质差异**：
- **References (引用)**：属于**强加载组合弧**。一旦打开 Stage，无论视口中是否可见，USD 底层解析器必须无条件将目标文件的几何图元、拓扑数组与材质全部读取到内存中；
- **Payloads (负载)**：属于**可选延迟流式加载弧 (Optional / Deferred Loading)**。在创建 Stage 时，可以通过 `Usd.Stage.OpenMasked()` 或 `SetLoadRules()` 显式指定**仅加载包围盒 (Bounding Box) 或代理模型**，几何高模完全不装载进内存。

如果所有资产全量使用 `references`，或者没有为高面数模型（面数 > 50,000）剥离独立的 Payload 文件，大场景组装必然导致显存瞬时爆满。

---

## 3. 规约与解决方案 (Solution)

### 3.1 工业管线显存分流原则
1. **面数门禁**：任何面数超过 **50,000 面**的多边形资产，一律禁止直接作为 Reference 引用，必须封装为带有 Payload 的组件资产；
2. **两阶段装配分离 (Assembly Decoupling)**：
   - 装配总场景 (`shot_layout.usd`) 只引入带有 Bounding Box 的 Payload 锚点；
   - 只有当相机视锥体 (Frustum) 覆盖该资产且距离满足阈值时，才触发按需装载；
3. **Karma 渲染分层**：
   - 调试阶段：全局设置 `Load Rule = None` 或 `Load Only Proxies`；
   - 最终出图：仅针对光照影响区激活 `Payloads`。

### 3.2 自动化 Payload 负载调度脚本

```python
from pxr import Usd

def stream_scene_with_load_rules(stage_path: str, active_camera_path: str):
    """基于自定义加载规则开启 Stage，避免整场暴力全开"""
    # 步骤 1: 默认全部不装载 (LoadNone)
    rules = Usd.StageLoadRules.LoadNone()
    
    # 步骤 2: 指定仅装载核心主角与相机视口内资产
    rules.AddRule("/World/Hero_Character", Usd.StageLoadRules.AllRule)
    rules.AddRule("/World/Environment/ground", Usd.StageLoadRules.AllRule)
    
    # 以受控规则打开 Stage
    stage = Usd.Stage.Open(stage_path, load=Usd.Stage.LoadNone)
    stage.SetLoadRules(rules)
    
    print("=== Stage 受控加载就绪 ===")
    print(f"当前装载规则: {stage.GetLoadRules()}")
    # 内存占用可降低 80%~95%
```
