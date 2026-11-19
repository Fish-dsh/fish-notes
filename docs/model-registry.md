# 反代模型能力登记表（实测，2026-11-01）

> 来源：三端口 70 个模型全量探针 + 三轮专项探针 + 1 次重复延迟采样。全部为真实 HTTP 请求，非猜测。

## 一、可用性总览

| 端口 | provider | 模型数 | 状态 |
|---|---|---|---|
| 42831 | workbuddy-cn | 15 | 12 可用 |
| 42851 | trae-cn | 45 | 43 可用，2 死 |
| 42871 | qoder-cn | 10 | **全灭（429 Billing daily count exceeded）** |

**结论：qoder 全线额度耗尽**。今天它的 701 次调用cost=0，但现在已经不可用；继续指它 = 立刻失败。

## 二、硬失败清单（不要去用）

| 模型 | 症状 | 错误原文 |
|---|---|---|
| qoder 全部 10 个 | 每请求 429 | `{"error":{"message":"Billing daily count exceeded","type":"upstream_error"}}` |
| trae/kimi-k3.1-pro | 每请求 402 | `code 1005 · {"plan":5}` 套餐等级不够 |
| trae/Doubao_1_6 | 全部探针超时 25s | 无响应 |
| trae/minimax-m2.1 | basic 探针超时 | 首轮不吐字，不稳 |
| workbuddy/kimi-k3.1-pro | 收到空 assistant 消息直接 400 | `{"code":11151,"msg":"a message has empty content"}` |
| workbuddy/kimi-k3-1 | 同上 | 同上 |
| workbuddy/kimi-k2.8-preview | 同上 | 同上 |

### ⚠️ 最高价值发现：kimi 系 + 空 content = 必死

`workbuddy` 的 kimi 系列只要会话历史里出现一条 `{"role":"assistant","content":""}`，上游直接 400 报 11151。
**这就是今天那批「一出生就死」的子代理的根因**：子代理历史里有空 assistant 消息时，kimi 直接拒绝。
→ 用 kimi 时必须保证 assistant 消息 content 非空（哪怕是单个空格）。

## 三、能力维度矩阵

维度说明：R2=工具循环/严格指令/大海捞针（各 1 分）；honest=无工具时是否老实承认不知道；long=40k 字符长上下文取针。

### 第一梯队（全项通过 + 低延迟）

| provider | model | 中位总延迟 | 抖动 | 思考字符 | 工具 | 备注 |
|---|---|---|---|---|---|---|
| workbuddy | deepseek-v4.1-flash | **1750ms** | 269ms | 0 | 3/3 | **当前会话在用，最快最稳** |
| trae | glm-5-turbo | 1745ms | 301ms | 0 | 3/3 | 与上面并列最快 |
| workbuddy | deepseek-v4-pro | 3055ms | — | 0 | 3/3 | 同族更慢，无额外收益 |
| trae | glm-4.7 | 1753ms | 232ms | 0 | 3/3 | 低延迟大模型 |
| trae | glm-5.1 | 1920ms | 408ms | 0 | 3/3 | |
| trae | deepseek-v4.1-flash | 1921ms | 375ms | 0 | 3/3 | 比 workbuddy 同名慢 170ms |

### 高吞吐（适合批量、子代理并行）

| provider | model | tps | 中位延迟 | 备注 |
|---|---|---|---|---|
| trae | qwen-3.7-plus | **1049** | 2170ms | 吞吐王，但抖 1931ms，思考 83 字符 |
| workbuddy | hy3 | 810 | — | 思考 1294 字符，needle 探针返回空 |
| trae | qwen-3.5 | 648 | 1728ms | 快且吞吐高，思考 115 字符 |

### 明确有缺陷（按缺陷使用）

| provider | model | 缺陷 | 影响 |
|---|---|---|---|
| trae | minimax-m3.1-flash | 无任一探针内容（空回复） | 别看 tps 高，直接空响应 |
| workbuddy | step-5-preview | 40k 长上下文取针失败 + 工具结果回填不稳 | 长文档任务不可用 |
| workbuddy | hy3 | 大海捞针探针返回空 | 长 prompt 中部信息丢失 |
| trae | mimo-v2.6-flash | 严格指令探针超时 | 慢，不适合强约束输出 |
| trae | kimi-k2-0905 | 诚实性探针失败 | 会编造文件内容 |
| workbuddy | minimax-m3.1-flash | 抖动 1080ms | 延迟不可预测 |

## 四、场景选型建议

| 场景 | 首选 | 备选 |
|---|---|---|
| 主会话 / 需要低延迟 | workbuddy/deepseek-v4.1-flash | trae/glm-5-turbo |
| 子代理并行洗数据 | trae/qwen-3.7-plus | trae/qwen-3.5 |
| 需要严密指令跟随 | trae/glm-5.3-flashx | trae/glm-5.1 |
| 长文档 / 大上下文 | trae/glm-5.3 | workbuddy/deepseek-v4.1-flash |
| 当前不要碰 | qoder 全部、trae/kimi-k3.1-pro、trae/Doubao_1_6 | — |

## 五、探测方法（可复现）

脚本在本目录：`probe2.mjs`（全量可用性）、`round2.mjs`（agent 能力）、`round3.mjs`（速度/诚实/长上下文）、`latency.mjs`（重复延迟）。

复跑：`node probe2.mjs && node round2.mjs && node round3.mjs && node latency.mjs`

原始数据：`probe.jsonl`、`probe2.jsonl`、`probe3.jsonl`、`probe-lat.jsonl`、`probe-ranked.json`、`probe-latency.json`。
