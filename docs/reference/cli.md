# 命令行参考

```python exec="1" idprefix=""
import gettext
import argparse
import re
from pathlib import Path
from pdm.core import Core


localedir = Path("locale")
t = gettext.translation('cli', localedir=localedir, languages=['zh-CN'])


parser = Core().parser  # 获取核心代码参数解析对象

MONOSPACED = ("pyproject.toml", "pdm.lock", ".pdm-python", ":pre", ":post", ":all")

def clean_help(help: str) -> str:
    # 使 dunders 单间距避免斜体标记渲染
    help = re.sub(r"__([\w\d\_]+)__", r"`__\1__`", help)
    # 使 env vars 保持单间距
    help = re.sub(r"env var: ([A-Z_]+)", r"env var: `\1`", help)
    for monospaced in MONOSPACED:
        help = re.sub(rf"\s(['\"]?{monospaced}['\"]?)", f"`{monospaced}`", help)
    return help


def render_parser(
    parser: argparse.ArgumentParser,
    title: str,
    heading_level: int = 2
) -> str:
    """将解析器帮助文档渲染为字符串。
    
    parser: 这是一个argparse.ArgumentParser对象，包含了命令行接口的定义
    title: 输出文档的标题。
    heading_level: 标题的级别（如Markdown中的#号数量），默认为2。
    """
    # 初始化结果列表:
    #   创建一个名为result的空列表，用于存储格式化后的帮助文档行。
    # 添加标题:
    #   根据title和parser.description，在result列表中添加标题行。如果title不是"pdm"且description存在，还会添加一个带有描述的引用行。
    result = [f"{'#' * heading_level} {title}\n"]
    # 如果title不是"pdm"且description存在，则添加一个带有描述的引用行。
    if parser.description and title != "pdm":
        result.append("> " + t.gettext(parser.description) + "\n")  # 说明汉化
    # 遍历所有动作组，并根据标题进行排序。
    for group in sorted(
        # 获取所有动作组，并根据标题进行排序。
        parser._action_groups, key=lambda g: g.title.lower(), reverse=True
    ):
        # 如果动作组没有任何选项字符串、目标或子解析器动作，跳过该组。
        if not any(
            bool(action.option_strings or action.dest)
            or isinstance(action, argparse._SubParsersAction)
            for action in group._group_actions
        ):
            continue
        # 添加动作组标题。
        result.append(f"{t.gettext(group.title.title())}:\n")  # 动作组标题汉化
        # 遍历动作组中的每个动作，并根据类型进行分类。
        for action in group._group_actions:
            # 如果动作是子解析器动作，则遍历子解析器并递归调用render_parser函数。
            if isinstance(action, argparse._SubParsersAction):
                for name, subparser in action._name_parser_map.items():
                    result.append(render_parser(subparser, name, heading_level + 1))
                continue
            # 添加动作选项和帮助信息。
            opts = [f"`{opt}`" for opt in action.option_strings]
            # 如果动作没有选项字符串，则添加一个空选项。
            if not opts:
                # 添加一个空选项。
                line = f"- `{action.dest}`"
            else:
                # 添加动作选项和帮助信息。
                line = f"- {', '.join(opts)}"
            # 如果动作有元变量，则添加元变量。
            if action.metavar:
                line += f" `{action.metavar}`"
            # 添加动作帮助信息。
            line += f": {clean_help(action.help)}"
            # 如果动作有默认值，则添加默认值。
            if action.default and action.default != argparse.SUPPRESS:
                # 添加默认值。
                default = action.default
                # 如果动作有可选项且默认值为True，则将默认值设置为False。
                if any(opt.startswith("--no-") for opt in action.option_strings) and default is True:
                    # 将默认值设置为False。
                    default = not default
                # 添加默认值。
                line += f" (default: `{default}`)"
            # 添加动作行。
            result.append(t.gettext(line))  # 整行汉化
        result.append("")

    return "\n".join(result)


print(render_parser(parser, "pdm"))
```
