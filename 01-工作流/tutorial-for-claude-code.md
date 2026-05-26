# Browser Model Bridge — 通过 CDP 操控 Monica 调用付费大模型

## 核心思路

用本地便宜的模型（DeepSeek）做调度，通过 CDP 操控 Edge 浏览器中的 Monica 网页版，调用 Monica 里集成的付费大模型（GPT-5.5、Claude 4.6 Sonnet、Gemini 3.1 Pro 等），然后把回复读回来，交给便宜模型整合。

**效果：几乎零成本使用顶级模型。**

## 架构

```
用户一句话
    │
    ▼
DeepSeek (调度层, 免费)
    │ 拆解任务 → 构建提示词
    ▼
CDP → Edge(9222) → Monica 网页版 → 付费模型执行
    │ 等待 → 读取回复
    ▼
DeepSeek (整合层, 免费)
    │ 去重、对比、提炼
    ▼
最终输出 + 存入灵感库
```

## 关键前置

1. Edge 以 `--remote-debugging-port=9222` 启动
2. Monica 网页版 `https://monica.im/home/chat/Monica/monica` 已登录
3. Hermes 的 config 中 browser.cdp_url 指向 http://127.0.0.1:9222
4. websockets Python 包已安装

## 核心难点 (已解决)

### React 17+ 拦截 isTrusted=false 事件

Monica 使用 React SPA。普通方式:

```javascript
textarea.dispatchEvent(new KeyboardEvent('keydown', {key: 'Enter'}));
// → isTrusted: false → React 直接忽略
```

**解决方案: CDP Input.dispatchKeyEvent**

CDP 的 Input 域在浏览器 compositor 层注入按键事件，到达页面时 isTrusted: true，React 正常处理。

```python
# CDP 按键序列
Input.dispatchKeyEvent({type: "rawKeyDown", key: "Enter", ...})
Input.dispatchKeyEvent({type: "keyDown", key: "Enter", ...})  
Input.dispatchKeyEvent({type: "keyUp", key: "Enter", ...})
```

### React 受控组件赋值

```javascript
// 必须用 native setter 绕过 React 状态管理
const s = Object.getOwnPropertyDescriptor(
    window.HTMLTextAreaElement.prototype, 'value'
).set;
s.call(textarea, '你的文本');
textarea.dispatchEvent(new Event('input', {bubbles: true}));
```

### CDP 会话绑定

sessionId 绑定到单一 WebSocket 连接。断开重连后 sessionId 失效。所有操作必须在同一个 ws 连接内完成。

## Monica 页面结构

### 侧边栏模型列表

```
DIV.bot-item--Juhis        ← 侧边栏模型项 (包含 Monica, GPT-5.5, Claude 4.7 Opus 等)
  SPAN.bot-title--yPuEN    ← 模型名称
```

### "更多"弹窗

```
DIV.name--yJU3v (点击"更多"打开弹窗)
  → DIV.popover-content-wrapper
    → DIV.bot-item--lXUfg    ← 弹窗内模型项（包含全部模型）
```

### 模型识别方式

```javascript
// 侧边栏模型: 文本完全匹配
document.querySelector('.bot-item--Juhis') 
// 检查 innerText 是否以模型名开头

// 弹窗模型: 文本以 "模型名\n官方" 格式
document.querySelector('.bot-item--lXUfg')
// 检查 innerText 是否以模型名开头
```

### 输入框

```css
textarea[data-input_node="monica-chat-input"]
/* placeholder="问我任何问题..." */
/* class 包含 textarea--kVRBg, textarea-primary--OH3BL */
```

### 读取回复

Monica 回复渲染在 `<p>` 和 `<li>` 元素中。轮询这些元素即可获取。

### 新对话

点击文本为"新对话"的可见元素。

## 完整代码 (MonicaController)

