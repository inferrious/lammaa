# Lammaa

**Minimal Hypervisor Host • Super Enhanced CachyOS Style • Immutable • VM Shimmed**

A bare-minimum, highly tweaked, immutable hypervisor appliance built as a CachyOS PKG fork.  
The host (Layer 1) runs a minimal kernel + initramfs with libvirtd as a powerful pre-boot control system. The main appliance and all workload VMs run inside a cleanly shimmed guest (Layer 2). Everything is immutable, versioned, and easy to roll back.

**GitHub**: https://github.com/inferrious/lammaa

---

## Vision

- Extremely cut-down appliance
- Kernel + minimal initramfs as the hypervisor shim (Layer 1)
- Full desktop + virt stack runs inside a shimmed VM (Layer 2)
- NixOS-style rebuild workflow for major changes
- Atomic, versioned squashfs layers with easy rollback
- LUKS2 + YubiKey as the outer security shell
- One kernel binary shared between host and guest where possible
- libvirt runs in **Layer 1** as the pre-boot control system (full XML-driven hardware shim)

---

## Architecture

### Dual SquashFS Layers

| Layer | Content | Purpose | Size Target |
|-------|---------|---------|-------------|
| **Layer 1 (Hypervisor)** | Custom kernel + minimal initramfs + **libvirtd** + dracut hook | Boots the system, unlocks LUKS, mounts layers, starts libvirtd, defines & launches appliance VM via XML | ~300-500 MB (acceptable on 3× NVMe RAID0) |
| **Layer 2 (Appliance)** | Minimal GNOME + virt-manager + libvirtd + your tools + app layers | Full desktop, VM management, workload VMs (nested) | ~2–4 GB |

### Boot Flow

1. Limine loads kernel from Layer 1 squashfs
2. Minimal initramfs runs dracut hook:
   - Unlocks LUKS2 (YubiKey)
   - Mounts both squashfs layers read-only
   - Creates overlayfs on btrfs writable subvolumes
3. Two modes (selected via kernel cmdline):
   - `traditional` → switch_root into overlay
   - `shim` → starts minimal `libvirtd`, defines the appliance VM from XML (stored in Layer 2), then launches it via QEMU (virtio-gpu + SPICE)
4. Guest (Layer 2) runs the full desktop + nested workload VMs (libvirt also available inside for further management)

### Storage

- LUKS2 container on Btrfs (multi-device or RAID0)
- Subvolumes: `@` (base), `@var`, `@etc`, `@layers` (your virtual-disk app layers), `@vms`, `@snapshots`

---

## Key Features

- **Super Enhanced CachyOS Kernel** — 1000 Hz, full `PREEMPT`, BORE scheduler, stripped debug, optimized for Ryzen + NVIDIA + NVMe + KVM
- **Immutable by Default** — Read-only squashfs bases + writable overlays only where needed
- **VM Shimmed** — The "real" OS runs inside QEMU for maximum isolation and clean separation
- **NixOS-style Rebuilds** — Update files in `build/` → one command → new dated squashfs versions. Old versions kept for instant rollback.
- **Powerful Pre-boot Control** — libvirt + libvirtd runs in Layer 1 for full XML-driven hardware shim from the earliest stage (size acceptable on 3× NVMe RAID0)
- **Matryoshka App Layers** (future) — Self-extracting virtual-disk layers using your Python cat technique
- **YubiKey LUKS2** — Main security layer

---

## Project Structure

```
lammaa/
├── build/                      # NixOS-style build workspace
│   ├── versions/               # Dated squashfs outputs (kept for rollback)
│   ├── hypervisor/             # mkosi profile for Layer 1
│   ├── appliance/              # mkosi profile for Layer 2
│   ├── configs/                # kernel cmdline, libvirt templates, shim config
│   └── scripts/
│       ├── build.sh            # Main rebuild script
│       └── rollback.sh
├── kernel/                     # Forked CachyOS linux-cachyos PKGBUILD
├── dracut/                     # Custom 99dual-squashfs module
├── docs/
│   ├── BUILD.md
│   ├── SHIM.md
│   └── ROLLBACK.md
├── README.md
└── LICENSE
```

