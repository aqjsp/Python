# Python入门

C++ 写 `int a = 1;`，编译器在当前栈帧开 4 字节，把 `1` 写进去。Go 的 `var a int = 1` 同理：名字对应一块固定大小的存储。Python 写 `a = 1`，做的事情完全不同——解释器在堆上找（或 intern）一个 `PyLongObject`，再把当前命名空间里的名字 `a` 绑到这个对象上。

```python
import sys

a = 1
print(type(a))          # <class 'int'>
print(id(a))            # 对象的身份，CPython 里就是堆地址
print(a.__class__)      # <class 'int'>
print(sys.getsizeof(a)) # 通常 28 字节，不是 4
```

`a` 不是「装 1 的盒子」，`a` 是标签。改绑定、共享对象、可变对象被原地修改——后面所有坑都从这个模型长出来。

```python
import sys

a = 1
print(sys.getsizeof(a))     # CPython 3.11+ 小整数约 28 字节
print(sys.getsizeof(10**20))  # 位数变多，对象变大
```

C++ 的 `int` 溢出是 UB（或绕回）；Go 的 `int` 溢出静默绕回；Python 的 `int` 永远不会溢出，它会换成更大的堆对象。代价是：每次算术都可能分配，没有「寄存器里加一下」那种便宜。

---

## 一、名字与对象：绑定，不是变量盒子

### 1、赋值语句在干什么

```python
x = 1000
y = x
x = 1001
print(y)  # 1000
```

C++ 程序员下意识会想「`y` 是 `x` 的拷贝」。对不可变对象，表面行为碰巧像拷贝：`y = x` 让两个名字指向同一个 `int`；随后 `x = 1001` 是给 `x` 换绑到新对象，`y` 还指着旧的。真正的拷贝从来没发生。

对可变对象，同一套规则立刻露出原形：

```python
a = [1, 2, 3]
b = a
a.append(4)
print(b)  # [1, 2, 3, 4]  —— b 不是副本
```

`b = a` 复制的是引用（名字到对象的指针），不是 list 本体。Go 的 slice 头拷贝、C++ 的指针拷贝，才是对应物；`std::vector` 的拷贝构造、Go 的值类型 struct 赋值，都不是。

用图说：

```
名字空间（dict）          堆
┌─────────┐              ┌──────────────────┐
│ a  ─────┼─────────────►│ list [1,2,3,4]   │
│ b  ─────┼─────────────►│                  │
└─────────┘              └──────────────────┘
```

函数参数传递也是「传对象引用」：形参是新名字，绑到同一个对象。不可变对象上看起来像传值，可变对象上看起来像传引用——本质都是传引用，差别只在对象能不能被原地改。

```python
def bump_int(n: int) -> None:
    n += 1          # 换绑局部名 n，外面的 x 不动

def bump_list(xs: list[int]) -> None:
    xs.append(1)    # 原地改，外面看得到

x = 10
bump_int(x)
print(x)            # 10

nums = [10]
bump_list(nums)
print(nums)         # [10, 1]
```

Go 里 `func f(n int)` 是拷贝，`func f(s []int)` 拷贝的是 slice 头（底层数组共享）。Python 没有这层区分：一律绑名字。想切断共享，必须显式拷贝（`xs[:]`、`list(xs)`、`copy.copy` / `copy.deepcopy`），数据结构篇会展开。

### 2、id()、is、==

- `id(obj)`：对象身份。CPython 实现为堆地址，规范只保证「在对象存活期内唯一」。
- `a is b`：`id(a) == id(b)`，是不是同一个对象。
- `a == b`：调 `a.__eq__(b)`，比的是值。

```python
a = [1, 2]
b = [1, 2]
print(a == b)   # True，值相等
print(a is b)   # False，两个 list 对象
print(id(a), id(b))
```

`None`、`True`、`False` 是单例，用 `is` 判断是约定也是正确做法：

```python
x = None
if x is None:       # 对
    pass
if x == None:       # 能跑，但可能被重载的 __eq__ 骗
    pass
```

自定义类如果乱写 `__eq__`，`== None` 可能返回奇怪的东西；`is None` 走身份，绕过 `__eq__`。

C++ 的 `==` 对内置类型比位模式，对 class 看你有没有重载；指针比的是地址。Go 的 `==` 对可比较类型比值，slice/map/func 不能直接 `==`。Python 把「身份」和「相等」拆成两个运算符，面试里最爱问的就是「什么时候用 `is`」。

规则就一条：身份用 `is`（`None`、单例、故意共享的哨兵）；业务相等用 `==`。

### 3、`is` 的假阳性：小整数和短字符串

```python
a = 256
b = 256
print(a is b)    # True，小整数 intern

a = 257
b = 257
print(a is b)    # 交互式里常是 False；同一编译单元的常量可能仍是 True
```

**不要用 `is` 比较数字或字符串。** intern 是解释器的实现细节，不是语言保证。下一节会把 intern 边界讲清楚。同一文件里的两个字面量 `257` 可能被编译器合并成一个常量，交互式一行一个表达式就不会——所以「我本地试过 `257 is 257` 是 True」不能当结论。

### 4、del 删的是名字，不是对象

```python
a = [1, 2, 3]
b = a
del a
print(b)   # [1, 2, 3]，对象还在
```

`del a` 从当前命名空间拿掉名字 `a`。对象的生命周期由引用计数（和循环 GC）决定，这篇不展开 GC。和 C++ `delete p` 立刻析构、Go 的 GC 扫不到就回收，心智不同：你几乎从不「释放对象」，你只是丢掉名字。

---

## 二、基本类型

Python 的「基本类型」全是堆上的对象，没有 C++ 那种寄存器宽度的内置标量。`int`、`float`、`bool`、`NoneType`、`str`、`bytes` 都是 class。

### 1、int：任意精度

