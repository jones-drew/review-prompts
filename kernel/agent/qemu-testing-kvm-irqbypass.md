---
name: qemu-kvm-irqbypass-tester
description: Generates and runs the RISC-V IOMMU irqbypass test suite (TC1–TC5). Builds on qemu-testing.md and qemu-testing-kvm.md. Requires a QEMU binary, kernel Image, buildroot rootfs, and the share64 directory containing lkvm-static.
tools: Bash, Read, Write
model: sonnet
---

# QEMU KVM irqbypass Test Agent

**Read `qemu-testing.md` and `qemu-testing-kvm.md` in that order first.**
Those files define the QEMU command construction, pexpect discipline, login
detection, SSH setup, dmesg health checks (including the broad error/warn/fail
scan), kvmtool invocation, shutdown patterns, and the "Updating Test Scripts"
process. This file only documents what is new: the irqbypass-specific QEMU
configuration, VFIO device management, QEMU trace analysis, and the TC1–TC5
test sequences.

## Updating Test Scripts

Follow the process defined in `qemu-testing.md` ("Updating Test Scripts"):
delete existing scripts, update the agent, regenerate. Never patch scripts
directly.

## Required Inputs

- **QEMU binary**: `qemu-system-riscv64`
- **Kernel Image**: RISC-V 64-bit kernel built with KVM, VFIO, IOMMUFD,
  RISCV_IOMMU, and e1000e as builtins; AIA support enabled. Auto-detect
  from `build/arch/riscv/boot/Image` per `qemu-testing.md`.
- **Rootfs**: buildroot ext2 image; root login, no password
- **Share directory**: directory shared into the guest via virtio-9p at
  `/share`, containing:
  - `bin/lkvm-static` — statically linked kvmtool binary (RISC-V 64-bit,
    built with AIA support; verify with `strings lkvm-static | grep disable-ssaia`)
  - `bin/runkvm` — launches a kvmtool guest (no VFIO binding; binding is
    done separately before calling runkvm)
  - `runqemu/Image` — copy of the kernel Image

## QEMU Configuration for irqbypass

```python
cfg = {
    'machine':       'virt',
    'machine_extra': ',aia=aplic-imsic,aia-guests=5,iommu-sys=on',
    'cpu':           'max',
    'smp':           6,
    'mem':           '4G',
    'cmdline':       ('root=/dev/vda console=ttyS0 earlycon '
                      'ignore_loglevel debug '
                      'vfio_iommu_type1.allow_unsafe_interrupts=1'),
    'ssh_port':      9997,
    'extra_args': [
        # 9p share
        '-fsdev local,id=p9,path=<share_dir>,security_model=mapped-xattr',
        '-device virtio-9p-device,fsdev=p9,mount_tag=p9',
        # Three e1000e devices on separate user-net subnets
        '-device e1000e,netdev=net1',
        '-netdev user,id=net1,net=192.168.0.0/24',
        '-device e1000e,netdev=net2',
        '-netdev user,id=net2,net=192.168.1.0/24',
        '-device e1000e,netdev=net3',
        '-netdev user,id=net3,net=192.168.2.0/24',
        # QEMU IOMMU trace to stderr
        '-trace riscv_iommu_*',
    ],
}
```

**Note on `allow_unsafe_interrupts`**: TC1–TC5 use kvmtool as the VMM,
which uses the legacy VFIO container path (`/dev/vfio/<group>`). This path
does not set `IRQ_DOMAIN_FLAG_ISOLATED_MSI`, so
`vfio_iommu_type1.allow_unsafe_interrupts=1` is required in the kernel
cmdline.

The iommufd cdev path (which sets `ISOLATED_MSI` lazily via
`IOMMU_CAP_VIRT_MSI_ISOLATION` and does not need `allow_unsafe_interrupts`)
is exercised when a VMM with iommufd support (e.g. QEMU with
`-device vfio-pci` on a platform with nested KVM) calls
`vfio_iommufd_physical_bind()`. kvmtool does not yet support iommufd.
iommufd path testing is deferred until kvmtool gains iommufd support or
an alternative VMM is available. See TC6 below.

QEMU stderr carries both IOMMU trace events and QEMU error messages.
Redirect to a file: `2>/tmp/qemu_trace.log`.

## Kernel Config for irqbypass Testing

Apply these overrides after `make defconfig`, then run `mod2noconfig` to
drop unneeded modules before resolving dependencies with `olddefconfig`:

```bash
./scripts/config --file build/.config \
    -e KVM \
    -e VFIO \
    -e VFIO_IOMMU_TYPE1 \
    -e VFIO_PCI \
    -e IOMMUFD \
    -e E1000E \
    -e RISCV_IOMMU

make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- O=build mod2noconfig
make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- O=build olddefconfig
```

All must be `=y` (builtin):

| Symbol | Purpose |
|--------|---------|
| `KVM` | KVM hypervisor |
| `VFIO` | VFIO framework |
| `VFIO_IOMMU_TYPE1` | Legacy VFIO IOMMU backend |
| `VFIO_PCI` | VFIO PCI driver — binds e1000e for passthrough |
| `IOMMUFD` | iommufd cdev — primary device assignment path |
| `E1000E` | Intel e1000e NIC — device under test |
| `RISCV_IOMMU` | RISC-V IOMMU driver |

