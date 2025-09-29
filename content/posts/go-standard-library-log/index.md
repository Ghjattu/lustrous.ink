+++
date = '2025-09-26T11:27:11+08:00'
lastmod = '2025-09-26T11:27:11+08:00'
draft = false
title = 'Go标准库 log 源码与原理解析'
slug = 'golang-std-lib-log-source-analysis'
description = '对Go v1.25.1的log标准库源码进行分析，涵盖核心Logger类型、日志前缀标志位、标准Logger、日志输出的完整流程'
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
  - `log.SetFlags`： 设置自动生成的前缀（时间戳、文件路径、行号等）
  - `log.SetPrefix`： 设置用户自定义的前缀，默认添加在每条日志的头部：
  ```go
	log.SetPrefix("LOG: ")
	log.Printf("Hello, World!")
	// LOG: 2025/09/20 23:50:11 Hello, World!
  ```

## 日志前缀配置
### SetFlags 与标志位
`log` 包定义了一组标志位，用于控制自动生成的前缀，多个标志可以用按位或运算进行组合：
| 标志位 | 描述 | 示例 |
|---|---|---|
| Ldate | 本地日期 | 2009/01/23 |
| Ltime | 本地时间 | 01:23:23 |
| Lmicroseconds | 微秒 | 01:23:23.123456 |
| Llongfile | 完整文件路径与行号 | /a/b/c/d.go:23 |
| Lshortfile | 文件名与行号, 覆盖 Llongfile | d.go:23 |
| LUTC | 使用 UTC 时间而不是本地时间 | - |
| Lmsgprefix | 控制用户自定义前缀的位置 | - |
| LstdFlags | 默认标志, 等价于 Ldate \| Ltime | - |

### SetPrefix 与 Lmsgprefix
如上文所述，`log.SetPrefix` 用于设置用户自定义的日志前缀文本，默认情况下，这个前缀会被添加在整条日志的头部，即日志的顺序为：用户自定义前缀 -> Flags 生成的前缀 -> 日志正文。

如果设置了 `Lmsgprefix` 标志位，那么日志顺序会变为：Flags 生成的前缀 -> 用户自定义前缀 -> 日志正文。


## Logger 类型与标准 Logger
### Logger 类型定义
`Logger` 是 `log` 包的核心结构，所有的日志输出函数都是该类型上的方法：
```go
type Logger struct {
    outMu sync.Mutex // 确保多个 goroutine 并发安全地访问 Writer
    out   io.Writer  // 日志输出目标

    prefix    atomic.Pointer[string]  // 用户自定义前缀
    flag      atomic.Int32            // 日志前缀信息标志位
    isDiscard atomic.Bool             // 日志输出目标是否为 io.Discard
}

func New(out io.Writer, prefix string, flag int) *Logger {
    l := new(Logger)
    l.SetOutput(out)
    l.SetPrefix(prefix)
    l.SetFlags(flag)
    return l
}
```

### isDiscard 的作用
`io.Discard` 可以看作是一个“黑洞 Writer“，它接收数据后什么也不做，并且永远返回写入成功：
```go
package io

var Discard Writer = discard{}

type discard struct{}
func (discard) Write(p []byte) (int, error) {
    return len(p), nil
}
```

`Logger.isDiscard` 字段会在 `SetOutput` 函数中被赋予值，记录日志输出目标是否为 `io.Discard`，
若为 `true`，则表示日志不会被实际输出，因此在日志输出的核心路径中可以尽早返回，避免后续的加锁、构造前缀、实际 I/O 等开销，从而提升性能。
```go
func (l *Logger) SetOutput(w io.Writer) {
	l.outMu.Lock()
	defer l.outMu.Unlock()
	l.out = w
	l.isDiscard.Store(w == io.Discard)
}
```

不过需要注意的是，`isDiscard` 只能避免日志输出路径的后续开销，而无法避免传入参数的计算开销，例如代码 `log.Print("Result:", cost())` 中的 `cost()` 函数仍然会被执行。

