# Edge ML OS

> A purpose-built, production-ready Linux distribution for Raspberry Pi + Hailo NPU edge AI devices.

## Vision

"The JetPack experience for the $150 hardware tier"

No purpose-built edge AI OS exists for Raspberry Pi + Hailo. This project fills that gap with:
- Zero-config Hailo NPU support
- Sub-2-second boot to inference
- Production-ready OTA updates
- Open source with optional commercial support

## Target Hardware

| Device | NPU | Performance | Cost |
|--------|-----|-------------|------|
| Raspberry Pi 5 + Hailo-8L | 13 TOPS | Good for most edge CV | ~$150 |
| Raspberry Pi 5 + Hailo-8 (AI HAT+) | 26 TOPS | High-performance inference | ~$200 |
| Raspberry Pi 4/CM4 + Hailo-8L | 13 TOPS | Production deployments | ~$130 |

## Architecture

```
┌─────────────────────────────────────────────────────┐
│              Fleet Management (hawkBit/RDFM)        │
└─────────────────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   OS Updates      Model Updates    App Updates
   (RAUC A/B)      (Delta .hef)    (Containers)
        │                │                │
        └────────────────┼────────────────┘
                         ▼
┌─────────────────────────────────────────────────────┐
│  Application Layer (Podman containers)              │
├─────────────────────────────────────────────────────┤
│  HailoRT (native) + Models (/opt/models/*.hef)      │
├─────────────────────────────────────────────────────┤
│  Minimal Yocto OS (read-only, dm-verity)            │
│  - PREEMPT_RT kernel 6.6+ (Pi fork)                 │
│  - OverlayFS for runtime writes                     │
│  - CMA=256M for NPU DMA                             │
├─────────────────────────────────────────────────────┤
│  Secure Boot (Pi native or UEFI)                    │
└─────────────────────────────────────────────────────┘
```

## Technical Decisions

### Build System: Yocto

**Why Yocto over Buildroot:**
- Official `meta-hailo` layer maintained by Hailo
- 8,400+ packages vs Buildroot's ~2,000
- Better for multi-variant products
- Mature SBOM and license compliance tooling
- Industry standard (automotive, medical, industrial)

**Mitigation for complexity:**
- Provide pre-built images for download
- Docker-based build environment
- Excellent documentation

### Update System: RAUC

