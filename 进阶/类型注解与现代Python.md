# 类型注解与现代Python

```python
def f(x):
    return x.decode("utf-8")
```

上线三个月。调用方一直传 `str`，某次下游改成了 `bytes`，`f` 的作者跟着改了实现，没人改调用方。`str` 没有 `decode`，爆炸发生在运行时、在离改动很远的请求路径上。注解拦不住这次运行——CPython 默认不看注解——但能让 pyright / mypy 在 CR 里把「`str` 没有 `decode`」和「调用方还在传 `str`」一起标红。

C++ 的类型在编译期就是契约，改签名链接器或编译器当场拒绝。Go 的 `func f(x string)` 同样。Python 的注解是**给检查器看的元数据**，不是给解释器看的指令。下面按「元数据怎么挂、检查器认什么、后端怎么落地」写。版本按 3.11–3.13；3.9 / 3.10 的语法分界会标出来。

---

## 一、注解是元数据，运行时默认不检查

### 1、写了不等于会验

```python
def greet(name: str) -> str:
    return 42                    # 运行时正常返回 int


print(greet(b"alice"))           # 运行时正常，name 是 bytes
print(greet.__annotations__)     # {'name': <class 'str'>, 'return': <class 'str'>}
```

`__annotations__` 是普通 dict，函数对象上的一块数据。调用 `greet` 时解释器不读它。C++ 没有「写了类型但不检查」这条路；Go 也没有。Python 把检查推迟给外部工具，换来的是动态运行时还能把函数当普通对象改。

要运行时验，得自己上：

```python
from pydantic import TypeAdapter, ValidationError

adapter = TypeAdapter(str)

def greet_checked(name: str) -> str:
    name = adapter.validate_python(name)
    return f"hi {name}"


# greet_checked(b"alice")        # ValidationError
```

pydantic 做边界（JSON / 表单 / 配置）。beartype / typeguard 做函数入口。内部热路径不要每层都 runtime check，那是把检查器的活搬进请求延迟。

### 2、`from __future__ import annotations`

```python
from __future__ import annotations

class Node:
    def add_child(self, n: Node) -> Node:   # 不写 future 时，这里 Node 还没绑上
        return n


print(Node.add_child.__annotations__)       # {'n': 'Node', 'return': 'Node'}  字符串
```

PEP 563：注解存成字符串，求值推迟。好处是前向引用、循环导入时注解不再立刻找名字。坏处是运行时真去读注解的库（早期 pydantic、运行时 DI）拿到的是 str，要 `typing.get_type_hints` 再求值。3.11+ 多数库已经按「可能是 str」处理。3.14 起 PEP 649 换成惰性求值，不靠 future；现在写 3.11–3.13 代码，模块顶上加这行仍然是稳妥默认。

### 3、检查器看到的世界和解释器看到的世界

| | 解释器 | pyright / mypy |
| --- | --- | --- |
| `x: int = "a"` | 赋值成功 | 报错 |
| `list.append` 异构 | 合法 | 按注解拒绝 |
| `Protocol` | 普通 class | 结构子类型 |
| `cast(int, s)` | 原样返回 | 之后当 int |
| `Any` | 无意义 | 关闭这棵子树的检查 |

两边同时正确：运行时是动态的，静态检查是可选的、局部的。后端把检查器接到 CR，等于把「签名是契约」从口头约定变成机器拒绝。

---

## 二、内置泛型：`list[int]`，不要再写 `typing.List`

### 1、PEP 585（3.9+）

```python
def ids_ok(xs: list[int]) -> dict[str, int]:
    return {str(x): x for x in xs}


# 3.8 及更早、或死守旧风格：
from typing import Dict, List

def ids_old(xs: List[int]) -> Dict[str, int]:
    return {str(x): x for x in xs}
```

`typing.List` 是 `list` 的别名，3.9 起内置类型本身可下标。新代码用 `list[int]`、`dict[str, int]`、`set[str]`、`tuple[int, ...]`、`tuple[str, int]`。`List` / `Dict` / `Set` / `Tuple` 在 3.9+ 是兼容层，3.13 仍能跑，检查器会提示 deprecated。

