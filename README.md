# Validity Sensors Fingerprint Reader on Kubuntu/Ubuntu 25.04 (Resolute)

> A guide to getting the **Validity Sensors `138a:0097`** fingerprint reader working on Kubuntu/Ubuntu 25.04 (Resolute) with Python 3.14.
> Tested on a **Lenovo ThinkPad** with the built-in Validity fingerprint reader.

---

## Device Info

| Field | Value |
|---|---|
| Vendor | Validity Sensors, Inc. |
| USB ID | `138a:0097` |
| OS | Kubuntu / Ubuntu 25.04 (Resolute) |
| Python | 3.14 |

---

## The Problem

The official `fprintd` package does not support this reader. The community driver `python-validity` works, but:

- Its PPA only supports up to Ubuntu 24.04 (Noble)
- On Ubuntu 25.04, the `cryptography` Python package has breaking changes incompatible with `python-validity`

This guide works around both issues safely without touching system packages.

---

## Prerequisites

Check that your device is the correct one:

```bash
lsusb | grep -i validity
```

You should see:
```
Bus 001 Device 004: ID 138a:0097 Validity Sensors, Inc.
```

Also confirm your Ubuntu version:
```bash
lsb_release -a
```

---

## Step 1 — Add the PPA (forced noble)

The PPA does not have a Resolute release, so we force the Noble version:

```bash
sudo add-apt-repository "deb https://ppa.launchpadcontent.net/uunicorn/open-fprintd/ubuntu noble main"
sudo apt update
```

---

## Step 2 — Install packages

```bash
sudo apt install open-fprintd python3-validity fprintd-clients
```

This will:
- Remove the stock `fprintd` and `libpam-fprintd`
- Install `open-fprintd`, `python3-validity`, and `fprintd-clients`
- Automatically download the Lenovo firmware from Lenovo's servers

> **Note:** If the firmware download fails with `FileNotFoundError: Directory does not exist: /var/run/python-validity/`, proceed to Step 3 — it will be fixed there.

---

## Step 3 — Fix the runtime directory

The directory `/var/run/python-validity/` is needed but gets cleared on every reboot. Create it persistently:

```bash
sudo mkdir -p /var/run/python-validity
sudo nano /etc/tmpfiles.d/python-validity.conf
```

Add this line:
```
d /var/run/python-validity 0755 root root -
```

Save and exit (`Ctrl+O`, Enter, `Ctrl+X`).

If the firmware wasn't copied during install, run it again now:
```bash
sudo validity-sensors-firmware
```

---

## Step 4 — Fix Python 3.14 compatibility with a virtual environment

The `python-validity` service crashes on Python 3.14 due to a breaking change in the `cryptography` module. The safe fix is to use a virtual environment — this does **not** touch any system packages.

```bash
# Create isolated virtual environment
sudo python3 -m venv /opt/validity-env --system-site-packages

# Install compatible cryptography version inside venv only
sudo /opt/validity-env/bin/pip install cryptography --upgrade
```

Now edit the systemd service to use the venv Python:

```bash
sudo nano /etc/systemd/system/python3-validity.service
```

Change the `ExecStart` line from:
```
ExecStart=/usr/lib/python-validity/dbus-service --debug
```
To:
```
ExecStart=/opt/validity-env/bin/python /usr/lib/python-validity/dbus-service --debug
```

Save and exit.

---

## Step 5 — Start the service

```bash
sudo systemctl daemon-reload
sudo systemctl restart python3-validity
sudo systemctl status python3-validity
```

You should see `Active: active (running)`. If you see TLS debug messages scrolling, the driver is communicating with the reader successfully.

---

## Step 6 — Enroll your fingerprint

```bash
fprintd-enroll
```

Swipe your finger multiple times as prompted. You should see repeated `enroll-stage-passed` messages followed by `enroll-complete`.

If it says a fingerprint is already enrolled, clear it first:
```bash
fprintd-delete $USER
fprintd-enroll
```

Verify it works:
```bash
fprintd-verify
```

---

## Step 7 — Enable fingerprint in PAM

```bash
sudo pam-auth-update
```

A menu will appear. Make sure **Fingerprint authentication** is checked. Press `Space` to toggle, `Tab` to navigate, `Enter` to confirm.

This automatically enables fingerprint for:
- ✅ `sudo`
- ✅ SDDM login screen
- ✅ KDE lock screen

---

## Step 8 — Test

Clear the sudo cache and test:
```bash
sudo -k
sudo ls
```

Swipe your finger when prompted — it should authenticate without a password!

---

## Persistence After Reboot

Everything persists across reboots because:
- `/opt/validity-env` is a permanent directory on disk
- `/etc/tmpfiles.d/python-validity.conf` recreates `/var/run/python-validity/` on every boot
- `python3-validity.service` is enabled and starts automatically

You can verify after reboot:
```bash
sudo systemctl status python3-validity
```

---

## Troubleshooting

**Service fails with `AttributeError: module 'cryptography.hazmat.bindings'`**
→ The venv cryptography fix wasn't applied. Redo Step 4.

**Service fails with `FileNotFoundError: /var/run/python-validity/`**
→ The runtime directory is missing. Redo Step 3.

**`fprintd-enroll` says fingerprint already exists**
→ Run `fprintd-delete $USER` then re-enroll.

**Fingerprint not working after reboot**
→ Check service status: `sudo systemctl status python3-validity`
→ If failed, check logs: `journalctl -u python3-validity -n 50`

---

## Credits

- [python-validity](https://github.com/uunicorn/python-validity) by uunicorn — reverse engineered Validity Sensors driver
- [open-fprintd](https://github.com/uunicorn/open-fprintd) by uunicorn — open fprintd replacement
