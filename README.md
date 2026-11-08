# Cisco Packet Tracer Lab: VLANs, 802.1Q Trunk, Router-on-a-Stick and Guest Isolation ACL

A small Packet Tracer lab built to learn Cisco fundamentals: segmenting a LAN into VLANs, carrying them over an 802.1Q trunk, routing between them with a router-on-a-stick, and isolating a guest VLAN from the internal networks with an extended ACL.

## Objectives

- Create and name VLANs on a switch
- Assign access ports to VLANs
- Configure an 802.1Q trunk with a native VLAN and a restricted list of allowed VLANs
- Set up inter-VLAN routing with router subinterfaces (router-on-a-stick)
- Isolate the guest VLAN from the internal VLANs with an extended ACL
- Verify everything with `show` commands and `ping`

## Network Diagram

![Topology](images/topology.png)

```
 Office devices (VLAN 10) --\
                             \
 Technical devices (VLAN 20) -- [ Switch ] --- Gi0/1 (trunk) --- Gi0/0 [ Router ]
                             /
 Guest devices (VLAN 30) ---/
```

## VLANs and Addressing Plan

| VLAN | Name | Network | Gateway (Router subinterface) |
|------|------|---------|-------------------------------|
| 10 | OFFICE | 172.17.10.0/24 | 172.17.10.1 (Gi0/0.10) |
| 20 | TECHNICAL | 172.17.20.0/24 | 172.17.20.1 (Gi0/0.20) |
| 30 | GUESTS | 172.17.30.0/24 | 172.17.30.1 (Gi0/0.30) |
| 99 | NATIVE | - | - (Gi0/0.99, native VLAN) |

End devices use the `.1` address of their VLAN as their default gateway.

## Trunk Link

| From | Port | To | Port |
|------|------|----|------|
| Switch | Gi0/1 | Router | Gi0/0 |

The trunk uses 802.1Q encapsulation, native VLAN 99, and only allows VLANs 10, 20 and 30.

## Step-by-Step Build

1. **Build the topology** in Packet Tracer and cable the switch to the router, then connect the end devices to access ports.
2. **Switch**: create the VLANs, configure the access ports, configure the trunk.
3. **Router**: bring up the physical interface (no IP) and create one subinterface per VLAN.
4. **ACL**: isolate the guest VLAN from the office and technical VLANs.
5. **Verify** with `show` commands and `ping` tests.
6. **Save** the configs and export the running-configs.

## Switch Configuration

### 1. Create and declare the VLANs

```
Switch> enable
Switch# configure terminal
vlan 10
 name office
vlan 20
 name technical
vlan 30
 name guests
vlan 99
 name native
exit
```

Verification:

```
Switch# show vlan brief           ! checks that the VLAN IDs exist and are active
```

### 2. Configure the access ports

```
! Single port
Switch(config)# interface fastEthernet 0/11
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 10
Switch(config-if)# no shutdown
Switch(config-if)# exit

! Port range
Switch(config)# interface range fastEthernet 0/1 - 10
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport access vlan 10
Switch(config-if-range)# no shutdown
Switch(config-if-range)# exit
```

Repeat the same pattern with `switchport access vlan 20` and `switchport access vlan 30` for the ports of the other VLANs.

Verification:

```
Switch# show interfaces fastEthernet 0/11 switchport
Switch# show ip interface brief
```

Check for `Operational Mode: static access`, the assigned VLAN ID, and a port status of `up / up`.

### 3. Configure the 802.1Q trunk

```
Switch(config)# interface gigabitEthernet 0/1
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk native vlan 99
Switch(config-if)# switchport trunk allowed vlan 10,20,30
Switch(config-if)# no shutdown
Switch(config-if)# exit
```

Verification:

```
Switch# show interfaces trunk
```

Check for trunking status, `802.1q` encapsulation, the native VLAN ID, and the restricted list of allowed VLANs.

### 4. Save and factory reset

```
! Save to NVRAM
Switch# copy running-config startup-config

! Full reset
Switch# erase startup-config
Switch# delete flash:vlan.dat
Switch# reload
```

Confirm each step with Enter without changing the file name.

## Router Configuration

### 1. Enable the physical interface

The interface carrying the trunk does not get an IP address itself.

```
Router> enable
Router# configure terminal
Router(config)# interface gigabitEthernet 0/0
Router(config-if)# no ip address
Router(config-if)# no shutdown
Router(config-if)# exit
```