`tuple[int, str]` 是长度 2、位置类型不同；`tuple[int, ...]` 是任意长、元素同质。这和 C++ `std::tuple<int, string>` / `std::vector<int>` 的区分同一件事。Go 没有异构 tuple 当一等类型，用 struct。

### 2、抽象容器用 `collections.abc`

```python
from collections.abc import Iterable, Mapping, Sequence

def sum_nums(xs: Iterable[int]) -> int:
    return sum(xs)

def pick(m: Mapping[str, int], k: str) -> int:
    return m[k]
```

参数写 `Iterable[int]`，list / tuple / 生成器 / dict.keys() 都能过。返回值写具体：`list[int]`，不要写 `Iterable`——调用方不知道能不能下标、能不能耗两次。

`Mapping` 只读；要写入用 `MutableMapping`。`Sequence` 有下标和长度；只要能 `for` 就用 `Iterable`。C++ 里这是 iterator 类别和 container 概念；Go 里 1.18+ 是 `iter.Seq` / 自己的 interface。

### 3、`dict` 的 value 异构时不要硬套

```python
# 差：检查器认为每个 value 都是 str | int | list[int]，下游全是收窄
Config = dict[str, str | int | list[int]]

# 好：形状固定就 TypedDict；运行时要校验就 pydantic
```

`dict[str, Any]` 是「我放弃这棵树」。能画出字段就不要用它。

---

## 三、`Optional` vs `T | None`（3.10+ PEP 604）

### 1、同一件事，写法不同

```python
from typing import Optional

def find_old(uid: str) -> Optional[str]:
    return None

def find_new(uid: str) -> str | None:
    return None
```

`Optional[T]` ≡ `Union[T, None]` ≡ `T | None`。3.10+ 用 `|`。`Union[A, B, C]` 写成 `A | B | C`。

`Optional` **不是**「这个参数可以不传」。可选参数是默认值：

```python
def connect(host: str, port: int | None = None) -> None:
    port = port or 5432
    ...
```

`port: int | None = None` 表示：可以不传（有默认），传了也可以是 `None`。`port: int = 5432` 才是「可省略、省略就是 5432」。面试里把 Optional 说成「可选参数」是错的。

### 2、检查器要你收窄

```python
def load(uid: str) -> str:
    name = find_new(uid)
    return name.strip()          # pyright: name 可能是 None
```

收窄：

```python
def load(uid: str) -> str:
    name = find_new(uid)
    if name is None:
        raise KeyError(uid)
    return name.strip()          # 这之后 name: str
```

`assert name is not None` 也能收窄，失败是 AssertionError，生产别靠它当控制流。C++ `std::optional` 解引用前要 `has_value`；Go 的 `(T, error)` 或 `(T, bool)` 同一类问题。Python 把 `None` 塞进同一条返回值，检查器强制你写分支。

### 3、容器里的 `| None`

```python
def first(xs: list[str | None]) -> str | None:
    return xs[0] if xs else None
```

`list[str] | None` 是「整份 list 可能不存在」；`list[str | None]` 是「list 在，元素可能空」。两种经常被写成同一个 `Optional[list]`。边界 API 上差一个就变成 NPE。

---

## 四、`TypeAlias`、`type` 语句、`TypeVar`、`ParamSpec`、`TypeVarTuple`

### 1、别名

```python
from typing import TypeAlias

UserId: TypeAlias = str                    # 3.10+ PEP 613，检查器当别名，不是新类型
type UserId12 = str                        # 3.12+ PEP 695，lazy，可泛型
```

`TypeAlias` 告诉检查器「这是别名不是赋值」。`type` 语句是 3.12 的语法，别名本身可带参数：

```python
type Pair[T] = tuple[T, T]

def swap[T](p: Pair[T]) -> Pair[T]:
    a, b = p
    return b, a
```

要「别名但不相等」，用 `NewType`：

```python
from typing import NewType

UserId = NewType("UserId", str)

def load(uid: UserId) -> str: ...

load(UserId("u1"))
# load("u1")                    # 检查器拒绝；运行时 UserId 就是 str
```

运行时 `UserId("u1")` 返回普通 `str`，零开销。C++ 的 `using UserId = std::string;` 拦不住误传；强类型要 `struct UserId { std::string v; }`。Go 的 `type UserId string` 是新定义类型，`UserId("u1")` 和 `string` 不能混。Python 的 `NewType` 只在检查器里像 Go 的 defined type，运行时像 C++ 的 typedef。

