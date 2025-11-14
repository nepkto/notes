# Linux User Accounts and Permissions Guide

**Jump to Section:**
- [1. User Accounts](#1-user-accounts)
- [2. Adding New Users](#2-adding-new-users)
- [3. Creating a Sudo User on Ubuntu](#3-creating-a-sudo-user-on-ubuntu)
- [4. Password Management](#4-password-management)
- [5. File Permissions](#5-file-permissions)
- [6. Ownership & Apache Permissions](#6-ownership--apache-permissions)
- [7. Listing Users and Groups](#7-listing-users-and-groups)
- [8. Best Practices](#8-best-practices)

---

## 1. User Accounts

User account information is maintained in `/etc/passwd`, with each user having a line:
```
loginName:x:uid:gid:userInfo:homeDir:initialProgram
```
- `x`: Indicates password is stored in `/etc/shadow`.
- Encrypted passwords are kept in `/etc/shadow`.

**Authentication:**  
Both `/etc/passwd` and `/etc/shadow` are used to authenticate users.

---

## 2. Adding New Users

- **Add a new user:**
  ```bash
  sudo useradd username
  ```
- **Add and create home directory:**
  ```bash
  sudo useradd -m username
  ```
- **Add with custom comment (user info):**
  ```bash
  sudo useradd -c "Test User Account" username
  ```
- **Add with expiry date:**
  ```bash
  sudo useradd -e YYYY-MM-DD username
  ```
- **Verify expiry date:**
  ```bash
  sudo chage -l username
  ```

### Granting Sudo Privileges

To allow a user to run commands with `sudo`, the user must be listed in `/etc/sudoers`.
- Use `visudo` to safely edit the file:
  ```
  username ALL=(ALL) ALL
  ```

---

## 3. Creating a Sudo User on Ubuntu

1. **Add a New User Account:**
    ```bash
    sudo adduser example_user
    ```
2. **Add the User to the Sudo Group:**
    ```bash
    sudo usermod -aG sudo example_user
    ```
    > This gives the user the ability to run commands with sudo privileges.

3. **Test (Switch to the new user):**
    ```bash
    su - example_user
    ```

---

## 4. Password Management

- **Change another user's password (as root):**
  ```bash
  sudo passwd username
  ```
- **Change own password:**
  ```bash
  passwd
  ```
- **Switch user:**
  ```bash
  su - username
  ```

Passwords are stored encrypted in `/etc/shadow`. Account information is in `/etc/passwd`.

---

## 5. File Permissions

Linux permissions use three categories:
- Owner (u)
- Group (g)
- Others (o)

### Changing Permissions

- **Make file executable for all users:**
  ```bash
  chmod a+x formmail.cgi
  ```
- **Set read, write, execute for owner only:**
  ```bash
  chmod u=rwx formmail.cgi
  ```
- **Remove write permission for group and others:**
  ```bash
  chmod go-w formmail.cgi
  ```

#### Permission Numbers (Octal Representation):

| Number | rwx | Meaning         |
|--------|-----|-----------------|
| 0      | --- | No permission   |
| 1      | --x | Execute only    |
| 2      | -w- | Write only      |
| 3      | -wx | Write & execute |
| 4      | r-- | Read only       |
| 5      | r-x | Read & execute  |
| 6      | rw- | Read & write    |
| 7      | rwx | Full            |

---

## 6. Ownership & Apache Permissions

- **Change ownership recursively:**
  ```bash
  sudo chown -R $USER:$USER /var/www/test1
  ```
  - `$USER`: currently logged-in user

- **Web server typical permissions:**
  ```bash
  sudo chown -R www-data:www-data /var/www/site1.your_domain
  sudo chmod -R 755 /var/www/site1.your_domain
  ```
  - 755: Owner (read/write/execute), group & others (read/execute)

---

## 7. Listing Users and Groups

### Groups:
- **Show groups for current user:**
  ```bash
  groups
  ```
- **Show all groups:**
  ```bash
  less /etc/group
  ```

### Users:
- **List user information:**
  ```bash
  getent passwd <optional_user>
  ```

#### Reference
- [UNIX/OSX permissions guide](https://www.comentum.com/unix-osx-permissions.html)

---

## 8. Best Practices

- Always use `visudo` when editing sudoers file to avoid syntax errors.
- Use strong passwords and change them regularly.
- Assign the least necessary permission to files and directories.
- Regularly review user accounts and remove unused ones.

---

**This guide covers basic and essential commands for managing user accounts and permissions on Linux systems. For more advanced topics, consult the references or man pages (e.g., `man useradd`, `man chmod`, `man chown`).**