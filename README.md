# 🦞 OpenClaw 部署与实战全攻略

## 1. 核心架构设计
本项目基于 **OpenClaw (v2026.3.*)**，采用容器化部署，通过 Mac 宿主机链路连接多个 API Provider。

### 1.1 双 Agent 架构
| Agent | 主模型 | 备选模型 | 特性 |
|-------|--------|---------|------|
| **claude-agent** (默认) | `claude-opus-4-6` | `claude-sonnet-4-20250514` → `claude-haiku-4-5-20251001` | 推理能力强，上下文 200K |
| **flash** | `gpt-4o` | `gpt-3.5` | 响应快速，支持 Reasoning |

### 1.2 API 提供商
* **Anthropic Claude** - 官方 API (`https://api.anthropic.com`)
* **Custom Flash Provider** - 本地反代 (`http://host.docker.internal:14725/v1`)

---

## 2. 生产环境全量配置 (Raw Mode JSON)
**关键提醒**：容器默认路径必须锁定为 `/home/node/`。

### 2.1 直接克隆项目

    # 通过 git 命令克隆项目源代码
    git clone https://github.com/openclaw/openclaw.git

### 2.2 项目docker镜像编译打包

    # 构建基础镜像并命名为 openclaw:local
    docker build -t openclaw:local .  

### 2.3 编辑 .env 环境变量文件并使用 docker-compose 文件

    2.3.1> 首先编辑 .env 文件， 需要填写 OPENCLAW_GATEWAY_TOKEN 与 TELEGRAM_BOT_TOKEN

    2.3.2> docker-compose.yml 一般不需要编辑，如何配置好 .env 文件后可以直接使用

    2.3.3> 启动服务，分别是 openclaw-gateway 与 openclaw-cli， openclaw-gateway服务会正常运行挂起，openclaw-cli服务只是一个客户端工具，会产生这个服务不会挂起(处于离线状态， 你把它理解成即用即调的工具即可， 这是正常的)

```bash
docker compose up -d
```

### 2.4 请开启你的反代理软件 Antigravity-Tool (已弃用该 2。4 的方案, 请使用下面的 claude.ai 的订阅方案)

    2.4.1> 在账号管理里请自己设置和绑定你的账号（省略操作，如果不会请自己查询资料）

    2.4.2> 点击 API 反代设置

        1> 设置监听端口，默认为 8045， 如果冲突可以自己随意修改端口
        2> 点击启动服务，开启反向代理服务

    2.4.3> 容器网络与连通性验证 (First Ping), 需要确认在docker容器内是否可以访问到到 Antigravity-Tool 的反代理工具
```bash
docker compose exec openclaw-gateway curl -v http://host.docker.internal:14725/v1/models
```

### 2.5 服务启动后的首次初始化配置命令

    2.5.1> 身份配对与授权 (Telegram Pairing)

    当你第一次在 Telegram 给机器人发消息时，系统会拦截并要求授权：在 TG 聊天框获取 Pairing Code（例如：123-456）。

    在 Mac 终端 执行批准命令, 目的：将你的 TG 账号设为该 Agent 的唯一受信任所有者。
```bash
docker compose exec openclaw-gateway openclaw pairing approve telegram [你的配对码]
```

    2.5.2> 文件系统权限初始化 (Permission Fix), 因为 Docker 内部用户是 node，需要手动修正挂载目录权限, 解决 EACCES: permission denied 导致的无法创建文件夹问题。
```bash
docker compose exec -u root openclaw-gateway chown -R node:node /home/node/.openclaw
```

    2.5.3> 记忆系统索引初始化 (Memory Init)
```bash
docker compose exec openclaw-gateway touch /home/node/.openclaw/workspace/MEMORY.md
```

    2.5.4> 核心配置项目设置，直接使用项目根目录下的 `openclaw-anthropic.json` 文件

    在 OpenClaw 控制面板的 Configuration -> Raw 模式下，复制 `openclaw-anthropic.json` 的全部内容。

    **⚠️ 请修改以下关键参数**：
    - `models.providers.flash.apiKey` - 从 Antigravity-Tool 反代工具获取（格式：`sk-xxx`）
    - `models.providers.claude.apiKey` - 从 [claude.ai](https://claude.ai) 购买订阅后生成
    - `channels.telegram.botToken` - Telegram 机器人令牌（可从 [@BotFather](https://t.me/botfather) 获取）
    - `gateway.auth.token` - OpenClaw 网页登录令牌（控制面板 Settings 获取）

    2.5.5> 健康检查与审计 (Doctor & Audit), 最后，我们通过这两个命令确认系统已经”健康”
```bash
# 检查 Agent 是否配置了 Key
docker compose exec openclaw-gateway openclaw doctor

# 深度审计安全隐患（处理 allowedOrigins 风险）
docker compose exec openclaw-gateway openclaw security audit --deep
```

## 3.配置 claude.ai 订阅

### 3.1 任何上网的机器电脑上安装 claude 命令行

    # 首先在网页上登录 claude.ai, 在上面的 Upgrade 里购买套餐，例如 $17 或 $100

    # 使用命令行(这里会提示使用网页进行授权, 注意，生成出token以后需要在编辑器里把生产的token的内容可能出现截断部分的, 如果发现请自行调整)
    claude setup-token

    # 在 openclaw 中执行（本操作是在宿主机上操作， 容器名称叫 openclaw-gateway）
    docker exec -it openclaw-gateway openclaw models auth paste-token --provider anthropic

    # 请使用 openclaw-anthropic.json 配置修改你的 oepnclaw.json 内容，然后请重新加载配置，直接再次执行 docker compose up -d 就可以了

## 4.配置 clawhub 相关

### 4.1> 基础设置

    # 使用淘宝镜像源以确保安装成功，不受海外 API 频率限制
    npm config set registry https://registry.npmmirror.com

    # 全局安装 clawhub 0.7.0
    npm install -g clawhub@0.7.0

### 4.2> 登录 clawhub

    # 打开网址，若无账号请先授权登录
    https://clawhub.ai/settings

    # 创建 api-token
    找打 API tokens -> create token 按钮，点击后生成 token 后进行复制

    # 在容器内执行
    clawhub auth set-token {你复制的token,请替换...}

### 4.3 安装 Skill

#### 4.3.1 必装 ✅：skill-vetter（安全检查工具）

**用途**：检查以后添加的 skill 是否存在安全隐患

基础安装：
```bash
clawhub install skill-vetter
```

强制重新安装最新版本：
```bash
clawhub install skill-vetter --force
```

---

#### 4.3.2 其他 Skill（按需安装）

根据实际需求安装其他 skill，安装命令格式：
```bash
clawhub install <skill-name>
```

常用选项：
- `--force` - 强制重新安装最新版本
- `--registry` - 指定自定义仓库地址
