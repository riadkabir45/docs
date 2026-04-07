# Installing VMware on Ubuntu with Secure Boot
This document outlines the professional setup of VMware Workstation/Player on an Ubuntu system, specifically addressing the kernel module signing required for **Secure Boot** compatibility.

## 1. System Preparation
Before installing VMware, you must install the build tools and kernel headers required to compile the virtual machine monitors.

```bash
sudo apt update
sudo apt install build-essential linux-headers-$(uname -r)
```

## 2. Installation Process

1. Download the VMware `.bundle` installer from the official VMware website.
2. Navigate to your `Downloads` folder in the terminal.
3. Grant execution permissions and run the installer

```bash
chmod +x VMware-Workstation-Full-*.bundle
sudo ./VMware-Workstation-Full-*.bundle
```

## 3. Secure Boot & Kernel Module Signing
On systems with Secure Boot enabled, the Linux kernel will refuse to load unsigned VMware modules (`vmmon` and `vmnet`). Follow these steps to create a trusted key and sign the modules.

### Step 3.1: Generate the Machine Owner Key (MOK)
Create a self-signed certificate and private key:

```bash
openssl req -new -x509 -newkey rsa:2048 -keyout VMWARE1.priv -outform DER -out VMWARE1.der -nodes -days 36500 -subj "/CN=VMware/"
```

### Step 3.2: Sign the VMware Modules
Use the kernel's signing utility to apply your new key to the VMware drivers:

```bash
sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 ./VMWARE1.priv ./VMWARE1.der $(modinfo -n vmmon)
sudo /usr/src/linux-headers-$(uname -r)/scripts/sign-file sha256 ./VMWARE1.priv ./VMWARE1.der $(modinfo -n vmnet)
```

### Step 3.3: Import the Certificate
Inform the system that it should trust your new certificate:

```bash
sudo mokutil --import VMWARE1.der
```

> **Note:** Enter a temporary password when prompted. You will need it during the reboot.

## 4. Enrolling the Key in BIOS

1. **Reboot** your machine.

2. Upon restart, the **MOK Management** blue screen will appear.

3. Select **Enroll MOK** > **Continue** > **Yes**.

4. Enter the password from Step 3.3.

5. Select **Reboot**.


## 5. Verification

Once Ubuntu boots, verify that the modules are loaded:

```bash
lsmod | grep vm
```

## 6. VM tweaks

These are not essential but can help extend power of VMware

### 6.1 Port forwarding with nginx
First we need to install nginx and also its stream module if separately packaged. After that, we need to add to configuration like below

**File:** `/etc/nginx/nginx.conf`

```bash
# Load nginx stream module before event block (Duplicate loads cause crash)
load_module /usr/lib/nginx/modules/ngx_stream_module.so;

# Create a stream block to handle and pass ssh connections
stream {
    server {
        listen 4545;
        proxy_pass VM_IP:22;
    }
}
```
