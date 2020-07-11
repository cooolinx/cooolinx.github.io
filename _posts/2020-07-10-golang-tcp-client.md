---
layout: post
title: "Go笔记 · 一条TCP连接讲透九大知识点"
subtitle: "Go网络编程与dial,defer,Read,EOF,for,go,chan,select,context"
background: '/assets/bg/puket.jpg'
date:   2020-07-10 19:07:00 +0800
categories: scinet
---

项目做了半年，现在要开发iOS版本。由于iOS的Network Extension对内存有15M限制，现成的实现方案都太耗内存，需要自己从头开发一个精简版。所以最近两周在学Go语言。

话不多说，正式开始，分为几步逐步来完善一个TCP客户端：

0. 用nc做简单服务器
1. 用Dial建立TCP客户端
2. 用defer在函数退出时执行关闭操作
3. 用Read接收服务端消息
4. 用io.EOF来识别正常读取结束
5. 用for持续读取服务端消息
6. 用go启动协程实现交互
7. 用chan的协程间通信实现控制流程
8. 用select的多通道监听实现简单超时
9. 用context实现标准超时

# 0 - 用nc作简单服务器

由于后面要做TCP客户端，需要一个方便可调试的服务端，也可以自己用Go写一个，但不是本篇重点，因此选用[有网络工具瑞士军刀之称的Netcat](https://zh.wikipedia.org/wiki/Netcat)。

启动服务器，监听1234端口

```sh
nc -l 1234
```

客户端测试连接

```sh
nc -v 127.0.0.1 1234
```

尝试在两端随意输入文字，会传输到另外一侧，按`Ctrl+C`中断连接

# 1 - 用Dial建立TCP客户端

编辑`tcpclient.go`文件：

```golang
package main

import (
    "net"
)

func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:1234")  // 以TCP协议连通127.0.0.1的1234端口
    if err != nil {
        conn.Close()
        panic(err)
    }
    c := []byte("hello\n")  // 将字符串转换为byte数组(slice)
    conn.Write(c)  // 将byte写入连接/发送至服务端
    conn.Close()
}
```

> 可以参考 [net.Dial](https://golang.org/pkg/net/#Dial) ，返回的`conn`实现了 [io.Reader](https://golang.org/pkg/io/#Reader) 和 [io.Writer](https://golang.org/pkg/io/#Writer) 接口

启动nc服务器

```sh
nc -l 1234
```

运行并向nc服务器发送`hello`

```sh
go run tcpclient.go
```

查看nc服务器那边，会显示接收到`hello`

# 2 - 用defer在函数退出时执行关闭操作

以上代码逻辑没问题，但有个不优雅的地方，就是我们在`if err != nil`的分支和最后两次调用了`conn.Close()`，如果一个正常函数，有七八处判断error的地方，难道我们也要写这么多次？

答案是`defer`，`defer`是用于**安排一段逻辑在离开函数时执行**的，修改后的代码是：

```golang
package main

import (
    "net"
)

func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:1234")
    if err != nil {
        panic(err)
    }
    defer conn.Close()  // 这段逻辑一定会在以任何方式离开函数时执行
    c := []byte("hello\n")
    conn.Write(c)
}
```


# 3 - 用Read接收服务端消息

上面一步我们只实现了发送消息，那么接收服务端消息要怎么实现呢？

修改一下上一步的代码：

```golang
package main

import (
    "net"
    "fmt"
)

func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:1234")
    if err != nil {
        panic(err)
    }
    defer conn.Close()
    c := []byte("hello\n")
    conn.Write(c)

    buf := make([]byte, 1024)  // 创建长度为1k的缓存区
    n, err := conn.Read(buf)  // 将接收到的服务端内容放入缓存区，并返回此次读取的长度
    if err != nil {
        panic(err)
    }
    fmt.Printf("received: %v", string(buf[:n]))  // 打印出来
}
```

> [make](https://golang.org/pkg/builtin/#make) 是Go的内置函数；
> 上节提到`conn`实现了 [io.Reader](https://golang.org/pkg/io/#Reader) 接口，因此它有`Read`方法；
> 与`Read`相关的函数还有很多，这里仅介绍这个是因为他是其他一切高级`Read`函数的基础。

启动后在服务端输入`how are you`，会在客户端收到`received: how are you`

**注意 `conn.Read()` 函数会阻塞住程序，直到它真正读取到数据，这很关键**。

# 4 - 用io.EOF来识别正常读取结束

再次启动服务端和客户端，然后尝试在服务端按`Ctrl+C`中断，观察客户端，抛出了异常。但在实际生产环境中，服务端因为各种原因关闭连接是很正常的，那我们要如何来判断呢？答案是`io.EOF`

```golang
package main

import (
    "net"
    "fmt"
    "io"  // 注意引入io包
)

func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:1234")
    if err != nil {
        panic(err)
    }
    defer conn.Close()
    c := []byte("hello\n")
    conn.Write(c)

    buf := make([]byte, 1024)
    n, err := conn.Read(buf)
    if err != nil {
        if err == io.EOF {  // 若异常是io.EOF，则正常退出函数不做panic
            return
        }
        panic(err)
    }
    fmt.Printf("received: %v", string(buf[:n]))
}
```

以后所有读取的地方，都需要注意这点。

# 5 - 用for持续读取服务端消息

上面仅能读取一次服务端消息，有没有办法让它能持续接收呢？答案是用`for`循环

```golang
package main

import (
    "net"
    "fmt"
    "io"
)

func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:1234")
    if err != nil {
        panic(err)
    }
    defer conn.Close()
    c := []byte("hello\n")
    conn.Write(c)

    buf := make([]byte, 1024)

    for {  // 无参for指令代表无限循环
        n, err := conn.Read(buf)
        if err != nil {
            if err == io.EOF {
                return
            }
            panic(err)
        }
        fmt.Printf("received: %v", string(buf[:n]))
    }
}
```

再次运行，在服务端输入`nice to meet you`和`whats up`，观察客户端的表现。按`Ctrl+C`结束客户端程序。


# 6 - 用go启动协程实现交互

以上代码已经完成了客户端持续接收服务端消息的能力，但客户端却不能对自己有任何操作，我们需要实现客户端同时可以输入。

先看一段错误代码：

```golang
package main

import (
    "net"
    "fmt"
    "io"
)

func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:1234")
    if err != nil {
        panic(err)
    }
    defer conn.Close()
    c := []byte("hello\n")
    conn.Write(c)

    buf := make([]byte, 1024)

    for {
        n, err := conn.Read(buf)
        if err != nil {
            if err == io.EOF {
                return
            }
            panic(err)
        }
        fmt.Printf("received: %v", string(buf[:n]))

        // 客户端可以输入消息并发送到服务端
        var inp string
        fmt.Scanln(&inp)
        conn.Write([]byte(inp + "\n"))
    }
}
```

启动之后我们会发现，仅有在服务端发送消息给客户端后，客户端才能够开始输入消息，这是因为我们之前说的**`conn.Read()`会阻塞整个代码**。那我们该怎么办呢？答案是`协程（goroutine）`，看下面修改的代码：

```golang
package main

import (
    "net"
    "fmt"
    "io"
)

func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:1234")
    if err != nil {
        panic(err)
    }
    defer conn.Close()
    c := []byte("hello\n")
    conn.Write(c)

    // 将读取部分放入到子协程中，不阻塞主协程运行
    go func() {
        buf := make([]byte, 1024)
        for {
            n, err := conn.Read(buf)
            if err != nil {
                if err == io.EOF {
                    return
                }
                panic(err)
            }
            fmt.Printf("received: %v", string(buf[:n]))
        }
    }()

    // 客户端可以输入消息并发送到服务端
    for {
        var inp string
        fmt.Scanln(&inp)
        conn.Write([]byte(inp + "\n"))
    }
}
```

> `go`指令启动子协程，其中任何操作都不会阻塞当前协程，因此主协程会直接执行到输入指令处。

# 7 - 用chan的协程间通信实现控制流程

上面的程序已经运转良好了，如果我们现在有一个需求，是当接收到服务端发来的`bye`消息时，客户端退出，该如何实现？可以先思考一下再看代码

```golang
package main

import (
    "net"
    "fmt"
    "io"
)

func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:1234")
    if err != nil {
        panic(err)
    }
    defer conn.Close()
    c := []byte("hello\n")
    conn.Write(c)

    quit := make(chan string, 1)  // 1. 创建长度为1的通道（chan）

    // 读取协程
    go func() {
        buf := make([]byte, 1024)
        for {
            n, err := conn.Read(buf)
            if err != nil {
                if err == io.EOF {
                    return
                }
                panic(err)
            }
            r := string(buf[:n])
            fmt.Printf("received: %v", r)

            if r == "bye\n" {  // 若接收到服务端发送过来的bye
                quit <-"server quit"  // 3. 向通道内写入内容
                return
            }
        }
    }()

    // 用户输入协程
    go func() {  // 将用户输入也变为子协程
        for {
            var inp string
            fmt.Scanln(&inp)
            conn.Write([]byte(inp + "\n"))
        }
    }()

    r := <-quit  // 2. 尝试从通道中读取内容，若通道为空，则阻塞在此
    fmt.Printf("command: %v", r)
}
```

以上代码解读：

1. 将阻塞操作`用户输入`也变为与网络读取一样的子协程
2. 创建通道`quit`
3. 在主协程最后，尝试从`quit`通道中取出内容，若通道为空，则阻塞在此
4. 在读取协程中，判断若服务端发送过来`bye`，则向`quit`通道中写入一个值（这个值可以是任意值）
5. 一旦`quit`中被写入一个值，`r := <-quit`就会成功取出，不再阻塞，退出程序

所以，关键点是**通道的取出是阻塞的**

练习题：尝试将以上代码改为客户端输入`bye`也可以退出


# 8 - 用select的多通道监听实现简单超时

如果我们希望添加超时怎么做？

思路其实很简单，要超时退出，就是要在刚刚以上的`bye`命令通知机制上，再加上时间通知。

```golang
package main

import (
    "net"
    "fmt"
    "io"
    "time"  // 引入time包
)

func main() {
    conn, err := net.Dial("tcp", "127.0.0.1:1234")
    if err != nil {
        panic(err)
    }
    defer conn.Close()
    c := []byte("hello\n")
    conn.Write(c)

    quit := make(chan string, 1)

    // 读取协程
    go func() {
        buf := make([]byte, 1024)
        for {
            n, err := conn.Read(buf)
            if err != nil {
                if err == io.EOF {
                    return
                }
                panic(err)
            }
            r := string(buf[:n])
            fmt.Printf("received: %v", r)

            if r == "bye\n" {
                quit <-"server quit"
                return
            }
        }
    }()

    // 用户输入协程
    go func() {
        for {
            var inp string
            fmt.Scanln(&inp)
            conn.Write([]byte(inp + "\n"))
        }
    }()

    // 将简单的读取quit通道，改为select多路通道监听
    select {
    case r := <-quit:
        fmt.Printf("command: %v", r)
    case <-time.After(5 * time.Second):  // 新增一个通道条件是5s之后通道中有值
        fmt.Printf("timeout")
    }
}
```

> [time.After](https://golang.org/pkg/time/#After) 返回一个通道，在指定时间到达时，将向通道内写入当前时间。而 [select](https://golang.org/ref/spec#Select_statements) 关键字则用于多通道监听。

运行后，任意互动，5s后程序就会退出。


# 9 - 用context实现标准超时

我们再看另一种更标准化的超时：

```golang
package main

import (
    "net"
    "fmt"
    "io"
    "time"
    "context"
)

func main() {
    // 创建context用于协程间传递
    ctx, cancel := context.WithTimeout(context.Background(), 5 * time.Second)
    defer cancel()  // 函数退出时需关闭context

    conn, err := net.Dial("tcp", "127.0.0.1:1234")
    if err != nil {
        panic(err)
    }
    defer conn.Close()
    c := []byte("hello\n")
    conn.Write(c)

    quit := make(chan string, 1)

    // 读取协程
    go func() {
        buf := make([]byte, 1024)
        for {
            n, err := conn.Read(buf)
            if err != nil {
                if err == io.EOF {
                    return
                }
                panic(err)
            }
            r := string(buf[:n])
            fmt.Printf("received: %v", r)

            if r == "bye\n" {
                quit <-"server quit"
                return
            }
        }
    }()

    // 用户输入协程
    go func() {
        for {
            var inp string
            fmt.Scanln(&inp)
            conn.Write([]byte(inp + "\n"))
        }
    }()

    select {
    case r := <-quit:
        fmt.Printf("command: %v", r)
    case <-ctx.Done():  // 改为监听ctx.Done()通道
        fmt.Printf("timeout")
    }
}
```

效果和刚刚一样，那为什么要用这种而不是`time.After()`？因为context更加强大，除此之外还可以做很多事情，看下面我们优化的代码：

```golang
package main

import (
    "net"
    "fmt"
    "io"
    "time"
    "context"
)

func main() {
    // 创建context用于协程间传递
    ctx, cancel := context.WithTimeout(context.Background(), 5 * time.Second)
    defer cancel()  // 函数退出时需关闭context

    var dialer net.Dialer  // 创建dialer
    conn, err := dialer.DialContext(ctx, "tcp", "127.0.0.1:1234")  // 在连接时就将context传入，可以确保连接时长也受context限制
    if err != nil {
        panic(err)
    }
    defer conn.Close()
    c := []byte("hello\n")
    conn.Write(c)

    // 读取协程
    go func() {
        buf := make([]byte, 1024)
        for {
            n, err := conn.Read(buf)
            if err != nil {
                if err == io.EOF || ctx.Err() != nil {  // 由context进行的主动退出，不要panic
                    return
                }
                panic(err)
            }
            r := string(buf[:n])
            fmt.Printf("received: %v", r)

            if r == "bye\n" {
                cancel()  // 直接调用cancel函数完成退出消息传递
                return
            }
        }
    }()

    // 用户输入协程
    go func() {
        for {
            var inp string
            fmt.Scanln(&inp)
            conn.Write([]byte(inp + "\n"))
        }
    }()

    select {
    case <-ctx.Done():  // 仅监听ctx.Done()通道
    }

    // 根据context的异常来判断退出原因
    err = ctx.Err()
    if err == context.Canceled {
        fmt.Printf("user canceled")
    } else if err == context.DeadlineExceeded {
        fmt.Printf("timeout")        
    }
}
```

`context`还实现了连接超时，并且现在超时之后报的`panic`错误也处理了，退出的原因也做了统一处理，一切都归到了context上，很和谐。

练习题：把以上代码改为5s没有接收到也没有发送任何消息才算超时

# 回顾

好了，以上就是本次的内容。涉及了以下知识：

1. net.dial
2. defer
3. io.Reader.Read
4. io.EOF
5. for
6. go
7. chan
8. select
9. context

建议平时再多看看其他资料，多写写。
