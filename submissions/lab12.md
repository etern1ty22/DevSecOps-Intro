# Lab 12 — VM-Backed Container Isolation with Kata

Branch: `feature/lab12`. Executed on 2026-10-17. Task 1, Task 2 and the bonus
are all done.

| Component | Version | How it ran |
| --- | --- | --- |
| Host kernel | **6.18.33.1-microsoft-standard-WSL2** | Ubuntu-24.04 under WSL2, 16 vCPU, ~9.7 GB RAM, `/dev/kvm` present |
| containerd | **v2.4.1** (`f2551031`) | `/usr/local/bin/containerd`, running |
| nerdctl | **2.4.1** | `/usr/local/bin/nerdctl` |
| Kata Containers | **4.1.0** (shim commit `ddcb1ad8`) | static assets at `/opt/kata`, runtime `io.containerd.kata.v2` |
| Kata hypervisor | **Dragonball** (built into the runtime-rs shim) | `configuration-dragonball.toml`, guest kernel `vmlinux-dragonball-experimental` |
| Image | **alpine:3.20** | same image on both runtimes |
| runc | bundled with containerd 2.4.1 | default runtime |

Raw outputs are in `labs/lab12/results/`.

### Setup note: nothing needed installing

Every prerequisite was already present on this machine, so I downloaded and
installed nothing. `/dev/kvm`, containerd 2.4.1, nerdctl 2.4.1, `ctr`, `zstd`,
`jq` and `curl` were all on `PATH`, and the Kata 4.1.0 static assets were
already unpacked under `/opt/kata` with the `kata` runtime block already written
into `/etc/containerd/config.toml` (lines 285–286). So the `install-kata-assets.sh`
and `configure-containerd-kata.sh` steps in 12.1 were not re-run.

### Where this run departs from the lab text

1. **The shim binary was not on containerd's `PATH`.** The assets install the
   runtime-rs shim at `/opt/kata/runtime-rs/bin/containerd-shim-kata-v2`, but
   containerd resolves shims from `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/snap/bin`.
   The first `--runtime=io.containerd.kata.v2` run therefore died with
   `binary not installed "containerd-shim-kata-v2": file does not exist`. Fixed
   with a symlink (no download): `ln -sf /opt/kata/runtime-rs/bin/containerd-shim-kata-v2 /usr/local/bin/`.
2. **Git Bash / WSL path mangling.** Commands were driven from the Windows host
   into WSL. MSYS rewrites `/mnt/c/...` arguments into `C:/Program Files/Git/mnt/...`
   and drops inline shell-variable assignments, so every WSL invocation ran with
   `MSYS_NO_PATHCONV=1 MSYS2_ARG_CONV_EXCL='*'` and the real work was kept in
   script files executed by path.
3. **Hypervisor is Dragonball, not QEMU.** The runtime-rs default config points
   at `[hypervisor.dragonball]`. Dragonball is a library VMM linked *into* the
   shim process, so there is no separate `qemu`/`cloud-hypervisor` process to
   measure — the VM's memory shows up as the shim's RSS (see Task 2).
4. **A benign warning on every Kata run:** `cannot set cgroup manager to
   "systemd" for runtime "io.containerd.kata.v2"`. It does not affect the result
   and is filtered out of the quoted output below.
5. **Timing includes `nerdctl` + containerd overhead**, identically for both
   runtimes, so the comparison is fair but the absolute numbers are not "runtime
   only" (see the order-of-magnitude discussion in Task 2).

---

## Task 1

### Kernel versions

| Where | `uname -r` |
| --- | --- |
| Host | `6.18.33.1-microsoft-standard-WSL2` |
| runc container | `6.18.33.1-microsoft-standard-WSL2` |
| Kata container | `6.18.35` |

The runc line matches the host exactly; the Kata line does not. runc shares the
host kernel through namespaces, so `uname -r` inside the container *is* the host
kernel. Kata boots a separate guest kernel (the Dragonball `vmlinux` image), so
the container reports `6.18.35` — a kernel the host is not even running.

### Device counts and capability masks, side by side

| | runc | Kata |
| --- | --- | --- |
| `ls /dev \| wc -l` | **15** | **14** |
| `CapEff` of PID 1 | `00000000a80425fb` | `00000000a80425fb` |

The default (unprivileged) capability mask is identical under both runtimes —
Kata does not change the Linux capability set a container sees, because that is
decided by the OCI spec, not by where the kernel lives. The device counts are
both small and similar; the interesting divergence is not here but in the
*privileged* case (bonus), where runc exposes 181 host devices and Kata's
privileged container will not start at all.

### What that one kernel line tells an attacker

