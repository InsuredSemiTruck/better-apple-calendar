# iCloud 日历网页版

把 iCloud 日历搬进浏览器：在你的 **Windows 或 macOS** 电脑上跑一个本地小服务，用网页以 **日 / 周 / 月** 三种视图查看日历，并可直接 **新建、修改、拖动改期、删除** iCloud 上的日程。所有写操作都通过 CalDAV 真正写回 iCloud，iPhone / iPad / Mac 上的日历会同步变化。

<!-- 修改：补充 python-dotenv 与 .env 凭据说明 -->
- **后端**：Python + Flask + [caldav](https://github.com/python-caldav/caldav) + `python-dotenv`，AI 提取使用 `openai` SDK 兼容 DeepSeek；凭据优先从项目根目录 `.env` 读取（也支持系统环境变量），不硬编码、不进 Git
- **前端**：原生 HTML / CSS / JavaScript 单文件，无框架、无构建步骤
- **数据**：本地 JSON 缓存（`events_cache.json` / `sync_token.json`），已在 `.gitignore` 中忽略

> 适用场景：只想在自己电脑上有一个「像 Apple 日历一样的网页」。项目默认只监听 `127.0.0.1`，不对外提供访问。

<!-- 想放界面截图：把图片放进仓库（例如 docs/ 目录），然后在这里插入 ![月视图](docs/month.png) -->

---

## 目录

- [功能一览](#功能一览)
- [快速开始](#快速开始)
- [环境变量](#环境变量)
- [使用说明](#使用说明)
- [循环日程的行为](#循环日程的行为重要)
- [数据与隐私](#数据与隐私)
- [HTTP 接口](#http-接口)
- [项目结构](#项目结构)
- [常见问题](#常见问题)
- [已知限制](#已知限制)
- [更新记录](#更新记录)
- [开发与自检](#开发与自检)
- [许可与免责声明](#许可与免责声明)

---

## 功能一览

<!-- 新增：功能一览新增「配置向导」「启动自动同步」「AI 提取」 -->
| 分类 | 功能 |
| --- | --- |
| 视图 | 月视图（**上下滚动连着看上/下一个月**：当前月高亮、相邻月变淡、月份标题条跟手滚；周日起始、跨天事件横条、每格「其他 X 个日程…」折叠）、周视图、日视图（00:00–24:00 可滚动时间轴、当前时间线） |
| 日程 | 双击新建 / 双击日程编辑、长按拖动改期（仅月视图）、删除（含二次确认） |
| 循环日程 | 双击只修改「点中的那一次」；删除可选「只删这一次」或「删除整个系列」 |
| 全天事件 | 单独一行显示；超长连续全天事件（超过 5 天）收进下方「超长连续全天日程」面板 |
| 筛选 | 左侧日历列表复选框按日历显示/隐藏；日历有内置配色（个人=绿、家庭=黄、课程=红、课外=橙、计划/作业=紫，其他日历自动分配稳定颜色） |
| 设置 | 齿轮面板：上次同步时间 + 缓存条数、「立即同步」按钮、深色模式开关（记忆在浏览器本地） |
| 配置向导 | 首次缺少凭据时网页自动弹出「初始化配置」；也可访问 `/?setup=1` 重新配置。设置面板（⚙︎）新增「修改账号与配置」。填写 Apple ID、App 专用密码、可选 DeepSeek Key，保存后写入 `.env` 并自动重载；可随时关闭 |
| AI 提取 | 工具栏 ✨ 粘贴长文本 → DeepSeek 解析成待确认卡片；可逐条修改/勾选，确认后才写入 iCloud |
| 启动自动同步 | 页面首次加载完成后自动静默调用一次 `POST /api/refresh`，成功后直接更新视图，不弹 Toast |
| 同步 | 本地缓存 + 增量刷新；右上角 ↻ 可手动刷新；缓存过期后读取时会自动后台刷新 |

---

## 快速开始

### 0. 前置条件

- **Windows 10/11** 或 **macOS 12+**（Linux 理论上可用，未做验证）
- Python **3.10+**
  - Windows：安装时勾选「Add python.exe to PATH」，用 `py -3 --version` 确认
  - macOS：系统自带的 `python3` 可能过旧，先跑 `python3 --version`；低于 3.10 时用 `brew install python@3.12` 或从 <https://www.python.org/downloads/macos/> 安装。没装过命令行工具会提示 `xcode-select: error`，先执行 `xcode-select --install`
- 一个开启了两步验证的 Apple ID
- 如果使用代理软件，请确保它不会中断到 iCloud 的 CalDAV 连接（macOS 上尤其注意 Clash / Surge 的「系统代理」与「增强模式」，见[常见问题](#常见问题)）

### 1. 生成 App 专用密码

> 不要把 Apple ID 主密码填进本项目，必须用「App 专用密码」。

1. 浏览器打开 <https://appleid.apple.com/>，用你的 Apple ID 登录
2. 进入「登录与安全」→「App 专用密码」→ 新建一个（名字随意，如 `calendar-web`）
3. 复制生成的 `xxxx-xxxx-xxxx-xxxx` 形状的密码，下一步会用到

### 2. 获取代码并安装依赖

**Windows（PowerShell）**

```powershell
git clone https://github.com/InsuredSemiTruck/better-apple-calendar.git
cd better-apple-calendar

py -3 -m venv .venv
& ".\.venv\Scripts\python.exe" -m pip install --upgrade pip
& ".\.venv\Scripts\python.exe" -m pip install -r requirements.txt
```

**macOS（zsh，默认终端）**

```zsh
git clone https://github.com/InsuredSemiTruck/better-apple-calendar.git
cd better-apple-calendar

python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
```

> 虚拟环境不能跨系统复制：Windows 上建的 `.venv` 在 macOS 里不能用（反之亦然）。在另一台机器上重新执行本节的 `venv` 命令即可。
>
> macOS 激活后终端提示符前会出现 `(.venv)`，此时 `python` / `pip` 都指向虚拟环境；用 `deactivate` 退出。不想激活也行，把命令里的 `python` 换成 `./.venv/bin/python` 即可。

<!-- 修改：快速开始第 3 步改为「两种配置方式」，推荐直接启动后在网页向导中填写 -->
### 3. 配置凭据（两种方式，任选其一）

**方式一（推荐，零配置文件）**

不需要提前设置环境变量。直接按第 4 步启动服务，然后打开网页：

- 如果还没有任何凭据，页面会自动弹出「初始化配置」；
- 如果本地已有缓存但缺少凭据，页面会先显示缓存，点右上角 `↻` 或访问 <http://127.0.0.1:5000/?setup=1> 打开向导。
- 填写 Apple ID、App 专用密码（可选 DeepSeek API Key），点「保存并启动」。后端会写入项目根目录 `.env`，页面自动刷新并开始同步。
- 之后如需修改，点顶部 ⚙︎ → 「修改账号与配置」；输入框留空表示保留原值（更换 Apple ID 时必须填写新的 App 专用密码）。
- 如需使用「AI 提取」，请在配置窗口填写 DeepSeek API Key。

**方式二（进阶，手动编辑 `.env`）**

复制模板并填写。

Windows（PowerShell）：

```powershell
Copy-Item .\env.example .\.env
notepad .\.env
```

macOS（zsh）：

```zsh
cp env.example .env
open -e .env
```

`.env` 内容示例：

```dotenv
ICLOUD_APPLE_ID=你的AppleID@example.com
ICLOUD_APP_PASSWORD=abcd-efgh-ijkl-mnop
DEEPSEEK_API_KEY=
```

保存后按第 4 步启动。`.env` 已被 `.gitignore` 忽略，不会提交到 Git。

> 仍支持系统环境变量（`$env:...` / `export ...`）；但本项目使用 `load_dotenv(..., override=True)`，**同名变量以 `.env` 为准**。

### 4. 启动

Windows（PowerShell）：

```powershell
& ".\.venv\Scripts\python.exe" ".\app.py"
```

macOS（zsh）：

```zsh
source .venv/bin/activate
python app.py
```

（不激活也可以：`./.venv/bin/python app.py`）

看到类似 `Running on http://127.0.0.1:5000` 就说明起来了。

### 5. 打开网页

浏览器访问 <http://127.0.0.1:5000>，首次加载会从 iCloud 拉取**今天前后各 30 天**的日程。首次加载需要从 iCloud 拉取约 60 天日程，会慢一些；之后有本地缓存，加载会快很多。

<!-- 修改：说明首次加载后的静默增量刷新 -->
页面渲染完成后，前端还会在后台静默调用一次 `POST /api/refresh` 做增量同步；成功后直接更新当前视图，不会弹 Toast。

> 起不来或读不到日历时，先单独跑一次连通性探测脚本，它会直接打印你的日历列表和未来 7 天的日程，能立刻区分是「凭据问题」还是「网页/服务问题」：
>
> Windows（PowerShell）：
>
> ```powershell
> & ".\.venv\Scripts\python.exe" ".\icloud_calendar_probe.py"
> ```
>
> macOS（zsh）：
>
> ```zsh
> source .venv/bin/activate
> python icloud_calendar_probe.py
> ```

---

## 环境变量

<!-- 新增：说明 .env 与 env.example -->
只有前两个是**必填**，其余都有默认值，按需覆盖即可。所有变量都可以写在项目根目录 `.env` 中（模板见 `env.example`），也可以继续用系统环境变量；`app.py` 使用 `load_dotenv(CONFIG_ENV_FILE, override=True)`，因此 `.env` 中的同名变量优先。

<!-- 新增：环境变量表新增 DEEPSEEK_API_KEY -->

| 变量 | 必填 | 默认值 | 说明 |
| --- | :-: | --- | --- |
| `ICLOUD_APPLE_ID` | ✅ | — | 你的 Apple ID（邮箱） |
| `ICLOUD_APP_PASSWORD` | ✅ | — | App 专用密码（`xxxx-xxxx-xxxx-xxxx`），**不是** Apple ID 主密码 |
| `DEEPSEEK_API_KEY` | ❌ | — | AI 提取使用；未配置时 `/api/ai_parse` 返回 400，日历读写不依赖它 |
| `ICLOUD_CALDAV_URL` | ❌ | `https://caldav.icloud.com/` | CalDAV 入口；iCloud 中国区账号可尝试 `https://caldav.icloud.com.cn/` |
| `ICLOUD_TIMEZONE` | ❌ | `Asia/Shanghai` | 判定「今天」、解析无时区时间、全天事件归属的时区；时间显示不对就改这里（如 `Asia/Tokyo`） |
| `CALENDAR_CACHE_TTL_SECONDS` | ❌ | `600`（最小 30） | 缓存多久后视为过期；过期后读取会在后台自动增量刷新 |
| `CALENDAR_CLIENT_TTL_SECONDS` | ❌ | `300`（最小 30） | CalDAV 客户端与日历列表复用时长（避免每次写入都重新做服务发现） |
| `CALENDAR_HTTP_TIMEOUT_SECONDS` | ❌ | `30`（最小 5） | 单次 HTTP 请求超时 |
| `PORT` | ❌ | `5000` | 本地网页端口（服务只绑定 `127.0.0.1`） |
| `FLASK_DEBUG` | ❌ | `0` | 设为 `1` 打开 Flask 调试模式（仅排错时用） |
| `CALENDAR_DEBUG_EVENTS` | ❌ | `0` | 设为 `1` 会把解析到的每条事件打进日志（排查「少了日程 / 时间不对」） |
| `ALLOWED_ORIGIN` | ❌ | 空 | 仅当把前端页面部署在**其他来源**时才需要：填前端地址（如 `http://localhost:8080`），后端才会返回 CORS 头，并把 `index.html` 里的 `meta[name="calendar-api-base"]` 指到后端地址 |

---

## 使用说明

<!-- 新增：使用说明新增「配置 / 重新配置凭据」「AI 提取」 -->
| 想做什么 | 怎么做 |
| --- | --- |
| 配置 / 重新配置凭据 | 首次缺少配置会自动弹出向导；也可点顶部 ⚙︎ → 「修改账号与配置」，或访问 `/?setup=1`。输入框始终空白，已配置项显示「已配置（如需修改请重新输入）」；留空字段保留原值；右上角 ×、点击遮罩或按 Esc 可关闭，不保存修改 |
| AI 提取日程 | 顶部 ✨ → 粘贴文本 → 智能提取 → 在待确认卡片中修改/勾选 → 确认添加选中日程 |
| 取消 AI 提取 | 输入框/确认弹窗右上角 ×、点击遮罩或 Esc 可关闭；确认弹窗写入中不可关闭 |
| 切换视图 | 顶部中间的「日 / 周 / 月」分段按钮 |
| 前后翻页 | 顶部 `‹` / `›`（月视图翻月、周视图翻周、日视图翻天） |
| 浏览相邻月份 | 月视图里直接**上下滚动**就能连着看上/下一个月（月份标题条跟着滚、当前月自动高亮、相邻月变淡）；也可以用顶部 `‹`/`›` 翻月，「今天」回到本月 |
| 回到今天 | 顶部「今天」 |
| 手动刷新 | 顶部 ↻（会立即同步一次并把最新结果显示出来） |
| 新建日程 | 顶部蓝色 `+`；或双击周/日视图的空白时间格（会预填该时间） |
| 编辑日程 | **双击**任意日程（月视图格子里的日程、跨天横条、周/日视图的时间块，以及「其他 X 个日程…」弹层里的条目都可以） |
| 查看某天全部日程 | 月视图里点格子内的「其他 X 个日程…」，弹出当天日程列表 |
| 拖动改期 | 在月视图里**长按约 0.5 秒**再拖到目标日期（松手后立即生效，失败会自动回到原位并提示） |
| 调整时长 | 周/日视图里**单击日程方块**让它高亮（也可长按直接进入拖动），再按住**上半部分**上下拖改开始时间、按住**下半部分**改结束时间；拖动时方块会实时变高变矮并显示「开始/结束 13:30 – 15:00」读数，松手才写回，失败会回到原时长并提示 |
| 删除日程 | 双击日程打开面板 → 操作行里的「删除日程」→ 按提示二次确认（详见下一节） |
| 筛选日历 | 左侧日历列表前的复选框，勾选/取消即显示或隐藏该日历 |
| 选择默认日历 | 点击左侧日历名称高亮；新建日程时表单默认选中它 |
| 深色模式 | 顶部 ⚙︎ → 「深色模式」（设置保存在浏览器 localStorage） |
| 查看同步状态 | 顶部 ⚙︎ → 显示上次同步时间与缓存条数 |

> 小提示：日程标题样式与颜色跟随所属日历；月视图里 22px 高的日程条与跨天横条只显示一行，超出部分会折叠为「其他 X 个日程…」。

---

## 循环日程的行为（重要）

循环（重复）日程在 iCloud 上是**一条资源**（master 带 `RRULE`，被单独改动的某次以 `RECURRENCE-ID` 覆盖）。本项目对它的处理规则：

| 操作 | 结果 |
| --- | --- |
| 双击编辑某一次 | 只改这一次（写入该次的 `RECURRENCE-ID` 覆盖，master 的重复规则不动） |
| 双击删除 →「只删这一次」 | 该次会从系列里排除（在重复规则上追加 `EXDATE`）；若这一次之前被拖动过，还会移除它对应的覆盖，因此原来那个日期也不再出现 |
| 双击删除 →「删除整个系列」 | 删除整个重复日程（连同所有单独改动的次次） |
| 拖动某一次到别的日期 | 本质就是「只改这一次」的位置，其余次次照旧 |
| 拖动某一次改时长 | 同样是「只改这一次」：只写该次的 `RECURRENCE-ID` 覆盖（`DTSTART`/`DTEND`），master 与其余次次不动 |
| 面板上的提示行 | 若同一日程在当前加载范围里出现多次，编辑面板会提示「循环日程：本次只修改 YYYY/M/D 这一次」 |

会返回明确中文提示、**不支持**的循环日程操作：

- 「此次及以后」（需要切分重复规则，暂未实现）
- 把循环日程换到别的日历
- 把循环日程在「定时」与「全天」之间切换

> 另：如果某个重复日程在当前加载范围内只出现了一次，删除面板只会给出「确认删除」（等于删除整个资源）——这是前端判断方式的局限，后端始终以 iCloud 上的真实重复规则为准。

---

## 数据与隐私

<!-- 修改：凭据现在优先来自 .env，并说明本地明文与权限 -->
- **凭据**：Apple ID、App 专用密码（以及可选的 DeepSeek Key）优先从项目根目录 `.env` 读取（`python-dotenv`，`override=True`）；也可用系统环境变量。网页「初始化配置」通过 `POST /api/config` 写入 `.env`，不会硬编码、不会打印到日志、不会写进缓存或 Git。`.env` 是本机明文文件（POSIX 下会尽量设为 `0600`），已被 `.gitignore` 忽略；只提交 `env.example` 模板。前端只能通过 `GET /api/config` 看到“是否已配置”的布尔值，不会收到已保存的密码/Key；配置模态框的输入框始终为空白。
- **本地缓存**：运行时会生成两个文件 —— `events_cache.json`（事件缓存）与 `sync_token.json`（增量同步令牌）。里面是你的日程标题、时间、地点、UID 与资源 URL，**不含密码**；两者都已写进 `.gitignore`，不会进 Git。想强制全量重新同步，直接删掉这两个文件再启动即可。
<!-- 新增：说明更换 Apple ID 的缓存处理 -->
- **更换 Apple ID**：保存新配置时，后端会在缓存中写入账号指纹/失效标记，不删除 `events_cache.json`，但会忽略上一个账号的残留缓存，并在下一次同步时用新账号数据替换。
- **网络流向**：浏览器 ↔ 本机 `127.0.0.1:5000`（只在本机内），后端 ↔ iCloud CalDAV（HTTPS 直连）；没有第三方中转服务器。仅当你主动使用「AI 提取」时，粘贴的文本会直接发送给 DeepSeek（`https://api.deepseek.com`）做解析。
- **AI 提取**：AI 只返回待确认草稿，不会自动写入；只有你点击「确认添加选中日程」后才会逐条调用 `POST /api/events`。DeepSeek API Key 只在后端使用，前端只能看到配置状态布尔值；用户文本与 AI 原文不会写入日志或缓存。
- **写操作是真的写**：新建、修改、拖动、删除都会写回 iCloud，手机与电脑上的日历会同步变化；**删除不可撤销**，建议先拿一条测试日程熟悉流程。
- **不要暴露到公网**：服务只绑定 `127.0.0.1`，且没有登录鉴权。若确实要远程访问，请自行加反向代理 + HTTPS + 访问控制，并注意本服务能读写你的全部日历。

---

## HTTP 接口

后端是一个很薄的 JSON 层，想二次开发（或者自己写个前端）可以直接用：

<!-- 新增：HTTP 接口新增 POST /api/config 与 POST /api/ai_parse -->
| 方法 | 路径 | 说明 |
| --- | --- | --- |
| `GET` | `/` | 返回前端页面 `index.html` |
| `GET` | `/api/events?start=YYYY-MM-DD&end=YYYY-MM-DD` | 读取事件。`start`/`end` 必须同时提供，单次范围不超过 366 天；不传则默认今天前后各 30 天。返回 `{"events":[...]}` |
| `POST` | `/api/events` | 新建日程。请求体 `{title, calendar, start, end, location?}`；`start`/`end` 用带时区的 ISO 时间，全天日程用 `YYYY-MM-DD`。成功返回 `201 {"event":{...}}` |
| `PUT` | `/api/events/<uid>` | 修改日程，请求体 `{title, calendar, start, end, location?, recurrenceId?}`。`calendar` 与当前不同＝把日程移动到该日历；循环日程必须带 `recurrenceId`（被修改那次的开始时间），只改这一次 |
| `DELETE` | `/api/events/<uid>` | 删除日程，请求体 `{scope, recurrenceId?}`。`scope="series"`（默认）＝删除整个日程/整个重复系列；`scope="instance"`＝只删循环日程的某一次（需带 `recurrenceId`）；非循环日程一律删除整条 |
| `GET` | `/api/config` | 只返回是否已配置的布尔值：`{configured:{appleId,appPassword,deepseekApiKey}, missing:[...]}`，绝不下发凭据 |
| `POST` | `/api/config` | 保存凭据到 `.env` 并立即更新当前进程环境变量。请求体 `{appleId?, appPassword?, deepseekApiKey?}`；留空字段保留原值；更换 Apple ID 时必须提供新的 App 专用密码；成功返回 `{"ok": true}` |
| `POST` | `/api/ai_parse` | AI 解析文本为待确认日程草稿，不写入日历。请求体 `{text, current_date}`；返回 `{events:[{title,start,end,location}]}`；缺少 DeepSeek Key 返回 400，格式/接口错误返回 502 |
| `POST` | `/api/refresh` | 立即同步一次并返回最新缓存：`{events, updated_at, range:{start,end}}` |
| `GET` | `/api/status` | 只读的同步状态：`{updated_at, events, calendars, age_seconds, cache_ttl_seconds}` |

<!-- 新增：说明缺少配置时的返回与缓存优先 -->
缺少 `ICLOUD_APPLE_ID` / `ICLOUD_APP_PASSWORD` 时：若当前范围有缓存，`GET /api/events` 仍返回缓存；无可用缓存则返回 `503 {"error":"缺少配置：..."}`，前端据此弹出配置向导。`POST /api/refresh` 在缺少配置时返回 503。

**事件字段**：`calendar`（日历名）、`title`、`start`、`end`、`location`（可能为空串）、`uid`。全天日程的 `start`/`end` 是纯日期（`YYYY-MM-DD`，`end` 为不含当天的那一天），定时日程是带时区偏移的 ISO 8601 字符串。

**常见状态码**：

| 状态码 | 含义 |
| --- | --- |
| `400` | 参数缺失或该操作暂不支持（返回中文 `error`，例如「循环日程需要指定要修改的实例」） |
| `404` | 缓存和 iCloud 上都找不到这条日程（可能已在其他设备删除） |
| `409` | iCloud 以 412 拒绝了写入（内容已被其他设备改动）——刷新后重试；反复出现时，查看后端终端日志获取详细原因 |
| `502` / `503` | 访问 iCloud 失败（网络中断、限流、iCloud 报错）。瞬时故障会自动重试最多 3 次（0.8s / 1.6s 退避），仍失败才返回 503；反复出现时，查看后端终端日志获取详细原因 |

---

## 项目结构

<!-- 新增：项目结构新增 env.example 与 .env -->
| 文件 | 作用 |
| --- | --- |
| `app.py` | Flask 服务：缓存与增量同步、读取、新建、修改（含循环日程只改此次）、删除（含只删此次）、CalDAV 定位与瞬时故障重试 |
| `icloud_calendar_probe.py` | 独立探测脚本：列出你的日历 + 打印未来 7 天日程，用来快速验证凭据与网络 |
| `index.html` | 单文件前端：月/周/日视图、拖动改期、编辑与删除面板、设置面板与配置入口、AI 提取与待确认卡片（原生 HTML/CSS/JS，无框架无构建） |
| `requirements.txt` | 依赖：`Flask>=3.0,<4`、`caldav>=3.0`、`python-dotenv>=1.0,<2`、`openai>=1.0,<2`（本项目实测于 caldav 3.6.0） |
| `env.example` | `.env` 模板：Apple ID、App 专用密码、可选 DeepSeek Key |
| `.env` | 配置向导或手动生成，已被 `.gitignore` 忽略，不提交 |
| `LICENSE` | MIT 许可证全文 |
| `AGENTS.md` | 本项目的开发约定、接口契约与功能边界（改代码前建议先读） |
| `STYLE.md` / `window.png` / `iPhone.png` | 界面风格记录与参考图（本地保留，未随仓库发布） |
| `events_cache.json` / `sync_token.json` | 运行时生成的本地缓存（已忽略提交，可随时删除） |

---

## 常见问题

**页面提示「无法读取日历」**
<!-- 修改：补充 .env / 配置向导 -->
先确认 `.env` 或系统环境变量中有凭据（也可以访问 `/?setup=1` 重新填写），然后单独跑一次探测脚本：

```powershell
# Windows（PowerShell）
& ".\.venv\Scripts\python.exe" ".\icloud_calendar_probe.py"
```

```zsh
# macOS（zsh）
source .venv/bin/activate
python icloud_calendar_probe.py
```

- 探测脚本报「CalDAV 连接或认证失败」→ 多半是 App 专用密码不对（注意：**不是** Apple ID 主密码），或 Apple ID 需要重新登录
- 探测脚本正常、只有网页打不开 → 看 `app.py` 所在终端的日志
- macOS 上 `command not found: python` → 忘了 `source .venv/bin/activate`，或改用 `./.venv/bin/python icloud_calendar_probe.py`

**提示「连接 iCloud 失败，请稍后重试」**
后端已自动重试 3 次仍失败才会这样提示。常见原因是代理软件（Clash / VPN / 抓包工具）中断了到 iCloud 的 TLS 长连接，或网络临时抖动：关掉代理、换网络后再点 ↻ 刷新。

**提示 iCloud 连接超时**
把 `CALENDAR_HTTP_TIMEOUT_SECONDS` 调到 `60` 再试；如果还是不行，检查代理软件或 IPv6 设置。
Windows：`$env:CALENDAR_HTTP_TIMEOUT_SECONDS = "60"`；macOS：`export CALENDAR_HTTP_TIMEOUT_SECONDS=60`，然后重启服务。

**macOS：一直卡在读取，日志里是 TLS 相关报错（Clash / Surge 等代理软件干扰）**
这是 macOS 上最常见的一类问题，典型日志是 `SSL: UNEXPECTED_EOF_WHILE_READING`、`TLSV1_ALERT_…`、`Connection reset by peer`，或请求长时间无响应（后端重试 3 次后返回 503）。原因是 ClashX / Clash Verge / Surge / Quantumult 等工具默认开启「系统代理」，甚至开启「增强模式 / TUN」，把 `caldav.icloud.com` 的请求塞进代理隧道，而 iCloud 的 CalDAV 长连接对隧道并不友好。按这个顺序排查：

1. **彻底退出**代理软件（菜单栏图标 → Quit，而不是只把开关关掉），再点 ↻ 刷新；
2. 必须常开代理时，给它加分流规则：把 `caldav.icloud.com`、`*.icloud.com`、`*.apple.com` 设为 **DIRECT（直连）**，或把这些域名加入「绕过代理」列表；
3. 确认终端里的代理变量没有残留：`env | grep -i proxy` 若有输出，先 `unset http_proxy https_proxy all_proxy`（GUI 启动的终端可能继承了系统代理设置）；
4. 用 curl 单独确认 TLS 通不通（返回 `401` / `207` 都说明网络正常，证书有问题才会报错）：
   ```zsh
   curl -sS -o /dev/null -w "%{http_code}\n" https://caldav.icloud.com/
   ```
5. 仍不行就把超时调大：`export CALENDAR_HTTP_TIMEOUT_SECONDS=60`，或换一个网络（比如手机热点）验证是否是本机网络环境问题。

**macOS：Finder 里看不到 `.venv`、`.gitignore`、`.env`？**
macOS 默认隐藏以 `.` 开头的文件/文件夹（Windows 上叫「隐藏项目」）。在 Finder 窗口里按 **`⌘ + ⇧ + .`**（Command + Shift + 句号）即可显示/隐藏隐藏文件，再按一次恢复。终端里用 `ls -a` 也能看到。想直接进项目目录最省事的办法是在终端执行 `open .`（当前目录）或用 `open -a Finder .`。

**macOS：`python3 -m venv .venv` 报错 / 装的包找不到**
- 提示 `xcode-select: error: invalid active developer path` → 先 `xcode-select --install` 装命令行工具
- `python3 --version` 低于 3.10 → 用 `brew install python@3.12`（装完是 `python3.12`，建环境用 `python3.12 -m venv .venv`）
- 提示 `No module named flask` → 说明当前 `python` 不是虚拟环境里的那个，确认终端提示符带 `(.venv)`，或用 `./.venv/bin/python -c "import flask; print(flask.__version__)"` 检查
- Apple Silicon 上用 Rosetta 终端（`arch` 显示 `i386`）可能装出架构不匹配的包，改用原生 arm64 终端重装即可

**某些日程没有显示 / 看着是旧的**
默认只查询**今天前后各 30 天**，翻页离开已加载范围时会自动请求新范围。若刚在手机上改过，点顶部 ↻ 或设置面板里的「立即同步」即可拉最新内容（缓存超过 10 分钟视为过期，读取时会自动后台刷新）。

**编辑 / 拖动 / 删除失败**
- 提示 `409`：iCloud 认为内容已被其他设备改动（412 冲突）——点 ↻ 刷新后再试
- 提示 `404`：这条日程在 iCloud 上已不存在（可能刚在手机上删了）
- 提示 `503`：网络问题（同上）

**时间整体差了几个小时 / 全天日程跑到前一天**
设置 `ICLOUD_TIMEZONE`（例如 `Asia/Tokyo`）后重启服务。

**删除能撤销吗？**
不能。删除会真实写回 iCloud，iPhone 上也会同步消失。建议先新建一条测试日程，把「编辑 / 拖动 / 只删这一次 / 删除整个系列」都试一遍。

**只删了一次，为什么原来那个日期也不显示了？**
「只删这一次」会让该次从系列里排除（在重复规则上追加 `EXDATE`）；若这一次之前被拖动过，还会移除它对应的覆盖，因此原来那个日期也不再出现——这是预期行为，详见[循环日程的行为](#循环日程的行为重要)。

**5000 端口被占用**
设置环境变量 `PORT`（如 `5050`）再启动，然后访问 <http://127.0.0.1:5050>。

- Windows：`$env:PORT = "5050"`，或先 `netstat -ano | findstr :5000` 找到占用进程
- macOS：`export PORT=5050`，或先 `lsof -nP -iTCP:5000 -sTCP:LISTEN` 看是谁占着（`kill <PID>` 结束它）。macOS 上 `5000` 常被「隔空播放接收器 / AirPlay Receiver」占用，可在「系统设置 → 通用 → 隔空播放与接力」里关掉它

**想换成别的 CalDAV 服务 / 中国区 iCloud**
设置 `ICLOUD_CALDAV_URL`（例如 `https://caldav.icloud.com.cn/`）后重启。

<!-- 新增：配置向导 / .env / 账号切换 FAQ -->
**首次打开没有弹出「初始化配置」？**
说明当前范围已有本地缓存，页面会优先显示缓存。点右上角 `↻`（缺少配置时会打开向导），或访问 <http://127.0.0.1:5000/?setup=1>。

**`.env` 在哪里？填错了怎么改？**
在项目根目录（与 `app.py` 同级）。可以直接编辑，也可以访问 `/?setup=1` 重新填写；保存后页面会自动刷新。

**换了 Apple ID，为什么旧数据不见了 / 还在？**
保存新配置时后端会写入账号指纹与失效标记，不会删除 `events_cache.json`，但旧账号缓存会被忽略，下一次同步用新账号数据替换。若仍看到旧数据，点 ↻ 触发同步。

**配置保存成功但读不到日历？**
检查终端是否还在用旧进程；重启 `python app.py`。若终端里导出过旧的 `ICLOUD_APPLE_ID`，`.env` 现在会覆盖它（`override=True`）。

**如何修改已保存的账号 / 密码？**
顶部 ⚙︎ → 「修改账号与配置」。输入框默认空白，已配置项会显示「已配置（如需修改请重新输入）」；只填要修改的字段即可，留空字段保留原值。更换 Apple ID 时必须同时填写新的 App 专用密码。

<!-- 新增：AI 提取 FAQ -->
**点击「智能提取」提示缺少 DeepSeek API Key？**
到顶部 ⚙︎ → 「修改账号与配置」，填写 DeepSeek API Key 后保存。AI 提取不依赖 Apple ID 密码，但日历写入仍需要 iCloud 凭据。

**AI 提示「AI 返回格式错误」或「AI 解析失败」？**
先换一段更明确的文本重试；仍失败可能是 DeepSeek 限流、网络或 Key 无效。后端会返回 502，不会写入任何日程。

**AI 解析后会自动创建日程吗？**
不会。解析结果只是待确认卡片；只有勾选并点击「确认添加选中日程」后才会逐条调用 `POST /api/events`。

**AI 提取的文本会保存吗？**
不会写入缓存或日志；仅在点击「智能提取」时发送给 DeepSeek 做一次性解析。

---

## 已知限制

- 循环日程不支持「此次及以后」（需要切分重复规则）；不支持更换日历；不支持在「定时」与「全天」之间切换（都会返回 400 与中文提示）
- 循环日程的「只删这一次 / 删除整个系列」由前端根据「同一日程在当前范围出现了几次」给出选项；范围里只出现一次时按「删除整个日程」处理，后端始终以 iCloud 上的真实规则为准
- 拖动改期只在月视图提供；调整时长只在周/日视图提供，且只对**单日定时日程**生效（跨天定时日程与「全天」横条不参与）；超长连续全天日程（超过 5 天）不参与拖动
- 月视图一次只渲染「当前月 ±1 个月」的连续周列表：滚到窗口两端要靠点日期或 `‹`/`›` 继续；窗口末尾可能出现只有 1 周的「下个月 peek」块，它滚不到顶部，因此不会成为高亮的当前月
- 月视图长按拖动改期没有边缘自动滚动，跨月拖动请先把目标日期滚进视野
- 调整时长按 15 分钟吸附、最短 15 分钟，最晚会夹在 24:00（不会跨到第二天），开始时间最晚为「结束时间 − 15 分钟」
- 左侧面板日历标题旁的 `+` 是装饰按钮，暂不支持在网页里新建日历（请在 iPhone 或 <https://www.icloud.com/calendar/> 创建）
- 移动端已做阶段一适配（宽度 ≤ 820px：默认进入日视图、顶部横向可滚动筛选条、贴底操作栏、悬浮 `+` 按钮、弹层改底部抽屉）；桌面（> 820px）保持原有完整布局
- AI 提取依赖 DeepSeek API；解析结果可能不准确，必须在待确认卡片中核对后再添加。
- 服务只监听 `127.0.0.1` 且没有登录鉴权，请勿直接暴露到公网
- 已实现的功能边界与设计取舍记录在 `AGENTS.md`，改代码前建议先读

---

## 更新记录

<!-- 新增：v1.3 更新记录 -->
| 版本 | 内容 |
| --- | --- |
| v1.3 | **AI 提取**：工具栏 ✨ 粘贴长文本 → DeepSeek 解析为待确认草稿；卡片可编辑/勾选，确认后才逐条写入；新增 `POST /api/ai_parse`；绝不自动写入 |
| v1.2 | **设置面板配置入口**：⚙︎ 设置面板新增「修改账号与配置」；配置模态框支持初始化/修改双模式，输入框始终空白，留空字段保留原值；新增 `GET /api/config`（只返回是否已配置的布尔值）；更换 Apple ID 时必须填写新的 App 专用密码；配置窗口支持右上角 × / 遮罩 / Esc 关闭 |
| v1.1 | **配置体验**：新增 `.env` 读取（`python-dotenv`）与网页「初始化配置」向导；支持 `POST /api/config` 写入凭据；页面首次加载完成后自动静默增量刷新；更换 Apple ID 时用账号指纹/失效标记避免旧账号缓存残留 |
| v1.0 | **日历功能已实现**：日 / 周 / 月视图读取、新建、编辑（含循环日程「只改这一次」）、拖动改期、拖动改时长、删除（含「只删这一次 / 删除整个系列」），写操作真实写回 iCloud；**前端界面重做**（纸感配色、衬线标题、月视图连续滚动 + 粘性月份标题条）；**新增手机界面**（≤820px 日 / 月双视图、衬线星期 + 星期圆片条、4 格底部操作栏） |
| v0.12 | 月视图**连续滚动**：一次渲染「当前月 ±1 个月」的连续周列表 + 粘性月份标题条，上下滚动时当前月自动高亮、相邻月变淡、标题条跟手滚；翻页 / 切视图 / 筛选后落到当前月顶部，改数据后滚动位置不再跳月 |
| v0.11 | 周/日视图拖动改时长：单击（或长按）日程方块高亮后，拖上半部分改开始时间、拖下半部分改结束时间，实时读数、松手写回；修改（含月视图拖动改期、编辑保存、删除）后**不再把视图滚回顶部** |
| v0.10 | 移动端适配（阶段一）：≤820px 默认日视图、顶部横向筛选条、贴底操作栏、悬浮 `+` 按钮、弹层改底部抽屉 |
| v0.9 | 日程删除：双击日程的面板里新增「删除日程」，循环日程可选「只删这一次」（`EXDATE`）或「删除整个系列」，均需二次确认 |
| v0.8 | 循环日程「只改这一次」：编辑与拖动都会只修改点中的那一次实例（`RECURRENCE-ID` 覆盖），并修复了编辑后重复日程其它实例从缓存消失的问题 |
| v0.7 | 月视图分层与「其他 X 个日程…」折叠、设置面板改为同步状态 + 深色模式、写入稳定性修复与网络瞬时故障自动重试 |

更早版本（初始探测脚本、网页版雏形、定时日程新增与修改）见 git 提交历史。

---

## 开发与自检

```powershell
# Windows（PowerShell）：后端语法自检
& ".\.venv\Scripts\python.exe" -m py_compile ".\app.py"

# 连通性自检（列出日历 + 未来 7 天日程）
& ".\.venv\Scripts\python.exe" ".\icloud_calendar_probe.py"

# 启动服务后查看缓存规模与同步时间
# 浏览器或 curl 打开 http://127.0.0.1:5000/api/status
```

```zsh
# macOS（zsh）：后端语法自检
source .venv/bin/activate
python -m py_compile app.py

# 连通性自检（列出日历 + 未来 7 天日程）
python icloud_calendar_probe.py

# 启动服务后查看缓存规模与同步时间
curl -s http://127.0.0.1:5000/api/status | python -m json.tool
```

前端是单文件、无构建：改完 `index.html` 直接刷新浏览器即可（想校验脚本语法可把内联 `<script>` 抽出来跑一次 `node --check`；macOS 需先装 Node，例如 `brew install node`）。

代码里所有文件路径都用 `Path(__file__).resolve().parent` / `app.root_path` 拼出来，没有硬编码盘符或 `/Users/...`，因此同一份代码在 Windows 与 macOS 上都能直接跑——写新脚本时请保持这一点。

改代码前请先读 `AGENTS.md`：其中写明了 API 契约、缓存规则、循环日程的写入方式，以及「写回 iCalendar 必须用 `component.add()`」这类容易踩坑的约束。

---

## 许可与免责声明

- 本项目以 **MIT 许可证**开源（全文见 [LICENSE](LICENSE)）：可自由使用、修改、分发，需保留版权与许可声明；作者不提供任何担保，也不承担由此产生的责任。
- 本项目会真实读写你的 iCloud 日历，且**删除不可撤销**；请在测试日历上先验证，风险自负。
- Apple ID 与 App 专用密码由你自行保管，本项目不会将其发送到任何第三方服务器。
- 本项目与 Apple 无关联；涉及 iCloud 的使用请遵守 Apple 的服务条款。

