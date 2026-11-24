# dsh-sync-plugins

一条命令，把 DSH profile 里的 `@deepseek-ai/dsh-*` 插件对齐到当前 `dsh` 运行时版本。

DSH（DeepSeek Harness）升级 `dsh` 本身之后，装在 profile 里的可选插件（比如 Claude Code hooks 桥接）不会跟着升级。版本一错开，DSH 启动时的 preflight 会把整行**静默禁用**——只在 stderr 打一句提示，界面上看起来就是"插件莫名其妙不工作了"。这个脚本把这件事变成升级后跑一条命令。

## 背景：为什么插件不跟着 dsh 升级

- 从 dsh `0.2.1-alpha.1` 起，CLI 安装包不再包含 hook 桥接。`@deepseek-ai/dsh-hooks-claude-code` 这类可选插件需要显式装进 profile（见 DSH 升级指南《可选能力的显式安装与组合》）。
- 全局安装的 `dsh` 和 `~/.dsh/profiles/<名字>` 是两套东西：升级只替换全局 npm 目录，不动 profile 的 `package.json` / `pnpm-lock.yaml`。
- profile 里的依赖写的是精确版本（如 `0.2.2-rc.5`，没有 `^`），pnpm 不会自己挪。
- 插件声明的 peer 依赖也是精确版本。插件版本与运行时不一致时，DSH 启动的 preflight 会禁用整个插件行。

对照：`@deepseek-ai/dsh-mcp-client` 这类 CLI 正式依赖随 dsh 自动升级，不需要本脚本。

## 要求

- Linux / macOS，bash 4.4+
- `dsh` 在 PATH 上
- `node`（读取 `package.json`、解析版本）
- 能访问 npm registry（只在需要更新时用）

## 安装

```bash
install -m 755 dsh-sync-plugins ~/.local/bin/
```

确保 `~/.local/bin` 在 `PATH` 里。也可以直接 `./dsh-sync-plugins` 运行。

## 用法

```bash
dsh-sync-plugins                  # 同步默认 profile（web）
dsh-sync-plugins --dry-run        # 只显示要改什么，不动文件
dsh-sync-plugins --profile tui    # 同步别的 profile
dsh-sync-plugins --help           # 用法
```

典型流程：升级 dsh → 跑 `dsh-sync-plugins` → 重启 `dsh web`（Ctrl+C 后重新启动）。

| 退出码 | 含义 |
| --- | --- |
| `0` | 已同步，或同步完成 |
| `1` | 安装失败 / 自检未通过 / profile 不存在 |
| `2` | 参数错误 |

环境变量：`DSH_PROFILE`（默认 profile 名）、`DSH_HOME`（dsh 家目录，默认 `~/.dsh`）。

## 工作原理

1. 读 `dsh --version` 得到运行时版本；
2. 从 profile 的 `package.json` 的 `dependencies` 里找出所有 `@deepseek-ai/dsh-*` 依赖；
3. 版本与运行时不符的，执行 `dsh plugin --profile <profile> add <包>@<运行时版本>`；
4. 自检：`node_modules` 里实际装上的版本、以及插件 peer 里写死的精确版本，都必须等于运行时版本。

已同步时脚本不调用 pnpm、不下载任何东西，只打印检查结果。

## 手动兜底

脚本用不了就直接敲：

```bash
dsh plugin --profile web add \
  @deepseek-ai/dsh-hooks-claude-code@$(dsh --version) \
  @deepseek-ai/dsh-hook-protocol@$(dsh --version)
```

校验：

```bash
dsh plugin --profile web list --depth 0
```

两个包都显示当前 dsh 版本就对了。重启后到插件管理页面确认对应插件是 active。

## 注意

- **别省略版本号**。不写版本会装 npm 上 `latest` 标签的版本，而 `dsh-hooks-claude-code` / `dsh-hook-protocol` 的 `latest` 还停在很旧的 `0.0.1-rc.x`（更新的 `0.2.3-alpha.x` 在 `alpha` 标签下），装了照样对不上运行时。
- 顺序无所谓：先装插件再升级 dsh 也行，只要最后两边版本一致。
- 自编译 / 未发布的 dsh 版本在 npm 上装不到，那时需要走 DSH 的版本豁免或本地构建。
- 脚本不会重启任何进程；重启 dsh 是新版本生效的唯一方式。

## 相关路径（dsh 默认布局）

| 东西 | 路径 |
| --- | --- |
| profile | `~/.dsh/profiles/<名字>/`（`package.json`、`pnpm-lock.yaml`） |
| 插件挂载点 | `~/.dsh/profiles/<名字>/cordis.patch.yml` |
| 机器级 patch | `~/.dsh/cordis.patch.yml` |

## 变更记录

**2026-11-24**

- 首个版本，从一次真实的 dsh `0.2.2-rc.1 → 0.2.2-rc.5` 升级事故里抽出来：当时 web profile 的两个 hooks 包停在 `rc.1`，`hooks-claude-code` 已被 preflight 禁用。
- 同时验证了 npm 上这两个包 `latest` 标签停留在 `0.0.1-rc.x` 的事实，写进"注意"一节。
