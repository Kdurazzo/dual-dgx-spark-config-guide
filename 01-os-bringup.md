# 01: OS Bring-up (from boot)

This section takes each DGX Spark from power-on to a base-configured OS:
hostname set, clock synced, operator user confirmed with sudo, passwordless
SSH between the two nodes, and firmware current. At the end the two nodes are
ready for the networking section (02), which wires up the QSFP RoCE data plane
and the Tailscale management plane.

Scope note: this section covers the Founders Edition (FE). OEM variants (MSI
EdgeXpert, ASUS Ascent GX10) ship their own media and procedures in places;
where that matters it is called out inline. There is no BMC/IPMI on the Spark,
so everything below is done over a physical display/keyboard or the first-boot
network hotspot, never a remote console.

## 0. Reference versions (verified)

Verified against NVIDIA's current DGX Spark User Guide as of 2026-09-03.

| Component | Version |
| --- | --- |
| NVIDIA DGX OS | 7.5.0 |
| Canonical Kernel | 6.17 |
| NVIDIA GPU Driver | 580.159.03 |
| NVIDIA CUDA Toolkit | 13.0.2 |
| UEFI | 1.110.13 |
| Embedded Controller (EC) | 3.5.8 |
| USB Power Delivery (USB PD) | 0.5.22 |
| Trusted Platform Module (TPM) | 7.516.1 |
| System on Chip (SoC) | 2.155.11 |

Source: https://docs.nvidia.com/dgx/dgx-spark/release-notes.html (Current
Software Versions table). These versions apply to the Founders Edition only;
GB10 partner systems may not receive updates on the same schedule. The exact
point releases change over time, so confirm against that page rather than
treating the table above as fixed.

DGX OS 7.5.0 is Ubuntu-based (ARM64). The Spark runs a Canonical kernel plus
NVIDIA's customized OS layer. The OS version string you will see is
"DGX OS 7.5.0" or a later 7.x, not a stock Ubuntu release string.

## 1. Prerequisites (physical)

Per node, have:

- The DGX Spark and its supplied power adapter.
- A display. If connecting over USB-C/DisplayPort and nothing shows, try HDMI;
  this is a documented out-of-box quirk, not a fault.
- A wired USB keyboard and mouse. Bluetooth keyboards can pair later, but a
  wired keyboard is required for UEFI work (some Bluetooth keyboards, even ones
  with a USB dongle, are not recognized during UEFI/recovery).
- A network path to the internet during first boot (wired Ethernet recommended;
  captive-portal or unstable Wi-Fi is not recommended for the initial update
  download).
- The QSFP direct-connect cable for the data plane. It is not needed for this
  section, but have it ready for section 02.
- A 16 GB or larger USB drive, only if you need system recovery (section 6).

Important: the DGX Spark powers on immediately when power is applied. Attach
the display, keyboard, mouse, and network cable before plugging in the power
supply.

## 2. First boot and initial setup

The first-time setup utility (NVIDIA calls it the OOBE, out-of-box experience)
runs automatically on first power-on. There are two ways to drive it.

Local setup (recommended for a clustering build): attach display, keyboard,
mouse, and Ethernet, then apply power. The wizard appears on the display.

Network appliance setup: apply power with no display. The Spark brings up a
Wi-Fi hotspot whose SSID and password are on the Quick Start Guide sticker; join
it from another computer and complete setup in a browser. This path needs a
display/keyboard only if it fails to join your network, so for a deterministic
two-node build, local setup is simpler.

The wizard walks through, in order:

1. Language and time zone selection.
2. Keyboard layout selection (local setup only).
3. Terms and conditions acceptance.
4. User account creation (username and password). The username must be
   lowercase letters; it becomes the administrative account used in place of
   root. Choose a strong password.
5. Optional analytics/crash-reporting preference.
6. Wi-Fi selection (skipped if a wired Ethernet link with internet is present).
7. Software download and installation. The system downloads and installs the
   full image, possibly rebooting more than once. This can take on the order of
   10 minutes. Do not power down or reboot during this step; interrupting it
   can corrupt the install.

After the initial setup, the Spark also prompts for a GRUB password (minimum 8
characters). Setting it is recommended; leaving it empty disables GRUB
protection. This is the DGX Station/DGX Spark first-boot path, distinct from
the enterprise DGX server path (the Spark has no BMC, so there is no BMC
administrator step).

