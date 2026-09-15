# Python面试连环问

面 Python 岗，面试官很少停在「GIL 是全局锁」。会顺着你的第一句往下挖：锁保护什么、哪条 bytecode 会放、`x += 1` 为什么丢更新、3.13 的 free-threaded 是不是默认。下面按真实出现过的点写，每题先给能说出口的答案，再「往下追」给实现、反例、和 C++/Go 的对照。代码按 CPython 3.11–3.13。

---

### 1、Python 的变量是名字绑定，还是盒子？

能说出口：名字是标签，对象在堆上。`a = 1` 不是在栈上开 4 字节盒子把 1 写进去，是把当前命名空间里的名字 `a` 绑到一个 `int` 对象上。`b = a` 复制的是绑定，不是对象。

往下追：**赋值改的是名字，还是对象？** 不可变对象上，`a = a + 1` 是换绑；可变对象上，`a.append(1)` 是原地改，所有绑到它的名字都看见。

```python
a = [1, 2]
b = a
a.append(3)
print(b)            # [1, 2, 3]，b 不是副本
print(a is b)       # True

x = 1000
y = x
x = 1001
print(y)            # 1000：x 换绑，y 仍指旧 int
```

函数传参同一套：形参是新名字，绑到同一个对象。看起来「int 像传值、list 像传引用」，本质都是传引用，差别只在对象能不能原地改。

C++ 的 `int a = 1` 是盒子；`int& r = a` 才是绑名字。Go 的 `n int` 是值拷贝，`[]int` 拷的是 slice 头（底层数组共享）。Python 没有这层区分。面试听到「Python 传引用」要补一句：传的是对象引用，不是 C++ 那种可 reseat 的 `T&`。

再追：**命名空间是什么？** 函数局部 3.11+ 是加速的局部数组（不是每步 dict），自由变量走 closure cell，模块和 class 体是 `dict`。`locals()` 在函数里是快照，改它不写回。

---

### 2、可变 vs 不可变，为什么 list 不能当 dict 的 key？

能说出口：不可变对象值不变、身份稳定，默认按值哈希；可变对象能原地改，哈希一旦进桶就不能变，否则找不到也删不掉。`list` / `dict` / `set` 没实现 `__hash__`（准确说是 `__hash__ = None`），塞进 dict/set 立刻 `TypeError`。

往下追：**tuple 一定能当 key 吗？** 不能。tuple 本身不可变，但元素如果不可哈希，整体也不可哈希。

```python
print(hash((1, 2)))
# hash(([1], 2))     # TypeError: unhashable type: 'list'
d = {}
# d[[1]] = 1         # TypeError
d[(1, 2)] = 1
```

**自己的 class 呢？** 默认 `__hash__` 按身份，`__eq__` 也按身份，能当 key，但两个「字段相同」的实例是两个 key。改了 `__eq__` 必须改 `__hash__`，且哈希用的字段在对象寿命内不能变。dataclass `frozen=True` 会自动做对；可变 dataclass 默认 `__hash__ = None`。

C++ `std::unordered_map` 的 key 也要求哈希期间等价关系稳定，但编译器不会拦你把 `vector` 当 key——你自己写 hash，改 vector 就是 UB 级的逻辑错误。Go 的 map key 必须 comparable，`slice` 直接编译失败。Python 把「不可哈希」做成运行时错误，检查器在注解里用 `Hashable`。

再追：**字符串不可变，为什么还能 `s += "x"`？** 那是换绑：新建一个 str，名字 `s` 指向新对象。循环里 `s += chunk` 是 O(n²) 分配（解释器对 `+=` 有过优化，但不要赌）；用 `join`。

---

### 3、浅拷贝和深拷贝差在哪？

能说出口：浅拷贝新容器、元素还是原来那些对象；深拷贝递归复制，碰到环用 memo 表。`list.copy()` / `xs[:]` / `copy.copy` 是浅的；`copy.deepcopy` 是深的。

往下追：**什么时候浅拷贝就够？** 元素全是不可变（`list[int]`、`list[str]`），浅拷贝已经切断容器共享。元素是 list/dict，浅拷贝只复制了外层。

```python
import copy

inner = [1, 2]
a = [inner, 3]
b = a.copy()
c = copy.deepcopy(a)
inner.append(9)
print(a, b, c)      # [[1,2,9], 3]  [[1,2,9], 3]  [[1,2], 3]
```

**环怎么处理？** `deepcopy` 用 `memo` 字典：id → 新对象。先进 memo 再递归，环不会爆栈。自己写拷贝漏 memo，环上会无限递归。

**切片是浅拷贝。** `a[:]` 新 list，元素指针一样。`l = l[:]` 切断的是外层绑定，不是嵌套对象。

C++ `vector<T>` 拷贝构造按 T 的拷贝语义，`vector<unique_ptr<T>>` 根本不能拷。Go 的 `append(append([]T{}, xs...), )` 拷 slice 头+元素值；元素是指针则仍共享。Python 默认赋值从不拷对象，必须显式 `copy`。

再追：**dataclass / pydantic 怎么拷？** `dataclasses.replace` 浅；嵌套 dataclass 要自己递归或 `deepcopy`。pydantic `model_copy(deep=True)` 走模型自己的深拷。

