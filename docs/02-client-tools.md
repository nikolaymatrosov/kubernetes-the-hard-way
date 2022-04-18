# Установка инструментов

На этом шаге вы установите инструменты командной строки, которые понадобятся для выполнения туториала: 
[cfssl](https://github.com/cloudflare/cfssl), [cfssljson](https://github.com/cloudflare/cfssl) 
и [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl).

## Установка CFSSL
`cfssl` и `cfssljson` утилиты, которые понадобятся для настройки 
[инфраструктуры открытых ключей](https://ru.wikipedia.org/wiki/Инфраструктура_открытых_ключей) и генерации TLS 
сертификатов.

Скачайте и установите `cfssl` и `cfssljson`:

### OS X

```bash
curl -o cfssl https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/darwin/cfssl
curl -o cfssljson https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/darwin/cfssljson
chmod +x cfssl cfssljson
sudo mv cfssl cfssljson /usr/local/bin/
```
Другой способ — установить при помощи пакетного менеджера [Homebrew](https://brew.sh):

```bash
brew install cfssl
```

### Linux

```bash
wget -q --show-progress --https-only --timestamping \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
  https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson
chmod +x cfssl cfssljson
sudo mv cfssl cfssljson /usr/local/bin/
```

### Проверка

Проверьте, что `cfssl` и `cfssljson` имеют версии 1.4.1 или выше:

```bash
cfssl version
```

> output

```
Version: 1.4.1
Runtime: go1.12.12
```

```bash
cfssljson --version
```
> output
```
Version: 1.4.1
Runtime: go1.12.12
```

## Установите kubectl

`kubectl` утилита командной строки используемая для взаимодействия с Kubernetes API Server. Скачайте и установите
`kubectl` из официальных источников:

### OS X

```bash
curl -o kubectl https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/darwin/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Или вы также можете воспользоваться Homebrew.
```bash
brew install kubectl
```

### Linux

```bash
wget https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

### Проверка

Убедитесь, что `kubectl` версии 1.21.0 или выше:

```bash
kubectl version --client
```

> output

```
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
```

Дальше: [Создание виртуальных машин](03-compute-resources.md)
