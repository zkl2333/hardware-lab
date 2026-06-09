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

## 3. Phase 1 硬件架构(PiSugar 2,目标 IC:IP5209T)

```
USB-C 5V ──→ IP5209T (I²C @0x75, QFN-24)
                ├─ VIN(pin20/21): USB 5V 输入
                ├─ VBAT(pin9): 单节锂电(4.2V)
                │    └─ 内置过充/过放/过流保护,无需外置 DW01A
                ├─ LX(pin13/14/15): DCDC 开关节点
                │    └─ 1µH 电感 (Isat/Idc >4.5A, DCR <0.01Ω)
                ├─ VOUT(pin16/17): 5V/2.4A → 树莓派 GPIO pin2/4
                ├─ ICHG(pin11): 82kΩ → GND (充电限流 ~1.5A; NC → 1.9A 默认)
                ├─ VTHS(pin4): → VREG (3.7V 截止平台; GND → 3.6V 平台)
                ├─ NTC(pin6): → GND (不用外置 NTC 时直接接 GND)
                ├─ VREG(pin3): 3.1V/5mA LDO (休眠时断开) → 仅 I²C 上拉
                ├─ SCL(pin24 L1): I²C 时钟 → Pi GPIO3;上拉 4.7kΩ → Pi 3.3V
                ├─ SDA(pin1 L2): I²C 数据  → Pi GPIO2;上拉 4.7kΩ → Pi 3.3V
                ├─ KEY(pin8): 电源键
                ├─ VSET(pin23): 悬空 → 4.2V 电池
                ├─ RSET(pin5): ~100kΩ → GND (电池内阻补偿)
                └─ LIGHT(pin22): 接 GND (不用手电筒功能)

SD3078 RTC (I²C @0x32)
    ├─ VCC: Pi GPIO header 3.3V(pin1/17)  ← 不能用 VREG(仅 5mA 且休眠断开)
    ├─ SCL/SDA: 与 IP5209T 共用 I²C 总线
    └─ 32.768kHz 晶振
```

**关键决策**:
- IP5209T 内置电池保护 → 无需外置保护 IC。
- SD3078 必须从 Pi 的 3.3V 取电:IP5209T VREG 仅 5mA 且休眠时断开,5mA 全给 I²C 上拉(2×2.2kΩ≈2.8mA)已紧张,不够再驱动 RTC。原版 PiSugar 2 也是从 Pi 3.3V 取电,行为一致(Pi 断电则 RTC 断电,时间需重设)。
- I²C 上拉接 Pi 3.3V(不接 VREG):IP5209T 用 3.3V > VREG(3.1V) → 唤醒时仍能检测到上拉高电平进入 I²C 模式,且不消耗 VREG 电流预算。

## 4. IP5209T 关键参数(手册核实 ✓)

| 参数 | 值 | 备注 |
|---|---|---|
| 封装 | QFN-24, 4×4mm, 0.5mm pitch | 与 IP5209 **尺寸相同但引脚功能不同** |
| LCSC | C284964(3000+有货) | IP5209 C181695 长期缺货 |
| Boost 输出 | 5V / 2.4A(typ), 2A@92.5%效率 | Pi Zero 足够 |
| 充电电流 | 由 ICHG 电阻设定: 82kΩ→1.5A, NC→1.9A | 无 CSIN 外置采样电阻 |
| 内置保护 | 过充/过放/过流/短路/NTC | 无需外置 DW01A |
| VREG | 3.1V / **5mA**,**休眠时断开** | ⚠️ 不能驱动 RTC |
| 待机电流 | 与 IP5209 同家族,类似 75µA | 未在手册中明确标注 |
| I²C 地址 | 0x75 | SCL=pin24, SDA=pin1 |
| 电感要求 | 1µH, Isat/Idc >4.5A, DCR <0.01Ω | 如 SPM70701R0 |
| VTHS | VREG → 4.2V 充满/3.7V 截止; GND → 4.2V/3.6V | 接 VREG 选择更稳健平台 |

## 5. I²C 寄存器(pisugar-power-manager 读取,实证于 ip5209.rs)

| 地址 | 名称 | R/W | IP5209T 兼容性 | 说明 |
|---|---|---|---|---|
| 0xa2 | VOLT_L | R | ✅ 大概率兼容 | 电压低字节 |
| 0xa3 | VOLT_H | R | ✅ 大概率兼容 | 电压高字节,bit5=符号 |
| 0xa4 | CURR_L | R | ⚠️ 可能恒为 0 | 电流低字节(IP5209T 无 CSIN) |
| 0xa5 | CURR_H | R | ⚠️ 可能恒为 0 | 电流高字节 |
| 0x55 | GPIO/CHG | R/W | ✅ 大概率兼容 | 充电使能控制 |
| 0x53 | GPIO_IN | R/W | ✅ 大概率兼容 | GPIO 输入使能 |
| 0x01 | SYS_CTL0 | R/W | ✅ 大概率兼容 | Boost/充电使能 |
| 0x02 | SYS_CTL1 | R/W | ✅ 大概率兼容 | 轻载关机/自动开机 |
| 0x26 | VSET_CTL | R/W | ✅ 大概率兼容 | VSET 来源(PIN vs 寄存器) |

**电压计算**: `V = (2600 ± raw × 0.26855) / 1000 V`(符号位由 VOLT_H bit5 决定)

**电流计算**: `I = raw × 0.745985 / 1000 A`(IP5209T 无 CSIN → 电流显示可能为 0mA)

**电量**: 电压→%插值曲线,3.1V=0%, 4.16V=100%

### 寄存器兼容性说明

