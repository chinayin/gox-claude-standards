# gox-claude-standards

团队 AI 协作规范的**版本源**：`steering/`（领域规范）+ `CLAUDE.template.md`（项目入口模板）。

由 [`goxctl claude`](https://github.com/chinayin/goxctl-claude) 按 git tag 拉取到项目本地
（`.kiro/steering` + `CLAUDE.md`），供 Kiro 与 Claude Code 共用。**规范版本号即本仓库的 tag**，
与工具（goxctl-claude 二进制）的版本线相互独立。

## 内容

- `steering/` — 规范文件（当前为 Go 微服务规范：`rules` / `cli` / `config` / `db-migrations` / `scaffold`，
  以及语言无关的 `karpathy-guidelines`）。
- `CLAUDE.template.md` — 项目顶层入口模板：通用框架 + 领域条件化（always-on 只引用通用准则，
  Go 规范在「Go projects」节软引用）。

## 使用（在项目里）

```bash
goxctl claude add               # 默认拉本仓库最新 release 并钉住
goxctl claude update            # 升级到最新
goxctl claude update v0.2.0     # 锁定到指定版本
goxctl claude check             # 校验本地规范与 lock 一致（CI）
```

## 维护

改规范 → commit → 打 tag（语义化：规范破坏性 `major` / 新增 `minor` / 修订 `patch`）→ push。
`release` workflow 自动创建 GitHub release，供 `goxctl claude` 解析最新版本与下载。
