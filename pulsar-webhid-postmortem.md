# Pulsar 鼠标 Web 驱动 · 两个会话轨迹复盘（公开版）

> 分析对象：一个本地工作区下的两个 DSH 会话轨迹
>
> | # | 会话标题 | 会话 ID | 起始时间（+08:00） | 记录条数 | turn 数 | 模型步骤数 | **执行模型** |
> |---|---|---|---|---|---|---|---|
> | 一 | **Pulsar 鼠标 Web 驱动开发** | `SESSION-A` | 2026-10-18 10:23:41 | 4537 | 11 | 358 | DeepSeek（`deepseek-flash`）全程 |
> | 二 | **修复鼠标Web驱动休眠误报问题** | `SESSION-B` | 2026-10-18 13:58:29 | 730 | 3 | 102 | **turn 1 = Kimi（`kimi-k3`，opencode go）**；turn 2–3 = DeepSeek |
>
> 本文回答三个问题：**模型做了哪些步骤、进行了什么思考、最终达成了什么结果**。
> 证据约定：`[seq N]` 为会话记录内的全局序号。
>
> ---
>
> ### 公开版删减说明（**请先读这一段**）
>
> 本文是**个人学习与互操作性研究的复盘**，不是教程、不提供任何可用于绕过产品限制的实现细节。为公开发布，本版相对内部版做了以下删减：
>
> | 删减项 | 说明 |
> |---|---|
> | **厂商私有协议的字节级细节** | 帧布局、校验算法、命令类/命令字编号、状态码取值、回报率映射表等**全部移除**，只保留"请求—轮询应答 + 命令回显"这一结构性描述 |
> | **设备与接口标识** | 厂商/产品 ID、接口序号对应的具体数值、设备唯一 UUID、固件与接收器版本号等**全部移除或泛化** |
> | **原始报文** | 真机抓包的十六进制报文与校验和样例**全部移除** |
> | **个人与环境信息** | 会话 ID、用户名、机器绝对路径、公网 IP、第三方账号/订阅信息中的可识别项等**已脱敏** |
> | **自相矛盾的表述** | 原内部文稿含有与公开发布相冲突的原始用途说明，已改写为中性表述 |
>
> **保留的内容**：方法论（如何判断一个软件是否被打包、如何选择静态分析路线）、团队编排与工程实践、以及**不同模型在同一类问题上的行为差异与代价对比**——这些才是本文的价值所在。
>
> 本文与 **Pulsar Gaming Gears** 及其关联公司无任何关联，非官方、未获授权、未获认可；**不包含**任何厂商软件、固件、二进制或配置文件的副本。如权利人认为本文内容不妥，请联系我，我会立即处理。

---

## 0. 数据来源与分析方法

**原始数据**：DSH 把每个工作区的会话落盘在
`C:\Users\<用户名>\.dsh\sessions\<工作区槽位>\<会话ID>\session.v4.jsonl.zstd`
（zstd 压缩的 JSONL，含每条模型思考全文 `reasoning`、工具调用参数、工具结果、团队事件、待办、工作区变更快照）。本次分析把两个会话解码为纯文本轨迹，再按 turn/步数切成 `dev-c01..c08` + `sleep-c01..c02` 共 **10 个**压缩片段（更早一代按 170 KB 分块得到 `dev-chunk-01..11` + `sleep-chunk-01..02` 共 13 个，已弃用）。

**方法**：Lead 负责解码、切分、总览与交叉核验；6 个 teammate 分别精读片段并产出带 `[seq N]` 的 findings（见附录 B）；Lead 汇总成文，并**独立复核了当前工作区的代码状态与测试结果**（见第 12 节）。

**本文自身的质量控制**：初稿完成后，另派一名 teammate 做了**对抗性核验**（逐条抽查 85 个断言：正确 72 / 错误 10 / 无法证实 3），10 处错误（时间线 2 处、版本归属 2 处、"27/27"分母、发言字符口径、体积归因、片段数、文件名、tests 计数、探针口径）**已全部修正**。核验报告见 `findings\doc-verification.md`。

**本版的额外修订来自用户本人的更正**（共 3 条，均已写入正文）：

1. 初版称会话一 turn 8"用 version 字段证明用户跑旧构建"是一个漂亮的事实锚点——**该判断是错的**，属于长上下文下把过期输入当新鲜证据的误判（详见 §3 turn 8 与 §13.2）。
2. 初版把会话二 turn 1 的根因定位与修复默认归给了 DeepSeek——**实际是 kimi-k3 完成的**（用户失去信任后改用 opencode go 订阅的 kimi-k3）；"幽灵 bug"由此解决（详见 §1.1、§7）。
3. **DPI 档位灯色**与**灯效/亮度即时生效**两项是**后来由 DeepSeek 解决的**（会话二 turn 2），不属 kimi 的工作（详见 §9）。

**必须说明的口径限制**（避免误读）：

1. 轨迹是模型视角的自我记录，不是第三方审计。凡原文未写的内容，findings 中一律标注"记录中未体现"。
2. 分段标注与实际内容存在偏差（切割按字节，任务书按推测的阶段命名）。经 Lead 复核，四段合起来**连续覆盖 turn 1 → turn 11**，无缺口；但任务书里"第 3 段= Web 前端构建"实际覆盖 turn 2 后半 → turn 6 前半，诸如此类。
3. 两个会话的对话是**同一串用户诉求的上下半场**，但**执行者换过人**：会话一末尾（1.2.3）遗留的"一直显示已休眠"，正是会话二的开场问题，而把它解决的是会话二 turn 1 的 kimi-k3，不是 DeepSeek。详见 1.1。

---

## 1. 两个会话的关系

### 1.1 模型归属（关键更正）

这一节是理解全文的前提。**两个会话不是同一个模型的上下半场，而是"换人接力"**：

| 阶段 | 时间 | 执行模型 | 做了什么 | 结果 |
|---|---|---|---|---|
| 会话一 turn 1 | 10:33–11:13 | **DeepSeek** | 侦察、逆向官方驱动、破译私有 HID 协议、组建 5 人团队、建出 Web 驱动 | 成功，且成本极低 |
| 会话一 turn 2–6 | 11:15–13:23 | DeepSeek | 真机反馈驱动的修复（连接竞态、错误分类、单标签锁等） | 部分成功 |
| 会话一 turn 7–10 | 13:34–13:56 | DeepSeek | 反复修"一直显示已休眠"/"啥都未知" | **未解决**，且 turn 8 出现一次误判（见 3 节 turn 8） |
| **会话二 turn 1** | **13:59–14:13** | **Kimi（`kimi-k3`，opencode go）** | **真机复现 → 定位三重根因 → 修 4 个文件 → 验证 57/57 + 真机 8/8** | **成功解决"幽灵 bug"** |
| 会话二 turn 2 | 14:21–14:26 | **DeepSeek** | 用户追加的 3 条需求：删休眠提醒、**DPI 档位灯色**、**灯效/亮度即时应用** | 成功 |
| 会话二 turn 3 | 14:27–14:28 | DeepSeek | 后台服务被取消的收尾 | — |

**用户口述的关键背景**（本文据此更正）：

- 会话一走到 turn 10 时用户**已经失去信任**（原话"你即将失去我的信任"），于是**把"幽灵 bug"交给了 opencode go 订阅的 kimi-k3**；
- **kimi 通过分轮向用户提问拿到了宝贵信息**，并成功解决该 bug（记录中为 2 次结构化 `ask_user_question`：请关闭 Edge 独占设备、请晃动鼠标保持唤醒；用户回忆为三轮问答往来）；
- 相比之下，**DeepSeek 在这一整段修复里几乎不提问**。统计证据：会话一全部 11 个 turn、358 个步骤中，DeepSeek 只调用过 **1 次** `ask_user_question`（`[seq 1826]`，请求用户在浏览器里点一次授权弹窗），而且那次**还超时无人应答**，之后它再没问过，转而用 node-hid 桥接绕过人工授权。会话二 turn 2（DeepSeek）同样 **0 次**提问。用户判断："**可能是长上下文丢失了提问功能**"。
- **成本对比**：kimi 修这一个顽固 bug 花掉了用户 **opencode go 5 小时订阅额度的 58%**；而 DeepSeek 那 40 分钟的完整逆向（会话一 turn 1）只花了 **约 10 元人民币**。

> 这条对照解释了本文后面几处读起来"很聪明"的推理为什么最终没能解决问题：**在信息不足时，DeepSeek 倾向于继续用已有证据自证，而不是向唯一的信息源——用户——提问。**

### 1.2 时间线与交接