Verify:
```bash
grep -E "^CONFIG_(KVM|VFIO|VFIO_IOMMU_TYPE1|VFIO_PCI|IOMMUFD|E1000E|RISCV_IOMMU)=" build/.config
```

## VFIO Device Binding

Bind a device to vfio-pci before launching kvmtool:

```sh
# Bind BDF to vfio-pci
BDF=0000:00:01.0
echo $BDF > /sys/bus/pci/devices/$BDF/driver/unbind 2>/dev/null
echo vfio-pci > /sys/bus/pci/devices/$BDF/driver_override
echo $BDF > /sys/bus/pci/drivers_probe

# Unbind from vfio-pci and return to e1000e
echo $BDF > /sys/bus/pci/devices/$BDF/driver/unbind 2>/dev/null
echo > /sys/bus/pci/devices/$BDF/driver_override
echo $BDF > /sys/bus/pci/drivers_probe
```

After unbinding, wait ~2 seconds and verify e1000e reprobed:
```bash
dmesg | tail -5 | grep -iE "e1000e.*eth"
```

## Share Directory Scripts

**`bin/runkvm`** — launch a kvmtool guest (no VFIO; bind devices first):

```sh
#!/bin/sh
CPUS=2
MEM=256
IMAGE=/share/runqemu/Image

lkvm-static run \
    -m "${MEM}" \
    -c "${CPUS}" \
    -p "console=ttyS0 earlycon ${CMDLINE}" \
    -k "${IMAGE}" \
    --debug \
    "$@"
```

Pass `--vfio-pci=<bdf>` arguments after binding devices:
```sh
# Bind then launch (PATH must include /share/bin for lkvm-static)
echo 0000:00:01.0 > /sys/bus/pci/devices/0000:00:01.0/driver/unbind
echo vfio-pci > /sys/bus/pci/devices/0000:00:01.0/driver_override
echo 0000:00:01.0 > /sys/bus/pci/drivers_probe
PATH=/share/bin:$PATH /share/bin/runkvm --vfio-pci=0000:00:01.0
```

No rootfs argument — kvmtool generates a minimal rootfs in `~/.kvm/`.
The host's `/` is shared into the guest at `/host`.

## Link-up Timing

After a device is unbound from vfio-pci and rebound to the e1000e driver,
or after a new kvmtool guest boots with a VFIO device, the NIC link takes
a moment to come up. Pinging immediately after `ip link set ethN up` may
fail with 0 packets received even though the interface is configured.

**`ip link set ethN up` timeout**: on the first VFIO bind the command
returns immediately. On the second and subsequent bind cycles (TC2), the
e1000e driver's `ndo_open()` takes 15–30 s because the IOMMU/KVM
irqbypass path does more work setting up an existing domain. Always use a
60 s timeout for `ip link set` and `ip addr add` when the device may have
been through a prior VFIO cycle:

```python
run(guest, f'ip link set {iface} up',              GUEST_PROMPT, timeout=60)
run(guest, f'ip addr add {ip}/24 dev {iface}',     GUEST_PROMPT, timeout=60)
```

**Pattern**: after the commands return, poll `ip link show ethN` for
`LOWER_UP` before pinging, then wait 2 s for the link to stabilise:

```python
for _ in range(10):
    out = run(guest, f'ip link show {iface}', GUEST_PROMPT)
    if 'UP' in out and 'LOWER_UP' in out:
        break
    time.sleep(1)
time.sleep(2)
```

This applies to:
- Guest eth1 in TC5 (second VFIO device in Guest A across iterations)
- Any interface that was recently brought up after a VFIO rebind cycle

## Network Topology

```
Host eth0 (virtio-net):  10.0.2.x/24  — management, DHCP, SSH via :9997
Host eth1 (e1000e net1): 192.168.0.x  — VFIO candidate, gw 192.168.0.2
Host eth2 (e1000e net2): 192.168.1.x  — VFIO candidate, gw 192.168.1.2
Host eth3 (e1000e net3): 192.168.2.x  — VFIO candidate, gw 192.168.2.2
```

**Host e1000e interfaces**: use `udhcpc` when devices are NOT in VFIO mode
(one device per subnet, slirp responds normally).

**Guest e1000e interfaces**: always use static IP — no DHCP server on
192.168.x subnets. Assign with:
```bash
ip link set eth0 up && ip addr add 192.168.0.15/24 dev eth0
```

**QEMU slirp limitation**: with multiple VFIO devices active simultaneously,
slirp becomes unreliable for some subnets:
- One VFIO device per guest: slirp responds normally, ping works.
- Two VFIO devices in one guest (TC3/TC5 Guest A): net1 (192.168.0.2)
  does not respond to ARP. net2 (192.168.1.2) responds normally.
- Three VFIO devices across two simultaneous guests (TC5): net3
  (192.168.2.2) responds intermittently.

For unreachable gateways, verify irqbypass via MSI-X vectors in
`/proc/interrupts` instead of ping. Three vectors per device (rx, tx,
misc) with non-zero interrupt counts confirm the irqbypass path is active.

