# KQMalloc 预编译动态库

KQMalloc（Kunpeng Quick Malloc）是面向鲲鹏处理器的高性能内存分配器，适用于内存申请、释放较频繁的应用。本仓库提供可直接使用的 `.so` 动态库，面向应用部署、测试和运维用户，无需获取 KQMalloc 源码或编译库，通过 `LD_PRELOAD` 即可接入已有的动态链接应用。

## 软件包与适用用户

本仓库提供 **3 个版本包**，每个包均包含 HIP08、HIP09、HIP10A、HIP11、HIP12 五种处理器的动态库。解压后，请选择与实际处理器型号对应的 `<处理器型号>/lib/` 目录。

| 版本 | 软件包 | 内容与适用用户 |
| --- | --- | --- |
| V0.22 | [kqmalloc-v0.22.0.tar.gz](./v0.22.0/kqmalloc-v0.22.0.tar.gz) | 每种处理器提供 6 种动态库，覆盖常规运行、指针标记、调试和分析场景；建议新接入用户优先使用。 |
| V0.20 | [kqmalloc-v0.20.0.tar.gz](./v0.20.0/kqmalloc-v0.20.0.tar.gz) | 每种处理器提供 4 种动态库，适用于已有 V0.20 部署、需要保持原版本或进行版本对比的用户。 |
| V0.19 beta0 | [kqmalloc-v0.19-beta0.gz](./v0.19-beta0/kqmalloc-v0.19-beta0.gz) | 每种处理器提供 4 种动态库，为历史测试版本，适用于已有环境复现和版本对比。 |

### V0.22：如何选择动态库

| 动态库 | 用途 | 适用用户 / 场景 |
| --- | --- | --- |
| `libkqmalloc.so` | 常规发布库，不使用指针标记 | 常规业务部署；内核不支持指针标记，或 Go、Java 等运行时存在指针标记兼容性限制的应用。 |
| `libkqmalloc.ptag.so` | 启用指针标记的发布库 | 已确认系统及应用支持指针标记、希望评估该特性的用户。 |
| `libkqmalloc.debug.so` | 不使用指针标记，启用断言及重复释放等检查 | 排查内存分配、释放问题的开发和测试人员。 |
| `libkqmalloc.ptag.debug.so` | 启用指针标记及调试检查 | 排查指针标记版本问题的开发和测试人员。 |
| `libkqmalloc.profiler.so` | 不使用指针标记，启用内存分析 | 分析内存分配行为和开销的性能测试人员。 |
| `libkqmalloc.ptag.profiler.so` | 启用指针标记及内存分析 | 分析指针标记版本分配行为和开销的性能测试人员。 |

V0.22 的六种库均支持单线程和多线程应用：初始化时按单线程模式运行，检测到第二个线程后自动切换为多线程模式。调试库和分析库会引入额外开销，日常业务及正式性能测试应选用发布库。

### V0.20 / V0.19 beta0：旧版库名

| 动态库 | 用途与适用场景 |
| --- | --- |
| `libkqmalloc.so` | 单线程库，仅用于单线程应用。 |
| `libkqmallocmt.so` | 多线程发布库，用于多线程业务及性能测试。 |
| `libkqmallocmt.debug.so` | 多线程调试库，用于排查内存分配、释放问题。 |
| `libkqmallocmt.profiler.so` | 多线程分析库，用于分析内存分配行为和开销。 |

**注意：旧版 `libkqmalloc.so` 与 V0.22 的同名库含义不同；旧版没有 `.ptag` 库，使用时须按对应版本选择库名。**

## 直接使用

适用环境为鲲鹏处理器上的 Linux，支持 4KB / 64KB 系统页和 48 位虚拟地址场景。已验证的操作系统及配套环境见[安装指南](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/installation_guide.md)。

> **glibc 版本要求：glibc >= 2.34。** 本仓库提供的预编译动态库**不支持 glibc 低于 2.34 的环境**，加载动态库时会报找不到 `GLIBC_2.34` 符号版本的错误，无法正常使用。请在运行应用前确认 glibc 版本；配套文档中的环境信息也须满足此要求。

```bash
getconf GNU_LIBC_VERSION
```

以下以 **V0.22、HIP09、常规发布库**为例，在 Linux 上执行：

```bash
# 解压下载的软件包
tar -xzf kqmalloc-v0.22.0.tar.gz

# 按实际解压位置和处理器型号修改路径
kqmalloc_root="$(pwd)/kqmalloc-v0.22.0"
kqmalloc_lib="$kqmalloc_root/HIP09/lib/libkqmalloc.so"

# 将程序路径及参数替换为实际业务命令
LD_PRELOAD="$kqmalloc_lib" ./your_application arg1 arg2
```

需要使用指针标记、调试或分析功能时，将 `kqmalloc_lib` 改为上表中的对应库文件。V0.20 多线程应用应选择 `libkqmallocmt.so`。V0.19 beta0 的文件名虽以 `.gz` 结尾，也可用 `tar -xzf kqmalloc-v0.19-beta0.gz` 解压。

可用以下命令检查动态库加载路径，输出应包含所选 KQMalloc 库，且没有 `not found` 或预加载失败提示：

```bash
LD_PRELOAD="$kqmalloc_lib" ldd ./your_application
```

加载成功后，使用同一业务输入对比启用前后的运行时间、吞吐量和内存占用，确认业务结果正确并评估实际收益。

## 配套文档

- [安装指南](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/installation_guide.md)：环境要求、动态库使用及移除。
- [快速入门](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/quick_start.md)：环境变量和性能验证方法。
- [版本说明](https://gitcode.com/boostkit/boostcore/blob/master/docs/zh/kqmalloc/release_notes.md)：各版本的功能变化。

配套文档当前面向 V0.22；使用旧版软件包时，请以本页的旧版库名及对应版本说明为准。