---

### 4、默认参数为什么不能写 `def f(xs=[])`？

能说出口：默认值在 **`def` 执行时求值一次**，挂在 `f.__defaults__` 上，所有没传这个参数的调用共享同一个对象。可变对象被原地改，下次调用还在。

```python
def f(a, items=[]):
    items.append(a)
    return items

print(f(1), f(2), f(3, ["x"]), f(4))
# [1]  [1,2]  ['x', 3]  [1,2,4]
print(f.__defaults__)     # ([1, 2, 4],)
```

往下追：**正确写法？** `None` 哨兵，函数体内新建。

```python
def f(a, items=None):
    if items is None:
        items = []
    items.append(a)
    return items
```

**不可变默认就没事吗？** `def f(n=1)` 安全，因为没人能原地改 int。`def f(t=())` 也安全。别把「默认参数一律 None」当成教条，不可变字面量更干净。

C++ 默认参数是调用点填临时量，`void f(vector<int> v = {})` 每次一份新的（拷贝/移动）。Go 没有默认参数。Python 的坑来自「函数是对象、默认值是对象上的槽」。

再追：**`def f(ts=time.time())` 会怎样？** 定义时的时间戳，不是调用时。要调用时取，写 `None` 再在体内 `time.time()`，或不要默认值。dataclass 的 `field(default_factory=list)` 是同一问题的正解。

---

### 5、`is` 和 `==` 差什么？小整数为什么有时 `is` 也成？

能说出口：`is` 比身份（`id()`，CPython 里是地址），`==` 走 `__eq__`。判断单例（`None` / `True` / `False` / `...`）用 `is`；比值用 `==`。不要写 `x == None`。

往下追：**小整数 intern。** CPython 启动时把 `[-5, 256]` 做成单例池，池内 `is` 和 `==` 碰巧一致。池外每次 `int()` 可能是新对象。

```python
print(256 is 256)           # True，池内
print(257 is 257)           # 常量折叠，编译期可能仍是同一对象
x = 257
y = 256 + 1
print(x is y)               # 常见 False，实现细节，不要依赖
print(-6 is -6)             # 字面量；运算出来的 -6 经常不是
```

**字符串 intern。** 标识符、代码里的字面量、`sys.intern` 过的 str，身份可能相同。用户输入的 `"hello"` 和字面量 `"hello"` 在 3.x 里经常 intern，**不是语言保证**。比内容用 `==`。

C++ `==` 对指针比地址、对值类型比值；Go 的 `==` 对 interface 比动态类型+值，对 slice 不能 `==`。Python 把身份和相等拆成两个算符，小整数池是解释器的分配优化，不是语义。

再追：**`True == 1`、`False == 0`？** 是，`bool` 是 `int` 子类。`True is 1` 为 False。dict 里 `True` 和 `1` 是同一个 key：`{True: "a", 1: "b"}` 只剩一个。这是哈希表用 `__eq__` 不是 `is` 的后果。

---

### 6、list / tuple / dict / set 底层是什么，复杂度？

能说出口：

| | 底层 | 下标 | 末尾增删 | 头上增删 | 查找 |
| --- | --- | --- | --- | --- | --- |
| list | 连续 `PyObject*` 数组，over-allocate | O(1) | 均摊 O(1) | O(n) | O(n) |
| tuple | 连续数组，定长 | O(1) | 不能 | 不能 | O(n) |
| dict | 开放寻址哈希表 + 插入序索引 | — | — | — | 均摊 O(1) |
| set | 哈希表，无顺序索引 | — | — | — | 均摊 O(1) |

往下追：**list 不是链表。** `pop(0)` / `insert(0, x)` 搬后面所有指针，一万次是平方。队列用 `collections.deque`。`+= [x]` 比 `append` 多一次临时 list 分配。

**dict 3.7+ 有序**是语言保证（3.6 CPython 已经有，3.7 写进规范）。实现是「哈希表 + 稠密条目数组」，迭代走条目数组，所以保插入序，且比 3.5 更省。`OrderedDict` 还提供 `move_to_end` / 按等值比较顺序，普通 dict 只保证迭代序。

**set 无序**，3.14 之前不要依赖迭代顺序。`{1, 2}` 和 `{2, 1}` 相等。

C++：`vector` ≈ list（但装的是 T 不是指针，除非 `vector<T*>`）；`unordered_map` ≈ 3.5 的 dict（链地址、无序）；`map` 是树。Go：`slice` ≈ list 的连续段，`map` 无序。Python list 装指针，所以 `list[int]` 和 `list[str]` 运行时是同一种对象，异构合法，缓存不友好。

再追：**为什么 `for k in d: d.pop(k)` 会炸？** 迭代时改 size 触发 `RuntimeError: dictionary changed size during iteration`。先 `list(d)` 再改，或收集要删的 key。

---

### 7、生成器和迭代器差什么？

能说出口：迭代器是实现 `__next__` / `__iter__` 的对象，一次性、有状态。生成器是用 `yield` 写出来的迭代器，调用生成器函数返回生成器对象，不跑函数体；每次 `next` 跑到下一个 `yield`，帧冻住。

