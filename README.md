# headscale
https://github.com/webees/headscale

# tailscale
```shell
curl -fsSL https://tailscale.com/install.sh | sh

tailscale up --login-server https://${server_url} --auth-key ${authkey} --force-reauth
```

# helm3
```shell
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version
```

# k3s
```shell
# High Availability with an External DB
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.25 sh -s - \
--datastore-endpoint="postgres://xxxxxxxx:xxxxxxxxxxxxxxxx@ep-polished-meadow-xxxxxxxx.us-west-2.aws.neon.tech/k3s?options=endpoint=ep-polished-meadow-xxxxxxxx" \
--kubelet-arg='eviction-hard=memory.available<1%,imagefs.available<1%,imagefs.inodesFree<1%,nodefs.available<1%,nodefs.inodesFree<1%' \
--node-external-ip      XX.XX.XX.XX \ # IPv4/IPv6 external IP addresses to advertise for node
--node-ip               XX.XX.XX.XX \ # IPv4/IPv6 addresses to advertise for node
--advertise-address     XX.XX.XX.XX \ # IPv4 address that apiserver uses to advertise to members of the cluster (default: node-external-ip/node-ip)
--flannel-backend       host-gw \
--flannel-iface         tailscale0

echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bash_profile
echo "export K3S_RESOLV_CONF=/etc/resolv.conf" >> ~/.bash_profile
source ~/.bash_profile
```

```shell
cat << EOF > /etc/rancher/k3s/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "[https://demo.mirrors.io](https://<my-docker-mirror-host>)"
EOF
```

```shell
# some commands
k3s kubectl get nodes
k3s kubectl get pods -A
k3s kubectl get svc -A
k3s kubectl delete pod --grace-period=0 --force --namespace $namespace $name
crictl ps
crictl info
```

```shell
# uninstall
/usr/local/bin/k3s-uninstall.sh
```

# cert-manager
```shell
# If you have installed the CRDs manually instead of with the `--set installCRDs=true` option added to your Helm install command, you should upgrade your CRD resources before upgrading the Helm chart:
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.11.0
```

# rancher
```shell
# https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-7-3
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

k3s kubectl create ns cattle-system

helm upgrade --install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version 2.7.0 \
  --set hostname=rancher.dev.run \
  --set replicas=1 \
  --set global.cattle.psp.enabled=false
```

```shell
# bootstrap
echo https://rancher.dev.run/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')

# reset-password
kubectl --kubeconfig $KUBECONFIG -n cattle-system exec $(kubectl --kubeconfig $KUBECONFIG -n cattle-system get pods -l app=rancher --no-headers | head -1 | awk '{ print $1 }') -c rancher -- reset-password
```

# k3s-node
```shell
cat /var/lib/rancher/k3s/server/node-token

curl -sfL https://rancher-mirror.oss-cn-beijing.aliyuncs.com/k3s/k3s-install.sh | INSTALL_K3S_VERSION=v1.20.15+k3s1 INSTALL_K3S_MIRROR=cn K3S_URL=https://192.168.28.100:6443 K3S_TOKEN=K10fb4da3a53effbed27e5f5875f5505eb45a49cf995d9adbd0f85064c2e2a7ae17::server:a019f0db1cf838ea796002d4b1bc54a1 sh -

curl -sfL https://get.k3s.io | \
K3S_URL=https://XX.XX.XX.XX:6443 \
K3S_TOKEN=XXXXXXXXXXXXXXXXXXXXXXXX \
INSTALL_K3S_VERSION=v1.25.9+k3s1 sh -s - \
--kubelet-arg='eviction-hard=memory.available<1%,imagefs.available<1%,imagefs.inodesFree<1%,nodefs.available<1%,nodefs.inodesFree<1%' \
--node-external-ip XX.XX.XX.XX \
--node-ip          XX.XX.XX.XX \
--flannel-iface    nm-netmaker
```

```shell
# uninstall
/usr/local/bin/k3s-agent-uninstall.sh
```