Once complete, the system lands at a login prompt. Record the account username
and hostname; you will use both below.

## 3. OS image and variants

The Founders Edition ships with DGX OS preinstalled; there is no separate OS
install step in the normal path, only the first-boot wizard above.

- Founders Edition: recovery media is a tar.gz archive, not an ISO, and is
  downloaded from developer.nvidia.com (no enterprise account needed). Do not
  use the Enterprise Download Center or a DGX OS ISO for the Spark; the Spark's
  recovery process is different from enterprise DGX systems.
- OEM variants (MSI EdgeXpert, ASUS Ascent GX10): recovery images and
  procedures come only from the respective OEM, and may differ from the FE
  procedure. Point readers at the OEM's documentation for those units.

## 4. Post-install configuration

Run the following on each node. Commands use spark1/spark2 as the node names
and operator as the example username; substitute your values. Any command requiring
sudo will prompt for the password set during first boot.

### 4.1 Hostnames

On the first node:

    sudo hostnamectl set-hostname spark1

On the second node:

    sudo hostnamectl set-hostname spark2

Verify:

    hostnamectl

Expected output includes:

      Static hostname: spark1

For the two nodes to resolve each other by name, add entries to /etc/hosts on
each node (or rely on Tailscale MagicDNS, configured in section 02). A minimal
/etc/hosts addition, using the management-plane addresses, is:

    100.64.0.1   spark1
    100.64.0.2   spark2

Replace the addresses with your real management IPs; leave this out if you are
using Tailscale MagicDNS.

### 4.2 Time sync and timezone

DGX OS is Ubuntu-based and uses systemd-timesyncd for NTP by default. Confirm
it is active and set the timezone (example, US Pacific):

    timedatectl set-timezone America/Los_Angeles
    timedatectl

Expected output shows:

      System clock synchronized: yes
      Time zone: America/Los_Angeles (PDT, -0700)

If "System clock synchronized" stays "no", check the service:

    systemctl status systemd-timesyncd

If you prefer chrony, install and enable it instead (sudo apt install chrony,
then sudo systemctl enable --now chrony), which disables timesyncd. Either is
fine for a two-node cluster; the requirement is simply that both nodes keep
time close enough for SSH and logging to agree. Set the same timezone on both
nodes.

### 4.3 Operator user and sudo

The account created during first boot already has sudo. Confirm membership:

    groups

Expected output includes the sudo group, for example:

    operator adm cdrom sudo dip plugdev lpadmin

Confirm sudo works without re-entering (it caches for a few minutes):

    sudo -v

If you need to add a second operator account later, use a different name:

    sudo adduser operator2
    sudo usermod -aG sudo operator2

### 4.4 SSH keys (passwordless both ways)

An alternative first: NVIDIA's `dgx-spark-playbooks` repo ships a
`discover-sparks` script (in the `connect-two-sparks` playbook, under
`assets/`) that automates this whole step. It discovers the peer node over the
direct-connect link using the mDNS `.local` hostname, then sets up
bidirectional passwordless SSH between the two nodes in one pass. To use it,
copy the script to one node and run `bash ./discover-sparks`; it prompts once
for each node's password and prints when both directions are working. Source:
https://github.com/NVIDIA/dgx-spark-playbooks

This guide uses the manual steps below as the primary path, so every action is
explicit and easy to debug; the script is a faster one-shot alternative if you
prefer it.

Generate a key on each node if one does not exist. Using ed25519:

On spark1:

    ssh-keygen -t ed25519 -C "operator@spark1" -f ~/.ssh/id_ed25519 -N ""

On spark2:

    ssh-keygen -t ed25519 -C "operator@spark2" -f ~/.ssh/id_ed25519 -N ""

Distribute the public keys. This step requires the two nodes to reach each
other; use the LAN/Tailscale address you configured in 4.1 (section 02 makes
this the permanent management path). From spark1:

    ssh-copy-id operator@spark2

From spark2:

    ssh-copy-id operator@spark1

Each ssh-copy-id prompts once for the target account's password, then installs
the key. Verify both directions with no password prompt:

From spark1:

    ssh spark2 hostname

Expected output:

    spark2

From spark2:

    ssh spark1 hostname

Expected output:

    spark1

If the hostnames do not resolve, substitute the management IP or Tailscale name
in place of spark1/spark2 until section 02 wires up name resolution.

## 5. Firmware

