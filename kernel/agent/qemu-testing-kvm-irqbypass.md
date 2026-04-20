---
name: qemu-kvm-irqbypass-tester
description: Generates and runs the full RISC-V IOMMU irqbypass test suite (TC1–TC5). Builds on qemu-testing.md and qemu-testing-kvm.md. Requires a QEMU binary, kernel Image, buildroot rootfs, and the share64 directory containing lkvm-static and helper scripts.
tools: Bash, Read, Write
model: sonnet
---

# QEMU KVM irqbypass Test Agent

**Read `qemu-testing.md` and `qemu-testing-kvm.md` in that order first.**
Those files define the QEMU command construction, pexpect discipline, login
detection, SSH setup, dmesg health checks, kvmtool invocation, and shutdown
patterns that this agent builds on. This file only documents what is new:
the irqbypass-specific QEMU configuration, network topology, VFIO device
management, QEMU trace analysis, and the TC1–TC5 test sequences.

## Required Inputs

- **QEMU binary**: `qemu-system-riscv64`
- **Kernel Image**: RISC-V 64-bit kernel built with KVM, VFIO, RISCV_IOMMU,
  and e1000e as builtins; AIA support enabled. If the CWD is a Linux kernel
  source tree, auto-detect from `build/arch/riscv/boot/Image` per
  `qemu-testing.md`. If no build exists, build one using the minimal kernel
  procedure in `qemu-testing.md` with the irqbypass config overrides below.
- **Rootfs**: buildroot ext2 image; root login, no password
- **Share directory**: directory shared into the guest via virtio-9p at `/share`,
  containing:
  - `bin/lkvm-static` — statically linked kvmtool binary (RISC-V 64-bit,
    built with AIA support)
  - `bin/runkvm-n` — launches a kvmtool guest with one or more VFIO e1000e
    devices (see below)
  - `bin/e1000e-unbind` — unbinds 0000:00:01.0 from vfio-pci, returns to
    e1000e driver
  - `bin/e1000e-unbind-all` — unbinds all three e1000e devices
  - `runqemu/Image` — copy of the kernel Image (copied by runqemu-riscv
    before boot, or copy manually)

## QEMU Configuration for irqbypass

The QEMU machine must have AIA, IOMMU, and three e1000e devices on separate
user-net backends. Build the command using the `build_qemu_cmd()` pattern
from `qemu-testing.md` with these settings:

```python
cfg = {
    'machine':       'virt',
    'machine_extra': ',aia=aplic-imsic,aia-guests=5,iommu-sys=on',
    'cpu':           'max',
    'smp':           6,
    'mem':           '4G',
    'cmdline':       ('root=/dev/vda console=ttyS0 earlycon '
                      'ignore_loglevel debug ftrace_dump_on_oops '
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

QEMU stderr carries both IOMMU trace events and QEMU error messages.
Redirect to a file: `2>/tmp/qemu_trace.log`.

## Kernel Config for irqbypass Testing

When building the kernel from source, apply these overrides after
`make defconfig` and before the final `make olddefconfig` (Step 3 of the
"Building a Minimal Kernel" procedure in `qemu-testing.md`):

```bash
./scripts/config --file build/.config \
    -e KVM \
    -e VFIO \
    -e VFIO_IOMMU_TYPE1 \
    -e VFIO_PCI \
    -e E1000E \
    -e RISCV_IOMMU

