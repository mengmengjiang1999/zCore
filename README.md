# zCore

[![CI](https://github.com/rcore-os/zCore/workflows/CI/badge.svg?branch=master)](https://github.com/rcore-os/zCore/actions)
[![Docs](https://img.shields.io/badge/docs-alpha-blue)](https://rcore-os.github.io/zCore/)
[![Coverage Status](https://coveralls.io/repos/github/rcore-os/zCore/badge.svg?branch=master)](https://coveralls.io/github/rcore-os/zCore?branch=master)

Reimplement [Zircon][zircon] microkernel in safe Rust as a userspace program!




## Getting started

Environments：

- [Rust toolchain](http://rustup.rs)
- [QEMU](https://www.qemu.org)

### Developing environment info

- current rustc -- rustc 1.64.0-nightly (f6f9d5e73 2022-08-04)
- current rust-toolchain -- nightly-2022-08-05
- current qemu -- 5.2.0 -> 6.2.0 -> 7.0.0

Clone repo and pull prebuilt fuchsia images:

```sh
git clone https://github.com/elliott10/zCore.git --recursive
cd zCore
make setup
```
Use docker container as standand develop environment, please refer to [tootls/docker](https://github.com/rcore-os/zCore/tree/master/tools/docker).

### Run zcore in libos mode

#### Run zcore in linux-libos mode

- step 1: Prepare Alpine Linux rootfs:

  ```sh
  make rootfs
  ```

- step 2: Compile & Run native Linux program (Busybox) in libos mode:

  ```sh
  cd zCore
  cargo run --release --features "linux libos" -- /bin/busybox [args]
  ```

  当然了，按照以上方法，编译出来的zcore会默认认为根目录在`./rootfs/libos`，可以通过以下命令修改默认根目录。
  我对./zCore/src/fs.rs中的代码进行了一定修改之后的方式。比如说我现在想要让根目录放在corpus目录下，与此同时busybox的绝对路径是

  `~/Project/fuzzing/zCore-fuzzing/zcore-test/corpus/libos/bin/busybox`
  
  就需要：

  ```sh
  export CARGO_MANIFEST_DIR=~/Project/fuzzing/zCore-fuzzing/zcore-test/corpus
  cargo run --release --features "linux libos"
  /home/mmj/Project/fuzzing/zCore-fuzzing/zcore-test/zCore/target/release/zcore /bin/busybox
  ```

  You can also add the feature `graphic` to show the graphical output (with [sdl2](https://www.libsdl.org) installed).

  To debug, set the `LOG` environment variable to one of `error`, `warn`, `info`, `debug`, `trace`.

#### Run native Zircon program (shell) in zircon-libos mode:

- step 1: Compile and Run Zircon shell

  ```sh
  cd zCore
  cargo run --release --features "zircon libos" -- prebuilt/zircon/x64/bringup.zbi
  ```

  The `graphic` and `LOG` options are the same as Linux.

### Run zcore in bare-metal mode

#### Run Linux shell in  linux-bare-metal mode:
---
**Quick start for RISCV64**

```sh
make image ARCH=riscv64
cd zCore
make run ARCH=riscv64 LINUX=1
#OR make run LINUX=1 MODE=release SMP=1 ARCH=riscv64 PLATFORM=qemu LOG=debug
```

**Quick start for AARCH64**

```sh
make image ARCH=aarch64
cd zCore
make run LINUX=1 ARCH=aarch64 PLATFORM=qemu LOG=warn
```

---
- step 1: Prepare Alpine Linux rootfs:

  ```sh
  make rootfs
  ```

- step 2: Create Linux rootfs image:

  Note: Before below step, you can add some special apps in zCore/rootfs

  ```sh
  make image
  ```

- step 3: Build and run zcore in  linux-bare-metal mode:

  ```sh
  cd zCore && make run MODE=release LINUX=1 [LOG=warn] [GRAPHIC=on] [ACCEL=1]
  ```

#### Run Zircon shell in zircon-bare-metal mode:

- step 1: Build and run zcore in  zircon-bare-metal mode:

  ```sh
  cd zCore && make run MODE=release [LOG=warn] [GRAPHIC=on] [ACCEL=1]
  ```

- step 2: Build and run your own Zircon user programs:

  ```sh
  # See template in zircon-user
  cd zircon-user && make zbi MODE=release
  
  # Run your programs in zCore
  cd zCore && make run MODE=release USER=1 [LOG=warn] [GRAPHIC=on] [ACCEL=1]
  ```

## Testing

### LibOS Mode Testing

#### Zircon related

Run Zircon official core-tests:

```sh
pip3 install pexpect
cd scripts && python3 unix-core-testone.py 'Channel.*'
```

Run all (non-panicked) core-tests for CI:

```sh
pip3 install pexpect
cd scripts && python3 unix-core-tests.py
# Check `zircon/test-result.txt` for results.
```

#### Linux related

Run Linux musl libc-tests for CI:

```sh
make rootfs && make libc-test
cd scripts && python3 libos-libc-tests.py
# Check `linux/test-result.txt` for results.
```

### Bare-metal Mode Testing

#### Zircon related

Run Zircon official core-tests on bare-metal:

```sh
cd zCore && make test MODE=release [ACCEL=1] TEST_FILTER='Channel.*'
```

Run all (non-panicked) core-tests for CI:

```sh
pip3 install pexpect
cd scripts && python3 core-tests.py
# Check `zircon/test-result.txt` for results.
```

#### x86-64 Linux related

Run Linux musl libc-tests for CI:

```sh
##  Prepare rootfs with libc-test apps
make baremetal-test-img
## Build zCore kernel
cd zCore && make build MODE=release LINUX=1 ARCH=x86_64
## Testing
cd scripts && python3 baremetal-libc-test.py
##
```

You can use [`scripts/baremetal-libc-test-ones.py`](./scripts/baremetal-libc-test-ones.py) & [`scripts/linux/baremetal-test-ones.txt`](./scripts/linux/baremetal-test-ones.txt) to test specified apps.

[`scripts/linux/baremetal-test-fail.txt`](./scripts/linux/baremetal-test-fail.txt) includes all failed x86-64 apps (We need YOUR HELP to fix bugs!)

#### riscv-64 Linux related

Run Linux musl libc-tests for CI:

```sh
##  Prepare rootfs with libc-test & oscomp apps
make riscv-image
## Build zCore kernel & Testing
cd scripts && python3 baremetal-test-riscv64.py
##
```

You can use[scripts/baremetal-libc-test-ones-riscv64.py](./scripts/baremetal-libc-test-ones-riscv64.py) & [`scripts/linux/baremetal-test-ones-rv64.txt`](scripts/linux/baremetal-test-ones-rv64.txt)to test
specified apps.

[`scripts/linux/baremetal-test-fail-riscv64.txt`](./scripts/linux/baremetal-test-fail-riscv64.txt)includes all failed riscv-64 apps (We need YOUR HELP to fix bugs!)

## Graph/Game

snake game: <https://github.com/rcore-os/rcore-user/blob/master/app/src/snake.c>

### Step1: compile usr app

We can use musl-gcc compile it in x86_64 mode

### Step2: change zcore for run snake app first.

change zCore/zCore/main.rs L176
vec!["/bin/busybox".into(), "sh".into()]
TO
vec!["/bin/snake".into(), "sh".into()]

### Step3: prepare root fs image, run zcore in linux-bare-metal mode

exec:

```sh
cd zCore #zCore ROOT DIR
make rootfs
cp ../rcore-user/app/snake rootfs/bin #copy snake ELF file to rootfs/bin
make image # build rootfs image
cd zCore #zCore kernel dir
make run MODE=release LINUX=1 GRAPHIC=on
```

Then you can play the game.
Operation

- Keyboard
  - `W`/`A`/`S`/`D`: Move
  - `R`: Restart
  - `ESC`: End
- Mouse
  - `Left`: Speed up
  - `Right`: Slow down
  - `Middle`: Pause/Resume

## Doc

```
make doc
```

### RISC-V 64 porting info

- [porting riscv64 doc](./docs/porting-rv64.md)

## Components

### Overview

![](./docs/structure.svg)

[zircon]: https://fuchsia.googlesource.com/fuchsia/+/master/zircon/README.md
[kernel-objects]: https://github.com/PanQL/zircon/blob/master/docs/objects.md
[syscalls]: https://github.com/PanQL/zircon/blob/master/docs/syscalls.md

### Hardware Abstraction Layer

|                           | Bare Metal | Linux / macOS     |
| :------------------------ | ---------- | ----------------- |
| Virtual Memory Management | Page Table | Mmap              |
| Thread Management         | `executor` | `async-std::task` |
| Exception Handling        | Interrupt  | Signal            |

### Small Goal & Little Plans

- <https://github.com/rcore-os/zCore/wiki/Plans>
## Dev Status

🚧 Working In Progress

- 2020.04.16: Zircon console is working on zCore! 🎉