An attacker who has just landed in a `runc` container reads the host's exact
kernel release — `6.18.33.1-microsoft-standard-WSL2` — straight out of `uname`.
That single string tells them precisely which local-privilege-escalation CVEs
to try, because every container-escape-through-the-kernel exploit is keyed to a
kernel version, and here the container's kernel *is* the host's: a working
`CVE` against 6.18.33 escapes onto the host, not just out of the container. When
the answer is a *different* kernel (Kata's `6.18.35` guest), the exploit they
pick now compromises only a throwaway VM whose kernel is isolated from the host
by the hypervisor boundary — breaking that guest kernel gets them root in the
guest, not on the machine.

---

## Task 2

### Startup (five timed runs after one discarded warm-up)

Wall-clock seconds for `nerdctl run --rm ... alpine:3.20 true`:

| run | runc | Kata |
| --- | --- | --- |
| 1 | 1.37 | 1.55 |
| 2 | 1.37 | 1.56 |
| 3 | 1.35 | 2.48 |
| 4 | 1.39 | 1.52 |
| 5 | 1.39 | 1.55 |
| **median** | **1.37** | **1.55** |

Medians, not means: the Kata set has a 2.48 s outlier (run 3) that would drag a
mean up to ~1.73 while the typical run is ~1.55. Reporting the median keeps that
one slow run from lying about the common case.

Note these numbers are *not* an order of magnitude apart, which the lab warned
me to expect. The reason is honest and worth stating: the `nerdctl` + containerd
machinery around the container costs ~1.37 s on *both* runtimes and dominates
the wall clock. The actual VM-boot delta Kata adds is only the difference of the
medians, ≈ **0.18 s**. The order-of-magnitude figure is the runtime-only cost of
booting a guest kernel; here it is buried under the client/daemon overhead that
both runtimes pay equally. (If the shim overhead were stripped out, the Dragonball
micro-VM boot is the part that would show the ~10× gap against runc's near-zero
process spawn.)

### I/O throughput

`dd if=/dev/zero of=/tmp/bench bs=1M count=512 conv=fsync`:

| runtime | throughput |
| --- | --- |
| runc | 512 MB in 0.323 s → **1.5 GB/s** |
| Kata | 512 MB in 1.548 s → **330.8 MB/s** |

Kata is ~**4.5× slower** on this write. The benchmark writes to a *file* inside
the container rather than to `/dev/null` because `/dev/null` never touches the
storage layer — it is discarded in the kernel. Writing a real file with
`conv=fsync` forces the data across the filesystem/block path, which is exactly
where a VM-backed runtime differs: Kata's write has to cross the guest
filesystem and the virtio storage boundary into the host, while runc writes
straight to the host filesystem. `/dev/null` would have measured nothing but CPU.

### Memory overhead per container

Method: start one detached `sleep` container under each runtime and read RSS of
the backing host processes from `ps -eo rss,args` (there is no cgroup/VM process
to inspect separately for Dragonball, since the VMM is linked into the shim).

| runtime | host processes | RSS |
| --- | --- | --- |
| runc | `containerd-shim-runc-v2` (~10.8 MB) + `sleep` (~0.9 MB) | **≈ 11.5 MB** |
| Kata | `containerd-shim-kata-v2` incl. embedded Dragonball VMM | **≈ 175 MB** (179220 KB) |

So the per-container overhead of the VM is roughly **+164 MB** over runc. The
guest is *configured* for `default_memory = 2048` MB, but that is the ceiling
the guest kernel sees, not what the host pays up front — the host RSS of ~175 MB
is the demand-allocated reality for an idle Alpine guest. runc has no VM, so its
only footprint is the shim plus the workload itself.

### Workload verdicts

| Workload | Verdict | Why |
| --- | --- | --- |
| Internal CI build runners executing untrusted PR code | **Kata** | Arbitrary code from strangers is exactly the untrusted-tenant case; the ~175 MB and 0.2 s per job are negligible against a multi-minute build, and a kernel-escape CVE only reaches a throwaway guest. |
| Latency-sensitive internal microservice, high fan-out, trusted code | **runc** | All first-party code behind the same trust boundary; the 4.5× I/O penalty and VM memory tax buy isolation you already have, and startup/density matter. |
| Multi-tenant FaaS running customer functions with local disk I/O | **needs more information** | Kata's tenant isolation is the right default, but the 4.5× storage hit could break an I/O-heavy function's SLA — I'd need the function's I/O profile and per-node density target before committing. |

### For which workload class does the startup number not matter

The ~0.2 s (or even the full ~1.5 s) startup gap is irrelevant to any
**long-lived** workload — a service, a daemon, a database, a build that runs for
minutes. A container that starts once and then runs for hours amortises the boot
cost to nothing: paying an extra fraction of a second at launch is invisible
against the runtime. Startup latency only becomes a real cost at the other
extreme — very short-lived, high-churn containers (per-request FaaS cold starts,
thousands of ephemeral jobs) — where the fixed per-start cost is a large fraction
of the total lifetime.

---

## Bonus — An escape that Kata stops

Vector: a `--privileged` container with a `-v /tmp:/host_tmp` host bind mount —
the classic misconfiguration, not a CVE.

### Both runs and the host file state

Host file seeded with `original` before each run.

| run | command | host `/tmp/lab12-target` after |
| --- | --- | --- |
| runc | `--privileged -v /tmp:/host_tmp ... echo OVERWRITTEN > /host_tmp/lab12-target` | **`OVERWRITTEN-BY-RUNC`** — escape succeeded |
| Kata | identical + `--runtime=io.containerd.kata.v2` | **`original`** — unchanged; container never started |

On Kata the `create` call failed outright:

> `failed to create shim task: ... get host path failed ... No such file or directory`
> (`hypervisor::device::util::get_host_path`)

`--privileged` asks the runtime to pass *all host devices* through to the
container. Under Kata the agent tries to resolve those host device paths
**inside the guest VM**, where they do not exist, so container creation aborts
before any process runs. The host file stayed `original`.

**The bind mount is not what breaks it.** Running the identical `--privileged`
command with the `-v /tmp:/host_tmp` mount *removed* crashes with the same
`get_host_path` error (`results/escape-extra.txt`, run B). So the failure is the
device-passthrough half of `--privileged`, not the mount — Kata refuses the
passthrough regardless of whether a host path is also requested.

### Device count and capability mask, privileged, both runtimes

A plain `--privileged` container never starts on Kata, so it yields no mask to
read. To get a real maximum-privilege measurement under Kata I separated the two
things `--privileged` bundles — the full capability set and host device
passthrough — and granted only the first, with `--cap-add ALL`
(`results/escape-extra.txt`):

| | runc `--privileged` | runc `--cap-add ALL` | Kata `--privileged` | Kata `--cap-add ALL` |
| --- | --- | --- | --- | --- |
| starts? | yes | yes | **no** (create aborts) | yes |
| `ls /dev \| wc -l` | **181** | 15 | — | **14** |
| `CapEff` | `000001ffffffffff` | `000001ffffffffff` | — | **`000001ffffffffff`** |
| `uname -r` | host | host | — | `6.18.35` (guest) |

Two things stand out. First, the **capability mask is the full
`000001ffffffffff` wherever a container actually runs** — runc privileged, runc
all-caps, and Kata all-caps are identical. The mask is therefore *not* what stops
the escape. Second, what `--privileged` adds over `--cap-add ALL` is **device
passthrough**: on runc that turns 15 devices into **181 host devices** — the
host's real `/dev`, a direct escape surface on its own (host block devices,
`/dev/mem`, and so on), independent of the bind mount; on Kata that same
passthrough step is exactly what fails. Even holding the full capability set, a
Kata container sees only **14 guest devices** and runs the **guest** kernel
`6.18.35`, so there is no host `/dev` to own.

