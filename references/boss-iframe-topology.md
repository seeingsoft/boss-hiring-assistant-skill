# BOSS iframe 拓扑图与操作策略

## 页面架构总览

BOSS 直聘前端为 **SPA shell + iframe 子应用**架构：

| 主页面 URL | iframe 实际 URL | 内容 |
|------------|----------------|------|
| `/web/chat/index` | `/web/frame/recommend/` | 推荐牛人页 |
| `/web/chat/index` | `/web/frame/recommend/interaction` | 收藏/互动页 |
| `/web/chat/index` | `/web/frame/job/list-new` | 职位管理/JD 页 |
| `/web/chat/index` | srcdoc iframe | 聊天列表（反自动化隔离） |

**关键事实**：主页面几乎为空壳，所有业务内容在 iframe 中。

## 操作策略决策表

| 目标内容 | 首选策略 | 备选策略 | 禁止策略 |
|----------|---------|---------|---------|
| 推荐牛人卡片 | `/navigate` 到 `/web/frame/recommend/` 在顶层操作 | eval 穿透 iframe contentDocument | 在主页 eval 直接查 `.card-item` |
| 职位 JD | `/navigate` 到 `/web/frame/job/list-new` 在顶层操作 | eval 穿透 iframe contentDocument | 在主页 eval 直接查 |
| 收藏/互动页 | `/navigate` 到 `/web/frame/recommend/interaction` 在顶层操作 | eval 穿透 iframe contentDocument | 在主页 eval 直接查 |
| 聊天列表 | ❌ DOM 不可直接访问（见 srcdoc 章节） | 直接构造聊天线程 URL 导航 | eval 穿透 srcdoc iframe |

## srcdoc iframe 特殊说明（聊天列表）

BOSS 聊天列表使用 `<iframe srcdoc="...">` 创建隔离 JS 上下文。

**表现**：
- `document.body.innerText` 不包含候选人文字
- `document.querySelector` 查聊天列表节点返回 null
- 聊天列表 DOM 完全不可通过常规 eval 访问

**可行替代路径**：
1. 直接导航到聊天线程 URL（如 `/web/chat/index#/chat/{candidate_id}-0`）
2. 通过全局引用 `parent.__xbcw` 访问（不稳定，不推荐）
3. 使用 /screenshot + OCR（仅限诊断，禁止作为主路径）

**禁止**：
- 不要尝试 eval 穿透 srcdoc iframe 的 contentDocument
- 不要把"srcdoc iframe 不可访问"当成 eval 失败并补试

## iframe 内 Vue 事件不响应问题

在 iframe contentDocument 中执行 `el.click()` 时：
- `event.isTrusted === false` → Vue 3 事件委托不响应
- `/click` 端点使用 `el.click()` → 同样不触发 Vue 事件

**解决方案**：
1. **首选**：导航到 iframe 实际 URL，在顶层页面使用 `/clickAt`（CDP Input.dispatchMouseEvent, isTrusted=true）
2. **备选**：在 iframe contentDocument 中计算元素坐标，返回主页面后用 CDP dispatchMouseEvent 手动点击（需自行实现坐标偏移计算）
3. **禁止**：在 iframe contentDocument 中反复尝试 el.click() 变体
