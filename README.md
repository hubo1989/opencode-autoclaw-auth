# opencode-autoclaw-auth

[![CI](https://github.com/hubo1989/opencode-autoclaw-auth/actions/workflows/ci.yml/badge.svg)](https://github.com/hubo1989/opencode-autoclaw-auth/actions/workflows/ci.yml)

[OpenCode](https://opencode.ai) / [magpie](https://usemagpie.ai) provider 插件：**AutoClaw（智谱澳龙 / OpenClaw 云端账号体系）**。

一个包注册两个供应商，按账号所在地分开登录、分开记额度：

| 供应商 id | 登录方式 | 上游 host | 模型引用 |
|---|---|---|---|
| `autoclaw` | Google / z.ai 网页 OAuth | `autoglm-api.autoglm.ai` | `autoclaw/zai_auto` 等 |
| `autoclaw-cn` | 手机验证码（+86） | `autoglm-api.zhipuai.cn` | `autoclaw-cn/zai_auto` 等 |

两个供应商都支持 magpie 的多账号、故障切换、token 自动轮换与积分余额显示。模型为带前缀的 routeModelId（`zai_` 智谱 / `zaicoding_` coding plan / `tdpsk_` 豆包 DeepSeek）：

- `zai_auto` / `zai_auto-fast`（自动路由，推荐默认）
- `zai_glm-5.3-flash` / `zai_glm-5-turbo`
- `zaicoding_glm-5.3`
- `tdpsk_deepseek-v4-flash-202605` / `tdpsk_deepseek-v4-pro-202606`

## 安装

```sh
magpie plugin add ./magpie-autoclaw-auth     # 本地文件夹（或 .tgz / npm 包名 / github:hubo1989/opencode-autoclaw-auth）
magpie plugin login autoclaw                   # Google / z.ai 网页登录
magpie plugin login autoclaw-cn                # 手机验证码
magpie provider test autoclaw zai_auto
magpie quota                                   # 积分余额
```

OpenCode 同样可加载（`opencode auth login`）。登录信息保存在 magpie 的 `plugin-auth.json`（600 权限）/ OpenCode 的 `auth.json`。

## 直连说明（不经本地网关）

插件**直连上游云 API**，登录/对话/刷新/积分全部自行完成，**不依赖 workbuddy2api / opencodex 面板**：

- 对话走 `POST {host}/autoclaw-proxy/proxy/autoclaw/v1/chat/completions`（`X-Authorization` 鉴权）；
- token 过期前自动轮换（refresh token 一次一换，`auth.refresh` 钩子交给 magpie 串行化，不会双花）；
- 积分余额通过 `auth.usage` 显示（纯积分制，无总量上限，`aside` 窗口展示余额）；
- **国际版登录是完整本地实现**：插件在 127.0.0.1 上取官方 TokenServer 端口池中的一个端口
  （18432/19654/19723/53699，OAuth client 回调白名单；被占自动顺延）起临时服务器，内嵌引导页完成
  阿里滑块验证（`AliyunCaptcha` SDK，`ali_captcha_verify_param` 回执）→ 获取授权页 →
  自动跳转 Google/z.ai → 回调自动捕获 code，全程浏览器完成、无需粘贴；登录结束服务器即关。

## 每日签到

`magpie quota` 的额度窗口里会显示**每日签到**状态（今日已签 ✓ / 今日未签 +N）。**自动签到默认开启**：每次查额度（`magpie quota`、GUI 刷新）发现未签就自动补签（服务端幂等，本地按天去重、失败按小时重试）；token 刷新链路上也会补签。

> **GUI 显示样式说明**：magpie 桌面端的原生签到卡片（绿点「今日已签到 +N · 连续 N 天」+「自动签到」开关按钮）是宿主 App 内置的，只覆盖 WorkBuddy / Qoder / Trae / MiniMax 等少数供应商。本插件不在内置名单里，签到状态以额度窗口行的形式展示（`每日签到：今日已签 ✓ · 自动签到开`），开关走下述 options/配置文件通道，功能等价。

开关有**两个通道**（插件 options 优先，配置文件兜底——GUI 插件页没有选项入口时用文件）：

```sh
# 通道 1：插件 options（magpie CLI）
magpie plugin options opencode-autoclaw-auth '{"signin": false}'     # 关闭自动签到
magpie plugin options opencode-autoclaw-auth '{"signinHour": 8}'     # 只在每天 8 点补签
magpie plugin options opencode-autoclaw-auth '{"signinOnUsage": false}' # 查额度时不补签（默认 true）

# 通道 2：配置文件 ~/.config/magpie/autoclaw.json（保存即生效，热读取）
cat > ~/.config/magpie/autoclaw.json <<'EOF'
{ "signin": true, "signinHour": null, "signinOnUsage": true }
EOF
```

| 选项 | 默认 | 说明 |
|---|---|---|
| `signin` | `true` | 总开关 |
| `signinOnUsage` | `true` | 查额度时自动补签（最主要的触发链路） |
| `signinHour` | `null` | 限定补签小时：`8` = 仅当本地时间处于 8:00–8:59 时补签（错过该时段当天不签）；`null` 不限 |

## 已知行为

- 手机号账号只认国内 host、z.ai/Google 账号只认国际 host；对话 fetch 已内置 401/403 自动换侧重试。
- 上游模型白名单随版本变化，模型下线时上游返回 `400 非法模型`（原样透传错误消息）。
- refresh token 与 AutoClaw 桌面端**共用会互踢**（双方都轮换），插件账号请勿同时在桌面端登录。
- 上游偶发改协议时，`CLIENT_VERSION` / 端点路径可能需要随官方客户端更新。

## 开发与测试

纯函数单测（无网络依赖，Node ≥ 18 自带 test runner）：

```sh
node --check index.mjs          # 语法检查
node --test tests/index.test.mjs  # 单测
```

可测辅助函数经 `_internal` 导出；涉及上游交互的函数（`dailySignin` / `signinStatus` / `refreshTokens` 等）需真实凭据，不在 CI 覆盖范围。

## 免责声明

本项目为**非官方**社区插件，与智谱 AI / OpenClaw / AutoClaw 无任何关联。实现基于对官方客户端行为的逆向分析与实测，仅供个人学习研究；使用本插件产生的账号风险（包括但不限于违反上游服务条款导致的限制或封禁）由使用者自行承担。上游协议变更可能导致插件随时失效。

## 来源

协议逆向与实测记录见 workbuddy2api 仓库 `docs/specs/2026-09-11-autoclaw-provider.md`。MIT，见 [LICENSE](LICENSE)。