```python
def g():
    print("start")
    yield 1
    yield 2

it = g()                # 不打印 start
print(next(it))         # start / 1
print(list(g()))        # 新对象，再跑一遍
```

往下追：**可迭代 vs 迭代器。** 可迭代有 `__iter__`，每次返回新迭代器（list、dict）。迭代器的 `__iter__` 返回 self，耗尽后空。`for x in xs` 反糖成 `it = iter(xs); while True: next(it)` 直到 `StopIteration`。

**生成器表达式 vs 列表推导。** `(x*x for x in xs)` 惰性；`[x*x for x in xs]` 立刻物化。文件过滤用生成器，别 `readlines()` 再推导。

**`yield from` 不是 `for x in sub: yield x`。** 它转发 `send` / `throw` / `close`，并把子生成器的 `return` 值拿回来。管道用 `yield from`。

C++ 的 iterator 是指针语义，ranges 是惰性视图。Go 1.23 `iter.Seq` 是 push 型回调。Python 生成器是拉模型，帧在堆上，比 goroutine 更轻，也没有抢占——卡在纯计算里谁也切不走，见第 12 题。

再追：**生成器里 `with` / 锁？** `yield` 把 `__exit__` 推迟到生成器被关。调用方不耗完又不 `close()`，文件、锁泄漏。`yield from` 包在 `with` 里，或文档写明必须耗尽。

---

### 8、装饰器怎么实现的，`functools.wraps` 干什么？

能说出口：`@deco` 就是 `f = deco(f)`。`deco` 吃函数、返回函数（或原函数）。带参装饰器是三层：外层收参数、中层收函数、内层收调用。

```python
from functools import wraps

def timed(fn):
    @wraps(fn)
    def inner(*args, **kwargs):
        return fn(*args, **kwargs)
    return inner

@timed
def login(user: str) -> str:
    return user
```

往下追：**不写 wraps 丢什么？** `__name__`、`__doc__`、`__module__`、`__annotations__`、`__qualname__`。按 `__name__` 注册路由会全部变成 `"inner"`，后注册覆盖先注册。`inspect.signature` 默认跟 `inner(*args, **kwargs)` 走，文档和类型检查一起坏。`wraps` 把这些拷回来，并设 `__wrapped__` 指原函数。

**叠放顺序。** 靠近 `def` 的先包：`@a @b def f` ≡ `f = a(b(f))`。执行时先外后内（先 `a` 的 inner，再 `b` 的 inner，再原函数）。

**方法上的 deco。** inner 要能收 `self`：`def inner(self, *args, **kwargs)` 或 `*args`。`@classmethod` / `@staticmethod` 是描述符，和自定义 deco 的叠放顺序会改绑定，classmethod 一般放最靠近 `def`。

C++ 没有装饰器，近亲是包装对象 / 宏。Go 中间件 `func(http.Handler) http.Handler` 同一件事，没有 `@`，也不会丢名字。Python 把函数当对象改写，糖和坑一起出现。

再追：**类型上怎么保签名？** `ParamSpec` + `TypeVar`，见类型注解篇。`wraps` 保运行时身份，`ParamSpec` 保检查器，两件都要。

---

### 9、闭包为什么会「拿到循环最后那个 i」？

能说出口：闭包绑的是**变量（cell）**，不是当时的值。循环里 `lambda: i` 全部指向同一个 `i`，函数真正跑的时候才去读，那时循环已经结束，`i` 是最后一次的值。

```python
fs = [lambda: i for i in range(3)]
print([f() for f in fs])        # [2, 2, 2]

fs = [lambda i=i: i for i in range(3)]
print([f() for f in fs])        # [0, 1, 2]  默认参数在定义时求值，拷了当时的 i
```

往下追：**`late binding` 不是 bug，是作用域规则。** `def` / `lambda` 里读自由变量，查找发生在调用时。`i = i` 默认参数是「定义时求值」的旁路。3.x 还有 `[:]` 在推导里造新 cell 的写法，不如默认参数直观。

**`nonlocal`。** 内层要给外层名字重新绑定（不是原地改对象），必须 `nonlocal x`。只读、或改 `x.append` 不用。漏写 `nonlocal`，内层 `x = 1` 会造局部名，外层不动；若还读过外层 `x`，编译期 `SyntaxError: no binding for nonlocal` 或运行时 `UnboundLocalError`（先读后写没声明）。

C++ lambda `[=]` 按值捕获当时的副本，`[&]` 按引用，循环里 `&i` 同一坑。Go 的循环变量在 1.22 之前每次迭代共享，`go func() { print(i) }` 全是最后那个；1.22 起每次迭代新变量。Python 没有「按值捕获」语法，默认就是 Go 1.21 的坑。

再追：**闭包会延长对象寿命。** 生成器 / 装饰器 / 回调把 cell 拿出去，cell 指向的对象（大 list、连接）活到闭包死。泄漏排查经常挖到这里，见第 19 题。

---

### 10、GIL 锁的是什么、不锁什么，3.13 怎样？

能说出口：GIL 是 **CPython 解释器锁**，不是语言的一部分。同一时刻一个进程里默认只有一个线程在跑 Python bytecode。它保护引用计数、对象内部结构、解释器全局状态。**不保护你的业务不变量。** `x += 1` 照样丢更新。

