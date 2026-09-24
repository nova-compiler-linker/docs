# Nova 语言规范 —— 第 2 章 词法与语言基本模型

> **Nova Language Specification, Chapter 2: Lexical Structure and Fundamental Model of the Language**
>
> 版本：0.4（草案，2026-09-24 按《跨章节语义协调》S-01…S-29 回改；v0.3 恢复 `int → float` 算术自动提升、`::` 原子项注、format 可变参豁免、遮蔽制重载查找对齐；v0.4 修正 2.5.1 名字唯一性规则（S-04 兜底）、`return` 加宽（S-29）、删除 `type()` 类转换（S-28）、订正 2.7.11 切片误引）｜ 依据：`nova-spec.md`（权威规范）、`01-语言设计.md`、`02-类型细节.md`、`nova-types.yaml`、`nova程序示例/`、`跨章节语义协调.md`、第三章模块包、第四章工作稿
> 本章撰写体例参照 ISO/IEC 14882（C++ 语言标准）。凡权威规范已定项，一律照写；凡规范未定而本章必须回答的问题，本章按业界最佳实践给出定义，并在 `nova-ch2-补充定义说明.md` 中逐条登记、说明理由。被跨章节协调推翻的原定义在正文中注明"推翻原 D-xx"。
> 语法描述采用 ISO EBNF（IEC 14977），产生式以等号定义，终结符用等宽字体加引号。

---

## 2.1 范围（Scope）

本章规定 Nova 程序的词法结构、基本概念（作用域、名字、对象、值、求值顺序）与类型系统的基本模型，并给出表达式与语句的语法骨架及其求值语义。

本章**不**规定：

- 类、继承与动态分派的完整规则（第 6 章）；
- 包、`import`、`::` 名称解析与工具链（第 7 章）；
- 并发语义（`async` / `await` / `yield`，目前仅为保留关键字，见 2.4.5）；
- 拷贝语义的完整规则（权威规范 §2.5 明确暂缓；本章仅定义其前提模型，见 2.5.4、2.6.5）。

## 2.2 术语与定义（Terms and definitions）

| 术语 | 定义 |
|---|---|
| **源程序**（source program） | 一个或多个 Nova 源文件（扩展名 `.nova`）的有序集合，经翻译后产生可执行程序。 |
| **翻译单元**（translation unit） | 单个源文件经词法分析、语法分析与语义分析后的编译产物。Nova 无头文件，翻译单元即源文件本身。 |
| **token** | 词法分析的最小语义单位：标识符、关键字、字面量、运算符、界符之一。 |
| **对象**（object） | 内存中一个有类型的存储区域，具有大小、对齐与生命周期。 |
| **值**（value） | 某类型的合法数据实例。值是抽象的，对象是值的存储载体。 |
| **变量**（variable） | 与一个对象（或引用绑定）关联的具名存储。 |
| **绑定**（binding） | 名字与实体（变量、函数、类型等）之间的关联。 |
| **左值**（lvalue） | 指代一个可寻址对象的表达式，可作赋值左值。 |
| **右值**（rvalue） | 指代一个值的表达式，未必可寻址。 |
| **实参求值**（evaluation） | 计算表达式得到值（及副作用）的过程。 |
| **副作用**（side effect） | 存储修改、`io` 交互或运行时错误的发生。 |
| **trap**（运行时错误终止） | 见 2.10.3：程序立即以非零退出码终止，不执行任何清理。 |
| **未定义行为**（undefined behavior） | 标准不施加任何要求的行为。Nova 刻意将多数常见错误归为 trap 而非未定义行为（见 2.10.3 表）。 |

## 2.3 记法（Notation）

- 语法以 EBNF 给出。`{ X }` 表示 `X` 重复零次或多次，`[ X ]` 表示可选，`( X | Y )` 表示二选一。
- 语义描述中的 **必须（shall）** 表示规范性要求，**可以（may）** 表示允许的裁量，**不应（should）** 表示建议。
- 公式使用 LaTeX 排版；$\mathbb{Z}$ 为整数集，$\mathbb{B}=\{\mathsf{false},\mathsf{true}\}$ 为布尔域。

---

## 2.4 词法结构（Lexical structure）

### 2.4.1 源字符集（Source character set）

Nova 源文件是**字节流**，按 UTF-8 解释。可出现在源文件**词法结构之外**（即不在字符串/字符字面量内）的字符仅限：

```
source-char-set = printable-ascii | "@" | "$" | white-space
```

- `printable-ascii`：ASCII 0x21–0x7E（可打印、非空白）；
- 空白：空格（0x20）、水平制表（0x09）、换行（0x0A）、回车（0x0D）；
- 换行序列：`\n`、`\r\n`、`\r` 三者等价，均终止行注释并分隔行。

**规定**：源文件的**第一行**若以 `#!/` 开头，该行整体作为脚本行忽略（不参与词法分析）。

字符串与字符字面量内允许的全部 Unicode 字符以 UTF-8 编码存储（见 2.4.6.4、2.4.6.5 的转义规则）。

### 2.4.2 翻译阶段与 token（Phases of translation）

源程序按下列阶段顺序翻译，后续阶段只见前一阶段的产物：

1. **换行归一**：所有换行序列统一为 `\n`。
2. **注释替换**：每个注释被替换为**一个空格**（即注释可分隔 token，见示例 2.4.3）。
3. **token 化**：按**最长匹配**（maximal munch）原则切分 token。
4. 语法分析、语义分析、代码生成（第 3–7 章）。

**最长匹配规定**：在任一位置，词法分析器必须匹配能构成合法 token 的**最长**字符序列，且**不依赖上下文**。据此：

- `a<>b` 切为 `a` `<>` `b`（不等号），而非 `a` `<` `>` `b`；
- `a=>b` **恒切为** `a` `=>` `b`（`=>` 是界符；最长匹配无上下文例外）。若 `=>` 出现在 `match` 分支之外，属**语法错误**，由语法分析阶段诊断——不得由词法层按上下文改切分；
- `a::b` 切为 `a` `::` `b`；`a : :b`（带空格）切为 `a` `:` `:` `b`，是语法错误。

token 分类：

```
token = identifier | keyword | literal | operator | delimiter
```

### 2.4.3 注释（Comments）

```
comment = "#" { any-char-except-newline } newline
```

- `#` 起始的行注释，到行尾结束。普通注释与文档注释统一用 `#`（文档注释由 doxygen 处理，不在本规范范围）。
- 注释在翻译阶段 2 被替换为一个空格。因此：

```nova
let a = 1 # 注释
+ 2;      # 合法：a 的值为 3（注释未切断表达式）
```

- 注释**不能**出现在 token 内部（如 `i#comment#d` 不构成 `id`），但可出现在任意两个 token 之间。
- 无块注释。

### 2.4.4 标识符（Identifiers）

```
identifier = ( letter | "_" | "$" ) { letter | digit | "_" | "$" }
letter     = "A" | ... | "Z" | "a" | ... | "z"
digit      = "0" | ... | "9"
```

- 大小写**敏感**：`Foo` 与 `foo` 是不同标识符。
- 标识符不得以数字开头，不得与关键字（2.4.5）同名。
- 标识符无长度上限；实现**不应**截断。
- `$` 是普通标识符字符，**不**赋予任何特殊语言含义（示例中 `$x` 形参仅是利用 `$` 与字段名区分的**书写约定**，非语言规则）。
- `_` 单独出现是合法标识符，惯例上表示"故意忽略的值"。

### 2.4.5 关键字（Keywords）

下列标识符为关键字，保留用途，不得作名字：

```
and      array    async    await    bool     break    char     class
const    continue else     elseif   false    float    for      function
if       import   in       int      let      match    none     not
or       override private  protected public   return   self     static
string   super    true     type     void     while    xor      yield
```

补充规定：

- `elseif` 是 `elif` 语义的语法糖（等价于嵌套 `else if`，见 2.8.2）；
- `type` 为类型别名引入符（相当于 C++ 的 `typedef`）；
- `import` 为模块导入关键字（第 7 章定义其语法；本章词法表登记，权威规范 §1.4 原缺，已补，见跨章节协调 S-14）；
- `async` / `await` / `yield` 为关键字，其语法与并发语义由第 8 部分（异步执行）定义；本章仅登记词法地位，不定义行为；
- 语法糖 `elif` **不是**关键字，也不被接受——只有 `elseif` 合法。

### 2.4.6 字面量（Literals）

#### 2.4.6.1 布尔字面量

```
boolean-literal = "true" | "false"
```

