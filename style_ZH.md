<!--

Editing this document:

- Discuss all changes in GitHub issues first.
- Update the table of contents as new sections are added or removed.
- Use tables for side-by-side code samples. See below.

Code Samples:

Use 2 spaces to indent. Horizontal real estate is important in side-by-side
samples.

For side-by-side code samples, use the following snippet.

~~~
<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
BAD CODE GOES HERE
```

</td><td>

```go
GOOD CODE GOES HERE
```

</td></tr>
</tbody></table>
~~~

(You need the empty lines between the <td> and code samples for it to be
treated as Markdown.)

If you need to add labels or descriptions below the code samples, add another
row before the </tbody></table> line.

~~~
<tr>
<td>DESCRIBE BAD CODE</td>
<td>DESCRIBE GOOD CODE</td>
</tr>
~~~

-->

# Uber Go 编码规范

## 目录

- [Uber Go 编码规范](#uber-go-%e7%bc%96%e7%a0%81%e8%a7%84%e8%8c%83)
  - [目录](#%e7%9b%ae%e5%bd%95)
  - [介绍](#%e4%bb%8b%e7%bb%8d)
  - [指导原则](#%e6%8c%87%e5%af%bc%e5%8e%9f%e5%88%99)
    - [指向Interfaces的指针](#%e6%8c%87%e5%90%91interfaces%e7%9a%84%e6%8c%87%e9%92%88)
    - [Receivers和Interfaces](#receivers%e5%92%8cinterfaces)
    - [零值Mutexes是有效的](#%e9%9b%b6%e5%80%bcmutexes%e6%98%af%e6%9c%89%e6%95%88%e7%9a%84)
    - [在边界处拷贝Slices和Maps](#%e5%9c%a8%e8%be%b9%e7%95%8c%e5%a4%84%e6%8b%b7%e8%b4%9dslices%e5%92%8cmaps)
      - [接收Slices和Maps](#%e6%8e%a5%e6%94%b6slices%e5%92%8cmaps)
      - [返回slices或maps](#%e8%bf%94%e5%9b%9eslices%e6%88%96maps)
    - [使用defer做清理](#%e4%bd%bf%e7%94%a8defer%e5%81%9a%e6%b8%85%e7%90%86)
    - [Channel的size是1或者无缓冲](#channel%e7%9a%84size%e6%98%af1%e6%88%96%e8%80%85%e6%97%a0%e7%bc%93%e5%86%b2)
    - [枚举从1开始](#%e6%9e%9a%e4%b8%be%e4%bb%8e1%e5%bc%80%e5%a7%8b)
    - [错误类型](#%e9%94%99%e8%af%af%e7%b1%bb%e5%9e%8b)
    - [Error Wrapping](#error-wrapping)
    - [Handle Type Assertion Failures](#handle-type-assertion-failures)
    - [Don't Panic](#dont-panic)
    - [Use go.uber.org/atomic](#use-gouberorgatomic)
  - [Performance](#performance)
    - [Prefer strconv over fmt](#prefer-strconv-over-fmt)
    - [Avoid string-to-byte conversion](#avoid-string-to-byte-conversion)
  - [Style](#style)
    - [Group Similar Declarations](#group-similar-declarations)
    - [Import Group Ordering](#import-group-ordering)
    - [Package Names](#package-names)
    - [Function Names](#function-names)
    - [Import Aliasing](#import-aliasing)
    - [Function Grouping and Ordering](#function-grouping-and-ordering)
    - [Reduce Nesting](#reduce-nesting)
    - [Unnecessary Else](#unnecessary-else)
    - [Top-level Variable Declarations](#top-level-variable-declarations)
    - [Prefix Unexported Globals with _](#prefix-unexported-globals-with)
    - [Embedding in Structs](#embedding-in-structs)
    - [Use Field Names to initialize Structs](#use-field-names-to-initialize-structs)
    - [Local Variable Declarations](#local-variable-declarations)
    - [nil is a valid slice](#nil-is-a-valid-slice)
    - [Reduce Scope of Variables](#reduce-scope-of-variables)
    - [Avoid Naked Parameters](#avoid-naked-parameters)
    - [Use Raw String Literals to Avoid Escaping](#use-raw-string-literals-to-avoid-escaping)
    - [Initializing Struct References](#initializing-struct-references)
    - [Format Strings outside Printf](#format-strings-outside-printf)
    - [Naming Printf-style Functions](#naming-printf-style-functions)
  - [Patterns](#patterns)
    - [Test Tables](#test-tables)
    - [Functional Options](#functional-options)

## 介绍

风格（style）是我们管理代码的约定。术语风格有点用词不当, 因为这些约定的覆盖范围不仅限于gofmt工具为我们处理源文件格式。

本指南通过详细描述uber内部编写go代码的行为准则来管理编写代码产生的复杂度。这些规则的存在是为了代码库易于管理的同时仍然允许工程师高效的使用go语言特性。

本手册最初是由[Prashant Varanasi]和[Simon Newton]编写，为了帮助同事更快速的使用go语言。
多年来它已被修改根据其他人的反馈。

  [Prashant Varanasi]: https://github.com/prashantv
  [Simon Newton]: https://github.com/nomis52

本文档记录了我们在Uber遵循的惯用约定。其中很多是Go的通用准则，而其他扩展准则源于以下外部指南：

1. [Effective Go](https://golang.org/doc/effective_go.html)
2. [The Go common mistakes guide](https://github.com/golang/go/wiki/CodeReviewComments)

所有的代码都应该通过`golint`和`go vet`检测后没有错误。我们建议您将编辑器设置为

- 保存时运行 `goimports`
- 用 `golint` 和 `go vet` 来检测错误

您可以在编辑器支持Go工具页面中找到信息:
<https://github.com/golang/go/wiki/IDEsAndTextEditorPlugins>

## 指导原则

### 指向Interfaces的指针

你几乎不需要指向interface的指针。你应该将interfaces作为值进行传递，本质上传递的底层数据仍然是指针。

一个interface包含两个字段:

1. 一个指向特定类型信息的指针。你可以认为它就是“类型”。
2. 数据指针。如果数据本身就是一个指针, 则直接存储数据本身.如果数据是一个值,则存储指向这个值的指针。

如果你希望interface方法修改底层数据，则必须使用指针。

### Receivers和Interfaces

带有receivers的方法可以通过值或者指针调用

例如,

```go
type S struct {
  data string
}

