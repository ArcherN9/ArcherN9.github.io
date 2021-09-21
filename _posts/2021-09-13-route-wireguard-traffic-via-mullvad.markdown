---
title: "Route Wireguard traffic via Mullvad"
author: ArcherN9
date: 2021-09-13 22:00:13 +0000
description: "Setup a private VPN server using Wireguard and route its traffic via Mullvad"
category: [Linux]
tags: [Mullvad, Wireguard, IPTables, NFTables, Routing, Networking, RaspberryPi, Linux, DietPi]
---

This post is a follow up to my previous article [here][1]
(hereafter referred to as linked post) about routing PiVPN traffic through
[Mullvad VPN][2]. Please refer to the *Note* added as a prologue on the linked
post.

The previous article relied on *iptables* which stands deprecated now and
*nftables* is supposedly to replace it. Refer the official *nftables* website
[here.][3] This guide attempts to re-do the entire configuration done on the
linked post. If using *iptables* or *iptables-legacy* is preferred, please use
the linked post instead.

### Problem Statement

This post attempts to respond to & perhaps provide a solution to the question,
"How do I route my PiVPN/Wireguard traffic via a commercial VPN?".

One would do such a thing in perhaps the following situations:

* A lot of people from the Linux community run [PiHole][4] on their local networks.
In order for them to use the custom domain blocker on devices not currently connected
to the local network, they would generally install [PiVPN][5] or [Wireguard][6].
Now, if the hardware that hosts PiHole is further connected to Mullvad VPN or a
different Wireguard interface, users would expect their remotely-connected device
traffic to be routed via the VPN as well.
* VPN providers generally impose a 5 active client limitation (Mullvad instead
limits how many Wireguard keys can be generated). Ergo, connecting more devices
is simply, not possible. Not directly anyway. By accumulating traffic on a single
private wireguard interface & flushing that through a single wireguard interface
registered with Mullvad, one can connect as many devices as limited by the Wireguard
protocol itself.

For me, it was bullet #1 and also to affirm myself that I could do such at thing.

### Environment Details

* A fresh headless [Diet Pi][7] v7.4.2 build ID
\#140fa04 (dated: 13th September 2021) for ARM-v6 installation
* RaspberryPi Zero W (henceforth referred to as Pi)
* `CONFIG_ENABLE_IPV6` is set to `1`; My router is an IPv6 only connection.
* DietPi Linux kernel: 5.10.52+
* Wireguard Version:
```sh
$ sudo mkdir -p /etc/wireguard/keys
$ wg --version
wireguard-tools v1.0.20210223 - https://git.zx2c4.com/wireguard-tools/
```
* **Mullvad API version??**

// Add an architecture diagram here.

*Note: This guide assumes that you have a working installation and setting up
DietPi is beyond the scope of this guide. If you are setting up a fresh DietPi
installation as well, I recommend using [this][8] guide to configure a headless
system.*

## Step One: Dependency Installation

SSH into your Pi and execute:

```sh
$ sudo apt install wireguard nftables git qrencode curl jq openresolv
# apt should install a few packages automatically. If not, install them
# manually: libedit2 libjansson4 libnftables1 libnftnl11
```

## Step Two: Configure a Wireguard server on the Pi

With the tools installed, confirm if Wireguard was installed correctly.

```sh
$ wg
# In a new wireguard installation, there should not be any output of the
# executed command.
# If you instead get an output like below, it may require some debugging.
# zsh: command not found: wg
```

*Note: Additionally, I took the liberty to setp NeoVim editor using my [DotFiles][9].
It does not directly impact the end result yielded from this guide. By no means
am I an expert with Wireguard. I followed an excellent guide myself, [here][10]*

Create a private, and public key for the server.

