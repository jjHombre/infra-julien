# Troubleshooting Index

Chronological list of major incidents and their resolutions.

## 2026

- **[2026-02-16] Kubernetes WireGuard MTU Issue** ([details](2026-02-kubernetes-wireguard-mtu-issue.md))
  - **Impact:** Node randomly NotReady, cluster instability
  - **Root cause:** MTU fragmentation over WireGuard tunnel
  - **Resolution:** Configured MTU cascade (WG 1380 → VM 1350 → Flannel 1300)

## Template

When adding new incidents, use this format:
- **[YYYY-MM-DD] Title** ([details](filename.md))
  - **Impact:** Brief description
  - **Root cause:** One-liner
  - **Resolution:** One-liner
