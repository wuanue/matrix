# Matrix 通信及其生态服务自建流程

> [!NOTE]
> 搭建环境：NAS + 公网 IP + 域名

## 前言

> [!TIP]
> 
> - Matrix 是一个开放的通信协议，用于实时通信，支持端到端加密，包括即时消息、音频和视频通话。
> - 它是去中心化的，任何人都可以搭建自己的 Matrix 服务器，并与其他服务器联邦互通。
> - Matrix 的目标是为不同的通信服务提供统一标准，通过 Bridge 桥接实现互联互通。

-----

## 目录

- [1. 准备工作](#1-准备工作)
- [2. 部署 pgzhparser 数据库](#2-部署-pgzhparser-数据库)
- [3. 部署 Synapse](#3-部署-synapse)
- [4. 部署辅助服务](#4-部署辅助服务)
- [5. Lucky 反代配置](#5-lucky-反代配置)
- [6. 中文搜索配置](#6-中文搜索配置)
- [7. mautrix-telegram 桥接](#7-mautrix-telegram-桥接)
- [8. Hermes Agent 集成](#8-hermes-agent-集成)
- [9. 性能优化与维护](#9-性能优化与维护)
- [10. 客户端推荐](#10-客户端推荐)
- [11. 推荐组件](#11-推荐组件)

-----

## 1. 准备工作

### 目录结构

```
matrix/
├── synapse/
│   ├── data/              # 配置与数据
│   └── py_config/         # 补丁脚本
├── pgzhparser/
│   └── data/              # 数据库数据
├── element-web/
├── mautrix-telegram/
└── backup/
```

### Docker Compose

```yaml
services:

  synapse:                             # Matrix Synapse homeserver
    image: matrixdotorg/synapse:latest
    container_name: synapse
    restart: unless-stopped
    depends_on:
      - pgzhparser
    environment:
      SYNAPSE_CONFIG_PATH: /data/homeserver.yaml
      UID: 0
      GID: 0
      TZ: Asia/Shanghai
      HTTP_PROXY: http://192.168.50.60:7890    # 代理，用于 Matrix 通知转发
      HTTPS_PROXY: http://192.168.50.60:7890
      NO_PROXY: localhost,127.0.0.1,pgzhparser,mautrix-telegram,172.24.0.0/16
    volumes:
      - /volume1/docker/matrix/synapse/data:/data
      # 中文搜索补丁（初始化完成后按说明放入）
      - /volume1/docker/matrix/synapse/py_config/search.py:/usr/local/lib/python3.13/site-packages/synapse/storage/databases/main/search.py
      # 共享密钥认证模块（用于 Bridge 桥接，按需启用）
      - /volume1/docker/matrix/synapse/py_config/shared_secret_authenticator.py:/usr/local/lib/python3.13/site-packages/shared_secret_authenticator.py
    ports:
      - "8008:8008"    # HTTP
      - "8448:8448"    # Matrix 联邦端口
      - "8413:8413"
    networks:
      matrix_net:
        ipv4_address: 172.24.0.50    # 用于连接自建的 Hermes 机器人，按需设置
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8008/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  synapse-admin:                       # Synapse Web 管理面板
    image: ghcr.io/etkecc/ketesa:latest
    container_name: synapse-admin
    restart: unless-stopped
    ports:
      - "8082:8080"
    networks:
      - matrix_net

  pgzhparser:                          # PostgreSQL + zhparser 中文分词扩展
    image: abcfy2/zhparser:17
    container_name: pgzhparser
    restart: unless-stopped
    environment:
      POSTGRES_USER: user              # 修改为你的数据库用户名
      POSTGRES_PASSWORD: password      # 修改为你的数据库密码
      POSTGRES_DB: postgres
      PGDATA: /var/lib/postgresql/data
      POSTGRES_MAX_CONNECTIONS: 250
    volumes:
      - /volume1/docker/matrix/pgzhparser/data:/var/lib/postgresql/data
    networks:
      - matrix_net

  element-web:
    image: vectorim/element-web:latest
    container_name: element-web
    restart: unless-stopped
    volumes:
      - /volume1/docker/matrix/element-web/config.json:/app/config.json:ro
    ports:
      - "8009:80"
    networks:
      - matrix_net

  mautrix-telegram:                    # 按需启用，需开启共享密钥认证模块
    image: dock.mau.dev/mautrix/telegram:latest
    container_name: mautrix-telegram
    restart: unless-stopped
    volumes:
      - /volume1/docker/matrix/mautrix-telegram:/data
    environment:
      - HTTP_PROXY=代理地址
      - HTTPS_PROXY=代理地址
    networks:
      - matrix_net

networks:
  matrix_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.24.0.0/16
```

-----

## 2. 部署 pgzhparser 数据库

### 连接数据库

```bash
docker exec -it pgzhparser bash
psql -U your_db_user -d postgres
```

### 建立数据库并启用中文分词

```sql
CREATE DATABASE synapsedb
    WITH OWNER = your_db_user
    ENCODING = 'UTF8'
    LC_COLLATE = 'C'
    LC_CTYPE = 'C'
    TEMPLATE = template0;

\c synapsedb

CREATE EXTENSION IF NOT EXISTS zhparser;
CREATE TEXT SEARCH CONFIGURATION chinese (PARSER = zhparser);
ALTER TEXT SEARCH CONFIGURATION chinese ADD MAPPING FOR n,v,a,i,e,l WITH simple;
\q
```

-----

## 3. 部署 Synapse

### 生成配置文件

```bash
docker run -it --rm \
    -v /volume1/docker/matrix/synapse/data:/data \
    -e SYNAPSE_SERVER_NAME=matrix.your-domain.com \
    -e SYNAPSE_REPORT_STATS=no \
    matrixdotorg/synapse:latest generate
```

### 修改 homeserver.yaml

**数据库配置**（替换默认 SQLite）：

```yaml
database:
  name: psycopg2
  args:
    user: your_db_user
    password: your_db_password
    database: synapsedb
    host: pgzhparser
    cp_min: 10
    cp_max: 20
```

**关闭 TLS**（由反代处理）：

> [!IMPORTANT]
> 本文使用 Lucky 做反代，Synapse 本身无需处理证书，故关闭 TLS。

```yaml
tls_certificate_path: null
tls_private_key_path: null
```

**监听配置**：

```yaml
listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true
    resources:
      - names: [client, federation]
        compress: false
```

**注册配置**：

```yaml
enable_registration: true           # 是否开启公开注册
registration_requires_token: true   # 是否需要邀请码
report_stats: false                 # 是否开启匿名统计
```

**上传限制**：

```yaml
max_upload_size: 500M
```

**缓存优化**：

```yaml
caches:
  global_factor: 2.0
  per_cache_factors:
    get_users_in_room: 3.0
```

**shared_secret_authenticator 模块**（Bridge 桥接所需）：

```yaml
modules:
  - module: shared_secret_authenticator.SharedSecretAuthProvider
    config:
      shared_secret: "your_shared_secret"
      m_login_password_support_enabled: true
```

**appservice 注册**：

```yaml
app_service_config_files:
  - /data/telegram-registration.yaml
```

### 创建管理员账号

```bash
docker exec -it synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml
```

-----

## 4. 部署辅助服务

### Element Web 配置

```bash
mkdir -p /volume1/docker/matrix/element-web
nano /volume1/docker/matrix/element-web/config.json
```

```json
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://matrix.your-domain.com:8118",
            "server_name": "matrix.your-domain.com"
        }
    },
    "brand": "Element",
    "default_theme": "dark",
    "integrations_ui_url": "",
    "integrations_rest_url": "",
    "integrations_widgets_urls": []
}
```

-----

## 5. Lucky 反代配置

### 需要反代的服务

|域名                            |后端地址                |
|------------------------------|--------------------|
|`matrix.your-domain.com`      |`http://NAS_IP:8008`|
|`element.your-domain.com`     |`http://NAS_IP:8009`|
|`registration.your-domain.com`|`http://NAS_IP:5000`|

### .well-known 配置（文本输出）

`/.well-known/matrix/client`：

```json
{"m.homeserver":{"base_url":"https://matrix.your-domain.com:8118"},"org.matrix.msc4143.rtc_foci":[{"type":"livekit","livekit_service_url":"https://jwt.your-domain.com:8118"}]}
```

`/.well-known/matrix/server`：

```json
{"m.server":"matrix.your-domain.com:8118"}
```

-----

## 6. 中文搜索配置

### 修改 search.py

从容器复制原始文件：

```bash
docker cp synapse:/usr/local/lib/python3.13/site-packages/synapse/storage/databases/main/search.py \
  /volume1/docker/matrix/synapse/py_config/search.py
```

修改 INSERT 语句，加入 `chinese_vector` 字段：

```bash
sed -i 's/(event_id, room_id, key, vector, stream_ordering, origin_server_ts)/(event_id, room_id, key, vector, chinese_vector, stream_ordering, origin_server_ts)/' \
  /volume1/docker/matrix/synapse/py_config/search.py

sed -i "s/VALUES (?,?,?,to_tsvector('chinese', ?),?,?)/VALUES (?,?,?,to_tsvector('chinese', ?),to_tsvector('chinese', ?),?,?)/" \
  /volume1/docker/matrix/synapse/py_config/search.py
```

修改搜索查询使用 `chinese_vector`：

```bash
sed -i 's/ts_rank_cd(vector,/ts_rank_cd(chinese_vector,/g' \
  /volume1/docker/matrix/synapse/py_config/search.py

sed -i 's/WHERE vector @@/WHERE chinese_vector @@/g' \
  /volume1/docker/matrix/synapse/py_config/search.py
```

> [!TIP]
> 最后还需在 nano 里手动在 args 元组中添加一行 `_clean_value_for_search(entry.value),`。

### 初始化数据库中文索引

```sql
ALTER TABLE event_search ADD COLUMN IF NOT EXISTS chinese_vector tsvector;

UPDATE event_search
SET chinese_vector =
    CASE
        WHEN event_search.key = 'content.body'
            AND TRIM(ej.json::jsonb->'content'->>'body') != ''
            THEN to_tsvector('chinese', ej.json::jsonb->'content'->>'body')
        WHEN event_search.key = 'content.name'
            AND TRIM(ej.json::jsonb->'content'->>'name') != ''
            THEN to_tsvector('chinese', ej.json::jsonb->'content'->>'name')
        WHEN event_search.key = 'content.topic'
            AND TRIM(ej.json::jsonb->'content'->>'topic') != ''
            THEN to_tsvector('chinese', ej.json::jsonb->'content'->>'topic')
        ELSE NULL
    END
FROM event_json ej
WHERE event_search.event_id = ej.event_id
  AND event_search.chinese_vector IS NULL;

CREATE INDEX CONCURRENTLY event_search_chinese_vector_idx
  ON event_search USING GIN (chinese_vector);
```

-----

## 7. mautrix-telegram 桥接

### 申请 Telegram API

访问 <https://my.telegram.org>，登录后申请 App API，获取 `api_id` 和 `api_hash`。

### 启用共享密钥模块

下载 `shared_secret_authenticator.py`：

```bash
wget -O /volume1/docker/matrix/synapse/py_config/shared_secret_authenticator.py \
  https://raw.githubusercontent.com/devture/matrix-synapse-shared-secret-auth/master/shared_secret_authenticator.py
```

在 `homeserver.yaml` 中填入：

```yaml
modules:
  - module: shared_secret_authenticator.SharedSecretAuthProvider
    config:
      shared_secret: "your_shared_secret"    # 与 registration_shared_secret 一致
      m_login_password_support_enabled: true
```

### 关键配置（config.yaml）

```yaml
homeserver:
    address: http://synapse:8008
    domain: matrix.your-domain.com

appservice:
    address: http://mautrix-telegram:29317
    hostname: 0.0.0.0
    port: 29317

network:
    api_id: your_api_id      # 在 my.telegram.org 申请
    api_hash: your_api_hash
    proxy:
        type: socks5
        address: "NAS_IP:socks_port"

database:
    type: sqlite3-fk-wal
    uri: file:/data/mautrix-telegram.db?_txlock=immediate

bridge:
    permissions:
        "*": relay
        "your-domain.com": user
        "@admin:your-domain.com": admin
    relay:
        enabled: false
    cleanup_on_logout:
        enabled: true
        manual:
            private: kick
            relayed: kick
            shared_no_users: kick
            shared_has_users: kick

encryption:
    allow: true
    default: false

double_puppet:    # 需登录后在 Element 机器人聊天框发送 login-matrix 开启
    servers: {}
    secrets:
        your-domain.com: as_token:your_as_token

logging:
    min_level: warn
```

### 优化配置（config.yaml）

```yaml
animated_sticker:          # 动画贴纸格式（改为 webp 更清晰）
    target: webp

sync:                      # 同步对话数量（默认仅 15 个）
    update_limit: 100
    create_limit: 50

matrix:                    # 消息回执（Telegram 已读同步到 Matrix）
    delivery_receipts: true

backfill:                  # 消息回填（登录后自动拉取历史消息）
    enabled: true
    max_initial_messages: 50
```

### 注册 Appservice

```bash
cd /volume1/docker/matrix
docker compose run --rm mautrix-telegram

# 将生成的 registration.yaml 复制到 Synapse 数据目录
cp /volume1/docker/matrix/mautrix-telegram/registration.yaml \
   /volume1/docker/matrix/synapse/data/telegram-registration.yaml
```

### 修改 registration.yaml（允许 double puppet）

```yaml
namespaces:
    users:
        - regex: ^@telegrambot:your-domain\.com$
          exclusive: true
        - regex: ^@telegram_.*:your-domain\.com$
          exclusive: true
        - regex: ^@.*:your-domain\.com$
          exclusive: false
```

### 登录 Telegram

在 Element 找到 `@telegrambot:your-domain.com`，发送：

```
login phone +86手机号
```

-----

## 8. Hermes Agent 集成

### 创建机器人账号

```bash
# 用户名: hermes，非管理员
docker exec -it synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml
```

### 获取访问令牌

```bash
curl -X POST http://localhost:8008/_matrix/client/v3/login \
  -H "Content-Type: application/json" \
  -d '{
    "type": "m.login.password",
    "user": "@hermes:your-domain.com",
    "password": "your_password"
  }'
```

### 网络配置

Hermes 的 compose 文件中加入 `matrix_net` 外部网络：

```yaml
networks:
  default:
    name: hermes-net
  matrix_matrix_net:
    external: true
```

`hermes-gateway` 服务加入两个网络，并配置 NO_PROXY：

```yaml
services:
  hermes-gateway:
    networks:
      - default
      - matrix_matrix_net
    environment:
      - NO_PROXY=localhost,127.0.0.1,hermes-gateway,hermes-dashboard,synapse,172.24.0.0/16
```

同时在 `config.yaml` 的 network 部分：

```yaml
network:
  no_proxy: localhost,127.0.0.1,hermes-gateway,hermes-dashboard,192.168.50.0/24,172.24.0.0/16
```

### 交互式配置

```bash
docker exec -it hermes-gateway hermes-agent gateway setup
```

按提示填写：

|字段            |值                                         |
|--------------|------------------------------------------|
|Homeserver URL|`http://synapse:8008` 或 `http://固定IP:8008`|
|Access token  |hermes 账号的访问令牌                            |
|Allowed users |`@user1:your-domain.com`（留空允许所有人）         |
|E2EE          |`N`（暂不支持）                                 |

-----

## 9. 性能优化与维护

### 自动备份脚本

创建 `/volume1/docker/matrix/backup.sh`：

```bash
#!/bin/bash
BACKUP_DIR="/volume1/docker/matrix/backup"
DATE=$(date +%Y%m%d_%H%M%S)
mkdir -p $BACKUP_DIR

docker exec pgzhparser pg_dump -U your_db_user synapsedb \
  | gzip > $BACKUP_DIR/synapsedb_$DATE.sql.gz

find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete

echo "备份完成: $BACKUP_DIR/synapsedb_$DATE.sql.gz"
```

设置权限并添加定时任务：

```bash
chmod +x /volume1/docker/matrix/backup.sh
crontab -e
# 添加: 0 3 * * * /volume1/docker/matrix/backup.sh
```

### 媒体文件清理（homeserver.yaml）

```yaml
media_retention:
  local_media_lifetime: 90d
  remote_media_lifetime: 14d
```

### Synapse-admin（Ketesa）

仅限内网访问，地址：`http://NAS_IP:8082`

登录时填写：

- **Homeserver URL**：`https://matrix.your-domain.com`
- 用户名 + 密码登录，无需访问令牌

-----

## 10. 客户端推荐

|客户端                                               |平台           |说明                                                                                                       |
|--------------------------------------------------|-------------|---------------------------------------------------------------------------------------------------------|
|[Element / Element X](https://element.io/download)|全平台          |最流行的客户端，支持线程、视频会议、消息置顶。Element X 额外支持滑动同步、二维码登录等 Matrix 2.0 功能                                           |
|[SchildiChat](https://schildi.chat/)              |Android / Web|Element 的扩展分支，功能特性见 [FEATURES.md](https://github.com/SchildiChat/SchildiChat-android/blob/sc/FEATURES.md)|
|[NerChat](https://www.neboer.site/nerchat)        |Web / Android|Element 分支，为国内用户做了连接优化                                                                                   |
|[Cinny](https://cinny.in/)                        |PC / Web     |UI 简洁清爽，专注桌面体验                                                                                           |

-----

## 11. 推荐组件

|组件                                                                                          |说明                                                                        |
|--------------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
|[synapse-admin (Ketesa)](https://github.com/etkecc/ketesa)                                  |Synapse 管理面板，集成常用 Admin API                                               |
|[coturn](https://github.com/coturn/coturn)                                                  |1:1 通话的 TURN 服务器，需公网 IP；Matrix 2.0 后由 MatrixRTC 取代                        |
|[element-call](https://github.com/element-hq/element-call)                                  |MatrixRTC 的实现，Element X 上将取代 coturn 和 Jitsi                               |
|[baibot](https://github.com/etkecc/baibot)                                                  |Matrix 上流行的 AI 机器人，支持文本、语音、图片生成                                           |
|[Maubot](https://github.com/maubot/maubot)                                                  |机器人插件平台，支持 RSS、Webhook 等，插件列表见 [plugins.mau.bot](https://plugins.mau.bot/)|
|[mautrix-telegram](https://github.com/element-hq/mautrix-telegram)                          |桥接 Telegram，同类还有桥接 Twitter 等                                              |
|[postmoogle](https://github.com/etkecc/postmoogle)                                          |将 Matrix 作为电子邮局，在 Matrix 上收发邮件                                            |
|[matrix-authentication-service](https://github.com/element-hq/matrix-authentication-service)|MAS 身份验证，支持二维码登录、第三方社交账号和 SSO                                             |