```sh
# Navigate to Keys directory 
$ mkdir -p /etc/wireguard/keys && cd /etc/wireguard/keys
# From the help page: wg genkey Generates a new private key and writes it to stdout
$ wg genkey > server_private_key
# You would probably receieve an error like so:
Warning: writing to world accessible file.
Consider setting the umask to 077 and trying again.
# You could resolve this issue by executing: $ umask 077 before executing the
# genkey command or executing: chmod 700 * after executing both genkkey and pubkey
# commands.

# From the help page: wg pubkey Reads a private key from stdin and writes a public
# key to stdout
$ wg pubkey < server_private_key > server_public_key

# An alternate all-in-one approach is to execute the below but I prefer segmenting
# my commands. I guess I'm just not that advanced a user yet. PICK THE ABOVE OR BELOW.
# NOT BOTH.
$ wg genkey | tee privatekey | wg pubkey > publickey
```

Navigate to the root wireguard directory at `/etc/wireguard` and create a configuration
file titled `wg0.conf`

```markdown
[Interface]
# The Private key is from the server_private_key file created above
# The key below is just a placeholder. Use your own.
PrivateKey = gK48mAIwIF0sMYXPWWkErZ2+Ofh/p7tv6UyhljiHX1g=

# This is the server address on the wireguard interface using ipv4 and ipv6
Address = 10.6.0.1/24, fd00:43:43::1/64

# The port on which Wireguard listens for connections
ListenPort = 51820

# Multiple Peer blocks can be added depending on how many peers want to connect
# to this server. For now, we declare just one
[Peer]

# The public key of the wireguard client - Could be another desktop, Linux system
# or a handheld device. The process to create keys on the client largely remain
# the same. Figure out a way to import the public key generated on the client to
# this file. Also, the key below is just a placeholder. Use your own.
PublicKey = DL6qKIOuU2tG4KKqSlMdKwDkSaOv6HM8YSPAr3XHnk8=

# The allowed IPs section declares the IP address of the client. A client may
# connect to this interface using the declared wireguard public key only if
# they declare the following IP address on their interface.
AllowedIPs = 10.6.0.2/24, fd00:43:43::2/64
```
With the configuration setup, its time to enable this interface.

```sh
# I executed this inside the /etc/wireguard directory and as a root user. 
# I've had past experiences wherein this command cannot be executed as a non-root
# user; and ZSH autocomplete doesn't work when one uses SUDO.
$ wg-quick up wg0

# You should see a similar output.
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
Warning: AllowedIP has nonzero host part: 10.6.0.2/24
Warning: AllowedIP has nonzero host part: fd00:43:43::2/64
[#] ip -4 address add 10.6.0.1/24 dev wg0
[#] ip -6 address add fd00:43:43::1/64 dev wg0
[#] ip link set mtu 1420 up dev wg0
```

To verify proper functionality of the wireguard interface, we execute the `wg` command
once again:

```sh
$ wg
interface: wg0
  public key: ZGMDXHJBEEFn9pLK8wpeFeEf2bidHek/kdImW56L9Xs=
  private key: (hidden)
  listening port: 51820

peer: DL6qKIOuU2tG4KKqSlMdKwDkSaOv6HM8YSPAr3XHnk8=
  allowed ips: 10.6.0.0/24, fd00:43:43::/64
```
Note that there are no connected clients to this interface right now. For easier
client management, create a folder `clients` inside `/etc/wireguard` and create
multiple conf files for each client. An example of `ArchS9.conf` is below.
```sh
[Interface]
# The assigned IP address to the client on the wireguard interface
Address = 10.6.0.2/24, fd00:43:43::2/64

# I'm not entirely sure of this property; If I had to guess, this is the port
# of origin for client requests.
ListenPort = 51820

# The private key of the client | This can be generated on the server or the
# client
PrivateKey = APceeHUY519mhG8rjvQU5UafRfa8NYS1ySMqCJhmHE0=

# The DNS to be used on the client. It should correspond to the Wireguard Server
# if it also runs unbound, PiHole or equivalent; A local (Local to the server)
# IP address of a different device running a DNS resolver. This field will not
# accept any other value except the server's wireguard interface IP unless
# Routing rules have been defined. Since we have not, we keep the DNS value
# to the server's Wireguard interface - regardless of availability of a DNS
# Server on that IP or not.
DNS = 10.6.0.1, fd00:43:43::1

[Peer]

# Public Key of the sever; Import from the server_public_key file created earlier.
PublicKey = ZGMDXHJBEEFn9pLK8wpeFeEf2bidHek/kdImW56L9Xs=

# The client will only connect to a server against the declared Public key
# if the allotted IP address is within the subnet range identified below.
AllowedIPs = 10.6.0.0/24, fd00:43:43::1

# The server endpoint. For anyone who is running their Wireguard servers behind
# a dynamic IP address, creation of a domain is recommended. Ex: htps://noip.com
# using A or AAA records for Ipv4 or IPv6 resolution
# This also warrants creation of an IP forwarding rule on your NAT if there is one.
# The topic is beyond the scope of this guide.
Endpoint = [MY-IPV6-ADDRES-WAS-HERE]:51820
```
## Step Three: Configure the client device & Test connectivity

