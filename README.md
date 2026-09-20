# EvilAppleJuice-ESP32 V2.0（串口控制版）

基于 EvilAppleJuice 的 ESP32 固件，模拟 Apple 设备 BLE 广播以触发附近 iPhone / iPad 的「附近 AirPods / 连接」弹窗。

V2.0 是**串口控制版**：去掉 WiFi 后台与 Web UI，只保留一根串口线 + 一套命令行 Shell。广播设备固定为「每轮从 35 种设备表中随机选取」，不再支持指定设备与 LED 模式表。

> 固件标识：`BadAppleJuice`　配置数据结构版本：`DATA_VERSION = 4`　适用 Arduino-ESP32 **3.x** 核心

---

## 1. 硬件与接线

| 硬件 | 说明 |
|------|------|
| 主控 | ESP32-S3（同时兼容 C3 / C2 / H2 / C6，发射功率宏已按芯片区分） |
| 右 LED | GPIO 12 |
| 左 LED | GPIO 13 |
| 一键关闭脚 | GPIO 1（接地一次 = 全部关闭） |
| 一键开启脚 | GPIO 2（接地一次 = 全部开启） |
| 串口 | USB CDC / UART0，115200 8N1 |

> V2.0 已删除 BOOT 按键相关逻辑（切模式 / 复位），GPIO 9 不再使用，无需接线。

LED 行为是**固定单一灯效**，与广播状态无关：

| LED | 状态 |
|-----|------|
| 左 LED（GPIO 13） | 常灭 |
| 右 LED（GPIO 12） | 500ms 亮 / 500ms 灭，约 1 Hz 闪烁 |

> 注意：LED 刷新在 `broadcastRunning` 判断之前执行，因此即使广播处于暂停状态，右灯仍在闪烁。灯的闪烁只代表「固件在跑」，不代表「正在广播」。

### 一键开关（GPIO1 / GPIO2）

两个引脚均配置为内部上拉、低电平有效。外接按钮或拨动开关，一端接该脚、一端接 GND：

| 引脚 | 触发动作 | 效果 |
|------|---------|------|
| GPIO 1 | 接地一次（高 → 低的那一刻） | 一键全部关闭 |
| GPIO 2 | 接地一次（高 → 低的那一刻） | 一键全部开启 |

「全部」指以下四项一起**打开或一起关闭**：

1. 广播运行状态
2. MAC 地址欺骗
3. 反追踪保护
4. 短广播间隔（20ms）

> 日志输出**不在**一键开关范围内，需单独用 `ba -l` 控制。

执行后立即写入 NVS，掉电保留。

行为细节：

- **边沿触发**：只在接地瞬间动作一次，持续接地不会重复触发；松手（回到高电平）后才能再次触发。
- **开机不误触发**：上电时先记录两脚初始电平，若开机瞬间已处于接地状态，不会触发动作。
- **抖动抑制**：两次动作之间至少间隔 200ms。
- **两脚同时接地**：以「全部关闭」优先。
- 不接任何外设时不影响其它功能（内部上拉保持高电平）。

---

## 2. 编译与烧录

### Arduino IDE

1. 安装 **esp32 by Espressif Systems** 开发板包（3.x）。
2. 开发板选择：`ESP32S3 Dev Module`（其它烧录选项默认即可）。
3. 打开 `EvilAppleJuice-ESP32-INO.ino`，点击**上传**。
4. 打开串口监视器（**115200**）即可看到启动日志与 `->` 提示符。

### arduino-cli

```powershell
arduino-cli compile --fqbn esp32:esp32:esp32s3 <草图目录>
arduino-cli upload  --fqbn esp32:esp32:esp32s3 -p <串口> <草图目录>
```

实测参考占用（ESP32-S3，arduino-esp32 3.3.10）：

| 项 | 占用 |
|----|------|
| Flash | 约 606 KB（46%） |
| 全局变量 | 约 27 KB（8%） |

> 若提示 `setDeviceAddress` / `BLE_ADDR_TYPE_RANDOM` 相关编译错误，请确认核心为 3.3.x 以上。
>
> 已知限制：若草图目录路径含中文，`ld.exe` 链接阶段可能报 `cannot open output file ... .elf: No such file or directory`。将草图复制到纯英文路径编译即可。

---

## 3. 串口 Shell

上电后串口输出初始化日志，随后显示提示符 `->`，直接键入命令并回车。

