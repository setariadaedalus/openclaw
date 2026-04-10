# OpenClaw 作为 Being VM 框架 — 选型评估报告

> 评估日期：2026-04-08
> 评估目标框架：OpenClaw（GitHub: openclaw/openclaw）
> 需求基准：BEING_VM_REQUIREMENTS.md v1.0
> 项目：同在 (Be With You) · Mirror Mind 5.15+

---

## 一、执行摘要

### 1.1 总体结论

**OpenClaw 在技术选型中可作为「强候选」框架，但存在一个关键的架构性挑战需要解决。**

OpenClaw 的执行能力（网页浏览、文件操作、IM 渠道接入、沙盒安全）高度匹配 Being VM 的需求，且架构成熟、文档完善、社区活跃。然而，OpenClaw 的核心设计假设是「内置 LLM 决策的 AI 代理」，而需求文档要求的是「纯执行层（VM），决策由外部 Soul Layer（bionicCOT）接管」—— 这一架构性gap需要在集成层面额外设计变通方案。

### 1.2 综合评分

| 评估维度 | 权重 | 得分（1-5） | 关键发现 |
|----------|------|-------------|----------|
| **决策层可替换性** | 30% | **2** | ACP 协议仅支持 coding harness 接入，bionicCOT 需通过 MCP 或 Hook 变通接入 |
| **持续运行能力** | 20% | **5** | Gateway + Cron + Heartbeat + Task Flow 完整支撑 7×24 有状态运行 |
| **执行能力丰富度** | 15% | **5** | 浏览器控制、文件操作、媒体生成、代码执行全覆盖 |
| **IM 平台接入** | 10% | **5** | Telegram/Discord/Slack 等 20+ 渠道官方支持 |
| **架构可扩展性** | 10% | **5** | 完整插件 SDK + Hook 机制 + MCP 桥接 |
| **社区与维护** | 5% | **5** | 活跃开源社区，文档质量高，更新频繁 |
| **部署复杂度** | 5% | **4** | Docker 部署成熟，配置灵活 |
| **性能与资源效率** | 5% | **4** | 单实例 <2GB 内存，5-10 并发需实测 |

**加权总分：3.85 / 5.00**

---

## 二、详细评估

### 2.1 决策层可替换性（权重 30%）— 得分：2/5

#### 评估结果：弱匹配

这是整个评估中最关键的 gap，也是决定 OpenClaw 能否作为 Being VM 的核心问题。

#### OpenClaw 架构分析

OpenClaw 是典型的 **AI Agent 框架**，其核心执行流程是：

```
用户输入 → OpenClaw 内置 LLM 决策 → 工具调用 → 返回结果
```

OpenClaw 提供了 `ACP（Agent Client Protocol）` 机制来接入外部 coding harness：

```
OpenClaw ACP → Codex / Claude Code / Gemini CLI / OpenCode 等
```

**但这些外部 harness 全部是 specialized coding agents，不是通用决策引擎。**

#### 需求文档要求

```
Soul Layer（bionicCOT）产生决策指令
       ↓
VM Layer（执行层）忠实执行，不做决策
```

bionicCOT 需要的是：**通用意图 → 动作映射** 能力，让 VM 执行它输出的指令。

#### Gap 分析

| 需求 | OpenClaw 支持情况 | 说明 |
|------|-------------------|------|
| 决策层完全可替换 | ❌ 不支持 | OpenClaw 内置 LLM 决策，无法完全禁用 |
| bionicCOT 作为决策引擎 | ⚠️ 需变通 | 只能通过 MCP 或 Hook 拦截实现 |
| 感知数据透传到 Soul Layer | ⚠️ 部分 | `before_tool_call` / `after_tool_call` Hook 可拦截工具调用 |
| 动作指令从 Soul Layer 接收 | ⚠️ 部分 | 工具调用是标准接口，但 OpenClaw 会先用内置 LLM 处理输入 |
| 异步双向通信 | ✅ 强 | Webhooks + Hooks + 消息通道完善 |

#### Workaround 方案

**方案 A：将 bionicCOT 作为 OpenClaw 的「模型提供商」接入（推荐）**