With a client configuration generated on the server, we need to import it to
a client device. I'm using my Samsung Galaxy S9 and the [Wireguard Android App][12]
to achieve this. The Wireguard app is capable of importing configurations via
a QR code as well. To make use of this feature, we need to go back to the Wireguard
server and generate a QR code.

```sh
$ cd /etc/wireguard/clients

# qrencode tool posts the QR code output in stdout. To have that in a file instead,
# attach: -o <filename>.png parameter.
$ qrencode -t ANSIUTF8 < ArchS9.conf
```

Depending on where the Wireguard server is being setup and whether access to
a desktop environment is available or not, we either print the QRCode on the
terminal itself or as a file to be accessed via something like [feh][13]. I'm
embarassed to admit this but since my setup was on a headless temporary PiZero with
a terminal that does not easily zoom in or out and the qrencode output is a behemoth
image, I found it very difficult to import the png file to my ArchLinux daily driver.
Quick fixes with `scp` failed & alterantives like creating an FTP server with
[proftpd][14] seemed overkill. I resorted to zooming out my terminal to fit the
qrcode within the display bounds and scanning it via the Wireguard Android app.
Not a very techy fix but it works.

With the configuration imported the Android app, use the switch to enable
interface and notice the `Rx` and `Tx` values dynamically change; They serve
as a visual feedback that the Android app was able to establish communication
with the Pi Wireguard server. For a more concrete affirmation:

```sh
# Execute the wg command to garner current Wireguard status
$ wg
interface: wg0
  public key: ZGMDXHJBEEFn9pLK8wpeFeEf2bidHek/kdImW56L9Xs=
  private key: (hidden)
  listening port: 51820

peer: DL6qKIOuU2tG4KKqSlMdKwDkSaOv6HM8YSPAr3XHnk8=
  endpoint: [MY-CLIENT-IPV6-ADDRESS]:51820
  allowed ips: 10.6.0.0/24, fd00:43:43::/64
  latest handshake: 4 seconds ago
  transfer: 28.72 KiB received, 14.25 KiB sent
```

## Step Four: Setup basic NetFilter Firewall to safeguard our server

For simplicity, we shall not attempt to interleave iptables and nftable configurations.
This is rather an optional step but highly recommended. By seting up a base
Netfilter firewall, we create a base to configure NAT routing on top of. Navigate
to `/etc/wireguard` folder and create a new folder `nft`; Inside it, a new file:
`primary.rules`.

