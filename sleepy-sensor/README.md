# sleepy-sensor

ESP32-C3 的「树莓派 Zero 形态」低功耗开发板 —— 兼做温度计（SHT40），可接树莓派墨水屏 HAT。

- **主控**：ESP32-C3-WROOM-02（原生 USB，无需 CP2102 转串口）
- **外形**：树莓派 Zero 兼容（65×30mm，标准 M2.5 孔位 + 40P GPIO 排针），可用 Pi 外壳 / HAT
- **电池**：单节锂电 102540 / 1100mAh（裸电芯，板载保护）
- **工具**：EasyEDA Pro（硬件）+ 嘉立创打样

## 现状

**v1 已点亮并接入 Home Assistant**：板子运行 [ESPHome](firmware/)，每 60 秒上报电池电压与 WiFi 信号。两轮排障（弱 LDO 换 HT7833、WiFi 发射尖峰致欠压降功率解决）的完整过程见 [docs/design.md](docs/design.md) 第 2、6 节——对同类 charger-only 小板有参考价值。

## 路线图

| 版本 | 内容 | 状态 |
|---|---|---|
| **v1** | 已打样；TP4056 + HT7533 极简电源。弱 LDO（100mA）带不动 WiFi 峰值 → 换 HT7833 救活 | **已点亮**，跑 ESPHome |
| **v2** | 一体板：BQ24074 电源路径 + RT9080 LDO（Iq 2µA）+ 裸电芯保护 + BOOT 按键 + 排针 SPI 映射 | 选型定稿，待落地 |
| **v3** | 无电池主板（排针 5V 取电，全手焊）；电源侧由 [pisugar-clone](../pisugar-clone/) 承担，不再自做电源 HAT | 构想 |

设计细节与选型理由见 **[docs/design.md](docs/design.md)**，参考资料见 **[docs/references.md](docs/references.md)**。

## 目录结构

```
sleepy-sensor/
├── docs/         设计文档(design.md)、参考资料、数据手册
├── hardware/     v1 原理图 PDF、EasyEDA 工程快照(.epro)
└── firmware/     ESPHome 配置(电压上报 Home Assistant)
```

## 硬件版本管理

EasyEDA Pro 工程在云端；每到里程碑用「文件 → 导出 → 工程(.epro)」把快照提交进 `hardware/`，对硬件设计做版本化与回滚。
