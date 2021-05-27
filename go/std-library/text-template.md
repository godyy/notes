[TOC]

template包实现了数据驱动的用于生成文本输出的模板。其实简单来说就是将一组文本嵌入另一组文本模版中，返回一个你期望的文本。

通过将模板应用于一个数据结构（即该数据结构作为模板的参数）来执行，来获得输出。模板中的注释引用数据接口的元素（一般如结构体的字段或者字典的键）来控制执行过程和获取需要呈现的值。模板执行时会遍历结构并将指针表示为'.'（称之为"dot"）指向运行过程中数据结构的当前位置的值。

用作模板的输入文本必须是utf-8编码的文本。"Action"—数据运算和控制单位—由"{{"和"}}"界定；在Action之外的所有文本都不做修改的拷贝到输出中。Action内部不能有换行，但注释可以有换行。

模版一旦解析成功，即可以并发的执行。但因为并发执行共享一个Writer，输出可能是交错的。

一个简单的例子，可以打印“17 of wool”。
```
type Inventory struct {
	Material string
	Count    uint
}
sweaters := Inventory{"wool", 17}
tmpl, err := template.New("test").Parse("{{.Count}} of {{.Material}}")
if err != nil { panic(err) }
err = tmpl.Execute(os.Stdout, sweaters)
if err != nil { panic(err) }
```

# 文本和空格

默认情况下，模版执行时会将`actions`之间的文本会原封不动的拷贝到结果中。

然而，为了帮助格式化模版源代码，如果`action`的左定界符（默认为“{{”）紧跟着一个负号和空格（“- ”），所有前置文本之后的空白都会被剔除（“前面非空白文本之后到{{- ”之前）。相似的，如果`action`的右定界符（“}}”）前为“ -”，所有“ -}}”之后到后置文本之前的空白都会被剔除。

```
"{{23 -}} < {{- 45}}"

// output would be
"23<45"
```

“空白”的定义与 **go** 中的相同：空格，水平制表符，回车，换行符。

# Actions
下面是有效的`action`列表:
```
{{/* 注释 */}}
{{- /* 剔除掉前后空白的注释 */ -}}
    注释；废弃的？。可以包含换行符。
    注释不能嵌套，并且必须始于和结束于定界符。
    
{{pipeline}}
    默认的文本表示，pipeline 的值被拷贝到 output。
    
{{if pipeline}} T1 {{end}}
    如果 pipeline 的值为空，不会生成输出；否则，执行 T1。
    “空”值为 false，0，any nil pointer or interface value，
    and any array，slice，map，or string of length zero。
    Dot is unaffected。
    
{{if pipeline}} T1 {{else}} T0 {{end}}
    pipeline 的值为空，执行 T0；否则，执行 T1。
    Dot is unaffected。
    
{{if pipeline}} T1 {{else if pipeline}} T0 {{end}}
    
{{range pipeline}} T1 {{end}}
    pipeline 的值必须是 array,slice,map,channel其中之一。
    如果 len(pipeline)=0，"."不受影响且执行 T0；否则，"."
    被连续的设置为 pipeline 中的元素，并且执行 T1。
    
{{template "name"}}
    模版"name“执行，with nil data。
    
{{template "name" pipeline}}
    执行"name"模版，并且将"."设置为 pipeline 的值。
    
{{block "name" pipeline}} T1 {{end}}
    一种简写，用来定义模版的同时，执行它。
    1. 定义 {{define "name"}} T1 {{end}}
    2. {{template "name" pipeline}}
    典型的用法是，先定义一批根模版，然后通过重定义块模版
    实现个性化。
    
{{with pipeline}} T1 {{end}}
    如果 pipeline 的值为空，T1不会执行；否则，"."会被设置为 pipeline 的值，并执行 T1。
    
{{with pipeline}} T1 {{else}} T0 {{end}}
```

# Arguments

