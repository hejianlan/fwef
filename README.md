# Git Watch

监测多个 GitHub 开发者的动态，发现 **新建仓库**、**私有仓库转公开**、**代码推送**、**公开仓库数量增加** 四类事件后实时通知，并可自动下载仓库内容快照供站内下载。仓库快照检测同时走 GitHub REST 与 GraphQL 两条通道。

## 功能

- **多开发者监测**：添加任意 GitHub 用户名，后台轮询其公开活动与仓库列表。
- **四类事件检测**：
  - 🆕 新建仓库
  - 🔓 私有仓库转公开
  - ⬆️ 代码推送（可开关）
  - 📈 公开仓库数量增加：公开仓库总数较基线增加时汇总通知（N → M + 新增仓库清单，标注新建/私转公），不触发快照下载
- **REST + GraphQL 双路检测**：
  - 事件流（CreateEvent / PushEvent）走 REST；仓库快照 REST 与 GraphQL 并行拉取，按仓库数字 id 对齐合并，互为兜底。
  - 事件卡片显示检测通道徽章（REST / GraphQL / 两路同时命中）。
  - GraphQL 与 REST 限额池相互独立（各 5000/小时）；GraphQL 故障或被限流时自动降级为仅 REST，不影响轮询。
- **四种通知方式**：
  - 站内实时事件流（SSE，无需刷新）
  - 浏览器桌面通知（需授权）
  - 邮件（SMTP，支持 465 SSL / 587 STARTTLS，可在设置中发测试邮件；运行错误也可邮件告警，同一错误恢复前只通知一次，可选恢复通知）
  - Webhook 群机器人（飞书 / 企业微信 / 钉钉 / 通用 JSON）
- **登录鉴权**：所有 `/api` 接口与页面需登录，首次启动默认账号 `admin` / `admin123`（请及时在设置中修改）。
- **仓库内容归档下载**：事件产生后服务器经代理自动抓取仓库 zip 快照，通过本站链接（`/api/events/:id/download`）下发，不经 git/GitHub 直链。
  - 默认保存到本机 `data/archives/`；**启用七牛云存储后改为上传七牛（Kodo），不在服务器磁盘保留 zip**，下载链接 302 跳转到七牛外链（私有空间自动签名，有效期 1 小时）。
  - 七牛外链域名为**可选**：留空即「仅上传」模式——快照仍上传七牛，但页面不提供下载（服务器不落盘，故此时下载不可用）。
  - 推送事件按精确 head SHA 抓取，新建/转公开按最新提交抓取。
  - 同仓库同版本只存一份；手动触发不受大小上限限制。
  - 状态机：下载中 / 就绪 / 空仓库 / 超限 / 失败。
- **出站代理**：自动探测本机 Clash（`mixed-port: 7890`），也支持手动指定 `http://host:port` 或 `direct` 直连；设置面板内置连通性测试。
- **可选 GitHub Token**：配置后 REST 限额从 60 提升至 5000/小时、GraphQL 获得独立 5000 points/小时池，并获取完整 PushEvent payload（匿名时需额外调 compare 补全）；codeload 下载不占 core 配额。
- **七牛云存储（可选）**：在设置中填入 AccessKey / SecretKey / 空间名并启用后，仓库快照直接上传七牛云对象存储，不再占用服务器磁盘。外链域名可选：填了则可从七牛外链下载（私有空间自动签名），留空则仅上传、不提供下载。区域由官方 SDK 自动探测，无需手动选择。
- **轻量存储**：数据存于 `data/db.json`，事件上限 2000 条；未启用七牛云时归档文件存于 `data/archives/`。

## 快速开始

### 环境要求

- Node.js >= 18

### 安装与启动

```bash
npm install
npm start
```

启动后控制台会提示访问地址，默认为 <http://localhost:3210>。

端口可用环境变量覆盖：`PORT=8080 npm start`。

## 使用方法

1. 打开 <http://localhost:3210>，使用账号 `admin` / 默认密码 `admin123` 登录（首次启动后请立即修改）。
2. 在输入框填入 GitHub 用户名（如 `torvalds`），点击添加，开始监测。添加时静默建立基线，之后公开仓库数量增加才会通知。
3. 新事件会实时出现在事件流中；未读数量显示在标题栏。
4. 每张事件卡片底部根据归档状态显示：
   - **下载中** — 服务器正在抓取快照
   - **就绪** — 点击下载按钮获取 zip
   - **空仓库** — 仓库无内容可下载
   - **超限** — 超过自动下载大小上限，可点击手动下载（不受上限约束）
   - **失败** — 可点击重试
