# IRQ Bypass Subsystem Details

Covers `virt/lib/irqbypass.c`, `include/linux/irqbypass.h`,
`virt/kvm/eventfd.c`, `drivers/vfio/pci/vfio_pci_intrs.c`, and the
architecture-independent contracts that any KVM irqbypass implementation
must satisfy. For RISC-V specifics see `riscv-irqbypass.md`.

## irqbypass Manager Contract

Violating the connect/disconnect callback ordering causes the guest to
receive interrupts during the transition window when neither the old nor
the new routing is fully established, dropping or misrouting interrupts.

The irqbypass manager (`virt/lib/irqbypass.c`) pairs producers (physical
interrupt sources) with consumers (hypervisor acceleration sinks) via a
shared `eventfd_ctx`. On connect and disconnect the manager calls callbacks
in a fixed order:

```
stop(producer), stop(consumer)
add_consumer(producer, consumer)   [optional, producer side]
add_producer(consumer, producer)   [mandatory, consumer side]
start(consumer), start(producer)
```

- `stop` / `start` are optional. Their purpose is to quiesce the guest
  around the transition window so no interrupt is lost between the old
  routing being torn down and the new one being established.
- `add_producer` is the mandatory consumer-side callback. It is called
  after `stop` and before `start`, so the guest is quiesced when it runs.
- The manager holds a global mutex across the entire connect/disconnect
  sequence. `add_producer` and `del_producer` are never called concurrently
  for the same consumer.
- `irq_bypass_register_producer(producer, trigger, irq)` matches the
  producer against any registered consumer sharing the same `eventfd_ctx`.
  If a match is found, `__connect()` is called immediately.

**REPORT as bugs**: `add_producer` implementations that perform operations
that must be atomic with respect to interrupt delivery without holding
`irqfds.lock` — a concurrent `kvm_irq_routing_update()` may read
`irqfd->producer` under that lock.

## VFIO Producer Side

The VFIO producer registers after `request_irq()` succeeds for an MSI/MSI-X
vector. It carries the Linux IRQ number that KVM uses to reach the IRQ chip.

- `irq_bypass_register_producer(&ctx->producer, trigger, irq)` is called
  from `vfio_msi_set_vector_signal()` after the host IRQ is set up.
- `irq_bypass_unregister_producer()` is called when the MSI vector is torn
  down (device reset, VFIO group release, or MSI disable).
- The VFIO producer does not implement `add_consumer`, `del_consumer`,
  `stop`, or `start`. It is a passive participant.
- `producer->irq` is the Linux IRQ number. KVM uses this to call
  `irq_set_vcpu_affinity(host_irq, vcpu_info)` on the host IRQ chip.

## KVM Consumer Side

The KVM consumer is registered per-irqfd when `kvm_arch_has_irq_bypass()`
returns true. Returning true unconditionally wastes resources registering
consumers on hardware that cannot accelerate interrupts.

- `kvm_irqfd_assign()` in `virt/kvm/eventfd.c` registers the consumer when
  `kvm_arch_has_irq_bypass()` returns true.
- `kvm_irq_routing_update()` (called under `irqfds.lock`) calls
  `kvm_arch_update_irqfd_routing(irqfd, old, new)` for every irqfd that has
  a producer, whenever the guest IRQ routing table changes.
- The generic code guards the call: `if (irqfd->producer)` — architectures
  must not assume `irqfd->producer` is non-NULL inside
  `kvm_arch_update_irqfd_routing()`.

**`kvm_arch_has_irq_bypass()`** must be conditional on a runtime capability
check. Returning unconditional `true` registers consumers for every irqfd
on every guest, even on hardware that cannot accelerate interrupts.

- x86: gated on `enable_device_posted_irqs` (runtime flag).
- arm64: returns `true` unconditionally but `add_producer` returns 0 early
  if GICv4 is unavailable, making bypass a silent no-op.
- New architectures: gate on the specific hardware capability being present
  and active (e.g., IMSIC available and AIA in hardware acceleration mode).
- RISC-V: `kvm_arch_has_irq_bypass()` returns `imsic_enabled()` — a single
  call that checks both IMSIC presence and hardware acceleration mode.  Do
  not change to `imsic_get_global_config() && cfg->nr_ids` — `imsic_enabled()`
  already encapsulates that check.

## kvm_arch_update_irqfd_routing() Contract

Calling this function with a stale or NULL producer, or without checking
the routing entry type, causes NULL pointer dereferences or spurious IOMMU
table updates for non-MSI interrupts.

- Called under `irqfds.lock` from `kvm_irq_routing_update()`.
- Must guard against `irqfd->producer == NULL` at function entry — the
  generic code checks this before calling, but defensive guards prevent
  future misuse.
- Must return early when `new->type != KVM_IRQ_ROUTING_MSI` — only MSI
  interrupts can be bypassed.
- Must return early when old and new MSI messages are identical — avoid
  unnecessary IOMMU table updates.
- Any lock acquired inside must be lower in the hierarchy than `irqfds.lock`.