## SSH Sessions

TC3–TC5 use SSH sessions to run kvmtool guests in the foreground of
independent terminal sessions. Set up SSH after host boot using the
pattern from `qemu-testing-kvm.md`. SSH port is 9997.

Open one SSH session per guest and run the bind+runkvm sequence in the
foreground of each session. This gives completely independent,
non-interleaved I/O per guest.

Use `searchwindowsize=8192` for SSH sessions (kernel dmesg lines can
arrive after the shell prompt and push it out of the search window).
Send `dmesg -n 1` immediately after the guest boots.

## QEMU Trace Command Encodings

The `riscv_iommu_cmd` trace event emits two 64-bit words per command:
`"%s: command 0x%x 0x%x"`. All fields are in dword0 unless noted.

All commands share the same field layout for opcode and func:
- `dword0 bits[6:0]`  = opcode
- `dword0 bits[9:7]`  = func

### IOTINVAL (opcode=1)

| func | Name | Key fields |
|------|------|-----------|
| 0 | IOTINVAL.VMA | AV=bit10, PSCV=bit32, GV=bit33, PSCID=bits[31:12], GSCID=bits[59:44] |
| 1 | IOTINVAL.GVMA | AV=bit10, GV=bit33, GSCID=bits[59:44] |

```python
# Decode IOTINVAL from dword0
def decode_iotinval(dw0):
    opcode = dw0 & 0x7f
    func   = (dw0 >> 7) & 0x7
    if opcode != 1:
        return None
    return {
        'type':  'IOTINVAL.GVMA' if func == 1 else 'IOTINVAL.VMA',
        'AV':    bool((dw0 >> 10) & 1),
        'PSCV':  bool((dw0 >> 32) & 1),
        'GV':    bool((dw0 >> 33) & 1),
        'PSCID': (dw0 >> 12) & 0xfffff,
        'GSCID': (dw0 >> 44) & 0xffff,
    }
```

### IOFENCE (opcode=2)

| func | Name | Key fields |
|------|------|-----------|
| 0 | IOFENCE.C | PR=bit12, PW=bit13, WSI=bit11, AV=bit10 |

```python
def is_iofence(dw0):
    return (dw0 & 0x7f) == 2 and ((dw0 >> 7) & 0x7) == 0
```

### IODIR (opcode=3)

| func | Name | Key fields |
|------|------|-----------|
| 0 | IODIR.INVAL_DDT | DV=bit33, DID=bits[63:40] |
| 1 | IODIR.INVAL_PDT | DV=bit33, DID=bits[63:40], PID=bits[31:12] |

`DV=1` means device-specific (DID field valid); `DV=0` means broadcast
(invalidate all cached DDT entries).

```python
def decode_iodir(dw0):
    opcode = dw0 & 0x7f
    func   = (dw0 >> 7) & 0x7
    if opcode != 3:
        return None
    return {
        'type': 'IODIR.INVAL_DDT' if func == 0 else 'IODIR.INVAL_PDT',
        'DV':   bool((dw0 >> 33) & 1),
        'DID':  (dw0 >> 40) & 0xffffff,
        'PID':  (dw0 >> 12) & 0xfffff,
    }
```

### Counting commands in trace analysis

```python
def count_commands(lines):
    """Count IOMMU commands by type from riscv_iommu_cmd trace lines."""
    import re
    counts = {'IOTINVAL.VMA': 0, 'IOTINVAL.GVMA': 0,
              'IOFENCE.C': 0, 'IODIR.INVAL_DDT': 0, 'IODIR.INVAL_PDT': 0}
    iodir_dv = 0   # device-specific IODIR.INVAL_DDT count

    for l in lines:
        m = re.search(r'riscv_iommu_cmd.*command (0x[0-9a-f]+)', l)
        if not m:
            continue
        dw0 = int(m.group(1), 16)
        opcode = dw0 & 0x7f
        func   = (dw0 >> 7) & 0x7

        if opcode == 1:
            counts['IOTINVAL.GVMA' if func == 1 else 'IOTINVAL.VMA'] += 1
        elif opcode == 2 and func == 0:
            counts['IOFENCE.C'] += 1
        elif opcode == 3:
            key = 'IODIR.INVAL_DDT' if func == 0 else 'IODIR.INVAL_PDT'
            counts[key] += 1
            if func == 0 and (dw0 >> 33) & 1:
                iodir_dv += 1

    return counts, iodir_dv
```

### Expected commands per test scenario

| Scenario | Expected commands |
|----------|------------------|
| Guest boot (VFIO attach) | IODIR.INVAL_DDT (DV=1 per device), IOTINVAL.VMA, IOTINVAL.GVMA, IOFENCE.C |
| Guest shutdown (VFIO detach) | IODIR.INVAL_DDT (DV=1 per device), IOTINVAL.VMA, IOTINVAL.GVMA |
| MSI table PTE update | IOTINVAL.GVMA (GV=1, GSCID=domain gscid) |
| vCPU migration | IOTINVAL.GVMA (GV=1) after each irq_set_vcpu_affinity call |
| DMA map/unmap | IOTINVAL.VMA (GV=1 when g-stage active) |

