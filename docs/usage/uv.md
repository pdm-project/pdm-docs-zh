# 使用 uv（实验性）

PDM 对 [uv](https://github.com/astral-sh/uv) 作为解析器和安装器有实验性支持。要启用它：

```
pdm config use_uv true
```

PDM 将自动检测系统上的 `uv`  二进制文件。你需要先安装 `uv`。更多详细信息请参阅 [uv 的安装指南](https://docs.astral.sh/uv/getting-started/installation/)。

## 局限性

尽管 uv 带来了显著的性能提升，但需要注意 uv 的以下局限性：

- 缓存文件存储在 uv 自己的缓存目录中，你必须使用 `uv` 命令来管理它们。
- 不支持 PEP 582 本地包布局。
- uv 不支持 `inherit_metadata` 锁定策略。 在写入锁定文件时，这将被忽略。
- 不支持除 `all` 和 `reuse` 之外的更新策略。
- 可编辑需求必须是本地路径。像 `-e git+<git_url>` 这样的需求不被支持。