# IOMMU Subsystem Details

Covers `drivers/iommu/`, `include/linux/iommu.h`, `drivers/iommu/generic_pt/`,
and the core IOMMU framework used by all IOMMU drivers.

## Domain Lifecycle and Attachment

Attaching a device to the wrong domain type, or failing to invalidate the
device context after domain changes, causes DMA faults, silent data corruption,
or use-after-free when the domain is freed while still attached.

- `struct iommu_domain` is the central abstraction. Drivers embed it in a
  larger struct and recover it via `container_of`. The `ops` pointer selects
  the domain's operation set (`struct iommu_domain_ops`).
- Domain types: `IOMMU_DOMAIN_BLOCKED` (deny all), `IOMMU_DOMAIN_IDENTITY`
  (passthrough), `IOMMU_DOMAIN_UNMANAGED` (driver-managed), `IOMMU_DOMAIN_DMA`
  (kernel DMA API managed), `IOMMU_DOMAIN_DMA_FQ` (deferred flush variant).
- `domain_alloc_paging()` / `domain_alloc_paging_flags()` are the preferred
  allocation paths for new drivers. The legacy `domain_alloc()` callback
  should not be used in new code.
- `attach_dev()` in `struct iommu_domain_ops` wires a device to a domain.
  The driver must update the hardware device context (DDT entry, stream table
  entry, etc.) atomically — the device may be issuing DMA during the switch.
- `probe_device()` returns a `struct iommu_device *` and sets up
  `dev->iommu` (a `struct dev_iommu`). `release_device()` tears it down.
  Both must be symmetric: every resource allocated in `probe_device()` must
  be freed in `release_device()`.
- `iommu_group` groups devices that share a translation unit. On most
  hardware each device is in its own group; PCIe ACS determines grouping.
  VFIO requires all devices in a group to be assigned together.

**REPORT as bugs**: `attach_dev()` implementations that do not issue a
device-context invalidation command after writing the new context to hardware.

## iommu_ops vs iommu_domain_ops

Confusing which ops struct to implement causes missing functionality or
crashes when the core calls an unimplemented callback.

- `struct iommu_ops` — driver-level ops registered via `iommu_device_register()`.
  Covers device probing, group assignment, capability queries, and domain
  allocation. One instance per IOMMU hardware device.
- `struct iommu_domain_ops` — domain-level ops embedded in the domain.
  Covers `attach_dev`, `map_pages`, `unmap_pages`, `flush_iotlb_all`,
  `iotlb_sync_map`, `iotlb_sync`, `iova_to_phys`. One instance per domain
  type (paging, blocking, identity).
- `iommu_ops.default_domain_ops` points to the ops used for the default
  paging domain. `iommu_ops.identity_domain` and `iommu_ops.blocked_domain`
  are static domain instances for those types.
- `iommu_ops.probe_device` must call `iommu_device_link()` to associate the
  device with the `struct iommu_device` registered by the driver.

## IOTLB Flush Discipline

Failing to flush the IOTLB after unmapping allows stale translations to
persist, causing devices to access freed memory.

- `flush_iotlb_all()` — synchronous full flush of all cached translations
  for the domain. Expensive; use only when necessary (e.g., domain teardown).
- `iotlb_sync_map()` — called after `map_pages()` to make new mappings
  visible to hardware. Some hardware requires an explicit sync; others make
  mappings visible on write.
- `iotlb_sync()` — flushes the ranges accumulated in `struct iommu_iotlb_gather`
  and frees the gather's page list. Called after one or more `unmap_pages()`
  calls.
- `struct iommu_iotlb_gather` accumulates unmap ranges for batched flushing.
  `iommu_iotlb_gather_add_range()` extends the range; `iommu_iotlb_sync()`
  flushes and resets it. In `DMA_FQ` mode the gather is queued and flushed
  lazily — drivers must handle `iommu_iotlb_gather_queued()` returning true.
