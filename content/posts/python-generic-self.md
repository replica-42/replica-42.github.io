---
title: "Python 的泛型 Self"
date: 2023-12-04T19:17:01+08:00
description: "Self-bounded Generic 的 Python 平替"
categories:
  - "Python"
---

## 环境

本文代码测试于以下环境：

```console
$ python -VV
Python 3.11.4 (tags/v3.11.4:d2340ef, Jun  7 2023, 05:45:37) [MSC v.1934 64 bit (AMD64)]
$ mypy -V
mypy 1.7.1 (compiled: yes)
```

## 一开始的需求总是很简单……

问题的由来是实际项目中遇到的一个场景：我有许多奇形怪状的数据类，这些类有各自的将自身序列化为 Python `dict` 的方法，也可以在 [dacite](https://github.com/konradhalas/dacite) 的支持下从结构合法的 `dict` 中反序列化为对应的实例（通常过程中还会参杂不少基于业务的对序列化后字典结构的调整）。以下是一个简短的示例：

```python
# 省略一些 import

@dataclass
class A:
    id: int
    name: str

    def to_dict(self) -> dict[str, object]:
        return {"id": str(self.id), "name": self.name}

    @classmethod
    def from_dict(cls, d: Mapping[str, object]) -> Self:
        return dacite.from_dict(cls, d, dacite.Config(cast=[int]))

if __name__ == "__main__":
    a = A(42, "replica42")
    print(a.to_dict())  # {'id': '42', 'name': 'replica42'}
    assert a == A.from_dict(a.to_dict())
```

注意到在序列化为 `dict` 时 `id` 字段的值被转换为了 `str` 类型，而反序列化时的类型转换是通过传入的 `dacite.Config` 实现的。现在需要为这些类增加对 Json 格式字符串的序列化与反序列化支持。因为 Python 标准库原生支持 `dict` 与 Json 字符串间的序列化与反序列化，为了避免重复编写模板代码，一个很直观的想法是写一个 Mixin 类并让这些数据类继承这个类。于是最初的示例就有了：

```python
# 省略一些 import

class JsonSerializableMixin:
    def to_json(self) -> str:
        return json.dumps(self.to_dict(), ensure_ascii=False)

    @classmethod
    def from_json(cls, s: str):
        return cls.from_dict(json.loads(s))

@dataclass
class A(JsonSerializableMixin):
  # 省略一些定义
  ...

if __name__ == "__main__":
    a = A(42, "replica42")
    print(a.to_dict())  # {'id': '42', 'name': 'replica42'}
    assert a == A.from_dict(a.to_dict())
    assert a == A.from_json(a.to_json())
```

运行一下上述代码，会发现第二条 assertion 不会报错，也就意味着功能实际上已经实现了。然而问题这才刚刚开始……

## 与 mypy 斗智斗勇

细心的读者大概已经发现了：Mixin 类的方法缺少类型标注。事实上在写完上面的代码的时候 vscode 的编辑器已经一片红了。运行一下 mypy 看看具体报了哪些错：

```console
$ mypy --strict src
src\demo\main.py:11: error: "JsonSerializableMixin" has no attribute "to_dict"  [attr-defined]
src\demo\main.py:14: error: Function is missing a return type annotation  [no-untyped-def]
src\demo\main.py:14: error: Type of decorated function contains type "Any" ("Callable[[type[JsonSerializableMixin], str], Any]")  [misc]
src\demo\main.py:15: error: "type[JsonSerializableMixin]" has no attribute "from_dict"  [attr-defined]
Found 4 errors in 1 file (checked 2 source files)
```

比较重要的是其中三行，分别提示 Mixin 类缺少 `to_dict` 属性和 `from_dict` 属性——这是当然的，因为在 `JsonSerializableMixin` 中并没有定义这些方法。除此之外还提示 `from_json` 缺少返回类型标注。既然有了错误原因，那么对症下药即可。对于 Mixin 类的类型标注，mypy 给出的[解决方案](https://mypy.readthedocs.io/en/stable/more_types.html#mixin-classes) 是使用 `Protocol` 对 `self` 进行标注。因此只需要定义一个对应的 `JsonSerializable` 协议并加上方法签名即可。

```python
# 省略一些 import

class JsonSerializable(Protocol):
    def to_dict(self) -> dict[str, object]:
        ...

    @classmethod
    def from_dict(cls, d: Mapping[str, object]) -> Self:
        ...


class JsonSerializableMixin:
    def to_json(self: JsonSerializable) -> str:
        return json.dumps(self.to_dict(), ensure_ascii=False)

    @classmethod
    def from_json(cls: type[JsonSerializable], s: str) -> JsonSerializable:
        return cls.from_dict(json.loads(s))
```

得益于 Python 的 structural typing，不需要显式标注 `A` 实现了 `JsonSerializable`。此时运行 mypy 发现不报错了，然而问题仍未解决……

## Generic Self

在上述测试代码之后添加如下语句：

```python
# 省略一些上文

if __name__ == "__main__":
    a = A(42, "replica42")
    print(a.to_dict())  # {'id': '42', 'name': 'replica42'}
    assert a == A.from_dict(a.to_dict())
    assert a == A.from_json(a.to_json())

    b = A.from_json(a.to_json())
    assert b.id == 42
```

运行同样可以通过所有断言，但是 mypy 输出：

```console
$ mypy --strict src
src\demo\main.py:48: error: "JsonSerializable" has no attribute "id"  [attr-defined]
Found 1 error in 1 file (checked 2 source files)
```

提示很直观：`b` 的类型被 mypy 分析为 `JsonSerializable`，而 `JsonSerializable` 并没有属性 `id`。其原因在于 Mixin 类中对 `from_json` 方法的返回类型标注为 `JsonSerializable`，而我们需要的不是这个协议，而是实现了协议的具体类。此时就需要引入类型变量，并使用类型变量标注对应的方法。

```python
# 省略一些上文

T = TypeVar("T", bound=JsonSerializable)


class JsonSerializableMixin:
    def to_json(self: JsonSerializable) -> str:
        return json.dumps(self.to_dict(), ensure_ascii=False)

    @classmethod
    def from_json(cls: type[T], s: str) -> T:
        return cls.from_dict(json.loads(s))
```

此时 `from_json` 成为了一个泛型方法，且类型变量标注于 `cls` 参数与返回值上，这样确保了在继承了该 Mixin 的类上调用 `from_json`，返回值的类型被约束为该类的实例。对于实例方法同样可以使用类型变量来标注 `self` 参数，这就是标题提到的 [Generic Self](https://mypy.readthedocs.io/en/stable/generics.html#generic-methods-and-generic-self)。如果是熟悉 Java 的朋友，大概会意识到这与 Java 中的 Self-bounded Generic 解决的是同样的问题（e.g. `class Enum<E extends Enum<E>>`）

完整的示例代码如下：

```python
import json
from collections.abc import Mapping
from dataclasses import dataclass
from typing import Protocol, Self, TypeVar

import dacite


class JsonSerializable(Protocol):
    def to_dict(self) -> dict[str, object]:
        ...

    @classmethod
    def from_dict(cls, d: Mapping[str, object]) -> Self:
        ...


T = TypeVar("T", bound=JsonSerializable)


class JsonSerializableMixin:
    def to_json(self: JsonSerializable) -> str:
        return json.dumps(self.to_dict(), ensure_ascii=False)

    @classmethod
    def from_json(cls: type[T], s: str) -> T:
        return cls.from_dict(json.loads(s))


@dataclass
class A(JsonSerializableMixin):
    id: int
    name: str

    def to_dict(self) -> dict[str, object]:
        return {"id": str(self.id), "name": self.name}

    @classmethod
    def from_dict(cls, d: Mapping[str, object]) -> Self:
        return dacite.from_dict(cls, d, dacite.Config(cast=[int]))


if __name__ == "__main__":
    a = A(42, "replica42")
    print(a.to_dict())  # {'id': '42', 'name': 'replica42'}
    assert a == A.from_dict(a.to_dict())
    assert a == A.from_json(a.to_json())

    b = A.from_json(a.to_json())
    assert b.id == 42
```

## 一些题外话

为什么选择用 Mixin + Protocol 的方式实现而不是抽象类？答案当然是 Python 支持多继承所以可以这么玩，单继承的语言就老老实实写抽象类用模板方法解决这样的问题好了（比如大名鼎鼎的 `AbstractQueuedSynchronizer`）。另一方面来说 Mixin 的语义更接近组合而非继承。在实际的项目场景里完全可以出现有一些类只需要序列化为字典而不需要序列化为 Json，此时序列化为字典的方法和序列化为 Json 的方法是完全解耦的，按需继承 Mixin 即可。换言之在强调实现功能多于定义接口的场景下，Mixin 是优于抽象类的。

除此之外考虑需要同时支持基于序列化后的字典进一步序列化为 Json 和 XML 的需求，此时如果采用抽象类方案，则将序列化为字典的抽象方法放在两个类中的任何一个都不合理，而如果单独提取出一个抽象类，则实质上回到了 Protocol + Mixin 的解决方案，且由于 structural typing 的特性，Protocol 并不需要抽象类那样显式注册或继承。