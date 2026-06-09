# hardware-lab

我的硬件学习仓 —— 每个子目录是一个独立的小项目。

## 子项目

| 项目 | 简介 | 状态 |
|---|---|---|
| **[sleepy-sensor/](sleepy-sensor/)** | ESP32-C3「树莓派 Zero 形态」低功耗温度计 / 开发板,可接墨水屏 HAT | v2 选型定稿待落地 |
| **[pisugar-clone/](pisugar-clone/)** | cleanroom 实现 PiSugar 3 I²C 协议的树莓派电源 / UPS HAT(协议兼容,非逆向) | 设计中 |

通用工作约定见 [CLAUDE.md](CLAUDE.md);各项目设计源头见各自 `docs/design.md`。

## 约定
- 硬件用 EasyEDA Pro(云端),通过「导出 → .epro」把工程快照提交进各项目 `hardware/` 做版本化。
- 选型前必查数据手册;手焊为主,有加热台可回流 QFN。
