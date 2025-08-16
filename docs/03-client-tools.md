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
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
source ~/.bashrc
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
git clone https://github.com/nikolaymatrosov/kubernetes-the-hard-way.git
cd kubernetes-the-hard-way
```

## Скачайте бинарные файлы
Скачайте необходимые бинарные файлы указанные в `downloads.txt`

```bash
wget -q --show-progress \
  --https-only \
  --timestamping \
  -P downloads \
  -i downloads.txt
```

Проверьте, что все файлы скачаны:

```bash
ls -oh downloads
```
Extract the component binaries from the release archives and organize them under the downloads directory.

Распакуйте бинарные файлы из архивов и создайте структуру директорий для управления бинарными файлами:

```bash
{
  ARCH=$(dpkg --print-architecture)
  mkdir -p downloads/{client,cni-plugins,controller,worker}
  tar -xvf downloads/crictl-v1.32.0-linux-${ARCH}.tar.gz \
    -C downloads/worker/
  tar -xvf downloads/containerd-2.1.0-beta.0-linux-${ARCH}.tar.gz \
    --strip-components 1 \
    -C downloads/worker/
  tar -xvf downloads/cni-plugins-linux-${ARCH}-v1.6.2.tgz \
    -C downloads/cni-plugins/
  tar -xvf downloads/etcd-v3.6.0-rc.3-linux-${ARCH}.tar.gz \
    -C downloads/ \
    --strip-components 1 \
    etcd-v3.6.0-rc.3-linux-${ARCH}/etcdctl \
    etcd-v3.6.0-rc.3-linux-${ARCH}/etcd
  mv downloads/{etcdctl,kubectl} downloads/client/
  mv downloads/{etcd,kube-apiserver,kube-controller-manager,kube-scheduler} \
    downloads/controller/
  mv downloads/{kubelet,kube-proxy} downloads/worker/
  mv downloads/runc.${ARCH} downloads/worker/runc
}
```
```bash
rm -rf downloads/*gz
```

Сделайте бинарные файлы исполняемыми:

```bash
{
  chmod +x downloads/{client,cni-plugins,controller,worker}/*
}
```



## Установите kubectl

`kubectl` утилита командной строки используемая для взаимодействия с Kubernetes API Server. Скачайте и установите
`kubectl` из официальных источников:

```bash
{
  chmod +x downloads/client/kubectl
  sudo cp downloads/client/kubectl /usr/local/bin/
}
```

### Проверка

Убедитесь, что `kubectl` версии 1.32.3 или выше:

```bash
kubectl version --client
```

> output

```
Client Version: v1.32.3
Kustomize Version: v5.5.0
```

Дальше: [Создание виртуальных машин](04-compute-resources.md)
