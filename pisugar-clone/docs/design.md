# pisugar-clone 设计文档

> 本文件是本项目的设计「源头」,记录决策与理由,随版本演进。最后更新:2026-06。

## 1. 目标与约束

- **Cleanroom 实现 PiSugar I²C 协议**的树莓派电源 / UPS HAT:从公开规范 + 开源客户端重新实现,**不逆向其 PCB/固件**。
- 目标——让开源 PiSugar Power Manager 和真树莓派把本板当成正品使用。
- 形态:树莓派 Zero 兼容(沿用 sleepy-sensor 形态)。

## 2. 分阶段策略

### Phase 1 — PiSugar 2 兼容(当前目标)
**用真 IP5209 直接上板**,纯硬件方案,几乎不需要写固件。树莓派直读 IP5209 寄存器(@0x75) + SD3078 RTC(@0x32)。

目标:快速出一块能跑 pisugar-power-manager 的板子,验证形态(boost/保护/顶针位置),建立自信。

### Phase 2 — PiSugar 3 兼容(后续)
MCU 暴露自定义寄存器表(@0x57),整套固件工作量大,待 Phase 1 完成后规划。

---

## 3. Phase 1 硬件架构与机械结构(目标 IC:IP5209)

### 3.1 电气架构

```
USB-C 5V ──→ IP5209 (I²C @0x75, QFN-24)
                ├─ VIN(pin20/21): USB-C 5V 输入(充电)
                ├─ VBAT(pin9): 单节锂电,内置过充/过放/过流/短路保护
                ├─ LX(pin13/14/15): DCDC 开关节点 → 1µH 电感(Idc>4.5A, DCR<0.01Ω)
                ├─ VOUT(pin16/17): 5V/2.4A boost 输出 ──┐
                ├─ CSIN(pin10)/CSIN_S(pin11): 0.01Ω/1% 采样电阻(1206)→ 电流检测
                ├─ VREG(pin3): 3.1V/50mA 常通 LDO → SD3078 VDD + I²C 上拉
                ├─ L1/SCL(pin24): I²C 时钟 ──┐ 各 4.7kΩ 上拉 → VREG
                ├─ L2/SDA(pin1):  I²C 数据 ──┤
                ├─ KEY(pin8): 电源键 ← 轻触按键 + 滑动开关
                ├─ VSET(pin23): 悬空 → 4.2V 截止
                ├─ RSET(pin5): 100kΩ → 电池内阻补偿
                ├─ NTC(pin6): 不用 NTC → 按官方 R7 NC 处理
                └─ LIGHT(pin22): 不用照明 → 接 GND
                                               │ │ │
   5V→Pi pin2/4  GND→Pi pin6  SDA→Pi pin3  SCL→Pi pin5
                          │ 经 5 根 pogo 顶针顶 Pi 背面 GPIO 角部(不焊排针)
                          ▼
                   树莓派 Zero

SD3078 RTC (I²C @0x32, SOP-8)
    ├─ VDD: VREG(3.1V)供电(工作 2.7–5.5V,典型 0.8µA)
    ├─ SCL/SDA: 与 IP5209 共用 I²C 总线(0x32 ≠ 0x75,不冲突)
    └─ 内置 32.768kHz 晶振 + TCXO 温补(±3.8ppm)→ 无需外置晶振/谐振电容
```

### 3.2 机械结构(完全兼容 PiSugar 2,已核实)

| 项目 | 规格 | 说明 |
|---|---|---|
| PCB 尺寸 | **65 × 30 mm** | = Pi Zero footprint |
| 安装孔 | 4× M2.5,间距 **58 × 23 mm**,孔径 2.75mm,角内缩 3.5mm | 对齐 Pi Zero 标准孔位 |
| 与 Pi 连接 | **pogo 弹簧顶针**(非排针),从 Pi **背面**顶 GPIO 角部 2×3 | 不占 GPIO、不挡其他 HAT |
| 顶针接触点 | Pi pin2/4=5V、pin6=GND、pin3=SDA、pin5=SCL | 仅这 5 点,2.54mm 网格 |
| 安装方式 | 板在 Pi **下方**,4 颗 M2.5 螺母对齐,螺丝从 Pi 面拧入 | 同 PiSugar 2 |
| 电源开关 | **3 脚 SPDT 滑动开关**(ON/CTRL/OFF) | ON-CTRL 短接=开,OFF-CTRL 短接=关 |
| 自定义键 | **轻触按键** → IP5209 KEY(pin8) | 单击/双击/长按由软件定义 |
| 电池 | 单节锂电(原版 1200mAh),贴板背面 | JST 连接器或焊盘 |

