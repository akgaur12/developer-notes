# Chapter 06 — Permissions & Ownership

## Learning Objectives

By the end of this chapter, you will:
- Read and understand Linux file permission strings
- Change file permissions with `chmod` using both numeric and symbolic modes
- Change file ownership with `chown` and `chgrp`
- Understand users, groups, and how they relate to permissions
- Know about special permissions: SUID, SGID, and Sticky Bit
- Apply correct permissions in real DevOps scenarios
- Use `sudo` safely

## Prerequisites

- Chapter 03 — File & Directory Commands

---

## 6.1 Why Permissions Matter in DevOps

Security begins at the filesystem. Wrong permissions cause:
- **Applications that can't read config files** (500 errors on startup)
- **Security vulnerabilities** (world-readable passwords, SUID exploits)
- **Scripts that can't execute** (permission denied)
- **Users who can't write to directories** they need

You'll set permissions constantly:
- After copying SSH keys (`~/.ssh/authorized_keys` must be `600`)
- When deploying applications (web server needs to read files)
- Writing deployment scripts that need `+x`
- Securing database config files with passwords

---

## 6.2 Linux Permission Model

Every file and directory in Linux has:
1. An **owner** (a user)
2. An **owning group** (a group)
3. Three sets of **permissions** (rwx for owner, group, others)

### The Permission String

```bash
ls -la /etc/nginx/nginx.conf
# -rw-r--r-- 1 root root 1234 Jun 23 nginx.conf
#  │││││││││
#  │││││││└┴─ other: r-- (read only)
#  ││││└┴┴─── group: r-- (read only)
#  │└┴┴─────── owner: rw- (read and write)
#  └─────────── file type: - (regular file)
```

### Decoding the 10-Character Permission String

```
Position 1: File type
  -  regular file
  d  directory
  l  symbolic link
  c  character device
  b  block device
  p  named pipe

Positions 2-4: Owner permissions (user)
Positions 5-7: Group permissions
Positions 8-10: Other permissions (everyone else)

Each set of 3:
  r  read    (4)
  w  write   (2)
  x  execute (1)
  -  no permission (0)
```

### Common Permission Examples

```
-rwxr-xr-x  regular file, owner=rwx, group=r-x, other=r-x
-rw-r--r--  regular file, owner=rw, group=r, other=r
-rw-rw-r--  regular file, owner=rw, group=rw, other=r
-rwx------  regular file, owner=rwx, group=none, other=none
drwxr-xr-x  directory, owner=rwx, group=r-x, other=r-x
drwx------  directory, owner only has access
```

### What Read/Write/Execute Means for Files vs Directories

| Permission | On Files | On Directories |
|-----------|----------|----------------|
| `r` (read) | Read file contents (`cat`, `less`) | List directory contents (`ls`) |
| `w` (write) | Modify file contents | Create/delete files inside dir |
| `x` (execute) | Run as a program | Enter the directory (`cd`) |

> **Key insight:** To `cd` into a directory, you need **execute** permission on it. To list its contents, you need **read**. Both for normal navigation.

---

## 6.3 Numeric (Octal) Permissions

Each permission is a bit:
- `r` = 4
- `w` = 2
- `x` = 1
- `-` = 0

Add them for each set:

```
rwx = 4+2+1 = 7
rw- = 4+2+0 = 6
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4
-wx = 0+2+1 = 3
-w- = 0+2+0 = 2
--x = 0+0+1 = 1
--- = 0+0+0 = 0
```

Three digits for owner/group/other:

```
755 = rwx r-x r-x    (typical executable or directory)
644 = rw- r-- r--    (typical file)
600 = rw- --- ---    (private file: SSH keys, secrets)
700 = rwx --- ---    (private directory)
777 = rwx rwx rwx    (DANGER: world-writable — avoid!)
```

### Memorize These

| Permission | Use Case |
|-----------|----------|
| `644` | Regular files (config, data) |
| `755` | Directories, executables |
| `600` | Private files (SSH keys, passwords) |
| `700` | Private directories |
| `640` | Files readable by owner + group |
| `750` | Directories for owner + group |
| `777` | Never use on production |

---

## 6.4 `chmod` — Change Mode (Permissions)

### Numeric Mode

```bash
chmod 644 file.txt          # rw-r--r--
chmod 755 script.sh         # rwxr-xr-x
chmod 600 ~/.ssh/id_rsa     # rw-------
chmod 700 ~/.ssh/           # rwx------
chmod 755 /var/www/html     # rwxr-xr-x

chmod -R 755 directory/     # recursive: apply to all files inside
chmod -R 644 /var/www/      # all files readable
```

### Symbolic Mode

Instead of numbers, use letters:

```
u = user (owner)
g = group
o = other
a = all (u+g+o)

+ = add permission
- = remove permission
= = set exact permission
```