func (s S) Read() string {
  return s.data
}

func (s *S) Write(str string) {
  s.data = str
}

sVals := map[int]S{1: {"A"}}

// 你只能通过值调用Read
sVals[1].Read()

// 不能被编译:
//  sVals[1].Write("test")

sPtrs := map[int]*S{1: {"A"}}

// 你可以通过指针调用Read和Write
sPtrs[1].Read()
sPtrs[1].Write("test")
```

同样地, 即使是带有值receiver的方法，interface可通过指针来满足条件。

```go
type F interface {
  f()
}

type S1 struct{}

func (s S1) f() {}

type S2 struct{}

func (s *S2) f() {}

s1Val := S1{}
s1Ptr := &S1{}
s2Val := S2{}
s2Ptr := &S2{}

var i F
i = s1Val
i = s1Ptr
i = s2Ptr

// 下面代码无法被编译。因为s2Val是一个值，而S2的f方法没有值receiver
//   i = s2Val
```

Effective Go中有很好的解释[Pointers vs. Values]。

  [Pointers vs. Values]: https://golang.org/doc/effective_go.html#pointers_vs_values

### 零值Mutexes是有效的

`sync.Mutex`和`sync.RWMutex`的零值是有效的, 所以你几乎不需要一个指向mutex的指针。

<table>
<thead><tr><th>坏用法</th><th>好用法</th></tr></thead>
<tbody>
<tr><td>

```go
mu := new(sync.Mutex)
mu.Lock()
```

</td><td>

```go
var mu sync.Mutex
mu.Lock()
```

</td></tr>
</tbody></table>

如果你使用结构体指针, mutex可以作为非指针字段。

通过嵌入mutex，私有结构体中mutex可以作为保护字段使用。

<table>
<tbody>
<tr><td>

```go
type smap struct {
  sync.Mutex // 只用于非导出类型

  data map[string]string
}

func newSMap() *smap {
  return &smap{
    data: make(map[string]string),
  }
}

func (m *smap) Get(k string) string {
  m.Lock()
  defer m.Unlock()

  return m.data[k]
}
```

</td><td>

```go
type SMap struct {
  mu sync.Mutex

  data map[string]string
}

func NewSMap() *SMap {
  return &SMap{
    data: make(map[string]string),
  }
}

func (m *SMap) Get(k string) string {
  m.mu.Lock()
  defer m.mu.Unlock()

  return m.data[k]
}
```

</td></tr>

</tr>
<tr>
<td>嵌入私有类型或者类型实现mutex接口</td>
<td>对于导出类型，使用私有字段。</td>
</tr>

</tbody></table>

### 在边界处拷贝Slices和Maps

slices和maps包含了指向底层数据的指针，所以在它们被复制时要特别小心。

#### 接收Slices和Maps

记住，当map或slice作为参数时，如果你存储了它们的引用，用户修改它们。

<table>
<thead><tr><th>坏用法</th> <th>好用法</th></tr></thead>
<tbody>
<tr>
<td>

```go
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = trips
}

trips := ...
d1.SetTrips(trips)

// 你的意思是修改d1.trips吗?
trips[0] = ...
```

</td>
<td>

```go
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = make([]Trip, len(trips))
  copy(d.trips, trips)
}

trips := ...
d1.SetTrips(trips)

