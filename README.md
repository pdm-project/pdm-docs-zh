# pdm-docs-zh

PDM 中文文档（社区维护）

[📖 查看文档](https://pdm-project.org/zh-cn/)

## 维护者

- [mhwy](https://github.com/522247020) 哪里会英语，全靠 AI 翻译，我只是🐂🐎搬运工！

## 贡献

在 `docs/` 目录下找到对应的文档文件，修改后提交 PR 即可。

安装依赖：

```bash
pdm install
```

预览文档：

```bash
pdm doc
```

### 约定

同步翻译目前没有较为方便的方法，所以目前只能手动同步。

1. 只是对原有的英文文档进行了翻译，不对内容二次添加创作。
2. 如有翻译错误，中文内容请提 PR 修改，英文内容在 [PDM](https://github.com/pdm-project/pdm) 进行修改。
3. 翻译文件与英文文档的行保持一致。
4. 翻译文件提交，提交消息为：`sync: 英文文档的提交对象（commit）8位哈希字串`，列如 `sync: 8068a364`。

### Cli 参考汉化说明

使用 Python 的 [多语种国际化服务](https://docs.python.org/zh-cn/3/library/gettext.html)
使用 Python 安装包内的 `msgfmt.py` 文件对翻译文件进行转换，使用方法如下：

```bash
python ...\msgfmt.py .\locale\zh-CN\LC_MESSAGES\cli.po
```

> 注意：需要手动补全 `msgfmt.py`文件路径

转换完成后会生成 `cli.mo` 文件，转换后默认会在同目录产生。
如果出现在其他目录下，则将其放入 `locale\zh-CN\LC_MESSAGES` 目录下。

可以使用工具 Poedit 进行翻译。╮(╯▽╰)╭ 不好用！能帮忙看是否重复翻译，能帮忙转换 mo 文件。

* * *

PDM Docs © 2024 by [PDM maintainers][1] is licensed under [CC BY 4.0][2].

[1]: https://github.com/pdm-project
[2]: https://creativecommons.org/licenses/by/4.0/
