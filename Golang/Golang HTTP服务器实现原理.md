# Golang HTTP 服务器实现原理



主要使用 `net/http` 包来创建 HTTP 服务器。

创建 HTTP 服务器的主要步骤：

1. 创建处理器；
2. 创建路由器，并注册处理器；
3. 启动服务器。



## 通过处理器结构方式实现 HTTP 服务器



### 处理器

从逻辑上来说，处理器的职责是响应客户端请求。

从技术上来说，处理器（`http.Handler`）是一个接口，任何类型只要实现了该接口的 `ServeHTTP` 方法，那么它就是一个处理器。

> ```go
> // 源码片段
> type Handler interface {
> 	ServeHTTP(ResponseWriter, *Request)
> }
> ```



### 服务器

```go
type MyHandler struct{}

func (mh *MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	_, err := w.Write([]byte("Index"))
	if err != nil {
		log.Println(err)
	}
}

func httpServeByStruct() {
	mh := MyHandler{}

	server := http.Server{
		Addr:    "127.0.0.1:8080",
		Handler: &mh,
	}

	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```



服务器（`http.Server`）是一个结构，其中有很多字段。

> ```go
> // 源码片段
> type Server struct {
> 	Addr string
> 	Handler Handler
>     	...
> }
> ```

`Server.Addr` 字段表示服务器监听的地址，格式是 `host:port`。

`Server.Handler` 字段表示处理器，其 `ServeHTTP` 方法包含处理请求的逻辑。

`Server.ListenAndServe` 方法会启动服务器。



启动服务器，打开浏览器尝试分别访问 http://127.0.0.1:8080/ 、http://127.0.0.1:8080/index 、http://127.0.0.1:8080/about ，发现返回的信息均是 *Index*。

说明 http://127.0.0.1:8080 下的任何请求均由一个处理器进行处理。

如果打算不同请求路径的请求由不同的处理器进行处理，则需要服务器设置路由器。



### 路由器

路由器（`http.ServeMux`）也叫多路复用器，是一个结构，也实现了 `http.Handler` 接口，所以从技术上来说，也是一个处理器。

只不过从逻辑上来说，其职责不同于处理器，而是分发客户端请求，即接收客户端请求后，分发到对应的处理器，让其响应请求。

```go
type MyHandler struct{}
type MyHandler2 struct{}

func (mh *MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	_, err := w.Write([]byte("Index"))
	if err != nil {
		log.Println(err)
	}
}

func (mh2 *MyHandler2) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	_, err := w.Write([]byte("About"))
	if err != nil {
		log.Println(err)
	}
}

func httpServeByStruct2() {
	mh := MyHandler{}
	mh2 := MyHandler2{}

	mux := http.NewServeMux()

	server := http.Server{
		Addr:    ":8080",
		Handler: mux,
	}

	mux.Handle("/index", &mh)
	mux.Handle("/about", &mh2)

	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```

启动服务器，打开浏览器尝试分别访问 http://127.0.0.1:8080/、http://127.0.0.1:8080/index、http://127.0.0.1:8080/about，发现请求路径为 `/` 的请求返回 *404*，而请求路径为 `/index` 和 `/adout` 的请求分别返回 *Index* 和 *About*。返回 *404* 是因为没有在路由器上为路由规则 `/` 注册对应的处理器。



`http.NewServeMux` 函数会创建一个路由器。

> ```go
> // 源码片段
> func NewServeMux() *ServeMux { return new(ServeMux) }
> ```

`ServeMux.Handle` 方法会在路由器上为特定的路由规则注册对应的处理器。

当请求到来时，请求路径和路由规则进行配对，找到最接近请求路径的路由规则，分发请求到该路由规则对应的处理器，让其响应请求。



### 代码的简化



#### http.DefaultServeMux

当服务器的 `Server.Handler` 字段为 `nil` 时，服务器所设置的路由器为 `net/http` 包内置的默认路由器：`http.DefaultServeMux`。

```go
func httpServeByStruct3() {
	mh := MyHandler{}

	server := http.Server{
		Addr:    ":8080",
		Handler: nil,
	}

	http.DefaultServeMux.Handle("/index", &mh)

	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```

这样编写代码的好处是无须手动创建路由器。



手动创建的路由器和 `http.DefaultServeMux` 本质是一样的。

> ```go
> // 源码片段
> var DefaultServeMux = &defaultServeMux
> var defaultServeMux ServeMux
> ```



#### http.Handle

```go
func httpServeByStruct4() {
	mh := MyHandler{}

	server := http.Server{
		Addr:    ":8080",
		Handler: nil,
	}

	http.Handle("/index", &mh)

	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```

`http.Handle` 函数的本质是代码的进一步封装，会使用内置的默认路由器注册处理器。

> ```go
> // 源码片段
> func Handle(pattern string, handler Handler) { DefaultServeMux.Handle(pattern, handler) }
> ```



#### http.ListenAndServe

```go
func httpServeByStruct5() {
	mh := MyHandler{}

	http.Handle("/index", &mh)

	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal(err)
	}
}
```

