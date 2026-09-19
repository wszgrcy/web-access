# CDP Proxy API 参考

## 基础信息

- 地址：`http://localhost:3456`
- 启动：`node ~/.claude/skills/web-access/scripts/cdp-proxy.mjs &`
- 启动后持续运行，不建议主动停止（重启需 Chrome 重新授权）
- 强制停止：`pkill -f cdp-proxy.mjs`

## API 端点

### GET /health
健康检查，返回连接状态。额外包含 `headless`：连的是不是 headless 浏览器（决定截图是否自动前置）。
```bash
curl -s http://localhost:3456/health
```

### GET /targets
列出所有已打开的页面 tab。返回数组，每项含 `targetId`、`title`、`url`。
```bash
curl -s http://localhost:3456/targets
```

### POST /new
创建新后台 tab。Proxy 会先创建 `about:blank`、完成 CDP attach，再显式导航并等待目标 URL 的 DOM 至少进入 `interactive`，避免把新标签页初始的空白文档误判为目标页面。**URL 通过 POST body 原样传入**，无需 URL-encode、不会因 query 中含 `&` 被切分。返回 `{ targetId }`。
```bash
curl -s -X POST --data-raw 'https://example.com' http://localhost:3456/new
# 含 query 的目标 URL（如带 token 的小红书笔记）也直接原样传：
curl -s -X POST --data-raw 'https://www.xiaohongshu.com/explore/xxx?xsec_source=app_share&xsec_token=ABC&type=normal' http://localhost:3456/new
```
> v2.5.3 起改为 POST。旧的 `GET /new?url=...` 返回 400 + 迁移指引，详见 `migration-2.5.3.md`。

### GET /close?target=ID
关闭指定 tab。
```bash
curl -s "http://localhost:3456/close?target=TARGET_ID"
```

### POST /navigate?target=ID
在已有 tab 中导航到新 URL，自动等待加载。**target 走 query（不带特殊字符的不透明 ID），URL 走 POST body**。
```bash
curl -s -X POST --data-raw 'https://example.com' "http://localhost:3456/navigate?target=ID"
```
> v2.5.3 起改为 POST。旧的 `GET /navigate?target=...&url=...` 返回 400 + 迁移指引，详见 `migration-2.5.3.md`。

> `/new` 和 `/navigate` 的等待是浏览器文档层的基础保证，不是业务内容完成保证。验证页、登录跳转、SPA 异步渲染等仍可能继续变化；调用后必须按主 Skill 的“页面就绪与完成判断”检查目标内容，不能只看 `readyState`。

### GET /back?target=ID
后退一页。
```bash
curl -s "http://localhost:3456/back?target=ID"
```

### GET /info?target=ID
获取页面基础信息（title、url、readyState）。
```bash
curl -s "http://localhost:3456/info?target=ID"
```

### POST /eval?target=ID
执行 JavaScript 表达式，POST body 为 JS 代码。
```bash
curl -s -X POST "http://localhost:3456/eval?target=ID" -d 'document.title'
```

### POST /click?target=ID
JS 层面点击（`el.click()`），POST body 为 CSS 选择器。自动 scrollIntoView 后点击。简单快速，覆盖大多数场景。
```bash
curl -s -X POST "http://localhost:3456/click?target=ID" -d 'button.submit'
```

### POST /clickAt?target=ID
CDP 浏览器级真实鼠标点击（`Input.dispatchMouseEvent`），POST body 为 CSS 选择器。先获取元素坐标，再模拟鼠标按下/释放。算真实用户手势，能触发文件对话框、绕过部分反自动化检测。
```bash
curl -s -X POST "http://localhost:3456/clickAt?target=ID" -d 'button.upload'
```

### POST /setFiles?target=ID
给 file input 设置本地文件路径（`DOM.setFileInputFiles`），完全绕过文件对话框。POST body 为 JSON。
```bash
curl -s -X POST "http://localhost:3456/setFiles?target=ID" -d '{"selector":"input[type=file]","files":["/path/to/file1.png","/path/to/file2.png"]}'
```

