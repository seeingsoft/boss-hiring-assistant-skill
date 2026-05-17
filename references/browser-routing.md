# 浏览器路由

## 唯一路径

在 BOSS 页面中，所有浏览器访问统一只允许使用 `web-access`。

`web-access` 安装来源固定为：

- `https://github.com/eze-is/web-access`

如果当前环境尚未安装 `web-access`，默认先由龙虾代为安装。
只有代装失败、权限不足或环境阻塞时，才把这个链接给用户并请用户手动安装。
不要自行重新联网搜索。

允许的主路径顺序：

1. `web-access` DOM 读取
2. `web-access` CDP / eval
3. `web-access` 低风险 click / read

对于 Boss 页面里的关键点击动作，点击路径优先级固定为：

1. `web-access clickAt`
2. `web-access` 底层 CDP `Input.dispatchMouseEvent`
3. DOM `element.click()` 仅作单次兜底

不要把 DOM click 当作默认主路径，尤其不要在批量打招呼时默认用 `btn.click()`。

## web-access eval 使用规则

当需要使用 `web-access` / cdpproxy 的 eval 接口时，不要一上来就提交复杂 JS 逻辑。

必须先遵守下面顺序：

1. 先做一次最小探针 eval，验证接口传参方式正确
2. 只有探针成功后，才提交复杂逻辑
3. 如果探针失败，不要继续堆复杂脚本反复试错

对于 eval 传参，优先使用稳定写法：

- 使用原始文本请求体，而不是容易被转义污染的表单拼接
- 明确使用 `Content-Type: text/plain`
- 优先使用 `--data-raw` 传递脚本文本

在没有验证传参方式正确前，不要开始读取 JD、提取列表或做复杂 DOM 处理。

如果 eval 接口连续失败，优先怀疑：

- 请求体编码
- 转义方式
- `Content-Type`
- `--data-raw` / `-d` 的差异

而不是立刻怀疑页面结构本身。

对于候选人列表页，DOM 读取不仅包括首屏读取，也包括在页面存在“还有更多项”信号时的受控增量读取。
不要把首屏可见内容直接误认为完整列表。

如果 `web-access` 无法确认列表是否已经完整读取，龙虾必须明确说明“当前结果可能只是部分列表”，而不是自行补全或强行断言完整。

但对已经稳定显示候选人卡片的推荐页，不要反过来误判成“页面未完成加载”：

- 如果卡片列表已经显示且内容稳定，就应直接读取 DOM
- 不要默认等待 30 秒
- 不要默认改走截图快照分析
- 不要先让用户确认页面是否加载完成

只有页面明确存在继续加载信号时，才继续做受控增量读取。

对于 Boss 聊天页这种左右分栏 SPA，不要默认依赖 URL 跳转判断页面是否切换成功。
如果左侧列表点击后右侧面板更新，则应优先通过 DOM 面板内容变化来判断是否已切到目标候选人。

## 禁止项

以下行为默认禁止：

- 使用 `browser-use`
- 自行切换到其他通用浏览器 skill
- 默认使用截图、OCR、图像理解来读页面
- 为普通页面读取起子代理
- 在一个动作里来回切换多种浏览器方案
- 把 Boss 页面关键点击默认实现成 DOM `element.click()`

## 截图路径的边界

截图、OCR、图像理解默认彻底禁止，不属于常规路径。

只有在下面条件同时满足时，才允许退回截图路径：

1. `web-access` 的 DOM / CDP 路径已经尝试失败
2. 在同一路径内最多又补试了 `2` 次
3. 龙虾已经告诉用户失败原因
4. 龙虾已经明确说明截图方案的风险：
   - token 消耗更高
   - 可能提高 BOSS 风控或强退风险
   - 默认更建议继续坚持 `web-access`
5. 用户明确同意切换到截图方案

否则，不允许自行退回截图模式。

## 进入任务前的检查

开始 BOSS 招聘任务前，必须检查：

