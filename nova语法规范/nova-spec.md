# Nova 语言规范（精简版）

> 本文件是 **Nova 的唯一权威规范**。
> 精简原则：只写结论，不写过程。每条都可直接用于实现或写 EBNF。
> 过程记录见 `01-语言设计.md`（仅供追溯，以本文件为准）。
> 状态：`✔` 已定 ｜ `✗` 明确不要 ｜ `?` 未定（见第 9 节）

---

## 1. 词法

### 1.1 字符集 `✔`

```
ASCII 可打印字符 + @ + $
```

### 1.2 标识符 `✔`

```
identifier = ( letter | "_" | "$" ) { letter | digit | "_" | "$" }
```

- 不得以数字开头
- 不得与关键字重名

### 1.3 注释 `✔`

- **`#` 行注释**，普通注释与 doxygen 文档注释统一用 `#`
- 依据：doxygen 支持 Python 风格

### 1.4 关键字 `✔`

```
and      array    async    await    bool     break    char     class
const    continue else     elseif   false    float    for      function
if       in       int      let      match    none     not      or
override private  protected public   return   self     static   string
super    true     type     void     while    xor      yield
```

**两个语法糖**

| 关键字 | 等价于 |
|---|---|
| `elseif` | `elif` |
| `type` | `typedef` |

### 1.5 字面量 `✔`

| 类型 | 字面量 |
|---|---|
| `bool` | `true` `false` |
| `char` | `'a'` |
| `int` | `123` |
| `float` | `1.5`（**IEEE754** 存储，格式见 §2.6） |
| `string` | `"a"` |
| `void` | **`none`** |
| `array` | `[1, 2, 3]` |

> **`void` 是类型，`none` 是该类型的值。** 二者不是两个类型。

### 1.6 运算符 `✔`

```
算术    +  -  *  /  %
关系    <  >  <=  >=  ==  <>          # 不等号只有 <>，无 !=
逻辑    and  or  xor  not             # 关键字形式，无符号形式
赋值    =                             # 仅普通等号
其他    ::  .  ()  []  正负号  &  type()   # type() 为类型转换
```

**明确没有** `++`　`--`　`+=` 等复合赋值　运算符重载

---

## 2. 类型

### 2.1 类型集合 `✔`

```
void   bool   char   int   float   string   array
```

- **强类型 + 有限自动推导**，占位符 `let`
- **`string` 是内置类型**
- **是 `array`，不是 `list`**；`array` 是关键字
- 类型分两大类：**值类型**（`void` `bool` `char` `int` `float`）与**类类型**（`class`，**`array` 归结为它**）

### 2.2 内存与 ABI `✔`

- **默认对齐**
- 关注**二进制兼容**：跨版本、跨架构
- 链接期引入，**必须携带符号表**
- 各类型的**字节数、对齐、可用运算**见 §2.6

### 2.3 传递语义 `✔`

| 类别 | 语义 |
|---|---|
| 简单类型 | 传值 |
| 复杂类型（数组等） | 传引用（参照 Python 模型） |
| 数组作为返回值 | **返回引用**（与 C++ 不同） |

### 2.4 引用 `✔`

引用是**别名**：「一个单元的不同名字」。

```nova
let &a = c;                          # 引用声明
function gcd(a: &int, b: int) -> int # 引用传参
```

**`const` 的位置有意义**，不是笔误：

| 写法 | 含义 |
|---|---|
| `const &array` | ? 待补 |
| `&string const` | ? 待补 |

> 两者**含义不同**，均合法。具体约束待课上展开。

### 2.5 拷贝 `?`

暂缓，实现编译器阶段再定。

### 2.6 类型细节：字节 / 对齐 / 运算 `✔`