```bash
chmod +x script.sh          # add execute for all
chmod u+x script.sh         # add execute for owner only
chmod go-w file.txt         # remove write from group and other
chmod a+r file.txt          # add read for everyone
chmod u=rw,go=r file.txt    # set exact: owner=rw, group=r, other=r
chmod o= file.txt           # remove ALL permissions from other

# Practical
chmod +x deploy.sh          # make script executable
chmod -R g+w /shared/dir/   # group can write to shared directory
```

### Common chmod Tasks

```bash
# Make a script executable
chmod +x script.sh
./script.sh            # now you can run it

# Secure SSH private key (REQUIRED — SSH refuses to use unsafe keys)
chmod 600 ~/.ssh/id_rsa
chmod 700 ~/.ssh/

# Set web server permissions
chmod 755 /var/www/html                    # directories
find /var/www -type f -exec chmod 644 {} \;  # files
find /var/www -type d -exec chmod 755 {} \;  # directories

# Secure a config file with passwords
chmod 600 /etc/myapp/config.env
```

---

## 6.5 `chown` and `chgrp` — Change Ownership

### `chown` — Change Owner (and Group)

```bash
chown user file.txt              # change owner
chown user:group file.txt        # change owner AND group
chown :group file.txt            # change group only (note: no user before :)
chown -R user:group directory/   # recursive

# Examples
sudo chown www-data /var/www/html    # nginx/apache user owns web files
sudo chown -R deploy:deploy /opt/app/
sudo chown akash:akash ~/myfile.txt
```

### `chgrp` — Change Group

```bash
chgrp developers file.txt
chgrp -R www-data /var/www/
```

### `id` — See Your User and Group IDs

```bash
id
# uid=1000(akash) gid=1000(akash) groups=1000(akash),4(adm),27(sudo),1001(docker)
```

---

## 6.6 Users and Groups

### View Users

```bash
cat /etc/passwd           # all users
cut -d: -f1 /etc/passwd   # just usernames
id username               # info about a specific user
whoami                    # current user
```

### View Groups

```bash
cat /etc/group            # all groups
groups                    # groups current user belongs to
groups username           # groups a specific user belongs to
```

### Manage Users (requires sudo)

```bash
sudo useradd username                    # create user (no home dir by default)
sudo useradd -m -s /bin/bash username    # with home dir and bash shell
sudo usermod -aG docker akash            # add user to docker group
sudo usermod -aG sudo akash              # add user to sudo group
sudo userdel username                    # delete user
sudo userdel -r username                 # delete user and home dir

# Set/change password
sudo passwd username
```

### Manage Groups

```bash
sudo groupadd developers      # create group
sudo groupdel developers      # delete group
sudo gpasswd -a user group    # add user to group
sudo gpasswd -d user group    # remove user from group
```

> **Important:** After adding yourself to a group (`sudo usermod -aG docker $USER`), you must **log out and log back in** for the group membership to take effect.

---

## 6.7 `sudo` — Superuser Do

`sudo` lets authorized users run commands as root (or another user) without needing the root password.

```bash
sudo command                   # run as root
sudo -u username command       # run as specific user
sudo -i                        # interactive root shell (be careful)
sudo !!                        # re-run last command as sudo
```

### The sudoers File

```bash
sudo visudo                    # safely edit /etc/sudoers
```

Example entries:
```
# User privilege specification
akash   ALL=(ALL:ALL) ALL      # akash can run any command as any user

# Group-based
%sudo   ALL=(ALL:ALL) ALL      # all users in sudo group can sudo

# No password required for specific commands
deploy  ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx
```

### sudo Best Practices

```bash
# NEVER do this in scripts (runs whole script as root)
sudo bash myscript.sh

# Do this instead: use sudo for specific commands inside script
sudo systemctl restart nginx
sudo cp config.conf /etc/myapp/

# Check what you can sudo
sudo -l
```

---

## 6.8 Special Permissions: SUID, SGID, Sticky Bit

### SUID (Set User ID) — Bit `4`

When set on an executable, it **runs as the file owner** regardless of who executes it.

```bash
ls -la /usr/bin/passwd
# -rwsr-xr-x 1 root root ... /usr/bin/passwd
#     ^  s = SUID set (means: runs as root even if non-root runs it)
```

`passwd` needs SUID to write to `/etc/shadow` (which only root can write).

```bash
chmod u+s executable     # add SUID
chmod 4755 executable    # set SUID numerically
```

> **Security:** SUID binaries owned by root are powerful attack vectors. Audit them:
> ```bash
> find / -perm -4000 -type f 2>/dev/null
> ```

### SGID (Set Group ID) — Bit `2`

On executables: runs with the file's group permissions.
On **directories**: files created inside inherit the directory's group (useful for shared directories).

```bash
chmod g+s shared_directory/   # SGID on directory
chmod 2775 shared_directory/  # SGID numerically

ls -la
# drwxrwsr-x  shared_directory  (s in group execute position = SGID)
```

