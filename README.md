# vCluster - Monitoring Private Nodes

Metrics collection agents for scraping kubelet/cAdvisor metrics from private node vClusters and forwarding to a central Prometheus via remote write.

## Problem Statement

When running vCluster with private/isolated nodes, the central Prometheus cannot directly scrape kubelet metrics from those nodes. These templates deploy an agent **inside the vCluster** that:

1. Uses Kubernetes service discovery to find nodes
2. Scrapes metrics directly from node InternalIPs (like kube-prometheus-stack)
3. Forwards metrics to a central Prometheus instance via remote write

## Approach: Direct IP Scraping

These templates use **direct IP scraping** - the same approach used by kube-prometheus-stack. The agent connects directly to each node's kubelet at `node-ip:10250`.

```
Agent → node InternalIP:10250 → kubelet metrics
```

### Why Direct IP Instead of API Proxy?

An alternative approach is to scrape via the Kubernetes API server proxy (`/api/v1/nodes/{node}/proxy/metrics`). However, this has a critical limitation:

- **InternalDNS Issue**: Cloud providers (AWS, GCP, Azure) add `InternalDNS` addresses to nodes. The API server tries to resolve these DNS names before routing, but vCluster's CoreDNS cannot resolve cloud-internal DNS (e.g., `*.eu-west-1.compute.internal`), causing scrapes to fail.

Direct IP scraping avoids this entirely - no DNS resolution needed.

## Network Requirements

For direct IP scraping to work, the agent pod must be able to reach each node's InternalIP. This requires **one of**:

1. **VPN node-to-node enabled** (`privateNodes.vpn.nodeToNode.enabled: true`)
   - Each node advertises its InternalIP as a Tailscale route
   - Agent can reach any node via the Tailscale mesh
   - **Recommended for multi-cloud/multi-network setups**

2. **All nodes on the same network**
   - Nodes can reach each other directly
   - Works for single-network private node deployments

## Agent Comparison

Only tested on my local setup, your results may vary.

| Agent | Memory | Summary |
|-------|--------|---------|
| **vmagent** | ~170MB | Most efficient. Purpose-built for scrape+forward. |
| **OTel Collector** | ~350-430MB | CNCF standard, vendor-neutral, extensible pipeline. |
| **Prometheus Agent** | ~500-600MB | Prometheus running in agent mode |
| **Standard Prometheus** | ~650MB | Full query/alerting if needed. Included for reference. |

**Recommendation**: Use vmagent for the lightest footprint.

## Prerequisites

1. Central Prometheus exposed via internal LoadBalancer (or accessible endpoint)
2. `enableRemoteWriteReceiver: true` on central Prometheus
3. VPN node-to-node enabled OR all private nodes on the same network

## Available Templates

### VictoriaMetrics Agent (Recommended)
- `vcluster-vmagent-template.yaml`

### OpenTelemetry Collector
- `vcluster-otel-agent-template.yaml`

### Prometheus Agent Mode
- `vcluster-prometheus-agent-template.yaml`

### Standard Prometheus
- `vcluster-prometheus-kubelet-template.yaml`

## Grafana Dashboard

- `vcluster-private-nodes.json` - Grafana dashboard for visualizing private node metrics

Import this dashboard into Grafana to view:
- Node CPU and memory usage
- Container resource metrics from cAdvisor
- Pod resource consumption
- vCluster API server metrics

To import: Grafana → Dashboards → Import → Upload JSON file

![Grafana Dashboard](images/Screenshot%202026-01-30%20223928.png)

## Usage

1. Copy the template file and replace placeholders:
   - `PROMETHEUS_REMOTE_WRITE_URL` - Your central Prometheus endpoint (e.g., `192.168.1.213:9090`)
   - `VCLUSTER_NAME` - Identifier for this vCluster

2. Apply inside your vCluster context:
   ```bash
   kubectl apply -f your-configured-file.yaml
   ```

