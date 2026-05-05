# commit-config

> 声明式提交规范配置仓库 —— 一份**无运行时依赖**的规则与钩子模板，用于在任何 Git 项目中强制执行 [Conventional Commits](https://www.conventionalcommits.org/)。

---

## 仓库内容

```
commit-config/
├── commitlint.config.js    # commitlint 规则文件（通用格式）
├── husky/
│   └── commit-msg          # husky 钩子模板（无需 node_modules）
└── README.md
```

- **`commitlint.config.js`** — 标准的 commitlint 配置，定义了允许的提交类型（`feat`, `fix`, `docs`, …）和其它校验规则。可直接复制到项目根目录，或通过 Home Manager 部署为全局兜底规则。
- **`husky/commit-msg`** — 一个**自包含**的 shell 脚本。它假设 `commitlint` 命令已在 `$PATH` 中（无论是通过 npm 安装还是系统包管理器 / Nix 提供）。将这个脚本放入 `.husky/` 并赋予执行权限，Git 就会在每次提交时调用 `commitlint --edit $1` 进行检查。

**关键设计**：本仓库**不包含任何 `package.json` 或 `node_modules`**。配置文件和钩子本身是纯文本，可以以多种方式消费。

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

# 复制规则文件
cp /path/to/commit-config/commitlint.config.js ./commitlint.config.js

# 安装钩子模板
cp /path/to/commit-config/husky/commit-msg .husky/commit-msg
chmod +x .husky/commit-msg
```

### 2. Nix / Home Manager 环境（推荐）

在 `nix-config` 的 `flake.nix` 中引入该仓库：

```nix
inputs = {
  commit-config = {
    url = "github:Redskaber/commit-config";
    flake = false;
  };
};
```

然后在 Home Manager 模块中部署全局配置，并自动为特定项目安装钩子：

```nix
{ config, lib, pkgs, inputs, ... }:
let
  commitConfigSrc = inputs.commit-config;
in
{
  # 全局工具（脱离 node_modules）
  home.packages = with pkgs; [
    nodePackages.commitlint
    husky
    commitizen          # 可选， cz commit 交互式提交
  ];

  # 全局兜底规则
  home.file.".commitlintrc.js".source = "${commitConfigSrc}/commitlint.config.js";

  # 为 nix-config 仓库安装钩子（示例）
  home.activation.setupCommitHooks = lib.hm.dag.entryAfter [ "writeBoundary" ] ''
    REPO="$HOME/projects/configs/nix-config"
    if [ -d "$REPO/.git" ]; then
      cd "$REPO"
      ${pkgs.git}/bin/git config core.hooksPath .husky
      mkdir -p .husky
      cp "${commitConfigSrc}/husky/commit-msg" .husky/commit-msg
      chmod +x .husky/commit-msg
      rm -f .husky/pre-commit
    fi
  '';
}
```

这之后，任何 `git commit` 不满足规范都会被拒绝，并且全局兜底规则会覆盖用户的所有其他项目。

---

## 提交规范速查

| 类型       | 说明                               |
| ---------- | ---------------------------------- |
| `feat`     | 新功能                             |
| `fix`      | 修复 bug                           |
| `docs`     | 仅文档更改                         |
| `style`    | 不影响代码逻辑的样式变更           |
| `refactor` | 既不修复错误也不添加功能的代码更改 |
| `perf`     | 提高性能的代码更改                 |
| `test`     | 添加缺失测试或更正现有测试         |
| `chore`    | 对构建过程或辅助工具的更改         |
| `revert`   | 回滚之前的提交                     |
| `ci`       | 持续集成配置的更改                 |

**示例**：

- `feat: add user authentication`
- `fix: resolve crash on empty input`
- `docs: update README installation guide`

交互式提交可使用 `cz commit`（如果安装了 commitizen）或 `npx cz`。

---

## 常见问题

**Q：钩子脚本中为什么是 `commitlint --edit $1` 而不使用 `npx`？**  
A：因为 Nix 已经将 `commitlint` 安装到了全局 `PATH`，直接调用即可。在传统 Node 项目中，也可以将钩子改为 `npx --no -- commitlint --edit $1` 以确保使用本地 `node_modules` 中的版本。

**Q：如何更新规则？**  
A：修改本仓库的 `commitlint.config.js` 并推送。然后通过 `nix flake update commit-config` 和 `home-manager switch`，或直接重新复制到项目目录。

**Q：如果我的项目有自己的 `commitlint.config.js` 怎么办？**  
A：项目级配置优先级更高。本仓库的全局配置仅作为没有项目级配置时的兜底。

---

## 许可

MIT

---

> 让每一次 `git commit` 都成为声明式工程的一砖一瓦。
