# dsh-plugin-session-title

设置 → **会话标题**：随时改自动取名系统提示词，并可锁定取名模型。

默认提示词是本机 OpenCode `agent.title.prompt` 的工单风格中文。清空提示词则使用 DSH 自带英文短指令。恢复默认只写回这段提示词，不改模型选择。

## 行为

- 时机仍是第一条合格用户消息（人手或 Host Automation）后取一次。之后不自动重取；侧栏手改过的标题钉住。
- 默认取名模型跟随当前对话路线。设置页可改成已装模型。
- 锁定的路线当时不可用：回退到对话路线，host 打警告。
- host 拦截 `purpose: 'session-title'` 的 LLM 调用。冻结的 `GenerateOptions` 会克隆后再发，不改原对象。
- `session/title-llm-request` 仍可能记录 helper 原始 system/route；真正发出去的是克隆后的信封。
- 设置页模型目录来自 `session/modelCatalog`（`ctx.remote.session.modelCatalog()`），不是已移除的 `connection.api.llm.models`。

## 安装 / 更新

```sh
./install.sh
# ~/.dsh/profiles/web/cordis.patch.yml:
#   - insert:
#       - id: session-title-settings
#         name: '@wuxie/dsh-session-title'
# 不要用 id: session-title：dsh-base 已经占用该行。
# 重启 dsh web（host 半）
# 刷新页面（设置页）
```

## 验证

- 纯函数：`node scripts/policy.test.mjs`
- `node --check lib/client.js`；ESM 的 `index.js` / `logic.js` 复制为 `.mjs` 再查
- 部署后：`cd ~/.dsh/profiles/web && node -e "await import('@wuxie/dsh-session-title')"`