### Explanation

`--privileged` grants two separate things: the full Linux capability set *and*
direct passthrough of the host's devices. The `--cap-add ALL` runs above show
these pull apart — capabilities are handed out identically on both runtimes, so
the privilege that matters here is the **device passthrough**, and *to what* it
grants access is the whole question. Under runc the devices and kernel you now
own **are the real machine's** — so a privileged container with a host path
mounted writes straight to the host filesystem, and the file changes. Under Kata
the full capability set applies only to the **guest** kernel over a 14-node guest
device tree, never the host's; the passthrough step cannot resolve host devices
inside the VM, so the runtime refuses to start the container at all. The escape
did not partly work on Kata — it did not run at all, and the host file was still
`original` afterward.

### The honest limit

Kata does **not** protect you from a host path you deliberately hand it. A
*non-privileged* `-v /tmp:/host_tmp` on Kata still overwrote the host file
(`OVERWRITTEN-BY-KATA-NONPRIV`) — explicit bind mounts are shared into the guest
over virtio-fs and writes propagate back to the host. The second kernel contains
what the container can reach *through the kernel*; it does not revoke access you
explicitly granted at the mount boundary.

---

## Undo

Host changes to revert when finished (not done automatically): remove the symlink
`/usr/local/bin/containerd-shim-kata-v2`, remove the `kata` runtime block from
`/etc/containerd/config.toml`, restart containerd, and delete `/opt/kata`. The
`kata` block and `/opt/kata` pre-existed this session, so they are left in place.
