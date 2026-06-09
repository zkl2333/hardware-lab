# pisugar-clone 设计文档

> 本文件是本项目的设计「源头」,记录决策与理由,随版本演进。最后更新:2026-06。

## 1. 目标与约束

- **Cleanroom 实现 PiSugar I²C 协议**的树莓派电源 / UPS HAT:从公开规范 + 开源客户端重新实现,**不逆向其 PCB/固件**。
- 目标——让开源 PiSugar Power Manager 和真树莓派把本板当成正品使用。
- 形态:树莓派 Zero 兼容(沿用 sleepy-sensor 形态)。

## 2. 分阶段策略

### Phase 1 — PiSugar 2 兼容(当前目标)
**用真 IP5209 直接上板**,纯硬件方案,几乎不需要写固件。树莓派直读 IP5209 寄存器(@0x75) + SD3078 RTC(@0x32)。

目标:快速出一块能跑 pisugar-power-manager 的板子,验证形态(boost/保护/排针位置),建立自信。

### Phase 2 — PiSugar 3 兼容(后续)
MCU 暴露自定义寄存器表(@0x57),整套固件工作量大,待 Phase 1 完成后规划。

---

## 3. Phase 1 硬件架构(PiSugar 2)

```
USB-C 5V ──→ IP5209 (I²C @0x75, QFN-24)
                ├─ VIN(pin20/21): USB 5V 输入
                ├─ VBAT(pin9): 单节锂电(4.2V)
                │    └─ 内置过充/过放/过流保护,无需外置 DW01A
                ├─ LX(pin13/14/15): DCDC 开关节点
                │    └─ 1µH 电感 (Isat/Idc >4.5A, DCR <0.01Ω)
                ├─ VOUT(pin16/17): 5V/2.4A → 树莓派 GPIO pin2/4
                ├─ CSIN/CSIN_S(pin10/11): 0.01Ω/1% 电流采样电阻(1206)
                ├─ VREG(pin3): 3.1V/50mA 常通 LDO → SD3078 VCC + I²C 上拉
                ├─ SCL(pin24 L1/SCL): I²C 时钟 → Pi GPIO3
                ├─ SDA(pin1 L2/SDA): I²C 数据  → Pi GPIO2
                ├─ KEY(pin8): 电源键
                ├─ VSET(pin23): 悬空 → 4.2V 电池
                ├─ RSET(pin5): ~100kΩ → GND (电池内阻补偿)
                ├─ NTC(pin6): R分压到 ~1V (NTC 不用时: R4=2MΩ, R6=1MΩ)
                └─ LIGHT(pin22): 接 GND (不用手电筒功能)

SD3078 RTC (I²C @0x32)
    ├─ VCC: VREG (3.1V)
    ├─ SCL/SDA: 与 IP5209 共用 I²C 总线
    └─ 32.768kHz 晶振
```

**关键决策**:IP5209 内置电池保护 → 无需外置保护 IC,BOM 精简。

## 4. IP5209 关键参数(手册核实 ✓)

| 参数 | 值 | 备注 |
|---|---|---|
| 封装 | QFN-24, 4×4mm, 0.5mm pitch | 有加热台可回流 |
| Boost 输出 | 5V / 2.4A(typ), 2A@92.5%效率 | Pi Zero 足够 |
| 充电电流 | 2.4A(typ) | |
| 内置保护 | 过充/过放/过流/短路/NTC | 无需外置 DW01A |
| VREG | 3.1V / 50mA 常通 LDO | 给 RTC 和上拉供电 |
| 待机电流 | 75µA (VBAT=3.7V, VIN=0) | |
| I²C 地址 | 0x75 | SCL=pin24, SDA=pin1 |
| 电感要求 | 1µH, Isat/Idc >4.5A, DCR <0.01Ω | 如 SPM70701R0 |

## 5. I²C 寄存器(pisugar-power-manager 读取,实证于 ip5209.rs)

| 地址 | 名称 | R/W | 说明 |
|---|---|---|---|
| 0xa2 | VOLT_L | R | 电压低字节 |
| 0xa3 | VOLT_H | R | 电压高字节 |
| 0xa4 | CURR_L | R | 电流低字节 |
| 0xa5 | CURR_H | R | 电流高字节 |
| 0x55 | GPIO/CHG | R/W | 充电使能控制 |
| 0x53 | GPIO_IN | R/W | GPIO 输入使能 |
| 0x01 | SHUTDOWN | R/W | 关机控制 |
| 0x02 | AUTO_OFF | R/W | 自动关机使能 |

