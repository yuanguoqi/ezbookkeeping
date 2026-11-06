# 部署说明（linkyou 分支）

面向自有 Ubuntu 服务器、无域名、IP 直连 `http://<服务器IP>:8080/` 的手动部署路径。

## 分支约定

| 分支 | 职责 |
|------|------|
| `main` | 仅同步上游 `https://github.com/mayswind/ezbookkeeping.git` |
| `linkyou` | 本部署文件 + 将来的少量定制 |

同步上游：

```bash
git fetch upstream
git checkout main && git merge upstream/main && git push origin main
git checkout linkyou && git merge main
```

## 仓库内文件

- `docker-compose.yml` — 官方镜像固定 tag `2.1.1`，端口 `8080`
- `conf/ezbookkeeping.ini.example` — 生产配置**模板**（无真实密钥）
- `conf/ezbookkeeping.ini` — 上游默认配置，仅供本地/镜像构建，**不要**写入生产密钥

真实配置只放在服务器：`/opt/ezbookkeeping/conf/ezbookkeeping.ini`（不进 git）。

## 服务器目录

```text
/opt/ezbookkeeping/
├── docker-compose.yml
├── conf/ezbookkeeping.ini    # 真实配置（不进 git）
├── storage/
├── data/
├── log/
└── backup/                   # pg_dump 等
```

## 首次部署要点

1. PostgreSQL 建库（全小写）：`CREATE DATABASE ezbookkeeping;`
2. 应用账号仅授该库权限；`pg_hba.conf` 允许本机与 Docker 网段（如 `172.16.0.0/12`）。
3. 从模板生成真实配置并修改：
   - `security.secret_key` → 随机长字符串
   - `database.passwd` → 数据库口令
   - `domain` / `root_url` → 服务器 IP
   - `user.enable_register = false`（仅自己/家人）
4. 容器通过 `extra_hosts: host.docker.internal:host-gateway` 访问宿主机 Postgres；ini 中 `host = host.docker.internal:5432`。

## 日常发布

```bash
# 本地
git checkout linkyou
# …修改并提交…
git push origin linkyou

# 服务器
cd /opt/ezbookkeeping
git pull   # 或 rsync 同步 compose 等
docker compose pull
docker compose up -d
```

升级镜像版本：改 `docker-compose.yml` 中的 tag → 服务器上重复 `pull && up -d`。

## 备份

- 每日：`pg_dump ezbookkeeping` → `/opt/ezbookkeeping/backup/`
- 同步备份：`storage/`、`conf/ezbookkeeping.ini`（含 `secret_key`，丢失会影响会话与加密相关数据）

## 明确不做（本阶段）

HTTPS / 域名 / 反向代理、CI 自动发布、开放注册、代码层定制。
