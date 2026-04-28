# RISC-V IOMMU Subsystem Details

Load `iommu.md` first for generic IOMMU framework knowledge. This file covers
`drivers/iommu/riscv/iommu.c`, `drivers/iommu/riscv/iommu.h`,
`drivers/iommu/riscv/iommu-bits.h`, and the RISC-V IOMMU hardware interface.
It does not cover interrupt remapping (irqbypass) — see `riscv-irqbypass.md`.

Baseline: `riscv/iommu-irqbypass-rfc-v3-rc4`.

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

## riscv_iommu_domain and the Generic Page Table

`struct riscv_iommu_domain` uses a union to alias `struct iommu_domain` with
`struct pt_iommu_riscv_64`. `PT_IOMMU_CHECK_DOMAIN` is a `static_assert`
that verifies the two `iommu_domain` fields are at the same offset — this
must hold for `container_of` aliasing to be valid.

- `pt_iommu_riscv_64_hw_info(&domain->riscvpt, &info)` retrieves
  `info.fsc_iosatp_mode` (the DC.fsc.MODE value: 8=SV39, 9=SV48, 10=SV57)
  and `info.ppn` (top-level PPN). Use these to build `dc.fsc` and `dc.ta`.
  Do not access `pgd_root` or `pgd_mode` — those fields do not exist.
- `pt_iommu_deinit(&domain->riscvpt.iommu)` frees all page table memory.
  Called from `riscv_iommu_free_paging_domain()`.
- `riscv_iommu_phys_to_ppn(pa)` / `riscv_iommu_ppn_to_phys(pn)` are in
  `iommu-bits.h`. They place/extract the PFN at bits 53:10 matching
  `RISCV_IOMMU_PPN_FIELD`. Round-trip is lossless for page-aligned addresses.

## PSCID and GSCID Allocators

Exhausting the PSCID or GSCID space returns -ENOMEM from `ida_alloc_range()`,
which is propagated as -ENOMEM from domain allocation. Both allocators use
`DEFINE_IDA` with `ida_alloc_range()` starting at 1 (ID 0 is reserved).
`RISCV_IOMMU_MAX_PSCID` and `RISCV_IOMMU_MAX_GSCID` are both 65535.

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

## Two-Stage Address Translation Semantics

The RISC-V IOMMU spec (and the priv spec hypervisor extension) define two
stages of address translation:

- **First stage (s-stage / `iosatp`)**: translates device IOVAs (virtual
  addresses) to Guest Physical Addresses (GPAs).  When the guest OS is not
  involved this is effectively IOVA → host physical address (HPA).
- **Second stage (g-stage / `iohgatp`)**: translates GPAs to Supervisor
  Physical Addresses (SPAs, i.e. host physical addresses).

When MSI_FLAT is active, `iohgatp.MODE` must be non-BARE.  DMA addresses
that pass through the s-stage produce a GPA; the g-stage then translates
that GPA to an SPA.  For the non-virtualised DMA case the driver installs
identity mappings (GPA == SPA) in the g-stage so that s-stage output
addresses pass through unchanged.

The g-stage translation modes are the SV*x4 variants defined by the
hypervisor extension.  Each SV*x4 mode accepts input addresses that are
2 bits wider than the corresponding SV* virtual address mode:

| `iohgatp.MODE` | Input (GPA) width | Corresponding s-stage mode |
|----------------|-------------------|---------------------------|
| Sv39x4 (8)     | 41 bits           | Sv39                      |
| Sv48x4 (9)     | 50 bits           | Sv48                      |
| Sv57x4 (10)    | 59 bits           | Sv57                      |

The extra 2 bits exist so that a hypervisor can virtualise a machine whose
physical address space is up to 4× larger than its virtual address space.
The root page table for SV*x4 is 16 KiB (2048 entries, 4 contiguous pages)
rather than the usual 4 KiB (512 entries).

