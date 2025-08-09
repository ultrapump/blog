---
title: How to avoid load balancers on Vultr Kubernetes Engine
description: How to run nginx ingress with TLS termination directly on a VKE node - and the hurdles Vultr puts in your way.
image: /assets/img/img2.png
---

Latency is the product. Trading signals that arrive late are wrong by definition. To deliver consistently low latency for users in the US, EU and APAC, we operate geo-distributed infrastructure: multiple Kubernetes clusters in different regions, close to the exchanges and to our users. Centralizing in one hyperscale region would simplify life, but it would also add avoidable milliseconds at the edge where they hurt the most.

Crypto projects do not receive friendly treatment from the big clouds. Credits are scarce, and compliance reviews are slow. We optimized for predictable performance per dollar instead. Vultr Kubernetes Engine (VKE) gives us the same compute and network firepower for a fraction of the cost: flat per‑node pricing, no control‑plane tax, and regions where we need them. That lets us scale horizontally across geographies without compromising throughput or runway.

Current architecture at a glance:

- Cloudflare Workers sit in front of both `ultrapump.io` and `api.ultrapump.io`.
- Each Worker routes based on user geo to `us.ultrapump.io`, `eu.ultrapump.io` or `apac.ultrapump.io`.
- Every regional hostname resolves to multiple A/AAAA records, one per Kubernetes cluster in that region. We can scale at two layers:
  - within a cluster by adding nodes;
  - across the region by adding another cluster and its addresses to DNS.
- Regional clusters run nginx ingress directly on the node network interface and accept traffic only from Cloudflare IPs.

This design gives us fast failover, linear scale, and consistent routing semantics for both the app and the API. The remainder of this article zooms into one practical aspect of making this work on VKE: running nginx ingress without managed load balancers, the hard way, and a subtle networking trap we hit along the way.

---

This setup is powerful. It gives us high availability and a snappy user experience. The tradeoff is operational complexity. During the rollout of our US cluster we hit an issue that turned a simple `curl` into a multi‑hour investigation. Below is how we designed ingress on VKE, the exact failure we saw, and how we fixed and hardened the system.

## Why no managed load balancers

Managed LBs add cost and another control plane. On VKE we get better price/performance by terminating TLS on each node and letting Cloudflare do global anycast and DDoS protection. Benefits:

- Fewer hops and predictable latency
- Lower fixed cost per cluster
- Simple, horizontal scale by adding nodes and/or clusters

We take on responsibility for network policy and firewalling at the cloud and node layers. EU and APAC worked flawlessly. The US cluster did not.

## Ingress model on VKE

- nginx Ingress runs as a DaemonSet with `hostNetwork: true` so it binds to `:80` and `:443` on every node
- Cloudflare IP ranges are allow‑listed on the cloud firewall and `ufw` on each node
- Health endpoints are probed through Cloudflare

Example (trimmed) spec:

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx
spec:
  template:
    spec:
      hostNetwork: true
      containers:
      - name: controller
        image: registry.k8s.io/ingress-nginx/controller:v1.x
        ports:
        - containerPort: 80
        - containerPort: 443
```

Minimal `ufw` allow rules (accept only Cloudflare to 80/443):

```bash
# default deny inbound
sudo ufw default deny incoming

