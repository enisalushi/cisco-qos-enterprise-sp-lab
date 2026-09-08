# Routing and MPLS Verification

This document contains operational verification for the routing and MPLS transport used by the QoS lab.

---

## 1. Provider OSPF Adjacencies

The Service Provider underlay runs OSPF Area 0 between:

```text
AMS-PE1 ---- FRA-P1 ---- LON-PE1
```

### AMS-PE1

Verification command:

```text
show ip ospf neighbor
```

Expected adjacency:

```text
FRA-P1 / 3.3.3.3 -> FULL
```

### FRA-P1

Verification command:

```text
show ip ospf neighbor
```

Expected adjacencies:

```text
AMS-PE1 / 2.2.2.2 -> FULL
LON-PE1 / 4.4.4.4 -> FULL
```

### LON-PE1

Verification command:

```text
show ip ospf neighbor
```

Expected adjacency:

```text
FRA-P1 / 3.3.3.3 -> FULL
```

---

## 2. BGP Verification

### HQ Customer Edge

Device:

```text
AMS-HQ-CE1
```

Verification command:

```text
show bgp ipv4 unicast summary
```

Expected eBGP peer:

```text
10.10.12.2
Remote AS: 65100
State: Established
```

---

### Amsterdam Provider Edge

Device:

```text
AMS-PE1
```

Verification command:

```text
show bgp ipv4 unicast summary
```

Expected peers:

```text
10.10.12.1
Remote AS: 65010
eBGP

4.4.4.4
Remote AS: 65100
iBGP
```

---

### London Provider Edge

Device:

```text
LON-PE1
```

Verification command:

```text
show bgp ipv4 unicast summary
```

Expected peers:

```text
2.2.2.2
Remote AS: 65100
iBGP

10.10.45.2
Remote AS: 65020
eBGP
```

---

### Branch Customer Edge

Device:

```text
LON-BR-CE1
```

Verification command:

```text
show bgp ipv4 unicast summary
```

Expected eBGP peer:

```text
10.10.45.1
Remote AS: 65100
State: Established
```

---

## 3. BGP-Free Provider Core

`FRA-P1` intentionally does not run BGP.

Its role is limited to:

- OSPF underlay routing
- MPLS label switching
- LDP

This demonstrates a BGP-free P-router design.

---

## 4. MPLS LDP Verification

MPLS/LDP operates only across the provider core:

```text
AMS-PE1 ---- FRA-P1 ---- LON-PE1
```

### AMS-PE1

```text
show mpls ldp neighbor
```

Expected LDP peer:

```text
3.3.3.3
```

---

### FRA-P1

```text
show mpls ldp neighbor
```

Expected LDP peers:

```text
2.2.2.2
4.4.4.4
```

---

### LON-PE1

```text
show mpls ldp neighbor
```

Expected LDP peer:

```text
3.3.3.3
```

---

## 5. MPLS Forwarding

Verification command on `FRA-P1`:

```text
show mpls forwarding-table
```

The provider core should contain labels for the PE loopbacks.

Example forwarding behavior:

```text
2.2.2.2/32 -> toward AMS-PE1
4.4.4.4/32 -> toward LON-PE1
```

Penultimate Hop Popping can be observed where the outgoing operation is:

```text
Pop Label
```

---

## 6. Customer Route Verification

### AMS-HQ-CE1

Verify Branch Voice network:

```text
show ip route 192.168.120.0
```

The route should be learned through eBGP from `AMS-PE1`.

### LON-BR-CE1

Verify HQ Voice network:

```text
show ip route 192.168.20.0
```

The route should be learned through eBGP from `LON-PE1`.

---

## 7. End-to-End Reachability

### HQ to Branch

Run on `AMS-HQ-CE1`:

```text
ping 192.168.120.1 source 192.168.20.1
```

Expected result:

```text
Success rate is 100 percent
```

### Branch to HQ

Run on `LON-BR-CE1`:

```text
ping 192.168.20.1 source 192.168.120.1
```

Expected result:

```text
Success rate is 100 percent
```

---

## Verification Summary

The following transport components are validated:

- OSPF adjacencies across the provider core
- eBGP between customer and provider edge routers
- iBGP between provider edge routers
- BGP-free provider core operation
- MPLS/LDP neighbor establishment
- MPLS label forwarding
- End-to-end customer route propagation
- Bidirectional customer reachability

This routing and MPLS foundation provides the transport used by the QoS policies demonstrated elsewhere in the project.
