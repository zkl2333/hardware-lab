# sleepy-sensor

ESP32-C3 的「树莓派 Zero 形态」低功耗开发板 —— 兼做温度计(SHT40),可接树莓派墨水屏 HAT。

- **主控**:ESP32-C3-WROOM-02(原生 USB,无需 CP2102)
- **外形**:树莓派 Zero 兼容(65×30mm,标准 M2.5 孔位 + 40P GPIO 排针),可用 Pi 外壳/HAT
- **目标**:低功耗深睡 · 开发板 · 温度计 · 墨水屏接口
- **工具**:EasyEDA Pro(硬件) + 立创/嘉立创打样
- **电池**:单节锂电 102540 / 1100mAh(裸电芯,需板载保护)

## 路线图

| 版本 | 内容 | 状态 |
|---|---|---|
| **v1** | 已打样;弱 LDO(HT7533 100mA)导致重启 → 换 HT7833 救活,验证最小系统 | 等器件 |
| **v2** | 一体板:BQ24074 电源路径 + RT9080 LDO + 裸电芯保护 + 无漏电电量检测 + BOOT 按键 + 排针 SPI 映射 | **选型定稿,待落地** |
| **v3** | 分体:无电池主板 + 独立可堆叠电源 HAT(树莓派 UPS 式) | 构想 |

设计细节与选型理由见 **[docs/design.md](docs/design.md)**,参考资料见 **[docs/references.md](docs/references.md)**。

## 目录结构

```
sleepy-sensor/
├── docs/         设计文档、选型理由、参考链接
├── hardware/     EasyEDA 工程导出快照(.epro)
└── firmware/     固件(待开始)
```

## 硬件版本管理

EasyEDA Pro 工程在云端;通过「导出 → .epro」把工程快照提交进 `hardware/`,即可对硬件设计做版本化、回滚。
