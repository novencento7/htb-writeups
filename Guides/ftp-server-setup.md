# Setting up an FTP server with vsftpd (anonymous read-only + read/write user)

This guide explains how to configure an FTP server on Linux (Debian/Ubuntu) using **vsftpd**, with two access levels to the **same shared folder**:

- **`anonymous` user**: **read-only** access
- **`test` user**: **read and write** access

The setup uses a secure two-level chroot structure (no need to disable vsftpd's built-in security check for writable chroot roots).

## Folder structure

```
/srv/ftp/shared/          ← chroot root, owned by root, NOT writable by "test"
└── test/                 ← working folder, owned by "test", writable
```

- `shared` is the chroot boundary for the `test` user **and** the root folder that `anonymous` reads from.
- `test` (inside `shared`) is the actual shared data folder: `anonymous` reads it, `test` reads and writes to it.

> Note: the subfolder is named `test` only because that's the username used in this example — it is **not** private to that user. It's simply the shared data folder both users access.

---

## 1. Install vsftpd

```bash
sudo apt update
sudo apt install vsftpd -y
```

Check that it's installed:

```bash
sudo systemctl status vsftpd
```

---

## 2. Create the folder structure

```bash
# Chroot root - not writable by "test"
sudo mkdir -p /srv/ftp/shared/test
sudo chown root:root /srv/ftp/shared
sudo chmod 755 /srv/ftp/shared

# Working folder - writable by "test", readable by anyone (including anonymous)
sudo chown test:test /srv/ftp/shared/test
sudo chmod 775 /srv/ftp/shared/test
```

> If the `chown test:test` command fails because the user doesn't exist yet, create the user first (step 3), then run this `chown`.

---

## 3. Create the `test` user

Create a system user whose home directory is the **chroot boundary** (`shared`), not the writable subfolder:

```bash
sudo adduser --home /srv/ftp/shared --no-create-home --shell /usr/sbin/nologin test
```

You'll be prompted to set a password — use a strong one.

Then make sure permissions are set as in step 2:

```bash
sudo chown test:test /srv/ftp/shared/test
sudo chmod 775 /srv/ftp/shared/test
```

`shared` stays owned by `root:root` with `755`: `test` can enter it (execute bit for "others") but cannot write directly into it — write access is limited to the `test/` subfolder.

### Important: register the login shell in `/etc/shells`

vsftpd (via PAM) checks that a local user's login shell is listed in `/etc/shells`. Since `/usr/sbin/nologin` is used here to prevent interactive/SSH login, it must still be **whitelisted** for FTP authentication to succeed — otherwise you'll get `530 Login incorrect` even with the correct password.

Check whether it's already listed:

```bash
cat /etc/shells
```

If `/usr/sbin/nologin` is missing, add it:

```bash
echo "/usr/sbin/nologin" | sudo tee -a /etc/shells
```

(If `/usr/sbin/nologin` doesn't exist on your system, check the actual path with `which nologin` and use that path consistently in both the `adduser` command and `/etc/shells`. `/bin/false` is an equally valid alternative if `nologin` is unavailable.)

---

## 4. Configure vsftpd

Edit the configuration file:

```bash
sudo nano /etc/vsftpd.conf
```

Set (or add) the following directives. Some of these are **not present by default** in the stock config file and need to be added manually — check with `grep -n "<directive>" /etc/vsftpd.conf` first to avoid duplicates.

```ini
# --- General ---
listen=YES
listen_ipv6=NO

# --- Local user access ("test") ---
local_enable=YES
write_enable=YES
local_umask=022

# Chroot: confine the local user to their home directory
chroot_local_user=YES
# NOTE: allow_writeable_chroot is NOT needed here, because the home
# ("shared") is not writable by the user — that's the whole point
# of this two-level structure: it avoids the writable-chroot-root
# security check entirely, instead of disabling it.

# --- Anonymous access ---
anonymous_enable=YES
anon_root=/srv/ftp/shared/test
anon_upload_enable=NO
anon_mkdir_write_enable=NO
anon_other_write_enable=NO
no_anon_password=YES

# --- Security / logging ---
xferlog_enable=YES
secure_chroot_dir=/var/run/vsftpd/empty
pam_service_name=vsftpd

# --- Passive port range (useful behind NAT/firewall) ---
pasv_enable=YES
pasv_min_port=40000
pasv_max_port=40100
```

### Key points

| Directive | Why |
|---|---|
| `chroot_local_user=YES` | Confines `test` to their home (`/srv/ftp/shared`) |
| *(absence of)* `allow_writeable_chroot` | Not needed: the home isn't writable, so vsftpd won't refuse the login |
| `anon_root=/srv/ftp/shared/test` | Anonymous lands **directly** in the shared working folder, read-only |
| `anon_upload_enable=NO` | Explicitly blocks anonymous uploads, on top of the Unix permissions |
| `no_anon_password=YES` | Skips the password prompt for anonymous logins (optional, purely for convenience) |

---

## 5. What each user sees

**`anonymous`** (FTP root = `/srv/ftp/shared/test`):
```
/          ← maps to /srv/ftp/shared/test
└── (existing files, read-only)
```
Can browse and download; cannot upload or modify anything — blocked both by `anon_upload_enable=NO` and by the `775` permissions (no write bit for "others").

**`test`** (FTP root = `/srv/ftp/shared`, real home):
```
/          ← maps to /srv/ftp/shared (not writable here)
└── test/  ← full read/write access here
```
After login, `cd test` is required to reach the writable area. This is the exact same physical folder (`/srv/ftp/shared/test`) that `anonymous` reads from.

---

## 6. Restart and test

```bash
sudo systemctl restart vsftpd
sudo systemctl enable vsftpd
```

### Test as anonymous

```bash
ftp <server-ip>
# Username: anonymous
# Password: (anything, or blank)
```

- `ls` should show the shared folder's contents directly.
- `get file.txt` should work.
- `put file.txt` should be **rejected**.

### Test as test

```bash
ftp <server-ip>
# Username: test
# Password: (the one set in step 3)
```

- On login you land in the chroot root (`/srv/ftp/shared`): a `put` here should **fail** (permission denied, not writable).
- `cd test`
- From here, both `get` and `put` should work.

If you get `530 Login incorrect` with the correct password, double-check `/etc/shells` contains the exact shell path used for the `test` user (see step 3).

---

## 7. Firewall

If using `ufw`:

```bash
sudo ufw allow 20/tcp
sudo ufw allow 21/tcp
sudo ufw allow 40000:40100/tcp
```

(Ports 40000–40100 match the passive range configured in step 4.)

---

## 8. Permissions summary

| Path | Owner | Permissions | Role |
|---|---|---|---|
| `/srv/ftp/shared` | `root:root` | `755` | Chroot boundary for `test`, not writable |
| `/srv/ftp/shared/test` | `test:test` | `775` | Working folder: anonymous reads, test reads and writes |

| User | Sees | Read | Write |
|---|---|---|---|
| `anonymous` | `/srv/ftp/shared/test` (as FTP root) | ✅ | ❌ |
| `test` | `/srv/ftp/shared` (root) → `test/` (writable) | ✅ | ✅ (only inside `test/`) |

---

## Security notes

- This structure requires **no security checks to be disabled**: separating the chroot boundary from the writable folder avoids the problem at the source.
- Plain FTP transmits credentials **unencrypted**. If the server is exposed to the internet, consider **FTPS** (`ssl_enable=YES` + a certificate) or **SFTP** (over SSH), generally the preferred option.
- Monitor access via `/var/log/vsftpd.log`.
- Make sure any login shell used for FTP-only users (e.g. `/usr/sbin/nologin` or `/bin/false`) is listed in `/etc/shells`, or PAM will reject the login with `530 Login incorrect`.
