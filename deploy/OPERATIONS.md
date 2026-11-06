# ezBookkeeping 操作、部署与运维手册

> 适用实例：`123.207.65.209`（Ubuntu，腾讯云）  
> 访问地址：`http://123.207.65.209:8080/`（需云安全组放行 TCP 8080）  
> 文档版本：2026-11-06  

---

## 1. 架构总览

```text
浏览器 / 手机 PWA
        │  http://123.207.65.209:8080
        ▼
┌─────────────────────────────────────────┐
│  Docker: mayswind/ezbookkeeping:2.1.1   │
│  端口 8080 → 容器 8080                   │
│  配置: /opt/ezbookkeeping/conf/*.ini    │
│  数据卷: storage/ data/ log/            │
└─────────────────┬───────────────────────┘
                  │ host.docker.internal:5432
                  ▼
┌─────────────────────────────────────────┐
│  Docker: postgres:16（已有实例）          │
│  库名: ezbookkeeping                     │
│  账号: yuanguoqi（超管，仅本库业务使用）    │
└─────────────────────────────────────────┘
```

| 组件 | 说明 |
|------|------|
| 应用镜像 | `mayswind/ezbookkeeping:2.1.1`（官方 Release 固定 tag） |
| 编排 | Docker Compose v2 |
| 数据库 | 宿主机已有 Postgres 容器，映射 `0.0.0.0:5432` |
| 反向代理 | **无**（本阶段直连 8080） |
| HTTPS | **无**（无域名，HTTP） |

### 1.1 代码仓库分支约定

| 分支 | 职责 |
|------|------|
| `main` | 只同步上游 `https://github.com/mayswind/ezbookkeeping.git` |
| `linkyou` | 部署文件 + 将来少量定制（compose、模板、补丁） |

- 你的 fork：`https://github.com/yuanguoqi/ezbookkeeping.git`（`origin`）
- 上游：`upstream` → `mayswind/ezbookkeeping`

同步上游：

```bash
git fetch upstream
git checkout main && git merge upstream/main && git push origin main
git checkout linkyou && git merge main
```

---

## 2. 部署

### 2.1 服务器目录

```text
/opt/ezbookkeeping/
├── docker-compose.yml      # 与仓库 linkyou 分支一致
├── conf/ezbookkeeping.ini  # 真实配置（含密钥，勿提交 git）
├── storage/                # 头像、交易图片等
├── data/                   # 预留（当前用 Postgres，非 sqlite）
├── log/                    # 应用日志
├── backup/                 # 备份输出
└── README.deploy.md        # 部署说明副本
```

### 2.2 docker-compose.yml（仓库内）

```yaml
services:
  ezbookkeeping:
    image: mayswind/ezbookkeeping:2.1.1
    container_name: ezbookkeeping
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      EBK_CONF_PATH: /ezbookkeeping/conf/ezbookkeeping.ini
    volumes:
      - ./conf/ezbookkeeping.ini:/ezbookkeeping/conf/ezbookkeeping.ini:ro
      - ./storage:/ezbookkeeping/storage
      - ./data:/ezbookkeeping/data
      - ./log:/ezbookkeeping/log
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

要点：

1. **配置外置**：镜像内默认 ini 被 volume 覆盖；真实配置只在服务器。
2. **连宿主机 Postgres**：必须 `host.docker.internal:host-gateway`，ini 中写 `host = host.docker.internal:5432`。容器内 `127.0.0.1` 指向容器自己，不能连库。
3. **固定镜像 tag**：升级时改 tag，不要盲目追 `latest`。

### 2.3 配置要点（`conf/ezbookkeeping.ini`）

| 配置项 | 当前值 / 要求 |
|--------|----------------|
| `server.domain` | `123.207.65.209` |
| `server.root_url` | `http://123.207.65.209:8080/` |
| `database.type` | `postgres` |
| `database.host` | `host.docker.internal:5432` |
| `database.name` | `ezbookkeeping` |
| `database.user` | `yuanguoqi` |
| `database.passwd` | **仅存服务器**，不进 git |
| `security.secret_key` | **随机 64 位**，仅存服务器；丢失会导致会话/相关数据异常 |
| `user.enable_register` | `false`（禁止自助注册） |
| `server.enable_gzip` | `true` |

仓库模板：`conf/ezbookkeeping.ini.example`（占位符，无真实密钥）。

### 2.4 云安全组（必须）

腾讯云控制台 → 轻量/CVM 安全组 → 入站规则：

| 协议 | 端口 | 源 | 动作 |
|------|------|-----|------|
| TCP | 8080 | `0.0.0.0/0`（或你的出口 IP） | 允许 |

未放行时：服务器本机 `curl http://127.0.0.1:8080` 正常，**外网超时**。

### 2.5 首次部署命令（服务器）

```bash
cd /opt/ezbookkeeping
docker compose pull
docker compose up -d
docker compose ps
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/
```

### 2.6 数据库初始化

```bash
# 在 postgres 容器内执行（口令见服务器 ini / 运维私密记录）
docker exec -e PGPASSWORD='***' postgres psql -h 127.0.0.1 -U yuanguoqi -d postgres \
  -c "CREATE DATABASE ezbookkeeping OWNER yuanguoqi;"
```

