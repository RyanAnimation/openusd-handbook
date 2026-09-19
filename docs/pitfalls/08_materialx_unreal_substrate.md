# 08. MaterialX 与 Unreal Engine 5.5 Substrate 映射与黑边修复

## 1. 痛点现象 (Symptoms)
在 Houdini/Maya 中定义标准的 OpenPBR 材质，并通过 USD 资产导入 Unreal Engine 5.5 (开启 Substrate 材质框架) 时：
- 金属与半透明图层混合处出现异常黑色轮廓光晕（Mipmap 黑边）；
- 清漆层 (Clearcoat) 与次表面散射 (Subsurface) 强光反射泛白失真；
- 多材质层混合时，Shader 指令数成倍膨胀导致实时帧率暴跌。

---

## 2. 根因剖析 (Root Cause)

1. **Substrate 模块化 Slabs 与 OpenPBR 堆叠模型的差异**：
   OpenPBR 1.0 采用能量守恒的层级吸收模型（Base -> Specular -> Transmission -> Subsurface -> Coat -> Fuzz）。
   UE 5.5 Substrate 将着色模型拆分为 `Substrate Slab BSDF`，通过水平混合 (Horizontal Blend) 与垂直层叠 (Vertical Layer) 组装。如果从 MaterialX 导出的着色网络直接粗暴映射为单一 Slab，折射率 (IOR) 和 F0 反射率在不同层之间的权重传递会发生折损。
2. **Mipmap 纹理边缘未膨胀 (Dilation Issue)**：
   在 UV 展开接缝处，若纹理贴图未执行 >=32px 的边缘像素膨胀（Padding/Dilation），实时引擎在远距离 Mipmap 下采样时会采样到黑色背景像素，导致金属或明亮边缘出现死黑轮廓。

---

## 3. 规约与解决方案 (Solution)

### 3.1 跨平台材质映射中台标准
1. **F0 / IOR 双向闭环公式**：
   在 MaterialX 与 Substrate 互通时，严格采用物理公式换算：
   $$F_0 = \left(\frac{\text{IOR} - 1}{\text{IOR} + 1}\right)^2$$
2. **烘焙贴图边缘膨胀规范**：
   所有输出给实时引擎的 PBR 贴图（ORM 通道打包），必须经过至少 **32 像素**的 UV 边缘扩散算法，彻底消除 Mipmap 黑色伪影。
3. **Coat 独立垂直分层**：
   清漆层严禁在单个 Slab 内通过标量混合，必须使用 `Substrate Vertical Layer` 单独叠加一层薄介质 Slab，保持微表面高光的独立反射。

---

## 4. 工业生产最佳实践检查表
- 检查贴图 ORM (Occlusion, Roughness, Metallic) 是否在导出前确认无预乘 Alpha；
- 检查法线贴图切线空间是否严格遵循 MikkTSpace 与 OpenGL (Y+) 标准。
