# 小米手环 9 第三方本地音乐播放器 —— 可行性 / 成本 / 方案评估

> 目标：正面解决 Azki Music 的两个痛点 —— ①重启后 App 消失要靠表盘重装；②下歌也要靠表盘。
> 结论口径：只用官方公开能力做骨架，把「未公开能力」单独隔离成可替换模块。

---

## 一、一句话结论

**能做得比 Azki 明显更好，但「出声」这一步是整个项目的生死线。**

- **下歌不再用表盘** → 有官方正路：`@system.interconnect`（手环9 支持、无阉割）+ `@system.file`。把歌从手机端 App 直接分片推进手环私有目录，完全跳过「每首歌打一个安装器表盘」。
- **重启不再全靠表盘** → 有两层解法：轻的是「启动器条目丢失」的恢复配方（社区已验证，零成本）；重的是伴生 App 检测到 `1001 APP_UNINSTALLED` 后一键恢复。
- **蓝牙耳机出声** → 公开 API 里没有配对接口。Azki 走的是未公开通路，这是它死绑 3.1.32 的原因。这一项需要一次**决定性逆向实验**（拆 Azki 的 rpk 看它 import 了哪个模块）才能定方案，是唯一可能导致我们要换目标产品形态的环节。

---

## 二、官方到底开放了什么（已逐页核对 iot.mi.com/vela）

### 手环侧 Vela 快应用，手环 9 的模块支持表

| 模块 | 手环 9 | 用途 |
|---|---|---|
| `@system.audio` | ✅ | src / currentTime(可写) / duration / volume / muted / meta / play·pause·stop / 事件 play·pause·ended·error |
| `@system.file` | ✅ | `internal://files` 私有沙箱读写、目录遍历 —— **下歌落盘就靠它** |
| `@system.interconnect` | ✅ | 手环 ↔ 手机 App 双向消息通道（借道小米运动健康已有的连接） |
| `@system.storage` / `volume` / `brightness` / `zip` / `prompt` / `router` / `app` | ✅ | 歌单持久化、解压、UI |
| `system.bluetooth.ble`（GATT） | ❌ | — |
| `system.fetch` / `request` / `network` | ❌ | 手环无网络能力 |
| 经典蓝牙 A2DP / 配对 | ❌ 无任何公开 API | ← 生死线 |

**关键点**：手环 9 的快应用**没有 BLE、没有网络**，这是固件级不支持，不是缺代码。
第三方项目 band-remote（2026-10 真机验证）原话：
> third-party quick apps on the Band 9 have no BLE and no network access … `system.interconnect`, which rides the existing Mi Fitness link, is the only working path.

### 手机侧 Android Wearable SDK v1.4（官方 demo 里可解包）

`getConnectedNodes` / `requestPermission(DEVICE_MANAGER, NOTIFY)` / `isWearAppInstalled` / `launchWearApp` / `MessageApi.sendMessage(byte[])` / `NotifyApi.sendNotify`。
**没有任何文件传输 API** → 传文件必须是我们自己在 `byte[]` 上造分片协议。这正好是我们要自研的部分，也正好是 Azki 没做好的部分。

### 后台续播

manifest 里 `config.background.features` 只白名单 `system.audio / system.request / system.geolocation`。**`system.audio` 在里面** → 息屏续播可以纯官方实现，这是 Azki 卡顿之外我们能稳赢的一项。

### 分发现实

Vela 三方应用无自助分发，官方原文「需要联系工作人员 … 进行合作发布」；真机装机入口是「小米运动健康 → 我的 → 关于 → Debug → 第三方应用 → Install third app」，而这个 Debug 版 App 要商务渠道拿。
**个人可行路径**：rpk 打包进表盘走 install-watchface 通道（社区已验证，用「表盘自定义工具」），或 AstroBox。

---

## 三、两个痛点，逐条判定

### 痛点② 下歌靠表盘 → **可以彻底干掉**

Azki 的链路：手机端把每首歌转码 → 塞进一个临时安装器表盘 → 推表盘 → 手环端释放 → 手机端清理 → 下一首。所以它下歌慢、还受 70% 分区限制、还要盯着亮屏等 99%。

我们的链路：
```
安卓 App                    小米运动健康（既有连接）           手环快应用
读本地音乐 ──► 转码 mp3 ──► sendMessage(分片) ──► interconnect.onmessage
                                                   │
                                            file.write(internal://files/music/*.mp3)
                                                   │
                                        storage 写歌单索引 / 封面
```
加分项（Azki 完全没有）：
- **断点续传**：分片带 offset + crc，断连后从 last ack 继续，不用从头再来
- **后台推送**：手机端前台服务推，手环端后台写，都不需要亮屏等 99%
- **手机端本地转码**：码率/声道可调，音量归一（ReplayGain），手环端只管播
- **空间自检**：推之前先问手环剩余空间，按剩余容量排队，不会推到一半 70% 爆掉
- **批量队列 + 失败重试**：整套歌单一次排好，中途拔连自动续