### 2. Configure the subinterfaces

Always set the 802.1Q encapsulation before the gateway IP address.

```
! Subinterface VLAN 10 (Office)
Router(config)# interface gigabitEthernet 0/0.10
Router(config-subif)# encapsulation dot1Q 10
Router(config-subif)# ip address 172.17.10.1 255.255.255.0
Router(config-subif)# no shutdown
Router(config-subif)# exit

! Subinterface VLAN 20 (Technical)
Router(config)# interface gigabitEthernet 0/0.20
Router(config-subif)# encapsulation dot1Q 20
Router(config-subif)# ip address 172.17.20.1 255.255.255.0
Router(config-subif)# no shutdown
Router(config-subif)# exit

! Subinterface VLAN 30 (Guests)
Router(config)# interface gigabitEthernet 0/0.30
Router(config-subif)# encapsulation dot1Q 30
Router(config-subif)# ip address 172.17.30.1 255.255.255.0
Router(config-subif)# no shutdown
Router(config-subif)# exit

! Subinterface for the native VLAN (security)
Router(config)# interface gigabitEthernet 0/0.99
Router(config-subif)# encapsulation dot1Q 99 native
Router(config-subif)# no shutdown
Router(config-subif)# exit
```

### 3. Inter-VLAN security filtering (ACL)

Isolate the guest VLAN from the internal subnets while still allowing everything else.

```
Router(config)# ip access-list extended ACL_GUEST_ISOLATION
Router(config-ext-nacl)# deny ip 172.17.30.0 0.0.0.255 172.17.10.0 0.0.0.255
Router(config-ext-nacl)# deny ip 172.17.30.0 0.0.0.255 172.17.20.0 0.0.0.255
Router(config-ext-nacl)# permit ip any any
Router(config-ext-nacl)# exit

Router(config)# interface gigabitEthernet 0/0.30
Router(config-subif)# ip access-group ACL_GUEST_ISOLATION in
Router(config-subif)# exit
```

### 4. Verification and troubleshooting

```
Router# show ip interface brief
```
`GigabitEthernet0/0` and each subinterface (`.10`, `.20`, `.30`) must be `up / up`.

```
Router# show ip route
```
Each network must appear with the `C` code (directly connected).

```
Router# show access-lists
```
Shows the match counters on the deny and permit rules.

```
Router# copy running-config startup-config
```
Saves the configuration.

## Tests and Expected Results

| Test | Expected result |
|------|-----------------|
| Office -> Technical (`ping`) | Success (inter-VLAN routing) |
| Technical -> Office (`ping`) | Success |
| Guests -> Office (`ping`) | Blocked by the ACL |
| Guests -> Technical (`ping`) | Blocked by the ACL |
| Office -> Guests (`ping`) | Blocked (the reply is filtered on the way back) |
| Guests -> their own gateway 172.17.30.1 | Success |

## Screenshots

| Topology | Successful ping | Blocked ping (ACL) |
|----------|-----------------|--------------------|
| ![](images/topology.png) | ![](images/ping-ok.png) | ![](images/ping-blocked.png) |

## Repository Structure

```
.
├── README.md
├── network.pkt          # Packet Tracer file
├── configs/
│   ├── switch.txt       # switch running-config
│   └── router.txt       # router running-config
└── images/
    ├── topology.png
    ├── ping-ok.png
    └── ping-blocked.png
```

## Skills Demonstrated

- VLAN creation and port assignment
- 802.1Q trunking (native VLAN, allowed VLAN list)
- Inter-VLAN routing with router subinterfaces
- Extended ACLs for network segmentation
- Troubleshooting with `show` commands and `ping`

## Notes and Known Issues

- The first pings may fail right after opening the file: ARP needs a moment to resolve.
- Add anything else you notice here (Packet Tracer quirks, things that did not work, etc.).

## What I Learned

- The difference between an access port and a trunk port
- Why a router is needed for VLANs to communicate, and why the native VLAN matters
- Why the encapsulation must be set before the IP address on a subinterface
- ACL direction (`in` / `out`) and why rule order matters

## Next Steps

- Add DHCP pools on the router, one per VLAN
- Add a server (DNS / Web) in its own VLAN
- Add NAT to simulate Internet access
- Harden the switch (port security, passwords, SSH)

## Tools

- Cisco Packet Tracer
- Git / GitHub
