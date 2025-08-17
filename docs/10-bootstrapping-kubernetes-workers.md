# Развертывание рабочих узлов Kubernetes

В этой лабораторной вы развернете два рабочих узла Kubernetes. Будут установлены следующие компоненты: [runc](https://github.com/opencontainers/runc), [container networking plugins](https://github.com/containernetworking/cni), [containerd](https://github.com/containerd/containerd), [kubelet](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet) и [kube-proxy](https://kubernetes.io/docs/concepts/cluster-administration/proxies).

## Предварительные требования

Команды в этом разделе должны выполняться с `jumpbox`.

Скопируйте бинарные файлы Kubernetes и файлы systemd unit на каждый экземпляр рабочего узла:

```bash
for HOST in node-0 node-1; do
  SUBNET=$(grep ${HOST} machines.txt | cut -d " " -f 4)
  sed "s|SUBNET|$SUBNET|g" \
    configs/10-bridge.conf > 10-bridge.conf

  sed "s|SUBNET|$SUBNET|g" \
    configs/kubelet-config.yaml > kubelet-config.yaml

  scp 10-bridge.conf kubelet-config.yaml \
  root@${HOST}:~/
done
```

```bash
for HOST in node-0 node-1; do
  scp \
    downloads/worker/* \
    downloads/client/kubectl \
    configs/99-loopback.conf \
    configs/containerd-config.toml \
    configs/kube-proxy-config.yaml \
    units/containerd.service \
    units/kubelet.service \
    units/kube-proxy.service \
    root@${HOST}:~/
done
```

```bash
for HOST in node-0 node-1; do
  scp \
    downloads/cni-plugins/* \
    root@${HOST}:~/cni-plugins/
done
```

Команды в следующем разделе должны выполняться на каждом экземпляре рабочего узла: `node-0`, `node-1`. Подключитесь к экземпляру рабочего узла используя команду `ssh`. Пример:

```bash
ssh root@node-0
```

## Подготовка рабочего узла Kubernetes

Установите зависимости ОС:

```bash
{
  apt-get update
  apt-get -y install socat conntrack ipset kmod
}
```

> Бинарный файл socat обеспечивает поддержку команды `kubectl port-forward`.

### Отключение Swap

Kubernetes имеет ограниченную поддержку использования swap памяти, поскольку сложно предоставлять гарантии и учитывать использование памяти подами когда задействован swap.

Проверьте, отключен ли swap:

```bash
swapon --show
```

Если вывод пустой, то swap отключен. Если swap включен, выполните следующую команду, чтобы немедленно отключить swap:

```bash
swapoff -a
```

> Чтобы убедиться, что swap остается отключенным после перезагрузки, обратитесь к документации вашего дистрибутива Linux.

Создайте директории для установки:

```bash
mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes
```

Установите бинарные файлы рабочего узла:

```bash
{
  mv crictl kube-proxy kubelet runc \
    /usr/local/bin/
  mv containerd containerd-shim-runc-v2 containerd-stress /bin/
  mv cni-plugins/* /opt/cni/bin/
}
```

### Настройка CNI сети

Создайте конфигурационный файл сети `bridge`:

```bash
mv 10-bridge.conf 99-loopback.conf /etc/cni/net.d/
```

Чтобы убедиться, что сетевой трафик, проходящий через CNI сеть `bridge`, обрабатывается `iptables`, загрузите и настройте модуль ядра `br-netfilter`:

```bash
{
  modprobe br-netfilter
  echo "br-netfilter" >> /etc/modules-load.d/modules.conf
}
```

```bash
{
  echo "net.bridge.bridge-nf-call-iptables = 1" \
    >> /etc/sysctl.d/kubernetes.conf
  echo "net.bridge.bridge-nf-call-ip6tables = 1" \
    >> /etc/sysctl.d/kubernetes.conf
  sysctl -p /etc/sysctl.d/kubernetes.conf
}
```

### Настройка containerd

Установите конфигурационные файлы `containerd`:

```bash
{
  mkdir -p /etc/containerd/
  mv containerd-config.toml /etc/containerd/config.toml
  mv containerd.service /etc/systemd/system/
}
```

### Настройка Kubelet

Создайте конфигурационный файл `kubelet-config.yaml`:

```bash
{
  mv kubelet-config.yaml /var/lib/kubelet/
  mv kubelet.service /etc/systemd/system/
}
```

### Настройка Kubernetes Proxy

```bash
{
  mv kube-proxy-config.yaml /var/lib/kube-proxy/
  mv kube-proxy.service /etc/systemd/system/
}
```

### Запуск служб рабочего узла

```bash
{
  systemctl daemon-reload
  systemctl enable containerd kubelet kube-proxy
  systemctl start containerd kubelet kube-proxy
}
```

Проверьте, запущена ли служба kubelet:

```bash
systemctl is-active kubelet
```

```text
active
```

Убедитесь, что вы выполнили шаги в этом разделе на каждом рабочем узле, `node-0` и `node-1`, прежде чем переходить к следующему разделу.

## Проверка

Выполните следующие команды с машины `jumpbox`.

Выведите список зарегистрированных узлов Kubernetes:

```bash
ssh root@server \
  "kubectl get nodes \
  --kubeconfig admin.kubeconfig"
```

```
NAME     STATUS   ROLES    AGE    VERSION
node-0   Ready    <none>   1m     v1.32.3
node-1   Ready    <none>   10s    v1.32.3
```

Дальше: [Настройка kubectl для удаленного доступа](11-configuring-kubectl.md)
