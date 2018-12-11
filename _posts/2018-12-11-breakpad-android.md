---
layout: article
title: Breakpad for Android
tags: crash
key: 2018-12-11-breakpad-android.md
---

# breakpad for android


breakpad源码：https://github.com/google/breakpad

使用的版本是：3bc301d4f9fd00911cbb26c90a9e6e829c4b14bc


## 简介


breakpad是google推出的一个用于崩溃收集相关的一整套框架，支持了大部分主流平台（windows、linux、macos、android、ios）。其中主要包括三个工具。
* libbreakpad_client.a 用于客户端使用来收集崩溃信息的库
* dump_syms 用于导出库文件的符号信息的工具
* minidump_stackwalk 用于解析dmp文件获取栈信息的工具


## 工具生成

先根据源码编译生成上述的三个文件。

### libbreakpad_client.a

参考 https://github.com/google/breakpad/blob/master/README.ANDROID

我使用的是在linux下交叉编译arm平台的库文件。需要android的ndk相关工具。上面的文档里列出了两种方法，一种是通过直接集成android.mk文件进行集成，另一种相对灵活，使用独立工具链进行编译。我这里采用独立工具链的方式。

#### 独立工具链

我使用的ndk版本r18。
先安装独立工具链，可以参考文档 https://developer.android.com/ndk/guides/standalone_toolchain?hl=en

```sh
$DNK/build/tools/make_standalone_toolchain.py \
    --arch arm --api 21 --install-dir /tmp/my-android-toolchain
```
执行完上述命令即可将独立工具链安装到/tmp/my-android-toolchain目录下。
```
export PATH=/tmp/my-android-toolchain/bin:$PATH
```
在将bin目录添加到path路径下。到这里独立工具链的准备就完成了。

#### 编译

```sh
$GOOGLE_BREAKPAD_PATH/configure --host=arm-linux-androideabi \
                                --disable-processor
                                --disable-tools
make -j4
```
执行上面命令进行编译，最后生成的库文件路径是src/client/linux/libbreakpad_client.a。

我在编译时遇到了一些错误，我对breakpad源码做了如下的修改来保证编译通过。

```patch
--- /tmp/WbYnr0_wchar.h
+++ /search/odin/develop/src/breakpad-new/src/src/common/android/testing/include/wchar.h
@@ -47,13 +47,13 @@
 extern "C" {
 #endif  // __cplusplus
 
-static wchar_t inline wcstolower(wchar_t ch) {
+wchar_t inline wcstolower(wchar_t ch) {
   if (ch >= L'a' && ch <= L'A')
     ch -= L'a' - L'A';
   return ch;
 }
 
-static int inline wcscasecmp(const wchar_t* s1, const wchar_t* s2) {
+int inline wcscasecmp(const wchar_t* s1, const wchar_t* s2) {
   for (;;) {
     wchar_t c1 = wcstolower(*s1);
     wchar_t c2 = wcstolower(*s2);



--- /tmp/NZYuSL_guid_creator.cc
+++ /search/odin/develop/src/breakpad-new/src/src/common/linux/guid_creator.cc
@@ -45,6 +45,10 @@
 
 #if defined(HAVE_SYS_RANDOM_H)
 #include <sys/random.h>
+#endif
+
+#if defined(HAVE_ARC4RANDOM)
+#include <cstring>
 #endif
 
 //


```

最终生成的libbreakpad_client.a文件大小约为2.9MB。

### dump_syms 、 minidump_stackwalk

生成linux平台下的可执行文件。其他平台的配置参数可以参考 https://chromium.googlesource.com/breakpad/breakpad/+/master/docs/

```sh
$GOOGLE_BREAKPAD_PATH/configure
make -j4
```

生成的文件分别位于
* src/tools/linux/dump_syms/dump_syms
* src/processor/minidump_stackwalk


## 客户端接入

1. 添加包含目录 $GOOGLE_BREAKPAD_PATH/src
2. 静态链接生成的libbreakpad_client.a

```cpp
#include "client/linux/handler/exception_handler.h"

static bool dumpCallback(const google_breakpad::MinidumpDescriptor& descriptor,
void* context, bool succeeded) {
  printf("Dump path: %s\n", descriptor.path());
  return succeeded;
}

void crash() { volatile int* a = (int*)(NULL); *a = 1; }

int main(int argc, char* argv[]) {
  google_breakpad::MinidumpDescriptor descriptor("/sdcard");
  google_breakpad::ExceptionHandler eh(descriptor, NULL, dumpCallback, NULL, true, -1);
  crash();
  return 0;
}
```
exception_handling相关文档 https://chromium.googlesource.com/breakpad/breakpad/+/master/docs/exception_handling.md

## 生成符号

使用dump_syms工具生成.sym文本格式符号文件。
符号文件格式可以参考 https://chromium.googlesource.com/breakpad/breakpad/+/master/docs/symbol_files.md
```
$GOOGLE_BREAKPAD_PATH/src/tools/linux/dump_syms test > test.sym
```
符号文件应该存放的路径，以便于后面的dump解析程序能够定位。
```
head -n1 test.sym MODULE Linux arm 6EDC6ACDB282125843FD59DA9C81BD830 test
mkdir -p ./symbols/test/6EDC6ACDB282125843FD59DA9C81BD830
mv test.sym ./symbols/test/6EDC6ACDB282125843FD59DA9C81BD830
```


