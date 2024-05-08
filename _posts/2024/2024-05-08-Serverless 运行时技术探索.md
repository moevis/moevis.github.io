---
layout: post
title:  "Serverless 运行时技术探索"
date:   2024-05-08 00:00:00
categories: cheatsheet
author: moevis
---

## 前言

Serverless 即无服务架构，是一种比较流行的基础。

这篇文章我们不会深入 Serverless 定义，我假定你已经懂得 Serverless 服务的一些特点，并懂得使用 C 或者 Go 进行基础的系统编程。我们将从底层实现出发，尝试探索构建一个 Serverless 运行时所需的技术。

> 在实践中，Serverless 运行时一般由容器构成，但是否真的需要使用容器，我们还是要根据实际的业务需求来确定。

Serverless 中的业务逻辑应该是无状态的，函数运行由事件驱动（如 http 请求，mq 消息），并可以根据流量进行快速横向扩容/缩容，以节省成本。从实践来说，架构方会事先准备基础运行环境/容器，在函数冷启动时，载入用户程序/代码执行并返回结果。

所以现在开始，思考一下，我们手中有哪些可以用的技术实现，运行自定义的代码/程序。

## 运行自定义代码

### 子进程

运行第三方代码/程序的方法，最简单直观的方式是调用子进程。

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int main(int argc, char *argv[]) {
    pid_t pid;

    if (argc < 2) {
        fprintf(stderr, "Usage: %s <command> [arguments...]\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    pid = fork(); // 创建一个子进程

    if (pid == -1) {
        // fork失败
        perror("fork failed");
        exit(EXIT_FAILURE);
    } else if (pid == 0) {
        // 在子进程中
        execvp(argv[1], &argv[1]); // 执行命令行参数指定的程序

        // 如果execvp返回，那说明出错了
        perror("execvp failed");
        exit(EXIT_FAILURE);
    } else {
        // 在父进程中
        int status;
        waitpid(pid, &status, 0); // 等待子进程结束
        if (WIFEXITED(status)) {
            printf("Child exited with code %d\n", WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            printf("Child terminated by signal %d\n", WTERMSIG(status));
        }
    }

    return 0;
}
```

```shell
gcc -o simple_run main.c
```

运行子程序也很简单

```
./simple_run ls -l
```

在这个 simple_run 里面，接受命令行参数指定第三方程序的运行命令，并通过父子进程方式进行运行时隔离。

这个方式极为简单直接，但是也是最为危险的，因为这里对第三方程序的限制非常少，需要进一步优化。

### 脚本执行模块

对于脚本化的用户程序，可以直接加载用户的代码。

最简单的例子如下。

```javascript
const clientCodePath = process.args[0];
const clientCode = require(clientCodePath); // 动态引入用户代码
const output = clientCode(); // 执行用户代码
console.log(output);
```

借由宿主的运行时环境来运行用户的脚本代码，本身的实现复杂度也非常低。

这个方案的问题有：

- 代码运行时版本由宿主运行时决定，无法自由切换
- 宿主运行时环境可能会被用户修改（如全局变量）
- 用户的代码崩溃影响宿主运行时

### 总结

刚刚列举出来了对于第三方程序的运行的最简单直接的两种方案。但实际上这两种方案的缺点显而易见：

- 没有处理用户代码的运行时版本与环境依赖
- 没有隔离限制运行时环境

实际上我们要解决的就是环境依赖打包以及用户运行时隔离，让我们接下来继续思考一下。

## 环境依赖打包

首先，在业界进行环境依赖打包一般基于容器技术，不过这个方案由于过于完善，无法让我们窥见技术底层，所以在这里，我们先介绍一些『土』方法来说明环境依赖打包的方案。

### 动态编译依赖

在 Linux 系统中，动态编译的可执行文件往往需要在系统路径中寻找并加载依赖库，如果依赖在运行时不存在，就会报错停止运行。

在执行 Linux 可执行文件运行时，系统会读取环境变量 `LD_LIBRARY_PATH` 的值，若存在该环境变量，则会从该环境变量指定的目录中查找可执行文件。

查看一个可执行文件的动态依赖库可以用命令 ldd，比如：

```bash
ldd /bin/ls
	linux-vdso.so.1 (0x00007ffda414f000)
	libcap.so.2 => /usr/lib/libcap.so.2 (0x00007f11a20af000)
	libc.so.6 => /usr/lib/libc.so.6 (0x00007f11a1ecd000)
	/lib64/ld-linux-x86-64.so.2 => /usr/lib64/ld-linux-x86-64.so.2 (0x00007f11a210a000)
```

能观察到 ls 命令的依赖有 libcap.so.2，libc.so.6, /usr/lib64/ld-linux-x86-64.so.2（麻烦忽略 `linux-vdso.so.1` 这个库，它实际上不存在）。

在进行依赖打包的时候，可以将依赖的动态依赖都打包起来（看起来像是手动进行静态依赖）。在运行程序的时候，设置环境变量 `LD_LIBRARY_PATH` 到依赖目录，即可指定加载对应的库文件。

### 脚本语言依赖

脚本语言往往有自己一套依赖管理方式，比如 Node.js 通过 package.json 记录依赖，将 npm 包进行本地缓存是一种较好的策略。

### 总结

可执行文件的动态依赖库可以通过 ldd 的依赖分析来进行打包，然后将依赖加入代码包中。

> 注意，ldd 命令可能会触发一些可执行文件运行，这里同样要注意安全性。

在实践中，打包的方式会要注意一些问题，使用压缩打包可以较为有效地压缩文件大小。但是注意如果用 zip 会导致丢失 Linux 可执行文件丢失可执行标记。

一些在运行时加载依赖库的方式，无法通过静态分析来得到依赖目录，比如通过 dlopen 方式加载的动态库。

假设现有一个动态库（例如 libexample.so），并且这个库中有一个函数 int add(int a, int b)。

以下是使用 dlopen 加载这个库并调用 add 函数的示例代码：

```C
#include <stdio.h>
#include <dlfcn.h>

int main() {
    void *handle;
    int (*add)(int, int); // 函数指针
    int result;

    // 使用 RTLD_LAZY 标志来延迟解析库中的符号，直到首次使用时才进行解析
    handle = dlopen("./libexample.so", RTLD_LAZY);
    if (!handle) {
        // 如果无法加载库，则打印错误并退出
        fprintf(stderr, "%s\n", dlerror());
        return 1;
    }

    // 清除现有的错误，确保后续调用 dlerror() 时得到的是与 dlsym() 相关的错误
    dlerror();

    // 获取 add 函数的地址
    add = (int (*)(int, int))dlsym(handle, "add");
    if (!add) {
        // 如果无法找到符号，则打印错误并退出
        fprintf(stderr, "%s\n", dlerror());
        return 1;
    }

    // 调用 add 函数并打印结果
    result = add(2, 3);
    printf("2 + 3 = %d\n", result);

    // 关闭动态库
    dlclose(handle);

    return 0;
}
```

该程序在编译后，无法通过 ldd 进行分析出依赖 `libexample.so`，所以这种动态加载技术很可能会导致依赖分析出错。

## 依赖加载优化

### 优化镜像大小

理论上，下载可执行文件并运行起来的耗时并不高，做好缓存的话，能做到秒级的启动。

但是如果是以镜像进行分发，你选择的镜像过大可能会对冷启动时间造成影响，比如千兆下行网全速下载一个 1G 大小的镜像所需的时间是 10s 左右。

实际上，我们可以针对业务内容去优化代码，而不是打入整个镜像。

如果希望内部有基础 Linux 命令可以使用，则可以打包 gnu core util 进入镜像的 /bin 目录。

实际上如果只是纯业务代码，镜像可以做到非常非常小，你甚至可以根据业务逻辑决定哪些资源需要加入：

- 如果要进行域名访问，镜像中需要有 resolve.cnf 文件
- 如果访问的是 https 资源，需要添加 cetificates 文件
- 如果你的代码是动态编译的，那么 libc.so 大概率是需要加入的

### 优化加载方案

镜像的目录大概率是不可更改的，可以打包成紧凑的只读镜像（erofs），很多文件调用在只读镜像上可以直接返回，相比可写的文件系统，erofs 对于随机读写，占用空间都有极大优化。

比如下面的命令创建了一个最简的只读 busybox 系统并挂载为 `/mnt/busybox_erofs`

```shell
# 创建一个只读镜像
dd if=/dev/zero of=busybox_erofs_image.img bs=1M count=100
# 将 busybox 系统写入镜像
dd if=busybox_fs.img of=busybox_erofs_image.img conv=notrunc
# 将镜像转换为 EROFS 格式
mkfs.erofs -i busybox_fs.img -o busybox_erofs.img

# 创建一个挂载点
mkdir /mnt/busybox_erofs

# 挂载镜像
sudo mount -t erofs -o loop busybox_erofs.img /mnt/busybox_erofs
```

另外，还有一种优化 serverless 服务加载时间的做法是动态加载，比如阿里巴巴提供的 nybus 镜像服务，会对容器镜像进行 chunk 化，以 10MB 作为一个单位分割，容器可以在镜像还未 ready 时即可启动，当业务代码需要容器中的某个文件时，会去下载对应的 chunk。chunk 可以跨不同的镜像进行复用，而不是以往的 layer 复用，这样的方案对于大镜像有很好的加速作用。

## 运行时隔离

使用独立线程跑用户代码天然就具备故障隔离的特性，脚本语言可能提供一个隔离的上下文运行环境，尽管大多数不那么完美。

我们希望做的运行时隔离，可以转化为对程序 IO 边界进行控制（内存，文件，网络等）。接下来我们会提出一些方案，以实现下面的一项或者多项目标。

- 限制内存
- 限制文件系统
- 限制网络
- 限制系统调用

### VM.js

Node.js 中的 vm 模块可以提供一定程度的代码隔离。这个隔离的等级是逻辑隔离，即在不同上下文中用户代码/变量等不会冲突覆盖。

```JavaScript
const vm = require('vm');

// 预设的全局变量
const sandbox = {
  result: 0
};

// 加载的用户代码
const code = `
  for (let i = 1; i <= 10; i++) {
    result += i;
  }
`;

vm.createContext(sandbox);
vm.runInContext(code, sandbox);

console.log(sandbox.result);
```

不过这个方法存在很多局限性，首先 vm 环境并不是真正的隔离环境，用户可以通过一些[**逃逸技术**](https://gist.github.com/maple3142/f491557a4e0a3afce1c1a5f36ae50332)来获得宿主环境的执行权限，同时如果用户代码存在死循环等问题，宿主环境的后续代码将无法继续执行。

### chroot jail

chroot 是一个 Linux 命令用于更改当前正在运行的进程及其子进程的根目录。某个进程经过 chroot 操作后，它的根目录将被锁定在命令参数所指定的位置，使得该进程及其子进程无法访问和操作该目录之外的其他文件。

一般来说，我们可以通过 busybox 模拟一个包含常见命令的 linux 根目录，然后通过 chroot 创建一个以该路径为根的新环境。

如果我们想以 BusyBox 路径为根目录，并在其中运行/bin/sh，我们可以使用以下命令：

```shell
chroot /path/to/busybox /bin/sh
```

执行此命令后，我们将进入一个以 BusyBox 为根的新 shell 界面。在这个界面中，我们无法访问原系统的根目录结构和文件，只能访问和操作 BusyBox 路径下的文件和目录。

通过这样的机制，使用 chroot 将用户进程对文件的读取限制在指定目录下，就可以避免用户代码对外界文件的访问，从而达到文件隔离的目的。

### WASM

WASM 是一个起初用于 web 的二进制指令集，用于在计算密集场景中替代 JavaScript。但是由于 web 的限制，实际上 WASM 是无法直接进行系统调用的，而目前很多运行时会额外提供 WASI 层（提供文件/网络模块等通用接口），这也给我们监控和限制用户代码提供了切入点。

如果用户代码使用 WASM 进行编写，我们可以通过 WASM 运行时进行加载和运行。

由于 golang 1.21 已经提供了 WASI 支持（在这之前需要用 tinygo 进行编译），所以我们可以编写一个简单的 golang 程序：

```go
package main

import (
	"bufio"
	"fmt"
	"os"
)

func main() {
	filename := "example.txt"
	file, err := os.Open(filename)
	if err != nil {
		fmt.Printf("Error opening file '%s': %v\n", filename, err)
		os.Exit(1)
	}
	defer file.Close()

	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		fmt.Println(scanner.Text())
	}

	if err := scanner.Err(); err != nil {
		fmt.Printf("Error reading file '%s': %v\n", filename, err)
		os.Exit(1)
	}
}
```

该程序的主要意图是读取当前目录下 example.txt 文件，并输出到 stdout。

编译命令如下：

```shell
GOOS=wasip1 GOARCH=wasm go build -o main.wasm
```

运行 main.wasm 可以用 [wazero](https://github.com/tetratelabs/wazero) 库，代码参考自 wazero 的示例。

注意，示例中使用 embed 命令将 `testdata/example.txt` 嵌入到可执行文件中，并去掉了 testdata 前缀，相当于在运行时模拟了一个文件系统，而不依赖外界文件。

```go
package main

import (
	"context"
	"embed"
	"fmt"
	"io/fs"
	"log"
	"os"

	"github.com/tetratelabs/wazero"
	"github.com/tetratelabs/wazero/imports/wasi_snapshot_preview1"
	"github.com/tetratelabs/wazero/sys"
)

// 将 example.txt 直接 embed 进可执行文件
//go:embed testdata/example.txt
var catFS embed.FS

func main() {
	// Choose the context to use for function calls.
	ctx := context.Background()

	// Create a new WebAssembly Runtime.
	r := wazero.NewRuntime(ctx)
	defer r.Close(ctx) // This closes everything this Runtime created.

	// Since wazero uses fs.FS, we can use standard libraries to do things like trim the leading path.
	rooted, err := fs.Sub(catFS, "testdata")
	if err != nil {
		log.Panicln(err)
	}

	// Combine the above into our baseline config, overriding defaults.
	config := wazero.NewModuleConfig().
		// By default, I/O streams are discarded and there's no file system.
		WithStdout(os.Stdout).WithStderr(os.Stderr).WithFS(rooted)

	// Instantiate WASI, which implements system I/O such as console output.
	wasi_snapshot_preview1.MustInstantiate(ctx, r)

	// InstantiateModule runs the "_start" function, WASI's "main".
	// * Set the program name (arg[0]) to "wasi"; arg[1] should be "/test.txt".
	if _, err = r.InstantiateWithConfig(ctx, catWasm, config.WithArgs("wasi", os.Args[1])); err != nil {
		if exitErr, ok := err.(*sys.ExitError); ok && exitErr.ExitCode() != 0 {
			fmt.Fprintf(os.Stderr, "exit_code: %d\n", exitErr.ExitCode())
		} else if !ok {
			log.Panicln(err)
		}
	}
}
```

WASM 的程序具有快速启动，易于控制的特点（不需要依赖虚拟化等重量技术），并且 IO 边界易于控制和定制，所以在轻量 Serverless 运行时中经常被采用。

### LD_PRELOAD

动态链接的 Linux 程序加载依赖时，会优先去 LD_PRELOAD 中找需要加载的方法。

比如进行文件 write 操作时，如果 LD_PRELOAD 中已经提供了 write 函数，则会。这样相当于劫持了程序对 libc.so 的调用。

在劫持后的系统调用中，可以加入鉴权或者改写返回代码。

这种方法也被用在知名的代理程序 proxychains 中，通过对网络相关调用进行 hook，可以将动态链接程序的网络请求转发到代理服务器中。

如果需要对程序永久添加依赖，可以用命令：

```bash
patchelf --add-needed libsandbox.so ./helloworld
```

不过该方案的绕过策略并不难，如果用户程序是静态编译的，或者是程序直接进行系统调用，则该方案不会起效。

另外，用户代码也可以直接使用 dlopen 来调用 libc.so，或者直接进行系统调用，比如下面的例子就是通过两种方式来向屏幕输出自定义文字：

```C
#include <dlfcn.h>
#include <stdlib.h>

void libc_printf();
void systemcall_printf();

int main() {
  libc_printf(); // 引用 libc 进行系统调用
  systemcall_printf(); // 直接通过中断进行系统调用
}

void libc_printf() {
  void *handle;
  void (*printf)(const char *, ...);
  char *error;

  handle = dlopen("libc.so.6", RTLD_LAZY); // 通过 dlopen 直接加载 libc
  if (!handle) {
    exit(EXIT_FAILURE);
  }

  printf = (void (*)(const char *, ...))dlsym(handle, "printf"); // 读取符号名 printf
  if ((error = dlerror()) != NULL) {
    exit(EXIT_FAILURE);
  }

  printf("Hello, world!\n");
  dlclose(handle);
}

void systemcall_printf() {
  asm(
    "mov $1, %rax\n"
    "mov $1, %rdi\n"
    "mov $message, %rsi\n"
    "mov $13, %rdx\n"
    "syscall\n"
    "ret\n"
    "message:\n"
    ".ascii \"Hello, world!\\n\"\n"
  );
}
```

### ptrace + seccomp

ptrace 是 Linux 中对其它程序进行控制的一系列控制命令，一些编译型语言的 Debugger 也是基于 ptrace。他能在应用程序进行系统调用的时候触发中断事件，通知调试器。我们可以对系统调用进行捕获和过滤。

只要能够控制系统调用，那么你就能完全掌握程序的 IO 边界，比如当程序执行网络操作时，会发起 connect 系统调用，那么父进程会在子进程发起系统调用时得到通知，并可以读取或者修改该调用的参数。

下面用一个实际例子说明一下，首先是子进程代码：

```c
#include <stdio.h>
#include <unistd.h>
#include <sys/ptrace.h>
#include <sys/types.h>

int main() {
    // 通知父进程开始追踪我们
    if (ptrace(PTRACE_TRACEME, 0, NULL, NULL) == -1) {
        perror("ptrace");
        return 1;
    }

    // 替换当前进程的映像为新的程序（例如：/bin/wget）
    execlp("/bin/wget", "wget", "https://baidu.com");
    perror("execlp");
    return 1;
}
```

接着用 golang 来实现父进程：

```go
package main

import (
    "fmt"
    "os/exec"
    "syscall"
    "unsafe"
)

func main() {
    // 启动子进程，这里的 ./child 是上面子进程经过编译后的可执行文件
    child := exec.Command("./child")
    child.Start()

    // 追踪子进程
    var status syscall.WaitStatus
    var pid int = child.Process.Pid
    if err := syscall.PtraceAttach(pid); err != nil {
        panic(err)
    }
    defer syscall.PtraceDetach(pid)

    var orig_rax uintptr
    for {
        var regs syscall.PtraceRegs
        if err := syscall.PtraceGetRegs(pid, &regs); err != nil {
            panic(err)
        }

        // 保存原始的RAX值（系统调用号）
        if orig_rax == 0 {
            orig_rax = regs.Rax
        }

        // 检查系统调用是否是我们感兴趣的（例如：connect）
        if orig_rax == 281 { // connect的系统调用号，可能因系统而异，可以通过`ausyscall64`命令查询
            var addr uintptr = regs.Rdi // connect的第一个参数是套接字文件描述符的地址
            var sockaddr syscall.Sockaddr
            var sockaddr_len uintptr

            // 读取套接字地址结构的大小
            _, _, errno := syscall.Syscall(syscall.SYS_PEEKDATA, addr+16, uintptr(unsafe.Pointer(&sockaddr_len)), 0)
            if errno != 0 {
                panic(errno)
            }

            // 读取套接字地址结构的内容到sockaddr变量中
            _, _, errno = syscall.Syscall(syscall.SYS_PEEKDATA, addr, uintptr(unsafe.Pointer(&sockaddr)), sockaddr_len)
            if errno != 0 {
                panic(errno)
            }

            // 假设它是IPv4地址，你可以根据需要添加IPv6的支持
            var sock_in_addr *syscall.SockaddrInet4 = (*syscall.SockaddrInet4)(unsafe.Pointer(&sockaddr))
            fmt.Printf("Connecting to: %s:%d\n", sock_in_addr.Addr[:], sock_in_addr.Port)
        }

        // 让子进程继续执行，直到下一个系统调用或退出
        if err := syscall.PtraceSyscall(pid, 0); err != nil {
            panic(err)
        }

        var ws syscall.WaitStatus
        pid, err := syscall.Wait4(pid, &ws, syscall.WALL, nil)
        if pid == -1 {
            panic(err)
        }

        if ws.Exited() || ws.Signaled() {
            break
        }
    }
}
```

不过要注意的是，docker 容器默认通过 seccomp 限制 ptrace 相关指令的运行，如果需要在 docker 中运行 ptrace，需要对容器进行额外的配置，这里不多赘述。

由于 ptrace 命令是不需要管理员权限的，一个很知名的使用场景是 proot，可以在 android 中通过捕捉和虚拟系统调用，模拟出一个 linux 环境，而且无需 root。

ptrace 实际上是一个非常完美的隔离方案，但是它对性能的影响非常大，每次进程进行系统调用的时候都需要父进程发出指令继续运行。

### setrlimit

在 Unix 和类 Unix 系统中，setrlimit 函数可以用来设置进程的资源限制。这对于控制子进程的资源使用特别有用，比如限制其可以打开的文件数量、内存使用量、CPU 使用时间等。

```c
#include <sys/resource.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/wait.h>

int main() {
    struct rlimit rl;

    // 设置要限制的资源类型和限制值
    rl.rlim_cur = 10; // 软限制：当前进程可以打开的最大文件描述符数量
    rl.rlim_max = 20; // 硬限制：进程在整个生命周期中可以打开的最大文件描述符数量

    // 设置资源限制为 RLIMIT_NOFILE，即文件描述符的数量
    if (setrlimit(RLIMIT_NOFILE, &rl) == -1) {
        perror("setrlimit");
        exit(EXIT_FAILURE);
    }

    pid_t pid = fork();
    if (pid < 0) {
        perror("fork");
        exit(EXIT_FAILURE);
    } else if (pid == 0) {
        // 子进程代码
        // 在这里，子进程将受到之前设置的资源限制

        // 例如，尝试打开超过软限制的文件描述符数量将失败
        for (int i = 0; i < rl.rlim_cur + 1; ++i) {
            int fd = open("/dev/null", O_RDONLY);
            if (fd == -1) {
                printf("Failed to open file descriptor %d: %m\n", i);
                break;
            }
        }

        exit(EXIT_SUCCESS);
    } else {
        // 父进程代码
        int status;
        waitpid(pid, &status, 0); // 等待子进程结束
        if (WIFEXITED(status)) {
            printf("Child exited with code %d\n", WEXITSTATUS(status));
        } else if (WIFSIGNALED(status)) {
            printf("Child terminated by signal %d\n", WTERMSIG(status));
        }
    }

    return 0;
}
```

### Landlock

Landlock 是 Linux 内核在 5.13 中引入的[特性](https://docs.kernel.org/userspace-api/landlock.html)，能够细化控制程序的网络或者文件访问。

可以控制的行为如下：

```
struct landlock_ruleset_attr ruleset_attr = {
    .handled_access_fs =
        LANDLOCK_ACCESS_FS_EXECUTE |
        LANDLOCK_ACCESS_FS_WRITE_FILE |
        LANDLOCK_ACCESS_FS_READ_FILE |
        LANDLOCK_ACCESS_FS_READ_DIR |
        LANDLOCK_ACCESS_FS_REMOVE_DIR |
        LANDLOCK_ACCESS_FS_REMOVE_FILE |
        LANDLOCK_ACCESS_FS_MAKE_CHAR |
        LANDLOCK_ACCESS_FS_MAKE_DIR |
        LANDLOCK_ACCESS_FS_MAKE_REG |
        LANDLOCK_ACCESS_FS_MAKE_SOCK |
        LANDLOCK_ACCESS_FS_MAKE_FIFO |
        LANDLOCK_ACCESS_FS_MAKE_BLOCK |
        LANDLOCK_ACCESS_FS_MAKE_SYM |
        LANDLOCK_ACCESS_FS_REFER |
        LANDLOCK_ACCESS_FS_TRUNCATE,
    .handled_access_net =
        LANDLOCK_ACCESS_NET_BIND_TCP |
        LANDLOCK_ACCESS_NET_CONNECT_TCP,
};
```

可以使用 go-landlock 库进行调用，比如下面的代码对 `/etc/passwd` 访问进行限制。

```go
package main

import (
  "fmt"
  "os"

  "github.com/shoenig/go-landlock"
)

func main() {
  l := landlock.New(
    landlock.File("/etc/os-release", "r"),
  )
  err := l.Lock(landlock.Mandatory)
  if err != nil {
    panic(err)
  }

  _, err = os.ReadFile("/etc/os-release")
  fmt.Println("reading /etc/os-release", err)

  _, err = os.ReadFile("/etc/passwd")
  fmt.Println("reading /etc/passwd", err)
}
```

运行输出：

```bash
go run main.go
reading /etc/os-release <nil>
reading /etc/passwd open /etc/passwd: permission denied
```

LandLock 本身由 bpf 技术实现，逻辑在内核态进行实现，效率非常高，并且对于网络和文件访问能做精确控制，比如限制文件删除/控制访问端口等。但是目前技术比较新，可能会有内核的兼容性问题。

### BPF

在谈了使用使用 BPF 的 LandLock 技术，那么我们进一步聊一下 BPF 技术。BPF 是一种内核观测技术，通过有限的内核虚拟机执行用户指令，观察资源指标。你需要做的就是编写 BPF 程序，检测监控核心变量，并由用户态程序判断是否执行某些操作。

BPF 程序本身是 c 语言的子集，可以使用 BCC（BPF Compiler Collection）将 BPF 程序编译并加载到内核中。

由于 BPF 更多用来观察资源，对进程进行限制需要额外的逻辑配合（比如 kill 掉资源使用过量的程序），所以下面给的例子使用 python 来示范：

```python
from bcc import BPF
import time

# 要监控的进程 PID，你可以根据需要修改这个值
target_pid = 12345  # 替换为实际的 PID

# 构造 BPF 代码，动态插入目标 PID
bpf_code_template = """
#include <uapi/linux/ptrace.h>
#include <bcc/proto.h>

BPF_HISTOGRAM(mmap_size);
BPF_MAP_DEF(pid_mem_usage, u32, u64, MAX_PID, 0);

int trace_mmap(struct __user *start, unsigned long len, unsigned long prot,
               unsigned long flags, unsigned long fd, unsigned long offset) {
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    if (pid == TARGET_PID) {
        u64 *mem_usage = pid_mem_usage.lookup(&pid);
        if (mem_usage) {
            *mem_usage += len;
        } else {
            pid_mem_usage.update_existing(&pid, &len);
        }
    }
    return 0;
}

int trace_munmap(struct __user *start, unsigned long len) {
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    if (pid == TARGET_PID) {
        u64 *mem_usage = pid_mem_usage.lookup(&pid);
        if (mem_usage) {
            if (*mem_usage >= len) {
                *mem_usage -= len;
            } else {
                *mem_usage = 0;
            }
        }
    }
    return 0;
}
"""

# 将 TARGET_PID 占位符替换为实际的 PID 值
bpf_code = bpf_code_template.replace("TARGET_PID", str(target_pid))

# 加载 BPF 程序
b = BPF(text=bpf_code)
b.attach_kprobe(event="do_mmap_pgoff", fn_name="trace_mmap")  # 注意在某些内核版本中可能需要使用 do_mmap_pgoff 而不是 mmap_pgoff
b.attach_kprobe(event="do_munmap", fn_name="trace_munmap")    # 注意在某些内核版本中可能需要使用 do_munmap

try:
    print(f"Monitoring memory usage for PID {target_pid}...")
    while True:
        mem_usage = b["pid_mem_usage"][target_pid]
        if mem_usage is not None:
            print(f"Current memory usage for PID {target_pid}: {mem_usage} bytes")
        time.sleep(5)  # 每隔 5 秒报告一次内存使用情况
except KeyboardInterrupt:
    pass
finally:
    b.detach_kprobe(event="do_mmap_pgoff")  # 或者 mmap_pgoff，根据你的内核版本
    b.detach_kprobe(event="do_munmap")       # 或者 munmap，根据你的内核版本
    b["pid_mem_usage"].clear()
```

## Container 以及 namespace/cgroups

这里我把容器技术和 namespace/cgroups 一起讲，但不代表容器一定基于 namespace/cgroups。

docker 作为容器的一个很经典实现，使用 namespace/cgroups 进行隔离。这个方案比前面一些隔离的好处是，通过指定 namespace，用户可以方便进入同一个空间进行调试。

由于该技术比较成熟且重型，我不再进行详细阐述。

另外容器不一定只使用 namespace/cgroups，还有很多使用虚拟机技术进行隔离的，比如 AWS 的 firecracker 直接通过 KVM 技术创建和管理虚拟机运行容器进程。

