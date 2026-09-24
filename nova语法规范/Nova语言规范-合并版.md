# Nova 语言规范（合并版 · 第 2–4 章）

> **Nova Language Specification — Consolidated Edition (Clauses 2–4)**
>
> 版本：1.0（2026-09-24）｜ 性质：**三章合并稿**——由 `第二章/`（词法与语言基本模型）、`第三章/`（核心语言语义）、`第四章/`（类、模块与异步执行）按《跨章节语义协调》（S-01…S-30）去重合并而成。
> 本文件**不改写**既有章节文件；既有文件与本文件关系：本文件是单一文件的**可读/可分发版本**，章节文件保留推导过程与理由。二者若出现文字差异，以 `nova语法规范/nova-spec.md` 的结论为准（本文件与其一致）。
>
> **体例**（参照 ISO/IEC 14882 与 Rust Reference）：
> - 条款采用**稳定标签**（如 `[lex.charset]`），供跨引用；条款增删只改编号，标签不变。
> - 段号 `[n]`：子条款内的段编号，跨引用写作 `§3.5 [6]`。
> - **规范用词**：**必须** shall ｜ **不得** shall not ｜ **可以** may ｜ **应该** should ｜
>   **格式错误**（ill-formed，必须诊断并拒绝）｜ **未定义行为** UB（不作保证）｜
>   **实现定义**（实现选定但必须文档化）｜ **移植期错误**（编译期错误）。
> - **注**（注：…）与**例**（例：…）为**非规范**内容。
> - 代码块中的 `...` / `{ ... }` 表示**省略**（非 Nova 语法），除标注「格式错误」的反例外均为合法代码。
> - 数学内容用 LaTeX 排版；语法用 ISO EBNF，终结符加引号。

## 条款导航

| 条款 | 内容 | 主要来源 |
|---|---|---|
| 0 | 范围、符合性、术语、记法 | 合并新增（工业体例前置） |
| 1 | 词法结构 | 第二章 §2.4 |
| 2 | 基本概念（作用域/对象/求值顺序） | 第二章 §2.5、第三章 §3.1/§3.2 |
| 3 | 类型系统 | 第二章 §2.6、第三章 §3.3.4 |
| 4 | 表达式 | 第二章 §2.7、第三章 §3.3 |
| 5 | 语句与控制流 | 第二章 §2.8、第三章 §3.3 |
| 6 | 声明与初始化 | 第二章 §2.8.6/§2.9、第三章 §3.1 |
| 7 | 引用、参数与赋值 | 第三章 §3.2 |
| 8 | 函数（含重载） | 第二章 §2.8.7、第三章 §3.4.1/§3.4.2 |
| 9 | 数组与字符串 | 第二章 §2.6.4、第三章 §3.4.3/§3.4.4 |
| 10 | 类 | 第四章 §6 |
| 11 | 模块与包 | 第四章 §7 |
| 12 | 异步执行 | 第四章 §8 |
| 13 | 诊断：移植期错误与 trap | 第二章 §2.10 |
| 14 | 图灵完备性 | 第二章 §2.11 |
| 附录 A | 语法汇总（EBNF 合并） | 第二章 §2.12 + 第四章附录 G |
| 附录 B | 未定项与开放问题 | `nova-spec.md` §9.4 等 |
| 附录 C | 实现定义与未指定行为 | 汇总 |
| 附录 D | 工具链实现约束 | 第三章 §3.5、第四章 §7.7/§7.8 |

---

## 0 范围、符合性与记法（Scope, conformance, notation）

### 0.1 范围 `✔` [scope]

**[1]** 本文件规定 Nova 程序的词法结构、基本模型、类型系统、表达式与语句、函数、类、模块与异步执行语义。
**[2]** 本文件**不**规定：标准库（`uni` / `math`）的完整接口、垃圾回收算法、调度器实现、目标平台 ABI 细节——这些见附录 B 的开放项清单。
**[3]** Nova 是**强类型**、编译型语言；源码文件扩展名 `.nova`，UTF-8 编码。

### 0.2 符合性 `✔` [conformance]

**[1]** 合规实现**必须**：接受所有符合本文件的程序（资源限制除外）；对格式错误的程序给出诊断并拒绝翻译；为每个「实现定义」项选定确定行为并写入文档。
**[2]** 实现**可以**提供扩展，但扩展**不得**改变合法程序的可观察含义。
**[3]** 本文件的「例」与「注」均为非规范内容。

### 0.3 术语 `✔` [terms]

| 术语 | 定义 |
|---|---|
| 源文件 / 模块 | 一个 `.nova` 文件即一个模块，也是编译单元；模块名 = 去后缀文件名 |
| 包 | 一个目录；包作用域 = 包内所有模块导出顶层名字的并集 |
| token | 词法最小单位：标识符、关键字、字面量、运算符、界符 |
| 对象 / 值 / 变量 / 绑定 | 对象=有类型的存储区域；值=类型的数据实例；变量=具名绑定 |
| 左值 / 右值 | 左值指代可寻址对象，可作赋值左值；右值指代值 |
| trap | 程序立即终止、非零退出码、无栈展开与清理 |
| 移植期错误 | 编译期错误，实现必须诊断并拒绝 |

### 0.4 记法 `✔` [notation]

**[1]** EBNF：`{ X }` 重复零次或多次，`[ X ]` 可选，`( X \| Y )` 二选一。
**[2]** 「关键字」指 §1.4 表中的标识符；`T(expr)` 为显式类型转换调用形式（§3.6.2）。

---

## 1 词法结构（Lexical structure）`✔`

### 1.1 源字符集 `✔` [lex.charset]

**[1]** 源文件是**字节流**，按 UTF-8 解释。词法结构之外（字符串/字符字面量之外）仅允许：

```
source-char-set = printable-ascii | "@" | "$" | white-space
printable-ascii = ASCII 0x21–0x7E
white-space     = 0x20 | 0x09 | 0x0A | 0x0D
```

**[2]** 换行序列 `\n`、`\r\n`、`\r` 等价。
**[3]** 第一行以 `#!/` 开头则整体忽略（脚本行）。
**[4]** 字符串/字符字面量内允许全部 Unicode 字符（UTF-8 存储，§1.5.4/§1.5.5）。

### 1.2 翻译阶段 `✔` [lex.phases]

**[1]** 顺序：① 换行归一；② 注释替换为**一个空格**；③ token 化（**最长匹配**，不依赖上下文）；④ 语法/语义分析与代码生成。

**例**（注释不断开表达式）：

```nova
let a = 1 # 注释
+ 2;      # 合法：a == 3
```

**[2]** **最长匹配规定**：任一位置取能构成合法 token 的最长字符序列。

**例**：`a<>b` 切为 `a` `<>` `b`；`a=>b` **恒切**为 `a` `=>` `b`（`=>` 是界符；出现在 `match` 之外属语法错误，由语法分析诊断，词法层不得按上下文改切分）；`a::b` 切为 `a` `::` `b`。

### 1.3 注释 `✔` [lex.comment]

```
comment = "#" { any-char-except-newline } newline
```

**[1]** `#` 行注释，到行尾结束；普通注释与文档注释统一用 `#`（doxygen 按 Python 风格处理）。
**[2]** 注释不得出现在 token 内部（`i#c#d` 不构成 `id`），可出现在任意两个 token 之间。
**[3]** 无块注释。

### 1.4 标识符 `✔` [lex.name]

```
identifier = ( letter | "_" | "$" ) { letter | digit | "_" | "$" }
```

**[1]** 大小写敏感；不得以数字开头；不得与关键字同名。
**[2]** `$` 是普通标识符字符，**无特殊语义**（`$x` 形参是书写约定，非语言规则）。
**[3]** `_` 单独出现合法，惯例表示"故意忽略"。

### 1.5 关键字与字面量 `✔` [lex.key]

```
and      array    async    await    bool     break    char     class
const    continue else     elseif   false    float    for      function
if       import   in       int      let      match    none     not
or       override private  protected public   return   self     static
string   super    true     type     void     while    xor      yield
```

**[1]** 下列为关键字，不得作名字。`elseif` 是嵌套 `else { if }` 的语法糖；`type` 是类型别名引入符（相当于 `typedef`）；`import` 是模块导入关键字（§11）；`async` / `await` / `yield` 的语法与语义由 §12 定义。
**[2]** `elif` **不是**关键字，也不被接受。

**字面量**：

```
boolean-literal = "true" | "false"
int-literal     = dec-literal | hex-literal | bin-literal
dec-literal     = digit { digit }
hex-literal     = "0x" hex-digit { hex-digit }
bin-literal     = "0b" ( "0" | "1" ) { "0" | "1" }
float-literal   = digits "." [ digits ] [ exponent ]
                | digits exponent
                | "." digits [ exponent ]
exponent        = ( "e" | "E" ) [ "+" | "-" ] digits
char-literal    = "'" ( graphic-char | escape-seq ) "'"
string-literal  = '"' { graphic-char | escape-seq } '"'
none-literal    = "none"
array-literal   = "[" [ expression { "," expression } ] [ "," ] "]"
```