| 特性 | 说明 |
|------|------|
| 波特率 | 115200 |
| 提示符 | `->` |
| 本地回显 | 支持（字符即时回显） |
| 行结束符 | `\r` / `\n` / `\r\n` 均可 |
| 退格 | 支持（`0x08` / `0x7F`，回显 `\b \b`） |
| 方向键等 ESC 序列 | 自动忽略（20ms 超时判定，避免吞掉 ESC 后的第一个字符） |
| 单行上限 | 128 字符；单行最多 16 个词元 |

命令解析按空格分词。命令名与参数均区分大小写。

---

## 4. 命令总表

### 4.1 顶层命令

| 命令 | 说明 |
|------|------|
| `help` | 显示帮助菜单 |
| `ba` | BadAppleJuice 配置程序（子命令见下） |
| `about` | 关于程序与免责声明 |
| `factory-reset` | 清除全部配置并重启（需 `y` 确认，不可逆） |

### 4.2 `ba` 子命令

| 选项 | 参数 | 说明 |
|------|------|------|
| `-rn,  --rename` | `'NAME'` | 设置设备名称（1~31 字符，可带单/双引号） |
| `-sm,  --set-mac` | `'M0' ... 'M5'` | 设置设备 MAC（6 个 `00`~`FF` 十六进制字节，空格分隔） |
| `-at,  --anti-tracking` | — | 配置反追踪（加权随机发射功率） |
| `-st,  --shorten-interval-time` | — | 配置短广播间隔（20ms） |
| `-mas, --mac-address-spoofing` | — | 配置 MAC 地址欺骗 |
| `-l,   --log` | — | 配置日志输出 |
| `-r,   --run` | — | 运行 / 暂停广播程序 |
| `-dl,  --list-devices` | — | 列出全部 35 种可选设备 |
| `-i,   --info` | — | 显示当前配置信息 |
| `-h,   --help` | — | 显示 `ba` 帮助信息 |

`-at` / `-st` / `-mas` / `-l` 四项为**交互式开关**：打印功能说明与当前状态 → 询问 `[y/n]` → `y` 翻转并保存，`n` 或 30 秒无输入视为取消。

确认过程**不阻塞主循环**：询问后立即返回，输入行由 Shell 按 `y/n` 处理，超时由主循环判定。等待期间广播继续每轮换设备，GPIO1/GPIO2 一键开关照常响应。`factory-reset` 同样如此。

`-r` **不是**交互式，执行一次即翻转运行状态并立即写入 NVS。

### 4.3 示例

```
-> ba -r                          # 开启广播（并持久化，下次上电自启动）
-> ba -i                          # 查看当前配置
-> ba -rn 'MySpeaker'             # 改名
-> ba -sm FE ED C0 FF EE 69       # 设置 MAC
-> ba -dl                         # 列出 35 种设备
-> ba -mas                        # 进入 MAC 欺骗开关的交互确认
-> factory-reset                  # 清空配置并重启
```

---

## 5. 参数与默认值

**V2.0 的所有开关默认关闭**，包括广播运行状态。也就是说刷入后不会自动广播，需要显式执行 `ba -r`。

| 项目 | 默认值 | 持久化 | 说明 |
|------|--------|--------|------|
| 广播运行状态 | 关闭 | 是 | `ba -r` 打开后会持久化，**下次通电自启动** |
| 设备名称 | `AirPods` | 是 | 1~31 字符 |
| 设备 MAC | 芯片 BT MAC | 是 | 默认取**芯片的 BT MAC**（`ESP_MAC_BT`，即 BLE 公共地址，比 esptool 打印的 base MAC 大 1）。注意欺骗关闭时广播实际不读这个值（见第 7 节），`ba -sm` 因此不生效 |
| 反追踪 | 关闭 | 是 | 开启后发射功率加权随机 |
| 短广播间隔 | 关闭 | 是 | 开启后广播间隔固定 20ms |
| MAC 地址欺骗 | 关闭 | 是 | 开启后每轮广播前生成随机源地址 |
| 日志输出 | 关闭 | 是 | 开启后最快每秒 1 行 |

### 发射功率策略

| 芯片 | 最大功率档 |
|------|-----------|
| ESP32-C3 / C2 / S3 | `ESP_PWR_LVL_P21` |
| ESP32-H2 / C6 | `ESP_PWR_LVL_P20` |
| 其它 | `ESP_PWR_LVL_P9` |

`antiTracking` 关闭时固定使用最大功率；开启时按权重随机降档：

| 概率 | 功率档 |
|------|--------|
| 70% | MAX |
| 15% | MAX-1 |
| 10% | MAX-2 |
| 4% | MAX-3 |
| 1% | MAX-4 |

