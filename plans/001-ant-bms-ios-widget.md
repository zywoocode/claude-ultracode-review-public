# 计划 001：ANT-BMS iOS 桌面小组件(自建伴侣 App「ANTWidget」)

- 状态:草稿
- 计划人:Claude Code(Fable 5 + ultracode 多智能体规划:2 路调研 → 3 方案并行设计 → 评审打分 → 对抗性验证,7 个智能体)
- 执行分支:`feat/001-ant-bms-ios-widget`
- 关联 PR:(开 PR 后回填)

## 1. 目标

官方 ANT BMS iOS App 没有桌面小组件。本任务自建一个极简 iOS 伴侣 App(代号 ANTWidget):通过 BLE 直连 ANT-BMS 读取电池数据(总电压、电流、SOC、温度、单体电压、MOS 状态),并提供 iOS 桌面/锁屏小组件展示「最后已知状态 + as-of 相对时间戳」,附交互式刷新按钮,可选 Live Activity 实时会话。验收后用户在 iPhone 桌面即可看到电池状态。

## 2. 背景与约束

### 为什么必须自建 App

iOS 小组件(WidgetKit extension)必须打包在宿主 App 的安装包内,第三方无法给官方 App 追加小组件。唯一路线是自建 App 自己连 BMS。

### 协议事实(多源调研,标注可信度)

- 新世代 ANT 模块用 **BLE**:GATT service `0xFFE0`,characteristic `0xFFE1`(notify + write,HM-10 式 BLE-UART 透传;个别批次可能 FFE1 notify + FFE2 write,不可硬编码)。底层串口协议 9600 8N1 透传。
- **旧世代模块用蓝牙 Classic SPP,iOS 无 MFi 认证不可达,本方案硬性无解**(只能走 Plan B,见附录)。判别法:BLE 扫描器(nRF Connect / LightBlue)中能看到广播名 `ANT-BLE` 开头、以序列号后 4 位结尾的设备 = BLE 世代。
- 协议至少存在三个方言,**全部字段偏移目前是二手情报,必须以 M0 真机抓帧 + 对照 `syssi/esphome-ant-bms` 源码(`components/ant_bms/ant_bms.cpp` + `docs/ANT_communication_protocol_EN.1.pdf`)为准**:
  1. 旧协议:发 `DBDB00000000`(6 字节)→ 140 字节响应,帧头 `AA 55 AA FF`;单体电压 2 字节大端 mV 顺排;电流 4 字节有符号约偏移 70;SOC 约偏移 74;末 2 字节为 16 位补码求和校验。
  2. 「2021」协议:`5A5A5A00005A` + `DBDB00000000` 握手,`7E A1` / `AA 55` 混合帧。
  3. `7E A1` TLV 框架(patman15/BMS_BLE-HA 支持的现行固件)——可能与 2 是不同的方言。
- BLE 通知会把帧分片,解析前必须按帧长重组。
- 参考实现:`syssi/esphome-ant-bms`(最完整)、`patman15/BMS_BLE-HA`、`juamiso/ANT_BMS`、`Louisvdw/dbus-serialbattery`。

### iOS 平台硬约束(不可绕过,决定产品语义)

- WidgetKit 小组件**没有实时刷新**:每日约 40–70 次时间线刷新预算,小组件进程自身不能做网络/BLE。产品语义只能是「最后已知值 + 相对时间戳」。
- **iOS 后台 App 没有定时器**:挂起后 `Timer` 不触发。ANT 协议是纯请求-响应(不写轮询命令就没有 notification),所以「后台保持连接就能持续拿数据」**不成立**——必须用「事件链式轮询」等对策(见核心思路第 4 条)。
- App Group 共享容器是 App → Widget 的唯一数据通道;免费 Apple ID 无 App Group、无稳定后台 BLE 授权,**正式版必须付费开发者账号($99/年)**。但 M0 抓帧诊断 App 仅需前台 BLE,免费账号即可跑——**付费决策后移到 M0 通过之后**。
- Live Activity(实时秒级显示)单会话上限 8 小时活跃,需用户手动开启,iOS 16.1+;交互式小组件按钮需 iOS 17+。
- 多数 BMS 模块只接受单个 BLE 连接:**本 App 连接期间官方 App 连不上**,反之亦然。

### 用户前提(任一不满足则主方案不可行)

1. BMS 是 BLE 世代(决策门 0 验证,见执行指令 M0)。
2. 一台可跑近期 Xcode 的 Mac + iPhone(iOS 17+ 最佳)。
3. M0 通过后愿意付 Apple Developer Program $99/年。
4. 手机需在 BMS 蓝牙范围内(BMS 装在金属箱/车体内时实际约 2–5 m);超范围数据即冻结。远程查看不在本方案能力内(见 Plan B)。

