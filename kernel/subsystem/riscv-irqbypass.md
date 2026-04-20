# RISC-V IOMMU IRQ Bypass Subsystem Details

Load `iommu.md`, `riscv-iommu.md`, and `irqbypass.md` first. This file covers
`drivers/iommu/riscv/iommu-ir.c`, `arch/riscv/kvm/vm.c`,
`arch/riscv/kvm/aia_imsic.c`, and the MSI domain stacking infrastructure
specific to RISC-V IOMMU interrupt remapping.

Baseline: `riscv/iommu-irqbypass-rfc-v3-rc3`.

## MSI Domain Stacking

The RISC-V IOMMU interrupt remapping domain sits between the device and the
IMSIC NEXUS domain. Incorrect bus token or parent ops cause MSI domain
creation to fail or route interrupts through the wrong domain.

```
device MSI domain
    └── IOMMU-IR domain  (DOMAIN_BUS_MSI_REMAP, IRQ_DOMAIN_FLAG_MSI_PARENT)
            └── IMSIC NEXUS domain  (DOMAIN_BUS_NEXUS, IRQ_DOMAIN_FLAG_MSI_PARENT)
```

- `DOMAIN_BUS_MSI_REMAP` is in `enum irq_domain_bus_token`. In
  `msi_lib_init_dev_msi_info()`, when `real_parent->bus_token ==
  DOMAIN_BUS_MSI_REMAP`, the `WARN_ON_ONCE(1)` guard is skipped and the
  function falls through to apply `real_parent->msi_parent_ops` flags.
  `domain != real_parent` is expected and correct here.
- `riscv_iommu_ir_msi_parent_ops` does not set `bus_select_token` (stays
  zero = `DOMAIN_BUS_ANY`). The IR domain uses `irq_domain_create_hierarchy()`
  with `riscv_iommu_ir_irq_domain_ops` which has no `.select` callback.
- `dev_set_msi_domain(dev, irqdomain)` installs the IR domain as the
  device's MSI domain, replacing the IMSIC NEXUS domain for that device.
  Called inside `riscv_iommu_ir_irq_domain_create()`.

## IRQ Domain Lifecycle

Incorrect ordering of `irq_domain_remove()` and `irq_domain_free_fwnode()`
leaks the fwnode handle. The correct teardown sequence is:

```c
fn = info->irqdomain->fwnode;   // save before remove
irq_domain_remove(info->irqdomain);
info->irqdomain = NULL;
irq_domain_free_fwnode(fn);     // free after remove
```

`riscv_iommu_ir_irq_domain_remove()` is called from both
`riscv_iommu_probe_device()` error paths (where `info->domain` is NULL) and
`riscv_iommu_release_device()` (where `info->domain` may be NULL for devices
never attached to a paging domain). All call sites guard with `if (domain)`
before calling `riscv_iommu_ir_free_msi_table(domain)`.

## MSI Table Setup and the One-Time Init Sentinel

`riscv_iommu_ir_attach_paging_domain()` uses `domain->msi_addr_mask == 0`
as a sentinel to detect the first attach and trigger
`riscv_ir_set_imsic_global_config()`. This is the only place the MSI
topology fields (`msi_addr_mask`, `msi_addr_pattern`, `group_index_bits`,
`group_index_shift`, `imsic_stride`, `msi_root`, `msi_pte_counts`) are
initialised.

- `msi_root` is only allocated when `RISCV_IOMMU_CAPABILITIES_MSI_FLAT` is
  set. On base-format hardware `msi_root` stays NULL even after first attach.
- `msi_lock` must be `raw_spinlock_t` because `irq_set_vcpu_affinity()` is
  called from KVM with IRQs disabled (atomic context).
- `msitbl_config` is a generation counter incremented on every full MSI table
  reconfiguration. IRQs store their config generation at alloc time; at free
  time only IRQs whose stored config matches `domain->msitbl_config` need
  explicit unmapping — stale IRQs from a previous config were already cleared.
- `riscv_iommu_gstage_best_mode()` must return non-zero before allocating
  the MSI table. If `MSI_FLAT` is set but no SV*x4 g-stage mode is available,
  the hardware is non-compliant; the driver warns and skips MSI table setup.

## IOTINVAL.GVMA Address Field

