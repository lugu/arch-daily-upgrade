# Write out the install script on-the-fly when the PKGBUILD is parsed
cat <<'EOF' >arch-daily-upgrade.install
post_install() {
    # Create the dedicated system user without a home directory in /home
    if ! id "pkgbuilder" &>/dev/null; then
        useradd -r -M -d /var/lib/pkgbuilder -s /bin/bash pkgbuilder
    fi

    systemctl daemon-reload
    systemctl enable --now arch-daily-upgrade.timer
}

post_upgrade() {
    post_install
}

pre_remove() {
    systemctl disable --now arch-daily-upgrade.timer
}
EOF

pkgname=arch-daily-upgrade
pkgver=6.9
pkgrel=1
pkgdesc="Fully automated daily system upgrade (Official and AUR) for Arch Linux using native systemd"
arch=('any')
url=""
license=('GPL')
depends=('pacman' 'sudo')
optdepends=('yay: AUR package support' 'yay-bin: Pre-compiled AUR package support')
backup=('usr/bin/arch-daily-upgrade.sh')
install=arch-daily-upgrade.install

package() {
  # 1. Create the upgrade script with clear operational logging
  install -Dm755 /dev/stdin "$pkgdir/usr/bin/arch-daily-upgrade.sh" <<'EOF'
#!/bin/bash
LOG_TAG="[arch-daily-upgrade]"
echo "$LOG_TAG Starting daily system upgrade at $(date)"

# Upgrade official repositories and clean cache
echo "$LOG_TAG Running official pacman upgrade..."
pacman -Syu --noconfirm
echo "$LOG_TAG Pacman upgrade complete. Cleaning package cache..."
yes | pacman -Scc

# Upgrade AUR packages if yay/yay-bin is available using a temporary runtime home
BUILD_USER="pkgbuilder"
if command -v yay &> /dev/null && id "$BUILD_USER" &>/dev/null; then
    echo "$LOG_TAG yay detected. Preparing temporary build environment for $BUILD_USER..."
    TEMP_HOME=$(mktemp -d -p /var/tmp)
    chown "$BUILD_USER":"$BUILD_USER" "$TEMP_HOME"

    echo "$LOG_TAG Running AUR upgrade via yay..."
    su -s /bin/bash -c "export HOME='$TEMP_HOME'; yay -Sua --noconfirm" "$BUILD_USER"

    echo "$LOG_TAG AUR upgrade complete. Cleaning temporary home and package cache..."
    rm -rf "$TEMP_HOME"
    yes | pacman -Scc
else
    echo "$LOG_TAG yay not found or build user missing, skipping AUR upgrade."
fi

echo "$LOG_TAG Daily system upgrade finished successfully at $(date)"
EOF

  # 2. Create the systemd service unit
  install -Dm644 /dev/stdin "$pkgdir/usr/lib/systemd/system/arch-daily-upgrade.service" <<'EOF'
[Unit]
Description=Automated Daily Arch Linux System Upgrade (Official and AUR)
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/bin/arch-daily-upgrade.sh
EOF

  # 3. Create the systemd timer unit (4 AM + 1-hour randomized jitter)
  install -Dm644 /dev/stdin "$pkgdir/usr/lib/systemd/system/arch-daily-upgrade.timer" <<'EOF'
[Unit]
Description=Run daily Arch Linux upgrade at 4 AM with jitter

[Timer]
OnCalendar=*-*-* 04:00:00
Persistent=true
WakeSystem=true
RandomizedDelaySec=1h

[Install]
WantedBy=timers.target
EOF

  # 4. Grant passwordless pacman execution to pkgbuilder for yay/makepkg
  install -dm750 "$pkgdir/etc/sudoers.d"
  install -m440 /dev/stdin "$pkgdir/etc/sudoers.d/pkgbuilder" <<'EOF'
pkgbuilder ALL=(ALL) NOPASSWD: /usr/bin/pacman
EOF
}