CPython 的 `int` 是变长数字，内部按 30-bit（64 位平台）或 15-bit（30 位平台）的 digit 数组存。`sys.int_info` 能看出来：

```python
import sys

print(sys.int_info)
# bits_per_digit=30, sizeof_digit=4, ...

print((10**100).bit_length())  # 333
print(sys.getsizeof(0))        # 28：空 digit 数组也有对象头
print(sys.getsizeof(1))        # 28
print(sys.getsizeof(2**30))    # 32：多一个 digit
print(sys.getsizeof(2**60))    # 36
```

对象头（类型指针、引用计数等）就占掉二十多个字节，所以「一个小整数 28 字节」不是浪费，是对象模型的税。C++ `int64_t` 8 字节、Go `int64` 8 字节，热路径上差一个数量级的常数。

运算规则：

```python
print(7 / 2)     # 3.5   真除，永远返回 float
print(7 // 2)    # 3     向下取整除法（不是向零截断）
print(-7 // 2)   # -4    注意：C++ 的 / 对整数通常向零，这里向负无穷
print(7 % 2)     # 1
print(-7 % 2)    # 1     符号跟除数走，满足 n == (n//d)*d + (n%d)
print(2 ** 10)   # 1024
print(pow(2, 10, 1000))  # 24  模幂，第三个参数是模
```

C++11 起整数 `/` 向零截断，`(-7)/2 == -3`；Python 的 `//` 是 floor division，`(-7)//2 == -4`。写移植算法、做哈希取模、做分页偏移时，这个差异会直接出 bug。

```python
# 分页：offset 必须非负
page, size = -1, 10
# C++ 风格直觉：(-1)/10 == 0（向零）
# Python：
print((-1) // 10)  # -1，错页
```

`bool` 是 `int` 的子类，`True == 1`、`False == 0`，能参与算术。不要用这个特性写业务，但读别人代码时要认得出：

```python
print(isinstance(True, int))  # True
print(True + True)            # 2
print({True: "a", 1: "b"})    # {True: "b"}  —— True 和 1 哈希相等，后写的覆盖
```

最后一个例子是面试高频：dict 的 key 用 `==` 和 `hash` 判重，`True`、`1`、`1.0` 三者互等，会撞到同一个槽。

整数没有溢出，但有内存上限。算个几百万位的数没问题；拿去当 list 下标或循环次数，会先把机器撑死。`range(10**18)` 本身不分配那多数，迭代它才会跑到天荒地老——`range` 是惰性对象，不是 list。

### 2、float

Python `float` 就是 C 的 `double`（IEEE 754 binary64）。没有 `float32` 字面量；要单精度走 `array`、`numpy`、`struct`。

```python
print(0.1 + 0.2 == 0.3)   # False
print(0.1 + 0.2)          # 0.30000000000000004
```

和 C++/Go 一样的二进制浮点坑，不是 Python 特有。比较浮点用容差，或用 `decimal.Decimal` / `fractions.Fraction`：

```python
from decimal import Decimal, getcontext
from fractions import Fraction

print(Decimal("0.1") + Decimal("0.2") == Decimal("0.3"))  # True
print(Fraction(1, 10) + Fraction(1, 5) == Fraction(3, 10))  # True
```

`float("inf")`、`float("nan")` 可用。NaN 的 `==` 永远 False，包括和自己：

```python
nan = float("nan")
print(nan == nan)     # False
print(nan is nan)     # True，同一个对象
import math
print(math.isnan(nan))  # True，正确判法
```

混合运算：`int` 遇上 `float` 会把 `int` 转成 `float`。超大 int 转 float 会丢精度，甚至变成 `inf`：

```python
n = 2**53 + 1
print(float(n) == float(n + 1))  # True，53 位尾数装不下
print(float(10**400))            # inf
```

### 3、bool 与 None

`bool` 只有两个实例：`True`、`False`。它们是 `int` 子类，所以 `True + 1 == 2`，但类型注解和 API 里应当当独立类型用。

真值测试（`if obj:`）走 `obj.__bool__`，没有则看 `obj.__len__`，再没有就为真。约定：

| 假 | 真 |
| --- | --- |
| `None` | 非空容器、非零数字、自定义对象默认 |
| `False`、`0`、`0.0`、`0j` | |
| `""`、`b""`、`[]`、`()`、`{}`、`set()` | |

```python
def ok(x):
    if x:           # 空 list、0、"" 全是假
        return True
    return False

print(ok([]), ok(0), ok(""), ok("0"), ok([0]))
# False False False True True
```

后端里最常见的坑：把「没传」和「传了空/0」混在一起。Go 的零值 `""`、`0`、`nil` 本身就是合法值；Python 用 `None` 表示缺省，用空容器表示「有这个字段但是空的」。接口层要分开判断：

```python
def apply_limit(limit: int | None) -> int:
    if limit is None:      # 没传，用默认
        return 100
    if not limit:          # 传了 0，可能是调用方想关掉
        return 0
    return limit
```

`None` 的类型是 `NoneType`，全局单例。注解写成 `x: int | None`（3.10+）或 `Optional[int]`。

### 4、str：不可变、Unicode、编码

Python 3 的 `str` 是 Unicode 码点序列，不可变。`s[0] = "a"` 会 `TypeError`。拼接看起来像修改，其实每次都造新对象：

```python
s = "ab"
print(id(s))
s += "c"
print(id(s))   # 不同 id，旧对象没人引用就回收
```

循环里 `s += chunk` 在 CPython 对「引用计数为 1 的 str」有过优化，但规范不保证。大量拼接用 `"".join(parts)`，或 `io.StringIO`。

索引按码点，不是按字节、也不是按用户看到的「字符」（grapheme cluster）：

