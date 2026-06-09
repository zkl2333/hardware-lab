# CLAUDE.md — sleepy-sensor

ESP32-C3「树莓派 Zero 形态」低功耗开发板 / 温度计,可接墨水屏 HAT。

> 通用工作约定见仓库顶层 [../CLAUDE.md](../CLAUDE.md);设计源头:[docs/design.md](docs/design.md)。

## 现状(2026-06)
- 路线图:**v1**(弱 LDO 致重启,换 HT7833 救活验证最小系统)→ **v2**(BQ24074 一体板电源,选型定稿待落地)→ **v3**(树莓派式分体可堆叠)。
- **v3 的「电源 HAT」已独立成 cleanroom PiSugar 克隆**,见 [../pisugar-clone/](../pisugar-clone/);本项目 v3 聚焦「无电池主板」侧。
- 硬件版本化:EasyEDA「文件→导出→工程(.epro)」放 `hardware/` 提交。