| 类型 | 字面量 | 规则 |
|---|---|---|
| `bool` | `true` / `false` | — |
| `char` | `'a'` | 值域 0–127 ASCII；转义 `\n \r \t \\ \' \0` |
| `int` | `123`、`0x10`、`0b101`、`1_000` | 无八进制、无后缀；超 `int` 范围为移植期错误 |
| `float` | `1.5`、`1.`、`.5`、`1e10` | IEEE 754 binary64；越界取 ±∞/±0（警告） |
| `string` | `"a"` | 字节序列，UTF-8；转义另加 `\"`；不相邻拼接 |
| `void` | `none` | 唯一值，无运行时表示（§3.5.4） |
| `array` | `[1, 2, 3]` | 元素须同型（提升后）；尾逗号限一个 |

### 1.6 运算符与界符 `✔` [lex.oper]

```
operator  = "+" | "-" | "*" | "/" | "%"
          | "<" | ">" | "<=" | ">=" | "==" | "<>"
          | "=" | "not" | "and" | "or" | "xor"
delimiter = "(" | ")" | "[" | "]" | "{" | "}"
          | ";" | "," | ":" | "::" | "." | "->" | "=>" | "<-" | "|" | "&"
```

**[1]** `&` 仅出现在类型与声明位置（引用，§7）；无取地址运算。`->` 返回类型箭头；`=>` match 分支箭头；`<-` 继承箭头；`|` 数组类型分隔符。
**[2]** 逻辑运算只有关键字形式。**无**位运算、`++`/`--`、复合赋值、运算符重载、`?:`、`!=`（不等只有 `<>`）。
**[3]** `type()` 是类型转换的**元记法**：落地为调用形式 `T(expr)`，`T ∈ { char, int, float }`（§3.6.2）；类类型不提供显式转换（§3.5.5）。

### 1.7 运算符优先级与结合性 `✔` [lex.prec]

从上到下递减；同级左结合（赋值右结合）：

| 优先级 | 运算符 | | 优先级 | 运算符 |
|---|---|---|---|---|
| 1 | `( ) [ ] .` 调用/下标/成员 | | 6 | `== <>` |
| 2 | 一元 `- + not` | | 7 | `and` |
| 3 | `* / %` | | 8 | `xor` |
| 4 | `+ -` | | 9 | `or` |
| 5 | `< > <= >=` | | 10 | `=` 与多目标赋值 |

**[1]** `a::b.c` 解析为 `(a::b).c`：`qualified_name` 是**原子项**（§4.2），"优先级最高"由构造保证，不列入本表。
**[2]** 赋值值为 `void`（§4.6），故 `a = b = c` 非法；`let x = (a = 1)` 非法。

---

## 2 基本概念（Fundamental concepts）`✔`

### 2.1 作用域与名字 `✔` [basic.scope]

**[1]** 词法作用域，四级：包（顶层 `function`/`class`/`type`/`const`）、文件（`import` 引入名）、函数（形参）、块（`let`、循环变量）。查找取**最近可见绑定**。

**[2]** 名字唯一性：
- **函数名可按重载共享**（签名 = 参数类型列表，§8.2）；
- 类名、`type` 别名、包级 `const` 名各自唯一，且**类名不得与任何函数名重名**；
- 同名 `let` 创建新绑定并遮蔽前者，已解析表达式仍指向原绑定。

**[3]** 遮蔽可嵌套；无逃逸语法（`::` 仅用于包/模块/类限定）。
**[4]** 块作用域变量**无前向引用**（无提升）；顶层函数、类、类型别名允许前向引用（两遍扫描）。
**[5]** 初始化器不得直接或间接读取正在初始化的变量（`let x = x;` 非法）。

### 2.2 对象、存储期与生命周期 `✔` [basic.life]

| 存储期 | 对象 | 结束 |
|---|---|---|
| 静态 | 字符串/数组字面量、包级 `const` | 程序终止 |
| 自动 | 块作用域 `let`、形参 | 块执行完毕 |
| 动态 | 类类型（含 `array`/`string` 底层存储） | 运行时回收 |

**[1]** 值类型对象存于栈/寄存器；类类型变量持堆对象引用。
**[2]** **包作用域无可变变量**：顶层只允许 `const`（§6.2），不存在包级 `let`——由此无模块初始化函数、无静态初始化顺序问题、循环 `import` 安全。
**[3]** 回收策略（引用计数或追踪式）属实现定义；规范保证**不存在可观察的悬垂引用**：`let &a = c;` 后 `c` 先结束生命周期，为移植期错误（静态可判定时）或 trap（否则）。

### 2.3 求值顺序 `✔` [basic.eval]

**[1]** 运算符操作数、函数实参、成员/下标访问**统一从左到右**。
**[2]** `and` / `or` 短路；`xor` 不短路。
**[3]** 多目标赋值：先求全部目标地址，再求全部右值，最后按序写入；重复地址保留最后一次写入。
**[4]** 语句按书写顺序求值。

### 2.4 值类别 `✔` [basic.lval]

**[1]** 左值：变量、字段访问、索引（`a[i]` 结果为元素引用）、引用别名。右值：字面量、运算结果、调用结果、块值。
**[2]** 只有左值可作赋值左值；`a + b = c` 是语法错误。
**[3]** 左值不自动拥有写权限（§7）。

---

## 3 类型系统（Type system: fundamental model）`✔`

### 3.1 类型分类 `✔` [type.cats]

```
type = value-type | reference-type | class-type | array-type
value-type     = "bool" | "char" | "int" | "float" | "void"
reference-type = "&" type
class-type     = class-name | "string"
array-type     = "array" "[" type "|" expression { "," expression } "]"
```

**[1]** 值类型：`bool char int float void`，赋值按位复制。
**[2]** 类类型：用户 `class`、内置 `string` 与 `array`（`array` 归结为类类型），变量持堆对象引用；派生→基类**隐式向上转换**合法，向下转换不提供。
**[3]** 引用类型 `&T` 是 `T` 的别名类型（§7），只出现在形参、`let &a` 与接口返回类型。
**[4]** **函数不是一等值**：不可作实参、不可绑定、不可存储；无 lambda/闭包。
**[5]** 类型集合封闭：无元组、无枚举、无指针。

### 3.2 大小与对齐 `✔` [type.size]

**对齐通例**：`align(T) = size(T)`（`void`/`class` 除外）；对象起始必须满足

$$
\operatorname{addr} \bmod \operatorname{align}(T) = 0
$$

| 类型 | size | align | 备注 |
|---|---|---|---|
| `bool` | 1 | 1 | 位模式仅 0/1 |
| `char` | 1 | 1 | ASCII 0–127 |
| `int` | 4 | 4 | 二补数 |
| `float` | 8 | 8 | IEEE 754 binary64 |
| `void` | 0 | 1 | 不占存储；不可作数组基类 |
| `string` | 16 | 8 | 句柄 `{ptr, length}` |
| `&T` | 8 | 8 | 机器字长 |
| `array[T \| n]` | $n \times size(T)$ + 16 句柄 | $align(T)$ | 行主序 |
| `class` | 视成员（含 vptr 8B） | $\max(8, \max_i align(m_i))$ | §10.6 |

**[1]** 成员按声明顺序排布，最小填充；实现**不得**重排（ABI 二进制兼容）。

### 3.3 值类型表示 `✔` [type.value]

**[1] `bool`**：1 字节，`0x00`/`0x01`；非法位模式不会出现于合法程序。
**[2] `char`**：1 字节 7 位 ASCII。**不直接参与算术与比较**——须先显式 `int(c)`；赋值/实参/重载排序不受限（§3.6.1）。
**[3] `int`**：二补数，$[-2^{31},\, 2^{31}-1]$。运算越界 → **trap**（`NOVA-R-INT-OVERFLOW`），不静默回绕（含 $-2^{31} \bmod -1$、一元负越界）；回绕语义由 `math` 包提供显式函数。溢出检查是语言保证。
**[4] `float`**：IEEE 754 binary64，支持 ±∞/±0/NaN，round-to-nearest-even；`-0.0 == 0.0` 为 `true`。
**[5] `void` 与 `none`**：`void` 是类型、`none` 是唯一值。`none` **无运行时表示**，仅作编译期占位：返回值类型 `-> void`、赋值表达式类型、块值 `void` 语境。`void` 表达式**不可**绑定 `let`、不可作实参/操作数，不可作索引。

### 3.4 传递语义 `✔` [type.pass]

| 类别 | 传参 | 赋值 | 返回 |
|---|---|---|---|
| 值类型 | 传值 | 按位复制 | 传值 |
| 类类型（`class`/`array`/`string`） | 复制句柄（共指同一对象） | 复制句柄（别名效果） | 返回句柄（引用） |
| `&T` 形参 | 绑定实参存储（无复制） | — | — |

**[1]** 类类型赋值 `b = a` 后二者是同一对象的两个名字（Python 式引用语义）。**无隐式拷贝**；独立副本须显式构造（§10.4 [14]，`nova-spec.md` §2.5）。
**[2]** 数组返回引用（句柄），生命周期由运行时保证（§2.2）。

### 3.5 类类型接口（`array` / `string`）`✔` [type.iface]

（结构化定义见 `nova-types.yaml`；以下为落定规则）

**[1] `string`**：句柄 16 字节，底层 UTF-8 字节数组。
- `length` 属性 ≡ `length()` 方法（同一访问两种拼写），返回字节数；
- `[index]` 返回**只读** `&char`（字符串不可变，越界 trap）；
- `[n:m]` 左闭右开切片，返回 `&string`（视图，不复制）；$0 \le n \le m \le \text{length}$ 否则 trap；
- `==` / `<>` 按**内容**比较（类类型中唯一例外）；
- `format(fmt, args...) -> void`：内置成员方法（享格式化 I/O 可变尾参豁免，§8.1），**一次性初始化写入**——接收者须为未初始化槽位或空串，写入后不可变（`point.nova` 依赖）。

