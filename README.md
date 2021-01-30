# kbots
Kubernetes Platform for Robots

## Install Fedora

See more details at https://pagure.io/arm-image-installer.

```bash
sudo dnf install arm-image-installer
```

```bash
sudo fedora-arm-image-installer \
  --image=$HOME/Downloads/Fedora-Server-33-1.3.aarch64.raw.xz
  --media=/dev/sdc \
  --target=rpi4 \
  --resizefs \
  --addkey $HOME/.ssh/id_ed25519.pub
```

## Configure Fedora

Update hostname

```bash
echo pi4 > /etc/hostname
reboot
```

Disable Swap

```bash
dnf remove zram-generator-defaults
reboot
```

Disable cgroups v2

```bash
grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=0"
reboot
```


Install wpa_supplicant to enable WiFi networking

```bash
dnf install wpa_supplicant
reboot

nmcli general status
nmcli device wifi connect <ssid> password <password>
```

Update system

```bash
dnf update
reboot
```

## Install Container Runtime

See more details at https://cri-o.io/.

```bash
dnf module enable cri-o:1.20
dnf install cri-o

systemctl enable crio
systemctl start crio
```

## Install Kubernetes

See more details at https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/.

Letting iptables see bridged traffic

```bash
cat <<EOF | tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system
```
Installing kubeadm, kubelet and kubectl

```bash
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet
```

Configure and start the deployment

```bash
cat <<EOF | tee /root/kubeadm-init.yaml
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.20.0
...
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
...
EOF

kubeadm init --config=kubeadm-init.yaml
```

Verify installation

```bash
export KUBECONFIG=/etc/kubernetes/admin.conf

kubectl get pods --all-namespaces
NAMESPACE     NAME                          READY   STATUS    RESTARTS   AGE
kube-system   coredns-74ff55c5b-rxqrt       1/1     Running   0          107s
kube-system   coredns-74ff55c5b-vnh7s       1/1     Running   0          107s
kube-system   etcd-pi4                      1/1     Running   0          98s
kube-system   kube-apiserver-pi4            1/1     Running   0          98s
kube-system   kube-controller-manager-pi4   1/1     Running   0          98s
kube-system   kube-proxy-5pddb              1/1     Running   0          107s
kube-system   kube-scheduler-pi4            1/1     Running   0          97s
```