### 2、`TypeVar`

```python
from typing import TypeVar

T = TypeVar("T")
S = TypeVar("S", bound=str)                # 必须是 str 的子类
Num = TypeVar("Num", int, float)           # 约束：只能是其中之一

def first[T](xs: list[T]) -> T | None:     # 3.12 内联，不必先声明 TypeVar
    return xs[0] if xs else None
```

`bound=`：T 至少是那个基类，方法可以当那个基类用。约束形式 `TypeVar("N", int, float)` 是「每次代入只能是其中一个」，`int | float` 联合不是一回事。3.12 的 `[T]` 写在 `def` / `class` 上，旧的模块级 `T = TypeVar("T")` 仍然合法，混用时同一语义不要两种写法。

### 3、`ParamSpec`：装饰器保住签名

```python
from collections.abc import Callable
from typing import ParamSpec, TypeVar

P = ParamSpec("P")
R = TypeVar("R")


def traced(fn: Callable[P, R]) -> Callable[P, R]:
    def inner(*args: P.args, **kwargs: P.kwargs) -> R:
        return fn(*args, **kwargs)
    return inner
```

`Callable[..., R]` 丢掉参数。`ParamSpec` 把原函数的参数列表当成一个变量传过去。3.10+。`Concatenate[Auth, P]` 表示「前面多一个 Auth」。装饰器篇写过 `wraps` 保运行时身份；`ParamSpec` 保检查器里的签名。两件都要做。

C++ 里这是 forwarding reference + `decltype`；Go 的泛型函数目前表达不了「任意参数列表」，装饰器式中间件只能 `func(http.Handler) http.Handler`。

### 4、`TypeVarTuple`：可变长度类型参数

```python
from typing import TypeVarTuple

Ts = TypeVarTuple("Ts")

def concat[T, *Ts](head: T, *rest: *Ts) -> tuple[T, *Ts]:
    return (head, *rest)


print(concat(1, "a", True))                 # tuple[int, str, bool]
```

PEP 646，3.11+。用在 `*args` 的精确类型、数组 shape、`asyncio.gather` 一类「N 个不同类型」的 API。日常后端少写，读库的 stub 会碰到。

---

## 五、Protocol：结构子类型，对照 ABC 和 Go interface

### 1、名义 vs 结构

面向对象篇点过：ABC 要继承，Protocol 看有没有这些方法。这里把检查器和运行时拆开。

```python
from abc import ABC, abstractmethod
from typing import Protocol, runtime_checkable

class HandlerABC(ABC):
    @abstractmethod
    def handle(self, body: bytes) -> bytes: ...

class Echo(HandlerABC):
    def handle(self, body: bytes) -> bytes:
        return body


class HandlerP(Protocol):
    def handle(self, body: bytes) -> bytes: ...

class Duck:
    def handle(self, body: bytes) -> bytes:
        return body


def run_abc(h: HandlerABC) -> bytes:
    return h.handle(b"x")

def run_p(h: HandlerP) -> bytes:
    return h.handle(b"x")


run_abc(Echo())
# run_abc(Duck())               # 检查器拒绝，运行时也不是实例
run_p(Duck())                   # 检查器接受：有 handle
run_p(Echo())                   # 也接受
```

`Duck` 没继承 `HandlerP`，检查器照样认。运行时 `isinstance(Duck(), HandlerP)` 默认是 **False**——Protocol 不是 ABC，不注册虚拟子类。要运行时认：

```python
@runtime_checkable
class HandlerP(Protocol):
    def handle(self, body: bytes) -> bytes: ...


print(isinstance(Duck(), HandlerP))         # True：只查方法在不在，不查签名
```

`runtime_checkable` 的 `isinstance` **不检查参数类型**，只检查属性存在。不能当校验器用。

### 2、和 Go interface、C++ 的对应

| | 名义 | 结构 |
| --- | --- | --- |
| C++ | 抽象基类 / 继承 | Concepts（编译期）、模板鸭子 |
| Go | 无（不能声明「我实现了」当强制） | `interface`，有方法就算 |
| Python | ABC、显式继承 | Protocol（检查器）；运行时要 `@runtime_checkable` |