```python
import asyncio, json, websockets

class MonicaController:
    """操控 Monica 网页版的完整封装"""

    def __init__(self, cdp_url, target_url="https://monica.im/home/chat/Monica/monica"):
        self.cdp_url = cdp_url
        self.target_url = target_url
        self.target_id = None
        self.ws = None
        self.sid = None
        self._mid = 0

    async def connect(self):
        """连接 CDP → 找到 Monica 标签页 → attach session"""
        self.ws = await websockets.connect(self.cdp_url)

        # 查找 Monica 标签页
        self._mid += 1
        await self.ws.send(json.dumps({"id": self._mid, "method": "Target.getTargets", "params": {}}))
        resp = json.loads(await self.ws.recv())
        for t in resp.get("result", {}).get("targetInfos", []):
            if self.target_url in t.get("url", ""):
                self.target_id = t["targetId"]
                break

        if not self.target_id:
            raise Exception(f"Monica tab not found at {self.target_url}")

        # Attach to target (flatten=True 保持 session 在响应中)
        self._mid += 1
        await self.ws.send(json.dumps({
            "id": self._mid,
            "method": "Target.attachToTarget",
            "params": {"targetId": self.target_id, "flatten": True}
        }))
        ev = json.loads(await self.ws.recv())  # Target.attachedToTarget event
        await self.ws.recv()                    # Response for our id
        self.sid = ev["params"]["sessionId"]
        return self

    async def _send(self, method, params=None):
        """发送 CDP 命令并接收响应"""
        self._mid += 1
        payload = {"id": self._mid, "method": method, "params": params or {}, "sessionId": self.sid}
        await self.ws.send(json.dumps(payload))
        return json.loads(await self.ws.recv())

    async def js(self, code, timeout=15000):
        """在页面上下文中执行 JavaScript"""
        r = await self._send("Runtime.evaluate", {
            "expression": code,
            "returnByValue": True,
            "timeout": timeout
        })
        return r.get("result", {}).get("result", {}).get("value", "")

    async def sleep(self, secs):
        await asyncio.sleep(secs)

    async def new_chat(self):
        """开启新对话"""
        await self.js("""
        (()=>{
            var e = [...document.querySelectorAll('*')]
                .find(x => x.innerText?.trim() === '新对话' && x.offsetParent);
            if (e) { e.click(); return 1 }
            return 0
        })()
        """)
        await self.sleep(3)

    async def switch_model(self, name):
        """切换模型。先试侧边栏，再试「更多」弹窗"""
        # 侧边栏
        r = await self.js(f"""
        (()=>{{
            var e = [...document.querySelectorAll('.bot-item--Juhis,.bot-item--lXUfg')]
                .find(x => x.innerText?.trim().startsWith('{name}') && x.offsetParent);
            if (e) {{ e.click(); return 1 }}
            return 0
        }})()
        """)
        if r == "1":
            return True

        # 点击「更多」打开弹窗
        await self.js("""
        (()=>{
            var e = [...document.querySelectorAll('*')]
                .find(x => x.innerText?.trim() === '更多'
                    && (x.className || '').includes('name--')
                    && x.offsetParent);
            if (e) { e.click(); return 1 }
            return 0
        })()
        """)
        await self.sleep(2)

        # 弹窗内查找
        r = await self.js(f"""
        (()=>{{
            var e = [...document.querySelectorAll('.bot-item--lXUfg')]
                .find(x => x.innerText?.trim().startsWith('{name}'));
            if (e) {{ e.click(); return 1 }}
            return 0
        }})()
        """)
        return r == "1"

    async def type_and_send(self, text):
        """输入文本并通过 CDP Enter 发送"""
        r = await self.js(f"""
        (()=>{{
            var ta = document.querySelector('textarea[data-input_node="monica-chat-input"]');
            if (!ta) return 0;
            var s = Object.getOwnPropertyDescriptor(
                window.HTMLTextAreaElement.prototype, 'value'
            ).set;
            s.call(ta, {json.dumps(text)});
            ta.dispatchEvent(new Event('input', {{bubbles: true}}));
            ta.focus();
            return 1;
        }})()
        """)
        if r != "1":
            return False

        await self.sleep(1)

        # 关键: CDP Input.dispatchKeyEvent (isTrusted: true)
        for evt_type, txt in [("rawKeyDown", "\r"), ("keyDown", "\r"), ("keyUp", None)]:
            p = {"type": evt_type, "key": "Enter", "code": "Enter", "windowsVirtualKeyCode": 13}
            if txt:
                p["text"] = txt
            await self._send("Input.dispatchKeyEvent", p)

        return True

    async def read_response(self, wait_max=45):
        """等待回复并返回文本"""
        for _ in range(wait_max // 2):
            await self.sleep(2)
            resp = await self.js("""
            (()=>{
                var els = [...document.querySelectorAll('p,li')]
                    .filter(function(e){
                        var t = e.innerText || '';
                        return t.length > 15
                            && !['有什么可以帮到您？', 'Chat', 'Agent', 'New'].includes(t.trim());
                    });
                if (els.length > 0) {
                    var res = els.slice(-5)
                        .map(function(e){ return e.innerText; })
                        .filter(function(t){
                            return !/^(Monica|Gemini|GPT|Claude|Grok)/.test(t);
                        });
                    return JSON.stringify(res);
                }
                return '[]';
            })()
            """)
            if resp and resp not in ("[]", '[]'):
                items = json.loads(resp)
                if items:
                    return items[-1]
        return None

    async def query(self, model, prompt):
        """完整周期: 切模型 → 发消息 → 读回复"""
        print(f"  [{model}] switching...", end=" ", flush=True)
        if not await self.switch_model(model):
            return f"SWITCH_FAILED:{model}"
        await self.sleep(2)

        print("sending...", end=" ", flush=True)
        if not await self.type_and_send(prompt):
            return "SEND_FAILED"

        print("waiting...", end=" ", flush=True)
        resp = await self.read_response()
        if resp:
            print("OK")
            return resp
        return "NO_RESPONSE"

    async def close(self):
        if self.ws:
            await self.ws.close()
```

