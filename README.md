# commit-config

> 声明式提交规范配置仓库 —— 一份**无运行时依赖**的规则、钩子模板与交互式配置，用于在任何 Git 项目中强制执行 [Conventional Commits](https://www.conventionalcommits.org/)。

---

## 仓库内容

```
commit-config/
├── .cz.toml                # Commitizen 交互式配置（类型列表与描述）
├── commitlint.config.js    # commitlint 规则文件（通用格式）
├── husky/
│   └── commit-msg          # husky 钩子模板（无需 node_modules）
└── README.md
```

- **`commitlint.config.js`** — 标准的 commitlint 配置，定义了允许的提交类型（`feat`、`fix`、`docs`、`build`…）及其它校验规则。可直接复制到项目根目录，或通过 Home Manager / `just` 部署。
- **`husky/commit-msg`** — 一个**自包含**的 shell 脚本。它假设 `commitlint` 命令已在 `$PATH` 中（无论是通过 Nix、Homebrew 或系统包管理器提供）。将该脚本放入 `.husky/` 并赋予执行权限，Git 就会在每次提交时调用 `commitlint --edit $1` 进行检查。
- **`.cz.toml`** — 用于 `commitizen`（Python 版 `cz`）的配置文件。它定义了交互式提交时的类型选项和描述，确保 `cz commit` 的可选类型与 `commitlint` 校验规则完全一致，两者**共用一个类型标准**。

**关键设计**：本仓库**不包含任何 `package.json` 或 `node_modules`**。配置文件和钩子均为纯文本，能够以多种方式消费——无论你是使用传统的 npm 项目，还是 Nix 托管的全局工具环境。

---

## 使用场景

### 1. 传统 Node.js 项目

```bash
# 在项目根目录
git init
npm init -y
npm install --save-dev husky @commitlint/cli @commitlint/config-conventional

# 初始化 husky
npx husky init
# 确保 hooksPath 指向 .husky
git config core.hooksPath .husky

# 复制规则文件和交互式配置
cp /path/to/commit-config/commitlint.config.js ./commitlint.config.js
cp /path/to/commit-config/.cz.toml ./.cz.toml

# 安装钩子模板
cp /path/to/commit-config/husky/commit-msg .husky/commit-msg
chmod +x .husky/commit-msg
```

### 2. Nix + `just` 环境（推荐，零 `node_modules`）

此方式依赖 **Nix** 全局安装的 `commitlint` 和 `commitizen`，并利用 `just` 自动化部署配置文件和钩子。这是作者 `nix-config` 仓库的实践方案。

#### 2.1 前置条件

- 已通过 Nix 安装 `commitlint` 与 `commitizen`（例如在 Home Manager 中配置 `home.packages = [ pkgs.nodePackages.commitlint pkgs.commitizen pkgs.husky ]`）。
- 目标项目中可通过 `nix eval` 获取 `commit-config` 的 store 路径，或直接克隆本仓库。

#### 2.2 快速部署（使用 `just`）

在 `nix-config` 中已封装好 `commit.just` 模块，可直接调用：

```bash
# 为当前项目部署规则文件与钩子（不含全局兜底）
just commit-setup
```

该命令会：

1. 复制 `commitlint.config.js` 到项目根目录。
2. 复制 `.cz.toml` 到项目根目录。
3. 设置 `core.hooksPath` 指向 `.husky`，并安装 `commit-msg` 钩子。
4. 清理可能干扰的 `pre-commit` 钩子。

如果你想为任意 Git 项目部署同样的规范：

```bash
# 仅为任意项目安装 husky 钩子（不复制配置文件）
just commit-husky-install ~/path/to/your/repo

# 单独复制 commitlint 和 commitizen 配置文件到项目
just commit-project-rules ~/path/to/your/repo
```

如果你希望所有项目共享一套全局兜底规则（单次执行）：

```bash
just commit-global-rules
```

这会将 `commitlint.config.js` 写入 `~/.commitlintrc.js`，所有没有自己 `commitlint.config.js` 的项目都将使用该规则。

#### 2.3 在 `nix-config` 中的集成原理

`nix-config` 的 `flake.nix` 中已声明：

```nix
inputs = {
  commit-config = {
    url = "github:Redskaber/commit-config";
    flake = false;
  };
};
```

并通过 `api.inputs` 暴露路径，`scripts/just/commit.just` 从 `nix eval .#api.inputs --json` 动态获取本仓库在 Nix store 中的路径，实现完全数据驱动的部署。

---

## 提交规范速查

| 类型       | 说明                                                   |
| ---------- | ------------------------------------------------------ |
| `feat`     | 新功能                                                 |
| `fix`      | 修复 bug                                               |
| `docs`     | 仅文档更改                                             |
| `style`    | 不影响代码逻辑的样式变更（空格、格式化、缺少分号等）   |
| `refactor` | 既不修复错误也不添加功能的代码更改                     |
| `perf`     | 提高性能的代码更改                                     |
| `test`     | 添加缺失测试或更正现有测试                             |
| `build`    | 影响构建系统或外部依赖的更改（例如：pip, docker, npm） |
| `ci`       | 持续集成配置的更改                                     |
| `chore`    | 其他不修改 src 或测试文件的更改（辅助工具、配置等）    |
| `revert`   | 回滚之前的提交                                         |

**提交示例**：

- `feat: add user authentication`
- `fix: resolve crash on empty input`
- `docs: update README installation guide`
- `build: update docker base image`

交互式提交可使用 `cz commit`（Python 版 Commitizen）或 `npx cz`（Node 版）。本仓库的 `.cz.toml` 会确保你选择的类型与校验规则一致。

---

## 常见问题

**Q：为什么钩子里是 `commitlint --edit $1` 而不是 `npx`？**  
A：在 Nix 环境中，`commitlint` 已全局安装，直接调用即可。传统 Node 项目中可改为 `npx --no -- commitlint --edit $1` 以使用本地 `node_modules`。

**Q：如何同时保证 `cz commit` 的选项和 commitlint 的规则一致？**  
A：本仓库提供了 `.cz.toml` 与 `commitlint.config.js`，两者使用相同的类型列表。部署到项目后，`cz` 会读取 `.cz.toml` 限制可选择类型，而 commitlint 则根据 `commitlint.config.js` 校验，确保完全统一。

**Q：如果我的项目有自己的 `commitlint.config.js` 怎么办？**  
A：项目级配置优先级更高。本仓库的全局配置 (`~/.commitlintrc.js`) 仅在没有项目级配置时生效。如果你想覆盖某些规则，直接修改项目内的 `commitlint.config.js` 即可。

**Q：如何更新规范？**  
A：修改本仓库的文件并推送，然后在 `nix-config` 中执行 `nix flake update commit-config`，最后重新运行 `just commit-setup`（或单独运行 `just commit-project-rules` 和 `just commit-husky-install`）即可同步最新规范。

---

## 许可

MIT

---

> 让每一次 `git commit` 都成为声明式工程的一砖一瓦。
