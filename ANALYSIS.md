# Why the card never reconnects after a failed roam

The `wl` driver reports a failed roam as a failed connection but never reports that the old link is gone. Up to Linux 5.19 the kernel cleaned up anyway. Since Linux 6.0 it does not, so it believes the card is still connected and refuses every new connection.

## What happens

1. The card is connected to access point A. `wpa_supplicant` roams to access point B with the same SSID.
2. The join fails. `wl_bss_connect_done()` in `src/wl/sys/wl_cfg80211_hybrid.c` calls `cfg80211_connect_result(..., WLAN_STATUS_AUTH_TIMEOUT)` with the BSSID of A, and never calls `cfg80211_disconnected()`.
3. The kernel handles the failed result in `__cfg80211_connect_result()` (`net/wireless/sme.c`). Since 6.0 it clears `ssid_len` but leaves `wdev->connected` set. Only `__cfg80211_disconnected()` clears that flag.
4. Every later connect hits `cfg80211_connect()`, which returns `-EALREADY` because `wdev->connected` is set and no `prev_bssid` is given.
5. On `-EALREADY`, `wpa_supplicant` disconnects and retries (`wpa_driver_nl80211_connect()`). But `cfg80211_disconnect()` only calls into the driver while `ssid_len` is non-zero. It is already 0, so the driver never hears about it and the flag stays set.

The card stays in this state until the interface is torn down. On the affected machine only a reboot recovered it.

## Why it started with Linux 6.0

| Kernel | Connected state | Cleared by a failed connect result |
|---|---|---|
| 5.15, 5.19 | `wdev->current_bss` | yes, set to `NULL` before the status check |
| 6.0, 7.2.15 | `wdev->connected` | no, only by `__cfg80211_disconnected()` |

Linux 6.0 moved station state to `wdev->connected` while reworking cfg80211 for multi-link operation. The missing `cfg80211_disconnected()` call in `wl` was harmless before that change.

## Evidence

The log from one incident, with SSID and BSSIDs replaced:

```
wpa_supplicant: wlp1s0: Trying to associate with <AP-B> (SSID='<ssid>' freq=2462 MHz)
wpa_supplicant: wlp1s0: CTRL-EVENT-ASSOC-REJECT bssid=<AP-A> status_code=16
wpa_supplicant: wlp1s0: Trying to associate with <AP-A> (SSID='<ssid>' freq=2412 MHz)
wpa_supplicant: wlp1s0: Association request to the driver failed
```

- The reject names A, the access point the card was leaving, with status 16. That matches what `wl_bss_connect_done()` sends on a failed join.
- "Association request to the driver failed" repeated 130 times over 23 minutes.
- During those failures the driver logged no `ERROR @wl_cfg80211_connect` line. Every error path in `wl_cfg80211_connect()` logs one, so the requests were rejected by cfg80211 before they reached the driver.
- Restarting NetworkManager three times did not recover the card. A reboot did.

Versions on the affected machine: `broadcom-wl-dkms` 6.30.223.271-50, `linux` 7.2.15, `wpa_supplicant` 2.12, `networkmanager` 1.58.1, card BCM4360 (14e4:43a0).

## The fix

[wl-failed-roam-disconnect.patch](wl-failed-roam-disconnect.patch) adds one branch to `wl_bss_connect_done()`. When a join fails while the driver still marks the old link as connected (`WL_STATUS_CONNECTED`), it reports that link with `cfg80211_disconnected()` and resets the driver's link state. That is the same thing the driver already does when the firmware reports a link-down event. The kernel then clears `wdev->connected`, and `wpa_supplicant` reconnects normally.