往下追：**何时释放？** 阻塞 I/O / `sleep` 进内核前放；numpy 一类 C 扩展在纯 C 循环里主动放；纯 Python 按时间片（默认 5ms）让出——那是交错不是并行。CPU 密集纯 Python，多线程墙钟不降；I/O 密集能叠等待。

**3.12：** PEP 684，每个 sub-interpreter 一把独立 GIL，不是取消 GIL。**3.13：** 可选 free-threaded 构建（PEP 703），**官方默认仍有 GIL**。扩展要适配；单线程有代价。面试别说「3.13 没 GIL 了」。

```python
import threading
x = 0

def bump():
    global x
    for _ in range(100_000):
        x += 1                  # LOAD / ADD / STORE，中间可切换

ts = [threading.Thread(target=bump) for _ in range(4)]
for t in ts: t.start()
for t in ts: t.join()
print(x)                        # 经常 < 400000
```

C++ 没有进程级解释器锁，`std::thread` 算 CPU 能并行。Go 的 goroutine 在 GOMAXPROCS 个 M 上并行跑 CPU。CPython 默认把 bytecode 串行化。细节在并发篇，这里够用的三句：锁在实现上、I/O 放锁、CPU 用进程或下沉到放锁的 C。

---

### 11、多线程、多进程、asyncio 怎么选？

能说出口：先看负载。等 fd / 网络 → 线程或 asyncio；纯 Python CPU → 进程；numpy / hashlib 大块（自己放 GIL）→ 线程可能有用。别先开池子再问为什么没加速。

| 负载 | 线程（默认 GIL） | 进程 | asyncio |
| --- | --- | --- | --- |
| socket / sleep | 能叠等待 | 能，重 | 同线程更轻 |
| 纯 Python 循环 | 墙钟几乎不降 | 能降 | 更差 |
| 放 GIL 的 C 扩展 | 可能降 | 能降 | 别放进 loop |
| 混合 | 只加速 I/O 段 | 加速计算段 | I/O 用 asyncio，计算丢 executor |

往下追：**asyncio 里调 `requests`？** 冻事件循环。Go 的 `http.Get` 会把 G 从 M 上卸下；`requests` 是阻塞，loop 那条线程睡在 syscall 上，别的协程全停。同步 SDK 用 `asyncio.to_thread` / `run_in_executor`。

**进程的代价。** 启动（spawn 要重新 import）、pickle 参数和返回值、共享内存才免拷。macOS / Windows 默认 spawn，入口必须 `if __name__ == "__main__"`。循环对象、lambda、生成器 pickle 不了。

**线程池无界队列。** `ThreadPoolExecutor(32)` 再 `submit` 一万个，任务堆在内存里，RSS 涨、进程一杀全丢。要身份、要重试、要取消，走任务队列（实战篇），别 fire-and-forget。

选型口诀：边缘 asyncio 收连接，CPU 丢进程池，同步 SDK 丢线程池，三套 `max_workers` 分开，队列有界。

再追：**「我们 QPS 不高，全用线程不行吗？」** 可以，直到某次下游 30s 超时占满 32 个线程，健康检查也排队。asyncio 的价值不是更快，是一万个等待只占一条线程 + 协程帧。QPS 低但连接数高（websocket、长轮询）更该 asyncio。QPS 低且全是 CPU，线程和 asyncio 都救不了，上进程。

---

### 12、协程、`yield`、`async` / `await` 是什么关系？

能说出口：历史上协程就是能 `yield` 出去再进来的生成器（`@asyncio.coroutine` + `yield from`）。3.5+ 的 `async def` / `await` 是单独的对象：coroutine object，不是生成器，不能 `next()`。两者都是**协作式**：只在 `await` / `yield` 点让出，纯计算中间无人能抢。

```python
async def add(a, b):
    return a + b

c = add(1, 2)               # 不执行，返回 coroutine
# c.send(None) 能把原生协程跑起来，但生产用 asyncio.run / create_task
```

往下追：**`await` 在等什么？** 一个 awaitable：实现 `__await__` 的对象，或原生 coroutine。`await asyncio.sleep(1)` 把当前 Task 挂起，loop 去跑别的；到期再唤醒。没有 await 的 `async def` 就是同步函数套了个皮，还占 loop。

**Task vs coroutine。** `async def` 返回的对象不进调度，直到 `create_task` / `gather` / `run`。扔在地上会 `RuntimeWarning: coroutine was never awaited`。

**取消。** `Task.cancel()` 在下一个 await 点扔 `CancelledError`。`to_thread` 里的阻塞调用取消不了线程。Go `context` 能传到 `http.Request`；Python 同步 HTTP 默认没有这条管道。

C++ 20 coroutine 是编译器拆帧，执行器自己选。Go 的 goroutine 有抢占（1.14+），不是协作。从 Go 迁过来最容易写一个没有 await 的热循环，把 loop 卡死。

再追：**生成器协程还能用吗？** `types.coroutine` / `yield from` 老路径还在，新代码不要写。`async def` 里 `yield` 是语法错误——那会变成异步生成器（`async for`），另一套协议。面试把「生成器 / 原生协程 / 异步生成器」三词分清：前两个都能暂停；第三个是 `async def` + `yield`，给 `async for` 用。