// 我们可以修改trips[0]，但不会影响d1.trips
trips[0] = ...
```

</td>
</tr>

</tbody>
</table>

#### 返回slices或maps

同样，小心map或slice作为内部状态被暴露后，用户可以修改它们。

<table>
<thead><tr><th>坏用法</th><th>好用法</th></tr></thead>
<tbody>
<tr><td>

```go
type Stats struct {
  sync.Mutex

  counters map[string]int
}

// Snapshot返回当前的状态.
func (s *Stats) Snapshot() map[string]int {
  s.Lock()
  defer s.Unlock()

  return s.counters
}

// snapshot不再被锁的保护
snapshot := stats.Snapshot()
```

</td><td>

```go
type Stats struct {
  sync.Mutex

  counters map[string]int
}

func (s *Stats) Snapshot() map[string]int {
  s.Lock()
  defer s.Unlock()

  result := make(map[string]int, len(s.counters))
  for k, v := range s.counters {
    result[k] = v
  }
  return result
}

// Snapshot现在是一个拷贝.
snapshot := stats.Snapshot()
```

</td></tr>
</tbody></table>

### 使用defer做清理

使用defer清理资源，例如文件和锁。

<table>
<thead><tr><th>坏用法</th><th>好用法</th></tr></thead>
<tbody>
<tr><td>

```go
p.Lock()
if p.count < 10 {
  p.Unlock()
  return p.count
}

p.count++
newCount := p.count
p.Unlock()

return newCount

// 有多个return时，很容易遗忘unlock
```

</td><td>

```go
p.Lock()
defer p.Unlock()

if p.count < 10 {
  return p.count
}

p.count++
return p.count

// 更可读
```

</td></tr>
</tbody></table>

Defer开销很小，只有你能证明你的函数是纳秒级别，才应该避免使用。defer可以花费很小的代价换取可读性。尤其是那些不仅仅是简单内存访问的大方法，其他计算的开销远超过 `defer`。

### Channel的size是1或者无缓冲

channel通常size应为1或是无缓冲。默认情况下，channel是无缓冲的，size为零。任何其他size都应该严格审查。确定大小时，应该考虑是什么阻止channel在负载下被填满并阻塞写入，以及发生这种情况时发生了什么。

<table>
<thead><tr><th>坏用法</th><th>好用法</th></tr></thead>
<tbody>
<tr><td>

```go
// 应该满足任何情况!
c := make(chan int, 64)
```

</td><td>

```go
// size为1
c := make(chan int, 1) // or
// 无缓冲channel，size为0
c := make(chan int)
```

</td></tr>
</tbody></table>

### 枚举从1开始

Go引入枚举的标准方法是声明一个自定义类型和一个使用了iota的const组。由于变量默认值为0，因此通常用非零值开始枚举值。

<table>
<thead><tr><th>坏用法</th><th>好用法</th></tr></thead>
<tbody>
<tr><td>

```go
type Operation int

const (
  Add Operation = iota
  Subtract
  Multiply
)

// Add=0, Subtract=1, Multiply=2
```

</td><td>

```go
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
)

// Add=1, Subtract=2, Multiply=3
```

</td></tr>
</tbody></table>

在某些情况下使用零值是有意义的，例如在零值是合适的默认行为时。

```go
type LogOutput int

const (
  LogToStdout LogOutput = iota
  LogToFile
  LogToRemote
)

// LogToStdout=0, LogToFile=1, LogToRemote=2
```

<!-- TODO: section on String methods for enums -->

### 错误类型

有多种声明错误（Error)的方式：

- [`errors.New`] 用于简单静态字符串错误
- [`fmt.Errorf`] 用于格式化字符串错误
- 实现 `Error()` 方法的自定义类型
- 使用 [`"pkg/errors".Wrap`] 封装的错误

返回错误时, 确定最佳方式考虑以下因素:

- 这是一个不需要额外信息的简单错误吗？如果是，[`errors.New`] 足够了。
- 客户需要检测并处理此错误吗？如果是，你应该使用自定义类型并实现 `Error()` 方法。
- 您是否正在传播下游函数返回的错误？如果是，请查看错误包装部分[错误包装](#错误包装(Error-Wrapping))
- 否则 [`fmt.Errorf`] 就可以。

  [`errors.New`]: https://golang.org/pkg/errors/#New
  [`fmt.Errorf`]: https://golang.org/pkg/fmt/#Errorf
  [`"pkg/errors".Wrap`]: https://godoc.org/github.com/pkg/errors#Wrap

如果客户端需要检测错误，并且您已使用创建了一个简单的错误[`errors.New`]，请使用一个错误变量。

<table>
<thead><tr><th>坏用法</th><th>好用法</th></tr></thead>
<tbody>
<tr><td>

```go
// package foo

func Open() error {
  return errors.New("could not open")
}

// package bar

func use() {
  if err := foo.Open(); err != nil {
    if err.Error() == "could not open" {
      // handle
    } else {
      panic("unknown error")
    }
  }
}
```

</td><td>

```go
// package foo

var ErrCouldNotOpen = errors.New("could not open")