The firmware path on the Spark is fwupdmgr. Check what is installed:

    sudo fwupdmgr get-devices

Refresh metadata and see what is available:

    sudo fwupdmgr refresh
    sudo fwupdmgr get-updates

Apply updates (fwupdmgr update and fwupdmgr upgrade are the same operation;
current NVIDIA docs show upgrade, older fwupdmgr releases used update):

    sudo fwupdmgr update

Then reboot to apply:

    sudo reboot

Expected output: get-updates either lists devices with available firmware
versions (for example UEFI, Embedded Controller, SoC) or reports "Devices with
no available firmware updates". After update and reboot, re-run get-devices
and confirm the versions match the table in section 0 (or a later release).

The Founders Edition ships with the lvfs-testing remote disabled. If
get-updates reports nothing even though the release notes list newer firmware,
first run sudo fwupdmgr get-remotes and confirm a stable remote (lvfs) is
enabled. Enabling the testing remote is optional and not recommended for a
production cluster, but if you need a specific newer firmware before it lands
on the stable remote:

    sudo fwupdmgr enable-remote lvfs-testing
    sudo fwupdmgr refresh
    sudo fwupdmgr get-updates

To roll back a regression:

    sudo fwupdmgr downgrade <device-id>

(Use the device id from get-devices output.)

One caveat that bites clustering builds: firmware updates have been observed to
reset the direct-connect interfaces' MTU to 1500. Section 02 sets MTU 9000, but
after any firmware update here, re-verify MTU and re-apply it if needed. There
is also no CLI factory reset on the Spark; the "reset defaults" commands seen
in enterprise DGX documentation belong to DGX-100 Series systems with a BMC and
do not apply here.

## 6. System recovery (reference)

This is a rebuild path, not the normal path. Use it for corruption, a fatal
misconfiguration, or an unbootable node. Physical access is required; there is
no remote or BMC path.

Requirements: a 16 GB or larger USB drive, a wired USB keyboard, and a display.
Download the recovery media from NVIDIA's DGX Spark System Recovery page
(developer.nvidia.com, no enterprise account needed). The FE archive is a
tar.gz, not an ISO; extract it and build the USB with the included script
(CreateUSBKey.sh on Linux, CreateUSBKey.cmd on Windows, CreateUSBKeyMacOS.sh on
macOS), run with administrator privileges. This erases the USB drive.

Then, with the USB in the Spark and a wired keyboard attached:

1. Power on and hold Esc or Del to enter UEFI settings. Use a keyboard plugged
   directly into a USB port on the Spark; some Bluetooth keyboards are not
   recognized here.
2. On the "Save & Exit" page, choose "Restore Defaults", answer "Yes" to "Load
   Optimized Defaults", then "Save Changes and Reset".
3. When it reboots, hold Esc or Del again. On the "Security" page, confirm
   "Secure Boot" is "Enabled", select "Restore Factory Keys", then on "Save and
   Exit" choose "Save Changes and Reset".
4. On the next reboot, hold Esc or Del a third time. On the "Save & Exit" page,
   move to the "Boot Override" section and select the USB drive.
5. The device boots the recovery environment. On the welcome screen press
   Enter. On the warning screen, choose [START RECOVERY] to wipe the internal
   SSD and reflash firmware and OS (or [EXIT] to abort). The reflash runs on
   screen, then the device reboots to the first-boot wizard.

The menu labels above ("Save & Exit", "Restore Defaults", "Load Optimized
Defaults", "Security", "Secure Boot", "Restore Factory Keys", "Boot Override",
"[START RECOVERY]", "[EXIT]") are the exact labels in NVIDIA's current
system-recovery documentation, verified 2026-09-03. Official reference:
https://docs.nvidia.com/dgx/dgx-spark/system-recovery.html

After recovery, re-run section 4 (hostname, time, SSH keys) and section 5
(firmware), then continue to section 02 for networking.

## Done criteria

Both nodes are at the same state when you can answer yes to all of these:

- hostnamectl shows the intended static hostname on each node.
- timedatectl shows "System clock synchronized: yes" and the same timezone on
  both nodes.
- ssh spark2 hostname (from spark1) returns "spark2" with no password, and the
  reverse returns "spark1".
- sudo fwupdmgr get-updates reports no pending firmware (or you have applied
  the pending updates and rebooted), and get-devices matches the current
  release-notes versions.
