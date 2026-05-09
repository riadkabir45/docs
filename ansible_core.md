# Basics of Using Ansible Core

Lets learn how to make things faster with ansible.

## 1. Test out ansible connection

Lets setup a basic connection test configuration so that we know our ansible connection setup

### 1.1 The command way

We can do a ping to all node servers from ansible with folloing commands

```bash
ansible all --key-file ~/.ssh/id_ed25519 -i .inventories -m ping
```

Where the `.inventories` file contain the ip addresses or name of the node servers seperated by new line. Also the key file to connection needs to be passphareless initially.

### 1.2 The configuration way

We can also setup an ansible configuratin file in home directory called `ansible.cfg`. There is another file in `/etc/ansible` but the home directory configuration takes precedence.

**File:** `~/ansible.cfg`
```conf
[defaults]
inventory = ~/.inventories
private_key_file = ~/.ssh/id_ed25519
```

With this we can shorthand the entire command to

```bash
ansible all -m ping
```

> **Note:** We can also use `ansible all -m gather_facts --limit IP` or `ansible all -m gather_facts` (which gathers for all nodes) to test our connections and their states.