```python
s = "中"
print(len(s))          # 1
print(s.encode("utf-8"))  # b'\xe4\xb8\xad'  3 字节

s = "👨‍👩‍👧‍👦"  # 家庭 emoji，多个码点 + ZWJ
print(len(s))          # 7，不是 1
```

和 Go 的 `string`（只读 `[]byte`，`len` 是字节数）、C++ 的 `std::string`（字节）、`std::u32string` 都不同。Python `len(s)` 是码点数。网络协议、文件、checksum 要按字节算，必须先 `encode`。

编码出口只有一处：和外界交换时。内部永远用 `str`；边界用 `bytes`。

```python
raw = b"\xe4\xb8\xad\xe6\x96\x87"
text = raw.decode("utf-8")
back = text.encode("utf-8")
print(text, back)

# 错误策略
try:
    b"\xff".decode("utf-8")
except UnicodeDecodeError as e:
    print(type(e).__name__, e)

print(b"\xff".decode("utf-8", errors="replace"))   # \ufffd
print(b"\xff".decode("utf-8", errors="ignore"))
```

源文件默认 UTF-8（PEP 3120）。Windows 控制台/管道的默认编码仍可能是 gbk，读写文件请显式 `encoding="utf-8"`：

```python
from pathlib import Path

Path("/tmp/aq_demo.txt").write_text("中文", encoding="utf-8")
print(Path("/tmp/aq_demo.txt").read_text(encoding="utf-8"))
```

字符串格式化三选一，新代码用 f-string：

```python
name, n = "q", 3
print(f"{name}={n:02d}")          # q=03
print("{0}={1:02d}".format(name, n))
print("%s=%02d" % (name, n))      # 旧式，C printf 味道，知道能读即可
```

f-string 在 3.12 起允许嵌套引号更自由；3.8+ 的 `f"{expr=}"` 调试很好用：

```python
user_id = 42
print(f"{user_id=}")       # user_id=42
print(f"{user_id=:#x}")    # user_id=0x2a
```

### 5、bytes 与 bytearray

`bytes` 不可变字节序列；`bytearray` 是它的可变版。元素是 0–255 的 int，不是长度为 1 的 bytes——这点和「切片返回同类型」容易混：

```python
b = b"abc"
print(b[0])       # 97，int
print(b[0:1])     # b'a'，bytes
print(list(b))    # [97, 98, 99]

ba = bytearray(b)
ba[0] = 65
print(ba)         # bytearray(b'Abc')
```

字面量：`b"hello"`、`b"\x00\xff"`。`bytes` 只能含 ASCII 字面量，非 ASCII 必须用转义或从 `str.encode` 来。

和 Go 的 `[]byte` / `string` 转换类似：`str.encode` / `bytes.decode` 是拷贝（CPython 对某些只读情况有优化，不要依赖）。和 C++ `std::string` 当字节用相比，Python 强制你在类型上把文本和字节分开——这是 3.x 相对 2.x 最大的正确性改进。

```python
# 网络包、校验、文件头
header = b"\x89PNG\r\n\x1a\n"
print(header.startswith(b"\x89PNG"))

# 十六进制
print(b"abc".hex())                 # '616263'
print(bytes.fromhex("616263"))      # b'abc'
```

---

## 三、intern：小整数池和字符串驻留

intern 是「把相等的不可变对象收成同一个」。为了让 `is` 在某些高频对象上碰巧等于 `==`，也为了省内存。它**不是语言语义**，代码不能依赖它。

### 1、小整数 -5 ~ 256

CPython 启动时把 `[-5, 256]` 做成单例数组 `small_ints`。这个范围内的 `int`，无论怎么算出来，都是同一个对象：

```python
a = 256
b = 255 + 1
c = int("256")
print(a is b is c)   # True

a = 257
b = 256 + 1
c = int("257")
print(a is b)        # 通常 False
print(a == b)        # True
```

边界是实现常量：`NSMALLNEGINTS=5`、`NSMALLPOSINTS=257`，源码在 `Include/internal/pycore_long.h` / `Objects/longobject.c`。PyPy、其他解释器不一定一样。**比较整数永远用 `==`。**

面试追问：「为什么是 -5 到 256？」经验范围：小负数常见于错误码、偏移；256 覆盖一个字节的无符号值。不是语言委员会的承诺。

同一编译单元里的常量折叠会让 257 也变成同一个对象：

```python
def f():
    x = 257
    y = 257
    return x is y

print(f())  # True，字节码里两个 LOAD_CONST 指向同一常量
```

交互式 REPL 一行编译一次，两个 `257` 可能各建一个。所以「我试了 `257 is 257`」的结果取决于你怎么试。

### 2、字符串 intern 的边界

规则比整数更绕，CPython 大致是：

1. 标识符风格的字符串（字母数字下划线）在编译期 intern。
2. `sys.intern(s)` 强制驻留。
3. 运行时拼出来的字符串、带空格/标点的、很长的，默认不 intern。
4. PEP 8 无关，纯实现。

```python
a = "hello"
b = "hello"
print(a is b)          # True，编译期 intern

a = "hello world"
b = "hello world"
print(a is b)          # 多数版本 True（字面量仍可能合并），不要赌

a = "hello"
b = "".join(["h", "ello"])
print(a == b, a is b)  # True False

import sys
c = sys.intern("".join(["h", "ello"]))
print(a is c)          # True，强制驻留后同一对象
```

dict 的 key 查找走哈希，不依赖 intern。intern 的收益是：大量重复的短标识符（属性名、模块名、dict 里的字段名）只存一份，且 `is` 可以当快速路径。自己写业务代码时，几乎不需要调 `sys.intern`；JSON 解析器、ORM 字段名这类会。

Go 的字符串是不可变字节，编译器也会合并字面量，但没有暴露 intern API。C++ `const char*` 字面量在只读段，`std::string` 是新分配。Python 把这件事做成运行时对象池，所以你看得到 `is` 的摇摆。