bionicCOT 可以通过 MCP（Model Context Protocol）桥接作为 OpenClaw 的推理引擎：
- 实现 MCP server，暴露 bionicCOT 的决策能力
- 配置 OpenClaw 通过 MCP 与 bionicCOT 通信
- OpenClaw 的内置 LLM 决策被绕过，实际推理由 bionicCOT 完成

```
bionicCOT（MCP Server）←→ mcporter（MCP Bridge）←→ OpenClaw
```

**方案 B：使用 Hook 拦截工具调用，自建调度层**

- 实现自定义 Hook 接收 bionicCOT 的指令
- OpenClaw 保持在后台运行，但不主动决策
- 所有工具调用由 Hook 层转发给 VM 执行

**方案 C：只用 OpenClaw 的执行能力，完全屏蔽其决策**

- 配置 `tools.allow` 只开放执行工具
- 用外部服务（bionicCOT）直接调用 OpenClaw Gateway API
- 绕过 OpenClaw 的内置 LLM，完全自建决策层

#### 结论

**决策层可替换性是 OpenClaw 作为 Being VM 的最大障碍，但不是不可逾越的。** 通过 MCP 桥接或外部调度层，可以实现 bionicCOT 主导的决策流程。需要在 POC 阶段验证方案 A 的可行性。

---

### 2.2 持续运行时（权重 20%）— 得分：5/5

#### 评估结果：强匹配

| 需求项 | P | OpenClaw 实现 | 评估 |
|--------|---|---------------|------|
| 7×24 持续运行 | P0 | `Gateway` 后台进程 + `Cron` 持久化调度 | ✅ 强 |
| 有状态持久化 | P0 | `Task Flow` 持久化多步流程、`SQLite` 记忆系统 | ✅ 强 |
| 活动节奏自主 | P1 | `Heartbeat`（默认30分钟）+ `activeHours` 配置 | ✅ 强 |
| 异步事件响应 | P0 | `Hooks` + `Webhooks` + ACP 事件机制 | ✅ 强 |
| 资源弹性 | P1 | 工具 allow/deny + Docker 沙盒资源限制 | ✅ 中 |

#### 关键机制说明

**Gateway 进程管理**
- OpenClaw Gateway 是长期运行的后台进程
- 支持 `systemd` / `launchd` 守护进程管理
- 重启后自动恢复任务状态

**Task Flow 持久化**
- 多步骤工作流状态持久化到磁盘
- 支持 flow 级别的 revision tracking
- 进程重启后 flow 状态可恢复

**Cron + Heartbeat 双轨制**
- `Cron`：精确调度（支持 cron 表达式、一次性任务）
- `Heartbeat`：周期性活动（默认30分钟），即使无用户交互也能保持 Being 活跃

**会话持久化**
- Session 状态持久化到 `~/.openclaw/sessions/`
- 支持 `sessions_spawn` 跨会话恢复

---

### 2.3 执行能力丰富度（权重 15%）— 得分：5/5

#### 评估结果：强匹配

| 需求项 | P | OpenClaw 实现 | 评估 |
|--------|---|---------------|------|
| 网页浏览 | P0 | `browser` 工具（完整 Chromium 控制） | ✅ 强 |
| 浏览范围管控 | P0 | `SSRF` 策略 + `browser.ssrfPolicy` 白名单/黑名单 | ✅ 强 |
| 内容创作 | P0 | `exec` + `write`/`edit` + 图片/视频/音乐生成 | ✅ 强 |
| 文件操作 | P1 | `read`/`write`/`edit`/`apply_patch` 完整工具组 | ✅ 强 |
| 多媒体消费 | P1 | `image` 分析 + `browser` 视频理解 | ✅ 中 |
| 游戏/娱乐 | P2 | `browser` 可支持简单 Web 游戏 | ✅ 弱 |
| 内容发布 | P2 | `message` 工具 + 20+ 渠道插件 | ✅ 强 |

#### 核心工具说明

**浏览器控制（browser tool）**
- 支持完整 Chromium 操作：导航、点击、输入、截图、快照
- 多 Profile 支持（`openclaw`、`work`、`user` 等）
- 支持 Browserless / Browserbase 等远程浏览器服务
- SSRF 防护：可配置 hostname 白名单/黑名单
- 沙盒化浏览器运行（Docker 容器内）

