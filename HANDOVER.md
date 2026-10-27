# Overlay Studio 交接文档

> 交接对象：WorkBuddy（接手的 AI Agent / 开发同学）
> 项目根目录：`d:\trae~ggg\jinrpentu`
> 最后更新：2026-10-27

---

## 1. 这是什么

一个**财经口播短视频动效卡片工作台**。输入一段口播文案 → 自动拆成数据卡片（KPI / 条形图 / 排行榜 / 仪表盘 / 折线 / VS / 时间线 / 标签）→ 导出 PNG、透明动效视频，或导入口播视频直接合成成片（免剪映）。

三个核心能力：

| 能力 | 实现位置 | 说明 |
|---|---|---|
| 文案 → 卡片结构 | `llm-core.js`（提示词）+ `index.html`（规则重建） | AI 只挑句子，卡型/数值由前端规则决定 |
| 卡片渲染与动效 | `index.html`（内联 SVG + CSS 动画） | 无框架、无构建，单文件 |
| 视频合成 | `server.js`（Node）+ 内置 ffmpeg | 抠像叠加，保留原声 AAC |

技术栈：**原生 HTML/CSS/JS 单文件 + Node 原生 http 模块**，无 npm 依赖（除了便携包里的 ffmpeg 二进制）。

---

## 2. 文件清单（清理后）

### 必须保留（运行所需）

| 文件 | 作用 |
|---|---|
| `index.html` | **全部前端**：UI、卡片规则、SVG 渲染、试听同步、导出（约 322KB，单文件） |
| `server.js` | 本地后端：静态服务 + 10 个 API（清单见 4.5 节） |
| `llm-core.js` | AI 提示词构建 + JSON 提取修复（前后端共用，Vercel 也用它） |
| `llm.config.json` | 本地私密配置：API Key、模型名、超时。**不入仓库** |
| `start.bat` | Windows 启动脚本（自动找 Node：PATH → 常见安装路径 → WorkBuddy 内置 Node） |
| `start.command` | macOS 启动脚本 |
| `allow_firewall_8123.bat` | 放行 8123 端口（管理员运行），手机/其他电脑访问必需 |
| `sync_deploy.bat` | 把 `index.html` + `llm-core.js` 同步到 `deploy-vercel\` |

### 可选保留（交付物 / 部署）

| 路径 | 作用 |
|---|---|
| `OverlayStudio-Portable/` | 便携包源目录：`runtime/`（Node 22 + ffmpeg 9.0）+ `index.html` / `server.js` / `llm-core.js` / `llm.config.json` / `start.bat` / `allow_firewall_8123.bat` |
| `OverlayStudio-Portable.zip` | 便携包成品（约 68.8MB），拷到没装 Node 的电脑解压即用。**⚠ 内含 `llm.config.json`（真实 API Key），禁止公开上传网盘/群/仓库** |
| `HANDOVER.md` / `handoff-prompt.md` | 交接文档与 WorkBuddy 开场指令（已推 GitHub） |
| `deploy-vercel/` | Vercel 部署目录：`api/llm.js`（Serverless 函数）、`vercel.json`、`.vercel/project.json`（已绑定项目） |
| `exports/` | 导出兜底目录（没有 `D:\下载` 时写这里，程序会自动创建） |
| `.trae/documents/` | 历史设计文档（6 篇），仅参考，不参与运行 |

### 已删除（本次清理）

176 项、约 **280.8MB** 调试残留：`_*.js` / `_*.png` / `_*.svg` / `_*.mp4` 自测产物、`_edge_prof*` Edge 测试用户目录（各 100MB+）、`_dl/`、`index.html.txt` 旧备份。**均无代码引用**，已确认 `index.html` 与 `server.js` 不含对这些文件的任何引用。

---

## 3. 三种运行形态（关键：功能不一样）

| 形态 | 地址 | AI 解析 | ffmpeg 合成 | 手机实时同步 | 依赖本机开机 |
|---|---|---|---|---|---|
| 本机工作台 | `http://localhost:8123` | ✅ | ✅ | ✅ | 是 |
| 局域网（其他设备） | `http://<本机IP>:8123` | ✅ | ✅ | ✅ | 是 |
| 便携包 | 解压后双击 `start.bat` | ✅ | ✅ | ✅ | 所在那台开机 |
| GitHub Pages | `https://dalu99956.github.io/doxiaogonztai/` | ❌ 自动关闭 AI 开关 | ❌ | ❌ | 否 |
| Vercel | `https://overlay-studio-cloud.vercel.app` | ✅ | ❌ | ❌ | 否 |