类型 `bool`。

#### 2.4.6.2 整数字面量

```
int-literal  = dec-literal | hex-literal | bin-literal
dec-literal  = digit { digit }
hex-literal  = "0x" hex-digit { hex-digit }
bin-literal  = "0b" ( "0" | "1" ) { "0" | "1" }
```

- 无八进制（避免 `010` 这类歧义）。
- 数字间允许单个 `_` 作分隔符：`1_000_000` 合法；`_` 不参与数值。
- **无类型后缀**（无 `L` / `u`）。字面量默认类型 `int`；若值超出 `int` 范围（2.6.3.3）但落在 `float` 可精确表示范围内，**仍是错误**——Nova 不提供隐式放大。
- 字面量的值以 2.6.3.3 的二补数语义解释；十进制字面量超过 $2^{31}-1$ 为**移植期错误**。

#### 2.4.6.3 浮点字面量

```
float-literal = digits "." [ digits ] [ exponent ]
              | digits exponent
              | "." digits [ exponent ]
exponent      = ( "e" | "E" ) [ "+" | "-" ] digits
```

- 类型 `float`（IEEE 754 binary64，见 2.6.3.4）。
- `1.`、`.5`、`1e10`、`1.5E-3` 均合法。
- 含小数点或指数但**无**其他后缀的十进制字面量一律为 `float`；`0` 是 `int`，`0.0` 是 `float`。
- 超出 binary64 可表示范围的字面量：绝对值过大者取 $+\infty / -\infty$（**警告**），过小的正数取 $+0$（**警告**）；实现应报告但可继续编译。

#### 2.4.6.4 字符字面量

```
char-literal = "'" ( graphic-char | escape-seq ) "'"
```

- 类型 `char`，值域为 0–127 的 ASCII 码（**基本字符集**）。
- 转义序列：

| 转义 | 含义 | 码值 |
|---|---|---|
| `\n` | 换行 | 10 |
| `\r` | 回车 | 13 |
| `\t` | 制表 | 9 |
| `\\` | 反斜杠 | 92 |
| `\'` | 单引号 | 39 |
| `\0` | NUL | 0 |

- 未列出的转义序列为错误。非 ASCII 字符字面量（`'中'`）为错误。

#### 2.4.6.5 字符串字面量

```
string-literal = '"' { graphic-char | escape-seq } '"'
```

- 类型 `string`。字符串是**字节序列**，字面量的源文本按 UTF-8 编码存入；转义序列在字面量内展开。
- 转义序列同 2.4.6.4，另加 `\"`（双引号，34）。
- 字符串字面量不可跨行（无多行字面量语法）。
- 相邻字符串字面量**不**自动拼接（与 C 不同）：`"a" "b"` 是语法错误；字符串拼接经 `uni` 包（2.6.4.1）。
- 字面量的**运行时表示**：只读、静态存储期对象，含长度信息（见 2.6.4.1）。

#### 2.4.6.6 `none` 字面量

```
none-literal = "none"
```

类型 `void`，值唯一（见 2.6.3.5）。

#### 2.4.6.7 数组字面量

```
array-literal = "[" [ expression { "," expression } ] [ "," ] "]"
```

- 元素类型必须一致（提升后，见 2.6.7）；长度为元素个数。
- 字面量的类型：`[e1, e2, ..., en]` 的元素类型均为 `T` 时，字面量类型为 `array[T | n]`。
- `[]`（空数组字面量）类型不定，**只能**在上下文可推断类型时使用：`let a: array[int | 0] = [];`。
- 末尾允许**一个**多余逗号（`[1, 2, 3,]`），便于逐行书写（与上文文法 `[ "," ]` 一致，只允许一个）。

### 2.4.7 运算符与界符（Operators and delimiters）

```
operator  = "+" | "-" | "*" | "/" | "%"
          | "<" | ">" | "<=" | ">=" | "==" | "<>"
          | "=" | "not" | "and" | "or" | "xor"
delimiter = "(" | ")" | "[" | "]" | "{" | "}"
          | ";" | "," | ":" | "::" | "." | "->" | "=>" | "<-" | "|" | "&"
```

说明：

- `&` 仅出现在**类型**与声明位置（引用，2.6.6），不是表达式运算符；Nova 无取地址运算符。
- `->` 是函数返回类型箭头；`=>` 是 `match` 分支箭头；`<-` 是继承箭头（第 6 章）；`|` 是数组类型中基类与长度的分隔符（2.6.4.2）。
- 逻辑运算**只有**关键字形式 `and` / `or` / `xor` / `not`，无 `&&` `||` `!` `~`。
- **无**位运算（`&` `|` `^` `~` `<<` `>>`），**无** `++` / `--`，**无**复合赋值（`+=` 等），**无**运算符重载，**无** `?:` 三元运算符，**无** `!=`（不等只有 `<>`），**无** `=` 与 `==` 之外的赋值/比较形式。
- 权威规范 §1.6 所列 `type()` 为**类型转换的元记法**（`type` 是占位符，非函数名）；本章将其具体化为**调用形式** `T(expr)`，`T` 为目标类型关键字（2.6.7.2）。

### 2.4.8 运算符优先级与结合性（Precedence and associativity）

下表从上到下优先级**递减**。同一行内运算符优先级相同，结合性如注。

| 优先级 | 运算符 | 结合性 | 类别 |
|---|---|---|---|
| 1 | `( )` `[ ]` `.` `()` 调用 | 左 | 后缀 |
| 2 | `-` `+`（一元）`not` | 右 | 一元 |
| 3 | `*` `/` `%` | 左 | 乘法 |
| 4 | `+` `-`（二元） | 左 | 加法 |
| 5 | `<` `>` `<=` `>=` | 左 | 关系 |
| 6 | `==` `<>` | 左 | 相等 |
| 7 | `and` | 左 | 逻辑与 |
| 8 | `xor` | 左 | 逻辑异或 |
| 9 | `or` | 左 | 逻辑或 |
| 10 | `=` | 右 | 赋值 |

**规定**：

- 同优先级且左结合的运算符链，等价于从左到右加括号：`a - b - c` ≡ `(a - b) - c`。
- 混合优先级必须加括号，否则按本表解析；实现**可以**（但不应）对易错组合（如 `a and b or c`）发出警告。
- 一元运算符优先级高于一切二元算术：`-a * b` ≡ `(-a) * b`；`not a and b` ≡ `(not a) and b`。
- 赋值表达式不能作为另一表达式的操作数参与算术（其值为 `void`，见 2.7.6），故 `a = b = c` 非法（权威规范 §4.1）。
- **`::` 不在本表内**：`qualified_name`（`a::b::c`）在词法/文法层归并为**单个原子项**（2.7.2），故其"优先级最高"（第四章 §7.4 [1]）由构造保证，无需列为独立运算符；`a::b.c` 解析为 `(a::b).c`。

---

## 2.5 基本概念（Fundamental concepts）

### 2.5.1 作用域（Scopes）

Nova 采用**词法作用域**（lexical scoping），四级：

| 作用域 | 引入构造 | 可见范围 |
|---|---|---|
| **包作用域** | 顶层 `function` / `class` / `type` / `const`（顶层无可变 `let`，见 2.5.4） | 整个翻译单元；跨包经 `::`（第 7 章） |
| **文件作用域** | `import` 引入的名字 | 本文件 |
| **函数作用域** | 形参 | 函数体 |
| **块作用域** | `let` 声明、`for` 循环变量、`match` 分支绑定 | 声明处至块结束 `}` |

- 内层作用域的名字**遮蔽**（shadow）外层同名者；被遮蔽者在其作用域内不可见，无 `outer::x` 之类的逃逸语法（`::` 仅用于包限定）。
- 同一作用域内的名字唯一性规则（2026-09-24 按 S-04 兜底修订——原「名字必须唯一」是禁重载时期的遗留句式，与 §2.7.10 直接冲突）：
  - **函数名可按重载共享**（签名 = 参数类型列表，2.7.10）；
  - 类名、`type` 别名、包级 `const` 名各自唯一，且类名不得与任何函数名重名（第四章 §6.1 [2]）；
  - 同名 `let` 创建新绑定并遮蔽前者，已解析的表达式仍指向原绑定（第三章 3.1.2）。
- **前向引用**：块作用域的变量在声明之前不可见（无提升）。函数与类在包作用域内**允许**前向引用（任意顺序声明，两趟解析）。

### 2.5.2 名字与绑定（Names and bindings）