---

## Build Workflow (NixOS-style)

```bash
cd build

# Make changes (kernel config, packages, configs, etc.)
nano configs/shim.json
# or edit kernel PKGBUILD, mkosi profiles, etc.

./scripts/build.sh --layer both --version $(date +%Y-%m-%d)

# Output:
# versions/hypervisor-2026-07-06.squashfs
# versions/appliance-2026-07-06.squashfs

# Update symlinks (optional)
ln -sf versions/hypervisor-2026-07-06.squashfs current-hypervisor.squashfs
ln -sf versions/appliance-2026-07-06.squashfs current-appliance.squashfs
```

Old versions remain in `versions/` for instant rollback via Limine boot menu.

---

## Roadmap / Phases

### Phase 0 – Foundation (Current)
- [x] Project structure & README
- [ ] Repo initialization on GitHub (inferrious/lammaa)
- [ ] Basic mkosi profiles for both layers

### Phase 1 – Custom Kernel (High Priority)
- Fork `linux-cachyos-bore`
- Apply 1000 Hz + full PREEMPT + aggressive stripping
- Build and test

### Phase 2 – Light Layer 1 + Dracut Hook
- Minimal mkosi profile for hypervisor
- Updated `dual-squashfs.sh` hook (lighter, with QEMU shim launch)
- Overlayfs + LUKS unlock integration

### Phase 3 – QEMU Shim Mode
- Direct QEMU launch in initramfs with virtio-gpu + SPICE
- Simple config/JSON parser for shim parameters
- Guest kernel passed via `-kernel` (same binary)

### Phase 4 – Appliance Layer (Layer 2)
- Minimal GNOME + virt-manager + libvirtd
- Pre-configured storage pools on btrfs subvolumes
- Firstboot setup script

### Phase 5 – Bootloader, LUKS & Storage
- Limine multi-version entries
- YubiKey LUKS2 enrollment script
- Full Btrfs subvolume layout

### Phase 6 – Rebuild Workflow & Tooling
- `build.sh` with versioned output
- `rollback.sh`
- Documentation

### Phase 7 – Testing & Hardening
- QEMU test matrix (traditional + shim)
- Hardware deployment
- Nested KVM validation
- Security review (minimal attack surface)

### Phase 8 – Matryoshka App Layers (Future)
- Self-extracting virtual-disk layers using your Python technique
- Trigger on VM ISO mount (optional)

---

## Technical Notes

### Kernel
- Single kernel binary used for both host and guest (passed via QEMU `-kernel`)
- Custom CachyOS fork with your exact 1000 Hz / PREEMPT / stripped config

### Pre-boot vs Guest Responsibilities
- **Pre-boot (Layer 1)**: LUKS unlock, layer mounting, overlay creation, starts minimal `libvirtd`, defines & launches the appliance VM from XML (full pre-boot control system)
- **Guest (Layer 2)**: Full GNOME + desktop experience, nested workload VMs, additional libvirt management if needed

**Note on libvirt in Layer 1**: On 3× NVMe RAID0 the increased size (~300-500 MB) is acceptable. This gives powerful XML-driven control of the hardware shim from the earliest boot stage.

---

## Getting Started (High Level)

1. Clone the repo
2. Enter `build/` and run `./scripts/build.sh --layer both`
3. Copy the two squashfs files to your target disk (inside LUKS + Btrfs)
4. Configure Limine with entries for current + previous versions
5. Boot and enjoy your minimal immutable shimmed hypervisor

Full instructions will be in `docs/BUILD.md` once Phase 6 is complete.

---

## Contributing

This is a personal research / appliance project.  
Issues and pull requests are welcome if they align with the "minimal, immutable, super-enhanced Cachy, VM-shimmed" philosophy.

---

## License

MIT (or your preferred license — to be decided)

---

**Status**: Early design + planning phase. Core architecture is solid. Implementation has begun with the dracut hook and build workflow.

**Last Updated**: 2026-07-06

---

*Built with obsession for minimalism, performance, and clean rollback.*
