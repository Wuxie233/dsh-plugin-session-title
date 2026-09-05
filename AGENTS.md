## dsh-std 集成约定（优先于下文旧部署说明）

- 当前入口是标准 facet，不再是 Cordis 插件；业务在 lib/host.js 的 createHost(api)。
- 原生 @deepseek-ai 导入只允许集中在同级 dsh-std/packages/adapter-dsh；插件通过 api 获取。
- session snapshots/read 和 Web composer 适配优先复用 adapter helper，未知日志 API 必须报错，不能当空会话。
- install.sh 运行全套候选门禁后复制部署；禁止手工追加旧 Cordis mount 行或只复制 index.js。
- 修改后必须运行本仓库 scripts/*.test.mjs（如存在）及共享 check-candidate.py；Web 插件须通过真实浏览器检查。
- 更新 DSH 前用独立、已安装依赖的候选源码运行门禁；通过后再切换生产版本。原生 ctx/events 仍需候选版本验证。

# AGENTS.md

dsh 双半插件：设置页改自动取名提示词和可选模型；host 拦截 `purpose=session-title` LLM 调用。

## Architecture

- `lib/logic.js` 纯函数：OpenCode 默认提示词、`resolveTitlePolicy`。
- `lib/index.js` host 半：注册 `session-title` 设置命名空间；waterfall 克隆冻结请求后 `ctx.llm.stream(clone)`。`export const name` 是 `session-title-settings`。
- `lib/client.js` 浏览器半：`settings.section` id `session-title`（会话标题页）；模型目录走 `ctx.remote.session.modelCatalog()`。
- Deploy by copy (`install.sh`), never symlink.

## Conventions

- Edit → `./install.sh` → restart dsh web（host 半）；刷新页面（浏览器半）。
- Package name must stay identical in three places: `package.json` `name`, `lib/client.js` `__ModuleLoader__.load({ id })`, and the `cordis.patch.yml` mount row `name`.
- Client inject: `slots`, `locale`, `connection`, `remote`, `remote.session`, `settingsScope`.
- Host inject: `llm`. Settings is optional via `ctx.inject(['settings'], …)` plus `ctx.get('settings')`.
- Browser `require()` only hits PLATFORM_MODULES (`packages/client/web/src/platform.ts`): `react`, `react/jsx-runtime`, `react-dom`, `react-dom/client`, `@deepseek-ai/cordis`, `@deepseek-ai/dsh-client-store`, `@deepseek-ai/dsh-client-ui-slots`, `@deepseek-ai/dsh-client-ui-primitives`.

## Gotchas & Decisions

- Never mutate frozen `GenerateOptions`. WeakSet-guard the clone and re-enter `ctx.llm.stream`; unpatched paths must `next()`. Title-llm requests are `deepFreeze`d with `purpose: 'session-title'` and are not agent-loop marked, so re-entry does not recurse or retitle.
- Empty / whitespace prompt leaves `system` undefined so DSH's English helper instruction stays.
- Restore default writes only `prompt`. Model mode stays.
- Custom route whose provider is not in `ctx.llm.listProviders()` keeps the conversation route and warns.
- `session/title-llm-request` is appended before this intercept; it may still show the helper's original system/route.
- Default prompt is duplicated in host `logic.js` and the browser bundle so the client half does not import host ESM.
- Profile mount row id is `session-title-settings`, not `session-title`. The base bundle already owns `session-title` (`@deepseek-ai/dsh-session-title`). Inserting the same id crash-loops `dsh-web`. Do not rename the package or invent a new mount id.
- Browser `createSnapshotStore` is `@deepseek-ai/dsh-client-store` (platform seed). Requiring `@deepseek-ai/dsh-client-runtime/client` white-screens the GUI.
- Model catalog is `ctx.remote.session.modelCatalog()` (`RemoteResult<ModelCatalog>`: `{ ok, value.groups }`). `connection.api` is gone; there is no `llm.models` Remote.

## Commands

- `./install.sh` — 部署到运行时（幂等）
- `node scripts/policy.test.mjs` — 纯函数
- `node --check lib/client.js`
- 生效验证：`cd ~/.dsh/profiles/web && node -e "await import('@wuxie/dsh-session-title')"`

## Module Map

单包。`lib/logic.js` + `lib/index.js` + `lib/client.js`。权威合同：`SPEC.md`。
