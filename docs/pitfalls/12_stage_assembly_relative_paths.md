# 12. 跨 DCC Stage 组装根锚定与相对路径规约

## 1. 痛点现象 (Symptoms)
- 将本地打包好的 USD 工程压缩发送给异地协作团队后，对方解压打开只有空空如也的根节点；
- 切换工作机或盘符（例如从 `E:/` 迁移至 `Z:/`），所有贴图与资产引用全部断开；
- 在 Maya 中保存的 USD 装配文件，在 Solaris 中因路径斜杠与根锚定混乱而无法重载。

---

## 2. 根因剖析 (Root Cause)

1. **绝对路径与 Windows 盘符污染**：
   Windows 环境特有的驱动器盘符（如 `C:\`、`D:\`）与反斜杠 `\` 严重违反跨平台规范。在 Linux 农场节点上，带有 Windows 盘符的路径会被当成无效的非法字符。
2. **缺乏锚定点 (Anchor Layer)**：
   若使用相对路径 `../geo/asset.usd`，OpenUSD 默认以声明该引用的 Layer 文件自身所在物理目录为锚点。如果装配文件层级结构不固定，层级跳变会导致向上回溯层级错位。

---

## 3. 规约与解决方案 (Solution)

### 3.1 工业组装路径铁律
1. **100% 相对路径与正斜杠 (Unix 风格)**：
   全量引用严禁使用绝对路径，路径分隔符统一使用正斜杠 `/`；
2. **根锚定工程结构 (Root-Anchored Architecture)**：
   推荐标准项目目录规范：
   ```
   Project_Root/
       assets/
           props/chair/chair.usd
       shots/
           shot_001/shot_001.usd  (引用相对路径: @../../assets/props/chair/chair.usd@)
   ```
3. **自动化路径规范化校验脚本**：

```python
import os
from pxr import Sdf

def check_for_absolute_paths_in_layer(layer_path: str):
    """检查单层文件内部是否存在硬编码绝对路径"""
    layer = Sdf.Layer.FindOrOpen(layer_path)
    if not layer:
        return
        
    for asset_path in layer.GetExternalReferences():
        if os.path.isabs(asset_path) or ":" in asset_path:
            print(f"[ERROR] 发现硬编码绝对路径污染: {asset_path} in {layer_path}")
        else:
            print(f"[OK] 相对路径合规: {asset_path}")
```
