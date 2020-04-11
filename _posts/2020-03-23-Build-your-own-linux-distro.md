---
layout: post
title: Build your own linux distribution
thumbnail: assets/images/yocto-logo-bg-dark.svg
categories: [embedded, linux, yocto]
---

In this post we are going to create our own yocto layer.
The [Yocto Project](https://www.yoctoproject.org/) is an open source project that lets you 
create your own embedded linux distribution. You create **recipes** that are bundled
into *layers* (which are usually called `meta-something`).
The recipes consist themselves of tasks (`do_compile`, `do_install`...) and let you
specify dependencies between tasks. These recipes are then **baked** into an image
with yocto's build system called `bitbake`.
The generated outputs of a recipe are called **packages** (one recipe can provide several packages).

![Yocto](/assets/images/yocto-logo-bg-dark.svg){: .center-image }

Now you are probably thinking: Why would I want to build yet another linux distro?
I can just say that in my case, we did not want to rely on an external distribution
provider for the embedded device we were developing. We wanted to have full control
over what software will run on the device and we did not want to deal with external
distribution support or updates.

While the projects documentation is excellent, the learning curve is quite steep.
In the begining it can be hard to find a good starting point. So instead of
wasting any time, let us just jump into creating an own distribution. All you need
is a Debian/Ubuntu host with at least 50 GBytes free disk space.

But be warned. Yocto builds can take a long time when run for the first time.

![xkcd advanced technology](https://imgs.xkcd.com/comics/advanced_technology.png){: .center-image }

# Let's start!

Open a terminal and switch to the directory where you would like to start the
project:
```
sudo apt-get install gawk wget git-core diffstat \
  unzip texinfo gcc-multilib build-essential chrpath socat
mkdir yocto && cd yocto
git clone git://git.yoctoproject.org/poky -b zeus
source poky/oe-init-build-env build
bitbake-layers create-layer ../meta-foundation --priority 10
bitbake-layers add-layer ../meta-foundation
cd ..
```
As first step we installed the dependencies, created a project directory and cloned
the poky (Yocto's example linux distribution) into it. We then sourced the yocto
build environment (which also creates also a build directory with the provided name if it does not exist yet) and created our own layer with the `bitbake-layers` command. 

# The project's structure

So let us have a look at the whole project and how it is organized:
```
yocto
├── build
│   ├── bitbake-cookerdaemon.log
│   ├── cache
│   ├── conf
│   └── tmp
├── meta-foundation
│   ├── conf
│   ├── COPYING.MIT
│   ├── README
│   └── recipes-example
└── poky
    ├── bitbake
    ├── contrib
    ├── documentation
    ├── LICENSE
    ├── LICENSE.GPL-2.0-only
    ├── LICENSE.MIT
    ├── meta
    ├── meta-poky
    ├── meta-selftest
    ├── meta-skeleton
    ├── meta-yocto-bsp
    ├── oe-init-build-env
    ├── README.hardware -> meta-yocto-bsp/README.hardware
    ├── README.OE-Core
    ├── README.poky -> meta-poky/README.poky
    ├── README.qemu
    └── scripts

```
We have the poky layer which provides us with the build system `bitbake`, a script `oe-init-build-env`to initialze the build environmet and the core layers (all subdirectories starting with `meta`). On the same level we have our newly created meta-foundation layer and the build directory, where all bitbake keeps all the build output and cache as well as some local
build configuration (e.g. how many threads to use for bitbake and make etc.).

Now let's have a closer look at the structure of our created layer `meta-foundation`:
```
.
├── conf
│   └── layer.conf
├── COPYING.MIT
├── README
└── recipes-example
    └── example
        └── example_0.1.bb
```

The `conf/layer.conf` tells bitbake how we organize our layer and e.g. with which
versions it is compatible with:
```
# We have a conf and classes directory, add to BBPATH
BBPATH .= ":${LAYERDIR}"

# We have recipes-* directories, add to BBFILES
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "meta-foundation"
BBFILE_PATTERN_meta-foundation = "^${LAYERDIR}/"
BBFILE_PRIORITY_meta-foundation = "10"

LAYERDEPENDS_meta-foundation = "core"
LAYERSERIES_COMPAT_meta-foundation = "warrior zeus"
```

With the bitbake-layers command we also created an example recipe `recipes-example/example/example_0.1.bb`:
```
SUMMARY = "bitbake-layers recipe"
DESCRIPTION = "Recipe created by bitbake-layers"
LICENSE = "MIT"

python do_build() {
    bb.plain("***********************************************");
    bb.plain("*                                             *");
    bb.plain("*  Example recipe created by bitbake-layers   *");
    bb.plain("*                                             *");
    bb.plain("***********************************************");
}

```
This only prints something during building. Let us changes this to do
something more meaningful. Often you want to include some library / application
to your OS. Since `CMake` is quite popular and I often use it for my projects, we
will add a recipe that builds a cmake project. 

# Add an own recipe

Now we will rename our recipe to the name of the library which we want to add
to our layer:
```
cd meta-foundation
mv recipes-example recipes-support
mv recipes-support/example recipes-support/dtr
mv recipes-support/dtr/example_0.1.bb recipes-support/dtr/dtr_git.bb
```

Overwrite our example recipe with one that builds a Cmake based library:
```
cat > recipes-support/dtr/dtr_git.bb <<'__EOF__'
SUMMARY = "dtr - C++ utility library"
LICENSE = "MIT"
LIC_FILES_CHKSUM = "\
  file://${COMMON_LICENSE_DIR}/MIT;md5=0835ade698e0bcf8506ecda2f7b4f302"

SRC_URI = "git://github.com/dtrussel/dtr.git"
SRCREV = "866c777907e096f9d88d01cf104984906afc6425"

S = "${WORKDIR}/git"

inherit cmake

FILES_${PN}-dev = "${includedir}"

DEPENDS = "boost"
__EOF__
```
I am not going into detail how to write recipes, but basically we just told bitbake
where to find the source by setting `SRC_URI` and `SRCREV`. With `inherit cmake` we
included the default cmake recipe tasks. In `DEPENDS` we can set the dependency of
this recipe on other recipes (here on the boost library which is provided by one of
the layers in the poky repo).

# The Distro

Since we want our own distribution we add a config file for our distro:
```
mkdir conf/distro
cat > conf/distro/foundation.conf <<'__EOF__'
require conf/distro/poky.conf

DISTRO = "foundation"
DISTRO_NAME = "Foundation (Linux Distribution)"
DISTRO_VERSION = "2020.1"
DISTRO_CODENAME = "asimov"
SDK_VENDOR = "-foundation"
SDK_VERSION = "${DISTRO_VERSION}"
SDK_NAME = "${DISTRO}-${DISTRO_VERSION}-${TUNE_PKGARCH}-${MACHINE}"
SDKPATH = "/opt/${DISTRO}/${SDK_VERSION}"

DISTRO_VERSION[vardepsexclude] = "DATE"
SDK_VERSION[vardepsexclude] = "DATE"
SDK_NAME[vardepsexclude] = "DATE"
__EOF__
```

# The Image

A distro can have several images (e.g. base, server, development, production),
which contain more or less packages.
So let's add a image that is based on the `core-image-base` and add our recipe
to it.
```
mkdir -p recipes-core/images/
cat > recipes-core/images/foundation-image-base.bb <<'__EOF__'
include recipes-core/images/core-image-base.bb

IMAGE_INSTALL_append = " dtr-dev"
__EOF__
```
Sidenote: The package we add is `dtr-dev`, and not `dtr` because it is a header-only library
and the main package of the cmake based recipe is just the test executable in this
case.

So now we are ready and can finally **bake it**:
```
DISTRO=foundation bitbake foundation-image-base
```

Great we are building our own linux distribution! Go grab a coffee while the initial
build will take a long time, since it builds everything from source. But don't worry, subsequent
builds will be incremental.

## Some tips:
- Only put your **build configuration** into `build/conf/local.conf` (and NOT your distro, image or machine configuration).
- Never modify another layer (if you want to modify an existing recipe, use a `recipes-something/somelibrary_<VERSION>.bbappend` file in your layer instead)

## References:
* [Yocto Project Overview](https://www.yoctoproject.org/docs/latest/overview-manual/overview-manual.html)
* [Yocto Reference Manual](https://www.yoctoproject.org/docs/latest/ref-manual/ref-manual.html)
* [Bitbacke User Manual](https://www.yoctoproject.org/docs/latest/bitbake-user-manual/bitbake-user-manual.html)