func Open() error {
  return ErrCouldNotOpen
}

// package bar

if err := foo.Open(); err != nil {
  if err == foo.ErrCouldNotOpen {
    // handle
  } else {
    panic("unknown error")
  }
}
```

</td></tr>
</tbody></table>

If you have an error that clients may need to detect, and you would like to add
more information to it (e.g., it is not a static string), then you should use a
custom type.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func open(file string) error {
  return fmt.Errorf("file %q not found", file)
}

func use() {
  if err := open(); err != nil {
    if strings.Contains(err.Error(), "not found") {
      // handle
    } else {
      panic("unknown error")
    }
  }
}
```

</td><td>

```go
type errNotFound struct {
  file string
}

func (e errNotFound) Error() string {
  return fmt.Sprintf("file %q not found", e.file)
}

func open(file string) error {
  return errNotFound{file: file}
}

func use() {
  if err := open(); err != nil {
    if _, ok := err.(errNotFound); ok {
      // handle
    } else {
      panic("unknown error")
    }
  }
}
```

</td></tr>
</tbody></table>

Be careful with exporting custom error types directly since they become part of
the public API of the package. It is preferable to expose matcher functions to
check the error instead.

```go
// package foo

type errNotFound struct {
  file string
}

func (e errNotFound) Error() string {
  return fmt.Sprintf("file %q not found", e.file)
}

func IsNotFoundError(err error) bool {
  _, ok := err.(errNotFound)
  return ok
}

func Open(file string) error {
  return errNotFound{file: file}
}

// package bar

if err := foo.Open("foo"); err != nil {
  if foo.IsNotFoundError(err) {
    // handle
  } else {
    panic("unknown error")
  }
}
```

<!-- TODO: Exposing the information to callers with accessor functions. -->

### Error Wrapping

There are three main options for propagating errors if a call fails:

- Return the original error if there is no additional context to add and you
  want to maintain the original error type.
- Add context using [`"pkg/errors".Wrap`] so that the error message provides
  more context and [`"pkg/errors".Cause`] can be used to extract the original
  error.
- Use [`fmt.Errorf`] if the callers do not need to detect or handle that
  specific error case.

It is recommended to add context where possible so that instead of a vague
error such as "connection refused", you get more useful errors such as
"call service foo: connection refused".

When adding context to returned errors, keep the context succinct by avoiding
phrases like "failed to", which state the obvious and pile up as the error
percolates up through the stack:

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "failed to create new store: %s", err)
}
```

</td><td>

```go
s, err := store.New()
if err != nil {
    return fmt.Errorf(
        "new store: %s", err)
}
```

<tr><td>

```
failed to x: failed to y: failed to create new store: the error
```

</td><td>

```
x: y: new store: the error
```

</td></tr>
</tbody></table>

However once the error is sent to another system, it should be clear the
message is an error (e.g. an `err` tag or "Failed" prefix in logs).

See also [Don't just check errors, handle them gracefully].

  [`"pkg/errors".Cause`]: https://godoc.org/github.com/pkg/errors#Cause
  [Don't just check errors, handle them gracefully]: https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully

### Handle Type Assertion Failures

The single return value form of a [type assertion] will panic on an incorrect
type. Therefore, always use the "comma ok" idiom.

  [type assertion]: https://golang.org/ref/spec#Type_assertions

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
t := i.(string)
```

</td><td>

```go
t, ok := i.(string)
if !ok {
  // handle the error gracefully
}
```

</td></tr>
</tbody></table>

<!-- TODO: There are a few situations where the single assignment form is
fine. -->

### Don't Panic

Code running in production must avoid panics. Panics are a major source of
[cascading failures]. If an error occurs, the function must return an error and
allow the caller to decide how to handle it.

  [cascading failures]: https://en.wikipedia.org/wiki/Cascading_failure

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func foo(bar string) {
  if len(bar) == 0 {
    panic("bar must not be empty")
  }
  // ...
}

func main() {
  if len(os.Args) != 2 {
    fmt.Println("USAGE: foo <bar>")
    os.Exit(1)
  }
  foo(os.Args[1])
}
```

</td><td>

```go
func foo(bar string) error {
  if len(bar) == 0
    return errors.New("bar must not be empty")
  }
  // ...
  return nil
}

func main() {
  if len(os.Args) != 2 {
    fmt.Println("USAGE: foo <bar>")
    os.Exit(1)
  }
  if err := foo(os.Args[1]); err != nil {
    panic(err)
  }
}
```

</td></tr>
</tbody></table>

Panic/recover is not an error handling strategy. A program must panic only when
something irrecoverable happens such as a nil dereference. An exception to this is
program initialization: bad things at program startup that should abort the
program may cause panic.

```go
var _statusTemplate = template.Must(template.New("name").Parse("_statusHTML"))
```

Even in tests, prefer `t.Fatal` or `t.FailNow` over panics to ensure that the
test is marked as failed.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// func TestFoo(t *testing.T)

f, err := ioutil.TempFile("", "test")
if err != nil {
  panic("failed to set up test")
}
```