---

## 四、控制流

Python 用缩进做块，没有 `{}`。TAB 和空格混用是 `TabError`。项目统一 4 空格。

### 1、if / elif / else

```python
status = 404
if status == 200:
    print("ok")
elif status == 404:
    print("missing")
else:
    print("other")
```

没有 C 那种「非 0 即真」的 int 习惯用法之外的东西——空容器也是假，见上一节。链式比较是语法，不是逻辑与的语法糖那么简单：

```python
x = 5
print(1 < x < 10)       # True，等价于 1 < x and x < 10
print(1 < x > 3)        # True，x 只求值一次
print(1 < x < 10 < x)   # False
```

`and` / `or` 返回**最后一个被求值的操作数**，不是规范化的 bool。这是从 C 的「返回操作数」传统来的，Go 的 `&&` `||` 只返回 bool：

```python
print(0 or [] or "ok")   # 'ok'
print(1 and 2 and 3)     # 3
print("" or "default")   # 'default'  —— 配置默认值的老写法
```

新代码更倾向显式：`value if value is not None else default`，避免 `""`、`0` 被当成缺省。

三元：`a if cond else b`，对应 C/Go 的 `cond ? a : b`。

### 2、for、while、for-else

`for` 迭代任意可迭代对象，不是 C 那种 `for(init; cond; step)`。要下标用 `enumerate`，要计数用 `range`。

```python
for i, name in enumerate(["a", "b", "c"], start=1):
    print(i, name)

for i in range(3):
    print(i)   # 0 1 2，不含终点，和 Go 的 for i := 0; i < 3; i++ 一致
```

`range(start, stop, step)` 是惰性序列，不是 list。`range(10**12)` 安全；`list(range(10**12))` 会炸内存。

`for` / `while` 可以带 `else`：**循环没有被 `break` 打断时执行 else**。名字极易误解，把它读成 `nobreak`：

```python
def find(xs: list[int], target: int) -> int | None:
    for x in xs:
        if x == target:
            break
    else:
        return None   # 没找到
    return x

print(find([1, 2, 3], 2))   # 2
print(find([1, 2, 3], 9))   # None
```

等价于 C++ 里设一个 `found` 标志。用得好能少一个变量；用得烂会让读者愣两秒。团队里不熟就别秀。

`while` 和 C/Go 一样。没有 `do-while`，要至少跑一次就 `while True` + `break`。

`pass` 是空语句，对应 C++ 的 `;` 空语句、Go 的 `_ = ...` 那种占位。`...`（`Ellipsis`）在注解和协议里当占位，函数体里更常见的是 `pass` 或 `raise NotImplementedError`。

### 3、match / case（3.10+）

不是 C 的 `switch`，是结构化模式匹配（PEP 634）。主体按值、结构、守卫去匹配，有析构能力。

```python
def http_status(code: int) -> str:
    match code:
        case 200 | 201 | 204:
            return "ok"
        case 301 | 302:
            return "redirect"
        case 400 | 404:
            return "client"
        case 500:
            return "server"
        case _:
            return "other"

print(http_status(201))
```

析构序列和映射：

```python
def handle(event: dict) -> str:
    match event:
        case {"type": "login", "user": str(name)}:
            return f"welcome {name}"
        case {"type": "ping"}:
            return "pong"
        case {"type": t}:
            return f"unknown type {t}"
        case _:
            return "invalid"

print(handle({"type": "login", "user": "aq"}))
print(handle({"type": "ping"}))
```

类模式（和 dataclass / 普通类的 `__match_args__` 配合）：

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

def quadrant(p: Point) -> str:
    match p:
        case Point(0, 0):
            return "origin"
        case Point(x, 0):
            return f"x-axis {x}"
        case Point(0, y):
            return f"y-axis {y}"
        case Point(x, y) if x > 0 and y > 0:
            return "Q1"
        case Point(_, _):
            return "other"

print(quadrant(Point(1, 2)))
print(quadrant(Point(0, 5)))
```

几个易错点：

- `case 400 | 404:` 是或模式，不是按位或。
- `case [x, y]:` 只匹配长度恰好为 2 的序列。
- `case {"type": t}:` 不要求 dict 只有这一个 key，多出来的 key 被忽略；要精确匹配加 `{}` 之外的守卫，或 3.10 的映射剩余模式 `{"type": t, **rest}`。
- `_` 是通配，不绑定。
- `match` 不会 fall-through，匹配到一个 case 就结束，不像 C `switch` 要自己 `break`。
- 主体是表达式一次求值，然后从上到下试模式。

Go 1.18+ 的 `switch` 仍然是值/类型 switch，没有析构。C++17 的 `if (auto [x,y] = ...)` 能拆 tuple，但没有 match 这种多模式分发。Python 的 `match` 更接近 Rust / Scala 的 pattern match，只是没有穷尽性检查——漏掉的情况会静默掉进 `case _` 或什么都不做。

### 4、海象运算符 `:=`（3.8+）

在表达式里赋值并返回值。典型场景：既要判断又要用这个值，避免算两遍或先赋再判。

```python
# 读文件到空行
import io

buf = io.StringIO("abc\n\ndef\n")
while (line := buf.readline()) != "":
    if (s := line.strip()):
        print(s)
```

正则：

```python
import re

text = "user_id=42"
if m := re.search(r"user_id=(\d+)", text):
    print(m.group(1))
```

列表推导里也能用，但可读性下降很快：

```python
raw = ["1", "x", "2"]
nums = [n for s in raw if (n := s.isdigit() and int(s))]
# 这个写法在 s="0" 时会丢掉 0，因为 0 是假值 —— 反例
print(nums)

