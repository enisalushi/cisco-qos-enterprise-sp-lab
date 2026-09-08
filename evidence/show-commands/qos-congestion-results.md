# QoS Congestion Test Results

This document records measured QoS behavior during controlled congestion tests performed on both Provider Edge routers.

The goal was to verify:

- Hierarchical shaping
- LLQ behavior
- CBWFQ bandwidth guarantees
- Best Effort treatment
- WRED behavior
- Parent/child drop accounting
- Symmetric QoS in both directions

---

# Test 1 — HQ to Branch

Traffic direction:

```text
AMS-HQ-CE1
    |
    v
AMS-PE1
    |
    v
FRA-P1
    |
    v
LON-PE1
    |
    v
LON-BR-CE1
```

The congestion point was intentionally created on:

```text
LON-PE1 Gi0/1
```

Provider customer-facing service rate:

```text
800 kbps
```

The HQ CE parent shaper was temporarily increased during this test so that the main congestion point occurred at `LON-PE1`.

---

## Parent Shaper Result

```text
Service-policy output: PM-SP-BR-EGRESS-SHAPER

Packets matched:      7598
Packets transmitted:  6249
Packets dropped:      1349

Shape rate:           800000 bps
```

Packet accounting:

```text
7598 - 6249 = 1349 drops
```

---

## Voice — EF

```text
Matched:              1515
Transmitted:           395
LLQ exceed drops:     1120

Priority allocation:  20%
Priority rate:         160 kbps
```

Observation:

The Voice class generated more priority traffic than the LLQ allocation allowed during congestion.

The strict-priority queue therefore protected the remaining classes by dropping excess priority traffic.

```text
1515 - 395 = 1120 drops
```

---

## Video — AF41

```text
Matched:              1515
Transmitted:          1508
Drops:                   7

Bandwidth guarantee:   30%
Minimum bandwidth:     240 kbps
```

Observation:

`bandwidth percent 30` is a minimum bandwidth guarantee, not a rate limit.

The Video class was able to use additional available bandwidth and experienced very few drops.

---

## Critical Data — AF31

```text
Matched:              1515
Transmitted:          1439
Drops:                  76

Bandwidth guarantee:   25%
Minimum bandwidth:     200 kbps
```

Observation:

Critical Data received its configured CBWFQ guarantee but still experienced queue pressure during the burst.

---

## Bulk Data — CS1

```text
Matched:              1491
Transmitted:          1491
Drops:                   0

Bandwidth guarantee:    5%
Minimum bandwidth:      40 kbps

WRED mean queue depth:   5 packets
WRED minimum threshold: 10 packets
WRED maximum threshold: 30 packets
Random drops:            0
Tail drops:              0
```

Observation:

The Bulk class had only a 40 kbps minimum guarantee, but it transmitted all packets because bandwidth guarantees are not rate caps.

The class borrowed unused bandwidth.

WRED did not begin dropping because the average queue depth remained below the configured minimum threshold.

---

## Best Effort

```text
Matched:              1562
Transmitted:          1416
Drops:                 146
```

Best Effort had no explicit bandwidth guarantee and experienced congestion after the explicitly scheduled traffic classes competed for the shaped link.

---

## Drop Accounting

The child-policy drops were:

```text
Voice:       1120
Video:          7
Critical:      76
Bulk:           0
Best Effort:  146
-----------------
Total:       1349
```

Parent shaper:

```text
Total drops: 1349
```

Therefore:

```text
Child queue drops = Parent shaper drops
```

This confirms that the congestion losses occurred inside the child QoS queues beneath the 800 kbps parent shaper.

---

# Test 2 — Branch to HQ

Traffic direction:

```text
LON-BR-CE1
    |
    v
LON-PE1
    |
    v
FRA-P1
    |
    v
AMS-PE1
    |
    v
AMS-HQ-CE1
```

The congestion point was intentionally created on:

```text
AMS-PE1 Gi0/0
```

Provider service rate:

```text
800 kbps
```

