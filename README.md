# 📜 Poetry Copybook Skill

> 每天让 AI 为你寄一封带诗词故事的「诗笺」字帖

把古诗词变成可以练字的 PDF，每天早上 8 点自动生成、自动发邮件。**邮件正文不仅有完整诗文，还有词人小传、创作背景和一句话精髓**——一边练字，一边学诗。

## ✨ 特性

- 🎨 **古风排版**：吴玉生行楷、方格、实心字，符合传统练字习惯
- 📖 **完整教学**：每天的邮件包含 完整诗文 + 词人小传 + 创作背景 + 一句话记住
- 🤖 **自动推送**：Cron job 每天 8:00 自动生成 + 自动发邮件，无需手动操作
- 🎲 **海量诗词池**：基于 [chinese-poetry](https://github.com/chinese-poetry/chinese-poetry) 数据库（~5万首），自动排除教材已学过的 173 首
- 📎 **便携 PDF**：每份字帖是 A4 PDF，打印即练
- 🔧 **可定制**：支持指定诗词、字体切换、行/列数自适应

## 📧 邮件样例

```
📝 今日练字字帖 - 满江红·送廖叔仁赴阙【宋】严羽

[完整诗文]
日近觚棱，秋渐满、蓬莱双阙。
正钱塘江上，潮头如雪。
...

📜 词人小传
严羽，南宋诗论家、诗人，字丹丘，自号沧浪逋客，
世称严沧浪。所著《沧浪诗话》被誉为四朝诗话第一人。

🧭 创作背景
这是严羽送友人廖叔仁赴临安任职时所作——外送友人
赴阙、内叹自己仕途失意，是一首双线词。

💡 一句话记住它
送友赴阙、自伤老去——严羽一辈子没出仕，借送别
词浇自己胸中块垒。

[附件] 字帖 PDF（吴玉生行楷，方格练习）
```

## 🛠️ 架构

```
每日 8:00
  ↓
auto_generate_zitie.py（Playwright 渲染 HTML → PDF）
  ↓
stdout: {"status":"success","pdf_path":"...","title":"...","author":"..."}
  ↓
Hermes cron 捕获 JSON
  ↓
web_search 查询作者小传 + 创作背景
  ↓
SMTP_SSL 发送 HTML 邮件（带 PDF 附件）
  ↓
收件箱收到「诗笺」
```

**核心原则**：自动化脚本只负责生成 PDF 并输出 JSON，邮件发送（含背景故事组装）由调用方完成——职责分离，便于测试。

## 📦 部署

### 前置条件

- Python ≥ 3.9
- Node.js ≥ 20（Playwright 运行时）
- 一个支持 SMTP 的邮箱（推荐 QQ 邮箱）
- chinese-poetry 数据库（可选，无则用保底池）

### 1. 克隆项目

```bash
git clone https://github.com/xiaoxuan353/Poetry-Copybook-skill.git
cd Poetry-Copybook-skill
```

### 2. 安装字帖生成器依赖

`auto_generate_zitie.py` 依赖 Playwright + Node.js。完整字帖生成器项目位于 [chinese-calligraphy-generator](https://github.com/...)，请先部署好：

```bash
# 项目结构（仅展示本 skill 部分）
/root/chinese-calligraphy-generator/
├── auto_generate_zitie.py      # 字帖生成脚本
├── zitie_output/                # PDF 输出目录
└── chinese-calligraphy-generator/  # 静态网页（含字体、HTML）
```

### 3. 配置 SMTP

在 SKILL.md 「邮件正文模板」章节中，将以下占位符替换为你的真实配置：

```python
your_email@example.com     # 你的邮箱地址
your_smtp_auth_code         # 你的 SMTP 授权码（不是登录密码）
```

QQ 邮箱获取授权码：设置 → 账户 → POP3/IMAP/SMTP/Exchange/CardDAV/CalDAV服务 → 生成授权码

### 4. 集成到 Hermes

将 `SKILL.md` 放到 `~/.hermes/skills/productivity/poetry-copybook-skill/SKILL.md`。

创建 Cron Job：

```bash
hermes cron create \
  --name "每日字帖" \
  --schedule "0 8 * * *" \
  --prompt "$(cat cron_prompt.txt)" \
  --skills productivity/poetry-copybook-skill
```

`cron_prompt.txt` 内容参见 SKILL.md「Cron Job 配置」章节。

## 🎯 使用场景

- 🖋️ **每日练字**：固定时间收到字帖，培养练字习惯
- 📚 **古诗学习**：每天读一首诗 + 了解背景，长期积累
- 🎁 **送礼**：把字帖 PDF 发给亲友做礼物
- ⏰ **早间仪式**：把"读诗笺"当作一天的开启仪式

## 🔧 进阶配置

### 指定某首诗词

把要练的诗词写入 `pending_poems.txt`（项目自带支持）：

```
赠汪伦
李白
李白乘舟将欲行，忽闻岸上踏歌声。
桃花潭水深千尺，不及汪伦送我情。
```

下次执行会优先消费此文件，消费后自动擦除。

### 字体切换

字帖生成器支持多种内置字体（吴玉生行楷、瑞美加楷书、田英章楷书），修改 `config.json` 中的 `FONT_FAMILY` 即可。

### 教材诗词排除

内置 `TEXTBOOK_EXCLUDED` 集合（173 首小学+初中部编版教材诗词），不会重复推送。更新排除列表参见 SKILL.md「教材诗词排除机制」章节。

## 🐛 故障排查

| 问题 | 排查 |
|:---|:---|
| PDF 没生成 | 检查 Playwright 是否安装：`node -e "const {chromium}=require('playwright');..."` |
| 邮件发不出去 | 检查 From 头是否严格为邮箱地址、授权码是否正确 |
| 缺字/豆腐块 | 检查字体 fallback 链，参见 SKILL.md「字体回退机制」|
| 诗词被截断 | 增大 `rows` 上限（当前 28）|
| 教材诗词出现 | 更新 `TEXTBOOK_EXCLUDED` 集合 |

详细故障排查见 SKILL.md。

## 📜 License

MIT

## 🙏 致谢

- [chinese-poetry](https://github.com/chinese-poetry/chinese-poetry) — 诗词数据库
- [chinese-calligraphy-generator](https://github.com/...) — 字帖渲染引擎
- [Hermes Agent](https://nousresearch.com/hermes) — AI 智能体平台