Go 的 interface 在编译期和运行时都是结构的：`var h Handler = Duck{}` 只要方法集包含。Python 的 Protocol 默认**只在检查器**里结构；运行时还是名义（普通 class）或浅层 `hasattr`。从 Go 迁过来最容易写 `isinstance(x, SomeProtocol)` 以为等价于 Go 的 type assert，签名不对也过。

C++ Concepts 是编译期结构约束，实例化失败在模板展开。Python Protocol 失败在 CI 的 pyright，不在 `python app.py`。

### 3、泛型 Protocol 和 `Self`

```python
from typing import Protocol, Self

class Copyable(Protocol):
    def copy(self) -> Self: ...

class Point:
    def __init__(self, x: int) -> None:
        self.x = x
    def copy(self) -> Self:
        return type(self)(self.x)
```

`Self`（3.11+，`typing.Self`）表示「这个类自己」，子类方法返回子类。写成 `-> Point` 则子类 `ColorPoint.copy()` 的返回值会被当成 `Point`。C++ 里 CRTP 干这件事；Go 里通常返回具体类型或 interface，没有 Self。

插件框架用 ABC：强制继承、文档化、`Handler()` 直接 TypeError。库边界「只要有 `read(n) -> bytes`」用 Protocol，别要求用户继承你的基类。

---

## 六、TypedDict / NamedTuple / dataclass / pydantic 怎么选

### 1、四套「带字段的东西」

```python
from typing import NamedTuple, TypedDict
from dataclasses import dataclass

class UserTD(TypedDict):
    id: str
    age: int

class UserNT(NamedTuple):
    id: str
    age: int

@dataclass
class UserDC:
    id: str
    age: int
```

| | TypedDict | NamedTuple | dataclass | pydantic.BaseModel |
| --- | --- | --- | --- | --- |
| 运行时对象 | 就是 `dict` | tuple 子类 | 普通实例 | 普通实例 |
| 校验 | 无 | 无 | 无（可 `__post_init__`） | 核心 |
| JSON 进出 | 本身就是 dict | 要 `_asdict` | `asdict`，嵌套弱 | `model_validate` / `model_dump` |
| 可变 | 是 | 否 | 默认可变 | 默认可变 |
| 当 dict key | 否（dict 不可哈希） | 是 | `frozen=True` 才 | `frozen=True` 才 |
| 依赖 | 无 | 无 | 无 | 第三方 |
| 适用 | JSON 形状、kwargs 展开 | 轻量不可变记录 | 内部领域、有一点行为 | 进程边界 |

面向对象篇给过 dataclass 的 `frozen` / `slots`。这里补边界和 TypedDict。

### 2、TypedDict：给 dict 画形状

```python
from typing import NotRequired, Required, TypedDict

class User(TypedDict):
    id: str
    age: int
    nickname: NotRequired[str]

class Patch(TypedDict, total=False):
    age: int
    nickname: str

def save(u: User) -> None:
    print(u["id"], u.get("nickname"))
```

`total=True`（默认）：所有键必填。`total=False`：所有键可选。单个键用 `Required` / `NotRequired`（3.11+ 也可 `typing_extensions`）。检查器认 `u["id"]`；运行时还是 dict，多键少键、值类型错，解释器不管。

`TypedDict` 不能继承普通 class，不能有方法。它不是类层次，是 dict 的结构注解。和 Go 的 `struct` + `json` tag 比，它更接近「JSON object 的 schema」，不是领域模型。

### 3、选法（后端）

- **配置文件、HTTP JSON、消息队列 payload**：pydantic。非法输入要在边界变成 400，不要漏进领域。
- **领域对象**（有不变量、有方法、活过一次请求）：dataclass 或手写 class。进门时从 pydantic model 转过来，出去再转回去。
- **函数之间传的、形状稳定的 dict**（尤其 `**kwargs` 展开、`json.loads` 之后第一站）：TypedDict。
- **当 dict key、当 set 元素的轻量记录**：NamedTuple 或 `frozen` dataclass。
- **不要**用 `dict[str, Any]` 传四层；**不要**让 pydantic model 当领域实体满天飞（ORM 身份、脏检查、和校验耦合）。

