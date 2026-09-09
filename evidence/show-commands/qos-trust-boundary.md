# QoS Trust Boundary and DSCP Spoofing Validation

This document records the provider QoS trust-boundary tests performed on the customer-facing Provider Edge interfaces.

The objective was to verify that customer DSCP markings are not blindly trusted.

The Provider Edge validates both:

```text
Expected source network
AND
Expected DSCP marking
```

Traffic that fails either condition is classified into `class-default` and remarked to DSCP 0.

---

# Trust Model

## HQ Side

Traffic enters the Service Provider through:

```text
AMS-HQ-CE1
    |
    v
AMS-PE1 Gi0/0
```

Trusted application mappings:

```text
192.168.20.0/24 + EF   -> Trusted Voice
192.168.30.0/24 + AF41 -> Trusted Video
192.168.40.0/24 + AF31 -> Trusted Critical
192.168.50.0/24 + CS1  -> Trusted Bulk
```

Anything else falls into:

```text
class-default
 set dscp default
```

---

# Legitimate Traffic Test

The four legitimate HQ traffic classes were generated using IP SLA.

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

This confirmed that the valid customer traffic satisfied both trust conditions.

---

# Spoofed EF Test

A fake high-priority stream was generated from the Users network:

```text
Source: 192.168.10.1
DSCP: EF 46
```

The packet therefore had a privileged Voice DSCP marking, but originated from the wrong source network.

Expected trusted Voice network:

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
```

The spoofed packets did not increment `SP-VOICE-TRUSTED`.

Instead, they were received by `class-default` and remarked to DSCP 0.

This demonstrated that an endpoint cannot obtain priority Voice treatment simply by setting EF.

---

# Legitimate Voice and Spoofed EF Together

The test was repeated with both:

```text
Legitimate Voice
192.168.20.0/24 + EF
```

and:

```text
Spoofed Voice
192.168.10.0/24 + EF
```

Observed on `AMS-PE1`:

```text
SP-VOICE-TRUSTED
606 packets
```

The legitimate Voice stream matched the trusted class.

The spoofed EF stream continued falling into `class-default`.

This demonstrated that the provider could distinguish between:

```text
Correct source + correct DSCP
```

and:

```text
Incorrect source + privileged DSCP
```

---

# Spoofed AF41 Test

The trust model was also tested using Video traffic.

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

Therefore the spoofed packet failed the source validation:

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
DSCP 0
```

The spoofed AF41 packets were successfully prevented from receiving Video treatment.

---

# Full HQ Trust Policy

The final provider ingress policy uses these trusted classes:

```text
SP-VOICE-TRUSTED
SP-VIDEO-TRUSTED
SP-CRITICAL-TRUSTED
SP-BULK-TRUSTED
```

Each class-map uses `match-all`.

Conceptually:

```text
class-map match-all SP-VOICE-TRUSTED
 match source subnet
 match dscp ef
```

This means both conditions must be true.

---

# Branch Trust Boundary

The same model was implemented on `LON-PE1`.

Trusted Branch mappings:

```text
192.168.120.0/24 + EF   -> Trusted Voice
192.168.130.0/24 + AF41 -> Trusted Video
192.168.140.0/24 + AF31 -> Trusted Critical
192.168.150.0/24 + CS1  -> Trusted Bulk
```

Observed during reverse-direction testing:

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

After correcting the IP SLA responder and allowing the actual UDP-jitter data stream to run, the Video trusted class increased to:

```text
641 packets
734354 bytes
```

This confirmed that the complete Branch-to-HQ data stream was correctly marked and trusted.

---

# Why This Matters

Without a trust boundary, a customer endpoint could mark arbitrary traffic as:

```text
EF
```

and attempt to gain strict-priority treatment.

The provider policy prevents this by validating traffic against the expected customer service network.

The trust decision is therefore:

```text
DSCP alone
!=
trusted traffic
```

Instead:

```text
Source identity
+
Expected QoS marking
=
Trusted traffic
```

---

# Packet Capture Validation

This behavior can also be verified using packet capture.

## Before Provider Trust Enforcement

A spoofed stream can be observed as:

```text
Source: 192.168.10.1
DSCP: EF 46
```

Wireshark filter:

```text
ip.dsfield.dscp == 46
```

## After Provider Trust Enforcement

The same untrusted traffic is expected to appear as:

```text
DSCP 0
```

Wireshark filter:

```text
ip.dsfield.dscp == 0
```

This provides packet-level evidence that the provider does not blindly trust customer QoS markings.

---
## Packet Capture Evidence

### Spoofed EF Before Provider Trust Boundary

The customer Users network sends traffic with an unauthorized EF marking.

![Spoofed EF Before Trust Boundary](../screenshots/spoofed-ef-before-trust-boundary.png)

Observed:

```text
Source: 192.168.10.1
Destination: 192.168.110.1
DSCP: EF (46)
```

### Same Traffic After Provider Trust Boundary

After `AMS-PE1` validates the source network and DSCP combination, the traffic fails the trusted Voice classification and is remarked to Best Effort.

![Spoofed EF After Trust Boundary](../QOS/spoofed-ef-after-trust-boundary.png)

Observed:

```text
Source: 192.168.10.1
Destination: 192.168.110.1
DSCP: Default / CS0 (0)
```

The raw packet captures are available in [`../packet-captures/`](../packet-captures/).
# Key Findings

The tests demonstrated:

- Customer DSCP markings are validated at the provider edge.
- `match-all` can enforce both source-network and DSCP requirements.
- Legitimate Voice, Video, Critical, and Bulk traffic are accepted.
- Spoofed EF traffic is denied privileged Voice classification.
- Spoofed AF41 traffic is denied privileged Video classification.
- Untrusted traffic is remarked to DSCP 0.
- The trust model works in both customer directions.

---

# Conclusion

The QoS trust-boundary implementation successfully protects the provider network from incorrect or intentionally spoofed customer DSCP markings.

The final design follows this principle:

```text
Trust but verify.

Correct source + correct DSCP
-> Preserve QoS treatment

Wrong source and/or wrong DSCP
-> Reset to Best Effort
```
