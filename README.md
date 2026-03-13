# CCIE EI VRF + MP-BGP Lab (No MPLS Data Plane)

This lab simulates a CCIE Enterprise Infrastructure control plane scenario using:
- VRF definitions
- MP-BGP with VPNv4 route exchange
- Proper route-target and RD configuration

> ⚠️ Note: This lab uses Catalyst 8000v (IOS XE 17.13.01a) which does **not** support MPLS forwarding. The focus here is on control plane logic, not MPLS transport.

---

## 🧪 Lab Overview

Two routers (R1 and R2) are configured with:

- VRFs `BLUE` and `RED`
- Loopbacks and subinterfaces mapped to VRFs
- MP-BGP over loopback peering (iBGP or eBGP)
- Redistribution of connected routes per VRF

---

## 🧰 Files

| Script | Purpose |
|--------|---------|
| `01_config_vrfs.py` | Defines VRFs, RDs, RTs on R1 and R2 |
| `02_config_interfaces.py` | Assigns IPs to loopbacks and subinterfaces |
| `03_config_bgp.py` | Configures MP-BGP with VPNv4 and VRF AFs |

---

## 📦 Requirements

- Python 3.9+
- `netmiko`, `python-dotenv`
- Catalyst 8000v routers with SSH access
- .env file with:
  ```env
  USERNAME=admin
  PASSWORD=Cisco123
  ```

---

## 🚀 Usage

```bash
source netauto-venv/bin/activate  # if using a virtual env
python3 01_config_vrfs.py
python3 02_config_interfaces.py
python3 03_config_bgp.py
```

---

## 🧠 Learning Goals

- Understand VRF-to-BGP integration
- Practice route-target policy with MP-BGP
- Build toward MPLS L3 VPN architecture (control plane)

---

## ⚠️ Known Limitations

- No `mpls ip`, `ip cef`, or `ldp` available in this image
- Routes are exchanged via MP-BGP but not labeled or forwarded
- Use real hardware or IOSv for full MPLS support

---

## 🔗 License

MIT

---

Maintained by [Your Name] — for CCIE prep and network automation learning.
