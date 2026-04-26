
# Neuron 

## Device Overview

`Neuron` is the main entry point into the homelab network. All users that want
to connect to a device/service within the lab must first connect to the VPN server 
run on `neuron`.

`Neuron` runs on a small linux OS to reduce operational costs and the attack surface. 
It also has extensive custom networking rules for correctly routing user groups 
to accessible services, as well as supporting detailed logging for all connections.

`Neuron` also uses a local static IP to ensure routing is predictable.

---

## Services

### Nftables

Originally, I was using UFW for firewall rules. However, I switched over to nftables
for more granular control over connections and greater control over custom logging.

Below is an older version of the configuration file used before custom service forwarding
implementation. The newest version is still a work in progress but will be updated once
complete.

Documented old config:

```config
#!/usr/sbin/nft -f

flush ruleset

# Defined variables
define pub_iface = "<network_interface_name>"
define wg_iface = "<wireguard_interface_name>"
define wg_port = <wireguard_listening_port>
define ssh_port = <ssh_listening_port>

# firewall rules for both ipv4 and ipv6 (indicated by `inet`)
table inet filter {

        # Rules for incomming connections
        chain input {
                
                # Set the default action to drop packets unless a later rule
                # specifically accepts them.
                type filter hook input priority filter; policy drop;

                # Allow loopback traffic
                iif lo accept

                # Allow established and related traffic
                ct state vmap { invalid:drop, established:accept, related:accept }

                # Drop new connections over rate limit (helps prevent brute force)
                ct state new limit rate over 1/second burst 10 packets drop

                # Accept SSH if coming from the public network interface
                iifname $pub_iface tcp dport $ssh_port accept

                # Accept Wireguard connections from the public network interface
                iifname $pub_iface udp dport $wg_port accept
        }
        # Location for specific wireguard forwarding rules. Will be updated
        # in the new version.
        #chain wg-forward {
                # Filter appropriate traffic
        #}

        # Rules for forwarding packets to other destinations
        chain forward {

                # Set the default action to drop packets attempting to be forwarded
                type filter hook forward priority filter; policy drop;

                # Accept established and related traffic
                #ct state vmap { invalid:drop, established:accept, related:accept }

                # Forward wg vpn packets to accessible devices
                # iifname $wg_iface oifname $pub_iface goto wg-forward
        }

        # Rules for outbound traffic from `neuron`. This might be updated in
        # Future configs for more control over what can be sent.
        chain output {

                # Set the default action to accept packets leaving the server
                type filter hook output priority filter; policy accept;
        }
}

# Rule for making wireguard connections look like the public interface when
# connecting to outside services.
table inet nat {
        chain postrouting {
                type nat hook postrouting priority 100; policy accept;
                iifname $wg_iface oifname $pub_iface masquerade
        }
}
```

---

### SSH (A)

`Neuron` has an active SSH server that uses key-based authentication. It is an admin
connection for remote servicing of the device, no normal users will have access.
Details:
- Key authentication
- listens on a non-standard port
- Blacklisting connections with too many failed attempts
- No X11 forwarding, limit attack surface for graphical applications
- Warning page for connecting users

---

### Wireguard (W/P)

`Neuron` runs a Wireguard VPN server that acts as the entry point for the network.
Configs for new users are automatically generated with a script that can create
either a config file for the user or a qr code to scan.

Due to the nature of Wireguard, users can choose the traffic that is routed through
the wireguard tunnel, using 0.0.0.0/0 for all traffic and defining subnets in the
`AllowedIPS` section for specific traffic.

Wireguard connections are further controlled by the nftables service. Rules have 
been created to control the services that users can try to access, limiting the
scope of connections to only trusted individuals.