</td><td>

```go
// func TestFoo(t *testing.T)

f, err := ioutil.TempFile("", "test")
if err != nil {
  t.Fatal("failed to set up test")
}
```

</td></tr>
</tbody></table>

<!-- TODO: Explain how to use _test packages. -->

### Use go.uber.org/atomic

Atomic operations with the [sync/atomic] package operate on the raw types
(`int32`, `int64`, etc.) so it is easy to forget to use the atomic operation to
read or modify the variables.

[go.uber.org/atomic] adds type safety to these operations by hiding the
underlying type. Additionally, it includes a convenient `atomic.Bool` type.

  [go.uber.org/atomic]: https://godoc.org/go.uber.org/atomic
  [sync/atomic]: https://golang.org/pkg/sync/atomic/

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type foo struct {
  running int32  // atomic
}

func (f* foo) start() {
  if atomic.SwapInt32(&f.running, 1) == 1 {
     // already running…
     return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running == 1  // race!
}
```

</td><td>

```go
type foo struct {
  running atomic.Bool
}

func (f *foo) start() {
  if f.running.Swap(true) {
     // already running…
     return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running.Load()
}
```

</td></tr>
</tbody></table>

## Performance

Performance-specific guidelines apply only to the hot path.

### Prefer strconv over fmt

When converting primitives to/from strings, `strconv` is faster than
`fmt`.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
for i := 0; i < b.N; i++ {
  s := fmt.Sprint(rand.Int())
}
```

</td><td>

```go
for i := 0; i < b.N; i++ {
  s := strconv.Itoa(rand.Int())
}
```

</td></tr>
<tr><td>

```
BenchmarkFmtSprint-4    143 ns/op    2 allocs/op
```

</td><td>

```
BenchmarkStrconv-4    64.2 ns/op    1 allocs/op
```

</td></tr>
</tbody></table>

### Avoid string-to-byte conversion

Do not create byte slices from a fixed string repeatedly. Instead, perform the
conversion once and capture the result.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
for i := 0; i < b.N; i++ {
  w.Write([]byte("Hello world"))
}
```

</td><td>

```go
data := []byte("Hello world")
for i := 0; i < b.N; i++ {
  w.Write(data)
}
```

</tr>
<tr><td>

```
BenchmarkBad-4   50000000   22.2 ns/op
```

</td><td>

```
BenchmarkGood-4  500000000   3.25 ns/op
```

</td></tr>
</tbody></table>

## Style

### Group Similar Declarations

Go supports grouping similar declarations.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
import "a"
import "b"
```

</td><td>

```go
import (
  "a"
  "b"
)
```

</td></tr>
</tbody></table>

This also applies to constants, variables, and type declarations.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go

const a = 1
const b = 2



var a = 1
var b = 2



type Area float64
type Volume float64
```

</td><td>

```go
const (
  a = 1
  b = 2
)

var (
  a = 1
  b = 2
)

type (
  Area float64
  Volume float64
)
```

</td></tr>
</tbody></table>

Only group related declarations. Do not group declarations that are unrelated.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
  ENV_VAR = "MY_ENV"
)
```

</td><td>

```go
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
)

const ENV_VAR = "MY_ENV"
```

</td></tr>
</tbody></table>

Groups are not limited in where they can be used. For example, you can use them
inside of functions.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func f() string {
  var red = color.New(0xff0000)
  var green = color.New(0x00ff00)
  var blue = color.New(0x0000ff)

  ...
}
```

</td><td>

```go
func f() string {
  var (
    red   = color.New(0xff0000)
    green = color.New(0x00ff00)
    blue  = color.New(0x0000ff)
  )

  ...
}
```

</td></tr>
</tbody></table>

### Import Group Ordering

There should be two import groups:

- Standard library
- Everything else

This is the grouping applied by goimports by default.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
import (
  "fmt"
  "os"
  "go.uber.org/atomic"
  "golang.org/x/sync/errgroup"
)
```

</td><td>

```go
import (
  "fmt"
  "os"

  "go.uber.org/atomic"
  "golang.org/x/sync/errgroup"
)
```

</td></tr>
</tbody></table>

### Package Names

When naming packages, choose a name that is:

- All lower-case. No capitals or underscores.
- Does not need to be renamed using named imports at most call sites.
- Short and succinct. Remember that the name is identified in full at every call
  site.
- Not plural. For example, `net/url`, not `net/urls`.
- Not "common", "util", "shared", or "lib". These are bad, uninformative names.

See also [Package Names] and [Style guideline for Go packages].

  [Package Names]: https://blog.golang.org/package-names
  [Style guideline for Go packages]: https://rakyll.org/style-packages/

### Function Names

We follow the Go community's convention of using [MixedCaps for function
names]. An exception is made for test functions, which may contain underscores
for the purpose of grouping related test cases, e.g.,
`TestMyFunction_WhatIsBeingTested`.

  [MixedCaps for function names]: https://golang.org/doc/effective_go.html#mixed-caps

### Import Aliasing

Import aliasing must be used if the package name does not match the last
element of the import path.

```go
import (
  "net/http"

  client "example.com/client-go"
  trace "example.com/trace/v2"
)
```

In all other scenarios, import aliases should be avoided unless there is a
direct conflict between imports.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
import (
  "fmt"
  "os"


  nettrace "golang.net/x/trace"
)
```

</td><td>

```go
import (
  "fmt"
  "os"
  "runtime/trace"

  nettrace "golang.net/x/trace"
)
```

</td></tr>
</tbody></table>

### Function Grouping and Ordering

- Functions should be sorted in rough call order.
- Functions in a file should be grouped by receiver.

Therefore, exported functions should appear first in a file, after
`struct`, `const`, `var` definitions.

A `newXYZ()`/`NewXYZ()` may appear after the type is defined, but before the
rest of the methods on the receiver.

Since functions are grouped by receiver, plain utility functions should appear
towards the end of the file.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func (s *something) Cost() {
  return calcCost(s.weights)
}

type something struct{ ... }

func calcCost(n []int) int {...}

func (s *something) Stop() {...}

func newSomething() *something {
    return &something{}
}
```

</td><td>

```go
type something struct{ ... }

func newSomething() *something {
    return &something{}
}

func (s *something) Cost() {
  return calcCost(s.weights)
}

func (s *something) Stop() {...}

func calcCost(n []int) int {...}
```

</td></tr>
</tbody></table>

### Reduce Nesting

Code should reduce nesting where possible by handling error cases/special
conditions first and returning early or continuing the loop. Reduce the amount
of code that is nested multiple levels.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
for _, v := range data {
  if v.F1 == 1 {
    v = process(v)
    if err := v.Call(); err == nil {
      v.Send()
    } else {
      return err
    }
  } else {
    log.Printf("Invalid v: %v", v)
  }
}
```

</td><td>

```go
for _, v := range data {
  if v.F1 != 1 {
    log.Printf("Invalid v: %v", v)
    continue
  }

  v = process(v)
  if err := v.Call(); err != nil {
    return err
  }
  v.Send()
}
```

</td></tr>
</tbody></table>

### Unnecessary Else

If a variable is set in both branches of an if, it can be replaced with a
single if.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var a int
if b {
  a = 100
} else {
  a = 10
}
```

</td><td>

```go
a := 10
if b {
  a = 100
}
```

</td></tr>
</tbody></table>

### Top-level Variable Declarations

At the top level, use the standard `var` keyword. Do not specify the type,
unless it is not the same type as the expression.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var _s string = F()

func F() string { return "A" }
```

</td><td>

```go
var _s = F()
// Since F already states that it returns a string, we don't need to specify
// the type again.

func F() string { return "A" }
```

</td></tr>
</tbody></table>

Specify the type if the type of the expression does not match the desired type
exactly.

```go
type myError struct{}

func (myError) Error() string { return "error" }

func F() myError { return myError{} }

var _e error = F()
// F returns an object of type myError but we want error.
```

### Prefix Unexported Globals with _

Prefix unexported top-level `var`s and `const`s with `_` to make it clear when
they are used that they are global symbols.

Exception: Unexported error values, which should be prefixed with `err`.

Rationale: Top-level variables and constants have a package scope. Using a
generic name makes it easy to accidentally use the wrong value in a different
file.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// foo.go

const (
  defaultPort = 8080
  defaultUser = "user"
)

// bar.go

func Bar() {
  defaultPort := 9090
  ...
  fmt.Println("Default port", defaultPort)

  // We will not see a compile error if the first line of
  // Bar() is deleted.
}
```

</td><td>

```go
// foo.go

const (
  _defaultPort = 8080
  _defaultUser = "user"
)
```

</td></tr>
</tbody></table>

### Embedding in Structs

Embedded types (such as mutexes) should be at the top of the field list of a
struct, and there must be an empty line separating embedded fields from regular
fields.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Client struct {
  version int
  http.Client
}
```

</td><td>

```go
type Client struct {
  http.Client

  version int
}
```

</td></tr>
</tbody></table>

### Use Field Names to initialize Structs

You should almost always specify field names when initializing structs. This is
now enforced by [`go vet`].

  [`go vet`]: https://golang.org/cmd/vet/

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
k := User{"John", "Doe", true}
```

</td><td>

```go
k := User{
    FirstName: "John",
    LastName: "Doe",
    Admin: true,
}
```

</td></tr>
</tbody></table>

Exception: Field names *may* be omitted in test tables when there are 3 or
fewer fields.

```go
tests := []struct{
  op Operation
  want string
}{
  {Add, "add"},
  {Subtract, "subtract"},
}
```

### Local Variable Declarations

Short variable declarations (`:=`) should be used if a variable is being set to
some value explicitly.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var s = "foo"
```

</td><td>

```go
s := "foo"
```

</td></tr>
</tbody></table>

However, there are cases where the default value is clearer when the `var`
keyword is use. [Declaring Empty Slices], for example.

  [Declaring Empty Slices]: https://github.com/golang/go/wiki/CodeReviewComments#declaring-empty-slices

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func f(list []int) {
  filtered := []int{}
  for _, v := range list {
    if v > 10 {
      filtered = append(filtered, v)
    }
  }
}
```

</td><td>

```go
func f(list []int) {
  var filtered []int
  for _, v := range list {
    if v > 10 {
      filtered = append(filtered, v)
    }
  }
}
```

</td></tr>
</tbody></table>

### nil is a valid slice

`nil` is a valid slice of length 0. This means that,

- You should not return a slice of length zero explicitly. Return `nil`
  instead.

  <table>
  <thead><tr><th>Bad</th><th>Good</th></tr></thead>
  <tbody>
  <tr><td>

  ```go
  if x == "" {
    return []int{}
  }
  ```

  </td><td>

  ```go
  if x == "" {
    return nil
  }
  ```

  </td></tr>
  </tbody></table>

- To check if a slice is empty, always use `len(s) == 0`. Do not check for
  `nil`.

  <table>
  <thead><tr><th>Bad</th><th>Good</th></tr></thead>
  <tbody>
  <tr><td>

  ```go
  func isEmpty(s []string) bool {
    return s == nil
  }
  ```

  </td><td>

  ```go
  func isEmpty(s []string) bool {
    return len(s) == 0
  }
  ```

  </td></tr>
  </tbody></table>

- The zero value (a slice declared with `var`) is usable immediately without
  `make()`.

  <table>
  <thead><tr><th>Bad</th><th>Good</th></tr></thead>
  <tbody>
  <tr><td>

  ```go
  nums := []int{}
  // or, nums := make([]int)

  if add1 {
    nums = append(nums, 1)
  }

  if add2 {
    nums = append(nums, 2)
  }
  ```

  </td><td>

  ```go
  var nums []int

  if add1 {
    nums = append(nums, 1)
  }

  if add2 {
    nums = append(nums, 2)
  }
  ```

  </td></tr>
  </tbody></table>

### Reduce Scope of Variables

Where possible, reduce scope of variables. Do not reduce the scope if it
conflicts with [Reduce Nesting](#reduce-nesting).

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
err := ioutil.WriteFile(name, data, 0644)
if err != nil {
 return err
}
```

</td><td>

```go
if err := ioutil.WriteFile(name, data, 0644); err != nil {
 return err
}
```

</td></tr>
</tbody></table>

If you need a result of a function call outside of the if, then you should not
try to reduce the scope.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
if data, err := ioutil.ReadFile(name); err == nil {
  err = cfg.Decode(data)
  if err != nil {
    return err
  }

  fmt.Println(cfg)
  return nil
} else {
  return err
}
```

</td><td>

```go
data, err := ioutil.ReadFile(name)
if err != nil {
   return err
}

if err := cfg.Decode(data); err != nil {
  return err
}

fmt.Println(cfg)
return nil
```

</td></tr>
</tbody></table>

### Avoid Naked Parameters

Naked parameters in function calls can hurt readability. Add C-style comments
(`/* ... */`) for parameter names when their meaning is not obvious.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true, true)
```

</td><td>

```go
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true /* isLocal */, true /* done */)
```

</td></tr>
</tbody></table>

Better yet, replace naked `bool` types with custom types for more readable and
type-safe code. This allows more than just two states (true/false) for that
parameter in the future.

```go
type Region int