```python
from dataclasses import dataclass

# 伪 pydantic：真实项目 from pydantic import BaseModel, Field
class UserIn:
    def __init__(self, id: str, age: int) -> None:
        if age < 0:
            raise ValueError("age")
        self.id, self.age = id, age

    @classmethod
    def from_json(cls, d: dict[str, object]) -> "UserIn":
        return cls(id=str(d["id"]), age=int(d["age"]))


@dataclass
class User:
    id: str
    age: int

    def bump_age(self) -> None:
        self.age += 1


def handle(body: dict[str, object]) -> dict[str, object]:
    incoming = UserIn.from_json(body)       # 边界校验
    user = User(incoming.id, incoming.age)  # 领域
    user.bump_age()
    return {"id": user.id, "age": user.age} # 出站再变成 dict / model_dump
```

边界用 pydantic 时，把 `UserIn` 换成 `BaseModel`，`from_json` 换成 `model_validate`。领域层不要 `model_validate`——已经信任内部不变量。

---

## 七、`Literal` / `Final` / `ClassVar` / `Self` / `Never` / `NoReturn`

### 1、`Literal`：值就是类型

```python
from typing import Literal

Mode = Literal["ro", "rw"]

def open_db(mode: Mode) -> None:
    if mode == "ro":
        ...
    elif mode == "rw":
        ...
    else:
        raise ValueError(mode)              # 检查器：此分支 Never
```

状态机、HTTP method、任务状态，用 `Literal` 或 `enum.Enum`。字符串字面量散落各处时，检查器帮你抓错字 `"rn"`。`Literal[1, 2, 3]` 对数字同样。

### 2、`Final` 和 `ClassVar`

```python
from typing import ClassVar, Final

MAX_RETRY: Final = 3                        # 不能再赋值（检查器）
MAX_RETRY = 4                               # pyright 报错；运行时成功

class Worker:
    kind: ClassVar[str] = "http"            # 类属性，不是 __init__ 字段
    n: int

    def __init__(self, n: int) -> None:
        self.n = n
```

dataclass / pydantic 里，没标 `ClassVar` 的注解会被当成实例字段。这是「类上的计数器被生成进 `__init__`」的根因。`Final` 对实例属性表示「构造之后不改」，和 `frozen` dataclass 重叠时选一个策略，别混。

运行时 `Final` 什么都不做。C++ `const` / `constexpr` 是语言强制；Go 的 `const` 只对编译期常量。Python `Final` 是检查器契约。

### 3、`Self`

见第五节。工厂、`copy`、fluent setter 返回 `Self`。`from __future__ import annotations` 下也可以写 `"Self"` 或直接 `Self`（3.11+）。

### 4、`Never` 和 `NoReturn`

```python
from typing import Never, NoReturn

def die(msg: str) -> NoReturn:
    raise RuntimeError(msg)

def exhaust(mode: Literal["a", "b"]) -> int:
    if mode == "a":
        return 1
    if mode == "b":
        return 2
    unreachable: Never = mode               # 漏分支时这里报错
    raise AssertionError(unreachable)
```

`NoReturn`：函数不返回（`raise` 或 `os._exit` 或死循环）。`Never`：空类型，没有任何值。3.11 起 `typing.Never`；`NoReturn` 更老，表示「调用之后控制流结束」。穷尽匹配用 `Never` 当断言；退出进程用 `NoReturn`。

`cast` 到 `Never` 没有意义。空列表的元素类型有时推断成 `list[Never]`，这是「没有任何元素」而不是 bug。

---

## 八、`cast`、`assert_type`、`TYPE_CHECKING` 与循环导入

### 1、`cast` 是对检查器说谎

```python
from typing import cast

raw: object = fetch()                       # 运行时其实是 dict
user = cast(UserTD, raw)                    # 检查器之后当 UserTD；运行时原样返回
```

`cast(T, x)` 运行时是 `return x`，连 `isinstance` 都不做。边界上 `json.loads` 的结果是 `Any`，一 `cast` 进去，后面整条链都是假安全。正确做法：pydantic `model_validate`，或手写把 `object` 收窄成字段。