### Sticky Bit — Bit `1`

On directories: **only the owner of a file can delete it**, even if others have write permission. Classic use: `/tmp`.

```bash
ls -la /
# drwxrwxrwt  ... tmp   (t = sticky bit set)

chmod +t /shared/dir/    # add sticky bit
chmod 1777 /shared/dir/  # sticky + world-writable (like /tmp)
```

### Numeric Summary

```
4000 = SUID
2000 = SGID
1000 = Sticky bit

4755 = SUID + rwxr-xr-x
2775 = SGID + rwxrwxr-x
1777 = Sticky + rwxrwxrwx (/tmp)
```

---

## 6.9 `umask` — Default Permissions

When a file is created, its permissions are determined by the `umask` (bits to **remove** from defaults).

Default permissions:
- Files: `666` (rw-rw-rw-)
- Directories: `777` (rwxrwxrwx)

Common umask `022`:
- Files: `666 - 022 = 644` (rw-r--r--)
- Directories: `777 - 022 = 755` (rwxr-xr-x)

```bash
umask                 # show current umask
umask 022             # set umask (only for current session)
umask 027             # more restrictive (group gets no write, others get nothing)

# Set permanently in ~/.bashrc:
echo "umask 022" >> ~/.bashrc
```

---

## 6.10 `ACL` — Access Control Lists (Advanced)

Standard Unix permissions allow one owner and one group. ACLs let you set permissions for specific users or groups beyond the standard model.

```bash
# Install ACL tools
sudo apt install acl

# View ACL
getfacl file.txt

# Give user 'jane' read access to a file
setfacl -m u:jane:r file.txt

# Give group 'devs' write access to a directory
setfacl -m g:devs:rw directory/
setfacl -Rm g:devs:rw directory/   # recursive

# Remove ACL entry
setfacl -x u:jane file.txt

# Remove all ACLs
setfacl -b file.txt
```

---

## Summary

```
Permission string: -rwxr-xr-x
                    │││├┤├┤├┤
                    │││ │ │ └── Other: r-x
                    │││ │ └──── Group: r-x
                    │││ └────── Owner: rwx
                    ││└──────── File type: regular file
                    │└───────── (unused, 10th char)
                    └────────── (1st char = type)

Numeric:   r=4, w=2, x=1
644 = rw-r--r--  (default file)
755 = rwxr-xr-x  (default dir/exec)
600 = rw-------  (private file)
```

---

## Knowledge Check

1. What does the permission `644` mean in human-readable form?
2. What command makes a script executable?
3. How do you recursively change ownership of a directory?
4. What is the difference between SUID on a file vs SGID on a directory?
5. What does the sticky bit do on `/tmp`?
6. What umask produces files with `640` permissions?

---

## Hands-On Exercise

```bash
# 1. Create files and practice chmod
mkdir -p ~/perm-practice
touch ~/perm-practice/{secret.key,public.html,deploy.sh,config.env}

ls -la ~/perm-practice/

# 2. Set appropriate permissions
chmod 600 ~/perm-practice/secret.key      # private key
chmod 644 ~/perm-practice/public.html     # public file
chmod 755 ~/perm-practice/deploy.sh       # executable script
chmod 640 ~/perm-practice/config.env      # readable by group only

ls -la ~/perm-practice/

# 3. Verify you can execute the script
echo '#!/bin/bash\necho "Deploy script ran!"' > ~/perm-practice/deploy.sh
chmod 755 ~/perm-practice/deploy.sh
~/perm-practice/deploy.sh

# 4. Check your own user and groups
id
groups

# 5. Find all world-writable files in /tmp (should be fine)
find /tmp -perm -o+w -type f 2>/dev/null | head

# 6. Find all SUID binaries (security awareness)
sudo find / -perm -4000 -type f 2>/dev/null

# 7. Test umask
umask 022
touch /tmp/test_umask_022.txt
ls -la /tmp/test_umask_022.txt
# Should be -rw-r--r-- (644)

umask 077
touch /tmp/test_umask_077.txt
ls -la /tmp/test_umask_077.txt
# Should be -rw------- (600)

# Reset umask
umask 022

# 8. Clean up
rm -rf ~/perm-practice/
```

**Challenge:** Create a shared directory where users in the same group can create and read each other's files, but only the owner of each file can delete it. Hint: SGID + Sticky bit.

---

## Further Reading

- `man chmod`, `man chown`, `man umask`
- `man 5 passwd` — passwd file format
- [Linux File Permissions Explained](https://www.redhat.com/sysadmin/linux-file-permissions-explained)

---

---


<div style="display:flex;justify-content:space-between;align-items:center;">
  <a href="05-text-processing.md">← Previous: Text Processing</a>
  <a href="./00-index.md">🏠 Index</a>
  <a href="07-process-management.md">Next: Process Management →</a>
</div>
