# Upstream status: remote desktop / remote login work in KDE

Findings gathered from the official KDE GitLab (invent.kde.org) on 2026-11-24.

## The main effort: system-level RDP multiseat

There is active, official work on exactly the "Windows-like" RDP multiseat
feature described in this project.

### krdp issue #50 — "System level RDP integration"

- URL: https://invent.kde.org/plasma/krdp/-/issues/50
- Author: David Edmundson
- Created: 2026-09-03, last updated: 2026-09-10
- Status: open

This is the design the user asked about: a system-level daemon owns the RDP
socket, clients connect and authenticate, then receive a redirection token and
reconnect to a freshly spawned per-user `krdpserver` backed by that user's
session. It explicitly cites GNOME's headless remote-session design.

The phased plan in the issue:

1. Introduce a daemon with the RDP server, auth, and debug output — done.
2. On each connection, spawn `krdpserver` and hand over the FD when an SSL
   redirection token is present; otherwise issue a token and ask the client to
   reconnect — done.
3. D-Bus control while running as the same user; connect with `xfreerdp` and
   start streaming.
4. Run `krdp-daemon` as a system user and start thinking seriously about
   security.
5. KCM handles root-level certificates and start/stop; `krdp-daemon` reads
   logind sessions and spawns a new session where relevant.
6. Security review.
7. Ship.

The issue's own caveat: the work is "woefully insecure" and will need a large
rebase/squash before it becomes a merge request. No ETA.

### Work-in-progress branches in `plasma/krdp`

- `work/olib-tp/krdp-daemon` — system daemon work, adds a `krdp-daemon/`
  directory. Latest commits 2026-09-07 by David Edmundson.
  https://invent.kde.org/plasma/krdp/-/tree/work/olib-tp/krdp-daemon
- `work/d_ed/multilogin` — multi-user / PAM login experiments (2025).
- `work/d_ed/pam_login_config` — PAM auth and dynamic monitor sizes.
- `work/d_ed/login_notification` — login notifications; updated 2026-11-05.

There is no clean public merge request yet; the work lives in personal `work/`
branches.

## Related KWin work

- KWin MR #9972 (draft, by `mkenfenheuer`): "backends/drm: Apply virtual-only
  configurations off the seat" — enables virtual outputs for remote sessions
  while the physical seat is inactive.
  https://invent.kde.org/plasma/kwin/-/merge_requests/9972
- KWin issue #331 (David Edmundson, 2026-10-30): "UX & behaviour proposal for
  multi-seat input" — an exploration of multi-seat input in Wayland, explicitly
  not a promise of implementation.
  https://invent.kde.org/plasma/kwin/-/issues/331

## Gaps

- `plasma-login-manager` itself has no active RDP/VNC remote-login work; its
  README still lists remote (VNC/RDP) support from startup as a future plan.
- VNC has no equivalent upstream effort. `krfb` is oriented around sharing an
  existing user session, and there is no visible work on login-screen or
  multiseat VNC. The interactive-login-VNC work scoped in this repository
  (portal pre-authorization) remains an unowned gap.

## Implication for this repository

The RDP-to-login-screen integration here is a stopgap until upstream
`krdp` #50 ships. True per-user headless RDP multiseat should not be
re-implemented independently; it is better to track and eventually consume the
upstream `krdp-daemon` work.
