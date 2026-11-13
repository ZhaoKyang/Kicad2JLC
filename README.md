# Gerber to EasyEDA (立创EDA专业版) 转换工具使用指南

本脚本基于Nitmi开发的GhostPCB-Web项目，感谢原作者。

`gerber_convert.py` 是一个轻量级、零依赖的自动化转换工具，旨在将 **KiCad**、**Altium Designer** 等 EDA 软件导出的 Gerber（Gerber X2 / RS-274X 4.6 格式）无缝转换为 **立创EDA专业版（EasyEDA Pro / Gerber 1）** 标准格式。

工具内置了立创EDA官方的**防伪数字水印动态 MD5 签名生成器**，支持**2层、4层、6层及多层板**，支持**顶底层双面贴片元器件**，转换后的文件可 100% 通过立创EDA在线版/客户端的来源检测与哈希验证，直接用于在线查看、编辑或嘉立创生产下单。

---

## 目录

- [核心特性](#核心特性)
- [环境要求](#环境要求)
- [快速开始](#快速开始)
- [命令参数说明](#命令参数说明)
- [图层转换与命名对照表](#图层转换与命名对照表)
- [钻孔文件拆分规范](#钻孔文件拆分规范)
- [立创EDA导入步骤](#立创eda导入步骤)
- [自检报告说明](#自检报告说明)
- [常见问题 (FAQ)](#常见问题-faq)

---

## 核心特性

1. **立创EDA防伪哈希载体光圈自动生成**
   - 逆向实现了立创EDA官方的隐写校验算法。
   - 针对各层自动预留插槽（顶层/钻孔图层为 `D15`、底层/内电层为 `D16`、边框/底层丝印为 `D11`）。
   - 计算排除载体后全文本的 MD5 摘要，取后两位字节模 100 得到 2 位校验对（`pair`），动态写入载体光圈第 3、4 位小数中。
   - 原光圈自动顺延，保证绘图引用 0 次调用与 0 冲突，完美通过“隐藏哈希值检验”。

2. **全层数与双面贴片支持**
   - 原生支持 **单面板、双面板、4层板、6层板乃至多层板**（内电层自动泛化映射为 `Gerber_InnerLayer1.G1` ～ `Gerber_InnerLayerN.GN`）。
   - 原生支持**双面贴片元器件**：顶底层走线、阻焊、丝印、钢网开孔（`GTP` / `GBP`）全套转换。

3. **三重图层识别机制**
   - **优先度 1**：解析 KiCad 的 `.gbrjob` 配置文件中的 `FileFunction`。
   - **优先度 2**：读取 `.gbr` 文本前 1024 字节内的 `%TF.FileFunction` 元数据（即使在 KiCad 中将层重命名为 `GND`、`PWR` 也能精准识别）。
   - **优先度 3**：基于正则通配符识别常规文件名模式。

4. **Gerber 规范与语法清洗**
   - 剥离 KiCad X2 属性标签（`%TF.*`、`%TA.*`、`%TO.*`、`%TD*`）与生成器注释。
   - 将 10 参数带算术运算的圆角矩形宏（`AMRoundRect`）清洗重构为立创EDA 5 参数标准宏。
   - 规范换刀指令为 `G54D<num>*`。
   - 坐标系从 4.6（1nm 分辨率）降阶为立创标准的 4.5（10nm 分辨率）整数坐标，带平滑四舍五入。

5. **钻孔文件智能拆分与图层合成**
   - 将 KiCad 的单份 Excellon PTH 文件精准拆分为：
     - `Drill_PTH_Through.DRL`（全量金属化沉铜孔，含元件孔和过孔）
     - `Drill_PTH_Through_Via.DRL`（独立换层过孔，孔径 $\le 0.4\text{mm}$ 或具有 Via 属性）
   - 处理非金属化孔为 `Drill_NPTH_Through.DRL`。
   - 自动提取全部钻孔坐标与刀具尺寸，合成 `Gerber_DrillDrawingLayer.GDD` 钻孔符号图层。

6. **开箱即用，零依赖**
   - 纯 Python 3 标准库编写（仅使用 `os`, `sys`, `re`, `json`, `zipfile`, `hashlib`, `pathlib`）。
   - 无需运行 `pip install`，无需联网，离线即可运行。

---

## 环境要求

- **操作系统**：Windows、macOS、Linux 均可运行。
- **Python 版本**：Python 3.7 及以上。

---

## 快速开始

### 1. 默认参数运行

如果在当前目录下存在名为 `gerber2` 的输入文件夹，直接运行脚本即可：

```bash
python3 gerber_convert.py
```
- 默认输入目录：`gerber2`
- 默认输出目录：`test_gerber2`
- 自动生成生产级压缩包：`test_gerber2/Gerber_Converted_for_EasyEDA.zip`

### 2. 指定输入（目录或 ZIP 压缩包）与输出目录

你可以指定任意来源 Gerber 目录或 **直接指定待转换的 ZIP 压缩包**：

```bash
# 方式 A：指定文件夹
python3 gerber_convert.py ./my_kicad_project ./converted_for_easyeda

# 方式 B：直接传入 ZIP 压缩包（自动解压并转换内部所有图层）
python3 gerber_convert.py ./Gerbers.zip ./converted_for_easyeda
```

---

## 命令参数说明

```text
python3 gerber_convert.py [SRC] [DST]
```

| 参数 | 类型 | 默认值 | 说明 |
| :--- | :---: | :---: | :--- |
| `SRC` | 字符串 | 智能检测当前目录 / `gerber2` | 待转换的原始 Gerber 文件夹路径，或 `.zip` 压缩包路径 |
| `DST` | 字符串 | `./Gerber_Converted` / `test_gerber2` | 转换后文件及生产级 ZIP 压缩包的输出目录（自动创建） |

---

## 图层转换与命名对照表

转换工具会自动按照立创EDA专业版规范进行文件命名映射：

| 物理功能 | KiCad 典型导出文件名 | 转换后的立创标准文件名 | EasyEDA 层名 | 哈希载体 D 码 |
| :--- | :--- | :--- | :--- | :---: |
| **板框层** | `*-Edge_Cuts.gbr` | `Gerber_BoardOutlineLayer.GKO` | `BoardOutlineLayer` | **D11** |
| **顶层铜箔** | `*-F_Cu.gbr` | `Gerber_TopLayer.GTL` | `TopLayer` | **D15** |
| **底层铜箔** | `*-B_Cu.gbr` | `Gerber_BottomLayer.GBL` | `BottomLayer` | **D16** |
| **内电层 1** | `*-In1_Cu.gbr` / 自定义名 | `Gerber_InnerLayer1.G1` | `InnerLayer1` | **D16** |
| **内电层 2** | `*-In2_Cu.gbr` / 自定义名 | `Gerber_InnerLayer2.G2` | `InnerLayer2` | **D16** |
| **内电层 3** | `*-In3_Cu.gbr` / 自定义名 | `Gerber_InnerLayer3.G3` | `InnerLayer3` | **D16** |
| **内电层 4** | `*-In4_Cu.gbr` / 自定义名 | `Gerber_InnerLayer4.G4` | `InnerLayer4` | **D16** |
| **内电层 N** | `*-InN_Cu.gbr` / 自定义名 | `Gerber_InnerLayerN.GN` | `InnerLayerN` | **D16** |
| **顶层阻焊** | `*-F_Mask.gbr` | `Gerber_TopSolderMaskLayer.GTS` | `TopSolderMaskLayer` | **D15** |
| **底层阻焊** | `*-B_Mask.gbr` | `Gerber_BottomSolderMaskLayer.GBS` | `BottomSolderMaskLayer` | **D16** |
| **顶层丝印** | `*-F_Silkscreen.gbr` | `Gerber_TopSilkscreenLayer.GTO` | `TopSilkscreenLayer` | **D15** |
| **底层丝印** | `*-B_Silkscreen.gbr` | `Gerber_BottomSilkscreenLayer.GBO` | `BottomSilkscreenLayer` | **D11** |
| **顶层钢网** | `*-F_Paste.gbr` | `Gerber_TopPasteMaskLayer.GTP` | `TopPasteMaskLayer` | **D15** |
| **底层钢网** | `*-B_Paste.gbr` | `Gerber_BottomPasteMaskLayer.GBP` | `BottomPasteMaskLayer` | **D16** |
| **钻孔字符** | 自动根据钻孔列表合成 | `Gerber_DrillDrawingLayer.GDD` | `DrillDrawingLayer` | **D15** |

> **注**：对于单面贴片工程，若源文件中不存在底层贴片钢网（`*-B_Paste.gbr`），脚本不会生成多余的 `GBP` 文件；若为双面贴片工程，则会自动补齐双面钢网。

---

## 钻孔文件拆分规范

KiCad 默认将所有金属化孔导出到一个 `*-PTH.drl` 文件中，而立创EDA（Protel 格式规范）要求将全量孔与独立过孔分离：

| 生成的钻孔文件 | 格式 | 说明 |
| :--- | :---: | :--- |
| `Drill_PTH_Through.DRL` | Excellon 2:5 Metric | 包含全部沉铜孔（插件焊盘孔 + 换层过孔） |
| `Drill_PTH_Through_Via.DRL` | Excellon 2:5 Metric | 仅包含换层过孔（孔径 $\le 0.4\text{mm}$ 或标记为 Via） |
| `Drill_NPTH_Through.DRL` | Excellon 2:5 Metric | 包含非沉铜孔（螺丝固定孔、定位孔，若无则不生成） |

---

## 立创EDA导入步骤

转换完成后，会在输出目录生成 `Gerber_Converted_for_EasyEDA.zip`。

### 方式 A：立创EDA 在线版 / 专业版导入

1. 打开浏览器访问 [立创EDA专业版在线版](https://pro.lceda.cn/editor) 或立创EDA专业版桌面客户端。
2. 点击顶部菜单栏：`文件` -> `打开` -> `Gerber`（或 `文件` -> `导入` -> `Gerber`）。
3. 选择生成的 `Gerber_Converted_for_EasyEDA.zip`。
4. 此时立创EDA将正确识别所有图层颜色、坐标与钻孔，不会弹出任何未识别或解析错误警告。

### 方式 B：直接用于嘉立创 PCB 下单

1. 登录嘉立创客户下单系统。
2. 在上传附件处直接上传生成的 `Gerber_Converted_for_EasyEDA.zip`。
3. 系统将自动完成计价与 DFM 自动解析，各项工艺参数（层数、孔径、外形尺寸）均可精准识别。

---

## 自检报告说明

每次执行转换完成后，脚本会在终端自动打印自检验证报告：

```text
========== 立创EDA防伪哈希载体自检报告 ==========
  [文件] Gerber_BoardOutlineLayer.GKO     | 载体: D11 (C,0.1833) | 校验: ✅ 匹配通过 (嵌入值=33, 计算值=33)
  [文件] Gerber_BottomLayer.GBL           | 载体: D16 (C,1.5207) | 校验: ✅ 匹配通过 (嵌入值=07, 计算值=07)
  [文件] Gerber_BottomPasteMaskLayer.GBP  | 载体: D16 (R,1.0718) | 校验: ✅ 匹配通过 (嵌入值=18, 计算值=18)
  [文件] Gerber_BottomSilkscreenLayer.GBO | 载体: D11 (C,0.3219) | 校验: ✅ 匹配通过 (嵌入值=19, 计算值=19)
  [文件] Gerber_BottomSolderMaskLayer.GBS | 载体: D16 (C,2.1019) | 校验: ✅ 匹配通过 (嵌入值=19, 计算值=19)
  [文件] Gerber_DrillDrawingLayer.GDD     | 载体: D15 (C,0.2099) | 校验: ✅ 匹配通过 (嵌入值=99, 计算值=99)
  [文件] Gerber_TopLayer.GTL              | 载体: D15 (R,0.9152) | 校验: ✅ 匹配通过 (嵌入值=52, 计算值=52)
  [文件] Gerber_TopPasteMaskLayer.GTP     | 载体: D15 (R,1.0739) | 校验: ✅ 匹配通过 (嵌入值=39, 计算值=39)
  [文件] Gerber_TopSilkscreenLayer.GTO    | 载体: D15 (C,0.4056) | 校验: ✅ 匹配通过 (嵌入值=56, 计算值=56)
  [文件] Gerber_TopSolderMaskLayer.GTS    | 载体: D15 (C,2.5059) | 校验: ✅ 匹配通过 (嵌入值=59, 计算值=59)
>> 隐藏哈希值检验: ✅ 100% 全部图层载体校验通过！可被立创EDA官方/在线版无阻打开！
```

- **嵌入值**：光圈尺寸小数点后第 3、4 位的特征数字。
- **计算值**：脚本模拟立创EDA官方检验器对剔除载体后的文件实时计算的 MD5 尾部哈希。
- 只要全显示 `✅ 匹配通过`，即代表可 100% 模拟立创EDA原生导出的出厂状态。

---

## 常见问题 (FAQ)

### Q1: 转换出来的文件为什么比 KiCad 原文件少了几行？

KiCad 在导出 Gerber 时包含大量的非标准属性注释（如 `%TF.GenerationSoftware...`、`%TA.AperFunction...`），这些在 RS-274X 规范中属于 Gerber X2 扩展标签，较老的或国内 EDA 软件并不支持。转换工具剥离了这些多余的标签，并统一写入了立创EDA官方的 `G04 EasyEDA Pro...` 规范文件头，完全保留了全部真实的几何图形与电气走线。

### Q2: 为什么我的工程转换后没有生成 `Gerber_BottomPasteMaskLayer.GBP`？

如果你的电路板是纯单面贴片设计（只有顶层有贴片元件，底层没有任何 SMD 焊盘），KiCad 就不会导出底层的锡膏层文件（`*-B_Paste.gbr`）。工厂制造时不需要刷底层锡膏，因此不生成 `GBP` 是完全正确的。如果底层放置了贴片元件，脚本会自动生成 `GBP`。

### Q3: 为什么没有看到 `FlyingProbeTesting.json`？

`FlyingProbeTesting.json` 是立创EDA内部的飞针测试辅助网表，由原理图/PCB工程直接导出。
PCB 制造工厂在缺少该 JSON 文件时，CAM 工业软件（Genesis / InCAM）会自动从 Gerber 的铜箔走线与焊盘中反向提取电气网络（Netlist Extraction）进行飞针测试，**完全不影响任何物理制造与电气测试**。

### Q4: 提示 `[文件] Gerber_DrillDrawingLayer.GDD | 校验: ⚠️ 未找到匹配的哈希载体光圈` 是怎么回事？

这通常由以下两种原因导致：
1. **工作目录与参数未匹配**：在终端（如 `D:\gerber2>`）直接输入 `gerber_convert.py` 未带参数时，若当前目录下没有名为 `gerber2` 的子目录，脚本可能只识别到了部分残留或单独放置的钻孔文件（如 `GC2SD_2.0-PTH.drl`），而没有识别到铜皮图形层。推荐在命令行显式指定参数：
   ```bash
   # 转换当前目录下的所有 Gerber 文件到 output 目录
   python gerber_convert.py . output
   ```
2. **Windows 平台换行符自动转换（CRLF）**：在 Windows 系统下，Python 写入文本文件时容易将 Unix 换行符 `\n` 自动转换成 Windows 默认换行符 `\r\n`，导致文件实际写入磁盘后的 MD5 与生成时光标计算的 MD5 不一致。当前最新版本已在所有文件写入处强制加入 `newline='\n'`，彻底杜绝了平台换行符导致的哈希校验失效问题。
