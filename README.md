# k3s

```shell
/usr/local/bin/k3s-uninstall.sh

curl -sfL https://get.k3s.io | sh -s - server \
   # --disable traefik \
   --datastore-endpoint="mysql://username:password@tcp(hostname:3306)/database"

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
EOF

systemctl restart k3s
systemctl status k3s
k3s kubectl get nodes
k3s kubectl get pods -A
k3s kubectl get svc -A
crictl ps
crictl info
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