### GET /scroll?target=ID&y=3000&direction=down
滚动页面。`direction` 可选 `down`（默认）、`up`、`top`、`bottom`。滚动后自动等待 800ms 供懒加载触发。
```bash
curl -s "http://localhost:3456/scroll?target=ID&y=3000"
curl -s "http://localhost:3456/scroll?target=ID&direction=bottom"
```

### GET /toFront?target=ID
把 tab 激活到前台（`Page.bringToFront`），再等一帧绘制（`requestAnimationFrame`，最多 500ms），返回 `{ visibility, hasFocus }`。
```bash
curl -s "http://localhost:3456/toFront?target=ID"
# {"visibility":"visible","hasFocus":true}
```
> 截图的前置条件。headless 下 `/screenshot` 已自动调用，一般不需要手动调。`visibility` 长期为 `hidden` 表示窗口被最小化或被其他窗口抢占。

### GET /screenshot?target=ID&file=/tmp/shot.png
截图。指定 `file` 参数保存到本地文件；不指定则返回图片二进制。可选 `format=jpeg`（jpeg 固定 quality=80）。
```bash
curl -s "http://localhost:3456/screenshot?target=ID&file=/tmp/shot.png"
curl -s "http://localhost:3456/screenshot?target=ID&nofront=1&file=/tmp/shot.png"
```

**截图必须在前台 tab。** headless=new 下 `visibilityState: "hidden"` 的 tab 不产帧，`Page.captureScreenshot` 会始终挂到 30s 超时（实测：hidden 状态下多次请求大多超时，`Page.bringToFront` 后同一请求 20~45ms 返回）。proxy 的处理：

| 参数 / 环境 | 行为 |
|------|------|
| 默认 + headless | 先 `bringToFront` 再截图；失败再前置一次重试 |
| 默认 + 有头 | **不**自动前置（不抢用户焦点），hidden tab 可能超时 |
| `nofront=1` | 强制不自动前置，需自行先调 `/toFront` |
| `CDP_SCREENSHOT_AUTO_FRONT=1` | 启动 proxy 时强制开启自动前置（包括有头） |
| `CDP_SCREENSHOT_AUTO_FRONT=0` | 启动 proxy 时强制关闭（包括 headless） |

headless 判定来自 `/json/version`：查 `Browser` **和** `User-Agent` 两个字段中的 `HeadlessChrome`。注意 Chrome 153 的 `Browser` 字段是 `"Chrome/x.y.z"`，headless 标记只出现在 `User-Agent` 里，只看 `Browser` 会漏判。判定结果在 `/health` 的 `headless` 字段暴露。

## /eval 使用提示

- POST body 为任意 JS 表达式，返回 `{ value }` 或 `{ error }`
- 支持 `awaitPromise`：可以写 async 表达式
- 返回值必须是可序列化的（字符串、数字、对象），DOM 节点不能直接返回，需要提取属性
- 提取大量数据时用 `JSON.stringify()` 包裹，确保返回字符串
- 根据页面实际 DOM 结构编写选择器，不要套用固定模板

## 错误处理

| 错误 | 原因 | 解决 |
|------|------|------|
| `Chrome 未开启远程调试端口` | Chrome 未开启远程调试 | 提示用户打开 `chrome://inspect/#remote-debugging` 并勾选 Allow |
| `attach 失败` | targetId 无效或 tab 已关闭 | 用 `/targets` 获取最新列表 |
| `CDP 命令超时: Page.captureScreenshot` | tab 处于 hidden 状态不产帧 | 去掉 `nofront=1` 让 proxy 自动前置，或先调 `/toFront`；确认窗口未被最小化 |
| `captureScreenshot 未返回数据` | CDP 返回了错误但无截图数据（tab 不可见/已关闭） | 先 `/toFront` 确认 `visibility=visible`，再重试 |
| `截图已获取但保存失败` | `file` 路径不合法 | 用当前平台能识别的完整路径 |
| `端口已被占用` | 另一个 proxy 已在运行 | 已有实例可直接复用 |