**[2] `array`**：句柄 16 字节 + 行主序连续堆存储。
- `length()` 返回**全维元素总数**；
- `[index]` 返回元素引用（左值）；**逗号全维下标** `a[i, j]` 一次消费全部维度，**无部分下标、无行视图**；
- **不支持 `[n:m]`**；越界 trap；
- 长度是类型一部分：`array[int | 9]` 与 `array[int | 10]` 不同类型；形参可省长度（`&array` 顶类型），`let` 位置未定（附录 B）。
- 动态数组 ✗。

**[3] `class`**：对象模型摘要见 §10.6（vptr 置头、声明顺序、方法表）。

### 3.6 转换 `✔` [type.conv]

**[1] 隐式提升**（§3.6.1）：方向唯一 `char → int → float`。适用于：赋值、实参匹配、多目标赋值两侧、重载排序、**算术/比较中的 `int → float`**。
**[2]** 被禁止的仅是 `char` 的自动提升**：`char` 入算术/比较前必须显式 `int(c)`；`int` 与 `float` 混合运算合法，结果 `float`。`bool` 不参与任何隐式转换（无 `bool→int`、无 `int→bool`，判非零写 `x <> 0`）。
**[3]** 注意区分 §4.9：`if`/`match` 分支类型必须**完全同型**（类型统一规则），不受运算数提升管辖——两条规则独立。

**[4] 显式转换**：调用形式 `T(expr)`：

| 转换 | 语义 |
|---|---|
| `int(c)`（c: char） | 数值不变 |
| `char(i)` | 仅 $0 \le i \le 127$，越界 trap |
| `float(i)` | 精确；$\|i\|>2^{24}$ 时 IEEE 舍入（警告） |
| `int(f)` | 截断向零；NaN/∞/越界 trap |
| `char(f)` | 先截断为 `int`，再按 `char(i)` 检查 |

- `bool(x)` 不提供；`string ↔ int/float` 无内置转换（`uni` 包，§11.5）。
- 类类型：仅派生→基类**隐式**向上转换；**不提供显式 `type()` 形式**（与构造调用同形、文法无法消歧），类↔值类型、无关类之间无转换；引用与值类型无转换。

**[5] 运算类型规则**：二元算术/关系运算操作数限 `int`/`float`，混合取最窄公共类型：

$$
\frac{T_1, T_2 \in \{\mathsf{int}, \mathsf{float}\} \quad U = \max(T_1, T_2)}{T_1 \mathbin{\text{op}} T_2 : U}
$$

---

## 4 表达式（Expressions）`✔`

### 4.1 语法总览 `✔` [expr.gram]

```
expression      = assignment-expression
assignment-expression = conditional-expression [ "=" assignment-expression ]
                      | multiple-assignment
conditional-expression = logical-or-expression
logical-or-expression  = logical-xor-expression { "or" logical-xor-expression }
logical-xor-expression = logical-and-expression { "xor" logical-and-expression }
logical-and-expression = equality-expression { "and" equality-expression }
equality-expression    = relational-expression { ( "==" | "<>" ) relational-expression }
relational-expression  = additive-expression { ( "<" | ">" | "<=" | ">=" ) additive-expression }
additive-expression    = multiplicative-expression { ( "+" | "-" ) multiplicative-expression }
multiplicative-expression = unary-expression { ( "*" | "/" | "%" ) unary-expression }
unary-expression       = [ "-" | "+" | "not" ] postfix-expression
postfix-expression     = primary-expression { call-suffix | index-suffix | member-suffix }
call-suffix            = "(" [ expression { "," expression } ] ")"
index-suffix           = "[" expression { "," expression } "]"
member-suffix          = "." identifier
primary-expression     = literal | qualified_name | "(" expression ")"
                       | "self" | "super" | array-literal | block-expression
                       | if-expression | match-expression | await-expr
qualified_name         = identifier { "::" identifier }
multiple-assignment    = lhs "," { "," lhs } "=" expression { "," expression }
lhs                    = identifier | postfix-expression "." identifier
                       | postfix-expression "[" expression { "," expression } "]"
```

### 4.2 字面量、标识符与限定名 `✔` [expr.prim]

**[1]** 标识符解析到其绑定；未声明为移植期错误。**[2]** 限定名 `a::b::c` 为原子项，使 `math::sqrt(x)`、`counter::bump()`、`pkg::module::name` 可解析；其解析规则（左操作数种类、可见性）属 §11，解析失败为移植期错误。
**[3]** `await 表达式` 的语法与限制见 §12.3。

### 4.3 算术表达式 `✔` [expr.arith]

**[1]** 二元 `+ - * / %` 操作数限 `int`/`float`（§3.6 [5]）：`char` 须先 `int(c)`；混合时 `int` 升 `float`。
**[2]** `int / int` 向零截断；除零或商越界 trap。`float / float` 遵 IEEE（除零得 ±∞/NaN，不 trap）。
**[3]** `%` 仅 `int`，$a = (a/b)\cdot b + (a \% b)$，余数符号随被除数；除零 trap。
**[4]** 一元 `-`：`int` 越界 trap；`float` 翻符号；`char` 不可直接取负。一元 `+` 无操作。

### 4.4 关系与相等 `✔` [expr.rel]

| 运算 | 操作数 | 结果 |
|---|---|---|
| `< > <= >=` | `int`/`float`（混合提升；`char` 先 `int(c)`） | `bool` |
| `== <>` | 上述 + `bool` + `string`（内容） + 类类型（引用同一性） | `bool` |

**[1]** 浮点遵 IEEE：NaN `==` 得 `false`、`<>` 得 `true`、排序一律 `false`；±0 相等。
**[2]** `string` **不支持**排序比较（归 `uni` 函数）。`bool` 仅可 `==` `<>`。
**[3]** 类类型 `==` 比较引用同一性；内容相等用普通成员函数（如 `equals`，命名由库约定）。

### 4.5 块表达式与"一切皆表达式" `✔` [expr.block]

```
block-expression = "{" { statement } [ expression ] "}"
```

**[1]** 块以表达式结尾 → 块值为该表达式；以语句结尾（`let`、`return`、空块）→ 类型 `void`、值 `none`。
**[2]** **语句封闭清单**（无值、不可出现在表达式位置）：`let`/`const`/`type`/`import` 声明、函数/类声明、`return`/`break`/`continue`。其余构造皆为表达式（含赋值，类型 `void`）。

### 4.6 赋值表达式 `✔` [expr.assign]

**[1]** 左值须为 §2.4 的左值。**赋值表达式类型为 `void`、值为 `none`**（权威规范 §4.1）⇒ `a = b = c` 非法。`const` 对象、`string` 索引结果、字面量作左值为移植期错误。

### 4.7 多目标赋值 `✔` [expr.multi]

**[1]** 左右值数量必须相等；先取址、后取值、再按序写入；`a, b = b, a` 因此正确交换；类型 `void`；无元组类型参与。
**[2]** **文法区分**：目标列表按**顶层逗号**切分；`[ ]` 内的逗号归属索引后缀。故 `a[i, j] = v;` 是单个多维下标目标的单赋值，`a[i], b = x, y;` 才是多目标赋值。

### 4.8 `if` 与 `match` 表达式 `✔` [expr.ifmatch]

**[1]** 无 `else` 的 `if` 不可作表达式。
**[2]** 值语境各分支（含 `match` 的 `else`，强制存在）块值必须**完全同型、不自动提升**；混型写显式转换（`float(1)`）。被选中分支完整求值。

### 4.9 调用表达式 `✔` [expr.call]

**[1]** 实参按位置匹配；无默认参数、无关键字实参。
**[2]** **允许重载**（§8.2）：签名 = 参数类型列表，返回类型不参与；精确匹配优先，无精确匹配时按 `char → int → float` 提升排序（提升少者优先），仍并列则 `NOVA-E-OVERLOAD`。三种引用形状不产生不同签名（§7.1）。
**[3]** `&T` 形参要求实参为可写稳定左值；`const &T` 接受任意 `T` 左值；`&T const` 可接受字面量/临时值。
**[4]** 构造函数调用 `ClassName(args)` 合法并可重载（§10.4）；递归无人为深度限制（栈耗尽 trap）。
**[5]** 实参从左到右求值。

### 4.10 索引与成员访问 `✔` [expr.index]

**[1]** `a[i]`：`array` 返回 `&T`（左值）、`string` 返回只读 `&char`；越界 trap。多维 `a[i, j]` 见 §9.1 [2]。
**[2]** `s[n:m]` 仅 `string`，左闭右开视图（§3.5 [1]）。
**[3]** `obj.field` / `obj.method(...)`：成员访问，`self`/`super` 规则见 §10.3。
**[4]** 包限定 `pkg::name` 见 §11。

---

## 5 语句与控制流（Statements）`✔`

### 5.1 总形式 `✔` [stmt.gen]

```
statement = expression-statement | declaration | if-statement | match-statement
          | while-statement | for-statement | jump-statement | block | ";"
```

**[1]** 表达式语句求值并丢弃值；`void` 表达式**只能**以语句形式出现。块本身是表达式，`if c { }` 无需分号。

### 5.2 分支 `✔` [stmt.if]

