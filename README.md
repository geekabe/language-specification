# Geekabe Language Specification

> Geekabe 语言标准（语言文法和相关说明）

`注意：此仓库只是记录想法和设计，Geekabe 语言的编译器当前还没有编码实现！`

Geekabe 是一门面向嵌入式领域，用于微控制器（MCU）的编程语言。其目标是通过提供更简单的高级语言，以及和该语言配套的现代化开发环境，实现嵌入式开发的门槛降低和效率提升。

考虑到这一目标过于宏大，当前（初代版） Geekabe 语言的目标场景只限于“玩具(Demo)场景”，比如电子玩具、少儿创客、原型等；不会考虑性能或生产环境使用的诸多细节；且暂时只支持 STM32 芯片。

Geekabe 语言规划的核心特点包括：

* 带垃圾回收和异步IO的静态编译型语言，语法简单，上手门槛低。
* 现代化的浏览器在线IDE开发环境（串口通讯也直接借助Chrome的串口协议实现）。
* 脱离物理芯片，纯软件面的代码模拟执行和调试能力（即在浏览器上把静态语言作为动态脚本直接运行和调试）。

目录导航：

* Geekabe 语言文法：[Geekabe Language.md](Geekabe%20Language.md)
* Geekabe 宏语言：[Geekabe Macro Language.md](Geekabe%20Macro%20Language.md)

## 符号规则

* 符号是指变量名、函数名、结构类型名、属性名以及类型别名。
* 符号只能由字母、数字和下划线组成，且不能以数字开头。
* 符号名不能是关键字（未来可考虑像 rust 或 typescript 那样更灵活地支持关键字做为符号名）。

## 变量声明

变量使用 `var` 关键字声明，且不区分常量还是变量。

> 关于不引入常量的考量：js/ts 语言中的 const 是不完备的常量，声明为 const 的变量内部属性仍然可变。rust 语言的常量是完备常量，但会使得常量变量的概念复杂，使用起来心智负担不轻（rust甚至需要引入 RefCell 这种内部可变的概念）。因此 geekabe 语言不引入常量，但编译器可以在一定程度上自动分析出未变更的变量，进行常量优化。

变量的类型可通过 `:` 指定，如果不指定则自动推导。整数默认为 `i32`，浮点默认为 `f32`。

```ga
var n0 = 10; // i32
var n1 = 10.; // f32
var n2:i8 = 0x10;
var n3:u16 = 0b11011;
var n3 = i8(32); // i8
var b0 = true; // bool
var b1:bool = false;
var s0 = "hello"; // str
var arr0 = [0, 0, 0]; // i32[]
var another_arr: u32[] = [0xff, 0xffff, 0xfffff];
```

可以直接使用 `Type(literal)` 的语法来初始化指定类型的变量。这个语法和类型转换的语法很一致，区别在于前者括号里是字面量常量，后者括号里是另一个变量名。详见后文的类型转换。

## 结构类型

使用 `type` 关键字定义复杂数据结构类型，类似于其它语言的 struct/class/interface 的概念，但进行了整合和简化。类型名必须使用大驼峰。

```ga
type Boy {
  name: string;
  age: u8
  say_hello: fn() {
    return 'Hello, everyone. My name is', me.name + `. I\'m {me.age} years old.`;
  }
}

