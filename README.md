# Scripts for Building a Linux/Wasm Operating System
This project contains scripts to download, build and run a Linux system that can be executed on the web, using native WebAssembly (Wasm).

These scripts can be run in the following way:
* Directly on a host machine.
* In a generic docker container.
* In a specific docker container (see Dockerfile).

## History
The initial release of this Wasm port was made by me, Joel. After the release I got into contact with Tom, who had also been working on an independent Wasm port, even during the same time frame as me. You should check out Tom's port at https://github.com/aripitek/tombl/linux . We have both arrived at very similar solutions, sometimes even eerily similar, but there are also some important differences in detail. Both ports were based on early 6.x-era kernels and I've since worked on trying to pick the best parts out of both and rebase on Linux 7.0. This is still work in progress but it has the basics and is now the default build target of this repo. Important parts that are in Tom's port that have not been added yet are virtio and device tree support. However, the 7.0 release does see 64-bit kernel support (the user space part for this is still quite experimental, the kernel part seems to work quite well). On top of this, the 7.0 release sees quite a few bug fixes and can now build allnoconfig and allyesconfig. All kunit tests also pass for both wasm32 and wasm64 (you may see a few spurious errors for tests that completely hog the CPU in kernel mode and timeout).

## Future
Apart from bug fixes and improvements, this is in the pipeline:
* MMU support
* ELF support
* Driver support (virtio, WASI, ...)
* Better runtime (replace the current JS hack, use more Wasm for speed)
* Yocto support (probably needs some of the above first)
* Suspending execution in the kernel (actually rarely needed but for correctness)

## Parts
The project is built and assembled from the following pieces of software:
* LLVM Project:
    * Base version: 18.1.2
    * Patches:
        * A hack patch that enables GNU ld-style linker scripts in wasm-ld.
        * Various fixes for Wasm that makes it more compatible with normal ELF programs.
    * Artifacts: clang, wasm-ld (from lld), compiler-rt
* LLVM-Wasm compatibility wrapper:
    * Notes:
        * A thin Python wrapper that translates a compiler command line that originally targeted an ELF build to something more suitable for Wasm.
        * This was not part of the initial Linux 6.4-era release, but keeps things cleaner and more generic and is needed for 7.0-era patches.
    * Artifacts: fake-llvm Python wrapper.
    * Dependencies: clang, wasm-ld.
* Linux kernel:
    * Base version: 7.0 (original release was based on 6.4.16)
    * Patches:
        * A patch for adding Wasm architecture support to the kernel.
        * A wasm binfmt feature patch, enabling .wasm files to run as executables.
        * A console driver for a Wasm "web console".
        * Support for catching panic() calls in an arch-specific way, improving the debugging experience on the host.
        * Various bux fixes to the base kernel (unrelated to Wasm).
    * Artifacts: vmlinux, exported (unmodified) kernel headers
    * Dependencies: clang, wasm-ld with linker script support, (compiler-rt is *not* needed)
* musl:
    * Base version: 1.2.5
    * Patches:
        * A hack patch (minimal and incorrect) that:
            * Adds Wasm as a target to musl (I guessed and cheated a lot on this one).
            * Allows musl to be built using clang and wasm-ld (linker script support may be needed).
    * Artifacts: musl libc
    * Dependencies: clang, wasm-ld, compiler-rt
* Linux kernel headers for BusyBox
    * Base version: from the kernel
    * Patches:
        * A series of patches, originally hosted by Sabotage Linux, but modified to suit a newer kernel. These patches allow BusyBox to include kernel headers (which is not really supported by Linux). This magically just "works" with glibc but needs modding for musl.
    * Artifacts: modified kernel headers
    * Dependencies: exported Linux kernel headers
* BusyBox:
    * Base version: 1.36.1
    * Patches:
        * A hack patch (minimal and incomplete) that:
            * Allows BusyBox to be built using clang and wasm-ld (linker script support might be unnecessary).
            * Adds a Wasm defconfig.
    * Artifacts: BusyBox installation (base binary and symlinks for ls, cat, mv etc.)
    * Dependencies: musl libc, modified headers for BusyBox
* A minimal initramfs:
    * Notes:
        * Packages up the busybox installation into a compressed cpio archive.
        * It sets up a pty for you (for proper signal/session/job management) and drops you into a shell.
    * Artifacts: initramfs.cpio.gz
    * Dependencies: BusyBox installation
