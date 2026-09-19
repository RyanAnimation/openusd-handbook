# 生产级开源工具链 (OpenUSD Production Tools)

为配合《OpenUSD 工业管线避坑手册》中提及的各项规范门禁，本项目开源了轻量级、无第三方重依赖的 Python 校验与清理工具集。

👉 **GitHub 官方仓库**：[`ryan-cg/openusd-production-tools`](https://github.com/ryan-cg/openusd-production-tools)

---

## 核心工具模块

### 1. 孤立覆写清理器 (`usd_over_cleaner.py`)
- **对应词条**：[02. SdfPath 路径与 Dangling Over 孤立覆写清洗](../pitfalls/02_dangling_over_cleanup.md)
- **功能**：自动递归扫描 SdfLayer，定位并剔除所有未在合成 Stage 中定义的无主 `over` 声明，显著缩减文件体积。

### 2. 资产负载优化器 (`usd_payload_optimizer.py`)
- **对应词条**：[07. Solaris Payload 延迟流式加载与 GPU 显存保护](../pitfalls/07_solaris_payload_delayed_loading.md)
- **功能**：针对面数超过 50,000 的大资产，自动重构为带有 Payload 外部引用的标准资产结构，并生成包围盒缓存。

### 3. 依赖树与 URI 门禁校验器 (`usd_resolver_validator.py`)
- **对应词条**：[10. OpenUSD Asset Resolver 2.0 与 Context Pinning](../pitfalls/10_asset_resolver_context_pinning.md)
- **功能**：拦截 Windows 绝对路径与死链，确保资产符合 `asset://` 命名规范。

---

## 安装与快速上手

```bash
git clone https://github.com/ryan-cg/openusd-production-tools.git
cd openusd-production-tools
pip install .
```

### 命令行用法示例
```bash
# 清理镜头文件中的孤立覆写
python -m openusd_tools.over_cleaner --input shot_lighting.usda --output shot_clean.usda

# 检查资产依赖合法性
python -m openusd_tools.resolver_validator --stage shot_assembly.usd
```