```
- go语法的布尔值、字符串、字符、整数、浮点数、虚数、复数，视为无类型字面常数

- 关键字nil，代表一个go的无类型的nil值

- 字符'.'（句点，用时不加单引号），代表dot的值

- 变量名，以美元符号起始加上（可为空的）字母和数字构成的字符串，如：$piOver2和$；
  执行结果为变量的值
- 结构体数据的字段名，以句点起始，如：.Field；
  执行结果为字段的值，支持链式调用：.Field1.Field2；
  字段也可以在变量上使用（包括链式调用）：$x.Field1.Field2；
  
- 字典类型数据的键名；以句点起始，如：.Key；
  执行结果是该键在字典中对应的成员元素的值；
  键也可以和字段配合做链式调用，深度不限：.Field1.Key1.Field2.Key2；
  虽然键也必须是字母和数字构成的标识字符串，但不需要以大写字母起始；
  键也可以用于变量（包括链式调用）：$x.key1.key2；
  
- 数据的无参数方法名，以句点为起始，如：.Method；
  执行结果为dot调用该方法的返回值，dot.Method()；
  该方法必须有1到2个返回值，如果有2个则后一个必须是error接口类型；
  如果有2个返回值的方法返回的error非nil，模板执行会中断并返回给调用模板执行者该错误；
  方法可和字段、键配合做链式调用，深度不限：.Field1.Key1.Method1.Field2.Key2.Method2；
  方法也可以在变量上使用（包括链式调用）：$x.Method1.Field；
  
- 无参数的函数名，如：fun；
  执行结果是调用该函数的返回值fun()；对返回值的要求和方法一样；函数和函数名细节参见后面。
  
- 上面某一条的实例加上括弧（用于分组）
  执行结果可以访问其字段或者键对应的值：
      print (.F1 arg1) (.F2 arg2)
      (.StructValuedMethod "arg").Field
```

Arguments可以是任何类型：如果是指针，在必要时会自动表示为指针指向的值；如果执行结果生成了一个函数类型的值，如结构体的函数类型字段，该函数不会自动调用，但可以在if等action里视为真。如果要调用它，使用call函数。

# Pipelines
pipeline 是一个（可能是链状的）command 序列。command 可以是一个简单值（argument）或者对函数或者方法的（可以有多个参数）调用：

```
Argument
    value of evaluating the argument
.Method [argument...]
    可以独立调用或者位于链式调用的末端，不同于链式调用中间的方法，可以传递参数。
functionName [argument...]
```

pipeline通常是将一个command序列分割开，再使用管道符'|'连接起来（但不使用管道符的command序列也可以视为一个管道）。在一个链式的pipeline里，每个command的结果都作为下一个command的最后一个参数。pipeline最后一个command的输出作为整个管道执行的结果。

command的输出可以是1到2个值，如果是2个后一个必须是error接口类型。如果error类型返回值非nil，模板执行会中止并将该错误返回给执行模板的调用者。

# Variables
action 内的 pipeline 可以通过初始化 variable 来捕获结果。语法如下：
```
$variable := pipeline
```
声明变量的 action 不会产生任何结果。

初始化后的 variable 可以赋值：
```
$variable = pipeline
```

如果 “range” action 只初始化一个变量，变量的值会被连续赋值为迭代中的元素值。

如果“range” action 声明两个变量，用“,”分隔：
```
range $index, $element := pipeline
```
`$index`和`$element`的值会被连续赋值为迭代中的 index/key 和 元素值。

variable 的作用域延伸到紧跟控制结构（“if”，“with”，or “range”）的“end” action，或者在没有控制结构的 template 最后。template 调用不会从它的调用店继承变量。

模版执行时，$ 会被设置为传递给 Execute 的参数，也就是"."的初始值。

# Examples
下面是一些单行模板，展示了pipeline和变量。所有都生成加引号的单词"output"：
```
{{"\"output\""}}
    字符串常量
{{`"output"`}}
    原始字符串常量
{{printf "%q" "output"}}
    函数调用
{{"output" | printf "%q"}}
    函数调用，最后一个参数来自前一个command的返回值
{{printf "%q" (print "out" "put")}}
    加括号的参数
{{"put" | printf "%s%s" "out" | printf "%q"}}
    玩出花的管道的链式调用
{{"output" | printf "%s" | printf "%q"}}
	管道的链式调用
{{with "output"}}{{printf "%q" .}}{{end}}
    使用dot的with action
{{with $x := "output" | printf "%q"}}{{$x}}{{end}}
	创建并使用变量的with action
{{with $x := "output"}}{{printf "%q" $x}}{{end}}
	将变量使用在另一个action的with action
{{with $x := "output"}}{{$x | printf "%q"}}{{end}}
	以管道形式将变量使用在另一个action的with action 
```

# Functions
执行模板时，函数从两个函数映射表中查找：首先是 template 函数映射表，然后是全局函数映射表。一般不在 template 内定义函数，而是使用Funcs方法添加函数到模板里。

