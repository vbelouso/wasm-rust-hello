- [WASM usage in OCI containers (based on RHEL 8.x/Podman/Buildah/Wasmtime)](#wasm-usage-in-oci-containers-based-on-rhel-8xpodmanbuildahwasmtime)
  - [Prerequisites](#prerequisites)
  - [Runtime configuration](#runtime-configuration)
    - [Enable crun](#enable-crun)
    - [Build crun with WASM support](#build-crun-with-wasm-support)
  - [Create Rust application](#create-rust-application)
    - [Using an example from the repository](#using-an-example-from-the-repository)
    - [Build Rust application from scratch](#build-rust-application-from-scratch)
  - [Container creation](#container-creation)
    - [Containerfile](#containerfile)
    - [Build](#build)
    - [Run](#run)

# WASM usage in OCI containers (based on RHEL 8.x/Podman/Buildah/Wasmtime)

## Prerequisites

Installed [Rust](https://www.rust-lang.org), [Wasmtime](https://wasmtime.dev), [Podman](https://podman.io/), [Buildah](https://buildah.io/), [Git](https://git-scm.com/)

Installation process

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
curl https://wasmtime.dev/install.sh -sSf | bash
sudo yum -y install podman buildah git
source "$HOME/.cargo/env"
source "$HOME/.bashrc"
```

## Runtime configuration

Check the current container runtime configuration for Podman.

```shell
podman info | grep ociRuntime -A10
```

If the command output is the same as following we need to enable `crun` as default runtime and build it with WASM support.

```text
name: crun
package: crun-1.5-1.module+el8.6.0+16771+28dfca77.x86_64
path: /usr/bin/crun
version: |-
  crun version 1.5
  commit: 54ebb8ca8bf7e6ddae2eb919f5b82d1d96863dea
  spec: 1.0.0
  +SYSTEMD +SELINUX +APPARMOR +CAP +SECCOMP +EBPF +YAJL
```

### Enable crun

```shell
cat | sudo tee /etc/containers/containers.conf <<EOF
[engine]
runtime = "crun"
EOF
```

### Build crun with WASM support

We need to use `Wasmtime C API` so that crun compiles successfully. [Wasmtime C API reference](https://docs.wasmtime.dev/c-api/)

```shell
curl -L -o wasmtime-v2.0.2-x86_64-linux-c-api.tar.xz https://github.com/bytecodealliance/wasmtime/releases/download/v2.0.2/wasmtime-v2.0.2-x86_64-linux-c-api.tar.xz

tar -xf wasmtime-v2.0.2-x86_64-linux-c-api.tar.xz
sudo cp -R wasmtime-v2.0.2-x86_64-linux-c-api/include/* /usr/local/include/
sudo cp -R wasmtime-v2.0.2-x86_64-linux-c-api/lib/* /usr/lib64/
rm -f wasmtime-v2.0.2-x86_64-linux-c-api.tar.xz
```

Dependencies installation

```shell
sudo yum install -y make automake \
    autoconf gettext \
    libtool gcc libcap-devel systemd-devel yajl-devel libgcrypt-devel \
    glibc-static libseccomp-devel python36
```

Build process

```shell
git clone git@github.com:containers/crun.git /tmp/crun
cd /tmp/crun
./autogen.sh
./configure --with-wasmtime
make
sudo make install
```

Please make sure that crun builded with `+WASM:wasmtime` support.

```shell
podman info | grep ociRuntime -A10
```

Example output:

```text
ociRuntime:
    name: crun
    package: Unknown
    path: /usr/local/bin/crun
    version: |-
      crun version 1.7.0.0.0.24-b42e
      commit: b42e7ec0b199a0ee6409d645228032a187d17c66
      rundir: /run/user/1003/crun
      spec: 1.0.0
      +SYSTEMD +SELINUX +APPARMOR +CAP +SECCOMP +EBPF +WASM:wasmtime +YAJL
  os: linux
```

## Create Rust application

You can create Rust application from scratch with the following process or just use an example from current repo.

### Using an example from the repository

Switch to the cloned repository directory and run the following commands.

```shell
buildah build --annotation "module.wasm.image/variant=compat" -t wasm-rust-hello
podman run wasm-rust-hello
```

### Build Rust application from scratch

```shell
cd /tmp
cargo new hello-world --vcs none --bin
cd hello-world
rustup target add wasm32-wasi
cargo build --target wasm32-wasi --release
wasmtime target/wasm32-wasi/release/hello-world.wasm
```

Example output:

```text
$ wasmtime target/wasm32-wasi/release/hello-world.wasm
Hello, world!
```

> :information_source: To reduce final container image size we can add the following option to `Cargo.toml`

```toml
[profile.release]
strip = true
```

```shell
cat << EOF | tee -a Cargo.toml
[profile.release]
strip = true
EOF
```

## Container creation

To run a container with WASM, we must add an annotation `"module.wasm.image/variant=compat"` for the created container image.

### Containerfile

```docker
FROM scratch
COPY ./target/wasm32-wasi/release/hello-world.wasm /
CMD ["/hello-world.wasm"]
```

### Build

```shell
buildah build --annotation "module.wasm.image/variant=compat" -t wasm-rust-hello
```

### Run

```shell
podman run wasm-rust-hello
```