---

### 13、垃圾回收怎么做？环怎么破？

能说出口：CPython 两套。热路径**引用计数**：绑名字 +1，解绑 -1，到 0 立刻 `tp_dealloc`。环上互相 +1，计数到不了 0，靠**分代 GC**（容器对象进 generational 链表，周期性扫不可达环，`tp_clear` 砍边）。

```python
a = []; b = []
a.append(b); b.append(a)
del a, b                    # 环还在，等 gc
import gc; gc.collect()     # 破环
```

往下追：**哪些对象进 GC 跟踪？** 可能参与环的容器（list/dict/用户 class）。`int` / `str` / `bytes` 不参与循环检测。`gc.get_objects()` 不是「所有堆对象」。

**`__del__` 和环。** 带终结器的环，GC 不敢随便收（3.4 之后大多数能收，但仍可能进 `gc.garbage`）。资源不要靠 `__del__`，用 `with` / `weakref.finalize`。

**弱引用。** `weakref.ref(obj)` 不增加计数，对象死了变 `None`。缓存、观察者、打断环用它。

C++ `shared_ptr` 环要 `weak_ptr`；`unique_ptr` 离开关闭作用域立刻释放。Go 没有引用计数，三色标记，环不是问题，暂停模型是。CPython 热路径立刻收、环延迟收，RSS 不回给 OS 是分配器 arena 的事，不是 GC 没跑。

再追：**`sys.getrefcount` 为什么偏高？** 参数本身就是一次临时引用。只能看相对变化。细节在 GC 篇。

---

### 14、`__new__` 和 `__init__` 谁先谁后？

能说出口：`Cls()` 先 `__new__(cls, ...)` **造**实例，返回的是 `cls` 的实例才调 `__init__(self, ...)` **初始化**。`__new__` 是静态的（第一个参数是类），`__init__` 必须返回 `None`。日常只写 `__init__`。

往下追：**什么时候必须写 `__new__`？**

1. 不可变内置的子类：`int` / `str` / `tuple` 造出来就不能改，值要在 `__new__` 里传给 `super().__new__`。
2. 单例 /  intern / 池化：截在分配之前。注意 `__init__` 每次 `Cls()` 仍会跑，初始化要做「只一次」的标志。
3. `__new__` 返回别的类型：跳过 `__init__`。别拿这个当工厂，写 `@classmethod`。

```python
class Upperstr(str):
    def __new__(cls, s: str):
        return super().__new__(cls, s.upper())

print(Upperstr("ab"))       # AB
```

C++ 构造函数是「分配 + 初始化」绑在一起（placement new 才拆开）。Go 没有构造，`NewT()` 是普通函数。Python 把分配和初始化拆成两个钩子，面试要能说出「单例为什么不能只写 `__init__`」：`__init__` 拦不住分配。

---

### 15、MRO 是什么，`super()` 找的是父类吗？

能说出口：MRO 是方法解析顺序，单继承就是自己往上走到 `object`。多继承用 **C3 线性化**。`super()` **不是「父类」**，是「当前类在**该实例 MRO** 上的下一个」。

```python
class A:
    def ping(self): return "A"
class B(A):
    def ping(self): return "B" + super().ping()
class C(A):
    def ping(self): return "C" + super().ping()
class D(B, C):
    def ping(self): return "D" + super().ping()

print(D.__mro__)            # D, B, C, A, object
print(D().ping())           # DBCA
```

`B` 里的 `super()` 下一个是 `C` 不是 `A`。菱形继承靠这个让每一层都跑到。合并不了 C3 会在 class 语句时报 `TypeError`。

往下追：**Mixin 怎么放？** Mixin 在左、业务基类在右，MRO 才是 Mixin → Job。协作式 `super()`：每层都调 `super()`，别写死父类名。Go 没有继承，embedding 是组合。C++ 多重继承靠 vtable 和虚基类调 this，没有一张线性 MRO。

`super(cls, obj)` 返回代理，属性查找从 `cls` 在 `obj.__mro__` 里的下一个开始。3.x 无参 `super()` 靠编译器填 cell。

---

### 16、描述符是什么，`property` 怎么工作？

能说出口：描述符是类上定义了 `__get__` / `__set__` / `__delete__` 的对象。属性查找会走它。`property`、函数（绑 `self`）、`classmethod`、`staticmethod`、`__slots__` 成员，全是描述符。

- 有 `__set__` 或 `__delete__` → **数据描述符**，优先于实例 `__dict__`
- 只有 `__get__` → **非数据描述符**（函数），实例 dict 能盖住

```python
class C:
    def getx(self): return self._x
    def setx(self, v): self._x = v
    x = property(getx, setx)

c = C(); c.x = 1
c.__dict__["x"] = 99
print(c.x)                  # 1，property 是数据描述符，盖不住
```

往下追：**`obj.method` 为什么能带 self？** 函数是非数据描述符，`__get__(obj, type(obj))` 返回 bound method。`Class.method(obj)` 绕过绑定，自己传 self。`obj.put = 1` 之后 `obj.put` 不再是方法。

