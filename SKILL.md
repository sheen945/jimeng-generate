---
name: jimeng-generate
description: 本机唯一默认的 AI 生图/生视频通道，通过本机 New API 调用即梦（Jimeng）双账号服务。任何「画一张图、生成图片、做图、配图、海报、封面、创意设计配图、AI 生图、text-to-image、image generation」以及视频生成需求，一律优先用本技能，禁止走 WorkBuddy 内置 ImageGen。触发词：即梦画图、即梦生成、画个、画一张、生成一张图、帮我做图、jimeng 生图/生视频。服务已开机自启，双账号自动轮询+失败冷却。
agent_created: true
---

# 即梦画图/视频生成技能

## 一句话用法

用户说「即梦画一张 XX」「用即梦做段 XX 视频」→ 直接按本技能流程执行，不要再问 Key 和地址（下面都有）。

## 链路配置（2026-09-15 实测通过）

- 调用入口（走 New API 网关）：`http://127.0.0.1:3000/v1`
- 认证头：`Authorization: Bearer sk-YOUR_NEWAPI_TOKEN_HERE`
- 底层即梦服务：`http://localhost:8000`（开机自启，计划任务 JimengFreeAPI-Autostart；双账号轮询、失败自动冷却由它内部处理，不用管）
- ⚠️ 不要拿 sk- Key 直连 8000 端口——即梦服务只认自己的 `jm_` Key 或明文 sessionid，直连会报 1015 登录校验错误
- 生成产物统一存：`C:\Users\Administrator\WorkBuddy\AI做视频相关\即梦生成\`

## 可用模型

画图（**一律用当前最强模型**，截至 2026-09 实测最强为 `jimeng-image-5.0-pro`；拿不准就先查 `/v1/models` 挑版本号最高的 pro 模型，禁止默认用低版本模型）：
- jimeng-image-3.0 / 3.1 / 4.0 / 4.1 / 4.5 / 5.0 / 5.0-pro 等 9 个

视频（默认用 `jimeng-video-seedance-2.5`，要快用 `seedance-2.0-fast`）：
- jimeng-video-seedance-2.5、seedance-2.0-fast、wan-3.0 等 11 个
- 完整列表随时可查：`curl -s http://127.0.0.1:3000/v1/models -H "Authorization: Bearer sk-..."`

## 生图流程

```bash
curl -s -m 180 http://127.0.0.1:3000/v1/images/generations \
  -H "Authorization: Bearer sk-YOUR_NEWAPI_TOKEN_HERE" \
  -H "Content-Type: application/json" \
  -d '{"model":"jimeng-image-4.0","prompt":"<提示词>","size":"1024x1024","response_format":"url"}'
```

- 返回 JSON 里的 `data[0].url` 是**临时签名链接，会过期**，必须立刻 curl 下载到本地
- 文件名规范：`即梦生成/主题-YYYYMMDD-HHMM.png`
- 生图耗时约 10-20 秒，超时设 180 秒
- 常用尺寸：1024x1024（方图）、1024x1536 或按模型支持的竖图/横图（抖音竖屏用竖图）

## 生视频流程（2026-09-24 实测修正）

**走 3000 网关必须用 `POST /v1/chat/completions`**——网关不认 `/v1/videos/generations`（404），该路径只能直连 8000 用（但直连又有 Key 校验问题，所以实际一律走 chat 通道）。

- 请求：`{"model":"jimeng-video-seedance-2.5","messages":[{"role":"user","content":[{"type":"text","text":"<提示词>，时长5秒，方形1:1画面"},{"type":"image_url","image_url":{"url":"data:image/png;base64,..."}}]}]}`（带参考图就是多模态 content 数组；纯文生视频可只给 text）
- 返回：`choices[0].message.content` 里含 `![video](https://...vlabvod.com...)`，正则 `https?://[^\s)\]"'\\]+` 提取后立刻下载（链接会过期）
- 比例/时长通过提示词控制（如「时长5秒，方形1:1画面」）；有首帧参考图时画面跟随参考图比例
- **1015 重试铁律**：双账号轮询常撞上冷却账号返回 `{"code":-2001,...1015}`（HTTP 200 包裹），必须循环重试直到出现 `choices`，每次间隔 10 秒，实测最多 4 次内成功，脚本按 8 次封顶
- 耗时：每条 5 秒视频约 1.5~5 分钟
- 参考脚本（可直接复用）：`C:\Users\Administrator\WorkBuddy\松林路社区相关\视频分析-盐菜扣肉\gen_songbao.py`

### 做「IP 形象贴纸动画」（叠加到实拍视频，剪映色度抠图用）

- **绿幕不是万能的：角色本身是绿色（如松宝）必须改用品红底 `#FF00FF`**，否则抠图会把角色身体一起抠掉；提示词写「纯品红色背景（#FF00FF）始终均匀不变，无阴影」
- **IP 三视图参考图两条路线（2026-09-24 实测）**：
  - **路线A（推荐，形象最忠实）**：整张三视图直接当参考图 + 提示词写「严格以参考图中左侧的正面形象为准…只生成正面这一个角色，不要侧面和背面形象，不要任何文字标签」。产出形象高度还原；代价：①第 0 帧是三视图原图，需剪掉前 0.1~0.2 秒；②画面比例跟随参考图（16:9 图出 640x360），提示词里的方形比例不生效
  - 路线B：抠正面单视图换品红底做首帧（参考脚本 `视频分析-盐菜扣肉\make_first_frame.py`）。画面干净，但**自动裁切极易缺角（2026-09-24 翻车：头顶和右侧被裁掉被用户打回）**，且模型会对着加工图二次发挥、形象走样；除非用户明确要求，否则别自己加工用户的 IP 素材
- 提示词必带「镜头固定不动，角色始终全身完整在画面中央」，否则模型爱加推镜导致角色出画
- 交付时提醒用户：剪映 → 画中画 → 色度抠图 → 吸取品红色

## 交付

生成完成后用 present_files 把本地文件给用户看，并附一句：用的哪个模型、提示词、文件路径。

## 排错

- 1015 登录校验错误 → 走错入口了，必须走 127.0.0.1:3000 不是 8000；若入口没错仍偶发 1015，是双账号池轮询到冷却中的账号，sleep 5-8 秒重试，一般 2 次内成功

## 图生图 / 参考图（2026-09-18 实测）

- 参考图参数名是 `filePath`（**不是** `image`），支持：本地路径（服务端文件系统）、http(s) URL、`data:image/jpeg;base64,...`；最多 10 张
- 示例：`{"model":"jimeng-image-4.0","prompt":"...","size":"1024x1536","response_format":"url","filePath":"data:image/jpeg;base64,..."}`
- 保人物参考图的提示词必须写明性别+年龄+关键特征（如「中年女性、细框眼镜、短卷发」），否则可能换人甚至换性别
- 完整文档：`C:\Users\Administrator\WorkBuddy\AI做视频相关\jimeng-free-api-all\API.md`
- 3000 端口无响应 → New API 没启动
- 8000 端口无响应 → 即梦服务挂了，检查计划任务或手动重启
- 提示账号异常/额度不足 → 即梦双账号中有一个失效，去 http://localhost:8000 管理后台看账号池状态
