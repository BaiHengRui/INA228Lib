# INA228Lib

## 📖 简介

Arduino 库，用于 Texas Instruments **INA228** 高精度数字功率/电流/电压监控器（20 位 Δ-Σ ADC）。

支持总线电压、分流电压、电流、功率、温度、能量（J/mWh）、电荷量（C/mAh）的读取，以及完整的阈值告警与诊断功能。

**特性:**
- I2C 通信（默认地址 `0x40`）
- 双 ADC 量程：±163.84 mV / ±40.96 mV（自动 SHUNT_CAL 修正）
- 分流电阻温度补偿（TEMPCOMP）
- 连续 / 单次转换模式
- 硬件能量与电荷累计
- 阈值告警：总线电压、分流电压、温度、功率
- 软复位、累计值复位

---

## 📦 安装

将仓库克隆到 Arduino `libraries` 目录，或 PlatformIO 的 `lib/` 目录：

```
git clone https://github.com/BaiHengRui/INA228Lib.git
```

依赖：`Wire.h`（Arduino 内置）。

---

## 🔌 快速开始

```cpp
#include <Wire.h>
#include "INA228.h"

INA228 ina(Wire);

void setup() {
    Serial.begin(115200);
    Wire.begin();
    ina.begin();                                  // 默认 I2C 地址 0x40

    ina.configure(INA228_AVERAGES_16,
                  INA228_CONV_TIME_1052US,
                  INA228_CONV_TIME_1052US,
                  INA228_CONV_TIME_1052US,
                  INA228_MODE_ALL_CONTINUOUS);    // 配置 ADC

    ina.calibrate(0.01f, 5.0f, ADC_RANGE_40_96mV); // 10mΩ 分流电阻, 最大 5A

    ina.setShuntTemperatureCoefficient(100);      // 100 ppm/°C（锰铜）
    ina.enableTemperatureCompensation(true);
}

void loop() {
    Serial.printf("Bus: %.3f V  Current: %.3f A  Power: %.3f W  Temp: %.1f °C\n",
                  ina.readBusVoltage(),
                  ina.readCurrent(),
                  ina.readPower(),
                  ina.readTemperature());
    delay(1000);
}
```

---

## 📚 API 参考

### 初始化

| 函数 | 说明 |
|------|------|
| `INA228(TwoWire &w)` | 构造函数，传入 `Wire` 实例 |
| `bool begin(uint8_t addr = 0x40)` | 初始化 I2C，返回是否成功 |

### ADC 配置

| 函数 | 说明 |
|------|------|
| `bool configure(avg, busConvTime, shuntConvTime, tempConvTime, mode)` | 配置平均次数、各通道转换时间、工作模式 |
| `bool setMode(ina228_mode_t mode)` | 运行时切换工作模式 |

**转换时间枚举 `ina228_convTime_t`：**

| 值 | 时间 |
|----|------|
| `INA228_CONV_TIME_50US` | 50 µs |
| `INA228_CONV_TIME_84US` | 84 µs |
| `INA228_CONV_TIME_150US` | 150 µs |
| `INA228_CONV_TIME_280US` | 280 µs |
| `INA228_CONV_TIME_540US` | 540 µs |
| `INA228_CONV_TIME_1052US` | 1052 µs |
| `INA228_CONV_TIME_2074US` | 2074 µs |
| `INA228_CONV_TIME_4120US` | 4120 µs |

**平均次数枚举 `ina228_averages_t`：**
`1, 4, 16, 64, 128, 256, 512, 1024`

**工作模式枚举 `ina228_mode_t`：**
- `INA228_MODE_SHUTDOWN` — 关断
- `INA228_MODE_*_SINGLE` — 单次触发（`BUS`/`SHUNT`/`TEMP` 及组合）
- `INA228_MODE_*_CONTINUOUS` — 连续转换（`BUS`/`SHUNT`/`TEMP` 及组合）
- `INA228_MODE_ALL_CONTINUOUS` — 全通道连续（推荐）

### 校准

```cpp
bool calibrate(float rShunt, float maxCurrent, ina228_adc_range_t range = ADC_RANGE_163_84mV);
```

| 参数 | 说明 |
|------|------|
| `rShunt` | 分流电阻值（Ω），例如 `0.005f` 表示 5 mΩ |
| `maxCurrent` | 最大期望电流（A） |
| `range` | `ADC_RANGE_163_84mV`（默认）或 `ADC_RANGE_40_96mV` |

