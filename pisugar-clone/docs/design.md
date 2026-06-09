# pisugar-clone 设计文档

> 本文件是本项目的设计「源头」,记录决策与理由,随版本演进。最后更新:2026-06。

## 1. 目标与约束

- **Cleanroom 实现 PiSugar 3 的 I²C 协议**,不逆向其 PCB/固件;让开源 [PiSugar Power Manager](https://github.com/PiSugar/pisugar-power-manager-rs) 和真树莓派把本板当正品 PiSugar 3 使用。
- **完全协议兼容** + **5V 输出**(带 boost,能给真树莓派供电)。深睡低功耗加分不强求。
- 形态:树莓派 Zero 兼容(沿用 sleepy-sensor 形态)。
- 为什么锁 PiSugar 3 而非 2:PiSugar 2 是树莓派**直读 IP5209 芯片寄存器**,没有独立协议层;PiSugar 3 才有 MCU 暴露的一套自定义 I²C 寄存器表,适合 cleanroom 兼容。

## 2. 协议契约(I²C 从机 0x57,SMBus 单字节读写)

> 实证来源:开源客户端 `pisugar-core/src/pisugar3.rs`(比官方 wiki 更全)。这是固件必须实现的接口。

| 寄存器 | 名称 | R/W | 实现要点 |
|---|---|---|---|
| 0x02 | CTR1 主控 | R/W | **bit7=USB 已插**、**bit6=允许充电**、**bit5=输出使能**;低位=延时/防误触/电源键 |
| 0x03 | CTR2 | R/W | 休眠 / 软关机(bit3/4/6) |
| 0x04 | 芯片温度 | R | −40…85 |
| 0x08 | TAP | R | 按键单击 / 双击 / 长按 |
| 0x0B | 写使能 | W | **写 0x29 解锁、0x00 上锁**;客户端每次写都「解锁→写→上锁」 |
| 0x20 / 0x21 | 电池控制 | R/W | 充电保护、SCL 唤醒位 |
| 0x22 / 0x23 | 电压 H/L | R | **大端 16bit,单位 mV**:`v=(VH<<8)|VL` |
| 0x26 / 0x27 | 输出电流 H/L | R | 大端 16bit |
| 0x2A | 电量 % | R | 直接一个 uint8 |
| 0x30 | RTC 控制 | R/W | |
| 0x31–0x37 | RTC 年/月/日/周/时/分/秒 | R/W | **全 BCD**;年寄存器 = 年−2000 |
| 0x40 | 闹钟控制 | R/W | **bit7=闹钟使能** |
| 0x44 | 闹钟周重复掩码 | R/W | |
| 0x45 / 0x46 / 0x47 | 闹钟 时/分/秒 | R/W | **BCD** |
| 0x50 | 自定义 I²C 地址 | R/W | |
| 0xE2… | 固件版本 | R | null 结尾 C 字符串(≤15 字节),如 `"1.0.0\0"` |

固件最小集 = 上表 + RTC 走时 + 「闹钟到点打开输出 = 定时开机」+ 电压→%曲线。

## 3. 硬件架构(草案,复用 sleepy-sensor v2 的 BQ24074 经验)

```
USB-C 5V ─→ BQ24074(power-path 充电)
               ├ BAT → 单节锂电 + DW01A/FS8205A 保护
               ├ /CHG、/PG 状态脚 → MCU GPIO(映射 CTR1 bit6 充电 / bit7 USB插)
               └ OUT ─→ 升压 5V(EN 由 MCU 控)─→ 树莓派 5V(排针 pin2/4)
MCU(I²C 从机 @0x57,跑自写固件):
   ├ ADC:电池电压(→0x22/23)、输出电流采样(→0x26/27)
   ├ RTC + 32.768k 晶振(→0x31-37 BCD),闹钟到点拉高 boost EN = 定时开机
   ├ boost EN(CTR1 bit5)、充电使能(CTR1 bit6)、按键/TAP(0x08)
   └ 自带常通小 LDO 从电池取电,维持 RTC + 协议响应
```
- BQ24074 的 **PG/CHG 状态脚**天然对应「USB 已插 / 允许充电」两位,映射干净。
- 不强求深睡 → MCU 可常醒(mA 级),`0x20` 的 SCL 唤醒位可先不做,省掉最难的「I²C 从机低功耗唤醒」。

## 4. 待定决策(选型必核手册,标 ⚠️ unclear)

1. ⚠️ **MCU**:方向 STM32G0/C0(便宜、I²C 从机、带 RTC+闹钟、ADC)。待核:I²C 从机地址匹配、RTC 闹钟、ADC 通道数、flash 够不够。
2. ⚠️ **boost**:形态是 Pi Zero,Zero 仅需 ~0.5A,选 ~1.5A 同步 boost 足够;若要喂 Pi4/5(3A)另选。具体型号待核。
3. **输出电流检测(0x26/27)**:做不做?不做就回 0,软件照跑,只是看不到放电电流。

## 5. 工作量预判

- PCB:中等(BQ24074 + boost + MCU + 保护),有加热台能回流。
- **固件 ≈ 80% 工作量**,是本项目主体。

## 6. 参考

- 协议金标准:[pisugar3.rs](https://raw.githubusercontent.com/PiSugar/pisugar-power-manager-rs/master/pisugar-core/src/pisugar3.rs)
- 官方寄存器说明:[PiSugar 3 I²C Datasheet](https://github.com/PiSugar/PiSugar/wiki/PiSugar-3-I2C-Datasheet)
- 开源上位机:[pisugar-power-manager-rs](https://github.com/PiSugar/pisugar-power-manager-rs)(GPL-3.0)