**连接关键**:沿用 PiSugar「SugarPin」思路——**5 根 2.54mm 立式弹簧顶针**顶在 Pi GPIO 角部 2×3 区对应的 5V/GND/SDA/SCL 焊盘背面。5V 针要载 boost 满载 2.4A,优先选大电流顶针,或借 Pi 的 pin2+pin4 两个 5V 点双针并联分流。顶针自由高 = 铜柱/螺母高 + 压缩余量,**需打样实测**接触可靠性(PiSugar 已知坑:Pi GPIO 焊盘背面残留阻焊会致接触不良)。

### 3.3 关键决策
- IP5209 内置电池保护 → 无需外置 DW01A。
- VREG 50mA 常通 → 同时供 SD3078(µA 级)+ I²C 上拉(2×0.66mA),预算充足。
- I²C 上拉接 VREG:IP5209 唤醒时检测 SCL/SDA 是否被拉到 VREG(3.1V)以进入 I²C 模式,上拉到 VREG 是正确做法。
- SD3078 内置晶振 → 省掉外置 32.768kHz 晶振 + 谐振电容,BOM 更简、可靠性更高。
- **不焊 40pin 排针**:PiSugar 形态精髓是 pogo 顶针背接,不占 GPIO(这也是与普通 UPS HAT 的根本区别)。
- 滑动开关与 IP5209 KEY 的精确接线(实现硬开/关机锁定)待原理图阶段参照 KEY 时序确定。

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
| 电感要求 | 1µH, Idc>4.5A, DCR<0.01Ω(官方手册) | 选 CY54-1.0UH(5.4A/13mΩ),见 BOM |

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

## 6. BOM(Phase 1,选型定稿)

被动件/结构件的 LCSC·JLC 料号已逐条核实(幻觉料号已剔除);标 *unclear* 的为下单时再确认。

**核心 IC / RTC**
| 位号 | 器件 | 规格 | 料号 | 备注 |
|---|---|---|---|---|
| U1 | IP5209 | QFN-24 移动电源 SoC | 淘宝 ~¥1(LCSC **C181695** 缺货) | 内置电池保护;备选 IP5109(**C181699**, pin2pin) |
| U2 | SD3078 | RTC, SOP-8,内置晶振+TCXO | JLC **C916255** | I²C@0x32;VDD←VREG;**无需外置晶振** |

**功率路径**
| 位号 | 器件 | 规格 | 料号 | 备注 |
|---|---|---|---|---|
| L1 | 功率电感 | 1µH, Idc 5.4A, DCR 13mΩ, 5.8×5.2mm | SHOU HAN CY54-1.0UH **C2929426** | 官方要 Idc>4.5A/DCR<10mΩ;此件 DCR 13mΩ 略高→满载多耗 ~0.2W,够用;欲更优挑 <10mΩ 款 |
| R_sense | 采样电阻 | 0.01Ω 1% **1W / 2512** | UNI-ROYAL 25121WF100MT4E **C127692** | CSIN 电流检测;**2512 非 1206**(0.01Ω/1W 1206 货少);车规备选 Viking **C2920591** |

**电容**(布局参考 IP5209 官方应用 BOM:22µF 主滤波分布于 VBAT/VOUT)
| 位号 | 规格 | 料号 | 备注 |
|---|---|---|---|
| C_BAT ×2 | 22µF 25V X5R 0805 | Samsung CL21A226MAQNNNE **C45783** (Basic) | VBAT 滤波 |
| C_OUT ×2 | 22µF 25V X5R 0805 | 同上 **C45783** | VOUT(→Pi 5V)滤波 |
| C_IN ×2 | 10µF 25V X5R 0603 | Samsung CL10A106MA8NRNC **C96446** (Basic) | VIN(USB-C)滤波 |
| C_dec ×2 | 0.1µF 50V X7R 0603 | Basic 通用 104(料号下单选) | 高频去耦 + SD3078 VDD |
| C_VREG | 2.2µF 16V X5R 0603 | Samsung CL10A225KO8NNNC **C23630** (Basic) | VREG 退耦 |
| C_LX | 2.2nF 50V X7R 0603 | Fenghua 0603B222K500NT **C1604** (Basic) | LX snubber |

