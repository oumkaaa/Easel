# 内网部署交接清单

这份文档给负责把 Easel 部署到公司内网服务器的同学看。目标：一台内网机器，团队共用一份，同事通过公司内网/VPN 访问。

## 这是什么

Easel 是一个社媒内容工作台（Web + OpenClaw Agent），仓库见本仓库根目录 README。这个 fork（`oumkaaa/Easel`）在上游基础上加了两处本地功能，见 [CHANGELOG.md](../CHANGELOG.md) 的 `[Unreleased]` 部分。

## 机器要求

| 项 | 要求 |
|---|---|
| 系统 | Linux（Ubuntu 22.04+ / CentOS 均可），不支持 Windows Server |
| 配置 | 最低 2 核 4G，建议 4 核 8G（跑浏览器自动化 + 媒体处理，内存别太紧） |
| 磁盘 | ≥ 40G |
| 网络（出站） | 必须能连到 `ada-cli-golang.ctripcorp.com`（内部 LLM 网关，走公司内网即可，不需要公网） |
| 网络（入站） | 同事需要能访问这台机器的 7860 端口（内网段或 VPN 内可达即可，**不要**把 7860 直接暴露公网） |
| SSH | 部署过程需要 root 或 sudo（装 Node/FFmpeg/Playwright 依赖） |

## 部署步骤

```bash
git clone https://github.com/oumkaaa/Easel.git
cd Easel
bash setup.sh
```

`setup.sh` 是幂等的引导安装器，会自动检查/装 Python 3.10+、Node.js 22.19+、FFmpeg、Playwright Chromium。终端里会问几个问题，直接跟着走。

**注意**：如果系统自带的 `python3` < 3.10（很多发行版默认 3.8/3.9），脚本会直接报错退出。装一个 3.10+（`apt install python3.10` 或用 `pyenv`），让它在 PATH 里能找到再重跑 `setup.sh`。

### LLM 接入：不要用我的 Key，申请你自己的

安装过程会问模型怎么接，选 **"其他 Anthropic 兼容服务" 或 "OpenAI 兼容服务"**，指向公司内部 ADA 网关：

```bash
# .env 里配置（setup.sh 会引导你填，或装完手动改 .env）
OPENAI_API_KEY=<你自己的 ADA_KEY，不要用别人的>
OPENAI_BASE_URL=http://ada-cli-golang.ctripcorp.com/coding-plan/openai/v1
OPENAI_MODEL=qwen3.7-plus
```

⚠️ **两个坑，我们踩过、帮你省时间：**

1. **模型名字**：ADA 网关的 `ADA_MODELS` 环境变量里列的 `claude-3-5-sonnet` / `gpt-4o` / `claude-3-opus` / `o3-mini` 目前**实测全部被网关拒绝**（`This model is not currently supported`），只有 `qwen3.7-plus` 验证可用。部署时先拿这个测，能用再考虑要不要跟平台方申请别的模型白名单。
2. **Qwen 默认开着"思考模式"，巨慢**：不处理的话每次 LLM 调用都会先吐一大段可见的 chain-of-thought，几个工具调用叠起来一次对话能卡到几十秒。修法是把这个模型注册成 OpenClaw 的 `vllm` 兼容 provider（即使实际不是 vllm，这只是为了触发它内置的 Qwen 关思考逻辑），并设 `compat.thinkingFormat: "qwen"`：

   ```bash
   source .venv/bin/activate
   openclaw --profile easel config set models.providers.vllm \
     '{"api":"openai-completions","apiKey":"<你的ADA_KEY>","baseUrl":"http://ada-cli-golang.ctripcorp.com/coding-plan/openai/v1","models":[{"id":"qwen3.7-plus","name":"Qwen3.7 Plus (ADA)","reasoning":true,"input":["text","image"],"compat":{"thinkingFormat":"qwen"}}]}'
   openclaw --profile easel config set agents.defaults.model.primary "vllm/qwen3.7-plus"
   openclaw --profile easel config unset models.providers.openai   # 清掉 setup.sh 生成的慢速默认配置
   openclaw --profile easel gateway restart
   ```

   效果：单次模型请求从平均 2-4 秒（偶尔到 10 秒）降到稳定 1.1-1.3 秒。

## 常驻运行（systemd）

本地 Mac 上我们用的是 launchd，服务器上换成 systemd。两个服务：`easel-gateway`（OpenClaw 网关，`openclaw gateway install` 装完会自动生成对应发行版的服务定义，Linux 下就是走 systemd，直接跑这条就行）和 `easel-web`（Easel 自己的 Web 层，需要手写 unit）。

```bash
sudo tee /etc/systemd/system/easel-web.service <<'EOF'
[Unit]
Description=Easel Web
After=network.target

[Service]
Type=simple
User=<部署用的系统账号>
WorkingDirectory=/opt/easel   # 换成实际克隆路径
ExecStart=/opt/easel/.venv/bin/python -m easel web --port 7860
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now easel-web
```

Gateway 那边：

```bash
source .venv/bin/activate
openclaw --profile easel gateway install
```

## 共享访问：加一道密码门

团队共用同一份数据（同一批平台登录态、同一份内容库），所以只需要在 Web 服务前面挡一道 **HTTP Basic Auth**，防止内网里不相关的人也能进来操作（尤其是能直接发布到已登录的社媒账号这一点，权限不轻）。用 Caddy 最省事：

```
# /etc/caddy/Caddyfile
easel.internal.company.com {
    basic_auth {
        <账号名> <bcrypt 密码哈希，用 caddy hash-password 生成>
    }
    reverse_proxy localhost:7860
}
```

或者用 nginx + `htpasswd`，效果一样。**18789（OpenClaw 网关端口）不要对外暴露**，只反代 7860。

## 验证部署成功

```bash
source .venv/bin/activate
easel doctor   # 环境检查，绿色为主，"Skills synced" 报红是已知的路径检查 bug，不影响功能
easel ping     # 连通性 + 真实 LLM 调用测试，两步都要 OK
```

浏览器打开 `http://<机器内网地址>:7860`（或配好的域名），出现工作台首页即成功。

## 交给部署同学的东西

- 这份文档的链接（仓库里，随时能看最新版）
- 仓库地址：`https://github.com/oumkaaa/Easel`（公开仓库，直接 clone，不需要邀请）
- **不要**把你自己的 `.env` / ADA_KEY 发给他——让他用自己的 ADA Key，或者找平台方申请一个团队共用的服务账号 Key（更规范，权限和账单都独立于个人）
- 共享密码门的账号密码，交接后自己拟一个，不要用你私人密码
