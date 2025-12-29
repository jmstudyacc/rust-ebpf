# Aya-rs Development Environment Setup for macOS

This guide provides comprehensive instructions for setting up an Aya-rs (eBPF) development environment on macOS using JetBrains RustRover as the IDE.

## Overview

[Aya](https://aya-rs.dev/) is a Rust library for building eBPF programs. While eBPF programs run exclusively on Linux, you can develop and cross-compile them from macOS. This guide covers:

- Installing required toolchains
- Configuring cross-compilation for Linux/eBPF targets
- Setting up RustRover IDE
- Creating and building Aya projects
- Testing with a Linux VM or remote Linux machine

## Prerequisites

### System Requirements

- macOS 12 (Monterey) or later
- At least 8GB RAM (16GB recommended)
- Xcode Command Line Tools
- Homebrew package manager

### Install Xcode Command Line Tools

```bash
xcode-select --install
```

### Install Homebrew

If you don't have Homebrew installed:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

## Step 1: Install Rust Toolchain

### Install rustup

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

Follow the prompts and select the default installation. After installation, reload your shell:

```bash
source "$HOME/.cargo/env"
```

### Verify Installation

```bash
rustc --version
cargo --version
```

### Install Required Rust Components

Aya requires the nightly toolchain and specific components:

```bash
# Install nightly toolchain
rustup install nightly

# Add the rust-src component (required for building eBPF programs)
rustup component add rust-src --toolchain nightly

# Install the BPF linker
cargo install bpf-linker
```

### Add Linux Cross-Compilation Targets

Since eBPF programs run on Linux, add the Linux target:

```bash
# For x86_64 Linux
rustup target add x86_64-unknown-linux-musl

# For aarch64 Linux (if targeting ARM-based Linux systems)
rustup target add aarch64-unknown-linux-musl
```

## Step 2: Install Additional Dependencies

### Install LLVM

LLVM is required for the BPF linker:

```bash
brew install llvm
```

Add LLVM to your PATH. Add to your `~/.zshrc` or `~/.bashrc`:

```bash
export LLVM_SYS_180_PREFIX="$(brew --prefix llvm)"
export PATH="$(brew --prefix llvm)/bin:$PATH"
```

Reload your shell configuration:

```bash
source ~/.zshrc  # or source ~/.bashrc
```

### Install Cross-Compilation Toolchain

For cross-compiling userspace programs to Linux:

```bash
# Install musl cross-compiler for Linux targets
brew install filosottile/musl-cross/musl-cross

# For x86_64 Linux
brew install filosottile/musl-cross/musl-cross --with-x86_64

# For aarch64 Linux
brew install filosottile/musl-cross/musl-cross --with-aarch64
```

### Install cargo-generate

Used to create new Aya projects from templates:

```bash
cargo install cargo-generate
```

## Step 3: Install and Configure RustRover

### Install RustRover

1. Download RustRover from [JetBrains](https://www.jetbrains.com/rust/)
2. Open the `.dmg` file and drag RustRover to Applications
3. Launch RustRover

Alternatively, install via JetBrains Toolbox:

```bash
brew install --cask jetbrains-toolbox
```

Then install RustRover through the Toolbox app.

### Configure RustRover for Aya Development

#### Set Up Rust Toolchain

1. Open RustRover
2. Go to **RustRover** → **Settings** (or `Cmd + ,`)
3. Navigate to **Languages & Frameworks** → **Rust**
4. Ensure the toolchain path points to your Rust installation (typically `~/.cargo/bin`)

#### Configure the Nightly Toolchain

1. In Settings, go to **Languages & Frameworks** → **Rust** → **Toolchain**
2. Click the `+` button to add a new toolchain
3. Select the nightly toolchain location: `~/.rustup/toolchains/nightly-*/bin`
4. Set it as the default for Aya projects

#### Install Recommended Plugins

Go to **Settings** → **Plugins** and install:

- **TOML** - For Cargo.toml syntax highlighting
- **Markdown** - For documentation
- **GitToolBox** - Enhanced Git integration
- **.ignore** - For .gitignore support

#### Configure File Watchers (Optional)

For automatic formatting on save:

1. Go to **Settings** → **Tools** → **File Watchers**
2. Add a new watcher for `rustfmt`:
   - File type: Rust
   - Program: `rustfmt`
   - Arguments: `$FilePath$`

## Step 4: Create an Aya Project

### Generate a New Project

Use the Aya template to create a new project:

```bash
cargo generate https://github.com/aya-rs/aya-template
```

You'll be prompted for:
- **Project name**: Enter your project name (e.g., `my-ebpf-project`)
- **eBPF program type**: Choose from options like `kprobe`, `xdp`, `tracepoint`, etc.

### Project Structure

After generation, your project will have this structure:

```
my-ebpf-project/
├── Cargo.toml                 # Workspace configuration
├── xtask/                     # Build tasks
│   ├── Cargo.toml
│   └── src/
│       └── main.rs
├── my-ebpf-project/           # Userspace application
│   ├── Cargo.toml
│   └── src/
│       └── main.rs
└── my-ebpf-project-ebpf/      # eBPF program (runs in kernel)
    ├── Cargo.toml
    ├── rust-toolchain.toml
    └── src/
        └── main.rs
```

### Open in RustRover

1. Open RustRover
2. Select **File** → **Open**
3. Navigate to your project directory
4. Click **Open**
5. Wait for RustRover to index the project and download dependencies

## Step 5: Configure Cross-Compilation

### Create Cargo Configuration

Create or edit `.cargo/config.toml` in your project root:

```toml
[build]
target-dir = "target"

[target.bpfel-unknown-none]
rustflags = ["-C", "link-arg=--btf"]

[target.x86_64-unknown-linux-musl]
linker = "x86_64-linux-musl-gcc"

[target.aarch64-unknown-linux-musl]
linker = "aarch64-linux-musl-gcc"
```

### Environment Variables

Add these to your shell configuration (`~/.zshrc` or `~/.bashrc`):

```bash
# For Aya/eBPF development
export CARGO_TARGET_X86_64_UNKNOWN_LINUX_MUSL_LINKER=x86_64-linux-musl-gcc
export CARGO_TARGET_AARCH64_UNKNOWN_LINUX_MUSL_LINKER=aarch64-linux-musl-gcc
```

## Step 6: Build the Project

### Build eBPF Program

From the project root:

```bash
# Build the eBPF program
cargo xtask build-ebpf
```

Or to build in release mode:

```bash
cargo xtask build-ebpf --release
```

### Build Userspace Application (for Linux)

To cross-compile the userspace application for Linux:

```bash
# For x86_64 Linux
cargo build --target x86_64-unknown-linux-musl

# For aarch64 Linux
cargo build --target aarch64-unknown-linux-musl
```

### Configure RustRover Build Tasks

1. Go to **Run** → **Edit Configurations**
2. Click `+` → **Cargo**
3. Create the following configurations:

**Build eBPF:**
- Name: `Build eBPF`
- Command: `xtask build-ebpf`
- Working directory: `$ProjectFileDir$`

**Build eBPF (Release):**
- Name: `Build eBPF Release`
- Command: `xtask build-ebpf --release`
- Working directory: `$ProjectFileDir$`

**Build for Linux x86_64:**
- Name: `Build Linux x86_64`
- Command: `build --target x86_64-unknown-linux-musl`
- Working directory: `$ProjectFileDir$`

## Step 7: Testing on Linux with Lima VM

Since eBPF programs can only run on Linux, you need a Linux environment for testing. This project uses [Lima](https://lima-vm.io/), a lightweight Linux VM solution for macOS that provides seamless file sharing and SSH access.

### Install Lima

```bash
brew install lima
```

### Create the Aya Development VM

Use the provided configuration file to create a VM with:
- **4 CPU cores**
- **8GB RAM**
- **50GiB storage**
- **Ubuntu 24.04 LTS** (excellent eBPF/BTF support)
- Pre-installed Rust toolchain, bpf-linker, and eBPF development tools

```bash
# Create and start the VM (from the project root)
limactl create --name=aya-dev lima/aya-dev.yaml
limactl start aya-dev
```

The first startup takes several minutes as it provisions the VM with all required tools.

### Connect to the VM

**Option A: Lima Shell (Recommended)**

```bash
limactl shell aya-dev
```

**Option B: Direct SSH (Port 2222)**

The VM is configured with a fixed SSH port for easy access:

```bash
# Connect using SSH on port 2222
ssh -p 2222 -i ~/.lima/_config/user $USER@127.0.0.1
```

Or add this to your `~/.ssh/config` for convenience:

```
Host aya-dev
    HostName 127.0.0.1
    Port 2222
    User <your-macos-username>
    IdentityFile ~/.lima/_config/user
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null
```

Then simply connect with:

```bash
ssh aya-dev
```

### File Sharing

Your macOS home directory is automatically mounted in the VM at the same path. No file copying needed!

```bash
# On macOS
cd ~/Projects/my-ebpf-project

# In the Lima VM - same path works
limactl shell aya-dev
cd ~/Projects/my-ebpf-project
cargo xtask build-ebpf
sudo ./target/debug/my-ebpf-project
```

### VM Management Commands

```bash
# Start the VM
limactl start aya-dev

# Stop the VM
limactl stop aya-dev

# Restart the VM
limactl stop aya-dev && limactl start aya-dev

# Delete the VM (removes all data)
limactl delete aya-dev

# List all VMs
limactl list

# Show VM info
limactl info aya-dev
```

### Configure RustRover for Lima SSH

1. Go to **RustRover** → **Settings** → **Tools** → **SSH Configurations**
2. Click `+` to add a new configuration:
   - **Host**: `127.0.0.1`
   - **Port**: `2222`
   - **User**: Your macOS username
   - **Authentication**: Key pair
   - **Private key**: `~/.lima/_config/user`
3. Test the connection

For remote development:
1. Go to **File** → **Remote Development** → **SSH**
2. Select your Lima SSH configuration
3. Choose the project directory (same path as on macOS)

## Step 8: Debugging

### RustRover Debugging Configuration

For debugging the userspace application on Linux:

1. Set up remote debugging in RustRover
2. Go to **Run** → **Edit Configurations**
3. Add a **Remote Debug** configuration:
   - Host: Your Linux machine's IP
   - Port: 2345 (default gdbserver port)

On the Linux machine:

```bash
# Start gdbserver
gdbserver :2345 ./my-ebpf-project
```

### eBPF Debugging Tips

1. **Use bpf_printk for logging:**
   ```rust
   // In your eBPF code
   use aya_ebpf::macros::map;
   use aya_ebpf::helpers::bpf_printk;

   bpf_printk!(b"Debug message: %d", value);
   ```

2. **View kernel trace output:**
   ```bash
   # On Linux
   sudo cat /sys/kernel/debug/tracing/trace_pipe
   ```

3. **Use bpftool for inspection:**
   ```bash
   # List loaded BPF programs
   sudo bpftool prog list

   # Show BPF maps
   sudo bpftool map list
   ```

## Troubleshooting

### Common Issues

#### "error: linker `bpf-linker` not found"

Ensure bpf-linker is installed and in your PATH:

```bash
cargo install bpf-linker
which bpf-linker  # Should show ~/.cargo/bin/bpf-linker
```

#### LLVM Version Mismatch

If you see LLVM-related errors:

```bash
# Check LLVM version
llvm-config --version

# Ensure it matches bpf-linker requirements (LLVM 15-18)
brew upgrade llvm
cargo install bpf-linker --force
```

#### "rust-src component not found"

```bash
rustup component add rust-src --toolchain nightly
```

#### Cross-compilation linker errors

Verify musl-cross is properly installed:

```bash
which x86_64-linux-musl-gcc
# Should output the path to the linker
```

If not found, reinstall:

```bash
brew reinstall filosottile/musl-cross/musl-cross
```

### Lima VM Issues

#### VM fails to start

```bash
# Check Lima logs
limactl info aya-dev
cat ~/.lima/aya-dev/ha.stderr.log

# Try recreating the VM
limactl delete aya-dev
limactl create --name=aya-dev lima/aya-dev.yaml
limactl start aya-dev
```

#### SSH connection refused

```bash
# Verify VM is running
limactl list

# Test SSH connection on port 2222
ssh -p 2222 -i ~/.lima/_config/user $USER@127.0.0.1

# Wait for provisioning to complete (check cloud-init status in VM)
limactl shell aya-dev -- cloud-init status --wait
```

#### Rust/cargo not found in VM

The provisioning script runs as the user. If Rust is missing:

```bash
limactl shell aya-dev

# Re-run Rust installation
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
rustup install nightly
rustup component add rust-src --toolchain nightly
cargo install bpf-linker
```

#### File mounts not working

```bash
# Verify mounts
limactl shell aya-dev -- mount | grep virtiofs

# Check if the directory exists on macOS
ls -la ~/Projects/my-ebpf-project
```

### RustRover-Specific Issues

#### Rust Analyzer not working

1. Go to **Settings** → **Languages & Frameworks** → **Rust**
2. Ensure "Use Rust Analyzer" is enabled
3. Invalidate caches: **File** → **Invalidate Caches** → **Invalidate and Restart**

#### Macros not expanding

Aya uses procedural macros. If they're not expanding:

1. Enable macro expansion: **Settings** → **Languages & Frameworks** → **Rust** → **Expand declarative macros**
2. Rebuild project indices

## Quick Reference

### Essential Commands

```bash
# Build eBPF program
cargo xtask build-ebpf

# Build eBPF (release)
cargo xtask build-ebpf --release

# Build userspace for Linux
cargo build --target x86_64-unknown-linux-musl --release

# Run on Linux (requires root)
sudo ./target/x86_64-unknown-linux-musl/release/my-ebpf-project

# Check loaded eBPF programs (on Linux)
sudo bpftool prog list
```

### Lima VM Commands

```bash
# Create the VM (first time only)
limactl create --name=aya-dev lima/aya-dev.yaml

# Start the VM
limactl start aya-dev

# Connect to the VM (Lima shell)
limactl shell aya-dev

# Connect via SSH (port 2222)
ssh -p 2222 -i ~/.lima/_config/user $USER@127.0.0.1

# Stop the VM
limactl stop aya-dev

# Check VM status
limactl list

# Delete the VM
limactl delete aya-dev
```

### Complete Workflow Example

```bash
# 1. On macOS - Build the eBPF program
cd ~/Projects/my-ebpf-project
cargo xtask build-ebpf --release

# 2. Start Lima VM (if not running)
limactl start aya-dev

# 3. Connect to VM and run
limactl shell aya-dev
cd ~/Projects/my-ebpf-project
sudo ./target/debug/my-ebpf-project

# 4. In another terminal, view eBPF logs
limactl shell aya-dev -- sudo cat /sys/kernel/debug/tracing/trace_pipe
```

### Useful Resources

- [Aya Book](https://aya-rs.dev/book/) - Official Aya documentation
- [Aya GitHub](https://github.com/aya-rs/aya) - Source code and examples
- [eBPF.io](https://ebpf.io/) - General eBPF resources
- [RustRover Documentation](https://www.jetbrains.com/help/rust/) - IDE documentation
- [Lima Documentation](https://lima-vm.io/) - Lima VM for macOS

## Example: Simple XDP Program

Here's a minimal XDP program that counts packets:

### eBPF Program (`my-ebpf-project-ebpf/src/main.rs`)

```rust
#![no_std]
#![no_main]

use aya_ebpf::{bindings::xdp_action, macros::xdp, programs::XdpContext};
use aya_log_ebpf::info;

#[xdp]
pub fn my_xdp(ctx: XdpContext) -> u32 {
    match try_my_xdp(ctx) {
        Ok(ret) => ret,
        Err(_) => xdp_action::XDP_ABORTED,
    }
}

fn try_my_xdp(ctx: XdpContext) -> Result<u32, u32> {
    info!(&ctx, "received a packet");
    Ok(xdp_action::XDP_PASS)
}

#[panic_handler]
fn panic(_info: &core::panic::PanicInfo) -> ! {
    unsafe { core::hint::unreachable_unchecked() }
}
```

### Userspace Application (`my-ebpf-project/src/main.rs`)

```rust
use anyhow::Context;
use aya::{include_bytes_aligned, Bpf};
use aya::programs::{Xdp, XdpFlags};
use aya_log::BpfLogger;
use log::{info, warn};
use tokio::signal;

#[tokio::main]
async fn main() -> Result<(), anyhow::Error> {
    env_logger::init();

    // Load the eBPF program
    #[cfg(debug_assertions)]
    let mut bpf = Bpf::load(include_bytes_aligned!(
        "../../target/bpfel-unknown-none/debug/my-ebpf-project"
    ))?;

    #[cfg(not(debug_assertions))]
    let mut bpf = Bpf::load(include_bytes_aligned!(
        "../../target/bpfel-unknown-none/release/my-ebpf-project"
    ))?;

    // Initialize logging
    if let Err(e) = BpfLogger::init(&mut bpf) {
        warn!("failed to initialize eBPF logger: {}", e);
    }

    // Get the XDP program
    let program: &mut Xdp = bpf.program_mut("my_xdp").unwrap().try_into()?;
    program.load()?;

    // Attach to network interface
    program.attach("eth0", XdpFlags::default())
        .context("failed to attach the XDP program")?;

    info!("XDP program attached. Waiting for Ctrl-C...");
    signal::ctrl_c().await?;
    info!("Exiting...");

    Ok(())
}
```

## License

This documentation is provided under the MIT License.
