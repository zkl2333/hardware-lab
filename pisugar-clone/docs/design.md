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

## 3. Phase 1 硬件架构(PiSugar 2,目标 IC:IP5209)

```
USB-C 5V ──→ IP5209 (I²C @0x75, QFN-24)
                ├─ VIN(pin20/21): USB 5V 输入
                ├─ VBAT(pin9): 单节锂电(4.2V)
                │    └─ 内置过充/过放/过流保护,无需外置 DW01A
                ├─ LX(pin13/14/15): DCDC 开关节点
                │    └─ 1µH 电感 (Isat/Idc >4.5A, DCR <0.01Ω)
                ├─ VOUT(pin16/17): 5V/2.4A → 树莓派 GPIO pin2/4
                ├─ CSIN(pin10) / CSIN_S(pin11): 0.01Ω/1% 电流采样电阻(1206)
                ├─ VREG(pin3): 3.1V/50mA 常通 LDO → SD3078 VCC + I²C 上拉
                ├─ SCL(pin24 L1): I²C 时钟 → Pi GPIO3;上拉 4.7kΩ → VREG
                ├─ SDA(pin1  L2): I²C 数据  → Pi GPIO2;上拉 4.7kΩ → VREG
                ├─ KEY(pin8): 电源键
                ├─ VSET(pin4): 悬空 → 4.2V 截止电压
                ├─ RSET(pin5): ~100kΩ → GND (电池内阻补偿)
                ├─ NTC(pin6): R 分压 ~1V (不用 NTC: R1=2MΩ VBAT→NTC, R2=1MΩ NTC→GND)
                └─ LIGHT(pin22): 接 GND (不用手电筒功能)

SD3078 RTC (I²C @0x32)
    ├─ VCC: VREG (3.1V)  ← VREG 50mA 常通,SD3078 <1mA,预算充足
    ├─ SCL/SDA: 与 IP5209 共用 I²C 总线
    └─ 32.768kHz 晶振
```

**关键决策**:
- IP5209 内置电池保护 → 无需外置 DW01A。
- VREG 50mA 常通 → 同时驱动 SD3078 和 I²C 上拉无压力(SD3078 <1mA + 上拉 2×0.66mA = ~2.3mA)。
- I²C 上拉接 VREG:IP5209 唤醒时检测 SCL/SDA 是否拉到 VREG(3.1V)来选择进 I²C 模式,上拉到 VREG 是正确做法。

## 4. IP5209 关键参数(手册核实 ✓)

| 参数 | 值 | 备注 |
|---|---|---|
| 封装 | QFN-24, 4×4mm, 0.5mm pitch | 有加热台可回流 |
| 货源 | 淘宝 ~¥1/颗 | LCSC C181695 长期缺货,淘宝有大量现货 |
| Boost 输出 | 5V / 2.4A(typ), 2A@92.5%效率 | Pi Zero 足够 |
| 充电电流 | 2.1A(typ) | 由外置 CSIN 采样电阻精确测量 |
| 内置保护 | 过充/过放/过流/短路/NTC | 无需外置 DW01A |
| VREG | 3.1V / 50mA 常通 LDO | SD3078 + I²C 上拉均从此取电 |
| 待机电流 | 75µA (VBAT=3.7V, VIN=0) | 手册标注值 |
| I²C 地址 | 0x75 | SCL=pin24(L1), SDA=pin1(L2) |
| 电流采样 | CSIN/CSIN_S 外置 0.01Ω 电阻 | 支持 0xa4/0xa5 寄存器读取 |
| 电感要求 | 1µH, Isat/Idc >4.5A, DCR <0.01Ω | 如 SPM70701R0 |

## 5. I²C 寄存器(pisugar-power-manager 读取,实证于 ip5209.rs)

| 地址 | 名称 | R/W | 说明 |
|---|---|---|---|
| 0xa2 | VOLT_L | R | 电压低字节 |
| 0xa3 | VOLT_H | R | 电压高字节,bit5=符号 |
| 0xa4 | CURR_L | R | 电流低字节(依赖 CSIN 采样) |
| 0xa5 | CURR_H | R | 电流高字节 |
| 0x55 | GPIO/CHG | R/W | 充电使能控制 |
| 0x53 | GPIO_IN | R/W | GPIO 输入使能 |
| 0x01 | SYS_CTL0 | R/W | Boost/充电使能 |
| 0x02 | SYS_CTL1 | R/W | 轻载关机/自动开机 |
| 0x26 | VSET_CTL | R/W | VSET 来源(PIN vs 寄存器) |