> **ADC 量程选择：** 计算 $V_{shunt\_max} = I_{max} \times R_{shunt}$，选择刚好覆盖的量程以获得最佳分辨率。当使用 `ADC_RANGE_40_96mV` 时，库会自动将 `SHUNT_CAL` 乘以 4（符合数据手册 §8.1.2）。

### 读取测量值

| 函数 | 返回值 | 单位 |
|------|--------|------|
| `readBusVoltage()` | `float` | V |
| `readShuntVoltage()` | `float` | V |
| `readCurrent()` | `float` | A |
| `readPower()` | `float` | W |
| `readTemperature()` | `float` | °C |
| `readEnergy()` | `float` | J（焦耳） |
| `readEnergy_mWh()` | `float` | mWh（毫瓦时） |
| `readCharge()` | `float` | C（库仑） |
| `readCharge_mAh()` | `float` | mAh（毫安时） |

### 温度补偿

| 函数 | 说明 |
|------|------|
| `bool setShuntTemperatureCoefficient(uint16_t ppm)` | 设置分流电阻温度系数（ppm/°C），范围 0–16383 |
| `bool enableTemperatureCompensation(bool enable)` | 启用/禁用温度补偿 |
| `uint16_t readShuntTemperatureCoefficient()` | 读取当前温度系数 |

常见分流材料温度系数：

| 材料 | ppm/°C |
|------|--------|
| 锰铜 (Manganin) | 50–100 |
| 康铜 (Constantan) | ±30 |
| 镍铬 (Nichrome) | 150 |
| 铜 (Copper) | 3930 |

### 阈值与告警

| 函数 | 说明 |
|------|------|
| `bool setShuntOverVoltageLimit(float V)` | 分流过压阈值 |
| `bool setShuntUnderVoltageLimit(float V)` | 分流欠压阈值 |
| `bool setBusOverVoltageLimit(float V)` | 总线过压阈值 |
| `bool setBusUnderVoltageLimit(float V)` | 总线欠压阈值 |
| `bool setTemperatureLimit(float °C)` | 温度上限阈值 |
| `bool setPowerLimit(float W)` | 功率上限阈值 |

### 诊断状态

| 函数 | 返回 | 说明 |
|------|------|------|
| `isConversionReady()` | `bool` | 转换数据就绪 |
| `isAlert()` | `bool` | 任一告警触发 |
| `isMathOverflow()` | `bool` | 数学溢出 |
| `isEnergyOverflow()` | `bool` | 能量寄存器溢出 |
| `isChargeOverflow()` | `bool` | 电荷寄存器溢出 |
| `isTempOverLimit()` | `bool` | 温度超限 |
| `isShuntOverLimit()` | `bool` | 分流电压超限 |
| `isShuntUnderLimit()` | `bool` | 分流电压欠限 |
| `isBusOverLimit()` | `bool` | 总线电压超限 |
| `isBusUnderLimit()` | `bool` | 总线电压欠限 |
| `isPowerOverLimit()` | `bool` | 功率超限 |

### 控制

| 函数 | 说明 |
|------|------|
| `bool reset()` | 软件复位 |
| `bool resetAccumulation()` | 能量/电荷累计清零 |
| `bool setADCRange(range)` | 手动切换 ADC 量程 |
| `bool setAlertLatch(bool)` | 告警锁存使能 |
| `bool setAlertInvertedPolarity(bool)` | 告警极性反转 |
| `uint16_t readConfig()` | 读取 CONFIG 寄存器 |
| `uint16_t readManufacturerID()` | 制造商 ID（应为 `0x5449` = "TI"） |
| `uint16_t readDeviceID()` | 设备 ID（`0x2280` 或 `0x2281`） |

---

## ⚠️ 注意事项

1. **calibrate 与 configure 相互独立：** `calibrate()` 操作 CONFIG（ADCRANGE）和 SHUNT_CAL 寄存器，`configure()` 操作 ADC_CONFIG 寄存器（平均次数、转换时间、模式），两者互不覆盖，调用顺序任意。
2. **ADC 量程匹配：** 确保 $V_{shunt\_max} = I_{max} \times R_{shunt}$ 在所选 ADC 量程内（±163.84 mV 或 ±40.96 mV）。
3. **能量/电荷累计：** 需要连续模式下运行，`resetAccumulation()` 可随时清零。

---

## 📄 许可

GPL v3.0 © 2026 BaiHengRui