`cast` 的合法用途：检查器比你笨的时候（C 扩展返回类型、已知的窄化检查器看不出）。每次 `cast` 当代码审缺陷看。

### 2、`assert_type`：把推断钉死在测试里

```python
from typing import assert_type

def f(x: int | str) -> None:
    if isinstance(x, int):
        assert_type(x, int)                 # 检查器验证这里是 int；运行时 no-op
```

3.11+。CI 跑 pyright 时，推断变了会红。不要当运行时 assert。

### 3、`TYPE_CHECKING` 打断注解引起的循环导入

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from app.user.service import UserService   # 只给检查器看

class OrderService:
    def __init__(self, users: "UserService") -> None:
        self._users = users
```

`TYPE_CHECKING` 运行时恒为 False，检查器当 True。注解用字符串或 future，运行时不求值。真正的循环导入（两边模块级立刻调对方）这个解决不了，得把共享类型拆到 `types.py`，或延迟到函数里再 import。

C++ 前向声明 + 指针；Go 不能循环 import，必须拆包。Python 运行时循环 import 有时能靠「函数内 import」混过去，类型检查会把两边的名字都要。`TYPE_CHECKING` 是给检查器的前向声明。

```python
from __future__ import annotations
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from collections.abc import Callable

def apply(fn: Callable[[int], int], x: int) -> int:
    return fn(x)
```

运行时不 import `Callable` 也能过——注解是字符串。这是「减少运行时导入」的旁路，不是默认风格。默认还是顶上正常 import，循环了再拆。

---

## 九、`overload`：同一函数多种签名

```python
from typing import Literal, overload

@overload
def loads(raw: bytes, *, as_bytes: Literal[True]) -> bytes: ...
@overload
def loads(raw: bytes, *, as_bytes: Literal[False] = False) -> str: ...

def loads(raw: bytes, *, as_bytes: bool = False) -> bytes | str:
    return raw if as_bytes else raw.decode()
```

`@overload` 的函数体必须是 `...`，真正实现紧跟在后面、不带 decorator。检查器按 overload 列表匹配调用点；运行时只有最后那个实现。

```python
s = loads(b"ab")                            # str
b = loads(b"ab", as_bytes=True)             # bytes
```

实现的返回值写成 `bytes | str` 是给运行时和漏网调用看的。调用点不应落到联合上，否则 overload 白写。

常见场景：`__getitem__` 接受 `int` 返回 `T`、接受 `slice` 返回 `list[T]`；`open` 的文本/二进制模式；工厂按 flag 返回不同子类。

C++ 函数重载是真重载，生成不同符号。Go 没有重载。Python 运行时仍然是一个对象，overload 纯静态。漏写实现、overload 顺序不对（更宽的签名在前），检查器会匹配错。从最窄写到最宽。

---

## 十、3.11 / 3.12 / 3.13 和类型相关的点

类型语法是主线。这几个版本还有几件后端会碰到的语言变化，点到为止，不和并发篇抢戏。

### 1、3.11：`ExceptionGroup` / `except*`、`tomllib`、`TaskGroup`

```python
try:
    async with asyncio.TaskGroup() as tg:   # 3.11，结构化并发
        tg.create_task(job(1))
        tg.create_task(job(2))
except* TimeoutError as eg:                 # except* 拆 ExceptionGroup
    log(eg.exceptions)
```

`TaskGroup` 里一个任务炸，其它被取消，异常打成 `ExceptionGroup`。`except*` 按类型从 group 里拆。类型上 `ExceptionGroup[Exception]` 在 stub 里能写，业务代码更常直接 `except*`。

`tomllib` 是 3.11 起标准库只读 TOML，配置文件不必再为「读 pyproject」引第三方。写 TOML 仍要第三方。

### 2、3.12：f-string 与 per-interpreter GIL

f-string 里终于可以随便写引号、换行、注释，和普通括号表达式一样：

```python
name = "x"
print(f"value={
    name.upper()        # 合法
}")
```

PEP 684：每个 sub-interpreter 一把独立 GIL。这不是取消 GIL，是「进程内多解释器各一把锁」。默认 `python app.py` 仍是一把。细节在并发篇。

`type` 语句、泛型函数 `def f[T](...)` 也是 3.12。新代码优先用，不必再在模块级堆 `TypeVar`。

### 3、3.13：free-threaded 一句，`@deprecated`

3.13 提供 **可选** 的 free-threaded 构建（PEP 703）。官方默认构建仍有 GIL。怎么选线程、锁粒度、扩展适配，见并发篇，这里不展开。

```python
from warnings import deprecated                 # 3.13 标准库

