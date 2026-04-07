# Better hostnamectl
hostnamectl does what its suppose to. But there are some caviots,

- It does not have a universal syntax
- It does not update the /etc/hosts. Which is required to keep system in sync

## Basic hostname structure
A hostname in system actually is stored in 2 instances

- `/etc/hostname`: This is where the system grabs the hostname from
- `/etc/hosts`: THis is used by network to resolve hostname

One can update these file manually but a lot of services require a sync trigger to know hostname is updated. So instead of updaing the /etc/hostname manually, it is better to call `hostnamectl`

## The Ultimate script
This script does what a person need to change the hostname systemwide. Of course you still need to restart applications to update them.

```bash
#!/bin/env bash

set -e

if hostnamectl --help | grep -q 'set-hostname';then
    SUBCMD=set-hostname
else
    SUBCMD=hostname
fi

echo "Old hostname is $HOSTNAME"
read -p "Enter a new hostname: " NEWHOST

sudo hostnamectl $SUBCMD $NEWHOST
sudo sed -i "/^127\.0\.1\.1/ s/\b$HOSTNAME\b/$NEWHOST/g" /etc/hosts

echo "HOSTNAME changed to $NEWHOST successfully"
```

This will robuslty handle updaing hostname. ADIOS