3. Verify the agent is running:
   ```bash
   kubectl get pods -n monitoring
   kubectl logs -n monitoring deployment/<agent-name>
   ```

4. Check metrics in central Prometheus - look for metrics with `vcluster="your-vcluster-name"` label

## Metrics Collected

All templates scrape the same metrics endpoints:

| Job Name | Endpoint | Description |
|----------|----------|-------------|
| `kubelet` | `:10250/metrics` | Kubelet operational metrics |
| `kubelet-cadvisor` | `:10250/metrics/cadvisor` | Container CPU, memory, network, filesystem |
| `kubelet-resource` | `:10250/metrics/resource` | Pod resource usage (newer endpoint) |
| `apiserver` | `/metrics` | vCluster API server metrics |

## How It Works

The agent uses Prometheus-style `kubernetes_sd_configs` with `role: node` to discover all nodes. The key relabel config extracts each node's InternalIP:

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_node_address_InternalIP]
    target_label: __address__
    replacement: ${1}:10250
```

This is the same pattern used by kube-prometheus-stack's kubelet ServiceMonitor.

## Memory Optimization Notes

### vmagent Optimizations
The vmagent template includes several memory optimizations:
- `-memory.allowedPercent=60` - Limits Go GC memory usage
- `-remoteWrite.maxDiskUsagePerURL=128MB` - Caps disk buffer for failed writes
- `-promscrape.noStaleMarkers` - Disables staleness tracking (minor memory saving)

### OTel Collector Optimizations
The OTel template includes:
- `memory_limiter` processor with 75% limit and 15% spike limit
- `batch` processor for efficient remote write batching
- `resource_to_telemetry_conversion: false` to avoid duplicate series

## Troubleshooting

### Scrape failures / connection refused
1. **Check VPN node-to-node is enabled**: Without it, the agent cannot reach other nodes' IPs
2. **Verify network connectivity**: From the agent pod, try `curl -k https://<node-ip>:10250/metrics`
3. **Check RBAC**: The agent needs `nodes` and `nodes/metrics` permissions

### Metrics not appearing in central Prometheus
1. Check agent logs for remote write errors
2. Verify network connectivity to central Prometheus
3. Ensure `enableRemoteWriteReceiver: true` is set

### Duplicate series errors (OTel Collector)
Ensure `resource_to_telemetry_conversion: enabled: false` in the exporter config. Old duplicate series will become stale after ~5 minutes.

### High memory usage
- vmagent: Should stay under 256MB with the provided config
- OTel Collector: Expect 350-430MB - memory_limiter should prevent OOM
- Prometheus Agent: Expect 500-600MB - this is normal

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     vCluster (Private Nodes)                │
│                                                             │
│  ┌─────────────┐                                            │
│  │   Agent     │                                            │
│  │ (vmagent/   │◄──── kubernetes_sd_configs: role: node     │
│  │  prom/otel) │                                            │
│  └──────┬──────┘                                            │
│         │                                                   │
│         │ direct scrape (via VPN mesh or same network)      │
│         │                                                   │
│         ├────────────────────┬────────────────────┐         │
│         ▼                    ▼                    ▼         │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐  │
│  │ Node A      │      │ Node B      │      │ Node C      │  │
│  │ :10250      │      │ :10250      │      │ :10250      │  │
│  │ (kubelet)   │      │ (kubelet)   │      │ (kubelet)   │  │
│  └─────────────┘      └─────────────┘      └─────────────┘  │
│                                                             │
└──────────────────────────┬──────────────────────────────────┘
                           │ remote write
                           ▼
                  ┌─────────────────────┐
                  │ Central Prometheus  │
                  │ (LoadBalancer)      │
                  └─────────────────────┘
```

## RBAC

The agent requires these permissions:

```yaml
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["nodes/metrics"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["services", "endpoints", "pods"]
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/metrics"]
    verbs: ["get"]
```

Note: `nodes/proxy` is NOT required for direct IP scraping.

## License

MIT