## QEMU Trace Analysis

QEMU IOMMU trace events are in the stderr log (`-trace riscv_iommu_*`).

Trace event formats (from `hw/riscv/trace-events`):
- `riscv_iommu_flt`: `"%s: fault %04x:%02x.%u reason: 0x%x iova: 0x%x"`
- `riscv_iommu_dma`: `"%s: translate %04x:%02x.%u #%u %s 0x%x -> 0x%x"`
- `riscv_iommu_msi`: `"%s: translate %04x:%02x.%u MSI 0x%x -> 0x%x"`
- `riscv_iommu_cmd`: `"%s: command 0x%x 0x%x"` (two 64-bit words)

BDF format in trace is `b:d.f` (e.g. `00:01.0`), not `0000:00:01.0`.

```python
def analyze_qemu_trace(trace_log, min_gscids=1):
    """
    Parse QEMU IOMMU trace log. Returns (passed, results).
    min_gscids: minimum distinct GSCID values expected in IOTINVAL.GVMA.
    """
    import re
    results = []
    passed = True

    with open(trace_log) as f:
        lines = f.read().splitlines()

    # Faults — always zero for a passing test
    faults = [l for l in lines if 'riscv_iommu_flt' in l]
    if faults:
        results.append(f'FAIL: {len(faults)} IOMMU fault(s):')
        for f in faults[:5]: results.append(f'  {f}')
        passed = False
    else:
        results.append('PASS: 0 IOMMU faults')

    # MSI translations per BDF (trace format: "b:d.f" e.g. "00:01.0")
    for bdf in ['00:01.0', '00:02.0', '00:03.0']:
        msi = [l for l in lines if 'riscv_iommu_msi' in l and bdf in l]
        results.append(f'{"PASS" if msi else "SKIP"}: '
                       f'{len(msi)} MSI translations for {bdf}')

    # Non-identity DMA translations (IOVA != HPA) prove both s-stage and
    # g-stage are active: IOVA --(s-stage)--> GPA --(g-stage)--> HPA.
    # MSI translations are separate: they use the MSI table to map IMSIC
    # GPA -> host VS-file HPA and do not go through the regular g-stage.
    non_id = []
    for l in lines:
        m = re.search(r'riscv_iommu_dma.*0x([0-9a-f]+) -> 0x([0-9a-f]+)', l)
        if m and m.group(1) != m.group(2):
            non_id.append(l)
    if non_id:
        results.append(f'PASS: {len(non_id)} non-identity DMA translations '
                       f'(s-stage and g-stage active)')
    else:
        results.append('FAIL: no non-identity DMA translations')
        passed = False

    # IOTINVAL.GVMA commands prove g-stage active
    # Decode from raw command words:
    #   dword0 bits[6:0] = opcode (1 = IOTINVAL)
    #   dword0 bits[9:7] = func   (1 = GVMA)
    #   dword0 bit[33]   = GV
    #   dword0 bits[59:44] = GSCID
    gscids = set()
    gvma_count = 0
    for l in lines:
        m = re.search(r'riscv_iommu_cmd.*command (0x[0-9a-f]+)', l)
        if not m:
            continue
        dw0 = int(m.group(1), 16)
        if (dw0 & 0x7f) == 1 and ((dw0 >> 7) & 0x7) == 1:
            gvma_count += 1
            if (dw0 >> 33) & 1:
                gscids.add((dw0 >> 44) & 0xffff)
    if gvma_count:
        results.append(f'PASS: {gvma_count} IOTINVAL.GVMA commands '
                       f'(g-stage active)')
    else:
        results.append('FAIL: no IOTINVAL.GVMA commands')
        passed = False
    if len(gscids) >= min_gscids:
        results.append(f'PASS: {len(gscids)} distinct GSCIDs: '
                       f'{sorted(gscids)}')
    else:
        results.append(f'FAIL: expected >={min_gscids} GSCIDs, '
                       f'found {sorted(gscids)}')
        passed = False

    # vCPU migration: same GPA translated to different HPAs across time
    msi_maps = {}
    for l in lines:
        m = re.search(
            r'riscv_iommu_msi.*MSI (0x[0-9a-f]+) -> (0x[0-9a-f]+)', l)
        if m:
            msi_maps.setdefault(m.group(1), set()).add(m.group(2))
    migrated = {g: h for g, h in msi_maps.items() if len(h) > 1}
    if migrated:
        results.append(f'PASS: vCPU migration confirmed — '
                       f'{len(migrated)} guest vCPU(s) show GPA->multiple-HPA')
        results.append(f'  (one GPA per guest vCPU IMSIC file; '
                       f'multiple HPAs = different host VS-files before/after migration)')
        for gpa, h in sorted(migrated.items()):
            results.append(f'  GPA {gpa} -> HPAs: {sorted(h)}')
    else:
        results.append('NOTE: no vCPU migration observed in trace')

    dma_count = len([l for l in lines if 'riscv_iommu_dma' in l])
    msi_count = len([l for l in lines if 'riscv_iommu_msi' in l])
    results.append(f'Trace stats: {msi_count} MSI, {dma_count} DMA, '
                   f'{gvma_count} GVMA')
    return passed, results
```

