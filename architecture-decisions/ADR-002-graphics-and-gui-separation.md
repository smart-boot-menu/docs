# ADR-002: Graphics & GUI Strategy — Shared Design System, Separate Renderers

- **Status:** Accepted (SBM 0.1 Baseline)
- **Date:** 2026-11-26
- **Supporting Research:** `research/hardware/gpu-and-graphics-acceleration.md`

## Context

`sbm-uefi` executes in pre-OS UEFI where only `EFI_GRAPHICS_OUTPUT_PROTOCOL` (GOP — linear CPU-writable framebuffer, zero GPU acceleration) exists. `sbm-core` executes in Linux with DRM/KMS, Mesa Vulkan/OpenGL, Wayland, XWayland (required for GTK3 GParted `1.8.1`), and software fallback (`llvmpipe`/`pixman`). We evaluated sharing a single GUI framework (specifically Slint `1.19.1`, which supports both `no_std` UEFI and Linux) versus separate renderers sharing a design system.

## Alternatives Considered

1. **Force Slint (`1.19.1`) across both `sbm-uefi` and `sbm-core`:**
   - *Pros:* Single `.slint` UI codebase.
   - *Cons:* Slint is licensed under `GPL-3.0-only OR LicenseRef-Slint-Royalty-free-2.0 OR Commercial`, conflicting with SBM's `MIT OR Apache-2.0` permissive license baseline for original Rust code; increases UEFI binary size and build complexity; still requires a Wayland + XWayland compositor in `sbm-core` to run GParted.
2. **Force `egui` / `vello_cpu` into UEFI:**
   - *Cons:* `egui` full rendering assumes `std`/tessellation pipelines; `vello_cpu 0.4.0` is promising (`MIT OR Apache-2.0`, `no_std`) but its API is still rapidly churning (`0.4.x`) and unnecessary for a crisp boot picker.
3. **Shared Design System + Purpose-Built Renderers (Selected):**
   - Share **design tokens, theme schemas, boot-entry view models, and iconography metadata** in `sbm-common`.
   - **`sbm-uefi`:** Lightweight custom `#![no_std]` + `alloc` 2D software renderer directly targeting GOP (double-buffered dirty-rect blitting, integer HiDPI scaling, `ab_glyph` or pre-baked font atlases, crisp PNG/QOI icons, plus serial/text fallback when GOP is absent).
   - **`sbm-core`:** Lightweight Wayland compositor (**`labwc`** + **XWayland** for GTK3 GParted + **`wayvnc`** for `noVNC` remote desktop) running an **Iced** (`MIT`, Rust) management shell with automatic `wgpu` GPU acceleration and `tiny-skia` CPU fallback (ensuring seamless rendering on Apple M4/M5 `simpledrm`, older HP laptops, and QEMU VMs).

## Decision

Do **not** force a shared GUI framework across UEFI and Linux. Share design tokens and state models via `sbm-common`, use a tiny custom `no_std` GOP software renderer in `sbm-uefi`, and use `labwc` (Wayland + XWayland) + `Iced` (with `wgpu` + `tiny-skia` fallback) in `sbm-core`.

## Open Experiment

- **EXP-GUI-01:** Prototype `sbm-uefi` GOP renderer on QEMU OVMF (1080p & 4K HiDPI) and measure first-frame latency (< 50 ms target) and EFI binary size (< 1.5 MB target).
- **EXP-GUI-02:** Verify `labwc` + XWayland + GParted `1.8.1` + `wayvnc` inside `sbm-core` under `virtio-gpu` and `llvmpipe` software fallback with ≤ 1 GiB VM RAM.
