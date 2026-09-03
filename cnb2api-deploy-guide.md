# CNB2API 服务器完整部署文档

## 一、服务器环境

- **操作系统**：Ubuntu 24.04.4 LTS (Noble Numbat)
- **Docker 版本**：29.7.2
- **反向代理**：Caddy（端口 80）
- **服务器 IP**：124.220.1.243

---

## 二、运行中的服务

| 容器名称 | 镜像 | 端口映射 | 用途 |
|----------|------|----------|------|
| cnb2api-gateway | cnb2api-cnb2api:latest | 7863:7863 | CNB API 反向代理网关 |
| cnb2api-toolforge | cnb2api-toolforge:latest | 18080:8080 | 前置中间件（上下文压缩/协议转换） |

**架构流程：**
```
客户端 -> ToolForge(:18080) -> cnb2api-gateway(:7863) -> CNB API
```

---

## 三、项目文件结构

```
/tmp/cnb2api/                          # 主项目目录
├── docker/
│   ├── config.yaml                     # ToolForge 配置文件
│   └── toolforge/                      # ToolForge 子模块
│       └── app/
│           ├── engine/orchestrator.py  # 核心编排逻辑
│           ├── stream/openai_sse.py    # OpenAI SSE 流式处理
│           └── fc/policy.py            # FC 模式策略
├── config.example.json                 # cnb2api-gateway 配置模板
├── docker-compose.yml                  # Docker Compose 编排文件
├── Dockerfile.cnb2api                  # cnb2api-gateway 构建文件
└── go.mod                              # Go 模块定义

/etc/caddy/Caddyfile                    # Caddy 反向代理配置
/usr/share/caddy/                       # Caddy 静态文件目录
```

---

## 四、配置文件详解

### 4.1 ToolForge 配置（/tmp/cnb2api/docker/config.yaml）

```yaml
server:
  host: "0.0.0.0"
  port: 8080
  timeout_seconds: 180

upstreams:
  - name: cnb2api
    type: openai_compat
    base_url: "http://cnb2api:7863/v1"
    api_key: "***"
    models: ["deepseek-v4-flash", "deepseek-v4-pro", "*"]
    native_fc: false
    is_default: true

features:
  fc_mode: prompt              # 重要：必须为 prompt
  enable_streaming: true
  enable_fc_error_retry: true
  fc_error_retry_max: 2
  strip_think_tags: true
  inject_protocol: XYML
  convert_developer_to_system: true
```

### 4.2 cnb2api-gateway 配置（/tmp/cnb2api/config.example.json）

```json
{
  "listen": ":7863",
  "api_key": "***",
  "model": "deepseek-v4-flash",
  "models": ["deepseek-v4-flash", "deepseek-v4-pro"],
  "pool_min": 2,
  "pool_max": 32,
  "ttl_minutes": 30
}
```

### 4.3 Caddy 配置（/etc/caddy/Caddyfile）

```
:80 {
    root * /usr/share/caddy
    file_server
}
```

---

## 五、完整部署步骤

### 步骤1：安装 Docker

```bash
# 安装 Docker
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable docker
sudo systemctl start docker

# 验证安装
sudo docker version
```

### 步骤2：安装 Caddy

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install caddy

# 配置 Caddy（静态文件服务）
sudo tee /etc/caddy/Caddyfile << 'EOF'
:80 {
    root * /usr/share/caddy
    file_server
}
EOF
sudo systemctl reload caddy
```

### 步骤3：克隆项目代码

```bash
# 克隆主项目
git clone https://github.com/miaopasixx/cnb2api.git /tmp/cnb2api
cd /tmp/cnb2api

# 初始化子模块
git submodule update --init --recursive
```

### 步骤4：配置 API 密钥

```bash
# 编辑 ToolForge 配置
vim /tmp/cnb2api/docker/config.yaml
# 修改 upstreams.api_key 为你的 CNB API 密钥

# 编辑 Gateway 配置
vim /tmp/cnb2api/config.example.json
# 修改 api_key 为你的 CNB API 密钥
```

### 步骤5：构建并启动服务

```bash
cd /tmp/cnb2api

