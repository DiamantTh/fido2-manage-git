# Maintainer: jaime [at] jamezrin [dot] name
_pkgname=fido2-manage
_libfido2ver=1.15.0
pkgname=$_pkgname-git
pkgver=r192.81b4155
pkgrel=1
pkgdesc="Rust FIDO2.1 CLI and Token2 Key Manager GUI for FIDO2, PIV and OTP devices"
arch=('x86_64')
url="https://github.com/Token2/fido2-manage"
license=('BSD-2-Clause')
depends=('glibc' 'libgcc' 'libcbor' 'pcsclite' 'openssl' 'systemd-libs'
         'zlib' 'gtk3' 'webkit2gtk-4.1' 'glib2' 'cairo' 'gdk-pixbuf2'
         'libsoup3' 'dbus' 'wayland' 'hicolor-icon-theme')
optdepends=('ccid: USB CCID smart card reader driver'
            'pcsc-tools: smart card reader utilities')
makedepends=('git' 'cmake' 'pkgconf' 'cargo')
provides=('fido2-manage')
conflicts=('fido2-manage')
# Cargo controls release LTO itself; do not add makepkg's C/Rust LTO flags.
options=('!lto')
source=("git+https://github.com/Token2/fido2-manage.git#branch=main"
        "libfido2::git+https://github.com/Yubico/libfido2.git#tag=$_libfido2ver")
sha256sums=('SKIP' 'SKIP')

pkgver() {
  cd "$srcdir/$_pkgname"
  printf 'r%s.%s' "$(git rev-list --count HEAD)" "$(git rev-parse --short HEAD)"
}

prepare() {
  # Keep Cargo downloads and build output inside the makepkg workspace.
  export CARGO_HOME="$srcdir/cargo-home"
  export CARGO_TARGET_DIR="$srcdir/cargo-target"

  cd "$srcdir/libfido2"
  # The Token2 transport and APDU extensions are not in the system libfido2.
  git apply --ignore-whitespace "$srcdir/$_pkgname/libfido2-token2.patch"
  # Preserve Arch's FORTIFY_SOURCE setting instead of redefining it as 2.
  sed -i '/set(CMAKE_C_FLAGS_RELEASE .* -D_FORTIFY_SOURCE=2/d' CMakeLists.txt

  cd "$srcdir/$_pkgname/token2-piv-gui/src-tauri"
  # Linux libraries are installed by package(), not bundled as Tauri resources.
  sed -i '/"libs\/\*"/d' tauri.conf.json
  cargo fetch --locked --target "$CARCH-unknown-linux-gnu"
}

build() {
  export CARGO_HOME="$srcdir/cargo-home"
  export CARGO_TARGET_DIR="$srcdir/cargo-target"

  # Build only the PIV library needed by the Rust CLI and GUI.
  cmake -S "$srcdir/$_pkgname/token2-piv-tool" -B "$srcdir/build-piv" \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/usr \
    -DBUILD_ONLY_LIB=ON \
    -DBUILD_STATIC_LIB=OFF \
    -DGENERATE_MAN_PAGES=OFF \
    -DENABLE_HARDWARE_TESTS=OFF
  cmake --build "$srcdir/build-piv" --target t2piv_shared

  # Static linkage keeps Token2's patched libfido2 separate from the system ABI.
  cmake -S "$srcdir/libfido2" -B "$srcdir/build-fido2" \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/usr \
    -DUSE_PCSC=ON \
    -DBUILD_STATIC_LIBS=ON \
    -DBUILD_SHARED_LIBS=OFF \
    -DBUILD_TOOLS=ON \
    -DBUILD_EXAMPLES=OFF \
    -DBUILD_TESTS=OFF \
    -DBUILD_MANPAGES=OFF
  cmake --build "$srcdir/build-fido2" --target fido2 fido2-token fido2-cred fido2-assert

  export T2PIV_LIB_DIR="$srcdir/build-piv/lib"
  export FIDO2_LIB_DIR="$srcdir/build-fido2/src"
  export FIDO2_LIB_NAME=fido2
  export FIDO2_LINK_STATIC=1
  export RUSTFLAGS="${RUSTFLAGS:+$RUSTFLAGS }-C link-arg=-Wl,-rpath,/usr/lib/$_pkgname"

  cd "$srcdir/$_pkgname/token2-piv-gui/src-tauri"
  cargo build --frozen --release --features tauri/custom-protocol --bin fido2-manage --bin token2-key-manager
}

package() {
  local tool
  install -Dm755 "$srcdir/cargo-target/release/fido2-manage" "$pkgdir/usr/bin/fido2-manage"
  install -Dm755 "$srcdir/cargo-target/release/token2-key-manager" "$pkgdir/usr/bin/token2-key-manager"
  # Keep the previous GUI entry point for existing shortcuts and scripts.
  ln -s token2-key-manager "$pkgdir/usr/bin/fido2-manage-gui"

  # The GUI invokes token2-fido2-token next to its executable for key notes.
  for tool in token cred assert; do
    install -Dm755 "$srcdir/build-fido2/tools/fido2-$tool" "$pkgdir/usr/bin/token2-fido2-$tool"
  done
  # Keep the old low-level tool name as an alias as well.
  ln -s token2-fido2-token "$pkgdir/usr/bin/fido2-token2"

  install -dm755 "$pkgdir/usr/lib/$_pkgname"
  cp -d --no-preserve=ownership "$srcdir/build-piv/lib/"libt2piv.so* "$pkgdir/usr/lib/$_pkgname/"

  install -Dm644 "$srcdir/$_pkgname/token2-piv-gui/src-tauri/icons/icon.png" \
    "$pkgdir/usr/share/icons/hicolor/256x256/apps/fido2-manage.png"
  install -Dm644 /dev/stdin "$pkgdir/usr/share/applications/fido2-manage.desktop" <<'DESKTOP'
[Desktop Entry]
Name=Token2 Key Manager
Comment=Manage FIDO2 passkeys, PIV certificates and OTP accounts
Exec=fido2-manage-gui
Icon=fido2-manage
Terminal=false
Type=Application
Categories=System;Security;
Keywords=FIDO2;Passkey;PIV;OTP;Security;Authentication;USB;NFC;
DESKTOP

  install -Dm644 "$srcdir/libfido2/LICENSE" "$pkgdir/usr/share/licenses/$pkgname/LICENSE.libfido2"
  install -Dm644 "$srcdir/$_pkgname/token2-piv-tool/COPYING" "$pkgdir/usr/share/licenses/$pkgname/COPYING.libt2piv"
  # Current upstream declares the application's license in README.md.
  install -Dm644 "$srcdir/$_pkgname/README.md" "$pkgdir/usr/share/licenses/$pkgname/README.md"
  install -Dm644 "$srcdir/$_pkgname/README.md" "$pkgdir/usr/share/doc/$pkgname/README.md"
  install -Dm644 "$srcdir/$_pkgname/token2-piv-gui/FIDO2-MANAGE.md" \
    "$pkgdir/usr/share/doc/$pkgname/FIDO2-MANAGE.md"
}