const (
  UnknownRegion Region = iota
  Local
)

type Status int

const (
  StatusReady = iota + 1
  StatusDone
  // Maybe we will have a StatusInProgress in the future.
)

func printInfo(name string, region Region, status Status)
```

### Use Raw String Literals to Avoid Escaping

Go supports [raw string literals](https://golang.org/ref/spec#raw_string_lit),
which can span multiple lines and include quotes. Use these to avoid
hand-escaped strings which are much harder to read.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
wantError := "unknown name:\"test\""
```

</td><td>

```go
wantError := `unknown error:"test"`
```

</td></tr>
</tbody></table>

### Initializing Struct References

Use `&T{}` instead of `new(T)` when initializing struct references so that it
is consistent with the struct initialization.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
sval := T{Name: "foo"}

// inconsistent
sptr := new(T)
sptr.Name = "bar"
```

</td><td>

```go
sval := T{Name: "foo"}

sptr := &T{Name: "bar"}
```

</td></tr>
</tbody></table>

### Format Strings outside Printf

If you declare format strings for `Printf`-style functions outside a string
literal, make them `const` values.

This helps `go vet` perform static analysis of the format string.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
msg := "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

</td><td>

```go
const msg = "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

</td></tr>
</tbody></table>

### Naming Printf-style Functions

