# AGENTS.md

dsh 双半插件：设置页改自动取名提示词和可选模型；host 拦截 `purpose=session-title` LLM 调用。

## Architecture

- `lib/logic.js` 纯函数：OpenCode 默认提示词、`resolveTitlePolicy`。
- `lib/index.js` host 半：注册 `session-title` 设置命名空间；waterfall 克隆冻结请求后 `ctx.llm.stream(clone)`。
- `lib/client.js` 浏览器半：`settings.section` id `session-title`（会话标题页）。
- Deploy by copy (`install.sh`), never symlink.

## Conventions

- Edit → `./install.sh` → restart dsh web（host 半）；刷新页面（浏览器半）。
- Package name must stay identical in three places: `package.json` `name`, `lib/client.js` `__ModuleLoader__.load({ id })`, and the `cordis.patch.yml` mount row `name`.
- Client inject: `slots`, `locale`, `connection`, `remote`, `settingsScope`.
- Host inject: `llm`. Settings is optional via `ctx.inject(['settings'], …)` plus `ctx.get('settings')`.

## Gotchas & Decisions

- Never mutate frozen `GenerateOptions`. WeakSet-guard the clone and re-enter `ctx.llm.stream`; unpatched paths must `next()`.
- Empty / whitespace prompt leaves `system` undefined so DSH's English helper instruction stays.
- Restore default writes only `prompt`. Model mode stays.
- Custom route whose provider is not in `ctx.llm.listProviders()` keeps the conversation route and warns.
- `session/title-llm-request` is appended before this intercept; it may still show the helper's original system/route.
- Default prompt is duplicated in host `logic.js` and the browser bundle so the client half does not import host ESM.
- Profile mount row id is `session-title-settings`, not `session-title`. The base bundle already owns `session-title` (`@deepseek-ai/dsh-session-title`). Inserting the same id crash-loops `dsh-web`.

## Commands

- `./install.sh` — 部署到运行时（幂等）
- `node scripts/policy.test.mjs` — 纯函数
- `node --check lib/client.js`
- 生效验证：`cd ~/.dsh/profiles/web && node -e "await import('@wuxie/dsh-session-title')"`

## Module Map

单包。`lib/logic.js` + `lib/index.js` + `lib/client.js`。权威合同：`SPEC.md`。
