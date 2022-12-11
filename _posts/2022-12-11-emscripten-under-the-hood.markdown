---
layout: post
title:  "Emscripten: Under the hood"
date:   2022-12-11 22:16:37 +0800
categories: tech
---

## 前言
WebSDK 在智创云已经驱动了模板预览/混剪/卡片模板以及通用视频编辑器，内置的 WASM 模块由 Emscripten 从 VE C++ 编译而成，附带一些 JS 胶水代码。

本文面向已经写过 Emscripten 三方库的老手和从没听说过 Emscripten 的前端开发者。将努力从不同视角还原 Emscripten 事实标准框架的运行原理，打破 WASM 黑盒，收获 WASM 和原生应用的性能&架构差异；通过对比理解 JavaScript 中一些理所当然现象背后隐藏的复杂逻辑。

ChatGPT 对 Emscripten 的理解:
![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707658658021.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

## 发布会

### 大一统

1974 年，贝尔实验室正式对外发布 Unix 及其源代码，这款分时操作系统的设计哲学如此统一，使其能在不同制造商机器上运行。Unix 非常受欢迎，一些像 BSD(**B**erkeley **S**oftware **D**istribution) 还有 Sun 公司的 Solaris 等等       Unix-like OS 相继冒了出来。到 80 年中期，各种衍生系统被 Unix 发行厂商加入新功能，越来越个性化。使软件在 OS 之间相互移植变得越来越困难，这严重违背了 [Unix 哲学](https://en.wikipedia.org/wiki/Unix_philosophy)："*Choose portability over efficiency.*"。于是 IEEE 为了拨乱反正，开始插手制定基于 Unix 的标准，涵盖网络、线程、文件 IO 以及 C 语言接口等，甚至包括开关机流程，标准定义了整个操作系统，由[自由软件运动](https://zh.wikipedia.org/wiki/%E8%87%AA%E7%94%B1%E8%BD%AF%E4%BB%B6%E8%BF%90%E5%8A%A8)精神领袖 Richard Stallman 命名为 [POSIX](https://en.wikipedia.org/wiki/POSIX)。

Linus 在《Just for Fun》中提到当年因为没有获取 POSIX 标准的渠道，几乎完全是照着 Sun 公司的 Unix 手册在写 Linux，因此很多 Unix 程序也能轻松迁移到 Linux。另外 Microsoft 为了拉新也跟风推出了 POSIX-``compliant 系统 Windows NT，嵌入到内核中。帮助 Win10 推出了 [WSL](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux)，1.0 版本即能无需编译、原生运行 Linux 应用。

### libc

POSIX 通过 C 语言声明了**系统调用**，C 语言也有自己的标准库，在历史发展中两者逐渐融合，前者或后者都可以用来称呼 libc。当前 Linux 最流行的 libc 实现诞生于 [GNU 计划](https://en.wikipedia.org/wiki/GNU_Project) 的 glibc。

glibc 包含系统调用声明和**库函数**(比如: `strlen`)，一般程序在被 GCC 或 clang 编译时会将库函数生成的对象文件静态链接到产物中，系统调用则依据 POSIX 规范生成指定汇编代码。因而只依赖 glibc 的程序在 macOS 和 Linux 之间迁移非常简单。

iOS 和 Android 都属于 unix-like 系统，应用无法做到源码移植的原因是在自家系统封装了 XNU 和 AOSP。构建 APP 大量依赖了 POSIX 之外的 API，两者关系如下图：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707662327548.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


> 使用源码安装像 Homebrew 这样的原生应用，需要安装 Xcode 开发者工具，因为其内置了 libc 和 Darwin 专属的系统调用声明。

几乎所有应用都通过 libc 实现系统调用，包括 Node.js 和 Python3.x 等解释器。因为系统调用有必要会切换到内核态的缘故，性能远低于普通函数。日常开发可使用 strace 或 dtruss 跟踪系统调用。

POSIX 系统横截面示意图:
![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707662663569.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


printf 最终把结果写入 /dev/stdout [设备文件](https://en.wikipedia.org/wiki/Device_file) 中，系统会将指令发送到对应驱动，在终端显示"hello world"。

### C++ to Web
WebAssembly 设计之初就考虑了 Web 的移植和性能问题，现在 LLVM backend 已经能输出 WASM 格式的二进制汇编文件。Emscripten 集成了 LLVM clang 和 backend 把 C++ 转换成 WASM，Emscripten 工作流程:

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707665265496.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

LLVM IR 规范让混合编译变得更加容易，backend 标准化了交叉编译的输入输出，现在 Rust/Golang 都能较轻松地转换到 WASM。而 Emscripten 比 WASM 技术还要古老，1.37.3 版本后才支持 WASM，在 WASM 之前仅支持 asm.js 格式。除了 wasm 之外，还包括 glue 胶水代码，除了管理 wasm 生命周期，WebAssembly 从设计之初就无法访问浏览器环境，因此用 JS 实现了 POSIX 中的系统调用 API。Emscripten横截面示意图:

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707667242743.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


如图，除却模块生命周期管理代码外，灰色区域逻辑由胶水代码负责：
1. 系统调用，Emscripten 会把 C++ 中依赖的系统调用 API 替换成 JS 代码，如 printf 最终会执行到 console.log，核心胶水代码
2. 胶水接口，一些系统调用能力也会通过 JavaScript 接口公开暴露到全局上下文 Module 中，例如 FS 文件系统
3. JS in C++，Emscripten 通过[宏定义](https://baike.baidu.com/item/C%2B%2B%E5%AE%8F%E5%AE%9A%E4%B9%89/462816)提供了在 C++ 中写 JS 代码的能力，编译后对应 JS 逻辑会放到胶水代码中

## 开箱

### 外观介绍

安装 Emscripten 需要先克隆 [emsdk](https://github.com/emscripten-core/emsdk) 到本地，内置 LLVM、Node.js、Python 以及 Java 等工具集，提供一个完整的编译环境。emsdk 提供了自动/手动安装指定 [emscripten-core/emscripten](https://github.com/emscripten-core/emscripten) 版本的命令，后者囊括了构建脚本、胶水代码库以及测试用例和官方文档；Emscripten 版本指的也是它，更版频率大约 1~2 周，最新已达 3.1.27。

**机身结构**

为了保证环境统一，emsdk 还可以安装/查询/管理 python/node/llvm/[musl](https://musl.libc.org/) 等依赖库，musl 是 Emscripten 使用的开源 libc。使用编译环境需要先执行 emsdk_env.sh，内部把各依赖的 /bin 路径添加到 $PATH 中。

安装好的 emscripten 内包含了胶水 JS 文件、C/C++ 库、以及测试用例。编译命令脚本 emcc/em++ 的编译选项定义在 settings.h 中，有 231 个配置项。include/emscripten.h 定义 Emscripten 单独提供的函数、宏定义以及类型别名，比如 emscripten_debugger，会调用 JS 的 debugger，从而快速打断点调试。

**快速上手**

通过测试用例学习某个库如何使用，查看生成的胶水代码可以单步调试分析内部流程逻辑，C/C++ 也一样，单独开启 source-map 后复原库和业务代码：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707670081510.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


### 开机启动

使用编译选项 `-sUSE_PTHREADS` 可开启多线程能力支持，PThread 即 POSIX Thread。编译产物包含 wasm 模块、wasm.js 胶水代码和 worker.js 线程(worker)启动脚本。单独 wasm 模块只用 `WebAssembly.instantiate` 就能完成初始化，但无法享受到胶水代码的 POSIX API 实现和其它便捷函数。

脚本模式下 wasm.js 作为 script 引入，自执行会触发初始化逻辑，通过回调通知外部 init 完成。Emscripten 生命周期钩子、第三方插件逻辑等基本都可以用回调赋值和编译替换两种方式注入。加载运行完整产物的流程如下：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707670576122.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

图中仅代表代码在文件中自上而下的顺序，与实际执行时序无关。如图，与原生应用中 main 函数作为应用执行的入口不同，WASM 中的 main 函数不影响程序的生命周期，其返回也不意味着 "WASM 程序" 终止。因而 Web 使用 WASM 模块并非内嵌了一个黑盒 APP，反而像是引入了一个**状态库**，库中有个 main 函数在初始化阶段可选的调用，C/C++ 除了变量声明赋值无法执行逻辑在函数体外。消息循环应用迁到 WASM 可使用 `emscripten_set_main_loop` 模拟：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707671628324.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


`emscripten_set_main_loop` 第二个参数表示循环周期，如果是 0 则使用 requestAnimationFrame。

**run 函数**
run 函数负责调用各种 pre/post 回调钩子，完成生成 WASM 实例后的初始化。时机有 **wasm.js** 自执行阶段和 **removeRunDependency** 移除完所有依赖。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707674174191.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


所有内置依赖都会同步的生成到 wasm.js 中，比如同样是自执行的 createWasm 函数，函数入口会增加 `wasm-instantiate` 依赖，直到编译、实例化 WASM 模块才移除依赖(执行 run 函数)。还有动态库预载(如果有) `preloadDylibs` 和 IndexedDB 缓存同步 `syncfs` 等，这些依赖在自执行阶段被添加，全都完成再触发 run 函数，通过 `onRuntimeInitialized` 告知业务初始化结束。

**pre/post-js**

`--pre-js=files` 和 `--post-js=files` 两个编译选项注入代码到文件头/尾，比如使用 pre-js 可以拿到真实脚本开始执行时间，避免下载和代码解析干扰。Emscripten 构建脚本 **emcc** 会用宏定义和槽位匹配两种方式替换生成代码。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707675132257.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

**线程生命周期**

除了在 C/C++ 中使用 POSIX 头文件 <pthread.h> 里的 API `pthread_create`，还可以在编译时增加                     `-sPTHREAD_POOL_SIZE=n` 参数，使胶水代码 wasm.js 于自执行阶段创建 n 条线程。线程创建流程如下：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707676469179.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


如果 unsedWorkers 池中无空闲线程，则新建 worker：

1. 发送 `load` 事件，带上已经构建好的 `[WebAssembly.Memory](https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/Memory){shared: true}`、 [WebAssembly.Module](https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/Module) 以及胶水代码 wasm.js
2. worker.js 利用 wasm.js 同步地创建 WASM 实例，返回 `loaded` 事件给主线程
3. 主线程发送 run 事件，worker 接收后开始初始化线程，记录线程启动时间和分配堆栈上下限

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707677277457.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


销毁线程分为回收和完全杀死两种，回收释放线程内存，worker 可留作下一次复用，节约启动时间；杀死则彻底销毁 worker。

## 拆机
### 产物分析
完整的 Emscripten 产物包含三个文件：
1. .wasm 二进制文件，C/C++ 逻辑转译产物
2. wasm.js，负责开启启动，还有承载了 POSIX 在 Web 上的模拟实现
3. worker.js，PThread 的 worker 实现，建立和主线程绑定关系后交给 wasm.js 启动

**转译 C/C++**
Emscripten 内部使用 clang 编译 C/C++ 代码，被编译的代码分为业务工程代码、内部框架代码和静态库三部分。复杂的业务工程可能需要由 make 或 Ninja 等规则工具。emcc 在交给 clang 之前，会把 musl 中 POSIX 系统声明以**文件粒度**替换成 Emscripten 实现，使用 .py 脚本拼接硬编码，生成 Ninja 规则，指导**链接顺序**：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707677925828.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


这些文件里的函数依托 Web 能力，最终将调用 JS 实现。以 POSIX 杀死线程 pthread_kill API举例，通过脚本完成 POSIX 库文件替换，前后区别：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707678181972.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


如图，`__pthread_kill_js` 是一个"外部实现"函数，C/C++ 只有函数声明，编译到 WASM 将生成一条     import 指令，声明实例化 WASM 需要传入的 JS 函数。[import 指令](https://webassembly.github.io/spec/core/syntax/modules.html#imports)是一个二级结构，其它编译工具默认一级名称为 env。Emscripten 共实现了 82 个替换文件。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707678471900.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


musl libc 中没有的函数，需要引入 libpng 和 libogg 等第三方库，增加命令行参数 `-sUSE_LIBPNG` 给 emcc，emcc 从 [Emscripten 依赖静态库列表](https://storage.googleapis.com/webassembly/) 下载静态库 .a，链接阶段 wasm-ld 把 .a 放到命令参数**靠右**位置，业务工程被链接的 .o 靠左，链接器 wasm-ld 从左到右解析文件，遇到函数声明就加入符号表，发现定义则删除。遍历一轮符号表不为空，报出错误 xxx 函数没找到。

> clang 编译生成的 .o 对象文件和第三方 .a 静态库，甚至 .so 动态库都遵从 [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 规范。不同的是 .o 和 .a 比 IR 更接近源代码，基本算是二进制的源码，可以把几个 .o 加入打包进另一个 .a。动态库与平台绑定，不参与程序本体的编译、链接流程。


**胶水 JS**
wasm.js 和 worker.js 来自于多份"预处理 JS"经 `parseTools.js` 解析，按行匹配代码，检测到宏执行对应操作，相当于一个模板引擎。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707679376283.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

如图：
1. 断言宏，callMain 执行时如果开启了断言(settings.ASSERTIONS = 1)，保留 #3 代码。运行阶段发现 run 依赖没清空则终止运行并报错。
2. include 宏，用 `Fetch.js` 内容替换当前位置。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707679550969.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


预处理完的代码会交给 `eval` 执行 `mergeInto`，将函数定义收集到统一对象 "Module" ，经过"二次预处理"后，输出到 wasm.js，挂载到 `Module['asm']`，在实例化 WASM 时传给它。"__sig"会生成一份"函数定义"，遇到 C++ 函数重载，要求签名一致。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707679877173.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


VESDK 快速导入读取 Blob 函数。

> 运行时动态生成函数可通过 WebAssembly.Table 实现，原理是 JS 和 C++ 侧把各自的函数对象、函数指针绑定到 Table 指定 index 中，Table 完成封装。

由于 WASM 汇编指令函数声明的 [参数列表/返回值类型](https://webassembly.github.io/spec/core/syntax/types.html#syntax-valtype) 只能是整数或浮点，传递复杂数据时，只能当指针用：

| 数据类型(C视角)| C/C++ 取值 | JS 取值 |
| --- | --- | --- |
| 整数 | WebAseembly 已原生支持 | WebAseembly 已原生支持 |
| 字符串 | 需要 JS 预先调用 malloc 在 C/C++ 申请一段内存，序列化 JS 字符串到内存。使用时遵循谁申请，谁释放原则 | 依靠字符串末尾 \0 这份约定，尝试读取 url 下方内存，直到指向的值是 \0 为止 |
| 整型数组 | JS 申请 4*n 大小内存，把 JS 长度为 n 数组复制到内存中。需要 JS 组头指针和数组长度两个参数 | 需要 C/C++ 额外告知数组长度 |
| 结构体 | 以 C/C++ 对象为模板时，需要 JS 理解 C/C++ 对象模型，申请内存后，按模型赋值 | 同样需要 JS 理解 C/C++ 对象模型，按模型取值 |

Emscripten 提供了 WebAssembly.Memory.buffer 的多种 HEAP View。在**取对应类型值得把指针按 item 长度整除**。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707683541121.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


结构体传递时，JS 理解 C/C++ 对象模型非常困难，还好 Emscripten 编译阶段提供了偏移计算语法：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707683632824.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


如图，fetchXHR 是定义在 library_fetch.js 的函数，为 C/C++ 提供网络能力支持。调用 fetchXHR 前，C/C++ 侧先申请一段 emscripten_fetch_t 结构体对应大小的内存，拿到内存首字节指针，给偏移量为"url"赋值。JS 则按相同偏移取值。

### 文件系统
Emscripten 可以在浏览器和 Node.js 运行，编译到 Node.js 使用自带的 fs API；浏览器出于安全考虑，无法访问宿主文件系统。而且 JS 只能异步读取文件 buffer，而 C/C++ 使用 POSIX API 同步读取，因此 Emscripten 提供了一套虚拟文件系统的 POSIX 实现，C/C++ 使用这套 FS 可直接 include fstream、stdio.h 这两个头文件。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707683786211.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


**MEMFS**

全称 Memory FS，使用纯 JS 模拟了一套文件系统，文件本体作为 buffer 在内存里（WebAssembly.Memory 外）；因此子线程读写文件需要代理到主线程：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707683962957.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


如图，Emscripten 许多 POSIX 能力都会代理到主线程执行，线程读取文件，创建代理任务投递到主线程执行，并陷入自旋锁，直到任务完成后解开，因此非常依赖主线程执行能力。如果主线程繁忙，worker 性能将会一起被拖累。

**wasmFS**

随着 [wasmfs](https://github.com/emscripten-core/emscripten/issues/15041) 的推进与 [OPFS](https://chromestatus.com/feature/5702777582911488) 标准的上线，这个问题将得到很大的改善，它将文件数管理移到了C++层，从而充分利用 SharedArrayBuffer 的跨线程能力，避免了所有操作代理到主线程的问题。
Emscripten: Under the hood

## 前言
WebSDK 在智创云已经驱动了模板预览/混剪/卡片模板以及通用视频编辑器，内置的 WASM 模块由 Emscripten 从 VE C++ 编译而成，附带一些 JS 胶水代码。

本文面向已经写过 Emscripten 三方库的老手和从没听说过 Emscripten 的前端开发者。将努力从不同视角还原 Emscripten 事实标准框架的运行原理，打破 WASM 黑盒，收获 WASM 和原生应用的性能&架构差异；通过对比理解 JavaScript 中一些理所当然现象背后隐藏的复杂逻辑。

ChatGPT 对 Emscripten 的理解:
![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707658658021.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

## 发布会

### 大一统

1974 年，贝尔实验室正式对外发布 Unix 及其源代码，这款分时操作系统的设计哲学如此统一，使其能在不同制造商机器上运行。Unix 非常受欢迎，一些像 BSD(**B**erkeley **S**oftware **D**istribution) 还有 Sun 公司的 Solaris 等等       Unix-like OS 相继冒了出来。到 80 年中期，各种衍生系统被 Unix 发行厂商加入新功能，越来越个性化。使软件在 OS 之间相互移植变得越来越困难，这严重违背了 [Unix 哲学](https://en.wikipedia.org/wiki/Unix_philosophy)："*Choose portability over efficiency.*"。于是 IEEE 为了拨乱反正，开始插手制定基于 Unix 的标准，涵盖网络、线程、文件 IO 以及 C 语言接口等，甚至包括开关机流程，标准定义了整个操作系统，由[自由软件运动](https://zh.wikipedia.org/wiki/%E8%87%AA%E7%94%B1%E8%BD%AF%E4%BB%B6%E8%BF%90%E5%8A%A8)精神领袖 Richard Stallman 命名为 [POSIX](https://en.wikipedia.org/wiki/POSIX)。

Linus 在《Just for Fun》中提到当年因为没有获取 POSIX 标准的渠道，几乎完全是照着 Sun 公司的 Unix 手册在写 Linux，因此很多 Unix 程序也能轻松迁移到 Linux。另外 Microsoft 为了拉新也跟风推出了 POSIX-``compliant 系统 Windows NT，嵌入到内核中。帮助 Win10 推出了 [WSL](https://en.wikipedia.org/wiki/Windows_Subsystem_for_Linux)，1.0 版本即能无需编译、原生运行 Linux 应用。

### libc

POSIX 通过 C 语言声明了**系统调用**，C 语言也有自己的标准库，在历史发展中两者逐渐融合，前者或后者都可以用来称呼 libc。当前 Linux 最流行的 libc 实现诞生于 [GNU 计划](https://en.wikipedia.org/wiki/GNU_Project) 的 glibc。

glibc 包含系统调用声明和**库函数**(比如: `strlen`)，一般程序在被 GCC 或 clang 编译时会将库函数生成的对象文件静态链接到产物中，系统调用则依据 POSIX 规范生成指定汇编代码。因而只依赖 glibc 的程序在 macOS 和 Linux 之间迁移非常简单。

iOS 和 Android 都属于 unix-like 系统，应用无法做到源码移植的原因是在自家系统封装了 XNU 和 AOSP。构建 APP 大量依赖了 POSIX 之外的 API，两者关系如下图：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707662327548.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


> 使用源码安装像 Homebrew 这样的原生应用，需要安装 Xcode 开发者工具，因为其内置了 libc 和 Darwin 专属的系统调用声明。

几乎所有应用都通过 libc 实现系统调用，包括 Node.js 和 Python3.x 等解释器。因为系统调用有必要会切换到内核态的缘故，性能远低于普通函数。日常开发可使用 strace 或 dtruss 跟踪系统调用。

POSIX 系统横截面示意图:
![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707662663569.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


printf 最终把结果写入 /dev/stdout [设备文件](https://en.wikipedia.org/wiki/Device_file) 中，系统会将指令发送到对应驱动，在终端显示"hello world"。

### C++ to Web
WebAssembly 设计之初就考虑了 Web 的移植和性能问题，现在 LLVM backend 已经能输出 WASM 格式的二进制汇编文件。Emscripten 集成了 LLVM clang 和 backend 把 C++ 转换成 WASM，Emscripten 工作流程:

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707665265496.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

LLVM IR 规范让混合编译变得更加容易，backend 标准化了交叉编译的输入输出，现在 Rust/Golang 都能较轻松地转换到 WASM。而 Emscripten 比 WASM 技术还要古老，1.37.3 版本后才支持 WASM，在 WASM 之前仅支持 asm.js 格式。除了 wasm 之外，还包括 glue 胶水代码，除了管理 wasm 生命周期，WebAssembly 从设计之初就无法访问浏览器环境，因此用 JS 实现了 POSIX 中的系统调用 API。Emscripten横截面示意图:

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707667242743.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


如图，除却模块生命周期管理代码外，灰色区域逻辑由胶水代码负责：
1. 系统调用，Emscripten 会把 C++ 中依赖的系统调用 API 替换成 JS 代码，如 printf 最终会执行到 console.log，核心胶水代码
2. 胶水接口，一些系统调用能力也会通过 JavaScript 接口公开暴露到全局上下文 Module 中，例如 FS 文件系统
3. JS in C++，Emscripten 通过[宏定义](https://baike.baidu.com/item/C%2B%2B%E5%AE%8F%E5%AE%9A%E4%B9%89/462816)提供了在 C++ 中写 JS 代码的能力，编译后对应 JS 逻辑会放到胶水代码中

## 开箱

### 外观介绍

安装 Emscripten 需要先克隆 [emsdk](https://github.com/emscripten-core/emsdk) 到本地，内置 LLVM、Node.js、Python 以及 Java 等工具集，提供一个完整的编译环境。emsdk 提供了自动/手动安装指定 [emscripten-core/emscripten](https://github.com/emscripten-core/emscripten) 版本的命令，后者囊括了构建脚本、胶水代码库以及测试用例和官方文档；Emscripten 版本指的也是它，更版频率大约 1~2 周，最新已达 3.1.27。

**机身结构**

为了保证环境统一，emsdk 还可以安装/查询/管理 python/node/llvm/[musl](https://musl.libc.org/) 等依赖库，musl 是 Emscripten 使用的开源 libc。使用编译环境需要先执行 emsdk_env.sh，内部把各依赖的 /bin 路径添加到 $PATH 中。

安装好的 emscripten 内包含了胶水 JS 文件、C/C++ 库、以及测试用例。编译命令脚本 emcc/em++ 的编译选项定义在 settings.h 中，有 231 个配置项。include/emscripten.h 定义 Emscripten 单独提供的函数、宏定义以及类型别名，比如 emscripten_debugger，会调用 JS 的 debugger，从而快速打断点调试。

**快速上手**

通过测试用例学习某个库如何使用，查看生成的胶水代码可以单步调试分析内部流程逻辑，C/C++ 也一样，单独开启 source-map 后复原库和业务代码：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707670081510.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


### 开机启动

使用编译选项 `-sUSE_PTHREADS` 可开启多线程能力支持，PThread 即 POSIX Thread。编译产物包含 wasm 模块、wasm.js 胶水代码和 worker.js 线程(worker)启动脚本。单独 wasm 模块只用 `WebAssembly.instantiate` 就能完成初始化，但无法享受到胶水代码的 POSIX API 实现和其它便捷函数。

脚本模式下 wasm.js 作为 script 引入，自执行会触发初始化逻辑，通过回调通知外部 init 完成。Emscripten 生命周期钩子、第三方插件逻辑等基本都可以用回调赋值和编译替换两种方式注入。加载运行完整产物的流程如下：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707670576122.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

图中仅代表代码在文件中自上而下的顺序，与实际执行时序无关。如图，与原生应用中 main 函数作为应用执行的入口不同，WASM 中的 main 函数不影响程序的生命周期，其返回也不意味着 "WASM 程序" 终止。因而 Web 使用 WASM 模块并非内嵌了一个黑盒 APP，反而像是引入了一个**状态库**，库中有个 main 函数在初始化阶段可选的调用，C/C++ 除了变量声明赋值无法执行逻辑在函数体外。消息循环应用迁到 WASM 可使用 `emscripten_set_main_loop` 模拟：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707671628324.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


`emscripten_set_main_loop` 第二个参数表示循环周期，如果是 0 则使用 requestAnimationFrame。

**run 函数**
run 函数负责调用各种 pre/post 回调钩子，完成生成 WASM 实例后的初始化。时机有 **wasm.js** 自执行阶段和 **removeRunDependency** 移除完所有依赖。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707674174191.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


所有内置依赖都会同步的生成到 wasm.js 中，比如同样是自执行的 createWasm 函数，函数入口会增加 `wasm-instantiate` 依赖，直到编译、实例化 WASM 模块才移除依赖(执行 run 函数)。还有动态库预载(如果有) `preloadDylibs` 和 IndexedDB 缓存同步 `syncfs` 等，这些依赖在自执行阶段被添加，全都完成再触发 run 函数，通过 `onRuntimeInitialized` 告知业务初始化结束。

**pre/post-js**

`--pre-js=files` 和 `--post-js=files` 两个编译选项注入代码到文件头/尾，比如使用 pre-js 可以拿到真实脚本开始执行时间，避免下载和代码解析干扰。Emscripten 构建脚本 **emcc** 会用宏定义和槽位匹配两种方式替换生成代码。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707675132257.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

**线程生命周期**

除了在 C/C++ 中使用 POSIX 头文件 <pthread.h> 里的 API `pthread_create`，还可以在编译时增加                     `-sPTHREAD_POOL_SIZE=n` 参数，使胶水代码 wasm.js 于自执行阶段创建 n 条线程。线程创建流程如下：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707676469179.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


如果 unsedWorkers 池中无空闲线程，则新建 worker：

1. 发送 `load` 事件，带上已经构建好的 `[WebAssembly.Memory](https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/Memory){shared: true}`、 [WebAssembly.Module](https://developer.mozilla.org/en-US/docs/WebAssembly/JavaScript_interface/Module) 以及胶水代码 wasm.js
2. worker.js 利用 wasm.js 同步地创建 WASM 实例，返回 `loaded` 事件给主线程
3. 主线程发送 run 事件，worker 接收后开始初始化线程，记录线程启动时间和分配堆栈上下限

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707677277457.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


销毁线程分为回收和完全杀死两种，回收释放线程内存，worker 可留作下一次复用，节约启动时间；杀死则彻底销毁 worker。

## 拆机
### 产物分析
完整的 Emscripten 产物包含三个文件：
1. .wasm 二进制文件，C/C++ 逻辑转译产物
2. wasm.js，负责开启启动，还有承载了 POSIX 在 Web 上的模拟实现
3. worker.js，PThread 的 worker 实现，建立和主线程绑定关系后交给 wasm.js 启动

**转译 C/C++**
Emscripten 内部使用 clang 编译 C/C++ 代码，被编译的代码分为业务工程代码、内部框架代码和静态库三部分。复杂的业务工程可能需要由 make 或 Ninja 等规则工具。emcc 在交给 clang 之前，会把 musl 中 POSIX 系统声明以**文件粒度**替换成 Emscripten 实现，使用 .py 脚本拼接硬编码，生成 Ninja 规则，指导**链接顺序**：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707677925828.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


这些文件里的函数依托 Web 能力，最终将调用 JS 实现。以 POSIX 杀死线程 pthread_kill API举例，通过脚本完成 POSIX 库文件替换，前后区别：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707678181972.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


如图，`__pthread_kill_js` 是一个"外部实现"函数，C/C++ 只有函数声明，编译到 WASM 将生成一条     import 指令，声明实例化 WASM 需要传入的 JS 函数。[import 指令](https://webassembly.github.io/spec/core/syntax/modules.html#imports)是一个二级结构，其它编译工具默认一级名称为 env。Emscripten 共实现了 82 个替换文件。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707678471900.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


musl libc 中没有的函数，需要引入 libpng 和 libogg 等第三方库，增加命令行参数 `-sUSE_LIBPNG` 给 emcc，emcc 从 [Emscripten 依赖静态库列表](https://storage.googleapis.com/webassembly/) 下载静态库 .a，链接阶段 wasm-ld 把 .a 放到命令参数**靠右**位置，业务工程被链接的 .o 靠左，链接器 wasm-ld 从左到右解析文件，遇到函数声明就加入符号表，发现定义则删除。遍历一轮符号表不为空，报出错误 xxx 函数没找到。

> clang 编译生成的 .o 对象文件和第三方 .a 静态库，甚至 .so 动态库都遵从 [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 规范。不同的是 .o 和 .a 比 IR 更接近源代码，基本算是二进制的源码，可以把几个 .o 加入打包进另一个 .a。动态库与平台绑定，不参与程序本体的编译、链接流程。


**胶水 JS**
wasm.js 和 worker.js 来自于多份"预处理 JS"经 `parseTools.js` 解析，按行匹配代码，检测到宏执行对应操作，相当于一个模板引擎。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707679376283.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)

如图：
1. 断言宏，callMain 执行时如果开启了断言(settings.ASSERTIONS = 1)，保留 #3 代码。运行阶段发现 run 依赖没清空则终止运行并报错。
2. include 宏，用 `Fetch.js` 内容替换当前位置。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707679550969.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


预处理完的代码会交给 `eval` 执行 `mergeInto`，将函数定义收集到统一对象 "Module" ，经过"二次预处理"后，输出到 wasm.js，挂载到 `Module['asm']`，在实例化 WASM 时传给它。"__sig"会生成一份"函数定义"，遇到 C++ 函数重载，要求签名一致。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707679877173.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


VESDK 快速导入读取 Blob 函数。

> 运行时动态生成函数可通过 WebAssembly.Table 实现，原理是 JS 和 C++ 侧把各自的函数对象、函数指针绑定到 Table 指定 index 中，Table 完成封装。

由于 WASM 汇编指令函数声明的 [参数列表/返回值类型](https://webassembly.github.io/spec/core/syntax/types.html#syntax-valtype) 只能是整数或浮点，传递复杂数据时，只能当指针用：

| 数据类型(C视角)| C/C++ 取值 | JS 取值 |
| --- | --- | --- |
| 整数 | WebAseembly 已原生支持 | WebAseembly 已原生支持 |
| 字符串 | 需要 JS 预先调用 malloc 在 C/C++ 申请一段内存，序列化 JS 字符串到内存。使用时遵循谁申请，谁释放原则 | 依靠字符串末尾 \0 这份约定，尝试读取 url 下方内存，直到指向的值是 \0 为止 |
| 整型数组 | JS 申请 4*n 大小内存，把 JS 长度为 n 数组复制到内存中。需要 JS 组头指针和数组长度两个参数 | 需要 C/C++ 额外告知数组长度 |
| 结构体 | 以 C/C++ 对象为模板时，需要 JS 理解 C/C++ 对象模型，申请内存后，按模型赋值 | 同样需要 JS 理解 C/C++ 对象模型，按模型取值 |

Emscripten 提供了 WebAssembly.Memory.buffer 的多种 HEAP View。在**取对应类型值得把指针按 item 长度整除**。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707683541121.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


结构体传递时，JS 理解 C/C++ 对象模型非常困难，还好 Emscripten 编译阶段提供了偏移计算语法：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707683632824.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


如图，fetchXHR 是定义在 library_fetch.js 的函数，为 C/C++ 提供网络能力支持。调用 fetchXHR 前，C/C++ 侧先申请一段 emscripten_fetch_t 结构体对应大小的内存，拿到内存首字节指针，给偏移量为"url"赋值。JS 则按相同偏移取值。

### 文件系统
Emscripten 可以在浏览器和 Node.js 运行，编译到 Node.js 使用自带的 fs API；浏览器出于安全考虑，无法访问宿主文件系统。而且 JS 只能异步读取文件 buffer，而 C/C++ 使用 POSIX API 同步读取，因此 Emscripten 提供了一套虚拟文件系统的 POSIX 实现，C/C++ 使用这套 FS 可直接 include fstream、stdio.h 这两个头文件。

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707683786211.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


**MEMFS**

全称 Memory FS，使用纯 JS 模拟了一套文件系统，文件本体作为 buffer 在内存里（WebAssembly.Memory 外）；因此子线程读写文件需要代理到主线程：

![](http://mayflower-blog.oss-cn-beijing.aliyuncs.com/blog/16707683962957.jpg?x-oss-process=image/auto-orient,1/interlace,1/quality,q_80)


如图，Emscripten 许多 POSIX 能力都会代理到主线程执行，线程读取文件，创建代理任务投递到主线程执行，并陷入自旋锁，直到任务完成后解开，因此非常依赖主线程执行能力。如果主线程繁忙，worker 性能将会一起被拖累。

**wasmFS**

随着 [wasmfs](https://github.com/emscripten-core/emscripten/issues/15041) 的推进与 [OPFS](https://chromestatus.com/feature/5702777582911488) 标准的上线，这个问题将得到很大的改善，它将文件数管理移到了C++层，从而充分利用 SharedArrayBuffer 的跨线程能力，避免了所有操作代理到主线程的问题。