- **REPORT as bugs**: `unmap_pages()` implementations that flush the IOTLB
  inline instead of accumulating into the gather — this defeats batching.
- **REPORT as bugs**: code that frees pages from an unmap before
  `iotlb_sync()` has been called — the device may still be reading them.

## map_pages / unmap_pages Contract

Incorrect size or alignment handling in `map_pages` / `unmap_pages` causes
partial mappings, alignment faults, or silent wrong-size translations.

- `map_pages(domain, iova, paddr, pgsize, pgcount, prot, gfp, mapped)`
  maps `pgcount` pages of size `pgsize` starting at `iova` → `paddr`.
  `pgsize` must be a power of two and supported by the hardware.
  `*mapped` is set to the number of bytes actually mapped on success.
- `unmap_pages(domain, iova, pgsize, pgcount, gather)` unmaps the range
  and returns the number of bytes unmapped. The return value must equal
  `pgsize * pgcount` for a fully successful unmap.
- Both functions must handle `pgsize` values that are not the hardware's
  minimum page size — the driver maps/unmaps multiple hardware pages per call.
- `iova_to_phys(domain, iova)` must return 0 for unmapped addresses, not
  a stale translation. After `unmap_pages()` + `iotlb_sync()`, the mapping
  must no longer be visible via `iova_to_phys()`.

## Generic Page Table Framework (drivers/iommu/generic_pt/)

Using the generic PT framework incorrectly — wrong feature flags, wrong
format, or calling format-specific functions on the wrong table type —
causes silent wrong mappings or kernel crashes.

- `struct pt_iommu` is the base type for all generic PT instances. Format-
  specific types (e.g., `struct pt_iommu_riscv_64`) embed it.
- `PT_IOMMU_CHECK_DOMAIN(outer_struct, pt_iommu_field, domain_field)` is a
  `static_assert` that verifies the `iommu_domain` embedded in the PT is at
  the same offset as the `iommu_domain` in the outer struct. This must pass
  for `container_of` aliasing to be valid.
- Feature flags (`PT_FEAT_*`) are set at init time via `pt_common.features`.
  Key flags:
  - `PT_FEAT_FLUSH_RANGE` — hardware supports range-based TLB invalidation.
  - `PT_FEAT_FLUSH_RANGE_NO_GAPS` — range flush covers all addresses in the
    range, not just mapped ones.
  - `PT_FEAT_DMA_INCOHERENT` — page table memory is not DMA-coherent; the
    framework calls `iommu_pages_flush_incoherent()` after writes.
  - `PT_FEAT_SIGN_EXTEND` — virtual addresses are sign-extended (used for
    RISC-V Sv39/48/57).
- `pt_iommu_riscv_64_init(table, cfg, gfp)` allocates and initialises a
  RISC-V 64-bit IOMMU page table. `cfg.common.hw_max_vasz_lg2` selects
  the address space size (39, 48, or 57 bits).
- `pt_iommu_riscv_64_hw_info(table, info)` fills `fsc_iosatp_mode` (the
  DC.fsc.MODE value) and `ppn` (top-level PPN). Use this to build device
  context entries — do not access `pgd_root` or `pgd_mode` directly.
- `pt_iommu_deinit(&domain->riscvpt.iommu)` frees all page table memory.
- `IOMMU_PT_DOMAIN_OPS(riscv_64)` expands to `.iova_to_phys =
  &pt_iommu_riscv_64_iova_to_phys` for use in `iommu_domain_ops`.
- The generic PT's `map_range` / `unmap_range` are called directly from
  `iommu_map_nosync()` when `domain->is_iommupt` is set, bypassing
  `ops->map_pages`. There is no per-mapping driver callback in
  `pt_iommu_driver_ops` — drivers that need to intercept every mapping
  (e.g. to maintain a shadow g-stage table) must replace the `ops` pointer
  after init. The RISC-V IOMMU driver does this via
  `riscv_iommu_gstage_install_ops()`: it saves the original
  `domain->riscvpt.iommu.ops` in `domain->saved_pt_ops`, copies the ops
  struct into `domain->gstage_pt_ops`, overrides `map_range` and
  `unmap_range` with wrappers that also update the g-stage identity table,
  then installs `&domain->gstage_pt_ops` as the new ops pointer.

