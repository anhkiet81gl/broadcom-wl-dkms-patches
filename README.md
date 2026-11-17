# broadcom-wl failed-roam fix

A patch for the Broadcom `wl` Wi-Fi driver (broadcom-wl 6.30.223.271) on Linux 6.0 and later. Without it, a failed roam to another access point leaves the card unable to connect again until you reboot.

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

## Status

Tested on Arch Linux with `broadcom-wl-dkms` 6.30.223.271-50, `linux` 7.2.15 and a BCM4360 card (14e4:43a0). The patched module has run on one machine since 2026-11-17 and connects normally. No roam has failed on that machine since then, so the new code path has not fired in the field yet.

## Install on Arch Linux

Before you start:

- `broadcom-wl-dkms` and the headers for your running kernel (`linux-headers`) are installed.
- Run the steps at the machine, not over SSH on the same Wi-Fi: step 3 reloads the driver.
- Run all steps in one shell, from this repository's directory. Later steps reuse the variables set in step 1.

1. Back up the current driver and source, so you can roll back later:

   ```bash
   src_dir=/usr/src/broadcom-wl-6.30.223.271
   kernel=$(uname -r)
   backup_dir=/var/lib/wl-patch
   sudo install -d -m 700 $backup_dir
   sudo install -m 600 /lib/modules/$kernel/updates/dkms/wl.ko.zst $backup_dir/wl-$kernel-original.ko.zst
   sudo install -m 600 $src_dir/src/wl/sys/wl_cfg80211_hybrid.c $backup_dir/wl_cfg80211_hybrid.c.original
   sudo sh -c "cd $backup_dir && sha256sum wl-$kernel-original.ko.zst wl_cfg80211_hybrid.c.original > SHA256SUMS"
   ```

2. Apply the patch and rebuild the module for the running kernel:

   ```bash
   sudo patch -p1 -d $src_dir < wl-failed-roam-disconnect.patch
   sudo dkms build broadcom-wl/6.30.223.271 -k $kernel --force
   sudo dkms install broadcom-wl/6.30.223.271 -k $kernel --force
   ```

   `patch` prints `patching file src/wl/sys/wl_cfg80211_hybrid.c`.

3. Load the patched module. Wi-Fi drops for a few seconds:

   ```bash
   sudo modprobe -r wl && sudo modprobe wl
   sudo systemctl restart wpa_supplicant.service NetworkManager.service
   ```

You are done when Wi-Fi reconnects and both checks pass:

```bash
zstd -dqc /lib/modules/$(uname -r)/updates/dkms/wl.ko.zst | grep -ac "roam failed"
```

prints `1`, and the loaded module matches the installed one:

```bash
cat /sys/module/wl/srcversion
modinfo -F srcversion /lib/modules/$(uname -r)/updates/dkms/wl.ko.zst
```

## Other distributions

The patch changes one function in `src/wl/sys/wl_cfg80211_hybrid.c`. Apply it with `patch -p1` from the directory that contains `src/`, then rebuild the module the way your distribution builds it. It has only been tested on Arch.

## After updates

- **Kernel update:** nothing to do. DKMS rebuilds the module for the new kernel from the patched source in `/usr/src`.
- **`broadcom-wl-dkms` update:** the package overwrites the source and rebuilds the module without the patch. Repeat the install steps.

## When the fix fires

When a roam fails, the driver logs `roam failed, reporting the old link as disconnected` and Wi-Fi reconnects on its own within seconds instead of staying down. To find it:

```bash
journalctl -k | grep "roam failed"
```

## Roll back

`wl-patch-rollback` restores the backups from step 1 and reloads Wi-Fi. It refuses to run if a backup is a symlink, is not owned by root, is writable by other users, or fails its SHA-256 check.

```bash
sudo install -m 755 wl-patch-rollback /usr/local/sbin/
sudo wl-patch-rollback
```

## Files

- `wl-failed-roam-disconnect.patch`: the fix.
- `wl-patch-rollback`: restores the original driver and source.
- `ANALYSIS.md`: log evidence, and the kernel change that exposed the bug.