var boy: Boy = { name: 'geekabe', age: 18 };
var boy3 = Boy({ name: 'geekabe', age: 18 })
var boys: Boy[] = [{...}, boy];
```

### 成员属性

结构类型的成员（包括成员函数），可以在定义类型时指定默认值，也可以在实例化类型时指定值（或覆盖默认值）。

```ga
type Boy {
  name: 'default', // 定义默认值和类型
  age: i8(10), // // 定义默认值和类型
  fa: fn() { return 'hello' + me.name } // 定义默认值和类型
  fb: fn() -> str, // 只定义类型
  kk: str   // 只定义类型
}
fn some_fn(me: Boy) {
  return 'world' + str(me.age)
}
fn another_fn(me: Boy) {
  return 'world2' + str(me.age)
}
fn wrong_fn() { return 'wrong!' }
var b1: Boy = { name: 'a', fb: some_fn };
var b2: Boy = { fa: some_fn, fb: some_fn }
var b3: Boy = {
  fb: fn() {
    return me.name
  }
}
b1.fa = another_fn;
b2.fa = b2.fb = another_fn;
b2.fa = wrong_fn; // 编译错误，fa 的签名和 wrong_fn 签名不匹配。
```

在成员函数中，通过 `me` 关键字，可以访问该类型的实例本身，类似于 `javascript` 语言的 `this`，`rust` 语言的 `self`。

我们约定，直接在类型定义或初始化类型实例时指定的成员函数，可以直接使用 `me`，相当于编译器会自动增加一个函数参数 `me`。

但如果要将在其它地方定义的函数（比如上述代码的 `another_fn` 函数定义在文件顶部）赋予某个结构类型的成员属性，则需要主动将 `me` 指定为函数的第一个参数。

### 类型继承

```ga
type Person {
  name: str,
  say: fn() {
    return 'Hello, My name is {me.name}.';
  }
}
type Animal {
  age: i8,
  say: fn() {
    return 'Woooo!';
  }
}
type Boy = Person & Animal & {
  say: fn() {
    return '{Person.say()} I'm a boy. {Animal.say()}';
  }
}
fn say_fn(me: Boy) {
  return '{Person.say()} I'm a boy. {Animal.say()}'
}
```

`geekabe` 语言的类型继承试图吸收 `rust` 等新派语言对于代码复用的思路，不引入标准面向对象语言的`类`，`构造函数`，`继承` 等概念，而是采用 `组合` 的思路。

借鉴 `typescript` 的语法， `geekabe` 的类型继承直接使用 `&` 符进行类型组合。`&` 符号连接的最后一项，是当前类型的定义，称为子类型；前面的一或多个项必须是已经声明的类型名，称为继承类型。

* 子类型以及多个继承类型之间，同名成员属性必须有相同的字段类型（如果成员属性是函数类型，则必须具备相同的函数签名）。
* 多个继承类型如果有同名成员属性，则子类型也必须定义该同名属性（否则编译器无法知道该调用哪个继承类的成员）。

### 静态函数

`geekabe` 语言不支持在结构类型上定义静态函数。可以在定义该结构的文件中，直接对外导出函数。例如：

```ga
// boy.ga
pub type Boy {
  name: str, // 类型为 str，没有默认值
  age: 18 // 默认值为 18，类型为 int
}
pub fn create_by_name(name: str) {
  return Boy({ name })
}

// main.ga
use ./boy
var b1 = boy.create_by_name("geekabe")
```

### 代码拆分

类似于 c# 语言的 partial 关键字，当类型复杂需要拆分到不同文件以便管理时，可以使用 `partial` 关键字扩展类型。例如：

```ga
// boy/boy.ga

type Boy {}

partial use ./name.ga
partial use ./age

// boy/name.ga
partial type Boy {
  name: str;
  say_hello: fn() {
    // boy/age.ga 中定义的 age 也可以在当前文件使用，因为都是定义在 Boy 这个类型上的原型。
    return 'Hello, everyone. My name is {this.name}, I\'m {this.age} years old.';
  }
  ~to str: fn() {
    // #{} 这个语法参看后文【宏语言】章节
    return #{ga.json_stringify({ name: 'str', age: 'i8' })}
  }
}

// boy/age.ga
partial type Boy {
  age: i8;
}

