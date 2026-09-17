# ifctl

hostname.if-ish (openbsd) style network configuration for linux in ~500sloc of POSIX shell

this exists to have clean, concise, organized, and readable network configuration and was created for my distro, Basix Linux

all private information in config files such as PSKs or wireguard keys do not leak to `ps`, and each config files permissions are kept at 600

despite this being a shell script, config files are parsed as opposed to sourced/executed

# dependencies

* an `ip` provider such as iproute2
* a dhcp provider, particularly dhcpcd or udhcpc
* `wpa_supplicant` for wireless, iwd is avoided due to the reliance on dbus
* `wg` from wireguard-tools for wireguard
* any POSIX-capable shell

# installation

put the script somewhere in $PATH, a good location tends to be `/usr/local/bin`

# usage

```
 usage: ifctl up   [interface ...]   (default: all of /etc/ifctl/interface.*)
        ifctl down [interface ...]
        ifctl show [interface ...]
        ifctl -n up eth0             (dry run: print commands, run nothing)
```

given ifctl itself is not a daemon, it should either be invoked manually or as part of a oneshot service from whatever service manager you use

configuration is done in `/etc/ifctl` by default and each interface gets one config file, e.g. `/etc/ifctl/interface.eth0`, and the default config directory can be overridden by setting the CONFDIR env var.

# bridge/static config example

interface.br0:
```
create bridge
inet 10.0.1.50/24
gateway 10.0.1.1
```

interface.eth0:
```
requires br0
master br0
```

# wireguard example

when using ifctl to configure a wireguard interface, all the required information may be extracted from a standard wireguard config file:

```
requires eth0
create wireguard
wgkey <wireguard key>
wgport 51820
wgpeer <peer key> \
    wgendpoint vpn.example.org 51820 \
    wgaip 10.7.0.0/24 wgaip fd00:7::/64 \
    wgpka 25
inet 10.7.0.2/24
mtu 1420
route 10.7.0.0/16
```

# full tunnel wireguard

a default route pointing at a tunnel also matches the tunnels own encrypted
packets to the peer endpoint, which is a routing loop

`wgtable` sets the interfaces fwmark and the policy rules that let wireguards
own traffic keep using the main table while everything else uses the tunnel:

```
requires eth0
create wireguard
wgkey <wireguard key>
wgpeer <peer key> \
    wgendpoint vpn.example.org 51820 \
    wgaip 0.0.0.0/0 wgaip ::/0 \
    wgpka 25
inet 10.7.0.2/24
inet6 fd00:7::2/64
mtu 1420

wgtable auto
route 0.0.0.0/0
route ::/0
```

`wgtable auto` picks the lowest table with no routes in it or gives it a number,
`route` and `gateway` lines that **come after it** go into that table unless they
name a table themselves

specific routes in the main table still win, only the main default is ignored,
so LAN traffic isn't tunneled

these rules are removed on `ifctl down`

## rp_filter

with strict reverse path filtering (`net.ipv4.conf.*.rp_filter=1`) the kernel
revalidates inbound packets by looking up their source as a destination

since inbound packets dont have a fwmark, that lookup resolves the peer endpoint
to the tunnel itself and the handshake reply gets dropped

`wgtable` handles this the same way wg-quick does, nft is used if present, iptables otherwise,
if neither of these are available ifctl will warn you, the only fix here is either like
installing one of those two things or setting `net.ipv4.conf.all.rp_filter` to 2 instead,
ifctl doesnt change that automatically because that would be stinky

# wifi/dhcp example

when defining multiple wireless interfaces, the one that matches available networks first from top to bottom will be the one that gets a connection:

```
join home wpakey psk
join school wpakey psk
join guest
inet autoconf
inet6 autoconf
```

# there's not enough features!

one may also run whatever commands they like when the interface is brought up/down:

```
inet autoconf

# ! executes on interface up
! sysctl -qw net.ipv4.conf.$if.rp_filter=2

# -! executes on interface down
-! sysctl -qw net.ipv4.conf.$if.rp_filter=1
```

`$if` is also valid as part of up/down commands, but only there

you may have as many `!` and `-!` lines as you like, and they will be executed in the order they're listed in

# contributing

as this script is largely considered complete feature additions are not the primary goal, however, strict minimalism is not particularly the point of this despite its small size- features that fit the vibe check are acceptable

bug reports/fixes, documentation improvements, etc. are desireable