官方寄存器手册只覆盖 IP5209/IP5109/IP5207/IP5108,**不覆盖 IP5209T**。

- **电压 0xa2/0xa3**:IP5209T 有 ADC 测量电池电压,应兼容。pisugar 核心功能(电量%、充电状态)依赖此寄存器。
- **电流 0xa4/0xa5**:IP5209T 无 CSIN 外置采样电阻,内部电流测量来源未知,可能返回 0。pisugar-power-manager 用电压变化趋势判断充电状态,不依赖电流读数决策,电流只是界面显示——即使为 0 不影响核心功能。
- **控制寄存器 0x01/0x02/0x55**:硬件功能相同,应兼容。
- **I²C 唤醒检测**:IP5209 家族唤醒时检测 SCL/SDA 是否被拉高(>VREG 3.1V)来决定进 I²C 还是 LED 模式。Pi 3.3V 上拉 > 3.1V,满足条件。

**建议**:焊板后第一件事 `i2cdump -y 1 0x75`,对照上表验证,若 0xa2/0xa3 有合理电压读数则整体可行。

## 6. BOM 草案(Phase 1,目标 IC:IP5209T)

| 位号 | 器件 | 规格 | 备注 |
|---|---|---|---|
| U1 | **IP5209T** | QFN-24, LCSC C284964 | 3000+有货;注意引脚与 IP5209 **不同** |
| U2 | SD3078 | SOT 封装, JLCPCB C916255 | RTC,从 Pi 3.3V(pin1/17)供电 |
| L1 | 电感 | 1µH, Isat>4.5A, DCR<0.01Ω | SPM70701R0 或等效 |
| R_ICHG | 电阻 | 82kΩ / 0603 | ICHG→GND,设定充电电流≈1.5A |
| R_VTHS | 跳线 | 接 VREG | VTHS→VREG → 3.7V 截止平台 |
| R_RSET | 电阻 | ~100kΩ / 0603 | 电池内阻补偿 |
| C_IN | 电容 | 10µF × 2 / 0603 / 16V+ | VIN 滤波 |
| C_OUT | 电容 | 10µF × 4 + 22µF × 2 / 0603 | VOUT 滤波 |
| C_BAT | 电容 | 10µF × 2 / 0603 | VBAT 滤波 |
| C_VREG | 电容 | 2.2µF / 0603 | VREG 滤波 |
| C_LX | 电容 | 2.2nF / 0603 | LX 节点 snubber |
| R_pull | 电阻 | 4.7kΩ × 2 / 0603 | I²C 上拉到 **Pi 3.3V**(不接 VREG) |
| J_USB | USB-C 母座 | — | 充电输入 |
| J_BAT | 电池连接器 | — | 单节锂电 |
| J_PI | 40pin GPIO 排针 | 2×20P | 连接树莓派 |
| SW1 | 按键 | 轻触 | 电源键 |
| X1 | 晶振 | 32.768kHz | SD3078 用 |

**与 IP5209 BOM 差异**:
- **去掉** R_sense(0.01Ω 采样):IP5209T 无 CSIN 引脚
- **去掉** R_NTC1/R_NTC2:IP5209T NTC 引脚直接接 GND
- **新增** R_ICHG(82kΩ):替代 CSIN 方式设定充电电流
- **I²C 上拉改接 Pi 3.3V**:VREG 仅 5mA 且休眠断开,不作上拉电源
- **SD3078 供电改接 Pi 3.3V**:同上理由

## 7. 待定决策

1. ⚠️ **IP5209T 寄存器实测**:买到 IP5209T 后,搭一块最小测试板,用 Pi 跑 `i2cdump -y 1 0x75`。重点确认:0xa2/0xa3 有合理电压读数;0xa4/0xa5 是否有电流值(预期可能为 0)。
2. **SD3078 是否 Phase 1 必须**:pisugar-power-manager 要 RTC 才能调度自动开关机;若只验证电源/电量,可先跳过 SD3078,引脚预留。
3. **LED SOC 指示**:IP5209T 支持 LED 灯柱功能;Pi Zero 形态空间紧,建议留焊盘但默认不焊。

## 8. 工作量预判(Phase 1)

- **固件:几乎为零** — IP5209 硬件自治,Pi 直读寄存器。
- **PCB**:中等(QFN-24 回流 + 电源布局),主要工作在画板。
- **调试**:安装 pisugar-power-manager,`i2cdetect` 确认 0x75/0x32 存在即成功。

## 9. 参考

- IP5209T 数据手册 V1.02 (INJOINIC): [docs/datasheets/IP5209T_INJOINIC.pdf](datasheets/IP5209T_INJOINIC.pdf) — LCSC C284964
- IP5209 数据手册 V1.01 (INJOINIC, 2014): [docs/datasheets/IP5209_INJOINIC.pdf](datasheets/IP5209_INJOINIC.pdf) — 原版(LCSC 缺货,仅参考)
- IP5209/IP5109/IP5207/IP5108 I²C 寄存器手册 V1.2: [docs/datasheets/IP5209_I2C_Registers_INJOINIC.pdf](datasheets/IP5209_I2C_Registers_INJOINIC.pdf)
- SD3078 数据手册: [docs/datasheets/SD3078_WAVE.pdf](datasheets/SD3078_WAVE.pdf) — JLCPCB C916255
- PiSugar 2 I²C 手册: [PiSugar Wiki](https://github.com/PiSugar/PiSugar/wiki/PiSugar-2-(Pro)-I2C-Manual)
- 驱动源码(寄存器金标准): [ip5209.rs](https://github.com/PiSugar/pisugar-power-manager-rs/blob/master/pisugar-core/src/ip5209.rs)
