# QoS Trust Boundary and DSCP Spoofing Validation

This document records the QoS trust-boundary tests performed on the customer-facing Provider Edge interfaces.

The objective was to verify that customer DSCP markings are **not blindly trusted**.

The Provider Edge validates both:

```text
Expected source network
AND
Expected DSCP marking
```

Traffic that fails either condition is classified into `class-default` and remarked to DSCP 0.

---

## 1. Trust Model

### HQ Side

Traffic enters the Service Provider through:

```text
AMS-HQ-CE1
    |
    v
AMS-PE1 Gi0/0
```

Trusted HQ mappings:

```text
192.168.20.0/24 + EF   -> Trusted Voice
192.168.30.0/24 + AF41 -> Trusted Video
192.168.40.0/24 + AF31 -> Trusted Critical
192.168.50.0/24 + CS1  -> Trusted Bulk
```

Traffic that does not match one of these trusted combinations falls into:

```text
class-default
 set dscp default
```

The provider therefore does not make a trust decision based on DSCP alone.

---

## 2. Legitimate Traffic Test

The four legitimate HQ traffic classes were generated using Cisco IP SLA.

Observed on `AMS-PE1`:

```text
SP-VOICE-TRUSTED
408 packets

SP-VIDEO-TRUSTED
1212 packets

SP-CRITICAL-TRUSTED
1212 packets

SP-BULK-TRUSTED
1212 packets
```

Only a small amount of unmatched traffic reached `class-default`.

This confirmed that valid customer traffic satisfied both conditions:

```text
Correct source network
+
Correct DSCP
=
Trusted traffic
```

---

## 3. Spoofed EF Test

A fake high-priority stream was generated from the HQ Users network.

```text
Source: 192.168.10.1
Destination: 192.168.110.1
DSCP: EF (46)
```

The packet carried a privileged Voice marking, but originated from the wrong source network.

Expected Voice source network:

```text
192.168.20.0/24
```

Actual source:

```text
192.168.10.1
```

Trust decision:

```text
192.168.10.1 + EF
        |
        X
SP-VOICE-TRUSTED
        |
        v
class-default
        |
        v
set dscp default
        |
        v
DSCP 0
```

The spoofed packets did not qualify for `SP-VOICE-TRUSTED`.

Instead, they were classified into `class-default` and remarked to DSCP 0.

This demonstrates that a customer endpoint cannot obtain trusted Voice treatment simply by marking traffic as EF.

---

## 4. Legitimate Voice and Spoofed EF Together

The test was repeated with both legitimate and spoofed Voice traffic running.

Legitimate Voice:

```text
192.168.20.0/24 + EF
```

Spoofed Voice:

```text
192.168.10.0/24 + EF
```

Observed on `AMS-PE1`:

```text
SP-VOICE-TRUSTED
606 packets
```

The legitimate Voice stream matched the trusted class.

The spoofed EF stream continued to fall into `class-default`.

This demonstrated that the Provider Edge could distinguish between:

```text
Correct source + correct DSCP
```

and:

```text
Incorrect source + privileged DSCP
```

---

## 5. Spoofed AF41 Test

The same trust model was tested with Video traffic.

Legitimate Video:

```text
Source: 192.168.30.1
DSCP: AF41
```

Spoofed Video:

```text
Source: 192.168.10.1
DSCP: AF41
```

The trusted Video class requires:

```text
192.168.30.0/24
AND
DSCP AF41
```

The spoofed packet therefore failed source-network validation:

```text
192.168.10.1 + AF41
        |
        X
SP-VIDEO-TRUSTED
        |
        v
class-default
        |
        v
set dscp default
        |
        v
DSCP 0
```

The spoofed AF41 traffic was successfully prevented from receiving trusted Video treatment.

---

## 6. Final HQ Trust Policy

The final provider ingress policy uses:

```text
SP-VOICE-TRUSTED
SP-VIDEO-TRUSTED
SP-CRITICAL-TRUSTED
SP-BULK-TRUSTED
```

Each trusted class-map uses `match-all`.

Conceptually:

```text
class-map match-all SP-VOICE-TRUSTED
 match access-group name ACL-SP-VOICE-TRUSTED
 match dscp ef
```

Both conditions must therefore match before traffic enters the trusted class.

The corresponding policy behavior is:

```text
Trusted class
-> preserve existing DSCP

class-default
-> set dscp default
```

---

## 7. Branch Trust Boundary

The same trust model was implemented on `LON-PE1` for Branch-to-Provider traffic.

Trusted Branch mappings:

```text
192.168.120.0/24 + EF   -> Trusted Voice
192.168.130.0/24 + AF41 -> Trusted Video
192.168.140.0/24 + AF31 -> Trusted Critical
192.168.150.0/24 + CS1  -> Trusted Bulk
```

During the initial validation, the four trusted classes each recorded:

```text
SP-BR-VOICE-TRUSTED
71 packets

SP-BR-VIDEO-TRUSTED
71 packets

SP-BR-CRITICAL-TRUSTED
71 packets

SP-BR-BULK-TRUSTED
71 packets
```