唯一代价：**传输速率待实测**。interconnect 是消息通道，估计在几十 KB/s ~ 数百 KB/s；一手 96kbps mono 的 4 分钟歌 ≈ 2.8MB，最坏情况一首歌要几十秒。这个数必须在第一周测出来，它直接决定「单曲时长上限」和「是否要预转码到 48~64kbps」。

### 痛点① 重启后 App 消失 → **分两种情况，都能改善**

社区实测（bandbbs 13016）：「重启后『更多』文件夹消失」，多人回复的解法是**在手机上往『更多』里拖一个自带应用即可恢复**。
bandbbs 12213 另一条：「通过表盘跳转到快应用实现保留，换绑设备不会删表盘，快应用从 Android 换绑 iOS 只是隐藏并没有删除」。

所以现象至少有两层：
1. **只是启动器条目丢了**（App 本体还在）→ 零成本恢复配方，做成 App 内「一键修复」或图文引导即可，**不需要表盘**。
2. **本体真被清了** → Azki 的做法是留一个安装器表盘手动点。我们的做法：伴生 App 通过 interconnect 探测 `1001 APP_UNINSTALLED`，一旦发现就在手机上弹「一键重装」，把重装从「找表盘→点亮→等 99%→点绿键→重启」压成一次点击。
   这一层仍要用表盘通道（这是手环 9 绕不开的系统限制），但**从"每次都靠它"降级成"只在真丢时兜底"**。

诚实结论：**痛点① 不能 100% 消灭，但能从"每次都要"降到"极少兜底"；痛点② 可以 100% 消灭。**

---

## 四、技术方案（三个阶段，风险前置）

### 阶段 0：决定性实验（3~5 天，先做，决定后面怎么走）
1. 搞到 Azki Music 的 rpk（从已装手环 dump、或表盘包里提取），**unzip 后看 app.js 里的 require/import 列表**。
   - 如果出现带 `bluetooth`/`bt`/`a2dp` 字样的模块名 → 我们直接照抄这个模块名，audio 输出问题当场解决，项目按「完整替代品」做。
   - 如果只有 `system.audio` → 说明配对是在系统级完成的：**去手环 9 系统设置里找有没有隐藏的蓝牙耳机配对页**（Azki 文档里「点搜索→选耳机确认配对」很可能只是唤起系统配对 UI）。
   - 如果两者都否定 → Azki 有 native 组件，我们要么走 Canopus 内核模块路线（手环 9 目前 `capabilities=[]` 未适配，风险极高），要么改产品形态（见第五节）。
2. 用官方 demo 起一个最小 rpk，实测 **interconnect 真实吞吐 + 最大单片大小**。
3. 实测 `@system.audio` 在手环 9 上能否出声（哪怕用蓝牙耳机在系统里先手动配上）。

**阶段 0 的产出直接决定要不要投后面的钱。**

### 阶段 1：播放器主体（纯官方 API，无风险，可并行）
- 手环端 rpk：歌单/播放/进度/音量/封面主题色/后台续播（`background.features: system.audio`）/存储自检/断点续传接收端
- UI 底板直接复用官方 `wearable/player` 模板（`src/{manifest.json,app.ux,pages/{player,list,volume}}`），改造成 192×490 胶囊屏布局
- 手环端 manifest 关键项（取自 band-remote 真机验证值）：`minPlatformVersion: 1000`、`designWidth: 192`、`permissions: [{origin:'*'}]`、`display.backgroundColor: '#000000'`

### 阶段 2：安卓伴生 App（下歌的关键）
- Kotlin + Wearable SDK v1.4（`xms-wearable-lib_1.4_release.aar`，官方 CDN 可下）
- **签名必须与 rpk 同源**，否则 SDK 的 `SignatureVerifyFailedException` 会**静默失败**（手机端永远显示 not connected）。做法：同一 keystore → `keytool -importkeystore` 转 p12 → `openssl pkcs12 -nodes` 拆出 `private.pem` + `certificate.pem` 放进 rpk 的 `sign/release/`
- 音乐库读取（MediaStore）+ 本地转码（MediaCodec → mp3）+ 分片推送 + 前台服务 + 断点续传状态机 + 重装探测

### 阶段 3：装机与恢复体验
- rpk → 表盘通道打包（`ebook.bin` 那套）做成一条命令
- 「重启后修复」图文/一键流程
- 可选：AstroBox 上架

