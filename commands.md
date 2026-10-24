# Commands — Project 02 Linux Network Fault Isolation

This is a **reproducible lab runbook assembled from the coached procedure and observed screenshots**, not a verbatim machine-exported command log. **For an empty lab only**: do not run setup twice on existing namespaces without checking state first. Requires administrator privileges. All example changes target only purpose-created virtual namespaces/interfaces.

## Setup — three namespaces / two veth pairs

```bash
sudo ip netns add dco-a
sudo ip netns add dco-r
sudo ip netns add dco-b

sudo ip link add veth-a type veth peer name veth-ra
sudo ip link add veth-b type veth peer name veth-rb
sudo ip link set veth-a netns dco-a
sudo ip link set veth-ra netns dco-r
sudo ip link set veth-b netns dco-b
sudo ip link set veth-rb netns dco-r

sudo ip -n dco-a addr add 10.10.10.2/24 dev veth-a
sudo ip -n dco-r addr add 10.10.10.1/24 dev veth-ra
sudo ip -n dco-r addr add 10.10.20.1/24 dev veth-rb
sudo ip -n dco-b addr add 10.10.20.2/24 dev veth-b

sudo ip -n dco-a link set veth-a up
sudo ip -n dco-a link set lo up
sudo ip -n dco-r link set veth-ra up
sudo ip -n dco-r link set veth-rb up
sudo ip -n dco-r link set lo up
sudo ip -n dco-b link set veth-b up
sudo ip -n dco-b link set lo up

sudo ip -n dco-a route replace default via 10.10.10.1 dev veth-a
sudo ip -n dco-b route replace default via 10.10.20.1 dev veth-b
sudo ip netns exec dco-r sysctl -w net.ipv4.ip_forward=1
```

## Verification — addressing, routing, normal ping

```bash
sudo ip -n dco-a -br addr
sudo ip -n dco-r -br addr
sudo ip -n dco-b -br addr
sudo ip -n dco-a route show
sudo ip -n dco-r route show
sudo ip -n dco-b route show
sudo ip netns exec dco-a ping -c 4 10.10.20.2
sudo ip netns exec dco-b ping -c 4 10.10.10.2
```

## Controlled fault — wrong Host A default gateway

```bash
sudo ip -n dco-a route replace default via 10.10.10.254 dev veth-a
sudo ip -n dco-a route show
sudo ip netns exec dco-a ping -c 4 -W 1 10.10.20.2
```

## Troubleshooting — find wrong next hop

```bash
sudo ip netns exec dco-a ping -c 3 10.10.10.1
sudo ip -n dco-a route get 10.10.20.2
sudo ip -n dco-a route show
```

## Recovery — restore correct gateway, retest

```bash
sudo ip -n dco-a route replace default via 10.10.10.1 dev veth-a
sudo ip -n dco-a route show
sudo ip -n dco-a route get 10.10.20.2
sudo ip netns exec dco-a ping -c 4 10.10.20.2
sudo ip netns exec dco-b ping -c 4 10.10.10.2
```

## Optional teardown — only if no longer needed

Deleting these namespaces removes their isolated virtual links and route configurations. **Do not run unless intentionally cleaning up** and do not touch the system disk or RAID1.

```bash
sudo ip netns del dco-a
sudo ip netns del dco-r
sudo ip netns del dco-b
sudo ip netns list
```

Network namespaces generally do not survive reboot unless explicitly recreated. No cleanup has been verified in the submitted screenshots.
