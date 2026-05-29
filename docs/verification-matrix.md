# Verification Matrix — Secure Campus Network

Complete end-to-end test plan validating every security policy in the project. Run each test from the specified source device and verify the expected result.

---

## Section A — Basic Connectivity (Foundation)

| # | From | Command | Expected | Why |
|---|---|---|---|---|
| A1 | HR-PC1 | `ping 172.16.10.11` | ✅ Reply | Same-VLAN switching |
| A2 | HR-PC1 | `ping 172.16.30.10` | ✅ Reply | Inter-VLAN routing (HR→IT allowed) |
| A3 | FIN-PC1 | `ping 172.16.30.10` | ✅ Reply | Inter-VLAN routing (FIN→IT allowed) |
| A4 | IT-PC1 | `ping 172.16.30.100` | ✅ Reply | IT to FileServer (same VLAN) |
| A5 | HR-PC1 | `ping 10.1.1.1` | ✅ Reply | WAN serial link reachable |

## Section B — HR Department Restrictions

| # | From | Command | Expected | Why |
|---|---|---|---|---|
| B1 | HR-PC1 | `ping 172.16.20.10` | ❌ Fail | HR cannot initiate to Finance |
| B2 | HR-PC1 | `ping 172.16.10.1` | ❌ Fail | HR cannot ping own gateway |
| B3 | FIN-PC1 | `ping 172.16.10.10` | ✅ Reply | Finance CAN ping HR (asymmetric policy) |
| B4 | HR-PC1 | Browser: `http://203.0.113.10` | ✅ Page loads | HR can reach DMZ web |
| B5 | HR-PC1 | `ping 203.0.113.10` | ❌ Fail | ACL allows only TCP/80,443 — not ICMP |

## Section C — Finance & IT Restrictions

| # | From | Command | Expected | Why |
|---|---|---|---|---|
| C1 | FIN-PC1 | `ping 172.16.20.1` | ❌ Fail | Finance cannot ping own gateway |
| C2 | IT-PC1 | `ping 172.16.10.1` | ✅ Reply | IT can ping any gateway |
| C3 | IT-PC1 | `ping 172.16.20.1` | ✅ Reply | IT can ping any gateway |
| C4 | IT-PC1 | `ping 172.16.30.1` | ✅ Reply | IT can ping any gateway |
| C5 | IT-PC1 | `ping 172.16.10.10` | ✅ Reply | IT unrestricted |

## Section D — Perimeter Security

| # | From | Command | Expected | Why |
|---|---|---|---|---|
| D1 | Internet-Host | `ping 203.0.113.10` | ❌ Fail | ICMP not permitted to web server |
| D2 | Internet-Host | Browser: `http://203.0.113.10` | ✅ Page loads | HTTP permitted to web server |
| D3 | Internet-Host | `ping 172.16.10.10` | ❌ Fail | Internet cannot reach internal subnets |
| D4 | Internet-Host | `ping 172.16.30.100` | ❌ Fail | Internet cannot reach internal subnets |
| D5 | Internet-Host | `ping 10.1.1.1` | ❌/✅ | No NAT — depends on routing |

## Section E — Management Access

| # | From | Command | Expected | Why |
|---|---|---|---|---|
| E1 | IT-PC1 | `ssh -l admin 172.16.30.1` | ✅ Login prompt | SSH works with local user |
| E2 | IT-PC1 | `telnet 172.16.30.1` | ❌ Refused | Telnet disabled |
| E3 | IT-PC1 | `ssh -l admin 172.16.20.1` | ✅ Login | IT can SSH to FIN gateway |
| E4 | HR-PC1 | `ssh -l admin 172.16.10.1` | ❌ Fail | HR can't ping own GW; SSH likely blocked |

## Section F — Port Security Verification

| # | Action | Expected | Verification Command |
|---|---|---|---|
| F1 | Initial state of Fa0/1 on SW-HR | Secure-up, 1 MAC learned | `show port-security interface fa0/1` |
| F2 | List all sticky MACs | HR-PC1's MAC tied to Fa0/1 | `show port-security address` |
| F3 | Plug a third device into Fa0/1 with HR-PC1 still connected | Violation counter increments | `show port-security interface fa0/1` |
| F4 | Check unused port Fa0/10 | administratively down | `show interfaces status` |

## Section G — Configuration Verification