Passing the host PA (from the PTE's PPN field) instead of the guest PA as
the IOTINVAL.GVMA ADDR operand invalidates the wrong cache entries, leaving
stale MSI table entries in hardware.

The RISC-V IOMMU spec section 7.3.3 defines the ADDR field as the *guest
physical address* — the MSI address the device writes (input to the MSI
table lookup), not the host PA stored in the PTE's PPN field.

- **REPORT as bugs**: code that passes `pfn_to_phys(FIELD_GET(
  RISCV_IOMMU_MSIPTE_PPN, pte->pte))` as the invalidation address.
- **REPORT as bugs**: calling `riscv_iommu_ir_clear_pte(pte)` before
  `riscv_iommu_ir_msitbl_inval()` when the inval function derives the
  address from `pte->pte` — after clear, `pte->pte == 0`.
- `gpa == 0` means broadcast invalidation (no `AV` bit set) — invalidates
  all MSI table entries for the domain.

## riscv_iommu_ir_chip_data and Per-IRQ State

One `chip_data` struct must be allocated per IRQ inside the alloc loop.
Allocating one struct for all IRQs in a range causes field corruption (writes
for IRQ i overwrite those for IRQ i-1) and double-free in the free loop.

`struct riscv_iommu_ir_chip_data` stores:
- `config` — generation counter snapshot at map time; compared against
  `domain->msitbl_config` at free time to skip unmapping stale IRQs.
- `gpa` — the guest MSI physical address used as the `IOTINVAL.GVMA` ADDR
  operand and to recompute the PTE index via
  `riscv_iommu_ir_compute_msipte_idx(domain, gpa)`.

## MSI Table Map/Unmap Double-Checked Locking

The map function uses a double-checked locking pattern with `refcount_t`:

```c
if (!refcount_inc_not_zero(&domain->msi_pte_counts[idx])) {
    scoped_guard(raw_spinlock_irqsave, &domain->msi_lock) {
        if (refcount_read(&domain->msi_pte_counts[idx]) == 0) {
            /* first mapper: write PTE, invalidate, set count=1 */
        } else {
            refcount_inc(&domain->msi_pte_counts[idx]);
        }
    }
}
```

`refcount_inc_not_zero` on a zero count returns false without incrementing;
the lock serialises the 0→1 transition and the PTE write. Calling
`riscv_iommu_ir_msitbl_inval()` (which uses `rcu_read_lock` and busy-waits
via `readx_poll_timeout`) while holding `raw_spinlock_irqsave` is safe
because no sleeping occurs.

## KVM Consumer: vm.c and aia_imsic.c

### kvm_arch_has_irq_bypass()

Returns `imsic_enabled()`. This correctly gates bypass on IMSIC being
present and in hardware acceleration mode. Do not change to unconditional
`true` — that would register consumers on systems without IMSIC or in AIA
emulation mode.  `imsic_enabled()` encapsulates the `imsic_get_global_config()`
and hardware-mode checks internally.

### kvm_arch_irq_bypass_add_producer()

Returns early when `kvm->arch.aia.mode == KVM_DEV_RISCV_AIA_MODE_EMUL` —
no VS-files available in emulation mode, bypass cannot work.

**Open**: Does not hold `irqfds.lock` around the `irqfd->producer`
assignment and the `kvm_arch_update_irqfd_routing()` call. See `irqbypass.md`
for the general constraint. The specific complication on RISC-V is that
`kvm_riscv_vcpu_irq_update()` acquires `irqfds.lock` internally, so it
cannot be called while the lock is held.

### kvm_arch_update_irqfd_routing()

- Guards: `if (!irqfd->producer) return` at entry; returns early when old
  and new MSI messages are identical; returns early when
  `new->type != KVM_IRQ_ROUTING_MSI`.
- Reads `vcpu->arch.aia_context.imsic_addr` without a lock. This is safe
  because `imsic_addr` is set once at AIA device configuration time and
  never changes. A comment explaining this invariant is needed.
- Calls `irq_set_vcpu_affinity(host_irq, &vcpu_info)` to populate the IOMMU
  MSI table entry mapping the guest IMSIC GPA to the host VS-file HPA.
- Calls `irq_write_msi_msg()` after a successful `irq_set_vcpu_affinity()`.
  This reprograms the device MSI target address to point at the guest VS-file
  GPA directly. This is intentional: the device targets the guest GPA; the
  IOMMU MSI table maps guest GPA → host VS-file HPA. Neither x86 nor arm64
  does this — it is a RISC-V-specific design that needs a comment.
- Holds `vsfile_lock(read)` around the `irq_set_vcpu_affinity()` and
  `irq_write_msi_msg()` calls to ensure `imsic->vsfile_pa` is stable.

### kvm_riscv_vcpu_irq_update()

Called from `kvm_riscv_vcpu_aia_imsic_update()` **after**
`write_unlock_irqrestore(&imsic->vsfile_lock)`. This is the fix for the
ABBA deadlock: the old code called it while holding `vsfile_lock(write)`,
which conflicted with `kvm_irqfd_update()` holding `irqfds.lock` then
acquiring `vsfile_lock(read)`.

- Acquires `irqfds.lock` internally to iterate `kvm->irqfds.items`.
- Breaks on any non-zero return from `irq_set_vcpu_affinity()`, with
  `WARN_ON_ONCE` for non-EOPNOTSUPP errors.

**REPORT as bugs**: any code path that acquires `irqfds.lock` while holding
`vsfile_lock`, or acquires `vsfile_lock` while holding `irqfds.lock`.

### struct riscv_iommu_ir_vcpu_info

Defined in `include/linux/irqchip/riscv-imsic.h`. Fields:
- `gpa` — guest MSI physical address (the untranslated address the device writes)
- `hpa` — host VS-file physical address (the IMSIC interrupt file HPA)
- `msi_addr_mask`, `msi_addr_pattern` — IMSIC address filter
- `group_index_bits`, `group_index_shift` — IMSIC topology parameters

**Resolved**: Moved to `include/linux/irqchip/riscv-imsic.h` in rc3.

## IMSIC Identity Mappings in the S-Stage Table

IMSIC addresses must be identity-mapped in the **s-stage** so the IOMMU
can match them against `msi_addr_pattern`/`msi_addr_mask` and redirect them
to the MSI table. Without s-stage mappings the s-stage translation faults
before the MSI table is ever consulted.

`riscv_iommu_ir_map_unmap_imsics()` calls `iommu_map(&domain->domain,
addr, addr, ...)` — this maps into the s-stage (`domain->domain`), not the
g-stage. The g-stage identity mappings for DMA are installed automatically
by the `gstage_install_ops()` wrapper on every `iommu_map()` call; IMSIC
addresses are handled separately via this explicit s-stage map.

**REPORT as bugs**: code that calls `riscv_iommu_gstage_map()` for IMSIC
addresses — that would map into the g-stage and have no effect on MSI
lookup.

## Open Issues in v3-rc3

- **`irqfds.lock` in add_producer**: `kvm_arch_irq_bypass_add_producer()`
  assigns `irqfd->producer` and calls `kvm_arch_update_irqfd_routing()`
  without holding `irqfds.lock`. The specific complication is that
  `kvm_riscv_vcpu_irq_update()` acquires `irqfds.lock` internally, so it
  cannot be called while the lock is held. See `irqbypass.md` for the
  general constraint.
- **`stop`/`start` callbacks**: not implemented; weak no-ops used. The
  IOMMU MSI table update is serialised by `IOFENCE.C` inside
  `riscv_iommu_ir_msitbl_inval()`, which may be sufficient — needs
  confirmation from the spec.
- **`imsic_addr` invariant comment**: the lockless read of
  `vcpu->arch.aia_context.imsic_addr` in `kvm_arch_update_irqfd_routing()`
  needs a comment explaining that `imsic_addr` is set once at AIA device
  configuration time and never changes.

## Design Notes

### irq_write_msi_msg() in kvm_arch_update_irqfd_routing()

Unlike x86 (which updates the IRTE and never touches the device MSI message)
and arm64 (which updates the ITS ITTE and never touches the device MSI
message), the RISC-V implementation calls `irq_write_msi_msg()` from
`kvm_arch_update_irqfd_routing()` to reprogram the device MSI target address
to point at the guest VS-file GPA directly.

This is intentional: the device targets the guest IMSIC GPA; the IOMMU MSI
table maps guest GPA → host VS-file HPA. The device must be programmed with
the guest GPA so that its MSI writes match `msi_addr_pattern`/`msi_addr_mask`
and are intercepted by the IOMMU MSI table. If the device were left
programmed with the host IMSIC physical address (as x86/arm64 leave devices
programmed with the host IRTE/ITS address), the IOMMU would not intercept
the write and the MSI would be delivered to the host IMSIC instead of the
guest VS-file.

This design requires a comment at the call site explaining the departure from
x86/arm64 behaviour.

## Quick Checks

- **`msi_root` NULL check is not a capability guard**: checking
  `domain->msi_root` before populating a local `dc` struct prevents non-zero
  MSI values, but does not prevent `riscv_iommu_iodir_update()` from writing
  zero values to hardware. A separate `MSI_FLAT` capability check is needed
  in the update function itself.
- **`raw_spinlock_t msi_lock`**: `irq_set_vcpu_affinity()` runs in atomic
  context (called from KVM with IRQs disabled). Using `spinlock_t` here
  would deadlock.
- **`imsic_get_global_config()` NULL check**: on systems without IMSIC the
  pointer is NULL and no IR domain is created. All callers must handle NULL.
- **`msitbl_config` generation counter**: stale IRQs from a previous config
  do not need explicit unmapping at free time — they were cleared by the
  config-change path. Only IRQs whose stored config matches the current
  `domain->msitbl_config` need `riscv_iommu_ir_msitbl_unmap()`.
- **`struct riscv_iommu_ir_vcpu_info` location**: defined in
  `include/linux/irqchip/riscv-imsic.h`. ✓ Resolved in rc3.
