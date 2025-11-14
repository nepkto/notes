# Secure Copy (SCP) Commands Guide

This guide explains how to use the `scp` command to securely copy files and directories between your local machine and a remote server.

## What is SCP?

SCP (Secure Copy Protocol) allows you to transfer files or directories securely between hosts on a network using SSH for authentication and encryption.

---

## Copying Files or Folders Using SCP

### 1. Copying from Server to Local Machine

To copy a **file** or an **entire directory** from a remote server to your local machine, use:

```bash
scp -r username@server_ip:/remote/path/to/file_or_folder /local/destination/path
```

**Example:**

```bash
scp -r user@ipaddress:/var/www/example.com /home/anil/Desktop/
```
- `-r` flag is used to copy directories recursively.
- Replace `user` with your server username.
- Replace `ipaddress` with your server IP or domain.

### 2. Copying from Local Machine to Server

To copy a **file** or a **folder** from your local machine to the remote server, use:

```bash
scp /local/path/to/file_or_folder username@server_ip:/remote/destination/path
```

**Example:**

```bash
scp storage/app/public/img/default-profile.jpg cwnepal@207.148.124.199:/var/www/uat.allsaman.com/storage/app/public/img
```
- No `-r` flag needed for a single file.
- Use `-r` if copying a directory.

---

## Additional Notes

- **Authentication:** SCP requires SSH credentials (username and password, or SSH key).
- **Ports:** If your server uses a non-default SSH port, add `-P port_number`:
  ```bash
  scp -P 2222 file.txt user@server_ip:/path/
  ```
- **Key-based Authentication:** To use SSH keys, make sure your public key is added to `~/.ssh/authorized_keys` on the remote server.

---

## Examples

#### 1. Copy a directory from server to local
```bash
scp -r user@ipaddress:/remote/folder /local/folder
```

#### 2. Copy a file from local to server
```bash
scp localfile.txt user@ipaddress:/remote/path/
```

#### 3. Copy a directory from local to server
```bash
scp -r /local/folder user@ipaddress:/remote/folder/
```

#### 4. Copy a file from server to local with custom SSH port
```bash
scp -P 2200 user@ipaddress:/remote/file.txt /local/path/
```

---

## Troubleshooting

- **Permission Denied:** Make sure you have the appropriate file permissions and SSH access.
- **Command Not Found:** If SCP is not installed, you can typically install it via your package manager:
  - For Ubuntu/Debian: `sudo apt install openssh-client`
  - For CentOS/RHEL: `sudo yum install openssh-clients`

---

## References

- [SCP Manual](https://linux.die.net/man/1/scp)
- [OpenSSH Documentation](https://www.openssh.com/manual.html)
