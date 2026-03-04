# Setting up SSH key for tunneling

This document shows how to generate a ssh key pair, load the key securely to the destination and connect with it successfully

--------

## 1. Generate key

First we need to generate a key pair. A public key to be sent to the destination and private key to keep to self securely. The system sings request with private key which destination can validate with public key

Generation command for key is:

```bash
ssh-keygen -t KEY_FORMAT -C COMMENT
```

> **Note:**  This will ask for a pass phrase and a key file location. It is better to set it in .ssh and the ssh by default looks for keys in this directory only

Common example of the command is

```bash
ssh-keygen -t ed25519 -C "KEY PURPOSE"
```

The command will ask for a passphrase. Keep this different from your other credentials in order to keep systems secure

## 2. Payload public key to host

We need to send the key file to the destination. Which will be done via SSH with password. This is the last time we will use destination password

```bash
# By default ssh loads all keys in .ssh if not specified
ssh-copy-id [ -i PRIVATE_KEY_LOCATION ] USERNAME@DEST_IP
```

## 3. Check connection with key

Now you can test by doing usual SSH login

```bash
ssh USERNAME@HOST # This will now ask passphrase instead
```
