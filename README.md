# fingerprint-validity-vfs0090-setup
Working Validity VFS0090 (138a:0090) fingerprint reader setup for Arch-based Linux (CachyOS/KDE Plasma) — fprintd + PAM
# Setting up the Validity VFS0090 fingerprint reader on Arch-based Linux (CachyOS/KDE Plasma)

A working, reproducible setup for fingerprint authentication on Arch-based distros
(CachyOS, Manjaro, Arch Linux) with KDE Plasma. Tested on a laptop with a
**Validity Sensors sensor (USB ID 138a:0090 / VFS7500)**, exposed by libfprint as
**Validity VFS0090 (press)**.

## 1. Install the packages

```bash
sudo pacman -S fprintd libfprint
```

Tested versions: `fprintd 1.94.5`, `libfprint 1.94.100`.
`fprintd.service` is `static` — it starts on demand when a PAM service calls it.

## 2. Verify the sensor is recognized

```bash
lsusb | grep -i validity
# Bus 001 Device 007: ID 138a:0090 Validity Sensors, Inc. VFS7500 ...
```

If the device is present but not listed via `fprintd-list`, update the system.

## 3. Enroll your fingerprints

```bash
fprintd-enroll                 # register (press your finger when prompted)
fprintd-list $USER             # list enrolled fingers
fprintd-verify $USER           # live check with one finger
```

Fingers that work reliably on this driver: the ones you set up. I enrolled six:
`left-middle`, `left-index`, `right-ring`, `right-middle`, `right-index`, `left-thumb`.

## 4. Storage configuration

`/etc/fprintd.conf`:

```ini
[storage]
type=file
```

## 5. PAM configuration — the critical part

For the reader to unlock sudo, the graphical login, and the login screen, you have
to add `pam_fprintd.so` as the **first `auth` line** with `sufficient`. That way a
matching finger passes immediately, and the password prompt only appears if the
finger is not recognized.

> I went through quite a battle here: if `pam_fprintd.so` is `required` instead of
> `sufficient`, an unregistered finger *blocks* authentication entirely. The whole
> point of these setups is having fingerprint as a bonus alongside the password
> fallback.

### `/etc/pam.d/sudo`

```
#%PAM-1.0
auth		sufficient	pam_fprintd.so
auth		include		system-auth
account		include		system-auth
session		include		system-auth
session		optional	pam_systemd.so class=none
```

### `/etc/pam.d/system-auth`

The `auth` section must start with:

```
auth       sufficient                pam_fprintd.so
auth       required                    pam_faillock.so      preauth
-auth      [success=2 default=ignore]  pam_systemd_home.so
auth       [success=1 default=bad]     pam_unix.so          try_first_pass nullok
...
```

### `/etc/pam.d/plasmalogin`

(Used by the KDE/Plasma login screen. On some distros this is `/etc/pam.d/kde`
or `/etc/pam.d/gdm` likewise.)

```
auth        sufficient  pam_fprintd.so
auth        include     system-login
-auth       optional    pam_gnome_keyring.so
-auth       optional    pam_kwallet5.so
...
```

## 6. Verification

```bash
fprintd-list $USER
fprintd-verify $USER
systemctl status fprintd     # should be active (running)
```

## Troubleshooting

- **Sensor not detected:** check `lsusb` for `138a:0090`. If missing, try a
  different USB port or check for firmware/BIOS toggle.
- **Fingerprint blocks login instead of falling back:** make sure every
  `pam_fprintd.so` line says `sufficient`, never `required`.
- **Only some fingers recognized:** the VFS0090 driver may be picky about
  coverage and press angle; re-enroll with a consistent position.

---
*Files referenced (exact copies of the working state on this machine):*
`/etc/pam.d/sudo`, `/etc/pam.d/system-auth`, `/etc/pam.d/plasmalogin`, `/etc/fprintd.conf`.