1. `browser-use` 等浏览器 skill 已由龙虾代为禁用或移除
2. `web-access` 已安装
3. 用户已经先在 Chrome / Chromium 中打开并登录 BOSS 直聘招聘者页面
4. 当前任务确实已切到 `web-access`
5. 能访问用户真实 Chrome / Chromium 会话
6. 当前会话里确实存在已登录的 Boss tab

如果检查不通过，立即暂停，不继续后续流程。

对于第 1 项，不要先向用户提问确认。
应当先由龙虾执行禁用/移除，再向用户汇报结果。

## 登录态与 tab 复用规则

Boss 任务中，优先级最高的浏览器规则是：

- 只复用用户当前已经登录的 Boss tab

因此，在开始接管浏览器前，必须先让用户完成：

- 打开 Chrome / Chromium
- 登录 BOSS 直聘招聘者页面

必须先做：

1. 列出当前可访问的 Boss targets
2. 找到已登录的 Boss tab
3. 记录它的 `targetId`
4. 后续所有读取、导航、点击都优先基于这个既有 tab 执行

不要默认执行：

- `web-access /new?url=...`
- 新建 tab 后再进入 Boss 招聘后台
- 新建窗口或新建浏览器上下文

因为新 tab / 新上下文可能没有继承用户的招聘登录态。

如果当前没有找到已登录的招聘者 tab，再提示用户登录，并在用户完成后继续复用该现有 tab。

## targetId 变化时的处理

`targetId` 变化不等于用户掉登录，也不等于必须让用户确认。

当发现原 `targetId` 失效、页面刷新、tab 重建、或 `/targets` 返回了新的 Boss target 时，必须先自动执行重绑定：

1. 重新调用 `/targets`
2. 在结果中寻找 URL 或 title 属于 Boss / BOSS 直聘 / zhipin 的 target
3. 对候选 target 做一次低成本登录态探针：
   - URL 是否仍在 `zhipin.com`
   - 页面是否出现招聘者相关内容
   - 是否没有明显登录页、验证码页、失效页
4. 如果探针通过，记录新的 `targetId` 并继续执行

不要因为 `targetId` 变化就立刻要求用户确认“是否还登录”。

只有在下面情况才暂停请用户处理：

- `/targets` 中找不到任何 Boss target
- 找到的 Boss target 明确是登录页、验证码页或异常页
- 登录态探针连续两次失败

暂停时只说明最小恢复动作，例如“请重新打开并登录 Boss 招聘者页面”，不要要求用户描述页面。

## 招聘后台进入规则

若需要进入职位管理或招聘后台：

1. 优先在已登录 Boss tab 内导航或点击进入
2. 优先复用同一个 `targetId`
3. 不要通过新开 tab 进入 `/web/recruiter/...` 或类似后台地址

如果当前 tab 只是 Boss 首页，也应先判断它是否仍属于已登录会话，再在同一 tab 内进入招聘后台。

## 子代理限制

默认不要把 Boss 浏览器控制任务交给子代理。

特别是这些动作，默认必须由主执行流完成：

- 找已登录 tab
- 进入职位管理
- 读取 JD
- 读取候选人列表
- 打开候选人线程
- 发送标准消息

这样可以减少子代理误开新 tab、误丢登录态、误切执行路径的风险。

---

## 截图调用前置门控

截图禁令的文本规则已写明，但模型在 eval 失败时仍可能退化到截图路径。
以下为结构性约束——必须按顺序完成所有前置动作，才能考虑截图：

### eval 失败标准诊断流程（截图之前的唯一合法路径）

当 eval 返回空结果或错误时，必须按以下顺序诊断，不可跳到截图：

1. 检查 Content-Type 是否为 text/plain
2. 检查 --data-raw 传参是否正确（不含 JSON 包裹）
3. 检查 targetId 是否仍然有效（/targets 刷新）
4. 检查 JS 表达式语法是否有错
5. 检查目标 DOM 是否在 iframe 内（需穿透或导航到 iframe URL）
6. 在同一路径补试最多 2 次

