# ติดตั้ง Kubernetes Worker Nodes
kubelet และ kube-proxy เป็นส่วนประกอบ 2 ส่วนของ Kubernetes ที่ติดตั้งที่ Worker Node ที่ใช้ในการจัดการสิ่งต่าง ๆ ที่เกิดขี้นใน Cluster และยังต้องสื่อสารกับ master node ตลอดเวลา นอกเหนือจาก นั้นก็มี Container Runtime อีกด้วย 
## เตรียม Kubernetes Worker Node Binaries [all worker node]
```
KUBEVER=v1.19.2

mkdir -p \
 /etc/cni/net.d \
 /opt/cni/bin \
 /var/lib/kubelet \
 /var/lib/kube-proxy \
 /var/lib/kubernetes \
 /var/run/kubernetes

dnf install -y wget
wget --show-progress https://dl.k8s.io/${KUBEVER}/kubernetes-node-linux-amd64.tar.gz

tar xvfz kubernetes-node-linux-amd64.tar.gz
cd kubernetes/node/bin/
mv kubectl kube-proxy kubelet /usr/local/bin/
cd
```
## เตรียมข้อมูลที่ใช้ในการทำงานของ kubelet [all worker node]
```
cp ${HOSTNAME}.key ${HOSTNAME}.crt /var/lib/kubelet/
cp ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
cp ca.crt /var/lib/kubernetes/

cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.crt"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.96.0.10"
resolvConf: "/etc/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}.key"
EOF
```
## สร้าง kubelet.service สำหรับ systemd [all worker node]
```
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=docker.service
Requires=docker.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --tls-cert-file=/var/lib/kubelet/${HOSTNAME}.crt \\
  --tls-private-key-file=/var/lib/kubelet/${HOSTNAME}.key \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
## เตรียมข้อมูลที่ใช้ในการทำงานของ kube-proxy [all worker node]
```
cp kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.0.0.0/16"
EOF
```
## สร้าง kube-proxy.service สำหรับ systemd [all worker node]
```
cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```
## เริ่มการทำงานของ kubelet และ kube-proxy [all worker node]
```
systemctl daemon-reload
systemctl enable kubelet kube-proxy --now
```
## ทดสอบและผลการทดสอบ [master0]
```
kubectl get nodes --kubeconfig admin.kubeconfig
```
>ผลการทดสอบ
```
NAME    STATUS     ROLES    AGE    VERSION
node0   NotReady   <none>   10m    v1.19.0
node1   NotReady   <none>   6m7s   v1.19.0
```
> ผลที่ได้ สถานะของ node จะยังเป็น Not Ready เนื่องจาก ในส่วนของ Network ยังไม่ได้ถูกกำหนดค่า

**Next>** [Kubernetes Object Authorization](11-kubernetes-object-authorization.md)

**<Prev** [ติดตั้ง Loadbalancer สำหรับ master node](09-loadbalancer.md)
