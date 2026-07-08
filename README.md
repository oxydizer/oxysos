# OxysOS

Declarative system management for Gentoo, written in Rust.

OxysOS gives you a full Gentoo/Portage system without making you prove you deserve one first. Define your system in Rust, compile it to a verified manifest, and let the tooling handle Portage resolution, disk provisioning, kernel builds, and service management.

This is not a theme pack. The default desktop is Niri + Noctalia shell, but that's just what ships — the project is the tooling underneath it.

> OxysOS is experimental. Things will break, schemas will change, and you should not run this on a machine you can't recover by hand.

## What OxysOS actually is

A Rust-based declarative layer over Gentoo that handles:

- System configuration as a compiled, checksum-locked manifest
- Portage package resolution with per-package USE flags, keywords, and binary/source control
- Disk provisioning (ZFS, Ext4, with Btrfs and LUKS planned)
- Reproducible kernel + module builds tied together with build IDs and vermagic checks
- Hardware detection (CPU, GPU, disks, laptop power profiles, vendor-specific tooling)
- A guided terminal installer so you don't have to do all of this by hand

The closest comparison is NixOS-style declarative management, but on Gentoo with Portage, so you keep USE flags, source builds, and the full Gentoo package ecosystem.

## What OxysOS is not

- Another wallpaper-and-theme distro
- A beginner-focused Ubuntu alternative
- A binary-only desktop distribution
- A NixOS clone
- A distribution that hides Gentoo from you

OxysOS is for people who want Gentoo's control without Gentoo's installation experience.

## Declarative configuration

Systems are defined in Rust using the `oxys` crate. You don't need to write Rust to install OxysOS — the installer handles everything — but the declarative layer is there when you want reproducible, version-controlled system definitions.

```rust
use oxys::prelude::*;

pub fn config() -> Oxys {
    Oxys {
        os: Os {
            hostname: "oxys".into(),
            timezone: "Europe/London".into(),
            locale: "en_US.UTF-8".into(),
            shell: Shell::Bash,
            libc: Libc::Glibc,
        },
        hardware: Hardware {
            gpu: detect_gpu(),
            power: match (is_laptop(), is_vendor("asus")) {
                (true, true) => Power::AsusCtl,
                (true, false) => Power::Tlp,
                (false, _) => Power::None,
            },
        },
        compiler: Compiler {
            march: March::X86_64V3,
            ..Default::default()
        },
        init_system: InitSystem::Openrc,
        packages: vec![
            Package::new("gui-wm/niri").use_flags(vec!["screencast"]),
            Package::new("gui-shells/noctalia")
                .binary(true)
                .keywords(["**"]),
            Package::new("www-client/firefox-bin").use_flags(vec!["wayland"]),
            Package::new("media-video/pipewire"),
        ],
        services: oxys::services! {
            enabled: ["NetworkManager", "sshd"],
            disabled: ["lvm2-monitor", "multipathd"],
        },
        ..Default::default()
    }
}

oxys::main!(config);
```

Running `oxys compile` generates a verified `manifest.toml` consumed by the installer and package engine.

## Installer

A guided terminal installer that walks through hardware detection, disk setup, system configuration, and installation. No Rust knowledge required.

```text
0 welcome → 1 hardware → 2 disk → 3 config → 4 confirm → 5 install → 6 done
```

## Package management

OxysOS uses Gentoo Portage. By default, a binary package host is enabled so most packages install as prebuilt binaries. You can fall back to source builds, customize USE flags, and use Portage normally. OxysOS doesn't replace Portage — it wraps a declarative layer around it.

## Build pipeline

The kernel, ZFS modules, and ISO are built from one pipeline:

- `oxys-build` compiles the kernel and `sys-fs/zfs-kmod` against a single Portage snapshot, tags them with a shared build ID, and verifies vermagic compatibility
- `oxys-iso` consumes those tagged artifacts directly, with Catalyst's own kernel step disabled

This means the kernel on the install media and the kernel on the installed system are the same build. Not "compatible" — the same.

## Init system

OpenRC by default. Systemd is supported if you want it. The installer, services, and desktop integration all target OpenRC as the primary path.

## Desktop

Niri (Wayland compositor) + Noctalia shell with OxysOS defaults, color palette, and wallpaper. Keyboard-friendly, Wayland-native, and visually consistent out of the box. This is the default, not the point.

## Repository layout

```text
oxys/              Rust manifest, resolver, CLI, declarative config
oxys-build/        Podman/Gentoo kernel + module build pipeline
oxys-iso/          Catalyst-based ISO builder
oxys-installer/    Ratatui terminal installer
oxys-login/        PAM-backed TUI login manager
my-system-config/  Example user configuration crate
```

Everything lives in one repo so the install media and the installed system can't silently drift apart.

## Status

Under active development. The configuration schema, installer flow, package defaults, and build commands may all change. Don't use this on machines you care about unless you're comfortable recovering a Linux system manually.

## Documentation

- [Architecture overview](oxys/OVERVIEW.md)
- [Configuration reference](CONFIG.md)
- [ISO builder](oxys-iso/README.md)

## License

License information coming soon.