### 用户须事先接受的预期

- 桌面小组件 = 最后已知值,典型 15–60 分钟一次可见更新;秒级实时只在打开 App 或 Live Activity 会话中。**这是 iOS 平台限制,不是方案缺陷;任何方案都做不到桌面秒级跳动。**
- 上滑手动杀 App 后小组件冻结,直到下次打开 App;低电量模式下刷新暂停;锁屏期间更新可能延后到解锁。
- 需要用官方 App 时,先在本 App(或其小组件快捷动作)里断开连接。

## 3. 核心思路

评审胜出方案:**务实混合型**——极简 SwiftUI 宿主 App(只做 BLE 采集与设置)+ App Group 数据管线 + 三层展示面。执行方不得偏离本节;发现不可行即停止反馈。

1. **架构与数据流**

   ```
   ANT-BMS (BLE 0xFFE0/0xFFE1)
      │ write(轮询命令) / notify(分片响应)
      ▼
   宿主 App:CBCentralManager(bluetooth-central 后台模式 + 状态恢复)
      FrameAssembler(按帧长重组分片) → 协议方言插件 Decoder → BmsReading
      ▼
   App Group 共享容器(最新一条 + 近 N 条历史,文件保护级别 .completeUntilFirstUnlock)
      ├─▶ WidgetKit TimelineProvider(small/medium + 锁屏 accessory 档)
      ├─▶ AppIntent「立即刷新」按钮(两段式,见第 5 条)
      └─▶ ActivityKit Live Activity(用户手动开启实时会话,本地 update,无服务器)
   ```

2. **协议层 = 方言插件 + 纯函数 + fixture 单测**。三个方言(旧 140 字节 / 2021 / 7E A1)各自独立解析器与抓包 fixture 集,M0 抓帧结果决定实现哪个。校验和 + 物理不可能值双保险(SOC>100、电压超界丢弃);连续 3 帧解析失败标记「协议不匹配」状态而非静默丢弃。连接后枚举特征值按属性动态绑定,不硬编码 UUID。

3. **v1 硬性只读**。写入白名单仅含已验证的轮询命令;探测顺序固定「新→旧」、逐条等超时后再发、绝不并发。**任何控制类功能(MOS 开关、参数写入)永久排除在本项目范围外**——对象是大电流锂电池,误写风险不可接受。

4. **后台采集 = 事件链式轮询(对抗性验证 A1 的对策①)**:收到响应帧的后台唤醒窗口内,用 `beginBackgroundTask` 撑住并立即写下一条轮询,形成约 30s 一轮的自维持循环;提供开关并在 M3 实测耗电,不可接受则降级(删除该档新鲜度承诺,只留 pending-connect 自动重连 + BGAppRefreshTask 兜底 + 交互按钮)。不依赖后台扫描(广播包未必含 0xFFE0);首次前台连接后持久化 peripheral identifier,此后一律 `retrievePeripherals` + pending connect。

5. **交互刷新按钮两段式**:连接存活 → 轮询等待 ≤5s 返回新值;冷连接 → 只发起 pending connect 并立即返回「连接中…」entry,数据到达后由 BLE 唤醒回调写入 + reload。不在 Intent 窗口内同步等冷连接。

6. **小组件 stale 语义预烘焙**:每次 `getTimeline` 生成多条未来 entry(+30min、+60min 各一条灰化/警示样式),App 死掉后系统仍能按计划切到 stale 渲染,不依赖任何刷新预算。stale 文案区分「数据过旧」与「连接中断」;BMS 休眠停止广播时提示需物理唤醒。

7. **技术栈**:Swift/SwiftUI + WidgetKit + ActivityKit + CoreBluetooth,零第三方依赖。分发默认 TestFlight(90 天/build,Xcode Cloud 免费额度自动续传),备选 development 签名(1 年,单用户更省事)。

## 4. 执行指令(交给执行方的完整任务书)

> **执行要求:ChatGPT 桌面 app Work 模式,模型必须选用 GPT-5.6 最高推理档位(extended/max reasoning),不得降档。**
>
> 本节自包含:执行方只读本节 + 仓库代码即可开工,不依赖对话上下文。
>
> 重要背景:执行方无法接触真机——不能跑 Xcode、不能连 BMS。每轮调试都是「用户操作 → 导出日志粘贴给执行方 → 执行方改码」的人肉回环,因此 M0 的诊断 App 是一切调试的中枢,必须最先交付且日志一键导出为纯文本。

