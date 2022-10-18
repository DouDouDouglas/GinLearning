# Gin简明教程----Geektutu

## 安装Gin

```shell
go get -u -v github.com/gin-gonic/gin
```

`-v`：打印出被构建的代码包的名字  
`-u`：已存在相关的代码包，强行更新代码包及其依赖包

## 第一个Gin程序

```go
// geektutu.com
// main.go
package main

import "github.com/gin-gonic/gin"

func main() {
	r := gin.Default()
	r.GET("/", func(c *gin.Context) {
		c.String(200, "Hello, Geektutu")
	})
	r.Run() // listen and serve on 0.0.0.0:8080
}
```

1. 首先，我们使用了`gin.Default()`生成了一个实例，这个实例即 WSGI 应用程序。
2. 接下来，我们使用`r.Get("/", ...)`声明了一个路由，告诉 Gin 什么样的URL 能触发传入的函数，这个函数返回我们想要显示在用户浏览器中的信息。
3. 最后用 `r.Run()`函数来让应用运行在本地服务器上，默认监听端口是 _8080_，可以传入参数设置端口，例如`r.Run(":9999")`即运行在 _9999_端口。



有时候我们需要动态的路由，如 `/user/:name`，通过调用不同的 url 来传入不同的 name。`/user/:name/*role`，`*` 代表可选。

```go
// 匹配 /user/geektutu
r.GET("/user/:name", func(c *gin.Context) {
	name := c.Param("name")
	c.String(http.StatusOK, "Hello %s", name)
})
```

```shell
$ curl http://localhost:9999/user/geektutu
Hello geektutu
```

## 路由

获取Query参数

```go
// 匹配users?name=xxx&role=xxx，role可选
r.GET("/users", func(c *gin.Context) {
	name := c.Query("name")
	role := c.DefaultQuery("role", "teacher")
	c.String(http.StatusOK, "%s is a %s", name, role)
})
```

```shell
$ curl "http://localhost:9999/users?name=Tom&role=student"
Tom is a student
```

获取POST参数

```go
// POST
r.POST("/form", func(c *gin.Context) {
	username := c.PostForm("username")
	password := c.DefaultPostForm("password", "000000") // 可设置默认值

	c.JSON(http.StatusOK, gin.H{
		"username": username,
		"password": password,
	})
})
```

```shell
$ curl http://localhost:9999/form  -X POST -d 'username=geektutu&password=1234'
{"password":"1234","username":"geektutu"}
```

Query和POST混合参数

```go
// GET 和 POST 混合
r.POST("/posts", func(c *gin.Context) {
	id := c.Query("id")
	page := c.DefaultQuery("page", "0")
	username := c.PostForm("username")
	password := c.DefaultPostForm("username", "000000") // 可设置默认值

	c.JSON(http.StatusOK, gin.H{
		"id":       id,
		"page":     page,
		"username": username,
		"password": password,
	})
})
```

```shell
$ curl "http://localhost:9999/posts?id=9876&page=7"  -X POST -d 'username=geektutu&password=1234'
{"id":"9876","page":"7","password":"1234","username":"geektutu"}
```

Map参数(字典参数)

```go
r.POST("/post", func(c *gin.Context) {
	ids := c.QueryMap("ids")
	names := c.PostFormMap("names")

	c.JSON(http.StatusOK, gin.H{
		"ids":   ids,
		"names": names,
	})
})
```

```shell
$ curl -g "http://localhost:9999/post?ids[Jack]=001&ids[Tom]=002" -X POST -d 'names[a]=Sam&names[b]=David'
{"ids":{"Jack":"001","Tom":"002"},"names":{"a":"Sam","b":"David"}}
```

重定向(Redirect)

```go
r.GET("/redirect", func(c *gin.Context) {
    c.Redirect(http.StatusMovedPermanently, "/index")
})

r.GET("/goindex", func(c *gin.Context) {
	c.Request.URL.Path = "/"
	r.HandleContext(c)
})
```

分组路由(Grouping Routes)

如果有一组路由，前缀都是`/api/v1`开头，是否每个路由都需要加上`/api/v1`这个前缀呢？答案是不需要，分组路由可以解决这个问题。利用分组路由还可以更好地实现权限控制，例如将需要登录鉴权的路由放到同一分组中去，简化权限控制。

```go
// group routes 分组路由
defaultHandler := func(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"path": c.FullPath(),
	})
}
// group: v1
v1 := r.Group("/v1")
{
	v1.GET("/posts", defaultHandler)
	v1.GET("/series", defaultHandler)
}
// group: v2
v2 := r.Group("/v2")
{
	v2.GET("/posts", defaultHandler)
	v2.GET("/series", defaultHandler)
}
```

```shell
$ curl http://localhost:9999/v1/posts
{"path":"/v1/posts"}
$ curl http://localhost:9999/v2/posts
{"path":"/v2/posts"}
```

## HTML模版

```go
type student struct {
	Name string
	Age  int8
}

r.LoadHTMLGlob("templates/*")

stu1 := &student{Name: "Geektutu", Age: 20}
stu2 := &student{Name: "Jack", Age: 22}
r.GET("/arr", func(c *gin.Context) {
	c.HTML(http.StatusOK, "arr.tmpl", gin.H{
		"title":  "Gin",
		"stuArr": [2]*student{stu1, stu2},
	})
})
```

```html
<!-- templates/arr.tmpl -->
<html>
<body>
    <p>hello, {{.title}}</p>
    {{range $index, $ele := .stuArr }}
    <p>{{ $index }}: {{ $ele.Name }} is {{ $ele.Age }} years old</p>
    {{ end }}
</body>
</html>
```

## 中间件Middleware

```go
// 作用于全局
r.Use(gin.Logger())
r.Use(gin.Recovery())

// 作用于单个路由
r.GET("/benchmark", MyBenchLogger(), benchEndpoint)

// 作用于某个组
authorized := r.Group("/")
authorized.Use(AuthRequired())
{
	authorized.POST("/login", loginEndpoint)
	authorized.POST("/submit", submitEndpoint)
}
```

```go
func Logger() gin.HandlerFunc {
	return func(c *gin.Context) {
		t := time.Now()
		// 给Context实例设置一个值
		c.Set("geektutu", "1111")
		// 请求前
		c.Next()
		// 请求后
		latency := time.Since(t)
		log.Print(latency)
	}
}
```

## 热加载

`go get -v -u github.com/pilu/fresh`

`fresh`


