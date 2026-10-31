# gygo · AI 协作约定

这份文件是给 AI（Qoder/Claude/Cursor 等）看的**项目宪法**。写代码前先读这一页，能省掉 80% 的返工。
人类 reviewer 也照这份对。

## 一、术语表（禁止混用）

| 术语 | 类型 | 含义 | 出现位置 |
|---|---|---|---|
| `Link` | `model.Link` | **后台**资源记录，一条网盘分享 | `internal/model`, `internal/store/store.go`, `internal/api/*.go`（鉴权路由下） |
| `PublicLink` | `model.PublicLink` | **门户**展示的资源，是 Link 的对外投影 | `internal/store/public.go`, `internal/api/portal.go`, `internal/api/seo.go` |
| `Scrape` | 动词 | 用 TMDB ID 抓元数据（标题/海报/年份…） | `internal/scraper/tmdb.go` |
| `Push` | 动词 | 把 Link 推送到 Telegram 频道 | `internal/notifier/telegram.go` |
| `Pipeline` | 名词 | 后台 worker 队列，串 Scrape→Push | `internal/pipeline/pipeline.go` |
| `ShareID` | 字段 | 光鸭分享链接的唯一标识，业务主键 | `Link.ShareID`，DB UNIQUE |
| `TMDBID` | 字段 | 刮削时人工/AI 校准后的 TMDB 条目 ID | `Link.TMDBID` |

**不要**：`resource` / `item` / `entry` / `movie` / `link` 混着叫。同一个概念在 Go 里、TS 里、DB 里、UI 文案里必须是同一个词。

## 二、包边界（依赖方向）

```
cmd/server ──▶ api ──▶ pipeline ──▶ { scraper, notifier, store, parser, ratelimit, netutil }
     │                │
     │                └──▶ config ──▶ (leaf)
     └──▶ logging ──▶ (leaf，只依赖 stdlib)

model / auth / web / ratelimit / netutil / parser / logging ─ 叶子包，禁止 import 项目内其它包（除 model）
store ──▶ model（仅此一个内部依赖）
scraper ──▶ { config, netutil, ratelimit }
notifier ──▶ { config, model, netutil, ratelimit }
api ──▶ 所有其它内部包
```

**硬性规则**（违反 = PR 打回）：

1. `internal/model` 不 import 任何内部包，只依赖 stdlib + `time`。
2. `internal/store` 只 import `internal/model`。SQL、事务、schema 都在这里，**API 层禁止裸写 `db.Exec`**。
3. `internal/scraper`、`internal/notifier` **禁止** import `internal/store`。它们的输出交给 pipeline 决定要不要落库。
4. `internal/pipeline` 是**唯一**能同时看到 store/scraper/notifier 的地方；跨阶段编排都写这里。
5. `internal/api` 处理 HTTP 语义（鉴权、限流、请求/响应格式），**不要**在 handler 里塞业务循环。
6. `cmd/server/main.go` 只做装配（依赖注入），不写逻辑。**唯一例外**：`requestLog` 访问日志分级中间件留在 main.go，因为它和 `logging.New`、slog handler 的组装是一件事——拆到 `api` 反而会让 api 反过来决定日志策略。再加日志相关的 HTTP 逻辑请放进 `internal/api`。

## 三、参考实现（照这些抄，别自造）