```
10:23  会话一启动（模型：DeepSeek / deepseek-flash 全程）：空工作区 D:\<工作区>
       turn 1（186 步 / 至 11:13:43）：侦察 → 逆向官方驱动 → 破译私有 HID 协议
                                     → 组建 5 人团队 → 建 Web 驱动 → 真机验证（v1.0.0）
       turn 2–4：用户真机试用不断报错，逐轮修复（1.0.0 → 1.1.0）
       turn 5–6：连接卡死 / 并发连接竞态修复（仍 1.1.0）
       turn 7–10："啥都未知"/"你到底行不行"/"最后一次机会"/"即将失去信任"
                  → 1.2.0（turn 7）→ 1.2.1（turn 8）→ 1.2.2（turn 9）→ 1.2.3（turn 10），
                   DeepSeek 判定根因为多客户端 HID 争用 —— 但**这个判定不是完整答案**，
                   真正的"冷启动热身 + 假失败 + 错误归因"三层 bug 一直没被找到
       turn 11：后台服务被用户取消（14:17:41），给出重启指引后结束（14:18:01）

13:58  ★ 交接点：用户失去信任，改把"幽灵 bug"交给 kimi-k3（opencode go 订阅）
13:58  会话二启动（turn 1 = Kimi / kimi-k3；turn 2 起 = DeepSeek）
       用户带着 1.2.3 的诊断 JSON 回来——
       "一直在显示已经休眠，即使我打开网页必定使用了鼠标也是如此"
       turn 1（53 步，★Kimi）：**分轮提问用户（关闭 Edge / 晃动鼠标）→ 真机复现 → 定位三重 bug
                                → 修 4 个文件 → 57/57 单测 + 真机 8/8 —— 幽灵 bug 解决**
       turn 2（47 步，DeepSeek）：用户追加 3 条需求（删休眠提醒 / DPI 档位灯色 / 灯效即时应用）
       turn 3（2 步，DeepSeek）：后台服务被取消（14:27:59），说明不需重启（14:28:03）
```

> **时间消歧**：会话二（13:58:29）启动时，会话一的 turn 10 刚结束（13:56:52）、turn 11 尚未触发（14:17:41 才由后台任务取消通知唤起）。因此**两段在时间上部分交叠**（会话二整段 13:58–14:28 与会话一 turn 10 尾–turn 11 重叠），但因果上仍是"会话一的 1.2.3 产物 → 会话二的问题输入"。各 turn 的精确边界：会话一 T1 10:33–11:13、T2 11:15–11:29、T3 11:30–11:33、T4 11:53–12:01、T5 12:26–12:29、T6 13:16–13:23、T7 13:34–13:42、T8 13:43–13:45、T9 13:46–13:49、T10 13:51–13:56、T11 14:17–14:18；会话二 T1 13:59–14:13、T2 14:21–14:26、T3 14:27–14:28。

**一句话因果**：会话一（DeepSeek）造出了驱动，但把"dongle 冷启动热身 + 假失败 + 错误归因"的三层 bug 留在了 1.2.3 里，并在 turn 7–10 反复误判；会话二 turn 1 由 **kimi-k3 用"向用户提问 + 真机复现"**把这个 bug 拆成三条清晰的根因并修掉；会话二 turn 2 再由 DeepSeek 完成 DPI 灯色与灯效即时生效。**三个模型的接力才凑出一个可用的个人 Web 驱动。**

---

# 第一部分 · 会话一：Pulsar 鼠标 Web 驱动开发

## 2. 任务与规模

### 2.1 用户需求（原文，`[11][16]`）

> 我的鼠标已经不受官方支持，我只能自己想办法做一个网页驱动来替代官方配置软件。
> 我现在的鼠标是pulsar xlite v4mini，本机有其驱动软件，我需要你参考 https://github.com/morluto/rea ，制作一个web驱动，能使用该鼠标官方驱动软件的所有功能

隐含约束：Windows 本机、工作目录 `D:\<工作区>`、个人用途、要"官方软件的全部功能"。

### 2.2 规模统计

| 维度 | 数值 |
|---|---|
| turn / 模型步骤 | 11 / 358（T1=186, T2=49, T3=13, T4=20, T5=10, T6=23, T7=21, T8=6, T9=10, T10=17, T11=3） |
| 工具调用 | 347 次，**全部是 `run_code`**（程序化工具调用：模型写 JS 脚本内部再并发调 `pwsh`/`read`/`write`/`web_fetch` 等） |
| 模型 | `deepseek-account/deepseek-flash`，reasoningEffort 由 `high` 提到 `max` `[seq 3][seq 10]` |
| 思考体量 | reasoning 约 **91.7 万**字符；模型对外发言（assistant text）约 **1.7 万**字符；工具结果约 77.3 万字符；另有用户/收件消息文本约 8.7 万字符 |
| 后台任务 | 272 个 job 注册/结算 |
| 上下文压缩 | 20 次 `compaction/prune` |
| 团队 | 5 个 teammate + 共享任务板（task-1..task-5，均 rev3 completed），5 个独立子会话 |
| 版本演进 | 1.0.0 → 1.1.0 → 1.2.0 → 1.2.1 → 1.2.2 → **1.2.3** |

> 注意：会话一**没有**用 read/edit/write 这类直接工具，而是用 `run_code` 把多步操作写成一段脚本并发执行。这解释了它 358 步却能完成天量工作，也解释了两处"read+write 整文件导致 app.js 被截断"的事故（见 2.7）。

---

## 3. 轨迹主线

### 阶段 A · 环境与设备侦察（t1/s1–s4，`[27]`–`[122]`）

**思考**："先做环境侦察再分工"；判断用户要的是"复刻官方驱动全部功能的 Web 驱动"，`rea` 只是逆向方法论/工具链而非鼠标项目 `[41][121]`。

**动作与结果**
- 并发调用 `working_directory`/`list_agents`/`pwsh`：工作区**完全为空** `[52][57]`。
- `web_fetch(github.com)` 被拒：`URL hostname "github.com" resolves to a non-public IP address` `[68]`。但 DNS 返回 `198.18.x.x` 伪造段、`curl.exe` 直连却是 200 —— 判定为 DNS 污染而非断网，此后所有 GitHub 访问改走 `curl`/`git` `[127]`。
- HID 设备枚举出现同一接收器的多个厂商自定义接口（`MI_02` / `MI_03`），FriendlyName 均为"符合 HID 标准的供应商定义设备" `[74][79]`。
- 注册表查到已安装官方配置软件（版本 1.0.5.4），`InstallLocation = <官方驱动安装目录>\` `[110]-[112]`。

### 阶段 B · 官方驱动逆向路线的确立（t1/s9–s21，`[144]`–`[952]`）

这是整个会话**思考密度最高**的一段：连续用"结构证据"否掉多条昂贵路线。

| 步骤 | 思考与判断 | 结果 |
|---|---|---|
| 看安装目录 | 目录里只有 3 个文件（`pulsar_fusion_ver1.0.5.4.exe` 5,888,848 B / `unins000.dat` / `unins000.exe`）→ exe 本身就是完整应用 | `[143]-[145]` |
| 读 PE 头 | `Machine=0x014C`(x86)、PE32、GUI；各节合计约 285 KB 而文件 5.88 MB → **存在巨大 overlay**，判定为打包/自解压启动器，怀疑 Enigma Virtual Box | `[474][547][561]` |
| 运行官方应用 | 抓到 **7-Zip SFX** 自解压到 `%TEMP%\7ZipSfx.000\`，得到真正的主程序（4,173,336 B） | `[583][602][609]` |
| 固化证据 | 整个解包目录复制到 `research\driver-extracted\`（268 文件 / 11.5 MB）；读到 `Config\my_models.csv`，Xlite V4 行给出无线 `3710;5402`、控制接口 `MI03` | `[671][716]-[718]` |
| 查导入表 | 主程序只导入 `mscoree.dll _CorExeMain` → **32 位 .NET 程序集** | `[731][732]` |
| **关键决策** | 既然目标是 .NET，就**放弃动态挂钩**（Frida、Detours DLL 注入、USBPcap/Wireshark、Ghidra/IDA 全部列为备选后否决），改为**直接反编译 C# 拿源码** | `[738]` |
| 装反编译器 | 机器只有 runtime 没有 SDK，无法通过包管理器安装命令行反编译器。绕过：改用开源反编译器 ILSpy 的命令行包，配已有的 .NET runtime 直接运行 | `[799][857][900][907]` |
| 反编译 | 得到完整 C# 工程；HID 依赖为 Costura 内嵌的 **HIDSharp** | `[936]-[952]` |
| 处理混淆 | C# 被控制流混淆（`while(false)` 填充、空 `SpinWait.SpinUntil`）。写 Python 脚本按大括号配对抽方法体、过滤样板行，压缩成 1406 行"协议方法清单"再人工解读 | `[1799][1873]` |

### 阶段 C · 私有 HID 协议破译（t1/s22–s33，`[298]`–`[436]`）

**推理链**：`FWProtocolBase` 是抽象基类 → 只有一个实现 `huAZ2tYGRX2LLl1Kis\QeugLfjKJdKnv2KrF5.cs`（4162 行）`[1018][1040]` → 从代码里读出全部协议常量与布局。

**破译出的协议（公开版只保留结构性描述）**

```
厂商私有的 Feature Report 协议，本质是一个"请求—轮询应答"模型：

- 每条请求写入一个固定长度的 Feature Report，然后轮询读回，直到状态字段表示"成功"；
- 应答里带**命令回显**，用于判断"这条应答是不是我这条命令的" —— 这是后来排查
  "读到上一条命令的旧应答"问题的关键；
