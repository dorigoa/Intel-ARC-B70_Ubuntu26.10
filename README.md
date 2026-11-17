# Making `xpu-smi` work with Intel Arc Pro B70 (Battlemage) on Ubuntu

Verified on **Ubuntu 26.10**, kernel `xe` driver, **2× Intel Arc Pro B70** (`Battlemage G31`),
`xpu-smi` 2.3.0.

---

## 1. The symptom

Graphics works perfectly (GUI, 165 Hz), the GPUs are listed by `lspci`, but `xpu-smi`
sees nothing:

```console
$ xpu-smi
No GPU devices found.

$ lspci | grep -i vga
2f:00.0 VGA compatible controller: Intel Corporation Battlemage G31 [Arc Pro B70]
33:00.0 VGA compatible controller: Intel Corporation Battlemage G31 [Arc Pro B70]
```

Or, a variant of the same problem — the devices appear, but with a wrong and alarming state:

```console
$ xpu-smi discovery
  Device ID     Device Information
  0             Device State: survivability mode
                Recovery Action: Firmware update and GPU reset
                PCI BDF Address: 0000:33:00.0
```

while the very same command under `sudo` reports `Device State: normal`.

**Both symptoms are permission problems, not driver or firmware problems.**
A GPU that is really in survivability mode cannot drive a display at all, so a working
desktop already rules that out.

---

## 2. Root cause

`xpu-smi` does not talk to the kernel driver directly. It goes through
**Level Zero Sysman**, which needs two privileges that an unprivileged user does not
have by default on Ubuntu:

| Privilege | Missing → symptom |
|---|---|
| Read/write on the DRM render nodes `/dev/dri/renderD*` (group `render`) | `No GPU devices found` |
| Access to the kernel performance counters (`kernel.perf_event_paranoid`) | Devices listed, but state misreported as `survivability mode` |

The second one is the non-obvious part: when the perf counters cannot be opened,
`xpu-smi` does not print an error — it falls back to reporting the device as being in
survivability mode. Running as root masks both issues, which is why `sudo xpu-smi` works
and makes the problem look like a bug in the tool.

Acronyms used below:
*DRM* = Direct Rendering Manager, the kernel graphics subsystem.
*BDF* = Bus:Device.Function, the PCI address of a card.
*DMC / GuC / HuC / GSC* = the GPU's display, scheduling, media and security firmware blobs.

---

## 3. Dependencies

### 3.1 Kernel driver

Battlemage (Arc B-series) requires the **`xe`** driver, not the legacy `i915`.
Ubuntu 25.10 and newer ship it enabled by default; nothing to install.

```bash
lsmod | grep -w xe
sudo dmesg | grep -iE 'xe .*firmware|GuC|HuC|DMC'
```

A healthy card loads all its firmware at boot:

```
xe 0000:2f:00.0: [drm] Finished loading DMC firmware i915/bmg_dmc.bin (v2.6)
xe 0000:2f:00.0: [drm] Tile0: GT0: Using GuC firmware from xe/bmg_guc_70.bin version 70.72.1
xe 0000:2f:00.0: [drm] Tile0: GT1: Using HuC firmware from xe/bmg_huc.bin version 8.2.10
```

### 3.2 Compute runtime and Level Zero

```bash
sudo apt install -y \
    intel-opencl-icd \
    libze-intel-gpu1 \
    libze1 \
    clinfo
```

| Package | Role |
|---|---|
| `intel-opencl-icd` | Intel compute runtime for OpenCL |
| `libze-intel-gpu1` | Level Zero backend for Intel GPUs (what `xpu-smi` and SYCL actually use) |
| `libze1` | oneAPI Level Zero loader |
| `clinfo` | Optional, handy for a second opinion on GPU enumeration |

On Ubuntu 26.10 the archive versions are recent enough (`26.27.x` compute runtime,
Level Zero `1.32.x`). On older releases, or if you need newer, use Intel's own
repository — see <https://dgpu-docs.intel.com/driver/installation.html>.
Do not mix the archive and Intel's repository without pinning; the `*-legacy*`
variants (`intel-opencl-icd-legacy`, `libze-intel-gpu-legacy1-1`) are for pre-Arc
hardware and are harmless to leave installed.

Check what you have:

```bash
dpkg -l | grep -E 'level-zero|libze|intel-opencl|intel-compute|xpu-smi'
```

### 3.3 `xpu-smi` itself

`xpu-smi` is **not** in the Ubuntu archive. Download the `.deb` packages from the
official release page:

**<https://github.com/intel/xpumanager/releases>**

Two packages are needed, and their versions must match:

