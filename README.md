Ansible Role: Interfaces
========================

Role for [Ansible](https://www.ansible.com/) to configure the network
interfaces of Debian-like machines.

It contains a Bash-script **watchdog intended to revert network configuration**
in case the machine could not ping the Ansible-host for more than a few seconds
(configurable).

Inspired by these roles:

* [michaelrigart/ansible-role-interfaces](https://github.com/michaelrigart/ansible-role-interfaces)
* [mrlesmithjr/ansible-config-interfaces](https://github.com/mrlesmithjr/ansible-config-interfaces)

Requirements
------------

**Only for Debian-like** machines.

Nothing else, except, obviously, an already working network interface, in order
to let Ansible connect to the machine by SSH.

Role Variables
--------------

All interfaces and their configuration are inside the list
`interfaces_interfaces`, which contains either a sublevel for IP version (`v4`
and `v6`) or not (in which case, it's supposed to be IPv4). Then, there are:

* `name`: Name of the interface (required);
* `auto`: Wether it must be started at boot or not (optionnal, default: true);
* `comment`: A purely informational comment (optionnal);
* `v4`|`v6`|`(null)`: Wether it's IPv4 (default if not specified) or IPv6;
  * `method`: The method (static, dhcp, manual, etc.) (required);
  * `address`: IP address (required if method=static);
  * `gateway`: The default gateway (optionnal);
  * `other_settings`: All other attributes could be setup here manually (broadcast, mtu, vlan-raw_device, etc.) (optionnal).

The watchdog guard is defined by `interfaces_watchdog_duration` (default 15s)
and `interfaces_watchdog_max_downtime` (default 4s).

Here is an example of `interfaces_interfaces`:


```yaml
interfaces_interfaces:

  # Ethernet interfaces ##########################
  - name: eth0
    comment: Management (Ethernet Port 1)
    v4:
      method: static
      address: 10.99.0.1/24
      other_settings:
        - dns-nameservers 10.0.0.1 10.0.0.2
        - dns-search oob.example.com example.com
        - up ip route add 10.1.1.0/24 via 10.99.0.254 dev $IFACE
        - down ip route del 10.1.1.0/24 via 10.99.0.254 dev $IFACE
    v6:
      method: static
      address: "fd6f:d42f:2670::1ab/64"
      other_settings:
        - "dns-nameservers fd6f:d42f:2670::1 fd6f:d42f:2670::2"

  - name: eth1
    comment: Ethernet Port 2
    method: manual

  - name: eth2
    comment: Ethernet Port 3
    method: manual

  - name: eth3
    comment: Storage Traffic (Ethernet Port 4)
    auto: false
    method: static
    address: 192.168.9.9/24
    other_settings:
      - mtu 9000

  # Bonding interfaces ###########################
  - name: bond0
    comment: VM Traffic
    method: manual
    other_settings:
      - bond-mode 802.3ad
      - bond-slaves eth1 eth2
      - bond-xmit_hash_policy layer3+4
      - bond-miimon    100
      - bond-downdelay 200
      - bond-updelay   200

  # VLAN interfaces ##############################
  - name: vlan9_dmz
    comment: DMZ VLAN
    method: manual
    other_settings:
      - vlan_raw_device bond0

  - name: vlan100_users
    comment: Users VLAN
    method: manual
    other_settings:
      - vlan_raw_device bond0

  # Bridge interfaces ############################
  - name: br_dmz
    comment: DMZ
    v4:
      method: static
      address: 10.0.101.1
      gateway: 10.0.101.254
      other_settings:
        - netmask 255.255.255.0
        - bridge_ports vlan9_dmz
        - bridge_stp off
        - bridge_fd 0
    v6:
      method: static
      address: 10.0.101.1
      gateway: 10.0.101.254

  - name: br_users
    comment: Users
    method: dhcp
    other_settings:
      - bridge_ports vlan100_users
      - bridge_stp off
      - bridge_fd 0
```

Hint: You can merge two lists if you want to split this `interfaces_interfaces`
variable into two variables (by example, one per machine, and another common to
all machines). Note however that you must **not** have the same interface
specified in both lists! Here is a way to do that:

```yaml
# Interfaces only on the current host:
interfaces_interfaces_local:
  - { name: eth0, ... }
  - { name: eth1, ... }

# Interfaces common to all hosts:
interfaces_interfaces_common:
  - { name: bond0, ... }
  - { name: vlan100, ... }

# Fusion(union) of these two lists:
interfaces_interfaces: "{{ interfaces_interfaces_local | union( interfaces_interfaces_common ) }}"
```

Dependencies
------------

None.

Example Playbook
----------------

```yaml
- hosts: servers
  roles:
  - role: mathieumd.interfaces
    interfaces_interfaces:
      - name: eth0
        comment: Default interface
        method: static
        address: 10.0.0.10/24
        gateway: 10.0.0.254
        other_settings:
          - dns-nameservers 10.0.0.253 10.0.0.254
          - dns-search oob.example.com example.com
          - up ip route add 10.1.1.0/24 via 10.0.0.54 dev $IFACE
          - down ip route del 10.1.1.0/24 via 10.0.0.54 dev $IFACE
```

Notes
-----

This roles works for adding interfaces and IP addresses. But to remove them,
you have to do it manually:

```bash
# Remove an IP from an interface:
IFNAME=ethX
IPADDR=192.0.2.123
ip address del ${IPADDR:?} dev ${IFNAME:?}

# Remove an interface:
IFNAME=bondX
IFTYPE=bond
ifdown ${IFNAME:?} \
  && ip link delete dev ${IFNAME:?} type ${IFTYPE:?} \
  && rm -f /etc/network/interfaces.d/ansible-${IFNAME:?}
```

License
-------

GPLv3

