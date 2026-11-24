# Project

This is the Arch Linux packaging repository for
[`plasma-login-manager`](https://invent.kde.org/plasma/plasma-login-manager),
KDE Plasma's display manager (a fork/replacement of SDDM). The source tarball is
downloaded by the PKGBUILD; the remote-access work lives in `remote/` and is
installed by the PKGBUILD's `package()`.

# Environment

- Arch Linux, x86_64.
- `plasmalogin` is installed from upstream pacman and controlled by
  `sudo systemctl start/stop plasmalogin`.
- GPU: AMD RX 7600 (`card0`, discrete) + AMD Raphael 610M (`card1`, integrated).
  The physical monitor is connected to `card1-HDMI-A-1`.
- At least one user is a `systemd-homed` user with a LUKS-encrypted home
  (relevant because RDP authentication goes through `pam_systemd_home`).
- `sudo` requires a password; there is no passwordless sudo. `faillock` is active.

# Goal / priorities

1. RDP multiseat access from the login screen, allowing remote login to any
   installed Wayland session (preferably Plasma).
2. Sunshine (Moonlight host) working at the login screen.
3. VNC server on port 5900 at the login screen.

# Current status

## RDP (priority 1) — mostly done

- `krdpserver` runs in the greeter session on `0.0.0.0:3389`.
- Auth is PAM (`SystemUserEnabled=true`), so any local account works, including
  `systemd-homed` LUKS accounts.
- The stream shows the physical login screen (`--monitor 0`), not the VNC
  virtual monitor.
- After login, the greeter exits and the RDP connection ends; the session starts
  on the physical seat. This is login-screen access, NOT GDM-style per-user
  headless RDP sessions. True headless remote sessions are a separate, larger
  feature and are not implemented.

Important fixes that made RDP work:

- `krdpserver` must use `QT_QPA_PLATFORM=wayland`. The Qt `minimal` platform
  leaves it without a Wayland connection to KWin, producing a black stream.
- `krdpserver`'s stock PAM conversation rejects informational
  `PAM_TEXT_INFO`/`PAM_ERROR_MSG` messages, which breaks `pam_systemd_home`.
  Apply `remote/krdp-pam-conv.patch` and rebuild `krdp` (build dependency:
  `plasma-wayland-protocols`). The rebuilt binary is currently installed manually.
- A self-signed TLS certificate must exist at
  `/var/lib/plasmalogin/.config/krdp/{server.crt,server.key}` (the package
  install scriptlet generates it). Without it krdpserver exits with a TLS error.

## VNC (priority 3) — view-only, not interactive

- `krfb-virtualmonitor` listens on `0.0.0.0:5900` and shows a virtual monitor
  with the login screen.
- Input does not work: krfb's Wayland input goes through `xdg-desktop-portal`
  (`xdp` events plugin), which is not running in the greeter session.
- To make it interactive: start `xdg-desktop-portal` +
  `xdg-desktop-portal-kde` + PipeWire in the greeter, pre-authorize krfb via the
  KDE portal headless "mega-authorization" (`org.freedesktop.impl.portal.PermissionStore`,
  table `kde-authorized`, id `remote-desktop`, app_id `""`), switch to
  portal-based `krfb`, and patch `xdg-desktop-portal-kde` so the ScreenCast
  dialog also honors mega-authorization.

## Sunshine (priority 2) — deferred

- Sunshine is a standalone process, not integrated with plasmalogin. It was
  crash-looping (SIGABRT) in the greeter session; root cause not fully pinned
  down. Treat it separately.
- The shipped default disables it (`PLASMALOGIN_REMOTE_SUNSHINE=0`).
- Note: Sunshine has a `kwingrab` backend for KWin direct capture and ships a
  KWin permission desktop file; when revisited, that is the intended path (not
  KMS, which conflicts with KWin's DRM master and multi-GPU selection).

# Architecture knowledge

The greeter runs as the `plasmalogin` user (uid 926) in its own systemd user
manager:

```text
plasmalogin.service (root)
  └─ startplasma-login-wayland (user plasmalogin)
       └─ plasma-login-wayland.target
            ├─ plasma-login-kwin_wayland.service   (KWin login compositor)
            └─ plasma-login.service                 (greeter QML UI)
```

Remote services are user units wired into that target via a drop-in:

```text
plasma-login-wayland.target.d/plasmalogin-remote.conf
  ├─ plasmalogin-rdp.service      → /usr/lib/plasmalogin/remote-rdp
  ├─ plasmalogin-vnc.service      → /usr/lib/plasmalogin/remote-vnc
  └─ plasmalogin-sunshine.service → /usr/lib/plasmalogin/remote-sunshine
```

Config: `/etc/plasmalogin-remote.conf` (systemd `EnvironmentFile`).

Protocol/component facts:

- KWin exposes `zkde_screencast_unstable_v1` (capture) and
  `org.kde.KWin.EIS.RemoteDesktop` (input). `krdpserver --plasma` uses both
  directly, no portal needed.
- `krfb-virtualmonitor` uses `zkde_screencast` for video but the `xdp` events
  plugin (portal) for input.
- User session login PAM service is `plasmalogin` (includes
  `system-login`/`system-auth`/`pam_systemd_home`). RDP pre-auth uses the `login`
  PAM service.
- PipeWire must run in the greeter session for screencast consumers.

# Operation

Install / update and restart:

```sh
makepkg -si
sudo systemctl restart plasmalogin
```

Test:

```sh
xfreerdp3 /v:localhost:3389 /u:<username> /cert:ignore
vncviewer localhost:5900
```

Set the VNC password in `/etc/plasmalogin-remote.conf`
(`PLASMALOGIN_REMOTE_VNC_PASSWORD`, 8 chars effective) and restart plasmalogin.

# Pitfalls / notes

- `pam_faillock` (default `deny=3`) locks the account after three failed
  authentications, which also breaks `sudo`. Wrong-password RDP/PAM tests can
  trigger this. Reset with `sudo faillock --user <user> --reset`; consider
  raising `deny` in `/etc/security/faillock.conf`.
- Manual replacement of `/usr/bin/krdpserver` is reverted by `pacman -S krdp`.
- `journalctl _UID=926` reads the plasmalogin greeter user journal;
  `journalctl --user` reads the caller's user journal.
- Do not spend effort on Sunshine inside plasmalogin; it is a separate concern.
