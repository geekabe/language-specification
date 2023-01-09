

# Geekabe Language

> Geekabe 语言


## 词法

词符（token）使用 `javascript` 语言的正则表达式表达，且文档里从上到下先定义的词法有更高的优先级。使用带状态转移的词法描述。初始状态为 default mode，即一个源码文件第 0 个字符的初始状态。

#### default mode

```
KEYWORD :（完整的关键字列表见下文）
OPERATOR :（完整的符号列表见下文）
ID : [a-zA-Z$]([a-zA-Z0-9$_])*
NUMBER : INT | FLOAT
INT : [0-9]+
FLOAT: (([0-9]+(\.[0-9]*)?) | (\.[0-9]+))(e[+-]?[0-9]+)?
BOOL : true | false
STRING_QUOTE: ' | "
COMMENT : SINGLE_LINE_COMMENT | MULTI_LINE_COMMENT
SINGLE_LINE_COMMENT : \/\/[^\n]*
MULTI_LINE_COMMENT : \/\*.*\*\/
```

该模式下，遇到 STRING_QUOTE，跳转到 string mode

#### string mode

```
EXPRESSION_START : ${
STRING_QUOTE : ' | " 
STRING : .*
```

该模式下：

* 遇到 EXPRESSION_START，跳到 string-expression mode
* 遇到 STRING_QUOTE，结束 string mode，回到 default mode

#### string-expression mode

```
EXPRESSION_END : }
STRING_QUOTE: ' | "
...(default mode)
```

该模式下：

* 遇到 STRING_QUOTE，跳到 string mode（递归进入）
* 遇到 EXPRESSION_END，结束 string-expression mode，回到 string mode

### 关键字 & 符号

关键字：

`if` `loop` `of` `var` `use` `pub` `inner` `private` `fn` `return` `u8` `u16` `u32` `u64` `uint` `int` `i8` `i16` `i32` `i64` `bool` `str` `type` `partial`

符号：

`>=` `<=` `==` `>>` `<<` `##` ``` `` ``` `#{` `${` `}` `:` `(` `)` `_` `.` `>` `<` `=` `+` `-` `*` `/` `%` `[` `]`

## 文法

文法由且只由以下元素构成：

* 大陀峰命名单词，代表文法规则（及子规则），例如 `Body`, `FunctionDecl`。
* 全大写字母（中间可加下划线），代表词符（token），例如 `ID`。
* 全小写字母，代表关键字，例如 `if`, `loop`。
* 单引号包裹的字符，代表文法中的字符串符号，例如 `'='`, `'>='`。
* 不被单引号包裹的字符，用于描述文法本身的逻辑关系，例如 `=`, `;`。

其中，用于描述文法本身的有效字符包括：

```
=   为（表明接下来是当前规则的构成细节）
|   或（多个子规则中 N 选 1）
()  分组（用于表达关系）
?   可选（出现 1 次或 0 次）
+   重复（出现至少 1 次）
*   重复（出现 0 到 n 次)
;   结束（表示当前规则的描述结束）
#   注释（用于解释文法规则，仅支持单行注释）
```

### Source

代码的存储单元是文件，每一个文件内容的整体，定义为 `Source` 节点。

```
# imports 语句必须整体放在文件顶部。
Source = ImportStmt* StmtList? ;
Block = '{' StmtList? '}' ;
StmtList = Block | Stmt;
```
### ImportStmt

```
ImportStmt = import IMPORT (as ID)? ';'?
```

### Stmt

```
Stmt = DeclStmt | ProtoStmt | IfStmt | SwitchStmt | LoopStmt | Expr;
```

* `DeclStmt` 声明语句，包括类型声明、变量声明、函数声明。
* `ProtoStmt` 原型定义语句。
* `IfStmt` 和 `SwitchStmt` 条件语句。
* `LoopStmt` 循环语句。
* `Expr` 表达式语句。

### IfStmt & SwitchStmt

```
IfStmt = if Expr Block (else if Expr Block)* (else Block)? ;

# 和 go 语言相同，switch case 默认不穿透（默认 break），除非使用 continue 关键字
SwitchStmt = switch Expr '{' (case Expr ':' Block (continue)? )* (default ':' Block)? '}' ;
```

### LoopStmt

```
LoopStmt = loop (LoopRange | LoopIter | LoopIf )? Block ;
LoopRange = ID of Expr? '..' Expr;
LoopIter = ID of Expr;
LoopIf = if Expr ;
```

