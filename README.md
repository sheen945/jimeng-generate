# jimeng-generate

本机唯一默认的 AI 生图 / 生视频通道：通过本机 New API 网关调用即梦（Jimeng）双账号服务，任何画图、配图、海报、封面、视频生成需求一律走本技能，禁止走 WorkBuddy 内置 ImageGen。

## 简介

这是一个 WorkBuddy / Claude Code 技能，把「调用即梦生成图片和视频」这件事在本机固化成一条唯一通道：智能体收到任何生图/生视频需求时，不再询问 Key 和地址，直接按技能内写死的链路配置与流程执行，保证输出质量和产物归置的一致性。

它解决的问题：即梦免费 API 服务（jimeng-free-api）跑在本机 8000 端口、带双账号轮询与失败冷却，但它的鉴权方式（`jm_` Key / sessionid）和 WorkBuddy 的统一令牌体系不兼容；New API 网关（3000 端口）做了这层适配，但生视频必须走 `chat/completions` 而非 `videos/generations`、双账号冷却会偶发 1015 错误、返回的是会过期的临时签名链接——这些坑全部已在技能里写明解法。

适合人群：在本机部署了 jimeng-free-api + New API 网关、希望智能体稳定调用即梦生图/生视频的用户。

## 功能特性

- **唯一通道约定**：所有「画一张图、生成图片、做图、配图、海报、封面、AI 生图、text-to-image」及视频生成需求一律优先走本技能，明确禁止走 WorkBuddy 内置 ImageGen。触发词：即梦画图、即梦生成、画个、画一张、生成一张图、帮我做图、jimeng 生图/生视频。
- **三层链路架构**：WorkBuddy → New API 网关（`http://127.0.0.1:3000/v1`，`sk-` 令牌认证）→ 即梦服务（`http://localhost:8000`，开机自启的计划任务 JimengFreeAPI-Autostart，内部处理双账号轮询与失败冷却）。严禁拿 `sk-` Key 直连 8000——会报 1015 登录校验错误。
- **生图能力**：9 个图像模型（jimeng-image-3.0 ~ 5.0-pro），规则是「一律用当前最强模型」（截至 2026-09 实测最强为 `jimeng-image-5.0-pro`），拿不准就查 `/v1/models` 挑版本号最高的 pro 模型。走标准 `POST /v1/images/generations`，支持方图/竖图/横图多种尺寸。
- **生视频能力**：11 个视频模型（默认 `jimeng-video-seedance-2.5`，求快用 `seedance-2.0-fast`）。走 3000 网关必须用 `POST /v1/chat/completions`（网关不认 `/v1/videos/generations`，返回 404）；比例和时长通过提示词控制；有首帧参考图时画面跟随参考图比例。
- **1015 重试铁律**：双账号轮询撞上冷却账号时会返回 HTTP 200 包裹的 `{"code":-2001,...1015}`，必须循环重试直到响应里出现 `choices`，每次间隔 10 秒，实测最多 4 次内成功、脚本按 8 次封顶。
- **临时链接即时落盘**：生图返回的 `data[0].url` 和视频返回里 `![video](...)` 的链接都是**会过期的临时签名链接**，必须立刻下载到本地；产物统一存到固定目录，文件名规范 `主题-YYYYMMDD-HHMM.png`。
- **图生图 / 参考图**：参考图参数名是 `filePath`（不是 `image`），支持本地路径、http(s) URL、base64 data URI，最多 10 张；保人物的提示词必须写明性别+年龄+关键特征，否则可能换人甚至换性别。
- **IP 形象贴纸动画专项经验**（叠加到实拍视频、剪映色度抠图用）：角色本身是绿色时必须改用品红底 `#FF00FF` 而非绿幕；三视图参考图推荐「整张直接当参考图」的路线 A（形象最忠实，代价是第 0 帧需剪掉 0.1~0.2 秒、比例跟随参考图），路线 B（抠正面单视图换底做首帧）已实测翻车（自动裁切缺角）；提示词必带「镜头固定不动，角色始终全身完整在画面中央」防止推镜出画。
- **完整排错手册**：1015 错误的两种成因（走错入口 / 撞上冷却账号）及各自解法；3000 无响应 = New API 没启动；8000 无响应 = 即梦服务挂了；账号异常/额度不足 = 去 8000 管理后台查账号池。