```
if-statement = "if" expression block
             | "if" expression block { "elseif" expression block } [ "else" block ]
```

**[1]** 条件必须为 `bool`，**无真值转换**（`if x`（x: int）非法，写 `if x <> 0`）。
**[2]** `{ }` 强制，即使只有一条语句。
**[3]** `elseif` 语义等价右嵌套 `else { if }`，但分支块值直接成为候选（避免"内层块以 `let` 结尾得 void"）。

### 5.3 `match` `✔` [stmt.match]

```
match-statement = "match" expression "{" { match-arm } [ else-arm ] "}"
match-arm       = pattern "=>" block
else-arm        = "else" "=>" block
pattern         = expression
```

**[1]** 命中判定由**模式形态**决定：
- **比较模式**（含 `< > <= >= == <>`）：求值为 `bool`，为 `true` 即命中（模式内可引用 obj 的变量，`match d { d > 0 => {...} }`）；
- **值模式**（字面量等无比较运算）：求值后与 obj 做 `==`。
- 两类模式**不可混用**；分支自上而下、首命中即执行。
**[2]** `else` 即 default；语句语境可省，表达式语境强制。
**[3]** 无 `case`、无解构（无枚举、无元组）。

### 5.4 循环与跳转 `✔` [stmt.loop]

```
while-statement = "while" expression block
for-statement   = "for" identifier "in" iterable block
iterable        = "range" "(" expression { "," expression } ")" | array-expr | string-expr
jump-statement  = "return" [ expression ] ";" | "break" ";" | "continue" ";"
```

**[1]** `while` 条件须 `bool`，先判后执；无 `do-while`。
**[2]** `range`：`range(n)` ≡ `range(0,n)`，`range(n,m)` ≡ `range(n,m,1)`；迭代 $v_k = n + k\cdot s$，步长为正取 $v_k < m$、为负取 $v_k > m$（左闭右开）；**步长为零 → trap**（`NOVA-R-ZERO-STEP`；常量可证明时编译期报同一错误码）。`range` 是**语言内建构造**（非一等值，编译为计数循环）。
**[3]** 可迭代对象：`range` 调用、`array`、`string`、字符串切片；循环变量是**只读副本**（值类型复制值、类类型复制句柄），赋值不改集合，写集合用 `a[i]`。
**[4]** `break`/`continue` 只作用于最内层**循环**，不跳出 `match`；必须在循环体内（否则移植期错误）。
**[5]** 循环是语句，无值。无 `++`/`--`/复合赋值。
**[6]** 无 `do-while`。

---

## 6 声明与初始化（Declarations and initialization）`✔`

### 6.1 声明形式 `✔` [decl.gen]

```
declaration = "let" [ "&" ] identifier [ ":" type ] [ "=" expression ] ";"
            | "const" identifier ":" type "=" expression ";"
            | "type" identifier "=" type ";"
            | function-declaration | class-declaration
```

**[1]** `let` 必须有类型标注或初始化器；`let x;` 非法。
**[2]** `let x = e`：从 `e` 完全推导（含函数返回值、块值、数组字面量）；`let x: T = e`：允许 §3.6 隐式方向。推导不跨语句/函数。
**[3]** `const`：绑定只读、不要求编译期常量；**包级 `const` 初值必须是编译期常量表达式**（字面量/常量名/其运算与显式转换；不得含调用、构造、`await`/`yield`），链接前完成求值。
**[4]** `type A = B`：纯类型别名，与 `B` 的类型、表示、对齐、ABI 等价。
**[5]** **模块顶层只允许** `import`、类、函数、`type`、`const`（及 `private` 修饰），**无可变 `let`**（§2.2 [2]）。

### 6.2 确定性初始化 `✔` [decl.definite]

**[1]** `let x: T;` 创建**未初始化槽位**（无默认值）；读取前每条可达路径必须已写入。
**[2]** 路径敏感数据流分析：`if` 全分支写入才算已初始化；`while`/`for` 可能零次，体内写入不证明循环后已初始化；`return`/`break`/`continue` 终止的路径不再参与。
**[3]** 写入的认定：赋值、`input(x)`（`uni` 输出参数原语）、`string.format`（§3.5 [1]）；普通调用不默认算写入。形参进入时已初始化。
**[4]** 读取前无法证明已初始化 → 移植期错误（`NOVA-E-USE-BEFORE-INIT`）。
**[5]** 初始化器不得读取正在初始化的变量（直接或间接）。

### 6.3 类字段初始化 `✔` [decl.field]

（摘要；完整规则 §10.3）
**[1]** 值类型字段无初值 → 零值（`false`/`'\0'`/`0`/`0.0`）。
**[2]** 类类型字段（含 `string`/`array`）**必须**有声明初值，或在本类每个构造函数的所有正常退出路径上赋值（C# definite assignment 式编译期检查）；隐式无参构造仅当全部类类型字段有初值时可用。不存在空引用。

### 6.4 局部变量与字段对照 `✔` [decl.table]

| 语境 | 无初始化器时 |
|---|---|
| 局部 `let x: T;` | 未初始化槽位，读取前须过确定性初始化检查（§6.2） |
| 类字段 | 值类型清零；类类型须构造保证（§6.3） |
| 包级 | 只允许 `const` + 编译期常量初值（§6.1 [3]） |

---

## 7 引用、参数与赋值（References, parameters, assignment）`✔`

### 7.1 引用与 `const` 三形状 `✔` [ref.const]

```
reference-decl = "let" "&" identifier "=" expression
reference-type  = [ "const" ] "&" type | "&" type [ "const" ]
```

引用是**别名**（「一个单元的不同名字」）。

| 写法 | 含义 |
|---|---|
| `&T` | 可写引用：只能绑定**稳定且可写的左值**；不重绑定 |
| `const &T` | **被引用对象只读**：只能绑定稳定左值，经其不可写，不可传给 `&T` 形参 |
| `&T const` | **调用期间的只读参数视图**：可绑定稳定左值、字面量或临时值；临时值不得逃出调用 |

**[1]** 三种形状**不产生不同重载签名**，只影响绑定检查、写权限与生命周期。
**[2]** `&T` 可隐式转 `const &T`，反向非法。裸 `const T`：绑定只读（类类型上等价 `const &T`）。
**[3]** 引用普通形式可绑定：变量/形参、可访问类成员、数组元素、索引结果、生命周期受保证的引用返回值；不能绑字面量/临时值（`&T const` 参数是唯一例外）。
**[4]** 返回局部变量/临时值的引用必须编译期拒绝。

### 7.2 赋值目标 `✔` [ref.assign]

**[1]** 仅可写稳定左值：变量、可写成员、数组元素、经引用别名写被引用对象。
**[2]** `const` 绑定、`const &T` 视图、只读成员拒绝写入。赋值表达式类型 `void`、值 `none`；级联赋值非法。
**[3]** 多目标赋值规则见 §4.7（数量相等、先取址后取值再写入、重复地址保留末次写入）。

### 7.3 `for` 循环变量 `✔` [ref.forvar]

**[1]** 循环变量每轮取得元素的**值**（只读副本）：基础类型复制值，类/数组/字符串复制句柄。
**[2]** 对循环变量赋值不修改集合；修改集合用 `a[i]` 或显式引用。

---

## 8 函数（Functions）`✔`

### 8.1 声明与返回 `✔` [func.decl]

```
function-declaration = "function" identifier "(" [ parameter-list ] ")" [ "->" type ] block
parameter      = identifier ":" type
async-function-declaration = "async" function-declaration
```

**[1]** 形参类型后置；无头文件。普通函数参数固定；**标准库格式化 I/O**（`print`/`println`/`string.format`）可接受异类型可变尾参数。
**[2]** 返回类型省略时从所有带值 `return` 推导：全部同型 → 该型；无带值返回 → `void`。
**[3]** `return e`：`e` 的类型与返回类型相同或沿提升链**加宽**到返回类型（`int` 可返回给 `float`；收窄须显式转换）。
**[4]** `void` 函数允许 `return;`/`return none;`，落尾隐式返回。**非 `void` 函数每条可达路径必须 `return`**，否则 `NOVA-E-RETURN`。
**[5]** `main`：合法入口仅 `function main() -> int` 与 `async function main() -> int`；不接受形参（命令行参数经 `uni::args()`，附录 B）；返回值即进程退出码（Nova 层完整 `int`，系统边界取 `mod 256`）；库文件可无 `main`，可执行程序有且仅有一个。

### 8.2 重载与继承查找 `✔` [func.overload]

**[1]** 普通函数与成员函数均可重载；签名 = 参数类型列表；返回类型不参与；仅返回类型不同的同名同参函数为格式错误。
**[2]** 调用解析：① 按名字与实参数量筛选；② 全部参数**精确匹配**优先；③ 无精确匹配时按 `char → int → float` 提升排序（提升少者优先）；④ 仍并列 → `NOVA-E-OVERLOAD`。
**[3]** **成员函数查找采遮蔽制**：在静态类型的继承链上自派生向基类逐层找同名成员集，**命中的第一个**同名集合为候选集，更基类的同名重载集被**遮蔽**（经 `super.名字(...)` 调用）。
**[4]** `override` 必须命中基类同签名成员；同签名是重写（复用方法表槽位），同名不同签名是遮蔽。`async` 修饰不一致的重写为格式错误。
**[5]** 函数名不是一等值；无匿名函数、lambda、闭包。

---

## 9 数组与字符串（Arrays and strings）`✔`

