# 参考资料

> 第三方数据手册/开源设计仅列链接,不在本仓库再分发(尊重各自版权/许可)。

## 关键芯片数据手册
- **BQ24074**(充电+电源路径,首选):https://www.ti.com/lit/ds/symlink/bq24074.pdf
- **RT9080**(LDO,2µA Iq):立创 C841192 / Richtek 官网
- **ME6217**(对比项,800mA/100µA Iq):https://datasheet.lcsc.com/lcsc/1912111437_MICRONE-Nanjing-Micro-One-Elec-ME6217C33M5G_C427602.pdf
- **LTC4054 / TP4054(丝印 LTH7R,charger-only)**:https://www.analog.com/media/en/technical-documentation/data-sheets/405442xf.pdf
- **DW01A + FS8205A**(单节锂电保护):立创搜索
- **2N7002**(电量检测 MOS 开关):立创 C8545
- **ESP32-C3** USB Serial/JTAG Console(腾出 IO20/21):https://docs.espressif.com/projects/esp-idf/en/stable/esp32c3/api-guides/usb-serial-jtag-console.html

## 接口参考
- **微雪 e-Paper Driver HAT** 引脚(SPI 映射依据):https://www.waveshare.com/wiki/E-Paper_Driver_HAT

## 参考开源板(电源方案调研)
- **极趣实验室 ZecTrix Note4**(ESP32-S3 墨水屏笔记):buck 降压 + 开关式充电,重载多媒体设备。
- **喵哎 MiaooAim 4.2" 墨水屏**(ESP32-S3,SSD1619):TP4054 + HE9073A33,charger-only 无电源路径。https://gitee.com/gxp666111/miaomiao
- 调研结论:重载多媒体→buck;低功耗深睡→低 Iq LDO(本项目选后者)。
