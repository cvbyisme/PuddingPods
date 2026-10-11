# MOONDROP PUDDING 适配说明

## 设备与连接

- 目标：**MOONDROP PUDDING / MD-TWS-056**
- HyperOS 内部兼容 Device ID：`01010607`
- Classic Bluetooth RFCOMM / SPP
- UUID：`00001101-0000-1000-8000-00805f9b34fb`
- 业务协议：Qualcomm GAIA v4

`01010607` 仅用于 HyperOS 高级耳机能力兼容，不代表真实水月雨型号 ID。

## 噪声控制

PUDDING TX：`00 1D 40 04 <mode>`；回包：`00 1D 41 04 <mode>`。

| 功能 | mode |
|---|---:|
| 关闭 | `00` |
| 自适应降噪 | `01` |
| 通透 | `02` |
| 抗风噪 | `03` |
| 基础降噪 | `04` |

三档映射：基础 → `04`，自适应 → `01`，抗风噪 → `03`。

## 电量

模块直接读取 PUDDING GAIA 电量，不把 Android 的单一 Bluetooth Battery 当作左右耳/充电盒三路电量。

- 查询：`00 1D 1A 00`、`00 1D 1A 01 01 02`
- 回包：`00 1D 1B ...`
- 类型 `1`：左耳
- 类型 `2`：右耳
- 类型 `3`：充电盒

单次回包缺少某组件时保留该组件最近一次有效值。

### GAIA 分帧：长度字段不含 4 字节帧头

PUDDING 使用 Qualcomm GAIA v4（vendor `0x1D`），帧头为 `FF 04 <len>`（两字节长度时是 `FF 04 <len_hi> <len_lo>`）。
这里的 `len` **只数 payload**，不包含 `[seq][vendor][feature][cmd]` 这 4 字节：

```
→ FF 04 00 02 00 1D 1A 01 01 02                  查询电量（len=2，帧体=6）
← FF 04 00 06 00 1D 1B 01 01 64 02 64 03 5E       电量响应（len=6，帧体=10）
                 └ seq vendor feat cmd ┘└ 01 64 ┘└ 02 64 ┘└ 03 5E ┘
                                         左耳100% 右耳100% 充电盒94%
```

即**帧体长度 = `len` + 4**，切片长度、拷贝长度与「已消费字节数」都要按这个算。
若按 `len` 切片，每条响应都会丢掉最后 4 字节：`[id][level]` 电量三元组只剩第一组，
右耳与充电盒取不到值，ANC（`0x41`）、增益（`0x1F`）、指示灯（`0x27`）的状态回读同样受影响。

## 增益与指示灯

- 增益查询：`00 1D 1E 01`
- 增益设置：`00 1D 1E 02 <level>`，`00/01/02 = 低/中/高`
- 指示灯查询：`00 1D 26 01`
- 指示灯设置：`00 1D 26 02 <state>`，`00/01 = 关/开`

## HyperOS 接入

LSPosed 作用域：

- `com.android.bluetooth`
- `com.xiaomi.bluetooth`
- `com.milink.service`
- `com.android.systemui`
- `com.android.settings`

Focus Island 采用 OppoPods 的 Xiaomi 耳机通知结构和 HyperOriG 的 `FocusNotification.buildV3` 路线。3.45 在 `com.xiaomi.bluetooth` 应用生命周期就注册广播接收器，并使用新的 `puddingpods_focus_island_v2` 通知通道，避免旧通道状态或构造函数时序导致岛消失。岛点击入口固定为 PuddingPods `PopupActivity`；弹窗“更多”再进入 HyperOS 原生 `MiuiHeadsetActivity`，因此焦点岛显示与系统耳机页面互不冲突。

系统中存在两类页面：

1. `MiuiHeadsetActivity`：HyperOS 高级耳机页面，用于兼容 `01010607`、产品图及高级耳机能力。
2. `BluetoothDeviceDetailsFragment`：系统「已配对的蓝牙设备」详情页，用于通话/媒体音频、LHDC、低延迟、与本机音量同步等系统 Profile/厂商控制。

PuddingPods 3.45 优先直接打开 `BluetoothDeviceDetailsFragment`；若 HyperOS 禁止第三方直接启动 `SubSettings`，则通过已注入 `com.android.settings` 的进程桥接，由 Settings 自己打开同一页面。


## HyperOS 原生 ANC 三档显示

`01010607` 对应的 HyperOS 模板自身带四档 ANC 控件。3.49 不删除系统 View，也不依赖修改系统 SeekBar 的 `max`；保留系统对象但隐藏原四档 slider/标签，并在相同区域放置三段 PUDDING 控件：基础 / 自适应 / 抗风噪。


## 3.45 页面路由

- Focus Island -> `PopupActivity`（不变）。
- HyperOS 官方高级耳机入口 `MiuiHeadsetActivity` -> `PuddingDetailActivity`。
- `PuddingDetailActivity` -> `BluetoothDeviceDetailsFragment` 可访问系统 LHDC、低延迟、音量同步等功能。
