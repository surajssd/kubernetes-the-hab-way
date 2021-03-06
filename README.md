# Kubernetes The Hab Way

Kubernetes the Hab way shows setting up a Kubernetes cluster in which
Kubernetes components are [Habitat](habitat.sh) packages and services.

## Requirements

* Linux or MacOS on the host (Windows is not supported)
* [Vagrant](https://www.vagrantup.com/downloads.html)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
* [cfssl](https://github.com/cloudflare/cfssl#installation) (with cfssljson)

## Quickstart

```
vagrant destroy -f ; ./scripts/setup
```

The above command can be used to setup everything from scratch with a single
command. Setup will take several minutes. Use the [smoke test](#smoke-test) to
verify things work correctly.

## Step-by-step Setup

### Virtual machines

```
vagrant up
```

Verify all 3 Habitat supervisors are running and have joined the ring:

```
./scripts/list-peers
```

### Certificates & kubeconfig

Create a CA and certificates for all components:

```
./scripts/generate-ssl-certificates
```

Create necessary kubeconfig files:

```
./scripts/generate-kubeconfig
```

### etcd

First we start etcd:

```
# On node-0
sudo hab sup start core/etcd --topology leader

# On node-1 and node-2
sudo hab sup start core/etcd --topology leader --peer 192.168.222.10
```

NB: a service with topology `leader` requires at least 3 members. Only when 3
members are alive, the service will start. We take advantage of that to make
sure all three etcd member nodes start with the same "initial cluster" setting
(and therefore have the same cluster ID to be able to form a cluster).

For each node and service instance, we have to add an etcd environment config
file with service instance specific configuration, as that's not supported
by Habitat (as far as I know).

```
# On each node, replace X with node number (i.e. '0' for node-0 and so on)
cat >/var/lib/etcd-env-vars <<ETCD_ENV_VARS
export ETCD_LISTEN_CLIENT_URLS="https://192.168.222.1X:2379"
export ETCD_LISTEN_PEER_URLS="https://192.168.222.1X:2380"
export ETCD_ADVERTISE_CLIENT_URLS="https://192.168.222.1X:2379"
export ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.222.1X:2380"
ETCD_ENV_VARS
```

Now we have to update the etcd.default service group configuration to make etcd
use our SSL certifcates (this has only to be done once for a service group; we
use node-0 here):

```
# On node-0
for f in /vagrant/certificates/{etcd.pem,etcd-key.pem,ca.pem}; do sudo hab file upload etcd.default 1 "${f}"; done
sudo hab config apply etcd.default 1 /vagrant/config/svc-etcd.toml
```

Finally we verify etcd is running:

```
# On node-0
export ETCDCTL_API=3
etcdctl --debug --insecure-transport=false --endpoints https://192.168.222.10:2379 --cacert /vagrant/certificates/ca.pem --cert /vagrant/certificates/etcd.pem --key /vagrant/certificates/etcd-key.pem member list
```

All 3 members should be started:

```
b82a52a6ff5c63c3, started, node-1, https://192.168.222.11:2380, https://192.168.222.11:2379
ddfa35d3c9a4c741, started, node-2, https://192.168.222.12:2380, https://192.168.222.12:2379
f1986a6cf0ad46aa, started, node-0, https://192.168.222.10:2380, https://192.168.222.10:2379
```

### Kubernetes controller components

#### kubernetes-apiserver service

Start the kubernetes-apiserver service:

```
# On node-0
sudo hab sup start core/kubernetes-apiserver
```

Now we have to update the kubernetes-apiserver.default service group
configuration to provide the necessary SSL certificates and keys:

```
# On node-0
for f in /vagrant/certificates/{kubernetes.pem,kubernetes-key.pem,ca.pem,ca-key.pem}; do sudo hab file upload kubernetes-apiserver.default 1 "${f}"; done
sudo hab config apply kubernetes-apiserver.default 1 /vagrant/config/svc-kubernetes-apiserver.toml
```

Verify the API server is running:

```
# On the host
curl --cacert certificates/ca.pem --cert certificates/admin.pem --key certificates/admin-key.pem https://192.168.222.10:6443/version
```

#### kubernetes-controller-manager

Similar to the kube-apiserver setup, start the service, provide necessary
files and configure it accordingly:

```
# On node-0
sudo hab sup start core/kubernetes-controller-manager
for f in /vagrant/certificates/{ca.pem,ca-key.pem}; do sudo hab file upload kubernetes-controller-manager.default 1 "${f}"; done
sudo hab config apply kubernetes-controller-manager.default 1 /vagrant/config/svc-kubernetes-controller-manager.toml
```

#### kubernetes-scheduler

The kube-scheduler doesn't require specific configuration:

```
# On node-0
sudo hab sup start core/kubernetes-scheduler
```

### `kubernetes-the-hab-way` kubectl context

Create a new context and set it default:

```
./scripts/setup-kubectl
```

### Configure RBAC auth for kubelets

```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
EOF

cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: Kubernetes
EOF
```

### Verify the controller is ready

To verify that all controller components are ready, run:

```
kubectl get cs
```

Output should look like this:

```
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok
scheduler            Healthy   ok
etcd-2               Healthy   {"health": "true"}
etcd-0               Healthy   {"health": "true"}
etcd-1               Healthy   {"health": "true"}
```

### Kubernetes worker components

#### kube-proxy

Configure and start kube-proxy:

```
# On node-0
sudo mkdir -p /var/lib/kube-proxy
sudo cp /vagrant/config/kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
sudo hab sup start core/kubernetes-proxy
sudo hab config apply kubernetes-proxy.default 1 /vagrant/config/svc-kubernetes-proxy.toml

# On node-1 and node-2
sudo mkdir -p /var/lib/kube-proxy
sudo cp /vagrant/config/kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
sudo hab sup start core/kubernetes-proxy
```

After successful setup, `iptables -nvL -t nat` will show multiple new chains,
e.g. `KUBE-SERVICES`.

To reach services from your host, you can do:

```
# On Linux
sudo route add -net 10.32.0.0/24 gw 192.168.222.10
# On macOS
sudo route -n add -net 10.32.0.0/24 192.168.222.10
```

#### kubelet

Configure and start the kubelet:

```
# On each node, replace X with node number (i.e. '0' for node-0 and so on)
mkdir -p /var/lib/kubelet-config/cni
cat >/var/lib/kubelet-config/kubelet <<KUBELET_CONFIG
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubeletconfig/v1alpha1",
  "podCIDR": "10.2X.0.0/16"
}
KUBELET_CONFIG
cat >/var/lib/kubelet-config/cni/10-bridge.conf <<CNI_CONFIG
{
  "cniVersion": "0.3.1",
  "name": "bridge",
  "type": "bridge",
  "bridge": "cnio0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [{"subnet": "10.2X.0.0/16"}]
    ],
    "routes": [{"dst": "0.0.0.0/0"}]
  }
}
CNI_CONFIG
for f in /vagrant/certificates/{$(hostname)/node.pem,$(hostname)/node-key.pem,ca.pem} /vagrant/config/$(hostname)/kubeconfig; do sudo cp "${f}" "/var/lib/kubelet-config/"; done
sudo hab sup start core/kubernetes-kubelet
sudo hab config apply kubernetes-kubelet.default 1 /vagrant/config/svc-kubelet.toml # noop on repeated calls
```

Verify the 3 nodes are up and ready:

```
kubectl get nodes
```

Output should look like this:

```
NAME      STATUS    ROLES     AGE       VERSION
node-0    Ready     <none>    23m       v1.8.2
node-1    Ready     <none>    23m       v1.8.2
node-2    Ready     <none>    23m       v1.8.2
```

## Smoke test

Test the setup by creating a Nginx deployment, expose it and send a request
to the service IP:

```
# On Linux
sudo route add -net 10.32.0.0/24 gw 192.168.222.10
# On macOS
sudo route -n add -net 10.32.0.0/24 192.168.222.10

# Run the smoke test
./scripts/smoke-test
```
