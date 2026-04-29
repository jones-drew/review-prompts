# RISC-V IOMMU IRQ Bypass Subsystem Details

Load `iommu.md`, `riscv-iommu.md`, and `irqbypass.md` first. This file covers
`drivers/iommu/riscv/iommu-ir.c`, `arch/riscv/kvm/vm.c`,
`arch/riscv/kvm/aia_imsic.c`, and the MSI domain stacking infrastructure
specific to RISC-V IOMMU interrupt remapping.

Baseline: `riscv/iommu-irqbypass-rfc-v3-rc5` (major redesign from rc4).

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
dev_set_msi_domain(dev, parent); // restore parent domain
```

`riscv_iommu_ir_irq_domain_remove()` is called from both
`riscv_iommu_probe_device()` error paths and
`riscv_iommu_release_device()`. All call sites that free the MSI table guard
with `if (domain)` before calling `riscv_iommu_ir_free_msi_table(domain)`.

## MSI Table Setup and the One-Time Init Sentinel

`riscv_iommu_ir_attach_paging_domain()` uses `domain->msi_addr_mask == 0`
as a sentinel to detect the first attach and trigger
`riscv_ir_set_imsic_global_config()`. This is the only place the MSI
topology fields (`msi_addr_mask`, `msi_addr_pattern`, `group_index_bits`,
`group_index_shift`, `imsic_stride`, `msi_root`, `msi_pte_counts`) are
initialised.

- `msi_root` is allocated on first attach for MSI_FLAT + imsic_enabled()
  domains. If `imsic_get_global_config()` returns NULL (no IMSIC), no MSI
  table is allocated and `msi_root` stays NULL.
- `msi_lock` must be `raw_spinlock_t` because `irq_set_vcpu_affinity()` is
  called from KVM with IRQs disabled (atomic context).
- `msitbl_config` is a generation counter incremented on every full MSI table
  reconfiguration. IRQs store their config generation at alloc time; at free
  time only IRQs whose stored config matches `domain->msitbl_config` need
  explicit unmapping — stale IRQs from a previous config were already cleared.
- `riscv_iommu_gstage_best_mode()` is no longer used — the SV*x4 mode is
  selected at domain allocation time and the GSCID is pre-allocated. There is
  no longer a separate g-stage page table to initialise on first attach.

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

## riscv_iommu_ir_irq_set_vcpu_affinity() — New Design (rc5)

`riscv_iommu_ir_irq_set_vcpu_affinity()` is the core irqbypass hook. It
directly manipulates the MSI table PTEs under `raw_spinlock_t msi_lock`
without calling `iommu_map()`. This avoids the PREEMPT_RT violation that
existed in earlier revisions.

**NULL vcpu_info means remove the mapping** (revert to host delivery):
```c
riscv_iommu_ir_msitbl_unmap(domain, data, old_idx);
```

**Config mismatch (new IMSIC topology)**:  calls
`riscv_iommu_ir_vcpu_new_config()`, which:
1. Calls `riscv_iommu_ir_msitbl_clear()` — zeroes all PTEs and refcounts.
2. Updates topology fields (`msi_addr_mask`, `msi_addr_pattern`,
   `group_index_bits`, `group_index_shift`).
3. Increments `msitbl_config` (generation counter).
4. Writes the new PTE directly to `domain->msi_root[idx]`.
5. Issues `riscv_iommu_ir_msitbl_inval_all()` — IOTINVAL.GVMA broadcast.
6. Sets `msi_pte_counts[idx] = 1`.
7. Calls `riscv_iommu_ir_msiptp_update()` to update all DCs via
   `riscv_iommu_iodir_update()`. This is called from `irq_set_vcpu_affinity()`
   which runs under `irqfds.lock` (IRQs disabled). `riscv_iommu_iodir_update()`
   uses only `readx_poll_timeout` (busy-wait), not sleeping locks — RT-safe.

**Config match, fast path** (same IMSIC topology, possibly different slot):
Holds `raw_spinlock` (not irqsave — caller already has IRQs disabled):
1. Computes new PTE value and writes if changed; issues IOTINVAL.GVMA.
2. If `old_config != msitbl_config` (IRQ survived a config change): bump
   refcount on new slot only.
3. If `new_idx != old_idx` (slot change): dec old refcount (freeing PTE if
   zero), inc new refcount.

**PREEMPT_RT status**: The fast path uses `raw_spinlock_t` (RT-safe) and
`readx_poll_timeout` (busy-wait, RT-safe). The topology-change path calls
`riscv_iommu_iodir_update()` which busy-waits via `readx_poll_timeout` but
does not acquire sleeping locks. This path is called under `irqfds.lock`
(`spin_lock_irq`). On PREEMPT_RT `spinlock_t` is a sleeping lock; confirm
that `riscv_iommu_iodir_update()` acquires no spinlock_t internally before
marking RT-safe.

## KVM Consumer: vm.c and aia_imsic.c

### kvm_arch_has_irq_bypass()

Returns `imsic_enabled()`. This correctly gates bypass on IMSIC being
present and in hardware acceleration mode. Do not change to unconditional
`true` — that would register consumers on systems without IMSIC or in AIA
emulation mode.

### kvm_arch_irq_bypass_add_producer()

Returns early when `kvm->arch.aia.mode == KVM_DEV_RISCV_AIA_MODE_EMUL` —
no VS-files available in emulation mode, bypass cannot work.

Assigns `irqfd->producer = prod` then calls
`kvm_arch_update_irqfd_routing(irqfd, NULL, &irqfd->irq_entry)`. Does not
hold `irqfds.lock` — the specific complication is that
`__kvm_riscv_vcpu_irq_update()` acquires `irqfds.lock` internally, so it
cannot be called while the lock is held. See `irqbypass.md` for the general
constraint.

### kvm_arch_irq_bypass_del_producer()

Implemented in rc5. Calls
`kvm_arch_update_irqfd_routing(irqfd, &irqfd->irq_entry, NULL)` before
clearing `irqfd->producer = NULL`. This ensures the IOMMU MSI table entry
is removed before the producer reference is dropped.

The irqbypass framework calls `producer->stop()` **before**
`consumer->del_producer()` (see `virt/lib/irqbypass.c: __disconnect()`),
so the device is quiesced when `irq_set_vcpu_affinity(NULL)` runs —
no in-flight MSIs reach the IOMMU during the unmap. There is no need to
reprogram the device MSI target address back to the host IMSIC HPA here;
the irqbypass `start()` callbacks and, if the device is fully unbound from
VFIO, the e1000e reprobe re-run `irq_write_msi_msg` with the correct host
IMSIC address.

### kvm_arch_update_irqfd_routing()

- Called from both `kvm_arch_irq_bypass_add_producer()` (not under
  `irqfds.lock`) and `kvm_irq_routing_update()` (under `irqfds.lock`).
- Returns early when old and new MSI messages are identical — avoid
  unnecessary IOMMU table updates.
- Returns early when `!new` (del_producer path): calls
  `irq_set_vcpu_affinity(host_irq, NULL)` to remove the MSI table mapping.
- Returns early when `new->type != KVM_IRQ_ROUTING_MSI`.
- Iterates `kvm_for_each_vcpu` to find the vCPU whose `imsic_addr` matches
  the MSI target address. Returns if no match (device not routing to a vCPU).
- Reads `vcpu->arch.aia_context.imsic_addr` without a lock. This is safe
  because `imsic_addr` is set once at AIA device configuration time and
  never changes.

  **Missing comment**: the lockless `imsic_addr` read needs a comment
  explaining this invariant. Flag if absent.

- Holds `vsfile_lock(read)` around `irq_set_vcpu_affinity()` and
  `irq_write_msi_msg()` to ensure `imsic->vsfile_pa` is stable.

- After a successful `irq_set_vcpu_affinity()`, calls
  `irq_data_get_irq_chip(irqdata)->irq_write_msi_msg(irqdata, &msg)` directly
  (not via `irq_write_msi_msg()`). This reprograms the device MSI target
  address to point at the guest VS-file GPA directly.

  **RISC-V-specific design**: unlike x86 (updates IRTE) and arm64 (updates
  ITS ITTE), RISC-V reprograms the device MSI target to the guest IMSIC GPA.
  The device writes to the guest GPA; the IOMMU MSI table maps guest GPA →
  host VS-file HPA. If the device were left targeting the host IMSIC physical
  address, the IOMMU would not intercept the write. This departure from x86/arm64
  must be documented at the call site.

- Any lock acquired inside must be lower in the hierarchy than `irqfds.lock`.

### kvm_riscv_vcpu_irq_update() / __kvm_riscv_vcpu_irq_update()

Called from `kvm_riscv_vcpu_aia_imsic_update()` **after**
`write_unlock_irqrestore(&imsic->vsfile_lock)`. This is the fix for the
ABBA deadlock: the old code called it while holding `vsfile_lock(write)`,
which conflicted with `kvm_irqfd_update()` holding `irqfds.lock` then
acquiring `vsfile_lock(read)`.

- Acquires `irqfds.lock` internally to iterate `kvm->irqfds.items`.
- Lock ordering comment in code: `irqfds.lock -> vsfile_lock(read)` — same
  order as `kvm_arch_update_irqfd_routing()`.
- Filters to irqfds whose MSI target matches the given vCPU's `imsic_addr`
  (not all irqfds in the VM).

**OPEN**: Breaks on the first `irq_set_vcpu_affinity()` failure (`if (ret) break`).
After a vCPU migration, if one device's `irq_set_vcpu_affinity()` fails, the
remaining irqfds in the VM still point to the old VS-file. The loop should
continue and log each failure rather than aborting early.

**REPORT as bugs**: any code path that acquires `irqfds.lock` while holding
`vsfile_lock`, or acquires `vsfile_lock` while holding `irqfds.lock`.

### struct riscv_iommu_ir_vcpu_info

Defined in `include/linux/irqchip/riscv-imsic.h`. Fields:
- `gpa` — guest MSI physical address (the untranslated address the device writes)
- `hpa` — host VS-file physical address (the IMSIC interrupt file HPA)
- `msi_addr_mask`, `msi_addr_pattern` — IMSIC address filter
- `group_index_bits`, `group_index_shift` — IMSIC topology parameters

## Open Issues in v3-rc5

- **irqfd loop breaks on first failure**: `__kvm_riscv_vcpu_irq_update()`
  breaks on the first `irq_set_vcpu_affinity()` failure, leaving remaining
  devices with stale PTE entries after vCPU migration. The loop should
  continue and log each failure.

- **`irqfds.lock` in add_producer**: `kvm_arch_irq_bypass_add_producer()`
  assigns `irqfd->producer` and calls `kvm_arch_update_irqfd_routing()`
  without holding `irqfds.lock`. The specific complication is that
  `__kvm_riscv_vcpu_irq_update()` acquires `irqfds.lock` internally, so it
  cannot be called while the lock is held. See `irqbypass.md` for the
  general constraint.

- **`stop`/`start` callbacks**: not implemented; weak no-ops used.  The
  IOMMU MSI table update is serialised by `IOFENCE.C` inside
  `riscv_iommu_ir_msitbl_inval()`, which may be sufficient — needs
  confirmation from the spec.

- **`imsic_addr` invariant comment**: the lockless read of
  `vcpu->arch.aia_context.imsic_addr` in `kvm_arch_update_irqfd_routing()`
  needs a comment explaining that `imsic_addr` is set once at AIA device
  configuration time and never changes.

- **NULL-deref in `riscv_iommu_ir_irq_domain_alloc_irqs()`**: `info->domain`
  is accessed without a null-check. If IRQ allocation somehow occurs before
  domain attach the driver will crash on `riscv_iommu_ir_compute_msipte_idx()`
  dereferencing `domain->group_index_bits`. Add a guard: `if (!domain) return -ENODEV`.

- **PREEMPT_RT topology-change path**: `riscv_iommu_ir_vcpu_new_config()`
  calls `riscv_iommu_ir_msiptp_update()` → `riscv_iommu_iodir_update()`.
  If `riscv_iommu_iodir_update()` acquires any `spinlock_t` internally, this
  is an RT violation when called from `kvm_arch_update_irqfd_routing()` under
  `irqfds.lock` (IRQs disabled). Verify the call chain is RT-safe.

- **TC3 IOMMU faults (QEMU only)**: QEMU testing of two e1000e devices in one
  guest produces 4 deterministic IOMMU faults per iteration. The fault record
  decodes as `CAUSE=23` (`RISCV_IOMMU_FQ_CAUSE_WR_FAULT_VS` — write
  guest-virtual page fault), `IOVA=0x8001000`. This is **not** an MSI-table
  fault; it means the IOMMU routed the write through the G-stage DMA path
  instead of the MSI path, implying the DC's `msi_addr_pattern/mask` did not
  match the IOVA at that instant.

  The `IOTINVAL.GVMA` usage in the fast path and `vcpu_new_config` path was
  verified against `iommu_sw_guidelines.adoc`: a targeted invalidation
  (`GV=AV=1, ADDR=GPA, GSCID=DC.iohgatp.GSCID`) is the spec-prescribed
  method for flushing a single MSI PTE cache entry; the broadcast form
  (`GV=1, AV=0`) invalidates all entries for a GSCID. The driver issues
  both forms correctly.

  The faults correlate with `IOTINVAL.GVMA + IOFENCE` sequences during vCPU
  migration (HPA update to MSI table PTE). The most likely cause is a QEMU
  RISC-V IOMMU emulation defect where the emulator briefly consults a stale
  DC entry during IOTINVAL processing and fails to recognise the IOVA as an
  MSI address. Functionally benign (e1000e retries) and not reproduced on
  real hardware. Do not raise as a kernel driver bug without first
  confirming on real hardware.

## Design Notes

### irq_write_msi_msg() via chip interface

In rc5, `kvm_arch_update_irqfd_routing()` calls
`irq_data_get_irq_chip(irqdata)->irq_write_msi_msg(irqdata, &msg)` directly
instead of the `irq_write_msi_msg()` wrapper. This bypasses the intermediate
domain translation steps and writes the MSI message directly via the chip.
The rationale is the same as before: reprogram the device to target the guest
IMSIC GPA so that subsequent MSI writes are intercepted by the IOMMU MSI
table.

### irq_write_msi_msg() in kvm_arch_update_irqfd_routing()

Unlike x86 (which updates the IRTE and never touches the device MSI message)
and arm64 (which updates the ITS ITTE and never touches the device MSI
message), the RISC-V implementation calls the chip's `irq_write_msi_msg`
from `kvm_arch_update_irqfd_routing()` to reprogram the device MSI target
address to point at the guest VS-file GPA directly.

This is intentional: the device targets the guest IMSIC GPA; the IOMMU MSI
table maps guest GPA → host VS-file HPA. The device must be programmed with
the guest GPA so that its MSI writes match `msi_addr_pattern`/`msi_addr_mask`
and are intercepted by the IOMMU MSI table. This design requires a comment at
the call site explaining the departure from x86/arm64 behaviour.

## Quick Checks

- **`msi_root` NULL check is not a capability guard**: `riscv_iommu_ir_msitbl_map()`
  returns early when `domain->msi_root == NULL` (e.g., base-format hardware).
  This prevents PTE writes but does not prevent `riscv_iommu_iodir_update()`
  from writing zero values to hardware. A separate `MSI_FLAT` capability check
  is needed in the update function itself.
- **`raw_spinlock_t msi_lock`**: `irq_set_vcpu_affinity()` runs in atomic
  context (called from KVM with IRQs disabled). Using `spinlock_t` here
  would deadlock on PREEMPT_RT.
- **`imsic_get_global_config()` NULL check**: on systems without IMSIC the
  pointer is NULL and no IR domain is created. All callers must handle NULL.
- **`msitbl_config` generation counter**: stale IRQs from a previous config
  do not need explicit unmapping at free time — they were cleared by the
  config-change path. Only IRQs whose stored config matches the current
  `domain->msitbl_config` need `riscv_iommu_ir_msitbl_unmap()`.
- **`struct riscv_iommu_ir_vcpu_info` location**: defined in
  `include/linux/irqchip/riscv-imsic.h`. ✓
- **`guard(raw_spinlock)` not `irqsave`**: `riscv_iommu_ir_irq_set_vcpu_affinity()`
  uses `guard(raw_spinlock)` (not irqsave) because the caller (`kvm_arch_update_irqfd_routing()`)
  already runs with IRQs disabled via `spin_lock_irq(&irqfds.lock)`.
