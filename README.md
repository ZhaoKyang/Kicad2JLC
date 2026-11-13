# Gerber to EasyEDA 转换工具

`gerber_convert.py` 是一个轻量级、开源的自动化 Gerber 格式转换工具，旨在将 **KiCad**、**Altium Designer** 等软件导出的 Gerber（RS-274X / Gerber X2）文件转换为 **立创EDA专业版（EasyEDA Pro）** 所兼容的标准格式。

本项目专注于解决跨 EDA 平台格式转换中的常见问题，包括图层命名映射、坐标精度转换、圆角矩形光圈宏重构、Excellon 钻孔文件拆分（全量孔与独立过孔分离）以及格式完整性校验。

---

## 目录

- [主要特性](#主要特性)
- [环境依赖](#环境依赖)
- [快速开始](#快速开始)
- [命令参数说明](#命令参数说明)
- [图层命名与映射对照表](#图层命名与映射对照表)
- [钻孔文件拆分规范](#钻孔文件拆分规范)
- [导入指南](#导入指南)
- [自检报告说明](#自检报告说明)
- [常见问题 (FAQ)](#常见问题-faq)
- [免责声明](#免责声明)
- [开源协议](#开源协议)

---

## 主要特性

1. **图层格式规范对齐与完整性校验**
   - 严格遵循立创EDA专业版图层文件命名与格式结构规范。
   - 自动生成符合规范的预留光圈参数与格式校验标记，保证导入立创EDA时不产生解析错误或图元丢失。
   - 自动检测并顺延已有光圈编号，避免图元编号冲突。

2. **多层板与双面贴片支持**
   - 支持单面板、双面板、4层板、6层板等多层板设计。
   - 完整支持顶底层双面贴片元件相关的走线、阻焊开窗、丝印及钢网（`GTP` / `GBP`）图层。

3. **智能图层识别**
   - 支持读取 KiCad `.gbrjob` 项目配置文件进行图层识别。
   - 支持解析 Gerber 文件头内的 `FileFunction` 元数据。
   - 兼容标准 Protel / Altium 文件后缀（`.gtl`, `.gbl`, `.gts`, `.gbs`, `.gto`, `.gbo`, `.gtp`, `.gbp`, `.gm1` 等）。

4. **Gerber 语法规范化**
   - 清理多余属性标签与调试注释，统一写入标准格式头。
   - 将 KiCad 的 10 参数 `AMRoundRect` 光圈宏自动转换为立创EDA所支持的标准 5 参数格式。
   - 统一换刀指令语法（`G54D<num>*`），并将 4.6 坐标平滑转换为立创EDA标准 4.5 整数坐标。

5. **钻孔文件拆分与符号图层合成**
   - 自动将 Excellon 钻孔文件拆分为：
     - `Drill_PTH_Through.DRL`（全量金属化沉铜孔及槽孔）
     - `Drill_PTH_Through_Via.DRL`（独立换层过孔）
     - `Drill_NPTH_Through.DRL`（非金属化安装孔/定位孔）
   - 完整保留椭圆形/长条形铣槽指令（`G00`, `G01`, `M15`, `M16`, `G85`）。
   - 自动提取全部钻孔坐标与刀具尺寸，合成 `Gerber_DrillDrawingLayer.GDD` 钻孔符号图层。

6. **轻量便携，无第三方依赖**
   - 基于 Python 3 标准库开发，无需安装任何额外包。
   - 支持直接传入文件夹路径或 `.zip` 压缩包。

---

## 环境依赖

- **操作系统**：Windows / Linux / macOS
- **运行环境**：Python 3.7 及以上版本

---

## 快速开始

### 1. 命令行基本用法

```bash
# 转换指定目录中的 Gerber 文件
python3 gerber_convert.py ./my_kicad_project ./converted_output

# 或直接转换 ZIP 压缩包
python3 gerber_convert.py ./my_project.zip ./converted_output
```

转换成功后，将在目标目录下生成标准文件以及 `Gerber_Converted_for_EasyEDA.zip` 压缩包。

### 2. 默认模式

在包含 Gerber 文件的目录下直接执行：
```bash
python3 gerber_convert.py
```
脚本会自动识别当前目录下的图层文件并输出到 `./Gerber_Converted` 文件夹。

---

## 命令参数说明

```text
python3 gerber_convert.py [SRC] [DST]
```

| 参数 | 类型 | 默认值 | 说明 |
| :--- | :---: | :---: | :--- |
| `SRC` | 字符串 | 智能检测当前目录 / `gerber2` | 来源目录或待转换的 `.zip` 压缩包路径 |
| `DST` | 字符串 | `./Gerber_Converted` / `test_gerber2` | 转换后文件与 ZIP 包的输出路径 |

---

## 图层命名与映射对照表

| 物理功能 | 常见输入文件名 | 转换后的立创标准文件名 | EasyEDA 层名 |
| :--- | :--- | :--- | :--- |
| **板框层** | `*-Edge_Cuts.gbr`, `*.gm1`, `*.gko` | `Gerber_BoardOutlineLayer.GKO` | `BoardOutlineLayer` |
| **顶层铜箔** | `*-F_Cu.gbr`, `*.gtl` | `Gerber_TopLayer.GTL` | `TopLayer` |
| **底层铜箔** | `*-B_Cu.gbr`, `*.gbl` | `Gerber_BottomLayer.GBL` | `BottomLayer` |
| **内电层 1** | `*-In1_Cu.gbr`, `*.g1` | `Gerber_InnerLayer1.G1` | `InnerLayer1` |
| **内电层 2** | `*-In2_Cu.gbr`, `*.g2` | `Gerber_InnerLayer2.G2` | `InnerLayer2` |
| **内电层 N** | `*-InN_Cu.gbr`, `*.gN` | `Gerber_InnerLayerN.GN` | `InnerLayerN` |
| **顶层阻焊** | `*-F_Mask.gbr`, `*.gts` | `Gerber_TopSolderMaskLayer.GTS` | `TopSolderMaskLayer` |
| **底层阻焊** | `*-B_Mask.gbr`, `*.gbs` | `Gerber_BottomSolderMaskLayer.GBS` | `BottomSolderMaskLayer` |
| **顶层丝印** | `*-F_Silkscreen.gbr`, `*.gto` | `Gerber_TopSilkscreenLayer.GTO` | `TopSilkscreenLayer` |
| **底层丝印** | `*-B_Silkscreen.gbr`, `*.gbo` | `Gerber_BottomSilkscreenLayer.GBO` | `BottomSilkscreenLayer` |
| **顶层钢网** | `*-F_Paste.gbr`, `*.gtp` | `Gerber_TopPasteMaskLayer.GTP` | `TopPasteMaskLayer` |
| **底层钢网** | `*-B_Paste.gbr`, `*.gbp` | `Gerber_BottomPasteMaskLayer.GBP` | `BottomPasteMaskLayer` |
| **钻孔字符** | 自动根据钻孔列表合成 | `Gerber_DrillDrawingLayer.GDD` | `DrillDrawingLayer` |

---

## 钻孔文件拆分规范

| 钻孔文件 | 格式 | 说明 |
| :--- | :---: | :--- |
| `Drill_PTH_Through.DRL` | Excellon 2:5 Metric | 包含全部沉铜通孔与槽孔（焊盘孔 + 换层过孔） |
| `Drill_PTH_Through_Via.DRL` | Excellon 2:5 Metric | 仅包含换层过孔（孔径 $\le 0.4\text{mm}$ 或元数据标记为 Via） |
| `Drill_NPTH_Through.DRL` | Excellon 2:5 Metric | 包含非沉铜孔（定位孔、机械安装孔，若无则不生成） |

---

## 导入指南

1. 打开 [立创EDA专业版](https://pro.lceda.cn/editor)（在线网页版或客户端）。
2. 在顶部菜单点击：`文件` -> `打开` -> `Gerber`（或 `导入` -> `Gerber`）。
3. 选择转换生成的 `Gerber_Converted_for_EasyEDA.zip` 压缩包。
4. 导入后即可直接查看各层布线、阻焊、丝印及孔位。

---

## 自检报告说明

转换流程结束后，工具会自动对打包后的图层文件进行格式完整性检查并输出自检日志：

```text
========== 格式兼容性与校验光圈自检报告 ==========
  [文件] Gerber_BoardOutlineLayer.GKO     | 校验: ✅ 匹配通过
  [文件] Gerber_BottomLayer.GBL           | 校验: ✅ 匹配通过
  [文件] Gerber_BottomPasteMaskLayer.GBP  | 校验: ✅ 匹配通过
  [文件] Gerber_BottomSilkscreenLayer.GBO | 校验: ✅ 匹配通过
  [文件] Gerber_BottomSolderMaskLayer.GBS | 校验: ✅ 匹配通过
  [文件] Gerber_DrillDrawingLayer.GDD     | 校验: ✅ 匹配通过
  [文件] Gerber_TopLayer.GTL              | 校验: ✅ 匹配通过
  [文件] Gerber_TopPasteMaskLayer.GTP     | 校验: ✅ 匹配通过
  [文件] Gerber_TopSilkscreenLayer.GTO    | 校验: ✅ 匹配通过
  [文件] Gerber_TopSolderMaskLayer.GTS    | 校验: ✅ 匹配通过
>> 图层完整性与校验光圈: ✅ 100% 全部图层校验通过！
```

全部显示匹配通过表明导出的 Gerber 语法与光圈定义完全符合立创EDA专业版的规范要求。

---

## 常见问题 (FAQ)

### Q1: 转换后的文件体积与原文件相比有所变化？
工具在转换过程中剔除了部分专用于 KiCad 的扩展注释，并将光圈宏与坐标格式重新标准化，保留了全部有效的物理图形与几何电气信息。

### Q2: 单面贴片工程未生成底层钢网文件（GBP）？
若原 PCB 仅顶层有贴片元件，KiCad 导出时不会产生底层锡膏文件（`*-B_Paste.gbr`）。此时不生成 `GBP` 符合实际生产要求；若底层存在贴片焊盘，工具会自动转换并保留该层。

### Q3: 为什么没有 `FlyingProbeTesting.json`？
该 JSON 文件属于立创EDA内部的电路网表辅助信息。常规 CAM 制造软件通常直接从 Gerber 铜箔与钻孔数据中提取网络进行飞针测试，缺少该文件不影响查看和生产解析。

---

## 免责声明

1. 本项目为开源技术研究与工程格式兼容工具，旨在协助工程师在不同 EDA 软件之间进行图纸格式转换与学习交流。
2. 本项目与立创EDA（EasyEDA）、嘉立创（JLCPCB）、KiCad、Altium 等商业公司或组织无任何官方从属或商业合作关系。文中涉及的所有商标名称均属于其合法持有者。
3. 用户在将转换后的文件用于实际打样或批量生产前，请务必使用专业 Gerber 查看器仔细核验图层走线、阻焊开窗与孔径尺寸。使用者需自行对最终制造结果负责。

---

## 开源协议

本项目采用 [MIT 许可证](LICENSE) 进行开源。您可以自由使用、修改和分发本项目代码，详情请参阅项目根目录下的 `LICENSE` 文件。
