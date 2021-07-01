# Cargo Remote

***Use with caution, I didn't test this software well and it is a really hacky
(at least for now). If you want to test it please create a VM or at least a separate
user on your build host***

## Build Guide

### Install `rsync`

On macOS, run:
```bash
brew install rsync
```
(It was reported that the `rsync` version shipped with macOS does not support the progress flag and thus fails when
`cargo-remote` tries to use it.)

### Install `cargo-remote`

Then, to install `cargo-remote`, run:
```bash
git clone https://github.com/howardwu/cargo-remote
cargo install --path cargo-remote/
```

### Configure `ssh`

1. Spin up a remote server (e.g. EC2) with __Ubuntu 20.04__.

2. Configure your `~/.ssh/config` with the following:
```
Host {INSTANCE_NAME}
    HostName {IP_ADDRESS}
    User {USER}
    IdentityFile {~/PATH/TO/PEM}
    Port 22
```

3. SSH into your remote server once to save the IP address to your `authorized_keys`.

4. While you are SSH'ed in, run the following:
```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install -y \
    build-essential \
    clang \
    gcc \
    git \
    libssl-dev \
    llvm \
    make \
    pkg-config \
    tmux \
    xz-utils

# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source $HOME/.cargo/env
```

### Configure `cargo-remote`

1. To inform `cargo-remote` about your remote server, run:
```bash
mkdir ~/.config/cargo-remote
touch ~/.config/cargo-remote/cargo-remote.toml
```

2. In your `cargo-remote.toml` file, add the following line:
```toml
remote={USER}@{IP_ADDRESS}
hidden = true
```

Note: you can alternatively place the `.cargo-remote.toml` (with a different name) at the root of your crate
if you don't want it to be global.

Setting `hidden = true` makes sure that cargo-remote also copies those files/directories starting with ".", 
which is sometimes necessary.

## Usage

To execute a Cargo command remotely, run:
```bash
cargo remote {YOUR NORMAL CARGO COMMAND}
```

For example, if you want to `cargo build`, run:
```bash
cargo remote build
```

### Flags & Options
```
USAGE:
    cargo remote [FLAGS] [OPTIONS] <command> [remote options]...

FLAGS:
    -c, --copy-back          Transfer the target folder back to the local machine
        --help               Prints help information
    -h, --transfer-hidden    Transfer hidden files and directories to the build server
    -V, --version            Prints version information

OPTIONS:
    -b, --build-env <build_env>              Set remote environment variables. RUST_BACKTRACE, CC, LIB, etc.  [default:
                                             RUST_BACKTRACE=1]
    -e, --env <env>                          Environment profile. default_value = /etc/profile [default: /etc/profile]
        --manifest-path <manifest_path>      Path to the manifest to execute [default: Cargo.toml]
    -r, --remote <remote>                    Remote ssh build server
    -d, --rustup-default <rustup_default>    Rustup default (stable|beta|nightly) [default: stable]

ARGS:
    <command>              cargo command that will be executed remotely
    <remote options>...    cargo options and flags that will be applied remotely

```

## That's it.

## Leftover "stuff" from the original repo (you can ignore)

For now only `cargo remote [FLAGS] [OPTIONS] <command>` works: it copies the
current project to a temporary directory (`~/remote-builds/<project_name>`) on
the remote server, calls `cargo <command>` remotely and optionally (`-c`) copies
back the resulting target folder. This assumes that server and client are running
the same rust version and have the same processor architecture. On the client `ssh`
and `rsync` need to be installed.

If you want to pass remote flags you have to end the options/flags section using
`--`. E.g. to build in release mode and copy back the result use:
```bash
cargo remote -c -- build --release
```

### Configuration
You can place a config file called `.cargo-remote.toml` in the same directory as your
`Cargo.toml` or at `~/.config/cargo-remote/cargo-remote.toml`. There you can define a
default remote build host and user. It can be overridden by the `-r` flag.

Example config file:
```toml
remote = "builds@myserver"
```

## Why I built it
One big annoyance when working on rust projects on my notebook are the compile
times. Since I'm using rust nightly for some of my projects I have to recompile
rather often. Currently there seem to be no good remote-build integrations for
rust, so I decided to build one my own.

## Planned capabilities
This first version is very simple (could have been a bash script), but I intend to
enhance it to a point where it detects compatibility between local and remote
versions, allows (nearly) all cargo commands and maybe even load distribution
over multiple machines.
