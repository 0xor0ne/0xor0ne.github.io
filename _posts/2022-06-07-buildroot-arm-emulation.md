---
title: Buildroot ARMv7 (32 bits) Cross-compilation and QEMU Emulation and Debugging
post_date: 2022-06-07
layout: default
---

{% comment %}
Optional comments
{% endcomment %}

# Buildroot ARMv7 (32 bits) Cross-compilation and QEMU Emulation and Debugging

In this short tutorial we'll see:

* how to use [Buildroot][1]{:target="blank"} for cross-compiling an image
  (kernel + root file system) targetting ARMv7 (32 bits) architecture;

* how to emulate the resulting image using [QEMU][2]{:target="blank"};

* how to use the `sdk` produced by Buildroot for cross-compiling an executable
  targetting the generated image;

* how to debug a process running on the QEMU emulated system;

We are going to use the [docker-x-builder project][0]{:target="blank"} which
includes scripts for creating a Docker image configured with all the required
dependencies for building Buildroot images.

The rest of the tutorial assumes that Docker is installed and configured to run
without sudo (if not, see [here][3]{:target="blank"} and
[here][4]{:target="blank"}).

## Environment setup

Start by cloning `docker-x-builder`:

```bash
git clone https://github.com/0xor0ne/docker-x-builder.git
cd docker-x-builder
```

and then build the Docker image (this could take a while depending on your
connection speed):

```bash
./scripts/docker_build.sh
```

Next, create a Docker persistent volume (this is where we will save all our
work):

```bash
docker volume create --name armv7-vol
```

Now, for running the container use:

```bash
./scripts/docker_run_inter.sh --volume armv7-vol
```

The option `--volume armv7-vol` mounts the newly created volume in
`~/workspace`.

At this point, inside the container, checkout the desired Buildroot version
(this tutorial has been tested with Buildroot `2022.02.02`):

```bash
cd ~/workspace/buildroot
git checkout 2022.02.2
```

Now the build environment setup is complete and it is possible to proceed with
the Buildroot ARMv7 image creation.

> **NOTE1:** if you need to attach to an already running container from another
> terminal you must use: `./scripts/docker_attach.sh`

> **NOTE2:** the host directory `docker-x-bulder` is shared inside the container
> in `~/shared`.

## ARMv7 Cross-Compilation with Buildroot

Our goal here is to create a Buildroot image that satisfy the following
requirements:

1. The image must include a non root user called `user` with password `user`;

2. The emulated system must allow SSH key-based authentication for both user
   `user` and user `root`;

3. The emulated system must have a network interface with the static IP address
   `192.168.0.2/24`;

4. the final image must target ARMv7 architecture (more specifically a target
   board that we can emulate with QEMU).

We use a custom [user table][5]{:target="blank"} to satisfy requirement `1`, a
custom [overlay][6]{:target="blank"} for requirements `2` and `3` and, finally,
for requirement `4` we will customize the configuration
`qemu_arm_vexpress_defconfig` provided by Buildroot.

Before proceeding, create a directory where we will save the `users table` and
the `overlay`:

```bash
# Run this inside the container
mkdir ~/shared/myarm32
```

### Users Table

`Users tables` are regular text files that allow to create new users using
the [`makeuser syntax`][7]{:target="blank"}.

In order to add the user `user` in the target system, create the file
`~/shared/myarm32/users_table` as shown below:

```bash
# Run this inside the container
echo "user -1 user -1 =user /home/user /bin/sh root User" > ~/shared/myarm32/users_table
```

With the content above, the user `user` will be configured as follow:

* Default password: `user`;
* Home directory: `/home/user`;
* Default shell: `/bin/sh`;
* Groups: `user` and `root`.

Later on, we will reference this `users table` in the Buildroot configuration.

### Overlay

An `overlay` is a tree of files that is copied directly over the target
filesystem after it has been built (it is one of the machanisms provided by
Buildrot to customize the target filesystem).

We are going to use the `overlay` to configure the authorized public SSH keys
for `root` and `user` and to configure the network interface `eth0` with the
static IP address `192.168.0.1`.

