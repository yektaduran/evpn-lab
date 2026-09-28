# evpn-lab: BGP Unnumbered Underlay, VXLAN-EVPN Overlay and Symmetric IRB

A reproducible 2-spine / 2-leaf Clos fabric built with [Containerlab](https://containerlab.dev/) and [FRRouting](https://frrouting.org/), running on WSL2.

The underlay is eBGP unnumbered (interface-based peering, no point-to-point subnets). The overlay carries two L2VNIs and one L3VNI inside a tenant VRF, so the fabric demonstrates EVPN route types 2, 3 and 5 and symmetric IRB routing between subnets.

Every node's routing configuration is bind-mounted from `configs/` and every kernel object is created from the topology's `exec` blocks, so the whole fabric rebuilds from this repo with one command.

## Topology

```
      spine1 (AS 65000)        spine2 (AS 65000)
        10.0.0.1                 10.0.0.2
         |     \                 /     |
    eth1 |      \  eth2    eth1 /      | eth2
         |       \             /       |
         |        \           /        |
    eth1 |         \         /         | eth2
         |          \       /          |
      leaf1 (AS 65011)        leaf2 (AS 65012)
        10.0.0.11                10.0.0.12
```

Each leaf has two uplinks, one to each spine, giving ECMP across the fabric.

| Node   | Loopback  | ASN   | Role        |
|--------|-----------|-------|-------------|
| spine1 | 10.0.0.1  | 65000 | transit     |
| spine2 | 10.0.0.2  | 65000 | transit     |
| leaf1  | 10.0.0.11 | 65011 | VTEP        |
| leaf2  | 10.0.0.12 | 65012 | VTEP        |

### Overlay

All VNIs live in VRF `tenant-a` (routing table 1000).

| VNI  | Type | Bridge  | VXLAN if | Subnet           | leaf1 SVI      | leaf2 SVI      | RT          |
|------|------|---------|----------|------------------|----------------|----------------|-------------|
| 10   | L2   | br10    | vni10    | 192.168.10.0/24  | 192.168.10.11  | 192.168.10.12  | 65000:10    |
| 20   | L2   | br20    | vni20    | 192.168.20.0/24  | 192.168.20.11  | 192.168.20.12  | 65000:20    |
| 1000 | L3   | br1000  | vni1000  | transit          | Rmac ...:00:11 | Rmac ...:00:12 | 65000:1000  |

Route distinguishers are per VTEP (`<loopback>:<vni>`); route targets are identical on both leaves. See "Why route targets are pinned manually" below.

## Requirements

- Docker
- Containerlab 0.79+
- `frrouting/frr:latest` (pulled automatically)

## Quick start

```bash
sudo clab deploy -t topo.clab.yml      # bring the fabric up
sudo clab redeploy -t topo.clab.yml    # rebuild after a change
sudo clab destroy -t topo.clab.yml     # tear down
```

Wait about 90 seconds after a deploy before testing. See "Startup timing" below.

Open a router CLI:

```bash
docker exec -it clab-evpn-basic-leaf1 vtysh
```

Open a shell inside a node, for `ip` and `bridge` commands:

```bash
docker exec -it clab-evpn-basic-leaf1 bash
```

## Verification

### Underlay

```bash
docker exec clab-evpn-basic-leaf1 vtysh -c "show ip bgp summary"
```

Both peers `Established` with `PfxRcd 2` on a leaf (the spine's loopback plus the far leaf's), `PfxRcd 1` on a spine.

### Overlay

```bash
docker exec clab-evpn-basic-leaf1 vtysh -c "show evpn vni"
```

```
VNI    Type  VxLAN IF   # MACs  # ARPs  # Remote VTEPs  Tenant VRF
10     L2    vni10      2       4       1               tenant-a
20     L2    vni20      2       4       1               tenant-a
1000   L3    vni1000    1       1       n/a             tenant-a
```

VNI 1000 must show `Type: L3`. If it shows `L2`, or `show evpn vni detail` reports `SVI-If: None` and `State: Down`, the L3VNI is not operational and no type-5 routes will be generated.

### Route types

```bash
docker exec clab-evpn-basic-leaf1 vtysh -c "show bgp l2vpn evpn"
docker exec clab-evpn-basic-leaf1 vtysh -c "show evpn mac vni 10"
```

Expect type-2 (MAC/IP), type-3 (IMET) and type-5 (IP prefix) routes. Type-5 entries carry an `Rmac` extended community, which is the L3VNI router MAC the remote VTEP forwards to.

### Symmetric IRB forwarding

```bash
docker exec clab-evpn-basic-leaf2 vtysh -c "show ip route vrf tenant-a"
docker exec clab-evpn-basic-leaf2 ip route get 192.168.10.11 vrf tenant-a
docker exec clab-evpn-basic-leaf2 ip vrf exec tenant-a ping -c3 192.168.10.11
```

```
B>* 192.168.10.11/32 [20/0] via 10.0.0.11, br1000 onlink
192.168.10.11 via 10.0.0.11 dev br1000 table 1000
```

`dev br1000` is the proof: the packet is routed into the L3VNI rather than bridged over the L2VNI. That is symmetric IRB.

The `/24` prefixes stay `C>*` (connected) because both subnets exist locally on both leaves, so the type-5 `/24` never wins best-path. The `/32` host routes win on longest-prefix match and are the ones that cross the L3VNI.

## Design notes

### Why route targets are pinned manually

FRR derives route targets as `AS:VNI`. With eBGP to the leaf, each leaf has a different ASN, so leaf1 would export `65011:10` while leaf2 imports `65012:10`. The RTs never match, routes land in the BGP table but are never imported into a VNI, and traffic silently fails while every session shows `Established`.

Every VNI here therefore carries an explicit `route-target import` / `export` of `65000:<vni>`, identical on both leaves. Route distinguishers stay per VTEP.

### L3VNI wiring

FRR's L3VNI requires a full chain:

```
vni1000 (vxlan)  ->  br1000 (bridge, acts as the SVI)  ->  tenant-a (VRF)
```

Enslaving the VXLAN interface directly to the VRF does not work. FRR reports `SVI-If: None`, the L3VNI stays `Down`, no router MAC is derived, and type-5 advertisement is silently impossible. The VXLAN interface of an L3VNI carries no IP address of its own.

`vrf tenant-a / vni 1000` in `frr.conf` is what designates VNI 1000 as the L3VNI. Without it the VNI is treated as another L2 segment.

### Type-5 generation

Under the VRF's BGP instance:

```
router bgp 65011 vrf tenant-a
 address-family ipv4 unicast
  redistribute connected
 address-family l2vpn evpn
  advertise ipv4 unicast
```

`redistribute connected` puts the connected subnets into the VRF's IPv4 table; `advertise ipv4 unicast` re-originates them as type-5. Both are needed. The command is `advertise ipv4 unicast`, with a space, in the VRF instance's `l2vpn evpn` family.

## Operational notes

These are the failure modes this lab actually hit. Each one costs an hour if you do not know it.

### Never use `docker restart` on a node

It recreates the container's network namespace and destroys the veth links Containerlab built. Interfaces disappear, unnumbered BGP has nothing to discover over, and every peer sits in `Idle` with AS `0` while the configuration still looks correct.

To restart routing only, leaving links intact:

```bash
docker exec clab-evpn-basic-leaf1 /etc/init.d/frr restart
```

### The lab does not survive a host reboot

Containerlab links are not recreated when Docker restarts containers after a reboot. Run `sudo clab redeploy -t topo.clab.yml` after every host restart. Because all state lives in this repo, that is cheap.

### Startup timing: wait 90 seconds

On this image zebra segfaults on its first start (`zebra crashed in startup, signal 11`, right after `Disabling MPLS support (no kernel support)`). Watchfrr detects it and restarts the daemons roughly 60 seconds later, after which everything works.

Inspecting the lab inside that window shows `zebra is not running`, empty interface lists and `Not all daemons are up, cannot write config`. None of that indicates a configuration problem. Wait, then look.

### FRR takes ownership of the config files

Running `write` inside vtysh rewrites `configs/<node>.conf` as FRR's own user with mode 0600. Afterwards `nano`, `sed`, `grep` and `git add` on the host all fail with `Permission denied`. Before editing or committing:

```bash
sudo chown $USER:$USER configs/*.conf && chmod 644 configs/*.conf
```

### Two CLIs, two layers

FRR does not create interfaces. `ip link`, `bridge` and VRF commands belong to the kernel and run in the node's shell; `router bgp`, `interface` and `show` belong to FRR and run in vtysh. Writing `interface br10` in vtysh only records intent for a device that must already exist. This is the main mental shift coming from an integrated vendor OS.

### Ping must be bound to the VRF

Once an SVI is enslaved to `tenant-a`, its subnet lives in table 1000, not the main table. A plain `ping` leaves through the default VRF, matches the management default route out `eth0` and vanishes with no error message. Use `-I <svi>` or `ip vrf exec tenant-a ping ...`.

### Enslaving an interface to a VRF clears its addresses

The topology re-applies SVI addresses with `ip addr replace` at the end of each leaf's `exec` block, after VRF enslavement, to win the race against FRR applying them from `frr.conf` first.

### Editing YAML

`nano`'s auto-indent silently breaks the topology file, producing `field links not found in type core.Config`. Disable it:

```bash
echo 'set autoindent off' >> ~/.nanorc
```

## Known-harmless messages

| Message | Why |
|---|---|
| `Can't open configuration file /etc/frr/vtysh.conf` | That file is not used. |
| `Error renaming /etc/frr/frr.conf to /etc/frr/frr.conf.sav: Resource busy` | A single-file bind mount cannot be renamed. FRR falls back to writing in place and the save succeeds. |
| `STARVATION: task vtysh_rl_read ran for ...ms` | A long foreground command such as `ping` blocked the vtysh thread. |
| `Disabling MPLS support (no kernel support)` | The WSL2 kernel has no MPLS module. Nothing here needs it. |

## Roadmap

- [x] eBGP unnumbered underlay with loopback reachability over ECMP
- [x] `l2vpn evpn` address-family activated on all peers
- [x] VTEPs and two L2VNIs (10, 20)
- [x] Type-3 (IMET) flooding and type-2 (MAC/IP) advertisement
- [x] VRF `tenant-a`, L3VNI 1000, symmetric IRB
- [x] Type-5 IP prefix routes between leaves
- [x] Full state rebuilt from the topology file, verified from a cold deploy
- [ ] Host containers on each leaf, with VNI 10 only on leaf1 and VNI 20 only on leaf2, so inter-subnet traffic must cross the L3VNI
- [ ] MAC mobility: move a host between leaves and watch the type-2 sequence number increment
- [ ] Link and node failure tests over ECMP