```sh
#!/sbin/nft -f

# Flushing means to empty; the flush command here empties the entire
# nftable ruleset. Since this is the primary file that is loaded onto the
# chronology of nf configurations, it is safe to flush the entire rulset first.
flush ruleset

# I tried segregating my configuration into multiple tables for comprehension
# But was unable to. If you have a better understanding of nftables and
# comprehend how this configuration can be broken apart into multiple tables,
# drop me an email!
table inet pinhole_filter {

    # allow all packets sent by Pi to the outside world
    chain output {
        type filter hook output priority 0; policy accept;
    }

    # This is the default base chain for inpuj
    chain default_input {

        # Blanket policy to drop all incoming packets unless specified below
        type filter hook input priority 0; policy drop;

        # If a connection was requested by Pi itself and established, let it pass
        ct state { established, related } counter accept comment "Accept connections made by the Pi itself"

        # Perhaps an overkill but if the protocol is an ICMP request, route it to a non base chain
        meta l4proto icmp goto icmp_filter
        ip6 nexthdr icmpv6 goto icmp_filter

        # Accept everything originating from loopback
        iifname "lo" accept

        # We segregate packets based on their origin interface. If a packet is
        # is received on the wlan0 interface, route it to the wlan0_filter
        # chain for further processing
        iifname "wlan0" jump wlan0_filter

        # Similarly, if the packet was received from wg0, we process it in
        # wg0_filter chain
        iifname "wg0" jump wg0_filter
    }

    # The ICMP chain for bothm ipv4 and ipv6 packets
    chain icmp_filter {

        # Accept ICMP requests on ipv4
        meta l4proto icmp iifname { "wlan0", "wg0" } ip saddr { 192.168.1.0/24, 10.6.0.0/29 } counter accept comment "icmpv4 : LAN & Wireguard"

         # Accept ICMPv6 requests on ipv6
        meta l4proto icmpv6 iifname { "wlan0", "wg0" } ip6 saddr { fe80::/112, fd00:43:43::/64 } ip6 daddr { ff02::/112, fe80::/112, 2401:4900::/96, fd00:43:43::/64 } counter accept comment "icmpv6: lo & Wireguard"

        # accept neighbour discovery otherwise IPv6 connectivity breaks
        # Refer: https://www.computernetworkingnotes.com/networking-tutorials/ipv6-neighbor-discovery-protocol-explained.html
        icmpv6 type { nd-neighbor-solicit, nd-router-advert, nd-neighbor-advert } accept
    }

    # The wlan0 chain allows us to specify pin hole rules for ports / services
    # That are received on the wlan0 interface
    chain wlan0_filter {

        # Accept SSH requests from wlan0
        tcp dport 22 counter accept comment "Accept: SSH"

        # Accept Wireguard connection requests
        udp dport 51820 counter accept comment "Accept connection requests: Wireguard"

        # Plex has a bad habit of flooding the broadcast address. Block packets originating
        # From plex
        # Note: This is not really required since the base policy is to drop
        # everything that is not allowed specifically. These entries are added
        # explicitly to help identify these ports in the future
        udp dport { 32414, 32412 } drop
    }

    # The wlan0 chain allows us to specify pin hole rules for ports / services
    # That are received on the wg0 interface
    chain wg0_filter {

        # Accept SSH requests
        tcp dport 22 counter accept comment "Accept: SSH"
        udp dport 53 counter accept comment "Accept: DNS"
    }
}
```

## Step Five: Verify ipv4 and ipv6 packet forwarding is enabled

So far, we've only established connectivity between the wireguard server and the
client. The most we can do at the moment is ping the Pi from our Android [Termux][15].
IP forwarding is synonymous with routing and seldom referred to as 'kernel IP
forwarding' because it is a feature of the Linux kernel. A router has multiple
network interfaces; If traffic comes in from one interface that matches a subnet
of another network interface, a router then forwards that traffic to the other
network interface. This functionality is achieved by enabling ipv4 & ipv6 forwarding
on non-router devices. Refer [this][17] for more information on this property.

**Pro Tip**: Updating the value of `net.ipv4.ip_forward` and
`net.ipv6.conf.all.forwarding` in `/proc/sys/net/ipv4` is a temporary gambit.
I suggest using `sysctl` instead.

```sh
# Check the current value; It should be zero since its a fresh install.
$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0
$ net.ipv6.conf.all.forwarding
net.ipv6.conf.all.forwarding = 0

# We overrite the current property with 1 to enable it
$ sysctl --write net.ipv4.ip_forward = 1
net.ipv4.ip_forward = 1
$ sysctl --write net.ipv6.conf.all.forwarding = 1
net.ipv6.conf.all.forwarding = 1
```

## Step Six: Creation of Mullvad Wireguard interfaces

To create Mullvad Wireguard interfaces for each of the servers they provide,
download the Mullvad Wireguard configuration file. Refer [this guide][18] but
follow the steps below:

```sh
# This step is a stow-away suggestion and archival for the script
$ cd /etc/wireguard && mkdir mullvad-config && cd mullvad-config

# Downloads the Mullvad Wireguard interface creation shell script. We shall modify this to our use case
$ curl -LO https://mullvad.net/media/files/mullvad-wg.sh
```

I suggest you read through the mullvad-wg.sh shell script to better understand
the purpose. 

The **for** loop that succeeds *"Writing WireGuard configuration files"* is
responsible for creating individual Wireguard configuration files to connect to
Mullvad. 

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

## Step Seven: Setup Pi with NAT routing

Return to the `/etc/wireguard/` directory and edit `wg0.conf` to incorporate the
new sections identified by `#NEW` tag.

```sh
[Interface]
# The Private key is from the server_private_key file created above
# The key below is just a placeholder. Use your own.
PrivateKey = gK48mAIwIF0sMYXPWWkErZ2+Ofh/p7tv6UyhljiHX1g=

# This is the server address on the wireguard interface
Address = 10.6.0.1/24

# The port on which Wireguard listens for connections
ListenPort = 51820

# Multiple Peer blocks can be added depending on how many peers want to connect
# to this server. For now, we declare just one
[Peer]

# The public key of the wireguard client - Could be another desktop, Linux system
# or a handheld device. The process to create keys on the client largely remain
# the same. Figure out a way to import the public key generated on the client to
# this file. Also, the key below is just a placeholder. Use your own.
PublicKey = DL6qKIOuU2tG4KKqSlMdKwDkSaOv6HM8YSPAr3XHnk8=

# The allowed IPs section declares the IP address of the client. A client may
# connect to this interface using the declared wireguard public key only if
# they accept the following IP address on their interface.
AllowedIPs = 10.6.0.2/24
```

========================= OLD POST =========================


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

========================= OLD POST =========================

## References

Though not mandatory, I do suggest sparing sometime to read through the following
documentation for a better comprehension of a lot of concepts used above:

1. [IP Routing Explained *by NetworkLessons.com*](https://networklessons.com/cisco/ccna-routing-switching-icnd1-100-105/ip-routing-explained)
2. [Configuring Chains *by nftables official wiki*](https://wiki.nftables.org/wiki-nftables/index.php/Configuring_chains)
3. [Man Page *by nftables official man page*](https://www.netfilter.org/projects/nftables/manpage.html)
4. [nftables experiments with ICMP *by a02*](https://ao2.it/en/blog/2018/04/27/nftables-experiments-icmpv6-hop-hop-options-header)
5. [Beginners guide to traffic filtering *by limnux-audit*](https://linux-audit.com/nftables-beginners-guide-to-traffic-filtering/)
6. [NFTables *by Gentoo Wiki*](https://wiki.gentoo.org/wiki/Nftables#Start_logging)
7. [NFTables jumping *by nftables official wiki*](https://wiki.nftables.org/wiki-nftables/index.php/Jumping_to_chain)
8. [NFTable examples *by Gentoo Wiki*](https://wiki.gentoo.org/wiki/Nftables/Examples#Basic_NAT)

[1]: https://archern9.github.io/posts/route-pivpn-traffic-via-mullvad/
[2]: https://mullvad.net/en/
[3]: https://www.netfilter.org/projects/nftables/index.html
[4]: https://pi-hole.net
[5]: https://pivpn.io
[6]: https://www.wireguard.com
[7]: https://github.com/MichaIng/DietPi
[8]: https://ianwright.dev/2020/05/05/headless-pihole-setup-with-dietpi.html
[9]: https://github.com/ArcherN9/Dot-files
[10]: https://www.scaleway.com/en/docs/tutorials/install-wireguard/
[11]: https://nlnetlabs.nl/projects/unbound/about/
[12]: https://play.google.com/store/apps/details?id=com.wireguard.android
[13]: https://feh.finalrewind.org
[14]: http://proftpd.org
[15]: https://termux.com
[16]: https://whatismyipaddress.com/nat
[17]: https://unix.stackexchange.com/questions/14056/what-is-kernel-ip-forwarding
[18]: https://mullvad.net/en/help/wireguard-and-mullvad-vpn/
[19]: 