**Why RAUC:**
- Smallest footprint (~512KB vs Mender's ~6.9MB)
- Mandatory cryptographic signing (security by design)
- Proven at scale (Steam Deck, Deutsche Bahn ICE trains)
- Works with hawkBit backend for fleet management

### Kernel Configuration

**Base:** Raspberry Pi kernel fork + PREEMPT_RT patches

**Key settings:**
```kconfig
CONFIG_PREEMPT_RT=y           # Real-time preemption (mainline 6.12+)
CONFIG_HZ=1000                # 1ms scheduling granularity
CONFIG_NO_HZ_FULL=y           # Tickless on isolated CPUs
CONFIG_CMA=y                  # Contiguous memory for NPU DMA
CONFIG_CMA_SIZE_MBYTES=256    # DMA buffer pool
CONFIG_KERNEL_LZ4=y           # Fast boot decompression
```

**Boot parameters:**
```
quiet loglevel=0 isolcpus=2-3 nohz_full=2-3 rcu_nocbs=2-3 cma=256M
```

### Partition Layout

```
/dev/mmcblk0p1  boot      FAT32    256MB   Kernel, DTB, initramfs
/dev/mmcblk0p2  rootfs_a  SquashFS 2GB     System A (dm-verity)
/dev/mmcblk0p3  rootfs_b  SquashFS 2GB     System B (dm-verity)
/dev/mmcblk0p4  data      ext4     rest    Models, containers, logs
```

## Hailo Integration

### Driver Requirements

- Use `hailo8` branch (NOT master) for Hailo-8/8L
- Kernel >= 6.6.31
- PCIe Gen3 mode for optimal performance
- Firmware: `/lib/firmware/hailo/hailo8_fw.bin`

### Licensing

| Component | License | Redistributable |
|-----------|---------|-----------------|
| libhailort | MIT | Yes |
| pyhailort | MIT | Yes |
| hailortcli | MIT | Yes |
| GStreamer plugin | LGPL 2.1 | Yes (dynamic link) |
| Firmware blob | Proprietary | First-boot download |

### Common Gotchas

1. **Version mismatch**: HailoRT and driver versions must match exactly
2. **Kernel symlinks**: Some distros missing `/lib/modules/$(uname -r)/build`
3. **Secure boot**: Driver is unsigned, requires disabling or self-signing
4. **PCIe speed**: Must enable Gen3 in config.txt for full performance

## Competitive Landscape

### What Exists

| Solution | Gap |
|----------|-----|
| Raspberry Pi OS | Desktop OS, not optimized for edge AI |
| balenaOS | No NPU support |
| JetPack | Expensive hardware ($200+ Jetson) |
| Ubuntu Core | No Hailo integration |

### Failed Projects (Lessons)

**Android Things (Google):**
- 90-second boot times
- Hardware requirements too high
- Google trust issues

**Key lessons:**
- Keep it simple and focused
- Boot time matters
- Long-term commitment essential
- Don't require smartphone-class hardware

## Proposed Repository Structure

```
edge-ml-os/
├── meta-edge-ml/                 # Yocto layer
│   ├── conf/
│   │   └── layer.conf
│   ├── recipes-bsp/
│   │   └── bootfiles/
│   ├── recipes-core/
│   │   ├── images/
│   │   │   └── edge-ml-image.bb
│   │   └── init/
│   ├── recipes-hailo/
│   │   ├── hailort/
│   │   └── hailo-firmware/
│   ├── recipes-support/
│   │   └── rauc/
│   └── wic/
│       └── edge-ml.wks
├── configs/
│   ├── rpi5-hailo8l.conf
│   ├── rpi5-hailo8.conf
│   └── rpi4-hailo8l.conf
├── models/                       # Reference models
│   └── yolov8n-hailo8l.hef
├── examples/
│   ├── object-detection/
│   └── image-classification/
├── scripts/
│   ├── build.sh
│   ├── flash.sh
│   └── setup-build-env.sh
├── docs/
│   ├── getting-started.md
│   ├── building.md
│   └── deployment.md
├── docker/
│   └── Dockerfile.build          # Reproducible build environment
├── Makefile
├── README.md
└── LICENSE                       # MIT or Apache 2.0
```

## Implementation Phases

### Phase 1: Minimal Viable OS
- [ ] Yocto build with meta-raspberrypi + meta-hailo
- [ ] Boot to shell with Hailo detected
- [ ] Basic inference demo working
- [ ] Pre-built image available for download

### Phase 2: Production Features
- [ ] RAUC A/B updates
- [ ] Read-only rootfs with overlay
- [ ] Sub-2-second boot
- [ ] Model update mechanism (separate from OS)

### Phase 3: Fleet Management
- [ ] hawkBit integration
- [ ] Remote shell access
- [ ] Telemetry and monitoring
- [ ] Canary deployment workflow

### Phase 4: Security Hardening
- [ ] dm-verity for rootfs integrity
- [ ] Secure boot (Pi 4/CM4 initially)
- [ ] Encrypted model storage
- [ ] Audit logging

## Use Cases

1. **Smart cameras** - Object detection, person tracking, license plate recognition
2. **Industrial IoT** - Anomaly detection, predictive maintenance, quality inspection
3. **Home automation** - Local AI for privacy (Frigate, Home Assistant)
4. **Retail** - Customer analytics, inventory monitoring
5. **Agriculture** - Crop monitoring, pest detection

## Resources

### Official Documentation
- [Raspberry Pi AI Kit](https://www.raspberrypi.com/documentation/accessories/ai-kit.html)
- [HailoRT GitHub](https://github.com/hailo-ai/hailort)
- [meta-hailo](https://github.com/hailo-ai/meta-hailo)
- [RAUC](https://rauc.io/)
- [Eclipse hawkBit](https://eclipse.dev/hawkbit/)

### Community
- [Hailo Community Forum](https://community.hailo.ai/)
- [Raspberry Pi Forums](https://forums.raspberrypi.com/)

### Similar Projects
- [Frigate NVR](https://docs.frigate.video/) - Uses Hailo for detection
- [balenaOS](https://www.balena.io/os) - Container-focused IoT OS
- [Torizon](https://www.toradex.com/torizon) - Industrial embedded Linux

---

*Research compiled: January 2026*
*Status: Idea / Planning*
