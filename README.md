# KQMalloc介绍

## 最新消息

- [2026-09-30]：发布KQMalloc V0.22，面向鲲鹏930处理器大幅提升aligned_alloc()等对齐分配函数及valloc()、pvalloc()性能；新增对齐对象本地缓存阈值和两类本地缓存低水位配置，降低分配开销。
- [2026-06-30]：发布KQMalloc V0.20，大幅提升单线程和GREAT内存分配基准性能，优化最大64MB大对象高频分配并降低生产者-消费者场景的内存占用；重构堆标记缓冲区，新增分级堆缓存和Quick Cache阈值配置，并优化脏页缓存阈值配置。
- [2026-03-30]：发布KQMalloc V0.19，大幅提升并行处理能力，支持数千线程和TB级内存。并新增鲲鹏HIP10A、fork()、malloc_trim()及最高128MB对齐分配支持；重构巨型对象分配，完善脏页阈值、内存分析和调试能力，统一64KB与2MB页大小的二进制版本。

## 项目介绍

KQMalloc（Kunpeng Quick Malloc）是专为鲲鹏处理器设计的并行内存分配器，针对Linux操作系统下4KB和64KB系统页大小、48位内存虚拟地址的系统场景进行了适配与优化。KQMalloc使用透明大页等技术减少页表缓冲未命中、降低缓存碎片和缓存别名问题。

本仓库提供KQMalloc预编译动态库，面向应用部署、测试和运维用户。客户下载并解压软件包后，可通过`LD_PRELOAD`预加载动态库，直接用于已有的动态链接应用。

V0.22动态库初始化时按单线程模式运行，检测到第二个线程后自动切换到多线程模式。从V0.21版本开始，KQMalloc不再发布独立的单线程动态库；本仓V0.20及V0.19 beta0软件包仍保留旧版库名，使用时请按对应版本选择。

## 特性介绍

**表1** KQMalloc特性介绍

| 特性名称 | 特性介绍 |
| --- | --- |
| 多级无锁缓存快速路径 | 每个线程拥有独立的本地缓存，小对象优先通过本地快速路径分配。未命中时逐级回退到Quick Cache（CAS原子操作的无锁缓存）和Fast Cache（自适应容量的共享缓存）分配。 |
| 大对象原地扩缩 | 大对象realloc通过虚拟内存提交/取消提交（commit/decommit）原地调整大小，避免分配新内存再拷贝。 |
| 精细的大小分级与低碎片堆选择 | 将对象按大小划分为100+级别，小对象以4字节为步进、大对象以缓存行为步进，每次分配匹配最接近的槽位。 |
| 双向堆脏页回收 | 尾部裁剪（tail clean）：当堆驻留超过阈值，通过MADV_DONTNEED释放尾部多余页面；堆压缩（compression）回收尾部虚拟地址空间；中部回收（body clean）：对释放后的中间区域排序合并，向相邻空闲cell扩展以凑齐页面对齐边界后统一释放。 |

以上为KQMalloc的主要特性，各版本的功能差异请参见《[版本说明书](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/release_notes.md)》。

## 目录结构

本仓库按版本存放软件包及其完整性校验文件，目录结构如下。

```text
kqmalloc-release/
├── README.md
├── public_sys-resources/
│   └── icon-note.gif
├── v0.19-beta0/
│   ├── kqmalloc-v0.19-beta0.tar.gz
│   └── SHA256SUMS
├── v0.20.0/
│   ├── kqmalloc-v0.20.0.tar.gz
│   ├── kqmalloc-v0.20.0.tar.gz.json
│   └── SHA256SUMS
└── v0.22.0/
    ├── kqmalloc-v0.22.0.tar.gz
    └── SHA256SUMS
```

每个软件包解压后均包含HIP08、HIP09、HIP10A、HIP11、HIP12五种处理器目录，对应的动态库位于`<处理器型号>/lib/`。请选择与实际处理器型号对应的目录。

## 版本说明

本仓库提供以下3个版本包，各版本的功能更新请参见《[版本说明书](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/release_notes.md)》。

**表2** 软件包与适用用户

| 版本 | 软件包 | 内容与适用用户 |
| --- | --- | --- |
| V0.22 | [kqmalloc-v0.22.0.tar.gz](./v0.22.0/kqmalloc-v0.22.0.tar.gz) | 每种处理器提供6种动态库，覆盖常规运行、指针标记、调试和分析场景；建议新接入用户优先使用。 |
| V0.20 | [kqmalloc-v0.20.0.tar.gz](./v0.20.0/kqmalloc-v0.20.0.tar.gz) | 每种处理器提供4种动态库，适用于已有V0.20部署、需要保持原版本或进行版本对比的用户。 |
| V0.19 beta0 | [kqmalloc-v0.19-beta0.tar.gz](./v0.19-beta0/kqmalloc-v0.19-beta0.tar.gz) | 每种处理器提供4种动态库，为历史测试版本，适用于已有环境复现和版本对比。 |