查找顺序（简化）：数据描述符 → 实例 dict → 非数据描述符 / 类属性 → `__getattr__`。

C++ 成员函数编译期定 `this`；Go `obj.Method()` 是 `T.Method(obj)` 的糖。Python 把绑定做成运行时协议。自己写描述符记住 `__set_name__`（3.6+）拿字段名，数据存在实例 dict，别存在描述符对象上（那是所有实例共享）。

---

### 17、`with` 干什么，上下文管理器协议是什么？

能说出口：`with expr as x` 调 `cm = expr; x = cm.__enter__()`，离开块（正常或异常）调 `cm.__exit__(exc_type, exc, tb)`。`__exit__` 返回真值会**吞掉**异常。文件、锁、租约、事务用它，不要靠 GC 关资源。

```python
class Lock:
    def __enter__(self):
        self.acquire(); return self
    def __exit__(self, et, e, tb):
        self.release()
        return False            # 不吞异常
```

往下追：**`__enter__` 失败会调 `__exit__` 吗？** 不会。申请到一半要自己清理，或拆成两个 CM，`ExitStack` 只对已经 enter 成功的做 exit。

**`@contextmanager`。** `yield` 前是 enter，后是 exit；`yield` 那边抛的异常在生成器里 `try` 住。`yield` 两次会 `RuntimeError`。

**没有 `defer`。** Go 的 `defer f.Close()` 在函数返回时跑；Python `f = open(...)` 指望函数结束时关，**不会关**。`return` 在 `with` 里，`__exit__` 照样跑，这点和 C++ RAII 一样。

再追：**生成器里 `yield` 在 `with` 中。** 退出推迟到生成器 close。协议的价值是把释放从「没人引用了」提前到「这块词法区域结束了」。

---

### 18、`*args`、`**kwargs`，仅关键字参数怎么写？

能说出口：`*args` 收多出来的位置参数成 tuple，`**kwargs` 收多出来的关键字成 dict。定义侧 `*` 后面的是仅关键字，`/` 前面的是仅位置（3.8+）。

```python
def connect(host, port=5432, /, *, timeout=5, **opts):
    ...

connect("127.0.0.1", 5432, timeout=2)
# connect("127.0.0.1", timeout=2, port=1)   # port 仅位置，不能关键字传
# connect("127.0.0.1", 5432, 2)             # timeout 仅关键字
```

往下追：**解包。** 调用侧 `f(*xs, **d)`。`xs` 要可迭代，`d` 的键要是 str。`def f(*args, **kwargs): return g(*args, **kwargs)` 是透明转发；漏 `wraps` 见第 8 题。

**可变默认值又来了。** 不要 `def f(opts={})`。`**kwargs` 每次调用是新 dict，函数如果 `kwargs["x"] = 1` 只脏这一次。

C++ 有默认参数、变参模板、C 风格 `...`，没有关键字参数（除非 designated initializer 那套）。Go 没有默认参数、没有关键字，常用 option func：`func WithTimeout(d Duration) Option`。Python 仅关键字是为了强制 `timeout=5` 这种自文档调用，API 设计里超时、flag 放 `*` 后面。

再追：**`*` 单独出现。** `def f(a, *, b)` 没有 `*args`，只是切开位置和关键字。`b` 必须用关键字传。

---

### 19、内存泄漏怎么查？

能说出口：Python 的「泄漏」多半是**根还握着**：全局 cache、无界 `lru_cache`、list 当日志缓冲、循环引用加 `__del__`、traceback 抓住 frame、C 扩展没 `DECREF`。RSS 不降还有 pymalloc arena 不还给 OS——对象已经死，页还在进程里。

往下追：**步骤。**

1. 先问是对象数涨还是 RSS 涨。`gc.get_objects()` / `len(cache)` 对对象；`/proc/self/status` 的 `VmRSS` 或 `resource.ru_maxrss` 对进程。
2. `tracemalloc.start(); ...; snapshot = tracemalloc.take_snapshot()`，按文件/行看分配。过滤自己的包。
3. 环：`gc.collect(); gc.set_debug(gc.DEBUG_SAVEALL)`，看 `gc.garbage`（带 `__del__` 的环）。
4. 全局和模块级 list / dict，`functools.lru_cache` 没 `maxsize`，信号量 / 回调没摘。
5. 闭包、生成器、`asyncio.Task` 没 await 也没 cancel，frame 钉住大对象。

```python
import tracemalloc
from functools import lru_cache

@lru_cache(maxsize=None)        # 无界，key 不重复就永远涨
def load(k: str) -> bytes:
    return k.encode() * 1000
```

C++ 泄漏是 new 无 delete / shared_ptr 环，用 asan。Go 泄漏是 goroutine 卡住 + 切片引用大数组底层，用 pprof。Python 先找**还活着的根**，再谈 GC 阈值。调 `gc.set_threshold` 救不了有根的 cache。

---

### 20、什么场景不该用 Python？和 C++ / Go 比。

能说出口：Python 适合 I/O 为主的胶水、数据/ML 生态、脚本、交付速度优先的后端。不该用的地方：

