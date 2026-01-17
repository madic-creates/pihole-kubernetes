# Pi-hole with DHCP Helper (Rootless)

This example shows how to deploy Pi-hole with DHCP functionality **without** using `hostNetwork: true` on the Pi-hole pod. Instead, a lightweight DHCP helper container handles the Layer 2 broadcast requirements.

**Rootless Operation**: This configuration runs the DHCP helper as an unprivileged user (nobody/65534) with all capabilities dropped, improving security.

## Problem

DHCP operates at OSI Layer 2 using broadcast messages. In Kubernetes, pods typically cannot receive Layer 2 broadcasts because they are isolated in their own network namespace. The common workaround is to use `hostNetwork: true`, but this can cause issues:

- **Port conflicts**: If any port required by Pi-hole FTL (e.g., UDP port 53, 67, or others) is already in use on the host, FTL will fail to start with the error `pihole-FTL: no process found`
- **Security**: Host networking reduces container isolation
- **Flexibility**: Only one pod can use a specific port on a node

## Solution

Use a DHCP relay/helper that:
1. Runs with `hostNetwork: true` to receive DHCP broadcasts
2. Forwards DHCP requests as unicast to the Pi-hole pod
3. Relays DHCP responses back to clients as broadcasts

This approach keeps Pi-hole in normal pod networking while only the minimal DHCP helper needs host access.

## Architecture

```
┌─────────────────┐     DHCP Broadcast      ┌──────────────────┐
│  DHCP Client    │ ───────────────────────►│  DHCP Helper     │
│  (Layer 2)      │      (port 67)          │  (hostNetwork)   │
└─────────────────┘          │              └────────┬─────────┘
                             │                       │ Unicast
                        tc redirect                  │ (port 1067)
                        67 → 1067                    ▼
                                           ┌──────────────────┐
                                           │  Pi-hole Pod     │
                                           │  (ClusterIP:1067)│
                                           └──────────────────┘
```

## Prerequisites: Traffic Control (tc) Rules

Since the DHCP helper runs as an unprivileged user, it cannot bind to privileged ports 67/68. Instead, it uses alternate ports 1067/1068. Traffic Control (tc) rules must be configured on each node to redirect DHCP traffic between the standard and alternate ports.

> **WARNING**: Traffic Control is a powerful tool that requires careful handling. Poorly configured tc rules can break networking and be difficult to remove. The production-ready script referenced below addresses these concerns.

**Recommended: Use the production-ready tc script**