- 每个 `let` 引入一个**绑定**：名字与存储位置的关联一经建立不可改变（无 `new`/`delete`，无重绑定语法）。对值类型，存储位置固定但其**值**可再赋；对类类型，绑定指向的堆对象可经赋值改变（2.6.5）。
- `let` 声明是**语句**而非表达式，无值（语句/表达式分类见 2.7.8）。

### 2.5.3 对象、值与类型

- 每个对象有确定的类型、大小与对齐（2.6.2）。
- 表达式的类型在编译期确定（强类型，无动态类型逃逸）。
- 每个值属于且仅属于一个类型；`void` 类型的唯一值是 `none`。

### 2.5.4 生命周期与存储期（Lifetimes）

| 存储期 | 对象 | 结束时刻 |
|---|---|---|
| **静态** | 字符串/数组字面量、包作用域 `const` | 程序终止 |
| **自动** | 块作用域 `let`（值类型）、形参 | 所在块执行完毕 |
| **动态** | `class` 类型（含 `array` / `string` 的底层存储）对象 | 由运行时管理（见下） |

- 值类型对象存于栈（或寄存器）；类类型对象存于堆，变量持有其引用。
- **包作用域不允许可变变量**：顶层只允许 `const`（编译期常量，见 2.8.6 与第 7 章），不存在包级 `let`（跨章节协调 B-4：顶层可变状态会引入初始化顺序问题，第 7 章"无模块初始化函数"的设计以此为前提）。
- 动态存储期对象的回收策略（引用计数或所有权检查）属实现阶段决定（拷贝语义暂缓的连带项），但规范保证：**任何时刻不存在可观察的悬垂引用**。`let &a = c;` 后 `c` 若先于 `a` 结束生命周期，程序为**移植期错误**（静态可判定时）或 **trap**（静态不可判定时）。

### 2.5.5 求值顺序（Order of evaluation）

**规定（全语言统一，无未指定行为）**：

- 运算符操作数**从左到右**求值（同 C++17）：`f(a, b)` 先 `a` 后 `b`；`x + y` 先 `x` 后 `y`；`a[i] = b[j]` 先 `a`、`i` 后 `b`、`j`。
- 短路：`a and b` 中 `a` 为 `false` 时不求值 `b`；`a or b` 中 `a` 为 `true` 时不求值 `b`；`xor` **不**短路（两侧均求值）。
- 多重赋值 `a, b = c, d;`：右侧全部求值完毕后，再按序赋值（见 2.7.7）。
- 函数体内：语句按书写顺序求值，块的值 = 最后一条表达式语句的值（2.7.8）。

### 2.5.6 值类别（Value categories）

表达式分为两类：

- **左值表达式**：变量、`self` 的字段访问、索引表达式 `a[i]`（其结果即"对元素的引用"，见 2.6.4.2）、引用别名。
- **右值表达式**：字面量、算术/逻辑结果、函数调用结果、块值。

只有左值表达式可作赋值左值。`a + b = c` 为语法错误。

---

## 2.6 类型系统基本模型（Type system: fundamental model）

### 2.6.1 类型分类

```
type = value-type | reference-type | class-type | array-type
value-type     = "bool" | "char" | "int" | "float" | "void"
reference-type = "&" type
class-type     = class-name | "string"
array-type     = "array" "[" type "|" { dimension } "]"
```

- **值类型**：`bool` `char` `int` `float` `void`。赋值即按位复制。
- **类类型**：用户 `class`、内置的 `string` 与 `array`（`array` 归结为类类型，权威规范 §3.1）。类类型变量持有对堆对象的引用，赋值语义见 2.6.5。派生类到基类的**隐式向上转换**合法（第 6 章，跨章节协调 B-5）；向下转换不提供。
- **引用类型** `&T`：不是独立类型，是 `T` 的**别名类型**（2.6.6），只出现在形参、`let &a = c` 与接口返回类型中。
- **函数不是一等值**（跨章节协调 B-3，推翻原 D-45）：`function` 名只能出现在声明与调用位置，**不可**作为实参传递、不可绑定 `let`、不可存入字段或数组。不支持函数值、lambda、闭包。
- 类型集合封闭：除上列之外无其他类型。无元组类型（多重赋值是语法构造而非元组值，2.7.7）、无枚举（✗）、无指针类型（引用替代）。

### 2.6.2 对象表示、大小与对齐

**对齐通例**：每个类型 `T` 的 `alignment(T)` 等于其 `size(T)`（`void` 与 `class` 除外）。对象起始地址必须满足

$$
\operatorname{addr} \bmod \operatorname{align}(T) = 0
$$

| 类型 | size（字节） | align | 备注 |
|---|---|---|---|
| `bool` | 1 | 1 | 只有 0/1 两个合法位模式 |
| `char` | 1 | 1 | ASCII 0–127 |
| `int` | 4 | 4 | 二补数 |
| `float` | 8 | 8 | IEEE 754 binary64 |
| `void` | 0 | 1 | 不占存储 |
| `string` | 16 | 8 | 句柄：`{ ptr: 8, length: 8 }`；底层字节数组另行分配 |
| `&T` | 8 | 8 | 机器字长（64 位平台） |
| `array[T | n]` | $n \times \operatorname{size}(T)$（堆上）+ 16（句柄） | $\operatorname{align}(T)$ | 见 2.6.4.2 |
| `class` | 视成员数量（含 vptr 8 字节） | $\max\big(8,\ \max_i \operatorname{align}(m_i)\big)$ | 含对象头部 vptr，见 2.6.4.3 与第四章 §6.11 [2] |

- 结构体内成员按**声明顺序**排布，相邻成员间插入最小填充使各成员对齐成立；结构体总大小为 `align` 的整数倍（尾部填充）。
- 实现**不得**重排成员顺序（ABI 要求：跨版本二进制兼容，权威规范 §2.2）。

### 2.6.3 值类型表示

#### 2.6.3.1 `bool`

1 字节。位模式 `0x00` 为 `false`，`0x01` 为 `true`；其他位模式**不出现**于合法程序——所有 `bool` 值都由字面量、比较运算结果或已初始化变量赋值产生，不存在产生非法位模式的途径。

#### 2.6.3.2 `char`

1 字节，值域 $[0, 127]$，即 7 位 ASCII 存于 1 字节高位补 0。`char` **不直接参与算术与比较运算**——算术前必须显式写 `int(c)`（跨章节协调 S-03，推翻原 D-29 的自动提升）；赋值、实参匹配、重载排序与 `int → float` 算术提升不受影响（2.6.7.1）。

#### 2.6.3.3 `int`

4 字节，**二补数**（two's complement），值域

$$
-2^{31} \le v \le 2^{31}-1
$$

运算结果超出值域时**不**回绕，而是 **trap**（`NOVA-R-INT-OVERFLOW`，见 2.10.3；跨章节协调 S-02，推翻原 D-27 的 $\operatorname{wrap}_{32}$ 回绕方案）。加、减、乘、整数除法的商以及显式窄化只要越出值域即触发，$-\mathsf{INT\_MIN}$（即 $-2^{31} / -1$）同样 trap。

- 溢出检查是**语言保证**，不是可选的调试行为——合规实现必须使越界可观察（trap），不得静默回绕。
- 需要回绕/饱和语义时由 `math` 包提供显式函数（`math::wrapping_add` 等，第 7 章）。
- 理由：教学语言下"错误必须可见"优先；静默回绕是 C/Java 事故的经典来源，且与 2.10.3 的 trap 哲学一致。

#### 2.6.3.4 `float`

8 字节，IEEE 754 **binary64**：

$$
v = (-1)^{s} \times 1.f \times 2^{e-1023}, \quad s \in \{0,1\},\ 0 < e < 2047,\ f \in [0,1)
$$

- 支持 $\pm\infty$、$\pm 0$ 与 NaN；
- 运算遵循 round-to-nearest-even 默认舍入；
- 浮点比较中 $\mathsf{NaN} \mathbin{<>} x$ 恒为 `true`（IEEE 语义）；
- `float` 无 `-0` 与 `0` 的区分需求：`-0.0 == 0.0` 为 `true`。

#### 2.6.3.5 `void` 与 `none`

- `void` 是**类型**，`none` 是其**唯一值**（权威规范 §1.5）。
- `none` **无运行时表示**：不占存储、无地址、不可参与运算。它是编译期占位值，仅出现在以下语境：
  1. 函数返回类型 `-> void`（或省略返回类型）时，调用表达式的类型；
  2. 赋值表达式的类型（2.7.6）；
  3. 块值为"无意义"时的类型（2.7.8）；
  4. 字面量 `none` 可显式书写，**只能**用于 `return none;`（返回 `void` 的函数中）。