**为什么云端没有 AI/合成**：
- GitHub Pages 是纯静态，没有后端，Key 放前端会泄露且被 CORS 预检拦死（智谱对 OPTIONS 不返回 `Access-Control-Allow-Origin`）
- Vercel 有 Serverless 可代理 AI，但装不了 105MB 的 ffmpeg 二进制
- `vercel.app` 域名在国内被 DNS 污染/阻断（实测解析到 `157.240.8.50`，TLS 被重置），**必须绑自有域名才能国内直连**

---

## 4. 配置与密钥（交接重点）

### `llm.config.json`（本机私密，不要提交仓库）

```json
{
  "apiKey": "<智谱 API Key>",
  "baseUrl": "https://open.bigmodel.cn/api/paas/v4/chat/completions",
  "model": "glm-4-flash",
  "reasoningEffort": "low",
  "maxTokens": 4000,
  "temperature": 0.1,
  "timeoutMs": 120000,
  "thinking": false
}
```

要点：
- `glm-4-flash` 是**免费文本模型**，但不支持 `thinking` / `reasoning_effort`，传了会报 `1210`，所以 `thinking: false`
- 免费档排队严重，单次 60–120 秒，超时必须 ≥120000
- **语音识别（`glm-asr` / `glm-asr-2512`）不在免费范围**，实测返回 `余额不足（1113）`

### 其他凭据 / 账号

