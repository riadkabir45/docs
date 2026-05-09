# Task - BIND and DHCP  Setup

This project is to showcase the skills to setup complex dns setup and dhcp server setup. The project will have following structures,

- There will be three vmware/servers
- One vmware will act as both dns and dhcp server
- Other two server will act as clients
- One will have static IP and one will have reserved IP

## 2. Package Installations

For this task we just need 2 items mainly. These are `bind` or **Berkeley Internet Name Domain** and `dhcpd`. We will also need `bind-utils` to test out these configurations.

```bash

dnf install bind bind-utils -y
```

## 3. Bind Configuration

In order to use Bind dns server, we need to make 3 component configuration. We need to tell bind what IP to listen for, what are the zones and what are addresses(the phone book).

### 3.1 Listening port and IP subnet

Lets first setup our listening configurations which is at **File:** `/etc/named.conf`

```conf
options {
    listen-on port 53 {
        127.0.0.1; 
        192.168.45.10;
        };
    directory "/var/named";
    allow-query { 
        192.168.45.0/24;
        };
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };
};
```

This will make the Bind listen on port 53 at IP 192.168.45.10, allow DNS queries from 192.168.45.0/24 subnet, look for phone books or DNS books at `/var/named` and forward unknown IP to next DNS server `8.8.8.8` or `8.8.4.4`.

### 3.2 Zone setup

Now we setup our zones. Zones allow us to group together. So its easier to apply rules on multple IP addresses. We create a zone named `lab.riad` at **File:** `/etc/named.rfc1912.zones` or we could do it in our `/etc/named.conf`

```conf
zone "lab.riad" IN {
    type master;
    file "lab.riad.db";
    allow-update { 
        none;
    };
};
```

This tells to make the zone type master. Meaning this device is the owner of the zone and it won't be dependent on another server for record updates. And we will keep records in file named "lab.riad.db". Also no client in our zone is allowed to update the records in our book.

### 3.3 Setup Record Book

Now we setup our hosts and their IP addresses in the file `lab.riad.db` in the directory `/var/named`

```conf
$TTL 1D
@   IN SOA  ns1.lab.riad. admin.lab.riad. (
                2026042301      ; serial
                1D              ; refresh
                1H              ; retry
                1W              ; expire
                3H )            ; minimum

@       IN NS   ns1.lab.riad.
ns1     IN A    192.168.45.10
host01  IN A 192.168.45.11
host02  IN A 192.168.45.51
```

Here we stictly follow the old bind syntax, pointing our dns server name `ns1.lab.riad` and a placeholder fake email `admin.lab.riad`. We also pass some DNS refresh parameters. And then we provide the acutal book reord.

> **Note:** Bind follows a very old and strict syntax. So make sure you follow things strickty like not using double quote instead of single quote.

## 4. DHCP Configuration

Now we setup DHCP which is the easy part. We just edit the **File:** `/etc/dhcp/dhcpd.conf`

```conf
option domain-name "lab.riad";
option domain-name-servers 192.168.45.10;

default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet 192.168.45.0 netmask 255.255.255.0 {
    range 192.168.45.50 192.168.45.100;
    option routers 192.168.45.1;
    option broadcast-address 192.168.45.255;
}
```

## 5. Verification Of Configuration

Its best practice to allways verify each configuration file speerately.

|Task|Command|
|-|-|
|Check DNS Syntax|`named-checkconf /etc/named.conf`|
|Check Zone Syntax|`named-checkzone lab.riad /var/named/lab.riad.db`|
|Check DHCP Syntax|`dhcpd -t -cf /etc/dhcp/dhcpd.conf`|

## 6. Boot Up System

Now we enable both Bind and DHCP and their firewwall  passthrought

```bash
sudo firewall-cmd --permanent --add-service={dns,dhcp}
sudo firewall-cmd --reload

systemctl enable bind --now
systemctl enable dhcpd --now
```

And now if we boot up our client devices. They will auto pick dynamic IP addresses. We configure the `.51` to `.11` and now we sould be able to do `nslookup` or `dig` for any device.