# allow Cloudflare IPv4 to 80/443
for cidr in $(curl -s https://www.cloudflare.com/ips-v4); do
  sudo ufw allow from $cidr to any port 80 proto tcp
  sudo ufw allow from $cidr to any port 443 proto tcp
done

# allow Cloudflare IPv6 to 80/443
for cidr in $(curl -s https://www.cloudflare.com/ips-v6); do
  sudo ufw allow from $cidr to any port 80 proto tcp
  sudo ufw allow from $cidr to any port 443 proto tcp
done

sudo ufw enable
sudo ufw status verbose
```

## Chapter 2: The Mystery of the Hanging `curl`

When we tried to hit our US endpoint, the result was maddeningly simple:

```bash
$ curl -v https://us.ultrapump.io
* Host us.ultrapump.io:443 was resolved.
* IPv4: 144.202.110.155
*   Trying 144.202.110.155:443...
# ...and it just hangs, forever.
```

A `curl` that hangs at the `Trying...` stage is a classic sign of a network-level problem. The TCP three-way handshake isn't completing. This isn't a certificate error or an application bug; the server isn't even answering the initial connection request.

Our troubleshooting journey began, following a logical path from the outside in:

1.  **DNS:** An initial misconfiguration pointed to the wrong IP. A quick fix, but the problem persisted. This was our first red herring.
2.  **Cloud firewall:** Using `vultr-cli`, we confirmed the firewall group allowed inbound 80/443 only from Cloudflare IP ranges; rules were present and counters were incrementing.
3.  **Node firewall:** We SSH'd into the node and checked `ufw status`. The allow‑lists matched Cloudflare and were active.
4.  **nginx process:** Still on the node, we ran `sudo netstat -tulpn | grep nginx`. This confirmed that the nginx process was, in fact, listening on `0.0.0.0:80` and `0.0.0.0:443`.

So, the firewalls were open and the process was listening. The drop had to live between the NIC and the process: `iptables`.

## Chapter 3: The `iptables` Rabbit Hole and the Calico Clue

A look at the node's `iptables` rules revealed a complex web of chains created by Kubernetes and, more importantly, by Calico, our CNI (Container Network Interface). The `INPUT` chain, which processes incoming traffic for the host, had a default policy of `DROP` and was handing off traffic to a series of `cali-` chains.

This was the smoking gun. Calico owned the host firewall. We tried a temporary `iptables` rule to accept traffic:

```bash
sudo iptables -I cali-INPUT -p tcp --dport 443 -j ACCEPT
```

It worked-briefly. Felix, Calico’s agent, reconciled the rules and removed our manual change. The problem wasn’t a missing rule; it was a control‑plane configuration gap.

## Chapter 4: The "Aha!" Moment - Configuration Drift

Why did this work in the EU and APAC but not in the US? The only possible answer was that the clusters were not identical. We pulled the YAML definition of the `calico-node` DaemonSet from both the working EU cluster and the broken US cluster and compared them.

The difference was stark. The US cluster's configuration was missing a critical set of environment variables:

```yaml
# Present in the working EU cluster, MISSING in the US cluster
- name: FELIX_BPFENABLED
  value: "true"
- name: FELIX_BPFEXTERNALSERVICEMODE
  value: tunnel
- name: FELIX_BPFHOSTNETWORKEDNATWITHOUTCTLB
  value: Disabled
- name: FELIX_BPFCONNECTTIMELOADBALANCING
  value: Enabled
- name: FELIX_BPFKUBEPROXYIPTABLESCLEANUPENABLED
  value: "true"
```

The US cluster used Calico’s default `iptables` data plane while EU/APAC used the modern eBPF data plane. With incomplete `iptables` policy, the default behavior was to drop traffic that wasn’t explicitly matched-including our ingress.

## Chapter 5: The Fix and the Moral

The fix was clear: align the US Calico config with the others. A targeted `kubectl patch` updated the DaemonSet and enabled the eBPF dataplane.

```bash
kubectl patch daemonset -n kube-system calico-node --context jigsx-us --type='json' -p='[
    {"op": "add", "path": "/spec/template/spec/containers/0/env/2", "value": {"name": "FELIX_BPFENABLED", "value": "true"}},
    {"op": "add", "path": "/spec/template/spec/containers/0/env/3", "value": {"name": "FELIX_BPFEXTERNALSERVICEMODE", "value": "tunnel"}},
    {"op": "add", "path": "/spec/template/spec/containers/0/env/4", "value": {"name": "FELIX_BPFHOSTNETWORKEDNATWITHOUTCTLB", "value": "Disabled"}},
    {"op": "add", "path": "/spec/template/spec/containers/0/env/5", "value": {"name": "FELIX_BPFCONNECTTIMELOADBALANCING", "value": "Enabled"}},
    {"op": "add", "path": "/spec/template/spec/containers/0/env/6", "value": {"name": "FELIX_BPFKUBEPROXYIPTABLESCLEANUPENABLED", "value": "true"}}
]'
```

The `calico-node` pod restarted, this time with the eBPF data plane enabled. The hostile `iptables` rules disappeared, and traffic began to flow. The `curl` command returned a glorious `200 OK`.

This was a textbook case of configuration drift. A subtle provisioning difference created a critical, hard‑to‑diagnose failure.

## Hardening and runbook

- Baseline Calico config via IaC (Terraform/Argo CD) and drift detection
- Pin the ingress image tag; roll with canaries per region
- Enforce Cloudflare IP allow‑lists at both the cloud firewall and `ufw`
- Health checks from the edge (through Cloudflare), not only inside the cluster
- Record a per‑region runbook: DNS, firewall, process, dataplane checks

Distributed systems amplify success and failure. Consistency and automation are non‑negotiable.

## Epilogue: The AI Assistant in the Debugging Loop

This work benefited from an AI co‑pilot: methodical checks, quick command generation, and fast access to networking context. Tools help, but process and discipline close incidents.


