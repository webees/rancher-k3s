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

# k3s

```shell
swapoff -a

# High Availability with Embedded DB
curl -sfL https://get.k3s.io | sh -

# High Availability with an External DB
curl -sfL https://get.k3s.io | sh -s - server \
   --datastore-endpoint="mysql://username:password@tcp(hostname:3306)/database"

# INSTALL_K3S_MIRROR
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_VERSION=v1.20.11+k3s2 INSTALL_K3S_MIRROR=cn sh -s - --kubelet-arg='eviction-hard=memory.available<100Mi,imagefs.available<0.1%,imagefs.inodesFree<0.1%,nodefs.available<0.1%,nodefs.inodesFree<0.1%'
```

```shell
cat << EOF > /etc/rancher/k3s/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "https://docker.mirrors.ustc.edu.cn" 
EOF
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

# node

```shell
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://192.168.28.100:6443 K3S_TOKEN=K10fb4da3a53effbed27e5f5875f5505eb45a49cf995d9adbd0f85064c2e2a7ae17::server:a019f0db1cf838ea796002d4b1bc54a1 sh -
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

# helm3

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

# rancher

```shell
1$ helm repo add rancher-stable https://releases.rancher.com/server-charts/stable

2$ helm repo add rancher-stable http://rancher-mirror.oss-cn-beijing.aliyuncs.com/server-charts/stable

3$ helm repo add rancher-stable http://rancher-mirror.cnrancher.com/server-charts/stable

helm repo update
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
k3s kubectl create ns cattle-system
k3s kubectl -n cattle-system create secret generic tls-ca --from-file=/etc/rancher/cacerts.pem

# helm upgrade --install
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --version 2.5.11 \
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
k3s kubectl --kubeconfig $KUBECONFIG -n cattle-system exec $(k3s kubectl --kubeconfig $KUBECONFIG -n cattle-system get pods -l app=rancher | grep '1/1' | head -1 | awk '{ print $1 }') -- reset-password
```

# gitlab-runner

> https://gitlab.com/help/user/project/clusters/add_remove_clusters.md

- API URL
```
kubectl cluster-info | grep 'Kubernetes master' | awk '/http/ {print $NF}'
```

- CA certificate
```
kubectl get secret $(kubectl get secret | grep default-token | awk '{print $1}') -o jsonpath="{['data']['ca\.crt']}" | base64 --decode
```

- Token
```
cat <<EOF | kubectl apply -f -  
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gitlab-admin
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: gitlab-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: gitlab-admin
  namespace: kube-system
EOF

kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep gitlab-admin | awk '{print $1}')
```

- check
```
kubectl get deploy -A | grep tiller
kubectl get pods -n gitlab-managed-apps
```

- fix x509
```
env:
- name: CI_SERVER_TLS_CA_FILE
  value: /home/gitlab-runner/.gitlab-runner/certs/ca.test.crt
- name: RUNNER_BUILDS_DIR
  value: /opt/gitlab-runner/builds
- name: RUNNER_CACHE_DIR
  value: /opt/gitlab-runner/cache
volumeMounts:
- name: ca-test
  mountPath: /home/gitlab-runner/.gitlab-runner/certs
```

- fix configmap
```
entrypoint: |-
############################################
sed -i '$d' ~/.gitlab-runner/config.toml
cat >> ~/.gitlab-runner/config.toml << EOF
    [[runners.kubernetes.volumes.host_path]]
      name = "gitlab-runner"
      mount_path = "/opt/gitlab-runner/"
      read_only = false
      host_path = "/home/gitlab/runner/"
    [[runners.kubernetes.volumes.host_path]]
      name = "docker"
      mount_path = "/var/run/docker.sock"
      read_only = true
      host_path = "/var/run/docker.sock"
EOF

# Start the runner
############################################
```

- fix hosts
```
- name: RUNNER_PRE_CLONE_SCRIPT # optional
  value: |-
    cat>> /etc/hosts <<EOF
    192.168.28.123 git.com
    192.168.28.123 reg.git.com
    EOF
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