| # | Device | Command | Expected |
|---|---|---|---|
| G1 | R-Core | `show ip access-lists` | HR-IN, FIN-IN, IT-IN all present |
| G2 | R-Core | `show ip interface gi0/0.10 \| include access list` | HR-IN applied inbound |
| G3 | R-Edge | `show ip access-lists` | FROM-INTERNET present |
| G4 | R-Edge | `show ip interface gi0/0 \| include access list` | FROM-INTERNET applied inbound |
| G5 | Any switch | `show vlan brief` | VLANs 10, 20, 30 present |
| G6 | Any switch | `show interfaces trunk` | Gi0/1 (and Gi0/2 where applicable) trunking |
| G7 | Any device | `show running-config \| include username` | admin user exists |
| G8 | Any device | `show ip ssh` | SSH version 2 enabled |
| G9 | Any device | `show running-config \| include password-encryption` | service password-encryption enabled |

---

## Test Procedure

1. Open Packet Tracer file
2. Wait for all interfaces to come up (~30 seconds after open)
3. Run Section A first — if any A test fails, do not proceed; foundation is broken
4. Run Sections B, C in any order
5. Run Section D from Internet-Host
6. Run Section E from IT-PC1
7. Run Section F on SW-HR (sample switch)
8. Run Section G commands on appropriate devices

## My Verification Results

```
DATE: 2026-05-29
TESTER: Kwadwo

Section A — Basic Connectivity (Foundation)
  A1 ✅  A2 ✅  A3 ✅  A4 ✅  A5 ✅
  → All foundation tests pass. Inter-VLAN routing working as designed.

Section B — HR Department Restrictions
  B1 ❌  B2 ❌  B3 ✅  B4 ✅  B5 ❌
  → HR cannot initiate to Finance or ping own gateway.
    Finance CAN ping HR (asymmetric policy via explicit echo-reply rule).
    HR can browse web server but cannot ping it (ACL permits TCP/80,443 only).

Section C — Finance & IT Restrictions
  C1 ❌  C2 ✅  C3 ✅  C4 ✅  C5 ✅
  → Finance protected from own gateway pings. IT unrestricted as designed.

Section D — Perimeter Security
  D1 ❌  D2 ✅  D3 ❌  D4 ❌  D5 ❌
  → Internet can reach web server on HTTP/HTTPS only.
    All internal subnets unreachable from the simulated internet.
    D5 fails because no NAT is configured (intentional — outside project scope).

Section E — Management Access
  E1 ✅  E2 ❌  E3 ✅  E4 ❌
  → SSH works with local user authentication. Telnet completely refused.
    E4 fails because HR-IN ACL blocks ICMP/management traffic to gateway.

Section F — Port Security Verification
  F1 ✅  F2 ✅  F3 (skipped — non-destructive testing)  F4 ✅
  → Sticky MACs learned, ports in Secure-up state.
    Unused ports administratively down.

Section G — Configuration Verification
  G1 ✅  G2 ✅  G3 ✅  G4 ✅  G5 ✅  G6 ✅  G7 ✅  G8 ✅  G9 ✅
  → All ACLs present and correctly applied.
    VLANs configured, trunks carrying 10/20/30.
    SSH v2 enabled, local users in place, password-encryption active.

Overall: PASS — all policy rules enforced as designed.

Notes:
- The ❌ marks throughout sections B, C, and D are EXPECTED failures —
  they represent the security policy working correctly (blocked traffic).
- The stateless-ACL asymmetric policy in HR-IN required explicit
  ICMP echo / echo-reply rules to make Finance → HR pings succeed
  while still blocking HR → Finance initiation.
- DMZ web server is reachable from internet on HTTP/HTTPS but not pingable —
  a real-world hardening practice that the ACL design enforces by default.
- No NAT configured by design; this would be the next extension
  to make outbound internet access work for internal subnets.
```

## Common Failure Modes

| Symptom | Likely Cause | Fix |
|---|---|---|
| All A tests fail | VLAN misconfiguration | `show vlan brief` on each switch; verify ports in correct VLAN |
| A1 works, A2 fails | Inter-VLAN routing broken | Check R-Core sub-interfaces with `show ip int brief` |
| Cross-VLAN partial fail | Trunk missing VLAN | `show interfaces trunk` — verify allowed VLANs |
| B3 fails (FIN→HR blocked) | HR-IN ACL too restrictive | Verify echo-reply permit line exists |
| D1 works (should fail) | Missing ACL or wrong direction | `show ip int gi0/0 \| inc access` — confirm FROM-INTERNET applied inbound |
| E1 fails | SSH not configured | Verify domain-name set, RSA keys generated, vty config has `transport input ssh` |
| F1 shows Secure-down | Port-security violation already triggered | Shutdown/no-shutdown the port |
