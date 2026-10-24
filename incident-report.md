# Incident Report — Simulated Incorrect Default Gateway

**Project:** Linux Network Configuration & Fault Isolation (Project 02)  
**Date:** 24 October 2026 (exact lab clock timestamps not captured)  
**Environment:** Ubuntu VM, three isolated Linux network namespaces  
**Classification:** Planned, controlled fault-injection exercise — **not** a real production incident  
**Status:** Resolved, verified by post-repair tests

## Incident summary

Host A (`dco-a`, `10.10.10.2/24`) initially reached Host B (`dco-b`, `10.10.20.2/24`) through router namespace `dco-r`. The operator deliberately changed Host A's default gateway from the legitimate router `10.10.10.1` to unused address `10.10.10.254`. A follow-up ICMP test toward Host B failed with no echo replies. Diagnosis established that the correct router interface was still reachable while Host A's forwarding decision pointed toward the wrong next hop. The original gateway was restored and two-way communication was successfully retested.

## Impact and boundaries

- **Observed impact:** A→B cross-subnet ICMP ping failed in the simulated environment (4 sent, 0 received, 100% packet loss, 3 `Destination Host Unreachable` errors).
- **Observed unaffected path:** Host A could still ping the router's real local interface, `10.10.10.1` (3/3 responses).
- **Not established:** Availability of other protocols or services while the fault was active; no production user impact.
- **Safety:** Change confined to `dco-a` namespace, not the Ubuntu primary network route or RAID storage.

## Evidence-driven timeline (sequence; not real clock times)

| Stage | Action / observation | Evidence |
|---|---|---|
| 1 — Configuration | Three namespaces with IPs 10.10.10.2, 10.10.10.1, 10.10.20.1, 10.10.20.2; interfaces UP | [01](evidence/01-network-topology.png) |
| 2 — Baseline | Valid host gateways and directly connected router routes | [02](evidence/02-routing-tables.png) |
| 3 — Baseline verification | A→B and B→A, each 4/4 replies (0% loss) | [03](evidence/03-connectivity-baseline.png) |
| 4 — Controlled change | `ip -n dco-a route replace default via 10.10.10.254 dev veth-a` | [04](evidence/04-fault-injection.png) |
| 5 — Failure observed | A→B, 4 sent / 0 replies / 100% loss; host unreachable | [04](evidence/04-fault-injection.png) |
| 6 — Diagnosis | Real local gateway ping 3/3; remote route via wrong 10.10.10.254 | [05](evidence/05-root-cause-analysis.png) |
| 7 — Correction | Restored `default via 10.10.10.1 dev veth-a` | [06](evidence/06-connectivity-recovered.png) |
| 8 — Recovery verified | A→B and B→A, each 4/4 replies (0% loss) | [06](evidence/06-connectivity-recovered.png) |

## Root cause

The simulated fault was **misconfigured IPv4 default gateway on Host A** (`10.10.10.254` in place of `10.10.10.1`). For the remote destination `10.10.20.2`, route lookup selected the wrong next hop. Because `10.10.10.254` was not assigned to the lab router or another reachable device, the remote-subnet communication failed. Continued ping reachability to `10.10.10.1` narrowed the fault to the route/next-hop selection rather than proving a failed router or veth physical-equivalent link.

## Corrective action

```bash
sudo ip -n dco-a route replace default via 10.10.10.1 dev veth-a
```

Verification commands:

```bash
sudo ip -n dco-a route show
sudo ip -n dco-a route get 10.10.20.2
sudo ip netns exec dco-a ping -c 4 10.10.20.2
sudo ip netns exec dco-b ping -c 4 10.10.10.2
```

**Observed recovery:** route showed `default via 10.10.10.1`, route lookup selected the correct gateway, and both ping directions returned 4/4 replies and 0% loss. See [Screenshot 06](evidence/06-connectivity-recovered.png).

## Lessons learned

1. Compare reachability to the local gateway with reachability to an off-subnet destination; distinguish local link problems from route-selection problems.
2. Inspect **selected next hop** with `ip route get`, not only general interface status or packet loss.
3. Keep a baseline test and retest the same paths after repair.
4. Record both the failure state and recovery evidence; do not infer repair success from a configuration command alone.

## Documentation provenance

This report is assembled from six screenshots shared by the lab operator in conversation and from the coached sequence of lab commands. Times are not invented. No packet capture, raw console export, physical device, or external incident ticket is claimed.