---

## 6. 广播行为详解

`loop()` 每一轮的完整流程：

1. 轮询串口命令（`serialShell()`）
2. 刷新 LED
3. 若广播已暂停 → `delay(20)` 后返回本轮
4. **随机选取设备**：`ALL_DEVICES[random(0, 35)]`
5. 生成广播源地址：`macSpoofing` 开启则每轮重新生成随机静态地址（掩码字节随后端不同，见第 7 节），否则不设置、走控制器的 `PUBLIC` 地址
6. 通过 Bluedroid / NimBLE 分支写入地址
7. 生成广播数据（`generatePacket()`，长度会校验，异常则跳过本轮）
8. 设置广播类型为 `ADV_IND`（取值随后端不同，见第 7 节）
9. 若 `shortenInterval` 开启 → 间隔设为 `0x20`（20ms）
10. `start()` 开始广播
11. **持续 50~200ms 的随机时长**（期间持续轮询串口与 LED，不阻塞操作）
12. `stop()` 停止本轮
13. 应用发射功率（`applyPower()`）

### 6.1 设备永远随机

V2.0 移除了「指定设备」功能。每一轮都从 35 种设备中独立随机抽取，无法固定到某一种设备，也没有任何模式表。

### 6.2 单轮时长随机 50~200ms

```cpp
const uint32_t BROADCAST_MS_MIN = 50;
const uint32_t BROADCAST_MS_MAX = 200;
```

每轮在此区间内独立取值，因此扫完 35 种设备大约需要 1.75~7 秒，且节奏不固定——避免形成可被识别的固定广播周期。`random()` 底层走 `esp_random()` 硬件真随机，未调用 `randomSeed()`，因此每次上电序列都不同。

### 6.3 日志节流

日志开启后，只在**距上次输出 ≥ 1000ms** 时打印一行：

```cpp
const unsigned long LOG_THROTTLE_MS = 1000;
```

输出内容形如：

```
BadAppleJuice:正在广播 Airpods (长度: 31)...
```

这样即使开启 20ms 短间隔，串口输出也被限制在每秒最多 1 行，不会刷屏影响输入。

---

## 7. BLE 广播实现要点

- **PDU 类型必须为 `ADV_IND`（可连接无向广播）**，Apple 弹窗只对可连接无向广播响应。注意 `setAdvertisementType()` 的**参数含义随 BLE 后端不同**，同一个数字在两条分支里是两套枚举：

  | 后端 | 实际写入字段 | 正确取值 |
  |------|--------------|----------|
  | Bluedroid | `m_advParams.adv_type`（`esp_ble_adv_type_t`） | `0x00` = `ADV_TYPE_IND` |
  | NimBLE | `m_advParams.conn_mode`（`ble_gap_conn_mode`） | `0x02` = `BLE_GAP_CONN_MODE_UND` |

  > ESP32-S3 在 arduino-esp32 3.x 下默认使用 **NimBLE**（`sdkconfig` 中 `CONFIG_BT_NIMBLE_ENABLED=y`、`CONFIG_BT_BLUEDROID_ENABLED` 未设）。此时传 `0x00` 会把 `conn_mode` 设为 `BLE_GAP_CONN_MODE_NON`，配合默认的 `disc_mode = GEN` 发出的是 **`ADV_SCAN_IND`（不可连接）**，串口日志照常打印「正在广播」但 iOS 不会弹窗——这正是「有广播但不触发」的根因。
  > NimBLE 的 `BLEAdvertising` 构造函数默认就是 `conn_mode = UND` + `disc_mode = GEN`（即正确的 `ADV_IND`），**不调用该方法反而是对的**；本项目为跨后端显式设置，故按后端分支取值。