nums = [int(s) for s in raw if s.isdigit()]  # 正确
print(nums)
```

海象不是「少写一行」的玩具。C++17 `if (auto x = f(); x)`、Go `if v, err := f(); err == nil` 解决的是同一类问题：声明的作用域收在条件里。Python 的 `:=` 作用域规则更绕——在推导式里它会泄漏到外层（3.8 的设计，被骂过，3.9+ 推导式内的 `:=` 仍会绑到 enclosing scope）：

```python
if any((n := i) > 5 for i in range(10)):
    print("found", n)   # n 泄漏到 if 外面，6
```

能不用泄漏当特性就不用。写清楚比写短重要。

### 5、循环控制：break / continue / return

和 C/Go 一样。`for` 里 `continue` 进下一次迭代。没有 label 形式的 break（不像 Go 的 `break Loop`、C 的 `goto`）。多层循环提前逃，包成函数 `return`，或抛异常，或用 for-else。

---

## 五、推导式入门

推导式是「用表达式造容器」的语法，对应 C++ 里手写 for + `push_back`、Go 里 `for + append`。这里只点到能写、能读；内存差异、生成器表达式放到数据结构篇。

```python
squares = [x * x for x in range(6) if x % 2 == 0]
print(squares)  # [0, 4, 16]

index = {name: i for i, name in enumerate(["a", "b", "c"])}
print(index)    # {'a': 0, 'b': 1, 'c': 2}

unique = {x % 3 for x in range(10)}
print(unique)   # {0, 1, 2}
```

嵌套：后写的 for 是内层，和普通嵌套循环顺序一致：

```python
pairs = [(i, j) for i in range(3) for j in range(2)]
print(pairs)  # (0,0)(0,1)(1,0)(1,1)(2,0)(2,1)
```

能读就行。超过两层、或里面有副作用，改回 for 循环。推导式里的名字在 3.x 是局部的，不会污染外层（海象除外）：

```python
x = 10
_ = [x for x in range(3)]
print(x)  # 10，还是外层的 x。Python 2 会覆盖，3 修好了
```

---

## 六、函数最小集

函数是对象。`def` 执行时创建一个 `function` 对象，绑到名字上。默认参数、闭包、作用域下一篇展开，这里只保证能写、能读、能看注解。

```python
def add(a: int, b: int = 0) -> int:
    """两数之和。默认参数只求值一次。"""
    return a + b

print(add(1, 2))
print(add(1))
print(add.__name__, add.__annotations__)
```

没有显式 `return` 就返回 `None`。可以返回 tuple 来实现「多返回值」，其实是一个 tuple：

```python
def divmod_safe(a: int, b: int) -> tuple[int, int] | None:
    if b == 0:
        return None
    return a // b, a % b   # 逗号造 tuple

match divmod_safe(7, 2):
    case None:
        print("div0")
    case q, r:
        print(q, r)
```

Go 的 `(int, error)` 是多返回值；Python 用 tuple / dataclass / 自定义 Result。错误处理默认走异常，不是 error 值，见第九节。

注解**不执行、不强制**。`a: int` 只是元数据，运行时传 string 也不会报错，除非你自己跑 mypy / pyright，或在运行时读 `__annotations__` 做校验（pydantic 那条路）。写法：

```python
from collections.abc import Iterable, Sequence

def concat(parts: Iterable[str], *, sep: str = ",") -> str:
    return sep.join(parts)

def head(xs: Sequence[int]) -> int:
    return xs[0]
```

3.9+ 内置集合可以直接当泛型：`list[int]`、`dict[str, int]`。3.10+ 用 `X | Y` 代替 `Union[X, Y]`、`X | None` 代替 `Optional[X]`。3.11+ 有 `Self`、`typing.Never` 等，类型注解篇再讲。

位置参数、关键字参数、`/` 仅位置、`*` 仅关键字——函数篇展开。现在只要知道调用可以 `add(1, b=2)`，以及不要用可变对象当默认值：

```python
def acc(x: int, bucket: list[int] | None = None) -> list[int]:
    if bucket is None:
        bucket = []
    bucket.append(x)
    return bucket
```

`def acc(x, bucket=[])` 会让所有调用共享同一个 list。这是 Python 第一课到面试终局都会再问一遍的题。

---

## 七、模块与包

### 1、import 在干什么

`import os` 做三件事：找到模块、执行它的顶层代码（第一次）、把模块对象绑到当前名字 `os`。模块对象缓存进 `sys.modules`，再 import 同一模块不会重新执行。

```python
import sys
import json as js          # 改名，对应 C++ namespace alias / Go import alias
from pathlib import Path   # 把 Path 这个名字绑过来
from math import sqrt, pi
from typing import Any     # 只为注解时，3.7+ 可 from __future__ import annotations 延迟求值
```

`from math import *` 会覆盖你的名字，禁止在非交互式代码里用。`from pkg import name` 如果 `name` 是子模块，行为依赖 `pkg/__init__.py` 有没有 import 它——循环依赖的温床。

查找顺序（简化）：

1. `sys.modules` 缓存
2. 内置模块
3. `sys.path`：脚本目录、`PYTHONPATH`、site-packages

```python
import sys
print(sys.path[:3])
```

### 2、`if __name__ == "__main__"`

模块有 `__name__`。被直接执行时是 `"__main__"`；被 import 时是模块名。这个 idiom 把「库」和「入口」放在同一文件：

```python
# app.py
def main() -> None:
    print("run")

if __name__ == "__main__":
    main()
```

Go 的 `package main` + `func main()`、C++ 的 `int main()` 是语言级入口。Python 任何文件都能当入口，所以要用这句把副作用挡住，否则 `from app import main` 会把脚本逻辑跑一遍。

测试、benchmark、脚本参数解析，都放进 `main()`，不要摊在顶层。顶层只放 import、常量、函数和类定义。

### 3、包

目录里有 `__init__.py`（3.3+ 命名空间包可以没有，但库代码写上更清楚）就是包。`pkg/__init__.py` 在 `import pkg` 时执行，常用来暴露公开 API：

```
mylib/
    __init__.py      # from .core import Client
    core.py
    utils.py
