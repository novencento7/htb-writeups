# Setting up an NFS server

This guide explains how to install, configure, secure, and test an NFS (Network File System) server on Linux using `nfs-kernel-server`.

* * *

## 1. Install the NFS server

Update the package list:

    sudo apt update

Install the NFS server:

    sudo apt install nfs-kernel-server -y

Check the service:

    sudo systemctl status nfs-kernel-server

* * *

## 2. Create the shared directory

Create a directory that will be exported through NFS:

    sudo mkdir -p /srv/nfs/share

Set the ownership:

    sudo chown -R nobody:nogroup /srv/nfs/share

Set the permissions:

    sudo chmod 777 /srv/nfs/share

> The permissions used here are intentionally simple for a basic lab or home-network setup. For a production environment, use more restrictive ownership and permissions.

* * *

## 3. Configure the NFS export

The main NFS configuration file is:

    /etc/exports

Create a backup before editing it:

    sudo cp /etc/exports /etc/exports.backup

Edit the file:

    sudo nano /etc/exports

Add the following line:

    /srv/nfs/share 192.168.1.0/24(rw,sync,no_subtree_check)

Replace `192.168.1.0/24` with the subnet used by your local network.

For example, if your network is:

    192.168.1.0/24

the export allows clients with addresses in that subnet to access the share.

* * *

## 4. NFS export options

The following options are used in the example:

Option | Meaning
--- | ---
`rw` | Allows read and write access
`ro` | Read-only access
`sync` | The server replies only after changes have been committed to stable storage
`async` | Can improve performance but may increase the risk of data loss or corruption after a failure
`secure` | Requires non-GSS requests to originate from a privileged port below 1024
`no_subtree_check` | Disables subtree checking and can improve reliability/performance for some configurations
`root_squash` | Maps requests from the client root user to an anonymous user
`no_root_squash` | Allows the client root user to retain root privileges on the exported filesystem

> `root_squash` is enabled by default and is generally preferable from a security perspective. Avoid `no_root_squash` unless you specifically need it.

* * *

## 5. Apply the NFS configuration

After modifying `/etc/exports`, apply the configuration:

    sudo exportfs -a

Display the current exports:

    sudo exportfs -v

You should see something similar to:

    /srv/nfs/share
        192.168.1.0/24(rw,wdelay,sync,root_squash,no_subtree_check,secure)

The exact output may vary depending on the installed NFS version and configuration.

* * *

## 6. Restart the NFS server

Restart the service:

    sudo systemctl restart nfs-kernel-server

Enable it at boot:

    sudo systemctl enable nfs-kernel-server

Check the status:

    sudo systemctl status nfs-kernel-server

The service should show:

    active (running)

* * *

## 7. Check the exported directories

Use:

    sudo exportfs -v

You can also display the active export table:

    sudo exportfs

To check the NFS-related services:

    sudo systemctl status nfs-kernel-server

* * *

## 8. Configure the firewall

If UFW is enabled:

    sudo ufw status

NFS uses several services and ports depending on the NFS version and configuration.

For a simple NFS setup on a trusted local network, make sure the firewall allows the required NFS traffic from the local network.

> Do not expose NFS directly to the public Internet. Restrict access to trusted clients and networks.

* * *

## 9. Find the server IP address

Use:

    hostname -I

or:

    ip addr

For example:

    192.168.1.100

This is the IP address that clients will use to connect to the NFS server.

* * *

## 10. Install the NFS client

On the client machine, install the NFS client package:

    sudo apt update
    sudo apt install nfs-common -y

* * *

## 11. Discover available NFS shares

From the client, use:

    showmount -e <server-ip>

For example:

    showmount -e 192.168.1.100

You should see:

    Export list for 192.168.1.100:
    /srv/nfs/share 192.168.1.0/24

This confirms that the NFS server is exporting the directory.

* * *

## 12. Create a mount point on the client

Create a local directory:

    sudo mkdir -p /mnt/nfs-share

* * *

## 13. Mount the NFS share

Mount the exported directory:

    sudo mount 192.168.1.100:/srv/nfs/share /mnt/nfs-share

Replace `192.168.1.100` with the IP address of your NFS server.

Check that the share is mounted:

    mount | grep nfs

You can also use:

    df -h

You should see the NFS share in the output.

* * *

## 14. Test read and write access

Move to the mounted directory:

    cd /mnt/nfs-share

Create a test file:

    touch test.txt

Check that it exists:

    ls -la

On the NFS server, verify that the file is present:

    ls -la /srv/nfs/share

If the file appears on the server, the NFS share is working correctly.

