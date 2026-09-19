# 10. OpenUSD Asset Resolver 2.0 与 Context Pinning 依赖树治理

## 1. 痛点现象 (Symptoms)
跨多个制作部门、异地工作室或多分支 Git 协作时：
- USD 文件中硬编码了本地操作系统的绝对路径（如 `D:/Projects/...` 或 `/mnt/nas/...`），发送给合作方或提交农场时出现全盘红色断链；
- 上游资产更新发布了 `v003`，导致正在出成片的几百个历史镜头未经测试自动静默升级，发生材质穿帮或几何穿插事故。

---

## 2. 根因剖析 (Root Cause)

1. **依赖硬编码与操作系统耦合**：
   直接使用文件系统物理路径不仅缺乏平台可移植性，且导致 USD 场景无法在容器化云端（Docker/Kubernetes）渲染农场中迁移。
2. **缺少版本上下文锚定 (Context Pinning)**：
   OpenUSD 默认的 `ArDefaultResolver` 只做简单的文件路径拼接。若上游将资产发布为 `asset_latest.usd` 软链接，下游引用该链接的所有 Stage 都会失去**可复现性（Reproducibility）**。任何不可控的资产更新都会直接破坏历史镜头的确定性。

---

## 3. 规约与解决方案 (Solution)

### 3.1 资产解析中台契约
1. **统一采用 URI 契约**：
   禁止在 USD 文件中出现物理绝对路径，所有资产引用与材质贴图统一使用抽象 URI 格式：
   `asset://{domain}/{category}/{asset_name}/{version}/{asset_name}.usd`
2. **Context Pinning（显式上下文版本锚定）**：
   基于 **OpenUSD Ar 2.0 (Asset Resolver 2.0)** 规范，镜头在打开时必须加载专属的 `ArResolverContext`。该上下文内部存储一张完整的 `{asset_id: exact_version}` 哈希映射表（如 shot_001 强制锁定 chair=v002, table=v001），杜绝静默升级。

### 3.2 依赖树扫描与断链排查脚本

```python
from pxr import Usd, Sdf, Ar

def scan_stage_external_dependencies(stage_path: str):
    """递归遍历 Stage 的所有层级，抽取并校验全部外部依赖项"""
    stage = Usd.Stage.Open(stage_path)
    root_layer = stage.GetRootLayer()
    
    # 提取所有外部资产依赖
    dependencies = Sdf.Layer.GetExternalReferences(root_layer.identifier)
    print(f"=== Stage 外部资产依赖报告 ({len(dependencies)} 项) ===")
    
    resolver = Ar.GetResolver()
    for dep in dependencies:
        resolved_path = resolver.Resolve(dep)
        status = "[OK]" if resolved_path else "[MISSING / BROKEN]"
        print(f"  {status} 原声明: {dep}")
        if resolved_path:
            print(f"       -> 解析为: {resolved_path}")
```