- **硬实时 / 微秒级尾延迟**：GIL、GC 暂停、对象头开销，没有 C++ 那种可控。
- **CPU 密集热路径还坚持纯 Python**：循环、解析、加密自己实现。用进程、或下沉 Cython/Rust/C，或换 Go/C++。
- **要在一个进程里把多核 CPU 吃满，又不想上 free-threaded / 多进程**：默认 CPython 做不到。Go 开箱 GOMAXPROCS；C++ 开线程。
- **内存极度敏感的连接数**：每个对象十几个字节头，百万小对象 RSS 难看。C++ 自己布局；Go 也有头，但值类型切片更紧。
- **启动延迟敏感的 CLI / serverless 热路径**：解释器 + import 链比 Go 静态二进制慢一个数量级。
- **需要语言级的值语义和编译期约束**：所有权、const、泛型单态化。Python 检查器是可选的。

往下追：**该用的时候别装腔。** 网络服务、管理面、编排、测试、把 C++ 引擎包一层 HTTP，Python 的人效是真的。瓶颈在下游数据库和 RPC，不在解释器。热点用 profiler（`py-spy` / `cProfile`）再决定要不要下沉，不要一上来用 C++ 重写整个服务。

对照：CPU 引擎 C++，并发网络服务 Go 很顺，业务编排和数据管道 Python 很顺。混部是常态，不是失败。

---

### 21、模块导入怎么走，循环导入怎么破，`__name__` 是什么？

能说出口：`import m` 把 `m.py`（或包）执行一遍，模块对象放进 `sys.modules`，再把名字绑到当前命名空间。已经在 `sys.modules` 里就直接拿，所以模块是单例。`from m import x` 在 `m` 里找名字 `x` 绑到当前；`m.x` 后来改了，你这个绑定还指旧对象（名字绑定，第 1 题）。

`__name__`：被直接执行时是 `"__main__"`，被 import 时是模块名。`if __name__ == "__main__":` 挡的是「import 时别跑入口」。spawn 多进程要求入口保护，否则子进程再 import 再 spawn。

往下追：**循环导入。** A 顶上 import B，B 顶上 import A，谁先执行谁还没把名字绑完，另一边 `from A import f` 会 `ImportError` 或拿到半成品模块。

破法：

1. 共享类型拆到 `types.py`，两边都 import 它，不要互相 import。
2. 延迟到函数里再 `import`（用的时候模块已经初始化完）。
3. 注解引起的循环：`TYPE_CHECKING` + 字符串注解，运行时不 import。

```python
# a.py
from typing import TYPE_CHECKING
if TYPE_CHECKING:
    from b import B
def make() -> "B":
    from b import B
    return B()
```

C++ 头文件循环用前向声明 + 指针。Go **禁止** import cycle，编译失败，必须拆包。Python 运行时有时能混过去，类型检查和 `from x import y` 最容易炸。

再追：**`import *`。** 只该出现在 REPL。它读 `__all__`，污染命名空间，静态分析困难。包的 `__init__.py` 里谨慎 re-export。

---

### 22、类型注解运行时有没有用？

能说出口：默认**没有**。注解是函数对象上的 `__annotations__` dict，调用时解释器不看。`def f(x: int) -> str: return 1` 运行时成功。有用的地方：pyright / mypy 在 CR 拦；运行时库主动读——pydantic、FastAPI、beartype、`dataclasses` 用注解生成 `__init__` 字段，**仍然不校验类型**（dataclass），pydantic 才校验。

往下追：**`from __future__ import annotations`。** 注解变成字符串，定义时不求值，前向引用、循环导入更顺；运行时真读注解的库要 `get_type_hints`。`cast` 运行时是恒等，只骗检查器。`Protocol` 运行时默认不是 `isinstance` 目标，除非 `@runtime_checkable`，而且它不查签名。

落地顺序：先边界 schema（API JSON），再核心领域，别全仓库 `Any`。细节在类型注解篇。面试答「注解只是提示」不够，要补「谁在读它」：检查器、pydantic、dataclass。

C++ / Go 的类型是语言强制。Python 把强制做成可选工具链。没有 CI 里的 pyright，注解就是注释。

再追：**`list[int]` 运行时能 `isinstance(xs, list[int])` 吗？** 3.9+ `list[int]` 是泛型别名，`isinstance` 对下标后的内置泛型会 `TypeError`（3.9–3.10）或只认 origin。运行时检查用 pydantic / 自己看元素，不要对 `list[int]` 做 isinstance。`get_origin` / `get_args` 是给写检查器插件的人用的。

---

### 23、metaclass 是什么，什么时候才需要碰？

能说出口：类是 `type` 的实例。`class Foo:` 大致是 `Foo = type("Foo", bases, namespace)`。metaclass 是 `type` 的子类，插进 `class Foo(metaclass=M):`，用来改「类怎么被造出来」：改 namespace、登记子类、强制接口。

```python
class Registry(type):
    def __new__(mcls, name, bases, ns):
        cls = super().__new__(mcls, name, bases, ns)
        if name != "Plugin":
            Plugin.registry[name] = cls
        return cls

class Plugin(metaclass=Registry):
    registry: dict[str, type] = {}

class Ping(Plugin):
    pass

print(Plugin.registry)      # {'Ping': <class Ping>}
```

