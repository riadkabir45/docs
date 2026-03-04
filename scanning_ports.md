# Scanning open ports without triggering security alerts

There are lots of hacky port scanners like nmap. But these are often messy and sends massive packages to test check if a port is open or not

Here are few alternatives that are safer to use without triggering IDS/IPS

## 1. Netcat

Netcat is the Swiss Army Knife of networking. We can do many things with it. Including sending a small connecttion request rather than full scan

```bash
nc -zv -w 2 HOST PORT
```

- `-z`: Scan mode (Do not send packets)

- `-W TIMEOUT`: Wait for connection for the given time

## 2. The usual bash

Most system won't except someone to use plain bash to hack in. So this won't trigger any alerts

```bash
# This command will pipe error on failing to open port
timeout 2 bash -c "</dev/tcp/192.168.2.129/22"
```

## 3. Telnet

This is an old school command line tcp connection maintainer. Because it keeps connections on. Most security system do no expect telnet to be used for attacks.

```bash
timeout 2 bash -c "</dev/tcp/192.168.2.129/22"
```
