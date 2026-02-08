# SUSE Rancher Support Matrix Summary

The [SUSE Rancher Support Matrix](https://www.suse.com/suse-rancher/support-matrix/all-supported-versions) is the **authoritative compatibility reference** for Rancher deployments.  
It defines supported **Kubernetes versions**, **operating systems**, **container runtimes**, and **certified integrations** (CNI, storage, cloud).

Use this matrix for:
- 🧭 Installation planning
- 🛠️ Compatibility troubleshooting
- 🔄 Upgrade & lifecycle decisions

Always verify versions against the matrix before deploying or upgrading.

---

# System Update and Kernel Version 🧠

Check kernel version and update system packages:

```bash
uname -r
apt-get update
apt-get upgrade
apt-get upgrade linux-image-generic
```

Install Ubuntu 22.04 HWE kernel (recommended):

```bash
apt update
apt install --install-recommends linux-generic-hwe-22.04 -y
reboot
```

---

# UFW Firewall Management 🔥

Inspect firewall status:

```bash
ufw status verbose
ufw show added
```

Allow required ports (HTTP / HTTPS / Tailscale / STUN):

```bash
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 41641/udp
ufw allow 3478/udp
ufw enable
```

Maintenance helpers:

```bash
# ufw status numbered
# ufw delete 1
# ufw disable
```

---

# Network Performance Testing (iperf3) 🚀

Start server:

```bash
iperf3 -s -p 8888
```

Run UDP client test:

```bash
iperf3 -u -p 8888 -c 1.1.1.1
```

---

# Enable BBR Congestion Control ⚡

Improve TCP throughput and latency:

```bash
echo net.core.default_qdisc=fq >> /etc/sysctl.conf
echo net.ipv4.tcp_congestion_control=bbr >> /etc/sysctl.conf
sysctl -p
sysctl net.ipv4.tcp_available_congestion_control
```

---

# Disable systemd DNS Stub (127.0.0.53) 🧩

Avoid DNS conflicts with Kubernetes and containers:

```bash
mkdir -p /etc/systemd/resolved.conf.d/
cat >/etc/systemd/resolved.conf.d/98-disable-127-53.conf << EOF
[Resolve]
DNSStubListener=no
EOF
systemctl daemon-reload && systemctl restart systemd-resolved.service && systemctl status -l systemd-resolved.service --no-pager
ss -tunlp
cat /etc/resolv.conf
```

---

# Enable IPv4 & IPv6 Forwarding (Persistent) 🌐

Required for routing and overlay networking:

```bash
sudo sed -i '/^net\.ipv4\.ip_forward/ d'              /etc/sysctl.conf
sudo sed -i '/^net\.ipv6\.conf\.all\.forwarding/ d'   /etc/sysctl.conf
echo 'net.ipv4.ip_forward = 1'          | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf

tail -5 /etc/sysctl.conf
sysctl -p /etc/sysctl.conf
sysctl -p
ip a
```

---

# Enable IPv6 Support 🟣

Ensure IPv6 is enabled at kernel level:

```bash
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

---

# Headscale 🐝

Self-hosted Tailscale control plane:

https://github.com/webees/headscale

---

# Tailscale 🧵

Official documentation:

https://tailscale.com/kb/

Install Tailscale client:

```bash
curl -fsSL https://tailscale.com/install.sh | sh

# tailscale up --login-server https://${server_url} --auth-key ${authkey} --force-reauth
```

---

# K3s Agent Setup 🤖

Copy `/etc/rancher/k3s/k3s.yaml` from the server to your local machine as `~/.kube/config`,  
then update the `server:` field to the K3s API endpoint.

Install K3s agent (version aligned with server):

```bash
curl -sfL https://get.k3s.io | \
K3S_URL=https://XX.XX.XX.XX:6443 \
K3S_TOKEN=XXXXXXXXXXXXXXXXXXXXXXXX \
INSTALL_K3S_VERSION=v1.31.6+k3s1 sh -s - \
--kubelet-arg        "eviction-hard=memory.available<1%,imagefs.available<1%,imagefs.inodesFree<1%,nodefs.available<1%,nodefs.inodesFree<1%" \
--kube-proxy-arg     "ipvs-scheduler=wlc,proxy-mode=ipvs" \
--node-external-ip   "$(curl -4 -s https://ifconfig.me)" \
--node-ip            "$(tailscale ip -4 | tr -d '\n')" \
--flannel-iface      "tailscale0"
```

Uninstall agent:

```bash
/usr/local/bin/k3s-agent-uninstall.sh
```

---

# Helm 3 📦

Install Helm:

```bash
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm version
```

---

# Node Feature Discovery 🧬

Detect node hardware and kernel capabilities:

```bash
export NFD_NS=node-feature-discovery
helm repo add nfd https://kubernetes-sigs.github.io/node-feature-discovery/charts
helm repo update
helm install nfd/node-feature-discovery --namespace $NFD_NS --create-namespace --generate-name
```

---

# K3s Server Setup 🧠

Install K3s server:

```bash
curl -sfL https://get.k3s.io | \
INSTALL_K3S_VERSION=v1.31.6+k3s1 sh -s - \
--cluster-cidr 10.42.0.0/16 \
--service-cidr 10.43.0.0/16 \
--kubelet-arg        "eviction-hard=memory.available<1%,imagefs.available<1%,imagefs.inodesFree<1%,nodefs.available<1%,nodefs.inodesFree<1%" \
--kube-apiserver-arg "service-node-port-range=1-65535" \
--kube-proxy-arg     "ipvs-scheduler=lc,proxy-mode=ipvs" \
--node-external-ip   "$(curl -4 -s https://ifconfig.me)" \
--node-ip            "$(tailscale ip -4 | tr -d '\n')" \
--advertise-address  "$(tailscale ip -4 | tr -d '\n')" \
--flannel-iface      tailscale0
```

Environment variables:

```bash
echo "export KUBECONFIG=/etc/rancher/k3s/k3s.yaml" >> ~/.bash_profile
echo "export K3S_RESOLV_CONF=/etc/resolv.conf" >> ~/.bash_profile
source ~/.bash_profile
```

Get cluster token:

```bash
cat /var/lib/rancher/k3s/server/node-token
```

Configure registry mirror:

```bash
cat << EOF > /etc/rancher/k3s/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "https://<my-docker-mirror-host>"
EOF
```

Common operations:

```bash
/usr/local/bin/k3s-killall.sh
journalctl -u k3s -f
kubectl describe nodes
kubectl get apiservice
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
  | xargs -I{} kubectl taint nodes {} no-pods=deny:NoSchedule
```

Uninstall server:

```bash
/usr/local/bin/k3s-uninstall.sh
```

---

# cert-manager 🔐

Install cert-manager via Helm:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

---

# Rancher 🐮

Install Rancher (Kubernetes v1.25+):

```bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

helm upgrade --install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --create-namespace \
  --version 2.11.3 \
  --set hostname=k3s.run \
  --set replicas=1 \
  --set global.cattle.psp.enabled=false
```

Bootstrap URL:

```bash
echo https://rancher.dev.run/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
```

Reset admin password:

```bash
kubectl --kubeconfig $KUBECONFIG -n cattle-system exec \
$(kubectl --kubeconfig $KUBECONFIG -n cattle-system get pods -l app=rancher --no-headers | head -1 | awk '{ print $1 }') \
-c rancher -- reset-password
```