At that stage, the IP SLA responder was not yet active, so these counters primarily reflected the initial SLA/control traffic.

After enabling the IP SLA responder and allowing the actual UDP-jitter data stream to run, the Video class increased to:

```text
SP-BR-VIDEO-TRUSTED
641 packets
734354 bytes
```

This confirmed that the actual Branch-to-HQ data traffic was correctly marked and accepted by the provider trust policy.

---

## 8. Packet Capture Evidence

Packet captures were used to validate the trust-boundary behavior directly on the wire.

### 8.1 Legitimate Voice — EF 46

A legitimate HQ Voice flow was captured between `AMS-HQ-CE1` and `AMS-PE1`.

![Legitimate Voice EF DSCP 46](../screenshots/voice-ef-dscp46-wireshark.png)

Observed:

```text
Source: 192.168.20.1
Destination: 192.168.120.1
UDP: 10020 -> 20020
DSCP: Expedited Forwarding (46)
```

This verifies that the CE marked legitimate Voice traffic as EF before it entered the provider.

Raw capture:

[`voice-ef-dscp46.pcap`](../packet-captures/voice-ef-dscp46.pcap)

---

### 8.2 Spoofed EF Before Provider Trust Boundary

A Users-network stream was intentionally marked as EF.

![Spoofed EF Before Trust Boundary](../screenshots/spoofed-ef-before-trust-boundary.png)

Observed:

```text
Source: 192.168.10.1
Destination: 192.168.110.1
UDP: 10060 -> 20060
DSCP: Expedited Forwarding (46)
```

This proves that the customer sent privileged EF marking into the Provider Edge.

Raw capture:

[`spoofed-ef-before-trust-boundary.pcap`](../packet-captures/spoofed-ef-before-trust-boundary.pcap)

---

### 8.3 Same Traffic After Provider Trust Boundary

The same flow was captured again after it had crossed the provider network.

![Spoofed EF After Trust Boundary](../screenshots/spoofed-ef-after-trust-boundary.png)

Observed:

```text
Source: 192.168.10.1
Destination: 192.168.110.1
UDP: 10060 -> 20060
DSCP: Default / CS0 (0)
```

The source and destination remained unchanged, but the DSCP value changed:

```text
Before AMS-PE1:
EF 46

After AMS-PE1:
CS0 / DSCP 0
```

This provides packet-level proof that the provider trust boundary rejected the unauthorized EF marking and remarked the flow to Best Effort.

Raw capture:

[`spoofed-ef-after-trust-boundary.pcap`](../packet-captures/spoofed-ef-after-trust-boundary.pcap)

---

## 9. Before-and-After Comparison

| Property | Before Trust Boundary | After Trust Boundary |
|---|---|---|
| Source | `192.168.10.1` | `192.168.10.1` |
| Destination | `192.168.110.1` | `192.168.110.1` |
| UDP Flow | `10060 -> 20060` | `10060 -> 20060` |
| DSCP | EF / 46 | CS0 / 0 |
| Trust Result | Untrusted EF arrives | Remarked to Best Effort |

Packet behavior:

```text
Customer Users Network
192.168.10.1
DSCP EF 46
      |
      v
AMS-PE1
      |
      | Source is NOT 192.168.20.0/24
      |
      X SP-VOICE-TRUSTED
      |
      v
class-default
      |
      | set dscp default
      v
Provider Network
      |
      v
LON-BR-CE1
DSCP 0
```

---

## 10. Why This Matters

Without a trust boundary, a customer endpoint could mark arbitrary traffic as:

```text
EF
```

and attempt to request privileged QoS treatment.

The provider prevents this by validating the expected relationship between:

```text
Source network
+
DSCP marking
```

The trust decision is therefore not:

```text
DSCP = EF
-> automatically trusted
```

Instead:

```text
Expected source network
+
Expected DSCP
=
Trusted class
```

Anything else is treated as Best Effort.

---

## 11. Key Findings

The tests demonstrated:

- Customer DSCP markings are validated at the provider edge.
- `match-all` requires both source-network and DSCP conditions to match.
- Legitimate Voice, Video, Critical, and Bulk traffic can retain their expected QoS markings.
- Spoofed EF traffic is denied trusted Voice classification.
- Spoofed AF41 traffic is denied trusted Video classification.
- Untrusted customer traffic is remarked to DSCP 0.
- Wireshark confirms EF 46 before the trust boundary and CS0 after enforcement.
- The same trust model is implemented on both customer-facing Provider Edge routers.
- Router policy counters and packet captures validate the same behavior from two different perspectives.

---

## 12. Conclusion

The QoS trust-boundary implementation successfully prevents incorrect or intentionally spoofed customer DSCP markings from automatically receiving privileged QoS treatment.

The final design follows this principle:

```text
Correct source + correct DSCP
-> Preserve trusted QoS treatment

Wrong source and/or wrong DSCP
-> class-default
-> Reset to DSCP 0
-> Best Effort
```

The combination of MQC counters and Wireshark packet captures provides end-to-end evidence that the policy behaves as designed.