### M0(决策门,免费 Apple ID 即可,1–2 天)——BLE 诊断/抓帧 App

1. 新建 Xcode 工程 `ANTWidget`(iOS 17 target,先不加任何 extension/capability,免费账号可签)。产出「用户手工步骤清单」文档:Xcode 安装、真机运行信任、(M1 前)开发者账号注册(身份验证可能 1–3 天,与 M0 并行启动)、App Group 与 bluetooth-central capability 配置、TestFlight 配置——这些执行方无法代劳。
2. 实现诊断页:BLE 扫描列出设备(名称/UUID/RSSI/广播包原始内容);连接 `ANT-BLE*` 设备后枚举全部 service/characteristic 及属性;可发送预置轮询命令(仅白名单:`DBDB00000000`;`5A5A5A00005A`+`DBDB00000000`,逐条等超时);以时间戳 + raw hex 记录全部收发字节,**一键导出文本**。
3. 用户跑诊断 App 完成决策门 0:
   - 扫不到 `ANT-BLE*` → BMS 是 Classic 旧世代,**主方案终止,转 Plan B(附录)**。
   - 扫到 → 抓 ≥20 帧完整响应,记录:广播包是否含 0xFFE0、特征值读写属性分布、冷连接全链路耗时(唤醒→连接→发现→订阅→写→收全帧)。
4. 执行方对照 `syssi/esphome-ant-bms` 的 `components/ant_bms/ant_bms.cpp`、`docs/ANT_communication_protocol_EN.1.pdf`、issue #20,以及 `patman15/BMS_BLE-HA` 的 ANT 解析器,逐字节核对用户抓帧,确定协议方言与字段偏移表,写入 `PROTOCOL.md` 并把抓到的真帧存为测试 fixture。**此前不得承诺 M1 工期。**
5. M0 通过(方言确定、可稳定解析)→ 用户付费 $99 注册开发者账号,进入 M1。

### M1(2–3 天)——宿主 App + 协议层

1. 协议层为独立 Swift Package/模块:`FrameAssembler`(分片重组)+ 方言 `Decoder`(纯函数,输入 `Data` 输出 `BmsReading{totalV, current, soc, socAh, temps[], cellV[], mosfetFlags, balancing, timestamp}`),用 M0 真帧 fixture 写单测,校验和 + 物理不可能值双门控。
2. `BleManager`:CBCentralManager,`CBCentralManagerOptionRestoreIdentifierKey` 状态恢复;首连后持久化 peripheral identifier,重连一律 `retrievePeripherals` + pending connect;特征值按属性动态绑定;写入白名单硬编码为仅两条轮询命令(代码层禁止其他写入)。
3. 主界面:连接状态、全量读数(总压/电流/功率/SOC/温度/单体电压列表/最高最低压差/MOS 状态)、显眼的「断开连接」按钮(onboarding 明确提示与官方 App 互斥)、保留 M0 诊断页入口。
4. App Group(`group.<bundleid>`):写入最新 `BmsReading` + 近 24h 采样历史;共享文件显式设 `.completeUntilFirstUnlock` 并加「锁屏状态下后台写入」测试用例;每次成功写入后 `WidgetCenter.reloadTimelines`。

### M2(1–2 天)——WidgetKit 小组件

1. Widget extension,三档:systemSmall(大字 SOC + 电压)、systemMedium(SOC/电压/电流/温度 + 迷你趋势)、accessoryCircular/Rectangular(锁屏)。信息层级:大字 SOC,小字电压/电流/温度,`Text(date, style: .relative)` 相对时间戳。
2. `getTimeline` 每次生成当前 entry + 未来 stale entry(+30min 灰化、+60min 警示图标),stale 文案区分「数据过旧」/「连接中断」。
3. 充电中/放电中/保护触发的状态角标;低 SOC、过温同时触发本地通知(`UNUserNotificationCenter`,后台 BLE 唤醒窗口内发,不依赖小组件刷新)。

### M3(1–2 天,不确定性最大)——后台采集

1. 事件链式轮询:notify 回调唤醒窗口内 `beginBackgroundTask` → 写下一条轮询,循环;App 内提供「后台持续采集」开关,默认开。
2. BGAppRefreshTask 兜底:注册每日数次;task 内 20s 硬超时,超时只写「连接失败」状态,绝不留旧数据装新。
3. 用户实测 24h 耗电并回报;不可接受 → 关闭链式轮询默认值,降级方案:pending connect 自动重连(回到范围即更新一次)+ 兜底 + 手动,同步修改 App 内新鲜度说明文案。

### M4(0.5–1 天)——交互刷新按钮

AppIntent 两段式(见核心思路第 5 条);小组件加「断开连接」快捷动作(与官方 App 切换用)。

