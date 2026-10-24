# Linux Network Configuration & Fault Isolation — Project 02

**Status: Completed (isolated, virtual lab) | Date: 24 October 2026 | Environment: Ubuntu VM / Linux network namespaces**

## Summary

Built a simulated two-subnet IPv4 network using three Linux network namespaces and two virtual Ethernet (veth) pairs. Configured host addressing, default gateways, and Linux forwarding; validated two-way inter-subnet connectivity. Then deliberately misconfigured Host A's default gateway, observed failed ICMP communication, isolated the problem through local gateway reachability and route inspection, restored the correct gateway, and verified two-way connectivity had recovered.

**Outcome:** Baseline **4/4 replies each direction**; after fault injection **0/4 replies from Host A to Host B** (100% loss); after repair **4/4 replies each direction** (0% loss).

This is a **controlled, non-production Linux networking lab**; no Cisco Packet Tracer, Cisco IOS, VLAN trunking, physical switching, or actual production incident is claimed.

## Network topology

```text
  namespace dco-a                 namespace dco-r                    namespace dco-b
  Host A                          Linux Router                       Host B
  10.10.10.2/24                   10.10.10.1/24  | 10.10.20.1/24     10.10.20.2/24
  veth-a <=======================> veth-ra       | veth-rb <========> veth-b
             10.10.10.0/24                           10.10.20.0/24

  Host A default gateway: 10.10.10.1
  Host B default gateway: 10.10.20.1
```

The virtual network was isolated in network namespaces in one Ubuntu VM. Its simulated connectivity did not require editing the VM host's primary network interface or existing RAID1 disks. Note that the lab configuration is **not necessarily persistent across VM reboot**.

## Evidence and observations

| # | Screenshot | What it proves |
|---|---|---|
| 01 | [IP addressing / topology configuration](evidence/01-network-topology.png) | Both host interfaces and both router interfaces UP; correct IPv4 addresses |
| 02 | [Routing tables](evidence/02-routing-tables.png) | Both hosts' correct default gateways; router's two connected routes |
| 03 | [Baseline inter-subnet connectivity](evidence/03-connectivity-baseline.png) | A→B and B→A both 4 sent, 4 received, 0% loss |
| 04 | [Controlled fault injection](evidence/04-fault-injection.png) | Host A default via unused 10.10.10.254; A→B 0 replies, 100% loss |
| 05 | [Root-cause diagnosis](evidence/05-root-cause-analysis.png) | Host A can still ping real router 10.10.10.1 (3/3); route to B instead via wrong next hop 10.10.10.254 |
| 06 | [Recovery verification](evidence/06-connectivity-recovered.png) | Gateway restored to 10.10.10.1; A→B and B→A both 4/4, 0% loss |

**Screenshot 01 is terminal evidence of interface/IP configuration**, rather than a rendered graphical network topology. All six screenshots are preserved copies of screenshots supplied by the lab operator; they are not generated terminal imagery.

### Key terminal results

**Baseline routing:**

```text
# dco-a
 default via 10.10.10.1 dev veth-a
 10.10.10.0/24 dev veth-a proto kernel scope link src 10.10.10.2
# dco-r
 10.10.10.0/24 dev veth-ra proto kernel scope link src 10.10.10.1
 10.10.20.0/24 dev veth-rb proto kernel scope link src 10.10.20.1
# dco-b
 default via 10.10.20.1 dev veth-b
 10.10.20.0/24 dev veth-b proto kernel scope link src 10.10.20.2
```

**Before fault:** `ping -c 4` from A→B and B→A: **4/4 responses each**.

**Fault injected:**

```bash
sudo ip -n dco-a route replace default via 10.10.10.254 dev veth-a
sudo ip netns exec dco-a ping -c 4 -W 1 10.10.20.2
```

```text
From 10.10.10.2 icmp_seq=1 Destination Host Unreachable
From 10.10.10.2 icmp_seq=2 Destination Host Unreachable
From 10.10.10.2 icmp_seq=3 Destination Host Unreachable
4 packets transmitted, 0 received, +3 errors, 100% packet loss, time 3105ms
```

**Diagnosis:** Ping Host A→router IP `10.10.10.1`: **3/3 replies, 0% loss**. But `ip -n dco-a route get 10.10.20.2` selected `via 10.10.10.254 dev veth-a`.

**Repair:**

```bash
sudo ip -n dco-a route replace default via 10.10.10.1 dev veth-a
```

**After repair:** Route to Host B selected `via 10.10.10.1` again; A→B and B→A both **4 transmitted, 4 received, 0% loss**.

## Analysis

The fault was an **incorrect default next hop on Host A**, rather than a complete loss of local reachability. An unused gateway IP (`10.10.10.254`) caused remote-subnet traffic to be sent toward a nonresponsive next hop. The local router's interface (`10.10.10.1`) remained reachable. Restoring the correct route recovered the tested ICMP flow.

See [Incident report](incident-report.md) for incident workflow, [Commands](commands.md) for a lab runbook, and [Evidence inventory](evidence/README.md) for provenance.

## Scope and limitations

- All hosts/router are **network namespaces inside one VM**; no physical hardware was involved.
- The lab did **not** test Cisco VLANs, trunking, IOS, DNS, firewall policy, NAT, real ISP networks, or production remediation.
- Screenshots show observed state, successful ICMP tests, and routing; they do not provide a full raw packet capture or independently record the IP forwarding sysctl state. Operational forwarding is supported by the successful inter-subnet pings, not by a sysctl screenshot.
- ICMP checks covered four packets per direction per baseline/recovery test; these are point-in-time checks, not latency SLAs or proof of prolonged availability.
- Commands in the runbook were assembled from the coached lab steps and verified results; they are not an independently exported shell history.

## CV-ready description

**Linux Network Configuration & Fault Isolation (Ubuntu VM)** — Built a two-subnet virtual network with Linux network namespaces, veth pairs, IP addressing, and routing. Simulated a faulty default gateway; diagnosed loss of connectivity using ping and routing tables, then restored successful two-way inter-subnet communication (4/4 replies in each recovery test). Documented six screenshots and an incident report.
