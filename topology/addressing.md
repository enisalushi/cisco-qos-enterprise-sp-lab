# Network Addressing

This document defines the IP addressing, loopbacks, autonomous systems, and service networks used in the Cisco Enterprise / Service Provider QoS lab.

---

# WAN Transit Links

| Link | Device | Interface | IP Address |
|---|---|---|---|
| AMS-HQ-CE1 ↔ AMS-PE1 | AMS-HQ-CE1 | GigabitEthernet0/0 | `10.10.12.1/30` |
| AMS-HQ-CE1 ↔ AMS-PE1 | AMS-PE1 | GigabitEthernet0/0 | `10.10.12.2/30` |
| AMS-PE1 ↔ FRA-P1 | AMS-PE1 | GigabitEthernet0/1 | `10.10.23.1/30` |
| AMS-PE1 ↔ FRA-P1 | FRA-P1 | GigabitEthernet0/0 | `10.10.23.2/30` |
| FRA-P1 ↔ LON-PE1 | FRA-P1 | GigabitEthernet0/1 | `10.10.34.1/30` |
| FRA-P1 ↔ LON-PE1 | LON-PE1 | GigabitEthernet0/0 | `10.10.34.2/30` |
| LON-PE1 ↔ LON-BR-CE1 | LON-PE1 | GigabitEthernet0/1 | `10.10.45.1/30` |
| LON-PE1 ↔ LON-BR-CE1 | LON-BR-CE1 | GigabitEthernet0/0 | `10.10.45.2/30` |

---

# Router IDs

| Device | Loopback0 | Router ID |
|---|---|---|
| AMS-HQ-CE1 | `1.1.1.1/32` | `1.1.1.1` |
| AMS-PE1 | `2.2.2.2/32` | `2.2.2.2` |
| FRA-P1 | `3.3.3.3/32` | `3.3.3.3` |
| LON-PE1 | `4.4.4.4/32` | `4.4.4.4` |
| LON-BR-CE1 | `5.5.5.5/32` | `5.5.5.5` |

---

# Autonomous Systems

| Device | AS Number |
|---|---:|
| AMS-HQ-CE1 | `65010` |
| AMS-PE1 | `65100` |
| FRA-P1 | BGP-free |
| LON-PE1 | `65100` |
| LON-BR-CE1 | `65020` |

---

# Amsterdam HQ Service Networks

| Service | Interface | IP Address | Network | DSCP |
|---|---|---|---|---|
| Users | Loopback10 | `192.168.10.1/24` | `192.168.10.0/24` | Default / 0 |
| Voice | Loopback20 | `192.168.20.1/24` | `192.168.20.0/24` | EF / 46 |
| Video | Loopback30 | `192.168.30.1/24` | `192.168.30.0/24` | AF41 / 34 |
| Critical Data | Loopback40 | `192.168.40.1/24` | `192.168.40.0/24` | AF31 / 26 |
| Bulk Data | Loopback50 | `192.168.50.1/24` | `192.168.50.0/24` | CS1 / 8 |

---

# London Branch Service Networks

| Service | Interface | IP Address | Network | DSCP |
|---|---|---|---|---|
| Users | Loopback110 | `192.168.110.1/24` | `192.168.110.0/24` | Default / 0 |
| Voice | Loopback120 | `192.168.120.1/24` | `192.168.120.0/24` | EF / 46 |
| Video | Loopback130 | `192.168.130.1/24` | `192.168.130.0/24` | AF41 / 34 |
| Critical Data | Loopback140 | `192.168.140.1/24` | `192.168.140.0/24` | AF31 / 26 |
| Bulk Data | Loopback150 | `192.168.150.1/24` | `192.168.150.0/24` | CS1 / 8 |

---

# Routing Domains

## OSPF Area 0

OSPF runs only inside the Service Provider core:

```text
AMS-PE1 ---- FRA-P1 ---- LON-PE1
```

Provider OSPF transit networks:

```text
10.10.23.0/30
10.10.34.0/30
```

Provider loopbacks:

```text
2.2.2.2/32
3.3.3.3/32
4.4.4.4/32
```

Customer-facing links are intentionally excluded from the provider OSPF domain.

---

# BGP Sessions

## eBGP

```text
AMS-HQ-CE1 AS65010
        |
        | 10.10.12.0/30
        |
AMS-PE1 AS65100
```

```text
LON-PE1 AS65100
        |
        | 10.10.45.0/30
        |
LON-BR-CE1 AS65020
```

## iBGP

```text
AMS-PE1 2.2.2.2
        |
        | iBGP AS65100
        |
LON-PE1 4.4.4.4
```

The iBGP session uses the PE loopback interfaces.

---

# MPLS / LDP Domain

MPLS is enabled only on the provider core links:

```text
AMS-PE1 Gi0/1
       |
       | MPLS / LDP
       |
FRA-P1 Gi0/0
FRA-P1 Gi0/1
       |
       | MPLS / LDP
       |
LON-PE1 Gi0/0
```

MPLS is not enabled on:

```text
AMS-PE1 Gi0/0
LON-PE1 Gi0/1
```

because these are customer-facing interfaces.

---

# QoS Bandwidth Model

## Customer CE WAN

Permanent parent shaper:

```text
1 Mbps
```

Child allocation:

| Class | Allocation |
|---|---:|
| Voice | 20% |
| Video | 30% |
| Critical Data | 25% |
| Bulk Data | 5% |
| Default | Remaining bandwidth |

---

## Provider Customer-Facing Egress

Provider egress service rate:

```text
800 kbps
```

Resulting class allocations:

| Class | Allocation |
|---|---:|
| Voice | 160 kbps LLQ |
| Video | 240 kbps minimum |
| Critical Data | 200 kbps minimum |
| Bulk Data | 40 kbps minimum |
| Default | Remaining bandwidth |

---

# Addressing Design Principles

The addressing plan intentionally separates:

- WAN transit links
- Router IDs
- HQ service networks
- Branch service networks
- Provider routing infrastructure

This makes packet flows, QoS classification, routing policy, and troubleshooting easier to understand during the lab.
