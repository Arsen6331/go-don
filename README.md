# Don - Go API Framework

[![GoDoc](https://pkg.go.dev/badge/github.com/abemedia/go-don)](https://pkg.go.dev/github.com/abemedia/go-don)

Don is a fast & simple API framework written in Go. It features a super-simple API and thanks to
[fasthttp](https://github.com/valyala/fasthttp) and a custom version of
[httprouter](https://github.com/abemedia/httprouter) it's blazing fast and has a low memory
footprint.

As Don uses the new Go generics it requires Go 1.18 to work. Also, it's still in beta and the API
may yet have breaking changes, so it's not recommended for production use yet.

## Contents

- [Overview](#don---go-api-framework)
  - [Basic Example](#basic-example)
  - [Configuration](#configuration)
  - [Support multiple formats](#support-multiple-formats)
    - [Currently supported formats](#currently-supported-formats)
    - [Adding custom encoding](#adding-custom-encoding)
  - [Request parsing](#request-parsing)
  - [Headers & response codes](#headers--response-codes)
  - [Sub-routers](#sub-routers)
  - [Middleware](#middleware)
  - [Benchmarks](#benchmarks)

## Basic Example

```go
package main

import (
  "context"
  "errors"
  "fmt"
  "net/http"

  "github.com/abemedia/go-don"
  _ "github.com/abemedia/go-don/encoding/json" // Enable JSON parsing & rendering.
  _ "github.com/abemedia/go-don/encoding/yaml" // Enable YAML parsing & rendering.
)

type GreetRequest struct {
  Name string `path:"name"`         // Get name from the URL path.
  Age  int    `header:"X-User-Age"` // Get age from HTTP header.
}

type GreetResponse struct {
  // Remember to add all the tags for the renderers you enable.
  Greeting string `json:"data" yaml:"data"`
}

func Greet(ctx context.Context, req GreetRequest) (*GreetResponse, error) {
  if req.Name == "" {
    return nil, don.Error(errors.New("missing name"), http.StatusBadRequest)
  }

  res := &GreetResponse{
    Greeting: fmt.Sprintf("Hello %s, you're %d years old.", req.Name, req.Age),
  }

  return res, nil
}

func Pong(context.Context, don.Empty) (string, error) {
  return "pong", nil
}

func main() {
  r := don.New(nil)
  r.Get("/ping", don.H(Pong)) // Handlers are wrapped with `don.H`.
  r.Post("/greet/:name", don.H(Greet))
  r.ListenAndServe(":8080")
}
```

## Configuration

Don is configured by passing in the `Config` struct to `don.New`.

```go
r := don.New(&don.Config{
  DefaultEncoding: "application/json",
  DisableNoContent: false,
})
```

### DefaultEncoding

Set this to the format you'd like to use if no `Content-Type` or `Accept` headers are in the
request.

### DisableNoContent

If you return `nil` from your handler, Don will respond with an empty body and a `204 No Content`
status code. Set this to `true` to disable that behaviour.

## Support multiple formats

Support multiple request & response formats without writing extra parsing or rendering code. The API
uses the `Content-Type` and `Accept` headers to determine what input and output encoding to use.

You can mix multiple formats, for example if the `Content-Type` header is set to `application/json`,
however the `Accept` header is set to `application/x-yaml`, then the request will be parsed as JSON,
and the response will be YAML encoded.

If no `Content-Type` or `Accept` header is passed the default will be used.

Formats need to be explicitly imported e.g.

```go
import _ "github.com/abemedia/go-don/encoding/yaml"
```

### Currently supported formats

#### JSON

MIME: `application/json`

Parses JSON requests & encodes responses as JSON. Use the `json` tag in your request & response
structs.

#### XML

MIME: `application/xml`, `text/xml`

Parses XML requests & encodes responses as XML. Use the `xml` tag in your request & response
structs.

#### YAML

MIME: `application/x-yaml`, `text/x-yaml`

Parses YAML requests & encodes responses as YAML. Use the `yaml` tag in your request & response
structs.

#### Form (input only)

MIME: `application/x-www-form-urlencoded`, `multipart/form-data`

Parses form data requests. Use the `form` tag in your request struct.

#### Text

MIME: `text/plain`

Parses non-struct requests and encodes non-struct responses e.g. `string`, `int`, `bool` etc.

```go
func MyHandler(ctx context.Context, req int64) (string, error) {
  // ...
}
```

If the request is a struct and the `Content-Type` header is set to `text/plain` it returns a
`415 Unsupported Media Type` error.

### Adding custom encoding

Adding your own is easy. See [encoding/json/json.go](./blob/master/encoding/json/json.go).

## Request parsing

Automatically unmarshals values from headers, URL query, URL path & request body into your request
struct.

```go
type MyRequest struct {
  // Get from the URL path.
  ID int64 `path:"id"`

  // Get from the URL query.
  Filter string `query:"filter"`

  // Get from the JSON, YAML, XML or form body.
  Content float64 `form:"bar" json:"bar" yaml:"bar" xml:"bar"`

  // Get from the HTTP header.
  Lang string `header:"Accept-Language"`
}
```

Please note that using a pointer as the request type negatively affects performance.

## Headers & response codes

Implement the `StatusCoder` and `Headerer` interfaces to customise headers and response codes.

```go
type MyResponse struct {
  Foo  string `json:"foo"`
}

// Set a custom HTTP response code.
func (nr *MyResponse) StatusCode() int {
  return 201
}

// Add custom headers to the response.
func (nr *MyResponse) Header() http.Header {
  header := http.Header{}
  header.Set("foo", "bar")
  return header
}
```

## Sub-routers

You can create sub-routers using the `Group` function:

```go
r := don.New(nil)
sub := r.Group("/api")
sub.Get("/hello")
```

## Middleware

Don uses the standard fasthttp middleware format of
`func(fasthttp.RequestHandler) fasthttp.RequestHandler`.

For example:

```go
func loggingMiddleware(next fasthttp.RequestHandler) fasthttp.RequestHandler {
  return func(ctx *fasthttp.RequestCtx) {
    log.Println(string(ctx.RequestURI()))
    next(ctx)
  })
}
```

It is registered on a router using `Use` e.g.

```go
r := don.New(nil)
r.Post("/", don.H(handler))
r.Use(loggingMiddleware)
```

Middleware registered on a group only applies to routes in that group and child groups.

```go
r := don.New(nil)
r.Get("/login", don.H(loginHandler))
r.Use(loggingMiddleware) // applied to all routes

api := r.Group("/api")
api.Get("/hello", don.H(helloHandler))
api.Use(authMiddleware) // applied to routes `/api/hello` and `/api/v2/bye`


v2 := api.Group("/v2")
v2.Get("/bye", don.H(byeHandler))
v2.Use(corsMiddleware) // only applied to `/api/v2/bye`

```

To pass values from the middleware to the handler extend the context e.g.

```go
func myMiddleware(next fasthttp.RequestHandler) fasthttp.RequestHandler {
  return func(ctx *fasthttp.RequestCtx) {
    ctx.SetUserValue(ContextUserKey, "my_user")
    next(ctx)
  })
}
```

This can now be accessed in the handler:

```go
user := ctx.Value(ContextUserKey).(string)
```

## Benchmarks

To give you a rough idea of Don's performance, here is a comparison with Gin.

### Request Parsing

Don has extremely fast & efficient binding of request data.

| Benchmark name           |     (1) |         (2) |       (3) |          (4) |
| ------------------------ | ------: | ----------: | --------: | -----------: |
| BenchmarkDon_BindRequest | 2497761 | 469.8 ns/op |  104 B/op |  2 allocs/op |
| BenchmarkGin_BindRequest |  296085 |  3970 ns/op | 1273 B/op | 31 allocs/op |

Source:
[benchmarks/binding_test.go](https://github.com/abemedia/go-don/blob/master/benchmarks/binding_test.go)

### Serving HTTP Requests

Keep in mind that the majority of time here is actually the HTTP roundtrip.

| Benchmark name    |   (1) |         (2) |       (3) |          (4) |
| ----------------- | ----: | ----------: | --------: | -----------: |
| BenchmarkDon_HTTP | 59714 | 19951 ns/op |   73 B/op |  3 allocs/op |
| BenchmarkGin_HTTP | 34322 | 34331 ns/op | 2304 B/op | 20 allocs/op |

Source:
[benchmarks/http_test.go](https://github.com/abemedia/go-don/blob/master/benchmarks/http_test.go)
