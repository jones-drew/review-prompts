# RISC-V IOMMU Subsystem Details

Load `iommu.md` first for generic IOMMU framework knowledge. This file covers
`drivers/iommu/riscv/iommu.c`, `drivers/iommu/riscv/iommu.h`,
`drivers/iommu/riscv/iommu-bits.h`, and the RISC-V IOMMU hardware interface.
It does not cover interrupt remapping (irqbypass) — see `riscv-irqbypass.md`.

Baseline: `riscv/iommu-irqbypass-rfc-v3-rc5` (major redesign from rc4).

## Device Context Table (DDT) and Device Contexts

Writing device context fields without the correct capability guard corrupts
adjacent entries in the DDT, causing translation faults or silent wrong
translations for other devices.

`struct riscv_iommu_dc` has eight u64 fields. When
`RISCV_IOMMU_CAPABILITIES_MSI_FLAT` is **not** set, the hardware uses a
*base format* device context that is only 4×u64 (32 bytes).
`riscv_iommu_get_dc()` uses pointer arithmetic scaled by `(3 - base_format)`
to pack base-format entries at 32-byte stride. The last four fields —
`msiptp`, `msi_addr_mask`, `msi_addr_pattern`, `_reserved` — do not exist
in base format; writing them overwrites the next entry's `tc` field.

- **REPORT as bugs**: any write to `dc->msiptp`, `dc->msi_addr_mask`, or
  `dc->msi_addr_pattern` without first checking
  `iommu->caps & RISCV_IOMMU_CAPABILITIES_MSI_FLAT`.
- Writing zero to these fields is equally harmful — it still corrupts the
  adjacent entry's `tc` field, potentially clearing `DC_TC_V` and causing
  translation faults for other devices sharing the DDT page.
- `domain->msi_root` being NULL only prevents non-zero values from being
  placed in the local `dc` struct; it does not prevent
  `riscv_iommu_iodir_update()` from writing those zero values to hardware.

`riscv_iommu_iodir_update()` performs a two-pass update: first pass clears
`DC_TC_V` and issues `IODIR.INVAL_DDT` for previously-valid entries; second
pass writes the new context and re-sets `DC_TC_V` with a `dma_wmb()` barrier
before the final `WRITE_ONCE(dc->tc, tc)`. Any new field added to the write
loop must be guarded by the appropriate capability check.

The second pass now writes `iohgatp` (added in rc5):
```c
if (iommu->caps & RISCV_IOMMU_CAPABILITIES_MSI_FLAT) {
    WRITE_ONCE(dc->iohgatp, new_dc->iohgatp);
    WRITE_ONCE(dc->msiptp, new_dc->msiptp);
    WRITE_ONCE(dc->msi_addr_mask, new_dc->msi_addr_mask);
    WRITE_ONCE(dc->msi_addr_pattern, new_dc->msi_addr_pattern);
}
```

**REPORT as bugs**: code that writes `dc->iohgatp` without the `MSI_FLAT`
capability guard.

## riscv_iommu_domain and the Generic Page Table

`struct riscv_iommu_domain` uses a union to alias `struct iommu_domain` with
`struct pt_iommu_riscv_64`. `PT_IOMMU_CHECK_DOMAIN` is a `static_assert`
that verifies the two `iommu_domain` fields are at the same offset — this
must hold for `container_of` aliasing to be valid.

- `pt_iommu_riscv_64_hw_info(&domain->riscvpt, &info)` retrieves
  `info.fsc_iosatp_mode` (the DC.fsc.MODE value: 8=SV39, 9=SV48, 10=SV57)
  and `info.ppn` (top-level PPN). Use these to build `dc.fsc` and `dc.iohgatp`.
  Do not access `pgd_root` or `pgd_mode` — those fields do not exist.
- `pt_iommu_deinit(&domain->riscvpt.iommu)` frees all page table memory.
  Called from `riscv_iommu_free_paging_domain()`.
- `riscv_iommu_phys_to_ppn(pa)` / `riscv_iommu_ppn_to_phys(pn)` are in
  `iommu-bits.h`. They place/extract the PFN at bits 53:10 matching
  `RISCV_IOMMU_PPN_FIELD`. Round-trip is lossless for page-aligned addresses.

