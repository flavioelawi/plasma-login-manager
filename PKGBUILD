# Maintainer: Antonio Rojas <arojas@archlinux.org>

pkgname=plasma-login-manager
pkgver=6.8.3
_dirver=$(echo $pkgver | cut -d. -f1-3)
pkgrel=1
arch=(x86_64)
pkgdesc='Plasma Login Manager'
url='https://kde.org/plasma-desktop/'
license=(GPL-2.0-or-later)
install=plasma-login-manager.install
backup=('etc/plasmalogin-remote.conf')
depends=(glibc
         kauth
         kcmutils
         kconfig
         kcoreaddons
         kdbusaddons
         ki18n
         kio
         kirigami
         kpackage
         kservice
         krdp
         krfb
         layer-shell-qt
         libkscreen
         libplasma
         libstdc++
         libxau
         openssl
         pam
         pipewire
         plasma-workspace
         qt6-5compat
         qt6-base
         qt6-declarative
         sh
         systemd-libs
         wireplumber)
makedepends=(extra-cmake-modules
             qt6-tools)
optdepends=('sunshine: Moonlight host for the login screen')
groups=(plasma)
source=(https://download.kde.org/stable/plasma/$_dirver/$pkgname-$pkgver.tar.xz{,.sig})
sha256sums=('5969f5a6bbf23a911db5dc1a44cc0ba6f341ce3af8edfe3374d75610b39a8d9d'
            'SKIP')
validpgpkeys=('E0A3EB202F8E57528E13E72FD7574483BB57B18D'  # Jonathan Esk-Riddell <jr@jriddell.org>
              '58743C0A93487794BC44ECE873C3D7758E7A544A'  # Bhushan Shah <bshah@kde.org>
              'D07BD8662C56CB291B316EB2F5675605C74E02CF'  # David Edmundson <davidedmundson@kde.org>
              '1FA881591C26B276D7A5518EEAAF29B42A678C20') # Marco Martin <notmart@gmail.com>

build() {
  cmake -B build -S $pkgname-$pkgver \
    -DKDE_INSTALL_LIBEXECDIR=lib \
    -DDBUS_CONFIG_FILENAME=plasma_org.freedesktop.DisplayManager.conf
  cmake --build build
}

package() {
  DESTDIR="$pkgdir" cmake --install build

  # Plasma Login remote access: RDP, VNC and Sunshine for the login screen
  install -Dm755 "$startdir/remote/usr/lib/plasmalogin/remote-rdp" \
    "$pkgdir/usr/lib/plasmalogin/remote-rdp"
  install -Dm755 "$startdir/remote/usr/lib/plasmalogin/remote-vnc" \
    "$pkgdir/usr/lib/plasmalogin/remote-vnc"
  install -Dm755 "$startdir/remote/usr/lib/plasmalogin/remote-sunshine" \
    "$pkgdir/usr/lib/plasmalogin/remote-sunshine"
  install -Dm644 "$startdir/remote/usr/lib/plasmalogin/sunshine/apps.json" \
    "$pkgdir/usr/lib/plasmalogin/sunshine/apps.json"
  install -Dm644 "$startdir/remote/usr/lib/plasmalogin/krdpserverrc" \
    "$pkgdir/usr/lib/plasmalogin/krdpserverrc"

  install -Dm644 "$startdir/remote/usr/lib/systemd/user/plasmalogin-rdp.service" \
    "$pkgdir/usr/lib/systemd/user/plasmalogin-rdp.service"
  install -Dm644 "$startdir/remote/usr/lib/systemd/user/plasmalogin-vnc.service" \
    "$pkgdir/usr/lib/systemd/user/plasmalogin-vnc.service"
  install -Dm644 "$startdir/remote/usr/lib/systemd/user/plasmalogin-sunshine.service" \
    "$pkgdir/usr/lib/systemd/user/plasmalogin-sunshine.service"
  install -Dm644 "$startdir/remote/usr/lib/systemd/user/plasma-login-wayland.target.d/plasmalogin-remote.conf" \
    "$pkgdir/usr/lib/systemd/user/plasma-login-wayland.target.d/plasmalogin-remote.conf"

  install -Dm644 "$startdir/remote/etc/plasmalogin-remote.conf" \
    "$pkgdir/etc/plasmalogin-remote.conf"
}