## MSI-X Verification

For devices where ping is unreliable (slirp limitation), verify irqbypass
via `/proc/interrupts`. Three MSI-X vectors per e1000e device (rx, tx,
misc) with non-zero interrupt counts confirm the irqbypass path is active.

```python
def check_msix_active(child, bdf_guest, label, prompt=r'/ #'):
    """
    Check /proc/interrupts for MSI-X vectors for the given guest BDF.
    bdf_guest: BDF as seen inside the guest (e.g. '0000:00:00.0').
    Returns True if >=1 PCI-MSIX entry found with non-zero interrupt count.
    """
    child.sendline(f'grep "PCI-MSIX-{bdf_guest}" /proc/interrupts')
    child.expect(prompt, timeout=30)
    lines = [l for l in child.before.splitlines()
             if 'PCI-MSIX' in l and bdf_guest in l]
    active = any(
        any(int(x) > 0 for x in re.findall(r'\b(\d+)\b', l)
            if x not in ('0', ''))
        for l in lines
    )
    count = len(lines)
    if active and count > 0:
        log(f'PASS: [{label}] MSI-X active ({count} vectors for {bdf_guest})')
        return True
    else:
        log(f'FAIL: [{label}] MSI-X not active for {bdf_guest}')
        return False
```

## Rootfs Health

Before running any test that launches kvmtool with VFIO, verify the
buildroot rootfs is not corrupted:

```bash
e2fsck -f -y /path/to/rootfs64.ext2
```

A corrupted rootfs causes kvmtool to SIGBUS in `copy_file()` when writing
to the generated guest rootfs via mmap. Symptom: kvmtool crashes with
SIGBUS on VFIO runs but works without VFIO. This is a rootfs issue, not
a kernel or irqbypass bug.

## dmesg Health Checks

Use the `check_dmesg_health()` function from `qemu-testing-kvm.md` for
both host and guest checks. This includes the broad error/warn/fail scan
with false-positive filtering. The `DMESG_FP` list may need extending for
irqbypass-specific benign messages — add them as discovered.

Known additional false positives for irqbypass tests:
- `kvm [1]: hypervisor extension not available` in guest dmesg — expected
  and correct; the guest is a KVM VM and does not itself run KVM.

### PTY buffer drain before post-guest host checks

In TC4/TC5, the main host console pexpect session (`child`) is idle while
SSH sessions interact with guest consoles. During this time the host
kernel prints VFIO/e1000e teardown messages to ttyS0 (even with
`dmesg -n 1` set), which accumulate as unread PTY output. When `child` is
next used, those buffered bytes land in `child.before` for the first
`run()` call, making them appear as output of the BUG grep — a false
positive.

**Fix**: drain the PTY buffer immediately before calling
`check_dmesg_health()` on the host console after SSH guests have shut down:

```python
def drain_buffer(child, prompt=HOST_PROMPT, timeout=5):
    """Drain buffered PTY content accumulated while child was idle."""
    child.sendline('')
    try:
        child.expect(prompt, timeout=timeout)
    except pexpect.TIMEOUT:
        pass

# After ssh_a and ssh_b have exited:
drain_buffer(child)
h_ok, h_res = check_dmesg_health(child, HOST_PROMPT, 'Host(post)', HOST_DMESG_FP)
```

Apply `drain_buffer()` any time `child` has been idle during SSH session
interactions before the next health check.


## Migration Testing

### Host IRQ Migration (irq_set_affinity path)

Force host IRQ migration to exercise `riscv_iommu_ir_irq_set_affinity()`.
The e1000e MSI-X vectors appear in `/proc/interrupts`; their IRQ numbers
can be found by grepping for the BDF.

**Key implementation notes:**

- Use `smp_affinity_list` (CPU list format, e.g. `echo 2`) not `smp_affinity`
  (hex bitmask). Writing to `smp_affinity` returns `EINVAL` for these IRQs
  due to how the IMSIC irqchip handles the hex mask format.
- Migration is deferred (`IRQCHIP_MOVE_DEFERRED`): the actual move completes
  when the interrupt fires on the old CPU and `__imsic_local_sync()` runs.
  Poll `effective_affinity_list` while generating traffic until the move
  completes — typically 1-3 polls of 5 pings each.
- Focus on the RX IRQ (first MSI-X vector, highest traffic) — it migrates
  reliably. TX and misc IRQs may not generate enough traffic to trigger the
  deferred move.