只有以上 6 步全部走完且仍然失败，才允许按原有 5 条放行条件考虑截图。

### eval 失败计数器（隐性规则）

同一动作（如"读候选人卡片""读 JD"）如果连续 3 次 eval 返回无效结果：
- 不继续尝试更多 eval 变体
- 立即暂停并汇报用户
- 告诉用户：已按标准诊断流程检查完毕，当前 eval 无法获取目标内容

---

## cdp-proxy /eval Boss 专用调用模板

### 最小探针（每次新 targetId 或新会话的第一步）

```bash
curl -s -X POST "http://localhost:3456/eval?target={TARGET_ID}" \
  -H "Content-Type: text/plain" \
  --data-raw "1+1"
```

预期返回：`{"value":2}` 或 `{"value":"2"}`

如果返回 `{"error":"..."}` 或 `attach failed`：
- 不要继续执行任何业务 eval
- 先检查：Content-Type、--data-raw、targetId

### iframe 穿透读取模板

当目标内容在 iframe 中时：

```bash
curl -s -X POST "http://localhost:3456/eval?target={TARGET_ID}" \
  -H "Content-Type: text/plain" \
  --data-raw "document.querySelector('iframe')?.contentDocument?.querySelector('.card-item')?.innerText || 'iframe_not_accessible'"
```

如果返回 `iframe_not_accessible`：
- 优先方案：导航到 iframe 实际 URL（如 `/web/frame/recommend/`），在顶层页面操作
- 不尝试更多 iframe 穿透变体

---

## navigate 后 eval 缓存陷阱

CDP `/navigate` 后立即调用 `/eval` 可能返回旧页面内容。

### 确认新内容已加载的标准流程

1. `/navigate` 到目标 URL
2. 调 `/info` 检查 `readyState` === `"complete"`
3. 执行一次轻量 eval 检查页面标识（如 `document.title` 或 `location.href`）
4. 确认返回内容与目标 URL 一致后，再执行业务 eval

### 如果 eval 返回疑似旧内容

- 检查返回内容中的 URL / title 是否匹配导航目标
- 如果不匹配：等待 2 秒后重试一次
- 如果仍返回旧内容：暂停并汇报，不要自行补全或假设

---

## DevToolsActivePort UUID 不匹配诊断

如果 `/health` 返回 `connected=false` 或 CDP Proxy 无法连接 Chrome：

1. 检查 Chrome 是否确实在运行且远程调试已启用
2. 检查 Chrome DevToolsActivePort 文件：
   - macOS: `~/.config/google-chrome/DevToolsActivePort` 或 Chrome 用户数据目录下
   - Windows: `%USERPROFILE%\AppData\Local\Google\Chrome\User Data\DevToolsActivePort`
3. 文件中的 WebSocket UUID 可能与 Chrome 实际会话不匹配
4. 解决方案：重启 Chrome（先完全关闭再重新以调试模式启动），或清理 DevToolsActivePort 文件

---

## 中文编码问题与 Base64 读取模板

### 问题描述

web-access 的 `/eval` 端点返回包含中文字符的字符串时，可能出现乱码。
例如预期返回 `"张阿梅 AI产品经理"`，实际返回 `" AI"`。

**根因**：CDP Proxy (`cdp-proxy.mjs`) 在 `JSON.stringify()` 序列化时，中文字符的 UTF-16 编码在 HTTP 响应传输过程中损坏。
这不是浏览器端问题，而是 Node.js HTTP 响应层的编码处理问题。

**影响范围**：Mac 和 Windows 均受影响。纯 ASCII 内容（数字、英文）不受影响。

### 何时必须使用 Base64 编码

以下场景 **必须** 使用 Base64 编码读取：

