# 第7章 常见枚举失败问题及排查

本章从嵌入式工程视角，按症状给出原因、排查思路、抓包方法、软件检查点与硬件检查点。

---

## 7.1 设备插入后 Host 检测不到设备

### 可能原因

- 无 VBUS 或设备电源未起
- D+/D− 上拉未使能，或上拉到错误的线（FS 应上拉 D+，LS 上拉 D−）
- ESD 电阻/电容过大，信号无法建立
- 连接器焊接虚焊、ID/CC 线误接到 USB 2.0 口
- Hub 端口被关掉电、过流保护动作
- 设备固件从未配置 USB 引脚复用

### 排查思路

1. 万用表测 VBUS≈5 V，设备 3.3 V 是否稳定  
2. 插入后测 D+（FS）是否被拉高到约 3.0~3.3 V  
3. 换线/换口/换 Host，排除线材与端口  
4. 确认芯片 USB_DP/DM 引脚配置与 PHY 时钟  

### 抓包分析

- 分析仪可能**完全看不到 Connect** → 问题在电气/上拉，尚未进入协议层  
- 若 Host 有 Connect 但立刻 Disconnect → 供电塌陷或复位回路问题  

### 软件检查点

- 是否调用了“使能 DP 上拉 / Connect”的 API  
- 时钟树是否开启 USB 模块  

### 硬件检查点

- 1.5 kΩ 上拉是否存在且接到正确网络  
- VBUS 电容、热插拔冲击  
- D+/D− 是否等长、有无对地短路  

---

## 7.2 Bus Reset 后设备没有响应

### 可能原因

- Reset 期间或结束后设备时钟未就绪
- UDC 未正确处理 Reset 中断，地址/端点状态错误
- 上拉过早，Host Reset 时固件还在跑 bootloader 未初始化 USB
- HS Chirp 异常导致速度状态不一致（HS 设备）
- 看门狗在枚举期复位 MCU，反复 Connect/Disconnect

### 排查思路

1. 确认 Reset 后是否进入 Default、地址寄存器为 0  
2. 用分析仪看 Reset 后第一次 SETUP 是否出现，设备有无 ACK  
3. 推迟上拉：先完成 UDC 初始化再 Connect  

### 抓包分析

```text
Reset SE0
SOF ...
SETUP ADDR=0 → 若无 ACK / 无任何响应 → 设备协议引擎未活
```

### 软件检查点

- Reset ISR 是否 `udc_set_address(0)` 并重新打开 EP0  
- 主循环是否在等 RTOS 调度才初始化 USB（过晚）  

### 硬件检查点

- 晶振起振时间、PLL lock  
- Reset 时 VDD 是否跌落  
- DP/DM 串联电阻是否符合芯片手册  

---

## 7.3 GET_DESCRIPTOR 返回 STALL

### 可能原因

- 固件未实现该 Descriptor Type/Index
- `wValue` 解析错误（大小端、Type/Index 拆分错误）
- 在错误 Recipient 下处理（如把 Interface 请求当 Device）
- 状态不允许（极少见；一般 Default 就应能读 Device Desc）
- 控制写入/读出阶段把 EP0 置成了 Halt

### 排查思路

1. 抓包看 STALL 出在 Data 还是 Status  
2. 打印 Setup 八字节，对照是否支持  
3. 确认描述符指针非 NULL、长度非 0  

### 抓包分析

```text
SETUP 80 06 xx xx ...
IN → STALL          // 设备明确拒绝提供数据
```

### 软件检查点

- `switch(type)` 是否覆盖 Device(1)/Config(2)/String(3)  
- 配置索引是否越界  
- 是否误对标准请求走了 `stall` 默认分支  

### 硬件检查点

- 较少见；若只有特定口失败，仍查信号完整性  
- 优先当固件逻辑问题处理  

---

## 7.4 只能返回前 8 字节 Device Descriptor，无法继续

### 可能原因

- Host 二次 Reset 后设备崩溃/未再枚举
- `bMaxPacketSize0` 声明与真实 FIFO 不符（如报 64 实际 8）
- `SET_ADDRESS` 失败或地址改早，后续新地址无应答
- 读完整 18 字节时分包/短包结束条件错误
- Data Toggle 错误导致 Host 丢弃数据

### 排查思路

1. 看 8 字节之后下一条：二次 Reset？SET_ADDRESS？再 GET_DESCRIPTOR？  
2. 核对返回的第 8 字节与 UDC EP0 MPS 配置一致  
3. 检查 `SET_ADDRESS` Status 是否成功，之后 Token 地址  

### 抓包分析

- 若卡在 `SET_ADDRESS`：Status 无应答或设备过早改地址  
- 若 `SET_ADDRESS` 成功但下一 SETUP（新 ADDR）无 ACK：地址寄存器未写对  
- 若完整 GET_DESCRIPTOR 数据长度错误：查短包  

### 软件检查点

- EP0 MPS 寄存器 = 描述符 `bMaxPacketSize0`  
- `pending_addr` 只在 Status 完成时提交  
- IN 传输结束条件：`sent == min(avail,wLength)` 且最后一包可以是短包或 ZLP（当正好对齐且还需结束时按栈要求）  

### 硬件检查点

- EP0 FIFO 深度是否真有 64 字节  
- DMA 长度寄存器是否可配到声明值  

---

## 7.5 SET_ADDRESS 后设备无法正常通信

### 可能原因

- **经典错误**：在 Setup 阶段就写入新地址，导致 Status（仍应在 ADDR0）失败
- 地址寄存器位域写错（有的芯片含使能位）
- Host 分配地址与设备保存不一致（多接口调试错乱）
- 二次枚举时旧地址过滤仍开启