```

相对 import：`from .core import Client`、`from .. import other`。只在包内有效，直接 `python mylib/core.py` 会炸，所以包内模块不要当脚本跑，用 `python -m mylib.core`。

### 4、venv 与 pip

系统 Python 不要直接 `pip install`。虚拟环境把解释器和 `site-packages` 隔开，对应 Go 的模块级依赖、C++ 的 conan/vcpkg 隔离，只是粒度在「这个项目的解释器」。

```bash
python3 -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate
python -m pip install -U pip
python -m pip install requests
python -m pip freeze > requirements.txt
python -m pip install -r requirements.txt
```

`python -m pip` 比直接 `pip` 更能保证装进当前这个解释器。3.11+ 有 `tomllib`；项目元数据现在主流是 `pyproject.toml`（PEP 621），`requirements.txt` 仍然是部署场景最常见的锁清单。锁版本用 `pip freeze` 或 uv/poetry 的 lockfile。

```bash
deactivate
```

`PYTHONPATH` 可以应急，项目里更干净的做法是把包 `pip install -e .` 成可编辑安装，或保证从仓库根目录 `python -m pkg.mod`。

---

## 八、常用内置

这些函数是日常密度最高的，C++ 里对应 `<algorithm>`，Go 里对应 `for` 手写或 `slices` 包。

### 1、range

```python
print(list(range(3)))          # [0, 1, 2]
print(list(range(1, 5, 2)))    # [1, 3]
print(list(range(5, 1, -1)))   # [5, 4, 3, 2]
print(3 in range(1000000000))  # True，O(1) 判断，不是扫描
```

`range` 实现了 `__contains__` 的 O(1) 算术判断（步长非零时）。它不是迭代器，是惰性序列：可多次迭代、可索引、可切片。

```python
r = range(10)
print(r[2], r[-1], r[2:8:2])  # 2 9 range(2, 8, 2)
```

### 2、enumerate 与 zip

```python
names = ["a", "b", "c"]
for i, name in enumerate(names):
    print(i, name)

# 和另一个序列对齐
ids = [10, 20, 30]
for i, name in zip(ids, names):
    print(i, name)

# 长度不一致：默认停在最短的。3.10+ strict=True 会抛
try:
    list(zip([1, 2], [1, 2, 3], strict=True))
except ValueError as e:
    print(e)
```

C++20 `std::views::enumerate`、Go 的 `for i, v := range` 是语言内置。Python 的 `for x in xs` 不给下标，逼你想清楚是不是真的需要 index——多数时候不需要。

`zip` 返回迭代器，不是 list。`zip(*matrix)` 是转置的惯用写法：

```python
matrix = [[1, 2, 3], [4, 5, 6]]
print(list(zip(*matrix)))  # [(1, 4), (2, 5), (3, 6)]
```

### 3、sorted / reversed / min / max

```python
xs = [3, 1, 2]
print(sorted(xs), xs)          # [1,2,3] 原 list 不动
xs.sort()                      # 原地，只 list 有
print(xs)

print(sorted(["ab", "a", "abc"], key=len))
print(sorted([{"n": 2}, {"n": 1}], key=lambda d: d["n"]))
```

`key` 只调一次，比 `cmp` 快（3.x 砍掉了 `cmp`，要的话 `functools.cmp_to_key`）。稳定性：相等元素保持原序，和 C++ `std::stable_sort`、Go `slices.SortStableFunc` 一类。

```python
print(min(3, 1, 2))
print(max(xs, default=-1))     # 空序列时 default，否则 ValueError
print(sum([1, 2, 3], start=0))
```

`sum` 对 `str` 会故意报错，逼你用 `"".join`。`sum(list_of_lists, [])` 能跑但平方复杂度，拼接用推导式或 `itertools.chain`。

### 4、any / all

```python
print(any(x > 2 for x in [0, 1, 3]))  # True，短路
print(all(x > 0 for x in [1, 2, 3]))  # True
print(all([]), any([]))               # True False  空量词约定
```

`all([])` 为 True、`any([])` 为 False，是谓词逻辑的 vacuous truth，不是 bug。生成器表达式丢给 `any`/`all` 不会造中间 list。

### 5、map / filter 先认识一下

```python
print(list(map(str, range(3))))
print(list(filter(lambda x: x % 2 == 0, range(6))))
```

可读性通常不如推导式 `[str(x) for x in range(3)]`。需要惰性、或已经有现成函数对象时用 `map`/`filter`。函数篇会对比。

### 6、其它每天会碰到的

```python
print(len("ab"), len([1, 2]), len({1: 2}))
print(isinstance(True, int))     # True
print(issubclass(bool, int))     # True
print(callable(len))             # True
print(repr("a\nb"))              # 给调试看的合法字面量
print(ascii("中"))               # 非 ASCII 转义
print(hex(255), bin(8), oct(8))
print(round(2.5), round(3.5))    # 2 4  银行家舍入：偶数优先，不是「四舍五入」
```

`round` 的银行家舍入（banker's rounding / round half to even）和很多语言的「四舍五入」不同，写金额、写测试断言时要对齐。

`isinstance(obj, (int, float))` 可以传 tuple。3.10+ 也可用 `isinstance(obj, int | float)`。

---

## 九、异常最小集

Python 的错误默认走异常，不是 Go 的 `error` 返回值，也不是 C++ 那种「可以抛也可以错误码」的混合。调用方不接就往上冒，直到进程干掉并打 traceback。

### 1、try / except / else / finally

```python
def parse_port(s: str) -> int:
    try:
        port = int(s)
    except ValueError as e:
        raise ValueError(f"invalid port: {s!r}") from e
    else:
        # try 没抛才走这里。把「成功路径」和 except 分开
        if not (0 < port < 65536):
            raise ValueError(f"port out of range: {port}")
        return port
    finally:
        # 无论是否抛，都会走。return 也挡不住
        pass

