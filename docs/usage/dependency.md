# 管理依赖项

PDM 提供了一系列有用的命令，帮助您管理项目和依赖项。以下示例在 Ubuntu 18.04 上运行，
如果您使用 Windows，则需要进行一些更改。

## Add dependencies

[`pdm add`](../reference/cli.md#add) 后面可以跟着一个或多个依赖项，依赖项的规范在 [PEP 508](https://www.python.org/dev/peps/pep-0508/) 中描述。

示例：

```bash
pdm add requests   # 添加 requests
pdm add requests==2.25.1   # 添加带有版本约束的 requests
pdm add requests[socks]   # 添加带有额外依赖项的 requests
pdm add "flask>=1.0" flask-sqlalchemy   # 添加多个具有不同规范的依赖项
```

PDM 还允许通过提供 `-G/--group <name>` 选项来添加额外的依赖项组，
这些依赖项将分别进入项目文件中的 `[project.optional-dependencies.<name>]` 表。

您可以在 `optional-dependencies` 中引用其他可选组，在上传包之前甚至可以引用它们：

```toml
[project]
name = "foo"
version = "0.1.0"

[project.optional-dependencies]
socks = ["pysocks"]
jwt = ["pyjwt"]
all = ["foo[socks,jwt]"]
```

之后，依赖项和子依赖项将被正确解析并安装，您可以查看 `pdm.lock` 查看所有依赖项的解析结果。

### 本地依赖项

可以使用其路径添加本地包。路径可以是文件或目录：

```bash
pdm add ./sub-package
pdm add ./first-1.0.0-py2.py3-none-any.whl
```

路径必须以 `.` 开头，否则它将被视为普通的命名要求。本地依赖项将以 URL 格式写入 `pyproject.toml` 文件：

```toml
[project]
dependencies = [
    "sub-package @ file:///${PROJECT_ROOT}/sub-package",
    "first @ file:///${PROJECT_ROOT}/first-1.0.0-py2.py3-none-any.whl",
]
```

??? note "使用其他构建后端"
    如果您使用的是  `hatchling` 而不是 PDM 后端，URL 将如下所示：

    ```
    sub-package @ {root:uri}/sub-package
    first @ {root:uri}/first-1.0.0-py2.py3-none-any.whl
    ```
    其他后端不支持在 URL 中编码相对路径，并将写入绝对路径。

### URL 依赖项

PDM 还支持直接从 Web 地址下载和安装包。

示例：

```bash
# 从普通 URL 安装 gzipped 包
pdm add "https://github.com/numpy/numpy/releases/download/v1.20.0/numpy-1.20.0.tar.gz"
# 从普通 URL 安装 wheel 包
pdm add "https://github.com/explosion/spacy-models/releases/download/en_core_web_trf-3.5.0/en_core_web_trf-3.5.0-py3-none-any.whl"
```

### VCS 依赖项

您还可以从 git 存储库 URL 或其他版本控制系统安装。支持以下内容：

- Git: `git`
- Mercurial: `hg`
- Subversion: `svn`
- Bazaar: `bzr`

URL 应该是这样的：`{vcs}+{url}@{rev}`

示例：

```bash
# 在标签 `22.0` 上安装 pip 存储库
pdm add "git+https://github.com/pypa/pip.git@22.0"
# 在 URL 中提供凭据
pdm add "git+https://username:password@github.com/username/private-repo.git@master"
# 为依赖项命名
pdm add "pip @ git+https://github.com/pypa/pip.git@22.0"
# 或使用 #egg 片段
pdm add "git+https://github.com/pypa/pip.git@22.0#egg=pip"
# 从子目录安装
pdm add "git+https://github.com/owner/repo.git@master#egg=pkg&subdirectory=subpackage"
```

要对 git 使用 ssh 方案，只需将 "https://" 替换为 "ssh://git@"

列如：

```bash
pdm add "wheel @ git+ssh://git@github.com/pypa/wheel.git@main"
```

### 在 URL 中隐藏凭据

您可以使用 `${ENV_VAR}` 变量语法在 URL 中隐藏凭据：

```toml
[project]
dependencies = [
  "mypackage @ git+http://${VCS_USER}:${VCS_PASSWD}@test.git.com/test/mypackage.git@master"
]
```

在安装项目时，这些变量将从环境变量中读取。

### 添加仅用于开发的依赖项

+++ 1.5.0

PDM 也支持定义对开发有用的依赖组。
例如，一些用于测试，另一些用于代码检查。我们通常不希望这些依赖项出现在发行版的元数据中。
我们通常不希望这些依赖项出现在分发的元数据中，因此使用 `optional-dependencies` 可能不是一个好主意。我们可以将它们定义为开发依赖项：

```bash
pdm add -dG test pytest
```

这将导致 pyproject.toml 如下：

```toml
[tool.pdm.dev-dependencies]
test = ["pytest"]
```

您可以有几个开发依赖项组。与 `optional-dependencies` 类似，它们不会出现在包分发的元数据中，例如 `PKG-INFO` 或 `METADATA`。
包索引不会知道这些依赖项。模式类似于 `optional-dependencies`，只不过在 `tool.pdm` 表中。

```toml
[tool.pdm.dev-dependencies]
lint = [
    "flake8",
    "black"
]
test = ["pytest", "pytest-cov"]
doc = ["mkdocs"]
```

为了向后兼容，如果只指定了 `-d` 或 `--dev`，则依赖项将默认进入 `[tool.pdm.dev-dependencies]` 下的 `dev` 组。

!!! NOTE
    相同的组名不能同时出现在 `[tool.pdm.dev-dependencies]` 和 `[project.optional-dependencies]` 中。

### 可编辑的依赖项

**本地目录** 和 **VCS 依赖项** 可以以 [可编辑模式](https://pip.pypa.io/en/stable/cli/pip_install/#editable-installs) 安装。如果您熟悉 `pip`，那么这就像是 `pip install -e <package>`。可编辑包仅允许在开发依赖项中使用：

!!! NOTE
    只有在 `dev` 依赖组中允许可编辑安装。其他组，包括默认组，将以 `[PdmUsageError]` 失败。

```bash
# 相对路径到目录
pdm add -e ./sub-package --dev
# 到本地目录的文件 URL
pdm add -e file:///path/to/sub-package --dev
# VCS URL
pdm add -e git+https://github.com/pallets/click.git@main#egg=click --dev
```

### 保存版本规范

如果未给出带有版本规范的包，如 `pdm add requests`。
PDM 提供了三种不同的行为来保存依赖项的版本规范，
即通过 `--save-<strategy>` 指定（假设 2.21.0 是可以找到的最新版本规范的版本）：

- `minimum`: 保存最低版本规范：`>=2.21.0`（默认）。
- `compatible`: 保存兼容版本规范：`>=2.21.0,<3.0.0`。
- `exact`: 保存精确版本规范：`==2.21.0`。
- `wildcard`: 不约束版本，并将规范保留为通配符：`*`。

### 添加预发布版本

可以给 [`pdm add`](../reference/cli.md#add) 提供 `--pre/--prerelease` 选项，以便允许为给定的软件包固定预发布版本。

## 更新现有依赖项

要更新锁文件中的所有依赖项：

```bash
pdm update
```

要更新指定的软件包：

```bash
pdm update requests
```

要更新多个依赖项组：

```bash
pdm update -G security -G http
```

或使用逗号分隔的列表：

```bash
pdm update -G "security,http"
```

要在指定组中更新给定的软件包：

```bash
pdm update -G security cryptography
```

如果未给出组，则 PDM 将在默认依赖项集中搜索要求，并且如果未找到任何要求，则会引发错误。

要更新开发依赖项中的软件包：

```bash
# 更新所有默认 + 开发依赖项
pdm update -d
# 更新指定组的开发依赖项中的软件包
pdm update -dG test pytest
```

### 关于更新策略

类似地，PDM 还提供了更新依赖项和子依赖项的 3 种不同行为，
通过 `--update-<strategy>` 选项指定：

- `reuse`: 保留所有已锁定的依赖项，除了命令行中给定的依赖项（默认）。
- `reuse-installed`: 尝试重用安装在工作集中的版本。**这也会影响命令行中请求的软件包**。
- `eager`: 尝试锁定命令行中的软件包及其递归子依赖项的较新版本，并保留其他依赖项不变。
- `all`: 更新所有依赖项和子依赖项。

### 将软件包更新忽略 pyproject.toml 中的版本规范

可以给 `-u/--unconstrained` 选项告诉 PDM 忽略 `pyproject.toml` 中的版本规范。
这类似于 `yarn upgrade -L/--latest` 命令。此外，
[`pdm update`](../reference/cli.md#update_2) 还支持 `--pre/--prerelease` 选项。

## 删除现有依赖项

从项目文件和库目录中删除现有依赖项：

```bash
# 从默认依赖项中删除 requests
pdm remove requests
# 从可选依赖项的 'web' 组中删除 h11
pdm remove -G web h11
# 从开发依赖项的 `test` 组中删除 pytest-cov
pdm remove -dG test pytest-cov
```

## 列出过时的软件包和最新版本

+++ 2.13.0

要列出过时的软件包和最新版本：

```bash
pdm outdated
```

您可以传递 **通配符** 模式来过滤要显示的软件包：

```bash
pdm outdated requests* flask*
```

## 选择要安装的依赖项组的子集

假设我们有一个具有以下依赖项的项目：

```toml
[project]  # 这是生产依赖项
dependencies = ["requests"]

[project.optional-dependencies]  # 这是可选依赖项
extra1 = ["flask"]
extra2 = ["django"]

[tool.pdm.dev-dependencies]  # 这是开发依赖项
dev1 = ["pytest"]
dev2 = ["mkdocs"]
```

| 命令                             | 功能                                                                 | 注释                  |
| ------------------------------- | -------------------------------------------------------------------- | ------------------------- |
| `pdm install`                   | 安装锁定在 lockfile 中的所有组                                        |       \                    |
| `pdm install -G extra1`         | 安装生产依赖项、开发依赖项和 "extra1" 可选组                           |              \             |
| `pdm install -G dev1`           | 安装生产依赖项和仅 "dev1" 开发组                                        |          \                 |
| `pdm install -G:all`            | 安装生产依赖项、开发依赖项和 "extra1"、"extra2" 可选组                   |           \                |
| `pdm install -G extra1 -G dev1` | 安装生产依赖项、"extra1" 可选组和仅 "dev1" 开发组                        |           \                |
| `pdm install --prod`            | 仅安装生产依赖项                                                    |             \              |
| `pdm install --prod -G extra1`  | 仅安装生产依赖项                                                     |              \             |
| `pdm install --prod -G dev1`    | 失败，`--prod` 不能与开发依赖项一起使用                  | 留下 `--prod` 选项 |

只要未传递 `--prod`，**所有开发依赖项都将被包含**，而 `-G` 没有指定任何开发组。

此外，如果您不希望根项目被安装，可以添加 `--no-self` 选项；当您希望所有软件包都以非可编辑版本安装时，可以使用 `--no-editable`。

您也可以在这些选项中使用 `pdm lock` 命令，这样只会锁定指定的组，这些组将记录在锁文件的 `[metadata]` 表中。如果未指定 `--group/--prod/--dev/--no-default` 选项，则 `pdm sync` 和 `pdm update` 将使用锁文件中的组。但是，如果在命令中提供了任何未包含在锁文件中的组，PDM 将引发错误。

## 依赖项覆盖

如果某个特定包的所有版本都不满足所有约束条件，解析将会失败。在这种情况下，您可以通过依赖项覆盖告诉解析器使用该包的特定版本。

在某些情况下，如果用户知道某个依赖项与某个包的更新版本兼容，但该包尚未更新以声明这种兼容性，那么覆盖将是一种有用的最终手段。

例如，如果一个传递依赖声明 `pydantic>=1.0,<2.0` ，但用户知道该包与 `pydantic>=2.0` 兼容，用户可以用 `pydantic>=2.0,<3` 覆盖声明的依赖，以使解析器能够继续。

在 PDM 中，有两种指定覆盖的方式：

### 在项目文件中

+++ 1.12.0

您可以在 `pyproject.toml` 文件中，在 `[tool.pdm.resolution.overrides]` 表下指定覆盖项：

```toml
[tool.pdm.resolution.overrides]
asgiref = "3.2.10"  # 确切版本
urllib3 = ">=1.26.2"  # 版本范围
pytz = "https://mypypi.org/packages/pytz-2020.9-py3-none-any.whl"  # 绝对 URL
```

该表中的每个条目都是一个包名称和一个版本说明符。版本说明符可以是一个版本范围、一个确切的版本或一个绝对 URL 。

### 通过命令行选项

+++ 2.17.0

PDM 还支持从需求文件中读取依赖项覆盖。该文件的工作方式类似于 pip 中的约束文件（`--constraint constraints.txt`），并且语法与需求文件相同：

```
requests==2.20.0
django==1.11.8
certifi==2018.11.17
chardet==3.0.4
idna==2.7
pytz==2019.3
urllib3==1.23
```

覆盖文件是一种简便的方式，用于将依赖项存储在一个集中的位置，您所在组织中的多个项目都可以共享。

您可以将约束文件传递给各种会执行解析的 PDM 命令，例如 [`pdm install`](../reference/cli.md#install)、[`pdm lock`](../reference/cli.md#lock)、[`pdm add`](../reference/cli.md#add) 等等。

```bash
pdm lock --override constraints.txt
```

此选项可以多次提供。

覆盖文件也可以通过 URL 提供，例如 `--override http://example.com/constraints.txt` ，以便您的组织能够在远程服务器中存储和提供它们。

## 显示已安装的软件包

类似于 `pip list`，您可以列出安装在软件包目录中的所有软件包：

```bash
pdm list
```

### 包括和排除组

默认情况下，将列出工作集中安装的所有软件包。
您可以通过 `--include/--exclude` 选项指定要列出的组，其中 `include` 优先级高于 `exclude`。

```bash
pdm list --include dev
pdm list --exclude test
```

存在一个特殊的组 `:sub`，如果包含，则还会显示所有传递依赖项。它默认已包含。

您还可以将 `--resolve` 传递给 `pdm list`，这将显示在 `pdm.lock` 中解析的软件包，而不是安装在工作集中的软件包。

### 更改输出字段和格式

默认情况下，列表输出将显示名称、版本和位置，您可以通过 `--fields` 选项查看更多字段或指定字段的顺序：

```bash
pdm list --fields name,licenses,version
```

有关所有支持的字段，请参阅  [CLI 参考](../reference/cli.md#list_1)。

此外，您还可以指定除默认表输出外的输出格式。支持的格式和选项有 `--csv`、`--json`、`--markdown` 和 `--freeze`。

### 显示依赖树

或通过以下方式显示依赖树：

```shell
$ pdm list --tree
tempenv 0.0.0
└── click 7.0 [ required: <7.0.0,>=6.7 ]
black 19.10b0
├── appdirs 1.4.3 [ required: Any ]
├── attrs 19.3.0 [ required: >=18.1.0 ]
├── click 7.0 [ required: >=6.5 ]
├── pathspec 0.7.0 [ required: <1,>=0.6 ]
├── regex 2020.2.20 [ required: Any ]
├── toml 0.10.0 [ required: >=0.9.4 ]
└── typed-ast 1.4.1 [ required: >=1.4.0 ]
bump2version 1.0.0
```

请注意，`--fields` 选项与 `--tree` 不兼容。

### 使用模式筛选软件包

您还可以通过将模式传递给 pdm list 来限制要显示的软件包：

```bash
pdm list flask-* requests-*
```

??? warning "小心使用 shell 展开"
    在大多数 `shell` 中，通配符 `*` 将在当前目录下有匹配文件时展开。
    为了避免获得意外结果，您可以使用单引号将模式括起来：`pdm list 'flask-*' 'requests-*'`。

在 `--tree` 模式下，只会显示匹配模式的包及其传递依赖项。这可以用来实现与 `pnpm why` 相同的目的，即显示为什么需要特定的包。

```bash
$ pdm list --tree --reverse certifi
certifi 2023.7.22
└── requests 2.31.0 [ requires: >=2017.4.17 ]
    └── cachecontrol[filecache] 0.13.1 [ requires: >=2.16.0 ]
```

## 管理全局项目

有时，用户可能还想跟踪全局 Python 解释器的依赖关系。
使用 PDM 可以通过大多数子命令支持的 `-g/--global` 选项轻松实现此目的。

如果传递该选项， `<CONFIG_ROOT>/global-project` 将用作项目目录，
这与普通项目几乎相同，只是 `pyproject.toml` 会自动为您创建并且不支持构建功能。
这个想法取自 `Haskell` 的堆栈。

但是，与 `stack` 不同的是，默认情况下，如果未找到本地项目，PDM 不会自动使用全局项目。
用户应该显式地传递 `-g/--global` 来激活它，因为如果包发送到错误的地方，那就不太令人愉快了。
但 PDM 也将决定权留给用户，只需将配置 `global_project.fallback` 设置为 `true` 即可。

默认情况下，当 pdm 隐式使用全局项目时，会打印以下消息： `Project is not found, fallback to the global project` 。
要禁用此消息，请将配置 `global_project.fallback_verbose` 设置为 `false` 。

如果您希望全局项目跟踪 `<CONFIG_ROOT>/global-project` 之外的另一个项目文件，
您可以通过 `-p/--project <path>` 选项提供项目路径。
特别是如果您传递 `--global --project .`，
PDM 会将当前项目的依赖项安装到全局 Python 中。

!!! warning
    使用全局项目时要小心 `remove` 和 `sync --clean/--pure` 命令，因为它可能会删除系统 Python 中安装的包。