* `libxpum2_<version>_amd64.deb` — the shared library
* `xpu-smi_<version>_amd64.deb` — the CLI

Each release publishes one build per supported Ubuntu version, with the suffix in the
filename (`~24.04`, `~26.04`, …). Pick the one closest to your release — on Ubuntu 26.10
the `~26.04` build is the right choice; the `~24.04` build also works if its dependencies
resolve. Checksums are published as `*.sha256` next to each asset.

```bash
# adjust version/suffix to the files you actually downloaded
VER=2.3.0-1~26.04
sha256sum -c libxpum2_${VER}_amd64.deb.sha256
sha256sum -c xpu-smi_${VER}_amd64.deb.sha256
sudo apt install ./libxpum2_${VER}_amd64.deb ./xpu-smi_${VER}_amd64.deb
```

Using `apt install ./file.deb` rather than `dpkg -i` lets apt pull in the dependencies.

---

## 4. The fix

### 4.1 Grant access to the render nodes

```bash
sudo usermod -aG render,video "$USER"
```

Then **log out and log back in** (a reboot also works). Group membership is attached to
the login session, so it does not apply to shells that are already open. To test
immediately in the current shell without logging out:

```bash
newgrp render
```

Verify:

```bash
id -nG | tr ' ' '\n' | grep -E 'render|video'
ls -l /dev/dri/
```

Expected — the render nodes belong to group `render` with `rw` for that group:

```
crw-rw----+ 1 root video  226,   0 card0
crw-rw----+ 1 root video  226,   1 card1
crw-rw----+ 1 root render 226, 128 renderD128
crw-rw----+ 1 root render 226, 129 renderD129
```

The trailing `+` means an ACL is present (`logind` grants the active seat user access to
`card*`). It does not interfere. If you want to be sure the effective access is there:

```bash
for d in /dev/dri/renderD*; do
    printf '%s: ' "$d"
    { [ -r "$d" ] && [ -w "$d" ] && echo "rw ok"; } || echo "NO ACCESS"
done
```

### 4.2 Allow the perf counters

Ubuntu ships `kernel.perf_event_paranoid=4`, which blocks unprivileged access.
`xpu-smi` needs `0` or lower.

Temporary, to confirm the diagnosis:

```bash
sudo sysctl -w kernel.perf_event_paranoid=0
xpu-smi discovery
```

Permanent:

```bash
sudo tee /etc/sysctl.d/99-intel-gpu.conf <<'EOF'
# xpu-smi / Level Zero Sysman need unprivileged access to perf counters
kernel.perf_event_paranoid = 0
# GPU hardware counters with the xe driver (i915 equivalent: dev.i915.perf_stream_paranoid)
dev.xe.observation_paranoid = 0
EOF

sudo sysctl --system
```

`kernel.perf_event_paranoid` is the one that fixes device discovery and state reporting.
`dev.xe.observation_paranoid` governs access to the *GPU's own* hardware counters and is
what you need for utilization/telemetry (`xpu-smi stats`, `xpu-smi dump`, VTune GPU
analysis). Set both unless you have a reason not to. If the second key does not exist,
`sysctl --system` will warn and continue — it simply means your kernel exposes it under a
different name or the `xe` module was not loaded when sysctl ran.

---

## 5. Verification

```bash
xpu-smi discovery          # no sudo
```

```
  Device ID     Device Information
  0             Device Name: Intel(R) Arc(TM) Pro B70 Graphics
                Device State: normal
                Vendor Name: Intel(R) Corporation
                PCI BDF Address: 0000:2f:00.0
                DRM Device: /dev/dri/card0
                Function Type: physical

  1             Device Name: Intel(R) Arc(TM) Pro B70 Graphics
                Device State: normal
                Vendor Name: Intel(R) Corporation
                PCI BDF Address: 0000:33:00.0
                DRM Device: /dev/dri/card1
                Function Type: physical
```

Then the telemetry path:

```bash
xpu-smi stats -d 0
xpu-smi dump -d 0 -m 0,5,22      # GPU utilization, power, memory used
```

And, independently of `xpu-smi`, that compute really works for your user:

```bash
clinfo -l
```