The `iohgatp.MODE` encoding values (8/9/10) are numerically identical to
the `iosatp.MODE` encoding values for Sv39/Sv48/Sv57, so the same integer
serves both fields without translation.

## G-Stage Page Table Infrastructure

`riscv_iommu_gstage_alloc()` initialises `domain->gstage_riscvpt` using
`pt_iommu_riscv_64` (without `PT_FEAT_SIGN_EXTEND` — g-stage inputs are
unsigned host physical addresses).  A GSCID is allocated from the IDA.
`riscv_iommu_gstage_free()` calls `pt_iommu_deinit()` and releases the GSCID.
`gstage_alloc` is called from `riscv_iommu_attach_paging_domain()` on first
attach when MSI_FLAT is active.  The sentinel is
`domain->gstage_riscvpt.iommu.ops == NULL`.

`riscv_iommu_gstage_install_ops()` wires the g-stage into every s-stage
mapping.  It saves `domain->riscvpt.iommu.ops` into `domain->saved_pt_ops`,
copies the ops struct into `domain->gstage_pt_ops`, overrides `map_range`
and `unmap_range` with wrappers that call the saved ops then add/remove a
matching g-stage identity mapping (paddr → paddr), and installs
`&domain->gstage_pt_ops` as the new ops pointer.  Called once per domain on
first attach, guarded by `domain->saved_pt_ops == NULL`.

Use `pt_iommu_riscv_64_hw_info(&domain->gstage_riscvpt, &info)` to read
`info.fsc_iosatp_mode` and `info.ppn` for building `dc.iohgatp`.  Do not
access `domain->gstage_mode` or `domain->gstage_root` — those fields do not
exist.

**PT_FEAT_SIGN_EXTEND must not be set on the g-stage.**  The g-stage input
is a host physical address (the s-stage output), which is unsigned.
`pt_check_range` with `PT_FEAT_SIGN_EXTEND` and `max_vasz_lg2 = N` requires
bits 63:N to equal bit N-1 (sign extension), which rejects physical addresses
in the upper half of the range [2^(N-1), 2^N).  Without sign extension,
`pt_check_range` requires only `addr >> N == 0`, giving a clean unsigned range
[0, 2^N).  The g-stage is therefore initialised with only `PT_FEAT_FLUSH_RANGE`.

**The s-stage `hw_max_oasz_lg2` must be capped to the g-stage
`hw_max_vasz_lg2` when MSI_FLAT is active.**  The g-stage only covers
[0, 2^hw_max_vasz_lg2); if the s-stage can produce a wider output address
the g-stage `iommu_map()` call returns `-ERANGE` and the DMA mapping fails.
`riscv_iommu_alloc_paging_domain()` sets `hw_max_oasz_lg2 =
hw_max_vasz_lg2` (39/48/57) when `RISCV_IOMMU_CAPABILITIES_MSI_FLAT` is
set, and 56 otherwise.  This limits DMA to physical addresses below
2^hw_max_vasz_lg2 when MSI_FLAT is active: 512 GB for Sv39x4, 256 TB for
Sv48x4.  Full coverage up to the hardware's 41-bit / 50-bit GPA limit
would require implementing the SV*x4 wider root page table.

**REPORT as bugs**: `iommu_map(&domain->domain, ...)` calls in code that
intends to update the g-stage DMA identity table directly — these map into
the s-stage and have no effect on the g-stage.  G-stage DMA identity mappings
are installed automatically by the `gstage_install_ops()` wrapper on every
s-stage `iommu_map()` call; never call `iommu_map` expecting a g-stage effect.

Note: IMSIC identity mappings for MSI table lookup are correctly placed in
the **s-stage** via `iommu_map(&domain->domain, addr, addr, ...)` by
`riscv_iommu_ir_map_imsics()`.  This is intentional and correct — the s-stage
must not fault on MSI writes before the hardware consults the MSI table.  Do
not move these to the g-stage.

