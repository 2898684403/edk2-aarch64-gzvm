# ADR-0001: GZVM EDK2 Firmware Architecture

**Status:** Accepted  
**Date:** 2026-06-15  
**Context:** GZVM is a Type-1 hypervisor on AArch64. Protected VMs have guest RAM
inaccessible to the host. DMA requires a shared bounce buffer pool.

## Decision

1. **PrePi (SEC phase)** runs from RAM (not flash). Firmware is loaded by the
   GZVM loader at 0x80000000. An arm64 Linux boot protocol header in the FDF
   allows the loader to identify and boot the image.

2. **Shared DMA pool** is discovered from the DTB's `reserved-memory` node with
   compatible `"restricted-dma-pool"`. The pool is excluded from EDK2's general
   memory allocator and managed solely by the bitmap allocator in
   `GzvmIoMmuDxe` via `EDKII_IOMMU_PROTOCOL`.

3. **DTB is the source of truth** for system topology (memory, GIC, timer, UART,
   PCI, etc.). ACPI is disabled (PcdForceNoAcpi=TRUE) but DynamicTablesPkg can
   still generate tables for Windows use cases.

4. **Firmware image is relocatable** — the arm64 boot protocol passes the DTB
   in x0 and the image base in x1. The PrePi entry point performs PE/COFF
   self-relocation and patches PcdFdBaseAddress/PcdFvBaseAddress accordingly.

5. **No SMM/SPM** — GZVM uses a simpler trap-and-emulate model. Variable
   storage is in-RAM (EmuVariableFvb) or optionally via uefi-vars-sysbus
   for persistent storage with QEMU_PV_VARS.

## Consequences

- DTB parsing is replicated across PrePi (FindMemnode), PEI constructor
  (QemuVirtMemInfoPeiLibConstructor), DXE MMU setup (QemuVirtMemInfoLib),
  and GzvmIoMmuDxe entry point — this is intentional to avoid cross-phase
  dependencies but requires careful maintenance to keep parsers in sync.
- FD image overlaps with DRAM (PcdFdBaseAddress == PcdSystemMemoryBase),
  which is unusual for EDK2. ASSERTs for non-overlap have been removed.