**REPORT as bugs**: code that reads `pt_iommu_riscv_64.pgd_root` or
`.pgd_mode` directly — these fields do not exist in the v3 driver. Use
`pt_iommu_riscv_64_hw_info()` instead.

## iommu_fwspec and Device Identity

Incorrect use of `iommu_fwspec` causes wrong device IDs to be programmed
into the DDT, routing DMA from one device through another device's domain.

- `struct iommu_fwspec` (accessed via `dev_iommu_fwspec_get(dev)`) holds
  the list of stream IDs / device IDs for a device as parsed from firmware
  (DT or ACPI).
- `fwspec->num_ids` is the count; `fwspec->ids[]` are the IDs. A device
  may have multiple IDs (e.g., a PCIe function with multiple requester IDs).
- `iommu_fwspec_add_ids()` appends IDs; called from `of_xlate()` or
  `acpi_iommu_fwspec_init()`.
- Drivers must iterate all `fwspec->ids` when programming device contexts —
  missing an ID leaves that stream ID in the blocked or identity domain.

## iommu_pages Allocation

Using the wrong allocator for IOMMU page table memory causes DMA coherency
failures on non-coherent hardware or incorrect physical addresses.

- `iommu_alloc_pages_node_sz(node, gfp, size)` — allocates physically
  contiguous pages for page table use. Returns a kernel virtual address.
  Use this, not `kmalloc`, for page table nodes.
- `iommu_free_pages(ptr)` — frees memory allocated by `iommu_alloc_pages_*`.
  Handles NULL safely.
- `iommu_pages_flush_incoherent()` — flushes CPU caches for non-coherent
  hardware. Only needed when `PT_FEAT_DMA_INCOHERENT` is set.

## Reserved Regions and the DMA Cookie

### The Cookie

A "cookie" is a blob of private bookkeeping state that the DMA-API layer
(`dma-iommu.c`) attaches to an `iommu_domain` after the driver creates it.
The `iommu_domain` struct has a `cookie_type` enum and two union pointers
(`iova_cookie` / `msi_cookie`). The IOMMU driver never touches the cookie —
it is purely the DMA layer's scratch space.

Two cookie flavours are relevant to MSI handling:

- **`IOMMU_COOKIE_DMA_IOVA`** — created by `iommu_get_dma_cookie()`,
  called automatically when a DMA paging domain is allocated. Contains an
  `iova_domain` (the IOVA allocator), a `msi_page_list`, and a flush queue.
  This is what every normal paging domain gets when `IOMMU_DMA` is enabled.

- **`IOMMU_COOKIE_DMA_MSI`** — created by `iommu_get_msi_cookie()`, used
  by drivers that manage their own IOVA allocation (e.g. VFIO type1
  unmanaged domains) but still want MSI compose-time machinery. Contains
  only a base IOVA address and a `msi_page_list`.

The `msi_page_list` in both cookies is a list of `iommu_dma_msi_page`
structs, each recording a `phys` → `iova` mapping for an MSI doorbell page.
It is the lookup table that `iommu_dma_sw_msi()` consults at MSI compose
time to find (or create) the IOVA a device should write to.

### The MSI Compose-Time Flow

When the kernel programs a device's MSI target address, the irqchip driver
calls `iommu_dma_prepare_msi(desc, msi_addr)`. This calls
`iommu_dma_sw_msi()`, which calls `iommu_dma_get_msi_page()`. That function
looks up `msi_addr` in the domain's `msi_page_list`. If found, it returns
the cached entry. If not found, it allocates a fresh IOVA from the IOVA
allocator, calls `iommu_map(domain, iova, msi_addr, ...)` to install an
s-stage mapping, and caches the result.

