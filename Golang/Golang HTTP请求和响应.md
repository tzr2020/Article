# Golang HTTP请求和响应

## Request 结构

`net/http` 包中的 `Request` 结构表示 **HTTP 请求消息**。

其重要字段有 Method、URL、Header、Body、Form。



### URL 字段

`Request.URL` 字段表示**请求行里的请求地址**，是一个指向 `url.URL` 结构的指针。

```go
// 源码片段
type URL struct {
	Scheme      string
	Opaque      string
	User        *Userinfo
	Host        string
	Path        string
	RawPath     string
	ForceQuery  bool
	RawQuery    string
	Fragment    string
	RawFragment string
}
```



URL 的通用格式：`scheme://[userinfo@]host/path[?query][#frament]`。

`URL.RawQuery` 字段：查询字符串参数

- 以 `&` 分隔的键值对，以 `=` 分隔键和值的字符串

- 可以通过 `URL.Query` 方法获取查询字符串参数，返回 `map[string][]string` 类型。

`URL.Fragment` 字段：frament

- 浏览器发送的请求无法获取 frament，因为浏览器发送请求时会把 frament 去掉
- 不是所有请求都是由浏览器发送，其它客户端发送的请求可以获取 frament



### Header 字段

`Request.Header` 字段表示 **请求头**，是 `header.Header` 类型，用来表述 HTTP 头部里的键值对。

```go
// 源码片段
type Header map[string][]string
```



### Body 字段

`Request.Body` 字段表示 **请求体**，是 `io.ReadCloser` 类型。

```go
// 源码片段
type ReadCloser interface {
	Reader
	Closer
}

type Writer interface {
	Write(p []byte) (n int, err error)
}

type Closer interface {
	Close() error
}
```



`Body.Read` 方法：获取请求体的内容

```go
len := r.ContentLength
body := make([]byte, len)
r.Body.Read(body)
fmt.Println(string(body))
```



### HTML 表单

HTML 表单提供了交互控制元件，来通过 HTTP 协议向服务器发送请求，提交数据。

用户可以在浏览器通过 HTML 表单向服务器提交结构化数据。

HTML 表单提交数据时的**关键属性**

- **action**：请求地址
- **method**：请求方法
  - GET：表单数据以**查询字符串参数**方式提交
    - 查询字符串参数格式：以 `?` 开头，以 `&` 分隔键值对，以 `=` 分隔键和值
  - POST：表单数据以**包体**方式提交
- **enctype**：包体中表单数据的编码方式
  - application/x-www-form-urlencoded
    - enctype 属性的默认值
    - 数据以 `&` 分隔键值对，以 `=` 分隔键和值
  - multipart/form-data
    - 数据被分成多个子包体



### Form 字段和 PostForm 字段

`Request.Form` 字段表示以**查询字符串参数**方式和以**包体**方式提交的**表单数据**，`Request.PostForm` 字段表示以**包体**方式提交的表单数据，它们都是 `url.Values` 类型。

```go
// 源码片段
// Values maps a string key to a list of values.
type Values map[string][]string
```



如果以**查询字符串参数**方式提交和以**包体**方式提交的表单数据里有相同的 Key，那么它们都被会放在同一个 value slice 里，以**包体**方式提交的表单数据的 value 在前，以**查询字符串参数**方式提交的表单数据的 value 在后。

要使 `Request.Form` 字段和 `Request.PostForm` 字段生效，需要先调用 `Request.ParseForm` 方法解析表单，该方法只支持解析 **application/x-www-form-urlencoded** 编码方式的表单数据。



### MultipartForm 字段

`Request.MultipartForm` 字段表示以**包体**方式提交的**表单数据**，是 `multipart.Form` 类型的指针。

```go
// 源码片段
type Form struct {
	Value map[string][]string
	File  map[string][]*FileHeader
}
```



第一个 map 包含表单数据，第二个 map 包含上传文件数据。

要使 `Request.MultipartForm` 字段生效，需要先调用 `Request.ParseMultipartForm` 方法解析表单。

该方法支持解析 **multipart/form-data** 编码方式的表单，内部会在有必要的时候调用 `Request.ParseForm` 方法 。



### FormValue 方法和 PostFromValue 方法

`Request.FormValue` 方法和 `Request.PostFormValue` 方法内部都会调用 `Request.ParseMultipartForm` 方法解析表单。

`Request.FormValue` 方法返回 `Request.Form` 字段指定 key 对应的第一个 value。

`Request.PostFormValue` 方法返回 `Request.PostForm` 字段指定 key 对应的 value。



## ResponseWriter 接口

`net/http` 包中的 `ResponseWriter` 接口，其包含 Write、WriteHeader、Header 三个方法。

非导出的 `http.response` 结构表示 **HTTP 响应消息**，其作为指针接收者实现了`ResponseWriter` 接口。



### Write 方法

`ResponseWriter.Write` 方法**用于将数据写入 HTTP 包体**。

如果没有设置 Context-type 头部，那么将通过检查被写入数据的前512个字节来决定 Context-type 头部。

```go
// 示例
str := "Hello, World！"
w.Write([]byte(str))
```



### WriteHeader 方法

`ResponseWriter.WriteHeader` 方法接收一个整数作为参数，并把它设置为 HTTP 响应的状态码。

如果该方法如果没有被显式调用，那么在第一次调用 `ResponseWriter.Write` 方法之前，会隐式调用 `WriteHeader(http.StatusOK)`，也就是设置状态码为200，所以显示调用 `ResponseWriter.WriteHeader` 方法**主要用于设置错误类型的状态码**。

调用 `ResponseWriter.WriteHeader` 方法之后，无法再次修改响应头部。



### Header 方法

`ResponseWriter.Header` 方法返回**响应头部**，可以配合 `Header.Set` 方法**修改响应头部**。

```go
// 示例：重定向
w.Header().Set("Location", "http://www.baidu.com")
w.WriteHeader(302)
```

```go
// 示例：输出JSON
w.Header.Set("Content-Type", "application/json")
p := &Person {
    Name:	"ZhangSan"
    Age:	20
}
json, _ := json.Marshal(p)
w.Write(json)
```


