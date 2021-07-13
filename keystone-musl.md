---
title: keystone-musl
layout: default
---

[**Go to Top Page**](https://uyiromo.github.io)

# Introduction
- This page describes how to build Keystone eapp using the musl libc
  - [musl libc](https://musl.libc.org/)
  - [Keystone](https://github.com/keystone-enclave/keystone)
- As described the following issue, Keystone has some bugs related to glibc initialization as of Jul. 14, 2021.
  - [Runtime Page faults when running hello.ke](https://github.com/keystone-enclave/keystone/issues/229)
- We can avoid it by using musl libc instaed of glibc
  

# How to build eapp by using musl libc
- This section describes how to build [hello example in keystone-sdk](https://github.com/keystone-enclave/keystone-sdk/tree/master/examples/hello).
  - It prints message in eapp directly.

## 1. Setup musl C compiler for RISC-V
- download and extract
```shell
wget https://musl.cc/riscv64-linux-musl-cross.tgz
tar -zxf riscv64-linux-musl-cross.tgz
```
-- Toolchains are extracted at `<riscv64-linux-musl-cross>/bin/`

## 2. Configure keystone-sdk repository to use musl toolchains
```shell
cd <keystone-sdk>
git diff
diff --git a/macros.cmake b/macros.cmake
index 34a53cd..a29a05f 100644
--- a/macros.cmake
+++ b/macros.cmake
@@ -7,7 +7,8 @@ macro(global_set Name Value)
 endmacro()
 
 macro(use_riscv_toolchain bits)
-  set(cross_compile riscv${bits}-unknown-linux-gnu-)
+  #set(cross_compile riscv${bits}-unknown-linux-gnu-)
+  set(cross_compile <riscv64-linux-musl-cross>/bin/riscv64-linux-musl-)
   execute_process(
     COMMAND which ${cross_compile}gcc
     OUTPUT_VARIABLE CROSSCOMPILE
```

## 3. Some compiler flags
- Ofc, musl toolchains will generate binaries dinamically linked to them.
  - When target RISC-V boards have not them, linkes show errors.
  - We can avoid the errors by using static linking.

```shell
cd <keystone-sdk>
git diff
diff --git a/examples/hello/CMakeLists.txt b/examples/hello/CMakeLists.txt
index 9f5ee48..9e4c1b8 100644
--- a/examples/hello/CMakeLists.txt
+++ b/examples/hello/CMakeLists.txt
@@ -19,7 +19,7 @@ target_link_libraries(${eapp_bin} "-static")
 # host
 
 add_executable(${host_bin} ${host_src})
-target_link_libraries(${host_bin} ${KEYSTONE_LIB_HOST} ${KEYSTONE_LIB_EDGE})
+target_link_libraries(${host_bin} ${KEYSTONE_LIB_HOST} ${KEYSTONE_LIB_EDGE} "-static")
 
 # add target for Eyrie runtime (see keystone.cmake)
diff --git a/examples/tests/CMakeLists.txt b/examples/tests/CMakeLists.txt
index 837013f..3b22e8d 100644
--- a/examples/tests/CMakeLists.txt
+++ b/examples/tests/CMakeLists.txt
@@ -98,7 +98,7 @@ set_target_properties(${all_test_bins}
 # host
 
 add_executable(${host_bin} ${host_src})
-target_link_libraries(${host_bin} ${KEYSTONE_LIB_HOST} ${KEYSTONE_LIB_EDGE} ${KEYSTONE_LIB_VERIFIER})
+target_link_libraries(${host_bin} ${KEYSTONE_LIB_HOST} ${KEYSTONE_LIB_EDGE} ${KEYSTONE_LIB_VERIFIER} "-static")
 
 # add target for Eyrie runtime (see keystone.cmake)
 
diff --git a/macros.cmake b/macros.cmake
index 34a53cd..a29a05f 100644
--- a/macros.cmake
+++ b/macros.cmake
@@ -28,7 +29,7 @@ macro(use_riscv_toolchain bits)
   set(AR              ${CROSSCOMPILE}ar)
   set(OBJCOPY         ${CROSSCOMPILE}objcopy)
   set(OBJDUMP         ${CROSSCOMPILE}objdump)
-  set(CFLAGS          "-Wall -Werror")
+  set(CFLAGS          "-Wall -Werror -static")
 
   global_set(CMAKE_C_COMPILER        ${CC}${EXT})
   global_set(CMAKE_ASM_COMPILER        ${CC}${EXT})
```

<!--
## 4. Some modification for header conflicts
- `riscv64-linux-musl/bits/alltypes.h`
```shell
git diff <path/to/riscv64-linux-musl-cross>/riscv64-linux-musl/include/bits/alltypes.h
index b4a7df6..18a151d 100644
--- a/riscv64-linux-musl/include/bits/alltypes.h
+++ b/riscv64-linux-musl/include/bits/alltypes.h
@@ -2,7 +2,7 @@
 #define _Int64 long
 #define _Reg long
    
-#define __BYTE_ORDER 1234
+//#define __BYTE_ORDER 1234
  #define __LONG_MAX 0x7fffffffffffffffL

  #ifndef __cplusplus
 @@ -224,7 +224,7 @@ struct timeval { time_t tv_sec; suseconds_t tv_usec; };
  #endif
  
  #if defined(__NEED_struct_timespec) && !defined(__DEFINED_struct_timespec)
 -struct timespec { time_t tv_sec; int :8*(sizeof(time_t)-sizeof(long))*(__BYTE_ORDER==4321); long tv_nsec; int :8*(sizeof(time_t)-sizeof(long))*(__BYTE_ORDER!=4321); };
 +struct timespec { time_t tv_sec; int :8*(sizeof(time_t)-sizeof(long))*(1234==4321); long tv_nsec; int :8*(sizeof(time_t)-sizeof(long))*(1234!=4321); };
  #define __DEFINED_struct_timespec
  #endif
```
-->

## 4. Build and Run
- Now, you should be able to build examples and run




