预定义的全局函数如下：
```
and
    函数返回它的第一个empty参数或者最后一个参数；
    就是说"and x y"等价于"if x then y else x"；所有参数都会执行；
or
    返回第一个非empty参数或者最后一个参数；
    亦即"or x y"等价于"if x then x else y"；所有参数都会执行；
not
    返回它的单个参数的布尔值的否定
len
    返回它的参数的整数类型长度
index
    执行结果为第一个参数以剩下的参数为索引/键指向的值；
    如"index x 1 2 3"返回x[1][2][3]的值；每个被索引的主体必须是数组、切片或者字典。
print
    即fmt.Sprint
printf
    即fmt.Sprintf
println
    即fmt.Sprintln
html
    返回其参数文本表示的HTML逸码等价表示。
urlquery
    返回其参数文本表示的可嵌入URL查询的逸码等价表示。
js
    返回其参数文本表示的JavaScript逸码等价表示。
call
    执行结果是调用第一个参数的返回值，该参数必须是函数类型，其余参数作为调用该函数的参数；
    如"call .X.Y 1 2"等价于go语言里的dot.X.Y(1, 2)；
    其中Y是函数类型的字段或者字典的值，或者其他类似情况；
    call的第一个参数的执行结果必须是函数类型的值（和预定义函数如print明显不同）；
    该函数类型值必须有1到2个返回值，如果有2个则后一个必须是error接口类型；
    如果有2个返回值的方法返回的error非nil，模板执行会中断并返回给调用模板执行者该错误；
slice
    返回根据剩余参数对其第一个参数进行切片的结果。Thus "slice x 1 2" is, in Go syntax, x[1:2],
	while "slice x" is x[:], "slice x 1" is x[1:], and "slice x 1 2 3"
	is x[1:2:3]. The first argument must be a string, slice, or array.
```

布尔函数会将任何类型的零值视为假，其余视为真。

下面是定义为函数的二元比较运算的集合：
```
eq      如果arg1 == arg2则返回真
ne      如果arg1 != arg2则返回真
lt      如果arg1 < arg2则返回真
le      如果arg1 <= arg2则返回真
gt      如果arg1 > arg2则返回真
ge      如果arg1 >= arg2则返回真
```

为了简化多参数相等检测，eq（只有eq）可以接受2个或更多个参数，它会将第一个参数和其余参数依次比较，返回下式的结果：
```
arg1==arg2 || arg1==arg3 || arg1==arg4 ...
（和go的||不一样，不做惰性运算，所有参数都会执行）
```

比较函数只适用于基本类型（或重定义的基本类型，如"type Celsius float32"）。它们实现了go语言规则的值的比较，但具体的类型和大小会忽略掉，因此任意类型有符号整数值都可以互相比较；任意类型无符号整数值都可以互相比较；等等。但是，整数和浮点数不能互相比较。

# 模版关联
每个 template 在创建时都要用字符串命名。同时，每一个模板都会和0或多个模板关联，并可以使用它们的名字调用这些模板；这种关联可以传递，并形成一个模板的名字空间。

一个模板可以通过模板调用实例化另一个模板；参见上面的"template" action。name必须是包含模板调用的模板相关联的模板的名字。

# 嵌套模版定义
当解析模板时，可以定义另一个模板，该模板会和当前解析的模板相关联。模板必须定义在当前模板的最顶层，就像go程序里的全局变量一样。

这种定义模板的语法是将每一个子模板的声明放在"define"和"end" action内部。

define action使用给出的字符串常数给模板命名，举例如下：
```
`{{define "T1"}}ONE{{end}}
{{define "T2"}}TWO{{end}}
{{define "T3"}}{{template "T1"}} {{template "T2"}}{{end}}
{{template "T3"}}`
```

它定义了两个模板T1和T2，第三个模板T3在执行时调用这两个模板；最后该模板调用了T3。输出结果是：
```
ONE TWO
```

采用这种方法，一个模板只能从属于一个关联。如果需要让一个模板可以被多个关联查找到；模板定义必须多次解析以创建不同的*Template 值，或者必须使用Clone或AddParseTree方法进行拷贝。

可能需要多次调用Parse函数以集合多个相关的模板；参见ParseFiles和ParseGlob函数和方法，它们提供了简便的途径去解析保存在文件中的存在关联的模板。

一个模板可以直接调用或者通过ExecuteTemplate方法调用指定名字的相关联的模板；我们可以这样调用模板：
```
err := tmpl.Execute(os.Stdout, "no data needed")
if err != nil {
	log.Fatalf("execution failed: %s", err)
}
```

或显式的指定模板的名字来调用：
```
err := tmpl.ExecuteTemplate(os.Stdout, "T2", "no data needed")
if err != nil {
	log.Fatalf("execution failed: %s", err)
}
```