```python
def get_effective_cpu(child, irq, prompt):
    """Return current effective CPU for this IRQ (integer)."""
    child.sendline(f'cat /proc/irq/{irq}/effective_affinity_list')
    child.expect(prompt, timeout=5)
    val = child.before.strip().splitlines()[-1].strip() if child.before.strip() else ''
    try:
        return int(val.split('-')[0])
    except ValueError:
        return -1

def migrate_irq_and_wait(child, irq, target_cpu, prompt, max_polls=10):
    """
    Set smp_affinity_list to target_cpu, then poll effective_affinity_list
    while generating traffic until the deferred move completes.
    Returns (success, final_cpu, polls_taken).
    """
    import time
    # Write target CPU using smp_affinity_list (not smp_affinity)
    child.sendline(f'echo {target_cpu} > /proc/irq/{irq}/smp_affinity_list 2>&1; echo rc=$?')
    child.expect(prompt, timeout=5)
    rc_line = [l for l in child.before.splitlines() if 'rc=' in l]
    if not rc_line or rc_line[0].strip() != 'rc=0':
        return False, -1, 0

    for poll in range(max_polls):
        # Generate traffic to trigger the deferred move on the old CPU
        child.sendline('ping -c 5 -I eth1 192.168.0.2')
        child.expect(prompt, timeout=15)
        time.sleep(0.5)

        eff_cpu = get_effective_cpu(child, irq, prompt)
        if eff_cpu == target_cpu:
            return True, eff_cpu, poll + 1

    return False, get_effective_cpu(child, irq, prompt), max_polls
```

**Verification after migration**:
1. `effective_affinity_list` shows target CPU
2. Interrupt counts appear on target CPU in `/proc/interrupts`
3. Ping continues with 0% packet loss
4. No errors/warnings in dmesg
5. QEMU trace: same GPA → different HPA confirms IOMMU MSI table updated

### Guest vCPU Migration (irq_set_vcpu_affinity path)

Force guest vCPU migration using `taskset` on the host to exercise
`kvm_riscv_vcpu_irq_update()` and the IOMMU MSI table update path.

Use a separate SSH session to the host to run `taskset -cp <cpu> <pid>`
while the guest runs in the foreground of the main pexpect session.
See the SSH setup pattern in `qemu-testing-kvm.md`.

```python
def migrate_guest_vcpu_via_ssh(ssh_port, from_cpu, to_cpu, back_cpu):
    """Migrate kvmtool vCPUs via SSH while guest runs in foreground."""
    import subprocess, time
    results = []

    def ssh(cmd):
        return subprocess.run(
            ['ssh', '-o', 'StrictHostKeyChecking=no',
             '-o', 'UserKnownHostsFile=/dev/null',
             '-o', 'ConnectTimeout=5',
             '-p', str(ssh_port), 'root@localhost', cmd],
            capture_output=True, text=True, timeout=15
        )

    # Find kvmtool PID using ps (pgrep not available in buildroot)
    pid = None
    for attempt in range(8):
        time.sleep(2)
        r = ssh("ps | grep lkvm-static | grep -v grep | awk '{ print $1 }'")
        if r.returncode == 0 and r.stdout.strip():
            pid = r.stdout.strip().split()[0]
            break
    if not pid:
        results.append('FAIL: lkvm-static process not found via SSH')
        return False, results
    results.append(f'kvmtool PID: {pid}')

    # Migrate to_cpu
    r = ssh(f'taskset -cp {to_cpu} {pid}')
    results.append(f'Migrated to CPU{to_cpu}: {r.stdout.strip()}')
    time.sleep(2)  # Allow migration + IOMMU table update

    # Migrate back
    r = ssh(f'taskset -cp {back_cpu} {pid}')
    results.append(f'Migrated back to CPU{back_cpu}: {r.stdout.strip()}')
    time.sleep(2)

    return True, results
```

**SSH setup for vCPU migration**: start sshd with pubkey auth after host boot:
```python
child.sendline('killall sshd 2>/dev/null')
child.expect(HOST_PROMPT, timeout=5)
child.sendline('/usr/sbin/sshd -o PermitRootLogin=yes '
               '-o PasswordAuthentication=no -o StrictModes=no &')
child.expect(HOST_PROMPT, timeout=5)
time.sleep(2)
```

**Verification after vCPU migration**:
1. QEMU trace: same GPA → different HPA (vCPU migration confirmed)
2. Guest dmesg: no new errors/warnings after migration
3. Host dmesg: no IOMMU errors after migration
4. Guest still receiving interrupts (MSI-X counts active)

## Two Test Paths

The irqbypass series supports two device assignment paths:

### Legacy VFIO Container Path (TC1–TC5)

- VMM: kvmtool (`--vfio-pci=<bdf>`)
- Kernel path: `vfio_iommu_type1` → `iommu_domain` → IOMMU driver
- `IRQ_DOMAIN_FLAG_ISOLATED_MSI`: **not set**
- `allow_unsafe_interrupts=1`: **required** in kernel cmdline
- Status: **testable now** with current kvmtool

### iommufd Cdev Path (TC6 — deferred)

- VMM: any VMM with iommufd support (e.g. QEMU `-device vfio-pci` with
  iommufd backend, or future kvmtool with iommufd support)
- Kernel path: `vfio_iommufd_physical_bind()` → iommufd → IOMMU driver
- `IRQ_DOMAIN_FLAG_ISOLATED_MSI`: **set lazily** when
  `IOMMU_CAP_VIRT_MSI_ISOLATION` is reported and `vdev->kvm != NULL`
- `allow_unsafe_interrupts=1`: **not required**
- Status: **deferred** — kvmtool does not yet support iommufd

## Test Cases

### TC1 — Basic irqbypass boot and network (5 iterations, full boot each)

