# gproc

进程管理以及进程间的通信是通过```gproc```包实现的，其中进程间通信采用的是本地socket通信机制。

使用方式：
```go
import "gitee.com/johng/gf/g/os/gproc"
```

方法列表：[godoc.org/github.com/johng-cn/gf/g/os/gproc](https://godoc.org/github.com/johng-cn/gf/g/os/gproc)


由于进程管理及通信的内容比较多，以下对常用的几种使用做简单介绍。

## 进程管理

### 主进程与子进程

由```gproc.Manager```对象创建的进程都默认带子进程标识，在子进程程序中可以通过```gproc.IsChild()```方法来判断自身是否为子进程。

```go
package main

import (
    "os"
    "time"
    "gitee.com/johng/gf/g/os/glog"
    "gitee.com/johng/gf/g/os/gproc"
)

func main () {
    if gproc.IsChild() {
        glog.Printfln("%d: Hi, I am child, waiting 3 seconds to die", gproc.Pid())
        time.Sleep(time.Second)
        glog.Printfln("%d: 1", gproc.Pid())
        time.Sleep(time.Second)
        glog.Printfln("%d: 2", gproc.Pid())
        time.Sleep(time.Second)
        glog.Printfln("%d: 3", gproc.Pid())
    } else {
        m := gproc.NewManager()
        p := m.NewProcess(os.Args[0], os.Args, os.Environ())
        p.Start()
        p.Wait()
        glog.Printfln("%d: child died", gproc.Pid())
    }
}
```
执行后，终端打印结果如下：
```shell
2018-05-18 14:35:41.360 28285: Hi, I am child, waiting 3 seconds to die
2018-05-18 14:35:42.361 28285: 1
2018-05-18 14:35:43.361 28285: 2
2018-05-18 14:35:44.361 28285: 3
2018-05-18 14:35:44.362 28278: child died
```

### 多进程管理

```gproc```除了能够创建子进程，管理子进程之外，也能管理非自身创建的其他进程。```gproc```可以同时管理多个进程，这里以单个进程为例来演示对进程的管理功能。

1. 我们使用```gedit```软件(Linux下常用的文本编辑器)随意打开一个文件，在进程当中我们看到该gedit的进程ID为```28536```
    ```shell
    $ ps aux | grep gedit
    john  28536  3.6  0.6 946208 56412 ?  Sl  14:39  0:00 gedit /home/john/Documents/text
    ```
1. 我们的程序如下：
    ```go
    package main

    import (
        "fmt"
        "gitee.com/johng/gf/g/os/gproc"
    )

    func main () {
        pid := 28536
        m   := gproc.NewManager()
        m.AddProcess(pid)
        m.KillAll()
        m.WaitAll()
        fmt.Printf("%d was killed\n", pid)
    }
    ```
	执行后，```gedit```被关闭，终端输出信息为：
    ```shell
    28536 was killed
    ```


## 进程通信

> 不要通过共享内存来通信，而应该通过通信来共享内存。

常见的进程通信方式有5种：```管道/信号量/共享内存/共享文件/Socket```。按照常见的并发架构的设计来讲，我们尽可能地少用```锁机制```，包括共享内存/共享文件其实都是需要依靠锁机制才能保证数据流的正确性，因为锁机制带来的维护复杂度往往会比其带来的好处更多。信号量常用在```*nix```系统中，跨平台性比较差。管道虽然实现起来比较简单，但是在稳定性上并没有Socket机制好。因此，gproc实现的进程通信采用的是Socket机制。需要注意的是，通信的两个进程都需要使用```gproc```包来实现发送&接收。


gproc的进程通信API非常简便，只需通过以下两个方法实现：
```go
func Send(pid int, data []byte) error
func Receive() *Msg
```
我们通过```Send```方法向指定的进程发送数据，在指定的进程中可以通过```Receive```方法获得数据。其中，```Receive```方法提供了类似消息队列的形式来收取其他进程传递的数据，当队列为空时，该方法将会```阻塞```等待。

我们来看一个进程间通信的基本使用示例：
```go
package main

import (
    "os"
    "fmt"
    "time"
    "gitee.com/johng/gf/g/os/gproc"
    "gitee.com/johng/gf/g/os/gtime"
)

func main () {
    fmt.Printf("%d: I am child? %v\n", gproc.Pid(), gproc.IsChild())
    if gproc.IsChild() {
        gtime.SetInterval(time.Second, func() bool {
            gproc.Send(gproc.PPid(), []byte(gtime.Datetime()))
            return true
        })
        select { }
    } else {
        m := gproc.NewManager()
        p := m.NewProcess(os.Args[0], os.Args, os.Environ())
        p.Start()
        for {
            msg := gproc.Receive()
            fmt.Printf("receive from %d, data: %s\n", msg.Pid, string(msg.Data))
        }
    }
}
```
该示例中，我们的主进程启动时创建了一个子进程，该子进程每隔1秒钟向主进程发送当前的时间，主进程收取到子进程发送的参数后输出到终端上。执行后，终端输出的内容如下：
```shell
29978: I am child? false
29984: I am child? true
receive from 29984, data: 2018-05-18 15:01:00
receive from 29984, data: 2018-05-18 15:01:01
receive from 29984, data: 2018-05-18 15:01:02
receive from 29984, data: 2018-05-18 15:01:03
receive from 29984, data: 2018-05-18 15:01:04
...
```
