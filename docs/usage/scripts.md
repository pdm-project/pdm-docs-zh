# PDM 脚本

与 `npm run` 类似，使用 PDM，你可以在加载本地包的情况下运行任意脚本或命令。

## 任意脚本

```bash
pdm run flask run -p 54321
```

它将在知晓项目环境中的包的环境中运行 `flask run -p 54321`。

## 单文件脚本

+++ 2.16.0

PDM 能够运行带有 [内联脚本元数据](https://peps.python.org/pep-0723/) 的单文件脚本。

以下是一个带有嵌入式元数据的脚本示例：

```python
# test_script.py
# /// script
# requires-python = ">=3.11"
# dependencies = [
#   "requests<3",
#   "rich",
# ]
# ///

import requests
from rich.pretty import pprint

resp = requests.get("https://peps.python.org/api/peps.json")
data = resp.json()
pprint([(k, v["title"]) for k, v in data.items()][:10])
```

当你使用 `pdm run test_script.py`， 运行它时，PDM 会创建一个临时环境，并安装指定的依赖项，然后运行脚本：

```python
[
│   ('1', 'PEP Purpose and Guidelines'),
│   ('2', 'Procedure for Adding New Modules'),
│   ('3', 'Guidelines for Handling Bug Reports'),
│   ('4', 'Deprecation of Standard Modules'),
│   ('5', 'Guidelines for Language Evolution'),
│   ('6', 'Bug Fix Releases'),
│   ('7', 'Style Guide for C Code'),
│   ('8', 'Style Guide for Python Code'),
│   ('9', 'Sample Plaintext PEP Template'),
│   ('10', 'Voting Guidelines')
]
```
如果你想复用上次创建的环境，可以添加 `--reuse-env` 选项。
你还可以在脚本元数据中添加 `[tool.pdm]` 部分来配置 PDM。例如：

```python
# test_script.py
# /// script
# requires-python = ">=3.11"
# dependencies = [
#   "requests<3",
#   "rich",
# ]
#
# [[tool.pdm.source]]  # 使用自定义索引
# url = "https://mypypi.org/simple"
# name = "pypi"
# ///
```

可以阅读这个 [规范](https://packaging.python.org/en/latest/specifications/inline-script-metadata/#inline-script-metadata) 以获取更多详细信息。

## 用户脚本

PDM 还支持在可选的 `[tool.pdm.scripts]` 部分中定义自定义脚本快捷方式。

!!! NOTE "与 `[project.scripts]` 混淆？"
    在 `pyproject.toml` 中有另一个字段 `[project.scripts]`， 并且这些脚本也可以使用 `pdm run` 来调用。它用于定义随包安装的控制台脚本入口点。因此，可执行文件只能在项目本身安装到环境中后才能运行。也就是说，你必须设置  `distribution = true`。

    相比之下， `[tool.pdm.scripts]` 定义了一些在你的项目中要运行的任务。无论 `distribution` 是 `true` 还是 `false` ，它对项目都有效。 这些任务主要用于开发和测试目的，并且支持更多类型和设置，这将在后面展示。你可以将它视为  `Makefile` 的替代品。它不需要项目被安装，但需要存在一个 `pyproject.toml` 文件。

    查看关于 `[project.scripts]` 的更多[解释](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/#creating-executable-scripts)。

然后，你可以运行 `pdm run <script_name>` 以在你的 PDM 项目的上下文中调用脚本。例如：

```toml
[tool.pdm.scripts]
start = "flask run -p 54321"
```

然后在终端中：

```bash
$ pdm run start
Flask server started at http://127.0.0.1:54321
```

随后的参数将附加到命令中：

```bash
$ pdm run start -h 0.0.0.0
Flask server started at http://0.0.0.0:54321
```

!!! note "像 Yarn 一样的脚本快捷方式"
    内置的快捷方式使所有脚本都可用作根命令，只要脚本不与任何内置或插件贡献的命令冲突。
    换句话说，如果你有一个 `start` 脚本，你可以同时运行 `pdm run start` 和 `pdm start`。
    但是如果你有一个 `install` 脚本，只有 `pdm run install` 会运行它，`pdm install` 仍然会运行内置的 `install` 命令。

PDM 支持 4 种类型的脚本：

### `cmd`

纯文本脚本被视为普通命令，或者你可以明确地指定它：

```toml
[tool.pdm.scripts]
start = {cmd = "flask run -p 54321"}
```

在某些情况下，例如想要在参数之间添加注释时，它可能更方便。
要将命令指定为数组而不是字符串，请执行以下操作：

```toml
[tool.pdm.scripts]
start = {cmd = [
    "flask",
    "run",
    # 在这里添加关于始终使用端口 54321 的重要注释
    "-p", "54321"
]}
```

### `shell`

Shell 脚本可用于运行更多与 Shell 相关的任务，例如管道和输出重定向。
这基本上是通过 `subprocess.Popen()` 和 `shell=True` 运行的：

```toml
[tool.pdm.scripts]
filter_error = {shell = "cat error.log|grep CRITICAL > critical.log"}
```

### `call`

脚本也可以定义为调用形式为 `<module_name>:<func_name>` 的 Python 函数：

```toml
[tool.pdm.scripts]
foobar = {call = "foo_package.bar_module:main"}
```

函数可以提供文字参数：

```toml
[tool.pdm.scripts]
foobar = {call = "foo_package.bar_module:main('dev')"}
```

### `composite`

此脚本可执行其他已定义的脚本：

```toml
[tool.pdm.scripts]
lint = "flake8"
test = "pytest"
all = {composite = ["lint", "test"]}
```

运行 `pdm run all` 将首先运行 `lint`，然后如果 `lint` 成功，则运行 `test`。

+++ 2.13.0

要覆盖默认行为并在失败后继续执行剩余的脚本，
将 `keep_going` 选项设置为 `true`：

```toml
[tool.pdm.scripts]
lint = "flake8"
test = "pytest"
all = {composite = ["lint", "test"]}
all.keep_going = true
```

如果 `keep_going` 设置为 `true`，复合脚本的返回代码要么是 '0'（如果所有都成功了），要么是最后一个失败的个别脚本的代码。

你还可以为调用的脚本提供参数：

```toml
[tool.pdm.scripts]
lint = "flake8"
test = "pytest"
all.composite = ["lint mypackage/", "test -v tests/"]
```

!!! note
    在命令行上传递的参数将传递给每个被调用的任务。

您还可以使用 `composite` 脚本来组合多个命令:

```toml
[tool.pdm.scripts]
mytask.composite = [
    "echo 'Hello'",
    "echo 'World'"
]
```

## 脚本选项

### `env`

当前 Shell 中设置的所有环境变量都可以被 `pdm run` 看到，并在执行时展开。
此外，你还可以在你的 `pyproject.toml` 中定义一些固定的环境变量：

```toml
[tool.pdm.scripts]
start.cmd = "flask run -p 54321"
start.env = {FOO = "bar", FLASK_ENV = "1"}
```

注意我们如何使用 [TOML 的语法](https://github.com/toml-lang/toml) 定义一个复合字典。

!!! note "关于环境变量替换"
    脚本规范中的变量可以在所有脚本类型中进行替换。
    在 cmd 脚本中，所有平台上只支持 `${VAR}` 语法，但在 `shell` 脚本中，语法是依赖于平台的。
    例如，Windows cmd 使用 `%VAR%`，而 bash 使用 `$VAR`。

!!! note
    在复合任务级别指定的环境变量将覆盖调用任务定义的环境变量。

### `env_file`

你还可以将所有环境变量存储在一个 dotenv 文件中，并让 PDM 读取它：

```toml
[tool.pdm.scripts]
start.cmd = "flask run -p 54321"
start.env_file = ".env"
```

dotenv 文件中的变量不会覆盖任何现有的环境变量。
如果你希望 dotenv 文件覆盖现有的环境变量，请使用以下命令：

```toml
[tool.pdm.scripts]
start.cmd = "flask run -p 54321"
start.env_file.override = ".env"
```

!!! note "环境变量加载顺序"
    从不同源加载的环境变量，按以下顺序加载：

    1. 操作系统环境变量
    2. 项目环境，如 `PDM_PROJECT_ROOT`, `PATH`, `VIRTUAL_ENV`, etc
    3. 由 `env_file` 指定的 Dotenv 文档
    4. `env` 指定的环境变量映射

    来自后一个源的环境变量将覆盖来自前一个源的环境变量。
    在复合任务级别指定的 dotenv 文件将覆盖调用任务定义的 dotenv 文件。

    环境变量可以包含对之前加载的源中的另一个环境变量的引用，例如：

    ```
    VAR=42
    FOO=hello-${VAR}
    ```
    will result in `FOO=hello-42`. The reference can also contain a default value with the syntax `${VAR:-default}`.

### `working_dir`

+++ 2.13.0

你可以为脚本设置当前工作目录：

```toml
[tool.pdm.scripts]
start.cmd = "flask run -p 54321"
start.working_dir = "subdir"
```

相对路径将相对于项目根目录解析。

+++ 2.20.2

为了识别原始的调用工作目录，每个脚本都会注入环境变量 `PDC_RUN_CWD`。

### `site_packages`

为了确保运行环境与外部 Python 解释器正确隔离，
除非以下任一条件成立，否则不会将所选解释器的 `site-packages` 加载到 `sys.path` 中：

1. 可执行文件来自 `PATH`，但不在 `__pypackages__` 文件夹内。
2. `-s/--site-packages` 标志跟随 `pdm run`.
3. 在脚本表或全局设置键 `_` 中有 `site_packages = true`。

请注意，如果启用了 PEP 582（不带 `pdm run` 前缀），则始终会加载 `site-packages`。

### 共享选项

如果你希望 `pdm run` 运行的所有任务共享选项，
你可以将它们写在 `[tool.pdm.scripts]` 表中的特殊键 `_` 下：

```toml
[tool.pdm.scripts]
_.env_file = ".env"
start = "flask run -p 54321"
migrate_db = "flask db upgrade"
```

此外，在任务中，`PDM_PROJECT_ROOT` 环境变量将被设置为项目根目录。

### 参数占位符

默认情况下，所有用户提供的额外参数都简单地附加到命令（或对于 `composite` 任务的所有命令）。

如果你想更多地控制用户提供的额外参数，你可以使用 `{args}` 占位符。
它适用于所有脚本类型，并且将为每个脚本适当地插入：

```toml
[tool.pdm.scripts]
cmd = "echo '--before {args} --after'"
shell = {shell = "echo '--before {args} --after'"}
composite = {composite = ["cmd --something", "shell {args}"]}
```

将生成以下插值（这些不是真正的脚本，只是为了说明插值）：

```shell
$ pdm run cmd --user --provided
--before --user --provided --after
$ pdm run cmd
--before --after
$ pdm run shell --user --provided
--before --user --provided --after
$ pdm run shell
--before --after
$ pdm run composite --user --provided
cmd --something
shell --before --user --provided --after
$ pdm run composite
cmd --something
shell --before --after
```

如果需要，你可以提供默认值，如果未提供用户参数，则将使用该默认值：

```toml
[tool.pdm.scripts]
test = "echo '--before {args:--default --value} --after'"
```

将产生以下结果：

```shell
$ pdm run test --user --provided
--before --user --provided --after
$ pdm run test
--before --default --value --after
```

!!! note
    一旦检测到占位符，就不再附加参数了。
    这对于 `composite` 脚本很重要，
    因为如果在子任务中检测到占位符，
    那么所有子任务都不会附加参数，
    你需要显式地将占位符传递给每个需要的嵌套命令。

!!! note
    `call` 脚本不支持 `{args}` 占位符，因为它们有
    直接访问 `sys.argv` 来处理如此复杂的情况等等。

### `{pdm}` 占位符

有时你可能有多个 PDM 安装，或者 `pdm` 安装了不同的名称。
这可能发生在 CI/CD 情况下，或者在不同的存储库中使用不同的 PDM 版本时。
为了使你的脚本更加健壮，你可以使用 `{pdm}` 来使用执行脚本的 PDM 入口点。
这将扩展为 `{sys.executable} -m pdm`。

```toml
[tool.pdm.scripts]
whoami = { shell = "echo `{pdm} -V` was called as '{pdm} -V'" }
```

将产生以下输出：

```shell
$ pdm whoami
PDM, version 0.1.dev2501+g73651b7.d20231115 was called as /usr/bin/python3 -m pdm -V

$ pdm2.8 whoami
PDM, version 2.8.0 was called as <snip>/venvs/pdm2-8/bin/python -m pdm -V
```

!!! note
    虽然上面的示例使用了 PDM 2.8，但此功能是在 2.10 系列中引入的，并且仅用于展示。

## 显示脚本列表

使用 `pdm run --list/-l` 来显示可用脚本快捷方式的列表：

```bash
$ pdm run --list
╭─────────────┬───────┬───────────────────────────╮
│ Name        │ Type  │ Description               │
├─────────────┼───────┼───────────────────────────┤
│ test_cmd    │ cmd   │ flask db upgrade          │
│ test_script │ call  │ call a python function    │
│ test_shell  │ shell │ shell command             │
╰─────────────┴───────┴───────────────────────────╯
```

你可以添加一个带有脚本描述的 `help` 选项，它将显示在上述输出的 `Description` 列中。

!!! note
    以下划线（`_`）开头的任务被视为内部（辅助...）任务，并且不会在列表中显示。

## 前后脚本

与 `npm` 类似，PDM 还支持通过前后脚本进行任务组合，前脚本将在给定任务之前运行，后脚本将在之后运行。

```toml
[tool.pdm.scripts]
pre_compress = "{{ Run BEFORE the `compress` script }}" # 在 `compress` 脚本之前运行
compress = "tar czvf compressed.tar.gz data/"
post_compress = "{{ Run AFTER the `compress` script }}"  # 在 `compress` 脚本之后运行
```

在此示例中，`pdm run compress` 将按顺序运行所有这 3 个脚本。

!!! note "管道快速失败"
    在前 - 自身 - 后脚本的管道中，失败将取消后续执行。

## 钩子脚本

在某些情况下，PDM 将寻找一些特殊的钩子脚本进行执行：

- `post_init`: 在 `pdm init` 之后运行
- `pre_install`: 在安装包之前运行
- `post_install`: 在安装包后运行
- `pre_lock`: 在依赖项解析之前运行
- `post_lock`: 在依赖项解析后运行
- `pre_build`: 在构建分发包之前运行
- `post_build`: 在构建分发包后运行
- `pre_publish`: 在发布分发包之前运行
- `post_publish`: 在发布分发包后运行
- `pre_script`: 在任何脚本之前运行
- `post_script`: 在任何脚本之后运行
- `pre_run`: 在运行脚本调用之前运行一次
- `post_run`: 在运行脚本调用之后运行一次

!!! note
    钩子脚本无法接收任何参数。

!!! note "避免名称冲突"
    如果在 `[tool.pdm.scripts]` 表中存在一个 `install` 脚本，
    则 `pre_install` 脚本可以同时被 `pdm install` 和 `pdm run install` 触发。
    因此，建议不要使用保留的名称。

!!! note
    复合任务也可以具有前后脚本。
    调用的任务将运行自己的前后脚本。

## 跳过脚本

有时，希望运行一个脚本，但不运行其钩子或前后脚本，
这时候可以使用 `--skip=:all` 来禁用所有钩子、前后脚本。
还有 `--skip=:pre` 和 `--skip=:post`，
分别允许跳过所有 `pre_*` 钩子和所有 `post_*` 钩子。

还可能需要前脚本但不需要后脚本，
或者需要复合任务的所有任务，除了一个。
对于这些用例，可以使用更精细的 `--skip` 参数，
该参数接受要排除的任务或钩子名称的列表。

```bash
pdm run --skip pre_task1,task2 my-composite
```

此命令将运行 `my-composite` 任务，并跳过 `pre_task1` 钩子以及 `task2` 及其钩子。

你还可以在 `PDM_SKIP_HOOKS` 环境变量中提供跳过列表，
但只要提供了 `--skip` 参数，它就会被覆盖。

关于钩子和前后脚本行为的更多详细信息，请参阅 [专用钩子页面](hooks.md).
