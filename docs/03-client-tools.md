# Установка инструментов

На этом шаге вы установите инструменты командной строки, которые понадобятся для выполнения туториала: 
[cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl) 
и [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).

## Подход с Jumpbox

В современной версии туториала мы используем **jumpbox** - центральную машину для управления кластером. 
Это упрощает развертывание и обновление компонентов.


## Установка CFSSL
`cfssl` и `cfssljson` утилиты, которые понадобятся для настройки 
[инфраструктуры открытых ключей](https://ru.wikipedia.org/wiki/Инфраструктура_открытых_ключей) и генерации TLS 
сертификатов.

Скачайте и установите `cfssl` и `cfssljson`:

```bash
sudo apt update && sudo apt upgrade -y
wget https://golang.org/dl/go1.24.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.24.5.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin:$HOME/go/bin' >> ~/.bashrc
source ~/.bashrc

go install github.com/cloudflare/cfssl/cmd/...@latest
sudo apt install -y jq git
```

### Проверка

Проверьте, что `cfssl` и `cfssljson` имеют версии 1.6.4 или выше:

```bash
cfssl version
```

> output

```
Version: dev
Runtime: go1.24.5
```

```bash
cfssljson --version
```
> output
```
Version: dev
Runtime: go1.24.5
```
## Склонируйте репозиторий
Склонируйте репозиторий с туториалом:

```bash
git clone 


## Установите kubectl

`kubectl` утилита командной строки используемая для взаимодействия с Kubernetes API Server. Скачайте и установите
`kubectl` из официальных источников:


### Linux

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
mv kubectl /usr/local/bin/kubectl
chmod +x /usr/local/bin/kubectl
```

### Проверка

Убедитесь, что `kubectl` версии 1.32.3 или выше:

```bash
kubectl version --client
```

> output

```
Client Version: v1.33.3
Kustomize Version: v5.6.0
```

## Централизованное управление бинарными файлами

### Создание структуры директорий

После создания jumpbox в следующем шаге, создайте структуру для управления бинарными файлами:

```bash
# Подключитесь к jumpbox
ssh yc-user@<jumpbox-external-ip>

# Создайте директории для компонентов
mkdir -p ~/kubernetes-the-hard-way/{client,controller,worker}
mkdir -p ~/kubernetes-the-hard-way/bin
```

## Установка компонентов

```bash
apt-get update
apt-get -y install wget curl vim openssl git
```

### Загрузка компонентов

```bash
# Kubernetes компоненты
cd ~/kubernetes-the-hard-way
KUBERNETES_VERSION="v1.32.3"
CONTAINERD_VERSION="v2.1.0"
CNI_VERSION="v1.6.2"
ETCD_VERSION="v3.6.0"

# Скачайте Kubernetes компоненты
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/${KUBERNETES_VERSION}/bin/linux/amd64/kube-apiserver"
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/${KUBERNETES_VERSION}/bin/linux/amd64/kube-controller-manager"
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/${KUBERNETES_VERSION}/bin/linux/amd64/kube-scheduler"
wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/${KUBERNETES_VERSION}/bin/linux/amd64/kubectl"

# Скачайте containerd
wget -q --show-progress --https-only --timestamping \
  "https://github.com/containerd/containerd/releases/download/${CONTAINERD_VERSION}/containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz"

# Скачайте CNI plugins
wget -q --show-progress --https-only --timestamping \
  "https://github.com/containernetworking/plugins/releases/download/${CNI_VERSION}/cni-plugins-linux-amd64-${CNI_VERSION}.tgz"

# Скачайте etcd
wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/${ETCD_VERSION}/etcd-${ETCD_VERSION}-linux-amd64.tar.gz"

# Скачайте crictl
wget -q --show-progress --https-only --timestamping \
  "https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.32.0/crictl-v1.32.0-linux-amd64.tar.gz"

# Скачайте runc
wget -q --show-progress --https-only --timestamping \
  "https://github.com/opencontainers/runc/releases/download/v1.3.0-rc.1/runc.amd64"
```

### Установка прав доступа

```bash
# Сделайте все файлы исполняемыми
chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl runc.amd64

# Переименуйте runc
mv runc.amd64 runc
```

### Распаковка архивов

```bash
# Распакуйте containerd
tar -xzf containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz
rm containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz

# Распакуйте CNI plugins
tar -xzf cni-plugins-linux-amd64-${CNI_VERSION}.tgz
rm cni-plugins-linux-amd64-${CNI_VERSION}.tgz

# Распакуйте etcd
tar -xzf etcd-${ETCD_VERSION}-linux-amd64.tar.gz
rm etcd-${ETCD_VERSION}-linux-amd64.tar.gz

# Распакуйте crictl
tar -xzf crictl-v1.32.0-linux-amd64.tar.gz
rm crictl-v1.32.0-linux-amd64.tar.gz
```

### Организация файлов

```bash
# Создайте структуру для разных типов машин
mkdir -p client controller worker

# Файлы для клиентских машин (jumpbox)
cp kubectl client/

# Файлы для control plane
cp kube-apiserver kube-controller-manager kube-scheduler controller/
cp etcd-${ETCD_VERSION}-linux-amd64/etcd controller/

# Файлы для worker nodes
cp kube-proxy worker/
cp containerd-shim-runc-v2 worker/
cp runc worker/
cp crictl worker/
cp -r bin/ worker/
```

Дальше: [Создание виртуальных машин](04-compute-resources.md)
