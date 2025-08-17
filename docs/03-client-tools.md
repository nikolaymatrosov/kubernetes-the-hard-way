# Установка инструментов

На этом шаге вы установите инструменты командной строки, которые понадобятся для выполнения туториала:
[cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl)
и [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).

## Подход с Jumpbox

В современной версии туториала мы используем **jumpbox** - центральную машину для управления кластером.
Это упрощает развертывание и обновление компонентов.

## Установка Yandex Cloud CLI

Установите Yandex Cloud CLI, который понадобится для управления ресурсами в Yandex Cloud.

Для установки Yandex Cloud CLI выполните следующие команды:

```bash
sudo apt update && sudo apt upgrade -y

sudo apt install -y jq git
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
source ~/.bashrc
```

### Проверка

```bash
yc --version
```
> output
```
Yandex Cloud CLI 0.158.0 linux/amd64
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
> output
```
total 566M
-rw-r--r-- 1 yc-user 51M Jan  6  2025 cni-plugins-linux-amd64-v1.6.2.tgz
-rw-r--r-- 1 yc-user 38M Mar 18 02:33 containerd-2.1.0-beta.0-linux-amd64.tar.gz
-rw-r--r-- 1 yc-user 19M Dec  9  2024 crictl-v1.32.0-linux-amd64.tar.gz
-rw-r--r-- 1 yc-user 23M Mar 27 23:15 etcd-v3.6.0-rc.3-linux-amd64.tar.gz
-rw-r--r-- 1 yc-user 89M Mar 12 03:31 kube-apiserver
-rw-r--r-- 1 yc-user 83M Mar 12 03:31 kube-controller-manager
-rw-r--r-- 1 yc-user 55M Mar 12 03:31 kubectl
-rw-r--r-- 1 yc-user 74M Mar 12 03:31 kubelet
-rw-r--r-- 1 yc-user 64M Mar 12 03:31 kube-proxy
-rw-r--r-- 1 yc-user 63M Mar 12 03:31 kube-scheduler
-rw-r--r-- 1 yc-user 12M Mar  4 12:14 runc.amd64
```

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
