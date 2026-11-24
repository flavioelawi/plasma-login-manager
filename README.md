# Plasma Login Manager — remote access (RDP / VNC / Sunshine)

This repository is the Arch packaging repo for
[`plasma-login-manager`](https://invent.kde.org/plasma/plasma-login-manager), KDE
Plasma's display manager (an SDDM replacement). The work here adds remote access
to the login screen: an RDP server, a VNC server, and (deferred) Sunshine
hosting.

## Status

| Area | Status |
| --- | --- |
| RDP on port 3389 | Working: interactive login screen, PAM auth for real system users (including systemd-homed/LUKS) |
| VNC on port 5900 | Partially working: shows the login screen, but is currently **view-only** (no input) |
| Sunshine / Moonlight | Deferred — separate process, to be handled independently |

See the sections below for the exact limitations.

## What works today

- `krdpserver` starts inside the `plasmalogin` greeter session and listens on
  `0.0.0.0:3389`.
- RDP clients connect with a normal local account (username + that user's
  password).
  Authentication goes through PAM and works for `systemd-homed` LUKS accounts.
- The RDP stream shows the physical login screen (monitor index 0). The VNC
  virtual monitor is not duplicated into the RDP view.
- `krfb-virtualmonitor` listens on `0.0.0.0:5900` and presents a virtual monitor
  with the login screen, protected by a shared VNC password.

## Known limitations

### RDP

- This is "RDP access to the login screen". Once a user logs in, the greeter
  exits and the session starts on the physical seat, so the RDP connection ends.
  It is not a GDM-style per-user headless session that keeps running inside the
  RDP window. That would require a headless-KWin-per-user launcher and is a
  larger, separate feature.
- Login takes a few seconds because `pam_systemd_home` unlocks the LUKS home.

### VNC

- `krfb-virtualmonitor` streams the virtual monitor through KWin's native
  `zkde_screencast` protocol, but krfb injects Wayland input through
  `xdg-desktop-portal`, which is not running in the greeter session. Therefore
  VNC is currently view-only.
- Making VNC interactive requires starting `xdg-desktop-portal` +
  `xdg-desktop-portal-kde` in the greeter, pre-authorizing krfb via the KDE
  portal's headless "mega-authorization", switching to portal-based `krfb`, and
  patching `xdg-desktop-portal-kde` so screen sharing also honors that
  pre-authorization.

### Sunshine

- Sunshine is intentionally not integrated with plasmalogin. It is a separate
  process and was crash-looping when run in the greeter session, so the shipped
  default disables it (`PLASMALOGIN_REMOTE_SUNSHINE=0`). It should be treated as
  its own task.

## Architecture

The greeter runs as the `plasmalogin` user in its own systemd user manager:

```text
plasmalogin.service (root)
  └─ startplasma-login-wayland (user plasmalogin)
       └─ plasma-login-wayland.target
            ├─ plasma-login-kwin_wayland.service   (KWin login compositor)
            └─ plasma-login.service                 (greeter QML UI)
```

The remote services are systemd **user** units pulled into
`plasma-login-wayland.target` by a drop-in:

```text
plasma-login-wayland.target.d/plasmalogin-remote.conf
  ├─ plasmalogin-rdp.service      → /usr/lib/plasmalogin/remote-rdp
  ├─ plasmalogin-vnc.service      → /usr/lib/plasmalogin/remote-vnc
  └─ plasmalogin-sunshine.service → /usr/lib/plasmalogin/remote-sunshine
```

Configuration is a systemd `EnvironmentFile` at
`/etc/plasmalogin-remote.conf`.

Key facts that drive the implementation:

- KWin exposes `zkde_screencast_unstable_v1` for capture and
  `org.kde.KWin.EIS.RemoteDesktop` for input. `krdpserver --plasma` uses both
  directly, so it does not need the portal.
- `krdpserver` must run with `QT_QPA_PLATFORM=wayland`; with the Qt `minimal`
  platform it has no Wayland connection to KWin and the RDP stream is black.
- `krdpserver` authenticates through the PAM `login` service. Its stock PAM
  conversation helper rejects `PAM_TEXT_INFO`/`PAM_ERROR_MSG`, which breaks
  `pam_systemd_home`. The fix is in `remote/krdp-pam-conv.patch` and requires a
  rebuilt `krdp`.
- The actual user session login (after the login screen) uses plasmalogin's own
  PAM service (`plasmalogin`), which includes `system-login`/`system-auth` and
  therefore full `pam_systemd_home` support.
- `krfb-virtualmonitor` captures with `zkde_screencast` but has no native input
  path; its Wayland input goes through the portal (`xdp` events plugin), hence
  view-only in the greeter.

## Files

```text
remote/
  etc/plasmalogin-remote.conf                        user config (EnvironmentFile)
  krdp-pam-conv.patch                                krdp PAM fix
  usr/lib/plasmalogin/
    remote-rdp, remote-vnc, remote-sunshine          launcher scripts
    krdpserverrc                                     enables PAM auth
    sunshine/apps.json                               minimal Sunshine app list
  usr/lib/systemd/user/
    plasmalogin-rdp.service
    plasmalogin-vnc.service
    plasmalogin-sunshine.service
    plasma-login-wayland.target.d/plasmalogin-remote.conf
plasma-login-manager.install                         install scriptlet (runtime setup)
PKGBUILD                                             builds and installs the above
```

## Install / update

```sh
makepkg -si
sudo systemctl restart plasmalogin
```

The package install scriptlet is idempotent: it creates the greeter account's
state directories, copies default Sunshine/krdp configuration only when absent,
and generates a self-signed RDP TLS certificate under
`/var/lib/plasmalogin/.config/krdp/`. `backup=(etc/plasmalogin-remote.conf)`
keeps an existing configuration across upgrades.

## Configuration

Edit `/etc/plasmalogin-remote.conf`:

```ini
PLASMALOGIN_REMOTE_RDP=1
PLASMALOGIN_REMOTE_RDP_PORT=3389
PLASMALOGIN_REMOTE_RDP_MONITOR=0
PLASMALOGIN_REMOTE_RDP_MODE=SharedAccess

PLASMALOGIN_REMOTE_VNC=1
PLASMALOGIN_REMOTE_VNC_PORT=5900
PLASMALOGIN_REMOTE_VNC_PASSWORD=ChangeMeNow
PLASMALOGIN_REMOTE_VNC_RESOLUTION=1920x1080

PLASMALOGIN_REMOTE_SUNSHINE=0
```

RDP uses PAM, so there is no separate RDP password. VNC has no user concept;
set `PLASMALOGIN_REMOTE_VNC_PASSWORD` (classic VNC auth uses the first 8
characters). Restart plasmalogin after changing the file.

## Testing

```sh
# RDP
xfreerdp3 /v:localhost:3389 /u:<username> /cert:ignore

# VNC (view-only for now)
vncviewer localhost:5900
```

## Pitfalls

- `pam_faillock` (default `deny=3`) locks the account after a few wrong
  passwords, which also breaks `sudo`. Reset with
  `sudo faillock --user <user> --reset`. Consider raising `deny` in
  `/etc/security/faillock.conf`.
- Replacing `/usr/bin/krdpserver` manually means `pacman -S krdp` will revert
  the PAM patch; rebuild/reinstall as described in `remote/krdp-pam-conv.patch`.