## 使用示例

### 获取 CDP URL 中的 browser_id

```python
import requests
info = requests.get("http://127.0.0.1:9222/json/version").json()
cdp_url = "ws://127.0.0.1:9222/devtools/browser/" + 
    info["webSocketDebuggerUrl"].split("/")[-1]
```

### 单模型查询

```python
ctrl = await MonicaController(cdp_url).connect()
resp = await ctrl.query("Claude 4.6 Sonnet", "用中文解释什么是递归？")
print(resp)
await ctrl.close()
```

### 三模型批量对比

```python
prompt = "用一句话解释第一性原理思维"
models = ["GPT-5.5", "Claude 4.6 Sonnet", "Gemini 3.1 Pro"]

ctrl = await MonicaController(cdp_url).connect()
await ctrl.new_chat()

results = {}
for m in models:
    # 每个模型用新对话避免上下文干扰
    resp = await ctrl.query(m, prompt)
    results[m] = resp
    await ctrl.sleep(1)

await ctrl.close()

# DeepSeek 整合
for m, r in results.items():
    print(f"--- {m} ---\n{r}\n")
```

## Monica 可用模型列表

| 模型 | 位置 |
|------|------|
| Monica | 侧边栏 |
| Gemini 3.5 Flash | 侧边栏 |
| GPT-5.5 | 侧边栏 |
| Claude 4.7 Opus | 侧边栏 |
| Grok 4.3 | 侧边栏 |
| GPT-5.4 | 「更多」弹窗 |
| Claude 4.6 Sonnet | 「更多」弹窗 |
| Claude 4.6 Opus | 「更多」弹窗 |
| Gemini 3.1 Pro | 「更多」弹窗 |
| Gemini 3.1 Flash-Lite | 「更多」弹窗 |
| Grok 4.2 | 「更多」弹窗 |

## 验证记录

| 测试项 | 结果 |
|--------|------|
| CDP 连接 Edge 9222 | ✅ |
| Monica 页面加载 | ✅ |
| 侧边栏模型切换 | ✅ |
| 「更多」弹窗模型切换 | ✅ |
| textarea native setter 赋值 | ✅ |
| CDP Input.dispatchKeyEvent 发送 Enter | ✅ (textarea 清空) |
| 回复读取 (p/li 元素) | ✅ |

## 注意事项

1. **不要外传** — 此方法利用 Monica 网页版登录态间接调用付费模型
2. CDP session 绑定到单个 WebSocket，不能断开重连
3. 每次 execute_code 块需要重新建立 ws 连接
4. 频繁操作可能触发 Monica 风控
5. Edge 远程调试端口 9222 不应暴露到公网
