# Evidence index and provenance

All six image files are **unaltered copies of screenshots supplied by the user** during the isolated Ubuntu networking exercise on **24 October 2026**. Screenshots are cropped to terminal content as originally supplied. They are not recreations or synthetic terminal images.

| File | Description | Observed key data |
|---|---|---|
| `01-network-topology.png` | Namespace/interface addressing (not a graphical topology diagram) | A=10.10.10.2, router=10.10.10.1 & 10.10.20.1, B=10.10.20.2; interfaces UP |
| `02-routing-tables.png` | Routing configuration before the fault | A gateway 10.10.10.1, B gateway 10.10.20.1, router connected routes |
| `03-connectivity-baseline.png` | Baseline ICMP test | A→B 4/4 and B→A 4/4, 0% loss |
| `04-fault-injection.png` | Intentional incorrect default gateway | via 10.10.10.254; A→B 4/0, 100% loss, host unreachable |
| `05-root-cause-analysis.png` | Isolated cause using ping and routing | A→real gateway 3/3; route to B via 10.10.10.254 |
| `06-connectivity-recovered.png` | Route repaired and reverified | via 10.10.10.1; A→B 4/4 and B→A 4/4, 0% loss |

**Scope:** This evidence demonstrates a lab in network namespaces on one Ubuntu VM. It is not evidence of real switch/server hardware work, Cisco Packet Tracer, VLAN configuration, real production troubleshooting, or long-term uptime. The images do not directly show IP forwarding sysctl settings, though successful inter-subnet pings demonstrate forwarding was operating during tests.
