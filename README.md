<p align="center">
  <a href="https://www.uni-duesseldorf.de/home/en/home.html"><img src="media/d3os.png" width=460></a>
</p>

**A new distributed operating system for data centers, developed by the [operating systems group](https://www.cs.hhu.de/en/research-groups/operating-systems.html) of the department of computer science at [Heinrich Heine University Düsseldorf](https://www.hhu.de)**

<p align="center">
  <a href="https://www.uni-duesseldorf.de/home/en/home.html"><img src="media/hhu.svg" width=300></a>
</p>

<p align="center">
  <a href="https://github.com/hhu-bsinfo/D3OS/actions/workflows/build.yml"><img src="https://github.com/hhu-bsinfo/D3OS/actions/workflows/build.yml/badge.svg"></a>
  <img src="https://img.shields.io/badge/Rust-2024-blue.svg">
  <img src="https://img.shields.io/badge/license-GPLv3-orange.svg">
</p>

## Requirements

For building D3OS, the following packages for Debian/Ubuntu based systems (or their equivalent packages on other distributions) need to be installed:
```bash
apt install rustup build-essential nasm dosfstools wget qemu-system-x86
```

This has been tested on Ubuntu 24.04.

For macOS, the same can be achieved with:
```bash
xcode-select --install
brew install rustup dosfstools nasm x86_64-elf-gcc gnu-tar wget qemu
brew link --force rustup
```

This has been tested on macOS 14.

[rustup](https://rustup.rs/) will download a _rust nightly_ toolchain on the first compile.

To run the build, the commands _cargo-make_ and _cargo-license_ are required. Install them with:
```bash
cargo install --no-default-features cargo-make cargo-license
```


## Build and Run

To build D3OS and run it in QEMU, just execute:
```bash
cargo make --no-workspace
```

To build a release version of D3OS (much faster) and run it in QEMU, just execute:
```bash
cargo make --no-workspace --profile production
```


To only build the bootable image _d3os.img_, run:
```bash
cargo make --no-workspace image
```

## Debugging 

### In a terminal with gdb

Open a terminal and compile and start D3OS in `qemu` halted by `gdb` with the following commands:
```bash
cargo make --no-workspace clean
cargo make --no-workspace debug
```

Open another terminal and start `gdb` with:
```bash
cargo make --no-workspace gdb
```
This will fire booting D3OS and stop in `boot.rs::start`.

Setting a breakpoint in `gdb`:
```bash
break kernel::naming::api::init
```
For further commands check [GDB Quick Reference](docs/gdb-commands.pdf).

## Creating a bootable USB stick

### Using towboot
D3OS uses [towboot](https://github.com/hhuOS/towboot) which is already installed after you have successfully compiled D3OS. 

Use following command (in the D3OS directory) to create a bootable media for the device referenced by `/mnt/external`

`$ towbootctl install /mnt/external --removable -- -config towboot.toml`

### Using balenaEtcher
Write the file `d3os.img` using [balenaEtcher](https://etcher.balena.io) to your USB stick.

## Passing an existing PCI device to the VM

To use a real device with QEMU, change the Makefile so that it uses `${CARGO_MAKE_WORKSPACE_WORKING_DIRECTORY}/qemu-pci.sh` instead of `qemu-system-x86_64`.
Also take a look at that script and fill in the constants at the top.

If you want to run D3OS on a different device, build with `cargo make --no-workspace image` and copy over `qemu-pci.sh`, `RELEASEX64_OVMF.fd` and `d3os.img`.
Run it with `./qemu-pci.sh -bios RELEASEX64_OVMF.fd -hda d3os.img`.

## RDMA Support

This section describes how to set up, build, run, and debug RDMA over InfiniBand within this project.

### InfiniBand Setup

All InfiniBand-related configuration files are located under `configs/infiniband`. Copy the following files from that directory:

- `nix-shell-config`
- `run_template.sh`
- The setup script you want to use

During development and testing, `d3os_run_simple.sh` was used, but the setup is flexible and allows alternative scripts to be integrated and evaluated.

The predefined aliases assume that all setup scripts are placed in `~/run`. In addition, the current scripts expect the nix configuration file to be located in `~/configs`. This directory layout can easily be prepared, for example by transferring the files with `scp`.

Before executing any setup script, required environment variables must be loaded into the shell. This is done by sourcing the `dev-pc` file from `configs/defaults`. That file should define the address of your development machine (typically your home PC). Sourcing it configures a remote filesystem that is accessible by the InfiniBand hosts and used to load the required images.

On each `IB*` host, start a `nix-shell` using the `nix-shell-config` from the InfiniBand directory. Entering the shell automatically defines an alias for running the configured routine. For the simple setup described above, this alias is `start-d3os-simple`.

### Build and Image Generation

The build system follows a per-instance model. Each InfiniBand host builds its own artifacts, embedding only host-specific information. As a result, generated images are tied to the host on which they were built.

For example, building on `ib3` produces an image named `d3os-ib3.img`, with all intermediate files written to `target-ib3`. The build process relies on custom rules and targets, and it is strongly recommended to use the provided build scripts rather than attempting to assemble images manually.

The boot loader uses profile-specific generated files, allowing independent configuration per InfiniBand instance.

When using the `simple-run` setup, network ports are currently fixed and must match exactly. Otherwise, the system will not work as expected. The required values are:

- **Host port:** `1797`
- **Guest port:** `1324` (must be explicitly defined, as it is not exposed as a symbol in the Rust modules)

The main entry point for building InfiniBand workloads is `scripts/infiniband-build.sh`. It supports multiple use cases through a set of command-line options:

- `-s` (source IP) and `-g` (gateway IP) are deprecated due to the migration to DHCP; any IPv4 address may be supplied as a placeholder.
- `-t` specifies the target IP address of the peer InfiniBand instance and is required.
- `-r` marks the current host as the RDMA receiver (this is unrelated to packet reception on the local network).
- `-h` identifies the current InfiniBand instance (e.g. `ib3`).
- `-d` selects the device family. Currently `mlx4` is supported, with `mlx5` planned as an extension.

Two build procedures are available and must be specified after all options using `--`: `test` and `bench`. Each procedure conditionally includes different code paths in the final executable.

- The `test` procedure supports the `-o` option to include `read`, `write`, or `stat` operations.
- The `bench` procedure supports the `-b` option to include latency, IOPS, or throughput benchmarks.

An example of building a test configuration with a read operation on `ib3` is shown below:

```bash
/bin/bash scripts/infiniband-build.sh \
  -d mlx4 -h ib3 -s 0.0.0.0 -g 0.0.0.0 -t <ib4-ip> -p 1797 -o read -- test
```

The corresponding build on the peer instance (`ib4`) would be:

```bash
/bin/bash scripts/infiniband-build.sh \
  -d mlx4 -h ib4 -s 0.0.0.0 -g 0.0.0.0 -t <ib3-ip> -p 1797 -r -o read -- test
```

### Connecting

To interact with the InfiniBand instances, connect to each host via SSH. Logging can be performed using the controlling terminal, or alternatively through a serial console for more persistent output.

If the InfiniBand instances are running under QEMU and you want their graphical output to appear on your development machine, make sure to enable X11 forwarding by using the `-X` option when connecting via SSH. Without X11 forwarding, the QEMU display will not be visible on your local screen.

### Debugging

Helper scripts for debugging are available under `configs/defaults` for each InfiniBand instance. These scripts query the host database and therefore require that each instance’s hostname is listed in `/etc/hosts` on both machines.

Once the hostnames have been added, source the appropriate default script for the instance and then run:


`scripts/infiniband-remote-gdb.sh`
