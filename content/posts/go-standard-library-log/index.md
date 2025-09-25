+++
date = '2025-09-20T20:56:36+08:00'
draft = false
title = 'Go标准库 log 源码与原理解析'
slug = 'golang-std-lib-log-source-analysis'
description = '对 Go v1.25.1 的 log 标准库源码进行分析，涵盖核心 Logger 类型、日志前缀标志位、标准 Logger、日志输出的完整流程，帮助读者理解 log 包的设计与实现原理。'
keywords = ["Go", "Golang", "标准库", "log", "日志", "源码解析", "Logger", "isDiscard", "calldepth"]
categories = ["Go源码学习"]
tags = ["Go", "log", "源码解析"]
+++

📌说明：本文基于 Go v1.25.1 源码解析，后续版本可能有所调整

## 简介
log 包是标准库中提供的一个简单的日志工具，可以将程序运行中产生的信息输出到控制台或文件等位置，便于调试和排查问题，它的主要作用可以概括为三点：
1. 输出日志
  - 默认情况下，日志会被输出到标准错误 stderr，并带有时间戳
  - 每条日志独占一行，如果日志正文（传入的参数）结尾没有换行符，那么会自动加上换行
  ```go
	log.Print("Hello, World!")
	// 2025/09/20 22:55:51 Hello, World!
  ```
2. 提供三类日志函数
  - `log.Print[f|ln]`：仅输出日志
  - `log.Panic[f|ln]`：等价于执行 `Print[f|ln]` 后触发 Panic，会执行 `defer`
  - `log.Fatal[f|ln]`：等价于执行 `Print[f|ln]` 后调用 `os.Exit(1)` 直接退出进程，不会执行 `defer`
3. 支持自定义输出位置与日志内容
  - `log.SetOutput`： 控制将日志写入何种实现了 `io.Writer` 接口的目标
  - `log.SetFlags`： 设置自动生成的前缀信息（时间戳、文件路径、行号等）
  - `log.SetPrefix`： 设置用户自定义的前缀文本，默认添加在每条日志的头部：
  ```go
	log.SetPrefix("LOG: ")
	log.Printf("Hello, World!")
	// LOG: 2025/09/20 23:50:11 Hello, World!
  ```

## 日志前缀配置
`log` 包里定义了一组标志位，用于控制日志自动生成的前缀信息，它们可以用按位或运算进行组合，并通过 `log.SetFlags` 函数进行设置，这些标志位包括：
```go
const (
    // 在日志前加上本地日期，格式为 2009/01/23
    Ldate         = 1 << iota
    // 在日志前加上本地时间，格式为 01:23:23
    Ltime
    // 在时间后加上微秒，格式为 01:23:23.123456，添加此标志时自动添加 Ltime 标志
    Lmicroseconds

    // 在日志前加上完整的文件路径与行号，格式为 /a/b/c/d.go:23
    Llongfile
    // 在日志前仅加上文件名与行号，格式为 d.go:23，会覆盖 Llongfile 标志
    Lshortfile

    // 使用 UTC 时区，而不是本地时区
    LUTC

    // 将 SetPrefix 定义的前缀文本放置在日志正文前，而不是日志头部，即：
    // 未设置此标志位时，日志的展示顺序为：用户自定义前缀 -> Flags生成的前缀 -> 日志正文
    // 设置此标志位后，日志的展示顺序为：Flags生成的前缀 -> 用户自定义前缀 -> 日志正文
    Lmsgprefix

    // 默认设置
    LstdFlags     = Ldate | Ltime
)
```

## Logger
### 类型定义
`log` 包的核心是 `Logger` 类型，该类型拥有 `Print[f|ln]` 等日志函数与 `SetOutput` 等自定义函数，通过这些方法来格式化输出日志：
```go
type Logger struct {
    outMu sync.Mutex // 确保多个 goroutine 并发安全地访问 Writer
    out   io.Writer  // 输出日志的位置

    prefix    atomic.Pointer[string]  // 用户自定义前缀
    flag      atomic.Int32            // 控制日志前缀信息的标志位
    isDiscard atomic.Bool             // 快速判断 out 字段是否为 io.Discard
}

func New(out io.Writer, prefix string, flag int) *Logger {
    l := new(Logger)
    l.SetOutput(out)
    l.SetPrefix(prefix)
    l.SetFlags(flag)
    return l
}
```

### isDiscard
`io.Discard` 可以看作是 `io` 包中实现了 `io.Writer` 接口的“黑洞”写入器，它接收数据后什么也不做，并且永远返回写入成功：
```go
package io

var Discard Writer = discard{}

type discard struct{}
func (discard) Write(p []byte) (int, error) {
    return len(p), nil
}
```

`isDiscard` 字段用来记录当前 Logger 的输出位置是否为 `io.Discard`，在 `SetOutput` 函数中会将传入的 `io.Writer` 与 `io.Discard` 进行比较，作为 `isDiscard` 的值。当 `isDiscard==true` 时，日志输出的核心路径会尽早返回，避免后续加锁、构造前缀、实际输出等开销。
```go
func (l *Logger) SetOutput(w io.Writer) {
	l.outMu.Lock()
	defer l.outMu.Unlock()
	l.out = w
	l.isDiscard.Store(w == io.Discard)
}
```

