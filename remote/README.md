# Plasma Login Manager remote access

This adds login-screen remote access to `plasmalogin`:

* RDP on port 3389 (`krdpserver --plasma`, streams the greeter over KWin's
  native `zkde_screencast` protocol).
* VNC on port 5900 (`krfb-virtualmonitor`, creates a virtual monitor that shows
  the login screen).
* Sunshine (Moonlight host) in the greeter session using KWin direct capture
  (`kwingrab`), avoiding the xdg-desktop-portal permission dialog.

All three services run as systemd user units in the `plasmalogin` greeter
session. They are pulled in by a drop-in for `plasma-login-wayland.target`, so
they start and stop together with the login screen. No changes to the
`plasmalogin` binary are required.

## Install

```sh
makepkg -si
sudo systemctl restart plasmalogin
```

Configuration lives in `/etc/plasmalogin-remote.conf` (a systemd
`EnvironmentFile`). Set a feature to `0` to disable it and change the default
passwords before exposing the machine to a network.

## systemd-homed / LUKS homes

`krdpserver` authenticates RDP users through PAM. Its stock PAM conversation
helper rejects informational `PAM_TEXT_INFO`/`PAM_ERROR_MSG` messages, which
breaks `pam_systemd_home` for LUKS-encrypted homes. Apply
[`krdp-pam-conv.patch`](krdp-pam-conv.patch) to the `krdp` source and rebuild it:

```sh
tar xf krdp-<version>.tar.xz
cd krdp-<version>
patch -p1 < /path/to/krdp-pam-conv.patch
cmake -B build -S . -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr \
  -DBUILD_EXAMPLES=OFF -DBUILD_TESTING=OFF
cmake --build build
sudo cmake --install build
```

## Authentication

RDP authenticates normal system users through PAM, so connect to
`localhost:3389` with any local account and that user's normal password. VNC
has no user concept; set the shared VNC password with
`PLASMALOGIN_REMOTE_VNC_PASSWORD` in `/etc/plasmalogin-remote.conf`, restart
plasmalogin, and then connect to port 5900 with that password. Moonlight pairs
with Sunshine as usual.

Note: the remote login screen is the *greeter*; after a user logs in, the
session starts on the local seat and the greeter (and therefore the RDP/VNC
stream) stops. This is "remote access to the login screen", not a
GDM-style per-user headless session.