make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- O=build olddefconfig
```

These symbols must all be `=y` (builtin), not `=m` (module):

| Symbol | Purpose |
|--------|---------|
| `KVM` | KVM hypervisor — required for kvmtool guests |
| `VFIO` | VFIO framework — required for device passthrough |
| `VFIO_IOMMU_TYPE1` | VFIO IOMMU backend — required for VFIO DMA mapping |
| `VFIO_PCI` | VFIO PCI driver — binds e1000e for passthrough |
| `E1000E` | Intel e1000e NIC driver — the device under test |
| `RISCV_IOMMU` | RISC-V IOMMU driver — the driver under test |

The RISC-V defconfig has `KVM=m`, `VFIO=m`, `VFIO_PCI=m` — all must be
promoted to `=y`. `RISCV_IOMMU` is not in the defconfig at all and must
be added. `E1000E=y` is already correct in the defconfig.

Verify the final config before building:
```bash
grep -E "^CONFIG_KVM=|^CONFIG_VFIO=|^CONFIG_VFIO_IOMMU_TYPE1=|^CONFIG_VFIO_PCI=|^CONFIG_E1000E=|^CONFIG_RISCV_IOMMU=" build/.config
```
Expected:
```
CONFIG_KVM=y
CONFIG_VFIO=y
CONFIG_VFIO_IOMMU_TYPE1=y
CONFIG_VFIO_PCI=y
CONFIG_E1000E=y
CONFIG_RISCV_IOMMU=y
```


## Share Directory Scripts

The share directory must contain these shell scripts in `bin/`. If they
are missing, generate them as part of test setup. All scripts run inside
the QEMU guest (they are POSIX sh, not bash-specific).

**`bin/runkvm`** — launch a single kvmtool guest with 0000:00:01.0 via VFIO:

```sh
#!/bin/sh
CPUS=2
MEM=256
IMAGE=/share/runqemu/Image

echo 0000:00:01.0 > /sys/bus/pci/devices/0000:00:01.0/driver/unbind
echo vfio-pci > /sys/bus/pci/devices/0000:00:01.0/driver_override
echo 0000:00:01.0 > /sys/bus/pci/drivers_probe

lkvm-static run \
    -m "${MEM}" \
    -c "${CPUS}" \
    -p "console=ttyS0 earlycon ${CMDLINE}" \
    -k "${IMAGE}" \
    --debug \
    --vfio-pci=0000:00:01.0 \
    "$@"
```

**`bin/runkvm-n`** — launch a kvmtool guest with one or more VFIO devices:

```sh
#!/bin/sh
# Usage: runkvm-n <bdf1> [bdf2 ...]
CPUS=2
MEM=256
IMAGE=/share/runqemu/Image
VFIO_ARGS=""

for bdf in "$@"; do
    echo $bdf > /sys/bus/pci/devices/$bdf/driver/unbind 2>/dev/null
    echo vfio-pci > /sys/bus/pci/devices/$bdf/driver_override
    echo $bdf > /sys/bus/pci/drivers_probe
    VFIO_ARGS="$VFIO_ARGS --vfio-pci=$bdf"
done

lkvm-static run \
    -m "${MEM}" \
    -c "${CPUS}" \
    -p "console=ttyS0 earlycon ${CMDLINE}" \
    -k "${IMAGE}" \
    --debug \
    ${VFIO_ARGS}
```

No rootfs argument — kvmtool generates a minimal rootfs automatically in
`~/.kvm/`. The host's `/` is shared into the guest at `/host`.

**`bin/e1000e-unbind`** — unbind 0000:00:01.0 from vfio-pci and return it
to the e1000e driver:

```sh
#!/bin/sh
echo 0000:00:01.0 > /sys/bus/pci/devices/0000:00:01.0/driver/unbind
echo > /sys/bus/pci/devices/0000:00:01.0/driver_override
echo 0000:00:01.0 > /sys/bus/pci/drivers_probe
```

**`bin/e1000e-unbind-all`** — unbind all three e1000e devices:

```sh
#!/bin/sh
for bdf in 0000:00:01.0 0000:00:02.0 0000:00:03.0; do
    if [ -e /sys/bus/pci/devices/$bdf/driver ]; then
        echo $bdf > /sys/bus/pci/devices/$bdf/driver/unbind 2>/dev/null
    fi
    echo > /sys/bus/pci/devices/$bdf/driver_override
    echo $bdf > /sys/bus/pci/drivers_probe
