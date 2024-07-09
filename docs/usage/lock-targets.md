# 针对特定平台或 Python 版本的锁定

+++ 2.17.0

默认情况下，PDM会尝试在 [`pyproject.toml` 内的 `requires-python`](./project.md#specify-requires-python) 中指定的 Python 版本内制作一个适用于所有平台的锁文件，这在开发过程中非常方便。您可以在您的开发环境中生成一个锁文件，然后使用这个锁文件在 CI/CD 或生产环境中复制相同的依赖版本。

但是，有时这种方法可能不起作用。例如，您的项目或依赖项具有一些特定于平台的依赖项，或取决于Python版本的条件依赖项，如下所示：

```toml
[project]
name = "myproject"
requires-python = ">=3.8"
dependencies = [
    "numpy<1.25; python_version < '3.9'",
    "numpy>=1.25; python_version >= '3.9'",
    "pywin32; sys_platform == 'win32'",
]
```

在这种情况下，很难为每个包在所有平台和Python版本（>=3.8）上获得单个解析方案。建议而是为特定平台或Python版本创建锁定文件。

## 生成锁定文件时指定锁定目标

生成锁定文件时指定锁定目标

- `--python=<PYTHON_RANGE>`: [PEP 440](https://www.python.org/dev/peps/pep-0440/)兼容的Python版本说明符。例如，`--python=">=3.8，<3.10"` 将为Python版本 `>=3.8` 和 `<3.10` 生成锁定文件。
- `--platform=<PLATFORM>`: 平台说明符。例如，`pdm lock--Platform=linux` 将为 x86_64 平台生成 Linux 锁文件。可用选项有：
    * `linux`
    * `windows`
    * `macos`
    * `alpine`
    * `windows_amd64`
    * `windows_x86`
    * `windows_arm64`
    * `macos_arm64`
    * `macos_x86_64`
    * `macos_X_Y_arm64`
    * `macos_X_Y_x86_64`
    * `manylinux_X_Y_x86_64`
    * `manylinux_X_Y_aarch64`
    * `musllinux_X_Y_x86_64`
    * `musllinux_X_Y_aarch64`
- `--implementation=cpython|pypy|pyston`: Python实现说明符。目前仅支持 `cpython`、`pypy` 和 `pyston`。

您可以忽略一些条件，例如，通过仅指定 `--Platform=linux`，生成的锁定文件将适用于 Linux 平台和所有实现。

!!! note "`python` 标准和 `requires-python`"

    锁目标中的 `--python` 选项或 `requires-python` 标准仍然受到 `pyproject.toml` 中的 `requires-python` 的限制。例如，如果 `requires-python` 是 `>=3.8` 并且您指定了 `--python="<3.11"`，则锁目标将是 `>=3.8，<3.11`。

## 分离锁定文件或合并为一个

如果您需要多个锁定目标，您可以为每个目标创建单独的锁定文件或将它们组合成一个锁定文件。PDM支持两种方式。

要创建具有特定目标的单独锁定文件：

```bash
# 为 Linux 平台和 Python 3.8 生成锁文件，将结果写入 py38-linux.lock
pdm lock --platform=linux --python="==3.8.*" --lockfile=py38-linux.lock
```

在 Linux 和 Python 3.8 上安装依赖项时，可以使用以下锁定文件：

```bash
pdm install --lockfile=py38-linux.lock
```

此外，您还可以为锁文件选择依赖组的子集，请参阅 [here](./lockfile.md#specify-another-lock-file-to-use) 获取更多详细信息。

如果您想对多个目标使用同一个锁定文件，请在 `pdm lock` 命令中添加 `--append`:

```bash
# 为 Linux 平台和 Python 3.8 生成锁文件，将结果附加到 pdm.lock
pdm lock --platform=linux --python="==3.8.*" --append
```

使用单个锁文件的优点是更新依赖项时不需要管理多个锁文件。但是，您不能在单个锁文件中为不同的目标指定不同的锁策略。更新锁的时间成本预计会更高。

更重要的是，每个锁定文件可以有一个或多个锁定目标，使其使用相当灵活。您可以选择在锁定文件中合并一些目标，并在单独的锁定文件中锁定特定的组和目标。我们将在下一节中用一个例子来说明这一点。

## 示例

这是 `pyproject.toml` 的内容:

```toml
[project]
name = "myproject"
requires-python = ">=3.8"
dependencies = [
    "numpy<1.25; python_version < '3.9'",
    "numpy>=1.25; python_version >= '3.9'",
    "pandas"
]

[project.optional-dependencies]
windows = ["pywin32"]
macos = ["pyobjc"]
```

在上面的示例中，我们为 `numpy` 提供了条件依赖版本，并为 Windows 和 MacOS 提供了特定于平台的可选依赖。我们希望为 Linux、Windows 和 MacOS 平台以及 Python 3.8 和 3.9 生成锁定文件。

```bash
pdm lock --python=">=3.9"
pdm lock --python="<3.9" --append

pdm lock --platform=windows --python=">=3.9" --lockfile=py39-windows.lock --with windows
pdm lock --platform=macos --python=">=3.9" --lockfile=py39-macos.lock --with macos
```
按顺序运行上述命令，会得到3个锁文件：

- `pdm.lock`: 默认的主锁定文件，可在所有平台以及 Python 版本 `>=3.8` 上运行。不包含特定平台的依赖项。在此锁定文件中，有两个版本的 `numpy`，分别适用于 Python 3.9 及以上和以下。PDM 安装程序将根据 Python 版本选择正确的版本。
- `py39-windows.lock`: 适用于 Windows 平台和 Python 3.9 及以上的锁定文件，包括 Windows 的可选依赖项。
- `py39-macos.lock`: 适用于 MacOS 平台和 Python 3.9 及以上的锁定文件，包括 MacOS 的可选依赖项。