5. 点击页面右上角「设置」可配置：
   - GitHub Token（可选，提高 API 限额）
   - 轮询间隔（最低 30 秒，默认 300 秒；多位开发者并发检查，间隔即检测周期）
   - 是否监测代码推送
   - 网络代理（自动 / 直连 / 手动 + 测试按钮）
   - 自动下载开关与大小上限（0 = 不限制）
   - 七牛云存储（AccessKey / SecretKey / 空间名 / 对象前缀 / 外链域名（可选）/ 是否私有空间），可点击「测试七牛云」校验配置；启用后快照不再保存在服务器磁盘
   - Webhook 地址与格式（飞书 / 企业微信 / 钉钉 / 通用）
   - SMTP 邮件（主机 / 端口 / 账号 / 授权码 / 收件人），保存后可发测试邮件；可另开「运行错误邮件告警」与「错误恢复通知」
   - 登录账号与密码修改
6. 桌面通知开关在页面顶部，点击授权后新事件会弹窗提醒。

> 配置了 GitHub Token 后 GraphQL 通道才会启用；未配置 Token 时自动退回纯 REST 单路。
>
> 轮询间隔是「发现事件的最快周期」：GitHub 事件流（Events API）本身有约 30–90 秒的传播延迟，事件在 GitHub 侧可见后才会被下一轮检测到，因此从推送发生到收到通知通常为「GitHub 延迟 + 最多一个间隔」。

## API 速览

除 `/api/login` 外，所有接口均需请求头 `Authorization: Bearer <token>`（登录返回的 JWT）。

| 方法         | 路径                                            | 说明                                            |
| ---------- | --------------------------------------------- | --------------------------------------------- |
| POST       | `/api/login`                                  | 登录 `{ username, password }`，返回 token（唯一免鉴权接口） |
| GET        | `/api/auth/status`                            | 当前登录状态                                        |
| PUT        | `/api/auth/credentials`                       | 修改账号/密码                                       |
| GET        | `/api/state`                                  | 总览状态（开发者、统计、设置）                               |
| POST       | `/api/developers`                             | 添加开发者 `{ login }`                             |
| DELETE     | `/api/developers/:id`                         | 移除开发者                                         |
| POST       | `/api/check-now`                              | 立即检测一次                                        |
| GET        | `/api/events?type=&developer=&unread=&limit=` | 事件列表（`type=repos_increased` 为数量增加事件）          |
| POST       | `/api/events/read`                            | 标记已读 `{ ids: [] }`                            |
| POST       | `/api/events/:id/archive`                     | 手动抓取/重试 `{ force: bool }`                     |
| GET        | `/api/events/:id/download`                    | 下载仓库 zip 快照（本机流式下发；已存七牛时 302 跳转外链，「仅上传」模式返回 409；数量增加事件返回 404） |
| GET/DELETE | `/api/errors`                                 | 运行错误日志列表 / 清空                                 |
| POST       | `/api/errors/read`                            | 错误日志标记已读                                      |
| GET/PUT    | `/api/settings`                               | 读取/更新设置（Token、SMTP 密码、七牛 AK/SK 等仅写入、不回显明文）      |
| GET        | `/api/proxy/status`                           | 代理状态                                          |
| POST       | `/api/proxy/test`                             | 代理连通性测试                                       |
| POST       | `/api/qiniu/test`                             | 七牛云配置测试（校验密钥/空间并上传删除探针对象、校验外链）              |
| GET        | `/api/avatar?u=<githubusercontent url>`       | 头像反代（域名白名单，防 SSRF）                            |
| GET        | `/api/stream`                                 | SSE 实时事件流                                     |
| POST       | `/api/webhook/test`                           | 发送测试 Webhook                                  |
| POST       | `/api/email/test`                             | 发送测试邮件                                        |

## 目录结构

```
.
├── src/
│   ├── server.js      # Express 入口，注册路由/SSE/鉴权，启动监测
│   ├── monitor.js     # 轮询引擎、REST+GraphQL 双路检测与事件生成
│   ├── github.js      # GitHub REST + GraphQL 客户端（经代理）
│   ├── notifier.js    # SSE 广播 + 桌面通知 + 邮件 + Webhook
│   ├── archive.js     # 仓库 zip 快照抓取/上传（本机或七牛云）/索引/清理
│   ├── qiniu.js       # 七牛云对象存储（上传、私有下载签名、删除、连通性测试）
│   ├── proxy.js       # 出站代理（自动探测 Clash）
│   ├── auth.js        # 登录鉴权（JWT、密码哈希）
│   ├── errorlog.js    # 运行错误日志
│   ├── routes.js      # REST API 路由
│   └── store.js       # data/db.json 读写
├── public/            # 前端静态资源
├── scripts/           # 辅助脚本（节点选择、归档测试等）
├── data/
│   ├── db.json        # 运行数据（已 gitignore）
│   └── archives/      # 仓库快照（已 gitignore）
└── package.json
```

## 许可

本仓库为私有项目，未发布开源许可。