- **规则（阻塞性）**：类型为 `void` 的表达式**不可**绑定到 `let`（`let x = f();` 当 `f` 返回 `void` 时为移植期错误，同 Rust 对 `()` 的宽容不同——Nova 选择严格），不可作实参、不可作索引、不可出现在运算数位置。
- `void` 不可作数组基类（`array[void | n]` 非法）。

### 2.6.4 类类型基本模型

#### 2.6.4.1 `string`

- 内置类类型（权威规范 §2.1）。变量持有 16 字节句柄 `{ internal_storage: ptr, length: int }`；底层字节数组存于堆，UTF-8 编码。
- 接口（`nova-types.yaml`，另含本章补全的 `format`，见下）：
  - 属性 `length`、`internal_storage`；
  - 操作 `length() -> int`、`[index] -> &char`、`[n:m] -> &string`、`format(fmt, args...) -> void`。
- **`length` 属性与 `length()` 方法**：同一访问的两个拼写，`length` 为字段语法糖，等价于 `length()`；二者均返回**字节数** $n_{\text{bytes}}$。
- `[index]`：返回第 `i` **字节**的引用（`&char`）。越界 → trap（2.10.3）。
- `[n:m]`：切片，**左闭右开**，返回 `&string`（共享底层存储的视图，不复制）。$0 \le n \le m \le \text{length}$，否则 trap。
- 字符串**不可变**：`s[i] = 'x'` 为移植期错误（索引结果 `&char` 指向只读存储，不可作左值）。
- 字符串相等比较 `==` / `<>` 为**内置支持**（按字节序列比较）——这是"类类型不参与内置运算"的**唯一例外**，理由：`==` 是等价关系而非算术运算，业界（Go/Rust/C#）普遍将字符串内容相等视为语言级能力（Java 的 `equals` 路线已被广泛诟病）。
- 拼接：`+` **不**用于字符串（遵守"不参与内置运算"）。拼接与格式化经 `uni` 包：`uni::concat(a, b)`（第 7 章）。
- **`format` 方法**（`point.nova:15` 在用）：`s.format(fmt, args...)` 是 `string` 的**内置成员方法**，享有格式化 I/O 的可变尾参豁免（普通函数不支持可变参数，2.7.10；本豁免与第三章 3.4.1"标准库格式化 I/O 可接受异类型可变尾参数"同一条规则，`format` 以内置方法形式提供）。按 `{}` 占位规则把 `args` 格式化后**写入接收者 `s` 自己的缓冲**，表达式类型为 `void`。
  - 前置条件：调用时 `s` 必须处于**未初始化槽位**状态（`let s: string;` 后尚未写入，2.9.1）或为**空串**；`format` 被声明为对该槽位的**写入**（确定性初始化分析据此认定 `s` 已初始化）；向已有内容的 `s` 再次写入为移植期错误——字符串一经产生内容即不可变（本条与"不可变"规则的一致性靠"一次性初始化写入"维持）。
  - 写入成功后 `s` 获得新内容与长度，此后 `s` 进入不可变状态。
  - 这一模型使 `let s: string; s.format(...); return s;`（`point.nova`）合法：`let s: string;` 创建未初始化槽位（2.9.1），`format` 完成初始化写入，`return s` 时读取合法。

#### 2.6.4.2 `array`

- 归结为类类型（权威规范 §3.1）。类型记法：

```
array-type = "array" "[" type "|" expression { "," expression } "]"
```

  `array[T | n]` 为一维，`array[T | n, m]` 为二维，语义 = **基类 + 维度 + 各维长度**。
- **长度是类型的一部分**：`array[int | 9]` 与 `array[int | 10]` 是不同类型（权威规范 §3.1，与 C++ 不同）。
- 形参处允许**省略长度**：`function f(a: &array)` 接受任意长度、任意基类的数组——这是"数组顶类型"（top type）简写，**仅确定用于形参**；`let` 位置是否允许 `&array` 未定（§9.3 第 5 项，留待课堂确认）。省略长度时运行时 `length()` 返回实际长度。
- 类型中的长度 `n` 与运行时 `length`：静态数组中二者恒等；`length` 属性/方法读取的是隐藏存储中的长度字段（与类型信息冗余存放，供 `&array` 形参场景使用）。
- 布局：句柄 `{ internal_storage: ptr, length: int }`（16 字节）+ 堆上 $n_1 \times n_2 \times \cdots \times \operatorname{size}(T)$ 的连续存储，行优先（row-major）。
- 接口：`length() -> int` 返回**所有维度元素总数** $n_1 \times n_2 \times \cdots$（跨章节协调 S-07，推翻原"第一维长度"定义）、`[index] -> &T`。
- `[index]` 返回**元素引用**（左值）：`a[0] = 1;` 修改底层存储本身（与 §2.4 引用语义一致）。
- **多维索引采用逗号全维下标**：`a[i, j]` 一次消费全部维度（`array[T | n, m]` 上 `i < n`、`j < m`）。**不存在部分下标**——`a[i]` 对多维数组是移植期错误，也不产生行视图（统一裁决 C-1：第三章"单下标行主序展平"与协调稿 `a[i][j]` 链式互斥，采逗号下标，语义唯一、越界检查最直接）。
- `array` **不支持** `[n:m]` 切片（跨章节协调 S-07，推翻原 D-24）：切片需要视图类型，而 `array` 无 `&string` 式的现成返回类型；需要子数组时经显式复制函数（第 7 章）。`string` 的 `[n:m]` 不受本条影响（2.6.4.1）。
- 越界（任一维下标为负或不小于该维长度）→ trap。
- 数组字面量 `[1, 2, 3]` 类型推断为 `array[int | 3]`（2.4.6.7）。
- 动态数组 `✗`（权威规范 §3.2）。

#### 2.6.4.3 `class`（对象模型摘要）

- 完整规则见第 6 章。本章仅固定基本模型：
  - 对象 = 成员按声明顺序的连续布局（2.6.2）+ **方法表指针（vptr）**，vptr 置于对象**头部**（8 字节），计入对象大小；
  - 继承时基类子对象置于派生类成员**最前**（单继承链下等价于 vptr + 基类前缀 + 新增成员）；
  - `override` 经 vptr 动态分派；
  - 类类型变量持有堆对象引用，赋值/传参语义见 2.6.5。

### 2.6.5 传递语义（Passing semantics）

| 类别 | 传参 | 赋值 | 返回 |
|---|---|---|---|
| 值类型（`bool` `char` `int` `float`） | 传值（复制） | 按位复制 | 传值 |
| 类类型（`class` `array` `string`） | **传引用**（复制句柄，指向同一堆对象） | 复制句柄（同一对象，别名效果，参照 Python 模型） | **返回引用**（句柄） |
| `&T` 形参 | 绑定实参的存储（别名，无复制） | — | — |

- 类类型赋值 `b = a` 后，`a` 与 `b` 是**同一对象的两个名字**（Python 式引用语义，非 C++ 式值语义）。**独立副本**需显式拷贝操作，拷贝语义暂缓（权威规范 §2.5），第 6 章定义。
- 数组从函数返回时返回**引用**（句柄），对象生命周期由运行时保证（2.5.4）。

### 2.6.6 引用与 `const`

引用是**别名**：「一个单元的不同名字」（权威规范 §2.4）。

```
reference-decl = "let" "&" identifier "=" expression
reference-type = [ "const" ] "&" type | "&" type [ "const" ]
```

**`const` 位置语义**（权威规范 §2.4 标 `?`；2026-09-24 按跨章节协调 S-09 修订，推翻原 D-14 的 C++ 顶层 const 方案——原方案接不住 `inherit.nova:20` 把字面量 `"tiger"` 传给 `&string const` 形参的既有用例）：

| 写法 | 含义 |
|---|---|
| `&T` | 可写引用，只能绑定**稳定且可写的左值**；引用本身不重绑定 |
| `const &T` | **被引用对象只读**：只能绑定稳定左值，经该引用不可写（`a[i] = 1` 移植期错误），不可传给 `&T`（非 const）形参 |
| `&T const` | **调用期间的只读参数视图**：可绑定稳定左值、字面量或临时值；临时对象的生命周期最多延伸到本次调用，不得从函数返回或保存 |

