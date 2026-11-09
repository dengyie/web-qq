<!-- markdownlint-disable-file MD033 MD045 -->
<div align="center">

# WebQQ

**高可用无头 QQ 社群运维操作台与自愈式 NapCat 插件运行时**

[![License](https://img.shields.io/badge/license-MIT-green.svg?style=flat-square)](LICENSE)
[![Version](https://img.shields.io/badge/version-v1.6.5-blue.svg?style=flat-square)](package.json)
[![Node.js](https://img.shields.io/badge/Node.js-20+-339933.svg?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED.svg?style=flat-square&logo=docker&logoColor=white)](compose.yml)
[![Protocol](https://img.shields.io/badge/Protocol-OneBot_v11-orange.svg?style=flat-square)](https://11.onebot.dev/)
[![Tests](https://img.shields.io/badge/tests-48%20passed-brightgreen.svg?style=flat-square)]()

[English](./README.md) · [简体中文](./README.zh-CN.md) · [核心特性](#核心特性) · [系统架构](#系统架构全景) · [快速上手](#快速上手) · [高可用自愈](#高可用与故障自愈) · [配置参数](#配置参数参考)

<br>

<p align="center">
  <b>彻底终结无头 QQ 机器人运维过程中频发的风控封号、僵尸锁死与人工繁琐干预。</b><br>
  运行于现代化 NapCat 框架与 OneBot v11 协议之上，深度融汇有界内核队列、僵尸槽位强制回收与多级看门狗自愈体系。<br>
  提供单页网页群组秒级搜索、防冲撞加群去重、代际隔离群缓存、滑动窗口防暴力破解与 HA8 Cloudflare 隧道高可用守护。
</p>

</div>

---

## 痛点与架构对比

在社群运维与自动化机器人场景中，无头 QQ 客户端经常面临会话意外掉线、登录风控锁死、内核并发阻塞引发的连锁雪崩，以及缺乏友好的轻量管理界面等困境。WebQQ 构筑了一套高可用、自愈且轻量的运维解决方案：

| 评估维度 | WebQQ 高可用运维平台 | 传统 QQ 机器人插件 | 商业闭源社群管理软件 |
| :--- | :--- | :--- | :--- |
| **架构与依赖** | 纯自研独立轻量插件 + Docker 编排，零第三方源码捆绑 | 臃肿的大一统框架，依赖繁杂且易崩溃 | 封闭商业客户端，强制收集敏感通信数据 |
| **内核防雪崩** | 有界并发队列 + 单调用超时 + 僵尸槽位自愈强制回收 | 无排队保护，高并发瞬间压垮底层协议端 | 简单限流或无保护，卡死必须人工重启服务 |
| **掉线重登策略** | 10s 轻量探活 + 30m 指数退避，遇到滑块/新设备即刻停手 | 盲目定时死循环刷账密，极易秒撞风控永封 | 依赖云端短信接码，存在严重资产泄露风险 |
| **Web 鉴权与安全** | 口令无缝换取 HttpOnly Cookie + CSRF 同源校验 + 429 锁定 | 明文 Token 挂在 URL，历史记录与日志严重泄露 | 弱密码登录，缺乏防暴力破解与安全标头 |
| **公网隧道高可用** | Cloudflare 隧道双机热备，指标探测连续 3 轮为零才重启 | 单点隧道掉线即失联，需人工登 VPS 修复 | 必须采购昂贵的高防 BGP 专线 |
| **测试与质量工程** | 涵盖 44 项 Mock 回归 + 竞态/排队/超时/重登全量取证 | 缺乏自动化测试，修改代码全靠肉眼测试 | 闭源黑盒，缺陷无法快速定位 |

---

## 核心特性

| 模块 | 特性亮点 | 技术实现与工程优势 |
| :--- | :--- | :--- |
| **单页群组操作台** | 极简直观的网页化群管 | 单页实现群号精准搜索、加群申请（支持 `dryRun` 预览）、快捷退群、群列表浏览与机器人群内代发消息。 |
| **并发闸门与防卡死** | 内核有界队列与超时保护 | `GroupApi`/`MsgApi` 限制最大并发槽位；队列上限 8 个、最长排队 10s；超时回 503 且槽位在底层 settle 时安全释放。 |
| **僵尸槽位强制回收** | NTEvent 挂死自愈保障 | 当内核调用既不返回也不报错超过超时阈值 5 倍且 ≥60s 时，触发强制槽位回收机制，插件自愈杜绝永久 503。 |
| **平滑鉴权换约机制** | 口令自动换安全 Cookie | 首次携带 `?token=` 自动下发 `HttpOnly; SameSite=Lax` 安全 Cookie 并 302 重定向至干净 URL，避免口令泄露。 |
| **CSRF 与防爆破装甲** | 多层防御保障接口安全 | Cookie 来源的 POST 严格校验同源 Origin；同 IP 带错口令达 20 次/10 分钟自动进入 429 锁定，成功即销账。 |
| **群列表代际缓存** | 杜绝并发脏读与历史回填 | 群列表设置 4s TTL 缓存并按代际递增管理；在途请求返回时代际已变则自动丢弃旧快照，退群操作即刻失效缓存。 |
| **智能防风控重登** | 掉线感知与自适应退避 | 常驻看门狗 10s 轮询登录态；离线时距上次尝试 ≥30 分钟才重试、单轮最多 3 次；滑块验证或新设备验证立即停手。 |
| **HA8 隧道看门狗** | 宁漏判不误杀的隧道守护 | 优先采集 Connector 本地 Metrics 指标，回退按 MainPID 统计边缘连接；连续 3 轮为零才重启，支持双机热备。 |

---

## 系统架构全景

```text
┌────────────────────────────────────────────────────────────────────────┐
│                        管理员浏览器客户端                              │
│   ├─ 极简单页群组操作台 (群搜索 · 加群申请 · 退群 · 模板发消息)        │
│   └─ HttpOnly 安全 Cookie 鉴权 + CSRF 同源防护 + 302 纯净重定向        │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ 本地回环 :19001 或 Cloudflare 隧道接入
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│              NapCat Joiner 核心插件 (napcat-plugin-joiner)             │
│  ┌──────────────────────┐ ┌──────────────────────┐ ┌────────────────┐  │
│  │ 有界内核队列与闸门   │ │ 僵尸槽位回收看门狗   │ │ 群列表代际缓存 │  │
│  │ 最大并发 2 / 队列 8  │ │ 挂死超阈值自动重置   │ │ 4s TTL 变更失效│  │
│  └──────────┬───────────┘ └──────────┬───────────┘ └────────┬───────┘  │
│             │                        │                      │          │
│             ▼                        ▼                      ▼          │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │          OneBot v11 协议适配与底层 NTQQ 核心 (NapCat Docker)     │  │
│  └──────────────────────────────────┬───────────────────────────────┘  │
└─────────────────────────────────────┼──────────────────────────────────┘
                                      │
                                      ▼
┌────────────────────────────────────────────────────────────────────────┐
│                       高可用自愈看门狗守护集群                         │
│  ├─ qq-relogin-watch.service: 10s 掉线探活 + 30m 防风控退避重登        │
│  └─ cloudflared-watchdog.timer: HA8 边缘连接探活 + 连续 3 轮判定重启   │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 快速上手

### 1. 启动容器

```bash
# 复制环境配置文件
cp .env.example .env

# 编辑配置填入 WEBUI_TOKEN 与 QQ_ACCOUNT
# 本机使用默认绑定 127.0.0.1 即可
docker compose up -d

# 注入官方插件白名单（容器重建后必须重跑）
bash patch-whitelist.sh && docker restart qq-napcat
```

如需经内网或 Mesh 访问，叠加 `compose.mesh.yml` 配置并配置 `MESH_IP`：

```bash
docker compose -f compose.yml -f compose.mesh.yml up -d
```

### 2. 提取口令并访问控制台

插件首次启动会自动在插件目录生成安全 Token：

```bash
cat napcat/plugins/napcat-plugin-joiner/token
```

在浏览器中打开：`http://<host>:19001/?token=<token>`。
系统将自动种下 `HttpOnly` Cookie 并 302 重定向至无 Token 的纯净地址，后续请求自动保持登录态。

---

## 高可用与故障自愈

生产环境根据故障域实施分层自愈防护：

| 故障域 | 自愈机制与策略 |
| :--- | :--- |
| **插件与容器** | 有界队列排队满回 503；单调用超时隔离；僵尸槽位自动回收；容器策略为 `restart: unless-stopped`。 |
| **QQ 意外掉线** | `qq-relogin-watch` 常驻守护进程：10s 探活，掉线才动作，30 分钟严格退避防风控封禁。 |
| **公网隧道异常**| `cloudflared-watchdog.timer` 每 60s 探活：优先读取 HA Metrics 指标，连续 3 轮连接数为 0 触发重启。 |
| **容器重启更新**| `docker restart` 保持文件系统白名单；若执行 `docker compose down && up` 重建容器，重跑 `patch-whitelist.sh`。 |

### 部署掉线看门狗

```bash
# 停止旧版本定时器
systemctl disable --now qq-relogin.timer

# 安装并启动常驻看门狗
cp systemd/qq-relogin-watch.service /etc/systemd/system/
systemctl daemon-reload && systemctl enable --now qq-relogin-watch
```

---

## 配置参数参考

| 环境变量 | 作用说明 | 默认值 |
| :--- | :--- | :--- |
| `WEBUI_TOKEN` | NapCat WebUI 鉴权口令（必填，请设高强度密钥） | 无 |
| `QQ_ACCOUNT` | 登录运行的机器人 QQ 账号 | 无 |
| `MESH_IP` | `compose.mesh.yml` 额外监听绑定的网卡 IP | 空 |
| `JOINER_ALLOW_HOSTS` | 除同源外额外允许访问的 Host 列表（逗号分隔） | `qq.example.com` |
| `JOINER_TRUSTED_PROXIES` | 允许信任代理标头（`X-Forwarded-For`/`CF-Connecting-IP`）的源 IP | `127.0.0.1,::1` |
| `JOINER_KERNEL_MAX` | 底层内核最大并发调用槽位数 | `2` |
| `JOINER_KERNEL_QUEUE_MAX` | 内核最大排队数量（超额立即返回 503） | `8` |
| `JOINER_KERNEL_WAIT_MS` | 单次在队列中排队的最长等待时间（毫秒） | `10000` |
| `JOINER_KERNEL_TIMEOUT_MS`| 单次内核调用超时（毫秒，0 表示使用默认 15s/20s） | `0` |
| `JOINER_KERNEL_ZOMBIE_MS` | 判定为僵尸调用并强制回收槽位的时间阈值（毫秒） | `0`（自动按阈值计算） |
| `JOINER_AUTH_FAIL_WINDOW_MS`| 鉴权失败防爆破锁定的滑动时间窗口（毫秒） | `600000` |
| `QQ_WATCH_INTERVAL` | 看门狗检测登录态的心跳周期（秒） | `10` |
| `QQ_RELOGIN_MIN_INTERVAL` | 两次账密自动重登之间的强制安全冷却时间（秒） | `1800` |
| `QQ_RELOGIN_MAX_TRIES` | 单轮掉线最多允许尝试重登的次数 | `3` |

---

## 自动化测试与质量工程

本项目具备完备的测试与取证体系，一键运行全量回归套件：

```bash
npm test
```

测试覆盖 5 个阶段（耗时约 35 秒）：
1. **Mock 回归测试**：44 项用例覆盖鉴权、CSRF、限流、配额、代际缓存与自愈逻辑；
2. **竞态防冲撞取证** (`race-check.mjs`)：验证同一群组高并发点击时严格去重仅向底层提交一次；
3. **有界队列取证** (`queue-check.mjs`)：验证队列满时快速 503 且排队超时正确释放；
4. **内核超时取证** (`timeout-check.mjs`)：验证底层挂死时槽位随 Promise settle 正确释放；
5. **重登防风控取证** (`relogin-check.py`)：使用 Mock WebUI 严格验证退避、上限、滑块与状态防损。

---

## 上游与致谢

- **NapCat / NapCatQQ**：第三方高性能 QQ 协议端，容器镜像为 `mlikiowa/napcat-docker`；
- **OneBot v11**：现代机器人插件与外部控制脚本之间的通用协议标准。

---

## 许可证

本项目采用 [MIT License](LICENSE) 开源协议。