**Lock ordering**: `irqfds.lock` is acquired by `kvm_irq_routing_update()`
before calling `kvm_arch_update_irqfd_routing()`. Any lock taken inside
(e.g., a vCPU-level lock, a redistributor lock) must always be acquired
after `irqfds.lock` in all code paths. Acquiring `irqfds.lock` while
holding such a lock is an ABBA deadlock.

## irq_set_vcpu_affinity() and the NULL Convention

`irq_set_vcpu_affinity(irq, vcpu_info)` programs the IRQ chip to deliver
the interrupt directly to the specified vCPU. Passing NULL signals "remove
the mapping" — the IRQ chip reverts to normal host delivery.

- The `vcpu_info` pointer type is architecture-defined. The IRQ chip's
  `irq_set_vcpu_affinity` callback receives it as `void *` and casts to
  the architecture-specific struct.
- Returning `-EOPNOTSUPP` means the IRQ chip does not support vcpu affinity
  (no IOMMU IR domain, or hardware acceleration not available). This is not
  an error — the interrupt will be delivered via the normal host path.
- **REPORT as bugs**: callers that treat `-EOPNOTSUPP` as a fatal error and
  abort the irqfd setup — bypass should degrade gracefully to software
  injection.
- The NULL convention is RISC-V-specific. x86 calls `kvm_pi_update_irte()`
  directly; arm64 calls `kvm_vgic_v4_unset_forwarding()` directly. Document
  the NULL convention at every call site in new architectures.

## stop/start Callbacks and the Transition Window

Without `stop`/`start`, an interrupt arriving during the transition between
old and new routing may be delivered to the wrong vCPU or dropped entirely.

- arm64 implements `stop` as `kvm_arm_halt_guest()` (sets
  `vcpu->arch.pause = true` for all vCPUs, sends `KVM_REQ_SLEEP`) and
  `start` as `kvm_arm_resume_guest()`.
- x86 does not implement `stop`/`start`; it uses `pi_start_bypass()` /
  `pi_block()` as a separate quiesce mechanism.
- New architectures must either implement `stop`/`start` or document why
  the hardware guarantees atomicity of the routing update (e.g., the IOMMU
  serialises MSI table updates with in-flight translations via a fence
  command).

## irqfds.lock and add/del_producer Serialisation

`irqfd->producer` is read under `irqfds.lock` by `kvm_irq_routing_update()`.
If `add_producer` assigns `irqfd->producer` without holding `irqfds.lock`,
a concurrent routing update may read a partially-initialised producer.

- x86 holds `irqfds.lock` for the entire `add_producer` / `del_producer`
  body, serialising the assignment and the IRTE update atomically.
- arm64 does not hold `irqfds.lock` in `add/del_producer`; it relies on
  the vgic lock hierarchy instead.
- **Constraint**: if `add_producer` holds `irqfds.lock`, it must not call
  any function that acquires `irqfds.lock` internally (e.g.,
  `kvm_riscv_vcpu_irq_update()` on RISC-V). Either restructure to avoid
  the nested acquisition, or accept the arm64 approach of not holding the
  lock.

## x86 Reference: Posted Interrupts

x86 uses Intel VT-d Posted Interrupts. The IRTE is updated to post the
interrupt directly into the vCPU's Posted Interrupt Descriptor (PI
descriptor), bypassing the host kernel when the vCPU is running.

- `kvm_pi_update_irte(kvm, host_irq, guest_irq, set)` updates the IRTE.
- The device MSI message always targets the host IOMMU IRTE. The IRTE is
  what gets updated to redirect to the vCPU. `irq_write_msi_msg()` is
  **not** called from `kvm_arch_update_irqfd_routing()`.
- `irq_set_vcpu_affinity()` is not used by x86.

## arm64 Reference: GICv4 vLPI Forwarding

arm64 uses GICv4 virtual LPI (vLPI) forwarding. The GIC ITS is programmed
to deliver the LPI directly to the vCPU's redistributor doorbell.

- `kvm_vgic_v4_set_forwarding()` / `kvm_vgic_v4_unset_forwarding()` update
  the ITS ITTE (Interrupt Translation Table Entry).
- The device MSI message always targets the host GIC ITS. The ITS ITTE is
  what gets updated. `irq_write_msi_msg()` is **not** called.
- `kvm_arch_update_irqfd_routing()` cannot call `kvm_vgic_v4_set_forwarding()`
  (requires `its_lock` mutex, cannot be acquired in spinlock context). It
  falls back to software injection and documents this in a comment.
- `irq_set_vcpu_affinity()` is not used by arm64.

## Quick Checks

- **`kvm_arch_has_irq_bypass()` must be conditional**: unconditional `true`
  registers consumers on hardware that cannot accelerate interrupts.
- **`add_producer` must check routing type**: only `KVM_IRQ_ROUTING_MSI`
  entries can be bypassed. Return 0 early for other types.
- **`kvm_arch_update_irqfd_routing()` must check routing type and equality**:
  return early if neither old nor new is MSI, or if the MSI message is
  unchanged.
- **Device MSI message**: x86 and arm64 do not modify the device MSI message
  from `kvm_arch_update_irqfd_routing()`. New architectures that do must
  document the design clearly.
- **`irq_set_vcpu_affinity(NULL)` means unmap**: document this at every
  call site; it is not obvious from the function signature.