**文件操作**
- `read`/`write`/`edit`：工作区文件完整操作
- `apply_patch`：多 hunk 文件补丁
- 工具策略控制（allow/deny）

**媒体生成与消费**
- 图片生成：`image_generate`（DALL-E、Flux 等）
- 图片理解：`image`（多模态分析）
- 视频生成：`video_generate`
- 音乐生成：`music_generate`
- TTS：`tts`

**代码执行**
- `exec`：Shell 命令执行，支持后台进程
- `code_execution`：沙盒化 Python 分析
- PTY 支持：交互式命令行

---

### 2.4 IM 平台接入（权重 10%）— 得分：5/5

#### 评估结果：强匹配

| 需求项 | P | OpenClaw 实现 | 评估 |
|--------|---|---------------|------|
| 同在原生界面 | P0 | OpenClaw 自有 Web UI + Gateway API | N/A |
| Telegram | P1 | `channels/telegram` 官方插件 | ✅ 强 |
| Discord | P1 | `channels/discord` 官方插件 | ✅ 强 |
| 微信 | P1 | 社区插件（`qqbot` 可研究） | ⚠️ 需开发 |
| 其他主流 IM | P2 | Slack, Teams, Line, WhatsApp, Matrix 等 | ✅ 强 |
| 统一消息抽象 | P0 | `message` 工具跨渠道发送 | ✅ 强 |
| 富媒体消息 | P1 | 各渠道分别实现富媒体 | ✅ 强 |

#### 官方支持渠道（核心插件）
- Telegram、Discord、Slack、Teams
- WhatsApp、Line、Zalo
- iMessage（BlueBubbles）、SMS
- Matrix、Nostr、IRC
- Feishu（飞书）、Google Chat
- Synology Chat、Nextcloud Talk

#### 统一消息抽象
`message` 工具支持跨渠道发送，格式统一：
```json
{
  "channel": "telegram",
  "to": "chat_id",
  "content": "消息内容"
}
```

---

### 2.5 架构可扩展性（权重 10%）— 得分：5/5

#### 评估结果：强匹配

| 机制 | OpenClaw 实现 | 评估 |
|------|---------------|------|
| 插件架构 | 完整 `openclaw.plugin.json` + `register()` API | ✅ 强 |
| Hook 系统 | 事件驱动生命周期 Hook（`before_tool_call` 等） | ✅ 强 |
| MCP 支持 | `mcporter` 桥接 MCP servers | ✅ 强 |
| 工具注册 | `registerTool/Channel/Provider` | ✅ 强 |
| 中间件机制 | Hook guard 语义（block/allow） | ✅ 强 |

#### 插件 SDK 核心接口

```typescript
export default definePluginEntry({
  id: "my-plugin",
  register(api) {
    api.registerProvider({ /* 模型提供商 */ });
    api.registerTool({ /* 工具注册 */ });
    api.registerChannel({ /* 渠道注册 */ });
    api.registerHook({ /* 生命周期钩子 */ });
    api.registerSpeechProvider({ /* 语音合成/STT */ });
    api.registerHttpRoute({ /* HTTP 路由 */ });
  },
});
```

#### Hook 事件类型

| 事件 | 触发时机 |
|------|----------|
| `before_tool_call` | 工具调用前（可 block） |
| `after_tool_call` | 工具调用后 |
| `message:received` | 收到消息 |
| `session:compact:before/after` | 会话压缩前后 |
| `gateway:startup` | Gateway 启动 |
| `agent:bootstrap` | Bootstrap 文件注入前 |

---

### 2.6 社区与维护（权重 5%）— 得分：5/5

#### 评估结果：强匹配

- **GitHub 活跃度**：文档完善、Issue 响应及时、PR review 规范
- **文档质量**：docs 目录下大量详细文档，覆盖所有功能
- **发布节奏**：根据 changelog，定期发布，更新频繁
- **社区插件**：ClawHub 生态，提供第三方插件分发

---

### 2.7 部署复杂度（权重 5%）— 得分：4/5

#### 评估结果：良好