## 环境兼容性信息

**表3** KQMalloc预编译动态库环境要求

| 项目 | 要求 |
| --- | --- |
| 处理器 | 鲲鹏处理器，按实际型号选择HIP08、HIP09、HIP10A、HIP11或HIP12目录 |
| 操作系统 | Linux，已验证的操作系统及配套环境见《[安装指南](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/installation_guide.md)》 |
| glibc | 2.34及以上，可通过`getconf GNU_LIBC_VERSION`检查 |
| 系统页大小 | 4KB或64KB |
| 虚拟地址 | 48位内存虚拟地址场景 |
| 指针标记 | 使用`.ptag`系列动态库时，需确认系统和应用均支持指针标记；存在兼容性限制时选择不带`.ptag`的对应动态库 |

>![表示说明的图片](./public_sys-resources/icon-note.gif) **说明：**
>
>本仓库提供的预编译动态库不支持glibc低于2.34的环境，加载时会报找不到`GLIBC_2.34`符号版本的错误，无法正常使用。请在运行应用前确认glibc版本。

## 安装部署

### 获取与校验软件包

从[版本说明](#版本说明)中选择并下载软件包及对应的`SHA256SUMS`文件。本仓软件包来自[官方发布目录](https://repo.boostkit.osinfra.cn/boostcore/kqmalloc/release/)，三个版本的压缩包均已与官方文件核对SHA256，内容一致，包名与官方保持一致。

**表4** 软件包完整性校验文件与来源记录

| 版本 | SHA256校验文件 | 来源记录 |
| --- | --- | --- |
| V0.22 | [SHA256SUMS](./v0.22.0/SHA256SUMS) | [官方溯源文件](https://repo.boostkit.osinfra.cn/boostcore/kqmalloc/release/v0.22.0/kqmalloc-v0.22.0.tar.gz.json) |
| V0.20 | [SHA256SUMS](./v0.20.0/SHA256SUMS) | [本仓补充来源记录](./v0.20.0/kqmalloc-v0.20.0.tar.gz.json) |
| V0.19 beta0 | [SHA256SUMS](./v0.19-beta0/SHA256SUMS) | [官方溯源文件](https://repo.boostkit.osinfra.cn/boostcore/kqmalloc/release/v0.19-beta0/kqmalloc-v0.19-beta0.tar.gz.json) |

V0.20的官方目录未提供JSON溯源文件，本仓补充记录其下载来源和SHA256，不提供未经确认的源码提交或构建时间。

将软件包及对应版本的`SHA256SUMS`放在同一目录中，在Linux上执行以下命令。

```shell
sha256sum -c SHA256SUMS
```

输出中对应软件包显示`OK`后再解压；若校验失败，请重新下载并核对。

### 解压软件包

以下以V0.22软件包为例。

```shell
tar -xzf kqmalloc-v0.22.0.tar.gz
kqmalloc_root="$(pwd)/kqmalloc-v0.22.0"
```

### 选择动态库

V0.22每种处理器目录下的`lib`子目录包含以下6种动态库。

**表5** V0.22动态库说明<a id="v022-library-table"></a>

| 动态库 | 用途 | 适用用户 / 场景 |
| --- | --- | --- |
| libkqmalloc.so | 常规发布库，不使用指针标记 | 常规业务部署；内核不支持指针标记，或Go、Java等运行时存在指针标记兼容性限制的应用。 |
| libkqmalloc.ptag.so | 启用指针标记的发布库 | 已确认系统及应用支持指针标记、希望评估该特性的用户。 |
| libkqmalloc.debug.so | 不使用指针标记，启用断言及重复释放等检查 | 排查内存分配、释放问题的开发和测试人员。 |
| libkqmalloc.ptag.debug.so | 启用指针标记及调试检查 | 排查指针标记版本问题的开发和测试人员。 |
| libkqmalloc.profiler.so | 不使用指针标记，启用内存分析 | 分析内存分配行为和开销的性能测试人员。 |
| libkqmalloc.ptag.profiler.so | 启用指针标记及内存分析 | 分析指针标记版本分配行为和开销的性能测试人员。 |

V0.22的6种库均支持单线程和多线程应用。调试库和分析库会引入额外开销，日常业务及正式性能测试应选用发布库。

V0.20及V0.19 beta0每种处理器目录下提供以下4种旧版动态库。

**表6** V0.20及V0.19 beta0动态库说明<a id="legacy-library-table"></a>

| 动态库 | 用途与适用场景 |
| --- | --- |
| libkqmalloc.so | 单线程库，仅用于单线程应用。 |
| libkqmallocmt.so | 多线程发布库，用于多线程业务及性能测试。 |
| libkqmallocmt.debug.so | 多线程调试库，用于排查内存分配、释放问题。 |
| libkqmallocmt.profiler.so | 多线程分析库，用于分析内存分配行为和开销。 |

请结合软件包版本和完整路径区分同名库。以HIP09处理器上的多线程应用为例：

- 使用V0.22包时，选择`kqmalloc-v0.22.0/HIP09/lib/libkqmalloc.so`。
- 使用V0.20包时，选择`kqmalloc-v0.20.0/HIP09/lib/libkqmallocmt.so`。
- 使用V0.19 beta0包时，选择`kqmalloc-v0.19-beta0/HIP09/lib/libkqmallocmt.so`。

V0.20及V0.19 beta0包中的`libkqmalloc.so`仅用于单线程应用，这两个版本不提供`.ptag`系列动态库。

### 启用透明大页

KQMalloc使用透明大页减少地址转换开销。运行应用前，可查看系统页大小和透明大页状态。

```shell
getconf PAGESIZE
cat /sys/kernel/mm/transparent_hugepage/enabled
cat /sys/kernel/mm/transparent_hugepage/defrag
```

若需启用透明大页，可由系统管理员执行以下命令，将`enabled`设为`always`，将`defrag`设为`madvise`。

```shell
echo always > /sys/kernel/mm/transparent_hugepage/enabled
echo madvise > /sys/kernel/mm/transparent_hugepage/defrag
```

上述命令修改系统级设置，配置方法和预期结果参见《[安装指南](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/installation_guide.md#启用透明大页)》。

### 使用KQMalloc

KQMalloc通过替换标准库中的内存分配函数实现优化。通过`LD_PRELOAD`预加载动态库即可用于已有的动态链接应用，无需修改应用代码。

以下以V0.22、HIP09处理器上的常规发布库为例，请按实际解压位置、处理器型号和应用命令修改路径及参数。

```shell
kqmalloc_lib="$kqmalloc_root/HIP09/lib/libkqmalloc.so"
LD_PRELOAD="$kqmalloc_lib" ./your_application arg1 arg2
```

需要使用指针标记、调试或分析功能时，请根据软件包版本查阅[表5 V0.22动态库说明](#v022-library-table)或[表6 V0.20及V0.19 beta0动态库说明](#legacy-library-table)，将`kqmalloc_lib`设置为所选库文件的完整路径。V0.20及V0.19 beta0多线程应用应选择`libkqmallocmt.so`。

### 安装后验证

可在相同环境下使用`ldd`命令检查动态库加载路径。

```shell
LD_PRELOAD="$kqmalloc_lib" ldd ./your_application
```

输出应包含所选KQMalloc动态库的实际路径，且没有`not found`或预加载失败提示。随后运行应用，确认业务结果正确。

### 移除KQMalloc

从应用启动命令或服务配置中移除KQMalloc的`LD_PRELOAD`设置，并重新启动应用。

## 快速入门

完成上述下载、校验、解压和预加载步骤后，即可使用KQMalloc。环境变量配置和性能验证方法请参见《[快速入门](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/quick_start.md)》。

评估加速效果时，请在相同环境及业务输入下，对比系统默认内存库与KQMalloc的运行时间、吞吐量和内存占用，并检查业务结果是否一致。动态库成功加载仅说明接入成功，实际收益以业务测试结果为准。

## 学习文档

**表7** KQMalloc学习文档

| 学习文档名称 | 学习文档简介 |
| --- | --- |
| [快速入门](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/quick_start.md) | 介绍预编译动态库的使用方法、环境变量和性能验证。 |
| [安装指南](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/installation_guide.md) | 介绍运行环境、软件包获取、动态库使用及移除方法。 |
| [API参考](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/api_reference.md) | 介绍malloc、calloc、realloc、free等函数。 |
| [版本说明书](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/release_notes.md) | 介绍各版本的功能更新和配套信息。 |
| [项目介绍](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/README.md#项目介绍) | 介绍KQMalloc项目、主要特性及配套文档。 |

配套文档当前面向V0.22；使用旧版软件包时，请以本页的旧版库名及对应版本说明为准。

## 贡献声明

欢迎大家为社区做贡献。如果使用过程中有任何问题或建议，或者需要反馈软件包交付、部署使用及文档问题，可以通过[Issues](https://gitcode.com/boostkit/kqmalloc-release/issues)联系我们。具体参与方式可参考[社区贡献指南](https://gitcode.com/boostkit/community/blob/master/docs/contributor/contributing.md)。

## 免责声明

本仓库提供KQMalloc预编译动态库及其使用说明。实际功能、兼容性和性能表现以所选版本及运行环境为准。请密切关注社区发布的通知、版本更新和安全信息，并结合业务场景完成操作系统安全加固。

## 许可证书

本项目的文档适用CC-BY 4.0许可证，具体请参见[文档许可证书](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/LICENSE)。

## 致谢

感谢来自社区的每一个PR，欢迎贡献KQMalloc！