The domain struct no longer has `gstage_riscvpt`, `saved_pt_ops`, or
`gstage_pt_ops`. These were removed in the rc5 redesign. Do not reference them.

## PSCID and GSCID Allocators

MSI_FLAT + imsic_enabled() domains are g-stage domains: they allocate a **GSCID**
at domain creation time and leave `pscid` zero.

Non-MSI_FLAT (or MSI_FLAT without IMSIC) domains are s-stage domains: they
allocate a **PSCID** at domain creation time and leave `gscid` zero.

Both allocators use `DEFINE_IDA` with `ida_alloc_range()` starting at 1 (ID 0
is reserved).  `RISCV_IOMMU_MAX_PSCID` and `RISCV_IOMMU_MAX_GSCID` are both
65535.

## Two-Stage Address Translation: New Design (rc5)

### MSI_FLAT + imsic_enabled() domains: g-stage only

In rc5, MSI_FLAT domains no longer maintain a separate g-stage page table with
software-managed identity mappings.  Instead, the primary page table
(`riscvpt`) is placed directly in the **g-stage** (`iohgatp`), and the
s-stage (`fsc`) is set to BARE:

```c
dc.fsc = RISCV_IOMMU_FSC_BARE;
dc.iohgatp = FIELD_PREP(MODE, pt_info.fsc_iosatp_mode) |
             FIELD_PREP(GSCID, domain->gscid) |
             FIELD_PREP(PPN, pt_info.ppn);
```

DMA writes use the g-stage for translation (IOVA is treated as a GPA,
translated by the g-stage to an HPA). MSI writes to IMSIC addresses are
intercepted by the MSI table (`msiptp`) before the g-stage is consulted.

This eliminates the old `gstage_riscvpt` second page table, the
`gstage_install_ops()` wrapper, and the IMSIC s-stage identity mappings
(`riscv_iommu_ir_map_unmap_imsics()`). There is no longer a
`saved_pt_ops`/`gstage_pt_ops` indirection.

- The condition `if (domain->msi_root)` in `riscv_iommu_attach_paging_domain()`
  selects g-stage mode (MSI table set up) vs. s-stage mode.
- When `domain->msi_root == NULL`, the DC uses `dc.fsc = primary PT` and no
  `iohgatp` (s-stage mode, non-MSI_FLAT path).
- **REPORT as bugs**: code that sets both `dc.fsc` to a non-BARE mode and
  `dc.iohgatp` to a non-BARE mode in the same DC update — only one stage is
  active at a time.

### Non-MSI_FLAT domains: s-stage only

Non-MSI_FLAT (or MSI_FLAT without IMSIC) domains use the primary page table as
the s-stage (`dc.fsc`), with `iohgatp` left at BARE. The g-stage is not
active.

### SV*x4 mode widths

When MSI_FLAT is active, the primary page table is allocated as an SV*x4
g-stage, using the actual input address width of the SV*x4 mode:

| SV*x4 mode     | hw_max_vasz_lg2 | Input (GPA) width |
|----------------|-----------------|-------------------|
| Sv39x4         | 41              | 41 bits           |
| Sv48x4         | 50              | 50 bits           |
| Sv57x4         | 59              | 59 bits           |

For non-MSI_FLAT domains, SV* modes use hw_max_vasz_lg2 = 39/48/57.
`hw_max_oasz_lg2` is always 56 regardless of mode. The old cap of
`hw_max_oasz_lg2 = min(hw_max_vasz_lg2, 56)` for MSI_FLAT was removed.

**REPORT as bugs**: code that sets `hw_max_vasz_lg2 = 39/48/57` for an
SV*x4 g-stage — the actual input width for Sv39x4 is 41 bits, etc.

## Command Queue and Synchronisation

Sending commands without a subsequent `IOFENCE.C` leaves the hardware in an
undefined state with respect to when the commands take effect.

- `riscv_iommu_cmd_send(iommu, cmd)` enqueues a command to the hardware
  command queue. It does not wait for completion.
- `riscv_iommu_cmd_sync(iommu, timeout_us)` sends `IOFENCE.C` and busy-waits
  for the hardware to signal completion. All previously enqueued commands are
  guaranteed complete after this returns.