- **广播源地址**有两种来源：
  - `macSpoofing` **关闭**（默认）：地址类型为 `PUBLIC (0x00)`，空口上用的是**芯片的 BT MAC**（= base MAC + 1，例如 `A4:CB:8F:C5:6F:69`）。
    > 注意：BLE 公共地址由控制器硬件决定，arduino-esp32 的 BLE 库从不去设置它，因此**软件改不了**。`deviceMac` 默认值取的就是这个地址，保证 `ba -i` 显示与空口一致；但实际广播时不读 `deviceMac`，所以 `ba -sm` 在欺骗关闭时不会生效。
    > esptool 烧录日志里打印的 `MAC:` 是 base MAC（`...68`，WiFi STA 用），比 BT MAC 小 1，两者是芯片出厂时连续分配的两个地址。
  - `macSpoofing` **开启**：每轮广播前重新生成随机源地址，避免 Apple 端对固定 MAC 限速或记录后不再弹窗。此时地址类型为 `RANDOM (0x01)`。
    > **随机静态地址的掩码字节随后端不同**（这是个易错点）：规范要求地址最高两位为 `11`，但两个后端的地址缓冲区**字节序相反**（`BLEAddress.cpp` 明确注释 `NimBLE address bytes are in INVERSE ORDER!`）：
    > - Bluedroid：`esp_bd_addr_t` 按可读序存放，最高字节在 `addr[0]` → `addr[0] |= 0xF0`
    > - NimBLE：缓冲区逆序，最高字节在 `addr[5]` → `addr[5] |= 0xC0`
    >
    > 若在 NimBLE 下照抄 Bluedroid 的 `addr[0] |= 0xF0`，等于给**最低**字节加掩码，最高字节仍是随机值，`ble_hs_id_set_rnd()` 会以 `BLE_HS_EINVAL` 失败（约 75% 概率）。该函数返回值若被忽略，地址就悄悄不再轮换，而 iOS 17.2+ 恰好会压制「同一 MAC 重复广播」——表现为「广播看起来正常但就是不弹窗」。
  - Bluedroid：`pAdvertising->setDeviceAddress(addr, addrType)`
  - NimBLE（需 core ≥ 3.3.0）：先 `BLEDevice::setOwnAddr(addr)` 写入随机地址，再 `BLEDevice::setOwnAddrType(addrType)` 声明类型。
    > **两步顺序不能颠倒**：`setOwnAddrType()` 内部会先调 `ble_hs_id_copy_addr(type & 1, NULL, NULL)` 校验该类型地址是否已存在，随机地址尚未写入时会直接返回失败并停留在 `PUBLIC`，MAC 欺骗形同虚设。更早的核心无公开接口，无法伪造地址。
- **地址类型必须与地址来源匹配**：BLE 规范要求随机静态地址首字节最高两位为 `11`。芯片 MAC 形如 `A4:CB:...`，最高两位是 `10`，若仍按 `RANDOM` 类型发送可能被 iOS 直接忽略，因此固定地址一律用 `PUBLIC (0x00)`。
- 广播数据由 `generatePacket()` 生成：

| 类型 | 长度 | 结构 |
|------|------|------|
| `APPLE_AUDIO`（22 种） | 31 字节 | 7 字节头 + 1 字节 modelId（索引 7）+ 11 字节体 |
| `APPLE_SETUP`（13 种） | 23 字节 | 13 字节前缀 + 1 字节 modelId（索引 13）+ 9 字节后缀 |

- 广播数据使用 `addData((char*)packet, packetLen)` 重载，arduino-esp32 2.x / 3.x 通用，避免版本宏未定义时静默不发数据。
- 广播循环中持续轮询串口 shell 与刷新 LED，操作不卡顿。

### 弹窗触发技巧

- **务必开启 MAC 地址欺骗**（`ba -mas`）。iOS 17.2+ 的抑制逻辑是「**同一 MAC 连续发同样广告就压制**」，靠每轮换随机 MAC 才能绕开；关掉欺骗后用固定地址重复广播，iPhone 基本不会弹。
- iPhone **亮屏并打开「设置 → 蓝牙」页面**时最容易触发，静置几秒即可；锁屏状态不弹。
- 距离要近：nRF Connect 里 RSSI 应到 **-40 dBm 左右**，-68 dBm 偏弱会影响触发率。
- 可关掉「反追踪」以排除发射功率随机波动的影响（`antiTracking` 开启时约 30% 的广播会降功率）。
- 用 nRF Connect 确认广播是 `ADV_IND`（有 **CONNECT** 按钮），以及源 MAC 是否每轮都在变。
- 串口日志（`ba -l`）每行会带 `源地址:`，开着 MAC 欺骗时它**必须每行都不同**；若一直是同一个地址，说明地址轮换没生效，先查第 7 节的字节序问题。
- 若以上都确认无误仍不弹，多半是 iOS 版本侧的拦截，换个较旧的 iOS 设备再试。

---

## 8. 配置持久化

使用 `Preferences`（NVS），命名空间 `my-app`：

| 键 | 类型 | 对应项 |
|----|------|--------|
| `ver` | int | `DATA_VERSION` |
| `name` | string | 设备名称 |
| `mac` | bytes(6) | 设备 MAC |
| `at` | bool | 反追踪 |
| `st` | bool | 短广播间隔 |
| `mas` | bool | MAC 地址欺骗 |
| `log` | bool | 日志输出 |
| `run` | bool | 广播运行状态 |