**What it tests**: end-to-end irqbypass path with one VFIO e1000e device.
IOMMU MSI table populated, MSI-X functional in guest, vCPU migration handled.

**Per-iteration sequence**:
1. Boot host. Verify host eth1 (`udhcpc` + ping 192.168.0.2).
2. Bind 0000:00:01.0 to vfio-pci.
3. Launch guest: `/share/bin/runkvm --vfio-pci=0000:00:01.0`.
4. In guest: `dmesg -n 1`, static IP on eth0. Poll for `LOWER_UP` before
   pinging — the VFIO device link takes a moment to come up (see
   "Link-up Timing"). Then ping 192.168.0.2.
5. Check MSI-X: `grep "PCI-MSIX-0000:00:00.0" /proc/interrupts`
   (use `timeout=30` — WSL2 scheduling can stall PTY delivery).
6. Shutdown guest (`poweroff -f`). Unbind device. Shutdown host.

**Pass criteria**:
- Host eth1 ping: 0% loss
- Guest eth0 ping: 0% loss (one VFIO device — slirp responds normally)
- MSI-X active in guest
- QEMU trace: 0 faults, MSI translations for 00:01.0, non-identity DMA,
  IOTINVAL.GVMA present, vCPU migration confirmed

### TC2 — VFIO rebind cycle (5 iterations, single host boot)

**What it tests**: e1000e correctly cycles between host e1000e driver and
vfio-pci. Tests IODIR.INVAL_DDT, domain teardown/rebuild.

**Per-iteration sequence**:
1. Verify host eth1 (ping 192.168.0.2).
2. Bind 0000:00:01.0 to vfio-pci. Launch guest. Verify guest eth0 ping.
3. Shutdown guest (`poweroff -f`).
4. Unbind 0000:00:01.0 from vfio-pci. Wait 2s.
5. Verify host e1000e reprobed (dmesg shows `eth1: Intel(R) PRO/1000`).
6. Re-acquire DHCP lease (`udhcpc -i eth1`) — IP is lost during the vfio cycle.
7. Verify host eth1 ping again.

**Pass criteria per iteration**: host eth1 ping OK before and after,
guest eth0 ping OK, e1000e reprobed.
**Cumulative trace criteria**: 0 faults, IODIR.INVAL_DDT commands present.

### TC3 — Multiple e1000e devices in one guest (5 iterations, full boot each)

**What it tests**: two e1000e devices simultaneously assigned to one guest.
Two independent IOMMU domains, MSI tables, and irqbypass paths.

**Setup**: SSH session for the guest (kvmtool foreground in SSH).

**Per-iteration sequence**:
1. Boot host. Verify host eth1 and eth2 (udhcpc + ping).
2. Bind 0000:00:01.0 and 0000:00:02.0 to vfio-pci.
3. Open SSH session. Launch guest:
   `/share/bin/runkvm --vfio-pci=0000:00:01.0 --vfio-pci=0000:00:02.0`.
4. In guest: `dmesg -n 1`.
5. eth0 (0000:00:01.0): static IP 192.168.0.15/24. Verify MSI-X.
   Ping not used — slirp net1 unreliable with two VFIO devices.
6. eth1 (0000:00:02.0): static IP 192.168.1.15/24. Verify MSI-X and
   best-effort ping 192.168.1.2.
7. Shutdown guest (`poweroff -f`). Unbind both devices. Verify reclaimed.

**Pass criteria**: both devices MSI-X active, eth1 ping best-effort,
QEMU trace: 0 faults, MSI for both BDFs, >=2 distinct GSCIDs.

### TC4 — One e1000e per guest, two simultaneous guests (3 iterations)

**What it tests**: two KVM guests running simultaneously, each with one
VFIO e1000e. Independent IOMMU domains, no cross-contamination.

**Setup**: host_console + ssh_a + ssh_b.

**Per-iteration sequence**:
1. Boot host. Verify host eth1 and eth2 (udhcpc + ping).
2. Bind 0000:00:01.0 to vfio-pci. Open ssh_a. Launch Guest A:
   `/share/bin/runkvm --vfio-pci=0000:00:01.0`.
3. Bind 0000:00:02.0 to vfio-pci. Open ssh_b. Launch Guest B:
   `/share/bin/runkvm --vfio-pci=0000:00:02.0`.
4. Guest A: `dmesg -n 1`, static IP 192.168.0.15/24 on eth0,
   ping 192.168.0.2.
5. Guest B: `dmesg -n 1`, static IP 192.168.1.15/24 on eth0,
   ping 192.168.1.2.
6. Concurrent pings: Guest A `ping -c 10 192.168.0.2`,
   Guest B `ping -c 10 192.168.1.2`. Both must complete 0% loss.
7. Shutdown both guests. Unbind both devices. Verify reclaimed.

**Pass criteria**: both guest pings OK, concurrent pings OK,
QEMU trace: 0 faults, MSI for both BDFs, >=2 distinct GSCIDs.

### TC5 — Multiple devices in multiple simultaneous guests (3 iterations)

**What it tests**: Guest A has two e1000e devices; Guest B has one.
Maximum stress test of concurrent MSI table management and vCPU migration.

**Setup**: host_console + ssh_a + ssh_b.

