# CLAUDE.md — pisugar-clone

Cleanroom 实现 **PiSugar 3 的 I²C 协议**的树莓派电源 / UPS HAT:从公开规范 + 开源客户端重新实现,**不逆向其 PCB/固件**。目标——让开源 PiSugar Power Manager 和真树莓派把本板当成正品 PiSugar 3。

> 通用工作约定见仓库顶层 [../CLAUDE.md](../CLAUDE.md);设计源头:[docs/design.md](docs/design.md)。

## 定位(已拍板)
- **完全协议兼容** + **5V 输出**(带 boost,能给真树莓派供电)。深睡低功耗是加分项,不强求。
- **固件是工作量大头**:I²C 从机 + 写保护门控 + 整张寄存器表 + RTC/BCD + 闹钟唤醒 + 电量曲线 + 按键状态机 + 版本串。
- 合法性:实现公开协议属 cleanroom;参考开源**客户端**(协议消费方)可以,**别读其 MCU 固件二进制**。
