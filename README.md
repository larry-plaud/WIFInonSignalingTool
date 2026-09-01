# WiFi Non-Signaling Test Tool

> WiFi 射频**非信令（Non-Signaling）**自动化测试上位机 —— 基于 R&S CMW500 与 Keysight N9020A，一键完成 WLAN 发射（TX）指标与二次谐波测量，并导出 Excel 报表。

一个面向产线 / 研发的 Windows 桌面工具（WPF · .NET 8）。它同时联动三类设备：

- **R&S CMW500** 综合测试仪（VISA / SCPI over TCP）—— 测量 TX 突发功率、EVM、频偏；
- **Keysight N9020A（MXA）** 频谱分析仪（SCPI over TCP）—— 测量基波 / 二次谐波与 THD；
- **被测设备 DUT**（Realtek Ameba 系列，MP 工程模式）—— 通过串口下发 `iwpriv` 指令控制发射。

工具在非信令模式下驱动 DUT 直接发包，无需与仪表建立信令连接，配置一次即可批量遍历多制式、多信道，自动采集并判读结果。

---

## 目录

- [功能特性](#功能特性)
- [界面概览](#界面概览)
- [硬件与连接](#硬件与连接)
- [软件环境](#软件环境)
- [快速开始](#快速开始)
- [测试流程](#测试流程)
- [默认测试矩阵](#默认测试矩阵)
- [增益表（gainset）导入 / 导出](#增益表gainset导入--导出)
- [报表导出](#报表导出)
- [从源码构建](#从源码构建)
- [自动发布（CI）](#自动发布ci)
- [常见问题](#常见问题)

---

## 功能特性

- **TX 发射测试**：逐条配置 DUT（带宽 → 信道 → 速率 → 功率）并启动连续发包，经 CMW500 测量 **突发功率（dBm）**、**EVM RMS**（11b 为 %、OFDM/HT 为 dB）、**中心频偏（Hz）**。
- **二次谐波测试**：DUT 发射基波，N9020A 谐波测量套件读取基波与 H2 功率，按线损（Fund / H2）分别补偿后给出 **H2（dBc / dBm）** 与 **THD（%）**。
- **多制式一键遍历**：勾选制式即批量运行，覆盖 2.4G / 5G 的 11b / 11g / 11a / 11n（HT20 / HT40）。
- **手动 / 自动两种模式**：自动模式连续跑完选中矩阵；手动模式逐条“设定 / 下一条”，仅下发 DUT 指令，便于调试。
- **运行控制**：开始 / 暂停 / 继续 / 停止，实时进度条与运行状态。
- **增益表管理**：`gainset.xlsx` 一键导入 / 导出，导入后永久保存到 `%APPDATA%`，即时覆盖默认功率。
- **Excel 报表**：TX / 谐波结果连同仪表 IDN、线损等元信息导出为带样式的 `.xlsx`。
- **实时日志**：完整记录 VISA / N9020A / DUT 的每一条收发指令，便于排障；支持清空。
- **深色主题界面**：紧凑的单窗口布局，状态指示灯与 PASS/FAIL 配色。
- **单文件绿色发布**：自包含单 EXE，目标机器无需预装 .NET 运行时。

## 界面概览

单窗口，自上而下分区：

| 区域 | 说明 |
|------|------|
| CMW500 Connection | 选择 / 输入 VISA 地址，连接后自动下发默认测量配置；含 gainset 导入 / 导出按钮 |
| N9020A Connection | 仅在「二次谐波」测试类型下显示，填写 IP 连接 |
| DUT Connection | 选择 COM 口与波特率，Connect → Initialize；含手动模式配置步进器 |
| Test Configuration | 选择测试类型、线损 / 谐波参数，勾选待测制式，Start / Pause / Stop |
| Result | 进度条 + TX 或谐波结果表格 + Export Excel |
| Log | 全量指令收发日志 |

## 硬件与连接

```
             ┌──────────────────────┐
             │  上位机 (本工具, Win) │
             └──────┬───────┬───────┘
              串口   │       │ 以太网 (VISA/SCPI, TCP)
        ┌───────────┘       └──────────────┬───────────────┐
        ▼                                  ▼               ▼
┌───────────────┐               ┌──────────────────┐  ┌──────────────┐
│ DUT (Ameba MP)│──── RF ──────▶│ R&S CMW500        │  │ Keysight     │
│  iwpriv 指令   │   (射频线)     │ (TX 测量)         │  │ N9020A (谐波)│
└───────────────┘               └──────────────────┘  └──────────────┘
```

- **仪表通信**：原生 TCP Socket 实现 VISA 风格资源串，默认端口 `5025`。
  - 资源串格式：`TCPIP0::<IP>::INSTR`（内置常用地址下拉，可自定义）。
  - N9020A 只需填写 IP，程序自动补全为 `TCPIP0::<IP>::INSTR`。
- **DUT 通信**：标准串口，8-N-1，默认 `1500000` 波特（可选 9600 ~ 3000000），换行 `\r\n`。
- **射频连接**：DUT 天线口经射频线缆接入 CMW500 / N9020A，线损在界面填写并参与补偿（TX 用 Cable Loss；谐波用 Loss Fund / Loss H2）。

## 软件环境

- **运行环境**：Windows 10 / 11（x64）。
  - 使用官方发布的单文件 EXE：**无需**安装 .NET 运行时。
  - 从源码运行：需 **.NET 8 SDK**（`net8.0-windows`，WPF）。
- **DUT 侧**：进入 MP（Manufacturing / 工程测试）模式，支持 `iwpriv mp_*` 指令集（Realtek Ameba 平台）。

## 快速开始

1. 下载并运行发布版 EXE（或见 [从源码构建](#从源码构建)）。
2. **连接 CMW500**：选择 / 输入 VISA 地址 → `Connect`。连接成功后工具自动执行 `*RST` 并下发默认测量配置（Free Run 触发、ENP=30 dBm、线损等），状态变为 `Ready`。
3. **连接 DUT**：选择 COM 口与波特率 → `Connect` → `Initialize`（进入 Ameba MP 模式）。
4.（谐波测试时）**连接 N9020A**：先把「Test Type」切到「二次谐波」，填 IP → `Connect`。
5. 填写 **Cable Loss**（及谐波的 Loss Fund / Loss H2 / Ref Level）。
6. 在 **Modes** 勾选要测的制式。
7. 点 **Start** 开始，进度条与结果表实时更新。
8. 完成后 **Export Excel** 导出报表。

## 测试流程

**TX（每条配置）**：
1. 停止上一次发射 → 2. DUT 配置（带宽/信道/速率/功率）→ 3. 启动连续发包（`mp_hwtx`）→ 4. CMW500 设置制式 / 带宽 / 频段 / 频率（含回读校验）→ 5. `INIT` 测量并轮询 `RDY` → 6. 读取并解析平均结果（DSSS / OFDM 字段序不同）→ 7. 停止发射。

**二次谐波（每条配置）**：
1. 停止上一次发射 → 2. DUT 配置并启动发包 → 3. N9020A 配置谐波套件（基波频率、谐波数=2、参考电平）→ 4. `INIT` 测量 → 5. 读取基波 / H2 幅度、频率、THD → 6. 线损补偿：`Fund_DUT = Fund_raw + LossFund`，`H2_DUT = H2_abs + LossH2`，`H2(dBc) = H2_DUT − Fund_DUT` → 7. 停止发射。

> 提示：H2 频率更高，其线损通常应大于基波线损；若二者相同工具会在日志给出告警。

## 默认测试矩阵

自动模式下按勾选制式从内置矩阵筛选（共 21 条，功率可被 gainset 覆盖）：

| 制式 | 频段 | 带宽 | 速率 | 信道 |
|------|------|------|------|------|
| 11b  | 2.4G | 20M | CCK-11M  | 3 / 7 / 11 |
| 11g  | 2.4G | 20M | OFDM-54M | 3 / 7 / 13 |
| 11n HT20 | 2.4G | 20M | MCS7 | 3 / 7 / 13 |
| 11n HT40 | 2.4G | 40M | MCS7 | 3 / 7 / 11 |
| 11a  | 5G   | 20M | OFDM-54M | 36 / 100 / 149 |
| 11n HT20 | 5G | 20M | MCS7 | 36 / 100 / 149 |
| 11n HT40 | 5G | 40M | MCS7 | 38 / 102 / 159 |

速率名到 DUT `mp_rate` 码值的映射见 `DutSerialService.RateCode()`（CCK / OFDM / MCS 全覆盖）。

## 增益表（gainset）导入 / 导出

用于按行覆盖各测试项的发射功率（DUT `mp_txpower` 的 `patha`）。

- **文件格式**：`.xlsx`，三列 —— `速率` / `信道号` / `gain`，**行顺序须与默认矩阵一致**（21 行，可含表头）。
- **导出模板**：点「导出gainset」，把当前配置写成模板便于编辑。
- **导入生效**：点「导入gainset」，解析 gain 列后**复制并永久保存**到
  `%APPDATA%\WIFInonSignalingTool\gainset.xlsx`，后续启动自动加载。
- **加载优先级**：用户导入文件（`%APPDATA%`）> 随程序发行的 `gainset.xlsx`（EXE 同目录）> 内置默认值。

## 报表导出

导出的 `.xlsx` 含：

- **Info** 表：生成时间、CMW500 IDN、Cable Loss；谐波测试另含 N9020A IDN 与 Loss Fund / H2。
- **TX** 表：Mode / BW / Rate / Channel / Freq / Gain / Burst Power / EVM RMS / Freq Error / Status（状态按颜色标注）。
- **Harmonic** 表：Mode / BW / Channel / Freq / Gain / Fund Raw / Fund DUT / H2 Freq / H2(dBc) / H2(dBm) / THD / Status。

## 从源码构建

```bash
# 需要 .NET 8 SDK
cd src

# 运行
dotnet run

# 发布：自包含 · 单文件 · ReadyToRun（x64）
dotnet publish -c Release -r win-x64 --self-contained true \
  -p:PublishSingleFile=true \
  -p:IncludeNativeLibrariesForSelfExtract=true \
  -p:EnableCompressionInSingleFile=true
```

产物为单个自包含 EXE，目标机无需安装 .NET 运行时。

## 自动发布（CI）

`.github/workflows/release.yml`：推送 `v*` / `V*` 标签即自动构建单文件 EXE 并发布到 GitHub Releases（产物形如 `WIFInonSignalingToolV<版本>.exe`）。手动触发（`workflow_dispatch`）仅上传构建产物、不创建 Release。

```bash
git tag v2.0
git push origin v2.0   # 触发构建与发布
```

**技术栈**：WPF · MVVM（CommunityToolkit.Mvvm）· System.IO.Ports · ClosedXML。

## 常见问题

- **连接 CMW500 失败**：确认 IP / VISA 地址与网络可达，端口 `5025` 未被占用；仪表处于 SCPI 远程可访问状态。
- **DUT 无回复 / 初始化失败**：确认 COM 口与波特率正确、DUT 已进入 MP 模式且支持 `iwpriv mp_*`。
- **结果为 `No Signal` / `--`**：检查射频连接与线损设置、DUT 是否正在发包、Expected Nominal Power 是否匹配。
- **CMW STAN mismatch**：制式回读与设定不符，通常为固件版本对 SCPI 枚举值支持差异（本工具基于 CMW V3.7.170 实测字段序解析）。
- **谐波结果偏乐观**：检查 Loss Fund / Loss H2 是否按实际线缆分别标定（H2 线损一般更大）。