| 项 | 位置 | 交接方式 |
|---|---|---|
| 智谱 API Key | `llm.config.json`、Vercel 环境变量 `GLM_API_KEY` | 线下单独传递，**不要写进仓库或文档**；建议交接后重新生成 |
| GitHub 仓库 | `DALU99956/doxiaogonztai`（`main` 分支：`index.html` + `HANDOVER.md` + `handoff-prompt.md`，**不含 Key**） | 转移仓库所有权或加协作者 |
| GitHub Token | 曾用于 API 推送（`repo` 权限） | 建议作废重发，只给接手方 |
| Vercel 项目 | `overlay-studio-cloud`，`projectId: prj_7OP6ckh594fsNvzuEfLETkbDkAbi`，`orgId: team_GXleUFIpuAcfJQnh0OVKwrPE` | 转移项目或邀请成员 |
| 本机 Node | 便携包 `runtime\`，或 `C:\Users\userDL\.workbuddy\binaries\node\versions\22.22.2-6\node.exe` | 建议装系统级 Node 18+ 更省事 |

---

## 4.5 运行时配置清单（**改配置前必看，最容易踩坑的一节**）

### 4.5.1 端口与路径

| 项 | 位置 | 默认值 | 说明 |
|---|---|---|---|
| 端口 | `server.js` 第 13 行 `const PORT = 8123` | **硬编码 8123** | **没有环境变量**。要改端口必须同时改 `server.js` + `allow_firewall_8123.bat`（文件名和内容里的端口） |
| 导出目录 | `process.env.EXPORT_DIR` | `D:\下载` | 回退顺序：`EXPORT_DIR` → `D:\下载` → `~/Downloads` → 项目 `exports\`（自动创建并测可写） |
| ffmpeg | `process.env.FFMPEG_PATH` | 自动探测 | 顺序：`FFMPEG_PATH` → 程序目录 `runtime/`、`bin/`、`.` → `~\.workbuddy\binaries\ffmpeg\*\bin\` → PATH |
| 临时目录 | `os.tmpdir()/overlay-studio` | `%TEMP%\overlay-studio` | 上传素材按 `token.ext` 存放；**合成成功才删，失败/中断会残留**，无自动清理 |
| 监听地址 | `server.listen(PORT, '0.0.0.0')` | 全网卡 | 局域网可访问是刻意设计，不是漏洞 |

### 4.5.2 环境变量（**优先级高于 `llm.config.json`**）

`server.js` 的实际读取逻辑是 `process.env.X || config.X`，所以环境变量会盖掉配置文件。改了 json 不生效时先查环境变量。

| 变量 | 作用 | 备注 |
|---|---|---|
| `GLM_API_KEY` | 覆盖 `apiKey` | 只在服务端用，浏览器拿不到 |
| `GLM_MODEL` | 覆盖 `model` | |
| `EXPORT_DIR` | 覆盖导出目录 | 见上表 |
| `FFMPEG_PATH` | 指定 ffmpeg 可执行文件 | 便携包换机报"未找到 ffmpeg"时用它兜底 |

### 4.5.3 ⚠ `llm.config.json` 字段不能省（缺字段会掉进付费模型）

`readLLMConfig()` 的兜底默认值是**为另一个付费模型写的**，必须与下表对照检查：

| 字段 | 缺失时的默认值 | 后果 |
|---|---|---|
| `model` | **`glm-5.3-flash`（付费）** | 直接报 `余额不足（1113）`，看起来像 Key 失效 |
| `thinking` | **`true`**（`!== false` 判定） | 会给 `glm-4-flash` 传 `thinking` 参数 → 报 `1210` |
| `timeoutMs` | `30000` | 免费档要 60–120 秒，30 秒必超时 |
| `temperature` | `0.3` | 输出更飘，JSON 更容易坏 |
| `maxTokens` | `4000` | 一般够用 |

**结论：`model`、`thinking`、`timeoutMs` 三个字段必须显式写死，不能依赖默认值。**

### 4.5.4 Vercel 环境变量（云端另一套，与本地不同名不同默认）

| 变量 | 默认 | 说明 |
|---|---|---|
| `GLM_API_KEY` | 无（必填） | 缺了报"云端未配置 GLM_API_KEY 环境变量" |
| `GLM_MODEL` | `glm-4-flash` | |
| `GLM_BASE_URL` | 智谱 v4 chat/completions | |
| `GLM_TIMEOUT_MS` | **`55000`**（比本地 120000 短） | Serverless 限制，长文案可能超时 |
| `GLM_TEMPERATURE` | `0.1` | |
| `GLM_THINKING` | 不设置=关闭 | 只有值为 **`on`** 才带 thinking（给 `glm-5.x` 用） |
| `GLM_REASONING_EFFORT` | `low` | 仅在 thinking 开启时生效 |

### 4.5.5 API 端点全清单（`server.js`）

| 方法 | 路径 | 作用 |
|---|---|---|
| POST | `/api/llm` | 文案 → 卡片 JSON（代理智谱，Key 不出服务端） |
| GET | `/api/llm/status` | 是否已配置 Key（不含 Key 本身） |
| POST | `/api/save` | 导出文件落盘（body 上限 200MB），保存后自动弹出资源管理器 |
| GET | `/api/open-exports` | 打开导出目录 |
| POST | `/api/upload?ext=.mp4` | 流式上传素材，返回 `token`（供 align/composite 使用） |
| POST | `/api/align` | ffmpeg `silencedetect` 检测停顿点，返回 `pauses[]` + `duration` |
| POST | `/api/composite` | **视频合成（注意是 `composite` 不是 `compose`）**：chromakey 抠像叠加，`-c:a copy` 原样保留音轨 |
| GET | `/api/lan` | 返回本机局域网 IP + 端口（手机预览用） |
| POST/GET | `/api/state` | 电脑端推送 / 手机端拉取工作台状态（上限 20MB） |
| GET | `/api/state/stream` | SSE 长连接，状态变更实时推送 |

### 4.5.6 前端持久化与手机预览

| 项 | 说明 |
|---|---|
| 本地存档 | `localStorage['financeMotionCards.v1']`，存文案、卡片、张数、主题、比例 |
| 手机预览 | URL 加 `?preview=1` 进入只读预览模式；**手机与电脑的 localStorage 不互通**，同步完全依赖 `/api/state` + `state/stream` |
| AI 开关 | 静态托管（GitHub Pages 等非内网地址）自动关闭，避免每次生成白等 404 |

---

## 5. 数据流与设计不变量（改代码前必读）

```
文案 → [AI 只挑句子] → 前端按整句重建卡型/数值 → 三道硬校验 → 张数补齐/截断
     → SVG 渲染 → 预览（播放条驱动）→ 导出 PNG / 合成 MP4