### 标准Logger
除了手动初始化 Logger 之外，`log` 包预定义了一个全局的标准 Logger：
```go
var std = New(os.Stderr, "", LstdFlags)

// Default returns the standard logger used by the package-level output functions.
func Default() *Logger { return std }
```
标准 Logger `std` 默认将日志输出到 stderr，没有用户自定义前缀，并且自动为日志生成本地日期与时间的前缀，`log` 包级别的日志函数（如 `log.Print`）本质上都是调用 `std` 上的方法，这揭示了上文中描述的 `log` 包的默认行为是怎么来的。

此外，也可以调用 `log.Default()` 函数获取 `std`，因此调用 `log.Print` 与调用 `log.Default().Print` 本质上是一样的，前者可以看作是一种语法糖。

## 日志输出流程解析
无论是自定义 Logger，还是标准 Logger，调用日志输出函数后都会走到 `Logger.output` 函数中，因此下文以 `Logger.Print` 函数为例，描述日志输出的完整路径：
```go
// Print 函数的参数格式与 fmt.Print 一致
func (l *Logger) Print(v ...any) {
	l.output(0, 2, func(b []byte) []byte {
		return fmt.Append(b, v...)
	})
}
```

### Logger.output 核心逻辑
`Logger.output` 函数简化后的关键流程为：
```go
// pc 和 calldepth 参数都可以用于获取日志的源文件路径与行数
// 目前包中的所有日志输出函数均将 pc 设为 0
// appendOutput 用于向缓冲区中追加日志正文，因为需要先写入日志前缀
func (l *Logger) output(pc uintptr, calldepth int, appendOutput func([]byte) []byte) error {
	// 如果日志输出目标是 io.Discard，那么直接返回，避免后续开销
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
			// pc 参数目前均为 0，这里暂不展开
		}
	}

	// 使用 sync.Pool 保存可复用的缓冲区，减小频繁的内存分配与回收压力
	// 这里的 buf 是 *[]byte 类型，目的是避免每次取出缓冲区时的 slice header 复制
	buf := getBuffer()
	defer putBuffer(buf)
	// 首先构造日志前缀写入缓冲区，再追加日志正文，最后自动补换行符
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

<!-- 根据对 `output` 函数的梳理，这里可以对 `isDiscard` 字段的避免开销的作用做一些补充：日志输出函数 `log.Print` 等会首先使用 `fmt` 包对输入参数进行格式化，然后才会调用 `output` 函数，也就是说，`isDiscard` 字段只能跳过 `output` 中的逻辑，而对于输入参数格式化，比如说将某个 `cost()` 函数的结果作为输入参数，这部分开销是无法避免的。 -->

### calldepth 参数
`calldepth` 参数表示需要回溯多少层函数调用栈来获取日志定位的文件路径与行号，值为 0 时表示回溯到 `runtime.Caller` 的调用方，以此类推。

举个简单例子，假设有以下代码：
```go
package main

import "log"

func test() {
	log.SetFlags(log.LstdFlags | log.Llongfile)
	log.Print("Hello") // Line 7
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
- 如果 `calldepth=2`，则输出用户程序中调用 `log.Print` 的位置（即 `main.go:7`），这通常是我们期望的日志定位，因此 `log` 包中的日志输出函数都会将 `calldepth` 设置为 2

另外，`log` 包还提供了一个支持用户自定义 `calldepth` 的 `Logger.Output` 函数：
```go
func (l *Logger) Output(calldepth int, s string) error {
	return l.output(0, calldepth+1, func(b []byte) []byte { // +1 for this frame.
		return append(b, s...)
	})
}
```

## 总结
本文基于 Go v1.25.1 源码解析了 `log` 标准库的实现原理：
- `Logger` 是核心结构，封装了日志输出前缀、目标与并发安全机制
- `Logger.isDiscard` 字段提供了一定程度上的性能优化
- 标准 Logger `std` 提供了默认的日志行为，让使用更加便捷
- 日志最终通过 `Logger.output` 函数完成输出，结合 `calldepth` 参数实现了日志定位

总而言之，`log` 标准库适用于简单、轻量的日志需求，但若需要日志分级、异步写入等更复杂的功能，则建议引入第三方日志库。