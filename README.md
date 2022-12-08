# mysql
```shell
mkdir -p /app/k3s/mysql

cd /app/k3s/mysql
```

```shell
cat << EOF > /app/k3s/mysql/my.cnf
[mysqld]
skip_host_cache
explicit_defaults_for_timestamp = true

bind-address = 0.0.0.0
skip-name-resolve
EOF
```

```shell
cat << EOF > /app/k3s/mysql/compose.yaml
version: '3.5'
services:
  mysql:
    restart: always
    container_name: k3s_mysql
    image: mysql:8
    ports:
      - 3306
    volumes:
      - ./mysql:/var/lib/mysql
      - ./my.cnf:/etc/mysql/conf.d/my.cnf
    environment:
      MYSQL_DATABASE: "k3s"
      MYSQL_USER: "k3s"
      MYSQL_PASSWORD: "TWc0XCfLRG7F3o381WleEuBgZDDN1F"
      MYSQL_ROOT_PASSWORD: "j3aQnsLQJdtBEZtZX7W6r6Wa4wiyFU"
    healthcheck:
      test: mysqladmin ping -h localhost
      interval: 10s
      timeout: 30s
      retries: 5
      start_period: 30s
EOF
```

# k3s
```shell
# High Availability with an External DB
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.24.8+k3s1 sh -s - \
--datastore-endpoint="mysql://username:password@tcp(hostname:3306)/database" \
--kubelet-arg='eviction-hard=memory.available<100Mi,imagefs.available<0.1%,imagefs.inodesFree<0.1%,nodefs.available<0.1%,nodefs.inodesFree<0.1%' \
--node-external-ip      XX.XX.XX.XX \
--node-ip               XX.XX.XX.XX \
--advertise-address     XX.XX.XX.XX \
--flannel-backend       host-gw \
--flannel-iface         tailscale0

echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bash_profile
source ~/.bash_profile
```

```shell
cat << EOF > /etc/rancher/k3s/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.mirrors.ustc.edu.cn" 
EOF
```

# helm3
```shell
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version
```

# cert-manager
```shell
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
# https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-7-0/

helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

k3s kubectl create ns cattle-system

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version 2.7.0 \
  --set hostname=rancher.dev.run \
  --set replicas=3
```

# reset-password
```shell
kubectl --kubeconfig $KUBECONFIG -n cattle-system exec $(kubectl --kubeconfig $KUBECONFIG -n cattle-system get pods -l app=rancher --no-headers | head -1 | awk '{ print $1 }') -c rancher -- reset-password
```

# k3s-node
```shell
cat /var/lib/rancher/k3s/server/node-token

curl -sfL https://rancher-mirror.oss-cn-beijing.aliyuncs.com/k3s/k3s-install.sh | INSTALL_K3S_VERSION=v1.20.15+k3s1 INSTALL_K3S_MIRROR=cn K3S_URL=https://192.168.28.100:6443 K3S_TOKEN=K10fb4da3a53effbed27e5f5875f5505eb45a49cf995d9adbd0f85064c2e2a7ae17::server:a019f0db1cf838ea796002d4b1bc54a1 sh -

curl -sfL https://get.k3s.io | \
K3S_URL=https://XX.XX.XX.XX:6443 \
K3S_TOKEN=XXXXXXXXXXXXXXXXXXXXXXXX \
INSTALL_K3S_VERSION=v1.24.8+k3s1 sh -s - \
--kubelet-arg='eviction-hard=memory.available<100Mi,imagefs.available<0.1%,imagefs.inodesFree<0.1%,nodefs.available<0.1%,nodefs.inodesFree<0.1%' \
--node-external-ip XX.XX.XX.XX \
--node-ip          XX.XX.XX.XX \
--flannel-iface    nm-netmaker
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

```shell
/usr/local/bin/k3s-uninstall.sh
/usr/local/bin/k3s-agent-uninstall.sh
```

# ansible
```shell
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