### 排查思路

1. 抓包确认 Status 阶段 Token 地址是否为 0，且有 ACK  
2. 下一条 SETUP 是否使用新地址且被 ACK  
3. 在 Status 完成回调打日志输出“地址已切换”  

### 抓包分析

```text
SET_ADDRESS setup @ADDR0 ACK
Status IN @ADDR0 → 若 NAK 到底 / 无应答 → 地址可能已被改走
下一 SETUP @ADDR=new → 无应答 → 设备仍在听 0 或听了别的地址
```

### 软件检查点

- 严格两阶段：`pending_addr` → status complete → `udc_set_address`  
- Reset 路径清 `pending_addr`  

### 硬件检查点

- 读回地址寄存器确认写入生效  
- 多 Device Controller 实例时勿写错基址  

---

## 7.6 Configuration Descriptor 长度异常

### 可能原因

- `wTotalLength` 填错（漏算类描述符或算多）
- 返回缓冲截断（数组不够、`min` 逻辑错）
- 中间某个 `bLength` 错误，Host 解析失步
- 把 Interface/Endpoint 描述符分开存放但发送时拼接错误

### 排查思路

1. 手工累加配置集合：每个 `bLength` 之和 == `wTotalLength`  
2. 抓包对比“声明总长”与“实际 DATA 字节数”  
3. 用 Wireshark 描述符解析器看是否报 Malformed  

### 抓包分析

- Host 先读 9 字节得 `wTotalLength=0xNN`  
- 第二次读若实际字节 ≠ NN，或中途 STALL → 固件发送逻辑问题  

### 软件检查点

```c
/* 编译期或启动时断言 */
_Static_assert(sizeof(desc_config_set) == CONFIG_TOTAL_LEN, "wTotalLength mismatch");
```

- 发送长度：`min(wTotalLength, wLength)`，不要多发垃圾  
- 修改描述符后同步改 `wTotalLength` 与 `bNumInterfaces/bNumEndpoints`  

### 硬件检查点

- 一般非硬件问题；除非 DMA 长度寄存器溢出截断  

---

## 7.7 SET_CONFIGURATION 后设备无法正常工作

### 可能原因

- 非零端点未按描述符配置（类型/方向/MPS/间隔）
- 端点未使能，或仍处于 Stall
- 类驱动请求未实现（如 HID Report Descriptor 取失败）
- `bConfigurationValue` 与固件 `switch` 不一致
- 只配置了 EP，未初始化类协议状态机
- 供电不足，配置后电流上升导致复位（总线供电设备）

### 排查思路

1. 确认枚举已到 Configured（抓包见 SET_CONFIGURATION 成功）  
2. 看随后类请求/Interrupt 或 Bulk 是否 STALL/NAK  
3. 核对端点寄存器与 Endpoint Descriptor 逐项一致  
4. 测电流与 3.3 V 跌落  

### 抓包分析

```text
SET_CONFIGURATION 成功
随后 GET_DESCRIPTOR(HID Report) STALL → 报告描述符路径问题
或 IN EP1 一直 NAK → 设备未投递报告
或 TOGGLE/STALL → ClearHalt 未处理
```

### 软件检查点

- `enable_config_endpoints()` 是否被调用  
- HID/MSC 等类请求处理是否注册  
- `SET_CONFIGURATION(0)` 路径是否能关闭端点  

### 硬件检查点

- 总线供电设备的功耗与线材压降  
- 中断端点需要的定时器/采样硬件是否工作  

---

## 7.8 偶发枚举失败或不稳定

### 可能原因

- 信号完整性差、线过长、无共地
- 中断优先级/关中断过久，偶发 Setup 超时
- 任务栈溢出、内存踩踏导致描述符偶发损坏
- ESD/热插拔复位，枚举中途掉线
- HS/FS 协商临界（仅 HS 设计）
- 多线程同时操作 UDC 无锁

### 排查思路

1. 统计失败率：换短线、换口、固定温度对比  
2. 失败时抓包，看失败点是否随机  
3. 加心跳日志：Reset/Setup/SetAddr/SetCfg 序列  
4. 检查关中断时间与 USB 中断优先级  
5. 用 MPU/金丝雀值查内存踩踏  

### 抓包分析

- 随机位置超时 → 软实时/供电/信号问题优先  
- 总在同一请求失败 → 逻辑 bug  
- Connect/Disconnect 抖动 → 机械或供电  

### 软件检查点

- USB 中断可抢占长临界区  
- 描述符放在只读段，防写穿  
- Reset 可重入，状态机可从任意点拉回 Default  

### 硬件检查点

- 线材质量、接地、共模扼流  
- VBUS 与 3.3 V 去耦  
- DP/DM 布线 90Ω 差分、远离大电流开关  

---

## 7.9 排障总流程（建议打印贴在工位）

```
看不到 Connect？ ──是──▶ 查 VBUS / 上拉 / 引脚复用
        │否
Reset 后无 SETUP ACK？ ──是──▶ 查时钟 / Reset ISR / 推迟 Connect
        │否
STALL？ ──是──▶ 打印 Setup 八字节，查描述符与请求实现
        │否
SET_ADDRESS 后死？ ──是──▶ 查地址改写时机
        │否
Config 长度怪？ ──是──▶ 累加 bLength vs wTotalLength
        │否
SetCfg 后功能不行？ ──是──▶ 查非零 EP 与类协议 / 供电
        │否
偶发？ ──▶ 信号 / 供电 / 中断延迟 / 内存破坏
```

下一章汇总图表与自测题：[第8章 知识总结与学习检验](./chapter8.md)