- 三种形状**不产生不同的重载签名**（2.7.10），只影响绑定检查、写权限与生命周期检查。
- 二者**含义不同**（与课堂结论一致）：`const &T` 限制对象写权限；`&T const` 是"可借临时值的只读形参"。
- 转换规则：`&T` 可隐式转为 `const &T`（加只读）；反向非法。
- 裸 `const T`（无 `&`，值类型）：合法，含义为该变量初始化后不可再赋值；类类型上 `const` 等价 `const &T`。
- 引用不能为空（无 null 引用）、不能重新绑定、不能作数组基类；返回局部变量/临时值的引用必须编译期拒绝。

### 2.6.7 转换（Conversions）

#### 2.6.7.1 隐式提升（widening）

编译期自动进行，方向唯一：

$$
\mathsf{char} \hookrightarrow \mathsf{int} \hookrightarrow \mathsf{float}
$$

- `bool` **不参与**任何隐式转换（无 `bool→int`，亦无 `int→bool`——需要判非零写 `x <> 0`，跨章节协调 S-03）。
- **提升的适用范围（2026-09-24 二次修订，S-03/第三章 3.3.4）**：赋值、实参对形参匹配、多重赋值两侧、重载候选排序，以及**算术与比较中的 `int → float`**。
- **被禁止的仅是 `char` 的自动提升**：`char` 参与算术/比较前必须显式 `int(c)`（2.6.3.2，S-03）；一旦转为 `int`，即按 `int` 参与。`int` 与 `float` 混合运算**合法**，`int` 自动提升为 `float`，结果为 `float`（第三章 3.3.4）。
- 注意与 S-15（`if`/`match` 分支完全同型）相区分：**算术/比较**允许 int→float 自动提升，**分支类型统一**不允许——两条规则各自独立，前者是运算数规则，后者是类型统一规则。

#### 2.6.7.2 显式转换

调用语法 `T(expr)`，`T ∈ { char, int, float }`：

| 转换 | 语义 |
|---|---|
| `int(c)`，`c: char` | 数值不变（0–127） |
| `char(i)` | 仅接受 $0 \le i \le 127$，越界 → **trap**（S-03，推翻原"回绕 mod 128"） |
| `float(i)` | 精确转换；$|i| > 2^{24}$ 时按 IEEE 舍入（可能失精，**警告**） |
| `int(f)` | **截断向零**：$\operatorname{trunc}(f)$；$f$ 为 NaN/∞ 或超 `int` 范围 → trap |
| `char(f)` | 先向零截断为 `int`，再按 `char(i)` 检查 0–127，越界 → trap |

- `bool(x)` **不提供**（S-03，推翻原 D-30 的 `int→bool`）：判非零一律写 `x <> 0`。
- `string ↔ int/float`：**无**内置转换，经 `uni` 包（第 7 章）。
- 类类型之间：仅派生→基类**隐式向上转换**（2.6.1），**不提供显式 `type()` 形式**（S-28：类名调用即构造调用，见第 6 章）；其余（类↔值类型、无关类之间）无转换。引用与值类型之间无转换。

#### 2.6.7.3 运算的类型规则

二元算术/关系运算的操作数限 `int` / `float`（`char` 须先显式 `int(c)`，`bool` 不可参与），混合时取提升链上最窄公共类型：

$$
\frac{T_1, T_2 \in \{\mathsf{int}, \mathsf{float}\} \quad U = \max(T_1, T_2) \text{（按 } \mathsf{int} < \mathsf{float} \text{ 序）}}{T_1 \mathbin{\text{op}} T_2 : U}
$$

- `int op int` → `int`；任一侧为 `float` → 另一侧自动提升为 `float`，结果 `float`（第三章 3.3.4）。
- `char` 或 `bool` 出现在操作数位置 → 移植期错误（`char` 先写 `int(c)`）。
- 比较运算（`< > <= >=`）同规则；`==`/`<>` 按 2.7.5。

---

## 2.7 表达式（Expressions）

### 2.7.1 语法总览

```
expression      = assignment-expression
assignment-expression = conditional-expression [ "=" assignment-expression ]
conditional-expression = logical-or-expression      # 无 ?:，if/match 表达式见 2.7.9
logical-or-expression  = logical-xor-expression { "or" logical-xor-expression }
logical-xor-expression = logical-and-expression { "xor" logical-and-expression }
logical-and-expression = equality-expression { "and" equality-expression }
equality-expression    = relational-expression { ( "==" | "<>" ) relational-expression }
relational-expression  = additive-expression { ( "<" | ">" | "<=" | ">=" ) additive-expression }
additive-expression    = multiplicative-expression { ( "+" | "-" ) multiplicative-expression }
multiplicative-expression = unary-expression { ( "*" | "/" | "%" ) unary-expression }
unary-expression       = [ "-" | "+" | "not" ] postfix-expression
postfix-expression     = primary-expression { call-suffix | index-suffix | member-suffix }
call-suffix            = "(" [ argument-list ] ")"
index-suffix           = "[" expression { "," expression } "]"
member-suffix          = "." identifier
primary-expression     = literal | qualified_name | "(" expression ")"
                       | "self" | "super" | array-literal | block-expression
                       | if-expression | match-expression
qualified_name         = identifier { "::" identifier }
```

### 2.7.2 字面量、标识符与限定名表达式

- 字面量的类型与值见 2.4.6。
- 标识符表达式解析到其绑定（2.5.1/2.5.2）；未声明即使用为移植期错误。变量在初始化表达式中不可引用自身（`let x = x;` 非法）。
- **限定名表达式** `qualified_name`（`identifier { "::" identifier }`）是 `primary-expression` 的一种，使 `math::sqrt(x)`、`counter::bump()`、`pkg::module::name` 在词法/文法层可解析（`eqroot.nova:11` 在用）。本章只固定其**语法形状**；`::` 左操作数的种类（包名/模块名/类名）、解析顺序与可见性检查属第 7 章（跨章节协调见第四章 §7.4）。限定名解析失败为移植期错误。

### 2.7.3 括号表达式

`( e )` 求值 `e`，类型与值类别同 `e`。括号用于改变结合顺序，不产生新值。

### 2.7.4 算术表达式

- 二元 `+ - * /`：操作数限 `int` / `float`（`char` 须先显式 `int(c)`；`int` 与 `float` 混合时 `int` 自动提升为 `float`，2.6.7.3）。
  - `int`：结果超出值域 → **trap**（`NOVA-R-INT-OVERFLOW`，2.6.3.3）。
  - `float`：IEEE 754 语义（2.6.3.4）。
- `/`（除法）：
  - `int / int`：**截断向零**的整除（同 Go/C99，非 Python 地板除）；
  - `b = 0` → **trap**（2.10.3）；$-2^{31} / -1$ 商越界 → trap；
  - `float / float`：IEEE 语义（`x/0.0` 得 $\pm\infty$，`0.0/0.0` 得 NaN，**不 trap**）。
- `%`（取余）：仅 `int`，满足恒等式

$$
a = (a / b) \times b + (a \% b)
$$

  故 `a % b` 的符号与 `a` 相同（截断除配套）。`b = 0` → trap。
- 一元 `-`：`int` 取算术负——$-\mathsf{INT\_MIN}$ 越界 → **trap**（与 2.6.3.3 一致，推翻原回绕方案）；`float` 翻转符号位；`char` 不可直接取负（须先 `int(c)`）。一元 `+` 无操作（仅 `int`/`float` 合法）。

### 2.7.5 关系与相等表达式

| 运算 | 操作数类型 | 结果 |
|---|---|---|
| `< > <= >=` | `int` / `float`（混合时 `int` 自动提升，2.6.7.3；`char` 须先 `int(c)`） | `bool` |
| `== <>` | 上述 + `bool` + `string`（按内容） + 类类型（引用同一性） | `bool` |

- 数值比较：`int` 精确；`float` 遵循 IEEE（NaN 与任何值 `==` 为 `false`、`<>` 为 `true`，`<` `>` `<=` `>=` 含 NaN 一律 `false`）。
- 类类型 `==`：比较**引用同一性**（是否同一对象），`<>` 为其否定。`string` 例外：`==` / `<>` 比较**内容**（2.6.4.1）。
- `string` **不支持** `<` `>` `<=` `>=` 排序比较（跨章节协调 A-11/S-08，推翻原"按字节字典序"定义）：无示例需求，排序比较归 `uni` 函数。
- `bool` 仅可 `==` `<>`，不可 `<`。

### 2.7.6 赋值表达式

```
assignment-expression = lhs "=" expression
```

（`lhs` 即 2.5.6 定义的左值表达式：变量、成员访问、索引访问、引用别名。）