| 部署方式 | 成熟度 | 说明 |
|----------|--------|------|
| Docker | ✅ 成熟 | 官方 Docker 镜像，`docker-compose` 示例 |
| npm 全局安装 | ✅ 成熟 | `npm install -g openclaw` |
| 源码构建 | ✅ 成熟 | TypeScript 源码，构建文档完善 |
| 云端部署 | ✅ 支持 | Railway、Vercel 等平台支持 |
| 桌面应用 | 🔄 开发中 | macOS/iOS/Android App 在路线图中 |

#### 安全部署考虑
- Gateway 默认 loopback 绑定
- Tailscale Serve 支持
- 共享密钥认证
- 沙盒隔离（Docker）

---

### 2.8 性能与资源效率（权重 5%）— 得分：4/5

#### 评估结果：良好

| 指标 | OpenClaw 情况 | 评估 |
|------|---------------|------|
| 动作执行延迟 < 5s | 通常满足 | 浏览器操作依赖网络 |
| 状态恢复 < 30s | ✅ 满足 | Task Flow 持久化 |
| 单实例内存 < 2GB | ✅ 满足 | 实际约 300-500MB（无并发） |
| 5-10 并发 Being | ⚠️ 需实测 | 内存和 CPU 取决于并发量 |

---

## 三、Integration Interface 设计建议

基于需求文档的「四协议」定义，评估 OpenClaw 的支持程度并给出集成建议：

### 3.1 感知协议（VM → Soul Layer）

**定义**：VM 向 Soul Layer 上报环境感知数据（屏幕内容、操作结果等）

**OpenClaw 支持**：
- `browser snapshot` 可获取页面内容
- `before_tool_call` / `after_tool_call` Hook 可拦截工具调用结果
- `message:preprocessed` Hook 可获取处理后的消息

**集成建议**：
```typescript
// 通过 Hook 拦截工具调用，将感知数据转发给 bionicCOT
api.registerHook({
  event: "after_tool_call",
  handler: async (event) => {
    // 将工具调用结果转发到 Soul Layer
    await bionicCOT.report Perception({
      tool: event.tool,
      result: event.result,
      timestamp: event.timestamp,
    });
  },
});
```

### 3.2 动作协议（Soul Layer → VM）

**定义**：Soul Layer 向 VM 下发动作指令（打开 URL、输入文字等）

**OpenClaw 支持**：
- 工具调用是标准接口
- `sessions_spawn` 可创建子任务
- Webhooks 可从外部触发任务

**集成建议**：
```json
// bionicCOT 输出的动作指令格式
{
  "action": "browser.navigate",
  "params": { "url": "https://example.com" }
}
```

OpenClaw 通过 Gateway API 或工具调用执行对应操作。

### 3.3 事件协议（异步通知）

**定义**：异步事件通知（用户消息到达、定时触发等）

**OpenClaw 支持**：
- `message:received` Hook：用户消息到达
- `gateway:startup` Hook：系统启动
- `Cron` + `Heartbeat`：定时触发
- Webhooks：外部 HTTP 触发

**集成建议**：
```json
// 事件格式
{
  "type": "message:received",
  "from": "user_123",
  "content": "帮我查一下天气",
  "channel": "telegram",
  "timestamp": "2026-04-08T10:00:00Z"
}
```

### 3.4 状态协议（持久化与恢复）

**定义**：Being 状态的序列化/反序列化格式

**OpenClaw 支持**：
- `Task Flow`：多步流程状态持久化
- `sessions`：会话状态持久化
- `memory`：记忆系统持久化（SQLite）
- `cron`：定时任务持久化

**集成建议**：
- Being 的完整状态应包含：会话快照、记忆索引、活跃 Flow、待执行任务
- 建议在 Soul Layer 层维护状态版本历史

---

## 四、风险分析

### 4.1 高风险项

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **决策层不可完全替换** | OpenClaw 内置 LLM 决策无法完全禁用 | POC 验证 MCP 桥接方案 |
| **bionicCOT 接入方式未验证** | 理论可行但无实际案例 | 尽快搭建 POC 原型 |

