# Golang Web 应用中的模板引擎



## 什么是模板

模板就是可以复用的、静态的 HTML 文本，这些文本通常会包含在文本文件里，如 HTML 文件。

模板里面通常会嵌入一些称为动作（Action）的命令。

模板引擎会将模板和动态数据组合在一起生成最终的 HTML 文本。



## 模板引擎类型

- **无逻辑模板引擎**
  - 指定的占位符被嵌入到模板中
  - 只做字符串替换处理（占位符替换成动态数据），不做任何逻辑处理
  - 处理由处理器来完成
  - 展示和逻辑完全分离
- **嵌入逻辑模板引擎**
  - 编程语言被嵌入到模板中
  - 不仅做字符串替换处理（占位符替换成动态数据），还可以做逻辑处理
  - 处理由模板引擎完成
  - 功能强大，但难以维护（逻辑处理代码散布在处理器和模板）

Golang 的模板引擎与大多数模板引擎一样，介于两者之间。



## 模板引擎的工作流程

1. 处理器调用模板引擎，并传入模板和动态数据
2. 模板引擎生成最终的 HTML 文本，并传递给 ResponseWrite
3. ResponseWrite 再将 HTML 文本写入 HTTP 响应，返回给客户端



## 使用模板引擎

1. 解析模板源（字符串或可读的文本文件的内容），从而创建一个完成解析的模板
2. 执行完成解析的模板，并传入 ResponseWrite 和动态数据
   - 这会触发模板引擎将 ResponseWrite 和动态数据组合在一起，生成最终的 HTML 文本，并将它传递给 ResponseWrite



### 解析模板源

#### ParseFiles 函数

main.go

```go
package main

import (
	"html/template"
	"log"
	"net/http"
)

func main() {
	http.HandleFunc("/", func(rw http.ResponseWriter, r *http.Request) {
		// 解析模板源
		// t, _ := template.ParseFiles("tmpl.html")

		// 相当于创建模板，然后解析文本文件内容到模板
		// t := template.New("tmpl.html")
		// t, _ = t.ParseFiles("tmpl.html")

		// 解析模板源，增加解析模板源时的错误处理
		t := template.Must(template.ParseFiles("tmpl.html"))

		// 执行模板
		t.Execute(rw, "Hello, world!")
	})
	log.Fatal(http.ListenAndServe("127.0.0.1:8080", nil))
}
```

tmpl.html

```html
<!DOCTYPE html>
<html>

<head>
    <title>tmpl</title>
</head>

<body>
    {{ . }}
</body>

</html>
```

`template.ParseFiles` 函数

- 解析文本文件，并创建一个完成解析的模板
- 完成解析的模板的名字是文件名
- 传入的参数是文件的相对路径（相对于程序二进制文件，即项目根目录）
- 传入的参数数量可变，但只返回一个模板
  - 当解析多个文件时，第一个文件作为返回的模板（名字、内容），其余的作为一个 map 供后继使用



### 解析模板源的错误处理

#### Must 函数

- 接收一个函数，返回一个模板指针和一个错误
  - 如果错误不为 `nil`，那么发生 `panic`
- 简化了解析模板源过程中的错误处理

```go
// 解析模板源，增加解析模板源的错误处理
t := template.Must(template.ParseFiles("tmpl.html"))
```



### 执行模板

- **Execute 函数**
  - 参数是 ResponseWrite、动态数据
  - 单模板：适用
  - 模板集：只能用第一个模板
- **ExecuteTemplate 函数**
  - 参数是 ResponseWrite、模板名、动态数据
  - 模板集：适用



### 模板动作命令（Action）

Action 是 Golang 模板中嵌入的命令，位于两组大括号之间 。

点（`.`）是最重要的一个 Action，它代表了传入模板的上下文数据。

Action 主要可以分为五类：条件动作、迭代动作、设置动作、包含动作、定义动作。



#### 条件动作

```html
{{ if arg }}
	some content
{{ else }}
	other content
{{ end }}
```

`arg` 是传递给条件动作的一个参数，根据 `arg` 的真假选择那部分内容。



#### 迭代动作

```html
{{ range array }}
	element: {{ . }}
{{ end }}
```

`array` 是传递给迭代动作的一个数组，也可以是切片、映射或者是通道。

`.` 在迭代动作内部时，被设置为当前被迭代的元素。



#### 设置动作

```html
{{ with arg}}
	{{ . }}
{{ end }}
```

`arg` 是传递给设置动作的一个参数，设置动作内的 `.` 被设置为 `arg`。



#### 包含动作

```html
{{ template "name" arg }}
```

包含动作允许模板包含其它模板。

`"name"` 是其它模板的名字。

`arg` 是传入其它模板的上下文数据，是可选的。



#### 定义动作

```html
{{ define "layout" }}
	{{ template "content" }}
{{ end }}
```

显式地定义一个名为 `"layout"` 的模板。



### 模板里的参数、变量和管道

传入模板的参数是 Golang 的变量。

如果参数是函数，那么要求该函数只返回一个值或返回一个值和一个错误。

动作命令中可以设置变量，变量以 `$` 开头。

```html
{{ range $key, $value := . }}
	{{ $key }}: {{ $value }}
{{ end }}
```

管道就是许多有序串联起来的参数，允许将前一个参数的输出的值传递给下一个参数，各个参数之间使用 `|` 分隔。

```html
{{ p1 | p2 | p3 }}
```

