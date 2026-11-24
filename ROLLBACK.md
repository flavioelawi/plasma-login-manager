# Rollback to Arch-managed packages

This reverts the system from the forked remote-access package back to the stock
`pacman`-provided `plasmalogin` and `krdp` packages. Run the steps in order.

## What this undoes

- The patched `/usr/bin/krdpserver` and `/usr/lib/libKRdp.so.6.8.3`.
- The `plasmalogin-rdp`, `plasmalogin-vnc` and `plasmalogin-sunshine` systemd
  user units and the `plasma-login-wayland.target` drop-in.
- The launcher scripts and config files installed under `/usr/lib/plasmalogin`,
  `/etc/plasmalogin-remote.conf`, and the greeter user's state in
  `/var/lib/plasmalogin/.config`.

## Steps

```sh
# 1. Stop the display manager while reverting.
sudo systemctl stop plasmalogin

# 2. Remove the systemd user units and the target drop-in.
sudo rm -f /usr/lib/systemd/user/plasmalogin-rdp.service
sudo rm -f /usr/lib/systemd/user/plasmalogin-vnc.service
sudo rm -f /usr/lib/systemd/user/plasmalogin-sunshine.service
sudo rm -f /usr/lib/systemd/user/plasma-login-wayland.target.d/plasmalogin-remote.conf
sudo rmdir /usr/lib/systemd/user/plasma-login-wayland.target.d 2>/dev/null || true

# 3. Remove the launcher scripts and the krdp PAM config we installed.
sudo rm -f /usr/lib/plasmalogin/remote-rdp
sudo rm -f /usr/lib/plasmalogin/remote-vnc
sudo rm -f /usr/lib/plasmalogin/remote-sunshine
sudo rm -f /usr/lib/plasmalogin/krdpserverrc

# 4. Remove the config and greeter-session state we created.
sudo rm -f /etc/plasmalogin-remote.conf
sudo rm -rf /var/lib/plasmalogin/.config/krdp
sudo rm -f /var/lib/plasmalogin/.config/krdpserverrc
sudo rm -rf /var/lib/plasmalogin/.config/sunshine

# 5. Restore the original krdp binaries from pacman.
sudo pacman -S --noconfirm krdp

# 6. Make sure nothing else was touched in the stock packages.
sudo pacman -S --noconfirm plasmalogin

# 7. Reload systemd and start the display manager again.
sudo systemctl daemon-reload
sudo systemctl start plasmalogin
```

The `rm -rf` targets above are the exact directories this project created; do
not broaden them. The greeter user's real configuration (other files under
`/var/lib/plasmalogin/.config`) is left untouched.

## Optional cleanup

`plasma-wayland-protocols` was installed only as a build dependency for the
patched krdp. If nothing else needs it:

```sh
sudo pacman -Rs plasma-wayland-protocols
```

## Verify

```sh
# Stock package integrity (both should report no modified/extra files, or only
# expected config-file changes).
sudo pacman -Qkk krdp plasmalogin

# The RDP/VNC services should no longer be referenced.
systemctl list-unit-files | grep -E 'plasmalogin-(rdp|vnc|sunshine)' || echo "clean"
```

## Repository revert (optional)

The working tree changes can be discarded, but note this also removes the
documentation files:

```sh
git restore PKGBUILD
git clean -fd
```

This deletes `remote/`, `README.md`, `AGENTS.md`, `UPSTREAM.md`, and
`ROLLBACK.md` from the checkout. Only run it if you also want the repository
back to its original state.
