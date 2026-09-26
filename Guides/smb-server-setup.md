# Setting up an SMB server with Samba

This guide explains how to configure an SMB file server on Linux (Debian/Ubuntu) using Samba.

* * *

## 1. Install Samba

    sudo apt update
    sudo apt install samba -y

Check that it's installed:

    sudo systemctl status smbd

* * *

## 2. Create the shared folder

    sudo mkdir -p /srv/samba/share

Create a dedicated group:

    sudo groupadd sambashare

Add your user to the group:

    sudo usermod -aG sambashare $USER

Set ownership and permissions:

    sudo chown -R root:sambashare /srv/samba/share
    sudo chmod -R 2770 /srv/samba/share

> Log out and back in after adding your user to the `sambashare` group.

* * *

## 3. Create a Samba user

Samba uses its own password database.

Add your Linux user to Samba:

    sudo smbpasswd -a $USER

Enable the Samba account:

    sudo smbpasswd -e $USER

The Samba password can be different from the Linux password.

* * *

## 4. Configure Samba

The main Samba configuration file is:

    /etc/samba/smb.conf

Create a backup before editing:

    sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.backup

Edit the configuration:

    sudo nano /etc/samba/smb.conf

Add the following section at the end of the file:

    [Share]
        path = /srv/samba/share
        browseable = yes
        read only = no
        writable = yes
        valid users = @sambashare
        force group = sambashare
        create mask = 0660
        directory mask = 2770

### Key points

Option | Why
--- | ---
`path` | Directory exported through SMB
`browseable` | Makes the share visible when browsing the server
`read only` | `no` allows clients to write
`writable` | Enables write access
`valid users` | Restricts access to members of `sambashare`
`force group` | Forces files and directories to use the `sambashare` group
`create mask` | Permissions applied to newly created files
`directory mask` | Permissions applied to newly created directories

* * *

## 5. Validate the configuration

Before restarting Samba, check the configuration:

    testparm

If the configuration is valid, `testparm` should report that the configuration loaded successfully.

Fix any errors before restarting the service.

* * *

## 6. Restart Samba

Restart the service:

    sudo systemctl restart smbd

Enable Samba at boot:

    sudo systemctl enable smbd

Check the status:

    sudo systemctl status smbd

The service should show:

    active (running)

* * *

## 7. Check that SMB is listening

SMB normally listens on TCP port `445`.

Check with:

    sudo ss -tulpn | grep :445

You should see `smbd` listening on port `445`.

* * *

## 8. Configure the firewall

If using `ufw`:

    sudo ufw allow Samba

Check the firewall:

    sudo ufw status

> Do not expose SMB directly to the internet. SMB should normally be accessible only from the local network or through a VPN.

* * *

## 9. Find the server IP address

Use:

    hostname -I

or:

    ip addr

For example:

    192.168.1.100

* * *

## 10. Test from another Linux machine

Install the Samba client tools:

    sudo apt install smbclient -y

List the available shares:

    smbclient -L //<server-ip> -U <username>

For example:

    smbclient -L //192.168.1.100 -U username

Connect directly to the share:

    smbclient //<server-ip>/Share -U <username>

For example:

    smbclient //192.168.1.100/Share -U username

* * *

## 11. Mount the SMB share on Linux

Install CIFS utilities:

    sudo apt install cifs-utils -y

Create a mount point:

    sudo mkdir -p /mnt/smb-share

Mount the share:

    sudo mount -t cifs //<server-ip>/Share /mnt/smb-share -o username=<username>

You will be prompted for the Samba password.

Check the mount:

    mount | grep smb

* * *

## 12. Connect from Windows

Open File Explorer and enter:

    \\<server-ip>\Share

For example:

    \\192.168.1.100\Share

Enter the Samba username and password when prompted.

The share can also be mapped as a network drive:

    This PC → Map network drive

* * *

## 13. Test read and write access

From the client, create or copy a test file into the SMB share.

On the server, verify that the file exists:

    ls -la /srv/samba/share

Test both reading and writing.

Then remove the test file when finished.

* * *

## 14. Troubleshooting

### Samba service is inactive

Check the service:

    sudo systemctl status smbd

Restart it:

    sudo systemctl restart smbd

Check the logs:

    sudo journalctl -u smbd --no-pager

### Check the configuration

    testparm

A syntax error in `/etc/samba/smb.conf` can prevent Samba from starting correctly.

### Check port 445

    sudo ss -tulpn | grep :445

### Check the firewall

    sudo ufw status

If necessary:

    sudo ufw allow Samba

### Check Samba users

List Samba users:

    sudo pdbedit -L

Add a user:

    sudo smbpasswd -a username

Enable the user:

    sudo smbpasswd -e username

* * *

## 15. Useful Samba commands

Command | Purpose
--- | ---
`testparm` | Validate the Samba configuration
`sudo systemctl status smbd` | Check the Samba service
`sudo systemctl start smbd` | Start Samba
`sudo systemctl stop smbd` | Stop Samba
`sudo systemctl restart smbd` | Restart Samba
`sudo systemctl enable smbd` | Enable Samba at boot
`sudo pdbedit -L` | List Samba users
`sudo smbpasswd -a username` | Add a Samba user
`smbclient -L //<server-ip> -U username` | List available shares
`smbclient //<server-ip>/Share -U username` | Connect to a share

* * *

## Security notes

- Do not expose SMB directly to the public internet.
- Restrict SMB access to the local network whenever possible.
- For remote access, use a VPN such as WireGuard instead of exposing TCP port `445`.
- If SSH is enabled for administration, preferably make it accessible only through the VPN.
- Use strong Samba passwords.
- Keep Samba updated.

A recommended remote-access architecture is:

    Internet
        |
        v
    WireGuard VPN
        |
        v
    Home Network
        |
        +---- SMB Server
        |
        +---- SSH Server

This allows remote clients to access the private network through the VPN without exposing SMB directly to the internet.