### M5(可选推荐,1–2 天)——Live Activity

用户手动开启「实时会话」(骑行/充电场景):锁屏 + 灵动岛显示 SOC/电流/功率,本地 `Activity.update()`(前台秒级,后台 BLE 唤醒窗口内十秒级),8h 上限到期自动结束并提示。无服务器、不接 APNs。

### M6(1 天)——打磨与分发

Onboarding(BLE 权限、互斥提示、新鲜度预期说明原文采用本计划「用户须事先接受的预期」)、TestFlight 上传、Xcode Cloud 自动续传配置(或 development 签名说明)。

### 通用要求

- 全程 Swift 5.9+/SwiftUI,零第三方依赖;协议层单测覆盖全部 fixture;每个里程碑单独 commit,PR 引用本计划文件。
- 遇到与本计划「核心思路」冲突的情况,停止执行并在 PR/对话中反馈,不得自行改方案。

## 5. 验收标准

- [ ] M0:诊断 App 能扫描、连接、抓帧并一键导出日志;`PROTOCOL.md` 含经真机验证的方言与偏移表;fixture 入库。
- [ ] M1:真机上 App 显示与官方 App 一致的电压(±0.1V)/电流/SOC/温度/单体电压;协议层单测全绿(`xcodebuild test`);断开按钮后官方 App 可正常连接。
- [ ] M2:桌面添加小/中/锁屏小组件,显示最新值与相对时间戳;强杀 App 后 30/60 分钟小组件自动转灰化/警示样式(预烘焙 entry 生效);手机锁屏时后台写入成功(解锁后可见更新)。
- [ ] M3:App 退后台且屏幕锁定 1 小时,期间小组件至少更新 2 次(链式轮询存活);24h 耗电用户可接受,或已按降级方案调整文案。
- [ ] M4:连接存活时点小组件刷新按钮 ≤5s 出新值;冷连接时显示「连接中…」且不卡死,连上后自动补数。
- [ ] M5(如实施):Live Activity 会话中锁屏数值秒~十秒级跳动。
- [ ] 全程无任何非白名单写入命令(代码审查确认)。

## 6. 审查要点(给三方审查用)

- **协议解析正确性**:字段偏移必须对照 fixture 与 esphome 源码逐字节核对;大小端、有符号电流、校验和算法(16 位补码求和)最易错。
- **后台生命周期**:链式轮询是否真的在 `didUpdateValueFor` 唤醒窗口内完成「写容器 + 发下一轮询」;状态恢复 delegate 是否完整实现;`beginBackgroundTask` 是否成对 end。
- **锁屏 Data Protection**:App Group 写入的文件保护级别是否 `.completeUntilFirstUnlock`,UserDefaults 若用到需实测锁屏行为。
- **只读白名单**:确认无任何路径能发出白名单外的写入(包括调试页)。
- **stale 预烘焙**:`getTimeline` 是否真的返回多条未来 entry 而非依赖 reload。
- **Intent 超时路径**:冷连接下是否两段式返回而非同步阻塞。
- **边界条件**:分片乱序/半帧断连、单体数 8–32 可变、负电流(充电)、温度负值、SOC 0/100、BMS 休眠断链。

## 7. 明确不做(Out of Scope)

- 任何写入/控制功能:MOS 充放电开关、参数修改、均衡控制——永久排除,不只是 v1。
- 蓝牙 Classic 旧世代 BMS 支持(iOS 平台不可行,走 Plan B)。
- 服务器/APNs 远程推送、远程(超蓝牙范围)查看、多 BMS 同时管理、Android 版、上架 App Store 审核发布。
- 官方 App 的任何逆向修改或重打包。

## 附录:Plan B——ESP32 + ESPHome + Home Assistant 桥接(不写代码路线)

触发条件(任一):① 决策门 0 判定 BMS 为 Classic 旧世代(ESP32 可走 UART-TTL 四线直连,绕开蓝牙世代问题);② 用户不愿付 $99/年;③ 需要「手机不在电池旁也能看」的远程查看;④ M3 链式轮询实测耗电不可接受且降级后体验不满足。

路线:ESP32(约 ¥20)刷 `syssi/esphome-ant-bms` → 接入 Home Assistant → 用 HA Companion iOS App 的通用小组件展示,关键告警走 HA 推送。优点:零 Swift 代码、协议实现最成熟、数据链路真实时、免 Apple 开发者账号;代价:需 ESP32 硬件 + 常开 HA 主机 + WiFi 覆盖,小组件为 HA 通用模板样式(非定制),且引入家庭服务器运维。如触发,另立计划 002 细化。