```

### AI 只允许做 4 件事

1. 挑出值得做卡的**句子**
2. 给出短**标题**
3. 给出原始**数值、单位、左右对比值、标签**
4. `dir`（涨跌，仅参考）

### 明确禁止 AI 做（前端硬拦截）

| 拦截 | 代码位置 |
|---|---|
| `text` 必须逐字来自原文（忽略标点空白） | `normalizeLLMCard` → `normCh(script).indexOf(...) < 0` 直接弃卡 |
| 数字必须出现在**该卡自己的 text** 里（防张冠李戴） | 同上 |
| 有数据卡时，`title` / `quote` 纯文字卡全丢 | `isTextOnly` 过滤 |
| 无数字标签卡全丢 | `isNoNumTags` 过滤 |
| 占位标题（"关键数据"）强制替换 | 标题兜底逻辑 |
| 视觉字段（颜色/字号/布局/动画）一律忽略 | 渲染层只认自己的主题系统 |
| 涨跌方向最终由规则复判 | `crossCheckDir` |

### 张数逻辑

- `张数 = 0` → **自动**：AI 卡 + 本地数据卡合并，同句去重（整句包含关系），按原文顺序排序，**不强凑、不拆碎片**
- `张数 > 0` → **强制出满 N 张**：`expandCardsTo` 拆多数据点图卡 → `fillToExact` 从本地池补位 → 按原文重排；真的没有第 N 个信息点时如实提示上限
- 补齐池只允许**带真实数字的数据卡**，金句/标签卡不进补位

### 试听同步逻辑（音画同一时钟）

- 播放控制条 **▶** 是唯一播放入口（`playPlayback → compClockStart`），不再有第二个播放器
- 有口播视频时，以 `#compBg.currentTime` 为**唯一时钟**，`seekTo()` 同时驱动卡片切换与进度条
- 时间轴 `buildAudioSegs()`：句子起点按字符比例预测 → 吸附到 ffmpeg `silencedetect` 检测的真实停顿点（`/api/align`，半径 8 秒且必须晚于上一句）→ 强制单调不重叠
- 播放点落在无卡空隙时**保持上一张**，不允许回退到最后一张
- 预览与成片共用同一套时间轴

---

## 6. 启动与验证

### 启动

```powershell
cd d:\trae~ggg\jinrpentu
node server.js          # 或双击 start.bat
# 浏览器打开 http://localhost:8123
```

首次启动 Windows 会弹防火墙提示，**必须勾"允许访问"**，否则局域网/手机访问连不上；漏点就管理员运行 `allow_firewall_8123.bat`。

### 环境要求（硬性）

| 项 | 要求 | 不满足的后果 |
|---|---|---|
| Node | **≥ 18**（用了全局 `fetch`） | 启动即报错 |
| 浏览器 | **新版 Chrome / Edge**，需要 `VideoEncoder`（WebCodecs） | 导出视频/合成会提示"当前浏览器不支持视频编码" |
| 浏览器降级 | 无 WebCodecs 时仅透明视频降级到 `MediaRecorder`，**`/api/composite` 合成路径无降级** | 合成不可用 |
| macOS | 首次需 `chmod +x start.command` | 双击无反应 |
| 磁盘 | 导出目录可写；临时目录需容纳口播原片 + 叠加素材 | 合成失败 |

### 交接验收清单（逐条实测）

| # | 检查项 | 通过标准 |
|---|---|---|
| 1 | 页面加载 | `http://localhost:8123/` 返回 200，无白屏 |
| 2 | 卡片生成 | 粘贴文案 → 生成，出卡数量符合"张数"设置 |
| 3 | 去重 | 任意两张卡 `text` 不相同、不互相包含 |
| 4 | 卡型多样 | 至少出现 2 种以上卡型（非全 KPI） |
| 5 | 试听 | 点播放条 ▶：有声音（`muted=false`）、画面显示、卡片随时间切换 |
| 6 | 顺序 | 卡片按文案顺序 0→1→2… 前进 |
| 7 | 导出 PNG | 保存成功，文件可打开，落在导出目录（`/api/save` 返回值里的 `path`） |
| 8 | 视频合成 | 输出 MP4 有画面、有 AAC 音轨、时长与口播一致 |
| 9 | 局域网 | `/api/lan` 返回本机 IP；另一设备能打开 `http://<IP>:8123` |
| 10 | 便携包 | 解压干净副本（PATH 无 node）双击 `start.bat` 可启动 |
| 11 | 手机预览同步 | 手机打开 `?preview=1`，电脑端改卡片 → 手机端跟着变（走 `/api/state/stream`） |
| 12 | AI 状态 | `/api/llm/status` 返回 `configured: true`，`model` 是 `glm-4-flash` 而非 `glm-5.3-flash` |

