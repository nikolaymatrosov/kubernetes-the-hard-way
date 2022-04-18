# Развертывание воркеров Kubernetes

В этой лабораторной вы настроите три ноды с воркерами Kubernetes. Будут установлены следующие компоненты:
[runc](https://github.com/opencontainers/runc)
, [container networking plugins](https://github.com/containernetworking/cni)
, [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet) и
[kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Важное

Команды из этой лабораторной должный выполняться на каждом инстансе воркере: `worker-0`, `worker-1`
и `worker-2`. Зайдите на каждый инстанс.

### Параллельное выполнение команд через tmux

[tmux](https://github.com/tmux/tmux/wiki) может быть использован для параллельного выполнения команд на нескольких
инстансах. Смотри [Параллельное выполнение команд в tmux](01-prerequisites.md)

## Настройка Kubernetes Worker Node

Установите системные зависимости:

```bash
{
  sudo apt-get update
  sudo apt-get -y install socat conntrack ipset
}
```

> `socat` необходим чтобы выполнять `kubectl port-forward`.

### Отключаем Swap

По умолчанию кублет будет падать если [swap](https://help.ubuntu.com/community/SwapFaq) включен.
[Рекумендуется](https://github.com/kubernetes/kubernetes/issues/7294) отключить своп, чтобы быть уверенным, что
Kubernetes правильное выделение ресурсов и качство работы.

Проверьте, что своп включен:

```bash
sudo swapon --show
```

Если вывод пустой, то своп отключен. Если же он включен, то выполните следующую команду, чтобы его отключить:

```bash
sudo swapoff -a
```

> Чтобы убедится, что своп останется отключенным после перезагрузки проверьте инструкцию вашего дистрибутива Linux.

### Скачайте и установите исполняемые файлы воркеров

```bash
K8S_VERSION=v1.21.0
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/${K8S_VERSION}/crictl-${K8S_VERSION}-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc93/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v0.9.1/cni-plugins-linux-amd64-v0.9.1.tgz \
  https://github.com/containerd/containerd/releases/download/v1.4.4/containerd-1.4.4-linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/${K8S_VERSION}/bin/linux/amd64/kubelet
```

Создайте необходимые директории:

```bash
sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Установите исполняемые файлы:

```bash
{
  mkdir containerd
  tar -xvf crictl-v1.21.0-linux-amd64.tar.gz
  tar -xvf containerd-1.4.4-linux-amd64.tar.gz -C containerd
  sudo tar -xvf cni-plugins-linux-amd64-v0.9.1.tgz -C /opt/cni/bin/
  sudo mv runc.amd64 runc
  chmod +x crictl kubectl kube-proxy kubelet runc 
  sudo mv crictl kubectl kube-proxy kubelet runc /usr/local/bin/
  sudo mv containerd/bin/* /bin/
}
```

### Настройте CNI сети

Получите CIDR диапазон для подов из метаданных текущего инстанса:

```bash
POD_CIDR=$(curl -s -H "Metadata-Flavor: Google" \
  http://metadata.google.internal/computeMetadata/v1/instance/attributes/pod-cidr)
```

Создайте конфигурационный файл для сети `bridge`

```bash
cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.4.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

Создайте конфигурационный файл для сети `loopback`

```bash
cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.4.0",
    "name": "lo",
    "type": "loopback"
}
EOF
```

### Настройте containerd

Создайте конфигурационный файл `containerd`:

```bash
sudo mkdir -p /etc/containerd/
```

```bash
cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
EOF
```

Создайте `containerd.service` файл описания юнита в systemd:

```bash
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### Настройте Kubelet

```bash
{
  sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
  sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
  sudo mv ca.pem /var/lib/kubernetes/
}
```

Создайте конфигурационный файл `kubelet-config.yaml`:

```bash
cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

> Конфиг `resolvConf` используется для того чтобы избежать циклов когда используется CoreDNS при работе в системах
> с `systemd-resolved`.

Создайте `kubelet.service` файл описания юнита в systemd:

```bash
cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

### Настройте Kubernetes Proxy

```bash
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Создайте конфигурационный файл `kube-proxy-config.yaml`:

```bash
cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

Создайте `kube-proxy.service` файл описания юнита в systemd:

```bash
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

### Запустите сервис воркеров

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd kubelet kube-proxy
  sudo systemctl start containerd kubelet kube-proxy
}
```

> Помните, все команды выше нужно выполнить на всех трех воркерах `worker-0`, `worker-1` и `worker-2`.

## Проверка

> Этот код нужно выполнять с машины откуда вы создавали окружение.

Выведите список нод Kubernetes:

```bash
CONTROLLER_IP=$(yc compute instance get --name=controller-0 --format json | jq '.network_interfaces[0].primary_v4_address.one_to_one_nat.address' -r)
ssh yc-user@${CONTROLLER_IP} "kubectl get nodes --kubeconfig admin.kubeconfig"
```

> output

```
NAME       STATUS   ROLES    AGE   VERSION
worker-0   Ready    <none>   22s   v1.21.0
worker-1   Ready    <none>   22s   v1.21.0
worker-2   Ready    <none>   22s   v1.21.0
```

Дальше: [Настраиваем удаленный доступ kubectl](10-configuring-kubectl.md)