## 工作原理 / 技术栈

```
WorkBuddy 智能体
   │  Authorization: Bearer sk-...（New API 令牌）
   ▼
New API 网关  http://127.0.0.1:3000/v1
   │  （适配层：OpenAI 兼容接口 → 即梦协议）
   ▼
jimeng-free-api  http://localhost:8000
   │  （双账号轮询、失败自动冷却；开机自启计划任务）
   ▼
即梦（Jimeng）图像/视频生成服务
```

- 生图：`POST /v1/images/generations`（标准 OpenAI 图像接口形态），耗时约 10-20 秒，超时设 180 秒。
- 生视频：`POST /v1/chat/completions`（多模态消息体，纯文生视频只给 text，带参考图则 content 数组里加 `image_url`），每条 5 秒视频约 1.5~5 分钟。
- 调用方式：curl / 任意 HTTP 客户端；技能内所有命令均可直接复制执行。

## 安装与使用

把本仓库目录复制到技能目录，文件夹名保持 `jimeng-generate`：

- WorkBuddy / CodeBuddy：`~/.workbuddy/skills/jimeng-generate/`
- Claude Code：`~/.claude/skills/jimeng-generate/`

重启会话后即可通过触发词自动匹配。

**前置配置**：参考 `.env.example`，需要本机 New API 网关令牌（环境变量 `NEWAPI_TOKEN`）；同时确保 New API（3000）与 jimeng-free-api（8000）两个服务都在运行。

**生图示例**：

```bash
curl -s -m 180 http://127.0.0.1:3000/v1/images/generations \
  -H "Authorization: Bearer sk-YOUR_NEWAPI_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{"model":"jimeng-image-4.0","prompt":"<提示词>","size":"1024x1024","response_format":"url"}'
```

返回的 `data[0].url` 立即 curl 下载到本地（链接会过期）。

**生视频要点**：用 `POST /v1/chat/completions`，提示词里写明时长与比例（如「时长5秒，方形1:1画面」）；从 `choices[0].message.content` 中用正则 `https?://[^\s)\]"'\\]+` 提取视频链接并立刻下载；遇 1015 按 10 秒间隔重试。

**交付**：生成完成后用 present_files 把本地文件给用户看，并附一句：用的哪个模型、提示词、文件路径。

## 项目结构

```
jimeng-generate/
├── SKILL.md        # 技能定义（frontmatter + 完整链路配置、流程、排错手册）
├── README.md       # 本文件（中文说明）
├── README_EN.md    # 英文说明
├── .env.example    # 环境变量样例（NEWAPI_TOKEN）
└── .gitignore      # 常规排除规则
```

技能提到的参考脚本（如 `gen_songbao.py`、`make_first_frame.py`）与完整 API 文档（`jimeng-free-api-all/API.md`）位于本机其他工作目录，不在本仓库内。

## 注意事项

- **强本机环境绑定**：链路配置（3000/8000 端口、产物存放路径）均为本机实测值，换机器部署需先搭好 New API + jimeng-free-api 两个服务并相应调整路径。
- **切勿直连 8000**：即梦服务只认自己的 `jm_` Key 或明文 sessionid，用 `sk-` 令牌直连必报 1015；所有调用必须走 3000 网关。
- **链接时效**：生图/生视频返回的都是临时签名链接，拿到后必须立刻下载，过期不候。
- **模型时效性**：「当前最强模型」以 `/v1/models` 实时查询为准，不要硬编码低版本模型。
- **IP 素材纪律**：除非用户明确要求，不要自行加工用户的 IP 参考图（裁切/换底），路线 B 已有翻车记录。
- 令牌属于敏感信息，请勿把真实 `sk-` 令牌提交进仓库（`.gitignore` 已排除 `.env`）。

## License

MIT

## 作者

sheen945
