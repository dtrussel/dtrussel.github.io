---
layout: post
title: "Yocto: Recipe flavors"
thumbnail: assets/images/yocto-logo-bg-dark.svg
categories: [embedded, linux, yocto]
---

If you have done some Yocto development you might already have encounter them in the wild...
`native` and `nativesdk` recipes...
Recipes cannot only be built for the target, but also for your build host or your SDK host.
This post gives a short summary about what the different recipe "flavors" are used for
and how to add them to your recipes.

![spices by Andra Ion](/assets/images/flavors.jpg){: .center-image }

# The holy trinity

1. **foo**
1. **foo-native**
1. **nativesdk-foo**

The most common case is just building you recipe `foo`. This builds the recipe for your target architecture e.g. `aarch64`.

But you also might need to build your recipe in the native flavor i.e. `foo-native`. This builds the recipe such that it can be used on the build host e.g. `x86_64`.
Why would you need that? Let's say you want to build a recipe that, as part of its build, needs to generate some code. This code is generated with python.
Now your recipe needs to add `DEPENDS += "python-native"` because you want to run this code generation as part of the build process on your host and not on the target machine. Adding a `DEPENDS += "python"` would not make sense, since that would be the cross compiled version of python, which cannot run on your build host.

What about `nativesdk-foo`?
Now there is another use case. Assume you want to build the same project mentioned above, but this time not within your Yocto project, but with the SDK which distributed to the application developers. So the SDK should include python as well, but again not the cross-compiled version but a version that can run on the host where the SDK is installed. In 99% that is probably the same architecture as the build host (e.g. `x86_64`) but theoretically the two could be different.
Hence we need to add `nativesdk-python` to our SDK.

# Add native and nativesdk support to your recipes
Let's consider the most simple case: Your recipe is build the same way for all architectures.
Then it is enough to just add
```
BBCLASSEXTEND = "native nativesdk"
```
to your recipe.

And if the package is built differently for each architecture then you need to write the additional recipes yourself i.e.
add a `foo-native.bb` and a `nativesdk-foo.bb` to your layer.

## References:
* [Yocto Reference Manual](https://www.yoctoproject.org/docs/latest/ref-manual/ref-manual.html)