## unmap_range and g-stage Ordering

Resolving the physical address after the s-stage unmap returns 0 and leaves
the g-stage entry pointing at freed memory.

In `riscv_iommu_gstage_unmap_range()`, the physical address must be resolved
via `pt_iommu_riscv_64_iova_to_phys()` on `&domain->domain` **before** calling the saved
`unmap_range`, because after the s-stage unmap the mapping no longer exists.

```c
// CORRECT
paddr = pt_iommu_riscv_64_iova_to_phys(&domain->domain, iova);
unmapped = saved_ops->unmap_range(...);
if (unmapped && paddr)
    riscv_iommu_gstage_unmap(domain, paddr, unmapped);

// WRONG
unmapped = saved_ops->unmap_range(...);
paddr = pt_iommu_riscv_64_iova_to_phys(&domain->domain, iova); // returns 0
```

The current implementation resolves a single paddr at the start of the range.
This is correct only for contiguous physical mappings. Scatter-gather unmaps
spanning multiple discontiguous physical regions will use the wrong paddr for
the tail — a known limitation deferred because the DMA API typically unmaps
ranges that were mapped as a single contiguous call.

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

## IMSIC Identity Mappings: S-Stage, Not G-Stage

IMSIC addresses must be identity-mapped in the **s-stage** so the IOMMU can
match them against `msi_addr_pattern`/`msi_addr_mask` and redirect them to
the MSI table.  Without s-stage mappings the s-stage translation faults
before the MSI table is ever consulted.

`riscv_iommu_ir_map_imsics()` calls `iommu_map(&domain->domain, addr, addr,
...)` — this maps into the s-stage (`domain->domain`), not the g-stage.

**REPORT as bugs**: code that calls `riscv_iommu_gstage_map()` for IMSIC
addresses — that would map into the g-stage and have no effect on MSI
lookup.  The g-stage identity mappings for DMA are installed automatically
by the `gstage_install_ops()` wrapper on every `iommu_map()` call; IMSIC
addresses are handled separately via the explicit s-stage map.

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
- **`domain->saved_pt_ops` as install sentinel**: `gstage_install_ops()` is
  called only when `domain->saved_pt_ops == NULL`. After installation,
  `saved_pt_ops` points to the original ops and `riscvpt.iommu.ops` points
  to `&domain->gstage_pt_ops`. `gstage_free()` restores the original ops
  pointer before freeing the g-stage table.
- **`gstage_alloc` sentinel**: `domain->gstage_riscvpt.iommu.ops == NULL`
  means the g-stage has not been allocated. `gstage_alloc()` is idempotent
  and returns 0 immediately if already allocated.
- **Reserved region type for IMSIC**: non-MSI_FLAT uses
  `IOMMU_RESV_DIRECT` (installs host IMSIC identity mappings at domain
  setup; correct because there is no guest involvement). MSI_FLAT uses
  `IOMMU_RESV_RESERVED` (IOVA collision prevention only; s-stage IMSIC
  mappings are managed dynamically because guest IMSIC GPAs may differ
  from host IMSIC physical addresses). See `iommu.md` for full detail
  on reserved region types.

## Error Path Discipline

Forgetting to sync the IOTLB gather after an unmap in an error path leaves
stale IOTLB entries that can cause devices to access freed memory.

In `riscv_iommu_gstage_map_range()`, the g-stage `map_range` failure path
calls `domain->saved_pt_ops->unmap_range(...)` to undo the s-stage mapping
and passes an `iommu_iotlb_gather` to collect the range.  The gather must be
synced with `iommu_tlb_sync(&domain->domain, &gather)` after the unmap;
without this call, the IOTLB is not invalidated for the rolled-back range.

**REPORT as bugs**: `unmap_range()` calls in error paths that initialise an
`iommu_iotlb_gather` but never call `iommu_tlb_sync()` on it.
