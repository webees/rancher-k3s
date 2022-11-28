
# netclient

```shell
curl -sL 'https://apt.netmaker.org/gpg.key' | sudo tee /etc/apt/trusted.gpg.d/netclient.asc
curl -sL 'https://apt.netmaker.org/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/netclient.list
sudo apt update
sudo apt install netclient
```

# k3s

```shell
# https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-5-16/
# High Availability with Embedded DB
curl -sfL https://get.k3s.io | sh -
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.20.15+k3s1 sh -

# High Availability with an External DB
curl -sfL https://get.k3s.io | sh -s - server \
   --datastore-endpoint="mysql://username:password@tcp(hostname:3306)/database"

# WireGuard
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.20.15+k3s1 sh -s - --kubelet-arg='eviction-hard=memory.available<100Mi,imagefs.available<0.1%,imagefs.inodesFree<0.1%,nodefs.available<0.1%,nodefs.inodesFree<0.1%' --node-external-ip SERVER_PUBLIC_IP --advertise-address NM_NODE_IP --node-ip NM_NODE_IP --flannel-iface nm-netmaker

echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bash_profile
source .bash_profile
```

```shell
cat << EOF > /etc/rancher/k3s/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.mirrors.ustc.edu.cn" 
EOF
```

# k3s-node

```shell
cat /var/lib/rancher/k3s/server/node-token

curl -sfL https://rancher-mirror.oss-cn-beijing.aliyuncs.com/k3s/k3s-install.sh | INSTALL_K3S_VERSION=v1.20.15+k3s1 INSTALL_K3S_MIRROR=cn K3S_URL=https://192.168.28.100:6443 K3S_TOKEN=K10fb4da3a53effbed27e5f5875f5505eb45a49cf995d9adbd0f85064c2e2a7ae17::server:a019f0db1cf838ea796002d4b1bc54a1 sh -

curl -sfL https://rancher-mirror.oss-cn-beijing.aliyuncs.com/k3s/k3s-install.sh | INSTALL_K3S_VERSION=v1.20.15+k3s1 INSTALL_K3S_MIRROR=cn K3S_URL=https://K3S_CONTROL_IP:6443 K3S_TOKEN=K10fb4da3a53effbed27e5f5875f5505eb45a49cf995d9adbd0f85064c2e2a7ae17::server:a019f0db1cf838ea796002d4b1bc54a1 sh -s - --node-external-ip SERVER_PUBLIC_IP --node-ip NM_NODE_IP --flannel-iface nm-netmaker
```

```shell
systemctl restart k3s
systemctl status k3s
k3s kubectl get nodes
k3s kubectl get pods -A
k3s kubectl get svc -A
k3s kubectl delete pod --grace-period=0 --force --namespace $namespace $name
crictl ps
crictl info
```

```
/usr/local/bin/k3s-uninstall.sh
/usr/local/bin/k3s-agent-uninstall.sh
```

# helm3

```shell
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version
```

# cert-manager
```
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.5.1/cert-manager.crds.yaml
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.5.1
```

```
# If you have installed the CRDs manually instead of with the `--set installCRDs=true` option added to your Helm install command, you should upgrade your CRD resources before upgrading the Helm chart:
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.1/cert-manager.crds.yaml

# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io

# Update your local Helm chart repository cache
helm repo update

# Install the cert-manager Helm chart
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.7.1
```

# rancher

```shell
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

k3s kubectl create ns cattle-system

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version 2.5.16 \
  --set hostname=rancher.dev.run \
  --set replicas=3

#########################################################################################################
k3s kubectl -n cattle-system create secret generic tls-ca --from-file=/etc/rancher/cacerts.pem

# helm upgrade --install
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version 2.5.16 \
  --set hostname=rancher.dev.run \
  --set ingress.tls.source=secret \
  --set tls=external \
  --set privateCA=true

k3s kubectl -n cattle-system rollout status deploy/rancher
k3s kubectl -n cattle-system get deploy rancher
k3s kubectl -n cattle-system get pods
```

- reset-password
```
kubectl --kubeconfig $KUBECONFIG -n cattle-system exec $(kubectl --kubeconfig $KUBECONFIG -n cattle-system get pods -l app=rancher --no-headers | head -1 | awk '{ print $1 }') -c rancher -- reset-password
```

# WireGuard 
```
https://github.com/linuxserver/docker-wireguard


docker run -d \
  --name=wireguard \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_MODULE \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Asia/Shanghai \
  -e SERVERURL=0.0.0.0 \
  -e SERVERPORT=51820 \
  -e PEERS=2 \
  -e PEERDNS=auto \
  -e INTERNAL_SUBNET=10.10.10.0 \
  -e ALLOWEDIPS=10.10.10.1/24 \
  -v /root/.wireguard/config:/config \
  -v /lib/modules:/lib/modules \
  --net=host \
  --restart unless-stopped \
  linuxserver/wireguard



docker run -d \
  --name=wireguard \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_MODULE \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Asia/Shanghai \
  -v /root/.wireguard/config:/config \
  -v /lib/modules:/lib/modules \
  --net=host \
  --restart unless-stopped \
  linuxserver/wireguard
```


# docker-compose
```
version: '3'
services:
  rancher:
    image: 'rancher/rancher:v2.5.12-rc1'
    privileged: true
    restart: unless-stopped
    network_mode: host
    volumes:
      - ./data:/var/lib/rancher
      - /run/shm/rancher/auditlog:/var/log/auditlog
    environment:
      - TZ=Asia/Shanghai
```

# ansible

```
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

# prometheus

```shell
apt install prometheus-node-exporter

curl -s http://127.0.0.1:9100/metrics | curl --data-binary @- http://127.0.0.1:9091/metrics/job/node/instance/"192.168.1.2:9100"
```

# nfs

```shell
# server
sudo apt-get update
sudo apt-get install nfs-kernel-server
sudo mkdir -p /nfs
sudo chown nobody:nogroup /nfs
# Modify the exports file (/etc/exports).
# /nfs    *(rw,sync,no_subtree_check,no_root_squash)
sudo systemctl restart nfs-kernel-server

# client
sudo apt update
sudo apt install nfs-common
sudo mkdir -p /nfs
sudo mount 192.168.1.2:/nfs /nfs
```

# traefik2

```shell
# Execute before k3s installation
mkdir -p /var/lib/rancher/k3s/server/manifests

cat << EOF > /var/lib/rancher/k3s/server/manifests/traefik.yml
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: traefik
  namespace: kube-system
spec:
  chart: traefik
  repo: https://containous.github.io/traefik-helm-chart
  set:
    image.tag: "2.2.1"  
EOF
```
