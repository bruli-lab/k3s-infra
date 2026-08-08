# kube-vip

kube-vip provides two independent highly available addresses on `eth0`:

| Purpose | Address | Leader lease |
| --- | --- | --- |
| K3s API | `192.168.1.19:6443` | `plndr-cp-lock` |
| Traefik services | `192.168.1.20:80/443` | `kubevip-traefik` |

The DaemonSet runs only on control-plane nodes, uses ARP mode, and performs a
separate leader election for every `LoadBalancer` Service. Traefik requests
`192.168.1.20` through `../traefik/helmchartconfig.yaml`.

## Apply

This is bootstrap infrastructure and can be applied directly before Argo CD is
available:

```bash
kubectl apply -k infra/kube-vip
kubectl apply -f infra/traefik/helmchartconfig.yaml
```

Verify the rollout and addresses:

```bash
kubectl -n kube-system rollout status daemonset/kube-vip-ds --timeout=180s
kubectl -n kube-system get pods -l app.kubernetes.io/name=kube-vip-ds -o wide
kubectl -n kube-system get leases | grep -E 'plndr|kubevip'
kubectl -n kube-system get service traefik -o wide
```

## Required K3s server configuration

Every server must include the API VIP in `/etc/rancher/k3s/config.yaml` and
disable the built-in ServiceLB. `node-ip` is different on each node:

```yaml
node-ip: 192.168.1.X
tls-san:
  - 192.168.1.19
disable:
  - servicelb
```

Current server addresses:

| Node | Address |
| --- | --- |
| `cluster-master` | `192.168.1.14` |
| `cluster-node1` | `192.168.1.13` |
| `cluster-node2` | `192.168.1.18` |

Restart K3s on one server at a time after changing its configuration, waiting
for it to become Ready before continuing with the next server.

## Failover check

Identify the current owners:

```bash
kubectl -n kube-system get leases \
  -o custom-columns='NAME:.metadata.name,HOLDER:.spec.holderIdentity,RENEWED:.spec.renewTime' \
  | grep -E 'plndr|kubevip-traefik'
```

During a real node failure both VIPs should move to surviving control-plane
nodes. A controlled `systemctl stop k3s` is not an equivalent test because the
K3s unit uses `KillMode=process`, which can leave kube-vip running.
