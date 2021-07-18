---
title: "Route PiVPN client traffic via Mullvad"
author: ArcherN9
date: 2020-06-28 14:23:21 +0000
description: "Setup a Raspberry Pi as a central VPN hub to route PiVPN traffic via Mullvad"
category: [Linux]
tags: [Mullvad, PiVPN, Wireguard, IPTables, Routing, Networking, RaspberryPi, Linux]
---

*Note: As of July 2021, this post is still accurate. However, considering
iptables has been deprecated and replacd with nftables (I think?), I plan to
re-write this post in the near future. In case it doesn't, check the About Me
section to get in touch with me.*

This post attempts to respond to enigmas similar to "How do I route my PiVPN
traffic through a commercial VPN?". By the end of this guide, you should have:

* A working PiVPN installation; (With Wireguard)
* A Mullvad VPN (With Wireguard) setup for use on multiple devices - Beyond the
* five client limit imposed by their system

I have used a fresh installation of DietPi on a Raspberry Pi Zero W (henceforth
referred to as DietPi) for this guide. The setup process is similar for all other
RaspberryPis and Raspberry Pi OS.

*Note: This guide assumes you're aware of PiVPN & its uses, have it installed on
the system and it functions as intended. For more information, I suggest going
through threads on Reddit and the official website first.*

## Step One: Verify ipv4 packet forwarding is enabled

IP forwarding is synonymous with routing and seldom referred to as 'kernel IP
forwarding' because it is a feature of the Linux kernel. A router has multiple
network interfaces; If traffic comes in from one interface that matches a subnet
of another network interface, a router then forwards that traffic to the other
network interface. This functionality is achieved by enabling ipv4 forwarding on
non-router devices. Refer [this][1] for more information on this property.

**Pro Tip**: Updating the value of `net.ipv4.ip_forward` in `/proc/sys/net/ipv4`
is a temporary gambit. I suggest using `sysctl` instead.

Check for the current ipv4 forwarding value; PiVPN enables it during installation;
In my experience, it does not sustain system reboots.

```sh
$ sudo sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 1
$ sudo sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1
```

## Step Two: Creation of Mullvad Wireguard interfaces

To create Mullvad Wireguard interfaces, we need to download the Mullvad Wireguard 
configuration file. Refer [this guide][2] but follow the steps below:

```sh
# This adds the Wireguard PPA repository to the system so Wireguard interfaces may be downloaded from the repository.
$ sudo add-apt-repository ppa:wireguard/wireguard

# This installs some dependencies required to execute the Mullvad shell script that we'll use later
$ sudo apt-get install curl jq resolvconf raspberrypi-kernel-headers wireguard-dkms wireguard-tools

# This step is a stow-away suggestion and archival for the script
$ cd /etc/wireguard && mkdir mullvad-config && cd mullvad-config

# Downloads the Mullvad Wireguard interface creation shell script. We shall modify this to our use case
$ curl -LO https://mullvad.net/media/files/mullvad-wg.sh

# Use an editor of your choice to include PostUp and PreDown sections to the script
$ nano mullvad-wg.sh # Update Jul 2021: In hindsight, I wonder why did I use nano and not Neovim.
```

I suggest you read through the mullvad-wg.sh shell script to better understand
the purpose. In the editor, search for the line that reads `DNS="193.138.218.74"`
and replace it with `DNS="x.x.x.x"` where `x.x.x.x` is your DietPi's LAN IP. In
my case, I set it up as `DNS="192.168.1.100"`

The **for** loop that succeeds *"Writing WireGuard configuration files"* is
responsible for creating individual Wireguard configuration files to connect to
Mullvad. Refer [this video][3] for more information on **iptables** & modify the
section:

