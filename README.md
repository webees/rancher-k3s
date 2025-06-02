# SUSE Rancher Support Matrix Summary

The [SUSE Rancher Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions) provides a **comprehensive compatibility guide** for Rancher deployments, detailing supported **Kubernetes versions**, **operating systems**, **container runtimes**, and **certified integrations** (CNI, storage, cloud). It defines **lifecycle policies**, **maintenance schedules**, and **upgrade paths**, while highlighting differences across **deployment modes**. This matrix is essential for **installation planning**, **compatibility troubleshooting**, and **upgrade strategy**, and is **regularly updated by SUSE** to ensure accuracy.

# System Update and Kernel Version
```
uname -r
apt-get update
apt-get upgrade
apt-get upgrade linux-image-generic
```

# UFW Firewall Management
```bash
ufw status verbose
ufw show added

ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 41641/udp
ufw allow 3478/udp
ufw enable

# ufw status numbered
# ufw delete 1
# ufw disable
```

# Network Performance Testing with iperf3
```
iperf3 -s -p 8888
iperf3 -u -p 8888 -c 1.1.1.1
```

# BBR
```
echo net.core.default_qdisc=fq >> /etc/sysctl.conf
echo net.ipv4.tcp_congestion_control=bbr >> /etc/sysctl.conf
sysctl -p
sysctl net.ipv4.tcp_available_congestion_control
```

# disable 127.0.0.53
```
mkdir -p /etc/systemd/resolved.conf.d/
cat >/etc/systemd/resolved.conf.d/98-disable-127-53.conf << EOF
[Resolve]
DNSStubListener=no
EOF
systemctl daemon-reload && systemctl restart systemd-resolved.service && systemctl status -l systemd-resolved.service --no-pager
ss -tunlp
cat /etc/resolv.conf
```

# Enable IPv4 and IPv6 Forwarding Persistently
```
sudo sed -i '/^net\.ipv4\.ip_forward/ d'              /etc/sysctl.conf
sudo sed -i '/^net\.ipv6\.conf\.all\.forwarding/ d'   /etc/sysctl.conf
echo 'net.ipv4.ip_forward = 1'          | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf

tail -5 /etc/sysctl.conf
sysctl -p /etc/sysctl.conf
sysctl -p
ip a
```

# Enable IPv6
```shell
cat /proc/sys/net/ipv6/conf/all/disable_ipv6

sudo sed -i '/^net\.ipv6\.conf\.all\.disable_ipv6/ d'       /etc/sysctl.conf
sudo sed -i '/^net\.ipv6\.conf\.default\.disable_ipv6/ d'   /etc/sysctl.conf
sudo sed -i '/^net\.ipv6\.conf\.lo\.disable_ipv6/ d'        /etc/sysctl.conf
echo 'net.ipv6.conf.all.disable_ipv6 = 0'     | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.default.disable_ipv6 = 0' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.lo.disable_ipv6 = 0'      | sudo tee -a /etc/sysctl.conf

tail -5 /etc/sysctl.conf
sysctl -p /etc/sysctl.conf
sysctl -p
ip a

netplan apply
ip -6 addr show
```

# Headscale
https://github.com/webees/headscale

# Tailscale
https://tailscale.com/kb/
```shell
curl -fsSL https://tailscale.com/install.sh | sh

# tailscale up --login-server https://${server_url} --auth-key ${authkey} --force-reauth
```

# K3s Agent
Copy ```/etc/rancher/k3s/k3s.yaml``` on your machine located outside the cluster as ```~/.kube/config```. Then replace the value of the ```server``` field with the IP or name of your K3s server. ```kubectl``` can now manage your K3s cluster.