- 左值必须为左值表达式（2.5.6）。
- 语义：求值 `rhs`（允许 2.6.7 隐式方向转换到左值类型），写入左值对象。
- **赋值表达式的类型为 `void`，值为 `none`**（权威规范 §4.1）。因此 `a = b = c` 非法（`void` 不可作操作数，2.6.3.5），`let x = (a = 1);` 非法。
- 类类型赋值 = 句柄复制（2.6.5）。`const` 对象、`string` 索引结果、字面量作左值 → 移植期错误。

### 2.7.7 多重赋值（Multiple assignment）

```
multiple-assignment = lhs "," { "," lhs } "=" expression { "," expression }
```

示例（`gcd.nova:3`、`sort.nova:16`）：`a, b = b, a;`、`a[k], a[i] = a[i], a[k]`。

**规则**（权威规范 §9.1 第 4 条标 `?`，本章定义，登记 D-15）：

- 左值个数必须等于右值个数，否则移植期错误；
- 求值顺序：先按序求值全部左值的**存储位置**，再按序求值全部右值，最后按序写入（"先取址、后取值、再写入"，同 Swift 元组交换）；
- 因此 `a, b = b, a` 正确交换；
- 多重赋值是**语句级构造**，类型为 `void`（同单赋值）；
- 左侧各表达式独立按普通表达式规则检查，无"元组类型"参与（Nova 无元组类型）。
- **与多维下标的文法区分（重要）**：多重赋值的左值列表按**顶层逗号**切分；`[ ]` **内**的逗号归属索引后缀（`index-suffix` 是 `postfix-expression` 的一部分，最长匹配优先）。因此 `a[i, j] = v;` 是**单个**多维下标目标的单赋值，`a[i], b = x, y;` 才是多目标赋值——两者由逗号所在的括号层级区分，parser 不应把 `[ ]` 内的逗号当作目标分隔符。

### 2.7.8 块表达式与"一切皆表达式"模型

Nova 的核心语义模型（权威规范 §4.1）：**几乎一切都是表达式**。

```
block-expression = "{" { statement } [ expression ] "}"
```

**块的值**：

- 块以表达式结尾 → 块的类型与值 = 最后一条表达式的类型与值；
- 块以非表达式语句结尾（`let`、`return`、空块 `{}`、仅含声明的块）→ 块的类型为 `void`，值为 `none`；
- 中间位置的表达式语句：求值、副作用保留、值丢弃。

**"几乎"的精确边界**——以下构造是**语句**，无值、不可出现在表达式位置（登记 D-17）：

| 语句 | 形式 |
|---|---|
| 变量声明 | `let x: T = e;` / `let x = e;` / `let &a = c;` |
| 类型别名 | `type Name = T;` |
| 导入 | `import pkg;` |
| 函数/类声明 | 声明语句 |
| 跳转 | `return e?;` / `break;` / `continue;` |

以下构造是**表达式**（可出现在表达式位置）：字面量、标识符、运算、调用、索引、成员访问、块、`if`、`match`、赋值（类型 `void`）。

### 2.7.9 `if` 与 `match` 表达式

语法见 2.8.2、2.8.3。作为表达式使用时：

- `if c { e1 } else { e2 }`：要求两分支块值类型**完全同型，不自动提升**（跨章节协调 S-15，推翻原"公共提升类型"方案）。注意本条是**分支类型统一**规则，与 2.6.7.3 的**算术操作数提升**（`int` 可自动升 `float`）是两条独立规则，互不适用——分支 `int`/`float` 混用仍非法，须写 `float(1)`。
- **无 `else` 的 `if` 不可作表达式**（`let x = if c { 1 };` 非法——登记 D-18。理由：`void` 与 `int` 无法统一，"缺分支得 none"会掩盖逻辑错误）。
- `match` 作表达式：所有分支（含 `else`）必须存在且块值类型**完全同型**；`match` 作表达式时 `else` 分支**强制**。
- 被选中分支的块完整求值，其值即表达式值；未被选中的分支不求值。

### 2.7.10 调用表达式

```
call-suffix = "(" [ expression { "," expression } ] ")"
```

- 函数调用：实参按位置与形参一一匹配（无默认参数、无关键字实参）；
- **允许函数重载**（跨章节协调 S-04，推翻原 D-20 的"包作用域名字唯一"）：同名函数的**签名 = 参数类型列表**，返回类型不参与区分。解析顺序：① 按名字与实参数量筛选；② 全部参数**精确匹配**者优先；③ 无精确匹配时按 `char → int → float` 提升链排序（提升次数更少者优先）；④ 仍有多个同等候选 → 移植期错误（`NOVA-E-OVERLOAD`），不得由实现猜测。`&T` / `const &T` / `&T const` 三种引用形状**不产生**不同签名（2.6.6）。
- 传参语义按 2.6.5；`&T` 形参要求实参为左值且可写性匹配（`const &T` 接受任意 `T` 左值；`&T const` 可接受字面量与临时值）；
- 构造函数调用 `ClassName(args)` 语法合法，规则见第 6 章（`point.nova:27` 在用）；构造函数同样可重载；
- 递归合法，无人为深度限制（栈耗尽 → trap）；
- 求值顺序：实参从左到右（2.5.5）。

### 2.7.11 索引与成员访问

- `a[i]`：`a` 为 `array` / `string`；返回 `&T`（元素引用，左值）或 `&char`（string）。越界 trap。多维 `a[i, j]` 见 2.6.4.2。
- `s[n:m]`（**仅 `string`**）：左闭右开切片视图，见 2.6.4.1；`array` **无切片**（2.6.4.2）。
- `obj.field` / `obj.method(args)`：成员访问；`self` 与 `super` 的规则见第 6 章。
- 包限定 `pkg::name`：见第 7 章。

---

## 2.8 语句（Statements）

### 2.8.1 表达式语句与空语句

```
statement = expression-statement | declaration | if-statement | match-statement
          | while-statement | for-statement | jump-statement | block | empty-statement
expression-statement = expression ";"
empty-statement      = ";"
```

- 表达式语句：求值表达式，丢弃值。类型 `void` 的表达式**只能**以语句形式出现。
- 块 `{ }` 本身是表达式（2.7.8），故 `if c { }` 无需分号。

### 2.8.2 分支语句

```
if-statement = "if" expression block
             | "if" expression block { "elseif" expression block } [ "else" block ]
```

- 条件表达式类型必须为 `bool`（**无真值转换**：`if x`（`x: int`）非法，必须写 `if x <> 0`——登记 D-19）。
- `{ }` 强制（权威规范 §5.1）。
- `elseif` 链在语义上等价于右嵌套 `else { if ... }`，但**不是**块表达式嵌套——各分支块的值直接成为 `if` 表达式的候选值（避免"内层块以 `let` 结尾得 void"的意外）。

### 2.8.3 `match` 语句

```
match-statement = "match" expression "{" { match-arm } [ else-arm ] "}"
match-arm       = pattern "=>" block
else-arm        = "else" "=>" block
pattern         = expression        # 形态（比较模式 / 值模式）由语义判定，见下
```

- 被匹配对象 `obj`：单值、比较表达式或逻辑表达式（权威规范 §5.2）。
- 匹配规则（登记 D-21，将 §5.2 的模糊表述精确化）——**命中判定由模式形态决定，而非 obj 形态**：
  - 模式为**比较模式**（含 `<` `>` `<=` `>=` `==` `<>` 的表达式）：求值模式，结果必须为 `bool`，**为 `true` 即命中**。模式内可自由引用 `obj` 中的变量：`match d { d > 0 => {...} }`（`eqroot.nova:12`）中 `d > 0` 独立求值判真。
  - 模式为**值模式**（字面量、布尔字面量、无比较运算的表达式）：求值模式得 `p`，命中条件 `obj == p`（比较按 2.7.5：`string` 按内容，类类型按同一性；两侧须满足 2.6.7.3 可比较）。
  - 两类模式**不可在同一 `match` 体内混用**（移植期错误）——保证一个 `match` 只表达一种分派意图。
- 分支**自上而下**顺序求值，首个命中分支执行后 `match` 结束；后续分支的模式**不求值**。
- `else` 即 default。`match` 作语句时 `else` 可省；作表达式时强制（2.7.9）。
- 无 `case` 关键字，无模式解构（无 enum、无元组，无可解构对象）。

### 2.8.4 循环语句

```
while-statement = "while" expression block
for-statement   = "for" identifier "in" iterable block
iterable        = "range" "(" expression [ "," expression [ "," expression ] ] ")"
                | array-expr | string-expr | slice
```