**电阻**
| 位号 | 规格 | 料号 | 备注 |
|---|---|---|---|
| R_RSET | 100kΩ 1% 0603 | UNI-ROYAL 0603WAF1003T5E **C25803** (Basic) | 电池内阻补偿 |
| R_pull ×2 | 4.7kΩ 1% 0603 | UNI-ROYAL 0603WAF4701T5E **C23162** (Basic) | I²C 上拉→VREG |
| R_CC ×2 | 5.1kΩ 1% 0603 | UNI-ROYAL 0603WAF5101T5E(*unclear*,同系列取值) | USB-C CC1/CC2 **各**下拉,**不可并** |

**机械 / 接口(完全兼容 PiSugar 2)**
| 位号 | 器件 | 料号 | 备注 |
|---|---|---|---|
| J_USB | USB Type-C 母座 16P SMD | Korean Hroparts TYPE-C-31-M-12 **C165948** | 仅供电;焊盘加大防撕脱 |
| J_BAT | 电池座 PH2.0 2P **卧贴** | JST S2B-PH-SM4-TB **C295747** | 单节锂电;立式 B2B(C160352)别选错 |
| PP1–5 | pogo 弹簧顶针 ×5 | YIYUAN YTC1P-2010-01 **C5221287**(*待核行程/电流/高度*) | 顶 Pi pin2/4=5V、pin6=GND、pin3=SDA、pin5=SCL;5V 针载 2.4A 选大电流款 |
| SW1 | SPDT 滑动开关 3P **SMD** | SHOU HAN MSK12C02 **C431540** | 电源 ON/CTRL/OFF;THT 备选 SK12D07VG4 **C393937** |
| SW2 | 轻触按键 SMD 4P 顶按 | XKB TS-1187A-B-A-B **C318884** | 自定义键 → IP5209 KEY |
| MH1–4 | M2.5 螺母 + 铜柱 ×4 | — | 对齐 Pi Zero 58×23 孔位 |

**料号性质**:标 Basic 的经 JLC Basic 清单核对(免上料费);采样电阻、USB-C、JST、滑动开关、轻触键属 Economic & Standard(常备、免上料费,有装配治具费);采样电阻无 Basic 款(必 Extended)。

**与官方/草案差异**:① 删 X1 晶振(SD3078 内置);② 采样电阻 1206→**2512**;③ 删 40pin 排针→ **pogo 顶针**;④ 删 NTC 分压(按官方 R7 NC);⑤ 新增 R_CC(USB-C 受电下拉)、SW1 滑动开关。

> **备选料**:若 IP5209 买不到,**IP5109(C181699)pin2pin + 寄存器官方兼容**,drop-in 替代(仅少 DCP,pisugar 用不到),BOM 其余不动。详见[附录 A](#附录-a--ip5209-家族选型对比)。

## 7. 待定决策

1. **焊后验证**:装好板第一件事 `i2cdetect -y 1` 确认 0x75/0x32 在线,再 `i2cdump -y 1 0x75` 对照寄存器表;0xa2/0xa3 出合理电压即电源链路 OK。
2. **pogo 顶针高度(机械关键)**:顶针自由高与压缩行程要配合 M2.5 铜柱/螺母高度,保证顶 Pi 背面焊盘有足够且不过量的压力。**第一版务必打样实测**——这是 PiSugar 形态最易翻车处。
3. **滑动开关接线**:MSK12C02 的 ON/CTRL/OFF 如何与 IP5209 KEY 配合实现硬开/关机锁定,原理图阶段参照 KEY 时序敲定(可对照真 PiSugar 2 实测行为)。
4. **SD3078 是否 Phase 1 必须**:pisugar-power-manager 要 RTC 才能调度自动开关机;若只验证电源/电量,可先跳过 SD3078,焊盘预留。
5. **LED SOC 指示**:IP5209 支持 LED 灯柱功能;Pi Zero 形态空间紧,建议留焊盘但默认不焊。

## 8. 工作量预判(Phase 1)

- **固件:几乎为零** — IP5209 硬件自治,Pi 直读寄存器。
- **PCB**:中等偏上(QFN-24 回流 + 大电流电源布局 + **pogo 顶针对位/高度**),主要工作在画板与第一版打样验证顶针接触。
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