**Store 事务写方法** → [internal/store/users.go](file:///Users/odin/Documents/Qoder/gygo/internal/store/users.go) 里 `UpdateUserGuarded` / `DeleteUserGuarded`  
要点：`tx.Begin()` → `defer tx.Rollback()` → 校验业务不变式（如最后一个管理员）→ 失败返回 sentinel error（如 `ErrLastAdmin`）→ `tx.Commit()`。

**限流 HTTP 客户端** → [internal/scraper/tmdb.go](file:///Users/odin/Documents/Qoder/gygo/internal/scraper/tmdb.go) 里 `get()`  
要点：`ratelimit.Limiter.Wait` → `http.NewRequestWithContext` → `io.LimitReader(resp.Body, maxAPIResponseBytes)` → 状态码分类。

**API handler 错误契约** → [internal/api/auth.go](file:///Users/odin/Documents/Qoder/gygo/internal/api/auth.go)  
要点：请求校验失败 → `400 + 中文 error`；鉴权失败 → `401`；权限不足 → `403`；资源不存在 → `404`；业务不变式违反（如 `ErrLastAdmin`）→ `400` + 具体中文；DB/内部错误 → `serverError(c, err)` 统一 500 + 日志。

**按大小滚动的日志文件与分级访问日志** → [internal/logging/logging.go](file:///Users/odin/Documents/Qoder/gygo/internal/logging/logging.go) 里 `Writer` / `rotate`，配 [cmd/server/main.go](file:///Users/odin/Documents/Qoder/gygo/cmd/server/main.go) 里 `requestLog`  
要点：slog 双写标准输出与 `data/logs/app.log`（目录 0750、文件 0600）；5xx→Error、4xx 或耗时超 1 秒→Warn、轮询与静态资源→Debug，`quietPaths` 是白名单而不是黑名单；单条日志超过上限也先滚动。新增字段一律小写下划线，**不要把密钥、Cookie 或提取码写进日志**。

**服务端注入 HTML head** → [internal/api/seo.go](file:///Users/odin/Documents/Qoder/gygo/internal/api/seo.go) 里 `headTags` / `injectHeadTags`  
要点：CSP 禁内联脚本，任何需要出现在首屏源码的 meta/link 都走服务端字符串注入；用户可控部分**必须**先经过白名单字符过滤（参考 `sanitizeCanonicalPath` / `sanitizeCanonicalQuery`）。

**前端 URL 绑定** → [web/src/lib/utils.ts](file:///Users/odin/Documents/Qoder/gygo/web/src/lib/utils.ts) 里 `safeExternalUrl`  
要点：`:href` 绑定用户/CSV 输入时**必须**过一层协议白名单（http/https），杜绝 `javascript:` / `data:` XSS。

## 四、代码风格约定

### Go
- 错误消息**中文**，短、动作化。"至少要保留一个启用的管理员"，而不是 "at least one enabled admin required"。
- 日志用 `slog.Info/Warn/Error(msg, "key", value, ...)`，msg 中文，key/value 保持小写下划线。级别按「会不会有人专门去捞它」选：正常完成的关键动作 `Info`（刮削完成、推送完成、批次结束），需要人看一眼但不致命 `Warn`（未匹配、待人工确认、回退、限流退让），只有排查现场要的细节 `Debug`（单条开始、请求状态、海报已缓存）。**批量循环里禁止逐条 `Info`**，5 MB 的滚动文件会被冲光。
- **禁止** `_ = someErr`。要么处理，要么 `slog.Error("...", "err", err)` 记下来。参考 pipeline 里 `updateStatus`。
- 上限常量命名 `max<X>`（如 `maxAPIResponseBytes`、`maxJSONBody`、`maxCSVBytes`），单位后缀（Bytes / MS / Count）。
- LIKE 查询必须走 `store.likeEscape` + `ESCAPE '\'`。
- 所有 `http.Response.Body` 读取都要 `io.LimitReader` 包一层，防止上游返回超大响应打爆内存。

### Vue/TypeScript
- 全站**统一系统默认字体**，禁止在 CSS 里声明 `font-family` 覆盖。
- `target="_blank"` 必须带 `rel="noopener noreferrer"`。
- **禁止** `v-html`。文案渲染走白名单解析器（参考 [web/src/views/PushView.vue](file:///Users/odin/Documents/Qoder/gygo/web/src/views/PushView.vue) 里的 `parseCaptionSegments`）。
- 用户可控数据绑到 `:href`/`:src` 前，走 `safeExternalUrl`。
- 组件内 `import` 顺序：Vue → lucide → 本地 UI → lib → 相对。

## 五、安全默认（不接受例外）

- 密码：`auth.HashPassword` 用 PBKDF2-HMAC-SHA256，`validPassword` 至少 8 位 + 2 类字符。
- 会话 token：`auth.HashToken` 用 HMAC-SHA256，密钥文件 `data/session.key`（0600）持久化。
- Cookie：`SameSite=Lax`；生产 TLS 下 `Secure`（`cfg.Server.SecureCookie` 覆盖）。
- CSRF：`requireSameOrigin` 中间件校验 Origin/Referer host。
- CSP：`script-src 'self'`，无 `'unsafe-inline'`（style-src 例外，Tailwind 需要）。
- 登录限流：`login_attempts` 表 + 内存缓存双层，IP + 用户名分别计数。
- SSRF：TMDB `imageBase` 只允许 `image.tmdb.org` / `themoviedb.org` / `api.themoviedb.org`。
- HSTS：只在**直连 TLS** 时下发，反代场景交给边缘。
- 运行日志：`<data_dir>/logs/app.log`（目录 0750 / 文件 0600，单文件 5 MB、保留 3 份）。访问日志取 `c.FullPath()` 的路由模式，查询串不落盘；下载只读启动时注入的固定路径，仅管理员可调，**不接受客户端指定的文件名**。

## 六、变更纪律（AI 特别容易踩）

1. **加东西之前先问能不能改**。看到"要新建一个 helper"，先在文件里搜 5 分钟。
2. **不做无关清理**。修 bug 就修 bug，不要顺手 rename 周边变量、加注释、拆导入。
3. **不加防御性分支**。`if x == nil { return }` 除非 x 真的能来自外部空值，否则删。
4. **不写文档/README/planning.md**，除非用户明确要求。
5. **不引入新依赖**，除非用户同意或 stdlib 明显没有对应能力。引入前先问一句。
6. **一个 PR ≤ 300 行 diff**。超过就拆——AI 一次生成 800 行时人的注意力会崩，评审只是走过场。
7. **每次改 `web/` 都必须 `make web && go build ./...`**（见 [auto memory: frontend-embed-rebuild]），否则 embed 里的前端和后端会不一致。

## 七、提交边界（哪些提交、哪些不提交）

### 必须提交

| 类别 | 文件 |
|---|---|
| 源码 | `cmd/`、`internal/`（除 `internal/web/dist/` 产物）、`web/src/` |
| 构建与部署 | `Makefile`、`Dockerfile`、`docker-compose.yml`、`.dockerignore`、`.github/` |
| 依赖锁定 | `go.mod`、`go.sum`、`web/package.json`、`web/package-lock.json` |
| 配置模板 | `config.example.yaml`、`.env.example` |
| 文档 | `README.md`、`docs/`、`AGENTS.md` |
| 占位文件 | `internal/web/dist/.gitkeep` —— 保证空目录下 `//go:embed all:dist` 仍可编译 |

### 绝不提交（`.gitignore` 已覆盖）

| 类别 | 文件 |
|---|---|
| 密钥与运行配置 | `config.yaml`、`.env`、`.env.*`（`!` 例外：`.env.example`） |
| 运行数据 | `data/`（含 `session.key`、数据库、海报、`branding/` 站点素材、`logs/` 运行日志）、`backups/`、`.workbuddy/` |
| 构建产物 | `bin/`、`web/dist/`、`internal/web/dist/*`（`!` 例外：`.gitkeep`）、`gygo`、`*.zip`、`*.tar.gz` |
| 依赖目录 | `node_modules/`、`web/node_modules/` |
| 杂项 | `.DS_Store`、`*.log`、`*.db`、`*.db-wal`、`*.db-shm` |

### 提交前须确认

- 新增依赖（`go.mod` / `package.json`）：先问是否必要。
- 数据库 schema 变更：`schema_meta.version` 必须 +1，并说明迁移路径。
- 示例里的真实域名、Bot Token、频道 ID、提取码必须替换为占位值。

### 提交前自检

```bash
git status --short          # 确认没有把 data/ config.yaml 之类带进来
make lint                   # vet + staticcheck + gosec + 前端 type-check
git diff --cached --stat    # 单次 diff 控制在 300 行内
```

## 八、Reviewer 快速清单

- [ ] 有无 `_ = err`？
- [ ] 有无 `http.Response.Body` 未包 `io.LimitReader`？
- [ ] 有无新的 SQL 未走 `store`？
- [ ] 有无 LIKE 拼字符串未走 `likeEscape`？
- [ ] 新 handler 的错误响应是不是**中文 + 4xx/5xx 分类正确**？
- [ ] 用户可控数据是不是过了 `safeExternalUrl` / `googleTokenRe` 之类白名单？
- [ ] `target="_blank"` 是不是带 `rel="noopener noreferrer"`？
- [ ] 有没有 5 行以上复制粘贴？
- [ ] 单元测试覆盖新逻辑的关键分支（成功/失败/边界各一条）？
- [ ] 新增日志：msg 中文、key 小写下划线、级别选对（批量循环里不逐条 `Info`）、有没有把密钥 / Cookie / 提取码写进去？
- [ ] 改了 `requestLog` / `quietPaths` 是不是同步了 `docs/dev-guide.md` §2 与 §10？

## 九、分支、版本与镜像

单人维护，分支越少越好。

### 分支

| 分支 | 规则 |
|---|---|
| `main` | 唯一长期分支，始终可部署。小改动（<300 行、不碰 schema）直接推 |
| `feat/<名>` `fix/<名>` | 仅在超过 300 行、涉及 schema 变更或需要灰度时开；合并回 `main` 后删除 |

不设 `develop` / `release` / `hotfix`，回滚靠 tag 而不是分支。

### 提交信息

统一前缀：`feat:` `fix:` `docs:` `ci:` `refactor:` `chore:`，冒号后中文描述。

### Tag（SemVer）

`v<MAJOR>.<MINOR>.<PATCH>`，只打在 `main` 上、CI 已通过的提交。预发布用 `v2.1.0-rc.1`。

- **MAJOR**：升级需人工介入（迁移不可回退、配置结构不兼容）
- **MINOR**：新功能，或 schema 变更但启动时自动迁移
- **PATCH**：bug 修复、文案、依赖小版本

日常提交靠 `main`/`sha-<7>`/`latest` 三条 tag 覆盖，semver tag 只在里程碑/需要固定版本部署时手动 `git tag -a v*` 打。

### 镜像 tag

| 触发 | 产出 | 说明 |
|---|---|---|
| main（CI 通过后） | `main`、`sha-<7>`、`latest` | 由 `workflow_run` 在 CI 成功后触发；失败提交不产出镜像；`latest` 每次绿的 main 自动推进 |
| 手动 `git push v*` | `v2.1.0`、`2.1` | 复制 `sha-<7>`（秒级），不重建；缺镜像才回退完整构建；顺带刷新 latest |
| 手动 dispatch | `edge` | 临时验证，可随时覆盖；不动 `latest` |

日常部署直接拉 `latest`；需要固定版本、跨环境一致性时切 `v<MAJOR>.<MINOR>.<PATCH>`，回滚改这一行即可。

### 构建触发

纯文档提交（`**.md`、`docs/**`、`LICENSE`、`.gitignore`）不触发 CI 与镜像构建；main 镜像在 CI 通过后才由 `workflow_run` 触发，同分支并发会自动取消旧构建。推送前先判断本次改动是否值得跑一次 amd64+arm64 构建。