print(parse_port("8080"))
```

`else` 不是装饰：它让你避免在 `try` 里塞太多代码，从而误捕「不该捕的异常」。`finally` 做清理，对应 Go 的 `defer`、C++ 的 RAII 析构。文件更推荐 `with`（上下文管理器），不要手写 `f = open(); try: ... finally: f.close()`：

```python
from pathlib import Path

p = Path("/tmp/aq_port.txt")
p.write_text("8080", encoding="utf-8")
with p.open(encoding="utf-8") as f:
    print(parse_port(f.read().strip()))
```

`with` 本质是 `contextmanager.__enter__ / __exit__`，异常发生时 `__exit__` 仍被调用，可以吞掉或继续抛。

### 2、捕获要精确

```python
try:
    n = int("x")
except ValueError:
    n = 0

# 不要这样
try:
    n = int("x")
except Exception:
    n = 0

# 更不要
try:
    n = int("x")
except:          # 连 KeyboardInterrupt、SystemExit 都捕
    n = 0
```

`except:` 和 `except BaseException` 会把 Ctrl-C 吃掉。最低限度用 `except Exception`，更好是捕具体类型。多个类型：`except (ValueError, TypeError) as e:`。

捕获之后还想抛出去：`raise` 光秃秃一个，会把当前异常原样再抛，traceback 完整。`raise NewError(...) from e` 是显式链式，`__cause__` 指向 `e`。`raise NewError(...) from None` 切断链，调用方看不到底层原因——库边界偶尔用，业务代码少用。

### 3、raise 与异常类型

```python
class ConfigError(Exception):
    pass

def load(path: str) -> str:
    if path == "":
        raise ConfigError("empty path")
    return path
```

自定义异常继承 `Exception`，不要继承 `BaseException`。内置里后端常用的：

| 类型 | 何时 |
| --- | --- |
| `ValueError` | 类型对，值不对 |
| `TypeError` | 类型不对 |
| `KeyError` / `IndexError` | 键/下标不存在 |
| `AttributeError` | 属性不存在 |
| `FileNotFoundError` | 文件没有（是 `OSError` 子类） |
| `TimeoutError` | 超时 |
| `RuntimeError` | 不想细分时的口袋 |

`assert` 抛 `AssertionError`，可以用 `-O` 关掉。**不要用 assert 做输入校验**，生产加了 `-O` 校验会消失。assert 只表达内部不变量。

```python
def pick(xs: list[int], i: int) -> int:
    if not 0 <= i < len(xs):
        raise IndexError(i)
    return xs[i]
```

### 4、和 C++ / Go 的错误心智

C++：异常、`std::expected`、错误码三套并存，项目里通常定一套。析构不能抛。  
Go：`if err != nil`，错误是值，panic/recover 留给真正的致命问题。  
Python：异常是主路。对「可预期的失败」（文件不存在、JSON 坏了）也抛。热点路径上异常比返回错误码贵（要建 traceback），所以解析循环里有时用 `dict.get` / 哨兵代替「先抛再捕」。

```python
d = {"a": 1}
print(d.get("b", 0))     # 不抛
print(d["b"] if "b" in d else 0)

# 反模式：用异常做控制流
try:
    v = d["b"]
except KeyError:
    v = 0
```

最后一种能跑，但比 `get` 慢、比 `in` 吵。EAFP（Easier to Ask Forgiveness than Permission）是 Python 文化：先做，捕异常。LBYL（Look Before You Leap）是先判断。文件存在性这种 TOCTOU 场景，EAFP 更正确；dict 缺 key 这种，`get` 更干净。

---

## 十、和 C++ / Go 的心智对照表

| 维度 | C++ | Go | Python |
| --- | --- | --- | --- |
| `int a = 1` | 栈上 4/8 字节标量 | 栈上机器字 | 名字绑到堆上 `PyLongObject` |
| 赋值 `b = a` | 拷贝值（或调拷贝ctor） | 拷贝值（slice/map 是头） | 拷贝引用（名字） |
| 可变性 | `const`、指针/引用自己管 | 值类型拷贝，切片共享底层 | 对象自己可变或不可变，名字不管 |
| 零值 | 不初始化是 UB；`{}` 零初始化 | 每个类型有零值，可用 | 名字不存在就 `NameError`；`None` 是对象不是零值 |
| 可见性 | `public`/`private`、匿名命名空间 | 首字母大小写跨包 | 约定 `_prefix` 私有，运行时拦不住 |
| 错误 | 异常或错误码 | `error` 值 + panic | 异常为主 |
| 字符串 | 字节 / `u32string` | 只读字节，`len` 字节数 | Unicode 码点，`len` 码点数 |
| 整数溢出 | UB 或绕回 | 绕回 | 变大，不溢出 |
| 入口 | `main` | `package main` `func main` | 任何文件 + `__name__ == "__main__"` |
| 类型 | 编译期强制 | 编译期强制 | 运行时鸭子；注解可选 |
| 迭代 | 下标或迭代器 | `for range` 给下标和值 | `for x in xs` 只给元素 |
| `==` | 值或重载 | 可比较类型比值 | `__eq__`；身份用 `is` |
| 空指针 | `nullptr`，解引用 UB | `nil`，解引用 panic | `None`，属性访问 `AttributeError` |
| 泛型容器 | `vector<T>` | `[]T` `map[K]V` | `list` 异构，注解 `list[int]` 不强制 |

零值这一行再钉一下。Go 里 `var s string` 是 `""`，`var n int` 是 `0`，`var p *T` 是 `nil`，你可以立刻用。Python 里 `s: str` 只是注解，没赋值就没有这个名字：

```python
def f() -> None:
    x: int
    # print(x)  # NameError: x is not defined

