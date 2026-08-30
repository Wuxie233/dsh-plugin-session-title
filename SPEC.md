# Session Title Settings Plugin

Accepted delivery contract for `@wuxie/dsh-session-title`.

## Goal

Web Settings gets a **会话标题** page. The user can edit the auto-title system prompt and optionally pin a title model. Later new automatic titles use the live settings. Existing titles and user-pinned titles stay unchanged.

## Scenario

A user opens Settings → 会话标题, pastes or tweaks the OpenCode-style Chinese ticket prompt, optionally picks a cheaper title model, and the next new conversation’s first eligible user message is titled with that prompt/route.

## In scope

- Dual-half web plugin under `~/CODE/dsh-plugins/session-title/`.
- Host half: register settings namespace `session-title`; intercept `purpose === 'session-title'` LLM streams; replace system prompt and/or route from live settings.
- Browser half: independent `settings.section` page (not a General row).
- Default prompt (verbatim OpenCode `agent.title.prompt`):

  `你只负责为会话生成标题。严格输出一行中文标题，不要解释，不要加引号，不要输出多行。目标风格：清晰、短、像工单标题，使用方括号标签突出上下文。标题格式必须为：[领域/对象][行为] 具体事项。规则：1) 第一标签写最能帮助识别上下文的领域或对象，优先使用用户明确提到的系统、产品、仓库、模块或业务域；否则从工作目录或任务内容推断，如 服务器、OpenCode、前端、后端、数据库、GitHub、文档、部署、网络、图片、论文。界面已经显示路径时，不要把 ~/CODE、root、CODE 这类泛路径当标题主体。2) 第二标签写行为，只能选一个最匹配词：调研、排查、修复、开发、优化、重构、配置、审查、整理、生成、同步、发布。3) 具体事项用 4-12 个中文字符概括核心对象和动作，必须具体可识别。禁止空泛词：代码开发、功能实现、项目优化、通用开发、问题处理、任务处理。要求：总长度 10-26 字符；不出现 新会话、标题、对话、会话 等词；不照抄原始提示词；不使用 emoji；除两个方括号标签外不要堆标点。示例：[服务器][调研] 宕机原因排查；[OpenCode][配置] 标题生成风格；[前端][修复] 移动端布局错位；[数据库][优化] 查询慢表索引；[GitHub][发布] PR创建流程。`
- Empty / whitespace-only prompt: do not replace system (DSH built-in English short instruction stays).
- Restore default: write the OpenCode prompt back. Do not change model selection.
- Default model mode: follow the current conversation route.
- Custom model: pick from `ctx.remote.session.modelCatalog()` groups. Persist provider + model.
- Custom route missing/disabled at call time: keep the conversation route and `ctx.logger.warn`.
- Cadence unchanged: first eligible human/automation prompt only (shipped first-prompt provider).
- Apply only to later title generations.

## Non-goals

- Do not patch `deepseek-harness` packages.
- Do not retitle existing sessions or unpin user titles.
- Do not switch cadence to all-prompts.
- Do not raise `maxTitleBytes` / `maxOutputTokens` in the base bundle.
- Do not persist secrets. Do not send QQ from this plugin.

## Constraints

- Copy-deploy via `install.sh` (never symlink).
- Package name identical in `package.json` `name`, `lib/client.js` module id, and `cordis.patch.yml` `name`.
- Browser half: plain JS, `React.createElement`, `window.__ModuleLoader__.load`.
- Browser `inject`: `slots`, `locale`, `connection`, `remote`, `remote.session`, `settingsScope`.
- Host `inject` must declare every service it reads (`llm` at least). Settings is optional via `ctx.inject(['settings'], …)` plus `ctx.get('settings')`.
- Frozen `GenerateOptions`: never mutate. Clone a new object and re-enter `ctx.llm.stream` with a WeakSet reentrancy guard. Always call `next()` for unpatched calls.
- Visual system: existing DSH settings (`--dsw-*` tokens). Mirror System prompts section spacing/type, not a new brand.
- `session/title-llm-request` may still record the original helper system/route. The dispatched call uses the patched envelope. Document this.

## Settings schema

Namespace `session-title`:

| Field | Type | Default | Meaning |
| --- | --- | --- | --- |
| `prompt` | string | OpenCode default | Title system prompt. `""` / whitespace → keep DSH built-in. |
| `modelMode` | `'follow'` \| `'custom'` | `'follow'` | Follow conversation route vs pin a model. |
| `provider` | string | `''` | Custom provider id. Ignored unless `modelMode === 'custom'` and both fields non-empty. |
| `model` | string | `''` | Custom model id. Same as provider. |

## Policy

`resolveTitlePolicy({ prompt, modelMode, provider, model, availableProviders, original })` returns:

- `system`: replacement string, or `undefined` to keep `original.system`
- `provider` / `model`: replacement, or `undefined` to keep original
- `fallback`: `'missing-provider'` when custom was requested but provider is not in `availableProviders`; otherwise `undefined`

Rules:

1. Trimmed prompt empty → `system` undefined.
2. Trimmed prompt non-empty → `system` is the trimmed prompt.
3. `modelMode !== 'custom'` or either custom field empty → keep original route.
4. Custom pair present and provider listed → use that pair.
5. Custom pair present and provider not listed → keep original route, `fallback: 'missing-provider'`.

## UI

- `settings.section` id `session-title`, `order: 22`, nav 会话标题 / Session title.
- Intro: first-message auto title; later titles unchanged; user rename still pins.
- Multiline textarea for prompt (persist on blur and 400ms debounce).
- Restore default button; disabled when prompt already equals the OpenCode default.
- Model control: first item 跟随对话 / Follow conversation; then catalog groups from `ctx.remote.session.modelCatalog()`. Persist immediately.
- Catalog load failure: keep Follow conversation; show a short error; do not invent routes.

## Acceptance

- `node scripts/policy.test.mjs` covers empty prompt, OpenCode prompt, follow, custom available, custom missing, custom incomplete, frozen-clone contract (policy does not mutate `original`).
- `node --check lib/client.js`; ESM host files copied to `.mjs` then `--check`.
- After `./install.sh` and a web patch insert of `id: session-title-settings` (not `session-title`; that id belongs to `@deepseek-ai/dsh-session-title`): `cd ~/.dsh/profiles/web && node -e "await import('@wuxie/dsh-session-title')"`.
- Host intercept clones frozen title requests, replaces system when prompt is set, and does not recurse.
- Settings page appears under Settings nav; saving prompt/model writes `session-title` in user settings.

## Resolved decisions

- Independent settings page.
- Prompt + selectable title model.
- Empty prompt → DSH English builtin; Restore default exists and only restores prompt.
- New titles only.
- Default model follows conversation.
- Unavailable custom model → conversation route + warning.
- First-prompt cadence only.
