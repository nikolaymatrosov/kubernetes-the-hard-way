# Развертывание воркеров Kubernetes

В этой лабораторной вы настроите две ноды с воркерами Kubernetes. Будут установлены следующие компоненты:
[runc](https://github.com/opencontainers/runc)
, [container networking plugins](https://github.com/containernetworking/cni)
, [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/admin/kubelet) и
[kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Важное

Команды из этой лабораторной выполняются на каждом инстансе воркере: `node-0` и `node-1`. Зайдите на каждый инстанс.

### Параллельное выполнение команд через tmux

[tmux](https://github.com/tmux/tmux/wiki) может быть использован для параллельного выполнения команд на нескольких
инстансах. Смотри [Параллельное выполнение команд в tmux](01-prerequisites.md)

## Настройка Kubernetes Worker Node

### Подключение к worker nodes

```bash
# Подключитесь к node-0
ssh yc-user@<node-0-external-ip>

# Подключитесь к node-1
ssh yc-user@<node-1-external-ip>
```

### Распаковка конфигураций

```bash
# Распакуйте конфигурации
tar -xzf node-0-certificates.tar.gz  # на node-0
tar -xzf node-1-certificates.tar.gz  # на node-1
```

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
K8S_VERSION=v1.32.3
wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.32.0/crictl-v1.32.0-linux-amd64.tar.gz \
  https://github.com/opencontainers/runc/releases/download/v1.3.0-rc.1/runc.amd64 \
  https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-amd64-v1.6.2.tgz \
  https://github.com/containerd/containerd/releases/download/v2.1.0/containerd-2.1.0-linux-amd64.tar.gz \
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
  tar -xvf crictl-v1.32.0-linux-amd64.tar.gz
  tar -xvf containerd-2.1.0-linux-amd64.tar.gz -C containerd
  sudo tar -xvf cni-plugins-linux-amd64-v1.6.2.tgz -C /opt/cni/bin/
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

Создайте конфигурационный файл CNI:

```bash
sudo tee /etc/cni/net.d/10-bridge.conf <<EOF
{
    "cniVersion": "1.0.0",
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

Создайте конфигурационный файл для loopback:

```bash
sudo tee /etc/cni/net.d/99-loopback.conf <<EOF
{
    "cniVersion": "1.0.0",
    "name": "loopback",
    "type": "loopback"
}
EOF
```

### Настройте containerd

Создайте конфигурационный файл containerd:

```bash
sudo mkdir -p /etc/containerd/

cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v2.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
    [plugins.cri.containerd.untrusted_workload_runtime]
      runtime_type = "io.containerd.runtime.v2.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = "/var/lib/containerd/io.containerd.runtime.v2.linux"
EOF
```

Создайте systemd сервис для containerd:

```bash
cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### Настройте kubelet

Создайте конфигурационный файл для kubelet:

```bash
sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
```

Создайте systemd сервис для kubelet:

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
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Создайте конфигурационный файл kubelet:

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
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubernetes/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubernetes/${HOSTNAME}-key.pem"
serverTLSBootstrap: true
EOF
```

### Настройте kube-proxy

Создайте конфигурационный файл для kube-proxy:

```bash
sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Создайте конфигурационный файл kube-proxy:

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

Создайте systemd сервис для kube-proxy:

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

### Запустите сервисы

```bash
{
  sudo systemctl daemon-reload
  sudo systemctl enable containerd kubelet kube-proxy
  sudo systemctl start containerd kubelet kube-proxy
}
```

### Проверка

Проверьте статус сервисов:

```bash
sudo systemctl status containerd kubelet kube-proxy
```

## Альтернатива: Больше worker nodes (для продакшена)

Для продакшен окружения рекомендуется использовать больше worker nodes. Вот пример для 3 worker nodes:

```bash
# Создайте дополнительные worker nodes
for i in 2; do
  name=node-${i}
  # ... создание машины
done
```

Дальше: [Настраиваем удаленный доступ kubectl](11-configuring-kubectl.md)