# 构建镜像
docker compose build

# 启动服务
docker compose up -d

# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f
```

### 步骤6：验证服务

```bash
# 检查 ToolForge 健康状态
curl http://localhost:18080/healthz

# 检查 Gateway 健康状态
curl http://localhost:7863/healthz

# 查看可用模型
curl http://localhost:18080/v1/models

# 测试聊天请求
curl http://localhost:18080/v1/chat/completions   -H "Content-Type: application/json"   -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"Hello"}]}'
```

---

## 六、已实施的修复

### 6.1 XYML/DSML 标签泄露修复

**问题：** 模型响应中出现未解析的 XYML/DSML 标签泄露到客户端

**修复文件清单：**

| 文件 | 修改内容 |
|------|----------|
| config.yaml | fc_mode 改为 prompt |
| fc/policy.py | 禁用 passthrough 分支 |
| engine/orchestrator.py | 添加正则清理逻辑 |
| stream/openai_sse.py | 添加流式过滤函数 |

**最终正则表达式：**
```python
pattern = r'</?[|｜] ?[^>]*(XYML|DSML)[^>]*>'
result = re.sub(pattern, '', text, flags=re.IGNORECASE)
```

### 6.2 修复部署方式

由于容器使用 bind mount 挂载配置文件（只读），修改代码需要：

```bash
# 1. 从容器复制文件到宿主机
sudo docker cp cnb2api-toolforge:/app/app/engine/orchestrator.py /home/ubuntu/orchestrator.py

# 2. 修改文件
vim /home/ubuntu/orchestrator.py

# 3. 复制回容器
sudo docker cp /home/ubuntu/orchestrator.py cnb2api-toolforge:/app/app/engine/orchestrator.py

# 4. 重启容器
sudo docker restart cnb2api-toolforge
```

---

## 七、常见问题与解决方案

### 7.1 fc_mode 配置不生效

**原因：** 配置值与代码中判断的值不匹配
- 配置文件注释说有效值是 `force_prompt`
- 但代码中判断的是 `prompt`

**解决：** 将 fc_mode 改为 `prompt`

### 7.2 非流式请求标签泄露

**原因：** 无工具定义时走 passthrough 透传模式

**解决：** 修改 fc/policy.py，禁用 passthrough 分支

### 7.3 正则过滤无效

**原因：** 正则表达式要求两个连续分隔符，但实际标签只有一个

**解决：** 修正正则表达式，去掉多余的分隔符要求

### 7.4 关闭标签泄露

**原因：** 正则不支持 `</` 前缀的关闭标签

**解决：** 添加 `/?` 匹配可选的斜杠

### 7.5 混合格式标签泄露

**原因：** DSML 和 XYML 标签嵌套，协议名不在固定位置

**解决：** 使用更宽松的正则 `[^>]*(XYML|DSML)[^>]*`

### 7.6 GitHub 推送卡住

**原因：** HTTPS 连接被 TLS 中断

**解决：** 改用 SSH 方式推送
```bash
git remote set-url origin git@github.com:miaopasixx/cnb2api-toolforge.git
```

---

## 八、GitHub 仓库

| 仓库 | 地址 | 用途 |
|------|------|------|
| cnb2api | https://github.com/miaopasixx/cnb2api | 主项目代码 |
| cnb2api-toolforge | https://github.com/miaopasixx/cnb2api-toolforge | 中间件子模块 |
| cnb2api-configs | https://github.com/miaopasixx/cnb2api-configs | 配置备份与文档 |

---

## 九、日常维护命令

```bash
# 查看容器状态
sudo docker ps

# 查看容器日志
sudo docker logs -f cnb2api-toolforge
sudo docker logs -f cnb2api-gateway

# 重启服务
sudo docker restart cnb2api-toolforge
sudo docker restart cnb2api-gateway

# 更新镜像
cd /tmp/cnb2api
git pull
docker compose build
docker compose up -d

# 备份配置
cd /tmp/cnb2api-configs
git add .
git commit -m "backup: 配置备份"
git push
```