First of all, create the rsa key pair with a null passphrase and save it in
`~/shared/myarm32/id_rsa` and `~/shared/myarm32/id_rsa.pub`:

```bash
# Run this inside the container
ssh-keygen -b 2048 -t rsa -f ~/shared/myarm32/id_rsa -q -N ""
```

Then, create the `overlay` directory structure:

```bash
# Run this inside the container
mkdir -p ~/shared/myarm32/overlay/{etc/network,root/.ssh,home/user/.ssh}
```

And, finally, create all the required files:

```bash
# Run this inside the container
cat ~/shared/myarm32/id_rsa.pub > ~/shared/myarm32/overlay/root/.ssh/authorized_keys
cat ~/shared/myarm32/id_rsa.pub > ~/shared/myarm32/overlay/home/user/.ssh/authorized_keys
cat > ~/shared/myarm32/overlay/etc/network/interfaces <<EOF
auto lo
iface lo inet loopback

auto eth0
  iface eth0 inet static
  address 192.168.0.2
  netmask 255.255.255.0
  gateway 192.168.0.1
  hostname \$(hostname)
EOF
```

### Buildroot Configuration

Generate the default configuration for [Arm Versatile Express board][10]
provided by Buildroot:

```bash
# Run this inside the container
cd ~/workspace/buildroot
make qemu_arm_vexpress_defconfig
```

and then run the interactive curses-based configurator:

```bash
# Run this inside the container
make menuconfig
```

In the menu, set the following options:

```bash
Toolchain:
-> Toolchain type
   -> External toolchain
-> Toolchain
   -> Bootlin toolchains
-> Bootlin toolchain variant
   -> armv7-eabihf glibc stable 2021.11-1 [Select]
-> Copy gdb server to the Target [Y]

System configuration:
-> Root password [set 'root']
-> Path to the users table [set ${HOME}/shared/myarm32/users_table]
-> Root filesystem overlay directories [set ${HOME}/shared/myarm32/overlay]

Target packages:
-> Networking applications:
   -> dropbear [Y]
      -> disable reverse DNS lookups [Y]

Host utilities:
-> host environment-setup [Y]
```

When all the options above have been set, exit the menu and save the
configuration.

### Image Build

Once all the steps above have been completed, you can start building the target
image (this is going to take a while to complete):

```bash
# Run this inside the container
cd ~/workspace/buildroot
make
```

The output artifacts will be located in `~/workspace/buildroot/output/images`:

* `zImage`: Linux kernel image;

* `rootfs.ext2`: root filesystem image;

* `vexpress-v2p-ca9.dtb`: device tree blob;

## Emulate the Target System with QEMU

The buildroot image can be emulated with the following command:

```bash
# Run this inside the container
~/workspace/buildroot/output/host/bin/qemu-system-arm -M vexpress-a9 \
  -smp 1 -m 256 -kernel ~/workspace/buildroot/output/images/zImage \
  -dtb ~/workspace/buildroot/output/images/vexpress-v2p-ca9.dtb -drive \
  file=~/workspace/buildroot/output/images/rootfs.ext2,if=sd,format=raw \
  -append "console=ttyAMA0,115200 rootwait root=/dev/mmcblk0" \
  -net nic -net tap,ifname=tap0,script=no,downscript=no -nographic
```

At the end of the boot process you can log into the system as `user` (password
`user`) or as `root` (password `root`). To stop `QEMU`, login as `root` and run
command `poweroff`.

You can also use SSH for log into the emulated system. First of all, attach to
the running container from another terminal:

```bash
./scripts/docker_attach.sh
```

and then use SSH to log into the system:

```bash
# Run this inside the container
ssh -i ~/shared/myarm32/id_rsa -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null user@192.168.0.2
```

NOTE: the container created with `docker-x-builder` is configured with a tap
interface (`tap0`) with static IP address `192.168.0.1`. Thanks to this
interface it is possible to communicate with the emulated system whose network
interface has been previously configured with the static IP address
`192.168.0.2`.

## Cross-compilation with Buildroot-generated SDK

For cross-compiling an application for the target system, it is possible to use
the toolchain used by Buildroot itself which is located in `output/host/bin`.