```

使用 `partial` 关键字有如下约束：

* `partial` 后一定是 `use` 或 `type` 关键字，分别代表引入扩展文件，以及扩展类型的定义。
* `partial` 关键字和 `use` 以及 `type` 关键字一样，都只能在文件顶部使用。
* 一个文件一但使用了 `partial type` 来扩展类型，则该文件不能被其它文件直接 `use` 使用，只能通过 `partial use` 来引入。换句话讲，可以认为该文件并不独立存在，只是将源码合并到了有 `partial use` 语句的原始文件中。
* 单个文件中，只能出现一次 `partial type` 语句，也就是只能扩展一个类型。同时，该文件只能被最多一个原始文件通过 `partial use` 引入。

### 类型匹配

在 `javascript` 等动态语言中，某个变量的类型可以任意切换（所谓弱类型）。在 `rust` 等静态语言中，一个变量只允许有一种类型，但引入了`多态`等概念，来更灵活地处理继承或组合关系的类型。

`geekabe` 语言的类型匹配规则，主要借鉴自 `typescript` 语言。某个变量如果是基础类型，则不允许切换类型；如果是结构类型，则采取“蕴含”原则：能“蕴含”当前变量的结构的类型都可以被当作该类型切换。

```ga
type A {
  name: str
}
type B {
  name: 'hello', // 指定默认值，类型为 str
}
type C {
  name: str('hello'),
  age: i8(90)
}
let a = A({ name: 'a' }); // a 变量的明面类型是 A，实质类型是 { name: str }
let b = B();
a = b; // 编译可通过，b 变量的实质类型也是 { name: str }
a = C(); // 编译可通过，类型 C 的实质类型是 { name: str, age: i8 }，完全【包含/涵盖/蕴含】了 A 类型的实质类型。
console.log(a.name);
console.log(a.age); // 编译不通过，虽然在内存中，a 变量实际的数据是 C 结构的实例数据（拥有 age 字段），但 a 变量的明面类型是 A，没有 age 字段。
```

对于数组而言，遵循同样的原则。

```ga
type Person {
  name: str,
  say: fn() {
    return 'My name is {me.name}.'
  }
}
type Boy = Person & {
  name: 'Bob',
  say: fn() {
    return '{Person.say()} I\'m a boy.';
  }
}
type Girl = Person & {
  name: 'Alice',
  say: fn() {
    return '{Person.say()} I\'m a girl.';
  }
}
type Alien {
  name: 'alien',
  planet: 'Mars',
  say: fn() {
    return 'I\'m alien'
  }
}

let persons: Person[] = [];
persons.push(Boy());
persons.push(Gir());
persons.push(Alien()); // 编译可通过，Alien 的实质类型蕴含了 Person 的实质类型
loop person of persons {
  console.log(person.say())
}
```

此外，函数参数也遵循这个原则。能蕴含函数参数的本质类型的值都可以传递给函数：

```ga
fn fa(person: Person) {
  console.log(person.say());
}
fa(Alien({ name: 'alien2' })); // 编译合法
```


## 类型别名

使用 `type` 关键字还可以进行类型的别名化。例如：

```ga
type MyInt = int;
type MyFn = fn(i8, str) -> MyInt
```

## 类型转换

类型转换的语法格式为 `TargetType(variable)`，基础类型之间的转换由编译器内置实现，复杂结构类型可以通过 `~to` 和 `~from` 特殊标记成员函数实现。

```ga
var n0 = 10;
var f0 = f32(n0);
var n1 = i16(f0);

type Girl { name: str; age: i8; }
type Boy {
  age: i8;
  name: str;
  ~to i8: fn() { return this.age };
  ~to str: fn() { return this.name };
  ~to Girl: fn() { return Girl({ name: this.name, age: this.age })}
  ~from Girl: fn(girl: Girl) {
    return Boy({ name: girl.name, age: girl.age });
  }
}

