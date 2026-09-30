# 自建 RustDesk：API + 通讯录（记住 ID/密码）部署包

解决两个问题：

1. **通讯录 / 记住密码 / 点击即连** —— 部署 `lejianwen/rustdesk-api`，用客户端自带通讯录，密码存你的服务器。
2. **延迟大** —— 双方都在国内却用美国中继；见下文「延迟优化」。

## 一、服务器部署（美国 154.12.39.18）

### 1. 准备目录

```bash
mkdir -p /opt/rustdesk-server && cd /opt/rustdesk-server
```

把本目录的 `docker-compose.yml` 复制到 `/opt/rustdesk-server/`。
文件里已写好服务器地址 `154.12.39.18`；换机器时全局替换该 IP 即可。

**重要：`hbbs -r` 必须是真实可达的 IP/域名。** 写错会导致远程连接报 `Failed to secure tcp: deadline has elapsed`。

### 2. 启动

```bash
docker compose up -d
```

首次启动后 `./data/id_ed25519.pub` 会生成公钥：

```bash
cat ./data/id_ed25519.pub
```

记下这串 Key，客户端要用。

### 3. 放行端口（防火墙 + 云安全组）

| 端口 | 协议 | 用途 |
|---|---|---|
| 21116 | TCP + **UDP** | ID 服务 / **UDP 打洞（延迟关键）** |
| 21117 | TCP | 中继 hbbr |
| 21114 | TCP | API + Web 后台 + Web 客户端 |

### 4. 打开 Web 后台

浏览器访问：

```text
http://154.12.39.18:21114/_admin/
```

- 初始用户名：`admin`
- 初始密码：看 API 容器日志（只打印一次）：

```bash
docker logs rustdesk-api 2>&1 | grep -i pwd
```

**登录后立刻改掉 admin 密码。**

### 5. 后台建账号

「用户管理」里创建你自己的账号（用于客户端登录），记下用户名/密码。

## 二、客户端配置（电脑/手机）

1. **设置 → 网络 / 服务器**
   - ID 服务器：`154.12.39.18`
   - 中继服务器：留空或 `154.12.39.18:21117`
   - API 服务器：`http://154.12.39.18:21114`
   - Key：粘贴 `id_ed25519.pub` 的内容
2. **登录**：右上角/菜单登录你刚才创建的账号。
3. **通讯录** 出现后：
   - 「添加」→ 填远程电脑 **ID、别名、密码**（密码会存到你的服务器）
   - 之后点通讯录条目即可连接，**不用再输 ID/密码**
4. 连接密码时勾选「记住」，也会写入通讯录。

## 三、延迟优化（反应快）

当前慢的主因：**中继在美国**，打洞失败时画面绕一圈太平洋。

### 立刻做（不加服务器）

1. 确认 **UDP 21116** 在防火墙和云安全组都已放行（很多人只开了 TCP）。
2. 客户端勾选 **「启用 UDP 打洞」**。
3. 效果：能直连就走 P2P，画面不过服务器，延迟接近局域网；失败才走中继。

### 要更快（推荐加国内中继）

在国内 VPS 再跑一个 `hbbr`，让中继流量走国内：

```yaml
# 国内服务器 /opt/rustdesk-relay/docker-compose.yml
services:
  hbbr:
    container_name: rustdesk-hbbr-cn
    image: rustdesk/rustdesk-server:latest
    command: hbbr -k _
    network_mode: host
    restart: unless-stopped
```

然后把美国 hbbs 的启动参数 `-r` 改成国内中继：

```text
hbbs -r 国内IP:21117 -k _
```

重启 hbbs。ID 注册仍在美国，**中继流量走国内**，延迟会明显下降。

## 四、验收清单

- [ ] `curl -I http://127.0.0.1:21114` 有 HTTP 响应
- [ ] Web 后台能登录，admin 密码已修改
- [ ] 客户端能登录账号
- [ ] 通讯录添加 ID+密码后，**重启客户端条目还在**（已在服务器）
- [ ] 点击通讯录条目免手输连接成功
- [ ] UDP 打洞成功时连接为「直接」

## 五、回滚

```bash
cd /opt/rustdesk-server
docker compose stop api
```

客户端清空 API 服务器即可；hbbs/hbbr 远程连接不受影响。

## 六、安全提示

- 21114 建议用 Nginx/Caddy 反代 + HTTPS，并限制来源 IP。
- 设置 `RUSTDESK_API_JWT_KEY`。
- `./data-api` 里有通讯录和密码，备份时注意保密。
- 保留 `-k _`（强制加密），不要去掉。

## 文件说明

| 文件 | 用途 |
|---|---|
| `docker-compose.yml` | hbbs + hbbr + API 一键编排 |
| `README.md` | 本说明 |