The resulting IOVA is stored in the `msi_desc` via
`msi_desc_set_iommu_msi_iova()`. Later, when the irqchip driver calls
`msi_msg_set_addr(desc, msg, msi_addr)` to build the MSI message,
`msi_msg_set_addr()` checks whether `iommu_msi_iova` is set on the
descriptor. If it is, it uses the IOVA instead of the physical address as
the MSI target. The device is therefore programmed to write to the IOVA,
which the IOMMU translates through the s-stage to the physical MSI doorbell.

**Critical**: this entire flow only works if the irqchip driver calls
`iommu_dma_prepare_msi()` and uses `msi_msg_set_addr()`. Irqchip drivers
that write `msg->address_hi/lo` directly (bypassing `msi_msg_set_addr()`)
will always program the device with the physical address, regardless of any
IOVA remapping the DMA layer has set up.

### Reserved Region Types

`get_resv_regions()` in `struct iommu_ops` returns a list of
`struct iommu_resv_region` entries. The type field controls how each region
is handled by both the DMA layer and iommufd.

**`IOMMU_RESV_DIRECT`**

Handled by `iommu_map_resv_regions()` (called from `iommu_setup_dma_ops()`),
not by `iommu_dma_init_domain()`. Installs identity mappings (phys == iova)
in the domain's page table for the specified range at domain setup time.
Also reserves the IOVA range in the allocator.

Use when: the device always writes to the physical address and the IOMMU
must pass it through unchanged. No irqchip changes needed. Correct for
non-MSI_FLAT RISC-V IMSIC: the IMSIC irqchip programs the physical address
directly and does not call `iommu_dma_prepare_msi()`.

**`IOMMU_RESV_RESERVED`**

At `iommu_dma_init_domain()` time: `reserve_iova()` — marks the range as
reserved in the IOVA allocator. That is all. No mapping, no `msi_page_list`
pre-population, no compose-time machinery.

In iommufd: `iopt_reserve_iova()` — blocks userspace from mapping the range
via `ioctl`. Kernel-internal `iommu_map()` calls bypass iopt entirely and
are unaffected.

Use when: IOVA collision prevention is needed but mappings are managed
dynamically by the driver (e.g. MSI_FLAT RISC-V where s-stage IMSIC
mappings are installed at attach time and guest IMSIC mappings at
`irq_set_vcpu_affinity` time with potentially different topologies).

**`IOMMU_RESV_MSI`**

At `iommu_dma_init_domain()` time:
1. `reserve_iova()` — reserves the physical address range in the IOVA
   allocator
2. `cookie_init_hw_msi_region()` — pre-populates `msi_page_list` with
   identity entries (`phys == iova`) for every page in the region

At MSI compose time: `iommu_dma_get_msi_page()` finds the pre-populated
identity entry and calls `msi_desc_set_iommu_msi_iova()`. No new `iommu_map()`
is issued by the DMA layer.

Use when: the IOMMU has a hardware MSI table that intercepts writes to
specific physical addresses. The device writes to `iova == phys`; the
hardware MSI table does the actual remapping. The driver must separately
install s-stage identity mappings so the write reaches the MSI table lookup.
Requires the irqchip to call `iommu_dma_prepare_msi()` and use
`msi_msg_set_addr()`.

**`IOMMU_RESV_SW_MSI`**

At `iommu_dma_init_domain()` time: **completely skipped** (`continue`). No
IOVA reservation, no `msi_page_list` pre-population.

At MSI compose time: `iommu_dma_get_msi_page()` finds no pre-existing entry,
allocates a fresh IOVA, calls `iommu_map(domain, iova, msi_addr, ...)` to
install an s-stage mapping, and programs the device with the IOVA.

Use when: there is no hardware MSI table and the IOMMU translates MSI writes
through the normal s-stage page table (e.g. ARM SMMUv3 with GIC ITS). The
irqchip must call `iommu_dma_prepare_msi()` and use `msi_msg_set_addr()`.
ARM SMMUv3 uses a fixed IOVA window (`MSI_IOVA_BASE` / `MSI_IOVA_LENGTH`)
for this purpose.