var b0 = Boy({ name: 'xiaoge', age: 18 });
var g0 = Girl(b0);
var b1: Boy = Boy(g0);
```

注意默认情况下，`geekabe` 语言的类型转换是重新分配内存的转换。直接在原内存地址上的强制类型转换（通过内存指针的强制转换）暂时不支持。


## 模块管理

和 `nodejs` 类似，`geekabe` 语言的代码模块以 `包（package）` 的形式组织和管理。一个 `包` 也可以理解为一个`项目（project）`，对应文件系统上的一个目录。

多个在文件系统上并列的包，构成工作空间（workspace）。但工作空间只是多个包构成的抽象概念，本身并不是一个文件系统目录。

每个包目录下，一定有 `src` 目录，所有代码都要放在该目录下。但模块之间引用时，不需要再书写这个目录。

`geekabe` 语言也提供了类似 `nodejs` 的 `npm` 包发布和管理平台，称为 `gapm`(geekabe package manager)。

包名约束为只能使用小写字母和下划线。

### 模块引用

模块引用和 `nodejs` 类似。

引用 `gapm` 上的包，直接使用包名：

```ga
use console   # 等价于 use console/main.ga，实际引用的是 console/src/main.ga
use console/some_dir           # 引用 console/src/some_dir 下的所有符号
use console/some_dir/some_file # 引用 console/src/some_dir/some_file.ga 文件里的符号
use console/some_file          # 引用 console/src/some_file.ga 文件里的符号
```

引用文件系统上的文件，必须指定相对或绝对路径：

```ga
use ./some_dir
use ../../some_dir/some_file.ga
use /home/projects/package_a/src/some_file  # 注意引入绝对路径时，src 目录不能省略。
```

使用 `~me` 打头，代表从当前包的 `src` 目录下引入文件：

```ga
use ~me/some_dir/some_file    # 引用 package_a/src/some_dir/some_file.ga 文件
```

使用 `~` 打头，代表从和当前包在文件系统上并列的某个包的 `src` 目录下引入文件：

```ga
// 假设当前包是 package_a，和它并列的一个包 package_b
use ~package_b      # 引用 package_b/src/main.ga
use ~package_b/some_dir/some_file  # 引用 package_b/src/some_dir/some_file.ga
```

引用模块后，可以通过引入的命名空间访问模块。直接引用包的，命名空间是包名；引用文件(夹)的，命名空间是引用的这个文件(夹)名（不包含父目录路径和文件扩展名）:

```ga
// 命令空间就是当前包名
use xxx;  xxx.some_symbol;
use ~package_a;  package_a.some_symbol;
// 命名空间是文件夹或文件名
use xxx/a/b; b.some_symbol;
use ./aa/bb; bb.some_symbol;
use package_a/kk.ga;  kk.some_symbol
```

还可以通过冒号显式地指定要引入模块内的某个符号（多个符号用逗号隔开）。

```ga
use xxx: some_symbol
use ~package_a: some_symbol, another_symbol
```

没有设计成 `use { some_symbol } of xxx` 的原因是，书写代码都是从左到右，先表明要用的包，再指定包内要引用的符号，IDE 才能更顺畅地进行代码提示。像 `typescript` 的语法最大的问题就是，要先写 `use {} from xxx`再把光标移到大括号里，才有代码提示。

如果引用的文件(夹)路径含有空格或冒号，可以用引号将路径包裹。如果文件(夹)名不满足变量名/包名的规则，则需要使用 `as` 关键字主动指定引入模块的命名空间。显然，`as` 和冒号不能同时在一行模块引用代码里出现。

```ga
use 'package_a/hello world/bb';
use 'package_a/helo:world/bb';
use package_a/aa+bb/bb;
// 上述三行代码引入的模块命名空间都是 bb
console.log(bb.some_symbol);
use package/dsd+32 as xxx;
use package/dsd+32: some_symbol
// use package as xx : some_symbol   // 编译错误
```

### 可见度

`geekabe` 模块可见度包括三层：`私有可见`，`内部可见`，`公共可见`，分别可以使用关键字 `private`, `inner`, `pub` 来指定。默认情况下，所有声明的符号是`内部可见`，即 `inner` 关键字可以省略 。

* `私有可见`：只在当前文件，或通过 `partial` 关联的扩展文件中可见。
* `内部可见`：在当前包中可见。
* `公共可见`：可被其它包可见。

假设一个工作空间（workspace）下有两个项目（project），文件目录如下：

```
- workspace
  - project_lib_a
    - src
      - suba.ga
      - liba.ga
  - project_lib_b
    - src
      - subb.ga
      - libb.ga
```

```ga
// project_lib_a/src/suba.ga
fn fa0() {}
pub var va0 = 10;
private type ta0 = {};

// project_lib_a/src/liba.ga
use ./suba
suba.fa0(); // 合法，默认所有符号都是项目内可见，liba.ga 和 suba.ga 属于同一个项目
suba.va0;   // 合法
suba.ta0;   // 不合法，ta0 使用 private 限制为文件内可见，只在 suba.ga 中可以访问

