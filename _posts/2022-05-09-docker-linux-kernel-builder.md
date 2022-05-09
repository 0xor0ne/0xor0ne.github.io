---
title: Docker Linux Kernel Builder
post_date: 2022-05-09
layout: default
---

{% comment %}
Optional comments
{% endcomment %}

# Linux Kernel Cross-compilation

This short post illustrates how to cross-compile the Linux kernel using
[docker-linux-kernel-builder][0]{:target="blank"}.

## Setup

Assuming Docker is installed on your machine and configured to run without sudo
(if not, see [here][1]{:target="blank"} and [here][2]{:target="blank"}), proceed
by cloning the project repository:

```bash
git clone https://github.com/0xor0ne/docker-linux-kernel-builder.git
cd docker-linux-kernel-build
```

then edit `config.env` and set the following variables:

* LK_DOWNLOAD_LINK: Linux kernel download URL;
* TC_DOWNLOAD_LINK: toolchain download URL;
* LK_ARCH: target architecture (must match the toolchain).

For this example we are going to build Linux kernel `5.15.37` for `arm64`
(`aarch64`) by setting the above variables with the following values:

```bash
LK_ARCH="arm64"
LK_DOWNLOAD_LINK=https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.37.tar.xz
TC_DOWNLOAD_LINK=https://toolchains.bootlin.com/downloads/releases/toolchains/aarch64/tarballs/aarch64--glibc--stable-2021.11-1.tar.bz2
```

At this point, download the kernel source and the toolchain by running the
following script:

```bash
./scripts/download_lk_tc.sh
```

For additional details and for using customized kernel source code see the
official [project README][3]{:target="blank"}.

## Docker Image

Build the docker image:

```bash
./scripts/docker_create_volume.sh && ./scripts/docker_build.sh
```

## Kernel Configuration

Use the default kernel configuration for the target architecture with:

```bash
./scripts/make.sh defconfig
```

For instructions on how to use a customized kernel configuration see the [project
README][3]{:target="blank"}.

## Kernel Build

Finally, build the kernel with (this might take a while):

```bash
./scripts/make_all_install_retrieve.sh
```

the output artifacts will be placed in `shared/install`.

```bash
>>> ls shared/install
config-5.15.37  include  lib  System.map-5.15.37  vmlinuz-5.15.37
```

\- The End -

[0]: https://github.com/0xor0ne/docker-linux-kernel-builder
[1]: https://docs.docker.com/engine/install/
[2]: https://docs.docker.com/engine/install/linux-postinstall/
[3]: https://github.com/0xor0ne/docker-linux-kernel-builder/blob/main/README.md