When you declare a `Printf`-style function, make sure that `go vet` can detect
it and check the format string.

This means that you should use pre-defined `Printf`-style function
names if possible. `go vet` will check these by default. See [Printf family]
for more information.

  [Printf family]: https://golang.org/cmd/vet/#hdr-Printf_family

If using the pre-defined names is not an option, end the name you choose with
f: `Wrapf`, not `Wrap`. `go vet` can be asked to check specific `Printf`-style
names but they must end with f.

```shell
$ go vet -printfuncs=wrapf,statusf
```

See also [go vet: Printf family check].

  [go vet: Printf family check]: https://kuzminva.wordpress.com/2017/11/07/go-vet-printf-family-check/

## Patterns

### Test Tables

Use table-driven tests with [subtests] to avoid duplicating code when the core
test logic is repetitive.

  [subtests]: https://blog.golang.org/subtests

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// func TestSplitHostPort(t *testing.T)

host, port, err := net.SplitHostPort("192.0.2.0:8000")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("192.0.2.0:http")
require.NoError(t, err)
assert.Equal(t, "192.0.2.0", host)
assert.Equal(t, "http", port)

host, port, err = net.SplitHostPort(":8000")
require.NoError(t, err)
assert.Equal(t, "", host)
assert.Equal(t, "8000", port)

