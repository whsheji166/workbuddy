---
title: Browser Model Bridge — 浏览器模型桥接法
tags: [workflow, monica, cdp, model-bridge, gpt-5.5, claude, gemini]
created: 2026-05-25
status: verified
---

# Browser Model Bridge — 浏览器模型桥接法

## 一句话概括

用**免费/便宜的模型（DeepSeek）做调度**，通过 CDP 操控 Edge 浏览器中的 **Monica 网页版**，调用**付费模型（GPT-5.5、Claude 4.6 Sonnet、Gemini 3.1 Pro 等）**干活，再把结果拉回来整合。**几乎零成本用最好的模型。**

## 架构图

```
你的一句话
     │
     ▼
┌─────────────────────────────┐
│  Layer 1: 调度器 (DeepSeek)  │  ← 免费模型
│  任务拆解 → 提示词构建          │
│  结果汇总 → 输出最终答案       │
└──────────┬──────────────────┘
           │ execute_code → MonicaController
           ▼
┌─────────────────────────────┐
│  Layer 2: 桥接层 (CDP)      │
│  WebSocket → Edge(9222)     │
│  → 操控 Monica 网页版         │
└──────────┬──────────────────┘
           │
     ┌─────┼─────┐
     ▼     ▼     ▼
┌──────┐┌──────┐┌──────────┐
│GPT-5.5││Claude││Gemini   │
│      ││4.6   ││3.1 Pro  │
│      ││Sonnet││         │
└──────┘└──────┘└──────────┘
```

## 前置条件

| 条件 | 说明 |
|------|------|
| Edge 远程调试 | 以 `--remote-debugging-port=9222` 启动 |
| Monica 已登录 | https://monica.im 账号有效，有积分/配额 |
| Hermes 模型 | 切换为 DeepSeek V4 Flash/Pro 等便宜模型 |
| CDP 配置 | `hermes config set browser.cdp_url "http://127.0.0.1:9222"` |
| websockets 包 | 已安装到 Hermes venv |

## 已验证的核心流程

### 1. 建立连接

```python
# 获取 browser_id
import requests
info = requests.get("http://127.0.0.1:9222/json/version").json()
# 从 webSocketDebuggerUrl 提取 browser_id
browser_id = info["webSocketDebuggerUrl"].split("/")[-1]
# 示例: "2fec4451-5589-455c-b53e-007e7980c00b"
```

### 2. 使用 MonicaController

完整代码见 Hermes Skill `browser-model-bridge`，或在 `execute_code` 中直接调用。

```python
ctrl = await MonicaController(browser_id).connect()

# 单模型查询
resp = await ctrl.query("Claude 4.6 Sonnet", "你的问题")
print(resp)

# 三模型批量对比
models = ["GPT-5.5", "Claude 4.6 Sonnet", "Gemini 3.1 Pro"]
results = {}
for m in models:
    results[m] = await ctrl.query(m, "同一个问题")
    
await ctrl.close()
```

### 3. 结果整合

DeepSeek 拿到三个模型的回复后：
- 去重
- 对比差异
- 提炼洞见
- 输出综合分析

## 可用模型列表

Monica「更多」弹窗内包含但不限于：

| 模型 | Monica 中名称 | 位置 |
|------|--------------|------|
| GPT-5.5 | GPT-5.5 | 侧边栏 |
| GPT-5.4 | GPT-5.4 | 「更多」 |
| Claude 4.6 Sonnet | Claude 4.6 Sonnet | 「更多」 |
| Claude 4.7 Opus | Claude 4.7 Opus | 侧边栏 |
| Claude 4.6 Opus | Claude 4.6 Opus | 「更多」 |
| Gemini 3.1 Pro | Gemini 3.1 Pro | 「更多」 |
| Gemini 3.5 Flash | Gemini 3.5 Flash | 侧边栏 |
| Grok 4.3 | Grok 4.3 | 侧边栏 |
| Grok 4.2 | Grok 4.2 | 「更多」 |

## 操作模式

### 模式 A：单模型深度查询
> "用 Claude 4.6 帮我分析这个商业计划书"

```
DeepSeek 构建详细提示词 → 切 Claude → 发送 → 读取回复 → 输出
```

### 模式 B：三模型交叉对比（推荐）
> "用三模型对比分析：短视频创业的机会在哪"

```
DeepSeek 构建提问
→ GPT-5.5 (市场视角)
→ Claude 4.6 Sonnet (技术/逻辑视角)  
→ Gemini 3.1 Pro (数据/趋势视角)
→ 汇总输出对比表
```

### 模式 C：多角度 Critique
> "帮我检查这个方案，三个模型从不同角度挑刺"

```
DeepSeek 拆解角度
→ GPT-5.5 看商业模式
→ Claude 4.6 看技术实现
→ Gemini 3.1 Pro 看数据风险
→ 输出综合评审报告
```

## 技术细节

### 为什么不能用 KeyboardEvent

```javascript
// ❌ 不工作 — isTrusted: false, React 17+ 忽略
textarea.dispatchEvent(new KeyboardEvent('keydown', {key: 'Enter'}));

// ✅ 正确 — CDP compositor 层注入, isTrusted: true
Input.dispatchKeyEvent({type: "rawKeyDown", key: "Enter", ...})
Input.dispatchKeyEvent({type: "keyDown", key: "Enter", ...})
Input.dispatchKeyEvent({type: "keyUp", key: "Enter", ...})
```

### React 输入框赋值

```javascript
// 必须用 native setter 绕过 React 受控组件
const s = Object.getOwnPropertyDescriptor(
    window.HTMLTextAreaElement.prototype, 'value'
).set;
s.call(textarea, '你的文本');
textarea.dispatchEvent(new Event('input', {bubbles: true}));
```

### CDP 会话管理

- sessionId 绑定到单个 WebSocket 连接
- 断开重连后 sessionId 失效
- 每次 `execute_code` 块内都要重新建立连接和 attach

## 验证记录

| 测试项 | 结果 |
|--------|------|
| CDP 连接 Edge 9222 | ✅ |
| Monica 页面加载识别 | ✅ |
| 侧边栏模型切换 | ✅ |
| 「更多」弹窗模型切换 | ✅ |
| Textarea 输入 | ✅ |
| CDP Enter 发送 | ✅ (textarea 清空确认) |
| Claude 回复读取 | ✅ (成功读到"第一性原理思维"回复) |
| 回复内容解析 | ✅ (从 markdown `<p>` 元素提取) |

## 风险与注意事项

- ⚠️ **仅供社群内部使用，不要外传！**
- ⚠️ 频繁自动化操作可能触发 Monica 风控
- ⚠️ 大量使用消耗 Monica 积分/配额
- ⚠️ 浏览器扩展可能更新导致 selectors 失效
- ⚠️ CDP 端口 9222 不应暴露到公网

## 相关链接

- Hermes Skill: `browser-model-bridge` (通过 `skill_view('browser-model-bridge')` 查看)
- Monica 网页版: https://monica.im/home/chat/Monica/monica
- Chrome DevTools Protocol: https://chromedevtools.github.io/devtools-protocol/