**电压计算**: `V = (2600 ± raw × 0.26855) / 1000 V`(符号位由 VOLT_H bit5 决定)

**电流计算**: `I = raw × 0.745985 / 1000 A`(CSIN 外置 0.01Ω 采样电阻,精度可靠)

**电量**: 电压→%插值曲线,3.1V=0%, 4.16V=100%

寄存器手册来源:IP5209/IP5109/IP5207/IP5108 I²C Registers V1.2(官方文档,覆盖 IP5209)。

## 6. BOM 草案(Phase 1,目标 IC:IP5209)

| 位号 | 器件 | 规格 | 备注 |
|---|---|---|---|
| U1 | **IP5209** | QFN-24, 4×4mm | 淘宝 ~¥1/颗;LCSC C181695 缺货 |
| U2 | SD3078 | SOP 封装, JLCPCB C916255 | RTC,VCC 接 VREG(3.1V) |
| L1 | 电感 | 1µH, Isat>4.5A, DCR<0.01Ω | SPM70701R0 或等效 |
| R_sense | 采样电阻 | 0.01Ω / 1% / **1206** | CSIN 电流检测,精度关键 |
| R_RSET | 电阻 | ~100kΩ / 0603 | 电池内阻补偿 |
| R_NTC1/2 | 电阻 | 2MΩ + 1MΩ / 0603 | NTC 引脚偏置(不用 NTC) |
| C_IN | 电容 | 10µF × 2 / 0603 / 16V+ | VIN 滤波 |
| C_OUT | 电容 | 10µF × 4 + 22µF × 2 / 0603 | VOUT 滤波 |
| C_BAT | 电容 | 10µF × 2 / 0603 | VBAT 滤波 |
| C_VREG | 电容 | 2.2µF / 0603 | VREG 滤波 |
| C_LX | 电容 | 2.2nF / 0603 | LX 节点 snubber |
| R_pull | 电阻 | 4.7kΩ × 2 / 0603 | I²C 上拉到 VREG(3.1V) |
| J_USB | USB-C 母座 | — | 充电输入 |
| J_BAT | 电池连接器 | — | 单节锂电 |
| J_PI | 40pin GPIO 排针 | 2×20P | 连接树莓派 |
| SW1 | 按键 | 轻触 | 电源键 |
| X1 | 晶振 | 32.768kHz | SD3078 用 |

