---
title: Building rust toolchain for armv7-unknown-linux-uclibceabi
---

So I decide to actually start writing some stuffs. As the first
real post in this blog, this one documents some of my experiences when
trying to build a rust cross-compile toolchain for `armv7-unknown-linux-uclibceabi`.
**Although in the end it isn't quite work as expected and can be considered failed**,
the process is useful for me to note down.

This is a preparation for the upcoming series of posts which are about my learning of
[shadowsocks](https://github.com/shadowsocks/shadowsocks-rust) and applying it in
a custom setup on my Netgear R7000 router. For this post let's focus on the toolchain itself.

Before the real content I'd like to state that I'm just a beginner level rustacean and have
no much experience in embedded systems so errors/mis-understandings are unavoidable.

Generally what I'd like to achieve is to be able to build a rust project on my
computer (desktop, laptop, etc) and being able to just copy the binaries
to my embedded devices (my smartphone or router and so on). Specifically and in the
sense of the jargon of "cross compiling" my primary _host_ is `x86_64-unknown-linux-gnu`,
and my _target_ is `armv7-unknown-linux-uclibceabi`. But I also want to learn the
general way so I can do similar things in other situations in the future. That's why
I want to learn how to build a toolchain instead of finding one someone else provides
(Although I yet to find an existing working one for my target).

# Determine the Target

First thing I ask: What if I know nothing about my target? How do I know it is
`arm-something-linux-something`? The most obvious way is to ssh into my box,
then `cat /proc/cpuinfo` to get a general idea of what the cpu is.

For my router the output looks like this:

```c
Processor       : ARMv7 Processor rev 0 (v7l)
processor       : 0
BogoMIPS        : 1998.84

processor       : 1
BogoMIPS        : 1998.84

Features        : swp half thumb fastmult edsp
...
```

So I know for sure it is an ARM v7 architecture. So we got the `arch` part,
what about the remaining parts (vendor, system, abi)? For rust we can filter
the target for `arm` and see what is available.

```plain
arm-linux-androideabi
arm-unknown-linux-gnueabi
arm-unknown-linux-gnueabihf
arm-unknown-linux-musleabi
arm-unknown-linux-musleabihf
armebv7r-none-eabi
armebv7r-none-eabihf
armv4t-unknown-linux-gnueabi
armv5te-unknown-linux-gnueabi
armv5te-unknown-linux-musleabi
armv5te-unknown-linux-uclibceabi
armv6-unknown-freebsd
armv6-unknown-netbsd-eabihf
armv6k-nintendo-3ds
armv7-apple-ios
armv7-linux-androideabi
armv7-unknown-freebsd
armv7-unknown-linux-gnueabi
armv7-unknown-linux-gnueabihf
armv7-unknown-linux-musleabi
armv7-unknown-linux-musleabihf
...
```

That's a total of 32 targets as of writing. But since we know its _armv7_
we can probably restrict the possibilities to the ones starting with `armv7`.

```plain
armv7-apple-ios
armv7-linux-androideabi
armv7-unknown-freebsd
armv7-unknown-linux-gnueabi
armv7-unknown-linux-gnueabihf
armv7-unknown-linux-musleabi
armv7-unknown-linux-musleabihf
armv7-unknown-linux-uclibceabi
armv7-unknown-linux-uclibceabihf
armv7-unknown-netbsd-eabihf
armv7-wrs-vxworks-eabihf
armv7a-kmc-solid_asp3-eabi
armv7a-kmc-solid_asp3-eabihf
armv7a-none-eabi
armv7a-none-eabihf
armv7r-none-eabi
armv7r-none-eabihf
armv7s-apple-ios
```

We want the most specific ones (so we can have more optimized code?) For my
router it does have a _Linux_ system (Asuswrt-Merlin firmware, version 380) so it seems we are left with the ones
starting with `armv7-unknown-linux-xxx` ones where xxx stands for one of
`gnueabi`, `musleabi` and `uclibceabi`. Its not the `hf` kind because from
the output of `cpuinfo` I see no sign of `fpu`. So which libc version is it using?
Actually I know by googling it is `uclibc`. If I don't know this,
I would invoke the libc binary to check. Here is the output for
**glibc** and **musl** (shared lib) on my desktop:

```sh
❯ /usr/lib/libc.so.6 
GNU C Library (GNU libc) stable release version 2.35.
Copyright (C) 2022 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.
Compiled by GNU CC version 12.1.0.
libc ABIs: UNIQUE IFUNC ABSOLUTE
For bug reporting instructions, please see:
<https://bugs.archlinux.org/>.

❯ /usr/lib/musl/lib/libc.so
musl libc (x86_64)
Version 1.2.3
Dynamic Program Loader
Usage: /usr/lib/musl/lib/libc.so [options] [--] pathname [args]
```

Not really useful for uclibc! Here is what I see on my router:
```sh
admin@R7000:/tmp/home/root#  /lib/libc.so.0
stack smashing detected:  terminated()
Segmentation fault
```

I also found this [stackoverflow question](https://stackoverflow.com/questions/50443164/how-to-detect-libc-name-and-version-in-cross-compiler-environments_ but the accepted answer doesn't really
work in our case. We have no _gcc_ and `ldd --version` outputs nothing useful.

I do figured out two ways to detect the presense of uclibc, the first being
using `ldd` on an executable and see if it helps:

```sh
admin@R7000:/tmp/home/root# ldd ls
        libcrypt.so.0 => /lib/libcrypt.so.0 (0x401ab000)
        libm.so.0 => /lib/libm.so.0 (0x40113000)
        libptcsrv.so => /usr/lib/libptcsrv.so (0x4016f000)
        libgcc_s.so.1 => /lib/libgcc_s.so.1 (0x40178000)
        libc.so.0 => /lib/libc.so.0 (0x401cf000)
        ld-uClibc.so.0 => /lib/ld-uClibc.so.0 (0x400fd000)
```
For dynamic linked binary we can see the name from the last line.

The second way is to just grep the `libc.so` file:

```sh
admin@R7000:/tmp/home/root# grep -i uclibc /lib/libc.so.0
__uclibc_progname
__GI___uClibc_fini
__GI___uClibc_init
__uClibc_main
ld-uClibc.so.0
/lib/ld-uClibc.so.0
```

But as you can imagine this only works if we suspect it is uclibc in the first place.

# Build using cross project

For cross compiling common targets [cross](https://github.com/cross-rs/cross) is
a fantastic tools. I tried it for android and some other targets and
its working great.

However as of now it doesn't support the target
I want. I think in the future if I'm lazy and just want
to build and run, I would first try with cross to see if it supports
the target.

# Build a rust toolchain for real

Googling for this there are notably 2 posts talking about this.

1. [Everything you need to know about cross compiling Rust programs!](https://rustrepo.com/repo/japaric-rust-cross-rust-embedded)
2. [armv7-unknown-linux-uclibceabi](https://doc.rust-lang.org/rustc/platform-support/armv7-unknown-linux-uclibceabi.html)

The first one is a generic one, I strongly recommend reading it if you want
to do similar stuffs or want to learn about this. The `Advanced topics` covers
how to build a rust toolchain ourself. But I do need to make some adjustments
to make the whole process works.

The second one talks about our target specifically, but to be honest it is kind
of confusing to me given I'm at the beginner level. It seems to me the final output
of the linked repo is a "native compilation" toolchain instead of a "cross compilation" one.
But I do learned a lot after reading it several times.


# C toolchain

From what I read to build a rust toolchain you first need to acquire a C
toolchain corresponding to your platform and use it to compile rust itself
and produce "stage2" compiler. Then you link it with a name so you can use
it to build your program for the target platform. Note that this is in the sense
that you want to use rust's `std` library instead of bare-metal kind known as `no_std`,
in which case should be a lot easier but I have no experience in that.

For the C toolchain [buildroot](https://buildroot.org/) is a convenient tool.
My initial try with it resulted in a seems to be working toolchain with which I
can compile a "helloworld" C program which runs successfully in my router. However
my initial "product" didn't contain a C++ compiler so that is actually not used to
compile my rust toolchain.

Instead I tried first with an existing "tomatoware" toolchain mentioned in the [platform support](https://doc.rust-lang.org/rustc/platform-support/armv7-unknown-linux-uclibceabi.html)
article. For recent rust versions I do need to combine the info given on that page
with its sibling page (which is for `uclibchf`) to actually got a "working" rust toolchain with `std`.

# Build the rust toolchain (std)

The process can be summarize below (with my naive understanding):

1. confirm the C toolchain works by actually compile something and run
2. `rustc -vV` to get current rust's commit hash and checkout into that
(I'm not sure if this is really required though, as I know almost nothing about rustc codebase)
3. modify config.toml inside repo root, make it something like this:

BTW the official [dev guide](https://rustc-dev-guide.rust-lang.org/building/how-to-build-and-run.html) is super helpful.

```toml
[build]
build-stage = 2
target = ["armv7-unknown-linux-uclibceabi"]

[target.armv7-unknown-linux-uclibceabi]
cc = "/path/to/arm-linux-gcc"
cxx = "/path/to/arm-linux-g++"
ar = "/path/to/arm-linux-ar"
ranlib = "/path/to/arm-linux-ranlib"
linker = "/path/to/arm-linux-gcc"
```
4. make sure ninja-build and other deps are installed
5. `./x.py build --stage 2` inside repo root
6. wait for a long time

If it finishes successfully (otherwise I would have no idea how to proceed except errors
like missing some command) you should have something in `${RUST}/build/${host}/stage2`, where `${RUST}`
is the repo root path and `${host}` is the host architecture (not the target). For my purpose
I am looking for the existence of `build/x86_64-unknown-linux-gnu/stage2/lib/rustlib/armv7-unknown-linux-uclibceabi`.

With this done we can link with `cargo toolchain link ${name} ${RUST}/build/x86_64-unknown-linux-gnu/stage2`
where `${name}` seems can be whatever you want (valid identifier only?). Finally when compiling we can use
command like this to compile:

```sh
cargo +${name} rustc --target armv7-unknown-linux-uclibceabi
```

The `${name}` is what you specified above. Note that for binary you need to also specify the linker to use.
Environment variable is possible:

```sh
CARGO_TARGET_ARMV7_UNKNOWN_LINUX_UCLIBCEABI_LINKER=${TOOLCHAIN}/bin/arm-buildroot-linux-uclibceabi-gcc
```

Also obviously `.cargo/config` can be used instead.


# Does it work?

Nooo. Not really.

With my first compiled rust toolchain I can compile a simple "helloworld" in rust. But when I deploy it to
my router it fails to run with error "**Program uses unsupported TLS data!**". A quick search
we can know it comes from [here](https://github.com/m-labs/uclibc-lm32/blob/defb191aab7711218035a506ec5cd8bb87c05c55/ldso/ldso/ldso.c#L711).
This is about thread local storage and where I failed. Admittedly I don't know anything about it.
([This](https://fasterthanli.me/series/making-our-own-executable-packer/part-13) and [that](https://maskray.me/blog/2021-02-14-all-about-thread-local-storage)
seems to be worth reading)

One thing I do know is to inspect the binary:
```sh
❯ llvm-readelf -S <file>
```

```sh
...
  [17] .ARM.exidx        ARM_EXIDX       00042e94 042e94 001080 00  AL 12   0  4
  [18] .eh_frame         PROGBITS        00043f14 043f14 000004 00   A  0   0  4
  [19] .tdata            PROGBITS        000545a0 0445a0 000014 00 WAT  0   0  4
  [20] .tbss             NOBITS          000545b4 0445b4 000015 00 WAT  0   0  4
  [21] .init_array       INIT_ARRAY      000545b4 0445b4 000004 00  WA  0   0  4
  [22] .fini_array       FINI_ARRAY      000545b8 0445b8 000004 00  WA  0   0  4
...
```
and from the result we can see there are `.tdata` and `.tbss` sections with `WAT` flag.
"T" stands for "TLS" which indicates the usage of thread-local-storage.

I tried to build an android target and the resulting binary does not have something
like this. Maybe the trick is to understand who determines there needs to be a `TLS`
section. I'm guessing it's the C toolchain that's in charge here and if I use the
correct toolchain for my router's system it may works? But like I said
I really have not much knowledge in this area for now.

The second toolchain I tried is from [here](https://github.com/gnuton/Asuswrt-Merlin-Toolchains-Docker) but the result is the same.
It is for a newer merlin version and underlying uclibc doesn't seems to be the same with my
router's (Yeah my router was using an old firmware version and the hardware isn't
officially supported for the new version).

# Conclusion

So in the end I have a rust toolchain that can compiles for my target but can not really
run for my specific device. I failed to find an existing toolchain that corresponds
to my router's system, neither did I managed to compile one myself from the source (
I do seems to find the [source](https://github.com/RMerl/asuswrt-merlin) but I was
becoming lazy and do not want to setup the environment for it to build for the time being).

In the future I may revisited this topic but for now I'd like to just use musl to compile a static binary for my purpose.