@deprecated("use fetch_v2")
def fetch_v1(uid: str) -> str:
    return uid
```

检查器和 IDE 会把调用点标过时。类型上它是装饰器，签名不变。3.12 及更早用 `typing_extensions.deprecated`。

---

## 十一、`pyproject.toml` 里最小的 mypy / pyright

```toml
[project]
name = "svc"
version = "0.1.0"
requires-python = ">=3.11"

[tool.pyright]
pythonVersion = "3.12"
typeCheckingMode = "basic"          # 先 basic，核心包再 strict
include = ["src"]
exclude = ["**/tests/fixtures"]
venvPath = "."
venv = ".venv"

[tool.mypy]
python_version = "3.12"
packages = ["src"]
warn_unused_ignores = true
check_untyped_defs = true
disallow_untyped_defs = false       # 全开会变成 any 战争，按包加
no_implicit_optional = true
```

只配一个检查器也能活。pyright 快、IDE 同源（Pylance）；mypy 插件生态（django-stubs、pydantic.mypy）。两边规则不完全相同，**选一个当 CR gate**，另一个当编辑器即可。两个都当 gate，同一处 `type: ignore` 要写两遍。

局部关闭：

```python
x = lib.untyped()                  # type: ignore[no-untyped-call]
```

`# type: ignore` 不写码是炸弹。码写上，CR 能搜。`Any` 显式写出来，比让检查器推断成 `Unknown` 再传十层更诚实。

`pyrightconfig.json` 和 `[tool.pyright]` 二选一，别两份打架。

---

## 十二、后端服务里的落地顺序

### 1、先边界，再领域，别全仓库开战

全仓库突然 `disallow_untyped_defs = true`，结果是一万个 `Any` 和 `# type: ignore`。检查器变绿，契约还是没有。顺序：

1. **进站 / 出站 schema**。HTTP body、消息、配置：pydantic model 或 TypedDict + 校验。这一层的 bug 是「下游改了字段我们还当 str」。开篇那个 `f(x)` 如果 `x` 来自 JSON，schema 改字段类型时 CR 直接红。
2. **核心领域函数**。订单、会话、计费：参数和返回值写具体类型，`Protocol` 描述端口（存储、时钟、下游 HTTP）。这里的注解是给自己人看的设计文档。
3. **适配器**。ORM row、redis 客户端、第三方 SDK。SDK 没 stub 就写一层薄包装，返回你的领域类型，把 `Any` 关在包装内。
4. **脚本、一次性迁移、测试夹具**。最后才管。测试里用 `assert_type` 钉住关键推断。

### 2、端口用 Protocol，实现用具体类

```python
from typing import Protocol

class UserRepo(Protocol):
    def get(self, uid: str) -> User | None: ...
    def save(self, user: User) -> None: ...

class SqliteUserRepo:
    def get(self, uid: str) -> User | None: ...
    def save(self, user: User) -> None: ...

def ban(repo: UserRepo, uid: str) -> None:
    u = repo.get(uid)
    if u is None:
        raise KeyError(uid)
    ...
```

`ban` 不依赖 sqlite。测试塞内存实现。Go 里这就是 `type UserRepo interface { ... }`；C++ 里是抽象基类或 template。Python 用 Protocol，实现类不必继承。

### 3、`Any` 的预算

每个包给一个「对外不出现 `Any`」的清单。内部和第三方交界允许，但要有包装函数把 `Any` 收成具体类型。`dict[str, Any]` 出现在公共签名上，等于告诉调用方「自己猜」。

开篇的 `def f(x):` 改成 `def f(x: bytes) -> str:` 之后，调用方 `f(user_name)` 在 CR 红。运行时那一次事故不会被注解拦住——已经上线的进程没有检查器——但下一份 CR 会。注解的价值是把类型变化从「三个月后的线上」提前到「这次 diff」。
