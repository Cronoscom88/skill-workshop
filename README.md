# skill-workshop

最小闭环：**Chrome 扩展录制 → 导出 YAML → 本机执行服务用 Playwright 回放**（配套 **Skill工坊** 管理端）。

## 目录

- `extension/` — Chrome 扩展（Manifest V3）
- `runner/` — Node 执行服务（Express + Playwright）
- `python-admin/` — **Skill工坊**（Python / FastAPI）：Skill Store 目录、`/api/chat` 调 LLM + 工具、`/api/skills/publish` 入库并同步 YAML 到 `runner/playbooks/`。**配置与使用说明：** [python-admin/docs/使用说明.md](python-admin/docs/使用说明.md)
- `skill-store/` — 默认 Skill Store 根目录（与 `python-admin` 配置一致）

## 1. 执行服务（先跑通）

```bash
cd runner
npm install
npx playwright install chromium
npm start
```

默认监听 `http://127.0.0.1:3456`。浏览器打开根路径 **`/`** 会返回 JSON 说明（不是网页）；**`Cannot GET /`** 为旧版行为，更新代码后重启 runner 即可。

健康检查：

```bash
curl -s http://127.0.0.1:3456/health
```

内置示例剧本 `smoke`（只打开 https://example.com/）：

```bash
curl -s -X POST http://127.0.0.1:3456/v1/playbooks/smoke/runs -H "Content-Type: application/json" -d "{\"parameters\":{}}"
```

若返回 JSON 里 `status` 为 `succeeded`，说明执行服务可用。

**有界面回放（便于对照录制）**：在启动前设置环境变量 `HEADLESS=false`，会弹出 Chromium 窗口逐步执行：

```powershell
$env:HEADLESS="false"
cd runner
node server.js
```

## 2. 安装扩展

1. 打开 Chrome → `chrome://extensions`
2. 开启「开发者模式」
3. 「加载已解压的扩展程序」→ 选择本仓库下的 `extension` 文件夹

## 3. 录制并回放

1. **加载或更新扩展后，请刷新一次目标网页**（否则旧标签页里还没有注入脚本）。
2. 打开任意 **http / https** 页面后，看 **页面右下角** 是否出现深色 **「Skill工坊」浮动条**（含 playbook_id、意图描述、开始/停止、复制 YAML、**一键复制 Skill 模板**）。**不必依赖右键菜单**；浮动条即主入口。
3. 也可点击工具栏里的扩展图标打开弹窗操作；若提示无法连接页面脚本，请 **刷新当前标签页**。
4. 点「开始录制」，在页面上正常操作业务流程，再点「停止并生成 YAML」。
5. 点「复制 YAML」，保存为 `runner/playbooks/<你的id>.yaml`（文件名即 `playbook_id`）。
6. 根据录制页域名，检查 YAML 里 `site.allowed_origins` 是否与真实 origin 一致（不一致时执行服务会拒绝导航）。
7. 调用：

```bash
curl -s -X POST http://127.0.0.1:3456/v1/playbooks/<你的id>/runs -H "Content-Type: application/json" -d "{\"parameters\":{}}"
```

表单类步骤可在 YAML 里把固定文案改成占位符，例如 `value: "{{ expense_reason }}"`，请求里传入：

```json
{ "parameters": { "expense_reason": "差旅" } }
```

执行后抓取页面摘要与链接（供智能总结等，可选）：

```json
{ "parameters": {}, "options": { "capture_result": true } }
```

或在 playbook 顶层设置 `capture_result: true`（见 `python-admin/docs/使用说明.md`）。

## 3.1 一键生成 Skill 模板（SKILL.md）

录制面板里可编辑：

- **playbook_id**：与保存 YAML 的文件名一致（不含 `.yaml`），例如 `pb-expense-xxx`。
- **意图描述 description**：给模型判断「何时调用」的说明（建议第三人称、含系统名与关键词）。

操作顺序建议：**填好两项 → 开始录制 → 停止 → 复制 YAML** 存为 `runner/playbooks/<playbook_id>.yaml`，再点 **「一键复制 Skill 模板（SKILL.md）」**。

在项目（或用户目录）下新建：

` .cursor/skills/<与业务相关的英文目录名>/SKILL.md `

将剪贴板内容粘贴保存。技能发现会优先读取 frontmatter 里的 `description`；正文说明如何调用你们的执行服务 HTTP API。

> 内置的 create-skill 规范见 Cursor 文档；**不要**往 `~/.cursor/skills-cursor/` 写入。

## 4. YAML 约定（正式版）

- `skill_meta.description`：业务人员可改为「何时由对话触发」。
- `site.allowed_origins`：允许打开的页面 origin，多一条安全校验。
- `steps`：支持 `navigate` / `click` / `fill`，`target.by` 为 `css` 时 `target.value` 为选择器。

## 已知边界（正式版）

- 扩展在部分系统页需刷新后再「开始录制」；`chrome://` 等受限页无法注入。
- 选择器为启发式生成，复杂页面需手工改 YAML 或让前端加 `data-testid`。
- 执行服务未做鉴权，仅适合本机或内网调试。