- `while`：条件必须 `bool`（同 2.8.2）。先判后执（无 do-while，✗）。
- `range(n)` ≡ `range(0, n)`；`range(n, m)` ≡ `range(n, m, 1)`；`range(n, m, s)`：迭代 $v_k = n + k \cdot s$，$s > 0$ 时取 $v_k < m$、$s < 0$ 时取 $v_k > m$ 的全体 $v_k$（**左闭右开**，与切片一致）。**步长为零 → trap**（`NOVA-R-ZERO-STEP`；若常量求值阶段可证明步长为零，编译期报同一错误码——跨章节协调 S-13，推翻原"字面量 0 一律移植期错误"）。
- `range` 是**语言内建构造**（非普通函数，不返回迭代器对象；`for` 在语法上特判 `range` 调用，编译为计数循环；不可 `let r = range(5)`——登记 D-22，第 7 章 `uni` 表中的 `range` 条目按本条理解）。
- `for x in arr`：按序迭代数组元素（`array`、`string`、切片均可迭代——统一裁决 B-10）；`x` 是元素的**只读副本**：值类型复制值，类类型复制句柄；对 `x` 赋值不改变集合元素，写数组须经 `a[i]`（跨章节协调 S-06，推翻原"只读别名"表述——副本语义与别名语义的唯一差别就在类类型元素的"重绑定 vs 改对象"，取副本）。`for c in "abc"` 按字节迭代。
- 循环变量 `x` 的作用域 = 循环体块；每轮迭代视为新绑定。
- `break`：结束**最内层**一条循环；`continue`：跳到最内层循环的下一轮判定。二者不可跳出 `match`（`match` 非循环）。
- 循环是**语句**，无值（`let x = while ... {}` 非法）。

### 2.8.5 跳转语句

```
jump-statement = "return" [ expression ] ";" | "break" ";" | "continue" ";"
```

- `return e`：结束当前函数，`e` 的类型须与返回类型相同，或沿提升链**加宽**到返回类型（返回类型 `float` 时 `return` 一个 `int` 变量合法；收窄方向必须显式转换——S-29）；`return;` 仅当返回类型为 `void`。
- 返回 `void` 的函数，控制流落到函数块末尾时隐式 `return none`；**非 `void` 函数的每条可达路径必须 `return`**，控制流可能落到函数末尾是移植期错误（`NOVA-E-RETURN`，第三章 3.4.1）。
- `break` / `continue` 必须在循环体内，否则移植期错误。

### 2.8.6 声明语句

```
declaration = "let" [ "&" ] identifier [ ":" type ] [ "=" expression ] ";"
            | "const" identifier ":" type "=" expression ";"
            | "type" identifier "=" type ";"
            | function-declaration | class-declaration
```

- **`let` 推导规则**（登记 D-16）：
  - 有标注 `let x: T = e;`：`e` 检查为 `T`（允许 2.6.7 隐式方向）；
  - 无标注 `let x = e;`：`x` 的类型 = `e` 的类型（从初始化表达式完全推导，含函数返回值、块值、数组字面量）；
  - `let x: T;`（无初始化）：合法，`x` 为**未初始化槽位**，读取前必须通过确定性初始化检查（2.9.1）；
  - `let x;`（两者皆无）：非法。
- **`const`**：绑定只读——初始化后不可再赋值，不要求编译期可求值（编译期常量是包级 `const` 的额外要求，见 2.9.3 与第 7 章）。
- 推导**不**跨语句、不跨函数（无全局推断引擎），仅"从初始化表达式取类型"。

### 2.8.7 函数声明

```
function-declaration = "function" identifier "(" [ parameter-list ] ")" [ "->" type ] block
parameter-list = parameter { "," parameter }
parameter      = identifier ":" type
```

- **返回类型省略时的推导**（跨章节协调 B-2，推翻原"省略 ≡ `-> void`"）：从函数体内所有带值 `return` 推导——全部带值返回**同型**则返回类型为该型；没有任何带值返回则为 `void`。`point.nova` 的 `__constructor__` 无返回类型、`hanoi.nova` 的 `-> void` 显式写法与省略等价。
- 形参类型后置（权威规范 §5.5）。
- `main` 的签名（跨章节协调 S-05，推翻原 D-25 的带参形式；权威规范 §9.1 第 5 条）：

```
function main() -> int
async function main() -> int   # 异步入口，语义见第 8 部分
```

  `main` **不接受形参**（命令行参数经 `uni::args()` 获取，第 7 章）；返回值即进程退出码（0 成功，系统边界取 `mod 256`，2.10.4）；库文件可无 `main`，可执行程序必须有且仅有一个 `main`。

---

## 2.9 变量与初始化（Variables and initialization）

### 2.9.1 未初始化变量与确定性初始化检查

`let x: T;`（无初始化器）合法，`x` 处于**未初始化**状态——不获得任何默认值；**读取前，每条可能到达读取点的路径都必须已写入 `x`**（跨章节协调 S-01，推翻原 D-13 的零值方案：零值会让漏初始化静默通过，而 `input(n)` 式输出参数需要"写入即初始化"的正规机制）。

编译器执行**路径敏感的确定性初始化分析**（同 C# definite assignment / Rust 借用检查的初始化部分）：

- `if` 的所有可达分支都写入，分支后才算已初始化；
- `while` / `for` 体可能零次执行，体内的写入不能证明循环后已初始化；
- `return` / `break` / `continue` 终止的路径不再参与后续分析；
- 读取点无法证明已初始化 → 移植期错误（`NOVA-E-USE-BEFORE-INIT`）。

**写入的认定**：赋值、`input(x)`（`uni` 输出参数原语，声明为"写入实参"）、`string.format`（2.6.4.1，写入空串槽位）算作写入；普通函数调用**不**默认算写入。形参在进入函数时即已初始化。

**类字段**（第 6 章规则，此处摘要）：值类型字段无初值时取**零值**（`false` / `'\0'` / `0` / `0.0`）——字段与局部变量的规则不同，因为对象构造前字段必须处于确定状态（清零成本由构造统一支付）；类类型字段必须有声明初值或在构造函数中赋值，不存在空引用。

| 语境 | 无初始化器时 |
|---|---|
| 局部 `let x: T;` | 未初始化槽位，读取前须通过确定性初始化检查 |
| 类字段 | 值类型字段清零；类类型字段须构造保证 |
| 包级 | 只允许 `const` + 编译期常量初值（2.9.3），无可变包级变量 |

### 2.9.2 初始化与赋值

- `let x = e;` 的 `e` 在声明时刻求值一次；此后 `x` 的存储可变（除非 `const`）。
- 初始化允许引用之前**已初始化**的名字，禁止自引用（2.7.2）。

### 2.9.3 包级常量

包作用域只允许 `const` 声明，且初值必须是**编译期可求值的常量表达式**（字面量、常量名、对上述量的运算与显式转换；不得含函数调用、构造、`await`/`yield`）。常量在链接前完成求值，不存在运行时初始化顺序问题（与第 7 章"无模块初始化函数"的设计互为前提）。

---

## 2.10 移植期错误与运行时错误（Diagnostics and errors）

### 2.10.1 移植期错误（Compile-time error）

下列情形**必须**诊断并拒绝编译（非穷举）：

- 未声明名字的使用；块作用域变量前向引用；
- 类型不匹配且无合法隐式提升（2.6.7.1 适用范围外）；`void` 表达式出现在值位置；
- `if` / `while` 条件非 `bool`；
- 左值要求不满足；`const` 对象被写；
- `if` / `match` 作表达式时分支类型不完全同型，或 `match` 缺 `else`（2.7.9）；
- 重载解析无匹配或有歧义（`NOVA-E-OVERLOAD`，2.7.10）；
- 读取点无法证明变量已初始化（`NOVA-E-USE-BEFORE-INIT`，2.9.1）；
- 非 `void` 函数存在不返回的可达路径（`NOVA-E-RETURN`，2.8.5）；
- `let x;`；字面量 `int / 0`（常量折叠时直接诊断）；
- 关键字作标识符；非法 token；
- 多重赋值左右个数不等。

### 2.10.2 警告（Warning）

实现**应**（但不强制）诊断：`float(i)` 失精、超范围浮点字面量、可证明的死代码、`a and b or c` 类易错优先级组合。

### 2.10.3 运行时错误：trap 模型

Nova 无异常处理（✗）。运行时错误采取 **trap**：