### 9.1 数组类型 `✔` [array.gen]

```nova
let a: array[int | 9] = [9, 3, 7, 5, 1, 8, 6, 4, 2];
```

**[1]** 语义 = **基类 + 维度 + 各维长度**；长度是类型一部分（与 C++ 不同）；每维长度是编译期正整数；运行时行主序连续存储。
**[2]** 多维下标 `a[i, j]` 逗号全维、一次消费全部维度，返回元素引用；**无部分下标、无行视图**（`a[i]` 对多维数组为编译错误）。
**[3]** 数组字面量元素按共同提升类型推导；空数组须类型上下文；元素数量须匹配声明长度。
**[4]** 无隐式深拷贝：传参/返回/赋值复制句柄。任一下标为负或不小于该维长度 → trap。
**[5]** 第三章特性边界：字符串切片仅 `string`（§3.5 [1]），数组无切片类型。
**[6]** 动态数组 ✗。

### 9.2 字符串 `✔` [string]

**[1]** `string` 是内置句柄类型：UTF-8 字节序列 + 长度；接口与规则见 §3.5 [1]（length/by-索引/切片/内容相等/format 一次性写入）。
**[2]** 字面量为只读静态存储对象；不相邻拼接；拼接经 `uni::concat`（§11.5）。
**[3]** `format` 与 `print`/`println` 共享 `{}` 占位规则。

### 9.3 入口与运行时 trap `✔` [array.entry]

程序入口规则见 §8.1 [5]；统一 trap 清单见 §13.3。

---

## 10 类（Classes）`✔`

### 10.1 类声明 `✔` [class.decl]

**[1]** 类声明引入类类型（§3.1）。类名是标识符；同一模块内类名不得与其它顶层声明（类、函数、`type`、常量、导入名）重名。
**[2]** 类**只能**在模块顶层；无嵌套类、无局部类。模块内两遍扫描，顶层声明互相可见、无前向声明。
**[3]** 类声明是**语句**，无值（§4.5 [2]）。类声明后可以有冗余 `;`。空类合法（大小 = vptr 宽度，§10.6 [5]）。

### 10.2 成员与访问组 `✔` [class.member]

**[1]** 成员分三类：字段、成员函数、静态成员；**必须**写在 `public`/`protected`/`private` 访问组 `{ }` 内，无默认访问级别。
**[2]** 访问级别含义：

| 级别 | 可访问 |
|---|---|
| `public` | 任何可见该类的代码 |
| `protected` | 本类及派生类的成员函数 |
| `private` | 仅本类成员函数（含构造函数） |

**[3]** `private` 不参与重写；派生类访问基类 `protected` 成员时，被访问对象静态类型须是本类或其派生类（`self` 恒满足）。
**[4]** 访问控制编译期检查；运行期无访问检查（无反射/RTTI）。
**[5]** 访问组书写顺序不影响布局（§10.6）；无 `friend`、无"同包可见"旁路。

### 10.3 字段与成员函数 `✔` [class.field]

**[1]** 字段 `name : type ;` 或 `name : type = expr ;`；字段名在整条继承链上**必须**唯一（派生类不得声明与基类同名字段——消除布局歧义）。
**[2]** 字段可以是值类型或类类型；初始化规则见 §6.3。`const` 字段必须在声明处给初值。
**[3]** 字段可在可访问处作左值（`p.x = 3.0`；赋值值 `none`）。字段访问是静态的（偏移编译期确定，§10.6），不参与动态分派。
**[4]** 成员函数体内隐式 `self`（不可书写、不可赋值）。**裸名解析顺序**：局部/形参 → 本类及基类成员（派生遮蔽基类）→ 本模块顶层 → `import` 引入名 → `uni` 预导入名。局部名遮蔽字段名时，字段须写 `self.名字`。
**[5]** `$x` 与 `x` 是两个标识符（§1.4），`$` 前缀形参因此不遮蔽同名字段——**这是约定不是语法**。
**[6]** 成员函数可只声明（`;` 结尾），定义须用 `类名::函数名` 在同一模块给出，否则为链接期错误。
**[7]** 成员函数可重载（§8.2）；`self` 可作实参与返回值。

### 10.4 构造与对象创建 `✔` [class.ctor]

**[1]** 构造函数是以 `__constructor__` 为名的成员函数；**不写**返回类型（写者为格式错误）；体内可 `return;` 提前结束。
**[2]** 对象由「类名作为被调用者」的调用表达式创建：`point(1.2, 3.4)`、`tiger()`；无 `new`/`delete`。类名调用即构造调用（故 `Base(derived_expr)` 不可能被解释为转换，§3.6 [4]）。
**[3]** 构造顺序（`B` 为直接基类）：

```
① 分配存储并清零（值类型字段取零值；vptr 置为 D 的方法表）
② 执行基类构造函数 B.__constructor__(...)
③ 按声明顺序求值 D 自己的字段初值
④ 执行 D 的构造函数体
⑤ 构造完成，对象引用交给调用者
```

**[4]** **基类构造函数不自动调用**：派生类的**显式**构造函数**必须**显式写出 `super.__constructor__(实参);`（**无例外**——`nova-spec.md` §6 已定结论）。
**[5]** **唯一豁免：隐式生成的无参构造函数**。类未声明任何构造函数时，其隐式无参构造函数若有基类，则在第 ② 步自动调用基类的无参构造函数（`nvc` 生成代码，不要求也不允许源码书写）；基类无可用的无参构造函数时，该类必须显式声明构造函数并写 `super` 调用，否则为格式错误。
**[6]** 显式 `super.__constructor__(...)` 必须是构造函数体第一条语句（书写位置）；执行时机固定在第 ② 步（早于本类字段初值）——两者独立。
**[7]** 类类型是**引用语义**（§3.4）：变量/字段/实参保存引用，`let b = a` 不产生副本。
**[8]** 构造没有失败通道（无异常）：可能失败的创建用「输出参数 + 状态返回值」的静态工厂函数；`none` 不得表示创建失败（类类型没有空值）。
**[9]** 无隐式对象拷贝：副本须显式声明的构造函数 + 构造调用产生（§3.4 [1]）。
**[10]** 对象存储由运行时自动回收、**非移动**；不可达对象（含引用环）必须最终回收——纯引用计数不合规，实现须用追踪式 GC 或带循环收集的引用计数。GC 元数据不属于 §10.6 布局。
**[11]** 构造期间把 `self` 传出、接收方在构造完成前访问该对象为**未定义行为**；构造期间对被重写函数的调用按正在构造的类分派。

**例**：

```nova
class point {
    private { x: float; y: float; }
    public {
        function __constructor__($x: float, $y: float) { x = $x; y = $y; }
        function __to_string__() -> string {
            let s: string;
            s.format("({}, {})", x, y);   # 一次性初始化写入（§3.5 [1]）
            return s;
        }
    }
}
```

### 10.5 继承、重写与动态分派 `✔` [class.virtual]

**[1]** 继承写 `<-`，**单继承**（多继承 ✗）；继承访问说明符必须显式（`public`/`protected`/`private`），决定基类成员在派生类中的访问级别（`<- public B` 不降级、`protected`/`private` 降级）。
**[2]** 基类须可见且完整声明；继承不得成环。
**[3]** `super` 引用直接基类（成员访问与构造调用）；只能出现在本类成员函数体内。
**[4]** **重写判定**：与可访问基类成员函数签名完全相同（名字 + 参数类型 + 返回类型，无协变）时**必须**写 `override`，漏写或写错均为格式错误。`private`/`static` 成员不参与重写。
**[5]** **无 `virtual` 关键字**：所有 `public`/`protected` 非静态成员函数天生可重写，调用按对象动态类型分派；构造期间例外（§10.4 [11]）。
**[6]** 派生→基类隐式向上转换（§3.6 [4]）；向下转换 ✗（无 RTTI）。
**[7]** 同名不同签名 = 遮蔽（§8.2 [3]）；无 `final`、无抽象类/接口/trait（✗）。

**例**：

```nova
class felid {
    protected { name: string; }
    public {
        function __constructor__(n: &string const) { name = n; }
        function cry() -> void { println("a {} cries", name); }
    }
}
class tiger <- public felid {
    public {
        function __constructor__() { super.__constructor__("tiger"); }
        override function cry() -> void { println("a tiger roars", name); }
    }
}
function main() -> int {
    let t: felid = tiger();   # 隐式向上转换
    t.cry();                  # 动态分派到 tiger::cry
    return 0;
}
```

### 10.6 静态成员、钩子与布局 `✔` [class.layout]

**[1] 静态成员**：`static total: int = 0;`（初值必须常量表达式）；静态函数无 `self`；访问用 `::`（`counter::bump()`，用对象访问为格式错误）；不占对象存储、不占方法表槽位、不参与重写。
**[2] 钩子**：`__名字__` 为保留标识符。本版本定义 `__constructor__` 与 `__to_string__() -> string`；`println`/`print`/`string.format` 的 `{}` 遇类类型值时调用**动态类型**的 `__to_string__`（未定义则填类名）。不定义相等/哈希/比较/析构钩子。
**[3] 对象布局**：

```
偏移 0 : vptr（唯一，跨访问组不变）
其后   : 基类字段（递归展开）→ 本类新字段，均按声明顺序
       : 尾部填充，总大小为对齐整数倍
size(class) = 填充后偏移量；align(class) = max(vptr 对齐, 字段对齐最大值)
```

