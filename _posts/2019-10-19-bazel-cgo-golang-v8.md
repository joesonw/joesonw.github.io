---
layout: post
title: "使用Bazel编译CGO项目 (Golang + V8)"
---

# Why
最近在写一个小项目, 需要在 Go 执行 JS. 看过了otto, goja 甚至 go-ducktype. 但是出于性能角度的考虑, 还是想用 V8. 

社区有不少 Golang +V8 项目. 但是绝大部分早已停止了维护. 最终看下来 只有 [augustoroman/v8](https://github.com/augustoroman/v8) 这个 V8 的 Go wrapper 和 Node.js 的作者 Ry Dahl 的[v8worker2](https://github.com/ry/v8worker2).

其中 [v8worker2](https://github.com/ry/v8worker2). 并未暴露 V8 的API, 而是在其基础上包装了一个基于 ArrayBuffer <-> []byte 的接口, 而且也停止了维护. 而 [augustoroman/v8](https://github.com/augustoroman/v8) 却不能直接 `go get` 使用, 与 `GO111MODULE` 不兼容等问题.

于是便有了 [joesonw/js8](https://github.com/joesonw/js8) 这个库. 在把已经编译好的 V8 文件打包进来, 并加上对 `Bazel` 的支持.

# What

当然了, 其中涉及到的代码其实很少, 先贴出来, 再分析.

`WORKSPACE`
```bazel
workspace(
    name = "js8",
)

load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")
load("@bazel_tools//tools/build_defs/repo:git.bzl", "git_repository", "new_git_repository")

http_archive(
    name = "io_bazel_rules_go",
    urls = [
        "https://storage.googleapis.com/bazel-mirror/github.com/bazelbuild/rules_go/releases/download/0.19.4/rules_go-0.19.4.tar.gz",
        "https://github.com/bazelbuild/rules_go/releases/download/0.19.4/rules_go-0.19.4.tar.gz",
    ],
    sha256 = "ae8c36ff6e565f674c7a3692d6a9ea1096e4c1ade497272c2108a810fb39acd2",
)

load("@io_bazel_rules_go//go:deps.bzl", "go_rules_dependencies", "go_register_toolchains")
go_rules_dependencies()
go_register_toolchains()
```

`BUILD`
```bazel
package(default_visibility = ["//visibility:public"])
load("@io_bazel_rules_go//go:def.bzl", "go_library", "go_test")
load("@rules_cc//cc:defs.bzl", "cc_library")

cc_library(
    name = "libv8",
    srcs = select({
        "@io_bazel_rules_go//go/platform:darwin": glob(["libv8-darwin/libv8/*.a"]),
        "@io_bazel_rules_go//go/platform:linux": glob(["libv8-linux/libv8/*.a"]),
        "//conditions:default": [],
    }),
    hdrs = select({
       "@io_bazel_rules_go//go/platform:darwin": glob(["libv8-darwin/include/*.h", "libv8-darwin/include/libplatform/*.h"]),
       "@io_bazel_rules_go//go/platform:linux": glob(["libv8-linux/include/*.h", "libv8-linux/include/libplatform/*.h"]),
       "//conditions:default": [],
   }),
)

go_library(
    name = "go_default_library",
    srcs = [
        "doc.go",
        "kind.go",
        "v8_c_bridge.cc",
        "v8_c_bridge.h",
        "v8_create_darwin.go",
        "v8_create_linux.go",
        "v8_darwin.go",
        "v8_linux.go",
    ] + select({
        "@io_bazel_rules_go//go/platform:darwin": glob(["libv8-darwin/include/*.h", "libv8-darwin/include/libplatform/*.h"]),
        "@io_bazel_rules_go//go/platform:linux": glob(["libv8-linux/include/*.h", "libv8-linux/include/libplatform/*.h"]),
        "//conditions:default": [],
    }),
    cdeps = [":libv8"],
    cgo = True,
    importpath = "github.com/joesonw/js8",
)

go_test(
    name = "go_default_test",
    srcs = [
        "benchmarks_test.go",
        "example_bind_test.go",
        "examples_test.go",
        "kind_test.go",
        "v8_test.go",
    ],
    embed = [":go_default_library"],
)
```


#HOW

`WORKSPACE` 不用讲了, 基础操作, 加载 `rules_go` 并初始化相关 `dependencies`

为什么不在要使用的项目里面 直接用 `gazelle` 的 `go_repository` 来使用呢? 一开始也是这样想的, 但是发现 `go_repository` 产出的规则对 `cgo` 的支持不太好, 其中使用到的是 `copts` , 而这个项目是 `c++` 的, 可以从 `-std=c++11` 这看出, 这样就导致了编译失败. 

```bazel
copts = select({
    "@io_bazel_rules_go//go/platform:darwin": [
        "-I. -Ilibv8-darwin/include -fno-rtti -fpic -std=c++11",
        "-Ilibv8-darwin -Ilibv8-darwin/include -fno-rtti -fpic -std=c++11",
    ],
    "@io_bazel_rules_go//go/platform:linux": [
        "-I. -Ilibv8-linux/include -fno-rtti -fpic -std=c++11",
    ],
    "//conditions:default": [],
})
```


最终的解决方案是通过将 V8 部分先封装成 `cc_library`, 然后 `go_library` 再将其作为依赖引入, 这样就只需要包含 `cgo` 部分代码用到的头文件包含进来即可.

#结语

可以看到最终的解决方案并不复杂, 但是提供了一个区别与普通 `go_library` 的解决方案,  在 `cgo` 项目里面不一定要一股脑的都塞到 `go_library` 里.