The Branch CE parent shaper was temporarily increased so that `AMS-PE1` became the primary congestion point.

---

## Parent Shaper Result

```text
Service-policy output: PM-SP-HQ-EGRESS-SHAPER

Packets matched:      5592
Packets transmitted:  4589
Packets dropped:      1003

Shape rate:           800000 bps
```

Packet accounting:

```text
5592 - 4589 = 1003 drops
```

---

## Voice — EF

```text
Matched:              1111
Transmitted:           286
LLQ exceed drops:      825

Priority allocation:   20%
Priority rate:          160 kbps
```

Observation:

The reverse-direction test reproduced the same LLQ protection behavior.

Excess EF traffic was dropped once the priority traffic exceeded the configured LLQ allowance during congestion.

---

## Video — AF41

```text
Matched:              1111
Transmitted:          1105
Drops:                   6

Bandwidth guarantee:   30%
Minimum bandwidth:     240 kbps
```

---

## Critical Data — AF31

```text
Matched:              1111
Transmitted:          1047
Drops:                  64

Bandwidth guarantee:   25%
Minimum bandwidth:     200 kbps
```

---

## Bulk Data — CS1

```text
Matched:              1111
Transmitted:          1111
Drops:                   0

Bandwidth guarantee:    5%
Minimum bandwidth:      40 kbps

WRED mean queue depth:   8 packets
WRED minimum threshold: 10 packets
WRED maximum threshold: 30 packets
Random drops:            0
Tail drops:              0
```

Observation:

Again, the 40 kbps bandwidth allocation behaved as a minimum guarantee rather than a maximum rate.

WRED remained inactive because the calculated average queue depth remained below the minimum threshold.

---

## Best Effort

```text
Matched:              1148
Transmitted:          1040
Drops:                 108
```

---

## Drop Accounting

```text
Voice:        825
Video:          6
Critical:      64
Bulk:           0
Best Effort:  108
-----------------
Total:       1003
```

Parent shaper:

```text
Total drops: 1003
```

Therefore:

```text
Child queue drops = Parent shaper drops
```

The same behavior was reproduced in the opposite direction.

---

# Bandwidth Guarantee vs Rate Cap Experiment

The Bulk class was also tested using two different configurations.

## Test A — Bandwidth Guarantee

```text
bandwidth percent 5
```

At an 800 kbps parent:

```text
Minimum guaranteed bandwidth = 40 kbps
```

Observed result:

```text
1491 packets matched
1491 packets transmitted
0 drops
```

The class borrowed additional unused bandwidth.

---

## Test B — Per-Class Shaper

The Bulk class was temporarily changed to:

```text
shape average 40000
```

Observed result:

```text
303 packets matched
213 packets transmitted
90 packets dropped

Target shape rate: 40000 bps
```

Packet accounting:

```text
303 - 213 = 90 drops
```

This demonstrated the important difference:

```text
bandwidth percent 5
= minimum bandwidth guarantee
= traffic may borrow unused capacity
```

versus:

```text
shape average 40000
= approximately 40 kbps average rate cap
```

---

# Key Findings

The congestion tests demonstrated:

- LLQ provides strict-priority treatment but protects the link from excessive priority traffic.
- `bandwidth percent` defines a minimum guarantee, not a maximum rate.
- CBWFQ classes can borrow unused bandwidth.
- Best Effort receives remaining capacity and can experience significant congestion.
- WRED uses average queue depth rather than instantaneous queue depth.
- A parent shaper creates the congestion point required for child queueing behavior to become visible.
- Child drop counters can be reconciled exactly with parent shaper drops.
- The same QoS behavior was successfully reproduced in both traffic directions.

---

# Conclusion

The lab successfully demonstrated end-to-end QoS behavior across an Enterprise / Service Provider topology.

QoS treatment was not validated only through configuration.

It was verified using:

- IP SLA traffic generation
- MQC counters
- Queue statistics
- LLQ drop counters
- WRED statistics
- Parent/child drop accounting
- Bidirectional congestion testing