| type | byte | alignment | operation |
|---|---|---|---|
| `bool` | 1 | 1 | 逻辑 |
| `char` | 1 | 1 | 整型运算 |
| `int` | 4 | 4 | 混合运算 |
| `float` | 8 | 8 | 混合运算；按 **IEEE754 浮点格式存储** |
| `void` | 0 | 1 | 无 |
| `string` | `?` | `?` | 不适用 |
| `class`（**`array` 归结为它**） | 视成员数量 | 最大成员的对齐 | 不适用 |

- `void` **不占存储**（0 字节），alignment 为 1
- `float` 的存储格式为 **IEEE754**
- 类类型的大小**视成员数量**，alignment 取**最大成员的对齐**
- 本表的 `operation` 列指**内置运算符**；类类型（`class`、`array`、`string`）**不参与内置运算**，能力由下面的**接口**提供

**类类型的接口**（结构化定义见同目录 **`nova-types.yaml`**）

| 类类型 | attributes | operations |
|---|---|---|
| `array` | `length`、`internal_storage` | `length() -> int`、`[index] -> reference` |
| `string` | `length`、`internal_storage` | `length() -> int`、`[index] -> reference`、`[n:m] -> &string` |

- `[index]` 取到的是**引用**（别名，§2.4）；`string` 的 `[n:m]` 切片取到的是 **`&string`**，**左闭右开**（§5.3）
- 未定项见 §9.2

---

## 3. 数组

### 3.1 类型语法 `✔`

```nova
let a : array[int | n, m]
```

- 数组语义 = **维度 + 长度 + 基类**
- **长度是数组定义的一部分**（与 C++ 不同）
- **不用** C++ 的 `int[n]` 写法
- `array` 关键字保留
- **`array` 归结为类类型（`class`）**，布局与大小规则见 §2.6

### 3.2 动态数组 `✗`

先不要。

---

## 4. 表达式

### 4.1 总原则 `✔`

**几乎一切都是表达式。**

| 规则 | 内容 |
|---|---|
| 块的值 | 块内**最后一条表达式**的值 |
| 赋值的值 | 赋值是表达式，其值为 **`none`** |
| 级联赋值 | **禁止**：`a = b = c` 非法 |
| 因此 | `a = b;` 的结果是 `none` |
| `if` 的值 | 所选中分支块的值 |
| 控制结构 | `if` / `match` 本身也是表达式 |

---

## 5. 语句

### 5.1 分支 `✔`

```
if cond { }
if c1 { } elseif c2 { } else { }
```

- **`{ }` 强制**，即使只有一条语句

### 5.2 `match` `✔`

```nova
match obj {
    pattern => { ... }
    else    => { ... }
}
```

| 项 | 规则 |
|---|---|
| `obj` | 单值、比较表达式、逻辑表达式 |
| `pattern` | bool 或单值 |
| `else` | 即 default 分支 |
| 无 `case` | 用 `=>` |

### 5.3 循环 `✔`

```nova
while cond { }
for x in range(n, m, j) { ... }
```

| 规则 | 内容 |
|---|---|
| 无 `do-while` | |
| **range** | Python-like `range(n, m, j)`，**用于迭代** |
| **切片 `[i:j]`** | **用于索引**，**左闭右开** |
| 无 `[n,m,j]` 字面量 | |

### 5.4 跳转 `✔`

```
return    break    continue
```

### 5.5 声明 `✔`

```nova
function gcd(a: int, b: int) -> int { ... }   # 类型后置，无头文件
let a: int = 10;
type MyInt = int;
```

---

## 6. 类 `✔`

```nova
class name {
    public    { ... }
    protected { ... }
    private   { ... }
}
```

- 组内可混放字段与函数
- 继承：`class tiger <- public felid`
- 重写：`override`
- 构造函数：**`__constructor__`**
- **构造函数不自动调 `super`，必须手动显式调用**：
  `super.__constructor__("tiger");`

---

## 7. 包与工具链 `✔`

| 工具 | 名称 |
|---|---|
| 编译器 | `nvc` |
| 链接器 | `nvld` |
| 包管理器 | `nvp` |

