---
layout: post
title: "Yocto: How to add packages to the SDK?"
thumbnail: assets/images/yocto-logo-bg-dark.svg
categories: [embedded, linux, yocto]
---

Normally a Yocto SDK includes all dependencies that are needed to build everything on your target image.
But how can you add something to your SDK specifically? That might be useful if you want to build
software components that are not included in your final image or you might want to provide developers with
some development tools.

As you might already have realized, the Yocto SDK usually consists of two sysroots (check the top directory of your installed SDK to verify this yourself).
It has a **host** and a **target** sysroot. The host sysroot includes all the libraries and executables that need to run on your SDK host e.g. the cross-compiler and code-generators (i.e. all the `nativesdk` packages). The target sysroot on the other hand includes all the cross-compiled libraries that are needed to build your software for the target.
Therefore there also exist two bitbake variables that specify what is packed into each of the SDK's sysroots. These are `TOOLCHAIN_HOST_TASK` and `TOOLCHAIN_TARGET_TASK`.

Somewhere in your image you probably inherit the `populate_sdk` class. You can then append the packages you want to add to the SDK to those variables. E.g. let's add googletest to the target sysroot and cmake and ninja to the host sysroot:
```
inherit populate_sdk

TOOLCHAIN_TARGET_TASK += "gtest"
TOOLCHAIN_HOST_TASK += "nativesdk-cmake nativesdk-ninja"

```

Happy baking!

## References:
* [Yocto Reference Manual](https://www.yoctoproject.org/docs/latest/ref-manual/ref-manual.html)