```shell
curl -sfL https://get.k3s.io | \
K3S_URL=https://XX.XX.XX.XX:6443 \
K3S_TOKEN=XXXXXXXXXXXXXXXXXXXXXXXX \
INSTALL_K3S_VERSION=v1.32.5+k3s1 sh -s - \
--kubelet-arg        "eviction-hard=memory.available<1%,imagefs.available<1%,imagefs.inodesFree<1%,nodefs.available<1%,nodefs.inodesFree<1%" \
--kube-proxy-arg     "ipvs-scheduler=lc,proxy-mode=ipvs" \
--node-external-ip   "$(curl -4 -s https://ifconfig.me)" \
--node-external-ip   "$(curl -4 -s https://ifconfig.me)" \
--node-ip            "$(tailscale ip -4 | tr -d '\n')" \
--flannel-iface      "tailscale0"
```

```shell
/usr/local/bin/k3s-agent-uninstall.sh
```














# helm3
```shell
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version
```

# Node Feature Discovery
```
export NFD_NS=node-feature-discovery
helm repo add nfd https://kubernetes-sigs.github.io/node-feature-discovery/charts
helm repo update
helm install nfd/node-feature-discovery --namespace $NFD_NS --create-namespace --generate-name
```

# k3s server
```shell
# High Availability with an External DB
curl -sfL https://get.k3s.io | \
INSTALL_K3S_VERSION=v1.31.6+k3s1 sh -s - \
# --datastore-endpoint "postgres://xxxxxxxx:xxxxxxxxxxxxxxxx@ep-polished-meadow-xxxxxxxx.us-west-2.aws.neon.tech/k3s?options=endpoint=ep-polished-meadow-xxxxxxxx" \
--kubelet-arg        "eviction-hard=memory.available<0.1%,imagefs.available<0.1%,imagefs.inodesFree<0.1%,nodefs.available<0.1%,nodefs.inodesFree<0.1%" \
--kube-apiserver-arg "service-node-port-range=1-65535" \
--kube-proxy-arg     "ipvs-scheduler=lc,proxy-mode=ipvs" \
--node-external-ip   XX.XX.XX.XX \ # IPv4/IPv6 external IP addresses to advertise for node
--node-ip            XX.XX.XX.XX \ # IPv4/IPv6 addresses to advertise for node
--advertise-address  XX.XX.XX.XX \ # IPv4 address that apiserver uses to advertise to members of the cluster (default: node-external-ip/node-ip)
# --flannel-backend  host-gw \
--flannel-iface      tailscale0

echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bash_profile
echo "export K3S_RESOLV_CONF=/etc/resolv.conf" >> ~/.bash_profile
source ~/.bash_profile
```

```shell
# K3S_TOKEN
cat /var/lib/rancher/k3s/server/node-token
```

```shell
cat << EOF > /etc/rancher/k3s/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "https://<my-docker-mirror-host>"
EOF
```

```shell
# some commands
/usr/local/bin/k3s-killall.sh
journalctl -u k3s -f
kubectl describe nodes
kubectl get apiservice
kubectl get componentstatus
kubectl get nodes
kubectl get pods -A
kubectl get pods --all-namespaces -o wide
kubectl get pods --all-namespaces --field-selector=status.phase!=Running
kubectl get svc -A
kubectl delete pod --grace-period=0 --force --namespace $namespace $name
kubectl get deployment -n $namespace $name -o yaml
kubectl scale deployment --all --replicas=0 -n $namespace

kubectl delete namespace $namespace
kubectl delete pods --all -n $namespace

crictl ps
crictl info

kubectl get nodes --no-headers \
  | grep '^vpn-' \
  | awk '{print $1}' \
  | xargs -I{} kubectl taint nodes {} no-pods=deny:NoSchedule \n
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
# https://www.suse.com/suse-rancher/support-matrix/all-supported-versions/rancher-v2-7-5
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

helm upgrade --install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --version 2.7.5 \
  --set hostname=rancher.dev.run \
  --set replicas=1 \
  --set global.cattle.psp.enabled=false # For Kubernetes v1.25 or later, set global.cattle.psp.enabled to false.
```

```shell
# bootstrap
echo https://rancher.dev.run/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')

# reset-password
kubectl --kubeconfig $KUBECONFIG -n cattle-system exec $(kubectl --kubeconfig $KUBECONFIG -n cattle-system get pods -l app=rancher --no-headers | head -1 | awk '{ print $1 }') -c rancher -- reset-password
```