## dump文件解析

使用minidump_stackwalk工具可以根据崩溃生成的dmp文件以及dump_syms工具生成的sym符号文件，就能生成文本形式的已解析的栈信息。
```
$GOOGLE_BREAKPAD_PATH/src/processor/minidump_stackwalk minidump.dmp ./symbols
```
生成的栈信息格式

```txt
Operating system: Android
                  0.0.0 Linux 3.18.20-perf-g9751366-dirty #1 SMP PREEMPT Tue Mar 21 13:14:51 CST 2017 armv8l
CPU: arm
     ARMv1 Qualcomm part(0x51002150) features: half,thumb,fastmult,vfpv2,edsp,neon,vfpv3,tls,vfpv4,idiva,idivt
     4 CPUs

GPU: UNKNOWN

Crash reason:  SIGSEGV /SEGV_MAPERR
Crash address: 0x0
Process uptime: not available

Thread 0 (crashed)
 0  libnative-lib.so!crash() [native-lib.cpp : 18 + 0x2]
     r0 = 0x00000000    r1 = 0x00000001    r2 = 0xffd47ba3    r3 = 0xf44b38de
     r4 = 0xffd47c40    r5 = 0xffd47d30    r6 = 0x0000001c    r7 = 0xffd47c58
     r8 = 0xf44b7f8c    r9 = 0xf44b7f00   r10 = 0xffd47c80   r12 = 0xf3a1ac34
     fp = 0xf44b7f00    sp = 0xffd47bb4    lr = 0xf39f627d    pc = 0xf39f6212
    Found by: given as instruction pointer in context
 1  libnative-lib.so!__ashldi3 + 0x2bb5
     sp = 0xffd47bd0    pc = 0xf3a138c0
    Found by: stack scanning
 2  libart.so + 0xba356
     sp = 0xffd47be4    pc = 0xf3e67358
    Found by: stack scanning
 3  libnative-lib.so!__ashldi3 + 0x2bb5
     sp = 0xffd47bf0    pc = 0xf3a138c0
    Found by: stack scanning
 4  dalvik-main space (deleted) + 0xdc43e
     sp = 0xffd47c14    pc = 0x12cdc440
    Found by: stack scanning
 5  dalvik-main space (deleted) + 0x1c06be
     sp = 0xffd47c34    pc = 0x12dc06c0
    Found by: stack scanning
 6  boot.art + 0x9bb76
     sp = 0xffd47c38    pc = 0x7009bb78
    Found by: stack scanning
 7  dalvik-main space (deleted) + 0x1c06be
     sp = 0xffd47c3c    pc = 0x12dc06c0
    Found by: stack scanning
 8  dalvik-main space (deleted) + 0xbcf1e
     sp = 0xffd47c40    pc = 0x12cbcf20
    Found by: stack scanning
 9  dalvik-LinearAlloc (deleted) + 0x1450e
     sp = 0xffd47c54    pc = 0xbdd2f510
    Found by: stack scanning
10  libart.so + 0xeaca9
     sp = 0xffd47c60    pc = 0xf3e97cab
    Found by: stack scanning
11  dalvik-main space (deleted) + 0x2ab7fe
     sp = 0xffd47c64    pc = 0x12eab800
    Found by: stack scanning
12  dalvik-main space (deleted) + 0x1c05fe
     sp = 0xffd47c6c    pc = 0x12dc0600
    Found by: stack scanning
13  dalvik-LinearAlloc (deleted) + 0x1450e
     sp = 0xffd47c74    pc = 0xbdd2f510
    Found by: stack scanning
14  dalvik-main space (deleted) + 0xe4fe
     sp = 0xffd47c80    pc = 0x12c0e500
    Found by: stack scanning
15  dalvik-LinearAlloc (deleted) + 0x1450e
     sp = 0xffd47c84    pc = 0xbdd2f510
    Found by: stack scanning
16  boot.art + 0x9856a
     sp = 0xffd47c88    pc = 0x7009856c
    Found by: stack scanning
17  boot.art + 0x9331a6
     sp = 0xffd47c8c    pc = 0x709331a8
    Found by: stack scanning
18  libart.so + 0xe65b1
     sp = 0xffd47ca8    pc = 0xf3e935b3
    Found by: stack scanning
19  dalvik-main space (deleted) + 0xe4fe
     sp = 0xffd47cb0    pc = 0x12c0e500
    Found by: stack scanning
20  dalvik-main space (deleted) + 0xe4fe
     sp = 0xffd47ccc    pc = 0x12c0e500
    Found by: stack scanning
21  boot.oat + 0x33c0461
     sp = 0xffd47cd4    pc = 0x73eb6463
    Found by: stack scanning
22  dalvik-LinearAlloc (deleted) + 0x1450e
     sp = 0xffd47ce8    pc = 0xbdd2f510
    Found by: stack scanning
23  libart.so + 0xe65b1
     sp = 0xffd47cf0    pc = 0xf3e935b3
    Found by: stack scanning
```

解析dmp相关文档 https://chromium.googlesource.com/breakpad/breakpad/+/master/docs/processor_design.md