### 排查方法

- 语法检查（单文件内联脚本）：把 `index.html` 里 `<script>` 内容抽出，`node --check`，或用浏览器控制台看报错
- 时间轴核对：控制台执行
  ```js
  buildAudioSegs(cards, document.getElementById('textInput').value, 77.02, 0, null)
  ```
  看 `start` 是否严格单调递增
- AI 是否真的在跑：看提示条 —— `🤖 AI 解析完成` 表示走了模型；`⚠ AI 不可用 … 已用本地规则出 X 张` 表示已降级
- 停顿检测：`POST /api/align`（先 `POST /api/upload` 拿 token）

---

## 7. 已知限制 / 待办

| 问题 | 现状 | 建议方案 |
|---|---|---|
| **字幕与配音精确对齐** | 误差 1–2 秒。当前无 ASR，只能按字数比例 + 停顿吸附 | 接本地 **Whisper**（离线免费、逐词时间戳，误差可到 0.3s）；智谱 ASR 需付费（约 0.0002 元/秒） |
| 免费模型不稳定 | `glm-4-flash` 偶发返回不可解析 JSON、少给卡 | 已有 JSON 修复器 + 本地规则兜底；要稳定需付费模型 |
| HEVC/H.265 视频 | 本机能解，部分 Chrome 无扩展会黑屏无声 | 已提示转 H.264 MP4；可加 ffmpeg 自动转码 |
| 国内访问云端 | `vercel.app` 被阻断 | 绑自有域名（Vercel 自动签 HTTPS） |
| 便携包更新 | 手动压缩 | 可写 `publish.bat` 一键：推 GitHub + 重打 ZIP |
| 无版本控制 | 根目录不是 git 仓库 | 建议 `git init`，`.gitignore` 排除 `llm.config.json`、`runtime/`、`*.zip`、`exports/` |

---

## 8. 交接执行流程

### 第一步：代码与产物

1. 拷贝整个 `d:\trae~ggg\jinrpentu`（已清理，剩余约 70MB，主要是便携包 ZIP）
2. 若只交接源码：`index.html` + `server.js` + `llm-core.js` + `start.bat` 四个文件即可跑起全部功能（AI 需配 Key）
3. 接手方新建 git 仓库时，`.gitignore` 至少排除：
   ```
   llm.config.json
   OverlayStudio-Portable/llm.config.json
   OverlayStudio-Portable/runtime/
   *.zip
   exports/
   deploy-vercel/.vercel/
   ```

### 第二步：账号与权限（按顺序）

1. **智谱**：新建一个 Key 给接手方，或直接转移账号；把 Key 写进对方的 `llm.config.json`
2. **GitHub**：仓库 `DALU99956/doxiaogonztai` → Settings → 转移所有权 / 加协作者；接手方重新生成 Token（`repo` 权限）
3. **Vercel**：项目 `overlay-studio-cloud` → Settings → 邀请成员或转移；确认环境变量 `GLM_API_KEY` 已配置
4. **作废旧凭据**：本对话中出现过的 GitHub Token 与旧 API Key 全部作废重发

### 第三步：环境

1. 装 Node 18+（或用便携包自带 `runtime\node.exe`）
2. 首次运行放行防火墙 8123
3. 确认导出目录：有 `D:\下载` 就写那里，没有则回退 `exports\` 或系统下载目录

### 第四步：验收

按第 6 节 12 条清单逐条实测，特别关注第 5、6、8 条（音画同步、卡片顺序、成片音轨）。

### 第五步：交接结束动作

- 删除本机 `llm.config.json` 中的旧 Key，换成接手方的
- 双方确认：谁是仓库 Owner、谁是 Vercel Owner、谁是智谱账号主体
- 把本文档第 5 节（设计不变量）作为后续改动红线，避免回退到"AI 自己写文案/拆碎片卡"的老问题

---

## 9. 一句话总结给接手方

> 这是一套**"AI 只做语义挑选、格式和数据全部由规则兜住"**的单文件动效卡片工具。改动的第一原则：**不要让 AI 碰数值、文案、视觉和卡型**；所有同步问题都以 `#compBg.currentTime` 为唯一时钟；所有张数问题都以"同句去重 + 原文顺序"为准。