```sh
...
for CODE in "${SERVER_CODES[@]}"; do
        CONFIGURATION_FILE="/etc/wireguard/mullvad-$CODE.conf"
        umask 077
        mkdir -p /etc/wireguard/
        rm -f "$CONFIGURATION_FILE.tmp"
        cat > "$CONFIGURATION_FILE.tmp" <<-_EOF
                [Interface]
                PrivateKey = $PRIVATE_KEY
                Address = $ADDRESS
                DNS = $DNS

                # This section is executed when the wireguard interface is starting

                # Creates a new entry in the NAT table | For all packets that traverse through the out-interface mullvad-$CODE, MASQUERADE the packets with the PI's IP address
                PostUp = iptables --table nat --append POSTROUTING --out-interface mullvad-$CODE --source 0.0.0.0/0 --destination 0.0.0.0/0 -j MASQUERADE
                
                # Add a default route via the gateway on wlan0 interface for a routing table pivpn | All packets against the routing table pivpn will be routed through the defaul gateway
                PostUp = ip route add default via 192.168.1.1 dev wlan0 table pivpn
                
                # All packets with FwMark 51820 to be routed against table pivpn | This is an important step because Mullvad Wireguard configuration disallows any packets without a fwmark 51820 to be routed.
                PostUp = ip rule add fwmark 51820 table pivpn
                
                # OPTIONAL : If you need any ports open only from the Mullvad interface but not on wlan0, open a random port for wireguard on the Mullvad website and add it here
                PostUp = iptables --table filter -A INPUT --in-interface mullvad-$CODE -p udp --dport 2836 -j ACCEPT

                # This section is executed when the wireguard interface is shutting down
                
                # All PreDown steps are inverse of PostUp statements so as to logically close the temporary setup which lives only while a Mullvad interface is connected

                PreDown = iptables --table nat -D POSTROUTING --out-interface mullvad-$CODE --source 0.0.0.0/0 --destination 0.0.0.0/0 -j MASQUERADE
                PreDown = ip route delete default via 192.168.1.1 dev wlan0 table pivpn
                PreDown = ip rule delete fwmark 51820 table pivpn
                PreDown = iptables --table filter -D INPUT --in-interface mullvad-$CODE -p udp --dport 2836 -j ACCEPT

                [Peer]
                PublicKey = ${SERVER_PUBLIC_KEYS["$CODE"]}
                Endpoint = ${SERVER_ENDPOINTS["$CODE"]}
                AllowedIPs = 0.0.0.0/0, ::/0
        _EOF
        mv "$CONFIGURATION_FILE.tmp" "$CONFIGURATION_FILE"
done
...
```
Note: Ports from Mullvad may be opened
[here](https://mullvad.net/en/account/#/ports). Save the script, exit and
execute.

```sh
$ chmod +x mullvad-wg.sh
$ ./mullvad-wg.sh
```

If you receive *"Please wait up to 60 seconds for your public key to be added to
the servers."* as the final output, it would imply your wireguard configurations
have been generated. Execute `ls -lsa` in `/etc/wireguard` directory to confirm.

## Step Three: Creation of pivpn table

To satisfy the parameters added to `PostUp` section in Mullvad configurations
and for them to function correctly, we shall create the pivpn routing table.

```sh
$ cd /etc/iproute2
$ nano rt_tables
# reserved values
#
255     local
254     main
253     default
0       unspec
#
# local
#
#1      inr.ruhep

# Add a new entry at the end of this file. The ID of the table should be unique and can be any numeric value.
100     pivpn
```

## Step Four: Verifying newly created Mullvad interfaces

Navigate to `/etc/wireguard` and verify if a random Mullvad interface functions
as intended.

```sh
$ cd /etc/wireguard
$ wg-quick up mullvad-ch4.conf

[#] ip link add mullvad-ch4 type wireguard
[#] wg setconf mullvad-ch4 /dev/fd/63
[#] ip -4 address add 10.67.235.185/32 dev mullvad-ch4
[#] ip -6 address add fc00:bbbb:bbbb:bb01::4:ebb8/128 dev mullvad-ch4
[#] ip link set mtu 1420 up dev mullvad-ch4
[#] resolvconf -a tun.mullvad-ch4 -m 0 -x
[#] wg set mullvad-ch4 fwmark 51820
[#] ip -6 route add ::/0 dev mullvad-ch4 table 51820
[#] ip -6 rule add not fwmark 51820 table 51820
[#] ip -6 rule add table main suppress_prefixlength 0
[#] ip6tables-restore -n
[#] ip -4 route add 0.0.0.0/0 dev mullvad-ch4 table 51820
[#] ip -4 rule add not fwmark 51820 table 51820
[#] ip -4 rule add table main suppress_prefixlength 0
[#] sysctl -q net.ipv4.conf.all.src_valid_mark=1
[#] iptables-restore -n
[#] iptables --table nat --append POSTROUTING --out-interface mullvad-ch4 --source 0.0.0.0/0 --destination 0.0.0.0/0 -j MASQUERADE
[#] ip route add default via 192.168.1.1 dev wlan0 table pivpn
[#] ip rule add fwmark 51820 table pivpn
[#] iptables --table filter -A INPUT --in-interface mullvad-ch4 -p udp --dport 2836 -j ACCEPT

```

If you see an output similar to the above, you have successfully connected to
Mullvad. Just to be sure, test the conection.

```sh
$ ip a
...
6: mullvad-ch4: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
    link/none 
    inet 10.67.235.185/32 scope global mullvad-ch4
       valid_lft forever preferred_lft forever
...

$ curl https://am.i.mullvad.net/connected
You are connected to Mullvad (server ch4-wireguard). Your IP address is 45.12.222.236

# Close the Mullvad interface
$ wg-quick down mullvad-ch4.conf
$ wg-quick down wg0
```

## Step Five: Add a FwMark to packets incoming from PiVPN

```sh
$ cd /etc/wireguard
$ nano wg0.conf

...
Address = 10.6.0.1/24
ListenPort = 51820
FwMark = 51820

### begin test-1 ###
...
```

Bump both of the interfaces.

```sh
$ cd /etc/wireguard
# Enable PiVPN
$ wg-quick up wg0

# Pick a random mullvad interface to test the connection
$ wg-quick up mullvad-dk2.conf 
```

## Step Six: Verify functionality on a client device

If the PiVPN client has a terminal, access it and use `curl` to verify:

```sh
$ curl https://am.i.mullvad.net/connected
You are connected to Mullvad (server dk2-wireguard). Your IP address is 45.129.56.201
```

Or open a browser, navigate to `https://duckduckgo.com/` and search for "ip
address".

![IP Address confirmation](/assets/img/mullvad_ipaddress.png)

[1]: https://unix.stackexchange.com/questions/14056/what-is-kernel-ip-forwarding
[2]: https://mullvad.net/en/help/wireguard-and-mullvad-vpn/
[3]: https://www.youtube.com/watch?v=kQYQ_3ayz8w
