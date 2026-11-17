# broadcom-wl failed-roam fix

A patch for the Broadcom `wl` Wi-Fi driver (broadcom-wl / broadcom-sta 6.30.223.271) on Linux 6.0 and later. Without it, a failed roam to another access point leaves the card unable to connect again until you reboot.

## Do you have this bug?

You are affected if all of these hold:

- Your card uses the `wl` driver: `lspci -k` shows `Kernel driver in use: wl`.
- Your kernel is 6.0 or later: check with `uname -r`.
- Wi-Fi went down after a roam and never came back, and `wpa_supplicant` logged an `ASSOC-REJECT` with `status_code=16` followed by repeated association failures:

  ```bash
  journalctl -t wpa_supplicant | grep -E "ASSOC-REJECT|Association request to the driver failed"
  ```

  ```
  wlp1s0: CTRL-EVENT-ASSOC-REJECT bssid=<old AP> status_code=16
  wlp1s0: Association request to the driver failed
  wlp1s0: Association request to the driver failed
  ```

- Restarting NetworkManager does not help, but a reboot does.

For the cause, see [why the card never reconnects](ANALYSIS.md).

## Tested on

| Distribution | Driver package | DKMS entry | Kernel | What was tested |
|---|---|---|---|---|
| Arch Linux | `broadcom-wl-dkms` 6.30.223.271-50 | `broadcom-wl/6.30.223.271` | 7.2.15 | Installed and running on a BCM4360 card since 2026-11-17 |
| Arch Linux | `broadcom-wl-dkms` 6.30.223.271-50 | `broadcom-wl/6.30.223.271` | 7.3.4 | Patch, build, roll back, in a container |
| Debian 12 | `broadcom-sta-dkms` 6.30.223.271-23 | `broadcom-sta/6.30.223.271` | 6.1.0-56 | Patch, build, roll back, in a container |
| Ubuntu 24.04 | `bcmwl-kernel-source`, which installs `broadcom-sta-dkms` 6.30.223.271-23ubuntu1.3 | `broadcom-sta/6.30.223.271` | 6.8.0-149 | Patch, build, roll back, in a container |
| Ubuntu 22.04 | `bcmwl-kernel-source` 6.30.223.271+bdcom-0ubuntu10~22.04.1 | `bcmwl/6.30.223.271+bdcom` | 6.8.0-149 (HWE) | Patch, build, roll back, in a container |

Ubuntu 22.04 on its original 5.15 kernel is not affected, because the bug needs Linux 6.0 or later.

On the Arch machine the card connects normally with the patched driver. No roam has failed there since it was installed, so the new code path has not fired in the field yet.

## Install

Before you start:

- The driver is installed through DKMS, and the headers for your running kernel are installed.
- Run the steps at the machine, not over SSH on the same Wi-Fi: step 4 reloads the driver.
- Run all steps in one shell, from this repository's directory. Later steps reuse the variables set in step 2.

1. Find the driver's DKMS entry:

   ```bash
   dkms status | grep -E "broadcom|bcmwl"
   ```

   The entry is the text before the first comma, for example `broadcom-sta/6.30.223.271`.

2. Set the entry, its source directory, and the running kernel. Replace the entry with yours:

   ```bash
   dkms_entry=broadcom-sta/6.30.223.271
   src_dir=/usr/src/${dkms_entry%%/*}-${dkms_entry#*/}
   kernel=$(uname -r)
   ```

3. Apply the patch and rebuild the module:

   ```bash
   sudo patch -p1 -d "$src_dir" < wl-failed-roam-disconnect.patch
   sudo dkms remove "$dkms_entry" -k "$kernel"
   sudo dkms install "$dkms_entry" -k "$kernel"
   ```

   `patch` prints `patching file src/wl/sys/wl_cfg80211_hybrid.c`, sometimes followed by `Hunk #1 succeeded at ... (offset ... lines)`. Both are fine. Use `remove` then `install`, not `dkms build --force`: DKMS 2.x skips a module it has already built, even with `--force`.

4. Load the patched module. Wi-Fi drops for a few seconds:

   ```bash
   sudo modprobe -r wl && sudo modprobe wl
   sudo systemctl restart wpa_supplicant.service NetworkManager.service
   ```

You are done when Wi-Fi reconnects and this prints `1`:

```bash
module_file=$(modinfo -n wl)
case "$module_file" in *.zst) zstd -dqc "$module_file" ;; *) cat "$module_file" ;; esac | grep -ac "roam failed"
```

## After updates

- **Kernel update:** nothing to do. DKMS builds the module for the new kernel from the patched source in `/usr/src`.
- **Driver package update:** the package overwrites the source in `/usr/src` and the module is rebuilt without the patch. Repeat the install steps.

## When the fix fires

When a roam fails, the driver logs `roam failed, reporting the old link as disconnected` and Wi-Fi reconnects on its own within seconds instead of staying down. To find it:

```bash
journalctl -k | grep "roam failed"
```

## Roll back

Reverse the patch and rebuild, using the variables from install step 2:

```bash
sudo patch -R -p1 -d "$src_dir" < wl-failed-roam-disconnect.patch
sudo dkms remove "$dkms_entry" -k "$kernel"
sudo dkms install "$dkms_entry" -k "$kernel"
sudo modprobe -r wl && sudo modprobe wl
sudo systemctl restart wpa_supplicant.service NetworkManager.service
```

You are done when the check from the install section prints `0`.

## Other distributions

Distributions that build `wl` without DKMS, such as Fedora with RPM Fusion's `akmod-wl`, are not covered here. The patch changes one function in `src/wl/sys/wl_cfg80211_hybrid.c`; apply it with `patch -p1` from the directory that contains `src/`, then rebuild the module the way your distribution does.

## Files

- `wl-failed-roam-disconnect.patch`: the fix.
- `ANALYSIS.md`: log evidence, and the kernel change that exposed the bug.