1. 读取候选人姓名、职位、公司等中文文本
2. 读取聊天消息内容
3. 读取 JD 文本
4. 读取简历面板文本
5. 读取任何包含中文的 `innerText` / `textContent`

以下场景 **不需要** Base64 编码：

1. 纯数字结果（如卡片数量 `12`）
2. 纯英文/ASCII 文本
3. CSS selector 查询（查询本身是 ASCII）
4. 布尔值 / 简单状态判断

### Base64 读取模板（标准版）

```bash
# Step 1: eval 中用 btoa+encodeURIComponent 编码中文，返回 Base64 字符串
curl -s -X POST "http://localhost:3456/eval?target={TARGET_ID}" \
  -H "Content-Type: text/plain" \
  --data-raw "JSON.stringify({b64:btoa(unescape(encodeURIComponent(document.querySelector('.target-selector')?.innerText||''))),len:document.querySelector('.target-selector')?.innerText?.length||0})"

# Step 2: 本地解码（Python）
python3 -c "
import json,base64,sys
raw=json.loads(sys.stdin.read())
if isinstance(raw.get('value'),str):
  inner=json.loads(raw['value'])
  if inner.get('b64'):
    text=base64.b64decode(inner['b64']).decode('utf-8')
    print(f'解码成功: {len(text)} 字符')
    print(text)
  else:
    print('无 b64 字段')
else:
  print(f'异常响应: {raw}')
" < /tmp/eval_result.json
```

### Base64 读取模板（单次提取多字段）

当需要从一张候选人卡片中提取多个中文字段时：

```bash
curl -s -X POST "http://localhost:3456/eval?target={TARGET_ID}" \
  -H "Content-Type: text/plain" \
  --data-raw "(function(){var c=document.querySelector('.card-item');if(!c)return JSON.stringify({found:false});var name=c.querySelector('.name')?.innerText||'';var title=c.querySelector('.job-title')?.innerText||'';var exp=c.querySelector('.experience')?.innerText||'';return JSON.stringify({found:true,b64:btoa(unescape(encodeURIComponent(JSON.stringify({name:name,title:title,exp:exp}))))}})()"
```

本地解码后 `JSON.parse` 即可得到结构化对象。

### Base64 读取模板（批量候选人列表）

当需要从推荐页提取多张卡片信息时：

```bash
curl -s -X POST "http://localhost:3456/eval?target={TARGET_ID}" \
  -H "Content-Type: text/plain" \
  --data-raw "(function(){var cards=document.querySelectorAll('.card-item');var list=[];cards.forEach(function(c){list.push({name:c.querySelector('.name')?.innerText||'',title:c.querySelector('.job-title')?.innerText||''})});return JSON.stringify({count:list.length,b64:btoa(unescape(encodeURIComponent(JSON.stringify(list))))}})()"
```

### Base64 读取的 token 经济性

- Base64 编码会使字符串长度增加约 33%
- 对于大段文本（如完整简历），建议先在浏览器端做字段筛选，只编码必要字段
- 不要对整页 `document.body.innerText` 做 Base64 编码，除非确实需要全部内容
- 优先用 selector 精确提取目标元素，再对提取结果做 Base64

### 编码问题诊断流程

当 eval 返回的内容包含乱码或疑似乱码时：

1. 检查返回值中是否出现 `` 或不可读字符
2. 如果出现，判断是否为中文内容
3. 如果是中文内容，切换到 Base64 编码模板重新读取
4. 不要怀疑页面结构或 selector 有误——中文乱码是 CDP Proxy 已知编码问题
5. 不要因为中文乱码而退回截图路径

### 短字段快速判断规则

- 如果返回值只包含 ASCII 字符且语义合理（如 `"12"`、`"active"`、`"true"`），无需 Base64
- 如果返回值包含 `` 或非预期空格/截断，大概率是中文编码问题，立即切 Base64
- 如果返回值与预期明显不符（如候选人姓名变成空字符串），先怀疑中文编码问题