- vptr 宽度/对齐实现定义；构造期间可临时改写槽位内容但不得改布局与 `sizeof`；
- 静态字段、成员函数不占对象存储；空类大小 = vptr 宽度；GC 元数据在对象之外。
**[4] 方法表**：槽位 = 继承链上所有可分派成员函数（`public`/`protected` 非 `static`），"基类先、声明顺序在后"排列；`override` 复用基类槽位、新增函数追加槽位。
**[5] 二进制兼容**：只在最派生类末尾追加字段/成员函数兼容；在被继承过的类上追加、插字段、重排字段、改签名、改 override 目标均**不**兼容。
**[6] 类与转换**：派生→基类隐式（可写 `type()`✗——§3.6 [4]）；其余不提供。

---

## 11 模块与包（Modules and packages）`✔`

### 11.1 模块单元 `✔` [module.unit]

**[1]** 一个 `.nova` 文件 = 一个模块 = 一个编译单元；模块名 = 去后缀文件名（须合法标识符）；无 `module` 关键字。
**[2]** 顶部声明限制见 §6.1 [5]。**无头文件**：跨模块信息来自编译器产出的**导出元数据**（§11.5）。
**[3]** 可执行程序的根模块必须提供 `main`（§8.1 [5]）；模块名在包内唯一。

### 11.2 包与文件系统 `✔` [module.pkg]

**[1]** 包 = 目录（名须合法标识符）；子包 = 子目录；限定名 `graphics::render`。
**[2]** 包搜索根由项目清单 `nvp.toml` 指定（本项目源码根 + 依赖包目录）。
**[3]** **包作用域** = 包内所有模块导出顶层名字的并集；同名冲突时裸限定访问为格式错误，须用 `包::模块::名字` 消歧。
**[4]** 同一包内模块间不需 `import`，用 `模块名::名字` 访问；模块名/子包名不得与导出顶层名字同名。

### 11.3 导入与名字解析 `✔` [module.name]

**[1]** 两种形式：`import 包;`（仅使包名可解析，访问写 `包::名字`）与 `import 包::模块;`（引入裸名）。**别名与选择性导入 ✗**。
**[2]** 所有 `import` 必须在模块最前（顺序不影响解析结果）。
**[3]** 冲突规则：与本模块顶层同名、与另一 `import 包::模块` 同名 → 格式错误；与 `uni` 预导入名同名 → 合法（遮蔽，用 `uni::名字` 显式访问）。
**[4]** 重复 `import` 幂等；**循环 `import` 合法**（不产生初始化代码）；解析失败是格式错误（编译期诊断）。
**[5]** 无传递可见性：`a` 导入了 `b`、`b` 导入了 `c`，`a` 用 `c::` 须自己 import。
**[6]** `::` 是路径解析运算符（左结合，原子项 §1.7 [1]）：左操作数限包名/模块名/类名，右操作数限名字；`obj::x` 为格式错误；无前导 `::`。
**[7]** 裸名解析顺序：局部 → 本类及基类成员 → 本模块顶层 → `import 包::模块` 引入名 → `uni` 预导入名；"最近者胜"，被遮蔽者可用限定名显式访问（`self.`/`super.`/`模块名::`/`uni::`）。
**[8]** 名字解析完全在编译期；无运行时名字查找。

**合法限定名形式**：`名字`｜`模块名::名字`｜`包名::名字`｜`包名::模块名::名字`｜`类名::静态成员`｜`包名::类名::静态成员`。

### 11.4 模块级可见性 `✔` [module.vis]

**[1]** 顶层声明**默认导出**；`private function ...` 仅本模块可见。`public`/`protected` 修饰顶层为格式错误（模块层无继承）。
**[2]** 模块私有名不得出现在导出接口（导出的参数/返回类型、导出类的基类与 public/protected 成员类型）中。
**[3]** 无 `export`/`extern`/宏/条件编译。

### 11.5 标准包 `uni` `✔` [module.uni]

**[1]** `uni` 是**唯一预导入**包，裸名可用、可遮蔽（`uni::名字` 显式访问）；`import uni;` 合法（等价无操作）。其余包（`math` 等）显式 `import`（`math` 与 `uni` 平级）。
**[2]** 语言级入口：

| 名字 | 语义 |
|---|---|
| `range(a, b)` / `range(a, b, j)` | `for` 迭代位内建（§5.4 [2]），非一等值 |
| `print(fmt, args...)` / `println(...)` | `{}` 占位格式化输出；可变尾参 |
| `input(targets...)` | 读取并**写入**实参（可写稳定左值；计入初始化写入，§6.2 [3]） |

**[3]** `format` 不是 `uni` 顶层入口，而是 `string` 成员方法（§3.5 [1]），占位规则与 `print` 相同。

### 11.6 `nvp` 与原生包桥接 `✔` [module.nvp]

**[1]** 工具链：编译器 `nvc`、链接器 `nvld`、包管理器 `nvp`。项目清单 `nvp.toml`（name/version/nova/src/[deps]/[native]）+ 锁文件 `nvp.lock`；"清单 + 锁相同 ⇒ 依赖集合与内容相同"（可复现构建）。命令：`up`/`add`/`remove`/`build`/`run`/`clean`/`list`。
**[2]** 流水线：解析清单 → `nvp up` → `nvc` 编译（每模块 → 目标文件 + **导出元数据**）→ 系统 C++ 编译器现场编译原生包 → `nvld` 链接（携带符号表）。
**[3]** **导出元数据**至少含：导出名字的修饰符号名、函数/成员函数类型签名、类布局与方法表布局、目标 ABI 版本。
**[4]** **原生包**（C++ 编写，`nvp` 下载源码现场编译）：须提供 Nova 接口文件 `<模块名>.novai`（受限产生式：只允许函数声明、类声明（字段/成员函数/构造入口/静态成员）、`type`、`const`；语法上即无函数体、无字段初值、无 `async`、构造函数无修饰符/无返回类型；实例字段不得是引用；静态字段仅限值类型（`bool/char/int/float`，别名展开后判定））。
**[5]** **名字修饰**：

```
模块级函数 : _NV enc(包) enc(模块) enc(名字) "_" 参数编码 "E"
成员函数   : _NV enc(包) enc(模块) enc(类名) enc(名字) "_" 参数编码 "E"
静态字段   : _NVG enc(包) enc(模块) [enc(类名)] enc(名字)
enc(s) = 十进制字节长度 + s；类型码：v b c i f s a R<T>=&T C<enc(限定名)>=类类型
async 在参数编码前加 "A"；返回类型不入符号名；别名必须展开；array 只编码 a
```

**[6]** **调用约定**：值类型按目标平台 C 约定传值；`&T`、类类型、`string`/`array` 按对象地址传；可跨边界值传递的类限"无基类且无可分派成员函数"且字段全值类型；其余类按不透明句柄。
**[7]** **内存所有权**：Nova 侧传入引用仅调用期间有效；C++ 侧不得长期保存；C++ 侧创建对象须经接口文件构造入口交 Nova 运行时管理；`_NVG` 静态字段的存储属原生库（Nova 侧经地址读写，初值由 C++ 静态初始化提供）。
**[8]** **错误传播**：无异常，C++ 异常**不得**跨边界（否则 UB）；失败用返回值/输出参数表达。原生函数不得 `async`。
**[9]** ABI 版本不匹配不得链接；越过边界后 Nova 的类型/内存安全保证不再成立；`nvp` 构建输出必须列出全部原生依赖。

---

## 12 异步执行（Async execution）`✔`

### 12.1 执行模型 `✔` [async.model]

**[1]** 执行流 = 一个或多个**任务**（task）；程序启动时存在执行 `main` 的初始任务。
**[2]** **协作式单线程并发**：任一时刻至多一个任务在运行；任务只在切换点让出。**不提供并行**（无多线程、无原子、无锁）——不存在真并行数据竞争，但**逻辑竞态仍可能**，程序应把必须一起生效的修改放在两个切换点之间（该区间相对其它任务原子）。
**[3]** 内存模型：任务内按程序顺序；**切换点是唯一跨任务同步边**（A 在 p 让出、B 在 p 后的读可见 A 在 p 前的写，happens-before）；无需内存屏障；不提供 `volatile`/原子/内存序标注。
**[4]** 被挂起任务的栈与局部变量**必须**被 GC 视为可达根。调度器不保证公平性；不含切换点的死循环永久占住调度器；无优先级/取消/超时。

### 12.2 任务与 `await` `✔` [async.task]

**[1]** `async function f(...) -> T` 声明异步函数；**返回类型注解表示结果类型**（`await` 得到的类型），省略则为 `void`。
**[2]** 调用异步函数**不阻塞**：创建任务交调度器，立即求值为**任务句柄**。事件顺序固定：① 求值实参 → ② 创建任务并登记到所有者块 → ③ 新任务可能被推进到第一个切换点（实现定义）→ ④ 调用者继续。
**[3]** 任务句柄类型 `Task⟨f⟩` **不可书写**（本规范元语言记号，非语言符号）：不得出现在形参/返回/字段/`type`/数组基类型，**只能**是局部变量（`let` 推导）。
**[4]** **结构化并发**：任务归**最近的词法块**所有；控制流离开块（正常结束/`return`/`break`/`continue`）时运行等待该块登记的未完成任务；同一任务不会被等待两次；等待顺序不保证；等待不取消。
**[5]** `await` 是前缀运算符：操作数必须是任务类型表达式；语义 = 求值操作数 →（未完成则挂起，切换点）→ 唤醒后取其结果。结果为 `void` 时表达式类型 `void`：**不可绑定 `let`**、不可作实参/操作数，只能作表达式语句丢弃。`await` 不传播错误（失败已编码在返回值）。`await` 只能出现在异步函数体内。
**[6]** **同步函数不得调用异步函数**（格式错误；无隐式阻塞原语）——`async` 沿调用链向上传播（函数着色）。
**[7]** 同一句柄可多次 `await`（结果相同）。