往下追：**多数时候不需要。** 子类注册用 `__init_subclass__`（3.6+）更轻。抽象方法用 ABC。ORM / ORM-like、声明式 schema、要在 class 语句结束时改字段的框架才会写 metaclass。自己业务代码写 metaclass，CR 应当问「`__init_subclass__` 为什么不够」。

**坑。** 多重继承两个不同 metaclass 且没有共同子类，class 语句 `TypeError: metaclass conflict`。调试 MRO 已经够烦，再叠 metaclass 冲突是灾难。

C++ 没有「类的类」；模板元编程发生在编译期。Go 没有类。Python 的类是一等对象，metaclass 是这件事实的钩子，不是日常 OOP 工具。

---

### 24、pickle 能干什么、不能干什么？

能说出口：pickle 把对象图序列化成字节，`multiprocessing`、部分缓存、Celery 默认都用它。协议按版本，默认协议跟解释器走。能 pickle 的：多数内置容器、在模块顶层能找到的 class / 函数（按 **全限定名** 引用，不把字节码塞进去）。

往下追：**不能 / 不该。**

- lambda、生成器、模块内嵌函数、动态 `type()` 出来没登记的类
- 打开的文件、socket、锁、线程
- 反序列化等于执行：恶意 pickle 能跑任意 `__reduce__`。**不要 pickle 不信任的数据。** 对外协议用 JSON / protobuf
- 跨版本、跨语言不稳。这不是 JSON

```python
import pickle

def dump(obj, path: str) -> None:
    with open(path, "wb") as f:
        pickle.dump(obj, f, protocol=pickle.HIGHEST_PROTOCOL)

# pickle.dumps(lambda x: x)     # PicklingError
```

**`__reduce__` / `__getstate__`。** 自定义 class 控制怎么拆。返回 `(callable, args)`，unpickle 时调 callable。这是攻击面。

C++ 没有标准 pickle，序列化自己写。Go 的 `gob` 是 Go 到 Go，也不要当对外协议。进程池传参走 pickle，所以 Worker 的函数必须是模块级 `def`，参数必须可 pickle——实战篇用 SQLite 存 bytes，避开这条。

---

### 25、GIL 下哪些操作算「原子」，哪些不是？

能说出口：原子的是**一条 bytecode / 一次 C 调用期间 GIL 不放**，不是一条 Python 语句。经验上（官方有 GIL 的构建）：`list.append` / `pop()`、`d[k]=v`、`d.pop`、`set.add` 看起来不会看到内部撕裂。`x += 1`、`d[k] += 1`、`if k not in d: d[k] = ...` 都是读-改-写，中间能切线程，丢更新。

```python
# 原子（实现细节，不是语言保证）
lst.append(1)
d[k] = v

# 不原子
x += 1
d[k] += 1
if k not in d:
    d[k] = []
d[k].append(1)              # append 本身像原子，check-then-act 不是
```

往下追：**「像原子」只保证看不到 list 指针撕裂，不保证业务不变量。** 两个线程 `if k not in d: d[k]=[]` 仍能双写。free-threaded、PyPy、给 list 加了钩子，都不一定还成立。共享可变状态用 `Lock` 或把状态收进单线程 asyncio / 一个进程。

面试别背「GIL 所以线程安全」。正确三句：GIL 保护解释器；单次 C 调用碰巧像原子；check-then-act 加锁。并发篇有完整表。

C++ 原子是 `std::atomic`，语句从来不是原子。Go 的 map 并发读写直接 panic，比 Python 诚实。

---

### 26、dict 从哪一版开始有序？和 `OrderedDict` 还要不要？

能说出口：**3.7 起语言保证**插入序：先插入的 key，迭代先出来。3.6 是 CPython 实现细节（紧凑 dict 的副产品），别在 3.6 代码里当保证。3.5 及更早无序，`{}` 迭代顺序和哈希种子有关。

往下追：**实现。** 紧凑 dict：哈希表存索引，条目存在稠密数组。删除是在条目上留 dummy，迭代跳过。`popitem()` 默认 LIFO（最后一个插入的）。`reversed(d)` 3.8+ 合法。

**`OrderedDict` 还要吗？** 要 `move_to_end`、按相等比较「顺序也相同」、或对 3.6- 承诺有序，才用。普通「JSON object 进出、配置、kwargs」用 dict。`OrderedDict` 更重，等值还比顺序。

```python
d = {}
d["a"] = 1; d["b"] = 2; d["a"] = 3
print(list(d))              # ['a', 'b']  更新值不改位置
print(list(d.keys()))
```

**关键字参数、类体、`**` 解包** 都吃这套顺序。`def f(a=1, b=2)` 的 `__defaults__` 按定义序；`{**a, **b}` 后面的 key 覆盖、新 key 接在后面。

C++ `std::unordered_map` 无序，`std::map` 按 key 排序不是插入序。Go 的 map 故意随机迭代序，防止依赖。Python 3.7 把插入序做成契约，测试断言 `list(d)` 在 3.7+ 稳定，写给 3.5 的库不行。

再追：**set 有序吗？** 语言没保证。不要用 set 当「去重且保序」，用 `dict.fromkeys(xs)` 或 3.7+ 的 dict 当有序集合。
