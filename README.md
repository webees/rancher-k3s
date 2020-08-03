# k3s

```shell
swapoff -a

/usr/local/bin/k3s-uninstall.sh

curl -sfL https://get.k3s.io | sh -s - server \
   # --disable traefik \
   --datastore-endpoint="mysql://username:password@tcp(hostname:3306)/database"

# INSTALL_K3S_MIRROR
curl -sfL https://docs.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -s - server \
   # --disable traefik \
   --datastore-endpoint="mysql://username:password@tcp(hostname:3306)/database"
```

```shell
cd /var/lib/rancher/k3s/agent/etc/containerd
cp config.toml config.toml.tmpl

cat >>/var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl<<EOF
[plugins.cri.registry.mirrors]
  [plugins.cri.registry.mirrors."docker.io"]
    endpoint = ["https://docker.mirrors.ustc.edu.cn"]
  [plugins.cri.registry.mirrors."reg.xxx.com"]
    endpoint = ["https://reg.git.com"]

[plugins.cri.registry.configs."reg.xxx.com".tls]
  insecure_skip_verify = true
  ca_file   = "/etc/rancher/cacerts.pem"

[plugins.cri.registry.configs.auths."https://reg.xxx.com"]
  auth = "xxxxxxxxxxxxx"
EOF

systemctl restart k3s
systemctl status k3s
k3s kubectl get nodes
k3s kubectl get pods -A
k3s kubectl get svc -A
crictl ps
crictl info
```

# helm3

```shell
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
$ helm version
```

# rancher

```shell
1$ helm repo add rancher-stable https://releases.rancher.com/server-charts/stable

2$ helm repo add rancher-stable http://rancher-mirror.oss-cn-beijing.aliyuncs.com/server-charts/stable

helm repo update
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
k3s kubectl create ns cattle-system
k3s kubectl -n cattle-system create secret generic tls-ca --from-file=/etc/rancher/cacerts.pem

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.dev.run \
  --set ingress.tls.source=secret \
  --set tls=external \
  --set privateCA=true

k3s kubectl -n cattle-system rollout status deploy/rancher
k3s kubectl -n cattle-system get deploy rancher
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