### 标准Logger
除了手动初始化 Logger 之外，`log` 包提供了一个名为 `std` 的默认的「标准 Logger」：
```go
package log

var std = New(os.Stderr, "", LstdFlags)

// Default returns the standard logger used by the package-level output functions.
func Default() *Logger { return std }
```
这个 `std` 会将日志输出到 stderr，没有用户自定义前缀文本，并且会为日志添加本地日期和时间的前缀，log 包级别的日志函数（如 `log.Print`）本质上都是调用 `std` 上的方法，这揭示了上文中描述的 `log` 包的默认行为是怎么来的。

此外，也可以调用 `log.Default()` 函数获取 `std` 这个标准 Logger，并在其上调用日志输出函数，因此，调用 `log.Print` 与调用 `log.Default().Print` 本质上是一样的，前者可以看作是一种语法糖，便于开发者使用。

## 日志输出的完整路径
无论是自定义 Logger，还是标准 Logger，调用日志输出函数后都会走到 `Logger.output` 函数中，因此下文以 `Logger.Print` 函数为例，描述日志输出的完整路径：
```go
// Print 函数的参数格式与 fmt.Print 一致
func (l *Logger) Print(v ...any) {
	l.output(0, 2, func(b []byte) []byte {
		return fmt.Append(b, v...)
	})
}
```

接下来是核心的 `Logger.output` 函数：
```go
// pc 和 calldepth 参数都可以用于获取日志的源文件路径与行数
// appendOutput 用于向缓冲区中追加日志正文，因为需要先写入日志前缀
func (l *Logger) output(pc uintptr, calldepth int, appendOutput func([]byte) []byte) error {
	// 如果日志会被输出到 io.Discard，那么直接返回，避免后续开销
	if l.isDiscard.Load() {
		return nil
	}

	now := time.Now()
	prefix := l.Prefix()
	flag := l.Flags()

	// 根据 pc 或 calldepth 调用不同的函数获取日志的源文件路径与行数
	var file string
	var line int
	if flag&(Lshortfile|Llongfile) != 0 {
		if pc == 0 {
			var ok bool
			_, file, line, ok = runtime.Caller(calldepth)
			if !ok {
				file = "???"
				line = 0
			}
		} else {
			fs := runtime.CallersFrames([]uintptr{pc})
			f, _ := fs.Next()
			file = f.File
			if file == "" {
				file = "???"
			}
			line = f.Line
		}
	}

	// 使用 sync.Pool 保存可复用的缓冲区，减小频繁的内存分配与回收压力
	// 这里的 buf 是 *[]byte 类型，目的是避免每次取出缓冲区时的 slice header 复制
	buf := getBuffer()
	defer putBuffer(buf)
	// 首先构造日志前缀写入缓冲区，再追加日志正文
	// formatHeader 函数的逻辑相对简单
	formatHeader(buf, now, prefix, flag, file, line)
	*buf = appendOutput(*buf)
	if len(*buf) == 0 || (*buf)[len(*buf)-1] != '\n' {
		*buf = append(*buf, '\n')
	}

	l.outMu.Lock()
	defer l.outMu.Unlock()
	_, err := l.out.Write(*buf)
	return err
}
```

根据对 `output` 函数的梳理，这里可以对 `isDiscard` 字段的避免开销的作用做一些补充：日志输出函数 `log.Print` 等会首先使用 `fmt` 包对输入参数进行格式化，然后才会调用 `output` 函数，也就是说，`isDiscard` 字段只能跳过 `output` 中的逻辑，而对于输入参数格式化，比如说将某个 `cost()` 函数的结果作为输入参数，这部分开销是无法避免的。

### calldepth
如果设置了 `Lshortfile` 或 `Llongfile`，那么后续会通过 `runtime.Caller(calldepth)` 来获取调用源信息，`calldepth` 决定了调用栈回溯的层数，
用于确定应该输出哪一个调用函数的路径与行号，值为 0 时表示回溯到 `runtime.Caller` 的调用方，以此类推。

假设有以下的简单代码：
```go
package main

import "log"

func test() {
	log.SetFlags(log.LstdFlags | log.Llongfile)
	log.Print("Hello")
}

func main() {
	test()
}
```
它的函数调用栈从栈顶向栈底可以简单描述为：
```text
runtime.Caller
log.(*Logger).output
log.Print
main.test
main.main
runtime.main
```
那么：
- 如果 `calldepth=0`，则输出 `log.(*Logger).output` 函数中调用 `Caller` 的位置
- 如果 `calldepth=1`，则输出 `log.Print` 函数中调用 `output` 的位置
- 如果 `calldepth=2`，则输出用户程序中调用 `log.Print` 的位置（即 `main.go:7`），这通常是我们期望的输出内容，
因此 `log` 包中的日志输出函数都会将 `calldepth` 设置为 2

另外，`log` 包还提供了一个支持用户自定义 `calldepth` 的 `Logger.Output` 函数：
```go
func (l *Logger) Output(calldepth int, s string) error {
	return l.output(0, calldepth+1, func(b []byte) []byte { // +1 for this frame.
		return append(b, s...)
	})
}
```