### 12.3 `yield` 与生成器 `✔` [async.gen]

**[1]** 含 `yield` 的异步函数是**生成器**；返回类型注解表示**元素类型**（不得 `void`）。
**[2]** 调用生成器返回流句柄 `Stream⟨f⟩`（同样不可书写）；**惰性启动**（首次请求元素才执行）。
**[3]** `yield 表达式;`（必须带值，表达式值 `none`）；生成器体内 `return` 不得带值。
**[4]** 消费形式 `for x in 流 { }`（取元素处是切换点，**必须**在异步函数体内）；同步序列（`range`/`array`/`string`，§5.4 [3]）的普通 `for` 不是切换点。
**[5]** 流的放弃（`break`/离开块/句柄重新赋值/丢弃）：终止生成器、等待其体内未完成任务、回收挂起状态；无用户清理代码。流是一次性的；耗尽 ≠ 放弃。
**[6]** `yield` 出现在非异步函数体内为格式错误。

### 12.4 异步与类 `✔` [async.class]

**[1]** 成员函数与静态成员函数可 `async`（动态分派照常，`await obj.f()`）。
**[2]** 构造函数**不得** `async`；钩子（`__to_string__`）不得 `async`。
**[3]** 重写要求 `async` 修饰一致；`async` 不是签名的一部分（不能靠它形成重载）。任务/流句柄不能作为字段类型。
**[4]** 跨模块 `await` 合法（按 §11 可见性）；原生函数不得 `async`；对原生函数的调用是**阻塞点**（调用期间调度器不推进其它任务，保证 §12.1 [2] 的原子性）。

### 12.5 生命周期与退出 `✔` [async.life]

**[1]** `main` 为异步函数时，运行时驱动 `main` 任务至完成，返回值即退出码；结构化并发保证退出时无未完成任务。
**[2]** 程序结束时不保证任何确定清理时机（无析构）；`main` 任务死锁则程序挂起（无检测/超时）。

### 12.6 错误处理与本版本限制 `✔` [async.limit]

**[1]** 无异常：异步失败必须编码在返回值——值类型用哨兵值/状态编码；类类型用「输出参数 + `bool` 状态返回」；`none` 不得表示失败。
**[2]** 不提供：取消/超时/优先级/detach、`await` 组合子、并行。已知限制：函数着色、原生调用阻塞调度器、无公平性保证。

---

## 13 诊断：移植期错误与 trap（Diagnostics）`✔`

### 13.1 移植期错误（格式错误）`✔` [diag.compile]

下列情形**必须**诊断并拒绝编译（非穷举）：未声明名字、块变量前向引用；类型不匹配且无合法提升；`void` 表达式出现在值位置；`if`/`while` 条件非 `bool`；左值/可写性不满足、`const` 被写；分支类型不完全同型、`match` 表达式缺 `else`；重载无匹配或歧义（`NOVA-E-OVERLOAD`）；读取未证明已初始化（`NOVA-E-USE-BEFORE-INIT`）；非 `void` 函数有不可返回路径（`NOVA-E-RETURN`）；`let x;`；字面量 `int / 0`；关键字作标识符；非法 token；多目标赋值左右数量不等；访问控制/`override`/继承/构造 `super` 相关违规（§10）。

### 13.2 警告 `✔` [diag.warn]

实现**应该**诊断：`float(i)` 失精、超范围浮点字面量、可证明的死代码、`a and b or c` 类易错优先级组合。

### 13.3 运行时错误：trap 模型 `✔` [diag.trap]

Nova 无异常处理。**trap** = 立即终止、输出诊断（错误类别与源位置）、非零退出码；不执行栈展开与清理。

| 事件 | 处理 |
|---|---|
| 数组/字符串索引、切片越界 | trap |
| `int` 除零、`% 0`、**算术溢出**（含 $-2^{31} \bmod -1$、一元负越界） | trap（`NOVA-R-INT-OVERFLOW`） |
| `int(f)`/`char(i)`/`char(f)` 越界、NaN、∞ | trap |
| `range` 步长为零 | trap（`NOVA-R-ZERO-STEP`；常量可证明时报同一码） |
| 静态不可判定的悬垂引用 | trap |
| 递归过深（栈耗尽） | trap |

**理由**：无异常语言中越界行为只有 UB / trap / 哨兵三选；哨兵破坏类型（`a[i]` 返回引用），UB 违背强类型与可教性，trap 是唯一与引用语义自洽且实现代价最低的方案。

### 13.4 错误码约定 `✔` [diag.code]

统一入口 `nova_trap(code, source_span)`；编译期与运行时的同一表达式检查（常量折叠）使用相同错误编号；每个错误携带源文件、行列、主错误码与相关声明位置。

---

## 14 图灵完备性（Turing completeness）`✔`

**[1]** 本章模型构成图灵完备的计算核心：无界存储（`while` 无界运行 + 堆分配类对象；内存耗尽 trap 不限制理论能力）、条件（`if`/`match`，§5.2/§5.3）、循环（`while`，§5.4）、数据编码（`array[int | n]` 表示 bignum，§9.1）、子程序与递归（§8）。
**[2]** 由上述可模拟任意图灵机：带存储于 `array`，头位置于 `int`，状态转移于 `match`，主循环于 `while`。
**[3]** 边界声明：`input`/`print`/`println` 是 `uni` 的 I/O 原语，不参与完备性论证；异步（§12）协作式并发不扩展可计算性。

---

# 附录 A（规范性）语法汇总（Grammar summary）

> 本文附录合并第二章 §2.12 的 EBNF 与第四章附录 G 的归属声明。合并后的完整语法以本附录为准；
> `class_decl` 等产生式在 §10/§11 正文给出。本附录未列的产生式（`.novai` 受限产生式、`native_*` 系列）
> 属原生包接口文件，见 §11.6 [4]。

## A.1 词法

```ebnf
token        = identifier | keyword | literal | operator | delimiter
identifier   = ( letter | "_" | "$" ) { letter | digit | "_" | "$" }
letter       = "A".."Z" | "a".."z"
digit        = "0".."9"
comment      = "#" { non-newline } newline
literal      = boolean-literal | int-literal | float-literal | char-literal
             | string-literal | none-literal | array-literal
int-literal  = digit { digit | "_" } | "0x" hex-digit { hex-digit }
             | "0b" ( "0" | "1" ) { "0" | "1" }
float-literal = [ digits ] "." digits [ exponent ] | digits exponent
char-literal = "'" ( graphic-char | escape-seq ) "'"
string-literal = '"' { graphic-char | escape-seq } '"'
escape-seq   = "\n" | "\r" | "\t" | "\\" | "\'" | "\0" | "\""
```

## A.2 类型

```ebnf
type            = value-type | reference-type | class-type | array-type
value-type      = "bool" | "char" | "int" | "float" | "void"
reference-type  = [ "const" ] "&" type | "&" type [ "const" ]
class-type      = class-name | "string"
array-type      = "array" "[" type "|" expression { "," expression } "]"
```

## A.3 表达式与语句

```ebnf
expression           = assignment-expression
assignment-expression = conditional-expression [ "=" assignment-expression ]
                       | multiple-assignment
multiple-assignment  = lhs "," { "," lhs } "=" expression { "," expression }
lhs                  = identifier | postfix-expression "." identifier
                      | postfix-expression "[" expression { "," expression } "]"
unary-expression     = [ "-" | "+" | "not" ] postfix-expression
postfix-expression   = primary-expression { call-suffix | index-suffix | member-suffix }
call-suffix          = "(" [ expression { "," expression } ] ")"
index-suffix         = "[" expression { "," expression } "]"
member-suffix        = "." identifier
primary-expression   = literal | qualified-name | "(" expression ")" | "self" | "super"
                      | array-literal | block-expression | if-expression
                      | match-expression | await-expr
qualified-name       = identifier { "::" identifier }
block-expression     = "{" { statement } [ expression ] "}"
await-expr           = "await" expression        (* 仅限 async 函数体内，§12.2 [5] *)

statement    = expression-statement | declaration | if-statement | match-statement
             | while-statement | for-statement | jump-statement | block | ";"
declaration  = "let" [ "&" ] identifier [ ":" type ] [ "=" expression ] ";"
             | "const" identifier ":" type "=" expression ";"
             | "type" identifier "=" type ";"
             | function-declaration | class-declaration
import-decl  = "import" identifier { "::" identifier } ";"   (* 模块最前，§11.3 [2] *)
if-statement = "if" expression block { "elseif" expression block } [ "else" block ]
match-statement = "match" expression "{" { pattern "=>" block } [ "else" "=>" block ] "}"
pattern      = expression
while-statement = "while" expression block
for-statement = "for" identifier "in" iterable block
iterable     = "range" "(" expression { "," expression } ")" | array-expr | string-expr
jump-statement = "return" [ expression ] ";" | "break" ";" | "continue" ";"
function-declaration = "function" identifier "(" [ parameter-list ] ")" [ "->" type ] block
async-function-declaration = "async" function-declaration
parameter    = identifier ":" type
```