* A runtime:
    * Notes:
        * Some example code of how a minimal JavaScript Wasm host could look like.
        * Error handling is not very graceful, more geared towards debugging than user experience.
        * This is the glue code that kicks everything off, spawns web workers, creates Wasm instances etc.

Hint: MVP Wasm lacks an MMU (edit: now available as an early proposal), meaning that Linux needs to be built in a NOMMU configuration. Wasm programs thus need to be built using -fPIC/-shared. Alternatively, existing Wasm programs can run together with a proxy that does syscalls towards the kernel. In such a case, each thread that wishes to independently execute syscalls should map to a thread in the proxy. The drawback of such an approach is that memory cannot be mapped and shared between processes. However, from a memory protection standpoint, this property could also be beneficial.

## Running
Run ./linux-wasm.sh to see usage. Downloads happen first, building afterwards. You may partially select what to download or (re)-build.

Due to a bug in LLVM's build system, building LLVM a second time fails when building runtimes (complaining that clang fails to build a simple test program). A workaround is to build it yet again (it works each other time, i.e. the 1st, 3rd, 5th etc. time).

Due to limitations in the Linux kernel's build system, the absolute path of the cross compiler (install path of LLVM) cannot contain spaces. Since LLVM is built by linux-wasm.sh, it more or less means its workspace directory (or at least install directory) has to be in a space-free path.

### Options
linux-wasm.sh takes some environment variables that affect its inner workings. Refer to the source for a full list. You do not need to set any of these, **they will default to sane defaults**.
```bash
# A path (relative or absolute) to where everything ends up. The default is a directory workspace/ in the repo root. Not used directly but derives LW_SRC (downloads), LW_BUILD (build files) and LW_INSTALL (final artifacts). For maximum flexibility, LW_SRC/LW_BUILD/LW_INSTALL can be set directly instead.
# Examples: my_workspace, workspaces/my_workspace, /absolute/path
LW_WORKSPACE=

# Currently supports wasm32_nommu and wasm64_nommu.
LW_VARIANT=

# One of:
#   (empty) - Default: build the Wasm-default defconfig.
#   rebuild - Re-create the Wasm-default defconfig (for developers).
#   dev - Development config (WERROR=y is added).
#   yes - allyesconfig (every kernel feature enabled, WERROR=y already default).
#   no - allnoconfig (very minimal build, WERROR=y is added for convenience).
#   ubsan - Find undefined behavior in kernel code.
#   kunit - Build and run KUnit tests at boot.
LW_KERNEL_CONFIG=

# If set to exactly 1, open menuconfig. NOTE: linux-wasm.sh always overwrites the config according to LW_KERNEL_CONFIG first, on every run. Thus, any changes must be saved before re-running. See how LW_KERNEL_CONFIG=rebuild works to see how you may want to save it. You may want to add special case if you really want to work with a fragile full .config file. Any real changes should modify base.config or $LW_ARCH.config instead to allow the automatic generation of defconfigs.
LW_KERNEL_MENUCONFIG=

# Build parallelism:
LW_JOBS_LLVM_LINK
LW_JOBS_LLVM_COMPILE
LW_JOBS_KERNEL_COMPILE
LW_JOBS_MUSL_COMPILE
LW_JOBS_BUSYBOX_COMPILE
```

### Docker
The following commands should be executed in this repo root.

There are two containers:
* **linux-wasm-base**: Contains an Ubuntu 20.04 environment with all tools installed for building (e.g. cmake, gcc etc.).
* **linux-wasm-contained**: Actually builds everything into the container. Meant as a disposable way to build everything isolated.

Create the containers:
```bash
docker build -t linux-wasm-base:dev ./docker/linux-wasm-base
docker build -t linux-wasm-contained:dev -f ./docker/linux-wasm-contained/Dockerfile .
```
Note that the latter command will copy linux-wasm.sh, in its current state, into the container.

To launch a simple docker container with a mapping to host (recommended for development):
```bash
docker run -it --name my-linux-wasm --mount type=bind,src="$(pwd)",target=/linux-wasm linux-wasm-base:dev bash
(Inside the bash prompt, run for example:) /linux-wasm/linux-wasm.sh all
```

To actually build everything inside the container (mostly useful for build servers):
```bash
docker run -it --name full-linux-wasm linux-wasm-contained:dev /linux-wasm/linux-wasm.sh all
```

To change workspace folder, docker run -e LW_WORKSPACE=/path/to/workspace ...blah... can be used. This may be useful together with docker volumes.