- `IOTINVAL.VMA` (opcode=1, func=0) invalidates s-stage TLB entries.
  `IOTINVAL.GVMA` (opcode=1, func=1) invalidates g-stage TLB entries.
  `IODIR.INVAL_DDT` (opcode=3, func=0) invalidates device context cache.
- IOTINVAL commands have `AV` (address valid), `GV` (GSCID valid), and
  `PSCV` (PSCID valid) bits. Setting `GV` scopes the invalidation to a
  specific guest domain; setting `AV` scopes it to a specific address.
- `riscv_iommu_iotlb_inval()` now branches on `domain->gscid`:
  - `domain->gscid != 0` → `IOTINVAL.GVMA` scoped by GSCID (MSI_FLAT domains)
  - `domain->gscid == 0` → `IOTINVAL.VMA` scoped by PSCID (non-MSI_FLAT domains)
  **REPORT as bugs**: implementations that always issue `IOTINVAL.VMA`
  regardless of `domain->gscid` — g-stage TLB entries are not invalidated by
  `IOTINVAL.VMA`.
- `riscv_iommu_iodir_iotinval()` must issue **both** `IOTINVAL.VMA` and
  `IOTINVAL.GVMA` (scoped to `DC.iohgatp.GSCID`) when `iohgatp.MODE !=
  BARE`, per spec section 6.3.1. The GVMA is issued inside the `else`
  branch (non-BARE path) immediately after the VMA. Omitting the GVMA
  leaves stale g-stage TLB entries after DC updates.
- **REPORT as bugs**: `riscv_iommu_iodir_iotinval()` implementations
  that only issue `IOTINVAL.VMA` when `iohgatp.MODE != BARE`.
- The QEMU IOMMU trace emits raw 64-bit command words. Decode IOTINVAL.GVMA:
  `opcode = dw0[6:0] == 1`, `func = dw0[9:7] == 1`, `GV = dw0[33]`,
  `GSCID = dw0[59:44]`. The string `GSCID=N` does not appear in the trace.

## riscv_iommu_bond and the bonds List

`struct riscv_iommu_bond` links a `riscv_iommu_domain` to its attached
devices. It is RCU-protected via `list_for_each_entry_rcu()` on
`domain->bonds`. An `smp_mb()` before `rcu_read_lock()` synchronizes with
`riscv_iommu_bond_link()`.

Functions that iterate `domain->bonds` to send commands to every unique IOMMU
device must deduplicate by tracking the previous IOMMU pointer — bonds for
devices sharing the same IOMMU are adjacent in the list.

## Probe and Release Symmetry

`riscv_iommu_probe_device()` allocates a `struct riscv_iommu_info` via
`kzalloc`, creates the IR irqdomain (if MSI_FLAT capable), and links the
device to the IOMMU. `riscv_iommu_release_device()` must free all of these
in reverse order. Any resource allocated in probe that is not freed in
release is a leak that accumulates across VFIO bind/unbind cycles.

## IOMMU_CAP_VIRT_MSI_ISOLATION

`riscv_iommu_capable()` returns `true` for `IOMMU_CAP_VIRT_MSI_ISOLATION`
when **both** `imsic_enabled()` and
`iommu->caps & RISCV_IOMMU_CAPABILITIES_MSI_FLAT` are true.  The IMSIC
check is required because MSI remapping is only meaningful when the platform
has IMSIC interrupt files to redirect to.

**REPORT as bugs**: implementations that check only `MSI_FLAT` without
`imsic_enabled()` — on platforms without IMSIC the IR domain is never
created and MSI remapping cannot function.

## MSI Table Ownership: Per-Domain, Not Per-Device

The MSI table (`msiptp`) is stored in `riscv_iommu_domain`, not per-device.
This is correct by design and should not be flagged as a bug.

The RISC-V IOMMU Device Context (spec section 2.1) is the single hardware
structure that references both the DMA page tables (`fsc`/`iohgatp`) and the
MSI table (`msiptp`). Since `iommu_domain` already owns the DMA page tables,
it is the natural owner of the MSI table too. Each device's DC gets its own
`msiptp` field written to point at the domain's shared table via
`riscv_iommu_iodir_update()` on attach.