- 帧内还有长度字段与校验字段。

> **公开版删减**：具体的帧字节布局、校验算法、命令类/命令字编号、状态码取值、
> 回报率映射表等属于**厂商私有协议细节，本公开版不予披露**。本文只保留
> "如何定位问题"的方法论部分。
```
（`[1114]-[1121][1158][1172][1233][1256]`）

**真机接口判定**：hidapi 枚举确认可用接口是 **MI_03**（厂商自定义 HID 集合，65 字节 Feature Report）；MI_02 虽同为厂商集合，但 Feature Report 仅 2 字节、写入直接失败，不可用 `[1504][1514]`。真机握手时的校验与代码构造完全一致 `[1512]`。

**工具选型**：曾用 `pywinusb`，因 `HIDP_CAPS` 无 `FeatureReportByteLength` 属性而放弃，改用 **hidapi** `[1427][1479]-[1487]`。

### 阶段 D · 组建团队（t1/s34，`[457]`）

**思考**：Lead 判断自己应独占"协议层"这一唯一事实源，把外围工作并行化。

- 建 5 个共享任务（每个都写明**只准写哪些文件 / 不要写哪些文件**）`[439]-[451]`；
- 5 个 teammate 在 **314 ms 内并发 spawn**（`10:42:15.323`–`.637`），均为 `context=fresh`、`delegationDepth=1`。

| 任务 | teammate | 职责 | 写范围（摘要） |
|---|---|---|---|
| task-1 | `proto-analyst` | 从反编译源码提取功能矩阵与 UI 语义 | `docs/features.md`、`docs/ui-spec.md`、`research/notes/**` |
| task-2 | `hid-harness` | 原生 HID 探针 + 真机协议验证 | `tools/pulsar_hid.py`、`probe/**`、`docs/device-report.md` |
| task-3 | `webapp-core` | 应用框架 + WebHID 传输 + Home/Performance | `web/index.html`、`web/js/*`（**除 protocol.js**）、`web/css/app.css` |
| task-4 | `webapp-custom` | Customize / Macros / Lighting | `web/js/pages/{customize,macros,lighting}.js`、`web/js/lib/*`、`web/css/custom.css` |
| task-5 | `platform-dev` | 本地服务 + 持久化 + 启动脚本 + 测试 | `server/**`、`scripts/**`、`tests/**`、`docs/{running,testing}.md` |
| — | `lead`（自己） | **独占** `web/js/protocol.js`、`docs/protocol.md`、README、集成与验收 | — |

### 阶段 E · 协议基础设施 + 真机验证（t1/s47–s91，`[649]`–`[1168]`）

**Lead 亲自交付的"契约"**：`docs/protocol.md`（10.6 K→12.8 K→13.5 K 字符）、`web/js/protocol.js`（707 行）、`docs/keycodes.json`（name_to_hid 118 / hid_usage 108 / modifier 8）、`tools/test_protocol.mjs`。随后群发"协议基础设施已就绪"给 5 个 teammate `[584]-[590]`。

**真机端到端**（用 node-hid 直接跑同一份 `protocol.js`）读出：
- 固件版本、机身 PID（型号自报）、MCU UUID（唯一值已脱敏）
- 电量 83%、**6 个 profile**、回报率 flag=8 → 1000 Hz、LOD 1.0 mm、debounce 3、sleep 30
- DPI 指示灯可读写（关闭/常亮/呼吸 + 速度 + 亮度）；**通用 RGB 灯效与接收器指示灯返回"不支持"**
（`[1578]-[1597][2069]-[2077]`）

**这一步推翻的重要假设**：宏命令"无响应"不是不支持，而是设备用不同的状态码区分了"目标不存在"与"机型不支持"；宏子系统实际可用，只是设备当前 0 条宏 `[1634][1717]`。

**发现的关键坑（后来直接导致会话二的 bug）**：
- 鼠标 **30 秒无操作即休眠**，休眠后无线命令恒返回"设备离线"，且**命令无法唤醒它，只能物理移动鼠标** `[2121]`。
- 官方 App 会**每约 500 ms 轮询一次固件版本**做心跳 `[1444]`；host 轮询**不能**防休眠。
- 响应 `[4]` 之外的字节是**上一条响应的残留脏数据**，解析必须按 `[4]` 截断 `[1640][2077]`。
- 有一条无线状态查询命令**不回显命令字**，Web 端必须对它跳过回显校验 `[2123]`。

据此 Lead 给 `protocol.js` 加了 `STATUS` 常量、`DeviceAsleepError`/`DeviceUnsupportedError`、`keepAlive()`、`waitForAwake()` `[1905][1950]`。

### 阶段 F · 用户反馈驱动的 6 轮迭代（turn 2–11，`[2374]`–`[4532]`）

turn 1 结束时（**11:13:43**）交付了 **v1.0.0** 与一整套文档；此后用户开始真机试用，会话变成"报错 → 定位 → 修复 → 升版本 → 再报错"的长链。**1.1.0 是在 turn 4 引入的** `[3248][3272]`（见上表），1.2.0/1.2.1/1.2.2 分别在 turn 7/8/9。

> **本段全部由 DeepSeek 执行，而且是它最终失败的一段。** 下表的 turn 7–10 逐轮升级版本号，却始终没有找到真正的根因；turn 8 还出现了一次误判（见该行）。用户在 turn 10 之后失去信任，把问题交给了 kimi-k3 —— 真正的根因是 kimi 在会话二 turn 1 找到的。因此**不要把下表的"版本递增"误读为"问题在逐步收敛"**。

| turn | 用户诉求（原文节选） | 模型的思考与判定 | 结果 |
|---|---|---|---|
| 2 | `部分状态读取失败…发送 Feature Report 失败：Failed to write the feature report` `[2374]` | 初判"选错接口 / 写长度不匹配 / 权限误标" | 转入原生 WebHID 复现 |
| 3 | 9 条混合错误：5 条"不支持" + 4 条"HID 通信失败" `[2944][2948]` | **"不支持"其实是旧缓冲区残留**，不是真的不支持；把字段映射回协议命令逐条核对 | 引入 `STALE_RESPONSE`/`NO_RESPONSE`；首次离线即中止整轮、只留 1 条错误 |
| 4 | `你到底行不行？我已经受不了了` + 4 条"设备无响应（可能已休眠）"；随后追问 `确认你逆向代码是不是有问题？你是否申请了浏览器的权限？` `[3100][3219]` | 判定 **HID Feature Report 是"单客户端邮箱"**，多标签页/官方软件会互抢应答；需给出证据链 | 加 **Web Locks 单实例锁**（第二标签页明确提示"设备正被另一个标签页占用"）；升 1.1.0；给出证据：Python 83/83 + 原生 WebHID 全部通过（记录自述 "27/27"） |
| 5 | `一直转圈{请在浏览器弹窗中选择 Pulsar 设备…}，但我已经选择了` `[3376]` | 逐个接口各等 20 s 唤醒 + 文案不更新 + 旧连接未释放 | 重构为 `connectBest()`：只对正确接口唤醒一次；加超时与进度文案；新增 `connect-flow.test.mjs` |
| 6 | `连接依旧卡在正在探测接口，必须刷新页面才连接上，同步也一直转圈` `[3513]` | **找到确定的并发 Bug**：设备选择弹窗授权时浏览器触发 HID `connect` 事件 → 热插拔监听又发起 `autoConnect`，与手动 `requestAndConnect` 同时抢设备；"刷新反而能连上"因此得到解释 `[3521]` | 加 `_serializeConnect` 连接互斥、全 IO 1.5 s 硬超时、同步看门狗 45 s、Esc 取消 |
| 7 | `现在啥都未知了` + 1.1.0 诊断 JSON `[3817]` | 真机复现出**约 6 秒窗口：`status=connected` 但 info 全为 null**，随后才填充 `[3937]` | 加"已连接但无数据"横幅 + info 缺失自动补读；升 1.2.0 |
| 8 | `我觉得你读的不对，我肯定移动了鼠标才进入网页驱动了，你到底行不行` `[4053]` | **❌ 误判（长上下文失效的典型）**：DeepSeek 把"用户贴的 JSON 里 `"version": "1.1.0"`" 当成"用户没刷新、还在跑旧构建"的**事实锚点** `[4108][4163]`。**实际并非如此**——那份诊断是**上一轮的诊断内容被重新提起**（模型自己在 `[1350]` 已注意到它标注的时间是 13:33 的旧抓取），而用户当时用的就是新构建。**用户更正：这正是长上下文下 DeepSeek 的问题——把过期/复现的输入当成新鲜证据。** | 首读失败 800 ms 后自动补读；连接后发现自动休眠 <60 s 时**弹窗建议**改为 900 s（明确否决"未经同意自动改设备设置"）；升 1.2.1 |
| 9 | `你只有最后一次修复机会，现在一直检测移动鼠标是休眠的` + 1.2.1 诊断（info 全 null、**两个 `opened: true` 句柄**）`[4167]` | "**This is the smoking gun**"：`readAllInternal` 从未跑完；`recover()`→`reacquire()` 里的 `navigator.hid.getDevices()` / `candidate.open()` **没有超时** → 命令永久挂起、队列被拖死 `[4173]` | 协议命令 12 s 硬超时 + `resetQueue()`；连接成功后关闭多余句柄；升 1.2.2 |
| 10 | `你即将失去我的信任` + 1.2.2 诊断（fw 读到、PID 失败、仅 1 个句柄）`[4303]` | 排查进程/启动项**未发现 Pulsar 残留** `[4319]`；hidapi 直测发现**所有命令都返回同一条缓存固件应答**、`send_feature_report` 大量返回 -1 `[4343]`；设计"双客户端争用实验"并复现出 `send_ret=65/-1` 交替 `[4387]` | 判定根因为**多客户端 HID 争用 / 接收器卡死**；升 1.2.3，把"写入被拒"明确提示为"可能有其他标签页/程序同时占用" |
| 11 | （无文字）后台服务 job 被用户取消 `[4503]` | "用户主动取消 ⇒ 不擅自重启" | 说明停服无副作用、给出 `scripts\start.bat` 重启方式与 5 步排查顺序 |

**这一段的"思考品质"要分开评价**（本版更正）：

- **真正有价值的推理**：turn 9 从"两个 `opened: true` 句柄"反推出 `recover()→reacquire()` 里 `open()` 没有超时、命令会永久挂起 `[4173]`；turn 10 不做猜测而是**设计双客户端对照实验**复现争用 `[4387]`。这两处是以可观测证据推进的正面案例。
- **反面案例**：turn 8 用"诊断 JSON 里的 `version` 字段"断定用户跑的是旧构建——**结论是错的**。它暴露的是长上下文下的一种典型失效：**把"曾经见过的输入"当成"刚刚发生的事实"**。模型甚至在 `[1350]` 自己识别出"那是 13:33 抓取的"，却仍在 `[1809][1843]` 写下 "strong evidence: they haven't loaded the latest fixes"，并据此回复用户。用户被这句误导，对模型的信任进一步下降。
- **更根本的差异**：信息不足时，DeepSeek 选择"用手头证据继续推理"，而不是"向唯一的信息源提问"。全会话 358 步只有 1 次 `ask_user_question`（`[seq 1826]`，还 120 s 超时无人应答）；而 kimi-k3 在会话二开局就主动问了两次，其中"我都没有打开edge"一句话直接推翻了 DeepSeek 此前最核心的假设。

### 阶段 F 内的两次严重自伤事故（值得单独记）

1. **第一次 app.js 截断**（turn 6 末，`[3703][3717]`）：用 `read` → 字符串替换 → `write` 批量改写大文件，写回后 `SyntaxError: Unexpected end of input`，文件变成 51,276 B、尾部断在 `diagnose` 中间。模型自己总结：**"Lesson: never read+write whole large file; use edit."**
2. **第二次 app.js 截断**（turn 10，`[4409]`）：同类操作再次截断（1155 行）。恢复路径依次评估并否决：用户浏览器（无调试端口）→ Edge HTTP/Code Cache（`candidates: 0`）→ 回收站/VSS/手工重建 250 行（风险高）→ **最终命中 DSH 工作区变更快照** `12c1faf06…`（完整 1.2.2，53,480 字符），恢复出 60,067 B / 1373 行，再用 `edit` 重放两处改动 `[4454][4466][4476][4484]`。

第一次截断的恢复更巧妙：用 **CDP `Debugger.getScriptSource`** 从仍加载着旧脚本的调试浏览器（端口 9503）里取回全文 48,841 字符 `[3739]`。

---

## 4. 团队编排与 5 条子轨迹

5 个 teammate 各是独立会话、单 turn 完成，**没有任何一个被续跑**。它们共同构成了会话一"turn 1 的后半段"。

### 4.1 交付与结果

| teammate | 步数 | 交付物（相对 `D:\<工作区>`） | 自述验证 |
|---|---|---|---|
| proto-analyst | 106 | `docs/features.md`、`docs/ui-spec.md`、`research/notes/analysis/*` | 功能矩阵含协议方法名 + 机型 gating + MainForm 行号 |
| hid-harness | 93 | `tools/pulsar_hid.py`（61 KB）、`probe/probe_all.py`、`probe/verify_spec.py`、`docs/device-report.md`（约 530 行）、`probe/logs/*` | 真机只读 **83/83 PASS**；186 条命令 0 意外失败；**宏写→读回→删除闭环 PASS** |
| webapp-core | 135 | `web/index.html`、`web/css/app.css`、`web/js/{app,hid,state,ui}.js`、`web/js/mock/mock-device.js`、`web/js/pages/{home,performance}.js`、`web/assets/xlitev4/*` | mock E2E 0 console error；5 页全渲染 |
| webapp-custom | 155 | `web/js/pages/{customize,macros,lighting}.js`、`web/js/lib/{keycodes,macro,custom-ui,custom-device}.js`、`web/css/custom.css` | jsdom 交互 11/11；团队 smoke 14/14 |
| platform-dev | 77 | `server/server.js` + `server/lib/*`、`scripts/{start.bat,start.ps1,smoke-test.mjs}`、`tests/*.test.mjs`、`docs/{running,testing}.md`、`package.json` | `npm test` 52/52；`npm run smoke` 14/14 |

> 交付时各 teammate 自述的体积（如 features.md "33 KB"、ui-spec.md "23.6 KB"）小于**今天在工作区实测的体积**（`features.md` 46.5 KB、`ui-spec.md` 32.7 KB、`device-report.md` 40.3 KB、`protocol.md` 18.8 KB）。差额来自交付后 Lead 在 **turn 1 之内**的整合与补充（这四份文件的 mtime 均落在 10:52–11:01，此后未再变动），不是后续 turn 追加的。引用体积时请注明是"交付时自述"还是"当前实测"。

### 4.2 协作机制（5 条轨迹共同反映）

1. **Lead 单点编排 + 任务板 + 写范围隔离**：每个 teammate 的固定动作序列是 `team_task_list → team_task_get → claim(expected_revision) → 干活 → send_message(lead) → complete`。全程**没有出现"直接改别人文件"**。
2. **契约先行，mock 是并行支点**：Lead 的 `protocol.js` 是唯一事实源；`?mock=1` 让 5 条线不必等真机就能并行跑通（UI 冒烟、jsdom、E2E）。
3. **跨范围缺陷走消息上报**，例如：`platform-dev` 发现前端 `PRODUCT_ID_WIRED` 导出缺失 → 上报 Lead；`webapp-custom` 发现 `index.html` 未引入 `custom.css` → 上报 Lead；`webapp-custom` 发现"Feature Report 命令没有全局串行化" → 建议协议层加互斥 → Lead 采纳并在 `protocol.js` 内置 `_enqueue` 全局串行队列。
4. **缺陷闭环 5 轮**：Lead 冒烟发现问题 → 精确报错（文件:行号 + 错误信息）→ 派给 owner → 复测。典型：`ui.js:296` 缺 `+`、`lighting.js:158` 注释缺 `//`、`performance.js` 缺失导致 boot 卡在"正在启动…"。
5. **产物互读交叉验证**：proto-analyst 读 hid-harness 的 `probe_all.json` 把"代码推断"与"真机实测"对齐；webapp-custom 读 proto-analyst 的 features/ui-spec；DPI 灯语义、sleep 单位都经过"文档 → 实现 → CDP 实测"三段验证。

### 4.3 子轨迹给出的最重要纠正

- **`sleepTime` 单位是"秒"不是"分钟"**：Lead 一度按"30 min"传达，proto-analyst 与 hid-harness 双双纠偏（真机值 30 = 30 秒，滑杆 30–900 秒，默认 900 秒 = 15 min）。Lead 随后修正 `protocol.js` 与 `docs/protocol.md`（`setSleepTime(profileId, seconds)`）。
- **`keepAlive` 不能防休眠**：Lead 原以为 500 ms 轮询可"保持唤醒"，hid-harness 实测静默 30–45 s 即休眠（轮询只把窗口延长到 116 s）。结论：keepAlive 只能刷新状态，UI 必须做"休眠提示 + 用户唤醒 + 自动重试"。
- **任务书与实际交付存在范围差**：task-4 要求完整 RGB 灯效页，但官方软件 **根本没有 Lighting 页**（`models.xlsx` 的 `EffectListID` 全为空），本机灯光命令返回"不支持"。最终决策：本机"灯光"按 **DPI 指示灯** 实现，RGB 只保留被 gating 的演示入口。

---

## 5. 会话一的最终结果

### 5.1 交付物（已核实的当前工作区状态）

```
D:\<工作区>\
├─ web\            前端（原生 ES modules，无打包器、无 CDN）
│   ├─ index.html · css\{app,custom}.css
│   ├─ js\{app,hid,protocol,state,ui}.js      protocol.js 34.9 KB
│   ├─ js\pages\{home,performance,customize,macros,lighting}.js
│   ├─ js\lib\{keycodes,macro,custom-ui,custom-device}.js
│   ├─ js\mock\mock-device.js                  ?mock=1 演示/测试
│   └─ assets\xlitev4\*                        官方配置图 + favicon
├─ server\         零依赖本地静态服务 + 配置持久化/备份 + /api/usb-info
├─ scripts\        start.bat / start.ps1 / smoke-test.mjs
├─ tests\          6 个 *.test.mjs（protocol/server/mime/profile-store/launchers/connect-flow）+ fixtures/ + helpers/
├─ docs\           protocol.md · features.md · ui-spec.md · device-report.md
│                  running.md · testing.md · verification-report.md · keycodes.json
│                  deploy-azure.md · screenshots\ · verification\
├─ research\       官方软件逆向证据（反编译 C#、解包配置），只读归档
├─ tools\          pulsar_hid.py（61 KB）等协议探针 + nodehid\ 桥接
├─ probe\          真机抓包日志与验证证据
├─ rea\            morluto/rea 参考克隆（只读）
└─ README.md · package.json · .gitignore
```

### 5.2 验证数字（会话记录中的口径）

| 项目 | 结果 |
|---|---|
| Python hidapi 真机只读回归 | **83/83 PASS**（`probe/logs/verify_spec.json`）；另有 186 条命令全量只读探针：**157 条成功、24 条"请求非法/目标不存在"、5 条"机型不支持"，0 条意外失败** |
| 真机全量读写（node-hid 桥接 + 页面内 WebHID 兼容层） | 读出 FW/PID/电量/回报率/DPI/去抖/LOD/休眠/灯光/按键/颜色；写入并读回全部项；**宏创建→读回字节一致→删除**闭环 PASS |
| 原生 Edge WebHID 真机验证 | **全部通过，跑了两遍**：记录原话自述为 "27/27"（读取 13 + 写入回读 11 + 开关 3 + 宏 1 项），证据文件 `docs/verification/native-webhid-suite.json` 实列 **28 项 `ok=true`** |
| Node 单测 | `npm test` 由 **52/52 → 54/54**，本段内 9 次全绿（会话二 turn 1 由 kimi 再加 3 条回归用例，带到 **57/57**，见第 12 节） |
| 浏览器冒烟 | `npm run smoke` 每次 `PASS ✅（14/14）`，附 7 张截图 |
| 关键机制发现 | 接收器暴露 **3 个同名的 HID 条目，只有其中一个可写**；另一个条目写入时报 `NotAllowedError: Failed to write the feature report.` |

### 5.3 遗留问题（会话结束时仍未闭环）

1. **用户侧从未确认成功**：最后一条用户消息是 turn 10 的"你即将失去我的信任"；1.2.2/1.2.3 的改动**没有任何成功回执**。
2. **根因未最终区分**：turn 10 的结论是"另一个客户端持续占用/轮询同一 HID 接口"或"接收器/USB 状态卡死"，用户未执行"完全退出 Edge + 拔插接收器"后的复测。
3. **写路径的真机验证不完整**：真机写入验证是通过 node-hid 桥接完成的；原生 WebHID 下的写入只做过"写同值读回"。
4. **首授权仍需人工**：Edge 策略注册表写入被拒（Medium 完整性级别）、CDP `Browser.grantPermissions(["hid"])` 报 `Unknown permission type: hid` —— 无头自动化授权被证伪。
5. **范围外/未实现**：固件升级 DFU、RF 配对写入、8K dongle、有线模式；浏览器无法修改 Windows 鼠标属性（页面已注明）。
6. **工程卫生**：仓库**没有 git**（`git log` 无输出），改动无版本记录可回溯；两次 app.js 被自己截断。

---

# 第二部分 · 会话二：修复鼠标Web驱动休眠误报问题

## 6. 输入：用户问题与诊断数据

**用户原文**（`[seq 9][16]`）：

> 我的鼠标已经不受官方支持，我只能自己想办法做一个网页驱动来替代官方配置软件。
> 已经让其他ai逆向并构建了web驱动，但目前存在问题，**一直在显示已经休眠，即使我打开网页必定使用了鼠标也是如此**，请查看代码，修复bug

**随附诊断 JSON 的关键字段**（version `1.2.3`，`http://127.0.0.1:8787/`，Edge 155）：

| 字段 | 值 | 含义 |
|---|---|---|
| `connected` | `true` | 连接是成功的 |
| `wakeNeeded` | **`true`** | 页面认为设备在休眠 |
| `readErrors` | 只有一条：`devicePID: "设备无响应（可能已休眠）：请移动或点击鼠标唤醒，然后点"刷新""` | **只有 PID 读失败** |
| `info.firmwareVersion` | 有值 | 固件读到了 |
| `info.dongleVersion` | 有值（由本机服务补充） | 接收器版本读到了 |
| `info.*` 其余 | 全为 `null`（PID/UUID/profile/battery/polling/DPI/settings） | 因为首项失败后**整批被中断** |
| `devices` | 3 条，其中**两条 `opened: true`** | 后来成为重点怀疑对象 |

这就是"误报休眠"的现场：**设备明明在工作、固件和接收器版本都能读到，却因为一条 `GetDevicePID` 失败被判定为休眠，进而整批读取中止、顶部一直挂着休眠横幅。**

---

## 7. 根因定位的推理链（会话二 turn 1 · **由 kimi-k3 完成**）

> **归属声明（本版更正）**：本节描述的全部推理、提问、复现与修复，**作者是 kimi-k3（opencode go），不是 DeepSeek**。会话二 turn 1（53 步，13:59–14:13）全程由 kimi-k3 执行；用户在 turn 2 开始前才把模型切回 DeepSeek（记录见 `[seq 379]`）。

模型没有直接改代码，而是走了"先排除、再复现、最后归因"的完整链条。

### 7.0 第一步不是读代码，而是**向用户提问**

这是 kimi 与 DeepSeek 最大的行为差异。kimi 在开局不久（`[seq 151]`）就向用户发起结构化提问，而且是有明确目的、给了选项的"事务性提问"：

1. **"我需要独占访问鼠标接收器来做一次干净的真机抓包。请关闭 Edge 中打开的驱动页面（127.0.0.1:8787）后告诉我。"** —— 目的是消除并发干扰。
   - 用户回答（自定义文本）：**"我都没有打开edge"** `[seq 155]`。
   - 这一句**直接推翻了"浏览器并发串扰"这个核心假设**，把矛头转向设备侧行为 + 代码本身，而不是用户环境。
2. **"请现在晃动/点击一下鼠标，让它保持唤醒状态，然后告诉我"** `[seq 156]` —— 用户回答"已晃动鼠标，正在使用中"。
   - 这一步换来了整个诊断最关键的对照数据：冷态下前几条命令失败、后几条 ~16 ms 成功 → **"dongle 热身期"假设成立**。

> 用户回忆这段为"三轮问答往来"；记录中为 **2 次** `ask_user_question` 调用（另加若干次在回复正文里的追问）。无论计几次，结论一致：**这个 bug 的破解靠的是"向唯一的信息源要信息"，而这一点恰恰是 DeepSeek 在长上下文里丢掉的能力。**

### 7.1 先排除"协议本身错了"

- 读 `docs/device-report.md` 的真机抓包：读「机身 PID」这条命令的请求与应答**逐字段对齐 —— 状态为成功、命令回显匹配、校验正确**。**帧、偏移、校验和、回显索引全部正确** `[seq 60]`。
- 读 `probe/logs/cli-pid.json` / `probe_all.json`：真机 6 ms 返回、状态成功、回显匹配 `[seq 70][seq 77]`。
- 同一 transport 下「读固件」这条命令稳定成功 → **WebHID 的 `reportId=0` / 传输层也没问题** `[seq 88]`。

### 7.2 读应用流程，发现"死循环结构"

- `readAllInternal` 的 `safe()`：任何 `isOfflineError` 都置 `asleep=true` 并 `handleAsleep` → 设 `wakeNeeded`；而当时的 `isOfflineError` **同时包含** `DEVICE_ASLEEP`、`HID_IO`、`STALE_RESPONSE`、`NO_RESPONSE`。
- 唤醒探测每 3 s 调 `getFirmwareVersion()`，**固件是 dongle 本地缓存命令、几乎总能成功** → `clearWake` → `refreshAll` → 又卡在 `GetDevicePID` → 再次 `handleAsleep`。
- 模型在思考里直接写下：**"Infinite loop! The banner would keep appearing"** `[seq 70][seq 77]`。

### 7.3 真机复现（关键转折）

**决策**：不劫持用户浏览器（否决"杀 msedge"），而是写 `tools\nodehid\repro-pid.mjs`，用 node-hid 模拟同一 transport、**复用同一份 `web/js/protocol.js`**，重放 `readAllInternal` 的 8 条读命令。

第一次运行（冷态）即复现 `[seq 143]`：

| 命令 | 结果 |
|---|---|
| `getFirmwareVersion` | OK 33 ms |
| `getDevicePID` | **FAIL `DeviceIOError/HID_IO`，1 ms**，消息 `HidD_SetFeature: (0x0000001F)`（Windows `ERROR_GEN_FAILURE`） |
| `getDeviceMCUUUID` | FAIL `STALE_RESPONSE`（读到的是**别的命令的旧响应**） |
| `getBattery` | FAIL `STALE_RESPONSE` |

**用户否定了一个假设**：模型怀疑是浏览器并发串扰，于是问用户关掉 8787 页面；用户回答 **"我都没有打开edge"** `[seq 150]` → "两个 open 句柄造成串包"的假设被推翻 —— **没有浏览器参与时设备本身就会失败** `[seq 155]`。

让用户晃动鼠标后重跑 `[seq 167]`：前几条仍失败，但**紧接着的 `getProfileID`/`getProfileIDList`/`getPollingInfo`/`getDPIStageInfo` 全部 ~16 ms 成功** —— 这就是"热身期"的直接证据。

**关键发现**：有一条命令的 `SetFeature` 在 OS 层报 `0x1F` "失败"，**但设备实际收到并执行了**，其响应后来出现在缓冲区 —— 即 **`HidD_SetFeature` 会谎报失败** `[seq 150][seq 155]`。

### 7.4 与"一直能用"的 Python 探针对比，定位 JS 侧三处过严

`tools/pulsar_hid.py` 的 `transact` 参数：`stale_echo_limit=25`、poll 10 ms、单次 timeout 700 ms、写入失败重连并最多重试 3 次。与 JS 版逐条对比 `[seq 177][seq 184]`：

| # | JS 侧问题 | Python 侧 |
|---|---|---|
| 1 | 回显不匹配计数 `> 6` 就抛 `STALE_RESPONSE`；pollMs=15 → **约 90–126 ms 就放弃** | 容忍 **25** 次（≈375–500 ms），正好覆盖真机热身期 |
| 2 | 写入失败**一次就 `recover()` 重开句柄**，新句柄又是冷态 → 恶性循环 | 写入失败重连 + 最多重试 3 次 |
| 3 | 旧代码**在回显校验之前**就判"设备离线" → 缓冲区里残留的一条旧"离线"应答也被当成真休眠 | 先校验回显，不匹配只算 stale、继续轮询 |

### 7.5 应用层错误归因

- `app.js` 的 `isOfflineError` 把 `STALE_RESPONSE`/`NO_RESPONSE` 与真休眠**一律**当作离线 → 设 `wakeNeeded` + 中断整批。
- `web/js/lib/custom-ui.js` 的 `isAsleepError` 用**消息文本正则** `/休眠|睡眠|唤醒|离线|asleep|offline/i` 兜底；而 `STALE_RESPONSE` 的文案里恰好含"休眠" → customize/macros 页也跟着误显示休眠横幅。

### 7.6 最终根因（kimi 自己的表述，`[seq 367]`）

> 「一直显示已休眠」= **三重 bug 叠加出的死循环**：
> 1. dongle 冷启动热身期（实测约 **300–500 ms**）旧响应（状态字段停在"成功"这个终止态）滞留缓冲区，而 `protocol.js` 回显不匹配容忍度只有 6 次（≈100 ms）就抛 `STALE_RESPONSE`（Python 探针容忍 25 次）；
> 2. `HidD_SetFeature` 谎报 `ERROR_GEN_FAILURE 0x1F`（帧其实已送达），旧代码把写入失败当致命错误并 `recover()` 重开句柄（新句柄又冷态），恶性循环；
> 3. `app.js` 把 `STALE_RESPONSE`/`NO_RESPONSE` 一律当"设备已休眠" → 设 `wakeNeeded`、中断整批读取 → 唤醒探针读固件（dongle 缓存必成功）→ 清除唤醒 → 刷新又在 `GetDevicePID` 失败 → 再误判……**无限循环**。

### 7.7 被验证 / 被推翻的假设

| 假设 | 结论 | 证据 |
|---|---|---|
| `GetDevicePID` 命令/协议偏移/校验和/回显索引错 | **推翻**（真机抓包完全一致） | `[60][70][77]` |
| WebHID `reportId=0`/传输层特例 | **推翻**（同一 transport 下"读固件"稳定成功） | `[88]` |
| 浏览器/其他程序并发占用导致串包 | **推翻**（用户没开 Edge；无浏览器时 node 复现仍失败） | `[150][155]` |
| 鼠标真的休眠导致 PID 读失败 | **推翻**（真休眠时 PID/UUID/无线状态仍由接收器缓存应答；实际失败形态是 STALE/HID_IO） | `[150][167]` |
| 缓冲区旧响应滞留 + 写入假失败 + JS 容错过紧 + 应用层错误归因 | **验证成立** | `[143]–[177][184][367]` |
| 修复后浏览器端"永久失败"的精确微观机制 | **未完全证实**（模型自述"不如先修掉这三个明确 bug 重要"） | `[184]` |

---

## 8. 修改清单与验证（**kimi-k3 的修复**）

### 8.1 修改清单（精确路径 + 针对的根因）

| 文件 | 改动 | 针对根因 |
|---|---|---|
| `web\js\protocol.js` | 新增 `staleLimit`（默认 **25**）、`writeRetries`（默认 2，含首次共 3 次尝试）、`writeRetryDelayMs`（80 ms） | ①③ |
| 同上 | `_execCommand` 写入失败改为**先在同一 attempt 内就地重试**，全部失败后才 `recover()` + 重写一次，最后才抛 `DeviceIOError` | ② |
| 同上 | 轮询/终止态处理**改为先校验命令回显**，回显不匹配只累加 `mismatches`、超过 `staleLimit` 才抛 `STALE_RESPONSE`；回显匹配后才依次处理 `5→offline / 1→返回 / 3→InvalidArgumentError / 4→重试一次或 Unsupported` | ①③ |
| 同上 | `STALE_RESPONSE` 文案**去掉误导性的"已休眠"字样** | ③ |
| `web\js\hid.js` | `_commit()` 提交连接后放宽默认重试：`defaultRetries = max(defaultRetries, 3)`（探测期为了快速失败用的是 1） | ①② |
| `web\js\app.js` | `isOfflineError` **删掉** `STALE_RESPONSE`/`NO_RESPONSE`，只保留真休眠与 `HID_IO`/`DeviceIOError` | ③ |
| 同上 | `readAllInternal` 的 `safe()` 分三支：真休眠→仅该项中断 + 横幅；`HID_IO`→占用文案；**其它错误→记录该项后继续读剩余项，不再中断整批** | ③ |
| `web\js\lib\custom-ui.js` | `isAsleepError` 改为**有协议错误码时以码为准，无码才用消息文本兜底** | ③ |
| `tests\protocol.test.mjs` | 新增真机故障模式回归测试（旧响应容忍 / 写入假失败重试 / 残留"离线"应答不误判） | ①②③ |
| `tools\nodehid\repro-pid.mjs`、`tools\cdp-lighting-check.mjs`、`tools\cdp-stage-color-check.mjs` | 新增复现/验证脚本（非产品代码） | — |

### 8.2 验证结果

| 项目 | 修复前 | 修复后 |
|---|---|---|
| `npm test` | 54 pass | **57 pass / 0 fail**（新增 3 个回归用例） |
| 真机 `node tools\nodehid\repro-pid.mjs`（冷态） | **4/8 失败**（机身 PID 报 `SetFeature 0x1F`、UUID/电量报 STALE） | **8/8 成功，0 重试 0 错误**：固件 21 ms、机身 PID 31 ms、UUID 31 ms、电量 62 ms(100%)、当前配置=1、配置列表=[1..6]、回报率 1000 Hz、DPI 档位 16–32 ms |
| 复测条件 | — | 两次复测都在设备**空闲数分钟（冷态）**后执行，正是最能触发原 bug 的时机 `[seq 254][seq 338]` |

---

## 9. turn 2：用户追加的 3 条需求（**由 DeepSeek 完成**）

> **归属声明（本版更正）**：本节全部内容由 **DeepSeek（`deepseek-flash`）**完成。模型切换发生在 **turn 1 结束、turn 2 开始之前**（用户在 `[seq 379]` 处切回 DeepSeek）——即"幽灵 bug"由 kimi-k3 修（第 7–8 节），随后的 **DPI 档位灯色**与**灯效/亮度即时生效**由 DeepSeek 做（本节）。

**用户原文**（`[seq 372][377]`）：

> 1.删除初次连接的连接休眠时长提醒
> 2.灯光部分：DPI 档位颜色没有正确反应dpi挡位灯光颜色的设置
> 3.灯光部分：灯效、亮度设置不应该点击应用才应用，应该一调整就应用

（记录 `[seq 379]`：`[model changed: assistant turns above this point were generated by opencode-go/kimi-k3; the session continues with deepseek-account/deepseek-flash]`。）

### 需求 1 · 删除初次连接的休眠时长提醒

- 背景：turn 8 时加的 `maybeOfferSleepFix()`（连接后若 `sleepSeconds < 60` 就弹窗建议改 900 秒）。
- 处理：**删除整个函数 + 删除调用点**；grep `maybeOfferSleepFix|sleepFixOffered|建议延长自动休眠` 0 命中。

### 需求 2 · DPI 档位颜色没有正确反映设置

这一条展示了很漂亮的**假设排除法**。模型先列并逐一排除了：协议偏移错（真机抓包 `resp[9..11]=R,G,B` 正确）、写路径 stage 号错、品牌调色板兜底、性能页与灯光页双数据源不一致、只显示活动档位颜色……

然后用 CDP 在**浏览器里实测复现** `[seq 482]`：4 个档位的**十六进制文本是对的**（`#0a0adc / #0000dc / #0ac80a / #c8c800`），但 4 个颜色圆点**全部渲染成同一个回退色** `rgb(10,10,220)`。

**真根因**（`[seq 490][708]`）：`custom-ui.js` 的 `applyAttrs` 用 `Object.assign(el.style, {...})` 设置 `--stage-color` —— **CSS 自定义属性必须用 `style.setProperty()`**，`Object.assign` 对 `--*` 属性静默无效。

修复后实测：圆点颜色与 hex 一一对应；改档位 1 为红色 → 圆点 `rgb(255,0,0)`、设备读回 `{stage:1, r:255, g:0, b:0}`。

### 需求 3 · 灯效/亮度应"一调整就应用"

- 移除"应用灯效与亮度"按钮，改为**灯效切换 / 速度 / 亮度滑块一律自动写入**：`scheduleDpiApply()` / `flushDpiApply()`（**400 ms 防抖**，`_dpiWriting`/`_dpiPending` 串行化）+ 状态行"正在写入设备… / 已写入设备（呼吸 · 亮度 50%）/ 写入失败：…"。
- RGB 区（本机 `hasRgbLighting:false`，属 gated）一并改为自动写入并移除"应用灯效"按钮。
- **修掉一个自引入 bug**：`onChange` 先乐观更新 `this.dpiLed.effect`，导致 flush 判断"无变化"而**没真正写设备**（设备端 effect 仍是 1）；改为"只在写入成功后更新内存值" `[seq 628][seq 634]`。
- 核查结论：性能页的灯效/亮度本来就是 `onChange` 即写，**无需改动** `[seq 692]`。

**验证**（CDP + mock，读设备状态）：点击"呼吸" → `DEVICE-STATE: {"effect":2,...}`；亮度拖到 50% → `brightness:128`；状态行同步显示"已写入设备（呼吸 · 亮度 50%）" `[seq 648]`。

### turn 3

后台 `node server/server.js` job 被用户取消 `[seq 713][717]`。模型判断输出里只有浏览器刷新/关页产生的 `clientError: read ECONNRESET`，**不是代码问题**，用户主动取消故不擅自重启，只说明可用 `start.bat` / `npm start` 拉起 `[seq 725]`。

---

## 10. 会话二的遗留问题

1. **浏览器端"永久失败"的精确微观机制未完全证实**：kimi 推测是"每次写入失败 → `recover()` 新冷句柄 → 再失败"的循环，并自述"不如先修掉三个明确 bug 重要"。
2. **"两个 `opened:true` 句柄"未直接修**：只通过减少 `recover()` 调用间接降低概率，未在修复后重新导出诊断 JSON 确认。
3. **修复后未在用户真实浏览器会话中做端到端复测**：真机复测走的是 node-hid 脚本（读命令 8/8）。
4. **写命令（Set 类）未在真机验证**：TURN 1 只跑了读命令；TURN 2 的自动写入验证用的是 mock 设备。
5. **用户未在记录中确认最终效果**：只提示"刷新页面即可生效"。
6. **仓库无 git**，改动无版本记录可回溯。
7. 临时验证脚本（`repro-pid.mjs`、`cdp-lighting-check.mjs`、`cdp-stage-color-check.mjs`）有意保留在工作区。

---

# 第三部分 · 结果与现状

## 11. 两个会话的因果闭环

```
会话一【DeepSeek】turn 1 ── 逆向出协议，正确识别出"休眠表现为设备离线、且命令无法唤醒"
      │
      ├─ turn 2–6  用户真机报错 → 把"写入失败"归因为状态5/权限、
      │            又误把 STALE/NO_RESPONSE 归入"离线"，引入 isOfflineError 过宽
      │
      ├─ turn 7–10 版本走到 1.2.3；根因判定为"多客户端 HID 争用"，
      │            但真正的"冷启动热身 + 假失败 + 错误归因"三层 bug 未被识别
      │            turn 8 还误判为"用户没刷新、在跑旧构建"（错）
      │
      └─ turn 10 结束（13:56:52）用户："你即将失去我的信任"  ← ★ 信任断裂
      │
      ▼  用户换人：把"幽灵 bug"交给 kimi-k3（opencode go 订阅）
会话二【Kimi】turn 1
      │
      ├─ 先向用户提问（关 Edge？晃鼠标？）→ "我都没有打开edge" 推翻旧假设 ★
      ├─ 用真机抓包排除"协议错"（这是会话一留下的正确资产）
      ├─ 再用 node-hid 复现（不信推测，要可观测证据）
      ├─ 用"Python 探针一直能用"做对照，量化出 JS 侧三处过严
      └─ 修 4 个文件 + 3 个回归用例 → 57/57 单测、真机 8/8   ← 幽灵 bug 解决
      │
      ▼  同一会话切回 DeepSeek
会话二【DeepSeek】turn 2 ── DPI 档位灯色（CSS 自定义属性）+ 灯效/亮度即时生效
```

> 这张图表达的是**因果依赖**，不是时间先后：会话二的起点（13:58:29）其实早于会话一的终点（turn 11，14:17:41），两段在时间上重叠——用户是在会话一还在处理 turn 10 反馈时，另开一个会话来报同一个"休眠误报"（时间线见第 1 节）。

**一个值得记下的判断**：会话一里"多客户端争用"的结论并非完全错误（turn 10 的双客户端实验确实复现了 `send_ret=-1` 交替），但它**被当成了唯一根因**，于是修法偏向"提示用户退出 Edge / 拔插接收器"，把责任推给了用户环境。kimi 的第二次提问拿到的"我都没有打开edge"直接否证了这条主线：**在没有浏览器参与的情况下设备本身也会失败**，真正可修的是客户端代码的容错与错误归因。这是"过早收敛到一个可解释的根因"的典型代价——**而破解它的唯一办法，是回头问用户。**

## 12. Lead 独立核验：当前工作区的真实状态

以下为本文写作时在 `D:\<工作区>` 上**实际执行**的核验（不是引用记录）：

| 核验项 | 结果 |
|---|---|
| `npm test` | **`tests 57 / pass 57 / fail 0`**（duration 6288.9 ms）——与会话二的自述一致 |
| `web\js\app.js` 版本号 | `version: '1.2.3'`（2 处） |
| `app.js` `isOfflineError` | 只判 `isAsleepError(err)` 与 `err.code === 'HID_IO' \|\| err.name === 'DeviceIOError'` —— 已不含 `STALE_RESPONSE`/`NO_RESPONSE` ✅ |
| `lib\custom-ui.js` `isAsleepError` | 先看 `DeviceAsleepError`/`DEVICE_ASLEEP`/`details.status === 5`；注释明确写"带明确协议错误码的（STALE_RESPONSE/NO_RESPONSE/HID_IO 等）不是休眠"；**有 code 直接 return false**，只有无码才用消息正则兜底 ✅ |
| `maybeOfferSleepFix` 残留 | grep `maybeOfferSleepFix\|sleepFixOffered\|建议延长自动休眠` → **0 命中**（需求 1 已落实）✅ |
| `protocol.js` `staleLimit` | 构造函数默认 **25** ✅ |
| 工程规模（排除 `node_modules`/`tests\.tmp`/`.git`） | **23,268 文件 / 5,168.7 MB**，其中 `probe\` 一个目录就占 **18,440 文件 / 5,094.8 MB**（真机验证期留下的 Edge 调试 profile），`rea\` 52.9 MB、`research\` 16.4 MB；真正的产品代码 `web\` 仅 36 文件 / 0.8 MB、`server\` 7 文件 / 0.1 MB、`tests\` 8 文件 |

> 结论：会话二 turn 1（kimi-k3）声称的三项根因修复**在代码里都能直接看到**，测试数字也完全对得上。这既验证了 kimi 的修复确实落地，也说明其自述是可靠的。

## 13. 复盘：为什么同一个"幽灵 bug"，DeepSeek 修不掉而 kimi-k3 修好了

这一节是本版新增的核心分析。

### 13.1 三个可量化的行为差异

| 维度 | DeepSeek（会话一 turn 7–10，4 个 turn / 71 步） | Kimi（会话二 turn 1，53 步） |
|---|---|---|
| **向用户提问** | 全会话 358 步仅 **1 次** `ask_user_question`（`[seq 1826]`，请用户点浏览器授权弹窗，**120 s 超时**），此后不再提问 | 开局即 **2 次**结构化提问（`[seq 151]` 关 Edge、`[seq 156]` 晃鼠标），每次都换回决定性信息 |
| **对关键否证的处置** | 用户说"我肯定移动了鼠标才进入网页"（`[4053]`）后，仍坚持"你在跑旧构建" | 用户说"我都没有打开edge"（`[155]`）后，**立刻推翻自己的核心假设**并转向设备侧 |
| **失败后的策略** | 继续在同一假设上叠补丁（超时、看门狗、互斥锁、单标签锁…），版本号从 1.2.0 升到 1.2.3 | 换验证手段（node-hid 复用同一份 `protocol.js` 真机复现），用可观测数据反推 |
| **结果** | 未解决，用户信任破裂 | 解决：57/57 单测、真机 8/8 |

### 13.2 为什么"提问"在这里是决定性的

这个 bug 的根因位于**用户环境与硬件行为**，而不在代码里：

- 代码正确（协议、校验和、回显全部对得上）→ 静态读代码**永远推不出来**；
- `HidD_SetFeature` 的**假失败**只在 dongle **冷启动热身期**出现 → 不控制"设备刚醒 / 空闲后"这个状态就复现不了；
- "浏览器里失败、Python 探针成功"的差异**只能由用户告知**（他有没有开浏览器、鼠标当时是不是刚动过）。

kimi 的两问恰好各拿到半个答案：**"没开 Edge"排除了干扰源，"刚晃过鼠标"给出了冷/热对照**。没有这两条信息，再多的推理也只是在猜。

### 13.3 代价对比（用户口述）

| | DeepSeek | Kimi（`kimi-k3`，opencode go） |
|---|---|---|
| 本次工作 | 会话一 40 分钟**完成整个官方驱动逆向 + 协议破译 + Web 驱动初版** | 修 1 个顽固 bug（会话二 turn 1，约 14 分钟） |
| 成本 | **约 10 元人民币** | **5 小时订阅额度的 58%** |

> 这组数字很值得注意：DeepSeek 的**单位成本效率极高**（天量逆向只花 10 元），但在需要"开口问一句"的场合反而卡死；kimi 单价高、额度消耗大，却靠两三次提问直接命中根因。**问题不在于"谁更聪明"，而在于长上下文里的行为退化——把提问能力丢掉了。**
>
> 用户的观察原话是："ds 在修复时，不太喜欢提问，可能是长上下文丢失了提问功能。"本文的记录与这个判断一致：DeepSeek 唯一的那次提问发生在 turn 1 的中后段（第 140 步 / 共 186 步，`[seq 1826]`），此后 7 个 turn 里 `ask_user_question` 再没出现过一次。

### 13.4 可迁移的经验

1. **修复阶段的提问应当是强制的**，不能只依赖模型自发：当同一现象连续两轮未闭环时，应主动向用户索要"环境状态"（有没有别的客户端、设备当时冷还是热、是否刷新过）。
2. **不要把"曾经见过的输入"当作"当前事实"**：turn 8 的误判源自把上一轮遗留的诊断快照当作新证据。带时间戳的输入必须按时间戳判定新鲜度（模型自己都在 `[1350]` 注意到了 13:33，却没据此否决结论）。
3. **成本与能力要分开算**：低成本模型适合大吞吐的确定性工作（逆向、抽取、批量改造）；高单价模型适合"信息缺失 + 需要交互"的排障。混合调度比单一模型更划算。

## 14. 附：同工作区的相邻会话（供理解上下文，不在本文分析范围）

`<工作区槽位>` 下还有第三个有标题的会话 **`清理 Azure 机器并部署 Web 驱动`**（`SESSION-C`，起始 2026-10-18 12:34:41），它把驱动部署到了 Azure VM `https://<Azure-VM>/`，其产物 `docs\deploy-azure.md`、`docs\screenshots\azure-remote-deploy.png` 也在工作区里。另有 3 个空会话目录。用户本次未要求分析这些，故仅记录存在。

---

## 附录 A · 数据来源路径

**原始会话（zstd 压缩）**
```
C:\Users\<用户名>\.dsh\sessions\<工作区槽位>\
  SESSION-A\session.v4.jsonl.zstd   (3.99 MB)  ← 会话一（Lead）
  SESSION-B\session.v4.jsonl.zstd   (0.76 MB)  ← 会话二
  SUB-1\session.v4.jsonl.zstd           ← proto-analyst
  SUB-2\session.v4.jsonl.zstd           ← hid-harness
  SUB-3\session.v4.jsonl.zstd           ← webapp-core
  SUB-4\session.v4.jsonl.zstd           ← webapp-custom
  SUB-5\session.v4.jsonl.zstd           ← platform-dev
```

**本次分析产生的中间产物**（本地工作区，**未随本文发布**）
```
dev-title.jsonl / sleep-title.jsonl      解码后的原始 JSONL（会话原文，含未脱敏信息 → 不发布）
dev-c01..c08.md / sleep-c01..c02.md      压缩轨迹片段（保留完整 thinking，裁剪工具载荷）共 10 个文件
dev-outline.md / sleep-outline.md        逐步大纲（思考摘要 + 工具调用 + 团队事件）
dev-index.json / sleep-index.json        结构化事件索引
sub\*-outline.md / -index.json / .jsonl  5 个 teammate 子会话
findings\*.md                            7 份带 [seq N] 证据的分析报告（见附录 B）
coverage.txt / timeline-check.txt        分段覆盖范围与 turn 边界时间核验
extract.py / index.py / overview.py / chunk.py / compact.py / subsessions.py / coverage.py / timeline_check.py / redact.py / publish_safe.py  工具脚本
```

> 上述中间产物、findings 与原始会话记录**均未包含在本仓库中**。附录 B 仅说明本文结论的来源结构，便于理解证据链。

> ⚠️ **脱敏范围提醒**：本文（脱敏修订版）已清理可识别信息，但**上述中间产物与 findings 原文件仍是未脱敏的**——`dev-title.jsonl` / `sleep-title.jsonl` 是会话原文，其中包含会话 ID、用户名、机器路径、公网 IP、设备 UUID 等。如需对外分享，请只分享本文，或先对这些文件做同等脱敏。

## 附录 B · findings 文件（每条的结论都可在此回溯 `[seq N]`）

| 文件 | 覆盖 | 大小 |
|---|---|---|
| `findings\dev-recon.md` | 会话一 t1/s1–s91（侦察 + 逆向路线） | 38.7 KB |
| `findings\dev-proto.md` | 会话一 t1/s92–s186、t2/s1–s5（协议修正、文档、真机验证、团队协作） | 45.7 KB |
| `findings\dev-web.md` | 会话一 t2/s6–t6/s15（原生 WebHID、错误分类、单标签锁、并发连接） | 42.8 KB |
| `findings\dev-verify.md` | 会话一 t6/s16–t11/s3（逐 turn、两次 app.js 截断与恢复、1.2.x 演进） | 37.4 KB |
| `findings\sleep-fix.md` | 会话二 turn 1（kimi-k3 的根因链、修改清单、验证）+ turn 2 三需求 | 40.2 KB |
| `findings\sub-traces.md` | 5 个 teammate 子会话（含 11 条矛盾、12 项未完成、30 条原文要点） | 88.6 KB |
| `findings\doc-verification.md` | 初版文档的对抗性核验（85 条断言） | 28.3 KB |

## 附录 C · 已知的矛盾与不确定性（诚实标注）

1. **任务书分段命名与实际内容不符**：已由 Lead 用 `coverage.txt` 核验为连续覆盖 turn 1→11，但个别片段标题（如"第 3 段=Web 前端构建"）与实际覆盖（turn 2 后半→turn 6 前半）不一致。
2. **`sleepTime` 单位**：会话一中期一度按"分钟"，经 proto-analyst 与 hid-harness 双向纠正为**秒**；最终代码与文档已是秒。
3. **`keepAlive` 是否防休眠**：Lead 原表述"保持唤醒"，实测是"延长窗口但不能阻止休眠"；两说并存但语义已澄清。
4. **测试计数口径**：platform-dev 发给 Lead 的收尾消息里 profile-store 写 13（合计 56，与 52 矛盾），文档已改为 9（合计 52）；**我实测当前为 57 项**（会话二 turn 1 又加了 3 条）。引用时应带时间点。
5. **探针命令数**：hid-harness 有两处数字（204 / 186），差异原因未在记录中说明，宜表述为"约 186–204 条"。
6. **同一反编译文件行数**：proto-analyst 记 3876 行、hid-harness 记 4162 行；后者为 `Get-Content` 实测，更可信。
7. **"真机验证通过"的口径**：hid-harness 的真机验证以**只读 + 宏写入闭环**为主，SetButton/DPI 写入由 Lead 用桥接验证；页面级真机 WebHID 链路留在会话一后续与用户侧。不要把"83/83 + 原生 WebHID 全通过"读成"整机全部功能已在真机端到端验证"。
8. **两次会话都没有用户侧的最终成功回执**，因此"问题已解决"只能由代码状态与测试数字支持（第 12 节），不能由用户确认支持。
9. **模型归属以轨迹中的 `model` 字段与 `model changed` 记录为准**：会话二 turn 1 = `opencode-go/kimi-k3`、turn 2 起 = `deepseek-account/deepseek-flash`（切换记录 `[seq 379]`）。用户在会话一中的角色是**唯一的信息源**，但其"三轮提问"为事后回忆，记录中为 2 次 `ask_user_question` 调用——两说并列出，不做取舍。
10. **成本数字（约 10 元 / 58% 额度）来自用户口述**，轨迹记录中不含计费信息，无法交叉验证。
