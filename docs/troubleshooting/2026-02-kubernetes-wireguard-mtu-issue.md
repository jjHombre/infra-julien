# MTU Configuration for Kubernetes over WireGuard

> **Date:** February 2026  
> **Duration:** 3 days (intermittent debugging)  
> **Impact:** Node randomly going NotReady, kubelet freezes, cluster instability  
> **Status:** ✅ Resolved

## Problem

Multi-server Kubernetes cluster with nodes connected via WireGuard tunnel:
- Server 1: master + slave1 + slave2 (LAN 192.168.200.0/24)
- Server 2: slave3 (LAN 10.0.1.0/24)
- Inter-server connection: WireGuard VPN

**Symptoms:**
- Node slave3 randomly becomes NotReady
- Kubelet timeout / freeze
- SSH inaccessible during crashes
- No OOMKill, no kernel panic, minimal error logging

**Root Cause:**

Packet fragmentation due to misconfigured MTU cascade:
```
Flannel packet (1450) + headers (28) = 1478 bytes
→ WireGuard encapsulation (+60) = 1538 bytes
→ Physical Ethernet MTU = 1500 bytes
→ FRAGMENTATION → kubelet timeout → node crash
```

The default 1500 MTU was carried through all layers without accounting for encapsulation overhead.

## Solution

Configure MTU cascade respecting each encapsulation layer:

### 1. WireGuard (on physical hosts)
```bash
# /etc/wireguard/wg0.conf
[Interface]
MTU = 1380
# ... rest of config
```

Apply:
```bash
sudo wg-quick down wg0
sudo wg-quick up wg0
```

### 2. VM Interfaces (all nodes)

**With netplan (Ubuntu):**
```yaml
# /etc/netplan/50-cloud-init.yaml
network:
  version: 2
  ethernets:
    eth0:
      mtu: 1350
      addresses:
        - 10.0.1.31/24
      gateway4: 10.0.1.1
      # ... rest of config
```

**Disable cloud-init to persist config:**
```bash
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
network: {config: disabled}
```

Apply:
```bash
sudo netplan apply
```

**With /etc/network/interfaces (Debian):**
```bash
auto eth0
iface eth0 inet dhcp
    mtu 1350
```

Apply:
```bash
sudo ifdown eth0 && sudo ifup eth0
```

### 3. Flannel (auto-calculated)

Force recreation of Flannel interfaces to recalculate MTU:
```bash
# On each node
sudo systemctl stop k3s-agent  # or k3s for master
sudo ip link delete flannel.1
sudo systemctl start k3s-agent
```

Expected result:
- flannel.1 MTU: ~1300
- cni0 MTU: ~1300

## Validation
```bash
# Verify MTU on all nodes
ip link show flannel.1 | grep mtu
ip link show cni0 | grep mtu

# Test for fragmentation (from slave3)
ping -M do -s 1250 192.168.200.20 -c 10
# 0% packet loss = OK

# Verify inter-node routes
ip route show | grep 10.42
```

## Final Architecture
```
Pod (slave3)
  ↓ MTU 1300
Flannel/CNI
  ↓ MTU 1300
VM eth0
  ↓ MTU 1350
WireGuard wg0
  ↓ MTU 1380 (1350 + 30 overhead)
Physical Ethernet
  ↓ MTU 1500
```

**Golden Rule:** Each layer must have MTU lower than the layer below minus overhead.

## Lessons Learned

1. **Always verify MTU on VPN tunnels** (WireGuard, IPSec, OpenVPN)
2. **Default MTU (1500) doesn't work with encapsulation**
3. **Kubernetes + Flannel + WireGuard = triple encapsulation** (watch your MTU!)
4. **MTU symptoms: random timeouts, no explicit errors** (silent killer)
5. **Diagnostic test:** `ping -M do -s <size>` to detect fragmentation
6. **Network issues hide between layers** - check each abstraction level

## Debugging Methodology

What finally worked:

1. Check for packet fragmentation: `ping -M do -s 1400 <target>`
2. Trace MTU at each network layer (physical → tunnel → overlay)
3. Verify WireGuard handshake stability: `wg show wg0`
4. Delete and recreate network interfaces to force recalculation
5. Monitor kubelet logs during failure: `journalctl -u k3s-agent -f`

## References

- [Flannel VXLAN overhead](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md#vxlan): ~50 bytes
- [WireGuard overhead](https://www.wireguard.com/papers/wireguard.pdf): ~60 bytes
- [Path MTU Discovery](https://en.wikipedia.org/wiki/Path_MTU_Discovery)

## Related Issues

- Initial deployment: [link to other doc if exists]
- Future consideration: Implement MTU monitoring/alerting with Prometheus
