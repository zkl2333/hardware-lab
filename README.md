# hardware-lab

个人硬件学习仓 —— 每个子目录是一个独立的小项目，**文档驱动**：每个项目的决策、选型与理由都沉淀在各自的 `docs/design.md`，改决策就改文档并提交，git 历史即设计演进史。

## 子项目

| 项目 | 简介 | 状态 |
|---|---|---|
| **[sleepy-sensor/](sleepy-sensor/)** | ESP32-C3「树莓派 Zero 形态」低功耗温度计 / 开发板，可接墨水屏 HAT | **v1 已点亮**（ESPHome 上报电压到 Home Assistant）；v2 选型定稿待落地 |
| **[pisugar-clone/](pisugar-clone/)** | cleanroom 实现 PiSugar I²C 协议的树莓派电源 / UPS HAT（按公开协议重新实现，非逆向） | Phase 1（PiSugar 2 兼容）选型定稿，待画板 |

## 仓库怎么读

- `<项目>/docs/design.md` —— **设计源头**：架构、选型对比、踩坑与修复全记录，全部带理由
- `<项目>/docs/references.md` —— 数据手册与参考项目
- `<项目>/hardware/` —— 原理图 PDF 与 EasyEDA 工程快照（.epro）
- `<项目>/firmware/` —— 固件 / 配置

## 工作方式

- 硬件用 EasyEDA Pro（云端），里程碑时「导出 → .epro」提交进 `hardware/` 做版本化
- **选型前必查官方数据手册**核实电流能力 / 封装 / Iq / 压降（v1 曾因漏查 LDO 电流能力整板重启报废，教训记录在 sleepy-sensor 设计文档）
- 手焊为主（0805 起步），有加热台可整板回流 QFN
