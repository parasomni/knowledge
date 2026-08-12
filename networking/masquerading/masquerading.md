# Masquerading & Tunneling

## Concept

Masquerading is a dynamic form of source NAT (SNAT). When a packet leaves an
interface, the kernel rewrites the packet's source address to the address of
the outgoing interface itself, and keeps a conntrack entry so replies get
rewritten back to the original source. Unlike static SNAT, MASQUERADE looks
up the current IP of the interface on every packet, which makes it the right
choice for interfaces with dynamic/DHCP addresses (the alternative, `-j SNAT
--to-source <ip>`, is cheaper but requires a fixed IP).

This is commonly used to:
- Let a private subnet (e.g. `10.0.0.0/24`) reach another network
  (`192.168.1.0/24`) through a router/pivot host, with replies routed back
  correctly.
- Turn a compromised or pivot host into a NAT gateway during a pentest so
  tools on your attack subnet can reach an internal segment they can't
  otherwise route to.
- Provide internet access to an isolated VM/container network.

### Enable IP forwarding

MASQUERADE alone does nothing without forwarding enabled - the kernel will
never route the packet between interfaces:

    sysctl -w net.ipv4.ip_forward=1
    # persist:
    echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.conf

### Accept the forwarded traffic

    iptables -A FORWARD -s 10.0.0.0/24 -d 192.168.1.0/24 -j ACCEPT
    iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

The first rule allows new connections from the private subnet to the
destination network. The second allows return traffic for connections already
tracked by conntrack - without it, replies get dropped by a default-deny
FORWARD policy.

### Masquerade NAT it out eth0 so replies come back

    iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -d 192.168.1.0/24 -o eth0 -j MASQUERADE

This rewrites the source of outgoing packets to `eth0`'s address so replies
from `192.168.1.0/24` come back to this host, which then un-NATs them and
forwards them to the original `10.0.0.0/24` sender via conntrack.

### Useful checks

    # Verify conntrack is tracking the NAT'd flows
    conntrack -L | grep 192.168.1

    # Verify forwarding policy / rule hit counts
    iptables -L FORWARD -v -n
    iptables -t nat -L POSTROUTING -v -n

    # Confirm forwarding is actually enabled
    cat /proc/sys/net/ipv4/ip_forward

---

## Setting up a jumphost (SSH pivot)

A jumphost (bastion/pivot) sits between your attack/admin machine and a
network you can't reach directly, forwarding traffic through itself. Below
are the common approaches, roughly cheapest-to-set-up first.

### 1. Native SSH ProxyJump (preferred, no extra tooling)

    ssh -J user@jumphost user@internal-target

Or persist it in `~/.ssh/config`:

    Host jumphost
        HostName 203.0.113.10
        User admin

    Host internal-target
        HostName 192.168.1.50
        User admin
        ProxyJump jumphost

This chains SSH sessions without opening extra listening ports - the jump
host just relays an encrypted stream, it never sees plaintext traffic.

### 2. Dynamic port forwarding (SOCKS proxy pivot)

Turn the jumphost into a SOCKS proxy so any SOCKS-aware tool (browser,
proxychains, curl) can reach the internal network through it:

    ssh -D 1080 -N -f user@jumphost

Then route arbitrary tools through it:

    proxychains curl http://192.168.1.50/
    # or configure proxychains.conf:
    #   socks5 127.0.0.1 1080

`-N` = no remote command, `-f` = background after auth.

### 3. Local port forward (single service pivot)

Expose one internal service on your local machine through the jumphost:

    ssh -L 8080:192.168.1.50:80 -N -f user@jumphost
    curl http://127.0.0.1:8080/

### 4. Remote port forward (expose your service to the far side)

Useful for callbacks/listeners when you can't route inbound to your machine:

    ssh -R 4444:127.0.0.1:4444 -N -f user@jumphost

Traffic hitting `jumphost:4444` gets forwarded back to `127.0.0.1:4444` on
your machine.

### 5. Full routed pivot (iptables MASQUERADE, as above)

When SSH forwarding is too limited (e.g. you need raw ICMP/UDP scanning
through the pivot, not just TCP), turn the jumphost into an actual router:

    # On the jumphost, with an interface on each network:
    #   eth0 -> attack/admin subnet (10.0.0.0/24)
    #   eth1 -> internal target subnet (192.168.1.0/24)

    sysctl -w net.ipv4.ip_forward=1

    iptables -A FORWARD -s 10.0.0.0/24 -d 192.168.1.0/24 -j ACCEPT
    iptables -A FORWARD -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
    iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -d 192.168.1.0/24 -o eth1 -j MASQUERADE

Then on your attack machine, add a route pointing the internal subnet at the
jumphost:

    ip route add 192.168.1.0/24 via 10.0.0.1   # jumphost's eth0 address

This gives full IP-level access (not just forwarded ports) to
`192.168.1.0/24` from your machine, at the cost of needing root on the
jumphost and modifying its firewall/routing.

### Hardening notes for a jumphost

- Restrict `AllowTcpForwarding`/`GatewayPorts` in `sshd_config` to only what's
  needed if this is a long-lived bastion, not a one-off pivot.
- Prefer key-based auth; disable password auth (`PasswordAuthentication no`).
- Log and rotate `~/.ssh/authorized_keys` access; treat the jumphost as a
  high-value target since it bridges two networks.
- Remove the iptables/forwarding rules and reset `ip_forward` when the
  engagement/pivot is no longer needed.