- **无头文件**
- **`uni`** —— 标准包：io、range、字符串处理 等
- **`math`** —— **独立包**，与 `uni` 平级
- `import math;` → `math::sqrt(...)`
- 包管理实现：下载 C++ 源码，**现场编译**

---

## 8. 编写 BNF 的覆盖清单 `✔`

须覆盖 8 项：字符集、运算符集、关键字、标识符、字面量、表达式、语句、类型系统。
其中表达式与语句各自都要覆盖：算术、赋值、控制语句（**4 分支 + 2 循环 + 3 跳转**）。

```
4 分支 = if / elseif / else / match
2 循环 = while / for-in-range
3 跳转 = return / break / continue
```

---

## 9. 未定项 `?`

### 9.1 阻塞规范定稿的

1. **`const &T` 与 `&T const` 各自的确切语义**
2. **`::` 与 `import` 的完整语法**（包名解析、`uni` 组织方式）
3. **`<-` 继承的完整规则**：继承处的 `public` 含义、是否多重继承
4. **元组赋值** `a, b = b, a;` 的语义
5. **`main` 签名与返回值语义**
6. **未初始化变量的默认值**

### 9.2 类型细节（§2.6）与类型接口（`nova-types.yaml`）引入的新未定项

1. **`none` 常量的存储方式** —— 课上明确「留个疑问」
2. **`string` 的 byte / alignment**（接口已定，见 `nova-types.yaml`）
3. **`string` 属于哪一类**（类类型，还是另设的特殊内置类型）
4. **`array` / `string` 的 `internal_storage`**：是否指针、几字节、可见性（`public` / `private`）
5. **`length` 属性与 `length()` 方法的关系**：两个东西，还是一个的两种写法
6. **类型里的长度与运行时 `length` 的关系**：`array[int | n]` 的 `n` 与 `length` 如何对应
7. **`array` 是否也有 `[n:m]` 切片**（本次只给了 `string` 的）
8. **`[index]` 越界的行为**（Nova 无异常处理，越界怎么办）
9. **类的布局细则**：是否有方法表指针（`override` 需要动态分派）、成员是否按声明顺序排布、继承时基类子对象的位置
10. **`array` 的大小怎么算**：属性是 `length` + `internal_storage`，但「视成员数量」里各成员的宽度未定（见第 4 条）；维度与长度是**类型信息**还是**隐藏成员**
11. **「混合运算」的确切含义**：`int` ↔ `float` 的隐式提升方向与结果类型
12. **`char` 的「整型运算」范围**：是否含与 `int` 混合、是否自动提升
13. **`bool` 的取值表示**：是否 1 字节存 0/1
14. **`void` 的 alignment = 1 的用途**：可否作数组基类 / 指针目标
15. **对齐是否成通例**：一律 `alignment = 自身 byte`（`void` 除外），还是每类型单独拍
16. **`void` / `none` 作为值时的运行时表示**（依赖第 1 条）

### 9.3 暂不阻塞的

1. `static` 成员语法（关键字存在但无用例）
2. `self` 何时必须写
3. `async` / `await` / `yield` 的线程模型
4. `$` 前缀形参是规则还是约定
5. `array` 能否省略长度参数
6. 自定义类型如何比较（运算符重载已排除）
7. `nvp` 现场编译 C++ 的 ABI 细节
8. 拷贝语义（已明确暂缓）
9. **`vector` 等容器类型、`reserve` 等容量操作** —— 明确**暂不写**（推迟，不是未定）

---

## 附：文件

- **本文件** = 权威规范
- `nova-types.yaml` = `array` / `string` 的 **attributes / operations** 接口（结构化）
- `01-语言设计.md` = 课堂过程记录（追溯用）
- `02-类型细节.md` = 类型字节 / 对齐 / 运算的表与待确认项
- `../nova/nova-bnf.md` = **阶段性仓库，过渡产物，勿全盘吸收**
- `../nova程序示例/` = 示例程序