---
## 5a. Enable LevelZero instead of OpenCL
```bash
echo /opt/intel/oneapi/umf/latest/lib | sudo tee /etc/ld.so.conf.d/intel-oneapi-umf.conf
sudo ldconfig
ldconfig -p | grep libumf
```
In case of llama-server usage, this should the check result on devices:
```bash
(base) alvise@ubuntu:/opt/intel/oneapi/redist/lib$ ONEAPI_DEVICE_SELECTOR='level_zero:*' /opt/llama.cpp/llama-server --list-devices
Available devices:
  SYCL0: Intel(R) Arc(TM) Pro B70 Graphics (32656 MiB, 31639 MiB free)
  SYCL1: Intel(R) Arc(TM) Pro B70 Graphics (32656 MiB, 32599 MiB free)
```
If LevelZero is not working:
```bash
(base) alvise@ubuntu:~/Strata$ /opt/llama.cpp/llama-server --list-devices
0.00.194.899 W SYCL GPU device 0 does not use Level Zero backend, disabling Level Zero memory API
0.00.199.118 W SYCL GPU device 1 does not use Level Zero backend, disabling Level Zero memory API
Available devices:
  SYCL0: Intel(R) Arc(TM) Pro B70 Graphics (31023 MiB, 31023 MiB free)
  SYCL1: Intel(R) Arc(TM) Pro B70 Graphics (31023 MiB, 31023 MiB free)


(base) alvise@ubuntu:~/Strata$ cd /opt/intel/oneapi/redist/lib && \
ldd libur_adapter_level_zero.so.0 libur_adapter_level_zero_v2.so.0 | grep -E ':$|not found'
libur_adapter_level_zero.so.0:
        libumf.so.1 => not found
libur_adapter_level_zero_v2.so.0:
        libumf.so.1 => not found
```
---

## 6. Security trade-off and the narrower alternative

`kernel.perf_event_paranoid = 0` lets **any local user** read CPU performance counters.
On a single-user workstation this is irrelevant; on a shared or multi-tenant machine it is
a real side-channel exposure (see <https://docs.kernel.org/admin-guide/perf-security.html>).

The surgical alternative is to leave the sysctl at its default and grant the capability to
the binary only:

```bash
sudo setcap cap_perfmon+ep /usr/bin/xpu-smi
getcap /usr/bin/xpu-smi
```

Caveat: file capabilities live on the inode, so they are **lost on every package upgrade**
and must be reapplied. If you go this route, a trivial `dpkg` trigger or a post-upgrade
hook is worth adding.

---

## 7. Troubleshooting matrix

| Output | Cause | Fix |
|---|---|---|
| `No GPU devices found` as user, works with `sudo` | not in `render` group, or group not yet active in this session | §4.1 + log out/in |
| `No GPU devices found` with `sudo` too | Level Zero backend missing | install `libze-intel-gpu1`, `libze1` (§3.2) |
| `Device State: survivability mode`, `normal` under `sudo` | `kernel.perf_event_paranoid` too high | §4.2 |
| `Device State: survivability mode` under `sudo` too, **and** `dmesg` shows survivability / firmware load failure, **and** `/sys/bus/pci/devices/<BDF>/survivability_mode` exists | genuine firmware problem | `xpu-smi updatefw` (requires root) |
| `xpu-smi stats` empty or zeros, discovery fine | GPU counters blocked | `dev.xe.observation_paranoid = 0` (§4.2) |
| `UNSUPPORTED_FEATURE` on health/config commands | Level Zero too old for that feature | update the compute runtime (§3.2) |

Quick way to tell a *real* survivability condition from a misreport — the kernel is the
authoritative source, `xpu-smi` is not:

```bash
sudo dmesg | grep -i survivab
ls /sys/bus/pci/devices/0000:2f:00.0/ | grep -i surviv
```

Both empty ⇒ the GPU is fine and the `xpu-smi` report is a privilege artifact.

---

## 8. Summary — the whole fix

```bash
# 1. compute stack
sudo apt install -y intel-opencl-icd libze-intel-gpu1 libze1 clinfo

# 2. xpu-smi from https://github.com/intel/xpumanager/releases
sudo apt install ./libxpum2_*_amd64.deb ./xpu-smi_*_amd64.deb

# 3. device access
sudo usermod -aG render,video "$USER"

# 4. perf counters
sudo tee /etc/sysctl.d/99-intel-gpu.conf <<'EOF'
kernel.perf_event_paranoid = 0
dev.xe.observation_paranoid = 0
EOF
sudo sysctl --system

# 5. log out and back in, then:
xpu-smi discovery
```

---

## References

* Intel XPU Manager releases — <https://github.com/intel/xpumanager/releases>
* Intel discrete GPU driver installation — <https://dgpu-docs.intel.com/driver/installation.html>
* Perf events security model — <https://docs.kernel.org/admin-guide/perf-security.html>
* Intel VTune, GPU analysis setup (source for `dev.xe.observation_paranoid` on the `xe` driver) —
  <https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/current/set-up-system-gpu-analysis.html>