This differs from Intel VT-d and AMD-Vi where interrupt remapping tables are
per-BDF structures independent of the DMA domain. In the RISC-V IOMMU the
MSI table is coupled to the DMA domain by design — the DC is the coupling
point.

**Do not report as a bug**: `domain->msi_root` being shared across all
devices attached to the same domain. This is architecturally correct.

## IMSIC MSI Interception: No Explicit S-Stage Mapping Needed

In rc5, the s-stage is BARE for MSI_FLAT domains. MSI writes from the device
go directly into the g-stage as GPAs. The IOMMU intercepts writes whose
address matches `msi_addr_pattern`/`msi_addr_mask` before the g-stage
translates them, redirecting to the MSI table (`msiptp`).

There is no longer an `riscv_iommu_ir_map_unmap_imsics()` function and no
explicit s-stage identity mapping for IMSIC addresses. The old
`IOMMU_RESV_RESERVED` / `IOMMU_RESV_DIRECT` distinction was:
- MSI_FLAT: `IOMMU_RESV_RESERVED` — prevented DMA allocation in the IMSIC
  aperture; s-stage IMSIC mappings were managed dynamically by the IR driver.
- Non-MSI_FLAT: `IOMMU_RESV_DIRECT` — installed host IMSIC identity mappings
  at domain setup.

In rc5, MSI_FLAT domains still use `IOMMU_RESV_RESERVED` to prevent DMA
allocation in the IMSIC address aperture. Non-MSI_FLAT domains still use
`IOMMU_RESV_DIRECT`.

**REPORT as bugs** (rc5+): any code that calls `iommu_map()` for IMSIC
addresses in the irqbypass path — there is no s-stage to map into for
MSI_FLAT domains.

## Quick Checks

- **`iommu->caps` vs `domain->msi_root`**: `iommu->caps` is the hardware
  capability register; `domain->msi_root` is the allocated MSI PTE table.
  The capability check must happen in `riscv_iommu_iodir_update()` using
  `iommu->caps`, not by checking `domain->msi_root` in the caller.
- **`riscv_iommu_get_dc()` pointer arithmetic**: the returned pointer is
  only valid for the fields present in the hardware format. For base-format
  hardware, only the first 4 u64 fields (`tc`, `fsc`, `ta`, `_reserved0`)
  are valid.
- **`RISCV_IOMMU_IOTINVAL_TIMEOUT`**: the busy-wait timeout for IOFENCE.C
  completion. If exceeded, the hardware is considered stuck and the driver
  logs an error. Do not reduce this timeout without hardware justification.
- **`ida_alloc_range()` error propagation**: both PSCID and GSCID allocators
  return -ENOMEM unconditionally on failure, discarding the actual error code
  from `ida_alloc_range()`. In practice -ENOMEM is the only realistic failure
  (IDA exhausted), but this is inconsistent with the rest of the kernel.
- **`domain->msi_root` as g-stage sentinel**: `riscv_iommu_attach_paging_domain()`
  uses `if (domain->msi_root)` to decide whether to configure the DC in
  g-stage mode (fsc=BARE, iohgatp=primary PT) or s-stage mode (fsc=primary
  PT, no iohgatp). `msi_root` is set by `riscv_iommu_ir_attach_paging_domain()`
  on first attach.
- **GSCID at creation time**: for MSI_FLAT + imsic_enabled() domains, GSCID
  is allocated in `riscv_iommu_alloc_paging_domain()`, not lazily on first
  attach. `domain->gscid > 0` means the domain is a g-stage domain.
- **Reserved region type for IMSIC**: non-MSI_FLAT uses
  `IOMMU_RESV_DIRECT` (installs host IMSIC identity mappings at domain
  setup). MSI_FLAT uses `IOMMU_RESV_RESERVED` (IOVA collision prevention
  only; no explicit IMSIC mapping is needed since s-stage is BARE).

## Error Path Discipline

In `riscv_iommu_attach_paging_domain()`, if `riscv_iommu_ir_attach_paging_domain()`
fails, the function returns before the bond is linked. No cleanup of
partially-initialised DC state is needed since the device was never
associated with this domain.

**REPORT as bugs**: `riscv_iommu_ir_attach_paging_domain()` error paths that
leave `domain->msi_root` allocated without a corresponding free in
`riscv_iommu_ir_free_paging_domain()`.