应用启动时 `auto_update_database = true` 会自动建表（约 17 张）。

---

## 3. 用户管理

注册已关闭（`enable_register = false`）。**只能通过 CLI 创建用户**。

### 3.1 创建用户

```bash
docker exec ezbookkeeping /ezbookkeeping/ezbookkeeping userdata user-add \
  --username <用户名> \
  --email <邮箱@example.com> \
  --nickname <昵称> \
  --password '<强密码>' \
  --default-currency CNY
```

注意：

- 两次 `user-add` 不要在同一秒内连续执行（内部 UID 按时间戳生成，可能撞主键）；失败时隔 1～2 秒重试。
- 密码含 `!`、`#`、`&` 等时，务必用单引号包裹，并在 shell 中正确转义。

### 3.2 查看 / 改密 / 启停

```bash
# 查看
docker exec ezbookkeeping /ezbookkeeping/ezbookkeeping userdata user-get --username <用户名>

# 改密
docker exec ezbookkeeping /ezbookkeeping/ezbookkeeping userdata user-modify-password \
  --username <用户名> --password '<新密码>'

# 禁用 / 启用
docker exec ezbookkeeping /ezbookkeeping/ezbookkeeping userdata user-disable --username <用户名>
docker exec ezbookkeeping /ezbookkeeping/ezbookkeeping userdata user-enable --username <用户名>
```

### 3.3 当前已创建用户（2026-11-06）

| 用户名 | 邮箱 | 默认货币 | 状态 |
|--------|------|----------|------|
| `yuanguoqi` | yuanguoqi@example.com | CNY | 已创建 |
| `liuzhaoqun` | liuzhaoqun@example.com | CNY | 已创建 |

初始密码：见会话交付说明（**不写入本 git 文档**）。登录后请立即在「设置 → 修改密码」改为自己的密码。

邮箱为占位符；未开 SMTP，**忘记密码邮件不可用**，忘记密码只能由运维用 CLI 改密。

---

## 4. 安卓 / iOS 如何使用

本项目**没有**应用商店原生 App，官方方式是 **PWA（渐进式 Web 应用）**：用系统浏览器打开站点，再「添加到主屏幕」，体验接近原生。

### 4.1 前提

1. 手机能访问 `http://123.207.65.209:8080/`（安全组已放行 8080；流量为 **HTTP 明文**）。
2. 已有账号（见上表），登录后使用。

### 4.2 Android（Chrome / 系统浏览器）

1. 用 **Chrome** 打开：`http://123.207.65.209:8080/`
2. 登录。
3. 右上角 `⋮` → **添加到主屏幕** / **安装应用**（若显示「安装」则点安装，生成独立图标与窗口）。
4. 从主屏幕图标进入，数据与浏览器会话一致（应用内登录态保存在本地）。

若菜单里只有「添加到主屏幕」没有「安装」：多为 HTTP 站点或未满足安装条件，仍可添加图标，功能可用，只是窗口形态略逊于完整安装。

### 4.3 iOS（Safari）

1. 用 **Safari** 打开：`http://123.207.65.209:8080/`（不要用 Chrome 添加主屏，iOS 以 Safari 为准）。
2. 登录。
3. 底部 **分享** 按钮 → **添加到主屏幕** → 添加。
4. 从主屏幕图标打开。

说明：

- iOS 对 **HTTPS** 的 PWA/全屏支持更完整；纯 HTTP + IP 时，通常仍可加主屏快捷方式，但全屏安装横幅、部分离线能力可能受限。
- 若需完整 PWA 体验：后续上域名 + 证书（Nginx/Caddy 反代 443）后再重新添加。

### 4.4 日常记账操作（两端相同）

1. 打开应用 → 登录。
2. 底部/侧栏进入交易列表 → **新建交易**（收入/支出/转账）。
3. 选择账户、分类、标签、金额、日期；可上传票据图片。
4. 「统计」查看图表；「设置」中可改语言、货币显示、锁定等。
5. 建议在设置中启用 **应用锁 / 2FA**（若使用共享手机）。

### 4.5 多设备

同一账号可在手机 + 电脑浏览器同时使用；数据在服务端 Postgres，换设备重新登录即可。

---

## 5. 日常运维

### 5.1 常用命令

```bash
cd /opt/ezbookkeeping

docker compose ps                 # 状态
docker compose logs -f --tail=100 # 日志
docker compose restart            # 重启
docker compose down               # 停止（不删卷/宿主机目录）
docker compose up -d              # 启动
```

应用日志文件：`/opt/ezbookkeeping/log/ezbookkeeping.log`（及容器 stdout）。

### 5.2 发布 / 升级镜像

**改了配置（ini）**：

```bash
# 服务器上直接改 conf/ezbookkeeping.ini 后：
cd /opt/ezbookkeeping && docker compose restart
```

**升级应用版本**：

```bash
# 1) 改仓库 docker-compose.yml 中 image tag
# 2) 推到 fork 后，在服务器同步 compose 文件，或手动编辑 /opt/ezbookkeeping/docker-compose.yml
cd /opt/ezbookkeeping
docker compose pull
docker compose up -d
curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/
```

