+++
title = "Go 中 io 包的使用方法"
author = "hao.wu"
github_url = "https://sorrycloud.com"
head_img = ""
created_at = 2019-11-22T01:09:25
updated_at = 2019-11-22T01:15:57
description = "go io 包缓冲区并发时用到 io 包,记录下心得。"
tags = ["go"]
+++

## 前言

在 Go 中，输入和输出操作是使用接口实现的，这些接口将数据模拟成可读的或可写的字节流。
为此，Go 的 `io` 包提供了 `io.Reader` 和 `io.Writer` 接口，分别用于数据的输入和输出，如图：

![`io.Reader` 和 `io.Writer` 工作流程](https://github.com/504807890/blog-docs/blob/master/pic/go-io-package-1.png?raw=true)

Go 官方提供了一些 API，支持对内存结构，文件，网络连接等资源进行操作
本文重点介绍如何实现标准库中 `io.Reader` 和 `io.Writer` 两个接口，来完成流式传输数据。

## `io.Reader`

`io.Reader` 表示一个读取器，它将数据从某个资源读取到传输缓冲区。在缓冲区中，数据可以被流式传输和使用。
如图：

![`io.Reader` 工作流程](https://github.com/504807890/blog-docs/blob/master/pic/go-io-package-2.png?raw=true)

对于要用作读取器的类型，它必须实现 `io.Reader` 接口的唯一一个方法 `Read(p []byte)`。
换句话说，只要实现了 `Read(p []byte)` ，那它就是一个读取器。

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

`Read()` 方法有两个返回值，一个是读取到的字节数，一个是发生错误时的错误。
同时，如果资源内容已全部读取完毕，应该返回 `io.EOF` 错误。

### 使用 Reader

利用 `Reader` 可以很容易地进行流式数据传输。`Reader` 方法内部是被循环调用的，每次迭代，它会从数据源读取一块数据放入缓冲区 `p` （即 Read 的参数 p）中，直到返回 `io.EOF` 错误时停止。

下面是一个简单的例子，通过 `string.NewReader(string)` 创建一个字符串读取器，然后流式地按字节读取：

```go
func main() {
    reader := strings.NewReader("Clear is better than clever")
    p := make([]byte, 4)

    for {
        n, err := reader.Read(p)
        if err != nil{
            if err == io.EOF {
                fmt.Println("EOF:", n)
                break
            }
            fmt.Println(err)
            os.Exit(1)
        }
        fmt.Println(n, string(p[:n]))
    }
}
```

```bash
输出打印的内容：
4 Clea
4 r is
4  bet
4 ter 
4 than
4  cle
3 ver
EOF: 0 
```

可以看到，最后一次返回的 n 值有可能小于缓冲区大小。

### 自己实现一个 Reader

上一节是使用标准库中的 `io.Reader` 读取器实现的。
现在，让我们看看如何自己实现一个。它的功能是从流中过滤掉非字母字符。

```go
type alphaReader struct {
    // 资源
    src string
    // 当前读取到的位置 
    cur int
}

// 创建一个实例
func newAlphaReader(src string) *alphaReader {
    return &alphaReader{src: src}
}

// 过滤函数
func alpha(r byte) byte {
    if (r >= 'A' && r <= 'Z') || (r >= 'a' && r <= 'z') {
        return r
    }
    return 0
}

// Read 方法
func (a *alphaReader) Read(p []byte) (int, error) {
    // 当前位置 >= 字符串长度 说明已经读取到结尾 返回 EOF
    if a.cur >= len(a.src) {
        return 0, io.EOF
    }

    // x 是剩余未读取的长度
    x := len(a.src) - a.cur
    n, bound := 0, 0
    if x >= len(p) {
        // 剩余长度超过缓冲区大小，说明本次可完全填满缓冲区
        bound = len(p)
    } else if x < len(p) {
        // 剩余长度小于缓冲区大小，使用剩余长度输出，缓冲区不补满
        bound = x
    }

    buf := make([]byte, bound)
    for n < bound {
        // 每次读取一个字节，执行过滤函数
        if char := alpha(a.src[a.cur]); char != 0 {
            buf[n] = char
        }
        n++
        a.cur++
    }
    // 将处理后得到的 buf 内容复制到 p 中
    copy(p, buf)
    return n, nil
}

func main() {
    reader := newAlphaReader("Hello! It's 9am, where is the sun?")
    p := make([]byte, 4)
    for {
        n, err := reader.Read(p)
        if err == io.EOF {
            break
        }
        fmt.Print(string(p[:n]))
    }
    fmt.Println()
}
```

```bash
输出打印的内容：
HelloItsamwhereisthesun
```

### 组合多个Reader
>目的是重用和屏蔽下层实现的复杂度

标准库已经实现了许多 `Reader`。
使用一个 `Reader` 作为另一个 `Reader` 的实现是一种常见的用法。
这样做可以让一个 `Reader` 重用另一个 `Reader` 的逻辑，下面展示通过更新 `alphaReader` 以接受 `io.Reader` 作为其来源。

```go
type alphaReader struct {
    // alphaReader 里组合了标准库的 io.Reader
    reader io.Reader
}

func newAlphaReader(reader io.Reader) *alphaReader {
    return &alphaReader{reader: reader}
}

func alpha(r byte) byte {
    if (r >= 'A' && r <= 'Z') || (r >= 'a' && r <= 'z') {
        return r
    }
    return 0
}

func (a *alphaReader) Read(p []byte) (int, error) {
    // 这行代码调用的就是 io.Reader
    n, err := a.reader.Read(p)
    if err != nil {
        return n, err
    }
    buf := make([]byte, n)
    for i := 0; i < n; i++ {
        if char := alpha(p[i]); char != 0 {
            buf[i] = char
        }
    }

    copy(p, buf)
    return n, nil
}

func main() {
    //  使用实现了标准库 io.Reader 接口的 strings.Reader 作为实现
    reader := newAlphaReader(strings.NewReader("Hello! It's 9am, where is the sun?"))
    p := make([]byte, 4)
    for {
        n, err := reader.Read(p)
        if err == io.EOF {
            break
        }
        fmt.Print(string(p[:n]))
    }
    fmt.Println()
}
```

这样做的另一个优点是 `alphaReader` 能够从任何 `Reader` 实现中读取。
例如，以下代码展示了 `alphaReader` 如何与 `os.File` 结合以过滤掉文件中的非字母字符：

```go
func main() {
    // file 也实现了 io.Reader
    file, err := os.Open("./alpha_reader3.go")
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    defer file.Close()
    
    // 任何实现了 io.Reader 的类型都可以传入 newAlphaReader
    // 至于具体如何读取文件，那是标准库已经实现了的，我们不用再做一遍，达到了重用的目的
    reader := newAlphaReader(file)
    p := make([]byte, 4)
    for {
        n, err := reader.Read(p)
        if err == io.EOF {
            break
        }
        fmt.Print(string(p[:n]))
    }
    fmt.Println()
}
```


## `io.Writer`

`io.Writer` 表示一个编写器，它从缓冲区读取数据，并将数据写入目标资源。

![`io.Writer` 工作流程](https://github.com/504807890/blog-docs/blob/master/pic/go-io-package-3.png?raw=true)

对于要用作编写器的类型，必须实现 `io.Writer` 接口的唯一一个方法 `Write(p []byte)`
同样，只要实现了 `Write(p []byte)` ，那它就是一个编写器。

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

`Write()` 方法有两个返回值，一个是写入到目标资源的字节数，一个是发生错误时的错误。

### 使用 Writer

标准库提供了许多已经实现了 `io.Writer` 的类型。
下面是一个简单的例子，它使用 `bytes.Buffer` 类型作为 `io.Writer` 将数据写入内存缓冲区。

```go
func main() {
    proverbs := []string{
        "Channels orchestrate mutexes serialize",
        "Cgo is not Go",
        "Errors are values",
        "Don't panic",
    }
    var writer bytes.Buffer

    for _, p := range proverbs {
        n, err := writer.Write([]byte(p))
        if err != nil {
            fmt.Println(err)
            os.Exit(1)
        }
        if n != len(p) {
            fmt.Println("failed to write data")
            os.Exit(1)
        }
    }

    fmt.Println(writer.String())
}
```

```bash
输出打印的内容：
Channels orchestrate mutexes serializeCgo is not GoErrors are valuesDon't panic
```
### 自己实现一个 Writer
下面我们来实现一个名为 `chanWriter` 的自定义 `io.Writer` ，它将其内容作为字节序列写入 `channel` 。

```go
type chanWriter struct {
    // ch 实际上就是目标资源
    ch chan byte
}

func newChanWriter() *chanWriter {
    return &chanWriter{make(chan byte, 1024)}
}

func (w *chanWriter) Chan() <-chan byte {
    return w.ch
}

func (w *chanWriter) Write(p []byte) (int, error) {
    n := 0
    // 遍历输入数据，按字节写入目标资源
    for _, b := range p {
        w.ch <- b
        n++
    }
    return n, nil
}

func (w *chanWriter) Close() error {
    close(w.ch)
    return nil
}

func main() {
    writer := newChanWriter()
    go func() {
        defer writer.Close()
        writer.Write([]byte("Stream "))
        writer.Write([]byte("me!"))
    }()
    for c := range writer.Chan() {
        fmt.Printf("%c", c)
    }
    fmt.Println()
}
```

要使用这个 `Writer`，只需在函数 `main()` 中调用 `writer.Write()`（在单独的`goroutine`中）。
因为 `chanWriter` 还实现了接口 `io.Closer` ，所以调用方法 `writer.Close()` 来正确地关闭`channel`，以避免发生泄漏和死锁。

## `io` 包里其他有用的类型和方法
>如前所述，Go标准库附带了许多有用的功能和类型，让我们可以轻松使用流式io。

### `os.File`
类型 `os.File` 表示本地系统上的文件。它实现了 `io.Reader` 和 `io.Writer` ，因此可以在任何 `io` 上下文中使用。
例如，下面的例子展示如何将连续的字符串切片直接写入文件：

```go
func main() {
    proverbs := []string{
        "Channels orchestrate mutexes serialize\n",
        "Cgo is not Go\n",
        "Errors are values\n",
        "Don't panic\n",
    }
    file, err := os.Create("./proverbs.txt")
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    defer file.Close()

    for _, p := range proverbs {
        // file 类型实现了 io.Writer
        n, err := file.Write([]byte(p))
        if err != nil {
            fmt.Println(err)
            os.Exit(1)
        }
        if n != len(p) {
            fmt.Println("failed to write data")
            os.Exit(1)
        }
    }
    fmt.Println("file write done")
}
```

同时，`io.File` 也可以用作读取器来从本地文件系统读取文件的内容。
例如，下面的例子展示了如何读取文件并打印其内容：

```go
func main() {
    file, err := os.Open("./proverbs.txt")
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    defer file.Close()

    p := make([]byte, 4)
    for {
        n, err := file.Read(p)
        if err == io.EOF {
            break
        }
        fmt.Print(string(p[:n]))
    }
}
```

### `标准输入、输出和错误`
`os` 包有三个可用变量 `os.Stdout` ，`os.Stdin` 和 `os.Stderr` ，它们的类型为 `*os.File`，分别代表 系统标准输入，系统标准输出 和 系统标准错误 的文件句柄。
例如，下面的代码直接打印到标准输出：

```go
func main() {
    proverbs := []string{
        "Channels orchestrate mutexes serialize\n",
        "Cgo is not Go\n",
        "Errors are values\n",
        "Don't panic\n",
    }

    for _, p := range proverbs {
        // 因为 os.Stdout 也实现了 io.Writer
        n, err := os.Stdout.Write([]byte(p))
        if err != nil {
            fmt.Println(err)
            os.Exit(1)
        }
        if n != len(p) {
            fmt.Println("failed to write data")
            os.Exit(1)
        }
    }
}
```

### `io.Copy()`
`io.Copy()` 可以轻松地将数据从一个 `Reader` 拷贝到另一个 `Writer`。
它抽象出 `for` 循环模式（我们上面已经实现了）并正确处理 `io.EOF` 和 字节计数。
下面是我们之前实现的简化版本：

```go
func main() {
    proverbs := new(bytes.Buffer)
    proverbs.WriteString("Channels orchestrate mutexes serialize\n")
    proverbs.WriteString("Cgo is not Go\n")
    proverbs.WriteString("Errors are values\n")
    proverbs.WriteString("Don't panic\n")

    file, err := os.Create("./proverbs.txt")
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    defer file.Close()

    // io.Copy 完成了从 proverbs 读取数据并写入 file 的流程
    if _, err := io.Copy(file, proverbs); err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    fmt.Println("file created")
}
```

那么，我们也可以使用 `io.Copy()` 函数重写从文件读取并打印到标准输出的先前程序，如下所示：

```go
func main() {
    file, err := os.Open("./proverbs.txt")
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    defer file.Close()

    if _, err := io.Copy(os.Stdout, file); err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
}
```

### `io.WriteString()`
此函数让我们方便地将字符串类型写入一个 Writer：

```go
func main() {
    file, err := os.Create("./magic_msg.txt")
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    defer file.Close()
    if _, err := io.WriteString(file, "Go is fun!"); err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
}
```

### `使用管道的 Writer 和 Reader`

类型 `io.PipeWriter` 和 `io.PipeReader` 在内存管道中模拟 `io` 操作。
数据被写入管道的一端，并使用单独的 `goroutine` 在管道的另一端读取。
下面使用 `io.Pipe()` 创建管道的 `reader` 和 `writer`，然后将数据从 `proverbs` 缓冲区复制到`io.Stdout` ：

```go
func main() {
    proverbs := new(bytes.Buffer)
    proverbs.WriteString("Channels orchestrate mutexes serialize\n")
    proverbs.WriteString("Cgo is not Go\n")
    proverbs.WriteString("Errors are values\n")
    proverbs.WriteString("Don't panic\n")

    piper, pipew := io.Pipe()

    // 将 proverbs 写入 pipew 这一端
    go func() {
        defer pipew.Close()
        io.Copy(pipew, proverbs)
    }()

    // 从另一端 piper 中读取数据并拷贝到标准输出
    io.Copy(os.Stdout, piper)
    piper.Close()
}
```

### `缓冲区 io`
标准库中 `bufio` 包支持 缓冲区 `io` 操作，可以轻松处理文本内容。
例如，以下程序逐行读取文件的内容，并以值 `'\n'` 分隔：

```go
func main() {
    file, err := os.Open("./planets.txt")
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    defer file.Close()
    reader := bufio.NewReader(file)

    for {
        line, err := reader.ReadString('\n')
        if err != nil {
            if err == io.EOF {
                break
            } else {
                fmt.Println(err)
                os.Exit(1)
            }
        }
        fmt.Print(line)
    }

}
```

### `ioutil`
`io` 包下面的一个子包 `utilio` 封装了一些非常方便的功能
例如，下面使用函数 `ReadFile` 将文件内容加载到 `[]byte` 中。

```go
package main

import (
  "io/ioutil"
   ...
)

func main() {
    bytes, err := ioutil.ReadFile("./planets.txt")
    if err != nil {
        fmt.Println(err)
        os.Exit(1)
    }
    fmt.Printf("%s", bytes)
}
```

## 总结
本文介绍了如何使用 `io.Reader` 和 `io.Writer` 接口在程序中实现流式IO。
阅读本文后，您应该能够了解如何使用 `io` 包来实现 流式传输IO数据的程序。
其中有一些例子，展示了如何创建自己的类型，并实现 `io.Reader` 和 `io.Writer` 。