> **备选料**:若 IP5209 一时买不到,**IP5109 是 pin2pin + 寄存器官方兼容**的 drop-in 替代(仅少 DCP 充电电流识别,pisugar 用不到),BOM 完全不动。详见[附录 A](#附录-a--ip5209-家族选型对比)。

## 7. 待定决策

1. **焊后验证**:装好板第一件事 `i2cdetect -y 1` 确认 0x75/0x32 在线,再 `i2cdump -y 1 0x75` 对照寄存器表;0xa2/0xa3 出合理电压即电源链路 OK。
2. **SD3078 是否 Phase 1 必须**:pisugar-power-manager 要 RTC 才能调度自动开关机;若只验证电源/电量,可先跳过 SD3078,引脚预留。
3. **LED SOC 指示**:IP5209 支持 LED 灯柱功能;Pi Zero 形态空间紧,建议留焊盘但默认不焊。

## 8. 工作量预判(Phase 1)

- **固件:几乎为零** — IP5209 硬件自治,Pi 直读寄存器。
- **PCB**:中等(QFN-24 回流 + 电源布局),主要工作在画板。
- **调试**:安装 pisugar-power-manager,`i2cdetect` 确认 0x75/0x32 存在即成功。

## 9. 参考

- IP5209 数据手册 V1.01 (INJOINIC):[datasheets/IP5209_INJOINIC.pdf](datasheets/IP5209_INJOINIC.pdf) — **主选**,淘宝有现货
- IP5209/IP5109/IP5207/IP5108 I²C 寄存器手册 V1.2:[datasheets/IP5209_I2C_Registers_INJOINIC.pdf](datasheets/IP5209_I2C_Registers_INJOINIC.pdf) — **寄存器金标准,官方覆盖 4 颗**
- IP5109 数据手册(pin2pin 备选):[datasheets/IP5109_INJOINIC.pdf](datasheets/IP5109_INJOINIC.pdf)
- IP5207 数据手册(参考):[datasheets/IP5207_INJOINIC.pdf](datasheets/IP5207_INJOINIC.pdf)
- IP5209T 数据手册 V1.02(精简料,无 CSIN,不推荐):[datasheets/IP5209T_INJOINIC.pdf](datasheets/IP5209T_INJOINIC.pdf)
- SD3078 数据手册:[datasheets/SD3078_WAVE.pdf](datasheets/SD3078_WAVE.pdf) — JLCPCB C916255
- PiSugar 2 I²C 手册:[PiSugar Wiki](https://github.com/PiSugar/PiSugar/wiki/PiSugar-2-(Pro)-I2C-Manual)
- 驱动源码(寄存器实证):[ip5209.rs](https://github.com/PiSugar/pisugar-power-manager-rs/blob/master/pisugar-core/src/ip5209.rs)

---

## 附录 A — IP5209 家族选型对比

来源:IP5109/IP5207 数据手册内《IP 系列移动电源 IC 型号选择表》+ 各自引脚定义(已逐脚核实)。

| 型号 | 放电 | 充电 | LED | DCP识别 | QC2.0 | I²C | CSIN采样 | 封装 | 与IP5209引脚 | LCSC |
|---|---|---|---|---|---|---|---|---|---|---|
| **IP5209** | 2.4A | 3.0A | 3/4/5 | ✓ | — | ✓ | ✓ | QFN24 | 基准 | C181695 缺货 |
| **IP5109** | 2.4A | 3.0A | 3/4/5 | — | — | ✓ | ✓ | QFN24 | **PIN2PIN** | C181699 缺货 |
| IP5209S | 3A | 4.8A | 3/4/5 | — | ✓ | ✓ | ✓ | QFN24 | 待核 | — |
| IP5207 | 1.2A | 1.2A | 3/4/5 | — | — | ✓ | ✓ | QFN24 | 引脚不同 | C181696 缺货 |
| IP5108 | 2.0A | 2.5A | 3/4/5 | — | — | ✓ | 未核 | eSOP16 | 不同封装 | C180943 有货 |
| IP5209T | 2.4A | ~1.9A | — | — | — | ✓ | ✗ 无CSIN | QFN24 | 引脚不同 | C284964 有货 |

> 选择表标题列为「放电 / 充电」,故 IP5209 = 放电 2.4A / 充电 3.0A。IP5209S/IP5108 未读全手册,参数取自选择表,引脚/CSIN 标「待核/未核」。

**选型结论**:
- **首选 IP5209**:原版 PiSugar 2 即此颗,DCP 协议齐全,寄存器手册金标准,淘宝 ~¥1。
- **备选 IP5109**:与 IP5209 **pin2pin**、同 QFN24、同 I²C、同 CSIN 电流采样、同 14bit ADC、同 2.4A放/3A充;**唯一差异是无 DCP 充电电流识别**——pisugar 不依赖 DCP,故为 drop-in 无损替代,且寄存器手册官方覆盖 IP5109,兼容性有背书。
- **不推荐 IP5209T**:虽 LCSC 现货,但①不在寄存器手册四颗之列;②无 CSIN,电流寄存器 0xa4/0xa5 存疑;③引脚与 IP5209 不同需重画。既然 IP5209 淘宝 ¥1,无必要冒此险。
- IP5207(1.2A 偏小且引脚不同)、IP5108(eSOP16)仅作了解。

**回到「寄存器读取完整兼容吗」**:寄存器手册《IP5209/IP5109/IP5207/IP5108 I²C Registers V1.2》官方**同时定义这四颗** → 它们 ADC/控制寄存器同源。用 IP5209(或 IP5109),pisugar-power-manager 读取**完整兼容,有官方文档背书**;唯独 IP5209T 无此背书,这也是不选它的核心理由。
