## 当前接入方式：dsh-std adapter

本插件通过 dsh-plugin.json 声明标准 host facet；lib/index.js 发布
plugins.starpivot.dev/v1 HostPlugin，lib/host.js 保留业务策略并接收 adapter API。
有浏览器界面的插件另声明 LocalModule，使用私有 WebPlugin surface。

执行 ./install.sh 会先验证全部九个自制插件的组合、Web 界面与 CodeCarry
原生 Remote，再备份并复制部署；失败不替换生产插件。需先安装同级 dsh-std
维护仓库及其依赖。部署后在没有活跃任务时重启 dsh，并刷新 Web。

不要把 lib/index.js 直接作为 Cordis 插件挂载；原插件的 cordis insert 行由
共享部署器移除，配置迁入 adapter.componentConfigs。不要链接运行时插件目录。
标准协议不承诺未来版本永久兼容；native ctx/hooks 的变化由候选门禁和集中
adapter 维护控制。Web 界面不会自动出现在原生 Android 中。

下方保留业务说明和历史修复记录；涉及旧式直接挂载、导入和安装步骤的内容，
以本节为准。

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