**电压计算**: `V = (2600 ± raw × 0.26855) / 1000 V`(符号位由 VOLT_H bit5 决定)

**电流计算**: `I = raw × 0.745985 / 1000 A`

**电量**: 电压→%插值曲线,3.1V=0%, 4.16V=100%

## 6. BOM 草案(Phase 1)

| 位号 | 器件 | 规格 | 备注 |
|---|---|---|---|
| U1 | IP5209 / IP5209T | QFN-24 | ⚠️ 见下方注意 |
| U2 | SD3078 | SOT 封装 | RTC, JLCPCB C916255 |
| L1 | 电感 | 1µH, Isat>4.5A, DCR<0.01Ω | SPM70701R0 或等效 |
| R_sense | 采样电阻 | 0.01Ω / 1% / **1206** | 电流检测,精度关键 |
| R_RSET | 电阻 | ~100kΩ / 0603 | 电池内阻补偿 |
| R_NTC1/2 | 电阻 | 2MΩ + 1MΩ / 0603 | NTC 引脚偏置(不用 NTC) |
| C_IN | 电容 | 10µF × 2 / 0603 / 16V+ | VIN 滤波 |
| C_OUT | 电容 | 10µF × 4 + 22µF × 2 / 0603 | VOUT 滤波 |
| C_BAT | 电容 | 10µF × 2 / 0603 | VBAT 滤波 |
| C_VREG | 电容 | 2.2µF / 0603 | VREG 滤波 |
| C_LX | 电容 | 2.2nF / 0603 | LX 节点 snubber |
| R_pull | 电阻 | 4.7kΩ × 2 / 0603 | I²C 上拉到 VREG |
| J_USB | USB-C 母座 | — | 充电输入 |
| J_BAT | 电池连接器 | — | 单节锂电 |
| J_PI | 40pin GPIO 排针 | 2×20P | 连接树莓派 |
| SW1 | 按键 | 轻触 | 电源键 |
| X1 | 晶振 | 32.768kHz | SD3078 用 |

**⚠️ IP5209 货源**:
- IP5209 (C181695) LCSC **缺货**;IP5209T (C284964) 有货 3000+,封装相同
- IP5209T 寄存器兼容性 **unclear** — 未找到 IP5209T 手册确认
- 建议先从淘宝搜原版 IP5209;或接受 IP5209T 风险(先买几颗测试再量产)

## 7. 待定决策

1. ⚠️ **IP5209T 兼容性**:买到 IP5209T 后,用 Pi 跑 `i2cdump -y 1 0x75` 验证寄存器响应是否与 ip5209.rs 一致。
2. **SD3078 是否 Phase 1 必须**:pisugar-power-manager 的 RTC 功能由 SD3078 提供;若只想先验证电源/电量部分,可暂时跳过 RTC,引脚预留即可。
3. **LED SOC 指示**:加 4 个 LED(IP5209 原生支持)or 省掉?Pi Zero 形态空间紧张。

## 8. 工作量预判(Phase 1)

- **固件:几乎为零** — IP5209 硬件自治,Pi 直读寄存器。
- **PCB**:中等(QFN-24 回流 + 电源布局),主要工作在画板。
- **调试**:安装 pisugar-power-manager,`i2cdetect` 确认 0x75/0x32 存在即成功。

## 9. 参考

- IP5209 数据手册 V1.01 (INJOINIC, 2014): [LCSC C181695](https://www.lcsc.com/product-detail/C181695.html)
- PiSugar 2 I²C 手册: [PiSugar Wiki](https://github.com/PiSugar/PiSugar/wiki/PiSugar-2-(Pro)-I2C-Manual)
- 驱动源码(寄存器金标准): [ip5209.rs](https://github.com/PiSugar/pisugar-power-manager-rs/blob/master/pisugar-core/src/ip5209.rs)
- SD3078 RTC: JLCPCB C916255, 内核驱动 `rtc-sd3078`