f()
```

「声明类型」和「引入名字」是两件事。类属性、函数参数默认值、`dataclasses` 的字段才会真正把名字造出来。从 Go 转过来的人，这里会连续踩坑。

可见性：`_name` 是约定「内部使用」，`from pkg import *` 不会导入它（除非 `__all__` 写了）。`__name` 在类里会触发名称改写（name mangling）变成 `_Class__name`，防的是子类覆盖，不是安全机制。真正想藏，做不到——`obj.__dict__`、`getattr` 都能拿。这和 C++ `private`、Go 的包级小写完全不是一个强度。

---

## 十一、几个会在代码审查里出现的具体问题

### 1、可变默认值，再看一眼

```python
def append_item(x, xs=[]):
    xs.append(x)
    return xs

print(append_item(1))
print(append_item(2))   # [1, 2]，不是 [2]
```

默认值存在函数对象的 `__defaults__` 里，定义时求值一次。下一篇会从函数对象的角度拆开。现在记住：默认值只用不可变对象，可变的用 `None` 占位。

### 2、`==` 和哈希一致性

```python
print(1 == 1.0 == True)
print(hash(1) == hash(1.0) == hash(True))
print({1, 1.0, True})   # {1}  三个是同一个集合元素
```

不要用 `True`/`False` 当 dict key 和 `0`/`1` 混用。JSON 里 `true` 和 `1` 进 Python 后类型不同，但一旦丢进 set/dict 会撞车。

### 3、字符串拼接与 `join`

```python
# 慢的写法（每次新对象；CPython 有时优化，别赌）
s = ""
for i in range(10000):
    s += str(i)

# 稳的写法
s = "".join(str(i) for i in range(10000))
```

### 4、文件编码

```python
# 错：依赖 locale
# open("a.txt").read()

# 对
open("/tmp/aq_demo.txt", encoding="utf-8").read()
```

Windows 上 locale 常是 cp936，UTF-8 文件会读爆。Linux 服务器 UTF-8 碰巧能过，同一段代码换机器就挂。显式 `encoding` 是纪律，不是风格。

### 5、`is` 比较字符串

```python
status = "".join(["O", "K"])
if status is "OK":      # 可能 False，SyntaxWarning: "is" with a literal
    print("fast path")
if status == "OK":      # 永远按值
    print("ok")
```

3.8+ 对 `is` 配字面量会警告。听它的。

### 6、作用域：循环变量泄漏

```python
for i in range(3):
    pass
print(i)   # 2，for 的名字留在外层

print([j for j in range(3)])
# print(j)  # 3.x NameError，推导式有自己的作用域
```

Go 1.22 修过循环变量每次迭代新变量的问题；Python 的 `for` 就是一个名字反复换绑。闭包延迟绑定那一坑，函数与作用域篇专门打。

---

## 十二、可运行的对照实验

把前面几条最容易「我以为我懂了」的点串成一个脚本。直接 `python` 跑：

```python
"""binding_vs_box.py — 名字绑定对照实验。Python 3.11+"""

from __future__ import annotations

import sys


def section(title: str) -> None:
    print(f"\n== {title} ==")


def main() -> None:
    section("1. sizeof 不是 4 字节")
    print("sizeof(1) =", sys.getsizeof(1))
    print("sizeof(2**100) =", sys.getsizeof(2**100))

    section("2. 绑定不是拷贝")
    a = [1, 2]
    b = a
    a.append(3)
    print("a", a, "b", b, "same", a is b)

    section("3. is vs ==")
    x = [1]
    y = [1]
    print("x == y", x == y, "x is y", x is y)
    print("None is None", None is None)

    section("4. 小整数 intern")
    print("256 is 255+1", (256) is (255 + 1))
    print("257 is 256+1", (257) is (256 + 1))

    section("5. 字符串 intern 边界")
    s1 = "hello"
    s2 = "hello"
    s3 = "".join(["hel", "lo"])
    print("literal is", s1 is s2)
    print("joined is", s1 is s3, "equal", s1 == s3)

    section("6. True/1 撞 key")
    print({True: "bool", 1: "int", 1.0: "float"})

    section("7. floor division")
    print("-7 // 2 =", -7 // 2, "  -7 % 2 =", -7 % 2)

    section("8. for-else")
    for n in [1, 3, 5]:
        if n % 2 == 0:
            print("even", n)
            break
    else:
        print("no even")

    section("9. match")
    code = 201
    match code:
        case 200 | 201 | 204:
            print("ok-ish")
        case _:
            print("other")

    section("10. 海象")
    text = "id=9"
    if (p := text.partition("="))[0] == "id":
        print("id", p[2])


if __name__ == "__main__":
    main()
```

跑完对照这张清单：sizeof 远大于 4；`b = a` 共享 list；`is` 不等于 `==`；256 intern、257 不保证；`True` 和 `1` 是同一个 dict key；`//` 向负无穷；for-else 在没 break 时触发。

这些现象全部来自同一件事：名字是引用，对象在堆上，类型是对象自己的属性而不是名字的属性。

```python
x = 1
print(type(x))
x = "hello"
print(type(x))   # 名字换绑到另一种对象，C++/Go 直接编译失败
```

动态类型不是「没有类型」，是「类型跟着对象走，名字不锁类型」。注解是给人（和 type checker）看的合同，解释器默认不执行合同。入门阶段把这个合同当文档写上，比追求「纯动态」更接近后端工程的用法。

下一篇用容器把「共享引用」推到会让线上变慢、变错的地方：`lst += [x]` 的平方复杂度、`list.pop(0)` 的队列、浅拷贝和默认参数里的那个 list。