### 4.2 中风险项

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **微信插件缺失** | 需求 P1 无官方支持 | 研究 qqbot 插件或自研 |
| **思维流可视化不在 VM 层** | OpenClaw 无内置思维流 | Soul Layer 自己实现 |
| **并发性能未经实测** | 5-10 Being 并发未验证 | 搭建压测环境 |

### 4.3 低风险项

| 风险 | 描述 | 缓解措施 |
|------|------|----------|
| **视频理解能力有限** | 主要通过浏览器截图理解 | 配合 `image` 工具增强多模态理解 |
| **Windows 沙盒支持** | Docker 沙盒在 Windows 上可能有差异 | 使用 WSL2 或远程 Docker 主机 |

---

## 五、Recommendations（建议）

### 5.1 立即行动项（P0）

1. **搭建 MCP 桥接 POC**
   - 实现 bionicCOT MCP server 的最小版本
   - 验证通过 mcporter 桥接 OpenClaw 的可行性
   - 如果 MCP 方案不可行，评估 Hook 拦截方案的复杂度

2. **验证核心执行能力**
   - 搭建 OpenClaw + 沙盒环境
   - 测试浏览器控制、文件操作、代码执行
   - 验证 SSRF 策略和白名单配置

### 5.2 短期评估项（P1）

1. **IM 渠道适配**
   - 确认 Telegram/Discord 渠道满足同在产品需求
   - 评估微信接入方案（自研或社区插件）

2. **性能压测**
   - 搭建 5-10 个 Being 并发场景
   - 测试内存占用和响应延迟

3. **状态持久化验证**
   - 验证 Task Flow + 记忆系统在重启后正确恢复
   - 测试断点续传和状态一致性

### 5.3 架构设计项（P1）

1. **Integration Interface 设计**
   - 基于「四协议」定义，编写详细的接口规范
   - 确认感知数据格式、动作指令格式、事件格式

2. **部署架构设计**
   - Docker 化部署方案
   - 资源配额和沙盒配置

---

## 六、附录

### A. 评估矩阵汇总

| 评估维度 | 权重 | 得分 | 关键发现 |
|----------|------|------|----------|
| 决策层可替换性 | 30% | 2 | ACP 仅支持 coding harness，需 MCP/Hook 变通 |
| 持续运行能力 | 20% | 5 | Gateway + Cron + Heartbeat + Task Flow 完整 |
| 执行能力丰富度 | 15% | 5 | 浏览器/文件/媒体/代码执行全覆盖 |
| IM 平台接入 | 10% | 5 | 20+ 官方/社区渠道插件 |
| 架构可扩展性 | 10% | 5 | 完整插件 SDK + Hook + MCP |
| 社区与维护 | 5% | 5 | 活跃开源社区，文档完善 |
| 部署复杂度 | 5% | 4 | Docker 成熟，配置灵活 |
| 性能与资源效率 | 5% | 4 | 单实例低内存，并发需实测 |

**加权总分：3.85 / 5.00**

### B. OpenClaw 关键文档索引

| 文档 | 路径 | 用途 |
|------|------|------|
| 插件开发 | `docs/tools/plugin.md` | 插件架构参考 |
| Hook 系统 | `docs/automation/hooks.md` | 生命周期拦截 |
| 浏览器工具 | `docs/tools/browser.md` | 网页浏览能力 |
| 任务系统 | `docs/automation/tasks.md` | 后台任务管理 |
| Task Flow | `docs/automation/taskflow.md` | 多步流程持久化 |
| Cron 调度 | `docs/automation/cron-jobs.md` | 定时任务 |
| ACP 协议 | `docs/tools/acp-agents.md` | 外部 harness 接入 |
| MCP 桥接 | `docs/tools/index.md`（提及 mcporter） | MCP 集成 |
| 沙盒安全 | `docs/gateway/sandboxing.md` | 安全隔离 |
| 模型提供商 | `docs/concepts/model-providers.md` | LLM 接入 |

### C. 需求文档对照

本文档评估基于以下需求规格：
- **来源**：`BEING_VM_REQUIREMENTS.md` v1.0
- **版本**：2026-04-05
- **项目**：同在 (Be With You) · Mirror Mind 5.15+

---

*评估人：Claude Code*
*评估日期：2026-04-08*
*文档版本：1.0*
