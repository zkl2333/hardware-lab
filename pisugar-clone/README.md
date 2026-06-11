# pisugar-clone

Cleanroom 实现 **PiSugar I²C 协议兼容**的树莓派电源 / UPS HAT —— 从公开协议规范与开源客户端（协议消费方）重新实现，**不逆向原厂 PCB / 固件**。目标：让开源的 [pisugar-power-manager](https://github.com/PiSugar/pisugar-power-manager-rs) 和真树莓派把这块板当成正品使用。

## 分阶段路线

| 阶段 | 兼容目标 | 思路 | 状态 |
|---|---|---|---|
| **Phase 1** | PiSugar 2 | 直接用真 IP5209（I²C @0x75）+ SD3078 RTC（@0x32），**纯硬件、几乎无固件**——客户端读的本来就是这两颗芯片 | **选型定稿，待画板** |
| **Phase 2** | PiSugar 3 | MCU 模拟自定义寄存器表（@0x57）：I²C 从机 + 写保护 + RTC/BCD + 闹钟唤醒 + 电量曲线 + 按键状态机 | 规划中 |

两阶段共同要求：完全协议兼容 + 5V boost 输出（能真正给树莓派供电）；机械结构完全兼容 PiSugar 2（磁吸 pogo pin 形态）。低功耗是加分项不是硬目标。

## 设计文档

架构、IP5209 家族选型对比、寄存器兼容性分析、BOM 与机械结构全部在 **[docs/design.md](docs/design.md)**；数据手册归档在 [docs/datasheets/](docs/datasheets/)。

## 合法性说明

实现公开 I²C 协议属 cleanroom 工程惯例：参考对象仅限协议**消费方**（开源客户端源码）与芯片原厂数据手册，不接触、不反汇编原厂 MCU 固件二进制。