启动时若 `ver != DATA_VERSION`，会判定为数据结构变更，执行 `applyDefaults()` 重新配置并保存——**旧配置会被清回出厂值**（名称回到 `AirPods`、MAC 回到默认值、所有开关回到关闭）。

`factory-reset` 直接 `preferences.clear()` 后 `ESP.restart()`。

---

## 9. 相比旧版（WiFi 控制版）的变更

| 项目 | 旧版（README.md 描述） | V2.0 |
|------|----------------------|------|
| 控制方式 | WiFi AP + Web UI + REST API | 仅串口 Shell |
| WiFi 功能 | `EAJ-Control` 热点、STA 接入、扫描 | **全部移除** |
| LED 模式 | 9 种模式表（`led.hpp`） | **删除模式表与 `led.hpp`**，固定单一灯效 |
| BOOT 按键 | 短按切模式 / 长按复位 | **删除**，GPIO 9 不再使用 |
| 广播设备 | 模式驱动 + 可指定任意设备 | **仅随机**，每轮从 35 种中抽取 |
| `-m / --mode` 命令 | 设置 LED 模式 0~8 | **删除** |
| `-d / --device` 命令 | 指定广播设备，`-1` 恢复模式控制 | **删除** |
| 单轮广播时长 | 固定 100ms | **每轮随机 50~200ms** |
| 日志输出 | 每轮打印一行（高频刷屏） | **时间节流**，最快每秒 1 行 |
| 默认值 | 部分开启 | **全部关闭**（含广播运行状态） |
| 参数校验 | `toInt()` 静默失败返回 0 | 严格解析 + `strtol` 校验，拒绝非法输入 |
| 一键开关 | 无 | **新增**：GPIO1 接地全部关闭 / GPIO2 接地全部开启 |
| 设备 MAC 默认值 | 硬编码 `FE-ED-C0-FF-EE-69` | **改为芯片的 BT MAC**（BLE 公共地址），保证 `ba -i` 与实际广播一致 |

> 关于设备数量：旧 README 写「45 种」，实际当前设备表为 **35 种**（22 Audio + 13 Setup），可用 `ba -dl` 查看。

---

## 10. 文件结构

```
EvilAppleJuice-ESP32-INO/
├── EvilAppleJuice-ESP32-INO.ino       # 全部代码：设备表 / 广播包 / 串口 Shell / BLE / 配置 / LED
├── EvilAppleJuice-ESP32-INO-V2.0.md   # 本文档
└── README.md                          # 旧版（WiFi 控制版）文档，已与当前代码不符
```

整个固件是**单文件 sketch**，不依赖任何外部 `.h/.cpp`，拷到任何目录都能直接编译。

---

## 11. 故障排查

| 现象 | 排查 |
|------|------|
| 串口无输出 | 确认波特率 115200；部分 arduino-cli 默认关闭 USB CDC，日志走 UART0，需与 IDE 设置对齐 |
| 完全无弹窗 | ① 确认已执行 `ba -r` 开启广播（默认关闭，`ba -i` 可查运行状态）；② iPhone 亮屏；③ 广播类型必须是 `ADV_IND (0x00)`；④ 用 nRF Connect 确认类型与 MAC 变化 |
| LED 一直闪但没广播 | 属正常：LED 闪烁只表示固件在运行，需用 `ba -i` 确认「广播运行状态」 |
| `ba -sm` 报字节无效 | 必须为 `00`~`FF` 的十六进制，6 个字节用空格分隔，不要带 `0x` 前缀 |
| `ba -rn` 改名后名称没变 | 当前为 NimBLE 后端时名称需重启生效，串口会给出提示；Bluedroid 后端即时生效 |
| 刷入后配置全丢 | `DATA_VERSION` 变更时会重新配置并清回出厂值，属预期行为 |
| 想恢复出厂 | 执行 `factory-reset`，确认 `y` 后自动重启 |
| 一键开关按了没反应 | 仅在「高 → 低」的瞬间动作一次，需先松手再触发；确认接的是 GND 而非 3V3；两次动作需间隔 ≥ 200ms |

---

> 本项目为 Apple BLE 邻近配对消息欺骗漏洞的 POC 验证程序，基于实验研究目的创建。
> 严禁用于非法用途，使用者需自行承担所有责任；请遵守所在地区相关法律法规。
