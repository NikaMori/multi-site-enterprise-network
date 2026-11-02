# Troubleshooting Log

Issues found while building and verifying the network. Each ticket follows the same format: symptom, impact, diagnosis, root cause, fix, verification and lesson.

## TKT-001: HSRP Split Brain on HR VLAN 20

| Field | Detail |
|---|---|
| Severity | High |
| Affected | HR VLAN 20 (10.1.20.0/24), gateway 10.1.20.1 |
| Status | Resolved |

### Symptom

Both core switches reported themselves as HSRP Active for VLAN 20. HQ-CORE-01 listed the standby router as "unknown", while HQ-CORE-02 listed HQ-CORE-01 as its standby. Communication was one-way.

![Core-01 standby unknown](../screenshots/13-tkt001-01-core-01-standby-unknown.png)
![Core-02 view](../screenshots/13-tkt001-02-core-02-standby.png)

### Impact

Two gateways answered for the same virtual IP and virtual MAC in the HR VLAN, risking intermittent connectivity for HR users. HSRP redundancy for HR was not working as designed.

### Diagnosis

| Step | Command | Finding | Screenshot |
|---|---|---|---|
| 1 | `ping 10.1.20.3` from HQ-CORE-01 | 0/5. Core-01 cannot reach Core-02 on VLAN 20. | 13-tkt001-03 |
| 2 | `show interfaces trunk` on HQ-CORE-02 | VLAN 20 allowed and forwarding on Po1. Trunking ruled out. | 13-tkt001-04 |
| 3 | `show spanning-tree vlan 20` on both cores | All ports forwarding, Po1 is Core-01's root port. STP ruled out. | 13-tkt001-05, 06 |
| 4 | `show access-lists HR_IN` on both cores | Core-01 HR_IN had 1 line. Core-02 had 7. | 13-tkt001-07, 08 |
| 5 | Removed HR_IN from Core-01 VLAN 20 as a test | HSRP converged immediately: Core-01 Standby, Core-02 Active | 13-tkt001-09, 10 |

VLANs 30 and 70 also have inbound ACLs on Core-01 and worked normally, which pointed to something specific to VLAN 20 rather than ACLs in general.

![Ping fail](../screenshots/13-tkt001-03-ping-fail.png)
![Trunk Core-02](../screenshots/13-tkt001-04-trunk-core-02.png)
![STP Core-02](../screenshots/13-tkt001-05-stp-core-02.png)
![STP Core-01](../screenshots/13-tkt001-06-stp-core-01.png)
![HR_IN Core-01 broken](../screenshots/13-tkt001-07-hr-in-core-01-broken.png)
![HR_IN Core-02](../screenshots/13-tkt001-08-hr-in-core-02.png)
![Remove ACL test](../screenshots/13-tkt001-09-remove-acl-test.png)
![Standby after test](../screenshots/13-tkt001-10-standby-after-test.png)

### Root Cause

The HR rules on HQ-CORE-01 had been entered into an ACL named `HR-IN` (hyphen), while the ACL actually applied to VLAN 20 was `HR_IN` (underscore) and contained a single line. The implicit deny at the end of `HR_IN` dropped HQ-CORE-02's HSRP hellos (sent to 224.0.0.2) and its ping replies inbound on VLAN 20. Core-01 never heard Core-02, so it stayed Active.

![Orphan ACL](../screenshots/13-tkt001-11-orphan-acl.png)
*Both `HR-IN` (unused, no matches) and `HR_IN` present on HQ-CORE-01.*

### Fix

1. Rebuilt `HR_IN` on HQ-CORE-01 to match HQ-CORE-02 line for line.
2. Reapplied it inbound on interface VLAN 20.
3. Removed the orphan `HR-IN`.
4. Saved the configuration.

![HR_IN rebuilt](../screenshots/13-tkt001-12-hr-in-rebuilt.png)

### Verification

| Check | Result | Screenshot |
|---|---|---|
| Core-01 reaches Core-02 on VLAN 20 | 5/5 | 13-tkt001-13-ping-success |
| HSRP VLAN 20 | Core-02 Active, Core-01 Standby, ACL applied | 05-hsrp-hq-core-01, 05-hsrp-hq-core-02 |
| ACLs on Core-01 | Clean list, matches Core-02 | 09-acls-hq-core-01 |

![Ping success](../screenshots/13-tkt001-13-ping-success.png)

### Lesson

One character in an ACL name created a silent failure that only showed up as a redundancy problem. On redundant pairs, compare configurations after every change, and use one naming convention everywhere.

## TKT-002: HR PC Cannot Reach 10.1.20.24

| Field | Detail |
|---|---|
| Severity | Low |
| Affected | Testing only |
| Status | Closed, test corrected |

### Symptom

HQ-HR-PC-02 could not ping 10.1.20.24, an address in its own VLAN. Same-VLAN traffic never passes through the gateway or the ACLs, so a failure here would mean a Layer 2 problem.

![HR ping fail](../screenshots/13-tkt002-01-hr-ping-fail.png)

### Diagnosis

| Step | Command | Finding |
|---|---|---|
| 1 | `ipconfig` on HQ-HR-PC-02 | PC is 10.1.20.21 |
| 2 | `ipconfig /all` on HQ-HR-PC-01 | PC is 10.1.20.22 |
| 3 | Compared against the HR VLAN device list | HR hosts are .21, .22 and the printer (.10). Nothing uses 10.1.20.24. |

![HR-PC-02 ipconfig](../screenshots/13-tkt002-02-hr-pc-02-ipconfig.png)

### Root Cause

The test targeted an address that was never assigned. This also affected two earlier ACL deny tests (Guest and Finance to 10.1.20.24), because an ACL block and a missing host both return "Destination host unreachable".

### Fix and Verification

Retested against a real HR host (10.1.20.22). HR to HR succeeds, and the Guest and Finance deny tests were rerun against the real host.

![HR ping success](../screenshots/13-tkt002-03-hr-ping-success.png)

### Lesson

Confirm the target exists before trusting a failed test. A deny test only proves something if the same target is reachable from a permitted source.

## TKT-003: Guest Internet Reported Down

| Field | Detail |
|---|---|
| Severity | Medium |
| Affected | Guest VLAN 70 |
| Status | Closed, no fault found |

### Symptom

A ping to the internet test server 192.0.2.10 failed with 100% loss. The source device was not recorded at the time.

![Initial failure](../screenshots/13-tkt003-01-initial-ping-fail.png)

### Diagnosis

Tested from HQ-GUEST-KIOSK-01, working up the path:

| Step | Test | Result |
|---|---|---|
| 1 | Ping gateway 10.1.70.1 | Success |
| 2 | NAT ACL on both edge routers | ACL 1 permits 10.0.0.0/8, which includes the guest subnet |
| 3 | `tracert 192.0.2.10` | Complete: HQ-CORE-02, HQ-FW-01, HQ-EDGE-01, destination. Hop 4 (ISP router) does not answer traceroute, which is normal. |
| 4 | Ping 192.0.2.10 | 4/4 success |

![Guest gateway](../screenshots/13-tkt003-02-guest-gateway.png)
![Guest tracert](../screenshots/13-tkt003-03-guest-tracert.png)

### Root Cause

Not reproducible. The original test was most likely run from a different device or during a failover test.

### Verification

![Guest internet works](../screenshots/09-test-guest-internet-permit.png)

### Lesson

Record the source device with every test. Working bottom-up (gateway, NAT, path, destination) confirmed the guest path in four steps.