**改了代码（linkyou 定制）**：需自建镜像（本阶段尚未启用自建）；当前仅配置定制，用官方镜像即可。

### 5.3 备份（必须做）

备份三件套：

1. **PostgreSQL**：`ezbookkeeping` 库  
2. **`storage/`**：图片、头像  
3. **`conf/ezbookkeeping.ini`**：含 `secret_key`（离线妥善保管，勿进 git）

示例（服务器 crontab，每日 03:17）：

```bash
#!/bin/bash
set -euo pipefail
TS=$(date +%Y%m%d_%H%M%S)
BK=/opt/ezbookkeeping/backup
mkdir -p "$BK"

docker exec -e PGPASSWORD='***' postgres \
  pg_dump -h 127.0.0.1 -U yuanguoqi -d ezbookkeeping \
  -Fc -f /tmp/ezbookkeeping.dump

docker cp postgres:/tmp/ezbookkeeping.dump "$BK/ezbookkeeping_${TS}.dump"
docker exec postgres rm -f /tmp/ezbookkeeping.dump

tar czf "$BK/storage_${TS}.tar.gz" -C /opt/ezbookkeeping storage
cp -a /opt/ezbookkeeping/conf/ezbookkeeping.ini "$BK/ezbookkeeping.ini.${TS}"

# 只保留 14 天
find "$BK" -name 'ezbookkeeping_*.dump' -mtime +14 -delete
find "$BK" -name 'storage_*.tar.gz' -mtime +14 -delete
find "$BK" -name 'ezbookkeeping.ini.*' -mtime +14 -delete
```

**恢复**：

```bash
docker exec -i -e PGPASSWORD='***' postgres \
  pg_restore -h 127.0.0.1 -U yuanguoqi -d ezbookkeeping --clean --if-exists \
  < /opt/ezbookkeeping/backup/ezbookkeeping_YYYYmmdd_HHMMSS.dump
# storage 解压回 /opt/ezbookkeeping/storage
```

### 5.4 监控与巡检

| 项 | 频率 | 方法 |
|----|------|------|
| HTTP 200 | 每日 | `curl -sS -o /dev/null -w '%{http_code}' http://127.0.0.1:8080/` |
| 容器 Up | 每日 | `docker compose ps` |
| 磁盘 | 每周 | `df -h /` |
| 备份存在 | 每日 | `ls -lt /opt/ezbookkeeping/backup | head` |
| 异常登录 | 需要时 | `docker compose logs | grep -i error` |

### 5.5 故障排查

| 现象 | 排查 |
|------|------|
| 外网打不开，本机 200 | 云安全组未放行 8080；检查入站规则 |
| 502 / 连不上库 | `docker compose logs`；确认 `host.docker.internal:5432`；`docker compose exec ezbookkeeping ...` 测连通 |
| 启动报 secret_key | 检查 ini 是否被空值覆盖 |
| 注册页报错/无注册 | 预期行为：`enable_register = false` |
| 忘记密码 | 无 SMTP：运维 CLI `user-modify-password` |
| 用户创建报 user_pkey | 同秒 UID 冲突，隔 1～2 秒重试 |
| 升级后表结构 | 一般自动迁移；升级前务必先备份 |

---

## 6. 安全基线

1. **密钥不进 git**：`secret_key`、数据库口令只放服务器 ini。  
2. **关闭注册**：仅 CLI 加人。  
3. **SSH**：建议保持密钥登录；口令登录可改为仅密钥（自行评估）。  
4. **端口暴露**：仅 8080（及已有业务端口）；Postgres 5432 若仅容器互联，可在安全组限制源 IP。  
5. **备份加密**：备份目录权限 `700`，必要时异地/对象存储再存一份。  
6. **HTTPS**：有条件时上域名 + 证书，再考虑收紧安全组与 PWA 完整安装。

---

## 7. 发布流程速查（开发者）

```text
本地:
  git checkout main
  git fetch upstream && git merge upstream/main && git push origin main
  git checkout linkyou && git merge main
  # 如需部署文件变更: 改 docker-compose.yml / conf 模板 / 文档
  git add … && git commit && git push origin linkyou

服务器:
  # 同步 compose（或手动 scp）
  cd /opt/ezbookkeeping && docker compose pull && docker compose up -d
  curl -sS -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/
```

---

## 8. 相关文件索引（仓库 `linkyou`）

| 路径 | 作用 |
|------|------|
| `docker-compose.yml` | 生产编排 |
| `conf/ezbookkeeping.ini.example` | 生产配置模板（无密钥） |
| `deploy/README.md` | 部署摘要 |
| `deploy/OPERATIONS.md` | 本手册 |
| `.gitea/workflows/deploy.yml` | 历史残留，**未启用** |

---

## 9. 变更记录

| 日期 | 内容 |
|------|------|
| 2026-11-06 | 初版：Compose 部署、Postgres、安全组说明、PWA 使用、用户 CLI、备份与升级 |
| 2026-11-06 | 创建用户 `yuanguoqi`、`liuzhaoqun`（CNY） |
