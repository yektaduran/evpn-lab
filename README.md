# evpn-lab: BGP Unnumbered Underlay on Containerlab + FRR

A reproducible 2-spine / 2-leaf Clos fabric built with [Containerlab](https://containerlab.dev/) and [FRRouting](https://frrouting.org/), running on WSL2. The underlay uses **eBGP unnumbered** (interface-based peering, no point-to-point subnets), and the `l2vpn evpn` address-family is already activated on every session for the VXLAN-EVPN overlay.

Every node's configuration is bind-mounted from `configs/`, so the lab survives `clab redeploy` and the whole fabric state lives in this repo.

## Topology

```
      +--------+          +--------+
      | spine1 |          | spine2 |   AS 65000
      +--------+          +--------+
       |      \            /      |
       |       \          /       |
  eth1 |   eth2 \        / eth1   | eth2
       |         \      /         |
       |          \    /          |
       |           \  /           |
       |            \/            |
       |            /\            |
       |           /  \           |
       |          /    \          |
       |         /      \         |
  eth1 |   eth2 /        \ eth1   | eth2
       |       /          \       |
       |      /            \      |
      +--------+          +--------+
      | leaf1  |          | leaf2  |   AS 65011 / AS 65012
      +--------+          +--------+
```

| Node   | Loopback  | ASN   | Links                                  |
|--------|-----------|-------|----------------------------------------|
| spine1 | 10.0.0.1  | 65000 | eth1 -> leaf1:eth1, eth2 -> leaf2:eth1 |
| spine2 | 10.0.0.2  | 65000 | eth1 -> leaf1:eth2, eth2 -> leaf2:eth2 |
| leaf1  | 10.0.0.11 | 65011 | eth1 -> spine1, eth2 -> spine2         |
| leaf2  | 10.0.0.12 | 65012 | eth1 -> spine1, eth2 -> spine2         |

## Requirements

- Docker
- Containerlab 0.79+
- `frrouting/frr:latest` (pulled automatically)

## Usage

```bash
sudo clab deploy -t topo.clab.yml      # bring the fabric up
sudo clab redeploy -t topo.clab.yml    # rebuild after a config change
sudo clab destroy -t topo.clab.yml     # tear down
docker exec -it clab-evpn-basic-leaf1 vtysh   # router CLI
```

## Verification

```bash
docker exec clab-evpn-basic-leaf1 vtysh -c "show ip bgp summary"
docker exec clab-evpn-basic-leaf1 vtysh -c "show bgp l2vpn evpn summary"
docker exec clab-evpn-basic-leaf1 vtysh -c "show ip route bgp"
docker exec clab-evpn-basic-leaf1 ping -c3 -I 10.0.0.11 10.0.0.12
```

Expected on a leaf: both peers `Established` with `PfxRcd 2` (the spine's loopback plus the far leaf's), and ECMP reachability to the other leaf's loopback across both spines. The `l2vpn evpn` session rides the same TCP connection and shows `PfxRcd 0` until a VNI is defined.

## How configuration persists

`configs/<node>.conf` is bind-mounted to `/etc/frr/frr.conf` inside each container, and `configs/daemons` enables `bgpd` (disabled in the stock image). Because `frr.conf` exists, vtysh writes in integrated single-file mode, so `write` inside vtysh updates the file in this repo directly.

Three things to know:

- **Never `docker restart` a node.** It recreates the container's network namespace and destroys the veth links Containerlab built. Interfaces vanish, unnumbered BGP has nothing to discover over, and every peer sits in `Idle` with AS `0`. To restart routing only: `docker exec clab-evpn-basic-leaf1 /etc/init.d/frr restart`.
- **`write` changes file ownership.** FRR rewrites the config as its own user with mode 0600, so a plain `sed`, `nano` or `git add` from the host fails with `Permission denied`. Fix before editing or committing: `sudo chown $USER: configs/*.conf && chmod 644 configs/*.conf`.
- **Host-side edits need a redeploy.** Editors and `sed -i` usually replace the file's inode, which a single-file bind mount does not follow, so the container may keep reading the old content. After editing on the host, run `sudo clab redeploy -t topo.clab.yml`.

## Known-harmless messages

| Message | Why |
|---|---|
| `Can't open configuration file /etc/frr/vtysh.conf` | That file is not used. |
| `Error renaming /etc/frr/frr.conf to /etc/frr/frr.conf.sav: Resource busy` | A single-file bind mount cannot be renamed; FRR falls back to writing in place and the save succeeds. |
| `STARVATION: task vtysh_rl_read ran for ...ms` | A long foreground command such as `ping` blocked the vtysh thread. |

## Roadmap

- [x] eBGP unnumbered underlay with loopback reachability over ECMP
- [x] `l2vpn evpn` address-family activated on all peers
- [ ] VTEP interfaces and a first L2VNI
- [ ] `advertise-all-vni`, type-2 / type-3 route verification
- [ ] Host containers and end-to-end L2 reachability across the fabric