## A.4 类与模块

```ebnf
class_decl   = "class" identifier inheritance? "{" member-group* "}" ";"?
inheritance  = "<-" access-specifier qualified-name
access-specifier = "public" | "protected" | "private"
member-group = access-specifier "{" member* "}"
member       = field-decl | method-decl | constructor-decl | static-member
field-decl   = "const"? identifier ":" type-spec ( "=" expression )? ";"
method-decl  = method-modifier* "function" identifier "(" param-list? ")"
               ( "->" type-spec )? ( block | ";" )
method-modifier = "override" | "async"
constructor-decl = "function" "__constructor__" "(" param-list? ")" block
static-member = "static" ( static-field | static-method )
static-field = "const"? identifier ":" type-spec "=" const-expr ";"
static-method = method-modifier* "function" identifier "(" param-list? ")"
               ( "->" type-spec )? ( block | ";" )
super-call   = "super" "." "__constructor__" "(" [ arg-list ] ")" ";"
construct-expr = qualified-name "(" [ arg-list ] ")"
module-unit  = { import-decl } { top-decl }
top-decl     = "private"? ( class-decl | function-decl | async-function-decl
             | type-decl | const-decl )
type-decl    = "type" identifier "=" type-spec ";"
const-decl   = "const" identifier ":" type-spec "=" const-expr ";"
const-expr   = expression        (* 常量性是语义检查，§6.1 [3] *)
```

## A.5 产生式说明（规范性约束）

1. `identifier` 不得取 `__constructor__`：构造函数一律用 `constructor-decl`（无返回类型）。
2. 修饰符合法组合由文法 + 语义共同确定：`static override` 是格式错误（§8.2、§10.3）。
3. `.novai` 接口文件的受限产生式（`native_*` 系列）见 §11.6 [4]，其文法本身即排除函数体、字段初值与 `async`。
4. 多目标赋值的左值列表按顶层逗号切分；`[ ]` 内逗号归属索引后缀（§4.7 [2]）。
5. 合并各章 EBNF 时同名产生式以本附录为准；做语法闭包检查（每个产生式有定义或明确标注外部引用）。

---

# 附录 B（资料性）未定项与开放问题

> 以下为**有意开放**项，不构成章节间冲突；状态与出处见 `nova-spec.md` §9.4 与第四章 §6.13/§7.10/§8.11。

| # | 未定项 | 影响 | 现状 |
|---|---|---|---|
| 1 | `vptr` 具体宽度与调用约定 | 目标平台 ABI、原生包结构体对应 | 取目标平台指针宽度（§10.6 [3]） |
| 2 | GC 具体算法与暂停策略 | 运行时 | 约束：非移动 + 回收含引用环；纯引用计数不合规（§10.4 [10]） |
| 3 | `uni` 内部模块划分与完整接口（含 `format` 最终签名） | 标准库设计 | 语言级入口已定稿（§11.5） |
| 4 | `nvp` 注册表/分发协议、版本约束语法 | 包管理器 | 只约束可复现性（§11.6 [1]） |
| 5 | 任务/流运行时表示与调度器接口 | 代码生成、ABI | 语义已定（§12），实现自由 |
| 6 | 成员函数常量性（`const` 方法） | 类型系统 | §10 只用了最小含义 |
| 7 | 命令行参数 API 具体名称（`uni::args()` 形态） | 标准库 | `main` 保持无参（§8.1 [5]） |
| 8 | ABI 版本号方案与兼容判定 | 原生包链接准入 | 不匹配不得链接（§11.6 [9]） |
| 9 | `array` 在 `let` 位置的省长度用法 | 类型推导 | 形参处已定（§9.1 [1]） |
| 10 | 抽象成员/接口、`equals` 等内容比较钩子 | 面向对象设计 | 明确排除（§10.5 [7]、§10.6 [2]）；内容相等用普通成员函数 |
| 11 | `vector` 等容器与 `reserve` 等容量操作 | 类型系统 | 明确**暂不写**（推迟，不是未定） |
| 12 | 深拷贝细则 | 编译器实现阶段 | 已定"无隐式拷贝、副本须显式"（§3.4 [1]） |

# 附录 C（规范性）实现定义行为汇总

实现必须为每个实现定义项选定确定行为并文档化：

| # | 项 | 出处 |
|---|---|---|
| 1 | `vptr` 宽度与对齐 | §3.2、§10.6 [3] |
| 2 | GC 算法与元数据位置（在满足 §10.4 [10] 约束下） | §10.4 [10] |
| 3 | 对象分配策略 | §10.6 |
| 4 | 方法表槽位在规则内的生成细节 | §10.6 [4] |
| 5 | 原生调用约定细节（寄存器/栈、结构体传参） | §11.6 [6] |
| 6 | 任务开始执行时机（调用点立即 / 入队后） | §12.2 [2] |
| 7 | 任务/流运行时表示与创建/唤醒接口（须跨模块一致） | §12.2 |
| 8 | 调度策略、就绪队列、唤醒顺序 | §12.1 |
| 9 | 任务栈大小与栈溢出检测 | §12.1 |
| 10 | trap 退出码的具体非零值 | §13.3 |
| 11 | 浮点运算是否用更高中间精度（最终须 binary64） | §3.3 [4] |

**未指定 / 不可移植（非 UB）**：任务推进量与相对速度、等待多个任务的顺序、`array`/`string` 内部布局细节（不透明处理前的实现选择）——程序**不得依赖**。

**未定义行为**：构造完成前把 `self` 传出并被访问（§10.4 [11]）；跨编译单元方法表槽位不一致；C++ 异常跨原生边界；原生包违反 §11.6 [6][7] 约定；任务栈溢出。

# 附录 D（资料性）工具链实现约束

## `nvc` 编译器

1. 词法：关键字、`<>`、`::`、`&T`、`const &T`、`&T const`、`array[T | n]` 唯一解析路径；最长匹配无上下文例外（§1.2）。
2. 名字解析：作用域树四层（§2.1）；类成员在成员解析阶段加入候选集，遮蔽制（§8.2 [3]）。
3. 类型检查：表达式同时记录类型、值类别、写权限、生命周期；`void` 表达式不得进入值位置。
4. 确定性初始化：未声明/未初始化/已初始化/不可继续四状态（§6.2）。
5. 重载解析：精确 + 固定提升链，产生唯一稳定符号键（§8.2 [2]）。
6. 常量求值与运行时代码生成复用同一套转换/溢出/越界/trap 检查与错误码（§13.4）。
7. 诊断含源位置与主错误码；最小错误码集：`NOVA-E-USE-BEFORE-INIT`、`NOVA-E-NOT-LVALUE`、`NOVA-E-BORROW-LIFETIME`、`NOVA-E-OVERLOAD`、`NOVA-E-TYPE`、`NOVA-E-RETURN`、`NOVA-E-RUNTIME-BOUND`。

## `nvld` 链接器

- 只处理已修饰符号，不重新执行类型检查/重载决议；函数符号含限定名 + 参数类型列表（返回类型不入符号键）；类方法符号含类限定名并记录虚调用关系；类元数据（字段偏移、方法表、ABI 版本）不兼容则拒绝；跨平台兼容由目标三元组区分。

## 编辑器连接器

- 依赖稳定语法树/符号表/诊断协议：补全按作用域树与成员可见性过滤；悬停显示「值 / 句柄 / 只读引用 / 可写引用」；未初始化读取、重载歧义、非法转换、数组越界（可静态判定时）用 §D `nvc` 的错误码定位到表达式；`match`/`if`/`range`/多目标赋值提供成对括号与分支模板；格式化保持多目标赋值左到右书写与 `array[T | n]` 形状。

## 示例回归清单（合并版验收）

下列 `nova程序示例/` 程序必须在本规范下通过解析、类型检查与诊断回归：

| 文件 | 覆盖点 |
|---|---|
| `gcd.nova` | 多目标交换、`<>`、`%`、入口 |
| `gcd_递归.nova` | 递归、early return 路径 |
| `hanoi.nova` | `let n: int;` + `input(n)` 初始化写入、递归 |
| `point.nova` | 类字段写入、构造调用、`string.format` 一次性写入、字符串返回 |
| `inherit.nova` | `&string const` 收字面量、显式 `super.__constructor__`、`override`、动态分派 |
| `sort.nova` | `const &array` 只读传参、数组下标左值、`a[k], a[i] = ...` 交换、`range` |
| `eqroot.nova` | `import math;`、`math::sqrt` 限定名、`match` 比较模式分支、浮点运算 |

---

# 文档尾注

- **合并来源映射**：§1–§3 主要来自第二章（v0.4）；§4–§9 合并第二章 §2.7/§2.8 与第三章 §3.2–§3.4；§10–§12 来自第四章（第 6–8 部分，按 S-01…S-30 回改后）；§13–§14 来自第二章 §2.10/§2.11。
- **与既有文件的关系**：本文件不改写 `第二章/`、`第三章/`、`第四章/`、`跨章节语义协调.md`、`nova-spec.md`。冲突裁决记录见 `跨章节语义协调.md`（S-01…S-30）；结论层权威仍为 `nova-spec.md`；本文件与其结论一致。
- **版本**：1.0（2026-09-24，合并版首发；已纳入 S-01…S-30 全部裁决与全部过时表述清订）。
