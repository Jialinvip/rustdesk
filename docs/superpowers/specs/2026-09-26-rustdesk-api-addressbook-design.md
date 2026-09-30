# RustDesk API + 通讯录（记密码）部署设计

日期：2026-09-26
范围：在现有自建 hbbs/hbbr 之上补齐「通讯录 + 保存 ID/密码 + 点击直连」，并给出降低延迟的拓扑建议。
非范围：用户/设备配额许可、OIDC/LDAP、分布式中继自动调度、自定义客户端生成器（后续子项目）。

## 背景

- 现状：仅部署 `rustdesk/rustdesk-server`（hbbs + hbbr），`-k _` 强制加密，中继指向 `154.12.39.18:21117`（美国）。
- 痛点 1：控制端与被控端均在国内，打洞失败时流量绕美国中继，延迟高。
- 痛点 2：客户端每次要手输 ID/密码；不记得 ID 就无法连接。
- 代码事实：客户端已有完整通讯录（`flutter/lib/models/ab_model.dart`），字段含 `id / alias / password / hash / tags / note`，连接支持 `loginWithPassword(password, remember)`。通讯录同步与登录依赖 API 服务（`userModel.isLogin` → `/api/login`、`/api/ab/*`），当前未部署 API，故功能不可用。

## 目标

1. 部署与官方客户端 API 兼容的服务端，支持登录、通讯录、密码保存到服务器。
2. 客户端登录后，从通讯录点击条目即可连接，自动填充密码。
3. 给出可执行的延迟优化路径（P2P 打洞 + 可选国内中继）。

## 方案选择

| 方案 | 结论 |
|---|---|
| A. 在现有 hbbs/hbbr 旁增加 `lejianwen/rustdesk-api` | **采用**。兼容 `/api/login`、`/api/ab/*`，含 Web 后台与日志，改动小可回滚 |
| B. 整体替换为 `lejianwen/rustdesk-server-s6` 一体化镜像 | 不采用。要迁现有数据，收益与 A 接近 |
| C. 自研轻量通讯录 API | 不采用。周期长，客户端已有现成对接面 |

## 架构

```
国内控制端 ──┐
             ├─ UDP 21116 打洞成功 → P2P 直连
             └─ 打洞失败 → hbbr 中继
                  │
服务器 154.12.39.18（当前在美国）
  hbbs   21116/udp+tcp   ID 注册 / 打洞协调
  hbbr   21117           中继
  api    21114  ← 新增    登录 / 通讯录 / Web 后台 / 日志
```

## 组件

### API 服务（新增）

- 镜像：`lejianwen/rustdesk-api:latest`
- 端口：`21114`（HTTP：PC API + `/_admin/` Web 后台 + Web Client）
- 数据：SQLite（默认），挂载到 `./data-api`，库内保存用户、通讯录（含密码字段）、日志
- 关键环境变量：
  - `RUSTDESK_API_RUSTDESK_ID_SERVER=154.12.39.18:21116`
  - `RUSTDESK_API_RUSTDESK_RELAY_SERVER=154.12.39.18:21117`
  - `RUSTDESK_API_RUSTDESK_API_SERVER=http://154.12.39.18:21114`
  - `RUSTDESK_API_RUSTDESK_KEY_FILE=/app/data/id_ed25519.pub`（与 hbbs 公钥一致）
  - `RUSTDESK_API_RUSTDESK_PERSONAL=1`（启用个人版 API，客户端通讯录需要）

### 客户端对接（不改代码）

1. 设置 → 网络：ID 服务器 `154.12.39.18`，API 服务器 `http://154.12.39.18:21114`，Key 填 `id_ed25519.pub` 内容。
2. 登录 API 账号（Web 后台创建或注册）。
3. 通讯录 → 添加 ID、别名、密码；密码写入服务器通讯录。
4. 点击通讯录条目 → 自动带出 ID/密码并连接。

### 通讯录数据流

```
客户端 addIdToCurrent(id, alias, password)
  → POST /api/ab/peer/add/{guid}
  → API 写入数据库
  → 客户端 pullAb 同步
  → 连接时 loginWithPassword(remembered) 自动填充
```

个人通讯录存密码哈希（`hash`），共享通讯录可写明文密码并按权限共享给组内成员。

## 延迟设计

延迟主因是**中继在美国**。分两层：

1. **立即（配置）**：放行 UDP 21116，让打洞成功走 P2P。直连不经过服务器，延迟最低。
2. **可选（拓扑）**：在国内 VPS 跑第二个 `hbbr`，将 hbbs 的 `-r` 改为国内中继地址。ID 注册可留美国。

本次交付不强制迁服务器；API 增量部署，延迟优化给出可复制配置。

## 安全

- 保留 `-k _` 强制加密。
- 首次登录后立即修改 admin 初始密码（控制台会打印一次）。
- 建议设置 `RUSTDESK_API_JWT_KEY`；生产用反代 + HTTPS 收紧 21114。
- 通讯录密码进 API 数据库，备份 `./data-api` 时注意敏感数据。
- 防火墙仅开放：21116/tcp+udp、21117/tcp、21114/tcp（或经反代 443）。

## 错误处理

| 现象 | 处理 |
|---|---|
| api 容器起不来 | `docker logs rustdesk-api`；检查 21114 占用 |
| 客户端登录失败 | 核对 API 地址、KEY 与 `id_ed25519.pub` 一致 |
| 通讯录为空/不同步 | 确认已登录；`RUSTDESK_API_RUSTDESK_PERSONAL=1` |
| 连接仍慢 | 查是否走中继；确认 UDP 21116 放行；必要时加国内 hbbr |

## 验收

1. `docker compose up -d` 后 `curl -I http://127.0.0.1:21114` 有响应。
2. 打开 `http://154.12.39.18:21114/_admin/`，用初始 admin 登录并改密。
3. 客户端登录 API 账号成功。
4. 通讯录添加 ID + 密码，重启客户端后条目仍在（已存服务器）。
5. 点击通讯录条目免手输连接成功。
6. （延迟）打洞成功时为直连；对比改造前中继路径延迟下降。

## 回滚

`docker compose stop api` 即回到纯 hbbs/hbbr；客户端清空 API 服务器配置即可，不影响远程连接本身。
