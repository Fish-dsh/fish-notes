# fish-notes · 杂项技术备忘

一个人维护的笔记仓库。写下来的都是**自己实测过的东西**，不是转述别人说的、也不是模型现编的。

适用读者：想知道"这些工具到底能不能用、用起来多快多贵"的人。

---

## 里面有什么

### [docs/model-registry.md](docs/model-registry.md) · 反代模型能力登记表

把桌面端 AI 软件（WorkBuddy / Trae / Qoder）的登录态反代成本地 OpenAI 兼容端口后，70 个模型的实测结果。

- 哪些能用、哪些已经死了（附原始错误码）
- 延迟、吞吐、工具调用、长上下文取针、诚实性五个维度的对比
- 一个具体到能救命的发现：**kimi 系模型只要历史里有一条空 content 的 assistant 消息，上游直接 400** —— 这就是"子代理一出生就死"的根因

### [docs/workbuddy-quota.md](docs/workbuddy-quota.md) · 额度怎么看、怎么省

积分制的 AI 产品，到底一分能换多少字。实测结论：**读进去的字便宜，写出来的字贵十倍**。还有哪个模型不扣分。

### [docs/mi-band-9-music-player.md](docs/mi-band-9-music-player.md) · 小米手环 9 本地音乐播放器可行性评估

手环上第三方音乐 App 的两个老大难（重启即消失、下歌必须靠表盘），能不能只靠官方公开能力绕过去。含成本估算和隔离方案。

---

## 几个约定

- 数字都标了测量日期。这类东西过期很快，看到旧的先自己重测一遍。
- 涉及具体的人的 IP、手机号、昵称，一律替换成占位符（`BOARD_IP`、`ACCOUNT_B`、`FRIEND_A`）。
- 推断和实测分开写。凡是我没验证过的，会明确标成"假设"或"待验证"。

## 相关仓库

- [douyin-spark](https://github.com/Fish-dsh/douyin-spark) —— 抖音自动续火花，Python + Playwright 写的无人值守脚本。本仓库的 `model-registry` 最初就是为了给它挑模型才做的。

## License

MIT
