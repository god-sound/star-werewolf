# star-werewolf · 星光狼人杀

**一套内核、两种部署、四种终端、多层品牌的开源线下狼人杀活动系统。**

线下狼人杀的组织成本从来不在规则本身，而在流程：谁坐哪个座位、谁该睁眼、票投给了谁、直播画面上的名字对不对、这一局到底发生了什么。star-werewolf 把这些环节收进一套可以自托管的系统，并刻意把「业务判定」「页面呈现」「品牌样式」「部署形态」拆成互不越界的三层——**规则只有一份，终端与品牌都只是它的外衣**。

[![License](https://img.shields.io/badge/license-MIT-22c55e.svg)](./LICENSE)
[![Node](https://img.shields.io/badge/node-%E2%89%A520-339933.svg)](https://nodejs.org/)
[![Storage](https://img.shields.io/badge/storage-MySQL%20%7C%20JSON-4479A1.svg)](#存储与配置)
[![Version](https://img.shields.io/badge/version-v0.1.0-F0B45A.svg)](./package.json)

![首页](./docs/preview/home.png)

## 目录

- [它解决什么问题](#它解决什么问题)
- [核心能力](#核心能力)
- [界面](#界面)
- [架构](#架构)
- [快速开始](#快速开始)
- [跑一局](#跑一局)
- [页面与路由](#页面与路由)
- [局型与角色](#局型与角色)
- [规则要点](#规则要点)
- [权限与安全](#权限与安全)
- [API](#api)
- [存储与配置](#存储与配置)
- [测试与自检](#测试与自检)
- [部署](#部署)
- [目录结构](#目录结构)
- [路线图](#路线图)
- [已知边界](#已知边界)
- [参与贡献](#参与贡献)
- [许可证](#许可证)

## 它解决什么问题

一套完整的线下狼人杀活动，实际需要同时满足四类人：主持人要控流程、玩家要只看到自己的身份、大屏要能直播、后台要能管房间和赛事。绝大多数实现方式是把这四件事写进同一份页面与判断里，于是换一个场地就要改配色，接一个新的终端就要复制一套规则。

star-werewolf 把被现场验证过的部分（规则、状态机、凭据体系、赛事流程）原样保留，只把它们从页面和部署里剥离出来，变成一份共享内核。判断这件事是否做成只有一个标准：**新增一种终端、换一套品牌、切一种部署，都不需要改动内核里的任何一条规则。**

| 维度 | 内容 | 边界 |
|---|---|---|
| 一套内核 | 领域模型、状态机、规则引擎、凭据、赛事、复盘 | 只做判定，不做呈现与品牌 |
| 两种部署 | Edge 现场版（单机）、Cloud 托管版（多租户） | 同一镜像两个档位，代码分支唯一 |
| 四种终端 | 主持、玩家、直播、后台 | 薄客户端，判定全在服务端 |
| 多层品牌 | L0 内核、L1 发行主题、L2 租户、L3 活动、L4 物料 | 只换外衣，不改规则与布局 |

## 核心能力

| 能力 | 说明 |
|---|---|
| 建房与入座 | 4 位房号、逐座邀请链接、空座命名、换座、踢出、指定选手；凭据存本机，换设备即换身份 |
| 规则引擎 | 4 档局型、7 种角色（含白痴变体），发牌、夜间合法性、屠边胜负、平票 PK、猎人开枪全在服务端判定 |
| 夜间流程 | 守卫 → 狼人 → 女巫 → 预言家逐段推进；玩家端只拿得到自己的行动，主持人逐段结算 |
| 发言与投票 | 遗言、按座位逐位计时发言（默认 120 秒）、投票、PK 轮、强制放逐 |
| 直播贴片 | `/stage/:roomId` 透明底只读页面，可直接作为 OBS 浏览器源；身份、票型、阶段、倒计时均可开关 |
| 赛事模式 | 叫号翻牌、选手库、按账号授予赛事权限；房号前缀区分普通 / 赛事 / 测试 |
| 管理后台 | 房间列表（房间码、对局 ID、日期区间、状态等筛选）、贴片预设、选手库、赛事权限、飞书、管理员 |
| 飞书集成 | 自建应用鉴权校验、选手绑定 `open_id` / `user_id`、座位通知发送结果回写 |
| 复盘 | 底牌、夜晚明细、票型、按天可折叠时间线；非参与方访问 403 |

## 界面

| 首页（主持 / 玩家双入口） | 建房（局型与测试房） |
|---|---|
| ![首页](./docs/preview/home.png) | ![建房](./docs/preview/create.png) |

| 直播贴片（OBS 浏览器源） | 后台（房间列表） |
|---|---|
| ![直播贴片](./docs/preview/stage.png) | ![后台](./docs/preview/admin.png) |

界面按 480px 竖屏优先设计（篝火金 + 星光紫、半透明紫卡），不是逐像素复刻某一版线上应用。

## 架构

```
                         ┌─────────────────────────────┐
   四种终端               │  Host 主持  Player 玩家     │
   （薄客户端）           │  Stage 直播  Admin 后台     │
                         └──────────────┬──────────────┘
                                        │ 同一套命令 / 查询接口
                         ┌──────────────▼──────────────┐
   一套内核               │  领域模型 · 状态机 · 规则引擎 │
                         │  座位凭据 · 赛事 · 复盘      │
                         │  SPI：身份 / 通知 / 存储 / 主题│
                         └──────┬────────────────┬──────┘
                                │                │
   两种部署          Edge 现场版（单机、局域网）  Cloud 托管版（多租户、公网）
```

| 层 | 位置 | 职责 |
|---|---|---|
| 内核 | `packages/core` | 局型、角色、阶段、规则、房间状态机、视图裁剪、凭据与权限、存储驱动 |
| 终端 | `packages/terminals` | 十个页面与共享 `app.js` / `app.css`，只渲染视图、只提交意图 |
| 主题 | `packages/themes/night` | L1 发行主题，只输出颜色 / 字体 / 资产路径令牌 |
| 部署 | `apps/edge/server.mjs` | 路由、鉴权、静态资源、飞书外发；单进程启动整个系统 |

依赖方向是单向的：`terminals → server → core`。内核里不出现 HTML、不出现 `req/res`；页面里不出现规则判断。**新增终端或换品牌时不允许修改 core 里的任何一条规则**——这是这套结构成立的前提。

运行时只有一个第三方依赖：`mysql2`（用于 MySQL 驱动）。没有 Web 框架，HTTP 服务由 `node:http` 直接实现。

## 快速开始

环境要求：**Node.js ≥ 20**。

```bash
git clone <repo-url> star-werewolf
cd star-werewolf
npm install          # 只装 mysql2
```

选一种存储方式启动：

**A. JSON 文件（零数据库，适合本地预览与演示）**

```powershell
$env:STORE_DRIVER="json"; node apps/edge/server.mjs
```

```bash
STORE_DRIVER=json node apps/edge/server.mjs
```

**B. MySQL（默认驱动，接近生产形态）**

先在 MySQL 里建库并授权（默认库名 / 用户名 / 密码均为 `star_werewolf`）：

```sql
CREATE DATABASE star_werewolf DEFAULT CHARACTER SET utf8mb4;
CREATE USER 'star_werewolf'@'localhost' IDENTIFIED BY 'star_werewolf';
GRANT ALL PRIVILEGES ON star_werewolf.* TO 'star_werewolf'@'localhost';
FLUSH PRIVILEGES;
```

```powershell
$env:DB_HOST="127.0.0.1"; $env:DB_USER="star_werewolf"; $env:DB_PASSWORD="star_werewolf"; $env:DB_NAME="star_werewolf"
node apps/edge/server.mjs
```

表结构在启动时自动创建；若同目录存在旧的 `data/db.json`，会先整体迁入 MySQL 再置为迁移标记，不会静默丢弃旧数据。

启动后打开 `http://localhost:8080/`。同一网段的手机可以直接用局域网地址访问（默认监听 `0.0.0.0`）。

> 上面是本地跑通的方式。**生产环境建议用宝塔面板构建**（Node 项目托管 + 站点反向代理 + 开机自启 + 备份），步骤见 [部署](#生产环境推荐宝塔面板)。

## 跑一局

1. 首页点「我是主持人」→ 选局型（12 / 12i / 8 / 6）建房，自动进入候场页；
2. 候场页给空座「命名并生成邀请」，把链接发给对应玩家；也可从选手库直接「指定选手」；
3. 玩家凭邀请链接或 4 位房号入座，凭据存在自己手机里；
4. 人齐后「开始游戏」→ 玩家各自翻牌确认身份（主持人可代确认）；
5. 主持人「进入首夜」，按 守卫 → 狼人 → 女巫 → 预言家 逐段推进，各角色在玩家端提交行动，主持人每段点「下一夜间阶段 / 结算天亮」；
6. 天亮公布死讯 → 有死者先进遗言 → 再按座位顺序发言（默认 120 秒/位）；
7. 发言结束进投票，主持人可先「设置候选」，封票后结算；平票进 PK 轮，再平则本轮无人出局；
8. 猎人夜间被狼杀或白天被放逐可开枪（被毒不可），可放弃；
9. 狼人全灭 → 好人胜；神职全灭或平民全灭 → 狼人胜（屠边），自动进入结束与复盘。

控场台的主按钮由服务端 `nextActions()` 给出「当前阶段唯一正确的一步」，现场不会因为按钮点错顺序而卡死。建房页的「创建测试房」座位自动补满，可跳阶段、可重置，专门用于验收流程。

首次打开后台 `/DB` 会引导创建第一个超级管理员——**没有默认口令**，也没有内置账号。

## 页面与路由

| 终端 | 地址 | 谁能用 |
|---|---|---|
| 首页 | `/` | 所有人，主持 / 玩家双入口 |
| 建房 | `/host/create` | 主持人，选局型后建房 |
| 候场 | `/host/room/:roomId` | 主持人，发座位、改昵称、指定选手、开局 |
| 控场 | `/host/game/:roomId` | 主持人，推进阶段、夜间结算、发言、投票 |
| 玩家登录 | `/player/login` | 赛事账号 / 飞书身份 / 普通玩家三入口 |
| 玩家入座 | `/player/join?code=` 或 `?seat=<邀请凭据>` | 凭房号或座位邀请 |
| 玩家房间 | `/player/room/:roomId` | 玩家，看身份、提交夜间行动、投票、开枪 |
| 复盘 | `/review/:roomId` | 主持、本局玩家、后台 |
| 直播贴片 | `/stage/:roomId` | 只读，透明底，可作 OBS 浏览器源 |
| 管理后台 | `/DB`（等价 `/db`） | 管理员账号登录 |

凭据通过 URL 参数或本机 `localStorage` 携带（`hostToken` / `playerToken` / `adminToken`），前端每 2 秒轮询状态。**换浏览器等于换身份**，现场建议主持人固定一台设备。

## 局型与角色

| 局型 | 座位 | 配比 |
|---|---|---|
| `12` 标准 | 12 | 狼人 4 · 预言家 1 · 女巫 1 · 猎人 1 · 守卫 1 · 村民 4 |
| `12i` 变体 | 12 | 狼人 4 · 预言家 1 · 女巫 1 · 猎人 1 · 白痴 1 · 村民 4 |
| `8` 进阶 | 8 | 狼人 3 · 预言家 1 · 女巫 1 · 猎人 1 · 村民 2 |
| `6` 新手 | 6 | 狼人 2 · 预言家 1 · 猎人 1 · 村民 2 |

角色共 7 种：狼人、预言家、女巫、猎人、守卫、白痴、村民。神职 / 平民 / 狼人三个类别只用于胜负判定。

状态机（`packages/core/domain.mjs`）：

```
waiting → role_confirm → night_guard → night_wolf → night_witch → night_seer
        → day_announce → last_words → day_speech → day_vote → hunter_shot → ended
```

守卫阶段仅在有守卫的局型出现，女巫阶段同理；遗言阶段仅在有死者时进入；猎人开枪仅在触发时进入。

## 规则要点

- **胜负取屠边**：狼人 0 → 好人胜；神职 0 或平民 0 → 狼人胜。
- **守卫**：不可连续两晚守同一人，不可守已出局者；**同守同救仍死**。
- **女巫**：解药与毒药同夜二选一；解药只救当夜被刀者；仅首夜可自救；被毒者不可开枪。
- **猎人**：被狼杀、被放逐可开枪，被毒不可；可主动放弃，之后进入下一夜。
- **白痴**：首次被放逐翻牌免死，此后失去投票权，也不再被投票。
- **投票**：首轮平票进 PK，PK 再平则本轮无人出局；主持人可强制放逐。
- **赛事叫号**：赛事房开局锁定翻牌顺序，只有被叫到的座位能确认身份，其余服务端返回 400。

所有判定集中在 `packages/core/rules.mjs`，终端与部署层不得复制其中任何一条。

## 权限与安全

权限分三层，全部在服务端判定：

1. **管理员会话**：PBKDF2-SHA256（10000 次迭代 + 随机盐），会话 7 天，连续 5 次失败锁定 15 分钟；首个管理员在 `/DB` 初始化，无内置口令。
2. **赛事权限**：按后台账号 `userId` 授予，`permissionOf()` 是唯一判定入口。创建或开启赛事房必须同时满足「服务端 `TEST_MODE=1`」与「账号已授权」，任一不满足直接 403；前端隐藏只是第一层。
3. **房间角色**：`hostToken` / `playerToken` / `adminToken` 决定拿到哪一份视图。

另外两条与安全直接相关的设计：

- **视图裁剪在服务端完成**：玩家视图的 JSON 本身就不含他人 `role`，不做「先给全量再前端隐藏」。
- **敏感字段落盘加密**：`hostToken`、`playerToken`、`inviteToken`、会话令牌、飞书 `appSecret` 以 AES-256-GCM 密文入库，主密钥取环境变量 `STARWOLF_SECRET_KEY` 或数据目录下的 `.secret-key`；密码只存 PBKDF2 哈希，不落明文。

房号前缀便于现场辨认：普通 4 位数字、赛事 `E`+3 位、测试 `T`+3 位、赛事测试 `TE`+3 位。

## API

服务端没有 Web 框架，路由表集中在 `apps/edge/server.mjs`。鉴权头为 `x-host-token` / `x-player-token` / `x-admin-token`（GET 也可用 `?token=`）；成功返回数据本体，失败统一 `{ "error": "中文原因" }` + 4xx。

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/api/health` | 健康检查 |
| GET | `/api/theme` | 主题令牌（公开，含 `testMode`） |
| GET | `/api/scripts` | 局型与配比 |
| POST | `/api/werewolf/rooms` | 建房（赛事房校验测试模式 + 权限） |
| POST | `/api/werewolf/rooms/join` | 凭房号或 `inviteToken` 入座 |
| POST | `/api/werewolf/player-login` | 飞书身份入座 |
| GET | `/api/werewolf/rooms/:id/state` | 状态查询（主持 / 玩家 / 公共视图） |
| GET | `/api/werewolf/rooms/:id/review` | 复盘（参与方或后台） |
| POST | `/api/werewolf/rooms/:id/start` · `/enter-first-night` · `/advance` | 开局、进首夜、推进阶段 |
| POST | `/api/werewolf/rooms/:id/night-action` · `/resolve-night` | 夜间提交与结算 |
| POST | `/api/werewolf/rooms/:id/vote` · `/finish-vote` · `/exile` · `/hunter-shot` | 投票、封票、放逐、开枪 |
| POST | `/api/werewolf/rooms/:id/call-next` · `/pass-seat/:seatNo` | 赛事叫号与手动放行 |
| GET/POST/PATCH/DELETE | `/api/werewolf/admin/stage-presets` | 直播贴片预设 |
| GET/POST/DELETE | `/api/werewolf/admin/tournament-permissions` | 赛事权限授予 |
| GET/POST/PATCH/DELETE | `/api/tournament-players` · `/batch-import` · `/bind-feishu` | 选手库与飞书绑定 |
| POST/GET | `/api/admin-auth/...` | 初始化、登录、会话、管理员账号 |
| POST/GET/DELETE | `/api/werewolf/test/rooms...` | 测试房建 / 重置 / 跳阶段 / 删 |

完整接口表见 [`docs/05 运行与部署/运行说明.md`](./docs/05%20运行与部署/运行说明.md) 第九节。

## 存储与配置

存储是可替换的 SPI，两个内置驱动共用同一组方法（`getRoom / putRoom / listRooms / dropRoom / list / add / remove`）：

| `STORE_DRIVER` | 说明 |
|---|---|
| `mysql`（默认） | 三张表（`sw_rooms` / `sw_records` / `sw_meta`），启动时自动建表；写库异步合并，不阻塞请求 |
| `json` | 单文件 `data/db.json`，写 `.tmp` 再改名，进程被强杀不会留半截文件 |

| 变量 | 默认值 | 说明 |
|---|---|---|
| `PORT` | `8080` | 监听端口 |
| `BIND` | `0.0.0.0` | 监听地址，现场版需要同网段手机访问 |
| `DATA_FILE` | `data/db.json` | JSON 驱动的数据文件，同时是 MySQL 首次迁移的旧文件路径 |
| `STORE_DRIVER` | `mysql` | `mysql` / `json` |
| `DB_HOST` · `DB_PORT` · `DB_USER` · `DB_PASSWORD` · `DB_NAME` | `127.0.0.1` · `3306` · `star_werewolf` · `star_werewolf` · `star_werewolf` | MySQL 连接参数 |
| `DB_PREFIX` | `sw_` | 表名前缀 |
| `STARWOLF_SECRET_KEY` | 无（回退 `data/.secret-key`） | 64 位十六进制主密钥，用于落盘加密 |
| `TEST_MODE` | `1` | `1` 允许演示赛事链路；正式对外务必设 `0`，赛事入口与接口一并关闭 |
| `PUBLIC_BASE` | 取请求 `Host` | 邀请链接的基地址，反代场景建议显式指定 |
| `MYSQL_BIN` | 自动探测 | `mysql` 客户端路径，仅在批量落盘与迁移时使用 |

重置数据：停进程 → 备份并移除数据文件（或清空库表）→ 重启，管理员需重新初始化。

## 测试与自检

两个自检脚本都不需要打开浏览器：

```powershell
node scripts\smoke.mjs         # 需要服务已启动，BASE 可用环境变量覆盖
node scripts\check-pages.mjs   # 十个页面的内联脚本语法与模板残留
```

`smoke.mjs` 从建房一路打到复盘，覆盖：健康与四档局型、按序入座与满座拒绝、发牌与身份可见性、夜间提交与结算、遗言 / 发言不卡死、投票与 `lastVote` 落库、复盘 403、后台初始化与分页、赛事双门（无权限 403 / 有权限可建 / 公共视图不暴露赛事房）、座位命名 / 邀请 / 换座 / 踢出、贴片设置与预设、房号前缀、叫号与手动放行、后台筛选、选手飞书身份同步。改动规则或接口后先补断言再改代码。

`check-pages.mjs` 对十个页面加两个共享资源逐个做语法解析，并检查模板残留字符。

## 部署

部署分两种用途：现场开局（单机、局域网）与生产对外（进程守护、反向代理、开机自启、备份）。

### 现场版（单机、局域网）

```powershell
scripts\start.ps1     # 隐藏窗口常驻，PID 与日志写入 logs\
scripts\stop.ps1
```

需要从其他设备访问时放行端口：

```powershell
New-NetFirewallRule -DisplayName "star-werewolf 8080" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow
```

### 生产环境（推荐：宝塔面板）

**生产环境建议使用宝塔面板构建**：Node 进程、站点、反向代理、开机自启、日志切割与备份全部交给面板托管，不必自建 systemd / 计划任务，也不必手写 Nginx 配置。面板在 Windows 与 Linux 上操作一致，下面按面板里的位置写。

1. **装 Node 运行环境**：软件商店 → 「Node.js 版本管理器」→ 安装 → 选一个 LTS（建议 22.x）。
2. **建 Node 项目**：网站 → Node 项目 → 添加。

   | 字段 | 填写 |
   |---|---|
   | 项目目录 | 仓库根目录 |
   | 启动文件 | `apps/edge/server.mjs` |
   | 运行方式 | node |
   | 端口 / 绑定 | `8080` / `0.0.0.0` |
   | 开机自启 | 勾选 |

   环境变量必须**一行一个 `KEY=VALUE`**，不要用引号、分号或空格拼成一行（整串会被当成一个变量名，`TEST_MODE` 不生效、`DATA_FILE` 还会被解析成非法路径）：

   ```
   STORE_DRIVER=mysql
   DB_HOST=127.0.0.1
   DB_PORT=3306
   DB_USER=star_werewolf
   DB_PASSWORD=<强口令>
   DB_NAME=star_werewolf
   STARWOLF_SECRET_KEY=<64 位十六进制>
   TEST_MODE=0
   ```

3. **建站点 + 反向代理**：站点根目录指向 `packages/terminals`，反向代理目标 `http://127.0.0.1:8080`，发送域名 `$host`。
4. **放行端口**：`80` / `443`。
5. **验收**：`/api/health` 返回 `{"ok":true,...}`；`/DB` 可正常打开并完成管理员初始化；`/api/theme` 的 `testMode` 为 `false`。
6. **备份**：面板计划任务按天备份数据库与 `data/` 目录，保留 7 天。

三条生产注意：

- **必须设 `TEST_MODE=0`**，赛事入口与赛事接口一并关闭；要办赛时临时改 `1` 再重启项目。
- **`STARWOLF_SECRET_KEY` 一旦启用不要更换**，否则已加密的凭据与会话无法解密（未配置时会回退数据目录下的 `.secret-key`，两者需一起备份）。
- **改代码后必须在面板点「重启」**，命令行结束不了面板托管的进程。

完整步骤、反向代理 `expires 12h` 缓存陷阱与验收清单见 [`deploy/宝塔生产部署.md`](./deploy/宝塔生产部署.md)。

Cloud 托管版（多租户与控制面）属于路线图 M2，本仓库当前落地的是 Edge 现场版。

## 目录结构

```
star-werewolf/
├── apps/
│   └── edge/server.mjs        # 部署层：路由、鉴权、静态资源、飞书外发
├── packages/
│   ├── core/                  # 内核：不碰页面
│   │   ├── domain.mjs         #   局型、角色、阶段与标签
│   │   ├── rules.mjs          #   发牌、夜间合法性、胜负、投票、开枪
│   │   ├── room.mjs           #   房间状态机、座位、视图、复盘、测试房
│   │   ├── auth.mjs           #   管理员、会话、赛事权限、选手库
│   │   ├── secret-box.mjs     #   落盘加密
│   │   ├── store.mjs          #   存储驱动入口（mysql / json）
│   │   ├── store-mysql.mjs    #   MySQL 驱动
│   │   └── store-json.mjs     #   JSON 驱动
│   ├── terminals/             # 四终端页面 + 共享 app.js / app.css + 品牌资产
│   └── themes/night/          # L1 发行主题（紫夜金）
├── scripts/                   # 启动、停止、冒烟、页面自检
├── deploy/                    # 部署脚本与宝塔部署说明
├── data/                      # 运行时数据（已 gitignore）
├── docs/                      # 需求基线、架构方案、运行与部署、复刻对照、品牌 VI
├── assets/                    # 发布用图（主视觉、品牌标识、角色卡）
└── 资料清单.md                 # 资料版本与缺口
```

## 路线图

- [x] Edge 现场版：单进程跑通建房到复盘，四终端页面与后台可用
- [x] 规则引擎与状态机抽离：判定集中在 `packages/core`
- [x] 存储 SPI：MySQL / JSON 双驱动，旧数据自动迁移
- [ ] M0 基线冻结：把现有线上行为固化为回放用例
- [ ] M1 内核抽离收尾：命令与查询彻底从路由层剥离
- [ ] M2 双部署：现场版与托管版共用同一镜像
- [ ] M3 四终端：主持、玩家、直播、后台统一接入同一 SDK
- [ ] M4 品牌分层：主题包与租户配置分离
- [ ] M5 开源发布：CI、示例活动与首个正式版本

## 已知边界

- 云端多租户、控制面与计费尚未实现；本仓库只有 Edge 现场版。
- 终端只包含 Web 四端；小程序与原生端未包含。
- 飞书为真实调用 `open.feishu.cn`，但需要自有应用凭据（`App ID` / `App Secret` 与对应 scope），平台登录态注入只在妙搭环境可用。
- 页面验证目前到「脚本语法 + HTTP 200 + 接口级断言」，未做浏览器内逐交互回归。

## 参与贡献

1. Fork 并新建分支，改动尽量只落在 `packages/core` 或某个终端内；
2. 先补断言、再改代码，提交前跑通 `check-pages.mjs` 与 `smoke.mjs`；
3. 规则改动必须补冒烟断言；新增接口同步更新 `docs/05 运行与部署/运行说明.md` 的接口表；
4. 不要在终端里复制规则判定——那被视为缺陷，而不是优化；
5. 新能力默认关闭，关闭状态下不得改变既有流程的任何一步。

## 许可证

[MIT License](./LICENSE)，Copyright (c) 2026 辽宁神音科技有限公司。

项目名称「star-werewolf / 星光狼人杀」、标识与默认主题品牌资源（`packages/terminals/assets/brand/`）属于著作权人的商标与品牌资产，不在许可证授权范围内，二次发行请更换名称与标识。

---

| 想了解 | 先看 |
|---|---|
| 项目要做什么、怎么分层 | [`docs/02 架构方案/star-werewolf 项目方案.docx`](./docs/02%20架构方案/) |
| 系统现行能力与需求基线 | [`docs/01 需求基线/狼人杀活动系统 开源项目说明 V2.0.docx`](./docs/01%20需求基线/) |
| 怎么跑、怎么配、怎么排障 | [`docs/05 运行与部署/运行说明.md`](./docs/05%20运行与部署/运行说明.md) · [`系统手册.md`](./docs/05%20运行与部署/系统手册.md) |
| 与线上应用的功能对照 | [`docs/06 复刻对照/线上功能对照清单.md`](./docs/06%20复刻对照/线上功能对照清单.md) |
| 品牌与 VI | [`docs/07 品牌与VI/VI 规范.md`](./docs/07%20品牌与VI/VI%20规范.md) |
| 资料版本与缺口 | [`资料清单.md`](./资料清单.md) |
