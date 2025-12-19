---
title: "SeaBed 文本提取二三事"
date: 2025-12-19T20:48:06+08:00
description: "易读性增加了.jpg"
categories:
  - "Python"
  - "SeaBed"
---

## 前言

请支持正版，请支持正版，请支持正版。本文不是解包教程，而且 SeaBed 的资产也没有加密，所以这里就指一下路大家自己去想办法吧。文本相关的都在游戏路径下的 `SeaBed_Data\StreamingAssets\scenario` 里。

## 使用

```python
#!/usr/bin/env python3
import argparse
from pathlib import Path
import re

name_blocks = {
    "T": {
        "EN": "POV: Takako",
        "CHS": "POV: 贵呼",
        "CHT": "POV: 貴呼",
        "JP": "POV: 貴呼",
        "KR": "POV: 타카코",
    },
    "S": {
        "EN": "POV: Sachiko",
        "CHS": "POV: 佐知子",
        "CHT": "POV: 佐知子",
        "JP": "POV: 佐知子",
        "KR": "POV: 사치코",
    },
    "N": {
        "EN": "POV: Narasaki",
        "CHS": "POV: 楢崎",
        "CHT": "POV: 楢崎",
        "JP": "POV: 楢崎",
        "KR": "POV: 나라사키",
    },
}


def func(path: str, merge: bool, lang_list: list[str]) -> None:
    """
    业务逻辑入口函数。

    :param path: 输入路径
    :param merge: 是否合并
    :param lang_list: 语言列表
    """
    content = Path(path).read_text(encoding="utf-16-le")

    titles = extract_title(content)
    blocks = extract_script(content)
    blocks.insert(0, titles)  # 将标题作为第一个块

    input_path = Path(path)
    stem = input_path.stem
    parent = input_path.parent

    # 1. 如果 merge 为 True，输出合并文件
    if merge:
        merge_lines = []
        for block in blocks:
            block_output = []
            for lang in lang_list:
                if lang in block:
                    block_output.append(f"{lang}: {block[lang]}")
            if block_output:
                merge_lines.extend(block_output)
                merge_lines.append("")  # 块间空一行

        # 移除最后一个多余的空行（可选）
        if merge_lines and merge_lines[-1] == "":
            merge_lines.pop()

        merge_path = parent / f"{stem}_merge.txt"
        with open(merge_path, "w", encoding="utf-8") as f:
            f.write("\n".join(merge_lines))

    # 2. 为 lang_list 中每种语言输出单独文件
    for lang in lang_list:
        lang_lines = []
        for block in blocks:
            if lang in block:
                lang_lines.append(f"{block[lang]}")
                lang_lines.append("")  # 每条后空一行
        # 移除末尾多余空行（保持格式简洁）
        if lang_lines and lang_lines[-1] == "":
            lang_lines.pop()

        lang_path = parent / f"{stem}_{lang}.txt"
        with open(lang_path, "w", encoding="utf-8") as f:
            f.write("\n".join(lang_lines))


def extract_script(content: str) -> list[dict[str, str]]:
    lang_map = {"0": "EN", "2": "CHS", "3": "CHT", "4": "KR"}

    result_blocks: list[dict[str, str]] = []
    current_block: dict[str, str] = {}
    in_if_block = False
    current_lang = None

    for line in content.splitlines():
        line = line.rstrip()

        # 匹配视点
        if line == "[msglayer2]":
            result_blocks.append(name_blocks["S"])
            continue
        if line == "[msglayer]" or line == "[msglayer_t]":
            result_blocks.append(name_blocks["T"])
            continue
        if line == "[msglayer3]":
            result_blocks.append(name_blocks["N"])
            continue

        # 特殊行跳过
        if line in {"[r]"}:
            continue

        # 检查是否为控制标签
        is_control = False
        if_match = re.match(
            r'^\[if\s+exp\s*=\s*"kag\.FBFPrepareLanguage\(\)\s*==\s*(\d+)"\s*\]$', line
        )
        elsif_match = re.match(
            r'^\[elsif\s+exp\s*=\s*"kag\.FBFPrepareLanguage\(\)\s*==\s*(\d+)"\s*\]$',
            line,
        )
        else_match = re.match(r"^\[else\]\s*$", line)
        endif_match = re.match(r"^\[endif\]\s*$", line)

        if if_match:
            in_if_block = True
            current_block = {}
            current_lang = None
            lang_code = if_match.group(1)
            current_lang = lang_map.get(lang_code)
            is_control = True

        elif elsif_match and in_if_block:
            lang_code = elsif_match.group(1)
            current_lang = lang_map.get(lang_code)
            is_control = True

        elif else_match and in_if_block:
            current_lang = "JP"
            is_control = True

        elif endif_match and in_if_block:
            # 结束当前块
            result_blocks.append(current_block)
            in_if_block = False
            current_lang = None
            current_block = {}
            is_control = True

        # 如果是控制标签，跳过内容处理
        if is_control:
            continue

        # 处理文本内容：仅当处于 if 块且当前语言有效时
        if in_if_block and current_lang is not None:
            # 移除 [prcm] 及之后内容
            text_part = line
            if "[prcm]" in text_part:
                text_part = text_part.split("[prcm]", 1)[0]
            if "[lr]" in text_part:
                text_part = text_part.split("[lr]", 1)[0]
            text_part = text_part.strip()
            text_part = process_line(text_part)
            if text_part:
                # 拼接：若已有内容，加空格；否则直接赋值
                if current_lang in current_block:
                    current_block[current_lang] += text_part
                else:
                    current_block[current_lang] = text_part
            # 注意：不重置 current_lang，允许多行连续拼接

    return result_blocks


def process_line(text: str) -> str:
    # 用直角引号代替[「]和[」]
    text = text.replace("[「]", "「").replace("[」]", "」").replace("[l]", "")
    # 先打印一下方括号包裹的标签和原始文本（如果有的话）
    if re.search(r"\[.*?\]", text):
        print(f"Processing line with tags: {text}")
    # 移除所有被方括号包裹的标签（控制内容）
    text = re.sub(r"\[.*?\]", "", text)
    return text.strip()


def extract_title(content: str) -> dict[str, str]:
    for line in content.splitlines():
        prefix = "[fbfstoretitle"
        if not line.strip().startswith(prefix):
            continue

        # 构造正则：匹配 lang="value" 的模式，支持转义引号（简单处理，假设值中不含未转义的 "）
        pattern = r'(\w+)="([^"]*)"'

        # 在前缀之后的部分中查找所有键值对
        content_part = line[len(prefix) :].rstrip("]").strip()
        matches = re.findall(pattern, content_part)

        # 构建结果字典
        result = {}
        for lang_key, value in matches:
            # 标准化语言键（如 eng -> EN）
            lang_map = {"eng": "EN", "chs": "CHS", "cht": "CHT", "jp": "JP", "kr": "KR"}
            if lang_key in lang_map:
                result[lang_map[lang_key]] = value
        return result
    return {}


def main() -> None:
    parser = argparse.ArgumentParser(description="命令行程序")

    # 必选参数：path
    parser.add_argument("path", type=str, help="输入路径")

    # 可选参数：merge，默认为 True
    parser.add_argument(
        "--merge",
        action=argparse.BooleanOptionalAction,
        default=True,
        help="是否合并（默认: True）",
    )

    # 可选参数：lang_list，默认为 ["EN", "JP", "CHS"]
    parser.add_argument(
        "--lang-list",
        nargs="+",
        choices=["EN", "CHS", "CHT", "JP", "KR"],
        default=["EN", "JP", "CHS"],
        help="语言列表（可选项: EN, CHS, CHT, JP, KR；默认: EN JP CHS）",
    )

    args = parser.parse_args()

    # 调用业务逻辑函数
    func(args.path, args.merge, args.lang_list)


if __name__ == "__main__":
    main()
```

使用方式

```shell
$ python main.py 001.txt
```

默认输出四个文件

* 001_merge.txt 包含英日中三语文本
* 001_EN.txt/001_JP.txt/001_CHS.txt 包含每种语言单独的文本

如果不需要合并后的文件

```shell
$ python main.py 001.txt --no-merge
```

如果需要指定语言

```shell
$ python main.py 001.txt --lang-list EN JP CHS CHT KR
```

原始文本示例

![before.png](https://s2.loli.net/2025/12/19/Q9LKejm8AlHMdGW.png)


解析后示例

![after.png](https://s2.loli.net/2025/12/19/tn38zCpByQIZVoj.png)

除正文外支持功能：提取标题，提取 pov。以及没有对所有场景测试过，不保证每一个都能 work
