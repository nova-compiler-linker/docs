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
| `float` | `1.5`（**16 位，无 double**） |
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

### 2.2 内存与 ABI `✔`

- **默认对齐**
- 关注**二进制兼容**：跨版本、跨架构
- 链接期引入，**必须携带符号表**

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

阻塞规范定稿的：

1. **`const &T` 与 `&T const` 各自的确切语义**
2. **`::` 与 `import` 的完整语法**（包名解析、`uni` 组织方式）
3. **`<-` 继承的完整规则**：继承处的 `public` 含义、是否多重继承
4. **元组赋值** `a, b = b, a;` 的语义
5. **`main` 签名与返回值语义**
6. **未初始化变量的默认值**

暂不阻塞的：

7. `static` 成员语法（关键字存在但无用例）
8. `self` 何时必须写
9. `async` / `await` / `yield` 的线程模型
10. `$` 前缀形参是规则还是约定
11. `array` 能否省略长度参数
12. 自定义类型如何比较（运算符重载已排除）
13. `nvp` 现场编译 C++ 的 ABI 细节
14. 拷贝语义（已明确暂缓）

---

## 附：文件

- **本文件** = 权威规范
- `01-语言设计.md` = 课堂过程记录（追溯用）
- `../nova/nova-bnf.md` = **阶段性仓库，过渡产物，勿全盘吸收**
- `../nova程序示例/` = 示例程序