However, this tutorial shows how to export an `SDK` for the target system. The
SDK is more flexible because it can be moved also to other machines and used for
cross-compiling applications for the target system.

For exporting the `SDK`, execute the following commands:

```bash
# Run this inside the container (./scripts/docker_run_inter.sh --volume armv7-vol)
cd ~/workspace/buildroot
make sdk
```

This generates a tarball named
`arm-buildroot-linux-gnueabihf_sdk-buildroot.tar.gz` in `output/images`.

In order to use the `SDK` you must extract the tarball and run the script
`relocate-sdk.sh` located inside the extracted directory (this script must be
executed only one time after the tarball extraction).

Then, when you need to use the toolchain you can source the file
`environment-setup` located inside the extracted toolchain directory. This
script exports a number of environment variables that will help cross-compile
your projects using the Buildroot SDK.

Here is a complete example showing how to extract and setup the `SDK` and how to
cross-compile a toy executable:

```bash
# Run this inside the container
mkdir ~/workspace/sdk
cp ~/workspace/buildroot/output/images/arm-buildroot-linux-gnueabihf_sdk-buildroot.tar.gz ~/workspace/sdk/
cd ~/workspace/sdk
mkdir armv7 && tar xf arm-buildroot-linux-gnueabihf_sdk-buildroot.tar.gz -C armv7 --strip-components 1
cd armv7 && ./relocate-sdk.sh
```

At this point it is possible to cross-compile code for the target system like
illustrated below:

```bash
# Run this inside the container
mkdir ~/workspace/hello-example

cat > ~/workspace/hello-example/hello.c <<EOF
#include <stdio.h>

int main()
{
  printf("Hello!\n");
  return 0;
}
EOF

source ~/workspace/sdk/armv7/environment-setup
arm-linux-gcc -g -o ~/workspace/hello-example/hello ~/workspace/hello-example/hello.c
```

The cross-compiled executable will be located in
`~/workspace/hello-example/hello`.

```bash
>>> file ~/workspace/hello-example/hello
/home/user/workspace/hello-example/hello: ELF 32-bit LSB pie executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, for GNU/Linux 4.9.0, with debug_info, not stripped
```

## Remote Process Debugging in Emulated System

In one terminal run the emulator as shown in section
[Emulate the Target System with QEMU](#emulate-the-target-system-with-qemu).

In a second terminal, attach to the running container:

```bash
./scripts/docker_attach.sh
```

and copy on the emulated system the executable built in section [Cross-compilation with Buildroot-generated
SDK](#cross-compilation-with-buildroot-generated-sdk):

```bash
scp -i ~/shared/myarm32/id_rsa -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ~/workspace/hello-example/hello user@192.168.0.2:
```

On the first terminal, where you have the `QEMU` prompt of the emulated device, login as
user `user` (password `user`) and run:

```bash
gdbserver :1234 hello
```

On the second terminal you can start the debugging session like in the example
shown below:

```bash
source ~/workspace/sdk/armv7/environment-setup
arm-linux-gdb --se=~/workspace/hello-example/hello -ex 'target remote 192.168.0.2:1234'
(gdb) disassemble main
(gdb) b *main+20
(gdb) continue
(gdb) x/s $r0
...
# Debug with standard GDB commands
```




NOTE: the same target image could also be emulated and debugged with the Docker
image provided by the [doc-revexp project][8] that has the advantage of
including more tools for debugging and reverse engineering activities like, for
example, gdb preconfigured with [GEF][9] plugin.

\- The End -

[0]: https://github.com/0xor0ne/docker-x-builder
[1]: https://buildroot.org
[2]: https://www.qemu.org/
[3]: https://docs.docker.com/engine/install/
[4]: https://docs.docker.com/engine/install/linux-postinstall/
[5]: https://buildroot.org/downloads/manual/manual.html#customize-users
[6]: https://buildroot.org/downloads/manual/manual.html#rootfs-custom
[7]: https://buildroot.org/downloads/manual/manual.html#makeuser-syntax
[8]: https://github.com/0xor0ne/doc-revexp
[9]: https://github.com/hugsy/gef
[10]: https://qemu-project.gitlab.io/qemu/system/arm/vexpress.html