Remove the test file:

    rm test.txt

* * *

## 15. Unmount the NFS share

To unmount the share:

    sudo umount /mnt/nfs-share

Verify that it is no longer mounted:

    mount | grep nfs

* * *

## 16. Mount the NFS share automatically at boot

To mount the NFS share automatically, edit `/etc/fstab` on the client:

    sudo nano /etc/fstab

Add:

    192.168.1.100:/srv/nfs/share /mnt/nfs-share nfs defaults,_netdev 0 0

Test the configuration without rebooting:

    sudo mount -a

If there are no errors, check:

    mount | grep nfs

> The `_netdev` option tells the system that the filesystem requires network connectivity.

* * *

## 17. NFS mount options

Common mount options include:

Option | Meaning
--- | ---
`rw` | Mount the filesystem with read/write access
`ro` | Mount the filesystem read-only
`sync` | Use synchronous I/O
`async` | Use asynchronous I/O
`_netdev` | Indicates that the filesystem requires the network
`hard` | Continue retrying requests if the server becomes unavailable
`soft` | Return an error after a timeout instead of continuing indefinitely
`nolock` | Disable file locking

> `nolock` is mainly relevant to older NFS configurations. Modern NFS setups generally use the standard locking mechanisms and do not require it.

* * *

## 18. Troubleshooting

### NFS service is inactive

Check the service:

    sudo systemctl status nfs-kernel-server

Restart it:

    sudo systemctl restart nfs-kernel-server

Check the logs:

    sudo journalctl -u nfs-kernel-server --no-pager

### Check the exports configuration

Display the active exports:

    sudo exportfs -v

Apply the configuration again:

    sudo exportfs -ra

### Check `/etc/exports`

Make sure the syntax is correct.

For example:

    /srv/nfs/share 192.168.1.0/24(rw,sync,no_subtree_check)

There must not be a space between the client/network and the opening parenthesis.

Correct:

    192.168.1.0/24(rw,sync)

Incorrect:

    192.168.1.0/24 (rw,sync)

### The client cannot see the share

From the client:

    showmount -e <server-ip>

For example:

    showmount -e 192.168.1.100

If the export is not displayed, check `/etc/exports` and the NFS service on the server.

### The client cannot mount the share

Check that the server is reachable:

    ping <server-ip>

Check the export:

    showmount -e <server-ip>

Check the NFS service:

    sudo systemctl status nfs-kernel-server

Check the firewall:

    sudo ufw status

### Permission denied

Check the permissions on the server:

    ls -ld /srv/nfs/share

Also check the ownership:

    ls -la /srv/nfs/share

Remember that NFS uses Linux filesystem permissions in addition to the export rules in `/etc/exports`.

* * *

## 19. Useful NFS commands

Command | Purpose
--- | ---
`sudo systemctl status nfs-kernel-server` | Check the NFS service
`sudo systemctl restart nfs-kernel-server` | Restart the NFS service
`sudo systemctl enable nfs-kernel-server` | Enable NFS at boot
`sudo exportfs -a` | Export all filesystems listed in `/etc/exports`
`sudo exportfs -ra` | Re-export all filesystems
`sudo exportfs -v` | Display active exports and options
`sudo exportfs` | Display exported filesystems
`showmount -e <server-ip>` | Display NFS exports from a server
`mount <server-ip>:/path /mountpoint` | Mount an NFS share
`sudo umount /mountpoint` | Unmount an NFS share

* * *

## Security notes

- Do not expose NFS directly to the public Internet.
- Restrict exports to specific IP addresses or trusted networks.
- Avoid using `*` in `/etc/exports` unless there is a specific reason.
- Prefer `root_squash` over `no_root_squash`.
- Use `ro` when clients only need read access.
- Use `rw` only when clients need to write to the share.
- Keep the NFS server and clients updated.
- Use a VPN such as WireGuard for remote access instead of exposing NFS directly to the Internet.

A recommended remote-access architecture is:

    Internet
        |
        v
    WireGuard VPN
        |
        v
    Home Network
        |
        +---- NFS Server
        |
        +---- SSH Server

This allows remote clients to access the private network through the VPN without exposing NFS directly to the Internet.

* * *

## 20. Final verification

On the NFS server:

    sudo systemctl status nfs-kernel-server

    sudo exportfs -v

On the client:

    showmount -e <server-ip>

    sudo mount <server-ip>:/srv/nfs/share /mnt/nfs-share

Then test:

    touch /mnt/nfs-share/test.txt

Verify the file on the server:

    ls -la /srv/nfs/share

If the file is visible on both machines, the NFS server and client are working correctly.