> **trap**：程序立即终止，向 `io` 输出诊断信息（错误类别与源位置），进程退出码为**非零实现定义值**。不执行栈展开、不调用任何清理、不保证缓冲区刷新。

| 事件 | 处理 |
|---|---|
| 数组/字符串索引越界 | trap |
| 切片越界 | trap |
| `int` 除零、`int % 0` | trap |
| `int` 算术溢出（含 $-2^{31}/-1$、$-\mathsf{INT\_MIN}$） | trap（`NOVA-R-INT-OVERFLOW`，2.6.3.3） |
| `int(f)` / `char(i)` / `char(f)` 越界、NaN、∞ | trap |
| `range` 步长为零 | trap（编译期可证明时报同一错误码，2.8.4） |
| 静态不可判定的悬垂引用（2.5.4） | trap |
| 递归过深（栈耗尽） | trap |

**理由**（登记 D-12）：无异常语言中，越界行为只有"UB / trap / 返回哨兵"三选。哨兵破坏类型系统（`a[i]` 返回 `&T`，无哨兵可放），UB 违背"强类型 + 可教"的定位，trap 是唯一与 `&T` 引用语义自洽且实现代价最低的方案（一次边界比较 + `abort`）。

### 2.10.4 `main` 的终止

`return code;`（`code: int`）正常退出，进程退出码取 `code mod 256`。程序亦可经 `uni` 的 `exit` 提前终止（第 7 章）。

---

## 2.11 图灵完备性说明（Turing completeness）

本章模型构成图灵完备的计算核心，构造如下：

1. **无界存储**：`while` 可无界运行 + 类对象（堆分配，第 6 章运行时回收）提供可分配存储（内存耗尽为 trap，不限制理论能力）；
2. **条件**：`if` / `match`（2.8.2、2.8.3）；
3. **循环**：`while`（2.8.4）；
4. **数据**：`int` 提供任意大整数的**编码基础**——以 `array[int | n]` 表示 bignum，逐位运算；
5. **子程序**：`function` + 递归（`gcd_递归.nova`、`hanoi.nova` 已示范）。

由 (1)–(5)，Nova 可模拟任意图灵机：带存储于 `array[int | n]`，头位置于 `int`，状态转移于 `match`，主循环于 `while`。

**边界声明**：`input` / `print` / `println` 是 `uni` 包提供的 I/O 原语（第 7 章），不参与完备性论证；`async` / `await` / `yield` 的协作式并发语义由第 8 部分定义，并发不扩展可计算性。

---

## 2.12 本章语法汇总（Summary: grammar of Chapter 2）

```ebnf
(* 词法 *)
token        = identifier | keyword | literal | operator | delimiter
identifier   = ( letter | "_" | "$" ) { letter | digit | "_" | "$" }
comment      = "#" { non-newline } newline
literal      = boolean-literal | int-literal | float-literal
             | char-literal | string-literal | "none" | array-literal

(* 类型 *)
type            = value-type | reference-type | class-type | array-type
value-type      = "bool" | "char" | "int" | "float" | "void"
reference-type  = [ "const" ] "&" type | "&" type [ "const" ]
array-type      = "array" "[" type "|" expression { "," expression } "]"

(* 表达式 *)
expression                = assignment-expression
assignment-expression     = conditional-expression [ "=" assignment-expression ]
                          | multiple-assignment
multiple-assignment       = lhs "," { "," lhs } "=" expression { "," expression }
lhs                       = identifier | postfix-expression "." identifier
                          | postfix-expression "[" expression { "," expression } "]"
logical-or-expression     = logical-xor-expression { "or" logical-xor-expression }
logical-xor-expression    = logical-and-expression { "xor" logical-and-expression }
logical-and-expression    = equality-expression { "and" equality-expression }
equality-expression       = relational-expression { ( "==" | "<>" ) relational-expression }
relational-expression     = additive-expression { ( "<" | ">" | "<=" | ">=" ) additive-expression }
additive-expression       = multiplicative-expression { ( "+" | "-" ) multiplicative-expression }
multiplicative-expression = unary-expression { ( "*" | "/" | "%" ) unary-expression }
unary-expression          = [ "-" | "+" | "not" ] postfix-expression
postfix-expression        = primary-expression { call-suffix | index-suffix | member-suffix }
primary-expression        = literal | qualified_name | "(" expression ")" | "self" | "super"
                          | block-expression | if-expression | match-expression
qualified_name            = identifier { "::" identifier }

(* 语句 *)
statement = expression-statement | declaration | if-statement | match-statement
          | while-statement | for-statement | jump-statement | block | ";"
declaration = "let" [ "&" ] identifier [ ":" type ] [ "=" expression ] ";"
            | "const" identifier ":" type "=" expression ";"
            | "type" identifier "=" type ";"
            | function-declaration | class-declaration
if-statement    = "if" expression block { "elseif" expression block } [ "else" block ]
match-statement = "match" expression "{" { pattern "=>" block } [ "else" "=>" block ] "}"
while-statement = "while" expression block
for-statement   = "for" identifier "in" iterable block
iterable        = "range" "(" expression { "," expression } ")" | array-expr | string-expr
jump-statement  = "return" [ expression ] ";" | "break" ";" | "continue" ";"
function-declaration = "function" identifier "(" [ parameter-list ] ")" [ "->" type ] block
```

---

## 2.13 示例（Examples）

### 2.13.1 词法与优先级

```nova
let a = 1 + 2 * 3;               # a = 7
let b = not (1 <> 2) or 3 > 4;   # b = false
let c = int('a') + 1;            # c: int = 98（char 须显式转 int，2.6.3.2）
let d = 7 % -2;                  # d = 1（符号随被除数）
let e = int(3.9);                # e = 3（截断向零）
let f = 1_000 + 0x10;            # f = 1016
let g = 2147483647 + 1;          # 运行期 trap（NOVA-R-INT-OVERFLOW，常量折叠期即诊断）
```

### 2.13.2 块值与表达式化分支

```nova
let n = 5;
let parity = if n % 2 == 0 { "even" } else { "odd" };  # parity: string（两分支同型）
let sign = match n {          # 比较模式：各分支条件独立求值判真（2.8.3）
    n > 0  => { 1 }
    n == 0 => { 0 }
    else   => { -1 }
};   # sign: int
# let x = if c { 1 } else { 2.0 };   # 非法：分支不完全同型（2.7.9）
# let nothing = { let x = 1; };      # 非法：块值为 void，void 不可绑定 → 移植期错误
```

### 2.13.3 引用与 const 三形状

```nova
function bump(a: &int) -> void { a = a + 1; }        # 可写引用：实参须为可写左值
function peek(a: const &int) -> int { return a; }    # 对象只读：经 a 写为移植期错误
function show(s: &string const) -> void {            # 只读参数视图：可绑字面量/临时值
    println("{}", s);
}

let v: int = 1;
let &alias = v;      # alias 是 v 的别名
bump(alias);         # v == 2
peek(v);             # 2；peek 内不可写
show("tmp");         # 合法：&T const 接受字面量（临时值活到调用结束，2.6.6）
```

### 2.13.4 多重赋值与迭代

```nova
let arr: array[int | 3] = [3, 1, 2];
arr[0], arr[2] = arr[2], arr[0];      # [2, 1, 3]
for x in range(0, arr.length()) {     # x 是只读副本（2.8.4）
    if x == 1 { continue; }
    print("{}", arr[x]);
}
for e in arr { print("{}", e); }      # array 直接迭代（B-10）
```

### 2.13.5 确定性初始化

```nova
function demo(read: bool) -> int {
    let x: int;            # 未初始化槽位
    if read { x = 1; }     # 仅一条分支写入
    return x;              # 移植期错误：NOVA-E-USE-BEFORE-INIT（2.9.1）
}
```

---

## 附：与权威规范（`nova-spec.md`）的对应

| 本章 | 权威规范 |
|---|---|
| 2.4 词法 | §1 |
| 2.5 基本概念 | §2.1–2.4（部分）、§4.1 |
| 2.6 类型模型 | §2.1、§2.2、§2.3、§2.4、§2.6、§3 |
| 2.7 表达式 | §4.1、§1.6 |
| 2.8 语句 | §5 |
| 2.9 初始化 | §9.1 第 6 条（已定义） |
| 2.10 错误模型 | §9.2 第 8 条（已定义） |
| 2.11 完备性 | ——（本章新增） |

所有**新增定义**（未见于权威规范、由本章按业界最佳实践补全者）逐条登记于 `nova-ch2-补充定义说明.md`。