循环语句涵盖了传统语句的 `for`, `while`, `do` 等表达式。

示例：

```
// 无限循环，loop 后不跟任何语句。
// 等价于 c 等语言的 do while true，可通过 break 退出
loop {}

// .. 符号代表循环某个范围，.. 符号两边必须是整数类型。
// .. 左边整数代表循环的起始数字（包含该数字），可省略，默认为 0
// .. 右边的整数代表循环的终止数字（包含该数字）。
// 以下代码等价于 c 语言中的 for (int n = 0; n <= 10; n++)
loop n of ..10 {}
loop n of 0..10 {}

// 以下代码等价于 c 语言中的 for (int n = 2; n <= 10; n++)
loop n of 2..10 {}

// 第二个 .. 符号代表循环的步进值，以下代码等价于 c 语言中的 for (int n = 2; n <= 10; n+=2)
loop n of 2..10..2 {}

// of 关键字后如果不是 .. 符号代表进行范围循环，则必须是一个数组类型的变量，遍历该数组
var arr = [0, 0, 0, 0];
loop n of arr {
  log(n);
}
// 参数后还可以再跟一个索引参数
loop n, i of arr {
  log(`${n} ${i}`);
}

// loop 后紧跟 if 关键字，则可以指定一个 bool 表达式控制循环。
var i = 0
loop if i < arr.length {
  log(arr[i])
  i += 1
}
```

### DeclStmt & ProtoStmt

```
DeclStmt = TypeDeclStmt | VarDeclStmt | FuncDeclStmt ;

VarDeclStmt = var ID (':' Type)? '=' Expr ';'? ; # 变量的类型可以省略

FuncType = '(' FuncArgs ')' '=>' (Type | void) ;
FuncArgs = (ID ':' Type ','?)* ;
FuncDeclStmt = func ID '(' FuncArgs ')' Block ;

ProtoStmt = proto ID '{' (AtProto | ID) '(' FuncArgs ')' Block)* '}' ;
AtProto = '@' (plus | minus | multiply | divide | to);
```

* 函数参数允许重载。
* `@` 符号打头的原型用于运算符重载和类型转换。

示例：

```
type Person {
  name: str
  age: u8
}
proto Person {
  new(age: u8, name: str) {
  }
  @from(age: u8) {
    return Person(name: 'unknown', age)
  }
  @to(u8) {
    return this.age
  }
  @to(str) {
    return #{strigify('name', 'age')}
  }
}
type I32 = int32
proto I32 {
  @to(Person) {
    return Person(name: 'unknown', age: this)
  }
}

var age: I32 = 18;
var person = Person(age);
var person = Person.from(18);
console.log(person); // "{ name: 'unknown', age: 18}"
console.log(u8(person)); // "18"
```

#### AssignStmt 

```
AssignStmt = ID '=' Expr ';'? ;
```

#### TypeDeclStmt

```
TypeDeclStmt = type ID '=' Type ';'? ;

Type = (PrimaryType | EnumType | StructType | FuncType | ID) ('[' ']')* ;

PrimaryType =
		'int' | 'i8' | 'i16' | 'i32'
	| 'uint' | 'u8' | 'u16' | 'u32' | 'bool'
	| 'str' 
	;

EnumType = '{' EnumField + '}' ;
EnumField = ID (':' NUMBER)? ','? ;
StructType = '{' StructField + '}' ; # 不支持 {} 这种空结构
StructField = ID ':' (PrimaryType | ID) ','? ;
```

### Expr

```
Expr =  ID | '-'? NUMBER | 
	AssignExpr | FuncCallExpr | BoolExpr | TypeConsExpr |
	'(' Expr ')' |
	Expr ('&' | '|') Expr |
	Expr ('>>' | '<<') Expr |
	Expr ('*' | '/') Expr |
	Expr ('+' | '-') Expr
	;
BoolExpr = Expr ((('>' | '<') '='?) | '==' | '&&' | '||') Expr ;
AssignExpr = ID ('+' | '-' | '*' | '/' | '&' | '|' | '>>' | '<<')? '=' Expr ;
FuncCallExpr = ID '(' (Expr ',')* ')' ;     # 函数调用表达式
TypeConsExpr = ID '(' (ID (':' Expr)?)* ')' ;    # object type 的构造表达式
```