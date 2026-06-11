# 参考资料

数据手册归档在 `docs/datasheets/` 以便离线查阅与版本钉死;版权归各原厂,仅作个人工程学习存档,同表附官方来源链接。

## 已归档(`docs/datasheets/`)— 核心 BOM
| 器件 | 用途 | 本地文件 | 来源 |
|---|---|---|---|
| **BQ24074** | v2 充电+电源路径 | `BQ24074_TI.pdf` | [TI](https://www.ti.com/lit/ds/symlink/bq24074.pdf) |
| **RT9080-33** | v2 LDO(2µA Iq/600mA) | `RT9080_Richtek.pdf` | [Richtek DS9080-09](https://www.richtek.com/assets/product_file/RT9080/DS9080-09.pdf) |
| **ME6217** | LDO 对比项(100µA Iq,未选) | `ME6217_MICRONE.pdf` | [LCSC C427602](https://www.lcsc.com/product-detail/C427602.html) |
| **ETA6953** | 国产开关式电源路径(备选) | `ETA6953_钰泰.pdf` | 钰泰 |
| **2N7002** | 电量检测 MOS 开关 | `2N7002_Diodes.pdf` | [Diodes Inc](https://www.diodes.com/assets/Datasheets/2N7002.pdf) |
| **ESP32-C3** | 主控芯片 | `ESP32-C3_datasheet_Espressif.pdf` | [Espressif](https://www.espressif.com/sites/default/files/documentation/esp32-c3_datasheet_en.pdf) |
| **ESP32-C3-WROOM-02** | 主控模组 | `ESP32-C3-WROOM-02_Espressif.pdf` | [Espressif](https://www.espressif.com/sites/default/files/documentation/esp32-c3-wroom-02_datasheet_en.pdf) |

## 仅链接(未归档,点击下载)
| 器件 | 用途 | 立创编号 | 链接 |
|---|---|---|---|
| **DW01A** | 裸电芯保护 IC | 立创搜索 | 立创/各代理 |
| **FS8205A / 8205A** | 保护双 N-MOS | ≈C32254 | 立创 |
| **TP4056** | 充电(v1 备选/分体) | — | 立创 |
| **LTC4054 / TP4054(丝印 LTH7R)** | charger-only(v1) | — | [ADI 405442xf](https://www.analog.com/media/en/technical-documentation/data-sheets/405442xf.pdf) |
| **HT7833** | v1 临时 LDO(500mA) | C50936 | Holtek |
| **SHT40** | 温湿度传感器 | — | [Sensirion](https://sensirion.com/products/catalog/SHT40) |
| **USB-C 16P 母座** | 下载/供电 | 看实际选型 | 立创 |

> 注:LTC4054 的 ADI 直链下载在本机疑被杀软误删,未能归档,留链接。其余「仅链接」多为常见小料,需要时按编号一键下载即可。

## 接口 / 协议参考
- **ESP32-C3 USB Serial/JTAG Console**(原生 USB 做 console、腾出 IO20/21):https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/api-guides/usb-serial-jtag-console.html
- **微雪 e-Paper Driver HAT** 引脚(40P 排针 SPI 映射依据):https://www.waveshare.com/wiki/E-Paper_Driver_HAT

## 参考开源板(电源方案调研)
- **极趣实验室 ZecTrix Note4**(ESP32-S3 墨水屏笔记):buck 降压 + 开关式充电,重载多媒体。原理图已归档 `references/参考_ZecTrix-Note4_原理图.pdf`。
- **喵哎 MiaooAim 4.2" 墨水屏**(ESP32-S3 / SSD1619):TP4054 + HE9073A33,charger-only 无电源路径。https://gitee.com/gxp666111/miaomiao
- 调研结论:重载多媒体 → buck;低功耗深睡 → 低 Iq LDO(本项目选后者,RT9080)。
