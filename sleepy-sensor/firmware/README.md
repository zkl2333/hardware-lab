# firmware — ESPHome

v1 板当前运行 [ESPHome](https://esphome.io/)，配置就一个文件：[sleepy-sensor.yaml](sleepy-sensor.yaml)。

## 功能

- 每 60s 上报**电池电压**（IO4 ADC，板上 1M/1M 分压 ×2 还原）与 **WiFi 信号强度**到 Home Assistant
- 日志走原生 USB Serial/JTAG（无外置串口芯片）
- 验证期常醒（无 deep sleep）；后续计划：深睡 + 周期唤醒读 SHT40 + 上报

## 部署

配置托管在 Home Assistant 机器的 ESPHome 面板（Docker），WiFi / OTA 密码放面板 Secrets（`!secret`，不入库）：

1. **首刷**（仅一次）：面板编译 → 下载 `firmware.factory.bin` → 本机 `esptool --chip esp32c3 write_flash 0x0 xxx.factory.bin`
2. **日常**：面板里改 yaml → Install → Wirelessly（OTA）

## v1 实测要点（写进配置的硬件经验）

| 配置 | 为什么 |
|---|---|
| `output_power: 8.5dB` | **防欠压**。满功率 TX 尖峰 ≈345mA 叠加 HT7833 压差会触发 brownout 复位循环；信号 -30dBm 余量巨大，降功率纯赚 |
| `power_save_mode: none` | modem-sleep 会让 mDNS / API / OTA 时断时续，HA 显示「不可用」；验证期直接关 |
| ADC `multiply: 2.0` | 板上 BAT+ —1M— IO4 —1M— GND 常通分压（漏电仅 ~2µA） |

另一条踩坑（C3 特性）：**每次打开 USB 串口都会复位芯片**；反复开串口 + 失败重启攒满 10 次会触发 ESPHome safe mode（业务组件全停）。验证启动稳定性时让板子安静跑 90 秒以上。
