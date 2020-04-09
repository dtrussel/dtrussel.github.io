---
layout: post
title:  "Yocto: Switch to systemd"
thumbnail: assets/images/systemd-dark.svg
categories: [embedded, linux, yocto, systemd]
---
Yocto's reference distribution [poky](VIRTUAL-RUNTIME_initscripts = "") comes
with [SysVinit](https://en.wikipedia.org/wiki/Init#SysV-style) as an initalization manager.
However many major linux distributions use [systemd](https://systemd.io/)
as a system and service manager. In this post we will look how to easily switch
your yocto distro to systemd.

![Yocto](/assets/images/systemd-dark.svg){: .center-image }

Systemd has been a quite controvorsial replacement for SysVinit and I am not
going to discuss the pros and cons of them here. This has already been done enough.
However I am used to systemd from working on Debian, Ubuntu and ArchLinux.
Therefore I also wanted my distribution to use systemd.
It turns out that this is actually quite easy. (Only be aware that systemd
will be bigger in size than SysVinit, although it can be customized for embedded
projects).

In your distribution config file `conf/distro/<distroname>.conf` add the following
lines. E.g. my `meta-foundation/conf/distro/foundation.conf`:
```
DISTRO_FEATURES_append = " systemd"
DISTRO_FEATURES_BACKFILL_CONSIDERED = "sysvinit"
VIRTUAL-RUNTIME_init_manager = "systemd"
VIRTUAL-RUNTIME_initscripts = ""
```
And that's it. We are done. (Note: the space in `DISTRO_FEATURES_append = " systemd"`
is required!)

With `DISTRO_FEATURES_append = " systemd"` and `VIRTUAL-RUNTIME_init_manager = "systemd"`
we added systemd and told bitbake to use it as the initialization manager.

With `DISTRO_FEATURES_BACKFILL_CONSIDERED = "sysvinit"` and `VIRTUAL-RUNTIME_initscripts = ""`
we completly removed all SysVinit dependencies in our image. If you do not specify
these you can still can use SysVinit for your rescue/minimal image.

`DISTRO_FEATURES_BACKFILL_CONSIDERED` lists features which should not be used for
(feature backfilling)[https://www.yoctoproject.org/docs/latest/ref-manual/ref-manual.html#ref-features-backfill].