**Per-iteration sequence**:
1. Boot host. Verify host eth1, eth2, eth3 (udhcpc + ping).
2. Bind 0000:00:01.0 and 0000:00:02.0. Open ssh_a. Launch Guest A:
   `/share/bin/runkvm --vfio-pci=0000:00:01.0 --vfio-pci=0000:00:02.0`.
3. Bind 0000:00:03.0. Open ssh_b. Launch Guest B:
   `/share/bin/runkvm --vfio-pci=0000:00:03.0`.
4. Guest A: `dmesg -n 1`.
   - eth0 (0000:00:01.0): static IP 192.168.0.15/24. Verify MSI-X.
     Ping not used — slirp net1 unreliable with two VFIO devices.
   - eth1 (0000:00:02.0): static IP 192.168.1.15/24. Wait for link up
     before pinging — the e1000e link takes a moment after rebind:
     ```python
     for _ in range(5):
         out = run(guest, "ip link show eth1", GUEST_PROMPT)
         if "UP" in out and "LOWER_UP" in out: break
         time.sleep(1)
     ```
     Then ping 192.168.1.2.
5. Guest B: `dmesg -n 1`.
   - eth0 (0000:00:03.0): static IP 192.168.2.15/24. Verify MSI-X.
     Ping best-effort — slirp net3 unreliable with multiple VFIO guests.
6. Concurrent pings:
   - Guest A: `ping -c 10 192.168.1.2` (must pass).
   - Guest B: `ping -c 10 192.168.2.2` (best-effort; log, not hard fail).
7. Shutdown both guests. Unbind all three devices. Verify reclaimed.

**Pass criteria**:
- All three host e1000e pings OK before and after
- Guest A eth0: MSI-X active; ping not used (slirp unreliable)
- Guest A eth1: ping 192.168.1.2 OK
- Guest B eth0: MSI-X active; ping best-effort
- Guest A eth1 concurrent ping (10 packets): 0% loss
- Guest B concurrent ping: best-effort (log result)
- QEMU trace: 0 faults, MSI for all three BDFs, >=3 distinct GSCIDs,
  no cross-domain interference

### TC6 — iommufd cdev path (deferred)

**What it tests**: device assignment via the iommufd cdev path. Verifies
that `IRQ_DOMAIN_FLAG_ISOLATED_MSI` is set lazily by
`vfio_iommufd_physical_bind()` when `IOMMU_CAP_VIRT_MSI_ISOLATION` is
reported, and that irqbypass works without `allow_unsafe_interrupts=1`.

**Prerequisites**:
- A VMM with iommufd support (kvmtool with iommufd, or QEMU with nested
  RISC-V KVM and `-device vfio-pci`)
- `allow_unsafe_interrupts=1` must NOT be in the kernel cmdline

**Verification**:
- dmesg: no `allow_unsafe_interrupts` warning from VFIO
- dmesg: `vfio-pci 0000:00:01.0: irq bypass producer registered`
- QEMU trace: same as TC1 (MSI translations, IOTINVAL.GVMA, vCPU migration)

**Status**: deferred — implement when a suitable VMM is available.

## Updating Test Scripts

Follow the process defined in `qemu-testing.md` ("Updating Test Scripts"):
delete existing scripts, update the agent, regenerate. Never patch scripts
directly.

## Task

1. Ask the user which test cases to run (TC1–TC5, TC6, or all).
   Default: TC1–TC5 (TC6 is deferred — skip unless a suitable VMM
   with iommufd support is available).
2. Ask for paths: `qemu_bin`, `kernel`, `rootfs`, `share_dir`.
   Verify `<share_dir>/bin/lkvm-static` exists and has AIA support
   (`strings lkvm-static | grep -c disable-ssaia` should return 1).
   If missing or no AIA support, follow the "Cross-Compiling kvmtool"
   procedure in `qemu-testing-kvm.md` to build a new binary.
   Verify or create `<share_dir>/bin/runkvm` and
   `<share_dir>/runqemu/Image` from the templates above.
3. Check rootfs health: `e2fsck -f -y <rootfs>`.
4. Verify SSH key for TC3–TC5 (`~/.ssh/id_ed25519`).
5. Write one Python test script per requested test case, using the
   patterns from this file and from `qemu-testing.md` /
   `qemu-testing-kvm.md`. Name the scripts `tc1_basic_boot.py` through
   `tc5_multi_device_multi_guest.py`.
6. Run each script in order. Report PASS/FAIL after each.
7. Print a summary table:

```
TC1: PASS  (5/5 iterations)
TC2: PASS  (5/5 iterations)
TC3: PASS  (5/5 iterations)
TC4: PASS  (3/3 iterations)
TC5: PASS  (3/3 iterations)
TC6: DEFERRED (iommufd VMM not available)
Overall: PASS
```

## Output Per Test Case

```
=== TC<N>: <name> ===
Iteration 1: PASS / FAIL
  ...
Trace analysis: PASS / FAIL
  0 IOMMU faults
  <N> MSI translations for 00:01.0
  ...
Overall: PASS / FAIL (<M>/<total> iterations passed)
Log: /tmp/tc<N>_results.log
```