// project_lib_b/src/libb.ga
use ~project_lib_a
project_lib_a.ta0;     // 不合法，ta0 只在文件内可见。
project_lib_a.fa0();   // 不合法，fa0 只在 project_lib_a 内可见，在其它 project 中不可见。
project_lib_a.va0;     // 合法，va0 是通过 pub 导出的项目外可见符号
```

使用 `type` 关键字定义的类型，其所有字段也可以通过 `private`, `inner`, `pub` 关键字来指定可见范围，默认情况也是 `inner`。但如果类型本身是 `inner` 的情况下，指定 `pub` 字段没有意义；类型本身是 `private` 的情况下，指定 `pub` 或 `inner` 字段没有意义，相当于字段也全部是 `private`。


### 导入导出

`geekabe` 的模块导入比较灵活。可以导入文件、目录或项目。导入文件比较好理解跟其它语言类似，导入目录则可以使用目录下所有文件中的声明的符号，导入项目则可以使用项目的 `src` 目录下的所有文件中声明的符号。

导入的文件名/目录名/项目名，默认就是导入的命名空间。

```
/* 目录结构
- some_lib
  - src
    - aa.ga
    - bb
      - cc.ga
      - bb.ga
*/

// aa.ga
pub fn fa() {}
// bb.ga
pub fn fb() {}
pub fn some_fn() {}
// cc.ga
pub fn fc() {}
pub fn some_fn() {}
```

```ga
use some_lib
use some_lib/bb            // 等价于 use some_lib/src/bb，引入 bb 目录。
use some_lib/bb/bb as bbb  // 上一行引入的 some_lib/bb 目录的命名空间名是 bb，和当前行会有冲突。因此需要使用 as 重命名。

// 可以访问所有 pub 标记的符号
some_lib.fa;
some_lib.fb;
some_lib.fc;
// 可以访问 src/bb 目录下所有 pub 标记的符号
bb.fb;
bb.fc;
// 只能访问 src/bb/bb/ga 文件中所有 pub 标记的符号
bbb.fb;
```

### 冲突处理

`geekabe` 的隐式导出（即不需要使用 export 明确导出，只要是 pub/inner 标记的符号就能被导入使用）的方式可能导致冲突。

比如在上述例子中，如果 `cc.ga` 和 `bb.ga` 都声明了公共可见的符号 `some_fn`，那引入 `some_lib/bb` 这个目录后，在 `bb` 命名空间下会有同名的 `some_fn` 函数可用。

对于这种情况，编译器允许导入目录；只有在代码中使用有冲突的同名符号，才会阻止。

```ga
use some_lib

some_lib.some_fn() // 编译不通过，会提示 some_lib/src 下多个文件都有 pub 标记的 some_fn 符号。
some_lib.fb()      // 编译通过，因为 fb 只在 bb.ga 文件中定义。

// 如果要使用 some_fn 这个符号，需要缩小导入的范围
use some_lib/bb/bb
bb.some_fn()       // 编译通过
```

## 字符串

`geekabe` 的字符串以 `'` 或 `"` 包裹，且默认支持模板字符串能力。类似于 `javascript` 中的`` ` ``字符包裹的模板字符串，但细节上有所不同。

字符串中`{}`包裹的是需要嵌入字符串中的表达式，类似于 `javascript` 模板字符串中的 `${}`。编译器会自动对表达式的结果，进行字符串类型的转换（基本类型由编译器实现，结构类型需要使用 `~to str: fn() {}` 指定转换函数）。

```ga
var s1 = 'abc\'de"oo'; // 字符串是 abc'de"oo
var s2 = "abc'de\"oo'; // 字符串是 abc'de"oo
var name = "'de\"";
var a = 'o';
var b = 'o';
var s3 = 'abc{name}{a+b}'; // 字符串是 abc'de"oo
// 使用转义符支持 { 符号本身
var s4 = 'abc\{name}'; // 字符串是 abc{name}
var s5 = 'abc{name'; // 编译错误，嵌入表达式没有结束标志
type Boy {
  name: str,
  ~to str: fn() { return "I'm {me.name}" }
}
var boy = Boy({ name: 'blob' });
var s6 = 'abc{boy}';  // 字符串是 abcI'm blob
type A {}
var s7 = 'abc{A()}';  // 编译错误，类型 A 没有定义 ~to str 的转换函数
```

## 宏语言

静态语言在很多场景下都需要在静态代码层面引入动态逻辑，因此静态语言几乎都有引入“宏”的概念。但不同语言的“宏”的设计很不一样，C 语言的宏能力过于简单，Rust 语言的宏概念又稍显复杂。

`geekabe` 语言的宏，吸收了 `php` 和 `react jsx` 的思想，支持使用动态语言来生成静态语言的代码。详见 [Geekabe Macro Language.md](Geekabe%20Macro%20Language.md)
