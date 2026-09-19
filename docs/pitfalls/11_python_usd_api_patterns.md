# 11. Python USD API 核心算子与批量巡检高效模式

## 1. 痛点现象 (Symptoms)
编写 Python 脚本对超大规模 USD 场景进行批量资产质检或属性更新时：
- 脚本运行极其缓慢，处理一个包含几千个图元的 Stage 耗时数十分钟；
- 每次 `SetAttribute` 操作都会导致整个 Stage 产生级联重通知 (Recomposition Notice)，视口频繁假死卡顿；
- 内存持续泄漏，批量遍历多个 USD 文件后 Python 进程被系统 OOM 终止。

---

## 2. 根因剖析 (Root Cause)

1. **未关闭合成通知的单步更新**：
   在 OpenUSD 中，Stage 内部维护着极其复杂的依赖监听树。如果在循环体中连续执行 `attr.Set(val)`，USD 会在每个属性更新后立即触发全场景的重评估。
2. **混淆 Usd (合成态) 与 Sdf (底层规格态)**：
   若仅需修改某个特定 Layer 中的底层元数据或文字，却粗暴调用 `Usd.Stage.Open()` 加载完整的合成图元树，会付出巨大的场景遍历与内存展开开销。

---

## 3. 规约与解决方案 (Solution)

### 3.1 性能黄金准则
1. **使用 `SdfChangeBlock` 批量合并修改**：在循环执行批量属性注入或层级修改时，必须使用上下文管理器 `with Sdf.ChangeBlock():`，将成千上万次局部修改合并为一次通知派发。
2. **轻量只读场景优先使用 Sdf**：如果只需读取或修改单层的特定路径，直接使用 `Sdf.Layer.FindOrOpen()`，速度提升 **10x~50x**。

### 3.2 高性能批量更新代码范式

```python
from pxr import Usd, Sdf

def batch_update_properties_efficiently(stage: Usd.Stage):
    """高性能批量更新属性范式"""
    # 关键：开启 SdfChangeBlock 阻止实时级联通知
    with Sdf.ChangeBlock():
        for prim in stage.Traverse():
            if prim.IsA(Usd.ModelAPI):
                # 批量打标自定义元数据
                prim.SetCustomDataByKey("pipeline_audit:passed", True)
                prim.SetCustomDataByKey("pipeline_audit:timestamp", "2026-09-17")
                
    print("=== 批量更新完毕，通知已合并合并派发 ===")
```