host, port, err = net.SplitHostPort("1:8")
require.NoError(t, err)
assert.Equal(t, "1", host)
assert.Equal(t, "8", port)
```

</td><td>

```go
// func TestSplitHostPort(t *testing.T)

tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  {
    give:     "192.0.2.0:8000",
    wantHost: "192.0.2.0",
    wantPort: "8000",
  },
  {
    give:     "192.0.2.0:http",
    wantHost: "192.0.2.0",
    wantPort: "http",
  },
  {
    give:     ":8000",
    wantHost: "",
    wantPort: "8000",
  },
  {
    give:     "1:8",
    wantHost: "1",
    wantPort: "8",
  },
}

for _, tt := range tests {
  t.Run(tt.give, func(t *testing.T) {
    host, port, err := net.SplitHostPort(tt.give)
    require.NoError(t, err)
    assert.Equal(t, tt.wantHost, host)
    assert.Equal(t, tt.wantPort, port)
  })
}
```

</td></tr>
</tbody></table>

Test tables make it easier to add context to error messages, reduce duplicate
logic, and add new test cases.

We follow the convention that the slice of structs is referred to as `tests`
and each test case `tt`. Further, we encourage explicating the input and output
values for each test case with `give` and `want` prefixes.

```go
tests := []struct{
  give     string
  wantHost string
  wantPort string
}{
  // ...
}

for _, tt := range tests {
  // ...
}
```

### Functional Options

Functional options is a pattern in which you declare an opaque `Option` type
that records information in some internal struct. You accept a variadic number
of these options and act upon the full information recorded by the options on
the internal struct.

Use this pattern for optional arguments in constructors and other public APIs
that you foresee needing to expand, especially if you already have three or
more arguments on those functions.

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
// package db

func Connect(
  addr string,
  timeout time.Duration,
  caching bool,
) (*Connection, error) {
  // ...
}

// Timeout and caching must always be provided,
// even if the user wants to use the default.

db.Connect(addr, db.DefaultTimeout, db.DefaultCaching)
db.Connect(addr, newTimeout, db.DefaultCaching)
db.Connect(addr, db.DefaultTimeout, false /* caching */)
db.Connect(addr, newTimeout, false /* caching */)
```

</td><td>

```go
type options struct {
  timeout time.Duration
  caching bool
}

// Option overrides behavior of Connect.
type Option interface {
  apply(*options)
}

type optionFunc func(*options)

func (f optionFunc) apply(o *options) {
  f(o)
}

func WithTimeout(t time.Duration) Option {
  return optionFunc(func(o *options) {
    o.timeout = t
  })
}

func WithCaching(cache bool) Option {
  return optionFunc(func(o *options) {
    o.caching = cache
  })
}

// Connect creates a connection.
func Connect(
  addr string,
  opts ...Option,
) (*Connection, error) {
  options := options{
    timeout: defaultTimeout,
    caching: defaultCaching,
  }

  for _, o := range opts {
    o.apply(&options)
  }

  // ...
}

// Options must be provided only if needed.

db.Connect(addr)
db.Connect(addr, db.WithTimeout(newTimeout))
db.Connect(addr, db.WithCaching(false))
db.Connect(
  addr,
  db.WithCaching(false),
  db.WithTimeout(newTimeout),
)
```

</td></tr>
</tbody></table>

See also,

- [Self-referential functions and the design of options]
- [Functional options for friendly APIs]

  [Self-referential functions and the design of options]: https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html
  [Functional options for friendly APIs]: https://dave.cheney.net/2014/10/17/functional-options-for-friendly-apis

<!-- TODO: replace this with parameter structs and functional options, when to
use one vs other -->
