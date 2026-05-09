# SELinux Lab Report: Impact of Context on Apache2 (httpd)

## 1. Initial State: SELinux Disabled
**Goal:** Establish a baseline for Apache functionality without Mandatory Access Control.

* **Action:** Modified `/etc/selinux/config` to `SELINUX=disabled` and rebooted.
* **Setup:**
    * Installed `httpd`.
    * Created a custom directory: `/opt/www/html`.
    * Created a test file: `/opt/www/html/index.html`.
    * Configured Apache `DocumentRoot` to point to `/opt/www/html`.
* **Result:** `curl localhost` returns the index page successfully. 
* **Observation:** Without SELinux, standard Linux DAC (Discretionary Access Control) allows the `apache` user to read `/opt/` because permissions were sufficient.

---

## 2. Enabling SELinux (The Unexpected Success)
**Goal:** Observe how SELinux reacts to the custom `/opt` path.

* **Action:** Set SELinux to `Enforcing` and rebooted.
* **Result:** `curl localhost` **STILL WORKS.**
* **Investigation:**
    * Ran `sesearch -A -s httpd_t -t usr_t` (or similar).
    * **Finding:** Many default SELinux policies (like the `targeted` policy) include rules that allow `httpd_t` (the Apache process) to read `usr_t` or generic types often found in `/opt`.
    * Because the system relabeled `/opt` during the transition to Enforcing, it likely assigned a "safe" generic type that Apache was already permitted to read.

---

## 3. The Failure Case: Label Mismatch via File Migration
**Goal:** Trigger a real-world SELinux "Permission Denied" scenario using a new path and the "Move" command.



### The Setup
1.  Created a new root-level directory: `/html`.
2.  Created `index.html` in the user's **home directory** (`/root/` or `/home/user/`).
    * *Initial Context:* `admin_home_t` or `user_home_t`.
3.  **The Critical Action:** Used `mv` to move `index.html` from home to `/html`.
    * *Effect:* Because `mv` was used, the file **retained** its home directory context (`user_home_t`).
4.  Configured Apache to point to `/html`.

### The Result
* **Action:** `curl localhost`.
* **Outcome:** `403 Forbidden`.
* **Log Analysis:** Checked `/var/log/audit/audit.log`.
* **Finding:** ```text
    type=AVC msg=audit(...): avc: denied { read } for pid=... comm="httpd" 
    name="index.html" dev="..." ino=... 
    scontext=system_u:system_r:httpd_t:s0 
    tcontext=unconfined_u:object_r:admin_home_t:s0 
    tclass=file permissive=0
    ```
* **Conclusion:** SELinux blocked the access because the process `httpd_t` is strictly forbidden from reading `admin_home_t` (home directory files) for security reasons. Apache "fell back" or simply failed to serve the content due to this denial.

---

## 4. Key Takeaways

| Scenario | Path | Result | Reason |
| :--- | :--- | :--- | :--- |
| **Disabled** | `/opt/www/html` | Success | No SELinux enforcement. |
| **Enforcing (New)** | `/opt/www/html` | Success | Policy allowed the generic type assigned to `/opt`. |
| **Enforcing (Move)** | `/html` | **Failure** | File kept the `home_t` label; `httpd_t` cannot access home folders. |

### How to Fix the Failure
To make the `/html` directory work, the context must be updated to a type Apache recognizes:
```bash
# Set the permanent policy
semanage fcontext -a -t httpd_sys_content_t "/html(/.*)?"
# Apply the label to the filesystem
restorecon -Rv /html