Use the [tc.sh script from Slyrc/dhcp-helper-container](https://github.com/Slyrc/dhcp-helper-container/blob/master/tc.sh) which provides:

- **Dedicated preferences (prefs)** - Rules can be cleanly deleted without affecting other tc rules
- **Checksum recalculation** - Recalculates checksums after packet modification, preventing warnings and rejected packets
- **Precise matching** - Only capture dhcp traffic
- **Multiple scenarios** - Handles limited and directed broadcast, and unicast renewals

Download and customize the script for your environment:

```bash
curl -O https://raw.githubusercontent.com/Slyrc/dhcp-helper-container/master/tc.sh
# Edit the script to set your DEV, NODE_IP, LAN_BCAST, and LAN_CIDR values
chmod +x tc.sh
./tc.sh
```

### Simplified example (not recommended for production)

The following simplified commands demonstrate the concept but have significant limitations:
- No preferences assigned, making rule cleanup difficult
- No checksum recalculation, causing packet warnings/rejections
- Overly broad matching that may affect unintended traffic
- Uses deprecated `nat` action

```bash
# Ingress: Redirect incoming DHCP requests (port 67 → 1067)
tc qdisc add dev eth0 ingress
tc filter add dev eth0 parent ffff: protocol ip u32 \
  match ip dport 67 0xffff action nat ingress any 1067

# Egress: Rewrite outgoing DHCP responses (port 1067 → 67, 1068 → 68)
tc qdisc add dev eth0 root handle 1: prio
tc filter add dev eth0 parent 1: protocol ip u32 \
  match ip sport 1067 0xffff action nat egress any 67
tc filter add dev eth0 parent 1: protocol ip u32 \
  match ip dport 1068 0xffff action nat egress any 68
```

**Note**: Replace interface names and IP addresses with your node's values. The exact tc configuration depends on your environment and may require adjustments. Configuration of tc rules is the responsibility of the user and is outside the scope of this Helm chart.

Possible approaches for managing tc rules:
- Manual configuration on each node
- DaemonSet with privileged init container
- Node provisioning tools (cloud-init, Ansible, etc.)
- CNI plugin configuration

## Deployment

### 1. Configure tc rules

Before deploying, ensure tc rules are configured on all nodes where the DHCP helper will run. See the Prerequisites section above.

### 2. Deploy Pi-hole

Deploy Pi-hole using the provided values file. Pi-hole runs with normal pod networking and exposes DHCP via a ClusterIP service on port 1067:

```bash
helm install pihole mojo2600/pihole -f pihole-values.yaml
```

### 3. Deploy DHCP Helper

Update the service name in `dhcphelper.yaml` if your namespace or release name differs, then deploy:

```bash
kubectl apply -f dhcphelper.yaml
```

## Configuration

### Pi-hole Values (`pihole-values.yaml`)

Key settings:
- `hostNetwork: false` - Pi-hole uses normal pod networking
- `serviceDhcp.enabled: true` - Exposes DHCP on a ClusterIP service
- `serviceDhcp.port: 1067` - Alternate port for rootless operation
- `dnsmasq.customSettings` - Configures dnsmasq for alternate ports:
  - `dhcp-alternate-port` - Use ports 1067/1068 instead of 67/68
  - `dhcp-broadcast` - Send DHCP responses as broadcasts

### DHCP Helper (`dhcphelper.yaml`)

Key settings:
- `hostNetwork: true` - Required to receive Layer 2 broadcasts
- `runAsUser: 65534` - Runs as unprivileged nobody user
- `capabilities.drop: ["ALL"]` - No capabilities required
- Container args:
  - `-d` - Skip capability checks (required for rootless operation)
  - `-p` - Use alternate ports 1067/1068 (ports are hardcoded internally)
  - `-s pihole-dhcp.<namespace>.svc.cluster.local` - Pi-hole service DNS name

## Troubleshooting

### DHCP clients not receiving addresses

1. Verify tc rules are active and check packet counters:
   ```bash
   # Show ingress filter rules and packet counters
   tc -s filter show dev eth0 ingress
   # Show egress filter rules and packet counters
   tc -s filter show dev eth0 egress
   ```
   The `-s` flag shows statistics including packet counts - if DHCP traffic is flowing, you should see the counters incrementing.

2. Verify the DHCP helper is running:
   ```bash
   kubectl get pods -l app=dhcphelper
   ```

3. Verify Pi-hole DHCP is enabled:
   - Access Pi-hole admin UI
   - Go to Settings → DHCP
   - Ensure DHCP server is enabled

### Multiple nodes

If you have multiple nodes and want DHCP on all of them, you can change the DHCP helper to a DaemonSet. Each node will then have its own helper forwarding to Pi-hole. Ensure tc rules are configured on all nodes.

## References

- [Slyrc/dhcp-helper-container](https://github.com/Slyrc/dhcp-helper-container) - Rootless DHCP helper documentation
- [homeall/dhcphelper](https://github.com/homeall/dhcphelper) - The DHCP helper container image
- [Pi-hole Docker DHCP documentation](https://github.com/pi-hole/docker-pi-hole#running-dhcp-from-docker-pi-hole)
- [dnsmasq manual](https://thekelleys.org.uk/dnsmasq/docs/dnsmasq-man.html) - dhcp-alternate-port and dhcp-broadcast options