`http.ListenAndServe` 函数的本质也是代码的进一步封装，会创建服务器并设置字段，然后启动服务器。

> ```go
> // 源码片段
> func ListenAndServe(addr string, handler Handler) error {
> 	server := &Server{Addr: addr, Handler: handler}
> 	return server.ListenAndServe()
> }
> ```



## 通过处理器函数方式实现 HTTP 服务器



### 代码的简化



#### http.HandlerFunc

```go
func index(w http.ResponseWriter, r *http.Request) {
	_, err := w.Write([]byte("Index"))
	if err != nil {
		log.Println(err)
	}
}

func httpServeByFunc() {
	http.Handle("/index", http.HandlerFunc(index))

	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal(err)
	}
}
```

这样编写代码的好处是无须手动创建处理器结构。



`http.HandlerFunc` 是 `net/http` 包内置的方法类型，是一个适配器，其实现了 `http.Handler` 接口的 `ServeHTTP` 方法，所以也是一个处理器。

`http.HandlerFunc()` 不是进行函数的调用，而是进行类型转换，被转换的方法类型将转化为处理器，该方法类型的参数列表要求和 `http.Handler` 接口的 `ServeHTTP` 方法的参数列表一致。

> ```go
> // 源码片段
> type HandlerFunc func(ResponseWriter, *Request)
> func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
> 	f(w, r)
> }
> ```



#### http.HandleFunc

```go
func httpServeByFunc2() {
	http.HandleFunc("/index", index)

	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal(err)
	}
}
```

`http.HandleFunc` 函数的本质是代码的进一步封装，使用内置的默认路由器，调用 `ServeMux.Handle` 方法注册处理器和使用 `http.HandlerFunc` 进行类型转换。

> ```go
> // 源码片段
> func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
> 	DefaultServeMux.HandleFunc(pattern, handler)
> }
> func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
> 	if handler == nil {
> 		panic("http: nil handler")
> 	}
> 	mux.Handle(pattern, HandlerFunc(handler))
> }
> ```



## 总结

无论是通过**处理器结构**方式还是**处理器函数**方式来实现 HTTP 服务器，本质都是一样的，究其原因是任何类型只要实现了 `http.Handler` 接口的 `ServeHTTP` 方法，那么就可以视为一个处理器；更多的区别是代码编写上的简洁性和灵活性。

为什么会有两种方式实现 HTTP 服务器？

原因是可以通过处理器结构方式，为已有的结构，编写 `http.Handler` 接口的 `ServeHTTP` 方法的具体逻辑代码来实现处理器，进而实现 HTTP 服务器。

这体现了组合相较于继承的优越性，通过添加或较小地改动代码，就可以添加新的功能。



## 代码

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	fmt.Println("Please visit http://127.0.0.1:8080/ or http://127.0.0.1:8080/index or http://127.0.0.1:8080/about")

	httpServeByStruct()
	// httpServeByStruct2()
	// httpServeByStruct3()
	// httpServeByStruct4()
	// httpServeByStruct5()
	// httpServeByFunc()
	// httpServeByFunc2()
}

type MyHandler struct{}
type MyHandler2 struct{}

func (mh *MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	_, err := w.Write([]byte("Index"))
	if err != nil {
		log.Println(err)
	}
}

func (mh2 *MyHandler2) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	_, err := w.Write([]byte("About"))
	if err != nil {
		log.Println(err)
	}
}

func httpServeByStruct() {
	mh := MyHandler{}

	server := http.Server{
		Addr:    "127.0.0.1:8080",
		Handler: &mh,
	}

	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}

func httpServeByStruct2() {
	mh := MyHandler{}
	mh2 := MyHandler2{}

	mux := http.NewServeMux()

	server := http.Server{
		Addr:    ":8080",
		Handler: mux,
	}

	mux.Handle("/index", &mh)
	mux.Handle("/about", &mh2)

	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}

func httpServeByStruct3() {
	mh := MyHandler{}

	server := http.Server{
		Addr:    ":8080",
		Handler: nil,
	}

	http.DefaultServeMux.Handle("/index", &mh)

	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}

func httpServeByStruct4() {
	mh := MyHandler{}

	server := http.Server{
		Addr:    ":8080",
		Handler: nil,
	}

	http.Handle("/index", &mh)

	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}

func httpServeByStruct5() {
	mh := MyHandler{}

	http.Handle("/index", &mh)

	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal(err)
	}
}

func index(w http.ResponseWriter, r *http.Request) {
	_, err := w.Write([]byte("Index"))
	if err != nil {
		log.Println(err)
	}
}

func httpServeByFunc() {
	http.Handle("/index", http.HandlerFunc(index))

	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal(err)
	}
}

func httpServeByFunc2() {
	http.HandleFunc("/index", index)

	if err := http.ListenAndServe(":8080", nil); err != nil {
		log.Fatal(err)
	}
}
```



## 参考资料

1. https://www.bilibili.com/video/BV1Xv411k7Xn
2. https://juejin.cn/post/6867149633328513038