---

## 五、成本

### 人力（AI 辅助 + 子代理并行，按自然日估）

| 阶段 | 内容 | 人天 | 自然日（并行 2~3 个子代理） |
|---|---|---|---|
| 0 | 决定性实验（rpk 逆向 + 吞吐实测 + 出声验证） | 3~5 | 2~3 |
| 1 | 手环端快应用（UI/播放/歌单/续传接收） | 6~9 | 3~4 |
| 2 | 安卓伴生 App（SDK/签名/转码/推送/续传） | 8~12 | 3~4 |
| 3 | 装机工具链 + 恢复体验 | 2~3 | 1~2 |
| — | 联调与真机坑（签名同源、连接被抢、卡顿） | 4~6 | 2~3 |
| **合计** | | **23~35 人天** | **约 8~12 个自然日** |

### 模型成本：**0**
所有代码生成/逆向分析走本机反代网关（已实测出流式 completion）：
- `127.0.0.1:42831` workbuddy CN（`deepseek-v4.1-flash` 等）
- `127.0.0.1:42841` workbuddy global（`glm-5.3` / `glm-5.3-flash`）
- `127.0.0.1:42871` qoder CN（`model=auto`）
不动你的 DeepSeek 官方 API。

### 开发环境已验证可用（不需要真机就能出 rpk）
- node v24.19.0 / npm 11.17.0；`aiot-toolkit` 2.0.5
- **坑已踩完**：默认装的 `@rspack/binding-win32-x64-msvc` 是坏包（19MB），会报 `Cannot find native binding` / `is not a valid Win32 application`。修复：
  ```bash
  rm -rf node_modules/@rspack/binding-win32-x64-msvc
  npm i @rspack/binding-win32-x64-msvc@1.7.11 --registry=https://registry.npmmirror.com
  node node_modules/aiot-toolkit/lib/bin.js build   # 必须这样调，npx aiot 会走错 node
  ```
- 已在本机成功产出签名 rpk（17KB，含 `META-INF/CERT`）

---

## 六、风险排序

| # | 风险 | 影响 | 处置 |
|---|---|---|---|
| 1 | **手环 9 上耳机配对无公开 API** | 致命：没有出声通道，产品只剩「手环端音乐库管理」 | 阶段 0 先做；两个备用剧本见第七节 |
| 2 | interconnect 吞吐太低 | 高：单曲限定 1~2 分钟 | 阶段 0 实测；不够就预转码 48kbps mono |
| 3 | 签名不同源 → 静默失败 | 高：极难排查（表现为一直 not connected） | 一套 keystore 贯穿 APK+rpk，写进构建脚本 |
| 4 | 小米运动健康抢连接 | 中：传输中断 | 引导「连接新手机」模式 + 前台服务 + 断点续传 |
| 5 | 手环 9 用户分区 70% 上限 | 中 | 空间自检 + 队列调度 |
| 6 | 固件升级断掉未公开接口 | 中（仅当走剧本 B） | 版本检测 + 降级提示 |
| 7 | 不能上官方商店 | 低 | AstroBox / 表盘自定义工具分发 |

---

## 七、如果阶段 0 判定「耳机配对走不通」，两个备用剧本

**剧本 A｜手环端音乐库 + 手机端播放（不碰音频栈，纯官方 API，100% 能做成）**
手环当「遥控器 + 播放列表显示器」，音频从手机走。价值点：手环端歌单管理、封面、歌词、音量、播放控制、断点续播全部官方可做；痛点和风险都归零。做得比 Azki 差的地方是「不脱离手机」，但这是唯一 0 风险能交付的版本。

**剧本 B｜完全替代（需攻破未公开接口）**
逆向 Azki 使用的模块 / 或等 Canopus 补齐手环 9 的 `capabilities`（现为 `[]`、标注 "Nothing here is device verified"）。周期不可控，且每次固件升级都可能断。

---

## 八、需要你拍板的三件事

1. **阶段 0 先做吗？**（3~5 天，只为一个答案：手环 9 上能不能让快应用自己出声。这个答案决定项目是「完整替代品」还是剧本 A。）
2. **伴生安卓 App 你能装自签 APK 吗？**（下歌链路必须它；需要一台安卓机 + 允许未知来源安装）
3. **交付形态**：先给我一个能在你手环上跑起来的最小可玩版（能导入、能列表、能播、能续播），还是一次性做完整套（含封面主题色、歌词、断点续传 UI）？

---

*本文所有能力结论均来自 iot.mi.com/vela 官方文档逐页核对、官方 demo 解包（Wearable SDK v1.4 PDF + rpk 工程）、以及 band-remote / bandbbs / Azki 官方 FAQ 的实测记录。*