done
```

**VFIO bind/unbind mechanics** (used by all scripts above):
- `driver/unbind` — detach the device from its current driver
- `driver_override` — set to `vfio-pci` to bind to VFIO, or empty to
  clear the override and allow the kernel to select the correct driver
- `drivers_probe` — trigger driver binding; the kernel re-probes the
  device and binds it to the overridden driver (or the best match if
  override is cleared)

After `e1000e-unbind` or `e1000e-unbind-all`, wait ~2 seconds for the
e1000e driver to reprobe. Verify with:
```bash
dmesg | tail -10 | grep -iE "e1000e.*eth[123]"
```
Expected: `e1000e 0000:00:0N.0 ethN: Intel(R) PRO/1000 Network Connection`

## Network Topology

```
Host eth0 (virtio-net):  10.0.2.x/24  — management, DHCP, SSH via :9997
Host eth1 (e1000e net1): 192.168.0.x  — VFIO candidate, gw 192.168.0.2
Host eth2 (e1000e net2): 192.168.1.x  — VFIO candidate, gw 192.168.1.2
Host eth3 (e1000e net3): 192.168.2.x  — VFIO candidate, gw 192.168.2.2
```

**Host e1000e interfaces**: use `udhcpc` (QEMU slirp responds on the host
side when devices are not in VFIO mode).

**Guest e1000e interfaces**: use static IP — no DHCP server on 192.168.x
subnets. Assign with `ip addr add`:
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

The test descriptions in `irqbypass-tests.txt` show guests launched with
`&` (background) and interacted with via the host console. In practice
this is unworkable because both guest consoles interleave on the same
terminal. The correct implementation is to open one SSH session per guest
and run `runkvm-n` in the foreground of each session. This gives
completely independent, non-interleaved I/O per guest.

Each SSH session spawns `runkvm-n <bdf...>` and waits for `/ #`.
After the guest boots, send `dmesg -n 1` to suppress console noise.

## QEMU Trace Analysis

QEMU IOMMU trace events are in the stderr log. Key checks:

```python
def analyze_qemu_trace(trace_log, min_gscids=1):
    """
    Returns (passed, results) dict.
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
        results.append(f'FAIL: {len(faults)} IOMMU fault(s)')
        passed = False
    else:
        results.append('PASS: 0 IOMMU faults')

    # MSI translations per BDF (QEMU format: "0000:01.0", no middle octet)
    for bdf in ['0000:01.0', '0000:02.0', '0000:03.0']:
        msi = [l for l in lines if 'riscv_iommu_msi' in l and bdf in l]
        results.append(f'{"PASS" if msi else "SKIP"}: '
                       f'{len(msi)} MSI translations for {bdf}')

    # Non-identity DMA translations prove s-stage page table walk
    non_id = [l for l in lines if 'riscv_iommu_dma' in l
              and re.search(r'(0x[89a-f][0-9a-f]{7,})\s+->\s+(?!\1)', l)]
    if non_id:
        results.append(f'PASS: {len(non_id)} non-identity DMA translations (s-stage active)')
    else:
        results.append('FAIL: no non-identity DMA translations')
        passed = False

    # IOTINVAL.GVMA commands prove g-stage active
    # Decode from raw command words: opcode=1 bits[6:0], func=1 bits[9:7],
    # GV=bit33, GSCID=bits[59:44]
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
        results.append(f'PASS: {gvma_count} IOTINVAL.GVMA commands (g-stage active)')
    else:
        results.append('FAIL: no IOTINVAL.GVMA commands')
        passed = False
    if len(gscids) >= min_gscids:
        results.append(f'PASS: {len(gscids)} distinct GSCIDs: {sorted(gscids)}')
    else:
        results.append(f'FAIL: expected >={min_gscids} GSCIDs, found {sorted(gscids)}')
        passed = False

    # vCPU migration: same GPA translated to different HPAs
    msi_maps = {}
    for l in lines:
        m = re.search(r'riscv_iommu_msi.*MSI (0x[0-9a-f]+) -> (0x[0-9a-f]+)', l)
        if m:
            msi_maps.setdefault(m.group(1), set()).add(m.group(2))
    migrated = {g: h for g, h in msi_maps.items() if len(h) > 1}
    if migrated:
        results.append(f'PASS: vCPU migration confirmed '
                       f'({len(migrated)} GPA(s) with multiple HPAs)')
    else:
        results.append('NOTE: no vCPU migration observed')

    dma = len([l for l in lines if 'riscv_iommu_dma' in l])
    msi = len([l for l in lines if 'riscv_iommu_msi' in l])
    results.append(f'Trace stats: {msi} MSI, {dma} DMA, {gvma_count} GVMA')
    return passed, results
```