**`IOMMU_RESV_DIRECT_RELAXABLE`**

Like `IOMMU_RESV_DIRECT` but the identity mapping may be relaxed (removed)
when the device is assigned to a VM. Used for regions that need identity
mapping for the host but can be remapped for guests.

### Choosing the Right Type: Decision Guide

| Scenario | irqchip uses msi_msg_set_addr? | Hardware MSI table? | Guest involvement? | Correct type |
|---|---|---|---|---|
| Non-MSI_FLAT RISC-V IMSIC | No (writes phys directly) | No | No | `IOMMU_RESV_DIRECT` |
| MSI_FLAT RISC-V, host | No (writes phys directly) | Yes (msiptp) | No | `IOMMU_RESV_RESERVED` + driver maps |
| MSI_FLAT RISC-V, guest | No (writes GPA directly) | Yes (msiptp) | Yes (GPA ≠ HPA) | `IOMMU_RESV_RESERVED` + driver maps |
| ARM SMMUv3 + GIC ITS | Yes | No | No | `IOMMU_RESV_SW_MSI` |

**Why not `IOMMU_RESV_DIRECT` for MSI_FLAT?** Guest IMSIC GPAs may differ
from host IMSIC physical addresses (VMM-dependent). Pre-installing host
IMSIC identity mappings would corrupt guest DMA mappings if the guest IMSIC
GPAs happen to overlap with other guest DMA IOVAs. All s-stage IMSIC
mappings must be managed dynamically by the driver.

**Why not `IOMMU_RESV_MSI` for RISC-V?** The RISC-V IMSIC irqchip
(`irq-riscv-imsic-platform.c`) writes `msg->address_hi/lo` directly without
calling `iommu_dma_prepare_msi()` or `msi_msg_set_addr()`. The
`msi_page_list` pre-population done by `IOMMU_RESV_MSI` is therefore unused
and the compose-time IOVA override never takes effect.

**Why not `IOMMU_RESV_SW_MSI` for RISC-V?** Same reason — the IMSIC irqchip
does not call `iommu_dma_prepare_msi()`, so the SW_MSI compose-time
`iommu_map()` call never happens and the device is still programmed with the
physical address, which has no s-stage mapping.

**`iommu_map()` bypasses iopt**: kernel-internal `iommu_map()` calls go
directly to the page table via `pt->ops->map_range()` or
`__iommu_map_domain_pgtbl()`. They do not consult the iommufd `io_pagetable`
and cannot be blocked by `IOMMU_RESV_RESERVED` iopt reservations. This is
intentional — driver-internal mappings (e.g. IMSIC identity maps installed
at attach time) must not be blocked by userspace-facing IOVA reservations.

## Quick Checks

- **`domain->ops` vs `iommu_ops`**: `domain->ops` is the domain-level ops
  (`struct iommu_domain_ops *`); `iommu_ops` is the driver-level ops
  (`struct iommu_ops`). They are different structs with different callbacks.
- **`iommu_group_get_iommudata()` / `iommu_group_set_iommudata()`**: used
  to attach driver-private data to a group. The data pointer is not
  reference-counted; the driver must ensure it outlives the group.
- **`iommu_device_register()` before `bus_set_iommu()`**: the device must
  be registered before the bus notifier fires, or devices probed before
  registration will not be attached to the IOMMU.
- **`iommu_map_nosync()` vs `iommu_map()`**: `iommu_map_nosync()` skips
  the `iotlb_sync_map()` call; the caller is responsible for flushing.
  Use `iommu_map()` unless batching is explicitly needed.
- **Nested domains**: `domain_alloc_nested()` creates a domain that
  translates through a parent domain. The child domain's page table is
  walked first; the result is then translated by the parent. Both domains
  must be kept alive while the nested domain is attached.
