# Phase 0 — Evidence 01: Host network adapters

**Exit criterion 1:** the Windows host must have no network adapter on any lab segment.
**Date:** 2026-07-28
**Command:** `ipconfig /all`

Redacted per [`../../docs/sanitization-policy.md`](../../docs/sanitization-policy.md).

---

## Output (relevant extract)

```
Host name . . . . . . . . . . . . : [REDACTED]

Ethernet adapter Ethernet:
   Description . . . . . . . . . . : Realtek Gaming 2.5GbE Family Controller
   Physical Address. . . . . . . . : [REDACTED]
   IPv4 Address. . . . . . . . . . : [REDACTED - home LAN]
   Default Gateway . . . . . . . . : [REDACTED - home LAN]

Ethernet adapter VMware Network Adapter VMnet1:
   IPv4 Address. . . . . . . . . . : 192.168.31.1
   Subnet Mask . . . . . . . . . . : 255.255.255.0

Ethernet adapter VMware Network Adapter VMnet8:
   IPv4 Address. . . . . . . . . . : 192.168.231.1
   Subnet Mask . . . . . . . . . . : 255.255.255.0
```

## Result

No adapter present on `10.10.10.0/24` — VMnet2, the lab LAN.
No adapter present on `10.10.20.0/24` — VMnet3, the detonation segment.

## Corroboration

Cross-checked against the Virtual Network Editor, which lists VMnet2 and VMnet3 with
type **Custom** and no host connection.

VMware reclassifies a host-only network as Custom once the host virtual adapter is
removed, so the label itself is corroborating evidence: a segment still reading
"Host-only" would mean the setting had reverted.

Two independent sources agree. **Criterion met.**