## MSI-X Verification

For devices where ping is unreliable (slirp limitation), verify irqbypass
via `/proc/interrupts`:

```python
def msix_active(child, bdf_guest, label, prompt=r'/ #'):
    """
    Check /proc/interrupts for MSI-X vectors for the given guest BDF.
    Returns True if >=1 PCI-MSIX entry found for that BDF.
    """
    child.sendline(f'grep -c "PCI-MSIX-{bdf_guest}" /proc/interrupts')
    child.expect(prompt, timeout=10)
    out = child.before
    nums = re.findall(r'\b(\d+)\b', out)
    count = int(nums[-1]) if nums else 0
    if count > 0:
        log(f'[{label}] MSI-X active ({count} vectors for {bdf_guest}): PASS')
        return True
    else:
        log(f'[{label}] MSI-X not active for {bdf_guest}: FAIL')
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
SIGBUS on VFIO runs but works without VFIO.

## ftrace Setup

Mount debugfs and enable tracing after host boot for TC1 and TC2:

```python
child.sendline('mount -t debugfs none /sys/kernel/debug 2>/dev/null')
child.expect(HOST_PROMPT, timeout=10)
child.sendline('echo 65536 > /sys/kernel/debug/tracing/buffer_size_kb')
child.expect(HOST_PROMPT, timeout=10)
child.sendline('echo 1 > /sys/kernel/debug/tracing/tracing_on')
child.expect(HOST_PROMPT, timeout=10)
```

Use 64MB buffer — VFIO DMA maps for 256MB guest RAM generate ~65536
`iommu_map_nosync` calls which fill a small buffer.

Collect after all iterations:
```python
child.sendline('echo 0 > /sys/kernel/debug/tracing/tracing_on')
child.expect(HOST_PROMPT, timeout=10)
child.sendline('cat /sys/kernel/debug/tracing/trace | grep msitbl_inval')
child.expect(HOST_PROMPT, timeout=30)
log(f'ftrace msitbl_inval:\n{child.before}')
```

## Test Cases

### TC1 — Basic irqbypass boot and network (5 iterations, full boot each)

**What it tests**: end-to-end irqbypass path with one VFIO e1000e device.
IOMMU MSI table populated, MSI-X functional in guest, vCPU migration handled.

**Setup**: standard QEMU config above. One guest per iteration.

**Per-iteration sequence**:
1. Boot host. Verify host eth1 (`udhcpc` + ping 192.168.0.2).
2. Launch guest: `PATH=/share/bin:$PATH runkvm` (uses `runkvm` which
   binds 0000:00:01.0 to vfio-pci and launches kvmtool).
3. In guest: `dmesg -n 1`, bring up eth0 with `udhcpc`, ping 192.168.0.2.
4. Check `/proc/interrupts` for `PCI-MSIX-0000:00:00.0` entries.
5. Shutdown guest (`poweroff -f`), shutdown host.

**Pass criteria**:
- Host eth1 ping: 0% loss
- Guest eth0 ping: 0% loss
- QEMU trace: 0 faults, MSI translations present, non-identity DMA,
  IOTINVAL.GVMA present, vCPU migration confirmed

**Note**: TC1 uses `runkvm` (not `runkvm-n`) which is hardcoded to
0000:00:01.0. With one VFIO device, slirp responds normally and udhcpc
works in the guest.

### TC2 — VFIO rebind cycle (5 iterations, single host boot)

**What it tests**: e1000e correctly cycles between host e1000e driver and
vfio-pci. Tests IODIR.INVAL_DDT, domain teardown/rebuild.

**Setup**: host booted once; 5 iterations within the same host session.
Single QEMU trace log covering all iterations.

**Per-iteration sequence**:
1. Verify host eth1 (ping 192.168.0.2).
2. Launch guest with `runkvm`. Verify guest eth0 ping.
3. Shutdown guest (`poweroff -f`).
4. Unbind: `PATH=/share/bin:$PATH e1000e-unbind`.
5. Verify host e1000e reprobed (dmesg shows `eth1: Intel(R) PRO/1000`).
6. Verify host eth1 ping again.

**Pass criteria per iteration**: host eth1 ping OK before and after,
guest eth0 ping OK, e1000e reprobed.
**Cumulative trace criteria**: 0 faults, IODIR.INVAL_DDT commands present.

### TC3 — Multiple e1000e devices in one guest (5 iterations, full boot each)

**What it tests**: two e1000e devices simultaneously assigned to one guest.
Two independent IOMMU domains, MSI tables, and irqbypass paths.

**Setup**: SSH session for the guest (kvmtool foreground in SSH).
Set up SSH after host boot.

**Per-iteration sequence**:
1. Boot host. Verify host eth1 and eth2 (udhcpc + ping).
2. Open SSH session. Launch guest: `runkvm-n 0000:00:01.0 0000:00:02.0`.
3. In guest: `dmesg -n 1`.
4. Assign static IPs: `ip addr add 192.168.0.15/24 dev eth0`,
   `ip addr add 192.168.1.15/24 dev eth1`.
5. Verify eth0 via MSI-X (`grep -c PCI-MSIX-0000:00:00.0 /proc/interrupts`).
6. Verify eth1 via MSI-X and best-effort ping 192.168.1.2.
7. Shutdown guest (`poweroff -f`). Unbind: `e1000e-unbind-all`.
8. Verify host eth1 and eth2 reclaimed.

**Pass criteria**: both devices MSI-X active, eth1 ping best-effort,
QEMU trace: 0 faults, MSI for both BDFs, >=2 distinct GSCIDs.

### TC4 — One e1000e per guest, two simultaneous guests (3 iterations)

**What it tests**: two KVM guests running simultaneously, each with one
VFIO e1000e. Independent IOMMU domains, no cross-contamination.

**Setup**: host_console + ssh_a + ssh_b. Set up SSH after host boot.

**Per-iteration sequence**:
1. Boot host. Verify host eth1 and eth2 (udhcpc + ping).
2. Open ssh_a. Launch Guest A: `runkvm-n 0000:00:01.0`.
3. Open ssh_b. Launch Guest B: `runkvm-n 0000:00:02.0`.
4. Guest A: `dmesg -n 1`, static IP `192.168.0.15/24` on eth0, ping 192.168.0.2.
5. Guest B: `dmesg -n 1`, static IP `192.168.1.15/24` on eth0, ping 192.168.1.2.
6. Concurrent pings: Guest A `ping -c 10 -I eth0 192.168.0.2`,
   Guest B `ping -c 10 -I eth0 192.168.1.2`. Both must complete 0% loss.
7. Shutdown Guest A and Guest B (`poweroff -f`). Unbind: `e1000e-unbind-all`.
8. Verify host eth1 and eth2 reclaimed.

**Pass criteria**: both guest pings OK, concurrent pings OK,
QEMU trace: 0 faults, MSI for both BDFs, >=2 distinct GSCIDs.

**Note**: one VFIO device per guest — slirp responds normally, ping works.

### TC5 — Multiple devices in multiple simultaneous guests (3 iterations)

**What it tests**: Guest A has two e1000e devices; Guest B has one.
Maximum stress test of concurrent MSI table management and vCPU migration.

**Setup**: host_console + ssh_a + ssh_b. Set up SSH after host boot.

**Per-iteration sequence**:
1. Boot host. Verify host eth1, eth2, eth3 (udhcpc + ping).
2. Open ssh_a. Launch Guest A: `runkvm-n 0000:00:01.0 0000:00:02.0`.
3. Open ssh_b. Launch Guest B: `runkvm-n 0000:00:03.0`.
4. Guest A: `dmesg -n 1`.
   - eth0 (0000:00:01.0): static IP `192.168.0.15/24`, verify MSI-X
     (`grep -c PCI-MSIX-0000:00:00.0 /proc/interrupts`, expect >0).
     Ping not used — slirp net1 does not respond to ARP with two VFIO
     devices in the same guest.
   - eth1 (0000:00:02.0): static IP `192.168.1.15/24`, ping 192.168.1.2.
5. Guest B: `dmesg -n 1`.
   - eth0 (0000:00:03.0): static IP `192.168.2.15/24`, verify MSI-X
     (`grep -c PCI-MSIX-0000:00:00.0 /proc/interrupts`, expect >0).
     Attempt ping 192.168.2.2 as best-effort — slirp net3 is unreliable
     with multiple simultaneous VFIO guests; treat failure as informational,
     not a hard FAIL.
6. Concurrent pings:
   - Guest A: `ping -c 10 -I eth1 192.168.1.2` (must pass).
   - Guest B: `ping -c 10 -I eth0 192.168.2.2` (best-effort; slirp
     unreliable — log result but do not fail the iteration on loss).
   Wait for both to complete.
7. Shutdown Guest A and Guest B (`poweroff -f`). Unbind: `e1000e-unbind-all`.
8. Verify host eth1, eth2, eth3 reclaimed.

**Pass criteria**:
- All three host e1000e pings OK before and after
- Guest A eth0: MSI-X active (>0 vectors for PCI-MSIX-0000:00:00.0);
  ping not used (slirp unreliable)
- Guest A eth1: ping 192.168.1.2 OK
- Guest B eth0: MSI-X active (>0 vectors for PCI-MSIX-0000:00:00.0);
  ping best-effort (log result, not a hard failure)
- Guest A eth1 concurrent ping (10 packets): 0% loss
- Guest B concurrent ping: best-effort (log result)
- QEMU trace: 0 faults, MSI translations for all three BDFs,
  >=3 distinct GSCIDs in IOTINVAL.GVMA
- No cross-domain interference: distinct GSCID per domain in IOTINVAL.GVMA
  commands confirms Guest A's two devices and Guest B's device each have
  independent IOMMU domains with no cross-contamination

## Task

1. Ask the user which test cases to run (TC1–TC5, or all). Default: all.
   The test descriptions are fully embedded in this agent (see the Test
   Cases section). If the user provides a path to `irqbypass-tests.txt`
   (or any other test description file), read it — it may contain test
   cases not yet known to this agent, in which case use the file as the
   authoritative description and generate the implementation from it
   using the patterns in this agent and its parents.
2. Ask for the paths: `qemu_bin`, `kernel`, `rootfs`, `share_dir`.
   Verify `<share_dir>/bin/lkvm-static` exists. If missing, follow the
   procedure in `qemu-testing-kvm.md` ("Cross-Compiling kvmtool") to
   build it from a kvmtool git repository. Then verify or generate the
   shell scripts:
   - `<share_dir>/runqemu/Image` — if missing, copy the kernel:
     ```bash
     mkdir -p <share_dir>/runqemu && cp <kernel> <share_dir>/runqemu/Image
     ```
   - `<share_dir>/bin/runkvm`, `runkvm-n`, `e1000e-unbind`,
     `e1000e-unbind-all` — if missing, write them from the templates in
     the "Share Directory Scripts" section above and `chmod +x` each.
3. Check rootfs health: `e2fsck -f -y <rootfs>` (offer to skip if user
   confirms it was recently checked).
4. Verify SSH key for multi-session tests (TC3–TC5). Check that
   `~/.ssh/id_ed25519` and `~/.ssh/id_ed25519.pub` exist on the host
   machine. If missing, generate a key:
   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ''
   ```
   TC1 and TC2 use the console only and do not require SSH.
5. Write one Python test script per requested test case to the test
   directory (default: current working directory), using the patterns
   from this file and from `qemu-testing.md` / `qemu-testing-kvm.md`.
   Name the scripts `tc1_basic_boot.py` through `tc5_multi_device_multi_guest.py`.
6. Run each script in order. Report PASS/FAIL after each.
7. After all scripts complete, print a summary table:

```
TC1: PASS  (5/5 iterations)
TC2: PASS  (5/5 iterations)
TC3: PASS  (5/5 iterations)
TC4: PASS  (3/3 iterations)
TC5: PASS  (3/3 iterations)
Overall: PASS
```

## Output Per Test Case

```
=== TC<N>: <name> ===
Iteration 1: PASS / FAIL
  ...
Trace analysis: PASS / FAIL
  0 IOMMU faults
  <N> MSI translations for 0000:01.0
  ...
Overall: PASS / FAIL (<M>/<total> iterations passed)
Log: /tmp/tc<N>_results.log
```
