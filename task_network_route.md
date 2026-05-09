# Task - SELinux Context Implementation

The goal of this task is to analyze how SELinux Mandatory Access Control (MAC) handles different directory structures and file operations. We will demonstrate three specific behaviors: why some custom paths work by default, how moving files breaks access, and how unknown root paths default to a restrictive state.

- Observe how `/opt` paths can inherit allowed generic types
- Trigger an AVC denial via Context Persistence (moving from `/home`)
- Trigger an AVC denial via Fallback Labeling (creating in `/html`)

## 1. Case 1: Custom Path with Generic Allowance (/opt)

In this scenario, we create a web root in `/opt`. Even with SELinux in `Enforcing` mode, this often works because of the default "Targeted" policy rules.

* Create `/opt/www/html` and configure Apache.
* Check the context assigned to the new path.

```bash
mkdir -p /opt/www/html
echo "Welcome to httpd by riad" > /opt/www/html/index.html
ls -Z /opt/www/html/index.html
```

**Result:** `curl localhost` works. 
**Reason:** Files created in `/opt` are often labeled as `usr_t` or `etc_t`. The SELinux policy for `httpd_t` (Apache) usually contains an allowance to read these generic system types, meaning no extra configuration was needed.

---

## 2. Case 2: Context Persistence and the Move Trap (/var/www/html)

Next, we demonstrate how moving a file preserves its original security context, leading to a denial even in a "valid" directory.

* Create a file in the user's **home directory**.
* Move the file into the standard Apache directory `/var/www/html`.

```bash
# Create file in home (gets user_home_t context)
echo "hello by riad" > ~/index.html

# MOVE the file (Preserves context)
mv ~/index.html /var/www/html/index.html

# Verify the mismatch
ls -Z /var/www/html/index.html
```

**Result:** `curl localhost` returns **403 Forbidden**. 
**Reason:** The file retained the `user_home_t` label. SELinux sees a web server attempting to read private user data and blocks the process, regardless of standard Linux permissions.

---

## 3. Case 3: Fallback Labeling on New Root Paths (/html)

Finally, we observe what happens when we create a directory that the SELinux policy database does not recognize at all.

* Create a new directory at the root level: `/html`.
* Create an index file and point Apache to it.

```bash
mkdir /html
touch /html/index.html
ls -Z /html/index.html
```

**Result:** `curl localhost` returns the original apache2 page. 
**Reason:** Because even `/html` is not a standard path defined in the SELinux policy, the kernel assigns it the `default_t` label. The `httpd_t` domain has no permission to interact with `default_t`, causing the system to "fall back" to the default page.

---

## 4. Troubleshooting and Audit Logs

We verify these denials by inspecting the Access Vector Cache (AVC) logs.

```bash
# Search for recent denials
ausearch -m AVC
```

```
time->Wed May  6 10:28:50 2026
type=PROCTITLE msg=audit(1778041730.798:340): proctitle=2F7573722F7362696E2F6874747064002D44464F524547524F554E44
type=SYSCALL msg=audit(1778041730.798:340): arch=c000003e syscall=6 success=no exit=-13 a0=7fe5100436f0 a1=7fe5166d57b0 a2=7fe5166d57b0 a3=1 items=0 ppid=3956 pid=3959 auid=4294967295 uid=48 gid=48 euid=48 suid=48 fsuid=48 egid=48 sgid=48 fsgid=48 tty=(none) ses=4294967295 comm="httpd" exe="/usr/sbin/httpd" subj=system_u:system_r:httpd_t:s0 key=(null)
type=AVC msg=audit(1778041730.798:340): avc:  denied  { getattr } for  pid=3959 comm="httpd" path="/html/index.html" dev="dm-0" ino=51219004 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=0
```
